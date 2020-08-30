# protocol

## 入门

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

[Protocol-Oriented Programming in Swift](https://developer.apple.com/videos/play/wwdc2015/408/)

WWDC15 上关于面向协议编程的视频。

首先讲述了 `class` 和 `struct` 的缺点，以一个小例子讲述如何使用 `protocol` 重构代码：

```swift
class Ordered {
  func preceds(other: Ordered) -> Bool { fatalError("implement me!") }
}

class Number: Ordered {
  var value: Double = 0
  override func preceds(other: Ordered) -> Bool {
    return self.value < (other as! Number).value
  }
}
```

`Swift` 不支持虚函数，所以我们在子类必须要重写的方法中使用 `fatalError` 来报错。由于父类中定义的方法参数是 `Ordered` 类型，所以必须要使用强制解包来转换为 `Number` 类型，如果参数不是 `Ordered` 类型，那么这里就会崩溃，非常不推荐使用这种做法。

使用 `protocol` 重构之后：

```swift
protocol Ordered {
  func preceds(other: Self) -> Bool
}

struct Number: Ordered {
  var value: Double = 0
  func preceds(other: Number) -> Bool {
    return self.value < other.value
  }
}

func binarySearch<T: Ordered>(sortedkeys: [T], forKey k: T) -> Int {
  var lo = 0
  var hi = sortedkeys.count
  while hi > lo {
    let mid = lo + (hi - lo) / 2
    if sortedkeys[mid].preceds(other: k) {
      lo = mid + 1
    } else {
      hi = mid
    }
  }
  return lo
}
```

里面的 `protocol` 使用了 `Self` ，表示这个参数的具体类型是实现的了这个协议的类型。

没有使用 `Self` 的 `protocol` :

- 可以作为类型来使用： `func sort(inout a: [Ordered])`
- 动态派发，因为不确定具体类型
- 性能较低

使用了 `Self` 的 `protocol` ：

- 只能作为范型限制来使用： `func sort<T: Ordered>(inout a: [T])`
- 静态派发
- 性能较高

使用 `where` 来进行类型限制：

```swift
extension Collection where Generator.Element: Equatable {
  public func indexOf(element: Generator.Element) -> Index? {
    for i in self.indices {
      if self[i] == element {
        return i
      }
    }
    return nil
  }
}
```

[Protocols](https://www.swiftbysundell.com/basics/protocols/)

跟上面的文章差不多，起手还是一个 `class` 经由 `protocol` 进行重构的例子：

```swift
class Player {
    private let avPlayer = AVPlayer()

    func play(_ song: Song) {
        let item = AVPlayerItem(url: song.audioURL)
        avPlayer.replaceCurrentItem(with: item)
        avPlayer.play()
    }

    func play(_ album: Album) {
        let item = AVPlayerItem(url: album.audioURL)
        avPlayer.replaceCurrentItem(with: item)
        avPlayer.play()
    }
}
```

从上面的两个 `play` 方法中可以看出需要为每个支持播放的类型都提供对应的方法，这导致了会有不少重复代码。重新审视下上面的代码，每个播放方法中我们需要使用到的参数其实只有一个 `url` ，定义一个 `Playable` 协议，实现如下：

```swift
protocol Playable {
    var audioURL: URL { get }
}
```

然后 `Player` 可以调整如下：

```swift
class Player {
    private let avPlayer = AVPlayer()

    func play(_ resource: Playable) {
        let item = AVPlayerItem(url: resource.audioURL)
        avPlayer.replaceCurrentItem(with: item)
        avPlayer.play()
    }
}
```

`Playable` 这个名字会给人一种如果遵循了这个类型就可以进行播放的感觉，但是 `Playable` 所有操作只是提供了一个 `audio URL` ，我们可以选择重命名为 `AudioURLConvertible` ，使其更加清晰：

```swift
protocol AudioURLConvertible {
    var audioURL: URL { get }
}

class Player {
    private let avPlayer = AVPlayer()

    func play(_ resource: AudioURLConvertible) {
        ...
    }
}
```

跟着可以定义一个 `PlayerProtocol` ，把播放方法抽象出来，通过新的协议可以隐藏底部的实现，借此可以提供不同的 `Player` 来自定义不同的播放方式：

```swift
class EnqueueingPlayer: PlayerProtocol {
    private let avPlayer = AVQueuePlayer()

    func play(_ resource: AudioURLConvertible) {
        let item = AVPlayerItem(url: resource.audioURL)
        avPlayer.insert(item, after: nil)
        avPlayer.play()
    }
}

extension Player: PlayerProtocol {}
```

我们可以为系统的 `protocol` 编写 `extension` ：

```swift
extension Collection where Element: Numeric {
    func sum() -> Element {
        // The reduce method is implemented using a protocol extension
        // within the standard library, which in turn enables us
        // to use it within our own extensions as well:
        reduce(0, +)
    }
}
let numbers = [1, 2, 3, 4]
numbers.sum() // 10
```

使用 `where` 限制 `Element` 遵循 `Numeric` 协议。

[The different categories of Swift protocols](https://www.swiftbysundell.com/articles/different-categories-of-swift-protocols/)

根据 `protocol` 的作用对其进行分类:
- 是否具备某种能力，大多数以 able 作结尾，比如 `Equatable` ；
- 必要的定义，使得可以确定某个特定的类型具备哪些属性或者方法，比如 `Sequence` ；
- 类型转换，明确声明可以转换为其它类型，如 `CustomStringConvertible` 和 `ExpressibleByStringLiteral` ；
- 接口抽象，接口无需关注具体的对象，只返回或者使用对应的协议；

