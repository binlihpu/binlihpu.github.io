---
layout: post
title: Git建立新的分支
categories: GIT
description: git的一些常用用法 
keywords: Git
---

比如你要从现在的master分支切出一个新分支dev


- 首先切换到master分支：
```go
$ git checkout master
```
- 然后先创建本地分支dev：
```go
$ git checkout -b dev
```
- 然后将本地分支dev推到远端：
```go
$ git push origin dev
```
- 最后将本地的dev分支和远端的dev分支关联起来：
```go
$ git branch --set-upstream-to=origin/dev dev
```
