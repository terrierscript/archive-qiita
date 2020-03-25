**2015/12/03 追記:** [MartyのReadmeより](https://github.com/martyjs/marty#marty-is-no-longer-actively-maintained-use-alt-or-redux-instead-more-info)
Martyはもうメンテをしないので、reduxかaltを使う事が推奨されるようです。ありがとうMarty。

以下過去の記事内容

---

[Flux Comparison](https://github.com/voronianski/flux-comparison)で色々眺めていたり、twitterを追っかけていたり自分で触ってみたりして、
[Marty.js](http://martyjs.org/)が個人的に本当に良い気がしている。

# Martyの特徴
#### 作者と質問者の会話より
ユーザーとのこんなやりとりがあり、Martyの特徴が要約されているのかなと思ったのでテキトーな意訳として載せておく。

> https://news.ycombinator.com/item?id=8923245

>質問：fluxxorと何が違うのか？
回答：
他のFlux実装は、Fluxっぽいデータフェッチ(APIなどのことだと思う)なヘルパーが無い。コンポーネントとのストアのバインディングのボイラープレートの方向に向かっている。あとデバッグ機能。他のはあんまりデバッグ機能拡充してない
Martyは下記のことを助ける

>
- コールバック無しの非同期データフェッチ
- viewのためのState Mixin
- Chrome Extensionのデバッガー

ということでこの辺りが特徴らしい

- コールバック無しの非同期データフェッチ
- viewのためのState Mixin
- Chrome Extensionのデバッガー

## Facebook Fluxとの対比
公式の図とMartyにおけるAPIの対応がどうなっているかをまとめる

だいぶ雑な図だけど僕の中でこんなイメージ。
![marty.png](https://qiita-image-store.s3.amazonaws.com/0/7307/86f92133-cc69-76c7-4031-ad50c11ecce4.png "marty.png")


### 対応表
Flux | Marty 
----				|---
Web API Utils		| State Source
Action Creators	|Action Creators
Store             | Store
Change Events <br>Store Queries  |State Mixin
Dispatcher       | Constants(*)
Callbacks        | Constants(*)

(*) DispatcherとConstantsなどは、隠蔽している関係などで、直接1:1ではなくなっている、このへんは自分なりの理解。


## 有益なサンプル
そもそもまだそんなに見つけられなかったのだけど、見るならこのへん

- https://github.com/jhollingworth/marty-todo-mvc
- https://github.com/voronianski/flux-comparison/tree/master/marty

# 各機能のざっくり説明など
だいたいコードとかは公式サンプル持ってきたりしています

## [State Source](http://martyjs.org/api/state-sources/index.html)
WebAPIUtilに相当する、データ取得の実装を巻き取る層。
Martyの目玉機能の一つだし僕が「これいいなあ」となった部分の一つ。

```UserAPI.js
var UserAPI = Marty.createStateSource({
  type: 'http',
  baseUrl: 'http://foo.com',
  getUsers: function () {
    // github/fetch 形式での取得（後述）
    return this.get('/users').then(function (res) {
      // 完了したら、ActionCreatorを呼ぶ
      SourceActionCreators.receiveUsers(res.body);
    });
  },
  createUser: function (user) {
    return this.post('/users', { body: user });
  }
});
```
こんな感じで`this.get`などで通信処理ができる。
HTTPの通信処理は[isomophic-fetch](https://github.com/matthew-andrews/isomorphic-fetch)が使われているようなので何も考えずにpromiseな感じで書いていける。

またHTTPだけでなくLocal StorageやSession Storageへのアクセスヘルパーも用意してくれている模様（未調査）

## [Constants](http://martyjs.org/api/constants/index.html)
このあたりからは他のflux実装とわりと似てくる。

```Constants.js
var Constants = Marty.createConstants([
  'RECEIVE_USERS',
  'DELETE_USER'
]);
```

- ほかのflux実装にもよくあるconstantとほぼ一緒。
- Actionとして発火される定数を定義
- オブジェクトとしてconstantを階層化することもできる
- 各Constantsが、`Action`を発火することのできる`Action Creator`の関数となっている（という認識）
- `{Action Type}_STARTING`、`{Action Type}_DONE`、`{Action Type}_FAILED`など、特定の接尾辞のついた定義をすることによってPromiseがらみの処理ができるらしい（未調査）
	- [公式ドキュメント](http://martyjs.org/guides/action-creators/index.html)


## [Action Creators](http://martyjs.org/api/action-creators/index.html)
これもわりと他にもよくある実装。

- ちょっとこれは理解するの時間がかかった。
- 言ってしまえばConstantsから生まれる`Action Creator`の集合体。
- `action.rollback()`などで失敗時にロールバックもできるらしい。（未調査）
- 体感としてこのあたりどうしても若干冗長に感じる

```ActionCreatro.js
var UserConstants = Marty.createConstants(["UPDATE_EMAIL"]);

var UserActionCreators = Marty.createActionCreators({
  updateEmail: UserConstants.UPDATE_EMAIL(function (userId, email) {
    this.dispatch(userId, email) //例
  })
});

UserActionCreators.updateEmail(123, "foo@bar.com");
```
- また注意点として、この部分の処理は`()=>`なarrow functionを利用するとthisのスコープが変更されてしまうので、避けるようにとのアナウンスがある。
	- 個人的には余り気にならないが、がっかりする人いそうだ
	- [関連pull req](https://github.com/jhollingworth/marty/pull/84)

## [Stores](http://martyjs.org/api/stores/index.html)
- だいたい他のfluxと同等のStore
- `handlers`の定義でConstantの`Action Creator`を指定している。
- `handlers`の定義は、複数のConstantsを受けたりとか色々複雑な指定ができる模様（未調査）
	- [公式ドキュメント](http://martyjs.org/api/stores/#handlers)

```Store.js
var UsersStore = Marty.createStore({
  displayName: 'Users',
  handlers: {
    addUser: Constants.RECEIVE_USER
  },
  getInitialState: function () {
    return {};
  },
  addUser: function (user) {
    this.state[user.id] = user;
    this.hasChanged();
  }
});
```
## [State Mixin](http://martyjs.org/api/state-mixin/)
- State <=> Component(ReactComponent)をつなぐ部分
- 結構公式だと長いサンプルがあるが、単純にStoreとComponentをつなぐなら簡易に書ける

```UserComponent.js
var UserState = Marty.createStateMixin(UserStore);

var UserComponent = React.createClass({
  mixins : [UserState],
	:
	:
})
```
- 上記は結構sugger syntaxっぽい気もするが、複数のStoreをListenしたりとか細かめに設定することも可能（未調査）

```UserState.js
var UserState = Marty.createStateMixin({
  listenTo: [UserStore, FooStore]
});
```

## Debug機能
なんかまだうまく使えてないので未調査。



# 結論
Martyええぞ。
