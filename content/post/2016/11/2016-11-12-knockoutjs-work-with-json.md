---
author: wukn
catalog: true
date: 2016-11-12T05:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] JSON'
url: /2016/11/12/knockoutjs-work-with-json/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## 加载或保存数据

ko不限定使用任何数据加载或保存的技术。最常用是jQuery的ajax方法，如getJSON、post和ajax。

例如，从服务器加载数据：

```js
$.getJSON("/some/url", function(data) {
    // Now use this data to update your view models,
    // and Knockout will update your UI automatically
})
```

保存数据到服务器：

```js
var data = /* Your data in JSON format - see below */;
$.post("/some/url", data, function(returnedData) {
    // This callback is executed if the post was successful     
})
```

除了jQuery也可以用其它的方式加载和保存数据。ko要做的就是：
* 保存时，将视图模型数据转换成简单的json格式。
* 加载时，用接收的数据更新到视图模型。

## 将视图模型数据转换成普通json

视图模型是js对象，因此可以用JSON.stringify（浏览器内置方法）或json2.js库将其序列化为json。但是视图模型中可能包含监控属性、计算监控属性、和监控数组，这些实际上是js函数，在序列化时不希望做一些额外的工作来清理。

ko提供了两个辅助方法：

* `ko.toJS`：复制视图模型对象，将监控属性替换其值，这样可以获取一个包含数据且与ko无关的普通的副本对象。
* `ko.toJSON`：将视图模型数据转换成json字符串。其内部实现是，先对视图模型调用`ko.ToJS`，再对结果调用浏览器的本地序列化方法。对于IE7及更早的版本没有json序列化方法，就需要引用json2.js库。

例如，定义视图模型：

```js
var viewModel = {
    firstName : ko.observable("Bert"),
    lastName : ko.observable("Smith"),
    pets : ko.observableArray(["Cat", "Dog", "Fish"]),
    type : "Customer"
};
viewModel.hasALotOfPets = ko.computed(function() {
    return this.pets().length > 2
}, viewModel)
```

将其转换成json字符串：

```js
var jsonData = ko.toJSON(viewModel);

// Result: jsonData is now a string equal to the following value
// '{"firstName":"Bert","lastName":"Smith","pets":["Cat","Dog","Fish"],"type":"Customer","hasALotOfPets":true}'
```

也可以得到序列化前的普通js对象：

```js
var plainJs = ko.toJS(viewModel);

// Result: plainJS is now a plain JavaScript object in which nothing is observable. It's just data.
// The object is equivalent to the following:
//   {
//      firstName: "Bert",
//      lastName: "Smith",
//      pets: ["Cat","Dog","Fish"],
//      type: "Customer",
//      hasALotOfPets: true
//   }
```

注意，`ko.toJSON`接收的参数是和`JSON.stringify`一样的。例如传递一些额外的参数：

```html
<pre data-bind="text: ko.toJSON($root, null, 2)"></pre>
```

## 使用json更新视图模型

如果从服务器更新了数据并希望更新到视图模型上，最直接的方法是对需要的属性赋值：

```js
// Load and parse the JSON
var someJSON = /* Omitted: fetch it from the server however you want */;
var parsed = JSON.parse(someJSON);

// Update view model properties
viewModel.firstName(parsed.firstName);
viewModel.pets(parsed.pets);
```

除了这种手动更新模型属性的方法，可以使用映射插件来减少代码量。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/json-data.html)
