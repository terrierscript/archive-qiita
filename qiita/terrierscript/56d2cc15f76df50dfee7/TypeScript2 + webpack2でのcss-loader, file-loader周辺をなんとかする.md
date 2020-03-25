
WebpackとTypeScriptの扱いについて、まだ手探りだが、今のところのプラクティスをまとめたい。

# css-loader, file-loaderなどで読み込むリソースが型エラーになるのをなんとかする話

webpack + babel + css-loader という構成のところからwebpack + typescriptにしていくにあたって、js, ts以外のファイルの読み込みは、当然そのままだと型がエラーとなる。

これを解決するのに、2点ほど手順が必要だった。

## 手順1. 読み込み方法を変更

file-loaderなどは`module.exports = "some-resource"`という形で吐き出しされるので、TypeScriptにあわせるのであれば、importの仕方を変える方が良い。
（Webpack2のtree shakingを利用する場合は、元のコードでも壊れず動いてしまうのだが、これについては後述）

```js
// 元のコード
import style from "./foo.css"
```

これをこうしていく。

```ts
import * as style from "./foo.css"
// または
import style = require("./foo.css")

```

## 2. typing

型定義をしないとtypescriptは落ちるので行う。
`tsconfg.json`で独自な`typings`ディレクトリを用意した上で型定義をすると良さそう。

```json
{
    ...
    "typeRoots" : [
        "typings",
        "node_modules/@types/"
    ]
    ...
}
```

型定義には[Wildcard module declaration](https://www.typescriptlang.org/docs/handbook/modules.html#wildcard-module-declarations)という定義方法を使う事で乗り越えられる。


「とりあえず`any`でいいから！なんとかしたい！」という場合は、こんな具合。

```ts
// typings/resource.d.ts

declare module "*.css"
declare module "*.png"
declare module "*.jpg"
```

「もうちょっと毛が生えた感じに！」という場合はこのぐらいまでならサクッと定義できる

```ts
// typings/resource.d.ts
declare module "*.css" {
  const classes: {[className: string]: string} // css-moduleの結果をstring型のobjectに
  export = classes
  // import style from "./foo.css"で読み込みたいなら下記（後述）
  // export default classes
}

declare module "*.png" {
  const content: string
  export = content
  // import style from "./some.png"で読み込みたいなら下記（後述）
  // export default content
}
```

この定義はこのへんを参考にした。
* https://github.com/s-panferov/awesome-typescript-loader/issues/146#issuecomment-248808206
* https://github.com/Quramy/typed-css-modules/issues/2#issuecomment-256794347)
* https://github.com/Microsoft/TypeScript/issues/6615#issuecomment-188593420

「もっとちゃんと型定義したい！」という場合は、下記などを導入するのが良いだろう。

* [TypeScript + React JSX + CSS Modules で実現するタイプセーフなWeb開発](http://qiita.com/Quramy/items/a5d8967cdbd1b8575130)


# webpackでの`import`解決のされ方はtsconfig.jsonのcompilerOptions.moduleによって変わるという話

**TL;DR**
* `module: es2015`の場合、typescriptはモジュール解決に関与しない。webpackにモジュール解決が移譲されて、babelっぽく扱われる（babelと互換性が維持されるが、ESmoduleの仕様とズレるので、今後どこかで変更されるだろうと思われる）
* `module: commonjs`だと、typescriptが`import / export`を変換する。互換性は維持されない（正しいESmoduleの仕様に準拠する。babelとの互換性はなくなる）


## 前提（と疑問）

例えば`url-loader`、`file-loader`は、下記のような吐き出しをする

```js
/******/ ([
/* 0 */
/***/ (function(module, exports) {

module.exports = "data:text/plain;base64,aGVsbG8="

/***/ }),
```

ESmoduleの読み方で言えば`import * as foo from "foo"`という読み方でないと読み込めない理屈になる。
しかし、自分の手元で正しく動いていたので、「これは何故だろう？」という疑問が出てきた。

## 実験

まずこんなTypeScriptファイルを用意

```ts
import * as txt1 from "./foo.txt"
import txt2 from "./foo.txt"

console.log("===========")
console.log("import * as txt1")
console.log("=>", txt1)
console.log("===========")
console.log("===========")
console.log("import txt2")
console.log("=>", txt2)
console.log("===========")
```

Webpack設定を用意。
今回は、外から`compilerOptions.module`を変更できるようにする。

```js
module.exports = ( env ) => {
  const { moduleType } = env
  return {
    entry: './src/index.ts',
    output: {
      path: `output/${moduleType}`,
      filename: 'bundle.js'
    },
    resolve: {
      extensions: ['.ts', '.js']
    },
    module: {
      rules: [
        { test: /\.txt$/, use: 'url-loader' },
        { test: /\.ts$/, use: [{
          loader: 'ts-loader',
          options: {
            compilerOptions: {
              // tsconfig.jsonのmoduleを上書き
              module: moduleType
            }
          }
        }]},
      ]
    }
  }
}
```

そしてビルド→実行をする

まずcommonjsの場合

```sh
$ webpack --env.moduleType=commonjs
$ node output/commonjs/bundle.js
===========
import * as txt1
=> data:text/plain;base64,aGVsbG8=
===========
===========
import txt2
=> undefined
===========
```

結果：`import txt2 from "./foo.txt"`は`undefined`になる。


次にes2015

```sh
$ webpack --env.moduleType=es2015
$ node output/es2015/bundle.js
===========
import * as txt1
=> data:text/plain;base64,aGVsbG8=
===========
===========
import txt2
=> data:text/plain;base64,aGVsbG8=
===========
```

結果：`import * as txt1 from "./foo.txt"`、`import txt2 from "./foo.txt"`、どちらもstringが取得できる。



## 解説

### commonjsの場合の話


Typescript(ts-loader)の段階で、`import module from "./module"`は下記のような感じで変換される

```js
var module_1 = require("./module");
var a = function () {
    module_1["default"]();
};
```

最終的にこんな感じに変換される。

```js
var txt1 = __webpack_require__(0);
var foo_txt_1 = __webpack_require__(0); // txt2として読み込んだもの
console.log("===========");
console.log("import * as txt1");
console.log("=>", txt1);
console.log("===========");
console.log("===========");
console.log("import txt2");
console.log("=>", foo_txt_1.default); // .defaultにアクセスしてる
console.log("===========");
```

file-loaderは`module.exports.default`に何も吐き出してないので、`commonjs`モードで`import foo from "./foo"`と読み込むと、undefinedになってしまう。

### es2015の場合の話

TypeScriptは`import / export`構文をいじらず、そのまま吐き出す。

```js
import mod from "./module"
const a = () => {
    mod()
}
```

これをwebpackが受け取り処理する。

webpackのloaderはbabel同様、`__esModule`フラグを利用した後方互換維持の処理をしている。（これを ESM interopと呼んでるそうだ）
[source](https://github.com/webpack/webpack/blob/93ac8e9c3699bf704068eaccaceec57b3dd45a14/lib/MainTemplate.js#L143-L151)

```js
/******/ 	// getDefaultExport function for compatibility with non-harmony modules
/******/ 	__webpack_require__.n = function(module) {
/******/ 		var getter = module && module.__esModule ?
/******/ 			function getDefault() { return module['default']; } :
/******/ 			function getModuleExports() { return module; };
/******/ 		__webpack_require__.d(getter, 'a', getter);
/******/ 		return getter;
/******/ 	};
```

最終的にはこんな感じになっている。

```js
"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__foo_txt__ = __webpack_require__(0);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__foo_txt___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_0__foo_txt__);


console.log("===========");
console.log("import * as txt1");
console.log("=>", __WEBPACK_IMPORTED_MODULE_0__foo_txt__);
console.log("===========");
console.log("===========");
console.log("import txt2");
console.log("=>", __WEBPACK_IMPORTED_MODULE_0__foo_txt___default.a); // .defaultへのアクセスではなく、Object.definePropertyのgetterとしてアクセスするように変換されている
console.log("===========");

```

これによって、結果的に`import foo from "foo.css"でも読み出せるようになってる模様。

この後方互換は今後どうするかは議論対象になっている様なので、ある程度注意はしたほうが良さそう（とはいえ、議論を軽く追ってみるとwebpack2の間に突然消える、のようなことが起きることはなさそうに思える）

* https://github.com/webpack/webpack/issues/4170
* https://github.com/airbnb/babel-plugin-dynamic-import-webpack/pull/14#issuecomment-275159815

Webpack2(babel)の仕様に併せて[importが壊れるケースもある](https://www.reddit.com/r/javascript/comments/5rikrz/why_airbnb_will_never_use_webpack_2/dd7qihm/)らしいので注意

## まとめ

* webpackのtree shakingな読み方はbabelと同じ事やってるので、`es2015`にすればloader系がいきなりぶっ壊れたりはしない。
* 可能なら、型によって読み出しを制御しつつ、TypeScriptの手法に併せていった方が無難と思われる。
* 「どうせloader使ってるならwebpackに併せてしまえばいいのでは？」という判断とかもありえるが、そこらへんは自己判断で。
* commonjsにして変な読み出しをさせないようにする、というのも無くはなさそうだが、型で防ぐでも十分には思える
* webpackもそのうちこの辺の読み出しに動きがあると思われるので、何処かでは書き換えが発生するものと考えたほうが良いだろう。

