
[vue.js@0.11.2以上ではrepeatするcomponentをtemplate化しないとけない](http://qiita.com/suisho/items/62f9572bb40c3c3ea378)ということがわかったので子要素をtemplate化したはいいものの、svgをtemplateにしたことでまた微妙な問題を踏んでしまった。



# `<image>`が`<img>`にされるよ！
`<svg>`と`<image>`を使うようなコンポーネントを作りたかったのでこんな感じでやると

```html
<html>
  <body>
    <svg id="sample">
      <g v-repeat="items" v-component="item"></g>
    </svg>
    <template id="item" >
      <image width=50 height=50 v-attr="'xlink:href': v"></image>
    </template>
    <script src="build.js"></script>
  </body>
</html>
```

```js
var Vue = require("vue")
Vue.component("item", {  
  template : "#item",
})

var app = new Vue({
  data: {   
    items : [
      {v : "http://cdn.qiita.com/assets/siteid-reverse-cd0519106e5814e34d402b3e821820cf.png"},
    ]
  },
  methods: {
  }
})

app.$mount("#sample")

```
↓こんな感じになってしまった。

```output.html
<g>
  <img width="50" height="50" xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="http://cdn.qiita.com/assets/siteid-reverse-cd0519106e5814e34d402b3e821820cf.png">
</g><!--v-repeat-->
    
```

ぶへ。なんでー。
となったが割りとさっくり解決できた。

### 解法1: templateに直接stringで書く
いや。これはちょっと個人的には微妙。

```js
Vue.component("item", {  
  template : '"<image width=50 height=50 v-attr="\'xlink:href\': v"></image>"',
})
```
### 解法2: scriptタグで書く。
というかやっていてそもそも\<template>を使うとかいきがったことをしているから悪かった。
(そもそも\<template>使うのいいんだっけ？となったけど[このPull Request](https://github.com/yyx990803/vue/issues/335)で取り込まれているので問題なさそう）

なので\<script>で記述すれば問題なかった。

```html
    <script type="x-template" id="item" >
      <image width=50 height=50 v-attr="'xlink:href': v"></image>
    </script>
```
### 解法3: templateの親要素をsvgにする
他には親要素をsvgにすればちゃんと出てくれる

```html
<svg id="item" ><!-- display:noneはサンプルなので省略 -->　
   <image width=50 height=50 v-attr="'xlink:href': v"></image>
</svg>
    
```

### 解法4: templateにnamespaceをつける
更にこんなんでもOKそう。

```html
    <template id="item" xmlns="http://www.w3.org/2000/svg">
      <image width=50 height=50 v-attr="'xlink:href': v"></image>
    </template>
```

## どういうことか。

これはvueの話ではなくブラウザ側の挙動の問題の模様。
(chromeで調査したので他だとどうなるかは未調査です)

こんなサンプルを用意すると、以下のような出力がされる。

```html
<div id="foo">
  <image></image>
</div>
<script>
  console.log(document.querySelector("#foo").innerHTML)
</script>
```
```output
<img>
```

つまりvueがこれと同じようなことをしていて、chromeさんがparseする段にsvgとみなせないような`<image>`タグを`<img>`として扱ったということだろう

ちゃんとした検証はしていないが解法2が一番カバー率が高いのではと思っている。


## ちなみにxlink:hrefのはなし。
「あれ、v-attrにxlink:hrefって使えなくね？」って思ったけど
[このissue](https://github.com/yyx990803/vue/pull/354)にすでに上がっていて`v-attr="'xlink:href':url`にすればよいよーと言われていた。

助かった。

