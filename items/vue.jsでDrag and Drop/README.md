
vue.jsでdrag and dropやりたかったけど[これ](http://codepen.io/safx/pen/dasnt)ぐらいしかサンプルが見つからず、割りと苦戦したのでメモ

# できたもの
- 動くやつ：http://suisho.github.io/example-vuednd/
	- ソース：https://github.com/suisho/example-vuednd
		- browserifyも利用。楽しい
- ２つリストが並んでいてアイテムをリスト感で移動できるようなやつ
- 今回は`item`と`list`というコンポーネントをつくる

## 感想
- vue.jsちょこちょこ触ってはいたけど、DnDの場合そこまで恩恵が大きくなかったかも？
- vueはそんなに色々めんどうみてくれるわけではないので、HTML5のふつーのDnDの知識が普通に必要


# 以下細かいところ
## どんな感じで設計するか

* 大きく下記３つのコンポーネントができる
	* item
		* ドラッグ対象なる要素
	* list
		* itemを内包するリスト。ドラッグはリストからリストになされるとする
	* app
		* listを内包するアプリケーション本体
* 通常のDnDのようにドラッグイベントを介してデータをやりとりするのはどうもあまりうまくいかなそうだった
	* dataTransfer.setDataでjsonを受け渡すのが微妙そうという理由
* なので親となるappで、データの移動などを管理させることにした
* イベントも下記のように伝播させて、appで処理させる。
	* item -> list -> app 


## テンプレート
(気分でtemplateタグを使ってしまっていますが、あまり意味はありません。)

#### item

```html
	<template id="item-template">
      <div class="item" draggable=true v-class="dragging : isDragging">
        <span v-text="id"></span>
        <span v-text="name"></span>
      </div>
    </template>
    
```
- 普通のDnDと同じように`draggable`をtrueにしとく
- ドラッグ中にopacityをかけたいがために`class= dragging: isDragging`をつけておく（あまり重要でない）

#### list（itemを内包するやつ）
```html
	 <template id="list-template">
      This is {{name}} List
      <div class="items">
        <div v-repeat="items" v-ref="itemView"
              v-component="item"
              v-on="dragstart:onDragStart,
                    dragend:dragEnd" 
              v-with="item"></div>
      </div>
    </template>
```
- 基本はただリスト並べるだけのcomponent。
- itemにあたる部分に `v-on`でdragstartとdragendを仕掛けておく

#### app（listを内包する奴）

```html
    <div id="container">
      <div class="list" 
          v-repeat="lists"
          v-with="list"
          v-ref="listView"
          v-component="list" 
          v-on="dragover: onDragOver, 
                drop: onDrop">
      </div>
    </div>
```
- listのコンポーネントには`v-on`でdragoverとdropをひかけておく

## コンポーネント
### どんなデータ入れておくか
vue.jsはarrayやらobjectやらをデータとして使う。今回はこんな具合

```js

var loadedData = [{
  name : "list1",
  items : [{ 
 	name: "foo",
	id : 1
  }]
}, {
  name : "list2",
  items : []
}]
```

#### itemのcomponent

```js
  Vue.component("item", {
    template: "#item-template",
    data : {
      isDragging : false
    },
    methods : {
      onDragStart : function(e){
        e.dataTransfer.dropEffect = "move"
        this.isDragging = true
        this.$dispatch("itemDrag", this)
      },
      dragEnd : function(){
        this.isDragging = false;
      }
    }
  })

```
- 基本dragging中を検出だけ
- isDraggingはcssのclass変更のため
- データの移動などは伝播させて親にまかせてしまう


#### listのcomponent

```js
Vue.component("list", {
  template: "#list-template",
  created : function(){
    this.$on("itemDrag", this.onItemDrag)
  },
  methods : {
    onDragOver : function(e){
      e.preventDefault()
    },
    onItemDrag : function(itemVM){
      // 伝播されたイベントをさらに伝播
      this.$dispatch("listDrag", this, itemVM)
    },
    onDrop : function(e){
      // 自身に発火されるイベントも伝播
      this.$dispatch("listDrop", this)
    },
    // itemに関するadd, delete処理
    addItem : function(item){
      this.items.push(item)
    },
    popItem : function(item){
      return this.items.splice(item.$index, 1)[0]
    }
  }
})

```

* こちらも同様イベント処理は伝播させる
* onDragOverはちゃんとpreventDefaultしてあげないといけない
* addItem/popItemは、ちょっとappから直接いじるのきもいかもと思って作ったぐらいなもの。


### App（起点）になるVue
```js
window.app = new Vue({
    el : "#container",
    data : { 
      // 現在どのリストとitemがアクティブになっているかを保持する
      dragActiveList : null,
      dragActiveItem : null,
      // data
      lists : loadedData
    },
    created : function(){
	  // 伝播されたイベントを受け取る
      this.$on("listDrag", this.onListDrag)
      this.$on("listDrop", this.onListDrop)
    },
    methods : {
      // drag時に、アクティブを保持
      onListDrag : function(listVM, itemVM){
        this.dragActiveList = listVM
        this.dragActiveUser = itemVM
      },
	  // drop時にデータを移動
      onListDrop : function(listVM){
        this.moveItem(this.dragActiveList, listVM, this.dragActiveUser )
      },
	  // 移動処理
      moveItem : function(listFrom, listTo, item){
        if(listFrom.$index === listTo.$index) return
        item.dragEnd()
        listTo.addItem(listFrom.popItem(item))
      }
    }
  })
``` 
* list / itemで伝播させてきたイベントを受け取り、データを移動する。
* データさえ移動しちゃえばあとはvueがやってくれるところはだいぶ恩恵を感じる部分

