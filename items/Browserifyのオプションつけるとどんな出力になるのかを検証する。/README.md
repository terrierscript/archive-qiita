[ドキュメント](https://github.com/substack/node-browserify)や[handbook](https://github.com/substack/browserify-handbook#excluding)見てもいまいち使い勝手がわかんなかったりする。

コンパイルして何が出てくるのか見てみるとちょっとだけ知見が深まったのでまとめたい。



### 免責
- サンプル貼り付けるにあたって、requireしているとこらへんとかは省略する
- 検証するにあたってformattedな状態で読みたかったので[出力は工夫した](http://qiita.com/inuscript/items/dd56f1c397419bfc42ca)。
	- 同様のことをやる場合は併せて参照されたい
- 試した結果のリポジトリはこんな感じ
	- https://github.com/inuscript/example-browserify-options
	- ちょいちょい試しつつなので、この記事と微妙にずれたりしているがご愛嬌

## その1. `entry`オプションだけ指定した基本的なやつ。
おさらい的な感じだけど

### 入力ファイル
なんらかのlibファイルとentryファイルを想定する

```somelib.js
module.exports = function(){
  console.log("somelibだよ！")
}
```
```entry1.js
console.log("entry1だよ！")
var lib = require("../lib/somelib")
lib()
```

### 実行
```
$ browserify -e ./src/entry1.js
```

### 結果
```outout.js
(function e(t, n, r) {
   // 省略
})({
    1: [function(require, module, exports) {
        module.exports = function() {
            console.log("somelibだよ！")
        }
    }, {}],
    2: [function(require, module, exports) {
        console.log("entry1だよ！")
        var lib = require("../lib/somelib")
        lib()

    }, {
        "../lib/somelib": 1
    }]
}, {}, [2]);
```

出ているものとしては1の方がlibで呼び出しているもので、2が`simple.js`に相当する。そしてentry指定しているので`[2]`が実行対象として呼び出されているらしい。

## その2. `entry`で複数指定するとどうなるか
ほぼその1のおまけ。複数のファイルを`-e`で対象にしてみる

### 実行
```
$ browserify -e ./src/entry.js -e ./src/entry3.js
```

### 出力
```outout.js
(function e(t, n, r) {
	// 省略
})({
    1: [function(require, module, exports) {
        module.exports = function() {
            console.log("somelibだよ！")
        }
    }, {}],
    2: [function(require, module, exports) {
        console.log("entry1だよ！")
        var lib = require("../lib/somelib")
        lib()

    }, {
        "../lib/somelib": 1
    }],
    3: [function(require, module, exports) {
        console.log("entry3だよ！")

    }, {}]
}, {}, [2, 3]);
```
実行対象が`[2,3]`となった！

## その3. `external`(`-x`)と`require`(`-r`)オプションを使う

これ理解したら一気にbrowserifyの世界が開けた。
コアな部分を`external`でentryには混ぜず、`require`でまとめると超捗る事が期待される。

### 入力

```somelib.js
module.exports = function(){
  console.log("somelibだよ！")
}
```
```entry.js
console.log("entry1だよ！")
var lib = require("../lib/somelib")
lib()
```

### 実行
```
$ browserify -r ./lib/somelib.js
$ browserify -e ./src/entry1.js -x ./lib/**/*.js
```

### 出力
```require.js
require = (function e(t, n, r) {
	// 省略
})({
    "/lib/somelib.js": [function(require, module, exports) {
        module.exports = function() {
            console.log("somelibだよ！")
        }
    }, {}]
}, {}, []);
```
```entry.js
(function e(t, n, r) {
	// 省略
})({
    1: [function(require, module, exports) {
        console.log("entry1だよ！")
        var lib = require("../lib/somelib")
        lib()

    }, {
        "../lib/somelib": "/lib/somelib.js"
    }]
}, {}, [1]);
```

requireとentryがこんな感じで出力される。requireがglobalに定義され、それを呼び出す形。htmlで呼びこむときはこんな感じで読むと良さそう

```jquery.html
    <script type="text/javascript" src="./build/require.js"></script>
    <script type="text/javascript" src="./build/entry.js"></script>
```


## その4. `exclude`(`-u`)オプション

できれば`require`と`external`で完結したいけど、例えばjqueryみたいなのは分割してbundleしたいみたいなときに使うとよいらしい。あんまりお世話になりたくない感じする。

### 入力

```entry-jquery.js
var $ = require("jquery")
console.log($(".foo"))
```

### 実行
```
$ browserify -r jquery 
$ browserify -r ./lib/somelib.js 
$ browserify ./src/entry-jquery.js -u jquery -x ./lib/somelib.js 
```

### 出力
```entry-jquery.js
(function e(t, n, r) {
	// 省略
})({
    1: [function(require, module, exports) {
        var $ = require("jquery")
        console.log($(".foo"))
    }, {
        "jquery": undefined
    }]
}, {}, [1]);
```

これで、下記のようにscriptタグを呼ぶと良い

```html
    <script type="text/javascript" src="../build/jquery-bundle.js"></script>
    <script type="text/javascript" src="../build/require-bundle.js"></script>
    <script type="text/javascript" src="../build/entry-jquery.js"></script>

```
`require-bundle`と`jquery-bundle`で二度window.requireが呼び出されてうまいこと処理されているのがなぜうまく動いてるのかわかってないので今度ちゃんと調べたい。

ちなみにこの例は[Handbook](https://github.com/substack/browserify-handbook#excluding)に乗っていたのだが、`standalone`オプションがあるとうまく動かなかった。（[issue#58](https://github.com/substack/browserify-handbook/pull/58/files)でも指摘されていた。これのせいで理解するまでめっちゃ時間かかった）

## その5. `standalone`(`-s`)オプション

正直使わない気がする。なにか最後の手段感がある。
雑な説明にはなるが、指定したmoduleを`window.指定した値`に指定するようなイメージをするとよさそう。

### 入力
```standalone.js
module.exports = function(){
  console.log("Standaloneなモジュールだよ”")
}
```
### 実行

```
$ browserify ./lib/standalone.js -s standalone
```

### 出力
```standalone.js
(function(f) {
	// 省略
	// window.standalone = module
	// になるようなコードが書かれている
    
})(function() {
	// 省略
   })({
        1: [function(require, module, exports) {
            module.exports = function() {
                console.log("Standaloneなモジュールだよ”")
            }
        }, {}]
    }, {}, [1])(1)
});
```
やっぱりあんまり積極的につかいたい感じはしない。

## その6. `ignore`(`-i`)オプション
これはたまにお世話になるかも。moduleの出力結果を空っぽにしてくれる。`fs`などはデフォルトでこのオプションの対象になっている

### 入力
```entry.js
console.log("entry1だよ！")
var lib = require("../lib/somelib")
lib()
```

### 実行
```
$ browserify -e ./src/entry.js -i ./lib/somelib.js
```

### 出力
```ignore.js
(function e(t, n, r) {
	// 省略
})({
    1: [function(require, module, exports) {

    }, {}],
    2: [function(require, module, exports) {
        console.log("entry1だよ！")
        var lib = require("../lib/somelib")
        lib()

    }, {
        "../lib/somelib": 1
    }]
}, {}, [2]);
```

こんな感じ。
1の`somelib`の方が空になっている。
excludeと何が違うのって話がわかっていなかったけど、ユースケース考えると明確。

例えば`iconv`みたいにnode-gypを使ってビルドするようなパッケージを部分的に利用しているnpmパッケージを利用していたとして、それをそのまま呼んでしまうと当然コケる。そんなときは`iconv`だけ空で出力してくれると都合が良い。

```
$ browserify -e ./src/ignore.js -i iconv
```

こんな感じ。

# まとめ
とりあえずこのぐらい知っておけば色々怖がらずに使えそう。
