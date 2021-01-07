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