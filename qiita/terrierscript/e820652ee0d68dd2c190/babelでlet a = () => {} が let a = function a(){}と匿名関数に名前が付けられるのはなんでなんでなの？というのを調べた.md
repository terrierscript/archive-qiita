
# 発端

https://twitter.com/highwide/status/737917395730374656

> let hoge = (x) => x + 1
がbabelで
var hoge = function hoge(x) {x + 1;}
的にコンパイルされるので、無名関数じゃないじゃん！なんで！と駄々こねて同僚の業務時間奪ってしまった。

という話をされて、「なるほど、まあ確かに。」と思ったので調べた。

# 結論
- `arrow-function`のそのものの機能 / 仕様ではない。
- `function.name`のための変換によって変換がされてるっぽい
	- function.name の詳細は[こちらの記事](http://js-next.hatenablog.com/entry/2016/04/19/190006)が有益。

# おためし
準備としてこんなコードを用意

```js
let a = () => {}       // arrow function
let b = function(){}   // 普通のfunction(無名）
let c = function d(){} // 名前付きfunction
```

### babel-preset-es2015でcompile

わりといつもやるようなやり方。

```
% $(npm bin)/babel foo.js --presets es2015               
```

```js
"use strict";

var a = function a() {};
var b = function b() {};
var c = function d() {};
```

確かに。無名関数に名前がつけられた。

### [transform-es2015-arrow-function](http://babeljs.io/docs/plugins/transform-es2015-arrow-functions/)だけ使う

babelは各機能毎に使えるので、`arrow-function`だけ利用してみると、ちょっと面白い事になった

```js
% $(npm bin)/babel foo.js --plugins transform-es2015-arrow-functions                        
```

```js
let a = function () {}; // まだ無名！
let b = function () {};
let c = function d() {};
```

関数に変換されるだけで名前はつけられてない状態になった。

### [transform-es2015-function-name](http://babeljs.io/docs/plugins/transform-es2015-function-name/)だけ使う

で、問題のfunction.nameの話。

```
% $(npm bin)/babel foo.js --plugins transform-es2015-function-name                          
```

```js
let a = () => {};
let b = function b() {}; // 名前がついた！
let c = function d() {};
```

arrow functionは当然変換されないのだが、bに注目してみると、`function b()`と名前がついているのが確認出来る。この２つのtransformによってarrow functionに名前が付くようになっていることがわかる。

### このふたつをどっちも使う

やるまでもないけど、一応

```
% $(npm bin)/babel foo.js --plugins transform-es2015-function-name,transform-es2015-arrow-functions
```

```js
let a = function a() {};
let b = function b() {};
let c = function d() {};
```

## `function.name`とは？
function.nameプロパティは、ざっくり言えば、関数の名前を取得する機能になる。

```js
// Function Statements
function fn () {}
console.log(fn.name) // "fn"

// variables
const a = function(){}
console.log(a.name) // "a"
const b = () => {}
console.log(b.name) // "b"
```

ここで、各環境でのサポート具合がわかる [Compatibility table ](http://kangax.github.io/compat-table/es6/#test-function_name_property) を参考にすると、それなりにこの機能をサポートしてないブラウザが多い。

サポートしてないブラウザの場合は、[core-js](https://github.com/zloirock/core-js)がpolyfillしてくれている。
そこで更にcore-jsが[何をやっているのかを見ると](https://github.com/zloirock/core-js/blob/master/modules/es6.function.name.js)、だいたいこんな感じで処理している。
（core-jsのソースが読みづらい場合は[Stack Overflowの類似記事](http://stackoverflow.com/questions/2648293/javascript-get-function-name)らへんを眺めると良いかもしれない）


1. functionをstringに変換して、関数の内容を文字列として取得する(`'' + that`)[^1]
2. `/^\s*function ([^ (]*)/`という正規表現で、名前の部分を取得する
3. 取得できた値を、Function.prototype.nameで返せるようにしている。

[^1]: 自分の記憶が確かなら、functionをstring変換すると文字列になるというのは、特に明確な仕様ではなく、「だいたいのブラウザがそう実装している」という類のものだったはず。

`transform-es2015-function-name`が`let a = function(){}`を`let a = function a(){}`にしてくれることによって、core-jsがここまで処理を出来る、といった具合になる。

### 余談：じゃあ`function.name`って何に使えるのか？
主観だが、基本的にはデバッグ向けぐらいじゃないかなーと思っている。

本格的に使わなければならない状況に陥ったとすると、無理のある実装をしている可能性を疑うだろう。

ただし、例えばデバッグの場合だったら、こんな場合とかで使えるだろう（あんまりいいコードの例が思いつかなかった）

```js
function hogeFunctionGenerator(someFlag, defaultCallback){
  if(someFlag){
    return function someFlagFn(){}
  }
  return defaultCallback
}

// どの関数が結局コールされるのか知りたい場合
console.log(hogeFunctionGenerator(true, function (){}).name) // "someFlagFn"
console.log(hogeFunctionGenerator(false, function (){}).name) // ""
console.log(hogeFunctionGenerator(false, function cb(){}).name) // "cb"
```

### 余談のおまけ:bableにおいてclassのconstructorではどうなるか？

function.nameを利用して、`class`がどのクラスなのかというのを判定することを`c.consturctor.name`を呼び出すことでが出来るということが[こちら](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/name#Examples)にて紹介されていた。

ただし、この場合で深めの使い方をすると、babelではなかなかうまくいかなそうだ。

```js
class Hoge {}

function createClassZ() {
  // Z classを返す
  return class Z{ 
    constructor(){
    }
  }
}
function createAnonymousClass() {
  // 匿名classを返す
  return class { 
    constructor(){
    }
  }
}
let ZX = createClassZ()
let zx = new ZX()

let X = createAnonymousClass()
let x = new X()

console.log("zx", zx.constructor.name)
console.log("x", x.constructor.name)
```


Chrome(V8)では

```
zx Z
x undefined
```
となってくれるところが、babelではclassをpolyfillしている都合でこんな感じになってしまう。

```
zx Z
x _class
```

このような使い方だとヤバそうなので注意したい。

類似の問題はissueとして上がっていた。
https://phabricator.babeljs.io/T7014

# 参考資料など
- http://www.ecma-international.org/ecma-262/6.0/#sec-setfunctionname
- http://kangax.github.io/compat-table/es6/#test-function_name_property
- https://bugzilla.mozilla.org/show_bug.cgi?id=883377
- http://www.2ality.com/2015/09/function-names-es6.html
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/name
- http://stackoverflow.com/questions/2648293/javascript-get-function-name
