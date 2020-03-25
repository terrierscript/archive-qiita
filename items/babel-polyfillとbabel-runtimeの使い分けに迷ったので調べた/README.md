
2018/03/20追記
この資料はbabel-preset-envに`useBuiltIns: usage`などが入るより前の資料になります。詳しくは最新版のbabel-preset-envをご確認ください
https://github.com/babel/babel/tree/master/packages/babel-preset-env#usebuiltins

---
---

2016/11/30追記: **更に詳細に調べた続編を書きました 
=> [babel-polyfill / babel-runtime の代わりにcore-jsを直接使うのはアリかナシか](http://qiita.com/inuscript/items/1993b6b18a76fcb70dc4)**

---
---

babelの変換において、古いブラウザでもある程度動かすようにしてくれるための[babel-polyfill](https://babeljs.io/docs/usage/polyfill/)。

しかし、調べてみると[babel-plugin-transform-runtime](http://babeljs.io/docs/plugins/transform-runtime/)というのもあった。
あまりドキュメントも充実しておらず何が違うのかわからなかったのでざっくり調べた。

# babel-plugin-transform-runtime
polyfill対象に該当するコードだけが未対応なブラウザでも動くように変換される。

例えば下記のような変換がなされる

```before.js
Object.entries({"foo": "baz"})
```

```after.js
(0, _entries2.default)({
    "foo": "baz"
});
```

- 利点
  - 必要な変換のみをするので、対象になるコードが少なければ、ファイルサイズは小さくすむ
	  - 変換する部分が多すぎると、肥大化する可能性がある（はず）
  - global展開せず、それぞれの場所で閉じて処理されるので、複数エントリファイルから呼び出していても問題無い
  - global汚染をしない
- 欠点
  - 最新ブラウザでも関係無くpolyfillを噛ませたコードになるので、場合によっては実行速度が遅くなる場合がある（はず）
  - [ドキュメント](http://babeljs.io/docs/plugins/transform-runtime/)が不親切

# babel-polyfill

~~単なるlibraryとして、下記のように読み込んで利用する~~
polyfillライブラリとして、通常のコードベースとは別途読み込む（2016/11/30追記：babe-importの読み込みと、コードベースを同時に行う事は推奨されない方法でした。誤解を生む書き方だったので、サンプルコードと説明を訂正しました）

```js
import "babel-polyfill"
```

```js
Object.entries({"foo": "baz"})
```

これで、core.jsをベースとしたpolyfillが使う・使わないにかかわらず全て読み込まれる。

- 利点
  - polyfillナシで対応がされている最新ブラウザにおいて、ブラウザネイティブが即時に利用されるので、動作が比較的早くなる（はず）
  - polyfillの対象になるコードを大量に含むような場合は、ファイルサイズが`transform-runtime`よりも縮む（はず）
  - [ドキュメント](http://babeljs.io/docs/usage/polyfill/)がそれなりにある
- 欠点
  - ファイルサイズが大きくなる
      - [core.min.js](https://github.com/zloirock/core-js/blob/master/client/core.min.js) を見ると77KB程度？
  - 複数のファイルから読み込むような環境だとエラーになる（後述）
      - ＝ entryファイルが一つだけになる場合でないと扱いづらい
  - polyfillの宿命だが、global汚染が発生する

### 問題点：複数のファイルから読み込むような環境だとエラーになる

たとえば`script1.js`、`script2.js`の２つでそれぞれbabel-polyfillを読んでいるとする

```html
<!-- どっちも import "babel-polyfill" しているとすると・・・ -->
<script type="text/javascript" src="./script1.js">
</script>
<script type="text/javascript" src="./script2.js">
</script>
```

こんなエラーが出る。

```
Uncaught Error: only one instance of babel-polyfill is allowed
```

## どう使い分ければ？
ざっくりこんな分けでいいと思われる

- 複数のエントリポイントが存在するなら
  - babel-plugin-transform-runtime
- SPAみたいな環境が用意できるなら
  - babel-polyfill

もうちょっと別な要因も気にするなら、こんな観点もありうる

- polyfill利用がすごく少ない場合なら
  - babel-plugin-transform-runtime
- polyfillを利用している箇所がすごく多いなら
  - babel-polyfill or core.jsを直接読み込んで利用する
- コードの実行対象が、だいたい最新ブラウザなら
  - babel-polyfill
- コードの実行対象が、だいたいレガシーブラウザなら
  - babel-plugin-transform-runtime
