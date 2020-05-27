# @propertyWrapper
`@propertyWrapper` 可以对 `get` 和 `set` 方法进行封装，不需要我们再去编写大量重复的 `set` 和 `get` 方法。

例子：
```swift
@propertyWrapper
struct UserDefault<T> {
  let key: String
  let defaultValue: T
  
  var wrappedValue: T {
    get {
      return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
    }
    set {
      UserDefaults.standard.set(newValue, forKey: key)
    }
  }
}

enum GlobalSettings {
  @UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false)
  static var isFooFeatureEnabled: Bool
  
  @UserDefault(key: "BAR_FEATURE_ENABLED", defaultValue: false)
  static var isBarFeatureEnabled: Bool
}
```

在定义 `isFooFeatureEnabled` 和 `isBarFeatureEnabled` 时，我们通过 `@propertyWrapper` 生成一个 `@UserDefault` 属性声明，这样在声明 `isFooFeatureEnabled` 和 `isBarFeatureEnabled` 时不需要再去编写 `set` 和 `get` 方法。

`enum` 类型对应的 `EnumUserDefault` ：

```swift
@propertyWrapper
struct EnumUserDefault<T: RawRepresentable> {
  let key: String
  let defaultValue: T
  
  init(key: String, defaultValue: T) {
    self.key = key
    self.defaultValue = defaultValue
  }
  
  public var wrappedValue: T {
    get {
      guard let rawValue = UserDefaults.standard.object(forKey: key) as? T.RawValue,
        let value = T(rawValue: rawValue) else {
          return defaultValue
      }
      return value
    }
    set {
      UserDefaults.standard.set(newValue.rawValue, forKey: key)
    }
  }
}
```

## 初始化方式

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
}


struct IntWrapper {
  @Wrapper var a = 0 // 隐式调用 init(wrappedValue:) 进行初始化
  @Wrapper(wrappedValue: 0) var b // 初始化方法式显示指定为属性的一部分
}
```

## 访问

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T

  var projectedValue: Wrapper<T> { return self }
}
struct IntWrapper {
  @Wrapper var x = 0
    
    func print() {
      print(x) // `wrappedValue`
      print(_x) // wrapper type itself
      print($x) // `projectedValue`
    }
}
```

外部无法访问 `_x` ，只能通过 `$x` 访问 `projectedValue` 。

## 注意点：
1. 必须要含有 `wrappedValue` 属性；
2. 不能在子类中重写；
3. 不能在 `protocol` 或者 `extensions` 中声明；
4. 不支持添加自定义 `set` 或者 `get` 方法；

## 更多应用和例子

1. [Property wrappers in Swift](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/)
2. [0258-property-wrappers.md](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md#user-defaults)
3. [Swift Property Wrappers - NSHipster](https://nshipster.com/propertywrapper/)
4. [A collection of Swift Property Wrappers](https://github.com/guillermomuntaner/Burritos)


