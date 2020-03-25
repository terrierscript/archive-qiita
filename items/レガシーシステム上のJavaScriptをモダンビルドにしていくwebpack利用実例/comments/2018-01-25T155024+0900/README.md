`window.SomeLib = SomeLib`
移行方式、使わせていただきました:thumbsup_tone1:

ちょうど、もっとレガシーな
`<input type="button" value="Check" onclick="SomeLib.hoge()">`
のような使われ方をしているJSが、肥大してメンテ不能になってきたので、webpackで、
最小の手間で、ie11でも動くことを維持しながら、minifyとモジュール化しようとしていたので
