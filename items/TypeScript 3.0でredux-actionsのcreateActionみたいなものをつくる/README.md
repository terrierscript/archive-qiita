# 前回までのあらすじ
https://qiita.com/terrierscript/items/b3f9dd95a4c7afe0b102

上記の記事に書いてたけどちょっと別物になってきたので記事わけた。

# もうちょっとなんとかしたかったこと
前回までのactionはボイラープレート多かったので、[redux-actions](https://github.com/redux-utilities/redux-actions)のcreateActionみたいなのがほしくなった。
本家のをそのまま使いたいところだが、payloadが`payload?`で定義されており若干今回のactionの付帯情報まで決定させたいということが出来なかった。

あと、TypeScript 3.0で出ていたTuple in Rest parameterの使い所じゃん！って思ってテンションがあがったので使いたかった。
ということで下記は3.0以上向けになるので注意

# コード

```ts

// Extraの部分を作成する関数。Tuple in Rest parameter大活躍！
type ExtraFunction<Arg extends any[], R> = (...args: Arg) => R
type ActionCreator<Arg extends any[], Action> = (...args: Arg) => Action

// 関数のoverload
export function createAppAction<A extends string>(
  type: A
): ActionCreator<any[], AppAction<A>>
export function createAppAction<A extends string, Arg extends any[], R>(
  type: A,
  fn: ExtraFunction<Arg, R>
): ActionCreator<Arg, AppAction<A, R>>

export function createAppAction(type, extraFunction?) {
  return (...args) => {
    if (extraFunction) {
      const extra = extraFunction(...args)
      return { type, ...extra }
    }
    return { type }
  }
}

```

利用側はこんな感じになる

```ts
const increment = createAppAction(ActionType.increment)
const decrement = createAppAction(ActionType.decrement)
const force = createAppAction(ActionType.force, (num: number) => {
  return {
    count: num
  }
})
```

また、このcreateAppActionの定義と[コメントいただいた](https://qiita.com/terrierscript/items/b3f9dd95a4c7afe0b102#comment-77f7521fdcec8ad716ad) `ReturnType`を駆使すると、更に楽ができる

```ts
// もとの定義
// type CounterAction =
//   | AppAction<ActionType.increment>
//   | AppAction<ActionType.decrement>
//   | AppAction<ActionType.force, { count: number }>

type CounterAction = ReturnType<
  typeof increment | typeof decrement | typeof force
>
```

気分が乗ってきたのでさらにactionをobjectでまとめてみるとこうなる

```ts
export type AppActionCreators<
  T extends { [key: string]: (...args) => any }
> = ReturnType<T[keyof T]>


export const counterActions = {
  increment: createAppAction(ActionType.increment),
  decrement: createAppAction(ActionType.decrement),
  force: createAppAction(ActionType.force, (num: number) => {
    return {
      count: num
    }
  })
}

type CounterAction = AppActionCreators<typeof counterActions>
```
