# +load
## 原理
官方文档：

[Apple Developer Documentation](https://developer.apple.com/documentation/objectivec/nsobject/1418815-load?language=objc)

The order of initialization is as follows:

1. All initializers in any framework you link to.
2. All `+load` methods in your image.
3. All C++ static initializers and C/C++ `__attribute__(constructor)` functions in your image.
4. All initializers in frameworks that link to you.

In addition:

- A class’s `+load` method is called after all of its superclasses’ `+load` methods.
- A category `+load` method is called after the class’s own `+load` method.

运行时机： Objective-C 运行时会收集所有 `+load` 方法的类，然后在镜像加载完成后调用，时机在主函数运行前。

初始化顺序：

1. 执行全部链接到的框架中的所有构造器；
2. 镜像（ Image ) 中所有的 `+load` 方法；
3. 镜像 （ Image ）中所有 C++ 静态构造器，以及 C/C++ 的 `__attribute__(constructor)` 方法；
4. 执行全部链接到当前框架的全部框架的所有构造器；

特点：

1. 类的 `+load` 方法会在父类的 `+load` 方法调用后再调用；
2. 分类 `Category` 的 `+load` 方法会在类的 `+load` 方法后调用；
3. Swift 中桥接到 Objective-C 的类不会自动调用 `+load` 方法；

[draveness/analyze](https://github.com/draveness/analyze/blob/master/contents/objc/%E4%BD%A0%E7%9C%9F%E7%9A%84%E4%BA%86%E8%A7%A3%20load%20%E6%96%B9%E6%B3%95%E4%B9%88%EF%BC%9F.md)

详细说明了 `+load` 方法的调用时机：

```c
0  +[XXObject load]
1  call_class_loads()
2  call_load_methods
3  load_images
4  dyld::notifySingle(dyld_image_states, ImageLoader const*)
11 _dyld_start
```

在有新的镜像加载后，都会调用 `load_images` 方法进行回调，这个方法是运行时在 `_objc_init` 方法中进行注册的：

```objectivec
dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
```

类的 `+load` 方法会在父类的 `+load` 方法调用后再调用：

```objectivec
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    // 类是否已经 realized
    assert(cls->isRealized());
		// 判断类是否有调用过 +load
    if (cls->data()->flags & RW_LOADED) return;
		// 递归调用，先执行父类的 +load 方法
    schedule_class_load(cls->superclass);
		// 添加当前类至列表
    add_class_to_loadable_list(cls);
    // 设置为已调用过 +load
    cls->setInfo(RW_LOADED); 
}
```

分类的 `+load` 方法在类之后调用：

```objectivec
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // 加载中，直接返回
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 调用类的 +load 方法，直到列表为空
        while (loadable_classes_used > 0) {
            // ➡️ 调用类的 +load 方法
            call_class_loads();
        }

        // 调用分类的 +load 方法一次
        more_categories = call_category_loads();

        // 如果有类或者分类未调用 +load 方法，则尝试再调用一次
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

调用分类 `+load` 方法时需要确保类已经加载：

```objectivec
if (cls  &&  cls->isLoadable()) {
    (*load_method)(cls, SEL_load);
    cats[i].cat = nil;
}
```

这篇文章的解析非常详细，有需要可以读一下：

[iOS 中的 +load 方法](https://kingcos.me/posts/2019/+load_in_ios/)

`+load` 方法的执行时机非常靠前，而且只会执行一次，所以一般来说我们可能会通过 `+load` 方法来执行一些 `hook` 操作，但是如果 `+load` 方法过多或者方法执行时间较长，就会影响增加应用的启动时间，所以在编写 `+load` 方法时需要非常小心。

## 监控 +load 方法的耗时
这篇文章讲述了如何监控 `+load` 方法的耗时：

[计算 +load 方法的耗时](https://triplecc.github.io/2019/05/27/%E8%AE%A1%E7%AE%97load%E8%80%97%E6%97%B6/)

实现有以下这几点需要注意：

dyld 加载的镜像中包含系统的镜像，需要对这块做过滤；

```objectivec
static bool isSelfDefinedImage(const char *imageName) {
    return !strstr(imageName, "/Xcode.app/") &&
    !strstr(imageName, "/Library/PrivateFrameworks/") &&
    !strstr(imageName, "/System/Library/") &&
    !strstr(imageName, "/usr/lib/");
}

static const struct mach_header **copyAllSelfDefinedImageHeader(unsigned int *outCount) {
    unsigned int imageCount = _dyld_image_count();
    unsigned int count = 0;
    const struct mach_header **mhdrList = NULL;
    
    if (imageCount > 0) {
        mhdrList = (const struct mach_header **)malloc(sizeof(struct mach_header *) * imageCount);
        for (unsigned int i = 0; i < imageCount; i++) {
            const char *imageName = _dyld_get_image_name(i);
            if (isSelfDefinedImage(imageName)) {
                const struct mach_header *mhdr = _dyld_get_image_header(i);
                mhdrList[count++] = mhdr;
            }
        }
        mhdrList[count] = NULL;
    }
    
    if (outCount) *outCount = count;
    
    return mhdrList;
}
```

如何获取定义了 `+load` 的类或者分类，在编译时期，包含 `+load` 的 `class` 和 `category` 会写入 Mach-O 文件 data 段的 `__objc_nlcslist` 和 `__objc_nlcatlist` 节，可以通过读取这两部分来获取 no lazy class 和 no lazy category 列表，即定义了 `+load` 方法的类或者分类

```objectivec
static NSArray <LMLoadInfo *> *getNoLazyArray(const struct mach_header *mhdr) {
    NSMutableArray *noLazyArray = [NSMutableArray new];
    unsigned long bytes = 0;
    Class *clses = (Class *)getDataSection(mhdr, "__objc_nlclslist", &bytes);
    for (unsigned int i = 0; i < bytes / sizeof(Class); i++) {
        LMLoadInfo *info = [[LMLoadInfo alloc] initWithClass:clses[i]];
        if (!shouldRejectClass(info.clsname)) [noLazyArray addObject:info];
    }
    
    bytes = 0;
    Category *cats = getDataSection(mhdr, "__objc_nlcatlist", &bytes);
    for (unsigned int i = 0; i < bytes / sizeof(Category); i++) {
        LMLoadInfo *info = [[LMLoadInfo alloc] initWithCategory:cats[i]];
        if (!shouldRejectClass(info.clsname)) [noLazyArray addObject:info];
    }
    
    return noLazyArray;
}
```

hook `+load` 方法：

```objectivec
static void swizzleLoadMethod(Class cls, Method method, LMLoadInfo *info) {
retry:
    do {
        SEL hookSel = getRandomLoadSelector();
        Class metaCls = object_getClass(cls);
        IMP hookImp = imp_implementationWithBlock(^ {
            info->_start = CFAbsoluteTimeGetCurrent();
            ((void (*)(Class, SEL))objc_msgSend)(cls, hookSel);
            info->_end = CFAbsoluteTimeGetCurrent();
            if (!--LMAllLoadNumber) printLoadInfoWappers();
        });
        
        BOOL didAddMethod = class_addMethod(metaCls, hookSel, hookImp, method_getTypeEncoding(method));
        if (!didAddMethod) goto retry;
        
        info->_nSEL = hookSel;
        Method hookMethod = class_getInstanceMethod(metaCls, hookSel);
        method_exchangeImplementations(method, hookMethod);
    } while(0);
}

static void hookAllLoadMethods(LMLoadInfoWrapper *infoWrapper) {
    unsigned int count = 0;
    Class metaCls = object_getClass(infoWrapper.cls);
    Method *methodList = class_copyMethodList(metaCls, &count);
    for (unsigned int i = 0; i < count; i++) {
        Method method = methodList[i];
        SEL sel = method_getName(method);
        const char *name = sel_getName(sel);
        if (!strcmp(name, "load")) {
            IMP imp = method_getImplementation(method);
            LMLoadInfo *info = [infoWrapper findLoadInfoByImp:imp];
            if (!info) {
                info = [infoWrapper findClassLoadInfo];
                if (!info) continue;
            }
            
            swizzleLoadMethod(infoWrapper.cls, method, info);
        }
    }
    free(methodList);
}
```

相应的实现：

[tripleCC/Laboratory](https://github.com/tripleCC/Laboratory/tree/master/HookLoadMethods)

## 应用

[Notification Once](https://blog.sunnyxx.com/2015/03/09/notification-once/)

利用 `+load` 的方法调用时机较早，实现 `AppDelegate` 的瘦身：

```objectivec
/// FooModule.m
+ (void)load
{
    __block id observer =
    [[NSNotificationCenter defaultCenter]
     addObserverForName:UIApplicationDidFinishLaunchingNotification
     object:nil
     queue:nil
     usingBlock:^(NSNotification *note) {
         [self setup]; // Do whatever you want
         [[NSNotificationCenter defaultCenter] removeObserver:observer];
     }];
}
```

- `+ load`方法在足够早的时间点被调用；
- `block` 版本的通知注册会产生一个`__NSObserver *`对象用来给外部 `remove` 观察者；
- `block` 对 `observer` 对象的捕获早于函数的返回，所以若不加`__block`，会捕获到 `nil` ；
- 在 `block` 执行结束时移除 `observer` ，无需其他清理工作；
- 这样，在模块内部就完成了在程序启动点代码的挂载。