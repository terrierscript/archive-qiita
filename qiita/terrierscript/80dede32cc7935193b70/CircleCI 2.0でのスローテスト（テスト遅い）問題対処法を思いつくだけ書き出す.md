# まえおき

スローテストの解消に関して、昨今のCIサービスを考慮した観点で自分なりの手法をまとめてみる。
CIで出来そうなことは可能な限り網羅したつもりだが、他にもあったらコメントか編集リクエストでご指摘いただきたい。

## とりあつかうこと・とりあつかわないこと

* CircleCI 2.0 を前提とする。
    * 1.0はもうすぐ無くなるので対象外
    * 他のCIサービスは今回対象としてないが、一部似たような機能があるかもしれない。
* テストフレームワーク固有の話はなるべく排除している
    * サンプルコードがnodeだったりrubyだったりで統一取れてないのはご了承いただきたい。
* `docker`モードを前提とする。`machine: true`での実行は検証していない
    * 検証してないだけなので、もしかしたら動くものもあるかもしれない
* CIではなくCD（継続デリバリー）に特化した話は除外する
    * 転用できる部分はあるものの、あまりフォーカスしない

## 大まかな分類
大まかな方針の分類は旧来なスローテスト問題の対策と変わらないこの４つになる。
この記事中でもこの四分類を軸に話を進めていく

1. テスト実行環境を強くする
2. テスト実行を並列化する
3. テスト実行する対象を絞り込む
4. テスト実行速度を上げる

# 1. テスト実行環境を強くする
開発者が5人以上いて、資金に余裕があるならばお金で解決するのがなんだかんだでまず早い。
Jenkinsなど自前サーバーでは増やすのも手間だが、CIはここに関してはすぐ出来る。

## 1-1. インスタンス数を増やす

後半に持ってきてしまったが、一番早いやり方。「開発者の人数が多くて詰まっている」というケースであれば有効。

逆にいえば1回のテストの実行時間が減るものではないので、開発者が少なくて詰まってるわけではないが純粋にテストが遅い、という場合効果は薄いだろう。

Plan Settingから増やすことで可能。
1台分は無料で、2台以降$50ずつ課金される形式となる。

なかなか絶妙なラインではあるものの、他の手法をとった場合の人件費など考えると思い切って増やしてから効率化していくというのがやりやすいと思う。


## 1-2. Performance Pricing planに加入する。
Performance Pricing planというちょっと隠しっぽいものがあり、これに入るとマシンのスペックそのものを上げる`resource_class`という設定を使えるようになったり、dockerのレイヤーキャッシュが使えるようになるらしい。

* [resource_class](https://circleci.com/docs/2.0/configuration-reference/#resource_class)

自分で使ったことが無いのでわからないが、問い合わせる必要があったり戻せないなどあるらしいので、そのへんは注意

* [CircleCIの新しい有料プラン Performance Pricing plan について](https://qiita.com/ara_tack/items/3386c59b59789f76b1e8)
* [CircleCI 2.0 の resource_class で CPU と RAM のリソースを変更してみる](https://tech.gamewith.co.jp/entry/2017/12/25/185239)


# 2. テスト実行を並列化する
並列化は1台CI環境で行おうとすると、DBを分割したりと結構な面倒くささがあったが、CI環境や、dockerであれば、そもそも環境が独立してるので、少しは楽に出来るようになっている

## 2-1. 独立した処理をjobとして別に分ける (workflow)

多くの場合、testとlintなど独立して処理出来る事がある。
あとはサーバー用とクライアント用のテストなども独立しうる。これらはworkflowの機能で独立させることが出来る。

workflowは、複数の処理を独立させ、並列や直列に並べるものとなる。
https://circleci.com/docs/2.0/workflows/
独立されたjobはそれぞれ別なコンテナで動作する。

最初は若干とっつきづらいが、読み方がわかればそこまで難解なものでもないだろう。

例えばサーバーとフロントで分離するならこんな具合

```yml
version: 2
jobs:
  fornt-test:
    docker:
      - image: circleci/node
    steps:
      - checkout
      - run: yarn install
      - run: yarn test
  server-test:
    docker:
      - image: circleci/ruby
    steps:
      - checkout
      - run: bundle install
      - run: bundle exec db:create test
      - run: bundle exec rspec
workflows:
  version: 2
  build:
    jobs: 
      - front-test
      - server-test
```

単純に独立させるだけなので、比較的簡易に導入できるだろう。

当然その分、1つでも重い処理があれば１つのPRに対するテスト時間は縮まらないなどの弱点もある。
とはいえ複数台コンテナを契約していれば、使い終わったら順次開放されるため全体的にはサイクルのスピードが上がるだろう。

## 2-2. parallerismで同じタスクを並列化する

いわゆるほんとの並列化。
効果は高いが難易度も他の手段に比べると多少高いため、後回しにしてもよいだろう

```yml
jobs:
  build:
    docker:
      - image: some-image
    parallelism: 3
```
parallelismの値を指定すれば、単純にマシンが並列化される。

当然そのままだと無駄に同じ処理が走るだけなので、各マシンでテストするファイルを分散するようにしないといけない

https://circleci.com/docs/2.0/parallelism-faster-jobs/

例えばrspecの場合はこんな具合になる。


```yml
      - run:
          # DBのセットアップ。parallerismを使った場合、それぞれのdockerでDBを分離出来るので、普通に実行して良い
          name: db:setup
          command: |
            bundle exec rails db:setup test
      - run:
          # testするファイルを特定して、rspecに渡している
          name: rspec
          command: |
            bundle exec rspec -- $(circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=filesize)
```

実行するファイルを分離するための`circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=filesize`

というコマンドが少々奇妙で壁になるかもしれない。
`circleci`というCLIコマンドが自動で挿入されるので使える（らしい。細かいところは未検証）

https://circleci.com/docs/2.0/local-cli/#installing-the-circleci-local-cli-on-macos-and-linux-distros

これは単なるutilなコマンドで、`circleci tests glob`は単純にglobしており、それをpipe(`|`)で`circleci tests split`というコマンドに渡して分離している。

あくまでutilなので、ファイルを適切に分割できれば何でも良い

「いやいやそんなのわけわからんので分割したノード番号がほしい」という場合は、`CIRCLE_NODE_INDEX`, `CIRCLE_NODE_TOTAL`という環境変数で取得が出来る。


## 2-3. 多段化する (workspace)

CircleCIのworkflowは、それぞれが独立して動くために、リソースを共有出来ない。
ここで`persist_to_workspace` , `attach_workspace`を利用することで共有する事ができる。

例えば「準備段階」と「実行段階」を分け、「準備は並列=1でやって、テスト本体は並列=4でやりたい」みたいな場合に利用できる。

例えばprepareで依存関係だけ解決して、testに流したいときはこんな具合。

```yml
version: 2
jobs:
  prepare:
    docker:
      - image: circleci/node
    steps:
      - checkout
      - run: yarn install # インストール処理
      - persist_to_workspace:
          root: ~/project # デフォルトのworking_directiroy
          paths: # 保持したいパスを指定
            - ./node_modules 
  test:
　　　　　　　　# parallelism: 2 ← 例えばこちらだけ並列化して使い回すというようなことをする
    docker:
      - image: circleci/node
    steps:
      - checkout
      - attach_workspace: 
          at: ~/project # デフォルトのworking_directiroy
      - run: yarn test # テストコマンド実行
workflows:
  version: 2
  build:
    jobs: # prepare -> testの順に行う
      - prepare
      - test:
          requires:
            - prepare
```

[railsの場合はこんな具合](https://github.com/CircleCI-Public/circleci-demo-workflows/blob/workspace-forwarding/.circleci/config.yml)のようだ。

今回は簡易的な例示としてnpmの依存解決とテスト実行を分けるのに使ったが、公式ドキュメントの使い分けとしてはnpmやgemは`save_cache`/`restore_cache`を使うと書いてある。(https://circleci.com/docs/2.0/configuration-reference/#attach_workspace)

キャッシュについてはまた後半に記述しているので、ご参照いただきたい。


本来のpersist_workspaceの使い所としては、例えば`[フロントエンドのコードビルド -> rspecのintegraitonテスト]`のような流れの場合はわかりやすいだろう。
アプリケーションに依存するビルドなど外部ライブラリのようにキャッシュしづらい場合だ。

個人的にはcacheのキー管理を考えたり、インストールの方法によってはキャッシュがし辛い場合もあるので、workspaceが楽なケースはありそうと感じる。
workspaceのprepareの段階でcacheを組み合わせるみたいなのも良いだろう。


## 2-4. 一部の処理を他のサービスに移譲する
並列化と言って良いかわからないが、考えうる手段の一つとして並べておく。

CirlceCI（に限らずCIツール）は割とコマンドでなんでも出来てしまうが、場合によっては複数のサービスに責務を分割するという手も考えてよいだろう。

当然、料金や手間が逆に増えてしまうパターンはありえるが、CircleCIのymlやツールそのものと格闘したくない場合は検討の余地があるだろう。

観点としては、下記のような部分を他に出したらどうか？など考えるのが良いだろう。

* Lint / Format
    * https://github.com/marketplace/category/code-quality
* Coverage
    * https://github.com/marketplace/category/code-review
* Deploy
    * https://github.com/marketplace/category/deployment

## 2-5. システム・リポジトリを分割する

少しアクロバティックな話になるが、もう一つ考えうる手段として、システム自体の分割についても記載しておく。
もしすぐにシステムやリポジトリが分割可能なのであれば、早いうちに分割するのが良いだろう。
システムが肥大化しそうなタイミングで適切に分割を繰り返していければそもそもスローテストが起きづらい環境が出来るはずだ。

モノリシックなシステムは管理はしやすいし、途中からマイクロサービスに分割していくのにエネルギーが必要なケースも多い。
しかしどこかで分けなければ組織の成長についていけなくる可能性もあるだろう。

仮に今すぐには出来ないとしても、少しず分割を計画していくことでサービス成長の足を引っ張る前には解決出来るかもしれない。

また、チームの形態によってはシステム自体を分けずとも、フロントとバックエンドでリポジトリだけ分割するなども一つ検討の余地があるだろう。

# 3. テスト実行を絞り込む

テストを実行しないということはそれだけ抜けも発生するので十分慎重に考えるべきことではある。
しかし遅すぎてCIが適切に回らないという場合であれば、一部のテストを実行しない（または任意のタイミングでのみ実行する）ということを検討しても良いだろう。

## 3-1. カテゴリ化テストで重いテストにマーキングする
JUnitで言うところのカテゴリ化テストというもの。

```java
    @Category(SlowTests.class)
    @Test
    public void someTest() {
        slowFunction()
    }
```

CIツールの場合、お約束的に環境変数に`CI=true`という値が入ってくるので、これを使える

* [CircleCI](https://circleci.com/docs/2.0/env-vars/#built-in-environment-variables)
* [TravisCI](https://docs.travis-ci.com/user/environment-variables#Default-Environment-Variables)
* [Codeship](https://documentation.codeship.com/basic/builds-and-configuration/set-environment-variables/#default-environment-variables)

### 各テストフレームワークでのskip
だいたいのテストフレームワークは環境変数を見ることが出来るので、skipする方法が提供されていれば、組み合わせてスキップ処理をかけることが出来るはずだ。

「CIのとき必ず飛ばしたいわけではなく、masterだけ走らせたいんだ」みたいな場合は`EXECUTE_SLOW_TEST=1`みたいな環境変数を別途設けて、それを利用すると良いだろう

#### rspec
https://relishapp.com/rspec/rspec-core/docs/pending-and-skipped-examples/skip-examples

```rb
describe SomeTest do
  it 'SomeTest', :skip => ENV["CI"] do
    hoge.should eq(fuga)
  end
end
```

#### mocha

https://mochajs.org/#inclusive-tests
mochaでは`this.skip()`が使える。（thisを利用するためallow functionでは動かない）

```js
describe("SomeTest", function() {
  it("test", function() {
    if (process.env.CI) {
      this.skip();
    }
    console.log("Some test");
  });
});
```

jestの場合は`test.skip`などは用意されてるが、内部で状態をみたskipはまだ実装されてなさそう。下記のように泥臭くやるしかなさそう。
https://stackoverflow.com/questions/32723167/how-to-programmatically-skip-a-test-in-mocha

#### phpunit
https://phpunit.de/manual/6.5/ja/incomplete-and-skipped-tests.html

```php
if(getenv("CI")){
    $this->markTestSkipped("skip test");
}
```

## 3-2. 絞り込んたテストの実行タイミングを制限する
テストの実行タイミングを制御する手法を取り上げる。
これを上記カテゴリ化テストと組み合わせて「masterのときのみslow testを含める」「深夜だれもいない時間帯に全テストを実行」などいろいろ検討する幅ができるだろう。

### 3-2-1. branch名やtag名でfilterする
[Git Tag Job Execution](https://circleci.com/docs/2.0/workflows/#git-tag-job-execution)

CircleCI 2.0 では workflowを作る
特定の命名規則で

```yml
workflows:
  version: 2
  build:
    jobs:
      - build:
          filters:
            branch:
              ignore: /no-test-.*/ # no-test-から始まったらテストしない
  build-with-slowtest
    jobs:
      - slowtest:
          filters:
            branch:
              only: # masterとdevelopだけslow test実行
              　- master
              　- develop
```

### 3-2-2. scheduleで実行するタイミングを指定する
[scheduling-a-workflow](https://circleci.com/docs/2.0/workflows/#scheduling-a-workflow)
cronの書式で、
カテゴリ化と組み合わせて、「深夜だけslow testを含めて実行」など検討しうる

```yml
workflows:
  version: 2
  build: # 通常のビルド
    jobs:
      - test
      - deploy
  nightly: # 夜だけビルド
    triggers: # triggerでタスクを指定 
      - schedule: # scheduleで実行
        　cron: "0 1 * * *" # UTCで記述。この場合は朝10時
          filters:
            branches:
              only:
                - master # masterだけ実行
```


### 3-2-3. `type: approval` でテストの実行を手動にする
[Holding a Workflow for a Manual Approval](https://circleci.com/docs/2.0/workflows/#holding-a-workflow-for-a-manual-approval)

例えば、「テストしないわけにはいかないけど、pushするたびにはしたくない、かといって`[skip ci]`って毎回書くのもめんどうだし。。。」
みたいな場合はapprovalというのが使えるだろう。

```yml
workflows:
  version: 2
  build:
    jobs:
      - hold:
          type: approval
          filters:
            branch:
              ignore: /wip-.*/ # wip-から始まったら手動テストにする
      - build:
          requires:
             - hold
```

ボタンを押すと確認が出る感じになる。

![画像](https://circleci.com/docs/assets/img/docs/approval_job_dialog.png)

ただ、CircleCI画面まで行ってボタン押さないといけないのは若干厄介かもしれない。


# 4. テスト実行速度を上げる

## 4-1. 依存ライブラリなどをキャッシュする (restore_cache / save_cache)
[Writing to the Cache in Workflows](https://circleci.com/docs/2.0/caching/#writing-to-the-cache-in-workflows)

`restore_cache`と`save_cache`を使うことで、依存関係で持ってきたファイルをキャッシュをすることが可能。gemやnpmのインストールが時間が短くなる。

効率的なのだがすこし記法がややこしいかもしれない。

```yml
      # キャッシュを復元
      - restore_cache:
          keys: # keyは複数指定できる。上から順になければ下のキーのキャッシュを見に行く
            - rails-demo-{{ checksum "Gemfile.lock" }}
            - rails-demo-
      # bundle install
      - run:
          name: Install dependencies
          command: bundle install --path=vendor/bundle --jobs 4 --retry 3
      # キャッシュを保存
      - save_cache:
          key: rails-demo-{{ checksum "Gemfile.lock" }}
          paths:
            - vendor/bundle
```

先に紹介したworkspaceの機能と組み合わせて、workspaceの準備段階でさらにキャッシュを使う、ということも考えられるだろう。

## 4-2. Docker Imageを先に作っておく
もしプロダクトが非常に複雑（かつDockerに詳しいメンバーがいる）などであれば、既存で用意されているのと別に、独自のdockerイメージをあらかじめ作っておきpushしておくという手があるだろう。

* https://circleci.com/docs/2.0/custom-images/
* https://circleci.com/docs/2.0/private-images/
* https://circleci.com/docs/2.0/configuration-reference/#docker

以前はECRからのImage取得が難しかったが、今はaws_authを指定すれば取得することが出来る。

ドキュメントで紹介されている
https://github.com/circleci-public/dockerfile-wizard
は、「基本はありモノ使いまわしつつちょこちょこいじりたい」という場合に使えそうだ。
当然docker hubなどを利用する必要はあるが、特段秘匿する必要が無いイメージであればこれでも十分そうだ

## 4-3. DBへのインサートを減らす・Factory・mockを利用する
これは昔からある一般的な話なので、列挙だけしておく。

テストごとにSeedを流すのをやめたり、FactoryBotでいえば`create`を`build`にする、外部通信しているようなテストをmockを利用するなどテストを早くするものはいろいろある。

このへんはアプリケーション固有の話になるので、テスト実行時のログを見ながらボトルネックを探していくことになるだろう。ぐぐるといろいろ出てくるので割愛する

## 4-4. CircleCIの記述をshellにせず、ymlに書く（未確定な情報）

正直これはなぜなのかわからないが、CircleCIでymlで書いていたものをshell化すると遅くなり、ymlに戻すと一瞬で終わる現象を確認したことがある
（commandの処理がどこかにキャッシュされている？）

もしかするとバグや考慮漏れに近い可能性もあるので、将来的には変わらなくなる可能性があるものの、shellファイルの実行がやたら時間がかかっている場合、一度ymlにうつしてみるといいかもしれない。
処理を共通化したい場合はymlのalias記法を使ったりして共通化できる。
下記のように記述することで、複数のコマンド実行も可能だ。

```yml
- run: |
    some command1
    some command2
```

公式ドキュメントには様々なconfigサンプル集があるので、困ったら見てみると良いだろう
https://circleci.com/docs/2.0/examples/
個人的には[facebook/react-native](https://github.com/facebook/react-native/blob/master/.circleci/config.yml)のconfigがいろいろすごくて学びになった
