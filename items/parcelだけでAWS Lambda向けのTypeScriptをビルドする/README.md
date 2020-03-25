AWS Lambda FunctionでTypeScriptを使いたい場合、通常Serverless frameworkやapexを使うのが一般的だろう。
しかし、ちょこっと使いたいだけの場合に重苦しい。

もしかしたらParcelだけでいけるのでは？と思ったらいけたのでメモ


# 手順

まずはTypeScriptインストール

```
yarn add -D typescript
```

parcelも入れる。zip化するとアップロードも楽になるのでpluginも入れる

```
yarn add -D parcel parcel-plugin-zip
```

そしてfunction本体。今回は`handler.ts`という名前にしておく

```typescript
export const handler = async (): Promise<any> => {
  console.log('Hello World!');
  return {};
}
```

これだけでもうビルドはできる。下記のようなコマンドだ

```
$ yarn parcel build handler.ts --target=node --global=handler -o index.js --bundle-node-modules --no-source-maps
```

それぞれ引数は下記のようなことをしている

* `--target=node` で対象をnodeに
* `--bundle-node-modules`でnode_modulesからの読み込み物を追加
* sourcemapは不要なので`--no-source-map`
* `-o index.js`で出漁対象をindex.jsにへこう
* `--global=handler` でhandlerに外からアクセスできる形でビルド（ちゃんと検証してないけどもしかしたら不要かも？）


これを`package.json`に仕掛ければおしまい

```json
"scripts": {
  "build": "yarn parcel build handler.ts --target=node --global=handler -o index.js --bundle-node-modules --no-source-maps",
  "deploy:aws": "aws lambda update-function-code --function-name my-special-function --zip-file fileb://./dist.zip --region=ap-northeast-1",
  "deploy": "yarn build; yarn deploy:aws",
}
```

上記では`aws`コマンドを使ってコードをアップロードするところまでやっている（ここについては今回説明は割愛する）

あとは`yarn deploy`でビルドまでできる

# 参考

* https://scotch.io/@nwayve/how-to-build-a-lambda-function-in-typescript
* https://github.com/parcel-bundler/parcel/issues/192#issuecomment-366032738
