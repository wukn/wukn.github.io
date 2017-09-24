---
author: wukn
catalog: true
date: 2016-11-12T09:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Microtask queue'
url: /2016/11/12/knockoutjs-microtask-queue/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

ko的微任务队列支持在异步运行时或尽量在I/O处理、重新流程处理、重新渲染界面之前对任务进行调度。在ko组件内部就是用它来维护异步行为和调度监控变量的deferred更新。

```js
ko.tasks.schedule(function () {
    // ...
});
```

使用上面的方法将提供的回调函数添加到微服务队列中。ko包含一个快速任务队列，它采用FIFO顺序执行任务知道队列为空。当第一个任务被调度时，如果浏览器支持的话，ko会使用浏览器的microtask来调度一个刷新事件。这样确保第一个任务和后续的任务相似。

微任务可被取消，使用`ko.tasks.schedule`返回的值。任务已经在运行或已经被取消时，再取消将不会处理。

```js
var handle = ko.tasks.schedule(/* ... */);
ko.tasks.cancel(handle);
```

## 错误处理

如果一个任务抛出了异常，并不会中断任务队列。异常会被推迟到后面的一个事件中，而且能被`ko.onError`或`window.onerror`捕获并处理。

当异常发生，在抛出原异常之前，可以使用`ko.onError`来处理异常。另外，由于原先的调用包装在try/catch中，传递给`ko.onError`的异常包含一个`stack`属性，而这是很多浏览器的`window.onerror`所没有的。

该功能适用于处理一下情形的错误：

* textInput和value绑定部分产生的异步更新
* 没有配置同步加载组件时，加载一个缓存的组件
* rateLimit计算中
* 由`ko.utils.registerEventHandler`添加的事件处理器，包括使用event和click绑定的

```js
ko.onError = function (error) {
    myLogger("error throwed.", error);
};
```

## 递归任务限制

由于ko会一直处理队列中的任务直到队列为空，不会返回到外层事件，大量的或耗时很长的任务会导致浏览器页面未响应。ko会在检测到有高层递归时取消剩余的所有任务，避免无线递归。例如，下面的例子中会最终停止并抛出错误：

```js
function loop() {
    ko.tasks.schedule(loop);
}
loop();
```

## 实现

当第一个任务被调度时（初始时和前一个刷新事件完成后），ko会安排一个刷新事件来处理微任务队列。如果可能的话，ko会尝试使用浏览器自身的microtask功能。在现代浏览器中会使用[DOM mutation observer](http://dom.spec.whatwg.org/#mutation-observers)，在老版本的IE中会使用`onreadystatechange`事件。这些方法可以使得在重新渲染页面之前开始处理任务队列。在其它浏览器中，将使用setTimeout来恢复处理。

## 高级队列控制

ko提供了一些高级方法来控制处理中的任务队列。

* `ko.tasks.runEarly()`：在需要时调用该方法来立刻处理微任务队列，直到队列为空。在与其它库集成时，可以在安排了一系列任务后，需要同步处理这些任务的效果时调用该方法。
* `ko.tasks.scheduler`：重写该方法来重定义或增加ko的调度处理和刷新队列的逻辑。ko在调度第一个任务时调用该方法，因此它必须调度事件并立即返回。你或许更倾向在刷新事件中使用`process.nextTick`：`ko.tasks.scheduler = process.nextTick;`。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/microtasks.html)
