---
author: wukn
catalog: true
date: 2016-11-12T00:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Bindings - Control flow'
url: /2016/11/12/knockoutjs-bindings-control-flow/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## foreach

foreach对数组中的每一项复制一份html片段并与其绑定，常用于加载列表和表格。

```html
table>
    <thead>
        <tr><th>First name</th><th>Last name</th></tr>
    </thead>
    <tbody data-bind="foreach: people">
        <tr>
            <td data-bind="text: firstName"></td>
            <td data-bind="text: lastName"></td>
        </tr>
    </tbody>
</table>

<script type="text/javascript">
    ko.applyBindings({
        people: [
            { firstName: 'Bert', lastName: 'Bertington' },
            { firstName: 'Charles', lastName: 'Charlesforth' },
            { firstName: 'Denise', lastName: 'Dentiste' }
        ]
    });
</script>
```

如果使用的是监控数组，那么以后对数组进行添加、删除或重新排序等，界面会自动更新。

### 注1：使用$data、$index、$parent等上下文属性

* $data用于访问每次循环时的数组项。
* $index为当前数组项的索引（从0开始）。这是个监控变量，会在数组项索引发生变化时更新。
* $parent用于访问foreach外层的数据。

例如，参数为js对象字面量数组时，使用$data来访问数组：

```html
<ul data-bind="foreach: months">
    <li>
        The item <b data-bind="text: $index"></b> is: <b data-bind="text: $data"></b>
    </li>
</ul>

<script type="text/javascript">
    ko.applyBindings({
        months: [ 'Jan', 'Feb', 'Mar', 'etc' ]
    });
</script>
```

访问数组项的属性时，也可以以$data为前缀（实际上完全不必要）。

```html
<td data-bind="text: $data.firstName"></td>
```

详见[binding context properties](http://knockoutjs.com/documentation/binding-context.html)。

### 注2：使用别名访问数组

除了使用$data，还是以使用别名来访问数组：

```html
<ul data-bind="foreach: { data: categories, as: 'category' }">
    <li>
        <ul data-bind="foreach: { data: items, as: 'item' }">
            <li>
                <span data-bind="text: category.name"></span>:
                <span data-bind="text: item"></span>
            </li>
        </ul>
    </li>
</ul>

<script>
    var viewModel = {
        categories: ko.observableArray([
            { name: 'Fruit', items: [ 'Apple', 'Orange', 'Banana' ] },
            { name: 'Vegetables', items: [ 'Celery', 'Corn', 'Spinach' ] }
        ])
    };
    ko.applyBindings(viewModel);
</script>
```

要注意的是，传递给as的是个字符串字面量，因为是给新变量命名，而不是读取已存在的变量的值。

### 注3：无承载元素时使用foreach

例如希望结果如下：

```html
<ul>
    <li class="header">Header item</li>
    <!-- The following are generated dynamically from an array -->
    <li>Item A</li>
    <li>Item B</li>
    <li>Item C</li>
</ul>
```

但foreach绑定不能用于ul，否则Header item会重复，而ul里除了li不能出现其他元素。这种情形下可以使用基于标注标签的虚拟元素。

```html
<ul>
    <li class="header">Header item</li>
    <!-- ko foreach: myItems -->
        <li>Item <span data-bind="text: $data"></span></li>
    <!-- /ko -->
</ul>

<script type="text/javascript">
    ko.applyBindings({
        myItems: [ 'A', 'B', 'C' ]
    });
</script>
```

### 注4：数组的变化使如何被检测和处理的

当数组变化时，foreach绑定以一种高效的差异算法来识别变化的部分，并且只更新变化部分的DOM元素。

* 添加数组元素时，foreach根据模板仅渲染新增的部分插入到已存在的DOM元素中。
* 删除数组元素时，foreach仅移除相应的DOM元素。
* 数组重新排序时，foreach将相应的DOM元素移到新的位置。为了保证算法的高效性，重排时也可能使用“移除+添加”来代替移动。

### 注5：默认数组项隐藏时销毁

有时希望数组项标记删除但仍存在于数组中时（non-destructive delete），可参考监控数组的destroy方法。
如果想显示删除的远渡，使用includeDestroyed选项。

```html
<div data-bind='foreach: { data: myArray, includeDestroyed: true }'>
    ...
</div>
```

### 注6：生成DOM元素的后处理或动画

如果希望在生成DOM元素后进行一些逻辑处理，可使用回调函数afterRender、afterAdd、beforeRemove、beforeMove、afterMove。

```html
<!-- need jQuery Color plugin (https://github.com/jquery/jquery-color) -->
<ul data-bind="foreach: { data: myItems, afterAdd: yellowFadeIn }">
    <li data-bind="text: $data"></li>
</ul>

<button data-bind="click: addItem">Add</button>

<script type="text/javascript">
    ko.applyBindings({
        myItems: ko.observableArray([ 'A', 'B', 'C' ]),
        yellowFadeIn: function(element, index, data) {
            $(element).filter("li")
                      .animate({ backgroundColor: 'yellow' }, 200)
                      .animate({ backgroundColor: 'white' }, 800);
        },
        addItem: function() { this.myItems.push('New item'); }
    });
</script>
```

## if

if用于控制一段html片段是否加载。

if和visible相似，区别是使用visible时元素仍保留在DOM文档中，只是用CSS来切换其可见性，而if会根据参数值添加或移除DOM元素。

```html
<label><input type="checkbox" data-bind="checked: displayMessage" /> Display message</label>
<div data-bind="if: displayMessage">Here is a message. Astonishing.</div>

<script type="text/javascript">
ko.applyBindings({
    displayMessage: ko.observable(false)
});
</script>
```

下面的例子中，div不会在Mercury中显示，而Earth中会显示。因为Mercury没有capital属性。如果不使用if，在显示Mercury时会尝试读取不存在的capital属性，会导致出错。

```html
<ul data-bind="foreach: planets">
    <li>
        Planet: <b data-bind="text: name"> </b>
        <div data-bind="if: capital">
            Capital: <b data-bind="text: capital.cityName"> </b>
        </div>
    </li>
</ul>


<script>
    ko.applyBindings({
        planets: [
            { name: 'Mercury', capital: null },
            { name: 'Earth', capital: { cityName: 'Barnsley' } }        
        ]
    });
</script>
```

注：无承载元素时使用if，可使用基于标注标签的虚拟元素。

```html
<ul>
    <li>This item always appears</li>
    <!-- ko if: someExpressionGoesHere -->
        <li>I want to make this item present/absent dynamically</li>
    <!-- /ko -->
</ul>
```

## ifnot

ifnot与if的相似，只是提供另一个选择，有时可以使代码更具有可读性。

```html
<div data-bind="ifnot: someProperty">...</div>
<!--… is equivalent to the following:-->
<div data-bind="if: !someProperty()">...</div>
```

## with

with用于创建一个新的绑定上下文，使内部的元素绑定到指定的对象。

with可嵌套使用在if、foreach等流程控制绑定中。

例1：with内的latitude或longitude不需要coords.前缀，因为绑定上下文已切换到coords。

```html
<h1 data-bind="text: city"> </h1>
<p data-bind="with: coords">
    Latitude: <span data-bind="text: latitude"> </span>,
    Longitude: <span data-bind="text: longitude"> </span>
</p>

<script type="text/javascript">
    ko.applyBindings({
        city: "London",
        coords: {
            latitude:  51.5001524,
            longitude: -0.1262362
        }
    });
</script>
```

例2：使用with绑定根据参数是否为null/undefined动态添加或移除子元素。

```html
<form data-bind="submit: getTweets">
    Twitter account:
    <input data-bind="value: twitterName" />
    <button type="submit">Get tweets</button>
</form>

<div data-bind="with: resultData">
    <h3>Recent tweets fetched at <span data-bind="text: retrievalDate"> </span></h3>
    <ol data-bind="foreach: topTweets">
        <li data-bind="text: text"></li>
    </ol>

    <button data-bind="click: $parent.clearResults">Clear tweets</button>
</div>

<script type="text/javascript">
function AppViewModel() {
    var self = this;
    self.twitterName = ko.observable('@example');
    self.resultData = ko.observable(); // No initial value

    self.getTweets = function() {
        var name = self.twitterName(),
            simulatedResults = [
                { text: name + ' What a nice day.' },
                { text: name + ' Building some cool apps.' },
                { text: name + ' Just saw a famous celebrity eating lard. Yum.' }
            ];

        self.resultData({ retrievalDate: new Date(), topTweets: simulatedResults });
    }

    self.clearResults = function() {
        self.resultData(undefined);
    }
}

ko.applyBindings(new AppViewModel());
</script>
```

在with内部使用父绑定的数据或方法时，可使用$parent、$root等。

注：无承载元素时使用with，可使用基于标注标签的虚拟元素。

```html
<ul>
    <li>Header element</li>
    <!-- ko with: outboundFlight -->
        ...
    <!-- /ko -->
    <!-- ko with: inboundFlight -->
        ...
    <!-- /ko -->
</ul>
```

## component

component用户向DOM元素中注入自定义的组件，并可选择地传递参数。

```html
<h4>First instance, without parameters</h4>
<div data-bind='component: "message-editor"'></div>

<h4>Second instance, passing parameters</h4>
<div data-bind='component: {
    name: "message-editor",
    params: { initialText: "Hello, world!" }
}'></div>

<script type="text/javascript">
ko.components.register('message-editor', {
    viewModel: function(params) {
        this.text = ko.observable(params && params.initialText || '');
    },
    template: 'Message: <input data-bind="value: text" /> '
            + '(length: <span data-bind="text: text().length"></span>)'
});

ko.applyBindings();
</script>
```

component绑定有两种使用方式。

* 简写语法：绑定一个字符串或监控属性作为组件名称。使用监控变量时，其值发生变化，组件也会重新加载。
* 完整语法：绑定一个有name和params属性的对象，分别为组件名称和包含多个参数键值对的参数。参数一般由组件的视图模型的构造函数接收。

```html
<!--shorthand syntax-->
<div data-bind='component: "my-component"'></div>
<div data-bind='component: observableWhoseValueIsAComponentName'></div>
```

```html
<!--full syntax-->
<div data-bind='component: {
    name: "shopping-cart",
    params: { mode: "detailed-list", items: productsList }
}'></div>
```

注：组件被移除时会被释放掉（disposed）。

### 注：组件的生命周期

当component绑定注入一个组件时：

* 组件加载器（component loaders）被要求提供视图模型工厂和模板。这里，可能多个组件加载器被访问，直到某个找到并提供了视图模型和模板。每个组件只会这样处理一次，ko会保存该结果。通常这步是异步处理。
* 组件的模板被复制并注入到容器元素中。已存在的内容会被移除和丢弃。
* 如果组件有视图模型，那么初始化它。如果视图模型有构造函数，那么ko会调用它YourViewModel(params)。如果视图模型由工厂方法createViewModel给出，ko会调用createViewModel(params, componentInfo)。这步始终是同步处理。
* 视图模型绑定视图。或者没有视图模型，那么将params参数绑定到视图。
* 激活组件。
* 组件被移除，视图模型被调用dispose释放。（浏览器直接导航到其他地址时，不会调用dispose，但浏览器会释放所有内存）

### 注：模板组件（template-only component）

组件通常含有视图模型，但这不是必须的。组件可以仅有模板。这种情况下，组件的视图绑定的对象就是传递给组件的params对象。

```js
ko.components.register('special-offer', {
    template: '<div class="offer-box" data-bind="text: productName"></div>'
});
```
```html
<div data-bind='component: {
     name: "special-offer",
     params: { productName: someProduct.name }
}'></div>
<!-- or using custom element-->
special-offer params='productName: someProduct.name'></special-offer>
```

注：无承载元素时使用component，可以使用虚拟元素。

```html
<!-- ko component: "message-editor" -->
<!-- /ko -->
<!-- ko component: {
    name: "message-editor",
    params: { initialText: "Hello, world!", otherParam: 123 }
} -->
<!-- /ko -->
```

注：使用component绑定的元素可能包含一些额外的元素。尽管这些额外的元素会被剥离且不绑定，但它们仍然存在。组件可以根据需要决定是否加载它们。

```html
<div data-bind="component: { name: 'my-special-list', params: { items: someArrayOfPeople } }">
    <!-- Look, here's some arbitrary markup. By default it gets stripped out
         and is replaced by the component output. -->
    The person <em data-bind="text: name"></em>
    is <em data-bind="text: age"></em> years old.
</div>
```

### 注：释放和内存管理

视图模型也许提供了dispose方法，当ko在移除组件时会调用它。

必须在dispose中释放gc不会处理的资源，否则会造成内存泄漏。例如：

* setInterval会一直执行直到被显式清除，需要使用clearInterval(handle)来停止它。
* ko.computed属性会继续受到外部依赖者的通知直到被显式释放，需要调用计算属性的.dispose()方法。也可以使用纯计算属性来避免手动释放。
* 对外部监控属性的订阅会一直执行直到被显式清除，需要调用订阅的.dispose()方法。
* 手动创建的外部DOM元素的事件处理器，需要被手动释放。

```js
var someExternalObservable = ko.observable(123);

function SomeComponentViewModel() {
    this.myComputed = ko.computed(function() {
        return someExternalObservable() + 1;
    }, this);

    this.myPureComputed = ko.pureComputed(function() {
        return someExternalObservable() + 2;
    }, this);

    this.mySubscription = someExternalObservable.subscribe(function(val) {
        console.log('The external observable changed to ' + val);
    }, this);

    this.myIntervalHandle = window.setInterval(function() {
        console.log('Another second passed, and the component is still alive.');
    }, 1000);
}

SomeComponentViewModel.prototype.dispose = function() {
    this.myComputed.dispose();
    this.mySubscription.dispose();
    window.clearInterval(this.myIntervalHandle);
    // this.myPureComputed doesn't need to be manually disposed.
}

ko.components.register('your-component-name', {
    viewModel: SomeComponentViewModel,
    template: 'some template'
});
```

严格上，不需要释放同一个视图模型属性的计算监控和订阅，因为这产生了循环引用，js的gc知道该如何处理。为了避免要记住哪些资源释放，尽量使用纯计算监控属性，然后不管是否需要都显式地释放其他计算监控属性和订阅。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/foreach-binding.html)
