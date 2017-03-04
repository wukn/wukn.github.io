---
layout:     post
title:      "[Angular2] Component Styles"
subtitle:   ""
date:       2017-03-04 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 让我们来看看如何为组件添加CSS样式。

{% raw %}

Angular应用中的样式用的就是标准的CSS。也就是说可以直接为组件应用CSS样式表、选择器、规则和媒体查询。

除此之外，Angular还可以为组件绑定组件样式（component style），这是比常规样式表更加模块化的设计。

### 使用组件样式

对于我们开发的每一个组件，我们不仅仅定义一个HTML模板，通常还会有一个跟模板对应的CSS样式，指定所需的选择器、规则和媒体查询。

一种方法是设置组件元数据的`styles`属性。`styles`属性的值是一个包含CSS代码的字符串数组。例如：

```js
@Component({
  selector: 'hero-app',
  template: `
    <h1>Tour of Heroes</h1>
    <hero-app-main [hero]=hero></hero-app-main>`,
  styles: ['h1 { font-weight: normal; }']
})
export class HeroAppComponent {
/* . . . */
}
```

在组件的style中定义的选择器只会作用与组件的模板内。这种模块化的机制比传统的CSS更好用。

* 可以在每个组件上下文中选择最有语义的CSS类名和选择器
* 类名和选择器只是相对与组件的，不会跟应用的其他地方发生冲突
* 修改应用中其他地方的样式不会影响组件的样式
* 可以将组件的TypeScript代码和HTML代码以及CSS代码放在一起，形成一个干净整洁的项目结构
* 修改或移除组件样式时，不需要去查找应用中其它哪些代码用到了它

### 特殊的选择器

组件样式有几个特殊的选择器，来自于Shadow DOM样式。

#### `:host`

使用`:host`伪类选择器来为组件的宿主元素指定样式。

```css
:host {
  display: block;
  border: 1px solid black;
}
```

`:host`选择器是唯一指向宿主元素的方式。你无法在组件内用其它选择器来指向宿主元素，因为宿主元素不是组件模板的一部分。宿主元素是组件模板的父元素。

使用函数方式在`:host`后的括号内指令另一个选择器，可以为宿主元素添加条件样式。例如，下面这个样式还是针对宿主元素的，但仅仅是在它有`active`CSS类时。

```css
:host(.active) {
  border-width: 3px;
}
```

#### `:host-context`

有的时候为组件视图的外部添加一些样式也是很有用的。例如，一个CSS主题类可以被应用到`<body>`上来改变组件在其中的显示样式。

使用`:host-context()`伪类选择器，它会查找组件宿主元素的父元素的CSS类，直到DOM的根元素。通常`:host-context()`跟另外的选择器结合起来使用。

```css
:host-context(.theme-light) h2 {
  background-color: #eef;
}
```

这个例子中，当组件的父元素有`theme-light`类时，组件内的`<h2>`元素应用`background-color`样式。

#### `/deep/`

组件样式通常只应用于组件自己的模板。

使用`/deep/`选择器可以强制样式应用到所有的子组件视图。它对任意层级的组件嵌套都有用。

```css
:host /deep/ h3 {
  font-style: italic;
}
```

上面的例子将作用于所有`<h3>`元素，包括从宿主元素到这个组件的所有子元素。

`/deep/`也可以写成`>>>`。

### 为组件添加样式

为组件添加样式的方式有几种：

* 设置`styles`或`styleUrls`元数据
* 在HTML模板中行内定义
* 使用CSS import

#### 元数据中设置styles

为`@Component`装饰器添加数组属性`styles`，数组的每一行都是CSS。

```js
@Component({
  selector: 'hero-app',
  template: `
    <h1>Tour of Heroes</h1>
    <hero-app-main [hero]=hero></hero-app-main>`,
  styles: ['h1 { font-weight: normal; }']
})
export class HeroAppComponent {
/* . . . */
}
```

#### 元数据中设置styleUrls

为`@Component`装饰器添加数组属性`styleUrls`，引入外部CSS文件。

```js
@Component({
  selector: 'hero-details',
  template: `
    <h2>{{hero.name}}</h2>
    <hero-team [hero]=hero></hero-team>
    <ng-content></ng-content>
  `,
  styleUrls: ['app/hero-details.component.css']
})
export class HeroDetailsComponent {
/* . . . */
}
```

这个URL相对于应用的根目录，通常也就是`index.html`所在的目录。也可以指定相对于组件文件的URL，见附2。

如果使用Webpack，还可以使用`styles`属性在构建时从外部文件加载样式。例如：

```js
styles: [require('my.component.css')]
```

#### 模板中行内定义样式

可以将样式写在模板中`<style>`标签内。

```js
@Component({
  selector: 'hero-controls',
  template: `
    <style>
      button {
        background-color: white;
        border: 1px solid #777;
      }
    </style>
    <h3>Controls</h3>
    <button (click)="activate()">Activate</button>
  `
})
```

#### 模板链接标签

还可以在模板内使用`<link>`标签。

跟`styleUrls`一样，标签的`href` URL是相对于应用的根目录的，不是相对组件文件的。

```js
@Component({
  selector: 'hero-team',
  template: `
    <link rel="stylesheet" href="app/hero-team.component.css">
    <h3>Team</h3>
    <ul>
      <li *ngFor="let member of hero.team">
        {{member}}
      </li>
    </ul>`
})
```

#### CSS `@imports`

可以使用标准的CSS `@import`规则来向CSS文件中引入CSS文件。

这种场景下，URL是相对于正在引入的这个文件。

```css
/* src/app/hero-details.component.css (excerpt) */

@import 'hero-details-box.css';
```

### 控制视图封装

正如之前所说，组件的CSS样式被封装到组件视图内，且不会影响应用的其它地方。

我们可以在组件的元数据中设置视图封装模式：

* `Native`视图封装：使用浏览器自身的shadow DOM实现来为组件的宿主元素附加一个shadow DOM，然后将组件视图放到这个shadow DOM中。组件的样式也包含在shadow DOM内。
* `Emulated`视图封装（默认模式）：通过预处理（重命名）CSS代码将作用域限制在视图模板内来模拟shadow DOM的行为。详见附1。
* `None`：Angular不做视图封装。Angular将CSS添加到全局样式中。

要设置组件的封装模式，使用组件元数据的`encapsulation`属性：

```js
// warning: few browsers support shadow DOM encapsulation at this time
encapsulation: ViewEncapsulation.Native
```

使用`Native`模式的前提是浏览器原生支持shadow DOM。

### 附1：一窥Emulated视图封装生成的CSS

当使用emulated视图封装时，Angular会预处理所有的组件样式，让它们接近标准的shadow CSS作用域规则。

启用emulated视图封装时，每个DOM元素会被附件一些额外的属性：

```html
<hero-details _nghost-pmm-5>
  <h2 _ngcontent-pmm-5>Mister Fantastic</h2>
  <hero-team _ngcontent-pmm-5 _nghost-pmm-6>
    <h3 _ngcontent-pmm-6>Team</h3>
  </hero-team>
</hero-detail>
```

有两种生成的属性：

* 在native封装模式中应该为shadow DOM宿主的那个元素会被生成一个`_nghost`属性。这通常就是组件的宿主元素。
* 组件视图内的元素会被生成一个`_ngcontent`属性，表明它属于哪一个模拟的shadow DOM。

这些属性的具体值是不重要的。它是自动生成的，并且代码中绝对不可能用到。但是它们会在生成的组件样式中用到，在DOM的`<head>`段中。

```css
[_nghost-pmm-5] {
  display: block;
  border: 1px solid black;
}

h3[_ngcontent-pmm-6] {
  background-color: white;
  border: 1px solid #777;
}
```

### 附2：使用相对URL加载样式

通常我们将组件的代码分成三部分，js代码，HTML和CSS分别放在三个文件中：

```
quest-summary.component.ts
quest-summary.component.html
quest-summary.component.css
```

我们通过元数据的`templateUrl`和`styleUrls`属性来引用它们。那么根据名称来引用它们比指定它们跟应用根目录的相对路径更好一些。

我们可以修改Angular的完整URL的计算规则，将组件元数据的`moduleId`属性赋值为`module.id`：

```js
@Component({
  moduleId: module.id,
  selector: 'quest-summary',
  templateUrl: './quest-summary.component.html',
  styleUrls:  ['./quest-summary.component.css']
})
export class QuestSummaryComponent { }
```

{% endraw %}

---

参考资料：

[Angular2 Documentation - Component Styles](https://angular.io/docs/ts/latest/guide/component-styles.html)
