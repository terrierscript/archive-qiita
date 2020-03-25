<blockquote class="twitter-tweet" data-lang="ja"><p lang="en" dir="ltr">🎉🎉 React Router v4 beta! 🎉🎉<br><br>Docs: <a href="https://t.co/xEsQD8H38v">https://t.co/xEsQD8H38v</a><br>Code: <a href="https://t.co/VULptzNaQs">https://t.co/VULptzNaQs</a><br>Try it: npm i react-router@next<br><br>We 😍 <a href="https://twitter.com/reactnative">@reactnative</a>!</p>&mdash; MICHAEL JACKSON (@mjackson) <a href="https://twitter.com/mjackson/status/826114904914489344">2017年1月30日</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

ということでreact-router@v4が[alpha](https://github.com/ReactTraining/react-router/tree/v4.0.0-alpha.6)から[beta](https://github.com/ReactTraining/react-router/tree/v4.0.0-beta.1/)になった。
かなりアグレッシブに変更されていて色々変える所あったのでメモ [^1] [^2]

[^1]: 「unstableだし覚悟はしてたけど、やってくれたな！」という気持ちありつつ、「乗り換えたいライブラリもないし、自前しても同じような感じになりそうだし、幸いv4-alphaのコードはある程度把握してるし、まあ気軽に心中してみるか」という気持ちで臨んだ。
[^2]: 自分が見つけられた部分だけです。他にもあったらコメント欄 or 編集リクエストでもいただければ幸いです。

# packageの分離
`react-router`は下記の3パッケージに分離した

* `react-router`
    * コア部分。
    * `react-router`の利用者がこのパッケージをインストールする機会は無いかもしれない。
    * このパッケージに含まれるもの
        * [`<Router>`](https://reacttraining.com/react-router/#router) (多分直で使う事はないローレベルなクラス）
            * [`<MemoryRouter>`](https://reacttraining.com/react-router/#memoryrouter)
            * [`<StaticRouter>`](https://reacttraining.com/react-router/#staticrouter)
        * [`<Route>`](https://reacttraining.com/react-router/#route) (旧Match ：後述）
        * [`<Switch>`](https://reacttraining.com/react-router/#switch) (新機能：後述）
        * Route(旧Match)
        * [`<Redirect>`](https://reacttraining.com/react-router/#redirect)
        * [`<Prompt>`](https://reacttraining.com/react-router/#prompt) (旧NavigationPrompt：後述)
        * [withRouter](https://reacttraining.com/react-router/#withrouter) 
        * [match(matchPath)](https://reacttraining.com/react-router/#match)

* `react-router-dom`
    * ブラウザ用
    * 今までのreact-rouerの機能がこっちにごそっと移動された感じ
    * 現行、`react-router`の機能をre-importして提供されているものもあるが、stable時にどうなるかは不明。
    * このパッケージに含まれるもの
        * [`<BrowserRouter>`](https://reacttraining.com/react-router/#browserrouter)
        * [`<HashRouter>`](https://reacttraining.com/react-router/#hashrouter)
        * [`<Link>`](https://reacttraining.com/react-router/#link)
        * `<NavLink>` (2017/02/01時点ではまだUndocumented,  まだ不安定気味。後述）
        * `react-router`を一通り再importしたもの

* `react-router-native`
    * ネイティブ用
    * あんまり自分の要件で使うことがなさそうなのでスルー気味。
    * Android用とiOS用が分岐してるっぽく見える。
    * [`<NativeRouter>`](https://reacttraining.com/react-router/#nativerouter)というのがRouterになるもよう

# documentationの変更

URLが新しくなった。

https://reacttraining.com/react-router/

alpha版のドキュメントリダイレクトされてしまうため、[github漁る](https://github.com/ReactTraining/react-router/tree/v4.0.0-alpha.6/website/api)ぐらいでしか出てこなそう

# installationsの変更

パッケージが分離したので、`react-router`ではなく`react-router-dom`をinstallするようになる。

```js
"react-router": "4.0.0-alpha.6",
```

↓

```js
"react-router-dom": "4.0.0-beta.3",
```

[Installation](https://reacttraining.com/react-router/#installation)には「`react-router-dom@next`でインストールしてね！」というのがあるが、この方法でインストールするとバージョンがダイナミックに変わるので、個人的にはバージョンベタ決めでいいんじゃないかと思っている。
（以前から`react-router@next`でinstallしてたら今回の変更で無事死亡していただろう）[^3]

[^3]: 本音を言えばreact-router@betaとかreact-router@v5.0.0-betaとかでpublishしてほしかった気持ちが無くもない

# コードベースの変更

## `<Match>`は廃止。`<Route>`になった（戻った）

```jsx
// 旧(alpah)
<Match pattern="/user/:id" />

// 新(beta)
<Route path="/user/:id" />
```

[このコミット](https://github.com/ReactTraining/react-router/commit/c2d40ff641df3698ab8b144b475fd7c84fae7144)で変わった。
明確な理由は見つけられなかったが、現行バージョンに近い形に戻された。

[v2/v3からの移植も楽になるっぽい](https://gist.github.com/mjackson/33f51e9d6b9ea18b467c613fbbcf50e1)。

* [`<Match>`の旧ドキュメント](https://github.com/ReactTraining/react-router/blob/v4.0.0-alpha.6/website/api/Match.md)
* [`<Route>`のドキュメント](https://reacttraining.com/react-router/#route)


## `<Miss>`が廃止。`<Switch>`で代用

下記のサンプルがわかりやすい。
https://reacttraining.com/react-router/examples/no-match

```jsx
// 旧(alpha)
<div>
  <Match pattern="/foo" component={ Foo } />
  <Miss component={ NotFound } />
</div>

// 新(beta)
<Switch>
  <Route path="/foo" component={ Foo } />
  <Route component={ NotFound } />
</Switch>
```

* [`<Miss>`の旧ドキュメント](https://github.com/ReactTraining/react-router/blob/v4.0.0-alpha.6/website/api/Miss.md)
* [`<Switch>`のドキュメント](https://reacttraining.com/react-router/#switch)


`<Switch>`は、若干理解しづらいが、「Routeが *包括的(inclusively)* にレンダリングするのに対し、Switchは *排他的(exclusively)* にレンダリングするという機能らしい。

このコンポーネントが出来た事で、「どれにも当てはまらなかった場合」という場合に`path`が無いrouteを出して排他的に処理することでNotFoundな場合のコンポーネントとして扱える。このため`Miss`は不要になった、ということらしい。


## `<Link>`から`active`系のが消えた。`<NavLink>`になる予定


これまで`<Link>`は現在のURLと同じURLの場合に`activeClassName`のstyleがあたる、というようなものを持っていたりしたが、新しい`<Link>`はこのあたりのstyle管理を持たなくなった。
代替として`<NavLink>`というコンポーネントがこの代替をする予定のようだが、今のところundocumentedで若干不安定。

今のところで言うと`<Route>`を作ってサクッと自前でやることも可能。
こちらのサンプルに乗っている
https://reacttraining.com/react-router/examples/custom-link

```jsx
// 旧(alpha)
<Link to="/about" activeClassName="active">
  About
</Link>

// 新(beta)(代替バージョン)
const SomeLink = ({ label, to, activeOnlyWhenExact }) => (
  <Route path={to} exact={activeOnlyWhenExact} children={({ match }) => (
    // children要素でfuncを渡すことで、この<Route>がmatchしたかどうかなど判定出来る。
    <div className={match ? 'active' : ''}>
      {match ? '> ' : ''}<Link to={to}>{label}</Link>
    </div>
  )}/>
)
```

* [Linkの旧ドキュメント](https://github.com/ReactTraining/react-router/blob/v4.0.0-alpha.6/website/api/Link.md)

## `<Match(Route)>`のproperty変更: `exactly` -> `exact`

`Match`(現Route)のexactlyというproertyがexactになった。

```jsx
// 旧(alpha)
<Match exactly ... />

// 旧(bet)
<Route exact ... />
```

https://github.com/ReactTraining/react-router/issues/4237

exactlyってよりexactの方が正しいのでは？ってことっぽい（あんまりここらへんのニュアンスわかってない）

## `<NavigationPrompt>` -> `<Prompt>`

個人的にはまだ使ってなかった機能だったのでハマってなかったが、ドキュメント眺めてたら変更されているようだった。おそらくreact-router-nativeを見越しての命名変更だろう。
ドキュメント見比べても中身はそんなに変わってなさそう

* [`<NavigationPrompt>`の旧ドキュメント](https://github.com/ReactTraining/react-router/blob/v4.0.0-alpha.6/website/api/NavigationPrompt.md)
* [`<Prompt>`のドキュメント](https://reacttraining.com/react-router/#prompt)

# 新機能など

## `<Route strict>`が追加された

https://reacttraining.com/react-router/#route.strict

個人的にすごい嬉しい機能。
今まで最後の/(スラッシュ）が無視されなかったが、デフォルトでは無視されるようになった。
`Match`と同様スラッシュを区別したい場合にこのプロパティを渡すと良い

* `<Route path="/foo/bar">` -> /foo/bar/でマッチ
* `<Route path="/foo/bar" strict>` -> /foo/bar/でマッチしない(旧Matchの仕様と同じ)

## `withRouter`が追加された

いわゆるRouterのHigherOrderComponent.
何か複雑なことをやる場合に必要になりそうだが、基本そんなに使わなそう

* [`withRouter`のドキュメント](https://reacttraining.com/react-router/#withrouter)

