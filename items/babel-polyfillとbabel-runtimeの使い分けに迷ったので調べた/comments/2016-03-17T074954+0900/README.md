ありがとうございます。

> ネイティブで使える函数についてはそちらが優先されて使われます。

こちら、書き方が悪かったです。（訂正しました）

どちらもpolyfillが関数そのものはネイティブなのですが、core-js内の関数をコールするオーバヘッドが、プロダクトによっては無視できなくなる程度のパフォーマンスの影響をあたえる可能性を考えておりました
（そのレベルでパフォーマンスを考えるプロダクトにおいてbabelを使うことの是非がそもそもありそうですが・・・）

またglobalの汚染については抜け落ちた視点でしたので、こちらについても追記させていただきました。

どちらが良いかというお話で言うと、`babel-plugin-transform-runtime`も、一部うまく変換できないのか[disableにされている](https://github.com/babel/babel/blob/master/packages/babel-plugin-transform-runtime/src/definitions.js#L29)場合があるようですし、なかなか一概には決めれないかもしれません
