Babel の export について検索したところ、この記事が一番上に出るようですので報告させていただきます。

Babel v6 から Babel 公式の transform である [transform-es2015-modules-commonjs](http://babeljs.io/docs/plugins/transform-es2015-modules-commonjs/) を使った場合 ES6 の `export default` を出力したときに `exports.default` に変更されてしまいました。
もし以前のように `module.exports` として出力したい場合は [add-module-exports](https://www.npmjs.com/package/babel-plugin-add-module-exports) を使う必要がありそうです。

もしよろしければ、追記していただけると幸いです。

（追記）[こちらの記事](http://qiita.com/59naga/items/10ec2b5ef0fd62a5b437)で説明されているようです。
