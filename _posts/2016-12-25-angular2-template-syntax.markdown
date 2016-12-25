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
<p><img src="{{heroImageUrl}}"> is the <i>interpolated</i> image.</p>
<p><img [src]="heroImageUrl"> is the <i>property bound</i> image.</p>

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

```html
<p><span>"{{evilTitle}}" is the <i>interpolated</i> evil title.</span></p>
<p>"<span [innerHTML]="evilTitle"></span>" is the <i>property bound</i> evil title.</p>
```

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


---

参考资料：

[Angular2 Documentation - Template Syntax](https://angular.io/docs/ts/latest/guide/template-syntax.html)
