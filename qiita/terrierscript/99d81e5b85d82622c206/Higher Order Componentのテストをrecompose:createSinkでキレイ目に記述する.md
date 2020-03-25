
Higher Order Componentは、コンポーネントというドメイン固有になりがちな部分から一歩切り離されたものなので、うまく作ればテストとの相性はだいぶ良い。（Dependency Injectionと近しい感じがあるかもしれない）

Componentの装飾に依存しない部分などをHigher Order Componentに外出しすれば、独立的にテストが書ける。
しかしComponentのMockを作るという性質上、若干そのままテストを書こうとすると若干の気持ち悪さがあったりしたので調べた

# 今回用意するサンプル

```js
// barで受け取った変数をbazにするHOC。
export const myEnhancer = (Component) => {
  return (props) => {
    const hocsProps = Object.assign({}, {
      baz: props.bar
    }, props)
    return <Component {...hocsProps} />
  }
}

// 通常の使い方はこんな感じとしてみる。
const Item = ({foo, baz}) => {
  return <div>
    <div>{foo}</div>
    <div>{baz}</div>
  </div>
}

export default myEnhancer(Item)
```

ざっくりテスト書いてみると、こんな感じになる

```js
import React from 'react';
import ReactDOM from 'react-dom';
import { myEnhancer } from './Item';

it('renders ', (done) => {
  const div = document.createElement('div');
  // 通常のItemとは別なMockItemを作る。
  const MockItem = (props) => {
    // renderの内部でassertをする。
    assert.equal(props.baz, 'bar value')
    done()
    // returnは適当
    return <noscript>
  }
  // MockItemをmyEnhancerで囲ってrenderする。
  const Mock = myEnhancer(MockItem)
  ReactDOM.render(<Mock bar={'bar value'} />, div);
});
```

上記だと、renderするタイミングの部分でpropsをassertする形。悪くはない。

# もうちょっといい感じに

しかしなんかこうもうちょっといい感じにしたいという場合に [recompose/createSink](https://github.com/acdlite/recompose/blob/master/docs/API.md#createsink)を使うと良い。
[ソースコード](https://github.com/acdlite/recompose/blob/master/src/packages/recompose/createSink.js)を見ると、何もrenderしないReact Componentを返してくれる事がわかる。

```
$ npm install recompose
```

こんな具合で書き換える。

```js
import createSink from 'recompose/createSink'

it('renders ', (done) => {
  const div = document.createElement('div');
  // createSink(callback) という形で、callback内でテストを書く
  // これがmockのItemになる。
  const MockItem = createSink((props) => {
    assert.equal(props.baz, 'bar value')
    done()
  })
  const Mock = myEnhancer(MockItem)
  ReactDOM.render(<Mock bar={'bar value'} />, div);
});
```

だいぶ綺麗な感じになる。良い。
`createSink`なんて何に使うんだ？と思っていたがこういう所で使えるらしい。
`props`以外のLifecycleをテストするような場合ではちょっとこれだと足らなそう
