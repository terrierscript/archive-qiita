
アヒルさんがぐるぐる回っててかわいかったので飛びついてみたけど色々サンプルが動かなくて可愛くなかったので動くまでもがいた。

http://redux-observable.js.org/

# 成果物

- [githubに置いた](https://github.com/inuscript/example-redux-observable-pingpong)
- pingボタンを押したらtrueになって1秒後にfalseになるサンプル。
- https://jsbin.com/vayoho/6/edit
	- ↑これとほぼおなじ

## コード
簡素化のために1ファイルに収めている

https://github.com/inuscript/example-redux-observable-pingpong/blob/master/index.js


```js
import React from 'react'
import ReactDom from 'react-dom'
import { createStore, applyMiddleware } from 'redux' 
import { createEpicMiddleware } from 'redux-observable'
import { connect, Provider } from 'react-redux'

// rxjsハマり部分。後述
import 'rxjs/add/operator/mapTo'
import 'rxjs/add/operator/filter'
import 'rxjs/add/operator/delay'

// sampleにあるreducer
const pingReducer = (state = { isPinging: false }, action) => {
  switch (action.type) {
    case 'PING':
      return { isPinging: true };
    case 'PONG':
      return { isPinging: false };
    default:
      return state;
  }
}

// sampleにあるepic
const pingEpic = action$ => action$.filter(action => action.type === 'PING')
  .delay(1000) 
  .mapTo({ type: 'PONG' })

// epicをmiddleware化 + Store化
const epicMiddleware = createEpicMiddleware(pingEpic);
const store = createStore(pingReducer, applyMiddleware(epicMiddleware))

// ボタン押したら文字が変わるcomponent
const PingComponent = ({dispatch, isPinging}) => {
  return (
    <div>
      <div>isPinging: {isPinging.toString()}</div>
      <div>
        <button onClick={ (e) => dispatch({type: 'PING'}) }>Dispatch Ping</button>
      </div>
    </div>
  )
}

// Build App
const App = () => {
  // stateそのまま流す
  let PingContainer = connect( state => state )(PingComponent)
  return (
    <Provider store={store}>
      <PingContainer />
    </Provider>
  )
}

// Render App
ReactDom.render(
  <App />,
  document.body.appendChild(document.createElement('div'))
)
```

# 留意点・知見・ハマりどころ

## Epicという用語
http://redux-observable.js.org/docs/basics/Epics.html
StreamなActionを受け取ってActionを返す。
これを定義してmiddlewareとして処理されるもの。
とりあえずこいつが`redux-observable`のキモだと思って良さそう

## RxJS 5の独特さ
JSBinサンプルの見てもちゃんと動いていてしばらくハマった部分。

たとえばこんなEpic

```js
const pingEpic = action$ => action$.filter(action => action.type === 'PING')
  .delay(1000) 
  .mapTo({ type: 'PONG' })
```
このまま走らせると
`Uncaught TypeError: action$.filter is not a function` と言われる。

これでしばらく悩んでいたが、RxJS 5系では、各Operatorを明示的にimportしてやる必要があった。[^2]

https://github.com/ReactiveX/rxjs#es6-via-npm

[^2]: RxJSろくに理解してなかったせいでドはまりしたポイントではあるが、Javascriptとしてだいぶ独特な感じで「これは初見殺しじゃないですか」と思った。

```js
import 'rxjs/add/operator/mapTo'
import 'rxjs/add/operator/filter'
import 'rxjs/add/operator/delay'
```

このようにそれぞれimportすると動く。
[rxjs/add/operator/filter](https://github.com/ReactiveX/rxjs/blob/master/src/add/operator/filter.ts)あたりの中身を覗いてみるとわかるが、

```js
Observable.prototype.filter = filter;
```
というような感じでObservableをprototype拡張している。

Epicで渡される`action$`はredux-observableで管理されている[ActionObservable](https://github.com/redux-observable/redux-observable/blob/master/src/ActionsObservable.js)というもので`Observable`を継承している。

「大量にimportするの面倒」というような場合は

```js
import Rx from 'rxjs/Rx'
```

で呼び出して全[Operatorのaddをする](https://github.com/ReactiveX/rxjs/blob/master/src/Rx.ts#L11)ことも出来る模様。（副作用的な感じもするので、良いやり方なのかどうかは不明。。。）

### 備考：function bindによるRxの読み込み

RxJSのReadmeに記載されているが、「prototype拡張してんのやだなー」という場合は、`function bind`を使う手法も一応ある。function-bindは今のところstage-0でかなり尖っているので注意。

```
$ npm i -D babel-plugin-transform-function-bind
```

.babelrcはこんな具合

```.babelrc
{
  "presets": ["es2015", "react"] ,
  "plugins": [
    "transform-function-bind"
  ]
}
```

そうするとこんな具合で書ける。

```js
import { mapTo } from 'rxjs/operator/mapTo';
import { filter } from 'rxjs/operator/filter';
import { delay } from 'rxjs/operator/delay';

const pingEpic = action$ => action$
  ::filter(action => action.type === 'PING')
  ::delay(1000) 
  ::mapTo({ type: 'PONG' })
```

stage-0なんて怖い！素で使いたい！
となると多分こんな具合。これはこれできつそう

```js
const pingEpic = action$ => mapTo.call(
  delay.call(
    filter.call(action$, action => action.type === 'PING'),
    1000
  ),
  { type: 'PONG' }
)
```

## type絞るなら`filter`使わずに`ofType`で良い
今回サンプルでは`filter`を使ったが、`ActionObeservable`には`ofType`というのが用意されているので、これを使ったほうがキレイ目に書ける。

```js
const pingEpic = action$ =>
  action$.ofType('PING')
    .delay(1000) // Asynchronously wait 1000ms then continue
    .mapTo({ type: 'PONG' })
```

## 新しく別なactionを流せるだけ。元のactionに手を加える事は出来ない。

redux-thunkやmiddlewareだと出来ていたことで出来ないのが「actionの変更・加工・握りつぶし」的な作用。

例えばこんな感じの事は出来ない（多分）。

```js
// PING actionだったら握りつぶしてPONGにするmiddleware
const fixupPingMiddleware = store => next => action => {
  if(action.type !== 'PING'){
    return next(action)
  }
  return next({
    type: 'PONG'
  })
}
```

[createEpicMiddleware](https://github.com/redux-observable/redux-observable/blob/master/src/createEpicMiddleware.js#L25)の中身を見ると、上記のコードで言えば、こんなことをやっている感じ。

```js
// PING actionだったらPONGも飛ばす
const fixupPingMiddleware = store => next => action => {
  result = next(action)
  store.dispatch({type: 'PONG'})
  return result
}
```

そのため、もし上記の利用しないactionをreducerなどで影響が出ないようにしておく必要がある。actionがimmutableになっていると意識すると良いようだ。

そもそも「加工したり握りつぶしたり」というのはそんなにスマートなやり方では無いが、意識してやらないように心がけていないとやりたくなる事は多いので注意したい。

## Epic作るのにRxJSに慣れてない時は、playground的なテストコード書いたら便利

RxJSに慣れていなかったので、[Writning Tests](http://redux-observable.js.org/docs/recipes/WritingTests.html)を参考に、下記のようなplaygroundを作ると、色々試せて良い。
[`redux-mock-store`](https://github.com/arnaudbenard/redux-mock-store)でactionが何が来たのか補足出来るので、最後に`store.getActions`してみると良い。
testにはava使ってみているけど、そこは別になんでも良い。

```js
import test from 'ava'
import configureMockStore from 'redux-mock-store';
import { createEpicMiddleware, combineEpics } from 'redux-observable';

// operatorの追加忘れず
import 'rxjs/add/operator/map'

test('Epic Playground', t => {
  // operatorとか色々試してみる
  const pingEpic = action$ => action$
    .ofType('PING')
    .map( action => {
      return {
        type: 'PONG',
        payload: action.payload
      }
    })

  const epicMiddleware = createEpicMiddleware(pingEpic);
  const mockStore = configureMockStore([epicMiddleware]);

  let store = mockStore()
  store.dispatch({ type: 'PING', payload: 1 })

  console.log(store.getActions())
  // output:
  // [ { type: 'PING', payload: 1 }, { type: 'PONG', payload: 1 } ]
})
```

## 参考資料など

* [Learn RxJS](https://www.learnrxjs.io/)
	* とっつきやすいRxJSのドキュメント
* [MIGRATION.md](https://github.com/ReactiveX/rxjs/blob/master/MIGRATION.md)
	* RxJS4 -> RxJS5のmigration情報。結構混在しているので、詰まったら見直すと良い

## ロゴ
どっかでこのロゴ見たことあるなと思ってたけどreduxのロゴ案として上がっていたやつだった。
https://github.com/reactjs/redux/issues/151#issuecomment-137833403

---
