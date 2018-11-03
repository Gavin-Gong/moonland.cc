# 使用 RxJS 进行状态管理

Rxjs 中 BehaviorSubject 能起到多播的作用, 在处理个个状态对应多个页面的时候尤其有用。
而且 BehaviorSubject 在订阅的时候能立马获取当前值而不是等待流发出新值，保证了状态更新不会出现 断流。
scan 操作符对状态进行 累计计算

```typescript
import { BehaviorSubject } from "rxjs";
import { scan, tap } from "rxjs/operators";

interface IState {
  [key: string]: any;
}
type IReducer = (curState: IState) => IState;

const initState: IState = {};

const store$$ = new BehaviorSubject((state: IState) => state);

export const state$ = store$$.pipe(
  tap(v => console.log(v)),
  scan((state: IState, op: IReducer) => op(state), initState),
  tap(v => console.log(v))
);
export const action$ = store$$;
```

<!--more-->

基本调用

```typescript
// 订阅状态更新并执行相应的操作， 不要忘记进行 unsubscribe
state$.subscribe(r => {
  console.log(r);
});

// Reduce 函数， 产生新的状态
action$.next(state => ({
  ...state,
  name: "xx"
}));

// 或者 pipe
action$.pipe(mapTo((s: any) => ({})));
```

不难发现有以下缺点：

- subscribe 状态更新之后需要手动 unsubscribe
