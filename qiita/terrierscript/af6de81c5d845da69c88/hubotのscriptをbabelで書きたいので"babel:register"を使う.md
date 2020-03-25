
coffeescriptもうあんま覚えてないしhubotのスクリプトもそろそろbabelで書きたかったのでどうにかした。

結論としては単に`require("babel/register")`すればよかった。
[electronで似たようなこと](http://qiita.com/suisho/items/a3822167604e5ad6c19b)をした時とだいたい着想は一緒。
あとはhubot-scriptはfunctionなので`robot`をパスしてやれば良いだけ（のはず）

```script/some-script.coffee
require("babel/register")
module.exports = (robot) ->
  require("../lib/babel-script")(robot)
```

呼び出され側はこんな感じ

```lib/babel-script.js
export default function(robot){
  robot.hear(/badger/i, (res) => {
    res.send("Badgers? BADGERS? WE DON'T NEED NO STINKIN BADGERS")
  })
}
```

