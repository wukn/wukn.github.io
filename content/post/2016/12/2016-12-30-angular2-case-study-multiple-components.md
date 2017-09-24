---
author: wukn
catalog: true
date: 2016-12-30T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Case Study - Multiple Components and Services'
url: /2016/12/30/angular2-case-study-multiple-components/
---

> 使用TypeScript构建Angular2应用。多组件交互和服务。

<!--more-->

### 多组件

随着应用的增长，我们需要将一些功能独立成可复用的组件，这又牵连出组件之间数据交互的问题。让我们把heroes列表和hero详细视图分成两个组件。（单一职责原则）

在`app`文件夹下创建新的文件`hero-detail.component.ts`，在文件中定义`HeroDetailComponent`：

```js
// app/hero-detail.component.ts(partial)

import { Component, Input } from '@angular/core';

@Component({
  selector: 'my-hero-detail',
})
export class HeroDetailComponent {
}
```

注意看下命名规范。组件`AppComponent`对应的文件名为`app.component.ts`，`HeroDetailComponent`对应的文件名为`hero-detail.component.ts`。也就是说组件的名字以`Component`结尾，对应的文件名后缀为`.component.ts`。组件的名字中各单词的首字母大写；而文件名中全部小写，单词之间用`-`相连，称为lower dash-case。

目前，Heroes和HeroDetail的视图都在`AppComponent`的模板中。我们将HeroDetail的视图拆到新的组件`HeroDetailComponent`中。

在`AppComponent`我们绑定了`selectedHero.name`属性。但是在`HeroDetailComponent`我们定义的属性是`hero`，所以这里要替换下。

{% raw %}
```js
// app/hero-detail.component.ts(template)

template: `
  <div *ngIf="hero">
    <h2>{{hero.name}} details!</h2>
    <div><label>id: </label>{{hero.id}}</div>
    <div>
      <label>name: </label>
      <input [(ngModel)]="hero.name" placeholder="name"/>
    </div>
  </div>
  `
```
{% endraw %}

为`HeroDetailComponent`添加`hero`属性：

```js
hero: Hero;
```

噢，我们的`Hero`还定义在`app.component.ts`文件中。把它也放到单独的文件中吧：

```js
// app/hero.ts

export class Hero {
  id: number;
  name: string;
}
```

分别在`app.component.ts`和`hero-detail.component.ts`中添加import：

```js
import { Hero } from './hero';
```

`HeroDetailComponent`的`Hero`属性是一个输入属性，是`AppComponent`告诉它的。`AppComponent`知道用户选中的是列表中的哪个，在它的`selectedHero`属性中。所以接下来我们要在`AppComponent`的模板中将它的`selectedHero`属性绑定到`HeroDetailComponent`的`hero`属性。这个绑定可能看起来是这样的：

```html
<my-hero-detail [hero]="selectedHero"></my-hero-detail>
```

等号左侧的是属性绑定的目标。在Angular中，我们需要将目标属性定义成输入属性（input），否则，Angular就不会绑定并且报错。

将属性声明为input有多中方式。最常用的就是对属性使用`@Input`装饰器：

```js
@Input()
hero: Hero;
```

回到`AppModule`，我们需要告诉它去使用`HeroDetailComponent`。引入`HeroDetailComponent`：

```js
import { HeroDetailComponent } from './hero-detail.component';
```

将`HeroDetailComponent`添加到`NgModule`的`declarations`列表中。这个列表包含的是所有我们创建的属于这个应用的组件、管道和指令。

```js
@NgModule({
  imports: [
    BrowserModule,
    FormsModule
  ],
  declarations: [
    AppComponent,
    HeroDetailComponent
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

更新`AppComponent`的模板：

{% raw %}
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
{% endraw %}

现在运行起来效果和之前是一样的。

### 服务

目前`AppComponent`中直接定义了模拟数据用于展示。这里有两个问题：一这些数据的获取不是组件的事儿，二这些数据无法与其它组件和视图共享。

So, 接下来我们将数据处理重构到一个单独的服务中。

数据服务毫无疑问是异步的，所以我们将要完成的是一个基于Promise的数据服务。

在`app`文件夹下创建`hero.service.ts`：

```js
import { Injectable } from '@angular/core';

@Injectable()
export class HeroService {
}
```

这里的命名还是文件名为lower dash-case，以`.service.ts`结尾。例如`SpecialSuperHeroService`对应的文件名应为`special-super-hero.service.ts`。

注意这里的`@Injectable()`装饰器。TypeScript遇到`@Injectable()`装饰器就会带上该service的元数据，以及该service的依赖的元数据。

添加一个`getHeroes`空方法：

```js
@Injectable()
export class HeroService {
  getHeroes(): void {} // stub
}
```

我们将数据的获取交给service去做，这样service的消费者，也就是组件，就不需要知道怎样去获取数据了。

之前我们在`AppComponent`定义的模拟数据。他不应该在那里，也不应该在service中，我们也将它放到一个单独的文件中。

将`app.component.ts`中的`HEROES`数组剪切出来，放到`app`文件夹下新创建的`mock-heroes.ts`中：

```js
// app/mock-heroes.ts

import { Hero } from './hero';
export const HEROES: Hero[] = [
  {id: 11, name: 'Mr. Nice'},
  {id: 12, name: 'Narco'},
  {id: 13, name: 'Bombasto'},
  {id: 14, name: 'Celeritas'},
  {id: 15, name: 'Magneta'},
  {id: 16, name: 'RubberMan'},
  {id: 17, name: 'Dynama'},
  {id: 18, name: 'Dr IQ'},
  {id: 19, name: 'Magma'},
  {id: 20, name: 'Tornado'}
];
```

由于`app.component.ts`中没有`HEROES`了，我们暂时将`heroes`置为未初始化的状态：

```js
heroes: Hero[];
```

回到`HeroService`中，我们引入`HEROES`，然后在`getHeroes`中返回它：

```js
// app/hero.service.ts

import { Injectable } from '@angular/core';

import { Hero } from './hero';
import { HEROES } from './mock-heroes';

@Injectable()
export class HeroService {
  getHeroes(): Hero[] {
    return HEROES;
  }
}
```

现在就可以在`AppComponent`中使用`HeroService`了：

```js
import { HeroService } from './hero.service';
```

我们需要自己实例化`HeroService`吗？像这样：

```js
heroService = new HeroService(); // don't do this
```

依赖注入会帮我们完成这样的工作。我们需要在构造函数中定义一个似有属性，并在组件的`providers`元数据中添加依赖的这个类：

```js
constructor(private heroService: HeroService) { }
```

```js
providers: [HeroService]
```

在`AppComponent`中创建`getHeroes`方法，从service中获取数据：

```js
getHeroes(): void {
  this.heroes = this.heroService.getHeroes();
}
```

现在问题来了，我们应该在那里调用这个方法呢？构造函数中？

经验告诉我们，不要在构造函数中写复杂的业务逻辑，特别是那些调用service获取数据的。构造函数应该只是简单的处理一些属性的初始化工作。

不是在构造函数中，那是在哪里调用`getHeroes`呢？

我们可以实现Angular的ngOnInit钩子（Lifecycle Hook）。Angular提供了一些接口能让我们在组件的生命周期的几个关键的点插入业务逻辑。每个接口对应一个方法。只要实现了接口，Angular就会在相应的时间点调用它。

在`AppComponent`组件中实现`OnInit`接口，调用`getHeroes`方法：

```js
import { OnInit } from '@angular/core';

export class AppComponent implements OnInit {
  ngOnInit(): void {
    this.getHeroes();
  }
}
```

现在运行起来效果和之前还是一样的。

#### 异步Service和Promise

`HeroService`中立刻返回了模拟数据，它的`getHeroes`方法是同步的：

```js
this.heroes = this.heroService.getHeroes();
```

真实的情况是，我们从远程服务器获取数据。所以需要将`getHeroes`改成异步模式：

```js
getHeroes(): Promise<Hero[]> {
  return Promise.resolve(HEROES);
}
```

对应的`app.component.ts`中调用service的地方也要修改：

```js
// app/app.component.ts

getHeroes(): void {
  this.heroService.getHeroes().then(heroes => this.heroes = heroes);
}
```

程序运行起来效果还是和之前一样的。

附：

我们可以模拟远程请求，让数据获取的慢一些：

```js
getHeroesSlowly(): Promise<Hero[]> {
  return new Promise(resolve => {
    // Simulate server latency with 2 second delay
    setTimeout(() => resolve(this.getHeroes()), 2000);
  });
}
```

---

参考资料：

[Angular2 Documentation - Case Study - Multiple Components](https://angular.io/docs/ts/latest/tutorial/toh-pt3.html)

[Angular2 Documentation - Case Study - Services](https://angular.io/docs/ts/latest/tutorial/toh-pt4.html)
