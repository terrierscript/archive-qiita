
CALayerなんかを使ってUIVewに線を追加するやり方なんかはわりとよくあるのだけど、InterfaceBuilder上でいい感じの位置に線を出したいとなるとちょいと工夫が必要だったのでメモ

## まずは結果から
<img src=https://qiita-image-store.s3.amazonaws.com/0/7307/676f09f6-0c6a-f8a1-8ee2-ce39d7a1527b.png width=300 style="border:1px solid #ccc">
求められているのは0.5の線。
いまいち理解が甘いのですがAutolayoutでHeight=1を設定しても2px程度になるようです。

## 新規に設定する場合
まずはさくっとviewを配置
<img src="https://qiita-image-store.s3.amazonaws.com/0/7307/52ae843e-0d99-d8e4-2440-ed14cd0d2ad2.png" width=300 style="border:1px solid #ccc">

そしてAutolayoutの設定で **Heightに.5と入力**
<img src=https://qiita-image-store.s3.amazonaws.com/0/7307/c71eb2c6-60ce-2744-25ac-c8d85e4b7479.png width=300 style="border:1px solid #ccc">

ここまではOK。

## 途中から設定する場合
しかし上記のHeight 0.5、初回にかけてあげることはできるようですが、
<img src=https://qiita-image-store.s3.amazonaws.com/0/7307/5471306f-9258-0cef-a755-2cbfee247414.png width=300 style="border:1px solid #ccc">
こちらのSize Inspectorからは設定できない模様・・・・

ちょっと新規でやりなおすのもな・・・という時は、下記のやり方もあるようです。

こんな感じでconstraintとoutletを紐付けて
<img src=https://qiita-image-store.s3.amazonaws.com/0/7307/ba03ba4f-bc1b-cb6d-775c-c136119822ee.png width=400 style="border:1px solid #ccc"> 

```swift
class ViewController: UIViewController {
	// constraint outlet
    @IBOutlet weak var heightConstraint: NSLayoutConstraint!

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
        heightConstraint.constant = 0.5
    }
    
}
```

こんな具合でconstraintを変更してやることでも可能なようです。
