すでにどこかにありそうですが
functional updateを使わないという条件付きですが、これをやると直感的に書けて便利でした。

```js
function useFunction(initialFn) {
  const [state, setState] = useState({ fn: initialFn });
  const setFn = fn => setState({ fn });
  return [state.fn, setFn];
}
```
