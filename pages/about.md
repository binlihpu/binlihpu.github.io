---
layout: page
title: About
description: 好记性不如烂笔头
keywords: Bin Li, 李斌
comments: true
menu: 关于
permalink: /about/
---

我是一名游戏后端开发者，

目前开发主要使用Go，

未来计划学习Python和JavaScript，

关注微信小程序的发展，

希望大家以后多多交流。

## 推荐链接

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## 技能标签

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
