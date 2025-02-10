## 1 编译器是什么？

通俗的讲，编译器就是一段程序，将源代码翻译为目标代码。

## 2 为什么需要编译器？

Vue 之所以需要 **编译器** ，是因为它提供了一个基于 **模板** 的开发方式，而浏览器本身并不能直接解析 Vue 的模板语法。因此，Vue 需要一个编译器来将 **模板转换为 JavaScript 渲染函数（Render Function）** ，从而实现高效的 DOM 更新和页面渲染。

## 3 编译器的完整流程

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9c40524a0eb34ab7b841bc956ae8f983~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=K%2B2shZr%2FYPG9eZfrF5dE5pAU%2FIw%3D)

<p align=center>图1.1完整的编译流程</p>
从图中可以看出编译分为三个部分：
1. 用来将模板字符串解析为模板 AST 的解析器(parser)  
2. 用来将模板 AST 转换为 JavaScript AST 的转换器(transformer)
3. 用来根据 JavaScript AST 生成渲染函数代码的生成器(generator)

## 4 解析器parser

作用是将模版转换为模版AST(抽象语法树)。解析器又大致被分为两个流程，1.对模版进行切割，获取tokens。2。根据tokens构建模版AST

### 4.1 模版切割

这一过程是将模版字符串进行切割形成多个token。比如将模版字符串

    <h1>zhangsan</h1>

切割成tokens

    <h1>
    zhangsan
    </h1>

解析器对模版的切割是根据“**有限状态机**”进行的。所谓“有限状态”，就是指有限个状态，而“自动机”意味着随着
字符的输入，解析器会自动地在不同状态间迁移。
下图为解析器的状态机图。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/ac9b9a87bb724ba8808aa5f87e8a78ff~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=08dccTuO3%2FEgkWZ98ESbDoFUyP0%3D) <p align=center>图2 解析器的状态迁移过程</p>

图2给出的状态机并不严谨。实际上，解析HTML并构造Token的过程是有规范可循的。在WHATWG发布的关于浏览器解析HTML的规范中，详细阐述了状态迁移。

#### 4.1.1 为什么要使用有限状体机

**1.模板解析的复杂性**

Vue 模板的语法非常丰富，包含了 **HTML 标签**、**属性**、**指令**、**插值表达式**、**事件绑定**、**动态内容** 等。这些不同的语法结构需要通过编译器逐个识别和处理。为了做到这一点，编译器必须能够在不同的 **状态** 下进行不同的处理，比如：

*   在解析 **HTML 标签** 时，状态机需要知道是否已经进入一个标签。
*   在遇到 **指令（如 v-if）** 时，状态机需要切换到处理指令的状态。
*   在遇到 **插值（如 {{ message }}）** 时，状态机需要切换到解析表达式的状态。

**2.高效的字符扫描与状态管理**

有限状态机能够高效地处理模板字符串的扫描。在模板解析过程中，字符一个接一个地被读取，根据当前状态进行适当的处理，并转到下一个状态。有限状态机的状态转移过程非常 **快速** 和 **明确**，可以快速识别模板中的不同部分，减少了代码的复杂性。

**3.处理多种语法结构**

Vue 的模板不仅仅是静态的 HTML，还包括了动态内容和复杂的指令语法。有限状态机帮助编译器处理,通过状态机的状态转移，编译器能够根据不同的 **状态** 来决定如何解析不同的语法结构。

#### 4.1.2 简易实现

```js
      // 状态机的状态
      const State = {
        initial: 1, // 初始状态
        tagOpen: 2, // 标签开始状态
        tagName: 3, // 标签名称状态
        text: 4, // 文本状态
        tagEnd: 5, // 结束标签状态
        tagEndName: 6, // 结束标签名称状态
      };

      // 判断字符是否为字母
      function isLetter(char) {
        return (char >= "a" && char <= "z") || (char >= "A" && char <= "Z");
      }

      // 接收模板字符串作为参数，并将模板切割为 Token 返回
      function tokenize(str) {
        // 初始状态
        let currentState = State.initial;
        // 用于保存当前解析的字符
        const chars = [];
        // 存储最终的 tokens
        const tokens = [];
        //字符下标
        let index = 0;

        // 遍历字符串
        while (index < str.length) {
          const char = str[index];
          switch (currentState) {
            // 处于初始状态
            case State.initial:
              // 遇到 <
              if (char === "<") {
                // 切换到标签开始状态
                currentState = State.tagOpen;
              } else if (isLetter(char)) {
                // 若是字母，切换到文本状态
                currentState = State.text;
                // 将字母push到数组
                chars.push(char);
              }
              break;

            // 标签开始状态
            case State.tagOpen:
              if (isLetter(char)) {
                // 切换到标签名称状态
                currentState = State.tagName;
                chars.push(char);
              } else if (char === "/") {
                // 切换到结束标签状态
                currentState = State.tagEnd;
              }
              break;

            // 处于标签名称状态
            case State.tagName:
              if (isLetter(char)) {
                chars.push(char);
              } else if (char === ">") {
                // 将标签名称添加到 tokens 中
                tokens.push({
                  type: "tag",
                  name: chars.join(""),
                });
                chars.length = 0;
                //切换状态
                currentState = State.initial;
              }
              break;

            // 处于文本状态
            case State.text:
              if (isLetter(char)) {
                chars.push(char);
              } else if (char === "<") {
                // 将文本内容添加到 tokens 中
                tokens.push({
                  type: "text",
                  content: chars.join(""),
                });
                // 清空字符数组
                chars.length = 0;
                currentState = State.tagOpen;
              }
              break;

            // 处于标签结束状态
            case State.tagEnd:
              if (isLetter(char)) {
                currentState = State.tagEndName;
                chars.push(char);
              }
              break;

            // 处于结束标签名称状态
            case State.tagEndName:
              if (isLetter(char)) {
                chars.push(char);
              } else if (char === ">") {
                // 将结束标签名称添加到 tokens 中
                tokens.push({
                  type: "tagEnd",
                  name: chars.join(""),
                });
                chars.length = 0; // 清空字符数组
                currentState = State.initial;
              }
              break;
          }
          index++;
        }
        // 返回 tokens 数组
        return tokens;
      }
```

`结合上面的状态迁移过程图，更容易理解。`

```js
     //   测试
      const tokens = tokenize(`<div>Hello <span>World</span></div>`);
      console.log("tokens", tokens);
```

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/211f8283d9d64f5995745b87c33ff733~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=EjUEsX99h954mVBQWW7IjJ29vFo%3D)

### 4.2 构建模版AST

根据 Token 列表构建 AST 的过程，其实就是对 Token 列表进行扫描的过程。从第一个 Token 开始，顺序地扫描整个 Token 列表，直到列表中的所有 Token 处理完毕。在这个过程中，我们需要维护一个栈elementStack，这个栈将用于维护元素间的父子关系。每遇到一个开始标签节点，我们就构造一个 Element 类型的 AST 节点，并将其压入栈中。类似地，每当遇到一个结束标签节点，我们就将当前栈顶的节点弹出。这样，栈顶的节点将始终充当父节点的角色。扫描过程中遇到的所有节点，都会作为当前栈顶节点的子节点，并添加到栈顶节点的 children 属性下。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/58b4add4b2094e3e93b3c7f15ff9772b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=VktFC7V03Uv5wZyikZ1E6H%2FNB0o%3D)

<p align=center>图3 Token 列表、父级元素栈和 AST</p>

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/639467850f3b4468adecb8390ae69f3b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=XyvCl8VvykRkD7n876YipmeTj9E%3D)

<p align=center>图4 构建过程中Token 列表、父级元素栈和 AST的状态。</p>
由于模版太简单，所以模版Ast呈现的比较简单。

#### 4.2.1 构建流程简易实现

```js
      // parse 函数接收模板作为参数，返回模版AST
      function parse(str) {
        // 首先对模板进行分割，得到 tokens
        const tokens = tokenize(str);
        // 创建 Root 根节点
        const root = {
          type: "Root",
          children: [],
        }; // 创建 elementStack 栈，起初只有 Root 根节点
        const elementStack = [root];
        while (tokens.length) {
          // 获取当前栈顶节点作为父节点 parent
          const parent = elementStack[elementStack.length - 1];
          // 当前扫描的 Token
          const t = tokens[0];
          switch (t.type) {
            case "tag":
              // 如果当前 Token 是开始标签，则创建 Element 类型的 AST 节点
              const elementNode = {
                type: "Element",
                tag: t.name,
                children: [],
              };
              // 将其添加到父级节点的 children 中
              parent.children.push(elementNode);
              // 将当前节点压入栈
              elementStack.push(elementNode);
              break;
            case "text":
              // 如果当前 Token 是文本，则创建 Text 类型的 AST 节点
              const textNode = {
                type: "Text",
                content: t.content,
              };
              // 将其添加到父节点的 children 中
              parent.children.push(textNode);
              break;
            case "tagEnd":
              // 遇到结束标签，将栈顶节点弹出
              elementStack.pop();
              break;
          }
          // 删除已经扫描过的 token
          tokens.shift();
        }
        // 最后返回 AST
        return root;
      }
```

## 5 转换器(transformer)

节点转换的目的是将 AST 中的每个节点转换为一个合适的 JavaScript 表达式或者语句

*   **元素节点转换（Element）**

    *   Vue 模板中每个标签（如 `<div>`、`<span>`）对应一个 **Element** 节点。
    *   我们通过 `createVNode` 来表示一个 Vue 虚拟节点（VNode），这个函数接收标签名称和其子节点作为参数。
    *   当我们遇到一个元素节点时，需要将它转换为一个 `createVNode` 调用。

*   **文本节点转换（Text）**

    *   文本节点是 Vue 模板中常见的一种节点，包含了纯文本内容（例如 `Hello World`）。
    *   文本节点在 AST 中通常表现为 **Text** 类型。我们将其转换为 JavaScript 字符串字面量，并在生成代码时将其插入到父节点的子节点数组中。

*   **根节点转换（Root）**

    *   根节点是整个模板的起始节点，包含了所有其他子节点。
    *   根节点的转换需要将其子节点（通常是元素节点或文本节点）包装为一个 `render` 函数的返回值。通过这种方式，Vue 的渲染系统能够根据根节点生成的 VNode 来更新页面。

上文中生成的模版AST如下图所示。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/7b3426cf8a7f4d54987caa67b3247f53~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=DmG5Ao%2B%2FVTmk3aGiMVnvouFIKZY%3D)

<p align=center>图5 模版AST</p>
转换后的javaScript AST

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/67569fadec8344c98845d6a88f667d01~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=ZuA6EfyuR%2Fl3lupi6NCNtw5P0JI%3D)

<p align=center>图6 javaScriptAST</p>

### 5.1 简易实现

```js
    // 节点遍历和转换函数，遍历 AST 树并执行转换
function traverseAST(node, context) {
  context.currentNode = node;
  
  const exitCallbacks = []; // 存储退出阶段的回调函数
  const transformFunctions = context.nodeTransforms;

  // 遍历每个转换函数并执行
  for (let i = 0; i < transformFunctions.length; i++) {
    // 执行转换函数，可能返回退出回调
    const exitCallback = transformFunctions[i](context.currentNode, context);
    if (exitCallback) {
      exitCallbacks.push(exitCallback);
    }

    if (!context.currentNode) return; // 如果节点已被删除，退出
  }

  const childNodes = context.currentNode.children; // 获取当前节点的子节点
  if (childNodes) {
    // 递归遍历所有子节点
    for (let i = 0; i < childNodes.length; i++) {
      context.parentNode = context.currentNode;
      context.childIndex = i;
      traverseAST(childNodes[i], context); // 递归处理子节点
    }
  }

  // 在所有子节点处理完后，执行缓存的退出回调函数
  let i = exitCallbacks.length;
  while (i--) {
    exitCallbacks[i](); // 执行回调
  }
}

// 转换文本节点的函数
function transformTextNode(node) {
  if (node.type !== "Text") return; // 如果节点不是文本类型，跳过
  node.jsNode = createStringLiteral(node.content); // 转换文本内容为字符串字面量
}

// 转换元素节点的函数
function transformElementNode(node) {
  return () => {
    if (node.type !== "Element") return; // 如果节点不是元素类型，跳过

    const callExpression = createCallExpression("createVNode", [
      createStringLiteral(node.tag), // 创建元素节点的标签名
    ]);

    if (node.children.length === 1) {
      callExpression.arguments.push(node.children[0].jsNode); // 只有一个子节点时直接使用
    } else {
      callExpression.arguments.push(createArrayExpression(node.children.map((child) => child.jsNode))); // 多个子节点时创建数组
    }

    node.jsNode = callExpression; // 将转换后的 JavaScript AST 存储在 jsNode 属性中
  };
}

// 转换根节点的函数
function transformRootNode(node) {
  return () => {
    if (node.type !== "Root") return; // 如果不是根节点，跳过

    const vnodeJSAST = node.children[0].jsNode; // 获取根节点的第一个子节点的 JavaScript AST
    node.jsNode = {
      type: "FunctionDecl", // 创建一个函数声明
      id: { type: "Identifier", name: "render" }, // 函数名为 render
      params: [], // 无参数
      body: [
        {
          type: "ReturnStatement", // 返回虚拟节点的 AST
          return: vnodeJSAST,
        },
      ],
    };
  };
}

// 转换 AST 的函数
function transformAST(ast) {
  const context = {
    currentNode: null, // 当前处理的节点
    parentNode: null, // 父节点
    replaceNode(newNode) {
      context.currentNode = newNode;
      context.parentNode.children[context.childIndex] = newNode; // 替换父节点的子节点
    },
    removeNode() {
      if (context.parentNode) {
        context.parentNode.children.splice(context.childIndex, 1); // 删除当前节点
        context.currentNode = null; // 清空当前节点
      }
    },
    nodeTransforms: [transformRootNode, transformElementNode, transformTextNode], // 节点转换函数
  };

  traverseAST(ast, context); // 开始遍历 AST
}

```

```js
      const ast = parse(`<div>Hello <span>World</span></div>`);
      transformAST(ast);
```

结果如图6所示。

## 6 生成器(generator)

当所有 AST 节点被转换完成后，我们可以通过 `generate` 函数将生成的 JavaScript AST 转换为实际的代码。

### 6.1 简易实现

```js
      // 节点遍历和转换函数，遍历 AST 树并执行转换
      function traverseAST(node, context) {
        context.currentNode = node;

        const exitCallbacks = []; // 存储退出阶段的回调函数
        const transformFunctions = context.nodeTransforms;

        // 遍历每个转换函数并执行
        for (let i = 0; i < transformFunctions.length; i++) {
          // 执行转换函数，可能返回退出回调
          const exitCallback = transformFunctions[i](
            context.currentNode,
            context
          );
          if (exitCallback) {
            exitCallbacks.push(exitCallback);
          }
          if (!context.currentNode) return; // 如果节点已被删除，退出
        }
        const childNodes = context.currentNode.children; // 获取当前节点的子节点
        if (childNodes) {
          // 递归遍历所有子节点
          for (let i = 0; i < childNodes.length; i++) {
            context.parentNode = context.currentNode;
            context.childIndex = i;
            traverseAST(childNodes[i], context); // 递归处理子节点
          }
        }

        // 在所有子节点处理完后，执行缓存的退出回调函数
        let i = exitCallbacks.length;
        while (i--) {
          exitCallbacks[i](); // 执行回调
        }
      }

      // 转换文本节点的函数
      function transformTextNode(node) {
        if (node.type !== "Text") return; // 如果节点不是文本类型，跳过
        node.jsNode = createStringLiterNode(node.content); // 转换文本内容为字符串字面量
      }

      // 转换元素节点的函数
      function transformElementNode(node) {
        return () => {
          if (node.type !== "Element") return; // 如果节点不是元素类型，跳过

          const callExpre = createCallExpreNode("createVNode", [
            createStringLiterNode(node.tag), // 创建元素节点的标签名
          ]);

          if (node.children.length === 1) {
            callExpre.arguments.push(node.children[0].jsNode); // 只有一个子节点时直接使用
          } else {
            callExpre.arguments.push(
              createArrayExpreNode(
                node.children.map((child) => child.jsNode)
              )
            ); // 多个子节点时创建数组
          }

          node.jsNode = callExpre; // 将转换后的 JavaScript AST 存储在 jsNode 属性中
        };
      }

      // 转换根节点的函数
      function transformRootNode(node) {
        return () => {
          if (node.type !== "Root") return; // 如果不是根节点，跳过

          const vnodeJSAST = node.children[0].jsNode; // 获取根节点的第一个子节点的 JavaScript AST
          node.jsNode = {
            type: "Function", // 创建一个函数声明
            id: { type: "Identifier", name: "render" }, // 函数名为 render
            params: [], // 无参数
            body: [
              {
                type: "ReturnStatement", // 返回虚拟节点的 AST
                return: vnodeJSAST,
              },
            ],
          };
        };
      }

      // 转换 AST 的函数
      function transformAST(ast) {
        const context = {
          currentNode: null, // 当前处理的节点
          parentNode: null, // 父节点
          replaceNode(newNode) {
            context.currentNode = newNode;
            context.parentNode.children[context.childIndex] = newNode; // 替换父节点的子节点
          },
          removeNode() {
            if (context.parentNode) {
              context.parentNode.children.splice(context.childIndex, 1); // 删除当前节点
              context.currentNode = null; // 清空当前节点
            }
          },
          nodeTransforms: [
            transformRootNode,
            transformElementNode,
            transformTextNode,
          ], // 节点转换函数
        };

        traverseAST(ast, context); // 开始遍历 AST
      }
      // 用来创建 StringLiter 节点
      function createStringLiterNode(value) {
        return {
          type: "StringLiter",
          value,
        };
      }

      // 用来创建 Identifier 节点
      function createIdentifierNode(name) {
        return {
          type: "Identifier",
          name,
        };
      }

      // 用来创建 ArrayExpre 节点
      function createArrayExpreNode(elements) {
        return {
          type: "ArrayExpre",
          elements,
        };
      }

      // 用来创建 CallExpre 节点
      function createCallExpreNode(callee, argumentsList) {
        return {
          type: "CallExpre",
          callee: createIdentifierNode(callee),
          arguments: argumentsList,
        };
      }

      // 代码生成器
      function generateCode(node) {
        const context = {
          code: "",
          append(code) {
            context.code += code; // 将代码添加到生成的代码中
          },
          currentIndentLevel: 0, // 当前缩进级别
          // 换行并保持缩进
          addNewline() {
            context.code += "\n" + " ".repeat(context.currentIndentLevel * 2);
          },
          // 增加缩进
          increaseIndent() {
            context.currentIndentLevel++;
            context.addNewline();
          },
          // 减少缩进
          decreaseIndent() {
            context.currentIndentLevel--;
            context.addNewline();
          },
        };

        processNode(node, context); // 处理节点生成代码

        return context.code; // 返回生成的代码
      }

      // 处理不同类型的节点
      function processNode(node, context) {
        switch (node.type) {
          case "Function":
            generateFunction(node, context);
            break;
          case "ReturnStatement":
            generateReturnStatement(node, context);
            break;
          case "CallExpre":
            generateCallExpre(node, context);
            break;
          case "StringLiter":
            generateStringLiter(node, context);
            break;
          case "ArrayExpre":
            generateArrayExpre(node, context);
            break;
        }
      }

      // 生成多个节点的代码
      function generateNodeList(nodes, context) {
        const { append } = context;
        nodes.forEach((node, index) => {
          processNode(node, context); // 递归生成代码
          if (index < nodes.length - 1) {
            append(", "); // 在节点之间加逗号
          }
        });
      }

      // 生成函数声明代码
      function generateFunction(node, context) {
        const { append, increaseIndent, decreaseIndent } = context;
        append(`function ${node.id.name} `);
        append("(");
        generateNodeList(node.params, context); // 生成函数参数代码
        append(") ");
        append("{");
        increaseIndent(); // 增加缩进
        node.body.forEach((n) => processNode(n, context)); // 递归生成函数体代码
        decreaseIndent(); // 减少缩进
        append("}");
      }

      // 生成数组表达式代码
      function generateArrayExpre(node, context) {
        const { append } = context;
        append("[");
        generateNodeList(node.elements, context); // 递归生成数组元素代码
        append("]");
      }

      // 生成返回语句代码
      function generateReturnStatement(node, context) {
        const { append } = context;
        append("return ");
        processNode(node.return, context); // 递归生成返回值代码
      }

      // 生成字符串字面量代码
      function generateStringLiter(node, context) {
        const { append } = context;
        append(`'${node.value}'`); // 直接输出字符串字面量
      }

      // 生成函数调用表达式代码
      function generateCallExpre(node, context) {
        const { append } = context;
        const { callee, arguments: args } = node;
        append(`${callee.name}(`);
        generateNodeList(args, context); // 递归生成函数参数代码
        append(")"); // 完成函数调用的括号
      }
```

结果如下图所示。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/773f7392e2504f90a3e3cdb6efb343e7~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCD6aaZ6I-c5ZCD5YK755qE:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTgwNTg1MzI2NDk4MjU5NSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1739779892&x-orig-sign=MluvAAMP8DDuof3PvbVC4tHDVhU%3D)

## 7 注意

1.  在转换过程中对每个节点进行转换处理时，往往需要根据其子节点的情况来也决定当前节点如何转换，这就使得当前节点的处理要等待其孩子节点都处理完之后在执行，所以节点的处理操作函数transformRootNode，transformTextNode以及transformElementNode等函数真正的处理程序都是放在其返回结果中。
2.  在构建AST中，开始标签要压入栈中，文本类的不要压入栈中，碰到结束标签，要弹出栈顶元素。
3.  剩下的没想到，想到再加

## 7 总结

不想写了。具体的可以去看看Vue设计与实现。
