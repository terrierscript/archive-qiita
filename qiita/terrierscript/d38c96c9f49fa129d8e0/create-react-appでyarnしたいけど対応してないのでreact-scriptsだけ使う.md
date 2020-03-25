
create-react-appが素振りに最適なのだけど、電車の中などyarnで行いたい。
けど今のところ[対応前なので](https://github.com/facebookincubator/create-react-app/issues/896)で、「`create-react-app`時にinstallはしない」、みたいなことも出来なそうなので、react-scriptsだけ美味しく使うことをやってみる。

今回は「start使えればいいや」ぐらいな気持ちなので、testなどは試していない。

---

## 1. 普段通りのプロジェクト作成
```
$ mkdir example-yarn-react
$ cd example-yarn-react
$ git init
```

npm init -yでもいいんだけど、せっかくなのでyarn使う

```
$ yarn init -y
```

## 2. 必要パッケージのinstall
`react-scripts`を入れる。`react-script`という類似のパッケージがあるが、全く別物なので注意

```
$ yarn add --dev react-scripts --prefer-offline
```

reactとreact-domも入れる

```
$ yarn add react react-dom --prefer-offline 
```

yarnさんも初回は時間がかかるのでじっくり待つ。。。

## 3. scripts部分を追加

`package.json`のscripts部分の追加。手打ちだけどしょうがない。
個人的には素振り目的なのでstartだけ入れる。

```js
  "scripts": {
    "start": "react-scripts start"
  }
```

フルならこんな感じ。[詳細はこのへん](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#available-scripts)

```js
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject"
  }
```

## 4. 起動のためのpublic/index.htmlファイル準備

```
$ mkdir public
$ touch public/index.html
```

index.htmlの最低限はこのぐらいな感じ。

```html
<html>
  <body>
    <div id="root"></div>
  </body>
</html>
```

## 5. srcファイル準備

```
$ mkdir src
$ touch src/index.js
```

中身はこんな感じ

```js
import React from 'react'
import ReactDOM from 'react-dom'

ReactDOM.render(
  <div>Hello</div>,
  document.getElementById('root')
)
```

## 6. 起動!!!
ここもせっかくなのでyarn使う

```
$ yarn start
```

起動した！

# 結論
* めんどくさい！！旨味半減
* 旨味半減だけど、半分ぐらいは旨い。のでそんなに悪くはない
    * webpack書かなくていいとかbabel設定しなくていいとかの部分で結構楽。
    * budoセットアップするのより気持ち楽？ぐらい。
* shell化するとかscript化するとかで改善できそうだけど、そこまでやる前に本体側が対応されそう

