# NSOperation

官方相关文档：

[Operation Queues](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html)

[Operation](https://developer.apple.com/documentation/foundation/operation)

[OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue)

[BlockOperation](https://developer.apple.com/documentation/foundation/blockoperation)

官方文档写的非常详细， `Operation` 和 `OperationQueue` 都写到，强烈推荐。

- `Operation` 支持异步和同步，可以通过 `isAsynchronous` 来判断是否异步，如果添加到 `OperationQueue` 中， `OperationQueue` 会忽略掉这个属性，且默认为异步，可以通过设置 `OperationQueue` 的 `maxConcurrentOperationCount` 为 1 来强制异步。
- 自定义 `Operation` 时记得设置相关状态和 `KVO` 配置，如果 A 操作依赖 B 操作，即使 B 操作取消了， A 操作也会执行，A 操作是否 Ready 是通过 B 操作的 `isFinished` 来判断的，所以可能需要加入额外的判断，判断 B 操作是否成功执行。
- `Operation` 的 `start()` 和 `main()` 的区别。
- 优先级配置： `Operation.QueuePriority` ， `Operation` 不是严格按照优先级高低来执行，如果高优先级的 `Operation` 还没准备好， `OperationQueue` 就会选择去执行低优先级的 `Opeartion` 。如果对顺序有严格要求的话还是要通过依赖来进行配置。
- `Operation` 执行完毕，它就会调用它的 `completionBlock` 。

中文版的详细总结：
[iOS多线程：『NSOperation、NSOperationQueue』详尽总结](https://juejin.im/post/6844903570467192845)

`NSHipster` 出品的简短介绍：
[NSOperation](https://nshipster.com/nsoperation/)