---
layout: post
title: 利用Shadowsocks服务搭建科学上网
categories: 杂记
description: 杂记
keywords: shadowsocks
---

## 特此声明
本篇文章主要目的是为了使用Google查询技术相关知识资料。
## 准备工作
在阿里云上申请一台在国外的云服务器，比如我用的是一台香港阿里云服务器，系统是CentOS。
## 搭建步骤
使用终端登陆云服务器

查看是否联网

```
# ping 114.114.114.114
```

安装Shadowsocks，需要先安装python

```
#cd /home
# curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
# python get-pip.py
# pip install --upgrade pip
# pip install shadowsocks
```

新建Shadowsocks配置文件（多端口/多用户）

```
# cd /etc/
# vi shadowsocks.json
{
  "server":"0.0.0.0",
  "local_address":"127.0.0.1",
  "local_port":1080,
  "port_password":{      
    "10111":"password.",
    "10112":"password.",
    "10113":"password."
  },
  "timeout":300,
  "method":"aes-256-cfb",
  "fast_open":false
}
```

注：只需要关注**port_password**的内容，这里是多端口即多用户；**10111**为端口，**password**为服务器的密码

启动服务

```
# ssserver -c /etc/shadowsocks.json -d start
```

查看服务是否启动

```
# systemctl status shadowsocks -l
```

添加Shadowsocks服务器到自开机启动

```
# cd /etc/systemd/system
# vi shadowsocks.service
[Unit]
Description=Shadowsocks
 
[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json
 
[Install]
WantedBy=multi-user.target
```

设置开机自启动

```
# systemctl enable shadowsocks
```
启动服务
```
# systemctl start shadowsocks
```
查看服务状态
```
# systemctl status shadowsocks -l
```

注：用于重新启动服务
```
#ssserver -c /etc/shadowsocks.json -d restart
```

如已启动Shadowsocks服务，查看上面配置的的端口是否开启
```
# netstat -an|grep 10111
```

开放防火墙的端口或者直接关闭防火墙
开启端口：
```
firewall-cmd --zone=public --add-port=10111/tcp --permanent
firewall-cmd --zone=public --add-port=10112/tcp --permanent
firewall-cmd --zone=public --add-port=10113/tcp --permanent
```
重新载入
```
firewall-cmd --reload
```
查看
```
firewall-cmd --zone=public --query-port=10111/tcp
firewall-cmd --zone=public --query-port=10112/tcp
firewall-cmd --zone=public --query-port=10113/tcp
```

注：删除使用下面的语句
```
firewall-cmd --zone=public --remove-port=80/tcp --permanent
```

或 直接关闭防火墙
```
systemctl stop firewalld.service           #停止firewall
systemctl disable firewalld.service     #禁止firewall开机启动
```

## 注意事项
开放端口配置文件对应的端口，因使用的是阿里云的服务器，需要在阿里云的控制平台开放端口
登陆阿里云网站，进入控制台        --->
点击云服务器ECS，进入服务器控制台，点击要选择的服务器        --->
进入服务器实例列表，找到想要增加端口的实例，点击后面的《更多》        --->
安全组列表，配置规则        --->
添加安全组规则        --->
填写对应的端口，然后保存即可。