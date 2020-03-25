Reactで Function as Child Component、Child Function, Function as Children と呼ばれるものがあって最近馴染んできたのでここらでまとめてみる。

* [function-as-children](https://reactpatterns.com/#function-as-children)
* [Reactのfunction as childとは何か](https://qiita.com/khatada/items/12756040f79dc6bbdfe9)

以下、この記事ではFunction as Childrenという呼称を利用する

見慣れたところで言うと[ReactのContext](https://reactjs![error]()
.org/docs/context.html)でも使われている手法で、`props.children`に渡される値をfunctionにする事でHoCsに似たような効果を得る事ができる手法になる。

例えばHoCsだとこんな具合なものを想定する

```jsx
const withStaticValue = (WrappedComponent) => {
  class HOC extends React.Component {
    render() {
      return (
        <WrappedComponent
          value={1234}
        />
      );
    }
  }    
  return HOC;
};

const MyItem = ({value}) => {
  return <div>{value}</div>
}

const WrappedMyItem = withStaticValue(MyItem)
```

するとこれが下記のように書ける。
HoCsでの複雑さが減る嬉しさがある。

```jsx

const WithStaticValue = ({ children }) => {
  return children({ value: 1234 })
}

const MyItem = ({ value }) => {
  return <div>{value}</div>
}

const WrappedMyItem = () => {
  return (
    <WithStaticValue>
      {({ value }) => (
        <MyItem value={value} />
      )}
    </WithStaticValue>
  )
};
```
効能としてはHoCsに近いので、利用箇所もHoCsと同じようなものだと思ってもらいたい。

Childrenを関数にするという若干「ほんとにそんなのしていいの？という手法だが、下記のようなライブラリでも使われているのが確認できた

* [react-spring](https://github.com/drcmda/react-spring)
* [react-final-form](https://github.com/final-form/react-final-form)
* [react-router](https://reacttraining.com/react-router/web/api/Route/children-func)
* [react-powerplug](https://github.com/renatorib/react-powerplug)
* [react-downshift](https://github.com/paypal/downshift)

# 例：BMIの計算ツールでStateの保持を分離していく

これを応用して、大きなコンポーネントからstateを分離していく例を考える。
規模感としては「redux入れるまでもないけど」ぐらいな場合だと思ってほしい。

## 初期段階：普通に書いた場合

まずは何も考えずに記述した場合を考える

```jsx
class App extends Component {
  state = {
    weight: 0,
    height: 0
  };
  computeBMI(weight, height) {
    const meterHeight = height / 100;
    return weight / (meterHeight * meterHeight);
  }
  handleWeight = e => {
    this.setState({
      weight: e.target.value
    });
  };
  handleHeight = e => {
    this.setState({
      height: e.target.value
    });
  };
  render() {
    const { weight, height } = this.state;
    const bmi = this.computeBMI(weight, height);
    return (
      <div className="App">
        <h1>BMI tool</h1>
        <div>
          <h2>Input</h2>
          <div>
            Height:
            <input value={height} type="number" onChange={this.handleHeight} />
            cm
          </div>
          <div>
            Weight:
            <input value={weight} type="number" onChange={this.handleWeight} />
            kg
          </div>
        </div>
        <div>
          <h2>Result</h2>
          <div>BMI: {bmi}</div>
        </div>
      </div>
    );
  }
}
```

1コンポーネントの行数としてはそこそこで、例えばここに「標準体重の計算も入れたい！」とか、「数値によって結果を出し分けたい」とかを考えると徐々にしんどくなってくるのが想像できる。

画面的にはこんな感じ。
<img width="288" alt="91316a4b.png" src="https://qiita-image-store.s3.amazonaws.com/0/7307/f3baebd3-7fe4-3e44-61c0-c9be31c73a9a.png">


## 第一段階：Formと結果表示を別コンポーネントにする

ここから分離をはじめていく。
本題であるFunction as Childrenを利用する前に、それ以外のところをわかりやすさのために切り出していく

```jsx

const InputForm = ({ weight, height, onChangeWeight, onChangeHeight }) => {
  return (
    <div>
      <div>
        Height:
        <input value={height} type="number" onChange={onChangeHeight} />
        cm
      </div>,
      <div>
        Weight:
        <input value={weight} type="number" onChange={onChangeWeight} />
        kg
      </div>
    </div>
  );
};

class BMIResult extends Component {
  compute() {
    const { weight, height } = this.props;
    const meterHeight = height / 100;
    return weight / (meterHeight * meterHeight);
  }
  render() {
    const result = this.compute();
    return <div>BMI: {result}</div>;
  }
}

class App extends Component {
  state = {
    weight: 0,
    height: 0
  };
  handleWeight = e => {
    this.setState({
      weight: e.target.value
    });
  };
  handleHeight = e => {
    this.setState({
      height: e.target.value
    });
  };
  render() {
    return (
      <div className="App">
        <h1>BMI tool</h1>
        <div>
          <h2>Input</h2>
          <InputForm
            {...this.state}
            onChangeHeight={this.handleHeight}
            onChangeWeight={this.handleWeight}
          />
        </div>
        <div>
          <h2>Result</h2>
          <BMIResult {...this.state} />
        </div>
      </div>
    );
  }
}
```

`<InputForm>`とそれを利用する`<BMIResult>`として切り離された。この時点でもコンポーネントはかなりすっきりしたが、まだ`<App>`が大きい。

## 第二段階: Stateだけ管理する部分を切り出す

いよいよ本題。今回はReduxの名前を借りて`Container`という名前をもらう。
体格のState部位のみを保持するContainerを作りたい

```jsx

//InputFormとBMIResultは上記と同じなので省略する

class BodyStateContainer extends Component {
  state = {
    weight: 0,
    height: 0
  };
  handleWeight = e => {
    this.setState({
      weight: e.target.value
    });
  };
  handleHeight = e => {
    this.setState({
      height: e.target.value
    });
  };
  render() {
    return this.props.children({
      ...this.state,
      handleHeight: this.handleHeight,
      handleWeight: this.handleWeight
    });
  }
}
```

`<BodyStateContainer>`を使う側はこうなる。

```jsx
class App extends Component {
  render() {
    return (
      <BodyStateContainer>
        {({ handleWeight, handleHeight, ...rest }) => (
          <div className="App">
            <h1>Health tool</h1>
            <div>
              <h2>Input</h2>
              <InputForm
                {...rest}
                onChangeHeight={handleHeight}
                onChangeWeight={handleWeight}
              />
            </div>
            <div>
              <h2>BMI Result</h2>
              <BMIResult {...rest} />
            </div>
          </div>
        )}
      </BodyStateContainer>
    );
  }
}

```

`return this.props.children({...})` という表記が出てきた。これがFunction as Childrenになる。
`BodyStateContainer`のみが状態を持ち、`App`はもう状態を管理していない。
しかし利用側では`<BodyStateContainer>{(props) => ...}`として相変わらずstateの値を受け取ったり、handlerを介して変更することが可能になっている。

また、もしほかの場所で同じように体重と慎重を扱いたい場合は`<BodyStateContainer>`を使い回すことも可能だろう。

仮にこのまま使いまわした場合、それぞれの`BodyStateContainer`がstateを持つが、もしそれをグローバルに扱いたければ`React.Context`の出番になってくるだろう（あんまり無いケースだとは思う）

## 第三段階:型をつける

通常の場合、childrenは必ずReactの要素であったが、このFunction as Childrenなどを使うとchildrenが何が何だか分からないという状況が起きる。
このへんもあるので、[Function as Childrenはアンチパターンだ！](https://americanexpress.io/faccs-are-an-antipattern/) という話もある。

ただ個人的にはTypeScriptで型があればある程度は秩序が保たれるとも思うので、型の付け方を記載しておきたい

まずBodyStateContainerを考える。childrenを関数にして、`ReactNode`を返すのがキモになる

```tsx

type BodyState = {
  weight: number;
  height: number;
};
type Handlers = {
  handleHeight: React.ChangeEventHandler;
  handleWeight: React.ChangeEventHandler;
  // もうちょっと雑にするなら (e: any) => void;
  
};
type BodyChildrenProps = Handlers & BodyState;
type BodyStateProps = {
  children: (props: BodyChildrenProps) => React.ReactNode;
  // もうちょっと雑にするなら  Function とか (props: any) => ReactNode とか
};


class BodyStateContainer extends Component<BodyStateProps, BodyState> {
   : 
  render() {
    return this.props.children({
     :
     :
    })
  }
}
```

そして子については、spread operatorを使うことでChildrenPropsがそのまま使える（ただし不要なパラメータも渡すので少々雑な型付けではある）

```tsx

class App extends Component<{},{}> {
  render() {
    return (
      <BodyStateContainer>
        {params => (
          <div className="App">
            <h1>Health tool</h1>
            <div>
              <h2>Input</h2>
              <InputForm {...params} />
            </div>
          </div>
        )}
      </BodyStateContainer>
    );
  }
}

// spread operatorで受け取るのでChildrenPropsをそのまま流用
const InputForm: React.SFC<BodyChildrenProps> = ({
  weight,
  height,
  handleWeight,
  handleHeight
}) => {
  return (
     ...
  );
};
```

型については、HoCsの場合OuterPropsとInnerPropsがあってややこしい感じがあったが、Function as Childrenの場合はほとんど迷わないしなんならきれいに書ける感じは良い。

# 余談
以下、いろいろ書いてて思いついた使い方などを余談的に記述していく
あくまで実験的なことも書いているので予めご了承いただきたい。

## その1:さらに他のFunction as Childrenを追加する

実用的なパターンは少ないが、複数のFunction as Childrenを組み合わせるパターンを示したい。

例えばここで「子供の場合はBMIじゃなくて別な指標を作りたい」みたいな場合を考える。
分離した`Container`のstateを増やしていくのものいいが、それも続けていくと肥大化してしまいそうだ。
例えば今回は`BodyStateContainer`はそのまま体重身長という部分にフォーカスさせたまま、別途年齢を管理するContainerを作ってみることを考える。

そして今回はついでに、「年齢は外に露見したくなくて、若いかどうかだけを返したい」みたいなreadonlyなケースで考えてみる。
するとこんな感じになるだろう

```jsx

class AgeStateContainer extends Component {
  state = {
    age: 20
  };
  isYoung() {
    return this.state.age < 16;
  }
  handleChange = e => {
    this.setState({ age: e.target.value });
  };
  render() {
    return (
      <div>
        Age <input value={this.state.age} onChange={this.handleChange} />
        {/*子に返すのはisYoungだけ*/}
        {this.props.children({ isYoung: this.isYoung() })}
      </div>
    );
  }
}
```

説明のためにFunction as Children側に`<input>`を配置しているが、表示要素をもたせるのとデザイン的に辛くなる可能性があるので注意したい

利用側はこんな具合。


```jsx

// ネストしまくるのを回避したいので、Containerのとこだけ切り出して子に渡す
const MixedContainer = ({ children }) => {
  return (
    <AgeStateContainer>
      {ageContainerProps => (
        <BodyStateContainer>
          {bodyStateContainerProps => {
            return children({
              ...ageContainerProps,
              ...bodyStateContainerProps
            });
          }}
        </BodyStateContainer>
      )}
    </AgeStateContainer>
  );
};

class App extends Component {
  render() {
    return (
      <div className="App">
        <h1>Health tool</h1>
        <MixedContainer>
          {({ isYoung, handleWeight, handleHeight, height, weigth  }) => (
            <div>
              {/* 今までと同じなので入力らへんは省略 */}
              <div>
                {isYoung ? (
                  <ForYoungHealthResult height={height}, weight={weigth}/>
                ) : (
                  <ForAdultHealthResult height={height}, weight={weigth}/>                  
                )}
              </div>
            </div>
          )}
        </MixedContainer>
      </div>
    );
  }
}
```

`isYoung`の結果フラグを使いつつ、`height`と`weight`をさらにその子に渡すことで、それぞれのContainerからの値を使えている



また、型をつけるならこんな感じになるはずだ。

```tsx
type AgeContainerChildrenProps = {
  isYoung: boolean;
};
type AgeContainerProps = {
  children: (props: AgeContainerChildProps) => React.ReactNode;
};

type MixedChildrenProps = BodyChildrenProps & AgeContainerChildrenProps;
type MixedContaienrProps = {
  children: (props: MixedChildrenProps) => React.ReactNode;
};
```

MixedChildrenPropsなどは利用するContainerのChildrenPropsのIntersection Typesになってちょっと気持ちが良い

## その2: Componentをただの関数化してやる

ここまで来ると、Componentをただの関数にできそうな事に気づく。
例えばBMIの計算周りは、こんな感じにできるはずだ。

```jsx

const ComputeBMI = ({ weight, height, children }) => {
  const meterHeight = height / 100;
  const result = weight / (meterHeight * meterHeight);
  return children({ result });
};
```

利用側はこんな具合になる

```jsx
const ForAdultHealthResult = rest => {
  return (
    <div>
      <h2>BMI Result</h2>
      <ComputeBMI　{...rest}>
        {({ result }) => {
          <div>Your BMI: {result}</div>;
        }}
      </ComputeBMI>
    </div>
  );
};
```

もはやこれはコンポーネントと呼ぶべきなのか関数と呼ぶべきなのかわからないが、「しょうがないからClass化してメソッド生やすしか無かった」みたいな部分に対するソリューションとしては使える場合がありそうだ。

## まとめ
ということで、Function as Childrenについてまとめてきた。
肥大化したComponentを分離する場合や、Redux使わずうまくしのぎたい場合にはいろいろ使い道があるだろう。

ただ、おそらくこの手法は型が無いと相当厳しくなっていくとも思われる。

当然、HoCs同様乱用すると可読性に影響がありうるものなので、適宜使える場合を見つけて導入すると良いだろう
