
Travis.ciではできていた複数バージョンでのビルド。
「どーやるとできますかー」ってhelpで聞いてみたら、教えてくれた
（チャットっぽいヘルプUIすごい。circle-ciのサポート力高い。）

```circle.yml
test:
  override:
    - nvm alias default 0.12.0
    - npm test
    - nvm alias default iojs-v1.3.0
    - npm test
    - nvm alias default 0.10.34
    - npm test
```
こんな感じ。
nvm入っているので切り替えていけばいい。

元々教えてもらったやり方だと、デフォルトのマシンを`0.12.0`にして最初のnvm切り替えを省略していたが、いまいち可読性が良くなかったので全部nvmで切り替えることにした。


もしparallelにやりたいならこうやると良いらしい。個人的に需要はなかったのでこれは試してない。

```circle.yml
machine:
  node:
    version: 0.12.0
 
dependencies:
  pre:
    - if [ $CIRCLE_NODE_INDEX == "1" ] ; then nvm alias default iojs-v1.3.0 ; fi

test:
  override:
    - npm test:
        parallel: true
```
