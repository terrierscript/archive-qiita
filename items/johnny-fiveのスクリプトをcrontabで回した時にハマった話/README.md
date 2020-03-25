
# 構成
よくあるRaspberry PiとArduinoつないだやつ。

- Raspberry PiとArduino Unoに接続
- Arduinoはjohnny-fiveの手順どおりStandardFirmataをUpload 
- Raspbian側でcrontabを回して、johnny-fiveを利用したスクリプトを実行

# ハマった話
温度をとるためにこんなスクリプトを書いた


```js
var five = require("johnny-five")
var board = new five.Board()

module.exports = function(cb){
  board.once("ready", function() {
    
    var temperature = new five.Temperature({
      controller: "LM35",
      pin: "A0"
    })

    temperature.once("data", function(err, data){      
      if(err){
        console.warn(err)
      }
      cb(err, data.celsius)
    })
  })
}

```

で、crontabをこんな感じにした

```
$ sudo crontab -l
* * * * * /usr/local/bin/node /path/to/script.js
```

が、どうも思った通りにデータを送信してくれない。
ログを見たりしていると、cronが回っているが出力がうまくいってないらしい。

## ログを吐かせてみる
```
* * * * * /usr/local/bin/node /path/to/script.js > /home/pi/c1 2>/home/pi/c2
```

こんな調子でログを吐かせてみるとざっくりこんなエラーが出てた（timestampとかは削っている）

```
Device(s) /dev/ttyACM0
Connected /dev/ttyACM0
Repl Initialized
Board Closing.
```
クロージングってなんでだよ。

## 解決編
散々悩んだ挙句、デフォルトがreplになっていて、それが起動してしまうことでうまく回ってくれなかったらしいことがわかった。

ということでこんな感じにしてやっと動いた。

```js
var five = require("johnny-five")
var board = new five.Board({
  repl: false
})
```

