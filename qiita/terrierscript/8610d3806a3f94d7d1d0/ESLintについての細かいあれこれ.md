
勢いで[前の記事](http://qiita.com/suisho/items/dcf48f56d8f484c0a1a8)書いてみたけどちゃんと触るとちょいちょいハマりどころもあったのでメモ程度に細かいことを。

# Editorの設定すると捗る。
このへん見ると良い
http://eslint.org/docs/integrations/

僕はatom推しなのでこいつらでさくっと入れれた。ちゃんと.eslintrc読んでくれるし今のところいい感じ。
https://atom.io/packages/linter-eslint
https://atom.io/packages/linter

# .eslintignoreでlintしたくないものを設定
lintしないファイルの指定。gitignoreとかと一緒の形式
このあたりが鉄板か。testを含めるかどうかは趣味によるところかも
bowerとか使わずDLしたファイルをどこから持ってきているような場合だとそいつらも含めるべきだろう。

```
node_modules/
test/
```

# .eslintrcについて
例えばこんな具合で設定する。

```
{
  "env": {
    "node" : true,
    "browser" : true
  },
  "ecmaFeatures" : {
    "jsx": true,
    "objectLiteralShorthandMethods" : true
  },
  "plugins": [
    "react"
  ],
  "rules": {
    "react/display-name": 1,
    "semi" : [2 , "never"],
    "strict" : false,
    "key-spacing" : [2, {
      "beforeColon" : true,
      "afterColon" : true,
    }],
    "no-unused-vars": [1, {"vars" : "all", "args" : "after-used"} ],
    "no-comma-dangle" : false
  }
}
```

## ES6 && JSXを使うのに　`ecmaFeatures`を指定する
<del>
[このへんの設定](http://eslint.org/docs/configuring/#specifying-language-options)が必要。

とりあえず僕はこんなもんで使っている。
一括設定とかあればいいのに。
</del>

**2015/05/11追記：**
いつの間にかenvでes6と指定すると一気にecmafeatureを見てくれるようになりました。
jsxだけはes6ではないので当然今までどおりecmaFeaturesとして必要です。

```json
"env" : {
  "es6": true
},
"ecmaFeatures": {
  "jsx": true
}
```

## rules
ここが肝になる。
僕が設定したのはこんな感じ。

```.json
  "rules": {
    "strict" : false,       // strictはbabelとかに任せる
    "key-spacing" : [2, {   // セミコロンの前後どっちにも空白つける（個人的な癖）
      "beforeColon" : true,
      "afterColon" : true,
    }],
    "semi" : [2 , "never"], // 趣味コードはアンチセミコロン派
    "no-unused-vars": [1, {"vars" : "all", "args" : "after-used"} ], // 未使用変数については警告レベルに
    "no-comma-dangle" : false // [1,2,]みたいなのへの警告はbrowserifyを通すので除外
  }
```




## [plugin](http://eslint.org/docs/configuring/#configuring-plugins)
[npmsearch.comでkeywords:eslintplugin](http://npmsearch.com/?q=keywords:eslintplugin)を検索するとわりとあるみたい。

試しに[eslint-plugin-react](https://github.com/yannickcr/eslint-plugin-react)を使ってみる


globalなeslintを使う場合であれば

```
$ npm install eslint -g
$ npm install eslint-plugin-react -g
```
という感じ。

ここで僕もはまったので注意なのが、eslintをglobalにインストールして使う場合、pluginもまたglobalにインストールしないと使えない。

ローカルに扱う場合は

```
$ npm install eslint -D
$ npm install eslint-plugin-react -D
```

とする。ローカルにeslintが入っているとglobalなeslintコマンドでもローカルのを方を見るようになるらしい。

あとは使いたいルールをrulesのセクションに入れる

```.eslintrc
  "plugins": [
    "react"
  ],
  "rules": {
       :
    "react/display-name": 1,
       :
  }
```

# envの指定
設定を変えたりグローバルを幾つか設定してくれたりする（例えばjQueryなら$をグローバルな変数として扱ってくれるとか）

ソースの[environment.js](https://github.com/eslint/eslint/blob/master/conf/environments.js)を見ればだいたい何やっているかわかる。どんな値をglobalにしてるかを知りたければ[globals](https://github.com/sindresorhus/globals/blob/master/globals.json)を見るのが良さそう。

browserifyで変換する前提なので今回はこんな具合。
IE8などのレガシーなブラウザな場合だとnode environmentで`no-console`など幾つか対策が必要なものが落とされるので注意したほうが良い。

```
"env": {
  "node" : true,
  "browser" : true
}
  
```

# 実行コマンドをpackage.jsonにまとめておく

このあたりは趣味。

```
$ npm run lint
```

```package.json
"scripts": {
  "lint": "eslint js/**/*.jsx"
}
```

npm-script の実行時には自動的にプロジェクトディレクトリの [./node_modules/.bin/ にPATHが通ります](https://docs.npmjs.com/misc/scripts#path）
`npm i -D eslint`をした上で上記のように指定しておけばローカルの`./node_modules`を見に行ってくれます（@shinnnさん 編集リクエストありがとうございました！）
