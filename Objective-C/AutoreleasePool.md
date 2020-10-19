# AutoreleasePool
[draveness/analyze](https://github.com/draveness/analyze/blob/master/contents/objc/%E8%87%AA%E5%8A%A8%E9%87%8A%E6%94%BE%E6%B1%A0%E7%9A%84%E5%89%8D%E4%B8%96%E4%BB%8A%E7%94%9F.md)

[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

整个 iOS 的入口都是放到 `@autoreleasepool` 的 `block` 中：

```objectivec
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

然后编译器会将其改写成下面的代码：

```objectivec
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```

我们也可以通过手动调用 `@autoreleasepool` 来创建自己的自动释放池。

RunLoop 每次处理事件时也会创建和释放 `autoreleasepool` 。App 启动后，会在主线程的 RunLoop 里注册两个 `autoreleasepool` 相关的 Observer ，其回调的方法都是 `_wrapRunLoopWithAutoreleasePoolHandler()` 。

1. 第一个 Observer 监听的事件是 Entry ，即将进入 Loop ，会调用 `_objc_autoreleasePoolPush()` 来创建自动释放池，order 是 -2147483647 ，优先级最高，确保自动释放池的创建在其它回调之前；
2. 第二个 Observer 监听了 Exit 事件，当推出 Loop 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池，order 是 2147483647 ，优先级最低，确保自动释放池的释放在所有回调之后。

```objectivec
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

`AutoreleasePage` 的定义如下：

```objectivec
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

自动释放池是由一系列的 `AutoreleasePoolPage` 组成，每个 `AutoreleasePoolPage` 的大小都是 4096 bit 大小。

- 自动释放池是由 `AutoreleasePoolPage` 以双向链表的方式实现的
- 当对象调用 `autorelease` 方法时，会将对象加入 `AutoreleasePoolPage` 的栈中
- 调用 `AutoreleasePoolPage::pop` 方法会向栈中的对象发送 `release` 消息

当使用容器的 `block` 枚举时，内部会自动添加一个 `AutoreleasePool` ：

```objectivec
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
```

但是普通的 for 循环和 for in 循环中是没有的，所以当遍历中的 `autorelease` 变量所占用的内存较大时，需要手动添加 `@autoreleasepool` 。

[@autoreleasepool uses in 2019 Swift](https://swiftrocks.com/autoreleasepool-in-2019-swift.html)

本文先是简单的介绍了 `autoreleasepool` 在 Objective-C 中的使用场景——在循环体中大量创建 `autorelease` 对象。而 ARC 对 Swift 的优化在过去几年中进步了很多，根据作者的测试，似乎 ARC for Swift 从不调用 `autorelease` ，而是用多次调用 `release` 来替代。所以对于纯粹的 Swift 对象我们可能不再需要 `autoreleasepool` 。但在 Swift 开发中 `autoreleasepool` 仍然有用，因为在 UIKit 和 Foundation 中仍然存在调用 `autorelease` 的遗留 `Objective-C` 类。在 Swift 5.2 上测试确实如此。

[Swifter Tips:autoreleasepool](https://swifter.tips/autoreleasepool/)
上文也有提到这个情况：
其实对于这个特定的例子，我们并不一定需要加入自动释放。在 Swift 中更提倡的是用初始化方法而不是用像上面那样的类方法来生成对象，而且从 Swift 1.1 开始，因为加入了可以[返回 `nil` 的初始化方法](https://swifter.tips/init-nil/)，像上面例子中那样的工厂方法都已经从 API 中删除了。今后我们都应该这样写：

```swift
let data = Data(contentsOfFile: path)
```

使用初始化方法的话，我们就不需要面临自动释放的问题了，每次在超过作用域后，自动内存管理都将为我们处理好内存相关的事情。