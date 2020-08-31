# KeyPath

提案： [Smart KeyPaths: Better Key-Value Coding for Swift](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md)

在 `Objective-C` 和 `Swift` 中， `KVC` 和 `KVO` 通过 `String` 来使用 `key-paths` 。为了使 `key-paths` 更加安全和提供编译器检查， `Swift` 提供了 `#keyPath()` ：

```swift
class Person: NSObject {
  @objc var firstName: String = ""
  @objc var lastName: String = ""
  @objc var friends: [Person] = []
  @objc var bestFriend: Person?

  init(firstName: String, lastName: String) {
    self.firstName = firstName
    self.lastName = lastName
  }
}

let chris = Person(firstName: "Chris", lastName: "Lattner")
let joe = Person(firstName: "Joe", lastName: "Groff")
let douglas = Person(firstName: "Douglas", lastName: "Gregor")
chris.friends = [joe, douglas]
chris.bestFriend = joe

#keyPath(Person.firstName) // => "firstName"
chris.value(forKey: #keyPath(Person.firstName)) // => Chris
#keyPath(Person.bestFriend.lastName) // => "bestFriend.lastName"
chris.value(forKeyPath: #keyPath(Person.bestFriend.lastName)) // => Groff
#keyPath(Person.friends.firstName) // => "friends.firstName"
chris.value(forKeyPath: #keyPath(Person.friends.firstName)) // => ["Joe", "Douglas"]
```

没有字符串，不需要手动检查，编译器会帮助我们完成这一切。

```swift
extension Person {
  class func find(name: String) -> [Person] {
    return DB.execute("SELECT * FROM Person WHERE \(#keyPath(firstName)) LIKE '%s'", name)
  }
}
```

上面的例子中把 `#keyPath(Person.firstName)` 中的 `Person.` 去掉了，可以直接使用 `#keyPath(firstName)` 。

`#keyPath` 只支持 `NSObject` 的子类和使用 `@objc` 进行声明的属性。

提案：[Smart KeyPaths: Better Key-Value Coding for Swift](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md)

`#keyPath()` 缺点：

- 缺少类型信息；
- 不必要的性能损耗
- 只支持 `NSObject` 的子类；

`Swift 4.0` 为我们提供了 `KeyPath` ，为了和属性区分开，使用 `\` 作为前缀，表达式如下：

`\<Type>.<path>` ， `path` 可以是属性，下标，支持可选和链式调用。如果 `Type`  可以从上下文中推断得出，则可以直接使用 `\.<path>` 。

```swift
class Person {
  var name: String
  var friends: [Person] = []
  var bestFriend: Person? = nil
  init(name: String) {
    self.name = name
  }
}

var han = Person(name: "Han Solo")
var luke = Person(name: "Luke Skywalker")
luke.friends.append(han)

// create a key path and use it
let firstFriendsNameKeyPath = \Person.friends[0].name
let firstFriend = luke[keyPath: firstFriendsNameKeyPath] // "Han Solo"

// or equivalently, with type inferred from context
luke[keyPath: \.friends[0].name] // "Han Solo"
// The path must always begin with a dot, even if it starts with a
// subscript component
luke.friends[keyPath: \.[0].name] // "Han Solo"
luke.friends[keyPath: \[Person].[0].name] // "Han Solo"

// rename Luke's first friend
luke[keyPath: firstFriendsNameKeyPath] = "A Disreputable Smuggler"

// optional properties work too
let bestFriendsNameKeyPath = \Person.bestFriend?.name
let bestFriendsName = luke[keyPath: bestFriendsNameKeyPath]  // nil, if he is the last Jedi
```

可以看出是一位星战粉丝。

## 设计细节

`Swift` 为了我们提供了多个不同的 `KeyPaths` 类型。

### AnyKeyPath

未确定的 `Path` / 未确定的 `Root` 类型。 `AnyKeyPath` 是一个全类型擦除的类，支持任何对象/值的任何路径。因为类型擦除的关系，操作有可能失败，所以是可选的：


```swift
class AnyKeyPath: CustomDebugStringConvertible, Hashable {
    // MARK - Composition
    // Returns nil if path.rootType != self.valueType
    func appending(path: AnyKeyPath) -> AnyKeyPath?
    
    // MARK - Runtime Information        
    class var rootType: Any.Type
    class var valueType: Any.Type
    
    static func == (lhs: AnyKeyPath, rhs: AnyKeyPath) -> Bool
    var hashValue: Int
}
```

### PartialKeyPath<Root>

未确定的 `Path` / 已确定的 `Root` 类型， `PartialKeyPath<Root>` ，可以看到返回的也是可选值，因为 `Path` 是不确定的：


```swift
class PartialKeyPath<Root>: AnyKeyPath {
    // MARK - Composition
    // Returns nil if Value != self.valueType
    func appending(path: AnyKeyPath) -> PartialKeyPath<Root>?
    func appending<Value, AppendedValue>(path: KeyPath<Value, AppendedValue>) -> KeyPath<Root, AppendedValue>?
    func appending<Value, AppendedValue>(path: ReferenceKeyPath<Value, AppendedValue>) -> ReferenceKeyPath<Root, AppendedValue>?
}
```

### KeyPath<Root, Value>

已确定的 `Path` / 已确定的 `Root` 类型，因为所有信息都已确定，所以返回的不是可选值：

```swift
class KeyPath<Root, Value>: PartialKeyPath<Root> {
    // MARK - Composition
    func appending<AppendedValue>(path: KeyPath<Value, AppendedValue>) -> KeyPath<Root, AppendedValue>
    func appending<AppendedValue>(path: WritableKeyPath<Value, AppendedValue>) -> Self
    func appending<AppendedValue>(path: ReferenceWritableKeyPath<Value, AppendedValue>) -> ReferenceWritableKeyPath<Root, AppendedValue>
}
```

上面获取到的属性是 `readonly` ，如果需要对属性进行修改，可以使用下面两个类型：

- `WritableKeyPath` ，支持对值类型进行修改
- `ReferenceWritableKeyPath` ，支持对引用类型进行修改

## 应用

[Swift Tip: Bindings with KVO and Key Paths](https://www.objc.io/blog/2018/04/24/bindings-with-kvo-and-keypaths/)

通过 `KVO` 和 `Key Paths` 提供一个轻量级的 `UI` 绑定功能。

```swift
extension NSObjectProtocol where Self: NSObject {
  func observe<Value>(_ keyPath: KeyPath<Self, Value>,
                      onChange: @escaping (Value) -> ()) -> Disposable {
    let observation = observe(keyPath, options: [.initial, .new]) { _, change in
        // The guard is because of https://bugs.swift.org/browse/SR-6066
        guard let newValue = change.newValue else { return }
        onChange(newValue)
    }
    return Disposable { observation.invalidate() }
  }
}
```

监听 `.initial` 和 `.new` ，变化时调用 `onChange` 。跟大多数响应时框架一样，返回一个 `Disposable` ，当 `Disposable` 释放时调用 `observation.invalidate()` 来停止监听。

```swift
extension NSObjectProtocol where Self: NSObject {
  func bind<Value, Target>(_ sourceKeyPath: KeyPath<Self, Value>,
                           to target: Target,
                           at targetKeyPath: ReferenceWritableKeyPath<Target, Value>) -> Disposable {
    return observe(sourceKeyPath) { target[keyPath: targetKeyPath] = $0 }
  }
}
```

当 `sourceKeyPath` 对应的属性产生变化时，会更新 `target` 的 `targetKeyPath` 中的 `value` :

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    disposables = [
        viewModel.bind(\.navigationTitle, to: navigationItem, at: \.title),
        viewModel.bind(\.hasRecording, to: noRecordingLabel, at: \.isHidden),
        viewModel.bind(\.timeLabelText, to: progressLabel, at: \.text),
        // ...
    ]
}
```

[Auto Layout with Key Paths](https://talk.objc.io/episodes/S01E75-auto-layout-with-key-paths)

不借助任何框架写出的 `AutoLayout` 代码如下：

```swift
let view = UIView()
let label = UILabel()
view.addSubview(label)
label.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
    label.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    label.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
    label.trailingAnchor.constraint(equalTo: view.trailingAnchor)
])
```

可以看到我们每次都需要设置 `translatesAutoresizingMaskIntoConstraints` 为 `false` ，调用 `NSLayoutConstraint.activate` ，设置各种 `Anchor` 。

下面来看看如何通过 `KeyPath` 进行改进。

首先定义一个 `typealias` ，接收 `child` 和 `parent` 参数，返回一个 `NSLayoutConstraint` ：

```swift
typealias Constraint = (_ child: UIView, _ parent: UIView) -> NSLayoutConstraint
```

结合 `KeyPath` 定义两个 `equal` 参数：

```swift
func equal<Axis, Anchor>(_ keyPath: KeyPath<UIView, Anchor>, _ to: KeyPath<UIView, Anchor>, constant: CGFloat = 0) -> Constraint where Anchor: NSLayoutAnchor<Axis> {
    return { view, parent in
        view[keyPath: keyPath].constraint(equalTo: parent[keyPath: to], constant: constant)
    }
}
func equal<Axis, Anchor>(_ keyPath: KeyPath<UIView, Anchor>, constant: CGFloat = 0) -> Constraint where Anchor: NSLayoutAnchor<Axis> {
    return equal(keyPath, keyPath, constant: constant)
}
```

然后我们可以通过类似以下方式来定义一个可以返回 `NSLayoutConstraint` 的 `block` ：

```swift
let constraint: Constraint = equal(\.topAnchor, \.safeAreaLayoutGuide.topAnchor)
```

最后把 `translatesAutoresizingMaskIntoConstraints = false` 也封装进来，同时通过 `Constraint` 设置约束：

```swift
extension UIView {
    func addSubview(_ child: UIView, constraints: [Constraint]) {
        addSubview(child)
        child.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate(constraints.map { $0(child, self) })
    }
}

let view = UIView()
let label = UILabel()
view.addSubview(label, constraints: [
   equal(\.leadingAnchor),
   equal(\.topAnchor, \.safeAreaLayoutGuide.topAnchor),
   equal(\.trailingAnchor)
])
```