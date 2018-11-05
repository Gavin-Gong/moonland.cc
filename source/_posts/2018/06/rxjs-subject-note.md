---
title: RxJS Subject 学习
date: 2018-06-20
widgets: []
tags:
  - RxJs
categories:
  - FE
---

## Subject

其实一个 `Observable` 可以被订阅多次, 但是并不共享一个流的数据,如下例

```javascript
let stream$ = Rx.of(1, 2, 3);
stream$.subscribe(r => console.log("a", r));

setTimeout(() => {
  stream$.subscribe(r => console.log("b", r));
}, 110);
```

输出结果

```
a 1
a 2
a 3
b 1
b 2
b 3
```

<!--more-->

他们是分开执行的或者说每次订阅都创建了一个新的执行, 如果我们想要创建一个流, 所有订阅者都共享流发出的数据, 我们需要引入一个新的概念`Subject`
`Subject`既是`Observable`也是`Observer`,这意味着 `Subject`既可以 emit 值, 也可以被订阅

```javascript
const stream$ = new Rx.Subject();

stream$.subscribe(r => console.log(r));
stream$.next("hello world");
```

而且会对内部的`Observer`进行组播`multicast`, 这意味着所有的`Observer`都是共享一条数据源

## BehaviorSubject

有时候我想在订阅的时候就获取当前最新的值, 而不是等到值更新了才知道, 这个时候需要用到 `BehaviorSubject`, 实例化时需要传递一个初始状态值

```javascript
let subject = new Rx.BehaviorSubject(0);

subject.subscribe(r => log(r)); // 立即输出 0
subject.next(1);
subject.next(2);

setTimeout(() => {
  subject.subscribe(v => log(v)); // 立即输出 2
}, 10000);
```

## ReplaySubject

顾名思义, 回放,在新订阅的时候 发送最后的指定个数元素

```javascript
let subject = new Rx.ReplaySubject(2); // 重放最后两个元素

subject.next(-2);
subject.next(-1);
subject.next(0);
subject.next(1);
subject.subscribe(r => log(r)); // 0 1
subject.next(2);
subject.next(3);
subject.next(4);
setTimeout(() => {
  subject.subscribe(r => log(r)); // 3 4
});
```

## AsyncSubject

AsyncSubject 会在 subject 结束后送出最后一个值

```javascript
const subject = new Rx.AsyncSubject();
subject.subscribe(r => log(r)); // 3
subject.next(1);
subject.next(2);
subject.next(3);
subject.complete();
setTimeout(() => {
  subject.subscribe(r => log(r)); // 10s 后输出 3
}, 10000);
```

## 简写操作

rxjs 内置了一些操作符对上面几个 subject 用法进行简写

### multicast & refCount

multicast 多播, refCount 引用计数代表当前 subject 上的订阅个数, 当个数为 0 时会停止发送, 大于 0, 开始发送数据
refCount 必须搭配 multicast 一起使用, 以下效果基本等同本文第一个例子,

```javascript
let s$ = Rx.interval(1).pipe(
  op.take(3),
  op.multicast(new Rx.Subject()),
  op.refCount()
);

s$.subscribe(r => log(r));

setTimeout(() => {
  s$.subscribe(v => log("b", v));
}, 0);
```

### publish

publish 其实是 `multicast(new Rx.Subject())`的语法糖,
依此类推

```javascript
multicast(new Rx.BehaviorSubject(1));
publishBehavior(1);

multicast(new Rx.ReplaySubject(1));
publishReplay(1);

multicast(new Rx.AsyncSubject());
publishLast(); // 注意有点不一致
```

### share

publish + refCount 又可以进一步简写为 share

```javascript
let s$ = Rx.interval(1).pipe(
  op.take(3),
  op.share()
);

s$.subscribe(r => log(r));

setTimeout(() => {
  s$.subscribe(v => log("b", v));
}, 0);
```
