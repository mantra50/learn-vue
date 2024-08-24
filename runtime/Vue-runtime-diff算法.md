# diff 算法

在之前完成过一个 `patchChildren` 的方法，该方法的主要作用是为了 **更新子节点**，即：**为子节点打补丁**。



子节点的类型多种多样，如果两个 `ELEMENT` 的子节点都是 `TEXT_CHILDREN` 的话，那么直接通过 `setText` 附新值即可。



但是如果 **新旧 `ELEMENT` 的子节点都为 `ARRAY_CHILDREN`** 的话，那么想要完成一个 **高效** 的更新就会比较复杂了。这个时候，就需要，**比较两组子节点**，以达到一个高效的更新功能。这种 **比较的算法** 就是 `diff` 算法。

`vue` 中对 `diff` 算法分为五个场景：

1. `sync from start`：自前向后的对比
2. `sync from end`：自后向前的对比
3. `common sequence + mount`：新节点多于旧节点，需要挂载
4. `common sequence + unmount`：旧节点多于新节点，需要卸载
5. `unknown sequence`：乱序



## 一、前置知识：VNode 虚拟节点 key 属性的作用

在之前，有一个判断两个 `vnode` 是否相同的方法：

```typescript
/**
 * 根据 key || type 判断是否为相同类型节点
 */
export function isSameVNodeType(n1: VNode, n2: VNode): boolean {
	return n1.type === n2.type && n1.key === n2.key
}
```

这里的 `key` 是一个 `props`，比如可以在 `vue` 源码中，利用 `h` 函数，构建这样的一个 `VNode`：

```typescript
const vnode = h('li', {
  key: 1,
}, '1')
```

`VNode` 的值为：

```json
{
  "__v_isVNode": true,
  "type": "li",
  "props": { "key": 1 },
  "key": 1,
  "children": "1",
  "shapeFlag": 9,
  ....
}
```

如果通过 `isSameVNodeType`方法，判断两个 `VNode` 是否为 **相同的** **`VNode`**，则只需要保证两个 `VNode` 的 `type 和 key` 相同即可:

```typescript
const vnode = h('li', {
  key: 1,
}, '1')
const vnode2 = h('li', {
  key: 1,
}, '2')
isSameVNodeType(vnode, vnode2) // true
```

因此需要在 `createBaseVNode` 中新增 `key` 属性：

```diff
const vnode = {
	__v_isVNode: true,
	type,
	props,
	shapeFlag,
+	key: props?.key || null
} as VNode
```

## 二、自前向后的 diff 对比

```typescript
  /**
   * diff
   */
  const patchKeyedChildren = (
    oldChildren: VNode[],
    newChildren: VNode[],
    container: RendererElement,
    parentAnchor: RendererNode | null
  ) => {
    // 索引
    let i = 0
    // 新的 children 长度
    const newChildrenLength = newChildren.length
    // 旧 children 最大下标
    let oldChildrenEnd = oldChildren.length - 1
    // 新 children 最大下标
    let newChildrenEnd = newChildrenLength - 1

    // 1.自前向后的 diff 对比。经过该循环之后，从前开始的相同 vnode 将被处理
    // (a b) c
    // (a b) d e
    while (i <= oldChildrenEnd && i <= newChildrenEnd) {
      const oldChild = oldChildren[i]
      const newChild = normalizeVNode(newChildren[i])
      // 如果 oldVNode 和 newVNode 被认为是同一个 vnode，则直接 patch 即可
      if (isSameVNodeType(oldChild, newChild)) {
        patch(oldChild, newChild, container, parentAnchor)
      }
      // 如果不被认为是同一个 vnode，则直接跳出循环
      else {
        break
      }
      // 下标自增
      i++
    }
  }

```

## 三、自后向前的 diff 对比

```typescript
  /**
   * diff
   */
  const patchKeyedChildren = (...) => {
    ...
    // 2.自后向前的 diff 对比。经过该循环之后，从后开始的相同 vnode 将被处理
    // a (b c)
    // d e (b c)
    while (i <= oldChildrenEnd && i <= newChildrenEnd) {
      const oldVNode = oldChildren[oldChildrenEnd]
      const newVNode = normalizeVNode(newChildren[newChildrenEnd])
      if (isSameVNodeType(oldVNode, newVNode)) {
        patch(oldVNode, newVNode, container, parentAnchor)
      } else {
        break
      }
      oldChildrenEnd--
      newChildrenEnd--
    }
  }
```

## 四、新节点多于旧节点时的 diff 比对

```typescript
  const patchKeyedChildren = (...) => {
    ...  
	// 3.新节点多余旧节点时的 diff 比对
    // (a b)
    // (a b) c
    // i = 2, e1 = 1, e2 = 2
    // (a b)
    // c (a b)
    // i = 0, e1 = -1, e2 = 0
    if (i > oldChildrenEnd) {
      if (i <= newChildrenEnd) {
        const nextPos = newChildrenEnd + 1
        const anchor =
          nextPos < newChildrenLength ? newChildren[nextPos].el : null
        while (i <= newChildrenEnd) {
          patch(null, normalizeVNode(newChildren[i]), container, anchor)
          i++
        }
      }
    }
  }
```

## 五、新节点少于旧节点时的 diff 对比

```typescript
  const patchKeyedChildren = (...) => {
    ...  
    // 4.新节点少于旧节点时的 diff 对比
    // (a b) c
    // (a b)
    // i = 2, e1 = 2, e2 = 1
    // a (b c)
    // (b c)
    // i = 0, e1 = 0, e2 = -1
    else if (i > newChildrenEnd) {
      if (i <= oldChildrenEnd) {
        while (i <= oldChildrenEnd) {
          unmount(oldChildren[i])
          i++
        }
      }
    }
  }
```

## 六、乱序节点前置：最长递增子序列

```typescript
/**
 * 获取最长递增子序列下标
 * 维基百科：https://en.wikipedia.org/wiki/Longest_increasing_subsequence
 * 百度百科：https://baike.baidu.com/item/%E6%9C%80%E9%95%BF%E9%80%92%E5%A2%9E%E5%AD%90%E5%BA%8F%E5%88%97/22828111
 */
function getSequence(arr: number[]): number[] {
  // 获取一个数组浅拷贝。注意 p 的元素改变并不会影响 arr
  // p 是一个最终的回溯数组，它会在最终的 result 回溯中被使用
  // 它会在每次 result 发生变化时，记录 result 更新前最后一个索引的值
  const p = arr.slice()
  // 定义返回值（最长递增子序列下标），因为下标从 0 开始，所以它的初始值为 0
  const result = [0]
  let i, j, u, v, c
  // 当前数组的长度
  const len = arr.length
  // 对数组中所有的元素进行 for 循环处理，i = 下标
  for (i = 0; i < len; i++) {
    // 根据下标获取当前对应元素
    const arrI = arr[i]
    //
    if (arrI !== 0) {
      // 获取 result 中的最后一个元素，即：当前 result 中保存的最大值的下标
      j = result[result.length - 1]
      // arr[j] = 当前 result 中所保存的最大值
      // arrI = 当前值
      // 如果 arr[j] < arrI 。那么就证明，当前存在更大的序列，那么该下标就需要被放入到 result 的最后位置
      if (arr[j] < arrI) {
        p[i] = j
        // 把当前的下标 i 放入到 result 的最后位置
        result.push(i)
        continue
      }
      // 不满足 arr[j] < arrI 的条件，就证明目前 result 中的最后位置保存着更大的数值的下标。
      // 但是这个下标并不一定是一个递增的序列，比如： [1, 3] 和 [1, 2]
      // 所以我们还需要确定当前的序列是递增的。
      // 计算方式就是通过：二分查找来进行的

      // 初始下标
      u = 0
      // 最终下标
      v = result.length - 1
      // 只有初始下标 < 最终下标时才需要计算
      while (u < v) {
        // (u + v) 转化为 32 位 2 进制，右移 1 位 === 取中间位置（向下取整）例如：8 >> 1 = 4;  9 >> 1 = 4; 5 >> 1 = 2
        // https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Right_shift
        // c 表示中间位。即：初始下标 + 最终下标 / 2 （向下取整）
        c = (u + v) >> 1
        // 从 result 中根据 c（中间位），取出中间位的下标。
        // 然后利用中间位的下标，从 arr 中取出对应的值。
        // 即：arr[result[c]] = result 中间位的值
        // 如果：result 中间位的值 < arrI，则 u（初始下标）= 中间位 + 1。即：从中间向右移动一位，作为初始下标。 （下次直接从中间开始，往后计算即可）
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          // 否则，则 v（最终下标） = 中间位。即：下次直接从 0 开始，计算到中间位置 即可。
          v = c
        }
      }
      // 最终，经过 while 的二分运算可以计算出：目标下标位 u
      // 利用 u 从 result 中获取下标，然后拿到 arr 中对应的值：arr[result[u]]
      // 如果：arr[result[u]] > arrI 的，则证明当前  result 中存在的下标 《不是》 递增序列，则需要进行替换
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        // 进行替换，替换为递增序列
        result[u] = i
      }
    }
  }
  // 重新定义 u。此时：u = result 的长度
  u = result.length
  // 重新定义 v。此时 v = result 的最后一个元素
  v = result[u - 1]
  // 自后向前处理 result，利用 p 中所保存的索引值，进行最后的一次回溯
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

## 七、乱序 diff 实现(难)

```typescript
    // 5.乱序下的 diff 对比
    // [i ... oldChildrenEnd + 1]: a b [c d e] f g
    // [i ... newchildrenEnd + 1]: a b [e d c h] f g
    // i = 2, oldChildrenEnd = 4, newchildrenEnd = 5
    else {
      // 旧子节点的开始索引：oldChildrenStart
      const oldStartIndex = i
      // 新子节点的开始索引：newChildrenStart
      const newStartIndex = i

      // 5.1 创建一个 <key（新节点的 key）:index（新节点的位置）> 的 Map 对象 keyToNewIndexMap。通过该对象可知：新的 child（根据 key 判断指定 child） 更新后的位置（根据对应的 index 判断）在哪里
      const keyToNewIndexMap: Map<string | number | symbol, number> = new Map()
      // 通过循环为 keyToNewIndexMap 填充值（s2 = newChildrenStart; newchildrenEnd = newChildrenEnd）
      for (i = newStartIndex; i <= newChildrenEnd; i++) {
        // 从 newChildren 中根据开始索引获取每一个 child（c2 = newChildren）
        const nextChild = (newChildren[i] = normalizeVNode(newChildren[i]))
        // child 必须存在 key（这也是为什么 v-for 必须要有 key 的原因）
        if (nextChild.key != null) {
          // 把 key 和 对应的索引，放到 keyToNewIndexMap 对象中
          keyToNewIndexMap.set(nextChild.key, i)
        }
      }

      // 5.2 循环 oldChildren ，并尝试进行 patch（打补丁）或 unmount（删除）旧节点
      let j
      // 记录已经修复的新节点数量
      let patched = 0
      // 新节点待修补的数量 = newChildrenEnd - newChildrenStart + 1
      const toBePatched = newChildrenEnd - newStartIndex + 1
      // 标记位：节点是否需要移动
      let moved = false
      // 配合 moved 进行使用，它始终保存当前最大的 index 值
      let maxNewIndexSoFar = 0
      // 创建一个 Array 的对象，用来确定最长递增子序列。它的下标表示：《新节点的下标（newIndex），不计算已处理的节点。即：n-c 被认为是 0》，元素表示：《对应旧节点的下标（oldIndex），永远 +1》
      // 但是，需要特别注意的是：oldIndex 的值应该永远 +1 （ 因为 0 代表了特殊含义，他表示《新节点没有找到对应的旧节点，此时需要新增新节点》）。即：旧节点下标为 0， 但是记录时会被记录为 1
      const newIndexToOldIndexMap = new Array(toBePatched)
      // 遍历 toBePatched ，为 newIndexToOldIndexMap 进行初始化，初始化时，所有的元素为 0
      for (i = 0; i < toBePatched; i++) newIndexToOldIndexMap[i] = 0
      // 遍历 oldChildren（s1 = oldChildrenStart; oldChildrenEnd = oldChildrenEnd），获取旧节点（c1 = oldChildren），如果当前 已经处理的节点数量 > 待处理的节点数量，那么就证明：《所有的节点都已经更新完成，剩余的旧节点全部删除即可》
      for (i = oldStartIndex; i <= oldChildrenEnd; i++) {
        // 获取旧节点（c1 = oldChildren）
        const prevChild = oldChildren[i]
        // 如果当前 已经处理的节点数量 > 待处理的节点数量，那么就证明：《所有的节点都已经更新完成，剩余的旧节点全部删除即可》
        if (patched >= toBePatched) {
          // 所有的节点都已经更新完成，剩余的旧节点全部删除即可
          unmount(prevChild)
          continue
        }
        // 新节点需要存在的位置，需要根据旧节点来进行寻找（包含已处理的节点。即：n-c 被认为是 1）
        let newIndex
        // 旧节点的 key 存在时
        if (prevChild.key != null) {
          // 根据旧节点的 key，从 keyToNewIndexMap 中可以获取到新节点对应的位置
          newIndex = keyToNewIndexMap.get(prevChild.key)
        } else {
          // 旧节点的 key 不存在（无 key 节点）
          // 那么我们就遍历所有的新节点（s2 = newChildrenStart; newChildrenEnd = newChildrenEnd），找到《没有找到对应旧节点的新节点，并且该新节点可以和旧节点匹配》（s2 = newChildrenStart; newChildren = newChildren），如果能找到，那么 newIndex = 该新节点索引
          for (j = newStartIndex; j <= newChildrenEnd; j++) {
            // 找到《没有找到对应旧节点的新节点，并且该新节点可以和旧节点匹配》（s2 = newChildrenStart; newChildren = newChildren）
            if (
              newIndexToOldIndexMap[j - newStartIndex] === 0 &&
              isSameVNodeType(prevChild, newChildren[j] as VNode)
            ) {
              // 如果能找到，那么 newIndex = 该新节点索引
              newIndex = j
              break
            }
          }
        }
        // 最终没有找到新节点的索引，则证明：当前旧节点没有对应的新节点
        if (newIndex === undefined) {
          // 此时，直接删除即可
          unmount(prevChild)
        }
        // 没有进入 if，则表示：当前旧节点找到了对应的新节点，那么接下来就是要判断对于该新节点而言，是要 patch（打补丁）还是 move（移动）
        else {
          // 为 newIndexToOldIndexMap 填充值：下标表示：《新节点的下标（newIndex），不计算已处理的节点。即：n-c 被认为是 0》，元素表示：《对应旧节点的下标（oldIndex），永远 +1》
          // 因为 newIndex 包含已处理的节点，所以需要减去 s2（s2 = newChildrenStart）表示：不计算已处理的节点
          newIndexToOldIndexMap[newIndex - newStartIndex] = i + 1
          // maxNewIndexSoFar 会存储当前最大的 newIndex，它应该是一个递增的，如果没有递增，则证明有节点需要移动
          if (newIndex >= maxNewIndexSoFar) {
            // 持续递增
            maxNewIndexSoFar = newIndex
          } else {
            // 没有递增，则需要移动，moved = true
            moved = true
          }
          // 打补丁
          patch(prevChild, newChildren[newIndex] as VNode, container, null)
          // 自增已处理的节点数量
          patched++
        }
      }

      // 5.3 针对移动和挂载的处理
      // 仅当节点需要移动的时候，我们才需要生成最长递增子序列，否则只需要有一个空数组即可
      const increasingNewIndexSequence = moved
        ? getSequence(newIndexToOldIndexMap)
        : []
      // j >= 0 表示：初始值为 最长递增子序列的最后下标
      // j < 0 表示：《不存在》最长递增子序列。
      j = increasingNewIndexSequence.length - 1
      // 倒序循环，以便我们可以使用最后修补的节点作为锚点
      for (i = toBePatched - 1; i >= 0; i--) {
        // nextIndex（需要更新的新节点下标） = newChildrenStart + i
        const nextIndex = newStartIndex + i
        // 根据 nextIndex 拿到要处理的 新节点
        const nextChild = newChildren[nextIndex] as VNode
        // 获取锚点（是否超过了最长长度）
        const anchor =
          nextIndex + 1 < newChildrenLength
            ? (newChildren[nextIndex + 1] as VNode).el
            : parentAnchor
        // 如果 newIndexToOldIndexMap 中保存的 value = 0，则表示：新节点没有用对应的旧节点，此时需要挂载新节点
        if (newIndexToOldIndexMap[i] === 0) {
          // 挂载新节点
          patch(null, nextChild, container, anchor)
        }
        // moved 为 true，表示需要移动
        else if (moved) {
          // j < 0 表示：不存在 最长递增子序列
          // i !== increasingNewIndexSequence[j] 表示：当前节点不在最后位置
          // 那么此时就需要 move （移动）
          if (j < 0 || i !== increasingNewIndexSequence[j]) {
            move(nextChild, container, anchor)
          } else {
            // j 随着循环递减
            j--
          }
        }
      }
    }
  }

  const move = (
    vnode: RendererNode,
    container: RendererElement,
    anchor: RendererNode | null
  ) => {
    const { el } = vnode
    hostInsert(el, container, anchor)
  }
```

