---
layout:     post
title:      "[Angular2] Structural Directives"
subtitle:   ""
date:       2017-03-03 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> Angular提供了一个强大的模板引擎供去我们来操作DOM结构中的元素。

{% raw %}

### 什么是结构指令

结构指令使用来构造HTML布局的。它通过添加、移除或操作元素来形成或改变DOM结构。

跟其他指令一样，结构指令使用在宿主元素上，它会操作该元素以及它的子元素。

结构指令很好识别，通常指令属性名有个星号前缀，例如：

```html
<div *ngIf="hero" >{{hero.name}}</div>
```

没有中括号，也没有小括号，就是一个字符串`*ngIf`。

待会儿会说到，星号是一种简记法，相对通常的模板语法来说，这个字符串是一种微语法，这是Angular提供的语法糖。Angular会将这种写法还原成用`<template>`包装宿主元素及其子元素的形式。

最常见的内建结构指令是`NgIf`、`NgFor`和`NgSwitch`。例如：

```html
<div *ngIf="hero" >{{hero.name}}</div>

<ul>
  <li *ngFor="let hero of heroes">{{hero.name}}</li>
</ul>

<div [ngSwitch]="hero?.emotion">
  <happy-hero    *ngSwitchCase="'happy'"    [hero]="hero"></happy-hero>
  <sad-hero      *ngSwitchCase="'sad'"      [hero]="hero"></sad-hero>
  <confused-hero *ngSwitchCase="'confused'" [hero]="hero"></confused-hero>
  <unknown-hero  *ngSwitchDefault           [hero]="hero"></unknown-hero>
</div>
```

也许你注意到了，指令既有UpperCamelCase拼写方式，也有lowerCamelCase方式，就像`NgIf`和`ngIf`。它们的区别在于，`NgIf`指的是指令类，`ngIf`指的是指令的属性名。这是约定的命名规范。

一个宿主元素上可以使用多个属性指令，但是只能使用一个结构指令。

### NgIf

`NgIf`是最简单的也是最容易理解的结构指令了。它接收一个布尔值来控制显示或隐藏一个DOM块。

```html
<p *ngIf="true">
  Expression is true and ngIf is true.
  This paragraph is in the DOM.
</p>
<p *ngIf="false">
  Expression is false and ngIf is false.
  This paragraph is not in the DOM.
</p>
```

`ngIf`指令不是通过CSS来隐藏元素的，而是直接在DOM中添加或移除它。可以用浏览器的开发者工具看一下：

![](/img/post/angular2/structural-directives/element-not-in-dom.png)

为什么是移除而不是隐藏呢？

通过设置元素的`display`为`none`可以隐藏元素：

```html
<p [style.display]="'block'">
  Expression sets display to "block"" .
  This paragraph is visible.
</p>
<p [style.display]="'none'">
  Expression sets display to "none" .
  This paragraph is hidden but still in the DOM.
</p>
```

虽然隐藏了，但是这个元素还是在DOM中的：

![](/img/post/angular2/structural-directives/element-display-in-dom.png)

对于上面的例子中的段落来说，用移除还是隐藏关系并不大。但如果宿主元素是个比较占用资源的组件，那么即使它隐藏了，它的一些行为还是可用的，例如监听事件。Angular还要继续检查那些可能会影响它的数据绑定。这样即使是隐藏的，它和它的子组件仍然占用着资源。

使用隐藏的方式有一个好处，就是再次显示它的时候就很快，因为组件的状态都是保留的，而且不需要重新初始化。初始化是很消耗资源的。因此有的时候适合选择隐藏这种方式。

但是当我们没有足够的理由保留组件时，我们应该选择把它从DOM中移除，以释放相关的资源。

### 使用<ng-container>为兄弟元素分组

通常我们是在一个根元素上使用结构指令。`<li>`是个典型的`NgFor`的宿主元素：

```html
<li *ngFor="let hero of heroes">{{hero.name}}</li>
```

当没有宿主元素可用时，我们可以使用原生的HTML容器元素，例如`<div>`：

```html
<div *ngIf="hero" >{{hero.name}}</div>
```

通常，用`<span>`或`<div>`来为一些元素分组也是可以的。但是有时这并不是最好的办法，可能会影响CSS样式。例如：

```html
<p>
  I turned the corner
  <span *ngIf="hero">
    and saw {{hero.name}}. I waved
  </span>
  and continued on my way.
</p>
```

假设样式设置如下：

```
p span { color: red; font-size: 70%; }
```

那么渲染出来的效果就不对了：

![](/img/post/angular2/structural-directives/bad-paragraph.png)


另外，有的HTML元素的子元素是特定类型的，例如`<select>`标签需要`<option>`子标签，然而并不能用`<span>`或`<div>`来包装`<option>`。

如果我们像下面这样写，那么下拉框将是空的：

```html
<div>
  Pick your favorite hero
  (<label><input type="checkbox" checked (change)="showSad = !showSad">show sad</label>)
</div>
<select [(ngModel)]="hero">
  <span *ngFor="let h of heroes">
    <span *ngIf="showSad || h.emotion !== 'sad'">
      <option [ngValue]="h">{{h.name}} ({{h.emotion}})</option>
    </span>
  </span>
</select>
```

![](/img/post/angular2/structural-directives/bad-select.png)

这就是`<ng-container>`存在的理由。`<ng-container>`是一个分组元素，它不会干扰样式或布局。Angular不会把它放到DOM中。

例如：

```html
<p>
  I turned the corner
  <ng-container *ngIf="hero">
    and saw {{hero.name}}. I waved
  </ng-container>
  and continued on my way.
</p>
```

现在就能正常显示了：

![](/img/post/angular2/structural-directives/good-paragraph.png)

使用`<ng-container>`来排除`<select>`：

```html
<div>
  Pick your favorite hero
  (<label><input type="checkbox" checked (change)="showSad = !showSad">show sad</label>)
</div>
<select [(ngModel)]="hero">
  <ng-container *ngFor="let h of heroes">
    <ng-container *ngIf="showSad || h.emotion !== 'sad'">
      <option [ngValue]="h">{{h.name}} ({{h.emotion}})</option>
    </ng-container>
  </ng-container>
</select>
```

下拉框正常了：

![](/img/post/angular2/structural-directives/select-ngcontainer-anim.gif)

`<ng-container>`是一个Angluar解析器能识别的一个语法元素。它不是指令、组件、类或者接口。它很像JavaScript中If块的花括号：

```js
if (someCondition) {
  statement1;
  statement2;
  statement3;
}
```

没有花括号的话，JavaScript只会将第一个表达式作为if的内容，而实际上我们想吧三个表达式作为一个代码块。`<ng-container>`为Angular语法实现了类似的需求。

### 星号前缀

注意指令名称前面的星号前缀：

```html
<div *ngIf="hero" >{{hero.name}}</div>
```

这是一个语法糖。Angular会通过两步将他解开。第一步，将`*ngIf="..."`翻译成模板属性`template="ngIf ..."`，就像这样：

```html
<div template="ngIf hero">{{hero.name}}</div>
```

然后将模板属性翻译成模板元素，将宿主元素包装在里面：

```html
<template [ngIf]="hero">
  <div>{{hero.name}}</div>
</template>
```

* `*ngIf`指令移到`<template>`标签上变成了属性绑定`[ngIf]`。
* `<div>`余下的部分，包括它的类属性，全部移到`<template>`标签内。

转换的过程不会被渲染，DOM中渲染的只是最终的结果：

![](/img/post/angular2/structural-directives/hero-div-in-dom.png)

`NgFor`和`NgSwitch`跟这个是类似的。

### NgFor

`NgFor`的三种写法：

```html
<div *ngFor="let hero of heroes; let i=index; let odd=odd; trackBy: trackById" [class.odd]="odd">
  ({{i}}) {{hero.name}}
</div>

<div template="ngFor let hero of heroes; let i=index; let odd=odd; trackBy: trackById" [class.odd]="odd">
  ({{i}}) {{hero.name}}
</div>

<template ngFor let-hero [ngForOf]="heroes" let-i="index" let-odd="odd" [ngForTrackBy]="trackById">
  <div [class.odd]="odd">({{i}}) {{hero.name}}</div>
</template>
```

`NgFor`比`NgIf`稍微复杂点。`NgFor`的必需属性和可选属性更多。它至少需要一个循环变量（`let hero`）和一个列表（`heroes`）。

把宿主元素移到`<template>`中时，`ngFor`字符串之外的东西仍然留在宿主元素上。上面的`[ngClass]="odd"`就仍然在`<div
>`上。

#### 微语法

`*ngFor="let hero of heroes; let i=index; let odd=odd; trackBy: trackById"`，右侧的字符串就是微语法。它让我们可以通过一个简洁的字符串来配置指令。微语法解析器会把这个字符串转换为`<template>`的属性。

* `let`关键字声明了一个模板输入变量，我们可以在模板中引用它。例子中的输入变量是`hero`、`i`和`odd`。解析器将`let hero`、`let i`和`let odd`转换成了变量`let-hero`、`let-i`和`let-odd`。
* 对于`of`和`trackby`，先将它们的首字母大写，然后加上指令的属性名作为前缀，形成`ngForOf`和`ngForTrackBy`。这两个就是`NgFor`输入属性的名称。
* `NgFor`对列表进行循环时，它会设置和重置它的上下文对象的属性。这些属性包括`index`、`odd`和一个特殊的属性`$implicit`。
* 变量`let-i`和`let-odd`被定义为`let i=index`和`let odd=odd`。Angular将他们赋值给当前上下文的`index`和`odd`属性。
* `let-hero`对应的上下文属性没有被指定。它是隐式的。Angular在这次迭代中将`let-hero`赋值给上下文的`$implicit`属性，来初始化hero。

#### 模板输入变量

模板输入变量（template input variable）就是在模板的单实例内可以引用的变量。例如上面的`hero`、`li`和`odd`，它们前面都加了个关键字`let`。

不管从语义上还是语法上来说，模板输入变量和模板引用变量（template reference variable）都是不一样的。

模板输入变量是用关键字`let`定义的（`let hero`），它的生命周期是被重复的模板的单个实例中。你可以在其它指令的定义中使用同样的变量名。

模板引用变量的定义是在变量名前加`#`（`#var`）。引用变量只想它被附加的元素、组件或指令。它可以在整个模板的任意地方被引用。

模板输入变量和引用变量各有它们自己的作用域。`let hero`和`#hero`中的这两个`hero`是不一样的。

#### 不能在一个元素上使用多个结构指令

再次说明，每个元素最多只能应用一个结构指令。假如在一个元素上同时使用`NgIf`和`NgFor`，它们之间可能的冲突或让Angular不知所措。所以不允许在一个元素上使用多个结构指令。我们可以将`*ngIf`应用在包装了`*ngFor`元素的容器元素上。

### NgSwitch

`NgSwitch`实际上是一组相关的指令：`NgSwitch`、`NgSwitchCase`和`NgSwitchDefault`。

举个栗子：

```html
<div [ngSwitch]="hero?.emotion">
  <happy-hero    *ngSwitchCase="'happy'"    [hero]="hero"></happy-hero>
  <sad-hero      *ngSwitchCase="'sad'"      [hero]="hero"></sad-hero>
  <confused-hero *ngSwitchCase="'confused'" [hero]="hero"></confused-hero>
  <unknown-hero  *ngSwitchDefault           [hero]="hero"></unknown-hero>
</div>
```

`NgSwitch`本身不是结构指令，它是一个属性指令，用来控制那两个switch指令的行为的。这就是为什么写的是`[ngSwitch]`，而不是`*ngSwitch`。

`NgSwitchCase`和`NgSwitchDefault`是结构指令。它们前面是有星号前缀的。`NgSwitchCase`会在它的值跟switch值匹配时显示宿主元素。当没有`NgSwitchCase`匹配时，`NgSwitchDefault`显示它的宿主元素。

将语法糖解开成模板属性形式：

```html
<div [ngSwitch]="hero?.emotion">
  <happy-hero    template="ngSwitchCase 'happy'"    [hero]="hero"></happy-hero>
  <sad-hero      template="ngSwitchCase 'sad'"      [hero]="hero"></sad-hero>
  <confused-hero template="ngSwitchCase 'confused'" [hero]="hero"></confused-hero>
  <unknown-hero  template="ngSwitchDefault"         [hero]="hero"></unknown-hero>
</div>
```

进一步解成模板元素形式：

```html
<div [ngSwitch]="hero?.emotion">
  <template [ngSwitchCase]="'happy'">
    <happy-hero [hero]="hero"></happy-hero>
  </template>
  <template [ngSwitchCase]="'sad'">
    <sad-hero [hero]="hero"></sad-hero>
  </template>
  <template [ngSwitchCase]="'confused'">
    <confused-hero [hero]="hero"></confused-hero>
  </template >
  <template ngSwitchDefault>
    <unknown-hero [hero]="hero"></unknown-hero>
  </template>
</div>
```

### 使用星号前缀方式

尽量始终选择星号前缀这种方式。星号前缀方式相对更简洁。如果没有元素作为指令的宿主，那么就用`<ng-container> `。

尽管很少用到结构指令的模板属性或模板元素形式，了解下有助于我们自己编写结构指令。

### `<template>`元素

HTML5中的`<template>`是用来渲染HTML的。它不会被直接显示。事实上，Angular在渲染 它之前会把它和它的内容放到注释中。

如果我们不使用结构指令而把一些内容放到`<template>`中时，这些元素会消失。例如：

```html
<p>Hip!</p>
<template>
  <p>Hip!</p>
</template>
<p>Hooray!</p>
```

![](/img/post/angular2/structural-directives/template-rendering.png)

### 写一个结构指令

让我们自己动手写一个结构指令。写一个跟`NgIf`相反的`UnlessDirective`吧。

```html
<p *myUnless="condition">Show this sentence unless the condition is true.</p>
```

创建指令和创建组件是类似的：

* 引入`Directive`装饰器
* 引入`Input`、`TemplateRef`和`ViewContainerRef`，这在任何结构指令中都是必需的
* 为指令类应用装饰器
* 设置CSS属性选择器

开始写吧：

```js
// unless.directive.ts (skeleton)

import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({ selector: '[myUnless]'})
export class UnlessDirective {
}
```

指令属性名称应该使用lowerCamelCase拼写方式，且带个前缀。不要使用`ng`作为前缀，它是Angular用的。

指令的类名以`Directive`结尾。Angular自己的指令并不是这样的。

这个简单的指令会从Angular生成的`<template>`生成一个嵌入的视图，并将它添加到邻近指令宿主元素的视图容器中。

你可以通过`TemplateRef`来获得`<template>`，通过`ViewContainerRef`来访问视图容器。

在指令的构造函数中注入它们：

```js
constructor(
  private templateRef: TemplateRef<any>,
  private viewContainer: ViewContainerRef) { }
```

这个指令的用法是希望绑定一个布尔结果的表达式给`[myUnless]`，这意味着指令需要一个`myUnless`输入属性：

```js
@Input() set myUnless(condition: boolean) {
  if (!condition && !this.hasView) {
    this.viewContainer.createEmbeddedView(this.templateRef);
    this.hasView = true;
  } else if (condition && this.hasView) {
    this.viewContainer.clear();
    this.hasView = false;
  }
}
```

完整的代码如下：

```js
// unless.directive.ts

import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';
/**
 * Add the template content to the DOM unless the condition is true.
 */
@Directive({ selector: '[myUnless]'})
export class UnlessDirective {
  private hasView = false;
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef) { }
  @Input() set myUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}
```

使用的时候把这个指令添加到应用的模块的`declarations`数组中，然后在模板中使用它。

```html
<p *myUnless="condition" class="unless a">
  (A) This paragraph is displayed because the condition is false.
</p>

<p *myUnless="!condition" class="unless b">
  (B) Although the condition is true,
  this paragraph is displayed because myUnless is set to false.
</p>
```

效果如下：

![](/img/post/angular2/structural-directives/unless-anim.gif)

{% endraw %}

---

参考资料：

[Angular2 Documentation - Structural Directives](https://angular.io/docs/ts/latest/guide/structural-directives.html)
