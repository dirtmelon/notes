# GCD
官方文档:
[Dispatch](https://developer.apple.com/documentation/dispatch)
源代码：
[libdispatch](https://opensource.apple.com/source/libdispatch/)

GCD 详细介绍：
[iOS多线程：『GCD』详尽总结](https://juejin.im/post/5a90de68f265da4e9b592b40#heading-27)
[细说GCD（Grand Central Dispatch）如何用](https://github.com/ming1016/study/wiki/%E7%BB%86%E8%AF%B4GCD%EF%BC%88Grand-Central-Dispatch%EF%BC%89%E5%A6%82%E4%BD%95%E7%94%A8)

使用 GCD 的 `DispatchSource` 构造一个 `Timer`
[Repeat](https://github.com/malcommac/Repeat)

Mikeash 自己实现的一个简易版的 Dispatch 多线程任务管理队列 ： [Friday Q&A 2015-09-04: Let's Build dispatch_queue](https://www.mikeash.com/pyblog/friday-qa-2015-09-04-lets-build-dispatch_queue.html)

## dispatch_once 实现原理
[Friday Q&A 2014-06-06: Secrets of dispatch_once
](https://www.notion.so/GCD-a0f552b226b44f3fb8f6cdbc7a8d73fe#0073163c4b1a4adeaae5a374007625fe)

```objc
static dispatch_once_t predicate;    
dispatch_once(&predicate, ^{
		// some one-time task
});
```
`dispatch_once`  只需要提供两个参数：

- `predicate` ，一个 `token` ，用于保证执行一次；
- `block` ，需要执行的具体操作；

在单线程中， 我们使用一个 `if` 就可以保证方法只执行一次。但是在多线程中，就需要通过 `dispatch_once` 来保证 `block` 只执行一次，且其它线程需要等待 `dispatch_once` 执行完成。自己实现对应的版本并不难，但是 `dispatch_once` 的速度极快，这点比较难实现。

单线程版本：
```objc
void SimpleOnce(dispatch_once_t *predicate, dispatch_block_t block) {
		if(!*predicate) {
		    block();
        *predicate = 1;
    }
}
```

`dispatch_once_t` 只是一个 `long` 的 `typedef` ，初始化为 `0` ， `block` 执行完毕后设置为 `1` 以保证不会多次执行。但是在多线程时，可能会同时进入到 `if` 条件判断中，导致多次执行。

关于 `dispatch_once` 的性能部分，有下面三个场景需要考虑清楚：

1. 首次调用 `dispatch_once` 的调用者会直接执行 `block` ；
2. 在首次调用到 `block` 完成执行之间调用，需要等待 `block` 完成；
3. 在 `block` 完成后调用，无需等待，可以直接继续后续流程；

1 和 2 都不是非常重要，1 只会出现一次，而 2 基本上很少出现。

最重要的是第3点，在程序中会有可能出现成千上万次，每次我们都需要保证 `dispatch_once` 只执行一次。可以使用 `SimpleOnce` 作为我们性能测试的黄金准则。

### Locks
```objc
void LockedOnce(dispatch_once_t *predicate, dispatch_block_t block) {
		static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

    pthread_mutex_lock(&mutex);
    if(!*predicate) {
		    block();
        *predicate = 1;
    }
    pthread_mutex_unlock(&mutex);
}
```
简易版的 `Lock` 实现，由于 `predicate` 是一个 `long` 指针，无法存放 `Lock` ，所以新建了一个全局 `mutex` 来保证线程安全，这样会导致不相关的 `predicate` 也需要互相等待，但是对于试验性的代码来说够用了。
### Spinlocks
```objc
void SpinlockOnce(dispatch_once_t *predicate, dispatch_block_t block) {
		static OSSpinLock lock = OS_SPINLOCK_INIT;

    OSSpinLockLock(&lock);
		if(!*predicate) {
				block();
        *predicate = 1;
		}
    OSSpinLockUnlock(&lock);
}
```
`Spinlocks` 会让线程忙等，而不是休眠，以此来减少唤醒线程的耗时。相比 `mutex` 版本有相当大的改进，但是比起单线程版本耗时还是较长。
### Atomic Operations
```objc
BOOL CompareAndSwap(long *ptr, long testValue, long newValue) {
		if(*ptr == testValue) {
		    *ptr = newValue;
        return YES;
    }
    return NO;
}
```
原子操作，提供 CPU 操作，不需要进行加锁操作。 

`ptr` 有三个值：

- 0 表示 `block` 从未执行
- 1 表示 `block` 执行中
- 2 表示 `block` 执行中

尽早退出，如果 `*predicate` 为 2 就 `return` ：
```objc
void EarlyBailoutAtomicBuiltinsOnce(dispatch_once_t *predicate, dispatch_block_t block) {
		if(*predicate == 2) {
		    __sync_synchronize();
        return;
    }

    volatile dispatch_once_t *volatilePredicate = predicate;

    if(__sync_bool_compare_and_swap(volatilePredicate, 0, 1)) {
		    block();
	      __sync_synchronize();
        *volatilePredicate = 2;
    } else {
        while(*volatilePredicate != 2)
			  ;
        __sync_synchronize();
    }
}
```
源码：[once.c](https://opensource.apple.com/source/libdispatch/libdispatch-913.60.2/src/once.c)

`dispatch_once` 的死锁分析: http://satanwoo.github.io/2016/04/11/dispatch-once/

延伸阅读：
[【整理】__builtin_expect 解惑](https://my.oschina.net/moooofly/blog/175019)
[什么是内存屏障(Memory Barriers)](http://lday.me/2017/11/04/0016_what_is_memory_barriers/)

## GCD 源码分析
结合源码分析用法和原理，非常详尽。

[深入浅出GCD之基础篇 | cocoa_chen](http://cocoa-chen.github.io/2018/03/01/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8B%E5%9F%BA%E7%A1%80%E7%AF%87/)

[深入浅出GCD之dispatch_queue | cocoa_chen](http://cocoa-chen.github.io/2018/03/05/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8Bdispatch_queue/)

[深入浅出GCD之dispatch_semaphore | cocoa_chen](http://cocoa-chen.github.io/2018/03/08/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8Bdispatch_semaphore/)

[深入浅出GCD之dispatch_queue | cocoa_chen](http://cocoa-chen.github.io/2018/03/05/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8Bdispatch_queue/)

[深入浅出GCD之dispatch_once | cocoa_chen](http://cocoa-chen.github.io/2018/03/15/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8Bdispatch_once/)

[深入浅出GCD之dispatch_source | cocoa_chen](http://cocoa-chen.github.io/2018/03/19/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAGCD%E4%B9%8Bdispatch_source/)

## GCD 注意点
[libdispatch efficiency tips](https://gist.github.com/tclementdev/6af616354912b0347cdf6db159c37057)

- 只使用非常少，明确定义的 `queues` 。 所有的 `queues` 激活后，就会使用很多线程。 `queues` 应该根据 App 中特定的环境进行定义：UI ，存储，后台工作等，以此从多线程中获利；
- 先使用主线程，当你发现性能瓶颈时，找到原因，如果多线程可以优化性能，必须要小心地应用，同时观察系统的压力。重复使用默认的 `queues` ，如果需要添加多一个 `queue` 必须要经过测量。在大多数 Apps 中，尽量不要使用超过 3 个或者 4 个 `queues` ；
- Queues that target other (non-global) queues are fine, these are the ones which scale. （这段不太明白）；
- 不要使用 `dispatch_get_global_queue()` ，它不能很好地处理优先级，同时会导致线程爆炸。使用自己的特定 `queue` 是最好的选择；
- 如果 `dispatch` 对应的 `block` 小于 1ms ，使用 `dispatch_async()` 会造成性能上的浪费，因为 `libdispatch` 的过载行为，很有可能会创建一个新的线程来执行这个 `block` 。使用锁来保护共享状态会是一个更好的选择；
- 一些类/库被更好地设计为复用其调用者的执行上下文，这意味这它们使用传统的锁来保证线程安全。 `os_unfair_lock` 通常是系统中的最快的锁（优先级更高，更少的上下文切换）；
- 如果并行运行，那么你的 `work item` 不应该相互竞争（竞态），否则性能会急剧下降。竞态有多种形式，锁是其中一种，这意味着使用共享资源有可能成为性能瓶颈：IPC/系统服务， `malloc(lock)` ， 共享内存， I/O ， ...
- 你不需要为了避免线程爆炸而一直使用同步方法。使用一定数量的 `queue` 而不是 `dispatch_get_global_queue()` 会是一个更好的选择；
- 异步编程的 bug 和复杂度都会增加，同步编程更容易阅读，编写和维护；
- 串行队列比并行队列优化得更好。只有在你需要性能改善时才使用并行队列，否则有可能是过早优化；
- 如果你需要在同一个队列中混合异步和同步调用，请使用 `dispatch_async` 和 `wait` 而不是 `dispatch_sync()` 。 `dispatch_async` 和 `wait` 结合使用可以减少队列切换；
- 充分利用3-4个以上的内核不是件容易的事，大多数尝试着么做的人都是在浪费精力来获得微不足道的性能；
- 测量 App 的真实性能，以此确保 App 通过优化后变得更快，而不是更慢。进行性能测试时应该进行全局的性能测试，而不是局部的性能测试，避免缓存影响和保持线程池活跃；
- `libdispatch` 非常有效率但是并不是魔术，资源是有限的。你无法忽略掉你正在使用的底层系统和硬件。不是所有代码可以并行运行；

检查你代码所有 `dispatch_async()` 的调用，看看它们需要执行的任务是否值得切换至不同的上下文来执行。大多数情况下，锁都是更好的选择。

一旦你开始使用定义的队列和复用它们，你有可能在调用 `dispatch_sync()` 时导致死锁，在队列用于线程安全时经常会出现这种情况，再次声明一下使用锁是一个比较好的解决方案，只有在需要切换至不同的上下文时才使用 `dispatch_async()` 。

## 如何取消 GCD 任务
如果还未执行的子线程可以用 `dispatch_block_cancel` 来取消，需要使用用 `dispatch_block_create` 创建 `dispatch_block_t` 。

```objc
- (void)stopSync {
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        NSLog(@"block1 begin");
        [NSThread sleepForTimeInterval:3];
        NSLog(@"block1 end");
    });

    dispatch_block_t block2 = dispatch_block_create(0, ^{
        NSLog(@"block2 ");
    });
    dispatch_async(queue, block1);
    dispatch_async(queue, block2);
    //取消执行block2
    dispatch_block_cancel(block2);
}
```

对于执行中的任务，可以通过变量判断是否需要提前 `return` 来取消任务。线程外设置`__block` 变量，配合线程中 `return` 结束:

```objc
- (void)stopAsync {
    __block BOOL isFinish =NO;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        for(long i=0; i<10000; i++) {
            NSLog(@"执行第 %ld 次",i);
            sleep(1);
            if(isFinish ==YES) {
                NSLog(@"停止");
                return;
            }
        };
    });
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW,(int64_t)(10 * NSEC_PER_SEC)),dispatch_get_main_queue(), ^{
        NSLog(@"停止任务");
        isFinish =YES;
    });
}
```

## GCD 造成卡顿
[iOS App 使用 GCD 导致的卡顿问题](https://zhuanlan.zhihu.com/p/37463055)
1. iOS 系统本身是一个资源调度和分配系统，CPU ，disk IO ，VM 等都是稀缺资源，各个资源之间会互相影响，主线程的卡顿看似 CPU 资源出现瓶颈，但也有可能内核忙于调度其他资源，比如当前正在发生大量的磁盘读写，或者大量的内存申请和清理，都会导致下面这个简单的创建线程的内核调用出现卡顿：

    `libsystem_kernel.dylib __workq_kernreturn`

    所以解决办法只能是自己分析各 `thread` 的 `call stack` ，根据用户场景分析当前正在消耗的系统资源。后面也确实通过最近提交的代码分析，发现是由于增加了一些非常耗时的磁盘 io 任务（虽然也是放在在子线程），才出现这个看着不怎么沾边的 `call stack` 。revert 之后卡顿警报就消失了。
2. 现有的卡顿检测工具都只能在超时的情况下 `dump call stack` ，但出现超时有可能是任务 A，B，C 共同作用导致的，A 和 B 可能是真正耗时的任务，C 不耗时但碰巧是最后一个，所以被当成元凶，而 A 和 B 却没有出现在上报日志里。我暂时也没有想到特别好的解决办法。很明显，`libsystem_kernel.dylib __workq_kernreturn` 就是一个不怎么耗时的 C 任务。
3. 在使用 GCD 创建 queue，或者说一个 App 内部使用 GCD 执行子线程任务时，最好有一套 App 所有团队都能遵循的队列使用机制，避免创建过多的 `thread` ，而出现意料之外的线程资源紧缺，代码无法及时执行的情况。这很难，尤其是在大公司动则上百人的团队里面。

### GCD 原理详解

[bestswifter/blog](https://github.com/bestswifter/blog/blob/master/articles/objc-gcd.md)

## 如何取消 `dispatch_after` 的 Block

[Canceling Blocks in GCD](https://mattrajca.com/2016/04/23/canceling-blocks-in-gcd.html)