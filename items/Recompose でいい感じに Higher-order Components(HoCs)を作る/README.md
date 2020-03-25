
# [Recompose](https://github.com/acdlite/recompose/)
RecomposeはHigher-order Componentsをサポートするライブラリ。

Higher-order Componentsについては下記記事で軽くまとめた。

- [ReactのHigher-order Component(HoCs)に関するメモ](http://qiita.com/inuscript/items/6eb6076a7e5237a2911a)

## どんなライブラリ？
- HoCsを作成・提供するための便利関数ライブラリ
- 「lodash的なノリで、関数群をまとめている」とのこと 
- 作者は[flummox](https://github.com/acdlite/flummox)や[react-action](https://github.com/acdlite/redux-actions)など、reactやredux関連を多く作っているライブラリの作者。
- まだ 0.20で微妙にAPIは安定してない部分もアリ

# まずはサンプル
## 例： 未サポートブラウザの時にメッセージ出す
説明だけだとわかりづらいので、個人的に一番とっつきやすかった`branch`を使って、特定のブラウザの場合に「お使いのブラウザはサポートされてません」と出すサンプルを書いてみる。

### 普通の`React.Component`で書く（Recomposeナシ）

```js
export const NotSupportedMessage = () => <h3>お使いのブラウザはサポートされておりません</h3>

export default class MainApp extends Component{
  isSupportedPlatform(){
    if(/*ブラウザが未サポートかの判定*/){
      return false
    }
    return true
  }
  render() {
    if(!this.isSupportedPlatform()){
      return <NotSupportedMessage />
    }
    return (
      <div className="SomeApp">
        何らかのアプリケーション
      </div>
    )
  }
}
```

まあこれでも全然問題ない。が、本筋の`SomeApp`の部分までスッとたどり着いていないし、このあたりの処理が肥大化すると色々辛いことになりそうなのが見える。


### Higher-order Componentで書く(Recomposeアリ）

```js
import branch from 'recompose/branch'
import renderComponent from 'recompose/renderComponent'

// branchは、下記のように引数を取る
// branch(判定関数, trueの時, falseの時)

const NotSupportedMessage = () => <h3>お使いのブラウザはサポートされておりません</h3>

const SomeMyApp = () => (
  <div className="SomeApp">
    何らかのアプリケーション
  </div>
)

const isSupportedBrowser = () => {
  if(/*ブラウザが未サポートかの判定*/){
    return false
  }
  return true
}

const withBrowserCheck = branch(
  isSupportedBrowser,
  component => component, // trueの時は、受け取っていた引数をそのまま流用
  renderComponent(NotSupportedMessage)
)

const MainApp = withBrowserCheck(SomeMyApp)
```

`withBrowserCheck`がHigerOrderCoponentとして、`ReactComponent`を渡すと、ブラウザチェックを挟んだ状態の`SomeMyApp`を`ReactComponent`として返してくれるようになる。

また、もし「desktopだけの判定もいれたい」とかになったら、下記のような感じで拡張されるので、本筋の`SomeMyApp`の肥大化が防げる

```js
const MainApp = compose(withBrowserCheck, withDeviceCheck)(SomeMyApp)
```


# Recomposeが提供してる関数の種類など

だいたいこんなあたりがある。

- [StateやらContextをWrapして隠したりする系](https://github.com/acdlite/recompose#perform-the-most-common-react-patterns)
  - [`withState()`](https://git.io/vodCm#withstate) , [`withContext()`](https://git.io/vodCm#withcontext),　[`withReducer()`](https://git.io/vodCm#withreducer) , [`shouldUpdate()`](https://git.io/vodCm#shouldupdate)など
  - stateやらcontextを利用したり。
  - `withReudcer()`は、ReduxやFlux Utilなどから影響受けている感じ。

- Propsを色々うまいことする系
  - [`mapProps()`](https://git.io/vodCm#mapprops) , [`withProps()`](https://git.io/vodCm#withprops), [`defaultProps()`](https://git.io/vodCm#defaultprops), [`flattenProp()`](https://git.io/vodCm#flattenprop), など
  - `branch`の次に扱いやすそうだなと思った分類。
  - `withProps`は`<Some hoge={fuga} />`を`withProps({hoge: fuga})(Some)`とかに出来る。
  - `defaultProps`とうまく組み合わせると結構良い。

- Lifecycle扱う系
  - [`shouldUpdate()`](https://git.io/vodCm#shouldupdate)、[lifecycle](https://git.io/vodCm#lifecycle)
  - `Stateless Functional`でも`React.Component`でのライフライクルを扱えるようになる。
  - `shouldUpdate`は、第一引数としてtest関数を渡すので、スマートに書けそう

- [Observebleなライブラリと組み合わせる系](https://git.io/vodCm#observable-utilities)
  - [`componentFromStream()`](https://git.io/vodCm#componentfromstream), [`mapPropsStream()`](https://git.io/vodCm#mappropsstream), [`createEventHandler()`](https://git.io/vodCm#createEventHandler), [`setObservableConfig()`](https://git.io/vodCm#setobservableconfig)
  - 元々、`rx-recompose`として別ライブラリでpublishされていたが、最近のバージョンアップで統合した。
  - `RxJS`だけでなく、`most`、`kefir`、`Bacon`、`xstream`など、だいたい一通りのStreamライブラリ使えるっぽい

- [Staticなpropertyを与える](https://git.io/vodCm#static-property-helpers)
  - [`setStatic()`](https://git.io/vodCm#setstatic), [`setPropTypes()`](https://git.io/vodCm#setproptypes), [`setDisplayName()`](https://git.io/vodCm#setdisplayname)
  - `propTypes`や`displayName`などComponentにstaticに与えられるプロパティを与える。

- Renderをいい感じにする系
  - [`branch()`](https://git.io/vodCm#branch), [`renderComponent()`](https://git.io/vodCm#rendercomponent), [`renderNothing()`](https://git.io/vodCm#rendernothing)
  - Sampleで触れたので省略

- [Reactのよくあるパターンを再現したりする](https://github.com/acdlite/recompose#perform-the-most-common-react-patterns)
  - [`componentFromProp()`](https://git.io/vodCm#componentfromprop),
  - READMEのサンプルにある。
  - ぶっちゃけあんまりまだちゃんと理解出来てない
- [パフォーマンスをいい感じにする系](https://github.com/acdlite/recompose#optimize-rendering-performance)
  - [`onlyUpdateForKeys()`](https://git.io/vodCm#onlyupdateforkeys), [`onlyUpdateForPropTypes()`](https://git.io/vodCm#onlyupdateforproptypes) など
  - READMEのサンプルにある。
  - updateする対象を絞ったりしてパフォーマンス調整に役立たせられる


# まとめ
## 雑感・良かった所
- [`Stateless Function`](https://facebook.github.io/react/docs/reusable-components.html#stateless-functions)を推してる感じがする。
- HoCsの下記の利点が活かしやすい
  - ["smart" vs "dumb" component pattern](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.3e46v6pu3)にのっとりやすい
  - Component巨大化を押さえやすい
  - HoCsに対するテストしやすい
  - 再利用性を高くしやすい
- いわゆるテンプレート的なviewとロジックをどんどん分離出来る感覚がとても良い
  - `withProps`あたりをうまく扱えると、`Component`が素のtemplateっぽくなっていって良い
- jsxの記述が減らしやすい
  - テンプレート的な扱いをするComponent以外だとほとんどjsxいらないとすら言えそう
- HoCsは簡単に自前実装出来るが、ちょっとだけ自前より嬉しい事が多い。
  - いい感じにHoCsの名前をつけてくれたりするので、デバッグに役立つ
  - `compose`など、自前だと微妙に面倒な部分を提供してくれる

## 良くなかった所、注意点
- HoCsに慣れるまでちょっとつらい。
- やりすぎると可読性酷くなりかねない
- jsxの記述量減っても、全体のコード量増えそう
  - 一周回って「あれ、これjsxの方が簡素に書けてね？」みたいな事がたまにある
