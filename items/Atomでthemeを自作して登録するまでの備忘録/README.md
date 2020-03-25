
某イカのゲームっぽいsyntax作れそうな気がしてatomに作ったのでざっくりメモ。

基本的なドキュメントはこのへん。
https://atom.io/docs/v0.60.0/creating-a-theme
https://atom.io/docs/v1.0.2/hacking-atom-creating-a-theme

# 初期化編

## 普通なやり方
apmというのがあるので、npmと一緒で`apm init`みたいなことすると良いかなーと思ったけどもうちょっといいやり方あった。

`cmd-shift-P`でコマンドパネル開いて`Package Generator: Generate Syntax Theme`を選ぶ
![font_scss_-__Users_inuscript_github_example-iconfont-scss_-_Atom.png](https://qiita-image-store.s3.amazonaws.com/0/7307/893bdacf-7a73-74f1-62ea-93bcc9668f3c.png "font_scss_-__Users_inuscript_github_example-iconfont-scss_-_Atom.png")

これで選択した所にテンプレートができてる。

`atom-template`というのがgithubに公式っぽくあがっているが、微妙にそっちは古そうだった

## 既存テーマをベースにするときのやり方
「このテーマ、ここだけなんとかしたい」とか「このテーマをベースにしたい」
という時は、もうちょっと違うやり方がある。
その時はざっくりこんな感じ

1. githubでベースとなるやつをcloneする
- cloneしたリポジトリの名前を変えておく
- ローカルにcheckoutする
- `package.json`の`name`を変えておく
	- 同じ名前では多分登録できないか、上書きされてしまう気がする
- `$apm link`を実行
	- `$npm link`とにたようなやつ。
	- `~/.atom/packages/`にシンボリックリンクする

# 起動編
`atom --dev .` で開く。
既に先ほど作ったsyntaxがインストール済みになっているはずなので、設定からそのsyntaxを選ぶ
（これがわからず最初詰まった）

あとはlessファイルをいじるだけ！

## lessいじるときの細かいテクニックとか
### [Pigments](https://atom.io/packages/pigments)プラグインは入れるべき
色を背景に入れてくれるpluginはこれまでにもあったが
こいつはlessの変数までしっかり読み込んでくれる。
マジ便利。

### lessのcolor functionは積極的に使う
[このあたり](http://lesscss.org/functions/#color-operations)の関数とか特に使える

### `atom --dev`だと簡単にelementをinspectできる
「あれ、ここの色どういうセレクタで調整するんだ？」ってなったら右クリックすると「Inspect Element」というのが増えている
![font-awesome_css_-__Users_inuscript_github_-_Atom.png](https://qiita-image-store.s3.amazonaws.com/0/7307/ad729e2a-81ea-5a5d-36a6-b16b6a1b2367.png "font-awesome_css_-__Users_inuscript_github_-_Atom.png")

chromeみたいにInspectorが開くので、どんなセレクタが有効になっているのかとかわかる。

### 【欄外】他のthemeをベースにする時のtips
割りとニッチな話。

今回、[monokai](https://github.com/kevinsawicki/monokai)をベースにいじった。ベースがしっかりしているのだが、色名が変数化されてないという難点があるので、下記のどっちかをする必要があった。

- [このpull req](https://github.com/kevinsawicki/monokai/pull/49/files)を取り込む
- がんばって置換する

今回は後者を選んだ。「どの色をどれにあてはめる」かを後で入れ替えたかったりしたため、`@pink`などの変数名だと都合が悪かった。なので全部`@color1` `@color2`などと変数化してあとで置き換える事にした。

atomのPackage Generatorでは、`@red`,`@green`などの色が最初から定数化されている。
最初はこちらを使ったが、入れ替えの調整しているうちに`@red`に青色が入ったりしていってわけがわからなくなってしまった。
結果として最初の開発時には番号化してしまうほうがいっそ楽だった。


# 登録編
だいたいnpmと一緒。

1. `$apm login`する
	- 「APIキーを貼ってね」的なことを言われるので、貼る。
2. `package.json`のtagにあわせて`$git tag`登録する
	- ~~`npm tag`みたいなのはなさそうだった。~~
	- ↑`$apm publish patch`みたいなのだけでよかった
3. `$apm publish`する
	- `repository`フィールドのurlが間違ってたり、tagが編だったりすると怒られる。
4. atom上で確認する

# 結果

[できました](https://atom.io/themes/ika-ink-syntax)
青がやたら使いづらくて困っています。

