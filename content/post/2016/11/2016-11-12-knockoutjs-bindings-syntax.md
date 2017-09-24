---
author: wukn
catalog: true
date: 2016-11-12T02:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Bindings - Syntax'
url: /2016/11/12/knockoutjs-bindings-syntax/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

ko的声明式绑定系统提供了简洁且功能强大的链接数据和UI的方式。

## data-bind语法

一个绑定由两部分组成，名称和值，用分号隔开。

```html
Today's message is: <span data-bind="text: myMessage"></span>
```

一个元素可以由多个绑定，绑定之间使用逗号隔开。

```html
<!-- related bindings: valueUpdate is a parameter for value -->
Your value: <input data-bind="value: someValue, valueUpdate: 'afterkeydown'" />

<!-- unrelated bindings -->
Cellphone: <input data-bind="value: cellphoneNumber, enable: hasCellphone" />

```

绑定的名称应该与一个注册的绑定处理器（包括ko内建的和自定义的）匹配，或者是另一个绑定的参数。否则会被忽略（没有错误或警告）。

绑定的值可以是值、变量或字面量（[value、variable、literal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types)），或者是任何有效的js表达式（[Javacript Expression](https://developer.mozilla.org/en-US/docs/JavaScript/Guide/Expressions_and_Operators)）。

当值是对象字面量时，对象的属性名必须是有效的js标识符，否则需要放在引号中。

当值是无效的js表达式或引用的是未知的变量时，ko会输出错误并停止绑定处理。

```html
<!-- variable (usually a property of the current view model -->
<div data-bind="visible: shouldShowMessage">...</div>

<!-- comparison and conditional -->
The item is <span data-bind="text: price() > 50 ? 'expensive' : 'cheap'"></span>.

<!-- function call and comparison -->
<button data-bind="enable: parseAreaCode(cellphoneNumber()) != '555'">...</button>

<!-- function expression -->
<div data-bind="click: function (data) { myFunction('param1', data) }">...</div>

<!-- object literal (with unquoted and quoted property names) -->
<div data-bind="with: {emotion: 'happy', 'facial-expression': 'smile'}">...</div>
```

绑定中可以包含任意数量的空字符（whitespace：spaces、tab、newlines），因此绑定中可以根据需要来排版。

```html
<!-- no spaces -->
<select data-bind="options:availableCountries,optionsText:'countryName',value:selectedCountry,optionsCaption:'Choose...'"></select>

<!-- some spaces -->
<select data-bind="options : availableCountries, optionsText : 'countryName', value : selectedCountry, optionsCaption : 'Choose...'"></select>

<!-- spaces and newlines -->
<select data-bind="
    options: availableCountries,
    optionsText: 'countryName',
    value: selectedCountry,
    optionsCaption: 'Choose...'"></select>
```

注：ko V3.0之后，可以声明没有值的绑定，值会被赋为undefined。这对绑定预处理是有很用的。

```html
<span data-bind="text">Text that will be cleared when bindings are applied.</span>
```

## 绑定上下文（binding context）

绑定上下文是可以在绑定内引用的包含数据的对象。使用绑定时，ko会自动创建和管理绑定上下文的层级关系。层级关系的根就是应用绑定`ko.applyBindings(viewModel)`的视图viewModel。然后每使用一个流程控制绑定（如with或foreach），就会创建一个指向嵌套的视图模型数据的子绑定上下文。

绑定上下文由如下特殊属性。

* $parent 它指向当前绑定上下文的父上下文的视图模型对象。对于根上下文，它指向undefined。

* $parents 它是所有父视图模型的数组。$parents[0]与$parent相同，指向父上下文的视图模型；$parents[1]指向$parents[0]的父上下文的视图模型；以此类推。

* $root 它指向根上下文的视图模型对象，通常就是传递给`ko.applyBindings`的对象。与`$parents[$parents.length - 1]`等价。

* $component 在component模板的上下文中，$component指向component的视图模型。在嵌套组件中，它指向最近一层组件的视图模型。

* $data 它指向当前上下文的视图模型对象。在根上下文中$data和$root是一样的。在嵌套的绑定中，该参数会被设置为当前数据项，当需要引用视图模型本身而不是视图模型的属性时，就可以用$data。

```html
<ul data-bind="foreach: ['cats', 'dogs', 'fish']">
    <li>The value is <span data-bind="text: $data"></span></li>
</ul>
```

* $index 它是foreach绑定中当前数组项的索引（从0开始）。与其他上下文属性不同，$index是可监控的，会随着数组项索引的变化而更新。

* $parentContext 它指向父层的绑定上下文对象。它根$parent是不一样的，$parent是指向父层的数据而不是绑定上下文。在根上下文中它指向undefined。
例如在内层上下文中访问外层foreach项的索引值：`$parentContext.$index`。

* $rawData 它是当前上下文的原始视图模型值。通常与$data相同。但是视图模型包装在监控属性内时，$data是未包装的视图模型，而$rawData是监控属性本身。

* $componentTemplateNodes 在component模板的上下文中，$componentTemplateNodes是包含传递给组件的DOM节点的数组。这使得构建可以接收模板的组件很容易，例如构建可以接收行模板的表格组件。

下面两个是绑定中的特殊变量，但不属于绑定上下文对象。

* context 它指向当前绑定上下文对象。

* element 它指向当前绑定的与元素DOM对象（element DOM object），或者虚拟元素的注释DOM对象（comment DOM object）。在需要访问当前元素的属性时很有用。

```html
<div id="item1" data-bind="text: $element.id"></div>
```

注：与with和foreach类似，在自定义绑定中可以改变子元素的绑定上下文，或者扩展绑定上下文对象来提供特殊的属性。

## 非侵入式事件绑定

在大部分情况下，`data-bind`属性提供了一种简洁的方式来绑定视图模型。但是在绑定事件处理函数时，在`data-bind`中通常使用匿名函数来传递参数。

```html
<a href="#" data-bind="click: function() { viewModel.items.remove($data); }">
    remove
</a>
```

此外，Ko还提供了两个实用函数来获取DOM元素关联的数据。

* `Ko.dataFor(element)`：返回元素绑定的数据。
* `Ko.contextFor(element)`：返回DOM元素的整个绑定上下文。

这两个函数可以在非侵入式事件绑定时使用，如jquery的bind或click。

```javascript
$(".remove").click(function () {
    viewModel.items.remove(ko.dataFor(this));
});
```

此外也可以在事件代理中使用，如jquery的live/delegate/on。这样绑定的事件对动态添加的元素也是生效的。

```javascript
$(".container").on("click", ".remove", function() {
    viewModel.items.remove(ko.dataFor(this));
});
```

例：

```html
<ul id="people" data-bind='template: { name: "personTmpl", foreach: people }'>
</ul>

<script id="personTmpl" type="text/html">
    <li>
        <a class="remove" href="#"> x </a>
        <span data-bind='text: name'></span>
        <a class="add" href="#"> add child </a>
        <ul data-bind='template: { name: "personTmpl", foreach: children }'></ul>
    </li>
</script>
```

```javascript
var Person = function(name, children) {
    this.name = ko.observable(name);
    this.children = ko.observableArray(children || []);
};

var PeopleModel = function() {
    this.people = ko.observableArray([
        new Person("Bob", [
            new Person("Jan"),
            new Person("Don", [
                new Person("Ted"),
                new Person("Ben", [
                    new Person("Joe", [
                        new Person("Ali"),
                        new Person("Ken")
                    ])
                ]),
                new Person("Doug")
            ])
        ]),
        new Person("Ann", [
            new Person("Eve"),
            new Person("Hal")
        ])
    ]);

    this.addChild = function(name, parentArray) {
        parentArray.push(new Person(name));
    };
};

ko.applyBindings(new PeopleModel());

//attach event handlers
$("#people").on("click", ".remove", function() {
    //retrieve the context
    var context = ko.contextFor(this),
        parentArray = context.$parent.people || context.$parent.children;

    //remove the data (context.$data) from the appropriate array on its parent (context.$parent)
    parentArray.remove(context.$data);

    return false;
});

$("#people").on("click", ".add", function() {
    //retrieve the context
    var context = ko.contextFor(this),
        childName = context.$data.name() + " child",
        parentArray = context.$data.people || context.$data.children;

    //add a child to the appropriate parent, calling a method off of the main view model (context.$root)
    context.$root.addChild(childName, parentArray);

    return false;
});
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/binding-syntax.html)
