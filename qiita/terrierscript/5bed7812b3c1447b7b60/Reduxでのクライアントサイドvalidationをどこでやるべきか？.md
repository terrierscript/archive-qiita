
入力フォームを利用するとやっぱり大事になってくるValidationについてあれこれ悩んだ。
結論が完全に自分の中でも出てないが、とりあえず考え尽くした所まで

# 前提など

## validation
validationとひとくちに言っても色々考える事がある

- 出力するエラーは一つ？複数？
- エラーが出たフォームを赤くしたいとかある？
- 複数の値を見てvalidationしたいとかある？
- validateするタイミングは随時？ボタン押されたら？

## 今回の話
- Redux + Reactを使う
- 簡易なTodoリストを想定する
- Actionの形式は[Flux Standad Action](https://github.com/acdlite/flux-standard-action)を使う
- 一旦細かいことは脇に置きつつ、下記のvalidationを想定して実装してみる
	- エラーメッセージは一つ
	- Todoのinputが空だったらエラーとする
	- Todoの追加ボタンが押されたタイミングでvalidateする
- [redux-form](http://erikras.github.io/redux-form)、[redux-validator](https://www.npmjs.com/package/redux-validator)など、外部ライブラリが存在しているが、しっくり来るのが無かったので今回は一旦考えない
	- 個人的にこれだ！というものが無かったため

## Todoリスト
シンプルにこんな感じ。
![TODO.png](https://qiita-image-store.s3.amazonaws.com/0/7307/c544c395-e593-1a8e-7988-924bdf88f17f.png "Electron_App.png")

ベースとなるコードはこちらに配置した[ソース](https://github.com/inuscript/example-validator-comparison/tree/without-validate)

### validationについて

空白が入力されたらエラーを出す、みたいなことを考える
validate自体は、一旦Redux、Reactから切り離してシンプルな関数があることを想定する

```validate.js
export const isValidTodo = (task) => {
  if(task === ""){
    return false
  }
  return true
}
```

また、エラーメッセージはreducerとして状態を持つことにする

```reducer/error-message.js
const errorMessage = (state = null, action) => {
  switch(action.type) {
  case 'ERROR_MESSAGE':
    return action.payload
  }
  return state
}

```



# 実装パターン
## 案1:Smart Componentで行う
propsを受けるSmart Componentでvalidateを行うやり方。
送信ボタンが行われたタイミングでvalidationをかませる。

```component.js
class InputComponent extends Component{
     :
  handleClick(e){
    let currentInput = this.state.currentInput
    // handle内でvalidate。
    // ダメなら値を入れない
    if(!isValidTodo(currentInput)){
      this.props.actions.errorMsg("Input some word")
      return
    }
    this.props.actions.appendTask(currentInput)
    this.setState({
      currentInput: ""
    })
  }
  render(){
    	:
      <TodoInput
        onSend={this.handleClick.bind(this)} 
        value={this.state.currentInput}
        />
        :
  }
}
class TodoInput extends Component{
  render(){
    const {onChange, onSend, value} = this.props
    return (
      <div>
        <input onChange={onChange} value={value} onKeyPress={this.handleKeydown.bind(this)}/>
        <button onClick={onSend} >Append</button>
      </div>
    )
  }
}

```

寸感：まあ、悪くはない。

### 利点
- 簡易なvalidationであればこれで十分事足りてる感。
- 後で「エラーが出たらそのフォームを赤くしたい」とかやりたくなったら割りとやりやすそう
- AとBの値を見てvalidateしたい。みたいな要望がある程度かなえられるかもしれない

### 欠点
- そもそもView層ってコードが肥大化しがちな所なのでそこに押し込めるのって微妙・・・
- エラーメッセージをReducerに流したいがために複雑度が上がっている感じがある

## 案2:Dumb Componentで行う
今回、前提としてreducerでエラーメッセージをやりとりすることを前提としたが、stateを利用することまで想定すれば、このぐらいまで出来そう

```js
  handleClick(e){
    let currentInput = this.state.currentInput

    // validationエラーをstateでやる
    if(!isValidTodo(currentInput)){
      this.setState({
        errorMessage: "Input some word"
      })
      return
    }
    this.props.actions.appendTask(currentInput)
    this.setState({
      currentInput: ""
    })
  }
```
寸感：これはナシかな・・・

### 利点
- 正直あんまりない・・・
- flux使ってないほど簡易で良いならこれでも良いかもしれない

### 欠点
- そもそもstate使うのあんまりやりたくない感じ
- AとBの値でvalidateしたい〜とかやりだすとstate肥大化の予感。

## 案3:Actionで行う
次にviewの一個奥のactionでやることを考える

```js
export const appendTask = (task) => {
  // エラーかどうかで返すactionを変える
  if(!isValidTodo(task)){
    return {type: "ERROR_MESSAGE", payload: "Input Some word"}
  }
  return {type: "TODO_ADD", payload: task}
}
```
寸感：良さそう

### 利点
- 一番まとまりが良さそう
- viewでやるよりも責務としては正しい気がする
- コンパクトに収まる
- サーバーサイドvalidationとかも透過的に扱えるかも


### 欠点
- task追加のactionを投げて別なactionが発火されるのは場合によっては驚きが大きいかも
- これもAとBの値を見つつCのvalidateしたいとかなったら辛い気がする。
	- `meta`を利用して他のstoreの値も受け取れるようにしておけばギリ可能かもしれない・・・
- （良し悪しだが）エラー出た表示を赤くしたい、とかなると`HOGEHOGE_VALIDATE_ERROR`みたいなaction typeが膨大になってしまうかも。

### 欄外:Actionでリアルタイムvalidation的な事やりたかったら？
上記のやり方、当然だが「片方のactionをキャンセルする」というやり方で、即時validationみたいなことをしたかった時にこまるのでちょっと考えてみた。

```component.js
class InputComponent extends Component{
    :
  handleClick(e){
       :
    this.props.actions.appendTask(currentInput)
    // validation用にもactionを飛ばす
    this.props.actions.checkTaskError(currentInput)
       :
```

```actions.js
// 値は値でreducerに返す
export const appendTask = (task) => {
  return {type: "TODO_ADD", payload: task}
}

// 別途、エラーがあれば、そちらもactionを生成する
export const checkTaskError = (task) => {
  if(!isValidTodo(task)){
    return {type: "ERROR_MESSAGE", payload: "Input Some word"}
  }
  return {type: "ERROR_MESSAGE", payload: ""}
}
```
考えとして複数の作用をさせるのだから複数のactionを発火させるというのは悪く無い気がするが、多分もうちょい色々考えたほうが良さそうな感じもしている。。。



## 案4:Reducerで行う
reducerでやるとしたパターンを考えると、
複数のreducerで同じaction.typeを処理する必要が出そうだ。

```reducer/tasks.js
const tasks = (state = [], action) => {
  switch (action.type) {
  case 'TODO_ADD':
    let task = action.payload
    // validate通った時だけpushする
    if(isValidTodo(task)){
      state.push(task)
    }
    return state.concat()
  case 'TODO_REMOVE':
     :
```
単に「データを入れないようにしたい」というだけであれば、taskのreducerで上記ようにpushを抑制すればいいが、エラーメッセージまで出したいなら下記まで必要だろう

```error-message.js
const errorMessage = (state = null, action) => {
  switch(action.type) {
	case "TODO_ADD":
    if(!isValidTodo(action.payload)){
      return "Invalid Task"
    }
  }
  return state
}
```
寸感：とても微妙・・・
あとReducerでやるにしてももっと別なやり方ありそうな気がしてくる・・・

### 利点
- 「値をどう扱うべきか」という事の責務をreducerが担うのはただしそう
- エラーメッセージのことを考えなければナシとはいえない。

### 欠点
- エラーメッセージのことを考えた途端結構微妙な感じになる。
- 1action typeに対して複数箇所での処理ってどうなのか？
	- 保守性悪そう・・・
- 多分そうでなくても、複雑なvalidationになったら途端に副作用起きるReducerが出来上がる危なさありそう

## 案5:Middlewareで行う
```middleware.js
const doValidate = (action) => {
  let errors = []
  switch(action.type){
    case "TODO_ADD":
      if(!isValidTodo(action.payload)){
        return "Invalid Task"
      }
  }
}

export const validateMiddleware = ({ dispatch, getStore }) => next => action => {
  let error = doValidate(action)
  if(error){
  	// errorがあったらerrorMsgのactionだけ発火させる
    dispatch(actions.errorMsg(error))
    return
  }
  let result = next(action)
  return result
}
```
寸感：アリかもしれない

### 利点
- 実装としては素直な感じ
- `getStore`で全ての値が取れるのである程度複雑なvalidationでも出来そう
- validateの処理が一箇所に収まる

### 欠点
- なんかreducerでやってることをやっている感じ・・・
- Middlewareもちょっと慣れるまで学習コスト高めかも？

## 案6:reselector(selector)で行う(2016/12/15追加)


詳しい実装に関しては有益な記事があったので追記

* [reselectを用いてReact Reduxにvalidationの仕組みを実装する 1/2](http://qiita.com/notsunohito/items/76d912c5e266670f2662)
* [reselectを用いてReact Reduxにvalidationの仕組みを実装する 2/2](http://qiita.com/notsunohito/items/a3a8324b40bbaea4bf9a)

> validationとはstateのある状態がvalidな状態なのかどうかを計算することに他なりません。

案3や案5の場合、冗長にstateが増えてしまう所を、上記のように「validateはあくまでstateのCompouted propsである」という発想のもと扱うことで、無駄にreducerやstateを増やすことを防止出来る。

発想としては案1のSmart Componentで行うのに近い。
Smart Componentよりもっと手前のselectorのレイヤーで行うようなイメージになるだろう。
また、案1はComponentが責務としている値でしかvalidateしづらかったのに対してstate全体の値でvalidationしやすくなることも良い部分だろうと思われる。

ただし、注意点として、この手法を取る場合、「reducerを通してstoreに入ったデータしかvalidate出来ない」という問題がある。validでないデータをstoreにも入れたくないというケースの場合、validateのロジックを切り出して、その値をComponentとselectorがどちらからも呼べるようにするなどの工夫が必要そうだ。


# まとめ
- 一定以上複雑なことになってきたら **Middleware** でのやり方に移植するとかは検討しても良いかもしれない
	- とはいえぶっちゃけそこまで複雑なパターンも無い（または回避出来る）気もしている。
	- 複雑なvalidationは、必要以上にクライアントサイドではやらないでサーバーサイドにまかせてしまうという割り切りも必要かも
- Smart Componentsでやるのは簡易ならよさそうだが、ごく１部にのみ使う場合かもしれない。とはいえ使いどころはありそう
- 案2 Dumb Component, 案4 Reducerはバッドパターンな気がする
