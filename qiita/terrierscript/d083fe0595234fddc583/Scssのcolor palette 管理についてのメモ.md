
今までなんとなしに色管理とかカラーパレットは

```scss
$color-red : #fe0000;
$color-red-dark : #ce0000;
$color-blue : #000ef;
$color-blue-dark : #000ec;
```
てな具合で十分かなと思っていたけど最近はmapとかあるしその辺使った管理とかどうかなとか思ったらよさ気なのがあったのでまとめ。

- http://erskinedesign.com/blog/friendlier-colour-names-sass-maps/
- http://maketea.co.uk/2014/07/21/managing-relationships-between-colours-with-sass.html

# mapを使ったいい感じのやり方
まず最終的なアウトプットから。

```scss
// パレットの定義
$color-palette :( 
  red : (
    base : #ff0000,
    dark : #ffcccc
  ),
  blue : (
    base : #0000ff,
    light : #00bbff,
  )
) !default;

// color(色系統, トーン)
@function color($color, $tone){
  @return map-get(map-get($color-palette, $color), $tone)
}
```

Sample usage

```sass
.red-button{
	background  : color(red);
}

.dark-blue-button{
	background  : color(blue, light);
}
```
関数名はまあなんでも良さげではあるけど`color()`とか`palette`とかがあるようだった。

こうすることでこれなにが利点なのか？

### 利点1：色ごとのセレクタ動的生成
というとmapなので繰り返しができること。

例えばstyleguide用に各色のサンプル出したいならこんな感じだし

```scss
// 一覧サンプル出力
@each $name, $palette in $color-palette {
  @each $tone, $code in $palette {
    .sample-color-palette-#{$name}-#{$tone}{
      background : color($name, $tone);
      .black-text{ color : #000 };
      .white-text{ color: #fff };
    }
  }
}
```

ボタンを特定の色で作りたかったらこんな感じとか

```scss
// redとblueのボタンを自動生成
@each $color in red, blue{
  .button-#{$color}{
    color: white;
    background : color($color);
  }
}

// 実際には色名をクラスには付けないのでこんな用途になりそう
$level-map :(
  error : red,
  warn : yellow,
  info : blue
);
@each $level, $color in $level-map{
  .log-#{$color}{
    color: white;
    background : color($color);
  }
}
```

### 利点2:色生成の隠蔽
今回paletteはmapで管理したけども、将来リファクタで整理されて、`lighter`などの色関数を使えばよくね？みたいなことになった時に影響少なめで変更できそう（と思ったけどあんまりこれが活きるパターンはないな・・・）


## 欄外：なんでこんなことを思ったのか

そもそも実は

```scss
$color-map : (
	red : #ff0000,
	blue : #0000ff
);
@each $color, $code in $color-map{
	$color-#{$color} : #{$code}
	.sample-#{$color} :{
		background : $color-#{$color}
	}
}
```
のように動的にvariableを展開するようなのとかできないのかなーと思って調べていたら似たようなことをして詰まっている人を見つけた

https://github.com/sass/sass/issues/1450

このissueでも上記のようなやり方が紹介されていて、提唱者は問題なかったといっている。めでたしめでたし。
