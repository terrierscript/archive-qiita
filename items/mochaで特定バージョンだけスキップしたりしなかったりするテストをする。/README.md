
[circle-ciで複数バージョンを検証](http://qiita.com/inuscript/items/f5dd5eacb10085d15ff7)できるようにしたので、特定バージョンだけでやるテストを書きたい。

`process.version`で現在のnodeのバージョンは取れるので、そこに`semver`を組み合わせると手軽にできる

```mocha.js
var semver = require("semver")
describe("何らかのテスト", function(){
  var itFn = it
  if(semver(process.version, ' < 0.12')){
    itFn = it.skip
  }

  itFn("0.12以上ならやるテスト", function(){
    // test...
  })
})
```
