@ykzts 

書き方が不十分でしたので訂正しました。複数ファイルで`require`してそれぞれ読み込んでいるような状況を指しておりました。

このような場合です

```html
<script src="/a.js"></script>
<script src="/b.js"></script>
```

```a.js
require("babel-polyfill")

// some code...
```

```b.js
require("babel-polyfill") // throw errro

// some code...
```

そんなに頻繁に起きるケースではないのですが、うっかりしてるとやってしまう系ですね・・・
