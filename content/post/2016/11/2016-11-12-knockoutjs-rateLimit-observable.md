---
author: wukn
catalog: true
date: 2016-11-12T08:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Rate-limiting observable notification'
url: /2016/11/12/knockoutjs-rateLimit-observable/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## 延时通知(rate-limiting notification)

通常监控属性在变化时是实时通知订阅者的，以便依赖它的计算监控属性和绑定能实时同步更新。而`rateLimit`扩展器是为了将变化通知阻止并延迟指定时间。延时监控属性是异步通知依赖者的。

`rateLimit`扩展器可用于任何监控属性，包括计算监控属性和监控数组。通常使用它的原因是：
* 使响应延迟一段时间。
* 将多个变化合并到一次更新中。

如果只是要合并更新而不需要延时，使用deferred监控效率更高。

### 使用rateLimit扩展器

`rateLimit`有两种参数格式：

```js
// Shorthand: Specify just a timeout in milliseconds
someObservableOrComputed.extend({ rateLimit: 500 });

// Longhand: Specify timeout and/or method
someObservableOrComputed.extend({ rateLimit: { timeout: 500, method: "notifyWhenChangesStop" } });
```

`method`选项是在通知发起时触发，有两个参数：

* `notifyAtFixedRate`：没有指定method选项时的默认值。从监控属性第一次改变开始（初始化时或上一次更新后），在指定时间后开始通知。
* `notifyWhenChangesStop`：监控属性在指定时间后没有变化时开始通知。监控属性每次变化时，定时器就会被更新。因此如果监控属性更新频率比指定时间还短时，就一直不会发出通知。

### 例1：基本的示例

```js
var name = ko.observable('Bert');

var upperCaseName = ko.computed(function() {
    return name().toUpperCase();
});
```

通常会这样改变name的值：

```js
name('The New Bert');
```

这时upperCaseName会立即更新。而如果使用了rateLimit：
```javascript
var name = ko.observable('Bert').extend({ rateLimit: 500 });
```

那么upperCaseName不会在name变化后立即重新计算，name会在值变化后等待500ms再将新的值通知给upperCaseName。不管在这500ms内name的值变化了多少次，upperCaseName只会收到一次最新值得通知。

### 例2：在用户停止输入时执行一些逻辑

在本例中，监控属性`instantaneousValue`是用户按下按键后的实时输入值，计算监控属性`delayedValue`配置为在`instantaneousValue`停止变化400ms后被通知。

```html
<p>Type stuff here: <input data-bind='textInput: instantaneousValue' /></p>
<p>Current delayed value: <b data-bind='text: delayedValue'> </b></p>

<div data-bind="visible: loggedValues().length > 0">
    <h3>Stuff you have typed:</h3>
    <ul data-bind="foreach: loggedValues">
        <li data-bind="text: $data"></li>
    </ul>
</div>
```

```js
function AppViewModel() {
    this.instantaneousValue = ko.observable();
    this.delayedValue = ko.pureComputed(this.instantaneousValue)
        .extend({ rateLimit: { method: "notifyWhenChangesStop", timeout: 400 } });

    // Keep a log of the throttled values
    this.loggedValues = ko.observableArray([]);
    this.delayedValue.subscribe(function (val) {
        if (val !== '')
            this.loggedValues.push(val);
    }, this);
}

ko.applyBindings(new AppViewModel());
```

### 计算监控属性的特殊情况

对于计算监控属性，rateLimit定时器是在它依赖的监控属性其中一个变化时开始触发，而不是在计算监控属性的值变化时。计算监控属性只会在需要它的值时才重新评估——延时已到应该发出通知了，或者当它的值被直接访问。如果需要计算监控属性的最新值，可以使用`peek`方法。

### 强制延时的监控属性始终通知订阅者

当监控属性的值是原始类型时（数字、字符串、bool值、null），监控属性只会在设置的值与之前的值不一样时才通知依赖者。因此，原始值得监控属性延时更新时，只会在延时timeout时，值的确不一样了才发出通知。也就是说如果原始类型监控属性延时更新，在延时timeout之前值变化了又变回原来的值，是不会通知更新的。

为了保证订阅者始终能收更新到通知，即使值一样，可以使用notify扩展器：

```js
myViewModel.fullName = ko.computed(function() {
    return myViewModel.firstName() + " " + myViewModel.lastName();
}).extend({ notify: 'always', rateLimit: 500 });
```

### 与deferred的比较

deferred与rateLimit相似，deferred是通过异步通知和更新的来达到延迟目的的。但defferred用的不是延时的方式，而是在当前任务结束后立刻更新。在使用ko v3.4.0版本之后可以用deferred来代替rateLimit：

```js
ko.computed(function() {
    // ....
}).extend({ deferred: true });
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/rateLimit-observable.html)
