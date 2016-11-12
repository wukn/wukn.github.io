---
layout:     post
title:      "[KnockoutJS] Deferred update"
subtitle:   "延迟更新"
date:       2016-11-12 07:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - KnockoutJS

---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

复杂的应用通常包含着大量错综复杂的依赖，更新一个监控属性可能会级联触发很多计算监控属性、手动的订阅、UI绑定更新。这些更新可能是代价昂贵的，一些不必要的中间值被推送给视图，或者导致额外的计算监控属性评估，严重影响效率。即使在简单的应用里，更新关联的多个属性或一个属性更新多次（如填充监控数组）也会导致类似的问题。

为此，可以使用deferred延迟更新来确保计算监控属性和UI绑定只在它们的依赖稳定下来后才更新。即使监控属性会经历多次中间值，只有最后一次值被用于更新它的依赖。为了促进这一点，使用[microtask]()来是所有通知成为异步的、调度的。听起来很像[rate-limiting]()，另一种阻止额外通知的方法，但是延迟更新可以不用添加延迟在整个应用里提供该功能。

标准模式、deferred和rate-limit的通知调度的区别为：
* 标准模式：通知是实时地、同步地发生。一些依赖经常收到中间值得通知。
* deferred：通知是异步发生的，在当前任务结束后通常也是UI重绘之前立刻发生。
* rate-limit：通知在指定的时间后发生（根据浏览器的不同，最小时间为2~10毫秒）。

## 启用deferred更新

默认deferred更新是关闭的。要使用它，必须在初始化视图模型前启用：

```js
ko.options.deferUpdates = true;
```

当deferUpdates启用了，所有的监控属性、计算监控属性和绑定都被设置为延迟更新和通知。在创建基于ko的应用之前启用该特性可以不用担心中间值更新问题了。但是，要注意的是，在已有的应用中启用延迟更新可能会破坏原代码中依赖的同步更新或中间值通知。

### 例：避免UI多次更新

下面的例子是刻意地为了演示deferred更新的功能和对性能的提高。

```html
<!--ko foreach: $root-->
<div class="example">
    <table>
        <tbody data-bind='foreach: data'>
            <tr>
                <td data-bind="text: name"></td>
                <td data-bind="text: position"></td>
                <td data-bind="text: location"></td>
            </tr>
        </tbody>
    </table>
    <button data-bind="click: flipData, text: 'Flip ' + type"></button>
    <div class="time" data-bind="text: (data(), timing() + ' ms')"></div>
</div>
<!--/ko-->
```

```js
function AppViewModel(type) {
    this.type = type;
    this.data = ko.observableArray([
        { name: 'Alfred', position: 'Butler', location: 'London' },
        { name: 'Bruce', position: 'Chairman', location: 'New York' }
    ]);
    this.flipData = function () {
        this.starttime = new Date().getTime();
        var data = this.data();
        for (var i = 0; i < 999; i++) {
            this.data([]);
            this.data(data.reverse());
        }
    }
    this.timing = function () {
        return this.starttime ? new Date().getTime() - this.starttime : 0;
    };
}

ko.options.deferUpdates = true;
var vmDeferred = new AppViewModel('deferred');

ko.options.deferUpdates = false;
var vmStandard = new AppViewModel('standard');

ko.applyBindings([vmStandard, vmDeferred]);
```

## 为指定监控属性使用deferred更新

不希望对整个应用使用deferred更新时，也可以为指定监控属性使用，通过使用`deferred`扩展器（对监控属性、计算监控属性和监控数组都适用）：

```js
this.data = ko.observableArray().extend({ deferred: true });
```

### 例：避免多次ajax请求

下面的模型表示一个可以渲染成分页列表的数据：

```js
function GridViewModel() {
    this.pageSize = ko.observable(20);
    this.pageIndex = ko.observable(1);
    this.currentPageData = ko.observableArray();

    // Query /Some/Json/Service whenever pageIndex or pageSize changes,
    // and use the results to update currentPageData
    ko.computed(function() {
        var params = { page: this.pageIndex(), size: this.pageSize() };
        $.getJSON('/Some/Json/Service', params, this.currentPageData);
    }, this);
}
```

可以看到，计算监控属性同时依赖于`pageIndex`和`pageSize`，因此在模型第一次实例化和以后`pageIndex`或`pageSize`更新时，代码会使用getJSON来重新加载`currentPageData`。

这时有个潜在的效率问题，例如同时改变`pageIndex`和`pageSize`的值时，会触发两次ajax请求：

```js
this.setPageSize = function(newPageSize) {
    // Whenever you change the page size, we always reset the page index to 1
    this.pageSize(newPageSize);
    this.pageIndex(1);
}
```
这会浪费带宽和服务器资源，并且导致不可预知的竞态条件。

对计算监控属性使用deferred更新可以保证当前任务中的依赖多次改变时只出发一次更新。

```js
ko.computed(function() {
    // This evaluation logic is exactly the same as before
    var params = { page: this.pageIndex(), size: this.pageSize() };
    $.getJSON('/Some/Json/Service', params, this.currentPageData);
}, this).extend({ deferred: true });
```

### 强制deferred更新提前发生

尽管可以延迟更新，更少的UI更新让异步通知看起来更好，但是如果想立刻更新UI怎么办？这种情况下可以使用`ko.tasks.runEarly method`：

```js
// remove an item from an array
var items = myArray.splice(sourceIndex, 1);

// force updates so the UI will see a delete/add rather than a move
ko.tasks.runEarly();

// add the item in a new location
myArray.splice(targetIndex, 0, items[0]);
```

### 强制延迟更新的监控属性始终通知订阅者

当监控属性的值是原始类型时（数字、字符串、bool值、null），监控属性只会在设置的值与之前的值不一样时才通知依赖者。因此，原始值得监控属性延迟更新时，只会在当前任务结束前，值的确不一样了才发出通知。也就是说如果原始类型监控属性延迟更新，当值变化了又变回原来的值，是不会通知更新的。

为了保证订阅者始终能收更新到通知，即使值一样，可以使用`notify`扩展器：

```js
ko.options.deferUpdates = true;

myViewModel.fullName = ko.pureComputed(function() {
    return myViewModel.firstName() + " " + myViewModel.lastName();
}).extend({ notify: 'always' });
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/deferred-updates.html)
