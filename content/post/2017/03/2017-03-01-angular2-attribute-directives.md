---
author: wukn
catalog: true
date: 2017-03-01T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Attribute Directives'
url: /2017/03/01/angular2-attribute-directives/
---

> 属性指令用于改变DOM元素的外观或行为。

<!--more-->

### 指令综述

Angular中共有三种指令：

* 组件：带模板的指令。组件是三种指令中最常见的。
* 结构指令：通过添加或移除DOM元素来改变DOM布局。结构指令用于改变视图的结构。例如`NgFor`和`NgIf`。
* 属性指令：改变元素、组件或另一个指令的外观或元素。属性指令被当做元素的属性来使用。例如`NgStyle`。

### 构建一个简单的属性指令

最简单的属性指令就是一个带有`@Directive`注解的controller类，注解中指定了标识属性的选择器。controller类实现期望的指令行为。

接下来演示构建一个`myHighlight`属性指令，它的作用是当用户鼠标悬停在元素上时改变它的背景颜色。可以这样使用它：

```html
<p myHighlight>Highlight me!</p>
```

按照[Quick Start](https://wukn.github.io/2016/11/29/angluar2-quick-start/)中创建一个项目`attribute-directives`。

创建文件`src/app/highlight.directive.ts`：

```js
import { Directive, ElementRef, Input } from '@angular/core';
@Directive({ selector: '[myHighlight]' })
export class HighlightDirective {
    constructor(el: ElementRef) {
       el.nativeElement.style.backgroundColor = 'yellow';
    }
}
```

从Angular的`core`中引入：

* `Directive`：提供`@Directive`装饰器
* `ElementRef`：注入到指令的构造函数中以便于在代码中访问DOM元素
* `Input`：允许数据从绑定表达式流向指令

`@Directive`注解中指定了一个CSS选择器。一个属性的CSS选择器是一个放在中括号里的属性名。在这里属性的选择器是`[myHighlight]`。Angular会定位模板中所有拥有名为`myHighlight`的属性的元素。

Angular会为每个匹配的元素创建一个指令的controller类的实例，并将`ElementRef`注入到构造函数中。`ElementRef`是一个Service，通过它的`nativeElement`属性可以直接访问DOM元素。

### 在模板中为元素使用属性指令

为了使用`HighlightDirective`，我们创建一个模板：

创建模板文件`app.component.html`：

```html
<!-- src/app/app.component.html -->

<h1>My First Attribute Directive</h1>
<p myHighlight>Highlight me!</p>
```

修改`AppComponent`中的模板配置：

```js
// src/app/app.component.ts

import { Component } from '@angular/core';
@Component({
  moduleId: module.id,
  selector: 'my-app',
  templateUrl: './app.component.html'
})
export class AppComponent {
  color: string;
}
```

接下来，在NgModule的`declarations`元数据中引入`Highlight`，这样Angular遇到模板中的`myHighlight`才知道该把他解析成指令。

```js
// src/app/app.module.ts

import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { HighlightDirective } from './highlight.directive';
@NgModule({
  imports: [ BrowserModule ],
  declarations: [
    AppComponent,
    HighlightDirective
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

现在运行应用，可以看到：

![](/img/post/angular2/attribute-directives/highlight-1.png)

### 响应用户初始化事件

目前`myHighlight`只是简单的给元素添加背景色。我们可以为指令添加一些动态的效果。比如可以响应鼠标的进入和移出事件来为元素设置或取消背景色。

在`import`中添加`HostListener`，`Input`也即将用到，一并添加上：

```js
import { Directive, ElementRef, HostListener, Input } from '@angular/core';
```

添加两个响应鼠标进入和离开的事件处理器：

```js
@HostListener('mouseenter') onMouseEnter() {
  this.highlight('yellow');
}

@HostListener('mouseleave') onMouseLeave() {
  this.highlight(null);
}

private highlight(color: string) {
  this.el.nativeElement.style.backgroundColor = color;
}
```

`@HostListener`用于订阅使用了该指令的元素的事件。

当然我们也可以使用标准的JavaScript来访问DOM元素和附加事件监听。但这至少有三个问题：

* 必须手动的写对应事件的监听器
* 在指令销毁时需要在代码中移除事件监听，以免内存泄露
* 直接调用DOM的API并不是一个很好的选择

修改指令的构造函数，注入DOM元素`el`供事件处理函数调用：

```js
constructor(private el: ElementRef) { }
```

完整的代码如下：

```js
// src/app/highlight.directive.ts

import { Directive, ElementRef, HostListener, Input } from '@angular/core';
@Directive({
  selector: '[myHighlight]'
})
export class HighlightDirective {
  constructor(private el: ElementRef) { }
  @HostListener('mouseenter') onMouseEnter() {
    this.highlight('yellow');
  }
  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(null);
  }
  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}
```

现在运行应用，当鼠标不是停留在“Highlight me”这一行时是没有背景色的，鼠标移上去之后就有了（截图的时候鼠标不见了...）：

![](/img/post/angular2/attribute-directives/highlight-2-1.png)

![](/img/post/angular2/attribute-directives/highlight-2-2.png)

### 使用@Input数据绑定向指令传递数据

现在颜色是硬编码在指令中。我们将它改成可设置的。

为指令添加一个`highlightColor`属性：

```js
@Input() highlightColor: string;
```

注意，这里用到了`@Input`装饰器。它让指令的`highlightColor`属性可绑定。这里用的是input属性，是因为数据从绑定表达式流向指令。

向`AppComponent`的模板中添加指令的绑定：

```html
<p myHighlight highlightColor="yellow">Highlighted in yellow</p>
<p myHighlight [highlightColor]="'orange'">Highlighted in orange</p>
```

为`AppComponent`添加一个`color`属性：

```js
export class AppComponent {
  color = 'yellow';
}
```

使用属性绑定来控制颜色：

```html
<p myHighlight [highlightColor]="color">Highlighted with parent component's color</p>
```

也可以同时使用指令并绑定属性：

```html
<p [myHighlight]="color">Highlight me!</p>
```

`[myHighlight]`会同时应用指令并绑定属性。但是需要将指令的`highlightColor`属性名改成`myHighlight`。

```js
@Input() myHighlight: string;
```

但是`myHighlight`作为属性名太糟糕了。我们可以绑定一个`@Input`别名：

```js
@Input('myHighlight') highlightColor: string;
```

在指令内部可以用属性名`highlightColor`，在指令外部可以绑定`myHighlight`。

既然绑定了`highlightColor`属性，那么就需要修改`onMouseEnter()`的方法了：

```js
@HostListener('mouseenter') onMouseEnter() {
  this.highlight(this.highlightColor || 'red');
}
```

完整的代码如下：

```js
// src/app/highlight.directive.ts

import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[myHighlight]'
})
export class HighlightDirective {

  constructor(private el: ElementRef) { }

  @Input('myHighlight') highlightColor: string;

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.highlightColor || 'red');
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(null);
  }

  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}
```

我们将`AppComponent`改成选择颜色的方式。更新`app.component.html`：

```html
<h1>My First Attribute Directive</h1>

<h4>Pick a highlight color</h4>
<div>
  <input type="radio" name="colors" (click)="color='lightgreen'">Green
  <input type="radio" name="colors" (click)="color='yellow'">Yellow
  <input type="radio" name="colors" (click)="color='cyan'">Cyan
</div>
<p [myHighlight]="color">Highlight me!</p>
```

将`AppComponent.color`还原回去：

```js
export class AppComponent {
  color: string;
}
```

效果如下：

![](/img/post/angular2/attribute-directives/highlight-3.png)

### 绑定另一个属性

现在这个指令有了一个可设置的属性。我们还可以再添加属性用于绑定。

例如，现在的默认颜色是硬编码的，我们将它也改成可绑定的。

为指令添加input属性`defaultColor`：

```js
@Input() defaultColor: string;
```

修改指令的`onMouseEnter`，先尝试使用`highlightColor`，然后是`defaultColor`，当两个属性都未定义时才使用`red`：

```js
@HostListener('mouseenter') onMouseEnter() {
  this.highlight(this.highlightColor || this.defaultColor || 'red');
}
```

现在问题来了，怎样在模板中绑定第二个属性呢？

跟组件类似，我们可以在模板中添加任意多的指令属性绑定，如下：

```html
<p [myHighlight]="color" defaultColor="violet">
  Highlight me too!
</p>
```

Angular会知道`defaultColor`绑定是属于`HighlightDirective`的，因为指令中这个属性使用`@Input`标记为public类型了。

---

参考资料：

[Angular2 Documentation - Attribute Directives](https://angular.io/docs/ts/latest/guide/attribute-directives.html)
