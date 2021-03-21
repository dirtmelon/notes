# 如何获取某个类的全部子类 

[Getting the subclasses of an Objective-C class](https://www.cocoawithlove.com/2010/01/getting-subclasses-of-objective-c-class.html)

获取某个 Objective-C 类的全部子类看起来是个相当简单的任务。虽然 Objective-C runtime 提供了高度的反射和内省，或许应该有个 `class_getSubclasses(Class parentClass)`  方法用于获取全部子类，但是很遗憾，没有。

Objective-C runtime 没有提供这个方法可能有以下几个原因：

- 类可以动态生成和加载；
- 需要考虑线程和锁；
- `class_t` 结构体的历史遗留问题；
- 过早的性能优化，或者说故意引导程序员脱离某些特定的设计。

通常情况下不推荐获取子类，除非是说只有这么个解决办法或者只在调试中使用，否则更推荐使用其它方法，比如说子类主动注册，

[A simple, extensible HTTP server in Cocoa](https://www.cocoawithlove.com/2009/07/simple-extensible-http-server-in-cocoa.html)

```objectivec
+ (void)load
{
    [HTTPResponseHandler registerHandler:self];
}
```

父类会在注册的子类中查找对应的 handler 来处理请求。

但是我们在调试时可以通过获取所有 `HTTPResponseHandler` 的子类来检查是否已经注册。

看下 `class` 在 Objective-C 中的定义：

```objectivec
struct objc_class {
    Class isa;
    Class super_class;
    const char *name;
    long version;
    long info;
    long instance_size;
    struct objc_ivar_list *ivars;
    struct objc_method_list **methodLists;
    struct objc_cache *cache;
    struct objc_protocol_list *protocols;
};
```

`objc_class` 包含指向父类的指针，但是没有包含指向所有子类的指针。如果想通过公开 API 来获取，那么只能获取所有类，然后判断它是否为某个类的子类。这是个非常笨拙的方法，它会遍历 Foundation ， CocoaFramework 和当前项目添加的类。虽然查询这些类只需要一两毫秒，但是如果在循环或者其它频繁调用的操作中进行，这里的耗时就显得非常大，所以可以考虑将结果缓存起来。获取所有类的操作如下：

```objectivec
// 获取所有类的数量
int numClasses = objc_getClassList(NULL, 0);
Class *classes = NULL;

// 分配存储所有类的空间
classes = malloc(sizeof(Class) * numClasses);
// 将类信息存到 classes 中
numClasses = objc_getClassList(classes, numClasses);

// do something with classes

free(classes);
```

那么如何判断是否为某个类的子类呢？一个比较直觉的方法如下：

```objectivec
NSMutableArray *result = [NSMutableArray array];
for (NSInteger i = 0; i < numClasses; i++)
{
    if ([classes[i] isSubclassOfClass:parentClass])
    {
        [result addObject:classes[i]];
    }
}
```

但是并不能这样操作， `isSubclassOfClass:` 是个 `NSObject` 方法，但是不是所有类都继承自 `NSObject` ，比如说 `_NSZombie_` 和 `NSProxy` 。所以只能通过 runtime 的方法来获取，使用 `class_getSuperclass()` 来查找子类的父类，然后进行比较：

```objectivec
NSArray *ClassGetSubclasses(Class parentClass)
{
    int numClasses = objc_getClassList(NULL, 0);
    Class *classes = NULL;

    classes = malloc(sizeof(Class) * numClasses);
    numClasses = objc_getClassList(classes, numClasses);
    
    NSMutableArray *result = [NSMutableArray array];
    for (NSInteger i = 0; i < numClasses; i++)
    {
        Class superClass = classes[i];
        do
        {
            superClass = class_getSuperclass(superClass);
        } while(superClass && superClass != parentClass);
        
        if (superClass == nil)
        {
            continue;
        }
        
        [result addObject:classes[i]];
    }

    free(classes);
    
    return result;
}
```

作者又提供了另外一种更快更 hack 的方法。在 Objective-C runtime 2.0 版本中， `class` 包含了一种直接链接至子类的方式。 `class_t` 结构如下：

```objectivec
typedef struct class_t {
    struct class_t *isa;
    struct class_t *superclass;
    Cache cache;
    IMP *vtable;
    class_rw_t *data;
} class_t;
```

再看看 `class_rw_t` 的结构：

```objectivec
typedef struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
    
    struct method_list_t **methods;
    struct chained_property_list *properties;
    struct protocol_list_t ** protocols;

    struct class_t *firstSubclass;
    struct class_t *nextSiblingClass;
} class_rw_t;
```

这意味着我们可以通过遍历 `firstSubclass` 和 `nextSiblingClass` 来获取所有子类，不需要通过 runtime 来遍历所有类。深度优选遍历：

```objectivec
typedef void *Cache;
#import "objc-runtime-new.h"

void AddSubclassesToArray(Class parentClass, NSMutableArray *subclasses)
{
    struct class_t *internalRep = (struct class_t *)parentClass;
    
    // Traverse depth first
    Class subclass = (Class)internalRep->data->firstSubclass;
    while (subclass)
    {
        [subclasses addObject:subclass];
        AddSubclassesToArray(subclass, subclasses);
    
        // Then traverse breadth-wise
        struct class_t *subclassInternalRep = (struct class_t *)subclass;
        subclass = (Class)subclassInternalRep->data->nextSiblingClass;
    }
}
```

然而这个高效的方法不是线程安全的，根据 

[](https://opensource.apple.com/source/objc4/objc4-437/runtime/objc-runtime-new.m)

的实现，在获取 `class_t` 的 `data` 数据时都需要 `runtimeLock` 来进行加锁，但是 `runtimeLock` 无法在外部获取。因此所有线程（包括 Cocoa 自动开启的线程）都会造成崩溃。

## 总结

1. 虽然说在新的 runtime 中已经有存储子类的数据，但是没有提供相关的 API 来给外部访问；
2. 不是所有的类都继承自 `NSObject` ；
3. 如果 Apple 在新版本的 runtime 中提供子类数据的 API ，效率会更高。