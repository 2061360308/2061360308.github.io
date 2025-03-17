---
title: "Python装饰器"
description: ""
date: 2025-03-17T11:36:31+08:00
lastmod: 2025-03-17T11:36:31+08:00
draft: false
---
## 一切皆对象

**装饰器**反应了Python作为一种面向对象语言的基本思想：**一切皆对象**

Everything is an object，这是 Python 语言设计的核心特性之一，你可能还不理解这是什么意思，下面我举例子说明一下。

`a = 5`这个简单的语句定义了一个变量a并给他赋值为5。那么这个a变量在python中到底是个什么存在呢。答案是他是一个对象。

你可以运行下面的代码进行检查：

```python
# 检查是否是类的实例
print(isinstance(5, int))    # 输出 True
print(isinstance("abc", str))  # 输出 True

# 查看类的方法和属性
print(dir(int))    # 列出 int 类的所有方法
print(help(str))   # 查看字符串类的文档
```

`a = 5`这个语句的背后则是创建了int类的一个实例对象，他的值为5。这点与其他语言有所不同。例如在 C 或 Java 中，基本类型（如 `int`, `float`）是原始值，不是对象。

一个很好区分他们的方法是在python中你可以在基本类型值上调用各种方法，例如Python 中基本类型（如字符串、整数等）可以直接调用自身的方法，这直观地体现了它们作为对象的特性（点号语法访问属于对象自身的属性或者方法）。而像 C 或 Java 这类语言中，基本类型（primitive types）是原始值而非对象，因此需要通过函数或类方法操作它们。。下面通过对比和示例进一步解释这一区别：

> 完成将字符串全部转大写这样一项工作，Python中的字符串可以直接调用字符串自身的`upper`方法，而在C语言中作为基本值只能通过将这个变量传入某个函数，之后接受函数返回这样的流程来实现。二者有明确的不同，即C中的字符类型本身没有那么复杂的功能，只是代表一个值。C语言中对其操作需要借助于外部的函数，而Python中的字符串类型本身是很复杂的东西，他拥有很多方法可以操作自身，不需要借助外部函数。

```python
# Python中的例子，数值和字符串本身拥有这些功能
x = 10
print(x.real)      # 输出 10（实部属性）
print(x.bit_length())  # 输出 4（二进制位数）

s = "hello"
print(s.upper())  # 输出 "HELLO"（调用 str 类的 upper() 方法）
```

```c
// C语言例子，需要借助于外部提供的函数来操作字符串。

#include <string.h>
#include <stdio.h>

int main() {
    char s[] = "hello";
    // 需要调用函数操作字符串，而不是方法
    printf("%s\n", strupr(s));  // 输出 "HELLO"（函数式操作）
    return 0;
}
```

> **总结**：以上论证了Python中基本类型也是对象的问题，进而可以理解Python中一切皆对象的概念。

那么，一切皆对象，函数当然也不例外。

在Python中则有：

1. 函数也是对象
2. 函数可以被直接输出打印，可以作为返回值

简单来说，既然`def a()`定义的`函数a`与`a = 5`定义的`整数a`二者都是对象。简单类比，你可以在函数中返回整数，当然在函数中返回函数也没有问题

## 函数返回的结果是函数

```python
def hi(name="高**"):
    def greet():
        return "现在执行在 greet() 函数中"
 
    def welcome():
        return "现在执行在 welcome() 函数中"
 
    if name == "高**":
        return greet
    else:
        return welcome
 
a = hi()
print(a)
#outputs: <function greet at 0x7f2143c01500>
 
#上面清晰地展示了`a`现在指向到hi()函数中的greet()函数
#现在试试这个
 
print(a())
#outputs: 现在执行在 greet() 函数中
```

> hi函数被调用后还是一个函数，即你可以调用这个特殊的函数两次`hi()()`,第一次调用
