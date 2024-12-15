---
layout: post
title: Crafting Excellence with Effective C++ Ch.1
author: Yuhan Zhang
tags:
- cpp
date: 2024-12-15 22:00 +0800
---
# 1 Accustoming Yourself to C++
## Item.1: View C++ as a federation of languages（将c++视作一个语言联邦）
C++ 最初是 C 的扩展，加入了面向对象（object-oriented）的特性。它最初的名字 "C with Classes" 就反映了这一点。

然而，今天的 C++ 已经演变为一种 多范式编程语言（multiparadigm programming language），支持 过程式编程（procedural）、面向对象编程（object-oriented）、函数式编程（functional）、泛型编程（generic） 和 元编程（metaprogramming） 等多种编程范式。

这种多样性使得 C++ 极具威力，但也带来了一些复杂性：几乎每种“”正确用法”似乎都存在例外。那么，我们该如何理解这样一种语言呢？  

最简单的方法是将 C++ 看作一个由 多个次语言（sublanguage）组成的联合体，而不是一个单一的语言。在每个次语言中，规则和惯例通常简单且直观。但当切换到另一个次语言时，这些惯例可能会发生变化。为了更好地理解 C++，你需要认识到其中的四个主要次语言：

- C 

C++ 的核心依然建立在 C 的基础之上。代码块（blocks）、语句（statements）、预处理器（preprocessor）、内置数据类型（built-in data types）、数组（arrays） 和 指针（pointers） 都源自 C。许多时候，C++ 提供的是对 C 中问题的高级解决方案（例如，*Item.2* 讨论了预处理器的替代方案，*Item.13* 涉及使用对象管理资源）。然而，当你在 C++ 的 C 部分中编程时，编程规范受限于 C 的局限性：没有 模板（templates）、没有 异常（exceptions）、没有 重载（overloading）等。 

- Object-Oriented C++

这部分也就是C with Classes 所诉求的，classes（包括构造和析构），封装encapsulation，继承inheritance，多态polymorphism，虚函数virtual functions动态绑定等。
- Template C++

模板（template）相关的考量贯穿整个 C++ 语言。有许多编程准则仅适用于模板（例如，*Item.46* 讨论了在调用模板函数时如何协助类型转换），进而有了新的编程范式--模板元编程（template metaprogramming, TMP）。虽然 *Item.48* 提供了 TMP 的概述，但除非你是“模板激进派（template enthusiast）”的核心成员，大多数情况下无需过于担忧这些内容。TMP 的规则很少会与 C++ 的主流编程实践交叉。
- STL

STL 是一个基于模板（template）的库，它为 容器（containers）、迭代器（iterators）、算法（algorithms） 和 函数对象（function objects） 提供了一个紧密配合且协调一致的框架。

记住这四个次语言，当你从某个次语言切换到另一个，导致高效编程守则要求你改变策略时，不要惊讶。例如对内置（也就是C-like）类型而言*pass-by-value*通常比*pass-by-referenc*e高效，但是当你从C part of C++ 切换到 Object-Oriented C++ 时，由于用户自定义（user-defined）构造函数和析构函数的存在，*pass-by-reference-to-const*往往更好。运用Template C++时尤其如此，因为彼时你甚至不知道所处理的对象的类型。然而一旦跨入STL你就会了解，迭代器和函数对象都是在C指针之上塑造出来的，因此旧式的C *pass-by-value*再次适用（参数传递方式的选择细节请见*Item.20*）。

## Item.2: Prefer consts, enums, and inlines to #defines（优先使用const、enum、inline替换#define）

#define的问题在于，它不被视为语言的一部分。

当你`#define ASPECT_RATIO 1.653`，记号名称ASPECT_RATIO或许从未被编译器看见，在编译器开始处理源码之前它就被预处理其移走了，没进入记号表（symbol table）内。于是当你运用此常量，进而捕获一个编译错误信息提到1.653而不是ASPECT_RATIO时，可能会带来困惑，尤其是在该常量被定义在一个非你自己缩写的头文件内。

解决之道是以一个常量替换上述的宏（#define）：
```cpp
const double ASPECT_RATIO = 1.653;  
```
作为一个语言常量，ASPECT_RATIO肯定会被编译器看到，因此它被置于记号表内。

此外对于浮点常量（floating point constant）而言，使用常量可能比适用#define导致较小量的码，因为预处理器“盲目地将宏名称ASPECT_RATIO替换成1.653”可能导致目标码（object code）出现多分1.653；而常量则不会。

**Tip 4**:
使用const常量代替#define有以下几点优势
1. 可见性：存储在记号表内，方便问题定位
2. 避免重复代码：编译器确保const常量仅存储一次（通常在制度数据段），而不会像宏直接进行多次替换，减少重复代码生成的可能性
3. 类型安全：宏定义的值是无类型的，但const常量有明确的类型，编译器会进行类型检查，从而避免隐式类型转换带来的问题。

