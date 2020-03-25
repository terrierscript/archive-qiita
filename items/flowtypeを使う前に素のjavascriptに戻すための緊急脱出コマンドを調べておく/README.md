
flowtypeがぼちぼち良い感じになりつつあるので使ってみたいが
心配症なので、転ばぬ先の杖として使う前に退路も準備しておきたいと思った。

---

# 緊急脱出コマンド

ざっくりこう。
（既にbabel利用されている環境を前提にしているので悪しからず）

```
% npm i babel-cli

# out-dirをsourceと同じにしているので注意！
% $(npm bin)/babel src --no-babelrc --plugins transform-flow-strip-types --out-dir src/
```

## 解説など

`bable-cli`と`transform-flow-strip-types`をベースにして、flowの型処理だけをする。
`babel-cli`はコード結合とかするものではないので、ソースコードの変換したいみたいな目的にも使える。
`--no-babelrc`もつけて、通常利用しているような変換は行ないようにする。
これをやらないとソースコードが変換されて望まない結果になりがち。

また、上記コマンドは`src`からそのまま`src`に吐くので、事前にgitの差分が無い状態で実行すること。
dry-run的に別ディレクトリに吐き出したいなら、`--out-dir`を任意に変更すると良い。

## 構文エラーとかで怒られる場合

jsxなどbabelのみが認識するような構文を扱っている場合は、それぞれに構文に対応する[`syntax`](https://babeljs.io/docs/plugins/#syntax-plugins)をpluginとして追加する必要もある。
`transform`ではなく`syntax`を利用するので、パースはするがコードが変換されることはない。

自分の場合は`syntax-object-rest-spread`, `syntax-jsx`を追加すればよさそうだった。

```
% $(npm bin)/babel src \
  --no-babelrc \
  --plugins=transform-flow-strip-types,syntax-object-rest-spread,syntax-jsx \
  --compact=false \
  --out-dir=src/
```

syntaxだけをまとめた[babel-preset-syntax-from-presets](https://www.npmjs.com/package/babel-preset-syntax-from-presets)というのもあるらしい（未検証）

## 微妙に変わっている部分を`eslint --fix`でなるべく戻す

`babel-cli`を通しても大きくコードは変更されないとはいえ、インデントやら改行周りやらセミコロンなど、細々した構文が変わる。
全部戻そうとするとかなり労力かかるが、`eslint`でそれなりに戻すことが出来る。
なのでdiff小さく保ちたければ、事前、事後でeslintを利用して、`--fix`をかけると良い。

このあたりが使われそうなルールだった。

* arrow-parens
* space-before-blocks
* keyword-spacing
* semi
* jsx-wrap-multilines (eslint-plugin-react)
* jsx-curly-spacing (eslint-plugin-react)

## 雑感

* TypeScriptも検討したが、このぐらいのコマンド程度で素のJavascriptに戻せるflow悪くなさそう。
* 「OCamlで書かれている！」という触れ込みにビビっていたが、OCaml使ってる部分は`flow-bin`の型チェックする部分のみっぽい。
  * ビルド時に作用するのはbabel(=Javascriptオンリー)なので、「ビルド時にOCamlがうまくいかなくて詰んだ」みたいなのはそんなに無いはず。
  * 逆に、ビルド時にチェックしてくれないと困る！というユースケースには一工夫いりそうな予感
* 初期段階ではテスト`@JSDoc`的な奴、ぐらいなゆるいつきあい方をしていくほうがいいのかもなーと思ったりしている。
  * TODO: [babel-plugin-typecheck](https://github.com/codemix/babel-plugin-typecheck) とかは良いかもしれないので調べる
  * 参考: http://qiita.com/mizchi/items/30a5f9560e86e0d5ab31
