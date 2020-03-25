Circle-CIでいつのまにcron実行する機能が追加されていたので試した。

* https://circleci.com/docs/2.0/configuration-reference/#triggers
* https://circleci.com/blog/manual-job-approval-and-scheduled-workflow-runs/

TravisCIには少し前から入っていた機能。

* https://docs.travis-ci.com/user/cron-jobs/

Circle-CIの場合、TravisCIほとさくっと使える感じではなかったのでメモ

# どんな時に使えるか？
例えばこんなユースケースが思いつく

* E2Eなど重くて全branchで実行するには辛いもの
* 外部のAPIに引きづられて落ちてないかのテスト
* masterを定期的にstagingと同期するdeploy処理

今回は一番上の「branchにCIは回したい、でも重いテストだけは一日一回にしたい」みたいな場合で考えてみる

# config

とりあえず最終的にどうなるかを記載。

```.circleci/config.yml
version: 2
jobs:
  test: # 普通のテスト
    docker:
      - image: node:8
    steps:
      - checkout
      - run: npm run test
  heavy_test: # 重いテスト
    docker:
      - image: node:8
    steps:
      - checkout
      - run: npm run heavy_test

workflows:
  version: 2
  normal_test_workflow:
    jobs:
      - test
  nightly_workflow:
    triggers:
      - schedule:
          cron: "0 1 * * *" # UTCで記述。この場合は朝10時
          filters:
            branches:
              only:
                - master
    jobs:
      - heavy_test
```

ポイントは下記の通り

* Circle-CIのconfigは2.0向けに書く必要がある
    * workflowを利用するので、workflow形式で。
* `trigger`の処理は、`job`ではなく、`workflow`に対して設定する必要がある。
    * 今回の場合だと、通常のbranch向けのjobとworkflowと、一日一回実行するjobとworkflowを記述する必要がある
    * 定期実行が通常のテストと同じで良いなら、jobを一つ、workflowだけ２つ書く感じになる。
    * それぞれでの共通化については、以前別記事を書いたので、必要があれば参考にしていただきたい
        * https://qiita.com/inuscript/items/3568690b5da29ab034dc
* Cronは普通にcrontab形式。ただし **UTC** なので注意。
* 記述を間違うと、失敗通知すら来ず無視されるのつらい
