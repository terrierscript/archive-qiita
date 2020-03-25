
# 最初思ってたこと vs 現実
- 思ってたこと：なんか軽量？
	- 現実：誤解だった。割りと作ってくと大きめになってくる。軽量感は無い
	- が、redux本体のコードは結構薄い。本当に関数群だけ提供しているイメージ
- 思ってたこと：他のfluxと圧倒的に違う？
	- 現実：これも誤解。構成としては色々作りこむと最終的には一般的なfluxと同じ感じになる（flummoxとかmartyあたりと体感一緒になる）
	- ActionとかActionCreatorとかその辺が一緒になってくる。
- 思ってたこと：ES6で圧倒的に書ける？
	- 現実：正解。圧倒的にES6を前提にしている。
- 思ってたこと：React用のもの？
	- 現実：別にそうでもない。`react-redux`を使うことでreactで使えるようになる。


# 詰まった所など
## 用語編
割りと初めて見たりする用語に出会った

- ActionCreator
	- martyあたりにもあった。`{type: "HOGE", name: "fuga"}`みたいな`action`を返すためのwrapper関数とでも考えるとよさそう
- Reducers
	- 前はStoresと呼ばれていた。
	- `(previousState, action) => newState`をするやつ
		- といわれるだけだと謎だが、実際そうだった。
	- Store関連の話と併せて後述するので参照されたい。
- containers
	- これは用語としていいのかわからない。俗語に近い気がする
	- [examples](https://github.com/rackt/redux/tree/master/examples)に結構出てくる
	- reactの起点ポイントをcontainerとしているっぽく見える

## Storeの色々編

### StoreとReducer
普通のfluxのことを思い浮かべると、Storeはこんな感じでリソースに対して記述するよなものをイメージしていた。

```js
class UsersStore extends Marty.Store {
  constructor(options) {
    super(options);
    this.state = [];
    this.handlers = {
      addUser: Constants.RECEIVE_USER
    };
  }
  addUser(user) {
    this.state.push(user);
    this.hasChanged();
  }
}
```

が、reduxだとこうならない。
Storeは結構隠蔽されている感じで、利用者は`reducer`というのだけを記述する（このへんからreducerの話）

reducerは例えばこんな感じ。

```js
function someReducer(state, action) {
  switch (action.type) {
  case SOME_ACTION:
    return Object.assign({}, state, {
      some: action.someValue
    });
  default:
    return state;
  }
}
```

初見で「どういうこと？」ってなったが、stateが他のFluxの例でのstoreの内部管理されていたような保持される変数だと思えば良い。これが`(previousState, action) => newState`している部分になる。
Reducerは __actionに応じてstateを変更する関数__　だと思えば良さそう。

### ハマりどころ：reducerは`state`を変更しないと変更が通知されない
先ほどのサンプルで、わざわざ`Object.assign({},state, {someProps})` のような小めんどくさいことをしている。

これは理由がある。
例えばこんなreducerの場合、viewが変更されない。

```動かないパターン.js
function someReducer(state, action) {
  switch (action.type) {
  case SOME_ACTION:
  	state.someValue = action.someValue
    return state;
  default:
    return state;
  }
}
```

stateを新規に作らないと更新されない。
正直どういう仕組でこうなっているのかちゃんと追えてない。追いたい。

## 細かい関数色々出ていて焦る編
色々出てきてビビる。が、一個一個は別に大したことをしていない。逆に言うとReduxはここに挙げる関数が全てだった（コード読んでみると結構小さい）

ざっくりと何やってるかを早見表的に。
詳細は[APIリファレンス](http://rackt.github.io/redux/)を参照のこと。

### redux編
- createStore
	- container層で使う。
	- `reducer`の関数群から`store`を生成する。
	- `store`に直接お目にかかれるのはこのあたり。
- bindActionCreators
	- `export function`で定義した関数群の`actionCreator`と`dispatcher`をbindしてくれる。わりと名前通り
	- Componentの一部で、`store`から取り出した`dispatch`とactionCreatorをつなぐ利用イメージ
- applyMiddleware
	- いわゆるplugin機構。
	- redux自体が薄めに作られているので、各関数に手をいれるような事はmiddlewareで行う
	- 例えばasyncなactionを扱う機能なども本体ではなく、middlewareとして切りだされている。
	- 結構この仕組みのお陰でredux本体がコンパクトになっているように見える。
- combineReducers
	- 複数のreducerをまとめて一つのreducerにしてくれる
	- 全く関連のないような`reducer`が一個のファイルにあるみたいな状態だと肥大化する一方なので、そういう時に分割しとけば便利

### react-redux編
結構「なにそれー」ってなった。
くわしくは[こちら](http://rackt.github.io/redux/docs/basics/UsageWithReact.html)

- connect
	- reduxから渡されるstateと、React.Componentのpropsのマッピングをする部分。
	- マッピングの定義は第一引数で関数として渡す。

```connectの例.js
function mapStateToProps(state) {
  return { todos: state.todos };
}

export default connect(mapStateToProps)(TodoApp);
```

- `<Provider>`
	- あえてタグ形式で書いたのは、`React.Component`継承なもののため。
	- 使うときには下記のような何か魔術的な記載をする（ここも深く追ってない）
	- 下記の例の`<App/>`部分は、`connect`でstoreと連結されている必要がある（でないとうまく動かぬ）

```Provierの例.js
<Provider store={store}>
  {() => <App/>}
</Provider>
```


# 参考
- http://rackt.github.io/redux/
- https://gist.github.com/yosuke-furukawab8acfbb1b2bfb0b3c23c
- action補助らしい
	- https://github.com/insin/redux-action-utils
	- https://github.com/acdlite/redux-actions
