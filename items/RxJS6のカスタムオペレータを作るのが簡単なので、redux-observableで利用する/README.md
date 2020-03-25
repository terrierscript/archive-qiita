RxJS6でpipeオペレータが使えるようになったことで、カスタムオペレータを作るのも簡易になった。

https://github.com/ReactiveX/rxjs/blob/master/doc/pipeable-operators.md#build-your-own-operators-easily

上記にある通り、例えば既存のオペレータをまとめる程度であればこのぐらい簡単にできると例示されている。

```ts
/**
 * You can also use an existing operator like so
 */
const takeEveryNthSimple = (n: number) => <T>(source: Observable<T>) =>
  source.pipe(filter((value, index) => index % n === 0 ))
```

## redux-observableで使ってみる

例えばよく使う`tap`でconsole.logに表示して`ignoreElements`で消すというのはよくやるデバッグの手段だろう

```ts
export const pingEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    tap(console.log),
    ignoreElements()
  )
```

今回はこれをカスタムオペレータとして置き換えてみるとこうなる

```ts
const debug = () => <T>(source: Observable<T>) =>
  source.pipe(
    tap(console.log),
    ignoreElements()
  )

export const pingEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    debug()
  )

```


オペレータは`Observable`を受け取って`Observable`を返せば良い。例示に併せて、ここでは関数化している。debugが関数を返す関数になっているので、例えばtapに渡す値を変更できるようにしたいならこういう具合だろう

```ts
const debug = (tapFn = console.log) => <T>(source: Observable<T>) =>
  source.pipe(
    tap(tapFn),
    ignoreElements()
  )

export const pingEpic = (action$) =>
  action$.pipe(
    ofType("PING"),
    debug(item => console.log("This is DEBUG => ", item))
  )
```
