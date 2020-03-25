
わりかし似たようなコンポーネントを作るとき、クラス継承みたいなことがやりたくなったりする。

例えば今回はactiveフラグの状態によって切り替わるiconなんかを作りたくなった。

# 何も考えずに抽象クラスくさいことをやってみる
うっかりclass構文が作れるので、これを利用してabstractっぽい抽象クラスを書いてみる。

```bad.jsx
class Icon extends React.Component{
  activeClass(){
    throw new Error("Not implemented error")
  }
  inactiveClass(){
    throw new Error("Not implemented error")
  }
  render(){
    var {key, active} = this.props;
    const classes = active ? this.activeClass() : this.inactiveClass()
    const cls = classNames("fa", classes);
    return (<i
      key={key}
      className={cls}
    />);
  }
}

class StarIcon extends Icon{
  activeClass(){
    return "fa-star"
  }
  inactiveClass(){
    return "fa-star-o"
  }
}
```

うむ。throwとかやってて大変よろしくなさそう。
なんとかならんものか

# spread attributesを使う
[transferring props](https://facebook.github.io/react/docs/transferring-props.html)のページと[spread attributes](https://facebook.github.io/react/docs/jsx-spread.html#spread-attributes)を見てみる

```jsx
  var props = {};
  props.foo = x;
  props.bar = y;
  var component = <Component {...props} />;
```
ふむ。なるほど。propsを一気に渡せるとな。

## やってみる
考え方としてはいわゆる「継承より移譲」というやつ。
StarIconは単にIconにpropsを渡しているだけ。

```good.jsx

class Icon extends React.Component{
  render(){
    var {key, classNames} = this.props;
    const classes = cx("fa", classNames);
    return (<i
      key={key}
      className={classes}
    />);
  }
}

class StarIcon extends React.Component{
  render(){
    var classNames = this.props.active ? "fa-star" : "fa-star-o"
    return (<LevelIcon { ...this.props } 
    	classNames={classNames} />)
  }
}
```

あとは制約をちゃんと掛けたければPropTypesをつけると。

```jsx
Icon.PropTypes = {
  clssName: React.PropTypes.string,
  key: React.PropTypes.string,
}
StarIcon.PropTypes = {
  active: React.PropTypes.boolean
}

```

いいんじゃないでしょうか。
