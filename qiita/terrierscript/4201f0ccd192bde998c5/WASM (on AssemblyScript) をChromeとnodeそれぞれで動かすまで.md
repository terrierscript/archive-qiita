

# とりあえずとっかかり編
* 「WASM全くわからない」の状態から「とりあえずWASMが動くとはどういうことか？」ぐらいまでを知るところまでやる
* AssemblyScriptが気になってたのでAssemblyScript使う
* ブラウザで動かす話があんまり見つからなかったので、ちょっとまとめてみる。

## chromeで動かす
### 1.セットアップ

WASMをchromeで動かそうとすると、ローカルファイルのfetchがCORSで弾かれてしまうので、サーバーを立てて動かす必要がある。
（firefoxは57.0.1で確認したところローカルで動くようなので、firefoxで試す場合は不要）


htmlとWASMファイルをpublicのlocalhostでアクセスできれば良いので、`poi`とか`parcel`でも`budo`でも`webpack-dev-server`でもなんでも良いが、今回は`create-react-app`を使って進める
（若干トイレットペーパーで鼻をかむのと似た気持ちの悪さがあるが、create-react-appはsandboxとしてはなんだかんだ使いやすくて困る・・・）

```
$ npx create-react-app wasm-sample
```

### 2. AssemblyScript + wasmの設定

今回はWASMの生成として、[AssemblyScript](https://github.com/AssemblyScript/assemblyscript/)を使う。

[wiki](https://github.com/AssemblyScript/assemblyscript/wiki)はだいたい情報が古くなってしまっていたので色々ひっかかりどころがあった。

```
$ yarn add -D assemblyscript
```

### 3. tsconfig.json
tsconfig.jsonをこんな感じで設定

```json
{
  "extends": "./node_modules/assemblyscript/tsconfig.assembly.json",
  "include": [
    "./**/*.ts"
  ]
}
```

extendsのファイルはwikiでは`./node_modules/assemblyscript/std/assembly.json"`になっているが、正しくは上記だった。

### 4. scripts設定

`yarn start`前にビルドしたいので、`package.json`をこんな感じにした

```json
  "scripts": {
    "start": "react-scripts start",
    "prestart": "asc wasm/bin.ts -b > public/bin.wasm"
  }
```

`asc`についてはwikiのオプション情報が古くなっているので、`asc --help`するのが良い

### 5. AssemblyScriptを書く

```ts
// wasm/bin.ts

// 値を返すだけのスクリプト
export function echo(val: f64): f64 {
  return val;
}
```

### 6. html置き換え

html側のロード。

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
  <body>
    <script>
      fetch('./bin.wasm').then(response => response.arrayBuffer()
      ).then(bytes => {
        return WebAssembly.instantiate(bytes, {})
      }).then( r => {
        const ex = r.instance.exports
        const result = ex.echo(3)
        document.querySelector("#root").innerHTML = result
      })
    </script>
    <div id="root"></div>
  </body>
</html>
```

`fetch`で持ってきてbufferを`WebAssembly.instantiate`でインスタンス化すると、値が返ってくることが確認できる。

<img width="182" alt="localhost_3000.png" src="https://qiita-image-store.s3.amazonaws.com/0/7307/02c074db-31e5-b454-8ab4-c500f6cbab8f.png">

これがWASMか！

## nodeでも動かす。

同じwasmをnodeでも動かしてみる。

```js
// src/node.js
const fs = require("fs")
const wasm = "./public/bin.wasm"
const buffer = fs.readFileSync(wasm)

const w = WebAssembly.compile(buffer)
  .then( module => {
    const instance = WebAssembly.Instance(module, {})
    console.log(instance.exports.echo(3))
  })

```

`fs`でbuffer化してそれをcompileする。

node でもちゃんと動いた！

```
% node src/node.js
3
```

# `assemblyscript-loader`でもうちょっとかんばる編

ここまで、メモリ設定やらlib設定やら色々すっ飛ばしてきた。
このへん自前でやるとめんどくさいが、[assemblyscript-loader](https://github.com/AssemblyScript/loader) というのが提供されているのでこれを使ったパターンもやってみる。

## chromeで動かす

`require`使いたいのでhtmlに直接書くのをここからはやめて、コンパイルされる方に書く

```js
// src/index.js

const { load } = require("assemblyscript-loader")

fetch('./bin.wasm')
  .then(result => result.arrayBuffer())
  .then( buf => load(buf) )
  .then( module => {
    const ex = module.exports
    const result = ex.echo(3)
    document.querySelector("body").innerHTML = result
  })
```

本当は `load('./bin.wasm')`でいけるはずなのだが、どうも正しく動いてないので通常のfetch関数を取得するようにした。
（軽く見た感じ、assemblyscript-loaderの`xfetch`にバグがあるのかも？）

## nodeで動かす

nodeの方はめちゃくちゃシンプルになる。

```js
// node.js
const { load } = require("assemblyscript-loader")

load("./public/bin.wasm").then( module => {
  console.log(module.exports.echo(3))
})
```

# まとめ
* wasmについて、「なんかバイナリ動くんでしょ」というぼんやりしたイメージのものだったが、やってみることでちょっとだけ理解が進む。
* AssemblyScriptまだまだお試し版で実用に耐えうるかというと微妙なステージ
    * 再帰関数などもまだ処理できてないっぽい？
* とはいえnpm/yarnだけで物事が完結してくれるのでとっかかりとしてすごく良かった。
    * AssemblyScript頑張って欲しい！


