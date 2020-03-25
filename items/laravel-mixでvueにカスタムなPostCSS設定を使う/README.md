
# TL;DR

`mix.js`などではなく、`mix.options.postCss`に設定する

## 設定例( postcss-custom-mediaの例 )

```js
mix.options({
  postCss: [
    require("postcss-custom-media")({
      "extensions": {
		  '--phone': '(min-width: 544px)',
		  '--tablet': '(min-width: 768px)',
		  '--desktop': '(min-width: 992px)',
      }
    })
  ]
})

```

# 詳しく
laravel-mixのvueにおいて、`postcss.config.js`を配置しても`.vue`のファイルには利用されずにうまく行かなった。

コードを深追いしてみると、`mix.options.postCss`を設定すれば効かせてくれるとわかった。
https://github.com/JeffreyWay/laravel-mix/blob/d0951a8f91feef2ff0d1dd4e8aba125c5aadf394/src/builder/webpack-rules.js#L252


Autoprefixerはこれと別途設定されている。
https://github.com/JeffreyWay/laravel-mix/blob/d0951a8f91feef2ff0d1dd4e8aba125c5aadf394/src/builder/webpack-rules.js#L160-L162
Autoprefixerはデフォルトでtrueになっている。
https://github.com/JeffreyWay/laravel-mix/blob/master/src/config.js#L108
