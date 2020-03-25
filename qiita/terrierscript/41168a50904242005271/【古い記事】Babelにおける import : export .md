#### 2016-01-31追記
この記事を最初に書いた頃は他に何も情報がありませんでしたが、今ではよりだいぶわかりやすい資料が出揃ってきました。しっかりとしたimport/exportの仕様をもっと知りたい場合は下記を参照することを推奨します

- https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/import
- http://www.slideshare.net/teppeis/effective-es6
- http://uehaj.hatenablog.com/entry/2015/11/07/001848

---

#### 2015/12/05追記
コメント欄より

>Babel v6 から Babel 公式の transform である transform-es2015-modules-commonjs を使った場合 ES6 の export default を出力したときに exports.default に変更されてしまいました。
もし以前のように module.exports として出力したい場合は add-module-exports を使う必要がありそうです。

Babelのversion upに伴い、この記事の情報は一部古くなっている部分が出てきました。ご注意下さい。

----

ES6のimportやらexportやら。こんな感想を持った。

- なんかpythonっぽい
- pythonのあれもいまいちよくわからん

なんとなーくこういうことなんだろうなというのはわかりつついまいち理解できてない。
Draftとかちゃんと追えるのがいいんだけど[なんだか難しい](http://www.2ality.com/2014/09/es6-modules-final.html)（軟弱）


そうだ、Babelの変換コードを見たら何やるのかわかるんじゃね！？


・・・邪道だなあと自分でも思いつつ、取っ掛かりとしてはよさそうなので[Try it out](https://babeljs.io/repl/)してみることにする。

## import編
まず読み込む方
### 普通の使いかた
```es6.js
import "foo"
```
```babel.js
require("foo");
```
なるほど。これ左辺無いの使いどころあるのか・・・？

### {}でかこむ
```es6.js
import { foo } from "foo" 
```
```babel.js
var foo = require("foo").foo;
```
おお。これよこれ。でもfooのfooを呼ぶのか。

### fromをあわせる
```es6.js
import foo from "foo"
```

```babel.js
var _interopRequire = function (obj) { return obj && obj.__esModule ? obj["default"] : obj; };

var foo = _interopRequire(require("foo"));
```

おおう。途端に複雑になったか。
いや、ちゃんと見ればそう難しくは無いはず。どうもこういうことらしい

- 基本的にはrequireを普通にしている
- `obj.__esModule`が`true`であれば `obj["default"]`を返す。
	- module.exports.defaultというアレか。
	- `__esModule`は他のtranspi特にbabel用ではないのか？ちょっと調べきれなかった。
- `__esModule`が無くても`require`されているオブジェクトを返しているだけだ。

### *を使う
```es6.js
import * as foo from "foo"
```
```babel.js

var _interopRequireWildcard = function (obj) { return obj && obj.__esModule ? obj : { "default": obj }; };

var foo = _interopRequireWildcard(require("foo"));
```
微妙に違う。

- objectそのものを`foo`に当てているようだ。
- `__esModule`でなければ`default`のプロパティに元のobjectが入っているobjectを返すらしい。

### { as }で囲んでみる
```es6.js
import {oof as foo} from "foo";
```
```babel.js
var foo = require("foo").oof;
```
なるほど。asの後が変数。

#### 欄外 `import 'utility' as utils;`みたいなやつ
このあたりを調べていると
http://www.sitepoint.com/understanding-es6-modules/
ここで上記のような記載がされていた。
が、どうもこれはコンパイル通っていない。策定前の話だろうか？

## export編
次はmodule書く方。
`export`と`default`を組み合わせていく。

### export default function

```es6.js
export default function(){
}
```

```babel.js
module.exports = function () {};
```

うん、わかりやすい。今までっぽい書き方はこうやるのね。

### export function
じゃあdefaultをつけないと？

```es6.js
export function foo(){
	console.log("foo")
}
```
```babel.js
exports.foo = foo;
Object.defineProperty(exports, "__esModule", {
	value: true
});

function foo() {
	console.log("foo");
}
```
ほほーん。また複雑な感じ。

- `Object.defineProperty`で__esModulesを設定している。
- `import foo from "foo"`で呼ばれているのがこれなんだろう。
- loose modeだと`exports.__esModule = true`
	- IE8だと動かない問題ありそう。
- `module.exports`ではなくなるのね。

### {} で囲んでみる
```es6.js
function foo() {
}
 
function baz(a, b) {
}
 
export { foo, baz }
```

```babel.js
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
function foo() {}

function baz(a, b) {}

exports.foo = foo;
exports.baz = baz;
```
- definePropertyまわりはいままでと一緒。
- exportsに色々突っ込んでいる。

### as を使う
```es6.js
function foo() {
}
 
export { foo as oof}
```

```babel.js
Object.defineProperty(exports, "__esModule", {
  value: true
});
function foo() {}

exports.oof = foo;
```
単純にこれは当て込む変数を変えているだけっぽい

### {}とdefaultを組み合わせる

```es6.js
function foo() {
}
export default { foo : foo }
```

```babel.js
"use strict";

function foo() {}
module.exports = { foo: foo };
```

こうするとたまによくやる`module.exports.foo`とかの代わりに使える。

# まとめ
ここまでをざっくり表にしてみるとこんな感じ
（一部表の都合で簡略化しているのはご了承）

#### import
| EcmaScript6 | Babel |
|:---|:---|
|`import "foo"`|`require("foo");`|
|`import { foo } from "foo" `|`var foo = require("foo").foo;`|
|`import foo from "foo"`|`var foo = require("foo")`　(*)|
|`import * as foo from "foo"` |`var foo = { "default" : require("foo")}`　(*)|
|`import {oof as foo} from "foo";`|`var foo = require("foo").oof;`|
(*) 1行で書けるように略式
#### export
| EcmaScript6 | Babel |
|:---|:---|
|`export default function(){}`|`module.exports = function () {};`|
|`export function foo(){}`|`exports.foo = foo;`|
|`export { foo, baz }`|`exports.foo = foo; exports.baz = baz;`|
|`export { foo as oof}`|`exports.oof = foo;`|



