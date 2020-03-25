
よくあるsvgで星が半分塗られた状態を再現する方法を考える

## 方法1. 半分の星の状態を用意する

とても素直。作れるならそれで良い気がします。

## 方法2. viewBoxをずらして色を半々で重ねる

![2-color.png](https://qiita-image-store.s3.amazonaws.com/0/7307/da04d784-7803-642e-edc0-d482b170a6da.png "127_0_0_1_8080.png")

結構望んでいたやつ。左右で配置する。

```star.svg.xml
<svg width="16" height="16" viewBox="0 0 16 16">
  <!-- left -->
  <svg width="8" height="16" viewBox="0 0 8 16">
    <use xlink:href="star.svg#star" fill="orange" />
  </svg>
  <!-- right -->
  <svg width="8" height="16" viewBox="0 0 8 16" x=8>
    <use x="-8" xlink:href="star.svg#star" fill="#ddd" />
  </svg>
</svg>
```

### 応用：strokeで一部塗りつぶしっぽくする
![stroke.png](https://qiita-image-store.s3.amazonaws.com/0/7307/55bc4b5b-f114-1088-47ea-f9b5dc49c3bd.png "127_0_0_1_8080.png")
strokeだけしている右側を作る。stroke分1pxずつずらしたり細工する

```star.svg.xml
<svg width="16" height="16">
  <svg width="8" height="16" viewBox="0 0 9 18">
    <use x="1" y="1" xlink:href="star.svg#star" fill="orange" stroke="orange" />
  </svg>
  <svg width="8" height="16" viewBox="0 0 9 18" x="8">
    <use y=1 x="-8" xlink:href="star.svg#star" fill="transparent" stroke="orange"/>
  </svg>
</svg>
```

### 方法2のボツ案

片方をviewboxでずらさずに上に引くやり方だとコード量は減るが、ジャギが出てしまってイマイチな感じになるが、参考までに掲載
![ボツ.png](https://qiita-image-store.s3.amazonaws.com/0/7307/d60de55d-e9ef-7170-a074-b93de9fd8608.png "127_0_0_1_8080.png")

```ボツ案.svg.xml
    <svg width="16" height="16" viewBox="0 0 16 16">
      <use xlink:href="star.svg#star" fill="orange" />
      <svg width="8" height="16" viewBox="0 0 8 16" x=8>
        <use x="-8" xlink:href="star.svg#star" fill="#ddd" />
      </svg>
    </svg>
```


## 方法3. preserveAspectRatioを使って半分塗りつぶす
方法2のほうがスマートだが、preserveAspectRatioを使っても再現できる
（書いた後に気づいたがどうもIE11でうまくいかなそう・・・）

```star.svg.xml
<svg width="16" height="16" viewBox="0 0 16 16">
  <!-- left -->
  <svg 
    width="8" height="16"
    preserveAspectRatio="slice xMinYMin">
    <use xlink:href="star.svg#star" fill="orange" />
  </svg>
  <!-- right -->
  <svg 
    x="8"
    width="8" height="16"
    preserveAspectRatio="slice xMaxYMax">
    <!-- x位置をずらす -->
    <use x="-8" xlink:href="star.svg#star" fill="#ddd" />
  </svg>
</svg>
```

## 方法4. colorとfillを使う

[こちらの記事](http://tympanus.net/codrops/2015/07/16/styling-svg-use-content-css/)を参考に、`currentColor`を利用してハックする。
この方法の場合、方法2,3のように単純なpathではなく、元々のsvgのpathを分ける

```star.svg
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
	<defs>
		<svg id="star" viewBox="0 0 16 16" >
			<path class="star-left" d="M-右半分のpath"/>
			<path class="star-right"
				fill="currentColor" d="M-左半分のpath"/>
		</svg>
	</defs>
</svg>
```

呼び出しはこんな感じ

```usage.svg
<svg class="rating-star" width="20" height="20"> 
  <use class="half" xlink:href="star.svg#star">
</svg>
```

そしてcssで、fillとcolorを指定する。

```css
.rating-star .half{
  fill: orange;
  color: gray;
}
```

## その他方法
コメント欄より:

GradientやPatternを利用した方法を教えていただきました
参考: [SVGで半分だけ塗られた星を再現する](http://okiru.net/misc/20160214/)
