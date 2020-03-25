
さくっとreact 0.14にあげてみたらこれがコケた

```js
import ExecutionEnvironment from 'react/lib/ExecutionEnvironment'
```

そもそもテストで行っているハックっぽいことなので、ドキュメントにも載ってなさげだった。
色々探して`fbjs`に含まれるようになったことがわかった

ということでこんなんで解決

```
$ npm i -D fbjs
```

```js
import ExecutionEnvironment from 'fbjs/lib/ExecutionEnvironment'
```
