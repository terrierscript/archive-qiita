- 引数のどれかが`nil`ならば関数は`nil`を返すことにする。
- `if let _a = a && _b = b`のようなものがほしい。
 
を満たすように，オプショナルなタプルを使う方法を考えてみました。
オプショナルな引数列をオプショナルなタプルへコンバートして，目的の関数本体でタプルに対して`lf let`を使えるようにします。

```swift
func makeOptionalTuple(a0: String?, a1: String?) -> (String, String)? {
    if a0 == nil || a1 == nil { return nil }
    return (a0!, a1!)
}

func foo(a: String?, b: String?) -> String? {
    if let (_a, _b) = makeOptionalTuple(a, b) {
        return _a + "&" + _b
    }
    return nil
}

foo("a", "b")    // {Some "a&b"}
foo(nil, "b")    // nil
foo("a", nil)    // nil
foo(nil, nil)    // nil
```
makeOptinalTuple側でタプルを作るときに強制アンラップを使っていますが，目的の関数自体はスッキリと記述できるかなと。
