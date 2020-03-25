
# tl;dr 

- [ESLint](http://eslint.org/)っぽいCSS用のLint
- [PostCSS](https://github.com/postcss/postcss)プラグインとして動作するので色々嬉しい
- SCSSもだいたい使える


# stylelint
ESLintのようなconfigでPostCSSプラグインとして動く[stylelint](http://stylelint.io/)。
唐揚げチャーハンのような「美味いものと美味いもの組み合わせたら絶対おいしいじゃん！」みたいなプロダクトが実際とても良かったので色々利点を記載。

導入についての手順などはすでに記事があったので、そちらへリンクしたい

- [CSSのLintをstylelintにする
](http://qiita.com/makotot/items/c266ed11ada1423cb96e)

## ESLintっぽくて良い所
javascriptのLinterとしてスタンダードになってきたESLintに非常に良く似て作られているので、一度そちらを導入していればstylelintの導入はおそらくすんなり出来る。

また、ESLintの持っていた良い所をそのまま持ってきたような良さがある。

### config周りがESLintにすごく近いので覚えることが少なめ

肝となる[stylelintのconfig](http://stylelint.io/user-guide/configuration/)は、だいたいこんな感じで、ESLintにかなり近く作られているので、先にESLintを導入していれば覚えることは少ないはず。

```.stylelintrc
{
  "plugins": [
    "../some-rule-set.js"
  ],
  "rules": {
    "block-no-empty": null,
    "color-no-invalid-hex": true,
  },
}
```


config関連は、下記のような部分もESLintと同様に扱える

- 設定は`package.json`,`.stylelintrc`でも設定可能。`stylelint.config.js`とすればJavaScript Objectとして設定を書ける
- 無視対象のファイルを`.stylelintignore`などで設定できる
- [plugin](http://stylelint.io/user-guide/plugins/), [extend](http://stylelint.io/user-guide/configuration/#extends)の機能がある（後述）

### デフォルトでルールが設定されてないので既存プロジェクトに導入しやすい

同様のツールである [CSSLint](https://github.com/CSSLint/csslint/) など、デフォルトでルールがたくさん設定されているLinterの場合、既存プロジェクトに導入するのは逐一ルールをdisableにするか、大量に吐き出されるであろうlintを潰さないとまともに使えない状況で困難だった（そしてだいたい途中で辛くなって挫折する）

stylelintはESLint同様、[デフォルトでは何のルールも設定されてない](http://stylelint.io/user-guide/configuration/#rules)。
この事に良し悪しはあれど、既存プロジェクトへの導入では上記のような苦悩が無く、無理なく適用可能なルールから導入していける。

### extendsの機能があるので、新規プロジェクト向けには[stylelint-config-standard](https://github.com/stylelint/stylelint-config-standard)が使える

ルールが何もないのは新規プロジェクトでは全く無意味。
「推奨ルールがあってくれないと何を入れたらいいかわからない」という人にも安心。ESLintでは`eslint:recommended`があるように、[stylelint-config-standard](https://github.com/stylelint/stylelint-config-standard)が用意されている。

```
$ npm install stylelint stylelint-config-standard -D
```

```.stylelintrc
{
  "extends": "stylelint-config-standard"
}
```
これでオススメルールが追加された状態になる。

もしもっと他のextendsを探したければ[stylelint-config](https://www.npmjs.com/browse/keyword/stylelint-config)のタグから探せる。


## PostCSSを利用している恩恵の良い所

stylelintはパーサーとしてPostCSSを利用しているので、その恩恵にあやかれる。

### gulpなどでPostCSSを利用しているなら追加するだけで良い

[autoprefixer](https://github.com/postcss/autoprefixer)など、PostCSSを既に導入しているプロジェクトも少なく無いだろう。そんなプロダクトにとってstylelintを導入するのはとても簡単。
既存のpostcssのpluginに`stylelint`と`postcssReporter`を追加するだけで良い

```Gulpfile.js
  :
var stylelint = require('stylelint')
var postcssReporter = require('postcss-reporter')
  :
gulp.task('postcss',  function () {
  gulp.src("path/to/css")
    .pipe(postcss([
      autoprefixer(autoprefixerOpts), 
      postcssOpacity,
      stylelint(), // これを追加
      postcssReporter({clearMessages: true}), // こっちも必要
    ]))
    .pipe(gulp.dest('./web/css'));
});

```

[gruntのexample](http://stylelint.io/user-guide/postcss-plugin/#example-a)や[postcssを直接扱うexample](http://stylelint.io/user-guide/postcss-plugin/#example-c)がほしければ公式にある。

当然、[Command Line](http://stylelint.io/user-guide/cli/)から直接叩くことも出来る。なので、gulpやらを使いたくない人はこちらを使うと良いだろう。

### Scss、SugerSSなどにも対応
PostCSSがCSS-likeな言語をサポートしているScss, [SugerSS](https://github.com/postcss/sugarss)などを、stylelintでもそのまま使える（LessはExperimentalの模様）

gulpで使いたければこんな感じになる

```Gulpfile.js
  :
var postcssScss = require('postcss-scss')
var stylelint = require('stylelint')
  :
gulp.task('scss-lint',  function () {
  gulp.src("path/to/scss")
    .pipe(postcss([
      stylelint(), // これを追加
      postcssReporter({clearMessages: true}), // こっちも必要
    ], {syntax: postcssScss}))
    .pipe(gulp.dest('./web/css'));
});
```

ただし、残念ながら[no-browser-hacks](http://stylelint.io/user-guide/rules/no-browser-hacks/)など、一部のルールはうまく扱えない[^1]

[^1]: 軽くコードを追ってみたが、どうもこれはPostCSSの仕様によるものらしい。

### PostCSSと同じAPIを利用してrulesが作れる
恩恵としては地味だが、独自Ruleを作る際に[PostCSSのAPI](https://github.com/postcss/postcss/blob/master/docs/api.md#root-node)を扱えるのは利点がある。

PostCSSのpluginを書いたことがある人ならば（そんな人がどの程度いるのかは不明）すぐ書けるし、ルールをpluginに移植したくなったり、既存のpluginをルール化したい時もそんなに難しくない[^2]。

[^2]: 実際、no-browser-hacksなどは、[stylehacks](https://github.com/ben-eb/stylehacks)という別なPostCSSプラグインを使って作られている。

例えばカスタムルールはこんな具合で書ける

```custom-plugin.js
// ruleはpluginの一部として記載する
module.exports = stylelint.createPlugin(ruleName, function(expectationKeyword, optionsObject) {
  return function(root, result) {
     // PostCSSのRoot Nodeが取得されるので、これを利用してruleを書いていく
  }
})
```


## その他良い所
### ルール違反時のメッセージをカスタマイズ出来る

多分ESLintなどには無い（と思う）が、表示するメッセージをカスタマイズすることが出来る。これはデザイナなど非エンジニアにもフレンドリーだろう。

```stylelint.config.js
{
  "color-hex-case": [ "lower", {
    "message": "色はhexで統一しましょう"
  } ]
}
```

### Atom周り充実
[linter-stylelint](https://github.com/AtomLinter/linter-stylelint)や`.stylelintrc`のアイコンなど、Atomプラグインが一通り対応されている。

### [stylefmt](https://github.com/morishitter/stylefmt)と組み合わせて自動修正

`eslint --fix`的な自動修正として、stylefmtというプラグインとの組み合わせが言及されている。実はまだここまで手をつけれていないが、組み合わせればかなり便利そう。

