---
author: wukn
catalog: true
date: 2016-11-12T00:00:00Z
tags:
- KnockoutJS
title: '[译][KnockoutJS] Bindings - Work with form fields'
url: /2016/11/12/knockoutjs-bindings-work-with-form-fields/
---

> "KnockoutJS文档官网：http://knockoutjs.com/documentation/introduction.html"

<!--more-->

## click

`click`用于为DOM元素添加点击`click`事件。通常用于`button`、`input`和`a`标签，但实际上可用于任何DOM元素。
参数是想绑定到元素`click`事件的函数。

可以绑定任意js函数（不是必须为视图模型的函数），如`click: someObject.someFunction`绑定对象`someObject`的`someFunction`函数。

```html
<div>
    You've clicked <span data-bind="text: numberOfClicks"></span> times
    <button data-bind="click: incrementClickCounter">Click me</button>
</div>

<script type="text/javascript">
    var viewModel = {
        numberOfClicks : ko.observable(0),
        incrementClickCounter : function() {
            var previousCount = this.numberOfClicks();
            this.numberOfClicks(previousCount + 1);
        }
    };
</script>
```

注1：向函数传递参数'当前项'

当触发点击事件调用绑定的函数时，ko会将当前模型值作为第一个参数传递。例如列表项点击时需要知道是哪个列表项点击的。

注意，如果是在一个嵌套的绑定上下文里（如`foreach`或`with`），但绑定的函数是在根视图模型或其他父上下文中时，需要使用`$parent`或`$root`前缀。

```html
<ul data-bind="foreach: places">
    <li>
        <span data-bind="text: $data"></span>
        <button data-bind="click: $parent.removePlace">Remove</button>
    </li>
</ul>

 <script type="text/javascript">
     function MyViewModel() {
         var self = this;
         self.places = ko.observableArray(['London', 'Paris', 'Tokyo']);

         // The current item will be passed as the first parameter, so we know which place to remove
         self.removePlace = function(place) {
             self.places.remove(place)
         }
     }
     ko.applyBindings(new MyViewModel());
</script>
```

注2：访问事件对象或传递更多参数

有些场景需要访问点击事件关联的DOM事件对象。ko将事件作为第二个参数传递。

```html
<button data-bind="click: myFunction">
    Click me
</button>

 <script type="text/javascript">
    var viewModel = {
        myFunction: function(data, event) {
            if (event.shiftKey) {
                //do something different when user has shift key down
            } else {
                //do normal action
            }
        }
    };
    ko.applyBindings(viewModel);
</script>
```

如果要传递更多的参数，一种方法是将事件函数包装到一个函数字面量中，并传递额外的参数。

```html
<button data-bind="click: function(data, event) { myFunction('param1', 'param2', data, event) }">
    Click me
</button>
```

如果想避免使用函数字面量，可以使用`[bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)`方法，

```html
<button data-bind="click: myFunction.bind($data, 'param1', 'param2')">
    Click me
</button>
```

注3：允许执行点击的默认行为

默认情况下，ko会阻止点击事件的默认行为。例如，在`a`标签上使用`click`，浏览器只会触发绑定的事件函数，而不会导航到`a`的`href`。如果需要默认的点击行为也触发，只需要在点击事件函数中`return true`。

注4：阻止事件冒泡

默认情况下，ko允许点击事件冒泡到更高层的事件处理中。例如，一个元素和其父元素都有点击事件，那么它们的点击事件都会触发。可以使用额外的值为`false`的`clickBubble`绑定来阻止事件冒泡。

```html
<div data-bind="click: myDivHandler">
    <button data-bind="click: myButtonHandler, clickBubble: false">
        Click me
    </button>
</div>
```

注5：存在jQuery时，ko会使用juery来处理事件。可以使用`ko.applyBindings`选项让ko使用原生的事件处理。

```javascript
ko.options.useOnlyNativeEvents = true;

```

## event

event用于为DOM元素添加指定事件。可以绑定任何事件，如`keypress`、`mouseover`或`mouseout`。
绑定的参数应该是个属性名为事件名、属性值为事件处理函数的js对象。

使用`event { mouseover: someObject.someFunction }`可以绑定任何事件处理函数，可以不是视图模型的函数。

```html
<div>
    <div data-bind="event: { mouseover: enableDetails, mouseout: disableDetails }">
        Mouse over me
    </div>
    <div data-bind="visible: detailsEnabled">
        Details
    </div>
</div>

<script type="text/javascript">
    var viewModel = {
        detailsEnabled: ko.observable(false),
        enableDetails: function() {
            this.detailsEnabled(true);
        },
        disableDetails: function() {
            this.detailsEnabled(false);
        }
    };
    ko.applyBindings(viewModel);
</script>
```

注：事件处理函数的两个参数、如何传递更多参数、允许默认事件行为、阻止事件冒泡、使用原生的事件处理，于`click`相同。

## submit

submit用于添加DOM元素的`submit`事件。通常使用于`form`元素。

如果在`form`中使用`submit`绑定，浏览器会调用绑定的事件处理函数，而不会提交服务器。在事件处理函数中`return true`可像正常的表单一样提交服务器。

绑定的函数可以为视图模型外的函数。

调用事件树立函数时，ko将`form`元素作为参数传递。可以使用[jQuery Validation](https://github.com/jzaefferer/jquery-validation)进行`if ($(formElement).valid()) { /* do something */ }`UI层验证。

```html
<form data-bind="submit: doSomething">
    ... form contents go here ...
    <button type="submit">Submit</button>
</form>

<script type="text/javascript">
    var viewModel = {
        doSomething : function(formElement) {
            // ... now do something
        }
    };
</script>
```

注：为什么不是仅仅向`submit`按钮绑定`click`呢？

`submit`可以捕捉一些其他的表单提交方式，如在文本框中按下`enter`键。

## enable

enable用于设置DOM元素是否启用。通常用于`input`、`select`、`textarea`等表单元素。

参数为非bool值时会被尽量转换成bool处理。如`0`或`null`相当于`false`，`21`或非空数组相当于`true`。

```html
<p>
    <input type='checkbox' data-bind="checked: hasCellphone" />
    I have a cellphone
</p>
<p>
    Your cellphone number:
    <input type='text' data-bind="value: cellphoneNumber, enable: hasCellphone" />
</p>

<script type="text/javascript">
    var viewModel = {
        hasCellphone : ko.observable(false),
        cellphoneNumber: ""
    };
</script>
```

参数可以使用js表达式。

```html
<p>
    <input type='checkbox' data-bind="checked: hasCellphone" />
    I have a cellphone
</p>
<p>
    Your cellphone number:
    <input type='text' data-bind="value: cellphoneNumber, enable: hasCellphone" />
</p>

<script type="text/javascript">
    var viewModel = {
        hasCellphone : ko.observable(false),
        cellphoneNumber: ""
    };
</script>
```

## disable

disable与enable类似，只是值为`true`时禁用元素，值为`false`时启用元素。

## value

value用于将DOM元素的值与视图模型的属性绑定。通常用于`input`、`select`、`textarea`等表单元素。

当绑定的监控属性时，用户在关联的表单控件中编辑值时，视图模型的属性值跟着更新。视图模型的属性值变化时，表单控件中显示的值也会跟着更新。

如果传递的参数不是数字或字符串时，显示的值则相当于`yourParameter.toString()`。

对于`checkbox`和`radio button`，使用`checked`而不是`value`来读写其状态。

```html
<p>Login name: <input data-bind="value: userName" /></p>
<p>Password: <input type="password" data-bind="value: userPassword" /></p>

<script type="text/javascript">
    var viewModel = {
        userName: ko.observable(""),        // Initially blank
        userPassword: ko.observable("abc"), // Prepopulate
    };
</script>
```

ko会在表单控件失去焦点时（如`change`事件）更新视图模型中的值。可以用`valueUpdate`参数来指定触发更新的事件。

* `input`：当` <input>`或`<textarea>`的值变化时更新视图模型。注意，一些老版本的浏览器可能不支持该事件（IE8及以下）。
* `keyup`：用户释放按键时更新视图模型。
* `keypress`：用户按住按键时更新视图模型。与`keyup`不同的是，当用户按住按键时会重复更新。
* `afterkeydown`：只要用户开始按下按键便更新视图模型。这是通过捕捉浏览器的`keydown`事件并异步处理的。在某些移动浏览器上不支持。

注1：要想实时获得`input`或`textarea`的值，应使用`textInput`绑定，避免使用`valueUpdate`选项。

注2：下拉列表的使用（如`<select>`）

ko对下拉列表有个特殊的使用方法，结合`value`和`options`可以读写一个为任意js对象的值，而不只是字符串。详见`options`和`selectedOptions`绑定。

同样可以对`<select>`元素使用`value`绑定来替代`options`绑定。这种情况下，可以在html文档中直接写出`<option>`元素，或使用`foreach`、`template`来生成。甚至可以将其嵌套于`<optgroup>`。

通常对`<select>`使用`value`绑定是希望用关联的模型值来描述`select`的选中项。但是如果模型的值不在列表中呢？默认ko是用当前下拉列表的选中项覆盖模型的值从而避免模型和视图不同步。

如果希望视图模型能够拥有`select`中没有的值，可以指定`valueAllowUnset: true`，这样，当模型的值不存在于`select`中时，`<select>`将没有选中值。

```html
<p>
    Select a country:
    <select data-bind="options: countries,
                       optionsCaption: 'Choose one...',
                       value: selectedCountry,
                       valueAllowUnset: true"></select>
</p>

<script type="text/javascript">
    var viewModel = {
        countries: ['Japan', 'Bolivia', 'New Zealand'],
        selectedCountry: ko.observable('Latvia')
    };
</script>
```

上例中，如果没有启用`valueAllowUnset`，ko会用值`undefined`覆盖`selectedCountry`，与`Choose one...`选项匹配。

注3：更新监控属性和非监控属性的值

使用`value`来链接表单元素和监控属性时，ko会建立双向绑定（2-way binding）使双方相互影响。

使用`value`来链接表单元素和非监控属性（如普通字符串或js表达式）时，ko的做法是：

* 如果引用的是简单属性，如模型的常规属性，ko会将属性的值作为元素初始状态值，当表单元素被编辑，ko将其回写给属性。但是属性变化时不会更新表单元素（因为是非监控属性）。这时是单向绑定（1-way binding）。
* 如果引用的不是简单属性，如函数的返回值等，ko会将该值作为元素初始状态值，但是当表单元素被编辑时，值不会被回写。这时是一次性绑定（one-time-only value setter）。

```html
<!-- Two-way binding. Populates textbox; syncs both ways. -->
<p>First value: <input data-bind="value: firstValue" /></p>

<!-- One-way binding. Populates textbox; syncs only from textbox to model. -->
<p>Second value: <input data-bind="value: secondValue" /></p>

<!-- No binding. Populates textbox, but doesn't react to any changes. -->
<p>Third value: <input data-bind="value: secondValue.length > 8" /></p>

<script type="text/javascript">
    var viewModel = {
        firstValue: ko.observable("hello"), // Observable
        secondValue: "hello, again"         // Not observable
    };
</script>
```

注4：同时使用`checked`和`value`

`checked`绑定用于chechbox`<input type='checkbox'>`和radio button`<input type='radio'>`。如果在使用`checked`时也使用`value`，那么`value`的作用和`checkedValue`相同。

注5：ko默认在jQuery存在时使用它来处理函数，可以在调用`ko.applyBindings`之前设置`ko.options.useOnlyNativeEvents = true;`来指定ko使用原生的事件处理。

## textInput

textnput用于链接`<input>`或`<textarea>`到视图模型的属性，提供双向更新。与`value`不同的是，它会在所用类型的用户输入时即时更新，包括autocomplete、drag-and-drop和clipboard事件。

```html
<p>Login name: <input data-bind="textInput: userName" /></p>
<p>Password: <input type="password" data-bind="textInput: userPassword" /></p>

<pre data-bind="text: ko.toJSON($root, null, 2)"></pre>

<script type="text/javascript">
    ko.applyBindings({
        userName: ko.observable(""),        // Initially blank
        userPassword: ko.observable("abc")  // Prepopulate
    });
</script>
```

## hasFocus

hasFocus用于链接视图模型属性和DOM元素的焦点状态（focus state）。它是双向绑定的。

```html
<input data-bind="hasFocus: isSelected" />
<button data-bind="click: setIsSelected">Focus programmatically</button>
<span data-bind="visible: isSelected">The textbox has focus</span>

<script type="text/javascript">
var viewModel = {
    isSelected: ko.observable(false),
    setIsSelected: function() { this.isSelected(true) }
};
ko.applyBindings(viewModel);
</script>
```

例：click-to-edit，使用`hasFocus`来切换编辑状态

```html
<p>
    Name:
    <b data-bind="visible: !editing(), text: name, click: edit">&nbsp;</b>
    <input data-bind="visible: editing, value: name, hasFocus: editing" />
</p>
<p><em>Click the name to edit it; click elsewhere to apply changes.</em></p>

<script type="text/javascript">
function PersonViewModel(name) {
    // Data
    this.name = ko.observable(name);
    this.editing = ko.observable(false);

    // Behaviors
    this.edit = function() { this.editing(true) }
}

ko.applyBindings(new PersonViewModel("Bert Bertington"));
</script>
```

## checked

checked用于复选框`<input type='checkbox'>`和单选框`<input type='radio'>`。

对于checkbox，如果参数值为bool型，那么值为true时设置`checked`，值为false时设置`unchecked`。如果参数是数组，那么checkbox的值在数组中设置`checked`，不在数组中设置`unchecked`。

```html
<p>Send me spam: <input type="checkbox" data-bind="checked: wantsSpam" /></p>

<script type="text/javascript">
    var viewModel = {
        wantsSpam: ko.observable(true) // Initially checked
    };

    // ... then later ...
    viewModel.wantsSpam(false); // The checkbox becomes unchecked
</script>
```

```html
<p>Send me spam: <input type="checkbox" data-bind="checked: wantsSpam" /></p>
<div data-bind="visible: wantsSpam">
    Preferred flavors of spam:
    <div><input type="checkbox" value="cherry" data-bind="checked: spamFlavors" /> Cherry</div>
    <div><input type="checkbox" value="almond" data-bind="checked: spamFlavors" /> Almond</div>
    <div><input type="checkbox" value="msg" data-bind="checked: spamFlavors" /> Monosodium Glutamate</div>
</div>

<script type="text/javascript">
    var viewModel = {
        wantsSpam: ko.observable(true),
        spamFlavors: ko.observableArray(["cherry","almond"]) // Initially checks the Cherry and Almond checkboxes
    };

    // ... then later ...
    viewModel.spamFlavors.push("msg"); // Now additionally checks the Monosodium Glutamate checkbox
</script>
```

对于radio button，只有value属性或checkedValue指定的值与参数值相等的radio button选项设置为`checked`。

```html
<p>Send me spam: <input type="checkbox" data-bind="checked: wantsSpam" /></p>
<div data-bind="visible: wantsSpam">
    Preferred flavor of spam:
    <div><input type="radio" name="flavorGroup" value="cherry" data-bind="checked: spamFlavor" /> Cherry</div>
    <div><input type="radio" name="flavorGroup" value="almond" data-bind="checked: spamFlavor" /> Almond</div>
    <div><input type="radio" name="flavorGroup" value="msg" data-bind="checked: spamFlavor" /> Monosodium Glutamate</div>
</div>

<script type="text/javascript">
    var viewModel = {
        wantsSpam: ko.observable(true),
        spamFlavor: ko.observable("almond") // Initially selects only the Almond radio button
    };

    // ... then later ...
    viewModel.spamFlavor("msg"); // Now only Monosodium Glutamate is checked
</script>
```

注：`checkedValue`参数

`checkedValue`用于替代`value`属性，这设置值为非字符串或动态设置值在时是很有用的。

```html
<!-- ko foreach: items -->
    <input type="checkbox" data-bind="checkedValue: $data, checked: $root.chosenItems" />
    <span data-bind="text: itemName"></span>
<!-- /ko -->

<script type="text/javascript">
    var viewModel = {
        items: ko.observableArray([
            { itemName: 'Choice 1' },
            { itemName: 'Choice 2' }
        ]),
        chosenItems: ko.observableArray()
    };
</script>
```

## options

options用于控制下拉列表（drop-down list）`<select>`或多选列表（multi-select list）`<select size='6'>`显示的选项。

option绑定的参数值应该是数组（或监控数组）。对于多选列表可以使用`selectedOptions`读写选中项，单选列表也可以使用`value`。

如果参数值是字符串数组，不需要指定额外参数。如果参数值是任意的js对象，则需要指定`optionsText`和`optionsValue`。

* optionsCaption：如果希望默认不选中任何项，但是单选列表是必须有选中项的，通常在列表前添加一个列表项如“请选择”（标题项 caption element）。使用optionsCaption，ko会在列表项前添加一个额外项，值为`undefined`，显示文本为optionsCaption的参数值，
* optionsText：参数值是任意的js对象时，需要用optionsText指定对象的哪个属性用来作为选择项的显示文本。可以使用js函数来对显示文本进行处理。
* optionsValue：与optionsText类似，参数值是任意的js对象时，需要用optionsValue指定对象的哪个属性用来作为选择项的值。可以使用js函数来对值进行处理。
* optionsIncludeDestroyed：当对数组元素进行标记删除时，option绑定不会显示标记删除的项，可以使用`optionsIncludeDestroyed: true`来指定显示删除的项。
* optionsAfterRender：可以使用`optionsAfterRender`在生成`option`插入列表之后执行自定义逻辑。回调函数包含两个参数，第一个是插入的`option`元素，第二个是绑定的数据项，或者标题项对应的`undefined`。
* selectedOptions：多选列表的项中项集合。
* valueAllowUnset：允许ko将模型的值设置为`<select>`中没有的值（相应的，`<select>`显示空白项）。

注：当options绑定更新`<select>`的选项时，ko会尽量保持用户的选中项（除非它们被移除了）。

例1：下拉列表

```html
<p>
    Destination country:
    <select data-bind="options: availableCountries"></select>
</p>

<script type="text/javascript">
    var viewModel = {
        // These are the initial options
        availableCountries: ko.observableArray(['France', 'Germany', 'Spain'])
    };

    // ... then later ...
    viewModel.availableCountries.push('China'); // Adds another option
</script>
```

例2：多选列表

```html
<p>
    Choose some countries you would like to visit:
    <select data-bind="options: availableCountries" size="5" multiple="true"></select>
</p>

<script type="text/javascript">
    var viewModel = {
        availableCountries: ko.observableArray(['France', 'Germany', 'Spain'])
    };
</script>
```

例3：使用任意js对象的下拉列表

```html
<p>
    Your country:
    <select data-bind="options: availableCountries,
                       optionsText: 'countryName',
                       value: selectedCountry,
                       optionsCaption: 'Choose...'"></select>
</p>

<div data-bind="visible: selectedCountry"> <!-- Appears when you select something -->
    You have chosen a country with population
    <span data-bind="text: selectedCountry() ? selectedCountry().countryPopulation : 'unknown'"></span>.
</div>

<script type="text/javascript">
    // Constructor for an object with two properties
    var Country = function(name, population) {
        this.countryName = name;
        this.countryPopulation = population;
    };

    var viewModel = {
        availableCountries : ko.observableArray([
            new Country("UK", 65000000),
            new Country("USA", 320000000),
            new Country("Sweden", 29000000)
        ]),
        selectedCountry : ko.observable() // Nothing selected by default
    };
</script>
```

例4：使用任意js对象的下拉列表，且显示经过函数计算的文本

```html
<!-- Same as example 3, except the <select> box expressed as follows: -->
<select data-bind="options: availableCountries,
                   optionsText: function(item) {
                       return item.countryName + ' (pop: ' + item.countryPopulation + ')'
                   },
                   value: selectedCountry,
                   optionsCaption: 'Choose...'"></select>
```

例5：渲染option元素的后处理（optionsAfterRender）

```html
<select size=3 data-bind="
    options: myItems,
    optionsText: 'name',
    optionsValue: 'id',
    optionsAfterRender: setOptionDisable">
</select>

<script type="text/javascript">
    var vm = {
        myItems: [
            { name: 'Item 1', id: 1, disable: ko.observable(false)},
            { name: 'Item 3', id: 3, disable: ko.observable(true)},
            { name: 'Item 4', id: 4, disable: ko.observable(false)}
        ],
        setOptionDisable: function(option, item) {
            ko.applyBindingsToNode(option, {disable: item.disable}, item);
        }
    };
    ko.applyBindings(vm);
</script>
```

## selectedOptions

selectedOptions用于控制多选列表的当前选中项。和`<select>`元素的`options`绑定同时使用。

如果绑定的不是监控数组，那么会是单向绑定，ko会将用户在界面选中的项更新到数组中。也就是说，始终可以读取到选中项。


```html
<p>
    Choose some countries you'd like to visit:
    <select data-bind="options: availableCountries, selectedOptions: chosenCountries" size="5" multiple="true"></select>
</p>

<script type="text/javascript">
    var viewModel = {
        availableCountries : ko.observableArray(['France', 'Germany', 'Spain']),
        chosenCountries : ko.observableArray(['Germany']) // Initially, only Germany is selected
    };

    // ... then later ...
    viewModel.chosenCountries.push('France'); // Now France is selected too
</script>
```

## uniqueName

uniqueName可以保证关联的DOM元素具有非空的name属性。

一般很少使用该绑定。例外情况有：
* jQuery Validation只验证有name的元素。
* IE6不允许选中没有name的radio button。

```html
<input data-bind="value: someModelProperty, uniqueName: true" />
```


---

参考资料：

[KnockoutJS documentation](http://knockoutjs.com/documentation/click-binding.html)
