

画像などを遅延読み込みをするlazyloaderが久々に必要になってナウいやつを探してたら見つけた。
[the ultimate and lightweigth lazyLoader](http://afarkas.github.io/lazysizes/)だそうです。つよそう。

軽く使って良さそうなのにあまり記事がなかったので個人的まとめ。

# 良いとこ
- ピュアな実装でjqueryなどフレームワークに依存していない
	- jquery.lazyloadあたり使いたいけどjqueryを入れねば、みたいな問題がない。
- レスポンシブ対応などが豊富
- プラガブルな実装
- 画像以外にもスクリプトやiframeにも対応しているよう
- Star数:2625（2014/12/27時点）。
- 作者はhtml5shivなどの作者らしい
	- https://github.com/aFarkas

# 悪いとこ

- initial commitが3ヶ月前(2014/10）でまだまだ枯れてない
- バージョンは0.6。まだベータ的扱いかも。
- testなどまだなさそう


# 使い方
[github](https://github.com/aFarkas/lazysizes)から大いに抜粋

## 基本
bowerなりnpmなりからリソースは落とすところは省略します。
(筆者はrailsプロジェクトでの利用だったので`rails-assets-lazysizes`を使いました。）

```html
<script src="lazysizes.min.js" async=""></script>
```
`script`のasyncは必須というわけではなさそうです。

### 普通のlazyloaderっぽい使い方

```html
<img src="placeholder.jpg" data-src="imageurl.jpg" class="lazyload" />
```
ふつうですね。
`<img class="lazyload">`というクラス指定で遅延対象を指定します。
対象とするクラス名は設定で変更可能（後述）

### レスポンシブ対応
未検証。
`srcset`をlazyにしてくれるみたい。
個人的に使ったことは無いけど使いどころありそう。

```html
<img data-srcset="responsive-image1.jpg 1x, 
responsive-image2.jpg 2x" class="lazyload" />
```
これを利用して[LQIP](https://github.com/aFarkas/lazysizes#lqip)という荒い画像を先行で読み込ませて遅延で高解像度に切り替えるという手法が紹介されています。

### 設定変更
`window.lazySizesConfig`に記載。

```js

window.lazySizesConfig = {
    lazyClass: 'postbone', // lazyloadの対象とするクラスの指定。
};

```

オプションはたくさんあるので詳しくは[このへん](https://github.com/aFarkas/lazysizes#js-api---options)
## プラグインの話
lazysizesは`lazybeforeunveil`というイベントを発火するため、これをフックすることで何がしかの処理をかませることが出来ます。
幾つか同梱されているプラグインはだいたいこのhookを利用しているようです。

```js
document.addEventListener('lazybeforeunveil', function(e){
    // 何らかの処理
}, false);
```

### 便利機能詰め合わせな[unveilhooks plugin](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/unveilhooks)
これscriptを遅延させたり動画を遅延させたり地味にいろんな機能があります。
背景画像なんかはよく使いたくなる割にドキュメントが奥のほうにあって見つけるの時間かかったりしたのではまった。

#### 背景画像
`unveilhooks`を読み込んで`data-bg`を使うだけ。

```html
<div class="bg lazyload" data-bg="http://cdn.qiita.com/assets/siteid-reverse-cd0519106e5814e34d402b3e821820cf.png"></div>

<script src="./lazysizes/plugins/unveilhooks/ls.unveilhooks.js"></script>
<script src="./lazysizes/lazysizes.js"></script>
```

### 他のプラグインざっくり
他のはあんまり触ってないのも多いのでドキュメント見つつ

- [RIAS plugin](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/rias)
	- responsiveな画像処理を外部サービスなんかに任せてしまうためのものらしい
	- `data-src="http://foo.com/img/{width}x{height}"`みたいな具合？
- [OPTIMUMX](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/optimumx)
	- スマホなどで最適なサイズを見つけているっぽいけどよくわからなかった・・・・
- [bgset plugin](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/bgset)
	- 背景に`srcset`的な動きをさせるもの。
	- 最初普通の背景のlazyにこっちを使ってしまったのでちょっと触った
	- [respimage](https://github.com/aFarkas/respimage)というpolyfillを使わないとchrome以外のブラウザなどでは動かない
	- そして体感たまに読み込み成功してくれない気がする（要調査）
- [include plugin](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/include)
	- htmlをパーシャル的にlazyでincludeするプラグインらしい。

- [print plugin](https://github.com/aFarkas/lazysizes/tree/gh-pages/plugins/print)
	- 印刷時に全部ロードするよーというものらしい。なるほど。

# まとめ
lazysizesとてもよいし未来を感じる。
