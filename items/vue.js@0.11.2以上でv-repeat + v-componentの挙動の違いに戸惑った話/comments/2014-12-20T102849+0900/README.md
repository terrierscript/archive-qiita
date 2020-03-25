こんにちは、上記の記事の例で子コンポーネントにspanタグをもたせていますので、親のcontentタグは不要ではないでしょうか？（v0.11.4で以下で動きます。）

```html
    <div id="sample">
      <div v-repeat="items" v-component="item"></div>
    </div>
    <template id="item">
      <span>baz</span>
    </template>
```

もしcontentタグを使う場合は、親で受け取ったノードを子コンポーネントの特定の位置にinjectするものだと思いますので、以下のようになるかと思います。

```html
    <div id="sample">
      <div v-repeat="items" v-component="item">
          <span>baz</span>
      </div>
    </div>
    <template id="item">
      <content></content>
    </template>
```

