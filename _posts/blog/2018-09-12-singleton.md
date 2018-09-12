---
layout: post
title: Go设计模式之单例模式
categories: 设计模式
description: Go版本单例模式
keywords: 设计模式
---

#### 简介
单例模式是使用最多的设计模式之一，在工作和面试中也常常提到这个设置模式。单例模式可以保证系统中，应用该模式的类一个类只有一个实例。即一个类只有一个对象实例。

#### 代码示例
`Go`的单例模式中有一个地方比较有趣，就是使用了`sync.Once`来控制具体类的实例化只会被创建一次。

新建`singleton/singleton.go`：

```go
package singleton

type singleton map[string]string

var (
    once sync.Once

    instance singleton
)

func New() singleton {
    once.Do(func() {
        instance = make(singleton)
    })

    return instance
}
```

然后新建`main.go`：

```go
package main

import (
	"fmt"
	"helloworld/singleton"
)

func main() {
	//实现单例singleton
	s := singleton.New()

	s["one"] = "one"
	//再次调用不会创建新的实例
	s2 := singleton.New()

	fmt.Println("This is ", s2["one"])
}
```

执行`go run main.go`,控制台打印信息：

```go
$ go run main.go
This is  one
```
需要注意的是，`sync.Once`可以保证赋值操作只会执行一次，这个过程还是线程安全的。具体的实现机制可以看下源码：

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package sync

import (
	"sync/atomic"
)

// Once is an object that will perform exactly one action.
type Once struct {
	m    Mutex
	done uint32
}

// Do calls the function f vif and only if Do is being called for the
// first time for this instance of Once. In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

```

从上面源码可以看出使用`sync/atomic`和`Mutex `进行加锁的原子操作来保证并发安全的。

每次调用`Do`函数都会检测`done`的值是否是`1`，如果不是`1`,将会对`done`值进行操作，第`40`行使用了`Mutex `加锁，锁的类型是互斥锁，第`42`行代码展示如果`done`等于`0`时就会导致`done`的值`+1`。

#### 应用场景
* 系统只需要一个实例对象，如系统要求提供一个唯一的序列号生成器或资源管理器，或者需要考虑资源消耗太大而只允许创建一个对象。
* 客户调用类的单个实例只允许使用一个公共访问点，除了该公共访问点，不能通过其他途径访问该实例。
* 由于配置文件一般都是共享资源，一般采用单例模式来实现。如：游戏服务端的配置文件的读取等。
* 创建对象时耗时过多或者耗资源过多，但又经常用到的对象。
* 数据库连接池的设计一般也是采用单例模式，因为数据库连接是一种数据库资源。数据库软件系统中使v用数据库连接池，主要是节省打开或者关闭数据库连接所引起的效率损耗，这种效率上的损耗还是非常昂贵的，因为何用单例模式来维护，就可以大大降低这种损耗。
* 多线程的线程池的设计一般也是采用单例模式，这是由于线程池要方便对池中的线程进行控制。
* 游戏客户端常常使用单例模式控制场景切换等减少唯一资源频繁创建消耗资源。
* 游戏中区服的计数器之类的，比如全服团购次数。
