---
title: iOS Memory management
date: 2016-12-29 13:44:29.000000000 +08:00
tags: Objective-C
---

## \# 前言

>反复地复习iOS基础知识和原理，打磨知识体系是非常重要的，本篇就是重新温习iOS的内存管理。

内存管理是管理对象生命周期，在对象不需要时进行内存释放的编程规范。

## \# MRR时代

### 概要 

Objective-C内存管理使用使用引用计数(Reference Counting)来管理内存。
>在OS X 10.8以后也不再使用垃圾回收机制，iOS则从来都没有支持垃圾回收机制。

当`create`或者`copy`对象时，会计数为1，其他对象需要`retain`时，会增加引用计数。持有对象的所有者也可以放弃所有权，放弃所有权时减少计数，当计数为0时就会释放对象。
如图：

![memory_management](http://ojaltanzc.bkt.clouddn.com/2016-12-29-memory-management/memory_management_2x.png)
<!-- more -->
#### Memory Management Policy 内存管理策略

- 通过分配内存或`copy`来创建任何对象
- 使用方法 `alloc`, `allocWithZone:`, `copy`, `copyWithZone:`, `mutableCopy` , `mutableCopyWithZone:`创建对象
- 通过`retain`来获取不是自己创建对象的所有权。以下两种情况使用`retain`:
	1.	在`accessor method`或者`init method`方法获取所需要的对象所有权为属性`property` 。
	2. 需要操作对象时，避免对象被释放而导致错误，需要`retain`持有对象。
- 发送`release`, `autorelease`消息来释放不需要的对象。
- 不要不是你创建的对象和没有所有权的对象发送`release`消息。

#### Practical Memory Management 实际内存管理

- Autorelease pools
	- 向对象发送`autorelease`消息，会将对象标记为延迟释放，当对象超出当前作用域时，释放对象。
	- `AppKit frameworks`和`UIKit frameworks`在事件循环的每个周期开始时，在主线程上创建一个自动释放池，并在此次时间循环结束时，释放它，从而释放在处理时生成的所有自动释放的对象。因此，通常不需要自己创建`autoreleasePool`，当然，以下情况你需要自己创建和销毁`autoreleasePool`：
	
		1. 如果你编写的代码不是基于`UI framework`的程序，如`command-line tool`命令行工具。
		2. 如果你需要写一个循环，创建许多临时对象，如读入大量的铜像同时改变图片尺寸，图像读入到`NSData`对象，并从中生成`UIImage`对象，改变该对象尺寸生成新的`UIImage`对象。
		3. 如果你创建一个长期存在线程并且可能产生大量的`autorelease`对象。
	
- `autoreleasePool`推荐使用以下方法：
	
```objc
@autoreleasepool {
	//do something
}
```
	
- dealloc

- 当`NSObject`对象的引用计数为0时，销毁该对象前会调用`dealloc`方法，用来释放该对象拥有的所有资源，包裹实例变量指向的对象。
	例子:
	
```objc
 //	MRR
	- (void)dealloc{
    [_firstName release];
    [_lastName release];
    [super dealloc];
	}
```

>Important: Never invoke another object’s dealloc method directly.You must invoke the superclass’s implementation at the end of your implementation.You should not tie management of system resources to object lifetimes; see Don’t Use dealloc to Manage Scarce Resources.When an application terminates, objects may not be sent a dealloc message. Because the process’s memory is automatically cleared on exit, it is more efficient simply to allow the operating system to clean up resources than to invoke all the memory management methods.
>不要直接调用另一个对象的dealloc方法。
>你必须在类使用结束时调用父类的实现。
>你不应该把系统资源与对象的生命周期绑定。
>因为进程的内存退出时，对象可能无法发送dealloc消息,该方法的内存被自动退出清零,所以让操作系统清理资源比调用所有的内存管理方法更有效。

### 内存管理实践

#### 使用访问器方法使内存管理更轻松

>如果类有一个属性是一个对象，你必须确保使用该对象时，它不会被释放。因此在设置时，必须声明对象的所有权。还必须保证持有这些对象所有权的放弃。

- 使用`set`和`get`方法来实现，更方便管理内存（主要是省写很多`retain`和`release`）。
例子如下：

```objc
@interface Counter : NSObject
@property (nonatomic, retain) NSNumber *count;
@end;
```

`Counter`类有一个属性是`NSNumber`对象，属性声明了`set`和`get`两个访问器方法，在`get`中就是返回`synthesized`实例变量，所以没必要`retain`或者`release`：

```objc
- (NSNumber *)count {
	return _count;
}
```

`set`方法：

```objc
- (void)setCount:(NSNumber *)newCount {
    [newCount retain]; // 先`retain`确保新数据不被释放
    [_count release];  // 释放旧对象所有权
    // Make the new assignment.
    _count = newCount; 	// 将新值赋给_count
}
```

先`retain`确保新数据不被释放，释放旧的对象所有权(Objective-C允许向`nil`发送消息)。你必须在`[newCount retain]`之后再`[_count release]`确保外部不会被`dealloc`。

#### 使用访问器方法设置属性

```objc
// 方法一
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [self setCount:zero];
    [zero release];
}
// 方法二
- (void)reset {
    NSNumber *zero = [[NSNumber alloc] initWithInteger:0];
    [_count release];
    _count = zero;
}
```

方法二没有对`count`属性赋新值时没有使用`set`访问方法，也不会触发`KVO`，可能在特殊情况导致错误（比如忘记了 `retain`或者`release`，或者如果实例变量的内存管理发生了变化）。除了第一种方法，或者直接使用`self.count = zero;`。

#### 不要在初始化和`dealloc`中使用访问器方法

不应该使用`set`和`get`方法在`init`和`dealloc`。应该使用`_`直接访问成员变量进行初始化和`dealloc`。如下：

```objc
- init {
    self = [super init];
    if (self) {
        _count = [[NSNumber alloc] initWithInteger:0];
    }
    return self;
}
// 由于Counter类具有对象实例变量，因此还必须实现dealloc方法。
// 它应该通过向任何实例变量发送一个释放消息来放弃它的所有权，最终它应该调用super的实现
- (void)dealloc {
    [_count release];
    [super dealloc];
}
```

#### 使用弱引用来避免循环引用

- `retain`对象，实际是对对象的强引用(strong reference),一个对象在所有强引用都没有被释放之前，不能释放对象。因此，如果有两个对象互相持有对方或者间接互相引用，会导致循环引用。这时候就需要弱引用对方来打破这个循环。

如父亲强引用儿子，儿子强引用孙子，那么倒过来孙子只能弱引用儿子，儿子也只能弱引用父亲。`Cocoa`建立了一个约定，副对象应该强引用子对象，并且子对象应该只对父对象弱引用。
`Cocoa`中常见的例子包括代理方法`delegate`，`data source`,`observer`,`target`等等

必须小心将消息发送到持有只是一个弱引用的对象。当发送消息给一个被`dealloc`的弱引用对象时，你的应用程序会崩溃（这是在`MRR`时期的代理`delegate`会出现，因为当时对代理弱引用的修饰符是`assign`,`assign`弱引用并不会在对象`dealloc`时，把对象置为`nil`。而`ARC`时代使用`weak`则会在对象`dealloc`时置为`nil`）。

#### 避免正在使用的对象被释放

- `Cocoa`的所有权策略规定接收的对象通常在整个调用方法的范围内保证有效。还应该是在当前方法范围内，而不必担心它被释放。对象的`getter`方法返回一个缓存的实例变量或者一个计算的值，这不重要，重要的是，对象在需要的使用时还是有效的。
- 有两类例外情况：
	- 当一个对象从基本的集合类删除时
	
	```objc
	heisenObject = [array objectAtIndex:n];
	[array removeObjectAtIndex:n];
	// heisenObject 现在可能无效
	```
		
	- `n`从集合`array`删除时也会向`n`发送`release`（而不是`autorelease`）消息。如果`array`集合时被删除`n`对象的唯一拥有者，被移除的对象`n`是立即被释放的。`heisenObject`并没有对`n`进行`retain`，所以当`n`从`array`删除时同时被释放。

	**正确的做法**
	
	```objc
	heisenObject = [[array objectAtIndex:n] retain];
	[array removeObjectAtIndex:n];
	// Use heisenObject...
	[heisenObject release];
	```

	- 当一个父对象被释放时
		
	```objc
	id parent = <#create a parent object#>;
	// ...
	heisenObject = [parent child] ;
	[parent release]; // Or, for example: self.parent = nil;
	// heisenObject 现在可能无效
		```
		
-	在某些情况下，从另一个对象获取的对象，然后直接或者间接的释放负对象。如果释放父对象导致它被释放，并且父对象是子对象唯一所有者，那么子对象`heisenObject`将被同一时间释放。所以正确的做法还是子对象`heisenObject`获取的时候先`retain`一次。

#### Collections类拥有它们所包含的对象所有权

- 添加一个对象到一个`collection`中，如(数组、字典、集合)时，`collection`会得到该对象所有权。当对象从`collection`删除或者`collection`自己被释放时，`collection`将释放它拥有的所有权。

```objc
NSMutableArray *array = <#Get a mutable array#>;
NSUInteger i;
// ...
for (i = 0; i < 10; i++) {
    NSNumber *allocedNumber = [[NSNumber alloc] initWithInteger:i];
    [array addObject:allocedNumber];
    [allocedNumber release];
}
```

#### 通过引用计数实现所有所有权策略

- 所有圈策略是通过引用计数实现的，通常`retain`方法后被称为`retain count`。每个对象都有一个引用计数。

	-	当你创建一个对象，它的引用计数为`1`
	-  当你给对象发送`retain`消息，引用计数`+1`
	-	当你给对象发送`release`消息，引用计数`-1`
	- 	当你给对象发送一个`autorelease`消息，它的引用计数器将在当前的自动释放池结束后`-1`
	-  当对象的引用计数为`0`时将被释放

## \# ARC时代

### 概要

iOS5后出现了`ARC`。那么`ARC`是什么呢？
自动引用计数`ARC`是一种编译器的功能，为`Objective-C`对象提供了自动化的内存管理。
在`ARC`不需要开发者考虑保留或者释放的操作，就是不用自己手动`retain`、`release`和`autorelease`（😄开心），让开发者可以专注写有趣的代码。

**当然`ARC`依然是基于引用计数管理内存。**

### ARC 强制新规则

`ARC`相对于`MRR`强制加了一些新的规则。

- 你不能主动调用`dealloc`、或者调用`retain`,`release`, `retainCount`,`autorelease`就是这些都不用你写了。也不能`@selector(retain)`, `@selector(release)`这样子调用。
- 你可以实现一个`dealloc`方法，如果你需要管理资源而不是释放实例变量(比如解除监听、释放引用、socket close等等)。在重写`dealloc`后需要`[super dealloc]`（在手动管理引用计数时才需要）。
- 仍然可以使用`CFRetain`，`CFRelease`等其它对象。
- 你不能使用`NSAllocateObject`或者`NSDeallocateObject`。
- 你不能使用`C`结构体，可以创建一个`Objective-C`类去管理数据而不是一个结构体。
- `id`和`void`没有转换关系,你必须使用`cast`特殊方式，以便在作为函数参数传递的`Objective-C`对象和`Core Foundation`类型之间进行转换。
- 你不能使用`NSAutoreleasePool`，使用`@autoreleasepool`。
- 没必要使用`NSZone`

### ARC 使用新修饰符

- `__strong` 强引用,用来保证对象不会被释放。
- `__weak`弱引用 释放时会置为`nil`
- `__unsafe_unretained`弱引用 可能不安全，因为释放时不置为`nil`。
- `__autoreleasing`对象被注册到`autorelease pool`中方法在返回时自动释放。

### 内存泄漏

`ARC`还是基于引用计数的管理机制所以依然会出现循环引用。

#### block使用中出现循环引用

- 常见的有情况在`block`使用中出现循环引用

```objc
	// 情况一
	self.myBlock = ^{
	self.objc = ...;
	};
	// 情况二
	Dog *dog = [[Dog alloc] init];
	dog.myBlock = ^{
		 // do something
	};
	self.dog = dog;
	```

	- 解决方法

	```objc
	__weak typeof (self) weakSelf = self;
	self.myBlock = ^{
	weakSelf.objc = ...;
	};
```
	
- 那么如果`block`内使用了`self`这个时候如果某一个时刻`self`被释放就会导致出现问题。

- 解决方法
	
```objc
	__weak typeof (self) weakSelf = self;
	self.myBlock = ^{
	__strong typeof(self) strongSelf = weakSelf;
	strongSelf.objc1 = ...;
	strongSelf.objc2 = ...;
	strongSelf.objc3 = ...;
	};
```
	
使用`__weak`打破循环引用。`__strong`用来避免在使用`self`过程中`self`被释放，`__strong`在`block`后会调用`objc_release(obj)`释放对象。
	
```objc
id __strong obj = [[NSObject alloc] init];
	
// clang 编译后
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);

```

两次调用`objc_msgSend`并在变量作用域结束时调用`objc_release`释放对象，不会出现循环引用问题。

#### NSTimer循环引用

为什么`NSTimer`会导致循环引用呢？

```objc
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti 
									  target:(id)aTarget 
									  selector:(SEL)aSelector 
									  userInfo:(nullable id)userInfo 
									  repeats:(BOOL)yesOrNo;
									  
									  
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti 
										        target:(id)aTarget 
										        selector:(SEL)aSelector 
										        userInfo:(nullable id)userInfo 
										        repeats:(BOOL)yesOrNo;
```

- 主要是因为`NSRunloop`运行循环保持了对`NSTimer`的强引用，并且`NSTimer`的`targer`也使用了强引用。

- 来自文档[NSTimer](https://developer.apple.com/reference/foundation/nstimer?language=objc)

>Note in particular that run loops maintain strong references to their timers, so you don’t have to maintain your own strong reference to a timer after you have added it to a run loop.
>**target**
The object to which to send the message specified by aSelector when the timer fires. The timer maintains a strong reference to this object until it (the timer) is invalidated.

举个🌰：

```objc

@interface ViewController ()<viewControllerDelegate>

@property (strong, nonatomic) NSTimer *timer;

@end

 - (void)viewDidLoad
 {
     [super viewDidLoad];
     self.timer = [NSTimer scheduledTimerWithTimeInterval:1
                                               target:self 
                                             selector:@selector(onTimeOut:) 
                                             userInfo:nil 
                                              repeats:NO];
 }
```

- 这里控制器强引用了`timer`，而`timer`也强引用了控制器,这个时候就是循环引用了，引用关系如下图：

```sequence
timer->控制器:strong
控制器->timer:strong
```

- 那么如果控制器对`timer`使用了`weak`呢？
使用`weak`是打破了循环引用,但是`run loop`还是强引用着`timer`,`timer`又强引用着控制器，所以还是会导致内存泄漏。引用关系如下图：

```sequence
runloop->timer:strong
timer->控制器:strong
控制器-->timer:weak
```

如果我们把`timer`加入主线程的`runloop`,主线程中的`runloop`生命周期只有主线程结束才会销毁，所以我们不主动调用`[timer invalidate]`,`runloop`会一直持有`timer`，`timer`又持有控制器，那么就一直不会释放控制器。

- 解决方法：手动调用`[timer invalidate]`来解除持有关系，释放内存。可能会想到在`dealloc`方法中来手动调用，但是因为`timer`持有控制器，所以控制器的`dealloc`方法永远不会调用，因为`dealloc`是在控制器要被释放前调用的。在[Timer Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Timers/Articles/usingTimers.html)中有特别说明。所以一般我们可以在下面这些方法中手动调用`[timer invalidate]`然后置为`nil`：

```objc
- (void)viewWillDisappear:(BOOL)animated; // Called when the view is dismissed, covered or otherwise hidden. Default does nothing
- (void)viewDidDisappear:(BOOL)animated;  // Called after the view was dismissed, covered or otherwise hidden. Default does nothing
```

>A timer maintains a strong reference to its target. This means that as long as a timer remains valid, its target will not be deallocated. As a corollary, this means that it does not make sense for a timer’s target to try to invalidate the timer in its dealloc method—the dealloc method will not be invoked as long as the timer is valid.



## \# 参考资料

- [About Memory Management](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
- [Transitioning to ARC Release Notes](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)
- [NSTimer](https://developer.apple.com/reference/foundation/nstimer?language=objc)
- [Timer Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Timers/Articles/usingTimers.html)
- [《Effective Objective-C 2.0 编写高质量iOS与OS X代码的52个有效方法》](https://book.douban.com/subject/25829244/)
- [《objc高级编程:iOS与OS X多线程和内存管理》](https://book.douban.com/subject/24720270/)


