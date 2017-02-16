---
title: 实现一个iOS渐变背景动画效果的Switch
date: 2017-2-16 16:16:21.000000000 +08:00
tags: Objective-C
---

## 前言
在dribbble看到一个Switch动画效果就手痒想实现，下面就是我实现的思路。
## 源代码
GitHub地址：[VGGradientSwitch](https://github.com/VeinGuo/VGGradientSwitch)
如果觉得不错，欢迎点star。
## 设计图
来自dribbble上的设计作者 [Nick Buturishvili](https://dribbble.com/nick_buturishvili)
![image](https://d13yacurqjgara.cloudfront.net/users/408943/screenshots/2272690/switch.gif)
## 效果图
![image](http://ojaltanzc.bkt.clouddn.com/switch_button_1.gif)
<!-- more -->

## 思路
- 首先解刨一下设计图

1. 外观和iOS原生UISwitch相同
2. 观察动图发现Switch背景图为渐变色，这也是这个开关设计的一大亮点
3. 开关上的纽扣，打开时状态是一个勾，关闭时是一个叉。
4. 打开动画，勾边线放大移动边做形变变成点再变换成叉放大后恢复原状，背景颜色由青色转换到橘黄色。
5. 关闭动画，叉边先放大移动边做形变再变成勾放大后恢复原状，背景颜色由橘黄色转换到青色


## 实现
- 渐变背景图是通过`CAGradientLayer`实现。通过设计图取色拿到颜色十六进制(0x08ded6,0x18deb9,0xef9c29,0xe76b39)四个颜色,创建出一个3倍的switch宽度渐变图
如图: ![image](http://upload-images.jianshu.io/upload_images/3672149-30b1f130c7724472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码如下:

```object
CAGradientLayer *gradientLayer = [CAGradientLayer layer];
gradientLayer.locations = @[@0, @.33, @.63, @1];
gradientLayer.startPoint = CGPointMake(0, 0);
gradientLayer.endPoint = CGPointMake(1, 0);
gradientLayer.frame = CGRectMake(0, 0, self.frame.size.width * 3, self.frame.size.height);
```

- 边框使用`UIBezierPath`设置一个圆角边框

![image](http://upload-images.jianshu.io/upload_images/3672149-16dc2c56abc88cf2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 勾、点和叉的实现使用到`UIBezierPath`提供path，然后`CAShapeLayer`创建，勾图形实现代码如下：

```object
UIBezierPath *tickPath = [UIBezierPath bezierPath];
[tickPath moveToPoint:CGPointMake(self.frame.size.width/8 * 3, self.frame.size.width/2)];
CGPoint p1 = CGPointMake(self.frame.size.width/2, self.frame.size.width/8 * 5);
[tickPath addLineToPoint:p1];
CGPoint p2 = CGPointMake(self.frame.size.width/8 * 6, self.frame.size.width/8 * 3);
[tickPath addLineToPoint:p2];

CAShapeLayer *layer = [[CAShapeLayer alloc] init];
layer.lineCap = kCALineCapRound;
layer.lineJoin = kCALineJoinRound;
layer.fillColor = [UIColor clearColor].CGColor;
layer.strokeColor = [UIColor whiteColor].CGColor;
layer.lineWidth = 2;
layer.path = tickPath.CGPath;
```
![image](http://upload-images.jianshu.io/upload_images/3672149-8012e458f4808968.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 渐变背景颜色的动画效果，笔者是将`CAGradientLayer`添加到一个UIView上然后直接使用，UIView的动画方法然后做位移，代码如下：

```object
[UIView animateKeyframesWithDuration:.5 delay:.1 options:UIViewKeyframeAnimationOptionCalculationModePaced animations:^{
        self.gradientView.frame = CGRectMake(-self.frame.size.width *2, 0, self.frame.size.width *3, self.frame.size.height);
    } completion:^(BOOL finished) {
    
    }];
```

![image](http://upload-images.jianshu.io/upload_images/3672149-e93f0003fde40f55.gif?imageMogr2/auto-orient/strip)

- 图形形变动画主要利用`Core Animation`实现,用到了`CAKeyframeAnimation`、`CABasicAnimation`、`CAAnimationGroup`
- 勾->点动画  先放大勾后然后做缩小形变成点 
- 点->勾动画  做形变成勾后做放大
- 动画使用了"path"和"transform"，形变动画使用path提供路径数组，这里提供原本勾的路径和点的路径这样路径就从勾形变到点
代码如下:

- 放大动画 使用`CABasicAnimation`

```object
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"transform"];
    CATransform3D tr = CATransform3DIdentity;
    tr = CATransform3DTranslate(tr, _rect.size.width/2, _rect.size.height/2, 0);
    tr = CATransform3DScale(tr, 1.2, 1.2, 1);
    tr = CATransform3DTranslate(tr, -_rect.size.width/2, -_rect.size.height/2, 0);
    animation.toValue = [NSValue valueWithCATransform3D:tr];
    animation.autoreverses = YES;
    animation.timingFunction  = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
```

- 形变动画 使用`CAKeyframeAnimation`具体的使用可以Google一下这里就不多赘言,`values`传入的是勾的path和点path

```object
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animationWithKeyPath:@"path"];
    animation.values = values;
    animation.keyTimes = keyTimes;
    animation.beginTime = beginTime;
```

- 使用`CAAnimationGroup`做组合动画,组合动画代理方法可以判断组合动画是否完成，`<CAAnimationDelegate>` 代码如下:

```object
	 // scaleAnimation 放大 lineAnimation线条形变
    CAAnimationGroup *animationGroup = [CAAnimationGroup animation];
    animationGroup.animations = @[scaleAnimation,lineAnimation];
    animationGroup.duration = .5;
    animationGroup.repeatCount = 1;
    animationGroup.removedOnCompletion = NO;
    animationGroup.fillMode = kCAFillModeForwards;
    animationGroup.timingFunction  = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    animationGroup.delegate = self;
    
    // CAAnimationDelegate  动画是否结束
    - (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag{
    }
```

![image](http://upload-images.jianshu.io/upload_images/3672149-6989c7df61f3ff5b.gif?imageMogr2/auto-orient/strip)

- 剩下的就是移动这个按钮然后由点形变成叉这里就不再说明，可以直接看GitHub代码,以上就是分解动画的一些思路和解决办法，但是动画要流畅和交互不违和还需要细微调整
- 有什么代码和实现的效果建议可以提issue给我，如果喜欢的话点一下 star 哦


