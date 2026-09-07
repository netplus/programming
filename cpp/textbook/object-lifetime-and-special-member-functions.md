# C++ 对象模型、生命周期与特殊成员函数

> **语义基线：C++17**  
> 本文从 **对象在内存中如何存在** 与 **C++ 对象语义如何建立** 出发，再推导 constructor / destructor / copy / move / `= default` / `= delete` 以及特殊成员函数之间的生成关系。  
> 学习目标不是背规则表，而是形成一套可以自行推导的对象模型。

---

## 目录

1. [从最基本的问题开始：什么是 C++ 对象](#1-从最基本的问题开始什么是-c-对象)
2. [对象的 storage、lifetime、type、value 与 identity](#2-对象的-storagelifetimetypevalue-与-identity)
3. [对象内存布局：object representation 与 value representation](#3-对象内存布局object-representation-与-value-representation)
4. [完整对象与子对象：C++ 类为什么天然是组合结构](#4-完整对象与子对象c-类为什么天然是组合结构)
5. [内存布局不等于对象语义](#5-内存布局不等于对象语义)
6. [构造与析构：让一块 storage 中的对象开始和结束生命](#6-构造与析构让一块-storage-中的对象开始和结束生命)
7. [为什么构造顺序和析构顺序由子对象结构决定](#7-为什么构造顺序和析构顺序由子对象结构决定)
8. [constructor 应该怎样分类](#8-constructor-应该怎样分类)
9. [六类特殊成员函数分别解决什么问题](#9-六类特殊成员函数分别解决什么问题)
10. [construction 与 assignment 为什么必须分开](#10-construction-与-assignment-为什么必须分开)
11. [默认 copy/move 为什么是逐子对象语义，而不是简单 memcpy](#11-默认-copymove-为什么是逐子对象语义而不是简单-memcpy)
12. [值语义、所有权语义与 RAII](#12-值语义所有权语义与-raii)
13. [理解规则前必须掌握的术语](#13-理解规则前必须掌握的术语)
14. [`= default`：显式选择编译器定义的默认对象语义](#14--default显式选择编译器定义的默认对象语义)
15. [`= delete`：在类型系统里禁止某种对象操作](#15--delete在类型系统里禁止某种对象操作)
16. [特殊成员函数第一阶段：声明与抑制关系](#16-特殊成员函数第一阶段声明与抑制关系)
17. [完整二维关系表](#17-完整二维关系表)
18. [为什么这张表必须区分 S、D、P](#18-为什么这张表必须区分-sdp)
19. [逐行解释抑制矩阵](#19-逐行解释抑制矩阵)
20. [第二阶段：base/member 能力如何传播到外层对象](#20-第二阶段basemember-能力如何传播到外层对象)
21. [第三阶段：具体调用点还要经过 overload resolution](#21-第三阶段具体调用点还要经过-overload-resolution)
22. [`std::move` 与对象语义：它并不移动内存](#22-stdmove-与对象语义它并不移动内存)
23. [为什么有名字的 `T&&` 是 lvalue expression](#23-为什么有名字的-t-是-lvalue-expression)
24. [move 不存在、显式 deleted、defaulted 后 deleted](#24-move-不存在显式-deleteddefaulted-后-deleted)
25. [为什么 constructor 中 `new` 不会改变隐式生成规则](#25-为什么-constructor-中-new-不会改变隐式生成规则)
26. [Rule of Three / Five / Zero](#26-rule-of-three--five--zero)
27. [多态对象、vptr 与 virtual destructor](#27-多态对象vptr-与-virtual-destructor)
28. [C++17 copy elision：对象甚至可能直接在最终 storage 中产生](#28-c17-copy-elision对象甚至可能直接在最终-storage-中产生)
29. [统一推导算法](#29-统一推导算法)
30. [常见误区](#30-常见误区)
31. [学习检查点](#31-学习检查点)
32. [参考资料](#32-参考资料)

---

# 1. 从最基本的问题开始：什么是 C++ 对象

学习 constructor、destructor、copy、move 之前，先暂时忘掉这些函数。

先看：

```cpp
struct Point {
    int x;
    int y;
};

Point p{1, 2};
```

从机器视角看，`p` 最终需要占据一片 storage；但从 C++ 语言视角看，**对象不只是若干字节**。

C++ 对象至少同时具有下面几个维度：

```text
              object
                 │
     ┌───────────┼───────────┐
     │           │           │
   storage      type       lifetime
     │           │           │
  若干字节     如何解释      何时对象存在
                 │
            value / identity
```

C++ 标准的对象模型明确区分“storage”和“object”：一个对象在其构造期间、生命周期期间以及析构期间占据一片 storage，但**storage 本身并不等价于一个具有完整语义的对象**。

这一区分是后面所有生命周期规则的起点。

---

# 2. 对象的 storage、lifetime、type、value 与 identity

可以先建立五个概念。

## 2.1 Storage：对象占据的存储区域

例如：

```cpp
Point p;
```

`p` 需要占据至少 `sizeof(Point)` 字节的 storage。

但“有这块内存”不代表其中任何时刻都必然存在一个活着的 `Point` 对象。

例如 placement new、显式析构、union 等高级场景都会让我们看到：

```text
storage lifetime
≠
object lifetime
```

当前阶段先建立这个概念即可。

## 2.2 Type：这些字节应按照什么规则解释

同样四个字节：

```text
00 00 00 01
```

它作为 `int`、`float` 或某个 class 的一部分时，语言语义完全不同。

所以对象不是“内存 + 猜测”，而是：

```text
storage
+
type
+
lifetime rules
```

## 2.3 Value：对象当前表示什么状态

例如：

```cpp
int x = 3;
```

对象 `x` 的 value 是 `3`。

对 class 来说，value 往往由多个子对象共同构成。

## 2.4 Identity：这是“哪一个对象”

```cpp
Point a{1, 2};
Point b{1, 2};
```

虽然：

```text
a 的值 == b 的值
```

但：

```text
&a != &b
```

它们是两个不同对象。

这一区别对 move 很重要：

> **move 通常改变对象的 value/resource state，但并不会把源对象本身从内存里“搬走”，也不会结束源对象的 lifetime。**

## 2.5 Lifetime：何时可以把这片 storage 当成该类型对象使用

constructor 与 destructor 的深层意义之一，就是参与建立和终止对象的生命周期语义。

因此以后不要只把 constructor 理解成“初始化几个变量”，而应理解为：

> **让一个特定类型的对象在 storage 中进入合法存在状态，并建立它所有子对象的初始状态。**

---

# 3. 对象内存布局：object representation 与 value representation

对一个完整非 bit-field 对象 `T`，标准用两个重要概念描述其内存表示。

## 3.1 Object representation

可以粗略理解为：

> **这个对象在内存中占据的全部字节。**

其大小对应 `sizeof(T)`。

例如：

```cpp
struct A {
    char c;
    int x;
};
```

一个典型实现可能形成：

```text
低地址
┌──────────────┐
│ c            │ 1 byte
├──────────────┤
│ padding      │ 3 bytes（示意）
├──────────────┤
│ x            │ 4 bytes
└──────────────┘
高地址
```

这里 padding 的具体数量取决于 ABI、alignment 和实现。

不要把示意图当成所有平台的固定布局。

## 3.2 Value representation

对象表示里的所有 bit 不一定都参与表示这个对象的 value。

没有参与 value 表示的 bit 可以是 padding bits。

所以：

```text
object representation
      │
      ├── value representation
      └── padding / representation detail
```

这个区别直接解释一个重要事实：

> **“两个对象具有相同的 C++ value”并不要求它们的每一个对象表示字节都完全相同。**

## 3.3 `sizeof(T)` 不是“所有成员 sizeof 直接相加”

因为对象布局还可能受到：

```text
alignment
padding
base-class layout
ABI requirements
implementation details
```

影响。

所以：

```cpp
sizeof(A)
```

不能一般性地通过“成员大小机械相加”推导。

## 3.4 不要把通用 class copy 理解成 object representation 的字节复制

对普通非 union class，隐式定义的 copy/move constructor 是**逐 base/member 子对象操作**。

只有 trivially copyable 等满足特定条件的类型，才允许以 `memcpy`/`memmove` 一类方式复制对象表示并保持语言允许的语义。

这是后面理解 default copy/move 的重要前提。

---

# 4. 完整对象与子对象：C++ 类为什么天然是组合结构

标准对象模型中，一个对象可以包含其他对象，这些内部对象称为 **subobject**。

典型 subobject 包括：

```text
member subobject
base-class subobject
array-element subobject
```

例如：

```cpp
struct Header {
    int type;
};

struct Packet : Header {
    std::string payload;
    int seq;
};

Packet p;
```

可以从语义上画成：

```text
complete object: p (Packet)
│
├── base subobject: Header
│   └── member subobject: type
│
├── member subobject: payload
└── member subobject: seq
```

这里最重要的不是精确地址，而是：

> **一个 class object 在语言语义上是由 base subobjects 和 member subobjects 组合而成的。**

这就是为什么 C++ 能自动生成 constructor/copy/move/destructor：

```text
外层对象语义
=
对子对象执行相应操作的组合
```

这是本教材最核心的模型之一。

---

# 5. 内存布局不等于对象语义

这是整个专题中非常重要的一条边界。

考虑：

```cpp
class Buffer {
    int* p;
    std::size_t size;
};
```

从对象内存布局看，`Buffer` 内部可能只是：

```text
p     : 一个地址值
size  : 一个整数值
```

但从程序设计语义看，`p` 可能有完全不同的含义：

```text
A. p 是 borrowed pointer
B. p 是 unique owner
C. p 指向共享资源
D. p 只是缓存地址
```

这些 ownership semantics **无法从那几个字节本身推断出来**。

同样：

```cpp
int fd;
```

在内存里就是一个整数，但语义上可能代表一个必须 `close()` 的 OS file descriptor。

所以应始终区分：

```text
physical representation
        │
        │ 不能自动推出
        ▼
semantic ownership / protocol
```

这正是后面 `Rule of Zero` 与 RAII 的根本价值：

> 把 ownership 从“隐藏在函数体和裸值里的程序员约定”，提升为“由成员类型明确表达的类型语义”。

---

# 6. 构造与析构：让一块 storage 中的对象开始和结束生命

现在重新看 constructor。

```cpp
class A {
public:
    A();
    ~A();
private:
    std::string s;
    int x;
};
```

## 6.1 Constructor 不是“先进入函数体，再创建成员”

实际上在 `A::A()` 的函数体执行之前，相关 base/member 子对象已经完成初始化。

概念上：

```text
获得 A 的 storage
        ↓
构造 base subobjects
        ↓
构造 member subobjects
        ↓
进入 A::A() 函数体
        ↓
A 成为完成构造的对象
```

所以 member initializer list 的地位非常重要：它决定子对象应该怎样构造，而不是在函数体中“事后赋值”。

## 6.2 Destructor 是反方向的生命周期协议

析构大致可以理解为：

```text
执行 ~A() 函数体
        ↓
销毁 member subobjects（逆序）
        ↓
销毁 base subobjects（逆序）
        ↓
A 的对象生命周期结束
```

因此 constructor / destructor 天然与对象的“子对象树”绑定。

---

# 7. 为什么构造顺序和析构顺序由子对象结构决定

考虑：

```cpp
struct B1 {};
struct B2 {};

struct D : B1, B2 {
    M1 m1;
    M2 m2;
};
```

忽略 virtual base 的额外规则，普通场景的初始化顺序可以理解为：

```text
B1
↓
B2
↓
m1
↓
m2
↓
D constructor body
```

注意：成员按照**声明顺序**初始化，不是按照 member-initializer-list 的书写顺序随意决定。

销毁则反向：

```text
D destructor body
↓
m2
↓
m1
↓
B2
↓
B1
```

为什么必须反向？

因为后构造的对象可能依赖先构造对象已经存在；反向销毁可以维持这种依赖关系直到最后一刻。

这也是典型的 stack / RAII 生命周期结构。

对于 virtual bases，最派生对象的构造还需要遵循 virtual-base 的专门初始化顺序；当前专题先抓住“对象是子对象组合，生命周期逐层建立/撤销”即可。

---

# 8. Constructor 应该怎样分类

不要简单分成“无参构造”和“带参构造”，因为这会把 copy/move constructor 混在普通构造函数里。

更稳定的分类是：

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
│
├── copy constructor
│      A(const A&)
│
└── move constructor
       A(A&&)
```

## 8.1 Default constructor

它的关键不是“参数表一定为空”，而是可以在不提供实参时调用。

```cpp
A();
A(int x = 0); // 也可以是 default constructor
```

## 8.2 Ordinary constructor

例如：

```cpp
A(int);
```

它表达：

> **如何根据业务输入创建一个新的 A。**

它不是特殊成员函数中的 copy/move protocol。

## 8.3 Copy/move constructor

```cpp
A(const A&);
A(A&&);
```

虽然都有参数，但它们描述的是：

```text
已有 A
   ↓
如何构造新的 A
```

因此必须从 ordinary constructor 中单独拿出来理解。

---

# 9. 六类特殊成员函数分别解决什么问题

```cpp
class T {
public:
    T();
    T(const T&);
    T(T&&);
    T& operator=(const T&);
    T& operator=(T&&);
    ~T();
};
```

| 函数 | 对象语义 |
|---|---|
| `T()` | 建立一个默认状态的新对象 |
| `T(const T&)` | 从现有对象复制构造一个新对象 |
| `T(T&&)` | 从可被消费状态的对象构造一个新对象 |
| `operator=(const T&)` | 目标对象已存在，复制新的状态给它 |
| `operator=(T&&)` | 目标对象已存在，把可转移状态交给它 |
| `~T()` | 结束对象生命周期并完成资源收尾 |

`T(int)` 等 ordinary constructor 不属于这六项特殊成员函数。

---

# 10. Construction 与 assignment 为什么必须分开

## 10.1 Copy construction

```cpp
T a;
T b = a;
```

`b` 原来不存在。

核心语义：

```text
在 b 的 storage 中建立一个新 T
↓
用 a 的状态初始化 b 的子对象
```

## 10.2 Copy assignment

```cpp
T a;
T b;
b = a;
```

这里 `b` 已经是一个活着的完整对象。

因此 assignment 必须面对额外问题：

```text
b 原来的资源 / 状态怎么办？
```

例如：

```text
释放旧资源？
复用旧 buffer？
处理自赋值？
如何保证异常安全？
```

所以：

```text
construction = 从无到有
assignment   = 已有对象改变 value/resource state
```

绝不能混为一谈。

---

# 11. 默认 copy/move 为什么是逐子对象语义，而不是简单 memcpy

假设：

```cpp
struct Record : Base {
    std::string name;
    std::vector<int> values;
};
```

对普通非 union class，隐式定义的 copy constructor 概念上做的是：

```text
copy Base subobject
copy name subobject
copy values subobject
```

而 move constructor 则概念上：

```text
move Base subobject
move name subobject
move values subobject
```

这叫 **memberwise copy/move**，但这里的 memberwise 应理解为广义的“bases + members 的逐子对象语义”。

例如：

```cpp
Record r2 = r1;
```

不是一般性地等价于：

```cpp
std::memcpy(&r2, &r1, sizeof(Record)); // 一般不成立
```

因为：

```cpp
std::string
std::vector<int>
```

内部有自己的不变量和资源协议。

它们必须执行自己的 copy constructor。

这直接给出一个极其重要的统一模型：

```text
外层对象的默认 copy/move
            ↓
对子对象分别执行对应 copy/move
            ↓
子对象能力决定外层能力
```

后面的“base/member capability propagation”其实就是这一点的语言规则化。

---

# 12. 值语义、所有权语义与 RAII

对象的字节布局只是底层载体；类型真正重要的是它表达什么语义。

可以把常见类型粗分成：

## 12.1 Value type

例如：

```cpp
std::string
std::vector<int>
Point
```

copy 通常意味着：

> 得到一个独立但等价的值。

## 12.2 Unique ownership type

例如：

```cpp
std::unique_ptr<T>
```

语义：

```text
copy  → 没有合理语义，禁止
move  → 转移唯一所有权
dtor  → 释放资源
```

## 12.3 Shared ownership type

例如：

```cpp
std::shared_ptr<T>
```

语义：

```text
copy  → 增加共享所有权
move  → 转移一个 handle
dtor  → 减少引用计数
```

## 12.4 Non-owning type

例如：

```cpp
T*
std::string_view
```

通常表示观察/借用，不负责资源生命期。

这些语义比成员在内存中占几个字节更重要。

RAII 的核心就是：

> **把资源 ownership 与对象 lifetime 绑定，并让这种语义由类型本身表达。**

---

# 13. 理解规则前必须掌握的术语

## 13.1 user-declared

程序员明确声明了函数。

```cpp
~A();
~A() = default;
~A() = delete;
```

以上都属于 user-declared destructor。

所以：

```text
= default
≠
“我没有声明”
```

## 13.2 implicitly-declared

程序员没写，但语言规则替类声明相应函数。

```text
implicitly-declared
≠
最终一定能用
```

之后还可能成为 deleted。

## 13.3 implicitly-defined

已经存在的 implicit/defaulted 函数，在需要真正生成语义时由编译器形成定义。

## 13.4 defaulted

```cpp
A(A&&) = default;
```

表示：

> 请按照标准规定的 base/member 默认语义生成这个函数。

## 13.5 deleted

```cpp
A(A&&) = delete;
```

函数存在，但被禁止调用。

## 13.6 user-provided

它比 user-declared 更细。

例如类内第一次声明：

```cpp
A() = default;
```

是 user-declared，但通常不是 user-provided。

而先声明、之后在类外 `= default`，分类会有所不同，并会影响 triviality 等属性。

当前专题只需记住：

```text
user-declared
≠
user-provided
```

---

# 14. `= default`：显式选择编译器定义的默认对象语义

为什么已经有隐式生成，还需要 `= default`？

因为：

> **“我要这个函数作为 API 存在”和“我要自己手写它的实现”是两回事。**

例如：

```cpp
class A {
public:
    A(int);
};
```

由于已经有 user-declared constructor，编译器不会再隐式给 `A()`。

但如果设计上需要：

```cpp
class A {
public:
    A() = default;
    A(int);
};
```

这表示：

```text
A() 是类型接口的一部分
+
它采用编译器规定的默认子对象构造语义
```

同理：

```cpp
A(const A&) = default;
A(A&&) = default;
A& operator=(const A&) = default;
A& operator=(A&&) = default;
~A() = default;
```

都表示显式选择默认语义。

但：

> **`= default` 不保证最终可用。**

如果某个 base/member 对应操作不可用，defaulted function 仍可能成为 deleted。

---

# 15. `= delete`：在类型系统里禁止某种对象操作

例如一个独占句柄：

```cpp
class UniqueHandle {
public:
    UniqueHandle(const UniqueHandle&) = delete;
    UniqueHandle& operator=(const UniqueHandle&) = delete;
};
```

这里不是“忘了实现 copy”。

而是：

```text
copy 本身违反类型语义
↓
从 API 层禁止表达这种操作
```

所以 `= delete` 是一种类型设计工具。

它和“函数根本不存在”也不同，因为显式 deleted function 仍然会参与 overload resolution。

---

# 16. 特殊成员函数第一阶段：声明与抑制关系

现在进入最容易被背成口诀的部分。

先明确：语言不会打开 constructor 函数体去研究：

```text
有没有 new？
有没有 malloc？
有没有 open？
是不是 owner？
```

特殊成员函数生成规则是 **declaration-driven** 的。

也就是说，第一阶段只看：

```text
类声明了哪些 constructor / copy / move / assignment / destructor？
```

然后根据固定标准规则决定其余特殊成员函数是否 implicit declaration。

这里有三种完全不同的后果：

```text
S：目标函数根本不 implicitly declared
D：目标函数 implicitly declared，但因 move 声明关系 defined as deleted
P：目标 copy 仍可 defaulted，但该隐式定义组合在 C++17 中已 deprecated
```

不能把它们统一写成一个“×”。

---

# 17. 完整二维关系表

为避免“带参数 constructor”的歧义，把普通非 copy/move constructor 单独记为 `OtherCtor`。

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
| `S` | Suppressed：目标函数根本不会被隐式声明 |
| `D` | 目标函数会隐式声明，但被定义为 deleted |
| `P` | C++17 中该 implicit copy 组合仍存在，但其隐式定义属于 deprecated |
| `✓` | 当前声明对目标没有第一阶段的直接关闭作用 |

注意：

```text
✓ ≠ 最终一定可调用
```

第二阶段还要检查所有 base/member。

---

# 18. 为什么这张表必须区分 S、D、P

## 18.1 S：函数不存在于 implicit candidate set 中

```cpp
class A {
public:
    A(const A&);
};
```

这里没有 implicit move constructor。

因此：

```cpp
A b = std::move(a);
```

仍可能由：

```cpp
A(const A&)
```

接住 xvalue 并发生 copy。

## 18.2 D：函数存在，但不可用

```cpp
class A {
public:
    A(A&&);
};
```

implicit copy constructor 会被定义为 deleted。

这是：

```text
函数存在
但禁止使用
```

不是“根本没有”。

## 18.3 P：兼容历史，但设计上不要继续依赖

例如：

```cpp
class A {
public:
    ~A();
};
```

C++17 下 implicit copy 仍可能存在，但“自己声明 destructor，却继续依赖 implicit copy”的组合属于 deprecated implicit-copy 方向。

它与 Rule of Three 的设计背景高度一致。

---

# 19. 逐行解释抑制矩阵

这里的“原因”必须区分两层：

```text
规范层：标准明确规定这些 declaration conditions
设计直觉层：这些规则大体围绕对象复制/转移/销毁协议形成
```

不要把设计直觉误当成“编译器理解 ownership”。

## 19.1 DfltCtor

```cpp
A() = default;
```

它表达：

```text
如何建立一个默认状态的新对象
```

本身不直接定制已有对象之间的 copy/move/assignment，也不定制 destruction。

所以其它五类目标均不因它直接关闭。

## 19.2 OtherCtor

```cpp
A(int);
```

它是 user-declared constructor，所以：

```text
implicit A() → S
```

但它仍主要是“新对象如何建立”的协议，不直接关闭 copy/move/assignment/dtor。

## 19.3 CopyCtor

```cpp
A(const A&);
```

首先它本身是 constructor：

```text
implicit A() → S
```

其次，根据 C++17 规则：

```text
implicit MoveCtor   → S
implicit MoveAssign → S
```

CopyAssign 仍可 defaulted，但对应 implicit-copy 组合属于 `P`。

## 19.4 MoveCtor

```cpp
A(A&&);
```

因为是 constructor：

```text
DfltCtor → S
```

同时：

```text
implicit CopyCtor   → D
implicit CopyAssign → D
implicit MoveAssign → S
```

这是 C++11 后 move-side 声明对 copy-side 默认能力的明确影响。

## 19.5 CopyAssign

```cpp
A& operator=(const A&);
```

它不属于 constructor，所以：

```text
DfltCtor → ✓
```

但：

```text
MoveCtor   → S
MoveAssign → S
CopyCtor   → P
```

## 19.6 MoveAssign

```cpp
A& operator=(A&&);
```

它会：

```text
CopyCtor   → D
CopyAssign → D
MoveCtor   → S
```

但不影响 implicit default constructor。

## 19.7 Destructor

```cpp
~A();
```

它不会关闭 default constructor，但会：

```text
MoveCtor   → S
MoveAssign → S
```

CopyCtor / CopyAssign 在 C++17 中仍可能存在，但属于 `P`。

这里可以得到一个重要认识：

> user-declared destructor 对 move 的影响是**声明规则**，不是因为编译器分析出 destructor 里面正在 `delete` 某个资源。

---

# 20. 第二阶段：base/member 能力如何传播到外层对象

二维表只解决第一阶段：

```text
这个函数会不会被 implicit/defaulted 地建立？
```

接下来必须回到对象模型。

假设：

```cpp
struct B {};

struct M {
    M(M&&) = delete;
};

struct A : B {
    M m;
};
```

`A` 在语义上包含：

```text
A complete object
├── B base subobject
└── M member subobject m
```

如果编译器尝试 default move `A`：

```text
move B
move m
```

那么对 `m` 必须找到一个可用的初始化路径。

如果对应操作不可用，A 的 defaulted move 可能被定义为 deleted。

所以能力传播模型是：

```text
T default constructible
  ← 所需 bases/members 能建立初始状态

T copy constructible
  ← bases/members 可执行相应 copy construction

T move constructible
  ← bases/members 从 xvalue 能完成相应 construction

T copy assignable
  ← bases/members 可完成 copy assignment

T move assignable
  ← bases/members 可完成 move assignment

T destructible
  ← bases/members 都可被销毁
```

这不是一组独立口诀，而是对象子结构的直接结果。

## 20.1 const/reference member 为什么常使 assignment 被 deleted

```cpp
struct A {
    const int x = 0;
};
```

`A` 已经存在后，对 `x` 没有普通“重新赋值”的语义，所以默认 assignment 会受限制。

再如：

```cpp
struct A {
    int& ref;
};
```

reference 绑定后不能通过 assignment 让引用本身重新绑定，因此外层默认 assignment 同样受影响。

从对象模型看，这并不神秘：

```text
外层 assignment
↓
要求每个子对象完成对应 assignment
↓
某个子对象语义不成立
↓
外层 defaulted assignment 不能成立
```

---

# 21. 第三阶段：具体调用点还要经过 overload resolution

类的特殊成员函数集合确定以后，还不能直接说“这一行一定 move”。

例如：

```cpp
A a;
A b = std::move(a);
```

需要依次问：

```text
A(A&&) 存在吗？
A(const A&) 存在吗？
各自可用吗？
当前参数是什么 value category？
哪个 conversion sequence 更好？
最佳候选是否 deleted / inaccessible？
```

所以完整分析有三层：

```text
第一层：class declaration
        ↓
哪些 special members 被 declared？

第二层：object composition
        ↓
这些 defaulted operations 是否可成立？

第三层：call expression
        ↓
overload resolution
        ↓
最终 copy / move / error
```

---

# 22. `std::move` 与对象语义：它并不移动内存

最常见误解：

```text
std::move(a)
→ 把 a 的内存移动走
```

错误。

`std::move` 本质上是一个 value-category cast，近似理解为：

```cpp
static_cast<T&&>(a)
```

它做的是：

```text
把表达式标记成“可以按右值语义参与重载”
```

而不是：

```text
memcpy
改变对象地址
结束源对象 lifetime
自动转移任何资源
```

真正资源如何转移，由最终选中的 move constructor / move assignment 及其成员操作决定。

例如：

```cpp
std::vector<int> v2 = std::move(v1);
```

常见实现会转移内部 buffer ownership，但：

```text
v1 对象本身仍在原地址
v1 lifetime 仍继续
之后仍会执行 v1 destructor
```

所以 move 应理解为：

> **改变/转移对象所代表的 value/resource state，而不是把对象实体的 storage 搬到别处。**

---

# 23. 为什么有名字的 `T&&` 是 lvalue expression

```cpp
void g(A&& x) {
    f(x);
    f(std::move(x));
}
```

`x` 的 declared type 是：

```cpp
A&&
```

但表达式：

```cpp
x
```

是 lvalue。

可以从对象 identity 理解：

> 一旦对象可以通过一个名字稳定定位，这个名字表达的是“这个有身份的对象”。

所以：

```cpp
f(x);            // 按 lvalue
f(std::move(x)); // 显式转换成 xvalue
```

这也是 move constructor 内部经常必须写：

```cpp
A(A&& other)
    : member(std::move(other.member))
{}
```

的原因。

---

# 24. Move 不存在、显式 deleted、defaulted 后 deleted

这三种情况必须分开。

## 24.1 Move 根本不存在

```cpp
class A {
public:
    A(const A&);
};
```

user-declared copy constructor 抑制 implicit move constructor。

于是：

```cpp
A b = std::move(a);
```

可能由 `A(const A&)` 接住右值并 copy。

## 24.2 显式 move = delete

```cpp
class A {
public:
    A(const A&);
    A(A&&) = delete;
};
```

`A(A&&)` 仍然进入 overload resolution，而且对 xvalue 通常匹配更好。

如果它被选中：

```text
best candidate = deleted
↓
compile error
```

不会自动退回第二名 copy constructor。

## 24.3 Defaulted move 因子对象能力失败而成为 deleted

```cpp
class A {
public:
    A(A&&) = default;
private:
    M m;
};
```

如果编译器按照子对象规则无法形成 A 的 move constructor，它可能被定义为 deleted。

C++ 对这种 **defaulted move constructor becomes deleted** 有专门的 overload-resolution 处理：它会被忽略，以免阻止从右值走可用的 copy 路径。

所以：

```text
explicit A(A&&) = delete
```

和：

```text
defaulted A(A&&) 因生成失败成为 deleted
```

不能简单视为同一种候选行为。

---

# 25. 为什么 constructor 中 `new` 不会改变隐式生成规则

考虑：

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

从业务语义看，它似乎在“管理资源”。

为什么 C++ 不因此自动关闭 copy/move？

答案就在“内存布局 vs 对象语义”的区别。

## 25.1 编译器无法从 `new` 推断 ownership

```cpp
p = pool.allocate();
```

可能表示：

```text
unique ownership
shared handle
borrowed memory
pool-owned resource
```

函数体本身不足以形成稳定类型规则。

## 25.2 没有 `new` 也可能拥有资源

```cpp
int fd;
```

可能是 OS file descriptor。

```cpp
uint64_t handle;
```

可能是 GPU / kernel resource handle。

所以“资源”是程序语义，不是简单语法模式。

## 25.3 Separate compilation 也要求 declaration-driven 规则

头文件：

```cpp
class A {
public:
    A();
private:
    int* p;
};
```

实现可能在另一个 `.cpp`。

其他 translation unit 不应该先分析 `A::A()` 函数体，才能知道 A 有没有 move constructor。

因此语言选择：

```text
class declaration
+
base/member type capabilities
```

作为特殊成员函数规则的事实来源。

所以最准确的结论是：

> **特殊成员函数自动生成机制不是“资源检测器”，而是一个基于类声明和对象子结构的语义组合机制。**

---

# 26. Rule of Three / Five / Zero

这些是设计经验，不是“标准要求必须写几个函数”。

## 26.1 Rule of Three

如果一个类需要自定义：

```text
Dtor
CopyCtor
CopyAssign
```

中的一个，往往意味着它的 copy/lifetime semantics 不再是普通 memberwise 默认语义，因此另外两个也值得重新审视。

## 26.2 Rule of Five

C++11 加入 move 后，需要进一步考虑：

```text
MoveCtor
MoveAssign
```

于是资源管理类要审视完整的五项协议。

这不是鼓励“每个类都手写五个”，恰恰相反。

## 26.3 Rule of Zero

更现代的做法是：

```cpp
class Buffer {
    std::vector<char> data;
};
```

或者：

```cpp
class ResourceHolder {
    std::unique_ptr<Resource> p;
};
```

让成员类型直接表达 ownership。

于是外层类的默认对象语义可以自然组合：

```text
vector       → copy + move + destroy
unique_ptr   → no copy + move + destroy
```

外层类型不必重复实现资源协议。

这就是 Rule of Zero。

---

# 27. 多态对象、vptr 与 virtual destructor

讨论对象内存布局时，很容易听到：

```text
“有 virtual function 的对象里一定有一个 vptr。”
```

工程实践中，主流 ABI 确实通常通过 vptr/vtable 实现动态分派。

但必须区分：

```text
C++ language semantics
vs
ABI implementation technique
```

标准要求的是：

> 对 polymorphic object，运行时必须能够实现规定的动态类型/虚函数语义。

标准并不要求所有实现必须以某个固定字段名为 `vptr` 的方式布局对象。

所以教材中可以画典型实现：

```text
Derived object（典型 ABI 示意）
┌─────────────┐
│ vptr ?      │  implementation detail
├─────────────┤
│ Base part   │
├─────────────┤
│ members     │
└─────────────┘
```

但不能把这个图当成 C++ 标准保证的唯一布局。

## 27.1 为什么 polymorphic base 常需要 virtual destructor

```cpp
Base* p = new Derived;
delete p;
```

如果 Base destructor 不是 virtual，而通过 Base* 删除动态类型为 Derived 的对象，会落入未定义行为。

正确的 polymorphic base 通常写：

```cpp
class Base {
public:
    virtual ~Base() = default;
};
```

这样 destruction 可以沿动态类型正确执行：

```text
Derived::~Derived()
↓
Base::~Base()
```

但别忘了：

```cpp
virtual ~Base() = default;
```

仍然是 **user-declared destructor**。

因此会影响 implicit move generation。

这正是对象模型、多态语义与特殊成员规则相互连接的地方。

---

# 28. C++17 Copy elision：对象甚至可能直接在最终 storage 中产生

考虑：

```cpp
A make() {
    return A{};
}

A a = make();
```

不要画成：

```text
临时 A
↓ move
返回值 A
↓ move
a
```

在 C++17 对应的 guaranteed copy-elision / prvalue 语义下，可以把它理解为：

```text
最终对象 a 的 storage
        ↓
A{} 直接在目标位置构造
```

因此：

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

在相应 guaranteed-elision 场景仍可以成立。

这再次说明：

> 学 C++ 对象语义时，应关注“最终有哪些对象真正开始 lifetime”，而不是机械地脑补一串临时对象和 move。

但：

```cpp
A make() {
    A x;
    return x;
}
```

属于 NRVO 场景，应与 `return A{};` 的 C++17 guaranteed prvalue semantics 区分。

---

# 29. 统一推导算法

以后碰到任何特殊成员函数问题，按下面顺序。

## Step 1：画对象结构

```text
complete object T
├── base subobject(s)
└── member subobject(s)
```

先知道外层操作最终必须作用到哪些子对象。

## Step 2：判断类型语义

它更接近：

```text
value type？
unique owner？
shared owner？
non-owning view？
polymorphic base？
```

这帮助判断“默认语义是否符合设计意图”。

## Step 3：列出 user-declared functions

```text
DfltCtor?
OtherCtor?
CopyCtor?
MoveCtor?
CopyAssign?
MoveAssign?
Dtor?
```

`= default` / `= delete` 都算 user-declared。

## Step 4：过第一阶段抑制矩阵

得到：

```text
SELF / S / D / P / ✓
```

## Step 5：检查 base/member 能力

对每个 defaulted/implicit operation 问：

```text
相应子对象 operation 是否存在？
是否 accessible？
是否 deleted？
```

## Step 6：确定 source expression 的 value category

```cpp
a               // lvalue
std::move(a)    // xvalue
A{}             // prvalue
```

以及：

```cpp
A&& x; // x 作为表达式是 lvalue
```

## Step 7：做 overload resolution

候选例如：

```cpp
A(const A&)
A(A&&)
```

确定最佳匹配。

## Step 8：检查 selected function

```text
deleted?
inaccessible?
特殊规则是否使 candidate 被忽略？
```

## Step 9：最后检查 copy elision

如果是：

```cpp
return T{};
T x = T{};
```

先检查 C++17 prvalue / guaranteed copy-elision 语义，不要先数 move 次数。

---

# 30. 常见误区

## 误区 1：对象就是一块内存

不完整。

对象还涉及：

```text
type
lifetime
value
identity
subobject structure
```

## 误区 2：默认 copy 就是 memcpy

错误。

普通非 union class 的 implicit/defaulted copy/move 是逐 base/member 子对象语义。

## 误区 3：`std::move` 会把对象搬到新地址

错误。

它主要改变 expression value category；源对象仍有 identity 和 lifetime。

## 误区 4：`A() = default` 等于什么都没声明

错误。

它是 user-declared default constructor。

## 误区 5：`A(int)` 会抑制 implicit move

错误。

普通 constructor 本身不会；copy/move constructor 有专门规则。

## 误区 6：程序员在 constructor 中 `new`，编译器就知道类是 owner

错误。

ownership 不是从函数体语法推断的。

## 误区 7：写了 destructor 就没有 copy

错误。

C++17 中它抑制 implicit move；implicit copy 仍可能存在，但对应隐式定义属于 deprecated 组合。

## 误区 8：deleted 等于函数不存在

错误。

显式 deleted function 仍参与 overload resolution。

## 误区 9：任何 deleted move 都和 `A(A&&)=delete` 一样

错误。

默认生成失败而 deleted 的 move operation 有专门的 overload-resolution 规则。

## 误区 10：多态对象一定按固定 vptr 布局

错误。

vptr/vtable 是主流 ABI 实现方式，不是标准规定的唯一物理布局。

## 误区 11：`return T{}` 一定调用 move

C++17 下错误。

先检查 guaranteed copy elision。

---

# 31. 学习检查点

完成本章后，应能不用背题解释下面问题。

### 检查点 1

为什么说：

```text
storage ≠ object
```

constructor 在这个区别中承担什么角色？

### 检查点 2

`object representation` 和 `value representation` 有什么区别？padding 为什么会出现？

### 检查点 3

对：

```cpp
struct D : B {
    M1 m1;
    M2 m2;
};
```

为什么 default copy/move/destruct 可以从 B/M1/M2 的能力组合出来？

### 检查点 4

为什么：

```cpp
A() = default;
```

不会因为“是 user-declared constructor”就抑制 implicit move？

### 检查点 5

为什么：

```cpp
A(int);
```

会使 implicit `A()` 不再产生，却不会直接关闭 implicit `A(A&&)`？

### 检查点 6

为什么 constructor 中：

```cpp
p = new int[100];
```

不能作为语言自动判断 ownership 的依据？

### 检查点 7

为什么 default copy/move 不能一般理解成 `memcpy`？

### 检查点 8

解释：

```cpp
A b = std::move(a);
```

为什么 `a` 的 storage、address 和 lifetime 通常仍然存在？真正“被移动”的是什么？

### 检查点 9

解释三者的区别：

```text
move 不存在
explicit move = delete
defaulted move becomes deleted
```

### 检查点 10

为什么：

```cpp
virtual ~Base() = default;
```

既解决 polymorphic destruction contract，又会影响 implicit move generation？

### 检查点 11

为什么：

```cpp
return T{};
```

在 C++17 中应该从“对象直接在最终 storage 中产生”的模型理解，而不是机械地数 copy/move？

如果这些问题能够从对象模型一步步推导出来，再进入选择题验证才有意义。

---

# 32. 参考资料

本文语义基线为 **C++17**。规则核对优先参考：

- C++17 working draft / N4659：`[intro.object]`、`[basic.types]`、`[class.ctor]`、`[class.copy]`、`[class.dtor]`、`[depr.impldec]`
- cppreference：Object、Constructors、Copy constructors、Move constructors、Copy assignment、Move assignment、Destructor、Copy elision、Value categories

后续专题将沿用同一学习方法：

```text
内存 / 对象模型
↓
语言语义
↓
标准规则
↓
工程设计
↓
少量题目验收
```
