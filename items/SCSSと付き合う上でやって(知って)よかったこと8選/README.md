
なんだかノウハウ溜まってきた感あるのでまとめる。

# 1. prefixの付与はautoprefixerに任せる方が良かった
基本的な部分だが、SCSSになんでもかんでもやらせようとするのはよくない。

例えば下記のようなprefix

```css
div {
    -webkit-box-shadow: 0px 0px 10px rgba(255,0,0,.5);
    -moz-box-shadow: 0px 0px 10px rgba(255,0,0,.5);
    box-shadow: 0px 0px 10px rgba(255,0,0,.5);
}
```
sass/scssであれば[Compass](http://compass-style.org/)、[Bourbon](http://bourbon.io/)、lessであれば[LESSPrefixer](http://lessprefixer.com/)など、mixinとしてprefix付与を提供してくれるライブラリがあったりするが、これよりもautoprefixerによる解決をおすすめしたい。

gulpなどを導入していない場合、autoprefixer導入には初期コストがちょっとかかるものの、運用コストのことまで考えると[autoprefixer](https://github.com/postcss/autoprefixer)を入れる方が圧倒的に楽だった。
（railsであれば[autoprefixer-rails](https://github.com/ai/autoprefixer-rails)というのもあるらしい。）

### (備考)autoprefixerとpostcss
[postcss](https://github.com/postcss/postcss)というscssのコンパイルした『後』に更にcssを変換するという考えのツールであり、autoprefixerにはその土台が利用されている。

流れとしては

```
sass -> css -> css (postcss後)
```
という変換の流れだ。

例えば下記のように、指定したブラウザで必要なprefixを自動的に付与してくれる。

```js
gulp.task('autoprefixer', function () {
    return gulp.src('./src/*.css')
        .pipe(postcss([ autoprefixer({ browsers: ['last 2 version'] }) ]))
        .pipe(gulp.dest('./dest'));
});
```

これを仕掛けておけばcssはprefixを付けない状態で何も気にせず書いておけば良い。

```こんな感じ.scss
div {
    box-shadow: 0px 0px 10px rgba(255,0,0,.5);
}
```

変換後にはprefixが自動で付与された状態になる。

[cssnext](http://cssnext.io/)もpostcssの一部として利用できるので、合わせて検討してもよいかもしれない

# 2. gulpを利用中ならgulp-ruby-sassよりnode-sassの方がオススメ
とにかくコンパイルが早い。機能も十分追従してきたのでオススメ。

詳細は過去にまとめた事があったので下記を参照のこと。
[gulpでruby-sassからnode-sassへ移行するときのハマりどころ](http://qiita.com/inuscript/items/8e91c825c525ad17539b)

# 3. 便利なサイト結構ある

便利なサイトまとめてみるか、と思ったら結構あった。
（かといって単発記事になる量じゃなかったので放出）

- __[Sassmeister](http://sassmeister.com/)__
	- sassのいわゆるplayground 
	- 「あれ？この挙動どうなるんだっけ？」って迷ったら使って見るとわかる。

- __[css2sass](http://css2sass.herokuapp.com/)__
	- cssをsassにしてくれるコンバータ。
	- 古くて大きいcssを手でscssにするのは大変なのでざっくりこれで変換する。便利。

- __[Sache](http://www.sache.in/)__
	- sassのライブラリが検索できる。
	- 直接使っても良いし、コード見るだけでも結構参考になる。

- __[Sass Guideline](http://sass-guidelin.es/)__
	- いわゆるガイドライン。
	- 色々参考になる情報がある。

- __[Can I Use?](http://caniuse.com/)__
	- どのブラウザがどんなプロパティやJavaScriptの機能などをサポートしているかを教えてくれる。
	-「このプロパティってどのぐらいのブラウザサポートしてたっけ」
みたいな事が知りたい時に重宝する。

- __[Sass Compability](http://sass-compatibility.github.io/)__
	- 上級者向け。ruby版のsassとlibsassの機能比較をしている。
	- 最近は[互換性維持](https://github.com/sass/libsass/wiki/The-LibSass-Compatibility-Plan)の計画をしているものの、まだ少々動かないものがあるので「おや？」となったら確認しに行くと良い。

# 4. ページ単位でscssを読み込み分けるのに良いやり方

昨今cssは最終的に1ファイルにconcatされるのが割りと今風だが、個別ページで読み込みたいようなクラスもある。
そういう時はこんな感じでやってみるとそれぞれ独立出来て良い。

```scss
body.foo-controller.baz-action{
	@import "path/to/component"
}

body.boo-controller.brr-action{
	@import "path/to/component2"
}

```

bodyにcontrollerやaction名を付与するやり方は下記などが参考になった。
http://qiita.com/naoty_k/items/d490b456e0f385942be8

# 5. グリッドシステムにSusyを利用すると良い
やっぱりグリッドシステムはほしいし、自作もしたくない。

sassにおいてグリッドしたいと、[Neat](http://neat.bourbon.io/)か[susy](http://susy.oddbird.net/
)のだいたい２つが筆頭候補になってくる。
どちらも試したが、susyの方が今風でよく出来ている印象があった。


こちらも過去に比較をまとめていたので詳細は下記。
http://qiita.com/suisho/items/8f54b7baeb61a3341c6d

node環境でビルドしているので、`npm susy`でインストールし、gulpのコンパイル時には`node_modules/susy`をインクルードパスとして含めている。

# 6. git上でのconflictが激しい場合は.gitattributesの設定を検討する

最終成果物のcssをgitに含めるかどうかも議論点ではあるが、gitに含めない運用をするにはサーバーでのprecompileが必要だったりとなかなか簡単にもいかない。
「だけどコミットしてるとsassがコンフリクトしまくって困るよ！」という場合は.gitattributesを利用して、コンパイル後のcssをbinary扱いしてしまう手がある

```.gitattirbute
*.css binary
```

ただし、当然意図せぬ衝突がなされてしまうことも予想されるので、取り扱いには十分注意。
CIなどでコンパイルチェックの機構を入れるなども併せて検討したほうが良いだろう。


# 7. Compassの導入は極力避ける

これは結構悩ましいが、個人的にはCompassを使わなかったことは正解かなと思っている。
理由は下記。

- ruby依存が強め
	- node-sassを使いたいときに若干移植しづらかった（一応node-sassはcompassもサポートはしている。僕は使ったことはないけど）
- 画像spriteなど独自なものを使ってしまうとCompassへの依存強くなりすぎて離れられない。
	- これが一番致命的。引っぺがすの結構辛い
- 巨大な割に実際そんなに使う関数多くない
	- autoprefixerでだいたいまかなえたり。自作で十分だったり
		- 一部だけマクロにほしかったらsacheで探せばいい感じもしている。
- ドキュメントが読みづらい
	- これは主観だけど、Bourbonとかに比べて結構読みづらさを感じる。


# 8. IE9以下セレクタ4096個以上無視される問題を回避するにはblesscssが良さそう

IE9以下では4096個以上のセレクタが無視される問題がある
https://www.softel.co.jp/blogs/tech/archives/3045

実際まだそこまでの規模に直面したことは無いので、「これが良さそう」ぐらいな温度感だが、[blesscss](http://blesscss.com/)を組み合わせるのが良さそう。
[gulpプラグイン](https://github.com/adam-lynch/gulp-bless)もあるようだ。

環境によっては「増えた場合に手動で読み込むcssを増やす」ということをしなければならないが、まあそう頻繁に起こることではないので今の所気にしていない。

