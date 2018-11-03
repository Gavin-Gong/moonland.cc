---
title: 使用装饰器自动取消订阅 React 组件中的 RxJS 流
widgets: []
tags:
  - React
  - RxJs
categories:
  - FE
---

最近自己的一个项目使用了 RxJS 进行状态管理, 需要在组件中订阅流来获取全局状态， 但是在组件卸载的时候需要手动取消订阅流.
这样写起来有点繁琐，所以考虑用装饰器简化下代码

```typescript
import * as React from "react";
import { Subscription } from "rxjs";
import { action$, state$ } from "../store/store";

class Artist extends React.Component {
  sub: Subscription;
  handleClick() {
    setInterval(() => {
      action$.next((state: any) => state);
    }, 1000);
  }
  componentDidMount() {
    this.sub = state$.subscribe(v => {
      console.log("artist", v);
    });
  }
  componentWillUnmount() {
    console.log("artist unsubscribe");
    this.sub.unsubscribe();
  }
  render() {
    return (
      <div>
        <button onClick={() => this.handleClick()}>Click Me</button>
      </div>
    );
  }
}

export default Artist;
```

<!--more-->

我们先实现一个组件卸载的时候取消订阅所有流的装饰器

```typescript
import { Subscription } from "rxjs";

/**
 * @desc 取消订阅某个 React 组件上的所有的流
 */
export function unsubscribeAll(target: any) {
  const originHook = target.prototype.componentWillUnmount;
  target.prototype.componentWillUnmount = function() {
    // 遍历实例上的所有属性， 一旦该属性是 Subscription 的实例，就执行unsubscribe
    for (let prop in this) {
      const val = this[prop];
      if (val && val instanceof Subscription) {
        val.unsubscribe();
      }
    }
    originHook &&
      typeof originHook === "function" &&
      originHook.apply(this, arguments);
  };
}
```

使用方法

```typescript
@unsubscribeAll
class Artist extends React.Component {
  // ...
}
```

但是有时候我并不想取消所有的流，只是想取消订阅指定的流呢? 我们可以这样实现代码

```typescript
/**
 * @desc 取消订阅某个 React 组件上的指定流
 * @param blacklist 要取消订阅的流
 */
export function unsubscribe(...list: Array<string>) {
  return function(target: any) {
    const originHook = target.prototype.componentWillUnmount;
    target.prototype.componentWillUnmount = function() {
      // 遍历实例上的所有属性， 一旦该属性是 Subscription 的实例，就执行unsubscribe
      for (let prop in this) {
        const val = this[prop];
        // 取消订阅指定流
        if (list.indexOf(prop) > -1 && val && val instanceof Subscription) {
          val.unsubscribe();
        }
      }
      originHook &&
        typeof originHook === "function" &&
        originHook.apply(this, arguments);
    };
  };
}
```

使用方法

```typescript
@unsubscribe("subscription1", "subscription2")
class Artist extends React.Component {
  // ...
}
```

RxJS 取消订阅并非只有`unsubscribe`， `takeUtil`和`take`等也是可以的, 例如

```typescript
import * as React from "react";
import { Subscription, Subject } from "rxjs";
import { action$, state$ } from "../store/store";
import { takeUntil } from "rxjs/operators";

class Artist extends React.Component {
  sub: Subscription;
  unmount$ = new Subject();
  handleClick() {
    setInterval(() => {
      action$.next((state: any) => state);
    }, 1000);
  }
  componentDidMount() {
    this.sub = state$.pipe(takeUntil(this.unmount$)).subscribe(v => {
      console.log("artist", v);
    });
  }
  componentWillUnmount() {
    this.unmount$.next(); // 发出通知
  }
  render() {
    return (
      <div>
        <button onClick={() => this.handleClick()}>Click Me</button>
      </div>
    );
  }
}

export default Artist;
```

其实这样代码量还是不少，但是在订阅比较多的情况下共用一个 unmount$ 来取消其他相关订阅还是减少了不少代码量， 相比原始方案还是有优势， 相比装饰器就没太大的优势了。

## 其他

对 TypeScript 的类型还是不够熟练， 写成 AnyScript 了， 捂脸。

## 参考链接

[Automagically Unsubscribe in Angular](https://netbasal.com/automagically-unsubscribe-in-angular-4487e9853a88)

[RxJS: Don’t Unsubscribe](https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87)

> 写于 2018-07-10 晚
