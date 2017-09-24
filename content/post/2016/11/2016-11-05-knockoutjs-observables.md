---
author: wukn
catalog: true
date: 2016-11-05T01:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Observables'
url: /2016/11/05/knockoutjs-observables/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

KnockoutJS是一个使用MVVM模式的前端框架，可实现数据模型和UI之间的自动更新。

## ko的核心特性

* 监控属性（observables）和依赖跟踪（dependency tracking）
* 声明式绑定（declarative bingdings）
* 模板（templating）

## MVVM

MVVM（Mode-View-ViewModel）是一种构建UI交互的设计模式，将复杂的UI交互分为三个部分：

* 模型（Model）：应用的存储数据模型。该模型表征业务领域的对象和操作，且独立于UI。使用ko时通常通过ajax请求服务器端来读写存储数据模型。

* 视图模型（ViewModel）：表征UI的数据和操作的纯代码。例如，要实现一个列表编辑器，那么视图模型就是一个包含列表项以及添加移除列表项方法的对象。视图模型不是UI本身，它不包含任何UI元素或元素样式等，也不是持久化的数据模型。使用ko时，视图模型与html无关的纯js对象。

* 视图（View）：呈现视图模型状态的可视化的交互UI。视图展示视图模型的数据，向视图模型发送命令，并且在视图模型数据变化时更新显示。使用ko时视图是通过声明式绑定链接到视图模型的html片段。也可以使用模板基于视图模型的数据生成html。

## 1.使用监控属性创建视图模型

使用ko创建视图模型，就是声明一个js对象：

```js
var myViewModel = {
    personName: 'Bob',
    personAge: 123
};
```

然后可以使用声明式绑定为这个视图模型创建一个简单的视图：

```html
<p>The name is <span data-bind="text: personName"></span></p>
<p>The age is <span data-bind="text: personAge"></span></p>
```

data-bind属性并不是html原生属性，它遵循HTML5语法，HTML4不识别但不会产生错误。由于浏览器不认识这个属性，因此需要激活ko使其生效：

```js
ko.applyBindings(myViewModel);
```

可以将这个代码块放在html文档底部，也可以放在页面上方，但需要在DOM-ready处理事件中调用，如jquery的$函数，使其在页面加载完毕后执行。

ko.applyBindings的参数：

* 第一个参数是想要激活的使用声明式绑定的视图模型。
* 第二个参数是可选参数，指定想应用视图模型的DOM元素如ko.applyBindings(myViewModel, document.getElementById('someElementId'))。这样便能将多个视图模型应用到同一个页面不同的视图上。

### 监控属性（observables）

上文展示了如何创建一个简单的视图模型，并且通过绑定来显示它的属性。但ko的一个重要优点是在视图模型变化时更新视图UI。那么，ko是如何知道视图模型发生变化了呢？答案是将模型的属性声明为监控属性（observables），这是一个特殊的js对象，能够通知订阅者发生的变化，并且能自动检测依赖。

```js
var myViewModel = {
    personName: ko.observable('Bob'),
    personAge: ko.observable(123)
};
```

视图部分则并不需要修改，这时，视图模型就能够监控变化，并更新视图。

### 读写监控属性

由于部分浏览器（咳咳，你应该知道这是指谁...）不支持js的getters和setters，为了保证兼容性，`ko.observable`对象实际上是一个函数。

读取监控属性当前值，只需要调用监控属性，不需要参数。本例中，`myViewModel.personName()`将会返回值'Bob'，`myViewModel.personAge()`将会返回值123。

修改监控属性的值，则调用监控属性，参数为要写入的值。同时监控属性支持链式语法，如`myViewModel.personName('Mary').personAge(50)`。

监控属性的意义就是它是被监控的，其他的代码可以订阅其变化的通知。这就是许多ko内置绑定的内部工作原理（注：这应该是观察者模式啊！）。`data-bind="text: personName"`的意思就是将该DOM元素text的绑定注册到自身，并订阅personName的变化。


### 显式订阅监控属性

高级用户可以使用subscribe函数注册自己的订阅来得到监控属性变化的通知。大部分ko组件的内部使用的就是subscribe。

```js
myViewModel.personName.subscribe(function(newValue) {
    alert("The person's new name is " + newValue);
});
```

subscribe函数可接受三个参数，callback是订阅的通知发生的回调函数，target（可选参数）定义回调函数中this的值，event（可选参数，默认为“change”）是接收到的通知事件的名称。

订阅也可以被终止。首先获取到订阅对象，然后调用其dispose函数。

```js
var subscription = myViewModel.personName.subscribe(function(newValue) { /* do stuff */ });
// ...
subscription.dispose();
```

如果想订阅监控属性在值正要变化之前的通知，可以订阅beforeChange事件：

```js
myViewModel.personName.subscribe(function(oldValue) {
    alert("The person's previous name is " + oldValue);
}, null, "beforeChange");
```

PS：KO不能保证beforeChange和Change事件是成对触发的，因为其他代码可能单独地触发某个事件。如果需要跟踪某个监控属性的旧值，是否用订阅完全取决于你。

### 强制监控属性始终通知订阅者

当监控属性为基本类型（数字，字符串、布尔值、null）时，只有值真正改变才会通知订阅者。可以使用内置的通知扩展（notify extender）来保证订阅者总是能够获取到监控属性变化的通知，即使修改后的值和修改前时一样的：

```js
myViewModel.personName.extend({ notify: 'always' });
```

### 延迟（dalaying）或禁止（suppressing）变化通知

通常，监控属性会在其变化时实时通知订阅者。但如果某个监控属性重复地变化，或者会触发一些代价昂贵的更新，可以通过限制或延迟监控属性的变化通知来提高性能。如使用rateLimit扩展限制某通知一定时间内最多通知一次：

```js
// Ensure it notifies about changes no more than once per 50-millisecond period
myViewModel.personName.extend({ rateLimit: 50 });
```

## 2.监控数组

如果要监控和响应一个对象的变化，应该使用监控属性（observables），如果要监控和响应一个集合的变化，那么应该用监控数组（observableArray）。

例如：

```js
var myObservableArray = ko.observableArray();    // Initially an empty array
myObservableArray.push('Some value');            // Adds the value and notifies observers
```

注意：监控数组只跟踪数组中对象的变化，而不是这些对象的状态。

单纯地将一个对象添加到监控数组中并不会使对象的属性成为可监控的。当然，你可以将对象的属性声明为监控属性。监控数组只跟踪它包含的对象，在这些对象被添加或移除时通知监听者。

### 预加载监控数组

如果不希望监控数组一开始就是空的，那么可以加载一个初始对象数组：

```js
// This observable array initially contains three objects
var anotherObservableArray = ko.observableArray([
    { name: "Bungle", type: "Bear" },
    { name: "George", type: "Hippo" },
    { name: "Zippy", type: "Unknown" }
]);
```

### 读取监控数组

监控数组实际上是一个值为数组的监控属性（但添加了一些额外的特性）。因此，可以像其他的监控属性一样，将监控数组作为函数无参调用获得其底层的js数组，并对数组进行读取等操作。

```js
alert('The length of the array is ' + myObservableArray().length);
alert('The first element is ' + myObservableArray()[0]);
```

从技术上来说，可以使用任何原生js数组函数来操作其底层数组，但是通常使用ko提供的更加有用的等效函数，因为：

* 它们兼容所有的浏览器。比如IE8及更早的版本不支持原生js中的indexOf函数，而ko的indexOf函数可用。
对于会修改数组内容的函数，如push和splice，ko提供的方法可以自动触发依赖跟踪机制，所有注册的监听者将能收到变化通知。
* 语法更便捷。如push，ko的写法为`myObservableArray.push(...)`，比底层js数组调用push的方法`myObservableArray().push(...)`更好一些。

下面是监控数组具有的方法。

* indexOf：返回与指定参数相等的第一个数组项的索引，找不到时返回-1。
* slice：与原生js函数slice等效，返回从指定的开始索引到结束索引之间的数组。myObservableArray.slice(...)与myObservableArray().slice(...)是一样的。

监控数组提供一些熟悉的函数（pop, push, shift, unshift, reverse, sort, splice）用于操作数组的内容，并通知监听者：

* `myObservableArray.push('Some new value')` 在数组末尾添加一个新项
* `myObservableArray.pop()` 移除数组的最后一项并返回它
* `myObservableArray.unshift('Some new value')` 在数组的起始项添加一个新项
* `myObservableArray.shift()` 移除数组的第一项并返回它
* `myObservableArray.reverse()` 反转整个数组的顺序
* `myObservableArray.splice()` 移除并返回从指定索引开始指定数量的元素。
* `myObservableArray.sort()` 对数组内容进行排序。默认按字母顺序排序。可以自定义排序函数，函数可接收数组的两项作为参数，前者小于后者则返回一个负数，前者大于后者则返回一个正数，相等则返回0。例如：

```js
myObservableArray.sort(function(left, right) { return left.lastName == right.lastName ? 0 : (left.lastName < right.lastName ? -1 : 1) })
```
关于这些函数的更多细节。可参考标准js数组函数。

监控数组提供了几个原生js数组不具备的函数（remove, removeAll, destroy, destroyAll）。

* `myObservableArray.remove(someItem)` 移除所有与参数相等的元素并将它们以数组形式返回
* `myObservableArray.remove(function(item) { return item.age < 18 })` 移除所有使参数函数返回true的元素并将它们以数组形式返回
* `myObservableArray.removeAll(['Chad', 132, undefined])` 移除所有与参数中其中一个相等的元素并将它们以数组形式返回
* `myObservableArray.removeAll()` 清空数组并返回数组

destroy和destroyAll是为了向Ruby on Rails开发者提供便利。

* `myObservableArray.destroy(someItem)` 查找数组中所有与参数相等的元素并给它们添加特殊属性_destroy并设置为true
* `myObservableArray.destroy(function(someItem) { return someItem.age < 18 })`查找所有使参数函数返回true的元素并给它们添加特殊属性_destroy并设置为true
* `myObservableArray.destroyAll(['Chad', 132, undefined])` 查找所有与参数中其中一个相等的元素并给它们添加特殊属性_destroy并设置为true
* `myObservableArray.destroyAll()` 为数组中所有元素添加特殊属性_destroy并设置为true

\_destroy到底为何物？请看原文：

>So, what’s this \_destroy thing all about? It’s only really interesting to Rails developers. The convention in Rails is that, when you pass into an action a JSON object graph, the framework can automatically convert it to an ActiveRecord object graph and then save it to your database. It knows which of the objects are already in your database, and issues the correct INSERT or UPDATE statements. To tell the framework to DELETE a record, you just mark it with \_destroyset to true.

>Note that when KO renders a foreach binding, it automatically hides any objects marked with_destroy equal to true. So, you can have some kind of “delete” button that invokes thedestroy(someItem) method on the array, and this will immediately cause the specified item to vanish from the visible UI. Later, when you submit the JSON object graph to Rails, that item will also be deleted from the database (while the other array items will be inserted or updated as usual).

### 延迟或禁止变化通知
与监控属性相同，监控数组可以使用rateLimit扩展。

```js
// Ensure it notifies about changes no more than once per 50-millisecond period
myViewModel.myObservableArray.extend({ rateLimit: 50 });
```

## 3.计算监控属性（computed observables）

当有了监控属性firstName和lastName，想要显示fullName怎么办？这时就需要计算监控属性（computed observables）了。计算监控属性依赖于一个或多个监控属性，并在其中任何一个变化时自动更新。

```js
function AppViewModel() {
    this.firstName = ko.observable('Bob');
    this.lastName = ko.observable('Smith');

    this.fullName = ko.computed(function() {
        return this.firstName() + " " + this.lastName();
    }, this);
}
```

视图中绑定计算监控属性：

```html
<p>The name is <span data-bind="text: fullName"></span></p>
```

### 依赖链（dependency chains）

显然，创建一个依赖链是很容易的。例如

* 一个名为items的监控属性表示item的集合
* 另一个名为selectedIndexes的监控属性存储被用户选中的item的索引集合
* 一个名为selectedItems的计算监控属性返回items中selectedIndexes对应的item集合
* 另一个计算监控属性根据selectedItems是否拥有某些属性值返回true或false。界面上一些元素绑定到该属性上。

items或selectedIndexes变化时，依赖链将会起作用，最终界面上绑定到它们的元素将得以更新。

### 管理this

ko.computed的第二个参数是声明计算绑定的this的值。如果不传递该参数，计算属性的函数内部无法访问到`this.firstName()`或`this.lastName()`。因为this表示当前代码所在的对象本身，因此计算监控属性的参数匿名函数内部的this表示该匿名函数自身，也就是说一个对象的this无法通过闭包传递到其内部的匿名函数中去。

通常的做法是避免跟踪this，通过定义一个变量来引用this。

```js
function AppViewModel() {
    var self = this;

    self.firstName = ko.observable('Bob');
    self.lastName = ko.observable('Smith');
    self.fullName = ko.computed(function() {
        return self.firstName() + " " + self.lastName();
    });
}
```

self可被函数闭包所捕获，它在任意嵌套的函数中都是可用且一致的。

### 纯计算监控属性（pure computed observables）

如果计算监控属性是对一些监控属性简单地计算并返回一个值，那么ko.pureComputed将比ko.computed更合适。

```js
this.fullName = ko.pureComputed(function() {
    return this.firstName() + " " + this.lastName();
}, this);
```

纯计算监控属性不会修改其他对象或状态，具有更高的效率和更少的内存占用。当没有其他代码对其有激活的依赖时，ko会自动挂起或释放它。

### 强制计算监控属性始终通知订阅者

与监控属性相同，计算监控属性可以使用notify扩展。

```js
myViewModel.fullName = ko.pureComputed(function() {
    return myViewModel.firstName() + " " + myViewModel.lastName();
}).extend({ notify: 'always' });
```

### 延迟或禁止变化通知

与监控属性相同，监控监控属性可以使用rateLimit扩展。

```js
// Ensure updates no more than once per 50-millisecond period
myViewModel.fullName.extend({ rateLimit: 50 });
```
### 检测一个属性是否为计算监控属性

ko提供ko.isComputed方法来检测一个属性是否为计算监控属性。例如向服务器发送数据时将计算监控属性排除掉：

```js
for (var prop in myObject) {
  if (myObject.hasOwnProperty(prop) && !ko.isComputed(myObject[prop])) {
      result[prop] = myObject[prop];
  }
}
```

此外，ko还提供类似的函数来操作监控属性和计算监控属性：

* `ko.isObservable`：当为监控属性、监控数组或计算监控属性时返回true
* `ko.isWritableObservable`：当为监控属性、监控数组或可写计算监控属性时返回true， (也可以写成`ko.isWriteableObservable`)

当计算监控属性仅用于UI时，可以简写：

```js
function AppViewModel() {
    // ... leave firstName and lastName unchanged ...

    this.fullName = function() {
        return this.firstName() + " " + this.lastName();
    };
}
```

UI元素的绑定中改为方法调用：

```js
<p>The name is <span data-bind="text: fullName()"></span></p>
```

ko会在内部创建一个计算监控属性来检测监控属性的依赖，并在关联的UI元素移除时自动销毁。

### 可写计算监控属性（writable computed observables）

可写计算监控属性是个比较高级的功能，并不常用。

一般，计算监控属性具有一个根据若干监控属性计算得出的值，因此该值通常是只读的。但是只需要提供一个合理的回调函数，可以将计算监控属性变成可写的。

可写计算监控属性的使用和一般的监控属性是一样的，只是需要提供自定义逻辑来拦截读和写。例如：

```js
myViewModel.fullName('Joe Smith').age(50);
```

例1：分解用户输入

以“first name + last name = full name”为例，可以将fullName作为可写计算监控属性，这样用户可以直接输入fullName，其write回调函数解析后映射回firstName和lastName这两个监控属性。

```html
<div>First name: <span data-bind="text: firstName"></span></div>
<div>Last name: <span data-bind="text: lastName"></span></div>
<div class="heading">Hello, <input data-bind="textInput: fullName"/></div>
```

```js
function MyViewModel() {
    this.firstName = ko.observable('Planet');
    this.lastName = ko.observable('Earth');

    this.fullName = ko.pureComputed({
        read: function () {
            return this.firstName() + " " + this.lastName();
        },
        write: function (value) {
            var lastSpacePos = value.lastIndexOf(" ");
            if (lastSpacePos > 0) { // Ignore values with no space character
                this.firstName(value.substring(0, lastSpacePos)); // Update "firstName"
                this.lastName(value.substring(lastSpacePos + 1)); // Update "lastName"
            }
        },
        owner: this
    });
}

ko.applyBindings(new MyViewModel());
```

关于计算属性更多的参数信息：[computed observable reference](http://knockoutjs.com/documentation/computed-reference.html)。

例2：全选/取消选择

当向用户展示一个具有可选项的列表时，通常会提供全选和取消全选功能。可以用一个bool值直观地表示是否全部选中，当其设置为true时全部选中，设置为false时取消全选。

```html
<div class="heading">
    <input type="checkbox" data-bind="checked: selectedAllProduce" title="Select all/none"/> Produce
</div>
<div data-bind="foreach: produce">
    <label>
        <input type="checkbox" data-bind="checkedValue: $data, checked: $parent.selectedProduce"/>
        <span data-bind="text: $data"></span>
    </label>
</div>
```
```js
function MyViewModel() {
    this.produce = [ 'Apple', 'Banana', 'Celery', 'Corn', 'Orange', 'Spinach' ];
    this.selectedProduce = ko.observableArray([ 'Corn', 'Orange' ]);
    this.selectedAllProduce = ko.pureComputed({
        read: function () {
            // Comparing length is quick and is accurate if only items from the
            // main array are added to the selected array.
            return this.selectedProduce().length === this.produce.length;
        },
        write: function (value) {
            this.selectedProduce(value ? this.produce.slice(0) : []);
        },
        owner: this
    });
}
ko.applyBindings(new MyViewModel());
```

例3：值转换器

有时界面上展示的值希望和底层存储的值格式不一样，例如底层用float存储价格，但界面上用户可以输入价格符号和固定位数的小数，像“$25.99”。

```html
<div>Enter bid price: <input data-bind="textInput: formattedPrice"/></div>
<div>(Raw value: <span data-bind="text: price"></span>)</div>
```
```js
function MyViewModel() {
    this.price = ko.observable(25.99);

    this.formattedPrice = ko.pureComputed({
        read: function () {
            return '$' + this.price().toFixed(2);
        },
        write: function (value) {
            // Strip out unwanted characters, parse as float, then write the
            // raw data back to the underlying "price" observable
            value = parseFloat(value.replace(/[^\.\d]/g, ""));
            this.price(isNaN(value) ? 0 : value); // Write to underlying storage
        },
        owner: this
    });
}

ko.applyBindings(new MyViewModel());
```

现在，不管存储的价格是什么，文本框在初始化显示价格时将带有货币符号，且保留固定的两位小数，这样具有更好的用户体验。

例4：过滤和验证用户输入

例1中的可写计算监控属性可以有效地过滤用户的输入，不符合格式的数据将不会被处理。不包含空格的fullName将被忽略。

更进一步，通过验证用户输入是否有效来切换isValid标识的值，并相应地在UI上显示提示信息。

```html
<div>Enter a numeric value: <input data-bind="textInput: attemptedValue"/></div>
<div class="error" data-bind="visible: !lastInputWasValid()">That's not a number!</div>
<div>(Accepted value: <span data-bind="text: acceptedNumericValue"></span>
```

```js
function MyViewModel() {
    this.acceptedNumericValue = ko.observable(123);
    this.lastInputWasValid = ko.observable(true);

    this.attemptedValue = ko.pureComputed({
        read: this.acceptedNumericValue,
        write: function (value) {
            if (isNaN(value))
                this.lastInputWasValid(false);
            else {
                this.lastInputWasValid(true);
                this.acceptedNumericValue(value); // Write to underlying storage
            }
        },
        owner: this
    });
}

ko.applyBindings(new MyViewModel());
```

注：对于简单的输入验证，使用jQuery Validation就行了，一些更复杂的场景则可以使用上面的技术。

### 依赖跟踪是如何工作的

依赖跟踪是简单而优美的。跟踪算法就像这样：

* 声明一个计算监控属性时，ko调用其evaluator函数计算出初始值。
* evaluator执行时，ko会设置一个订阅给任何evaluator读取的监控属性（包括其他的计算监控属性），订阅的回调函数时使evaluator再次执行，循环处理直到回到步骤1（释放任何不再应用的旧回调函数）。
* 当计算监控属性的值发生变化时，ko通知其订阅者。

因此ko不仅在第一次执行evaluator时检测依赖，实际上每次都会重新检测。这意味着依赖是可以动态变化的，依赖A可以决定计算监控属性是否也依赖B和C，然后当A或者目前的选择B或C变化时重新evaluate，这不需要声明依赖，可以由代码执行时决定。

另一个小技巧是声明式绑定是计算监控属性的简单实现。因此一个绑定读取一个监控属性的值时，这个绑定将依赖于该监控属性，当监控属性值发生变化时绑定将会被重新evaluate。

纯计算监控属性的工作方式略有不同：[pure computed observable](http://knockoutjs.com/documentation/computed-pure.html)。

### 使用peek控制依赖

ko的自动依赖跟踪通常与期望是一样的。 但是可能有时需要控制哪些监控属性可以更新计算监控属性，尤其是在计算监控属性会执行如ajax请求等操作时。使用peek函数可以访问监控属性或计算监控属性而不产生依赖。

下面的例子中，该计算属性通过使用ajax和来自另外两个监控属性的参数重新加载一个名为currentPageData的监控属性。该计算监控属性会在pageIndex变化时更新而忽略使用了peek的selectedItem变化。

```js
ko.computed(function() {
    var params = {
        page: this.pageIndex(),
        selected: this.selectedItem.peek()
    };
    $.getJSON('/Some/Json/Service', params, this.currentPageData);
}, this);
```

注：如果不想计算监控属性频繁更新，可使用[rateLimit](http://knockoutjs.com/documentation/rateLimit-observable.html)。

### 忽略一些计算监控属性中的依赖

`ko.ignoreDependencies`函数可以实现指定计算监控属性内部的代码但不触发依赖。在自定义绑定中很常用，比如希望调用会访问监控属性的一段代码，但不想重新触发对这些监控属性变化的绑定。

```js
ko.ignoreDependencies( callback, callbackTarget, callbackArgs );
```

例如：

```js
ko.bindingHandlers.myBinding = {
    update: function(element, valueAccessor, allBindingsAccessor, viewModel, bindingContext) {
        var options = ko.unwrap(valueAccessor());
        var value = ko.unwrap(options.value);
        var afterUpdateHandler = options.afterUpdate;

        // the developer supplied a function to call when this binding updates, but
        // we don't really want to track any dependencies that would re-trigger this binding
        if (typeof afterUpdateHandler === "function") {
            ko.ignoreDependencies(afterUpdateHandler, viewModel, [value, color]);
        }

        $(element).somePlugin("value", value);
    }
}
```

### 注：为什么循环依赖没有意义

计算监控属性是将多个输入监控属性映射成一个输出监控属性，因此依赖链中包含循环并没有意义。循环依赖不像递归，而是像电子表格中两个相互计算的单元格一样。这会导致无限的evaluate循环。

如果依赖图（dependency graph）中包含循环是ko如何处理呢？通过强制遵循以下规则来避免无限循环：ko不会evaluate一个正在evaluate的计算监控属性。这可能会影响代码的执行。相关的两种情况：当两个计算监控属性相互依赖（也许其中一个或者两个都使用deferEvaluation选项），或者当一个计算监控属性写另一个（直接或间接）依赖它的属性。这两种情况下可以使用peek函数来避免循环。

### 纯计算监控属性

纯计算监控属性在V3.2.0以后才支持。与常规的计算监控属性相比具有更好的性能和内存使用。因为如果没有其他对象订阅它，那它就不需要维护依赖者的订阅信息。具有以下特性：

* 避免了来自应用中不再被引用但依赖仍存在的计算监控属性产生内存泄漏。
* 不重复计算不被监控的监控属性来减少计算开销。

一个纯计算监控属性根据它是否有变化订阅者自动在两种状态之间切换：

* 当没有变化订阅者时，它是睡眠状态。进入睡眠状态会释放所有依赖的订阅，在这种状态下，它不会订阅任何调用evaluate方法的监控属性（但仍然对它们保持跟踪），如果在睡眠状态时读取它的值，它的任何依赖发生变化会被自动重新evaluate。
* 当它有变化订阅者时，它是唤醒的并保持监听状态。当进入监听状态时会立即订阅任何依赖。这种状态就是一个常规的计算监控属性。

### 为什么“纯”？

这是借用纯函数（pure function）概念，因为这一特性只适用于evaluator是纯函数的计算监控属性：

* 计算监控属性evaluate不会有副作用。
* 计算监控属性的值不随着evaluate的次数或其他“隐藏”信息而变化。其值仅仅依赖于应用中其他监控属性的值，这些监控属性的值正是纯函数的参数。

定义纯计算监控属性的标准方法是使用ko.pureComputed。

```js
this.fullName = ko.pureComputed(function() {
    return this.firstName() + " " + this.lastName();
}, this);
```

另外，还可以使用ko.computed的pure选项。

```js
this.fullName = ko.computed(function() {
    return this.firstName() + " " + this.lastName();
}, this, { pure: true });
```

更多信息见[computed observable reference](http://knockoutjs.com/documentation/computed-reference.html)。

### 何时应该使用纯计算监控属性

在临时的视图和视图模型中使用并共享持久的视图模型时，纯计算监控属性具有一些优势。在持久化的视图模型中使用纯计算监控属性具有更好的运算性能。在临时视图模型中使用具有更地的内存占用。

下面的例子中，纯计算监控属性fullName只会在最后一步中绑定和更新到视图。

```html
<div class="log" data-bind="text: computedLog"></div>
<!--ko if: step() == 0-->
    <p>First name: <input data-bind="textInput: firstName" /></p>
<!--/ko-->
<!--ko if: step() == 1-->
    <p>Last name: <input data-bind="textInput: lastName" /></p>
<!--/ko-->
<!--ko if: step() == 2-->
    <div>Prefix: <select data-bind="value: prefix, options: ['Mr.', 'Ms.','Mrs.','Dr.']"></select></div>
    <h2>Hello, <span data-bind="text: fullName"> </span>!</h2>
<!--/ko-->
<p><button type="button" data-bind="click: next">Next</button></p>
```

```js
function AppData() {
    this.firstName = ko.observable('John');
    this.lastName = ko.observable('Burns');
    this.prefix = ko.observable('Dr.');
    this.computedLog = ko.observable('Log: ');
    this.fullName = ko.pureComputed(function () {
        var value = this.prefix() + " " + this.firstName() + " " + this.lastName();
        // Normally, you should avoid writing to observables within a pure computed
        // observable (avoiding side effects). But this example is meant to demonstrate
        // its internal workings, and writing a log is a good way to do so.
        this.computedLog(this.computedLog.peek() + value + '; ');
        return value;
    }, this);

    this.step = ko.observable(0);
    this.next = function () {
        this.step(this.step() === 2 ? 0 : this.step()+1);
    };
};
ko.applyBindings(new AppData());
```

### 何时不应该使用纯计算监控属性

希望在计算监控属性的依赖发生变化时执行一定的行为，那么就不适用pure特性。例如：

使用计算监控属性来执行基于多个监控属性的回调函数：

```js
ko.computed(function () {
    var cleanData = ko.toJS(this);
    myDataClient.update(cleanData);
}, this);
```

在自定义绑定的init函数中，使用计算监控属性来更新绑定的元素：

```js
ko.computed({
    read: function () {
        element.title = ko.unwrap(valueAccessor());
    },
    disposeWhenNodeIsRemoved: element
});
```

当evaluator有期望的一些重要的副作用时，不该使用纯计算监控属性，因为当他没有激活的订阅者时evaluator不会执行。如果希望在依赖发生变化是evaluator始终执行，那么使用常规的计算监控属性。

### 检测一个属性是否为纯计算监控属性

使用ko.isPureComputed可检测一个属性是否为纯计算监控属性。

```js
var result = {};
ko.utils.objectForEach(myObject, function (name, value) {
    if (!ko.isComputed(value) || ko.isPureComputed(value)) {
        result[name] = value;
    }
});
```

### 状态变化通知

纯计算监控属性在进入监听状态时会发出awake事件（使用当前值），在进入睡眠状态时会发出asleep事件（使用undefined值）。一般不需要知道它的内部状态，但由于它的内部状态与是否绑定到视图一致的，可以使用这一信息做一些视图模型的初始化或清理工作。

```js
this.someComputedThatWillBeBound = ko.pureComputed(function () {
    ...
}, this);

this.someComputedThatWillBeBound.subscribe(function () {
    // do something when this is bound
}, this, "awake");

this.someComputedThatWillBeBound.subscribe(function () {
    // do something when this is un-bound
}, this, "asleep");
```

（常规计算监控属性使用deferEvaluation选项创建时也会有awake事件。）

### 4.使用fn添加自定义函数

ko的监控属性、监控数组和计算监控属性的依赖关系如下：

![](/img/post/knockoutjs/observables/observable_dependences.png)

可以使用fn为它们添加自定义函数：
* ko.subscribable.fn
* ko.observable.fn
* ko.observableArray.fn
* ko.computed.fn

例：监控数组筛选方法

定义的filterByProperty函数在后续产生的ko.observableArray实例都是可用的：

```js
ko.observableArray.fn.filterByProperty = function(propName, matchValue) {
    return ko.pureComputed(function() {
        var allItems = this(), matchingItems = [];
        for (var i = 0; i < allItems.length; i++) {
            var current = allItems[i];
            if (ko.unwrap(current[propName]) === matchValue)
                matchingItems.push(current);
        }
        return matchingItems;
    }, this);
}
```

该函数返回一个计算得到的筛选后的数组，原数组不变。由于筛选后的数组是计算监控属性，在每次原数组变化时它都会重新计算。

```html
<h3>All tasks (<span data-bind="text: tasks().length"> </span>)</h3>
<ul data-bind="foreach: tasks">
    <li>
        <label>
            <input type="checkbox" data-bind="checked: done" />
            <span data-bind="text: title"> </span>
        </label>
    </li>
</ul>

<h3>Done tasks (<span data-bind="text: doneTasks().length"> </span>)</h3>
<ul data-bind="foreach: doneTasks">
    <li data-bind="text: title"></li>
</ul>
```

```js
function Task(title, done) {
    this.title = ko.observable(title);
    this.done = ko.observable(done);
}

function AppViewModel() {
    this.tasks = ko.observableArray([
        new Task('Find new desktop background', true),
        new Task('Put shiny stickers on laptop', false),
        new Task('Request more reggae music in the office', true)
    ]);

    // Here's where we use the custom function
    this.doneTasks = this.tasks.filterByProperty("done", true);
}

ko.applyBindings(new AppViewModel());
```

如果只是偶尔用到数组筛选功能，不想用自定义函数，可以通过手动构造一个doneTasks来代替：

```js
this.doneTasks = ko.pureComputed(function() {
    var all = this.tasks(), done = [];
    for (var i = 0; i < all.length; i++)
        if (all[i].done())
            done.push(all[i]);
    return done;
}, this);
```

---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/observables.html)
