---
title: 关于 Vue 的一些冷门技巧
tag:
  - Vue
categories:
  - FE
widgets: []
thumbnail: https://images.unsplash.com/photo-1530903677198-7c9f3577a63e
---

## watch

有时候我们会写一些这样的代码, 在 created 一个组件之后获取数据, 然后根据某个值得变动来获取数据, 正常套路是按照下面写的,

```js
created() {
 this.fetchUserList()
},
watch: {
  searchText () {
    this.fetchUserList ()
  }
}
```

但是 watch 的 api 其实很复杂, 语法糖甜的掉牙了. 于是我们可以简化为

<!--more-->

```js
watch: {
  searchText: {
    immediate: true, // 这样我就会在 created 之后立即调用一次
    handler() {
      this.fetchUserList ()
    }
  }
}
```

其实函数还可以是个字符串, 所以也可以这样

```js
watch: {
  searchText: {
    immediate: true,
    handler: 'fetchUserList'
  }
}
```

## 组件, 路由, vuex module 自动注册

利用 webpack 的 require.context 可以达到自动引用一堆文件

注册全局组件

```js
import Vue from "vue";
import upperFirst from "lodash/upperFirst";
import camelCase from "lodash/camelCase";
// Require in a base component context
const requireComponent = require.context(".", false, /base-[\w-]+\.vue$/);
requireComponent.keys().forEach(fileName => {
  // Get component config
  const componentConfig = requireComponent(fileName);
  // Get PascalCase name of component
  const componentName = upperFirst(
    camelCase(fileName.replace(/^\.\//, "").replace(/\.\w+$/, ""))
  );
  // Register component globally
  Vue.component(componentName, componentConfig.default || componentConfig);
});
```

注册 vuex module

```js
import camelCase from "lodash/camelCase";
const requireModule = require.context(".", false, /\.js$/);
const modules = {};
requireModule.keys().forEach(fileName => {
  // Don't register this file as a Vuex module
  if (fileName === "./index.js") return;
  const moduleName = camelCase(fileName.replace(/(\.\/|\.js)/g, ""));
  modules[moduleName] = {
    namespace: true,
    ...requireModule(fileName)
  };
});
export default modules;
```

## router-view

一旦出现从 card/1 跳转到 card/2, 这个时候可能会进行组件重用, 而又刚好你获取数据的时机是在 `created` 或者 `mounted`, 而没有用 beforeRouteUpdate, beforeRouteEnter 之列的钩子函数, 这个时候页面十有八九是不会刷新了, 于是乎一个思路是控制组件重新渲染. 如下

```html
<router-view :key="$route.fullPath"></router-view>
```

## 组件封装

在对组件进行封装的时候, 想把组件内部元素的原生事件以及原生属性暴露出去, 而又不想傻乎乎地自己手动一个个 emit 和 props, 于是我们需要借助于两个冷门的属性 `$listeners` 和 `$attr`, 代码如下

```html
<template>
    <label>
    {{ label }}
    <input
      v-bind="$attrs"
      :value="value"
      v-on="listeners"
    >
  </label>
</template>

<script>
  export default {
    inheritAttrs: false, // 组件根元素不再接受非 prop 属性, $attrs 其实就是非 prop 属性的集合
    computed: {
      listeners() {
        return {
          ...this.$listeners,
          input: event => this.$emit('input', event.target.value)
        }
      }
    }
  }
</script>
```

## 一个组件其实可以有多个根元素

```html
<script>
export default {
  functional: true,
  render() {
    return [1, 2, 3].map(item => {
      return <li>{item}</li>
    })
  }
}
</script>
```

## 参考链接

- [7 Secret Patterns Vue Consultants Don’t Want You to Know ](https://www.youtube.com/watch?v=7lpemgMhi0k&index=17&list=WL&t=2355s)
