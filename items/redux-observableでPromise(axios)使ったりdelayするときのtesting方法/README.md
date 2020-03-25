# 問題点

redux-observableのドキュメントには[WritingTests](https://redux-observable.js.org/docs/recipes/WritingTests.html)のページがある。
が、例えばこれがPromiseを返すaxiosとかと組みあわせたりした時ちょっとうまくいかない。

こんなepicを用意して考える。

```js
const axios = require("axios")
const { createEpicMiddleware } = require('redux-observable')
require("rxjs")

const payload = { id: 123 }

// test用にmock化
const mockAdapter = (config) => {
  return new Promise((resolve, reject) => {
    resolve({data: payload, status: 200 })
  })
}

const fakeApi = axios.create({
  adapter: mockAdapter
})

const FETCH_USER = "FETCH_USER"
const FETCH_USER_FULFILLED = "FETCH_USER_FULFILLED"

const fetchUserFulfilled = payload => {
  return { type: FETCH_USER_FULFILLED, payload }
}

const fetchUserEpic = action$ => {
  return action$.ofType(FETCH_USER)
    .mergeMap(action => fakeApi.get(`/api/users/${action.payload}`))
    .map( ({ data }) => fetchUserFulfilled(data) )
}
```

そして、テストを書いてみる。
これが上記ドキュメントにあるようにそのまま書くと通らない。

```js
const configureMockStore = require('redux-mock-store').default
const epicMiddleware = createEpicMiddleware(fetchUserEpic);
const mockStore = configureMockStore([epicMiddleware]);

describe('fetchUserEpic', () => {
  it('produces the user model', () => {
    // うまくいかないパターン。
    const store = mockStore();
    store.dispatch({ type: FETCH_USER })
    expect(store.getActions()).toEqual([
      { type: FETCH_USER }, // こちらのactonはある
      { type: FETCH_USER_FULFILLED, { id: 123 } } // こちらが無い
    ])
  })
})
```

これは、下記のようなepicでもtestが通らない事が確認できた

```js
const fetchUserEpic = action$ => {
  return action$.ofType(FETCH_USER)
    .delay(1000)
    .map( () => fetchUserFulfilled(payload) )
```

つまり、 delayやpromiseで非同期的な動きをする場合にこのままだとうまくいかない

# どうするか？

## 解法1: setTimeoutする
すごい微妙なやりかた。

```js
describe('fetchUserEpic', () => {
  it('produces the user model', (done) => {
    const store = mockStore();
    store.dispatch({ type: FETCH_USER })
    setTimeout( () => {
      expect(store.getActions()).toEqual([
        { type: FETCH_USER }, 
        { type: FETCH_USER_FULFILLED, { id: 123 } }
      ])
      done()
     }, 0)
  })
})
```

`setImmediate`とか`nextTick`でも良いと思う。

## 解法2: redux-mock-storeのsubscribe機能を使う

もうちょっとスマートに行きたいので、[redux-mock-store](http://arnaudbenard.com/redux-mock-store/)側の機能を使ってみる。

reduxの本物のstore同様subscribeが用意されているので、これを引っ掛ける

```js
describe('fetchUserEpic', () => {
  it('produces the user model', (done) => {
    const store = mockStore();
    store.dispatch({ type: FETCH_USER })
    const unsubscribe = store.subscribe( () => {
      // ここでassertion
      expect(store.getActions()).toEqual([
        { type: FETCH_USER },
        { type: FETCH_USER_FULFILLED,  { id: 123 } }
      ])
      unsubscribe()
      done()
    })
  })
})
```
subscribeのタイミングでassertionしている。
今回は「単一のactionが来る事だけを期待したテスト」をしているが、複数のactionが飛んでくることを想定する場合はもうちょっと工夫する必要があるだろう。

更に共通化して扱いたかったらこんな感じにも出来そう

```js
const assertAction = (targetEpic, dispatchAction, expectAction, cb) => {
  const epicMiddleware = createEpicMiddleware(targetEpic);
  const mockStore = configureMockStore([epicMiddleware]);

  const store = mockStore();
  store.dispatch(dispatchAction)
  const unsubscribe = store.subscribe( () => {
    expect(store.getActions()).toEqual(expectAction)
    unsubscribe()
    cb()
  })
}

describe('fetchUserEpic', () => {
  it('produces the user model', (done) => {
    assertAction(fetchUserEpic, { type: FETCH_USER }, [
      { type: FETCH_USER },
      { type: FETCH_USER_FULFILLED,  { id: 123 } }
    ], done)
  })
})
```
