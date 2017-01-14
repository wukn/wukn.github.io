---
layout:     post
title:      "[Angular2] Animations"
subtitle:   ""
date:       2017-01-14 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Angular2

---

> 看一看Angular2的动画系统。

在现代Web应用中，动画是很重要的一方面。设计良好的动画让UI不仅有趣，而且还易于使用。

Angular的动画系统遵循[Web Animations API](https://w3c.github.io/web-animations/)标准。只有[]版本比较新的部分浏览器](http://caniuse.com/#feat=web-animation)才支持。老的浏览器需要[web-animations.min.js](https://github.com/web-animations/web-animations-js)。

### 案例：在两个状态之间转换

我们可以基于模型的属性让一个元素在两种状态之间切换，就像这样：

![](/img/post/angular2/animations/animation_basic_click.gif)

动画是在`@Component`的元数据内定义的。在添加动画之前需要引入一些相关的函数：

```js
import {
  Component,
  Input,
  trigger,
  state,
  style,
  transition,
  animate
} from '@angular/core';
```

有了这些以后，可以在组件的元数据中定义一个`heroState`动画触发器（animation trigger）。使用动画在`active`和`inactive`两个状态之间切换。

```js
animations: [
  trigger('heroState', [
    state('inactive', style({
      backgroundColor: '#eee',
      transform: 'scale(1)'
    })),
    state('active',   style({
      backgroundColor: '#cfd8dc',
      transform: 'scale(1.1)'
    })),
    transition('inactive => active', animate('100ms ease-in')),
    transition('active => inactive', animate('100ms ease-out'))
  ])
]
```

使用`[@triggerName]`语法为元素添加动画：

```js
template: `
  <ul>
    <li *ngFor="let hero of heroes"
        [@heroState]="hero.state"
        (click)="hero.toggleState()">
      {{hero.name}}
    </li>
  </ul>
`,
```

这里为每个元素都添加了动画，属性值绑定的是表达式`hero.state`，它的值要么是`active`要么是`inactive`。

这样就完成了动画的添加。完整的代码如下：

```js
import {
  Component,
  Input,
  trigger,
  state,
  style,
  transition,
  animate
} from '@angular/core';
import { Heroes } from './hero.service';
@Component({
  moduleId: module.id,
  selector: 'hero-list-basic',
  template: `
    <ul>
      <li *ngFor="let hero of heroes"
          [@heroState]="hero.state"
          (click)="hero.toggleState()">
        {{hero.name}}
      </li>
    </ul>
  `,
  styleUrls: ['hero-list.component.css'],
  animations: [
    trigger('heroState', [
      state('inactive', style({
        backgroundColor: '#eee',
        transform: 'scale(1)'
      })),
      state('active',   style({
        backgroundColor: '#cfd8dc',
        transform: 'scale(1.1)'
      })),
      transition('inactive => active', animate('100ms ease-in')),
      transition('active => inactive', animate('100ms ease-out'))
    ])
  ]
})
export class HeroListBasicComponent {
  @Input() heroes: Heroes;
}
```

### 状态和切换

Angular的动画定义分为逻辑状态和状态之间的切换。

动画状态是代码中定义的一个字符串值，例如上面的`active`和`inactive`。状态的来源可以是对象的属性，也可以是一个方法计算出来的值。重要的是要能在组件的模板中读到这个值。

可以为动画的每个状态定义样式，当元素切换到某个状态时就使用相应的样式：

```js
state('inactive', style({
  backgroundColor: '#eee',
  transform: 'scale(1)'
})),
state('active',   style({
  backgroundColor: '#cfd8dc',
  transform: 'scale(1.1)'
})),
```

定义完状态后就要定义状态之间的切换，每个transition控制状态之间的切换时间：

```js
transition('inactive => active', animate('100ms ease-in')),
transition('active => inactive', animate('100ms ease-out'))
```

![](/img/post/angular2/animations/ng_animate_transitions_inactive_active.png)

如果多个transition的动画内容相同，可以将它们合并到同一个定义中：

```js
transition('inactive => active, active => inactive',
 animate('100ms ease-out'))
```

如果两个方向的转换的动画内容相同，还可以使用简写`<=>`：

```js
transition('inactive <=> active', animate('100ms ease-out'))
```

在动画的过程中添加样式，但是它不会覆盖元素的最终样式。可以在`transition`中以行内的方式定义中间样式。

```js
transition('inactive => active', [
  style({
    backgroundColor: '#cfd8dc',
    transform: 'scale(1.3)'
  }),
  animate('80ms ease-in', style({
    backgroundColor: '#eee',
    transform: 'scale(1)'
  }))
]),
```

#### 通配状态*

通配符*匹配任意的动画状态。例如：

* `active => *`表示元素从`active`切换到任意其他状态
* `* => *`任意的状态切换

![](/img/post/angular2/animations/ng_animate_transitions_inactive_active_wildcards.png)

#### void状态

有个特殊的状态`void`可以用于动画中。它表示元素没有在视图中这种状态，可能是它还没有添加到视图中，也可能是它已经被移除了。void状态对定义进入和离开动画很有用。

例如，`* => void`动画会在元素离开视图的时候起作用。

![](/img/post/angular2/animations/ng_animate_transitions_void_in.png)

通配符*也会匹配void状态。

### 案例：进入和离开动画

使用`void`和`*`可以定义元素进入和离开的动画。进入为`void => *`，离开为`* => void`。

![](/img/post/angular2/animations/animation_enter_leave.gif)

```js
animations: [
  trigger('flyInOut', [
    state('in', style({transform: 'translateX(0)'})),
    transition('void => *', [
      style({transform: 'translateX(-100%)'}),
      animate(100)
    ]),
    transition('* => void', [
      animate(100, style({transform: 'translateX(100%)'}))
    ])
  ])
]
```

注意上面void状态的样式是直接在transition中定义的，而不是单独定义一个`state(void)`。这样，进入和离开的动画就不一样，从左边进入，从右边离开。

这两种常见的动画有个别名：

```js
transition(':enter', [ ... ]); // void => *
transition(':leave', [ ... ]); // * => void
```

### 案例：从不同的状态进入和离开

结合之前的状态可以定义不同状态的元素的进入和离开动画。

* Inactive hero enter: `void => inactive`
* Active hero enter: `void => active`
* Inactive hero leave: `inactive => void`
* Active hero leave: `active => void`

![](/img/post/angular2/animations/animation_enter_leave_states.gif)

这样我们可以设计出更精细的动画：

![](/img/post/angular2/animations/ng_animate_transitions_inactive_active_void.png)

```js
animations: [
  trigger('heroState', [
    state('inactive', style({transform: 'translateX(0) scale(1)'})),
    state('active',   style({transform: 'translateX(0) scale(1.1)'})),
    transition('inactive => active', animate('100ms ease-in')),
    transition('active => inactive', animate('100ms ease-out')),
    transition('void => inactive', [
      style({transform: 'translateX(-100%) scale(1)'}),
      animate(100)
    ]),
    transition('inactive => void', [
      animate(100, style({transform: 'translateX(100%) scale(1)'}))
    ]),
    transition('void => active', [
      style({transform: 'translateX(0) scale(0)'}),
      animate(200)
    ]),
    transition('active => void', [
      animate(200, style({transform: 'translateX(0) scale(0)'}))
    ])
  ])
]
```

### 可作为动画的属性和单位

由于Angular的动画是基于Web Animations的，我们可以使用任何浏览器认为可作为动画（animatable）的属性。这包括位置、大小、转换、颜色、边框、等等。W3C维护了一个[可作为动画的属性](https://www.w3.org/TR/css3-transitions/#animatable-properties)列表。

例如，拥有数字值的位置属性，可以使用包含值和单位的字符串，如`50px`、`3em`、`100%`。如果没有单位，那么Angular默认为`px`。

### 自动属性计算

有时候一些样式的坐标属性值直到运行的时候才能够知道。例如元素根据它的内容和屏幕的大小决定它的宽度和高度。

![](/img/post/angular2/animations/animation_auto.gif)

这种情况下可以使用特殊的`*`属性值，这样就是在运行时计算出这个属性值并添加到动画中。

上面图片这个例子中，元素离开的动画，不管在离开之前元素的高度是多少，都从那个高度切换到0。

```js
animations: [
  trigger('shrinkOut', [
    state('in', style({height: '*'})),
    transition('* => void', [
      style({height: '*'}),
      animate(250, style({height: 0}))
    ])
  ])
]
```

### 动画的定时

每个transition可以定义三种时间属性，持续时间（duration）、延迟时间（delay）和缓冲函数（easing function）。它们是在一个transition的定时字符串中。

#### 持续时间

持续时间控制动画从开始到结束的时长。有三种定义方式：

* 纯数字，以毫秒为单位：`100`
* 以毫秒为单位的字符串：`'100ms'`
* 以秒为单位的字符串：`'0.1s'`

#### 延迟时间

延迟时间控制从动画触发到动画开始的时长。可以加在定义持续时间的字符串后面，格式和持续时间一样。

* 等待100ms然后持续200ms：`'0.2s 100ms'`

#### 缓冲

[擦除函数](http://easings.net/)控制动画运行时候的加速和减速。例如`ease-in`，会让动画在开始的时候比较慢，然后逐渐加速。可以将它作为第三个值放在持续时间和延迟时间后面，如果没有延迟时间，那就把它放在第二个值。

* 延迟100ms，然后运行200ms，带缓冲：`'0.2s 100ms ease-out'`
* 运行200ms，带缓冲：`'0.2s ease-in-out'`

#### 案例

下面这个例子中，进入和离开动画的持续时间都是200ms，但是缓冲特效不一样。

![](/img/post/angular2/animations/animation_timings.gif)

```js
animations: [
  trigger('flyInOut', [
    state('in', style({opacity: 1, transform: 'translateX(0)'})),
    transition('void => *', [
      style({
        opacity: 0,
        transform: 'translateX(-100%)'
      }),
      animate('0.2s ease-in')
    ]),
    transition('* => void', [
      animate('0.2s 10 ease-out', style({
        opacity: 0,
        transform: 'translateX(100%)'
      }))
    ])
  ])
]
```

### 多步骤动画

动画关键帧（keyframes）将一个简单的切换变成了有一个或多个中间样式的复杂的动画。

对于每个关键帧，制定一个偏移量（offset）制定在哪个点应用关键帧。offset的值介于0和1之间。0就是动画的开始，1就是动画的结束。

例如，使用关键帧为进入和离开动画添加弹跳效果。

![](/img/post/angular2/animations/animation_multistep.gif)

```js
animations: [
  trigger('flyInOut', [
    state('in', style({transform: 'translateX(0)'})),
    transition('void => *', [
      animate(300, keyframes([
        style({opacity: 0, transform: 'translateX(-100%)', offset: 0}),
        style({opacity: 1, transform: 'translateX(15px)',  offset: 0.3}),
        style({opacity: 1, transform: 'translateX(0)',     offset: 1.0})
      ]))
    ]),
    transition('* => void', [
      animate(300, keyframes([
        style({opacity: 1, transform: 'translateX(0)',     offset: 0}),
        style({opacity: 1, transform: 'translateX(-15px)', offset: 0.7}),
        style({opacity: 0, transform: 'translateX(100%)',  offset: 1.0})
      ]))
    ])
  ])
]
```

关键帧的offset参数可以忽略。如果忽略那么就是每个关键帧之间时间间隔相等。例如，三个没有offset的关键帧将会接收到0、0.5和1作为offset。

### 并行动画组

前面看到过怎样在同一个时刻添加多个样式属性，就是将它们放到同一个`style()`中。

也许你想为一个动画添加并行作用的不同的定时效果。例如，你想用两个CSS属性做动画，但是每个使用不同的缓冲函数。

这种场景可以使用动画组（group）。例如，为进入和离开动画使用两个不同的定时效果，它们会并行的作用在同一个元素上，但是是分开执行的。

![](/img/post/angular2/animations/animation_groups.gif)

```js
animations: [
  trigger('flyInOut', [
    state('in', style({width: 120, transform: 'translateX(0)', opacity: 1})),
    transition('void => *', [
      style({width: 10, transform: 'translateX(50px)', opacity: 0}),
      group([
        animate('0.3s 0.1s ease', style({
          transform: 'translateX(0)',
          width: 120
        })),
        animate('0.3s ease', style({
          opacity: 1
        }))
      ])
    ]),
    transition('* => void', [
      group([
        animate('0.3s ease', style({
          transform: 'translateX(50px)',
          width: 10
        })),
        animate('0.3s 0.2s ease', style({
          opacity: 0
        }))
      ])
    ])
  ])
]
```

一个组作用于元素的转换和宽度，另一个组作用于调整元素的透明度。

### 动画回调

动画开始和结束的时候会触发回调。

在关键帧那个例子中，我们可以有一个名为`@flyInOut`的触发器。可以这样添加回调函数：

```js
template: `
  <ul>
    <li *ngFor="let hero of heroes"
        (@flyInOut.start)="animationStarted($event)"
        (@flyInOut.done)="animationDone($event)"
        [@flyInOut]="'in'">
      {{hero.name}}
    </li>
  </ul>
`,
```

回调函数接受一个`AnimationTransitionEvent`，它包含一些有用的属性，如`fromState`、`toState`和`totalTime`。

Those callbacks will fire whether or not an animation is picked up.

---

参考资料：

[Angular2 Documentation - Animations](https://angular.io/docs/ts/latest/guide/animations.html)
