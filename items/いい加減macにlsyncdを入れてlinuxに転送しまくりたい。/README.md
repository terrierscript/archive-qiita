
のっぴきならない事情でLinuxのリモートマシンで作業をしないといけないことがある。
とはいえ作業は手元のmacでやりたい（source tree使いたい！atom使いたい！）

ということで今までgruntでwatchしてrsync・・・みたいな力技でちょろまかしていたけど、ちょっと人に伝授する機会があって今まで何度か挫折してたlsyncdを導入してみた。

今回の要件としてはこんな感じ

- なるべくセットアップを簡易に
- 遅延はなるべく無く
- 転送は片方向だけでよしとする（ mac -> linux )
- sshの鍵設定はしているものとする
	- してないと転送のたびにパスワード聞かれてしまった。

# install
今回はrsyncを使った転送をするので、rsync、あとはatomあたりでハイライトするようにluaも入れておく。

```
brew update
brew install homebrew/dupes/rsync
brew install lsyncd lua
```


## lsync.conf
今回デザイナーさんなどに伝授するにあたり、設定ファイルを分離するとかちょっとだけ複雑なことをした。

まず設定ファイルだけ外にまとめとく。
ここだけいじれば大丈夫よ的な感じに。

```conf.lua
return {
  user   = "your.name",
  host   = "0.0.0.0",
  source = "/path/to/source",
  dest   = "/path/to/dest",
}
```

あと本筋のlsync.confの方がこんな感じ。

```lsync.conf.lua
-- variables
local conf = dofile("conf.lua")

local homeDir = os.getenv("HOME")
local target = conf.user.."@".. conf.host..":".. conf.dest

-- lsyncd
settings{
  logfile     = "/tmp/lsyncd.log",
  statusFile  = "/tmp/lsyncd.status",
  insist      = true,
  nodaemon    = true,
  delay       = 0,
  statusInterval  = 10,
  maxProcesses    = 2,
}

sync{
  default.rsync,
  delete = false,
  source = conf.source,
  target = target,
  exclude = { ".DS_Store", ".sass-cache/**/*"},
  rsync = {
    binary = "/usr/local/bin/rsync",
    archive = true,
    compress = true,
    rsh = "/usr/bin/ssh -i "..homeDir.."/.ssh/id_rsa"
  }
}
```

なんか注釈としてはこんな感じ

- `dofile`とかでファイル読み込んでる。
- sshの鍵ファイル特定を怠けるのに`$HOME`を`getenv`で読み込んでる
- delayがデフォルト15ぐらいになっていて即時反映されなくて困るのでここは変えておく
- excludeは適当に設定。

# 起動

```
sudo lsyncd lsync.conf.lua
```

なんかwarnが出るのだが仕方なさそう。挙動は今のところ問題なし
https://github.com/axkibe/lsyncd/issues/291



### まとめ
lsyncdについてわりとmacでの情報はなさ気だった。
けど割りと普通に使えた。
lua楽しい。

とりあえず今回作ったのはここに置いた。
https://github.com/suisho/lsyncd-boilerplate
