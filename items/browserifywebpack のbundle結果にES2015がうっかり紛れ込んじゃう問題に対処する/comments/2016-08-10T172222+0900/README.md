> ミニマムなサーバーサイド向けのnpmパッケージであれば、ES2015でtranspileナシで書きたいという気持ちは非常に良くわかりますし、そこで元々想定していないブラウザ利用へのサポートを検討しなければならないと考えると、ちょっと辛いよなあと感じてしまい

その通りですね。「browserでもいけます」と謳っていないものに関しては、利用側の自己責任ですよね。
enginesフィールドさえ書いていれば免責されるかなと思います。

> package.jsonへのマーキングや、install時のWARNなど、npmがこのあたりにメスを入れてくれることに期待したいなと思っている次第です。

あとは、AST作って、ECMAの何に相当するのか確かめてくれてバッヂを提供してくれるサービスとかあったらいいですねー。
Universalな未来をつくるために自分も力になれる部分があればコミュニティに貢献したいです。
