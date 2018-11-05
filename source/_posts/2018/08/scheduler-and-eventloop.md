---
title: Rxjs Scheduler 下的 eventloop
date: 2018-08-02
tags:
  - RxJs
  - Event Loop
categories:
  - FE
widgets: []
---

本文将简单介绍 event loop 在 RxJS 中运用. 偏重于 RxJS Scheduler 而不是 event loop

## event loop

event loop 是一种任务调度机制, 用来处理异步任务, 在 JavaScript 单线程语言中, 在同一时间内只能只能处理一件事, 当我们遇到需要处理异步的情况, 不能让异步操作阻塞线程, 我们可以让异步任务先搁置, 先执行同步任务, 等异步时间来临再来执行异步任务, 这就是 event loop.

JavaScript 在执行过程中, 变量与对象会存入对中, 另外还会维护一个任务队列(包括宏任务队列, 微任务队列), 执行函数会被推入栈中, 一旦函数执行完毕就会从函数中推出, 一旦整个执行栈被清空, 就开始处理任务队列, 按照一定逻辑将任务推入执行栈中执行, 执行过程中可能会往任务队列中添加其他任务,微任务具有高优先级, 在单个宏任务的切换中间会检查执行整个微任务队列, 再去执行下一个宏任务

<!--more-->

event loop 是优化性能的手段之一，每个 eventloop 由三个阶段构成：执行一个 Macrotask，然后执行 Microtask 队列，然后执行 ui render (浏览器会自行判断是否需要进行 ui render, 也就是说 ui render 不是必须的). 也就是说一次 event loop 只会在最后执行零次或者一次 ui render. 避免多次的 ui render 无疑是有效提高 web 应用性能的方法, 所以我们把导致 ui render 的操作集中放到一个 Macrotask, 你也可以一个或者一队 Microtask 中, 最终只会导致一次渲染.

Macrotask 包括以下情形

- I/O
- UI rendering
- timer(setTimeout, setInterval) 等
- ajax
- events (绑定的事件)

Microtask 包括以下情形

- then (Promise)
- messageChannel
- MutationObersve
- Object.observe

## Scheduler

Scheduler 是什么?
Scheduler 就是调度器的意思, RxJS 中的 Scheduler 就是从细微角度控制数据流的推送时机的. 为什么说从细微角度控制推送时机? 先来看看 RXJS 中的四种 Scheduler 类型

- queue
- asap (Micro Task)
- async (Macro Task)
- animationFrame

queue 以同步的形式按照顺序执行, 上一个任务结束才会执行下一个任务. 优先于 event loop 执行

asap 是 As soon as possible 的缩写, 顾名思义是个优先级很高的 Scheduler. 其实是基于 Micro Task 的调度, 所以优先级很高,更加深入一点, 代码内部其实是用 Promise 实现的.

async Scheduler 其实是基于 Macro Task 的调度器, 代码内部其实是基于 setInterval 实现的, 所以它的优先级比前两个都低

animationFrame Scheduler 是基于 requestAnimationFrame API, 也是异步的

看一组简单的练习

```js
// 引入三种 scheduler
import { queueScheduler, asapScheduler, asyncScheduler } from "rxjs";

asyncScheduler.schedule(() => console.log("async", 1));
asapScheduler.schedule(() => console.log("asap", 2));
queueScheduler.schedule(() => console.log("queue", 3));

// 输出结果
// queue 3
// asap 2
// async 1

// 其实等同于以下代码输出
setTimeout(() => console.log("async", 1), 0);
Promise.resolve().then(() => console.log("asap", 2));
console.log("async", 1);
```

验证一下 queue 的队列形式,

```js
console.log(1); // 同步输出 1
// 将 foo 推入队列,  发现只有 foo 这个任务 开始执行 foo
queueScheduler.schedule(function foo() {
  queueScheduler.schedule(function bar() {
    console.log(2);
  }); // 推入队列, 等待 foo 执行完毕之后开始执行
  console.log(3); // 同步输出
  queueScheduler.schedule(function zoo() {
    console.log(4);
  }); // 推入队列, 等待 bar 执行完毕之后开始执行
});
console.log(5); // 同步输出

// 输出 1, 3, 2, 4, 5
```

### Scheduler 操作符分类

默认使用 queue

- of
- from
- range
- ...

默认使用 async

- timer
- interval
- ...

默认使用 asap

### observeOn

observeOn 会重新调度通知的发射(其实是改变 next, error, complete 的执行), 但是并不会改变原 Observable 的行为

```js
import { of, asapScheduler } from "rxjs";

import { observeOn, tap } from "rxjs/operators";

of(99)
  .pipe(
    tap(v => console.log("tap", v)),
    observeOn(asapScheduler)
  )
  .subscribe(v => console.log(v));

Promise.resolve().then(() => console.log(2));
console.log(1);
// 输出内容
// tap 99
// 1
// 99
// 2
```

`of` 操作符是 `queueScheduler` 默认是同步操作, 在使用 `observeOn(asapScheduler)` 之后并不会改变原 `Observable` 的行为, 因此 `tap 99` 是最先输出的, 然后输出 `1`, `subscribe` 和 `promise` 都是 Microtask, 但是 `subscribe` 先一步, 所以先输出 `99`, 然后再由 `Promise` 输出 `2`.

可以尝试将 `asapScheduler` 切换为 `asyncScheduler`, 然后发现输出结果如下

```
tap 99
1
2
99
```

### subscribeOn

`subscribeOn` 会修改原 `Observable` 的执行, 另外, `subscribeOn` 和 `observeOn` 本质是一个操作符, 从它们的引用方式就可以看出来, 它们只对上一个 `Observable` 产生作用, 来看一下代码

```js
of(99)
  .pipe(
    tap(v => console.log("tap", v)),
    subscribeOn(asyncScheduler),
    merge(of(98))
  )
  .subscribe(v => console.log(v));

Promise.resolve().then(() => console.log(2));
console.log(1);

// 输出
// 98
// 1
// 2
// tap 99
// 99
```

不出意外, `subscribeOn`, 修改了源 `Observable`, 导致 第一个`of` 变成了 `asyncScheduler`, 导致 `tap` 和 `subscribe` 的输出都延后了. 但是之后得 `merge(of(98))` 并没有受到 `subscribeOn`, 所以依然是同步输出

## 参考链接

- https://www.youtube.com/watch?v=X_RnO7KSR-4
- https://www.youtube.com/watch?v=AL8dG1tuH40
- https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
- https://www.404forest.com/2017/07/18/how-javascript-actually-works-eventloop-and-uirendering/#4-requestAnimationFrame-callback-%E7%9A%84%E6%89%A7%E8%A1%8C%E6%97%B6%E6%9C%BA

- https://staltz.com/primer-on-rxjs-schedulers.html
