---
author: wukn
catalog: true
date: 2016-11-12T03:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Bindings - Creating custom binding'
url: /2016/11/12/knockoutjs-bindings-creating-custom-binding/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## 自定义绑定

### 注册自定义绑定

向`ko.bindingHandles`添加一个子属性来注册一个绑定。

```js
ko.bindingHandlers.yourBindingName = {
    init: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        // This will be called when the binding is first applied to an element
        // Set up any initial state, event handlers, etc. here
    },
    update: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        // This will be called once when the binding is first applied to an element,
        // and again whenever any observables/computeds that are accessed change
        // Update the DOM element based on the supplied values here.
    }
};
```

然后就可以向DOM元素使用它了。

```html
<div data-bind="yourBindingName: someValue"> </div>
```

### 回调函数

创建自定义绑定时需要提供回调函数`init`和`update`，可以只提供其中一个。

ko会在绑定应用到元素上时调用`init`，之后便不再调用，一般用于初始化元素状态和注册事件的回调函数。

ko会在初始绑定应用到元素上时调用`update`，然后在其访问的任意依赖变化时也会调用。

回调函数的参数：
* element：使用绑定的DOM元素。
* valueAccessor：返回绑定的模型属性的js函数。例如调用`valueAccessor()`来获取当前模型属性值。为了能兼容监控变量和普通值，使用`ko.unwrap`，如`ko.unwrap(valueAccessor())`。
* allBindings：可以用来访问DOM元素绑定的所有模型值的js对象。例如`allBindings.get('name')`获取`name`绑定的值，不存在时为`undefined`。可以用`allBindings.has('name')`来检测其是否存在。
* bindingContext：包含该元素的绑定可用的上下文的对象。（在v3版本之前，该参数是绑定的试图模型`viewModel`，现在可用`bindingContext.$data`或`bindingContext.$rawData`来代替。）

例如，可以用`visible`绑定来控制元素的可见性，但希望在元素状态变化时添加动画。可以使用jQuery的`slideUp`/`slideDown`创建一个自定义绑定。

```js
ko.bindingHandlers.slideVisible = {
    update: function(element, valueAccessor, allBindings) {
        // First get the latest data that we're bound to
        var value = valueAccessor();

        // Next, whether or not the supplied model property is observable, get its current value
        var valueUnwrapped = ko.unwrap(value);

        // Grab some more data from another binding property
        var duration = allBindings.get('slideDuration') || 400; // 400ms is default duration unless otherwise specified

        // Now manipulate the DOM element
        if (valueUnwrapped == true)
            $(element).slideDown(duration); // Make the element visible
        else
            $(element).slideUp(duration);   // Make the element invisible
    }
};
```

```html
<div data-bind="slideVisible: giftWrap, slideDuration:600">You have selected the option</div>
<label><input type="checkbox" data-bind="checked: giftWrap" /> Gift wrap</label>

<script type="text/javascript">
    var viewModel = {
        giftWrap: ko.observable(true)
    };
    ko.applyBindings(viewModel);
</script>
```

如果希望`slideVisible`在页面第一次加载时不使用动画，只是在用户改变模型状态时才出现动画，可以添加`init`方法:

```js
ko.bindingHandlers.slideVisible = {
    init: function(element, valueAccessor) {
        var value = ko.unwrap(valueAccessor()); // Get the current value of the current property we're bound to
        $(element).toggle(value); // jQuery will hide/show the element depending on whether "value" or true or false
    },
    update: function(element, valueAccessor, allBindings) {
        // Leave as before
    }
};
```

### DOM变化后更新监控属性

上面展示了如何使用`update`来根据监控属性变化更新DOM元素，那怎样在用户操作DOM元素之后更新关联的监控属性呢？
可以在`init`中注册一些事件的回调函数来更新监控属性。

```js
ko.bindingHandlers.hasFocus = {
    init: function(element, valueAccessor) {
        $(element).focus(function() {
            var value = valueAccessor();
            value(true);
        });
        $(element).blur(function() {
            var value = valueAccessor();
            value(false);
        });
    },
    update: function(element, valueAccessor) {
        var value = valueAccessor();
        if (ko.unwrap(value))
            element.focus();
        else
            element.blur();
    }
};
```

```html
<p>Name: <input data-bind="hasFocus: editingName" /></p>

<!-- Showing that we can both read and write the focus state -->
<div data-bind="visible: editingName">You're editing the name</div>
<button data-bind="enable: !editingName(), click:function() { editingName(true) }">Edit name</button>

<script type="text/javascript">
    var viewModel = {
        editingName: ko.observable()
    };
    ko.applyBindings(viewModel);
</script>
```

## 控制后代的自定义绑定

默认情况下，绑定只会作用于应用它的元素。如果希望作用于所有后代元素，只需在`init`中`return { controlsDescendantBindings: true }`。

```js
ko.bindingHandlers.allowBindings = {
    init: function(elem, valueAccessor) {
        // Let bindings proceed as normal *only if* my value is false
        var shouldAllowBindings = ko.unwrap(valueAccessor());
        return { controlsDescendantBindings: !shouldAllowBindings };
    }
};
```

```html
<div data-bind="allowBindings: true">
    <!-- This will display Replacement, because bindings are applied -->
    <div data-bind="text: 'Replacement'">Original</div>
</div>

<div data-bind="allowBindings: false">
    <!-- This will display Original, because bindings are not applied -->
    <div data-bind="text: 'Replacement'">Original</div>
</div>
```

### 向后代绑定传递额外参数

通常使用`controlsDescendantBindings`的绑定也会调用`ko.applyBindingsToDescendants(someBindingContext, element)`，使后代绑定应用于一些修改的上下文。

例如，自定义的绑定`withProperties`向绑定上下文附加一些额外的属性，以便在后代绑定中能访问。

```js
ko.bindingHandlers.withProperties = {
    init: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        // Make a modified binding context, with a extra properties, and apply it to descendant elements
        var innerBindingContext = bindingContext.extend(valueAccessor);
        ko.applyBindingsToDescendants(innerBindingContext, element);

        // Also tell KO *not* to bind the descendants itself, otherwise they will be bound twice
        return { controlsDescendantBindings: true };
    }
};
```

可以看到，绑定上下文有个`extend`方法，返回一个复制的并带有额外属性的上下文。`extend`方法接受的参数要么是一个带有属性的对象，要么是能返回这样的对象的函数。建议使用函数作为参数，这样后续的绑定值变化也会更新到绑定上下文中，并且不影响当前的绑定上下文，也就是说不影响兄弟层级的元素，只会影响后代元素。

```html
<div data-bind="withProperties: { emotion: 'happy' }">
    Today I feel <span data-bind="text: emotion"></span>. <!-- Displays: happy -->
</div>
<div data-bind="withProperties: { emotion: 'whimsical' }">
    Today I feel <span data-bind="text: emotion"></span>. <!-- Displays: whimsical -->
</div>
```

### 在绑定上下文中添加额外的层级

`with`和`foreach`绑定会在绑定上下文中添加额外的层级。这意味着它们的后代能够使用`$parent`、`$parents`、`$root`或`$parentContext`来访问外层数据。

如果在自定义绑定中实现该功能，除了使用`bindingContext.extend()`，还可以使用`bindingContext.createChildContext(someData)`。它会返回一个新上下文，该上下文的视图模型是`someData`，父上下文`$parentContext`是`bindingContext`。如果需要，还可以使用`ko.utils.extend`为该子上下文扩展额外的属性。

```js
ko.bindingHandlers.withProperties = {
    init: function(element, valueAccessor, allBindings, viewModel, bindingContext) {
        // Make a modified binding context, with a extra properties, and apply it to descendant elements
        var childBindingContext = bindingContext.createChildContext(
            bindingContext.$rawData,
            null, // Optionally, pass a string here as an alias for the data item in descendant contexts
            function(context) {
                ko.utils.extend(context, valueAccessor());
            });
        ko.applyBindingsToDescendants(childBindingContext, element);

        // Also tell KO *not* to bind the descendants itself, otherwise they will be bound twice
        return { controlsDescendantBindings: true };
    }
};
```

这样，`withProperties`绑定就可以嵌套使用了，并且每个嵌套层级可以通过`$parentContext`访问父层。

```html
<div data-bind="withProperties: { displayMode: 'twoColumn' }">
    The outer display mode is <span data-bind="text: displayMode"></span>.
    <div data-bind="withProperties: { displayMode: 'doubleWidth' }">
        The inner display mode is <span data-bind="text: displayMode"></span>, but I haven't forgotten
        that the outer display mode is <span data-bind="text: $parentContext.displayMode"></span>.
    </div>
</div>
```

## 支持虚拟元素的自定义绑定

使用`ko.virtualElements.allowedBindings`告诉ko自定义绑定能够识别虚拟元素。

例如，创建一个对DOM节点随机排序的自定义绑定：

```js
ko.bindingHandlers.randomOrder = {
    init: function(elem, valueAccessor) {
        // Pull out each of the child elements into an array
        var childElems = [];
        while(elem.firstChild)
            childElems.push(elem.removeChild(elem.firstChild));

        // Put them back in a random order
        while(childElems.length) {
            var randomIndex = Math.floor(Math.random() * childElems.length),
                chosenChild = childElems.splice(randomIndex, 1);
            elem.appendChild(chosenChild[0]);
        }
    }
};
```

这对普通DOM元素是可用的：

```html
<div data-bind="randomOrder: true">
    <div>First</div>
    <div>Second</div>
    <div>Third</div>
</div>
```

但是对虚拟元素却不起作用，会报错“The binding 'randomOrder' cannot be used with virtual elements”：
```html
<!-- ko randomOrder: true -->
    <div>First</div>
    <div>Second</div>
    <div>Third</div>
<!-- /ko -->
```

要想`randomOrder`支持虚拟元素，需要配置ko：

```js
ko.virtualElements.allowedBindings.randomOrder = true;
```

这样就不会报错了，但仍然不起作用，因为绑定中调用的是普通DOM的API，不识别虚拟元素。这就是为什么ko要求显式地声明支持虚拟元素。使用虚拟元素API更新自定义绑定：

```js
ko.bindingHandlers.randomOrder = {
    init: function(elem, valueAccessor) {
        // Build an array of child elements
        var child = ko.virtualElements.firstChild(elem),
            childElems = [];
        while (child) {
            childElems.push(child);
            child = ko.virtualElements.nextSibling(child);
        }

        // Remove them all, then put them back in a random order
        ko.virtualElements.emptyNode(elem);
        while(childElems.length) {
            var randomIndex = Math.floor(Math.random() * childElems.length),
                chosenChild = childElems.splice(randomIndex, 1);
            ko.virtualElements.prepend(elem, chosenChild[0]);
        }
    }
};
```

现在该绑定对普通DOM元素和虚拟元素都支持了，因为`ko.virtualElements`的API是兼容普通DOM元素的。

### 虚拟元素API

* `ko.virtualElements.allowedBindings`
该对象的key决定哪些绑定可用于虚拟元素。如`ko.virtualElements.allowedBindings.mySuperBinding = true`意味着允许`mySuperBinding`用于虚拟元素。

* `ko.virtualElements.emptyNode(containerElem)`

移除真实或虚拟元素`containerElem`的所有子节点（通过清除他们关联的任何数据来避免内存泄露）。

* `ko.virtualElements.firstChild(containerElem)`

返回真实或虚拟元素`containerElem`的第一个子节点，不存在时返回null。

* `ko.virtualElements.insertAfter(containerElem, nodeToInsert, insertAfter)`

添加`nodeToInsert`为真实或虚拟元素`containerElem`的子节点，位置在`insertAfter`之后（`insertAfter`必须是`containerElem`的子节点）。

* `ko.virtualElements.nextSibling(node)`

返回真实或虚拟元素的`node`的后的兄弟节点，不存在时返回null。

* `ko.virtualElements.prepend(containerElem, nodeToPrepend)`

将`nodeToPrepend`添加为真实或虚拟元素`containerElem`的第一个子节点。

* `ko.virtualElements.setDomNodeChildren(containerElem, arrayOfNodes)`

移除真实或虚拟元素`containerElem`的所有子节点（通过清除他们关联的任何数据来避免内存泄露），并将`arrayOfNodes`中的节点作为的新的子节点插入。

## 自定义dispose

在使用模板绑定或流程控制绑定时，一些DOM元素会被动态添加或移除。当创建自定义绑定时，通常需要添加ko移除关联的元素时的清除逻辑。

### 为元素的释放注册回调函数

调用`ko.utils.domNodeDisposal.addDisposeCallback(node, callback)`来注册节点被删除的回调函数。

```js
ko.bindingHandlers.myWidget = {
    init: function(element, valueAccessor) {
        var options = ko.unwrap(valueAccessor()),
            $el = $(element);

        $el.myWidget(options);

        ko.utils.domNodeDisposal.addDisposeCallback(element, function() {
            // This will be called when the element is removed by Knockout or
            // if some other part of your code calls ko.removeNode(element)
            $el.myWidget("destroy");
        });
    }
};
```

### 覆盖外部数据的清除逻辑

当ko移除元素时会执行清除逻辑来清除该元素关联的任何数据。在清除逻辑中，如果jQuery可用，那么是通过调用jQuery的`cleanData`方法。在一些高级的应用场景中，可能希望阻止或自定义数据移除的方式。可以通过覆盖`ko.utils.domNodeDisposal.cleanExternalData(node)`支持自定义逻辑。例如，要阻止调用`cleanData`，可以用一个空函数来替代标准的`cleanExternalData`实现。

```js
ko.utils.domNodeDisposal.cleanExternalData = function () {
    // Do nothing. Now any jQuery data associated with elements will
    // not be cleaned up when the elements are removed from the DOM.
};
```

###使用预处理（preprocessing）扩展ko的绑定语法

从ko V3.0版本开始，ko支持通过添加一个可以在绑定过程中重写DOM节点并绑定字符串的回调函数来定义一个自定义语法。

### 预处理绑定字符串

可以为某个绑定处理器（click、visible或自定义绑定）提供一个绑定预处理器（binding preprocessor）来链接到ko的`data-bind`解释逻辑中。

通过对绑定处理器添加`preprocess`函数。

```js
ko.bindingHandlers.yourBindingHandler.preprocess = function(stringFromMarkup) {
    // Return stringFromMarkup if you don't want to change anything, or return
    // some other string if you want Knockout to behave as if that was the
    // syntax provided in the original HTML
}
```

### 例1：为绑定设置默认值

如果绑定是没有指定value，会默认为`undefined`。可以使用预处理器使绑定有个初始值。

```js
ko.bindingHandlers.uniqueName.preprocess = function(val) {
    return val || 'true';
}
```

`uniqueName`将允许绑定时不指定value，而默认为`true`。

```html
<input data-bind="value: someModelProperty, uniqueName" />
```

### 例2：为事件绑定表达式

如果希望为`click`事件绑定表达式（而不是ko期望的函数引用），可以使用预处理器。

```html
ko.bindingHandlers.click.preprocess = function(val) {
    return 'function($data,$event){ ' + val + ' }';
}
```

```html
<button type="button" data-bind="click: myCount(myCount()+1)">Increment</button>
```

### 绑定预处理器

ko.bindingHandlers.<name>.preprocess(value, name, addBindingCallback)

如果定义了该函数，那么在绑定被评估之前每个`<name> `的绑定时都会调用该函数。

`preprocess`的参数如下：
* value：ko试图解析绑定之前，绑定的值的语法。（例如`yourBinding: 1 + 1`中，关联的值是字符串"1 + 1"）
* name：绑定的名称。（例如`yourBinding: 1 + 1`中，名称是字符串"yourBinding"）
* addBinding：可选的回调函数，用于为当前元素添加另一个绑定。它有两个参数，`name`和`value`。例如，在预处理方法中，调用`addBinding('visible', 'acceptsTerms()');`来让ko表现得看起来就像元素已经有了`visible: acceptsTerms()`绑定。

`preprocess`必须返回一个新的字符串值来被解析并传递给绑定，或者返回`undefined`来移除绑定。返回非字符串值是没有意义的，因为HTML一定是字符串。例如将'value + ".toUpperCase()"'作为字符串返回，那么`yourBinding: "Bert"`会被解析成就像HTML中包含`yourBinding: "Bert".toUpperCase()`一样。ko会按照正常方式解析返回的值，因此必须是合法的js表达式。

### 预处理DOM节点

可以提供一个节点预处理器（node preprocessor）来链接到ko的DOM处理中。ko会在遍历每个DOM节点时调用该函数一次，包括初次绑定UI，和后续的任意DOM子树被注入时。
通过向`bindingProvider`提供一个`preprocessNode`：

```js
ko.bindingProvider.instance.preprocessNode = function(node) {
    // Use DOM APIs such as setAttribute to modify 'node' if you wish.
    // If you want to leave 'node' in the DOM, return null or have no 'return' statement.
    // If you want to replace 'node' with some other set of nodes,
    //    - Use DOM APIs such as insertChild to inject the new nodes
    //      immediately before 'node'
    //    - Use DOM APIs such as removeChild to remove 'node' if required
    //    - Return an array of any new nodes that you've just inserted
    //      so that Knockout can apply any bindings to them
}
```

### 例3：虚拟模板元素

用常规的语法在虚拟元素上使用模板时比较复杂。通过使用预处理，可以添加一个使用单注解的模板格式：

```js
ko.bindingProvider.instance.preprocessNode = function(node) {
    // Only react if this is a comment node of the form <!-- template: ... -->
    if (node.nodeType == 8) {
        var match = node.nodeValue.match(/^\s*(template\s*:[\s\S]+)/);
        if (match) {
            // Create a pair of comments to replace the single comment
            var c1 = document.createComment("ko " + match[1]),
                c2 = document.createComment("/ko");
            node.parentNode.insertBefore(c1, node);
            node.parentNode.replaceChild(c2, node);

            // Tell Knockout about the new nodes so that it can apply bindings to them
            return [c1, c2];
        }
    }
}
```

然后可以在视图中这样使用模板：

```html
<!-- template: 'some-template' -->
```

## ko.bindingProvider.instance.preprocessNode(node)

在绑定处理前每个DOM节点都会调用该函数。它可以修改、移除或替换节点`node`。任何新节点回被立刻插入到`node`之前，并且如果有节点被添加或`node`被移除，该函数必须返回html文档中`node`位置的新的节点数组。


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/custom-bindings.html)
