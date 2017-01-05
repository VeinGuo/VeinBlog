---
title: autoreleasepool是如何实现的？ 
date: 2016-12-13 09:30:00.000000000 +08:00
tags: objc
---


虽然在ARC时代我们可以完全不知道`Autorelease`就能管理好内存，但在了解`objc`内存管理还是十分重要的，在阅读了书籍和一些干货并动手验证之后，决定总结`autoreleasePool`的实现。

---

## 什么是autorelease

`autoreleasePool`如何实现需要先知道什么是`autorelease`？

`autorelease`类似于C语言中[Automatic variable](https://en.wikipedia.org/wiki/Automatic_variable)自动变量，程序执行时，若某自动变量超出其作用域，该自动变量将被自动废弃。



## autorelease何时释放

面试时提问`objc`内存管理基本都会问到`autorelease`何时释放,在没有使用`@autoreleasepool`的情况,`autorelease`对象是在当前的`runloop`迭代结束时释放。

每个runloop中都会创建一个 `autoreleasepool` 并在runloop迭代结束进行释放。

如果是手动创建`autoreleasepool`,自己创建Pool并释放：

``` objc
// MRC
NSAutoreleasePool *pool = [NSAutoreleasePool alloc] init];

id obj = [NSObject alloc] init];
[obj autorelease];
[pool drain];

// ARC
@autoreleasepool {
		id obj = [NSObject alloc] init];
    }
```

`Apple`文档中提到：
```
@autoreleasepool blocks are more efficient than using an instance of NSAutoreleasePool directly; you can also use them even if you do not use ARC.
```

不管是`MRC`还是`ARC`最好使用@autoreleasepool blocks。
<!-- more -->
## @autoreleasepool

上面提到的使用`@autoreleasepool`来手动创建并释放`autorelease`
`@autoreleasepool` 使用`clang`编译之后

``` C++
 /* @autoreleasepool */ 
{ 
	__AtAutoreleasePool __autoreleasepool; 
}
```

`@autoreleasepool`被转转换成`__AtAutoreleasePool` 结构体类型

``` C++
struct __AtAutoreleasePool 
{
  __AtAutoreleasePool() 
  {
		atautoreleasepoolobj = objc_autoreleasePoolPush();
  }
  ~__AtAutoreleasePool() 
  {
  		objc_autoreleasePoolPop(atautoreleasepoolobj);
  }
  void * atautoreleasepoolobj;
};
```

可以看到 `__AtAutoreleasePool()` 构造函数调用`objc_autoreleasePoolPush()`,`~__AtAutoreleasePool()` 析构函数调用 `objc_autoreleasePoolPop()`

在 `MRC` 中我们使用 `NSAutoreleasePool` 来创建AutoreleasePool,那么相应的实现如下：

```objc
NSAutoreleasePool *pool = [NSAutoreleasePool alloc] init];
// 相当于调用构造函数也就是 objc_autoreleasePoolPush();
[pool drain];
// 相当于调用析构函数也就是 objc_autoreleasePoolPop(pool);
```

`objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop` 是什么呢？

`objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop` 实现需要从[runtime](https://opensource.apple.com/tarballs/objc4/)源代码看到,文中的代码的是最新的`objc4-706.tar.gz`

在 `NSObject.mm` 文件中：
![NSObject.mm](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/2.png)

实际上是调用`AutoreleasePoolPage`的`push`和`pop`两个类方法

### AutoreleasePoolPage
首先来看一下`AutoreleasePoolPage`这个类

``` C++
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
```

![AutoreleasePoolPage](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/AutoreleasePoolPage.png)

*	`magic` 检查校验完整性的变量
*	`next`  指向新加入的autrelease对象下一个位置如下图：

![next](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/next.png)


* `thread` page当前所在的线程
* `parent`	父节点 指向前一个page
* `child`	子节点 指向下一个page
* `depth` 链表的深度，节点个数
* `hiwat` high water mark 数据容纳的一个上限

* `EMPTY_POOL_PLACEHOLDER` 空池占位
* `POOL_BOUNDARY` 是一个边界对象 `nil`,之前的源代码变量名是 `POOL_SENTINEL`哨兵对象,用来区别每个`page`即每个 `AutoreleasePoolPage` 边界

* `PAGE_MAX_SIZE` 定义的大小在下图可以看到:

![PAGE_MAX_SIZE](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/PAGE_MAX_SIZE.png)

*	`PAGE_MAX_SIZE` = 4096, 为什么是4096呢？其实就是虚拟内存每个扇区4096个字节,[4K对齐](https://zh.wikipedia.org/zh-hans/4K%E5%AF%B9%E9%BD%90)的说法。
* `COUNT` 一个`page`里对象数

**在自动释放池中每一个`AutoreleasePoolPage`都是以双链表的形式连接起来的：**

![Pool](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/Pool.png)

**`parent` 指向前一个 `page` , `child` 指向下一个 `page`**

## push
![PUSH](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/push.png)
每当自动释放池调用`objc_autoreleasePoolPush`时都会把边界对象放进栈顶,然后返回边界对象,用于释放。
``` C++
atautoreleasepoolobj = objc_autoreleasePoolPush();
```
`atautoreleasepoolobj` 就是返回的边界对象

`push`就是压栈的操作,先加入边界对象然后添加A对象在边界对象之后,下一个B对象压入A对象之后,就像羽毛球筒放羽毛球一样

## pop
![POP](http://ojaltanzc.bkt.clouddn.com/2016-12-13-aotureleasepool/pop.png)
自动释放池释放是传入 `push` 返回的边界对象,

``` C++
objc_autoreleasePoolPop(atautoreleasepoolobj);
```

然后将边界对象指向的这一页 `AutoreleasePoolPage` 内的对象释放

##  @End
 **总结：**
 1. 自动释放池是一个个 `AutoreleasePoolPage` 组成的一个`page`是4096字节大小,每个 `AutoreleasePoolPage` 以双向链表连接起来形成一个自动释放池
 2. `pop` 时是传入边界对象,然后对`page` 中的对象发送`release` 的消息

文章有什么理解不是很到位的希望指出

## 参考资料 

*	[《objc高级编程:iOS与OS X多线程和内存管理》](https://book.douban.com/subject/24720270/)
* [自动释放池的前世今生 ---- 深入解析 autoreleasepool](http://draveness.me/autoreleasepool/)
* [NSAutoreleasePool](https://developer.apple.com/reference/foundation/nsautoreleasepool#//apple_ref/occ/cl/NSAutoreleasePool)
* [黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)



<!— more -->

