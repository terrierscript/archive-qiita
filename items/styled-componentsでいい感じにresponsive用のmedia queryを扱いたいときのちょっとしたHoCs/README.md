styled-componentsを使う上でなかなかresponsive用のmedia queryがしっくり来てなかったのを解決出来そうだったのでメモ

# コンポーネントから別な場合
この場合はreact-mediaがおすすめ。

https://github.com/ReactTraining/react-media

そのままでも使えるが、もしbreakpointが決まっているならこんな関数を用意すると良いだろう

```jsx

export const withMediaComponent = (DesktopComponent, MobileComponent) => {
  return props => {
    return (
      <Media query="(max-width: 768px)">
        {matches =>
          matches ? (
            <MobileComponent {...props} />
          ) : (
            <DesktopComponent {...props} />
          )
        }
      </Media>
    );
  };
};
```

使い方はこんな具合

```js
const SpItem = () => {
  return <div>sp</div>;
};
const PcItem = () => {
  return <div>pc</div>;
};
const Item = withMediaComponent(PcItem, SpItem);

export const MediaComponent = () => {
  return (
    <div>
      <Item />
    </div>
  );
};
```

# コンポーネントは一緒でstyleだけ変えたい場合
本当にやりたかった方。
styled-media-queryというのがある

https://github.com/morajabi/styled-media-query

だいたいこんな感じになる。

```js
import mediaQuery from "styled-media-query";

const mediaMobile = mediaQuery.lessThan("medium");

const Item = styled.div`
  color: red
  ${mediaMobile`
    color: blue;
  }
`
```
が、syntax highlightがなかったりちょっと書き方をもうちょっとしたい

そこでこんな関数を用意してみる。

```js
import mediaQuery from "styled-media-query";
export const mediaMobile = mediaQuery.lessThan("medium");

export const withMediaStyle = (Component, extend) => {
  return styled(Component)`
    ${mediaMobile`
      ${extend}
    `};
  `;
};
```

するとこんな具合に書ける。

```jsx
const BaseItem = styled.div`
  color: red;
`;

const spExtend = css`
  color: blue;
`;

const SomeItem = withMediaStyle(BaseItem, spExtend);

const SpSomeItem = styled.div`
  ${spExtend};
`;

export const MediaStyle = () => {
  return (
    <div>
      <SomeItem>aaa</SomeItem>
    </div>
  );
};
```

`withMediaStyle(もとのコンポーネント, メディアクエリで拡張するCSS)`と言った感じで書ける。

[CSS](https://www.styled-components.com/docs/api#css)関数はコンポーネント化せずにCSS部分を扱うものになる。
