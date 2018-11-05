---
title: JavsScript 函数式学习手记
date: 2018-06-14
widgets: []
tags:
  - JavaScript
categories:
  - FE
---

## 高阶函数

- map
- filter
- some
- every
- reduce
- unary
- once
- pluck
- pick
- zip
- flatten
- merge
- takeLast
- uniq
- omit
- memoized
- compose
- pipe
- debounce
- throttle
  <!--more-->

### tap

一个没什么用但是调试很有用的函数, 获取函数参数后输出， 与此同时不影响 业务函数的执行

```js
const tap = v => fn => {
  console.log(v);
  typeof fn === "function" && fn(v);
};
```

### unary

unary === 一元的, 可以将多元函数转化为一元函数

```js
const unary = fn => {
  return fn.length === 1 ? fn : arg => fn(arg);
};
```

有意思的一个点 函数存在一个`length`属性来获取它的参数个数， 当然可变参数是会被忽略的存在。以前见过的一道题

```js
["1", "2", "3"].map(parseInt); // => [1, NaN, NaN]
```

其实相当于

```js
["1", "2", "3"].map((v, i) => parseInt(v, i)); // => [1, NaN, NaN]
```

当 parseInt 解析失败就会返回一个 NaN，但是 i 不传值一般会默认 10 进制， 详情见
链接[parseInt](http://devdocs.io/javascript/global_objects/parseint)
我们可以用 unary 转化一下让 parseint 只接受一个参数

```
['1', '2', '3'].map(unary(parseInt))
```

### once

只运行一次的函数， 我们需要用到闭包的特性来保存函数是否已经运行过的状态

```js
const once = fn => {
  let done = false;
  return () => {
    if (done) return;
    done = true;
    fn(arguments);
  };
};
```

使用

```js
once(say)(); // 我只说一次。。。
```

### memoized

当需要进行大量重复计算的时候可以缓存计算结果

```
const memoized = fn => {
  const cacheMap = {}
  return (arg) => cacheMap[arg] || (cacheMap[arg] = fn(arg))
}
```

## 科里化 和 偏应用

定义

> 是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术

通俗易懂版本

> 用闭包把参数保存起来，当参数的数量足够执行函数了，就开始执行函数

比如

```js
add(x, y, z);
// 变成下面这中形式的调用

const curryAdd = curry(add);
curryAdd(x)(y)(z);
```

实现一个能够进行上面这种变化的函数

```js
function curry(fn) {
  let len = fn.length; // 参数长度
  args = args || []; // 保存每次调用传递进来的参数
  return function judgeCurry(...arg) {
    if (args.length < len) {
      args = args.concat(arg);
      return judgeCurry;
    } else {
      return fn.apply(this, args);
    }
  };
}
```

去掉 args 这个局部变量

```js
function curry(fn) {
  let len = fn.length
  return function judgeCurry(..arg) {
    if (arg.length < len) {
      return (arg2) => judgeCurry.apply(null, arg.concat(arg2))
    } else {
      return fn.apply(this, arg)
    }
  }
}
```

把 len 也去掉, es6 写法

```js
const curry = (fn) =>
  judgeCurry(...arg) =>
    args.length >= fn.length ? fn(...args) : (arg) => judgeCurry(...args, arg)
```

### 使用场景

笼统来讲 curry 在缓存数据的时候非常有用 那么有哪些具体的场景可以使用到 curry 呢

#### React 绑定事件

React 绑定事件一般会用 `bind`或者`arrow function`，我们也可以借助 curry 实现

```jsx
<button onClick={curry(handleClick)(data)}>Click</button>
```

[def](https://zh.wikipedia.org/wiki/%E6%9F%AF%E9%87%8C%E5%8C%96)

[curry](https://segmentfault.com/a/1190000008248646)

[curry](https://github.com/mqyqingfeng/Blog/issues/42)

[curry](https://github.com/MrErHu/blog/issues/8)

[curry](https://juejin.im/post/5af13664f265da0ba266efcf)

### compose & pipe

compose 的定义

> 串联函数， 将前一个函数的结果作为后一个函数的参数进行传递
> 举个例子

```js
c(b(a(foo)));

// 变成下面这种形式的调用

compose(
  c,
  b,
  a
)(foo);

// 如果是 pipe 的话
pipe(
  a,
  b,
  c
)(foo);
```

我们可以看到 compose 是从右往左的调用顺序， pipe 是从左到右的数据流顺序， 尝试实现一个 compose 函数

```js
export const compose = (...args) => {
  let len = args.length;
  let start = len - 1;
  return (...args1) => {
    let result = args[start].apply(this, args1); // 初始参数
    let i = start;
    while (i--) {
      result = args[i].call(this, result);
    }
    return result;
  };
};
```

更加简洁的写法是利用 reduce 函数

```js
export const compose1 = (...funcs) => {
  if (funcs.length === 0) {
    return arg => arg;
  }
  if (funcs.length === 1) {
    return funcs[0];
  }
  return funcs.reduce((acc, fn) => (...args) => acc(fn(...args)));
};
```

或者

```js
export const composeN = (...fns) => v => {
  return fns.reverse().reduce((acc, fn) => fn(acc), v);
};
```

pipe 于 compsoe 函数的差异只在于数据流动的方向, 所以 pipe 可以写为

```js
export const pipe = (...fns) => v => {
  return fns.reduce((acc, fn) => fn(acc), v);
};
```

### 场景

compose, pipe 函数可以很好地将存在数依赖关系的一系列函数串起来， 便于理解阅读

## 函子 (functor)

函子是一个持有值的容器，是一个普通对象实现了 map 函数，在遍历每个对象值的时候生成一个新对象

### Maybe

Maybe  函子可以用于处理错误情况，在遇到错误情况的时候不至于中断执行。

```js
const MayBe = function(val) {
  this.val = val
}
MayBe.of = function(val) {
  return new MayBe(val)
}
}
MayBe.prototype.isNothing = function() {
  return (this.value === null || this.value === undefined)
}
MayBe.prototype.map = function(fn) {
  return this.isNothing() ? MayBe.of(null) : MayBe.of(fn(this.value))
}
```

例如，以下代码并不会报错

```js
Maybe.of(null)
  .map(v => v.toUpperCase())
  .map(v => v["data"]);
```

### Either

Maybe 函子也仅仅事能保证运行不出错，如果我想精细地处理运行过程中的错误呢， 这个时候就引出 Either 函子， 看看它的实现吧

```js
const Nothing = function (val) {
  thi s.value = val
}
Nothing.of = function (val) {
  return new Nothing(val)
}
Nothing.prototype.map = function () {
  return this //注意这里返回 this
}

const Some = function(val) {
  this.value = val
}
Some.of = function(val) {
  return new Some(val)
}
Some.prototype.map = function (fn) {
  return Some.of(fn(this.value))
}

export {
  Nothing,
  Some
}
```

实际运用，需要一个函数来处理

```js
let fetchData = () => {
  let res;
  try {
    res = Some.of(fetch("xxxx"));
  } catch (err) {
    res = Nothing.of({ msg: err.message });
  }
  return res;
};
```

[函数式术语](https://github.com/gnipbao/iblog/issues/13)
