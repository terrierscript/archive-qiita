Reactでアニメーションをさせたい時、[react-pose](https://popmotion.io/pose/)や[react-spring](https://www.react-spring.io/)などのライブラリがある。これらライブラリはすごくリッチなのだがちょちょっとCSSのtransitionを使うぐらいの時には若干オーバーになる。

一方で過去公式だった（今はコミュニティ運用） [react-transition-group](https://github.com/reactjs/react-transition-group)もある。こちらは非常にシンプルだ。

`styled-componets`でさくっとアニメーションさせたい時、`react-transition-group`の`<Transition>`を使うと割とさくっとできることに気付いたのでまとめたい。

# 例示

まずTransitionを利用する側を書く。
この例ではReact Hooksを使っているが別にそこは本筋ではないのでclass componentsでも問題無い。

```jsx
import { Transition } from "react-transition-group"
import { Animation } from "./Animation"

export const AnimateItem = () => {
  // animationの状態を持つのにstateが必要。ここだけは面倒だけどしょうがない
  const [animate, setAnimate] = useState(false)

  // ボタンを押したらtrueになって3000ms後にfalseになるように仕組む
  const doAnimate = useCallback(() => {
    setAnimate(true)
    setTimeout(() => {
      setAnimate(false)
    }, 3000)
  }, [])

  return (
    <div>
      {/* Transition は `in` に渡した値で変化する */}
      <Transition in={animate} timeout={500}>
        {(state) => (
          // stateは下記の順序で変遷する
          // exited -> entering -> entered -> exiting -> exited
          <Animation state={state}>Hello</Animation>
        )}
      </Transition>
      <button onClick={doAnimate}>Animate</button>
    </div>
  )
}
```

次にanimationを設定するstyled-components側を作る。
propsのcallbackを受け取れるのでこれに応じて値を変えるだけだ。

```js
// Animation.js
import styled from "styled-components"

export const Animation = styled.div`
  transition: 0.5s;
  width: 300px;
  height: 200px;
  /* 要素を動かす */
  transform: translateX(
    ${({ state }) => (state === "entering" || state === "entered" ? 400 : 0)}px
  );
  /* 色を変える */
  background: ${({ state }) => {
    switch (state) {
      case "entering":
        return "red"
      case "entered":
        return "blue"
      case "exiting":
        return "green"
      case "exited":
        return "yellow"
    }
  }};
`
```

Animationの部分がちょっと分厚くなったり汚くなりがちなので、例えば基礎となる要素とanimation部分を分けて書くのもよいだろう

```jsx

const BaseItem = styled.div`
  width: 300px;
  height: 200px;
`

export const Animation = styled(BaseItem)`
  transition: 0.5s;
  transform: translateX(
    ${({ state }) => (state === "entering" || state === "entered" ? 400 : 0)}px
  );
`
```

# 結果

![a.gif](https://qiita-image-store.s3.amazonaws.com/0/7307/47a73edc-87f7-a1b4-ae9d-4d0d91bf0e07.gif)

# 例えばfadeIn / fadeOutするなら

この理屈で例えばアニメーションの前後で要素を消したいfadeInのようなものを作るならこんな感じだろう

```jsx
export const Fade = styled.div`
  transition: 0.5s;
  opacity: ${({ state }) => (state === "entered" ? 1 : 0)};
  display: ${({ state }) => (state === "exited" ? "none" : "block")};
`
```

もしくは若干開始タイミングの制御が雑でもよければ`unmountOnExit`、 `mountOnEnter`を利用して下記のような方法もある

```jsx

export const Fade2 = styled.div`
  transition: 5s;
  opacity: ${({ state }) => (state === "entered" ? 1 : 0)};
`

const Item = () => {
  // ...
  return <Transition in={animate} timeout={500} unmountOnExit mountOnEnter>
    {(state) => <Fade2 state={state}>Fade In</Fade2>}
  </Transition>
}

```

