---
author: wukn
catalog: true
date: 2016-12-25T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Template Syntax'
url: /2016/12/25/angular2-template-syntax/
---

> 让我们看看如何编写模板来展示数据和利用数据绑定处理用户事件。

<!--more-->

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

{% raw %}
```js
{{expression}}
[target] = "expression"
bind-target = "expression"
```
{% endraw %}

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

### Property binding（属性绑定）

当我们需要将一个视图元素的属性设置为一个模板表达式的值时，我们使用属性绑定。

最常见的属性绑定就是将元素的属性设置为组件的属性值：

```html
<img [src]="userImageUrl">
```

另一个例子是禁用按钮：

```html
<button [disabled]="isUnchanged">Cancel is disabled</button>
```

或者是设置指令的属性：

```html
<div [ngClass]="classes">[ngClass] binding to the classes property</div>
```

或者是自定义组件的模型属性（也是父子组件交互的一种方式）：

```js
<user-detail [user]="currentUser"></user-detail>
```

人们通常将属性绑定称为单向数据绑定，因为数据是从组件的属性流向目标元素的属性。我们无法通过属性绑定来读取一个目标元素的值。

同样我们也无法通过属性绑定来调用目标元素的事件。如果元素触发了事件，我们可以用事件绑定来处理它。如果我们必须读取一个目标元素的属性或调用它的方法，需要借助一些其它的手段，例如[ViewChild](https://angular.io/docs/ts/latest/api/core/index/ViewChild-decorator.html)和[ContentChild](https://angular.io/docs/ts/latest/api/core/index/ContentChild-decorator.html)。

#### 绑定目标

中括号内的名称指定了绑定的目标属性，例如设置img的src属性：

```html
<img [src]="userImageUrl">
```

除了中括号，还可以使用`bind-`前缀，这是标准模式：

```html
<img bind-src="userImageUrl">
```

目标名称总是property名称。我们看到`src`会认为它是attribute名称，实际上是img元素property的名称。

元素属性是最常见的目标，但是Angular会先在已知的指令中查找是否有该名称的属性。如果在已知指令或元素的属性都没有找到该名称的属性，Angular会返回一个`unknown directive`错误。

记得避免使用有副作用的模板表达式。如果我们在模板表达式中使用了`getFoo()`，我们自己需要知道这个方法里面到底做了什么，如果这个方法里修改了其它绑定项的值，那这就不好了。

使用属性绑定时，要注意模板表达式返回正确的数据类型，也就是说模板表达式计算得到的值的类型应该和目标属性值类型一样。

例如。`UserDetail`的`user`属性，期望的是一个`User`对象，而我们属性绑定的正好是：

```html
<user-detail [user]="currentUser"></user-detail>
```

别忘了属性绑定的中括号。如果忘记了中括号，Angular会把字符串作为常量，并且用该字符串初始化目标属性，而不是计算字符串的值。

```html
<!--
  BAD!
  UserDetailComponent.User expects a User object,
  not the string "currentUser"
 -->
<user-detail user="currentUser"></user-detail>
```

如果我们想要的是这样的效果，那么去掉中括号是完全可以的：

* 目标寺属性接受的值是字符串类型
* 这个字符串值是我们想在模板里使用的一个固定值
* 这个初始值永远不会改变

这种属性初始化方式对组件或指令也是适用的。例如将`UserDetailComponent`组件的`prefix`属性初始化为一个固定的字符串：

```html
<user-detail prefix="The name is " [user]="currentUser"></user-detail>
```

#### 选择属性绑定还是插值绑定

通常我们在属性绑定和插值绑定之间二选一，它们俩的作用是一样的：

{% raw %}
```html
<p><img src="{{userImageUrl}}"> is the <i>interpolated</i> image.</p>
<p><img [src]="userImageUrl"> is the <i>property bound</i> image.</p>

<p><span>"{{title}}" is the <i>interpolated</i> title.</span></p>
<p>"<span [innerHTML]="title"></span>" is the <i>property bound</i> title.</p>
```
{% endraw %}

事实上Angular在渲染视图之前会吧插值绑定转换成属性绑定。

#### 内容安全问题

假设有这样个变量：

```javascript
evilTitle = 'Template <script>alert("evil never sleeps")</script>Syntax';
```

Angular在数据绑定的时候会对危险的HTML发出告警。它会在显示值之前进行处理，不会让script标签泄露到HTML中，无论是插值绑定还是属性绑定都不会。

{% raw %}
```html
<p><span>"{{evilTitle}}" is the <i>interpolated</i> evil title.</span></p>
<p>"<span [innerHTML]="evilTitle"></span>" is the <i>property bound</i> evil title.</p>
```
{% endraw %}

![](/img/post/angular2/template-syntax/evil-title.png)

### Attribute, class, and style binding

模板语法还提供了一些特殊的单向绑定。

#### Attribute binding

我们可以使用attribute绑定来设置attribute的值。(前面说到绑定的目标是property，这里是唯一一个例外。)

上一篇已经说了attribute和property的区别。为什么Angular还要提供attribute绑定呢？因为有的attribute没有对应的property。例如aria，svg，table的span，它们是纯attribute，没有对应的元素属性。

如果我们这样做：

{% raw %}
```html
<tr><td colspan="{{1 + 1}}">Three-Four</td></tr>
```
{% endraw %}

会得到这样的错误：
```
Template parse errors:
Can't bind to 'colspan' since it isn't a known native property
```

因为`<td>`元素没有名为`colspan`的property。它有`colspan`attribute，但是插值绑定和property绑定只会设置property，而不是attribute。

因此，我们需要attribute绑定。

attribute绑定跟property绑定是相似的。只是中括号内不是property名称，而是加了前缀`attr.`的attribute名称。

```html
<tr><td [attr.colspan]="1 + 1">One-Two</td></tr>
```

最主要的使用场景就是设置ARIA属性：

{% raw %}
```html
<table border=1>
  <!--  expression calculates colspan=2 -->
  <tr><td [attr.colspan]="1 + 1">One-Two</td></tr>

  <!-- ERROR: There is no `colspan` property to set!
    <tr><td colspan="{{1 + 1}}">Three-Four</td></tr>
  -->

  <tr><td>Five</td><td>Six</td></tr>
</table>
```
{% endraw %}

最主要的一个应用场景是绑定ARIA的attribute：

{% raw %}
```html
<!-- create and set an aria attribute for assistive technology -->
<button [attr.aria-label]="actionName">{{actionName}} with Aria</button>
```
{% endraw %}

### Class binding

我们使用class绑定来为元素的class属性添加或移除CSS类。

class绑定也是跟property绑定相似的。中括号内是`class`或者是加了前缀`class.`的CSS类名称：`[class]`、`[class.class-name]`。

`[class]`这种方式是根据模板表达式的值添加或者移除所有的CSS类。

```html
<!-- reset/override all class names with a binding  -->
<div class="bad curly special"
     [class]="badCurly">Bad curly</div>
```

也可以指定类名：

```html
<!-- toggle the "special" class on/off with a property -->
<div [class.special]="isSpecial">The class binding is special</div>

<!-- binding to `class.special` trumps the class attribute -->
<div class="special"
     [class.special]="!isSpecial">This one is not so special</div>
```

如果要同时管理多个CSS类，更好的做法是使用`NgClass`指令。

#### Style binding

我们可以使用style绑定来设置内联样式。

style绑定也是跟property绑定相似的。中括号内是加了前缀`style.`的CSS样式属性名称：`[style.style-property]`。

```html
<button [style.color] = "isSpecial ? 'red' : 'green'">Red</button>
<button [style.background-color]="canSave ?'cyan' : 'grey'" >Save</button>
```

有的style绑定有单位。

```html
<button [style.fontSize.em]="isSpecial ? 3 : 1" >Big</button>
<button [style.fontSize.%]="!isSpecial ? 150 : 50" >Small</button>
```

如果要同时管理多个内联样式，更好的做法是使用`NgStyle`指令。

### Event binding（事件绑定）

事件绑定是从视图到组件的单向数据绑定。

事件绑定语法由等号左边的放在小括号内的目标事件和等号右边的放在引号内的模板声明两部分组成组成。例如将按钮的点击事件绑定到组件的`onSave()`方法：

```html
<button (click)="onSave()">Save</button>
```

#### 目标事件

小括号内的名称表明目标事件。也可以使用`on-`前缀，这是标准形式：

```html
<button on-click="onSave()">On Save</button>
```

元素的事件是最常见的绑定目标，但是Angular会先查找已知的指令是否有该名称的事件属性。

```html
<!-- `myClick` is an event on the custom `ClickDirective` -->
<div (myClick)="clickMessage=$event">click with myClick</div>
```

如果没有匹配的元素事件或已知指令的事件属性，Angular会返回`unknown directive`错误。

#### $event和事件处理声明

在事件绑定中，Angular为目标事件设置事件处理器。当事件触发了，事件处理器会执行模板声明。

事件绑定使用一个名为`$event`的对象承载事件的相关信息，包括数据值。`$event`对象的类型取决于目标事件。如果目标事件是DOM元素事件，那`$event`就是DOM事件对象，有诸如`target`和`target.value`这样的属性。

例如：

```html
<input [value]="currentUser.firstName"
       (input)="currentUser.firstName=$event.target.value" >
```

这段代码将input的`value`属性绑定到`firstName`属性。为了监听值的变化，代码里绑定了input的`input`事件。当用户做出修改操作时，`input`事件被触发，绑定的声明会在包含DOM事件对象的上下文中执行，也就是`$event`。

如果绑定事件属于指令（记住，组件也是指令哦），那么`$event`取决于指令。

#### 使用EventEmitter自定义事件

指令通常使用EventEmitter来触发自定义事件。每个指令会创建一个`EventEmitter`并将它作为属性暴露出来。组件调用`EventEmitter.emit(payload)`来发起一个事件，并传入一个消息。父指令通过绑定该属性监听该事件，并通过`$event`对象访问传递的参数`payload`。

例如，`UserDetailComponent`展示user的信息并响应用户的操作。尽管`UserDetailComponent`有个删除按钮，但是它不知道如何删除一个user。最好的做法是它发起一个事件来处理用户的删除请求。

`UserDetailComponent`的部分代码如下：

{% raw %}
```html
template: `
<div>
  <img src="{{userImageUrl}}">
  <span [style.text-decoration]="lineThrough">
    {{prefix}} {{user?.fullName}}
  </span>
  <button (click)="delete()">Delete</button>
</div>`
```
{% endraw %}

```js
// This component make a request but it can't actually delete a user.
deleteRequest = new EventEmitter<User>();

delete() {
  this.deleteRequest.emit(this.user);
}
```

假设一个托管父组件绑定了`UserDetailComponent`的`deleteRequest`事件：

```html
<user-detail (deleteRequest)="deleteUser($event)" [user]="currentUser"></user-detail>
```

当发起`deleteRequest`时，Angular调用父组件的`deleteUser`方法，使用`$event`变量传递要删除的user。

### 双向数据绑定

通常我们希望既显示数据属性又接受用户对数据的修改。从元素的角度来看就是将设置属性值和监听元素变化事件结合起来。

Angular提供了一个特殊的双向数据绑定语法，`[(x)]`。就是将属性绑定和中括号和事件绑定的小括号结合起来了。

当元素有一个可设置的属性x和对应的事件xChange时，这个语法很容易理解。

假设`SizerComponent`有一个`size`属性和`sizeChange`事件：

{% raw %}
```js
import { Component, EventEmitter, Input, Output } from '@angular/core';
@Component({
  selector: 'my-sizer',
  template: `
  <div>
    <button (click)="dec()" title="smaller">-</button>
    <button (click)="inc()" title="bigger">+</button>
    <label [style.font-size.px]="size">FontSize: {{size}}px</label>
  </div>`
})
export class SizerComponent {
  @Input()  size: number | string;
  @Output() sizeChange = new EventEmitter<number>();
  dec() { this.resize(-1); }
  inc() { this.resize(+1); }
  resize(delta: number) {
    this.size = Math.min(40, Math.max(8, +this.size + delta));
    this.sizeChange.emit(this.size);
  }
}
```
{% endraw %}

初始`size`值是来自属性绑定。点击按钮调整`size`的值，在最大值最小值的限制内，并在调整值的时候触发`sizeChange`事件。

下面的`AppComponent.fontSizePx`是`SizerComponent`的双向数据绑定：

```html
<my-sizer [(size)]="fontSizePx"></my-sizer>
<div [style.font-size.px]="fontSizePx">Resizable Text</div>
```

在最开始`AppComponent.fontSizePx`会作为`SizerComponent.size`的初始值。点击按钮可以通过双向数据绑定更新`AppComponent.fontSizePx`的值。而`AppComponent.fontSizePx`又通过Style绑定作用到div。

双向数据绑定只是属性绑定加事件绑定的一个语法糖。去掉语法糖就是这个样子：

```html
<my-sizer [size]="fontSizePx" (sizeChange)="fontSizePx=$event"></my-sizer>
```

`$event`包含`SizerComponent.sizeChange`事件。当点击按钮时Angular会将`$event`值赋给`AppComponent.fontSizePx`。

我们希望在input或select这样的HTML元素上使用双向数据绑定。但是很不幸，HTML元素不遵循`x`值和`xChange`这样的范式。

好消息是我们可以使用NgModel指令。

### NgModel双向数据绑定

NgModel属性用于双向数据绑定，既显示数据属性又接受用户的改变：

```html
<input [(ngModel)]="currentUser.firstName">
```

也可以使用前缀方式：

```html
<input bindon-ngModel="currentUser.firstName">
```

在使用`ngModel`之前，我们要引入`FormsModule`并添加到模块的`imports`列表中。

```js
// app/app.module.ts

import { NgModule } from '@angular/core';
import { BrowserModule }  from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';

@NgModule({
  imports: [
    BrowserModule,
    FormsModule
  ],
  declarations: [
    AppComponent
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

其实我们可以使用`<input>`元素的`value`属性和`input`事件实现跟`ngModel`相同的效果：

```html
<input [value]="currentUser.firstName"
       (input)="currentUser.firstName=$event.target.value" >
```

这样子太繁琐了。我们需要记住元素要设置的属性和监听的事件。`ngModel`指令隐藏了这些细节。`ngModel`包装了元素的`value`属性，监听`input`事件，并且提供了自己的`ngModel`输入属性和`ngModelChange`输出属性。

```html
<input
  [ngModel]="currentUser.firstName"
  (ngModelChange)="currentUser.firstName=$event">
```

`ngModel`数据属性设置元素的value属性，`ngModelChange`事件属性监听元素value的变化。

具体的实现各个元素不相同。因此`NgModel`指令只适用于特定的元素，例如input。

自定义组件是不能使用`[(ngModel)]`的，除非我们为它写了合适的值存取器（value accessor）。

这个还有改进的地方。为什么我们要绑定两次呢？Angular应该能做到用一个声明来读取和设置组件书数据属性，那就是`[(ngModel)]`：

```html
<input [(ngModel)]="currentUser.firstName">
```

在内部，Angular会将`ngModel`映射为`ngModel`输入属性和`ngModelChange`输出属性。

当然，在一些场景下还是有理由将`[(ngModel)]`拆成两个绑定的。例如，强制将用户的输入转换成大写：

```html
<input
  [ngModel]="currentUser.firstName"
  (ngModelChange)="setUpperCaseFirstName($event)">
```

### 内建指令

下面是一些常见的内建指令。

#### NgClass

我们通常通过添加或移除CSS类来控制元素的外观。我们可以绑定`NgClass`来同时添加或移除多个CSS类。

之前提到过使用class绑定来添加或移除单个CSS类：

```html
<!-- toggle the "special" class on/off with a property -->
<div [class.special]="isSpecial">The class binding is special</div>
```

而`NgClass`指令可以一次添加或移除多个CSS类。使用`NgClass`的方式是为它绑定一个key-value对象。对象的key是CSS类名；value为true时添加该类，为false时移除该类。

假设组件提供了`setClasses`方法管理CSS类的状态：

```js
setClasses() {
  let classes =  {
    saveable: this.canSave,      // true
    modified: !this.isUnchanged, // false
    special: this.isSpecial,     // true
  }
  return classes;
}
```

将`NgClass`属性绑定到`setClasses`方法：

```html
<div [ngClass]="setClasses()">This div is saveable and special</div>
```

#### NgStyle

之前提到过使用style绑定来添加或移除单个样式：

```html
<div [style.fontSize]="isSpecial ? 'x-large' : 'smaller'" >
  This div is x-large
</div>
```

而`NgStyle`指令可以一次添加或移除多个内联样式。使用`NgStyle`的方式是为它绑定一个key-value对象。对象的key是样式名称；value就是样式的值。

假设组件提供了`setStyles`方法管理样式：

```js
setStyles() {
  let styles = {
    // CSS property names
    'font-style':  this.canSave      ? 'italic' : 'normal',  // italic
    'font-weight': !this.isUnchanged ? 'bold'   : 'normal',  // normal
    'font-size':   this.isSpecial    ? '24px'   : '8px',     // 24px
  }
  return styles;
}
```

将`NgStyle`属性绑定到`setStyles`方法：

```html
<div [ngStyle]="setStyles()">
  This div is italic, normal weight, and extra large (24px)
</div>
```

#### NgIf

NgIf指令可以控制是否向DOM添加或移除一个元素。绑定的表达式值为真值时添加元素，为假值时移除元素。

{% raw %}
```html
<div *ngIf="currentUser">Hello, {{currentUser.firstName}}</div>
```
{% endraw %}

{% raw %}
```html
<!-- not displayed because nullUser is falsey.
    `nullUser.firstName` never has a chance to fail -->
<div *ngIf="nullUser">Hello, {{nullUser.firstName}}</div>

<!-- User Detail is not in the DOM because isActive is false-->
<user-detail *ngIf="isActive"></user-detail>
```
{% endraw %}

我们可以也使用class绑定或style绑定控制显示或隐藏DOM元素：

```html
<!-- isSpecial is true -->
<div [class.hidden]="!isSpecial">Show with class</div>
<div [class.hidden]="isSpecial">Hide with class</div>

<!-- UserDetail is in the DOM but hidden -->
<user-detail [class.hidden]="isSpecial"></user-detail>

<div [style.display]="isSpecial ? 'block' : 'none'">Show with style</div>
<div [style.display]="isSpecial ? 'none'  : 'block'">Hide with style</div>
```

但是隐藏DOM元素跟使用`NgIf`移除是不一样的。当我们隐藏DOM元素时，元素仍然在DOM中。相应的组件都是存在的，保留着它们的状态，仍然消耗着内存，Angular可能会继续检查它的属性的变化。而`NgIf`值为false时，Angular会将元素中DOM中移除，释放掉组件，不再占用资源。

#### NgSwitch

我们使用`NgSwitch`指令来显示多个元素中的一个。

```html
<span [ngSwitch]="toeChoice">
  <span *ngSwitchWhen="'Eenie'">Eenie</span>
  <span *ngSwitchWhen="'Meanie'">Meanie</span>
  <span *ngSwitchWhen="'Miney'">Miney</span>
  <span *ngSwitchWhen="'Moe'">Moe</span>
  <span *ngSwitchDefault>other</span>
</span>
```

注意，`ngSwitch`用的是属性绑定，不用加`*`，而`ngSwitchWhen`和`ngSwitchDefault`前面是加了`*`的。

`ngSwitch`跟`ngIf`类似，最多只会在DOM中显示一个span，其它的会被移除掉，而不是隐藏起来。

#### NgFor

`NgFor`是一个repeater指令。

我们的目标是现实一个列表，我们用一个HTML片段定义单个列表项如何显示，然后告诉Angular用这个HTML片段作为模板来渲染列表中的每一项。

例如，对`<div>`使用`NgFor`：

{% raw %}
```html
<div *ngFor="#user of useres">{{user.fullName}}</div>
```
{% endraw %}

也可以对组件元素使用`NgFor`：

```html
<user-detail *ngFor="let user of useres" [user]="user"></user-detail>
```

##### NgFor微语法

赋值给`*ngFor`的字符串不是模板表达式，而是一个微语法（microsyntax）。

例子中的`let user of useres`的意思是：对`useres`数组中的每一项，将它赋值给本地变量`user`，`user`在每次迭代的HTML中可访问。Angular会将该绑定转换成一系列的元素和绑定。

`let`关键字会创建一个名为`user`的模板输入变量（跟模板引用变量不同）。

##### NgFor索引

`ngFor`指令有个索引`index`，从0开始。我们可以用一个模板输入变量来捕获索引并在模板中使用。

{% raw %}
```html
<div *ngFor="let user of useres; let i=index">{{i + 1}} - {{user.fullName}}</div>
```
{% endraw %}

还有一些值如`last`、`even`、`odd`。


`NgFor`指令在列表很大时性能可能并不是很好。例如，我们从服务器更新useres数组，新的数组中可能大部分都是之前显示的，我们可以根据id来判断每个user是否改变。但是Angular只会看到新的数组引用，它会移除掉之前的所有元素，根据新数组重新构建并插入元素。

我们可以提供一个tracking函数来告诉Angular如何识别数组项是否一样：具有相同的`user.id`的user是相同的。

```js
trackByUseres(index: number, user: User) { return user.id; }
```

设置`NgForTrackBy`指令为定义的tracking函数，Angular提供了好几种绑定语法，例如：

{% raw %}
```html
<div *ngFor="let user of useres; trackBy:trackByUseres">({{user.id}}) {{user.fullName}}</div>
```
{% endraw %}

tracking函数不能消除所有的DOM更新。如果同一个对象的属性有变化，那Angular还是会更新它的。

#### `*`和`<template>`

我们注意到在使用`NgFor`、`NgIf`、`NgSwitch`时，用到了一个很奇怪的语法，在指令名前加前缀`*`。

`*`是个语法糖，让指令的阅读和编写更加简单。`NgFor`、`NgIf`和`NgSwitch`指令添加和移除的元素都包装在`<template>`中。但是我们并没有看到`<template>`，因为`*`前缀语法让我们可以忽略它，从而更多地关注我们要添加、移除或者重复的HTML元素。

##### 展开`*ngIf`

我们可以将`*`前缀语法展开成模板语法。例如使用`*ngIf`时：

```html
<user-detail *ngIf="currentUser" [user]="currentUser"></user-detail>
```

第一步，将`ngIf`（没有星号的）和它的内容转换成赋给`template`指令的表达式：

```html
<user-detail template="ngIf:currentUser" [user]="currentUser"></user-detail>
```

第二步（也是最后一步），将HTML 放到`<template>`标签内，并添加`ngIf`属性绑定：

```html
<template [ngIf]="currentUser">
  <user-detail [user]="currentUser"></user-detail>
</template>
```

##### 展开`*ngSwitch`

`*ngSwitch`的转换也是类似的：

```html
<span [ngSwitch]="toeChoice">

  <!-- with *NgSwitch -->
  <span *ngSwitchWhen="'Eenie'">Eenie</span>
  <span *ngSwitchWhen="'Meanie'">Meanie</span>
  <span *ngSwitchWhen="'Miney'">Miney</span>
  <span *ngSwitchWhen="'Moe'">Moe</span>
  <span *ngSwitchDefault>other</span>

  <!-- with <template> -->
  <template [ngSwitchWhen]="'Eenie'"><span>Eenie</span></template>
  <template [ngSwitchWhen]="'Meanie'"><span>Meanie</span></template>
  <template [ngSwitchWhen]="'Miney'"><span>Miney</span></template>
  <template [ngSwitchWhen]="'Moe'"><span>Moe</span></template>
  <template ngSwitchDefault><span>other</span></template>

</span>
```

##### 展开`*ngFor`

`*ngFor`转换也是类似的。转换之前：

```html
<user-detail *ngFor="let user of useres; trackBy:trackByUseres" [user]="user"></user-detail>
```

将`ngFor`转换成`template`指令：

```html
<user-detail template="ngFor let user of useres; trackBy:trackByUseres" [user]="user"></user-detail>
```

进一步转换成`<template>`元素：

```html
<template ngFor let-user [ngForOf]="useres" [ngForTrackBy]="trackByUseres">
  <user-detail [user]="user"></user-detail>
</template>
```

### Template reference variable（模板引用变量）

模板引用变量是模板内的一个DOM元素或指令的引用。它可以在DOM元素中或组件中使用。

#### 引用模板引用变量

我们可以在当前模板的任何位置使用模板引用变量。

```html
<<!-- phone refers to the input element; pass its `value` to an event handler -->
<input #phone placeholder="phone number">
<button (click)="callPhone(phone.value)">Call</button>

<!-- fax refers to the input element; pass its `value` to an event handler -->
<input ref-fax placeholder="fax number">
<button (click)="callFax(fax.value)">Fax</button>
```

`#`开头的`phone`意思就是我们定义了一个`phone`变量。

不喜欢使用`#`的同学可以使用`ref-`前缀。

这个变量是如何获取值的？

Angular将该变量所在的元素赋值给这个变量。我们在`input`元素上定义了变量`phone`，那么我们就可以在`button`元素这个使用这个变量来访问对应的`input`。

#### NgForm和模板引用变量

看这个HTML表单：

```html
<form (ngSubmit)="onSubmit(theForm)" #theForm="ngForm">
  <div class="form-group">
    <label for="name">Name</label>
    <input class="form-control" name="name" required [(ngModel)]="currentUser.firstName">
  </div>
  <button type="submit" [disabled]="!theForm.form.valid">Submit</button>
</form>
```

这个本地变量`theForm`的值到底是什么呢？

如果没有Angular，表单是一个HTMLFormElement。而在Angular中，它实际上是`ngForm`，Angular内建指令`NgForm`的引用，它包装了HTMLFormElement，并提供了一些额外的特性。

### Input and output property（输入属性和输出属性）

之前我们关注的都是绑定声明右侧的模板表达式和模板声明，也就是数据绑定源。接下来再看一下绑定目标，绑定声明左侧的指令属性。这些指令属性必须声明为输入还是输出。

绑定目标就是绑定符号（`[]`、`()`、`[()]`）内的属性或事件，绑定源就是引号（`""`）或这插值绑定（`{}`）内的内容。

只有显式标识为inputs或outputs的属性才能作为绑定目标。

例如`UserDetailComponent`中：

```html
<user-detail [user]="currentUser" (deleteRequest)="deleteUser($event)">
</user-detail>
```

`UserDetailComponent.user`和`UserDetailComponent.deleteRequest`都是绑定目标。`UserDetailComponent.user`在小括号内，它是个属性绑定的目标；`UserDetailComponent.deleteRequest`在中括号内，它是个事件绑定的目标。

#### 声明输入属性和输出属性

目标属性必须被显式地声明为输入或者输出：

```js
@Input()  user: User;
@Output() deleteRequest = new EventEmitter<User>();
```

或者在指令元数据的`inputs`和`outputs`数组中指定：

```js
@Component({
  inputs: ['user'],
  outputs: ['deleteRequest'],
})
```

输入属性通常接收数据值。输出属性暴露出事件生产者，例如`EventEmitter`对象。

`UserDetailComponent.user`是个输入属性，因为数据从模板绑定表达式流向属性。`UserDetailComponent.deleteRequest`是一个输出属性，因为事件从该属性发出，指向模板声明的处理者。

#### 输入/输出属性的别名

有时候我们希望为输入/输出属性提供一个对外的名称。

这在attribute指令中很常见。指令使用者希望绑定到指令的名称上。例如，我们想绑定一个selector为`myClick`的指令到`<div>`标签上，我们下网绑定的目标属性也叫`myClick`：

```html
<div (myClick)="clickMessage=$event">click with myClick</div>
```

然而指令的名称没有具体的指令类里属性的名字那样带有那么多含义。指令的名称很少会描述属性到底是干嘛的。

将组件的`clicks`事件属性起个别名`myClick`。

```js
@Output('myClick') clicks = new EventEmitter<string>(); //  @Output(alias) propertyName = ...
```

或者实在元数据中指定：

```js
@Directive({
  outputs:['clicks:myClick']  // propertyName:alias
})
```

`myClick`在指令内部实际上是`clicks`属性。

### Template expression operator（模板表达式运算符）

模板表达式扩展了一些JavaScript没有的操作符： pipe和Elvis。

#### pipe operator ( | )

有时候表达式的值在绑定前需要做一些转换。例如，将数字转换成货币格式、将字符串转成大写、对数组进行过滤或者排序。

管道（Pipe）是个简单的函数，接收出入值，返回转换后的值。例如：

{% raw %}
```html
<div>Title through uppercase pipe: {{title | uppercase}}</div>
```
{% endraw %}

管道运算符将左侧表达式的值传递给右侧的管道函数。

我们可以将多个管道函数链接起来：

{% raw %}
```html
<!-- Pipe chaining: convert title to uppercase, then to lowercase -->
<div>
  Title through a pipe chain:
  {{title | uppercase | lowercase}}
</div>
```
{% endraw %}

也可以对管道函数添加参数：

{% raw %}
```html
<!-- pipe with configuration argument => "February 25, 1970" -->
<div>Birthdate: {{currentUser?.birthdate | date:'longDate'}}</div>
```
{% endraw %}

`json`是个很有用的调试工具：

{% raw %}
```html
<div>{{currentUser | json}}</div>

<!-- Output:
  { "firstName": "Hercules", "lastName": "Son of Zeus",
    "birthdate": "1970-02-25T08:00:00.000Z",
    "url": "http://www.imdb.com/title/tt0065832/",
    "rate": 325, "id": 1 }
-->
```
{% endraw %}

#### Safe navigation operator ( ?. )

这个操作符是为了保证属性路径中有null或undefined值时不会出错：

{% raw %}
```html
The current user's name is {{currentUser?.firstName}}
```
{% endraw %}

这样，即使`currentUser`为null，Angular也不会报错了。

这在长属性调用中也可以，例如`a?.b?.c?.d`。

---

参考资料：

[Angular2 Documentation - Template Syntax](https://angular.io/docs/ts/latest/guide/template-syntax.html)
