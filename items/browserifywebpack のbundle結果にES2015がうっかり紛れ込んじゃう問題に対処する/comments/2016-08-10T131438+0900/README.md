デフォルトでは `node_modules`以下はtranspileされないので、手動にはなるのですが、

```js:tools/babel-register.js
module.exports = require('babel-register')({
    ignore: /node_modules\/(?!modules-to-transform)/
})
```

こんなのを入れてみるとか。
ただ確かに、どのモジュールに動かないコードがあるのか、を事前に探す必要があったのですね...失礼。

やはりこれはライブラリ提供側の問題。ある程度の規模なら配布用のファイルをbabelで生成すべきです。。
1ファイルで、大げさになりそうなら、devDependenciesでacornを入れてテスト内に組み入れるべきですね、
自分もやってみます。
