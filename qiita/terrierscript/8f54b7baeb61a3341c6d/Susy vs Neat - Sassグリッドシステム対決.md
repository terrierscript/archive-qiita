
sassでグリッドデザインをするときに有効なgrid frameworkの話。


## 今回比較するやつら
数値は2015/02/03時点

### [Susy](http://susy.oddbird.net/)
- sassの公式サイトでも使われている（らしい）
- すーじーと読むらしい。（ロゴあたりをhoverすると読みが出てくる）
- Star数: 2367

### [Neat](http://neat.bourbon.io/)
- Bourbonシリーズ
  - Bourbonは軽量ライブラリ
- throughbot製
- ドキュメントが丁寧
- Star数: 2632

## 今回除外したもの
### [Profound grid](http://www.profoundgrid.com/)
- サイトが黄色い。
- 最終更新が5ヶ月前とあまり更新が活発でなさげ
- Star数: 902

あまり活発でなさそうなので除外。

## 用語
用語みたいなものが複雑な感じでぶれそうなので
一旦このドキュメントでは下記のように定義しておく。

- column
  - 12分割などしたうちの1つ
- span
  - 複数のcolumnを含むブロック
- gutter
  - columとcolumの間の空白の隙間


## Installation
どちらもgruntなら

```Gruntfile.js
sass: {
  dist: {
    options: {
      style: 'expanded',
      require: ['susy', 'neat']
    },
    files: {
      'css/style.css': 'scss/style.scss'
    }
  }
}
```

とかlibsassのrequireで利用可能。

### Neat

```
gem install bourbon
gem install neat
bourbon install
neat install
```

これで素のsassファイルを管理していけば良いというのはシンプルで好き。

### Susy
bundle, compass, Bower, npmが用意されている模様。(npmはどう使うんだろう)

bourbonライクに素のsassを使うならgithubからzip落とすとかしかなさそう。

## 2 column gridの例

2カラムでの場合
どちらも吐だされるコードはわりと一緒。

### Neat

```neat.scss
@import "bourbon/bourbon";
@import "neat/neat";

.two{
  @include outer-container();
  .side{
    @include span-columns(4);
  }
  .main{
    @include span-columns(8);
  }
}
```
- 最後のcolumnに対し`:last-child`で打ち消しをしているため、susyの`last`のように付与しなくてもよいのは魅力
- IE8ではちょっと残念に（後述）

### Susy

```susy.scss
@import "susy/susy";
$susy: ( 
  columns: 12, // 絶対に設定が必要。最初ここではまった。
);

.two{
  @include container();
  .side{
    @include span(4);
  }
  .main{
    @include span(8 last);
  }
}
```
- 最終要素に`last`とつけるのはちょっとださいかもとも。
- これをつけると`float:right`になるので、右左を入れ替えたいとかいう時は楽ちん（邪道）

2015/02/05追記：

```
$susy : (
  columns : 12,
  gutter-position : split
)
```
とすればgutterが左右均等につくのでlastなどを書かなくて良くなった。

## タイル状に並ぶパターン
結構これはsusyのgalleryありきで比較してしまっているけど頻出パターンなので。

### Neat

```neat.scss
.items{
  @include outer-container();
  .item{
    @include span-columns(4);
    @include omega(3n); // 12 / 4 で 3
  }
}
```
- 個数を変えたくなったときに`span-columns`と`omega`をどっちも書き換えないといえないのはちょっとだるい。
- `nth-child(3n)`に対してgutterを消すようなstyleが出力される

### Susy

```susy.scss
.items{
  @include container();
  .item{
    @include gallery(4); // 4カラムずつ
  }
}
```
- galleryはだいぶイケてる機能
- ただsusyは`nth-child(3n)`,`nth-child(3n+1)`,`nth-child(3n+2)`..と全部作る。
- IE8では（後述）

## IE8対応
やりたくなくてもやらなきゃいけないこの対応。

### Neat
サポートブラウザについては言及済み。https://github.com/thoughtbot/neat#browser-support
[selectivizr](http://selectivizr.com/)を使えば対応できる、と書いてあるものの、bourbonがIE8の対応を切っているため実際どうなんだろう？という所。

基本的なgridも

```neat.scss
.two{
  @include outer-container();
  .side{
    @include span-columns(4);
  }
  .main{
    @include span-columns(8);
    @include omega(); // for IE8
  }
}
```
このように`omega()`をつけないとダメっぽい。
これだとsusyと似た感じに（または記述が長いのでより悪いかも）

参考 : https://github.com/thoughtbot/neat/issues/165

### Susy
サポートブラウザについても明言は見当たらない。
TODOには`Add IE support to new syntax.`というのが見当たる。
またissueを見ているとNeatよりは答えてくれている印象。

galleryについてはgutterが綺麗にいかない。
ちょっとトリッキーだが

```susy.scss
.items{
  @include container();
  .item{
    // @include gallery(4);
    @include span(4 inside);
  }
}
```
とすることで等間隔のブロックを作る事ができる。
`inside`はgutterをspanの中に内包してしまうやり方。まあこれだと`width : 33%`とほぼ等価かもしれないけど・・・

参考：

- http://stackoverflow.com/questions/24639512/susy-2-gallery-mixin-is-incompatible-with-ie8-workaround
- https://github.com/ericam/susy/issues/363

## config変数

### Neat
ずらずらっと変数で記載。

```neat.scss
$column: 90px;
$gutter: 30px;
$grid-columns: 10;
$max-width: em(1088);
```

途中で潰せるように`@mixin reset-all()`なんていうのもあったけどなんだか危なっかしいイメージ。

### Susy
mapで記載。ナウい。

```susy.scss
$susy: (
  columns: 12,
  gutters: 1/4,
  math: fluid,
  output: float,
  gutter-position: inside,
);
```
スコープ毎に設定する仕組みっぽい。
しかし結構色々変数がありすぎてちょっと理解に時間かかりそう

## responsive
あんまり必要に迫られてないのでちゃんと調べてない

### Neat

```
$mobile: new-breakpoint(max-width 480px 4);

@include media($mobile) {
  @include span-columns(4);
}
```
breakpointというmobileとそうじゃないときの分岐点を決めて`@include media`で分岐部分を記載する。

### Susy

```
// mobile 
@include susy-breakpoint(0 480px, 4) {
  .example { @include span(3); }
}

```
breakpointという考えはだいたい一緒。ちょっとmixinの入力が値がひとつだと`min-width`扱いされるようなので注意。
## その他

### Susyのfunction
susyは幅計算などを関数単体として呼べるようになっている。

```
.item {
  width: span(2);
}
```
APIを見てもだいぶ低レイヤーなことまで出来るイメージ。

## 雑感
- なんとなくsusy寄りになってしまっているなあと思うけどsusyは確かに悪くないかなあと思った。
  - 結構[NeatからSusyにした](http://www.sitepoint.com/sass-grids-neat-susy/)みたいな記事などある。
- ドキュメントは個人的にはneatのほうが断然わかりやすかった
  - Neatのほうがコンパクトというのもあるだろう。
- IE8まで考えるとsusyのほうがよさそうかもという体感。
  - issue見ていてもbourbn勢はIE8に冷たい
