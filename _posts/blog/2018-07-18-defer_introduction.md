---
layout: post
title: defer入门
categories: GO
description: defer的用法
keywords: 入门
---

## 1.Defer的定义
> defer:英文意思是推迟，延迟。
所以defer语句表示推迟函数的执行，直到函数执行完毕返回为止。特别要提示是defer会在return语句之后执行。
```
package main

import "fmt"

func main() {
	test()
}

func test() (int, error) {
	defer fmt.Println("world")
	fmt.Println("hello")
	return fmt.Println("123")
}
```
执行结果：
```
hello
123
world
```
> 从结果来看，在test()函数执行的顺序defer语句是在return语句之后执行的，也就是说defer是在整个函数执行完之后才会调用

## 2. defer使用情景

#### 2.1 关闭文件
> Go语言中延迟（defer）语句是种不错的设计，你可以在函数中添加多个defer语句。当函数执行到最后时，这些defer语句会按照逆序执行，最后该函数返回。特别是当你在进行一些打开资源的操作时，遇到错误需要提前返回，在返回前你需要关闭相应的资源，不然很容易造成资源泄露等问题。

```
func ReadWrite() bool {
	file.Open("file")
	//不管读写文件中出现什么错误，
	//只要函数执行完了，
	//就会调用defer关闭文件
	defer file.Close() 
	if failureX {
		return false
	}
	if failureY {
		return false
	}
	return true
}
```
#### 2.2 异常捕捉
> Go中可以抛出一个panic的异常，然后在defer中通过recover捕获这个异常，然后正常处理。

```
package main

import "fmt"

func main() {

	defer func() { // 必须要先声明defer，否则不能捕获到panic异常

		fmt.Println("c")

		if err := recover(); err != nil {

			fmt.Println(err) // 这里的err其实就是panic传入的内容，55

		}

		fmt.Println("d")

	}()

	f()

}

func f() {

	fmt.Println("a")

	panic(55)

	fmt.Println("b")

	fmt.Println("f")

}

```
> 执行结果：

```
a
c
55
d
```