
# 概要
CircleCI上でフロントエンドビルドしてpushまでする、みたいなのをworkflow使うといい感じにできるよという話。1.0の頃は全部実行されていたのが、manual approvalによってほしいときだけに実行する素敵機能になってくれた。

（ビルドの生成物gitにpushするのどうよ？という話もありそうだけど、今回そこは置いておく）

今回は、CircleCIのこのへんの機能を利用する

* [workflow](https://circleci.com/docs/2.0/workflows)
* [manual approval](https://circleci.com/docs/2.0/workflows/#holding-a-workflow-for-a-manual-approval)
* [environment](https://circleci.com/docs/2.0/env-vars/)
* [when attribute](https://circleci.com/docs/2.0/configuration-reference/#the-when-attribute)

# config.yml

今回はlaravel-mixなprojecteでの例示をする。
多分フロントエンドじゃないプロジェクトでも使えると思う

```.config.yml

version: 2
jobs:
  unittest: # 通常のテスト処理など
    docker: php:5.3
    steps:
      - checkout
      - run: some-test-command 
  frontend-push:
    docker: node:8
    steps:
      - checkout
      - run:
          name: git configの設定 ( botのメアドとnameを設定 )
          command: |
            git config user.email "0000000+some-bot@users.noreply.github.com"
            git config user.name "some-bot"
      - run:
          name: merge ( originのマージ。やらなくてもいいかも）
          command: git merge origin/master -m "AUTO MERGE ${CIRCLE_BUILD_URL}"
      - run:
          name: Build実行
          command: |
            yarn install
            yarn lint
            yarn production --no-progress
      - run:
          name: 必要箇所をcommit ( 全部addしてしまうと色々おかしなものが紛れ込む"
          command: |
            # プロジェクトごとにいい感じにadd。
            git add yarn.lock
            git add resources/assets
            git add public/build
            git add public/mix-manifest.json
            git commit -m "AUTO BUILD ${CIRCLE_BUILD_URL}"
      - run:
          name: 失敗時のデバッグ
          command: |
            git status
          when: on_fail
      - run:
          name: push ( $GITHUB_ACCESS_TOKEN は自分で設定しておく )
          command: |
            git remote add upstream https://${GITHUB_ACCESS_TOKEN}@github.com/${CIRCLE_PROJECT_USERNAME}/${CIRCLE_PROJECT_REPONAME}
            git push upstream ${CIRCLE_BRANCH}:${CIRCLE_BRANCH}
workflows:
  version: 2
  my_workflow:
    jobs:
      - unittest
      - HOLD-FRONT-BUILD: # ボタン押したら実行。
          type: approval
          filters:
            branches:
              ignore: master # masterにpushさせるのは微妙なのでやらない
      - frontend-push:
          requires:
            - HOLD-FRONT-BUILD
```

# ポイントなど
* `type: approval`を使う
    * 今までのCircleCIのこれ系のビルドだと、毎回実行されるので、うっとおしかった
        * Holdボタンがあるので、必要なときだけ使える
    * 1.0の頃は毎回実行されるので、無限ビルドされないように工夫が必要だった。
        * 実行は手動なので、特にやらなくてもいい
        * 必要があれば、コミットメッセージに`[skip ci]`を含めると良い
    * 1.0の頃は「変に失敗されると困る」みたいな感じで色々実行するしないの判定が必要だったが、やってない
	    * 「手動でやるのでコケたらコケたでまあいいや」ぐらいなスタンスで使える
* ビルドしてる部分はビルド&addすることでconflictはわりと無理やり解決してるので、このやり方が合うケースとそうじゃないケースはあるかも
* X-OAUTH-TOKENを利用して、pull/push周りを簡素化している
    * sshでもいいのだけど、結構面倒なので最近はX-OAUTH-TOKEN使ってpull/pushするの楽だなと感じている
    * GITHUB_ACCESS_TOKENはenvironmentで自前設定する必要あり
        * githubにtokenを含んだcommitをpushすると、自動でrevokeされるので注意
* 全部addせず、必要なところだけaddしている
* `on_fail`を使って失敗時はstatus出すようにしとくとデバッグが便利になる
* 見つけやすいようにHOLDのジョブ名を大文字にしている
* 今は同じbranchにpushしているけど、このあとでAPIまで駆使すれば自動でPull req作るとこまでできるかも
	* masterのときは別branch作ってpushしてpull req作る、とかまでやったらカッチョ良さそうだけど面倒だったので諦めた

# 微妙なところ
* CircleCIのボタンの反応がイマイチ
	* なんか押されてるのか押されてないのかわからないときがある
* CircleCIのアクセスが面倒
    * github上から押せるボタンとか出せるAPIとかあればいいのに
* 色々整合性あわなかったときはコケる
    * 良し悪し。個人的にはこれはこれでいいかなと思っているけど、「絶対に成功させたい」みたいな場合は微妙そう
* config.emailとconfig.nameの設定が必要
    * 地味にめんどくさい。
