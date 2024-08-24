# 一、renderer 基本架构

1. `renderer` 渲染器本身，需要构建出 `baseCreateRenderer` 方法

   ```typescript
   import { Comment, Fragment, Text, VNode } from './vnode'
   import { ShapeFlags } from 'packages/shared/src/shapeFlags'
   
   export interface RendererOptions {
     /**
      * 为指定 element 的 prop 打补丁
      */
     patchProp(el: Element, key: string, prevValue: any, nextValue: any): void
     /**
      * 为指定的 Element 设置 text
      */
     setElementText(node: Element, text: string): void
     /**
      * 插入指定的 el 到 parent 中，anchor 表示插入的位置，即：锚点
      */
     insert(el, parent: Element, anchor?): void
     /**
      * 创建指定的 Element
      */
     createElement(type: string)
   }
   
   export function createRenderer(options: RendererOptions) {
     return baseCreateRenderer(options)
   }
   
   /**
    * 生成 renderer 渲染器
    * @param options 渲染器配置选项
    */
   function baseCreateRenderer(options: RendererOptions): any {
     const patch = (
       oldVNode: VNode,
       newVNode: VNode,
       container,
       anchor = null
     ) => {
       if (oldVNode == newVNode) {
         return
       }
   
       const { type, shapeFlag } = newVNode
       switch (type) {
         case Text:
           break
         case Fragment:
           break
         case Comment:
           break
         default:
           if (shapeFlag & ShapeFlags.ELEMENT) {
             // TODO：处理元素
           } else if (shapeFlag & ShapeFlags.COMPONENT) {
             // TODO：处理组件
           }
           break
       }
     }
   
     const render = (vnode: VNode, container) => {
       if (vnode == null) {
         return
       } else {
         patch(container._vnode || null, vnode, container)
       }
       container._vnode = vnode
     }
   
     return {
       render
     }
   }
   
   ```

   **在这个方法当中会导出一个 render 的函数，这个函数是运行渲染器的入口，而这整个方法就是渲染器处理不同情况时的逻辑。**

   

2. 所有和 `dom` 的操作都是与 `core` 分离的，而和 `dom` 的操作包含了 **两部分**：
   1.  `Element` 操作：比如 `insert`、`createElement` 等，这些将被放入到 `runtime-dom` 中
   2. `props` 操作：比如 **设置类名**，这些也将被放入到  `runtime-dom` 中
   
   `nodeOps.ts` 文件存放各种 `Element` 的操作
   
   ```typescript
   const doc = document
   
   export const nodeOps = {
     /**
      * 插入指定元素到指定节点
      * @param children
      * @param parent
      * @param anchor
      */
     insert: (children, parent: HTMLElement, anchor) => {
       parent.insertBefore(children, anchor || null)
     },
     /**
      * 创建 element 标签
      * @param tag
      * @returns
      */
     createElement: (tag): Element => {
       const el = doc.createElement(tag)
       return el
     },
   
     /**
      * 为指定的 element 设置 textContent
      * @param el
      * @param text
      */
     setElementText: (el: Element, text) => {
       el.textContent = text
     }
   }
   ```
   
   `patchProp.ts` 存放各种补丁操作
   
   ```typescript
   import { isOn } from '@vue/shared'
   import { patchClass } from './../modules/class'
   
   /**
    * 为 prop 进行打补丁操作
    */
   export const patchProp = (el: Element, key, preValue, nextValue) => {
     if (key === 'class') {
       // class处理
       patchClass(el, nextValue)
     } else if (key === 'style') {
       // 样式处理
     } else if (isOn(key)) {
       // 事件处理
     } else {
     }
   }
   
   // modules/class.ts
   /**
    * 为 class 打补丁
    */
   export function patchClass(el: Element, value: string | null) {
     if (value === null) {
       el.removeAttribute('class')
     } else {
       el.className = value
     }
   }
   ```



# 二、Element 

## 1. 节点挂载

在 `baseCreateRenderer` 函数渲染器函数中，创建 `Element` 节点挂载函数 `processElement`

```typescript
function baseCreateRenderer(options: RendererOptions): any {
  const {
    insert: hostInsert,
    createElement: hostCreateElement,
    setElementText: hostSetElementText,
    patchProp: hostPatchProp
  } = options
  /**
   * ELement 的打补丁操作
   */
  const processElement = (
    oldVNode: VNode | null,
    newVNode: VNode,
    container,
    anchor
  ) => {
    if (oldVNode == null) {
      // 没有旧的 vnode 则进行挂载操作
      mountElement(newVNode, container, anchor)
    } else {
      // TODO: 更新操作
    }
  }
  /**
   * Element 挂载
   */
  const mountElement = (vnode: VNode, container, anchor) => {
    const { type, props, shapeFlag } = vnode
    // 创建 element
    const el = (vnode.el = hostCreateElement(type))
    // 根据 shapeFlag 进行字节点的设置
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
      // 子节点为文本
      hostSetElementText(el, vnode.children)
    } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      // TODO: 子元素为数组的情况
    }
    // 处理props
    if (props) {
      for (const key in props) {
        hostPatchProp(el, key, null, props[key])
      }
    }
    // 插入 el 到指定位置
    hostInsert(el, container, anchor)
  }
  const patch =(...)=>{
    ...
	switch (type) {
      ...
      default:
        if (shapeFlag & ShapeFlags.ELEMENT) {
          // TODO：处理元素
          processElement(oldVNode, newVNode, container, anchor)
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          // TODO：处理组件
        }
        break
    }
   }
  ...
}
```

### 导出 render 函数

在 `rontime-dom/src/index.ts` 中完成导出操作

```typescript
// 将 element 操作函数 和 props 操作函数 组合起来
const renderOptions = extend({ patchProp }, nodeOps)
let renderer
/**
 * 触发 createRenderer 函数获取到返回的 对象
 * @returns 一个对象：{ render }
 */
function ensureRenderer() {
  return renderer || (renderer = createRenderer(renderOptions))
}

export const render = (...args) => {
  ensureRenderer().render(...args)
}
```

## 2、render- element 更新操作

处理 `processElement` 中的 TODO: 更新操作，创建更新函数 `patchElement` ：

```typescript
  const patchElement = (oldVNode: VNode, newVNode: VNode) => {
    const el = (newVNode.el = oldVNode.el!)
    const oldProps = oldVNode.props || EMPTY_OBJ
    const newProps = newVNode.props || EMPTY_OBJ

    // 更新子节点操作
    patchChildren(oldVNode, newVNode, el, null)

    // 更新 props
    patchProps(el, newVNode, oldProps, newProps)
  }
```

更新操作分为两部分:

1. 更新子节点

   ``` typescript
   /**
    * 更新子节点
    */
   const patchChildren = (
     oldVNode: VNode,
     newVNode: VNode,
     container,
     anchor
   ) => {
     const c1 = oldVNode && oldVNode.children
     const preShapeFlag = oldVNode ? oldVNode.shapeFlag : 0
     const { children: c2, shapeFlag } = newVNode
     if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
       if (preShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
         // TODO: 卸载子节点
       }
       // 新旧节点都为 文本节点 且 内容不同
       if (c1 !== c2) {
         hostSetElementText(container, c2)
       }
     } else {
       // 旧节点为 ARRAY_CHILDREN
       if (preShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
         // 新节点也为 ARRAY_CHILDREN
         if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
           // TODO: diff 算法
         }
         // 新节点不是 ARRAY_CHILDREN 直接卸载旧的子节点
         else {
           // TODO: 卸载子节点
         }
       } else {
         // 旧节点是 TEXT_CHILDREN
         if (preShapeFlag & ShapeFlags.TEXT_CHILDREN) {
           // 删除旧文本
           hostSetElementText(container, '')
         }
         // 新节点是 ARRAY_CHILDREN
         if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
           // TODO: 挂载子节点
         }
       }
     }
   }
   ```

2. 更新 `props`:

   ```typescript
   /**
    * 更新 props
    * @param el
    * @param vnode
    * @param oldProps
    * @param newProps
    */
   const patchProps = (
     el: RendererNode,
     vnode: VNode,
     oldProps: any,
     newProps: any
   ) => {
     // 新旧 props 不相同时才进行处理
     if (oldProps !== newProps) {
       for (const key in newProps) {
         const prev = oldProps[key]
         const next = newProps[key]
         if (prev !== next) {
           hostPatchProp(el, key, prev, next)
         }
       }
       // 存在旧的 props 时
       if (oldProps !== EMPTY_OBJ) {
         for (const key in oldProps) {
           if (!(key in newProps)) {
             hostPatchProp(el, key, oldProps[key], null)
           }
         }
       }
     }
   }
   ```

   此时的更新操作只能处理相同标签的情况，接下来需要处理标签不相同时的更新操作。

   当进入 `patch` 函数开始的时候 进行标签类型的判断：

   ```typescript
   // renderer.ts => baseCreateRenderer fn
   const ={
     ...
     remove: hostRemove
   } = options
   // renderer.ts => baseCreateRenderer fn => patch fn
   // patching & not same type, unmount old tree
   if (oldVNode && !isSameVNodeType(oldVNode, newVNode)) {
       unmount(oldVNode)
       oldVNode = null
   }
   // nodeOps.ts => nodeOps
   const nodeOps = {
     ...
     remove(child) {
       const parent = child.parentNode
       if (parent) {
           parent.removeChild(child)
       }
     }
   }
   // baseCreateRenderer => unmount fn
   const unmount = (vnode: VNode) => {
       hostRemove(vnode.el!)
   }
   // vnode.ts
   export function isSameVNodeType(n1: VNode, n2: VNode) {
     return n1.type === n2.type && n1.key === n2.key
   }
   ```

## 3、节点的卸载操作

`render`  函数下的 TODO: 卸载操作。当新节点为空，且拥有旧节点时卸载旧节点

```typescript
if (vnode == null) {
    // TODO 卸载操作
    if (container._vnode) {
        unmount(container._vnode)
    }
    ...
}
```

## 4、其他属性的挂载

1. #### 因为 HTML Attributes 和 DOM Properties **针对不同的属性，需要通过不同方式进行设置** 所以需要在代码中需要进行属性的判断

   ```typescript
   export const patchProp: DOMRendererOptions['patchProp'] = (
     el,
     key,
     preValue,
     nextValue
   ) => {
     if (key === 'class') {
       // class处理
       patchClass(el, nextValue)
     } else if (key === 'style') {
       // 样式处理
     } else if (isOn(key)) {
       // 事件处理
     } else if (shouldSetAsProp(el, key)) {
       patchDOMProp(el, key, nextValue)
     } else {
       patchAttr(el, key, nextValue)
     }
   }
   
   function shouldSetAsProp(el: Element, key: string) {
     if (key === 'form') {
       return false
     }
     if (key === 'list' && el.tagName === 'INPUT') {
       return false
     }
     if (key === 'type' && el.tagName === 'TEXTAREA') {
       return false
     }
     return key in el
   }
   /**
    * 使用 setAttribute 设置属性
    */
   export function patchAttr(el: Element, key: string, value: any) {
     if (value == null) {
       el.removeAttribute(key)
     } else {
       el.setAttribute(key, value)
     }
   }
   /**
    * 使用 DOM API 给元素设置属性
    */
   export function patchDOMProp(el: Element, key: string, value: any) {
     try {
       el[key] = value
     } catch (error) {
       throw new Error(`[runtime-dom] set ${key} error`)
     }
   }
   ```

2. #### style 属性挂载

   ```typescript
   export function patchStyle(el: Element, prev, next) {
     // 提取 style 对象
     const style = (el as HTMLElement).style
     // 判断新的样式是否为纯字符串
     const isCssString = isString(next)
   
     if (next && !isCssString) {
       // 如果新的样式是对象，则遍历对象，设置新的样式
       for (const key in next) {
         setStyle(style, key, next[key])
       }
       // 清理旧样式
       if (prev && !isString(prev)) {
         for (const key in prev) {
           // 删除就对象中存在，而新对象中不存在的样式
           if (next[key] == null) {
             setStyle(style, key, '')
           }
         }
       }
     }
   }
   
   function setStyle(style: CSSStyleDeclaration, key: string, val: string) {
     style[key] = val
   }
   ```

## 5、事件挂载

所有的事件触发实际上是使用 `invorker` 对象里面的 `value` 触发，当不存在缓存时，会创建一个 `invorker` 对象并放到 `el._vei` 中：

` const invorker = (el._vei[rawName] = createInvoker(nextValue))`

这个缓存可以极大的减少事件挂载和删除的操作，以提升性能。原理是这个 `invorker` 里面会使用 `value` 属性来触发真实的操作，而想要更新这个操作就只需要更改这个 `invorker` 的 `value` 属性，也就是：

`existingInvoker.value = nextValue`

```typescript
interface Invoker extends EventListener {
  value: EventValue
}

type EventValue = Function

export function patchEvent(
  el: Element & { _vei?: Record<string, Invoker | undefined> },
  rawName: string,
  preValue: EventValue | null,
  nextValue: EventValue | null
) {
  // vei = vue event invokers
  const invorkers = el._vei || (el._vei = {})
  // 是否存在缓存事件
  const existingInvoker = invorkers[rawName]
  // 如果当前事件存在缓存，并且存在新的事件行为，则判定为更新操作。直接更新 invoker 的 value 即可
  if (nextValue && existingInvoker) {
    existingInvoker.value = nextValue
  } else {
    // 获取用于 addEventListener || removeEventListener 的事件名
    const name = parseName(rawName)
    if (nextValue) {
      // add
      const invorker = (el._vei[rawName] = createInvoker(nextValue))
      el.addEventListener(name, invorker)
    } else if (existingInvoker) {
      // remove
      el.removeEventListener(name, existingInvoker)
      // 删除缓存
      invorkers[rawName] = undefined
    }
  }
}
/**
 * 直接返回剔除 on，其余转化为小写的事件名即可
 */
function parseName(name: string) {
  return name.slice(2).toLowerCase()
}

/**
 * 生成 invoker 函数
 */
function createInvoker(initialValue: EventValue) {
  const invoker = (e: Event) => {
    invoker.value && invoker.value()
  }
  // value 为真实的事件行为
  invoker.value = initialValue
  return invoker
}
```

## 6、小结

以上就是针对于 `ELEMENT` 的各种处理：

1. 挂载
2. 更新
3. 卸载
4. `path props` 打补丁操作:
   1. `class`
   2. `style`
   3. `event`
   4. `attr`

针对于 挂载、更新、卸载而言，主要使用了 `packages/runtime-dom/src/nodeOps.ts` 中的浏览器兼容方法进行的实现，如：

1. `doc.createElement`
2. `parent.removeChild`

等。

而对于 `patch props` 的操作而言，因为 `HTML Attributes 和 DOM Properties` 的不同问题，所以需要针对不同的 `props` 进行分开的处理。

最后的 `event`，本身并不复杂，但是 `vei` 的更新思路是非常值得学习的一种事件更新方案。

# 三、`Text` 、`Comment` 以及 `Component` 的渲染行为

1. ## `Text` 操作

   ```typescript
   const {
     ...
     createText: hostCreateText,
     setText: hostSetText
   } = options  
   /**
    * 挂载 Text 节点
    */
   const processText = (
       oldeVNode: VNode | null,
       newVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
   ) => {
       // 如果旧的 vnode 不存在，则进行挂载操作
       if (oldeVNode == null) {
           newVNode.el = hostCreateText(newVNode.children)
           hostInsert(newVNode.el, container, anchor)
       } else {
           // 如果旧的 vnode 存在，则进行更新操作
           const el = (newVNode.el = oldeVNode.el!)
           if (newVNode.children !== oldeVNode.children) {
               hostSetText(el, newVNode.children)
           }
       }
   }
   ```

2. ## Comment 操作

   ```typescript
     /**
      * 创建注释节点
      */
     const processCommentNode = (
       oldeVNode: VNode | null,
       newVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
     ) => {
       if (oldeVNode == null) {
         newVNode.el = hostCreateCommentNode(newVNode.children)
         hostInsert(newVNode.el, container, anchor)
       } else {
         // 没有更新操作
         newVNode.el = oldeVNode.el
       }
     }
   ```

3. ## Fragment 操作

   ```typescript
     const processFragment = (
       oldVNode: VNode | null,
       newVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
     ) => {
       if (oldVNode == null) {
         // 挂载操作
         mountChildren(newVNode.children, container, anchor)
       } else {
         patchChildren(oldVNode, newVNode, container, anchor)
       }
     }
     /**
      * 循环挂载子节点
      */
     const mountChildren = (
       children: any,
       container: RendererElement,
       anchor: RendererNode | null
     ) => {
       if (typeof children === 'string') {
         children = children.split('')
       }
       // 循环渲染子节点
       for (let i = 0; i < children.length; i++) {
         const child = (children[i] = normalizeVNode(children[i]))
         patch(null, child, container, anchor)
       }
     }
     
     
   /**
    * 标准化 VNode
    */
   export function normalizeVNode(child: any): VNode {
     if (typeof child === 'object') {
       // 如果是对象则直接返回 vnode
       return cloneVNode(child)
     } else {
       // 如果不是对象则创建 Text vnode 并返回
       return createVNode(Text, null, String(child))
     }
   }
   
   export function cloneVNode(child: VNode) {
     return child
   }
   
   ```

# 四、总结

1. `render` 函数的渲染机制：

   `render` 函数是渲染器对象中的一个函数。但是这个渲染函数是不可以直接被用户访问的。所以需要额外在 `packages/runtime-dom/src/index.ts` 中构建一个 `render` 函数，用来给用户使用 render 渲染器 。

2. 在整个的 `render` 渲染过程中，所有的核心逻辑放入到了 `runtime-core` 文件夹内，把所有和浏览器相关的 API 操作，放到了 `runtime-dom` 文件夹下。以此来达到兼容性的目的。

3. 针对不同的节点，具体的渲染逻辑会有所不同。分别从：

   1. `ELEMENT`
   2. `TEXT`
   3. `COMMENT`

   这三种节点类型入手，讲解了 挂载、更新、卸载 的对应操作逻辑。

   这三种节点中，`ELEMENT` 的实现是最复杂的。

4. 对于 `ELEMENT` 而言，它的渲染除了本身的 挂载、更新、卸载 之外，还包含了 `props` 属性 的挂载、更新、卸载操作。

   对于 `props` 属性而言，因为 `HTML Attributes` 和 `DOM Properties` 的原因，所以需要针对不同的 props 分别进行处理。

   而对于 `event` 事件而言，这里 `vue` 通过 `vei` 的方式进行了更新逻辑，这个设计非常精妙。

5. 未处理的问题：

   1. `diff` ：在旧节点和新节点的子节点都为 Array 时，需要进行 diff 运算，已完成高效的更新。
   2. `component` 组件：组件的处理也是 `vue` 中非常重要的一块内容，这个也需要进行单独的处理。
