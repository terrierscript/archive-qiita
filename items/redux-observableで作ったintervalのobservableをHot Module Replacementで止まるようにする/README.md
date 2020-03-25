
epicでintervalを利用するような場合、HMRが効いてるとそのまま動いてしまう。
storeはHMRで更新されるので影響は無いが、consoleなどで追っかけているとObserverが動いていてちょっと気持ち悪い

# 解決方法： Subjectを利用して`module.hot.dispose`で発火して止める

```js
import { combineEpics } from "redux-observable"
import { interval, Subject } from "rxjs"
import { map, takeUntil } from "rxjs/operators"

const disposer = new Subject()

export const timerEpic = () => {
  return interval(1000).pipe(
    takeUntil(disposer), // disposerが動くまでtimerを動かす
    map((time) => ({
      type: "TIMER",
      value: time
    }))
  )
}

if (module.hot) {
  module.hot.dispose((data) => {
    disposer.next()
  })
}

```

typescriptの場合、`module.hot`は`@types/webpack-env`に定義されているので入れると良い

### HMR部分を分離する

こんな感じにHot Reload関連の部分を切り出しておくのも良いだろう。

```ts
// hotReload.ts
import { Subject } from "rxjs"
import { takeUntil } from "rxjs/operators"

const hmrDisposer = new Subject()

export const registerDisposeHandler = (module) => {
  if (module.hot) {
    module.hot.dispose(() => hmrDisposer.next())
  }
}

export const takeUntilHotReload = () => takeUntil(hmrDisposer)
```

ただし`module.hot`のフックは全てのdependencyのルートであるrootEpicの記述箇所と一緒にしておかないと上手く発火されないので注意

```ts
export const rootEpic = combineEpics(pingEpic)

// 全てのepicが更新されたタイミングでdisposeを呼び出すように仕掛ける
registerDisposeHandler(module)
```
