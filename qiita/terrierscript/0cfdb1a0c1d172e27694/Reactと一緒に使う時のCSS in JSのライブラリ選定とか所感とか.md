
# CSS in JSの基礎
* 原点は[こちらのスライド](https://speakerdeck.com/vjeux/react-css-in-js)がよく挙げられる。
* いわゆる「CSSのあらゆる問題をJSで解決する」という感じのもの。
* 先行の記事としてはこのような感じ
  * [CSS in JS(Elm)したら想像以上に良かった](http://jinjor-labo.hatenablog.com/entry/2016/05/30/165816)
  * [Free-Style のススメ ~ CSS Modules は解決策ではない](http://qiita.com/kimamula/items/2a5e69269e77ca85008e)
* とりあえず今回はReactと一緒に使う前提のことを考える

## CSS in JS ライブラリの実装系統
CSS in JSライブラリもかなり色々あるが、だいたい下記の観点で分類できそうだった。

### スタイルの再現に関する実装
* `<style>`タグを生成して、`<head>`にinsertする実装パターン
  * 昨今のスタンダードなライブラリの使っている手法
  * CSSの疑似要素や`@media` queryもだいたい使える
* InlineにCSSを展開する実装パターン
  * わりと絶滅危惧種っぽい（開発止まっていたり）
  * そもそも素のreactがこの機能を提供しているので、あんまりライブラリの旨味はない

### スタイルの記法に関する実装
* JSに記載
  * JSのObjectとして記載
  * template StringなどでCSSを記載
  * transpileを前提にした独自記法
* 別ファイルに記載
  * CSSに記載

## Library比較
今回は、下記を前提として比較した。

* `:hover`など疑似要素利用可能
* `@media`、`@fontface`、`@keyframes`利用可能
* クラス名の衝突回避をしてくれるもの
* template stringとしてCSSを書くものは除外
  * パースエラーの検出など考えると頭痛いため
* Reactへの依存性が少ないならなお良し
* 参考資料はこのへん。ただし、割りと既に古かったりするので、網羅性だけを見たほうが良さそう
  * https://github.com/MicheleBertoli/css-in-js
  * https://github.com/FormidableLabs/radium/tree/master/docs/comparison


### Aphrodite

`className`に記載してるクラスは、ハッシュ化されたsuffixがつけられ、名前空間が重複しないようになる (参考：[Aphrodite Output tool](https://output.jsbin.com/qoseye?code=%7B%0A%20%20red%3A%20%7B%0A%20%20%20%20color%3A%20%27red%27%2C%0A%20%20%20%20%27%3Ahover%27%3A%20%7B%0A%20%20%20%20%20%20opacity%3A%200.8%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D) )

```js
import { StyleSheet, css } from 'aphrodite';
const style = StyleSheet.create({
  red: {
    color: 'red',
    ':hover': {
      opacity: 0.8
    }
  },
})
const Label = ({children}) => {
  <span className={css(styles.red)}>
    {children}
  </span>
}
```

**利点**

* `<style>`の生成関連が、一番気使っている感じがする
* animation周りが一番書きやすい
* Star数が多く、人気高め
* `!important`をつけて、他のCSSの影響を殺す事も出来る
  * `aphroidte/no-important`を利用すれば、これを無効化することも出来る 
* autopreifxerを内部で自動的につけてくれる
* 開発が個人でなく組織なのでちょっとだけ安心度高い
* hypertermなどで実績あり

**欠点**

* `className`以外へのCSS適応を許さない姿勢なので、resetなどは別途考えないといけない
* 子要素のCSSを指定する方法も限定されているので、わりと厳格にCSS in JSに書いていく必要がある 


### [Free-style](https://github.com/blakeembrey/free-style)

```js
const FreeStyle import 'free-style'

// Create a container instance.
const Style = FreeStyle.create()

const LABEL_STYLE = Style.registerStyle({
  color: 'red',
  '&:hover': {
    opacity: 0.8
  }
})
const Label = ({children}) => {
  <span className={LABEL_STYLE}>
    {children}
  </span>
}
```

**利点**

* React非依存
* selectorの指定は、それなりの柔軟さがある

**欠点**

* aphroditeに比べると、手数の多さが気になる

### [glamor](https://github.com/threepointone/glamor)

`style`や`hover`などを関数定義して展開させるという独自の記法。なかなか悪くないと感じた。
`aphrodite`からの乗り換えは楽に出来るように作られているので、必要になったら乗り換えようかな、という感覚。

```js
import { style, hover } from 'glamor'
const Label = ({children}) => {
  <span   
    {...style({ color: 'red' })} 
    {...hover({ opacity: 0.8 })}>
    {children}
  </span>
}
```

**利点**

* `StyleSeet.create`的な記法が無いので、手数が少ない
* `glamor/reset`、`glamor/aphrodite`、`glamor/jsxstyle `など、他のライブラリに記法を近づけるようなものも提供している
* `select("ul li")`のような関数を使うと、子要素へstyleを適用出来る
  * このあたりの使い勝手は良さそうだが、注意深く使わないと、CSSの厄介な部分を引き出しそう

**欠点**

* べったりstyleに記載
* 内部的な実装として、`[data-hogehoge]`というCSSセレクタに変換するため、パフォーマンス大丈夫だろうか？と心配になる。


## その他の実装や手法（見送り）
### ライブラリ無し(素react)
* `@media`などは我慢できるが、`hover`まで`onHover`などで実装しなきゃいけないのは辛すぎるのでボツ

### [Radium](https://github.com/formidablelabs/radium)
* Star数はかなり多い
* Reactに強依存。React以外で使う事は想定されてなさそう。
* decoratorを利用しているが、decoratorの仕様策定がまだ不安定な情勢なので、かなり怖い
* decoratorを利用しない場合の動作がどうにも不安定（Stateless Functional Componentと相性悪い？）

### [cssx](https://github.com/krasimir/cssx/)
* トランスパイラー前提として、独自記法で書いていく
* さすがに挑戦的すぎたので見送った。
* 方向性は面白いが、この方向性だと、流石に開発体制の基盤がほしい
* facebookがまともにこの方向性のものを作るなら、ちょっと興味があるかもしれない。

### [jss](https://github.com/cssinjs/jss)
* cssinjsというそのものズバリな名前で作成されている
* pluginなどがあって面白い。
  * 短所としては、結構plugin入れないと使い物にならない感じあるかもしれない。
* keyframes周りがあまり強くない印象

### [CSS Modules](https://github.com/css-modules/css-modules)
* 知名度高し。導入敷居は低い。
* 厳密にはCSS in JSとも言いづらい。広義ならCSS in JSと言えそう。

* 利点
  * Webpackとは相性が良い
  * 資料が豊富
  * 既存のCSSを完全にそのまま使える

* 欠点
  * Webpackへのロックインが現状だとだいぶきつい
	    * 一応Browserifyなどがんばればイケるが・・・
  * `import "foo.css"`のような形式だと、testツールで辛みが多い
	    * testに関わらず、上記のような記法は避けれるなら避けたい

* 軽く使ってみるとやっぱりテスト周りが主に辛い＆CSS in JSともずれる、というところで見送った。

### [jsxstyle](https://github.com/petehunt/jsxstyle)

* `<Flex>`や`<Block>`などのコンポーネントまで提供している。
* `curry`などの機能は面白そう。
* なんとなく自分が使いたいものとはズレるものと見えたので余り調べてない
* もしかすると組み合わせて使えるのかも？

### [ReactShadow](https://github.com/Wildhoney/ReactShadow)

* Shadow Domを使う変わり種。
* 将来性考えると面白いかもしれない

### [react-look](https://github.com/rofrischmann/react-look)
* React Nativeにも対応しているらへんが魅力
* 今回自分の要件からは外れるので見送った


# CSS in JSについての雑感など

## 結局どれを使いそうか？ → Aphrodite

個人的には、Aphroditeが一番記法的に手数がそれほど多くなく、必要十分な感じだった。

glamorやFreeStyleにあるような子要素へ影響を及ぼすセレクタを書けない事やresetを別途管理しなければならないことは弱点でもあるが、その分CSSで無理な解決をさせず、コンポーネントが適切に分解されることが期待できそうと感じる。

「やっぱりCSS in JS辛い！CSSに戻す！」となったときも、記法の制限が多い分捨てやすいだろうとも感じるし、逆に「制限がきつすぎる！」という場合だったらglamorに乗り換えるのも容易そうだった。

## CSS in JSはそもそもやるべきか？

2016年段階で、万人に向けて「是非どうぞ！」と勧めれるような手法ではないだろうと思う。ただ、現実的にトライできるフェーズまでは来ているのかなと感じた。
個人的にもいくつかの動機付けがあった。
このあたりが共感出来るなら、トライする価値があるかもしれない。

* 小規模なSPA的な感じで、CSS in JSは相性は悪くなかった。
* SCSSで分離していたcss部分をJSにまとめれると嬉しい状況があった
  * webpackでの結合を避けたかった
* デザイナーの理解を得ることが出来た **（重要）**
* デザイン自体は一旦完成して、それをリプレースする状態だったので、そんなに障壁が高くなかった。
* CSSに対するベストプラクティスがまだ無いので、チャレンジしたかった
  * BEMやSMACSSの命名規約で縛る設計方法に限界を感じていた。
  * ここが満足している場合だったら不要なのかも？
* CSSとJSが凝縮された世界というのは Angular2や、Shadow Dom 、Riot.jsのscoped css、vue.jsのモジュール化などちょこちょこ見かけるようになっているので、「そろそろやってみても良いかな」という気分があった。

## 書いてみた所感など

* 書いてみると、キモさや背徳感と、相反して合理性が同居する複雑な気持ち
  * JSXを初めて触った頃の感じによく似ている気がする
* CSSに疎いサーバーエンジニアには概ね好評
* Atomic Designを意識して、Atom（原子）の部分を明確に区切ってCSS in JSでstyle付けして組み立てるのは相性良さそう
  * `<Padding>`などの調整についてはそれ専用のComponentを作る割り切りが合ったほうが良いかもしれない。
* 今まで「うーん、このパーツだけで単体ファイル化しても・・・」と思っていた数行のコンポーネントも、styleもまとめると適度な粒度になった。
* デザイナが全体感を捉えるのに向いてないのでは？というのはまだぶつかってないが、課題感も感じている
  * 多分Atomic Designが抱えている問題の感覚と近そう。うまくStorybookなどを使って解決していきたい。
* モーダルなど、外からの影響を遮断したいようなコンポーネントだったらすごい相性良さそう。
* React関連のライブラリ提供側で、CSSのimportを要求されるのすごく困るので、そういうライブラリはさっさとCSS in JSに移行してくれたらいいのに、と思った。
* [houdini](https://github.com/w3c/css-houdini-drafts)の話とかあるので、将来的にどうなるかはまだまだ未知の世界


