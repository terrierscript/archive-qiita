
先日、npmから`const`など一部ブラウザが対応してないコードが入ってしまって[ハマっ](http://qiita.com/inuscript/items/2dd2221d032c777dab29)たが、逆にpublishする側になったら以外と「これどうしようか？」と思って調べた。

---

# 目的
* npmにpublishするためのビルドがしたい
* なるべく簡易に、ミニマムに。
* うっかりbrowser対応されてないようなコードで利用者が不幸になるのなるべく防止したい。ぐらいな心持ち
* 「えっbabel-cilって同名ファイル吐き出してるだけじゃん。これ何に使うの」という疑問の解消

### 目的外のこと
  * 積極的にbabelっていく話
    * `.babelrc`次第でどんどん使えますが、そこ自体は目的にしてません。お好みで。
  * node.js向けオンリーのnpmパッケージの事
    * 用途次第で、ブラウザが対象外なものなら、複雑化させずにそのままpublishしていいんじゃないか派です。
  * module 解決したり結果bundleしたりminifyしたりする話
    * しません。browserify、webpack、rollupらへん使いましょう
  * umdなども吐く話 
    * しません。browserify、webpack、rollupらへん使いましょう
    * なんとなく著名パッケージ眺めると良いかもしれません（後述）

# 方法：babel-cliを使う

## babel-cliはそもそも何をしてくれるのか？

単純に`babel`というコマンドで、引数として与えられたディレクトリのファイルを変換する。
単体で使った場合には、純粋にコードの静的変換だけをしている。
当然、Webpackやbrowserifyを使った時のように、`require`や`import/export`の解決をしたりはしない。

```
% npm run build

> sample@0.0.1 build /path/to/project
> babel src --out-dir lib

src/index.js -> lib/index.js
src/some-module.js -> lib/some-module.js
```

通常のWeb開発だと`browserify`や`webpack`と組み合わせてbundleするので、あんまり使い所ないのでは？と思っていたが、npmにpublishするときは逆にbundleせずにバラバラに変換するだけ、というのが都合が良い。

## Setup
`babel-cli`と`babel-preset-es2015`入れておく

```
$ npm i -D babel-cli babel-preset-es2015
```

お好みで`rimraf`。ビルド前にディレクトリクリーンしておくのに便利

```
$ npm i -D rimraf
```


## ディレクトリ構成

だいたいこんな感じ。

```
├── src // ビルド前のファイル
│   ├── some-module.js
│   └── index.js
├── test // テスト
├── lib // ビルド後ファイル。
│   ├── some-module.js
│   └── index.js
├── README.md
└── package.json
```

## package.json
`babel-cli`をディレクトリごと`lib/`に出力して、公開対象を`lib/`にする。

package.jsonはこんな感じになる。

```package.json
{
   :
  "main": "lib/index.js", // 重要
  "scripts": {
    "prebuild": "rimraf lib",
    "build": "babel src --out-dir lib", // 重要
    "preversion": "npm run build",
     :
     :
  },
  "files": [ 
    "lib" // 重要
  ],
  "devDependencies": {
     :
    "babel-cli": "^6.14.0",
    "babel-preset-es2015": "^6.14.0",
    "rimraf": "^2.5.4"
     :
  }
}
```

* `main`フィールドは`lib/main`
* 同様、`file`には`["lib"]`と指定。
* `npm run build`で`babel src --out-dir lib` で src -> lib と吐き出す。
* 他のscriptsは単なる便利化。
  * `pre`,`post`をつけておけば自動的にビルド実行してくれる。`npm version`、`npm publish`時に漏れないようにbuildしておく。
  * `prebuild`で`rimraf`を使ってディレクトリクリーンしておく。

もし`src`で`import/export`を利用して、Webpack 2系やrollupのtree shakingを使いたいと思っているのであれば、下記を記載しておいても良い。

```package.json
{
    :
  "main": "lib/index.js", 
  "module": "src/index.js", // for webpack 2
  "jsnext:main": "src/index.js", // for rollup
    :
}
```

Tree shakingについて詳しく知りたい場合は下記あたりが詳しめなので参照のこと。
* http://qiita.com/cognitom/items/e3ac0da00241f427dad6#tree-shaking
* http://chuckwebtips.hatenablog.com/entry/2016/01/26/020235

## .gitignore

生成ターゲットは基本ignoreしといていいはず。

```.gitignore
lib/
```


## .babelrc

あんまり特殊な事は無いが、最低限これ入れておけば大丈夫だろう、ぐらいなチョイスとして`preset-es2015`を入れた。[^1]

[^1]: [`preset-latest`](http://babeljs.io/docs/plugins/preset-latest/)でも良いかなとも思っているが、まだちゃんと調べられてないので保留した。

`loose`オプションを`true`にすべきかなど悩ましいが、[redux](https://github.com/reactjs/redux/blob/0e80e7a61489cbf5b746c922d6f3b98fa3ea711a/.babelrc)あたりを参考に`true`にした。

```.babelrc
{
  "presets": [
    ["es2015", { "loose": true }]
  ]
}
```

#### 欄外メモ：.babelrcのenvと`BABEL_ENV`

[reduxのbabelrc](https://github.com/reactjs/redux/blob/master/.babelrc)を覗くと、`env`というオプションが設定されている。

```.babelrc
{
  "plugins": [
    :
    :
  ],
  "env": {
    "commonjs": {
      "plugins": [
        ["transform-es2015-modules-commonjs", { "loose": true }]
      ]
    }
  }
}
```

これは特定の場合だけに特定のpresetオプションを効かせたい場合に使えるらしい。

```
$ BABEL_ENV=commonjs babel src --out-dir lib
```
とすると、`env.commonjs`に記載したpresetを追加で効かせる、という効果。
普通そんなに使わなそうだなと思ったが、[bablejs.io](https://babeljs.io/docs/usage/babelrc/#env-option)によると、このような特定bundle向けの他に、「developmentの時だけ特定のpluginを効かせたい」というようなことも利用想定としているようだ。

# まとめ

## 参考にした package.json

#### npm向けはbabel-cliにしてるパッケージ
今回取った手法 + webpack的な感じ。
webpack併用しているのが多いが、npm用というよりumd配布など向けっぽい

  * [redux](https://github.com/reactjs/redux/blob/5da0f7abeaaa31fc453ae343c2bafe99ecdbcca6/package.json#L26)
  * [aphrodite](https://github.com/Khan/aphrodite/blob/de0bd50dc9901a22a04f8dd48fec796e58ff136c/package.json#L10-L25)
  * [history](https://github.com/mjackson/history/blob/master/package.json#L15)
  * [redux-observable](https://github.com/redux-observable/redux-observable/blob/master/package.json#L9)
  * [normalizr](https://github.com/paularmstrong/normalizr/blob/master/package.json)

#### Grunt使ってビルド
  * [react](https://github.com/facebook/react/blob/7d7defe30f9e8124185676fa7e2c841ffee17ed5/package.json#L81)
    * gruntからbrowserifyっぽい
  * [axios](https://github.com/mzabriskie/axios/blob/master/package.json#L9)
    * gruntからwebpackっぽい

#### rollup派
思ったよりパラパラ見てたらrollup派がいた。

  * [vue.js](https://github.com/vuejs/vue/blob/dev/package.json)
  * [preact](https://github.com/developit/preact/blob/master/package.json)
  * [redom](https://github.com/pakastin/redom/blob/3421a2000ed4b0cc1167f7985369cd886962df02/package.json#10)
  * [d3](https://github.com/d3/d3/blob/master/package.json)


## 雑感
* babel-cli今までデバッグ用ぐらいにしか使えない子だと思っててごめんなさい
* webpackとかbrowserifyナシで全然いけるのはだいぶ楽
* umdとか結構配布しているけど、需要どれだけあるんだろう？と思った。

