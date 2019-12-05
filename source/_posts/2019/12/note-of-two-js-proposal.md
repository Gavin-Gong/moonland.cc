---
title: JS 最近进入 Stage 4 的两个提案
widgets: []
date: 2019-12-05
tags:
  - JavaScript
categories:
  - FE
---

今天刷推特的时候看到 TC39 的成员说 js 的两个语法糖提案进入 stage 4，最终板上钉钉，修成正果。

- Optional Chaining
- Nullish Coalescing
  其中 Optional Chaining 在这之前的 TS 3.7 中就被引入了

## Optional Chaining

在此之前使用我们通常获取一个多层级对象一个比较内层的属性，为了避免为空的情况不得不这样写

```JavaScript
let prop = a.b && a.b.c && a.b.c.d // 取到值或者赋值为 undefined
```

习惯了 Lodash 之后

```JavaScript
let prop = _.get(a, "b.c.d", undefined) // 可以自定义默认值
```

使用 Optional Chaining 之后

```JavaScript
let prop = a?.b?.c?.d
```

从提案中找到这么一句话

> If the operand at the left-hand side of the ?. operator evaluates to undefined or null, the expression evaluates to undefined. Otherwise the targeted property access, method or function call is triggered normally.

以 `?.` 为界限分为左边(LHS)和右边(RHS)，对 LHS 进行界定，如果为 null 或者 undefined 那么返回 undefined，如果既不为 null 也不为 undefined，继续向后进行 evalute。
babel 转译出来的结果相比提案案例更为直观

```javascript
let obj = {};
obj?.a;

// 转译为
("use strict");
var obj = {};
obj === null || obj === void 0 ? void 0 : obj.a;
```

除了取可选属性之外还可以取可选方法`a.?()`，对 a 进行 null 和 undefined 检验通过之后，如果 a 不是函数，还是会抛出错误

```
Uncaught TypeError: a is not a function
```

还可以 `a?.[++x]`
总结来说三种语法

```js
obj?.prop; // optional static property access
obj?.[expr]; // optional dynamic property access
func?.(...args); // optional function or method call
```

<!-- more -->

## Nullish Coalescing

在此之前，比较常用来设置默认值得方式为 `||`,

```js
let val = a.b || 1;
```

但是当本身已经有值为 0 或者 false 的情况就不再适用，看看 Nullish Coalescing 的定义

> If the expression at the left-hand side of the ?? operator evaluates to undefined or null, its right-hand side is returned.

与 Optional Chaining 相同的是，它也是使用 **undefined or null**进行界定的。于是我们之前的代码可以变为

```js
let val = a.b ?? 1;
```

## 总结

结合这两个提案可以写出这样的代码

```js
let age = info?.person?.age ?? 25;
```

对比使用 lodash 的话

```js
_.get(info, "person.age", 25);
```

我还是用 lodash 吧

## 参考

- https://github.com/tc39/proposal-nullish-coalescing

- https://github.com/tc39/proposal-optional-chaining

- https://babeljs.io/repl
