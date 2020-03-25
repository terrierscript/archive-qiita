
[Styledown](https://github.com/styledown/styledown)についての話。あんまり日本語記事がなさそうなので

# デザインスタイルガイド
存在自体は割りとメジャー感出てきた気がするデザインのスタイルガイド。
生成ツールとして有名どころはだいたい下記な感じではなかろうか

- KSS
- Kalei
- Styledocco

今まで自分の体験ではKSSをがっつり使ってみたりStyledoccoはちょっと触ったりとか。
でもkssはkss-railsがどうもイマイチだったりで再度使うのもなあ、と思ったりで新しくスタイルガイドを作る上で再選定を考えて[The styleguide guide](http://vinspee.me/style-guide-guide/)とかでstyledownというのにぶちあたった。


特徴としてはこんな感じ

- Markdownで記載
- CSSのコメントとしても書けるし分離したmarkdownファイルでも書ける
	- 個人的には分離した.mdファイルがおすすめ（理由は後述）
- サンプルのHTMLコードに **jade** が使える
	- これかなり魅力。サンプル書くのがすごい楽。
- headタグ部分やレイアウトもmarkdwon + jadeで記載

# Usage
すげーシンプルで[公式](https://github.com/styledown/styledown)見ればだいたいわかるのでざっくり

#### cssコメントモードでの記法

ほぼ公式通りですがこんなん。styledoccoチックなものなんで特に何がという感じでもないですね。

```compoennt.css
/**
 * コンポーネント名:
 * `.button` - このへんマークダウンで記載
 *
 *     @example
 *     .button
 *       h3 Sample code
 *       p goes here
 */

.your-component-here {
  display: block;
  ...
}
```

こんな感じだけど雑感を下記に

- 結構コメントの記述がだいぶシビアで、上記のようなちゃんとしたブロックコメントでないと今のところダメらしい。
	- なのでこのモードを使うならstyledoccoとかでいいかもと思ってしまう。
	- この問題は[Issue](https://github.com/styledown/styledown/issues/3#issuecomment-62667881
)としては上がっているのでいつか解決されることを期待したいところ。
- 一応ソースを見るとcss,　scss, jadeあたりは上記で書ける模様。

#### markdownモード

個人的にはこっちのほうが好き。

```compoennt.md
### Button
`.button` - ボタンです 

    @example
	a.button Button
	a.button.primary Button
```

罠としてはh1にあたる`# hoge`のような記載をするとその下から全部表示されなくなること。
configとして認識される模様？。

こっちをメインで使ってみると、実はこいつの比較対象はKSSやstyledoccoなどではなく[Styleguide Boilerplate](http://bjankord.github.io/Style-Guide-Boilerplate/)、または[Assemble](http://assemble.io/)のような静的サイトジェネレーターみたいなものなんじゃないかと思う。

#### build

公式に乗っているのはこんな感じ。

```sh
$ styledown css/*.css css/config.md > styleguides.html
```

ちょっとこれしばらくわかっていなかったんだけどこういうことらしい。

```sh
$ styledown 対象ファイル configファイル > 出力
```

最後のパラメータがconfigであるというわけではなく、読み込んだものの最後の`h1`ブロックがconfigとして使われているよう。

#### configファイル

レイアウトファイルという言い方のほうがイメージつきやすいかも。

```sh
$ styledown --conf > config.md
```

で生成するデフォルト設定が生成できる。
こなれてくるまではデフォルトでよさそう。

### gulp / grunt

どちらもある。けどまだまだstar数とか少ないしまだまだマイナー感高い。

- https://github.com/drakeh/grunt-styledown
- https://github.com/st44100/gulp-styledown

# まとめ
- 全体的にシンプルで好き
- markdown + jade でさくさく書けるのとてもいい
- cssに埋め込む記法はだいぶ融通が効かなくて今はだいぶ辛い
- 個人的には上記の理由でmarkdownファイルに別途記載するやり方がおすすめ
- KSSやstyledoccoとの比較というより[Assemble](http://assemble.io/)のような静的サイトジェネレーターの特化版とかんがえると良い。
- config.mdをいじれば色々できる
