---
layout: post
title: Go中的Defer, Panic和 Recover的用法
categories: GO
description: Go的异常处理
keywords: Defer, Panic, Recover
---

---
> Go具有控制流程的常用机制：if，for，switch，goto。它还有go语句在单独的goroutine中运行代码。在这里，我想讨论一些不太常见的问题：defer，panic和recover。

defer语句将函数调用推送到列表（栈）中。在周围函数返回后执行已保存调用的列表（栈）。defer通常用于简化执行各种清理操作的功能。

例如，让我们看一个打开两个文件并将一个文件的内容复制到另一个文件的函数：
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }

    written, err = io.Copy(dst, src)
    dst.Close()
    src.Close()
    return
}
```
上面程序看起来没什么问题，但有一个bug。如果对os.Create的调用失败，该函数将返回而不关闭源文件。这可以通过在第二个return语句之前调用src.Close来轻松解决，但如果函数更复杂，则问题可能不会那么容易被注意到并解决。通过引入defer语句，我们可以确保文件在函数执行的最后关闭：
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```
defer语句允许我们考虑在打开它之后立即关闭每个文件，保证无论函数中的返回语句数量如何，文件都将被关闭。
defer语句的行为是直截了当且可预测的。有三个简单的规则：
1. 在评估defer语句时，将评估defer函数的参数。
在此示例中，在延迟Println调用时计算表达式“i”。函数返回后，延迟调用将打印“0”：
```go
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```
2. 在周围函数返回后，延迟函数调用以"后进先出"顺序执行。此功能打印“3210”：
```go
func b() {
    for i := 0; i < 4; i++ {
        defer fmt.Print(i)
    }
}
```
3. 延迟函数可以读取并分配给返回函数的命名返回值：
```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}
```
这样便于修改函数的错误返回值;我们很快就会看到一个例子。

Panic是一种内置功能，可以阻止正常的控制流并出发程序异常。当函数F调用panic时，F的执行停止，F中的任何defer函数都正常执行，然后F返回其调用者。对于呼叫者，F然后表现得像是对恐慌的呼唤。进程继续向上移动，直到当前goroutine中的所有函数都返回，此时程序崩溃。可以通过直接调用panic来启动panic。它们也可能由运行时错误引起，例如数组越界访问。
Recover是一个内置函数，可以在defer函数内重新控制panic的goroutine。在正常执行期间，对recover的调用将返回nil并且没有其他效果。如果当前goroutine处于panic状态，则对恢复的调用将捕获给予panic的值并恢复正常执行。此时程序不会因此down掉。
下面是一个示例程序，演示了panic和defer的机制：
```go
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```
函数g取int i，如果i大于3则panic，否则它用参数i + 1调用自身。函数f推出一个调用recover并打印恢复值的函数（如果它是非零的）。在阅读之前尝试描绘该程序的输出可能是什么。
该程序将输出：
```go
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```
如果我们从f中删除defer函数，则不会恢复panic并到达goroutine调用堆栈的顶部，从而终止程序。此修改后的程序将输出：
```go
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
panic: 4
 
panic PC=0x2a9cd8
[stack trace omitted]
```
有关panic和recovery的真实示例，请参阅Go标准库中的json包。它使用一组递归函数对JSON编码的数据进行解码。当遇到格式错误的JSON时，解析器调用panic将堆栈展开到顶级函数调用，该函数调用从panic中恢复并返回适当的错误值（请参阅decode.go中的decodeState类型的'error'和'unmarshal'方法）。
Go库中的约定是即使包在内部使用panic，其外部API仍然会显示明确的错误返回值。
defer的其他用法（前面给出的关闭示例：超出文件）包括释放互斥锁：
```go
mu.Lock()
defer mu.Unlock()
```
打印页脚：
```go
printHeader()
defer printFooter()
```
总之，defer语句（带有或没有panic和recover的）为控制流提供了一种不寻常且强大的机制。它可用于模拟由其他编程语言中的专用结构实现的许多功能。
