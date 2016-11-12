---
layout:     post
title:      "[KnockoutJS] Mapping"
subtitle:   "映射"
date:       2016-11-12 10:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - KnockoutJS

---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

ko允许将任意js对象作为视图模型使用。只要视图模型的一些属性是监控属性，就可以将它们绑定到UI上，UI会随着监控属性的变化自动更新。

部分应用都需要从服务器获取数据，但是服务器端是与监控属性无关的，它只会提供普通的js对象（通常序列化为json）。如果不想手动使用服务器提供的普通js对象数据来构建视图模型，可以使用映射插件将普通js对象转换为有监控属性的视图模型。

knockout.mapping：[下载地址](https://github.com/SteveSanderson/knockout.mapping/tree/master/build/output)

### 例：不使用ko.mapping插件时的手动映射

定义一个视图模型：

```js
var viewModel = {
    serverTime: ko.observable(),
    numUsers: ko.observable()
}
```

对应的视图：

```html
The time on the server is: <span data-bind='text: serverTime'></span>
and <span data-bind='text: numUsers'></span> user(s) are connected.
```

使用jQuery’的`$.getJSON`或`$.ajax`从服务器获取数据：

```js
var data = getDataUsingAjax();          // Gets the data from the server
```

服务器返回的数据类似这样：

```javasvript
{
    serverTime: '2010-01-07',
    numUsers: 3
}
```

将数据更新到视图模型：

```javasvript
// Every time data is received from the server:
viewModel.serverTime(data.serverTime);
viewModel.numUsers(data.numUsers);
```

### 例：使用ko.mapping插件自动映射

使用映射插件创建视图模型时，使用`ko.mapping.fromJS`函数创建`viewModel`：

```js
var viewModel = ko.mapping.fromJS(data);
```

它会自动为`data`的每个属性创建监控属性。每次从服务器接受数据后，可以再次使用`ko.mapping.fromJS`函数更新`viewModel`：

```js
// Every time data is received from the server:
ko.mapping.fromJS(data, viewModel);
```

映射时：

* 对象的所有属性转换成监控属性。如果更新会改变它的值，那么就更新对应的监控属性。
* 数组转换成监控数组。如果更新会改变数组项的数量，它会采取适当的添加或移除操作。同样，会尽量保持顺序和原js数组一致。

### 取消映射

如果要将映射的对象还原成普通的js对象：

```js
var unmapped = ko.mapping.toJS(viewModel);
```
这样会创建一个非映射的对象，它的属性只包含映射对象中是原js对象属性的部分。也就是说，手动添加到视图模型的属性或函数会被忽略。有一个例外，就是`_destroy`属性会被映射回来，因为它可能时ko在删除监控数组中的项时产生的。

### 与json字符串同时使用

如果ajax请求返回的是json字符串（没有反序列化为js对象），映射时使用`ko.mapping.fromJSON`，取消映射时使用`ko.mapping.toJSON`。

### 控制映射

有时可能需要对映射操作进行更多的处理。可以在调用`ko.mapping.fromJS`时指定映射选项，后续调用时就不需要指定了。

#### 使用keys唯一标示对象

假设有js对象如下：

```js
var data = {
    name: 'Scot',
    children: [
        { id : 1, name : 'Alicw' }
    ]
}
```

可以默认这样映射：

```js
var viewModel = ko.mapping.fromJS(data);
```

现在假设数据被更新了：

```js
var data = {
    name: 'Scott',
    children: [
        { id : 1, name : 'Alice' }
    ]
}
```

可以这样用新数据来更新视图模型：

```js
ko.mapping.fromJS(data, viewModel);
```

这时，`name`会跟期望的一样被更新。而`children`数组，原来的（`Alicw`）会被移除，再添加一个新的（`Alice`），这也许并不是期望的结果，希望的是数组项的`name`从`Alicw`更新为`Alice`，而不是整个数组项被替换。

这是因为映射插件默认是简单地比较数组中对象是否一样，而对象`{ id : 1, name : 'Alicw' }`和对象`{ id : 1, name : 'Alice' }`是不相等的，插件默认认为该数组项需要被移除，替换为新的。

为了解决这个问题，可以指定映射插件使用哪个`key`来比较对象。

```js
var mapping = {
    'children': {
        key: function(data) {
            return ko.utils.unwrapObservable(data.id);
        }
    }
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

这样，每次映射插件在检查数组`children`的项时，会根据`id`属性来决定对象是应该被移除还是仅仅只需要更新。

#### 使用create自定义对象构造函数

如果希望映射时部分自己处理，可以提供`create`回调函数。

假设有js对象如下：

```js
var data = {
    name: 'Graham',
    children: [
        { id : 1, name : 'Lisa' }
    ]
}
```

如果想自己映射`children`数组，可以这样指定：

```js
var mapping = {
    'children': {
        create: function(options) {
            return new myChildModel(options.data);
        }
    }
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

`create`回调函数的参数`options`是包含以下属性的对象：

* `data`：该数组子项对应的包含数据的js对象
* `parent`：该数组子项所属的父对象或数组。

当然，在`create`回调函数中还可以使用`ko.mapping.fromJS`。典型的案例就是向原js对象添加额外的监控属性：

```js
var myChildModel = function(data) {
    ko.mapping.fromJS(data, {}, this);

    this.nameLength = ko.computed(function() {
        return this.name().length;
    }, this);
}
```

#### 使用update自定义对象更新

指定`update`回调函数可以自定义对象更新的逻辑。它接收需要更新的对象和options对象，并需要返回更新后的值。

options包含以下属性：

* `data`：该数组子项对应的包含数据的js对象
* `parent`：该数组子项所属的父对象或数组。
* `observable`：如果该属性是监控属性，那么它就是实际的监控属性

```js
var data = {
    name: 'Graham',
}

var mapping = {
    'name': {
        update: function(options) {
            return options.data + 'foo!';
        }
    }
}
var viewModel = ko.mapping.fromJS(data, mapping);
alert(viewModel.name());
```

这将会弹出提示框“Grahamfoo!”。

#### 使用ignore忽略指定属性

如果在映射时需要忽略一些属性，那么指定一个忽略属性名称数组：

```javasvript
var mapping = {
    'ignore': ["propertyToIgnore", "alsoIgnoreThis"]
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

映射选项中指定的忽略数组会和默认的忽略数组合并。默认忽略数组可以这样使用：

```js
var oldOptions = ko.mapping.defaultOptions().ignore;
ko.mapping.defaultOptions().ignore = ["alwaysIgnoreThis"];
```

#### 使用include包含指定属性

将视图模型还原为js对象时，映射插件默认只会包含原对象中的属性，还有生成的原对象中没有的_destroy属性。
可以自定义这样一个数组来指定包含一些属性：

```js
var mapping = {
    'include': ["propertyToInclude", "alsoIncludeThis"]
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

映射选项中指定的包含数组会和默认的包含数组合并。默认的包含数组默认之包含_destroy。默认包含数组可以这样使用：

```js
var oldOptions = ko.mapping.defaultOptions().include;
ko.mapping.defaultOptions().include = ["alwaysIncludeThis"];
```
#### 使用copy复制指定属性

将视图模型还原为js对象时，可以强制映射插件只是复制属性而不是转换为监控属性：

```js
var mapping = {
    'copy': ["propertyToCopy"]
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

映射选项中指定的复制数组会和默认的复制数组合并。默认的复制数组默认为空。默认复制数组可以这样使用：

```js
var oldOptions = ko.mapping.defaultOptions().copy;
ko.mapping.defaultOptions().copy = ["alwaysCopyThis"];
```

#### 使用observe监控指定属性

映射时只希望将指定属性映射为监控属性，其它保持复制时，可以执行监控属性的名称数组：

```js
var mapping = {
    'observe': ["propertyToObserve"]
}
var viewModel = ko.mapping.fromJS(data, mapping);
```

映射选项中指定的监控属性名称数组会和默认的监控属性名称数组合并。默认的监控属性名称数组默认为空。默认监控属性名称数组可以这样使用：

```js
var oldOptions = ko.mapping.defaultOptions().observe;
ko.mapping.defaultOptions().observe = ["onlyObserveThis"];
```

ignore和include仍然会继续有效。copy数组可用来高效地复制含有子对象的数组或对象属性。如果数组或对象属性没有在copy和observe中指定，那它会被递归地映射：

```js
var data = {
    a: "a",
    b: [{ b1: "v1" }, { b2: "v2" }]
};

var result = ko.mapping.fromJS(data, { observe: "a" });
var result2 = ko.mapping.fromJS(data, { observe: "a", copy: "b" }); //will be faster to map.
```

`result`和`result2`都将是：

```js
{
    a: observable("a"),
    b: [{ b1: "v1" }, { b2: "v2" }]
}
```

深入数组或对象时也能使用，copy和observe会冲突：

```js
var data = {
    a: "a",
    b: [{ b1: "v1" }, { b2: "v2" }]
};
var result = ko.mapping.fromJS(data, { observe: "b[0].b1"});
var result2 = ko.mapping.fromJS(data, { observe: "b[0].b1", copy: "b" });
```

`result`将会是：

```js
{
    a: "a",
    b: [{ b1: observable("v1") }, { b2: "v2" }]
}
```

而`result2`将会是：

```js
{
    a: "a",
    b: [{ b1: "v1" }, { b2: "v2" }]
}
```

#### 指定更新目标

就像上面的例子，在一个对象内部映射时，可能希望使用this来作为更新操作的目标。`ko.mapping.fromJS`的第三个参数就是这个更新目标。

```js
ko.mapping.fromJS(data, {}, someObject); // overwrites properties on someObject
```

如果希望映射js对象到this：

```js
ko.mapping.fromJS(data, {}, this);
```

### 多源映射

多次调用`ko.mapping.fromJS`可以将多个js对象合并到一个视图模型中。

```js
var viewModel = ko.mapping.fromJS(alice, aliceMappingOptions);
ko.mapping.fromJS(bob, bobMappingOptions, viewModel);
```

每个调用指定的映射选项会被合并。

### 映射的监控数组

映射插件生成的监控数组增加了一些可以使用keys映射的方法：

* mappedRemove
* mappedRemoveAll
* mappedDestroy
* mappedDestroyAll
* mappedIndexOf

它们从功能上来看和普通`ko.observableArray`的函数是一样的，但是他们可以基于对象的key来处理完成一些处理。例如：

```js
var obj = [
    { id : 1 },
    { id : 2 }
]

var result = ko.mapping.fromJS(obj, {
    key: function(item) {
        return ko.utils.unwrapObservable(item.id);
    }
});

result.mappedRemove({ id : 2 });
```

映射的监控数组还暴露了`mappedCreate`函数：

```js
var newItem = result.mappedCreate({ id : 3 });
```

它会首先检查key是否已存在，如果已存在则抛出错误。下一步，它会调用create和update回调函数，如果有的话，来创建新对象。最后，将对象添加进数组并返回它。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/plugins-mapping.html)
