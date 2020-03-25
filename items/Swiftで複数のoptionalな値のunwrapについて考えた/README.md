
### 2015/03/27追記
**Swift 1.2から、[if let構文の複数宣言に対応](http://qiita.com/karly/items/3e9c4387f60bfea8424d#%EF%BC%92if-let%E3%81%AE%E8%A4%87%E6%95%B0%E5%AE%A3%E8%A8%80%E3%81%AB%E5%AF%BE%E5%BF%9C)していたようです！**
今後はそちらを利用されることをオススメします

### ↓以下過去の情報

swiftでoptionalな値のunwrap

```swift
func foo(a : String?) -> String?{
    if let _a = a{
        return _a + "foo"
    }
    return nil
}
```

これが複数になったとき、

```swift

func foo(a : String?, b: String?) -> String? {
    if let _a = a { 
        if let _b = b {
            return _a + "&" + _b
        }
    }
    return nil
}
```

だとこんな具合になってネストがえらい深くなってその先に処理が書かれることになって悲しみが生まれる。
`if let _a = a && _b = b` みたいなの無いのかなあと思ったけどどうも無いらしい。

ちょっと無理すれば

```swift
func foo2(a : String? , b:String?) -> String?{
    if a == nil{
        return nil
    }
    if b == nil{
        return nil
    }
    
    return a! + "&" + b!
}
```
こんな感じにもできるけど、force unwrapするのはあまりやりたくない・・・
もうちょっと何とかならないかとか色々考えたり調べたりした。

### tupleとswitchを利用

ちょっと調べるとstackoverflow先生に次のような[解法](http://stackoverflow.com/questions/24548999/unwrapping-multiple-optionals-in-if-statement)があった

```swift
func foo(a : String?, b : String?) -> String?{
	switch(a,b){ // tuple化して
	case(let .Some(a), let .Some(b)): //Some型でマッチ
    	return a + "&" + b
	default:
    	return nil
	}
}
```

なるほど。確かにこれならネストを何段も重ねるという点が解消される。記述がちょと面倒な感じはあるけど・・・

### unwrapした場合のメソッドを多重定義する
ネストの深さが解消されるわけではないけど、ネストした先の実処理さえ別関数として分離しておけば多少マシになるのではないかというやり方
 
```swift
// optionalな引数の場合はunwrapするだけ
func foo(a : String? , b: String?) -> String?{
    if let _a = a{
        if let _b = b{
            return foo(a,b)
        }
    }
    return nil
}

// 実際の処理
func foo(a : String, b: String) -> String{
    return a + "&" + b
}

```

うーん。実際の処理はそれなりに複雑にはなるとしても、やっぱりそのまま書くよりはマシという程度。

### 寸感

* switchのやり方はスマートではあるけど一瞬なんだっけってなりそう。でも値が多い時は良さそう
* メソッドの多重定義はまあ書き方としては素直なのである程度色々解決はしそう。
* もうちょっと良いやり方がほしい
