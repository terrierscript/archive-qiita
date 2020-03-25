いい記事の投稿ありがとうございます。
さて、１のサンプルコードの部分で、

```js
const mockAdapter = (config) => {
  return new Promise((resolve, reject) => {
    resolve({data: mockData, status: 200 })
  })
}
```

というのがありますが、これは以下のほうがコード量が短いです。

```js
const mockAdapter = (config) => {
  return Promise.resolve({data: mockData, status: 200 });
}
```

`new Promise((resolve, reject) => {})`内は戻り値を認識しないため、個人的にはあまり頻繁に使うべきではないと思ってます。
使うしかない場面は存在するのですが、その場合も極力小さくしてほしかったりします。
