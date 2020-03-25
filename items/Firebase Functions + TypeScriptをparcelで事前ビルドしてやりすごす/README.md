Firebase functionsは現在標準でTypeScriptに対応している。
しかしHostingなど他の機能も使いたい場合や、それらとモジュールを共有したい場合、tsconfigやpackage.jsonが複数のディレクトリに割れたりしてちょっとしんどくなる。
この場合lernaなどでmonorepo構成を取るのが良いのだろうが、TypeScriptが絡んだりするとなかなかまだハマりどころが少なく無く面倒になりがちだ。

そこで以前[AWS Lambda向けのファイルをparcelでビルドした](]https://qiita.com/terrierscript/items/e3452704af812fa14fc0)のと似たようなことをやってみたら思ったより悪くなかったのでまとめておく。

## 1. 準備

まずは適当にディレクトリを作るとこから

```
$ mkdir some-firebase-project
$ cd some-firebase-project
$ yarn init -y
$ touch firebase.json
```

parcelとtypescriptも入れてしまおう

```
$ yarn add parcel typescript
$ yarn tsc --init
```

あと今回は`firebase init`は使わないのでそこは自前で行う

```
$ mkdir functions
$ touch functions/package.json
```

## 2. package install

次にプロジェクト直下でパッケージインストール。トップディレクトリでfunctionsに必要なパッケージを入れる。最終的にbundleしたものをアップするのでpeerDependencyも入れる

```
$ yarn add firebase-admin @firebase/database @firebase/app @firebase/app-types @firebase/database-types firebase-tools
```

## 3. functionファイルの用意

function本体となるファイルを配置する。

```
$ mkdir src
$ mkdir src/functions
```

```typescript
// src/functions/index.ts
import * as functions from "firebase-functions" // 全インポートにしないとひっかかるっぽい

exports.helloWorld = functions.https.onRequest((request, response) => {
  response.send("Hello from Firebase!")
})

```

## 4. スクリプト設定（本題）
package.jsonをこんな感じで設定

```json
{
  "scripts": {
    "build:functions": "parcel build src/functions/index.ts --target=node -d functions -o index.js --bundle-node-modules --no-source-maps --no-minify",

```

だいたい[AWS Lamdaの場合](https://qiita.com/terrierscript/items/e3452704af812fa14fc0
)と一緒で、`--target=node`、`--bundle-node-modules`にしている。
またfirebase-admin関連がsourcemap関連のwarningを吐き出してくるので`--no-source-maps`を付けている。
また吐き出し先を`-d functions`でfunctions下にしている。

`--no-minify`をしてるのはビルド速度を上げたい＆サーバーサイドのスクリプトなのでminifyする意味あんまり感じない＆デバッグ楽そう　ぐらいな気持ちなので好みで決めていい

これで`yarn build:functions`が出来た。

### 4.5. Cannot resolve dependency 'http2' の対策をする

本当はここまでで通るはずなのだが、そのまま実行すると`Cannot resolve dependency 'http2'`のエラーが出る事がある。
どうもparcelの問題として、http2モジュールをうまく読み込めずビルドが失敗する状態にあるようだ

https://github.com/parcel-bundler/parcel/issues/2921

ということでその場合は下記のようにhttp2パッケージをインストールしてしまえば解決する

```
$ yarn add http2
```


## 5. firebase.jsonの準備

次にfirebase.jsonを記述する。だいたいこんな感じ。predeployで指定を与える部分が重要

```json
{
  "functions": {
    "source": "functions",
    "predeploy": "yarn build:functions"
  },
  "hosting": {
    ... 
  }
}

```

## 6. functions/package.jsonをいじる

AWS Lambdaと違ってpackage.jsonが無いとdeploy時に失敗してしまうので、これを回避するためにpackage.jsonを置いとく。
バージョンは10にしておく。今回はparcel側で`--bundle-node-modules`するやり方をとっているので`dependencies`は不要だ。

```functions/package.json
{
  "name": "functions",
  "engines": {
    "node": "10"
  },
  "main": "index.js",
  "private": true
}
```

## 7. gitignoreする（オプション）

細かいがgitignoreとしてfunctions/index.jsを追加しておくと良い。

```.gitignore
functions/index.js
dist
```

好みでなければ吐き出し先を`functions/lib/`などディレクトリ先にしてそこごとignoreするのも良いだろう。

## 8. deploy

レッツデプロイ。

```
$ firebase deploy --only functions
```

## おまけ: hostingがここに入った場合

hositngが入るとこんな具合のディレクトリ構成になる。

```
.
├── dist # hostingの出力先
│   ├── hosting.09bd9689.js
│   └── index.html
├── functions # functionsの出力先
│   ├── index.js
│   └── package.json
├── src
│   ├── functions # functionsのソース
│   │   └── index.ts
│   ├── hosting # hostingのソース
│   │   ├── index.html
│   │   └── index.tsx
│   └── lib # 共通部分
│       └── database-client.ts
├── firebase.json
├── package.json
├── renovate.json
├── tsconfig.json
└── yarn.lock
```

firebase.jsonとpackage.jsonはこんな感じになる

```package.json
  "script": {
    "build": "parcel build src/index.html",
      ...
    
```

```firebase.json
  "hosting": {
    "public": "dist",
    "predeploy": "yarn build",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
```
