# KVO 
## 基础

官方文档：

[Introduction to Key-Value Observing Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)

`automaticallyNotifiesObserversForKey:` 默认返回 `YES` ，当重写并对某个 `Key` 返回 `NO` 时，那么修改属性时就需要手动调用 `(void)willChangeValueForKey:(NSString *)key` 与 `-(void)didChangeValueForKey:(NSString *)key` 发送通知，我们也可以通过这样在 `Setter` 方法判断对象是否真的发生改变，只有真的发生改变时才发送通知。

[iOS 中的 KVO](https://kingcos.me/posts/2019/kvo_in_ios/)

这篇文章非常详细，从 `KVO` 的使用到原理都进行了说明。

[ObjC 中国 - KVC 和 KVO](https://objccn.io/issue-7-3/)

一个需要注意的地方是，KVO 行为是同步的，并且发生与所观察的值发生变化的同样的线程上。没有队列或者 Run-loop 的处理。

[objcio/issue-7-lab-color-space-explorer](https://github.com/objcio/issue-7-lab-color-space-explorer/blob/master/Lab%20Color%20Space%20Explorer/KeyValueObserver.m)

[mikeash.com: Friday Q&A 2009-01-23](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)

Mikeash 关于 KVO 原理的文章：

- 动态生成一个 `KVO` 的子类，实现了 `dealloc` ， `_isKVOA` ， `class` 方法；
- 只会生成一个 `KVO` 子类，对所有监听的属性的设置方法都进行了替换，如果针对不同的属性监听生成不同类，就需要动态生成大量的不同的类，所以苹果选择了只生成一个类；
- 替换了对应的方法的 `IMP` ，改用内部的 `NSSet...ValueAndNotify` ；

[刨根问底KVO原理](https://juejin.im/post/5c22023df265da6124157a25)

通过源码相关的伪代码来探究 `KVO` 的实现方式，如果需要深入了解 `KVO` 的原理，可以阅读下这篇文章。 `KVO` 的原理看起来虽然比较简单，但是实现时还是有不少坑，比如说多线程，系统的具体实现也体现了这一点，通过 `pthread_mutex_lock` 来保证线程安全。

[draveness/analyze/KVOController](https://github.com/draveness/analyze/blob/master/contents/KVOController/KVOController.md)

[facebookarchive/KVOController](https://github.com/facebookarchive/KVOController)

为了解决 `KVO` 非常难用的问题，Facebook 开源了 `KVOController` ，优点如下：

1. 不需要手动移除 `observer` ，这里利用了关联属性在对象释放时也会被释放的原理，在关联属性的 `dealloc` 方法中移除 `observer` ；
2. 支持使用 `block` ，减少复杂度，添加监听和处理通知的代码可以放在同一处；

[一种基于KVO的页面加载，渲染耗时监控方法](http://satanwoo.github.io/2017/11/27/KVO-Swizzle/)

在做 `ViewController` 的耗时检测时，我们需要记录各个 `UIViewController` 子类对应方法的耗时，如果只是针对 `UIViewController` 的方法进行 `hook` ，那么只能记录到 `UIViewController` 的方法耗时，无法获取子类的方法耗时。

在进行 `KVO` 时 `runtime` 实际上会帮你创建一个 `KVO` 相关的子类，由此可以在初始化时进行一次 `KVO` 来生成一个新的子类，然后对这个子类方法进行耗时检测。

至于为什么使用 `KVO` 的方式，下面这篇文章有进行解释，而且也给出了具体实现代码：

[巧妙利用KVO实现精准的VC耗时检测](https://punmy.cn/2018/06/18/15278496835424.html)

[panmingyang2009/VCProfiler](https://github.com/panmingyang2009/VCProfiler)

[KVO在不同的二进制中多个符号并存的Crash问题](http://satanwoo.github.io/2017/09/11/KVO-CRASH/)

结论：

当两个产物都有相同的类名时，这两个类都会被 realize ，都能够被正常调用。但是由于全局类表的存在，在动态创建 `KVO` 的子类时，只能产生一个。所以就导致 `allocate` 失败，从而引发` register` 过程的 Crash 问题。