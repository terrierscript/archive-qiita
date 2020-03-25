
こちらは[React Hooksだけでライブラリ使わずにgoogle mapを利用する(基礎編)](https://qiita.com/terrierscript/items/4a50ff418a7d425f775d)の続きになります。

---

Google MapのReact Hooksでの利用。前回マップを出すとこまでをとしてまとめた。
ここからは応用編としていろんな手段を書いて行きたい。

* 3. クリックしたらマーカーを増やすようにする
* 4. クリックしたら削除もするようにする
* 5. InfoWindowを出す

応用のため遠慮せずすべてのhooksを色々利用していく。
なるべく説明は書いているが不足を感じたら公式ドキュメントをご参照いただきたい
https://reactjs.org/docs/hooks-reference.html

また要所要所でリファクタを挟んでいる。
コードは前回の続きからとしてご覧いただきたい。

# 3. クリックしたらマーカーを増やすようにする

クリックされたらマーカーを増やすようなことを試してみる。

追加するhooksは下記の２つになる

```js
// hooks.js

// markerをstate管理する
export const useMarkerState = (initialMarkers) => {
  const [markers, setMarkers] = useState(initialMarkers)
  // マーカーの追加処理はsetMarkersを加工する形に
  const addMarker = ({ lat, lng }) => {
    setMarkers([...markers, { lat, lng }])
  }
  return {
    markers,
    addMarker
  }
}

// Mapがクリックされたらイベントを飛ばすhooks
export const useMapClickEvent = ({ onClickMap, googleMap, map }) => {
  useEffect(() => {
    if (!googleMap || !map) {
      return
    }
    
    const listener = googleMap.maps.event.addListener(map, "click", (e) => {
      onClickMap({
        lat: e.latLng.lat(),
        lng: e.latLng.lng()
      })
    })
    // onClickMapが変更されたらつくったイベントをクリアする
    //（じゃないとクリックするたびにイベントが大量閣下さえる）
    return () => {
      googleMap.maps.event.removeListener(listener)
    }
  }, [googleMap, map, onClickMap])
}

```

また、`useMapMarker`も書き換える。
markerが多重描画されないように、markerの実体を保存する。
stateで保持しようとすると遅延アップデートがされる都合でうまく保持がされないため

```js
export const useDrawMapMarkers = ({ markers, googleMap, map }) => {
  // stateだと初回描画ほ保持がうまくいかないのでここではrefを利用する
  const markerObjectsRef = useRef({})
  
  // markersが変わるたびに実行する
  useEffect(() => {
    // 初期化がまだなら何もしない
    if (!googleMap || !map) {
      return
    }
    const { Marker } = googleMap.maps
    markers.map((position, i) => {
      if (markerObjectsRef.current[i]) {
        // すでに描画済みだったら何もしない。
        return
      }
      const markerObj = new Marker({
        position,
        map,
        title: "marker!"
      })
      markerObjectsRef.current[i] = markerObj
    })
  }, [markers, googleMap, map])
}
```

addMarkerのところは`useCallback`を利用してもいいだろう

```js
const addMarker = useCallback(({ lat, lng }) => {
  setMarkers([...markers, { lat, lng }])
}, [markers]) // markersが変更したら関数自体を変える。そうしないとstateの状態とmapの実体がずれてくる
```

そして利用する側がこんな具合になるだろう。

```jsx
export const MapApp = () => {
  const googleMap = useGoogleMap(API_KEY)
  const mapContainerRef = useRef(null)
  const map = useMap({
    googleMap,
    mapContainerRef,
    initialConfig
  })

  // stateとして管理するマーカー
  const { markers, addMarker } = useMarkerState(initialMarkers)

  // 描画する
  useDrawMapMarkers({ markers, googleMap, map })
  // クリックイベントを追加
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })

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

## 3をちょっとリファクタ

ちょっとMapAppが膨れてきてしまったのでリファクタしてみる。
MapをマウントするコンテナのCSS部分をstyled-components化するのとマーカーのhooksを利用するだけのコンポーネントで分離する

```jsx
import styled from "styled-components"

// コンテナのCSS部分をstyled-componentsにする
const MapContainer = styled.div`
  height: 100vh;
  width: 100%;
`

// マーカーのhooksを利用する
const MapMarkers = ({ googleMap, map }) => {
  // stateとして管理するマーカー
  const { markers, addMarker } = useMarkerState(initialMarkers)
  // 描画する
  useMapMarker({ markers, googleMap, map })
  // クリックイベントを追加
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })
  // hooksのためだけのコンポーネントになるのでこのコンポーネント自体は何も返さない。
  // nullを返すのが気持ち悪ければ`<script />`, `[]`, `""`を返すなどもアリ
  return null
}

export const MapApp = () => {
  const googleMap = useGoogleMap(API_KEY)
  const mapContainerRef = useRef(null)
  const map = useMap({
    googleMap,
    mapContainerRef,
    initialConfig
  })

  return (
    <>
      <MapContainer ref={mapContainerRef} />
      <MapMarkers googleMap={googleMap} map={map} />
    </>
  )
}
```

hooksを利用するだけのコンポーネントがはて良いものかというのは少し悩むところにも感じる...

hooksの部分を独自に切り出して下記のようにするのも良いだろう

```jsx
// hooksを単純に呼び出してるhooks
const useMapMarkerSetup = ({ googleMap, map }) => {
  const { markers, addMarker } = useMarkerState(initialMarkers)
  useMapMarker({ markers, googleMap, map })
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })
}
const MapMarkers = ({ googleMap, map }) => {
  useMapMarkerSetup({ googleMap, map })
  return null
}
```

また、例えばこんな風にMapが初期化されるまで待つようなコンポーネントを作る事もできるだろう

```jsx
const WaitForMap = ({ googleMap, map, children }) => {
  if (!googleMap || !map) {
    return null
  }
  return children
}
export const MapApp = () => {
  const googleMap = useGoogleMap(API_KEY)
  const mapContainerRef = useRef(null)
  const map = useMap({
    googleMap,
    mapContainerRef,
    initialConfig
  })
  return (
    <>
      <MapContainer ref={mapContainerRef} />
      <WaitForMap googleMap={googleMap} map={map}>
        <MapMarkers googleMap={googleMap} map={map} />
      </WaitForMap>
    </>
  )
}
```


```js
// hook.js
export const useMapMarker = ({ markers, googleMap, map }) => {
  useEffect(() => {
    // このチェックが不要になる
    // if (!googleMap || !map) {
    //   return
    // }
    const { Marker } = googleMap.maps
    // ...
```

### DEMO
https://gist.github.com/terrierscript/d8c9665f1a1761c48ee3739c0350ab76

# 4. クリックしたら削除もするようにする
更に応用。クリックされたら削除されることを考えてみる。ちょっとここからはだいぶ複雑度が増してくる。

ここでは下記２つの方法を示す

* A: markersを配列として描画する
* B: markersを一つずつのコンポーネントとして処理する

## 共通部分: Stateをreducer化する
ここまでマーカーは配列で処理してきたが、削除のことまで考えるとobjectで暑かったほうが都合が良くなるので`useReducer`を利用してreducer化する。


```js
import uuid from "uuid/v4"

const markerReducer = (state, action) => {
  switch (action.type) {
    case "ADD":
      const id = uuid() // 追加するたびにuuidをidとして発行
      return {
        ...state,
        [id]: action.payload
      }
    case "REMOVE":
      const { [action.payload]: removeItem, ...rest } = state
      return rest
    default:
      return state
  }
}

// 初期データもreducerを通してあげたいので、initializerを作成
const mapReducerInitializer = (initialMarkers) => {
  return initialMarkers.reduce((state, marker) => {
    return markerReducer(state, {
      type: "ADD",
      payload: marker
    })
  }, {})
}

// markerをstate管理する
export const useMarkerState = (initialMarkers) => {
  const [markers, dispatch] = useReducer(
    markerReducer,
    initialMarkers,
    mapReducerInitializer
  )
  // マーカーの追加・削除のaction関数
  // ここも効率化のためにuseCallbackを使っているが必須ではない。
  // const addMarker = (position) => dispatch({ type: "ADD", payload: position }) などでも十分だろう
  const addMarker = useCallback(
    (position) => dispatch({ type: "ADD", payload: position }),
    [dispatch]
  )
  const removeMarker = useCallback(
    (removeUuid) => dispatch({ type: "REMOVE", payload: removeUuid }),
    [dispatch]
  )

  // 外向けにobjectではなくarrayとして返すためのselector
  const getMarkers = useCallback(
    () => Object.entries(markers).map(([id, position]) => ({ id, position })),
    [markers]
  )
  return {
    // markers // 元のobjectとしてのmarkerは隠蔽する
    addMarker,
    removeMarker,
    getMarkers
  }
}
```

## 4-A: markersを配列として描画する

先程までのuseDrawMapMarkersを拡張する形でまずは書いてみる。
こちらの方が手軽といえば手軽だろう

```js
export const useDrawMapMarkers = ({
  markers,
  googleMap,
  map,
  onClickMarker
}) => {
  // マーカーの再描画を防ぐためrefsに保持
  const markerObjectsRef = useRef({})
  useEffect(() => {
    const { Marker } = googleMap.maps
    markers.map(({ id, position }) => {
      // すでに描画済みなmarkerだったら描画しない
      if (markerObjectsRef.current[id]) {
        return
      }
      const markerObj = new Marker({
        position,
        map,
        title: "marker!"
      })
      // markerがクリックされた時のイベントを追加する
      markerObj.addListener("click", (e) => {
        onClickMarker(id, markerObj, markerObjectsRef.current, e)
      })
      markerObjectsRef.current[id] = markerObj
    })
  }, [markers, googleMap, map])
}

```

利用側はこんな感じになる

```js
const useMapMarkerSetup = ({ googleMap, map }) => {
  // stateとして管理するマーカー
  const { addMarker, removeMarker, getMarkers } = useMarkerState(initialMarkers)
  const markers = getMarkers()
  // 描画する
  useDrawMapMarkers({
    markers,
    googleMap,
    map,
    // 削除イベントを追加
    onClickMarker: (id, markerObj, markerObjectsRef) => {
      removeMarker(id)
      markerObj.setMap(null)
      markerObjectsRef[id] = null
    }
  })
  // クリックイベントを追加
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })
}

const MapMarkers = ({ googleMap, map }) => {
  useMapMarkerSetup({ googleMap, map })
  return null
}
```

なかなか分厚い状態なのと、Markerのオブジェクトを利用側でいじる形になるのは少々気持ちが悪いかもしれない。

### コード

https://gist.github.com/terrierscript/861df9328f80f077ac0b534569f3e8e1

## 4-B: markersを一つずつのコンポーネントとして処理する

ということでこれを改修して、「マーカー1つ1つにコンポーネントとフックを対応させる」という方向でやってみる。

```js

export const useDrawMapMarker = ({
  position,
  googleMap,
  map,
  onClickMarker
}) => {
  const markerObjectsRef = useRef(null)
  useEffect(() => {
    const { Marker } = googleMap.maps
    // すでに描画済みなmarkerだったら描画しない
    if (markerObjectsRef.current) {
      return
    }
    const markerObj = new Marker({
      position,
      map,
      title: "marker!"
    })
    // markerがクリックされた時のイベントを追加する
    markerObj.addListener("click", (e) => {
      onClickMarker(e)
    })
    markerObjectsRef.current = markerObj
    
    // effectの返却の関数として、コンポーネントが消え場合の処理を書けるので、ここでmarkerもmapから消すように仕掛ける。
    return () => {
      if (markerObjectsRef.current === null) {
        return
      }
      markerObjectsRef.current.setMap(null)
    }
  }, [googleMap, map])
}
```

利用側はこのようになる

```jsx
// marker一つ一つを担当するComponent
const Marker = ({ googleMap, map, position, onClickMarker }) => {
  useDrawMapMarker({
    googleMap,
    map,
    position,
    onClickMarker
  })
  return null
}

const MapMarkers = ({ map, googleMap }) => {
  const { addMarker, removeMarker, getMarkers } = useMarkerState(initialMarkers)
  const markers = getMarkers()
  // クリックイベントを追加
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })
  return (
    <>
      {markers.map(({ id, position }) => (
        <Marker
          key={id} // hooksがkeyに紐づく。これがないと適切なマーカーが消えなくなる
          position={position}
          onClickMarker={() => {
            removeMarker(id)
          }}
          map={map}
          googleMap={googleMap}
        />
      ))}
    </>
  )
}
```

こうなってくると「なんだかhooks以前のReactと手間変わらない気もする・・・」というのもあるのだが、結局はこの方がスマートになる印象だ。


### コード

https://gist.github.com/terrierscript/f57ace64b776848d68be6b5bc736e2ed

### リファクタ: onClickイベントを分離する・

先程までの`useDrawMapMarker`だとイベントが変更しても感知しないものになっていた。
もう少しこの点きれいに考えると下記のようになるだろう。
`useDrawMapMarker`から帰ってくるMarkerオブジェクトを別途effectしてフックするような形になる。

また、これまでhooksの内部でしか使っていなかったため`useRef`で良かったが、Markerが外部に出すものとなったので`useState`で書き換えている。

```js
export const useDrawMapMarker = ({ position, googleMap, map }) => {
  const [markerObject, setMarkerObject] = useState(null)
  useEffect(() => {
    const { Marker } = googleMap.maps
    // すでに描画済みなmarkerだったら描画しない
    if (markerObject) {
      return
    }
    const markerObj = new Marker({
      position,
      map,
      title: "marker!"
    })
    setMarkerObject(markerObj)
    // コンポーネントが消えたらmarkerもmapから消すように仕掛ける。これはすっ
    return () => {
      if (markerObj === null) {
        return
      }
      markerObj.setMap(null)
    }
  }, [googleMap, map]) // markerObjectを更新対象にするとすぐ消えてしまうので、対象にしないようにする。この辺の勘所ちょっと慣れが必要そう

  return markerObject
}

export const useMarkerClickEvent = (marker, onClickMarker) => {
  // イベントが変更される事を考慮する
  useEffect(() => {
    if (!marker) {
      return
    }
    const listener = marker.addListener("click", (e) => {
      onClickMarker(e)
    })
    return () => {
      listener.remove()
    }
  }, [marker, onClickMarker])
}

```

```jsx
const Marker = ({ googleMap, map, position, onClickMarker }) => {
  const marker = useDrawMapMarker({
    googleMap,
    map,
    position
  })
  // イベントの呼び出しはこっちで行う
  useMarkerClickEvent(marker, onClickMarker)
  return null
}
```


## もう一つリファクタ: Context化する。

ここまでgoogleMapとmapをやたらと引き回してしまった。
そろそろこの辺をContext化する（正直もうちょっと手前でやっておけば良かった感じがある）

```js
export const MapContext = createContext({
  googleMap: null,
  map: null
})
```

途中で作成したWaitForMapのタイミングが`Provider`をするのにちょうどよいだろう

```jsx
const WaitForMap = ({ googleMap, map, children }) => {
  if (!googleMap || !map) {
    return null
  }
  const value = {
    googleMap,
    map
  }

  return <MapContext.Provider value={value}>{children}</MapContext.Provider>
}
```

例えば`useMapMarkerSetup`などは引数が不要になる。
利用側より親でProviderで囲む事を忘れないようにだけ注意が必要だ

```js
const useMapMarkerSetup = () => {
  const { googleMap, map } = useContext(MapContext)
  const { addMarker, removeMarker, getMarkers } = useMarkerState(initialMarkers)
  const markers = getMarkers()
  useMapClickEvent({
    onClickMap: ({ lat, lng }) => {
      addMarker({ lat, lng })
    },
    map,
    googleMap
  })
  return { markers, removeMarker }
}
```

# 5. InfoWindowを出す


最後に[InfoWindow](https://developers.google.com/maps/documentation/javascript/examples/infowindow-simple)を出すところまでやってみる。
内容物としてDOMNodeかHTMLの文字列を渡さなければならずなかなか苦戦するものだった。

## 準備：クリックで削除をダブルクリックで削除にする
クリックの処理をinfoWindowにさせたいので、削除処理はダブルクリックに移動させたい。

```js
// Markerのイベントフックを柔軟にしてどのイベントにも対応できるようにする
export const useMarkerEvent = ({ marker, eventHandler, eventName }) => {
  // イベントが変更される事を考慮する
  useEffect(() => {
    if (!marker) {
      return
    }
    const listener = marker.addListener(eventName, (e) => {
      eventHandler(e)
    })
    return () => {
      listener.remove()
    }
  }, [marker, eventName, eventHandler])
}

```

```jsx
const Marker = ({ position, onDoubleClickMarker }) => { // 先程context化をしたのでgoogleMap/mapについては気にしなくしている
  const marker = useDrawMapMarker({position})
  // イベントの呼び出しはこっちで行う
  useMarkerClickEvent({marker, eventName: "dblckick", eventHandler: onDoubleClickMarker)
  return null
}
```

## InfoWindowを作る

ほとんどこれまでの応用になるので、あとはhooksとコンポーネントを作っていくだけになる

```js
// googleMapのinfoWindowを生成して返す。
// contentNodeはDOM要素かstringなので、ここではDOMNodeを想定する
export const useMapInfoWindow = (content) => {
  const [infoWindowState, setInfoWindow] = useState(null)
  const { googleMap } = useContext(MapContext)

  useEffect(() => {
    if (!content) {
      return
    }
    // infoWindowの再描画防止
    if (infoWindowState) {
      return
    }
    const infoWindowObj = new googleMap.maps.InfoWindow({ content })

    setInfoWindow(infoWindowObj)
    return () => { // 消す時はcloseする
      infoWindowObj.close()
    }
  }, [googleMap, content])
  return infoWindowState
}
```


```jsx

// 表には見せない要素。vueのv-cloakから名前を拝借した
const Cloak = styled.div`
  display: none;
`

// infoWindowを仕掛けるコンポーネント
const MarkerInfoWindow = ({ marker, position }) => {
  const { map } = useContext(MapContext)
  const contentRef = useRef(null)
  // contentRefのDOMNodeを表示要素としたinfoWindowを作る
  const infoWindow = useMapInfoWindow(contentRef.current)

  useMarkerEvent({
    marker,
    eventName: "click",
    // クリックしたら開く
    eventHandler: () => infoWindow.open(map, marker)
  })
  return (
    <Cloak>
      <div ref={contentRef}>
        <b>hello</b>, {position.lat}, {position.lng}
      </div>
    </Cloak>
  )
}

const Marker = ({ position, onDoubleClick }) => {
  const marker = useDrawMapMarker({
    position
  })
  // ダブルクリックしたら消す
  useMarkerEvent({ marker, eventName: "dblclick", eventHandler: onDoubleClick })
  // markerは先程nullにしていたが、InfoWindowを表示する役割が出来た
  return <MarkerInfoWindow marker={marker} position={position} />
}
```

### DEMO

ここだけCodesandboxのDEMOで。hooksファイルが肥大化してしまったので分割したバージョンにしている
https://codesandbox.io/s/wnk87oykzw

# 感想など
* Hooksは単純なものをすごくライトに書けるようになるのはものすごく効果を感じた
    * ちょっとreducerとaction書くだけであればすごく楽だし、reduxやvuexの頃の知識はほぼそのまま使える
    * 地味に`selector`に相当する関数もhooksから返せるのはアリかもしれない
* やはりそこそこ複雑度のあるコードにはそれなりの複雑度になってくると感じた（それでもだいぶ楽ではある）
    * 曲線が変わった印象があり、複雑度が低い時が部分の難易度ぐぐっと下がって、複雑度が上がってくると徐々に変わらなくなってくる印象
    * hooksはhooksでハマるタイミングがある
    * （それでもやっぱりhooksのほうが楽なのは違いない）
* 誤解を恐れずにいうと「React先生がずーっと無限ループしててその途中でFunctional Componentが呼ばれていてその中でhooksの関数だけがその無限ループの外側で管理されてる」みたいな世界観を脳みそに宿しながら触っていると「あー、そういうことか」となる気がした（あっているかは微妙）
* 地味に4-Aで見せたような「少しReactらしくないコード」でも割となんとかなってしまうというのは結構利点に感じている。
* `useEffect`は今回のような外部のコード連携はとでも相性がよいし、返却の関数でclean upできるのも非常に良いと感じた。
    * ただ更新対象とする値をうっかり間違えると躓くことがしばしばあった
    * `useState`とかの値と組み合わせると無限レンダリングが走ったりするのをやってたりするので、そこらへんは注意点
* `useState`はimmutableな分、mutableに扱える部分で`useRef``に頼りたくなってしまうが、再レンダリングされず結構ハマるので注意したほうが良さそう
    * 特に外部に出す変数は`useRef`はやめたほうが良さそう。ハマる。
* 「hooksはトップレベルでしか使ってはいけません」というルールを迂回したくなるとヘンなコードを書いてしまいそうになるなーという感じがする。
    * 今回もhooks使うだけで何もしないコンポーネントというのをやったりしてしまったが「ちょっと微妙かも・・・」という気持ちは出てくる
    * このへんは今後ちょこちょこプラクティスが出てくる予感がするかもしれない
