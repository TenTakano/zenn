---
title: "PhoenixでReactを使う"
emoji: "🖋"
type: "tech"
topics: ["phoenix", "react"]
published: true
---

# 誰に向けた記事か

- 普段elixir（Phoenix）でサーバーサイドの開発をしている
- フロントエンドの知識が全くないが，フロントを書かなくてはいけない
- しかし，種々の理由でサーバーサイドレンダリングを使えないor使いたくない（アニメーションが多用されるページとか？）

# Phoenix バージョン

```
$ mix phx.new -v
Phoenix v1.5.5
```

# 記事の背景

サーバーエンドでPhoenix，フロントエンドでReactを使ってwebアプリを開発したいという機会があった（というかちょこちょこ開発したいと思っていたのだが，ようやく重い腰を上げてフロント側も書こうと言う気になった）のだが，実際にやり始めると情報自体がほとんど無く，あっても古いといったことがしばしばあったのでメモ代わりに書いておく．

筆者は普段elixirでサーバーサイドしかほぼ書いていないので，フロントの事情を全くわかっていなかった．なので，Phoenixからみてどのようにフロント（React）を取り扱うかを書いていく．Reactのお作法についてはネットに溢れているので，あくまでもPhoenixで扱うならという観点を取り扱う．

# Phoenixはどうフロントのassetを取り扱うか

この章はPhoenixの振る舞いについての覚え書きなので，やり方だけが知りたい人は読み飛ばしてもらって構わない．

`phx.new`で生成されたphoenixのプロジェクトを覗くと，`assets`配下と`priv/static`配下にそれらしきファイルが格納されている．しかしよく見ると，`assets`の内容と`priv/static`の内容は重複している．これはphoenixのドキュメントを読めば書いてあるのだが，`assets`内に格納されたフロントのコードをphoenixがバンドル（jsのimportを解決してひとつのファイルにまとめる作業）して`priv/static`に格納しているかららしい．

バンドルを行うのがバンドラーと呼ばれる物で，Phoenixは1.3（？）からwebpackを使うようになった．なので，バンドルの細かい制御をしたければ`asset/webpack.config.js`を書き換える必要がある．
実際に`asset/webpack.config.js`の中身を見ると，バンドルしたファイルの出力先が`/priv/static/js`になっていることがわかる．

```js:webpack.config.js
    output: {
      filename: '[name].js',
      path: path.resolve(__dirname, '../priv/static/js'), // ここ
      publicPath: '/js/'
    },
```

# PhoenixとReactのつなぎ込み

## Phoenixプロジェクトの生成

まずはPhoenixのプロジェクトを準備．特に変わったことはせずに普通にプロジェクトを作る．

```
mix phx.new [プロジェクト名]
```

必要なければ`--no-ecto`なりなんなり`--no-webpack`以外のオプションを好きにつけてOK．

## React関連のモジュール導入

プロジェクトファイルが生成されたら，`asset`ディレクトリに移動して，フロント側で必要なモジュールをインストールする．

```
cd assets
npm install react react-dom --save
npm install --save-dev @babel/preset-react
```

## Babelの設定変更

そのままだとReactがプリコンパイルされず，jsxが取り扱えない旨のエラーが出るので，プリコンパイルされるようにbabelの設定（`assets/.babelrc`）を修正する．

```json:assets/.babelrc
{
    "presets": [
        "@babel/preset-env",
        "@babel/preset-react"
    ]
}
```

## Webページの編集

```haml:lib/project_web/templates/page/index.html.eex
<div id="root"></div>
```

```js:assets/js/app.js
import React from 'react'
import ReactDOM from 'react-dom'

const MyReactComponent = () => <div>Hello world</div>
ReactDOM.render(<MyReactComponent />, document.getElementById('root'))
```

もちろん固定で表示させたい物があれば`index.html.eex`を好きに弄って構わない．ちなみに，ヘッダ部分については`lib/project_web/templates/layout/app.html.eex`に書かれているのでそちらを編集する必要がある．

## あとがき

- このあたりの操作は今後頻繁に繰り返すことになりそうなので，[ボイラープレート](https://github.com/TenTakano/Revive/tree/master)を作成しはじめた．
- やはりelixir自体がまだ新興なせいか，日本語のみならず英語での記事もほとんど見当たらずなかなか手間取ってしまった．同じようなことをしようとして行き詰まっている人の助けになればと思う．
- PhoenixのLiveViewもフロントエンドの形態として興味はあるのだが，こちらはelixirとphoenixよりもさらに新興なので，まだまだといった所だと思う．また，ReactはReact nativeでネイティブアプリとしても動作できる利点もあるので，完全に代替可能とはいかないし，できるようになるとしてもまだあと数年はかかるような気がしている．
- フロント側の知識が全くないところからの手探り（普段はjsonを返すAPIしか触ってないので，サーバーサイドレンダリングすらよく分かっていない状態）だったので，何か間違いや良くない書き方をしているところがあれば指摘頂けると助かります．

## 参考にさせてもらった記事

- [React1.4で、Reactを動作させる](https://qiita.com/tajihiro/items/eece75b3fba4dceb0fd3)
操作としてはほとんどこちらの記事と同じなのだが，preset周りが変わったのか，そのままではうまく動作させることができなかった．

*****

[2020/11/08 追記]
react-routerを使ってページ遷移させる方法を書きました．
[続・PhoenixでReactを使う ~Router編~](https://zenn.dev/ten_takano/articles/20201108-phoenix-react-router-config)
