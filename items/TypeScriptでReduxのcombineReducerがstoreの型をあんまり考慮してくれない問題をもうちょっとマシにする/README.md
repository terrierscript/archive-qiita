# TL, DR
* combineReducerの型がanyになっていて、正しくないものが許容されている。
* なのでここに補助的な自前の型を作ることでなんとかしようという試み
* 次期バージョン(redux 4〜)で公式にサポートされるので、待てるならば待って良い

# 詳細：combineReducerの現状の型の問題点
`combineReducer`は、下記のように定義されている。
https://github.com/reactjs/redux/blob/master/index.d.ts

```ts
export interface ReducersMapObject { [key: string]: Reducer<any> }
export function combineReducers<S>(reducers: ReducersMapObject): Reducer<S>;
```

返り値となるState型`<S>`を指定できるものの、`ReducersMapObject`が`Reducer<any>`とされているため、stateと違うredcerを入れてもエラーになってくれない

```ts
const fooReducer = (state: number) => {
  return state
}
const bazReducer = (state: string) => {
  return state
}

export interface CombinedState {
  foo: number,
  baz: string,
}

// こんな感じになっていてもエラーにならない
export const rootReducer = combineReducers<CombinedState>({
  foo: bazReducer,
  baz: fooReducer
})
```

なので、自前で型を作って対応する必要がある。

## combineReducerとstateを整合する型

適当にこんな感じで作った。

```ts

export type CombineReducerMap<S extends {}> = { [K in keyof S]: Reducer<S[K]> }
```

こんな感じで使う

```ts
export interface CombinedState {
  foo: number,
  baz: string,
}

// おかしなreducerの場合はこの変数がエラーとなる
const reducerMap : CombineReducerMap<CombinedState> = {
  foo: fooReducer,
  baz: bazReducer
}

export const someReducer = combineReducers<CombinedState>(reducerMap)

```

`combineReducers`自体の型を上書きする方が良さそうだったが、さくっとやれなかったので一旦これで妥協した。

## 補足

nextブランチでは既にこの問題は修正されているので、一旦我慢しておいてもそのうちなんとかなりそうな見込みである
https://github.com/reactjs/redux/pull/1413
https://github.com/reactjs/redux/blob/next/index.d.ts

# 最終的なサンプル

```ts
import { Reducer, combineReducers } from "redux"


export type CombineReducerMap<S extends {}> = { [K in keyof S]: Reducer<S[K]> }


const fooReducer = (state: number) => {
  return state
}
const bazReducer = (state: string) => {
  return state
}

export interface CombinedState {
  foo: number,
  baz: string,
}

const reducerMap : CombineReducerMap<CombinedState> = {
  foo: fooReducer,
  baz: bazReducer
}

export const rootReducer = combineReducers<CombinedState>(reducerMap)


// 下記はエラーになる
const invalidReducerMap : CombineReducerMap<CombinedState> = {
  foo: bazReducer,
  baz: fooReducer
}

```



