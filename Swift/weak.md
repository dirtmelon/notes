# Weak

[Friday Q&A 2017-09-22: Swift 4 Weak References](https://www.mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)

对应的中文版本：[Swift 4 弱引用实现](https://swift.gg/2018/08/02/swift-4-weak-references/)

Mike Ash 写的关于 `Swift` 中弱引用实现原理的文章。
分析了应该如何存储这些信息的两种方案：
1. 使用外部表，如 `Objective-C` 对象的关联对象和 `weak` 实现。当你需要操作关联对象时，运行时机制会使用内存地址去一个大的哈希表中查找它。为了实现多线程安全，该表在操作时会加锁，所以存在一定程度访问速度问题。引用计数的保存位置，则取决于具体操作系统版本和 CPU 架构，它有时位于对象内存中，而有时又存储在外部表中。
2. 放在对象内存中，在 `Swift` 旧有实现中，类信息，引用计数和存储属性全部内联在对象内存中，而 `weak` 信息则存储在单独的外部表中。

方案一将数据存储在外部数据表中可以节省空间，因为不是每个对象都需要提供关联对象和 `weak` 记录；
方案二可以提高访问速度，但是会占用更多的内存空间；

`Objective-C` 旧有的解决方案会将引用计数存储在外部表中，这样不需要在对象内存中腾出 4 个字节（32位）空间来存储引用计数，大多数情况下引用计数都是1，在外部表中可以通过默认缺省值来表示1，减少内存占用。在设备性能不够好的远古时代，这是一个非常好的解决方案。

对象中需要存储的信息：
1. 对象的类信息，每次动态消息派发都需要使用到对象的类信息，类信息应该直接保存在对象内存中；
2. 属性，毫无疑问，存储在对象内存中；
3. 引用计数，现在设备性能足够好，可以牺牲空间还时间，所以可以直接保存在对象内存中；

使用 `SideTable` 解决多余的内存占用问题：



## 应用

在 [weak Objective-C](../Objective-C/weak.md) 中有提到 `weak singleton` 的实现， `Swift` 的版本如下：

```swift
class SharedManager {
    private static weak var weakShared: SharedManager?
    
    static var shared: SharedManager {
        get {
            if let shared = weakShared {
                return shared
            } else {
                let shared = SharedManager()
                weakShared = shared
                return shared
            }
        }
    }
    let name: String = "I am SharedManager"
    
    init() {
        print("init")
    }
    
    deinit {
        print("deinit")
    }
}
```

测试代码如下：

```swift
var shared1: SharedManager? = SharedManager.shared
var shared2: SharedManager? = SharedManager.shared

shared1 = nil
shared2 = nil

var shared3: SharedManager? = SharedManager.shared
shared3 = nil
```

控制台输出如下：

```
before shared1 and shared2 init
init
deinit
after shared1 and shared2 deinit
before shared3 init
init
deinit
before shared3 deinit
```

符合预期。