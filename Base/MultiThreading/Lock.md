# Lock
## 自旋锁

当线程等待自旋锁时不会进入睡眠，自旋锁由于在获取锁时，线程会一直处于忙等状态，有可能会造成任务的优先级反转。

### OSSpinLock

> 新版 iOS 中，系统维护了 5 个不同的线程优先级/QoS: background，utility，default，user-initiated，user-interactive。高优先级线程始终会在低优先级线程前执行，一个线程不会受到比它更低优先级线程的干扰。这种线程调度算法会产生潜在的优先级反转问题，从而破坏了 spin lock。

> 具体来说，如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。这并不只是理论上的问题，libobjc 已经遇到了很多次这个问题了，于是苹果的工程师停用了 OSSpinLock。

[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

## 互斥锁

当等待互斥锁时，线程会进入睡眠，锁释放时就会唤醒线程。互斥锁又分为递归锁和非递归锁：

- 递归锁：可重入，同一个线程在锁释放前可再次获取锁，可以递归调用；
- 非递归锁：不可重入，必须等锁释放后才能再次获取。

### pthread_mutex

`pthread_mutex` 是互斥锁，对性能要求比较高的场景可以使用， API 比较简单：

```objectivec
// 导入头文件
#import <pthread.h>
// 全局声明互斥锁
pthread_mutex_t _lock;
// 初始化互斥锁
pthread_mutex_init(&_lock, NULL);
// 加锁
pthread_mutex_lock(&_lock);
// 这里做需要线程安全操作

// 解锁 
pthread_mutex_unlock(&_lock);
// 释放锁
pthread_mutex_destroy(&_lock);
```

### @synchronized

[mikeash.com: Friday Q&A 2015-02-20: Let's Build @synchronized](https://www.mikeash.com/pyblog/friday-qa-2015-02-20-lets-build-synchronized.html)

性能比较低，因为有容错处理，和使用全局表。

1. 不能使用`非OC对象`作为加锁条件——`id2data`中接收参数为id类型
2. 多次锁同一个对象会有什么后果吗——会从高速缓存中拿到data，所以只会锁一次对象
3. 都说@synchronized性能低——是因为在底层`增删改查`消耗了大量性能
4. 加锁对象不能为nil，否则加锁无效，不能保证线程安全

[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

### NSLock

`NSLock` 对 `pthread_mutex` 进行了一层封装，提供了 `Objective-C` 层级的 API ：

```objectivec
NSLock *lock = [[NSLock alloc] init];
[lock lock];
[lock unlock];
```

从 Swift 开源版的 Foundation 中可以看到 NSLock 是基于 `pthread_mutex` 的封装：

[apple/swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation/blob/main/Sources/Foundation/NSLock.swift)

删除其它平台的代码后大概实现如下：

```swift
open class NSLock: NSObject, NSLocking {
		public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }

		open func lock() {
        pthread_mutex_lock(mutex)
    }

    open func unlock() {
        pthread_mutex_unlock(mutex)
        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }
}
extension NSLock {
    // 同步执行 closure 操作
    internal func synchronized<T>(_ closure: () -> T) -> T {
        self.lock()
        defer { self.unlock() }
        return closure()
    }
}
```

而 `NSLock` 非递归锁，在递归调用时会堵塞。如果需要递归调用可以通过 `NSRecursiveLock` 加锁， `NSRecursiveLock` 实现与 `NSLock` 类似，只是在初始化时设置 `mutex` 为 `RECURSIVE` 类型：

```swift
open class NSRecursiveLock: NSObject, NSLocking {
    internal var mutex = _RecursiveMutexPointer.allocate(capacity: 1)
    private var timeoutCond = _ConditionVariablePointer.allocate(capacity: 1)
    private var timeoutMutex = _MutexPointer.allocate(capacity: 1)

    public override init() {
        super.init()
        var attrib = pthread_mutexattr_t()
        withUnsafeMutablePointer(to: &attrib) { attrs in
            pthread_mutexattr_init(attrs)
						// 设置为 RECURSIVE
            pthread_mutexattr_settype(attrs, Int32(PTHREAD_MUTEX_RECURSIVE))
            pthread_mutex_init(mutex, attrs)
        }
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }
    
    open func lock() {
        pthread_mutex_lock(mutex)
    }
    
    open func unlock() {
        pthread_mutex_unlock(mutex)
        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }
}
```

而递归锁同时对同一个对象使用锁时也会产生死锁：

### NSCondition

[Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nscondition)

`NSCondition` 对象在给定线程中充当锁和检查点 （ checkpoint ）。锁在检测条件和执行由条件触发的任务时保护你的代码。检查点则要求线程在执行其任务之前条件为 `true` 。当条件为 `false` 时，线程会阻塞，直到另一个线程向条件对象发出信号。伪代码：

```bash
lock the condition
while (!(boolean_predicate)) {
    wait on condition
}
do protected work
(optionally, signal or broadcast the condition again or change a predicate value)
unlock the condition
```

与信号量类似， Swift 源码中也有 `NSCondition` 的实现：

```swift
open class NSCondition: NSObject, NSLocking {
    internal var mutex = _MutexPointer.allocate(capacity: 1)
    internal var cond = _ConditionVariablePointer.allocate(capacity: 1)

    public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(cond, nil)
    }
    
    deinit {
        pthread_mutex_destroy(mutex)
        pthread_cond_destroy(cond)
        mutex.deinitialize(count: 1)
        cond.deinitialize(count: 1)
        mutex.deallocate()
        cond.deallocate()
    }
    
    open func lock() {
        pthread_mutex_lock(mutex)
    }
    
    open func unlock() {
        pthread_mutex_unlock(mutex)
    }
    
    open func wait() {
        pthread_cond_wait(cond, mutex)
    }

    open func wait(until limit: Date) -> Bool {
        guard var timeout = timeSpecFrom(date: limit) else {
            return false
        }
        return pthread_cond_timedwait(cond, mutex, &timeout) == 0
    }
    
    open func signal() {
        pthread_cond_signal(cond)
    }
    
    open func broadcast() {
        pthread_cond_broadcast(cond)
    }
    
    open var name: String?
}
```

用法如下：

```swift
let cond = NSCondition()
var available = false
var SharedString = ""

class WriterThread : Thread {
    
    override func main(){
        for _ in 0..<5 {
            cond.lock()
            SharedString = "😅"
            available = true
            cond.signal() // 通知并且唤醒等待的线程
            cond.unlock()
        }
    }
}

class PrinterThread : Thread {
    
    override func main(){
        for _ in 0..<5 { // 循环 5 次
            cond.lock()
            while(!available){   // 通过伪信号进行保护
                cond.wait()
            }
            print(SharedString)
            SharedString = ""
            available = false
            cond.unlock()
        }
    }
}

let writet = WriterThread()
let printt = PrinterThread()
printt.start()
writet.start()
```

### NSConditionLock

[Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nsconditionlock)

`NSConditionLock` 与 `NSCondition` 不同，自带支持复杂的条件锁，比如说：消费者-提供者场景。 `lock(whenCondition:)` 在条件成立时可以获取到锁，或者等待另外一个线程调用 `unlock(withCondition:)` 释放锁和设置对应的值。

```swift
let NO_DATA = 1
let GOT_DATA = 2
let clock = NSConditionLock(condition: NO_DATA)
var SharedInt = 0

class ProducerThread : Thread {
    
    override func main(){
        for i in 0..<5 {
            clock.lock(whenCondition: NO_DATA) //当条件为 NO_DATA 获取该锁
              // 如果不想等待消费者，直接调用 clock.lock() 即可
            SharedInt = i
            clock.unlock(withCondition: GOT_DATA) //解锁并设置条件为 GOT_DATA
        }
    }
}

class ConsumerThread : Thread {
    
    override func main(){
        for i in 0..<5 {
            clock.lock(whenCondition: GOT_DATA) // 当条件为 GOT_DATA 获取该锁
            print(i)
            clock.unlock(withCondition: NO_DATA) //解锁并设置条件为 NO_DATA
        }
    }
}

let pt = ProducerThread()
let ct = ConsumerThread()
ct.start()
pt.start()
```

### dispatch_semaphore

[Apple Developer Documentation](https://developer.apple.com/documentation/dispatch/dispatchsemaphore)

`distpatch_semaphore` 在 Swift 上已替换为 `DispatchSemaphore` ，相应的 API 也有说改变。

信号量，可以作为同步锁使用，也可以控制 GCD 的最大并发数：

```swift
// 创建信号量
let semaphore = DispatchSemaphore(value:0)
// 信号量 +1 ，得到的信号量大于 0 后会去执行 wait 后面的代码
semaphore.signal()
// 信号量 -1 ，当信号量小于 0 时会阻塞线程
semaphore.wait()
```

### os_unfair_lock

`os_unfair_lock` 是苹果在 iOS 10/macOS 10.12 上提供的，用于替换 `OSSpinLock` 。

[Apple Developer Documentation](https://developer.apple.com/documentation/os/1646466-os_unfair_lock_lock)

## 性能测试

[Updated for Xcode 8, Swift 3; added os_unfair_lock](https://gist.github.com/steipete/36350a8a60693d440954b95ea6cbbafc)

这里有个锁的性能测试，如果只需要支持到 iOS10 ，那么使用 `os_unfair_lock_s` 是最好的选择， `OSSpinLock` 性能最好，但是有优先级反转的问题：

如果需要支持 iOS10 以下那么信号量或者 `pthread_mutex` 在性能上表现最好。如果需要支持 `Linux` 平台可以选择 `pthread_mutex` 。

其余的 `NSLock` ， `DispatchQueue` 和 `@Syncronized` 性能都较差，因为苹果在里面做了不少容错处理和进行一些全局记录（ `@Syncronized` ）

## 其它文章

Friday Q&A 这篇讲得特别好，各个锁都有详细说明：[mikeash.com: Friday Q&A 2017-10-27: Locks, Thread Safety, and Swift: 2017 Edition](https://www.mikeash.com/pyblog/friday-qa-2017-10-27-locks-thread-safety-and-swift-2017-edition.html)

各个锁的大概实现：[bestswifter/blog](https://github.com/bestswifter/blog/blob/master/articles/ios-lock.md)

[如何理解互斥锁、条件锁、读写锁以及自旋锁？](https://www.zhihu.com/question/66733477/answer/1267625567?utm_source=wechat_session&utm_medium=social&utm_oi=52825543409664&from=groupmessage&isappinstalled=0)

[iOS探索 细数iOS中的那些锁](https://juejin.im/post/5ec9f3ec6fb9a047f0125fd2)