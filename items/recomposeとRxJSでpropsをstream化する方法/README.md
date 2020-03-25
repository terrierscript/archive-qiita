長年recomposeでのObservable利用というのが理解出来なかったが、一部唐突に理解できたので忘れる前にメモ。

# 前提の考え：propsだけをstreamとして考える

とにかくRxを考える上で自分の理解を妨げていたのが「Everything is stream」という考え方だった。

[redux-observableを覚えた際](https://medium.com/@terrierscript/redux-observable%E3%82%92%E9%80%9A%E3%81%98%E3%81%A6rx%E3%82%92%E5%AD%A6%E3%82%93%E3%81%A0%E8%A9%B1-e9eadc75161d)は、streamとなるのがReduxのActionだけであると限定出来たから理解できたと考えている。

そのため今回は `mapPropsStream` の使い方のみフォーカスする。そのためstreamになるのはコンポーネントに渡され続けるpropsだけだ。
（それ以外のものは未だに使い方がいまいちわかってない所がある。特に `componentFromStream`はいまいち用途を想像出来てない。）

# 例：文字数カウンター

例えばこんな文字数カウンターを考える

```jsx
    export class TextArea extends React.Component {
      constructor(props) {
        super(props)
        this.state = {
          text: ""
        }
      }
      render() {
        return (
          <div>
            <textarea
              value={this.state.text}
              onChange={(e) => {
                this.setState({ text: e.target.value })
              }}
            />
            <TextCounter text={this.state.text} />
          </div>
        )
      }
    }
```

TextCounterの実装は一番愚直に考えればこんな感じだろう

```jsx
    const TextCounter = ({ text }) => {
      const count = text.count
      return (
        <div>
          <div>
            {count}
            文字
          </div>
          <div>{text}</div>
        </div>
      )
    }
```

ここで文字入力の度に更新が走ることを危惧した場合、class化してメモ化するなど様々手段は考えられるが、今回ここでpropsをObservableにする方法で解決してみたい。

## まずはObservable使わずmapPropsだけで解決する

ひとまずcounterの計算処理をComponentから剥がしてみよう。

それだけであればObservableは不要で `mapProps` で十分だ。

```jsx
    const withTextCounter = mapProps(({ text }) => {
      return {
        text,
        count: text.length
      }
    })
    const TextCounterInner = ({ text, count }) => {
      // const count = text.count
    　// カウンターはもう外からもらっているので計算不要
      return (
        <div>
          <div>
            {count}
            文字
          </div>
          <div>{text}</div>
        </div>
      )
    }
    const TextCounter = withTextCounter(TextCounterInner)
```

## 更新回数をmapPropsStreamを利用して間引く。

ここで例えば更新回数が多すぎるということでObservableで更新を間引きたい。

今回はRxJSを使う。
不幸にもRxJS@6で上手く動いてないバグを踏み抜いてしまったので、`rxjsObservableConfig`  は今回使わずにセットアップする

```js
    import { tap, auditTime, map } from "rxjs/operators"
    import { mapPropsStreamWithConfig } from "recompose"
    import { from } from "rxjs"
    
    const rxjsConfig = {
      fromESObservable: from,
      toESObservable: (stream) => stream
    }
    // configを与えてmapPropsStreamを作成
    const mapPropsStream = mapPropsStreamWithConfig(rxjsConfig)
```

これで `mapPropsStream` の準備が出来たので、今回のcomponent用のstreamを作成する。

これはredux-observableのEpicととても良く似ている記述になる。今回は `audiTime`を利用して200msに一度だけ更新されるようにする

```js
    const withTextCounter = mapPropsStream((props$) =>
      props$.pipe(
        // tap(props => console.log("before props", props)), 
        // -> before props, {text: "aaa"}
    
        auditTime(200), 
        // 200msの入力をまとめる。throttleやdebounceと似てるが最後にemitするので一番都合が良い
    
        map(({ text }) => {
          return {
            text,
            count: text.length
          }
        }),
        // textとcountに分岐する。ここは先程のmapPropsと一緒
    
        // tap(props => console.log("after props", props)),
        // -> after props, { text: "aaa", count: 3 }
      )
    )
    
    const TextCounter = withTextCounter(TextCounterInner)
```

カウンターは先程までのものと同じのが利用できるだろう
