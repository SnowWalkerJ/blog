---
title: Python TypeHint
layout: post
date: 2020-05-07 23:16:15
---

## 什么是type hint

众所周知，Python是一门动态类型的语言，每个变量的类型都是可以变化的，这种特性给我们提供了足够的灵活性，让我们免于处理各种prototyping，而能把注意力
集中在具体的功能上。然而动态类型也有它的问题，除了不能依赖类型信息做优化以外，可读性也会显著下降。为了解决可读性的问题，python3.5之后引入了 type hint，
用来给变量标记“类型提示”。需要注意的是，虽然可以给变量标记类型，但这个类型毕竟只是“提示”，python解释器并不会帮你检查这个变量究竟是不是符合标记的类型。
Python依旧是动态类型的。

<!-- more -->

以下是type hint的一些使用方法：

```python
x: int = 1

class MyClass:
    value: float = 0.0

def strlen(s: str) -> int:
    n: int = 0
    n += len(s)
    return n
```

## 为什么要用type hint

### 类型即注释

很多有java、c++经验的同学转学Python的时候都会很不适应。在静态类型的语言中，变量的类型承担了一部分“注释”的作用。我们观察一个函数的参数类型，往往就能知道这个函数该怎么用，有什么效果。但是Python缺乏类型信息，仅仅看代码往往很难明白一个函数到底在干吗。举个例子，

```python
def transform(data):
    ...   # 若干代码
```

是不是云里雾里，这个data参数究竟应该是啥呢？我们用type hint给个小小的修饰：

```python
def transform(data: List[float]) -> float:
    ...   # 若干代码
```

就清晰多了：我们知道这个函数是用来把一组数据聚合成一个数的。

### 帮助IDE重构、提示

使用Pycharm或者VSCode等IDE时，如果给你的代码标记上type hint，IDE会根据变量的类型更好地给你提供与该类型有关的方法、属性的提示

### 静态类型检查

虽然Python解释器不会在运行时帮你检查变量类型，但是有很多静态类型检查器（如mypy）可以帮助你检查类型是否正确，如

```python
number: int = 0

def f(x: str):
    ...

f(number)
```

上面这段代码会被静态类型检查器报错，因为把一个int类型的变量传给了f函数，而f函数接收的是一个str类型的参数。

### 其他应用

卖个关子，请看下文。

## typing模块

仅使用int、float、str这些类型，能表达的类型种类似乎有点有限。如果我想表达一个list类型怎么办？更复杂一点的，一个int的列表，一个由str映射到float的字典？

typing模块为我们提供了很多类型来解决这些需求。

### 容器

容器很像C++中的模板，它们接收若干个类型参数，来生成一个新的类型。如`List[float]`表达的是一个元素为float类型的列表，而`Dict[str, int]`则表达一个键为str类型，
值为int类型的字典。其他容器包括`Set`、`Tuple`等。

### 多类型

有时候，我们可能希望一个变量是几种类型之一的，常见的情况是，一个函数既能接收float类型的参数，也能接收int类型的参数，typing提供了`Union`来实现这个效果。
`Union[int, float]`即可表达int或float类型的变量。还有的时候，一个函数的参数是可以为None的，就可以用`Optional[str]`来表达一个str或None的类型。

### Prototype

在其他语言里，一个函数的参数不仅可以限定为某一个具体的类型，有时也可以限定为某一种接口。typing模块也提供了这个方案，称为Prototype。一个类型只需要实现了一个Prototype中定义的方法，就会自动称为这个Prototype的子类，而不需要显式地继承这个Prototype。这允许我们不用侵入式的修改已有的代码就能实现接口。内置的Prototype包括：

- `Sequence[T]`
- `Mapping[KT, VT]`
- `Sized`
- `Iterable[T]`
- `Callable[[Input], Ret]`

## 不要滥用type hint

我刚了解到typehint的时候觉得“哇，这东西很酷！”然后就一股脑把所有的代码都加上了typehint，结果就是代码乱了很多。
我想强调的是，typehint是个工具，要善用它，不要滥用它。

Python的本质还是动态类型的语言，一股脑加上typehint只会让你的代码变得特别的冗长，你可能还会不自觉放弃它作为动态类型的很多优势。

很多时候变量名本身就自带类型信息，如“name”几乎肯定是str类型的；“length”肯定是个int；“value”多是float。没有必要给这些变量加typehint

有些变量的类型过于复杂，比如通过字典来给一个函数传递参数，这个字典可能嵌套更多的字典和列表，以至于这个类型过于复杂难以用typehint来标注。
即使你标注了，代码的读者也很难通过你留下的typehint来猜测这个字典到底是怎么回事。这种情况，不如放弃typehint，老老实实在docstring里写下关于这个字典的
说明和样例，更能让读者了解到有用的信息。

## 运行时应用

刚才提到，Python解释器不会在运行时检查类型信息，但这不代表你的typehint不会有运行时的影响。
事实上，几乎所有的typehint都会被保存在所属的对象的`__annotations__`字段中，并能在运行时被读取。这就允许我们在运行时利用这些信息并做一些有趣的应用。

一些潜在或已经实现的关于typehint的应用包括：

### 运行时类型检查

解释器不帮我们做，我们可以自己做。
[https://gist.github.com/SnowWalkerJ/a0a98acfdacb4b9ad75b5605ff220176](type check)是我写的一个运行时类型检查器，通过读取一个函数的`__annotations__`字段，并和实际传入的参数类型比较，如果不匹配就报TypeError错误

### 运行时函数重载

基于运行时类型检查，很容易就可以实现Python的函数重载，免于反复做isinstance判断。相关代码仍旧在[https://gist.github.com/SnowWalkerJ/a0a98acfdacb4b9ad75b5605ff220176](type check)中

### 依赖类型的性能加速

Cython、numba都是比较好的例子。

我自己写的[https://github.com/SnowWalkerJ/StaticPy](StaticPy)也是一个大量依赖typehint，并把python代码翻译为C++代码的工具，既能实现代码生成，也能自动绑定会Python实现加速。

### 命令行解析

[https://github.com/google/python-fire](fire)是一个优秀的命令行解析工具，但是它解析的命令行参数的类型却不是十分智能。如果可以结合typehint来将参数转换成正确的类型，将会十分有用。

### 自动生成API

[https://github.com/encode/apistar](apistar)充分利用typehint来自动生成restful api，节省了大量的变成工作，同时提供了充分的灵活性。

## 参考资料

- PEP484, 526, 544
- [https://docs.python.org/3/library/typing.html](https://docs.python.org/3/library/typing.html)
