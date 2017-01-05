---
title: Learn Swift - 1 (基础类型)
date: 2016-09-25 19:38:42.000000000 +08:00
tags: Learn Swift
---

* * *
&nbsp;  Apple 发布了Swift 3.0,版本语法应该相对稳定了，开始新一轮学习。

&nbsp;  Swift学习主要从The Basics开始学习，虽然之前已经学习过Objective-C,但还是过一遍基本的语法。


* * *
>`Swift` provides its own versions of all fundamental `C` and `Objective-C` types, including `Int` for integers, Doubleand `Float` for floating-point values, `Bool` for Boolean values, and `String` for textual data. `Swift` also provides powerful versions of the three primary collection types, `Array`, `Set`, and `Dictionary`

<!-- more -->

### Naming Constants and Variables（常量与变量）


1. 常量使用 ```let``` 
2. 变量使用 ```var```


```swift
let maximumNuberOfLoginAttempts = 10
var currentLoginAttempts = 0
```

### Printing Constants and Variables(类型标注)


类型和大部分变成语法相同有

```swift
var welcomeMessage: String

welcomeMessage = "Hi big brother"

var red, green, blue: Double

print("你好,大兄弟。\(welcomeMessage)")

let cat = "🐱"; print(cat)
```
   
### Type Safety and Type Inference(类型安全和类型推断)


```swift
let minValue = UInt8.min
let maxValue = UInt8.max


let meaningOfLife = 42
//  自动推断为 Int 类型

let pi = 3.1415926
// 推断为 Double 类型

let anotherPi = 3 + 0.14159
// Double 类型
```
```swift
// 17

let decimalInteger = 17
let binaryInteger = 0b10001
let octalInteger = 0o21
let hexadeimalInteger = 0x11
```

#### Numeric Type Conversion(数字类型转换)


```swift
// 整数转换
// let cannotBeNegative: UInt8 = -1
// UInt8 类型不能存储负数

// let tooBig: Int8 = Int8.max + 1
// Int8 类型不能存储超过最大值的数，所以会报错

let twoThousand: UInt16 = 2_000  // 这里2_000 与 2000 相同
let one: UInt8 = 1
let twoThousandAndOne = twoThousand + UInt16(one)

// UInt16 与 UInt8 整数类型相加 由于存储不同范围的值，所以必须不同情况选择性不同数值类型转换。
// UInt8 + UInt16    先把 UInt8 -> UInt16 后再进行运算

/*
 SomeType(ofInitialValue) 是调用 Swift 构造器并传入一个初始值的默认方法。
 在语言内部，UInt16 有一个构造器，可以接受一个UInt8类型的值，所以这个构造器可以用现有的 UInt8 来创建一个新的 UInt16。
 注意，你并不能传入任意类型的值，只能传入 UInt16 内部有对应构造器的值。
*/
```

### Type Aliases(类型别名)


```swift
// Type Aliases

typealias Vein = UInt16

var maxVein = Vein.max
```

### Booleans(布尔类型)


```swift
// Booleans

let orangesAreOrange = true
let turnipsAreDelicious = false

if turnipsAreDelicious {
    print("1")
}else {
    print("2")
}
```
```swift
// this example will not compile, and will report an error
let i = 1
if 1 {
}

let i = 1
if i == 1 {
    // this example will compile successfully
}

// i == 1 comparison is of type Bool

```
----

元祖类型是我在OC中没有接触的类型，我将在下一篇学习笔记中完整的学习和分享在实际使用中的一些例子。

### 参考

[The Swift Programming Language 中文版](http://wiki.jikexueyuan.com/project/swift/)

[The Swift Programming Language (Swift 3)](https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0)



