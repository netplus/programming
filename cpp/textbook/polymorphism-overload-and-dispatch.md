# C++ 多态、重载与函数派发：从对象模型到调用规则

> **语义基线：C++17**  
> 本文延续 `object-lifetime-and-special-member-functions.md` 的学习方法：先建立对象模型与调用模型，再推导 overload / name lookup / hiding / override / virtual dispatch 等规则。  
> 学习目标不是记忆“哪个函数被调用”，而是能够把一次调用拆成若干独立阶段，并从静态类型、动态类型、名字查找和重载决议自行推出结果。

---

## 目录

1. [为什么 C++ 需要多态与重载](#1-为什么-c-需要多态与重载)
2. [从对象模型开始：Base 子对象与 most-derived object](#2-从对象模型开始base-子对象与-most-derived-object)
3. [静态类型与动态类型](#3-静态类型与动态类型)
4. [两类多态：静态多态与动态多态](#4-两类多态静态多态与动态多态)
5. [一次函数调用其实分很多阶段](#5-一次函数调用其实分很多阶段)
6. [名字查找：先决定“看得见谁”](#6-名字查找先决定看得见谁)
7. [函数重载到底是什么](#7-函数重载到底是什么)
8. [哪些差异可以构成重载，哪些不可以](#8-哪些差异可以构成重载哪些不可以)
9. [成员函数的隐式对象参数与 const/ref 重载](#9-成员函数的隐式对象参数与-constref-重载)
10. [候选函数、可行函数与最佳可行函数](#10-候选函数可行函数与最佳可行函数)
11. [隐式转换序列与重载优先级](#11-隐式转换序列与重载优先级)
12. [`nullptr` 为什么容易形成重载二义性](#12-nullptr-为什么容易形成重载二义性)
13. [动态类型不参与 overload resolution](#13-动态类型不参与-overload-resolution)
14. [继承中的名字隐藏 name hiding](#14-继承中的名字隐藏-name-hiding)
15. [`using Base::f` 到底改变了什么](#15-using-basef-到底改变了什么)
16. [override：它和 overload / hiding 根本不是一层机制](#16-override它和-overload--hiding-根本不是一层机制)
17. [override 的精确匹配条件](#17-override-的精确匹配条件)
18. [`virtual`、`override`、`final` 的职责分工](#18-virtualoverridefinal-的职责分工)
19. [函数不可见也可能发生 override](#19-函数不可见也可能发生-override)
20. [协变返回类型](#20-协变返回类型)
21. [虚函数调用：overload resolution 之后才进入动态派发](#21-虚函数调用overload-resolution-之后才进入动态派发)
22. [final overrider：运行时究竟执行哪一个实现](#22-final-overrider运行时究竟执行哪一个实现)
23. [显式限定为什么会关闭虚派发](#23-显式限定为什么会关闭虚派发)
24. [默认参数为什么是静态绑定](#24-默认参数为什么是静态绑定)
25. [虚函数 + 默认参数：为什么会出现 Derived::f(1)](#25-虚函数--默认参数为什么会出现-derivedf1)
26. [构造与析构期间为什么不会派发到更派生类](#26-构造与析构期间为什么不会派发到更派生类)
27. [对象切片为什么会让动态多态消失](#27-对象切片为什么会让动态多态消失)
28. [多态删除与 virtual destructor](#28-多态删除与-virtual-destructor)
29. [vptr / vtable：典型实现，而不是语言规则本身](#29-vptr--vtable典型实现而不是语言规则本身)
30. [多继承中的三种不同“二义性”](#30-多继承中的三种不同二义性)
31. [纯虚函数与抽象类](#31-纯虚函数与抽象类)
32. [`dynamic_cast` 与 RTTI：运行时类型关系检查](#32-dynamic_cast-与-rtti运行时类型关系检查)
33. [`typeid` 与多态对象](#33-typeid-与多态对象)
34. [完整对照表：overload / override / hiding / virtual dispatch](#34-完整对照表overload--override--hiding--virtual-dispatch)
35. [统一推导算法：看到一次调用应该怎样分析](#35-统一推导算法看到一次调用应该怎样分析)
36. [综合案例：把所有阶段串起来](#36-综合案例把所有阶段串起来)
37. [常见误区](#37-常见误区)
38. [工程设计建议](#38-工程设计建议)
39. [学习检查点](#39-学习检查点)
40. [参考规则索引](#40-参考规则索引)

---

# 1. 为什么 C++ 需要多态与重载

先从两个完全不同的问题开始。

第一个问题：

```cpp
void print(int);
void print(double);
void print(const std::string&);
```

程序员希望表达：

> **同一个抽象动作 `print`，面对不同参数类型时采用不同实现。**

这是 overload 解决的问题。

第二个问题：

```cpp
struct Shape {
    virtual double area() const = 0;
};

struct Circle : Shape {
    double area() const override;
};

struct Rectangle : Shape {
    double area() const override;
};
```

程序员希望表达：

> **调用者只依赖 Shape 接口，但真正执行哪个实现由对象实际是什么类型决定。**

这是 runtime polymorphism / virtual dispatch 解决的问题。

两者表面上都表现成：

```text
“同一个函数名最后执行不同函数”
```

但底层机制完全不同：

```text
overload
→ 编译期根据调用表达式选择函数

virtual dispatch
→ 编译期先确定虚函数接口
→ 运行时再根据对象动态类型选择 final overrider
```

如果不把这两个阶段分开，后面的 name hiding、override、默认参数都会混成一团。

---

# 2. 从对象模型开始：Base 子对象与 most-derived object

考虑：

```cpp
struct Base {
    int b;
    virtual void f();
};

struct Derived : Base {
    int d;
    void f() override;
};

Derived obj;
```

从对象模型看，`obj` 不是“一个 Derived 再附带一个 Base 标签”。

它包含一个真实存在的 `Base` 基类子对象：

```text
Derived complete object
┌──────────────────────────────┐
│ Base subobject               │
│   └── b                      │
├──────────────────────────────┤
│ Derived own state            │
│   └── d                      │
└──────────────────────────────┘
```

因此：

```cpp
Base* p = &obj;
```

从概念上说，`p` 指向的是：

```text
Derived complete object 中的 Base subobject
```

但是这个 Base 子对象仍属于一个更大的：

```text
most-derived object = Derived
```

这就是动态多态能够成立的对象基础。

如果把 Base 子对象单独复制出来：

```cpp
Base b = obj;
```

新得到的 `b` 已经是：

```text
独立的 Base complete object
```

它不再属于一个 Derived most-derived object。

这就是后面 object slicing 的根本原因。

---

# 3. 静态类型与动态类型

多态最重要的一对概念是：

```text
static type
vs
dynamic type
```

考虑：

```cpp
Derived d;
Base& r = d;
Base* p = &d;
```

对表达式：

```cpp
r
```

可以理解为：

```text
静态类型：Base
动态对象：Derived
```

对：

```cpp
*p
```

同样：

```text
静态类型：Base
动态对象：Derived
```

## 3.1 静态类型解决什么问题

静态类型在编译期已知，它影响：

```text
名字查找
可见成员
重载候选集合
overload resolution
默认参数选择
非虚成员函数调用
隐式转换规则
```

## 3.2 动态类型解决什么问题

动态类型只在特定运行时机制中发挥作用，最典型的是：

```text
virtual dispatch
dynamic_cast
typeid（多态 glvalue 场景）
```

因此必须建立一个非常牢固的原则：

> **C++ 的普通名字查找与 overload resolution 不会因为运行时对象实际是 Derived，就突然把 Derived 的成员加入候选集合。**

动态类型不是重载决议输入。

---

# 4. 两类多态：静态多态与动态多态

“多态”在工程语境中通常可以分成两大类。

## 4.1 静态多态

包括：

```text
function overload
operator overload
templates
CRTP 等
```

例如：

```cpp
void f(int);
void f(double);

f(1);     // 编译期选择 f(int)
f(1.0);   // 编译期选择 f(double)
```

核心特点：

```text
候选集合、参数类型、转换排名
都能在编译期确定
```

## 4.2 动态多态

例如：

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    void f() override;
};

Base& r = derived;
r.f();
```

编译期能确定：

```text
“这里调用的是 Base::f 这条 virtual interface”
```

但最终函数体要等到运行时根据 most-derived object 决定。

所以可以把两者画成：

```text
静态多态
call expression
    ↓
compile-time overload resolution
    ↓
某个具体函数

动态多态
call expression
    ↓
compile-time lookup + overload resolution
    ↓
某条 virtual function interface
    ↓
runtime final overrider selection
```

---

# 5. 一次函数调用其实分很多阶段

这是整篇教材最重要的总图。

考虑：

```cpp
obj.f(arg);
```

不要直接问：

```text
“最后调用哪个 f？”
```

应该拆成：

```text
阶段 1：确定表达式静态类型
        ↓
阶段 2：name lookup
        ↓
阶段 3：形成 candidate functions
        ↓
阶段 4：筛选 viable functions
        ↓
阶段 5：比较 implicit conversion sequences
        ↓
阶段 6：得到 best viable function
        ↓
阶段 7：检查 deleted / access 等可用性
        ↓
阶段 8：若是 virtual 且未被显式限定
        ↓
阶段 9：根据 dynamic type 找 final overrider
```

默认参数还要插在静态阶段理解：

```text
缺失实参
→ 使用静态查找到的函数声明上的 default argument
```

因此：

> **名字查找、重载决议、虚派发不是一个步骤。**

后面所有复杂题都只是这条流水线的组合。

---

# 6. 名字查找：先决定“看得见谁”

C++ 在 overload resolution 之前，必须先做 name lookup。

例如：

```cpp
void f(int);
void f(double);

f(1);
```

名字查找先找到同一作用域中的：

```text
f(int)
f(double)
```

它们形成 overload set。

然后才进入 overload resolution。

因此：

```text
name lookup
≠
overload resolution
```

## 6.1 为什么这个区分在继承中特别重要

考虑：

```cpp
struct Base {
    void f(int);
};

struct Derived : Base {
    void f(double);
};

Derived d;
d.f(1);
```

很多人会想：

```text
f(int) 比 f(double) 更匹配
→ 应该调用 Base::f(int)
```

错误发生在更早阶段。

对于 `d.f` 的成员名字查找，从 `Derived` 开始：

```cpp
Derived::f(double)
```

已经找到同名声明，于是基类同名声明被隐藏。

因此 overload resolution 根本没有机会比较：

```text
Base::f(int)
vs
Derived::f(double)
```

它看到的只有：

```text
Derived::f(double)
```

于是 `1` 转成 `double`。

这说明：

> **最好的重载也必须先“活过 name lookup”才有资格参加比赛。**

---

# 7. 函数重载到底是什么

函数重载的核心是：

> **同一作用域中，同名函数存在多个可区分的声明，由 overload resolution 在调用点选择最佳候选。**

例如：

```cpp
void f(int);
void f(double);
void f(const char*);
```

调用：

```cpp
f(10);
f(3.14);
f("hello");
```

分别选择不同函数。

注意重载是：

```text
同一名字
+
同一查找集合
+
可区分的参数/成员函数限定信息
```

不是“只要全程序里有多个同名函数就是重载”。

作用域和名字查找决定它们能否进入同一个 overload set。

---

# 8. 哪些差异可以构成重载，哪些不可以

## 8.1 参数类型不同：可以

```cpp
void f(int);
void f(double);
```

## 8.2 参数数量不同：可以

```cpp
void f(int);
void f(int, int);
```

## 8.3 引用类型不同：可以

```cpp
void f(int&);
void f(const int&);
```

## 8.4 指针所指对象 cv 不同：可以

```cpp
void f(int*);
void f(const int*);
```

这里参数类型确实不同。

## 8.5 仅返回类型不同：不可以

```cpp
int    f(int);
double f(int);   // error
```

因为调用：

```cpp
f(1);
```

单凭实参无法选择返回类型版本。

## 8.6 by-value 参数的顶层 const 不构成重载

```cpp
void f(int);
void f(const int);   // 同一个函数的重复声明，不是新 overload
```

原因是对按值传参：

```text
调用者给函数的是一个新的参数对象
```

参数对象内部是否允许函数体修改，是函数实现内部属性，不改变调用接口的实参匹配。

对比：

```cpp
void f(int*);
void f(const int*);
```

这里 `const` 修饰的是：

```text
指针所指向的对象
```

所以参数类型不同。

## 8.7 默认参数不能单独制造 overload

```cpp
void f(int x);
void f(int x = 1);   // 同一函数的后续声明，不是 overload
```

默认参数甚至不是函数类型的一部分。

## 8.8 noexcept 不能单独制造 overload

不能仅靠异常说明区别两个同名同参数函数。

因此以后判断“能否 overload”，不要只看源码文本是否不同，而要看语言是否把这些差异视为不同调用接口。

---

# 9. 成员函数的隐式对象参数与 const/ref 重载

成员函数重载比普通函数多一个维度：

```text
调用这个函数的对象本身
```

考虑：

```cpp
struct A {
    void f();
    void f() const;
};
```

可以建立一个近似心智模型：

```text
void f(A* this)
void f(const A* this)
```

真实标准模型更精细，但这种近似非常适合理解 overload resolution。

于是：

```cpp
A a;
const A ca;

a.f();    // 非 const 版本更匹配
ca.f();   // 只能调用 const 版本
```

## 9.1 ref-qualifier 也可参与成员函数重载

```cpp
struct A {
    void f() &;
    void f() &&;
};
```

那么：

```cpp
A a;
a.f();       // & 版本
A{}.f();     // && 版本
```

这说明 value category 不只影响普通参数，还可以影响“对象参数”的重载选择。

---

# 10. 候选函数、可行函数与最佳可行函数

name lookup 成功以后，并不意味着所有找到的函数都能调用。

重载决议可以分成三层。

## 10.1 Candidate functions

名字查找和具体调用上下文形成候选集合。

例如：

```cpp
void f(int);
void f(double);
void f(const char*);
```

对：

```cpp
f(1);
```

三个都可能先进入候选集合。

## 10.2 Viable functions

候选函数必须满足调用基本条件。

例如：

```cpp
void f(int);
void f(int, int = 0);
void f(int, int, int);
```

对：

```cpp
f(1);
```

前两个参数数量都能成立：

```text
f(int)               viable
f(int, int = 0)      viable
```

第三个缺少参数且没有足够默认值：

```text
not viable
```

此外，每个实参必须能转换/绑定到对应参数。

## 10.3 Best viable function

如果有多个 viable candidate，比较每个参数需要的 implicit conversion sequence。

必须存在唯一最佳函数，overload resolution 才成功。

否则：

```text
ambiguous
→ ill-formed
```

---

# 11. 隐式转换序列与重载优先级

重载决议本质上在问：

> **哪一个函数需要对实参做“更好的转换”？**

学习初期可以先掌握下面的主层级：

```text
exact match
    ↓ 优于
promotion
    ↓ 优于
ordinary conversion
    ↓ 优于
user-defined conversion
    ↓ 优于
ellipsis
```

## 11.1 exact match

```cpp
void f(int);
void f(double);

f(1);
```

`1` 是 `int`，所以：

```text
int -> int      exact
int -> double   conversion
```

选择：

```cpp
f(int)
```

## 11.2 promotion 比普通 conversion 好

```cpp
void f(int);
void f(long);

short s = 1;
f(s);
```

通常：

```text
short -> int    integral promotion
short -> long   integral conversion
```

选择 `f(int)`。

## 11.3 “两个都能转”不代表有唯一最佳

```cpp
void f(long);
void f(double);

f(1);
```

从 `int` 到 `long` 与到 `double` 都属于标准转换，但在这种组合中可能没有一个转换序列严格更好，于是会产生二义性。

所以不要只判断：

```text
“能不能转换”
```

还要判断：

```text
“谁更好”
```

---

# 12. `nullptr` 为什么容易形成重载二义性

考虑：

```cpp
void f(int*);
void f(double*);

f(nullptr);
```

`nullptr` 的类型是：

```cpp
std::nullptr_t
```

它可以进行空指针转换到不同指针类型。

所以：

```text
nullptr -> int*       viable
nullptr -> double*    viable
```

两个转换没有谁优于谁。

因此：

```text
ambiguous
```

对比：

```cpp
void f(int);
void f(int*);

f(nullptr);
```

这里 `nullptr` 可以转换为 `int*`，但不是普通整数 `0`，不会作为 `int` 版本的合法普通匹配。

最终选择：

```cpp
f(int*)
```

这一例子很好地说明：

```text
overload resolution
=
候选集合 + 类型系统 + 转换排名
```

---

# 13. 动态类型不参与 overload resolution

这是多态与重载交叉时最重要的规则之一。

```cpp
struct Base {};
struct Derived : Base {};

void g(Base&);
void g(Derived&);

Derived d;
Base& r = d;

g(r);
```

虽然 `r` 实际引用的是 `Derived` 对象，但调用表达式：

```cpp
g(r)
```

中 `r` 的静态类型是：

```text
Base
```

因此 overload resolution 选择：

```cpp
g(Base&)
```

不会因为运行时对象是 Derived 而选择：

```cpp
g(Derived&)
```

可以把原则写成：

> **overload resolution 是静态多态；它只看编译期可见的类型和值类别。**

动态类型只有在后续 virtual dispatch 等运行时机制中参与。

---

# 14. 继承中的名字隐藏 name hiding

考虑：

```cpp
struct Base {
    void f(int);
    void f(const char*);
};

struct Derived : Base {
    void f(double);
};
```

`Derived` 声明了一个同名 `f`。

对于：

```cpp
Derived d;
d.f(1);
```

成员名字查找首先在 Derived 中发现：

```cpp
Derived::f(double)
```

于是 Base 中同名成员被隐藏。

注意：

> **hiding 按“名字”发生，不要求参数列表相同。**

所以：

```cpp
Base::f(int)
Base::f(const char*)
```

都会被同名 `Derived::f(double)` 挡在当前 lookup 之外。

因此：

```cpp
d.f(1);
```

调用：

```cpp
Derived::f(double)
```

而不是 Base::f(int)。

这不是 overload resolution 选错了，而是 Base 函数根本没进入 overload set。

---

# 15. `using Base::f` 到底改变了什么

如果希望 Derived 的直接调用同时看到 Base 的同名 overload，可以写：

```cpp
struct Derived : Base {
    using Base::f;
    void f(double);
};
```

现在：

```cpp
Derived d;
d.f(1);
```

成员 lookup 能看到：

```text
Base::f(int)
Base::f(const char*)
Derived::f(double)
```

然后再进入 overload resolution。

于是 `1` 对：

```cpp
Base::f(int)
```

是 exact match，优于转成 double。

最终调用 Base::f(int)。

因此：

> **`using Base::f` 主要改变 name lookup / overload set。**

它不是“开启 virtual”，也不是“改变 override 关系”。

---

# 16. override：它和 overload / hiding 根本不是一层机制

考虑：

```cpp
struct Base {
    virtual void f(int);
};

struct Derived : Base {
    void f(int) override;
};
```

这里 Derived::f 并不是和 Base::f 形成普通意义上的“同一作用域 overload”。

它建立的是：

```text
virtual overriding relation
```

也就是：

```text
Base::f(int)
    ↓ virtual interface
Derived::f(int)
    ↓ overrides
```

对比：

```cpp
struct Derived : Base {
    void f(double);
};
```

这里 `f(double)`：

```text
名字相同
→ 会 hide Base::f

参数列表不同
→ 不 override Base::f(int)
```

这就是为什么：

```text
同名
≠
override
```

---

# 17. override 的精确匹配条件

在 C++17 中，一个派生类成员函数要 override 基类 virtual function，核心匹配条件包括：

```text
同名
相同 parameter-type-list
相同 cv-qualification
相同 ref-qualifier（或都没有）
```

返回类型要求：

```text
相同
或满足协变返回规则
```

## 17.1 const 不同就不是 override

```cpp
struct Base {
    virtual void f() const;
};

struct Derived : Base {
    void f();   // 不 override Base::f() const
};
```

因为：

```text
Base::f      const
Derived::f   non-const
```

这是两个不同成员函数接口。

## 17.2 ref qualifier 不同也不是 override

```cpp
struct Base {
    virtual void f() &;
};

struct Derived : Base {
    void f() &&;
};
```

也不形成 override。

## 17.3 default argument 不参与 override 匹配

```cpp
struct Base {
    virtual void f(int = 1);
};

struct Derived : Base {
    void f(int = 2) override;
};
```

它们仍然 override，因为真正参与函数接口匹配的是参数类型 `int`，不是默认值。

---

# 18. `virtual`、`override`、`final` 的职责分工

## 18.1 `virtual`

第一次在继承体系中声明动态派发接口：

```cpp
struct Base {
    virtual void f();
};
```

一旦函数成为 virtual，后续正确 override 的函数即使不再写 `virtual`，也仍然是 virtual。

```cpp
struct Derived : Base {
    void f();   // 仍是 virtual，并 override Base::f
};
```

## 18.2 `override`

`override` 的主要价值是：

> **让编译器验证“我确实打算 override 某个基类虚函数”。**

```cpp
struct Base {
    virtual void f(int);
};

struct Derived : Base {
    void f(long) override;   // error
};
```

没有 `override` 时，这种错误很容易静默变成：

```text
Derived::f(long) hides Base::f(int)
```

而不是程序员想要的 override。

因此工程上强烈建议：

```text
所有 override 都显式写 override
```

## 18.3 `final`

函数级 final：

```cpp
virtual void f() final;
```

表示后续派生类不得再 override。

类级 final：

```cpp
struct D final : Base {};
```

表示该类不得再被继承。

---

# 19. 函数不可见也可能发生 override

这是名字查找与 override 关系分离的最好例子。

```cpp
struct B {
    virtual void f();
};

struct D : B {
    void f(int);   // hides B::f()
};

struct D2 : D {
    void f();
};
```

在 D2 中做普通名字查找时，`D::f(int)` 会挡住更上层的 `B::f()`。

但是 override 判定不是简单沿用 ordinary member lookup。

`D2::f()` 仍能 override：

```cpp
B::f()
```

因此要记住：

> **visibility / hiding 和 overriding relation 是两个不同机制。**

甚至 access control 也不决定能否 override。

一个 private virtual function 仍然可以被派生类 override。

---

# 20. 协变返回类型

通常 override 要求返回类型相同：

```cpp
struct Base {
    virtual Base* clone();
};
```

派生类可以返回更具体的派生类指针：

```cpp
struct Derived : Base {
    Derived* clone() override;
};
```

这叫 covariant return type。

核心允许形式是：

```text
Base 返回 Base*
Derived 返回 Derived*
```

或对应的 class reference 类型。

这种设计很自然：

```text
通过 Base 接口调用
→ 至少能看作 Base*

通过 Derived 静态接口调用
→ 可以得到更精确 Derived*
```

注意协变只适用于特定 class pointer/reference 关系，不是任意返回类型都允许变化。

例如：

```cpp
virtual int f();
double f() override;   // 不允许
```

---

# 21. 虚函数调用：overload resolution 之后才进入动态派发

这是整篇教材的核心。

考虑：

```cpp
struct Base {
    virtual void f(int);
    void f(double);
};

struct Derived : Base {
    void f(int) override;
};

Derived d;
Base* p = &d;

p->f(1);
```

完整过程不是：

```text
看到 Derived 对象
→ 去 Derived 里找 f
```

而是：

```text
1. p 的静态类型是 Base*

2. 对 Base 做成员 name lookup
   找到：
   Base::f(int)
   Base::f(double)

3. overload resolution
   实参 1 是 int
   → 选择 Base::f(int)

4. 发现选中的 Base::f(int) 是 virtual

5. 检查 *p 所在 most-derived object
   dynamic type = Derived

6. 找 Base::f(int) 的 final overrider
   → Derived::f(int)

7. 执行 Derived::f(int)
```

这条流水线说明：

> **virtual dispatch 不会重新做一次 overload resolution。**

运行时只是沿已经选定的 virtual interface 找 final overrider。

---

# 22. final overrider：运行时究竟执行哪一个实现

假设：

```cpp
struct A {
    virtual void f();
};

struct B : A {
    void f() override;
};

struct C : B {
    void f() override;
};
```

当：

```cpp
C c;
A& r = c;
r.f();
```

静态阶段确定：

```text
调用 A::f 这条 virtual interface
```

在 most-derived object `C` 中，对这条 virtual function 的 final overrider 是：

```cpp
C::f
```

所以执行 C::f。

可以把它画成：

```text
A::f
  ↑ override
B::f
  ↑ override
C::f   ← final overrider for C object
```

`final overrider` 是比“看 vtable 最后一项”更标准、更抽象的语言概念。

---

# 23. 显式限定为什么会关闭虚派发

考虑：

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    void f() override;

    void g() {
        Base::f();
    }
};
```

这里：

```cpp
Base::f();
```

是显式 qualified call。

它明确要求：

```text
调用 Base::f 本身
```

因此 virtual dispatch 被抑制。

同理：

```cpp
p->Base::f();
```

也不是运行时选择 Derived override，而是明确调用 Base::f。

这非常符合语义：

> **当程序员已经通过 scope qualification 明确指定实现时，语言没有理由再动态改写目标。**

---

# 24. 默认参数为什么是静态绑定

默认参数不是函数体的一部分。

考虑：

```cpp
void f(int x = 10);
```

调用：

```cpp
f();
```

可以把编译期行为理解成：

```cpp
f(10);
```

默认参数信息附着在函数声明上，并在调用点根据可见声明使用。

它不是运行时函数对象中携带的一份“默认参数表”。

因此：

```text
default argument
→ compile-time call-site mechanism
```

而不是：

```text
runtime dispatch mechanism
```

这也是为什么默认参数不会自动“继承”到 overriding function。

---

# 25. 虚函数 + 默认参数：为什么会出现 `Derived::f(1)`

考虑：

```cpp
struct Base {
    virtual void f(int x = 1) {
        std::cout << "Base:" << x;
    }
};

struct Derived : Base {
    void f(int x = 2) override {
        std::cout << "Derived:" << x;
    }
};

Derived d;
Base* p = &d;

p->f();
```

正确结果是：

```text
Derived:1
```

必须拆成两个阶段。

## 25.1 静态阶段：默认参数

`p` 的静态类型：

```text
Base*
```

因此调用点看到：

```cpp
Base::f(int x = 1)
```

缺失参数被补成：

```cpp
p->f(1);
```

## 25.2 动态阶段：函数体

选中的函数接口：

```cpp
Base::f(int)
```

是 virtual。

`p` 实际指向 Derived most-derived object，所以 final overrider：

```cpp
Derived::f(int)
```

最终相当于：

```cpp
Derived::f(1)
```

因此：

```text
默认参数：静态绑定
函数体：动态绑定
```

这不是一条“奇怪例外”，而是两个不同语言阶段自然叠加的结果。

---

# 26. 构造与析构期间为什么不会派发到更派生类

考虑：

```cpp
struct Base {
    Base() {
        f();
    }

    virtual void f();
};

struct Derived : Base {
    int x;
    void f() override;
};
```

构造 `Derived` 时：

```text
Base subobject 先构造
Derived own part 后构造
```

在 `Base::Base()` 执行期间，Derived 部分还没有完成构造。

因此此时：

```cpp
f();
```

不会派发到：

```cpp
Derived::f()
```

而调用 Base 构造阶段对应的 final overrider。

为什么？

假设 Derived::f 可以运行：

```cpp
void Derived::f() {
    use(x);
}
```

但 `x` 此时可能尚未初始化。

语言因此把构造/析构阶段的“当前有效动态层级”限制在正在执行 constructor/destructor 的类。

析构同理：

```text
Derived 部分先结束
↓
再执行 Base destructor
```

进入 Base destructor 时，更派生部分已经处在销毁完成/不再可依赖的状态，所以不会再虚派发回 Derived。

---

# 27. 对象切片为什么会让动态多态消失

考虑：

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    int extra;
    void f() override;
};

void g(Base b) {
    b.f();
}

Derived d;
g(d);
```

关键不是 virtual，而是：

```cpp
void g(Base b)
```

按值传递会创建一个新的独立 Base 对象。

可以画成：

```text
原对象 d：
┌────────────────────┐
│ Base subobject     │
├────────────────────┤
│ Derived::extra     │
└────────────────────┘

按值复制后 b：
┌────────────────────┐
│ Base complete obj  │
└────────────────────┘
```

进入 `g()` 后，`b` 的 most-derived type 就是 Base。

所以：

```cpp
b.f();
```

调用 Base::f。

如果改成：

```cpp
void g(Base& b);
```

就没有新建 Base complete object，而只是保留对原 Derived 对象中 Base subobject 的引用。

于是动态类型仍是 Derived，virtual dispatch 才成立。

因此：

> **多态依赖对象身份被保留；按值 Base copy 会切掉 derived identity。**

---

# 28. 多态删除与 `virtual destructor`

考虑：

```cpp
Base* p = new Derived;
delete p;
```

这不是普通 `p->f()` 调用，而是在做：

```text
通过 Base 接口结束 most-derived object 的整个生命周期
```

如果 Base destructor 非 virtual：

```cpp
~Base();
```

通过 `Base*` 删除实际 Derived 对象的典型场景是未定义行为。

正确的多态基类通常应该写：

```cpp
struct Base {
    virtual ~Base() = default;
};
```

这样 delete 才能从 most-derived type 开始执行正确析构链：

```text
Derived::~Derived()
↓
Base::~Base()
```

所以 virtual destructor 本质上不是“为了让 Base destructor 自己变动态”，而是：

> **让“通过基类接口销毁完整动态对象”成为类型协议的一部分。**

---

# 29. vptr / vtable：典型实现，而不是语言规则本身

在主流 C++ ABI 中，多态类通常使用：

```text
vptr
+
vtable
```

实现 virtual dispatch。

一个典型概念布局可能是：

```text
Derived object
┌──────────────────────┐
│ Base/vptr            │ ───► virtual table
│ Base data            │
├──────────────────────┤
│ Derived data         │
└──────────────────────┘
```

调用：

```cpp
Base* p = &d;
p->f();
```

实现上可能近似：

```text
读取 p 所指 subobject 的 vptr
↓
在 vtable 中找到 f slot
↓
调用 Derived final overrider
```

但要明确：

> **C++ 标准保证 virtual semantics，不要求实现必须存在一个叫 vptr 或 vtable 的字段。**

多继承、虚继承下一个 complete object 甚至可能有多个与动态派发相关的表指针和 this-adjustment thunk。

因此学习时：

```text
标准层：final overrider / dynamic type
实现层：vptr / vtable / thunk
```

二者要分开。

---

# 30. 多继承中的三种不同“二义性”

多继承最容易把多种完全不同的问题统称成“二义性”。

必须拆开。

## 30.1 名字查找二义性

```cpp
struct B1 { void f(); };
struct B2 { void f(); };
struct D : B1, B2 {};

D d;
d.f();   // ambiguous
```

这里的问题是：

```text
member name lookup 找到两个同名来源
```

可以显式指定：

```cpp
d.B1::f();
```

## 30.2 基类转换路径二义性

```cpp
struct A {};
struct B1 : A {};
struct B2 : A {};
struct D : B1, B2 {};

D d;
A* p = &d;   // ambiguous
```

因为 D 中存在两个独立 A subobject：

```text
D
├── B1
│   └── A
└── B2
    └── A
```

编译器不知道你要哪个 A。

## 30.3 final overrider 二义性

虚继承场景可能只有一个共享 virtual base，但两条分支都 override 同一个 virtual function：

```cpp
struct A {
    virtual void f();
};

struct B1 : virtual A {
    void f() override;
};

struct B2 : virtual A {
    void f() override;
};

struct D : B1, B2 {
};
```

现在 shared A::f 对 D 对象存在两个竞争 final overrider：

```text
B1::f
B2::f
```

如果 D 不提供统一 override，类定义会有问题。

修复：

```cpp
struct D : B1, B2 {
    void f() override;
};
```

因此三类问题分别是：

```text
lookup ambiguity
base-subobject conversion ambiguity
final-overrider ambiguity
```

不要混成一条规则。

---

# 31. 纯虚函数与抽象类

```cpp
struct Shape {
    virtual double area() const = 0;
};
```

`= 0` 把 virtual function 声明为 pure virtual。

包含尚未获得 concrete final overrider 的 pure virtual function 的类是 abstract class，不能直接创建对象：

```cpp
Shape s;   // error
```

但可以有：

```cpp
Shape* p;
Shape& r = derived;
```

派生类提供 override：

```cpp
struct Circle : Shape {
    double area() const override;
};
```

于是 Circle 可以成为 concrete class。

纯虚函数表达的是：

> **Base 定义接口槽位，但不提供该类型可直接实例化所需的最终实现。**

纯虚析构函数是一个特别值得注意的工程场景：即使声明 pure virtual，基类析构在派生对象销毁链中仍然会被调用，所以仍需要可用定义。

---

# 32. `dynamic_cast` 与 RTTI：运行时类型关系检查

virtual dispatch 解决：

```text
“沿着已知虚接口调用哪个实现？”
```

`dynamic_cast` 解决的是另一个问题：

```text
“这个 Base 视图背后的动态对象到底能不能安全地看成 Derived？”
```

例如：

```cpp
Base* p = get_object();

if (auto* d = dynamic_cast<Derived*>(p)) {
    d->derived_only();
}
```

对于运行时 checked downcast/cross-cast，源类通常需要是 polymorphic class。

## 32.1 pointer cast 失败

```cpp
Derived* d = dynamic_cast<Derived*>(p);
```

失败结果：

```cpp
nullptr
```

## 32.2 reference cast 失败

```cpp
Derived& d = dynamic_cast<Derived&>(r);
```

失败会抛出：

```cpp
std::bad_cast
```

## 32.3 为什么需要运行时信息

普通 `static_cast<Derived*>(p)` 主要依据静态继承关系进行转换，不会像 `dynamic_cast` 一样验证对象的动态类型是否真的满足要求。

因此二者表达的意图不同：

```text
static_cast
→ 我根据静态关系确信这样转换合理

dynamic_cast
→ 请运行时验证动态对象关系
```

---

# 33. `typeid` 与多态对象

RTTI 的另一部分是：

```cpp
#include <typeinfo>
typeid(expr)
```

对于多态类的 glvalue 表达式，`typeid` 可以反映动态对象类型。

例如：

```cpp
Derived d;
Base& r = d;

std::cout << (typeid(r) == typeid(Derived));
```

在这个多态场景下，可以识别 Derived dynamic type。

而对非多态对象体系，`typeid` 通常主要体现静态类型语义。

工程上不要把 `typeid` 当成手工模拟 virtual dispatch 的常规工具。

如果代码大量出现：

```cpp
if (typeid(x) == ...)
```

往往值得重新审视接口是否应该通过 virtual function 表达行为差异。

---

# 34. 完整对照表：overload / override / hiding / virtual dispatch

| 机制 | overload | override | hiding | virtual dispatch |
|---|---|---|---|---|
| 发生阶段 | 编译期 | 类定义关系 | 名字查找 | 运行时调用阶段 |
| 是否需要继承 | 否 | 是 | 常见于继承 | 是 |
| 是否需要 virtual | 否 | Base 必须 virtual | 否 | 是 |
| 核心依据 | 参数匹配 / 转换排名 | 同名 + matching signature/cv/ref | 同名声明 | dynamic type + final overrider |
| 是否依赖动态类型 | 否 | 否 | 否 | 是 |
| 参数列表可不同吗 | 必须能区分 | 核心参数类型需匹配 | 可以不同 | 不重新比较参数 |
| 返回类型作用 | 不能单独 overload | 相同或协变 | 无关 | 已在 override 关系中确定 |
| `using Base::f` 影响 | 可以扩展 overload set | 不决定 override | 可以解除隐藏 | 不改变 final overrider 本身 |
| 默认参数作用 | 可影响 viability | 不参与 override 匹配 | 无关 | 默认值仍静态选择 |
| 常见错误 | ambiguity | 签名不匹配 | 误以为 Base overload 仍可见 | 误以为运行时重新 overload |

最重要的一句：

> **overload 决定“选哪条接口”；virtual dispatch 决定“这条虚接口在当前动态对象上执行哪个 final overrider”。**

---

# 35. 统一推导算法：看到一次调用应该怎样分析

以后看到：

```cpp
expr.f(args...);
```

固定按下面顺序。

## Step 1：对象有没有先被切片

先问：

```text
expr 是原多态对象的引用/指针视图？
还是已经按值复制成独立 Base object？
```

如果已经 slicing，动态类型可能早已变成 Base。

## Step 2：写出静态类型

例如：

```cpp
Base* p = &derived;
```

对：

```cpp
p->f()
```

静态对象类型从 Base 开始分析。

## Step 3：做 name lookup

问：

```text
当前静态类型作用域里哪些 f 可见？
有没有 Derived 同名成员 hiding Base overload？
有没有 using Base::f？
有没有显式 Base::f qualification？
```

## Step 4：形成 candidate set

只把 lookup 后可参与的函数放进候选集合。

## Step 5：筛 viable functions

检查：

```text
参数数量
默认参数
引用绑定
可转换性
对象 cv/ref qualifier
```

## Step 6：比较 conversion sequences

判断：

```text
exact / promotion / conversion / user-defined / ellipsis
```

以及引用绑定谁更优。

## Step 7：得到 best viable function

如果不存在唯一最佳：

```text
ambiguous / ill-formed
```

## Step 8：检查函数可用性

例如：

```text
deleted?
accessible?
```

注意 access/deleted 不是“提前从候选里随便忽略”的通用理由。

## Step 9：默认参数来自哪里

如果有缺失实参，使用静态调用上下文确定的函数声明上的默认参数。

## Step 10：选中的函数是不是 virtual

若不是：

```text
到此结束
```

若是：继续。

## Step 11：是否显式 qualified

如果：

```cpp
Base::f()
p->Base::f()
```

virtual dispatch 被抑制。

## Step 12：确定 dynamic type / most-derived object

如果通过 Base&/Base* 引用 Derived complete object：

```text
dynamic type = Derived
```

## Step 13：找 final overrider

沿 override 链找到当前 most-derived object 对选中 virtual function 的 final overrider。

## Step 14：检查是否处于 construction/destruction

如果虚调用发生在 ctor/dtor 期间，动态派发层级受当前正在构造/析构的类限制。

这套算法足以覆盖绝大多数中高级 C++ 多态/重载题。

---

# 36. 综合案例：把所有阶段串起来

考虑：

```cpp
#include <iostream>

struct Base {
    virtual void f(int x = 1) {
        std::cout << "Base-int:" << x << '\n';
    }

    void f(double x) {
        std::cout << "Base-double:" << x << '\n';
    }
};

struct Derived : Base {
    using Base::f;

    void f(int x = 2) override {
        std::cout << "Derived-int:" << x << '\n';
    }

    void f(const char* s) {
        std::cout << "Derived-str:" << s << '\n';
    }
};

Derived d;
Base& r = d;
```

现在逐个分析。

## 36.1 `d.f(1)`

静态类型：

```text
Derived
```

`using Base::f` 使 lookup 看到 Base overload，再加 Derived declarations。

候选大致包括：

```text
Derived::f(int)
Derived::f(const char*)
Base::f(double)
```

`int -> int` exact，选 Derived::f(int)。

它本身就是当前 Derived object 的 final overrider。

结果：

```text
Derived-int:1
```

## 36.2 `d.f(1.5)`

候选仍来自 Derived lookup set。

```text
double -> Base::f(double)    exact
double -> Derived::f(int)    conversion
```

所以选择：

```cpp
Base::f(double)
```

它不是 virtual，所以没有后续动态派发。

结果：

```text
Base-double:1.5
```

## 36.3 `r.f(1)`

静态类型：

```text
Base
```

因此 name lookup 只基于 Base interface：

```text
Base::f(int)
Base::f(double)
```

`1` 选择 Base::f(int)。

这个函数是 virtual。

`r` 实际引用 Derived object。

final overrider：

```cpp
Derived::f(int)
```

结果：

```text
Derived-int:1
```

## 36.4 `r.f()`

静态 lookup 仍然是 Base。

只有：

```cpp
Base::f(int x = 1)
```

能通过默认参数成为 viable。

调用点先形成：

```cpp
r.f(1)
```

再 virtual dispatch 到：

```cpp
Derived::f(1)
```

结果：

```text
Derived-int:1
```

不是 `Derived-int:2`。

## 36.5 `r.f(1.5)`

静态 Base overload set：

```text
Base::f(int)
Base::f(double)
```

`double` exact 选择：

```cpp
Base::f(double)
```

它不是 virtual。

因此结果：

```text
Base-double:1.5
```

即使 dynamic object 是 Derived，也不会重新去 Derived 搜索一个“更合适”的函数。

这一组例子把整个主题压缩成一句话：

```text
static type
→ lookup
→ overload
→ selected interface
→ only if virtual: dynamic dispatch
```

---

# 37. 常见误区

## 误区 1：同名就是 overload

错误。

派生类同名函数可能只是 hide 基类函数。

## 误区 2：同名就是 override

错误。

override 要满足 virtual + matching signature/cv/ref 等规则。

## 误区 3：运行时对象是 Derived，所以 overload resolution 会考虑 Derived overload

错误。

overload resolution 使用静态类型。

## 误区 4：virtual 会在运行时重新选择参数最匹配的函数

错误。

运行时只对已经静态选定的 virtual interface 找 final overrider。

## 误区 5：Derived 声明 `f(double)` 后，Base::f(int) 仍会自动一起 overload

错误。

同名派生成员会 hide Base 同名成员；需要 `using Base::f` 才能重新引入普通 lookup。

## 误区 6：`override` 会让函数变 virtual

不够准确。

函数之所以 virtual，是因为它确实 override 了基类 virtual function；`override` 是编译器校验标记。

## 误区 7：Derived override 必须再写 `virtual`

错误。

一旦 override virtual function，它天然继续 virtual。

## 误区 8：private virtual function 不能被 override

错误。

access control 不参与 override 关系判定。

## 误区 9：默认参数跟着 virtual dispatch 一起动态变化

错误。

默认参数按静态类型/静态声明选择。

## 误区 10：构造函数里调用 virtual 一定会进 Derived

错误。

构造/析构期间不会派发到尚未构造完成或已经销毁的更派生部分。

## 误区 11：只要函数是 virtual，按值传 Base 也能保持 Derived 多态

错误。

按值 copy 可能先发生 object slicing。

## 误区 12：vtable 是 C++ 标准规定的对象布局

错误。

vtable/vptr 是主流实现技术，不是语言层必须使用的具体数据结构。

## 误区 13：显式 `Base::f()` 仍会动态派发

错误。

explicit qualification 会抑制 virtual call mechanism。

## 误区 14：返回类型不同可以 overload

错误。

返回类型不能单独形成 overload。

## 误区 15：`void f(int)` 和 `void f(const int)` 是两个 overload

错误。

by-value 参数的顶层 const 不构成不同调用接口。

---

# 38. 工程设计建议

## 38.1 override 一律写 `override`

```cpp
void f(int) override;
```

不要只写：

```cpp
void f(int);
```

这样 signature 漂移会在编译期暴露。

## 38.2 避免 virtual function 使用不同 default arguments

虽然合法：

```cpp
Base::f(int = 1)
Derived::f(int = 2)
```

但它会制造：

```text
函数体动态
默认值静态
```

这种很难维护的接口。

更好的做法通常是把默认值集中到非虚包装层。

例如：

```cpp
struct Base {
    void f() { f_impl(1); }

private:
    virtual void f_impl(int) = 0;
};
```

这类设计也常被称为 Non-Virtual Interface 风格的一部分。

## 38.3 多态接口通过 pointer/reference 传递

避免：

```cpp
void process(Base b);
```

如果目标是保留动态多态，应使用：

```cpp
void process(Base& b);
void process(const Base& b);
```

或明确 ownership 的智能指针。

## 38.4 多态基类要明确销毁协议

如果允许：

```cpp
delete base_ptr;
```

基类析构应是 virtual。

如果设计上明确禁止通过 Base 删除，也应让接口明确体现这种限制，而不是留下模糊行为。

## 38.5 不要依赖名字隐藏制造“意外接口”

派生类如果需要保留基类 overload set，显式写：

```cpp
using Base::f;
```

比期待读者自行理解隐藏关系更清晰。

## 38.6 少用 RTTI 分支替代真正的虚接口

如果代码不断：

```cpp
if (dynamic_cast<A*>(p)) ...
else if (dynamic_cast<B*>(p)) ...
```

说明行为很可能应该回到类型体系中，以 virtual interface 表达。

RTTI 更适合确有类型检查语义的场景，而不是手工模拟动态派发。

---

# 39. 学习检查点

在进入下一轮题目之前，建议能够不查资料自行解释下面问题。

## 检查点 1

```cpp
struct Base {
    void f(int);
};

struct Derived : Base {
    void f(double);
};

Derived d;
d.f(1);
```

为什么不是 Base::f(int)？

必须能回答：

```text
问题发生在 name lookup 阶段，而不是 conversion ranking 阶段。
```

## 检查点 2

```cpp
struct Base {
    virtual void f(int);
};

struct Derived : Base {
    void f(double);
};
```

为什么 Derived::f(double) 不 override Base::f(int)？

## 检查点 3

```cpp
Derived d;
Base& r = d;

g(r);
```

如果存在：

```cpp
void g(Base&);
void g(Derived&);
```

为什么选择 `g(Base&)`？

## 检查点 4

为什么：

```cpp
Base* p = &derived;
p->f();
```

需要先做静态 overload resolution，再做动态 virtual dispatch？

## 检查点 5

为什么：

```cpp
Base::f()
```

会抑制 virtual dispatch？

## 检查点 6

为什么：

```cpp
virtual void f(int = 1);
```

的 default argument 不跟随 final overrider 动态变化？

## 检查点 7

为什么 Base constructor 内 virtual call 不会调用 Derived override？

要求从：

```text
subobject lifetime / construction order
```

解释，而不是只背“构造函数里虚函数失效”。

## 检查点 8

为什么：

```cpp
void process(Base b);
```

会破坏原 Derived object 的动态多态？

## 检查点 9

`using Base::f` 改变的是：

```text
lookup / overload set
```

还是：

```text
override relation / dynamic dispatch
```

为什么？

## 检查点 10

用一句话分别定义：

```text
overload
override
hiding
virtual dispatch
```

并说明四者分别发生在哪个阶段。

---

# 40. 参考规则索引

本文以 C++17 规则为基线，后续如需核对标准措辞，优先查看以下主题：

```text
[basic.lookup]          name lookup
[basic.scope.hiding]   name hiding
[class.member.lookup]  class member lookup
[over.match]           overload resolution
[over.match.viable]    viable functions
[over.match.best]      best viable function
[over.best.ics]        implicit conversion sequences
[class.virtual]        virtual functions / override / final overrider
[dcl.fct.default]      default arguments
[class.cdtor]          construction/destruction 中的 virtual call
[class.dtor]           destructor
[class.abstract]       pure virtual / abstract class
[expr.dynamic.cast]    dynamic_cast
[expr.typeid]          typeid
```

学习时始终保持这条主线：

```text
对象模型
   ↓
静态类型 / 动态类型
   ↓
name lookup
   ↓
overload resolution
   ↓
selected function interface
   ↓
virtual ?
   ├── no  → 直接调用
   └── yes → dynamic type → final overrider
```

只要这条链条稳定，多态、重载、名字隐藏、默认参数和虚派发就不再是互相冲突的一堆规则，而是同一个调用模型中的不同阶段。
