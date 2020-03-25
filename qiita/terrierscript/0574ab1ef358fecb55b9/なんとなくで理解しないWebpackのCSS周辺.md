[CSS in JSに夢を見た](http://qiita.com/inuscript/items/0cfdb1a0c1d172e27694)が、なかなか一筋縄では行かなかったので[^1]、webpackにおけるCSSと本気で向き合ってみた。

[^1]: 既存のコードにはバッチリだったが、新規開発にはちょっと向いてない部分が多かった。

しかしまだ理解が甘いところがあったのでloader, pluginまわりの関係性を整理した。

# （前置き）webpackの基礎情報

css関連の本題にはいる前に、webpackの基礎を再確認する。

## Webpackの特徴

webpackの特徴的な事項として、CSSや画像など、javascriptでないデータも基本的に全てをjavascriptで扱ってしまう、という事が挙げられる。

同等の対抗として挙げられるbrowserifyやrollupは、あくまでも「javascriptのmodule解決」にフォーカスしているのに対して、webpackは全く違う方向を向いている

## loaderとpluginの違い
結構あやふやに扱っていたが、上記のwebpackの基本部分を明確にして考えると考えやすい。

* loader
    * 「どんなものでも最終的な出力をjavascriptで実行できる形に変換する」もの。
    * 受け付けるinputは、画像やらCSSやらなんでも良い
    * outputは、必ずjavascript。
    * 例
        * css-loader -> CSSを、javascriptのコードとして扱わせる
        * url-loader -> 画像データなどを、base64エンコードして、javascriptで扱えるようにする
* plugin 
    * 「何らかの別な出力や、作用をもたらすもの」
    * inputもoutputも制限は無いので、ある意味「loader以外の色々」と言っても語弊は無さそう
    * 例
        * javascriptファイル以外の書き出し
        * コンソール出力の加工

## CSSをwebpackで扱うことの利点と欠点

下記が挙げられる。特に欠点については、注意深く取り扱うべきものがある

* 利点
    * ビルドツールの単一化。javascriptのビルドとcssのビルドを一元化しやすい。
    * 既存の資産を活かしやすい
    * CSS modulesなど、style情報とjavascriptの結びつきを強めた状態を無理なく作りやすい
        * モジュール密度を高めた開発が可能になる
* 欠点
    * webpackでしか解決できない機能が多く、依存度が高くなる。
        * 脱webpackなどの時代が来た時に負債化しやすい
        * browserify向けのcss-modulesifyというのもあるが、highly experimentalな上に、browserifyとしてもイレギュラーな扱いになる。
    * webpack以外のエコシステムでエラーになる
        * テストツールなどでよく問題になる。
            * mochaなどでは、[compilerを工夫する](http://stackoverflow.com/questions/32683440/handle-webpack-css-imports-when-testing-with-mocha)など、そこそこ頑張らないと解決出来ない事がある。
        * 比較的モダンなテストツールの場合、標準的に対応オプションを明示してくれている場合もある
            * jestでは[moduleNameMapper](http://facebook.github.io/jest/docs/configuration.html#modulenamemapper-object-string-string)オプションが用意されている。
    * loaderは内部的にかなり無茶をやっている場合があるので、注意深く選定する必要がある。

総じて、利点としてはデザイナーフレンドリーで、かつCSSの問題点を解決する手法を得る事が出来る。
対して、欠点としてはwebpackが独特であるが故にもたらされる事象が多い。

# (本題) CSS周りの様々なloader, pluginの責務分割

だいたい下記のような責務に分割出来る。
一番左は便宜的に通し番号をつけている。

| # | 役割 | 代表的なloader,plugin | 必須？ |
|---|---|---|---|
|A|CSSのデータ -> HTML上で読み込めるものにする|style-loader, extract-text-webpack-plugin|yes|
|B|CSS -> javascriptで読み込めるという状態にする役割|css-loader, raw-loader|yes|
|C|メタCSS言語をコンパイルする役割|sass-loader, stylus-loader, less-loader|no|
|D|CSSの事後処理を担当するもの|postcss-loader|no|

一般的な場合、それぞれの分類をから一つずつ選択することになる。
例えば一般的な組み合わせで言えば、style-loader(A) + css-loader(B) + postcss-loader(D) や、extract-text-webpack-plugin(A) + raw-loader(B) + sass-loader(C)など。

逆に、同じ役割のものを複数扱う場合というのはレアケースだろう。
AとBはCSSの利用にあたって必要になる。
Dは技術的には必須ではないが、実務上はほぼ必須と言っても良い。

以下、それぞれの分類を細かめに砕いていく

## A. CSSのデータ -> HTML上で読み込めるものにする

対象は以下

* style-loader
* extract-text-webpack-plugin

大本の所になる、CSSを「扱える状態にする」もの。
CSSの内容そのものについては全く関与しない。

### style-loaderのやっていること
style-loaderは、loaderなので、前述通り最終的なOutputはjavascriptである。

どのようなコードを出力するのかを端的に言えば「CSSファイルを`<style>`タグを自動で生成して、その中に指定のCSSのソースを入れ込むjavascript」という歯切れの悪い感じになる。

利点として、下記のようなことが考えられる。

* CSSについて、javascriptのみで完結することができる
* 更新した場合のcache busterやdigestについて気にしなくて良い。
    * rails上であればsprocketsとの連携なども気にしなくてよくなる。
* Hot module reloadやsource mapなどを別途対応せずとも対応してくれる

逆に欠点として、下記のような部分が存在する。ここが問題となる場合は、extract-text-webpack-pluginを利用すると良い

* `<style>`を差し込む形なので、レガシーなCSSの置き換えが目的であったり、SSRを前提としている場合は、style適用に遅延が発生して、画面がチラつくなどの問題が出やすい。
* 共通のCSSを色々なjavascriptから読み込んでいる場合、ブラウザキャッシュが効かないので非効率になる
* CSSとjavascriptがワンパックなので、それぞれを並列で読み込んだ場合より遅くなりやすい
    * CSSが肥大化してくるとより顕著になってくることが考えられる

### extract-text-webpack-pluginのやっていること

[extract-text-webpack-plugin](https://github.com/webpack/extract-text-webpack-plugin/)は、「loaderが実行した結果を別途textに吐き出す」というもの。（CSSに限ったpluginではない）。loaderではなくpluginとして提供されているのも、最終的にjavascriptへ変換するものではないからだ。

`style-loader`では「javascriptとして吐き出す」だったのに対し、こちらでは変換されたCSSファイルを「通常のCSSファイル」として吐き出すことが出来る。

当然だが、CSSの読み込みそのものは別に管理してくれないので、自分で`<link>`タグの読み込みやdigestは別途管理することになる。
この辺りをハンドリングしたい場合や、`style-loader`での欠点を解消しなければならない場合は利用価値が高い。

## B. CSS -> javascriptで読み込めるという状態にする役割

対象は以下

* css-loader
* raw-loader

`import foo from 'foo.css'`や`require('foo.css')`といった、「javascriptからCSSを読む」という、webpack独特なやり方を解決する役割として、css-loaderやraw-loaderが存在する。

あくまでも「javascript上で扱えるようにする」という部分のみを責務としている。

### css-loader
cssをloadする場合、そこそこ高機能になるのが`css-loader`。
基本はこちらを使う方がスタンダードと思われる。

* url解決
* `@import`解決
* css-moudles対応（local/globalスコープなど）

この他にもかなり色々機能を持っている。（そのうち別途まとめたい）

### raw-loader

css-loaderは色々やってくれるが、「いや、単にCSSをjs上で解決出来てくれればそれでいいんだ」という人には`raw-loader`で十分になる。

raw-loaderはごくシンプルに

```js
module.exports = JSON.stringify(対象のファイル)
```

ということをやっているだけである。
extract-text-webpack-plugin同様、これもcssを主目的としたloaderではないので、対象は何でも利用できる。

## C. メタCSS言語をコンパイルする役割

対象は以下。この他にもおそらく存在するが、ここでは代表的なものを取り上げる

* sass-loader
* stylus-loader
* less-loader

いわゆるメタCSS言語であるscssやstylus, less。
これらの変換には、別途それぞれのloaderが存在する。
このあたりを利用する場合、gulpなどで変換して「cssファイルにする」、というのが馴染み深い手法だが、loaderを利用した場合、あくまでも「cssの文字列」にしているだけである。

less-loaderの文頭にはこんな感じでの記載が存在している。

```js
var css = require("!raw!less!./file.less");
// => returns compiled css code from file.less, resolves imports
var css = require("!css!less!./file.less");
// => returns compiled css code from file.less, resolves imports and url(...)s
```

この場合、「CSSとして変換されたlessファイルを`css`というjavascript上の変数に格納している」という処理をしているに過ぎない。

そのため、スタイルシートとして利用するには

```js
require("!style!css!less!./file.less");
```

のように`style-loader`を利用してHTML上でスタイルシートとして扱えるようにする必要がある。
前述で述べた通り`style-loader`は`extract-text-webpack-plugin`でも代替可能なものである。`extract-text-webpack-plugin`を利用した場合はgulpを利用するのと近い形でcssをファイルとして取得出来る

## D. CSSの事後処理を担当するもの

下記が対象。

* postcss-loader

autoprefixer-loaderというのも存在するが、こちらはdeprecated。

postcssについては昨今ドキュメントや利用事例も増加してきているので、多くは記載しないが、「CSSの事後処理」と位置づけられる工程となる。

様々な [plugin](https://github.com/postcss/postcss#plugins) の組み合わせから、下記のようなことを選択的に行える。

* `-webkit`,`-moz`などの、prefixerと呼ばれる接頭辞の追加（autoprefixer)
    * 昨今の開発ではほぼ必須。
* 様々な最新のCSS構文を後方互換的に変換(cssnext)
    * javascriptにおけるbabelに近い
    * 但し、cssnextで取り扱われているものの中には、少々将来的に取り込まれる可能性が低いのでは？と思えるものも一部存在するので、取扱いには要注意

postcssのプラグインの中には、postcss-assetsなど画像を扱うものもあるが、これらはwebpack上で扱う場合には役割が重複してしまうので、あまり出番は無いと思われる。
