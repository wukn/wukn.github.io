---
layout:     post
title:      "[Angular2] Case Study -HTTP"
subtitle:   ""
date:       2017-01-05 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---
{% raw %}

> 使用TypeScript构建Angular2应用。将服务改成HTTP服务。

现在我们要将数据的获取改成从服务器查询，并且添加、修改和删除操作能回写到服务器中。通过HTTP调用远程服务器的web API。

### 提供HTTP服务

`HttpModule`模块不是Angular的核心模块。它在可选的`@angular/http`包中，是Angular的npm包中的一个单独的文件。

幸运的是，我们可以直接从`@angular/http`中引入，因为`systemjs.config`配置了SystemJS去加载我们所用到的库。

#### 注册HTTP服务

我们的应用将依赖http服务，而它又依赖一些其它的支撑服务。`HttpModule`包含了所有http服务相关的provider。

在`HttpModule`中引入`HttpModule`，这样整个应用就能使用这些服务了。

```js
// app/app.module.ts

import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule }   from '@angular/forms';
import { HttpModule }    from '@angular/http';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent }         from './app.component';
import { DashboardComponent }   from './dashboard.component';
import { HeroesComponent }      from './heroes.component';
import { HeroDetailComponent }  from './hero-detail.component';
import { HeroService }          from './hero.service';
@NgModule({
  imports: [
    BrowserModule,
    FormsModule,
    HttpModule,
    AppRoutingModule
  ],
  declarations: [
    AppComponent,
    DashboardComponent,
    HeroDetailComponent,
    HeroesComponent,
  ],
  providers: [ HeroService ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

### 模拟Web API

我们的应用离真实的场景还远着呢。我们都还没有可以处理请求的Web服务器呢。先模拟一个吧。

我们这里从一个模拟的service中数据的获取和保存，一个内存中的Web API。

```js
// app/app.module.ts

import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule }   from '@angular/forms';
import { HttpModule }    from '@angular/http';

import { AppRoutingModule } from './app-routing.module';

// Imports for loading & configuring the in-memory web api
import { InMemoryWebApiModule } from 'angular-in-memory-web-api';
import { InMemoryDataService }  from './in-memory-data.service';

import { AppComponent }         from './app.component';
import { DashboardComponent }   from './dashboard.component';
import { HeroesComponent }      from './heroes.component';
import { HeroDetailComponent }  from './hero-detail.component';
import { HeroService }          from './hero.service';

@NgModule({
  imports: [
    BrowserModule,
    FormsModule,
    HttpModule,
    InMemoryWebApiModule.forRoot(InMemoryDataService),
    AppRoutingModule
  ],
  declarations: [
    AppComponent,
    DashboardComponent,
    HeroDetailComponent,
    HeroesComponent,
  ],
  providers: [ HeroService ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

这里是用`InMemoryWebApiModule`模拟服务器API的。`InMemoryWebApiModule.forRoot(InMemoryDataService)`中的参数`InMemoryDataService`是一个提供内存数据库初始值的类：

```js
// app/in-memory-data.service.ts

import { InMemoryDbService } from 'angular-in-memory-web-api';
export class InMemoryDataService implements InMemoryDbService {
  createDb() {
    let heroes = [
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
    return {heroes};
  }
}
```

现在可以删掉`mock-heroes.ts`了。

### HTTP

看一下目前`HeroService`中的实现：

```js
getHeroes(): Promise<Hero[]> {
  return Promise.resolve(HEROES);
}
```

我们用模拟数据返回了一个resloved Promise。那么现在要改成从HTTP客户端异步获取数据：

```js
// app/hero.service.ts (updated getHeroes and new class members)

private heroesUrl = 'api/heroes';  // URL to web api

constructor(private http: Http) { }

getHeroes(): Promise<Hero[]> {
  return this.http.get(this.heroesUrl)
             .toPromise()
             .then(response => response.json().data as Hero[])
             .catch(this.handleError);
}

private handleError(error: any): Promise<any> {
  console.error('An error occurred', error); // for demo purposes only
  return Promise.reject(error.message || error);
}
```

同时还要更新下import：

```js
// app/hero.service.ts (updated imports)

import { Injectable }    from '@angular/core';
import { Headers, Http } from '@angular/http';

import 'rxjs/add/operator/toPromise';

import { Hero } from './hero';
```

刷新浏览器，数据可以成功地从模拟的server中获取。

#### HTTP Promise

我们返回的仍然是一个Promise，但是创建的方式跟之前有些不同。

Angular的`http.get`方法返回的是一个RxJS的`Observable`。Observables是一种管理异步数据流的方式。这里我们使用了`.toPromise()`方法将`Observable`转换成`Promise`。但是Angular自己的`Observable`没有提供`toPromise`操作符，所以我们使用了RxJS库。

#### 在then的回调中抽取数据

在promise的`then`回调中我们用HTTP `Response`的`json`方法来获取数据。

```js
.then(response => response.json().data as Hero[])
```

响应结果JSON中有个data属性，这个属性的值就是Web API返回的值。在这里正好是我们想要的数组，所以直接将它作为resolved Promise值返回。

#### 错误处理

在`getHeroes()`方法的结尾我们`catch`了服务器的错误并提供了一个错误处理函数：

```js
.catch(this.handleError);
```

这一步很重要。我们必须假定HTTP调用失败是很常见的。

```js
private handleError(error: any): Promise<any> {
  console.error('An error occurred', error); // for demo purposes only
  return Promise.reject(error.message || error);
}
```

这里我们只是记录了错误到console中，同时返回一个rejected promise，这样调用者可以将错误显示给用户。

#### getHero by id

目前`HeroService`中的`getHero`是把所有数据获取过来之后再做查询的。实际情况中一般是不会这样做的。一般Web API都会支持get-by-id方式的请求，`api/hero/:id`（例如`api/hero/11`）：

```js
getHero(id: number): Promise<Hero> {
  const url = `${this.heroesUrl}/${id}`;
  return this.http.get(url)
    .toPromise()
    .then(response => response.json().data as Hero)
    .catch(this.handleError);
}
```

### 保存更新后的数据

现在我们可以在Hero的详细信息视图中更新数据，但是这只是在当前视图中，跳转到其它页面后更新就丢失了，因为我们的修改没有写回服务器中。

在模板中添加一个保存按钮：

```html
<!-- app/hero-detail.component.html (save) -->

<button (click)="save()">Save</button>
```

在`save`方法中调用service的`update`方法：

```js
// app/hero-detail.component.ts (save)

save(): void {
  this.heroService.update(this.hero)
    .then(() => this.goBack());
}
```

service中的`update`方法如下：

```js
// app/hero.service.ts (update)

private headers = new Headers({'Content-Type': 'application/json'});

update(hero: Hero): Promise<Hero> {
  const url = `${this.heroesUrl}/${hero.id}`;
  return this.http
    .put(url, JSON.stringify(hero), {headers: this.headers})
    .toPromise()
    .then(() => hero)
    .catch(this.handleError);
}
```

通过URL中的id指定我们要修改的hero，把修改后的hero转换成JSON字符串放在POST请求的body中，并在请求的Herder中制定了内容的类型。

刷新浏览器试一试，现在修改数据后可以保存了。

### 添加Hero

添加一个Hero只需要知道它的name就行了，所以在`heroes.component.html`中加一个input元素和一个按钮：

```html
<!-- app/heroes.component.html (add) -->

<div>
  <label>Hero name:</label> <input #heroName />
  <button (click)="add(heroName.value); heroName.value=''">
    Add
  </button>
</div>
```

点击添加按钮后调用组件的`add`方法，保存数据并晴空input文本框：

```js
// app/heroes.component.ts (add)

add(name: string): void {
  name = name.trim();
  if (!name) { return; }
  this.heroService.create(name)
    .then(hero => {
      this.heroes.push(hero);
      this.selectedHero = null;
    });
}
```

`HeroService`中的`create`方法：

```js
// app/hero.service.ts (create)

create(name: string): Promise<Hero> {
  return this.http
    .post(this.heroesUrl, JSON.stringify({name: name}), {headers: this.headers})
    .toPromise()
    .then(res => res.json().data)
    .catch(this.handleError);
}
```

刷新浏览器，试试效果。

### 删除Hero

在heroes列表视图中添加删除功能。

在列表的`<li>`标签内的最后加一个删除按钮：

```js
// app/heroes.component.html (li-element)

<li *ngFor="let hero of heroes" (click)="onSelect(hero)"
    [class.selected]="hero === selectedHero">
  <span class="badge">{{hero.id}}</span>
  <span>{{hero.name}}</span>
  <button class="delete"
    (click)="delete(hero); $event.stopPropagation()">x</button>
</li>
```

删除按钮点击后，除了调用组件的`delete`方法，还要阻止事件往上层冒泡。我们不希望在点击删除按钮后还会出发`<li>`的点击事件。

组件的`delete`方法：

```js
// app/heroes.component.ts (delete)

delete(hero: Hero): void {
  this.heroService
      .delete(hero.id)
      .then(() => {
        this.heroes = this.heroes.filter(h => h !== hero);
        if (this.selectedHero === hero) { this.selectedHero = null; }
      });
}
```

这里调用service的`delete`方法，但是界面上的更新还是要组件自己负责，从数组中移除已删除的，并且根据需要重置`selectedHero`。

给删除按钮加个样式，让它显示在最后面：

```css
/* app/heroes.component.css (additions) */

button.delete {
  float:right;
  margin-top: 2px;
  margin-right: .8em;
  background-color: gray !important;
  color:white;
}
```

`heroService`中的`delete`方法：

```js
// app/hero.service.ts (delete)

delete(id: number): Promise<void> {
  const url = `${this.heroesUrl}/${id}`;
  return this.http.delete(url, {headers: this.headers})
    .toPromise()
    .then(() => null)
    .catch(this.handleError);
}
```

刷新浏览器，看一下效果。

### Observables

`Http`服务的方法返回的是一个HTTP `Response`对象的`Observable`。

`HeroService`中将`Observable`转换成一个`Promise`返回给调用者。现在我们看下如何直接返回`Observable`。

#### 背景知识

observable是一个我们可以使用类似数组的操作符来处理的事件流。

Angular核心库提供了observable的基本支持。推荐使用[RxJS Observable](http://reactivex.io/rxjs/)库来支持一些操作符和扩展。

还记得上面的代码中，我们将`http.get`返回的`Observable`通过链式调用`toPromise`转换成`Promise`返回给用户。

通常这样做是个好主意。因为通常我们调用`http.get`就是为了获取数据，当我们获取到数据就结束了。将结果转换成promise对消费者来说处理起来比较简单。

但是有的请求不是一次就完成的。可能有这样的场景：发起请求——取消它——在服务器响应第一个请求之前发起另一个请求。这种request-cancel-new-request就不好用Promise实现。我慢将会看到Observables很容易实现。

#### 根据name搜索

接下来为应用添加一个搜索功能。

先创建一个`HeroSearchService`向Web API发送查询：

```js
// app/hero-search.service.ts

import { Injectable }     from '@angular/core';
import { Http, Response } from '@angular/http';
import { Observable } from 'rxjs';
import { Hero }           from './hero';
@Injectable()
export class HeroSearchService {
  constructor(private http: Http) {}
  search(term: string): Observable<Hero[]> {
    return this.http
               .get(`app/heroes/?name=${term}`)
               .map((r: Response) => r.json().data as Hero[]);
  }
}
```

这里调用`http.get()`跟之前是类似的，只是不再调用`toPromise`了。

创建`HeroSearchComponent`组件来调用`HeroSearchService`。

`HeroSearchComponent`的模板和样式：

```html
<!-- app/hero-search.component.html -->

<div id="search-component">
  <h4>Hero Search</h4>
  <input #searchBox id="search-box" (keyup)="search(searchBox.value)" />
  <div>
    <div *ngFor="let hero of heroes | async"
         (click)="gotoDetail(hero)" class="search-result" >
      {{hero.name}}
    </div>
  </div>
</div>
```

```css
/* app/hero-search.component.css */

.search-result{
  border-bottom: 1px solid gray;
  border-left: 1px solid gray;
  border-right: 1px solid gray;
  width:195px;
  height: 20px;
  padding: 5px;
  background-color: white;
  cursor: pointer;
}
#search-box{
  width: 200px;
  height: 20px;
}
```

用户在搜索框中输入关键字，当按键释放时，调用组件的`search`方法：

由于`heroes`属性现在是`Observable`数组，所以要将它经过`async`管道（AsyncPipe）进行处理。`async`管道会订阅`Observable`并产生heroes数组供`*ngFor`使用。

`HeroSearchComponent`组件类和元数据：

```js
// app/hero-search.component.ts

import { Component, OnInit } from '@angular/core';
import { Router }            from '@angular/router';
import { Observable }        from 'rxjs/Observable';
import { Subject }           from 'rxjs/Subject';
import { HeroSearchService } from './hero-search.service';
import { Hero } from './hero';
@Component({
  moduleId: module.id,
  selector: 'hero-search',
  templateUrl: 'hero-search.component.html',
  styleUrls: [ 'hero-search.component.css' ],
  providers: [HeroSearchService]
})
export class HeroSearchComponent implements OnInit {
  heroes: Observable<Hero[]>;
  private searchTerms = new Subject<string>();
  constructor(
    private heroSearchService: HeroSearchService,
    private router: Router) {}
  // Push a search term into the observable stream.
  search(term: string): void {
    this.searchTerms.next(term);
  }
  ngOnInit(): void {
    this.heroes = this.searchTerms
      .debounceTime(300)        // wait for 300ms pause in events
      .distinctUntilChanged()   // ignore if next search term is same as previous
      .switchMap(term => term   // switch to new observable each time
        // return the http search observable
        ? this.heroSearchService.search(term)
        // or the observable of empty heroes if no search term
        : Observable.of<Hero[]>([]))
      .catch(error => {
        // TODO: real error handling
        console.log(error);
        return Observable.of<Hero[]>([]);
      });
  }
  gotoDetail(hero: Hero): void {
    let link = ['/detail', hero.id];
    this.router.navigate(link);
  }
}
```

重点看一下`searchTerms`：

```js
private searchTerms = new Subject<string>();

// Push a search term into the observable stream.
search(term: string): void {
  this.searchTerms.next(term);
}
```

`Subject`是一个observable事件流的生产者。`searchTerms`产生一个字符串类型的`Observable`。

每次调用`search`时，通过`next`方法将新的字符串放到subject的observable流中。

`Subject`也是一个`Observable`。我们想把搜索关键词流转换成Hero数组流，并将结果赋给`heroes`属性：

```js
heroes: Observable<Hero[]>;

ngOnInit(): void {
  this.heroes = this.searchTerms
    .debounceTime(300)        // wait for 300ms pause in events
    .distinctUntilChanged()   // ignore if next search term is same as previous
    .switchMap(term => term   // switch to new observable each time
      // return the http search observable
      ? this.heroSearchService.search(term)
      // or the observable of empty heroes if no search term
      : Observable.of<Hero[]>([]))
    .catch(error => {
      // TODO: real error handling
      console.log(error);
      return Observable.of<Hero[]>([]);
    });
}
```

如果将每次的用户按键事件都直接调用`HeroSearchService`，那么会发起大量的HTTP请求。这里用了一些方法来减少请求次数同时有保证请求的相对实时性：

* `debounceTime(300)`：在传递最新的字符串之前，字符串事件流每次暂停300毫秒。确保请求的频率不会地狱300毫秒。
* `distinctUntilChanged`：确保只在关键词有变化时才发起请求。
* `switchMap`：对关键词调用查询服务时让它经过`debounce`和`distinctUntilChanged`两个处理。它会取消和丢弃之前的查询的observables，只返回用最新的关键字查询得到的observable。

即使请求至少每300毫秒才发起一次，仍有可能多个请求都在路上还没返回。`switchMap`会只返回最新的那次http请求的observable，之前的请求的结果会被取消和丢弃。

这里还加了一个短路操作。如果关键词为空，那么直接返回一个包含空数组的observable。

#### 引入RxJS操作符

Angular核心库不支持RxJS操作符，所以要单独引入。可以在一个单独的文件中引入所有操作符，然后在`AppModule`中引入这个文件。也可以在用到的时候引入所需要的操作符。（不同的开发者的观点不同）

```js
// app/rxjs-extensions.ts

// Observable class extensions
import 'rxjs/add/observable/of';
import 'rxjs/add/observable/throw';

// Observable operators
import 'rxjs/add/operator/catch';
import 'rxjs/add/operator/debounceTime';
import 'rxjs/add/operator/distinctUntilChanged';
import 'rxjs/add/operator/do';
import 'rxjs/add/operator/filter';
import 'rxjs/add/operator/map';
import 'rxjs/add/operator/switchMap';
```

```js
// app/app.module.ts (rxjs-extensions)

import './rxjs-extensions';
```

#### 将搜索组件添加到Dashboard

将`hero-search`标签添加到`DashboardComponent`的模板中：

```html
<!-- app/dashboard.component.html -->

<h3>Top Heroes</h3>
<div class="grid grid-pad">
  <a *ngFor="let hero of heroes"  [routerLink]="['/detail', hero.id]"  class="col-1-4">
    <div class="module hero">
      <h4>{{hero.name}}</h4>
    </div>
  </a>
</div>
<hero-search></hero-search>
```

在`AppModule`中引入`HeroSearchComponent`并将它添加到`declarations`数组中：

```js
// app/app.module.ts (search)

declarations: [
  AppComponent,
  DashboardComponent,
  HeroDetailComponent,
  HeroesComponent,
  HeroSearchComponent
],
```

刷新浏览器，打开Dashboard试试效果吧。

---

参考资料：

[Angular2 Documentation - Case Study -HTTP](https://angular.io/docs/ts/latest/tutorial/toh-pt6.html)

{% endraw %}
