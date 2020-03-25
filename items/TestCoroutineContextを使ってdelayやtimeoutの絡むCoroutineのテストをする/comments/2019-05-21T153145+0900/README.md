`kotlinx-coroutines-core` にあった `TestCoroutineContext` は *Deprecated* になりました。

代わりに [kotlinx-coroutines-test:1.2.1](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-test/)から `runBlockingTest`, `TestCoroutineDispatcher`, `TestCoroutineScope` などが提供されています。

[Kotlin Coroutineのdelay()をユニットテストのときだけ待ち時間なしで実行する](https://qiita.com/kunigaku/items/ee061f563ba188349f59)
