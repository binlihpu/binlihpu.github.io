---
layout: post
title: Go slice append 陷阱
categories: Go
description: 
keywords: Go
---

# `Go slice append 陷阱`

`Go`中的`slice`也叫切片，经常用于处理动态数组，简单方便。但是，有时它们的行为并不总是像人们期望的那样。
我猜想，如果您正在阅读此文章，则说明您已经对`Go`有所了解，因此我不会以任何方式介绍它，而我只会关注`slice`。

`Go`中的`slice`是非常有用的`Go`数组抽象。 `slice`和`array`都有类型化的值，其中`array`定义了静态长度，我们通常不会使用`array`，因为`array`基本不能操作什么，而`slice`则可以追加、剪切和一般化操作，而这些都是`array`缺少的。

`slice`最常用的就是使用内置函数`append`:

```go
package main
import "fmt"

func main() {
	// create new slice with few elements:
	s := []int{1,2,3}
	fmt.Printf("slice content: %v\n", s)
	
	// append new element to slice:
	s = append(s, 4)
	fmt.Printf("slice after append: %v\n", s)
}
```

`append`函数第一个参数是`slice`，第二个参数是需要追加的数据，这里需要注意的是`append`返回的是一个全新的`slice`，而不是单纯的在第一个`slice`上拼接一个数据，所以我们一般情况下会将`append`的返回值赋值给原来的`slice`，以此达到在原来的`slice`上追加数据的目的。

是不是看来很简单？
但是我要告诉你，这里有一个陷阱：

```go
package main

import "fmt"

func create(iterations int) []int {
    a := make([]int, 0)
    for i := 0; i < iterations; i++ {
        a = append(a, i)
    }
    return a
}

func main() {
    sliceFromLoop()
    sliceFromLiteral()

}

func sliceFromLoop() {
    fmt.Printf("** NOT working as expected: **\n\n")
    i := create(11)
    fmt.Println("initial slice: ", i)
    j := append(i, 100)
    g := append(i, 101)
    h := append(i, 102)
    fmt.Printf("i: %v\nj: %v\ng: %v\nh:%v\n", i, j, g, h)
}

func sliceFromLiteral() {
    fmt.Printf("\n\n** working as expected: **\n")
    i := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    fmt.Println("initial slice: ", i)
    j := append(i, 100)
    g := append(i, 101)
    h := append(i, 102)
    fmt.Printf("i: %v\nj: %v\ng: %v\nh:%v\n", i, j, g, h)
}
```
你可以试着运行下，看看结果：

```go
** NOT working as expected: **

initial slice:  [0 1 2 3 4 5 6 7 8 9 10]
i: [0 1 2 3 4 5 6 7 8 9 10]
j: [0 1 2 3 4 5 6 7 8 9 10 102]
g: [0 1 2 3 4 5 6 7 8 9 10 102]
h:[0 1 2 3 4 5 6 7 8 9 10 102]


** working as expected: **
initial slice:  [0 1 2 3 4 5 6 7 8 9 10]
i: [0 1 2 3 4 5 6 7 8 9 10]
j: [0 1 2 3 4 5 6 7 8 9 10 100]
g: [0 1 2 3 4 5 6 7 8 9 10 101]
h:[0 1 2 3 4 5 6 7 8 9 10 102]
```

`sliceFromLoop`函数运行后打印的结果和我们预期不一样：我们预期的`j`和`g`最后追加的应该是`100`和`101`，但是这里都是`102`！也就是最后一个`append`的值会修改前面两个`append`的值！

为什么会造成这样的结果呢？

这是因为`Go`中`append`修改的是`slice`中底层的数组，并且新的`slice`也是基于此数组的。这就意味着基于`append`返回的新的`slice`可能会导致难以发现的问题。

怎么解决这种问题呢？

**只用`append`在老的`slice`增加元素，不用来创建新的`slice`**

```go
someSlice = append(someSlice, newElement)
```

如果你有需求必须从老的`slice`创建新的`slice`，那么首先要使用内置函数`copy`函数复制老的`slice`底层的数组数据到新的`slice`，然后在新的`slice`上执行`append`操作：

```go
func copyAndAppend(i []int, vals ...int) []int {
    j := make([]int, len(i), len(i)+len(vals))
    copy(j, i)
    return append(j, vals...)
}
```

或者是创建老的`slice`的**浅拷贝（shallow copy）**:

```go
newSlice := append(T(nil), oldSlice...)
```

这个毕竟常规用法，写了几年的`Go`的生产代码才发现这个问题😂确实不太容易理解，另外就是容易让人产生过误解和困惑。

原文地址（需要翻墙）：https://medium.com/@Jarema./golang-slice-append-gotcha-e9020ff37374
