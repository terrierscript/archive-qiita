coroutineでタイムアウトを取り扱うようなテストをしようとした時に詰まったのでメモ
# 準備
今回はざっくりこんなコードがあった場合を考える

```kotlin
class Sample {
    // テストしたいコード
    suspend fun methodWithTimeout(suspendItem: SuspendItem): Int?{
        return withTimeoutOrNull(100L){
            suspendItem.someAsync()
        }
    }
}

class SuspendItem {
    // ランダムにdelayして値を返すような関数を考える。テスト時はこちらをmockすることを考える
    suspend fun someAsync(): Int {
        delay(Random.nextLong(10000L))
        return 10
    }
}
```

# テストを書く

mockには今回`mockito-kotlin`を利用している。
正常系を考える場合は下記のように特に考えずにmockすれば良い

```kotlin
@Test
fun methodWithMock() {
    val s = mock<SuspendItem>{
        onBlocking {
            someAsync()
        } doReturn 10
    }
    runBlocking {
        val result = Sample().methodWithTimeout(s)
        assertEquals(10, result)
    }
}
```

そしてタイムアウトを考える場合、`Thread.sleep`などを考えたくなるがこれはうまくいかない

```kotlin
// 駄目なケース
@Test
fun methodWithMock_invalid() {
    val s = mock<SuspendItem>{
        onBlocking {
            someAsync()
        } doAnswer {
            Thread.sleep(10000)
            10
        }
    }
    runBlocking {
        val result = Sample().methodWithTimeout(s)
        assertEquals(null,result) // 通らない！
    }

}
```

## 解法：TestCoroutineContextを利用する

どうすれば良いのか色々調べると`TestCoroutineContext`を利用すれば良いと判明した

* [test-coroutine-context](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.test/-test-coroutine-context/)
* [mockito-kotlin#311](https://github.com/nhaarman/mockito-kotlin/issues/311#issuecomment-459265712)

こんな感じになる

```kotlin
@Test
fun methodWithMock_withTestCoroutineContext() {
    val context = TestCoroutineContext()
    runBlocking(context) {
        val job = GlobalScope.launch(context) {
            val s = mock<SuspendItem> {
                onBlocking {
                    someAsync()
                } doAnswer {
                    context.advanceTimeBy(10L) // 時間を進める
                    10
                }
            }
            val result = Sample().methodWithTimeout(s)
            assertEquals(10, result)
        }
        job.join()
        assertEquals(true, job.isCompleted)
    }
}
```
```kotlin
@Test
fun methodWithMock_withTestCoroutineContext_timeout() {
    val context = TestCoroutineContext()
    runBlocking(context) {
        val job = GlobalScope.launch(context) {
            val s = mock<SuspendItem> {
                onBlocking {
                    someAsync()
                } doAnswer {
                    context.advanceTimeBy(20000000L) // タイムアウトするぐらい時間を進める
                    10
                }
            }
            val result = Sample().methodWithTimeout(s)
            assertEquals(null, result) // Timeoutしてnullが返ってきた！
        }
        job.join() // joinを忘れるとテストが走らずにpassしてしまうことに注意
        assertEquals(true, job.isCompleted)
    }
}
```

ひとまずこれで十分に動いたが、下記などは理解が浅いので未解決な箇所として残っている。

* `GlobalScope.launch`以外のscopeなどでは上手く動かすことができなかった。
* `runBlocking`に`context`を渡さないとデッドロックが起きる？のかうまく動かなかった

おそらくもう少しシンプルに書く手法がありそうだが、今回は切り上げた。
