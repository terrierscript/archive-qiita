
# Higher-order Components

ReduxなどReact関連のライブラリで時々出てくるHigher-order Componentsについて色々わかんなくなったので調べたことをメモ程度に留めてまとめる

通称 HoCとかHoCsとか略されるが、この記事では Higher-order Components、HoCsで統一しておく。

## Higher-order Components(HoCs)とは？
語の元となったのは[高階関数](https://ja.wikipedia.org/wiki/%E9%AB%98%E9%9A%8E%E9%96%A2%E6%95%B0)(Higher-order function)であると思われる。

高階関数は、「関数を引数にとり、関数を戻り値とする」ような関数の事。

HoCsも同様で、「Componentを引数にとり、Componentを戻り値とする」ような関数（または実装パターン）の事 [^1]

[^1]: 「関数を適応したComponent」を指している場合もある。若干このあたり理解甘い。

## なぜこんな概念が出てきたのか？

この記事が良く出される

[Mixins Are Dead. Long Live Composition](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750)

ざっくり要約すると

- ES2015のclass記法においては、mixinは使えなくなった
- 今後、classのmixinが取り込まれる可能性はある
- けど、mixinはそもそも色々モロい部分もある
- 多分Reactに復帰することも積極的ではなさそう
- そこで、Higher-order Componentsですよ。という話


## Mixinって何？またはなんだっけ？

ちょっと忘れているのでおさらい。

- [React.jsのmixinについて](http://qiita.com/koba04/items/55d28f0aba4d1f53e735)
- [Mixins(公式)](https://facebook.github.io/react/docs/reusable-components.html#mixins)

Reactの`createClass`で利用できる、いわゆるMixin。
継承をせずに、Componentに同一の挙動を与えるものだった。


## ざっくりどんな実装をしているのか？
https://gist.github.com/sebmarkbage/ef0bf1f338a7182b6775
HoCsの初出（らしい）こちらのGistがわかりやすい。

噛み砕いてコードリーディングしてみる

- 登場するのはこの２つ
  - Enhancer = コンポーネントを受け取ってコンポーネントを返す関数。HoCs
  - MyComponent = 例として適応される側のコード
- Enhancerを見てみる
  - `Enhancer = (ComposedComponent) => class extends Component...`という部分がかなり特殊に見える。
    - 「[ワンライナーでやってるけどこういうことだよ](https://gist.github.com/sebmarkbage/ef0bf1f338a7182b6775#gistcomment-1392065)」と展開後のコメントで書かれている
    - やっていることは「`ComposedComponent`」という元のComponentを受けて、匿名のクラス（`class extends Component`）を返している
    - この時に、`componentDidMount`だったり、`state`の利用だったりをして、mixinがやっていたようなことをやっている
- `export default Enhance(MyComponent)`で、HoCs適応済みのComponentを返している。

ということで、やっていること自体はとても簡素。好みによっては`props.children`でも代用出来るかもしれない（多分コードの簡素さと、DOMが深くならないとかの理由でclassでwrapしてる方を採用しているんだと思う）。

## 各ライブラリにおける実例

HoCsは、Reduxやらreact-routerやら有名ライブラリで使われている。以下見つけられた例。

- [react-redux#connent](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options)
  - 例：`connect(null, actionCreators)(TodoApp)`　というHoCsを提供している
- [react-router#withRouter](https://github.com/reactjs/react-router/blob/master/docs/API.md#withroutercomponent)
  - `withRouter`という関数が、HoCs。
- [react-i18next#translate](https://github.com/i18next/react-i18next#translate-hoc)
  - i18nをするライブラリ。`translate`をHoCsとして提供している 
- [baobab-react#higher-order-components](https://github.com/Yomguithereal/baobab-react#higher-order-components)
  - 名前まま。
  - Qiitaにもこれを利用した実例の記事があった[baobab-reactを使って軽量fluxを回す](http://qiita.com/airtoxin/items/00b92b221bc0043c93b1)

### HoCs に関する記事、参考文献など

- [Higher-Order Componetを使ってES6 Class記法でmixinっぽいことをする](http://qiita.com/pirosikick/items/9a32933e1afcb4a2b38f)
  - 探した限り唯一ぐらいの日本語記事
- [Higher Order Components: Theory and Practice](http://engineering.blogfoster.com/higher-order-components-theory-and-practice/)
- [React Fundamentals: Higher Order Components (replaces Mixins)(動画)](https://egghead.io/lessons/react-react-fundamentals-higher-order-components-replaces-mixins)
- [What are Higher Order Components?](https://medium.com/@franleplant/react-higher-order-components-in-depth-cf9032ee6c3e#.855f4r6xc)
- [Structuring React Applications: Higher-Order Components](http://jamesknelson.com/structuring-react-applications-higher-order-components/)
- [React Higher Order Components in depth
](https://medium.com/@franleplant/react-higher-order-components-in-depth-cf9032ee6c3e)
	- [日本語訳記事](http://postd.cc/react-higher-order-components-in-depth/)
