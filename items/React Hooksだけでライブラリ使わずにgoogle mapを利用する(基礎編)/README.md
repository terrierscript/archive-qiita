[Vue.jsでライブラリ使わずにgoogle mapを利用する](https://qiita.com/terrierscript/items/9a9dda5a5ca5b3293d48) のをReact Hooksでやってみたらすぐ書けてびっくりしたのでまとめる

今回は基礎編なので、`useState`, `useEffect`を中心に使う。本当は[Basic Effect](https://reactjs.org/docs/hooks-reference.html#basic-hooks)のみでやろうと思ったが、`useRef`はどうしても必要だったので今回利用している

ライブラリを使わず、と言いつつgoogle-maps-api-loaderを利用しているのはご了承いただきたい

# 1. まずMapの表示だけする

全体のコードがこんな感じ。
これで表示までできる。
`useGoogleMap`と`useMapEffect`という２つのhooksをここまでで作っている

```jsx
// hooks.ts

import { useEffect, useState } from "react"
import GoogleMapsApiLoader from "google-maps-api-loader"
const API_KEY = "XXXXXXXXXXXX"

// Google Mapのオブジェクトを呼び出すだけのhooks
export const useGoogleMap = (apiKey) => {
  const [googleMap, setGoogleMap] = useState(null)
  useEffect(() => {
    GoogleMapsApiLoader({ apiKey }).then((google) => {
      setGoogleMap(google)
    })
  }, []) // useEffectの第二引数を[]にすることで、初回1回目だけ実行される
  return googleMap
}

// 実際にMapをDOMにマウントする処理。
export const useMap = ({ googleMap, mapContainerRef, initialConfig }) => {
  const [map, setMap] = useState(null)
  useEffect(() => {
    // googleMapかmapContainerRefが初期化されてなければ何もしない
    if (!googleMap || !mapContainerRef.current) {
      return
    }
    const map = new googleMap.maps.Map(mapContainerRef.current, initialConfig)
    setMap(map)
  }, 
  // googleMapかmapContainerRefが変化したらeffectが発火する。
  // initialConfigは変わったとしても再マウントするとおかしなことになるので更新対象にしない
  [googleMap, mapContainerRef])

  // あとで使えるようにmapを返すようにする
  return map
}

```

```jsx
import React from "react";
import { useGoogleMap, useMap } from "./hooks";
import { useRef } from "react";

const API_KEY = undefined

const initialConfig = {
  zoom: 12,
  center: { lat: 35.6432027, lng: 139.6729435 }
}
// hookを利用して表示するコンポーネント
export const MapApp = () => {
  const googleMap = useGoogleMap(API_KEY)
  const mapContainerRef = useRef(null)
  useMap({ googleMap, mapContainerRef, initialConfig })
  return (
    <div
      style={{
        // ホントはstyled-componentsとかで良いのだけど簡略化
        height: "100vh",
        width: "100%"
      }}
      ref={mapContainerRef}
    />
  )
}

ReactDOM.render(<MapApp />, document.querySelector("#root"))

```

### DEMO
デモがこちら(APIキーを設定してないのでdevモード）
https://codesandbox.io/s/zx060qr7q4

### 注釈：NGなhooksの書き方

ここで注意するとすれば例えば下記のような書き方はしてはいけない。

```tsx
const useMapEffect = ({ googleMap, mapContainerRef, mapConfig }) => {
  // ❌ こう書くとhooksは動かない
  if (!googleMap || !mapContainerRef.current) {
    return
  }
  useEffect(() => {
    const { Map } = googleMap.maps
    new Map(mapContainerRef.current, mapConfig)
    // 第二引数のいずれかが変更されたら再度処理される
  }, [googleMap, mapContainerRef, mapConfig])
}

const useMapEffect = ({ googleMap, mapContainerRef, mapConfig }) => {
  // ❌ これもNGなパターン
  if (googleMap && mapContainerRef.current) {
    useEffect(() => {
      const { Map } = googleMap.maps
      new Map(mapContainerRef.current, mapConfig)
      // 第二引数のいずれかが変更されたら再度処理される
    }, [googleMap, mapContainerRef, mapConfig])
  }
}
```


Hooksはトップレベルでしか呼び出せず、ifやループの内部で呼ぶこともNGとなる。詳しくは下記にルールがある。
https://reactjs.org/docs/hooks-rules.html

[eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks)を使えばこれらエラーを検出してくれる。

ついでにいうと[hooksは`use`をprefixにしましょうという決まりごとがある](https://reactjs.org/docs/hooks-custom.html#extracting-a-custom-hook)。
たとえば下記のようにhooksを利用しているにもかかわらず`use`から始まらないというのも避けるべきだ。これらが無いとlinterもうまく動いてくれない
（逆にhooksが関係ない`use`から始まる関数がlinterに誤検知される可能性もありそうな気がする？）

```js
// ❌ hooksなのに`use`から始まってないのもNG。ESLintで正しく検出できなくなる
const myAwesomeHook = (...) => {
  useEffect()
}

// 下記のようにhooksを生成する関数はLinter的にはOK。でもこのへん使い出すと可読性がヤバくなりそう（主観）
function createHook() {
  return function useHookWithHook() {
    useHook();
  }
}
```

どんなものがOKでどんなものがNGかは[テストケース](https://github.com/facebook/react/blob/master/packages/eslint-plugin-react-hooks/__tests__/ESLintRulesOfHooks-test.js)のファイルを見ると色々書いてある


# 2. マーカーを表示する

ここに`googleMap.maps.Marker`を利用してマーカーを表示する

```js
// markerを追加するhooksを追加
export const useMapMarker = ({ markers, googleMap, map }) => {
  useEffect(() => {
    // 初期化がまだなら何もしない
    if (!googleMap || !map) {
      return
    }
    const { Marker } = googleMap.maps
    const mapMarkerObj = markers.map(
      (position) =>
        new Marker({
          position,
          map,
          title: "marker!"
        })
    )
  }, [googleMap, map])
}
```
ちなみに上記hooksはmarkersが変更は検知しないようにしている。markerが重なって多重に描画されることを避けるためだ。この対処については後述する

呼び出し側はこんな感じに呼び出す

```jsx
const markers = [
  { lat: 35.6432027, lng: 139.6729435 },
  { lat: 35.5279833, lng: 139.6989209 },
  { lat: 35.6563623, lng: 139.7215211 },
  { lat: 35.6167531, lng: 139.5469376 },
  { lat: 35.6950961, lng: 139.5037899 }
]

export const MapApp = () => {
  const googleMap = useGoogleMap(API_KEY)
  const mapContainerRef = useRef(null)
  const map = useMap({
    googleMap,
    mapContainerRef,
    initialConfig
  })
  // markerを渡す
  useMapMarker({ markers, googleMap, map })
  return (
    <div
      style={{
        height: "100vh",
        width: "100%"
      }}
      ref={mapContainerRef}
    />
  )
}
```

### DEMO
https://codesandbox.io/s/wknx1j1vxk

# 応用編
下記に続きます
https://qiita.com/terrierscript/items/cebc4a5185c65547715c
