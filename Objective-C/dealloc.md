# dealloc
[Clang 12 documentation](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#dealloc)

> A class may provide a method definition for an instance method named dealloc. This method will be called after the final release of the object but before it is deallocated or any of its instance variables are destroyed. The superclass’s implementation of dealloc will be called automatically when the method returns.

`dealloc` 在最后 `release` 时调用，但此时实例变量（ `Ivars` ）并未释放，父类的  `dealloc` 会在子类的 `dealloc` 返回后调用。

> The instance variables for an ARC-compiled class will be destroyed at some point after control enters the dealloc method for the root class of the class. The ordering of the destruction of instance variables is unspecified, both within a single class and between subclasses and superclasses.

实例变量会在 `root class` （根类）的 `dealloc` 中释放，一般来说就是 `NSObject` 的 `dealloc` 方法，释放顺序不确定。

[聊聊dealloc](https://zhongwuzw.github.io/2017/09/21/%E8%81%8A%E8%81%8Adealloc/)

### dealloc 调用时机

当对象调用 `release` 方法时会走到 `sidetable_release` 这个方法中，而 `sidetable_release` 这个方法会判断是否需要调用 `dealloc` 方法：

```objectivec
uintptr_t
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    // 找到当前对象所对应的 SideTable
    SideTable& table = SideTables()[this];
    bool do_dealloc = false;
    table.lock();
    // 找到当前对象所对应的引用计数
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) {
        // 如果找不到所对应的应用计数，则表示可以执行 dealloc ，
			  // 同时设置对应的值为 SIDE_TABLE_DEALLOCATING
        do_dealloc = true;
        table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
    } else if (it->second < SIDE_TABLE_DEALLOCATING) {
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        // 如果引用计数小于 SIDE_TABLE_DEALLOCATING ，则表示引用计数为 0 ，可以执行 dealloc
        do_dealloc = true;
        it->second |= SIDE_TABLE_DEALLOCATING;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
        // 引用计数减 1
        it->second -= SIDE_TABLE_RC_ONE;
    }
    table.unlock();
    // 进行释放操作，执行 dealloc
    if (do_dealloc  &&  performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return do_dealloc;
}
```

`dealloc` 有可能在任何线程调用，在最后一个调用 `release` 方法的线程中调用。

函数调用顺序： `dealloc->_objc_rootDealloc->objc_object::rootDealloc->object_dispose->objc_destructInstance` ：

```objectivec
- (void)dealloc {
    _objc_rootDealloc(self);
}
```

```objectivec
void
_objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}
```

```objectivec
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?
		// 判断 isa 的各个标志位，确认是否需要进行快速释放。
    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

```objectivec
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}
```

```objectivec
void *objc_destructInstance(id obj)
{
    if (obj) {
        Class isa_gen = _object_getClass(obj);
        class_t *isa = newcls(isa_gen);

        // Read all of the flags at once for performance.
        bool cxx = hasCxxStructors(isa);
        bool assoc = !UseGC && _class_instancesHaveAssociatedObjects(isa_gen);

        // This order is important.
        if (cxx) object_cxxDestruct(obj); // 1
        if (assoc) _object_remove_assocations(obj); // 2

        if (!UseGC) objc_clear_deallocating(obj); // 3
    }

    return obj;
}
```

1. `object_cxxDestruct` 调用 C++ 析构器，释放实例变量；
2. `_object_remove_assocations` 清除 `Associated` 对象；
3. `objc_clear_deallocating` ARC 相关操作，清理 `SideTable` ， `weak` 设置为 `nil` 等；

    [ARC下dealloc过程及.cxx_destruct的探究](http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)

    这篇文章对 `.cxx_destruct` 做了深入研究，包括 `.cxx_destruct` 如何释放实例变量，如何调用 `[super dealloc]` 。这两者都是由编译器帮我们完成，插入这部分的代码。