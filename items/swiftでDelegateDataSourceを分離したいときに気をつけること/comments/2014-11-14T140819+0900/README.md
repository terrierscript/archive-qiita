> 一体何なんだこれはと思っていたがおそらくdataSourceの参照先が開放されてしまっているのだろう。

で合っていると思います。
UITableViewのdataSourceは弱参照のようですので、ほかに強参照がなければ解放されてしまいます。
こういうケースは定義を見れば一発ですね。

```swift:
unowned(unsafe) var dataSource: UITableViewDataSource?
```
