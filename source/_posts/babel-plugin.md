---
title: Babel 插件开发实践
date: 2019-01-03 15:18:47
tags:
  babel
---

## 背景

在上一篇介绍 webpack 升级 webpack 4 版本的时候，在最后提到几个在实际项目中遇到的问题里，有一个是在配合 webpack 升级的过程中，vue-loader 需要对应升级到 15.x，但是这个升级导致原有的用 commonjs 写法去 `require` vue 组件时出错了，原因是在 vue-loader 的 14 版本后 vue 文件导出的模块一定是 esModule。详见这个 [Issue](https://github.com/vuejs/vue-loader/issues/1172)。

Issue 中尤大大提到的解决方案是可以写一个 Babel 插件去解决这个问题。Babel 大家应该都很熟悉，我们写的 ES6 和 JSX 代码都是靠 Babel 转成浏览器兼容的代码。那 Babel 插件呢，下面开始介绍一下 Babel 插件。

<!-- more -->

## Babel 运行过程

介绍 Babel 插件前，我们先来看看 Babel 转码的三个处理步骤。首先先介绍一下抽象语法树

### 抽象语法树（AST）

在 Babel 的处理过程中的每一步都涉及到创建或是操作抽象语法树，亦称 AST。

``` js
function square(n) {
  return n * n;
}
```

上面的代码可以被表示成下面的树形结构

```
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

其中的每一层类似 `{ type: 'FunctionDeclaration', ... }` 的结构都被称为节点（Node），一个 AST 可以由单一的节点或是成百上千个节点构成。 它们组合在一起可以描述用于静态分析的程序语法。

字符串形式的 type 字段表示节点的类型，我们后续的插件就是通过 type 来判断节点进行不同的处理

### Babel 的处理步骤

Babel 转码过程分成三个阶段：分析(parse)、转换(transform)、生成(generate)

其中，分析、生成阶段由 Babel 核心完成，而转换阶段，则由 Babel 插件完成，这也是我们下面介绍的重点

**分析**

Babel读入源代码，经过词法分析、语法分析后，生成抽象语法树（AST）。

**转换**

经过前一阶段的代码分析，Babel 得到了 AST。在原始 AST 的基础上，Babel 通过插件，对其进行修改，比如新增、删除、修改后，得到新的 AST。

**生成**

通过前一阶段的转换，Babel 得到了新的 AST，然后就可以逆向操作，生成新的代码。

代码生成其实很简单：深度优先遍历整个 AST，然后构建可以表示转换后代码的字符串。

## Babel 插件

Babel 插件的主要工作就是在转换的步骤中对 AST 中的节点进行新增、删除和修改操作。

**Visitor（访问者）**

Babel 在递归遍历 AST 语法树时，会访问节点，之所以用访问这个词，是因为有访问者模式这个概念。

访问者(Visitor)是用于 AST 遍历的跨语言模式。简单说他是一个对象，定义了用于在树状结构中获取具体节点的方法。看一下下面的例子

```js
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```

这是一个简单的访问者，把它用于遍历中时，每当在树中遇见一个 Identifier 的时候会调用 Identifier() 方法。

如果我们需要在遇到调用表达式（CallExpression）的时候做一些处理，就可以定义一个 `CallExpression` 回调方法做相应处理

**Path（路径）**

AST 中有很多节点，每个节点可能有不同的属性，并且节点之间可能存在关联。path 是个对象，它代表了两个节点之间的关联。你可以在 path 上访问到节点的属性，也可以通过 path 来访问到关联的节点（比如父节点、兄弟节点等）

例如，如果有下面这样一个节点及其子节点︰

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

将子节点 Identifier 表示为一个路径（Path）的话，看起来是这样的：

```js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```

路径对象中还会包含添加、更新、移动和删除节点等有关的其他方法

例如我们想替换路径中的节点，可以使用 `replaceWith` 方法，还有很多其他方法，可以通过 Babel 官方文档查看。

### 编写插件

前面介绍的是下面开发我们的插件必备的只是，还有其他未提及的插件相关的知识可以看[开发手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)。开发插件主要是构建 Visitor，有下面两步

 - 确定访问条件
 - 确定转换逻辑

但是在构建 Visitor 之前，我们要先分析源文件和目标文件的抽象语法树。通过 [AST explorer](http://astexplorer.net/#/Z1exs6BWMq)清晰地看到我们的语法树

**回到背景**

说回我们编写插件的背景。在 vue-loader 版本升级后，默认的 vue 单文件导出默认变成了 esModule 模块导出。这就导致了之前我们通过 require 方式引入的 vue 组件

`const component = require("./component.vue")`

必须变成下面的方式引用

`const component = require("./component.vue").default`

当然我们可以修改所有源码加上 `.default` 属性调用，但作为一个有追求的程序员，这样显得不够优雅

通过我们前面了解的 Babel 插件的知识，我们可以优雅的处理这个问题

**插件思路**

首先分析一下转换前的语法树

```
{
  "type": "CallExpression",
  "start": 18,
  "end": 44,
  "loc": ...
  },
  "callee": {
    "type": "Identifier",
    "start": 18,
    "end": 25,
    "loc": {
      "start": {
        "line": 1,
        "column": 18
      },
      "end": {
        "line": 1,
        "column": 25
      },
      "identifierName": "require"
    },
    "name": "require"
  },
  "arguments": [
    {
      "type": "StringLiteral",
      "start": 26,
      "end": 43,
      "loc": {
        "start": {
          "line": 1,
          "column": 26
        },
        "end": {
          "line": 1,
          "column": 43
        }
      },
      "extra": {
        "rawValue": "./component.vue",
        "raw": "\"./component.vue\""
      },
      "value": "./component.vue"
    }
  ]
}
```

可以看到 require 调用被转换成 一个 CallExpression，我们需要将 CallExpression 装换成另外一个语句

下面再看看目标代码的语法树

```
{
  "type": "MemberExpression",
  "start": 18,
  "end": 52,
  "loc": ...,
  "object": {
    "type": "CallExpression",
    "start": 18,
    "end": 44,
    "loc": ...,
    "callee": {
      "type": "Identifier",
      "start": 18,
      "end": 25,
      "loc": ...,
        "identifierName": "require"
      },
      "name": "require"
    },
    "arguments": [
      {
        "type": "StringLiteral",
        "start": 26,
        "end": 43,
        "loc":...,
        "extra": {
          "rawValue": "./component.vue",
          "raw": "\"./component.vue\""
        },
        "value": "./component.vue"
      }
    ]
  },
  "property": {
    "type": "Identifier",
    "start": 45,
    "end": 52,
    "loc": ...,
      "identifierName": "default"
    },
    "name": "default"
  },
  "computed": false
}
```

可以看到我们的 CallExpression 被一个叫 MemberExpression 取代了，`property` 属性是一个名为 `default` 的 Identifier。同时 MemberExpression 的 `object` 对应的正是前面的 MemberExpression，内容基本一样。

按照上面的分析，我们可以在 visitor 的访问过程中，处理 CallExpression 的回调方法，判断调用的的方法名称是 `require`，且参数是以 `.vue` 结尾的字符串，我们就可以用一个 MemberExpression 来替换这个表达式。具体我们可以用到 path 中的 `replaceWith` 方法。

完整的代码如下

```js
module.exports = function({ types: t }) {

  const isRequireCall = (path) => {
    return path.node.callee.name === 'require'
  }

  const isRequireVue = (path) => {
    let firstArg = path.node.arguments[0]
    if (t.isStringLiteral(firstArg) && firstArg.value.endsWith('.vue')) {
      return true
    }
    return false
  }

  const isParentMemberExpression = (path) => {
    return path.parent.type === 'MemberExpression'
  }

  return {
    // name: "add-vue-module-exports",
    visitor: {
      CallExpression(path) {
        // 判断为 require vue 文件 且 require 没有调用其他属性
        if ( !isRequireCall(path) || !isRequireVue(path) || isParentMemberExpression(path) ) {
          return;
        }
        path.replaceWith(
          t.MemberExpression(
            path.node,
            t.identifier('default')
          )
        )
      }
    }
  };
}

```

这个插件在是尤大提供后去了解 Babel 的插件机制后试着写出来的，也体会到了 AST 的强大之处。AST 可以做到很多事情，也让我想起来前段时间有个开发者不满微信小程序不能使用 eval 动态执行脚本，自己写了一个运行在小程序中的 JavaScript 的解释器，其中也离不开 AST。上面记录了编写插件的思路过程，希望对有需要的同学有所帮助。

参考

 - [Babel 插件手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)