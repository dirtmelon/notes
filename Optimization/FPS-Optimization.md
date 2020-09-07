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

- 即刻大量应用AsyncDisplayKit(Texture)作为主要渲染框架，对于文字和图片的异步渲染操作交由框架来处理。关于这方面可以看我[之前的一些介绍](https://link.zhihu.com/?target=https%3A//medium.com/jike-engineering/asyncdisplaykit%25E4%25BB%258B%25E7%25BB%258D-%25E4%25B8%2580-6b871d29e005)
- 对于图片的圆角，统一采用“precomposite”的策略，也就是不经由容器来做剪切，而是预先使用CoreGraphics为图片裁剪圆角
- 对于视频的圆角，由于实时剪切非常消耗性能，我们会创建四个白色弧形的layer盖住四个角，从视觉上制造圆角的效果
- 对于view的圆形边框，如果没有backgroundColor，可以放心使用cornerRadius来做
- 对于所有的阴影，使用shadowPath来规避离屏渲染
- 对于特殊形状的view，使用layer mask并打开shouldRasterize来对渲染结果进行缓存
- 对于模糊效果，不采用系统提供的UIVisualEffect，而是另外实现模糊效果（CIGaussianBlur），并手动管理渲染结果
