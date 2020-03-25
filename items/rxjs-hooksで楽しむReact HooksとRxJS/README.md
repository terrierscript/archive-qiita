[16.8で正式にリリースされる](https://github.com/facebook/react/pull/14692/files)React Hooks。

[rxjs-hooks](https://github.com/LeetCode-OpenSource/rxjs-hooks/)を使うとHooksでRxJS使えていい感じだったのでまとめたい

# 1. 検索機能を実装する

以前にredux-observableでやったことのあった[検索機能](https://qiita.com/terrierscript/items/c672ddad4c24936b1061)をhooksでやってみる。

* [redux-observableでの最終型](https://qiita.com/terrierscript/items/c672ddad4c24936b1061#%E5%AE%8C%E6%88%90%E5%93%81--%E6%9C%80%E7%B5%82%E5%BD%A2)

## 前提
ひとまず前回と同じくAPIとコンポーネントをざっくり作る

```ts
// api.ts

import fetchJsonp from "fetch-jsonp"
import qs from "querystring"

export const searchApi = (word) => {
  const baseURL = "https://ja.wikipedia.org/w/api.php"
  const params = {
    action: "opensearch",
    format: "json",
    search: word
  }
  const url = `${baseURL}?${qs.stringify(params)}`
  return fetchJsonp(url)
    .then((response) => response.json())
    .then((json) => json[1])
}
```

```tsx
const Search = ({ word, changeInput }) => (
  <div>
    <input value={word} onChange={(e) => changeInput(e.target.value)} />
  </div>
)

const Result = ({ result }) => (
  <div>
    <h3>Result</h3>
    <ul>
      {result.map((r, i) => (
        <li key={i}>{r}</li>
      ))}
    </ul>
  </div>
)
```

ここから検索につながる部分を作っていく


## パターンA: `useState`と`useObservable`を使うパターン

おそらくこっちが適切であろうパターン。

```tsx
import React, { useState } from "react"
import { useObservable } from "rxjs-hooks"
import { map, bufferTime, distinctUntilChanged, switchMap } from "rxjs/operators"

const useSearch = (word) =>
  useObservable(
    (word$) => // 渡されるwordのstream
      word$.pipe(
        debounceTime(400),
        distinctUntilChanged(),
        switchMap((word) => searchApi(word)),
        map((result) => (result !== undefined ? result : []))
      ),
    [], // 結果の初期値
    [word] // input。配列にして渡す必要があることに注意
  )

export const App = () => {
  // word自体はuseStateで取り出す
  const [word, setWord] = useState("")
  // これを独自実装したuseSearchへ渡す
  const result = useSearch(word)
  return (
    <div>
      <Search changeInput={setWord} word={word} />
      <Result result={result} />
    </div>
  )
}
```

この場合は検索は責務を絞ることができるので、下記のように分離することもできる

```tsx
const SearchAndResult = (word) => {
  let result = useSearch(word)
  return <Result result={result} />
}

export const App2 = () => {
  const [word, setWord] = useState("")
  return (
    <div>
      <Search changeInput={setWord} word={word} />
      <SearchAndResult word={word} />
    </div>
  )
}
```


## パターンB: `useEventCallback`を使うパターン（イマイチ）

もう一つやり方があるので記述する。「どうしても全部RxJSに任せたいんだ！」という場合は良いかもしれないがおそらくこの場合においてはデメリットのほうが大きいだろう

```tsx
import React from "react"
import { useEventCallback } from "rxjs-hooks"
import { map, startWith, distinctUntilChanged, debounceTime, switchMap } from "rxjs/operators"

export const App = () => {
  const initial: [string, string[]] = ["", []]
  const [keyboardCallack, [word, result]] = useEventCallback(   (event$) =>
      combineLatest(
        event$, // 入力を即時反映させるために、素のstremを流す必要がある
        event$.pipe(
          startWith([]), // 初期値をstatWithで指定
          debounceTime(400),
          distinctUntilChanged(),
          switchMap((word) => searchApi(word)),
          map((result) => result || [])
        )
      ),
    initial
  )
  return (
    <div>
      <Search changeInput={keyboardCallack} word={word} />
      <Result result={result} />
    </div>
  )
}
```

`useEventCallback`は結果の１つ目がコールバック、二つ目がStreamの結果が返ってくる。

ただSearchのinputを入力に正しく同期するために、注1のように入力をそのまま流してcombineLatestを使ったり、startWithを使って入力の度にstreamが来るようにする工夫が必要になる。

また、入力のコールバックに強く依存してしまう部分も難点だろう。

# 2. `useEventCallback`で秒間クリック数を取る
上記はinputに対する入力だったため`useObservable`のほうが都合が良かったが、クリックなどマウスイベント関連なら有効になる。

例示として秒間クリックを取得して出すようなことを考えてみるとこんな具合になりそうだ（そんなユースケースがあるかどうかは不明）

```jsx
import { useEventCallback } from "rxjs-hooks"
import { map, bufferTime, distinctUntilChanged, tap } from "rxjs/operators"

const Button = ({ updateCount }) => {
  const [eventCallback, result] = useEventCallback(
    (event$) =>
      event$.pipe(
        bufferTime(1000), // 1秒間bufferする
        map((v) => v.length), // 受け取ったイベント＝クリック数
        distinctUntilChanged()
        tap((result) => updateCount(result))
      ),
    0
  )
  return (
    <div>
      <button onClick={eventCallback}>Click!</button>
    </div>
  )
}

export const App = () => {
  // 状態自体はstateに収める
  const [count, setCount] = useState(0)
  return (
    <div>
      <Button updateCount={setCount} />
      <div>{count} click / second</div>
    </div>
  )
}
```

# 3.(おまけ) ↑↑↓↓←→←→BAを検出する

[有名なコマンド](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%8A%E3%83%9F%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89)が入力されたら発火するようなイースターエッグを仕込みたいときに使えるかもしれない。

```jsx
import { useObservable } from "rxjs-hooks"
import { map, filter, scan } from "rxjs/operators"
import { fromEvent, pipe } from "rxjs"

const arrowMap = {
  ArrowUp: "↑",
  ArrowDown: "↓",
  ArrowLeft: "←",
  ArrowRight: "→"
}

const convertVisible = (key) => {
  if (arrowMap[key]) return arrowMap[key]
  if (key.length === 1) return key.toUpperCase()
  return null
}

// keydownイベントのストリームを配列化
export const useKeyCommnads = (length) =>
  useObservable(
    () =>
      fromEvent(document, "keydown").pipe(
        pluck("key"),
        map(convertVisible),
        filter((value) => !!value),
        scan<string>((curr, next) => [...curr, next], []),
        map((cmd: string[]) => cmd.slice(-1 * length))
      ),
    []
  )

// 流れてきたコマンドが任意のものかどうかを検証する
export const useTargetCommand = (targetCommand) => {
  const keyeventLog = useKeyCommnads(targetCommand.length)
  return {
    keyeventLog,
    konami: targetCommand.join("") === keyeventLog.join("")
  }
}

export const App = () => {
  const command = ["↑", "↑", "↓", "↓", "←", "→", "←", "→", "B", "A"]

  const { keyeventLog, succeed } = useTargetCommand(command)
  return (
    <div>
      <div>please key type: {keyeventLog}</div>
      <div>{succeed ? <Success /> : null}</div>
    </div>
  )
}
```

斜め入力まで考慮したい場合のサンプルは下記においたので興味があればご参照を
https://codesandbox.io/s/mzxx7wy318?module=%2Fsrc%2Findex.tsx

# 他のRxJS + Hooks手法

* https://github.com/jadbox/rxhooks_examples/blob/master/src/App.tsx
* https://github.com/jadbox/rxhooks/
* https://mobile.twitter.com/YassineElouafi2/status/1093562456302108673
* https://mobile.twitter.com/tavaresn/status/1092956526661165057
