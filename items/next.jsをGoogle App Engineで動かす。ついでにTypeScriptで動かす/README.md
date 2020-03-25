
next.jsをFirebase FunctionやGoogle Cloud Containerで動かす話はあったが、GAEの話があんまり無さそうだったのでざっくりメモる。

* https://qiita.com/mizchi/items/60722563b8a938e7336f
* https://medium.com/google-cloud/next-js-tutorial-deploy-to-docker-on-google-cloud-container-engine-6b0c19dd8ecb

# GAEについて
Google App Engine、ざっくり言うとHerokuっぽいもの。

GAEにはflexibleとstandardというのがあり、NodeのStandardが追加されたということ。

https://qiita.com/comameito/items/3738262923c49da94fb9

この部分に疎いので、どちらが良いのかわからないが、下記のデモなどでもStandardを利用しているため、Standardを利用する（Flexibleだと何がまずいのかいまいちわかっていない）

https://github.com/blainegarrett/node-next-gae-demo

# GAEの手続き

このあたりをやった（このへんは資料溢れているし簡単だったので、すっとばす）

* カード登録をしてのGCPの有効化
    * 無料枠がそこそこあるので、すぐに課金とかは無さそう
* プロジェクトの作成
    * ~~リージョンが後から変更出来ないので、東京になるようにしておく必要がある~~
    * ~~firebaseと紐付けるなど考えると、firebaseプロジェクトより先にこっちを作っておいたほうがいいのかもしれない~~
    * ↑firestore使えなかったりするので今の所デフォルトのリージョンじゃないと不幸があるっぽい
    * チュートリアルが出てくるがスキップした。
* 予算の設定
    * 一応変な動きしてたらまずいので100円でアラートなるようにした

## nextの設定

server側もtypescriptで動かしたい。
https://github.com/zeit/next.js/tree/canary/examples/custom-server-typescript
を見るといちいちコンパイルしていてちょっとやりたくなかったので、ts-nodeで直で動かせる方法を考えた。

楽さをとってるので、上記のほうが自動再起動とかしてくれる嬉しさはあると思う。

### next.config.js

```
yarn add next @zeit/next-typescript ts-node
```

```js
const withTypescript = require("@zeit/next-typescript")

module.exports = withTypescript()
```

package.jsonのスクリプトコマンドはこんな具合。

```js
  "scripts": {
    "dev": "ts-node -T server/server.ts ",
    "prestart": "next build",
    "start": "NODE_ENV=production ts-node -T server/server.ts ",
    "deploy": "gcloud app deploy"
  },
```

ポイントとしては、ts-nodeの引数に`-T`のtranspileOnlyを指定してやること。
そして、prestartを仕掛けて、start前にビルドされるようにする

### 欄外: nodemonも入れる
exampleよく見たらdevの場合だけnodemonするの良さそう。
（動作確認出来てないのでなにか足りてないかも）

```
yarn add nodemon
```

```package.js
{
    "dev": "nodemon --exec ts-node -T server/server.ts",
}
```

```nodemon.json
{
  "watch": ["server/**/*.ts"],
  "execMap": {
    "ts": "ts-node --emit"
  }
}
```

## gcloudのコマンドインストール

```
brew cask install google-cloud-sdk
```

```
gcloud
gcloud auth login
```

そして先程設定したproject-idで

```
gcloud config set project <Project-ID>
```

を設定する

## app.yamlの設定

```yaml
runtime: nodejs8
```

## deploy

Cloud Container Builderの有効化を促されるので従う。

```
gcloud app deploy
```

1分ちょいかかる感じ

# エラーログを見る

なんか動かなかったらエラーログを見るとよい
App Engine -> バージョン -> 「診断」のカラムにある「ツール」-> 「ログ」で吐き出されてるログがそのまま見れるはず。
