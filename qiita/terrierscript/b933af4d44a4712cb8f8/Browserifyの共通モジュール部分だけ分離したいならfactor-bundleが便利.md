
Brwoserifyを使うにあたって、普通にbuildしていると１ファイル１ファイルがモリモリ肥大化してしまう。であれば共通部分の切り出しだ！
というところで[factor-bundle](https://github.com/substack/factor-bundle)というのがあるのだけど日本語記事とか無いのでサンプルを記載しつつまとめたい。

# factor-bundleとは？
browserifyの対象ファイルを解析して __共通部分になるmoduleだけを自動的に切り出してくれる__ pluginである。

立ち位置としてはbrowserifyのpluginである。
また、browserify作者であるsubstack氏によるプロジェクトであるので、挙動してそんなに不安を感じる必要もないだろう。
コードも実質index.js一つなのでそう複雑なことをやっているわけでもない。

## 使い方
公式にはコマンドラインを利用したこんなサンプルが添えられている

```
browserify x.js y.js -p [ factor-bundle -o bundle/x.js -o bundle/y.js ] \
  -o bundle/common.js
```

x、yというファイルをentryにして、それ以外がcommonに流れていく。
どれが共通になるのかは自動決定される。


# gulpによるサンプル
私感だが、分割したい要件って中規模ぐらいなはずで、gulpの一つぐらい必要になりそうだなーと思った。
が、gulp使ったサンプルがなかったので作ってみる。
コマンドラインでの詳しい使い方は本家READMEなどを参照されたい。


## npm install
必要なパッケージをどしどしと入れる。
二行目のものたちは、今回に必須というわけでもないけどあると便利系なのであとはお好みで。

```sh
$ npm i -D gulp browserify babelify factor-bundle vinyl-source-stream
$ npm i -D mkdirp glob del
```

## ディレクトリ構成
だいたいこんな感じ。

```
.
├── output
│   ├── client
│   └── entry
├── src
│   ├── entry
│   └── lib
└── vendor
```

## gulpfile
要点を小うるさく注釈していく。

```gulpfile.js
var gulp = require('gulp')
var path = require("path")
var browserify = require('browserify')
var babelify = require('babelify')
var source = require('vinyl-source-stream')
var glob = require("glob")
var del = require("del")
var mkdirp = require("mkdirp")
var factor = require('factor-bundle')

var dest = './output/'

gulp.task('clean', function() {
  del(dest)
})

gulp.task('browserify', function() {
  // entry対象が結構あるような状態を想定
  var entries = "./src/entry/**/*"
  
  // 出力先をこのあたりに・・・
  var outputDir = "./output/entry"
  
  // factor-bundleの出力先を決める。
  // ちゃんとディレクトリ出力先を変更しておかないと元ファイル書き換えられちゃうので注意！
  var files = glob.sync(entries, {nodir: true})
  var outputs = files.map(function(file){
    return file.replace("./src/entry", outputDir)
  })

  // factor-bundleの出力先のディレクトリが無かったりすると面倒見てくれないので作っとく。
  mkdirp.sync(outputDir)

  // browserify部分開始
  browserify({
    entries: files,
    extensions: ['js', 'jsx'],
  })
  // transformしたいならこんな具合。最近鉄板なのでbabelっとく。
  // babelのtransformに際して、何らかの理由でnpm入りしないなーみたいなのがあれば対象外にしておく。
  // ignoreオプションではなくonlyオプションを使うのもよい
  .transform(babelify.configure({
    ignore: /vendor/
  }))
  // factor-bundleの登場。先ほど用意したoutputを追加
  .plugin(factor, {
    output: outputs
  })
  // 共通部分を出力してやる。
  .bundle()
  .pipe(source("common.js")) 
  .pipe(gulp.dest(path.join(dest, "client")))
})

gulp.task('watch', function() {
  gulp.watch('src/**/*', ['browserify'])
})

gulp.task('default', ['browserify', 'watch'])

```

これで`output`には`src/entry`にあったファイルが変換されたものと、`common.js`が出力される。

あとはこんな感じで呼び出せば完成

```html
<script type="text/javascript" src="./output/client/common.js"></script>
<script type="text/javascript" src="./output/entry/foo.js"></script>
``` 

# 感想とか
- 共通部分を作りたい要件あるならこのpluginはおすすめ。
	- requireとかexternal駆使してやるの、最初やってみたけど割りと辛かった。
	- 「このプロジェクトはjqueryぐらいしか共通部分無い」とかだったらrequireとexternalぐらいでも十分だろうとは思う
	- が、たとえば「コアライブラリ」「お手製lib部分」「お手製entry部分」ぐらいな粒度で分割しだすとちょっともう辛い。factor-bundle使う方が圧倒的に良い。
	- 逆に大規模開発になってきて更新の度に本番環境への負荷がヤバイ、とかなったら再検討してもよいだろうと思う。
		- common部分が作りによってゴリゴリ変わっちゃうので、キャッシュの仕組みによってはcommon部分がキャッシュ効かない〜というのはありそう。
		- あんまり考えたくない。
- ちなみにpartition-bundleというのも紹介されていたけどよくわかんなかった。

## 参考文献
- [factor-bundle](https://github.com/substack/factor-bundle)
	- プラグインのページ
- [Browserify Handbook](https://github.com/substack/browserify-handbook)
	- partitioningという章がある。（が、あまり詳しく書いている感じではない）
- [browserify for wbpack users](https://gist.github.com/substack/68f8d502be42d5cd4942)
	- browserify作者によるwebpack -> browserifyの移行ガイド
- [Create separate JavaScript bundles with a shared common library using Browserify and Gulp](http://stackoverflow.com/questions/23748841/create-separate-javascript-bundles-with-a-shared-common-library-using-browserify)
	- 概ねここのサンプルを元にした。
