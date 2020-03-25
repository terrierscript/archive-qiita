例えばVue.jsでモーダルを実装し、それを開くフックは別なところに引っ掛けたい、みたいなパターンでどうするか？

# EventBusを利用する

この解決として、EventBusというのが公式に提案されている

* [親子間以外の通信](https://jp.vuejs.org/v2/guide/components.html#%E8%A6%AA%E5%AD%90%E9%96%93%E4%BB%A5%E5%A4%96%E3%81%AE%E9%80%9A%E4%BF%A1)

`new Vue()`でインスタンスを生成し、それを仲介してイベントを疎通しあう、という物になる。
windowグローバルにイベントを飛ばしたりするのと実質は変わらないが、ある程度キレイに実装出来そうだと感じる。

# Modalでのサンプル

## ベース
今回の基礎となるModalEventはこういう感じでシンプル。

```ModalEvent.js
import Vue from "vue"

export const ModalEvent = new Vue()
```

今回はModalの`vue-js-modal`を利用する。

```html
<!--HelloModal.vue -->
<template>
  <modal name="hello-modal">
    <div class="container">
      Hello!
    </div>
  </modal>
</template>

<script>
import Vue from "vue"
import VModal from "vue-js-modal"
// ModalEventをimportする
import { ModalEvent } from "./ModalEvent"

Vue.use(VModal)

export default {
  created(){
    // created / mountedでMountEventをlistenさせて、modalを開く
    ModalEvent.$on("openModal", () => {
      this.$modal.show("hello-modal")
    })
  }
}
</script>
```

## 別々にmountしたvueコンポーネント

```index.html
   :
    <div id="app">
      <modal-open-button></modal-open-button>
      <hello-modal></hello-modal>
    </div>
   :
```

Modalを開く側のボタンはこんな具合になる

```html
<!--ModalOpenButton.vue-->
<template>
  <div>
    <button @click="onClick">Open from Vue</button>
  </div>
</template>

<script>
import { ModalEvent } from "./ModalEvent"

export default {
  methods:{
    onClick(){
      // クリックしたらイベントを発火
      ModalEvent.$emit("openModal")
    }
  }
}
</script>
```

これで`ModalEvent`を介してイベントが疎通して開くようになる

## jQueryの場合

あんまり用途は無い（ことが望ましい）が、jQueryから発火するのも出来るようになった。

```index.html
    <div id="app">
      <div>
        <button id="jquery-external-button">Open from jQuery</button>
      </div>
      <hello-modal></hello-modal>
    </div>
```
```js
import $ from "jquery"
import { ModalEvent } from "./ModalEvent"

$(function(){
  $("#jquery-external-button").click(function(){
    ModalEvent.$emit("openModal")
  })
})
```

## vanillaなjsからの発火

jQueryとさして変わらないが、もう一歩踏み込んでjQueryも使わない場合。
あまりキレイではないが、`window`にModalEventを渡して呼び出せる。

```js
import Vue from "vue"
export const ModalEvent = new Vue()
window.ModalEvent = ModalEvent
```

```index.html
    <div id="app">
      <div>
        <button id="vanilla-external-button">Open from Vanilla</button>
      </div>
      <hello-modal></hello-modal>
    </div>
    <!-- built files will be auto injected -->
    <script>
      document.onreadystatechange = function(){
        var elm = document.getElementById("vanilla-external-button");
        elm.onclick = function(){
          window.ModalEvent.$emit("openModal");
        }
      }
    </script>
```

# おまけ：もうちょっとEventを整然とする

無造作にEventを飛ばすのを防ぐのに、`methods`で限定しておくと少しは整然とするであろう。

```ModalEvent.js
import Vue from "vue"

const OPEN_MODAL = "openModal"
export const ModalEvent = new Vue({
  methods: {
    emitOpenModal(event){
      this.$emit(OPEN_MODAL, event)
    },
    onOpenModal(cb){
      this.$on(OPEN_MODAL, cb)
    }
  }
})

```

# 参考資料
* https://medium.com/@jilsonthomas/create-a-global-event-bus-in-vue-js-838a5d9ab03a
* https://medium.com/@andrejsabrickis/https-medium-com-andrejsabrickis-create-simple-eventbus-to-communicate-between-vue-js-components-cdc11cd59860
* https://alligator.io/vuejs/global-event-bus/
