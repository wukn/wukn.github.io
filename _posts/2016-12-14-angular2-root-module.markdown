---
layout:     post
title:      "[Angular2] Root Module"
subtitle:   ""
date:       2016-12-14 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 根模块是用来告诉Angular如何构建和启动应用的。

一个Angular的模块类描述了应用的各部分是如何组装在一起的。每个应用至少有一个模块，那就是负责启动应用的根模块。根模块的名字可以任意取，通常命名为`AppModule`。

```js
// app/app.module.js

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

在`import`语句后是一个带有`@NgModule`装饰器的类。

`@NgModule`装饰器表明`AppModule`是一个Angular模块类（也叫[NgModule](https://angular.io/docs/ts/latest/guide/ngmodule.html)类）。`@NgModule`接收的元数据对象可以告诉Angular如何编译和启动应用。

* `imports`：`BrowserModule`是每个应用在浏览器里运行所必需的模块
* `declarations`：应用自身的模块
* `bootstrap`：Angular会创建和插入`index.html`的根组件

### imports数组

Angular模块将一些整合的功能分散成很多更小的单元。很多Angular自己的功能也是被组织成模块。例如，HTTP服务在`HttpModule`中。路由功能在`RouterModule`中。

当应用需要用到某个模块的功能时，就把它加到`imports`数组中。

注意，只有`NgModule`类才能放到`imports`数组中。

### declarations数组

每个组件必须在一个（且只能在一个）Ngmodule类中声明。通过模块的`declarations`数组告诉Angular哪些组件属于这个模块。

另外两种必须添加到`declarations`数组的类是指令（directive）和管道（pipe）。

注意，只有组件、指令和管道可以被添加到`declarations`数组。`NgModule`类、service类、model类都是不能放进去的。

### `bootstrap`数组中列出的组件并添加到浏览器的DOM中。

应用的启动是通过启动根模块`AppModule`来启动的。bootstrapping进程会创建`bootstrap`数组中列出的组件并添加到浏览器的DOM中。

每个启动的组件又是又是它自己的组件树的根。因此，添加一个启动的组件通常会级联触发组件的创建，形成整个组件树。

可以在Web页面中放多个组件树，但这种情况不常见。大部分应用只有一个组件树，启动唯一的根组件。

### 在main.ts中启动

其实启动应用的方式有很多种，取决于你想怎样编译应用和运行它。

之前我们是用JIT编译器动态编译然后在浏览器中运行的。对于JIT编译好的浏览器应用，推荐将启动的代码放在一个单独的文件`app/main.ts`中。

```js
// app/main.ts

import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule }              from './app.module';

platformBrowserDynamic().bootstrapModule(AppModule);
```

上面的代码创建了一个可动态编译和启动`AppModule`的平台。

启动的过程是，设置执行的环境，找到`AppModule`模块的`bootstrap`数组中的根组件`AppComponent`，创建一个组件实例并添加到这个组件的`selector`所标识的元素标签中。

例如，`AppComponent`的selector是`my-app`，那么Angular就会在`index.html`中寻找`<my-app>`标签：

```html
<my-app><!-- content managed by Angular --></my-app>
```

然后在这个标签里显示`AppComponent`。

`main.ts`这个文件通常设置好了之后就不会再动它了。

---

参考资料：

[Angular2 Documentation - Root Module](https://angular.io/docs/ts/latest/guide/appmodule.html)
