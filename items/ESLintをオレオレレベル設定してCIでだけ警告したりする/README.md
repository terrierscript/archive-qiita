# ESLintでCIのときだけエラー警告してほしい

## おさらい：ESLint の標準的なレベル分け

ESLintでのレベル分けは３つ定義されている

http://eslint.org/docs/user-guide/configuring#configuring-rules

* `off`, `0` - オフ。何も警告しない
* `warn`, `1` - 警告するが、exit codeに影響を与えない（CIではコケない）
* `error`, `2` - 警告する。exit codeを1にする

## 問題：いい具合のレベル感がない

プロダクション環境でESLintをかけていると、`no-console`や`no-unused-var`など、「これproductionビルドまでにはエラーとして検出したいのだけど、開発中は見逃してほしい」みたいなのがたまにある。
（`error`にしてしまうとstorybookとかでエグくなって辛い等もあった）

ESLint自体のエラーレベルは三段階のみで、CLIからwarnがあってもexit codeを1で返すようなコマンドも見つからなかった（多分このへんはレベル管理をシンプルにしているからだと推測できる）


## 解決法：レベル分けの定数管理を利用してCI判定

reactの[`eslintrc.js`](https://github.com/facebook/react/blob/v15.4.1/.eslintrc.js)を見てみるとこのエラーレベルを定数定義していることがわかる

```js
const OFF = 0;
const WARNING = 1;
const ERROR = 2;
 :
 rules: {
    'accessor-pairs': OFF,
    'brace-style': [ERROR, '1tbs'],
 }
```

なるほど。そういうのもあるのか。

これを応用して、CIの場合だけWARNINGの挙動をエラー化させれば良い。circle-ci, travis-ciはどちらも`CI=true`という環境変数を設定しているので、これを利用する


```js
const ERROR = "error"
const WARNING = (process.env.CI == true) ? "error" : "warn"
const OFF = "off"

```

これでCI時のみ警告出来るようなった。

* https://docs.travis-ci.com/user/environment-variables/#Default-Environment-Variables
* https://circleci.com/docs/environment-variables/

`process.env.CI`が使えないなら自前で変数設定してもいいんじゃないかと思う


```js
const ERROR = "error"
const WARNING = (process.env.STRICT_LINT == true) ? "error" : "warn"
const OFF = "off"

// こんな感じもアリだろう
// const WARNING = (process.env.STRICT_LINT == true || process.env.CI == true) ? "error" : "warn"



```

```json
{
  "scripts": {
    "lint": "STRICT_LINT=true eslint"
  }
}
```

# 応用編：意味付けでオレオレレベル設定をする

上記のレベルの定数管理を見て思いついてやってみてる案。

ESLintはレベル分割を最低限の挙動の違いだけをシンプルに分割している。

だが実は実務上だと「このルールなんでつけたんだっけ？」みたいなのもあって、もうちょっと意味を持たせたくなったりする。ということで色々細分化させてみたら良いんじゃないかと思いついている（逆にわかりづらくなったりするかもしれない）

だいたいこんなイメージ

```js
// no-evalとかコードの書き方が変わる禁止事項
const CRITICAL = "error"

// エラーなのだけどデバッグ時には無視していい警告だけする
const ERROR = (process.env.CI == true) ? "error" : "warn"

// たまに気が向いたら直すルール（無い方が良いかも？）
const WARNING = "warn"

// OFFはそのまま
const OFF = "off"

//// ここから読み手向けなど ////

// 「手で治すのめんどいけどfixableだったらつけとくか」的なルール。dev時は出さない
const FIXABLE = (process.env.CI == true) ? "error" : "off"

// おためしルール。きつかったらチームで相談しようね、という宣言
const TRIAL = "error"

// 本当はつけたいけど、直すの重くて妥協しているやつ。rubocop_todo的なやつに近いイメージ
const TODO = "off"
```

FIXABLEは結構「これなんでやったんだっけ」ってなるやつが多いので、結構いいと思っている。TRIALとTODOは思いついたものの使い所が無くて今のところ運用出来てない


