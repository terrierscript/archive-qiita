
# 大本命。ESLint

2015年現在、JavaScriptのLinting toolといえばJSHintかJSLintみたいな風潮ありますが、もうESLintで行きましょう。

- [公式ページ](http://eslint.org/)
- [github](https://github.com/eslint/eslint)

### 大きな特徴

- プラガブルな実装
	- 全てのルールのON/OFFが可能
	- 独自のルールの追加が可能
- 独自のフォーマッターでの出力が可能
- [ECMAScript 6 / React JSX](http://eslint.org/blog/2014/11/es6-jsx-support/)をサポート

## [Philosophy](http://eslint.org/docs/about/#philosophy)
ESLintは下記のPhilosophyを掲げています。

#### 全てはPluggableである。
* Rule APIはバンドルされたものもカスタムもどっちも使える
* Formatterはバンドルされたものもカスタムもどっちも使える
* 追加のルールとフォーマッターは実行時に指定できる
* バンドルされたルールとフォーマットを使わなくても良い

#### 全てのルールは
* 独立している
* 全てのルールはoffにもonにもできる（どのルールも「オフにするには重要すぎる」とはみなさない）
* 個別にwarningかerrorに指定することができる
* non-zeroな値を与えてonにできてzeroを与えてオフにできる

#### 加えて
* ルールは「アジェンダフリー」 - ESLintはどのようなコーディングスタイルをも促進させるものではない
* バンドルされるルールは一般化できるもの

#### プロジェクトは
* 豊富でクリーンにドキュメンテーションする
* できるだけ透過に
* テストを重んじる。

## 使い方とか
ESLintの基本的な使い方についてはすでに先行の記事もあるのでそちらにお任せ。
http://qiita.com/makotot/items/822f592ff8470408be18

個人的にはファイルごとに書くのはルールが散って気持ちがよくないので`.eslintrc`にまとめて書く方をおすすめしたいところ。

### grunt / gulp
grunt: https://www.npmjs.com/package/grunt-eslint
gulp: https://www.npmjs.com/package/gulp-eslint
未調査だけどそれなりに利用されているっぽい。

## ルールの種類
bundleなものでも[結構な量が用意されている](http://eslint.org/docs/rules/)ので一つ一つ説明をこの記事ではしきれないが、だいたい名前だけで理解できるものが多い。
（全てデフォルトでonというわけではない。後述）

カテゴリとしては

* エラーになりうるもの
* ベストプラクティス
* 変数について
* Node.js関連
* コーディングスタイル関連
* ECMAScript 6に関連
* JSHint / JSLint互換を保つためのLegacy Rule

というふうになされている。

それぞれしっかり名前がついているので、どのルールなのかがJSHintに比べてわかりやすいと思う。

```eshint.js
/* eslint radix:1 */
```

```jshint.js
/* jshint -W065 */
```

### デフォルトでonになっているルール
**「Pluggableである」** とはいえそれはデフォルトで **「何も設定されていない」というわけではない**。ESLintのPhilosopyに従い精査されたルールがデフォルトで適応されている。このあたりはrubyの[rubocop](https://github.com/bbatsov/rubocop)などがイメージとして近いかもしれない。

offになっているものに関しては`(off by default)`という記述がなされているので、どんなルールがどんな理由でoffになっているか知りたい方は各ルールのドキュメントを確認されたい。

そうでない方はデフォルトの状態で使うで問題ないだろう。
エラーになりうるものなどは多くがデフォルトONになっているので、そのままの状態で使ってみるというスタンスでそれほど問題ないはずだ。

JavaScriptにおいては、プロジェクトチームの利用したいスタイルだけではなく、対応ブラウザなどでも適応すべきルールが変わって来るので、そのプロダクトにあわせて適応するのがよいだろう。

## [JSLint / JSHintとの比較](https://github.com/eslint/eslint#eslint)
公式リポジトリで、このように説明がされている。

- ESLintは内部のパーサーとして[Espree](https://github.com/eslint/espree)という[Esprima](https://github.com/jquery/esprima)のforkが使われている
	- forkの理由は[こちら](http://eslint.org/blog/2014/12/espree-esprima/)原文がある。[こちらのスライド](http://azu.github.io/slide/crosushi/shift-ast.html)にこの理由についてが要約されているが、ざっくり言えばES6/JSXのサポートをESLintに含めたいのと、Esprimaの動きが遅かった事が原因のようだ。
- ESLintはAST（抽象構文木）評価を使っている
	- なのでプラガブルにルールが書きやすいのだろう。
- plugableである（前述通り）
	- 元々このあたりがJSHintでは実装されなそうという理由でプロジェクトがスタートしたようだ。
	- https://github.com/eslint/eslint#why-dont-you-like-jshint


## [速度が弱点](https://github.com/eslint/eslint#how-does-eslint-performance-compare-to-jshint)
ESLintはどうしてもASTを使う都合上、JSHintより1ファイルあたり2-3倍時間がかかってしまうとのことだ。

とはいえ筆者がそれなりの規模のプロジェクトにESLintを試しにかけた感じでは、十分に実用に耐えうるレベルの速度感ではあった。Lintをとにかく早くさせたい！という目的にはそぐわないようだ。

## ES6サポート
version 0.12.0から（現在は0.14.1)一つ一つ対応の実装をしている。

# 終わりに
個人的な思い出で、JSHintは昔[Missing radix prameter](https://github.com/jshint/jshint/issues/628)をdisableに出来ず、更にそのIssueが解決してないタイミングでクローズされたというところで見限って「これからは俺が俺を信じていくんだ！」という謎決意して数年が経ちました。
これからしばらくはESLintで良いJavaScript生活を送れそうです。

あと哲学って大事だなと思いました。
