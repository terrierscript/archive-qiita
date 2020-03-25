
reactのアニメーション関連で結構メジャーっぽいreact-motionが意外ととっつきにくかったので、簡易なProgressバー作って素振りした。

# 生成物
https://inuscript.github.io/example-motion-routing/

![Kapture 2017-04-20 at 0.09.01.gif](https://qiita-image-store.s3.amazonaws.com/0/7307/37b7bb6f-c60b-ec06-3603-3007cc77d1d8.gif "Kapture 2017-04-20 at 0.09.01.gif")

# 作り方

まずAnimationの基となるComponentを作る。
css周りは`styled-components`を使った。

```js
import styled from "styled-components"

// 不慣れであれば<div className="BarContainer" />みたいに読み替えてもらえると良い。
const BarContainer = styled.div`
  width:500px;
  position: relative;
  height: 20px;
  background: lightgray;
`
const BarInnter = styled.div`
  display: block;
  position: relative;
  overflow: hidden;
  height: 100%;
  width: 0;
  background-color: darkslateblue;
`

// 外から`width`を受け取って`style`によってwidthが変わるようなComponentを作る
const Bar = ({width}) => {
  return (
    <BarContainer>
      <BarInnter style={ {width: `${width}%`} } />
    </BarContainer>
  )
}
```

ここらへんから`react-motion`の出番。

```js
import { Motion, spring } from "react-motion"

export class ProgressBar extends React.Component {
  shouldComponentUpdate(nextProps){
    // 不正な値が来たら更新を無視する
    // (ただ動かすだけであれば不要だが、あったほうが良い）
    const { progress } = nextProps
    return !isNaN(parseInt(progress, 10))
  }
  render() {
    const progress = parseInt(this.props.progress, 10)
    return <Motion defaultStyle={{p: 0}} style={{p: spring(progress)}}>{ (value) => {
      return <div>
        <div>{Math.ceil(value.p)}%</div>
        <Bar width={value.p} />
      </div>
    }}</Motion>
  }
}
```

`<Motion>`のchildrenがFunctionになっており、そこにanimationに従った値が入ってくる。

### spring??
`spring`はanimationの動きを調整する関数。`spring(変更後の値, option)`という感じ。今回は外から渡される`progress`の値まで動かすという形にしている。

試しに`spring(progress, {stiffness: 400, damping: 5}`とかしてみるとびよんびよんする。楽しい
![spring](https://qiita-image-store.s3.amazonaws.com/0/7307/52bb00e0-81fc-83fa-09d5-d8ee90513532.gif "Kapture 2017-04-20 at 0.01.21.gif")


### 複数のanimation値設定

styleの値はobjectなので、複数の値でそれぞれ`spring`のアニメーションを変えるのも可能。

```js
// 例えば色も変わるprogressにしてみる
const Bar = ({width, color}) => {
  return (
    <BarContainer>
      <BarInnter style={ {width: `${width}%`, backgroundColor: color} } />
    </BarContainer>
  )
}

export class ProgressBar extends React.Component {
  :
  :
  render() {
    const progress = parseInt(this.props.progress, 10)
    return <Motion defaultStyle={{p: 0, color: 0}} style={{
      p: spring(progress), 
      color: spring(progress, {stiffness: 1000, damping: 5} )
    }}>{ (value) => {
      return <div>
        <div>{Math.ceil(value.p)}%</div>
        <Bar width={`${value.p}`} color={`hsl(${value.color*2}, 50%, 50%)`} />
      </div>
    }}</Motion>
  }
}
```

![Kapture 2017-04-20 at 0.44.00.gif](https://qiita-image-store.s3.amazonaws.com/0/7307/c354405e-ddb5-2b44-dc48-a83cc1f06877.gif "Kapture 2017-04-20 at 0.44.00.gif")

色アニメーションだけ変える需要はなさそうだけど、使い所はありそう

# react-router(v4)と組合わせる

[公式サンプル](https://reacttraining.com/react-router/web/example/animated-transitions)だとcss-transition-groupを利用しているので、違いを知る意味で素振りしてみる。

```js
import {
  BrowserRouter,
  Route
} from 'react-router-dom'

// gh-pages向けにbaseUrlある設定にしている
const baseUrl = "/example-motion-routing" 

const App = () => {
  return (
    <BrowserRouter basename={baseUrl}>
      <div className="App">
      　{/*どこにも該当しない時用のredirect*/}
        <Route exact path="/" render={() => (
          <Redirect to="/40"/>
        )}/>
        <Route path={`/:progress`} component={ProgressRoute} />
        <Links />
      </div>
    </BrowserRouter>
  );
}

// Routingを受けるComponent。
// parmeterをProgressBarへ引き渡す
const ProgressRoute = ({match}) => {
  const { progress } = match.params
  return <ProgressBar progress={progress} />
}
```

基本的な思想は変わらない。URLパラメータを受け取り、それをreact-motionを使っているComponentへ渡している。
結果としては結構すっきりした。react-routerとreact-motionは割と相性良いかもしれない。
