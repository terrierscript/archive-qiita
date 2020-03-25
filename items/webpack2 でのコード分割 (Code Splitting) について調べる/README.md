
code splittingを頑張らずに切り抜けられるかと思ったけど、まだちょっと逃げ切れなそうだったのでちゃんと調べた。
（[Chrome Canaryに`<script module>`](https://jakearchibald.com/2017/es-modules-in-browsers/) が来てたり数年スパンだと不要になりそうな技術だけど、もう少しの間はこのへんを扱っていかないといけなそう）

# Code splitting

## まず普通のコード

こんなコードを用意する

```js
// index.js
const lib = require("./lib")

lib()
```

```js
// lib.js
module.exports = () => {
  console.log("This is Lib")
}
```

するとこんなふうにビルドされる

```js
// モジュール解決する基本的な部分が差し込まれる
// ...
// ...
/******/ 	function __webpack_require__(moduleId) {
/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId]) {
/******/ 			return installedModules[moduleId].exports;
/******/ 		}
/******/    }
// ...
// ...
// それが終わると、自分が書いたモジュールが差し込まれる

/******/ ([
/* 0 */
/***/ (function(module, exports) {

module.exports = () => {
  console.log("This is Lib")
}


/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

const lib = __webpack_require__(0)

lib()

/***/ })
/******/ ]);
```
`__webpack_require__`によって`require`が解決される様子が見て取れる。

`__webpack_require__`の詳細は、[lib/MainTemplate.js](https://github.com/webpack/webpack/blob/aecc297c509f08f95148ccb2915a2b9ff3ee5fa3/lib/MainTemplate.js)あたりで構築されている（文字列でjavascriptが組み立てられているっぽい。つらそう）


## webpackランタイム部分の分離（Manifest分離）

この`__webpack_require__`に関するランタイム部分をCommonChunkPluginを利用することで分離することが紹介されている。

webpack公式ではこれをManifest fileと呼称している。
railsあたりと組み合わせた時のmanifestと役割が似ているが、微妙に違う

https://webpack.js.org/plugins/commons-chunk-plugin/#manifest-file

```js
// webpack.config.js
  plugins: [
    new webpack.optimize.CommonsChunkPlugin({ 
      name: 'manifest',
      minChunks: Infinity
    })
  ]
```

`CommonChunkPlugin`で分離されたmanifestには、webpack内部でのモジュール解決に利用される関数のみが抽出される。
非同期ローディング用の関数（`requireEnsure`）も含まれるようになる。

```js
// manifest.js
 (function(modules) { // webpackBootstrap
 	// install a JSONP callback for chunk loading
 	var parentJsonpFunction = window["webpackJsonp"];
 	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) {
    // manifest外に書かれたscriptをJSONPとして実行する。
    // ...
 	};

 	// webpack内部のrequire関数
 	function __webpack_require__(moduleId) {
    // ...
    // ...
 	}

  // requireEnsure。関数を非同期ロードする仕組み。
 	__webpack_require__.e = function requireEnsure(chunkId) {
    // ... 
    // ...
 		
 		// コード読み込みは、<script>タグを生成してDOMに埋め込んで読み込む
 		var head = document.getElementsByTagName('head')[0];
 		var script = document.createElement('script');
 		script.type = 'text/javascript';
    // ...
    // ...
        // 後述する非同期読み込みを行うと、それらの読み込みをうまく解決するコードが差し込まれる。ｍanifestっぽさのある部分
        script.src = __webpack_require__.p + "" + ({"1":"main"}[chunkId]||chunkId) + ".js?" + {"0":"d5186cef317c5c895373","1":"eb51fb214d292368d33e"}[chunkId] + "";

 		head.appendChild(script);

 		return promise;
 	};
  // ここから下は様々便利関数などが展開される
  // ...
  // ...

 })
 ([]);
```

残されたメインのコード側は、`webpackJsonp`で囲まれる。
これが`manifest.js`によってうまく読み込み解決される

```js
// main.js

webpackJsonp([0],[
/* 0 */
/***/ (function(module, exports) {

module.exports = () => {
  console.log("This is Lib")
}

/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

const lib = __webpack_require__(0)

lib()

/***/ })
],[1]);
```

## import() / require.ensure / bundle でLazy importする

非同期処理するのは現在webpackは3つほどやり方が提供されている。

### import()

`import()`は[Dynamic import](https://github.com/tc39/proposal-dynamic-import)と呼ばれる現在stage-3の機能。
非同期をPromise読み込みする。一番記述量が少なく書ける。

https://webpack.js.org/guides/code-splitting-async/#dynamic-import-import-

```js
const main = () => {
  const lib = import("./lib")
  lib()
}

```

```js
// main.js
webpackJsonp([1],[
/* 0 */
/***/ (function(module, exports, __webpack_require__) {

const main = () => {
  const lib = __webpack_require__.e/* import() */(0).then(__webpack_require__.bind(null, 1))
  lib()
}


/***/ })
],[0]);
```

webpackでは内部的に`requireEnsure`(`__webpack_require__.e`)が利用される


### require.ensure

webpack1時代から存在するwebpack固有のrequire拡張。`import()`より少し機能が多いが、`import()`に置き換えられるということがアナウンスされている。

https://webpack.js.org/guides/code-splitting-async/#require-ensure-

```js
const main = () => {
  require.ensure([], (require) => {
    const lib = require("./lib")
    lib()
  })
}
```

```js
webpackJsonp([1],[
/* 0 */
/***/ (function(module, exports, __webpack_require__) {

const main = () => {
  __webpack_require__.e/* require.ensure */(0).then(((require) => {
    const lib = __webpack_require__(1)
    lib()
  }).bind(null, __webpack_require__)).catch(__webpack_require__.oe)
}


/***/ })
],[0]);
```

### bundle-loader 

require-ensureの別解として`bundle-loader`も存在する。
こちらもおそらく`import()`を利用すれば使う事は無いと思われるが、比較として掲載する

```js
const main = () => {
  const lib = require("bundle-loader?lazy!./lib")
  lib()
}
```

```js
// main.js
webpackJsonp([1],[
/* 0 */
/***/ (function(module, exports, __webpack_require__) {

module.exports = function(cb) {
	__webpack_require__.e/* require.ensure */(0).then((function(require) {
		cb(__webpack_require__(2));
	}).bind(null, __webpack_require__)).catch(__webpack_require__.oe);
}

/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

const main = () => {
  const lib = __webpack_require__(0)
  lib()
}


/***/ })
],[1]);
```
### System.import

System.import is deprecated だそうです
https://webpack.js.org/guides/code-splitting-async/#system-import-is-deprecated
さして難しくも無いはずなので、`import()`へ置き換えるだけでよさそう

### import()についてもうちょっと深いトピック

#### webpackでの`import()`の制限

なんとなく`import()`の構文を見ると

```js
const lazy = (modulePath) => import(modulePath)
```

的なことが出来そうな気がしてしまうが、これは下記のようなWarningが出る

```js
10:9-22 Critical dependency: the request of a dependency is an expression
```

webpackにおいては、`import()`をビルド時に静的解決するので、これが出来ないような作りになっている。

実際のビルド結果では、不正なパッケージとして扱われ、その実態は`webpackEmptyContext`に置き換えられてしまう。

```js
var lazySomeApp = lazy("./SomeApp");

var lazy = path = __webpack_require__(281)(path);

/***/ 281:
/***/ (function(module, exports) {

function webpackEmptyContext(req) {
	throw new Error("Cannot find module '" + req + "'.");
}
webpackEmptyContext.keys = function() { return []; };
webpackEmptyContext.resolve = webpackEmptyContext;
module.exports = webpackEmptyContext;
webpackEmptyContext.id = 281;

/***/ }),

```

#### babel-plugin-syntax-dynamic-import

babelの`babel-plugin-syntax-dynamic-import`を使うやり方も記載されている。

https://webpack.js.org/guides/code-splitting-async/#usage-with-babel

webpack側では解決されず、ただのpromiseに変換される。ファイル分割もされなくなる。
「import()は使いたいけどコード分離はしたくない」とかの場合には使えるかもしれない。

```js
// babelがimport()をこんな感じの関数に置き換える。ただのpromiseになる
const importConverted = (module) => Promise.resolve(module)
```

#### TypeScriptとの組み合わせ

TypeScriptと組み合わせた時、現状だと構文エラー扱いになってしまう。
https://github.com/Microsoft/TypeScript/issues/12364

`import()`を利用する箇所だけ`js`ファイル化する妥協が必要になる。型がつかない問題もあるものの、そのへんは`d.ts`の型定義ファイルを別で作るか、2.3以上から対応された[JSDoc](https://github.com/Microsoft/TypeScript/wiki/JSDoc-support-in-JavaScript)でカバーすると良さそう。

```js
// import.js
export const lazySomeApp = () => import("./SomeApp")
```

```ts
// import.d.ts
import * as SomeApp from "./SomeApp"

export const lazySomeApp = () => Promise<typeof SomeApp>
```

```js
// Typescript 2.3以降でのJSDoc利用
/**
 * @template T
 * @return {Promise<T>}
 */
export const lazySomeApp = () => import("./SomeApp")
```


その他にもissueでは、`_import()`でちょろまかす手段も提案されている。かなりトリッキーかもしれない。
https://github.com/Microsoft/TypeScript/issues/12364#issuecomment-270819757

```ts
declare function _import<T>(path: string): Promise<T>;
const lazyPromise = _import("./SomeApp")
```

```js
// webpack.config.js
// webpack側でこれを置き換えている。力技感
{
    loader: 'string-replace-loader',
    options: {
        search: '_import(',
        replace: 'import('
    }
},
```

## reactとの組み合わせ

非同期なcomponentをreactで扱おうとすると一工夫する必要が出て来る。
そんなに難しいものではないので[react-routerのcode splitting](https://reacttraining.com/react-router/web/guides/code-splitting)ページあたりを参考に作るでも良いが、ざっくり調べるとこのへんがあった

* [async-reactor](https://github.com/xtuc/async-reactor/)
    * Star: 200 over
    * `async/await`をベースにしたcomponentローダー
    * コードベースが小さく素直な作りに見える
* [react-async-component](https://github.com/ctrlplusb/react-async-component)
    * Star: 300 over
    * ベータ扱い
    * SSRも対応している
* [react-loadable](https://github.com/thejameskyle/react-loadable)
    * Star: 2000 over
    * Issueが閉じられてPull Requ受付のみになっている。
    * 内部で色々やっている


react-loadableは良いのだがissueを受け付けてないという点に少々怖さもあり、個人的に利用するならasync-reactorかreact-async-componentが良さそうだなと感じている。

# （おまけ）更なるCode splittingについても少し調べる

ちょっと実戦投入するには勇気がいりそうなものを触りだけ。

## SWPrecacheWebpackPlugin
https://github.com/goldhand/sw-precache-webpack-plugin

service-workerのPrecacheをする

```js
const SWPrecacheWebpackPlugin = require('sw-precache-webpack-plugin')
// ...
// ...
// ...
plugins: [
  new SWPrecacheWebpackPlugin({
    cacheId: 'dynamic-example',
  })
```

これを実行すると、`service-worker.js`というファイルが出力される。
これを下記のように読み込むことでservice-workerが有効になる

```js
(function() {
  if('serviceWorker' in navigator) {
    navigator.serviceWorker.register('service-worker.js');
  }
})();
```

react-routerのwebsiteでも使われているが、現在はコメントアウトされている

## AggressiveSplittingPlugin

読んで字の如くアグレッシブに分割するplugin。特性上、ほぼHTTP2向けという謳い文句。

https://github.com/webpack/webpack/tree/master/examples/http2-aggressive-splitting

作者による記事もある。
https://medium.com/webpack/webpack-http-2-7083ec3f3ce6


```js
plugins: [
  new webpack.optimize.AggressiveSplittingPlugin({
    minSize: 30000,
    maxSize: 50000
  }),
]
```

指定されたminSize以上maxSize未満を目指して分離していく。
reactあたりをimportしてみるとわかりやすく分割される

```js
const react = require("react")
const main = () => {
  console.log("This is main")
  console.log(react)
}
```

0,1,2に分割される

```
      Asset     Size  Chunks             Chunk Names
       0.js  50.4 kB       0  [emitted]
       1.js    52 kB       1  [emitted]
       2.js  42.4 kB       2  [emitted]
```

`__webpack_require__`などの挙動、コードベースは特に変更されない。
また、こちらは特にlazyなloadingを目指したものではないので、`<script>`タグで全部読み込むのが前提に鳴る

```js
<script src="0.js"></script>
<script src="1.js"></script>
<script src="2.js"></script>
```

