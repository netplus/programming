# C++ 继承、多态、重载与虚函数：从对象模型到编译器实现

> **语义基线：C++17**  
> 本文是本仓库关于继承、重载、运行时多态和虚函数的**唯一主教材**。  
> 学习方法与 `object-lifetime-and-special-member-functions.md` 保持一致：**先建立对象模型，再解释语言规则，最后进入编译器/ABI 实现。**  
> 学习目标不是记忆“哪个函数最后被调用”，而是能够从对象关系、静态类型、名字查找、重载决议、动态类型和生命周期自行推出程序行为。

---

## 总体学习图

```text
继承关系
    ↓
对象中形成 base subobject
    ↓
Derived → Base 的对象 / 指针 / 引用关系
    ↓
static type
    ↓
name lookup
    ↓
overload resolution
    ↓
选定一个具体函数接口
    ↓
如果它不是 virtual ───────────────→ 静态调用结束
    ↓
如果它是 virtual
    ↓
most-derived object / dynamic type
    ↓
final overrider
    ↓
运行时实现
    ↓
典型 ABI：vptr / vtable / slot / thunk / offset metadata
```

整篇教材最重要的原则是：

> **名字查找、重载决议、override 关系、virtual dispatch、对象布局，是不同层次的机制。**

复杂题目之所以难，通常不是因为某一条规则特别复杂，而是因为多个层次同时出现。

---

# 第一部分：继承首先是对象模型

# 1. 为什么从继承对象模型开始

学习多态时最常见的错误是直接从 `virtual` 开始。

实际上，在 C++ 中：

```cpp
struct Base {
    int b;
};

struct Derived : Base {
    int d;
};
```

`Derived` 对象之所以能被当成 `Base` 使用，不是因为编译器“假装”它是 Base，而是因为：

> **一个 Derived complete object 中真实包含一个 Base base-class subobject。**

概念结构：

```text
Derived complete object
┌──────────────────────────────┐
│ Base subobject               │
│   Base::b                    │
├──────────────────────────────┤
│ Derived own part             │
│   Derived::d                 │
└──────────────────────────────┘
```

所以后面的：

```text
upcast
object slicing
static type / dynamic type
virtual dispatch
multiple inheritance
virtual inheritance
```

全部建立在 **subobject** 模型之上。

---

# 2. complete object、base subobject 与 most-derived object

考虑：

```cpp
Derived obj;
```

`obj` 是一个：

```text
complete object
```

其中的 `Base` 部分是：

```text
base-class subobject
```

对这个完整对象来说：

```text
most-derived class  = Derived
most-derived object = obj
```

这几个概念一定要分清：

```text
Derived complete object
│
├── Base subobject
│
└── Derived own subobjects
```

当我们写：

```cpp
Base* p = &obj;
```

更准确的对象语义是：

> `p` 指向 `obj` 中的 Base subobject；这个 Base subobject 仍然属于一个 most-derived type 为 Derived 的完整对象。

这句话就是运行时多态的对象基础。

---

# 3. 继承访问方式：public / protected / private 到底改变什么

C++ 的继承不只有：

```cpp
Derived : public Base
```

还有：

```cpp
Derived : protected Base
Derived : private Base
```

它们首先影响的是 **base class 的可访问性**。

工程语义上，public inheritance 最适合表达：

```text
Derived is-a Base
```

也就是说外部代码可以把 Derived 当作 Base 接口使用。

例如：

```cpp
struct Base {};
struct Derived : public Base {};

Derived d;
Base* p = &d;       // OK
Base& r = d;        // OK
```

而：

```cpp
class Derived : private Base {};
```

对普通外部调用者来说，`Base` 通常不是可访问基类，因此：

```cpp
Derived d;
Base* p = &d;       // 外部上下文中通常 ill-formed
```

所以不要把“Derived 一定能隐式转成 Base”背成无条件规则。

正确条件至少要问：

```text
Base 是否是可访问的？
Base 是否是无歧义的？
```

---

# 4. Derived → Base：三类常见关系必须分开

教材里经常把下面三种情况统称为“赋值兼容”：

```cpp
Base b = derived;
Base* p = &derived;
Base& r = derived;
```

但它们在 C++ 里不是同一种机制。

必须分别理解。

---

## 4.1 Derived object → Base object：产生新的 Base 对象

```cpp
Derived d;
Base b = d;
```

这里不是“让 b 指向 d 的 Base 部分”。

而是：

```text
从 d 中的 Base subobject
        ↓ copy construction
创建一个全新的 Base complete object b
```

概念图：

```text
d
┌─────────────────────┐
│ Base subobject      │ ───── copy ─────┐
├─────────────────────┤                  │
│ Derived part        │                  ▼
└─────────────────────┘           ┌───────────────┐
                                  │ Base object b │
                                  └───────────────┘
```

如果使用普通 copy construction，可以近似理解为：

```cpp
Base::Base(const Base&);
```

其实参绑定到 `d` 中对应的 Base subobject。

Derived 自己新增的状态不会进入 `b`。

这就是：

```text
object slicing
```

---

## 4.2 assignment 与 construction 仍然不是一回事

```cpp
Base b;
Derived d;

b = d;
```

这调用的是 assignment 语义：

```cpp
Base::operator=(const Base&)
```

目标 `b` 已经存在。

所以要严格区分：

```cpp
Base b = d;   // 构造新的 Base complete object
b = d;        // 修改已有 Base complete object
```

两者都会只处理 Base 语义，但生命周期操作不同。

---

## 4.3 Derived* → Base*：指向 Base subobject

```cpp
Derived d;
Derived* pd = &d;
Base* pb = pd;
```

这里没有创建新对象。

标准语义是：

> 转换后的 `pb` 指向 Derived complete object 中正确的 Base subobject。

简单单继承下，Base subobject 常常位于对象开头：

```text
Derived
address + 0
┌───────────────────┐
│ Base              │ ◀── pb
├───────────────────┤
│ Derived           │
└───────────────────┘
▲
pd
```

这种情况下机器地址值甚至可能不需要改变。

但这不是语言保证。

多继承时可能必须做地址调整，后文详讲。

---

## 4.4 Derived object → Base&：引用绑定 Base subobject

```cpp
Derived d;
Base& r = d;
```

同样没有创建新 Base 对象。

概念上：

```text
Derived d
┌─────────────────────┐
│ Base subobject      │ ◀── r
├─────────────────────┤
│ Derived part        │
└─────────────────────┘
```

因此：

```text
没有 slicing
对象 identity 没有丢失
most-derived object 仍然是 Derived
```

这就是为什么 Base 引用仍然可以参与 virtual dispatch。

---

# 5. 为什么 upcast 可以隐式，而 downcast 通常不可以

向上转换：

```text
Derived → Base
```

安全的根本原因是：

> 每个有效的 Derived complete object 必然包含对应的 Base subobject。

但是：

```cpp
Base* p;
Derived* d = p;       // 不允许普通隐式转换
```

仅凭 `Base*` 无法知道它背后的完整对象是：

```text
Base
Derived
OtherDerived
```

所以反向转换不是静态必然安全的。

C++ 提供不同工具表达不同责任：

```text
static_cast
→ 程序员依据静态关系承担正确性责任

dynamic_cast
→ 请求运行时验证多态对象关系
```

尤其要注意：从 virtual base 反向做某些 `static_cast` 是语言直接禁止的，后文会解释原因。

---

# 6. static type 与 dynamic type

考虑：

```cpp
Derived d;
Base* p = &d;
Base& r = d;
```

对于：

```cpp
*p
```

可以建立：

```text
static type  = Base
dynamic type = Derived
```

对于 `r` 同理。

## 6.1 static type 主要控制什么

静态类型在编译期已知，主要影响：

```text
name lookup
可见成员
候选 overload set
overload resolution
默认参数
非虚调用
静态转换是否合法
```

## 6.2 dynamic type 主要参与什么

动态类型主要参与运行时机制：

```text
virtual dispatch
dynamic_cast
typeid（多态 glvalue）
```

所以必须牢牢记住：

> **运行时对象实际是 Derived，不会让普通 name lookup 或 overload resolution 突然看到 Derived 的新成员。**

---

# 第二部分：多继承、菱形继承与虚继承

# 7. 多继承首先意味着多个 base subobject

```cpp
struct B1 {
    int x;
};

struct B2 {
    int y;
};

struct D : B1, B2 {
    int z;
};
```

概念布局可以是：

```text
D complete object
┌────────────────────┐
│ B1 subobject       │
├────────────────────┤
│ B2 subobject       │
├────────────────────┤
│ D own part         │
└────────────────────┘
```

因此：

```cpp
D* pd;
B1* p1 = pd;
B2* p2 = pd;
```

`p1` 和 `p2` 指向的是不同 base subobject。

典型 ABI 下：

```text
D* → B1*   可能 offset 0
D* → B2*   可能需要 +N
```

所以 `Derived* -> Base*` 的标准语义是“定位正确 subobject”，而不是“机器地址保持不变”。

---

# 8. 普通菱形继承为什么会出现两个共同 Base

经典菱形：

```cpp
struct A {
    int a;
};

struct B1 : A {};
struct B2 : A {};

struct D : B1, B2 {};
```

继承图：

```text
        A
       / \
     B1   B2
       \ /
        D
```

但如果两条边都是普通非虚继承，`D` 中不是“自动合并成一个 A”。

对象图实际上是：

```text
D
├── B1 subobject
│   └── A subobject #1
│
└── B2 subobject
    └── A subobject #2
```

C++17 对非虚基类的基本对象模型就是：

> class lattice 中每一个不同的 non-virtual base occurrence，对应一个独立的 base subobject。

因此：

```cpp
D d;
A* p = &d;       // ambiguous
```

编译器不知道你要：

```text
B1::A
还是
B2::A
```

可以显式指定：

```cpp
A* p1 = static_cast<B1*>(&d);
A* p2 = static_cast<B2*>(&d);
```

---

# 9. 普通菱形的两个问题要分开

普通菱形经常同时产生两类问题。

## 9.1 状态重复

如果 A 有数据：

```cpp
struct A {
    int state;
};
```

那么 D 中存在两个独立：

```text
A::state
```

即：

```text
B1 路径一份
B2 路径一份
```

这有时正是想要的语义，有时则不是。

## 9.2 base conversion ambiguity

```cpp
A* p = &d;
```

有两条候选 A subobject 路径，因此无法唯一定位。

这和：

```text
member name lookup ambiguity
```

仍然不是同一概念。

后面会统一分类。

---

# 10. virtual inheritance：共享一个 virtual base subobject

如果设计语义要求整个最派生对象只拥有一个共同 A：

```cpp
struct A {};

struct B1 : virtual A {};
struct B2 : virtual A {};

struct D : B1, B2 {};
```

对象模型变成：

```text
D complete object
├── B1 subobject ──┐
├── B2 subobject ──┼──► shared A virtual-base subobject
└── D own part     │
                   └── exactly one A in D
```

C++17 对 virtual base 的核心保证是：

> 对每一个 distinct virtual base class，most-derived object 中只有一个对应的 base-class subobject。

因此 virtual inheritance 解决的是：

```text
多条继承路径共享共同 base state
```

它和 virtual function 是完全不同的机制。

---

# 11. virtual function 与 virtual inheritance 不要混淆

| 维度 | virtual function | virtual inheritance |
|---|---|---|
| 核心问题 | 动态行为替换 | 共同基类子对象共享 |
| 作用对象 | member function | inheritance graph |
| 核心语言概念 | final overrider | virtual base subobject |
| 常见 ABI 信息 | virtual function slot | virtual-base offset |
| 是否要求动态调用 | 是 | 不一定 |

同一个关键字 `virtual` 只是历史语法复用。

语义上二者必须分开学习。

---

# 12. virtual base 为什么由 most-derived class 负责构造

考虑：

```cpp
struct A {
    explicit A(int);
};

struct B1 : virtual A {
    B1() : A(1) {}
};

struct B2 : virtual A {
    B2() : A(2) {}
};

struct D : B1, B2 {
    D() : A(3), B1(), B2() {}
};
```

构造：

```cpp
D d;
```

整个 D 中只有一个 A virtual-base subobject。

如果 B1 和 B2 都真正构造一次 A，就会产生两次构造，与“共享一个 A”矛盾。

因此 C++ 规定：

> **virtual base 由 most-derived class 的 constructor 负责初始化。**

所以构造 `D` 时：

```text
D 是 most-derived class
        ↓
D::D() 中的 A(3) 生效
        ↓
B1() 中 A(1) 的 mem-initializer 被忽略
B2() 中 A(2) 的 mem-initializer 被忽略
```

但如果单独构造：

```cpp
B1 b;
```

此时 `B1` 自己就是 most-derived class，所以：

```cpp
B1() : A(1)
```

中的 A(1) 又会生效。

这不是特殊补丁，而是从：

```text
一个 complete object 中只有一个 shared virtual base
```

自然推出来的责任模型。

---

# 13. virtual base 的精确构造顺序

在非 delegating constructor 中，C++17 的顺序是：

```text
1. virtual base classes
   仅由 most-derived constructor 初始化
   按 inheritance graph 的 depth-first、left-to-right 顺序

2. direct non-virtual base classes
   按 base-specifier-list 中的声明顺序

3. non-static data members
   按类中声明顺序

4. constructor body
```

注意：

```text
不是 mem-initializer list 的书写顺序
```

例如：

```cpp
D::D()
    : member(...), B2(...), A(...), B1(...)
{}
```

也不会改变语言规定的真实初始化顺序。

---

# 14. virtual base 的析构责任

析构是构造的逆向生命周期协议。

对于最派生对象 D：

```text
D destructor body
    ↓
D members
    ↓
D 的 non-virtual direct bases
    ↓
shared virtual bases
```

关键点：

> **virtual base 也只由 most-derived object 的完整析构过程销毁一次。**

所以菱形虚继承中不会出现：

```text
B1 析构 A 一次
B2 又析构 A 一次
```

这也是底层 ABI 经常需要区分“作为完整对象析构”和“作为 base subobject 析构”的原因之一。

---

# 15. copy / move 与 virtual base

virtual inheritance 还会影响 special member semantics。

## 15.1 copy/move constructor

C++17 规定 defaulted copy/move constructor 对非 union class 执行逐 base/member 的 copy/move。

对 virtual base：

> **由 implicitly-defined copy/move constructor 只初始化一次 virtual base subobject。**

这与普通构造的“shared virtual base 只有一个”保持一致。

## 15.2 copy/move assignment 有一个容易忽略的边界

defaulted copy/move assignment 对直接 base 和成员执行 memberwise assignment。

但在菱形虚继承中：

> **C++17 对 shared virtual base 是否会被 implicitly-defined copy/move assignment 赋值超过一次，规定为 unspecified。**

因此如果 virtual base 的 assignment 有可观察副作用，不应该依赖“恰好调用一次”的假设。

这是一个很典型的高级语言边界。

---

# 16. virtual base 为什么让地址计算更复杂

普通非虚 base 在一个具体 derived type 里的位置通常可以由编译器静态知道。

例如：

```text
D
├── B1 offset 0
└── B2 offset 16
```

则：

```text
D* → B2*
```

机器层可以近似：

```text
pointer + 16
```

但 virtual base 的相对位置取决于**最终的 most-derived layout**。

例如同一个：

```cpp
struct B : virtual A {};
```

B 作为：

```text
complete object
某个 D1 的 subobject
某个 D2 的 subobject
```

时，shared A 相对 B 的位置不必统一为同一个固定常量。

所以主流 ABI 可能需要：

```text
vptr / vbptr 等对象状态
        ↓
相关 table metadata
        ↓
virtual-base offset
        ↓
定位 shared A subobject
```

具体组织方式取决于 ABI；标准只保证最终定位正确。

---

# 第三部分：静态多态——名字查找与重载

# 17. 两种“多态”必须先分开

C++ 中常说：

```text
static polymorphism
runtime polymorphism
```

静态多态常见于：

```text
function overload
operator overload
templates
```

运行时多态主要是：

```text
virtual function + dynamic dispatch
```

二者最关键区别：

```text
overload
→ 编译期选择“哪一个函数接口”

virtual dispatch
→ 静态接口已经选定后，运行时选择“该接口的哪个 override 实现”
```

---

# 18. 一次成员函数调用不是一步完成的

对于：

```cpp
obj.f(arg);
```

推荐固定拆成：

```text
阶段 1：确定 obj 表达式的 static type
        ↓
阶段 2：name lookup
        ↓
阶段 3：形成 candidate functions
        ↓
阶段 4：筛 viable functions
        ↓
阶段 5：比较 implicit conversion sequences
        ↓
阶段 6：得到 best viable function
        ↓
阶段 7：检查 deleted / access 等最终可用性
        ↓
阶段 8：补充需要在调用点决定的 default arguments
        ↓
阶段 9：如果选中函数是 virtual 且未显式限定
        ↓
阶段 10：根据 dynamic type / most-derived object 找 final overrider
```

因此：

> **name lookup、overload resolution、virtual dispatch 绝对不能合成一个“找函数”步骤。**

---

# 19. name lookup：先决定“哪些名字看得见”

```cpp
void f(int);
void f(double);

f(1);
```

名字查找先找到：

```text
f(int)
f(double)
```

然后才进入 overload resolution。

继承里这一点更加重要：

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

很多人会直接比较：

```text
int → int
int → double
```

然后认为 Base::f(int) 更优。

但错误发生在更前面。

成员 lookup 在 Derived 中已经找到：

```cpp
Derived::f(double)
```

因此 Base 中同名 `f` 被隐藏。

Base::f(int) 根本没进入这次 overload set。

---

# 20. overload 到底是什么

函数重载表达：

> 同一个作用域中使用相同函数名，根据参数类型和调用上下文在编译期选择不同函数。

例如：

```cpp
void print(int);
void print(double);
void print(const char*);
```

调用：

```cpp
print(1);
print(1.0);
print("x");
```

目标在编译期已经确定。

所以 overload 不需要：

```text
vptr
vtable
dynamic type
```

---

# 21. 哪些差异能构成 overload

典型可区分维度包括：

```text
参数数量
参数类型
参数顺序
成员函数 cv qualifier
成员函数 ref qualifier
```

例如：

```cpp
struct A {
    void f();
    void f() const;

    void g() &;
    void g() &&;
};
```

成员函数末尾的 `const` / ref qualifier 会影响隐式对象参数，因此可以形成成员函数重载。

---

# 22. 不能仅靠返回类型重载

```cpp
int f();
double f();     // ill-formed
```

调用：

```cpp
f();
```

仅从调用表达式无法由返回类型唯一选择函数，因此 C++ 不允许只靠 return type 建立普通 overload。

协变返回属于 override 的特殊规则，不是这里的 overload 规则。

---

# 23. 顶层 const 与参数类型

例如：

```cpp
void f(int);
void f(const int);
```

对按值参数来说，顶层 const 不构成两个不同的可重载函数接口。

但：

```cpp
void f(int&);
void f(const int&);
```

是不同的引用参数类型，可以参与重载。

这类问题的关键是：

```text
const 修饰的是传入的独立值对象？
还是引用/指针所指向的对象类型？
```

---

# 24. 成员函数的隐式对象参数

可以把：

```cpp
struct A {
    void f();
    void f() const;
};
```

建立近似心智模型：

```text
f()       需要 non-const A object
f() const 需要 const-compatible A object
```

所以：

```cpp
A a;
const A ca;

a.f();      // non-const 版本通常更匹配
ca.f();     // 只能选择 const-compatible 版本
```

这比机械背“成员函数后面 const 参与重载”更容易推广。

---

# 25. candidate、viable、best viable

不要把所有同名函数直接看成“都在竞争”。

更准确的过程是：

```text
lookup
   ↓
candidate set
   ↓
参数数量 / default arguments / reference binding / conversions 等检查
   ↓
viable functions
   ↓
比较 implicit conversion sequence
   ↓
best viable function
```

如果没有唯一最佳：

```text
ambiguous
```

如果没有 viable：

```text
no matching function
```

---

# 26. implicit conversion sequence 的基本层次

对普通重载学习，先建立最常见的粗粒度顺序：

```text
exact match
    优于
promotion
    优于
conversion
    优于
user-defined conversion
    优于
ellipsis
```

例如：

```cpp
void f(int);
void f(long);

short x = 1;
f(x);
```

通常：

```text
short → int   integral promotion
short → long  integral conversion
```

所以 `f(int)` 更好。

但要注意：

```text
两个转换都合法
```

不代表一定能选出唯一更优者。

---

# 27. `nullptr` 与 overload ambiguity

```cpp
void f(int*);
void f(double*);

f(nullptr);
```

`nullptr` 类型是：

```cpp
std::nullptr_t
```

它可以进行标准空指针转换到不同 object pointer 类型。

这里：

```text
nullptr → int*
nullptr → double*
```

两边没有唯一更优，所以调用 ambiguous。

对比：

```cpp
void f(int);
void f(int*);

f(nullptr);    // 选择 f(int*)
```

不要再把 `nullptr` 当成旧式整数常量 `0` 来推理。

---

# 28. dynamic type 不参与 overload resolution

```cpp
struct Base {};
struct Derived : Base {};

void g(Base&);
void g(Derived&);

Derived d;
Base& r = d;

g(r);
```

这里：

```text
r 的 static type = Base
r 引用的 dynamic object = Derived
```

但普通 overload resolution 选择：

```cpp
g(Base&)
```

不会因为运行时对象是 Derived 就重新选择 `g(Derived&)`。

所以：

> **dynamic type 不是 overload resolution 的输入。**

---

# 29. inheritance 中的 name hiding

```cpp
struct Base {
    void f(int);
    void f(const char*);
};

struct Derived : Base {
    void f(double);
};
```

Derived 声明同名 `f` 后，普通成员名字查找会把 Base 中同名 overloads 隐藏在当前 lookup 之外。

注意：

```text
hiding 按名字发生
```

不要求参数列表相同。

所以：

```cpp
Derived d;
d.f(1);
```

可能选择：

```cpp
Derived::f(double)
```

即使 Base::f(int) 从参数匹配角度“看起来更好”。

因为它在更早的 name lookup 阶段已经被隐藏。

---

# 30. `using Base::f` 改变的是 lookup set

```cpp
struct Derived : Base {
    using Base::f;
    void f(double);
};
```

现在 Derived 成员 lookup 可以把 Base 的 `f` 重新带进可见集合。

于是：

```cpp
Derived d;
d.f(1);
```

overload resolution 可以比较：

```text
Base::f(int)
Derived::f(double)
```

最终 exact match 的 Base::f(int) 获胜。

所以：

> `using Base::f` 改变 name lookup / overload set；它不负责建立 override，也不负责开启 virtual dispatch。

---

# 第四部分：运行时多态与 virtual function

# 31. virtual function 解决的到底是什么问题

考虑接口：

```cpp
struct Shape {
    virtual double area() const = 0;
};
```

我们希望：

```text
调用者只依赖 Shape interface
        ↓
Circle object    → Circle::area
Rectangle object → Rectangle::area
```

因此 virtual function 解决的是：

> **编译期已经选定某个 Base 接口后，运行时允许 most-derived object 提供该接口的最终实现。**

这就是 runtime polymorphism。

---

# 32. 什么函数能 virtual，什么函数不能

核心直觉是：

> virtual dispatch 需要一个具体对象实例，以及它的 dynamic type / most-derived object。

因此：

| 函数类别 | 能否 virtual | 原因 |
|---|---:|---|
| 普通非静态成员函数 | 可以 | 有对象实例 |
| destructor | 可以 | 属于对象生命周期协议 |
| conversion function | 可以 | 非静态成员行为 |
| 某些 member operator | 可以 | 本质仍是非静态成员函数 |
| constructor | 不可以 | 对象/动态类型尚在建立 |
| static member function | 不可以 | 没有对象实例 / `this` |
| free function | 不可以 | 不属于 class object dispatch |
| member function template | 不可以作为 virtual function | virtual interface 需要确定成员函数槽位，而模板实例集合开放 |

---

# 33. 为什么 constructor 不能 virtual

普通虚调用逻辑依赖：

```text
已有对象
    ↓
找到当前 base subobject
    ↓
知道 most-derived object
    ↓
确定 final overrider
```

但 constructor 的职责恰恰是：

```text
建立对象及其 subobjects
```

如果 constructor 自身依赖“已经存在的 dynamic type”来决定调用谁，就会形成循环依赖。

C++ 采用的是：

```text
调用点静态决定要构造 Derived
        ↓
准备 Derived storage
        ↓
依次建立 virtual bases / direct bases / members / Derived body
```

如果需要运行时选择创建哪种派生对象，工程上通常使用：

```text
factory
virtual clone
registration / creator table
```

而不是 virtual constructor。

---

# 34. 为什么 destructor 可以并且常常应该 virtual

对象销毁时情况完全不同。

对象已经存在，运行时可以知道：

```text
Base* 静态视图
实际 most-derived object = Derived
```

所以：

```cpp
Base* p = new Derived;
delete p;
```

如果 Base destructor 是 virtual，语言就能先动态选择正确的最派生析构入口。

这让：

> **“通过基类接口销毁完整动态对象”成为 Base 类型协议的一部分。**

如果 Base 设计上允许通过 Base* 多态删除，通常应该：

```cpp
virtual ~Base() = default;
```

---

# 35. override 建立的不是普通 overload 关系

```cpp
struct Base {
    virtual void f(int);
};

struct Derived : Base {
    void f(int) override;
};
```

这里建立的是：

```text
Base::f(int) virtual interface
        ↑ overridden by
Derived::f(int)
```

不是“运行时拿两个同名函数重新 overload resolution”。

运行时只沿已经确定的 virtual interface 找 final overrider。

---

# 36. override 的精确匹配条件

C++17 中，派生函数 override 基类 virtual function 的核心条件包括：

```text
同名
相同 parameter-type-list
相同 cv-qualification
相同 ref-qualifier（或都无）
```

返回类型要求：

```text
相同
或满足特定 class pointer/reference 的 covariant return
```

因此：

```cpp
struct Base {
    virtual void f() const;
};

struct Derived : Base {
    void f();             // 不 override
};
```

以及：

```cpp
struct Base {
    virtual void g() &;
};

struct Derived : Base {
    void g() &&;          // 不 override
};
```

都不是 override。

显式写：

```cpp
void f() override;
```

可以让编译器替你验证这条关系。

---

# 37. `virtual`、`override`、`final` 的职责

## 37.1 `virtual`

首次声明一个动态派发接口：

```cpp
virtual void f();
```

一旦某个函数沿继承链成为 virtual，正确 override 的后续函数即使不重复写 `virtual`，也仍然是 virtual。

## 37.2 `override`

表示：

```text
“我期望这个函数覆盖某个 base virtual function，请编译器检查。”
```

强烈建议所有 override 都显式写。

## 37.3 `final`

函数级：

```cpp
void f() final;
```

禁止更派生类继续 override。

类级：

```cpp
struct D final : Base {};
```

禁止再继承 D。

---

# 38. visibility / access 与 override 是不同机制

一个 virtual function 即使：

```text
在普通 lookup 中被隐藏
或
是 private
```

仍然可能被正确 override。

例如：

```cpp
struct Base {
private:
    virtual void hook();
};

struct Derived : Base {
    void hook() override;     // 可以
};
```

`private` 控制的是访问权限，不决定 override relation 是否存在。

这也是 NVI 模式能够成立的基础。

---

# 39. covariant return

```cpp
struct Base {
    virtual Base* clone();
};

struct Derived : Base {
    Derived* clone() override;
};
```

这种返回类型变化合法，因为满足 class pointer/reference 的协变规则。

核心直觉：

```text
Derived* 本身可以转换成 Base*
```

所以 Base contract 没有被破坏。

但：

```cpp
virtual int f();
double f() override;     // 不允许
```

普通可转换数值类型不构成协变返回。

---

# 40. virtual dispatch：静态阶段结束后才开始

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

完整流程：

```text
1. p 的 static type = Base*

2. 对 Base 做 member name lookup
   → Base::f(int)
   → Base::f(double)

3. overload resolution
   实参 1 是 int
   → 选择 Base::f(int)

4. 检查 selected function
   Base::f(int) 是 virtual

5. *p 所属 most-derived object = Derived

6. 找 Base::f(int) 在 Derived object 中的 final overrider
   → Derived::f(int)

7. 执行 Derived::f(int)
```

最关键结论：

> **virtual dispatch 不会重新进行 overload resolution。**

---

# 41. final overrider

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

对于：

```cpp
C c;
A& r = c;
r.f();
```

静态阶段选中：

```text
A::f 这条 virtual interface
```

在 C most-derived object 中，它的 final overrider 是：

```cpp
C::f
```

所以动态派发到 C::f。

`final overrider` 是标准层概念，比“查 vtable 最后一个函数”更准确。

---

# 42. 显式 qualified call 为什么关闭 virtual dispatch

```cpp
struct Derived : Base {
    void f() override;

    void call_base() {
        Base::f();
    }
};
```

这里：

```cpp
Base::f();
```

已经显式指定实现。

同理：

```cpp
p->Base::f();
```

也不会动态派发到 Derived override。

实现层通常可以直接生成普通 direct call。

---

# 43. default argument 为什么不是 virtual

```cpp
struct Base {
    virtual void f(int x = 1);
};

struct Derived : Base {
    void f(int x = 2) override;
};

Derived d;
Base* p = &d;
p->f();
```

调用结果使用：

```text
Derived function body
+
Base default value 1
```

也就是近似：

```cpp
Derived::f(1);
```

原因要拆开：

```text
静态阶段
p static type = Base*
→ 调用点看到 Base::f(int = 1)
→ 缺失参数补成 1

动态阶段
selected interface Base::f(int) 是 virtual
→ final overrider = Derived::f
```

所以：

```text
default argument = static call-site mechanism
virtual function body = dynamic dispatch mechanism
```

工程上不要在 override 层级设置不同默认值。

---

# 44. ctor / dtor 中的 virtual call

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

构造 Derived 时，Base subobject 先建立。

在 `Base::Base()` 执行期间：

```text
Derived 自己的部分尚未构造完成
```

因此 Base constructor 中的 virtual call 不会派发到 Derived::f。

否则 Derived::f 可能访问尚未初始化的 `x`。

析构时反过来：

```text
更派生层先退出
进入 Base destructor 时，Derived 层已经不可再依赖
```

所以 Base destructor 中的虚调用也不会“跳回”已经析构的 Derived。

正确模型不是简单记：

```text
ctor/dtor 关闭 virtual
```

而是：

> **构造/析构期间的 dynamic dispatch 受当前有效生命周期层级限制。**

---

# 45. pure virtual function 与 abstract class

```cpp
struct Shape {
    virtual double area() const = 0;
};
```

`= 0` 声明 pure virtual function。

一个类如果仍有未被 concrete final overrider 满足的 pure virtual interface，就是 abstract class，不能直接实例化：

```cpp
Shape s;      // ill-formed
```

但可以存在：

```cpp
Shape* p;
Shape& r = derived;
```

抽象类的价值是：

```text
定义协议
而不是必须自身提供可直接创建的完整对象实现
```

---

# 46. pure virtual function 可以有定义

`= 0` 表示：

```text
这个 virtual interface 是 pure
```

并不等价于：

```text
这个函数绝对不能存在函数体
```

纯虚函数可以在类外提供定义，并可通过显式 qualified call 使用。

但派生类仍需要提供 concrete override，才能让相应类型不再因为该 pure virtual interface 而保持 abstract。

---

# 47. pure virtual destructor 为什么必须有定义

```cpp
struct Base {
    virtual ~Base() = 0;
};
```

Base 因为 pure virtual destructor 而成为 abstract class。

但销毁 Derived 时：

```text
Derived::~Derived()
    ↓
Base::~Base()
```

Base destructor 仍然是生命周期链的一部分。

因此它必须有定义，例如：

```cpp
Base::~Base() = default;
```

“pure”控制的是抽象接口属性，不意味着析构链可以跳过 Base destructor。

---

# 48. NVI：public interface 与 customization point 分离

一个很典型的设计：

```cpp
class Base {
public:
    void run() {
        check_precondition();
        do_run();
        check_postcondition();
    }

private:
    virtual void do_run() = 0;
};
```

Derived 只 override：

```cpp
void do_run() override;
```

这叫 Non-Virtual Interface 思路。

它把：

```text
public API / invariants
```

和：

```text
runtime customization hook
```

分开。

这也是 private virtual 仍可 override 的工程价值。

---

# 49. object slicing 为什么让 virtual polymorphism 消失

```cpp
void call(Base b) {
    b.f();
}

Derived d;
call(d);
```

传参时已经创建：

```text
一个新的 Base complete object b
```

因此进入函数后：

```text
b 的 most-derived type = Base
```

virtual function 不会“记住它原来从 Derived copy 而来”。

多态依赖当前 object identity，而不是对象历史来源。

所以多态接口通常通过：

```text
Base&
Base*
smart pointer<Base>
```

保留完整动态对象 identity。

---

# 50. virtual destructor：动态入口与静态析构链是两层机制

```cpp
Base* p = new Derived;
delete p;
```

如果 Base destructor 是 virtual，可以把过程拆成两层。

## 50.1 第一层：找到正确析构入口

```text
Base* 静态视图
    ↓ virtual destructor dispatch
most-derived type = Derived
    ↓
进入 Derived 的完整析构入口
```

这是 `virtual` 解决的问题。

## 50.2 第二层：沿 subobject tree 反向销毁

进入 Derived destructor 后：

```text
Derived destructor body
    ↓
Derived members
    ↓
non-virtual direct bases
    ↓
virtual bases（若 Derived 是 most-derived）
```

这不是“每一级都重新走一次 virtual dispatch”。

继承结构在编译 Derived destructor 时已经知道，编译器可以生成相应的直接析构调用链。

因此：

> **virtual destructor 主要决定“从哪一层开始”；普通 destructor 生命周期规则决定“从这里以后怎样向内拆除”。**

---

# 第五部分：编译器 / ABI 如何实现动态多态

# 51. 标准语义与 ABI 实现必须严格分开

C++ 标准保证的是：

```text
Base subobject
static / dynamic type
virtual function
final overrider
correct derived-to-base conversion
correct construction/destruction semantics
```

标准不要求实现一定存在：

```text
vptr
vtable
thunk
vbptr
vbtable
```

下面讨论的是 GCC/Clang/MSVC 等主流实现常见思路。

学习时始终使用：

```text
标准层：必须成立的语言行为
实现层：编译器通常怎样做到
```

两层模型。

---

# 52. 单继承下典型 vptr / vtable 模型

```cpp
struct Base {
    int b;
    virtual void f();
    virtual void g();
};

struct Derived : Base {
    int d;
    void f() override;
};
```

典型布局可概念化为：

```text
Derived object
┌─────────────────────────┐
│ vptr ───────────────────┼──► Derived vtable
├─────────────────────────┤
│ Base::b                 │
├─────────────────────────┤
│ Derived::d              │
└─────────────────────────┘
```

虚表可近似：

```text
Derived vtable
┌─────────────────────────┐
│ slot f → Derived::f     │
├─────────────────────────┤
│ slot g → Base::g        │
└─────────────────────────┘
```

因此：

```cpp
Base* p = &d;
p->f();
```

实现层可能近似：

```text
load vptr from p
    ↓
load function address from slot(f)
    ↓
indirect call
```

---

# 53. virtual slot 为什么不是 runtime overload set

`Base::f(int)` 成为 virtual interface 后，可以把它想成一个稳定逻辑槽位：

```text
interface slot = Base::f(int)
```

Derived override：

```cpp
void f(int) override;
```

是在这个接口槽位上提供新的实现。

因此运行时不是：

```text
重新拿所有 f(int)/f(double)/f(...) 比较参数
```

而是：

```text
已经知道 slot(f-int)
        ↓
根据 dynamic type 取得该 slot 的 final implementation
```

这就是 overload 与 virtual dispatch 的结构性区别。

---

# 54. `this` adjustment 与 multiple inheritance

```cpp
struct B1 {
    virtual void f();
    int x;
};

struct B2 {
    virtual void g();
    int y;
};

struct D : B1, B2 {
    int z;
    void g() override;
};
```

一种典型布局：

```text
D complete object
+0
┌──────────────────────┐
│ B1 subobject         │
│   vptr1              │
│   x                  │
├──────────────────────┤
│ B2 subobject         │ ◀── B2* 可能指这里
│   vptr2              │
│   y                  │
├──────────────────────┤
│ D::z                 │
└──────────────────────┘
```

调用：

```cpp
B2* p = &d;
p->g();
```

`p` 指向 B2 subobject，但最终执行 D::g。

D::g 的实现可能需要适合 D 的 `this`。

ABI 因此可能在 slot 中放：

```text
this-adjusting thunk
```

概念上：

```text
B2-subobject this
    ↓ adjust
D-object this
    ↓
D::g
```

这解释了为什么多继承虚调用不只是“一张 vtable”。

---

# 55. virtual inheritance 下的 offset metadata

virtual base 相对当前 subobject 的位置可能依赖 most-derived layout。

因此实现可能需要：

```text
当前 subobject 的 dispatch/layout metadata
        ↓
virtual base offset
        ↓
定位 shared virtual base
```

在不同 ABI 中具体组织方式不同。

例如：

```text
Itanium C++ ABI
MSVC ABI
```

可能使用不同表结构、额外指针或 offset 描述。

教材要求掌握的是：

> **virtual base address 常常不能像普通 non-virtual base 一样仅靠一个对所有 most-derived type 都固定的静态 offset 表达。**

---

# 56. constructor / destructor 期间的 vptr 状态

标准只规定：

```text
ctor/dtor 期间 dynamic dispatch 不进入尚未构造或已经析构的更派生层
```

主流 ABI 为实现这条语义，通常会在构造/析构不同阶段安装适合当前 class layer 的虚调用状态。

因此实际机器层可能观察到：

```text
vptr 在构造过程中被逐层写入
析构过程中又按层级变化
```

但不要反过来记成：

```text
“因为编译器改 vptr，所以标准才这么规定”
```

因果关系应该是：

```text
语言语义先规定生命周期中的 dispatch 行为
        ↓
ABI 选择某种 vptr/vtable 策略实现它
```

---

# 57. virtual destructor 的典型 ABI 入口

以常见 Itanium C++ ABI 思路为例，一个 destructor 在机器/ABI 层可能存在多个入口，例如概念上的：

```text
base-object destructor
complete-object destructor
deleting destructor
```

它们分别处理不同责任：

```text
base-object destructor
→ 当前类作为别人的 base subobject 被销毁

complete-object destructor
→ 销毁完整对象，包括由 most-derived 负责的 virtual bases

deleting destructor
→ 完整析构后再执行合适 deallocation
```

这是 ABI 实现例子，不是 C++ 源语言要求存在这些名字。

它很好地解释了为什么：

```cpp
delete Base*;
```

通过 virtual destructor 可以完成：

```text
动态找到 most-derived deleting/complete destruction path
        ↓
正确销毁整个 subobject tree
        ↓
释放原始 storage
```

---

# 58. RTTI 与 vtable 的关系

一个声明或继承 virtual function 的类是 polymorphic class。

对多态对象，C++ 支持运行时类型机制：

```text
dynamic_cast
typeid
```

主流 ABI 常把 RTTI metadata 与 vtable/相关 runtime metadata 组织在一起。

但标准只规定：

```text
dynamic type checking / identification 的语义
```

不规定具体表布局。

因此不要说：

```text
“标准规定 dynamic_cast 就是查 vtable”
```

更准确的是：

```text
标准定义运行时类型语义
ABI 常利用虚表相关 metadata 实现
```

---

# 59. virtual call 的典型成本

典型未优化虚调用可能包含：

```text
读取 vptr
读取 slot 中函数地址
间接 branch/call
可能的 this adjustment
```

因此可能带来：

```text
额外 load
间接分支预测压力
限制普通内联
对象空间中额外 dispatch metadata 指针
```

但：

> **virtual 不等于一定慢。**

现代优化器经常能够 devirtualize。

---

# 60. devirtualization

如果编译器能证明 final overrider 唯一，例如：

```cpp
Derived d;
d.f();
```

或者通过：

```text
局部类型传播
whole-program analysis
LTO
final class / final function
```

获得足够信息，就可能把：

```text
vptr → slot → indirect call
```

优化成：

```text
direct call
甚至 inline
```

重要的是：

```text
devirtualization 改变实现方式
不改变语言 virtual semantics
```

程序语义分析仍然按标准 virtual dispatch 推导。

---

# 第六部分：多继承、虚继承与调用二义性

# 61. 三种常见“二义性”必须拆开

多继承场景里至少有三种完全不同的问题。

---

## 61.1 member name lookup ambiguity

```cpp
struct B1 { void f(); };
struct B2 { void f(); };
struct D : B1, B2 {};

D d;
d.f();             // ambiguous
```

问题是：

```text
lookup 找到两个不同基类来源的同名成员
```

可以显式限定：

```cpp
d.B1::f();
```

---

## 61.2 base-subobject conversion ambiguity

普通菱形：

```cpp
struct A {};
struct B1 : A {};
struct B2 : A {};
struct D : B1, B2 {};

D d;
A* p = &d;          // ambiguous
```

问题是 D 中存在两个独立 A subobject。

这和函数 lookup 无关。

---

## 61.3 final-overrider ambiguity

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

D 中只有一个 shared A virtual-base subobject。

但对 A::f 这条 virtual interface，存在两个竞争 override：

```text
B1::f
B2::f
```

如果 D 没有提供唯一统一 override，则 final overrider 不唯一，程序 ill-formed。

修复：

```cpp
struct D : B1, B2 {
    void f() override;
};
```

---

# 62. virtual inheritance 消除 base duplication，不自动消除所有 name ambiguity

这是很重要的一条。

写了：

```cpp
B1 : virtual A
B2 : virtual A
```

只保证：

```text
D 中共享一个 A subobject
```

它不意味着：

```text
所有 B1/B2 中同名成员查找都会自动唯一
所有 virtual override 都自动唯一
```

所以必须分别检查：

```text
object graph
name lookup graph
override graph
```

---

# 63. `static_cast` 与 virtual base downcast

普通非虚 public base：

```cpp
struct B {};
struct D : B {};

B& br = d;
D& dr = static_cast<D&>(br);    // 满足前提时可用
```

但如果 B 是 D 的 virtual base，C++17 不允许使用这种 base-to-derived `static_cast` 形式完成 downcast。

为什么？

因为普通 static downcast 依赖一种可由静态继承关系逆向恢复 enclosing derived object 的关系；virtual base 可能被多个路径共享，其 enclosing most-derived relationship 需要运行时对象信息，不能由这种静态逆转换规则安全表达。

需要运行时验证时，应考虑：

```cpp
dynamic_cast<Derived*>(base_ptr)
```

前提是满足 polymorphic source 等语言条件。

---

# 64. `dynamic_cast` 的 pointer 与 reference 失败语义

```cpp
Base* p = get();

if (auto* d = dynamic_cast<Derived*>(p)) {
    // success
}
```

pointer downcast 失败：

```text
nullptr
```

reference：

```cpp
Derived& d = dynamic_cast<Derived&>(r);
```

失败会抛出：

```cpp
std::bad_cast
```

`dynamic_cast` 的目标不是“更快地调用 virtual function”，而是：

> **运行时验证当前完整动态对象是否满足目标类型关系。**

---

# 65. `typeid` 与 polymorphic glvalue

```cpp
Derived d;
Base& r = d;
```

如果 Base 是 polymorphic class，那么：

```cpp
typeid(r)
```

可以反映动态对象类型。

如果继承体系并非 polymorphic，相同语法的动态类型行为并不相同。

工程上不要用大量 `typeid` 条件分支手工模拟本应由 virtual interface 表达的行为差异。

---

# 第七部分：统一比较与完整推导方法

# 66. overload / hiding / override / virtual dispatch 对照表

| 机制 | overload | hiding | override | virtual dispatch |
|---|---|---|---|---|
| 主要阶段 | 编译期 | name lookup | 类定义关系 | 运行时 |
| 是否需要继承 | 否 | 常见于继承 | 是 | 是 |
| Base 是否需 virtual | 否 | 否 | 是 | 是 |
| 核心依据 | 参数 / 转换排名 | 同名声明 | matching function interface | dynamic type + final overrider |
| 动态类型参与吗 | 否 | 否 | 否 | 是 |
| 参数列表可不同吗 | 需要形成不同 overload | 可以 | 必须满足 override 匹配 | 不重新比较参数 |
| 返回类型 | 不能单独区分 overload | 无关 | 相同或协变 | 已由 override relation 确定 |
| `using Base::f` | 可扩展 overload set | 可解除 hiding | 不决定 | 不改变 final overrider |
| 默认参数 | 可影响 viability | 无关 | 不参与 override 匹配 | 仍然静态选择 |

一句话：

> **overload 决定“选哪条接口”；virtual dispatch 决定“这条接口在当前动态对象上执行哪个实现”。**

---

# 67. object / pointer / reference 的 Derived → Base 对照

| 写法 | 是否创建新对象 | 是否 slicing | 是否保留 dynamic identity | 典型实现 |
|---|---:|---:|---:|---|
| `Base b = d` | 是 | 是 | 否 | 调 Base copy ctor |
| `b = d` | 否，目标已存在 | Base 语义覆盖 | 否 | 调 Base assignment |
| `Base* p = &d` | 否 | 否 | 是 | 定位 Base subobject，可能调地址 |
| `Base& r = d` | 否 | 否 | 是 | 绑定 Base subobject，通常可理解为调整后的地址 |

这是理解多态最重要的一张基础表。

---

# 68. 普通菱形与虚菱形对照

| 维度 | 普通菱形 | virtual inheritance 菱形 |
|---|---|---|
| 最终 A subobject 数量 | 两个 | 一个共享 |
| `D* → A*` | 通常 ambiguous | 若访问条件满足，可唯一定位 shared A |
| A 状态 | 两份 | 一份 |
| A 构造责任 | 各路径自己的 A | most-derived constructor |
| A 析构 | 两个 A 各自销毁 | most-derived 完整析构一次 |
| 地址定位 | 每条非虚路径通常固定 offset | 常需要 virtual-base offset metadata |
| copy/move ctor | 各 non-virtual occurrence | virtual base 初始化一次 |
| defaulted assignment | 两个 A 各自赋值 | shared virtual base 可能被赋值一次或多次，C++17 unspecified |

---

# 69. 一次成员调用的统一推导算法

以后看到：

```cpp
expr.f(args...);
```

固定按以下顺序。

## Step 1：当前是否还是原完整对象

先问：

```text
有没有发生 Base-by-value copy / slicing？
```

如果已经：

```cpp
Base b = derived;
```

则 `b` 的 most-derived type 就是 Base。

## Step 2：写出 expr 的 static type

这是后续 lookup 的入口。

## Step 3：分析继承访问和 subobject 唯一性

如果表达式涉及 upcast，先检查：

```text
base accessible?
base unambiguous?
virtual or non-virtual?
```

## Step 4：做 name lookup

问：

```text
当前 static type 能看到哪些同名函数？
有没有 hiding？
有没有 using Base::f？
有没有 multiple-inheritance lookup ambiguity？
```

## Step 5：形成 candidate functions

只处理 lookup 后真正可参与的声明。

## Step 6：筛 viable functions

检查：

```text
参数数量
default arguments
reference binding
cv/ref object qualification
implicit conversions
```

## Step 7：比较 conversion sequences

例如：

```text
exact
promotion
conversion
user-defined
ellipsis
```

## Step 8：得到 best viable function

如果没有唯一最佳：

```text
ambiguous / ill-formed
```

## Step 9：检查 selected function 的最终可用性

包括：

```text
deleted
access control
```

## Step 10：处理 default arguments

默认参数来自调用点静态可见声明，不进行动态绑定。

## Step 11：selected function 是否 virtual

不是 virtual：

```text
静态调用目标确定
```

是 virtual：继续。

## Step 12：调用是否显式 qualified

```cpp
Base::f()
p->Base::f()
```

会抑制动态派发。

## Step 13：确定 current dynamic type / most-derived object

若通过 Base*/Base& 指向 Derived complete object：

```text
dynamic type = Derived
```

但 ctor/dtor 期间要根据当前生命周期层级修正。

## Step 14：找 final overrider

只沿已选中 virtual interface 的 override relation 找最终实现。

不要重新 overload resolution。

## Step 15：如果分析实现，再进入 ABI 层

问：

```text
当前 pointer 指向哪个 subobject？
upcast 是否需要 address adjustment？
哪个 vptr / table？
slot 是 direct function 还是 thunk？
virtual base 是否需要 runtime offset？
编译器能否 devirtualize？
```

---

# 70. 综合案例：lookup、overload、default argument、virtual dispatch

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

## 70.1 `d.f(1)`

static type = Derived。

`using Base::f` 让 Base overload 参与 lookup。

候选主要有：

```text
Derived::f(int)
Derived::f(const char*)
Base::f(double)
```

int → int 是 exact match。

选择 Derived::f(int)。

结果：

```text
Derived-int:1
```

## 70.2 `d.f(1.5)`

```text
double → Base::f(double) exact
double → Derived::f(int) conversion
```

选择 Base::f(double)。

它不是 virtual，因此无后续动态派发。

## 70.3 `r.f(1)`

static type = Base。

lookup 只基于 Base interface：

```text
Base::f(int)
Base::f(double)
```

选择 Base::f(int)。

它是 virtual。

`r` 引用 Derived complete object，所以 final overrider 是：

```text
Derived::f(int)
```

结果：

```text
Derived-int:1
```

## 70.4 `r.f()`

static lookup 看到：

```cpp
Base::f(int x = 1)
```

缺省实参先变成：

```text
1
```

然后 virtual dispatch 找 Derived::f(int)。

最终：

```text
Derived-int:1
```

而不是 2。

---

# 71. 综合案例：虚菱形 + final overrider

```cpp
struct A {
    virtual void f();
    int state;
};

struct B1 : virtual A {
    void f() override;
};

struct B2 : virtual A {
    void f() override;
};

struct D : B1, B2 {
    void f() override;
};
```

对象结构：

```text
D
├── B1 ──┐
├── B2 ──┼──► one shared A
└────────┘
```

这里：

```text
A state 只有一份
```

同时 D::f 统一覆盖 B1/B2 对 A::f 的竞争，因此：

```text
A::f 在 D most-derived object 中的 final overrider = D::f
```

如果删除 `D::f()`，则对 shared A::f 出现两个 final overrider 候选，类定义 ill-formed。

---

# 72. 常见误区

## 72.1 “Base* 指向 Derived，所以能访问所有 Derived 成员”

错误。

```text
可见接口由 static type 决定
```

Base* 只能通过 Base 静态接口操作。

virtual 只是让已存在的 Base virtual interface 可以执行 Derived override。

---

## 72.2 “只要同名就是 override”

错误。

```text
同名可能只是 hiding
```

override 必须满足 virtual interface 匹配规则。

---

## 72.3 “运行时 virtual dispatch 会重新选 overload”

错误。

```text
overload resolution 先在编译期结束
virtual dispatch 只找该接口 final overrider
```

---

## 72.4 “virtual inheritance 就是 virtual function 的一种”

错误。

一个解决：

```text
共享 base subobject
```

另一个解决：

```text
运行时行为替换
```

---

## 72.5 “virtual diamond 后所有二义性都会消失”

错误。

它只解决共同 virtual base 的 subobject duplication。

仍可能存在：

```text
member lookup ambiguity
final overrider ambiguity
```

---

## 72.6 “Derived → Base pointer 一定不改地址”

错误。

多继承和虚继承中可能需要 address adjustment。

---

## 72.7 “constructor list 写谁在前就先构造谁”

错误。

真正顺序由：

```text
virtual bases
→ direct bases
→ member declaration order
```

决定。

---

## 72.8 “virtual base 由每个中间 base constructor 构造”

错误。

构造 complete object 时，shared virtual base 由 most-derived constructor 负责。

---

## 72.9 “pure virtual 就绝对不能有定义”

错误。

pure virtual 可以有定义；pure virtual destructor 在实际派生对象存在时还必须有定义。

---

## 72.10 “virtual = 一定是一次间接调用”

错误。

语言语义是 virtual；编译器可能通过 devirtualization 生成 direct call 或 inline。

---

# 73. 工程设计原则

## 73.1 public inheritance 应表达替代关系

如果外部接口希望：

```text
Derived 可以自然作为 Base 使用
```

优先考虑 public inheritance。

不要为了代码复用随意建立公开继承层级。

## 73.2 多态基类必须明确 destruction contract

如果允许：

```cpp
delete base_pointer;
```

Base destructor 应为 virtual。

## 73.3 override 显式写 `override`

它把大量意外 hiding / signature mismatch 变成编译错误。

## 73.4 避免 virtual default arguments

尤其不要在不同 override 中给不同默认值。

## 73.5 ctor/dtor 不依赖更派生 virtual behavior

生命周期还没准备好或已经拆除。

## 73.6 虚继承只在确实需要共享共同 base identity/state 时使用

virtual inheritance 会增加：

```text
对象布局复杂度
地址定位复杂度
构造责任复杂度
ABI metadata
```

不要仅因为“有菱形图形”就机械改成 virtual inheritance；先确认语义上是否真的应该共享同一 Base subobject。

## 73.7 能用值语义时，不必强行 runtime inheritance

有些问题更适合：

```text
composition
std::variant
templates / concepts（更高版本）
CRTP
function objects
```

runtime polymorphism 应在确实需要运行时替换行为、保持对象 identity 的地方使用。

---

# 74. 学习检查点

完成本专题后，应能不靠背题回答下面问题。

### 对象模型

1. `Derived` 对象中为什么存在 `Base` subobject？
2. `Base b = derived` 与 `Base& r = derived` 为什么本质不同？
3. `Derived* → Base*` 为什么机器地址可能变化？
4. 什么条件会让 derived-to-base conversion ambiguous？

### 多继承 / 虚继承

5. 普通菱形为什么有两个 A subobject？
6. virtual inheritance 为什么变成一个 shared A？
7. shared virtual base 为什么必须由 most-derived constructor 初始化？
8. 中间类的 virtual-base mem-initializer 在什么情况下被忽略？
9. virtual base 的构造和析构顺序是什么？
10. copy/move constructor 与 assignment 对 virtual base 有什么不同边界？

### overload / lookup

11. name lookup 与 overload resolution 为什么必须分开？
12. Derived 同名成员为什么会 hide Base overload？
13. `using Base::f` 改变了什么？
14. dynamic type 为什么不参与普通 overload resolution？
15. `nullptr` 为什么可能造成多个 pointer overload ambiguous？

### virtual

16. virtual dispatch 发生在 overload resolution 之前还是之后？
17. constructor 为什么不能 virtual？
18. destructor 为什么可以 virtual？
19. Base* 为什么看不到 Derived-only 成员，却能执行 Derived override？
20. default argument 为什么不跟随 final overrider？
21. ctor/dtor 中 virtual call 为什么不进入更派生层？
22. pure virtual function 为什么仍然可以有定义？
23. pure virtual destructor 为什么必须有定义？

### ABI / 编译器

24. vptr / vtable 是标准要求还是典型实现？
25. multiple inheritance 为什么可能需要 thunk？
26. virtual base 为什么可能需要 runtime offset metadata？
27. virtual destructor 为什么在 ABI 中可能存在多个析构入口？
28. devirtualization 为什么不改变 C++ 语言语义？

如果这些问题都可以从对象模型自行推导，而不是靠短语记忆，这个专题才算真正建立起来。

---

# 75. C++17 规则索引

后续需要逐条核对标准时，优先查看 C++17 draft N4659 中这些章节：

```text
[intro.object]          object / subobject / most-derived object
[class.derived]         derived classes / base-class subobjects / virtual bases
[class.access.base]     base class accessibility
[conv.ptr]              derived-to-base pointer conversion
[dcl.init.ref]          reference initialization
[class.member.lookup]   member name lookup in class hierarchies
[over.match]            overload resolution
[over.best.ics]         ranking implicit conversion sequences
[class.virtual]         virtual function / override / final overrider
[class.abstract]        pure virtual / abstract class
[class.base.init]       base/member/virtual-base initialization order
[class.dtor]            destructor order / virtual destructor
[class.cdtor]           construction/destruction phase rules
[expr.static.cast]      static_cast and base/derived relations
[expr.dynamic.cast]     dynamic_cast
[expr.typeid]           RTTI type identification
[class.copy]            copy/move of bases, members and virtual bases
```

---

# 总结：把整套机制压缩成一张图

```text
                C++ inheritance / polymorphism
                           │
                           ▼
                    object hierarchy
                           │
            ┌──────────────┴──────────────┐
            │                             │
      base subobjects               inheritance access
            │                             │
            ├── single                    ├── public
            ├── multiple                  ├── protected
            ├── diamond                   └── private
            └── virtual base
                    │
                    ▼
           object / pointer / reference
                    │
                    ▼
                 static type
                    │
                    ▼
                name lookup
                    │
                    ▼
            overload resolution
                    │
                    ▼
         selected function interface
                    │
             ┌──────┴──────┐
             │             │
         non-virtual     virtual
             │             │
             ▼             ▼
        static target   dynamic type
                           │
                           ▼
                    final overrider
                           │
                           ▼
                    runtime behavior
                           │
                           ▼
          typical ABI implementation layer
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
      vtable          address adjust       RTTI
      / slot          / thunk / vbase       metadata
```

如果这张图能够在脑中稳定存在，那么重载、隐藏、override、虚函数、菱形继承和虚继承就不再是若干彼此独立的“C++ 特例”，而是一套由**对象结构 + 静态类型系统 + 运行时对象身份**共同组成的连续模型。
