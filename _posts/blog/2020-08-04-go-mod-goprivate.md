---
layout: post
title: Go module 使用私有库配置
categories: Go
description: 主要配置GOPRIVATE字段和gitconfig文件
keywords: Go
---

# Go module 使用私有库配置
本次配置Go版本是1.14.4，Git版本2.24.3,操作系统是macOS。

由于GOPROXY代理的是开源项目，可以不用授权就能`go get`下来，所以国内一般配置GOPROXY参数为：
`export GOPROXY=https://goproxy.cn`
这样配置在国内拉取依赖会快很多，GOPROXY不配置的，默认`https://goproxy.io`,这个是在国外，所以速度会很慢。
如果你的依赖项目是在自己域名下部署的（`github.com`,`gitlab.com`同理），例如，我的项目下依赖`git.abc,com/app/golibs.git`这个项目，那么我应该配置如下，我的环境变量在`.bash_profile`文件中，我用`vim`编辑这个文件，增加全局变量GOPRIVATE：
`export GOPRIVATE=git.abc.com`
这样配置后，保存`.bash_profile`文件，执行`source .bash_profile`,把刚才添加的变量应用到全局中。这样你项目中所有依赖`git.abc.com`前缀的，都不通过代理拉取。

另外，这样配置后拉取`git.abc.com`的项目依赖也可能报错：主要是go module默认拉取远端的的方式是`https`的方式，在本地开发环境中，我们一般用`ssh`方式拉取代码。这个需要将本地生成的ssh密钥配置到远端。同时需要在`.gitconfig`文件下添加：

```shell
[url "ssh://git@git.abc.com:"]
        insteadOf = https://git.abc.com/
```

这样配置就可以了，再去项目下`go build`会自动拉取依赖，也不会报错了。
