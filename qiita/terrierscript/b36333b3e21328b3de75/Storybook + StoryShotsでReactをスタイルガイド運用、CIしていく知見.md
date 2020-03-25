

2016/10/07追記：StorybookとStoryShotsについて説明してくれている記事が出てきたため、こちらの記事はもう少し深入りした部分にフォーカスするように（install関連のとこなどを主にばっさり削除）しました

* [React Storybook で変わるUI開発フロー (Redux Flavor)
](http://qiita.com/kitaly/items/85254fd346e2e575582b)

# [Storybook](https://getstorybook.io/)とは
Reactのためのスタイルガイドジェネレータ。

基本的な所は結構先人が記事を残してくれている。
特に今回この記事では`stories`そのものの記載については言及しないので、そこらへんは下記の記事などを参考にするのが良いだろう

* [react-storybookを用いたReactコンポーネント開発](http://developer.hatenastaff.com/entry/2016/04/14/150000)
* [React Storybook入門：コンポーネントカタログがさくさく作れちゃうかもしれないオシャレサンドボックス環境](http://qiita.com/beijaflor/items/4fc01f8d557c1926c38d)


だいぶ手に馴染んできてたので、自分がよく使う部分中心に、基本的によく使う設定 ＋ CIで運用する事あたりまでについて記載してきたい。

# Storybook

## よく使う or 基本的な設定、カスタマイズ
### config.js
`.storybook/config.js`に配置される。Storybookの基本設定ファイル。


#### 特定のパターンで動的読み込み
だいたい`stories`ファイルはまとめて一箇所に置くより、各コンポーネントの近い所に配置したい。

そうなると[動的読み込み](https://getstorybook.io/docs/basics/writing-stories#loading-stories-dynamically)が活躍する。

だいたいこんな具合

```.storybook/config.js
// `../src/`がターゲットディレクトリ。
// `hoge.stories.js`みたいなファイルを対象とする。
const req = require.context('../src/', true, /.stories.js$/)

function loadStories() {
  req.keys().forEach((filename) => req(filename))
}

configure(loadStories, module)
```

#### Decorator
Storybook上の表示の共通化のために、[Decorator](https://getstorybook.io/docs/basics/writing-stories#using-decorators)という仕組みが用意されている。

「全部のコンポーネント外枠つけたいな。」みたいな場合がよくあるので、こんな感じで使う。

```.storybook/config.js

const LayoutDecorator = (story) => (
  <MyLayout>
    {story()}
  </MyLayout>
)

function loadStories() {
  addDecorator(LayoutDecorator)
  req.keys().forEach((filename) => req(filename))
}

configure(loadStories, module);
```

特定のstorydだけで使うことも可能。

### [babelの設定](https://getstorybook.io/docs/configurations/custom-babel-config)
`.babelrc`があれば読み込んでくれる。
特にいじったことは今のところ無いが、個別に設定したい場合は`.storybook/.babelrc`に配置する。

### [Webpack設定](https://getstorybook.io/docs/configurations/custom-webpack-config)

Storybookはwebpackを裏で動かしている。
設定ファイルは`.storybook/webpack.config.js`として配置する。
webpackをプロジェクトのビルドをしてるなら特に考える事も無いかもしれない。

自分の環境では良くビルドには`browserify`や`rollup`を利用する事もあるが、`storybook`の部分はproductionビルドには関わって来ないので特に問題にならない。
（正確には、browserify側とStorybook側の設定値で同じような設定を二箇所でやらなければならないでちょっと冗長になるが、ギリ耐えられるという感じ。）

自分の場合、パス解決だけよく必要になったりするので、こんな感じで設定したりしたりする。

```js
const path = require('path')
module.exports ={
  resolve : {
    root: path.resolve(__dirname, '../src')
  }
}
```

### [ヘッダファイルはhead.htmlで加工](https://getstorybook.io/docs/configurations/add-custom-head-tags)

CSSを外部管理していたり、normalize.cssやsanitize.cssなどのreset系CSSを外から読みたかったり、フォントを読み込みたかったり する場合など、`<head>`タグに記載したい諸々は`.storybook/head.html`で管理する
参考：[Add Custom Head Tags](https://getstorybook.io/docs/configurations/add-custom-head-tags)

sanitize.cssをCDNから読むからこんな具合

```head.html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/10up-sanitize.css/4.1.0/sanitize.css">
```

# StoryShotsでStorybookにCIをもたらす

スタイルガイドはよく腐る。
しかし単なる「スタイルガイド」という視点ではなく「描画確認可能なのUnit Testだ」と考えたらどうだろう？少しは希望を感じる。

そんな希望に応えそうなのが`StoryShots`。

Storybookで記載したコンポーネントをsnapshotとして保存するテストを行うことが出来る。これはJestなどでも行われている手法。
（参考：[Snapshot Testing in React Storybook](https://voice.kadira.io/snapshot-testing-in-react-storybook-43b3b71cec4f)）

Jestでスナップショットテストをするのは良いと思うが、表示確認が出来るという部分で個人的には`StoryShots`に分があると思っている。

また、やってみた感想にはなるが、「Unit Testを書いているんだ」という気持ちで`stories`ファイルを書いてみると、コンポーネントを疎結合に書くことを意識できる良い効能もあった。


## ハマり回避テクニック
### loader, polyfillでハマり回避

ブラウザで動かすように作られたコードをCIするのはやっぱりまだまだ辛い。
StoryShotsではこの問題に対し、`loader`と`polyfills`というオプションが用意されている。

`loader`は`css`や`jpg`などwebpackのloaderを利用しているようなところをなんとかするものっぽい。自分はまだ世話になったことがない。

`polyfills`はjs側のコード的なところを調整するもの。
[src/default_config/polyfills.js](https://github.com/kadirahq/storyshots/blob/master/src/default_config/polyfills.js)からコピーして使う。

[clipboard.js](https://github.com/zenorocha/clipboard.js)を利用したコンポーネントで自分がハマったときは下記二行を足した。

```js
global.Element = global.window.Element;
global.HTMLElement = global.window.HTMLElement;
```

テストスクリプトもこんな感じで書き換える。　

```json
{
    :
    "test-storybook": "storyshots --polyfills=.storybook/polyfills.js"
    :
}
```

### テスト時だけmock化したい部分は`NODE_ENV=test`で回避

当たり前だが、乱数などを使ってる箇所はsnapshotテストではコケる。
こういう箇所に対しては、`NODE_ENV=test`で部分的なmockにすると良いだろう。
hack的ではあるが、まれによく見る慣例だ。

例えばuuidを使うような箇所はこんな感じにする。

```js
import uuid from 'uuid'

export default () => {
  if(process.env.NODE_ENV === 'test'){
    return 'unique-id'
  }
  return uuid()
}
```

テストの呼び出しにも`NODE_ENV`を設定する

```json
{
    :
    "test-storybook": "NODE_ENV=test storyshots"
    :
}
```

ちょっと冗長にはなるが、保守性は高くなるし費用対効果としては悪くないと思っている。

### テスト出来ないコンポーネントは諦める

身も蓋もない！という部分もあるが、とはいえCIにあんまり頑張りすぎて疲弊してもしょうがない。
無理を感じたら、一部のstoriesについてはいっそ諦めるのも一つの手だろう。

`-g`でgrepして一部絞込、`-x`で一部除外などがある。

```
npm run test-storybook -- -g "grep_target"
npm run test-storybook -- -x "exclue"
```

#### Decoratorを利用してskipする
`Decorator`と`NODE_ENV=test`を利用して、テスト時にスキップするデコレータを明示的に記載することも考えられる

```js
const TestSkipDecorator = (story) => {
  if(process.env.NODE_ENV === 'test'){
    return <div>Test Skip</div>
  }
  return story()
}

storiesOf('story', module)
  .addDecorator(TestSkipDecorator)
  .add('Module', () => (
    <SomeModule />
     :
```

### CIを回す

通常のテストとは分離して定義して、[npm-run-all](https://github.com/mysticatea/npm-run-all)などを使ってCI上での実行をする

```package.json
"scripts": {
    :
  "test": "mocha",
  "test-storybook": "storyshots",
  "test:all": "npm-run-all test test-storybook"
}
```

```circle.yml
test:
  - npm run test:all
```

## 番外：もっと色々テストしたいなら？

個人的には保守性のトレードオフを考えると、あまり複雑にテストするより、snapshotを保存するだけのStructural Testで十分に思えるが、下記の手法も紹介されている。

* [Interaction Testing](https://getstorybook.io/docs/testing/interaction-testing)
    * jestやmochaのテストもまとめれる。
    * [storybook-addon-specifications](https://github.com/mthuret/storybook-addon-specifications)を利用する
* [CSS/Style Testing](https://getstorybook.io/docs/testing/css-style-testing)
    * iframeで直接表示させる時の話
    * CIの話というより表示確認のため。
    * フルサイズ表示するような場合に使う。
    
