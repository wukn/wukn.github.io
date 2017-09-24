---
author: wukn
catalog: true
date: 2016-12-24T01:00:00Z
tags:
- Angular2
title: '[译][Angular2] Dependency Injection'
url: /2016/12/24/angular2-dependency-injection/
---

> Angular的依赖注入系统会准时（just-in-time）创建和传递依赖的服务。

<!--more-->

依赖注入是一个很重要的设计模式。Angular提供了自己的依赖注入框架。构建Angular应用是离不开它的。

### 为什么要使用依赖注入

看下面的这段代码：

```js
export class Car {
  public engine: Engine;
  public tires: Tires;
  public description = 'No DI';
  constructor() {
    this.engine = new Engine();
    this.tires = new Tires();
  }
  // Method using the engine and tires
  drive() {
    return `${this.description} car with ` +
      `${this.engine.cylinders} cylinders and ${this.tires.make} tires.`;
  }
}
```

`Car`在它的构造函数里创建它所需的所有东西。这有什么问题呢？

问题就是，`Car`类是脆弱的，固定的，不易于测试。`Car`需要engine和tires，然后就在构造函数里从指定的类`Engine`和`Tires`实例化，创建它自己的对象。

如果`Engine`类修改了，它的构造函数需要传入参数了怎么办？这样，我们就必须修改`Car`类了，`this.engine = new Engine(someParameter)`。`Enging`类的改动会导致`Car`类需要改动，这使得`Car`类是脆弱的。

如果我们要将`Car`的tires换成另一个类型呢？上面的代码限定了我们只能使用`Tires`。这使得`Car`类固定的，不具有柔性。

如果我们要编写`Car`类的测试用例，在测试环境能够创建一个新的`Engine`吗？`Engine`所依赖的类呢？这些依赖它们所依赖的类呢？实例化`Engine`类会不会发起对服务器的同步调用？显然我们不希望在测试时发生这样的情况。

如果`Car`要在胎压不足时给出警告信号，我们如何测试呢？我们没法在测试时改变Car内部的依赖。如果我们没法控制一个类的依赖，就很难对它进行测试。

如何让`Car`类具有健壮性、柔性和易测试性呢？

答案就是使用依赖注入！

```js
export class Car {
  public description = 'DI';

  constructor(public engine: Engine, public tires: Tires) { }

  // Method using the engine and tires
  drive() {
    return `${this.description} car with ` +
      `${this.engine.cylinders} cylinders and ${this.tires.make} tires.`
  }
}
```

我们将依赖的定义移到了构造函数中。`Car`类不再自己创建engine和tires，而是直接使用它们。

> 这里使用了TypeScript的构造函数语法，同时声明参数和属性。

创建car时只要将engine和tires传给构造函数。

```js
// Simple car with 4 cylinders and Flintstone tires.
let car = new Car(new Engine(), new Tires());
```

这样，所依赖的engine和tires的定义就从`Car`类中解耦出来了。我们可以传入任何类型的engine和tires，只要它们符合engine和tires的接口。

如果扩展了`Engine`类，那么`Car`类本身没有任何问题，不需要修改。但是`Car`的使用者需要修改：

```js
class Engine2 {
  constructor(public cylinders: number) { }
}
// Super car with 12 cylinders and Flintstone tires.
let bigCylinders = 12;
let car = new Car(new Engine2(bigCylinders), new Tires());
```

这样`Car`类就很方便测试了，我们可以传入模拟数据来测试。

```js
class MockEngine extends Engine { cylinders = 8; }
class MockTires  extends Tires  { make = 'YokoGoodStone'; }

// Test car with 8 cylinders and YokoGoodStone tires.
let car = new Car(new MockEngine(), new MockTires());
```

依赖注入就是将类从外部接收依赖而不是自己创建他们。

Cool！但是`Car`类的使用者怎么办呢？它必须知道`Car`、`Engine`和`Tires`。`Car`类将问题抛给了它的使用者了。我们需要有人来负责组装这些部件。

例如，使用工厂类：

```js
import { Engine, Tires, Car } from './car';
// BAD pattern!
export class CarFactory {
  createCar() {
    let car = new Car(this.createEngine(), this.createTires());
    car.description = 'Factory';
    return car;
  }
  createEngine() {
    return new Engine();
  }
  createTires() {
    return new Tires();
  }
}
```

这个看起来还不错，但是随着应用的增长，工厂类变成了一个巨大的依赖网，维护起来就很麻烦了。

可不可以我们只需要列出我们想创建的东西而不用定义它应该注入哪些依赖呢？这就是依赖注入框架要解决的问题。假设框架中有个`injector`，我们将一些类注册到这个`injector`，它会算出如何创建这些类。当我们需要一个`Car`时，直接向`injector`要就行了。

```js
let car = injector.get(Car);
```

所有问题都解决了。完美！`Car`不需要知道如何创建`Engine`或`Tires`。使用者也不需要知道如何创建`Car`。这些工作全部交给依赖注入框架。

### Angluar的依赖注入

Angular提供了自己的依赖注入框架。这个框架也可以作为一个独立的模块用于其他应用和框架。

我们看看Angular在构造组件时依赖注入是怎么工作的。以[Tour of Heroes]()里的`HeroesComponent`的简化版为例：

```js
// app/heroes/hero.ts

export class Hero {
  id: number;
  name: string;
  isSecret = false;
}
```

```js
// app/heroes/heroes.component.ts

import { Component }          from '@angular/core';
@Component({
  selector: 'my-heroes',
  template: `
  <h2>Heroes</h2>
  <hero-list></hero-list>
  `
})
export class HeroesComponent { }
```

```js
// app/heroes/hero-list.component.ts

import { Component }   from '@angular/core';
import { HEROES }      from './mock-heroes';
@Component({
  selector: 'hero-list',
  template: `
  <div *ngFor="let hero of heroes">
    {{hero.id}} - {{hero.name}}
  </div>
  `
})
export class HeroListComponent {
  heroes = HEROES;
}
```

```js
// app/heroes/mock-heroes.ts

import { Hero } from './hero';
export var HEROES: Hero[] = [
  { id: 11, isSecret: false, name: 'Mr. Nice' },
  { id: 12, isSecret: false, name: 'Narco' },
  { id: 13, isSecret: false, name: 'Bombasto' },
  { id: 14, isSecret: false, name: 'Celeritas' },
  { id: 15, isSecret: false, name: 'Magneta' },
  { id: 16, isSecret: false, name: 'RubberMan' },
  { id: 17, isSecret: false, name: 'Dynama' },
  { id: 18, isSecret: true,  name: 'Dr IQ' },
  { id: 19, isSecret: true,  name: 'Magma' },
  { id: 20, isSecret: true,  name: 'Tornado' }
];

```

`HeroesComponent`是`HeroListComponent`的父组件。

现在`HeroListComponent`从`HEROES`获取数据。如果我们想进行测试，或者将数据获取方式改成从服务器获取，那么我们就要修改`heroes`的实现，而且可能还要修改用到`HEROES`模拟数据的地方。因此我们创建service来隐藏数据的获取方式。

```js
// app/heroes/hero.service.ts

import { Injectable } from '@angular/core';
import { HEROES }     from './mock-heroes';
@Injectable()
export class HeroService {
  getHeroes() { return HEROES;  }
}
```

`HeroService`提供了`getHeroes`方法来返回模拟数据，而它的使用者就不需要知道数据从哪来。

这里不是一个真正的Service类。真实情况应该是从远程服务器获取数据，这个API得是异步的，也许返回的是一个Promise。就是说使用该服务的组件也要修改代码。

一个service在Angular2中就只是一个普通的类，除非我们将它注册到Angular的injector中。

#### 配置injector

我们不需要自己创建injector，Angular在启动阶段会创建一个应用层的injector。

```js
// app.main.ts

platformBrowserDynamic().bootstrapModule(AppModule);
```

但是我们需要通过注册provider来配置injector。我们可以在一个`NgModule`或者组件中注册provider。

#### 在`NgModule`中注册provider

在AppModule中注册一个`UserService`和`APP_CONFIG`：

```js
// app/app.module.ts

@NgModule({
  imports: [
    BrowserModule
  ],
  declarations: [
    AppComponent,
    CarComponent,
    HeroesComponent,
    /* . . . */
  ],
  providers: [
    UserService,
    { provide: APP_CONFIG, useValue: HERO_DI_CONFIG }
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

#### 在组件中注册provider

这是改进后的`HeroesComponent`，添加了注册`HeroService`：

```js
// app/heroes/heroes.component.ts

import { Component }          from '@angular/core';

import { HeroService }        from './hero.service';

@Component({
  selector: 'my-heroes',
  providers: [HeroService],
  template: `
  <h2>Heroes</h2>
  <hero-list></hero-list>
  `
})
export class HeroesComponent { }
```

如何选择在`NgModule`还是组件中注册呢？

NgModule中的provider是注册到根jinjector中。这意味着NgModule中注册的provider将在整个应用中可访问。而在组件中注册的provider只是在该组件及其子组件中可用。

我们希望`APP_CONFIG`服务是在整个应用中可访问的，而`HeroService`仅仅在Heroes特性域中使用。

#### 修改HeroListComponent

将`HeroListComponent`改成从注入的`HeroService`获取数据：

```js
// app/heroes/herp-list.component.ts

import { Component }   from '@angular/core';
import { Hero }        from './hero';
import { HeroService } from './hero.service';
@Component({
  selector: 'hero-list',
  template: `
  <div *ngFor="let hero of heroes">
    {{hero.id}} - {{hero.name}}
  </div>
  `
})
export class HeroListComponent {
  heroes: Hero[];
  constructor(heroService: HeroService) {
    this.heroes = heroService.getHeroes();
  }
}
```

注意看修改后的构造函数：

```js
constructor(heroService: HeroService) {
  this.heroes = heroService.getHeroes();
}
```

这里不是仅仅为构造函数添加了一个参数。我们看到参数的类型是`HeroService`，并且`HeroListComponent`类有一个`@Component`装饰器。再回想一下父组件`HeroesComponent`的`providers`中的`HeroService`。所有这些信息在一起，告诉了Angular在创建新的`HeroListComponent`实例的时候注入一个`HeroService`实例。

#### 隐式依赖创建

在之前的代码里我们看到如何创建injector和创建实例：

```js
injector = ReflectiveInjector.resolveAndCreate([Car, Engine, Tires]);
let car = injector.get(Car);
```

实际上我们不需要这样做显式调用injector，Angular会为在创建组件的时候创建和调用injector。我们完全可以将这部分工作交给Angular来做，享受自动依赖注入的便捷。

#### 单例service

由injector提供的依赖都是单例的。在上面的例子中，`HeroesComponent`和`HeroListComponent`会共享同一个`HeroService`实例。

然而，Angular的依赖注入是分层的。也就是说嵌套的injector可以创建它们自己的service实例。

#### 测试组件

之前提到使用依赖注入设计类可以让类易于测试。例如，编写测试用例如下：

```js
let expectedHeroes = [{name: 'A'}, {name: 'B'}]
let mockService = <HeroService> {getHeroes: () => expectedHeroes }

it('should have heroes when HeroListComponent created', () => {
  let hlc = new HeroListComponent(mockService);
  expect(hlc.heroes.length).toEqual(expectedHeroes.length);
});
```

#### 当service依赖另外的service？

上面的`HeroService`很简单，没有依赖其它service。如果它也有依赖的service呢，比如说logging service？跟上面一样，添加依赖注入：

```js
// app/heroes/hero.service

import { Injectable } from '@angular/core';
import { HEROES }     from './mock-heroes';
import { Logger }     from '../logger.service';
@Injectable()
export class HeroService {
  constructor(private logger: Logger) {  }
  getHeroes() {
    this.logger.log('Getting heroes ...');
    return HEROES;
  }
}
```

### `@Injectable`

注意到上面的service类使用了`@Injectable()`装饰器。`@Injectable()`表示这个类可以用injector实例化。一般来说，injector在为没有标记`@Injectable()`的类实例化注入依赖的时候会报错。之前不用加是因为我们那时还没有注入参数。

建议始终为service添加`@Injectable()`，这样一来保证了所有service的一致性，二今后如果要对service进行升级时也不会忘记添加了。

`HeroesComponent`组件也有注入的依赖，为什么它不用添加`@Injectable()`呢？

尽管我们也可以给它添加上`@Injectable()`。实际上它标记为`@Component`，这个装饰器（还有`@Directive`和`@Pipe`）是`Injectable`的子类型。实际上是`Injectable`这个装饰器标识了一个类可以被injector实例化。

> 在运行时，injector会在编译成JavaScript的代码中读取类的元数据，并使用构造函数参数类型信息来决定注入什么。
> 不是所有的JavaScript类都有元数据。TypeScript编译器默认会丢掉元数据。如果`tsconfig.json`中的编译参数`emitDecoratorMetadata`设置为true，那么编译器会对所有的至少有一个装饰器的类添加元数据。
> 由于任意的装饰器都会触发这一效果，对于服务类使用`Injectable`就表明了我们的意图。

### 创建和注册logger服务

向`HeroService`注入Logger service分两步。

1. 创建`Logger` service；
2. 向应用注册它。

```js
// app/logger.service

import { Injectable } from '@angular/core';
@Injectable()
export class Logger {
  logs: string[] = []; // capture logs for testing
  log(message: string) {
    this.logs.push(message);
    console.log(message);
  }
}
```

我们希望在整个应用中使用同一个的logger service，所以我们将它放在项目的`app`文件夹下，并且注册到`AppModule`的provider数组中。

```js
// app/app.component.ts

providers: [Logger]
```

如果我们忘记注册它，那么Angular会报错：

```js
EXCEPTION: No provider for Logger! (HeroListComponent -> HeroService -> Logger)
```

### Injector provider

injector依赖provider来创建要注入到组件或其他服务的服务实例。

前面我们在`AppModule`的providers中注册了`Logger`：

```js
// app/app.module.ts

providers: [Logger]
```

有很多种方式来提供一个看起来和表现起来都像`Logger`的东西。`Logger`类本身当然是可以的。但它不是唯一的方式。

我们可以用可选的provider来配置injector，传递一个表现起来像`Logger`的对象。我们可以提供一个子类，可以提供一个很像logger的对象，可以提供一个调用logger工厂函数的provider。任意一种方式都有适用的场景。

重要的是当injector需要一个`Logger`时它有一个provider去访问。

#### `Provider`类和`provide`对象字面量

在前面`providers`数组是这样写的：

```js
providers: [Logger]
```

它实际上是一个简写方式，实际上是有两个属性的`provider`对象字面量：

```js
[{ provide: Logger, useClass: Logger }]
```

第一个属性是token，作为定位依赖值和注册provider的key。

第二个属性是provider定义对象，我们可以认为是创建依赖值的菜单。有很多种方式创建依赖值...同样有很多种方式类写这个菜单。

#### 替换类provider

偶尔我们会要求使用另一个类来提供服务。下面的代码告诉injector在别人需要`Logger`的时候返回一个`BetterLogger`：

```js
[{ provide: Logger, useClass: BetterLogger }]
```

#### 有依赖的类provider

也许`EvenBetterLogger`需要从注入的`UserService`中获取用户信息，正好我们在应用层注入了它。

```js
@Injectable()
class EvenBetterLogger extends Logger {
  constructor(private userService: UserService) { super(); }

  log(message: string) {
    let name = this.userService.user.name;
    super.log(`Message to ${name}: ${message}`);
  }
}
```

注册它：

```js
[ UserService,
  { provide: Logger, useClass: EvenBetterLogger }]
```





## Injector provider

injector依赖`providers`来创建实例。我们必须注册service的provider，injector才知道如何来创建实例。例如，我们在`AppComponent`的`providers`中注册了`Logger`：

```js
providers: [Logger]
```

`providers`数组里看起来是个service类。实际上是能提供那个service类的`Provider`类实例。有很多种方式来提供看起来和表现起来像`Logger`的一些东西。`Logger`类它自己自然是可以的。除此之外，我们还可以提供一个`Logger`相似的对象或者会调用`Logger`工厂函数的类。

### provide函数

之前我们是这样写`providers`数组的：`[Logger]`。实际上这是一个简写，它相当于`[new Provider(Logger, {useClass: Logger})]`，是个Provider类的实例。更友好的方式是使用provider函数：`[provide(Logger, {useClass: Logger})]`。

不管是`Provider`类还是`provide`函数，都需要提供两个参数：第一个参数是用来定位依赖值和注册provider的标识；第二个参数是一个provider定义对象。

例如，我们可以用另一个类来提供服务，当service需要`Logger`时，injector会提供`BetterLogger`：

```js
[provide(Logger, {useClass: BetterLogger})]
```

甚至是依赖其他service的`EvenBetterLogger`：

```js
@Injectable()
class EvenBetterLogger {
  logs:string[] = [];
  constructor(private _userService: UserService) { }
  log(message:string){
    message = `Message to ${this._userService.user.name}: ${message}.`;
    console.log(message);
    this.logs.push(message);
  }
}
```

在providers中配置：

```js
[ UserService,
  provide(Logger, {useClass: EvenBetterLogger}) ]
```

### 别名类provider

假设一个老组件依赖`OldLogger`。`OldLogger`有着跟`NewLogger`一样的接口。由于某些原因，我们没法更新`OldLogger`。现在我们希望，老的组件在在需要`OldLogger`时，用`NewLogger`实例来代替它。也就是希望injector在组件需要`NewLogger`或`OldLogger`时都返回`NewLogger`实例。相当于`OldLogger`是`NewLogger`的别名。

我们可能会想到这样做：

```js
[ NewLogger,
  // Not aliased! Creates two instances of `NewLogger`
  { provide: OldLogger, useClass: NewLogger}]
```

但是这样在应用中会有两个`NewLogger`实例。解决方案是使用`useExisting`选项：

```js
[ NewLogger,
  // Alias OldLogger w/ reference to NewLogger
  { provide: OldLogger, useExisting: NewLogger}]
```

####  Value provider

有时提供一个已经存在的对象比要求injector创建一个更简单：

```js
// An object in the shape of the logger service
let silentLogger = {
  logs: ['Silent logger says "Shhhhh!". Provided via "useValue"'],
  log: () => {}
};
```

那么我们可以使用`useValue`选项：

```js
[{ provide: Logger, useValue: silentLogger }]
```

#### Factory provider

有时候我们需要动态地创建依赖的值，基于一些直到最后才能知道的一些信息。还有注入的服务没有独立访问这些资源的权限等情况。

这样的情况就需要factory provider了。

假设有这样的业务需求：`HeroService`对普通用户隐藏secret属性值为true的heroes，只有授权的用户才能看到。

就像`EvenBetterLogger`，`HeroService`就需要知道user的信息。而且这个授权信息对每个应用会话是不一样的。跟`EvenBetterLogger`不同的是，我们不能将`UserService`注入`HeroService`。`HeroService`不会直接访问用户信息来决定谁有没有权限。

我们为`HeroService`的构造函数添加一个布尔标识来控制显示隐藏数据。

```js
// app/heroes/hero.service.ts

constructor(
  private logger: Logger,
  private isAuthorized: boolean) { }

getHeroes() {
  let auth = this.isAuthorized ? 'authorized ' : 'unauthorized';
  this.logger.log(`Getting heroes for ${auth} user.`);
  return HEROES.filter(hero => this.isAuthorized || !hero.isSecret);
}
```

我们可以注入`Logger`，但是却没法注入布尔值`isAuthorized`。我们将`HeroService`创建实例的工作交给一个factory provider。


```js
// app/heroes/hero.service.provider.ts

let heroServiceFactory = (logger: Logger, userService: UserService) => {
  return new HeroService(logger, userService.user.isAuthorized);
};
```

尽管`HeroService`不能访问`UserService`，但是这个工厂函数可以。

将`Logger`和`UserService`注入到factory provider：

```js
// app/heroes/hero.service.provider.ts

export let heroServiceProvider =
  { provide: HeroService,
    useFactory: heroServiceFactory,
    deps: [Logger, UserService]
  };
```

`useFactory`字段告诉Angular这个provider是一个工厂函数，它的实现是`heroServiceFactory`。

`deps`属性是一个provider token数组。`Logger`和`UserService`类将作为它们自己的类provider的token。injector会解析这些token并注入与工厂函数的参数匹配的服务。

注意，我们捕获了工厂provider作为一个export变量，`heroServiceProvider`。这样使得工厂provider可重用。我们可以在需要的时候使用这个变量来注册`HeroService`。

在例子中，我们只是在`HeroesComponent`需要它。将`HeroesComponent`中的`HeroService`替换为`heroServiceProvider`：

```js
// app/heroes/heroes.component

import { Component }          from '@angular/core';
import { heroServiceProvider } from './hero.service.provider';
@Component({
  selector: 'my-heroes',
  template: `
  <h2>Heroes</h2>
  <hero-list></hero-list>
  `,
  providers: [heroServiceProvider]
})
export class HeroesComponent { }
```

### 依赖注入token

我们向injector注册一个provider，关联了一个依赖注入token，injector内部维护了一个token-provider映射map，token就是这个map的key。

在之前的例子中，依赖的值是一个类的实例，这个类的类型就作为token了。就像这样：

```js
heroService: HeroService = this.injector.get(HeroService);
```

我们在构造函数中指定参数的类型后，Angular就知道该注入那个实例了：

```
constructor(heroService: HeroService)
```

### Non-class dependency

如果依赖的值不是类怎么办？如果我们想注入的是一个字符串、函数或者对象呢？

应用中通常会定义很多配置对象来提供一些配置参数。这些配置对象有时候不是类的实例。例如：

```js
// app/app-config.ts

export interface AppConfig {
  apiEndpoint: string;
  title: string;
}

export const HERO_DI_CONFIG: AppConfig = {
  apiEndpoint: 'api.heroes.com',
  title: 'Dependency Injection'
};
```

我们想把`config`对象注册到injector中可以使用value provider。但是如何使用它的token呢？并没有一个类作为它的token。也没有`AppConfig`类。

#### TypeScript接口不是有效的token

`HERO_DI_CONFIG`常量有一个接口`AppConfig`，但是它不能作为token：

```js
// FAIL!  Can't use interface as provider token
[{ provide: AppConfig, useValue: HERO_DI_CONFIG })]
```

```js
// FAIL! Can't inject using the interface as the parameter type
constructor(private config: AppConfig){ }
```

这样看起来很奇怪，因为在强类型语言中使用依赖注入，接口是最好的依赖查找key。

这不是Angular的问题。因为Javascript中没有接口，接口是TypeScript的设计时的东西。TypeScript接口在编译成JavaScritp之后就不见啦。

### OpaqueToken

针对non-class dependencies这种情况，一种方法是定义和使用OpaqueToken：

```js
import { OpaqueToken } from '@angular/core';

export let APP_CONFIG = new OpaqueToken('app.config');
```

注册和使用：

```js
providers: [{ provide: APP_CONFIG, useValue: HERO_DI_CONFIG }]
```

```js
constructor(@Inject(APP_CONFIG) config: AppConfig) {
  this.title = config.title;
}
```

尽管`AppConfig`接口在依赖注入中没什么用，但它能提供类型支持。

或者我们可以在一个ngModule提供和注入配置对象，例如，在`AppModule`中：

```js
// app/app.module.ts

providers: [
  UserService,
  { provide: APP_CONFIG, useValue: HERO_DI_CONFIG }
],
```

Angular内部就是使用OpaqueToken的，例如HTTP_PROVIDERS 。实际上Provider内部会把所有字符串和类类型都转成OpaqueToken。

### 直接使用injector（罕见的情况）

{% raw %}
```js
// app/injector.component.ts

@Component({
  selector: 'my-injectors',
  template: `
  <h2>Other Injections</h2>
  <div id="car">{{car.drive()}}</div>
  <div id="hero">{{hero.name}}</div>
  <div id="rodent">{{rodent}}</div>
  `,
  providers: [Car, Engine, Tires, heroServiceProvider, Logger]
})
export class InjectorComponent {
  car: Car = this.injector.get(Car);
  heroService: HeroService = this.injector.get(HeroService);
  hero: Hero = this.heroService.getHeroes()[0];
  constructor(private injector: Injector) { }
  get rodent() {
    let rousDontExist = `R.O.U.S.'s? I don't think they exist!`;
    return this.injector.get(ROUS, rousDontExist);
  }
}
```
{% endraw %}

`Injector`自身也是可注入的。

注意这些服务他们自己是没有注入到组件的，它们是通过调用`injector.get`来获取的。

这个`get`方法在它无法解析所需要的服务时会抛出错误。可以指定第二个参数（当服务不存在时返回的值），就像`rodent`方法里所写的那样。

> 这里所描述的技术称为service locator pattern。（不懂）
> We avoid this technique unless we genuinely need it. It encourages a careless grab-bag approach such as we see here. It's difficult to explain, understand, and test. We can't know by inspecting the constructor what this class requires or what it will do. It could acquire services from any ancestor component, not just its own. We're forced to spelunk the implementation to discover what it does.

### 可选的依赖

我们的`HeroService`需要`Logger`。怎样让我们没有`Logger`该service也能工作呢？如果有就使用，如果没有就忽略。

引入`@Optional()`装饰器，修改构造函数，告诉injector `_logger`是可选的：

```js
import { Optional } from '@angular/core';
```

```js
constructor(@Optional() private logger: Logger) {
  if (this.logger) {
    this.logger.log(some_message);
  }
}
```

使用`@Optional()`时，我们必须要在代码中处理值为null的情况。如果在上面的代码中没有注册logger，那么injector会将`logger`的值赋为null。

---

参考资料：

[Angular2 Documentation - Dependency Injection](https://angular.io/docs/ts/latest/guide/dependency-injection.html)
