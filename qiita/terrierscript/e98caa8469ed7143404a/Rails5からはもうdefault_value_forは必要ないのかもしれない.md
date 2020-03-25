
# Rails4 -> Rails5 のアップデートで`ActiveModel::MissingAttributeError`で引っかかる。

Rails5にアップデートしたところ、`FactoryGirl`が`ActiveModel::MissingAttributeError`を出してしまっていた。

最初はなんだかよくわかっていなかったが、どうにも`default_value_for`を利用しているmodelが死んでいるようだった。

[Pull Request](https://github.com/FooBarWidget/default_value_for/pull/57)は既に出ているが、今のところ取り込まれてない（2016/07/08現在）

# どうすれば？ -> Rails5で導入されたActiveRecord attributeを使う。

Modelのデフォルト値は[attribute](http://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html)で対応出来る。

なので、こんな感じのコードを

```rb
class SomeModel < ActiveRecord::Base
  default_value_for :is_public, false
end
```

こうするだけで良くなった。

```rb
class SomeModel < ActiveRecord::Base
  attribute :some_flag, :boolean, default: -> { false }
end
```
