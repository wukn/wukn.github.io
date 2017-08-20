---
layout:     post
title:      "[Angular2] Case Study"
subtitle:   ""
date:       2016-12-26 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 用一个案例学习如何构建Angular2应用。


Angular官方文档上的案例很有趣，构建一个应用来帮助人力资源机构维护hero的信息。即使是hero也需要找工作。

应用的界面包括一个Dashboard和hero列表，并能编辑hero详细信息。最终效果如下：

![](/img/post/angular2/case-study/toh.gif)

我们会学习到使用内置的指令来显示/隐藏元素和显示数据列表，创建一个显示hero详细信息的组件和一个显示hero数组列表的组件，为只读数据使用单向数据绑定，为可编辑的字段使用双向数据绑定，为用户事件绑定组件的方法，从列表试图中选择一项后跳转到该项的详细试图，使用pipe处理数据格式，创建服务，使用路由在不同的视图和组件之间导航。

### 对象

首先，创建一个项目文件夹`angular2-tour-of-heroes`，按照[QuickStart](https://wukn.github.io/2016/11/29/angluar2-quick-start/)创建项目。

修改`AppComponent`，添加两个属性：

```js
export class AppComponent {
  title = 'Tour of Heroes';
  hero = 'Windstorm';
}
```

并更新模板：

{% raw %}
```js
template: '<h1>{{title}}</h1><h2>{{hero}} details!</h2>'
```
{% endraw %}

现在效果如下：

![](/img/post/angular2/case-study/1.png)

这里用到了单向数据绑定的插值绑定方法。

接下来我们将hero抽象成一个类。创建一个`Hero`类，包含两个属性`id`和`name`。在`app.component.ts`文件的`import`语句之后加入下面的代码：

```js
export class Hero {
  id: number;
  name: string;
}
```

将组件的`hero`属性从字符串改成`Hero`类型：

```js
hero: Hero = {
  id: 1,
  name: 'Windstorm'
};
```

同时修改下模板中的绑定：

{% raw %}
```js
template: '<h1>{{title}}</h1><h2>{{hero.name}} details!</h2>'
```
{% endraw %}

现在`app.component.ts`代码是这样的：

{% raw %}
```js
import { Component } from '@angular/core';

export class Hero {
  id: number;
  name: string;
}

@Component({
  selector: 'my-app',
  template: '<h1>{{title}}</h1><h2>{{hero.name}} details!</h2>'
})
export class AppComponent {
  title = 'Tour of Heroes';
  hero: Hero = {
    id: 1,
    name: 'Windstorm'
  };
}
```
{% endraw %}

浏览器中显示的效果很上面还是一样的。

让我们来显示更多的数据。修改模板：

{% raw %}
```js
template: '<h1>{{title}}</h1><h2>{{hero.name}} details!</h2><div><label>id: </label>{{hero.id}}</div><div><label>name: </label>{{hero.name}}</div>'
```
{% endraw %}

显示效果如下：

![](/img/post/angular2/case-study/2.png)

为了让模板更具有可读性，我们将模板改成多行模式：

{% raw %}
```js
template:`
  <h1>{{title}}</h1>
  <h2>{{hero.name}} details!</h2>
  <div><label>id: </label>{{hero.id}}</div>
  <div><label>name: </label>{{hero.name}}</div>
  `
```
{% endraw %}

注意，模板的值用的是反引号。

接下来我们希望可以编辑hero的name。因此在模板中将name的绑定换成input：

{% raw %}
```js
template:`
  <h1>{{title}}</h1>
  <h2>{{hero.name}} details!</h2>
  <div><label>id: </label>{{hero.id}}</div>
  <div>
    <label>name: </label>
    <input value="{{hero.name}}" placeholder="name">
  </div>
  `
```
{% endraw %}

现在name就显示在文本框里了：

![](/img/post/angular2/case-study/3.png)

这时候我们发现，修改文本框里的值，name并不会改变。因为这里只是单向数据绑定。改成双向数据绑定：

在`app.module.ts`中引入`FormsModule`，并将`NgModule`装饰器添加到`imports`数组列表中。这样我们就能在应用中使用`ngModel`双向数据绑定了。

```js
import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule }   from '@angular/forms';
import { AppComponent }  from './app.component';
@NgModule({
  imports: [
    BrowserModule,
    FormsModule
  ],
  declarations: [
    AppComponent
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

将模板中的`input`改成双向数据绑定：

```js
<input [(ngModel)]="hero.name" placeholder="name">
```

现在我们修改文本框里的值，hero的name属性值也跟着变了：

![](/img/post/angular2/case-study/4.png)

完整的`app.component.ts`如下：

{% raw %}
```js
import { Component } from '@angular/core';
export class Hero {
  id: number;
  name: string;
}
@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <h2>{{hero.name}} details!</h2>
    <div><label>id: </label>{{hero.id}}</div>
    <div>
      <label>name: </label>
      <input [(ngModel)]="hero.name" placeholder="name">
    </div>
    `
})
export class AppComponent {
  title = 'Tour of Heroes';
  hero: Hero = {
    id: 1,
    name: 'Windstorm'
  };
}
```
{% endraw %}

### 列表

创建一个heroes数组，后面我们会改成从web服务中获取数据，这里就先用模拟数据，暂时还放在`app.component.ts`文件内：

```js
const HEROES: Hero[] = [
  { id: 11, name: 'Mr. Nice' },
  { id: 12, name: 'Narco' },
  { id: 13, name: 'Bombasto' },
  { id: 14, name: 'Celeritas' },
  { id: 15, name: 'Magneta' },
  { id: 16, name: 'RubberMan' },
  { id: 17, name: 'Dynama' },
  { id: 18, name: 'Dr IQ' },
  { id: 19, name: 'Magma' },
  { id: 20, name: 'Tornado' }
];
```

在`AppComponent`中添加一个属性`heroes`：

```js
heroes = HEROES;
```

这里我们没有定义`heroes`的类型，TypeScript会根据`HERROES`数组推断出来的。

在模板中使用`ngFor`指令来显示heroes数组：

```html
<h2>My Heroes</h2>
<ul class="heroes">
  <li *ngFor="let hero of heroes">
    <span class="badge">{{hero.id}}</span> {{hero.name}}
  </li>
</ul>
```

为列表添加个样式，让界面美观点。设置`@Component`的`styles`属性即可，后面我们再把样式移到单独的文件中：

```js
styles: [`
  .selected {
    background-color: #CFD8DC !important;
    color: white;
  }
  .heroes {
    margin: 0 0 2em 0;
    list-style-type: none;
    padding: 0;
    width: 15em;
  }
  .heroes li {
    cursor: pointer;
    position: relative;
    left: 0;
    background-color: #EEE;
    margin: .5em;
    padding: .3em 0;
    height: 1.6em;
    border-radius: 4px;
  }
  .heroes li.selected:hover {
    background-color: #BBD8DC !important;
    color: white;
  }
  .heroes li:hover {
    color: #607D8B;
    background-color: #DDD;
    left: .1em;
  }
  .heroes .text {
    position: relative;
    top: -3px;
  }
  .heroes .badge {
    display: inline-block;
    font-size: small;
    color: white;
    padding: 0.8em 0.7em 0 0.7em;
    background-color: #607D8B;
    line-height: 1em;
    position: relative;
    left: -1px;
    top: -4px;
    height: 1.8em;
    margin-right: .8em;
    border-radius: 4px 0 0 4px;
  }
`]
```

现在列表就出来了：

![](/img/post/angular2/case-study/5.png)

完整的`app.component.ts`如下：

{% raw %}
```js
import { Component } from '@angular/core';

export class Hero {
  id: number;
  name: string;
}

const HEROES: Hero[] = [
  { id: 11, name: 'Mr. Nice' },
  { id: 12, name: 'Narco' },
  { id: 13, name: 'Bombasto' },
  { id: 14, name: 'Celeritas' },
  { id: 15, name: 'Magneta' },
  { id: 16, name: 'RubberMan' },
  { id: 17, name: 'Dynama' },
  { id: 18, name: 'Dr IQ' },
  { id: 19, name: 'Magma' },
  { id: 20, name: 'Tornado' }
];

@Component({
  selector: 'my-app',
  template:`
    <h1>{{title}}</h1>
    <h2>{{hero.name}} details!</h2>
    <div><label>id: </label>{{hero.id}}</div>
    <div>
      <label>name: </label>
      <input [(ngModel)]="hero.name" placeholder="name">
    </div>
    <h2>My Heroes</h2>
    <ul class="heroes">
      <li *ngFor="let hero of heroes">
        <span class="badge">{{hero.id}}</span> {{hero.name}}
      </li>
    </ul>
    `,
  styles: [`
    .selected {
      background-color: #CFD8DC !important;
      color: white;
    }
    .heroes {
      margin: 0 0 2em 0;
      list-style-type: none;
      padding: 0;
      width: 15em;
    }
    .heroes li {
      cursor: pointer;
      position: relative;
      left: 0;
      background-color: #EEE;
      margin: .5em;
      padding: .3em 0;
      height: 1.6em;
      border-radius: 4px;
    }
    .heroes li.selected:hover {
      background-color: #BBD8DC !important;
      color: white;
    }
    .heroes li:hover {
      color: #607D8B;
      background-color: #DDD;
      left: .1em;
    }
    .heroes .text {
      position: relative;
      top: -3px;
    }
    .heroes .badge {
      display: inline-block;
      font-size: small;
      color: white;
      padding: 0.8em 0.7em 0 0.7em;
      background-color: #607D8B;
      line-height: 1em;
      position: relative;
      left: -1px;
      top: -4px;
      height: 1.8em;
      margin-right: .8em;
      border-radius: 4px 0 0 4px;
    }
  `]
})
export class AppComponent {
  title = 'Tour of Heroes';
  hero: Hero = {
    id: 1,
    name: 'Windstorm'
  };
  heroes = HEROES;
}
```
{% endraw %}

现在列表和上面的详细信息视图是没有关系的。我们希望点击列表里的某一个hero时将他显示到详细信息视图中。这就需要绑定click事件，绑定到组件的`onSelect`方法。

{% raw %}
```js
<li *ngFor="let hero of heroes" (click)="onSelect(hero)">
  <span class="badge">{{hero.id}}</span> {{hero.name}}
</li>
```
{% endraw  %}

接下来创建组件的`onSelect`方法。在这之前我们要先为组件加一个属性`selectedHero`：

```js
selectedHero: Hero;
```

在`onSelect`方法中将当前点击的hero赋给`selectedHero`：

```js
onSelect(hero: Hero): void {
  this.selectedHero = hero;
}
```

同时更新详细信息视图模板里的input绑定：

{% raw %}
```html
<h2>{{selectedHero.name}} details!</h2>
<div><label>id: </label>{{selectedHero.id}}</div>
<div>
    <label>name: </label>
    <input [(ngModel)]="selectedHero.name" placeholder="name"/>
</div>
```
{% endraw %}

如果我们现在页面会报错，因为`selectedHero`没有初始值，也就是null，而页面在初始加载时的绑定会读取null的name属性，从而就报错了。我们修改为，如果`selectedHero`不为null才显示详细信息视图。

{% raw %}
```html
<div *ngIf="selectedHero">
  <h2>{{selectedHero.name}} details!</h2>
  <div><label>id: </label>{{selectedHero.id}}</div>
  <div>
    <label>name: </label>
    <input [(ngModel)]="selectedHero.name" placeholder="name"/>
  </div>
</div>
```
{% endraw %}

现在页面就正常啦。初始加载时只显示列表，点击一个hero之后才会显示详细信息视图。

现在点击一个hero之后我们很难看出来选中的是哪个，所以为点击选中的hero添加个样式。



![](/img/post/angular2/case-study/6.png)

完整的`app.component.ts`如下：

{% raw %}
```js
import { Component } from '@angular/core';

export class Hero {
  id: number;
  name: string;
}

const HEROES: Hero[] = [
  { id: 11, name: 'Mr. Nice' },
  { id: 12, name: 'Narco' },
  { id: 13, name: 'Bombasto' },
  { id: 14, name: 'Celeritas' },
  { id: 15, name: 'Magneta' },
  { id: 16, name: 'RubberMan' },
  { id: 17, name: 'Dynama' },
  { id: 18, name: 'Dr IQ' },
  { id: 19, name: 'Magma' },
  { id: 20, name: 'Tornado' }
];

@Component({
  selector: 'my-app',
  template:`
    <h1>{{title}}</h1>
    <div *ngIf="selectedHero">
      <h2>{{selectedHero.name}} details!</h2>
      <div><label>id: </label>{{selectedHero.id}}</div>
      <div>
        <label>name: </label>
        <input [(ngModel)]="selectedHero.name" placeholder="name"/>
      </div>
    </div>
    <h2>My Heroes</h2>
    <ul class="heroes">
      <li *ngFor="let hero of heroes"
          [class.selected]="hero === selectedHero"
          (click)="onSelect(hero)">
        <span class="badge">{{hero.id}}</span> {{hero.name}}
      </li>
    </ul>
    `,
  styles: [`
    .selected {
      background-color: #CFD8DC !important;
      color: white;
    }
    .heroes {
      margin: 0 0 2em 0;
      list-style-type: none;
      padding: 0;
      width: 15em;
    }
    .heroes li {
      cursor: pointer;
      position: relative;
      left: 0;
      background-color: #EEE;
      margin: .5em;
      padding: .3em 0;
      height: 1.6em;
      border-radius: 4px;
    }
    .heroes li.selected:hover {
      background-color: #BBD8DC !important;
      color: white;
    }
    .heroes li:hover {
      color: #607D8B;
      background-color: #DDD;
      left: .1em;
    }
    .heroes .text {
      position: relative;
      top: -3px;
    }
    .heroes .badge {
      display: inline-block;
      font-size: small;
      color: white;
      padding: 0.8em 0.7em 0 0.7em;
      background-color: #607D8B;
      line-height: 1em;
      position: relative;
      left: -1px;
      top: -4px;
      height: 1.8em;
      margin-right: .8em;
      border-radius: 4px 0 0 4px;
    }
  `]
})
export class AppComponent {
  title = 'Tour of Heroes';
  hero: Hero = {
    id: 1,
    name: 'Windstorm'
  };
  heroes = HEROES;

  selectedHero: Hero;

  onSelect(hero: Hero): void {
    this.selectedHero = hero;
  }
}
```

---

参考资料：

[Angular2 Documentation - Case Study - Introduction](https://angular.io/docs/ts/latest/tutorial/)

[Angular2 Documentation - Case Study - Hero Editor](https://angular.io/docs/ts/latest/tutorial/toh-pt1.html)

[Angular2 Documentation - Case Study - Master/Detail](https://angular.io/docs/ts/latest/tutorial/toh-pt2.html)
