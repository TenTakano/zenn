---
title: "PhoenixでArangodbを使うときのecto周りの設定"
emoji: "🖋"
type: "tech"
topics: ["phoenix", "arangodb"]
published: false
---

# はじめに

趣味の開発でelixirのPhoenixライブラリからArangoDBを使おうとしたところ、ectoの設定関係で盛大に躓いてしまいました。解決こそできたものの、そのものズバリな記事が見当たらなかったので防備録として残しておきます。

# ことの発端

趣味で開発をしていたところ、スキーマレスなDBが欲しいなぁということがありました。スキーマレスというと仕事で使っているのもあってとりあえずmongo使うかって思ったのですが、そういやmongoってライセンス周りややこしくなったんだよなぁというのを思い出していきなりモチベーションが低下してしまいました(´・ω・｀)
まぁバージョン5系のmongoを使えばよいかとなんとか気を取り直して、mongoのectoアダプタをさがして`mix deps.get`したらectoのバージョンが合わないからできないと怒られる・・・どうやらmongoのectoアダプタは開発がだいぶ前で止まってしまっているようで、どうしてもmongoが使いたければectoを介さないドライバを使うか、直接叩くしかない模様。スキーマレスでecto使わなくてもよくね？という気持ちもあったのですが、ectoがないとパラメータバリデーションが面倒だよなぁという気持ちが勝ってしまって、mongoを使う気が完全に萎えてしまいました。

postgresやmysqlのjson型を使うか他のDBを探すか迷っていたのですが、ボーッとmongoの代替DBを探していたところ、ArangoDBというDBを見つけて面白そうだったので使ってみることにしました。

# Why ArangoDB?

脱線してしまいますが、なぜArangoDBが面白そうと思ったかというと、mongoと同じNoSQL(document)だけでなく、Graph、KVSも扱えるマルチモデルであることとと（Graphは使う予定ないけど）、オープンソースで開発されていたからですね。（mongoもオープンソースだったから安心できるわけじゃないけれど）
まだ日本では一般的ではないようですね。

# 立ちはだかったエラー

ectoを使っているだけあって、最初のうちは特に困ることもなくサクサクと書けていたのですが、問題は本格的に開発しようとテストを書き始めた時に発生しました。以下がその時に現れた今回の本題となるエラーです。

```
17:39:35.229 [debug] ArangoXEcto.Adapter.ensure_all_started

17:39:35.324 [debug] ArangoXEcto.Adapter.ensure_all_started

17:39:35.341 [error] Could not create schema migrations table. This error usually happens due to the following:

  * The database does not exist
  * The "schema_migrations" table, which Ecto uses for managing
    migrations, was defined by another library
  * There is a deadlock while migrating (such as using concurrent
    indexes with a migration_lock)

To fix the first issue, run "mix ecto.create".

To address the second, you can run "mix ecto.drop" followed by
"mix ecto.create". Alternatively you may configure Ecto to use
another table and/or repository for managing migrations:

    config :MyApp, MyApp.Repo,
      migration_source: "some_other_table_for_schema_migrations",
      migration_repo: AnotherRepoForSchemaMigrations

The full error report is shown below.

** (UndefinedFunctionError) function ArangoXEcto.Adapter.execute_ddl/3 is undefined or private. Did you mean:

      * execute/5

    (arangox_ecto 1.2.0) ArangoXEcto.Adapter.execute_ddl(%{adapter: ArangoXEcto.Adapter, cache: #Reference<0.920217074.1490157569.44619>, pid: #PID<0.292.0>, repo: MyApp.Repo}, {:create_if_not_exists, %Ecto.Migration.Table{comment: nil, engine: nil, name: :schema_migrations, options: nil, prefix: nil, primary_key: true}, [{:add, :version, :bigint, [primary_key: true]}, {:add, :inserted_at, :naive_datetime, []}]}, [timeout: :infinity, log: false, schema_migration: true, telemetry_options: [schema_migration: true]])
    (ecto_sql 3.9.0) lib/ecto/migrator.ex:672: Ecto.Migrator.verbose_schema_migration/3
    (ecto_sql 3.9.0) lib/ecto/migrator.ex:486: Ecto.Migrator.lock_for_migrations/4
    (ecto_sql 3.9.0) lib/ecto/migrator.ex:398: Ecto.Migrator.run/4
    (ecto_sql 3.9.0) lib/ecto/migrator.ex:146: Ecto.Migrator.with_repo/3
    (ecto_sql 3.9.0) lib/mix/tasks/ecto.migrate.ex:141: anonymous fn/5 in Mix.Tasks.Ecto.Migrate.run/2
    (elixir 1.13.4) lib/enum.ex:2396: Enum."-reduce/3-lists^foldl/2-0-"/3
    (ecto_sql 3.9.0) lib/mix/tasks/ecto.migrate.ex:129: Mix.Tasks.Ecto.Migrate.run/2
```

# 全然ちゃうやん・・・

初めて使うライブラリや環境でのエラーはあるあるなので、最初はとりあえず上から順に見ていこうと思って読み始めました。最初に現れるマイグレーションができていないというエラー文、そして丁寧にも問題の解決方法まで載せてくれている。しかし実は全く問題の原因と関係なく、逆に混乱の原因となってしまっていたのでした。以下に自戒を込めてアホな試行錯誤のドタバタデバッグ記録を残しておきます。**結論だけ知りたい方は次の章まで読み飛ばしてください。**

### config足らない疑惑

まずはdev環境で見たことがない問題だったので、test環境の設定し忘れかと思って`config/dev.exs`と`config/test.exs`を見比べました。そうすると案の定MyApp.Repoの設定が漏れていました。
そりゃ失敗するよねとtest環境用に設定を追記して再トライ・・・変わらず。足りてなかったのは足りてなくて別の問題の原因になるので、潰せたのは良しとして、これじゃなかったらしい。

### マイグレーション忘れてたのか〜。うっかりうっかり〜（スキーマレスDBということを忘れるバカの図）

なんだかよくわからないけれどスタックトレースを見るとarangox_ectoの中でプログラムがコケているのが原因のようだったので、きっとおまじないを忘れたのだろうと楽観してphoenixの言うとおりやってみることにしました。まずは`MIX_ENV=test mix ecto.create`。とりあえず得られる成功っぽい出力。しかし結果のエラー文は変わりません。にわかに焦り始める僕。
ひょっとして何か設定漏れがまだあるんじゃないかと「Ecto test」でググって[Testing with Ecto](https://hexdocs.pm/ecto/testing-with-ecto.html)を眺めてmix.exsのところが少しだけ違っていることに気づきます。

```
  # 僕のmix.exs
  defp aliases do
    [ ...
      test: ["ecto.create --quiet", "ecto.migrate --quiet", "test"]
    ]
  end
```

```
  # Testing with Ectoのmix.exs
  defp aliases do
    [ ...
     "test": ["ecto.create --quiet", "ecto.migrate", "test"]
    ]
  end
```

あ、`"test":`が`test:`になってる〜（バカ）

`test:`を`"test":`になおしてっと・・・

aliasで叩かれるコマンドは`ecto.create --quiet`と`ecto.migrate`と`test`ね。ふむふむ。テーブル作ってマイグレーションしてテストするっと。問題ないな（バカ。スキーマレスDBが使いたいとはなんだったのか）

一応`--quiet`の効果も調べて・・・あ、ログを出さないようにするのね。じゃぁ大丈夫だ。

よし、問題ないな（大アリ）

`mix test`・・・・状況変わらず(´・ω・｀)

### きっと僕は悪くない。ライブラリがバグってる！

落ち着いてもう一回エラーを見てみよう。

```
** (UndefinedFunctionError) function ArangoXEcto.Adapter.execute_ddl/3 is undefined or private. Did you mean:

      * execute/5
```

( ﾟдﾟ)ﾊｯ!設定は間違っていないのに関数がないって怒られてる！（バカ）
これは修正してPRしなきゃダメかも！（大バカ）
まずはEctoアダプタの`execute_ddl/3`について調べなきゃ！（そうじゃない）

> execute_ddl(command)
>
> @callback execute_ddl(command :: Ecto.Adapter.Migration.command()) :: String.t() | [iodata()]
>
> Receives a DDL command and returns a query that executes it.
>
> [Ecto SQLドキュメント](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.Connection.html)より

(´ε｀；)ｳｰﾝ・・・DDLってなんぞや・・・ググる。

> Data Definition Language の略で、データ定義言語の意味です。
> データベースのデータ構造を定義する言語です。
> DDLのコマンドは CREATE , DROP , ALTER , TRUNCATE の4つです。
>
> [DDLとは？SQL文を確認したよ](https://qiita.com/miyuki_samitani/items/1b38ea0e87ea05a0db80)より

なるほど。データ構造を定義する言語なのか。つまりデータ構造を作る所で失敗してて・・・

データ構造を作る所で・・・？

データ構造を作る？？？？？

**あ、今使ってるのスキーマレスDBじゃん（；＾ω＾）**

# いらない命令を消す

`mix.exs`にあるaliases関数は、mixコマンドを入力したときにキーに指定したサブコマンドを検知すると、値に指定したコマンドで置換して実行してくれます。`mix phx.new my-app`でPhoenixプロジェクトを生成するとデフォルトでは以下のようになっています。

```
  defp aliases do
    [ ...
      test: ["ecto.create --quiet", "ecto.migrate --quiet", "test"]
    ]
  end
```

前の章で盛大に勘違いをしていましたが、そもそもArangoDBはスキーマレスDBなので、"ecto.migrate --quiet"の命令は必要ありません。（ちゃんと読んでないのですが、一応マイグレーションという概念を導入することもarangoXEctoではできるようです。デフォルトでオフになっています。）
なので、以下のように書き換えます。

```
  defp aliases do
    [ ...
      test: ["ecto.create --quiet", "test"]
    ]
  end
```

そして再度`mix test`を実行すると悩まされていたエラーが消えて、代わりに別のエラーが現れました（まだあるんかい）

# 次の問題

そこそこの時間をかけてArangoDBはスキーマレスDBであることを再確認したわけですが（）、次はこちらのエラーが出現しました。

```
** (UndefinedFunctionError) function Ecto.Adapters.SQL.Sandbox.child_spec/1 is undefined or private
    (ecto_sql 3.9.0) Ecto.Adapters.SQL.Sandbox.child_spec({Arangox.Connection, [telemetry_prefix: [:my_app, :repo], otp_app: :my_app, timeout: 15000, username: "*******", password: "*********", hostname: "localhost", pool: Ecto.Adapters.SQL.Sandbox, pool_size: 1, database: "_system"]})
    (db_connection 2.4.2) lib/db_connection.ex:446: DBConnection.start_link/2
    (arangox_ecto 1.2.0) lib/arangox_ecto/behaviour/storage.ex:16: ArangoXEcto.Behaviour.Storage.storage_up/1
    (ecto 3.9.1) lib/mix/tasks/ecto.create.ex:53: anonymous fn/3 in Mix.Tasks.Ecto.Create.run/1
    (elixir 1.13.4) lib/enum.ex:937: Enum."-each/2-lists^foreach/1-0-"/2
    (mix 1.13.4) lib/mix/task.ex:397: anonymous fn/3 in Mix.Task.run_task/3
    (mix 1.13.4) lib/mix/task.ex:455: Mix.Task.run_alias/5
    (mix 1.13.4) lib/mix/cli.ex:84: Mix.CLI.run_task/2
```

またundefind or privateかよ(´・ω・｀)

# 奇跡的なひらめきで問題解決

そのときはSandbox機能がなんなのかよくわかっていなかったので、まずはSandbox機能から調べることにしました。

余談:
Sandbox機能とは、DBの絡んだunit testでマルチスレッドでテスト同士が干渉しないようにセッションのオーナーが誰なのかという情報を付与してテスト同士を独立に保つための機能のようです。（ちゃんとは理解してない）

[Ecto.Adapters.SQL.Sandboxのドキュメント](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.Sandbox.html)を読んでいたところ以下を発見しました。

```
While both PostgreSQL and MySQL support SQL Sandbox,
```

Sandbox機能はPostgreSQLとMySQLしかサポートしていないとのこと。つまり、他のDBではSandbox機能を使うこと自体おかしいのでは？とひらめきます。

とりあえず該当しそうなコードを削除。

```
@@ -30,9 +30,4 @@ defmodule MyAppWeb.ConnCase do
       @endpoint MyAppWeb.Endpoint
     end
   end
-
-  setup tags do
-    MyApp.DataCase.setup_sandbox(tags)
-    {:ok, conn: Phoenix.ConnTest.build_conn()}
-  end
 end
```
```
@@ -27,19 +28,6 @@ defmodule MyApp.DataCase do
     end
   end
 
-  setup tags do
-    MyApp.DataCase.setup_sandbox(tags)
-    :ok
-  end
-
-  @doc """
-  Sets up the sandbox based on the test tags.
-  """
-  def setup_sandbox(tags) do
-    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
-    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
-  end
-
```

`Finished in 0.09 seconds (0.09s async, 0.00s sync)`
`2 tests, 0 failures`
ようやくテストを通過させることができました。

# まとめ

今回の問題の原因と対策をまとめると、

- スキーマレスDBではマイグレーションコマンドが不要
- PostgreSQLとmysql以外はSandbox機能が不要なので該当コードの削除が必要

ということでした。みなさんも超メジャーどころから外れたDBをPhoenixで使いたいときは気をつけてくださいね！
