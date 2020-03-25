思い出した。
私もnode.jsでjohnny-fiveモジュールを使った時、
$ nohup node app.js &
とかでバックグラウンドで動かそうとしたらエラーメッセージが出て動かなかった。。
Johnny-fiveの開発者の掲示板で質問を投げて、replを無効にしたらいい、と助言を受けて何とか解決しました。
https://github.com/rwaldron/johnny-five/issues/538
