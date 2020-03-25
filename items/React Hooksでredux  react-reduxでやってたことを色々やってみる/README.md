※ **React HooksはRFCの段階です。この記事はあくまで実験の産物としてお読み下さい。**
-> 16.8でリリースされました。それほど大きな変更はなさそうです

また、出たばかりなので探り探りで書いています。何かある場合はコメントや編集リクエストをいただければ幸いです。

# 準備
React hooksは16.7にのみ予定されているので、下記のコマンドで16.7を入れる

```
yarn add react@^16.8.0 react-dom@^16.8.0
```

# 1. useReducerでcombineReducersだけ使ってみる
reduxにおいてはcombineReducersを利用してネストすることが出来た。
hooksの機能としてはこの部分だけは提供されてないので、reduxから流用して組み合わせることをやってみる

例えばこんなreducer

```js
import { combineReducers } from "redux"

const counter = (state = 0, action) => {
  switch (action.type) {
    case "INCREMENT":
      return state + 1
    case "DECREMENT":
      return state - 1
  }
  return state
}

const inputValue = (state = "foo", action) => {
  switch (action.type) {
    case "UPDATE_VALUE":
      return action.value
  }
  return state
}

export const rootReducer = combineReducers({
  counter,
  // サンプルとしてネストしてみる
  someNested: combineReducers({
    inputValue
  })
})
```

利用側はこんな感じ

```jsx
import React, { useReducer } from "react"

const App = () => {
  const [state, dispatch] = useReducer(rootReducer, undefined, {
    type: "DUMMY_INIT"
  })

  return (
    <div className="App">
      <div>
        <h1>counter</h1>
        <div>count: {state.counter}</div>
        <button onClick={(e) => dispatch({ type: "INCREMENT" })}>+</button>
        <button onClick={(e) => dispatch({ type: "DECREMENT" })}>-</button>
      </div>
      <div>
        <h1>Input value</h1>
        <div>value: {state.someNested.inputValue}</div>
        <input
          value={state.someNested.inputValue}
          onChange={(e) =>
            dispatch({
              type: "UPDATE_VALUE",
              value: e.target.value
            })
          }
        />
      </div>
    </div>
  )
}
```

一点工夫する点として、initialStateとinitialActionをダミーでも渡してデフォルトのinitial値を使う必要があるようだ

```js
const [state, dispatch] = useReducer(rootReducer, undefined, {
  type: "DUMMY_INIT"
})
```


# 2: createContextとuseContextを組み合わせてProviderを作る

`useReducer`だけだと孫までバケツリレーが必要になる。
ここはuseReducerにcontextを利用してみよう。
Contextを使うようなケースでも無いのかも？とは思ったが、一応公式にもこれに類似したドキュメントが存在しているようだ
https://reactjs.org/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down

まずContexを作成。初期値は空

```js
const ReducerContext = createContext()
```

Providerはこんな具合で書かれる

```jsx
// ContextのProviderをラップしているだけ。値としてstateとdispatchを渡してしまうことにする
const Provider = ({ children }) => {
  const [state, dispatch] = useReducer(rootReducer, undefined, {
    type: "DUMMY_INIT"
  })
  return (
    <ReducerContext.Provider value={{ state, dispatch }}>
      {children}
    </ReducerContext.Provider>
  )
}

const App = () => {
  return (
    <Provider>
      <div className="App">
        <Counter />
        <InputValue />
      </div>
    </Provider>
  )
}

```

利用側(Consumer）はuseContexを利用するとこうなる。
ちょっとだけuseContextの周りの値が不定で気持ち悪い感も否めない

```jsx
const Counter = () => {
  const { state, dispatch } = useContext(ReducerContext)
  return (
    <div>
      <h1>counter</h1>
      <div>count: {state.counter}</div>
      <button onClick={(e) => dispatch({ type: "INCREMENT" })}>+</button>
      <button onClick={(e) => dispatch({ type: "DECREMENT" })}>-</button>
    </div>
  )
}
const InputValue = () => {
  const { state, dispatch } = useContext(ReducerContext)
  return (
    <div>
      <h1>Input value</h1>
      <div>value: {state.someNested.inputValue}</div>
      <input
        value={state.someNested.inputValue}
        onChange={(e) =>
          dispatch({
            type: "UPDATE_VALUE",
            value: e.target.value
          })
        }
      />
    </div>
  )
}
```

useContextを使わずConsumerを使うならこうだろう

```jsx

const Counter = () => {
  return (
    <ReducerContext.Consumer>
      {({ state, dispatch }) => {
        return (
          <div>
            <h1>counter</h1>
            <div>count: {state.counter}</div>
            <button onClick={(e) => dispatch({ type: "INCREMENT" })}>+</button>
            <button onClick={(e) => dispatch({ type: "DECREMENT" })}>-</button>
          </div>
        )
      }}
    </ReducerContext.Consumer>
  )
}
```

# 3. useCallbackでbindActionCreactors っぽいこと

actionのbindをしたければuseCallbackが使えそうだ。
useCallbackはmemorizeしてくれる関数で、実は無くても動くが再レンダリングが起きてもmemo化されてくれる。引数として`[dispatch]`を与えているので、dispatchがもし変更された場合に再生成される（はず）

```jsx
  const increment = useCallback((e) => dispatch({ type: "INCREMENT" }), [
    dispatch
  ])
  const decrement = useCallback((e) => dispatch({ type: "DECREMENT" }), [
    dispatch
  ])
  const updateValue = useCallback(
    (e) =>
      dispatch({
        type: "UPDATE_VALUE",
        value: e.target.value
      }),
    [dispatch]
  )
  return <div>
   :
    <button onClick={increment}>+</button>
    <button onClick={decrement}>-</button>
   :
  </div>
```
# 4. useMemoでmapStateToPropsでreselect使っていたような部分をやってみる

masStateToPropsでやっていたようなstateからpropsへの変換や、よくそこで使われていたreselectのmemo化機能を`useMemo`で扱えそうだ。

```jsx
const InputValue = () => {
  const { state, dispatch } = useContext(ReducerContext)
  // state.someNested.inputValueが変更されるまでmemo化する
  const inputValue = useMemo(() => state.someNested.inputValue, [
    state.someNested.inputValue
  ])

  return (
    <div>
      <h1>Input foo</h1>
      <div>foo: {inputValue}</div>
      <input
        value={inputValue}
        onChange={(e) =>
          dispatch({
            type: "UPDATE_VALUE",
            value: e.target.value
          })
        }
      />
    </div>
  )
}
```

# 5. Containerを再現する

ここまでの応用で、Containerのような作用をするhooksを作ってみる。
inputValueの方のみ例示する。Counterの場合も一緒だ。


```jsx

const useCounterContext = () => {
  const { state, dispatch } = useContext(ReducerContext)
  const counter = useMemo(() => state.counter, [state.counter])
  const increment = useCallback(
    (e) => setTimeout(() => dispatch({ type: "INCREMENT" }), 500),
    [dispatch]
  )
  const decrement = useCallback((e) => dispatch({ type: "DECREMENT" }), [
    dispatch
  ])

  return { counter, increment, decrement }
}

const Counter = () => {
  const { counter, increment, decrement } = useCounterContext()
  return (
    <div>
      <h1>counter</h1>
      <div>count: {counter}</div>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  )
}

// containerという名前をつけるべきかはまだ悩ましい
const useInputContainer = () => {
  const { state, dispatch } = useContext(ReducerContext)
  // memo化したcallbackを作成
  const updateValue = useCallback(
    (e) =>
      dispatch({
        type: "UPDATE_VALUE",
        value: e.target.value
      }),
    [dispatch]
  )
  // 値をmemo化。selectのような作用
  const inputValue = useMemo(() => state.someNested.inputValue, [
    state.someNested.inputValue
  ])
  return {
    updateValue, inputValue
  }
}

const InputValue = () => {
  // Component側でContainerの作用をするhookを呼び出す。
  const { updateValue, inputValue } = useInputContainer()
  return (
    <div>
      <h1>Input foo</h1>
      <div>value: {inputValue}</div>
      <input value={inputValue} onChange={updateValue} />
    </div>
  )
}

// ここは相変わらず
const Provider = ({ children }) => {
  const [state, dispatch] = useReducer(rootReducer, undefined, {
    type: "DUMMY_INIT"
  })
  const value = { state, dispatch }
  return (
    <ReducerContext.Provider value={value}>{children}</ReducerContext.Provider>
  )
}
```

# できたもの

5まで行った状態のコードを下記に置いた
https://stackblitz.com/edit/github-hgrund?file=src/App.js

# 番外編：middleware
非同期処理をやるようなmiddlewareは、`useEffect`でも可能だが、**[これは推奨されない使い方で、将来的にSuspenseがその役割を担う可能性が高い](https://mobile.twitter.com/dan_abramov/status/1057011888335347713)** ため、あくまで番外編としている。

まずreducerはこのような感じで生やす

```js
const fetchedData = (state = {}, action) => {
  switch (action.type) {
    case "FETCH_DATA":
      return action.value
  }
  return state
}
```

そしてfetchに相当する関数を作る。
今回はデータ取得の代わりに100ms後に乱数を返してみる。

```js

const fetchData = (dispatch) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ random: Math.random() })
    }, 100)
  })
  // 本当の通信であればこんな感じ
  // return fetch("./async.json")
  //   .then((res) => res.json())
  //   .then((data) => {
  //     return data
  //   })
}

```

そしてcontainer相当の部分の作成。
注意点として`useEffect`の第二引数を空配列とすることでmount,unmount時に一度だけ実行されるようにすること。
ここに渡した値が変更される度にeffectに指定した関数が実行される。
何もしない場合レンダリングの度に毎回実行されるので注意。この場合でいうとfetchからdispatchしてstateが変更されることで無限ループが起きているものと推測される。
（参照：https://reactjs.org/docs/hooks-effect.html#tip-optimizing-performance-by-skipping-effects)

```jsx
const useFetchDataContainer = () => {
  const { state, dispatch } = useContext(ReducerContext)

  // 初回実行。第二引数を空arrayにすることで1度だけ実行にする
  useEffect(() => {
    fetchData().then((data) => {
      dispatch({
        type: "FETCH_DATA",
        value: data
      })
    })
  }, []) // ← ここに注意

  // 再実行関数を定義する
  const reload = useCallback(() => {
    fetchData().then((data) => {
      dispatch({ type: "FETCH_DATA", value: data })
    })
  })

  const data = useMemo(
    () => {
      return JSON.stringify(state.fetchedData, null, 2)
    },
    [state.fetchedData]
  )
  return { data, reload }
}

// あとは利用部分のみ
const FetchData = () => {
  const { data, reload } = useFetchDataContainer()
  return (
    <div>
      <h1>Fetch Data</h1>
      <pre>
        <code>{data}</code>
      </pre>
      <button onClick={reload}>Reload</button>
    </div>
  )
}
```

書き味としては`redux-thunk`やもっと古い所で言えば個人的には[`redial`](https://github.com/markdalgleish/redial)に近い形になってくる。

また、`useReducer`にactionをフックするような仕組みは無いため、`redux-observabel`や`redux-saga`のようなアプローチはとれないように思える。
もしそういうものが必要な場合は`useState`から拡張される形ような気がする
