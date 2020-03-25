
TypeScriptをしばらく触って、「あれ、もっと型推論に頼っていいじゃん！」と今更判ってきたのでまとめる。

今回は[`reselect`](https://github.com/reactjs/reselect)を例に挙げる

### reselectについて

`reselect`において`createSelector`の型定義は下記のような具合で列挙されている

```ts
// 本当はもうちょっと複雑だが、説明用に簡略化している
export function createSelector<S, R1, R2, T>(
  selector1: (state: S) => R1,
  selector2: (state: S) => R2,
  combiner: (res: R1, res: R2) => T,
): (state: S) => T
```

つまり、`selector1`, `selector2`でそれぞれ計算された結果を、`combiner`でまとめて、`<T>`型を返す、という具合に定義されている。

### 型パラメータを明示する（型推論しない）場合

`reslect`を利用するときに型パラメータを指定する場合、下記のような具合になる

```ts
import { createSelector } from "reselect"

interface Item{
  price: number
}
interface State {
  items: Item[]
}

// それぞれ
// <元のstateの型, 最初の関数の返り値, 最後の返り値>
const itemSumPrice = createSelector<State, number[], number>( 
  (state) => state.items.map( item => item.price),
  (prices) => {
    return prices.reduce( (a, b) => a + b, 0)
  }
)

```

これはかなり保守がしんどい。ちょっと拡張しようとして、例えば「消費税を計算して、単位もつける」となるとこんな具合になる。

```ts
interface State {
  items: Item[]
  tax: number
  unit: string
}

// <Stateの型、最初の関数の返り値,二番目の関数の返り値.... 最終的な返り値>
const itemSumPrice = createSelector<State, number[], string, number,string>( 
  (state) => state.items.map( item => item.price),
  (state) => state.unit,
  (state) => state.tax,
  (prices, unit, tax) => {
    return `${prices.reduce( (a, b) => a + b, 0)  * tax} ${unit}`
  }
)
```

型が多くなってだいぶ読み辛い。脳内スタックが追いつかない

### 型推論に頼る

ということで、型推論に頼る。`createSelector`の型パラメータをあえて省略する。

```ts
const itemSumPrice = createSelector( 
  (state : State) => state.items.map( item => item.price),
  // pricesはnumber[]型として推論されてくれる
  (prices) : number => {
    return prices.reduce( (a, b) => a + b, 0)
  }
)
```

`State`を明示することで、その後の`prices`の型が推論される。

`State`の型をつけるのがしんどい場合は、あまり褒められ無いやり方だが、anyにしてしまって返り値を指定することである程度推論してくれる

```ts
const itemSumPrice = createSelector( 
  (state: any) : number[] => state.items.map( item => item.price),
  // pricesがnumber[]として推論されてくれる
  (prices) : number => {
    return prices.reduce( (a, b) => a + b, 0)
  }
)

```

この方法を使えば、複雑化しても見通しがそこそこ良くなる

```ts
const itemSumPrice = createSelector( 
  (state: State): number[] => state.items.map( item => item.price),
  (state: State): string => state.unit,
  (state: State): number => state.tax,
  // prices, unit, taxがそれぞれ推論される。
  (prices, unit, tax): string => {
    return `${prices.reduce( (a, b) => a + b, 0)  * tax} ${unit}`
  }
)
```

また、下記のような矛盾するselectorを書いてもちゃんと怒ってくれる

```ts
const invalidCreator= createSelector( 
  (state: number): number => state,
  (state: string): string => state,
  (item1, item2): string => {
    return `${item1} ${item2}`
  }
)
// =>
// The type argument for type parameter 'S' cannot be // inferred from the usage. Consider specifying the type arguments explicitly.
```

`react-redux`の`connect`でも似たような感じで推論させたほうが良かったりするので、どんどん推論に頼っていきたい
