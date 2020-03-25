
備忘録

webpack2で下記のような警告が出る

```
WARNING in asset size limit: The following asset(s) exceed the recommended size limit (250 kB).
This can impact web performance.
Assets:
  my-file.min.js (251 kB)

WARNING in entrypoint size limit: The following entrypoint(s) combined asset size exceeds the recommended limit (250 kB). This can impact web performance.
Entrypoints:
  main (251 kB)
      my-file.min.js


WARNING in webpack performance recommendations:
You can limit the size of your bundles by using import() or require.ensure to lazy load some parts of your application.
For more info visit https://webpack.js.org/guides/code-splitting/
```

警告は直せるほうが良いとはいえ、デフォルト設定がかなりきつく（多分require.ensureとかSystem.importをベースに考えられている）
真正面から向き合おうとするとなかなか簡易じゃないパターンがあったり、ライブラリビルドの場合そもそも無用なものだったりするパターンがある。

# 解決編

ざっくりオフにしてしまうなら下記

```js
module.exports = {
  module: {
   //...
  },
  plugins: [
   //...
  ],
  output: {
   //...
  },
  // ...
  performance: { hints: false } // これ
}
```

詳しくは下記参照

https://webpack.js.org/configuration/performance/#performance-hints

全部オフにせず閾値を変えるのも出来る

* maxEntrypointSize
    * entrypointサイズの警告の閾値を変更する。
    * defaultは250kb
* maxAssetSize
    * assetのファイルサイズ警告の閾値を変更する
    * defaultは250kb
* assetFilter
    * asset警告を出すファイルを決定する関数
    * `.map`ファイルは無視するとか出来る
