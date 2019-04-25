---
layout: post
title: Go channel堵塞特性的一点见解
categories: GO
description: Go channel堵塞特性的一点见解
keywords: GO
---

今天我同事问了我一个关于channel堵塞打印信息显示顺序的问题，话不多说，先敬上一段代码：
```go
package main

import "fmt"

func sum(s []int, c chan int) {
        sum := 0
        for _, v := range s {
                sum += v
        }
        c <- sum // 把 sum 发送到通道 c
}

func main() {
        s := []int{7, 2, 8, -9, 4, 0}

        c := make(chan int) //声明并初始化一个无缓冲的chan
        go sum(s[:len(s)/2], c) //第一个goroutine
        go sum(s[len(s)/2:], c) //第二个goroutine
        x, y := <-c, <-c // 从通道 c 中接收

        fmt.Println(x, y, x+y)
}
```
从代码上我们不难预测结果：

第一个gorontine执行后会向chan中发送17（`s[:len(s)/2]`代表s的一个人子切片，其值是`[7,2,8]`）; 

第二个gorontine执行后会向chan中发送-5（`s[len(s)/2:]`代表另外一个s的子切片，其值是`[-9,4,0]`）。

也就是说打印结果会是：`17，-5，12`。

但是出乎意料的是打印了：
```go
$ go run main.go
-5 17 12
```
和我们预想的结果不太一样：为什么不是先打印第一个goroutine的值呢？

有以下几点原因：

首先，因为c是一个无缓冲的chan，也就是我们说的这个channel是堵塞的，也就是说这种类型的channel要求发送数据的goroutine和接受数据的goroutine要同时准备好，才能完成发送和接受操作。如果发送方或者接受方有一个没有准备好，就会导致先执行发送或接受的goroutine阻塞等待。也就都说上述代码中第一个goroutine在发送完数据后，由于没有接收方导致一直阻塞。

其次，由于紧接着执行了第二个goroutine，由于没有接收方，同样也是阻塞等待。本地队列中的两个goroutine本来是按照顺序进行排序的（正常执行顺序），但是由于第一个goroutine阻塞了，调度器会将这个goroutine所在的线程从处理器分离出来，同时创建一个新的线程继续处理第二个goroutine，发现这个goroutine也是阻塞，也将其从处理器上分离出来，由于从`go-1.5`版本之后go默认会为每个可用的物理处理器分配一个逻辑处理器，所以者两个阻塞的goroutine在执行语句：
```go 
 x, y := <-c, <-c // 从通道 c 中接收
```
会有系统调度器通过一定的算法，对于多处理器有可能将两个goroutine同时分配到两个逻辑处理器上，具体输出结果要看执行的快慢了。我们将上述代码稍作修改，来验证这种说法：
```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 把 sum 发送到通道 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int) //声明并初始化一个无缓冲的chan
	for i := 0; i < 10; i++ { //执行10次
		go sum(s[:len(s)/2], c) //第一个goroutine
		go sum(s[len(s)/2:], c) //第二个goroutine
		x, y := <-c, <-c        // 从通道 c 中接收

		fmt.Println(x, y, x+y)
	}
}

```
通过运行上述代码：
```go
$ go run *.go
-5 17 12
-5 17 12
-5 17 12
17 -5 12
-5 17 12
17 -5 12
-5 17 12
-5 17 12
-5 17 12
-5 17 12
```
每次得出的结果有可能不同，既`-5 17 12`也有`17 -5 12`的输出这也正体现了goroutine的并发特性。
通过结果我们可以看出我们可以看出两个goroutine都被从逻辑处理器分离出来过。
