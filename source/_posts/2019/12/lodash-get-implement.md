---
title: Lodash get 三个基础实现版本
widgets: []
date: 2019-12-08
tags:
  - JavaScript
categories:
  - FE
---

接上篇，在 Vue 中你可以 这样 `a.b.c.d` 进行 `watch`，在 lodash 你也可以以同样的形式进行属性的读取，思索其中中源码实现才有了这篇文章
`_.get` 基本语法

```js
// 基本语法
_.get(obj, path, "defaultValue");

_.get(obj, "a.b[0].c", "defaultValue");

// 或者 path 以数组的形式
_.get(obj, ["a", "b", "0", "c"], "defaultValue");
```

path 支持两种形式的传值，数组和字符串，我们需要转成统一的数组形式，便于取值

<!-- more -->

```js
/**
 * @desc 转化 path成数组
 * @param {Array<String> | String} path
 * @returns {Array<String>} 路径数组
 */
function convPath(path) {
  if (!Array.isArray(path)) {
    return path.split(/[.\[\]]/).filter(v => v);
  }
  return path;
}
```

## 循环版 get

```js
/**
 * @desc loop 版 get
 * @param {{[key: String]: any}} obj
 * @param {Array<String> | String} path
 * @param {any} dft
 */
function getWithLoop(obj, path, dft) {
  let arrPath = convPath(path);
  let len = arrPath.length;
  let res = obj;
  let i = 0;
  while (i < len) {
    if (res[arrPath[i]] === undefined) {
      return undefined || dft;
    } else {
      res = res[arrPath[i]]; //更新暂存对象
    }
    i++;
  }
  return res;
}
```

## reduce 版 get

代码最简洁

```js
/**
 * @desc reduce 版 get
 * @param {{[key: String]: any}} obj
 * @param {Array<String> | String} path
 * @param {any} dft
 */
function getWithReduce(obj, path, dft) {
  let arrPath = convPath(path);
  return (
    arrPath.reduce((acc, cur) => {
      return (acc || {})[cur];
    }, obj) || dft
  );
}
```

## 递归版 get

```js
/**
 * @desc 递归版 get
 * @param {{[key: String]: any}} obj
 * @param {Array<String> | String} path
 * @param {any} dft
 */
function getWithRecursive(obj, path, dft) {
  let arrPath = convPath(path);
  if (arrPath.length <= 1) {
    return (obj ? obj[arrPath[0]] : undefined) || dft;
  } else {
    return getWithRecursive(obj[arrPath[0]], arrPath.slice(1), dft);
  }
}
```

## 测试

随意写的测试，没怎么考虑 edge case，主要是为了了解 get 函数的实现思路

```js
let testObj = {
  a: {
    b: {
      c: "get it",
      d: [{}, { e: "get it" }]
    }
  }
};

let logLine = title =>
  console.log(`---------------------------${title}---------------------------`);

logLine("loop");
console.log(
  getWithLoop(testObj, ["a", "b", "d", "1", "e"], 2),
  getWithLoop(testObj, "a.b.d[1].e", 2),
  "get it"
);
console.log(
  getWithLoop(testObj, ["a", "b", "d", "0", "e"], "default"),
  getWithLoop(testObj, "a.b.d[0].e", "default"),
  "default"
);
console.log(
  getWithLoop(testObj, ["a", "b", "d", "0", "e"]),
  getWithLoop(testObj, "a.b.d[0].e"),
  undefined
);
logLine("loop");

logLine("reduce");
console.log(
  getWithReduce(testObj, ["a", "b", "d", "1", "e"], 2),
  getWithReduce(testObj, "a.b.d[1].e", 2),
  "get it"
);
console.log(
  getWithReduce(testObj, ["a", "b", "d", "0", "e"], "default"),
  getWithReduce(testObj, "a.b.d[0].e", "default"),
  "default"
);
console.log(
  getWithReduce(testObj, ["a", "b", "d", "0", "e"]),
  getWithReduce(testObj, "a.b.d[0].e"),
  undefined
);
logLine("reduce");

logLine("Recursive");
console.log(
  getWithRecursive(testObj, ["a", "b", "d", "1", "e"], 2),
  getWithRecursive(testObj, "a.b.d[1].e", 2),
  "get it"
);
console.log(
  getWithRecursive(testObj, ["a", "b", "d", "0", "e"], "default"),
  getWithRecursive(testObj, "a.b.d[0].e", "default"),
  "default"
);
console.log(
  getWithRecursive(testObj, ["a", "b", "d", "0", "e"]),
  getWithRecursive(testObj, "a.b.d[0].e"),
  undefined
);
logLine("Recursive");
```
