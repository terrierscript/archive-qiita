
CSSとHTMLでやるこういうタブの実装
![JS_Bin.png](https://qiita-image-store.s3.amazonaws.com/0/7307/c2a04885-7240-ad5f-c527-01056fab6657.png "JS_Bin.png")

選択されているものを`.active`などのクラスをつけて色を変えるような実装についての思いつき。

### いつもやる感じのやつ
普通にBootstrapを習ったやり方をするとこんな具合

```html
<ul id="list1">
    <li>AAAA</li>
    <li class="active">BBBB</li>
    <li>CCCC</li>
</ul>
```

```scss
#list1{
  li{
    cursor: pointer;
    display: inline-block;
    border: 1px gray solid;
    border-bottom: 0px;
    padding: 10px;
  }
  .active{
    background: #ccb;
  }
}
```

これをサーバーサイドで何らかやろうとすると、だいたいこんな感じになると思う。今回はjadeを使って再現

```jade
- var activeTab = "bbbb"
- var tabs = ["aaaa", "bbbb", "cccc"]
ul#list1
  each tab,i in tabs
    li(class=(tab == activeTab ? "active" : "" ))= tab        
```
だいたいこんな感じになる。


### 思いついたやつ
もしかしてこれ、scssの`@each`とかある今ならもうちょっといい感じに出来たりするんじゃね？って思って考えてみた

```html
<ul id="list2" class="active-bbbb"> <!-- 親で指定する -->
    <li class="tab-aaaa">AAAA</li>
    <li class="tab-bbbb">BBBB</li>
    <li class="tab-cccc">CCCC</li>
</ul>
```
```scss
#list1{
  li{
    cursor: pointer;
    display: inline-block;
    border: 1px gray solid;
    border-bottom: 0px;
    padding: 10px;
  }
  $tabs: aaaa, bbbb, cccc; // 利用するタブを並べる
  @each $tab in $tabs{
    &.active-#{$tab}{
      .tab-#{$tab}{ // 親と一致したらactive化する
        background: #ccb; 
      }
    }
  }
}
```

CSS化するとこんな感じ

```css
#list1 li {
  cursor: pointer;
  display: inline-block;
  border: 1px gray solid;
  border-bottom: 0px;
  padding: 10px;
}

#list1.active-aaaa .tab-aaaa {
  background: #ccb;
}

#list1.active-bbbb .tab-bbbb {
  background: #ccb;
}

#list1.active-cccc .tab-cccc {
  background: #ccb;
}
```

テンプレ的にはこんな感じ

```jade
- var activeTab = "bbbb"
- var tabs = ["aaaa", "bbbb", "cccc"]
ul#list2(class="active-" + activeTab)    
  each tab,i in tabs
    li(class="tab-" + tab)= tab
```

`<li>`でのロジックが消えてくれた。
今回の簡易な例だとあんまり恩恵が見えないが、`<li>`の中身が複雑で別途パーシャル化しないといけないとかの時には使えるかもしれない
