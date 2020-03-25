
swift + cocoapodsでYMCPhysicsDebuggerを入れてみたけどエラーが出てしまった。

が、よく調べるとすでにこれはSpritekitに標準で機能が備わっているっぽい


```swift
class GameViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()

        if let scene = GameScene.unarchiveFromFile("GameScene") as? GameScene {
            
            // Configure the view.
            let skView = self.view as! SKView
            skView.showsFPS = true
            skView.showsNodeCount = true
            
            skView.ignoresSiblingOrder = true
            skView.showsPhysics = true // これ
            scene.scaleMode = .AspectFit
            
            skView.presentScene(scene)
        }
    }

}
```

オブジェクトに青線が出る。
よかった。使えた。
