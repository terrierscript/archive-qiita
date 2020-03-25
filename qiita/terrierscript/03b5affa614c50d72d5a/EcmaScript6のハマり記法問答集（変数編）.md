
Reactで開発的なことをしてみたら、わりとReactそのものよりES6の記法的な質問がそれなりにあったのを感じたので雑多に問答集

# Q. `var`と`let`と`const`って何が違うの？
A. ざっくり言えばこう

```js
var a = "hoge" // もう忘れていい
let b = "fuga" // 変更できる値。mutable
const c = "baz" // 変更できない値。immutable
```
`var`に関しては、多分使うことはもう無いはず（と僕は思っている）。ESLintでも[no-var](http://eslint.org/docs/rules/no-var)というルールがあるので、設定しちゃっている。

# Q. `let obj = { hoge } `っていうこれなに
A. `{hoge}`は`{hoge: hoge}`となる。いわゆるシュガーシンタックス的なもの

こんな感じ。便利。

```js
let hoge = "fuga"

let obj = {
	hoge
}

console.log(obj) 
// { hoge: "fuga" }
```

# Q. `const { hoge } = obj`っていうこれなに
A. さっきの逆だと思っていい。`let hoge = c.hoge`と展開する。これは割りと他の言語にありがちな記法

```js
const { hoge } = obj

console.log(hoge)
// "fuga"
```

## 補足：さらにネストも展開できます。
これだとあんまり他の言語だと見ない（気がする）
そしてこの記事書いてる時に気づいたけど展開されるのは末端の要素だけのようだ

```js
let a = {
  b : {
    c : "d"
  }
}

let { b : { c } } = a
console.log(c) // "d"
console.log(b) // undefined
```


## 補足：関数の引数にも使えるのでこんなふうに組み合わせられる

```js
let fooFn = function({ hoge, fuga }){
	console.log(hoge, fuga)
}

fooFn({
	hoge: "aaa",
	fuga: "bbb"
}) 
// aaa, bbb
```
javascriptだとオプションとかをobject形式で渡すこと多いのでよくつかう。

## 補足：reactだったらこんなふうに利用する

```js
let foo = "baz"
let bee = "boo"
retrun <Cmp { ...{ foo, bee } } />
// <Cmp foo={foo} bee={bee} /> と同じ
```

### 補足の補足：↑の`{ ...{ a, b } }` こういうの何
`{a,b}`と一緒なのだけど、reactの記法上こう書くしか無い

```js
let a = 1
let b = 2
let c = { a, b }
let d = { ...{ a, b } }

console.log(c) 
console.log(d) 
// どっちもこうなる{ a: 1, b:2 }
```
詳しくは[Reactのドキュメント](https://facebook.github.io/react/docs/jsx-spread.html#spread-attributes)参照。ES7になったら改善しそう

# Q. `a = { [b] : c }`っての何

展開した変数をキーとして使えます。
使わないようでわりとあると便利。

```js
let b = "foo"
let a = { [b] : "baz" }
// a = { foo: "baz" }
```

