---
author: wukn
catalog: true
date: 2016-11-12T04:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Components'
url: /2016/11/12/knockoutjs-components/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## 组件和自定义元素

组件的作用有：
* 构建独立的控件或部件。
* 拥有自己的视图，通常（但不是必须）包含试图模型。
* 可以通过AMD或其他模型系统异步加载或预加载（根据需要）。
* 可以接收参数，可选地可回写变化或调用回调函数。
* 可以相互组合或嵌套，或继承自其他组件。
* 可易于打包，项目间可重用。
* 可自定义配置和加载逻辑

这种模式对于构建大型应用是很有益的。它通过清晰地组织和封装功能与代码简化了开发，同时根据需要增量加载应用代码和模板等也提升了应用的性能。

自定义元素是组件的一个可选的但很便利的语法。与使用`<div>`注入组件绑定相比，使用自定义元素名称更具有自描述性。

### 例1：一个喜欢/不喜欢widget

可以使用`ko.components.register`来注册一个组件（技术上来说，使用注册是可选的，也是最简单的方式）。一个组件的定义需要指定`viewModel`和`template`。

```js
ko.components.register('like-widget', {
    viewModel: function(params) {
        // Data: value is either null, 'like', or 'dislike'
        this.chosenValue = params.value;

        // Behaviors
        this.like = function() { this.chosenValue('like'); }.bind(this);
        this.dislike = function() { this.chosenValue('dislike'); }.bind(this);
    },
    template:
        '<div class="like-or-dislike" data-bind="visible: !chosenValue()">\
            <button data-bind="click: like">Like it</button>\
            <button data-bind="click: dislike">Dislike it</button>\
        </div>\
        <div class="result" data-bind="visible: chosenValue">\
            You <strong data-bind="text: chosenValue"></strong> it\
        </div>'
});
```

注意，通常从外部文件加载模板和视图模型。

现在可以使用组件绑定或自定义元素来使用上面定义的绑定了。例如使用自定义元素的方式：

```html
<ul data-bind="foreach: products">
    <li class="product">
        <strong data-bind="text: name"></strong>
        <like-widget params="value: userRating"></like-widget>
    </li>
</ul>

<script type="text/javascript">browserify
function Product(name, rating) {
    this.name = name;
    this.userRating = ko.observable(rating || null);
}

function MyViewModel() {
    this.products = [
        new Product('Garlic bread'),
        new Product('Pain au chocolat'),
        new Product('Seagull spaghetti', 'like') // This one was already 'liked'
    ];
}

ko.applyBindings(new MyViewModel());
</script>
```

例2：从外部文件加载部件

一般组件的视图模型和模板是以单独文件的方式存在的。如果配置ko通过一个AMD模块加载器如require.js来获取它们，那么可以根据需要预加载（或许还打包或压缩了）或者立即加载。

将视图模型和模板的代码分别放到单独的文件中，"files/component-like-widget.js"和"files/component-like-widget.html"。

```js
ko.components.register('like-widget', {
    viewModel: { require: 'files/component-like-widget' },
    template: { require: 'text!files/component-like-widget.html' }
});
```

## 定义和注册组件

为了让ko能加载并实例化组件，需要使用`ko.components.register`来注册组件并提供相应的配置。
注：可以使用自定义的组件加载器代替配置。

### 将视图模型/模板对作为组件

按如下格式注册一个组件：

```js
ko.components.register('some-component-name', {
    viewModel: <see below>,
    template: <see below>
});
```

建议组件名为小写字母和连接号"-"组成，这样组件可以以自定义元素方式使用。
视图模型是可选的。
模板是必须的。

### 视图模型

视图模型有以下几种指定方式：

#### 构造器函数

```javascript
function SomeComponentViewModel(params) {
    // 'params' is an object whose key/value pairs are the parameters
    // passed from the component binding or custom element.
    this.someProperty = params.something;
}

SomeComponentViewModel.prototype.doSomething = function() { ... };

ko.components.register('my-component', {
    viewModel: SomeComponentViewModel,
    template: ...
});
```

ko在每次实例化组件时都会调用该构造器函数。结果对象或原型链上的属性在视图的绑定中都是可用的。

#### 共享的对象实例

如果希望所有组件实例共享同一个视图模型对象实例（通常不会希望这样）：

```js
var sharedViewModelInstance = { ... };

ko.components.register('my-component', {
    viewModel: { instance: sharedViewModelInstance },
    template: ...
});
```

#### `createViewModel`工厂函数

如果希望在关联的元素绑定到视图模型之前进行一些处理，或动态选择要实例化的视图模型类：

```js
ko.components.register('my-component', {
    viewModel: {
        createViewModel: function(params, componentInfo) {
            // - 'params' is an object whose key/value pairs are the parameters
            //   passed from the component binding or custom element
            // - 'componentInfo.element' is the element the component is being
            //   injected into. When createViewModel is called, the template has
            //   already been injected into this element, but isn't yet bound.
            // - 'componentInfo.templateNodes' is an array containing any DOM
            //   nodes that have been supplied to the component. See below.

            // Return the desired view model instance, e.g.:
            return new MyViewModel(params);
        }
    },
    template: ...
});
```

注意，要直接操作DOM元素时，最好使用自定义绑定而不是操作`createViewModel`内部的`componentInfo.element`。

`componentInfo.templateNodes`对构建可以接收任意html的组件很有用。（例如将提供的html注入grid，list，dialog或tab）

#### 值指向视图模型的AMD模块

如果页面中使用了AMD加载器（如require.js），可以使用它来获取视图模型。

```js
ko.components.register('my-component', {
    viewModel: { require: 'some/module/name' },
    template: ...
});
```

返回的AMD模块对象可以是任意形式的允许的视图模型（构造器函数、共享对象实例、`createViewModel`工厂函数、甚至另一个AMD模块对象的引用）。

```js
// AMD module whose value is a component viewmodel constructor
define(['knockout'], function(ko) {
    function MyViewModel() {
        // ...
    }

    return MyViewModel;
});
```

```js
// AMD module whose value is a shared component viewmodel instance
define(['knockout'], function(ko) {
    function MyViewModel() {
        // ...
    }

    return { instance: new MyViewModel() };
});
```

```js
// AMD module whose value is a 'createViewModel' function
define(['knockout'], function(ko) {
    function myViewModelFactory(params, componentInfo) {
        // return something
    }

    return { createViewModel: myViewModelFactory };
});
```

```js
// AMD module whose value is a reference to a different AMD module,
// which in turn can be in any of these formats
define(['knockout'], function(ko) {
    return { module: 'some/other/module' };
});
```

### 模板

模板有以下几种指定方式，最常用的是已存在的元素ID和AMD模块：

#### 已存在的元素ID

```html
<template id='my-component-template'>
    <h1 data-bind='text: title'></h1>
    <button data-bind='click: doSomething'>Click me right now</button>
</template>
```

```js
ko.components.register('my-component', {
    template: { element: 'my-component-template' },
    viewModel: ...
});
```

注意，指定的元素内部的元素才会被复制到组件的实例中，容器元素（`<template>`不会当做实例的一部分）。
也可以使用除了`<template>`的其他元素，但它是很方便的，因为浏览器不会渲染他们。

#### 已存在的元素实例

```js
var elemInstance = document.getElementById('my-component-template');

ko.components.register('my-component', {
    template: { element: elemInstance },
    viewModel: ...
});
```

#### html片段字符串

```js
ko.components.register('my-component', {
    template: '<h1 data-bind="text: title"></h1>\
               <button data-bind="click: doSomething">Clickety</button>',
    viewModel: ...
});
```

#### DOM节点数组

```js
var myNodes = [
    document.getElementById('first-node'),
    document.getElementById('second-node'),
    document.getElementById('third-node')
];

ko.components.register('my-component', {
    template: myNodes,
    viewModel: ...
});
```

#### 文档片段

如果拥有文档片段对象（DocumentFragment object），可以将它作为组件模板：

```js
ko.components.register('my-component', {
    template: someDocumentFragmentInstance,
    viewModel: ...
});
```

#### 值指向模板的AMD模块

当页面中使用AMD加载器时：

```js
ko.components.register('my-component', {
    template: { require: 'some/template' },
    viewModel: ...
});
```

返回的AMD模块对象可以是任何格式的允许的模板。因此也可以是一个html片段字符串，如require.js的text插件：

```js
ko.components.register('my-component', {
    template: { require: 'text!path/my-html-file.html' },
    viewModel: ...
});
```

### 指定额外的组件选项

就像`template`和`viewModel`，组件配置对象还可以有额外的任意属性。在自定义组件加载器时可以访问该配置对象。

#### 控制同步/异步加载

如果组件配置中有个bool值属性`synchronous`，ko会根据它决定组件是否允许被同步地加载和注入。默认值是`false`，强制异步。

```js
ko.components.register('my-component', {
    viewModel: { ... anything ... },
    template: { ... anything ... },
    synchronous: true // Injects synchronously if possible, otherwise still async
});
```

为什么默认为强制异步加载？

通常情况下，ko确保组件加载和注入为异步处理，是因为有时候除了异步加载别无选择（例如需要请求服务器）。即使一个组件的实例可以被同步注入，ko也还会异步处理（因为组件定义已经加载了）。这种始终异步处理策略是处于一致性考虑，也是其他异步js技术（如AMD）约定俗成的。该约定是个安全的默认项，因为它降低了开发者有时使用同步处理有时又使用异步处理的潜在bug。

为什么有时需要同步加载？

使用`synchonous: true`后，ko也许在第一次是异步加载，后续将都是同步加载。如果使用同步加载来，那就要考虑其它等待组件加载的代码。如果组件可以始终是被同步地加载和初始化，那么启用该选项是可以保证始终同步处理。这对于在`foreach`中使用组件并想用`afterAdd`或`afterRender`做后处理时很有用。

在ko v3.4.0版本之前，可以使用同步加载来阻止同时包含多个组件时（如foreach）的"multiple DOM reflow"。在v3.4.0版本中，组件使用ko的`microtasks`来确保异步能表现得跟同步一样好。

### ko如何通过AMD加载组件

当使用`require`声明来加载视图模型或模板时：

```js
ko.components.register('my-component', {
    viewModel: { require: 'some/module/name' },
    template: { require: 'text!some-template.html' }
});
```

ko所做的工作只是调用`require(['some/module/name'], callback)`和`require(['text!some-template.html'], callback)`，然后使用异步返回的对象作为视图模型和模板的定义。

这里并不严格依赖于某一个加载器，只要是提供了AMD类型的`require`API模块加载器都可以。如果使用其他API的模块加载器，需要实现自定义组件加载器。

ko不会推断模块的名称，只是将它传递给`require()`。

ko不知道也不关心AMD模块是否为匿名的。实际上将组件定义为匿名模块是最方便的，但这与ko是无关的。

AMD模块仅在需要时被加载

ko只会在组件即将实例化时调用`require([moduleName], ...)`，这就是组件会根据需要加载，而不是前置加载。
例如在if绑定中的某个元素内的组件，只会在if值为true时调用AMD模块。另外，当模块已被加载（比如，已在预加载包中），那么`require`调用时不会再请求http，因此可以根据需要控制什么被预加载和加载。

### 将组件组件注册为独立的AMD模块

为了更好地封装代码，可以将组件打包为独立的自描述的AMD模块，然后引用它：

```js
ko.components.register('my-component', { require: 'some/module' });
```

文件"some/module.js"可能为这样：

```js
// AMD module 'some/module.js' encapsulating the configuration for a component
define(['knockout'], function(ko) {
    function MyComponentViewModel(params) {
        this.personName = ko.observable(params.name);
    }

    return {
        viewModel: MyComponentViewModel,
        template: 'The name is <strong data-bind="text: personName"></strong>'
    };
});
```

最实用的创建组件的AMD模块是这样的，行内的视图模型类和AMD依赖外部模板文件。

```js
// Recommended AMD module pattern for a Knockout component that:
//  - Can be referenced with just a single 'require' declaration
//  - Can be included in a bundle using the r.js optimizer
define(['knockout', 'text!./my-component.html'], function(ko, htmlString) {
    function MyComponentViewModel(params) {
        // Set up properties, etc.
    }

    // Use prototype to declare any public methods
    MyComponentViewModel.prototype.doSomething = function() { ... };

    // Return component definition
    return { viewModel: MyComponentViewModel, template: htmlString };
});
```
这样做的好处是：
* 可以平凡地引用它：`ko.components.register('my-component', { require: 'path/my-component' });`。
* 组件只需要两个文件，视图模型文件（path/my-component.js）和模板文件（path/my-component.html）。
* 对模板的依赖在`define`中，这与`r.js optimizer`或其它打包工具是兼容的。视图模型和模板可以在构建时打包到同一个文件。

## 组件绑定

见“[绑定-流程控制]”中的“组件绑定”部分。

## 使用自定义元素

自定义元素提供了一种便捷的向视图注入组件的方式。

自定义元素是组件绑定的另一种语法选择。

```html
<!-- using component binding -->
<div data-bind='component: { name: "flight-deals", params: { from: "lhr", to: "sfo" } }'></div>

<!-- using custom element -->
<flight-deals params='from: "lhr", to: "sfo"'></flight-deals>
```

例：

```html
<h4>First instance, without parameters</h4>
<message-editor></message-editor>

<h4>Second instance, passing parameters</h4>
<message-editor params='initialText: "Hello, world!"'></message-editor>
```

```js
ko.components.register('message-editor', {
    viewModel: function(params) {
        this.text = ko.observable(params.initialText || '');
    },
    template: 'Message: <input data-bind="value: text" /> '
            + '(length: <span data-bind="text: text().length"></span>)'
});

ko.applyBindings();
```

### 向自定义元素传递参数

使用`params`属性向组件的视图模型传递参数。属性的内容对被解释为js对象字面量（就像`data-bind`属性）。

```html
<unrealistic-component
    params='stringValue: "hello",
            numericValue: 123,
            boolValue: true,
            objectValue: { a: 1, b: 2 },
            dateValue: new Date(),
            someModelProperty: myModelValue,
            observableSubproperty: someObservable().subprop'>
</unrealistic-component>
```

### 父子组件间通信

如果在`params`属性中引用模型属性，那么是在引用组件外部的视图模型属性，因为组件本身还未实例化。这就是如何从父视图模型向子组件传递属性，如果属性是可监控的，那么父视图模型就能接收自组件对它的修改。

### 传递监控表达式

```html
<some-component
    params='simpleExpression: 1 + 1,
            simpleObservable: myObservable,
            observableExpression: myObservable() + 1'>
</some-component>
```

上例的`params`包含三个值：
* 简单表达式：它将会是数值2。它不是监控属性或计算监控属性，因此不涉及监控。通常如果参数的评估不涉及监控评估，那它的值会被当做字面量传递。如果值是一个对象，那么子组件可以修改它，但父模型不会知道它改变了，因为它不是可监控的。
* 简单监控变量：它将会是父模型的`myObservable`的`ko.observable`实例。它不是包装过的，而是父模型引用的同一个实例。通常如果参数的评估不涉及监控评估（`myObservable`仅被传递但没有被评估），那它的值会被当做字面量传递。
* 监控表达式：当表达式被评估时读取了监控属性，那么表达式的值会随着监控属性的变化而变化。为了保证子组件可以处理表达式值的变化，ko会自动将其更新为计算监控属性。因此子组件可以用`params.observableExpression()`来读取当前值，或者使用`params.observableExpression.subscribe(...)`。

### 向组件传递html片段

有时候希望创建的组件可以接收html片段并作为输出的一部分。比如构建类似grid、list、dialog、tab这样的“容器”元素。

```html
<my-special-list params="items: someArrayOfPeople">
    <!-- Look, I'm putting markup inside a custom element -->
    The person <em data-bind="text: name"></em>
    is <em data-bind="text: age"></em> years old.
</my-special-list>
```

默认情况下，`<my-special-list>`内的节点会被删除（没有绑定视图模型时）且替换为组件的输出。但是这些节点没有丢失，有两种方式提供给组件：

* `$componentTemplateNodes`数组，对组件模板的任意绑定表达式中可用。这是常用的方式。
* `componentInfo.templateNodes`数组被传递给`createViewModel`函数。

模板可以将提供的DOM节点作为输出的一部分，例如在组件模板的元素上使用`template: { nodes: $componentTemplateNodes }`。

完整的代码示例如下：

```html
<!-- This could be in a separate file -->
<template id="my-special-list-template">
    <h3>Here is a special list</h3>

    <ul data-bind="foreach: { data: myItems, as: 'myItem' }">
        <li>
            <h4>Here is another one of my special items</h4>
            <!-- ko template: { nodes: $componentTemplateNodes, data: myItem } --><!-- /ko -->
        </li>
    </ul>
</template>

<my-special-list params="items: someArrayOfPeople">
    <!-- Look, I'm putting markup inside a custom element -->
    The person <em data-bind="text: name"></em>
    is <em data-bind="text: age"></em> years old.
</my-special-list>
```

```js
ko.components.register('my-special-list', {
    template: { element: 'my-special-list-template' },
    viewModel: function(params) {
        this.myItems = params.items;
    }
});

ko.applyBindings({
    someArrayOfPeople: ko.observableArray([
        { name: 'Lewis', age: 56 },
        { name: 'Hathaway', age: 34 }
    ])
});
```

注：在使用组件绑定而不是自定义元素时，同样可以传入html片段。

### 控制自定义元素的tag名称

默认情况下，ko将组件注册的名称当成自定义元素的名称。
可以通过重写`getComponentNameForNode`来为自定义元素重新命名。

```js
ko.components.getComponentNameForNode = function(node) {
    var tagNameLower = node.tagName && node.tagName.toLowerCase();

    if (ko.components.isRegistered(tagNameLower)) {
        // If the element's name exactly matches a preregistered
        // component, use that component
        return tagNameLower;
    } else if (tagNameLower === "special-element") {
        // For the element <special-element>, use the component
        // "MySpecialComponent" (whether or not it was preregistered)
        return "MySpecialComponent";
    } else {
        // Treat anything else as not representing a component
        return null;
    }
}
```

### 注册自定义元素

如果使用的是默认的组件加载器，那么通过`ko.components.register`注册自定义元素。

如果使用的是自定义的组件加载器，且没有使用`ko.components.register`，那么需要告诉ko想当成自定义元素使用的元素名称，同样是调用`ko.components.register`，但不需要指定任何配置等。

```js
ko.components.register('my-custom-element', { /* No config needed */ });
```

此外，还可以重写`getComponentNameForNode`来动态控制元素和组件名称的映射。

### 自定义元素和普通绑定结合

自定义元素除了`params`属性外，还可以有`data-bind`属性，

```html
<products-list params='category: chosenCategory'
               data-bind='visible: shouldShowProducts'>
</products-list>
```

但是像这样对自定义元素使用会修改元素内容的绑定是没有意义的，因为他们会覆盖组件注入的模板。

ko会阻止任何使用`controlsDescendantBindings`的绑定的这种用法，因为可能会在绑定它的视图模型到注入的模板时和组件发生冲突。因此在if和foreach中，必须使用包装的自定义元素，而不是直接使用。

```html
<!-- ko if: someCondition -->
    <products-list></products-list>
<!-- /ko -->
```

```html
<ul data-bind='foreach: allProducts'>
    <product-details params='product: $data'></product-details>
</ul>
```

### 自定义元素不会自关闭

必须这样写`<my-custom-element></my-custom-element>`，而不能这样写`<my-custom-element />`。

### IE6~8使用自定义元素

IE6~8支持自定义元素，但是必须在html解析器遇到它们之前注册，否则解析器会丢弃不识别的元素。为了保证自定义元素不被丢弃，需要确保在html解析器遇到`<your-component>`之前调用`ko.components.register('your-component')`，或者调用`document.createElement('your-component')`。

例如，页面的代码这样布局是可以的：

```html
<!DOCTYPE html>
<html>
    <body>
        <script src='some-script-that-registers-components.js'></script>

        <my-custom-element></my-custom-element>
    </body>
</html>
```

使用AMD时可能更希望是这样：

```html
<!DOCTYPE html>
<html>
    <body>
        <script>
            // Since the components aren't registered until the AMD module
            // loads, which is asynchronous, the following prevents IE6-8's
            // parser from discarding the custom element
            document.createElement('my-custom-element');
        </script>

        <script src='require.js' data-main='app/startup'></script>

        <my-custom-element></my-custom-element>
    </body>
</html>
```

如果不想使用`document.createElement`，可以在上一层的元素使用组件绑定代替自定义元素。只要所有组件在调用`ko.applyBindings`之前注册，在IE6~8上都不会有问题。

```html
<!DOCTYPE html>
<html>
    <body>
        <!-- The startup module registers all other KO components before calling
             ko.applyBindings(), so they are OK as custom elements on IE6-8 -->
        <script src='require.js' data-main='app/startup'></script>

        <div data-bind='component: "my-custom-element"'></div>
    </body>
</html>
```

### 访问`$raw`参数

考虑下面这种不常见的场景，`useObservable1`、`observable1`和`observable2都是监控属性：

```html
<some-component
    params='myExpr: useObservable1() ? observable1 : observable2'>
</some-component>
```

因为评估`myExpr`需要读取监控属性`useObservable1`，ko会将该参数作为计算监控属性提供给组件。然而，该计算属性的值本身也是可监控的。这貌似会导致一个尴尬的问题，读取它的值时会double-unwrapping（即`params.myExpr()()`），第一对括号是给出表达式的值，第二对括号给出监控属性实例的值。

这样的double-unwrapping既丑陋也不方便，也是不期望的。因此ko会自动设置生成的监控属性（param.myExpr）unwrap其值。

在一些不可能的事件中，不希望自动unwrap，因为希望直接访问`observable1`/`observable`，那么可以从`params.$raw`中读取其值。

```js
function MyComponentViewModel(params) {
    var currentObservableInstance = params.$raw.myExpr();

    // Now currentObservableInstance is either observable1 or observable2
    // and you would read its value with "currentObservableInstance()"
}
```

## 组件加载器

当使用组件绑定或自定义元素时注入组件时，ko使用一个或多个组件加载器来获取模板和视图模型。组件加载器的工作是异步提供给定名称的组件的模板/视图模型对。

### 默认的组件加载器

内建的组件加载器`ko.components.defaultLoader`是基于组件定义的“注册表”。它需要在使用组件之前明确地注册组件配置。

### 组件加载器实用函数

读写默认的组件加载器注册表：
* `ko.components.register(name, configuration)`：注册组件。
* `ko.components.isRegistered(name)`：返回指定的组件是否被注册。
* `ko.components.unregister(name)`：取消注册组件。

以下方法适用于所有注册的组件加载器列表：
* `ko.components.get(name, callback)`：依次询问注册的加载器，找到指定名称的组件的视图模型/模板定义，然后调用`callback`来返回该视图模型/模板定义。如果没有加载器找到组件则调用`callback(null)`。
* `ko.components.clearCachedDefinition(name)`：通常ko会对每个组件名称询问一次所有的加载器后将结果缓存起来。调用该函数可以清除某个组件的缓存，在下次需要该组件时重新询问加载器。

由于`ko.components.defaultLoader`也是一个加载器，所以它有以下方法可以调用（例如作为自定义加载器的一部分）：
* ko.components.defaultLoader.getConfig(name, callback)
* ko.components.defaultLoader.loadComponent(name, componentConfig, callback)
* ko.components.defaultLoader.loadTemplate(name, templateConfig, callback)
* ko.components.defaultLoader.loadViewModel(name, viewModelConfig, callback)

### 实现一个自定义组件加载器

如果希望使用一个命名规范来代替注册法加载组件，或使用第三方库来从外部文件加载视图模型和模板，可以实现一个自定义加载器。

自定义组件加载器时一个对象，其属性是以下函数的任意组合：

* `getConfig(name, callback)`
如果希望提供基于名称的代码的方式进行配置（例如，实现一个命名规范），那么定义该函数。
如果定义了该函数，那么ko在每次实例化组件之前会调用该函数来获取配置对象。
调用`callback(componentConfig)`来提供配置。`componentConfig`是任意可被加载器的`loadComponent`方法理解的对象。默认的加载器只会提供由`ko.components.register`注册的对象。
例如，`componentConfig`为`{ template: 'someElementId', viewModel: { require: 'myModule' } } `，可被默认的加载器理解和实例化。
如果希望自定义的加载器不提供指定名称的组件配置，那么调用`callback(null)`，ko会继续询问其它加载器，直到某个返回了非空的结果。

* `loadComponent(name, componentConfig, callback)`
如果希望控制组件配置是如何被解释的，那么定义该函数。
如果定义了该函数，那么ko会调用它将`componentConfig`对象转换成视图模型/模板对。
调用`callback(result)`来提供视图模型/模板对。`result`对象由两个属性：
`template`是必须的，是DOM节点数组；
`createViewModel(params, componentInfo)`是可选的，该函数会在后面被调用来为组件的每个实例提供视图模型对象。
调用`callback(null)`来让加载器不提供视图模型/模板对。ko会继续询问其他加载器直到某个返回了非空的结果。

* `loadTemplate(name, templateConfig, callback)`
使用自定义逻辑根据给定的模板配置提供节点（例如使用ajax请求获取模板）。
默认的加载器会调用其他注册了的且声明了该函数的加载器来调用该函数，将组件的模板配置转换成DOM节点数组，并缓存起来，被每个组件的是实例复制。
`templateConfig`值是任意`componentConfig`对象的`template`属性。
调用`callback(domNodeArray)`来提供DOM节点数组。
如果不希望返回结果（例如因为不识别配置格式），调用`callback(null)`。ko会继续询问其他加载器直到某个返回了非空的结果。

* `loadViewModel(name, templateConfig, callback)`
使用自定义逻辑根据给定的视图模型配置提供视图模型工厂。
默认的加载器会调用其他注册了的且声明了该函数的加载器来调用该函数，将组件的视图模型配置转换成视图模型工厂函数，并缓存起来，被每个需要视图模型的组件的是实例调用。
`viewModelConfig`值是任意`componentConfig`对象的`viewModel`属性。它可以是构造器函数，或者自定义格式，如`{ myViewModelType: 'Something', options: {} }`。
调用`callback(yourCreateViewModelFunction)`来提供视图模型工厂函数。视图模型工厂函数必须接收参数`(params, componentInfo)`，每次调用时异步返回一个视图模型实例。
如果不希望返回结果（例如因为不识别配置格式），调用`callback(null)`。ko会继续询问其他加载器直到某个返回了非空的结果。

### 注册自定义组件加载器

ko允许同时使用多个加载器。
`ko.components.loaders`数组包含当前激活的所有加载器。默认只包含`ko.components.defaultLoader`，要想添加其他加载器，把他们加到该数组中就行了。

### 控制优先级

通过改变加载器在`ko.components.loaders`数组中的位置来改变它们的优先级，越前面的优先级越高。

```js
// Adds myLowPriorityLoader to the end of the loaders array.
// It runs after other loaders, only if none of them returned a value.
ko.components.loaders.push(myLowPriorityLoader);

// Adds myHighPriorityLoader to the beginning of the loaders array.
// It runs before other loaders, getting the first chance to return values.
ko.components.loaders.unshift(myHighPriorityLoader)
```

如果需要，甚至可以将默认加载器移出数组。

### 调用顺序

ko第一次需要构建一个指定名称的组件时：
* 依次调用每个注册的加载器的`getConfig`函数，直到某个返回了非空的`componentConfig`。
* 使用`componentConfig`对象，依次调用每个注册的加载器的`loadComponent`函数，直到某个返回了非空的视图模型/模板对。

当默认加载器的`loadComponent`执行时，它同时：
* 依次调用每个注册的加载器的`loadTemplate`函数，直到某个返回了非空的DOM数组。默认加载器自身也有`loadTemplate`函数。
* 依次调用每个注册的加载器的`loadViewModel`函数，直到某个返回了非空的`createViewModel`函数。默认加载器自身也有`createViewModel`函数。

自定义加载器可以插入任何处理过程，如提供参数、解释参数、提供DOM节点、提供视图模型工厂函数。通过组件在`ko.components.loaders`数组中的顺序可以控制不同加载策略的优先级。

### 例1：设置命名规范的组件加载器

为了实现命名规范，自定义加载器只需要实现`getConfig`：

```js
ar namingConventionLoader = {
    getConfig: function(name, callback) {
        // 1. Viewmodels are classes corresponding to the component name.
        //    e.g., my-component maps to MyApp.MyComponentViewModel
        // 2. Templates are in elements whose ID is the component name
        //    plus '-template'.    
        var viewModelConfig = MyApp[toPascalCase(name) + 'ViewModel'],
            templateConfig = { element: name + '-template' };

        callback({ viewModel: viewModelConfig, template: templateConfig });
    }
};

function toPascalCase(dasherized) {
    return dasherized.replace(/(^|-)([a-z])/g, function (g, m1, m2) { return m2.toUpperCase(); });
}

// Register it. Make it take priority over the default loader.
ko.components.loaders.unshift(namingConventionLoader);
```

```html
<div data-bind="component: 'my-component'"></div>

<!-- Declare template -->
<template id='my-component-template'>Hello World!</template>

<script>
    // Declare viewmodel
    window.MyApp = window.MyApp || {};
    MyApp.MyComponentViewModel = function(params) {
        // ...
    }
</script>
```

### 例2：使用自定义代码从外部文件加载的组件

如果自定义组件实现了` loadTemplate`和/或`loadViewModel`，那么可以在加载过程中插入自定义代码。也可是使用这两个函数来解释自定义参数格式。

```js
ko.components.register('my-component', {
    template: { fromUrl: 'file.html', maxCacheAge: 1234 },
    viewModel: { viaLoader: '/path/myvm.js' }
});
```

```js
var templateFromUrlLoader = {
    loadTemplate: function(name, templateConfig, callback) {
        if (templateConfig.fromUrl) {
            // Uses jQuery's ajax facility to load the markup from a file
            var fullUrl = '/templates/' + templateConfig.fromUrl + '?cacheAge=' + templateConfig.maxCacheAge;
            $.get(fullUrl, function(markupString) {
                // We need an array of DOM nodes, not a string.
                // We can use the default loader to convert to the
                // required format.
                ko.components.defaultLoader.loadTemplate(name, markupString, callback);
            });
        } else {
            // Unrecognized config format. Let another loader handle it.
            callback(null);
        }
    }
};

// Register it
ko.components.loaders.unshift(templateFromUrlLoader);
```

```js
var viewModelCustomLoader = {
    loadViewModel: function(name, viewModelConfig, callback) {
        if (viewModelConfig.viaLoader) {
            // You could use arbitrary logic, e.g., a third-party
            // code loader, to asynchronously supply the constructor.
            // For this example, just use a hard-coded constructor function.
            var viewModelConstructor = function(params) {
                this.prop1 = 123;
            };

            // We need a createViewModel function, not a plain constructor.
            // We can use the default loader to convert to the
            // required format.
            ko.components.defaultLoader.loadViewModel(name, viewModelConstructor, callback);
        } else {
            // Unrecognized config format. Let another loader handle it.
            callback(null);
        }
    }
};

// Register it
ko.components.loaders.unshift(viewModelCustomLoader);
```

如果你乐意，也可以将`templateFromUrlLoader`和`viewModelCustomLoader`合并为一个。

### 注：自定义组件加载器和自定义元素

如果使用自定义加载器来来用命名规范方式获取组件，且没有用`ko.components.register`来注册组件，那么这些组件默认是不能当作自定义元素使用的。



---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/component-overview.html)
