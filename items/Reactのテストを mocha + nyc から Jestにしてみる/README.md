
# mocha + nycのころ

## プロジェクト構成など

package.jsonはこんな感じ。

```package.json
{
  "test": "NODE_ENV=test mocha test/",
  "coverage": "nyc -c npm test"
}
```
* /srcにソースディレクトリ
* /testにテストディレクトリ
* MISC
	* [jsdom-global](https://github.com/rstacruz/jsdom-global)使って色々小細工していた。

## mocha -> jestにするモチベーション

* 興味本位
* Jestがv15.0.0でAutomockingを捨てて本気を出してきたのでちょっと使ってみたかった気持ち
* Railsプロジェクトにjavascriptプロジェクトを混ぜようとしたとき、mochaのテストディレクトリに悩んでしまうぐらいなら、Jestのテストフでやってみようかと思った
    * jestでは、/srcなどのコードに対して、`/src/__tests__`というディレクトリを作るか、`some.test.js`という名前をつけるのがデフォルト設定になっている。
    * mochaでも設定によっては可能
* coverage周りに感じた魅力


# jestへのmigrate

## package.jsonまわり

とりあえずmochaからの移動をまず考えるので、`test/**/*.js`のような感じでテストディレクトリの指定を行う。
coverage入りなので、coverageコマンドも変える

```package.json
{
  "scripts":{
    "test": "NODE_PATH=${npm_package_config_resolve} jest ",
    "coverage": "npm test -- --coverage",
  },
  "jest": {
    "testRegex": "test/.*.js"
  }
   :
}
```

`test/mock`みたいなところに非テストデータ入れてるならignorePatternを設定

```json
  jest": {
    "testPathIgnorePatterns": ["node_modules", "test/mock"],
  }
```

## ~~テストのskip~~

**2016/10/14追記：いつのまにか`.skip`が実装されていたようです。**
https://facebook.github.io/jest/docs/api.html#basic-testing
https://github.com/facebook/jest/pull/1632


mochaだと`.skip`だった

```js
describe('some test', () => {
  it.skip('skip')
})
```

jestだと`xit`や`xdescribe`になる。

```js
xdescribe('some test', () => {
  xit('skip')
})
```


## 空ファイルはエラーになる
```
Your test suite must contain at least one test.
```

```js
xdescribe("todo", () => {
  xit("todo")
})
```

```js
it("todo", () => {}
```

## jsdomは消せる
jestにして嬉しいポイント

mochaだと色々頑張っていたが、既に入っているので気にしなくて良くなる
http://facebook.github.io/jest/docs/api.html#testenvironment-string

## assertとexpect

標準では`expect`というmatcherが用意されており、そちらに重点が置かれている。
個人的には`assert`を使っていきたいので、ちょっと不満ポイント。

assertも使えないわけではない。普通にthrowされればエラーとなる。

ただし、expect(foo) で書くとエラー時の表示がきれいになる。

このぐらい違う。
無理に全部置き換えるまででもないかなと思ったが、郷に入れば郷に従えで今後はexpectに置き換えていくだろうなと思う。

`addMatchers`などあるのでもしかするとなんとかならないかなと思っているが未調査。

![6__npm_run_test__node_.png](https://qiita-image-store.s3.amazonaws.com/0/7307/825f40dd-6056-ef6c-6df9-44ede54e41f5.png "6__npm_run_test__node_.png")

![6__npm_run_test__node_.png](https://qiita-image-store.s3.amazonaws.com/0/7307/2539883c-87fb-08d6-9fbb-a572952c0827.png "6__npm_run_test__node_.png")


## watchモード

jestにちょっと不満ポイント２。

mochaだと`-w`で良かった

```
$ mocha -w
```

jestは`--watch`。
ちなみに`-w`は`--maxWorkers`という値の省略になってしまっている。

```
$ jest --watch
```

しかしwatchモードの挙動自体は結構イケてる

# 感想
* 移植自体は簡単
* やはりReactとは相性がとても良い。
* シンプルさというよりはフルスタックさに振っている感じがある
* 並列でテストを走らせてくれているっぽい。
* React以外のプロジェクトで普段使いするかというと、うーんという感じ。
* 無理して移植するほどのメリットがあるかというと、そんな強くもない
	* とはいえ、そもそも移植コスト掛かってないので、興味があったらやってみてもいいと思う
* Snapshotテストとか手出し始めたらまた感想変わるかも
* なぜかCircle-CIでテストが遅くなっているので調査中
* そこそこ大規模なReactプロジェクトで「テスト入れるぞ！」となったときの選択肢としては全然アリ
	* mochaでちょっとしんどかったjsdomまわり気にしなくていいのも大きい 
