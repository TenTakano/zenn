---
title: "SurrealDB"
emoji: "🖋"
type: "tech"
topics: ["surrealdb"]
published: true
---

# はじめに

少し前に知人からRust製で何でもできるSurrealDBというものが出てきたらしいということを聞いたので、どんなものなのか触ってみることにした。
印象としては何でもかんでもとにかく詰め込んだ「ぼくのかんがえたさいきょうのDB」みたいな感じなのだが、しっかり資金調達もして開発をしているので本気のようだ。

細かい特徴や詳細は既に日本語でも解説記事が出ているのでそちらに譲るとして、実際に触ってみた記録を残しておく。

# dockerで立ち上げる

インストール自体はメチャクチャ簡単。dockerじゃなくても簡単に入るらしい。

```
$ docker pull surrealdb/surrealdb:latest
$ docker run --name surreal -d -p 8000:8000 surrealdb/surrealdb:latest start --user root --pass root
```

公式ドキュメントには記載がないが、startコマンドにオプションで`--user root --pass root`をつけておかないと、[InstructionのQuick start](https://surrealdb.com/docs/introduction/start)で躓いてしまう。（何もつけない場合何がデフォルトになっているのかちゃんと調べられていない。まだまだドキュメントも整備中っぽいので下手するとコードを読み解くことになるかも）

# APIから利用してみる

とりあえずバージョンチェック。こちらのAPIは認証も不要

```
$ curl http://localhost:8000/version
surrealdb-1.0.0-beta.9+20230402.5eafebd
```

レコードを既存のDB上に作成するためのAPIなどの一部の機能は専用のAPIが利用されているが、あまり充実しているわけではないので、現時点では実際に使おうとすると任意のSQLを実行できるAPIを叩くことになりそう。

```
$ curl -X POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "INFO FOR DB;" http://localhost:8000/sql | jq

[
  {
    "time": "98.505µs",
    "status": "OK",
    "result": {
      "dl": {},
      "dt": {},
      "fc": {},
      "pa": {},
      "sc": {},
      "tb": {}
    }
  }
]
```

実行時間が98.5マイクロ秒となっているのでめちゃくちゃ早い気がする。さすがのRust製。

# データを挿入してみる

てっきりテーブル諸々を設定しなきゃいけないんだろうと思い込んで調べまくってドツボにハマっていたが、名前空間・データベース・テーブルは必要に応じて自動的に作成されるようだ。
Quick Startにあるとおり入れてみたらすんなりできた。

```
$ curl -k -L -s --compressed POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "CREATE account SET name = 'ACME Inc', created_at = time::now();" http://localhost:8000/sql | jq
[
  {
    "time": "474.961µs",
    "status": "OK",
    "result": [
      {
        "created_at": "2023-04-29T17:09:28.320232319Z",
        "id": "account:2hw59m0nebs79vt0t4af",
        "name": "ACME Inc"
      }
    ]
  }
]
```

データが入ると、DB情報も変化が見られた。

```
$ curl -X POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "INFO FOR DB;" http://localhost:8000/sql | jq

[
  {
    "time": "64.752µs",
    "status": "OK",
    "result": {
      "dl": {},
      "dt": {},
      "fc": {},
      "pa": {},
      "sc": {},
      "tb": {
        "account": "DEFINE TABLE account SCHEMALESS PERMISSIONS NONE"
      }
    }
  }
]
```

当然クエリすれば挿入したデータが帰ってくる。

```
$ curl -k -L -s --compressed POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "SELECT * FROM account" http://localhost:8000/sql | jq

[
  {
    "time": "104.766µs",
    "status": "OK",
    "result": [
      {
        "created_at": "2023-04-29T17:09:28.320232319Z",
        "id": "account:2hw59m0nebs79vt0t4af",
        "name": "ACME Inc"
      }
    ]
  }
]
```

# データを削除してみる。

今度はデータを削除してみる。

```
$ curl -k -L -s --compressed POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "DELETE account:2hw59m0nebs79vt0t4af" http://localhost:8000/sql | jq 
[
  {
    "time": "100.329µs",
    "status": "OK",
    "result": []
  }
]
```

これでテーブルのデータはなくなるので全部消えたかと思いきや、テーブルの情報は残るらしい。

```
$ curl -X POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "INFO FOR DB;" http://localhost:8000/sql | jq

[
  {
    "time": "78.337µs",
    "status": "OK",
    "result": {
      "dl": {},
      "dt": {},
      "fc": {},
      "pa": {},
      "sc": {},
      "tb": {
        "account": "DEFINE TABLE account SCHEMALESS PERMISSIONS NONE"
      }
    }
  }
]
```

SurrealQLとやらのドキュメントを参照すると、送るSQLを`DELETE account;`とすることで全レコードが削除できるようだが、これを実行してもテーブルの定義はなくならなかった。
ずっと残ってしまうのは気持ち悪いなぁと思っていたら、REMOVE statementなるものがあった。どうやら`REMOVE TABLE account`とすれば良いらしい。

```
$ curl -k -L -s --compressed POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "REMOVE TABLE account" http://localhost:8000/sql | jq
[
  {
    "time": "55.254µs",
    "status": "OK",
    "result": null
  }
]

$ curl -X POST -H "Accept: application/json" -H "NS: test" -H "DB: test" -u "root:root" -d "INFO FOR DB;" http://localhost:8000/sql | jq

[
  {
    "time": "60.485µs",
    "status": "OK",
    "result": {
      "dl": {},
      "dt": {},
      "fc": {},
      "pa": {},
      "sc": {},
      "tb": {}
    }
  }
]
```

そういえばremoveとdeleteでは意味がちょっと違うんだった。
未だにこれをちゃんと自分の中で使い分けられてないなぁ。
