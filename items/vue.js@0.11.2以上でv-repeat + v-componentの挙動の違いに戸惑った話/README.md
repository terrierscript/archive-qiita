
**2014/12/20: ご指摘を受けた部分について修正しました**

# 0.11.0での話
もともと0.11.0で下記のような感じで作っていた

```html
<html>
  <body>
    <div id="sample">
      <div v-repeat="items" v-component="item">
        <span>baz</span>
      </div>
    </div>
    <script src="build.js"></script>
  </body>
</html>
```

```app.js
var Vue = require("vue")

// 子要素はコンポーネント化
Vue.component("item", {})

var app = new Vue({
  data :{
    items : [
      {v :1},
      {v :2},
      {v :3},
    ]
  }
})

app.$mount("#sample")
```

で、出力はこんな具合。

```output.html
    <div>
      <span>baz</span>
    </div><div>
      <span>baz</span>
    </div><div>
      <span>baz</span>
    </div><!--v-repeat-->
```

# さーて0.11.4にしてみるかー

とするとどうも出力が・・・

```output.html

      <div></div><div></div><div></div><!--v-repeat-->
```
あれーーーー
どうなってんのーーー

しばらく格闘して0.11.2以上でこうなることはわかったけど
どうすりゃいいのかわからんと思ってつたない英語で質問してみた。

https://github.com/yyx990803/vue/issues/646

~~ふむ、`<span>`は外出しにして`<content>`に挿入するように書き換えよということ~~
↑2014/12/20：勘違いしておりました。contentは必要ありませんでした

[公式ブログ](http://vuejs.org/2014/12/08/011-component/#Where_Does_It_Belong?)情報もあわせて教えてもらった。

### 解決編


```html
<body>
    <div id="sample">
      <div v-repeat="items" v-component="item">
      </div>
    </div>
    <template id="item">
      <span>baz</span>
    </template>
</body>
```


```app.js
var Vue = require("vue")

// 子要素はコンポーネント化
Vue.component("item", {
  template : "#item" // テンプレート利用
})

var app = new Vue({
  data :{
    items : [
      {v :1},
      {v :2},
      {v :3},
    ]
  }
})

app.$mount("#sample")
```

こうすることで確かに期待通り動いた！

これ困る人いるんじゃないかなあ・・・

warn出してくれたらいいのにと思ってコード追っかけてみたけどさすがにどこの変更の影響なのかは特定できず。。。

### もうちょっと試した

[@tejitakさんからコメント](http://qiita.com/suisho/items/62f9572bb40c3c3ea378#comment-f27514d091e5482ec57e)を受けて\<content>についてもうちょっと試してみた。

上記の例では簡略化のため`<span>`には何もつけていなかったが、子要素にディレクティブではcontentを使う/使わないが変わってきそうだった。



#### まず\<content>を使う例
http://jsfiddle.net/suisho/xetpretq/5/

```html
    <div v-repeat="items" v-component="item" >        
        <span v-text="str"></span> 
    </div>
    <template id="item2">
        <content></content>
    </template>
```
こんな具合。
出力は

```
<span></span>
```

となってしまいディレクティブは無視されてしまう。

#### template側にデータをつける
http://jsfiddle.net/suisho/xetpretq/4/

```html
    <div v-repeat="items" v-component="item" >
    </div>
    <template id="item2">
      <span v-text="str"></span> 
    </template>
```
こんな具合。
これだとちゃんとディレクティブが効いてくれた。

```
  <span>baz</span> 
```

### まとめ
componentのディレクティブを効かせるためには、やっぱり`<template>`側にディレクティブ付きの子要素を書く必要がある模様。
`<content>`はそういうのが必要無くinjectしたいときに使うと良いのだろう。




## どうでもいいはなし
vue 0.11.4で`Vue.config.debug=true`たいへんべんり。今までのデバッグしづらさが一気に解消した感じ。



