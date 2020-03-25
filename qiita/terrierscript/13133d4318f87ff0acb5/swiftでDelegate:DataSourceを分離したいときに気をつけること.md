
UITableViewのDataSourceやDelegateを分離しようとしてうまくいかなかった話。

## DelegateになるにはNSObjectを継承している必要がある
UIKitのDelegateはそれぞれNSObjectProtocol継承であるっぽい。
なので

```swift
class MyTableViewDataSource : NSObject, UITableViewDataSource{

}
```

といった具合で継承されてないとうまくいかない

## Delegateはメンバ変数化しておく

これは完全にswiftから入ったがゆえかなあと思った。

例えばさくっとこんな感じで書いたとすると

```swift
class TableViewController: UITableViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        self.tableView.dataSource = TableViewDataSource()
    }

    override func numberOfSectionsInTableView(tableView: UITableView) -> Int {
        return 1
    }
	
	// DataSourceの件数そのまま出すといいだろうーみたいな感覚
    override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return  (self.tableView.dataSource as TableViewDataSource).data.count
    }
}
```
みたいなことをしていると`EXC_BAD_ACCESS`に見舞われる。
一体何なんだこれはと思っていたがおそらくdataSourceの参照先が開放されてしまっているのだろう。

なので下記のようにViewControllerのメンバーにしておく必要がある模様。

```swift
class TableViewController: UITableViewController {
    var dataSource : TableViewDataSource = TableViewDataSource()
    override func viewDidLoad() {
        super.viewDidLoad()
        self.tableView.dataSource = dataSource
    }

    override func numberOfSectionsInTableView(tableView: UITableView) -> Int {
        return 1
    }

    override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return dataSource.data.count
    }
}
```
