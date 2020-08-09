# 多线程
## 官方文档
[Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html)

中文版本：

[Threading Programming Guide(1)](http://yulingtianxia.com/blog/2017/08/28/Threading-Programming-Guide-1/)

[Threading Programming Guide(2)](http://yulingtianxia.com/blog/2017/09/17/Threading-Programming-Guide-2/)

[Threading Programming Guide(3)](http://yulingtianxia.com/blog/2017/10/08/Threading-Programming-Guide-3/)

## 官方的并发编程指南
[Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html)

### 远离线程
#### 为什么要使用并发编程
1. 充分利用计算机的多核；
2. 更好的用户体验；

#### 为什么避免使用线程（`NSThread`）
1. 使用 `NSThread` 你需要自己管理线程池；
2. 需要根据系统的状态调整线程的数量；
3. 需要保持线程的高效运行，避免互相干扰；

#### Dispatch Queues
Dispatch Queues 是一种基于 C 的机制，可执行自定义任务，支持串行和并行，任务按照 FIFO 顺序执行。
优点：
- 提供直观简单的接口；
- 提供自动和全面的线程池管理；
- 提供了优化后的速度；
- 更高效率的内存占用（线程栈帧不需要常驻内存）；
- 不需要与内核交互；
- 分发到异步队列的任务不会造成队列死锁；
- 优雅的扩展；
- 串行队列提供了一个比锁或者其它原始的同步方案更高效的替代品；

#### Dispatch Sources
Dispatch Sources 是一种基于 C 的机制，用于异步处理特定类型的系统。 Dispatch Source 会包含有关特定类型系统事件的信息，在事件发生时将特定的 block 提交给 Dispatch Queue 。你可以使用 Dispatch Sources 来监听以下几种类型的系统事件：

- Timer 定时器；
- Signal 监听UNIX信号；
- Descriptor-related events 监听文件和 Socket 相关操作；
- Process-related events 监听进程相关状态；
- Mach port events 监听 Mach 相关事件；
- Custom events that you trigger 监听自定义事件；

Dispatch sources 是 GCD 的一部分。

#### Operation Queues
Operation Queue 是 Cocoa 提供的，和并行的 Dispatch Queue 是相同概念的东西，具体类型为 `NSOperationQueue` 类。跟 Dispatch Queue 保持 FIFO 的执行顺序不同， Operation Queue 支持自定义的执行顺序。你可以在定义任务时设置相关的依赖，以此来创造一个复杂的执行顺序图。

Operation Queue 中任务对应的类型为 `NSOperation` 类。`NSOperation` 封装了你需要执行的任务和所有相关的数据。 `NSOperation` 是一个抽象的基类，你可以自定义一些子类来执行自己的任务。系统也提供了一些特定的子类。

`NSOperation` 提供了 KVO 通知，可用于监听任务的进度。Operation Queue 通常会并发执行任务，你可以通过设置依赖来保证它们按照所计划的顺序执行。

### Asynchronous Design Techniques
异步编程会增加你代码的复杂度，编写和 debug 代码都会变得更加困难。所以在重构代码以支持并发编程前，需要充分考虑清楚。如果错误地使用异步编程，有可能导致代码运行得比之前更慢。
每个应用都有着不同的任务需要执行，所以不可能编写一个文档来准确地告诉你如何设计你的应用和它的相关任务，但是可以提供一些相关的指导让你在设计时做出对的选择。

#### Define Your Application’s Expected Behavior
当你想要给自己的应用添加并发代码时，你应该先定义好应用的预期行为，以此来验证接入并发编程后的行为是否正确以及测试性能收益。

定义相关任务和数据结构。理清各个任务间的依赖关系。

#### Factor Out Executable Units of Work
找出任务的最小执行单元，将其封装进 `block` 或者 `NSOperation` ，然后派发到适当的队列中。不用担心任务分得太细而影响性能。队列会帮你处理好这一切，当然了，最好还是通过性能测试来调整任务大小。

#### Identify the Queues You Need

确定好队列的属性。
当使用 GCD 时，如果需要特定的执行顺序，使用串行队列，如果不需要特定的执行队列，可使用并发队列或者多个队列。
当使用 `NSOperation` 时，可通过设置各个 `NSOperation` 之间的依赖来调整它们之间的执行顺序。

#### Tips for Improving Efficiency
- 如果你的应用已经受内存限制，那么现在直接使用计算值可能比从主内存加载缓存的值要快。 计算值直接使用处理器核心的寄存器和缓存，这比主内存快得多。 当然了，还是需要性能测试来确定是否有利于性能优化；
- 找出串行任务，尽力把它们变得更加并发。 如果由于某个任务依赖某些共享资源而必须串行执行该任务，请考虑更改架构以删除该共享资源。 可以考虑为每个需要共享资源的用户拷贝一份对应的资源，或者完全清除共享资源；
- 避免使用锁。Dispatch queue 和 Operation Queue 在大多数情况下都不需要使用锁。避免使用锁来保护共享资源，使用串行队列或者设置 NSOperation 间的依赖来确保以正确的顺序执行任务；
- 尽量使用系统框架。实现功能时优先考虑现有的系统 API 是否可以满足需求；

## 具体用法
[NSOperation](NSOperationn.md)

[GCD](GCD.md)
[pthread 和 NSThread](pthreadAndNSThread.md)

## 相关文章

### ObjC 专题

[并发编程：API 及挑战](https://objccn.io/issue-2-1/)

[底层并发 API](https://objccn.io/issue-2-3/)

[线程安全类的设计](https://objccn.io/issue-2-4/)

[测试并发程序](https://objccn.io/issue-2-5/)
