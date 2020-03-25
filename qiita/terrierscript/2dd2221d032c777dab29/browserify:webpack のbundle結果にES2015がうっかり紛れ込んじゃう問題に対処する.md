先日やらかしたので、その問題と対処


# 問題について
- 何が起きたか？
  - browserifyでbundleしたファイル結果にES2015でのみ動く記述（`const`や`arrow function`）が混ざって動かなくなった。
	  - 結合しているファイルがParse Errorによりごっそり全部動かない事態に。
  - babelifyも組み合わせていたが、node_modules以下はtranspile対処ではないので、変換されなかった（2016/08/10追記)
- なぜ起きたか？
  - browser対応してないパッケージを利用してしまっていた
    - 依存するパッケージのソースが、ES2015で書かれていた
      - NodeJSは既にES2015で問題なく書ける
      - パッケージ提供側は全く悪くない。利用者側（＝自分）のミス
  - 同じ機能を持つブラウザ向けのパッケージがあるのを見逃していた
- なぜすぐ発見が出来なかったのか？
  - chromeなどでは、ES2015が動いてしまうので、手元の動作検証で見つけられなかった

# 防止・対処策
自分のミスではあるのだけど、とはいえ二度と起きないように防ぎたい。
じゃあどう対処するかという話

地道な動作環境みたいなのは（この問題以外の点でも）やる方が良いのだけど、そこに行くとE2Eテストの話になって行きそうなので、今回はスコープ外とする[^1]

[^1]: npm updateは細かくやりたいけど、その度に全部検証やると若干辛いし、かといってnpm updateしなくなるみたいなのもわりと悪循環になるジレンマを解消すると考えると、seleniumとか頑張っていくみたいな方向しか無いだろうなと感じている。

## 案1: acornでパースする
[acorn](https://github.com/ternjs/acorn)でパースしてみるという手がある。

acornは`eslint`などで利用されているパーサーで、`babel`の利用している[babylon](https://github.com/babel/babylon)もacornをベースにしている。

`acorn`はコマンドラインからのパースも出来るので、生成後のファイルに対して`--ecma5`モードでパースをためしてみることで、生成後のファイルがES5で読めるのか検証することが出来る。普通に実行すると当然parse結果が出てくるので、検証のために利用するなら`--silent`で出力を抑える

`postbuild`あたりにscriptとして仕掛けてみるのも良いかもしれない。どんなふうに仕掛けるかはお好みで。

```
% npm i -D acorn
```
```package.json
{
   :
  "scripts": {
    "postbuild": "acorn --ecma5 --silent /path/to/build.js"
  }
   :
}
```

コマンドラインで走らせるならこんな感じ

```
% $(npm bin)/acorn --ecma5 --silent /path/to/build.js
```

## 案2: 難読化、圧縮化（uglufy/minify) する
uglify・minifyによる最適化を既にしている場合、実はこの問題ほとんど気にしなくて良い。[^2]

[^2]: 今回自分がハマったコードは、残念ながらminifyしていなかった。

uglifyやminifyに使われる[uglifyjs](https://github.com/mishoo/UglifyJS2)などの一般的なメジャーなライブラリは、buildした生成物をparseして最適化や圧縮をしている。
当然、これらでecma2015な構文が入れば、パースエラーとなり、その時点でこの問題を防止出来る。
ちなみにuglifyjsはacronとは別な独自なパーサーを利用している様だ。

webpackだとファイル単位でuglifyjsのパースをして具体的どこでエラー出てるか解析してくれている。([このへん](https://github.com/webpack/webpack/blob/bc29455e92f4b760db745775a4fca5cde0d01d23/lib/optimize/UglifyJsPlugin.js#L44)でやってる)

```
% wepback -p
SyntaxError: Unexpected token name «i», expected punc «;» [./~/isemail/lib/index.js:130,0]
```

browserifyで[uglifyify](https://github.com/hughsk/uglifyify)を使うならこうなる。`-t`では自分の書いたコードだけしかuglifyされないので、`-g`で全体的にuglifyifyをかける。

```
% browserify -g uglifyify /path/to/file.js -o /path/to/output.js
```

難読化して困ることが無いならば、難読化と一緒にこの問題もついでに対処してしまうというのは一つの手だろう。

ただし、minifyは難読化、ファイル圧縮のためなので、この問題の対処という意味ではあくまで副次的な作用なので、そこらへんは注意して扱う必要がある。

## 案3: (ボツ）nodeのバージョンを0.12にする
- nodeのバージョンが古ければ、es6そもそも対応してないという考え。
- ナシとは言わないが、基本的にはおすすめしない。


# 予防策
根本的なところではないが、やっておくと多少この問題に当たる事をより少なくする予防策のお話

## save exactで保存しておく
ブラウザでbundleするような前提のパッケージであれば、`.npmrc`の設定で`save-exact`を有効にする。これを行うと、`^1.0.0`という形式でなく、`1.0.0`という固定バージョンでインストールされるので、依存パッケージがいつの間にか変わっていた、というような機会を減らしてくれるはず。
npmrcについては[こちら](http://qiita.com/inuscript/items/86dbfd26abe6905756c0)について詳しくまとめたので、併せて参考にしてもらえれば幸い。

```.npmrc
save-exact=true
```

コマンドで行うならば`npm install -E`になる

```
% npm -S -E some-package
```

最新版のパッケージに更新したい場合は、下記のようなコマンドを打つことで最新化出来る。

```
% npm -S -E some-package@latest
```

## shrinkwrapをする
`save-exact`の更なる部分だけだと、install元のパッケージだけで、孫パッケージなどのバージョンは固定されないので、`shrinwkrap`を併用することもおすすめする。
ただし、運用は若干面倒に感じてしまう事もあるかもしれない。

```
% npm shrinkwrap
```

## 無理して使おうとしていたら注意信号と感じ取る。
例えばwebpackで下記のような記法をしたときは、黄色信号と見て良い。

```js
module.exports = {
   :
  // `node`で標準moduleを空に出来る
  node: {
    'dns': "empty"
  }
   :
}

```
これはnodeの標準モジュールをmockやemptyとするものだが、はほぼ非公式なhackをしているに近いので、やっていたら「マズい」と感じで良い。今回の問題だけでなく、様々問題が出うることが予測される。
そのような場合は、下記について考えてみると良いだろう


- そもそもそのパッケージがブラウザ対応しているのか？
  - 類似でもっとbrowserに対応しているパッケージは無いのか？
  - issueなどでこの点は言及されていないか？
	  - もっと適合したパッケージが勧められている可能性がある

- どうやって見分けるか？
	- READMEを読む
	   - browser compabilityについてどうしているのか？
	   - そもそもbrowser向けとしてないかも？
	- `package.json`に`browser`フィールドがあるかを確認してみる
		- あれば安心度高そう
		- browserフィールドについては、下記の記事が参考になる
		    - [package.json の browser field 入門編](http://qiita.com/shinout/items/4c9854b00977883e0668)


# まとめ
npmは、[npm and front-end packaging](http://blog.npmjs.org/post/101775448305/npm-and-front-end-packaging)という表明がされていたり、frontendのパッケージ管理にも積極的ではあるのだが、npmはそもそもNodeJS向けのパッケージマネージャで、全てのパッケージがbrowser対応してないるわけではないのは当然の事。

npmの更なる改善に期待しつつ、うまく付き合っていきたい。
