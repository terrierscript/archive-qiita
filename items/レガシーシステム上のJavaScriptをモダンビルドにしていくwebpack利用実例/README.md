Livesense Advent Calendar 3日目の記事です。

この記事では自分たちが業務でやってきたJavaScriptコードのwebpack + babel導入実例ついてまとめていきたいと思います。一部でも参考になれば幸いです。

# まえがき・前提

## 今回取り扱うこと

今回この記事では下記の観点で進めていきます

* JavaScriptのビルドのお話し
    * CSS関連は取り扱わない（ちょっと触れる）
* 実践的なwebpackの利用方法・ノウハウ
    * 入門的な部分は扱わない
* 既存コードが存在するプロダクション環境で既存のコードを置き換えることを前提とした泥臭めな話中心
    * ワークアラウンドとしてダーティな手法を利用している部分も含まれる
    * 新規開発を始める際などは結構違う文脈なところが多い。
        * 別なプロジェクトでは一部機能のみBabelビルドしたり、完全に最初からフロントエンドビルド部分をwebpackに任せていたりする。それらと比較するとそちらとはちょっと違うテクニックが必要だと体感している

既に良いドキュメントが充実してきた入門的な部分や新規から始めるケースでの話は割愛し、比較的見かけない「既存コードをどうコンパイルしていくか？」というような部分にフォーカスを当てていきます。

## 記事中で扱う「レガシー」の定義

レガシーという単語が広義になってしまうので、この記事では自分達が元々置かれていた状況のような状態をレガシーと定義して進めます。

我々の場合、下記のように複数のJavaScriptの読み込みをしていました。

```html
<script src="lib/jquery.js"></script>
<script src="lib/jquery.someFunction.js"></script>
<script src="common.js"></script>
<script src="pc/someFunction.js"></script>
<script src="pc/hogeFunction.js"></script>
```

JavaScriptのファイル自体はこのような具合。

```js
(function(){
  var self = this;
  $("button").click(function(){
    //
  })
})()

// 古いコードはglobalに定義された何か
SomeLib.prototype.foo = function(){
    //
}
```

下記のような特徴があります。

* ライブラリ系（jQueryやjQuery.functionなど）は別途読み込み
* common.jsのようなよくある共通ファイルが存在
* 1ページ中で、scriptはファイル単位で複数読み込み
* 原始的なJavaScript配信状態（Sprocketsのようなビルドプロセスとは全く無縁な状態）
* jQuery以外に、ライブラリ・フレームワークなどは基本入っていない
* 色々とJavaScript初心者殺しな記法が多い
    * `(function(){})()`を利用したglobal汚染回避
    * `self = this`のようなthis束縛
* 雑多なGlobal管理

# 本文：JavaScriptビルド戦略

## 基本戦略・方針

前述通り今回の目的として「一部分だけ新しい部分を導入する」ではなく「プロジェクト上に存在する既存コードを変換する」という方向性に定め、まず思想的な部分での方針を考ることからはじめました。

### その1: 無理せず背伸びせず。程度は妥協も許す

前提がそれなりに厳しい環境の場合、いきなり高みを目指すと破綻してしまうことはよくあります。
自分たちの環境もそこそこ難しい環境だったので、ベストよりベターを選択していくようにしました。

* 既存コードは移植時にはなるべく維持。
    * 我慢して必要以上にコード自体はいじらない。ES2015にしたくなっても我慢する。
* ちょっとずつ動作確認出来るよう、ページ単位で置き換えるようにする
    * 全部のコードをいきなりワンパックにするとかはかなり無理が出そうなので諦める
* jQueryは外部ファイルとしてそのまま持ってくる。無理してモジュール化しない
    * versionも1.7と古いが、迂闊なアップデートは辛いのでそのまま持ってくる。
* 導入作業時にいきなりReactとかデカ目のフレームワークはいきなり入れない。落ち着いてから検討

### その2: 達成目標

一定の妥協を許したはしたものの、軸がぶれないように「ここは達成したい」というのも併せて明確にしました。

* babelによるES2015が利用可能な状態
* npmを通したパッケージ利用
* module分割管理
* minify / コメント除去

## ビルドツール選定：webpack vs Browserify vs rollup とか

ビルドツールとして、webpack , Browserify , rollup を選定対象として検討しました。
最終的にはwebpackに落ち着きましたが、そこまでどのような観点で比較したかを下記に記載します。

* Browserify
    * コマンドラインで動くことを前提に作られていて非常にシンプル
    * configファイル不要
    * npmを利用するなら非常に扱いやすい
    * 複数ファイル分割が辛い。
        * factor-bundleなどを使えば、出来なくはない。
        * サイト全体で利用するスクリプトをいきなり1ファイルにまとめるのはどう頑張っても厳しい
    * どうしても`package.json`で扱う引数が多くなってしまって、チーム開発のことを考えると保守の面で職人技になりがち
* webpack（採用）
    * なんでも出来る
    * 複数ファイルへの分割、グローバル変数をよしなにする部分など、レガシー環境が気にしないといけない所を色々面倒見てくれる
    * configファイル必須（設定ファイルは思ってたより破綻しづらい）
    * webpack2になったらtree shakingできそうという期待あり
    * 選定していた頃はまだwebpackもそこまで「ぼちぼち流行ってきた」程度でだったので他のシステムとの互換性について心配していたが、[現状かなり広まった](http://stateofjs.com/2016/buildtools/)おかげか最近は結構他のOSSが追従しだしている感覚を受ける
* rollup
    * Browserifyと思想的に近い
        * とはいえ、configファイル前提な作り
    * tree shakingとかはちょっと面白い
    * 流石に出たてで導入する勇気は無かった。
    * 結果を1ファイルとして吐き出すことが得意。多ファイルに分けるのは不得意

個人的な趣味で言えば色々やれすぎちゃうwebpackよりもシンプルに収まるBrowserifyの方が好みです。
とはいえ、やはりどうしても柔軟な対応が求められる環境においてはwebpackが一番適しているという判断に落ち着きました。

### webpackとの付き合い方

前述どおり、webpackそのものはかなり自由度が高いツールです。やろうと思えばどこまでも出来てしまいます。
この自由度を持ったまま無造作に使うのは破綻の予感がしたので、導入時は下記の通り制約付きで利用することにしました。

* JavaScriptの変換以外は責務としない
    * loaderは基本`babel-loader`だけ
    * scss変換、fonticonなどは、現行で利用していたgulpに任せる
        * `npm scripts`を通してビルドタスクを管理していたので、併用しても利用上は意識しない
    * 自分がbabelに慣れていたので`babel-loader`をすぐ導入したが、特に必須なものではない。
        * webpackをローダー無しでモジュール解決にだけ使う方法も良い方法だと思う。
* `require('something!babel')`などの`require`経由のloader指定は利用しない
* babel-loaderもqueryを利用せず、`.babelrc`を利用する
    * 個人的な趣味に近かったが、`query=XXX`でbabel設定を書くのではなく、`.babelrc`へ記載するようにした。
    * `webpack.config`の1ファイルに全てまとめるのも悪くないのだが、なるべく標準的な方に寄せた

## ディレクトリ構成とバンドル

ディレクトリは試行錯誤の末このような形になりました。

```
.
├── assets
│   └── javascripts
│       ├── desktop
│       │   ├── entries
│       │   │   ├── company.js
│       │   │   ├── home.js
│       │   │   └── work.js
│       │   ├── modules
│       │   │   ├── company
│       │   │   │   ├── industryDropDown.js
│       │   │   │   └── toggleItem.js
│       │   │   ├── home
│       │   │   └── work
│       │   └── file
│       ├── sp
│       │   ├── entries
│       │   ├── modules
│       │   └── file
│       ├── legacy
│       │   ├── jquery-fn
│       │   │   ├── jquery.bazFunction.js
│       │   │   └── jquery.someFunction.js
│       │   └── oldLib
│       │       ├── SomeLib.js
│       │       └── util.js
│       ├── override
│       │   └── common.js
│       └── lib
└── web
    └── javascript
        ├── dist
        │   ├── desktop
        │   │   ├── company.js
        │   │   └── work.js
        │   └── sp
        └── common.js
```

* assets/ - 新しいJavaScriptファイルは基本全部こっちに
    * desktop/ - PC向けのスクリプト
    * sp/ - スマホ向けのスクリプト
    * (desktop|sp)共通で存在してるディレクトリ（後述）
        * entries/ - 変換対象になるエントリポイントファイル（ページ単位）
        * modules/ - entriesから呼び出される想定のモジュール
    * lib/ - 共通ライブラリなど
    * override/ - buildへ結果を吐くのではなく、既存ファイルの置き換えとするファイル（後述）
    * legacy/ - 長期的に使いたいつもりはないが、場当たり的になんとかするファイルの置き場（後述）
* web/ - publicなディレクトリ
    * build/ - 生成結果吐き出し先ファイル
    * XXX.js - 基本的には手運用している古いファイル。たまに`override`ディレクトリから上書きされたものがある（後述）

ディレクトリ構成とあわせて、bundleには3種類の方法を扱う形になりました。
下記それぞれの方法について説明していきます。

### 1. entries/modulesディレクトリ (モジュール管理バンドル)

はじめに説明するのは、一番わかりやすい標準的なバンドル。
モジュールを分解・結合して管理する、いわゆる普通のwebpackでの使い方が出来る部分。

ここを構成するにあたって、JavaScriptファイルの種類を`entries`と`modules`という２つの概念で分離することにしました。
それぞれ次のような分類になります。

* entries
    * webpackのコンパイル対象となる部分
    * ロジックは一切書かず、moduleの呼び出しだけを行う
    * 基本原則として、「読み込まれるページの種類単位」で区切る
* modules
    * モジュール単位のファイル
    * ロジックを記載する
    * モジュール同士で互いに呼び出しを行ったり、`lib/`のモジュールを参照する

この２つを利用し、ビルドはだいたいこんな具合で行うようにしました。

```js

// entiresファイルを抜き出す
const entries = glob.sync("**/entries/**/*.js", {cwd: baseDir}).reduce( (entries, file) => {
  entries[file] = baseDir + file
  return entries
}, {})

const webpackConfig = {
  entry: buildEntries,
  output: {
    filename: '[name]',
    path: './web/js/build',
  }
  　 :
  　 :
}
```

例として、下記のようなコードを想定してみます。

```html
<script src="some/lecagy/foo.js"></script>
<script src="some/lecagy/foo2.js"></script>
<script src="some/lecagy/foo3.js"></script>
```

```foo.js
$(document).ready(function () {
  // 何らかの処理
}
```

これらを下記のように書き換えていきました。

```html
<script src="some/bundle/fooPage.js"></script>
```

```modules/foo.js
export default function () {
  $(document).ready(function () {
    // 何らかの処理
  }
}
```

```entries/fooPage.js
import fooFunctionHook from '../modules/company/foo.js'
import bazFunctionHook from '../modules/company/baz.js'

fooFunctionHook()
bazFunctionHook()
```

置き換えフローは下記のような形に落ち着いています。

* 「このページ拡張したいな」となったところから置き換えれる仕組みにする。
    * 動作確認など考えると、ページ単位でのまとめるのが一番無難であろうと判断。
* レガシーコードを　`function()`で囲んでexportしていく。
* entriesで、それをHookする形。
* 仮に一箇所のページでしか利用してないmoduleでも、entriesにはロジックや具体的な実装を書かずに、moduleに持っていってhookする

### 2. overrideディレクトリ（既存コード上書きバンドル）

全体的な基本指針としては「entries/modulesで置き換えて行きたい」としつつ、どうしても呼び出し箇所が多いなど、置換しようにも足を引っ張ってしまうファイルが出てきてしまいました。

この影響範囲が大きくなってしまうのを防ぐために、既存コードを上書きする方式のバンドル方法を併用する事にしました。

`override`という、`entries`とは別途な部分を用意し、modulesを利用して既存ファイルを再現する形です。

```override/some.js
import someModule from '../desktop/modules/someModule'

// someModuleを利用して既存コードを再現する

```

webpack側も`web/js`へ直接吐き出して上書きする形になりました。

```js

const overrideBundle = glob.sync("./override**/*.js", {cwd: baseDir }).reduce( (entries, file) => {
  entries[file] = path.resolve(overrideDir , file)
  return entries
}, {})

const webpackConfig = {
  entry: overrideBundle,
  output: {
    filename: '[name]',
    path: "./web/js", // web/js/buildではなく、web/jsへ直接上書きする
  }
}
```

当然これを行ってしまうことで、ビルド済みファイルと、そうでないファイルが入り混じってしまいます。
そのため、あくまでもワークアラウンドのための「必要最低限に留めるもの」として運用するようにしています。

当初はこの`overrdie`を主軸とする方針に振り切り「既存コードを全て一回変換してしまう」というのも検討したのですが、今回は下記の観点でボツとなりました。

* バラバラのコードが単にcompileされている状態はそんなに旨味が無い
* require関連とかなんともしようがなくなりそうで結構辛い
* 無駄にファイルサイズが肥大化するだけで終わりそう
* 確認無しで意図せぬ変換があるのはちょっと許容しがたい
    * 逆に全部確認するのも、量的にきつい


### 3. legacyディレクトリ

最後に、`legacy`というバンドルを説明していきます。

「上記の`entries`と`override`だけではなんともしづらい」という「その他」のような部分です。

entryのキーに配列を利用することで複数ファイルをまとめるテクニックなどを使っています。
（entryの値の使い分けは[こちらの記事](http://qiita.com/chuck0523/items/caacbf4137642cb175ec#3-entry--文字列-vs-配列-vs-オブジェクト)などでまとめられています。詳しくはそちらを参照）。

```js
const legacyBundle = {
  "legacy/util.js": "./assets/javascripts/legacy/oldLib/util.js"
  "legacy/jquery.fn.js" : glob.sync("./assets/javascripts/legacy/jquery.fn/**/*.js"), // 1ファイルに対して配列を入れる
}

const webpackConfig = {
  entry: legacyBundle,
  output: {
    filename: '[name]',
    path: './web/js/build',
  }
}
```

#### legacy: windowグローバルに扱っているファイルの取りまとめ

古いコードの中には、prototype拡張形式で記述されたコードがいくつか存在していました。
これらは共通コードとして様々なスクリプトから利用されていたので、「モジュールとして読み込める ＆ webpackへ置き換え前のスクリプトからも呼び出せる形」を再現する必要がありました。

元々存在していたコードはこのような形でした。

```util.js
var SomeLib = function () {
 :
}
SomeLib.prototype.foo = function () {
 :
}

var BeeLib = function () {
 :
}
BeeLib.prototype.boo = function () {
 :
}

```

これを下記のような感じで分解していきました。
（ちなみに、上記は`class`構文で書き直せば更にきれいになるのだが、殆どのコードが後々捨てる事になりそうだったのでそこまではやらないことにしました）

```someLib.js
// SomeLibは古いコードからそのまま持ってくる
var SomeLib = function () {
 :
}
SomeLib.prototype.foo = function () {
 :
}

// 最後にexportだけする
export default SomeLib
```

module化はここまでで完了。そこから今度はglobalから呼び出せるための互換処理の部分を作成していきます。

この再現方法として、あまり良いやり方では無いのですが、windowオブジェクトに独自オブジェクトを追加する手法で再現することにしました。
考え方は`entires`と`modules`でやったことと一緒で、この部分ではwindowに対して割り当てた形です。

```util.js
import SomeLib from './someLib'
import BeeLib from './beeLib'

window.SomeLib = SomeLib
window.BeeLib = BeeLib
```

このようなことを行う場合、webpackでは`externals`や`library`などの機能が提供されております。
実際このあたりも試したのですが、自分たちだけが内部で取り扱う程度の要件の上では若干オーバースペックだなと感じました。

#### legacy: jQuery.functionの取りまとめ

globalオブジェクトと類似した話として、`jQuery.function`の拡張も`legacy`として特殊に処理しました。

下記のように書かれたjQuery.functionのファイルをwebpackのentriesにarrayとして渡す事で、ごっそり結合だけさせて吐き出す仕組みでうまく行きました。

```jquery.fn.fuga.js
$.fn.fuga = function(){
  console.log("This is fuga")
}
```

最初は若干「webpack内から`$.fn`で設定してもちゃんと動くだろうか？」という不安もあったものの、単にconcatしているのと変わらないこともあり、問題無く解決出来ました。

## 既存ライブラリとの付き合い方

今回扱っている環境に対して、それぞれライブラリとの付き合い方について下記に述べていきます

### jQueryとの付き合い方

昨今、jQuery脱却が叫ばれている時代ですが、とはいえレガシー環境についてで言えば、まだまだしばらくは付き合っていくことになるだろうと感じています。
（一方、jQuery依存も不要な箇所が増えてきているので「es6記法にしていく、npmパッケージに代替していく」という方向で徐々に減らしていく方向は粛々と進めていければなとも画策中）

管理方法に関しては、前述通りモジュール扱いせずに素のファイルとして管理することにしました。
完全にwebpackから切り離した別物として読み込みする形です。

最初はモジュールとして混ぜ込めないかと頑張ったものの、下記のような具合で色々と難がありました。

* npm上ではjQuery 1.8.3など、1.9.0より古いバージョンの場合、depreactedな扱いになっている。
    * npm install時に警告される。
* 残念ながら1.7.2が入っており、大規模に変更されている1.9以上にアップグレードするのはかなり動作確認負荷が高かった。
    * 1.8までは頑張ってみたが、それでもかなりハマりどころが多かった。
* 部分的にjQuery 3を使っていく手もあったが、ちょっと混乱の元になりそうだったのでこの方向もひとまず保留することにした。

`entries: ["/vendor/jQuery.min.js", "/legacy/jQuery.some.fuction.js"]`のような形でjQuery.functionと一緒にビルドする事など検討していますが、初期段階では後回しにしました。

また、jQueryに対しては併せて`ProvidePlugin`と`externals`も利用することにしました。

単純にjQueryをglobalに扱うだけならこの設定は不要なものです。
長期的にはjQueryのバージョンアップをモジュールとして扱ってやっていければ良いかなという観点で付与する方向にしました（これはちょっとYAGNIだったかなという反省もあります）

```js
const webpackConfig = {
    :
  plugins: [
    new webpack.ProvidePlugin({
      '$':          'jquery',
      '$j':         'jquery',
      'jQuery':     'jquery',
    }),
  ],
  externals: {
    "jquery": "jQuery",
  },
    :
}
```

これを挟むことで、クロージャでjQuery呼び出し部分を囲んでくれるようになります。

例えば下記のようなファイルがあった場合を考えます。

```js
const $jquery = require('jquery')

module.exports = function(){
  $(function(){
    $("#baz").click(function(){ console.log("$.click") })
    jQuery("#foo").click(function(){ console.log("jQuery.click") })
    $jquery("#bla").click(function(){ console.log("jquery with require") })
  })
}
```

このファイルが下記のように変換されます。

```build.js
/***/ function(module, exports) {
	module.exports = jQuery;
/***/ },
 :
/***/ function(module, exports, __webpack_require__) {
	/* WEBPACK VAR INJECTION */(function($, jQuery) {
	const $jquery = __webpack_require__(2)

	module.exports = function(){
	  $(function(){
	    $("#baz").click(function(){ console.log("$.click") })
	    jQuery("#foo").click(function(){ console.log("jQuery.click") })
	    $jquery("#bla").click(function(){ console.log("jquery with require") })
	  })
	}
	/* WEBPACK VAR INJECTION */}.call(exports, __webpack_require__(2), __webpack_require__(2)))
```

上記のように、`require`から呼び出した`$jquery`という関数も、`$`、`jQuery`という宣言が無かったファイルもrequire読み込みとして扱ってくるようになりました。

現状では`$`だったり`$j`だったり`jQuery`だったりと、色々な呼び出し方をglobalからしている部分を除々に引き剥がして行ければ良いかなと目論んでいます。

### Prototype.jsとの付き合い方

我々のレガシーコードには、jQuery以前に勢力を持っていた[Prototype.js](http://prototypejs.org/)というライブラリを利用している箇所が幾つかありました。

こちらは標準的な`String`や`Array`を`prototype`拡張して機能提供をしているライブラリです。
これはさすがにbabelとの組み合わせることを考えた場合に、「もはや仲良くしていくのは難しい」と判断して依存を削除することにしました。

実際に問題があるか細かく検証した訳ではありませんが、古めのPrototype.jsに存在する`Array.reduce`など[core-js](https://github.com/zloirock/core-js)のPolyfillと衝突して問題を起こす可能性は高いと考えられます。([参考記事](http://qiita.com/jwhaco/items/b5b474883d3020f6dc5f))

Prototype.js自体のバージョンアップは依存を削除するより難易度が高いので検討対処から除外しました。

依存を削除する方法は下記のパターンがありました。
利用箇所ごとに適切な方法を選ぶようにしました。

* 使ってなかったものはガンガン捨てる
    * 古いコードがそのまま放置され、アクセスがほぼ無い箇所など
    * 読み込まれてないコードがそのまま放置されているパターンもあった
* マイクロサービス的にサブプロジェクトとして切り出して大胆に置き換え
    * コストは高いが、ビジネス的に重要な箇所などはトータルで言うとそのぐらい価値になると判断出来た
    * 別サービスではまっさらな状態からReactなどライブラリを採用して別途開発
* jQueryへ気合で置き換え
    * 辛さで言うと一番つらいパターン
    * 幸い利用しているPrototype.jsの機能がそんなに幅広くも無かったので、ある程度パターンで置き換えていった。
    * サーバーサイドで`?no-prototype-js=1`のような値を渡すとPrototype.jsを読み込まない仕組みを作ったりして動作確認をした。

この記事上で最も辛かった箇所はここでした。
これに関してはリソースを捻出する事が出来たチームと、依存削除の作業に尽力してくれたメンバーに感謝しかありません。

## ESLint

今や定番となったESLintも取り入れました。
ただし、基本的にコードを揃える目的よりも、possible errorであるルールを重視しての導入でした。
その為、ここもあまり頑張りすぎない設定にして、徐々に厳しくしていこうと考えています

* eslint:recommendのみをベースにする。
* 古いコードの散財するwebディレクトリと新しいコードの多いassetsディレクトリはルールを別で用意。
* 新しいコードの多いassetsディレクトリも、コード移植のタイミングで下手にいじってしまうことを避ける。
    * [`// eslint-disable-next-line`や`// eslint-disable-line`](http://eslint.org/docs/user-guide/configuring#disabling-rules-with-inline-comments)などを利用して、部分的に無視していく
    * 「この記法は真似しないでね」という合図になるので結構これだけでも悪くない。
    * 「なぜ`disable`にしていくか？」についてもなるべく記載していくようにしている。
* `globals`は有効活用して、自分たち固有のglobalなどを追加してく。ある程度global依存しちゃっている部分は仕方がないと割り切る。
* ある程度置き換えが落ち着いたタイミングから、少しずつ`--fix`をつけて徐々にコードを整えていく。fixalbeなルールは加えやすい
    * そうでないルールはあんまりやりすぎると疲弊してしまうので、そこらへんは頑張りすぎない。
* eslint自体はwebpackのフローには組み込まず、別途CI上で実行。
    * eslint-loaderを組み込むのも考えたものの、若干そこは後手に回っている

また、`override`で作成したビルド後のファイルをlintしてしまわないように、`BannerPlugin`を利用して先頭に`eslint-disable`のコメントを差し込んでいます。

```js
plugins: [
 　new webpack.BannerPlugin("/*eslint-disable */", {raw: true}),
]
```

## ビルドプロセス・生成物をどう管理するか？

フロントエンドをビルドする際、障壁になることの一つに「ビルドをどのタイミングで行うか」ということが上げられます

これに関しては下記が主に考えられるのでは無いでしょうか。

* 本番サーバーでbuildして生成
* 手元サーバーでbuildして、生成物をGitに含めて管理
* CI / CD サーバーでbuildして本番サーバーに配布

この中で、我々は現状 **手元サーバーでbuildして、生成物をGitに含めて管理** という手法を採用しました。

当然、ベストではないと感じています。
とはいえそれ以外の方法ではコストが合わないと感じる部分がありました。

* 本番サーバーでbuildして生成（不採用）
    * Productionサーバーにnodeを入れて実行すれば良い。
        * オンプレ環境とかだとちょっと面倒見るの辛い
    * 何ら本番サーバーでだけビルド失敗があった場合などに結構辛そう
        * 更に失敗が特定の１台のみとかnpmのキャッシュ周りとかだと死にたくなりそう
    * 色々将来的な不安を考えると、ちょっと自分たちには不都合だった。
* 手元サーバーでbuildして、生成物をGitに含めて管理（採用）
    * 完全に妥協策。
    * 既存のJavaScriptがGitコミットされている状況と同等なので、導入コスト自体は低い。
        * gulp + scssが既にこの運用だったのでそこに乗っかったというのもある。
    * 今回`override/`というあたりディレクトリを使ったために、.gitignoreでの対象管理が若干面倒だった。
    * 運用コストは少なからず上がる。
    * ビルド漏れや、個人環境差異によるズレの対処が必要
        * ある程度なんかあったらフロントエンドエンジニアが「マシン見ます」とか「こっちで再ビルドしておきます」とケツを持つ覚悟は持つ
    * ビルド生成物は各個人によってずれうるので、ズレを防止する施策が必要
        * `npm shrinkwrap`や`yarn`による固定
            * yarnの場合[`yarn check --integrity`](https://yarnpkg.com/en/docs/cli/check#toc-yarn-check-integrity)というコマンドがあるので、多分これも有益そうと見ている。
        * `git diff --exit-code`で差分チェックだけCIに行わせるという手も使っている。たまにずれて面倒な感じもあるが、手軽でポカ避けにはちょうどいい。
    * レガシーな環境だと、そこまでJavaScriptの変更自体が活発というわけでもないので結構なんとかなっているという部分が大いにある。
        * 頻度が高くなったら辛くなりやすいと想像出来る
* CI / CD サーバーでbuildして本番サーバーに配布（将来的にやりたい）
    * 一番モダンで安定したやり方。目指したい方向
    * CIサーバーとデプロイプロセスの連携が必要で、自分たちの場合結構導入コストが高かった
    * 余力があったらこっちに切り替えていきたい
    * CIサーバーでビルドして、botにプルリクさせるというのも考えたが、これもこれで面倒が多そうでやめた

## おまけ：configファイル

### webpack.config.js

参考までに、webpack.configを晒していきます。
利用当初はgruntを彷彿とさせる記法に「ちゃんと管理できるだろうか？」と不安もありましたが、今のところまだ見通せるレベルだなと感じています。

```js
// webpack 1.x向け
const webpack = require('webpack')
const glob = require('glob')
const path = require("path")
const baseDir = './assets/javascripts/'

const buildEntries = glob.sync("**/entries/**/*.js", {cwd: baseDir} ).reduce( (entries, file) => {
  entries[file] = baseDir + file
  return entries
}, {})

const overrideBundle = glob.sync("./override**/*.js", {cwd: baseDir }).reduce( (entries, file) => {
  entries[file] = path.resolve(overrideDir , file)
  return entries
}, {})

const legacyBundle = {
  "legacy/jquery.fn.js" : glob.sync("./assets/javascripts/legacy/jquery.fn/**/*.js"), // 1ファイルに対して配列を入れる
  "legacy/util.js": "./assets/javascripts/legacy/oldLib/util.js"
}

const eslintDisableComment = "/*eslint-disable */"

// 共通設定
const defaultConfig = {
  plugins: [
    new webpack.ProvidePlugin({
      '$':          'jquery',
      '$j':         'jquery',
      'jQuery':     'jquery',
    }),
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false
      }
    }),
    new webpack.BannerPlugin(eslintDisableComment, {raw: true}),
  ],
  externals: {
    "jquery": "jQuery",
  },
  module: {
    loaders: [ {
      test: /\.js$/,
      loader: 'babel',
      exclude: /(node_modules)/,
    } ]
  },
  devtool: '#hidden-source-map',
}

module.exports = [ Object.assign({}, defaultConfig, {
  entry: buildEntries,
  output: {
    filename: '[name]',
    path: './web/js/build',
    sourceMapFilename: "[file].map"
  }
}), Object.assign({}, defaultConfig, {
  entry: legacyBundle,
  output: {
    filename: '[name]',
    path: './web/js/build',
    sourceMapFilename: "[file].map"
  }
}), Object.assign({}, defaultConfig, {
  // overrideディレクトリ下のファイル群だけは、書き出し先を変える
  entry: overrideBundle,
  output: {
    filename: '[name]',
    path: "./web/js",
  }
}) ]
```

ここまで説明を割愛している部分も上記には含まれていますので、ザッと色々書いてみます。

* UglifyJsPluginの警告はレガシーには辛すぎるので抑止している。そのへんはESLintに任せていきたい。
* dev用のビルドと本番用のビルドは変えないようにしている。
    * sourcemapは`hidden-source-map`にしてHTTP Headerから呼び出しするようにしている。
        * 詳しくは[以前書いた記事](http://qiita.com/inuscript/items/eec749c3ba00eb8ac693)でまとめているので、興味があればご参照を
* webpackでよく使われる下記はまだ利用していない
    * `DefinePlugin`
        * NODE_ENVなどの埋め込みするプラグイン
        * 今のところ利用箇所が特に無いので無理して入れてない。Reactなど入れるようになったらきっと利用する
    * `AssetsPlugin`
        * digest生成
        * 幸いサーバーサイド側でdigestを管理する機能が実装されているので、無理せずそれをそのまま利用している。
    * `CommonsChunkPlugin`
        * ファイル分割の自動化的な部分はまだやっていない。
        * ページ単位で、そんなに大きなライブラリも使っていないため、実際手動で分割しているというのに近い
        * とはいえ最近がっつりライブラリ入れたりしているので、ぼちぼちなんとかしたほうがいいとは感じたりしている。

### `package.json`のscripts

多少ビルド周りで小細工しているので、こちらも載せてみます。

```js
"scripts": {
     :
    "build": "npm-run-all --parallel build:*",
    "build:css": "gulp build",
    "build:js": "webpack -c --progress",
    "watch": "npm-run-all --parallel watch:*",
    "watch:css": "gulp build watch",
    "watch:js": "webpack --watch --progress",
    "lint": "eslint web/ assets/ --rulesdir scripts/eslint-rules/rules --fix",
    "lint:strict": "eslint web/ assets/ --rulesdir scripts/eslint-rules/rules",
     :
}
```

序盤で述べた通り、css関連は無理せずgulpのままとしているので、[npm-run-all](https://www.npmjs.com/package/npm-run-all)で並列処理するようにしています。

# 総括

ここまで自分たちが進めてきたことをまとめてきました。

当然開発リソースを割いてしまうし、色々処理に気を使ったりその後の運用上ではビルド処理が発生してしまうという負荷も発生する所だと思います。

それでもやってみて利点は少なくなかったなと感じています。

* Moduleを分割管理して保守性を上げれた
    * なんなら部分的にならテストもできそう
* `Template String`や`arrow function`など使えるだけで学習コスト下げれた
    * thisでのハマりとかでハマるメンバーや、`(function($){ /* some*/})(jQuery)` みたいなおまじないを説明する機会が減った
* [lodash](https://lodash.com/)、[js-cookie](https://github.com/js-cookie/js-cookie)、[platform.js](https://github.com/bestiejs/platform.js)など、レガシーでも併用しやすく有益なライブラリが結構使える
* ソースにコメントをちゃんと書いて保守性を上げれる環境が作れた
    * それまでは生ファイルだとユーザーに見えるので、妙にコメントを記載するのに躊躇してしまう事が多かった。

また、ここまで土台を整えるという所にフォーカスして進めてきたので、まだまだやり残したと感じる部分があります。
このへんは、来年ぼちぼち取り組みたいと考えています

* 全ファイルのモダン化。`override`、`legacy`など、場当たり的に凌いだディレクトリの廃止
* テスト + CI
* 一端保留したjQueryのモジュール化、バージョンアップ
* デプロイ周りの改善
* webpack2化
* webapck-dev-server / hot-reload など、webpackを生かした機能
* yarn化してinstall高速化、依存バージョン固定の安定化

2017年も楽しくフロントエンドと付き合っていきたいものです
