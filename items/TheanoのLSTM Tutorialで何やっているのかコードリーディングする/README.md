
http://deeplearning.net/tutorial/lstm.html

このサンプルが中で何やってんのかざっと読んだ時のメモ。
LSTMの本実装部分は正直全然読み込めてない。

一旦ある程度自由に検証出来るようにいじれるぐらいまでの読み具合。

# 訓練データの分類
コード全体として、入力される訓練データは3種類ある。

- `train`
	- 学習に使われるデータ
- `valid` 
	- 誤差率を検証するためのデータ
	- 誤差の数値はhistoryとして記録されている
- `test` 
	- 訓練とは独立して誤差検証に利用されるデータ
	- 過学習が起きてないかどうかを検証するために使っているっぽい。
  	    - 実訓練とは完全に独立して利用されている。
	- `valid`のエラー率同様、`test`の誤差率もhistoryに記録される（後述）

# コードリーディング
## [imdb.py](http://deeplearning.net/tutorial/code/imdb.py)

データの準備、などを行っている。独自データを扱う際にはこの部分の拡張をするのが一番手軽。

### `prepare_data()`
- 複数の訓練サンプルを受け取り、転置行列、labelとして渡された値、maskの配列を返す。
- `maxlen`で指定された以上の要素数を持つデータは、除外される（縮められるとかではない）
- 独自データを使いたいとしてもこっちをいじる必要はなさそう

### `load_data()`
- train, valid, testのデータを生データから準備している
- 引数
	- `path`:キャッシュ的な働きをしてるっぽい
		- [ここ](http://www.iro.umontreal.ca/~lisa/deep/data)からデータをダウンロードしたりとかやっている
		- 元データが`train`用と`test`用がそもそも別れており、`valid_portion`に設定された割合で`train`のデータを`train`と`valid`に分割している
	- その他オプションでこのへん
		- `n_words`
		  - ボキャブラリの数の上限決定
		- `sort_by_len`
		  - 配列長でのソート。やったほうが早くなるらしい？
		- `maxlen`
		  - `prepare_data`と同様。これを超えているデータはスキップされる
	        
### [imdb_preprocess.py](http://deeplearning.net/tutorial/code/imdb_preprocess.py)
データ準備のためのスクリプト

- word -> idへの変換などをしているっぽい。
- [perl](https://github.com/moses-smt/mosesdecoder/raw/master/scripts/tokenizer/tokenizer.perl)を使ってtokenizeしている
	- なんかHTMLのタグとかを外してるっぽい。
	- わざわざperlなのは単に歴史的経緯とかに思える。（正直pythonでも全然出来そうな処理）
	- 多言語対応できてるっぽいけど当然、形態素解析とかしないとわかち書きできない日本語は未対応。

## [lstm.py](http://deeplearning.net/tutorial/code/lstm.py)
学習実行のスクリプト

```
$ python lstm.py
```
で動かせる
### メインの関数
#### `train_lstm()`
実質のエンドポイント。
ここ関数の引数が色々いじれるパラメータになっている。結構多い。
とりあえずいじりそうなものを列挙。

- 学習関連
	- `dim_proj`
		- hidden unitの数。
		- デフォルトの128で試すと結構時間かかる。
	- `vaildFreq`
          - 誤差率の検証を行う頻度に関する設定値。
	- `patience`
		- 早期終了を行うタイミングに関するの変数。
		- ざっくり見ると`validFreq`の結果が同一なパターンが`patience`回続くとEary stoppingする
	- `max_epochs`
	  - エポック実行の最大数
	- `use_dropout` 
	  - dropout層の有無。デフォルトTrue
	- `optimizer`
	  - 最適化関数。
	  - デフォルトはAdaGrad。AdaGrad, RMSprop, SGDが選べる
          - けど「SGDは扱い難しいから注意な」とある。
          - [こちらの記事](http://qiita.com/hogefugabar/items/1d4f6c905d0edbc71af2) を参考にすると、デフォルトのAdaGradは十分に精度が高いと見える。
	- `decay_c`
		- Weight decay。重み減衰。
		- まだちゃんと使ってない
- 学習データ関連
	- `n_words`
	  - ボキャブラリの最大数。デフォルト10000
	- `maxlen`
	  - １学習サンプルあたりの要素数上限。これが`imdb`の学習データ読み込みに渡される
- その他
	- `saveto`
	  - モデルの最終結果ファイルの出力先
	- `reload_model`
		- 前回保存したモデルを初期値として学習を開始する
		- 多分バグっぽいが、`lstm_model.npz`というファイルを読み込んでいる。
	- `dispFreq`
		- ログ表示頻度。
		- デフォルト10だが1にしとくほうが実行速度見れて楽しい

### その他ちょっと知っておくと良かった関数
#### `build_model()`
LSTMモデルを構築している部分。`train_lstm`で学習し終わったモデルの再現にも使う。

#### `init_tparams()`
パラメータをtheano向けに変換している。
`build_model`に投げるパラメータはこの関数通す必要アリ。

#### `init_params()`
LSTM以外についてのグローバル設定

#### `pred_error()`, `pred_probs()`
モデルを実行する関数ら。
`pred_error`は誤差計算に利用される。
`pred_probs`は結果出力をする。学習時には利用されない。
両者は`f_pred`を使うか`f_pred_prob`を使うかが違う。

`f_pred_prob`はそれぞれの確率の結果を出し、`f_pred`はそのうち最大である要素数（＝どのクラスに属したか）を返している。

#### `sgd()`, `adadelta()`, `rmsprop()`
最適化関数。optimizerで選ぶ奴

#### `param_init_lstm()`, `lstm_layer()`
LSTMの本実装部分。このへんから奥がまだ謎。
しかし`lstm_layer`でやっているコードと数式を照らし合わせるとだいたい同じような事が行われている事がわかる

### `dropout_layer()`
dropoutの実装部分。

# 構築済みモデルの中身に何入っているか

```python
model = numpy.load("lstm_model.npz")
```
numpyのデータ形式で保存されるので、`numpy.load`で読み込める。

- `train_err`,`valid_err`, `test_err`にそれぞれ最後に取得できたエラー率が入る
- `history_errs`が割りと便利そう。`validFreq`ごとに記録された誤差率が`[valid_err, test_err]`という形で入っている。
- その他はLSTMの学習結果のパラメータ
