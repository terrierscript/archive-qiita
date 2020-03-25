
xcodeをアップデートしたりすると

```
Apple LLVM error
```

```
fatal error: file '/Applications/ ~~~~~~~~~ .pch' was built
```

みたいなエラーで、cocoapodsでインストールしたやつらでコケてはまって対処忘れるのでメモ

## 対処法
```
rm ~/Library/Developer/Xcode/DerivedData/ModuleCache/*
```

以上
