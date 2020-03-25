@eleven_2012 
文字列からの数値化についてはgensimの`corpora.Dictionary`を利用しました。
ざっくり主要な部分のコードとしては、こんな感じでサンプルデータを結合したデータを食わせました

```py
from gensim import corpora

chars = [ list(r.strip()) for r in concatinated_sample_string_data ]
dic = corpora.Dictionary(chars)
```

ここらへんのやり方はTensorFlowで同様のことを行っていた下記スクリプトも参考の一端としましたので、こちらも参考になるかもしれません
https://github.com/dennybritz/sentiment-analysis/blob/master/utils/ymr_data.py#L36
