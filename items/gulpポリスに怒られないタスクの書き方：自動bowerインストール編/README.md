
タスクの中でbower install的なことをしたかった話。

## ダメだった方
最初に目についた`gulp-install`というのを使ってざっくり下記な感じでやってみた。

が、これが癖者。finishのイベントのハンドリングがうまくいかない
 
```bad-case.js
gulp.task('install', function(cb){
  gulp.src(["./bower.json"])
    .pipe(install())
    .on('finish', function(){
      cb();
 	 });
  });
});
```

## もっかい調べてみる。

調べると類似で`gulp-bower`なんかを見つけたりした。
が、ここでひとまず落ち着いて[blacklist](https://github.com/gulpjs/plugins/blob/master/src/blackList.json)を見てみる。

`gulp-install`と`gulp-bower`、どっちも載っている。
これではgulpポリスに怒られてしまう。

## なおす
gulp-installについては `"use gulp-spawn"`だそうな。

今回やりたい事はbowerを自動でインストールしたいというところなので、`gulp-bower`あたりをみる。

```
"use the bower module directly"
```

なるほど。bower直接使えと。

[bowerの公式](http://bower.io/docs/api/#programmatic-api)を見ながらこんな感じで書いてみる

```bower-directory.js
gulp.task('install', function(cb){
  bower.commands.install()
    .on("log", function(log){
      gutil.log(log.message);
    })
    .on("end", function(){
      cb();
    });
});
```

なるほど。うまく動いたっぽい。

## 締め
blacklist、普段はあのやり方好きじゃないなーと思って最初見てなかったけど、今回は普通に役に立った。
