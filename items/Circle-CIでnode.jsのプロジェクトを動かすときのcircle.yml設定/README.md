
忘れがちなのでメモ。

nodeバージョンは適宜最新バージョンにあわせる

```circle.yml
machine:
  node:
    version: 6.3.0 
  post:
    - npm install -g npm@3
  timezone: Asia/Tokyo
```

- nodeのバージョンのデフォルトが`0.10.33`とすごい古いのでバージョンを上げておく（ほんとこれなんとかしてほしい）
- npmもあわせて2系になっているので、3系に上げる
- timezoneは好みによるが、時間絡んだテストしてたりすると変な落ち方したりするのでやっといて損はないかも
