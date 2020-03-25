
ここ数日、[rails-assetsは終わった](https://github.com/rails-assets/rails-assets/issues/291)とかいやいや[bowerはまだ終わってない](http://bower.io/blog/2015/bower-alive-looking-contributors/)と騒がしいですが、そんな中、最近自分もbowerを最近捨てていたので[^1]そのあたりの話。

# どんな環境か？
- プロダクト全体ではまだbrowserifyなどは使えてない
	- 使いたいけどなかなかそこまで手が回らず・・・
	- browserifyとか使っているのであればbowerからの移行先は基本npmで良いだろう
- bowerはChart.jsとかちょっとしたライブラリを持ってくるのに使っていた。
- bower -> gulpでconcatして出力して管理していた。

このぐらいな環境であれば、jsdelivrはそれなりにおすすめできる。

# 結果：CDNで対応しよう

ある程度有名なライブラリばかりだったので、CDNで対応するのが楽そうだった。

CDNは色々あるが、[jsdelivr](https://www.jsdelivr.com/)を選定した。選定理由はこんな感じ

- ライブラリ数が十分に豊富
	- cdnjsより豊富な気がする（調べてないので体感）
	- jquery.sticky-kitとかはjsdelivrにしかなかった。
- [複数のファイルの同時呼び出し](https://github.com/jsdelivr/jsdelivr#load-multiple-files-with-single-http-request)に対応 __(重要)__
- [複数のCDNでBalanicing](https://www.jsdelivr.com/features/multi-cdn-load-balancing)しているらしい
	- これに関してはそこまでがっつり読み込んでない。

個人的には刺さったのが二番目。

例えばjQueryとjQuery UIを取りたいなら

```
https://cdn.jsdelivr.net/g/jquery@1.11.3,jquery.ui@1.11.4
```
こんな具合。
形式としては

```
https://cdn.jsdelivr.net/g/パッケージ名(@version),パッケージ名...
```
という感じ。もっと細かい指定やカスタム指定はjsdeliverのgithubページで見てもらうと良い。

すごく構造化されているので、例えばサーバーサイドでこのURLを構築するのもざっくりこんな具合にさっくりできる。

```ruby

def generate_jsdelivr_url(pkgs)
    path = pkgs.map{ |pkg_name, v| "#{pkg_name}@#{v}" }.join(",")
    "https://cdn.jsdelivr.net/g/#{path}"
end

pkgs = {
    "jquery": "1.11.3",
    "jquery.ui": "1.11.4"
}

generate_jsdelivr_url(pkgs)
// => "https://cdn.jsdelivr.net/g/jquery@1.11.3,jquery.ui@1.11.4"
```

もしCDNでなく自前のサーバーから配信したいなどがあれば、上記のURLをwgetでもすればよいのではなかろうかと思う（良いかはわからんけど、bower捨てる手法としては使えそう）。

### 番外編：jsdelivrに登録されてないライブラリどうするか？
順当にいけばCONTRIBUTEするのが一番。
とはいえ登録は多少面倒なところ[^2]

これに対して便利なのは[npmcdn](https://npmcdn.com/)。
npmに上がっているものは全て使えるので嬉しい。
ただ企業が運用しているしっかりとしたようなものでもないので、
polyfillなどのさほど重要でもないもの利用に留めている。
jsdelivrも自動でnpm配信やってほしい・・・



[^1]: 今回の騒動ありきというわけでもなく「なんか最近globalインストールした場合とそうでない場合で結果が変わる」みたいな問題があったから。何かの兆候だった気もする。

[^2]: しかも面倒な割に数日前に一回[pull requestを投げてみた](https://github.com/jsdelivr/jsdelivr/pull/7768)けどなかなか音沙汰が無い・・・
