
rails 5.1で取り込まれそうになっていたりまた熱い動きがありそうなyarnさん。
いつの間にか入っていた`upgrade-interactive`コマンドが非常に便利だった。

https://github.com/yarnpkg/yarn/pull/1444

今日現在（2016/12/05）、公式ドキュメントにまだ載ってないが、便利すぎるのでみんな使ったほうが良いと思う

### 準備

yarnを最新にしておく。
v0.17.0 以上であれば利用できる。

```
$ yarn self-update
```

### 何が出来るか？

npm outdatedの超すごい奴という認識で良い気がする。
upgradeをinteractiveに出来る。（名前のまま）

```
$ yarn upgrade-interactive
```

これを叩くとこんな画面が出て来る

![9__yarn_upgrade-interactive__node_.png](https://qiita-image-store.s3.amazonaws.com/0/7307/ef478d4d-1e8f-b96b-e308-ec1e8c222c0a.png "9__yarn_upgrade-interactive__node_.png")

アップグレードが出来るパッケージが表示される。
カーソルを動かしてアップグレードしたいパッケージを選び、<kbd>Space</kbd>キーで選択する。<kbd>a</kbd>キー（all)を押すと、全選択、<kbd>i</kbd>キーを押すとフラグが反転する。

選んだら<kbd>Enter</kbd>キーで実行

![7__inuscript_MacBook-Air____office_some_project__zsh_.png](https://qiita-image-store.s3.amazonaws.com/0/7307/dbefa563-551b-dda1-758c-f6e1867a6323.png "7__inuscript_MacBook-Air____office_some_project__zsh_.png")

これだけでアップグレード完了！

マジで便利すぎる。yarn依存症になってしまいそう。
