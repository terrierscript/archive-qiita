
CircleCI 2.0において、複数のworkflow間で処理を共通化したい。
shellで共通化するなども出来るが、`run`で直接記載するのにくらべてかなり遅くなる（理由は不明だが、runの場合は何か裏でキャッシュをしてくれてるように見える）

しかし、workflowで複数のjobの処理を共通するときにどちらにも同じことを書くのはまた面倒になる。

そこでymlのアンカーとエイリアスで共通化することで解決できた

```yml
# アンカーをまとめる場所
references:
  commands:
    setup_php: &setup_php
      name: Setup PHP
      command: |
        sudo docker-php-ext-install mcrypt && sudo docker-php-ext-enable mcrypt
        sudo docker-php-ext-install pdo_mysql && sudo docker-php-ext-enable pdo_mysql
        sudo docker-php-ext-enable xdebug
        composer install -n --prefer-dist

    setup_node: &setup_node
      name: Setup Node
      command: |
        npm install
        npm run gulp

# circle-ciの設定
version: 2
jobs:
  test1:
    steps:
      - checkout
      - run: *setup_php
      - run: *setup_node
      - run: some command

  test2:
    steps:
      - checkout
      - run: *setup_php
      - run: *setup_node
      - run: some command2

workflows:
  version: 2
  test_and_buld:
    tests:
      - test1
      - test2
```

`referecnes`を上の方でまとめる手法はドキュメント化されてないが、「Add Project」時に「Others」を選ぶとアンカーとエイリアスを利用したサンプルが出て来る。
