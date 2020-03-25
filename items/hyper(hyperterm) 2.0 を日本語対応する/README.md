hyper(term) 2.0 以上で日本語化する

hypertermが日本語対応ずっとされてなかったのが

https://github.com/zeit/hyper/issues/2322
https://github.com/zeit/hyper/pull/2913

このへんで解決されたようだが、デフォルトではやはりうまく対応されてなかった

# 答え：localeは設定する必要があった。
.bashrcや.zshrcに設定していれば不要。今回はhyperterm用に設定する

メニューから `hyper` -> `preferences` で、.hyper.jsファイルを開いて、envに下記を設定

```.hyper.js
env: {
  LANG: "ja_JP.UTF-8",
  LC_ALL: "ja_JP.UTF-8"
},
```

この後、**hyperのプロセスを完全に落として再起動**する

localeを実行して下記のようになっていれば成功

```
% locale
LANG="ja_JP.utf8"
LC_COLLATE="ja_JP.UTF-8"
LC_CTYPE="ja_JP.UTF-8"
LC_MESSAGES="ja_JP.UTF-8"
LC_MONETARY="ja_JP.UTF-8"
LC_NUMERIC="ja_JP.UTF-8"
LC_TIME="ja_JP.UTF-8"
LC_ALL="ja_JP.UTF-8"
```

日本語入力も表示もできるはず
