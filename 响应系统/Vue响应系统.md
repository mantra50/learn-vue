# 一、reactive的响应性实现

在源码内部要实现响应式功能，主要分为三个部分：

1. 创建 proxy。
2. 收集 effect 的依赖。
3. 触发收集到的依赖。

## 1. 创建 reactive 函数

- 这个函数中接收一个复杂数据类型 object 然后使用自定义的 **``createReactiveObject``**  方法进行处理

  ```typescript
  /**
   * 为复杂数据类型，创建响应性对象
   * @param target 被代理对象
   * @returns 代理对象
   */
  export function reactive(target: object) {
  	// 代理对象生成器
    return createReactiveObject(target, mutableHandlers, reactiveMap)
  }
  ```

- 缓存对象  ``reactiveMap`` 以及处理函数 ``createReactiveObject`` 和 ``porxyHandlers`` 

  ```typescript
  /**
   * 缓存代理对象
   * key: target
   * value: proxy
   */
  const reactiveMap = new WeakMap()
  
  /**
   * 创建响应性对象
   * @param target 被代理对象
   * @param baseHandlers get set 代理方法
   * @param proxyMap 缓存代理对象
   *
   * @returns proxy代理对象
   */
  function createReactiveObject(
    target: object,
    baseHandlers: ProxyHandler<any>,
    proxyMap: WeakMap<object, any>
  ) {
    // 检查是否存在缓存
    const existingProxy = proxyMap.get(target)
    if (existingProxy) {
      return existingProxy
    }
    // 创建代理对象 并设置缓存
    const proxy = new Proxy(target, baseHandlers)
    proxyMap.set(target, proxy)
    return proxy
  }
  
  /**
   * 响应性的 handler
   */
  export const mutableHandlers: ProxyHandler<object> = {
      // TODO:设置 set 和 get 函数
  }
  ```

## 2. 在 ``mutableHandlers`` 中的 ``get`` 里收集依赖，``set`` 里触发依赖

1. **``get``** 函数

   ```typescript
   const get = createGetter()
   
   /**
    * 创建 getter 回调方法
    * @returns function get():any
    */
   function createGetter() {
     return function get(target: object, key: string | symbol, receiver: object) {
       const res = Reflect.get(target, key, receiver)
       // TODO: 收集依赖
   	track(traget,key)
       return res
     }
   }
   ```

2. **``set``** 函数

   ```typescript
   const set = createSetter()
   
   /**
    * 创建 setter 回调方法
    * @returns function set(): boolean
    */
   function createSetter() {
     return function set(
       target: object,
       key: string | symbol,
       value: unknown,
       receiver: object
     ) {
       const res = Reflect.set(target, key, value, receiver)
       // TODO: 触发依赖
   	trigger(track,key)
       return res
     }
   }
   
   ```

3. 依赖收集 **``track``**

   ```typescript
   /**
    * 用于收集依赖的方法
    * @param target WeakMap 的 key
    * @param key 代理对象的 key，当依赖被触发时，需要根据该 key 获取
    */
   export function track(target: object, key: unknown) {
     // 依赖收集
     console.log('收集依赖')
   }
   ```

4. 触发依赖 **``trigger``**

   ```typescript
   /**
    * 触发依赖的方法
    * @param target WeakMap 的 key
    * @param key 代理对象的 key，当依赖被触发时，需要根据该 key 获取
    * @param newValue 指定 key 的最新值
    * @param oldValue 指定 key 的旧值
    */
   export function trigger(target: object, key?: unknown, newValue?: unknown) {
     // 触发依赖
     console.log('触发依赖')
   }
   
   ```

## 3. 初步完成 effect 函数 以及创建 `ReactiveEffect` 类

```typescript
/**
 * effect 函数
 * @param fn 执行函数
 * @returns 以 ReactiveEffect 实例为 this 的执行函数
 */
export function effect<T = any>(fn: () => T) {
  const _effect = new ReactiveEffect(fn)
  _effect.run()
}

/**
 * 单例的，当前的 effect
 */
export let activeEffect: ReactiveEffect | undefined

/**
 * 响应性触发依赖时的执行类
 */
export class ReactiveEffect<T = any> {
  constructor(public fn: () => T) {}
  run() {
    activeEffect = this
    return this.fn()
  }
}
```

## 4. 创建依赖收集的 ``WeakMap`` 并设置依赖的收集与触发

创建Map并收集依赖 并设置 依赖触发

```typescript
type KeyToDepMap = Map<any, ReactiveEffect>
/**
 * 收集所有依赖的 WeakMap 实例：
 * 1. `key`：响应性对象
 * 2. `value`：`Map` 对象
 * 		1. `key`：响应性对象的指定属性
 * 		2. `value`：指定对象的指定属性的 执行函数
 */
const targetMap = new WeakMap<any, KeyToDepMap>()

export function track(target: object, key: unknown) {
  // 如果当前不存在 activeEffect，则不需要收集依赖
  if (!activeEffect) return
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  depsMap.set(key, activeEffect)
}

export function trigger(target: object, key?: unknown, newValue?: unknown) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return
  const effect = depsMap.get(key) as ReactiveEffect
  if (!effect) return
  effect.run()
}
```

## 5. 单一依赖的 reactive

构建了一个简单的 `reactive` 函数，使用 `reactive` 函数，配合 `effect` 可以实现出一个 **响应式数据渲染功能**

1. 创建了一个 `reactive` 函数，该函数可以帮助生成一个 `proxy` 实例对象。
2. 通过该 `proxy` 实例的 `handler` 可以监听到对应的 `getter` 和 `setter。`
3. 然后创建了一个 `effect` 函数，通过该函数可以创建一个 `ReactiveEffect` 的实例，该实例的构造函数可以接收传入的回调函数 `fn`，并且提供了一个 `run` 方法。
4. 触发 `run` 可以为 `activeEffect` 进行赋值，并且执行 `fn` 函数。
5. 需要在 `fn` 函数中触发 `proxy` 的 `getter`，以此来激活 `handler` 的 `get` 函数。
6. 在 `handler` 的 `get` 函数中，通过 `WeakMap` 收集了 **指定对象，指定属性** 的 `fn`，这样的操作，可以把它叫做 **依赖收集。**
7. 可以在 **任意时刻**，修改 `proxy` 的数据，这样会触发 `handler` 的 `setter`。
8. 在 `handler` 的 `setter` 中，可以根据 **指定对象** ``target``的 **指定属性** `key` 来获取到保存的 **依赖**，然后只需要触发依赖中的``fn``，即可达到修改数据的效果。

## 6. 多依赖收集 响应数据对应多个 effect

1. 上面的 ``WeakMap`` 的 value 不再是 ``ReactiveEffect`` 而是一个 ``ReactiveEffect`` 的 ``Set `` 数组。因此可以创建一个新方法来创建这个 ``Set``数组

```typescript
export type Dep = Set<ReactiveEffect>

export function createDep(effects?: ReactiveEffect[]): Dep {
  const dep = new Set<ReactiveEffect>(effects) as Dep
  return dep
}
```

2. 改造 ``track`` 和  ``trigger`` 

```typescript
type KeyToDepMap = Map<any, Dep>

export function track(target: object, key: unknown) {
    ...
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = createDep()))
  }
  trackEffects(dep)
}
/**
 * 添加依赖的方法
 */
function trackEffects(dep: Dep) {
  dep.add(activeEffect!)
}

export function trigger(target: object, key?: unknown, newValue?: unknown) {
    ...
  // 依据指定的 key 获取 dep 实例
  let dep: Dep | undefined = depsMap.get(key)
  if (!dep) return
  triggerEffects(dep)
}
/**
 * 依次触发 dep 中保存的依赖
 */
export function triggerEffects(dep: Dep) {
  // 把 dep 构建为一个数组
  const effects = isArray(dep) ? dep : [...dep]
  for (const effect of effects) {
    triggerEffect(effect)
  }
}
/**
 * 触发指定的依赖
 */
export function triggerEffect(effect: ReactiveEffect) {
  effect.run()
}
```

## 7. 总结

1. **因为 ``Proxy`` 的原因，``reactive`` 方法只能处理复杂数据类型，无法处理简单数据类型。**
2. **通过 ``effect`` 函数 来创建一个类( ``ReactiveEffect`` )，这个类包含一个在响应式数据被修改时需要运行的函数**。
3. **通过 ``WeakMap`` 来存储响应性数据和它被使用时触发的方法进行一个关系映射，这个 ``Prox`` 作为 ``key`` 而第二步创建的 ``ReactiveEffect`` 作为 ``value`` 由于会出现一个数据多次被调用，所以这里的 ``value`` 应该是一个不重复的 ``ReactiveEffect`` 数组**。



# 二、ref的响应式实现

ref分为 简单数据类型响应性 和 复杂类型数据响应性

## 1. 复杂数据类型

ref 的复杂数据类型响应性 主要实现是依赖 reactive 进行实现，主要实现过程分为以下步骤：

1. 创建 ref 函数 和 Ref interface

```typescript
export interface Ref<T = any> {
  value: T
}

/**
 * ref 函数
 * @param value unknown
 */
export function ref(value?: unknown) {
  return createRef(value, false)
}
```

2. 创建 ``createRef`` 函数，使用 ``isRef`` 函数判断是否已经是 ref 数据。若不是则返回 ``RefIml`` 对象

```typescript
/**
 * 创建 RefImpl 实例
 * @param rawValue 原始数据
 * @param shallow boolean 形数据，表示《浅层的响应性（即：只有 .value 是响应性的）》
 * @returns
 */
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  private _value: T

  public dep?: Dep = undefined

  // 是否为 ref 类型数据的标记
  public readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean) {
    // 如果 __v_isShallow 为 true，则 value 不会被转化为 reactive 数据，即如果当前 value 为复杂数据类型，则会失去响应性。
    this._value = __v_isShallow ? value : toReactive(value)
  }

  /**
   * get语法将对象属性绑定到查询该属性时将被调用的函数。
   * 即：xxx.value 时触发该函数
   */
  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {}
}

/**
 * 指定数据是否为 RefImpl 类型
 */
export function isRef(r: any): r is Ref {
  return !!(r && r.__v_isRef === true)
}

```

3. 创建 ref 的 依赖收集函数 ``trackRefValue`` 和 转换函数 ``toReactive``

```typescript
// ref.ts 文件
/**
 * 为 ref 的 value 进行依赖收集工作
 */
export function trackRefValue(ref: any) {
  if (activeEffect) {
    trackEffects(ref.dep || (ref.dep = createDep()))
  }
}

// reactive.ts 文件
/**
 * 将指定数据变为 reactive 数据
 */
export const toReactive = <T extends unknown>(target: T): T =>
  isObject(target) ? reactive(target as object) : target

// 公共文件
export const isObject = (obj: any) => obj !== null && typeof obj === 'object'
```

## 2. 简单数据类型

**简单数据类型，不具备数据件监听的概念**，即本身并不是响应性的。因为`vue` 只是通过 `set value()` 的语法，把 **函数调用变成了属性调用的形式**，然后通过主动调用该函数，来完成一个“类似于”响应性的结果。

为 `ref.ts` 文件添加以下代码：

```typescript
class RefImpl<T> {
  private _rawValue:T
    ...
  constructor(value: T,...) {
    this._rawValue = value
	...
  }
  set value(newVal) {
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = toReactive(newVal)
      triggerRefValue(this)
    }
  }
}

/**
 * 触发 ref 的依赖
 * @param ref ref 实例
 */
export function triggerRefValue(ref: any) {
  if (ref.dep) {
    triggerEffects(ref.dep)
  }
}

// 公共方法
export const hasChanged = (newValue: any, oldValue: any): boolean =>
  !Object.is(newValue, oldValue)

```

## 3. 总结

1. `ref` 函数就是生成了一个 `RefImpl` 类型的实例对象，这个对象包含被 `set` 和`get` 标记的两个 value 函数，这两个函数就是 `ref` 让简单数据类型实现 “类似于” 响应性的方式。
2.  `ref` 复杂数据类型其实也是通过 `reactive` 来实现响应性。
3. `ref` 之所以需要使用 `.vlaue` 来访问值，是因为对简单数据类型来说，它无法通过 `proxy` 建立代理，所以需要通过 `get value()` 和 `set value()` 两个属性函数，来主动触发 **依赖收集** 和 **触发依赖**。因此 `ref` 必须通过 `.value` 来访问值，以确保证响应性。



# 三、computed 实现

## 1. 读取计算属性的值

```typescript
export type ComputedGetter<T> = (...args: any[]) => T

/**
 * 计算属性类
 */
export class ComputedRefImpl<T> {
  private _value!: T
  public dep?: Dep = undefined

  public readonly effect: ReactiveEffect<T>

  public readonly __v_isRef = true

  constructor(getter: ComputedGetter<T>) {
    this.effect = new ReactiveEffect(getter)
    this.effect.computed = this
  }
  get value() {
    trackRefValue(this)
    this._value = this.effect.run()
    return this._value
  }
}

/**
 * 计算属性
 */
export function computed<T>(getterOrOptions: ComputedGetter<T>) {
  let getter

  const onlyGetter = isFunction(getterOrOptions)
  if (onlyGetter) {
    getter = getterOrOptions
  }
  const cRef = new ComputedRefImpl(getter)
  return cRef
}
```

## 2. 完成 computed 响应性功能

- 在 `ComputedRefImpl` 的构造函数中 通过一个 `_dirty` 属性进行状态改变时 触发依赖的行为

  ```typescript
  constructor(getter: ComputedGetter<T>) {
      this.effect = new ReactiveEffect(getter, () => {
        if (!this._dirty) {
          this._dirty = true
          triggerRefValue(this)
        }
      })
    }
  ```

  

- 相对应的 `get value()` 函数也应该使用 `_dirty` 属性进行判断 是否需要改变 `_value` 的值。如此即可完成 computed 的缓存

  ```typescript
  get value() {
      trackRefValue(this)
      if (this._dirty) {
        this._dirty = false
        this._value = this.effect.run()
      }
      return this._value
    }
  ```

  

- 以上改动依赖于以下操作：

  1. scheduler：调度器，创建 `ReactiveEffect` 的第二个参数，如果一个 `effect` 中包含了这个，则会执行调度器，若不包含则执行 `run` 函数

  ``` typescript
  // class ReactiveEffect
  export type EffectScheduler = (...args: any[]) => any
  constructor(
    ...
      public scheduler: EffectScheduler | null = null
  ){}
  
  // fn triggerEffect
  function triggerEffect(effect: ReactiveEffect) {
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      effect.run()
    }
  }
  ```

- 以上功能完成后就可以实现 computed 对应的响应性功能。但是现在有一个问题，在进行 `set` 操作时，会出现循环触发 computed 的回调函数，这是因为`_dirty` 属性不断地被改变，引起 `triggerRefValue` 不断触发。需要解决此问题只需按以下步骤操作

  ```typescript
  /**
   * 依次触发 dep 中保存的依赖
   */
  export function triggerEffects(dep: Dep) {
   ...
   // 如果存在 computed，则先触发 computed
    for (const effect of effects) {
      if (effect.computed) {
        triggerEffect(effect)
      }
    }
    for (const effect of effects) {
      if (!effect.computed) {
        triggerEffect(effect)
      }
    }
  }
  ```

  原理是在第一遍循环当中只有 computed 里的 effect 会被触发，而不会调用 `get` 操作，从而不会将 `_dirty` 属性变为 `false` ，也就不会不断触发依赖，从而跳出死循环

  至此 computed 的响应性功能全部完成。

## 3. 总结

1. computed 的实现方式本质上也是创建了一个 `ComputedRefImpl` 的实例。
2. computed 的响应性方式和 Ref 十分类似，都是通过 `.value` 访问属性来手动触发依赖收集和触发依赖，只是在触发依赖时是通过 `调度器` 来控制的。
3. 在触发依赖时需要注意：**先触发有 computed 的依赖，再触发非 computed 的依赖**。



# 四、watch 监听器

## 1. 实现前置 — scheduler 调度器

1. 懒执行: 为 `effect` 添加配置项参数，以实现懒执行功能

   ```typescript
   interface ReactiveEffectOptions {
     lazy?: boolean
     scheduler?: EffectScheduler
   }
   function effect<T = any>(fn: () => T, options?: ReactiveEffectOptions) {
     const _effect = new ReactiveEffect(fn)
     if (!options || !options.lazy) {
       _effect.run()
     }
   }
   ```

2. 调度器：根据希望的执行顺序和执行规则进行依赖的执行。需要完成一下两个功能。

   1. 控制执行顺序：只需要合并配置项到 `effect` 就可以在执行相关依赖时触发 `scheduler` 而非 `run` 函数

      ```typescript
      function effect<T = any>(fn: () => T, options?: ReactiveEffectOptions) {
        ...
        if (options) {
          extend(_effect, options)
        }
        ...
      }
      ```

   2. 控制执行规则：通过 `promise.then` 实现一个微任务队列，以实现执行规则的控制

      ```typescript
      // 对应的 promise 的 pending 状态
      let isFlushPending = false
      
      /**
       * 一个 resolved 的 promise
       */
      const resolvedPromise = Promise.resolve() as Promise<any>
      
      /**
       * 当前执行的任务
       */
      let currentFlushPromise: Promise<void> | null = null
      
      /**
       * 待执行的任务队列
       */
      const pendingPreFlushCbs: Function[] = []
      
      /**
       * 队列预处理函数
       * @param cb 待执行的任务
       */
      export function queuePreFlushCb(cb: Function) {
        queueCb(cb, pendingPreFlushCbs)
      }
      
      /**
       * 队列处理函数
       * @param cb 待执行的任务
       * @param pendingQuque 任务队列
       */
      function queueCb(cb: Function, pendingQuque: Function[]) {
        pendingQuque.push(cb)
        queueFlush()
      }
      
      /**
       * 依次处理队列中执行函数
       */
      function queueFlush() {
        if (!isFlushPending) {
          isFlushPending = true
          currentFlushPromise = resolvedPromise.then(flushJobs)
        }
      }
      
      /**
       * 处理队列
       */
      function flushJobs() {
        isFlushPending = false
        flushPreFlushCbs()
      }
      
      /**
       * 依次处理队列中的任务
       */
      function flushPreFlushCbs() {
        if (pendingPreFlushCbs.length) {
          let activePreFlushCbs = [...new Set(pendingPreFlushCbs)]
          pendingPreFlushCbs.length = 0
          for (let i = 0; i < activePreFlushCbs.length; i++) {
            activePreFlushCbs[i]()
          }
        }
      }
      
      ```

      通过以上两步操作就完成了整个调度器的功能。



## 2. 初步实现 watch 数据监听器

```typescript
import { EMPTY_OBJ, hasChanged } from '@vue/shared'
import { ReactiveEffect } from 'packages/reactivity/src/effect'
import { isReactive } from 'packages/reactivity/src/reactive'
import { queuePreFlushCb } from './scheduler'

export interface WatchOptions<immediate = boolean> {
  immediate?: immediate
  deep?: boolean
}

/**
 * watch 监听函数
 * @param source 监听的源
 * @param cb 触发的回调
 * @param options 配置项
 */
export function watch(source, cb: Function, options: WatchOptions) {
  return doWatch(source as any, cb, options)
}

/**
 * watch 函数的实现
 * @param source 监听的源
 * @param cb 触发的回调
 * @param options 配置项
 */
function doWatch(
  source,
  cb: Function,
  { immediate, deep }: WatchOptions = EMPTY_OBJ
) {
  // 触发 getter 的指定函数
  let getter: () => any

  // 判断是否为 reactive 对象
  if (isReactive(source)) {
    getter = () => source
    deep = true
  } else {
    getter = () => {}
  }

  // 存在回调函数且深度监听
  if (cb && deep) {
    // TODO: 主动循环所有 source 完成依赖收集
    const baseGetter = getter
    getter = () => baseGetter()
  }

  // 旧值
  let oldValue = {}
  // job 任务
  const job = () => {
    if (cb) {
      // watch(source, cb)
      const newValue = effect.run()
      if (deep || hasChanged(newValue, oldValue)) {
        cb(newValue, oldValue)
        oldValue = newValue
      }
    }
  }

  // 调度器
  let scheduler = () => queuePreFlushCb(job)

  const effect = new ReactiveEffect(getter, scheduler)

  if (cb) {
    if (immediate) {
      job()
    } else {
      oldValue = effect.run()
    }
  } else {
    effect.run()
  }

  return () => {
    effect.stop()
  }
}
```

此时的 `watch` 方法并没有进行依赖收集，且在 `watch` 函数中并没有 `getter` 行为，所以，需要在 `watch` 中主动触发 `getter` 行为以此达到依赖收集的目的。

```typescript
/**
 * 依次执行 getter，从而触发依赖收集
 */
export function traverse(value: unknown) {
  if (!isObject(value)) {
    return value
  }
  for (const key in value as object) {
    traverse((value as object)[key])
  }
  return value
}
```

以上就是 `watch` 响应性的主要功能实现方式。

## 3. 总结

1. `watch` 实现响应式的主要核心：一个是 **调度器** 一个是 `traverse` ，调度器 是用来触发依赖 `traverse` 则是用来手动触发 `get` 行为来收集依赖。



# 五、响应性系统总结

1. 整个响应系统分成了四个：
   1. reactive
   2. ref
   3. computed
   4. watch
2. 响应式的核心 `API` 为 `Proxy`。整个 `reactive` 都是基于此来进行实现
3. `Porxy` 只能代理 **复杂数据类型**，所以延伸除了 `get value` 和 `set value` 这样 **以属性形式调用的方法**， `ref` 和 `computed`  之所以需要 `.value` 就是因为这样的方法

