---
title: NSNotificationCenter实现原理？
date: 2016-12-21 17:07:05.000000000 +08:00
tags: Objective-C
---

## \#前言

Cocoa中使用NSNotification、NSNotificationCenter和KVO来实现观察者模式，实现对象间一对多的依赖关系。

本篇文章主要来讨论NSNotification和NSNotificationCenter


## \# NSNotification

`NSNotification`是方便`NSNotificationCenter`广播到其他对象时的封装对象，简单讲即通知中心对通知调度表中的对象广播时发送`NSNotification`对象。

```objc
@interface NSNotification : NSObject <NSCopying, NSCoding>

@property (readonly, copy) NSNotificationName name;
@property (nullable, readonly, retain) id object;
@property (nullable, readonly, copy) NSDictionary *userInfo;
```

`NSNotification`对象包含名称、object、字典三个属性，名称是用来标识通知的标记，object是要通知的对象可以为`nil`,字典用来存储发送通知时附带的信息，也可以为`nil`。
<!-- more -->
## \# NSNotificationCenter

`NSNotificationCenter`是类似一个广播中心站，使用`defaultCenter`来获取应用中的通知中心，它可以向应用任何地方发送和接收通知。在通知中心注册观察者，发送者使用通知中心广播时，以`NSNotification`的`name`和`object`来确定需要发送给哪个观察者。为保证观察者能接收到通知，所以应先向通知中心注册观察者，接着再发送通知这样才能在通知中心调度表中查找到相应观察者进行通知。

### 发送者
发送通知可使用以下方法发送通知

```objc
- (void)postNotification:(NSNotification *)notification;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
```

* 三种方式实际上都是发送`NSNotification`对象给通知中心注册的观察者。

*	发送通知通过name和object来确定来标识观察者,name和object两个参数的规则相同即当通知设置name为kChangeNotifition时，那么只会发送给符合name为kChangeNotifition的观察者，同理object指发送给某个特定对象通知，如果只设置了name，那么只有对应名称的通知会触发。如果同时设置name和object参数时就必须同时符合这两个条件的观察者才能接收到通知。

### 观察者

你可以使用以下两种方式注册观察者

```objc
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;

- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block NS_AVAILABLE(10_6, 4_0);
    // The return value is retained by the system, and should be held onto by the caller in
    // order to remove the observer with removeObserver: later, to stop observation.
```

1.	第一种方式是比较常用的添加`Oberver`的方式，接到通知时执行aSelector。
2.	第二种方式是基于Block来添加观察者，往通知中心的调度表中添加观察者，这个观察者包括一个`queue`和一个`block`,并且会返回这个观察者对象。当接到通知时执行`block`所在的线程为添加观察者时传入的`queue`参数，`queue`也可以为`nil`，那么`block`就在通知所在的线程同步执行。

**这里需要注意的是如果使用第二种的方式创建观察者需要弱引用可能引起循环引用的对象,避免内存泄漏。**

### 移除观察者

在对象被释放前需要移除掉观察者，避免已经被释放的对象还接收到通知导致崩溃。
移除观察者有两种方式：

```objc
- (void)removeObserver:(id)observer;
- (void)removeObserver:(id)observer name:(nullable NSNotificationName)aName object:(nullable id)anObject;
```

* 传入相应的需要移除的`observer` 或者使用第二种方式三个参数来移除指定某个观察者。
* 如果使用基于`-[NSNotificationCenter addObserverForName:object:queue:usingBlock:]`方法在获取方法返回的观察者进行释放。基于这个方法我们还可以让观察者接到通知后只执行一次：

```objc
__block __weak id<NSObject> observer = [[NSNotificationCenter defaultCenter] addObserverForName:kChangeNotifition object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
       NSLog(@"-[NSNotificationCenter addObserverForName:object:queue:usingBlock:]");
      [[NSNotificationCenter defaultCenter] removeObserver:observer];
   }];
```

在`iOS9`中调整了`NSNotificatinonCenter`
![NSNotification](http://ojaltanzc.bkt.clouddn.com/2016-12-21-Notification/NSNotificationCenter.png)
`iOS9`开始不需要在观察者对象释放之前从通知中心移除观察者了。但是如果使用`-[NSNotificationCenter addObserverForName:object:queue:usingBlock:]`方法还是需要手动释放。因为`NSNotificationCenter`依旧对它们强引用。

## \# NSNotificationQueue
`NSNotificationQueue`通知队列，用来管理多个通知的调用。通知队列通常以先进先出（FIFO）顺序维护通。`NSNotificationQueue`就像一个缓冲池把一个个通知放进池子中，使用特定方式通过`NSNotificationCenter`发送到相应的观察者。下面我们会提到特定的方式即合并通知和异步通知。

* 创建通知队列方法:

```objc
- (instancetype)initWithNotificationCenter:(NSNotificationCenter *)notificationCenter NS_DESIGNATED_INITIALIZER;
```

*	往队列加入通知方法:

```objc
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle;
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle coalesceMask:(NSNotificationCoalescing)coalesceMask forModes:(nullable NSArray<NSRunLoopMode> *)modes;
```

* 移除队列中的通知方法:

```objc
- (void)dequeueNotificationsMatching:(NSNotification *)notification coalesceMask:(NSUInteger)coalesceMask;
```

###  发送方式
* `NSPostingStyle`包括三种类型：

```objc
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1,
    NSPostASAP = 2,
    NSPostNow = 3	 
};
```

**NSPostWhenIdle**：空闲发送通知 当运行循环处于等待或空闲状态时，发送通知，对于不重要的通知可以使用。
**NSPostASAP**：尽快发送通知 当前运行循环迭代完成时，通知将会被发送，有点类似没有延迟的定时器。
**NSPostNow** ：同步发送通知 如果不使用合并通知 和`postNotification:`一样是同步通知。

### 合并通知

* `NSNotificationCoalescing`也包括三种类型：

```objc
typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
    NSNotificationNoCoalescing = 0,
    NSNotificationCoalescingOnName = 1,
    NSNotificationCoalescingOnSender = 2
};
```
**NSNotificationNoCoalescing**：不合并通知。
**NSNotificationCoalescingOnName**：合并相同名称的通知。
**NSNotificationCoalescingOnSender**：合并相同通知和同一对象的通知。

* 通过合并我们可以用来保证相同的通知只被发送一次。
* `forModes:(nullable NSArray<NSRunLoopMode> *)modes`可以使用不同的`NSRunLoopMode`配合来发送通知，可以看出实际上`NSNotificationQueue`与`RunLoop`的机制以及运行循环有关系，通过`NSNotificationQueue`队列来发送的通知和关联的`RunLoop`运行机制来进行的。


## \# NSNotificatinonCenter实现原理

* `NSNotificatinonCenter`是使用观察者模式来实现的用于跨层传递消息，用来降低耦合度。
* `NSNotificatinonCenter`用来管理通知，将观察者注册到`NSNotificatinonCenter`的通知调度表中，然后发送通知时利用标识符`name`和`object`识别出调度表中的观察者，然后调用相应的观察者的方法，即传递消息（在Objective-C中对象调用方法，就是传递消息，消息有name或者selector，可以接受参数，而且可能有返回值），如果是基于`block`创建的通知就调用`NSNotification`的`block`。

##\# 参考资料

* [Notification Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Notifications/Introduction/introNotifications.html#//apple_ref/doc/uid/10000043-SW1)
* [NSNotificationCenter part 4: Asynchronous notifications with NSNotificationQueue](http://www.hpique.com/2013/12/nsnotificationcenter-part-4/)
* [NSNotification & NSNotification Center](http://nshipster.com/nsnotification-and-nsnotificationcenter/)


<!— more -->

