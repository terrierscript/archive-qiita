redux-observableで１つのactionから最終的に複数のactionを発火する方法を毎回ググってるのでメモる

# 中間にActionをつくる

これが一番スタンダードだと思われる。8割は解決するはず。

今回は例えばこんなボタンを用意した時を考える

```jsx
const Button = ({ dispatch }) => (
  <button onClick={(e) => dispatch({ type: "PING" })}>
    Ping
  </button>
)
```

中間Actionを用意する場合のEpicはこうなる。

```ts
// ボタンからPINGのActionが来たら中間のTEMP_PINGを発火
const seedEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    filter(action => !!action.payload), // 例としてここでは事前処理のepicでfilterをしてみる
    map((action: any) => ({
      type: "TEMP_PING",
      payload: action.payload
    }))
  )

// TEMP_PINGを受けて値を2倍にするEpic
const doubleEpic = (action$) =>
  action$.pipe(
    ofType("TEMP_PING"),
    map((action: any) => ({
      type: "PONG",
      payload: action.payload * 3
    }))
  )

// TEMP_PINGを受けて値を3倍にするEpic
const tripleEpic = (action$) =>
  action$.pipe(
    ofType("TEMP_PING"),
    map((action: any) => ({
      type: "PUNG",
      payload: action.payload * 4
    }))
  )

export const pingEpic = combineEpics(debug(), seedEpic, doubleEpic, tripleEpic)

```

# mergeMapからの配列で返す

単純に複数actionを一関数中で発火出来るならmergeMapと配列を使う事でも実現出来る

```ts
const pingEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    mergeMap((action: any) => {
      return [
        {
          type: "PONG",
          payload: action.payload * 2
        },
        {
          type: "PUNG",
          payload: action.payload * 3
        }
      ]
    })
  )
```


# Partitionで分離する（できる場合）

ここまでは「どっちも発火したい」であったが条件によって変えるなら`partition`で十分だろう

```ts
export const pingEpic = (action$) => {
  const [even, odd] = action$.pipe(
    ofType("PING"),
    partition((action: any) => action.payload % 2 === 0)
  )
  return merge(
    // 偶数の場合2倍
    even.pipe(
      map((action: any) => ({
        type: "PONG",
        payload: action.payload * 2
      })),
    ),
    // 奇数の場合3倍
    odd.pipe(
      map((action: any) => ({
        type: "PUNG",
        payload: action.payload * 3
      }))
    )
  )
}
```

# おまけ：Rxを駆使する
ここから下はほぼ使うことはない。思考実験に近い。
一応こういうのもあるという紹介

## mergeMapとofで対応する

こんな感じでmergeMapを使う事が出来る
後述するがこれは冗長だろう

```ts
export const pingEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    mergeMap(action => {
      const source$ = of(action)
      return merge(
        source$.pipe(
          map((action: any) => ({
            type: "PONG",
            payload: action.payload * 2
          })),
        ),
        source$.pipe(
          map((action: any) => ({
            type: "PUNG",
            payload: action.payload * 3
          }))
        )
      )
    })
  )

```

## mergeで２つを組み合わせる
上記の冗長さは下記のように分岐するとこまでをこんな感じで切り出して後でmergeするやり方もある。partitionでのやり方に近い

```ts
export const pingEpic = (action$) => {
  // ここまでが共通
  const source$ = action$.pipe(
    ofType("PING"),
  )
  // 分岐するsourceをそれぞれ Observable.mergeで混ぜる。
  return merge(
    source$.pipe(
      map((action: any) => ({
        type: "PONG",
        payload: action.payload * 2
      })),
    ),
    source$.pipe(
      map((action: any) => ({
        type: "PUNG",
        payload: action.payload * 3
      })),
    )
  )
}
```

例えばこれだと下記のように「連打されたActionをただログ出すだけと連打を検知するものに分けたい」ようなことも可能（使い所は不明）

```ts
export const pingEpic = (action$) => {
  const source$ = action$.pipe(
    ofType("PING"),
  )
  return merge(
    source$.pipe( // こっちはLOG出力だけ
      map((action: any) => ({
        type: "LOG",
        payload: action.payload
      })),
    ),
    source$.pipe(
      bufferTime(1000), // 連打
      filter((items: any[]) => items.length > 0),
      map((actions: any[]) => { // 受け取ったactionが配列になる
        return {
          type: "MASH",
          payload: actions.length
        }
      })
    )
  )
}
```


# 別途Subjectを作る

更にやらないであろう手法。
思いついたのでせっかくなので供養する。

```ts

// Subjectを別に作る。
// 本来redux-observableがSubjectを管理しているので、ここにわざわざ作るのはいったい、という感じ
const pongSubject = new Subject()
const pungSubject = new Subject()

const mainEpic = (action$) => {
  return action$.pipe(
    ofType("PING"),
    // tapでそれぞれsubjectを発火
    tap((action: any) => pongSubject.next({ type: "PONG", payload: action.payload * 2 })),
    tap((action: any) => pungSubject.next({ type: "PUNG", payload: action.payload * 3 })),
    ignoreElements() // tapだけでそのまま返すと無限PINGで死ぬ
  )
}

// SubjectをasObservableで変換してEpicと扱う
const pongEpic = () => pongSubject.asObservable()
const pungEpic = () => pungSubject.asObservable()

export const pingEpic = combineEpics(mainEpic, pongEpic, pungEpic)
```

使い所は殆どないが、例えばどうしてもメインの処理があってそこにうまくログのような処理をはさみたい場合とかがあれば使いどころになるかもしれない

```ts
const logSubject = new Subject()

const mainEpic = (action$) => {
  return action$.pipe(
    ofType("PING"),
    tap((action: any) => logSubject.next({ type: "LOG", payload: action.payload })),
    bufferTime(500), // 連打
    filter((items: any[]) => items.length > 0),
    map((actions: any[]) => { // 受け取ったactionが配列になる
      return {
        type: "MASH",
        payload: actions.length
      }
    }),
  )
}
const logEpic = () => logSubject.asObservable()

export const pingEpic = combineEpics(mainEpic, logEpic)
```
