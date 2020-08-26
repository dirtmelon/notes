# 集合类型
## NSMapTable

`Swift` 中可以直接使用 `NSMapTable` ，用法跟在 `Objective-C` 的区别不大：

```swift
var mapTable: NSMapTable<NSString, AnyObject> = NSMapTable(keyOptions: .copyIn,
                                                           valueOptions: .weakMemory)
autoreleasepool {
  var objectA: NSObject? = NSObject()
  mapTable.setObject(objectA, forKey: "objectA" as NSString)
  print(mapTable)

  objectA = nil
}

print(mapTable)
```

输入如下：

```swift
NSMapTable {
[14] objectA -> <NSObject: 0x6000033fc1f0>
}

NSMapTable {
}
```

注意 `NSMapTable` 是一个 `Objective-C` 类，除了只能使用引用类型之外，`key` 的相等性也不不是通过 `Swift` 的 `Hashable` 协议来确定的。如果您不得不使用 `NSMapTable` ，建议 `key` 不要使用 `NSString` 以外的类型，以防止出现奇怪的 `bug` 。当然了，也可以通过 `NSPointerFunctions` 来自定义相关的判断是否相等的函数。

## NSHashTable

`NSHashTable` 在 `Swift` 中使用也非常方便。在测试的时候发现了一个有趣的点：

```swift
autoreleasepool {
    let stringA: NSString? = "A string" as NSString
    let stringB: NSString? = "B string very long" as NSString
    hashTable.add(stringA)
    hashTable.add(stringB)
    print(hashTable.allObjects)
}

print("after autorelease: \(hashTable.allObjects)")
```

理论上来说在 `autoreleasepool` 后的 `print(hashTable.allObjects)` 应该是一个空数组 `[]` ，然而并不是，输出如下：

```swift
[A string, B string very long]
after autorelease: [A string]
```

猜测因为较短的 `NSString` 使用了 `TaggedPointer` 进行优化，引用计数没有走正常的流程，导致 `NSHashTable` 没有释放掉，搜了一圈没有找到解决办法。

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
1. [Swift Arrays Holding Elements With Weak References](https://marcosantadev.com/swift-arrays-holding-elements-weak-references/)
2. [Swift Tip: Weak Arrays](https://www.objc.io/blog/2017/12/28/weak-arrays/)
3. [Weak Dictionary Values in Swift](https://swiftrocks.com/weak-dictionary-values-in-swift)
