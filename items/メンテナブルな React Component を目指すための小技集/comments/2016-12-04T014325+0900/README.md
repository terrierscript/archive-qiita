早速の回答ありがとうございます。
貴重な意見ありがとうございます。というか、変なところに食いついてお手数おかけしてすみませんです。

> 後々設定をカスタマイズしたり、ejectする必要性を感じているのであれば自分であれば使いたいと思わないのが正直な所です。

ejectへの考え方について、なるほどと思いました。確かに依存性は完全には脱却できないのは厳しいですね。
おそらく、今後はこの辺の見極めが重要になってくると思うので、参考にさせて頂きます。:pray:


いま実践に向けて調査中なのですが、その知見でご回答にレスさせていただきます。

> 例えばESLintルール増やしたくてもできなそう

実はこれは出来ます。今ですが下記のようにreact-appとairbnbのスタイルガイドを適用しています
```.eslintrc
{
  "extends": ["reatc-app","airbnb"],
   "rules": {
        "no-console":0
    }
}
```

ejectに関して言うと、「今のところ必要ないのでは？」と考えております。
理由としては

- 思った以上にBABELに手を入れなくても行ける(propTypesをstaticで書いてもStage-0なくても行ける)
- React+Reduxに乗っていると、実はWebpackってそんなにカスタマイズ不要？
- create-react-app Wayに乗っかると、実はチーム内でコーディングの制約ができる(これはまだ検証中)

それと全体的なメリットして「初学者が多いプロジェクトだとシンプルな構成は本当に助かる」というのがあります。

設定ファイルを徹底的に減らすというのは本当に助かっており、大きな問題に当たらなければこのまま導入しようかと思っています。
React自体の思想として「ダメだったらコンポーネントを再利用して別に作ればいい」という大らかな思想があると思うので、ひどい話ですが見切り発車でも有りかなぁと。
この辺の実際にプロダクションに導入した知見が溜まったら、Qiitaで公開すると思うので、生暖かい目で見守っていただければと思います:joy:



