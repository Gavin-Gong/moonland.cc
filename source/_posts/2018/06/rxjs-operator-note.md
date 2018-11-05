---
title: RxJS 操作符笔记
date: 2018-06-16
widgets: []
tags:
  - RxJs
categories:
  - FE
---

> RxJS 6

### 操作符

一些常用的操作符

- of
- from
- first
- last
- tap
- interval
- timer
- forkJoin
- filter
- map
- switchMap
- scan
- takeWhile
- takeUtil
- take
- concat
- throttle
- debounce
- merge

#### of

将数字转化为 `Observable`

```javascript
Rx.of("1", "2").subscribe(v => console.log(v));
// 输出 1, 2
```

<!--more-->

#### concat

前面的流执行完成在执行后面的流

```javascript
Rx.from(["x", "y", "z"])
  .pipe(concat(Rx.timer(10000), Rx.of("3", "2")))
  .subscribe(res => log("concat: emit", res));
```

输出

```
concat: emit x
concat: emit y
concat: emit z
concat: emit 0
concat: emit 3
concat: emit 2
```

#### forkJoin

类似`Promise.all`, 同时执行多个流, 在所有流执行完成之后传递一个数据数组.

```javascript
Rx.forkJoin(fetchData(1), fetchData(2)).subscribe(r =>
  console.log("promise: emit", r)
);
```

#### mergeMap(flatMap)

流相乘

```javascript
Rx.of("a", "b", "c")
  .pipe(
    mergeMap(x => {
      return Rx.interval(1000).pipe(
        map(i => {
          return i + x;
        })
      );
    })
  )
  .subscribe(r => log(r));
```

输出

```
0a
0b
0c
1a
1b
1c
2a
2b
2c
......
```

#### switchMap

当接收到新的流发出的信息, 会取消订阅前一个流, 在进行多次点击的情况并不会执行多次 log 而是输出最新的流返回的数据

```javascript
Rx.fromEvent(document, "click")
  .pipe(switchMap(e => Rx.from(fetchData)))
  .subscribe(res => log(res));
```

#### takeUtil

用 takeUntil 接收一个通知流, 来控制另外一个流的结束,

```javascript
Rx.interval(1)
  .pipe(takeUntil(Rx.timer(10)))
  .subscribe(r => log(r));
```

输出

```
1
2
3
// 10毫秒后结束流不再输出
```

#### takeWhile

类似 js 中的`while`语句, 当满足 takeWhile 的条件才会执行, 否则跳出循环结束流

```javascript
Rx.from(["iris", "diana", "appolo", "luna"])
  .pipe(op.takeWhile(v => v !== "appolo"))
  .subscribe(r => log(r));
```

输出

```
iris
diana
// 结束
```

#### buffer

将一个流的数据缓存到 buffer 中, 等待另一个流通知结束, 再将 buffer 的数据以数组的形式 emit 出来

```javascript
Rx.interval(0)
  .pipe(buffer(Rx.timer(10000)))
  .subscribe(r => log(r));
// 10s 后输出一大堆数据
```

#### debounceTime

防抖, 常用于持续触发的事件, 滚动等, 以下例子, 在没有数据输出, 等待 1s 后才会输出最后一条数据

```javascript
Rx.interval(0)
  .pipe(
    takeUntil(Rx.timer(1000)), // 一秒后结束流
    debounceTime(1000) // 输出最后一个值
  )
  .subscribe(r => log(r));
```

#### throttleTime

节流, 以下例子, 每隔 1s 取一个数据, 所以输出的数字并不会连续

```javascript
Rx.interval(0)
  .pipe(throttleTime(1000))
  .subscribe(r => log(r));
```

#### scan

类似 js 中的`reduce`, 将累计值 和当前流传递发出的值进行 `相加`, 然后保存到累计值

```javascript
Rx.of({ name: "xx", state: "inited" })
  .pipe(scan((acc, cur) => ({ ...acc, ...cur }), { state: "@@initState" }))
  .subscribe(r => log(r));
```

### 学习资料

[30 day](https://jiayisheji.gitbooks.io/30-days-proficient-in-rxjs/content/chapter1.html)

[20 example](https://angularfirebase.com/lessons/rxjs-quickstart-with-20-examples/)
