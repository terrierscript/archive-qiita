個人的にgeneratorは知見が少ないので、あまり参考にならないかもですが、やはりそのあたりの変換自体は[`babel-plugin-transform-runtime`]
(http://babeljs.io/docs/plugins/transform-runtime/)を入れて`{ "plugins": ["transform-runtime"] }`を追加して解決するというのが本筋な気がしております。

上記ドキュメント見ると、`You should also install babel-runtime itself`ともあるので、この２つがペアになるのは「まあそんなものなのかなあ」という感覚でした。

また、考えられる手として`peerDependency`として`babel-runtime`を依存させるような手もあるのかなとも思いましたが、これはこれでたまに辛いことを引き起こしそうな気がすることを考えると、やはりおとなしく`npm i -S babel-runtime`してしまうのが一番いいんじゃないかなと思っています。

ちなみに、babel-runtimeに依存すること自体もそこまで辛いと感じていなかったのですが、どのあたりが辛いと感じるポイントなのでしょう？
