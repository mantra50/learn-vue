# component 组件的设计原理和渲染方案

组件本身是一个对象（仅考虑对象的情况，忽略函数式组件）。它必须包含一个 `render` 函数，该函数决定了它的渲染内容。

如果想要定义数据，那么需要通过 `data` 选项进行注册。`data` 选项应该是一个 **函数**，并且 `renturn` 一个对象，对象中包含了所有的响应性数据。

除此之外，还可以定义例如 生命周期、计算属性、`watch` 等对应内容。

## 一、无状态基础组件挂载（无数据情况）

1. 首先需要在 `renderer.ts` 中的 `patch` 方法中进行 `component` 分支判断时触发 `processComponent` ：

   ```typescript
   else if (shapeFlag & ShapeFlags.COMPONENT) {
       // TODO：处理组件
       processComponent(oldVNode, newVNode, container, anchor)
   }
   ```

2. 创建 `processComponent`:

   ```typescript
   const processComponent = (
       oldVNode: VNode | null,
       newVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
   ) => {
       if (oldVNode == null) {
           // 挂载组件
           mountComponent(newVNode, container, anchor)
       }
   }
   ```

3. 创建挂载组件函数 `mountComponent`:

   ```typescript
   const mountComponent = (
       initialVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
   ) => {
       // 生成组件实例
       initialVNode.component = createComponentInstance(initialVNode)
       // 浅拷贝 绑定同一块内存空间
       const instance = initialVNode.component
       // 标准化组件实例数据
       setupComponent(instance)
       // 设置组件渲染
       setupRenderEffect(instance, initialVNode, container, anchor)
   }
   ```

4. 在 `component.ts` 文件中创建组件生成函数 `createComponentInstance`:

   ```typescript
   type Fn<T = void> = () => T
   // 组件实例的类型
   export interface ComponentInternalInstance {
     uid: number
     vnode: VNode
     type: any
     subTree: VNode | null
     effect: ReactiveEffect | null
     update: Function | null
     render: any | null
     isMounted: boolean
     data?: Fn<Record<string, any>>
   }
   let uid = 0
   export function createComponentInstance(
     vnode: VNode
   ): ComponentInternalInstance {
     const type = vnode.type
     const instance = {
       uid: uid++, // 唯一标记
       vnode, // 虚拟节点
       type, // 组件类型
       isMounted: false, // 是否挂载
       subTree: null!, // render 函数的返回值
       effect: null!, // ReactiveEffect 实例
       update: null!, // update 函数，触发 effect.run
       render: null // 组件内的 render 函数
     }
     return instance
   }
   ```

5. 在 `component.ts` 文件中创建规范化组件实例数据函数 `setupComponent`：

   ```typescript
   /**
    * 规范化组件实例数据
    */
   export function setupComponent(instance) {
     // 为 render 赋值
     const setupResult = setupStatefulComponent(instance)
     return setupResult
   }
   
   function setupStatefulComponent(instance: ComponentInternalInstance) {
     finishComponentSetup(instance)
   }
   
   export function finishComponentSetup(instance: ComponentInternalInstance) {
     const Component = instance.type
     instance.render = Component.render
     // TODO：处理有状态时的情况
     // applayOptions(instance)
   }
   ```

6. 在 `renderer.ts` 中创建设置组件渲染函数 `setupRenderEffect`:

   ```typescript
     const setupRenderEffect = (
       instance: ComponentInternalInstance,
       initialVNode: VNode,
       container: RendererElement,
       anchor: RendererNode | null
     ) => {
       const componentUpdateFn = () => {
           // 当前处于 mounted 之前，即执行 挂载 逻辑
         if (!instance.isMounted) {
           // 从 render 中获取需要渲染的内容
           const subTree = (instance.subTree = rendererComponentRoot(instance))
   		// 渲染组件
           patch(null, subTree, container, anchor)
           // 把组件根节点的 el，作为组件的 el
           initialVNode.el = subTree.el
         }else{
           // 更新
         }
       }
       // 创建包含 scheduler 的 effect 实例
       const effect = (instance.effect = new ReactiveEffect(
         componentUpdateFn,
         () => queuePreFlushCb(update)
       ))
       // 生成 update 函数
       const update = (instance.update = () => effect.run())
   
       // 触发 update 函数，本质上触发的是 componentUpdateFn
       update()
     }

7. 在 `componentRenderUtils.ts` 文件中创建 `rendererComponentRoot` 函数:

   ```typescript
   /**
    * 解析 render 函数的返回值
    * @param instance 组件实例
    * @returns 标准化 vnode
    */
   export function rendererComponentRoot(
     instance: ComponentInternalInstance
   ): VNode {
     const { vnode, render } = instance
     let result
     try {
       // 解析到状态组件
       if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
         result = normalizeVNode(render!())
       }
     } catch (error) {
       console.error(error)
     }
     return result
   }
   ```
   
   至此则完成了无状态组件的挂载操作。
   
8. 无状态组件更新操作，所谓的组件更新，其实本质上就是一个 **卸载**、**挂载** 的逻辑，这个操作已经在 `patch` 方法中进行了处理：

   ```typescript
   // patching & not same type, unmount old tree
   if (oldVNode && !isSameVNodeType(oldVNode, newVNode)) {
       // 通过卸载旧节点来完成更新
       unmount(oldVNode)
       oldVNode = null
   }
   ```

## 二、有状态组件（存在响应式数据时）

1. 完成在 `setupComponent` 中的 `applayOptions` 函数:

   ```typescript
   export function finishComponentSetup(instance: ComponentInternalInstance) {
     const Component = instance.type
     instance.render = Component.render
     // 改变 options 中的 this 指向
     applayOptions(instance)
   }
   function applyOptions(instance: ComponentInternalInstance) {
   	const { data: dataOptions } = instance.type
   	// 存在 data 选项时
   	if (dataOptions) {
   		// 触发 dataOptions 函数，拿到 data 对象
   		const data = dataOptions()
   		// 如果拿到的 data 是一个对象
   		if (isObject(data)) {
   			// 则把 data 包装成 reactiv 的响应性数据，赋值给 instance
   			instance.data = reactive(data)
   		}
   	}
   }
   ```

2. 在 `renderComponentRoot` 函数中通过 `call` 方法修改 `render` 的 `this` 的指向，将 `this` 指向 `component` 中的 `data` 响应式数据:

   ```typescript
   /**
    * 解析 render 函数的返回值
    * @param instance 组件实例
    * @returns 标准化 vnode
    */
   export function rendererComponentRoot(
     instance: ComponentInternalInstance
   ): VNode {
     const { vnode, render, data } = instance
     let result
     try {
       // 解析到状态组件
       if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
         result = normalizeVNode(render!.call(data))
       }
     } catch (error) {
       console.error(error)
     }
     return result
   }
   ```

   在完成以上操作后就可以渲染拥有数据的组件。

## 三、生命周期回调处理

生命周期回调处理分为两部分：`beforCreate` 、`created` 和其他生命周期，这些钩子函数都要先在 `applayOptions` 函数中进行处理

1.  `beforCreate` 和 `created`:

   ```typescript
   // 生命周期回调类型
   type LifecycleHook<TFn = Function> = TFn | null
   // 生命周期枚举类型
   export const enum LifecycleHooks {
     BEFORE_CREATE = 'bc',
     CREATE = 'c',
     BEFORE_MOUNT = 'bm',
     MOUNTED = 'm',
     BEFORE_UPDATE = 'bu'
   }
   // 更新类型
   export interface ComponentInternalInstance {
     ...
     data?: Fn<Record<string, any>>
     [LifecycleHooks.BEFORE_CREATE]: LifecycleHook
     [LifecycleHooks.CREATE]: LifecycleHook
     [LifecycleHooks.BEFORE_MOUNT]: LifecycleHook
     [LifecycleHooks.MOUNTED]: LifecycleHook
   }
   
   
   function applayOptions(instance: ComponentInternalInstance) {
     // 从 instance.type 中获取 data 选项 和 生命周期钩子函数
     const {
       data: dataOptions,
       beforeCreate,
       created,
       beforeMount,
       mounted
     } = instance.type
     // beforeCreate 生命周期
     if (beforeCreate) {
       callHook(beforeCreate)
     }
     // 如果存在 data 选项时 获取 data 的返回值，并将它包装成 proxy 赋值给 instance.data
     if (dataOptions) {
   	...
     }
     // create 生命周期
     if (created) {
       created()
     }
     // 注册生命周期钩子函数
     function registerLifecycleHooks(register: Function, hook?: Function) {
       register(hook, instance)
     }
     //注册生命周期
     registerLifecycleHooks(onBeforeMount, beforeMount)
     registerLifecycleHooks(onMounted, mounted)
   }
   // 触发器
   function callHook(hook: Function) {
     hook()
   }
   ```

   - 处理生命周期挂载 `apiLifecycle.ts` 文件:

     ```typescript
     import { ComponentInternalInstance, LifecycleHooks } from './component'
     // 将生命周期函数挂载到 instance 实例当中
     export function injectHook(
       lifecycle: LifecycleHooks,
       hook: Function,
       target: ComponentInternalInstance | null = null
     ) {
       if (target) {
         target[lifecycle] = hook
         return hook
       }
     }
     export const createHook =
       <T extends Function = () => any>(lifecycle: LifecycleHooks) =>
       (hook: T, target: ComponentInternalInstance | null = null) =>
         injectHook(lifecycle, hook, target)
     
     export const onBeforeMount = createHook(LifecycleHooks.BEFORE_MOUNT)
     export const onMounted = createHook(LifecycleHooks.MOUNTED)
     
     ```

2. 更改生命周期 `this` 指向：

   ```typescript
   function callHook(hook: Function, proxy) {
     hook.bind(proxy)()
   }
   
   function registerLifecycleHooks(register: Function, hook?: Function) {
     register(hook?.bind(instance.data), instance)
   }
   ```

## 四、组件响应性更新

在 `setupRenderEffect` 函数中开始处理更新操作：

本质上就是新创建了一个 `rendererComponentRoot` 放入 `nextTree` 并赋值给 `instance.subTree` 中，再使用 `patch` 进行更新打补丁操作，最后将 `nextTree.el` 赋值给 `next.el`

```typescript
  const setupRenderEffect = (
    instance: ComponentInternalInstance,
    initialVNode: VNode,
    container: RendererElement,
    anchor: RendererNode | null
  ) => {
    const componentUpdateFn = () => {
        // 当前处于 mounted 之前，即执行 挂载 逻辑
      if (!instance.isMounted) {
       ...
       instance.isMounted = false
      }else{
        let { next, vnode } = instance
        if (!next) {
          next = vnode
        }
        // 更新操作
        const nextTree = rendererComponentRoot(instance)
        const prevTree = instance.subTree
        instance.subTree = nextTree
        patch(prevTree, nextTree, container, anchor)
        next.el = nextTree.el
      }
    }
   ...
  }
```

## 五、setup 函数

`setup` 函数本质上就是在组件中设置了一个处理函数，可以在 `setup` 函数内部返回一个 `vnode` 节点，也就是相当于 `render` 函数，在这个函数里直接使用响应式数据，就可以完成对应的 **自洽** ，所以无需通过 `call` 方法来改变 `this` 指向，就可以拿到真实的 `render`

```typescript
function setupStatefulComponent(instance: ComponentInternalInstance) {
  const Component = instance.type
  const { setup } = Component
  // 是否拥有 setup 函数
  if (setup) {
    const setupResult = setup()
    handleSetupResult(instance, setupResult)
  } else {
    finishComponentSetup(instance)
  }
}

function handleSetupResult(
  instance: ComponentInternalInstance,
  setupResult: () => VNode
) {
  // 存在 setupResult，并且它是一个函数，则 setupResult 就是需要渲染的 render
  if (isFunction(setupResult)) {
    instance.render = setupResult
  }
  finishComponentSetup(instance)
}

export function finishComponentSetup(instance: ComponentInternalInstance) {
  ...
  // 如果不存在 render 函数，则直接赋值给 instance.render
  if (!instance.render) {
    instance.render = Component.render
  }
}
```

## 六、总结

1. 道组件本质上就是一个**对象（或函数）**，组件的渲染本质上是 `render` 函数返回值的渲染。
2. 组件渲染的内部，构建了 `ReactiveEffect` 的实例，其目的是为了实现组件的响应性渲染。
3. 在组件中访问响应性数据时的两种情况:
   1. 通过 `this` 访问：对于这种情况需要改变 `this` 指向，改变的方式是通过 `call` 方法或者 `bind` 方法
   2. 通过 `setup` 访问：这种方式因为不涉及到 `this` 指向问题，反而更加简单
4. 当组件内部的响应性数据发生变化时，会触发 `componentUpdateFn` 函数，在该函数中根据 `isMounted` 的值的不同，进行了不同的处理。
5. 组件的生命周期钩子，本质上只是一些方法的回调，当然，如果我们希望在生命周期钩子中通过 `this` 访问响应式数据，那么一样需要改变 `this` 指向。

