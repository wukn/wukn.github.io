---
layout:     post
title:      "[Angular2] Case Study - Multiple Components and Services"
subtitle:   ""
date:       2016-12-30 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 使用TypeScript构建Angular2应用。多组件交互和服务。

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

注意看下命名规范。组件`AppComponent`对应的文件名为`app.component.ts`，`HeroDetailComponent`对应的文件名为`hero-detail.component.ts`。也就是说组件的名字以`Component`结尾，对应的文件名后缀为`.component.ts`。组件的名字中各单词的首字母大写；而文件名中全部小写，单词之间用`-`相连。

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



---

参考资料：

[Angular2 Documentation - Case Study - Multiple Components](https://angular.io/docs/ts/latest/tutorial/toh-pt3.html)

[Angular2 Documentation - Case Study - Services](https://angular.io/docs/ts/latest/tutorial/toh-pt4.html)
