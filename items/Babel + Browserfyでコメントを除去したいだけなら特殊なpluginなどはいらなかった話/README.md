
minifyじゃない。コメントだけ消したいんだよ！
みたいなのにちょっと苦戦したのでメモ。

# 結論
特にpluginなどはいらなかった。`babelify`で十分

```js
gulp.task('babel-build', function(){
  browserify({
    entries: "someFile.js",
    extensions: ['js', 'jsx'],
  })
  .transform(babelify.configure({
    comments: false // これ！
  }))
  .pipe(source("bundle.js"))
  .pipe(gulp.dest("/dest"))
})
```

## ドキュメントとか
https://babeljs.io/docs/usage/options/
↑
ここ。
`comments`オプションがあった。最初は「コメント除去とかはbabelの責務なわけ無い」と思ってたけどどうもそうでもないらしい。思い込みだった。
sourcemapとかも多分一番綺麗に対応してくれる気がする。

## 経緯とかメモとか

- 最初、[gulp-uglify](https://www.npmjs.com/package/gulp-uglify)を試してみた
	- → 失敗。コメントだけ消したいけど結構色々変換されててだるい。
	- 色々オプションいじってみるが思った感じにならぬ。辛い。
	- まあ別にそれでもいいんだけど・・・
- 次に[minifyify](https://www.npmjs.com/package/minifyify)を試す
	- → callbackの書き方とgulpの組み合わせが微妙そうで挫折。
- 次に[gulp-strip-comments](https://www.npmjs.com/package/gulp-strip-comments)を試す
	- → 一応動くっぽいがsourcemapがうまくいかない・・・
- ここでbabelのドキュメント読み返す。
	- あるじゃん！
	- おしまい。
