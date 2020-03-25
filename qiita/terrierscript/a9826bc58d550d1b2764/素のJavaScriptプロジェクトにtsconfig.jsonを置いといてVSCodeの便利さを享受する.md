VSCodeの便利さを使いたいがTypeScript化するほど手間かけれない、みたいなときに`tsconfig.json`だけ作っておくとちょっと便利になりそうだったのでメモる。

**追記**： TypeScriptへ移植する予定が無い場合であれば、[`jsconfig.json`](https://code.visualstudio.com/docs/languages/jsconfig)を配置するでも十分そうです（違いはallowJSがdefaultでtrueなこととぐらい。あとは`tsconfig.json`は後述のコマンドでボイラープレートを生成できるという点ぐらいと思われます）

具体的にはこのへんとか使える

* 未使用のimport検出
* ライブラリの型情報を利用した検出
* JSDocがあればそれを利用した型検証

# やり方

まず`tsconfig.json`を生成する。npx使う

```
$ npx typescript --init
```

次に、`allowJs`と`checkJs`を`true`にする

```json
"allowJs": true,
"checkJs": true,
```

あとtscを叩いて上書きなどされないようにnoEmitも追加しておく

```json
"noEmit": true
```         

ついでに`strict`も必要があれば`false`にしておく。

```json
"strict": false,
```

Reactの場合はjsx周りのエラーを回避するために下記も追加

```json
"jsx": "react"
```

また、`@types/react`も必要

```
$ yarn add -D @types/react
```

# エラーが出たら？

## 無視するシグネチャをつける方法で回避

コメントによって一部チェックを無視することが可能。

行単位で無視したいなら下記

```js
// @ts-ignore
```

ファイルごと無視したいなら下記

```js
// @ts-nocheck
```

詳しくは下記

https://github.com/Microsoft/TypeScript/wiki/Type-Checking-JavaScript-Files

## 型をつける方法で回避

JSDocを指定して型指定すると回避できる。
`@type {*}`で値をany扱いにするのが雑ではあるものの一番ライト。
当然型の恩恵は受けれなくなるものだが、今はVSCodeの恩恵を受けることを目的とするので目をつぶる。

```js
/** @type {*} */
const Mark = styled.div`
  background: ${
    /** @type {*} */
    props => props.primary ? 'palevioletred' : 'white'
  };
`
```

ちゃんと型を指定することもできるものの、ここまでやるならもうTypeScriptにしたほういいのでは？という感じはある。

```js
import styled, { ThemedStyledProps, StyledComponentClass } from "styled-components"

/**
 * @type {StyledComponentClass<any, any>}
 */
const Mark = styled.div`
  background: ${
    /** @type {ThemedStyledProps<{ primary: Boolean}, {}> } */
    props => props.primary ? 'palevioletred' : 'white'
  };
`
```

さらにJSDoc周りについての詳細は公式の下記を見ると良い。
https://github.com/Microsoft/TypeScript/wiki/JSDoc-support-in-JavaScript

