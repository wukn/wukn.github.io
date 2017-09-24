---
author: wukn
catalog: true
date: 2017-03-06T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Hierarchical Injectors'
url: /2017/03/06/angular2-hierarchical-injectors/
---

> Angular的层级依赖注入系统支持与组件树并行的嵌套注入。

<!--more-->

Angular提供了一个层级依赖注入系统。实际上注入器也是树结构，跟组件树并行的。

### 注入树

在[依赖注入](https://wukn.github.io/2016/12/24/angular2-dependency-injection/)已经学习了如何配置依赖注入。

事实上并没有一个injector这样的东西存在。一个应用可能有多个注入器。Angular应用是一个组件树。每个组件实例都有它自己的注入器。组件树与注入器树是并行的。

组件的注入器可以代理更高层的注入器。这是为了提高效率。我们不需要关注具体实现的差异，只要记住每个组件有它自己的注入器就行了。

在之前的案例中，顶部的`AppComponent`拥有几个子组件，其中一个是`HeroesListComponent`。`HeroesListComponent`又包含并管理多个`HeroTaxReturnComponent`的实例。当同时有三个实例时，组件树就像下图所示：

![](/img/post/angular2/hierarchical-injectors/component-hierarchy.png)

#### 注入器冒泡

当一个组件请求一个依赖时，angular会尝试用组件自身的注入器注册的提供者来满足依赖。如果组件的注入器缺少这个提供者，那么就把这个请求向上传递给父组件的注入器。如果父组件的注入器仍不能解缺，那么再向上层传递。请求会一直向上冒泡，知道Angular找到可满足依赖的注入器，或者超出根注入器。如果一直到根都没找到，Angular会抛出错误。

冒泡是可以被取消的。一个中间组件可以被声明为`host`组件。那么请求冒泡到它之后将不会再往上层冒泡。

#### 在不同的层级重复提供一个服务

可以在注入器树的多个层级重复注册一个特定依赖的提供者。其实不需要重复注册，但是真想这样做也是可以的。

根据上面描述的解决依赖的逻辑过程，第一个遇到的注入器将完成依赖注入，并中断底层的请求。

我们可以只在顶层（通常是`AppModule`）指定提供者，但这样依赖树将比较扁平，所有的请求都会冒泡到在`bootstrapModule`方法中配置的根`NgModule`注入器中。

### 组件注入器

在不同的层级配置不同的提供者是有很多作用的。

#### 服务隔离

架构因素可能要求我们需要将服务限制在应用的某些领域中。

假设`VillainsListComponent`用来显示一个坏蛋列表。它从`VillainsService`获取数据。

我们可以在根`AppModule`中提供`VillainsService`，但是会让它在应用的任何地方都可以调用。例如Hero模块。

因此，我们应该把`VillainsService`配置在`VillainsListComponent`的`providers`元数据中：

```js
@Component({
  moduleId: module.id,
  selector: 'villains-list',
  templateUrl: './villains-list.component.html',
  providers: [ VillainsService ]
})
```

这样`VillainsService`只在`VillainsListComponent`和它的子组件中可访问。它仍然是一个单例，只不过它是villain领域的一个单例。

#### 多个编辑会话

很多应用允许用户同时打开多个处理多个任务。

假设`HeroListComponent`显示一个hero列表。现在要计算每个hero的税收。工作人员点击一个hero的名字，打开一个编辑计税的组件。每个选中的hero都会打开一个它自己的计税组件，这样就有多个计税组件同时运行。每个计税组件有它自己的编辑会话，修改它们将不会相互影响。

![](/img/post/angular2/hierarchical-injectors/hid-heroes-anim.gif)

`HeroTaxReturnService`如下，它调用一个`HeroTaxReturn`：

```js
// src/app/hero-tax-return.service.ts

import { Injectable }    from '@angular/core';
import { HeroTaxReturn } from './hero';
import { HeroesService } from './heroes.service';
@Injectable()
export class HeroTaxReturnService {
  private currentTaxReturn: HeroTaxReturn;
  private originalTaxReturn: HeroTaxReturn;
  constructor(private heroService: HeroesService) { }
  set taxReturn (htr: HeroTaxReturn) {
    this.originalTaxReturn = htr;
    this.currentTaxReturn  = htr.clone();
  }
  get taxReturn (): HeroTaxReturn {
    return this.currentTaxReturn;
  }
  restoreTaxReturn() {
    this.taxReturn = this.originalTaxReturn;
  }
  saveTaxReturn() {
    this.taxReturn = this.currentTaxReturn;
    this.heroService.saveTaxReturn(this.currentTaxReturn).subscribe();
  }
}
```

`HeroTaxReturnComponent`如下：

```js
// src/app/hero-tax-return.component.ts

import { Component, EventEmitter, Input, Output } from '@angular/core';
import { HeroTaxReturn }        from './hero';
import { HeroTaxReturnService } from './hero-tax-return.service';
@Component({
  moduleId: module.id,
  selector: 'hero-tax-return',
  templateUrl: './hero-tax-return.component.html',
  styleUrls: [ './hero-tax-return.component.css' ],
  providers: [ HeroTaxReturnService ]
})
export class HeroTaxReturnComponent {
  message = '';
  @Output() close = new EventEmitter<void>();
  @Input()
  get taxReturn(): HeroTaxReturn {
    return this.heroTaxReturnService.taxReturn;
  }
  set taxReturn (htr: HeroTaxReturn) {
    this.heroTaxReturnService.taxReturn = htr;
  }
  constructor(private heroTaxReturnService: HeroTaxReturnService ) { }
  onCanceled()  {
    this.flashMessage('Canceled');
    this.heroTaxReturnService.restoreTaxReturn();
  };
  onClose()  { this.close.emit(); };
  onSaved() {
    this.flashMessage('Saved');
    this.heroTaxReturnService.saveTaxReturn();
  }
  flashMessage(msg: string) {
    this.message = msg;
    setTimeout(() => this.message = '', 500);
  }
}
```

如果`HeroTaxReturnService`这个服务是应用级别的单例，那就有大问题了。每个组件实例共享同一个服务实例。每个组件将会覆盖其它组件的数据。

所以代码中是将它放在`HeroTaxReturnComponent`的提供者里。由于每个组件实例有它自己的注入器，每个组件将有一个它自己的似有的服务实例。从而避免了冲突。

#### 特殊提供者

另一种场景需要重注册服务的场景是，在组件树的低层提供一个服务的特定实现。

还是以Car为例，我们在根注入器中指定了泛化的`CarService`，`EngineService`和`TiresService`。

假设我们创建一个名为A的car组件来显示由上面三个泛化的服务组成的car。接着，创建一个子组件B时，使用的是`CarService`和`EngineService`的一个具体实现。组件B又有个子组件C，C又使用了一个它所特有的CarService实现。

![](/img/post/angular2/hierarchical-injectors/car-components.png)

这时就需要为每个组件的注入器注册不同的提供者：

![](/img/post/angular2/hierarchical-injectors/injector-tree.png)

---

参考资料：

[Angular2 Documentation - Hierarchical Injectors](https://angular.io/docs/ts/latest/guide/hierarchical-dependency-injection.html)
