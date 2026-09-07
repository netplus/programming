# C++17 特殊成员函数关系：完整抑制矩阵与设计原因

> 语义基线：**C++17**。
>
> 本文的目标不是背“谁关掉谁”，而是建立一个可以自行推导的模型。
>
> 需要始终区分三件事：
>
> 1. **不再隐式声明（not implicitly declared）**；
> 2. **仍然隐式声明，但随后被定义为 deleted**；
> 3. **仍可隐式声明/定义，但这种依赖方式在 C++17 中属于 deprecated 方向**。
>
> 这三种结果在实际调用点的行为完全不同，不能都笼统叫“抑制”。

---

# 1. 六类特殊成员函数

```cpp
class T {
public:
    T();                         // default constructor
    T(const T&);                 // copy constructor
    T(T&&);                      // move constructor

    T& operator=(const T&);      // copy assignment operator
    T& operator=(T&&);           // move assignment operator

    ~T();                        // destructor
};
```

为了讨论方便，后文简称为：

```text
DfltCtor   = default constructor
CopyCtor   = copy constructor
MoveCtor   = move constructor
CopyAssign = copy assignment
MoveAssign = move assignment
Dtor       = destructor
```

注意：

```text
CopyCtor 和 MoveCtor 本身也是 constructor。
```

因此，“只要类存在任意 user-declared constructor，就不会再额外隐式声明 default constructor”这条规则也会被 copy/move constructor 触发。

---

# 2. 先统一表格符号

下面二维表描述的是：

> **当左侧这一项已经被 user-declared 时，顶部目标函数的隐式生成规则会发生什么？**

符号定义：

| 符号 | 含义 |
|---|---|
| `SELF` | 该函数已经由用户声明，不再讨论同一个函数的隐式声明 |
| `S` | **Suppressed**：目标函数根本不会被隐式声明 |
| `D` | **Deleted**：目标函数仍会被隐式声明，但会因为该声明关系被定义为 `deleted` |
| `P` | **Deprecated implicit copy**：目标 copy 仍可隐式声明/默认化，但 C++17 已把这种依赖方式标记为 deprecated 方向 |
| `✓` | 这一行对该目标没有直接的 declaration-level 抑制/删除作用；仍需继续检查 base/member 能力 |

特别注意：

```text
✓ ≠ 最终一定可调用。
```

`✓` 只说明：

```text
当前这一项声明，不会在“第一阶段”直接关闭目标函数。
```

之后还必须检查成员和基类是否支持对应操作。

---

# 3. 核心二维表：所有特殊成员函数之间的直接关系

| 已 user-declared ↓ / 目标 implicit → | DfltCtor | CopyCtor | MoveCtor | CopyAssign | MoveAssign | Dtor |
|---|---:|---:|---:|---:|---:|---:|
| **DfltCtor** | SELF | ✓ | ✓ | ✓ | ✓ | ✓ |
| **CopyCtor** | **S** | SELF | **S** | **P** | **S** | ✓ |
| **MoveCtor** | **S** | **D** | SELF | **D** | **S** | ✓ |
| **CopyAssign** | ✓ | **P** | **S** | SELF | **S** | ✓ |
| **MoveAssign** | ✓ | **D** | **S** | **D** | SELF | ✓ |
| **Dtor** | ✓ | **P** | **S** | **P** | **S** | SELF |

如果只记一张表，就记这一张。

但真正重要的是：**能够解释每一个格子为什么是这个结果。**

---

# 4. 第一类规律：default constructor 只关心“有没有任何用户声明的 constructor”

默认构造函数的核心规则是：

> 只有当类没有任何 user-declared constructor（以及 constructor template）时，编译器才会隐式声明普通无参 default constructor。

因此：

```cpp
class A {
public:
    A(int);
};
```

不会再自动得到：

```cpp
A();
```

同理：

```cpp
class A {
public:
    A(const A&);
};
```

这里 `A(const A&)` 也是 constructor，因此：

```text
implicit A() → 不再产生
```

move constructor 也是一样：

```cpp
class A {
public:
    A(A&&);
};
```

同样不会额外产生 implicit `A()`。

## 为什么这样设计？

default constructor 回答的是：

```text
“允许不提供任何显式初始化参数就创建一个 T 吗？”
```

一旦程序员主动声明任意 constructor，语言认为：

```text
这个类型允许哪些构造入口，已经开始由类作者显式定义。
```

编译器不会再擅自增加一个新的 `T()` API。

这里不是“资源检测”。

例如：

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

即使默认构造内部申请了资源，编译器也不会分析函数体中的 `new` 来推断 copy/move 规则。

特殊成员函数生成规则主要依据：

```text
类声明本身
+
base/member 类型能力
```

而不是函数体语义分析。

---

# 5. 为什么 DfltCtor 不抑制 copy / move / assignment / destructor？

看第一行：

| 已声明 DfltCtor | CopyCtor | MoveCtor | CopyAssign | MoveAssign | Dtor |
|---|---:|---:|---:|---:|---:|
| 结果 | ✓ | ✓ | ✓ | ✓ | ✓ |

例如：

```cpp
class A {
public:
    A() = default;
};
```

这个类仍然可以按各自规则隐式获得：

```cpp
A(const A&);
A(A&&);
A& operator=(const A&);
A& operator=(A&&);
~A();
```

原因不是“默认构造不重要”，而是它描述的是不同维度：

```text
DfltCtor
→ 一个全新的对象怎样开始存在

Copy/Move
→ 已存在对象的状态怎样复制或转移到另一个对象

Assignment
→ 一个已经存在的目标对象怎样接收另一个对象的状态

Dtor
→ 对象怎样结束生命周期
```

声明 `T()` 本身没有声明 copy/move/assignment/destruction 协议，因此语言没有理由关闭那些默认机制。

---

# 6. MoveCtor 为什么最容易被抑制？

隐式 move constructor：

```cpp
T(T&&);
```

只有在类没有 user-declared：

```text
CopyCtor
CopyAssign
MoveAssign
Dtor
```

并且没有自己声明 MoveCtor 时，才会被隐式声明。

因此二维表中：

| 已声明 | implicit MoveCtor |
|---|---:|
| DfltCtor | ✓ |
| CopyCtor | S |
| MoveCtor | SELF |
| CopyAssign | S |
| MoveAssign | S |
| Dtor | S |

## 为什么？

这不是编译器在猜“类是不是拥有资源”。

更准确的理解是：

```text
copy / move / assignment / destruction
```

都属于“已有对象的状态与生命周期协议”。

当程序员主动声明其中任意关键操作时，语言不再自动假定：

```text
memberwise move 一定是类作者想要的移动语义。
```

例如：

```cpp
class Buffer {
public:
    Buffer(const Buffer&);  // 作者已经介入 copy 语义
};
```

此时编译器不会再偷偷增加一个普通 implicit：

```cpp
Buffer(Buffer&&);
```

如果作者希望 move 存在，可以明确写：

```cpp
Buffer(Buffer&&) = default;
```

这时它是：

```text
user-declared + explicitly-defaulted
```

而不是 implicit move。

---

# 7. MoveAssign 与 MoveCtor 为什么关系近似？

隐式 move assignment：

```cpp
T& operator=(T&&);
```

只有在没有 user-declared：

```text
CopyCtor
MoveCtor
CopyAssign
Dtor
```

并且没有自己声明 MoveAssign 时，才会产生。

因此：

| 已声明 | implicit MoveAssign |
|---|---:|
| DfltCtor | ✓ |
| CopyCtor | S |
| MoveCtor | S |
| CopyAssign | S |
| MoveAssign | SELF |
| Dtor | S |

原因仍然是同一套“声明层协议”思想。

一旦类作者已经介入：

```text
如何复制对象
如何移动对象
如何赋值
如何结束生命周期
```

编译器不会再额外假定默认 move-assignment semantics 一定正确。

---

# 8. 为什么 move 声明会让 implicit copy 变成 deleted，而不是直接消失？

这是 copy 和 move 最重要的不对称。

看二维表中的四个 `D`：

```text
MoveCtor   → CopyCtor   = D
MoveCtor   → CopyAssign = D
MoveAssign → CopyCtor   = D
MoveAssign → CopyAssign = D
```

例如：

```cpp
class A {
public:
    A(A&&) = default;
};
```

如果没有显式 copy constructor，语言仍会隐式声明 copy constructor；但它会被定义为：

```text
deleted
```

类似地：

```cpp
class A {
public:
    A& operator=(A&&) = default;
};
```

也会导致 implicit copy constructor / copy assignment 被定义为 deleted。

## 为什么不是“根本不声明 copy”？

从语言模型上，copy constructor / copy assignment 仍属于类的特殊成员函数集合，隐式声明规则本身仍发生；但是显式出现 move operation 表达了一个很强的类型语义：

```text
这个类型已经明确支持“状态转移”。
```

为了避免一个作者明显按 move-oriented 方式设计的类型又意外获得隐式 copy，标准把 implicit copy 定义为 deleted。

结果是：

```cpp
A a;
A b = a;
```

会在编译期明确失败。

这也帮助形成常见的 move-only 类型。

---

# 9. 为什么 CopyCtor / CopyAssign / Dtor 不直接删除 copy，而只是出现 P？

二维表中：

```text
CopyCtor   → CopyAssign = P
CopyAssign → CopyCtor   = P
Dtor       → CopyCtor   = P
Dtor       → CopyAssign = P
```

`P` 的含义是：

```text
copy 仍然可以被隐式声明并 defaulted，
但 C++17 已把这种依赖隐式 copy 的组合列入 deprecated 方向。
```

例如：

```cpp
class A {
public:
    ~A();
};
```

C++17 中，implicit copy constructor 和 copy assignment 并不会因此像 move 那样直接消失。

但是：

```text
“作者自己声明 destructor，
却仍依赖编译器猜 copy semantics”
```

被标准视为一种不值得鼓励的旧式设计模式。

## 为什么不直接在 C++11/C++17 中删掉这种能力？

核心是兼容性。

C++ 在 move semantics 出现以前，大量老代码遵循类似 Rule of Three 的历史模式，并且存在许多“声明 destructor、但仍依赖 implicit copy”的类型。

如果新标准立即让这些 copy 全部消失，会破坏大量既有代码。

因此这里体现的是一种标准演进策略：

```text
保留旧行为
+
标记 deprecated
+
鼓励新代码显式表达 copy/move 设计
```

这也是为什么：

```text
move 的规则更严格，
copy 的历史兼容包袱更重。
```

---

# 10. 为什么 destructor 会抑制 move，但不抑制 default constructor？

二维表中：

| 已声明 Dtor | DfltCtor | CopyCtor | MoveCtor | CopyAssign | MoveAssign |
|---|---:|---:|---:|---:|---:|
| 结果 | ✓ | P | S | P | S |

解释时不要说：

```text
“编译器看见 destructor，就知道你在管理资源。”
```

这不准确。

编译器并不知道 destructor 里面做了什么。

例如：

```cpp
~A() = default;
```

函数体甚至完全由编译器生成，但它仍然是：

```text
user-declared destructor
```

因而照样抑制 implicit MoveCtor / MoveAssign。

真正的规则是：

```text
类作者已经在声明层明确介入 destruction protocol。
```

move semantics 作为另一个状态转移协议，因此不再被自动补齐。

但是 destructor 并没有声明：

```text
“这个类型是否允许无参数构造？”
```

所以它不会抑制 default constructor。

---

# 11. `= default` 与 `= delete` 都算 user-declared

这一点必须和二维表一起使用。

例如：

```cpp
class A {
public:
    ~A() = default;
};
```

`~A() = default` 仍然属于：

```text
user-declared Dtor
```

因此套用 Dtor 那一行：

```text
MoveCtor   → S
MoveAssign → S
CopyCtor   → P
CopyAssign → P
```

同理：

```cpp
A(const A&) = delete;
```

仍然是 user-declared CopyCtor。

因此它同样会：

```text
抑制 implicit MoveCtor
抑制 implicit MoveAssign
```

`= delete` 的含义只是：

```text
这个被声明出来的函数禁止被使用。
```

并不是：

```text
“当作没有声明”。
```

---

# 12. 第一阶段矩阵之外，还有第二阶段：base/member 能力传播

上面的二维表只回答：

> **目标特殊成员函数是否会进入类的声明模型？**

接下来还必须检查：

```text
base/member 是否支持对应操作？
```

这才决定 defaulted/implicit special member 是否最终可用。

可以用下面这张表理解第二阶段。

| 外层函数 | 要求 base/member 具备什么能力 | 常见导致 deleted 的原因 |
|---|---|---|
| DfltCtor | 能完成默认构造 | 成员/基类没有可用 default ctor；引用成员缺少初始化等 |
| CopyCtor | 能复制各 base/member | 某个 base/member 的 copy ctor deleted / inaccessible / ambiguous；特殊成员形态不满足 |
| MoveCtor | 能用对应构造从源子对象建立目标子对象 | 对应构造不可用；相关 destructor 不可访问/已删除 |
| CopyAssign | 能 copy-assign 各 base/member | const 非类成员、reference member、某个 base/member copy assignment 不可用 |
| MoveAssign | 能 move-assign 各 base/member | const 非类成员、reference member、某个 base/member move assignment 不可用 |
| Dtor | 能销毁各 base/member | 某个 base/member destructor deleted / inaccessible 等 |

因此必须使用两阶段模型：

```text
第一阶段：类声明关系
        ↓
目标函数是否 implicitly-declared？
        ↓
第二阶段：base/member 能力
        ↓
目标 implicit/defaulted 函数是否 usable / deleted？
```

---

# 13. “根本没有 move”与“move 被 deleted”为什么必须区分？

## 情况 A：move 根本没有被隐式声明

```cpp
class A {
public:
    A(const A&);
};
```

CopyCtor 会抑制 implicit MoveCtor。

因此：

```cpp
A a;
A b = std::move(a);
```

候选中没有普通 implicit `A(A&&)`。

如果：

```cpp
A(const A&)
```

可用，因为 `const A&` 可以绑定右值，最终可能发生：

```text
copy
```

---

## 情况 B：程序员显式声明 deleted move

```cpp
class A {
public:
    A(const A&);
    A(A&&) = delete;
};
```

这里 MoveCtor 存在，而且进入 overload resolution。

对：

```cpp
std::move(a)
```

`A(A&&)` 通常是更好的匹配。

于是：

```text
选中 MoveCtor
↓
发现 deleted
↓
编译失败
```

不会自动退回 copy。

---

## 情况 C：defaulted move 因 base/member 原因被定义为 deleted

例如外层默认 move 无法生成。

C++ 对 defaulted move 有一个专门规则：

```text
如果 defaulted MoveCtor / MoveAssign 因生成条件失败而被 defined as deleted，
它会被 overload resolution 忽略。
```

这样做是为了让某些右值场景仍有机会由 copy 接管。

因此：

```text
explicit T(T&&) = delete
```

和：

```text
defaulted T(T&&) 最终 defined as deleted
```

不能混为一谈。

---

# 14. 为什么语言采用这种“声明驱动”规则，而不是分析函数体？

你的一个非常重要的反例是：

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

这里显然发生了资源获取，但它不会抑制 implicit copy/move。

原因是：

> **C++ 的特殊成员规则不是资源检测机制。**

编译器无法可靠地通过函数体判断 ownership：

```cpp
int fd;
T* p;
uint64_t handle;
```

它们可能代表：

```text
独占资源
共享资源
borrowed handle
普通数值
外部系统对象
```

单看语法无法区分。

而且 C++ 支持 separate compilation：

```cpp
// A.h
class A {
public:
    A();
};
```

构造函数实现可以在另一个 `.cpp` 中。

其他翻译单元不应为了判断：

```text
A 是否有 move constructor
```

而必须先分析 `A::A()` 的函数体。

因此 C++ 选择一个机械、局部、可预测的模型：

```text
看 class declaration
        ↓
决定哪些 special member 被声明
        ↓
看 base/member 类型能力
        ↓
决定 defaulted function 能否成立
```

这也是现代 C++ 鼓励用：

```cpp
std::unique_ptr
std::shared_ptr
std::vector
std::string
std::fstream
```

来表达 ownership 的原因。

所有权尽量编码在**类型**里，而不是隐藏在裸指针和函数体实现中。

---

# 15. 从设计目的重新理解每一行

## 15.1 DfltCtor 行

```text
只声明“对象如何从无到有”。
```

因此：

```text
不直接改变 copy/move/assignment/destruction 的 implicit 规则。
```

---

## 15.2 CopyCtor 行

```text
作者已经明确介入“如何复制构造一个新对象”。
```

因此：

```text
MoveCtor   → 不再隐式补
MoveAssign → 不再隐式补
```

而 CopyAssign 因历史兼容仍可隐式存在，但进入 deprecated 关系。

---

## 15.3 MoveCtor 行

```text
作者明确表达 move construction semantics。
```

因此：

```text
implicit copy operations → defined as deleted
implicit MoveAssign      → 不再自动补
```

同时因为 MoveCtor 是 constructor：

```text
implicit DfltCtor → 不再产生
```

---

## 15.4 CopyAssign 行

```text
作者已经介入“已有目标对象如何接受复制状态”。
```

因此：

```text
implicit MoveCtor   → 不再产生
implicit MoveAssign → 不再产生
```

CopyCtor 仍可存在，但属于 deprecated implicit-copy 组合。

---

## 15.5 MoveAssign 行

```text
作者明确表达 move assignment semantics。
```

因此：

```text
implicit copy operations → defined as deleted
implicit MoveCtor         → 不再产生
```

---

## 15.6 Dtor 行

```text
作者明确介入对象结束生命周期的协议。
```

因此语言不再自动补 move：

```text
MoveCtor   → S
MoveAssign → S
```

但为了历史兼容，copy 仍保留 implicit 路径，只是进入 deprecated 关系。

Dtor 与“是否允许无参数构造”没有直接声明层联系，因此：

```text
DfltCtor → ✓
```

---

# 16. 一张更容易记忆的压缩图

先记 default constructor：

```text
任何 user-declared constructor
        ↓
implicit DfltCtor 不再产生
```

然后记 move：

```text
CopyCtor
CopyAssign
MoveAssign
Dtor
        ↓
implicit MoveCtor 不再产生
```

以及：

```text
CopyCtor
MoveCtor
CopyAssign
Dtor
        ↓
implicit MoveAssign 不再产生
```

再记 copy：

```text
MoveCtor 或 MoveAssign
        ↓
implicit CopyCtor / CopyAssign
        ↓
defined as deleted
```

最后记历史兼容：

```text
CopyCtor ↔ CopyAssign
Dtor → CopyCtor / CopyAssign
        ↓
copy 仍可能存在
但属于 deprecated implicit-copy 方向
```

---

# 17. 最终统一推导流程

以后遇到任何 special member 题目，不要直接问：

```text
“有没有 move？”
```

按照下面四步推导。

## 第一步：列出 user-declared 特殊成员函数

包括：

```text
= default
= delete
```

它们都仍然算 user-declared。

---

## 第二步：查二维矩阵

先确定：

```text
目标函数是否根本不 implicit-declare？
是否 implicit-declare 但直接成为 deleted？
是否仍可生成但属于 deprecated copy 组合？
```

---

## 第三步：检查 base/member

```text
对应 default/copy/move/assign/destruct 操作能否成立？
```

若不能，defaulted special member 还可能进一步被定义为 deleted。

---

## 第四步：到具体调用点做 overload resolution

例如：

```cpp
A b = std::move(a);
```

继续问：

```text
MoveCtor 是否存在？
MoveCtor 是否 usable？
CopyCtor 是否存在？
哪些候选参加重载决议？
最佳匹配是谁？
```

只有走到这一步，才能确定：

```text
copy / move / compile error
```

---

# 18. 本专题真正要形成的心智模型

不要把规则背成：

```text
“写了 destructor，所以没 move。”
```

应该理解成：

```text
C++ 不分析函数体中的资源语义
        ↓
只根据 class declaration 建立对象语义协议
        ↓
不同 special member 有不同 implicit-declaration 条件
        ↓
move 的自动生成条件最保守
        ↓
copy 保留更多历史兼容路径
        ↓
base/member 再决定 defaulted operation 能否真正成立
        ↓
overload resolution 最后决定具体调用谁
```

这才是特殊成员函数规则的统一模型。

---

# 19. 规范依据（C++17）

后续若需要核对标准措辞，重点看：

```text
[class.default.ctor]
[class.copy.ctor]
[class.copy.assign]
[class.dtor]
[depr.impldec]
```

尤其注意标准措辞中的：

```text
user-declared
implicitly-declared
defaulted
defined as deleted
deprecated
```

这些术语不能互换。
