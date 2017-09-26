---
author: wukn
catalog: true
date: 2016-11-12T01:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Bindings - Template'
url: /2016/11/12/knockoutjs-bindings-template/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## template

template的作用是用模板来填充相关的DOM元素。它提供了一种简单的方法来用视图模型的方法构建复杂的UI结构。

template有两种使用方式：
* 原生html模板（native templating），使用`foreach`、`if`、`with`等流程控制的绑定。这些绑定捕捉元素内的html片段，将其作为模板。
* 字符串模板（string-based templating），使用第三方模板引擎。ko会将模型数据传递给外部模板引擎，将结果注入到html文档中。常见的模板引擎有jquery.tmpl和Underscore。

template参数为字符传值时，ko会把它当成模板的ID。提供给模板的数据就是当前的上下文模型对象。

template参数为js对象时，可以有以下属性：
* name：包含模板的DOM元素的ID。
* nodes：直接传递DOM节点数组作为模板。这将是个非监控数组，且会覆盖当前元素内的内容。当name非空时，该选项会被忽略。
* data：为模板提供数据的对象。如果忽略该参数，ko会继续寻找foreach参数，或回退到使用当前模型对象。
* if：如果提供了if参数，那么ko仅在其为true时渲染模板。
* foreach：指示ko使用foreach模式来渲染模板。
* as：使用foreach时，可以用as来为数组项定义别名。
* `afterRender`、`afterAdd`、`beforeRemove`：渲染DOM元素后的回调函数。

例1：渲染一个具名模板

使用流程控制绑定时，可以将模板隐式并匿名地定义在DOM元素的内部，这时模板就不需要name。也可以将模板提出来，放到单独的DOM元素中，并为其指定name。

```html
<h2>Participants</h2>
Here are the participants:
<div data-bind="template: { name: 'person-template', data: buyer }"></div>
<div data-bind="template: { name: 'person-template', data: seller }"></div>

<script type="text/html" id="person-template">
    <h3 data-bind="text: name"></h3>
    <p>Credits: <span data-bind="text: credits"></span></p>
</script>

<script type="text/javascript">
     function MyViewModel() {
         this.buyer = { name: 'Franklin', credits: 250 };
         this.seller = { name: 'Mario', credits: 5800 };
     }
     ko.applyBindings(new MyViewModel());
</script>
```

例2：具名模板使用foreach选项

```html
<h2>Participants</h2>
Here are the participants:
<div data-bind="template: { name: 'person-template', foreach: people }"></div>

<script type="text/html" id="person-template">
    <h3 data-bind="text: name"></h3>
    <p>Credits: <span data-bind="text: credits"></span></p>
</script>

 function MyViewModel() {
     this.people = [
         { name: 'Franklin', credits: 250 },
         { name: 'Mario', credits: 5800 }
     ]
 }
 ko.applyBindings(new MyViewModel());
```
上面的视图与下面的是等价的：

```html
<div data-bind="foreach: people">
    <h3 data-bind="text: name"></h3>
    <p>Credits: <span data-bind="text: credits"></span></p>
</div>
```

例3：使用as为foreach项赋别名

在嵌套的foreach模板中，可以使用`$parent`等绑定上下文来访问更高层的对象。更简单且优雅的方式是使用as。

```html
<ul data-bind="template: { name: 'seasonTemplate', foreach: seasons, as: 'season' }"></ul>

<script type="text/html" id="seasonTemplate">
    <li>
        <strong data-bind="text: name"></strong>
        <ul data-bind="template: { name: 'monthTemplate', foreach: months, as: 'month' }"></ul>
    </li>
</script>

<script type="text/html" id="monthTemplate">
    <li>
        <span data-bind="text: month"></span>
        is in
        <span data-bind="text: season.name"></span>
    </li>
</script>

<script>
    var viewModel = {
        seasons: ko.observableArray([
            { name: 'Spring', months: [ 'March', 'April', 'May' ] },
            { name: 'Summer', months: [ 'June', 'July', 'August' ] },
            { name: 'Autumn', months: [ 'September', 'October', 'November' ] },
            { name: 'Winter', months: [ 'December', 'January', 'February' ] }
        ])
    };
    ko.applyBindings(viewModel);
</script>
```

注意，传递给as的是一个字符串字面量而不是变量，因为这里是为一个新变量命名，而不是读取一个已存在的变量的值。

例4：动态选择模板

如果有多个具名模板，可以为`name`传递一个监控变量，当监控变量的值发生变化时，元素的内容会使用相应的模板重新渲染。也可以传递一个回调函数来决定使用哪个模板。如果时foreach模板模式，ko会对数组的每一项进行处理，将数组项的值作为唯一参数。否则，参数会是`data`的值，或回退到整个当前模型对象。函数的第二个参数是绑定上下文，因此在动态选择模板时可以使用`$parent`等绑定上下文。
如果函数中引用了监控变量，那么变量变化时，绑定也会更新，也就是说，会根据数据和相应的模板重新渲染。

```html
<ul data-bind='template: { name: displayMode,
                           foreach: employees }'> </ul>

<script>
    var viewModel = {
        employees: ko.observableArray([
            { name: "Kari", active: ko.observable(true) },
            { name: "Brynn", active: ko.observable(false) },
            { name: "Nora", active: ko.observable(false) }
        ]),
        displayMode: function(employee) {
            // Initially "Kari" uses the "active" template, while the others use "inactive"
            return employee.active() ? "active" : "inactive";
        }
    };

    // ... then later ...
    viewModel.employees()[1].active(true); // Now "Brynn" is also rendered using the "active" template.
</script>
```

例5：使用基于字符串的外部模板引擎jQuery.tmpl

使用jQuery.tmpl时，需要引用jQuery和jQuery.tmpl。目前jQuery.tmpl已停止更新，不建议使用。

```html
<h1>People</h1>
<div data-bind="template: 'peopleList'"></div>

<script type="text/html" id="peopleList">
    {{each people}}
        <p>
            <b>${name}</b> is ${age} years old
        </p>
    {{/each}}
</script>

<script type="text/javascript">
    var viewModel = {
        people: ko.observableArray([
            { name: 'Rod', age: 123 },
            { name: 'Jane', age: 125 },
        ])
    }
    ko.applyBindings(viewModel);
</script>
```

例6：使用模板引擎Underscore

Undercore默认使用“ERB-style delimiters”（`<%= ... %>`），也可以设置使用其他分割符。

```html
<script type="text/html" id="peopleList">
    <% _.each(people(), function(person) { %>
        <li>
            <b><%= person.name %></b> is <%= person.age %> years old
        </li>
    <% }) %>
</script>
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/template-binding.html)
