
何らかのMockを作るときなど、アバターアイコンを色々配色させたいことがある。
Twitterの昔あった卵アイコンみたいなやつ。
styled-componentsとcssのhue-rotateを利用すると雑にpng一枚用意するだけで出来る。

↓こんな感じにできる。
<img width="662" alt="React_App.png" src="https://qiita-image-store.s3.amazonaws.com/0/7307/84abae81-0029-22e3-705b-e06d7e01cd45.png">


「SVGでやったほうがいいんだけどちょっとだるい」みたいなときにおすすめ。
実装とか色々考えると圧倒的にsvgとかでやったほうがいいので繋ぎの手段としてどうぞ。

あと今回はstyled-componentsでやっているが、CSSの機能なのでsassとかでも出来るはず。
styled-components使うことで動的に処理できるという点が旨味

# 実装
例えばこんな画像を用意する
![icon.png](https://qiita-image-store.s3.amazonaws.com/0/7307/10d589c7-bc46-f61a-442d-f609f58fdf3e.png)

で、こんな具合に実装。

```js
import React, { Component } from "react";
import icon from "./icon.png";
import styled from "styled-components";

const randomAngle = () => {
  const num = 7; // ランダムに出したい色数
  return (Math.ceil(Math.random() * num) * 360) / num;
};

// ここでhue-rotate使う
const MockAvater = styled.img`
  filter: hue-rotate(${() => `${randomAngle()}deg`});
`;

class App extends Component {
  render() {
    return (
      <div className="App">
        <MockAvater src={icon} width="80" />
      </div>
    );
  }
}
```

もし「色数は変数を渡したい」みたいな場合であればこんな感じ


```js
const getAngle = (idx, max) => {
  console.log(idx);
  return (360 * idx) / max;
};
const MockAvaterManual = styled.img`
  filter: hue-rotate(${({ idx, max }) => `${getAngle(idx, max)}deg`});
`;

class App extends Component {
  render() {
    return (
      <MockAvaterManual src={icon} width="80" max={8} idx={0} />
    )
  }
}
```

