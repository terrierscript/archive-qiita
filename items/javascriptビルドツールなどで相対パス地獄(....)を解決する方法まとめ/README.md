
`require`、`import`などのモジュール解決において、相対パスを解決しようとすると../../../と大量に出てしまう問題。
relative paths hellと呼ばれたりもするらしい。

- [Better local require() paths for Node.js
](https://gist.github.com/branneman/8048520)
- [browserify-handbook avoiding ../../../../../../..](https://github.com/substack/browserify-handbook#avoiding-)

基本`require`の挙動としてはNode.jsのmodulesと同じ挙動を原則としたかったりわりと小めんどくさかったので結構今までガマンしておとなしく../../と書いたりしていた（モジュール小分けにしてあんまりこの問題に当たらないようにしていたというのもある）

しかし「`import`構文だと実はそもそもの原則違うのでは？[^1]」とか「frontend向けに組んでいる時にそこ厳密にすることでもないのでは？」とか「趣味レベル越えたコードだとモジュール小分けとかも出来ないのでは？」とか思い各buildツールで相対パス地獄をどうすると出来るのかみたいなのを調べた。

[^1]: importの実際の解決方法に関しては正直知識が薄いので不安感強め

# 前提など

こんな感じのプロジェクトを想定

```
.
└── src
    ├── foo
    │   └── baz.js
    └── index.js
```
そして`index.js`にて

```js
import baz from "foo/baz"
```

で解決できるような世界を考える。
import / exportを利用する前提として、必要があればbabelを使う

## 基本
### node.jsのModulesではどうか？
`require`での話。結構複雑だが、概ねこんな感じ

- `./node_modules`を解決する
	- ローカルもグローバルも見る
- `$NODE_PATH`の環境変数に指定されたパスも解決する

詳細はドキュメントを
https://nodejs.org/api/modules.html

### (参考) [browserify handbook](https://github.com/substack/browserify-handbook#avoiding-)で紹介されている解決法

node.jsの世界に近めの解決法が紹介されている。とはいえあんまり頻繁に更新されてるものでもないので情報古めかも。

- symlinkを`node_modules`下に貼る
- そもそも`node_modules`の下に自前のモジュールを作る
- `$NODE_PATH`を使ってカスタムパスを使う（後述）

## Build System編
### webpack
webapckはなんでも出来てしまうので、普通に出来る。
loaderの`include`設定を利用する

```js
module.exports = {
  entry: "./src/index.js",
  // resolve.rootで指定
  resolve: {
    root: [
      path.join(__dirname, "./src")
    ]
  },
  module: {
    loaders: [
      {
        test: /^src\/.*\.js$/,
        include: [
          path.resolve(__dirname, "src"),
        ],
        loader: "babel-loader"
      }
    ]
  }
}
```

### browserify

#### コマンドラインの場合
node.jsの基本と一緒。
`$NODE_PATH`の環境変数を仕込む

```
NODE_PATH=./some/lib browserify -e src/index.js -o build/index.js
```

#### API利用の場合
`opt.paths`の設定があるのでこれを利用できる

```js
browserify("src/index.js", {
  paths:[
    "./node_module", "./src"
  ]
}).bundle(function(err, out){
  // out.toStirng() とかでコード取れる
})
```

ちなみにpackage.jsonに`paths`とか仕込んだりして読み取ってはくれないし、同等のCLIオプションも用意されてない。

### rollup
`import`構文はrollupではデフォルトなので、相対パスの解決だけ考えれば良い

[include-path](https://github.com/dot-build/rollup-plugin-includepaths)プラグインを利用する。

手元でちゃんと試せてないが、概ねこんな具合。

```js
import includePaths from 'rollup-plugin-includepaths';

let includePathOptions = {
    include: {},
    paths: ['src'],
};

export default {
    entry: './src/index.js',
    dest: 'build/index.js',
    plugins: [ includePaths(includePathOptions) ],
};
```

## test framework編

### mocha
`$NODE_PATH`を設定する。

webpackと組み合わせている場合は二重に設定を書くことになる。

```
NODE_PATH=src/ mocha ./test
```

### ava
同じく`$NODE_PATH`を設定する。

```
NODE_PATH=src/ ava ./test
```
# 言語編

### TypeScript

tsconfig.jsonにこう

```json
{
    "baseUrl": "./src"
}
```

### flowtype

0.32時点。

`module.system.node.resolve_dirname`を設定する必要ありそう。`NODE_PATH`だけだと解決してくれないっぽい

```.flowconfig
[options]

module.system.node.resolve_dirname=./node_modules
module.system.node.resolve_dirname=./src
```

# まとめ

- どうやっても邪道になる印象があったけど、結構なんだかんだ各ツール手法は出揃ってる
- NODE_PATHを抑えておけばとりあえずはなんとかなる。
- テストツールのことを考えるとむしろNODE_PATHを使うのをベースにするほうが良いのかも。
- 当たり前だけどnpmにpublishするような予定のプロジェクトなら多分やめとくほうがいい
