
「最初からreduxいれんのめんどくさいな〜。かといって後でcomponent書き換えるのもだるいんだよなー」みたいな微妙な感じの時にこんなの考えた。

# 考え方
- 結局`actions`を通じてなんかstateが変更される関数である
- componentsは`actions`のことを基本的に気にしていれば良い
- データとしてのpropsは、親元のstateを受け取れば良い

# コード
だいたい３つで動いた。
メインは`App.jsx`

```index.js
import React from 'react';
import App from './containers/App.jsx';

React.render(
  <App />,
  document.getElementById('root')
);
```

```containers/App.jsx
import React, { Component } from "react"
import Counter from '../components/Counter.jsx';

export default class App extends Component{
  constructor(){
    super()
    this.state = {
      counter: 0
    }
  }
  get actions(){
    // actionCreator + reducerでやってることをさらっとやる。
    return {
      increment : () => {
        this.setState({
          counter: this.state.counter + 1
        })
      }, 
      decrement : () => {
        this.setState({
          counter: this.state.counter - 1
        })
      }
    }
  }
  render(){
    let props = {
      counter: this.state.counter,
      ...this.actions
    }
    return <Counter { ...props }/>
  }
```

Counterはreduxのサンプルから拝借

```components/Counter.jsx
mport React, { Component, PropTypes } from 'react';

class Counter extends Component {
  render() {
    const { increment, decrement, counter } = this.props;
    return (
      <p>
        Clicked: {counter} times
        {' '}
        <button onClick={increment}>+</button>
        {' '}
        <button onClick={decrement}>-</button>
      </p>
    );
  }
}

export default Counter;
```

# まとめ
サンプル: http://inuscript.github.io/example-modoki-redux/
ソース: https://github.com/inuscript/example-modoki-redux

## 感想
- そんなに悪くない
- とはいえ場当たり的。わりと早い段階でめんどくさくなる。
- actionだけ通じてstateの変更するってことだけ考えるのはシンプルで良いかもしれない
