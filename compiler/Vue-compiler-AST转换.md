# compiler 编译器 - AST 转换成 JavaScript AST 对象

## 一、整个 `transform` 的逻辑可以大致分为两部分： 

1. 深度优先排序，通过 `traverseNode` 方法，完成排序逻辑

   ```typescript
   /**
    * 遍历转化节点，转化的过程一定要是深度优先的（即：孙 -> 子 -> 父），因为当前节点的状态往往需要根据子节点的情况来确定。
    * 转化的过程分为两个阶段：
    * 1. 进入阶段：存储所有节点的转化函数到 exitFns 中
    * 2. 退出阶段：执行 exitFns 中缓存的转化函数，且一定是倒叙的。因为只有这样才能保证整个处理过程是深度优先的
    */
   export function traverseNode(node, context: TransformContext) {
     // 通过上下文记录当前正在处理的 node 节点
     context.currentNode = node
     // 获取当前所有 node 节点的 transform 方法
     const { nodeTransforms } = context
     // 存储转化函数的数组
     const exitFns: any[] = []
     // 循环获取节点的 transform 方法，缓存到 exitFns 中
     for (let i = 0; i < nodeTransforms.length; i++) {
       const onExit = nodeTransforms[i](node, context)
       if (onExit) {
         if (isArray(onExit)) {
           exitFns.push(...onExit)
         } else {
           exitFns.push(onExit)
         }
       }
     }
   
     // 转化所有子节点
     switch (node.type) {
       case NodeTypes.ELEMENT:
       case NodeTypes.ROOT:
         traverseChildren(node, context)
         break
     }
   
     // 退出时执行所有 transform
     context.currentNode = node
     let i = exitFns.length
     while (i--) {
       exitFns[i]()
     }
   }
   
   /**
    * 循环处理子节点
    */
   export function traverseChildren(parent, context: TransformContext) {
     parent.children.forEach((node, index) => {
       context.parent = parent
       context.childIndex = index
       traverseNode(node, context)
     })
   }
   
   ```

2. 通过保存在 `nodeTransforms` 中的 `transformXXX` 方法，针对不同的节点，完成不同的处理

   ```typescript
   /**
    * 将相邻的文本节点和表达式合并为一个表达式。
    *
    * 例如:
    * <div>hello {{ msg }}</div>
    * 上述模板包含两个节点：
    * 1. hello：TEXT 文本节点
    * 2. {{ msg }}：INTERPOLATION 表达式节点
    * 这两个节点在生成 render 函数时，需要被合并： 'hello' + _toDisplayString(_ctx.msg)
    * 那么在合并时就要多出来这个 + 加号。
    * 例如：
    * children:[
    * 	{ TEXT 文本节点 },
    *  " + ",
    *  { INTERPOLATION 表达式节点 }
    * ]
    */
   export const transformText: NodeTransform = (
     node,
     context: TransformContext
   ) => {
     if (
       node.type === NodeTypes.ROOT ||
       node.type === NodeTypes.ELEMENT ||
       node.type === NodeTypes.FOR ||
       node.type === NodeTypes.IF_BRANCH
     ) {
       return () => {
         // 获取所有的子节点
         const children = node.children
         // 当前容器
         let currentCantainer
   
         // 循环处理所有的子节点
         for (let i = 0; i < children.length; i++) {
           const child = children[i]
           // 当前节点是 Text 节点
           if (isText(child)) {
             // j = i + 1 表示下一个节点
             for (let j = i + 1; j < children.length; j++) {
               const next = children[j]
               // 当前节点 child 和 下一个节点 next 都是 Text 节点
               if (isText(next)) {
                 if (!currentCantainer) {
                   // 生成一个复合表达式节点
                   currentCantainer = children[i] = createCompoundExpression(
                     [child],
                     child.loc
                   )
                 }
                 // 在 当前节点 child 和 下一个节点 next 中间，插入 "+" 号
                 currentCantainer.children.push(' + ', next)
                 // 把下一个删除
                 children.splice(j, 1)
                 j--
               }
               // 当前节点 child 是 Text 节点，下一个节点 next 不是 Text 节点，则把 currentContainer 置空即可
               else {
                 currentCantainer = undefined
                 break
               }
             }
           }
         }
       }
     }
   }
   
   export function createCompoundExpression(children, loc) {
     return {
       type: NodeTypes.COMPOUND_EXPRESSION,
       children,
       loc
     }
   }
   
   /**
    * 对 element 节点的转化方法
    */
   export const transformElement: NodeTransform = (
     node,
     context: TransformContext
   ) => {
     return function postTransformElement() {
       node = context.currentNode
       if (node.type !== NodeTypes.ELEMENT) {
         return
       }
       const { tag } = node
       let vnodeTag = `"${tag}"`
       let vnodeProps = []
       let vnodeChildren = node.children
       node.codegenNode = createVNodeCall(
         context,
         vnodeTag,
         vnodeProps,
         vnodeChildren
       )
     }
   }
   
   export function createVNodeCall(context, tag, props?, children?) {
     if (context) {
       context.helper(CREATE_ELEMENT_VNODE)
     }
     return {
       type: NodeTypes.VNODE_CALL,
       tag,
       props,
       children
     }
   }
   ```

   最后需要在 `transform` 函数中设置 根节点的 `helpers` 、`codegenNode` 等元素

   ```typescript
   export function transform(root, options) {
     const context = createTransformContext(root, options)
     traverseNode(root, context)
     createRootCodegen(root)
     root.helpers = [...context.helpers.keys()]
     root.components = []
     root.directives = []
     root.imports = []
     root.hoists = []
     root.temps = []
     root.cached = []
   }
   
   function createRootCodegen(root) {
     const { children } = root
     if (children.length === 1) {
       // 单节点情况
       const child = children[0]
       if (isSingleElementRoot(root, child)) {
         const codegenNode = child.codegenNode
         root.codegenNode = codegenNode
       }
     } else if (children.length > 1) {
       // vue3 支持多节点
     }
   }
   ```

