# compiler 编译器 - 根据 JavaScript AST 对象 生成 render 函数

1. # 解析 JavaScript AST 拼接 render 函数

   这里的 render 函数 实际上就是一个字符串最终解析的函数如下：

   ```javascript
   const _Vue = Vue
   
   return function render(_ctx, _cache) {
       const { createElementVNode: _createElementVNode } = _Vue
   
       return _createElementVNode("div", [], [" hello world "])
   }
   ```

   ## 解析实现：

   ```typescript
   // utils.ts
   export function getVNodeHelper(ssr: boolean, isComponent: boolean) {
     return ssr || isComponent ? CREATE_VNODE : CREATE_ELEMENT_VNODE
   }
   
   import { isArray, isString } from '@vue/shared'
   import { NodeTypes } from './ast'
   import { helperNameMap } from './runtimeHelpers'
   import { getVNodeHelper } from './utils'
   
   export interface CodegenContext {
     // render 函数代码字符串
     code: string
     // 运行时全局的变量名
     runtimeGlobalName: 'Vue'
     // 模板源
     source: any
     isSsr: boolean
     // 缩进级别
     indentLevel: number
     // 需要触发的方法，关联 JavaScript AST 中的 helpers
     helper: (key) => string
     /**
      * 插入代码
      */
     push: (code: string, node?: any) => void
     /**
      * 新的一行
      */
     newline: Function
     /**
      * 控制缩进 + 换行
      */
     indent: Function
     /**
      * 控制缩进 + 换行
      */
     deindent: Function
   }
   
   const aliasHelper = (s: symbol) => `${helperNameMap[s]}: _${helperNameMap[s]}`
   
   export function createCodegenContext(ast): CodegenContext {
     const context: CodegenContext = {
       // render 函数代码字符串
       code: ``,
       // 运行时全局的变量名
       runtimeGlobalName: 'Vue',
       isSsr: false,
       // 模板源
       source: ast.loc.source,
       // 缩进级别
       indentLevel: 0,
       // 需要触发的方法，关联 JavaScript AST 中的 helpers
       helper(key) {
         return `_${helperNameMap[key]}`
       },
       /**
        * 插入代码
        */
       push(code) {
         context.code += code
       },
       /**
        * 新的一行
        */
       newline() {
         newline(context.indentLevel)
       },
       /**
        * 控制缩进 + 换行
        */
       indent() {
         newline(++context.indentLevel)
       },
       /**
        * 控制缩进 + 换行
        */
       deindent() {
         newline(--context.indentLevel)
       }
     }
   
     function newline(n: number) {
       context.code += '\n' + '  '.repeat(n)
     }
   
     return context
   }
   
   /**
    * 根据 JavaScript AST 生成
    */
   export function generate(ast) {
     // 生成上下文 context
     const context = createCodegenContext(ast)
     // 获取 code 拼接方法
     const { newline, push, indent, deindent } = context
   
     // 生成函数的前置代码：const _Vue = Vue
     genFunctionPreamble(context)
   
     // 创建方法名称
     const functionNmae = `render`
     // 创建方法参数
     const args = ['_ctx', '_cache']
     const signature = args.join(', ')
   
     // 利用方法名称和参数拼接函数声明
     push(`function ${functionNmae}(${signature}) {`)
     // 缩进 + 换行
     indent()
   
     // 明确使用到的方法。如：createVNode
     const hasHelpers = ast.helpers.length > 0
     if (hasHelpers) {
       push(`const { ${ast.helpers.map(aliasHelper).join(', ')} } = _Vue`)
       push(`\n`)
       newline()
     }
   
     // 最后拼接 return 的值
     newline()
     push(`return `)
   
     // 处理 renturn 结果。如：_createElementVNode("div", [], [" hello world "])
     if (ast.codegenNode) {
       genNode(ast.codegenNode, context)
     } else {
       push(`null`)
     }
   
     // 收缩缩进 + 换行
     deindent()
     push(`}`)
   
     return {
       ast,
       code: context.code
     }
   }
   
   /**
    * 生成 "const _Vue = Vue\n\nreturn "
    */
   function genFunctionPreamble(context: CodegenContext) {
     const { push, runtimeGlobalName, newline } = context
     const VueBinding = runtimeGlobalName
     push(`const _Vue = ${VueBinding}\n`)
     newline()
     push(`return `)
   }
   
   /**
    * 区分节点进行处理
    */
   function genNode(node, context: CodegenContext) {
     switch (node.type) {
       case NodeTypes.VNODE_CALL:
         genVNodeCall(node, context)
         break
       case NodeTypes.TEXT:
         genText(node, context)
         break
     }
   }
   
   /**
    * 处理 TEXT 节点
    */
   function genText(node, context: CodegenContext) {
     context.push(JSON.stringify(node.content))
   }
   
   /**
    * 处理 VNODE_CALL 节点
    */
   function genVNodeCall(node, context: CodegenContext) {
     const { push, helper } = context
     const { tag, props, children, patchFlag, dynamicProps, isComponent } = node
   
     const callHelper = getVNodeHelper(context.isSsr, isComponent)
   
     push(helper(callHelper) + `(`)
   
     const args = genNullableArgs([tag, props, children, patchFlag, dynamicProps])
   
     genNodeList(args, context)
   
     push(`)`)
   }
   
   /**
    * 处理 createXXXVnode 函数参数
    */
   function genNullableArgs(args: any[]) {
     let i = args.length
     while (i--) {
       if (args[i] != null) break
     }
     return args.slice(0, i + 1).map(arg => arg || `null`)
   }
   
   /**
    * 处理参数的填充
    */
   function genNodeList(nodes, context: CodegenContext) {
     const { push } = context
     // node 是字符串
     for (let i = 0; i < nodes.length; i++) {
       const node = nodes[i]
       if (isString(node)) {
         // 字符串直接 push 即可
         push(node)
       }
       // 数组需要 push "[" "]"
       else if (isArray(node)) {
         genNodeListAsArray(node, context)
       }
       // 对象需要区分 node 节点类型，递归处理
       else {
         genNode(node, context)
       }
       if (i < nodes.length - 1) {
         push(', ')
       }
     }
   }
   
   function genNodeListAsArray(nodes, context: CodegenContext) {
     context.push(`[`)
     genNodeList(nodes, context)
     context.push(`]`)
   }
   ```

   ## 导出 compile 函数

   新建一个包 `\packages\vue-compat\src` 在这里面处理函数转换操作：

   ```typescript
   
   function compileToFunction(template:string, options?) {
     const { code } = compile(template, options)
     const render = new Function(code)()
     return render
   }
   
   export { compileToFunction as compile }
   ```
   

## compiler 编译器 - 小结

到这里就已经完成了一个基础的编辑器处理。

整个编辑器的处理过程分成了三部分：

1. 解析模板 `template` 为 `AST` 

   1. 在这一步过程中，使用了一下两个概念 

      - 有限自动状态机解析模板得到了 `tokens`

      - 通过扫描 `tokens` 最终得到了 `AST`

2. 转化 `AST`  为 `JavaScript AST` 

   1. 这一步是为了最终生成 `render` 函数做准备
   2.  利用了深度优先的方式，进行了自下向上的逐层转化

3. 生成 `render` 函数 

   1. 这一步是最后的解析环节，需要对 `JavaScript AST` 进行处理，得到最终的 `render` 函数

整个一套编辑器的流程非常复杂，目前只完成了最基础的编辑逻辑，目前只能支持 `<div>文本</div>` 的处理。如果想要处理更复杂的逻辑，比如：

1. 响应性数据
2. 多个子节点
3. 指令

的话需要进行更多处理。

# 复杂情况下的 compiler 处理

1. ## 响应性数据

   1. 解决 AST 处理响应性数据的解析逻辑

      ```typescript
      function parseChildren(context: ParserContext, ancestors) {
          ...
          // 判断是否需要解析插值和标签开始
          if (startsWith(s, '{{')) {
            // TODO: 处理插值
            node = parseInterpolation(context)
          } else if (s[0] === '<') {
          	...
          }
          ...
      }
      
      /**
       * 解析插值表达式 {{ xxx }}
       */
      function parseInterpolation(context: ParserContext) {
        const [open, close] = ['{{', '}}']
        advanceBy(context, open.length)
      
        // 获取插值表达式中间的值
        const closeIndex = context.source.indexOf(close, open.length)
        const preTrimContent = parseTextData(context, closeIndex)
        const content = preTrimContent.trim()
      
        advanceBy(context, close.length)
        return {
          type: NodeTypes.INTERPOLATION,
          content: {
            type: NodeTypes.SIMPLE_EXPRESSION,
            isStatic: false,
            content
          }
        }
      }
      ```

   2. 处理 `JavaScript AST` 转换逻辑: 增加一个插值表达式的处理，再新增一个 helpers

      ```typescript
      export function traverseNode(node, context: TransformContext) {
        ...
        // 转化所有子节点
        switch (node.type) {
          case NodeTypes.ELEMENT:
          case NodeTypes.ROOT:
            traverseChildren(node, context)
            break
          case NodeTypes.INTERPOLATION:
            context.helper(TO_DISPLAY_STRING)
            break
        }
        ....
      }
      
      // runtimeHelper.ts
      export const TO_DISPLAY_STRING = Symbol('toDisplayString')
      export const helperNameMap = {
       ...
       [TO_DISPLAY_STRING]: 'toDisplayString'
      }
      
      ```

   3. 处理 generate 生成 render 函数: 完成 `generate` 的函数拼接

      1.  在 `generate` 方法中，新增 `with` 的处理：

         ```typescript
         export function generate(ast) {
         	...
         
         	// 增加 with 触发（加到 hasHelpers 之前）
         	push(`with (_ctx) {`)
         	indent()
         
         	// 明确使用到的方法。如：createVNode
         	const hasHelpers = ast.helpers.length > 0
         	...
         
         	// with 结尾
         	deindent()
         	push(`}`)
         
         	// 收缩缩进 + 换行
         	...
         
         	return {
         		ast,
         		code: context.code
         	}
         }
         ```

      2.  在 `genNode` 中处理其他节点类型： 

         ```typescript
         function genNode(node, context: CodegenContext) {
           switch (node.type) {
         	...
             // 简单表达式处理
             case NodeTypes.SIMPLE_EXPRESSION:
               genExpression(node, context)
               break
             // {{}} 处理
             case NodeTypes.INTERPOLATION:
               genInterpolations(node, context)
               break
             // 复合表达式处理
             case NodeTypes.COMPOUND_EXPRESSION:
               genCompoundExpression(node, context)
               break
           }
         }
         // 简单表达式处理
         function genExpression(node, context: CodegenContext) {
           const { content, isStatic } = node
           context.push(isStatic ? JSON.stringify(content) : content)
         }
         // {{}} 处理
         function genInterpolations(node, context: CodegenContext) {
           const { push, helper } = context
           push(`${helper(TO_DISPLAY_STRING)}(`)
           genNode(node.content, context)
           push(`)`)
         }
         // 复合表达式处理
         function genCompoundExpression(node, context: CodegenContext) {
           for (let i = 0; i < node.children.length; i++) {
             const child = node.children[i]
             if (isString(child)) {
               context.push(child)
             } else {
               genNode(child, context)
             }
           }
         }
         ```

2. ## 多个子节点

   想要处理子节点渲染时，需要处理当前的 `type = 1` 的节点：

   `type = 1` 对应的是 `NodeTypes.ELEMENT` 节点，因此需要在 `codegen.ts` 文件中处理 `genNode` 中的分支情况

   ```typescript
   /**
    * 区分节点进行处理
    */
   function genNode(node, context) {
   	switch (node.type) {
   		case NodeTypes.ELEMENT:
   			genNode(node.codegenNode!, context)
   			break
   		...
   	}
   }
   ```

3. ## 指令 v-if

   1. 更新 `AST` 的生成

      ```typescript
      /**
       * 解析开始标签
       */
      function parseTag(context: ParserContext, type: TagType) {
        // 处理 标签中的属性 和 vue指令
        advanceSpaces(context)
        let props = parseAttributes(context, type)
        ...
        return {
          ...
          props
        }
      }
      
      /**
       * 去除空格
       */
      function advanceSpaces(context: ParserContext) {
        const match = /^[\r\n\t\f ]+/.exec(context.source)
        if (match) {
          advanceBy(context, match[0].length)
        }
      }
      
      function parseAttributes(context: ParserContext, type: TagType) {
        // 解析后的 props 数组
        const props: any = []
        // 属性名数组
        const attributeNames = new Set<string>()
        while (
          context.source.length > 0 &&
          !startsWith(context.source, '>') &&
          !startsWith(context.source, '/>')
        ) {
          const attr = parseAttribute(context, attributeNames)
          if (type === TagType.Start) {
            props.push(attr)
          }
          advanceSpaces(context)
        }
        console.log(props)
        return props
      }
      
      function parseAttribute(context: ParserContext, nameSet: Set<string>) {
        const match = /^[^\t\r\n\f />][^\t\r\n\f />=]*/.exec(context.source)!
        const name = match[0]
        nameSet.add(name)
      
        advanceBy(context, name.length)
        // 获取属性值。
        let value: any = undefined
      
        // 解析模板，并拿到对应的属性值节点
        if (/^[\t\r\n\f ]*=/.test(context.source)) {
          advanceSpaces(context)
          advanceBy(context, 1)
          advanceSpaces(context)
          value = parseAttributeValue(context)
        }
        // 针对 v- 的指令处理
        if (/^(v-[A-Za-z0-9-]|:|\.|@|#)/.test(name)) {
          // 获取指令名称
          const match =
            /(?:^v-([a-z0-9-]+))?(?:(?::|^\.|^@|^#)(\[[^\]]+\]|[^\.]+))?(.+)?$/i.exec(
              name
            )!
          // 指令名。v-if 则获取 if
          let dirName = match[1]
          // TODO：指令参数  v-bind:arg
          // let arg: any
      
          // TODO：指令修饰符  v-on:click.modifiers
          // const modifiers = match[3] ? match[3].slice(1).split('.') : []
          return {
            type: NodeTypes.DIRECTIVE,
            name: dirName,
            exp: value && {
              type: NodeTypes.SIMPLE_EXPRESSION,
              content: value.content,
              isStatic: false,
              loc: value.loc
            },
            arg: undefined,
            modifiers: undefined,
            loc: {}
          }
        }
        return {
          type: NodeTypes.ATTRIBUTE,
          name,
          value: value && {
            type: NodeTypes.TEXT,
            content: value.content,
            loc: value.loc
          },
          loc: {}
        }
      }
      
      /**
       * 获取属性（attr）的 value
       */
      function parseAttributeValue(context: ParserContext) {
        let content = ''
        const quote = context.source[0]
        const isQuoted = quote === `"` || quote === `'`
        if (isQuoted) {
          advanceBy(context, 1)
          const endIndex = context.source.indexOf(quote)
          if (endIndex === -1) {
            content = parseTextData(context, context.source.length)
          } else {
            content = parseTextData(context, endIndex)
            advanceBy(context, 1)
          }
        }
        return {
          content,
          isQuoted,
          loc: {}
        }
      }
      ```

   2. `AST` 转换 `JavaScript AST` （好难啊！！！）
   
      ```typescript
      ```
   
   3. 解析成 `render` 函数
   
      ```typescript
      ```
   
      
   
   



