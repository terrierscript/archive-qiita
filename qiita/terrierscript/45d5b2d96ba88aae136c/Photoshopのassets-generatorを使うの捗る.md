
Photoshop CCでsvg吐き出すのどうやんだろうかと調べたら
[こちら](http://stocker.jp/diary/photoshop-generator-svg/)の記事を見つけた。
ほほー、assets-generatorなんてあんのかーと色々調べると色々面白かったのでメモ

## セットアップ
1. homeディレクトリに`.generator.json`を作成
	- 最初`generator.json`じゃないといけないのかなー、ヤだなーと思っていたけど[調べたら](https://github.com/adobe-photoshop/generator-core/wiki/Generator-Configuration-File-Format)dotfile的な感じでもOKらしい。
	- homesickの管理下に置くと捗る

1. `.generator.json`をこんな感じで下記を記載
	- svgを吐き出したかったのでやったけどしなくても動くかも

 	```json
 	{
    	  "generator-assets":  { 
       	  	"svg-enabled": true,
         	"webp-enabled": true
          }
 	}
 	```
	- ほかにも[こんなオプションが設定可能](https://github.com/adobe-photoshop/generator-assets/wiki/Configuration-Options)

1. **Photoshopを再起動**（重要）

1. Photoshopのメニューから「ファイル」→「生成」→「画像アセット」を選択してチェックを入れる
	- このチェック、起動のたびに押さないといけなそう
1. ファイル名_assetsというディレクトリにもりもり出てくる

## 使い方＆Tips
- 基本的にはレイヤーの名前を`foo.png`のようにすることで画像がぼこぼこ生成される
- グループ化されたレイヤーにつけると、グループがまとまった状態で生成される
- 複数フォーマットほしいときは`foo.png, foo.svg`のようにできて熱い
- `200% foo.png` とすると二倍化された画像が出てきて熱い
	- `100x100 foo.png`なんかも[動くらしい](http://helpx.adobe.com/photoshop/using/generate-assets-layers.html)けど未検証

