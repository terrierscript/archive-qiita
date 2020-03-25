vuejsではvue2-google-mapなどgoogle-mapを利用するライブラリがあるが、色々小回りを効かせたいくて直接APIを触れるようにしたかったのでやってみたらvueのあらゆる機能を触ることになったので実装をまとめてみる。

# 出来たもの
Demo: https://vue-google-map-provider-sample.netlify.com/
Source: https://github.com/inuscript/example-vue-inject-provide-google-map

# 実装

登場人物は下記のようになる

* index.html (起動するファイル）
* MyMap.vue (マップの大本の実装。コンポーネントを組み合わせる親）
* MapLoader.vue (Google Mapを呼び出すだけ）
* MapProvider.vue (vueのprovideを提供するだけ）
* ChildMarker.vue (例として子要素でマーカーを表示する）

google-map以外でもcallbackを利用するような場合に扱える知見を得ることが出来たので、ここから実装について書いてみる。

## index.html

```html
    <!-- index.html -->
    <div id="app">
      <my-map :markers='[
        {"lat":35.6432027,"lng":139.6729435},
        {"lat":35.5279833,"lng":139.6989209},
        {"lat":35.6563623,"lng":139.7215211},
        {"lat":35.6167531,"lng":139.5469376},
        {"lat":35.6950961,"lng":139.5037899}
      ]'>
      </my-map>
    </div>
```

index.htmlは呼び出すだけなので何も無し


## MyMap.vue

呼び出しの中核はこんな感じにしてみた。
ここは汎用的に使えないものとして書いている。

```html
<!-- MyMap.vue -->
<template>
  <div>
    <h1>Map</h1>
    <map-loader 
      :map-config="mapConfig"
      apiKey="YOUR API KEY"
    >
      <template v-for="marker in markers">
        <child-marker :position="marker" />
      </template>
    </map-loader>
  </div>
</template>

<script>
import MapLoader from "./MapLoader.vue"
import ChildMarker from './ChildMarker'

export default {
  props: {
    markers: Array
  },
  data(){
    return {
      mapConfig: {
        zoom: 12,
        center: this.markers[0]
      }
    }
  },
  components: {
    MapLoader,
    ChildMarker
  }
}
</script>
```

ここから`map-loader`を呼び出し、内部で`child-marker`を利用している


## MapLoader

肝の部分その１。google mapをロードを担う

```html
<!-- MapLoader.vue -->
<template>
  <div>
    <div id="map"></div> <!-- point 1 -->
    <template v-if="!!this.google && !!this.map"> <!-- point 2 -->
      <map-provider
        :google="google"
        :map="map"
      >
        <slot/>
      </map-provider>
    </template>
  </div>
</template>

<script>
import GoogleMapsApiLoader from 'google-maps-api-loader'
import MapProvider from './MapProvider'

export default {
  props:{
    mapConfig: Object,
    apiKey: String
  },
  components: {
    MapProvider
  },
  data(){
    return {
      google: null,
      map: null
    }
  },
  mounted () { // point 3
    GoogleMapsApiLoader({
      apiKey: this.apiKey
    }).then((google) => {
      this.google = google
      this.initializeMap()
    })
  },
  methods: {
    initializeMap (){
      const mapContainer = this.$el.querySelector('#map') // point 1
      const { Map } = this.google.maps
      this.map = new Map(mapContainer, this.mapConfig)
    }
  }
}
</script>

<style scoped>
#map {
  height: 100vh;
  width: 100%;
}
</style>
```

* point 1: `#map` google mapを吐き出す部分としてdomを使っている。それを後半で`this.$el`を利用してマウントしている。
* point 2: 一番のキモ。`google`と`map`がロード完了するまで子を **レンダリングさせない**。この制御をすることで、provide/injectをうまく利用出来る。
* point 3: ロードの処理どのタイミングでやるの？というのは`mounted`で行う。ここで`GoogleMapApiLoader`



## MapProvider
キモその２。
vueのprovide/injectを利用するために、loaderから分離した。
provideはコンポーネントがロードされた初期のタイミングでのみ評価されるらしく、loaderで値の算出が完了した後に実行されるようにしないといけない。
loaderが`google`と`map`が揃うまで子要素をレンダリングしないようにすることで、`MapProvider`は確実に`google`と`map`の値を取得出来る状態になっている。

```html
<!-- MapProvider.vue -->
<template>
  <div>
    <slot />
  </div>
</template>

<script>
export default {
  props: {
    google: Object,
    map: Object,
  },
  provide() {
    return {
      google: this.google,
      map: this.map
    }
  },
}
</script>
```

## ChildMarker

```html
<!-- ChildMarker.vue -->
<template></template>
<script>
export default {
  inject: ["google", "map"],
  props: {
    position: Object
  },
  data(){
    return { marker: null}
  },
  mounted(){
    const { Marker } = this.google.maps
    this.marker = new Marker({
      position: this.position,
      map: this.map,
      title: "Child marker!"
    })
  }
}
</script>

```

子要素の実装側。
`inject`を利用して`provide`された値を貰ってくる。
`MapLoader`でやったのと同じく、`mounted`で処理をする。表示は何もしなくて良いので空にしている。


# provide/injectを使わないパターン（slot-scopeを使う）

provide / injectは公式ドキュメントによれば「ライブラリ向けで、一般ユーザーは使うべきでない」という話もある。
これらを使わない場合は、値をバケツリレーして子要素に渡していくことになる。
ここでvueの場合は`slot-socpe`を利用する必要が出てくる


MapLoaderはMapProvierを利用せず、`slot`に直接`google`と`map`の値を渡す

```html
<!-- MapLoader.vue -->
<template>
  <div>
    <div id="map"></div>
    <template v-if="!!this.google && !!this.map">
      <slot
        :google="google"
        :map="map"
      />
    </template>
  </div>
</template>

<script>
 :
</script>
```

そしてMyMap側で`slot-scope`を利用して、`map-loader`から受け取った`google`と`map`の変数を`child-marker`へ渡す

```html
<!-- MyMap -->
<template>
  <div>
    <h1>Map</h1>
    <map-loader 
      :map-config="mapConfig"
      apiKey="YOUR API KEY">
      <template slot-scope="scopeProps"> <!-- slot-scope -->
        <child-marker 
          v-for="(marker,i) in markers"
          :key="i"
          :position="marker" 
          :google="scopeProps.google"
          :map="scopeProps.map"
        />
      </template>
    </map-loader>
  </div>
</template>
```

ChildMarkerは`inject`がなくなり`props`で値を受けるようになる

```html
<!-- ChildMarker -->
<template></template>
<script>
export default {
  props: {
    google: Object,
    map: Object,
    position: Object
  },
  data(){
    return { marker: null }
  },
  mounted(){
    const { Marker } = this.google.maps
    this.marker = new Marker({
      position: this.position,
      map: this.map,
      title: "Child marker!"
    })
  }
}
</script>
```


