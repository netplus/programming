# C++ 对象生命周期与特殊成员函数：从设计哲学到规则体系

> **语义基线：C++17**  
> 本文是一份连续学习教材，不以刷题或背规则为目标。选择题只能用于最后验证：如果不能从对象语义和语言规则推导出答案，就不算真正掌握。

---

## 目录

1. [为什么 C++ 需要特殊成员函数](#1-为什么-c-需要特殊成员函数)
2. [先建立对象生命周期总图](#2-先建立对象生命周期总图)
3. [构造函数应该怎样分类](#3-构造函数应该怎样分类)
4. [六类特殊成员函数分别解决什么问题](#4-六类特殊成员函数分别解决什么问题)
5. [construction 与 assignment 为什么必须分开](#5-construction-与-assignment-为什么必须分开)
6. [编译器为什么可以自动生成这些函数](#6-编译器为什么可以自动生成这些函数)
7. [理解规则前必须掌握的术语](#7-理解规则前必须掌握的术语)
8. [`= default` 的准确含义](#8--default-的准确含义)
9. [`= delete` 的准确含义](#9--delete-的准确含义)
10. [第一阶段：特殊成员函数的隐式声明与抑制关系](#10-第一阶段特殊成员函数的隐式声明与抑制关系)
11. [完整二维关系表](#11-完整二维关系表)
12. [逐行解释二维表为什么这样设计](#12-逐行解释二维表为什么这样设计)
13. [第二阶段：base/member 能力如何传播到外层类](#13-第二阶段basemember-能力如何传播到外层类)
14. [第三阶段：具体调用点还要经过重载决议](#14-第三阶段具体调用点还要经过重载决议)
15. [`std::move` 到底做了什么](#15-stdmove-到底做了什么)
16. [有名字的右值引用为什么仍是左值表达式](#16-有名字的右值引用为什么仍是左值表达式)
17. [move 不存在、move 被 delete、defaulted move 被 delete：三者必须区分](#17-move-不存在move-被-deletedefaulted-move-被-delete三者必须区分)
18. [析构函数与资源所有权](#18-析构函数与资源所有权)
19. [为什么“默认构造函数里 new 了内存”不会改变隐式 move 规则](#19-为什么默认构造函数里-new-了内存不会改变隐式-move-规则)
20. [Rule of Three / Five / Zero](#20-rule-of-three--five--zero)
21. [C++17 拷贝消除：不要机械地数 copy/move](#21-c17-拷贝消除不要机械地数-copymove)
22. [多态基类为什么通常需要 virtual destructor](#22-多态基类为什么通常需要-virtual-destructor)
23. [一套可重复使用的推导算法](#23-一套可重复使用的推导算法)
24. [常见误区与纠正](#24-常见误区与纠正)
25. [学习检查点](#25-学习检查点)
26. [参考资料](#26-参考资料)

---

# 1. 为什么 C++ 需要特殊成员函数

先不要从语法开始，而要从一个更基本的问题开始：

> **一个对象从创建到销毁，语言必须怎样维护它所代表的状态与资源？**

最简单的对象：

```cpp
struct Point {
    int x;
    int y;
};
```

`Point` 基本只是两个值的组合：

```text
Point
├── x
└── y
```

复制 `Point` 时，复制 `x/y` 就足够；销毁时也没有特殊资源需要释放。

但是下面这个类型不同：

```cpp
class File {
    int fd;
};
```

一个 `File` 对象可能不只是保存整数 `fd`，它还代表：

```text
一个 C++ 对象
+
一个操作系统文件描述符
+
“谁负责 close(fd)”这一生命周期责任
```

再例如：

```cpp
class Buffer {
    char* p;
};
```

`p` 可能只是一个借用指针，也可能代表对象独占的一块动态内存。

于是 C++ 类型需要回答一系列问题：

```text
对象怎样创建？
从另一个对象复制时怎么办？
从另一个对象转移资源时怎么办？
给已经存在的对象赋值时怎么办？
对象销毁时资源怎么办？
```

特殊成员函数就是 C++ 用来描述这套**对象生命周期协议（object lifetime protocol）**的语言机制。

---

# 2. 先建立对象生命周期总图

一个对象的主要生命周期操作可以画成：

```text
                       对象 T
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       创建           状态复制/转移         销毁
        │                │                │
  constructor       copy / move        destructor
                         │
                ┌────────┴────────┐
                │                 │
         construction        assignment
          创建新对象           目标已存在
```

这里已经包含了后面绝大多数规则的逻辑来源。

构造函数解决：

> **对象如何开始存在。**

拷贝/移动解决：

> **已有对象的值或资源如何复制、转移。**

析构解决：

> **对象生命周期结束时，它负责的状态/资源如何收尾。**

一个重要原则是：

> **C++ 不会分析函数体来猜测你的 ownership 语义；它主要根据类声明和成员/基类的类型能力来推导默认对象语义。**

这一点后面会反复使用。

---

# 3. 构造函数应该怎样分类

“无参构造”和“带参构造”这种分类太粗，会把 copy/move constructor 混进去。

更稳定的分类方式是：

```text
constructor
│
├── default constructor
│      A()
│      A(int x = 0)
│
├── ordinary constructor
│      A(int)
│      A(std::string)
│      A(int, double)
│
├── copy constructor
│      A(const A&)
│
└── move constructor
       A(A&&)
```

## 3.1 Default constructor

默认构造函数不是简单定义为“参数数量为 0”。

更准确地说，它是**能够在不提供实参的情况下调用的构造函数**。

例如：

```cpp
class A {
public:
    A();
};
```

显然是 default constructor。

下面这个同样可以是 default constructor：

```cpp
class A {
public:
    A(int x = 0);
};
```

因为：

```cpp
A a;
```

可以成立。

## 3.2 Ordinary constructor

例如：

```cpp
class A {
public:
    A(int);
};
```

它要求提供参数，所以不是 default constructor。

它的一个重要影响是：

> **一旦类存在任意 user-declared constructor，编译器就不会再额外隐式声明普通 default constructor。**

因此：

```cpp
class A {
public:
    A(int);
};

A a;       // error：没有 A()
A b(10);   // OK
```

但 `A(int)` 本身不会直接抑制 copy/move/assignment/destructor。

## 3.3 Copy/Move constructor 必须单独分类

虽然：

```cpp
A(const A&)
A(A&&)
```

从形式上也都是“带参数 constructor”，但它们属于特殊成员函数，会参与复杂的特殊成员函数生成规则。

所以不要记：

```text
“带参数 constructor 不影响 move”
```

而要记：

```text
ordinary constructor 不直接抑制 implicit move
copy/move constructor 则有专门规则
```

---

# 4. 六类特殊成员函数分别解决什么问题

C++17 中，本专题最重要的六类特殊成员函数是：

```cpp
class T {
public:
    T();                         // default constructor
    T(const T&);                 // copy constructor
    T(T&&);                      // move constructor

    T& operator=(const T&);      // copy assignment
    T& operator=(T&&);           // move assignment

    ~T();                        // destructor
};
```

| 函数 | 核心问题 |
|---|---|
| `T()` | 一个新对象怎样建立默认状态？ |
| `T(const T&)` | 如何由已有对象复制出一个新对象？ |
| `T(T&&)` | 如何由一个可被“消费”的对象构造新对象？ |
| `operator=(const T&)` | 目标对象已经存在时，如何复制状态？ |
| `operator=(T&&)` | 目标对象已经存在时，如何转移状态/资源？ |
| `~T()` | 生命周期结束时如何收尾？ |

注意：ordinary constructor，例如 `T(int)`，是 constructor，但不是这里六个 special member functions 之一。

---

# 5. construction 与 assignment 为什么必须分开

下面两段代码看起来都像“把 a 给 b”，但语义不同。

## 5.1 Copy construction

```cpp
T a;
T b = a;
```

这里 `b` 原先不存在。

因此调用的是：

```cpp
T(const T&)
```

核心任务：

```text
创建一个新对象 b
并让它拥有 a 的对应状态
```

## 5.2 Copy assignment

```cpp
T a;
T b;
b = a;
```

这里 `b` 已经存在。

因此调用：

```cpp
T& operator=(const T&)
```

它比 copy construction 多了一个问题：

> **b 原来已经拥有的状态/资源怎么办？**

例如一个管理动态内存的类型，assignment 可能需要：

```text
释放 b 的旧资源
↓
复制/获取 a 的新资源
↓
保证异常安全和自赋值行为
```

所以 construction 和 assignment 绝不能混为一个操作。

---

# 6. 编译器为什么可以自动生成这些函数

对下面这种普通值类型：

```cpp
struct Record {
    std::string name;
    std::vector<int> values;
};
```

正确的 copy/move/destruction 语义通常可以机械地由成员组合出来。

例如默认 copy 大致是：

```text
copy name
copy values
```

默认 move 大致是：

```text
move name
move values
```

默认析构大致是：

```text
destroy values
destroy name
```

因此 C++ 的设计是：

> **如果类声明没有表达冲突的特殊意图，编译器可以根据 base/member 的对应操作自动构造外层类的对象语义。**

这是一种组合式设计：

```text
T 的默认生命周期语义
=
Base1 的语义
+ Base2 的语义
+ Member1 的语义
+ Member2 的语义
+ ...
```

现代 C++ 推荐把 ownership 封装到 `std::string`、`std::vector`、`std::unique_ptr`、`std::shared_ptr` 等 RAII 类型里，就是为了让这种组合推导安全可靠。

---

# 7. 理解规则前必须掌握的术语

很多 C++ 特殊成员函数题目之所以混乱，不是因为规则本身无法理解，而是因为以下术语被混用了。

## 7.1 user-declared

程序员在类接口中明确声明了函数。

例如：

```cpp
class A {
public:
    ~A();
};
```

是 user-declared destructor。

下面同样是：

```cpp
class A {
public:
    ~A() = default;
};
```

因为 `= default` 不等于“没有声明”。

## 7.2 implicitly-declared

程序员没写对应声明，但规则要求编译器替类声明该函数。

例如：

```cpp
struct A {
    int x;
};
```

只要规则允许，编译器会隐式声明 copy constructor 等函数。

注意：

```text
implicitly-declared
≠
最终一定可调用
```

它还可能在下一阶段变成 deleted。

## 7.3 implicitly-defined

一个函数已经被隐式声明，并且在规则要求真正生成函数体时，编译器完成其定义。

因此要区分：

```text
声明阶段
↓
定义阶段
↓
调用阶段
```

## 7.4 defaulted

```cpp
A(const A&) = default;
```

表示：

> **这个函数由程序员明确要求存在，但实现采用语言规定的默认 memberwise/base-wise 语义。**

它不是空函数。

## 7.5 deleted

```cpp
A(const A&) = delete;
```

表示：

> **这个函数被声明了，但禁止使用。**

“deleted”与“根本不存在”对 overload resolution 的影响不同，这是后面的关键知识点。

## 7.6 user-provided

这是比 user-declared 更细的术语。

例如第一次声明时：

```cpp
A() = default;
```

是 user-declared，但通常不是 user-provided。

而先声明：

```cpp
class A {
public:
    A();
};
```

再在类外：

```cpp
A::A() = default;
```

会落入不同的 user-provided 分类。

这个区别会影响 triviality 等更深入属性。当前专题最重要的是先牢牢记住：

```text
user-declared
≠
user-provided
```

---

# 8. `= default` 的准确含义

`= default` 的设计价值是：

> **显式表达“我要这个函数作为类型接口的一部分，但它的实现采用默认语义”。**

例如：

```cpp
class A {
public:
    A(int);
};
```

由于已经存在 user-declared constructor，编译器不会再隐式生成 `A()`。

如果你确实希望保留 default construction，可以写：

```cpp
class A {
public:
    A() = default;
    A(int);
};
```

这里表达的是：

```text
A() 是 API 的一部分
+
它采用默认构造语义
```

同样：

```cpp
A(const A&) = default;
A(A&&) = default;
A& operator=(const A&) = default;
A& operator=(A&&) = default;
~A() = default;
```

都表示显式要求默认语义。

但是：

> **`= default` 仍然是 user-declared。**

因此：

```cpp
class A {
public:
    ~A() = default;
};
```

仍然存在 user-declared destructor，从而会影响 implicit move constructor / move assignment 的生成。

另一个重要结论：

> **`= default` 不保证函数最终可用。**

如果 base/member 无法完成对应操作，这个 defaulted function 仍可能被定义为 deleted。

---

# 9. `= delete` 的准确含义

`= delete` 表示一个函数存在于接口和候选集合中，但语言禁止调用它。

例如：

```cpp
class UniqueHandle {
public:
    UniqueHandle(const UniqueHandle&) = delete;
    UniqueHandle& operator=(const UniqueHandle&) = delete;
};
```

这里的设计意图是：

```text
UniqueHandle 表达独占所有权
↓
copy 没有合理语义
↓
从类型系统层面禁止 copy
```

这比“运行到某个地方再发现复制错误”更安全。

注意：`= delete` 不只可以用于特殊成员函数；它是更一般的函数接口控制机制。但本教材主要讨论它在 special member functions 中的用途。

---

# 10. 第一阶段：特殊成员函数的隐式声明与抑制关系

接下来进入最容易混乱的地方。

首先必须明确：所谓“某函数影响另一个函数”，可能是三种完全不同的机制。

```text
1. S：目标函数根本不再 implicitly declared

2. D：目标函数仍 implicitly declared
      但由于当前声明关系被 defined as deleted

3. P：目标 copy 操作仍可 implicit/defaulted
      但 C++17 已将这种依赖组合标为 deprecated
```

因此不能简单使用一个 `×` 表示“不能用”。

此外还需要：

```text
SELF：这个函数已经由用户自己声明
✓：当前声明对目标函数没有直接第一阶段影响
```

注意：

```text
✓ ≠ 最终一定可调用
```

它只意味着第一阶段没有被当前声明直接关闭。

---

# 11. 完整二维关系表

为了避免“带参数构造函数”造成歧义，这里额外加入 `OtherCtor`（普通非 copy/move constructor）。

| 已 user-declared ↓ / 目标 implicit → | DfltCtor | CopyCtor | MoveCtor | CopyAssign | MoveAssign | Dtor |
|---|---:|---:|---:|---:|---:|---:|
| **DfltCtor** | SELF | ✓ | ✓ | ✓ | ✓ | ✓ |
| **OtherCtor** | **S** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **CopyCtor** | **S** | SELF | **S** | **P** | **S** | ✓ |
| **MoveCtor** | **S** | **D** | SELF | **D** | **S** | ✓ |
| **CopyAssign** | ✓ | **P** | **S** | SELF | **S** | ✓ |
| **MoveAssign** | ✓ | **D** | **S** | **D** | SELF | ✓ |
| **Dtor** | ✓ | **P** | **S** | **P** | **S** | SELF |

符号：

| 符号 | 含义 |
|---|---|
| `SELF` | 该函数已经由用户声明 |
| `S` | Suppressed：目标函数根本不隐式声明 |
| `D` | 目标函数会隐式声明，但被定义为 deleted |
| `P` | C++17 中仍可默认生成，但属于 deprecated implicit-copy 组合 |
| `✓` | 当前这一项对目标没有直接第一阶段影响 |

这张表只解决：

> **根据 class declaration，编译器在第一阶段怎样建立 special member function 集合。**

它不是“类型最终支持哪些操作”的最终答案。

---

# 12. 逐行解释二维表为什么这样设计

## 12.1 DfltCtor 行

```cpp
class A {
public:
    A() = default;
};
```

对应：

```text
DfltCtor → SELF
其他五项 → ✓
```

原因是 default constructor 主要表达：

> **一个新对象怎样从无到有。**

它本身没有声明 copy/move/assignment/destruction 的特殊策略。

因此它不直接抑制这些操作。

这里必须避免一个错误直觉：

```text
“只要程序员写了 constructor，编译器就不再生成 move”
```

这是错的。

## 12.2 OtherCtor 行

```cpp
class A {
public:
    A(int);
};
```

`A(int)` 是 user-declared constructor，因此：

```text
implicit A() → S
```

但它不是 copy/move protocol 的组成部分，所以：

```text
CopyCtor   ✓
MoveCtor   ✓
CopyAssign ✓
MoveAssign ✓
Dtor       ✓
```

从设计上看，它仍主要描述“新对象如何创建”。

## 12.3 CopyCtor 行

```cpp
class A {
public:
    A(const A&);
};
```

首先，copy constructor 自己也是 constructor，所以：

```text
implicit default ctor → S
```

其次，C++ 不再隐式声明 move constructor / move assignment：

```text
MoveCtor   → S
MoveAssign → S
```

原因不是编译器分析到你“在管理资源”，而是：

> **类作者已经明确介入了对象之间的复制协议；语言不再自动假设 move 协议应该存在。**

CopyAssign 仍可默认生成，但 C++17 把依赖这种组合的 implicit copy 定义为 deprecated 方向，因此表中为 `P`。

## 12.4 MoveCtor 行

```cpp
class A {
public:
    A(A&&);
};
```

它也是 constructor，因此：

```text
DfltCtor → S
```

如果类显式声明 move constructor，则隐式 copy constructor 和隐式 copy assignment 会被定义为 deleted：

```text
CopyCtor   → D
CopyAssign → D
```

同时不会再额外隐式声明 move assignment：

```text
MoveAssign → S
```

这个组合体现了一个重要的语言设计：

> 当类型明确表达“可移动”但没有明确表达“可复制”时，语言不会擅自假设 copy 仍然正确。

这非常适合 unique ownership 类型。

## 12.5 CopyAssign 行

```cpp
class A {
public:
    A& operator=(const A&);
};
```

它不是 constructor，所以不影响 implicit default ctor。

但它表示类作者已经介入对象之间的复制/替换协议，因此：

```text
MoveCtor   → S
MoveAssign → S
```

CopyCtor 仍可能存在，但这种 implicit copy 组合在 C++17 中属于 deprecated 方向，所以为 `P`。

## 12.6 MoveAssign 行

```cpp
class A {
public:
    A& operator=(A&&);
};
```

它不是 constructor，因此：

```text
DfltCtor → ✓
```

但 move assignment 的声明会使：

```text
CopyCtor   → D
CopyAssign → D
MoveCtor   → S
```

也就是说，类型已经明确表达 move-side 语义时，语言不会默认保留 copy-side 语义。

## 12.7 Dtor 行

```cpp
class A {
public:
    ~A();
};
```

destructor 不属于 constructor，因此不影响 implicit default ctor。

但 user-declared destructor 会使：

```text
MoveCtor   → S
MoveAssign → S
```

而 copy constructor / copy assignment 在 C++17 中仍可存在，但这种“自己声明析构、继续依赖隐式 copy”的模式属于 deprecated 方向，因此表中为 `P`。

这也是 Rule of Three / Five 的历史背景之一。

---

# 13. 第二阶段：base/member 能力如何传播到外层类

二维表只回答“第一阶段是否声明”。

接下来编译器还要问：

> **如果我要 default/copy/move/assign/destruct 一个 T，它的 base/member 对应操作是否成立？**

例如：

```cpp
class M {
public:
    M(M&&) = delete;
};

class A {
    M m;
};
```

从 `A` 自身声明看，它没有 user-declared copy/move/destructor，所以第一阶段允许考虑 implicit move constructor。

但生成：

```cpp
A(A&&)
```

时，需要处理：

```cpp
m
```

这就要求找到可用的：

```cpp
M(M&&)
```

而它被 delete。

因此 `A` 的默认 move 也会受到影响。

这种传播可以统一理解为：

```text
T 能 default construct
  ← bases/members 能否完成所需 default initialization

T 能 copy construct
  ← bases/members 能否 copy construct

T 能 move construct
  ← bases/members 对 xvalue 初始化能否找到可用构造路径

T 能 copy assign
  ← bases/members 能否 copy assign

T 能 move assign
  ← bases/members 能否 move assign

T 能 destruct
  ← bases/members 是否可析构
```

## 13.1 `= default` 也必须经过这一层

```cpp
class A {
public:
    A(A&&) = default;
private:
    M m;
};
```

并不表示：

```text
“我写了 = default，所以 move 一定可用”
```

更准确是：

```text
我要求一个 defaulted move ctor
↓
编译器按 base/member 规则尝试生成
↓
若组合语义不成立
↓
该函数可能成为 deleted
```

## 13.2 assignment 对 const/reference 成员尤其敏感

例如：

```cpp
struct A {
    const int x = 0;
};
```

对 `x` 做重新赋值没有普通可写语义，因此默认 assignment 能力会受到限制。

又例如 reference member：

```cpp
struct A {
    int& ref;
};
```

引用一旦绑定，并不存在普通“让引用自身重新绑定”的 assignment 语义，因此默认 assignment 同样容易成为 deleted。

不要把这些现象背成孤立规则；它们仍然可以放进：

```text
外层操作
↓
要求每个 base/member 完成对应操作
↓
某个成员对应操作不成立
↓
外层 defaulted/implicit 操作不能成立
```

---

# 14. 第三阶段：具体调用点还要经过重载决议

即使知道类里有哪些 special member functions，也还没有结束。

例如：

```cpp
A a;
A b = std::move(a);
```

真正需要问：

```text
有哪些候选 constructor？
↓
每个候选是否可行？
↓
哪个 conversion sequence 更好？
↓
最佳候选是否 deleted / accessible？
```

所以完整模型是：

```text
第一阶段：class declaration
    ↓
哪些 special member 被 declared？

第二阶段：base/member capabilities
    ↓
这些 implicit/defaulted 函数是否 deleted？

第三阶段：call expression
    ↓
overload resolution
    ↓
最终调用 copy / move / 报错
```

这是整个专题最值得长期使用的推导框架。

---

# 15. `std::move` 到底做了什么

最容易形成的错误直觉是：

```text
std::move(x)
→ 调用 move constructor
```

这是错的。

`std::move` 的核心作用近似可以理解为：

```cpp
static_cast<T&&>(x)
```

也就是：

> **把表达式转换为适合参与右值重载匹配的值类别（通常是 xvalue）。**

它本身不搬任何资源，也不直接调用 constructor。

真正调用谁仍然由 overload resolution 决定。

例如：

```cpp
class A {
public:
    A(const A&) { /* copy */ }
};

A a;
A b = std::move(a);
```

这里由于 user-declared copy constructor 抑制了 implicit move constructor，候选中可能只有：

```cpp
A(const A&)
```

而 `const A&` 可以绑定右值，所以最终仍然 copy。

因此正确判断链是：

```text
std::move(a)
    ↓
产生 xvalue
    ↓
检查 A(A&&) 是否存在且可用
    ↓
如果没有，A(const A&) 是否可接？
    ↓
进入正常 overload resolution
```

---

# 16. 有名字的右值引用为什么仍是左值表达式

看：

```cpp
void g(A&& x) {
    f(x);
    f(std::move(x));
}
```

虽然变量 `x` 的声明类型是：

```cpp
A&&
```

但是表达式：

```cpp
x
```

是 lvalue。

原因可以从语言语义理解：

> **一旦一个对象有了名字，你就可以稳定地通过这个名字再次定位它；这个名字表达的是一个有身份的对象。**

因此：

```cpp
f(x);
```

按左值参与重载。

而：

```cpp
f(std::move(x));
```

才重新把它转换成 xvalue。

这也是为什么 move constructor 内部经常写：

```cpp
A(A&& other)
    : member(std::move(other.member))
{}
```

`other` 虽然类型是 `A&&`，但 `other.member` 通过有名字对象访问时不会自动保持“右值属性”。

---

# 17. move 不存在、move 被 delete、defaulted move 被 delete：三者必须区分

这是中高级 C++ 中非常重要的一组区别。

## 17.1 情况一：move 根本不存在

```cpp
class A {
public:
    A(const A&);
};
```

由于 user-declared copy constructor：

```text
implicit A(A&&) → 不再声明
```

于是：

```cpp
A b = std::move(a);
```

如果 `A(const A&)` 可用，则可能直接由 copy constructor 接住右值。

## 17.2 情况二：move 被程序员显式 delete

```cpp
class A {
public:
    A(const A&);
    A(A&&) = delete;
};
```

对：

```cpp
A b = std::move(a);
```

`A(A&&)` 通常是更好的匹配。

overload resolution 先选中它，然后发现：

```text
selected function is deleted
```

于是编译失败。

**不会因为第一名 deleted 就退回第二名 copy constructor。**

## 17.3 情况三：implicitly-declared / explicitly-defaulted move 在生成时成为 deleted

这是一条容易忽略的特殊规则。

如果 defaulted move constructor 因为 base/member 原因被定义为 deleted，C++ 会把这种 deleted move constructor 从 overload resolution 中忽略，以避免它阻止从右值走 copy 路径。

move assignment 也有类似设计。

所以：

```text
explicit T(T&&) = delete
```

和：

```text
defaulted T(T&&)
因为成员能力而 becomes deleted
```

不是完全相同的 overload 行为。

这是为什么我们必须精确区分“显式 delete”和“默认生成失败”。

---

# 18. 析构函数与资源所有权

析构函数解决：

> **对象生命周期结束时，需要执行哪些收尾动作？**

如果成员本身是 RAII 类型：

```cpp
class A {
    std::string s;
    std::vector<int> v;
};
```

通常不需要自己写 destructor。

编译器生成的析构流程会递归调用成员析构：

```text
~A
↓
destroy v
↓
destroy s
```

如果自己直接管理裸资源：

```cpp
class Buffer {
public:
    Buffer() : p(new int[100]) {}
    ~Buffer() { delete[] p; }

private:
    int* p;
};
```

那么 destructor 已经明确介入资源生命周期。

此时默认浅 copy：

```text
a.p ----┐
        ├── same allocation
b.p ----┘
```

就可能导致 double delete。

因此“自己需要 destructor”通常是一个很强的信号：copy/move 语义也需要重新设计。这就是 Rule of Three/Five 的来源。

---

# 19. 为什么“默认构造函数里 `new` 了内存”不会改变隐式 move 规则

这是理解语言设计哲学的关键问题。

看：

```cpp
class A {
public:
    A() {
        p = new int[100];
    }

private:
    int* p;
};
```

从程序员语义看，这显然可能是在获取资源。

但 C++ **不会分析函数体里是否出现 `new`** 来决定 copy/move 是否生成。

原因有三层。

## 19.1 `new` 不足以表达 ownership

```cpp
p = pool.allocate();
```

编译器无法仅凭这一行知道：

```text
p 是独占 owner？
p 是共享 handle？
p 只是 borrowed pointer？
资源由 pool 释放？
资源由 A 释放？
```

## 19.2 没有 `new` 也可能拥有资源

```cpp
int fd;
```

可能只是普通整数，也可能是必须 `close(fd)` 的文件描述符。

```cpp
uint64_t handle;
```

可能是普通数值，也可能代表 GPU/OS 资源。

资源所有权是一种**程序语义**，不是靠简单语法扫描能够可靠恢复的属性。

## 19.3 C++ 需要 declaration-driven、可局部推导的规则

在头文件中：

```cpp
class A {
public:
    A();
private:
    int* p;
};
```

构造函数实现可能在另一个 `.cpp` 文件。

其他 translation unit 不应该先分析 `A::A()` 函数体，才能决定 `A` 有没有 move constructor。

因此 C++ 采用：

```text
class declaration
+
base/member type capabilities
```

来建立特殊成员函数规则。

所以更准确的设计结论是：

> **特殊成员函数自动生成机制不是资源检测机制，而是基于类声明和组成类型能力的对象语义协议推导机制。**

这也是现代 C++ 强调“让 ownership 体现在类型里”的原因。

例如：

```cpp
class A {
    std::unique_ptr<int[]> p;
};
```

`unique_ptr` 自身明确表达：

```text
copy  → prohibited
move  → supported
Dtor  → release resource
```

于是 `A` 的能力可以由成员类型自动传播，而不用编译器猜裸指针的含义。

---

# 20. Rule of Three / Five / Zero

这些不是“标准强制你必须写几个函数”的语法规则，而是从资源语义和特殊成员函数生成规则总结出来的设计经验。

## 20.1 Rule of Three

经典 C++ 中，如果一个类需要自己定义以下任意一个：

```text
Destructor
CopyCtor
CopyAssign
```

通常意味着它直接管理某种资源，因此另外两个也需要认真设计。

例如：

```cpp
class Buffer {
    char* p;
};
```

如果 destructor 要：

```cpp
delete[] p;
```

那么默认 copy 只复制地址通常不正确。

所以需要同时考虑：

```text
deep copy？
禁止 copy？
共享所有权？
```

## 20.2 Rule of Five

C++11 引入 move semantics 后，还要考虑：

```text
MoveCtor
MoveAssign
```

于是资源管理型类的完整协议扩展到五项：

```text
CopyCtor
CopyAssign
MoveCtor
MoveAssign
Dtor
```

特别是：user-declared destructor/copy 操作会抑制隐式 move，因此如果类型确实应该高效移动，往往需要明确表达 move 语义。

## 20.3 Rule of Zero

现代 C++ 更推荐：

> **业务类尽量不要直接管理裸资源，让成员类型自己承担资源所有权。**

例如：

```cpp
class Document {
    std::string name;
    std::vector<int> data;
    std::unique_ptr<Resource> resource;
};
```

通常无需手写五个特殊成员函数。

外层类的 copy/move/destruct 能力由成员自动推导：

```text
string      → copy + move
vector      → copy + move
unique_ptr  → no copy + move
```

因此 `Document` 会自然获得与成员组合一致的对象语义。

这就是 Rule of Zero：

```text
资源管理下沉到专门 RAII 类型
↓
业务类型只组合这些类型
↓
尽量使用编译器生成的特殊成员函数
```

---

# 21. C++17 拷贝消除：不要机械地数 copy/move

考虑：

```cpp
class A {
public:
    A();
    A(const A&);
    A(A&&);
};

A make() {
    return A{};
}

A a = make();
```

如果按照“先创建临时对象，再 move，再 move”的旧直觉，很容易错误地数出多个移动构造。

C++17 对这种 prvalue 场景提供 guaranteed copy elision 语义：

```cpp
return A{};
```

可以直接构造最终目标对象，不需要存在那些中间 copy/move 操作。

因此甚至：

```cpp
class A {
public:
    A() = default;
    A(const A&) = delete;
    A(A&&) = delete;
};

A make() {
    return A{};
}

A a = make();
```

在相应 C++17 guaranteed-elision 场景下仍可成立。

但要区分：

```cpp
A make() {
    A a;
    return a;
}
```

这里是 NRVO（Named Return Value Optimization）场景，不应简单与 `return A{};` 的 C++17 guaranteed copy elision 混为一谈。

学习原则：

```text
return T{}
→ 先想到 C++17 prvalue / guaranteed elision

T x;
return x;
→ 先想到 NRVO，继续分析
```

---

# 22. 多态基类为什么通常需要 `virtual destructor`

如果：

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

通过基类指针删除动态类型为派生类的对象，而基类析构不是 virtual，在这种典型场景下会产生未定义行为。

正确的多态基类通常应该提供：

```cpp
class Base {
public:
    virtual ~Base() = default;
};
```

于是：

```cpp
Base* p = new Derived;
delete p;
```

能够沿动态类型正确执行析构链：

```text
Derived::~Derived()
↓
Base::~Base()
```

但是这里又连接回本专题的一个重要规则：

```cpp
virtual ~Base() = default;
```

仍然是 user-declared destructor。

所以它会影响 implicit move constructor / move assignment。

这正是为什么“多态基类设计”和“Rule of Five/Zero”经常相互关联。

---

# 23. 一套可重复使用的推导算法

以后看到任何特殊成员函数问题，不要先背口诀，而按下面步骤。

## Step 1：列出类中 user-declared 的函数

先分类：

```text
DfltCtor?
OtherCtor?
CopyCtor?
MoveCtor?
CopyAssign?
MoveAssign?
Dtor?
```

特别注意：

```cpp
= default
= delete
```

都仍然属于 user-declared。

## Step 2：使用二维表判断第一阶段

对目标函数判断：

```text
SELF
S
D
P
✓
```

此时只回答：

> **编译器是否建立这个函数，以及声明关系是否直接导致 deleted/deprecated。**

## Step 3：检查 base/member 能力

问：

```text
对应的 base/member 操作是否存在？
是否 accessible？
是否 deleted？
成员本身是否允许 assignment？
```

由此判断 defaulted/implicit special member 是否最终 deleted。

## Step 4：确定表达式值类别

例如：

```cpp
a               // lvalue
std::move(a)    // xvalue
A{}             // prvalue
```

以及：

```cpp
void f(A&& x) {
    // x 作为表达式是 lvalue
}
```

## Step 5：做 overload resolution

候选可能有：

```cpp
A(const A&)
A(A&&)
```

判断最佳匹配。

## Step 6：检查 selected function 是否可用

最后再检查：

```text
deleted?
inaccessible?
```

不要在 overload resolution 之前就用“deleted 所以忽略”这种粗糙模型；只有标准特别规定被忽略的 defaulted-deleted move 等情况才按相应规则处理。

## Step 7：最后检查 C++17 copy elision

如果是：

```cpp
return T{};
T x = T{};
```

不要先机械假设一定发生 copy/move。

---

# 24. 常见误区与纠正

## 误区 1：没写特殊成员函数，编译器一定全部生成

错误。

正确模型：

```text
先看声明抑制规则
↓
再看 base/member 能力
↓
再看是否 deleted
```

## 误区 2：`= default` 等价于“没写”

错误。

```cpp
~A() = default;
```

仍是 user-declared destructor，因此会抑制 implicit move。

## 误区 3：`= delete` 等于函数不存在

错误。

显式 deleted function 仍参与候选集合；如果它是最佳候选，调用会报错。

## 误区 4：`std::move` 会移动资源

错误。

它主要进行值类别转换。资源是否真正移动由被选中的函数实现决定。

## 误区 5：变量类型是 `T&&`，使用它时就是右值

错误。

有名字的 `T&& x` 中，表达式 `x` 是 lvalue。

## 误区 6：带参数 constructor 都不会影响 implicit move

不准确。

`A(int)` 这种 ordinary constructor 不直接抑制 move，但：

```cpp
A(const A&)
A(A&&)
```

也是带参数 constructor，却属于 special member functions，有专门规则。

## 误区 7：程序员在 constructor 中 `new`，编译器就知道这是 owner

错误。

C++ 不通过函数体分析 ownership。裸指针/整数 handle 的资源语义对编译器并不透明可知。

## 误区 8：写了 destructor 就没有 copy

错误。

C++17 中 user-declared destructor 会抑制 implicit move，但 implicit copy 仍可能存在，只是这种组合属于 deprecated implicit-copy 方向。

## 误区 9：move 被 delete 后一定不能从右值构造

不够精确。

需要区分：

```text
move 根本没声明
explicit move = delete
defaulted move 因组合失败成为 deleted
```

它们对 overload resolution 的行为并不完全相同。

## 误区 10：`return T{}` 一定调用一次 move

C++17 下错误。

先检查 guaranteed copy elision。

---

# 25. 学习检查点

在进入新的题目之前，建议能够不用查表回答并解释下面问题。

### 检查点 1

```cpp
class A {
public:
    A() = default;
};
```

为什么它不会抑制 implicit copy/move/assignment/destructor？

### 检查点 2

```cpp
class A {
public:
    A(int);
};
```

为什么它会阻止 implicit `A()`，却不会因为这一点阻止 implicit move constructor？

### 检查点 3

```cpp
class A {
public:
    ~A() = default;
};
```

为什么 `= default` 的析构仍会影响 implicit move？

### 检查点 4

为什么：

```cpp
A(A&&) = delete;
```

和“根本没有 `A(A&&)`”在 overload resolution 中不是同一件事？

### 检查点 5

为什么 C++ 不通过 constructor 中是否出现 `new` / `malloc` / `open` 判断类型是否拥有资源？

### 检查点 6

解释这三个层次：

```text
declaration suppression
↓
defaulted function becomes deleted
↓
overload resolution
```

### 检查点 7

为什么现代 C++ 更推荐：

```cpp
std::unique_ptr<T>
```

而不是：

```cpp
T* owner;
```

来表达独占 ownership？

### 检查点 8

为什么：

```cpp
void f(A&& x) {
    g(x);
}
```

中的 `x` 不会自动按右值参与 `g` 的重载决议？

如果这些问题都可以从设计逻辑推导，而不是背结论，这一专题的基础模型才算真正建立。

---

# 26. 参考资料

本文语义基线为 C++17，规则整理主要对应以下标准主题：

- ISO C++17 working draft N4659：`[class.ctor]`、`[class.copy]`、`[class.copy.ctor]`、`[class.copy.assign]`；
- cppreference：Default constructors；
- cppreference：Copy constructors；
- cppreference：Move constructors；
- cppreference：Copy assignment operator；
- cppreference：Move assignment operator；
- cppreference：Destructors；
- cppreference：The rule of three/five/zero。

阅读标准材料时，要特别区分标准措辞中的：

```text
user-declared
implicitly-declared
implicitly-defined
defaulted
deleted
```

不要用“编译器帮我生成/没生成”一句话覆盖所有不同阶段。

---

# 结语：真正应该带走的模型

这一专题最终不是为了记住一张二维表。

真正需要形成的是下面这个模型：

```text
            class declaration
                  │
                  ▼
     第一阶段：哪些 special member
            会被 declared？
                  │
                  ▼
      base/member type capabilities
                  │
                  ▼
     第二阶段：defaulted/implicit
        function 是否 deleted？
                  │
                  ▼
          call expression
       + expression value category
                  │
                  ▼
       第三阶段：overload resolution
                  │
                  ▼
        copy / move / compile error
```

再往上抽象一层：

> **C++ 不试图猜测你的资源管理意图；它要求类型通过声明和成员类型表达对象语义，然后由语言规则机械地组合这些语义。**

这正是 RAII、Rule of Zero、move semantics 以及整套特殊成员函数规则能够统一起来的设计逻辑。