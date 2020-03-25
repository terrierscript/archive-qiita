
mochaを使ってテストを書いている時、branchを切ったりしているけど途中だったりするとこpushしたりすると失敗通知が結構来てうざい。かといって失敗することがわかっているテストだし消したくない・・・そんな時の話

# やり方
適当にskipする命名規則決めといて
`mocha -i skip_ci`とかでスキップしちゃうという手もある。

けどこれだとskipとしても出てこないし、、、と思ったがこんなやり方を思いついた

```test.js
// この二行
var isCi = process.env.CI ? true : false
var itSkipIfCi = (isCi ? it.skip : it)


// あとはこんな感じ
describe("logic", function(){
  it("普通のテスト ", function(){
  })
  itSkipIfCi("まだコケるのでとばしときたいテスト ", function(){
  })
})
  
```

## 解説
`process.env.CI`は[CircleCI](https://circleci.com/docs/environment-variables)、[Travis](http://docs.travis-ci.com/user/ci-environment/#Environment-variables) を見ると`TRUE`が設定されているようだ。（僕はCircleCIしか使っていない）

あとは`it`か`it.skip`かを決めるだけだがそこは下記の記事を参考にした
http://qiita.com/shoito/items/9dd99e09d0ca8eaecd4a


## その他
TravisCIの初期設定どうやるか忘れてCircleCI使ってみたけどとても良い。もうTravisに戻れない。
