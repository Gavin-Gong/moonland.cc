---
title: React, Vue2, Vue3 diff 算法解析
date: 2021-03-07
widgets: []
tags:
  - vdom
  - js
  - 算法
categories:
  - FE
  - 算法
---

## 前言

前端的虚拟 DOM 框架，一般是用 js 树结构来对视图层进行抽象表达，通过对树的遍历创建 UI 元素，渲染到视图。
如果想要对视图进行更新，这时候会产生一颗新的树，显然根据新的数据全量重新创建 UI 元素，渲染到视图是极其消耗性能的。
因此需要对新老两颗树进行对比，找到差异点，将差异部分更新到存在的视图。

为了保证性能，我们需要考虑以下下几点:

1. 尽可能复用现有 UI 元素
2. 减少对 UI 元素的操作
3. 减少遍历次数

在实际场景中，其实基本不会遇到子树的层级跃迁移动，往往只会对同一层级的元素进行更新。所以没必要进行跨层级的比较。于是就变成了两个数组之间的比较。

## 正文

基于以上可以抽象为一道算法题：
存在 oldChildren newChildren 两个数组，数组元素在该数组内都是唯一且不重复的，UIChildren 初始值为 oldChildren 的副本，

1. 尽可能少地操作读取 UIChildren
2. 尽可能少地读写 oldChildren 和 newChildren
3. 优先满足第一点

尽可能利用 UIChildren 中的现有元素（newChildren，oldChildren 中的交集元素在 UIChildren 中 只能被移动不能被删除），
通过对比 oldChildren 和 newChildren 实现将 UIChildren 变为 newChildren 的副本
例如；
oldChildren 为 [1, 3, 4]
newChildren 为 [1, 2, 4]
那么 UIChildren 的初始值为 [1, 3, 4]
最后返回的 UIChildren 为 [1, 2, 4]
值得注意的是，对 UIChildren 进行操作的过程中，1 和 4 是属于可以复用的元素，所以只可以被移动但是不可以被删除的元素。

``` js
function diff(oldChildren, newChildren) {
  let UIChildren = oldChildren.map(v => v) // clone
  // TODO:
  return UIChildren
}
```

对 UIChildren 进行操作提供了以下几个 API（其实是为了模拟 DOM 的 API，它和数组用法上设计风格差异很大，一个基于 index，一个基于元素）

- inesertBefore(UIChildren, child, refChild) 讲一个元素插入到某一个元素前面
- removeChild(UIChildren, child) 移除某一个元素
- nextSibling(UIChildren, child) 获取某一个元素之后的元素
具体实现[参见](https://github.com/Gavin-Gong/js/blob/master/DSA/algorithms/vnode-diff/_util.js)

### React

``` js
/**
 * @desc React diff 算法
 * 核心思想：进行双层循环, 一旦遇到相同的节点, 
 * 与上次记录的 oldChildren 中的最大 index 进行比较，
 * 如果本次 index >  lastIndex 则说明顺序新旧一致，无需移动，只需记录本次 index 到 lastIndex
 * 否则说明需要将本节点移动到上一个节点之后
 * 由于始终使用后移的方式来调整顺序，
 * 这种场景下 [1, 2, 3, ..., 100] -> [100, 1, 2, ..., 99]，需要将 1-99 向后移动，共计需要99次操作。
 * 而不是将 100 插入到 1 前面，这样只需一次操作
 * 操作指使用提供的 DOM API
 * @param {[number]} oldChildren 
 * @param {[number]} newChildren 
 */
export function diff(oldChildren, newChildren) {
  let UIChildren = oldChildren.map(v => v);
  let lastIndex = 0;

  if (newChildren.length === 0) {
    UIChildren = []
  }
  
  for (let n = 0; n < newChildren.length; n++) {
    const newNode = newChildren[n]
    // 暂存上一个节点的的最大 index, 
    // 如果后续节点相同且 index < lastIndex, 则说明该节点需要向后移动
    let hasFind = false; // 是否找到元素
    let o = 0; 
    for(o; o < oldChildren.length; o++) {
      const oldNode = oldChildren[o]
      if (newNode === oldNode) {
        hasFind = true
        if (o < lastIndex) {
          // 当前节点和 n-1 节点顺序错位, 需要将当前节点插入到 n-1 后一位（的前面）, 
          const refNode = nextSibling(UIChildren, newChildren[n - 1])
          inesertBefore(UIChildren, oldNode, refNode)
        } else {
          lastIndex = o
        }
        // 一旦有节点相同, 中断内层循环, 防止继续更新下去浪费性能
        break
      }
    }

    // oldChildren 里面找不到 newChildren 的元素 说明要新增
    // i 为 0 ,说明遍历的是第一个元素, 插入到 UIChildren 的第一个节点即可
    // i >= 1, 直接插入到该元素前面
    if (!hasFind) {
      const refNode = n - 1 < 0 ? oldChildren[0] : nextSibling(UIChildren, newChildren[n - 1])
      inesertBefore(UIChildren, newChildren[n], refNode)
    }

    // newChildren 里面找不到 oldChildren 里面的元素说明要移除
    for (let i = 0; i < oldChildren.length; i++) {
      const item = oldChildren[i];
      const hasFind = newChildren.find(v => v === item)
      if (!hasFind) {
        removeChild(UIChildren, item)
      }
    }
  }
  return UIChildren
}
```

[代码参见](https://github.com/Gavin-Gong/js/blob/master/DSA/algorithms/vnode-diff/react.js)

### Snabddom / Vue2

```js
/**
 * @desc snabbdom & vue2 中采用的双端比较算法
 * 一层循环同时遍历两个数组, 通过组合 oldStart, oldEnd 和 newStart，newEnd 进行双端比较
 * 一共会有四种情况，直到某一个数组遍历完成
 * 然后处理未遍历到的区间
 * @param {[number]} oldChildren 
 * @param {[number]} newChildren 
 */
export function diff(oldChildren, newChildren) {
  let UIChildren = oldChildren.map(v => v) // 模拟 UI 元素
  let oldStartIdx = 0
  let newStartIdx = 0

  let oldEndIdx = oldChildren.length - 1
  let newEndIdx = newChildren.length - 1

  let oldStartNode = oldChildren[0]
  let newStartNode = newChildren[0]

  let oldEndNode = oldChildren[oldEndIdx]
  let newEndNode = newChildren[newEndIdx]

  while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (!oldStartNode) {
      oldStartNode = oldChildren[++oldStartIdx]
    } else if(!oldEndNode) {
      oldEndNode = oldChildren[--oldEndIdx]
    } else  if (oldStartNode === newStartNode) {
      oldStartNode = oldChildren[++oldStartIdx]
      newStartNode = newChildren[++newStartIdx]
    } else if (oldEndNode === newEndNode) {
      oldEndNode = oldChildren[--oldEndIdx]
      newEndNode = newChildren[--newEndIdx]
    } else if (oldStartNode === newEndNode) {
      // 后移
      // 将节点移动到剩余遍历区域最后一个节点之后
      inesertBefore(UIChildren, oldStartNode, nextSibling(UIChildren, oldEndNode))
      oldStartNode = oldChildren[++oldStartIdx]
      newEndNode = newChildren[--newEndIdx]
    } else if (oldEndNode === newStartNode) {
      // 前移
      // 将节点移动到遍历区域最第一个节点之前
      inesertBefore(UIChildren, oldEndNode, oldStartNode)
      oldEndNode = oldChildren[--oldEndIdx]
      newStartNode = newChildren[++newStartIdx]
    } else {
      // 其他情况
      // 拿 newChildren 中的起始节点尝试去 oldChildren 中寻找
      const newEle = newChildren[newStartIdx]
      const oldIdx = oldChildren.indexOf(newEle)
      if (oldIdx === -1) {
        // 找不到说明是新元素，插入 oldStartNode 之前
        inesertBefore(UIChildren, newEle, oldStartNode)
      } else {
        // 能找到执行插入操作
        inesertBefore(UIChildren, oldChildren[oldIdx], oldStartNode)

        // 此时oldChildren[oldIdx] 一定处于非边缘位置
        // 而且未通过 while 循环遍历过
        // 所以置空， 再结合对空元素的判断，跳过这个已经被移动的节点
        oldChildren[oldIdx] = undefined 
      }
      // newChildren 的第一个元素处理过了，向右移动
      newStartNode = newChildren[++newStartIdx]
    }
  }

  // 上面循环并不能保证完整遍历两个数组的所有元素
  // 当一个数组遍历完之后，另一个数组可能残留一段区间，尚未遍历到
  // 当 oldChildren 残留区间未遍历 说明要删除这些元素
  // 当 newChildren 残留区间未遍历 说明要新增这些元素
  // 这段区间之内不可能存在可以复用的元素
  if (oldStartIdx > oldEndIdx) {
    // 新增
    // 始终插入到 newChildren[newEndIdx + 1] 之前能维持新插入的元素，从前到后的顺序
    // null 默认插入到最后面
    const before = newChildren[newEndIdx + 1] == null ? null : newChildren[newEndIdx + 1]
    for (let i = newStartIdx; i <= newEndIdx; i++) {
      inesertBefore(UIChildren, newChildren[i], before)
    }
  } else if (newEndIdx < newStartIdx) {
    // 移除
    for (let i = oldStartIdx; i <= oldEndIdx; i++) {
      removeChild(UIChildren, oldChildren[i])
    }
  }
  return UIChildren
}
```

[代码参见](https://github.com/Gavin-Gong/js/blob/master/DSA/algorithms/vnode-diff/vue.js)

### Inferno / Vue3

``` js
/**
 * @desc inferno & vue3 中采用的最长子序列算法
 * 糅合了前两个算法的一些技巧，再加上最长子序列算法来减少节点的移动操作
 * @param {[number]} oldChildren
 * @param {[number]} newChildren
 */
export function diff(oldChildren, newChildren) {
  const UIChildren = oldChildren.map((v) => v); // clone

  let oldEndIndex = oldChildren.length - 1;
  let newEndIndex = newChildren.length - 1;

  let i = 0;
  let oldEndNode = oldChildren[i];
  let newEndNode = newChildren[i];

  // 类似 vue 2 的算法，进行双向比较
  outer: {
    // 从数组前面开始比对
    while (oldEndNode === newEndNode) {
      i++;
      // 一旦有一个数组遍历完成，跳出执行块
      if (i > oldEndIndex || i > newEndIndex) {
        break outer;
      }
      oldEndNode = oldChildren[i];
      newEndNode = newChildren[i];
    }

    // 从数组后面开始比对
    oldEndNode = oldChildren[oldEndIndex];
    newEndNode = newChildren[newEndIndex];
    while (oldEndNode === newEndNode) {
      newEndIndex--;
      oldEndIndex--;
      // 一旦有一个数组遍历完成，跳出执行块
      if (i > oldEndIndex || i > newEndIndex) {
        break outer;
      }
      oldEndNode = oldChildren[oldEndIndex];
      newEndNode = newChildren[newEndIndex];
    }
  }

  if (i > oldEndIndex && i <= newEndIndex) {
    // i > oldIndex 说明 oldChildren 已经遍历完成
    // i <= newIndex 说明 newChildren 尚未遍历完
    // 此时，这些遍历完的 newChildren 元素一定是需要新增的
    // 新增操作
    const refNode = newChildren[newEndIndex + 1] == null ? null: newChildren[newEndIndex + 1]
    while(i <= newEndIndex) {
      inesertBefore(UIChildren, newChildren[i++] ,refNode)
    }
  } else if (i > newEndIndex) {
    // newChildren 已经遍历完
    // oldChildren 已经暂未遍历完成
    // 此时，这些未遍历完的oldChildren 元素一定是需要移除的
    // 移除操作
    while(i <= oldEndIndex) {
      removeChild(UIChildren, oldChildren[i++])
    }
  } else {
    // 上边处理了比较简单的情况

    const newRemaining = newEndIndex - i + 1 // newChildren 中未处理节点数量
    const source = new Array(newRemaining).fill(-1) // 用来保存未处理的 newChildren 区间元素对应的元素在 oldChildren 的 index

    const oldStartIdx = i
    const newStartIdx = i
    let shouldMove = false
    let lastIndex = 0
    const keyIndexMap = {} // { value/key: index } newChildren未处理区间索引表
    for (let k = newStartIdx; k <= newEndIndex; k++) {
      keyIndexMap[newChildren[k]] = k
    }
    let patched = 0
    
    // 遍历 oldChildren 剩余节点
    for (let i = oldStartIdx; i <= oldEndIndex; i++) {
      const oldNode = oldChildren[i]
      if (patched < newRemaining) {
        const k = keyIndexMap[oldNode] // 找到 oldNode 在 newChildren 中的 index
        if (typeof k !== 'undefined') {
          patched ++
          source[k - newStartIdx] = i

          // 判断是否需要移动，类似 React 的 diff 的思路
          if (k < lastIndex) {
            shouldMove = true
          } else {
            lastIndex = k
          }
        } else {
          // 在 newChildren 中找不到了，移除
          removeChild(UIChildren, oldNode)
        }
      } else {
        // 此时的 oldChildren 未遍历完成， newChildren 已经遍历完成
        // 剩余的oldChildren 中的元素未出现在 newChildren，应当被移除
        removeChild(UIChildren, oldNode)
      }
    }

    if (shouldMove) {
      const seq = lis(source) // 返回最长增长子序列 index 数组
      let j = seq.length - 1

      // 处理未处理的 newChildren 区间
      for(let i = newRemaining - 1; i>= 0; i --) {
        if (source[i] === -1) {
          // 不存在说明是新节点，直接插入到上一个节点处理好的节点即可
          const curIndex = i + newStartIdx
          const newNode = newChildren[curIndex]
          const refNode = newChildren[curIndex + 1] == null ? null : newChildren[curIndex + 1]
          inesertBefore(UIChildren, newNode, refNode)
        } else if (i !== seq[j]) {
          // 说明该节点需要移动
          // 其实和上面的 if 处理逻辑一致
          const curIndex = i + newStartIdx
          const newNode = newChildren[curIndex]
          const refNode = newChildren[curIndex + 1] == null ? null : newChildren[curIndex + 1]
          inesertBefore(UIChildren, newNode, refNode)
        } else {
          // i === seq[j] 说明当前节点在增长子序列中，无需移动
          // j 往前移动
          j--
        }
      }
    }
  }
  return UIChildren;
}
```

[代码参见](https://github.com/Gavin-Gong/js/blob/master/DSA/algorithms/vnode-diff/vue-next.js)

## 参考

- <http://hcysun.me/vue-design/zh/>
- <https://github.com/snabbdom/snabbdom/blob/master/src/package/init.ts>
