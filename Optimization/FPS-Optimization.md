# FPS Optimization

## 如何检测卡顿
### CADisplayLink
`CADisplayLink` 支持以和屏幕刷新率同步的方式更新屏幕上的内容，当注册到 `RunLoop` 后，每次当屏幕时，都会执行 `CADisplayLink` 的方法，与 `NSTimer` 不同的时其内部操作的是一个 `Source` 。 `YYKit` 中通过 `CADisplayLink` 写了个 `debug` 时显示 `FPS` 的小工具：

```objc
- (void)tick:(CADisplayLink *)link {
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    
    _count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
    if (delta < 1) return;
    _lastTime = link.timestamp;
    float fps = _count / delta;
    _count = 0;
    
    CGFloat progress = fps / 60.0;
    UIColor *color = [UIColor colorWithHue:0.27 * (progress - 0.2) saturation:1 brightness:0.9 alpha:1];
    
    NSMutableAttributedString *text = [[NSMutableAttributedString alloc] initWithString:[NSString stringWithFormat:@"%d FPS",(int)round(fps)]];
    [text setColor:color range:NSMakeRange(0, text.length - 3)];
    [text setColor:[UIColor whiteColor] range:NSMakeRange(text.length - 3, 3)];
    text.font = _font;
    [text setFont:_subFont range:NSMakeRange(text.length - 4, 1)];
    
    self.attributedText = text;
}
```

`CADisplayLink` 刷新频率跟屏幕刷新一致，刷新频率为 `60HZ` ，每次刷新时 `_count` 都会加 1 ，如果 `delta < 1` ，则表示没有超过 `1s` ，直接返回，如果超过 `1s` 则通过 `_count` 计算出 `FPS` 。记得 `CADisplayLink` 需要添加 `NSRunLoopCommonModes` 。

## 渲染原理
[iOS Rendering 渲染全解析](https://github.com/RickeyBoy/Rickey-iOS-Notes/blob/master/%E7%AC%94%E8%AE%B0/iOS%20Rendering.md)

具体涉及的内容：

1. 计算机渲染原理：CPU + GPU ， 图形渲染流水线；
2. 屏幕成像和卡顿：屏幕撕裂，掉帧的原因，双缓冲与三缓冲；
3. iOS 中的渲染框架：CALayer，CALayer 与 UIView ；
4. Core Animation 渲染全内容：CA 渲染流水线， Commit Transaction ， Rendering Pass ；
5. 离屏渲染；

> 屏幕卡顿的根本原因：CPU 和 GPU 渲染流水线耗时过长，导致掉帧。

> Vsync 与双缓冲的意义：强制同步屏幕刷新，以掉帧为代价解决屏幕撕裂问题。

> 三缓冲的意义：合理使用 CPU、GPU 渲染性能，减少掉帧次数。

## 一些优化
[关于iOS离屏渲染的深入研究](https://zhuanlan.zhihu.com/p/72653360)

- 即刻大量应用 `AsyncDisplayKit(Texture)` 作为主要渲染框架，对于文字和图片的异步渲染操作交由框架来处理。关于这方面可以看[之前的一些介绍](https://link.zhihu.com/?target=https%3A//medium.com/jike-engineering/asyncdisplaykit%25E4%25BB%258B%25E7%25BB%258D-%25E4%25B8%2580-6b871d29e005)
- 对于图片的圆角，统一采用 `precomposite` 的策略，也就是不经由容器来做剪切，而是预先使用CoreGraphics为图片裁剪圆角
- 对于视频的圆角，由于实时剪切非常消耗性能，我们会创建四个白色弧形的 `layer` 盖住四个角，从视觉上制造圆角的效果
- 对于 `view` 的圆形边框，如果没有 `backgroundColor` ，可以放心使用 `cornerRadius` 来做
- 对于所有的阴影，使用 `shadowPath` 来规避离屏渲染
- 对于特殊形状的 `view` ，使用 `layer mask` 并打开  `shouldRasterize` 来对渲染结果进行缓存
- 对于模糊效果，不采用系统提供的 `UIVisualEffect` ，而是另外实现模糊效果（ `CIGaussianBlur` ），并手动管理渲染结果

[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

产生卡顿的原因和解决方案：

CPU 层级：

1. 对象创建，对象创建时会分配内存，调整属性，甚至说读取文件等操作，可以考虑使用更轻量的对象，如使用 `CALayer` 代替 `UIView` 。延迟对象的创建事件，复用机制，如 `UITableViewCell/UICollectionViewCell` ；
2. 对象调整，这里特别说一下 `CALayer` ： `CALayer` 内部并没有属性，当调用属性方法时，它内部是通过运行时 `resolveInstanceMethod` 为对象临时添加一个方法，并把对应属性值保存到内部的一个 `Dictionary` 里，同时还会通知 `delegate` 、创建动画等等，非常消耗资源。 `UIView` 的关于显示相关的属性（比如 `frame/bounds/transform` ）等实际上都是 `CALayer` 属性映射来的，所以对 `UIView` 的这些属性进行调整时，消耗的资源要远大于一般的属性。对此你在应用中，应该尽量减少不必要的属性修改；
3. 布局计算，视图布局的计算是 App 中最为常见的消耗 CPU 资源的地方。如果能在后台线程提前计算好视图布局、并且对视图布局进行缓存，那么这个地方基本就不会产生性能问题了；
4. 文本计算与渲染，文本的排版和绘制都是在主线程进行，当有大量不定高的文字内容需要进行高度计算和渲染时，会占用非常多的 CPU 的时间，可以使用 `TextKit` 或者 `CoreText` 对文本进行异步绘制，缓存绘制结果，在复用时使用；
5. 图片解码与绘制，当使用 `UIImage` 或者 `CGImageSource` 创建图片时，图片数据不会立刻解码。只有当 `CALayer` 提交到 GPU 前， `CGImage` 中的数据才会进行解码，而且是发生在主线程。常见的做法是在后台线程先把图片绘制到 `CGBitmapContext` 中，然后从 `Bitmap` 直接创建图片。目前常见的网络图片库都自带这个功能。绘制则是指常用的以 `CG` 开头的方法把图像绘制到画布中，也可以通过异步绘制来进行优化：

```objectivec
- (void)display {
    dispatch_async(backgroundQueue, ^{
        CGContextRef ctx = CGBitmapContextCreate(...);
        // draw in context...
        CGImageRef img = CGBitmapContextCreateImage(ctx);
        CFRelease(ctx);
        dispatch_async(mainQueue, ^{
            layer.contents = img;
        });
    });
}
```

GPU 层级：

1. 纹理的渲染，所有的 `Bitmap` ，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，GPU 占用非常高，界面仍然会掉帧。避免这种情况的方法只能是尽量减少在短时间内大量图片的显示，尽可能将多张图片合成为一张进行显示。当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU 都会带来额外的资源消耗；
2. 视图的混合，当多个视图重叠在一起时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，把多个视图预先渲染为一张图片来显示；
3. CALayer 的 `border` 、圆角、阴影、遮罩（ `mask` ）， `CASharpLayer` 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，但这会把原本离屏渲染的操作转嫁到 CPU 上去。对于只需要圆角的某些场合，也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性；
