---
author: wukn
catalog: true
date: 2016-12-24T00:00:00Z
tags:
- Angular2
title: '[译][Angular2] Forms'
url: /2016/12/24/angular2-forms/
---

> Angular表单通常包含控件数据绑定、变化跟踪、输入验证、错误展示等。

<!--more-->

有经验的Web开发者会使用合适标签来构建一个用户体验良好的HTML表单，这需要一些技巧。同时需要框架支持双向数据绑定、变化跟踪、验证和错误处理等。

接下来我们学习如何构建一个表单：

* 使用组件和模板构建Angular表单
* 使用双向数据绑定语法`[(ngModel)]`读写input的值
* 使用`ngModel`指令跟踪表单的状态变化和有效性
* 为表单控件添加CSS类，提供可视化的反馈
* 显示验证的错误消息并启用/禁用表单控件
* 使用模板引用变量在HTML元素之间共享信息

我们要实现一个简单的的注册页面，当有必填项没填时，显示验证信息并禁用提交按钮：

![](/img/post/angular2/forms/form.png)

首先，创建一个项目文件夹`angular2-forms`，按照[QuickStart](https://wukn.github.io/2016/11/29/angluar2-quick-start/)创建项目。

### 创建User模型类

用户输入表单数据，我们捕捉变化并更新到模型的实例上。所以我们要先设计数据模型，然后再设计表单布局。

一个数据模型可以简单的是某个事物的属性集合。

在`app`文件夹下新建文件`user.ts`，创建一个`User`类，其中有三个必填字段（`id`、`name`、`email`），一个可选字段（`address`）：

```js
export class User {
  constructor(
    public id: number,
    public name: string,
    public city: string,
    public email?: string
  ) {  }
}
```

TypeScript编译器会为`constructor`的每个`public`参数生成一个公有字段，并在创建新实例的时候将参数值赋给对应的字段。

`email`是可选的（看到email后面的问号了吗），所以实例化的时候可是省略这个参数。

现在我们可以这样创建一个user：

```js
let user =  new User(1, 'Kun',
                       'HF',
                       'kun@wukn.org');
console.log('My name is ' + user.name); // "My name is Kun"
```

### 创建Form组件

一个Angular表单包含两部分：基于HTML的模板和处理数据及用户交互的组件。

在文件夹`app`下创建文件`user-form.component.ts`，

```js
import { Component } from '@angular/core';
import { User }    from './user';
@Component({
  moduleId: module.id,
  selector: 'user-form',
  templateUrl: 'user-form.component.html'
})
export class UserFormComponent {
  cities = ['HeFei', 'BeiJing', 'ShangHai',
            'NanJing', 'ShenZhen'];
  model = new User(18, 'Xiao Hui', this.cities[2], '123@123.com');
  submitted = false;
  onSubmit() { this.submitted = true; }
  // TODO: Remove this when we're done
  get diagnostic() { return JSON.stringify(this.model); }
}
```

创建组件没有什么特别之处，前面都提到过了。

1. 从Angular库中引入`Component`装饰器
2. 引入刚刚创建的`User`数据模型
3. `@Component`的selector指定我们的组件要放在`<user-form>`标签中。
4. `moduleId: module.id`属性 sets the base for module-relative loading of the `templateUrl`（什么意思？）
5. `templateUrl`属性指向一个独立的模板文件
6. 在组件内我们定义了一些数据作为演示。后面我们可以注入一个数据服务来获取及保存数据，或者将这些属性暴露出来供父组件绑定。
7. 在最后我们抛出一个`diagnostic`属性来返回模型的JSON字符串。这样便于开发过程中跟踪调试。

### 修改`app.module.ts`

`app.module.ts`中定义了应用的根模块。在这个文件中，我们声明要使用的外部模块以及属于该模块的组件例如刚创建的`UserFormComponent`。

因为模板驱动的表单是在它们自己的模块中，我们需要在`imports`数组中加入`FormsModule`，这样我们才能使用表单。

```js
import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule }   from '@angular/forms';
import { AppComponent }  from './app.component';
import { UserFormComponent } from './user-form.component';
@NgModule({
  imports: [
    BrowserModule,
    FormsModule
  ],
  declarations: [
    AppComponent,
    UserFormComponent
  ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }
```

修改的地方有三处：

1. 引入`FormsModule`和新创建的`UserFormComponent`。
2. 将`FormsModule`添加到`ngModule`装饰器的`imports`数组列表中。这样应用就可以使用模板驱动表单这个特性了，包括`ngModel`。
3. 将`UserFormComponent`添加到`ngModule`装饰器的`declarations`数组列表中。这样在整个模块中`UserFormComponent`都是可见的。

### 修改`app.component.ts`

`app.component.ts`是应用的根组件。它将负责管理`UserFormComponent`。

```js
import { Component } from '@angular/core';
@Component({
  selector: 'my-app',
  template: '<user-form></user-form>'
})
export class AppComponent { }
```

这里我们只是将模板改成了新创建的`UserFormComponent`的selector。


### 创建HTML表单模板

在`app`文件夹下创建文件`user-form.component.html`：

```html
<div class="container">
    <h1>User Form</h1>
    <form>
      <div class="form-group">
        <label for="name">Name</label>
        <input type="text" class="form-control" id="name" required>
      </div>
      <div class="form-group">
        <label for="email">Email</label>
        <input type="text" class="form-control" id="email">
      </div>
      <button type="submit" class="btn btn-default">Submit</button>
    </form>
</div>
```


这是一个普通的HTML5片段，显示了`User`的两个字段，`name`和`email`。`name`对应的input控件有个HTML5的`required`属性，为`email`没有因为它不是必填的。

到目前位置还没有添加Angular的绑定和指令，只是页面布局。其中，`container`、`form-group`、`form-control`和`btn`这些样式来自Twitter Bootstrap。

安装Bootstrap，在项目根文件夹下执行：

```bash
npm install bootstrap --save
```

在`index.html`中添加样式文件的引用：

```html
<link rel="stylesheet" href="node_modules/bootstrap/dist/css/bootstrap.min.css">
```

### `*ngFor`绑定数组

用户在表单中使用下拉列表来选择城市，使用`NgFor`来绑定数组：

```js
<div class="form-group">
  <label for="city">City</label>
  <select class="form-control" id="city" required>
    <option *ngFor="let city of cities" [value]="city">{{city}}</option>
  </select>
</div>
```

这里将对`cities`数组中的每个`city`重复生成`<options>`标签。

## 使用`*ngModel`双向绑定数据

运行应用，结果如下：

![](/img/post/angular2/forms/form1.png)

界面上没有数据，是因为我们还没有绑定`User`。前面我们已经了解了属性绑定和事件绑定来更新组件属性后和页面的值。这里使用一种新的方法，使用`[(ngModel)]`指令实现双向数据绑定。

将`name`对应的inut标签更新如下：

```html
<input type="text"  class="form-control" id="name"
       required
       [(ngModel)]="model.name" name="name">
  TODO: remove this: {{model.name}}
```

注意，TODO那一行是在暂时调试使用的，后面会删掉。

现在我们主要来看绑定语法`[(ngModel)]="..."`。

如果现在运行应用，改变界面上Name的输入值，可以看到TODO那一行会在每次修改Name时显示当前的值，说明双向数据绑定生效了！

![](/img/post/angular2/forms/ngmodel1.png)

注意到我们还为`<input>`标签添加了一个`name`属性。这个属性值得是唯一的。在表单中使用`[(ngModel)]`时`name`属性是必需的。

> 从内部实现来看，Angular创建`FormControls`并使用Angular附带的`NgForm`指令注册到`<form>`标签。每个`FormControl`是注册到我们指定的`name`属性。

为另外两个属性也加上`[(ngModel)]`绑定，并在最上面添加一个`diagnostic`属性的绑定用于调试。

```html
<div class="container">
    <h1>User Form</h1>
    <form>
      {{diagnostic}}
      <div class="form-group">
        <label for="name">Name</label>
        <input type="text"  class="form-control" id="name"
          required
          [(ngModel)]="model.name" name="name">
      </div>
      <div class="form-group">
        <label for="city">City</label>
        <select class="form-control" id="city" required
          [(ngModel)]="model.city" name="city">
          <option *ngFor="let city of cities" [value]="city">{{city}}</option>
        </select>
      </div>
      <div class="form-group">
        <label for="email">Email</label>
        <input type="text" class="form-control" id="email"
          [(ngModel)]="model.email" name="email">
      </div>
      <button type="submit" class="btn btn-default">Submit</button>
    </form>
</div>
```

看起来结果如下：

![](/img/post/angular2/forms/form2.png)

### `[(ngModel)]`的原理

在属性绑定中，值从模型流向视图中的目标属性，我们将目标属性放在方括号`[]`中，这是从模型到视图的单向数据绑定。

在事件绑定中，我们将视图中的目标属性的值流向模型，我们将目标属性放在圆括号`()`中，这是从视图到模型的单向数据绑定。

Angular将两者结合起来，形成`[()]`，来标识双向数据绑定。

事实上，我们也可以将`NgModel`绑定拆分成两部分：

```js
<input type="text" class="form-control" id="name"
  required
  [ngModel]="model.name" name="name"
  (ngModelChange)="model.name = $event" >
```

这里的属性绑定跟之前介绍的是一样的，但是事件绑定看起来就有点奇怪了。`ngModelChange`并不是`<input>`元素的事件，它实际上是`NgModel`指令的事件属性。当Angular遇到`[(x)]`这样的绑定时，它希望指令`x`有一个`x`输入属性和一个`xChange`输出属性。

另一个奇怪的地方是模板表达式`model.name = $event`。我们在前面见过来自DOM事件的`$event`对象。`ngModelChange`属性并不产生DOM事件，它只是一个Angular的`EventEmitter`属性，当该事件触发时，它返回的是input文本框的值。

### 使用`ngModel`跟踪状态变化和有效性

一个表单不仅仅有数据绑定。我们还希望知道表单中控件的状态。

表单中使用的`ngModel`不仅仅提供了双向数据绑定的功能，它还能告诉我们用户是否点击了控件、值是否有变化、以及值是否有效，

这个指令不仅仅跟踪状态，它还会使用特殊的Angular的CSS类更新组件。我们可以利用这些CSS类来改变控件的外观或控制控件的显示与隐藏。

假设我们定义控件的三个状态：

* 控件是否被访问	 ng-touched  ng-untouched
* 控件的值是否变化  ng-dirty  ng-pristine
* 控件的值是否有效  ng-valid  ng-invalid

我们为`name`对应的input添加一个模板引用变量`spy`：

```html
<input type="text" class="form-control" id="name"
  required
  [(ngModel)]="model.name" name="name"
  #spy >
<br>TODO: remove this: {{spy.className}}
```

运行后我们按照下面四步来检查效果：

1.打开后不点击
![](/img/post/angular2/forms/ngmodel-class1.png)
2.点击输入框，再点其他地方离开输入框
![](/img/post/angular2/forms/ngmodel-class2.png)
3.在输入框中输入若干字符
![](/img/post/angular2/forms/ngmodel-class3.png)
4.清空输入框
![](/img/post/angular2/forms/ngmodel-class4.png)

测试完毕后删掉模板引用变量`spy`和TODO。

### 添加自定义CSS

接下来，我们使用CSS样式来为提示字段是否必填和数据有效性。数据验证有效则输入框左侧显示绿色，无效则显示红色。效果如图：

![](/img/post/angular2/forms/validate.png)

添加两个样式来实现上面的效果。在项目根文件夹下创建文件`forms.css`：

```css
.ng-valid[required], .ng-valid.required  {
  border-left: 5px solid #42A948; /* green */
}

.ng-invalid:not(form)  {
  border-left: 5px solid #a94442; /* red */
}
```

并在`index.html`中添加引用：

```html
<link rel="stylesheet" href="forms.css">
```
上面的提示只指示验证是否有效，如果无效的话，没有给出错误消息。因此我们加上提示消息：

![](/img/post/angular2/forms/validate-msg.png)

为`name`对应的input添加模板引用变量，并在它下面添加显示错误信息的`<div>`：

```html
<label for="name">Name</label>
<input type="text" class="form-control" id="name"
  required
  [(ngModel)]="model.name" name="name"
  #name="ngModel">
<div [hidden]="name.valid || name.pristine"
  class="alert alert-danger">
  Name is required.
</div>
```

这里，我们在控件为`valid`或`pristine`时隐藏提示信息。`pristine`表示页面加载后我们还没有改变控件值的状态。

> 上面的代码中，引用模板变量`name`的值为`ngModel`。为什么是`ngModel`呢？一个指令的`exportAs`属性用来告诉Angular如何链接引用变量到指令。我们将`name`赋值为`ngModel`是因为，`ngModel`指令的`exportAs`属性正好是`ngModel`。

有的小伙伴可能会觉得这个提示信息有点迷惑人。他们只想在用户输入无效数据时才显示这个提示信息，当控件是`pristine`不需要显示。

### 添加User并重置表单

添加一个`New User`按钮，，将点击事件绑定到组件的方法：

```html
<button type="button" class="btn btn-default" (click)="newUser()">New User</button>
```

在组件中添加方法：
```js
newUser() {
  this.model = new User(42, '', '');
}
```

运行应用并点击`Add User`按钮，必填项的文本框左侧会显示红色，但是不会显示提示信息，因为我们并没有改变控件中的值。

输入名称后再点击`Add User`按钮，这时候会现实提示信息。其实我们并不想在创建新用户的时候显示这个提示信息。为什么会这样呢？我们显示的还是一个空的User也显示提示消息了。

这是因为Name文本框不再是原始状态了。用新的User替换原来的之后，控件的状态并没有被恢复为未修改状态。（Angular不能分辨是model被替换了，还是只清除了name属性的值）。我们必须重置表单控件。

为`form`添加一个模板引用变量：

```html
<form #userForm="ngForm">
```

在`Add User`按钮的`click`事件中重置表单：

```html
<button type="button" class="btn btn-default" (click)="newUser(); userForm.reset()">New User</button>
```

### 使用`ngSubmit`提交表单

用户填写完表单后应该能提交表单。在页面地步添加一个`Submit`按钮，该按钮本身并不做什么，但是它会出发表单提交。因为它的`type`为`submit`。但是目前表单提交是无效的。我们要在`<form>`上使用`NgSubmit`指令，绑定到组件的`UserFormComponent.submit()`方法：

```html
<form (ngSubmit)="onSubmit()" #userForm="ngForm">
```

这里，我们定义了一个本地变量`#userForm`，赋值为`ngForm`，这样`userForm`就指向管理整个表单的`NgForm`指令了。

等等，怎么是`NgForm`指令呢？我们根本就没有添加`NgForm`指令啊？

这是因为Angular自动为`<form>`创建并添加`NgForm`指令。

`NgForm`指令为`form`元素补充了一些额外的特性。它包含我们用`ngModel`指令和`name`属性为元素创建的控件，监控它们的属性包括有效性。它本身也有一个`valid`属性，当它包含的控件全都有效时为`true`。

将Submit按钮的`disabled`属性绑定到表单的有效性：

```html
<button type="submit" class="btn btn-default" [disabled]="!userForm.form.valid">Submit</button>
```

现在运行应用，表单打开时是有效状态，`Submit`按钮也是可以点击的。清空Name输入框的内容后，`Submit`按钮将被禁用。

### 在表单之间切换

用一个`<div>`包装表单，将它的`hidden`属性绑定到组件的`submitted`属性：

```html
<div  [hidden]="submitted">
 <h1>User Form</h1>
 <form *ngIf="active" (ngSubmit)="onSubmit()" #userForm="ngForm">

    <!-- ... all of the form ... -->

 </form>
</div>
```

这个表单在页面打开时是可见的，因为`submitted`值为`false`，直到我们提交了表单，它才会是`true`。

```js
submitted = false;

onSubmit() { this.submitted = true; }
```

当我们点击Submit按钮，`submitted`值变成`true`，表单就会被隐藏。这时候显示一些其他信息：

```html
<div [hidden]="!submitted">
  <h2>You submitted the following:</h2>
  <div class="row">
    <div class="col-xs-3">Name</div>
    <div class="col-xs-9  pull-left">{{ model.name }}</div>
  </div>
  <div class="row">
    <div class="col-xs-3">Email</div>
    <div class="col-xs-9 pull-left">{{ model.email }}</div>
  </div>
  <div class="row">
    <div class="col-xs-3">City</div>
    <div class="col-xs-9 pull-left">{{ model.city }}</div>
  </div>
  <br>
  <button class="btn btn-default" (click)="submitted=false">Edit</button>
</div>
```

我们添加了`Edit`按钮，点击后将清除`submitted`标志，这样就重新显示之前的可编辑的表单啦。

在浏览器中的效果如下：

![](/img/post/angular2/forms/form3.png)



---

参考资料：

[Angular2 Documentation - Forms](https://angular.io/docs/ts/latest/guide/forms.html)
