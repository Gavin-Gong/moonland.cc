---
title: Vue 异步计算属性实现
date: 2018-07-28
widgets: []
tags:
  - Vue
categories:
  - FE
---

前段时间在 GitHub 上看到一个 Vue 异步计算属性的库 - [vue-async-computed](https://github.com/foxbenjaminfox/vue-async-computed), 将异步操作转化为计算属性, 例如这样用

```js
new Vue({
  data: {
    userId: 1
  },
  asyncComputed: {
    username () {
      return Vue.http.get('/get-username-by-id/' + this.userId)
        .then(response => response.data.username)
    }
  }
}
```

好奇其中原理, 看了源码, 了解其中巧妙的实现思路, 绘制了一张核心思路的原型图 ![](http://opazkqh2d.bkt.clouddn.com/18-7-28/27548766.jpg).

接下来我们实现一个简(阉)易(割)版本的 `vue-async-computed` 插件,

趁着 `beforeCreate` 往 `data` 中添加 `asyncComputed` 的 `key`, 使之响应式.

<!--more-->

```js
Vue.mixin({
  beforeCreate() {
    const data = this.$options.data;
    this.$options.data = function() {
      let _data = typeof data === "function" ? data.call(this) : data; // 有的时候 data 可能不是函数, 而是对象

      // 遍历 asyncComputed 注册 key
      for (const key in this.$options.asyncComputed || {}) {
        _data[key] = null;
      }
      return _data ? _data : {};
    };
});
```

注入计算魔改后的计算属性

```js
const prefix = "async$";
Vue.mixin({
  beforeCreate() {
    for (const key in this.$options.asyncComputed || {}) {
      if (!this.$options.computed) {
        this.$options.computed = {};
      }
      this.$options.computed[prefix + key] = this.$options.asyncComputed[key];
    }
  }
});
```

在 `created` 中 `watch` 魔改后的计算属性, 并且更新 `data` 上的值

```js
Vue.mixin({
  created() {
    for (const key in this.$options.asyncComputed || {}) {
      this.$watch(
        prefix + key,
        newPromise => {
          newPromise()
            .then(val => {
              this[key] = val;
            })
            .catch(err => {
              console.log(err);
            });
        },
        { immediate: true } // watch 之后立即调用一次, 因为此时有 Promise 实例了
      );
    }
  }
});
```

以上只是简单的实现, 源码其实还包括 lazy 计算属性, 默认值, 错误处理等特性, 具体可以看源码. 这是注释版源码
[vue-async-computed](https://github.com/Gavin-Gong/source/tree/master/vue-async-computed)
