# protocol

[面向协议编程与 Cocoa 的邂逅 (上)](https://onevcat.com/2016/11/pop-cocoa-1/)

[面向协议编程与 Cocoa 的邂逅 (下)](https://onevcat.com/2016/12/pop-cocoa-2/)

OOP 困境：

- 横切关注点
- 菱形缺陷
- 动态派发安全性

个人感觉动态派发安全性应该属于 `Objective-C` 的问题，是语言层级的，而且现在 `Objective-C` 已经支持 `NSArray<id<Protocol>> *` 的写法，如果添加的对象不支持 `Protocol` ，编译器就会产生警告。

`Swift` 协议支持通过扩展提供默认实现：

- 协议定义
    - 提供实现的入口
    - 遵循协议的类型需要对其进行实现
- 协议扩展
    - 为入口提供默认实现
    - 根据入口提供额外实现

第二篇文章中提供了一个如何使用 `protocol` 对现有代码进行重构的例子，如何通过 `protocol` 的默认实现和关联类型来进行重构，分离各个模块的内容，抽成更小的模块。更小的模块意味着我们可以更加容易得验证各个模块间的内容或者进行重构。 `POP` 编程也不是银弹，日常开发中我们更多还是与 `Cocoa` 这套基于 `OOP` 概念的框架打交道，而 `POP` 也有它自己的问题，所以还是说具体问题具体分析，不可一概而论。

[The different categories of Swift protocols](https://www.swiftbysundell.com/articles/different-categories-of-swift-protocols/)

根据 `protocol` 的作用对其进行分类:
- 是否具备某种能力，大多数以 able 作结尾，比如 `Equatable` ；
- 必要的定义，使得可以确定某个特定的类型具备哪些属性或者方法，比如 `Sequence` ；
- 类型转换，明确声明可以转换为其它类型，如 `CustomStringConvertible` 和 `ExpressibleByStringLiteral` ；
- 接口抽象，接口无需关注具体的对象，只返回或者使用对应的协议；

