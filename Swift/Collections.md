# 集合类型
## Weak Array
### NSPointerArray
Swift 中也支持调用 `NSPointerArray` ，但是需要通过 `Unmanaged` 做一些指针的转换操作：

```swift
let array = NSPointerArray.weakObjects()
let obj: MyClass = MyClass()
let pointer = Unmanaged.passUnretained(obj).toOpaque()
array.addPointer(pointer)
```

可以通过 `extension` 给 `NSPointerArray` 添加一些快捷方法：

```swift
extension NSPointerArray {
  
  func addObject(_ object: AnyObject?) {
    guard let object = object else { return }
    addPointer(Unmanaged.passUnretained(object).toOpaque())
  }
  
  func insertObject(_ object: AnyObject?, at index: Int) {
    guard let object = object, index < count else { return }
    insertPointer(Unmanaged.passUnretained(object).toOpaque(), at: index)
  }
  
  func replaceObject(at index: Int, withObject object: AnyObject?) {
    guard let object = object, index < count else { return }
    replacePointer(at: index, withPointer: Unmanaged.passUnretained(object).toOpaque())
  }
  
  func object(at index: Int) -> AnyObject? {
    guard index < count,
      let pointer = pointer(at: index) else { return nil }
    return Unmanaged<AnyObject>.fromOpaque(pointer).takeUnretainedValue()
  }
  
  func removeObject(at index: Int) {
    guard index < count else { return }
    removePointer(at: index)
  }
}
```

上面这种写法不是类型安全的，无法确定 `NSPointerArray` 里面对应的类型是什么，每次使用时都需要进行类型转换。

### Weak Box
使用 `WeakBox` 把 `value` 封装起来，然后在 `WeakArray` 里面将普通的 `element` 转换为 `WeakBox` ：

```swift
class WeakBox<A: AnyObject> {
  weak var unbox: A?
  init(_ value: A) {
    unbox = value
  }
}

struct WeakArray<Element: AnyObject> {
  private var elements: [WeakBox<Element>] = []
  
  init(_ elements: [Element]) {
    self.elements = elements.map { WeakBox($0) }
  }
}

extension WeakArray: Collection {
  var startIndex: Int { return elements.startIndex }
  var endIndex: Int { return elements.endIndex }
  
  subscript(_ index: Int) -> Element? {
    return elements[index].unbox
  }
  
  func index(after idx: Int) -> Int {
    return elements.index(after: idx)
  }
}
```

```swift
let weakArray = WeakArray([UIView(), UIView()])
let firstElement = weakArray.filter { $0 != nil }.first // firstElement is nil
```

### 相关链接
[Swift Arrays Holding Elements With Weak References](https://marcosantadev.com/swift-arrays-holding-elements-weak-references/)
[Swift Tip: Weak Arrays](https://www.objc.io/blog/2017/12/28/weak-arrays/)
