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

使用 Side Table 解决多余的内存占用问题：

`Swift` 中 Side Table 对应的类是 `HeapObjectSideTableEntry` ，普通的 `Swift` 对象不包含 Side Table ，使用的是普通的引用计数方法。 对象的内存布局如下：

```
HeapObject {
  isa
  InlineRefCounts {
    atomic<InlineRefCountBits> {
      strong RC + unowned RC + flags
      OR
      HeapObjectSideTableEntry*
    }
  }
}
```

一般情况下， `atomic<InlineRefCountBits>` 有 `strong` ， `unowned` 计数和 `flags` 组成。如果有 Side Table ， `atomic<InlineRefCountBits>` 就用于指向 `HeapObjectSideTableEntry` 对象，而 `HeapObjectSideTableEntry` 的布局如下：

```
HeapObjectSideTableEntry {
  SideTableRefCounts {
    object pointer
    atomic<SideTableRefCountBits> {
      strong RC + unowned RC + weak RC + flags
    }
  }   
}
```

`SideTableRefCounts` 中包含对象的地址， `weak` 对象指向的是对应的 `HeapObjectSideTableEntry` ， 通过 `SideTableRefCounts` 获取到对应的对象。
以下情况会使用 Side Table ：
1. 创建 `weak` 引用，或者说将来的一些用途；
2. `strong` 或者 `unowned` 计数溢出， `inline` 计数限制在 32 位；
3. 关联对象的存储，目前还没实现；
4. 等等；

[RefCount.h](https://github.com/apple/swift/blob/master/stdlib/public/SwiftShims/RefCount.h)

`Swift` 源码的注释中有详细说明对象在有无 SideTable 下的生命周期。对象的生命周期跟 Side Table 的生命周期是分离的，当强引用计数为 0 而弱引用计数不为了 0 时，只有 `HeapObject` 被释放了，
被引用对象释放了为什么还能直接访问 Side Table？其实 Swift ABI 中 Side Table 的生命周期与对象是分离的，当强引用计数为 0 时，只有 `HeapObject` 被释放了。只有弱引用计数为 0 后， Side Table 才能得以释放：

```c++
void HeapObjectSideTableEntry::decrementWeak() {
  // FIXME: assertions
  // FIXME: optimize barriers
  bool cleanup = refCounts.decrementWeakShouldCleanUp();
  if (!cleanup)
    return;

  // Weak ref count is now zero. Delete the side table entry.
  // FREED -> DEAD
  assert(refCounts.getUnownedCount() == 0);
  delete this;
}
```

虽然 Side Table 只占用很小的内存，如果追求极致的话，可以发发现弱引用释放时手动设置为 `nil` ，释放 Side Table 。
`Swift` 的实现跟 `Objective-C` 的不同，虽然都是使用 Side Table 来维护引用计数，不同的地方就是 `Objective-C` 对象在内存布局中没有 Side Table 指针，而是通过一个全局的外部来维护对象和 Side Table 之间的关系，效率比 Swift 低。另外 `Objective-C runtime` 在对象释放时会将所有的 __weak 变量都指向 `nil` ，减少内存占用，而 `Swift` 会保持 Side Table 。

相关文章： [从源码解析 Swift 弱引用](https://zhuanlan.zhihu.com/p/58179258)

这里有一篇分析 `Swift3` 中 `weak` 实现的文章 [Swift3之Weak引用](https://zhongwuzw.github.io/2017/06/17/Swift%E4%B9%8BWeak%E5%BC%95%E7%94%A8/) ，虽然说当前版本(5.1) 的实现跟之前的不太一样了，但是文章中提供了一个 `UnsafeBufferPointer` 获取指针大小的内存块，转换成 16 进制来查看具体内容的思路。我们可以通过这个操作来查看引用计数的变化和是否含有 Side Table 。

或许过几个版本再来看又会不同的实现了。

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