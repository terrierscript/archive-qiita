
[hyperなterm](https://hyper.is/)。

plugin作りたくて色々中身見たり、そもそもどうなってんのみたいな興味本位で読み解いたらreduxの使い方とかちょいちょい面白かったので感想とか雑感
コードはversion 0.6 ~ 7.1らへん。
https://github.com/zeit/hyperterm/tree/v0.7.1

# 全体感
- Coreな所
  - Electron 
    - おなじみ
  - hterm
    - chromiumのプロダクトの一つ（？）
    - コードは[このあたり](https://chromium.googlesource.com/apps/libapps/+/HEAD/hterm)
      - あんまりこっちは追ってない
- Electronの中で使われてる技術要素
  - React + Redux
  - CSS in JS
  - ビルドにwebpack

# プロジェクト構成など
- `/` (root)
  - webpackで管理している部分
  - libに関連するpackageはroot管理
- `/lib`
  - hypertermの本体部分。
  - React + Reduxなプロダクト
  - plugin作るときとかこのへん見るパターンが多くなる
- `/app`
  - `/lib`以外の色々
  - `electron`に関連する部分とか
	    - menuとかpluginとかindex.htmlとかnotify.htmlとか。
  - plugin関連だったりconfig関連だったり。
  - libとは独立的にpackage.jsonを管理している。
  - 起動時には`$ electron app`的な事で起動している
- `/assets`, `/build`
  - assetだったりelectron用のアイコンだったり

## Code

### hterm
- そもそもterminalどうやってんだよ？という所の回答が`hterm`というのを使ってるというやつ（前述通り）
  - https://chromium.googlesource.com/apps/libapps/+/HEAD/hterm

  - hypertermでは下記らへんでhtermが絡んでいる
	    - [components/term.js](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/components/term.js#L1)
	      - React的なComponentとhtermを繋いでる箇所
	      - htermへの設定追加なども行っているっぽい
    - [lib/term.js](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/hterm.js)
	      - htermそのままでは動かない部分をprototype継承でパッチしたりしている。


### reduxまわり
結構おもしろいと思った。

#### actionと`effect-middleware`
`redux-thunk`を使っているので、関数を返して非同期処理をしている。
ちょっと面白いと思ったのが`effect`というパラメータ

https://github.com/zeit/hyperterm/blob/477e40e4338b54a6d66c557ab8f9c3638f14d948/app/lib/actions/header.js#L4

```js
export function someAction (uid) {
  return (dispatch, getState) => {
    dispatch({
      type: SOME_TYPE,
      value,
      effect () {
        dispatch(otherAction(uid));
      }
    });
  };
}
```

これを[effect](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/utils/effects.js)というmiddlewareで、色々処理した後に実行している。

#### middleware

[index.js](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/index.js#L23)を見ると、`thunk` -> `plugin.middleware` -> `thunk` -> `effect`とやっている。
thunk二回やってるのは何か辛そうなものを感じる。
`plugin`は、名の通りpluginが仕掛けるmiddleware。

#### Reducerらへん
- [seamless-immutable](https://github.com/rtfeldman/seamless-immutable)でimmutable化してる
  - ArrayとかObject準拠なimmutable library?
- terminalのセッションの管理には[`uuid`](https://github.com/defunctzombie/node-uuid)が使われている
  - reducerでよく追加するものがある場合に`nextId`的なもので管理することとかあるけど、uuid使う方が良さそう

#### Container色々
- ReduxとReact繋いでるところ。
- dispatchを利用するhandler関数はContainerレイヤーで作って、viewにはdispatchを意識させないようにしているらしい
- reslectは[containers/header.js](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/containers/header.js)でのみつかわれている。
  - 若干のノリで入れてみたんじゃないだろうか感も感じてしまう
  - reselectorとcontainerを同じファイルに書くのは結構アリかも
- 各Container
  - Hyperterm
	    - いわゆるメイン
	    - キーバインドとかしてある
  - HeaderContainer
	    - ヘッダ部分（タブとか）
  - NotificationContainer
	    - Notification関連
	    - Electronのnotificationではなく、hyperterm内部のnotification。
  - TermsContainer
	    - Terminal関連Container

### React関連
#### Component
- hypertermで利用されているComponentは、ReactのComponentを直で使うのではなく、一つ独自にComponentを作ってwrapしている
  - https://github.com/zeit/hyperterm/blob/v0.7.1/lib/component.js
- PureRenderMixinを利用している
- ほぼCSS in JSの為という印象
- コンポーネントは`render`ではなく、`template`メソッドに、Virtual DOMを記載する
- `render`は、CSSと`template`のvdomを組み合わせる役割をするようになっている
- pluginを作る時は、この辺りの勘所注意だろう

#### CSS in JS
- jsonなconfigで色設定などをできるようにしている都合もあるのだろう、CSS in JSを行っている。
- 利用しているのは[aphrodite](https://github.com/Khan/aphrodite)というライブラリ
  - 実際にはaphrodite-simpleというforkしたっぽいライブラリを使ってるようだが、コードが見当たらないので正確な違いはわからに


#### Component細かく
ほぼContainerと対をなしている感じ。

- header, tab, tabs
  - terminal上方のTab周り
  - `header > tabs > tab` みたいな構成
- notification, notifications
  - あんまりちゃんと見てない
- term, terms
  - terminal表示部分。
  - [term.js](https://github.com/zeit/hyperterm/blob/v0.7.1/lib/components/term.js#L1)が`hterm`と直接連携している（前述通り）
  - termsの`this.terms`
	    - 子termの管理として、`this.term`を管理してる 
	    - 中身は`hterm`のインスタンスでpureなobjectではない
	    - stateやreduxのstateではなく、Componentのpropertyでの管理しているのは、多分pureなObjectじゃないから扱いづらいとかそういう感じに見える

### plugin
- [app/plugins.js](https://github.com/zeit/hyperterm/blob/v0.7.1/app/plugins.js#L19)に、hook出来そうなplugin列挙されている
- ドキュメントが整備されている感じではないので、何出来るかは気持ちを察していくしかない

#### decorate
- pluginがhypertermに行う加工に関して、`decorator`とか`decorate`とか名前付けられている
- Componentなどを覗いたりしても、ちょこちょここの`Decorate`という話が出てくる。
  - Componentのdecorateは、いわゆるHocsっぽい形になっている。
- plugin側で色々拡張させるための工夫っぽい
  - うまく作ってないと途端にやばくなりそうではあるワイルドさを感じる
- objectのdecorateは単に`Object.assign`で拡張される感じ

### rpc

- htermとelectronとhypertermの命令疎通をしている。
- `electron <-> rpc <-> redux(action)`という感じの双方向な通信
- EventEmitter
- このへん
  - https://github.com/zeit/hyperterm/blob/v0.7.1/lib/utils/rpc.js
	    - 内部側
  - https://github.com/zeit/hyperterm/blob/v0.7.1/app/rpc.js
	    - electron側


### config
- [gaze](https://github.com/shama/gaze)はfs.watchのwrapper。
  - 更新監視してるっぽい
- configファイルの評価は`request`だとか`JSON.stringify`ではなく[`vm.Script`](https://nodejs.org/api/vm.html)とからへんを使っている模様

### [app/session](https://github.com/zeit/hyperterm/blob/v0.7.1/app/session.js)
- ptyとかttyとかあんまり詳しくないので利害が微妙。
- 結局shellのタイトルを表示するために実行中のプロセスのコマンドを取得しているように見える
- titleを定期的に取得してemitしている。
- 使っているのはこのあたりのパッケージ
  - [child_pty](https://github.com/Gottox/child_pty)
  - [default-shell](https://github.com/sindresorhus/default-shell)
