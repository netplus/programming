# C++ 对象生命周期与特殊成员函数：从设计哲学到规则体系

> 语义基线：**C++17**  
> 本文不是刷题笔记，而是一篇用于建立长期心智模型的专题学习文档。  
> 选择题只应作为最后的验收手段：如果不能解释“为什么语言要这么设计”，就不算真正掌握规则。

---

# 0. 先说结论：这一章真正要学的是什么

很多 C++ 学习资料会把下面这些内容拆成若干条需要背诵的规则：

- 什么时候有默认构造函数；
- 什么时候有拷贝构造函数；
- 什么时候有移动构造函数；
- `= default` 是什么；
- `= delete` 是什么；
- 为什么写了析构函数就可能没有隐式 move；
- Rule of Three / Five / Zero 是什么。

这样学很容易碎片化。

更好的方式是先理解 C++ 的核心问题：

> **一个对象不仅是一块内存，它还可能代表某种“资源所有权”和生命周期责任。**

例如：

```cpp
class File {
    FILE* fp;
};
```

一个 `File` 对象背后可能代表：

```text
一个内存对象
+
一个操作系统文件句柄
+
“谁负责关闭这个文件”的所有权关系
```

于是当对象：

```text
被创建
被复制
被移动
被赋值
被销毁
```

语言必须回答：

```text
资源应该怎么办？
```

C++ 的特殊成员函数，本质上就是围绕这个问题建立的一套**对象生命周期协议**。

本章要建立的是下面这张总图：

```text
                一个对象 T
                    │
        ┌───────────┼───────────┐
        │           │           │
      创建         转移/复制      销毁
        │           │           │
  default ctor   copy / move    destructor
                    │
          ┌─────────┴─────────┐
          │                   │
      construction        assignment
```

也就是说，所谓“特殊成员函数”并不是几条孤立语法，而是 C++ 对：

> **对象如何诞生、如何拥有状态、如何复制/转移状态、如何结束生命周期**

这一整套语义的编码方式。

---

# 1. 为什么 C++ 要让编译器自动生成这些函数

先从最简单的类开始：

```cpp
struct Point {
    int x;
    int y;
};
```

如果程序员每定义一个类，都必须手写：

```cpp
Point();
Point(const Point&);
Point(Point&&);
Point& operator=(const Point&);
Point& operator=(Point&&);
~Point();
```

那会非常繁琐。

而对 `Point` 这种普通值类型，正确语义其实很机械：

```text
构造 Point
→ 构造 x、y

拷贝 Point
→ 拷贝 x、y

移动 Point
→ 移动 x、y（对 int 与 copy 几乎等价）

销毁 Point
→ 销毁 x、y
```

因此 C++ 采用一个重要设计：

> **如果程序员没有表达特殊意图，编译器可以按照成员和基类的语义，自动合成对象生命周期操作。**

可以把它理解成：

```text
对象 T 的默认生命周期语义
=
对每一个 base/member 分别执行对应的生命周期操作
```

例如：

```cpp
struct A {
    std::string s;
    std::vector<int> v;
};
```

默认拷贝 `A`，本质上类似：

```text
copy s
copy v
```

默认移动 `A`，类似：

```text
move s
move v
```

默认析构 `A`，类似：

```text
destroy v
destroy s
```

这就是默认特殊成员函数最根本的思想：

> **组合（composition）的生命周期语义可以由成员自己的生命周期语义递归构成。**

这也是为什么现代 C++ 非常鼓励：

```text
让资源由成熟的 RAII 类型拥有
而业务类型尽量不要自己手写资源管理代码
```

这会自然导向后面的 **Rule of Zero**。

---

# 2. 六类最重要的特殊成员函数

为了形成统一模型，先把它们放在一起看。

```cpp
class T {
public:
    T();                         // 1. 默认构造

    T(const T&);                 // 2. 拷贝构造
    T(T&&);                      // 3. 移动构造

    T& operator=(const T&);      // 4. 拷贝赋值
    T& operator=(T&&);           // 5. 移动赋值

    ~T();                        // 6. 析构
};
```

它们分别解决：

| 操作 | 问题 |
|---|---|
| `T()` | 一个全新的对象怎样开始存在？ |
| `T(const T&)` | 用已有对象复制出一个新对象时怎么办？ |
| `T(T&&)` | 用一个即将不再需要的对象构造新对象时，资源能否转移？ |
| `operator=(const T&)` | 两个对象都已存在，把一个状态复制给另一个怎么办？ |
| `operator=(T&&)` | 两个对象都已存在，把资源转移给另一个怎么办？ |
| `~T()` | 对象生命周期结束时如何释放它负责的资源？ |

这里首先要理解：

> **construction 和 assignment 不是一回事。**

例如：

```cpp
T a;
T b = a;
```

`b` 原本不存在，所以是：

```text
copy construction
```

而：

```cpp
T a;
T b;
b = a;
```

`b` 已经存在，因此是：

```text
copy assignment
```

这一区别对资源管理尤其重要。

赋值操作通常要处理：

```text
目标对象原来已经持有的资源怎么办？
```

构造则不存在这个问题。

---

# 3. 学规则之前必须统一的术语

这是整章最重要的基础之一。

很多“看起来互相矛盾”的 C++ 规则，其实只是因为混淆了不同术语。

---

## 3.1 user-declared：用户声明过

例如：

```cpp
class A {
public:
    ~A();
};
```

显然析构函数是用户声明的。

但是：

```cpp
class A {
public:
    ~A() = default;
};
```

它同样是：

```text
user-declared destructor
```

因为程序员明确写出了这个声明。

所以：

```text
= default
≠
“我什么都没有写”
```

这是后面理解 move 生成规则的关键。

---

## 3.2 implicitly-declared：编译器替你声明

例如：

```cpp
struct A {
    int x;
};
```

程序员没有写：

```cpp
A(const A&);
```

但只要规则允许，编译器会隐式声明相应的 copy constructor。

这叫：

```text
implicitly-declared
```

注意：

```text
implicitly-declared
```

只是说明：

```text
“这个函数进入了类的接口模型”
```

并不自动意味着：

```text
“最终一定可调用”
```

后面还可能被定义为：

```text
deleted
```

---

## 3.3 defaulted：要求编译器提供默认语义

例如：

```cpp
A() = default;
A(const A&) = default;
A(A&&) = default;
~A() = default;
```

含义不是“空函数”。

而是：

> **请按照语言规定的默认成员/基类语义生成这个函数。**

例如：

```cpp
A(const A&) = default;
```

意味着：

```text
按照默认 copy 规则复制 base/member
```

而不是：

```cpp
A(const A&) {
    // 什么都不做
}
```

这两个完全不同。

---

## 3.4 deleted：这个函数不能被使用

例如：

```cpp
A(const A&) = delete;
```

它的意思不是：

```text
“这个函数不存在”
```

而是：

```text
这个函数被声明了
但程序不允许调用它
```

这是一个非常有意识的 API 设计工具。

例如：

```cpp
class Mutex {
public:
    Mutex() = default;
    Mutex(const Mutex&) = delete;
    Mutex& operator=(const Mutex&) = delete;
};
```

这里是在告诉使用者：

> **mutex 代表一个不可复制的同步实体，复制这个概念本身没有合理语义。**

所以 `= delete` 本质上是：

> **在类型系统层面禁止某种对象操作。**

---

## 3.5 user-provided：用户真正提供了实现

这是比 user-declared 更细的一层概念。

例如在类内第一次声明时直接：

```cpp
A() = default;
```

它是：

```text
user-declared
```

但通常不算：

```text
user-provided
```

而：

```cpp
class A {
public:
    A();
};

A::A() = default;
```

由于第一次声明时并没有 default，之后才在类外 default，规则上会落入不同的 user-provided 分类。

这个差别会影响 triviality 等更细的语言属性。

当前阶段不需要死背所有后果，但要知道：

```text
user-declared
和
user-provided
不是同义词
```

后续遇到 trivial / constexpr / aggregate 等专题时再深入。

---

# 4. `= default` 的设计哲学

为什么 C++ 要提供：

```cpp
= default
```

而不是让程序员“什么都不写”？

原因是：

> **“我要这个函数存在”与“请按默认方式实现它”是两个不同的设计意图。**

看一个最典型的例子：

```cpp
class A {
public:
    A(int x);
};
```

一旦程序员声明了构造函数：

```cpp
A(int)
```

编译器不会再自动提供普通无参默认构造：

```cpp
A()
```

但是程序员可能明确希望：

```text
既允许 A()
又允许 A(int)
```

于是可以写：

```cpp
class A {
public:
    A() = default;
    A(int x);
};
```

这句话表达的是：

```text
我明确要求 A() 是这个类型 API 的一部分
但它不需要自定义行为
请使用语言的默认构造语义
```

因此 `= default` 的核心价值是：

> **显式表达“我需要默认语义”。**

这比“依赖编译器碰巧帮我生成”更明确。

---

# 5. `= delete` 的设计哲学

`= delete` 解决的是另一个问题：

> **有些操作从语法上可以描述，但从类型语义上根本不应该存在。**

例如一个表示独占文件句柄的类：

```cpp
class File {
    int fd;
};
```

如果直接使用默认 copy：

```cpp
File a;
File b = a;
```

可能得到：

```text
a.fd == 3
b.fd == 3
```

于是两个对象都认为自己拥有同一个资源。

最后：

```text
~a → close(3)
~b → close(3) again
```

就会产生重复释放问题。

这时候一个设计选择是禁止 copy：

```cpp
class File {
public:
    File(const File&) = delete;
    File& operator=(const File&) = delete;
};
```

于是错误会在编译期暴露。

这体现了 C++ 类型设计中的一个核心思想：

> **如果一个操作没有正确语义，最好让它在类型系统中不可表达，而不是等到运行时出错。**

---

# 6. 默认构造函数：对象如何“从无到有”

默认构造函数通常指可无实参调用的构造路径，例如：

```cpp
A();
```

最重要的一条规则是：

> **如果类没有用户声明任何构造函数，编译器才会考虑隐式声明默认构造函数。**

例如：

```cpp
class A {
public:
    int x;
};
```

可以：

```cpp
A a;
```

而：

```cpp
class A {
public:
    A(int x);
};
```

此时：

```cpp
A a;
```

通常编译失败。

为什么？

因为程序员已经介入了“对象应该怎样被创建”的设计。

语言选择：

```text
既然你已经定义了构造协议
编译器就不再擅自增加一个额外的无参构造入口
```

这很符合接口设计原则。

如果确实需要无参构造，就明确声明：

```cpp
A() = default;
```

---

## 6.1 default constructor 存在不等于可用

例如：

```cpp
class M {
public:
    M(int);
};

class A {
    M m;
};
```

如果编译器试图生成：

```cpp
A::A()
```

它必须先默认构造：

```cpp
m
```

但 `M` 没有可用的无参构造函数。

因此 `A` 的默认构造能力会受成员约束。

这就是一个贯穿所有特殊成员函数的原则：

> **外层类的默认生命周期能力，建立在 base/member 对应能力之上。**

后面 copy、move、destructor 都是同样逻辑。

---

# 7. 析构函数：为什么它是资源管理的核心

析构函数：

```cpp
~A();
```

解决的问题是：

> **当对象生命周期结束时，它负责的资源应该怎样被释放？**

如果类型只包含自动管理资源的成员：

```cpp
class User {
    std::string name;
    std::vector<int> values;
};
```

通常根本不应该手写析构函数。

因为：

```text
~User
→ ~values
→ ~name
```

成员会自己递归释放资源。

这就是 RAII 的力量。

---

## 7.1 为什么手写析构往往意味着“这个类在直接管理资源”

历史上看到：

```cpp
class Buffer {
    char* p;
public:
    ~Buffer() {
        delete[] p;
    }
};
```

通常意味着：

```text
这个类直接承担资源所有权
```

一旦你拥有资源，就必须立刻追问：

```text
copy 应该怎么办？
move 应该怎么办？
assignment 应该怎么办？
```

这正是 Rule of Three / Five 的来源。

语言因此对：

```text
user-declared destructor
```

和隐式 move 的关系采取谨慎策略。

如果程序员专门写了析构函数，语言会推测：

> **这个类型可能有非平凡资源管理语义，自动“偷走资源”的 move 不应该轻率生成。**

所以一个 user-declared destructor 会抑制普通隐式 move constructor / move assignment 的声明。

这不是随意的语法陷阱，而是保守的资源安全设计。

---

# 8. 拷贝构造：复制“对象值”究竟意味着什么

典型 copy constructor：

```cpp
A(const A&);
```

语义是：

```text
旧对象仍然存在
+
新对象得到一份逻辑上等价的状态
```

对于纯值对象：

```cpp
struct Point {
    int x;
    int y;
};
```

默认 memberwise copy 完全合理。

但对拥有裸资源的类：

```cpp
class Buffer {
    char* p;
};
```

默认 copy 只是复制指针值：

```text
b.p = a.p
```

而不是复制指针指向的资源。

这就导致两个对象共享同一个原始所有权指针。

如果两个析构函数都：

```cpp
delete[] p;
```

就会 double free。

因此“默认 copy 是否正确”不是语法问题，而是类型语义问题：

> **这个类型究竟是 value semantics、shared ownership，还是 unique ownership？**

不同所有权模型决定 copy 的设计。

---

# 9. 移动构造：为什么 C++11 要引入 move

移动构造：

```cpp
A(A&&);
```

它的核心设计目标不是“比 copy 更高级”，而是：

> **当源对象即将被丢弃时，允许把昂贵资源直接转移给新对象，而不是复制资源。**

例如：

```cpp
std::vector<int> a(1'000'000);
std::vector<int> b = std::move(a);
```

理想 move 可以只交换/接管内部指针：

```text
b 接管 a 的 buffer

a 进入一个合法但内容未指定的状态
```

而不需要复制一百万个元素。

因此 move 的设计哲学是：

```text
copy：两个对象最终都保留各自有效逻辑值
move：目标对象取得源对象资源，源对象仍有效但其旧值不再保证
```

注意：

> **move 并不意味着销毁源对象。**

源对象仍然必须保持：

```text
valid but unspecified state
```

并且最终还会正常析构。

---

# 10. `std::move` 为什么“不移动”

`std::move(x)` 的名字很容易误导初学者。

它本质上只是一个类型/值类别转换工具，可近似理解为：

```cpp
static_cast<T&&>(x)
```

它表达的是：

> **我允许后续重载决议把这个对象当作可被移动来源来处理。**

真正是否移动，要看最终选中的函数。

例如：

```cpp
class A {
public:
    A(const A&);
};

A a;
A b = std::move(a);
```

这里没有可用的 `A(A&&)`，于是：

```cpp
A(const A&)
```

仍然可以绑定右值。

最终发生 copy。

所以正确模型是：

```text
std::move(x)
    ↓
把表达式变成 xvalue
    ↓
进行正常 overload resolution
    ↓
最终可能 move、copy，甚至编译失败
```

---

# 11. 为什么“有名字的 T&&”仍然是左值表达式

考虑：

```cpp
void f(A&& x) {
    use(x);
}
```

`x` 的声明类型是：

```cpp
A&&
```

但表达式：

```cpp
x
```

是 lvalue。

为什么设计成这样？

因为一旦一个对象有了名字：

```text
你就可以在当前作用域中重复引用它
```

因此它不是“马上就要消失的一次性匿名值”。

如果函数内部仍然希望转移资源，程序员必须再次明确表达：

```cpp
std::move(x)
```

这是一种很重要的安全机制：

> **进入函数以后，不会因为参数最初来自右值，就自动在每次使用时不断偷走资源。**

---

# 12. 移动构造函数什么时候会被隐式声明

这是最容易被背成口诀的一条规则，但理解设计动机后就简单很多。

对于类 `T`，编译器只有在没有以下 user-declared 特殊成员时，才会普通地隐式声明 move constructor：

```text
1. copy constructor
2. move constructor
3. copy assignment operator
4. move assignment operator
5. destructor
```

可以把它们分成两组理解：

```text
复制/移动策略被程序员介入
    copy ctor
    move ctor
    copy assignment
    move assignment

资源释放策略被程序员介入
    destructor
```

只要程序员已经介入了这些资源语义，编译器就不再轻率推断：

```text
“按 memberwise move 一定是正确的”
```

这就是规则背后的统一逻辑。

注意：

```cpp
T() = default;
```

默认构造函数不在上面五项中，因此它本身不会抑制 move constructor 的隐式声明。

---

# 13. `~T() = default` 为什么仍然抑制隐式 move

例如：

```cpp
class T {
public:
    ~T() = default;
};
```

很多人会想：

```text
析构还是编译器生成的默认实现
为什么要影响 move？
```

关键是语言规则观察的是：

```text
user-declared destructor
```

而不是：

```text
析构函数内部是不是手写 delete
```

从接口角度看，程序员已经明确介入析构协议。

因此语言采用保守规则：

```text
不要继续自动推导 move
```

这条规则的历史兼容性和语言演进背景也很复杂，但从使用者角度最重要的是形成这个模型：

> **`= default` 可以要求默认实现，但不会抹去“程序员已经声明过这个函数”的事实。**

---

# 14. “没有 move”与“move 被 delete”是两件不同的事

这是非常关键的一层。

---

## 14.1 情况 A：move constructor 根本没有进入候选集合

例如：

```cpp
class A {
public:
    A(const A&);
};
```

用户声明了 copy constructor，因此不会普通地隐式声明 move constructor。

调用：

```cpp
A a;
A b = std::move(a);
```

重载候选里没有：

```cpp
A(A&&)
```

但有：

```cpp
A(const A&)
```

它能绑定右值，于是 copy 可以发生。

---

## 14.2 情况 B：你明确写了一个 deleted move

```cpp
class A {
public:
    A(const A&);
    A(A&&) = delete;
};
```

现在：

```cpp
A(A&&)
```

明确存在于重载集合里，而且对 xvalue 是比 `const A&` 更好的匹配。

因此：

```cpp
A b = std::move(a);
```

会先选：

```cpp
A(A&&)
```

然后发现：

```text
这个最佳候选被 delete
```

于是程序 ill-formed，编译失败。

编译器不会说：

```text
“第一名不能用，那我自动换第二名 copy”
```

这是 `= delete` 作为 API 禁令必须具有的语义。

---

# 15. 还要再区分：默认生成的 move 因成员原因被 deleted

这里比上一节更细。

考虑：

```cpp
class M {
public:
    M(const M&) = default;
    M(M&&) = delete;
};

class A {
public:
    M m;
};
```

`A` 自己没有手写 move/copy/destructor，所以编译器会按照规则考虑 `A(A&&)`。

但默认 move 要做的事情类似：

```text
move-construct m
```

而 `M(M&&)` 不可用。

于是 `A` 的默认 move 能力会受到限制，并可能成为 deleted。

这里要理解一个重要原则：

> **默认特殊成员函数不是凭空拥有能力，它只是把操作递归下发到 base/member。**

所以：

```text
成员不能 move
→ 外层默认 move 也可能不能成立
```

同理：

```text
成员不能 copy
→ 外层默认 copy 也可能不能成立
```

```text
成员不能 default construct
→ 外层默认构造也可能不能成立
```

这是非常统一的一套规则，而不是三套孤立规则。

> 细节提醒：隐式声明/在首次声明处 default 的 move constructor 如果由于这些默认生成条件而成为 deleted，在重载决议中有特殊处理，不能简单等同于程序员明确写出的 `T(T&&) = delete`。这部分在后续“重载决议 + special member”专题再进一步展开；当前阶段先掌握两者不是同一种来源。

---

# 16. 拷贝与移动之间为什么会互相影响

一个类型如果自己声明：

```cpp
T(T&&);
```

这通常是在说：

```text
我为资源转移设计了特殊语义
```

那么语言就不能继续假定默认 copy 一定合理。

类似地，如果用户声明：

```cpp
T& operator=(T&&);
```

也说明程序员已经介入资源所有权语义。

所以在 C++11 以后，一些隐式 copy 操作会因为用户声明 move 操作而被定义为 deleted。

这背后的思路仍然是：

> **一旦程序员声明了非常规资源转移策略，编译器宁可保守禁止某些自动操作，也不要默默制造潜在所有权 bug。**

因此不要把这些规则背成：

```text
写了 X → Y deleted
```

而要理解为：

```text
程序员介入了 ownership semantics
→ 编译器减少自动猜测
```

---

# 17. Copy Assignment 为什么比 Copy Construction 多一个问题

比较：

```cpp
T b = a;
```

与：

```cpp
b = a;
```

第一种：

```text
b 尚不存在
```

第二种：

```text
b 已经是一个完整对象，并可能持有资源
```

因此 copy assignment 必须先考虑：

```text
b 原来的资源怎么办？
```

例如：

```cpp
class Buffer {
    char* p;
public:
    Buffer& operator=(const Buffer& other) {
        // 不能只是 p = other.p;
    }
};
```

一个正确实现通常至少要考虑：

```text
1. 释放/替换旧资源
2. 复制新资源
3. 自赋值
4. 异常安全
```

这就是为什么手写资源管理代码很容易出错。

现代 C++ 更鼓励：

```cpp
class Buffer {
    std::vector<char> data;
};
```

然后完全使用默认特殊成员函数。

---

# 18. Move Assignment 还必须考虑“目标已有资源”

移动赋值：

```cpp
b = std::move(a);
```

和移动构造：

```cpp
T b = std::move(a);
```

也不同。

移动构造时：

```text
b 还没有资源
```

移动赋值时：

```text
b 已经可能拥有资源
```

因此典型 move assignment 要做：

```text
1. 清理/替换 b 原来的资源
2. 接管 a 的资源
3. 把 a 留在可析构的有效状态
```

这再次说明：

> constructor 与 assignment 必须分开理解。

---

# 19. 析构函数与继承：为什么多态基类通常需要 virtual destructor

考虑：

```cpp
class Base {
public:
    ~Base();
};

class Derived : public Base {
public:
    ~Derived();
};

Base* p = new Derived;
delete p;
```

如果 `Base::~Base()` 不是 virtual，那么通过基类指针删除派生对象会产生未定义行为。

为什么这和对象生命周期模型一致？

因为一个完整 `Derived` 对象的析构链应该是：

```text
Derived::~Derived()
↓
成员析构
↓
Base::~Base()
```

如果只凭 `Base*` 静态类型终止生命周期，语言无法正确执行最派生对象的完整析构协议。

因此当一个基类被设计成：

```text
允许通过 Base* 拥有/删除派生对象
```

它通常应该：

```cpp
virtual ~Base() = default;
```

这里的 `= default` 很有表达力：

```text
我需要一个 virtual destructor 作为多态接口契约
但没有额外析构工作
```

这正是 `= default` 的设计价值之一。

---

# 20. Rule of Three：为什么它会出现

C++11 之前最经典的经验法则是 Rule of Three：

如果一个类需要自定义下面任意一个：

```text
1. destructor
2. copy constructor
3. copy assignment
```

那么通常应该认真检查另外两个是否也需要自定义。

为什么？

因为这三者往往共同围绕：

```text
同一份资源所有权
```

例如：

```cpp
class Buffer {
    char* p;
};
```

如果需要：

```cpp
~Buffer() { delete[] p; }
```

就说明 `p` 是 owning pointer。

那么默认 copy constructor：

```text
只复制 p 地址
```

就很可能错误。

默认 copy assignment 同理。

所以 Rule of Three 不是语言强制语法，而是一条设计推论：

> **如果析构语义不是默认的，复制语义往往也不会天然正确。**

---

# 21. Rule of Five：move 出现后为什么从 Three 变成 Five

C++11 引入：

```text
move constructor
move assignment
```

于是资源管理协议从三件事扩展到五件事：

```text
destructor
copy constructor
copy assignment
move constructor
move assignment
```

如果一个类直接管理资源并需要自定义其中一部分，通常要系统审视全部五个操作。

这就是 Rule of Five。

但注意：

> Rule of Five 并不是鼓励“每个类都写五个函数”。

恰恰相反，它是在警告：

```text
一旦你开始手工管理资源
你就承担了大量生命周期正确性责任
```

所以现代 C++ 更进一步发展成 Rule of Zero。

---

# 22. Rule of Zero：现代 C++ 真正希望你达到的状态

Rule of Zero 的思想是：

> **业务类尽量不要直接管理裸资源，让专门的 RAII 成员类型承担资源生命周期。**

例如不要写：

```cpp
class Buffer {
    char* p;
    size_t n;

public:
    ~Buffer();
    Buffer(const Buffer&);
    Buffer& operator=(const Buffer&);
    Buffer(Buffer&&);
    Buffer& operator=(Buffer&&);
};
```

而尽量写：

```cpp
class Buffer {
    std::vector<char> data;
};
```

这时：

```text
vector 已经正确实现：
copy
move
destroy
assignment
```

所以 `Buffer` 可以依赖编译器默认组合：

```text
Rule of Zero
```

这其实是 C++ 对象模型设计最值得掌握的结论之一：

> **最好的特殊成员函数代码，往往是根本不需要写特殊成员函数。**

---

# 23. RAII：构造/析构规则背后的统一哲学

RAII：

```text
Resource Acquisition Is Initialization
```

不要只把它翻译成“资源获取即初始化”。

它真正表达的是：

> **把资源所有权绑定到对象生命周期。**

例如：

```cpp
{
    std::unique_ptr<int> p = std::make_unique<int>(42);
}
```

进入作用域：

```text
对象构造
→ 获得资源
```

离开作用域：

```text
对象析构
→ 自动释放资源
```

这样资源释放就不再依赖：

```text
程序员是否记得在每个 return/exception 分支手动 cleanup
```

而是依赖语言本身的对象生命周期规则。

所以构造/析构绝不是“考试语法”，而是 C++ 资源安全模型的核心。

---

# 24. `unique_ptr` 是理解特殊成员函数设计的最好例子之一

`std::unique_ptr<T>` 表示：

```text
独占所有权
```

既然是独占，就不能 copy：

```cpp
unique_ptr(const unique_ptr&) = delete;
```

但所有权可以转移，所以支持 move：

```cpp
unique_ptr(unique_ptr&&);
```

析构时自动释放资源：

```text
~unique_ptr → delete owned object
```

这三个行为不是偶然拼起来的，而是从“unique ownership”这个设计哲学自然推出的：

```text
独占所有权
    │
    ├─ copy：禁止
    ├─ move：允许，表示转移所有权
    └─ destroy：释放所有权对应资源
```

因此以后看到特殊成员函数，不要先问：

```text
标准规定怎么背？
```

先问：

```text
这个类型的 ownership model 是什么？
```

很多规则会自然变得可理解。

---

# 25. `shared_ptr` 又体现了另一种 copy 语义

`shared_ptr` 表示：

```text
共享所有权
```

因此 copy 是合理的：

```text
copy shared_ptr
→ 增加引用计数
```

move 也合理：

```text
move shared_ptr
→ 转移当前句柄，不需要增加引用计数
```

析构则：

```text
减少引用计数
→ 最后一个 owner 消失时销毁资源
```

所以：

```text
copy/move/destruct 的正确语义
```

取决于：

```text
类型所表达的所有权模型
```

这正是 C++ 特殊成员函数存在的根本原因。

---

# 26. C++17 copy elision：为什么有时 copy/move 根本不发生

理解 special member functions 后，还要避免另一种机械思维：

```text
看到返回对象
→ 一定 copy 或 move
```

例如：

```cpp
T make() {
    return T{};
}

T x = make();
```

在 C++17 对这类 prvalue 场景，语言语义允许/要求直接在最终目标位置构造对象，而不是先制造一个独立临时对象再 move。

因此应该理解成：

```text
直接构造最终 T
```

而不是：

```text
T{}
↓ move
返回临时对象
↓ move
x
```

这也是现代 C++ 值类别模型演进的一部分。

---

## 26.1 Guaranteed Copy Elision 与 NRVO 要分开

这两个非常容易混淆。

### prvalue 场景

```cpp
T make() {
    return T{};
}
```

C++17 下属于非常强的直接构造语义。

### named local 场景

```cpp
T make() {
    T x;
    return x;
}
```

这里是 NRVO：

```text
Named Return Value Optimization
```

NRVO 不是同一套“强制”规则。

所以不能把：

```cpp
return T{};
```

和：

```cpp
return x;
```

当成完全一样的语言模型。

---

# 27. 一个统一的“能力传播模型”

现在把前面所有规则压缩成一个非常重要的模型。

假设：

```cpp
class T : public B {
    M1 m1;
    M2 m2;
};
```

编译器要生成某个默认特殊成员函数，比如：

```cpp
T(T&&)
```

本质上要问：

```text
B 能不能 move？
M1 能不能 move？
M2 能不能 move？
```

如果所有子对象都能完成相应操作，外层默认操作才有机会成立。

因此可以形成统一表：

```text
T 的 default construction
    ← bases/members 能否 default construct

T 的 copy construction
    ← bases/members 能否 copy construct

T 的 move construction
    ← bases/members 能否 move construct

T 的 copy assignment
    ← bases/members 能否 copy assign

T 的 move assignment
    ← bases/members 能否 move assign

T 的 destruction
    ← bases/members 能否正确 destruct
```

这就是比死背规则更有价值的“递归对象模型”。

---

# 28. 为什么编译器宁可 delete，也不替你“猜一个差不多的实现”

C++ 的哲学之一是：

> **隐式操作只能建立在明确、局部、可组合的规则上。**

如果成员：

```cpp
M(M&&) = delete;
```

语言不会替外层类发明一种“也许 copy M 就算 move T”的任意语义。

因为这可能违反类型作者的意图。

所以特殊成员函数经常表现为：

```text
条件不满足
→ deleted
```

这其实是一种保守性：

```text
不知道正确语义
就要求程序员明确表达
```

而不是默默猜测。

---

# 29. 手写 destructor 为什么常常是一个“设计信号”

以后阅读代码时，如果看到：

```cpp
~T() {
    ...
}
```

应该马上产生一个工程问题：

```text
为什么这个类需要自己析构？
```

可能原因包括：

```text
1. 直接拥有裸资源
2. 需要特殊 teardown 协议
3. 基类需要 virtual destructor
4. 需要日志/统计等副作用
5. 历史代码没有采用 RAII 成员
```

接下来应该系统检查：

```text
copy 语义正确吗？
move 语义正确吗？
assignment 正确吗？
```

这比单纯记住：

```text
“析构会抑制 move”
```

更有工程价值。

---

# 30. `virtual ~Base() = default` 为什么是一个非常典型的好设计

```cpp
class Base {
public:
    virtual ~Base() = default;
};
```

这句话同时表达两层意图：

```text
virtual
→ 这个类支持通过基类接口正确结束派生对象生命周期

= default
→ Base 自己没有额外资源释放逻辑，使用默认析构即可
```

这就是现代 C++ 中 `= default` 最漂亮的用途之一：

> **接口语义需要这个函数，但实现行为不需要自定义。**

不过也正因为它是 user-declared destructor：

```text
隐式 move 规则会受到影响
```

因此如果这个类本身还需要 move 语义，就要进一步显式设计。

这不是矛盾，而是语言要求程序员在多种语义发生交叉时明确选择。

---

# 31. `= delete` 不只可以用于 copy constructor

虽然最常见：

```cpp
T(const T&) = delete;
```

但 `= delete` 是更一般的语言工具。

例如禁止某个重载：

```cpp
void f(int) = delete;
void f(double);
```

调用：

```cpp
f(10);
```

不会自动退而选择 `f(double)`，因为最佳匹配 `f(int)` 被明确禁止。

这说明 `= delete` 的设计不是“缺少实现”，而是：

> **明确告诉 overload resolution：这个接口存在，但使用它是程序错误。**

这也帮助理解：

```cpp
T(T&&) = delete;
```

为什么和“根本没有 move constructor”语义不同。

---

# 32. 为什么 Rule of Zero 比背生成规则更重要

如果你大量依赖：

```cpp
std::string
std::vector
std::unique_ptr
std::shared_ptr
std::mutex
```

这些类型本身已经编码好了：

```text
copyable?
movable?
destructible?
ownership semantics?
```

你的外层类只要组合它们，编译器的默认特殊成员函数就会把这些语义传播上来。

例如：

```cpp
class Session {
    std::unique_ptr<Socket> socket;
};
```

因为 `unique_ptr` 不可 copy、可 move，所以 `Session` 的默认能力也会自然表现出类似所有权特征。

这比：

```cpp
Socket* socket;
```

然后手写五个函数更安全、更清晰。

这就是现代 C++ 对象设计真正希望你掌握的东西。

---

# 33. 一张总表：看到一个类时应该怎样分析

以后遇到任何特殊成员函数问题，不要直接套口诀。

按下面顺序分析。

## 第一步：这个类型表达什么语义？

```text
纯 value？
unique ownership？
shared ownership？
non-owning view？
同步原语？
多态基类？
```

---

## 第二步：成员和基类分别具有什么生命周期能力？

```text
default constructible?
copyable?
movable?
destructible?
```

---

## 第三步：程序员声明过哪些特殊成员函数？

重点检查：

```text
constructor
copy ctor
move ctor
copy assignment
move assignment
destructor
```

尤其区分：

```text
没声明
= default
= delete
自己提供函数体
```

---

## 第四步：编译器还会隐式声明什么？

例如 move constructor，要检查那五类 user-declared 特殊成员函数。

---

## 第五步：即使声明了，默认实现能成立吗？

递归检查：

```text
base/member 对应操作是否可用
```

---

## 第六步：调用处再进行 overload resolution

例如：

```cpp
T b = std::move(a);
```

不能直接说 move。

应该问：

```text
T(T&&) 是否存在？
是否可用？
是否 deleted？
T(const T&) 是否能绑定？
哪个是最佳匹配？
```

---

# 34. 一张总表：默认生成规则的学习版心智模型

下面不是标准条文逐字抄写，而是用于建立稳定思维的学习表。

| 操作 | 首先问什么 |
|---|---|
| 默认构造 | 是否已经有 user-declared constructor？成员/base 能否默认构造？ |
| 拷贝构造 | 是否有 user-declared copy ctor？是否被 move 相关声明影响？成员/base 能否 copy？ |
| 移动构造 | 是否有 user-declared copy/move ctor、copy/move assignment、destructor？成员/base 能否 move？ |
| 拷贝赋值 | 是否有 user-declared copy assignment？是否被 move 声明影响？成员/base 能否 copy assign？ |
| 移动赋值 | 是否有 user-declared copy/move ctor、copy/move assignment、destructor？成员/base 能否 move assign？ |
| 析构 | 是否有 user-declared destructor？成员/base 是否可析构？多态删除是否需要 virtual？ |

不要把这张表作为终点。

它只是前面设计模型的速查图。

---

# 35. 最终应该形成的设计观

学完这一专题，希望最终形成以下判断方式。

### 观点 1：特殊成员函数首先是“类型语义”，不是语法技巧

```text
copy / move / destroy
```

都是在表达：

```text
这个类型对状态和资源拥有怎样的所有权关系
```

---

### 观点 2：编译器默认生成不是魔法，而是递归组合

```text
T 的默认行为
=
base/member 的默认行为组合
```

---

### 观点 3：程序员一旦介入资源语义，编译器就会变得更保守

这解释了：

```text
为什么某些 user-declared special member 会抑制其它隐式操作
```

---

### 观点 4：`= default` 是“显式选择默认语义”

不是：

```text
什么都没写
```

---

### 观点 5：`= delete` 是类型/API 设计工具

不是：

```text
没有实现
```

而是：

```text
这个操作在本类型语义中被明确禁止
```

---

### 观点 6：move 是所有权转移机会，不是 `std::move` 这个函数本身执行移动

```text
std::move
→ 值类别转换
→ overload resolution
→ 最终才决定 copy / move / error
```

---

### 观点 7：现代 C++ 的目标通常不是 Rule of Five，而是 Rule of Zero

真正好的业务类型往往通过：

```text
RAII member types
```

把复杂生命周期管理下沉给：

```text
vector / string / unique_ptr / shared_ptr / lock 等成熟组件
```

从而让自己的特殊成员函数保持默认状态。

---

# 36. 后续学习顺序

在继续做题之前，建议按下面顺序消化本专题：

```text
第一层：对象生命周期
    constructor / destructor

第二层：值语义
    copy construction / copy assignment

第三层：资源转移
    rvalue / move construction / move assignment

第四层：语言控制工具
    = default / = delete

第五层：自动生成机制
    implicit declaration / deleted default operations

第六层：设计原则
    RAII
    Rule of Three
    Rule of Five
    Rule of Zero

第七层：组合场景
    inheritance
    virtual destructor
    members that are non-copyable/non-movable
    copy elision
```

只有在这几层关系能够自己解释之后，再进入下一轮题目。

后续题目的作用不是训练“看到题型选哪个字母”，而是验证：

> **能否从类型设计、对象生命周期、所有权语义和编译器生成规则四个层次，自洽地推导出答案。**
