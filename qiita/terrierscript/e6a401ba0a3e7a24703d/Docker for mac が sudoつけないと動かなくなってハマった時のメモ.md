
勢いで入れたDocker for macが`docker ps`しても`Cannot connect to the Docker daemon`とか言われてしまい、`sudo docker ps`としないと動かなくて`boot2docker`も消したのにおかしいなーとなってハマった。


### 解決

下記に`boot2docker`と`Docker for mac`の関係性についての記事があると教えてもらった

https://docs.docker.com/docker-for-mac/docker-toolbox/

`DOCKER_*`のenv設定があると、どうもboot2dockerを動かしてしまいDocker for mac見てくれないらしい。

```
unset DOCKER_TLS_VERIFY
unset DOCKER_CERT_PATH
unset DOCKER_MACHINE_NAME
unset DOCKER_HOST
```

`.zshrc`などにも上記設定に関する項目が入っている時はそっちも削除する

### 教訓

ドキュメントちゃんと読もう
