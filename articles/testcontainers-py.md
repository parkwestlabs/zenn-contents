---
title: "Python と Testcontainers による PostgreSQL のインテグレーションテストのサンプル"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "fastapi", "pytest", "testcontainers", "postgres"]
published: true
published_at: 2025-10-08 23:35
---

* Python 3.14 FastAPI で Testcontainers を試した際の Gemini との対話録です。
* 結果のコードはこちら → https://github.com/parkwestlabs/hello-testcontainers-py

---

## Testcontainers とは？

Testcontainers は、テストコードから使い捨ての軽量なDockerコンテナをプログラム上で起動・管理できるオープンソースのライブラリです。

### 主な特徴とメリット

* 実際のサービスでテスト: モックやインメモリデータベースを使わず、本番環境と同じデータベースやメッセージキューなどをコンテナ上で動かして統合テストを実行できます。
* 自動クリーンアップ: テストの終了に合わせてコンテナが自動で破棄されるため、環境やデータが汚れません。
* 多言語対応: Java、Go、.NET、Node.js、Python、Rustなど、多くのプログラミング言語をサポートしています。
* 豊富なモジュール: PostgreSQL、MySQL、Redis、Kafkaなど、よく使われるミドルウェア向けの専用モジュールが多数用意されています。

もしよろしければ、以下について教えてください：

* お使いのプログラミング言語（Java、Go、Pythonなど）
* テストで使いたいミドルウェア（MySQL、PostgreSQL、Redisなど）

具体的な導入手順やサンプルコードをご案内できます。

---

## docker compose との違い

> localでdocker composeを使ってpostgresを起動してテストするのとどう違うのですか？ちなみに、もし言語の例を出すならpythonでお願いします。

---

ローカルで docker compose を使って手動（またはスクリプト）で PostgreSQL を起動する運用と、Testcontainers を使う運用の決定的な違いは、「コンテナのライフサイクル管理がテストコード（プログラム）の内側にあるか外側にあるか」 にあります。

具体的な違いと、Python での書き方の変化を比較してみましょう。

### 1. 主な違いの比較

一言で言えば、Docker Compose は「テストを実行する前に、開発者が事前に環境を作っておく」アプローチですが、Testcontainers は「テストコードが実行される瞬間に、必要な環境を自分で作って自分で片付ける」アプローチです。


| 比較項目 | ローカルの Docker Compose 運用 | Testcontainers 運用 |
| --- | --- | --- |
| 起動・停止の手間 | テスト前に docker compose up -d、終了後に down のコマンド実行が必要。 | テストを走らせるだけで、プログラムが自動で起動から破棄まで行う。 |
| ポートの競合 | 5432:5432 のようにホスト側ポートを固定するため、他のプロジェクトやテストとポートが競合しやすい。 | 空いているランダムなポートに自動でマッピングされるため、絶対に競合しない。 |
| 並列テストの可否 | 1つの共通データベースを奪い合うため、テストを並列で走らせるとデータの混ざり（干渉）が起きる。 | テストプロセスごとに独立したコンテナを複数同時に立ち上げられるため、安全に並列実行できる。 |
| CI（GitHub Actions等） | CI環境側で PostgreSQL サービスを立ち上げる設定（YAMLの記述）が別途必要。 | CIサーバーに Docker さえ入っていれば、ローカルと全く同じテストコードがそのまま動く。 |
| データの初期化 | 前のテストデータが残りやすいため、テストごとに TRUNCATE するなどの配慮が必要。 | テストごとに毎回「まっさらなコンテナ」を使い捨てる運用の構築が容易。 |


### 2. Python でのコード例の比較

Python でテストフレームワークの pytest を使って、PostgreSQL に接続するテストを書く場合の構成を比較します。

#### ❌ パターンA：従来の Docker Compose 運用

事前に docker compose up -d を裏で実行しておく必要があります。ポートは 5432 固定です。

```py
import pytest
import psycopg2

# ⚠️ 事前にコンテナが起動していないと、このテストはエラーで落ちる
@pytest.fixture(scope="session")
def db_connection():
    # ポートが5432で固定されている
    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        user="testuser",
        password="testpassword",
        database="testdb"
    )
    yield conn
    conn.close()

def test_insert_data(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT 1;")
    assert cursor.fetchone()[0] == 1
```

#### ⭕ パターンB：Testcontainers (Python) 運用

事前に何も起動しておく必要はありません。pytest コマンドを叩くだけで、Python が勝手にコンテナを持ってきてテストし、終わったら消去します。

Python では、公式ライブラリである testcontainers-python （特に testcontainers[postgres]）を使用します。

```py
import pytest
import psycopg2
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def db_connection():
    # 1. テスト開始時に、自動でPostgreSQLコンテナを裏で起動する
    # ポートは自動的に「空いているランダムなポート」に割り振られます
    with PostgresContainer("postgres:16-alpine") as postgres:

        # 2. コンテナが割り当てたランダムな接続情報を取得する
        conn = psycopg2.connect(
            host=postgres.get_container_host_ip(),
            port=postgres.get_exposed_port(postgres.port),
            user=postgres.username,
            password=postgres.password,
            database=postgres.dbname
        )

        yield conn
        conn.close()
    # 3. `with` ブロックを抜けると、コンテナは自動的に停止・消去（クリーンアップ）される

def test_insert_data(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT 1;")
    assert cursor.fetchone()[0] == 1
```

### まとめ：どちらを選ぶべき？

* Docker Compose が向いているケース:
ローカルでアプリケーション本体を手動で動かしながら、画面をポチポチ触ってデバッグしたい場合（＝開発環境の常駐）。
* Testcontainers が向いているケース:
CI/CD パイプラインでの自動テストや、ローカルでのユニットテスト・統合テスト（＝テストの自動化・高速化・並列化）。

「テストを実行するためだけに、わざわざ docker compose up を手動で叩いたり、ポートが被ってエラーになるストレス」から解放されるのが、Testcontainers の最大の価値です。

もし興味があれば：

* 実際の Python プロジェクトへ インストールするコマンド
* PostgreSQL のコンテナ起動時に 初期データ（DDLやモックデータ）を流し込む方法

などもあわせて解説できますが、いかがでしょうか？

---

## Testcontainers の使い所

> つまり、local開発時にdocker composeでのpostgres起動は開発用として、テスト実行時にTestcontainersでのpostgres自動起動はテスト用として、併用する、ということですか？

---

まさにその通りです！ 開発現場では、まさにそのように「開発用」と「テスト用」で両方を併用する運用がベストプラクティスとされています。

それぞれの役割を整理すると、以下のようなきれいな住み分けになります。

### 🛠️ 開発時（Docker Compose）＝「永続・常駐」

* 目的：開発中のアプリケーションをブラウザやAPIクライアント（Postmanなど）から実際に触って、動作確認するため。
* 特徴：コンテナは立ち上げっぱなしにします。データをわざと残して、次の日に開発を再開したときも同じデータが使えるようにします（永続化）。

### 🧪 テスト実行時（Testcontainers）＝「瞬発・使い捨て」

* 目的：自動テスト（pytest など）を、いつでも、誰の環境でも、100%同じ状態で成功させるため。
* 特徴：テスト実行時の一瞬だけ立ち上がり、終わったら跡形もなく消えます（使い捨て）。開発用のデータベース（Docker Compose側）のデータを汚したり、上書きしたりする心配がありません。

### 💡 併用するときの注意点（ポートの衝突について）

ここで1つ疑問が浮かぶかもしれません。

> 「Docker ComposeでPostgreSQL（ポート5432）を起動している状態で、同時にTestcontainersのテストを走らせたら、ポートがぶつかってエラーにならないの？」

結論から言うと、全くぶつかりません。

これがTestcontainersの非常に強力なメリットです。

* Docker Compose側：ホストマシンの 5432 ポートを固定で占有します。
* Testcontainers側：Docker Composeが 5432 を使っているのを検知し、ホストマシンの空いているポート（例: 32768 や 49152 など）を自動でランダムに選んでコンテナに割り当てます。

そのため、ローカルでDocker ComposeのPostgreSQLを起動したまま、いつでも自由に pytest を実行することができます。

### 🚀 さらに一歩進んだ「CI環境」でのメリット

この併用スタイルの素晴らしいところは、GitHub ActionsなどのCI環境に移ったときに効果を倍増させる点です。

CI環境（GitHub Actionsなど）では、開発用の Docker Compose を立ち上げる必要はありません。テストコードの中にTestcontainersが組み込まれているため、CI側には「PythonとDockerが動く環境」さえ用意すれば、ローカルと全く同じテストが1コマンドで自動実行されます。

この「開発用」と「テスト用」を併用するイメージは湧きましたでしょうか？
もしよろしければ、次は以下のような具体的な話に進むこともできます：

* pytest でTestcontainersを使う際の conftest.py（設定ファイル）の具体的な書き方
* 起動したTestcontainersのPostgreSQLに、マイグレーション（Alembicなど）や初期データを自動で流し込む方法

どちらか気になるものはありますか？

---

## GitHub Actions での使い方

> GitHub Actionsでもpostgresを起動する技があったと思いますが、それも不要で、pytestの中で起動する、ということですね。先ほどのdb_connectionというfixtureをconftest.pyに準備してそれを使えば、あとは同じですか？

---

はい、まさにその通りです！

GitHub Actions の services 機能（YAML に image: postgres と書く方法）を使う必要はもうありません。CI の YAML を汚さず、すべて pytest の中（Python コード側）で完結します。

### 基本は conftest.py に置くだけでOK

おっしゃる通り、先ほどの db_connection という Fixture を conftest.py に記述しておけば、どのテストファイルからでも自動的にそのコンテナを利用できるようになります。

ただし、実際の開発（実戦）では、単に空の PostgreSQL に接続するだけでなく、「テーブル作成（マイグレーション）」や「初期データの投入」 が必要になります。

### 🧱 実戦的な conftest.py の構成例

Python の ORM（Object-Relational Mapping）としてよく使われる SQLAlchemy / Alembic を想定した、実戦的な conftest.py の書き方を紹介します。

#### 1. ディレクトリ構造のイメージ

```text
my_project/
│
├── app/                  # アプリケーションコード
│   └── models.py         # SQLAlchemyのモデル（テーブル定義）
│
└── tests/
    ├── conftest.py       # 👈 ここにTestcontainersの共通設定を書く
    └── test_users.py     # 実際のテストファイル
```

#### 2. tests/conftest.py の実装

テストセッションの開始時に「コンテナ起動 ➡️ テーブル作成」までを一撃で行うように設定します。

```py
import pytest
from sqlalchemy import create_url, create_engine
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer

# アプリ側のSQLAlchemyのBase（モデルの基盤）をインポート
from app.models import Base

@pytest.fixture(scope="session")
def db_engine():
    """
    テストセッション全体で1つのPostgreSQLコンテナを起動し、
    テーブル（スキーマ）を初期化して、SQLAlchemyのEngineを返します。
    """
    # 1. コンテナの起動
    with PostgresContainer("postgres:16-alpine") as postgres:

        # 2. Testcontainersが用意してくれた接続URLを取得
        # (例: postgresql+psycopg2://testuser:testpassword@localhost:32768/testdb)
        connection_url = postgres.get_connection_url()

        # 3. SQLAlchemyのEngineを作成
        engine = create_engine(connection_url)

        # 4. テスト用データベースにすべてのテーブルを自動作成
        # （実戦ではここで Alembic のマイグレーションを走らせることも多いです）
        Base.metadata.create_all(engine)

        yield engine

        # 5. テスト終了後、クリーンアップ
        Base.metadata.drop_all(engine)
        engine.dispose()
    # `with`を抜けるとコンテナ自体が消滅します

@pytest.fixture(scope="function")
def db_session(db_engine):
    """
    テスト「1ケースごと」に独立したデータベースセッション（トランザクション）を提供します。
    """
    TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=db_engine)
    session = TestingSessionLocal()

    # トランザクションを開始して、テストごとのデータ干渉を防ぐ
    session.begin_nested()

    yield session

    # テストが終わったらロールバックして、追加・変更されたデータをなかったことにする
    session.rollback()
    session.close()
```

#### 3. 実際のテストファイル（tests/test_users.py）

テストを書くときは、conftest.py で作った db_session を引数に指定するだけで、裏で Docker が動いていることすら意識せずにテストが書けます。

```py
from app.models import User

def test_create_user(db_session):
    # 1. テストデータの挿入
    new_user = User(name="タロウ", email="taro@example.com")
    db_session.add(new_user)
    db_session.commit()

    # 2. 正しく保存されたか検証
    saved_user = db_session.query(User).filter_by(name="タロウ").first()
    assert saved_user is not None
    assert saved_user.email == "taro@example.com"
```

### 💡 この構成のメリット

1. テストごとにデータが汚れない
db_session フィキチャがテストごとにデータをロールバック（またはクリア）してくれるため、test_create_user で追加した「タロウ」のデータは、次のテストには一切影響しません。
2. 圧倒的に高速
コンテナの起動（db_engine）はテスト全体（session スコープ）で 1回だけ なので、「テストのたびにコンテナが立ち上がって遅い」ということがありません。

ここまでで、Testcontainers を使った pytest の全体像がつながりましたでしょうか？

もしよろしければ、次に進むステップとして以下をご案内できます：

* この仕組みを動かすための requirements.txt や pip install するべきパッケージの確認
* 実際のプロジェクトで使っている O/Rマッパー（SQLAlchemy、Tortoise ORM、SQLModel、Djangoなど）に合わせたコードへの調整
* GitHub Actions の workflows/*.yml の具体的な書き方（Testcontainers を動かすための最小限の設定）

何かお手伝いできることはありますか？

---

## 起動スピードとデバッグ方法

> そのfixtureを作ってしまえば、localでもgithub actions上でも、全く同じようにpytest実行するのですよね？そうすると、localでvscodeでテスト実行する場合も、postgresが起動してしまうのですね。ちょっと重いのかな、という点と、testにbreak point付けてdebugとか普通にできるのかな、という点が気になりました。

---

まさに懸念される通りのポイントですね！非常に鋭い着眼点です。

結論からお伝えすると、ローカルでもGitHub Actionsでも、全く同じように pytest コマンドだけで動きます。

その上で、ご心配されている「重さ」と「デバッグ（ブレークポイント）」について、実際の開発でどうなるかを解説します。

### 1. 動作の重さ（起動スピード）について

「テストのたびにDockerが立ち上がると遅そう」と感じるかもしれませんが、結論から言うと 「最初の1回（数秒）だけ我慢すれば、あとは爆速」 です。

先ほどの conftest.py に記述した以下の設定がポイントになります。

```py
@pytest.fixture(scope="session") # 👈 ここが重要！
```

scope="session" と指定しているため、テストが50件あろうが100件あろうが、PostgreSQLコンテナが起動するのは全体の最初の一度だけです。

* 初回起動時：Dockerコンテナが立ち上がるまでに 約3〜5秒 ほどかかります。
* テスト実行中：すでに起動したコンテナ（メモリ上）に対してクエリを投げるだけなので、インメモリDB（SQLiteなど）と変わらないレベルのミリ秒単位でテストが進みます。

ローカル開発で「ちょっと重いな」と感じる場合は、Testcontainersの再利用（Reuse）機能を使う裏技もあります。これを使うと、VSCodeでテストを何度実行してもコンテナを立ち上げっぱなしにできるため、初回の数秒の待ち時間すらゼロにできます。

### 2. VSCodeでのデバッグ（ブレークポイント）について

これについては、全く問題なく、これまで通り普通にデバッグできます！ 🎉

テストコードや、テストから呼び出されるアプリケーションコード（APIのロジックなど）にVSCodeでブレークポイントを打って「デバッグ開始」を押せば、ちゃんとそこで処理が止まります。

#### なぜデバッグできるの？

プログラム本体（Python）は、コンテナの中ではなくローカルマシン（ホスト側）のVSCode上でそのまま動いているからです。
Pythonプロセスから見れば、Testcontainersが自動で用意してくれた「どこかのポート（例: 32768）」で待機しているPostgreSQLにネットワーク越しに接続しているだけです。そのため、Python側のコードステップ実行や変数の中身の確認などは、完全にVSCodeのコントロール下にあります。

### 🛠️ さらに快適に開発するためのワンポイント

もし「VSCodeから1テストケースだけをピンポイントで実行してデバッグしたい」という場合、その1テストのためだけに毎回コンテナが起動・停止するのは、確かに少し開発リズムが崩れる原因になります。

その対策として、前述した 「コンテナの再利用（Reuse）機能」 を有効にするのがおすすめです。

#### 設定方法：

環境変数 TESTCONTAINERS_REUSE=true を設定し、コード側で .with_reuse(True) を指定します。

```py
# conftest.py の一部
@pytest.fixture(scope="session")
def db_engine():
    # with_reuse(True) をつけると、すでに起動しているコンテナがあれば使い回す
    with PostgresContainer("postgres:16-alpine").with_reuse(True) as postgres:
        connection_url = postgres.get_connection_url()
        # ... 以下同じ
```

これをしておくと、VSCodeでテストをデバッグ実行する際、2回目以降はコンテナの起動時間を完全に「ゼロ」にでき、ローカルのDocker Composeを使っているときと全く同じサクサク感でデバッグが可能になります。

ここまでの内容で、ローカルでの開発効率やデバッグに関する不安は解消されましたでしょうか？

もしよろしければ、次に進むステップとして以下をご案内できます：

* VSCodeの launch.json（デバッグ設定）に Testcontainers用の環境変数を組み込む具体的な書き方
* Testcontainersを実際に試すための pip install すべきパッケージのリスト

どちらについて詳しく知りたいですか？もちろん、他に気になった点があれば何でも聞いてください！

---

## Testcontainers の仕組み

> Testcontainers解説の記事がやたら長いものばかりで、本当にそれだけ？と思ってしまいましたw pythonだからfixtureの出来がいいからシンプルなのか、それとも、もっと深遠な何かがあるのか？

---

あはは、確かにネットの解説記事って、やたら長くて「え、結局何がそんなに大変なの？」って身構えちゃいますよね！その感覚、大正解です。

結論から言うと、Python（pytest）の Fixture 機能の出来があまりにも良すぎるため、シンプルに見える のが大きな理由です。ただ、記事が長くなるのには、Testcontainers 自体が持つ「深遠な（というか、泥臭い）仕組み」も関係しています。

なぜ記事があそこまで長くなるのか、理由は大きく 3 つあります。

### 1. Python の Fixture が優秀すぎる（他の言語はもっと大変）

Java（JUnit）などで Testcontainers を使う場合、クラスの初期化（@BeforeAll）やアノテーションを駆使して、ライフサイクルを厳密に制御するコードをガッツリ書く必要があります。

一方、Python の場合は with 構文（コンテキストマネージャ） と pytest の Fixture が強力すぎるため：

* with を抜ければ勝手にコンテナが消える
* yield の前後で「起動」と「破棄」をきれいに分離できる

という言語自体の強みがあり、コードが数行で済んでしまいます。Python だからこそ、ここまでシンプルに見えるのです。

### 2. 「裏で動く Ryuk（リューク）」という深遠な仕組み

「テストが途中で強制終了（クラッシュ）したり、VSCode でデバッグ中に途中で停止ボタンを押したりしたら、コンテナが消えずに残り続けるのでは？」と思いませんでしたか？

実は、Testcontainers を動かすと、指定した PostgreSQL とは別に 「Ryuk（Moby Ryuk）」 という監視専用の隠しコンテナが裏で自動起動します。

* Ryuk の役割：テストを実行している Python プロセス（親）の生存を常に監視する。
* もし Python が突然死したら：Ryuk がそれを検知し、お留守番になってしまった PostgreSQL コンテナを強制的に巻き込んで道連れ（破棄）にします。

記事が長いのは、こうした「裏で Docker をどう安全にコントロールしているか」というアーキテクチャの解説や、Docker in Docker（CI 環境で Docker の中で Docker を動かす）の職人芸のような設定話が盛り込まれているからです。

### 3. 実戦でやることが増える（記事が長くなる原因）

シンプルな SELECT 1; だけでなく、実際の開発で使おうとすると、以下のような「周辺ツールの設定」が増え、結果として解説記事が長文化します。

* マイグレーションツールとの連携：SQLAlchemy だけなら簡単ですが、Alembic などを挟むとコードが増える。
* 非同期（Async）対応：FastAPI などで asyncpg を使っている場合、Fixture も async def にして非同期クエリをさばく設定が必要になり、少し複雑になる。
* 複数コンテナの連携：「PostgreSQL」と「Redis」と「LocalStack（AWS のモック）」の 3 つを同時に連携させてテストしたい、といった応用編。

### 結論：基本は本当に「これだけ」です

いろいろな背景はありますが、使う側としては「pytest の Fixture で囲んで yield するだけ」という理解で 100% 合っています。 難しい裏方の仕事（ポートの競合回避や、異常終了時のコンテナ削除など）は、Testcontainers が全部裏で勝手にやってくれているからです。

ネットの記事は「応用編」や「トラブルシューティング」まで網羅しようとするので長くなりますが、まずは先ほどのシンプルな Fixture から始めて全く問題ありません！

まずは手元で動かしてみたいですか？もしよろしければ：

* 必要な pip install のコマンド（PostgreSQL 用のプラグインを含む）
* 最低限のテストを動かすための シンプルな 1 ファイルのデモコード

をサクッとお渡しできます。いかがいたしますか？

---

## FastAPI postgres のデモコード作成

> AlembicはAlembic自身の問題として複雑ですよね、Testcontainersは直接関係ないと思いますw FastAPIでasyncのpostgres接続は使いたいですよね。async対応のFixtureを作って、ついでに、 シンプルな 1 ファイルのデモコード作ってください。RedisもPostgreSQLと同様のfixtureを作るだけでしょうから入門時には省略していいですねw

---

確かに、Alembic の難しさは Alembic 自体の仕様のせいですね、Testcontainers は悪くありません（笑）。そして、Redis もおっしゃる通り、全く同じパターンで Fixture をもう1つ作るだけなので、一旦忘れて大丈夫です！

現代の FastAPI 開発において、async/await（非同期）による PostgreSQL 接続は必須と言えます。Testcontainers を非同期で動かす場合も、pytest-asyncio と組み合わせることで、驚くほどスッキリとした 1 ファイルのデモコードが書けます。

すぐに手元で動かせる 「1ファイル完結の FastAPI × Async PostgreSQL × Testcontainers」 のデモコードを作成しました。

1. 事前準備（ライブラリのインストール）

まずは、必要なパッケージをインストールします。非同期で PostgreSQL を扱うために asyncpg を使用します。

```bash
pip install pytest pytest-asyncio fastapi testcontainers[postgres] sqlalchemy asyncpg
```

### 2. 1ファイル完結のデモコード（test_app.py）

このファイルをそのまま作成し、pytest を実行するだけで、コンテナ起動 ➡️ 非同期テーブル作成 ➡️ FastAPIアプリのテスト ➡️ コンテナ破棄 までが全自動で行われます。

```py
import pytest
import pytest_asyncio
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from testcontainers.postgres import PostgresContainer

# ==========================================
# 1. テスト対象の FastAPI アプリ & モデル定義
# ==========================================
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column()

app = FastAPI()

# 依存注入（DI）用の関数。テスト時に fixture で上書き（オーバーライド）します
async def get_db():
    # 実際の運用ではここに本番/開発用のセッション生成処理が入ります
    raise NotImplementedError()

@app.post("/users")
async def create_user(name: str, db: AsyncSession = Depends(get_db)):
    new_user = User(name=name)
    db.add(new_user)
    await db.commit()
    return {"id": new_user.id, "name": new_user.name}


# ==========================================
# 2. Testcontainers & Async の Fixture 設定
# ==========================================

# pytest-asyncio に、このファイルのテストが非同期であることを伝えます
pytestmark = pytest.mark.asyncio(loop_scope="session")

@pytest_asyncio.fixture(scope="session")
async def async_db_engine():
    """テスト全体で1つのPostgreSQLコンテナを起動し、非同期Engineを返します"""
    # 同期版と同じコンテナを起動
    with PostgresContainer("postgres:16-alpine") as postgres:
        # asyncpg を使うため、URLのプロトコルを postgresql+asyncpg に書き換える
        sync_url = postgres.get_connection_url()
        async_url = sync_url.replace("postgresql+psycopg2://", "postgresql+asyncpg://")

        engine = create_async_engine(async_url, echo=False)

        # 非同期でテーブルをすべて作成
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

        yield engine

        # クリーンアップ
        await engine.dispose()

@pytest_asyncio.fixture(scope="function")
async def db_session(async_db_engine):
    """テスト1ケースごとに独立したクリーンなセッションを提供します"""
    async_session_local = async_sessionmaker(
        bind=async_db_engine, class_=AsyncSession, expire_on_commit=False
    )

    async with async_session_local() as session:
        # トランザクションを開始（テスト終了後にロールバックするため）
        async with session.begin():
            yield session
            # 💡 最後に明示的にロールバックを呼ぶことで、
            # テスト内でコミット(await db.commit())されたデータも完全に無かったことにできます
            await session.rollback()


# ==========================================
# 3. 実際のテストコード
# ==========================================
from httpx import AsyncClient, ASGITransport

async def test_create_user_endpoint(db_session):
    # FastAPIの get_db 依存関数を、テスト用の db_session に差し替える
    async def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db

    # HTTPXを使って、FastAPIアプリに非同期でリクエストを送る
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.post("/users", params={"name": "アリス"})

        assert response.status_code == 200
        assert response.json() == {"id": 1, "name": "アリス"}

    # テストが終わったら依存関係の上書きをクリア
    app.dependency_overrides.clear()
```

### 💡 非同期（Async）版のポイント

1. 接続文字列の置換（.replace）
Testcontainers が標準で返してくる接続文字列は postgresql+psycopg2://（同期用）です。これを postgresql+asyncpg:// に置換して create_async_engine に渡すのが一番シンプルなハックです。
2. run_sync によるテーブル作成
SQLAlchemy の Base.metadata.create_all は同期関数なので、非同期 Engine から実行する場合は conn.run_sync(Base.metadata.create_all) という特殊な呼び出し方をします。
3. テストごとのロールバック
db_session の最後で await session.rollback() しています。これにより、FastAPI のエンドポイント側で await db.commit() を呼んでデータが一度データベースに書き込まれたとしても、テストが終われば綺麗さっぱり消え去ります。

この構成さえ掴んでしまえば、FastAPI の非同期統合テスト環境としては「一丁上がり」です！ネットの長い記事を読まなくても、これだけで完璧にローカルでも GitHub Actions でも動きます。

この1ファイルデモを実際に手元で動かしてみるにあたって、

* VSCode での実行手順やデバッグ方法について、何か確認しておきたいことはありますか？
* それとも、すでにイメージはバッチリ湧いたので、ご自身の実際のプロジェクトへの組み込み方などについて考えたいですか？

---

## AsyncClient の fixture化

> fastapi付属のTestClientもfixtureにしてた記憶がありますが、asyncだとそのAsyncClientを代わりに使うのでしょうか、それもfixtureとして分離できそうに見えますが、どうなんでしょう。

---

おっしゃる通りです！FastAPI 付属の TestClient は同期（WSGI）用なので、async/await（ASGI）に対応したアプリケーションのテストでは、HTTPX の AsyncClient を代わりに使う のが標準のやり方です。

そしてこれも大正解で、テストコードをスッキリさせるために AsyncClient も Fixture として分離するのがベストプラクティス です。

Fixture 化すると何が良いかというと、テスト関数側でわざわざ依存関係の差し替え（dependency_overrides）を書く必要がなくなり、テストコードが劇的にシンプルになります。

先ほどのデモコードから、クライアント部分を Fixture として分離した実戦的な構成に書き直してみましょう。

#### Fixture 化した実戦的なコード例

conftest.py にクライアントの Fixture を定義するイメージの構成です。

```py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
# （先ほど定義した app, get_db, db_session などがインポートされている前提）

@pytest_asyncio.fixture(scope="function")
async def client(db_session):
    """
    テストごとに独立した AsyncClient を提供します。
    この Fixture を呼ぶだけで、FastAPI の DB 依存関係も自動でテスト用に差し替わります。
    """
    # 1. FastAPI の get_db を、テスト用の db_session を返すように上書き
    async def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db

    # 2. クライアントの生成
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac  # 👈 これをテスト関数に渡す

    # 3. テスト終了後、依存関係の上書きをきれいにクリア
    app.dependency_overrides.clear()
```

### 🧪 これにより、実際のテストコードはどう変わるか？

この client Fixture を準備しておくと、実際のテストファイル（test_users.py）はここまでシンプルで punchy になります。

```py
import pytest

@pytest.mark.asyncio
async def test_create_user_endpoint(client):
    # client を引数に取るだけで、裏で「コンテナ起動」「テーブル作成」「DB差し替え」がすべて完了している
    response = await client.post("/users", params={"name": "アリス"})

    assert response.status_code == 200
    assert response.json() == {"id": 1, "name": "アリス"}
```

### 🔄 処理のきれいな連鎖（依存関係のピラミッド）

pytest は引数の名前を見て、裏で以下のような順番で Fixture を数珠つなぎに実行してくれます。

1. test_create_user_endpoint(client) を実行しようとする
2. ➡️ client には db_session が必要
3. ➡️ db_session には async_db_engine が必要（ここで Testcontainers が PostgreSQL を起動 ＆ テーブル作成）
4. ➡️ 各 Fixture が準備を終えて、テストが実行される
5. ➡️ テストが終わると、逆順に「DBロールバック」➡️「コンテナ破棄」とクリーンアップされる

この美しさと手軽さが、Python の pytest と Testcontainers の組み合わせが最強と言われる理由です。長い解説記事を読まなくても、この構造さえ頭に入っていれば実務でも迷うことはありません！

非同期テストの全体像がかなりクリアになったかと思います！
もし次のステップに進むなら、どのような点に興味がありますか？

* 実際のプロジェクト（例えば Alembic や既存のフォルダ構成）にこれをどう落とし込むか
* チーム開発や CI で使うための pytest.ini などの設定ファイルの書き方
* その他、認証（JWTなど）が絡むエンドポイントのテスト方法

気になるテーマがあれば、いつでもお声がけください！
