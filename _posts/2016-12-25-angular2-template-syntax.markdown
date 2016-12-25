---
layout:     post
title:      "[Angular2] Template Syntax"
subtitle:   ""
date:       2016-12-25 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 让我们看看如何编写模板来展示数据和利用数据绑定处理用户事件。

Angular应用通过组件类实例和它的面向用户的模板来管理用户可见的和可操作的内容。

很多小伙伴都熟悉MVC或MVVM模式。在Angular中，组件就相当于controller/viewmodel的角色，模板就是view。

### HTML

HTML是Angular模板的语言。几乎所有的HTML语法都是有效的模板语法。

有些HTML元素在模板中没有什么意义，如`<html>`、`<body>`、`<base>`。

`<script>`在Angular模板中是不允许使用的，为了避免脚本注入风险。

我们可以使用组件和指令作为新的元素和属性来扩展HTML词汇。

### Interpolation（插值绑定）

插值绑定是单向数据绑定方式。可以在元素内或者元素的属性内绑定：

{% raw %}
```html
<p>My name is {{currentUser.firstName}}</p>
```
{% endraw %}

{% raw %}
```html
<h3>
  {{title}}
  <img src="{{userImageUrl}}" style="height:30px">
</h3>
```
{% endraw %}

双括号内通常是组件属性的名称。Angular会使用对应的组件属性值来替换双括号内的名称。

事实上，双括号内是模板表达式（template expression），Angular会先计算它再转换成字符串。

{% raw %}
```html
<!-- "The sum of 1 + 1 is 2" -->
<p>The sum of 1 + 1 is {{1 + 1}}</p>
```
{% endraw %}

表达式中还可以调用组件的方法：

{% raw %}
```html
<!-- "The sum of 1 + 1 is not 4" -->
<p>The sum of 1 + 1 is not {{1 + 1 + getVal()}}</p>
```
{% endraw %}

Angular首先计算双括号内的表达式的值，然后将结果转换成字符串，将它连接到相邻的字面量字符串中。最后，将组成的插值结果赋给一个元素或指令属性。

看起来我们是将结果添加到元素标签内和赋值给属性。这样想也不会有问题。但事实上不是这样的。插值绑定是个特殊的语法，Angular会将它转换成属性绑定。

### Template expression（模板表达式）

模板表达式能计算出一个值。Angular执行表达式并将它赋给属性绑定的目标上，这个绑定目标可以是HTML元素、组件或指令。

我们将模板表达式放在插值绑定的双大括号内，就像`{{1+1}}`；或者在属性绑定中等号右边的引号内，`[property]="expression"`。

模板表达式看起来像是JavaScript表达式，但并不是所有的JavaScript表达式都是合法的模板表达式。具有副作用的JavaScript表达式是被禁止的，包括：
* 赋值表达式（`=`、`+=`、`-=`）
* new操作符
* 有`;`或者`,`的链式表达式
* 自增和自减操作符（`++`、`--`）

还有跟JavaScript表达式不一样的地方有：
* 不支持位运算符（`|`、`&`）
* 新的模板表达式操作符（如`|`、`?`）

#### 表达式上下文

模板表达式中不能访问全局命名空间，如`window`和`document`。也不能调用`console.log`或`Math.max`。它们被限制只能引用表达式上下文的成员。

表达式的上下文是组件实例，也就是绑定值的来源。

当我们看到`{{title}}`和`[disabled]="isUnchanged"`，我们就知道`title`和`isUnchanged`是数据绑定组件的属性。

组件自身通常就是表达式的上下文，也就是模板表达式中通常引用的就是这个组件。

模板表达式还可以访问其它对象，例如模板引用变量（template reference variable）。

#### 表达式注意事项

使用模板表达式的时候要注意：

1.没有明显的副作用

模板表达式除了改变绑定属性的值不能应用的其它状态。

这条规则对Angular的单向数据流策略（unidirectional data flow）很重要。我们不需要担心在读取组件值的时候会改变其它的显示值。在一次渲染过程中视图是稳定的。

2.能快速计算

Angular会经常计算模板表达式，比我们想象的更频繁。

按下按键或者移动鼠标都可能会导致重新计算表达式。因此表达式要能快速计算。否则用户体验会很差。可以考虑很复杂的计算中尽量使用其他数据的缓存值。
3.保持简单

尽量不要在模板表达式中包含复杂的逻辑。

通常只是属性名或者要调用的方法，或者包含`!`这样的很简单的运算。另外，将应用和业务逻辑放到组件内，这样也更便于开发和测试。

4.幂等性

幂等表达式是最理想的，因为它完全没有副作用，并且可以提高Angular变化检测的性能。

在Angular中，幂等表达式的值只会在它所依赖的值变化时才变化。

依赖的值在单次事件循环中不应该有变化。如果一个幂等表达式返回一个字符串或一个数字，那么在一行中调用它两次返回的值应该是一样的。如果表达式返回的是对象，那么一行中调用他两次应该返回同一个对象引用。

## Template statement（模板声明？）

模板声明对应绑定一个元素、组件或指令的绑定目标触发的事件。

例如事件绑定`(event)="statement"`中等号右边引号内的部分。

模板声明是有副作用的，也就是Angular如何从用户输入更新应用状态的。

对事件的响应是Angular单向数据流的另一个副作用。我们可以在单次事件循环中改变根和地方的任何值。

跟模板表达式相似，模板声明也是看起来像JavaScript表达式。与模板表达式不同的是，模板声明支持赋值运算符`=`和使用`;`和`,`的链式表达式。它同样不支持以下几个：
* new操作符
* 自增和自减操作服（`++`、`--`）
* 赋值表达式（`+=`、`-=`）
* 位运算符（`|`、`&`）
* 新的模板表达式操作符（如`|`、`?`）

#### 声明上下文

跟模板表达式一样，模板声明只能引用它的上下文的内容，通常也就是绑定的事件的那个组件实例。

模板声明不能访问全局命名空间。如`window`和`document`。也不能调用`console.log`或`Math.max`。

模板声明的上下文是绑定事件的组件实例。`(click)="onSave()"`中的`onSave`肯定就是绑定的组件实例的一个方法。

模板声明的上下文除了组件可能还有其他对象，如模板引用变量。我们会经常看到`$event`符号。

跟模板表达式一样，建议避免使用复杂的模板声明。通常是调用一个方法或简单的属性绑定。

### Binding syntax（绑定语法）

数据绑定是协调用户看到的和应用的数据值的一种机制。我们可以自己读写表单HTML的值，但是将这个工作交给框架来完成。我们只需要在绑定源和目标HTML元素岁之间声明绑定关系就行了。

数据绑定分为三种：

* 从数据源到目标视图的单向数据绑定，绑定类型有Interpolation、Property、Attribute、Class、Style。
```js
{{expression}}
[target] = "expression"
bind-target = "expression"
```
* 从目标视图数据源到数据源的单向数据绑定，绑定类型有Event。
```js
(target) = "statement"
on-target = "statement"
```
* 双向数据绑定。绑定类型有Two-way。
```js
[(target)] = "expression"
bindon-target = "expression"
```

除了插值绑定，其它绑定类型都有一个在等号左侧的目标名称，要么放在中括号或小括号内（`[]`、`()`），要么就有个前缀（`bind`、`on`、`bindon-`）。

什么是绑定目标？在回答这个问题之前，我们先从另一个角度看看模板HTML。

#### 一个新的思维方式

数据绑定加上我们可以创建自定义标签来扩展HTML词汇，我们可以将模板HTML看成加强版的HTML。

在标准的HTML开发中，我们使用HTML元素来创建可视化结构，通过设置元素的属性来修改元素：

```html
<div class="special">Mental Model</div>
<img src="images/user.png">
<button disabled>Save</button>
```

在Angular模板中，我们还是用这种方式创建结构和初始化属性值。

再看我们使用组件来创建新元素的方式，我们将HTML封装起来放到模板中，然后组件看起来就像是一个普通的HTML元素。

```html
<!-- Normal HTML -->
<div class="special">Mental Model</div>
<!-- Wow! A new element! -->
<user-detail></user-detail>
```

这就是加强版的HMTL。

现在我们再看数据绑定：

```html
<!-- Bind button disabled state to `isUnchanged` property -->
<button [disabled]="isUnchanged">Save</button>
```

直觉告诉我们，这句代码的意思是将按钮的`disabled`属性的当前值设置为组件的`isUnchanged`属性值。

然而并不是这样，我们被大脑内根深蒂固的HTML思维误导了。当我们使用数据绑定时，我们使用的就不再是HTML属性了。我们不是在设置属性（attribute），而是设置DOM元素、组件和指令的属性（property）。

#### HTML attribute vs. DOM property

Attribute是针对HTML的，Propertiy是针对DOM的。

一些HTML attribute和DOM property是一一对应的，例如`id`。

一些HTML attribute没有对应的DOM property，例如`colspan`。

一些DOM property没有对应的HTML attribute，例如`textContext`。

HTML attribute用于初始化DOM property，之后就没有它们什么事了。DOM property可以改变，而HTML attribute的值不会变。

例如，当浏览器渲染`<input type="text" value="Tom">`，浏览器会创建一个相应的DOM节点，节点的`value`属性的值初始化为"Tom"。当用户向文本框中输入"Jerry"，DOM元素的value属性值为"Jerry"，而HTML的value属性还是"Tom"，使用`input.getAttribute('value')`可以看到。

HTML attribute的值是指定初始值，DOM property的值是当前值。

disabled是个特例。一个按钮的disabled属性默认为false，按钮是启用的。当我们添加了disabled属性，它的disabled属性就被初始化为true，按钮被禁用。也就是说，通过添加或移除disabled属性来禁用或启用按钮。该属性的值是不可变的，因此我们不能通过`disabled="false"`来启用按钮。

设置按钮的disabled property来禁用或启用按钮，只有property的值才能变化并起作用。

HTML attribute和DOM property不是同一个东西，即使它们的名称是一样的。

总之，模板绑定使用的是property和event，不是attribute。在Angular2中，attribute唯一的作用就是初始化元素和指令的状态。当我们绑定数据时，我们使用的是元素和指令的属性以及事件。

#### Binding target（绑定目标）

数据绑定的目标是DOM中的一些东西。它可以是（元素/组件/指令）property、（元素/组件/指令）event或者（极少见的）attribute名称。

根据绑定类型的不同分为以下几种：

Property绑定
```html
<!--Element property-->
<img [src] = "userImageUrl">

<!--Component property-->
<user-detail [user]="currentUser"></user-detail>

<!--Directive property-->
<div [ngClass] = "{selected: isSelected}"></div>
```
Event绑定
```html
<!--Element event-->
<button (click) = "onSave()">Save</button>

<!--Component event-->
<user-detail (deleteRequest)="deleteUser()"></user-detail>

<!--Directive event-->
<div (myClick)="clicked=$event">click me</div>
```
Two-way绑定
```html
<!--Event and property-->
<input [(ngModel)]="userName">
```
Attribute绑定
```html
<!--Attribute (the exception)-->
<button [attr.aria-label]="help">help</button>
```
Class绑定
```html
<!--class property-->
<div [class.special]="isSpecial">Special</div>
```
Style绑定
```js
<!--style property-->
<button [style.color] = "isSpecial ? 'red' : 'green'">
```

---

参考资料：

[Angular2 Documentation - Template Syntax](https://angular.io/docs/ts/latest/guide/template-syntax.html)
