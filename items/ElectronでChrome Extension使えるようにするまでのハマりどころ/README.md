
そんなに使わないのでいつも流し見していたのだけど、どうしてもReact Dev ToolをElectronに入れたくなって本気出したら詰んだ。

ドキュメントはこのあたり。両方読まないとはまりがち。

- https://github.com/electron/electron/blob/master/docs/tutorial/devtools-extension.md
- https://github.com/electron/electron/blob/master/docs/api/browser-window.md#browserwindowadddevtoolsextensionpath


最終形のjsはこんな具合

```js
const {BrowserWindow} = electron;

const os = require('os')
const path = require('path')
const fs = require('fs')

function createWindow() {
  　:
  　:
  // react dev toolのID
  const id = "fmkadmapgofadopljbjfkapdkoienihi"
  
  // extensionの場所、"~/Library"だとダメだった
  const extensionDir = path.resolve(os.homedir(), "Library/Application Support/Google/Chrome/Default/Extensions/")

  // version指定
  const versions = fs.readdirSync(`${extensionDir}/${id}`).sort()
  const version = versions.pop()

  // 利用
  BrowserWindow.addDevToolsExtension(`${extensionDir}/${id}/${version}`)
}

app.on('ready', createWindow);

```

# ポイントなど
- ドキュメントには書いてることとか
  - Chromeからインストールしておく必要がある。
  - extensionのIDを調べておかないといけない
- ハマりどころとか工夫とか
  - `BrowserWindow.addDevToolsExtension`は`ready`後じゃないと使えない
  - `"~/Library"`のような相対指定だとうまくいかないっぽかった。
    - なので`os.homedir()`と`path.resolve`を使って解決。（直接絶対パスにしてもいい）
  - versionも指定しないといけない。
    - 調べてもいいけど、多分だるいだろうなーと思ったので、`fs.readdirSync`を使うって楽をする。
      - `sort`して`pop`すれば多分最新versionがとれる。はず。
