---
author: wukn
catalog: true
date: 2016-11-12T06:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Extending observables'
url: /2016/11/12/knockoutjs-extending-observables/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## 创建扩展器

向`ko.extenders`对象添加一个方法来创建一个扩展器。添加的方法有两个参数，第一个参数监控属性自身，第二个参数是任意的参数。它可以返回监控属性本身，或者返回使用原监控属性的计算监控属性。

例如，定义一个扩展器`logChange`，订阅监控属性，使用配置的信息在控制台中输出日志：

```js
ko.extenders.logChange = function(target, option) {
    target.subscribe(function(newValue) {
       console.log(option + ": " + newValue);
    });
    return target;
};
```

使用该扩展器时，调用监控属性的`extend`方法，传入一个包含`logChange`属性的对象作为参数。

```js
this.firstName = ko.observable("Bob").extend({logChange: "first name"});
```

### 例1：强制输入为数字

```html
<p><input data-bind="value: myNumberOne" /> (round to whole number)</p>
<p><input data-bind="value: myNumberTwo" /> (round to two decimals)</p>
```

```js
ko.extenders.numeric = function(target, precision) {
    //create a writable computed observable to intercept writes to our observable
    var result = ko.pureComputed({
        read: target,  //always return the original observables value
        write: function(newValue) {
            var current = target(),
                roundingMultiplier = Math.pow(10, precision),
                newValueAsNum = isNaN(newValue) ? 0 : parseFloat(+newValue),
                valueToWrite = Math.round(newValueAsNum * roundingMultiplier) / roundingMultiplier;

            //only write if it changed
            if (valueToWrite !== current) {
                target(valueToWrite);
            } else {
                //if the rounded value is the same, but a different value was written, force a notification for the current field
                if (newValue !== current) {
                    target.notifySubscribers(valueToWrite);
                }
            }
        }
    }).extend({ notify: 'always' });

    //initialize with current value to make sure it is rounded appropriately
    result(target());

    //return the new computed observable
    return result;
};

function AppViewModel(one, two) {
    this.myNumberOne = ko.observable(one).extend({ numeric: 0 });
    this.myNumberTwo = ko.observable(two).extend({ numeric: 2 });
}

ko.applyBindings(new AppViewModel(221.2234, 123.4525));
```

注意，为了自动清除界面上的格式不对的值，需要对计算监控属性调用`extend({ notify: 'always' })`，否则可能会出现用户输入了无效的`newValue`后在截断时返回了未改变的`valueToWrite`。然后，由于模型值没变，就不会通知文本框更新。使用`extend({ notify: 'always' })`会强制文本框更新值，即使计算属性的值没变。

### 例2：为监控属性添加验证

本例创建一个可以验证监控属性的扩展器。不同于返回新对象，该扩展器只是为已有的监控属性添加额外的子监控属性。由于监控属性是函数，它们可以有自己的属性。然而当视图模型转换为json时，子监控属性会被忽略。这样能很方便地添加只跟UI相关不需要与服务器交互的额外功能。

```html
<p data-bind="css: { error: firstName.hasError }">
    <input data-bind='value: firstName, valueUpdate: "afterkeydown"' />
    <span data-bind='visible: firstName.hasError, text: firstName.validationMessage'> </span>
</p>
<p data-bind="css: { error: lastName.hasError }">
    <input data-bind='value: lastName, valueUpdate: "afterkeydown"' />
    <span data-bind='visible: lastName.hasError, text: lastName.validationMessage'> </span>
</p>
```

```js
ko.extenders.required = function(target, overrideMessage) {
    //add some sub-observables to our observable
    target.hasError = ko.observable();
    target.validationMessage = ko.observable();

    //define a function to do validation
    function validate(newValue) {
       target.hasError(newValue ? false : true);
       target.validationMessage(newValue ? "" : overrideMessage || "This field is required");
    }

    //initial validation
    validate(target());

    //validate whenever the value changes
    target.subscribe(validate);

    //return the original observable
    return target;
};

function AppViewModel(first, last) {
    this.firstName = ko.observable(first).extend({ required: "Please enter a first name" });
    this.lastName = ko.observable(last).extend({ required: "" });
}

ko.applyBindings(new AppViewModel("Bob","Smith"));
```

## 同时应用多个扩展器

在一次调用`extend`方法时可以同时应用多个扩展器。

```js
this.firstName = ko.observable(first).extend({ required: "Please enter a first name", logChange: "first name" });
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/extenders.html)
