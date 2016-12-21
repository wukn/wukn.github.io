---
layout:     post
title:      "[Angular2] User Input"
subtitle:   ""
date:       2016-12-21 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 用户操作出发DOM事件。我们通过事件绑定来监听事件，将更新的值回写到组件和数据模型中。

用户点击链接、按下按钮、或者输入文字时，这些行为都会触发DOM事件。这一篇讲述如何使用Angular事件绑定语法来绑定这些事件。

### 绑定用户输入事件

我们可以用Angular的事件绑定来响应DOM事件。很多DOM事件都是由用户输入触发的。绑定这些事件提供了一种获取用户输入的方法。

Angular事件绑定与DOM事件是对应的。将事件名称放在小括号内，并为它赋一个放在引号内的模板表达式。例如，绑定点击事件：

```html
<button (click)="onClickMe()">Click me!</button>
```

等号左边的`(click)`表示绑定目标是按钮的点击事件。右侧引号内的内容是模板表达式（template statement），表示调用组件的`onClickMe`方法来响应按钮的点击事件。

使用绑定时需要注意模板表达式的执行上下文（execution context）。表达式内出现的标识符是属于一个特殊的上下文对象，通常是该模板对应的组件。

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <button (click)="onClickMe()">Click me!</button>
    {{clickMessage}}`
})
export class AppComponent {
  clickMessage = '';

  onClickMe() {
    this.clickMessage = 'You are my hero!';
  }
}
```
{% endraw %}

当用户点击按钮，Angular调用组件的`onClickMe`方法。

![](/img/post/angular2/user-input/click-event.png)

### 从$event对象获取用户操作

DOM事件会携带一些对组件有用的信息。接下来我们为`input`绑定`keyup`事件来获取用户每次按键后的输入：

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
      <input (keyup)="onKey($event)">
      <p>{{values}}</p>
    `
})
export class AppComponent {
  values='';

  onKey(event:any) { // without type info
    this.values += event.target.value + ' | ';
  }
}
```
{% endraw %}

当用户按下按键并释放后，`keyup`事件就触发了，Angular会在内`$event`变量里提供一个DOM事件对象，并传递给组件的`onKey()`方法。

`$event`对象的类型是由触发的事件的类型决定的。不同类型的DOM事件对应`$event`对象的属性不相同。

所有的[标准DOM事件对象](https://developer.mozilla.org/en-US/docs/Web/API/Event)都有个`target`属性，指向触发该事件的元素。上面的例子中，`target`就是`<input>`元素的引用，`event.target.value`也就是这个元素的当前值。

![](/img/post/angular2/user-input/keyup-event-1.png)

注意，组件的`onKey`方法的参数`$event`是`any`类型，放弃了强类型来简化代码。通常还是建议使用强类型：

```js
export class AppComponent {
  values = '';

  onKey(event: KeyboardEvent) { // with type info
    this.values += (<HTMLInputElement>event.target).value + ' | ';
  }
}
```

现在`$event`就是指定的`KeyboardEvent`。不是所有的元素都有`value`这个属性，所以这里将`target`强制转换成一个`input`元素。

使用强类型将DOM事件传递给组件方法也会带来一些问题：组件的方法需要知道模板的太多信息，有点违反了模板一组件之间需要分离关注度这一原则。

### 从模板引用变量获取用户输入

Angular有一个称为本地模板变量（template reference variable）的语法特性。这种变量可以让我们在模板中直接访问一个元素。模板引用变量的声明是在标识符前加一个`#`前缀。

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <input #box (keyup)="0">
    <p>{{box.value}}</p>
  `
})
export class AppComponent { }
```
{% endraw %}

上面的代码中，为`<input>`元素声明了一个模板引用变量`box`。它就是`<input>`元素的引用，因此可以在模板的其他地方访问这个变量。

这样模板就跟组件分离开了。模板不再需要绑定组件，组件也不需要做什么了。

![](/img/post/angular2/user-input/keyup-event-2.png)

上面的代码中，`<input>`绑定了`keyup`事件，但是该事件实际上什么都不做，绑定了数字0（能想到的最短的表达式）。绑定一个空事件是因为，Angular只会在有异步事件需要响应时才更新绑定。

使用模板引用变量代替上面的`$event`来获取文本框的值，这样可以使得组件中从视图获取数据的代码更加整洁，不再需要知道`$event`以及它的结构了。

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <input #box (keyup)="onKey(box.value)">
    <p>{{values}}</p>
  `
})
export class AppComponent {
  values = '';
  onKey(value: string) {
    this.values += value + ' | ';
  }
}
```
{% endraw %}

### 按键事件过滤（使用`key.enter`）

上面的`(keyup)`事件处理了所有的按键。也许我们只关心Enter键，因为塔代表用户完成输入了。一种方法是，在处理方法中进行过滤，先检查`$event.keyCode`是否为Enter，如果是再继续后面的处理。

还有一种更简单的办法。绑定Angular的伪事件`keyup.enter`。Angular会在用户每次按下Enter键调用该事件处理方法。

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <input #box (keyup.enter)="onEnter(box.value)">
    <p>{{value}}</p>
  `
})
export class AppComponent {
  value = '';
  onEnter(value: string) { this.value = value; }
}
```
{% endraw %}

### blur事件

上面的例子没有考虑用户把鼠标移到页面其它地方使input失去焦点的情况。接下来为文本框添加`blur`事件的监听。

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <input #box
      (keyup.enter)="update(box.value)"
      (blur)="update(box.value)">

    <p>{{value}}</p>
  `
})
export class AppComponent {
  value = '';
  update(value: string) { this.value = value; }
}
```
{% endraw %}

### 案例

{% raw %}
```js
@Component({
  selector: 'my-app',
  template: `
    <input #newHero
      (keyup.enter)="addHero(newHero.value)"
      (blur)="addHero(newHero.value); newHero.value='' ">

    <button (click)=addHero(newHero.value)>Add</button>

    <ul><li *ngFor="let hero of heroes">{{hero}}</li></ul>
  `
})
export class AppComponent {
  heroes = ['Windstorm', 'Bombasto', 'Magneta', 'Tornado'];
  addHero(newHero: string) {
    if (newHero) {
      this.heroes.push(newHero);
    }
  }
}
```
{% endraw %}

![](/img/post/angular2/user-input/list-example.png)

借助这个例子，我们复习一下事件绑定的要点：

* 使用模板引用变量来引用元素。模板变量`newHero`指向`<input>`元素的引用，可以在`<input>`元素的兄弟元素和子元素内使用`newHero`。
* 传递值，而不是传递元素。我们不是向组件的`addHero`方法传递`newHero`这个元素，而是而是获取输入框的值传给`addHero`。
* 保持模板表达式简洁。可以看到`(blur)`事件绑定了两个JavaScript表达式。

### 总结

我们学习了基本的响应用户操作的方法。但是这个技术在小规模的demo应用中还可以，处理大量用户操作就显得很繁琐了。双向数据绑定是一种更优雅和简洁的方式，也就是`NgModel`，见下篇。

---

参考资料：

[Angular2 Documentation - User Input](https://angular.io/docs/ts/latest/guide/user-input.html)
