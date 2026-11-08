---
title: "FastAPI + async SQLAlchemy のテストを Testcontainers で書く"
emoji: "🐳"
type: "tech"
topics: ["python", "fastapi", "pytest", "testcontainers", "postgres"]
published: true
---

:::message
これは下書きです。コードは [hello-testcontainers-py](https://github.com/parkwestlabs/hello-testcontainers-py) の現在の内容に合わせています。
:::

## この記事で扱うこと

- Testcontainers とは何か、Docker Compose とどう違うか
- FastAPI + async SQLAlchemy の構成で、本物の PostgreSQL に対するテストを書く方法
- テストごとにデータを分離する方法（rollback を使う）
- 実際にハマったポイント

コードはすべて上記リポジトリにあります。

## Testcontainers とは

Testcontainers は、**テストの実行中に Docker で本物のミドルウェアを起動し、テスト終了後に自動で片付けてくれる**ライブラリです。Python 版は `testcontainers` パッケージです。

Docker Compose との違いは、コンテナの管理をテストコードの中で行うかどうかです。

| 項目 | Docker Compose | Testcontainers |
|---|---|---|
| 起動・停止 | テスト前後に手動またはスクリプトで実行 | テストコードが自動で実行 |
| ポート | 固定（5432 など）で衝突しやすい | 空いているポートを自動で割り当て |
| CI | CI 側に別途設定が必要 | Docker があれば同じコードが動く |

開発用の常駐 DB は Docker Compose、テスト用の使い捨て DB は Testcontainers、と使い分けるのが一般的です。

## 前提と構成

- Python 3.15（3.14 でも動作を確認済み）、uv、Docker（起動していること）
- FastAPI、SQLAlchemy（async）、asyncpg、PostgreSQL 18

```text
src/hello_testcontainers_py/
  main.py      FastAPI アプリとエンドポイント
  db.py        Engine、セッションファクトリ、get_db
  models.py    SQLAlchemy のモデル
tests/
  conftest.py            fixture モジュールの登録
  fixtures/database.py   コンテナの起動、セッション、テスト用クライアント
  test_app.py            テスト本体
```

依存関係の追加は次の通りです。

```bash
uv add "fastapi[standard]" "sqlalchemy[asyncio]" asyncpg
uv add --dev pytest pytest-asyncio "testcontainers[postgres]"
```

## 1. 本番のセッションを用意する

エンドポイントは `Depends(get_db)` でセッションを受け取ります。`get_db` は環境変数 `DATABASE_URL` から Engine を作ります。

```python
# src/hello_testcontainers_py/db.py
@lru_cache
def get_engine() -> AsyncEngine:
    database_url = os.environ.get("DATABASE_URL")
    if not database_url:
        raise RuntimeError("DATABASE_URL environment variable is not set")
    return create_async_engine(database_url)


def create_session_factory(
    bind: AsyncEngine | AsyncConnection,
) -> async_sessionmaker[AsyncSession]:
    # expire_on_commit=False が重要（後述）
    return async_sessionmaker(bind=bind, class_=AsyncSession, expire_on_commit=False)


async def get_db() -> AsyncGenerator[AsyncSession]:
    session_factory = create_session_factory(get_engine())
    async with session_factory() as session:
        yield session
```

`create_session_factory` を本番とテストの両方で使うことがポイントです。設定を 1 か所にまとめると、テストと本番の設定がずれません。

## 2. コンテナを起動する（session スコープ）

`tests/fixtures/database.py` で PostgreSQL を起動します。

```python
from testcontainers.community.postgres import PostgresContainer


@pytest.fixture(scope="session")
def postgres_container_url() -> Generator[str]:
    with PostgresContainer("postgres:18-alpine", driver="asyncpg") as postgres:
        yield postgres.get_connection_url()
```

ポイントは次の 2 つです。

- **`scope="session"`**：コンテナの起動は数秒かかるため、テスト全体で 1 回だけ起動します。
- **`with` 文**：ブロックを抜けると、コンテナは自動で停止・削除されます。

:::message alert
`from testcontainers.postgres import PostgresContainer` と書くと、現在のバージョン（4.15.0）では `DeprecationWarning` が出ます。`testcontainers.community.postgres` を使ってください。
:::

## 3. テストごとにデータを分離する（rollback）

コンテナは全テストで共有するため、データが残ると他のテストに影響します。最初は `session.begin()` で囲んでいましたが、これは正常終了時に **commit** するため、データが残りました。

修正後の fixture は、外側のトランザクションを rollback する方式です。

```python
@pytest_asyncio.fixture()
async def db_session(async_db_engine: AsyncEngine) -> AsyncGenerator[AsyncSession]:
    async with async_db_engine.connect() as conn:
        # 外側のトランザクションを開始
        outer = await conn.begin()

        # SAVEPOINT モード: セッション内の commit() は SAVEPOINT の解放だけ
        async with create_session_factory(conn)(
            join_transaction_mode="create_savepoint"
        ) as session:
            yield session

        # テスト終了時に外側を rollback して、全変更を取り消す
        await outer.rollback()
```

`join_transaction_mode="create_savepoint"` を指定すると、エンドポイント内の `commit()` が外側のトランザクションを確定させなくなります。そのため、テスト終了時の rollback で全変更が取り消されます。

分離が効いているかは、テストで確認できます。

```python
async def test_db_is_isolated_between_tests(db_session: AsyncSession):
    count = await db_session.scalar(select(func.count()).select_from(User))
    assert count == 0
```

前のテストで作ったユーザーは残らないため、このテストは常に件数 0 から始まります。

## 4. FastAPI の依存関係を差し替える

テスト用のクライアントでは、`get_db` をテスト用のセッションに差し替えます。

```python
@pytest_asyncio.fixture()
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient]:
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

    app.dependency_overrides.clear()
```

エンドポイントのコードを変えずに、テスト用の DB を使えます。

## 5. テストを書く

```python
async def test_create_user_endpoint(client: AsyncClient):
    response = await client.post("/users", params={"name": "アリス"})

    assert response.status_code == 200
    body = response.json()
    assert body["name"] == "アリス"
    assert isinstance(body["id"], int)
```

ID の値そのものは固定しないことにしました。コンテナを共有するため、ほかのテストが先にユーザーを作ると、ID の値は変わるからです。

## ハマったポイント

### expire_on_commit=False を忘れると失敗する

エンドポイントは `commit()` の後に `new_user.id` を読みます。`expire_on_commit` がデフォルトの `True` のままだと、async セッションでは期限切れの属性を遅延読み込みできず、次のエラーになります。

```text
MissingGreenlet: greenlet_spawn has not been called; can't call await_() here.
```

テストで `expire_on_commit=False` を指定していると、本番で設定を忘れても、テストは通ってしまいます。そこで、本番とテストで同じ `create_session_factory` を使うようにしました。

### テストが通っても、分離が効いているとは限らない

最初の版では、テストは通っていましたが、データは残っていました。`ID == 1` のアサートも、テストの実行順に依存していました。分離が効いているかは、「空から始まること」を明示的に確認すると分かりやすいです。

### 非 ASCII 文字は URL エンコードする

手動で `curl` を使うとき、`name=アリス` をそのまま URL に入れると、uvicorn が `Invalid HTTP request received` で拒否します。次のように `--data-urlencode` を使ってください。

```bash
curl -G -X POST --data-urlencode "name=アリス" "http://127.0.0.1:8000/users"
```

## 実行する

```bash
uv run pytest -v
```

初回は PostgreSQL のイメージを取得するため時間がかかります。実行には Docker の起動が必要です。

テストは `create_all` で自分のテーブルを作るため、事前の準備は不要です。一方、手元でアプリを動かす場合は `DATABASE_URL` を設定して `uv run fastapi dev` を実行します。アプリは起動時にテーブルを作成するので、空のデータベースでもそのまま使えます。

GitHub Actions でも、`ubuntu-latest` の Docker を使って同じコマンドを実行できます。PostgreSQL のサービスを YAML に書く必要はありません。

## まとめ

- Testcontainers を使うと、本番と同じ DB に対してテストできます。
- コンテナは `session` スコープで 1 回だけ起動し、テストごとに rollback でデータを分離します。
- `create_session_factory` で本番とテストの設定を共有すると、`expire_on_commit` のような設定ミスをテストで検出できます。

## 未解決の点

- アプリは起動時に `create_all` でテーブルを作成します。これは手軽さのための処理で、本番ではマイグレーション（Alembic など）に置き換える必要があります。
- コンテナの再利用（Reuse）機能は、この記事では扱っていません。
