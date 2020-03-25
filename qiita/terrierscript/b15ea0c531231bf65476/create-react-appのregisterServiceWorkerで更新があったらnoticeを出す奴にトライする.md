`create-react-app` には `registerServiceWorker.js` というservice workerを登録する仕組みが乗っかっている。

これはoffline cacheを実現するものなのだが、cache-first strategyという手法を取っており、コードを更新してデプロイした際の取り回しがなかなか手こずるので、割といつもdisableにしがち。

だが、いつまでもこのままというのもちょっと気持ち悪いので、一度service-workerの更新に合わせて「更新してください」みたいなモーダルを出すexampleをやってみた。


---
![demo](https://qiita-image-store.s3.amazonaws.com/0/7307/efb15ef8-ac36-d878-033b-5c78e8c127a3.gif)

[ソース](https://github.com/inuscript/example-service-worker.git)

---

## 仕組みおさらい

create-react-appは [sw-precache-webpack-plugin](https://github.com/goldhand/sw-precache-webpack-plugin/)というpluginを組み込んでserviceWorkerを組み込んでいる。

このpluginはwebpack上でマニフェスト的な値を生成し、それと[sw-precache](https://github.com/GoogleChromeLabs/sw-precache)を組わせて裏側でfetchする仕組みをもたせている。


```js
// service-worker.js
 :
// ここらへんの値をwebpackから吐いている。
var precacheConfig = [["/index.html","08af9741a3d97d13dec114a974062844"],["/static/css/main.c17080f1.css","302476b8b379a677f648aa1e48918ebd"],["/static/js/main.0ccb3c41.js","426e3841d0fed136b6bb86108f30e5e2"],["/static/media/logo.5d5d9eef.svg","5d5d9eefa31e5e13a6610d9fa7a283bb"]];

// :
// あとはsw-preacacheの生成コード
 
```

そして、これを `registerServiceWorker.js`が登録することでキャッシュが動いている。

```js
// registerServiceWorker.js
  :
  :
      const swUrl = `${process.env.PUBLIC_URL}/service-worker.js`;
      if (isLocalhost) {
        // This is running on localhost. Lets check if a service worker still exists or not.
        checkValidServiceWorker(swUrl);
```

## 1. 準備

`create-react-app`の標準状態だと、hot reloadがあったりしてlocalでのservice workerの検証がしづらい。

なので`eject`して、加えて[`serve`](https://github.com/zeit/serve)でローカルサーバーで実行する。 (serveはcache-controlを0にする必要もある)

```js
{
  "scripts": {
    "start":
      "NODE_ENV=production webpack --config config/webpack.config.prod.js --watch",
    "static": "serve -c 0 -s build -o --local"
  }
}
```

これでこんな感じでstart。

```
$ yarn start & yarn static
```

## 2. `onupdatefound`からイベントを発火

ServiceWorkerが更新を検知した際、registerServiceWorkerの[`onupdatefound`](https://github.com/facebook/create-react-app/blob/0b1d6365768ae3bd267b042b74bab249673f1a9f/packages/react-scripts/template/src/registerServiceWorker.js#L63-L69) が発火するので、ここにevent発火を仕掛ける。

```js
registration.onupdatefound = () => {
  const installingWorker = registration.installing;
    installingWorker.onstatechange = () => {
        if (navigator.serviceWorker.controller) {
          // At this point, the old content will have been purged and
          // the fresh content will have been added to the cache.
          // It's the perfect time to display a "New content is
          // available; please refresh." message in your web app.
          console.log("New content is available; please refresh.");
        
          // イベント発火
          const event = new Event("newContentAvailable");
          window.dispatchEvent(event);
        } else {
           :
```

ちなみに、[nextバージョン](https://github.com/facebook/create-react-app/blob/cb3f835586d2f1e3b9ddcf3a29442c9d76eaef84/packages/react-scripts/template/src/serviceWorker.js#L71-L73) では、`config.onUpdate` というコールバックが追加されており、`registerServiceWorker.js`に直接手を加えなくても大丈夫になりそう


## 2. `<ReloadModal>`を作る

次に `<ReloadModal>`を作る。これは発火されたグローバルな `newContentAvailable` イベントを拾ってくるためのもの。
この例では、わかりやすさ重視でstate使ったり、ライフサイクルをそのまま使っているので、必要に応じてreduxにしたりなにかしたりすると良さそう。

```jsx

// Main App
class App extends Component {
  render() {
    return (
      <div className="App">
        <ReloadModal />
            :
      </div>
    );
  }
}

class ReloadModal extends Component {
  state = {
    show: false
  };
  componentDidMount() {
    // グローバルイベントを引っ掛ける。
    window.addEventListener("newContentAvailable", () => {
      this.setState({
        show: true
      });
    });
  }
  onClick = () => {
    // リロードする
    window.location.reload(window.location.href);
  };
  render() {
    if (!this.state.show) {
      return null;
    }
    // <Modal> は単なる固定モーダルのコンポーネントなので省略。
    return (
      <Modal onClick={this.onClick}>
        <span> New Content Available!please reload </span>
      </Modal>
    );
  }
}
```

## まとめ

とりあえずこれで最初に出したデモみたいなものは出来た。
今回のは若干怪しいと自分でも思っているので、もっと正しいやり方があったら知りたい。

また、なんとなく次期バージョンだともうちょっと扱いやすくなってそうなので、それまで待ってもいいのかもしれない。

