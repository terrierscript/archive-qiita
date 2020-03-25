
[npmrc](https://docs.npmjs.com/files/npmrc)のドキュメントを読みなおしていたら、`.npmrc`は`/path/to/my/project/.npmrc`のようにプロジェクト毎に配置することが出来る事に気づいて、ちょっと使ってみたら便利だった。

globalやhomeディレクトリへの設定を前提とした`npm config`の記事は結構あるが、プロジェクト毎で`npm-config`について書かれている記事があんまり無かったのでまとめてみる。

## npm-configの何が良いのか？ project毎に出来ると何が良いのか？

[npm-config](https://docs.npmjs.com/misc/config)の設定をすると、色々コマンドを省略出来たりして良い事がある。
 (参考:[2016年版 Node.jsで幸せになれる10の習慣](http://qiita.com/ksato9700/items/b21383e613b6dc308dca#2-%E8%B3%A2%E3%81%8Fnpmrc%E3%82%92%E4%BD%BF%E3%81%88))

npmrcやnpm-configは、個人開発用であれば、`$HOME/.npmrc`への設定だったり`npm config set`での設定で十分。
また、npm registryに登録するライブラリやオープンソース向けのプロジェクトの場合だと、プロジェクト毎の`.npmrc`があるとむしろ弊害になることもありそう。

しかし、frontend向けの開発をチームで行う際では色々設定しておくのは便利に使えるものが多い。

例えば「`npm shrinkwrap`するときは`--dev`つけてください」と言わなくても済むなど効果が見込める。


### この記事の対象者・対象プロジェクト
- npmを実行するメンバーが一定数以上いるプロジェクト
	- 個人開発とかならglobalで設定しましょう
- node.js、javascriptに明るくないメンバーが一定数いるプロジェクト
	- みんなが詳しいチームだったら不要でしょう
- npm registryに登録したり、オープンソースで運用したりする予定ではないプロジェクト
	- 例：frontend開発のために、npmをパッケージマネージャとして利用しているプロジェクト
	- オープンソース前提の場合は、多分無いほうが良いでしょう

# npmrcの基本・記法

## 記法

iniファイルっぽい。こんな感じ

```.npmrc
some-setting=value
```

複数値を設定する方法も紹介されている

```.npmrc
key[] = "first value"
key[] = "second value"
```
## 故障かな？と思ったら
意図した設定されてない！となった場合は、下記のコマンドを叩くと良い

```
$ npm config list
```

これで、`.nprmc`のパース結果を出してくれる。globalの設定も含めてくれる。

間違った値を入れた場合はこの結果一覧には出てこなくなる。

# 個人的おすすめ便利設定

とりあえず個人的におすすめ設定を晒してみる。
詳細は後述

```.npmrc
### だいたい設定しといて良さそう（スタメン）
also=dev
save-exact=true
progress=false
engine-strict=true

### 場合によっては使うことがありそう
# loglevel="error"
# if-present=true
# save-prefix="~"
# link=true
```

# スタメン（だいたいのプロジェクトで使える系）
## [also](https://docs.npmjs.com/misc/config#also)=dev

`dev`か`development`を設定することで、`shrinkwrap`、`outdated`、`update`コマンドを実行するときに`dev`を含めるようにしてくれる。
`shrinkwrap`などは`--dev`つけないと警告が辛い事がほとんどなので、`dev`にしておくのがベターなイージ

この逆っぽい[`only`](https://docs.npmjs.com/misc/config#only)というのもある。（個人的に必要になった事が無いのでそんなに調べてない）


## [save-exact](https://docs.npmjs.com/misc/config#save-exact)=true

`npm install`、`npm rm`時に自動的にオプションをつけるように変更してくれる。

`save-exact=true`にすると、`npm install -S some-package`が`npm install -S -E some-package`と同じ動きをしてくれる。

フロントエンド向け開発だと、`shrinkwrap`をする場合が多いので、結局`exact`もしてしまって差し支えない事がそれなりにある。
そういう場合にこのオプションが便利。


## [progress](https://docs.npmjs.com/misc/config#progress)=false

falseにするとnpm installするときに出てくるprogressバーを抑止出来る。
「[progressしてるとnpm installめっちゃ遅い！](https://github.com/npm/npm/issues/11283)」という話があったりするので、progressあんまりいらないな、と思ったら切って良い。

CI向けの高速化としても使える。

## [engine-strict](https://docs.npmjs.com/misc/config#engine-strict)=true

install対象のpackageの[package.json](https://docs.npmjs.com/files/package.json#enginestrict)の`engines`を見て、Node.jsのバージョンが合致しなかった場合は`npm install`を失敗扱いにしてくれる。

ちゃんとしたパッケージならそれなりのバージョン対応しているし、このへんが甘めなパッケージは`engines`を設定してない可能性が高いので防御力は低いが、それでも無いよりマシぐらいな物ではあるだろう。


# ベンチ入り（必要に応じて使いそう）
## save関連
save-exact以外も結構色々あるので、必要に応じて使い分ける

### [save](https://docs.npmjs.com/misc/config#save), [save-dev](https://docs.npmjs.com/misc/config#save-dev) 

```.npmrc
save=true
save-dev=true
```

「とりあえずこのプロジェクトは全部`devDependency`にしときたい」という場合があればこのあたりが使えそう

ちなみに、`save-dev=true`にすると、`npm install -S hoge`でも`devDependency`に保存されてしまう。バグなのか仕様なのかは追ってない。

[save-bundle](https://docs.npmjs.com/misc/config#save-bundle)、[save-optional](https://docs.npmjs.com/misc/config#save-optional) というのもあるが、あんまり用途が思いつかない

### [save-prefix](https://docs.npmjs.com/misc/config#save-prefix)

```.npmrc
save-prefix="~"
```

defaultで`npm install`すると、`^1.0.0`という、バージョニングになるのを変更出来る。

参考：[semantic versioning](http://semver.org/lang/ja/)

### [link](https://docs.npmjs.com/misc/config#link)

```.npmrc
link=true
```

`npm install`時の挙動を`npm install --link`にする。
結構癖があるイメージなので、ちょっと取り扱い注意かもしれない

## エラー抑止・CI関連
### [loglevel](https://docs.npmjs.com/misc/config#loglevel)

```.npmrc
loglevel=error
```
npm実行のログレベルをいじる。デフォルトは`warn`。

warnだと、依存したパッケージのエラーが出てしまってどうしようもなかったりすることがある（当然消せたほうが良いが）。
あまり綺麗な方向ではないのですごく悩ましいが、チームの考え方次第では`error`に設定するのもアリかもしれない。

ちなみに、この設定があっても下記のようなオプションをつければログが出てくれる。

```
$ npm i --loglevel=warn
```


### [if-present](https://docs.npmjs.com/misc/config#if-present) 

```.npmrc
if-present=true
```

`npm run sonza-sinai-command`のように、存在しないコマンドを叩いた時、デフォルトではerrorを吐くが、何も吐かなくなる。
circle-ciなどCIツールの邪魔をしないようにするなどが想定されている。（`npm set if-present true`とかでも十分かもしれない）

当然だが、設定すると、エラー時に何が起きたかわからない。

### [color](https://docs.npmjs.com/misc/config#color)

```.npmrc
color=false
```
色をオフにする。CIとかで使いたい時に使う


## 通信系
### [proxy](https://docs.npmjs.com/misc/config#proxy), [http-proxy](https://docs.npmjs.com/misc/config#https-proxy)

Proxyを通す必要がある場合など、プロジェクト単位で設定しておけばデザイナーなど非エンジニアの環境セットアップが少し楽になりそう。
個人的に必要に迫られたことが無いのであんまりちゃんと追ってない


## 補欠（おまけ。工夫次第で使えそうなもの）
別名：global向けなのでprojectスコープだと使い所が見つけづらいもの

### [heading](https://docs.npmjs.com/misc/config#heading) 

```.npmrc
heading="some-heading"
```

コンソール出力の「npm」ってなっているところを変更できる。

絵文字とか使えるので、コンソールを華やかにしたいときに使える

```.npmrc
heading="🌸"
```

```
% npm i react                                              
example-dir@1.0.0 /project/example-dir
└── react@15.3.0

🌸 WARN example-dir@1.0.0 No description
🌸 WARN example-dir1.0.0 No repository field.
```

### [editor](https://docs.npmjs.com/misc/config#editor)

```.npmrc
editor="emacs"
editor="vi"
```

`npm edit` `npm config edit`時に利用するエディタを`$EDITOR`を使わせずに、強制させる。
どうしてもエディタについて喧嘩をしなければならない時の嫌がらせとして使える。

同類で[`shell`](https://docs.npmjs.com/misc/config#shell)というのもある。shellで争う場合はこちら。

※ みんな仲良くnpmを使おう

### [searchexclude](https://docs.npmjs.com/misc/config#searchexclude)

```.npmrc
searchexclude=grunt
```
`$ npm search`する際のexclude条件を指定する。
なにか強い意思で特定のプロダクト関連を検索させたくないという場合はつかえるが、そもそも`npm search`を使わないので、プロジェクト毎設定する意味は無い。（global設定だとそれなりに活きる）
