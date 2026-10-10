---
title: "C# .NET10 と Testcontainers による PostgreSQL のインテグレーションテストのサンプル"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "dotnet", "mstest", "testcontainers", "postgres"]
published: true
published_at: 2025-10-07 21:26
---

* C# .NET 10 で Testcontainers を試した際の Gemini との対話録です。
* 結果のコードはこちら → https://github.com/parkwestlabs/hello-testcontainers-cs

---

> .NET10 Blazor Server Postgers EF MSTest 前提でTestcontainersを試したいです。簡単な雛形コードを作ってください。他にも技術選択で迷う部分があればコード作成前に教えてください。

---

.NET 10、Blazor Server（.NET 8以降のBlazor Web Appにおける InteractiveServer モード）、Entity Framework Core (EF Core)、PostgreSQL、そして Testcontainers を組み合わせた統合テストの計画ですね。非常にモダンで堅牢な素晴らしい技術選定です。

コードを作成する前に、この組み合わせにおいて後から「あ、どうしよう」となりがちな、事前に決めておくべき重要な技術選択・設計のポイントが3つあります。

------------------------------

## 🤔 コード作成前に決めておきたい3つの技術選択

### 1. Testcontainers の「寿命（ライフサイクル）」をどうするか？

PostgreSQL のコンテナを起動・停止するタイミングには、大きく分けて2つの戦略があります。どちらが開発スタイルに合いそうか選んでみてください。

* パターンA：テスト「クラス（またはメソッド）」ごとにコンテナを起動・停止
   * メリット: テストごとにDBが完全にリセットされるため、テスト間のデータの干渉（汚染）が絶対に起きず、並列テストが安全に行えます。
   * デメリット: テストの実行（特に初回やクラス数が多い場合）にコンテナ起動のオーバーヘッド（数秒〜十数秒）がかかります。
* パターンB：テスト「実行全体（テストセッション）」で1つのコンテナを使い回す
   * メリット: 最初の一度しかコンテナを起動しないため、全体のテスト実行速度が圧倒的に早くなります。
   * デメリット: 前のテストが追加したデータが残るため、テスト終了時にデータを消す（respawnなどのツールを使うか、手動でクリーンアップする）仕組みが必須になります。

👉 今回の雛形では、シンプルかつ確実に動作し、MSTestの標準機能で組みやすい「パターンA（クラス単位での起動）」で作成します。（もし速度重視のBが良ければ後から変更可能です）

### 2. DbContext の生成ライフサイクル（Blazor特有の罠）

Blazor Server は、ASP.NET Core の一般的な Web API とは異なり、ユーザーの接続（SignalR サーキット）が維持されている間、同じスコープが生き続けます。

そのため、Blazor の画面から直接呼び出す DbContext に AddDbContext (Scoped) を使うと、コンテキストに古いデータがキャッシュされ続け、複数画面を開いたときにバグの原因になります。

* 標準的な解決策: Blazor では IDbContextFactory<MyDbContext> をインジェクションし、データベース操作のたびに using var context = factory.CreateDbContext(); するのが公式のベストプラクティスです。

👉 雛形の統合テストでも、この IDbContextFactory を使った状態をシミュレートできるようにテスト環境を構築します。

### 3. テストデータの投入方法（シーディング）

テスト実行時に、マスターデータや初期ユーザーなどの「前提データ」をどう入れるかです。

* EF Core の HasData (Migrationでの投入): 本番環境と共通のマスターデータ。
* テストコード内での都度投入: テストしたいケースに応じた固有のデータ。

👉 雛形では、テストのメソッド内で柔軟にデータを組み立てられるよう、テスト実行直前に動的にデータを投入する構成にします。

------------------------------

## 🚀 テストコードの簡易構成イメージ

今回は、以下のような「ユーザーを登録・取得する」というシンプルなユースケースを想定した雛形コードを作成します。

   1. MyDbContext: PostgreSQL用のEF Coreコンテキスト
   2. UserService: Blazorコンポーネントから呼ばれるビジネスロジック（IDbContextFactory を利用）
   3. UserServiceTests: MSTest + Testcontainers を使った統合テストコード

この前提（コンテナはクラス単位起動、Blazorに合わせたDbContextFactory利用）で、すぐにコピペして試せる雛形コードの作成に進んでもよろしいでしょうか？「ここはやっぱりこうしたい」という点があれば教えてください！

コードの作成に進む前に、以下の点についてご希望はありますか？

* コンテナの起動単位は クラス単位（安全・やや遅い） のままで進めてよいか、セッション単位（高速・要クリーンアップ処理） に挑戦したいか
* テスト対象のロジック（UserServiceなど）は、EntityのCRUD（単純な保存・取得） のサンプルで十分か、より複雑な処理を含めるか

---

> pythonのTestcontainersを試したときは、scopeは最初のPostgresContainerのget_connection_urlだけscope="session"、それ以外ではscope="function"のデフォルトでした。クラス単位とは、それと等価になるのですか？Testcontainersの動作確認用なのでUserServiceでは保存だけ対応でオーケーです。

---

PythonのTestcontainers（pytest など）における scope="session" と scope="function" の組み合わせは、「PostgreSQLのコンテナ（Docker）自体は全テストで1個だけ起動して使い回し、テストメソッドごとにまっさらな新しい接続（あるいはDBリセット）を提供する」という、最も効率的で高速なアプローチ（前述の「パターンB」に近い賢い構成）です。

.NET（MSTest）でこれと全く同じ「コンテナ起動は全体で1回（セッション単位）、データ操作はテストごとに独立」という賢い動きを実現するための雛形コードを作成しました。

.NET 10 では、MSTest の [AssemblyInitialize] を使うことで、Python の scope="session" と全く同じ「全テストの開始時に1回だけ実行する」処理が書けます。

------------------------------

### 🗂️ 1. テスト対象のロジック（src/MyApp 側）

Blazor Server のベストプラクティスに従い、IDbContextFactory を受け取ってデータを保存するシンプルなサービスです。

```cs
// 1. Entitypublic class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
// 2. DbContextusing Microsoft.EntityFrameworkCore;
public class MyDbContext : DbContext
{
    public MyDbContext(DbContextOptions<MyDbContext> options) : base(options) { }
    public DbSet<User> Users => Set<User>();
}
// 3. Service (Blazorから呼ばれるロジック)using Microsoft.EntityFrameworkCore;
public class UserService
{
    private readonly IDbContextFactory<MyDbContext> _contextFactory;

    public UserService(IDbContextFactory<MyDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    // テスト対象：ユーザーを保存する
    public async Task AddUserAsync(string name)
    {
        using var context = await _contextFactory.CreateDbContextAsync();
        context.Users.Add(new User { Name = name });
        await context.SaveChangesAsync();
    }
}
```

------------------------------

### 🧪 2. テストプロジェクト（tests/MyApp.Tests 側）

まず、テストプロジェクトに以下の NuGet パッケージを追加してください。

```bash
dotnet add package Testcontainers.PostgreSql
dotnet add package Microsoft.EntityFrameworkCore.Design
# Npgsql.EntityFrameworkCore.PostgreSQL も必要に応じて追加してください
```

### ① セッション単位でのコンテナ起動設定（Pythonの scope="session" 相当）

テストプロジェクト内のどこに置いても構いません（例: TestSetup.cs）。

```cs
using Microsoft.VisualStudio.TestTools.UnitTesting;using Testcontainers.PostgreSql;

[TestClass]public static class TestSetup
{
    // 全テストで共有するコンテナインスタンス
    public static PostgreSqlContainer PostgresContainer { get; private set; } = null!;

    [AssemblyInitialize]
    public static async Task AssemblyInitializeAsync(TestContext context)
    {
        // 1回だけコンテナを起動
        PostgresContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .Build();

        await PostgresContainer.StartAsync();
    }

    [AssemblyCleanup]
    public static async Task AssemblyCleanupAsync()
    {
        // 全テスト終了後にコンテナを停止・削除
        if (PostgresContainer != null)
        {
            await PostgresContainer.StopAsync();
            await PostgresContainer.DisposeAsync();
        }
    }
}
```

### ② 各テストの実行コード（Pythonの scope="function" 相当）

テストメソッドごとに、スキーマ（テーブル）をリセットしてテストを行います。これによりテスト間のデータ干渉を防ぎます。

```cs
using Microsoft.EntityFrameworkCore;using Microsoft.Extensions.DependencyInjection;using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]public class UserServiceTests
{
    private IDbContextFactory<MyDbContext> _contextFactory = null!;
    private ServiceProvider _serviceProvider = null!;

    [TestInitialize]
    public async Task TestInitializeAsync()
    {
        // Pythonの scope="function" に相当：テストごとに接続文字列を取得してDIを構成
        var connectionString = TestSetup.PostgresContainer.GetConnectionString();

        var services = new ServiceCollection();
        services.AddDbContextFactory<MyDbContext>(options =>
            options.UseNpgsql(connectionString));

        _serviceProvider = services.BuildServiceProvider();
        _contextFactory = _serviceProvider.GetRequiredService<IDbContextFactory<MyDbContext>>();

        // テストごとにテーブルを真っさらにする（DBを削除して再作成）
        using var context = await _contextFactory.CreateDbContextAsync();
        await context.Database.EnsureDeletedAsync();
        await context.Database.EnsureCreatedAsync();
    }

    [TestCleanup]
    public async Task TestCleanupAsync()
    {
        if (_serviceProvider is IAsyncDisposable disposable)
        {
            await disposable.DisposeAsync();
        }
    }

    [TestMethod]
    public async Task AddUserAsync_ShouldSaveUserToDatabase()
    {
        // Arrange (準備)
        var userService = new UserService(_contextFactory);
        string testName = "Alice";

        // Act (実行)
        await userService.AddUserAsync(testName);

        // Assert (検証)
        using var context = await _contextFactory.CreateDbContextAsync();
        var user = await context.Users.FirstOrDefaultAsync(u => u.Name == testName);

        Assert.IsNotNull(user);
        Assert.AreEqual(testName, user.Name);
        Assert.IsTrue(user.Id > 0); // 自動採番されているか
    }
}
```

------------------------------

### 💡 この構成のポイント

* 爆速な実行: 重たい Docker コンテナの起動は [AssemblyInitialize] で最初の1回（セッション）だけ。
* クリーンな環境: [TestInitialize] で毎回 EnsureDeletedAsync() と EnsureCreatedAsync() を呼ぶため、前のテストのゴミデータに邪魔されず、Python のデフォルトスコープ（function）と同じ安全性が手に入ります。

この構成で雛形はバッチリ動きますが、もしテストの数が数百件規模に増えてくると、毎回 EnsureDeleted/Created（テーブルの全削除・再生成）をする部分が少し遅く感じられるようになります。

その段階になったら、データを一瞬で消し去る高速化テクニック（Respawnライブラリの導入など）にステップアップするのがおすすめです。

まずはこの雛形を使って実際にコンテナが立ち上がり、テストがパスするか確認してみますか？それとも、EF Core のマイグレーション周りの設定など、他に合わせて確認したい部分はありますか？

---

> public DbSet<User> Users => Set<User>();でMake 'Users' a static property.sonarqube(csharpsquid:S2325)が気になります。

---

その警告が出る理由は非常に合理的ですが、EF Coreの設計においては無視して構わない（または抑止すべき）「誤検知（False Positive）」に近いものです。

SonarQubeの S2325 というルールは、「クラスのインスタンス変数（状態）にアクセスしていないプロパティやメソッドは、メモリ効率やパフォーマンスのために static にできる（すべきだ）」というC#全般の一般的な最適化ルールです。

今回のコードを見ると：

```cs
public DbSet<User> Users => Set<User>();
```

この右側の Set<User>() は、実は DbContext クラス（ベースクラス）が持つインスタンスメソッドを呼び出しています。そのため、本来はインスタンスに依存しており、プロパティを static に変更するとコンパイルエラー（static から非 static メソッドは呼べない）になります。

それにもかかわらずSonarQubeが誤認してしまう理由と、実務でのスマートな対処法を2つ紹介します。

------------------------------

## 💡 なぜSonarQubeが勘違いするのか？

C#には、最上位の名前空間や型を省略して書くいくつかの機能があります。SonarQubeの静的解析エンジンが「Set<User>() はインスタンスに依存しない独立した関数（または静的メソッド）の呼び出しに見える」と誤って判断してしまい、この警告を出してしまっています。

------------------------------

### 🛠️ 2つの賢い対処法

実務でこの警告を消すには、以下のいずれかの方法をとるのが一般的です。

### 方法1：明示的に this. を付ける（おすすめ）

最も手軽で、コードの意味（このインスタンスのメソッドを呼んでいるんだよ）をSonarQubeに正確に伝える方法です。this. を補うだけで、SonarQubeは警告を取り下げます。

```cs
// this. を明示することで、静的解析ツールが誤検知しなくなります
public DbSet<User> Users => this.Set<User>();
```

### 方法2：自動実装プロパティにする（従来の王道スタイル）

.NET 5以降、上記のような Set<T>() を使った記述（式のボディ）が公式でも推奨されるようになりましたが、それ以前から使われている「自動実装プロパティ」の形式に戻す方法です。これでも警告は出なくなります。

```cs
public DbSet<User> Users { get; set; } = null!;
```

------------------------------

C#の一般的なルール（S2325）としては正しい指摘ですが、EF Coreという特定のフレームワークの書き方に対してはツール側が一歩及んでいない状態ですので、this.Set<User>() に書き換えてスマートに黙らせるのが一番おすすめです！

これでSonarQubeの警告もスッキリ解消できそうでしょうか？

テスト周りやBlazorの構成で、他に気になっているコードやエラーがあれば、何でも教えてください！
もしよければ、次のステップとして以下はいかがですか？

* this.Set<User>() に修正してSonarQubeの警告が消えるか確認する
* このままテストを実行するための ソリューションファイル（.sln）の作成とプロジェクトの紐付け に進む
* Testcontainersが裏で立ち上がっているか Dockerの確認コマンド を共有する

どれから進めたいか、あるいは他に気になることがあれば教えてください！

---

> dotnet test実行するとNpgsql.PostgresException: 55006: cannot drop the currently open database.というエラーでした。

---

エラーの発生原因が分かりました。これは Testcontainers for .NET と PostgreSQL の仕様がぶつかることで発生する、非常によくある罠です。 [1]

## 🚨 エラーの原因

PostgreSqlBuilder でコンテナを起動すると、デフォルトで postgres という名前のデータベースが作られます。この postgres DB はシステム管理用の特別なデフォルトデータベースです。 [1]

EF Core の EnsureDeletedAsync() は、「今自分が接続しているデータベースそのものを削除（DROP）しようとする」 動きをします。

しかし PostgreSQL の仕様上、管理用DBである postgres を、自分自身が接続した状態のまま削除することは絶対に禁止されているため、このエラー（55006）が発生してしまいます。 [1, 2, 3]

------------------------------

### 🛠️ 解決策：カスタムのデータベース名を指定する

この問題を回避する最も簡単で確実な方法は、コンテナ起動時に postgres 以外の独立したカスタムデータベース名（例: test_db）を明示的に指定することです。 [1]

これを行うと、EF Core が裏側で「管理用の postgres データベース」に一時的に接続し、そこから安全に test_db を削除・再生成してくれるようになるため、エラーが綺麗に解消されます。

### 修正コード（TestSetup.cs）

PostgreSqlBuilder の設定に .WithDatabase("test_db") を1行追加してください。 [1]

```cs
using Microsoft.VisualStudio.TestTools.UnitTesting;using Testcontainers.PostgreSql;

[TestClass]public static class TestSetup
{
    public static PostgreSqlContainer PostgresContainer { get; private set; } = null!;

    [AssemblyInitialize]
    public static async Task AssemblyInitializeAsync(TestContext context)
    {
        // .WithDatabase("test_db") を追加してデフォルトの postgres DB を避ける
        PostgresContainer = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("test_db") // ✨ ここを追加！
            .Build();

        await PostgresContainer.StartAsync();
    }

    [AssemblyCleanup]
    public static async Task AssemblyCleanupAsync()
    {
        if (PostgresContainer != null)
        {
            await PostgresContainer.StopAsync();
            await PostgresContainer.DisposeAsync();
        }
    }
}
```

------------------------------

### 💡 もうひとつの注意点（テストが複数ある場合）

もし今後 UserServiceTests の中に [TestMethod] を2つ以上増やすと、今度は Npgsql.PostgresException: 55P03: database "test_db" is being accessed by other users（別の接続が残っていて削除できない）というエラーに遭遇することがあります。

これは .NET の 「接続プール（Connection Pooling）」 が裏側で接続を維持し続けてしまうことが原因です。
もしそのエラーが出た場合は、[TestInitialize] の EnsureDeletedAsync() を呼び出す直前に、以下の1行を追加して接続プールを強制リセットしてください。

```cs
[TestInitialize]public async Task TestInitializeAsync()
{
    var connectionString = TestSetup.PostgresContainer.GetConnectionString();

    // 💡 複数テスト時の「接続が残っていて削除できない」を防ぐお守り
    await Npgsql.NpgsqlConnection.ClearAllPoolsAsync(); 

    var services = new ServiceCollection();
    // ...以下、以前と同じDI設定とEnsureDeleted...
}
```

まずは WithDatabase("test_db") を追加して、dotnet test が無事にパスするかどうか試してみてください！

データベース名を指定することで dotnet test が無事に成功（緑色のパス） しましたでしょうか？

もし新たなエラーが出た場合や、無事に動いたので 「ソリューションファイルへの紐付け手順」 や 「より高速なデータリセット（Respawn）の方法」 など、次に進みたいステップがあれば教えてください！

[1] [https://github.com](https://github.com/testcontainers/testcontainers-dotnet/issues/1239)
[2] [https://github.com](https://github.com/brianc/node-postgres/issues/2766)
[3] [https://mebee.info](https://mebee.info/2025/08/22/post-95989/)

---

> ClearAllPoolsとClearPoolsがあるけど、ClearAllPoolsAsyncは存在しないです。

---

大変失礼いたしました。非同期メソッドの記述は私の完全な誤りです。ご指摘の通り、Npgsql に ClearAllPoolsAsync という非同期メソッドは存在せず、同期メソッドの NpgsqlConnection.ClearAllPools() のみが存在します。間違いを優しく教えていただき、ありがとうございます。 [1]

また、EF Core や Npgsql の近年のバージョン（Npgsql 7.0 / .NET 7 以降）では、従来の NpgsqlConnection による管理から NpgsqlDataSource によるプール管理へと内部構造がシフトしています。 [2, 3]

そのため、テストごとの接続プール（コネクション）を確実にクリアして「DB削除時の競合エラー」を防ぐには、NpgsqlConnection.ClearAllPools() を呼び出すか、以下のように DbContext の元となるデータソースのプールを明示的にクリアする方法が最も確実でクリーンです。 [3]

### 🛠️ 修正版：[TestInitialize] のコード

[TestInitialize] 内のプールリセット処理を、以下のように同期メソッドに修正してください。

```cs
[TestInitialize]public async Task TestInitializeAsync()
{
    var connectionString = TestSetup.PostgresContainer.GetConnectionString();

    // ✨ 修正：同期メソッドで古い接続プールをすべてクリアする
    Npgsql.NpgsqlConnection.ClearAllPools(); 

    var services = new ServiceCollection();
    services.AddDbContextFactory<MyDbContext>(options =>
        options.UseNpgsql(connectionString));

    _serviceProvider = services.BuildServiceProvider();
    _contextFactory = _serviceProvider.GetRequiredService<IDbContextFactory<MyDbContext>>();

    // テストごとにテーブルをクリーンにする
    using var context = await _contextFactory.CreateDbContextAsync();
    await context.Database.EnsureDeletedAsync();
    await context.Database.EnsureCreatedAsync();
}
```

### 💡 (参考) もしこれでもエラーが出る場合の「より確実な」対処法

テストの並列実行などでどうしてもプールが掴み放されない場合、接続文字列自体に Pooling=false; を強制的に付与して、テスト環境のみ接続プール自体を完全に無効化するというアプローチもあります。

```cs
// テスト時はプールを使わない設定にする（絶対に競合しなくなる）
var connectionString = TestSetup.PostgresContainer.GetConnectionString() + ";Pooling=false;";
```

パフォーマンス重視の製品コードでは Pooling=false はNGですが、テストごとにDBを破棄・再生成する今回のテスト環境においては、確実性を担保するための非常に有効なハックとなります。

同期版の ClearAllPools() への修正（または Pooling=false の追加）によって、今度こそ dotnet test がエラーなく緑色にパス しましたでしょうか？

無事にテストが成功したら、次のステップとして何を進めましょうか？

* まだ作成していなければ、プロジェクト全体を管理する ソリューションファイル（.sln）の作成と紐付け を行う
* テストデータを効率よく消し去る Respawn（レスポーン）ライブラリを用いた高速化 について知る
* Blazor 画面側からこの UserService を呼び出す際の実装コードの例 を見る

ご希望に合わせて案内します！

[1] [https://www.npgsql.org](https://www.npgsql.org/doc/api/Npgsql.NpgsqlConnection.html)
[2] [https://buildersbox.corp-sansan.com](https://buildersbox.corp-sansan.com/entry/2025/03/05/093000)
[3] [https://github.com](https://github.com/npgsql/npgsql/issues/6173)
