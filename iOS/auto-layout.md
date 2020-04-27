# Auto Layout

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

