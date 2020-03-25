request処理をするときに使い勝手が良くて気にっている[axios](https://github.com/mzabriskie/axios/)をそこそこ使うようになって溜まった知見。

---

# 1. `adapter`を使って簡易mocking

リクエスト関連は結構テストが辛くなりがち。
このあたりを解決する方法として、[nock](https://github.com/node-nock/nock)などリクエスト系をサードパーティ化するのがよくあるが、
axiosの場合、adapter機能を利用するとmockを簡単につくれる

```js
const axios = require('axios')

const mockData = [{
  id: 1,
  title: 'Some Article',
}]

const mockAdapter = (config) => {
  return new Promise((resolve, reject) => {
    resolve({data: mockData, status: 200 })
  })
}

axios.get('/some/url', {
  adapter: mockAdapter
}).then( response => {
  console.log(response.data) // mockData
  console.log(response.status) // 200
})
```

自前したくない場合は[moxios](https://github.com/mzabriskie/moxios)とか[axios-mock-adapter](https://github.com/ctimmerm/axios-mock-adapter)とかもある。


# 2. 基本configを固定するやり方 ( `axios.create()`, `axios.defaults`)

例えば様々なAPIで同じような設定をしたいことがよくあると思う。

```js
axios.get('/some/url', { 
  xsrfHeaderName: 'X-CSRF-Token',
  withCredentials: true
})
axios.post('/some/other/url',{foo: 'bar'}, { 
  xsrfHeaderName: 'X-CSRF-Token',
  withCredentials: true
})
```

このような場合には、[`axios.create()`](https://github.com/mzabriskie/axios#creating-an-instance)を利用して別インスタンスとして生成してしまうのが良い

```js
const myHttpClient = axios.create({ 
  xsrfHeaderName: 'X-CSRF-Token',
  withCredentials: true
})
myHttpClient.get('/some/url') // 設定が反映されてリクエストがトブ
```

また、別な方法として、globalな設定として[`defaults`](https://github.com/mzabriskie/axios#global-axios-defaults)で設定することも可能だ

```js
axios.defaults.withCredentials = true
axios.defaults.xsrfHeaderName =  'X-CSRF-Token'
```

`timeout`など共通的な設定で十分なものだけで済むなら使って良さそうだろう。

defaults機能はaxiosが影響を受けたangular1の`$http`に由来するものと思われる。
angularのようにシステムでひとまとめにするフレームワーク上では都合が良いことが多かったと想像出来るが、`axios`で利用するシーンはそうではない場合が多いと想像出来るので、個人的には`axios.create`のほうがおすすめできる

# 3. 処理差し込み

## 3-1. `interceptors`で処理差し込み

例えば[normalizr](https://github.com/paularmstrong/normalizr)と併用して、API結果を変換した状態にしたいならこんな具合に利用出来る。

```js
// ユーザーデータを変換するAPI
const userApi = axios.create()

userApi.interceptors.response.use( (response) => {
  response.data = normalize(res.data, article)
  // response.dataを上書きしたくなければ、下記の用にしても良いだろう
  // response.normalized = normalize(res.data, article)
  return res
})

userApi.get('/foo')
  .then( r => console.log(r.data)) // normalizedしたデータが取れる
```

interceptorは`request`,`response`が用意されている。
`use`は複数追加することも出来るので、何か処理をわけたいなら下記のようにも書ける

```js
// request
userApi.interceptors.request.use( (config) => {
  // configには送信前のdataやurlとかが入っている
    :
  return config
})

// response
userApi.interceptors.response.use( (response) => {
  /* 処理ひとつめ */
    :
  return response
})
userApi.interceptors.response.use( (response) => {
  /* 処理ふたつめ */
    :
  return response
})
```

また、`intercept`をejectする機能も用意されている

```js
var myInterceptor = axios.interceptors.request.use(function () {/*...*/});
axios.interceptors.request.eject(myInterceptor);
```

## 3-2. `transformResponse` / `transformRequest`で処理差し込み

`transformResponse` / `transformRequest` はそれぞれinterceptよりもリクエストに近いタイミングで処理される。
`transformRequest`は送信の形式json以外の特殊な場合などに利用すると良いだろう。

`transformResponse`は主に文字化け対策などで利用できる

```js
// 文字化け対処の例
axios(url, {
  responseType: "arraybuffer",
  transformResponse: [ (data) => {
    return iconv.convert(data, 'EUCJP', 'UTF8').toString()
  }]
})
```


## おまけ：それぞれどういう順序で行われるのか検証する

こんなスクリプトで調べる

```js
const axios = require('axios')

const api = axios.create({
  adapter: function(config){
    console.log("adapter")
    return new Promise((resolve, reject) => {
      console.log("adapter(promise)")
      resolve({data: config.data})
    })
  },
  transformRequest: [(data, header) => {
    console.log("transformRequest")
    return data
  }],
  transformResponse: [(data) => {
    console.log("transformResponse")
    return data
  }]
})

api.interceptors.request.use((config) => {
  console.log("interceptors.request")
  return config
})
api.interceptors.response.use((response) => {
  console.log("interceptors.response")
  return response
})
api.post('/foo', { mock: "data"})
```

結果がこちら

```
$ node axios-interceptor.js
interceptors.request
transformRequest
adapter
adapter(promise)
transformResponse
interceptors.response
```

`interceptor` => `transform` => `adapter`という形で処理が行われている事がわかる。

# 4. 複数APIの並列処理 (`axios.all`, `axios.spread`)

`axios.all`と`spread`を利用して並列処理に扱える。

```js
const axios = require('axios')
const api = axios.create()

axios.all([
  api.get('/my/api'),
  api.get('/my/api2')
]).then( axios.spread( (api1Result, api2Result) => {
  //
})
```

但し、`axios.all`は`Promise.all`のラッパー、`axios.spread`は配列を引数化しているだけなので、ES2015のDestructuring構文を組み合わせると実はそちらでも十分出来てしまう。

```js
const axios = require('axios')
const api = axios.create()

Promise.all([
  api.get('/my/api'),
  api.get('/my/api2')
]).then( ([api1Result, api2Result]) => {
  //
})
```

# まとめ
* 色々使い方覚えると、結構使い勝手が変わるのでオススメ
    * 特に`axios.create`や`interceptors`などは結構違う
* その他にも[Cancel](https://github.com/mzabriskie/axios#cancellation)機能やら[form形式での送信](https://github.com/mzabriskie/axios#using-applicationx-www-form-urlencoded-format)など結構READMEに色々書いてある
