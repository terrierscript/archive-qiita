
# tl,dr;
`<MemoryRouter>`使えば良い。

# 例示

## シンプルなパターン
例えばこんなrouter設定で考える。

```js

import React, { Component } from 'react';
import { BrowserRouter, Switch, Route } from "react-router"

export const SomeRouter = ({path = ""}) => {
  return (
    <Switch>
      <Route path={`${path}/foo`}><div className="foo">foo</div></Route>
      <Route path={`${path}/baz`}><div className="baz">baz</div></Route>
      <Route path={`${path}/shallow`}><div><div className="shallowTarget">shallow</div></div></Route>
      <Route><div className="default">default</div></Route>
    </Switch>
  )
}

class App extends Component {
  render() {
    // RouterとRoute定義は分けておくとテストしやすい
    return (
      <BrowserRouter>
        <SomeRouter />
      </BrowserRouter>
    );
  }
}
```

そしてtestコード。
今回はenzyme + jestでのサンプル。

```js
  import { mount, render } from "enzyme"

  it('memory routing', () => {
    const wrapper = render(
      <MemoryRouter initialEntries={["/mount"]}><SomeRouter /></MemoryRouter>
    )
    expect(wrapper.find(".testTarget").length).toBe(1)
  })
```
`<MemoryRouter initialEntries={["対象にしたいパス"]}><SomeRouter /></MemoryRouter>`という感じでwrapするとテストが出来るようになる
MemoryRouterは、ReactNativeなど、ブラウザのnavigationに依存せず、配列でのみhistoryを擬似的に再現しているので、どんなテスト環境でも使える。
storybookなどで、`<Link>`を利用してコケたりする場合も`MemoryRouter`で解決する。

enzymeについては、`shallow`だとうまくfindのテストなどができなそうだったので、`mount`か`render`を利用する必要があるようだ（理由あんま深く調べてない）。

最近avaでも出来るようになったsnapshotテストならばこんな感じ。

```js
  import React from 'react';
  import { MemoryRouter } from "react-router-dom"
  import { create } from 'react-test-renderer'

  it('default routing', () => {
    const tree = create(
      <MemoryRouter initialEntries={["/"]} ><SomeRouter /></MemoryRouter>
    ).toJSON()
    expect(tree).toMatchSnapshot()  
  })
  
  it('/foo routing', () => {
    const tree = create(
      <MemoryRouter initialEntries={["/foo"]}><SomeRouter /></MemoryRouter>
    ).toJSON()
    expect(tree).toMatchSnapshot()  
  })

  it('/baz routing', () => {
    const tree = create(
      <MemoryRouter initialEntries={["/baz"]}><SomeRouter /></MemoryRouter>
    ).toJSON()
    expect(tree).toMatchSnapshot()  
  })
```

## ちょっとだけ入り組んだパターン

例えば`/user/:name`みたいなのも同じ要領でテスト可能

```js
const Hello = ({match}) => (
  <div>Hello {match.params.name}</div>
)
const NotFound = () => (
  <div>Oops</div>
)
export const DynamicRouter = ({path = ""}) => {
  return (
    <Switch>
      <Route path={`${path}/user/:name`} component={Hello}></Route>
      <Route component={NotFound}></Route>
    </Switch>
  )
}
```
```js
describe("dynamic snapshot", () => {
  it('/user/bob', () => {
    const wrapper = render(
      <MemoryRouter initialEntries={["/user/bob"]} ><DynamicRouter /></MemoryRouter>
    )
    expect(wrapper.text()).toBe("Hello bob")
  })
  it('/user/sam', () => {
    const wrapper = render(
      <MemoryRouter initialEntries={["/user/sam"]} ><DynamicRouter /></MemoryRouter>
    )
    expect(wrapper.text()).toBe("Hello sam")
  })
  it('/unknown', () => {
    const wrapper = render(
      <MemoryRouter initialEntries={["/unknown"]} ><DynamicRouter /></MemoryRouter>
    )
    expect(wrapper.text()).toBe("Oops")
  })
})
```

# 参考
* https://github.com/ReactTraining/react-router/issues/4078
* https://github.com/ReactTraining/react-router/issues/4227
