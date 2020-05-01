---
title: TS 一些工具泛型的使用及其实现(续)
date: 2018-07-20
tags:
  - TypeScript
  - Generic
categories:
  - FE
widgets: []
---

之前写了一篇 `TS 一些工具泛型的使用及其实现`, 但是一直没怎么使用 TS，回首看文章，发现自己都看不懂了。
期间内 TS 也有一些变化，所以这一篇将会承接上篇文章，分析解读更多的工具泛型，主要来自 [utility-types](https://github.com/piotrwitek/utility-types)项目的源码。
阅读本流水账需要对 TS 中的以下东西有所了解

- extends
- keyof
- in
- infer
- &
- |
- ?
- -?
- +?
- never
- unkown
- any
- readonly
- void

----

## 正文

### ArrayElement

提取数组成员类型,
一个思路是 用 extends 限制数组类型, 然后用数组 key 类型为 number 的特性取出其属性类型

``` ts
type ArrayElement<T extends readonly unknown[]> = T[number];
```

第二种写法的核心思路就是用 infer 来隐射 数组的属性类型

``` ts
type ArrayElement<A> = A extends readonly (infer T)[] ? T : never
```

### Exclude & Extract vs. Diff & Filter

TS 内置类型定义涵盖了 Exclude & Extract, 但是在它的官方文档又给出了另外的名字

``` ts
type Diff<T, U> = T extends U ? never : T;
type Filter<T, U> = T extends U ? T : never;
```

就类型定义的代码而言，Exclude === Diff, Extract === Filter，蜜汁操作

### NonNullable

从类型 T 中排除 null 和 undefined

``` ts
type NonNullable<T> = Exclude<T, null | undefined>

```

### Parameters

拿到函数的参数类型，不定参数的组织形式就是一个数组，参考 `ArrayElement`的第二种写法，利用`infer`去取到类型

```ts
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) =>  any ? P : never
```

### ConstructorParameters

要拿到构造函数参数的类型，参考 `Parameters`，加上 `new` 即可

```ts
type ConstructorParameters<T extends new (...args: any) => any> = T extends new (...args: infer P) =>  any ? P : never
```

### InstanceType

获取实例类型，跟 `Parameters` 和 `ConstructorParameters` 差不多，不过这次不 `infer` 参数了，而是 `infer` 函数返回数据

```ts
type InstanceType<T extends new (...args: any) => any> = T extends new (...args: any) =>  infer R ? R : never
```
<!-- more -->
### NonFunctionKeys

拿到对象中所有非函数类型属性的 key，比如

```ts
type NonFunctionKeys<T extends object> = { [K in keyof T]-?: T[K] extends Function ? never : T[K] }[keyof T]
```

### NonFunction

如果要把对象中的函数剔除，留下其他的话，我们只要把 `NonFunctionKeys` 的结果再 `Pick` 一下就好了。

```ts
type NonFunction<T extends object> = Pick<T,NonFunctionKeys<T>>
```

### PickByValue

不再局限于属性值为函数，根据给定的类型进行挑选对象成员，比如我只想要一个对象中属性值类型为 `number | string`的成员。

``` ts
type PickByValue<U, T> = Pick<U, {[K in keyof U]-?: U[K] extends T ? K : never}[keyof U]>

```

但是其实你在给 `T` 传入类似 `number | string` 这样的类型其实是有些暧昧的，`number | string` 是代表 `number | string` 这个类型本身，又或者可以包含它的所有子类型呢，即 `number` 类型行，`string` 类型也接受，`any`, `never` 来者不拒, 显然这儿 `PickByValue` 是可以 `Pick` 到 `T` 的子类型。

### OmitByValue

有了 `PickByValue`，怎么能没有 `OmitByValue` 呢

``` ts
type OmitByValue<U, T> = Pick<U, {[K in keyof U]: U[K] extends T ? K : never}[keyof U]>
```

于是你就会发现，上面的 `NonFunction`, 用 `OmitByValue` 有了更加简单的写法

```ts
type NonFunction<T> = OmitByValue<T, Function>
```

### PickByValueExact

我们需要一个 `PickByValueExact` 来进行精确的 `Pick`，只选中类型 `T` 本身，忽略其子类型。 所以我们判断一个类型是不是其类型本身。

``` ts
A extends B  -> A <= B
B extends A  -> B >= A
// 根据我多年前的数学经验，满足 A extends B && B extends A 就说明 A == B 为同一类型（其实只是可以互相 assign 额）
```

于是

``` ts
type Same<A, B, X = 1, Y = 0> = A extends B ? B extends A ? X : Y : Y;

// 实验一下
type K = Same<number | string, string> // -> 0 | 1
```

发现情况有点不对，`K` 的类型是 `0 | 1`，跟预期的有点不一致。条件类型（ `T extends U ? X : Y`）在 `T` 为联合类型（例如 `A | B`）的时候会自动分发类型，`(A extends U ? X : Y) | (B extends U ? X : Y)`
于是

``` ts
type K = Same<number | string, string> // 相当于展开成下面的

type L = （number extends string ? string extends number ? 1 : 0 : 0） | (string extends string ? string extends string ? 1 : 0 : 0) // -> 0 | 1
```

kk, 有点烦自动分发条件类型，所幸只有对联合类型才会触发该行为，所以我们在处理的时候包一层, 把它统一塞到数组(或者转成函数)里面去，这样就可以绕过去了

``` ts
type Same<A, B, X = A, Y = never> = [A] extends [B] ? [B] extends [A] ? X : Y : Y;
```

于是 `PickByValueExact` 就可以这样写了

``` ts
export type PickByValueExact<T, V> = Pick<
  T,
  {
    [K in keyof T]-?: Same<T[K], V, K>
  }[keyof T]
>;
```

PS: `Same` 没法 cover 一些顶级类型(`any, unkown`)和可选属性的 case

```ts
type A = Same<{ b?: string }, {}, 1, 0> // 1 -> {} == { b?: string }
type B = Same<{ a?: string }, {}, 1, 0> // 1 -> {} == { a?: string }
// 那么 { a?: string } === { b?: string } ？？？，然而

type C = Same<{ b?: string }, { a?: string }, 1, 0> // 0 -> {a ?: string} != {b ?: string}

// 另外 Same 对于 unkown 和 any 这两顶级类型判断是有问题的
type D = Same<{ a: any }, {  a: string }, 1, 0> // 1
type D = Same<{ a: any }, {  a: unkown }, 1, 0> // 1
```

### OmitByValueExact

有 `PickByValueExact`，自然就有 `OmitByValueExact` 与之对应, 还是利用一下之前写好的 `Same`

``` ts
type OmitByValueExact<T, V> = Omit<
  T,
  {
    [K in keyof T]-?: Same<T[K], V, K>
  }[keyof T]
>;

// 还可以用 Pick & 倒转 Same 的返回， 实现这个 OmitByValueExact
type OmitByValueExact<T, V> = Pick<
  T,
  {
    [K in keyof T]-?: Same<T[K], V, never, K>
  }[keyof T]
>;
```

### Equals

之前泛型Same并没有办法推断出两个类型是否绝对等同，类似带有属性修饰器的类型例如 { readonly a: string } 跟 { s: string }， Same 会判断成一致。
双向 extends 的方法显然适用性有限。如何判断两个类型是否绝对一致，TS issue 区有人给出了一个比较 Hack 的[解决方案](https://github.com/Microsoft/TypeScript/issues/27024#issuecomment-421529650)

```ts
type Equals<X, Y，A = X, B = never> =
    (<T>() => T extends X ? 1 : 2) extends
    (<T>() => T extends Y ? 1 : 2) ? A : B;
```

其核心思路是利用 [conditional types(延时条件类型)](https://www.typescriptlang.org/docs/handbook/advanced-types.html#conditional-types)依赖于内部类型一致性检查。
`Equals` 依赖了三个泛型参数，`X`, `Y`，`T`，即使在只传入`X`,`Y`的情况下也会根据现有的信息进行初步的类型推断，如果能推断出就返回最终的类型，推断不出最终类型就返回当前推断结果， `defer` 推断过程，等待新的类型参数进来。
我们可以大胆推测，在只传了 `X`，`Y`的情况下，ts 内部把 `defer` 的条件类型标注成了新的类型比如 `X''` 和 `Y''`

``` ts
type Equals<X, Y> =
    X'' extends
    Y'' ? true : false;
```

是否等同的判断就变成了，对 `X''` 和 `Y''`的判断，而 `X''`和`Y''`的组织形式是一致的，`X'' = fn(X)`, 最后就变成了 `X` 和 `Y` 内部一致性的检查。最终推断出是不是真正的类型等同。
因为资料比较少，以上有部分推测，仅供参考。

### RequiredKeys

通过 `Omit` `Pick` 我们可以根据对象的 `key` 来剔除、选择对象某些属性，通过 `OmitByValue` `PickByValue` 我们可以根据值的类型剔除、挑选某些对象属性。
那么有没有办法找到对象必填`key`的集合呢？的确有, 需要一个小 Trick
`{} extends { a ?: string} ? X : Y` 一定会返回 `X` 类型，但是 `{} extends { a : string} ? X : Y` 一定会返回 `Y`，于是我们可以这样来取出所有的必填属性 `key`

``` ts
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K
}[keyof T]
```

### OptionalKeys

与 RequiredKeys 相反的自然就是 OptionalKeys，交换一下 never 和 K 的位置即可

``` ts
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : Never
}[keyof T]
```

### ReadonlyKeys

之前实现的 `Equals` 可以让我们最大限度低判断两个类型是不是相等，差不多算是js中的 `===` 了，也可以用它来拿到所有 `readonly` 的 `key`, 在不知道一个属性修饰符是否为`readonly`的情况下，移除掉 `readonly` 之后还与之前是等同的（`Equals`）, 那就说明其本来是带有 `readonly` 修饰符的。

``` ts
type ReadonlyKeys<T extends object> = {
  [P in keyof T]-?: Equals<
    { [Q in P]: T[P] },
    { -readonly [Q in P]: T[P] },
    never,
    P
  >;
}[keyof T];
```

### MutableKeys

`readonly` 与之相反的就是 `mutable` 了，`MutableKeys` 相比 `ReadonlyKeys` 只需要调整一下 `Equals` 的返回逻辑。

``` ts
type MutableKeys<T extends object> = {
  [P in keyof T]-?: Equals<
    { [Q in P]: T[P] },
    { -readonly [Q in P]: T[P] },
    P
  >;
}[keyof T];
```

## 结尾

其实大数多未必会用到，只是训练自己对泛型的用法的熟练度，所以水了一篇文章。
后面开始写业务啦，有机会再水一篇。

## 参考

- https://github.com/piotrwitek/utility-types
- https://www.typescriptlang.org/v2/docs/handbook/utility-types.html
- https://www.zhihu.com/question/276172039
- https://stackoverflow.com/questions/52443276/how-to-exclude-getter-only-properties-from-type-in-typescript
- https://stackoverflow.com/questions/41253310/typescript-retrieve-element-type-information-from-array-type