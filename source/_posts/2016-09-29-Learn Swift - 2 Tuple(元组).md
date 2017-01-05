---
title: Learn Swift - 2 (Tuple)
date: 2016-09-29 23:52:00.000000000 +08:00
tags: Learn Swift
---

### Tuple(元组)

>&nbsp; Tuples group multiple values into a single compound value. The values within a tuple can be of any type and do not have to be of the same type as each other.
>&nbsp; 元组（tuples）把多个值组合成一个复合值。元组内的值可以是任意类型，并不要求是相同类型。

* * *

#### Example
```swift
let person = ("Vein", 66)
```

&nbsp; 上面这个例子中，`（“Vein”, 66）` 是一个人的姓名和体重的元组。
`("Vein", 66)` 元组把一个`String` 和一个`Int` 两个不同的类型组合起来。

你也可以随意组合不同类型如：（`Int`, `Int`, `Double`、(`String`, `Bool`)或者（`String`, `String`, `String`)等等其他任何你想组合的元组。

<!-- more -->

##### 分解（decompose）
你可以分解元组的内容，变成单独的常量和变量，然后你可以拿来直接使用。

```swift
let (name, weight) = person
print("姓名:\(name)")
print("体重:\(weight)")
```

过滤掉某一部分的话可以使用`（_, weight）`的方法，这样只取出元组中的weight:

```swift
var (_ ,weight) = person
print("体重:\(weight)")
```

你也可以像取数组中元素一样，使用下标获取元组中的元素，下标从`0`开始：

```swift
print("姓名:\(person.0)")
print("体重:\(person.1)")
```

当然你也可以在定义元组时就对单个元素命名并用命名来获取这些元素的值：

```swift
let person = (name: "Vein", weight: 66)
print("姓名:\(person.name)")
print("体重:\(person.weight)")
```

#### 元组实际的应用例子

```swift
func maxValueAndMinValue ( values:[Int] ) -> (max:Int, min:Int)    {
    
    var maxValue = values[0]
    var minValue = values[0]
    
    for value in values
    {
        maxValue = max(maxValue, value)
        minValue = min(minValue, value)
    }
    
    return (maxValue, minValue)
}
maxValueAndMinValue(values: [1,2,3,4,5,6,7])
```

上面这个函数使用元组返回两个值，一个`max`和一个`min`


>&nbsp; Tuples are particularly useful as the return values of functions. A function that tries to retrieve a web page might return the (Int, String) tuple type to describe the success or failure of the page retrieval. By returning a tuple with two distinct values, each of a different type, the function provides more useful information about its outcome than if it could only return a single value of a single type. For more information, see Functions with Multiple Return Values.
>&nbsp; 作为函数返回值时，元组非常有用。一个用来获取网页的函数可能会返回一个 (Int, String) 元组来描述是否获取成功。和只能返回一个类型的值比较起来，一个包含两个不同类型值的元组可以让函数的返回信息更有用。请参考函数参数与返回值。

### NOTE(注意)
>&nbsp; Tuples are useful for temporary groups of related values. They are not suited to the creation of complex data structures. If your data structure is likely to persist beyond a temporary scope, model it as a class or structure, rather than as a tuple. For more information, see Classes and Structures.
>&nbsp; 元组在临时组织值的时候很有用，但是并不适合创建复杂的数据结构。如果你的数据结构并不是临时使用，请使用类或者结构体而不是元组。请参考类和结构体。

----

### 参考

[The Swift Programming Language 中文版](http://wiki.jikexueyuan.com/project/swift/)

[The Swift Programming Language (Swift 3)](https://developer.apple.com/library/prerelease/content/documentation/Swift/Conceptual/Swift_Programming_Language/index.html#//apple_ref/doc/uid/TP40014097-CH3-ID0)


<!— more -->

