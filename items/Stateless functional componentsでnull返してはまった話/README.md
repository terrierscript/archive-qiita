
React 0.14から入ったStateless functional components

どうもnullを返してしまうとRenderが無いと怒られてしまうらしい。

```js
const FooItem = ({ show }) => {
  if(!show){
    return null
  }
  return (
    <div>some component</div>
  )
}
```

解決策としてはこんな感じ

```js
const FooItem = ({ show }) => {
  if(!show){
    return <noscript />
  }
  return (
    <div>some component</div>
  )
}
```

`<noscript>`に深い意味があるわけでもないが、Reactの標準的な動作がこれらしいのであわせた。
spanでも多分別に良い。

Animation関連でもnull返した際にエラーがあったので、もしかするとnull返すComponentはそもそもあんまり良くないのかもしれない。


ちなみに、現象としてはissueに上がっていた。
https://github.com/facebook/react/issues/5218
でもshadowRenderに限定されてしまっている。
