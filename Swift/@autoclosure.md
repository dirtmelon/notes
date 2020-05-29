# @autoclosure

使用 `@autoclosure` 声明的闭包支持把表达式自动封装成一个闭包，这样在使用的时候看起来跟普通的参数一样。

```swift
func log(message: @autoclosure () -> String) {
    print(message())
}

log(message: "A auto closure")
```

同时因为闭包延迟执行的特性，对应的表达式会等到调用时才执行，减少性能损耗。 Swift 的 `??` ， `||` 和 `&&` 也是通过这种方法来实现。

```swift
// Optional
@_transparent
public func ?? <T>(optional: T?, defaultValue: @autoclosure () throws -> T)
    rethrows -> T {
  switch optional {
  case .some(let value):
    return value
  case .none:
    return try defaultValue()
  }
}
```
https://github.com/apple/swift/blob/master/stdlib/public/core/Optional.swift#L609-L618

```swift
// Bool
extension Bool {
  @_transparent
  @inline(__always)
  public static func && (lhs: Bool, rhs: @autoclosure () throws -> Bool) rethrows
      -> Bool {
    return lhs ? try rhs() : false
  }

  @_transparent
  @inline(__always)
  public static func || (lhs: Bool, rhs: @autoclosure () throws -> Bool) rethrows
      -> Bool {
    return lhs ? true : try rhs()
  }
}
```
https://github.com/apple/swift/blob/master/stdlib/public/core/Bool.swift#L245-L325

## 相关资料

1. [Building assert() in Swift, Part 1: Lazy Evaluation](https://developer.apple.com/swift/blog/?id=4)
2. [@AUTOCLOSURE 和 ??](https://swifter.tips/autoclosure/)
3. [Using @autoclosure when designing Swift APIs](https://www.swiftbysundell.com/articles/using-autoclosure-when-designing-swift-apis/)