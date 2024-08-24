# 一、初见 h 函数

在 `vue` 中 `h` 函数能够处理很多不同情况下的标签类型，这里只实现部分情况下的标签，`text` 、`fragement`、`comment`、`element`、`component` 这五种类型就是部分实现的内容。`h` 函数核心是用来：**创建 `vnode`** 的，`vnode` 这个对象是对需要创建的节点的描述，`vue` 的 `renderer` 就是根据 `vnode` 的描述进行虚拟 DOM 到 真实 DOM 的转换操作。

在 `vue` 中实现不同标签的识别靠的就是 `shapeFlag` 的值，而这个值会根据不同的 `type` 取值按位或运算取得。

# 二、 Element 标签类型识别

1. 创建 `shapeFlags` 枚举类型：

   ```typescript
   export const enum ShapeFlags {
   	/**
   	 * type = Element
   	 */
   	ELEMENT = 1,
   	/**
   	 * 函数组件
   	 */
   	FUNCTIONAL_COMPONENT = 1 << 1,
   	/**
   	 * 有状态（响应数据）组件
   	 */
   	STATEFUL_COMPONENT = 1 << 2,
   	/**
   	 * children = Text
   	 */
   	TEXT_CHILDREN = 1 << 3,
   	/**
   	 * children = Array
   	 */
   	ARRAY_CHILDREN = 1 << 4,
   	/**
   	 * children = slot
   	 */
   	SLOTS_CHILDREN = 1 << 5,
   	/**
   	 * 组件：有状态（响应数据）组件 | 函数组件
   	 */
   	COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT
   }
   ```

2. 构建 `h` 函数：`h` 函数里面只是处理了一下传入的参数，实现自动识别不同传参方式。主要的 `vnode` 创建由 `createVNode` 函数完成

   ```typescript
   import { isArray, isObject } from '@vue/shared'
   import { createVNode, isVNode, VNode } from './vnode'
   
   export function h(type: any, propsOrChildren?: any, children?: any): VNode {
   	// 获取用户传递的参数数量
   	const l = arguments.length
   	// 如果用户只传递了两个参数，那么证明第二个参数可能是 props , 也可能是 children
   	if (l === 2) {
   		// 如果 第二个参数是对象，但不是数组。则第二个参数只有两种可能性：1. VNode 2.普通的 props
   		if (isObject(propsOrChildren) && !isArray(propsOrChildren)) {
   			// 如果是 VNode，则 第二个参数代表了 children
   			if (isVNode(propsOrChildren)) {
   				return createVNode(type, null, [propsOrChildren])
   			}
   			// 如果不是 VNode， 则第二个参数代表了 props
   			return createVNode(type, propsOrChildren)
   		}
   		// 如果第二个参数不是单纯的 object，则 第二个参数代表了 props
   		else {
   			return createVNode(type, null, propsOrChildren)
   		}
   	}
   	// 如果用户传递了三个或以上的参数，那么证明第二个参数一定代表了 props
   	else {
   		// 如果参数在三个以上，则从第二个参数开始，把后续所有参数都作为 children
   		if (l > 3) {
   			children = Array.prototype.slice.call(arguments, 2)
   		}
   		// 如果传递的参数只有三个，则 children 是单纯的 children
   		else if (l === 3 && isVNode(children)) {
   			children = [children]
   		}
   		// 触发 createVNode 方法，创建 VNode 实例
   		return createVNode(type, propsOrChildren, children)
   	}
   }
   ```

3. 创建 `createVNode` 方法：

   ```typescript
   /**
    * 生成一个 VNode 对象，并返回
    * @param type vnode.type
    * @param props 标签属性或自定义属性
    * @param children 子节点
    * @returns vnode 对象
    */
   export function createVNode(type, props, children): VNode {
   	// 通过 bit 位处理 shapeFlag 类型
   	const shapeFlag = isString(type) ? ShapeFlags.ELEMENT : 0
   
   	return createBaseVNode(type, props, children, shapeFlag)
   }
   
   /**
    * 构建基础 vnode
    */
   function createBaseVNode(type, props, children, shapeFlag) {
   	const vnode = {
   		__v_isVNode: true,
   		type,
   		props,
   		shapeFlag
   	} as VNode
   	// 处理 children
   	normalizeChildren(vnode, children)
   	return vnode
   }
   
   export function normalizeChildren(vnode: VNode, children: unknown) {
   	let type = 0
   	const { shapeFlag } = vnode
   	if (children == null) {
   		children = null
   	} else if (isArray(children)) {
   		// TODO: array
   	} else if (typeof children === 'object') {
   		// TODO: object
   	} else if (isFunction(children)) {
   		// TODO: function
   	} else {
   		// children 为 string
   		children = String(children)
   		// 为 type 指定 Flags
   		type = ShapeFlags.TEXT_CHILDREN
   	}
   	// 修改 vnode 的 chidlren
   	vnode.children = children
   	// 按位或赋值
   	vnode.shapeFlag |= type
   }
   ```



# 三、component 组件类型

对于 **组件** 而言，它的一个渲染，与之前不同的地方主要有两个：

1. `shapeFlag` === 4
2. `type`：是一个 **对象（组件实例）**，并且包含 `render` 函数

因为这里不涉及子节点，所以只需要在 `createdVNode` 中修改 `shapeFlage` 的初始表达赋值即可。

```typescript
  // 通过 bit 位处理 shapeFlag 类型
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : isObject(type)
    ? ShapeFlags.STATEFUL_COMPONENT
    : 0
```





# 四、其他标签类型处理

`Fragment` 标记为 **片段**。它相对比较特殊，是  `Vue3` 中新提出的一个概念，主要应对与 **包含多个根节点的模板** 。

`Text`  标记为 **文本**。即：纯文本的 `VNode`。

`Comment` 标记为 **注释**。即：注释节点的 `VNode`。

```typescript
export const Fragment = Symbol('Fragment')
export const Text = Symbol('Text')
export const Comment = Symbol('Comment')
```



# 五、增强虚拟节点下的 class 和 style

在 `vue` 中对 [Class 和 Style ](https://cn.vuejs.org/guide/essentials/class-and-style) 做了专门的增强，使其可以使用 `Object` 和 `Array`

原理是根据传入 `createVNode` 中的 `props` 进行对 class 和 style 的预处理。class 和 style 处理过程相似，这里就只写 class 处理过程

```typescript
// vnode.ts
function createBaseVNode(type, props, children, shapeFlag): VNode {
  if (props) {
    let { class: klass, style } = props
    if (klass && !isString(klass)) {
      props.class = normalizeClass(klass)
    }
  }
  ...
}
  
// normalizeProps.ts
/**
 * 规范化 class 类，处理 class 的增强
 */
export function normalizeClass(value: unknown): string {
  let res = ''
  if (isString(value)) {
    return value
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      const normalize = normalizeClass(value[i])
      if (normalize) {
        res += normalize + ' '
      }
    }
  } else if (isObject(value)) {
    for (const name in value) {
      if (value[name]) {
        res += name + ' '
      }
    }
  }
  return res.trim()
}
```

