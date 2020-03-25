結論： `npm pack`と`tar`コマンドを組み合わせる。

残念ながら今のところ`npm publish --dry-run`オプションとかは無い。
（出典：https://github.com/npm/npm/issues/6351#issuecomment-119051586）

```
$ tar -tf $(npm pack)
package/package.json
package/README.md
package/index.js
```

npm packすると、publishの代わりとしてtarで固められるので、その出力を`tar -tf`に通すと一覧が出てくるという寸法。頭いい。

yarnでも`yarn pack`はあるが、出力がうまくいかないのでそのまま使えなそう

---

npm publishのhelpを見ると、一応下記のような記述までは存在してる。

`$ npm help publish`
> For a "dry run" that does everything except actually publishing to  the
   registry,  see  npm  help  npm-pack,  which figures out the files to be
   included and packs them into a tarball to be uploaded to the  registry.



