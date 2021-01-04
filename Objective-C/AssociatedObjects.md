# Associated Objects
[Associated Objects](https://nshipster.com/associated-objects/)

通过以下三个函数可以进行关联对象的相关操作：

```objectivec
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```

`key` 应该是常量的，唯一的，在 `setter` 和 `getter` 方法中可以进行访问：

```objectivec
static char kAssociatedObjectKey;

objc_getAssociatedObject(self, &kAssociatedObjectKey);
```

但是由于 `selector` 是唯一的，所以可以直接使用 `selector` ：

[https://twitter.com/bbum/status/3609098005](https://twitter.com/bbum/status/3609098005)

不要调用 `objc_removeAssociatedObjects` 来移除关联对象，因为会移除所有关联对象。正确的做法是调用 `objc_setAssociatedObject` 方法并传入 `nil` 来清除关联。

> 比起其他解决问题的方法，关联对象应该被视为最后的选择（事实上 `category` 也不应该作为首选方法）。

> 和其他精巧的 trick 、 hack 、 workaround 一样，一般人都会在刚学习完之后乐于寻找场景去使用一下。尽你所能去理解和欣赏它在正确使用时它所发挥的作用，同时当你选择_这个_解决办法时，也要避免当被轻蔑地问起“这是个什么玩意？”时的尴尬。

[关联对象 AssociatedObject 完全解析 - 面向信仰编程](https://draveness.me/ao/)

- 关联对象其实就是 `ObjcAssociation` 对象
- 关联对象由 `AssociationsManager` 管理并在 `AssociationsHashMap` 存储
- 对象的指针以及其对应 `ObjectAssociationMap` 以键值对的形式存储在 `AssociationsHashMap` 中
- `ObjectAssociationMap` 则是用于存储关联对象的数据结构
- 每一个对象都有一个标记位 `has_assoc` 指示对象是否含有关联对象

[iOS 中的关联对象](https://kingcos.me/posts/2019/associated_objects_in_ios/)

![](media/16097232278502.jpg)

[ChenYilong/CYLDeallocBlockExecutor](https://github.com/ChenYilong/CYLDeallocBlockExecutor)

通过关联属性在对象 `dealloc` 会释放的原理，可以给对象添加一个属性，然后在这个属性 `dealloc` 时进行相关操作，可以达到对象 `dealloc` 进行对应操作的目的。