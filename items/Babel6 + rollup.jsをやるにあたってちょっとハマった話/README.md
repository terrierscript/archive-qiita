

*2016/09/06追記: Babel 6.13以上より、presetsのオプションとして、`modules`指定ができるようになったため、`babel-preset-es2015-rollup`は不要になったようです。*
*詳しくは[rollup-plugin-babel](https://github.com/rollup/rollup-plugin-babel)をご確認下さい*

---

ちょっと調べれば良い事だけどちょっとハマったのでメモ

# 発端
とりあえずbabel公式ページの[rollupのinstall](https://babeljs.io/docs/setup/#rollup)にしたがってみたがどうにもうまくいかない

## 何が起きたか？
```
It looks like your Babel configuration specifies a module transformer. Please disable it. If you're using the "es2015" preset, consider using "es2015-rollup" instead. See https://github.com/rollup/rollup-plugin-babel#configuring-babel for more information
Error: It looks like your Babel configuration specifies a module transformer. Please disable it. If you're using the "es2015" preset, consider using "es2015-rollup" instead. See https://github.com/rollup/rollup-plugin-babel#configuring-babel for more information
```
こんなエラーが。

# 解決

結局`babel-preset-es2015-rollup`が必要ということだった。

```
$ npm install rollup -D
$ npm install rollup-plugin-babel -D
$ npm install babel-preset-es2015-rollup -D # これが必要
```

babelの設定もあわせてこんな感じで行う

```package.json
    :
  "babel": {
    "presets": [
      "es2015-rollup"
    ]
  },
   :
```

あとはrollupの設定

```rollup.config.js
import babel from "rollup-plugin-babel"
export default {
  entry: "src/index.js",
  dest: "dest/index.js",
  plugins: [babel()]
}
```

うまくいった！
