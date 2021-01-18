---
layout: post
title: Go的随机数
categories: Go
description: 随机数
keywords: Go, 开发
---
###### 在游戏开发中经常用到随机数，使用Go的自带包`math/rand`可以很轻松的获得一个随机整数。但是，如果直接使用将会在我们的程序中获得一个固定的随机数不再变化。所以我们在使用前会加入时间种子，如果不首先设置种子，生成的随机数将在第一次运行时返回相同的数字。

###### 在我们的例子中，我们要产生一个随机数介于两个数字。
```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func random(min int, max int) int {
    return rand.Intn(max-min) + min
}

func main() {
    rand.Seed(time.Now().UnixNano())
    randomNum := random(1000, 2000)
    fmt.Printf("Random Num: %d\n", randomNum)
}
```