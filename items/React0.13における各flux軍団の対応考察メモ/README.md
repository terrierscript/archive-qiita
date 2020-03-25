
React0.13になってから触ってみたらいくらかflux事情も変わっていたので調べた話

# React 0.13で何が変わったのか？

- mixinのdepreacted化
	- http://qiita.com/mizchi/items/ce9a128aa2a47804e083
- React.ComponentによりES6な記法で書ける

fluxに絡むところでいうとこのへんがでかい。

やっぱり0.13からはclass使いたい！

今までのfluxライブラリは今まで概ね`mixin`を使うことでviewとstoreを紐付けてきた。

じゃあこれからどうすんの？？？

# 各flux達の対応具合

## [flummox](http://acdlite.github.io/flummox)
~~個人的には今後はこれを使っていこうかなと思っているのがflummox。~~

**2015/07/23追記：[flummoxは4.0で更新止めるからこれからはreduxを使ってね！](https://github.com/acdlite/flummox#40-will-likely-be-the-last-major-release-use-redux-instead-its-really-great)**だそうだ・・


fluxxorと名前似ててややこしい。


[Why flux component is better than flux mixin](https://github.com/acdlite/flummox/blob/master/docs/docs/guides/why-flux-component-is-better-than-flux-mixin.md)という記事があるので詳しくはこちら。

具体的にはざっくりこんな感じ

```jsx
class Container extends React.Component{
  render(){
  	//一個storeとつなぐ親コンポーネントがラップする
    return (
      <FluxComponent flux={this.flux} connectToStores={{
        messages: store => ({
          foo: store.getFoo(),
          baz: store.getBaz()
        })
      }}>
        <InnderCompoent />
      </FluxComponent>
    )
  }
}

class InnderCompoent extends React.Component{
  render(){
  	// 取り出したstoreはpropとして子に伝播
    const { foo, baz } = this.props
    return (
      <div>
      </div>
    )
  }
}
```

概要で言うと、FluxComponentというElementを外側に配置し、それがStoreからデータを呼び出しpropsとして子要素に流してくれる。
結構このパターン多い。みたい。

余談だがflummoxにおいてstoreのサンプルではasync/awaitを利用していた。ナウい。

## [Alt](http://alt.js.org/)
だいたいflummoxと同様な対応っぽいので省略
http://alt.js.org/docs/components/altContainer/
Altは先鋭的ではあったけど若干複雑さを感じて断念

## [Marty](http://martyjs.org/)
以前自分で[こんな紹介記事](http://qiita.com/suisho/items/22ad9efcac90127f87a1)を書いてみたりしていたくせにというのがありつつ、今回のReact0.13の対応の話だと若干微妙かもなあというのがMarty。
どことなく場当たりな雰囲気を感じざる負えない。

```jsx
class User extends React.Component {
  render() {
    return <div className="User">{this.props.user}</div>;
  }
}

// createContainerで今までと似たような記法をする
module.exports = Marty.createContainer(User, {
  listenTo: UserStore,
  fetch: {
    user() {
      return UserStore.for(this).getUser(this.props.id);
    }
  },
  failed(errors) {
    return <div className="User User-failedToLoad">{errors}</div>;
  },
  pending() {
    return this.done({
      user: {}
    });
  }
});
```

結構やっていることが今までのmixinっぽさなのだが、結局ES6っぽくなくてうーんな感じ。

## 未対応っぽいFlux達
あまりちゃんとは追ってないけどこのあたりの初期からある者達対応追いつけてない感じがある。
issueやRoadmapまでは未確認。

- Reflux
	- https://github.com/spoike/refluxjs#convenience-mixin-for-react
- Fluxxor
	- http://fluxxor.com/guides/quick-start.html
- delorean
	- https://github.com/deloreanjs/delorean/blob/master/docs/views.md
- McFly
	- https://github.com/kenwheeler/mcfly#stores


# 結論
もうflummoxでいいんじゃないかな
