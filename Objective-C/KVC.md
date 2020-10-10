# KVC

## 基础

官方文档：

[About Key-Value Coding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107-SW1)

`KVC` 是通过 `NSKeyValueCodinng` 协议来实现的。当一个对象支持 `KVC` 时，它的属性可以通过字符串来进行访问。 `KVC` 对象提供了简单的接口，通过接口和字符串可以访问所有的属性。 `KVC` 是 Cocoa 中一些功能的基石，如 `KVO` ， `Cocoa` 绑定机制， Core Data 等。

### 用途

只要继承自 `NSObject` 就可以使用 `KVC` ， `NSObject` 已默认支持 `NSKeyValueCoding` 协议和提供默认的必须方法， `KVC` 提供了以下特性：

- 获取对象属性。协议定义了一些方法，比如说 `valueForKey:` 和 `setValue:forKey:` ，使用字符串作为参数，可以访问到对象的属性或者对属性进行设置；
- 操作集合属性，跟其它属性一样，提供了对集合属性进行操作的方法，如果需要对集合进行修改， `KVC` 也提供了独特高效的方法；
- 集合属性的操作符，当访问对象的集合属性时， `KVC` 为我们提供了一些操作符，通过这些操作符可以直接对集合获取某些属性，继续计算转换等；
- 获取非对象属性， `KVC` 也支持获取非对象属性，包括纯量属性和结构体等，会自动将它们和对象之间进行转换，以便协议的方法进行调用；

### 适配 KVC

如果想要你的对象支持 `KVC` ，那么你需要使得它们遵循 `NSKeyValueCoding` 协议。幸运的是， `NSObject` 已经为我们做好一切工作，因此如果你想要使用 `KVC` ，那么只需要继承自 `NSObject` 即可。为了保证 `KVC` 生效，你需要保证对象的存取器和变量名遵守相关的规则。

### 获取对象属性

一个对象会在它的 `interface` 声明中定义属性，而属性则会分成以下几个分类：

- 属性，系统提供的一些比较简单的值，如纯量属性，字符串， `Bool` 值等。
- 一对一关系，对于拥有者来说它们是可变对象。一个对象的属性可以在对象本身不改变的情况下发生改变。举个例子，比如说一个银行客户的对象拥有一个 `Person` 的 `owner` 属性， `Person` 拥有一个地址属性。 `owner` 就可以在不改变银行客户的引用关系的前提下改变自己的地址属性；
- 一对多，集合对象，比如说 `NSArray` 或者 `NSSet` ，也可以使用其它的一些自定义集合类型；

```objectivec
@interface BankAccount : NSObject
 
@property (nonatomic) NSNumber* currentBalance;              // An attribute
@property (nonatomic) Person* owner;                         // A to-one relation
@property (nonatomic) NSArray< Transaction* >* transactions; // A to-many relation
 
@end
```

为了保持封装性，一个对象会提供为属性提供存取方法作为它的接口。

```objectivec
[myAccount setCurrentBalance:@(100.0)];
```

这样很直接，但是会缺少灵活性。 `KVC` 为对象提供了一种通过字符串来获取属性的机制。

### 通过 `Keys` 或者 `KeyPaths` 识别对象的属性

`key` 是一个字符串，对应某个属性。通常情况下， `key` 会跟属性的名字一致。使用  `ASCII` 编码，不包含空格，以小写字母开头 （当然了，也会有例外，比如说 `URL` 属性）。

对于 `BankAccount` 来说，我们可以通过以下属性来设置 `currentBalance` ：

```objectivec
[myAccount setValue:@(100.0) forKey:@"currentBalance"];
```

实际上，我们可以使用相同的方法，不同的 `key` 参数来获取 `myAccount` 对象的所有属性。

我们可以通过 `.` 来使用 `KeyPath` 。假设 `Person` 和 `Address` 也符合 `KVC` 规范，我们可以通过 `owner.address.street` 的方式来访问账户所有者的地址中的街道信息。

`NSObject` 已经实现了 `NSKeyValueCoding` 协议所需要的方法，所以只需要继承自 `NSObject` ，就可以得到默认的实现和支持 `KVC` 。

- `valueForKey:` ，返回一个以 `key` 参数来进行命名的属性。如果说属性无法被 `key` 通过定好的规则搜索到，对象会调用 `valueForUndefinedKey:` 方法，这个方法的默认实现是抛出一个 `NSUndefinedKeyException` 异常，但是子类可以通过重写这个方法来更优雅地处理这个场景；
- `valueForKeyPath:` ，返回接收器中满足 `keyPath` 路径的值。所有在这个 `keyPath` 路径中的对象都需要满足特定的 `key` 对应的 `KVC` 机制，如果说 `valueForKey:` 找不到对应的存取方法，就会收到 `valueForUndefinedKey:` 消息；
- `dictionaryWithValuesForKeys:` ，返回 `value` 和 `key` 组成的 `NSDictionary` ，它会为数组的每个 `key` 调用 `valueForKey:` 方法来获取对应的值。

集合对象，比如说 `NSArray` ， `NSSet` 和 `NSDictionary` ，不可以包含 `nil` 。你可以使用 `NSNull` 对象来替换 `nil` ， `NSNull` 提供了一个单例来表示 `nil` 值。 `dictionaryWithValuesForKeys:` 和 `setValuesForKeysWithDictionary:` 会在 `NSNull` （ dictionary 参数）和 `nil` （属性）中自动切换。

`KeyPath` 也支持多对一关系，当 `key-path` 路径中有一对多的关系时，那么就会返回数组。比如说 `transactions.payee` 会以数组形式返回所有 `transactions` 中的 `payee` 对象。

### 通过 `Keys` 设置属性值

和 `getter` 一样， `KVC` 也提供了一组通用的 `setter` 方法，由 `NSObject` 中 `NSKeyValueCoding` 协议的默认方法提供：

- `setValue:forKey:` ，使用 `value` 来设置对象中对应 `key` 的属性。 `setValue:forKey:` 的默认实现会自动对 `NSNumber` 和 `NSValue` 对象进行解包，把它们转换为对应的纯量和结构体，然后设置到对应的属性中。如果对象中没有和 `key` 对应的 `setter` ，那么对象就会调用它自己的 `setValue:forUndefinedKey:` 方法，这个方法的默认实现会抛出一个 `NSUndefinedKeyException` 异常。子类可以通过重写这个方法来 实现自定义逻辑。
- `setValue:forKeyPath:` ，使用 `value` 来设置对象中与 `keyPath` 路径相符的属性。当存在 `keyPath` 路径上不支持对应的 `key` 的 `KVC` 时，就会收到 `setValue:forUndefinedKey:` 消息。
- `setValuesForKeysWithDictionary:` ，批量设置属性，使用 `dictionary` 中的 `key` 来指明属性。它通过调用 `setValue:forKey:` 方法来为每一对 `key-value` 进行设置，自动将 `NSNull` 对象替换为 `nil` 。

在默认的实现中，当你尝试设置一个非对象的属性为 `nil` 时， `KVC` 会调用 `setNilValueForKey:` 方法。这个方法的默认实现会抛出一个 `NSInvalidArgumentException` ，对象可以通过重写这个行为来提供一个默认值或者标记值（ marker value ）。

[iOS 开发：『Crash 防护系统』（三）KVC 防护](https://juejin.im/post/6844903934662803464)

`KVC` 崩溃防护。上面提到 `KVC` 相关的崩溃，这篇文章中相关防护也是对这些方法进行 `hook` ，替换掉原来的实现。

```objectivec
/********************* NSObject+KVCDefender.h 文件 *********************/
#import <Foundation/Foundation.h>

@interface NSObject (KVCDefender)

@end

/********************* NSObject+KVCDefender.m 文件 *********************/
#import "NSObject+KVCDefender.h"
#import "NSObject+MethodSwizzling.h"

@implementation NSObject (KVCDefender)

// 不建议拦截 `setValue:forKey:` 方法
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{

        // 拦截 `setValue:forKey:` 方法，替换自定义实现
        [NSObject yscDefenderSwizzlingInstanceMethod:@selector(setValue:forKey:)
                                       withMethod:@selector(ysc_setValue:forKey:)
                                        withClass:[NSObject class]];

    });
}

- (void)ysc_setValue:(id)value forKey:(NSString *)key {
    if (key == nil) {
        NSString *crashMessages = [NSString stringWithFormat:@"crashMessages : [<%@ %p> setNilValueForKey]: could not set nil as the value for the key %@.",NSStringFromClass([self class]),self,key];
        NSLog(@"%@", crashMessages);
        return;
    }

    [self ysc_setValue:value forKey:key];
}

- (void)setNilValueForKey:(NSString *)key {
    NSString *crashMessages = [NSString stringWithFormat:@"crashMessages : [<%@ %p> setNilValueForKey]: could not set nil as the value for the key %@.",NSStringFromClass([self class]),self,key];
    NSLog(@"%@", crashMessages);
}

- (void)setValue:(id)value forUndefinedKey:(NSString *)key {
    NSString *crashMessages = [NSString stringWithFormat:@"crashMessages : [<%@ %p> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key: %@,value:%@'",NSStringFromClass([self class]),self,key,value];
    NSLog(@"%@", crashMessages);
}

- (nullable id)valueForUndefinedKey:(NSString *)key {
    NSString *crashMessages = [NSString stringWithFormat:@"crashMessages :[<%@ %p> valueForUndefinedKey:]: this class is not key value coding-compliant for the key: %@",NSStringFromClass([self class]),self,key];
    NSLog(@"%@", crashMessages);
    
    return self;
}

@end
```