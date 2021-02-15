# initialize
[Apple Developer Documentation](https://developer.apple.com/documentation/objectivec/nsobject/1418639-initialize)

1. `initialize` 会在第一次给当前类发送消息（即调用方法）时调用；
2. 先调用父类的，再调用子类的；
3. `initialize` 是线程安全的，它会在第一次给类发送消息的当前线程中运行，而其它线程尝试给类发送消息的线程则需要等待 `initialize` 执行完毕；
4. 如果子类没有实现 `initialize` 方法，则会调用父类的，所以一个 `initialize` 有可能会多次调用，我们可以通过对当前类进行判断来防止多次调用；
5. 因为 `initialize` 有阻塞机制，所以尽量不要执行复杂的初始化方法，不然有可能会造成死锁；
6. 每个类的 `initialize` 方法只会调用一次，如果需要分类和类的初始化方法都执行，可以使用 `load` 方法。

如何防止 `initialize` 方法多次调用：

```objectivec
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

[draveness/analyze](https://github.com/draveness/analyze/blob/master/contents/objc/%E6%87%92%E6%83%B0%E7%9A%84%20initialize%20%E6%96%B9%E6%B3%95.md)

`initialize` 的源码解析，与 `load` 不同，`initialize` 方法调用时，所有的类都**已经加载**到了内存中。

## 如何在 Swift 中使用 `initialize`

[Handling the Deprecation of initialize()](http://jordansmith.io/handling-the-deprecation-of-initialize/)

`load` 和 `initialize` 方法在 Swift 中都不会调用，所以需要一个替代的方案，在 Swift 中也可以起到 `load` 或者 `initialize` 的作用。

一个简单的替代方案：

直接在 `delegate` 的 `application(_:didFinishLaunchingWithOptions:)` 的方法中调用对应的方法，但是这样会有不少缺点：

- 可能有大量的类需要处理，这会使得 `delegate` 变得笨重，因为它直接依赖了这些类，即使说把这部分的方法调用挪至单独的功能模块中，这个模块也是直接依赖这些类；
- 可能说没有权限来获取 `delegate` ，在只是负责开发其中一小部分或者只是一个 SDK 时会有这种情况发生。

一个不简单的替代方案：

这个方案和 `load` 或者 `initialize` 方法类似，不需要主动调用，也不会影响 `delegate` 。

首先定义以下类和协议：

```swift
protocol SelfAware: class {
    static func awake()
}

class NothingToSeeHere {

    static func harmlessFunction() {

        let typeCount = Int(objc_getClassList(nil, 0))
        let types = UnsafeMutablePointer<AnyClass?>.allocate(capacity: typeCount)
        let safeTypes = AutoreleasingUnsafeMutablePointer<AnyClass?>(types)
        objc_getClassList(safeTypes, Int32(typeCount))
        for index in 0 ..< typeCount { (types[index] as? SelfAware.Type)?.awake() }
        types.deallocate(capacity: typeCount)

    }

}
```

可以看到 `harmlessFunction` 方法通过 `objc_getClassList` 来获取所有的类，如果类支持 `SelfAware` 的 `awake` 方法，那么就会进行调用，接下来需要无侵入地调用 `harmlessFunction` 方法：

```swift
extension UIApplication {

    private static let runOnce: Void = {
        NothingToSeeHere.harmlessFunction()
    }()

    override open var next: UIResponder? {
        // Called before applicationDidFinishLaunching
        UIApplication.runOnce
        return super.next
    }

} 
```

但是这里有个不好的地方，就是需要通过 `objc_getClassList` 来获取所有的类，也就是只能是 `Objective-C` 的类，如果是纯 Swift 的类，是不支持的。