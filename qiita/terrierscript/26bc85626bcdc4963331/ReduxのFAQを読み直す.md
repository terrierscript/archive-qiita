ReduxのFAQを読み直したら「アレ？結構初期と変わってる！」とか「あ、やっぱそうだったんだ」みたいな事などがあったので自分の中で知識差分アップデートしたメモ。

順序は気になった項目順。
QとAはそれぞれFAQからのざっくりな意訳。

# [React Redux](http://redux.js.org/faq/react-redux)

## Q.  React-Reduxで、Topコンポーネントだけをconnectすべきですか？

[Should I only connect my top component, or can I connect multiple components in my tree?](http://redux.js.org/faq/react-redux#should-i-only-connect-my-top-component-or-can-i-connect-multiple-components-in-my-tree)

### A. 初期の頃はそう言ってましたが、最近はそうでもないです。適切にconnectしましょう

* 初期のreduxドキュメントにおいて、「トップに近い最小限のコンポーネントだけをconnectすべきと推奨していた」
* しかし、昨今、一般的な要求としてはこれだと、コンテナそれぞれの必要とするデータが多くなりすぎることがわかってきた
* 現在のベストプラクティスとしては下記のような感じ
    * `Presentational`(Connectしない)と`Container`(Connectする)のコンポーネントは明確に分類する
    * 利便性に応じてContainerを作成・接続する
    * 同一種類のデータを受け取る子要素のために親Componentの記述が重複しているなら、Containerを外出しすると良い
    * 親が子要素の個別具体なデータやactionについて多く知りすぎていると感じた場合は、そこがコンテナを抽出するタイミングといえる


### その他の所感・補足
* Containerが肥大化するのを感じたり、reselectorを利用した部分がやたらと複雑になってしまうパターンは、個人的によく出くわしたので、「あ、カジュアル目に切ってもいいんだな」とわかった。
* 肌感だが、Containerが扱うPropertyが3個ぐらい上限が適切な範囲で、5個とかを超えてくると、分離するタイミングだと思っている。
* ただし、当然Containerは再利用性が低いので、再利用性の高いPresentationalなコンポーネントを意識して作るべきというところは意識すべきと思われる
    * Containerから書いてしまうと全部Containerになっていくので、Presentaionalなコンポーネントを先に書いて、あとでContainerを書く方がうまくいくイメージ

# [Code Structure](http://redux.js.org/faq/code-structure)

## Q. どんなファイル構成がありますか？どのようにaction creatorとreducerをグルーピングすべきですか？どこにselectorsを置くべきですか？

[What should my file structure look like? How should I group my action creators and reducers in my project? Where should my selectors go?](http://redux.js.org/faq/code-structure#what-should-my-file-structure-look-like-how-should-i-group-my-action-creators-and-reducers-in-my-project-where-should-my-selectors-go)

### A. Reduxはデータライブラリなので、特に決めはありません

プロジェクトとしての直接的な意見はないが、下記の傾向がある

* Rails-style
    * actions | constants | reducers | containers | components みたいな分け方
* Domain-style
    * ドメイン・機能単位でディレクトリを分割する。
    * ファイルタイプに応じてサブディレクトリを作ることもできる 
* "Ducks"
    * Domain-styleと近い。ただし、`action`と`reducer`を **同じ** ファイルに記載するやり方。
    * 参考：https://medium.com/@scbarrus/the-ducks-file-structure-for-redux-d63c41b7035c#.lkekioku5

### その他の所感・補足
* 序盤はRails-styleが前提だった気がしていたが、公式として「そこはそんなに決めはない」ということらしい
    * Rails-styleだといまいちコード量がばらついてイマイチだなーという部分もあったので、色々検討して良いというのは嬉しいかもしれない。 
    * まだ微妙にどれもしっくり来てないのでもっと色々知りたい


## Q. ビジネスロジックをどこに書くべきですか？reducerとactionでどう分割すべきですか？

[How should I split my logic between reducers and action creators? Where should my “business logic” go?](http://redux.js.org/faq/code-structure#how-should-i-split-my-logic-between-reducers-and-action-creators-where-should-my-business-logic-go)

### A. 明確な回答はありません。

* ユーザーによって様々
    * ファットなAction creatorと、来たデータをmergeするだけの薄いシンプルなreducerもいる
    * 逆に、Actionを極力小さくし、`getState`の利用を少なくするユーザーもいる
    * この質問においては、sagaやobservableなど非同期アプローチは、Action Creator側として扱う
* これら２つのバランスが取れた時、あなたはReduxをマスターしているでしょう

### その他の所感・補足
* 「決めてない」っていうの多くない！？と思いつつ、そこらへんはまだ伸び代のあるところな気がする
* 色々Middlewareの話が出てくるとぐっとややこしい話になるのも事実
* 個人的な肌感だと、reducerは太らせない方が良さそうかなという感覚がある
* Reducerにロジックをまとめる系の話しは、下記にも少し記載されている
    * [Structuring Reducers](http://redux.js.org/docs/recipes/StructuringReducers.html)
    * [Reusing Reducer Logic](http://redux.js.org/docs/recipes/reducers/ReusingReducerLogic.html)


# [Reducer](http://redux.js.org/faq/Reducers.html)

## Q. 2つのreducerのデータを共有したいのはどうすればいいですか？

[How do I share state between two reducers? Do I have to use combineReducers?](http://redux.js.org/faq/Reducers.html#how-do-i-share-state-between-two-reducers-do-i-have-to-use-combinereducers)

### A. `combineReducer`では許可してませんが、考えられる手立てがいくつかあります

* 基本として、reducerはstoreを薄切り(slice)分割してる場合と、ドメインレベルで分割している場合がある
    * 薄切り(slice)の分割というのはドメイン的には同一だが、一部のデータだけ取り出して扱っている場合を言っていると思われる
* 多くのユーザーが2つのreducer間でのデータ共有をしようとするが、`combineReducer`はこれを許可していない。これを行うには幾つかのアプローチがある
    * A1. Reducerのツリー構造を見直す：reducerが別のsliceのデータを知る必要がある場合は、単一のreducerが管理する対象を増やすように、tree構造を再構成する
    * A2. Actionの幾つかを処理するcustom関数を作り、`combineReducer`の代わりにそれを利用する
        * [reduce-reducer](https://github.com/acdlite/reduce-reducers)などを利用して、`combineReducers`を使いつつ、特殊なreducerを実行させるということも考えられる
    * A3. `redux-thunk`など非同期Actionを扱う場合、`getState`でstate全体を取得できる。ここからactionに対してデータを入れる

### その他の所感・補足
* `reducer`の構成を見直して解決出来るパターンというのが一番解決方法としてはすっきりしそう
* `reduce-reducer`や、`redux-thunk`などのmiddlewareに頼るのは、割りと奥の手っぽく見えた。
* 自分が同類の悩みにぶつかった場合も、`reducer`の見直しが長期的には一番うまくいく感じがした
* view絡みでこの辺の問題にぶち当たる場合は、`reselector`などの`Computed Data`として扱えば解決することも多かった
    * 参考: [Computing Deriverd Data](http://redux.js.org/docs/recipes/ComputingDerivedData.html)


# [Organizing State](http://redux.js.org/faq/organizing-state)

## Q. 全てのstateをreduxに入れるべきですか？`setState`を使うべきですか？

[Do I have to put all my state into Redux? Should I ever use React's setState()?](http://redux.js.org/faq/organizing-state#do-i-have-to-put-all-my-state-into-redux-should-i-ever-use-reacts-setstate)

### A. local componentの状態を使用すること自体は問題ありません

* "正しい" 答えはない。全データをreduxに入れたがるユーザーもいるし、ドロップダウンの開け閉めなどUIに関するcriticalでないデータをコンポーネントのstateにするのを好むユーザーも居る
* 一般的な経験則としては、下記に基づいてReduxに入れるかを考えると良い
    * アプリケーションの他の部分がそのdataを扱うことがあるか？
    * そのデータから派生して、別なdataを作成する必要があるか？
    * 複数のcomponentを動かすのに同一のdataが利用されているか？
    * その値を特定の状態に戻すことに価値があるか？（Time Travel Debugなどをしたいか？０
    * データをcacheしたいか？（データを再要求せず、現在のデータを扱いたいか？）
* componentレベルのstateをstoreする実装系もいくつかある
    * [redux-ui](https://github.com/tonyhb/redux-ui), [redux-react-local](https://github.com/threepointone/redux-react-local), [redux-component](https://github.com/tomchentw-deprecated/redux-component)など

### その他の所感・補足
* 何でもかんでもstoreにぶっこんでいるのは結構面倒なので、UI関連をlocal stateにする場面は結構多い
* 上記判断基準は、どこまでがlocal stateとしてOK？というのを考えるのに有益そう


# [Store Setup](http://redux.js.org/faq/store-setup)

## Q. 複数のStoreを作成して使う事べきですか？できますか？componentが直接storeを使うことができますか？

[Can or should I create multiple stores? Can I import my store directly, and use it in components myself?](http://redux.js.org/faq/store-setup#can-or-should-i-create-multiple-stores-can-i-import-my-store-directly-and-use-it-in-components-myself)

### A. 基本的にreduxで複数のstoreは不要です。一部、複数storeを利用する場合が正当になることも一部あります

* 原典となるfluxは、複数のstoreを扱い、Domainのデータを保持していた。
    * `waitFor`で他のstoreを待つ必要があるなど問題が出がちだった
    * reduxは、これが不要になっている
    * なので複数store
* 片一方、複数のstoreが単一のページ中に出て来ることはありえる（ただし、あくまでstoreはそれぞれ単一で保持される場合に限る）
    * パターン1. 一部のupdateが頻繁なことが計測され、分割によってパフォーマンスの問題が解決する場合
    * パターン2. 巨大なReduxアプリケーションを独立させ、root ComponentのインスタンスごとにStoreを作成する場合
* storeのsingletonなinstanceを直接importして利用するのも推奨されない
    * Reduxアプリケーションを、更に巨大なアプリケーションの一部から独立的に切り離すのが難しくなる
    * サーバー上でreques毎にstore instanceを作ってしまうため、サーバーサイドレンダリングも難しくなる

### その他の所感・補足
* 巨大なアプリケーションの分割・インスタンスごとにstoreを持つ例に関しては、[IsolatingSubapps](http://redux.js.org/docs/recipes/IsolatingSubapps.html)が役立つ

# [Actions](http://redux.js.org/faq/actions)

* actionについて

## Q. なぜAction typeはstringやserializeできる値である必要がありますか？

[Why should type be a string, or at least serializable? Why should my action types be constants?](http://redux.js.org/faq/actions#why-should-type-be-a-string-or-at-least-serializable-why-should-my-action-types-be-constants)

### A. シリアライズ可能だと、デバッグなどで役立ちます

* Actionがserializableだと、state同様タイムトラベルデバッグが有効になったりする
* symbolを利用したりすると、これが破壊される
* middlewareでの利用を意図して、Symbolやpromiseをactionに利用することは可能
    * ただし、Actionがstoreとreducerに到達するまでには、serializableである必要がある
* パフォーマンス上の理由からreduxがチェックしているのは、actionが素のobjectであること、`type`が定義されていること、という部分のみである
    *なので、シリアライズじゃない値を渡すこと自体は可能だが、シリアライズ可能なほうがデバッグに役立つ

### その他の所感・補足
* Storeの値についても、同様の回答がされいる
    * [http://redux.js.org/faq/organizing-state#can-i-put-functions-promises-or-other-non-serializable-items-in-my-store-state](http://redux.js.org/faq/organizing-state#can-i-put-functions-promises-or-other-non-serializable-items-in-my-store-state)

## Q. ActionとReducerは1:1でマッピングすべきですか？

[Is there always a one-to-one mapping between reducers and actions?](http://redux.js.org/faq/actions#is-there-always-a-one-to-one-mapping-between-reducers-and-actions)

### A. そうでもないです。1つのactionが複数のreducerを変更しても良いです

* そんなことはない
    * reducer関数は、特定のstateのupdateに責務を持つ小さく独立した関数にすることを推奨する
    * これを、`reducer composition`パターンと呼ぶ
* Actionは、stateの全部、または一部を操作できる。
    * これによりコンポーネントと実際のデータの変更を切り離されたままになる。
    * 1つのActionがstate treeの様々な部分に影響を与える可能性がある
    * Componentはこれらを知る必要が無い
* ducksの構成など、密接に結びつけるユーザーもいる。
    * デフォルトでは、actionとreducerの1:1マッピングは全く無い
    * 1つのactionで、複数のreducerをハンドリングしたい場合は、そのパラダイムから脱却する時

### その他の所感・補足
* 例えば`CLEAR`みたいなactionが来たら全部のデータを消す、みたいなパターンで、1:1対応しない事はあるので、このへんは柔軟に活用するのがよさそう
    * `CLEAR_FOO_DATA`, `CLEAR_BAZ_DATA`みたいに分ける必要は無いよね、ということだと思われる
* 片一方、あんまりカジュアルにやりすぎると、偶然Action名が重複して事故るとかもありえそう。

