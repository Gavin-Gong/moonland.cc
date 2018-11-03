---
title: 装饰器学习
widgets: []
tags:
  - JavaScript
categories:
  - FE
---

## 前言

一直觉得装饰器的写法有种蜜汁好感和好奇，例如 `@component`或者`@connect(x, y)`。

装饰器在 React 和 Angular 中很常见，因为这两个框架很强调类，而装饰器的作用范围正是类和类成员，来看装饰器提案的一句话

> Decorators make it possible to annotate and modify classes and properties at design time.

指出了装饰器作用对象，注意到最后三个单词 `at design time`, 再抄袭一句话

> 装饰器对类的行为的改变，是代码编译时发生的，而不是在运行时。这意味着，装饰器能在编译阶段运行代码。也就是说，装饰器本质就是编译时执行的函数

<!--more-->

装饰器对类的修改是一次性的，而且是在代码编译阶段完成装饰
看一个类装饰器

```js
export const decorator = （target, name, descriptor) => {
  console.log("gggg");
  target.useDecorator = true;
};

@decorator // 装饰的时候立即输出 gggg， 在实例化Test得时候不会再输出了
class Test {}
```

以上包含了装饰器的基本使用，所谓装饰器本质上就是一个通过`@`调用的特殊函数, 这个函数包括三个参数 target(类), name(key), descriptor(属性描述), 在作为*类装饰器*使用时没有 name 和 descriptor， 另外 descriptor 包含以下属性

```js
descriptor: {
   value: <value>, // 其实就是要装饰的成员
   writable: true,
   enumerable: false,
   configurable: true
 }
```

## babel 环境配置

除了 env 之外 还需要安装以下依赖

> yarn add -D babel-plugin-transform-decorators-legacy

```js
{
  "presets": ["env"],
  "plugins": ["transform-decorators-legacy"]
}
```

## 类装饰器

对一个类进行装饰， 可以修改类的原型链

```js
// 添加类的装静态性
export const decorator = target => {
  target.useDecorator = true;
};

// 使用方法
@decorator
class Test {}
```

```js
// 修改原型链 -》 对类的每个实例都器作用
export const decorator = target => {
  target.prototype.useDecorator = true;
};
```

## 类成员装饰器

```js
// 标记某个类成员为只读
export const readonly = (target, name, descriptor) => {
  descriptor.writable = false;
  return descriptor;
};

class Test {
  @readonly
  say() {}
}
```

```js
// 对类方法进行装饰， 自定义参数 输出日志信息
export const log = text => {
  return (target, name, descriptor) => {
    const oldFn = descriptor.value;
    descriptor.value = (...args) => {
      console.log(text);
      return oldFn(...args);
    };
    return descriptor;
  };
};

class Test {
  @log("gg")
  say() {}
}
```

## TL; DR

总的来说

- 装饰器的是一个能够被`@`调用的特殊*包装*函数，
- 装饰器只作用于*类*和*类成员*
- 装饰器函数能拿到对应的类或者对应的属性以及它的值和 descriptor 进行修改， 达到装饰的目的
- 装饰器对类成员的改变只在代码编译时发生

## 引用链接

[装饰器提案](https://github.com/wycats/javascript-decorators)

[ES6 入门](http://es6.ruanyifeng.com/#docs/decorator)
