
何回か使ううちに微妙な所も見えつつ、とはいえ結局より良いfluxは出てき無いので付き合っている感じのRedux。

毎回最初にどう書くんだったか忘れてしまうのでここらで自分が毎回やっているのをまとめておく。

だいたい下記を入れておいてる前提

- react関連
	- react
	- react-dom
- redux関連
	- redux
	- react-redux
	- redux-actions
	- redux-thunk

- babel必要ならこのへんのpreset
	- babel-preset-react
	- babel-preset-es2016

# main

エントリポイントになる所。あんまり考えることはない。
`./store`、`./container`は下記で記載する。

あんまり気合いれない時はこのファイルにstoreもcontainerもまとめちゃう。

```js
import React, { Component } from "react"
import ReactDom from "react-dom"
import setupStore from "./store"
import createContainer from "./container"

export default function render(){
  let containerEl = document.querySelector("#container")
  let store = setupStore()
  ReactDom.render(createContainer(store), containerEl)
}
```
必要があれば`render(containerEl)`みたいに外からcontainerのDOMもらったりする

# store

initialState + middleware設定とかしとく。
ちょいちょいいじる必要でるくせにどこでやるんだったかわすれがち

```js
import reducers from "./reducers"
import {createStore, applyMiddleware} from "redux"
import thunk from "redux-thunk"

export default function setupStore(){
  const initialState = {
    todo: []
  }
  return createStore(
  	reducers, 
  	initialState, 
  	applyMiddleware(thunk)
  )
}

```

# container
reduxとreactつなぐ部分。
connectするらへん
一番忘れてて悩む所。
ここらへんすっきりしてくれると楽なのになー。。。

とりあえず`mapStateToProps `、`mapDispatchToProps`どちらも最初に作っておいてしまう派。


```js
import { bindActionCreators } from "redux"
import { connect, Provider } from "react-redux"
import App from "./Component/App.js"

import * as actions from "./actions"

function mapStateToProps(state){
  // 必要に応じてreselectとかここで使う
  return state
}

function mapDispatchToProps (dispatch) {
  // dispatchとかcomponent以下に渡したくないのでここでbindしてしまう。
  return bindActionCreators(actions, dispatch)
}

export default function createContainer(store){
  let ConnectedApp = connect(mapStateToProps, mapDispatchToProps)(App)
  return (
    <Provider store={store}>
      <ConnectedApp />
    </Provider>
  )
}

```

`container`と`store`を分離するパターンの時には取り回しよくなるように`createContainer(store)`としておく。
mainファイルにまとめてしまう時はあんまりそのへん気にしない
# action
actionはシンプルだが、`redux-actions`使うとだいぶ便利で良い。
ちょっとredux-actもいいなと思っている昨今。

```js
import { createAction } from "redux-actions"
import * as types from "./types"

export const setTodo = createAction(SET_OBJ, (obj) => obj)
```

## types
単純な定数定義

```js
export const SET_OBJ = "set_obj"
```

# reducer
あんまり忘れない部分。
もうちょっと楽したい気もする。けどredux-actionsが未だに馴染んでないので普通に書いてる

```js
import { combineReducers } from 'redux'
import * as types from "./types"

const someItem = (state = {}, action) => {
  switch(action.type){
    case types.SET_OBJ:
      return Object.assign({}, state, action.payload)
    default:
      return state
  }
}

const rootReducer = combineReducers({
  someItem
})

export default rootReducer
```
