
## 問題

Electronの webContents.executeJavascript は、stringでjavascriptコードを受け取る。
例えば **IDとパスワードを渡してログインしたい** と思った時にこれがちょっと工夫が必要だった。

## 結論

こんな関数を用意するだけでいけた！

```js
const wrap = (fn, ...args) => `(${fn.toString()})(${args.map( arg => `"${arg}"` ).join(',')})`
```


### 使い方はこんな具合

```js

const webview = document.querySelector("#browser")

// webviewで実行したい関数
const input = (name, pass) => {
  document.querySelector("#username").value = name
  document.querySelector("#password").value = pass
  document.querySelector("#login").click()
}

// wrap!
const scr = wrap(input, "my-name", "my-password")

webview.addEventListener('did-finish-load', (e) => {
	// execute!
    webview.executeJavaScript(scr, () => {
     ....
    }) 
}
```


### 解説など

ワンライナーって言いたかっただけの理由で一行で書いてみたが、わかりやすく書くとこんな感じになる。

```js
const wrap = (fn, ...args) => {
  const callArgs = args.map( arg => `"${arg}"` ).join(',')
  const functionString = fn.toString()
  return `(${functionString})(${callArgs})`
}
```

wrapで作られた`scr`の変数はこんな感じになっている。

```js
((name, pass) => {
  document.querySelector("#username").value = name
  document.querySelector("#password").value = pass
  document.querySelector("#login").click()
})("my-name", "my-password")
```

* `(function(){})()`みたいな即時関数で囲む
* `Function.toString()`で関数の文字列を取り出している
* ES2015構文大活躍
    * `...args`でその後の引数をquoteで囲んで、即時関数の引数にしている。
        * [spread演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Spread_operator)で受け取っているのはわりと好み。別に第二引数を配列にしてても良かったとは思う。
    * templateリテラリ大活躍

下記あたりを参考に思いついた

* [electronのwebview.executeJavaScriptで書くjavascriptを見やすくしてみた](http://qiita.com/FaithnhMaster/items/ead9ed07205537d53069)
* [segmentio/nightmare:lib/javascript.js](https://github.com/segmentio/nightmare/blob/443cb4439d80bebc48135e824df71f283aac3833/lib/javascript.js)


