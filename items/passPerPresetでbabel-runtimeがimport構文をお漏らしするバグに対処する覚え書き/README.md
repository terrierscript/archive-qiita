
下記のissueで上げられているバグで色々ハマった。
https://github.com/babel/babel/issues/2877

理由や内部構造までは深追いする気力が無かったが、解決がちょっと苦労したのでメモを残す。

# 問題点：`babel-runtime`と、`export * from 'xxx'`という構文を組み合わせた場合にうまくいかない

下記のようなコードとbabelrc設定で、importが露出するバグに見舞われた

**コード**

```index.js
export * from './hoge'
```
```hoge.js
export class hoge {
  state = { a: "b" }
}
```

**babelrc**

```js
{
  "comments": false,
  "plugins": [
    "transform-runtime",
    "transform-class-properties",
  ],
  "presets": [
    ["es2015", {"loose": true} ],
    "stage-3",
    "react"
  ]
}
```

**出力結果**

```js
 :
/* 0 */
/***/ function(module, exports, __webpack_require__) {

	'use strict';

	import _Object$defineProperty from 'babel-runtime/core-js/object/define-property';
	import _Object$keys from 'babel-runtime/core-js/object/keys';
	exports.__esModule = true;

	var _hoge = __webpack_require__(1);

	_Object$keys(_hoge).forEach(function (key) {
  :
```

こんな感じでimportがおもらしされる。

# 解決方法：passPerPresetを多重化させて使う

色々試して、最終的にうまく行った設定が下記

```js
{
  "comments": false,
  "passPerPreset": true,
  "presets": [
    { "plugins": [ "transform-runtime"] },
    {
      "passPerPreset": false,
      "presets": [
        ["es2015", { "loose": true}],
        "stage-3",
        "react",
        { "plugins": [ "transform-class-properties" ] }, 
      ]
    }
  ]
}
```
* issue途中の[コメント](https://github.com/babel/babel/issues/2877#issuecomment-245402025)に書いてあったものでうまく行った
* passPerPresetを多重化させていて結構トリック的。
* `transform-class-properties`を組み合わせていたので更に混乱があったが、presetは逆順適応（[出典](https://babeljs.io/docs/plugins/#plugin-preset-ordering)）と気づいて、`transform-class-properties`を最後に持って行くことでうまくコンパイル出来た。
	* 一番最初に持ってきたりすると、構文エラーになる。


## その他実証メモ
* issueの[途中にあったコメント](https://github.com/babel/babel/issues/2877#issuecomment-245402025)にて、[transform-export-extensions](https://babeljs.io/docs/plugins/transform-export-extensions/)が足りないのでは？というのがあって混乱したが、上記のpluginは、`export * as ns from 'mod';`や`export v from 'mod';`という記法が対象で、`export * from 'xxx'`や`export {hoge} from './hoge'`という記法とは関係無かった（ややこしい）
* `export * from 'xxx'`という記法も、別段必須になるものでもないので、さっくりやめて素直な記法に書き直してしまうというのも一つの解決策
    * とはいえ[redom](https://github.com/pakastin/redom/blob/master/src/index.js)あたりではいい感じに多用されていたりするので、ちょっとそれだと名残惜しいかもしれない

###「エラーにはならないが、babel-runtimeの変換も行われない」というパターン

検証してたら見つけたパターン。
気づけない分、ちょっと注意が必要そう

#### その1. exportを{}にした変えたパターン

こんな感じでのexport利用

```index.js
export {hoge} from './hoge'
```

#### その2. 下記のように、全部一括でpassPerPresetした場合

途中まで「やったー！エラーでなくなったぞ！」とぬか喜びしていたconfig

```js
{
  // エラーにはならないが、babel-runtime変換されないbabelrc
  "comments": false,
  "passPerPreset": true,
  "presets": [
    {
      "plugins": [
        "transform-runtime",
        "transform-class-properties",
      ]
    },
    ["es2015", {"loose": true} ],
    "stage-3",
    "react"
  ]
}
```



