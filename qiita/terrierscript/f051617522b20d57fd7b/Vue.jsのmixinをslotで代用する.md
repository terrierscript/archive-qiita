Vueにはmixinという機能がある。

https://vuejs.org/v2/guide/mixins.html

同一の機能を複数のコンポーネントに適応させるというものだ。

共通の機能を提供する便利なものだが、あまり楽観的に使えるものではない。

例えばReactでも過去には存在していたが、既に廃止されている。
なぜ廃止されたかは理由を読めばmixinのどのような点が危険な点かが見えてくるだろう。

* https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html
* https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750

ざっくりと要約するとこのへんが上げられている

* 暗黙の依存関係を導入してしまう
* 名前の衝突を起こす
    * 複数のmixinがあり、それらがプロパティを上書きするために名前が衝突してしまう
* 拡張のたびに雪崩的に複雑さを引き起こす

そこでVue.jsでもmixinを使わない方法を今回は記述したい

# どうするか？

Reactでは高次コンポーネントを導入することで解決している。
Vue.jsにおいては`<slot>`を使えばこれは多くの場合解決出来そうだ。

（render propsによる方法もあるが、こちらは下記の記事におまかせしたい。
[Vue.jsにおけるRender PropとScoped Slotsについて](https://qiita.com/seya/items/abccdd7b502535844659) ）

今回は例としてマウスイベントをトラッキングする処理を考える。

## まずはmixinを使った実装

マウスイベントをとるmixinはこんな感じ。

```js
// mouseMixin.js
export default {
  data() {
    return {
      x: 0,
      y: 0
    };
  },
  methods: {
    mousemove(e) {
      this.x = e.clientX;
      this.y = e.clientY;
    }
  }
}
```

利用側はこんな具合になる。

```html
<!-- Item -->
<template>
  <div @mousemove="mousemove">
    <div>Item with mixin</div>
    <div>x: {{ x }}</div>
    <div>y: {{ y }}</div>
  </div>
</template>

<script>
import mouseMixin from "./mouseMixin"

export default {
  mixins: [ mouseMixin ],
};
</script>
```

呼び出しはあとは下記のような感じになる。

```html
<!-- Parent.vue -->
<template>
  <div>
    <Item />
  </div>
</template>

<script>
export default {
  components: {
    Item,
  }
};
</script>
```

先のmixinの問題点として上げたとおり、`x`,`y`,`mousemove`が暗黙的になっており、定義を探すのに`mouseMixin`をたどることになる。
mixinが一つぐらいなら問題になることは少ないかもしれないが、これが増えてくると厳しくなってくるだろう。

## slotにする

まずmixinを利用する側。
`x`と`y`をもらうだけで至ってシンプル

```html
<!-- Item.vue -->
<template>
  <div>
    <div>Item with scope</div>
    <div>x: {{ x }}</div>
    <div>y: {{ y }}</div>
  </div>
</template>

<script>
export default {
  props: {
    x: Number,
    y: Number
  }
};
</script>
```

次にMouseEventの処理をするMouseEvent.vueを分離したもの。
mousemoveを取得して`slot`に対して`x`と`y`を渡す。

```html
<!-- MouseEvent.vue -->
<template>
  <div @mousemove="mousemove">
    <slot :x="x" :y="y"></slot>
  </div>
</template>

<script>
export default {
  data() {
    return {
      x: 0,
      y: 0
    };
  },
  methods: {
    mousemove(e) {
      this.x = e.clientX;
      this.y = e.clientY;
    }
  }
};
</script>
```

最後に組み合わせるところ。`<MouseEvent>`の子として`slot-scope`を定義し`Item`に渡す

```html
<!-- Parent.vue -->
<template>
  <div>
    <MouseEvent>
      <div slot-scope="{x, y}">
        <Item :x="x" :y="y" />
      </div>
    </MouseEvent>
  </div>
</template>

<script>
export default {
  components: {
    MouseEvent,
    Item,
  },
};
</script>
```

## まとめ

* Vueだとscopeを使ってしまう場合、記述量が増えて冗長になってしまうのは否めない。
    * 願わくばもう少し簡易に書けると嬉しいが・・・・
    * そしてscope-slotの使い方は覚えづらいので慣れが必要な感じ
* とはいえ、それぞれ暗黙的な呼び出しは無くなったり、複数組み合わせても名前の衝突が起きないことは利点だろう。
* mixinを避けたい場合はそこそこあるので、使えるパターンだろう。

