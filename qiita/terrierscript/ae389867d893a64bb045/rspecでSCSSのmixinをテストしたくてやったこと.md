
scssのテストというと、[bootcamp](https://github.com/thejameskyle/bootcamp)とかみたいにfunctionのテストなんかはよくあるみたい。
が、mixinのテストとなるとなかなかなさそうだった。

なんか良い手はないかなーと思い返すとgruntプラグインのテストなんかであれば変換前の元ソースと処理後のソースを比較するテストとかはよくやるなあと思ったのでそんなアプローチでこんな感じでテストをした。

```ruby
RSpec.describe "style" ,type: :feature do
  def parse_css(url)
    visit url
    parser = CssParser::Parser.new
    parser.add_block! page.source
    parser
  end

  it 'variant' do
    expect = parse_css '/assets/specs/some_mixin_expect.css'
    result = parse_css '/assets/specs/some_mixin_spec.css'
    expect(result.to_s).to include expect.to_s
  end
end
```

- 基本実際のassetsへアクセスすること実現している。
- `/assets/`下にspec用のファイルを置くのはかっこ悪いなあと思いつつもscssのコンパイルオプションの変更とかも検出しやすいので一旦このやり方にしといた。
- 改行による差異とかが出たりしないように`parse_css`というsupport関数を使って一度パースしなおしている。
	- このへんは[bourbon neat](https://github.com/thoughtbot/neat/blob/master/spec/support/parser_support.rb)のテストを参考にした。
	- コメントも除去される。

ちょっと複雑なmixinがほしかったのでやってはみたけど、そんなに気軽にやりたくはない感じかも
