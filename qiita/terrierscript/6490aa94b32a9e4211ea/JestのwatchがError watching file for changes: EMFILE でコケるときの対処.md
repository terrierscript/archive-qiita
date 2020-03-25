
Jestのwatchモードが下記のようなエラーで落ちてハマったので対処法をメモ。
多分そのうち治る物と思われる

```
$ yarn test -- --watch
2017-01-05 15:18 node[24494] (FSEvents.framework) FSEventStreamStart: register_with_server: ERROR: f2d_register_rpc() => (null) (-22)
2017-01-05 15:18 node[24494] (FSEvents.framework) FSEventStreamStart: register_with_server: ERROR: f2d_register_rpc() => (null) (-22)
events.js:160
      throw er; // Unhandled 'error' event
      ^

Error: Error watching file for changes: EMFILE
    at exports._errnoException (util.js:1022:11)
    at FSEvent.FSWatcher._handle.onchange (fs.js:1282:11)
```

**（とりあえずの）解決法: watchmanをbrewから入れる**

```
$ brew install watchman
```

[watcnman](https://facebook.github.io/watchman/)はFacebookの作っているファイル監視するツール。jestが内部的に依存している

* [Watch mode stopped working on macOS Sierra #1767](https://github.com/facebook/jest/issues/1767)
    * [#issuecomment-248883102](https://github.com/facebook/jest/issues/1767#issuecomment-248883102)

どうもmacOS Sierraで起きている問題の模様。
node_modulesを消すと治った人もいる模様だったが、自分の場合はうまくいかなかった。
