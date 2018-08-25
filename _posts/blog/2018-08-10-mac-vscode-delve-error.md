---
layout: post
title: Mac下使用vscode的delve调试Go报错
categories: 开发心得
description: 踩过的坑，记录一下
keywords: 随记, 开发心得
---

 在``` mac ```下使用``` vscode ```开发``` Go ```程序时使用``` delve ```来``` debug ```如果报下面的错误：
```
could not launch process: exec: "lldb-server": executable file not found in $PATH
Process exiting with code: 1
```
可以直接打开终端（```Terminal```）输入:
``` $ xcode-select --install ```

输出：
``` xcode-select: note: install requested for command line developer tools ```
系统会自动更新软件，并弹出

![avatar]({{site.url}}/images/blog/mac-vscode-delve-01.png)

等待更新完毕后即可！


