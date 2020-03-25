
* 2016/09/15 現在の情報。
  * 出たばっかりなので今後変わる可能性大。
* 今まで個人的にはreact上のroutingはあんま使って来てなかったので、あんまりちゃんとした比較はできてない
  * [死んだと言われたり復活したと言われたり](https://medium.com/rackt-and-roll/rrtr-is-dead-long-live-react-router-ce982f6f1c10)、[必要ないよ！](https://medium.freecodecamp.com/you-might-not-need-react-router-38673620f3d)と言われたりと不安定感すごかったので見送ってた。
  * あんまり好きじゃなかったし、そこ飲んでまで使うほどの要件も無かった
* [v4のドキュメント](https://react-router-website-xvufzcovng.now.sh/)のページ見たら可能性感じたのでとりあえず眺め回してみることにした。
  * [twitter上からも歓喜の声が聞こえてくる](https://twitter.com/search?src=typd&q=react-router%20lang%3Aen)
  * ドキュメントのサンプルは素晴らしいのだが、APIやQuickStartは、何故かスクロールが変な感じなので、githubを直接見るほうが良いかもしれない
    * https://github.com/ReactTraining/react-router/tree/v4/website
* 現行のstableは2、3はalphaを出してしまったけどそこからbreaking changeしたらしくversionが4となったっぽい。


# ざっくり

* 基本今までのと全部違うと言っていいぐらい違う
* めっちゃコード量減ってる
* 旧バージョンからアップデートするとなると結構書き換えは必要そうかも

# installして試したかったら？

```
$ npm install react-router@next
```

# package.json から見る構造変更
* v2（現行）
  * [history](https://github.com/mjackson/history)に依存しており、`<Router />`には`history`オブジェクトを渡す必要があった。
  * 内部でhistoryとの結合とかを色々やってた。
* v4
  * history APIにがっぷりな部分は[react-history](https://github.com/ReactTraining/react-history/tree/master/modules)に移植された感じ
    * メンテナやOrganizationはだいたい一緒っぽい。
  * ルーティング解決は[path-to-regexp](https://github.com/pillarjs/path-to-regexp)
  * 構成的には、[You might not need React Router](https://medium.freecodecamp.com/you-might-not-need-react-router-38673620f3d#.uuq4y04fb)で紹介されているものとかなり近い。

# Quick start比較

それぞれざっくり特徴が見えるところを抜粋

### v2

```js
import React from 'react'
import { render } from 'react-dom'
import { Router, Route, Link, browserHistory } from 'react-router'

render((
  <Router history={browserHistory}>
    <Route path="/" component={App}>
      <Route path="about" component={About}/>
      <Route path="users" component={Users}>
        <Route path="/user/:userId" component={User}/>
      </Route>
      <Route path="*" component={NoMatch}/>
    </Route>
  </Router>
), document.getElementById('root'))
```

### v4

[quick-start (v4)](https://github.com/ReactTraining/react-router/blob/v4/website/pages/quick-start.md) からざっくり抜粋

```js
import React from 'react'
import { render } from 'react-dom'

import { BrowserRouter, Match, Miss, Link } from 'react-router'

const App = () => (
  <BrowserRouter>
    <div>
      <ul>
        <li><Link to="/">Home</Link></li>
        <li><Link to="/about">About</Link></li>
        <li><Link to="/topics">Topics</Link></li>
      </ul>
      <Match exactly pattern="/" component={Home} />
      <Match pattern="/about" component={About} />
      <Match pattern="/topics" component={Topics} />
      <Miss component={NoMatch}/>
    </div>
  </BrowserRouter>
)

render(<App/>, document.querySelector('#root'))
```

`history`を渡さず、Componetとして定義出来るようになったことかなりのすっきり感がある。
`<Match>`のルールも子コンポーネントで更に定義するなどもあって、親`<Router />`が全体のRoutingを把握しなくても良くなっているっぽい

また、`<div>`などの標準的な記法と`<Match/>`の組み合わせで作られているので、HeaderやFooterやMenuなどの組み合わせがやりやすい。

https://react-router-website-xvufzcovng.now.sh/basic


# ファイル数

```
### v2
% git clone https://github.com/ReactTraining/react-router.git
% ls -1 modules/ | grep .js | wc -l
      38

### v4
% git co v4
% ls -1 modules/ | grep .js | wc -l
      16
```
38 -> 16 と大激減！
多けりゃいい少なけりゃいいというものではないけど、内容としても見通しがよく感じる。
かなり読みやすいと感じた。

historyとのつなぎ込み部分が`react-history`に移動されたことで減った部分が多い模様。


# ロゴ
[<Font color="cyan">青かった</font>](https://github.com/ReactTraining/react-router/blob/42319709c3df1f660d67e6aac03c38d193f6d1ec/logo/vertical.png)のが[<Font color="red">赤くなった</font>](https://github.com/ReactTraining/react-router/blob/ecebbf09bc062214140e9b857f364d12f9253783/logo/Vertical.png)

# ハマりどころ：`<Router/>`の直下は単一のコンポーネントである必要がある

ちょっと使って見つけた部分。

```
MatchProvider.render(): A valid React element (or null) must be returned. 
You may have returned undefined, an array or some other invalid object.
```
こんなエラーが出てきてなんだろうなーと思ったら`<Router />`直下のComponentは一つのコンポーネントでないといけない。なので`<div></div>`とかで囲まないといけない。

これは仕様らしい。
https://github.com/ReactTraining/react-router/issues/3837#issuecomment-246919642

```js
// ダメ
const BadRoute = () => (
  <Router>
    <Match pattern="/" component={Foo} />
    <Match pattern="/:bar" component={Baz} />
  </Router>
)
```
```js
// ダメ
const BadRoute = () => (
  <Router>
    <div>
      <Match pattern="/" component={Foo} />
      <Match pattern="/:bar" component={Baz} />
    </div>
    <div></div>
  </Router>
)
```
```js
// これならOK
const Route = () => (
  <Router>
    <div>
      <Match pattern="/" component={Foo} />
      <Match pattern="/:bar" component={Baz} />
    </div>
  </Router>
)
```

# FAQより「なんで巨大な変更をするのか？」
2016/09/16追記項目

[FAQ](https://github.com/ReactTraining/react-router/tree/v4#v4-faq) がREADMEに追加された。最初の項目に関して結構思想的な背景が見える部分だったので、雑訳してみる。
2016/09/16追記：[FAQ](https://github.com/ReactTraining/react-router/tree/v4#v4-faq) がREADMEに追加された。最初の項目に関して結構思想的な背景が見える部分だったので雑訳。

* Q. [なんで巨大な変更するのか？](https://github.com/ReactTraining/react-router/tree/v4#why-the-huge-change-again)
* A.  tl; dr: Declarative Composability（宣言的構築性？）のため
  * 「我々はReact Routerでわくわくしたことなかった」
  * 「私達が学んだことは、Reactの最も愛すべき理由が **Declarative Composabillity**であることです」
      * 要するにReactと同じように、宣言的に記述出来るのが良いよね。という感じっぽい。
  * 「staticなrouterのconfigをしないといけないせいで、下記のようなルーターは書けなかったし、Routeをラップすることもできなかった」
	  * 「`const CoolRoute = (props) => <Route {...props} cool={true}/>`」
	  * 宣言的な記述でない例として、静的なrouting configを書かないといけないことが弊害として上げられているものと思われる。
  * 「routeコンポーネントのレンダリングに関与するために、`<Router createElement render>`とか`createRouterMiddleware`快適と言えないAPIを提供していた。我々は`createElement`を取り除き、あなたたちに返す」
     * うまく訳せないが、きっとAPIとしては提供しなくてもdeclarativeにすることで、「middlewareらへんはあなた達の好きな様にやってくれたらいい」という意味っぽく感じる。
  * 「`onEnter`、`onLeave`、`onChange`のライフサイクルフックを再作成していた。けどReactには既に`componentWillMount`、`componentWillReceiveProps`、`componentWillUnmunt`がある」
  * 「Route configはviewヒエラルキーを記述していた。しかし、そもそもReact Componentは既にviewヒエラルキーを記述していた。」
  * 「今までのReact Routerは『Reactのルーター』ではなく、『Reactのルーティングフレームワーク』だった」[^1]
      * 「Reactにとって余計なAPIだけでなく、ビルドエコシステムに対しても驚くほどやっかいな偶発的なフレームワークとなっていた」
  * 「あなたはcomponentをレンダリングしてpropsを渡すことでroutingをコントロール出来る。」


[^1]: 個人的にこの表現が「おー」ってなった。









