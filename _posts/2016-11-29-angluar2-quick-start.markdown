---
layout:     post
title:      "[Angular2] Quick Start"
subtitle:   ""
date:       2016-11-29 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 使用TypeScript构建Angular2应用。

### 组件基本介绍

Angular2的应用是由组件（component）组成的。一个组件包括HTML模板和控制显示的组件类两部分。例如：

{% raw %}
```js
// app/app.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'my-app',
  template: `<h1>Hello {{name}}</h1>`
})
export class AppComponent { name = 'Angular'; }
```
{% endraw %}

每一个组件以一个装饰器函数`@Component`为开始，这个装饰器函数接受一个元数据对象，这个元数据对象描述了HTML模板和组件类是如何工作的。

元数据对象的selector属性告诉Angular在index.html文件的自定义标签`<my-app>`内部显示该组件。

```html
<!-- index.html -->

<my-app>Loading AppComponent content here ...</my-app>
```

元数据对象的template属性指定了组件对应的模板。模板使用了插值绑定表达式来进行数据绑定。Angular会用组件的name属性值来替换`{% raw %}{{name}}{% endraw %}`。

### 搭建开发环境

最便捷的搭建开发环境的办法是，从github上clone一个基础代码到本地，然后安装npm包并运行`npm start`来运行样例程序。

```bash
git clone https://github.com/angular/quickstart.git quickstart
cd quickstart
npm install
npm start
```

先看下`/app`文件夹下的三个TypeScript文件：

{% raw %}
```js
// app/app.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'my-app',
  template: `<h1>Hello {{name}}</h1>`
})
export class AppComponent { name = 'Angular'; }
```
{% endraw %}

```js
// app/app.module.ts

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

```js
// app/main.ts

import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule }              from './app.module';

platformBrowserDynamic().bootstrapModule(AppModule);
```

`app.component.ts`文件定义了一个名为`AppComponent`的组件，它是应用的根组件。

`app.module.ts`文件定义了一个名为`AppModule`的模块，它用来告诉Angular如何组装应用。

`main.ts`文件使用JiT编译器编译整个应用，并启动应用，运行在浏览器中。后续会看到其它的编译和部署方式。

在浏览器中访问http://localhost:3000，结果如下：

![](/img/post/angular2/quick-start/angular2-quick-start.png)

---

参考资料：

[Angular2 Documentation]()
