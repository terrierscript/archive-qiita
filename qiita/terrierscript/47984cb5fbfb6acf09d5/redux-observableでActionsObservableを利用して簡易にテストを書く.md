
redux-observableのテストとして、mock-storeを利用したテストやmarble testingが挙げられる。
mock-storeを利用したテストは[非同期で小面倒くさ](http://qiita.com/inuscript/items/6b64eecad73ddabf256c)かったり、storeの中身のテストになっていてepicのテストとして扱いづらい。
marble testingはスマートだけどRx慣れしてない者にとってはちょっと難易度が高い。


* [redux-observable - WritingTests](https://redux-observable.js.org/docs/recipes/WritingTests.html)
* [redux-observable の処理を marble testing で簡単にテストする](http://jyane.jp/2016/12/26/redux-observable.html)
* [marble testing - JSBin](http://jsbin.com/pufima/edit?js,output)


# ActionsObservableを使って小さめのテストする。

今回は[ActionsObservable](https://github.com/redux-observable/redux-observable/blob/master/src/ActionsObservable.js)を利用したテストを紹介する。
ActionsObservableはundocumentedではあるが、[index.d.ts](https://github.com/redux-observable/redux-observable/blob/master/index.d.ts#L6) などで定義されていたり、marble testingのサンプルとして利用されている。
テストで利用する分には問題ない気がするが、それ以上に使う時は注意が必要かもしれない。

今回は`mocha`を利用している前提でサンプルコードを書いていく

## ActionsObservableでepicだけテスト

とりあえずこんなepicを用意する

```js
require("rxjs")
const pingEpic = (action$) => {
  return action$.ofType("PING")
    .map( () => {
      return { type: "PONG" }
    })
}
```

そしてこんな感じでテストが書ける

```js
const assert = require("assert")
const { ActionsObservable } = require("redux-observable")

describe("epic test", () => {
  it("ping epic", (done) => {
    // `ActionObservable`は`Observable`の拡張なので、`of`など使える。
    const mockAction = ActionsObservable.of({type: "PING"})
    pingEpic(mockAction)
      .toArray() // assertしやすいように`toArray`する
      .subscribe( result => { // subscribeで受取り
        assert.deepEqual(result, [
          {type: "PONG"} // こういうactionが来るハズ
        ])
        done()
      })
  })
})
```
Actionをmockするだけでその結果に来るはずのactionをテストすることが可能になった。嬉しい。

また、複数のactionを飛ばせば複数のactionがsubscribe出来る。

```js
  it("ping epic", (done) => {
    const mockAction = ActionsObservable.of(
      {type: "PING"},
      {type: "PING"}
    )
    pingEpic(mockAction)
      .toArray()
      .subscribe( result => {
        assert.deepEqual(result, [
          {type: "PONG"},
          {type: "PONG"}
        ])
        done()
      })
  })
```

## combineEpicsと組み合わせる

`combineEpics`を使えば複数のepicがある前提のテストも出来る

```js
require("rxjs")
const pingEpic = (action$) => {
  return action$.ofType("PING")
    .map( () => {
      return { type: "PONG" }
    })
}

const anotherPingEpic = (action$, store) => {
  return action$.ofType("PING")
    .map( () => {
      return { type: "PUNG", payload: "foo"}
    })
}
```

```js
const assert = require("assert")
const { ActionsObservable, combineEpics } = require("redux-observable")

describe("epic test", () => {
  it("ping epic ( with combine )", (done) => {
    const mockAction = ActionsObservable.of({type: "PING"})
    const combinedEpic = combineEpics(
      pingEpic,
      anotherPingEpic
    )
    combinedEpic(mockAction)
      .toArray()
      .subscribe( result => {
        // 一つのactionに対して複数のepicからの処理が返る
        assert.deepEqual(result, [
          {type: "PONG"}, // pingEpicが吐き出したやつ
          {type: "PUNG", "payload": "foo"}, // anotherPingEpicが吐き出したやつ
        ])
        done()
      })
  })
})
```

何らか複数のepicが絡む場合のテストがしたいなら使えるだろう

## 非同期扱う
Promiseを利用した非同期も問題なく扱える

```js
// lazyPing
const lazyPing = () => {
  return new Promise( res => {
    setTimeout( () => {
      res("DELAY")
    }, 100)
  })
}
```

```js
// epic.js
require("rxjs")
const pingEpic = (action$) => {
  return action$.ofType("PING")
    .mergeMap( () => lazyPing() ) // promiseはmergeMapなどで処理出来る
    .map( (result) => {
      return { type: "PONG", payload: result}
    })
}
```

```js
// test.js
const assert = require("assert")
const { ActionsObservable, combineEpics } = require("redux-observable")

describe.only("epic test", () => {
  it("lazy ping epic", (done) => {
    const mockAction = ActionsObservable.of({type: "PING"})
    pingEpic(mockAction)
      .toArray()
      .subscribe( result => {
        assert.deepEqual(result, [
          {type: "PONG", payload: "DELAY"}
        ])
        done()
      })
  })
})
```

## storeをmock化して使う

storeも使って何かしらやったりするのも出来る

```js
// epic.js
require("rxjs")

const counterEpic = (action$, store) => {
  return action$.ofType("DO_INCREMENT")
    .map( (action) => {
      return { 
        type: "INCREMENT", 
        payload: action.payload + store.getState().baseCount
      }
    })
}

```

```js
// test.js
const assert = require("assert")
const { ActionsObservable, combineEpics } = require("redux-observable")

describe("epic test", () => {
  it("counter epic", (done) => {
    // actionと一緒にstoreもmockする
    const mockAction = ActionsObservable.of({type: "DO_INCREMENT", payload: 2})
    const mockStore = {
      getState() {
        return { baseCount: 10 }
      }
    }
    counterEpic(mockAction, mockStore)
      .toArray()
      .subscribe( result => {
        assert.deepEqual(result, [
          {type: "INCREMENT", payload: 12},
        ])
        done()
      })
  })
})
```

# まとめ
* ある程度簡易なテストはActionsObservableだけで出来た。
* storeのmockから解放されるのは結構嬉しい
    * reducerまで絡んでくるとかになるまでは使わなくて良さそう
* タイミングなどもっとシビアにやりたいならmarble testingを導入してくのが良さそう
    * [Observable.timer](https://www.learnrxjs.io/operators/creation/timer.html)とか[Observable.interval](https://www.learnrxjs.io/operators/creation/interval.html)とかで元actionwお作れそうな気もするが、多分marble testingのほうが見通し良くなりそうな気がする
