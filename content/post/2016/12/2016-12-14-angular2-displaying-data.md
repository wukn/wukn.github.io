---
author: wukn
catalog: true
date: 2016-12-14T01:00:00Z
tags:
- Angular2
title: '[译][Angular2] Displaying Data'
url: /2016/12/14/angular2-displaying-data/
---

> 利用属性绑定在UI中显示数据。

<!--more-->

Angular通过将HTML模板中的标签与组件的属性绑定的方式来在UI显示数据。

假设我们想展示这样的一个UI：

![](/img/post/angular2/display-data/final.png)

## 插值法(interpolation)显示组件属性

显示组件属性最简单的方法就是使用插值法（interpolation）。插值方法就是将组件属性直接放在双层大括号里：

```js
import { Component } from '@angular/core';
@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <h2>My favorite hero is: {{myHero}}</h2>
    `
})
export class AppComponent {
  title = 'Tour of Heroes';
  myHero = 'Windstorm';
}
```

我们为组件添加了两个属性：`title`和`myHero`。同时在模板中添加了这两个属性的显示。

在应用运行过程中，Angular会自动读取组件的属性，接将属性值添加到浏览器中。这些属性变化时Angular也会更新UI。准确地说，属性的更新显示发生在与视图相关的某些异步事件之后，如击键、timer完成、异步XHR响应。

![](/img/post/angular2/display-data/title-and-hero.png)

PS：内联模板还是模板文件？

组件的模板可以直接内联在组件中，使用`template`属性。或者将模板定义在单独的文件中，然后将`@Component`的`templateUrl`属性指向该文件。

PS：构造函数初始化还是变量初始化？

尽管上面的例子中用的是变量初始化方式，我们也可以改成在构造函数中中初始化属性：

```js
export class AppCtorComponent {
  title: string;
  myHero: string;

  constructor() {
    this.title = 'Tour of Heroes';
    this.myHero = 'Windstorm';
  }
}
```

### NgFor显示组件的数组属性

我们为组件添加一个数组属性`heroes`，然后将`myHero`重定义为数组的第一个元素。在模板中使用`NgFor`指令显示数组中的每一项数据。

```js
import { Component } from '@angular/core';
@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <h2>My favorite hero is: {{myHero}}</h2>
    <p>Heroes:</p>
    <ul>
      <li *ngFor="let hero of heroes">
        {{ hero }}
      </li>
    </ul>
    `
})
export class AppComponent {
  title = 'Tour of Heroes';
  heroes = ['Windstorm', 'Bombasto', 'Magneta', 'Tornado'];
  myHero = this.heroes[0];
}
```

在`<li>`标签上使用了`*ngFor`，意思是将`<li>`元素（以及它的子元素）作为迭代模板。

指令中的`hero`是定义了一个模板输入变量（template input variable），详情请看[模板语法](https://angular.io/docs/ts/latest/guide/template-syntax.html)中的[microsyntax](https://angular.io/docs/ts/latest/guide/template-syntax.html#ngForMicrosyntax)。

Angular会为数组中的每一项复制一个`<li>`，将当前迭代项赋值给hero变量，然后使用该变量作为上下文向迭代模板中插值。

![](/img/post/angular2/display-data/hero-names-list.png)

### 创建数据类

这里是将数据定义在属性中。实际上通常要显示的数据是对象，也就是类的实例。接下来创建`hero.ts`文件，定义`Hero`类：

```js
// app/hero.ts

export class Hero {
  constructor(
    public id: number,
    public name: string) { }
}
```

看起来好像我们没有定义属性，其实这里是利用的TypeScript的缩写方式。看构造函数的第一个参数`public id: number,`，实际上它有三层含义：

* 定义构造函数的参数和类型
* 定义一个同名的共有属性
* 实例化这个类时，用相应的参数初始化属性



修改`app.component.ts`，使用`Hero`对象，并更新模板中的绑定为对象的属性：

```js
import { Component } from '@angular/core';
import { Hero } from './hero';

@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <h2>My favorite hero is: {{myHero.name}}</h2>
    <p>Heroes:</p>
    <ul>
      <li *ngFor="let hero of heroes">
        {{ hero.name }}
        </li>
    </ul>
  `
})
export class AppComponent {
  title = 'Tour of Heroes';
  heroes = [
    new Hero(1, 'Windstorm'),
    new Hero(13, 'Bombasto'),
    new Hero(15, 'Magneta'),
    new Hero(20, 'Tornado')
  ];
  myHero = this.heroes[0];
}
```

显示效果与上面是一样的。

## NgIf条件显示

有时候，应用在某些特定条件满足时才显示某些视图。`NgIf`指令会根据条件表达式的值决定是添加视图还是移除视图。
在模板最后添加：
```js
<p *ngIf="heroes.length > 3">There are many heroes!</p>
```

`*ngIf="heroes.length > 3"`的双引号里是模板表达式，看起来跟TypeScript很像。当该表达式成立时，Angular会将这个标签添加到DOM中，显示这条信息。表达式不成立时，Angular会忽略这个标签，就不显示这条信息了。

![](/img/post/angular2/display-data/final.png)

Angular不是简单地根据模板表达式的值来决定是显示还是隐藏`<p>`元素，而是向DOM添加或移除`<p>`元素。这个在大型项目中会提升应用的性能。

将heroes数组删掉两个之后：

![](/img/post/angular2/display-data/final2.png)

---

参考资料：

[Angular2 Documentation - Displaying Data](https://angular.io/docs/ts/latest/guide/displaying-data.html)
