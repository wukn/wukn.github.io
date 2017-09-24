---
author: wukn
catalog: true
date: 2017-01-11T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] NgModule'
url: /2017/01/11/angular2-ngmodules/
---

> 使用@NgModule定义应用的模块。

<!--more-->

Angular模块有助于将应用划分为内聚的功能模块。

Angular的模块是一个使用了`@NgModule`装饰器的类。`@NgModule`接收一个元数据对象作为参数，元数据告诉Angular怎样编译和运行模块。它会指出哪些是模块自己的组件、指令和管道，把其中的一些标记为公共的以便外部组件来调用它们。它还会将服务提供者添加到应用的依赖注入器中。等等。

前面我们已经见过了根模块。

### Angular的模块化

模块是组织应用和让应用具有从外部进行扩展能力的一种伟大的方式。

Angular的库都是模块（`FormsModule`、`HttpModule`、`RouterModule`），很多第三方库也是以模块的方式提供（[Material Design](https://material.angular.io/)、[Ionic](http://ionicframework.com/)、[AngularFire2](https://github.com/angular/angularfire2)）。

Angular模块将组件、指令和管道揉合在一起形成高内聚的功能块，每个模块关注一个特性域、应用的业务领域、流程和常用的工具集合。

模块也可以用来向应用添加服务。例如内部开发的日志服务。它们可以来自外部的源例如Angular路由器和Http客户端。

模块的加载可以在应用启动时加载（贪婪式加载），也可以通过路由器异步加载（懒惰式加载）。

Angular模块是一个装饰有`@NgModule`元数据的类。元数据：

* 声明哪些组件、指令和管道属于该模块
* 把一些类作为公共的暴露出去，供其它组件模板使用
* 引入该模块所需要的其它模块的组件、指令和管道
* 提供应用层级的服务，让整个应用的组件都可以使用

每个Angular应用至少包含一个模块，那就是根模块。我们通过加载这个模块来启动应用。

### AppModule——应用的根模块

应用的根模块类名通常为`AppModule`，文件名为`app.module.ts`：

```js
// app/app.module.ts (minimal)

import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent }  from './app.component';

@NgModule({
  imports:      [ BrowserModule ],
  declarations: [ AppComponent ],
  bootstrap:    [ AppComponent ]
})
export class AppModule { }
```

`@NgModule`的元数据中引入了辅助模块`BrowserModule`，这是每个浏览器应用必须要引入的。`BrowserModule`会注册一些重要的服务的provider，例如指令`NgIf`和`NgFor`，这样在这个模块的组件模板中就能直接使用这些指令了。

`declarations`声明了应用的根组件。一个最简单的`AppComponent`示例如下：

```js
// app/app.component.ts (minimal)

import { Component } from '@angular/core';

@Component({
  selector: 'my-app',
  template: '<h1>{{title}}</h1>',
})
export class AppComponent {
  title = 'Minimal NgModule';
}
```

最后，`@NgModule.bootstrap`属性声明`AppComponent`为启动组件。

### 启动根模块

应用的启动是通过在`main.ts`文件中加载`AppModule`的。

Angular针对不同的平台提供了各种各样的启动方式。看下最常用的针对浏览器平台的两种方式。

#### 结合JIT（Just-in-time）编译器动态加载

dynamic方式，编译器在浏览器中编译程序并启动应用。

```js
// app/main.ts (dynamic)

// The browser platform with a compiler
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

// The app module
import { AppModule } from './app.module';

// Compile and launch the module
platformBrowserDynamic().bootstrapModule(AppModule);
```

### 结合AOT（Ahead-Of-time）编译器静态加载

static方式可以产生更小的应用，启动起来会更快，尤其在移动终端和高延迟网络环境下。

在这种方式下，Angular编译器会提前运行部分构建流程，在各自的文件中产生一些工厂类集合。其中有`AppModuleNgFactory`。

```js
// app/main.ts (static)

// The browser platform without a compiler
import { platformBrowser } from '@angular/platform-browser';

// The app module factory produced by the static offline compiler
import { AppModuleNgFactory } from './app.module.ngfactory';

// Launch with the app module factory.
platformBrowser().bootstrapModuleFactory(AppModuleNgFactory);
```

因为整个应用是预编译过的，就不需要将编译器发布到浏览器中，也不需要在浏览器中编译。这样下载到浏览器中的代码就比动态方式的更小，而且是可以直接运行的。这会有很明显的性能提升。

JIT和AOT这两种编译器对同一个`AppModule`文件都会生成`AppModuleNgFactory`类。JIT编译器是需要的时候才生成、在内存中、在浏览器中。AOT编译器会将编译结果放到我们引入的这个文件中。

通常来说，`AppModule`不应该也不需要知道它自己是如何被启动的。

随着应用的发展`AppModule`会向前演进。但`main.ts`中的启动代码是不变的。这是我们最后一次看到`main.ts`。Farewell。

### 声明指令和组件

#### 定义指令

定义一个`HighlightDirective`指令，作用是为元素设置背景颜色。它是一个属性指令。

```js
// app/highlight.directive.ts

import { Directive, ElementRef } from '@angular/core';

@Directive({ selector: '[highlight]' })
/** Highlight the attached element in gold */
export class HighlightDirective {
  constructor(el: ElementRef) {
    el.nativeElement.style.backgroundColor = 'gold';
    console.log(
      `* AppRoot highlight called for ${el.nativeElement.tagName}`);
  }
}
```

更新`AppComponent`中的模板，使用这个指令：

```js
template: '<h1 highlight>{{title}}</h1>'
```

如果现在启动应用，会发现指令没有起作用。因为必须要在`AppModule`中声明这个指令：

```js
declarations: [
  AppComponent,
  HighlightDirective,
],
```

#### 定义组件

将模板中的Title重构成一个单独的组件`TitleComponent`。组件的模板中绑定组件的`title`和`subtitle`属性：

```html
<!-- app/title.component.html -->

<h1 highlight>{{title}} {{subtitle}}</h1>
```

```js
// app/title.component.ts

import { Component, Input } from '@angular/core';

@Component({
  moduleId: module.id,
  selector: 'app-title',
  templateUrl: 'title.component.html',
})
export class TitleComponent {
  @Input() subtitle = '';
  title = 'Angular Modules';
}
```

将`AppComponent`中的模板改成使用该组件：

```js
// app/app.component.ts (v1)

import { Component } from '@angular/core';

@Component({
  selector: 'my-app',
  template: '<app-title [subtitle]="subtitle"></app-title>'
})
export class AppComponent {
  subtitle = '(v1)';
}
```

Angular现在还不识别`<app-title>`标签，因为还没有在`AppModule`中声明它：

```js
declarations: [
  AppComponent,
  HighlightDirective,
  TitleComponent,
],
```

### Service Provider

模块是为模块内的所有组件提供service的一种好方法。

前面的[依赖注入](https://angular.io/docs/ts/latest/guide/dependency-injection.html)篇章中描述了Angular的层级式依赖注入系统，以及怎样在组件树的不同层级配置provider。

模块可以将provider添加到应用的根依赖注入器中，让服务在整个应用中都可以使用。

举个例子。很多应用需要获取当前登录用户的信息，并且通过user service让该信息可被访问。我们写一个假的`UserService`：

```js
// app/user.service.ts

import { Injectable } from '@angular/core';

@Injectable()
/** Dummy version of an authenticated user service */
export class UserService {
  userName = 'Sherlock Holmes';
}
```

假设我们要在应用的标题下面显示一条用户登录信息。更新`TitleComponent`的模板：

```js
<!-- app/title.component.html -->
<h1 highlight>{{title}} {{subtitle}}</h1>
<p *ngIf="user">
  <i>Welcome, {{user}}</i>
<p>
```

更新`TitleComponent`类，在构造函数中注入`UserService`并设置组件的`user`属性：

```js
// app/title.component.ts

import { Component, Input } from '@angular/core';
import { UserService } from './user.service';

@Component({
  moduleId: module.id,
  selector: 'app-title',
  templateUrl: 'title.component.html',
})
export class TitleComponent {
  @Input() subtitle = '';
  title = 'Angular Modules';
  user = '';

  constructor(userService: UserService) {
    this.user = userService.userName;
  }
}
```

我们定义了`UserService`。现在让它可以为所有组件所用，将它加到`AppModule`的`providers`属性中就行啦：

```js
providers: [ UserService ],
```

### 引入模块

注意上面的`TitleComponent`模板中用到了`*ngIf`指令。尽管`AppModule`没有声明`NgIf`，但是Angular却认识它，这是为什么呢？

因为在`AppModule`中引入了`BrowserModule`：

```js
imports: [ BrowserModule ],
```

引入了`BrowserModule`模块之后，在`AppModule`的组件模板中就可以使用被引入的模块的公共组件、指令和管道。

注：实际上，`NgIf`是在`@angular/common`的`CommonModule`中声明的。`NgFor`也在这个模块中。`BrowserModule`引入了`CommonModule`又重新导出了它。这样引入`BrowserModule`之后就自动得到了`CommonModule`中的指令。

很多其它常见的组件不在`CommonModule`中。例如`NgModel`在`FormsModule`中，`RouterLink`在`RouterModule`中。当使用这些指令时需要引入相应的模块。

作为演示，我们为应用添加一个组件`ContactComponent`，它是一个从`FormsModule`引入表单功能的表单组件。它的功能是一个通讯录编辑器，用模板驱动式表单来实现的。

#### Angular表单风格

Angular的表单组件有两种风格：模板驱动式表单（template-driven form ）和响应式表单（reactive form）。

模板驱动式表单从`@angular/forms`中引入`FormsModule`。响应式表单引入的是`ReactiveFormsModule`。

`ContactComponent`的selector为`<app-contact>`。在`AppComponent`的模板中添加：

```js
// app/app.component.ts (template)

template: `
  <app-title [subtitle]="subtitle"></app-title>
  <app-contact></app-contact>
`
```

`ContactComponent`还比较复杂，它有自己的`ContactService`，一个名为`Awesome`的自定义管道，还有一个指令`HighlightDirective`。

```html
<!-- app/contact/contact.component.html -->

<h2>Contact of {{userName}}</h2>
<div *ngIf="msg" class="msg">{{msg}}</div>
<form *ngIf="contacts" (ngSubmit)="onSubmit()" #contactForm="ngForm">
  <h3 highlight>{{ contact.name | awesome }}</h3>
  <div class="form-group">
    <label for="name">Name</label>
    <input type="text" class="form-control" required
      [(ngModel)]="contact.name"
        name="name"  #name="ngModel" >
    <div [hidden]="name.valid" class="alert alert-danger">
      Name is required
    </div>
  </div>
  <br>
  <button type="submit" class="btn btn-default" [disabled]="!contactForm.form.valid">Save</button>
  <button type="button" class="btn" (click)="next()" [disabled]="!contactForm.form.valid">Next Contact</button>
  <button type="button" class="btn" (click)="newContact()">New Contact</button>
</form>
```

```js
// app/contact/contact.component.ts

import { Component, OnInit }      from '@angular/core';
import { Contact, ContactService } from './contact.service';
import { UserService }    from '../user.service';
@Component({
  moduleId: module.id,
  selector: 'app-contact',
  templateUrl: 'contact.component.html',
  styleUrls: [ 'contact.component.css' ]
})
export class ContactComponent implements OnInit {
  contact:  Contact;
  contacts: Contact[];
  msg = 'Loading contacts ...';
  userName = '';
  constructor(private contactService: ContactService, userService: UserService) {
    this.userName = userService.userName;
  }
  ngOnInit() {
    this.contactService.getContacts().then(contacts => {
      this.msg = '';
      this.contacts = contacts;
      this.contact = contacts[0];
    });
  }
  next() {
    let ix = 1 + this.contacts.indexOf(this.contact);
    if (ix >= this.contacts.length) { ix = 0; }
    this.contact = this.contacts[ix];
  }
  onSubmit() {
    // POST-DEMO TODO: do something like save it
    this.displayMessage('Saved ' + this.contact.name);
  }
  newContact() {
    this.displayMessage('New contact');
    this.contact = {id: 42, name: ''};
    this.contacts.push(this.contact);
  }
  /** Display a message briefly, then remove it. */
  displayMessage(msg: string) {
    this.msg = msg;
    setTimeout(() => this.msg = '', 1500);
  }
}
```

```css
/* app/contact/contact.component.css */

.ng-valid[required] {
  border-left: 5px solid #42A948; /* green */
}
.ng-invalid {
  border-left: 5px solid #a94442; /* red */
}
.alert {
  padding: 15px;
  margin: 8px 0;
  border: 1px solid transparent;
  border-radius: 4px;
}
.alert-danger {
  color: #a94442;
  background-color: #f2dede;
  border-color: #ebccd1;
}
.msg {
  color: blue;
  background-color: whitesmoke;
  border: 1px solid transparent;
  border-radius: 4px;
  margin-bottom: 20px;
}
```

```js
// app/contact/contact.service.ts

import { Injectable } from '@angular/core';
export class Contact {
  constructor(public id: number, public name: string) { }
}
const CONTACTS: Contact[] = [
  new Contact(21, 'Sam Spade'),
  new Contact(22, 'Nick Danger'),
  new Contact(23, 'Nancy Drew')
];
const FETCH_LATENCY = 500;
@Injectable()
export class ContactService {
  getContacts() {
    return new Promise<Contact[]>(resolve => {
      setTimeout(() => { resolve(CONTACTS); }, FETCH_LATENCY);
    });
  }
  getContact(id: number | string) {
    return this.getContacts()
      .then(heroes => heroes.find(hero => hero.id === +id));
  }
}
```

```js
// app/contact/awesome.pipe.ts

import { Pipe, PipeTransform } from '@angular/core';
@Pipe({ name: 'awesome' })
/** Precede the input string with the word "Awesome " */
export class AwesomePipe implements PipeTransform {
  transform(phrase: string) {
    return phrase ? 'Awesome ' + phrase : '';
  }
}
```

```js
// app/contact/highlight.directive.ts

import { Directive, ElementRef } from '@angular/core';
@Directive({ selector: '[highlight], input' })
/** Highlight the attached element or an InputElement in blue */
export class HighlightDirective {
  constructor(el: ElementRef) {
    el.nativeElement.style.backgroundColor = 'powderblue';
    console.log(
      `* Contact highlight called for ${el.nativeElement.tagName}`);
  }
}
```

注意模板中的双向数据绑定`ngModel`，它是`NgModel`指令的选择器。

尽管`NgModel`是Angular指令，但是编译器还不认识它，因为（1）`AppModule`没有声明它，（2）它不是通过`BrowserModule`引入的。将`FormsModule`添加到`AppModule`元数据的`imports`列表中：

```js
imports: [ BrowserModule, FormsModule ],
```

>>> Never re-declare classes that belong to another module.

在声明自定义的组件、指令和管道之前，应用是不能成功编译的。更新`AppModule`的`declarations`：

```js
declarations: [
  AppComponent,
  HighlightDirective,
  TitleComponent,

  AwesomePipe,
  ContactComponent,
  ContactHighlightDirective
],
```

有两个指令都叫`HighlightDirective`。这里采取了一种变通的方式，为第二个起了个别名：

```js
import {
  HighlightDirective as ContactHighlightDirective
} from './contact/highlight.directive';
```

`ContactComponent`从构造函数中注入的`ContactService`获取数据。我们必须在某个地方提供这个这个服务才行。`ContactComponent`可以提供它，但这样就只能在这个组件内使用了。我们希望在其它跟这个组件相关的组件也能使用这个服务。因此将它添加到`AppModule`元数据的`providers`列表中。

```js
providers: [ ContactService, UserService ],
```

#### Application-scoped Providers

`ContactService`Provider现在是应用程序范围的。因为Angular会将模块的`providers`注册到应用的根注入器中。从技术上来说`ContactService`属于Contact业务领域。其它领域的类不需要引入也不应该注入`ContactService`。

也许我们期望Angular提供模块范围的机制来强制这种设计。但是事实不是这样的。Angular模块的实例不像组件实例，它没有自己的注入器，因此它不能有自己的provider作用域。

这个疏忽是有意设计成这样的。Angular模块最初的设计摸底是用来扩展应用的，用模块的功能来丰富应用。

在实际情况中服务的作用域基本不会出现问题。非Contact组件是不能注入`ContactService`的。要注入`ContactService`必须引入它的类型。只有Contact组件才应该引入`ContactService`类型。

### 解决指令冲突

上面出现了指令冲突问题。有两个`HighlightDirective`：

```js
// app/highlight.directive.ts

import { Directive, ElementRef } from '@angular/core';
@Directive({ selector: '[highlight]' })
/** Highlight the attached element in gold */
export class HighlightDirective {
  constructor(el: ElementRef) {
    el.nativeElement.style.backgroundColor = 'gold';
    console.log(
      `* AppRoot highlight called for ${el.nativeElement.tagName}`);
  }
}
```

```js
// app/contact/highlight.directive.ts

import { Directive, ElementRef } from '@angular/core';
@Directive({ selector: '[highlight], input' })
/** Highlight the attached element or an InputElement in blue */
export class HighlightDirective {
  constructor(el: ElementRef) {
    el.nativeElement.style.backgroundColor = 'powderblue';
    console.log(
      `* Contact highlight called for ${el.nativeElement.tagName}`);
  }
}
```

Angular会只使用其中一个吗？不。实际上两个都会生效。两个指令会竞争为元素设置背景颜色，后声明的那个指令最终赢得竞争，因为它修改DOM时会覆盖前面的。

其实真正的问题是引入了两个不同的类去做同一件事。如果是多次引入同一个类，Angular会自动去重的。但现在的情况是在不同的文件中定义了两个不同的类，只是碰巧它俩的名字一样。从Angular的角度来看它俩是不同的。Angular会同时保留它俩，并依次作用到元素上。

这样至少程序还能运行。如果定义两个组件类用相同的selector名字，那编译器就会报错了，因为不能在同一个DOM中添加两个组件。

消除组件和指令冲突可以通过创建特性模块（feature module），将一个模块的声明从另一个模块的声明隔离开。

### 特性模块

这个应用的规模现在还不是很大，但是已经出现一些问题了：

* `AppModule`会随着新的类的增加而增加
* 出现指令冲突问题
* 应用在Contact功能和其它应用特性之间缺乏清晰的界限。

接下来通过特性模块来消除这些问题。

#### 特性模块

特性模块跟根模块类似，是一个使用`@NgModule`装饰器的类。它的元数据的属性跟根模块的也是一样的。

根模块和特性模块共享同一个执行上下文。它们共享同一个依赖注入器，也就是说在一个模块中提供服务那么在所有的模块中都能访问。

根模块和特性模块之间的区别在于：

* 根模块是用来启动应用的，特性模块是用来扩展应用的
* 特性模块可以对其他模块暴露或隐藏它的实现

特性模块提供一组内聚的功能，聚焦于一个应用业务领域、工作流、基础设施（表单、http、路由）、一个相关公共功能的集合。

尽管我们可以在根模块中做所有的事情，但是特性模块有助于将应用根据功能分块。

一个特性模块通过它提供的服务以及它暴露出的公共组件、指令和管道来跟根模块和其他模块进行交互。

#### 将Contact重构成特性模块

创建`ContactModule`:

```js
// app/contact/contact.module.ts

import { NgModule }           from '@angular/core';
import { CommonModule }       from '@angular/common';
import { FormsModule }        from '@angular/forms';
import { AwesomePipe }        from './awesome.pipe';
import { ContactComponent }   from './contact.component';
import { ContactService }     from './contact.service';
import { HighlightDirective } from './highlight.directive';
@NgModule({
  imports:      [ CommonModule, FormsModule ],
  declarations: [ ContactComponent, HighlightDirective, AwesomePipe ],
  exports:      [ ContactComponent ],
  providers:    [ ContactService ]
})
export class ContactModule { }
```

模块不会从其他模块继承声明的组件、指令和管道。`AppModule`引入的东西跟`AppModule`引入的是不相关的。`ContactComponent`要想使用`[(ngModel)]`绑定，那么`ContactModule`必须引入`FormsModule`。

这里`BrowserModule`替换成了`CommonModule`。原因见[FAQ](https://angular.io/docs/ts/latest/cookbook/ngmodule-faq.html#q-browser-vs-common-module)。

我们在模块的`declarations`中声明了模块的组件、指令和视图。
我们导出了`ContactComponent`，让其他引入`ContactModule`的模块能够使用它。其它的默认是私有的。`AwesomePipe`和`HighlightDirective`对应用的其它部分来说就是不可见的。现在`HighlightDirective`就不会作用到`AppComponent`的title了。

#### 重构AppModule

移除`AppComponent`中所有跟Contact相关的。

移除`imports`中的`FormsModule`，`AppComponent`不需要它。只保留需要的。

引入`ContactModule`，这样应用可以继续使用`ContactModule`导出的`ContactComponent`。

```js
// app/app.module.ts (v2)

import { NgModule }           from '@angular/core';
import { BrowserModule }      from '@angular/platform-browser';
/* App Root */
import
       { AppComponent }       from './app.component';
import { HighlightDirective } from './highlight.directive';
import { TitleComponent }     from './title.component';
import { UserService }        from './user.service';
/* Contact Imports */
import
       { ContactModule }      from './contact/contact.module';
@NgModule({
  imports:      [ BrowserModule, ContactModule ],
  declarations: [ AppComponent, HighlightDirective, TitleComponent ],
  providers:    [ UserService ],
  bootstrap:    [ AppComponent ],
})
export class AppModule { }
```

### 通过路由器延迟加载模块

现在应用包括两个模块了，Hero和Contact。我们想再加一个模块Crisis。然后在应用启动的时候加载`ContactComponent`。也就是说`ContactModule`保持贪婪式加载，而`HeroModule`和`CrisisModule`改为延迟加载。

修改下`AppComponent`的模板：一个标题、三个链接、一个`<router-outlet>`：

```js
// app/app.component.ts (v3 - Template)

template: `
  <app-title [subtitle]="subtitle"></app-title>
  <nav>
    <a routerLink="contact" routerLinkActive="active">Contact</a>
    <a routerLink="crisis"  routerLinkActive="active">Crisis Center</a>
    <a routerLink="heroes"  routerLinkActive="active">Heroes</a>
  </nav>
  <router-outlet></router-outlet>
`
```

#### 应用的路由

相应地修改`AppModule`：

```js
// app/app.module.ts (v3)

import { NgModule }           from '@angular/core';
import { BrowserModule }      from '@angular/platform-browser';
/* App Root */
import { AppComponent }       from './app.component.3';
import { HighlightDirective } from './highlight.directive';
import { TitleComponent }     from './title.component';
import { UserService }        from './user.service';
/* Feature Modules */
import { ContactModule }      from './contact/contact.module.3';
/* Routing Module */
import { AppRoutingModule }   from './app-routing.module.3';
@NgModule({
  imports:      [
    BrowserModule,
    ContactModule,
    AppRoutingModule
  ],
  providers:    [ UserService ],
  declarations: [ AppComponent, HighlightDirective, TitleComponent ],
  bootstrap:    [ AppComponent ]
})
export class AppModule { }
```

该模块仍然引入了`ContactModulE`，因为它的路由和组件要在应用启动的时候加载。

注意这里没有引入`HeroModule`和`CrisisModule`。它们将会在用户导航到它们的路由时动态地获取和加载。

这里还引入了`AppRoutingModule`，它是一个处理路由的模块：

```js
// app/app-routing.module.ts

import { NgModule }             from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'contact', pathMatch: 'full'},
  { path: 'crisis', loadChildren: 'app/crisis/crisis.module#CrisisModule' },
  { path: 'heroes', loadChildren: 'app/hero/hero.module#HeroModule' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

这个文件中定义了三个路由。

第一个路由是将空URL（例如`https://host.com`）重定向到`contact`(`https://host.com/contact`)。`contact`路由没有在这里定义，它是定义在Contact特性的自己的路由模块中（`contact-routing.module.ts`）。为拥有路由组件的特性模块定义一个它自己的路由文件可以说是一个最佳实践。

另外两个路由使用了延迟加载语法来告诉路由器去哪里找对应的模块。延迟加载模块的路径是一个字符串，也就是`#`后面的部分。

#### RouterModule.forRoot

`RouterModule`类的静态方法`forRoot`接收路由配置，将它添加到`imports`表明该模块是用来处理路由的。

返回的类`AppRoutingModule`就是一个路由模块，它包含`RouterModule`指令和可以产生配置好的`Router`的依赖注入provider。`AppRoutingModule`只是做为应用的根模块。

注：千万不要在特性路由模块中调用`RouterModule.forRoot`。

回到`AppModule`中，将`AppRoutingModule`添加到imports列表。这样应用就可以启动了。

```js
imports:      [
  BrowserModule,
  ContactModule,
  AppRoutingModule
],
```

#### 路由到特性模块

创建文件`app/contact/contact-routing.module.ts`，在这个文件里定义上面提到的`contact`路由以及提供`ContactRoutingModule`：

```js
// app/contact/contact-routing.module.ts (routing)

@NgModule({
  imports: [RouterModule.forChild([
    { path: 'contact', component: ContactComponent }
  ])],
  exports: [RouterModule]
})
export class ContactRoutingModule {}
```

这次是将路由配置清单传递给`RouterModule`的`forChild`方法。它是专门为特性模块设计的，只是为了提供额外的路由。

`ContactModule`中有两处细微但很重要的变化，它引入了`ContactRoutingModule`，并且不再导出`ContactComponent`：

```js
// app/contact/contact.module.3.ts

@NgModule({
  imports:      [ CommonModule, FormsModule, ContactRoutingModule ],
  declarations: [ ContactComponent, HighlightDirective, AwesomePipe ],
  providers:    [ ContactService ]
})
export class ContactModule { }
```

现在是通过路由导航到`ContactComponent`，所以没有理由再将它设置为公共的。它甚至都不需要selector了，因为没有模板引用它了。

#### 路由延迟加载的模块

延迟加载的`HeroModule`和`CrisisModule`同样遵循特性模块的相关原则，它们跟`ContactModule`其实看起来没有什么区别。

`HeroModule`的结构比`CrisisModule`稍微复杂一些，如下：

```
hero
|-hero-detail.component.ts
|-hero-list.component.ts
|-hero.component.ts
|-hero.module.ts
|-hero-routing.module.ts
|-hero.service.ts
|-highlight.directive.ts
```

这就是前面提到过的子路由的场景。`HeroComponent`是该特性的顶部组件并负责路由。它的模板中有个`<router-outlet>`来显示`HeroList`或`HeroDetail`。组件中数据的获取和保存都是代理给`HeroService`处理的。这里还加了一个`HighlightDirective`为元素设置一个不同背景色。等会会在共享模块中提到重复和不一致的问题。

`HeroModule`：

```js
// app/hero/hero.module.ts (class)

@NgModule({
  imports: [ CommonModule, FormsModule, HeroRoutingModule ],
  declarations: [
    HeroComponent, HeroDetailComponent, HeroListComponent,
    HighlightDirective
  ]
})
export class HeroModule { }
```

它引入了`FormsModule`，因为`HeroDetailComponent`模板中用到了`[(ngModel)]`。跟`ContactModule`和`CrisisModule`一样，它引入了`HeroRoutingModule`。

`CrisisModule`是类似的，不再细说。

```js
// app/crisis/crisis-detail.component.ts

import { Component, OnInit } from '@angular/core';
import { ActivatedRoute }    from '@angular/router';

@Component({
  template: `
    <h3 highlight>Crisis Detail</h3>
    <div>Crisis id: {{id}}</div>
    <br>
    <a routerLink="../list">Crisis List</a>
  `
})
export class CrisisDetailComponent implements OnInit {
  id: number;
  constructor(private route: ActivatedRoute) {  }

  ngOnInit() {
    this.id = parseInt(this.route.snapshot.params['id'], 10);
  }
}
```

```js
// app/crisis/crisis-list.component.ts

import { Component, OnInit } from '@angular/core';

import { Crisis,
         CrisisService }     from './crisis.service';

@Component({
  template: `
    <h3 highlight>Crisis List</h3>
    <div *ngFor='let crisis of crisises | async'>
      <a routerLink="{{'../' + crisis.id}}">{{crisis.id}} - {{crisis.name}}</a>
    </div>
  `
})
export class CrisisListComponent implements OnInit {
  crisises: Promise<Crisis[]>;

  constructor(private crisisService: CrisisService) { }

  ngOnInit() {
    this.crisises = this.crisisService.getCrises();
  }
}
```

```js
// app/crisis/crisis-routing.modlue.ts

import { NgModule }            from '@angular/core';
import { Routes,
         RouterModule }        from '@angular/router';

import { CrisisListComponent }    from './crisis-list.component';
import { CrisisDetailComponent }  from './crisis-detail.component';

const routes: Routes = [
  { path: '', redirectTo: 'list', pathMatch: 'full'},
  { path: 'list',    component: CrisisListComponent },
  { path: ':id', component: CrisisDetailComponent }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class CrisisRoutingModule {}
```

```js
// app/crisis/crisis.modlue.ts

import { NgModule }      from '@angular/core';
import { CommonModule }  from '@angular/common';

import { CrisisListComponent }    from './crisis-list.component';
import { CrisisDetailComponent }  from './crisis-detail.component';
import { CrisisService }          from './crisis.service';
import { CrisisRoutingModule }    from './crisis-routing.module';

@NgModule({
  imports:      [ CommonModule, CrisisRoutingModule ],
  declarations: [ CrisisDetailComponent, CrisisListComponent ],
  providers:    [ CrisisService ]
})
export class CrisisModule {}
```

```js
// app/crisis/crisis.service.ts

import { Injectable } from '@angular/core';

export class Crisis {
  constructor(public id: number, public name: string) { }
}

const CRISES: Crisis[] = [
  new Crisis(1, 'Dragon Burning Cities'),
  new Crisis(2, 'Sky Rains Great White Sharks'),
  new Crisis(3, 'Giant Asteroid Heading For Earth'),
  new Crisis(4, 'Procrastinators Meeting Delayed Again'),
];

const FETCH_LATENCY = 500;

@Injectable()
export class CrisisService {

  getCrises() {
    return new Promise<Crisis[]>(resolve => {
      setTimeout(() => { resolve(CRISES); }, FETCH_LATENCY);
    });
  }

  getCrisis(id: number | string) {
    return this.getCrises()
      .then(heroes => heroes.find(hero => hero.id === +id));
  }

}
```

### 共享模块

前面遗留了一个问题，我们一共实现了三个版本的`HighlightDirective`。让我们添加一个共享模块来放这些常用的组件、指令和管道，然后将它们共享给所需要的模块。

创建`app/shared`文件夹，将`app/contact`中的`AwesomePipe`和`HighlightDirective`移过来，删掉`app/`和`app/hero`中的`HighlightDirective`。创建`SharedModule`来管理这些共享的东西：

```js
// app/app/shared/shared.module.ts

import { NgModule }            from '@angular/core';
import { CommonModule }        from '@angular/common';
import { FormsModule }         from '@angular/forms';
import { AwesomePipe }         from './awesome.pipe';
import { HighlightDirective }  from './highlight.directive';
@NgModule({
  imports:      [ CommonModule ],
  declarations: [ AwesomePipe, HighlightDirective ],
  exports:      [ AwesomePipe, HighlightDirective,
                  CommonModule, FormsModule ]
})
export class SharedModule { }
```

要注意的是：
* 这里引入了`CommonModule`，因为它的组件中需要用到一些常用指令
* 它声明并导出了工具性质的管道、指令和组件，正如我们所期望的
* 它还重新导出了`CommonModule`和`FormsModule`

#### 重新导出常用模块

回顾整个应用，当很多组件需要`SharedModule`中的指令时同时也用到`CommonModule`中的`NgIf`或`NgFor`以及`FormsModule`中的`[(ngModel)]`。这些组件的声明中必须引入`CommonModule`、`FormsModule`和`SharedModule`。

我们可以通过在`SharedModule`重新导出那两个模块来减少这个重复。

目前`SharedModule`中的组件没有使用`[(ngModel)]`。因此从技术上它不需要引入`FormsModule`。但是它还是可以导出`FormsModule`的。

#### 为什么UserService不共享

尽管很多组件会共享同一个服务实例，但是这是靠依赖注入来实现的一种共享，而不是模块系统。

在我们的例子中有好几个组件注入了`UserService`。它们应该是整个应用的同一个服务实例。也就是说，`UserService`是一个应用级别的单例。我们不希望每个模块都有它自己的一个实例。

如果`SharedModule`提供`UserService`就会导致延迟加载的模块注入它自己的`UserService`实例。

### 核心模块

现在在根文件夹比较杂乱，有`UserService`和只在`AppComponent`中出现的`TitleComponent`。而他们又不适合放到`SharedModule`中。

因此我们将它们放到一个单独的模块`CoreModule`中，在这里面放那些只在应用启动的时候会import一次而其它任何地方都不会再用到的组件。

创建文件夹`app/core`。将`UserService`和`TitleComponent`从`app/`移到`app/core`中。创建`CoreModule`。更新`AppRoot`模块，引入`CoreModule`。

```js
// app/app/core/core.module.ts

import {
  ModuleWithProviders, NgModule,
  Optional, SkipSelf }       from '@angular/core';
import { CommonModule }      from '@angular/common';
import { TitleComponent }    from './title.component';
import { UserService }       from './user.service';
@NgModule({
  imports:      [ CommonModule ],
  declarations: [ TitleComponent ],
  exports:      [ TitleComponent ],
  providers:    [ UserService ]
})
export class CoreModule {
}
```

重构了`CoreModule`和`SharedModule`之后，其它的模块也要做一些清理。

更新后的`AppModule`：

```js
// app/app.module.ts (v4)

import { NgModule }       from '@angular/core';
import { BrowserModule }  from '@angular/platform-browser';
/* App Root */
import { AppComponent }   from './app.component';
/* Feature Modules */
import { ContactModule }    from './contact/contact.module';
import { CoreModule }       from './core/core.module';
/* Routing Module */
import { AppRoutingModule } from './app-routing.module';
@NgModule({
  imports: [
    BrowserModule,
    ContactModule,
    CoreModule,
    AppRoutingModule
  ],
  declarations: [ AppComponent ],
  bootstrap:    [ AppComponent ]
})
export class AppModule { }
```

更新后的`ContactModule`：

```js
// app/contact/contact.module.ts (v4)

import { NgModule }           from '@angular/core';
import { SharedModule }       from '../shared/shared.module';
import { ContactComponent }     from './contact.component';
import { ContactService }       from './contact.service';
import { ContactRoutingModule } from './contact-routing.module';
@NgModule({
  imports:      [ SharedModule, ContactRoutingModule ],
  declarations: [ ContactComponent ],
  providers:    [ ContactService ]
})
export class ContactModule { }
```

### 使用CoreModule.forRoot配置核心服务

一个向应用添加provider的模块也可以提供配置这些provider的东西。

根据约定，静态方法`forRoot`可以同时提供和配置服务，它接收一个服务配置对象，返回一个`ModuleWithProviders`，这个对象有两个属性：

* `ngModule`：`CoreModule`类
* `providers`：配置好的provider

根模块`AppModule`引入`CoreModule`并将`providers`添加到`AppModule`的providers中。准确的说，Angular在附加`@NgModule.providers`列表中的项之前会累积所有引入的provider。这个顺序确保了我们显式添加到`AppModule`的providers中的优先于引入的模块的provider。

现在添加一个`CoreModule.forRoot`方法来配置`UserService`。

将`UserService`扩展成注入一个可选的`UserServiceConfig`。如果`UserServiceConfig`存在，那么`UserService`就从这个配置中获取并设置用户名：

```js
// app/core/user.service.ts (constructor)

constructor(@Optional() config: UserServiceConfig) {
  if (config) { this._userName = config.userName; }
}
```

这里是接收`UserServiceConfig`对象的`CoreModule.forRoot`：

```js
// app/core/core.module.ts (forRoot)

static forRoot(config: UserServiceConfig): ModuleWithProviders {
  return {
    ngModule: CoreModule,
    providers: [
      {provide: UserServiceConfig, useValue: config }
    ]
  };
}
```

最后，在`AppModule`的`imports`中调用它：

```js
// app//app.module.ts (imports)

  imports: [
    BrowserModule,
    ContactModule,
    CoreModule.forRoot({userName: 'Miss Marple'}),
    AppRoutingModule
  ],
```

### 阻止CoreModule的重复引入

只有根模块`AppModule`应该引入`CoreModule`。延迟加载的模块引用它会出现一些奇怪的问题。

我们可以寄希望与开发者不犯这个错误。或者我们通过添加`CoreModule`构造函数让发现问题快速暴露出来。

```js
constructor (@Optional() @SkipSelf() parentModule: CoreModule) {
  if (parentModule) {
    throw new Error(
      'CoreModule is already loaded. Import it in the AppModule only');
  }
}
```

构造函数中将自己注入给了自己。这看起来是个很奇怪的循环。

如果Angular实在当前的注入器中查找`CoreModule`，那么这的确是个死循环。装饰器`@SkipSelf`意思是“在父层级的注入器中查找`for CoreModule`”。

如果这个构造函数实在我们所期望的`AppModule`中执行的，那么是没有父注入器可以提供`CoreModule`的。这样构造函数就不会注入了。

默认注入器在找不到所需要的provider时会报错。而装饰器`@Optional`则表示这不到也没问题，就注入null。

如果我们在延迟加载的模块中（例如`HeroModule`）引入了`CoreModule`。Angular会为延迟加载的模块创建一个注入器，它是根注入器的子节点。`@SkipSelf`会让Angular在根注入器中查找`CoreModule`，当然它能找到，那么`parentModule`就不会null，这时构造函数就抛出错误。

---

参考资料：

[Angular2 Documentation - Angular Modules](https://angular.io/docs/ts/latest/guide/ngmodule.html)
