
ReactDomで、下記のように直接`document.body`にレンダリングすると怒られる


```index.js
ReactDom.render(
　　<Main />, 
　　document.body
)
```

```
Warning: render(): Rendering components 
directly into document.body is discouraged,
since its children are often manipulated by third-party
scripts and browser extensions. 
This may lead to subtle reconciliation issues. 
Try rendering into a container element created for your app.
```

サードパーティscriptやextensionが`document.body`に依存してる可能性あるからやめておけという警告。とても真っ当。
productionコードなら、`<div id="container"></div>`のようにdivを書いてそちらを指定して回避する。

が、[budo](https://github.com/mattdesl/budo)などを使ってちょこちょこっと素振りやprototypingするときに、index.htmlをいちいち作りたくない。

# 解決策
`createElement`を直接`appendChild`してそれをターゲットとする。

```index.js
ReactDom.render(
　　<Main />,
　　document.body.appendChild(document.createElement('div'))
)
```

繰り返しになるが、budoのようなprototyping用の想定なので、productionではやらない。おとなしく`<div id="container"></div>`とhtmlに書く方が良い
