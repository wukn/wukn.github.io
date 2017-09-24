---
author: wukn
catalog: true
date: 2016-11-05T00:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Introduction'
url: /2016/11/05/knockoutjs-introduction/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

KnockoutJS是一个使用MVVM模式的前端框架，可实现数据模型和UI之间的自动更新。

## 主要特性

* 依赖跟踪（dependency tracking）：数据模型的变化自动更新到对应的UI上。
* 声明式绑定（declarative binding）：一种简单而直接的方式来关联数据模型和UI之间的对应关系。
* 可扩展性（Trivially extensible）：可轻松地创建自定义的声明绑定。

此外，ko还具有以下优点：

* 纯javascript库：可与任何服务器端和客户端技术同时使用。
* 添加到已有的web项目中并不需要很大的改动。
* 体积小：gzip压缩后仅十几KB。
* 兼容主流浏览器（IE 6+, Firefox 2+, Chrome, Safari等）。
* 行为驱动开发。

## ko与Query的关系

相对于传统的操作DOM元素的方式，jQuery提供了一种便利的操作元素和事件的方式，但是仍需要大量的js代码。ko通过声明数据模型和UI的对应关系，实现两者之间的同步更新。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/introduction.html)
