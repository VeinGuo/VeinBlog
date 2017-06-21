# Swift 封装一个视频播放器 VGPlayer

![Banners.png](http://upload-images.jianshu.io/upload_images/3672149-555ba04e6c5a5e30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## \# 前言
之前学习了 Swift 一直想做一个项目，这次下定决心花了近1个月的空闲时间基于 AVPlayer 封装了一个视频播放器。
## \# 源代码
- GitHub地址：[VGPlayer](https://github.com/VeinGuo/VGPlayer)
- 有什么意见建议可以提 issues,在博文下留言，如果觉得不错，欢迎点star。

## \# 演示

![demo1.gif](http://upload-images.jianshu.io/upload_images/3672149-8d4002b808f1637d.gif?imageMogr2/auto-orient/strip)


![demo2.gif](http://upload-images.jianshu.io/upload_images/3672149-d7eb8ea47b03b5e4.gif?imageMogr2/auto-orient/strip)

<!-- more -->

## \# 功能
- 集成了视频播放器常有的手势，包括单击显示控制视图，双击暂停，水平滑动快进、后退，竖直滑动亮度和音量调节。
- 全屏播放，自适应手机屏幕旋转方向。
- 自定义控制视图

## \# 实现思路

![流程图.png](http://upload-images.jianshu.io/upload_images/3672149-97c3599a629ed811.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### VGPlayer
VGPlayer是一个对AVPlayer封装提供播放功能，displayView为播放器画面绘制。
主要是使用了以下几个类：
- AVURLAsset是 AVAsset的子类，用来本地或者网络视频地址的初始化网络请求，也可以用来获取视频每一帧的画面来实现滑动提前预览图的功能（后续应该会版本迭代加上此功能）
- AVPlayerItem 是对AVPlayer播放的视频数据管理，对播放的Asset资源进行记录，提供或者视频的时间，播放状态等。
- AVPlayer 调控数据和视图
- AVPlayerLayer 进行视频视图绘制

VGPlayer封装AVPlayer提供给调用者可选代理方法

```Swift
// player delegate
    // play state
    func vgPlayer(_ player: VGPlayer, stateDidChange state: VGPlayerState)
    // playe Duration
    func vgPlayer(_ player: VGPlayer, playerDurationDidChange currentDuration: TimeInterval, totalDuration: TimeInterval)
    // buffer state
    func vgPlayer(_ player: VGPlayer, bufferStateDidChange state: VGPlayerBufferstate)
    // buffered Duration
    func vgPlayer(_ player: VGPlayer, bufferedDidChange bufferedDuration: TimeInterval, totalDuration: TimeInterval)
    // play error
    func vgPlayer(_ player: VGPlayer, playerFailed error: VGPlayerError)
```

#### VGPlayerView
- VGPlayerView负责画面的展示,，只作为展示，而绘制层则是AVPlayerLayer提供，可继承此类进行控制视图的自定义
- VGPlayerView封装AVPlayerLayer提供可选代理方法

```Swift
// player view delegate
    /// fullscreen
    func vgPlayerView(_ playerView: VGPlayerView, willFullscreen fullscreen: Bool)
    /// close play view
    func vgPlayerView(didTappedClose playerView: VGPlayerView)
    /// displaye control
    func vgPlayerView(didDisplayControl playerView: VGPlayerView)

```

#### VGPlayerError
- VGPlayerError一个 struct 用来播放出现Error时返回

## \# 细节调整
- 后台播放的实现
设置工程

![backgroundModes.png](http://upload-images.jianshu.io/upload_images/3672149-679c678013de6ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```Swift
// AppDelegate settings
 func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        // Override point for customization after application launch.
        do
        {
            try AVAudioSession.sharedInstance().setCategory(AVAudioSessionCategoryPlayback)
        }
        catch let error as NSError
        {
            print(error)
        }
        return true
    }
```
设置VGPlayer的Background mode
```swift
self.player.backgroundMode = .proceed
```

- VGPlayerUtils 提供判断视频类型方法和一些通用的方法
- UIButton+VGPlayer 扩展按钮点击范围
- Timer+VGPlayer 解决Timer的 retain cycle问题

## \# 参考
- https://techblog.toutiao.com/2017/03/28/fullscreen/
- https://developer.apple.com/library/content/qa/qa1668/_index.html
- https://developer.apple.com/documentation/avfoundation
- https://stackoverflow.com/questions/808503/uibutton-making-the-hit-area-larger-than-the-default-hit-area/13977921
- https://gist.github.com/onevcat/2d1ceff1c657591eebde

## \# 总结
- 了解了AVPlayer的整体结构，对播放过程完整的思路和一些遇到的问题。
- 踩了屏幕旋转细节、按钮点击范围调整的一些交互细节的坑

