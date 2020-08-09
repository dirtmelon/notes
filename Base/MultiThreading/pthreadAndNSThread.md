# pthread 和 NSThread
## 远离线程
[Migrating Away from Threads
](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html)
苹果官方推荐使用 `GCD` 和 `NSOperationQueu` 来代替线程，好处如下：

- 减少存储线程栈帧的内存占用；
- 清除创建和配置线程的代码；
- 清除在线程中管理和安排任务的代码；
- 简化所需要编写的代码；

### 替换线程

在 App 中使用线程的方式：

- 执行单次任务的线程。创建一个线程执行一个任务，然后在任务完成后释放它；
- 工作者线程。创建一个或者多个线程来执行一些特定的任务。定期分配任务给每个线程；
- 线程池。创建一个包含线程的线程池，为每个线程起一个 `RunLoop` 。当你需要执行一个任务时，从线程池中提取一个线程来执行。如果没有空闲的线程，就加入到任务队列中，等到有线程可用时执行；

替换方案：

- 对于单任务线程，把任务封装进 `block` 或者 `Operation` 对象中，然后提交到并发队列；
- 对于工作者线程，你需要决定使用串行队列还是并发队列。如果你使用工作者线程来同步一些特定任务的执行，那么使用串行队列。如果你使用工作者线程来执行任意一个任务，没有相互依赖，可以使用并发队列；
- 对于线程池，可以把你的任务封装到 `block` 或者 `Operation` 对象，然后把它们分配到指定的并发队列中执行 ；

对于任务间的共享资源，使用队列仍然有不少好处。它可以提供一种更加可预测的方式来执行你的代码。这意味着可以不通过锁或者其它重量级的装置来同步代码的执行顺序。你可以使用队列来执行相同的任务。

## 清除基于锁的代码

相比于锁，队列更加高效。即是不实在竞争状态下，也需要消耗一定的性能来获取锁。在竞争状态下，无法确定等待锁释放需要多长时间。

结合之前 `GCD` 部分提到的如何高效使用 `GCD` ，如果是那种耗时较小的任务，那么直接使用锁是个更好的选择，因为线程的上下文切换也是需要消耗时间的，如果耗时较长的任务，建议使用 `GCD` 。  

使用 `dispatch_async` 和串行队列可以建立异步的锁，在对应的 `block` 中执行任务，由于任务先进先出的关系，任务会按照调用的顺序执行，而且不会阻塞调用者的线程。

使用 `dispatch_sync` 执行同步任务，只有当需要等待任务执行完毕时才调用 `dispatch_sync` 时才调用。

## 改进循环代码

如果代码中的每次循环对于其它循环来说都是独立的，那么可以考虑使用 `dispatch_apply` 或者 `dispatch_apply_f` 。它们会把循环中每次迭代独立提交到队列中。当与并发队列结合时，就可以同时执行多个迭代。必须要保证每次迭代都是可重入的。

```objectivec
queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n", i);
});
```

当然，也可以通过在一个 `dispatch_apply` `block` 里执行多个迭代来提高效率。你需要确定 `dispatch_apply` 的步长。当原来的迭代数字非常大时，通过设置步长可以减少派发 `block` 的次数，这意味着需要更多时间来执行 `block` ，而不是派发 `block` 。你需要通过性能测试来确定具体的步长。

### 替换 Thread Joins

Thread Joins 允许你创建一个或多个线程，然后在当前线程中等待其它线程完成任务。如果父线程需要创建多个子线程来完成任务，可以通过这个方法来实现。

`Dispatch groups` 提供了相同意义上的机制，同时还有其它优点。它可以阻塞当前线程以等待子任务完成。可以同时执行多个子任务 。由于使用 `Dispatch queues` 的关系，效率非常高。

也可以通过 `Operation` 对象间的依赖关系来实现上面的需求。

[iOS多线程：『pthread、NSThread』详尽总结](https://juejin.im/post/5a66c9b751882573520d8abc)

[iOS 多线程技术实践之 pthreads（一）](https://kingcos.me/posts/2019/multithreading_techs_in_ios-1/)
