---
layout: post
title: "6model - Data Structures"
date: 2013-07-26 17:02
categories: 
- perl6 
- translation
---

本文是[6model/overview.pod](https://github.com/jnthn/6model/blob/master/overview.pod)中一个段落的翻译。

# Data Structures
  6model的核心是三种类型的数据结构。

## Objects
  除了内建的类型，所有用户直接接触的东西是`Object`。在`6model`中，`Object`是用户唯一会接触的数据类型。一个`Object`是一片连续的存储空间。惟一的限制是这片存储空间的第一个位置是一个指向一个`Shared Table`的指针/引用。

## Representations
  一个`Object`开头是一指向`Shared Table`的指针/引用，我们当然需要知道它其余的部分代表什么，这就是`Representation`的工作。一个`Representation`处理一个`Object`的内存申请，属性(Attribute)的存储和操作(存储和操作意义上)，装箱和拆箱(boxing and unboxing，值和引用类型之间的转换), GC(garbage collection, 取决于VM)。`Representation`更像于单例(Singleton),或者表现得更像实例(Instance)。一个`Representation`如何被实现取决于所使用的VM。事实上基本上`Representation`需要做的所有事都与VM有关。

## Shared Tables
  对于每个对象,使用一个`meta-object`来描述它的语义(Semantics)，使用`Representation`来描述他的存储结构。相对于在一个`Object`中存储多个指针，使用一个指向`Shared Table`的指针能使`Object`保持更小的大小。

        +--------+       +----------------+
        | Object |   +-->|  Shared Table  |
        +--------+   |   +----------------+
        | STABLE |---+   | Meta-object    |-----> Just another Object
        |  ....  |       | Representation |-----> Representation data structure
        |  ....  |       | V-table Cache  |-----> Array
        +--------+       | Type object    |-----> Just another Object
                         | <other bits>   |
                         +----------------+
