---
layout: post
title: MongoDB基础用法及查询操作
categories: MongoDB
description: 数据库
keywords: MongoDB, 开发
---

## MongoDB基础
###### MongoDB中和我们常见的Mysql数据库类似，可以做个简单的对比快速入门：
传统的关系数据库一般由数据库（database）、表（table）、记录（record）三个层次概念组成，

MongoDB是由数据库（database）、集合（collection）、文档对象（document）三个层次组成。

MongoDB对于关系型数据库里的表，但是集合中没有列、行和关系概念，这体现了模式自由的特点。

###### 插入多条测试数据
```
for(i=1;i<=1000;i++){
    db.blog.insert({"title":i,"content":"内容"+i,"name":"admin"});
}
```
```sql
db.blog.list.find().limit(10).forEach(function(data){print("title:"+data.title);}) //循环forEach 用法
db.blog.findOne(); //取一条数据
db.blog.find(); //取多条数据
db.blog.remove(); //删除数据集
db.blog.drop(); //删除表
```
删除一个数据库：
```javascript
1.use dbname
2.db.dropDatabase()
```