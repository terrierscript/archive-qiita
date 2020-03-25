create-react-appでtypescript使って素振りしたくなったとき、parcelに即切り替えるのアリなのでは、と思ったのでまとめる。

今回は例示としてはじめから作ってるが、create-react-appで始めたものの、途中でTypeScriptにしたくなったときにも手順はそれほど変わらないはず。

# 手順
## とりあえず作る
```
$ npx create-react-app example-app
```

## parcelを入れる。あとTypeScriptも

```
$ yarn add parcel-bundler parcel-plugin-typescript typescript
```

parcel標準でcssやらhtmlやらだいたいよしなにしてくれるが、typescriptだけpluginいるので入れる。
pluginはpackage.jsonに入っているものは強制で全部読み込まれるので、installするだけで良い
`parcel-plugin-typescript`はお行儀よくtypescriptを`peerDependency`にしてるので、それに従う

あとはreactあたりは型定義を入れとくと良いだろう

```
$ yarn add @types/react
```

もし、TypeScriptが不要という場合は下記の記事が参考になるだろう
https://qiita.com/MegaBlackLabel/items/5d5592d624bc5f8a341d

## tsconfigを設定

tsconfigの雛形を作成

```
$ yarn tsc --init
```

jsxだけ最低限設定。あとはお好みで

```json
"jsx": "react"
```

## package.jsonを書き換える

もともとこうなってるのを

```json
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
```

こうじゃ！

```json
  "scripts": {
    "start": "parcel serve public/index.html",
    "build": "parcel build public/index.html",
   }
```

## index.htmlいじる。

ヘッダ部分の`%PUBLIC_URL%`がパースコケるので消す。parcelだと相対パスにしとけば普通にバンドルしてくれる

```html
<!-- 
  <link rel="manifest" href="%PUBLIC_URL%/manifest.json">
  <link rel="shortcut icon" href="%PUBLIC_URL%/favicon.ico">
-->

<link rel="manifest" href="./manifest.json">
<link rel="shortcut icon" href="./favicon.ico">

```

javascriptの読み込み部分も追加。相対パスで指定してやるといい感じにやってくれる

```html
  <!-- ↓追加-->
  <script src="../src/index.js"></script>
</body>
```

## gitignore設定

```.gitignore
# parcel
/.cache
/dist
```

## App.js -> App.tsx

ファイル名変更

```
$ mv src/App.js src/App.tsx
```

あとsvgファイルが型定義無くでコケるのめんどいのでts-ignore

```ts
// @ts-ignore
import logo from "./logo.svg";
```

べつにlogo周り使わないだろうし消してもいいと思う。

css周りは何もせずに動く


## start!

```
$ yarn start
```

これで動く。

## ビルドするとき

```
$ yarn build
```
でOK。netlifyにもすぐ上げられるはず。

# なんかおかしいとき

parcelは結構キャッシュをがっつり持つっぽいので、cacheと出力先ファイルを消すと良い

```
$ rm .cache
$ rm dist/
```

`--no-cache`オプションでキャッシュさせないというのもアリ

```
$ yarn start --no-cache
```
