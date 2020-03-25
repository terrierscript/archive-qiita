
[babel-polyfillとbabel-runtimeの使い分けに迷ったので調べた](http://qiita.com/inuscript/items/d2a9d5d4daedaacff924) の続編・追加調査。

使い分け的なのわかったものの、未だにどうにも困りごとが多いので、更に深掘りしてみた。

# それぞれのおさらい & 困りごと
[polyfill](https://babeljs.io/docs/usage/polyfill/)のページで改めておさらいしつつ、それぞれの困りごとを書いてみる

* babel-polyfill
    * [core-js](https://github.com/zloirock/core-js)と[regenerator-runtime](https://github.com/facebook/regenerator)を読み込んでいる
    * polyfillとして、windowグローバルを拡張する仕組み
    * 困りごと
        * 複数ファイルからロードするとthrow errorされてしまうので、うっかりしてるとハマるので気を使う
        * 上記のような問題のため、ブラウザの場合は、別途[dist/polyfill.jsなど、別管理してを読み込む](https://babeljs.io/docs/usage/polyfill/#usage-in-browser)。
	        * `require('babel-polyfill')`の様にコードベース上から読み込むことは推奨されてない。
            * これをやると二度読みでコードが完全に死んでしまうので大変危険
        * polyfillを別なscriptファイルとして読み込ませることになったりする。
* babel-runtime / babel-plugin-transform-runtime
    * Polyfillが必要なコードを静的解析して、コードごと置き換える。
    * 静的変換なので、インスタンスメソッドの`Array.prototype.includes`とかは変換出来ない(`[1,2,3].inclues(3)`みたいなコードのはずなので、そりゃ辛そう)。
    * 困りごと
        * `.babelrc`やたらコンパイルでエラー起こしてくる問題児感。
        * [export構文との組み合わせにバグ](https://github.com/babel/babel/issues/2877)が現行バージョンでは存在したりする。[ワークアラウンドも存在する](http://qiita.com/inuscript/items/65456941bd8a778b0641))が、ちょっとしんどい。
        * 個人的なユースケースだと`polyfill`でも十分という印象。

## じゃあ`core-js`を読み込むのはどうなのか？

babel-polyfill / babel-runtime　それぞれ内部的に`core-js`を利用している。

「じゃあ`core-js`を直接使うのはどうなのだろう？」と考えた（regenerator-runtimeについては個人的にあまりお世話にならない都合で割愛させていただく。ほとんど扱いは一緒であろうと思われる）。

ここで改めて[Polyfill](https://babeljs.io/docs/usage/polyfill/)のページを見てみると、こんな事が書いてある。

> Note: Depending on what ES2015 methods you actually use, you may not need to use `babel-polyfill` or the runtime plugin. You may want to only [load the specific polyfills you are using](https://github.com/zloirock/core-js#commonjs) (like `Object.assign`) or just document that the environment the library is being loaded in should include certain polyfills.

要約すると、「必要なものだけ（例えば`Object.assign`など）を[選択してロード](https://github.com/zloirock/core-js#commonjs)しても良い」

とのこと。つまり、「core-jsでpolyfillするのはアリかナシか？」という事については **アリ** と言えそうだ。

# ちょっと待った。それでもbabel-polyfillがごにょごにょしてるのは気になる

自前でやるにしても、それはbabel-polyfillとどのぐらい差異が出るのかは気になる。

[babel-polyfillのコード](https://github.com/babel/babel/blob/v6.19.0/packages/babel-polyfill/src/index.js)を読んでここから細かく検証してみたい（versionは6.19.0）

大まかにこんな具合になっている。

1. globalの変数を利用して、複数読み込まれるのを制御([L3-6](https://github.com/babel/babel/blob/v6.19.0/packages/babel-polyfill/src/index.js#L3-L6))
2. [core-js](https://github.com/zloirock/core-js)と[regenerator-runtime](https://github.com/facebook/regenerator)の読み込み([L8-9](https://github.com/babel/babel/blob/v6.19.0/packages/babel-polyfill/src/index.js#L8-L9))
3. core-jsに対するモンキーパッチ的な処理([L11-29](https://github.com/babel/babel/blob/v6.19.0/packages/babel-polyfill/src/index.js#L11-L29))

ということで、1と3について見てみる。

## babel-polyfillの読み込みを一回しか許可しない

なぜbabel-polyfillはなぜ1回しか読み込みを許していないのか？

この疑問を調べるのに、下記２つのissueが役に立った

https://github.com/babel/babel/issues/4019
https://github.com/babel/babel/pull/2050

この２つを総括すると、こんな具合が見て取れる

* polyfillというのは得てして冪等ではない。**複数回読み込む事で副作用が起きる懸念がある。**
    * これがcore-jsのことを指しているのはpolyfill全体のことを言っているのかはちょっと掴めなかった。
* polyfillが衝突することで、問題を起こしやすい
* そのため、複数箇所で読むべきではない
* とはいえ、「`throw error`するのはやりすぎなのでもうちょっとスマートにやってほしい、ベストエフォートな振る舞いにしてほしい」という声が意見が散見される（個人的にもこれに同意）
* libraryを作るなら、globalを汚染せぬよう、babel-transform-runtime使うべきという情報もあった。

ワークアラウンドとして、babelが利用している`_babelPolyfill`という変数をチェックするハックも紹介されている。

```js
if (!window._babelPolyfill) {
  require('babel-polyfill')
}
```

ということで、babel-polyfillは **polyfillの複数回読み込みによるバグを回避するために複数回読み込みを禁止している** とわかった。

ただし、今回の調査では「どのようなバグが起きるのか？」という事までは調べられなかった（もし知っている方がおりましたら是非コメントいただければ幸いです）。
core-jsも覗いてみたが、「複数回読み込むな、読み込むとバグがある」という報告や「冪等である（冪等になった）」というchangelogは見当たらない。
実装を読み得る限りで読む分には、[nativeを優先する](https://github.com/zloirock/core-js/blob/v2.4.1/modules/_export.js#L22-L23)ようになっているようにも読めた。

この「一度のみ読み込ませる処理」というのは[6to5](https://github.com/babel/babel/commit/377212290fd6f12cfbaa4f279ad5a861efb7c545)時代から引き継がれているものだったりするので、実は今のcore-jsでは問題無い可能性（または少なくなっている可能性、かなりのエッジケースだったりベストエフォートとして片付けられる可能性）も考えられる

とはいえ、基本としては複数回読み込むことへの副作用は注意すべき観点だろう

## core-jsに対するモンキーパッチ？

core-jsに対して色々モンキーパッチされている箇所は、文頭に
`// Should be removed in the next major release`とある。
つまり、babel 7系になった際にはこれら処理は消えるものと思われる（core-jsのnext-versionとも読めてしまうが、下記の文脈を考えるとbabelのmajor versionの事と推測できた）

では具体的に何が書かれているかというと、[core-js 2.0.0へのアップデート](https://github.com/zloirock/core-js/blob/master/CHANGELOG.md#200---20151224)時にremoveされたものを、後方互換措置として差し戻す処理をしている。

具体的には、下記のような差し戻し処理がされている。

* `core-js/`から`regexp/escape`の読み込み
    *  `RegExp.escape moved from the es7 to the non-standard core namespace`を差し戻す後方互換処理
    * `import "core-js/fn/regexp/escape"`しているだけ
* `padLeft`, `padRight`の互換維持
    * `renamed String#{padLeft, padRight} -> String#{padStart, padEnd}`という変更を差し戻す後方互換処理。
    * `padStart` -> `padLeft` / `padEnd` -> `padRight`と扱えるようにしている。
    * `padLeft` / `padRight`というメソッドは[TC39 Meeting](https://github.com/rwaldron/tc39-notes/blob/master/es7/2015-11/nov-17.md#stringpadleftright)でrenameが決定しているので、もしこれを利用しているなら速やかに`padStart` / `padEnd`に置き換えるべきだろう
* `Array`のジェネリクスの再定義
    * `removed Mozilla Array generics`を差し戻す後方互換処理
    * すごくコード的にわかりづらいが、`Array.pop`みたいな呼び出しで、`[1,2,3].pop`と同じメソッドが呼び出されるようになっている
    * この再現方法が結構難解だったので、だいたい何やっているかを自分向けに分解した[gist](https://gist.github.com/inuscript/7c2b578c7d02200ba6b0279e1b54cc3a)も併せて貼っておく
    * core-jsとしては、もしこの機能がほしい場合は[array-generics](https://github.com/plusdude/array-generics)という別なshimを利用するようアナウンスされている

# FYI: [create-react-app](https://github.com/facebookincubator/create-react-app)も独自にpolyfllしてる。

偶然見つけたが、reactをさっくり作るためのツールであるcreate-react-appも、独自なpolyfillを作成している。
このコードは`npm run eject`するとコードベースに出力される。

https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/config/polyfills.js

下記がPolyfillされている

* `Promise` ( [promise](https://github.com/then/promise) )
* `fetch()` ( [whatwg-fetch](https://github.com/github/fetch) )
* `Object.assign` ( [object-assgin](https://github.com/sindresorhus/object-assign) )

# まとめ
* core-jsなどで別途自己管理でpolyfillするのは割りとアリ
	* core-js自体は独立したpolyfillライブラリなので、スタンドアローンに十分使える
* ただし、polyfillは冪等というわけではないので、そのあたりの管理はやったほうが良い
    * core-jsでpolyfillするにしても、複数回呼び出しは避けるのが賢明と言えそう
* `babel-polyfill`が差し戻してる`core-js`から落とされた機能を利用する場合には、何らか処理が必要
	* とはいえ、おそらくどこかで消えるので、なるべく早めに根本解決したほうが良い。
* ライブラリ提供を目的としてるならbabel-runtime一択。global汚染するpolyfillに出番は無い
