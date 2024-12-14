---
layout: post
title: Crafting Excellence Effective C++ #1
author: Yuhan Zhang
tags:
- jekyll theme
- jekyll
date: 2024-12-13 22:00 +0800
---
# 0 导读
作者提出的忠告大致分为两类：一般性的设计策略，以及带有具体细节的特定语言特性。

例如什么是赋值（assignment）操作符的return type?何时改令析构函数为virtual？当operatornew无法找到足够内存时该怎么办？这些细节很是重要。
## 术语（Terminology）
**声明式（Declaration）**：就是告诉编译器某个东西的名称和类型，但略去细节
```cpp
extern int x;                       // object
std::size_t numDigits(int number);  // function(notion: std::size_t = unsigned int)
class Widget;                       // class
template<typename T>;               // template
class GraphNode;                    // "typename"的使用见条款42
```
值得注意的是，此处将基本类型int等和用户自定义类型均视为object。每个函数的声明式揭示其signature，即参数和返回类型。

**定义式（Definiton）**的任务时提供给编译器一些声明式所遗漏的细节。于对象而言，则是编译器为其分配内存的地点。针对function或者function template，定义式提供了代码本体；但对于class或者class template，定义式则是列出其成员。
```cpp
int x;                              // object definition
std::size_t numDigits(int number)   // function definition
{
    std::string digits = std::to_string(number);
    return digits.length();
}
class Widget {                      // class definition
    Widget();
    ~Widget();
    ...
};

template<typename T>
class GraphNode  {                   // class template definition
public:
    GraphNode();
    ~GraphNode();
    ...
}
```
**Tip 1**：如果是`function`或者`function template`，定义式就是代码本身，你必须提供一个包括函数体的完全体；而针对`class`或者`class template`，列出其成员（成员变量和成员函数）就构成了定义，这里的成员函数本身可以在类定义之外单独定义。

**Tip 2**：为什么`int x;`是定义式而`extern int x;`是声明式？因为前者为x分配了内存，而后者没有。

**初始化（Initialiation）**：是*给对象赋初值*的过程。对于用户自定义类型的对象而言，初始化由其构造函数执行。而default构造函数是一个可被调用而不带任何实参的构造函数，要么没有参数，要么就是每个参数都有缺省值：
```cpp
class A {
    A();                                    // default constructor
};

class B {
public:
    explicit B(int x = 0, bool b = true);   // default constructor
};

class C {
public:
    explicit C(int x);                      // not default constructor  
};
```
上述的`class B/C`的构造函数都被声明为explicit，意味着不能执行隐式类型转换（implicit type conversions），但仍可以用来进行显示类型转换（explicit typeconversions）:
```cpp
void doSomething(B bObj);                   // 函数，接受一个B类的实例
B bObj1;
doSomething(bObj1);                         // OK
B bObj2(28);                                // OK, 根据int 28构造一个B，bool参数缺省为true
doSomething(28);                            // Error!!! 函数应该接受一个B，而不是int，并且没有隐式转换
doSomething(B(28));                         // OK, 显示类型转换
```
被声明为explicit的构造函数通常比其non-explicit兄弟更受欢迎，因为explicit构造函数不能用于隐式类型转换，所以它们不能被用来实现非预期行为。

**Tip 3**:
如果一个类只有没有任何成员变量，并且只有一个不带参数的default的构造函数，那么可以如下定义：
```cpp
class Empty1 {};

class Empty2 { 
public:
    Empty2() {} 
}; // 与Empty1等价
```
这是因为编译器会提供默认的构造函数和析构函数。

**Question 1**:
给出如下代码的输出，并解释为什么？
```cpp
Empty1 a;
Empty2 *b;
std::cout << sizeof(a) << sizeof(b) << std::endl;
```

copy构造函数用于*通过同类对象实例化自身对象*，cpoy aeeignment操作符则用于*通过同类对象赋值给自身对象赋值*。
```cpp
class Widget {
public:
    Widget();                               // default constructor
    Widget(const Widget& rhs);              // copy constructor
    Widget& operator=(const Widget& rhs);   // copy assignment operator
    ~Widget();                              // destructor
};
Widget w1;                                 // 调用default构造函数
Widget w2(w1);                             // 调用copy构造函数
w1 = w2;                                   // 调用copy assignment操作符
```
当你看到赋值符号请小心，因为=也可用来调用copy构造函数：
```cpp
Widget w3 = w2;                           // 调用copy构造函数
```
区别copy构造和copy赋值的关键在于*是否有新对象被定义*。