---
title: Python装饰器
description: ''
date: 2025-03-17T11:36:31+08:00
lastmod: 2025-03-17T11:36:31+08:00
draft: false
'': ''
cover: https://blog-image-1303709080.cos.ap-chengdu.myqcloud.com/c391b25599ed6f13.png
---
## 概念


> 第一次阅读无需了解这些概念，你只需要简单阅读对其中提到的几个关键名词有所印象即可，后续在文章适当的位置我还会重申这些概念



**闭包函数：**声明在一个**函数**中的函数，叫做闭包函数。

**闭包**: 内部函数总是可以访问其所在的外部函数中声明的参数和变量，即使在其外部函数被返回（寿命终结）了之后。


**高阶函数**是至少满足下列一个条件的函数：

* 接受一个或多个函数作为输入
* 输出一个函数


Python 的**装饰器**是高阶函数的语法糖。

Python 的**装饰器**是闭包的一种应用，采用了闭包的思路并将内部闭包函数返回，在不改变原函数代码的前提下为函数增添新的功能


## 一切皆对象

> Python作为一种面向对象语言的基本思想：**一切皆对象**

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

那么，你是否想过我们为什么要在一开始如此篇幅强调**一切皆对象**这个概念呢？

答案是：一切皆对象，函数当然也不例外。我们下面要讨论的哪些概念并不是所有编程语言中均可实现的，他是有前提的，而说这么多只是让为了阐述一个概念，Python中的函数正好符合这些条件。Python的函数也是对象，因此，你除了直接调用它外也可以发挥想象做很多**骚操作**。

即在Python中有：

1. 已知函数也是对象
2. 基本类型是对象
3. 你可以直接输出基本类型的变量，也可以将其作为函数接收的参数，返回值
4. 那么类比可得：函数可以被直接输出打印，可以作为自身的参数与返回值

简单来说，既然`def a()`定义的`函数a`与`a = 5`定义的`整数a`二者都是对象。简单类比，你可以在函数中返回整数，当然在函数中返回函数也没有问题

**在函数中定义函数**，**将函数作为参数传递**，**将函数作为返回值**这些操作听起来就感觉很奇怪，但是依赖于Python中的实现机理，他们都是合法且实际可行的，接下来让我们进入正题，看看这么做到底有什么意义。

## 函数嵌套中定义函数

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

> hi函数被调用后还是一个函数，即你可以调用这个特殊的函数两次`hi()()`,有点意思是吧，但是貌似没有用？

上述这个例子很是简单，但是请你从作用域的角度来思考下面两个问题：

1. `hi`函数外部是否可以直接调用它内部定义的`greet`和`welcome`函数
2. 在`greet`和`welcome`函数中我们能否使用属于`hi`函数内定义的变量，例如传入参数`name`

怎么思考上面两个问题？我们可以类比从全局作用域与其下子函数作用域

![作用域示意图](https://blog-image-1303709080.cos.ap-chengdu.myqcloud.com/c0134b9dab65c23e.png)

让我们从概念出发，函数作用域存在的意义就是隔离变量名进而复用这些变量名，想象一下如果没有作用域的帮助，如果你要新入手维护一个很大的项目，你将不得不面临这样一个困境：你必须通读整个庞大的项目，查看并记录他所有用到的变量名，以避免之后起名的时候重复进而造成不可预知的问题。很绝望是吧，即便你愿意那么做，我们说这种事情也是绝对不可能被允许的，因为那样做太容易犯错了，你的代码质量无法被保证。

故而有了作用域这个东西，在此条件下，上图中的`hello`函数不必关心`hi`函数内部定义变量使用了哪些变量名，它只需要注意全局与它自身下设使用的变量名即可，在某种意义上这便实现了变量复用。

将所有的作用域看成一颗树，兄弟节点与他们的树之间相互是独立的例如`hi`和`hello`独立`hello`也不关心`hi`下的`wecome`和`greet`，同时子节点能够访问所有父节点的作用域（包括非直系）例如`wecome`与`greet`可以访问`变量B`，但这种传递关系权重以最近为原则，即hi中重写了`变量A`（由`A1`到了`A2`）那么`wecome`访问`A变量`就会得到`A2`而不是全局的`A1`。

现在相比已经可以回答一开始的两个问题了

1. `hi`函数外部不能直接调用它内部定义的`greet`和`welcome`函数，因为这些属于函数内部的局部变量。
2. 在`greet`和`welcome`函数中我们能否使用属于`hi`函数内定义的变量，当然可以！就像我们在函数内调用全局变量那样，函数的局部变量对其内部定义的子函数可见，但对外部不可见。

以上就是Python中作用域搜索的核心规则，我们的**骚操作**`函数中定义函数`发生的有趣事情也出现了。在本例子中

`greet`和`welcome`函数成为了一个很“bug”的存在。我们不能在外部访问属于`hi`函数内部作用域的变量，但是`greet`和`welcome`两个函数可以，这一开始可能没什么用，但是如果我们将这两个子函数返回了呢，正如我们示例中做的那样。

一个更短的例子：

```python
def hi(name):
    def greet():
        print(name)
    return greet

a = hi()
```

1. `name`变量属于`hi`函数内部作用域，外部无法访问
2. `greet`函数就在`hi`函数内部，它可以访问`name`变量
3. `hi`函数最后将`greet`函数返回，让外部有可能访问到这个函数
4. 函数外面我们将`a`指向了`hi`函数的返回，即返回的`greet`函数
5. `a`变量是一个全局变量，可以在外部访问
6. 我们可以访问`a`变量进而访问`greet`函数，从而访问到`hi`函数中的`name`变量

我们可以这么形容`greet`函数，原先的`hi`函数的作用域对于外部来说是一个封闭不可访问的，`greet`函数则是将这个作用域的内容“打包”提供了外界，而他本身则是作为包裹的唯一出口，通过这个口子我们才能从里面“掏东西”出来。

> 可以想象一下有一个口子的包裹，类比去理解`greet`函数的作用。如果没有这个函数，相当于包裹没有开口，里面的东西自然也就没办法拿出来了

自然的引出了闭包函数的概念：**闭包函数：**声明在一个**函数**中的函数，叫做闭包函数。

**闭包**：内部函数总是可以访问其所在的外部函数中声明的参数和变量，即使在其外部函数被返回（寿命终结）了之后。

闭包函数可以理解了，那么闭包的概念又如何理解呢，再看一个例子。

```python
def hello(name):
    say = "hello~"+name
    print(say)

hello("小明")
hello("小王")

def hi(name):
    def greet():
        print(name)
    return greet

a_fun = hi("小明")
b_fun = hi("小王")

a_fun()
b_fun()
a_fun()
```

对比`hello`和`hi`两个函数，`hello`就是普通的函数，而`hi`函数内用到了闭包函数`greet`。我们先给name传入值“小明”之后传递值“小王”，得到两个函数a_fun和b_fun，调用a_fun会输出“小明”调用b_fun会输出“小王”，发现了没？只要a_fun和b_fun存在之前我们传入的name就一直会存在，这就是之前所说的——内部函数总是可以访问其所在的外部函数中声明的参数和变量，即使在其外部函数被返回（寿命终结）了之后。

```python
class Hi:
    def __init__(self, name):
        self.name = name
    def greet(self):
        print(self.name)

a_fun = Hi("小明").greet
b_fun = Hi("小王").greet

a_fun()
b_fun()
a_fun()
```

另外一件事情就是，上述的`hi`函数与我们定义的`Hi`类非常相似，都是定义了一个模板，让我们可以将对应的name填充进去。这也是闭包的一个作用，一种比类更加轻量的写法。这里读者只是需要了解即可，事实上闭包的两个作用：“读取函数内部的变量”和“让函数内部的局部变量始终保持在内存中”，都可以被 Python 中现成的对象“类”很好地实现。之所以需要理解闭包这个概念，是为了之后研究“装饰器”这样的概念做铺垫。就好比你学会了加法之后想学习幂运算，在此最好先学一下乘法一样。

最后，让我们总结之前分析得出的有关“闭包”这个东西的特性：

1. **可以读取函数内部的变量**
2. **让函数内部的局部变量始终保持在内存中**
3. **局部变量无法共享和长久的保存，而全局变量可能造成变量污染，闭包既可以长久的保存变量又不会造成全局污染。**

> 由于闭包可以**让函数内部的局部变量始终保持在内存中**，所以它会**增加内存消耗**，所以不能滥用闭包，否则会造成程序的**性能问题**，可能导致**内存泄露**。

一句话总结，**“爸爸的爸爸叫爷爷，函数的函数叫？👀️”**

## 函数中接收函数参数

理解了“**一切皆对象**”后，函数中接收的参数可以是函数当然也很容易接受了。我们直接看一个例子。

```python
names=["Alex","amanda","xiaowu"]

def fitter_fun(name):
    if name == 'Alex':
        return true
    else:
        return false

print(list(filter(fitter_fun,names)))
```

> Python中的 filter 函数是接收一个函数和一个序列的高阶函数，其主要功能是 `过滤`。

上述代码中使用的python内置函数filter就是一个能够接收函数的函数，代码的意思也不难理解，对names列表进行过滤，只保留`Alex`。

而像这样，至少接受一个函数变量作为参数的函数被称作**高阶函数**，高阶函数的意义就是我们能够将函数功能的某一部分交给后续调用者自己灵活定义，就像这里的filter函数将是筛选的标准交由用户自己决定。这样做能够极大程度上提高函数的灵活程度。

能够返回一个函数的函数也叫高阶函数，满足高阶函数的要求是接收至少一个函数，或者输出一个函数这两个条件至少任意满足其一即可。

这里的返回一个函数并不一定是函数内部定义的“闭包函数“，可以是外部的函数，例如：

```python
def a():
    pass
def b():
    return a
```

这里的b也算是高阶函数。

**维基百科**这样定义：

**高阶函数**是至少满足下列一个条件的函数：

* 接受一个或多个函数作为输入
* 输出一个函数

## 装饰器

```python
def eat_food():
    print("吃饭")

def bathroom():
    print("上厕所")

def study():
    print("学习")

def take_medicine():
    print("吃药")

....
```

设想这样一个场景，你是一个非常爱干净的人，干什么之前都要洗手。但是呢你也会不定时抽风，有的时候又会不想洗。按照这个需求我们为上述各种活动加上洗手的步骤，由于你会抽风，我们不能直接修改每个函数，但是单独为每个函数写一个“洗手版”又太过麻烦了，为此，我们可以设计下面这样一个wash_hands函数


![20180129192412_t2Vfz.gif](https://blog-image-1303709080.cos.ap-chengdu.myqcloud.com/b245d636766a4c67.gif)

```python
def wash_hands(func):
    def wash():
        print("正在洗手")
        // 干其他事情
        func()
    return wash
```

这个函数不难理解，传入函数，然后返回watch函数，而在watch函数中都会先洗手，之后干什么取决于你传入的内容。而我要告诉你的，这正是 python 中装饰器做的事情！它们封装一个函数，并且用这样或者那样的方式来修改它的行为。

根据前面讲述的内容分析一下，这个函数既是一个高阶函数（接收一个函数作为参数）又使用了闭包方法，其中wash就是一个闭包函数。它同时满足高阶函数和闭包的定义，这种东西就是一个装饰器。

这是一个很贴切的一个名字：一个能够批量在一定程度上修改函数功能却又不会影响函数本身的工具。

**装饰**的含义我认为一部分是由于它无法更改函数的核心功能，而是只能在函数执行前以及执行后进行操作，因而叫“装饰器”而不是“修改器”。而另一方面则是装饰器不影响函数本身，只是把函数放进去“装饰”一下，在需要的时候我们还能够使用原来的函数，这种需要的时候可以用，不需要的时候移除也很方便，这不正是装饰品的特性吗？

回到例子本身，让我们提取一下一个装饰器函数应该满足的条件：

1. 接收一个函数作为参数，作为需要被装饰的函数
2. 拥有一个闭包函数，这个闭包函数需要调用被装饰的函数，并在调用前和调用后执行自己的特殊行为
3. 返回值是内部的闭包函数

现在你也许疑惑，我们在代码里并没有使用 **@** 符号？那只是一个简短的方式来生成一个被装饰的函数。准确来说他被我们称作"**语法糖**"，一种额外设计用来帮你简化工作的语法，用了它能够让你写代码变的很爽。下面我们就用@符号的方式使用一下我们之前使用的wash_hands装饰器

```python
@wash_hands
def eat_food():
    print("吃饭")

@wash_hands
def bathroom():
    print("上厕所")

@wash_hands
def study():
    print("学习")

@wash_hands
def take_medicine():
    print("吃药")
```

是不是比你手动调用wash_hands函数传值再接收方便多了，爽不爽？这就是**语法糖**，Python之所以让我们这么喜欢很大程度上离不开其贴心的**语法糖**设计。而且在使用@方法调用装饰器后，装饰之名更是实至名归了，像一顶优雅的帽子，轻轻戴上就能提供额外的功能，不需要了也可以简单的摘下，灵巧又方便这就是装饰器啦。

什么是装饰器，看了这么多你明白了吗？

1. Python 的**装饰器**是高阶函数的语法糖。
2. Python 的**装饰器**是闭包的一种应用，采用了闭包的思路并将内部闭包函数返回，在不改变原函数代码的前提下为函数增添新的功能

## 带参数的装饰器

装饰器使用起来非常方便，相信很多人不理解它原理的情况下也能正常使用别人提供的各种装饰器。本篇文章却是要教会你怎么自己写**装饰器**。看到这儿的人像写一个上述`wash_hands`那样的装饰器可以说是手到擒来了，这小节让我们拓展一下，改造`wash_hands`装饰器让其可以接收参数。

**情景如下**，熬夜学习编程的你病情加重，现在不仅会不定时抽风不想洗手，而且有时候抽风干某事前会洗很多次手。



![14644018657215620.gif](https://blog-image-1303709080.cos.ap-chengdu.myqcloud.com/0df2c400b975e155.gif)


怎么设计呢？让我们从原来的函数开始分析

```python
def wash_hands(func):
    def wash():
        print("正在洗手")
        // 干其他事情
        func()
    return wash
```

现在这个函数已经能够实现事前洗手了，我们要做的是添加一个次数变量来控制洗手的次数。

```python
def wash_hands(func, frequency=1):
    def wash():
        for i in range(frequency):
            print("正在洗手")
        // 干其他事情
        func()
    return wash
```

很好，现在我们给这个函数加上了一个`frequency`参数用于控制洗手次数。但是这次改造也造成了一个问题，我们只能手动调用传参，而无法使用语法糖使用这个装饰器了

```python
@wash_hands
def eat_food():
    print("吃饭")

@wash_hands(2)
def eat_food():
    print("吃饭")

```

就像这样@语法会默认将eat_food传递给wash_hands但是我们没办法添加其他参数，像第二种错误的示例则是将2赋值给了func参数，之后整个watch函数就会被直接返回，没有办法再传入eat_food这个函数了。

不过@wash_hands(2)这种写法给我们一些启发，我们的想法是只要这个东西返回一个装饰器就好了，还记得之前讨论闭包的时候说的一个例子吗？

```python
def hi(name):
    def greet():
        print(name)
    return greet

a_fun = hi("小明")
b_fun = hi("小王")
```

这个例子里我们利用闭包批量创建引用不同name值的greet函数，只要这里的greet函数替换成我们的原始wash_hands不就行了吗？调用一次hi并传入frequency就能生成一个携带不同frequency值的装饰器函数。

```python
def hi(frequency):
    def wash_hands(func):
        def wash():
            for i in range(frequency):
                print("正在洗手")
            // 干其他事情
            func()
        return wash
    return greet

// frequency = 1
new_func_1 = hi(1)
// frequency = 2
new_func_2 = hi()

@new_func_1
def eat_food():
    print("吃饭")

@new_func_1
def bathroom():
    print("上厕所")


```

new_func_1和new_func_2均是被返回的装饰器，区别是他们绑定的frequency值不同，而且我们能够通过类似的方式生成任意的new_func_n来洗n次手，接下里的不用解释了吧，使用这个装饰器就实现了吃饭前洗手一次，上厕所前洗手两次的惊人成就~~~

> 这里的绑定就是指利用闭包特性，在函数调用结束之后将局部变量依旧保留在内存中，供后续内部函数继续使用

作为一个严谨的程序员，让我们整理一下上面的代码，给出一个较为完整的示例：

```python
def wash_hands(frequency):
    def wraps(func):
        def wash():
            for i in range(frequency):
                print("正在洗手")
            // 干其他事情
            func()
        return wash
    return wraps

@wash_hands(1)
def eat_food():
    print("吃饭")

@wash_hands(2)
def bathroom():
    print("上厕所")

@wash_hands(10)
def study():
    print("学习")

@wash_hands(5)
def take_medicine():
    print("吃药")
```

## 学以致用

一切皆对象，闭包，高阶函数，到最后我们讲完了装饰器，其中我们列举了很多例子，为了能够便于理解这些例子都以简短易懂为主，下面我列举一些实际应用中的例子。

### 授权(Authorization)

装饰器能有助于检查某个人是否被授权去使用一个 web 应用的端点(endpoint)。它们被大量使用于 Flask 和 Django web 框架中。这里是一个例子来使用基于装饰器的授权：

```python
from functools import wraps
 
def requires_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        auth = request.authorization
        if not auth or not check_auth(auth.username, auth.password):
            authenticate()
        return f(*args, **kwargs)
    return decorated
```

### 日志(Logging)

日志是装饰器运用的另一个亮点。这是个例子：

```python
from functools import wraps
 
def logit(func):
    @wraps(func)
    def with_logging(*args, **kwargs):
        print(func.__name__ + " was called")
        return func(*args, **kwargs)
    return with_logging
 
@logit
def addition_func(x):
   """Do some math."""
   return x + x
 
 
result = addition_func(4)
# Output: addition_func was called
```

## 参考


[Python 函数装饰器 | 菜鸟教程](https://www.runoob.com/w3cnote/python-func-decorators.html)

[Python/装饰器 - 维基教科书，自由的教学读本](https://zh.wikibooks.org/wiki/Python/%E8%A3%85%E9%A5%B0%E5%99%A8)

[高阶函数 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)

[Python 闭包（Closure）详解 - 知乎](https://zhuanlan.zhihu.com/p/453787908)

[闭包，看这一篇就够了——带你看透闭包的本质，百发百中-CSDN 博客](https://blog.csdn.net/weixin_43586120/article/details/89456183)
