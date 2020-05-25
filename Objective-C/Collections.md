# 集合类型
## NSArray/NSMutableArray
通过一个可能为 nil 的数组创建一个可变数组，可以这么写:
```objc
NSMutableArray *mutableObjects = [array mutableCopy];
if (!mutableObjects) {
    mutableObjects = [NSMutableArray array];
}
```
三元运算符:
```objc
NSMutableArray *mutableObjects = [array mutableCopy] ?: [NSMutableArray array];
```
更好的解决方案是使用 `arrayWithArray:` ，即使原数组为 `nil` ，该方法也会返回一个数组对象:
```objc
NSMutableArray *mutableObjects = [NSMutableArray arrayWithArray:array];
```
## NSPointerArray
[NSPointerArray](https://developer.apple.com/documentation/foundation/nspointerarray)

- 行为与 `NSArray` 类似，但是可以存储 `NULL` 值
- `NULL` 值会影响 `count` 
- 可以通过设置 `count` 来插入 `NULL` ;
- 支持 weak ；
- `allObjects` 可以转换成 `NSArray` ，所有的 `NULL` 值都会被去掉，如果 `NSPointerArray` 插入了非对象指针， `allObjects` 会报 `EXC_BAD_ACCESS` 崩溃；
- `compact` 可以去掉数组中的 `NULL` ，调用前需要通过 `addPointer:` 方法添加一次 `NULL` ，否则不起作用

```objc
NSPointerArray *array = [NSPointerArray strongObjectsPointerArray];
array.count = 10;
[array insertPointer:@"Pointer1" atIndex:0];
for (id object in array) {
    NSLog(@"%@", object); // 输出结果为 1 个 NSString 和 11 个 NULL
}
NSLog(@"%lu", (unsigned long)array.count); // 11
NSLog(@"%@", array.allObjects); 
[array addPointer:NULL]; // 注释掉这行下面的 count 还是 11
[array compact];
NSLog(@"%lu", (unsigned long)array.count);
/* 
(
    Pointer1
)
*/
```

## NSSet/NSMutableSet
`allObjects` 方法返回 `NSArray` 。
`NSMutableSet` 有几个很强大的方法，例如 `intersectSet:` ， `minusSet:` 和` unionSet:` 。

![](media/15904201728543.jpg)