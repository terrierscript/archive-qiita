
割りと最近だとエディタ部分がShadow DOMなのでスタイルが効かないことがある模様

```style.less
::shadow{
  .line {
    .trailing-whitespace{
      background-color: #F92672;  
      color: white;
    }
  }
}
```

こんな具合で`::shadow`でくくってやると無事色がつく


##### 参考
https://github.com/atom/atom/blob/master/docs/upgrading/upgrading-your-ui-theme.md
