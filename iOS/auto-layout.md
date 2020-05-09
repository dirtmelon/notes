# AutoLayout

## 使用 NSLayoutAnchor 进行布局

```Swift
let view = UIView()
view.translatesAutoresizingMaskIntoConstraints = false
superview.addSubview(view)
NSLayoutConstraint.activate([
  view.leadingAnchor.constraint(equalTo: superview.leadingAnchor.leadingAnchor, constant: 16),
  view.trailingAnchor.constraint(equalTo: superview.trailingAnchor.trailingAnchor, constant: -16),
  view.centerYAnchor.constraint(equalTo: superview.centerYAnchor.centerYAnchor),
  view.heightAnchor.constraint(equalToConstant: 100)
])
```

## UITableView/UICollectionView 使用 AutoLayout 计算高度产生布局冲突

假设布局代码如下：
```swift
NSLayoutConstraint.activate([
  imageView.heightAnchor.constraint(equalToConstant: 80.0),
  imageView.widthAnchor.constraint(equalToConstant: 80.0),
  imageView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16.0),
  imageView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -16.0),
  imageView.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 16.0),
  label.centerYAnchor.constraint(equalTo: imageView.centerYAnchor),
  label.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16.0),
  label.leadingAnchor.constraint(equalTo: imageView.trailingAnchor, constant: 8.0)
])
```

看起来没什么问题，跑起来界面布局也正常，但是控制台输出如下警告：
```shell
Unable to simultaneously satisfy constraints.
    Probably at least one of the constraints in the following list is one you don't want. 
    Try this: 
        (1) look at each constraint and try to figure out which you don't expect; 
        (2) find the code that added the unwanted constraint or constraints and fix it. 
(
    "<NSLayoutConstraint:0x60000069acb0 UITableViewCellContentView:0x7f8794e11f10.height == 112   (active)>",
    "<NSLayoutConstraint:0x60000069b4d0 'UIView-Encapsulated-Layout-Height' UITableViewCellContentView:0x7f8794e11f10.height == 112.333   (active)>"
)

Will attempt to recover by breaking constraint 
<NSLayoutConstraint:0x60000069acb0 UITableViewCellContentView:0x7f8794e11f10.height == 112   (active)>

Make a symbolic breakpoint at UIViewAlertForUnsatisfiableConstraints to catch this in the debugger.
The methods in the UIConstraintBasedLayoutDebugging category on UIView listed in <UIKitCore/UIView.h> may also be helpful.
```

可以看到这两个约束产生了冲突：
```
"<NSLayoutConstraint:0x60000069acb0 UITableViewCellContentView:0x7f8794e11f10.height == 112   (active)>",
"<NSLayoutConstraint:0x60000069b4d0 'UIView-Encapsulated-Layout-Height' UITableViewCellContentView:0x7f8794e11f10.height == 112.333   (active)>"
```

`NSLayoutConstraint:0x60000069acb0` 是我们设置的，`NSLayoutConstraint:0x60000069b4d0` 是系统在通过 AutoLayout 计算出 Cell 高度之后添加的，而且会比我们设置的高度多出 0.333 。虽然不管也不会有什么问题，反正界面也正常，如果想要去掉警告，可以设置 `topAnchor` 或者 `bottomAnchor` 对应的 `NSLayoutConstraint` 的 `priority` 为 `UILayoutPriority.defaultLow` 即可：
```swift
bottomConstraint.priority = UILayoutPriority.defaultLow
```

这样系统可以优先采用它自己添加的约束。

