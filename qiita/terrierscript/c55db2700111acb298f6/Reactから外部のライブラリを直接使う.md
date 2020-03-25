
Reactで例えばグラフなんかを生成したい場合、d3やChartistを使いたくなる。
いくつかグラフ用のコンポーネントは散見されるが、簡単なものであればスッと利用できそうだが、ちょっと複雑なカスタマイズなどするにはなかなか厳しそうに見えた。

なのでそのまま使うにはどうするかみたいなのを考えた。

（基本的に元のライブラリにパフォーマンスが引きづられがちになったりと若干の邪道感があるやり方になっているのでそこらへんは要注意）


# Example
今回は[Chartist](https://gionkunz.github.io/chartist-js/)を使った。

下記のような感じでRendererを用意する。

```ChartistRenderer.js
class ChartistRenderer{
  render(targetSelector, props){
    let {data} = props
    // 必要があればここでデータ加工したり。
    var chart = new Chartist.Bar(targetSelector, {
      labels: data[0],
      series: [ data[1] ]
    })
  }
}
```

関数名とかはReact.Componentに気持ちだけ合わせておく。
この程度ならComponent側にメソッドとして持たせても良かったのだけど、実際に扱おうとするとこの辺りの処理が肥大化するし、Reactからそこらへんの感心事は切り出したい感じしたのでやっておく。

で、あとはコンポーネント。

```Chart.jsx
class Chart extends React.Component{
  constructor(){
    super()
    this.chartClassName = "chart"
    this.renderer = new ChartistRenderer()
  }
  componentDidMount(){
    this.renderChart()
  }
  componentDidUpdate(){
    this.renderChart()
  }
  renderChart(){
    var targetSelector = "." + this.chartClassName    
    this.renderer.render(targetSelector, this.props)    
  }
  render(){
    return <div className={this.chartClassName}  ></div>
  }
}
```
renderではChartistのコンテナとなるdivだけ作る。
そして`componentDidUpdate`と`componentDidMount`でこのコンテナに対してrenderingをする。
renderの際にレンダリング対象のクラス名とデータ（propsを利用）を渡す。
propsでデータを渡すことで、propsがが変更されれば`componentDidUpdate`がフックされてくれるという仕組み。

あとは起動部分。適当に。

```app.jsx
var data = [
  ["foo", "baz", "bar"],
  [1, 10, 20]
]

var body = document.querySelector('.container')
React.render(<Chart data={data} />, body);

```

おしまい！

### サンプル
jsbinのbabelモードうまく使えnなかったのでcodepenで
http://codepen.io/suisho/pen/GJEwaP/

![A_Pen_by_suisho_と_ReactでReact以外のライブラリをそのまま使う_と_Developer_Tools_-_http___codepen_io_suisho_pen_GJEwaP_.png](https://qiita-image-store.s3.amazonaws.com/0/7307/0bb58b20-d045-587d-9a23-234d184e9eb9.png "A_Pen_by_suisho_と_ReactでReact以外のライブラリをそのまま使う_と_Developer_Tools_-_http___codepen_io_suisho_pen_GJEwaP_.png")

# 参考
あとで調べたらd3で似たようなこと考えている人いた。
http://nicolashery.com/integrating-d3js-visualizations-in-a-react-app/
