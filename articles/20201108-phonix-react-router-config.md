---
title: "続・PhoenixでReactを使う ~Router編~"
emoji: "🖋"
type: "tech"
topics: ["phoenix", "react"]
published: true
---

# 誰に向けた記事か

この記事は前回の記事（[PhoenixでReactを使う](https://zenn.dev/ten_takano/articles/20201107-phoenix-react)）の続編です．前回の記事でカバーしきれなかったRouterの設定について紹介します．したがって，対象者は前回と同じです．

- 普段elixir（Phoenix）でサーバーサイドの開発をしている
- フロントエンドの知識が全くないが，フロントを書かなくてはいけない
- しかし，種々の理由でサーバーサイドレンダリングを使えないor使いたくない（アニメーションが多用されるページとか？）

# Phoenix バージョン

```
$ mix phx.new -v
Phoenix v1.5.5
```

# まえがき

前回の記事ではPhoenixで書いたアプリのフロントをReactで構成する方法を紹介しました．しかし前回の記事で紹介した方法では，シングルページしか対応出来ておらず，routerはphoenixのものを使用したままになっていました．小規模なアプリならこれでもOKですが，小規模であればそもそもReactいらなくね？という話にもなってくるので，react-routerを使ってページ遷移をできるように調査をしました．

この記事の内容は既に[ボイラープレート](https://github.com/TenTakano/Revive)にマージ済みです．

# やりかた

前回分の設定が完了している前提での解説です．まだ設定していない場合は[こちら](https://zenn.dev/ten_takano/articles/20201107-phoenix-react)を参考に先に設定をしてください．

## node_moduleのインストール

```
cd assets
npm install --save react-router react-router-dom
npm install --save @types/react-router @types/react-router-dom   # typescriptを使用する場合
```

## Routerに対応したスクリプトの準備

以前と少し設定が変わっているため，typescriptでの実装になっています．jsで実装する場合は，前回の記事の`assets/js/app.js`を書き換えてください．typescriptに対応させたい場合は，特にphoenix側で設定することはないので，`assets`内でtypescript用のモジュールをインストールし，`webpack.config.js`を書き換えてください．

```ts:assets/src/index.txs
import React from 'react';
import ReactDOM from 'react-dom';
import { BrowserRouter, Redirect, Route, Switch } from 'react-router-dom';

const MyMain: React.FunctionComponent<{}> = () => {
  return (
    <div>This is a root page</div>
  );
}

const MySecond: React.FunctionComponent<{}> = () => {
  return (
    <div>This is a second page</div>
  );
}

const Root: React.FunctionComponent<{}> = () => {
  return (
    <Switch>
      <Route exact path="/" component={MyMain} />
      <Route exact path="/second" component={MySecond} />
      <Redirect to="/" />
    </Switch>
  );
}

ReactDOM.render(
  <BrowserRouter>
    <Root />
  </BrowserRouter>,
  document.getElementById('root')
);
```

`Root`のパラメータに`exact`が渡してあるのですが，これはパスに完全マッチさせるために必要なオプションなので，`exact`なしだと意図したとおりにページが表示されてくれません．（フロントに慣れている方だと当り前なんでしょうね・・・僕は知らなかったので結構沼りました）

## routerの設定

phoenixでフロントもAPIサーバーも提供する場合，フロントのページへアクセスしようとした場合，必ず`lib/project_web/router.ex`を通ることになります．なので，route制御をReact側でやりたい場合にはphoenixのrouterからフロント側にフォワーディングしてやる必要があります．

**`phoenix forwarding`あたりでググると，plugの解説がいくつもヒットしますが，plugでやる必要はないです**
（最初plugで実装する必要があるのかと思ってました）

特定のパス以下を全てフォワーディングしたい場合には以下のように`router.ex`を書き換えます．

```elixir:lib/project_web/router.ex
  scope "/", ReviveWeb do
    pipe_through :browser

    get "/*path", PageController, :index
  end
```

このように設定した場合，`http://localhost:8000/something.html`にアクセスすると，`PageController.index/2`の`parameters`に`%{path => "something.html"}`が渡されます．つまり，`router.ex`のパスでアスタリスクの後に続けられたキーにパスがマッチされます．

既に前章で`/`と`/second`にページを作ってあるので，`mix phx.server`でサーバーを立ち上げると，`http://localhost:8000/`にアクセスすれば`This is a root page`，`http://localhost:8000/second/`にアクセスすれば`This is a second page`と表示されます．

phoenixのテンプレートプロジェクトだと，dashboardのルーティングがデフォルトで定義されています．これよりも前に上記のルーティングを定義してしまうとwarningが出てしまいます．
`router.ex`の定義順序は所詮elixirのマッチングで処理されるので，ドメインのルート以下全てをマッチングさせたい場合は一番下に持ってくると良いかと思います．

こんな感じですね．

```elixir:lib/project_web/router.ex
  if Mix.env() in [:dev, :test] do
    import Phoenix.LiveDashboard.Router

    scope "/" do
      pipe_through :browser
      live_dashboard "/dashboard", metrics: ReviveWeb.Telemetry
    end
  end

  scope "/", ReviveWeb do
    pipe_through :browser

    get "/*path", PageController, :index
  end
```

# あとがき

これで一通り必要な物が揃ったのかなと思っています．まだLintやらなんやら必要なパッケージはある気がしますが，フロント側で完結できる設定しか残っていないはずなので，検索すれば出てくる情報ばかりになるのかなと．
少し気になっている点としては，react-nativeを利用する場合にはどうなるのかですが，こちらは僕がまだreact-nativeがどのように動くのかすら全く理解してないので，必要になればそのうち書くかも知れません．

# 参考にさせてもらった記事

- [A Phoenix+React initial setup that actually works](https://medium.com/@resir014/a-phoenix-react-initial-setup-that-actually-works-c943e48f1e9e)
    前回の記事を書いた時点では見つけられていなかったのですが，前回の記事文含めてこちらかなり詳しく解説されていますね．