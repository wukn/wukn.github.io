---
author: wukn
catalog: true
date: 2016-11-30T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Architecture Overview'
url: /2016/11/30/angular2-architecture-overview/
---

> Angular是一个使用HTML和JavaScript或能编译成JavaScript的TypeScript等编写客户端应用的框架。

<!--more-->

### 架构概览

编写Angular应用包括：使用Angular化的标记语言组装HTML模板（template），编写组件（component）类来管理这些模板，在服务（service）中添加业务逻辑，将组件和服务封装为模块（module）。

然后通过引导根模块来启动应用。Angular将负责在浏览器中展示应用的内容，相应用户的操作。

![](/img/post/angular2/architecture/architecture-overview.png)

架构图中描述了Angular应用的8个主要概念：

* 模块（Module）
* 组件（Component）
* 模板（Template）
* 元数据（Metadata）
* 数据绑定（Data Binding）
* 指令（Directive）
* 服务（Service）
* 依赖注入（Dependency Injection）

### 模块（Module）

Angular应用是模块化的。Angular的模块化系统称为[NgModule](https://angular.io/docs/ts/latest/guide/ngmodule.html)。

每个Angular应用至少有一个模块，那就是[根模块](https://angular.io/docs/ts/latest/guide/appmodule.html)，通常命名为`AppModule`。

一个模块就是一个带有`@NgModule`装饰器的类。

`@NgModule`是一个装饰器函数。它接受一个元数据对象作为参数，这个元数据对象的属性用来描述模块。最重要的属性有：

* `declarations`：属于该模块的视图类（view class）。Angular有三种视图类，组件component、指令directive和管道pipe。
* `exports`：`declarations`的子集，对其它模块的组件模板可见且可调用。
* `imports`：该模块引入的由其它模块export的类。
* `providers`：该模块提供的全局服务，对整个用用来说它们都是可访问的。
* `bootstrap`：应用的主视图，调用根组件。只有根模块才应该设置设个属性。

一个简单的根模块：

```js
// app/app.module.ts

import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
@NgModule({
  imports:      [ BrowserModule ],
  providers:    [ Logger ],
  declarations: [ AppComponent ],
  exports:      [ AppComponent ],
  bootstrap:    [ AppComponent ]
})
export class AppModule { }
```

PS：这里只是为了展示export的用法，其实不需要export AppComponent。一个根模块不需要export任何东西，因为没有其它模块会import根模块。

通过引导根模块来启动应用。例如在`main.ts`文件中引导`AppModule`：

```js
// app/main.ts

import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app.module';

platformBrowserDynamic().bootstrapModule(AppModule);
```

#### Angular模块 vs. JavaScript模块

Angular模块是带有`@NgModule`装饰器的类，它是Angular的一个特性。

JavaScript也有它自己的模块系统。在JavaScript中，每一个文件就是一个模块，所有在该文件内定义的对象都属于该模块。使用关键字export将模块中的对象声明为public类型。使用import来访问其它模块的public类型的对象。

```js
import { NgModule }     from '@angular/core';
import { AppComponent } from './app.component';
```

```js
export class AppModule { }
```

#### Angluar Libraries

![](/img/post/angular2/architecture/library-module.png)

Angular内置了很多JavaScript模块，它们的名字以`@angular`开头。可以使用npm包管理器来安装他们，然后用JavaScript的`import`语句导入他们。

例如，从`@angular/core`中引入Angularde`Component`装饰器：

```js
import { Component } from '@angular/core';
```

还有，使用JavaScript的import语句从Angular库中引入Angular模块：

```js
import { BrowserModule } from '@angular/platform-browser';
```

这是之前根模块的简单例子，该模块需要调用`BrowserModule`，在它的`@NgModule`元数据的`imports`属性中添加：

```js
imports:      [ BrowserModule ],
```

这种方式是将Angular模块和JavaScript模块系统结合起来使用的。

### 组件（Component）

组件控制屏幕以部分区域的显示，这个屏幕显示区域被称为视图。

我们在组件类中定义组件的业务逻辑，描述它如何与视图交互。例如，`HeroListComponent`有个属性`heroes`，这个属性返回的值是从一个服务获取的heroes数组。`HeroListComponent`还有一个方法`selectHero() `在用户点击列表中的一个`hero`之后设置`selectedHero`属性。

```js
// app/hero-list.component.ts (class)

export class HeroListComponent implements OnInit {
  heroes: Hero[];
  selectedHero: Hero;

  constructor(private service: HeroService) { }

  ngOnInit() {
    this.heroes = this.service.getHeroes();
  }

  selectHero(hero: Hero) { this.selectedHero = hero; }
}
```

Angular会根据用户的操作创建、更新以及销毁组件。可以使用生命周期钩子（lifecycle hook）看看组件的每个生命周期阶段。

### 模板（Template）

创建组件的同时会创建组件的视图，也就是模板。模板是一个HTML表单，用来告诉Angular如何渲染组件。

模板看起来根标准的HTML有些相似，但是也有些区别。

```js
// app/hero-list.component.html

<h2>Hero List</h2>
<p><i>Pick a hero from the list</i></p>
<ul>
  <li *ngFor="let hero of heroes" (click)="selectHero(hero)">
    {{hero.name}}
  </li>
</ul>
<hero-detail *ngIf="selectedHero" [hero]="selectedHero"></hero-detail>
```

代码中的`*ngFor`、`{{hero.name}}`、`(click)`、`[hero]`和`<hero-detail>`这些都是Angular的[模板语法](https://angular.io/docs/ts/latest/guide/template-syntax.html)。

最后一行的`<hero-detail>`标签是一个自定义元素，代表一个组件`HeroDetailComponent`。

组件与子组件之间的关系形成一个组件树。

![](/img/post/angular2/architecture/component-tree.png)

### 元数据（Metadata）

元数据用于告诉Angular如何来处理一个类。

看上面的`HeroListComponent`的代码，可以看到它只是一个类，并没有明显的Angular框架的痕迹。事实上，它的确就是一个类，只有把它告诉Angular之后它才是一个组件。

为了告诉Angular `HeroListComponent`是一个组件，我们需要给类附加一些元数据。

在TypeScript中，我们可以使用装饰器来附加元数据。例如：

```js
@Component({
  moduleId: module.id,
  selector:    'hero-list',
  templateUrl: 'hero-list.component.html',
  providers:  [ HeroService ]
})
export class HeroListComponent implements OnInit {
/* . . . */
}
```

装饰器`@Component`标识这个类是一个组件类。

装饰器`@Component`接受一个配置对象作为参数，Angular根据这个参数来创建和展示这个组件和它的视图。

常见的`@Component`配置项有：

* `moduleId`：模块相对地址的基础地址（`module.id`）。
* `selector`：CSS选择器，用来告诉Angular在这个标签内创建和插入组件实例。例如，如果应用的HTML里包含`<hero-list></hero-list>`，那么Angular会在这个标签内添加一个`HeroListComponent`实例。
* `templateUrl`：组件的HTML模板的模块相对地址。
* `providers`：他是一个数组，数组的每个元素都是模块所需的依赖注入提供者。它告诉Angular这个组件的构造函数需要`HeroService`。

模板、元数据和组件三者在一起称之为一个视图。

### 数据绑定（Data Binding）

没有框架时，我们需要手动将数据值放到HTML控件中，将用户的行为转换为操作和值更新。

Angular提供了数据绑定机制，用于协调视图和组件之间的对应关系。

![](/img/post/angular2/architecture/data-binding.png)

例如，`HeroListComponent`的模板包括三个元素：

```js
// app/hero-list.component.html (binding)

<li>{{hero.name}}</li>
<hero-detail [hero]="selectedHero"></hero-detail>
<li (click)="selectHero(hero)"></li>
```

`{{hero.name}}`插值表达式，将组件的`hero.name`属性值显示在`<li>`标签内。

`[hero]`属性绑定，将父组件`HeroListComponent`的`selectedHero`赋给子组件`HeroDetailComponent`的`hero`属性。

`(click)`事件绑定，当用户点击时调用组件的`selectHero`方法。

还有一种方式为双向数据绑定，使用`ngModel`指令，将属性绑定和事件绑定合并到一个记号中。例如：

```js
// app/hero-detail.component.html (ngModel)

<input [(ngModel)]="hero.name">
```

Angular会在每个JavaScript事件循环中处理所有的数据绑定，从组件树的根开始遍历所有组件。

![](/img/post/angular2/architecture/parent-child-binding.png)

数据绑定也是父子组件之间通信的一种重要方式。

### 指令（Directive）

Angular模块是动态的。当Angular渲染它们时是根据指令来转化成DOM结构的。

指令就是一个包含指令元数据的类。在TypeScript中使用`@Directive`装饰器来为类附加指令元数据。

组件是一个包含模板的指令。在Angular中组件是一个非常重要的概念，所以架构上把组件从指令中单独拎出来了。

另外两种指令是结构指令（structural directive）和属性指令（attribute directive）。它们通常作为元素标签的属性出现，有时是元素名称，但大部分情况下是绑定的目标对象。

__结构指令__ 通过添加、删除和替换DOM中的元素来修改页面内容。例如：

```js
// app/hero-list.component.html (structural)

<li *ngFor="let hero of heroes"></li>
<hero-detail *ngIf="selectedHero"></hero-detail>
```

× `*ngFor`告诉Angular为`heroes`列表中的每一个`hero`创建一个`<li>`标签。
* `*ngIf`当组件的selectedHero存在时显示一个`HeroDetail`组件

__属性指令__ 用于改变一个已经存在的元素的外观或行为。在模板中它们看起来就像是标准的HTML属性，包括名称也是。

用于双向数据绑定的`ngModel`就是一个属性指令。它可以通过设置显示值和相应事件来修改一个已经存在的元素的行为。

```js
// app/hero-detail.component.html (ngModel)

<input [(ngModel)]="hero.name">
```

其它常用的指令有`ngSwitch`、`ngStyle`和`ngClass`。

### 服务（Service）

服务是应用所需的一些值、函数或功能等。Angular本身并没有定义service。

```js
// app/logger.service.ts

export class Logger {
  log(msg: any)   { console.log(msg); }
  error(msg: any) { console.error(msg); }
  warn(msg: any)  { console.warn(msg); }
}
```

`HeroService`获取数据并返回一个Promise。该服务依赖于`LogService`和`BackendService`。

```js
// app/hero.service.ts

export class HeroService {
  private heroes: Hero[] = [];

  constructor(
    private backend: BackendService,
    private logger: Logger) { }

  getHeroes() {
    this.backend.getAll(Hero).then( (heroes: Hero[]) => {
      this.logger.log(`Fetched ${heroes.length} heroes.`);
      this.heroes.push(...heroes); // fill cache
    });
    return this.heroes;
  }
}

```

组件类经保持简单，它不应该处理从服务器获取数据、校验输入数据、记录日志等。这些任务应该委托给服务来处理。组件应该只是视图和应用逻辑的中间人。

一个好的组件应该只为数据绑定提供属性和方法，将所有其它不相关的事情委托给服务来处理。

### 依赖注入（Dependency Injection）

![](/img/post/angular2/architecture/dependency-injection.png)

依赖注入是为一个新实例化的类提供所有它所需要的依赖。大部分依赖都是服务。

Angular会根据构造函数的参数类型分辨所需要的服务组件。例如：

```js
// app/hero-list.component (constructor)

constructor(private _service: HeroService){ }
```

当Angular创建组件时，它向Injector询问组件所需的服务。Injector维护一个它创建的服务实例的容器。当请求的服务实例不在容器中时，injector就创建一个并添加到容器中，然后将服务返回给Angular。当所有所需的服务都返回了，Angular就可以用这些服务作为参数调用组件的构造函数了。

![](/img/post/angular2/architecture/injector-injects.png)

如果Injector不包含一个服务时，它怎么知道如何创建该服务呢？因此，需要在这之前为Injector注册一个该服务的提供者。提供者是一个可以创建服务的某个东西，通常是服务本身。

可以在模块或组件中注册提供者。通常在根模块中注册，这样该服务的实例可以在应用的任何地方使用。相当于一个单例服务了。

```js
// app/app.module.ts (module providers)

providers: [
  BackendService,
  HeroService,
  Logger
],
```

此外，也可以在组件层面注册：

```js
// app/hero-list.component.ts (component providers)

@Component({
  moduleId: module.id,
  selector:    'hero-list',
  templateUrl: 'hero-list.component.html',
  providers:  [ HeroService ]
})
```

在组件层面注册意味着每一个新的组件实例都有一个新的服务实例。

关于依赖注入，要注意的以下几点：

* Angular内置了依赖注入，并且广泛使用
* _injector_ 是最重要的概念。它维护了一个它所创建的服务实例的容器；它可以从 _provider_ 中创建新的服务实例
* _provider_ 是创建服务的清单
* 使用injector来注册 _provider_

---

参考资料：

[Angular2 Documentation - Architecture Overview](https://angular.io/docs/ts/latest/guide/architecture.html)
