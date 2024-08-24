# compiler 编译器 - 生成 AST 对象

在 `vue` 中整个编译时的过程分为三大步：

1. 根据模板，生成 `AST` 抽象语法树
2. 转换 `AST` 得到 `JavaScript AST`
3. 生成 render 函数

这里所有的处理就是第一步

## 一、构建 compile 函数

`compiler-core/src/compile.ts` 文件

```typescript
import { baseParse } from "./parse";

export function baseCompile(template: string, options) {
  const ast = baseParse(template);
  return {};
}
```

`compiler-core/src/parse.ts`文件

```typescript
export interface ParserContext {
  source: string;
}
function createParserContext(content: string): ParserContext {
  return {
    source: content,
  };
}
export function baseParse(content: string) {
  const context = createParserContext(content);
  console.log(context);
  return {};
}
```

`compiler-dom/src/index.ts` 文件导出测试使用 `compile`

```typescript
import { baseCompile } from "packages/compiler-cor/src/compile";

export function compile(template: string, options) {
  return baseCompile(template, options);
}
```

## 二、实现 `parseChildren` 解析子节点函数

`parseChildren` 函数是整个 `compile` 生成 AST 的关键所在，整个处理过程分为两大部分：

1. 首先构建有限自动状态机来解析模板
2. 扫描 token 生成 AST 结构对象

创建 `parseChildren` 函数：

```typescript
function parseChildren(context: ParserContext, ancestors) {
  const nodes = [];

  /**
   * 循环解析所有 node 节点，可以理解为对 token 的处理。
   * 例如：<div>hello world</div>，此时的处理顺序为：
   * 1. <div
   * 2. >
   * 3. hello world
   * 4. </
   * 5. div>
   */
  while (!isEnd(context, ancestors)) {
    const s = context.source;
    let node;
    // 判断是否需要解析插值和标签开始
    if (startsWith(s, "{{")) {
      // TODO: 处理插值
    } else if (s[0] === "<") {
      if (/[a-z]i/.test(s[1])) {
        node = parseElement(context, ancestors);
      }
    }

    // 如果 node 不存在则可以确定这个 token 是文本
    if (!node) {
      node = parseText(context);
    }

    pushNode(nodes, node);
  }
  return nodes;
}
```

以上代码中涉及到了这几个方法：

1. `isEnd`：判断是否为结束节点

   ```typescript
   /**
    * 是否是结束标签
    */
   function isEnd(context: ParserContext, ancestors): boolean {
     const s = context.source;
     if (startsWith(s, "</")) {
       for (let i = ancestors.length - 1; i >= 0; i--) {
         if (startsWithEndTagOpen(s, ancestors[i].tag)) {
           return true;
         }
       }
     }
     return !s;
   }
   /**
    * 判断当前是否为《标签结束的开始》。比如 </div> 就是 div 标签结束的开始
    * @param source 模板。例如：</div>
    * @param tag 标签。例如：div
    * @returns
    */
   function startsWithEndTagOpen(source: string, tag: string): boolean {
     return (
       startsWith(source, "</") &&
       source.slice(2, 2 + tag.length).toLowerCase() === tag.toLowerCase() &&
       /[\t\r\n\f />]/.test(source[2 + tag.length] || ">")
     );
   }
   ```

2. `startsWith`：判断是否以指定文本开头

   ```typescript
   /**
    * 是否以指定文本开头
    */
   function startsWith(source: string, searchString: string): boolean {
     return source.startsWith(searchString);
   }
   ```

3. `pushNode`：为 `array` 执行 `push` 方法

   ```typescript
   function pushNode(nodes: any[], node: any) {
     nodes.push(node);
   }
   ```

4. **复杂：**`parseElement`：解析 `element`

   ```typescript
   /**
    * 解析标签
    */
   function parseElement(context: ParserContext, ancestors) {
     // 解析开始标签
     const element = parseTag(context, TagType.Start);
     // 解析子节点
     ancestors.push(element);
     const children = parseChildren(context, ancestors);
     ancestors.pop();
     element.children = children;
     // 解析结束标签
     if (startsWithEndTagOpen(context.source, element.tag)) {
       parseTag(context, TagType.End);
     }
     return element;
   }

   /**
    * 解析开始标签
    */
   function parseTag(context: ParserContext, type: TagType) {
     // 通过正则取出标签名
     const match = /^<\/?([a-z][^\r\n\t\f />]*)/i.exec(context.source)!;
     // 标签名
     const tag = match[1];
     // 右移原始模板内容
     advanceBy(context, match[0].length);
     // 是否为单标签
     let isSelfClosing = startsWith(context.source, "/>");
     // 去除标签
     advanceBy(context, isSelfClosing ? 2 : 1);

     const tagType = ElementTypes.ELEMENT;
     return {
       type: NodeTypes.ELEMENT,
       tag,
       tagType,
       props: [],
       children: [],
     };
   }
   ```

5. **复杂：**`parseText`：解析 `text`

   ```typescript
   /**
    * 解析文本
    */
   function parseText(context: ParserContext) {
     // 结束白名单
     const endTokens = ["<", "{{"];
     // 结束下标
     let endIndex = context.source.length;
     // 校准结束下标
     for (let i = 0; i < endTokens.length; i++) {
       const index = context.source.indexOf(endTokens[i]);
       if (index !== -1 && index < endIndex) {
         endIndex = index;
       }
     }
     // 拿到文本内容
     const content = parseTextData(context, endIndex);
     return {
       type: NodeTypes.TEXT,
       content,
     };
   }
   
   function parseTextData(context: ParserContext, length: number) {
     const rawText = context.source.slice(0, length);
     advanceBy(context, rawText.length);
     return rawText;
   }
   ```

## 三、将 `parseChildren` 的返回值 通过 `baseParse` 返回出去

```typescript
export function createRoot(children: any[]) {
  return {
    type: NodeTypes.ROOT,
    children,
    loc: {},
  };
}
export function baseParse(content: string) {
  const context = createParserContext(content);
  const children = parseChildren(context, []);
  return createRoot(children);
}
```
