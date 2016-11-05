---
layout:     post
title:      "[KnockoutJS] Bindings - Controlling text and appearence"
subtitle:   "绑定-控制文本和外观"
date:       2016-11-04 02:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - KnockoutJS

---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"


## visible

visible绑定用于控制DOM元素隐藏或显示。

```html
<div data-bind="visible: shouldShowMessage">
    You will see this message only when "shouldShowMessage" holds a true value.
</div>

<script type="text/javascript">
    var viewModel = {
        shouldShowMessage: ko.observable(true) // Message initially visible
    };
    viewModel.shouldShowMessage(false); // ... now it's hidden
    viewModel.shouldShowMessage(true); // ... now it's visible again
</script>
```

当参数值为假时（false-like value：bool值false、数字0、null、undefined等），该绑定会设置DOM元素的`yourElement.style.display`属性为none使其隐藏。它的优先级高于CSS中的定义的隐藏样式。

当参数值为真时（true-like value：bool值true、不为null的对象或数组等），该绑定会移除DOM元素的`yourElement.style.display`属性使其可见。注意此时CSS中定义的样式将会生效。

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

绑定的参数可以使用函数或表达式。

```html
<div data-bind="visible: myValues().length > 0">
    You will see this message only when 'myValues' has at least one member.
</div>

<script type="text/javascript">
    var viewModel = {
        myValues: ko.observableArray([]) // Initially empty, so message hidden
    };
    viewModel.myValues.push("some value"); // Now visible
</script>
```

## text

text绑定用于DOM元素显示文本，通过设置元素的innerText或textContent。
通常用于<span>或<em>元素用来显示文本，实际上可用于任何元素。

```html
Today's message is: <span data-bind="text: myMessage"></span>

<script type="text/javascript">
    var viewModel = {
        myMessage: ko.observable() // Initially blank
    };
    viewModel.myMessage("Hello, world!"); // Text appears
</script>
```

如果绑定参数的不是字符串或数字，那么DOM元素的text相当于`yourParameter.toString()`。

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

绑定的参数可以使用函数或表达式。

注1：与html不同，参数值中若包含html标签，显示时不会解析成html，而是原样显示。

注2：使用text但没有承载元素

如果想用ko进行text绑定但没有元素可用于绑定时，例如option元素中不能包含其他元素：

```html
<select data-bind="foreach: items">
    <option>Item <span data-bind="text: name"></span></option><!--not work-->
</select>
```

可以使用基于标注标签的无承载标签语法（containerless syntax），以`<!--ko-->`和`<!--/ko-->`为开始标签和结束标签定义一个“虚拟元素 ”（virtual element）。

```html
<select data-bind="foreach: items">
    <option>Item <!--ko text: name--><!--/ko--></option>
</select>
```

注3：IE6的空格问题

IE6并不会显示span标签后紧跟的空格，因此

```html
Welcome, <span data-bind="text: userName"></span> to our web site.<!--not whitespace before 'to' in IE6-->

Welcome, <span data-bind="text: userName">&nbsp;</span> to our web site.<!--show as you wish-->
```

## html

html绑定用于DOM元素显示html内容。

```html
<div data-bind="html: details"></div>

<script type="text/javascript">
    var viewModel = {
        details: ko.observable() // Initially blank
    };
    viewModel.details("<em>For further details, view the report <a href='report.html'>here</a>.</em>"); // HTML content appears
</script>
```

jquery可用时，ko使用jquery的html函数，否则将参数解析成html节点并添加为绑定DOM元素的子节点。

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

如果绑定参数的不是字符串或数字，那么DOM元素的innerHTML相当于`yourParameter.toString()`。

## CSS

css绑定用于向DOM元素添加或移除命名的CSS类。

例1：绑定静态CSS类

```html
<div data-bind="css: { profitWarning: currentProfit() < 0 }">
   Profit Information
</div>

<script type="text/javascript">
    var viewModel = {
        currentProfit: ko.observable(150000) // Positive value, so initially we don't apply the "profitWarning" class
    };
    viewModel.currentProfit(-50); // Causes the "profitWarning" class to be applied
</script>
```

当currentProfit小于0时，CSS类profitWarning被应用；当currentProfit大于等于0时，CSS类profitWarning被移除。

例2：绑定动态CSS类

```html
<div data-bind="css: profitStatus">
   Profit Information
</div>

<script type="text/javascript">
    var viewModel = {
        currentProfit: ko.observable(150000)
    };

    // Evalutes to a positive value, so initially we apply the "profitPositive" class
    viewModel.profitStatus = ko.pureComputed(function() {
        return this.currentProfit() < 0 ? "profitWarning" : "profitPositive";
    }, viewModel);

    // Causes the "profitPositive" class to be removed and "profitWarning" class to be added
    viewModel.currentProfit(-50);
</script>
```

当currentProfit非负时，应用CSS类profitPositive，否则应用CSS类profitWarning。

如果使用静态CSS类名，可以传递一个属性名为CSS类名、值为可被计算为bool值的js对象。也就是说，可以一次设置多个CSS类：

```html
<div data-bind="css: { profitWarning: currentProfit() < 0, majorHighlight: isSevere }">
```

甚至可以通过将多个CSS类名时用引号包装用同一个条件控制：

```html
<div data-bind="css: { profitWarning: currentProfit() < 0, 'major highlight': isSevere }">
```

非bool值会被宽松地解释为bool值。如0和null会被当成false，21或非null对象会被当成true。

如果使用动态CSS类名，将CSS类名作为字符串传递给参数就行。如果绑定的时监控属性，监控属性变化时，之前添加的CSS类将被移除而重新添加。

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

绑定的参数可以使用函数或表达式。

当CSS类名是非法的js变量名时，例如my-class，不能直接这样写：

```html
<div data-bind="css: { my-class: someValue }">...</div>
```

应该时用引号使其成为字符串字面量（string literal），从而在js对象字面量（object literal）中时合理的。

```html
<div data-bind="css: { 'my-class': someValue }">...</div>
```

## style

style绑定用于向DOM元素添加或移除style值。

```html
<div data-bind="style: { color: currentProfit() < 0 ? 'red' : 'black' }">
   Profit Information
</div>

<script type="text/javascript">
    var viewModel = {
        currentProfit: ko.observable(150000) // Positive value, so initially black
    };
    viewModel.currentProfit(-50); // Causes the DIV's contents to go red
</script>
```

当currentProfit值小于0时，元素的style.color属性被赋值red，大于等于0时赋值black。

绑定的参数可以使用属性名为style名称、值为style值的js对象。

```html
<div data-bind="style: { color: currentProfit() < 0 ? 'red' : 'black', fontWeight: isSevere() ? 'bold' : '' }">...</div>
```

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

绑定的参数可以使用函数或表达式。

不能使用非js标识符的style名称（如font-weight或text-decoration等），必须使用style的js名称（如{ fontWeight: someValue }或{ textDecoration: someValue }等）。

详见JavaScript Style Attributes。

## attr

attr绑定提供更通用的方式向DOM元素添加属性。

```html
<a data-bind="attr: { href: url, title: details }">
    Report
</a>

<script type="text/javascript">
    var viewModel = {
        url: ko.observable("year-end.html"),
        details: ko.observable("Report including final year-end statistics")
    };
</script>
```

绑定的参数可以使用属性名为属性名称、值为属性值的js对象。

如果参数是监控属性，该绑定会随着参数值变化而更新元素，否则只会绑定一次而不再更新。

属性名为关键字或非法的js变量名时，需要使用引号包裹。

[Javacript Keywords](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#Keywords)。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/visible-binding.html)
