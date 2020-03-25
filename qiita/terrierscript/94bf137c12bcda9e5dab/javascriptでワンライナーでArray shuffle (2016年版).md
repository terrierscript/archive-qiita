
小ネタ

たまに「じゃんけんする代わりぐらいなノリでワンライナーでシャッフルしたい！
rubyの`shuffle`ぐらいなやつでやりたい！でも手元にはchromeぐらいしかない！」

という事があるが、ぐぐってみると思ったほど自分にヒットするやり方が載ってなかったのでメモる。

---
※ ワンライナーでやるために`sort`を利用しているが、[Array#sort実装のshuffleは偏る](http://qiita.com/minodisk/items/94b6287468d0e165f6d9) らしいので、
今回の方法はくじ引きの代わりぐらいに留めて、ちゃんとしたい場合はlodashなどのライブラリを利用するほうが良い。

### やり方

```js
["apple", "banana", "orange"].sort( () => Math.random() - 0.5)

// 一人だけ選びたいならこんな感じ
["apple", "banana", "orange"].sort( () => Math.random() - 0.5)[0]

// 複数人選びたいならこんな感じ
["apple", "banana", "orange"].sort( () => Math.random() - 0.5).splice(0,2)
```

`sort`時に`Math.random()`の返す乱数を元に `true / false`をテキトーに返す。
`Math.random()`はseedを設定したり出来ないので、再現性があるようなコードではないので、production利用用途とかならもうちょっと別な手法のほうが良いだろう。


手元にIEしかない！という時はarrow functionやめればいける。

```js
["apple", "banana", "orange"].sort( function(){ return Math.random() - 0.5 })
```

数列をshuffleしたいならこんな感じ。魔術感出てきた。

```js
[...Array(3).keys()].sort( () => Math.random() - 0.5)
```
(参考: http://rochefort.hatenablog.com/entry/2015/01/08/005043)


