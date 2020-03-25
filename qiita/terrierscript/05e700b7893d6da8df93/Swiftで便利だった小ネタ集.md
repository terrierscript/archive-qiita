
記事にするのもアレ or いまいちちゃんと調べきれてない雑多な小ネタ集

# Swiftの小ネタ
## printlnにtuple使うと便利
複数の値を一気にコンソールに出したいとき、最初は

```swift
println("some value")
println(value)

```

とやっていたけどタプル化して

```swift
println(("some value", value))
```

とすればよい。地味に便利

### 追記(2014/12/10)
Xcode6.1.1で試してたところ

```swift
println("foo","baz")
```

が使えるようになっていました。

## optionalな値のデフォルト
scalaでいうとこのgetOrElseなのを探してたらこんなかんじだった。
javascriptとかrubyでよくやる記法に近い気がする

```swift
let baz : String? = nil
let foo = baz ?? "default value"
```


# XCodeがらみの小ネタ
## xibで紐付けたoutletをオプショナル型にすればIBの紐付けが無くても大丈夫になって便利

複数のxibを扱うようなViewCellなどのクラスで、InterfaceBuilderから紐付けた時基本`!`のunwrap記号が付与されるが、これを`?`にすれば片一方のxibに存在していないようなヒモ付ができた（TODO：あまり仕組みちゃんと調べてないので要検証）

```swift
class SomeCell: UICollectionViewCell{
    @IBOutlet var labelView: UILabel!
    @IBOutlet weak var image: UIImageView!
    @IBOutlet weak var optionalLabel: UILabel? // xibによっては存在しなかったりするならこんな感じで
}
```

## lldbでは`fr v someValue`を使えばコンソール出力できる
`po someValue`でエラーが出ちゃってなんだろうなーと思っていたけど`fr v`で出るっぽい。（ちゃんとわかってない）
出展このへん：http://natashatherobot.com/swift-debugging/
