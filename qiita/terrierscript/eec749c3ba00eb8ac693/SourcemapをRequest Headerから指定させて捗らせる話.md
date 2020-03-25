
SCSSなどのprecompileを使うときにはもはや当たり前のになってきたたSourcemap。
あんまり気にせず使うことも出来るが、レガシー環境などで快適に使いたくなったりするとちょいと工夫が欲しくなる。

## Sourcemapを使う方法

Sourcemapを読み込む方法として、だいたい3つぐらい手法が用意されている

1. 該当の出力コードにインラインで記載する
2. 該当の出力コードに、別途出力したSourcemapファイルの位置を指定する
3. `X-Sourcemap`のリクエストヘッダで、別途出力したSourcemapファイルの位置を指定する

## 出力コードに手を加えるやり方

1, 2のソースコードが手法はわりとメジャー。
結構良く説明されている。qiitaでも下記のあたりの記事が見つかった

- http://qiita.com/kyaido/items/04658c04a7dbe9f05c16
- http://qiita.com/yokoh9/items/bc948a6ea7dd21f9c31d

が、これらの手段、こんな欠点がある

- 出力cssそのまま本番環境にアップロードしてしまうと、元コードが見られたりファイルパス書かれたりするのでちょっと気まずい
- Sourcemapファイルをuploadしなければ、元コードは参照されないが、一部ブラウザでは読み込みエラーなどが吐かれてしまいちょっと気まずい
- このあたりを回避するとなると、本番uploadする前に何らかの形で再ビルドすることになるってちょっと面倒

モダン環境だとビルドをちゃんとやってくれたりするので気にならないが、そうじゃない環境だと結構捗らない。

## ということでHeaderでSourcemap指定をする

あんまりこの手法を使っている人が少ないのかもしれないが、この方法についてはいまいち文献が少なかったりする。
せいぜい見つけられたのがこのあたりだった。
[Introduction to JavaScript Source Maps](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)

とはいえ別に複雑なこともなく、リクエストヘッダの`X-SourceMap`で指定してやれば良いというだけだった。

```
X-SourceMap: /path/to/script.js.map
```

### タスクランナーの設定をいじる
gulpなどの設定で普通のSourcemap指定と違うのは、`{addComment: false}`の指定をして出力cssには影響が出ないようにすること。
これで`some.css`のSourcemapファイルは`some.css.map`てな具合に出力され、元のcssは普通にSourcemapナシ状態が作れる。

```gulp.js
var gulp       = require('gulp');
var sass       = require('gulp-sass');
var sourcemaps = require('gulp-sourcemaps');

gulp.task('sass',  function () {
  gulp.src(pathToSrcSass)
    .pipe(sourcemaps.init())
      .pipe(sass())
    .pipe(sourcemaps.write("./", {addComment: false, debug:true}))
    .pipe(gulp.dest(pathToOutputSass))
})
```

### X-Sourcemapが出力されるようにする。

ヘッダーで`X-Sourcemap`を指定するにはapacheの設定で指定したり`.htaccess`を使ったりあたりが思いつくが、今回のサンプルでは`.htaccess`を使う。

```.htaccess
<IfModule mod_rewrite.c>
  RewriteEngine On
  # source map
  RewriteCond %{REQUEST_URI}          ^(.+\.(css|js))$
  RewriteCond %{DOCUMENT_ROOT}%1.map  -f
  RewriteRule .                       - [L,E=SOURCE_MAP:%{REQUEST_URI}.map?%{QUERY_STRING}]
  Header set X-SourceMap "%{SOURCE_MAP}e" env=SOURCE_MAP

</IfModule>
```

やっていることとしては`css`か`js`の場合に`%{DOCUMENT_ROOT}%1.map`が存在していれば`X-SourceMap`を指定するという具合な感じ。

また、出力先を`./map`とかにしたい場合は併せて`%{DOCUMENT_ROOT}%1.map`の部分を変える感じにすればよい。


これによりリクエストヘッダでのSourcemap指定が歓声するので、本番用ビルドをしたり、Sourcemapファイルが無かった場合に妙な感じになったりしなくてとても良い環境が出来る。
