
typescript-fsaなど、TypeScriptとReduxを利用する上でサードパーティライブラリが勧められる事があったが、現状のTypeScript　2.9、3.0-rcで普通に書いてみたところ、素reduxでもVSCodeでわりとサクサク補完されるようだ。

追記：[色々考えた結果最終型](https://qiita.com/terrierscript/items/b3f9dd95a4c7afe0b102#%E6%9C%80%E7%B5%82%E5%9E%8B)を追加した

# 本題（古いバージョン）

とりあえず最初に考えたのはこんな感じ

```ts
import {
  createStore,
  Reducer,
  Action,
  combineReducers,
  ActionCreatorsMapObject,
  ReducersMapObject,
  ActionCreator
} from "redux"

//// State全体の定義
export type AppState = {
  counter: number
}

//// ActionとActionCreatorの定義
// type CounterAction = Action<"INCREMENT" | "DECREMENT"> でも可
type CounterActionType = "INCREMENT" | "DECREMENT"
type CounterAction = Action<CounterActionType>

// これだけだるいが利用側のために二重定義してる。後述
export interface CounterActionCreators
  extends ActionCreatorsMapObject<CounterAction> {
  increment: ActionCreator<CounterAction>
  decrement: ActionCreator<CounterAction>
}
export const counterActions: CounterActionCreators = {
  increment: () => {
    return { type: "INCREMENT" }
  },
  decrement: () => {
    return { type: "DECREMENT" }
  }
}

//// Reducer。genericsらへんはちょっと怠けてる
export const counterReducer: Reducer<any, CounterAction> = (
  state = 0,
  action
) => {
  switch (action.type) {
    case "INCREMENT": // このへんVSCode補完効いて最高の気分
      return state + 1
    case "DECREMENT":
      return state - 1
  }
  return state
}

export const generateStore = () => {
  const reducerMap: ReducersMapObject<AppState> = {
    counter: counterReducer
  }
  return createStore(combineReducers(reducerMap))
}

```

で、利用側

```tsx
// Counter sample
import React, { Component } from "react"
import { connect } from "react-redux"
import { bindActionCreators, Dispatch } from "redux"
import {
  counterActions,
  AppState,
  CounterActionCreators
} from "../store"

// State -> Propsに変換する例。
// 変換不要なら type StateProps = AppState でいいだろう
type StateProps = {
  cnt: number
}
// 子のPropsはStatePropsとCounterActionCreatorsを持つ
type ChildProps = StateProps & CounterActionCreators

class CounterInner extends Component<ChildProps> {
  render() {
    return (
      <div>
        <div>{this.props.cnt}</div>
        <button onClick={this.props.increment}>+</button>
        <button onClick={this.props.decrement}>-</button>
      </div>
    )
  }
}

const connectCounter = connect(
  (state: AppState): StateProps => ({
    cnt: state.counter
  }),
  (dispatch: Dispatch) => bindActionCreators(counterActions, dispatch)
)
export const Counter = connectCounter(CounterInner)

```

この流れで一点だけヒジョーにイケてないのが `export interface CounterActionCreators extends ActionCreatorsMapObject<CounterAction>`の部分。

ActionCreatorsとして定義されているプロパティが例えば`keyof typeof counterActions`などで`increment | decrement`など推論がとれてくれればこんなものは不要になるのだが、現状上記のようなものだと`string | number`となってしまうため、致し方なく`CounterActionCreators`を定義している。

ここらへんもっといい方法を考えたい

### 解決策1: bindActionCreatorを使わない。

今回のイケてない部分、結局の所bindActionCreatorに頼っていることが面倒の原因と言える。

```tsx
import { Dispatch } from "redux"
import { AppState } from "~/client/store/store"
import { counterActions } from "~/client/store/counter"
type StateProps = {
  cnt: number
}
type DispatchProps = {
  dispatch: Dispatch
}
type ChildProps = StateProps & DispatchProps
class CounterInner extends Component<ChildProps> {
  render() {
    const { dispatch } = this.props
    const { increment, decrement } = counterActions
    return (
      <div>
        <div>{this.props.cnt}</div>
        <button onClick={(e) => dispatch(increment())}>+</button>
        <button onClick={(e) => dispatch(decrement())}>-</button>
      </div>
    )
  }
}
```
多少記述量が増えたと言えるが、どう考えても型のための記述量を考えたらマシとは十分言えそうだ

### 解決案2: Mapped Type使う（ボツ案）

ちょっとイマイチポイントは残るが、かなりマシな感じでいける。

```tsx

const increment = (): CounterAction => {
  return { type: "INCREMENT" }
}
const decrement = (): CounterAction => {
  return { type: "DECREMENT" }
}
const force = (num: number): CounterAction => {　// 引数をとる例
  return {
    type: "FORCE",
    count: num
  }
}

// ここでまとめ直しが必要
export const counterActions = { increment, decrement, force }
````

で、利用側はこうなる

```tsx
// typeof counterActionsをしてる。ただしこのactionが引数を正しくは認識出来ない
type ChildProps = StateProps & typeof counterActions
class CounterInner extends Component<ChildProps> {
  render() {
     // 省略
  }
}
```
## CreateActionをもっと推論する
Actionについてももう少しなんとかしたい。
例えばActionをここまで書いてしまえばpayloadまで推論出来る

```ts
// 元
// type CounterActionType = "INCREMENT" | "DECREMENT"
// type CounterAction = Action<CounterActionType>

type CounterAction = {
  type: "INCREMENT"
} | { 
  type: "DECREMENT" 
} | {
  type: "FORCE"
  count: number
}
```

## ActionCreatorとの重複を避ける
actionCreatorと重複するのを避けるなら下記のようにすると多少マシになる

```ts

// Flux-standardなaction。typescript-fsaからほぼパクってきた
interface AppAction<ActionName extends string, Payload = null> {
  type: ActionName
  payload?: Payload
  error?: boolean
  meta?: Object
}

type CounterAction =
  | AppAction<"INCREMENT">
  | AppAction<"DECREMENT">
  | AppAction<"FORCE", { count: number }>
```

上記の例ではpayloadを必須にしてない。必須にする場合は、combineReducerの部分を`ReducersMapObject<AppState, any>`などにすればなんとかなるようだ（ちょっと型がうまくあわせられなかった）

### さらにひと手間:Mapped Typeを使い再現する

MappedTypeでこういう感じに落ち着く

```ts
import { Action } from "redux"

// Actionに対して、Extraな値をMappedTypeにして追加する形で認識させたものを定義。AnyActionの改良版。
export type AppAction<T extends string, Extra extends {} = {}> = 
  Action<T> &
  { [K in keyof Extra]: Extra[K] }

type CounterAction =
  | AppAction<"INCREMENT">
  | AppAction<"DECREMENT">
  | AppAction<"FORCE", { count: number }>
```
これであればReducersMapObjectも受け入れてくれる。
この例ではpayloadは無くFlux Standard Action形式ではないが、TypeScriptであればFSAにこだわる理由も薄かろう

それと`|`や`&`が先頭に来ているが、これはprettierの結果なので深い意味はない。
https://github.com/prettier/prettier/issues/3986

### ActionTypeをEnumにしちゃう

オプショナルな話なので無くても良いが、ActionTypeをenumにしておくのも良い

```ts
enum ActionType {
  increment = "INCREMENT",
  decrement = "DECREMENT",
  force = "FORCE"
}
```

あとでActionTypeの名前変えたくなった！みたいなときにVSCodeにお任せ一発リファクタリングみたいなことが出来て夢がある。

# 最終型
とここまで考えてたらこのぐらいスッキリさせることが出来る

```tsx
export type AppAction<T extends string, Extra extends {} = {}> = 
  Action<T> & 
  { [K in keyof Extra]: Extra[K] }

enum ActionType {
  increment = "INCREMENT",
  decrement = "DECREMENT",
  force = "FORCE"
}

type CounterAction =
  | AppAction<ActionType.increment>
  | AppAction<ActionType.decrement>
  | AppAction<ActionType.force, { count: number }>

const increment = (): CounterAction => {
  return { type: ActionType.increment }
}
const decrement = (): CounterAction => {
  return { type: ActionType.decrement }
}
const force = (num: number): CounterAction => {
  return {
    type: ActionType.force,
    count: num
  }
}

export const counterActions = { increment, decrement, force }

export const counterReducer: Reducer<number, CounterAction> = (
  state = 0,
  action
) => {
  switch (action.type) {
    case ActionType.increment:
      return state + 1
    case ActionType.decrement:
      return state - 1
    case ActionType.force:
      return action.count
  }
  return state
}

```

```tsx

type StateProps = {
  cnt: number
}

type ChildProps = StateProps & typeof counterActions
class CounterInner extends Component<ChildProps> {
  render() {
    const { increment, decrement } = this.props
    return (
      <div>
        <div>{this.props.cnt}</div>
        <button onClick={increment}>+</button>
        <button onClick={decrement}>-</button>
      </div>
    )
  }
}

const connectCounter = connect(
  (state: AppState): StateProps => ({
    cnt: state.counter
  }),
  (dispatch) => bindActionCreators(counterActions, dispatch)
)
export const Counter = connectCounter(CounterInner)
```

# [TS 3.0] redux-actionのcreateActionを作る。

↓こっちの記事に分離した。
https://qiita.com/terrierscript/items/b9687f610a96ab964ab2

