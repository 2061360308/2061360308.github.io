---
date: 2025-08-29T19:49:46+08:00
title: Codemirror6包 用法总结
lastmod: 2025-04-02T00:56:40+08:00
categories:
  - 日常总结
tags:
  - Node
  - JavaScript
  - Codemirror6
---

## Codemirror6 包 用法总结

> 本次用到的 Codemirror6 有关的包如下

|            语法支持包             |           语法支持包            |            核心插件             |
| :-------------------------------: | :-----------------------------: | :-----------------------------: |
|    @codemirror/lang-yaml 6.1.1    |   @codemirror/lang-less 6.0.2   | @codemirror/autocomplete 6.16.0 |
|    @codemirror/lang-xml 6.1.0     |  @codemirror/lang-liquid 6.2.1  |   @codemirror/commands 6.5.0    |
|  @codemirror/lang-angular 0.1.3   | @codemirror/lang-markdown 6.2.5 |   @codemirror/language 6.10.1   |
|    @codemirror/lang-cpp 6.0.2     |   @codemirror/lang-php 6.0.1    | @codemirror/language-data 6.5.1 |
|    @codemirror/lang-css 6.2.1     |  @codemirror/lang-python 6.1.6  | @codemirror/legacy-modes 6.4.0  |
|     @codemirror/lang-go 6.0.0     |   @codemirror/lang-rust 6.0.1   |     @codemirror/lint 6.7.0      |
|    @codemirror/lang-html 6.4.9    |   @codemirror/lang-sass 6.0.2   |    @codemirror/search 6.5.6     |
|    @codemirror/lang-java 6.0.1    |   @codemirror/lang-sql 6.6.4    |     @codemirror/state 6.4.1     |
| @codemirror/lang-javascript 6.2.2 |   @codemirror/lang-vue 0.1.3    |     @codemirror/view 6.26.3     |
|    @codemirror/lang-json 6.0.1    |   @codemirror/lang-wast 6.0.2   |                                 |

> 上述包用到的导出内容如下

```js
import { EditorState } from "@codemirror/state";
import { EditorView } from "@codemirror/view";
import {
  lineNumbers,
  highlightActiveLineGutter,
  highlightSpecialChars,
  drawSelection,
  dropCursor,
  rectangularSelection,
  crosshairCursor,
  highlightActiveLine,
  keymap,
} from "@codemirror/view";
import {
  foldGutter,
  indentOnInput,
  syntaxHighlighting,
  bracketMatching,
  foldKeymap,
} from "@codemirror/language";
import { languages } from "@codemirror/language-data";
import { history, defaultKeymap, historyKeymap } from "@codemirror/commands";
import { highlightSelectionMatches, searchKeymap } from "@codemirror/search";
import {
  closeBrackets,
  autocompletion,
  closeBracketsKeymap,
  completionKeymap,
} from "@codemirror/autocomplete";
import { lintKeymap } from "@codemirror/lint";
```

> 此外，还有一个`codemirror`包，从中可以导出基础模式的配置和迷你模式的配置
> 例如：import { basicSetup } from "codemirror";

> 使用方法

```js
const editorTheme = EditorView.theme(
  {
    // 输入的字体颜色
    "&": {
      color: theme.workspaceTextColor,
      backgroundColor: theme.workspaceBgColor,
    },
    // 背景色
    ".cm-content": {
      caretColor: theme.workspaceTextColor,
    },
    //光标的颜色
    "&.cm-focused .cm-cursor": {
      borderLeftColor: theme.cursorColor,
    },
    // 选中的状态
    "&.cm-focused .cm-selectionBackground, ::selection": {
      // color: theme.selectionBgColor, // Todo: 选中文字背景色无效
      backgroundColor: `${theme.selectionBgColor} !important`,
    },
    // 激活序列的背景色
    ".cm-gutterElement.cm-activeLineGutter": {
      backgroundColor: theme.currentLineColor,
    },
    // 激活背景色
    ".cm-activeLine.cm-line": {
      backgroundColor: `${theme.currentLineColor} !important`,
    },
    // 左侧侧边栏的颜色
    ".cm-gutters": {
      backgroundColor: theme.workspaceBgColor,
      color: theme.gutterLineNumColor,
      border: `1px solid ${theme.workspaceBgColorLight}`,
    },
  },
  { name: "my-theme" }
);

let doc = docStore.docs[id];
console.log(doc, doc.stateJson);
const state = EditorState.create({
  doc: doc.stateJson.doc.content,
  extensions: [
    editorTheme,
    lineNumbers(),
    highlightActiveLineGutter(),
    highlightSpecialChars(),
    history(),
    foldGutter(),
    drawSelection(),
    dropCursor(),
    EditorState.allowMultipleSelections.of(true),
    indentOnInput(),
    syntaxHighlighting(markdwonHighlightStyle, { fallback: false }),
    bracketMatching(),
    closeBrackets(),
    autocompletion(),
    rectangularSelection(),
    crosshairCursor(),
    // highlightActiveLine(),
    highlightSelectionMatches(),
    keymap.of([
      ...closeBracketsKeymap,
      ...defaultKeymap,
      ...searchKeymap,
      ...historyKeymap,
      ...foldKeymap,
      ...completionKeymap,
      ...lintKeymap,
    ]),
    markdown({ codeLanguages: languages }),
  ],
});
```

`editorTheme`自定义了编辑器的主题

`doc: doc.stateJson.doc.content`是创建 state 实例的初始文本

`markdown({ codeLanguages: languages })`中的`languages`是来自`@codemirror/language-data`包的导出，这个并不是固定的，language-data 是定义了一些语言包的动态导入，可以根据需求仿照他的源码进行自定义编写

`markdwonHighlightStyle`是自定义的语法高亮标记，定义实现如下

> 默认的定义样式可见:`import { defaultHighlightStyle } from "@codemirror/language";`

```js
import { HighlightStyle } from "@codemirror/language";
import { tags } from "@lezer/highlight";

const markdwonHighlightStyle = HighlightStyle.define([
  {
    tag: tags.heading1,
    fontSize: "26px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  {
    tag: tags.heading2,
    fontSize: "24px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  {
    tag: tags.heading3,
    fontSize: "22px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  {
    tag: tags.heading4,
    fontSize: "20px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  {
    tag: tags.heading5,
    fontSize: "18px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  {
    tag: tags.heading6,
    fontSize: "16px",
    fontWeight: "bold",
    color: "#9876aa",
  },
  { tag: [tags.strong], fontWeight: "bold" },
  { tag: tags.emphasis, fontStyle: "italic" },
  { tag: tags.strikethrough, textDecoration: "line-through" },
  { tag: tags.meta, color: "#c57330" },
  { tag: tags.link, color: "#287bde" },
  { tag: tags.quote, color: "#6a8759" },
  { tag: tags.number, color: "#2e5795" },
  { tag: tags.keyword, color: "#c57330" },
  {
    tag: [
      tags.atom,
      tags.bool,
      tags.url,
      tags.contentSeparator,
      tags.labelName,
    ],
    color: "#8e75a7",
  },
  { tag: [tags.literal, tags.inserted], color: "#a5c261" },
  { tag: [tags.string, tags.deleted], color: "#68844e" },
  {
    tag: [tags.regexp, tags.escape, tags.special(tags.string)],
    color: "#e40",
  },
  { tag: tags.definition(tags.variableName), color: "#8ea4b8" },
  { tag: tags.local(tags.variableName), color: "#30a" },
  { tag: [tags.typeName, tags.namespace], color: "#e7b350" },
  { tag: tags.className, color: "#317467" },
  {
    tag: [tags.special(tags.variableName), tags.macroName],
    color: "#256",
  },
  { tag: tags.definition(tags.propertyName), color: "#00c" },
  { tag: tags.comment, color: "#5f9654" },
  { tag: tags.invalid, color: "#f00" },
  { tag: tags.self, color: "#c57330" },
]);
```

> 简单翻译了给出的tags的值,如下

```js
/**
 * 可能需要的tags值
 * 英文原版和更多详细信息请查看官方文档 https://lezer.codemirror.net/docs/ref/#highlight.Tag
 * 以下内容通过机器翻译，可能有误
comment: 注释
lineComment: 行注释
blockComment: 块注释
docComment: 文档注释
name: 任何类型的标识符
variableName: 变量名
typeName: 类型名
tagName: 标记名（typeName的子标记）
propertyName: 属性或字段名称
attributeName: 属性名称（propertyName的子标记）
className: 类名
labelName: 标签名称
namespace: 命名空间名称
macroName: 宏的名称
literal: 文字值
string: 字符串文字
docString: 文档字符串
character: 字符文字（string的子标记）
attributeValue: 属性值（string的子标记）。
number: 数字
integer: 整数
float: 浮点数
bool: 布尔
regexp: 正则表达式
escape: 转义文字，例如字符串中的反斜杠转义
color: 颜色
url: 链接
keyword: 语言中的关键字
self: self或本对象的关键字(this)
null: null值
atom: 表示某种原子值的关键字
unit: 表示单位的关键字
modifier: 修饰符关键字
operatorKeyword: 充当运算符的关键字
controlKeyword: 与控制流相关的关键字
definitionKeyword: 定义某个事物的关键字
moduleKeyword: 与定义模块或与模块接口相关的关键字
operator: 运算符
derefOperator: 取消引用某物的运算符
arithmeticOperator: 算术运算符
logicOperator: 逻辑运算符
bitwiseOperator: 位运算符
compareOperator: 比较运算符
updateOperator: 更新操作数的运算符
definitionOperator: 定义某物的运算符
typeOperator: 与类型相关的运算符
controlOperator: 控制流运算符
punctuation: 程序或标记语言的标点符号
separator: 分隔符
bracket: 括号式标点符号
angleBracket: 尖括号（通常是 < 和 > ）
squareBracket: 方括号（通常是 [ 和 ] ）
paren: 圆括号（通常是 ( 和 ) ）。是bracket的子标签
brace: 大括号（通常是 { 和 } ）。是bracket的子标签
content: 内容，例如XML或标记文档中的纯文本
heading: 表示标题的内容
heading1: 一级标题
heading2: 二级标题
heading3: 三级标题
heading4: 四级标题
heading5: 五级标题
heading6: 六级标题
contentSeparator: 散文分隔符（如水平线）
list: 表示列表的内容
quote: 表示引用的内容
emphasis:  强调（斜体）
strong: 强调（加粗）
link: 链接的内容
monospace: 被设置为代码或等宽字体的内容
strikethrough: 有删除线样式的内容
inserted: 在变更跟踪格式中插入的文本
deleted: 删除的文本
changed: 更改的文本
invalid: 无效或不符合语法的元素
meta: 元数据或元指令
documentMeta: 应用于整个文档的元数据
annotation: 注解或添加属性到给定语法元素的元数据
processingInstruction: 处理指令或预处理器指令。是meta的子标签
definition(tag: Tag) → 表示给定元素正在被定义的修饰符。预期与各种名称标签一起使用
constant(tag: Tag) → 表示某物是常量的修饰符。主要预期与变量名一起使用。
function(tag: Tag) → 用于表示变量或属性名被调用或定义为函数的修饰符。
standard(tag: Tag) → 可以应用于名称的修饰符，表示它们属于语言的标准环境。
local(tag: Tag) → 表示给定名称在某个范围内是局部的修饰符。
special(tag: Tag) → 一个通用的变体修饰符，可以用来标记某些常见标签的语言特定的替代变体。建议主题至少为字符串和变量名标签定义特殊形式，因为这些经常出现。
 */
```

## 使用过程中遇到的问题

1. 使用`basicSetup`时，发现当前行选中时背景颜色无法突显。找到原因是其中`highlightActiveLine`扩展的当前行背景色与自定义背景色冲突，最后自己复制了`basicSetup`的代码删去`highlightActiveLine`扩展的使用成功解决。
2. 在`Codemirror`联合`vue`使用的情况下，使用`pinia`仓库集中管理所有的`EditorState`实例（编辑器多标签），发现语法高亮会自动丢失，原因可能是`EditorState`实例对象太过于复杂，首先尝试了可以用`markRaw`标记`EditorState`实例让`vue`保持原始数据且不对数据进行监测。最后发现了更好的办法，state提供了`toJSON`和`fromJson`方法，可以通过使用`pinia`管理这个json对象,在需要的时候利用json对象重建`EditorState`实例对象传递给`EditorView`实例显示。

## 总结

`Codemirror6`的文档还是很齐全的，但是可能由于篇幅的原因，很多方法只是给出了文字说明，外加函数原型。但是新手在不了解的情况下，看到函数原型里传入数据的各种内部类型标注还是很难理解的，解决方案可以从他的源码看起，编辑器提供了许多默认的功能，先找到编辑器默认的定义，之后对照文档的说明，就可以照猫画虎自定义想要的功能了。
