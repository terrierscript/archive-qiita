`babel-plugin-transform-runtime` を使用して `babel-runtime` を `import` するコードに変換した場合も、`babel-runtime` が `babel-polyfill` と同様に `core-js` を使う前提になっているため、ネイティブで使える函数についてはそちらが優先されて使われます。
実行に必要な函数しかロードされなかったり、グローバルを polyfill で汚染しなかったりする点を考えると、やむを得ない事情がある場合を除いては `babel-plugin-transform-runtime` を使用される方がいいかと思います。
