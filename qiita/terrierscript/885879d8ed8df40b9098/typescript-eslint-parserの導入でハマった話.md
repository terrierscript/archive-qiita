
[typescript-eslint-parser](https://github.com/eslint/typescript-eslint-parser) を入れようとしてちょいちょいハマった
やんごとなき事情でJavaScriptとTypeScriptが混ざっている場合前提なので、TypeScriptオンリーであればここまで困らないと考えられる

# TL;DR

* jsとtsが混在している場合、に`--ext .js .ts`みたいにjsファイルも一緒に食わせると色々コケて上手く使えない
    * typescript用のコマンドにして、configを別途渡そう
* 一部のルールは上手く動かない
	* 仕方ないけどdisableしよう
* `// eslint-disable-line`とか`// eslint-disable-next-line`行単位の無効化が上手く動いてない
	* ファイル先頭での指定に切り替えていこう

# 詳細

## Typescript用に別途設定ファイルを用意する

だいたいこんな感じで`.eslintrc.js`があると思う。これをTypescriptオンリーなプロジェクトならこれを`typescript-eslint-parser`にしちゃうで良いが、混在のピロジェクトだとjsファイルを`typescript-eslint-parser`が上手くパースできず色々なルールでコケる（コケないルールもある）

```js
// .eslintrc.js 
module.exports = {
  :
  "parser": "babel-eslint",
  :
}
```

そのため、`.eslintrc`をextendしたconfigを別途作ると良い（typescriptとルール共有しなくていいなら別に化粧とかさせなくてもいだろう）

```js
// .eslintrc.typescript.js
const baseConfig = require("./.eslintrc.js")

const overrideConfig = {
  parser: "typescript-eslint-parser"
}

module.exports = Object.assign({}, baseConfig, overrideConfig)
```

当然package.jsonでの起動も別途設定になる。`-c`オプションでconfigを渡し直す

```json
// package.json
{
    "lint": "eslint javascripts/ --ext .js --fix",
    "lint:ts": "eslint javascripts/ -c .eslintrc.typescript.js --ext .ts --fix",
}
```

https://github.com/eslint/eslint/issues/3611
こちらの機能が実装されれば分割しなくてよくなりそうとのこと（ @mysticatea さんに教えていただきました！情報ありがとうございます！）

## 一部のルールは上手く動かないので除外する必要がある

ESLintのルールは当然Typescriptを考慮したものでないので、幾つか不都合のあるルールが存在するとのこと

https://github.com/eslint/typescript-eslint-parser/issues/77

このへんは惜しいが必要に応じてdisableにしていくしかない。自分の場合は`no-unused-vars`, `no-undef`をdisableする必要があった（インターフェース周りでどうしてもうまくいかない）
先ほどのeslintrc上書きを使って、ruleを打ち消してもいいかもしれない

```js
// .eslintrc.typescript.js
const baseConfig = require("./.eslintrc.js")

const overrideConfig = {
  parser: "typescript-eslint-parser"
  rules: {
   "no-unused-var": 0
  }
}

module.exports = Object.assign({}, baseConfig, overrideConfig)
```


ちなみに、typescript-eslint-parserのreadmeには[ESLintのルールのバグはこっちには報告しないでね](https://github.com/eslint/typescript-eslint-parser#reporting-bugs)とあるのでバグ報告する場合は注意

## eslint-disable-lineとかが動いてない
バグな模様

https://github.com/eslint/typescript-eslint-parser/issues/5

こういうのが動かない。

```js
console.warn(message) // eslint-disable-line
```

ファイル先頭に書く場合は動くので、そっちで代用

```js
/* eslint-disable no-console */
 :
 :
console.warn(message)
```

ちょっと不便だが致し方なし
