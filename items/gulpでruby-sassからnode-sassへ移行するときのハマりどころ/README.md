 
[node-sass 3.0](https://github.com/sass/node-sass/releases/tag/v3.0.0)が出て、Libsassも3.2に上がった。これによって@at-rootなど（個人的に）重要な機能が使えるようになったので移行した。
 
互換性に関しては[sass-compability](http://sass-compatibility.github.io/)が有用。

 
# なんでlibsassにしたいのか？
- gulp-sassのコンパイルが超早い。
	- 今回の環境では100ファイルぐらいで6〜8秒ぐらいなのが1〜2秒ぐらいになった。
- gulp-ruby-sassの独特な記法から逃れられる
- rubyを別途依存させる必要がない

とかあたり。

# ハマりどころ
互換性が担保されているとはいえ、ちょいちょい引っ掛かりがあった。

## 不正な色コードはエラーになる
 
今まではカラーコードが不正でも無視してくれていたが、libsassでは怒られてしまう。まあこれは普通になおすかそもそも効いてなかったので消せばよかった。
 
```scss
.foo{
  color: #12345
}
```
 
## IEのexpression構文がコケる
IEのexpression構文がどうもコケる。現在node-sassはlibsass3.2.2を利用しているようだが、[3.2.4では改善されるらしい。](https://github.com/sass/libsass/issues/1102)
 
```scss
html { 
  filter: expression(document.execCommand("BackgroundImageCache",false, true));
}
```
が、わりと古い対応だったので削った。
 
本気で対応したい場合はJavaScriptで書きなおしちゃうのが筋として良いかもしれない。
 
 
## @at-rootでの#{&}が不正なエラーになる。
 
これは一番変なハマり方した。
`@at-root`と`#{&}`の組み合わせでエラーになるようだ。
エラーメッセージも`unknown`となってしまってなかなか発見に時間がかかった。
 
```scss
@mixin box-sizing-border{
  @at-root{
    #{&}{
      box-sizing: border-box;
 
      *, *:before, *:after {
        box-sizing: inherit;
      }
    }
  }
}
```
 
この問題は一時変数に&を入れることで解決。
 
```scss
@mixin box-sizing-border{
  @at-root{
    $selector: &
    #{$selector}{
      box-sizing: border-box;
 
      *, *:before, *:after {
        box-sizing: inherit;
      }
    }
  }
}
```
 
完全にバグを踏んだ気がするので、[issue](https://github.com/sass/libsass/issues/1204)を報告してみたらこれも3.2.4で動くようになるから待っててねという感じっぽい。
 
## calcの中で`-o-calc`が動かない
`-o-calc(100% - 16px)`だと%とpxで計算しているぞと怒られてしまう。
 
おそらく[-ms-calcの対応issue](https://github.com/sass/libsass/pull/900)があるので、同様に`-o-calc`も入れなければなさそう。

今回の場合は別途で既に[autoprefixer](https://github.com/postcss/autoprefixer)を入れていて、その消し忘れだったのでさくっと消した。
おそらくそうでなくてもautopreifxerを使うようにシフトするのが良いだろうと思う。

 
## 相対パスが効かない？(エラーが出ない）
これはちょっと深くは未検証。pathオプションなどの問題かもしれない
 
だいたいこんな構成だったとして
 
```
sass
├── _basic.scss
└── application.scss
```
 
```application.scss
// 読み込まれない（エラー出ない）
@import "basic";			
 
// 読み込まれる
@import "sass/basic";

```
 
というように相対パスなら解決出来そうな（ruby-sassだとできていた）ものが読み込まれなくなっていた。
 
エラーも無かったのでdiffでやっと気づいた。あぶないあぶない。
