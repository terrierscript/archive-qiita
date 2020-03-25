
結構ちゃんと調べて知っておくと無駄なクエリ書かなくて済むのがある。

- このあたりのドキュメントが基本
	- https://prestodb.io/docs/current/
	- http://docs.treasuredata.com/articles/presto-udfs



## Case 1. X日前からの日付指定したい時: [TD_TIME_ADD](https://docs.treasuredata.com/articles/presto-udfs#tdtimeadd) ,[TD_TIME_RANGE](https://docs.treasuredata.com/articles/presto-udfs#tdtimerange), [TD_SCHEDULED_TIME](https://docs.treasuredata.com/articles/presto-udfs#tdscheduledtime)

集計する時　X日前までほしい、とかいう要件多いけど、「今日が何日で何日前までで・・・」とかカレンダー見るのめんどい。

下記を覚えとけばいつでも実行タイミングからX日前とかで取得できるので何も考えずにコピペで使いまわせる。

`TD_SCHEDULED_TIME` でクエリの実行タイミングを起点とし、`TD_TIME_ADD`で1日前にして、それを`TD_TIME_RANGE`で区間としている

```sql
SELECT
  :
WHERE 
  -- 1日間のデータなら -1d。一ヶ月なら-30dとか。
  TD_TIME_RANGE(time,
  	TD_TIME_ADD(TD_SCHEDULED_TIME(), '-1d', 'JST'), 
  	null,
  	'JST'
  )
```
覚えづらいのでだいたいどっかからコピペしてくる。

## Case 2. URLのパースしたい(presto): [URL function](https://prestodb.io/docs/current/functions/url.html)

prestoの標準関数
`regexp_like`とかで頑張りたくなるけど`url_extract_***`という関数が用意されている。
超便利。

```sql
SELECT *
FROM access
WHERE 
  url_extract_host(referer) = "google.com"
  -- 正規表現でやるとこんな感じ
  -- regexp_like(referer, '.*google.com/.*')
```

## Case 3. UAからのブラウザ判定したい：[TD_PARSE_AGENT](https://docs.treasuredata.com/articles/presto-udfs#tdparseagent)

ブラウザをUAから簡単に判定してくれる関数。超便利。

```sql
SELECT
  TD_PARSE_AGENT(user_agent)['os_family']
  TD_PARSE_AGENT(user_agent)['os_major']
  TD_PARSE_AGENT(user_agent)['os_minor']
  TD_PARSE_AGENT(user_agent)['ua_family']
  TD_PARSE_AGENT(user_agent)['ua_major']
  TD_PARSE_AGENT(user_agent)['ua_minor']
  TD_PARSE_AGENT(user_agent)['device']
FROM access
```

|os_family|os_major|os_minor|ua_family|ua_major|ua_minor|device|
|---|---|---|---|---|---|---|---|
|Windows 7 | null | null|IE|11 | 0 |Other|
|Android|5|0|Chrome Mobile|47|0|SC-04F|

この関数使うときだいたいIEだけバージョン取りたいとかなるので`WITH`と組み合わせてこんなふうにする

```sql
WITH a AS (
  SELECT 
    TD_PARSE_AGENT(user_agent)['ua_family'] AS family,
    TD_PARSE_AGENT(user_agent)['ua_major'] AS version
  FROM 
    access
),
b AS (
  SELECT
    if(family = 'IE', family || version, family) AS browser
  FROM a
)
SELECT * FROM b
```

## Case 4. ボット判定したい [TD_PARSE_AGENT](https://docs.treasuredata.com/articles/presto-udfs#tdparseagent)(agent)['category']

`TD_PARSE_AGENT`は、ボット判定などにも便利
アクセスログ解析だとBotも厄介な問題。だが`category`を利用すると、これが結構いい感じに解決出来る。

値としては "pc", "smartphone", "mobilephone", "appliance", "crawler", "misc", "unknown"のどれかが返ってくるようなので、これを判定すると良い

```sql
SELECT 
  *
  TD_PARSE_AGENT(user_agent)['ua_family'] AS family,
  TD_PARSE_AGENT(user_agent)['ua_major'] AS version
FROM
  access
WHERE TD_PARSE_AGENT(user_agent) IN ['pc', 'smartphone']
```

## Case 5. X分間のアクセスを同一としてセッション数で取りたい: [TD_SESSIONIZE](https://docs.treasuredata.com/articles/presto-udfs#tdsessionize)

第三引数をキーとして、第二引数で指定した時間以内のアクセスであれば同一のセッションとしてIDを生成してくれる。
滞在時間とか知りたい時良さそう。
TD_SESSIONIZEする値は、ソートされている必要があるので注意（これちゃんと読んでなくて

```sql
SELECT TD_SESSIONIZE(time, 3600, ip_address) as session_id
FROM ...
ORDER BY ip_address, time 
-- ↑ソートしないと正しく出ない！
```

## Case 6. array<T>のunique化(presto)したい : `map_keys`, `histogram`

ハック的。`array_agg`で特定のカラムの値を`array<T>`型にしてくれのだが、同値が大量に並んだ状態になる

```sql
SELECT 
  some_name,
  map_keys(
    histogram(some_type)
  )
FROM foo
GROUP BY some_name
```

## Case 7. 大きいテーブルに対して試しで実行したい(presto) [TABLESAMPLE](https://prestodb.io/docs/current/sql/select.html#tablesample)

でかいテーブルに対して、そのままクエリ投げて遅くて試しづらい時は`TABLESAMPLE`が使える。

```sql
SELECT *
FROM access TABLESAMPLE BERNOULLI(1)
```
入れる値はパーセンテージらしいので、上記なら1%。
