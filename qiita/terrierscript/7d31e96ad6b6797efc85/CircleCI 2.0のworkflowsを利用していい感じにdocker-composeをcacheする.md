
# 前置き
CircleCIのworkflowで上手いことdocker-composeの重い処理をなんとかするのをやってみた。

今回はdocker-composeのbuild処理が重くてtest並列化するとそこがボトルネックになるような場合を想定している。
build自体が重くない場合や並列化が不要な場合は、むしろcache生成・呼び出しで遅くなるパターンもある。

[Enabling Docker Layer Caching](https://circleci.com/docs/2.0/docker-layer-caching/)など公式のdockerキャッシュもあるが、machineモードでないとdocker-composeがフルに利用できない（[volumesが利用できない](https://circleci.com/docs/2.0/docker-compose/)）のでsave_cache / restore_cacheを使うやり方を色々ひねった。

# 手始めにdockerをcacheしてみる

docker-composeに飛びつく前に、docker単体のpullとtestをworkflowで分離してみる。

dockerのcache利用自体は下記で語られているのをほぼ利用する。

* [CircleCI 2.0 Beta における Docker イメージのビルド](http://lambdar.hatenablog.com/entry/2017/04/18/084900)
* [CircleCI 2.0 では Docker ビルドキャッシュが効く](http://qiita.com/na-o-ys/items/7e7a7e4cde378fc54b32)

docker imagesをまるっとsaveするやり方は下記を参考にした

* [How to save all docker images and copy to another machine?](https://stackoverflow.com/questions/35575674/how-to-save-all-docker-images-and-copy-to-another-machine)

今回はなんでもよかったが、nodeのimageを利用した。

```circle.yml
version: 2
jobs:
  generate_cache:
    machine: true
    steps:
      - checkout
      - run: docker pull node:8.2.1-onbuild
      - run: # cacheのために吐き出し
          command: |
            mkdir -p ~/caches
            docker save $(docker images -q) -o ~/caches/images.tar
      - save_cache:
          key: docker-{{ .Revision }}
          paths: ~/caches/images.tar
  test:
    machine: true
    steps:
      - checkout
      - restore_cache:
          key: docker-{{ .Revision }}
          paths: ~/caches/images.tar
      - run:
          command: |
            set +o pipefail
            docker load -i ~/caches/images.tar | true
      # - run: docker images # imageが持ってこれてることを確認したければこんなコマンドを差し込む
      - run:
          name: pull # キャッシュ利用しているため、早い（0秒）
          command: docker pull node:8.2.1-onbuild
workflows:
  version: 2
  build:
    jobs:
      - generate_cache
      - test:
          requires:
            - generate_cache
```

cacheのキーはリビジョンを利用した。`test`側でのbuildが0秒になる。

# docker-composeをcacheする。

やり方はほぼ一緒。`docker-compose pull`と`docker-compose build`をしてsaveする。

```circle.yml
version: 2
jobs:
  generate_cache:
    machine: true
    steps:
      - checkout
      - restore_cache:
          key: docker-{{ .Branch }}-{{ checksum "docker-compose.yml" }}-{{ checksum "Dockerfile" }}
          paths: ~/caches/images.tar
      - run:
          command: |
            set +o pipefail
            docker load -i ~/caches/images.tar | true
      - run: pip install docker-compose
      - run: docker-compose pull
      - run: docker-compose build
      - run:
          command: |
            mkdir -p ~/caches
            docker save $(docker images -q) -o ~/caches/images.tar
      - save_cache:
          key: docker-{{ .Branch }}-{{ checksum "docker-compose.yml" }}-{{ checksum "Dockerfile" }}
          paths: ~/caches/images.tar
  test:
    machine: true
    # parallelism: 3 # 並列化したいときはparallelismを利用する
    steps:
      - checkout
      - run: pip install docker-compose
      - restore_cache:
          key: docker-{{ .Branch }}-{{ checksum "docker-compose.yml" }}-{{ checksum "Dockerfile" }}
          paths: ~/caches/images.tar
      - run:
          command: |
            set +o pipefail
            docker load -i ~/caches/images.tar | true
      - run: docker-compose pull # 早い
      - run: docker-compose build # 早い
      - run: お好きなテストコマンド
workflows:
  version: 2
  build:
    jobs:
      - generate_cache
      - test:
          requires:
            - generate_cache
```

こんな具合でgenerate_cacheでdocker-compose周りの遅い処理を片付けてしまって、testを並列化することが出来る。


