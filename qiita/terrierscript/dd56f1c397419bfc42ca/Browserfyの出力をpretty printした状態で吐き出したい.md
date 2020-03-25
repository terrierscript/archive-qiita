
Browserifyさんどんな出力吐いてるの？と思って研究したい時に欲しくなった。

# 答え
[js-beautify](https://github.com/beautify-web/js-beautify)使う。

```sh
$ npm i -D js-beautify
```

```package.json
"scripts": {
  "beautify-build":"browserify -e ./src/index.js | js-beautify -f - > build/index.js"
}
```
だいたいこんな感じ。
beautifyfyとかあるかなと思ったけど無かった。
