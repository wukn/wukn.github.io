---
author: wukn
catalog: true
date: 2017-01-04T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Case Study - Routing'
url: /2017/01/04/angular2-case-study-routing/
---

> 使用TypeScript构建Angular2应用。使用路由功能在多个视图之间导航。

<!--more-->

现在我们接收到新的需求：

1. 添加一个Dashboard视图
2. 在Heroes列表视图和Dashboard视图之间跳转
3. 点击列表视图或Dashboard视图内的Hero跳转到Hero详细视图
4. URL直接访问某个Hero相信信息视图

![](/img/post/angular2/routing/nav-diagram.png)

我们将使用Angular的Router来实现该需求。

我们打算这样做：

1. 将`AppComponent`组件改成只处理页面跳转的应用程序壳
2. 将`AppComponent`中的Heroes相关的业务分离到`HeroesComponent`中
3. 添加路由
4. 创建新的`DashboardComponent`
5. 将Dashboard添加到导航结构中

注：Routing就是Navigation（导航/跳转）的另一个名字。Router就是从一个视图导航到另一个视图的一种机制。

### 重构AppComponent

目前应用程序启动`AppComponent`后会立刻加载和显示heroes列表。我们要将它改成包含一系列视图的壳，然后默认加载某一个视图。

`AppComponent`应该只处理视图间的跳转，因此要将它里面的显示heroes列表的业务逻辑移到`HeroesComponent`中。

我们`app.component.ts`重命名为`heroes.component.ts`，并将该文件中的类名`AppComponent`改为`HeroesComponent`，选择器`my-app`改为`my-heroes`ruinous创建一个新的`app.component.ts`文件就可以了。

创建新的文件`app/app.component.ts`，定义`AppComponent`类，在模板中使用`<my-heroes>`元素：

```js
// app/app.component.ts

import { Component } from '@angular/core';
@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <my-heroes></my-heroes>
  `
})
export class AppComponent {
  title = 'Tour of Heroes';
}
```

`AppModule`中也要做一些修改。在`declarations`数组中添加`HeroesComponent`，要不Angular不认识`my-heroes`标签;在`providers`数组中添加`HeroService`，因为在其它每个视图中都要用到它；添加相应的文件的import：

```js
// app/app.module.ts

import { NgModule }       from '@angular/core';
import { BrowserModule }  from '@angular/platform-browser';
import { FormsModule }    from '@angular/forms';
import { AppComponent }        from './app.component';
import { HeroDetailComponent } from './hero-detail.component';
import { HeroesComponent }     from './heroes.component';
import { HeroService }         from './hero.service';
@NgModule({
  imports: [
    BrowserModule,
    FormsModule
  ],
  declarations: [
    AppComponent,
    HeroDetailComponent,
    HeroesComponent
  ],
  providers: [
    HeroService
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule {
}
```

还要移除`HeroesComponent`的模板中`<h1>{{title}}</h1>`；移除的`providers`数组中的`HeroService`。因为这两个都移到`AppComponent`中去了。

现在应用程序跑起来还是跟之前的效果是一样的。说明重构生效了。

### 添加路由

接下来就是添加路由了。

Angular路由器是一个外部的可选模块`RouterModule`。路由器包括 multiple provided services（`RouterModule`）、multiple directives（`RouterOutlet`，`RouterLink`，`RouterLinkActive`）和configuration (`Routes`)。

#### 添加base标签

在文件`index.html`的`<head>`段的顶部添加`<base href="/">`：

```html
<head>
  <base href="/">
```

#### 配置路由

现在应用里还没有路由（Route），我们先创建路由的配置。

路由（Route）是用来告诉路由器（Router）当用户点击某个链接或在浏览器的地址栏输入URL后该显示哪个视图。

```js
// app/app.module.ts

import { RouterModule }   from '@angular/router';

RouterModule.forRoot([
  {
    path: 'heroes',
    component: HeroesComponent
  }
])
```

路由就是一个路由定义数组。目前我们只加了一个路由定义。

路由定义包含两部分：

* `path`：路由器使用path来匹配URL
* `component`：当导航到路由时路由器该创建的组件

#### 使用路由器

我们创建了路由配置。现在需要将它添加到`AppModule`。我们将配置的`RouterModule`添加到`AppModule`的`imports`数组中。

```js
// app/app.module.ts

import { NgModule }       from '@angular/core';
import { BrowserModule }  from '@angular/platform-browser';
import { FormsModule }    from '@angular/forms';
import { RouterModule }   from '@angular/router';

import { AppComponent }        from './app.component';
import { HeroDetailComponent } from './hero-detail.component';
import { HeroesComponent }     from './heroes.component';
import { HeroService }         from './hero.service';

@NgModule({
  imports: [
    BrowserModule,
    FormsModule,
    RouterModule.forRoot([
      {
        path: 'heroes',
        component: HeroesComponent
      }
    ])
  ],
  declarations: [
    AppComponent,
    HeroDetailComponent,
    HeroesComponent
  ],
  providers: [
    HeroService
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule {
}
```

这里使用了`forRoot`方法，因为我们是在为应用的根路径提供配置的路由器。

如果我们在浏览器的地址栏中添加`/heroes`，那么这个URL就匹配到`heroes`这个路由并显示`HeroesComponent`。但是在哪里显示呢？

我们必须在模板的底部添加一个`<router-outlet>`元素来显示路由对应的视图。`RouterOutlet`是`RouterModule`提供的一个指令。

#### 路由链接

我们不希望每次用户都得在浏览器的地址栏中去输入和粘贴URL来跳转，所以在模板中添加一个链接标签，点击该链接触发跳转到`HeroesComponent`。

```js
// app/app.component.ts(partial)

template: `
   <h1>{{title}}</h1>
   <a routerLink="/heroes">Heroes</a>
   <router-outlet></router-outlet>
 `
```

注意，我们绑定的是链接标签的`routerLink`。`RouterLink`是`RouterModule`提供的另一个指令，告诉路由器当用户点击这个链接是该跳转到哪里。

刷新浏览器，我们将应用的标题和heroes链接，不会看到heroes列表。

![](/img/post/angular2/routing/route-root.png)

点击链接后，浏览器的地址栏中会更新成`/heroes`，heroes列表也会显示出来了。

![](/img/post/angular2/routing/route-heroes.png)

### 添加Dashboard

路由只有在多个视图需要切换时才有用。所有再创建一个新的视图。

新建`app/dashboard.component.ts`文件：

```js
// app/dashboard.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'my-dashboard',
  template: '<h3>My Dashboard</h3>'
})
export class DashboardComponent { }
```

回到`app.module.ts`中，在路由定义数组中添加一条路由：

```js
{
  path: 'dashboard',
  component: DashboardComponent
},
```

并在`declarations`数组中添加`DashboardComponent`：

```js
declarations: [
  AppComponent,
  DashboardComponent,
  HeroDetailComponent,
  HeroesComponent
],
```

我们希望在应用启动的时候就默认显示Dashboard。也就是将`/`重定向到`/dashboard`：

```js
{
  path: '',
  redirectTo: '/dashboard',
  pathMatch: 'full'
},
```

在模板中添加导航链接：

```js
template: `
   <h1>{{title}}</h1>
   <nav>
     <a routerLink="/dashboard">Dashboard</a>
     <a routerLink="/heroes">Heroes</a>
   </nav>
   <router-outlet></router-outlet>
 `
```

这里将两个链接放到了`<nav>`标签里，是为了方便后面添加样式。

现在显示的效果如下：

![](/img/post/angular2/routing/route-dashboard.png)

#### 为Dashboard添加内容

在Dashboard中显示最前面的四个hero。这里我们将模板放到单独的文件中，创建文件`dashboard.component.html`：

```html
<h3>Top Heroes</h3>
<div class="grid grid-pad">
  <div *ngFor="let hero of heroes" class="col-1-4">
    <div class="module hero">
      <h4>{{hero.name}}</h4>
    </div>
  </div>
</div>
```

配置`DashboardComponent`的模板，设置`moduleId`属性，并将`template`改成`templateUrl`：

```js
@Component({
  moduleId: module.id,
  selector: 'my-dashboard',
  templateUrl: 'dashboard.component.html',
})
```

修改`dashboard.component.ts`文件，添加import：

```js
import { Hero } from './hero';
import { HeroService } from './hero.service';
```

实现`DashboardComponent`类：

```js
export class DashboardComponent implements OnInit {

  heroes: Hero[] = [];

  constructor(private heroService: HeroService) { }

  ngOnInit(): void {
    this.heroService.getHeroes()
      .then(heroes => this.heroes = heroes.slice(1, 5));
  }
}
```

现在刷新浏览器，可以看到Dashboard会显示四个hero。

![](/img/post/angular2/routing/route-dashboard2.png)

#### 导航到详细信息视图

尽管我们在`HeroesComponent`中可以显示Hero详细信息视图，但是在需求中制定的三种方式跳转到`HeroDetailComponent`还没有实现：

* 从Dashboard选中的Hero
* 从Heroes列表中选中的Hero
* URL直接访问Hero

像上面一样，添加一个hero-detail路由。

这里有点特别的是，我们需要告诉`HeroDetailComponent`该显示哪个Hero，目前`HeroesComponent`是直接通过属性传递参数的，而`DashboardComponent`还不知道任何要传递的参数。

```js
<my-hero-detail [hero]="selectedHero"></my-hero-detail>
```

属性绑定这种方式在路由的场景里似乎也不行。我们不能将整个hero对象放到URL中，也不希望这样做。

#### 参数化的路由

我们可以将hero的`id`放到URL中作为参数。比如访问`id`为11的hero：

```
/detail/11
```

添加路由定义：

```js
{
  path: 'detail/:id',
  component: HeroDetailComponent
},
```

`:id`表示这是一个路由参数，当导航到`HeroDetailComponent`时将`id`值传给该组件。

现在路由是不可用的，`HeroDetailComponent`还需要很大的修整。

目前的`hero-detail.component.ts`是这样的：

```js
import { Component, Input } from '@angular/core';
import { Hero } from './hero';
@Component({
  selector: 'my-hero-detail',
  template: `
    <div *ngIf="hero">
      <h2>{{hero.name}} details!</h2>
      <div>
        <label>id: </label>{{hero.id}}
      </div>
      <div>
        <label>name: </label>
        <input [(ngModel)]="hero.name" placeholder="name"/>
      </div>
    </div>
  `
})
export class HeroDetailComponent {
  @Input() hero: Hero;
}
```

新的`HeroDetailComponent`应该不再接受Hero对象作为参数，而是从`ActivatedRoute`的`params`中得到的参数`id`，然后使用`HeroService`获取对应id的hero。

添加相应的import：

```js
// Keep the Input import for now, we'll remove it later:
import { Component, Input, OnInit } from '@angular/core';
import { ActivatedRoute, Params }   from '@angular/router';
import { Location }                 from '@angular/common';

import { HeroService } from './hero.service';
```

在构造函数中注入`ActivatedRoute`、`HeroService`和`Location`这三个服务：

```js
constructor(
  private heroService: HeroService,
  private route: ActivatedRoute,
  private location: Location
) {}
```

还要引入`switchMap`操作符，处理路由的参数`Observable`时要用到：

```js
import 'rxjs/add/operator/switchMap';
```

实现`OnInit`接口：

```js
export class HeroDetailComponent implements OnInit {
  ...
}
```

```js
ngOnInit(): void {
  this.route.params
    .switchMap((params: Params) => this.heroService.getHero(+params['id']))
    .subscribe(hero => this.hero = hero);
}
```

注意，`switchMap`操作符将路由参数observable中的id映射成一个新的`Observable`，也就是`HeroService.getHero`方法的返回值。

如果用户重新导航到该组件个`getHero`方法还没执行完，那么`switchMap`会在重新调用`HeroService.getHero`之前取消旧的请求。

Hero的`id`属性是数字，而路由参数始终是字符串。这里用JavaScript的`+`操作符将字符串转换成数字。

`Router`会管理它提供的observables并使得订阅局部化。当组件被销毁时订阅也会被清除，为了避免内存泄露。

添加`HeroService.getHero`：

```js
getHero(id: number): Promise<Hero> {
  return this.getHeroes()
             .then(heroes => heroes.find(hero => hero.id === id));
}
```

我们有好几种方式导航到`HeroDetailComponent`，但是当我们完成操作后想跳转到其它地方怎么办？

一种方法是点击`AppComponent`上面的两个链接。我们还可以添加一个`goBack`方法跳转回之前的页面：

```js
// app/hero-detail.component.ts (goBack)

goBack(): void {
  this.location.back();
}
```

在模板中添加回退按钮：

```html
<button (click)="goBack()">Back</button>
```

这里我们顺便做一下重构，将模板放到独立的文件中：

```html
<!-- app/hero-detail.component.html -->

<div *ngIf="hero">
  <h2>{{hero.name}} details!</h2>
  <div>
    <label>id: </label>{{hero.id}}</div>
  <div>
    <label>name: </label>
    <input [(ngModel)]="hero.name" placeholder="name" />
  </div>
  <button (click)="goBack()">Back</button>
</div>
```

配置组件元数据中的`moduleId`和`templateUrl`：

```js
@Component({
  moduleId: module.id,
  selector: 'my-hero-detail',
  templateUrl: 'hero-detail.component.html',
})
```

#### Dashboard跳转到详细信息视图

当我们在Dashboard视图中点击一个Hero时也应该跳转到对应的`HeroDetailComponent`视图。

修改`dashboard.component.html`，将`<div *ngFor...>`标签改成可点击的`<a>`标签，并为每个hero添加路由链接：

```html
<a *ngFor="let hero of heroes"  [routerLink]="['/detail', hero.id]"  class="col-1-4">
```

看这里的`[routerLink]`绑定。之前绑定的都是固定链接，如/dashboard"和"/heroes"。这里绑定的是一个包含链接参数数组的表达式。数组包含两个元素，path是目标路由，路由参数会赋值对应的hero的id值。

刷新浏览器可以看到效果。

### 重构路由模块

现在`AppModule`中有很大一部分的代码都是路由相关的。大多数应用会有很多路由配置，并且会添加安全拂来来避免未授权的链接等。这样就违背了`AppModule`的本意：为Angular编译器和整个应用建立连接。

所以我们要将路由配置重构到单独的类中。那么这个类是什么类型呢？`RouterModule.forRoot()`产生的是`ModuleWithProviders`。这意味着路由配置的类应该是某个模块。实际上应该是路由模块（Routing Module）。

创建`app-routing.module.ts`文件：

```js
import { NgModule }             from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { DashboardComponent }   from './dashboard.component';
import { HeroesComponent }      from './heroes.component';
import { HeroDetailComponent }  from './hero-detail.component';
const routes: Routes = [
  { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
  { path: 'dashboard',  component: DashboardComponent },
  { path: 'detail/:id', component: HeroDetailComponent },
  { path: 'heroes',     component: HeroesComponent }
];
@NgModule({
  imports: [ RouterModule.forRoot(routes) ],
  exports: [ RouterModule ]
})
export class AppRoutingModule {}
```

值得注意的地方有，一般路由模块会：

* 将路由定义到一个变量中。未来可以将它export。
* 在`imports`中添加`RouterModule.forRoot(routes)`。
* 在`exports`中添加`RouterModule`。这个该模块相关的模块就可以直接使用`RouterLink`和`RouterOutlet`等。
* 没有`declarations`。这是相关模块的职责。
* 如果有一些安全性的service，添加到`providers`。本例中没有。

将`AppModule`中的路由配置删掉，再引入`AppRoutingModule`：

```js
import { NgModule }       from '@angular/core';
import { BrowserModule }  from '@angular/platform-browser';
import { FormsModule }    from '@angular/forms';
import { AppComponent }         from './app.component';
import { DashboardComponent }   from './dashboard.component';
import { HeroDetailComponent }  from './hero-detail.component';
import { HeroesComponent }      from './heroes.component';
import { HeroService }          from './hero.service';
import { AppRoutingModule }     from './app-routing.module';
@NgModule({
  imports: [
    BrowserModule,
    FormsModule,
    AppRoutingModule
  ],
  declarations: [
    AppComponent,
    DashboardComponent,
    HeroDetailComponent,
    HeroesComponent
  ],
  providers: [ HeroService ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```
#### HeroesComponent跳转到详细信息视图

跟Dashboard一样，我们要处理一下`HeroesComponent`中点击某个Hero后的跳转。

目前`HeroesComponent`的模板是master/detail样式的。

```js
template: `
  <h1>{{title}}</h1>
  <h2>My Heroes</h2>
  <ul class="heroes">
    <li *ngFor="let hero of heroes"
      [class.selected]="hero === selectedHero"
      (click)="onSelect(hero)">
      <span class="badge">{{hero.id}}</span> {{hero.name}}
    </li>
  </ul>
  <my-hero-detail [hero]="selectedHero"></my-hero-detail>
`,
```

我们的目标是当用户决定要编辑选中的hero时跳转到详细视图。

将顶部的`<h1>`标题删掉，标题移到`AppComponent`中了。删除最后一行`<my-hero-detail>`标签，我们不再需要在这里显示`HeroDetailComponent`了。

我们将这个视图里的detail改成一个mini的只读的视图。将之前`<my-hero-detail>`的地方替换成下面的内容：

```html
<div *ngIf="selectedHero">
  <h2>
    {{selectedHero.name | uppercase}} is my hero
  </h2>
  <button (click)="gotoDetail()">View Details</button>
</div>
```

现在点击后的效果如下：

![](/img/post/angular2/routing/mini-detail.png)

还没完呢。我们还要处理View Details按钮的点击事件。

但是现在这个文件的内容已经很多了，所以我们重构一下：

1. 将模板放到单独的`heroes.component.html`文件中
2. 将样式放到单独的`heroes.component.css`文件中
3. 将元数据的`templateUrl`和`styleUrls`指向对应的文件
4. 设置`moduleId`属性值为`module.id`，这样`templateUrl`和`styleUrls`才跟组件相关联

```js
@Component({
  moduleId: module.id,
  selector: 'my-heroes',
  templateUrl: 'heroes.component.html',
  styleUrls: [ 'heroes.component.css' ]
})
```

`HeroesComponent`跳转到`HeroDetailComponent`是通过点击按钮的。按钮绑定了`gotoDetail`方法：

```js
// app/heroes.component.ts (gotoDetail)

gotoDetail(): void {
  this.router.navigate(['/detail', this.selectedHero.id]);
}
```

在这个方法之前要先引入`Router`并在构造函数中注入。

```js
import { Router } from '@angular/router';
```

```js
constructor(
  private router: Router,
  private heroService: HeroService) { }
```

注意`gotoDetail`方法中的`navigate`，我们传递的参数是包含两个元素的链接参数数组，路径和路由参数。

到这里我们就实现了所有的需求。可以在浏览器中看一下效果。

### 添加样式

`dashboard.component.css`：

```css
[class*='col-'] {
  float: left;
  padding-right: 20px;
  padding-bottom: 20px;
}
[class*='col-']:last-of-type {
  padding-right: 0;
}
a {
  text-decoration: none;
}
*, *:after, *:before {
  -webkit-box-sizing: border-box;
  -moz-box-sizing: border-box;
  box-sizing: border-box;
}
h3 {
  text-align: center; margin-bottom: 0;
}
h4 {
  position: relative;
}
.grid {
  margin: 0;
}
.col-1-4 {
  width: 25%;
}
.module {
  padding: 20px;
  text-align: center;
  color: #eee;
  max-height: 120px;
  min-width: 120px;
  background-color: #607D8B;
  border-radius: 2px;
}
.module:hover {
  background-color: #EEE;
  cursor: pointer;
  color: #607d8b;
}
.grid-pad {
  padding: 10px 0;
}
.grid-pad > [class*='col-']:last-of-type {
  padding-right: 20px;
}
@media (max-width: 600px) {
  .module {
    font-size: 10px;
    max-height: 75px; }
}
@media (max-width: 1024px) {
  .grid {
    margin: 0;
  }
  .module {
    min-width: 60px;
  }
}
```

`hero-detail.component.css`：

```css
label {
  display: inline-block;
  width: 3em;
  margin: .5em 0;
  color: #607D8B;
  font-weight: bold;
}
input {
  height: 2em;
  font-size: 1em;
  padding-left: .4em;
}
button {
  margin-top: 20px;
  font-family: Arial;
  background-color: #eee;
  border: none;
  padding: 5px 10px;
  border-radius: 4px;
  cursor: pointer; cursor: hand;
}
button:hover {
  background-color: #cfd8dc;
}
button:disabled {
  background-color: #eee;
  color: #ccc;
  cursor: auto;
}
```

`app.component.css`：

```css
h1 {
  font-size: 1.2em;
  color: #999;
  margin-bottom: 0;
}
h2 {
  font-size: 2em;
  margin-top: 0;
  padding-top: 0;
}
nav a {
  padding: 5px 10px;
  text-decoration: none;
  margin-top: 10px;
  display: inline-block;
  background-color: #eee;
  border-radius: 4px;
}
nav a:visited, a:link {
  color: #607D8B;
}
nav a:hover {
  color: #039be5;
  background-color: #CFD8DC;
}
nav a.active {
  color: #039be5;
}
```

记得在Component类的元数据中配置`StyleUrls`。`AppComponent`还要配置`moduleId`。

#### routerLinkActive

Angular的路由器提供了一个`routerLinkActive`指令，可用来为激活的那个路由添加样式：

```js
template: `
  <h1>{{title}}</h1>
  <nav>
    <a routerLink="/dashboard" routerLinkActive="active">Dashboard</a>
    <a routerLink="/heroes" routerLinkActive="active">Heroes</a>
  </nav>
  <router-outlet></router-outlet>
`,
```

#### 全局样式

`style.css`：

```css
/* Master Styles */
h1 {
  color: #369;
  font-family: Arial, Helvetica, sans-serif;
  font-size: 250%;
}
h2, h3 {
  color: #444;
  font-family: Arial, Helvetica, sans-serif;
  font-weight: lighter;
}
body {
  margin: 2em;
}
body, input[text], button {
  color: #888;
  font-family: Cambria, Georgia;
}
/* . . . */
/* everywhere else */
* {
  font-family: Arial, Helvetica, sans-serif;
}
```

在`index.html`中引用：

```html
<link rel="stylesheet" href="styles.css">
```

---

参考资料：

[Angular2 Documentation - Case Study - Routing](https://angular.io/docs/ts/latest/tutorial/toh-pt5.html)
