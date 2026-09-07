# C++ 虚函数专题：从对象模型、语言语义到编译器实现

> **语义基线：C++17**  
> 本文是 `polymorphism-overload-and-dispatch.md` 的虚函数专题深化。学习目标不是记住“有 `virtual` 就动态绑定”，而是从 **Base subobject / most-derived object / static type / dynamic type** 出发，理解虚函数为什么存在、标准语义究竟保证什么，以及主流编译器/ABI 通常如何把这套语义落到对象布局、vptr/vtable、地址调整和间接调用上。

---

## 目录

1. [为什么 C++ 需要虚函数](#1-为什么-c-需要虚函数)
2. [虚函数建立在什么对象模型之上](#2-虚函数建立在什么对象模型之上)
3. [静态类型、动态类型与 most-derived object](#3-静态类型动态类型与-most-derived-object)
4. [什么叫 virtual function](#4-什么叫-virtual-function)
5. [哪些函数可以是 virtual，哪些不可以](#5-哪些函数可以是-virtual哪些不可以)
6. [为什么 constructor 不能是 virtual](#6-为什么-constructor-不能是-virtual)
7. [为什么 destructor 可以且常常应该是 virtual](#7-为什么-destructor-可以且常常应该是-virtual)
8. [override 到底建立了什么关系](#8-override-到底建立了什么关系)
9. [`virtual`、`override`、`final` 的职责分工](#9-virtualoverridefinal-的职责分工)
10. [一次虚函数调用的完整语言流水线](#10-一次虚函数调用的完整语言流水线)
11. [为什么 Derived* 可以变成 Base*](#11-为什么-derived-可以变成-base)
12. [为什么 Base* 仍能调用 Derived override](#12-为什么-base-仍能调用-derived-override)
13. [编译器在单继承下通常怎么实现虚派发](#13-编译器在单继承下通常怎么实现虚派发)
14. [vptr、vtable 与 virtual slot](#14-vptrvtable-与-virtual-slot)
15. [`this` 指针在虚调用里发生了什么](#15-this-指针在虚调用里发生了什么)
16. [多继承为什么需要 this adjustment / thunk](#16-多继承为什么需要-this-adjustment--thunk)
17. [虚继承下为什么更复杂](#17-虚继承下为什么更复杂)
18. [标准语义与 ABI 实现必须分开](#18-标准语义与-abi-实现必须分开)
19. [显式限定调用为什么关闭虚派发](#19-显式限定调用为什么关闭虚派发)
20. [默认参数为什么不是虚的](#20-默认参数为什么不是虚的)
21. [构造与析构期间的虚调用](#21-构造与析构期间的虚调用)
22. [纯虚函数与抽象类](#22-纯虚函数与抽象类)
23. [纯虚函数为什么仍然可以有定义](#23-纯虚函数为什么仍然可以有定义)
24. [private virtual function 为什么仍可被 override](#24-private-virtual-function-为什么仍可被-override)
25. [NVI：把接口权限和虚派发拆开](#25-nvi把接口权限和虚派发拆开)
26. [协变返回类型](#26-协变返回类型)
27. [虚函数与 overload / hiding 的边界](#27-虚函数与-overload--hiding-的边界)
28. [对象切片为什么会让虚派发消失](#28-对象切片为什么会让虚派发消失)
29. [`dynamic_cast` / `typeid` 与虚函数的关系](#29-dynamic_cast--typeid-与虚函数的关系)
30. [虚函数与 virtual inheritance 不是一回事](#30-虚函数与-virtual-inheritance-不是一回事)
31. [虚调用的性能成本从哪里来](#31-虚调用的性能成本从哪里来)
32. [编译器为什么可以 devirtualize](#32-编译器为什么可以-devirtualize)
33. [`final` 为什么有时能帮助优化](#33-final-为什么有时能帮助优化)
34. [工程设计中的虚函数使用原则](#34-工程设计中的虚函数使用原则)
35. [完整规则表](#35-完整规则表)
36. [统一推导算法](#36-统一推导算法)
37. [常见误区](#37-常见误区)
38. [学习检查点](#38-学习检查点)

---

# 1. 为什么 C++ 需要虚函数

先不要从 `virtual` 关键字开始。

考虑一个图形接口：

```cpp
struct Shape {
    double area() const;
};
```

如果调用者只拿到：

```cpp
Shape& s
```

那么普通成员函数调用：

```cpp
s.area();
```

只能根据 `Shape` 的静态接口确定实现。

但我们真正希望表达的是：

```text
Circle object    → Circle::area
Rectangle object → Rectangle::area
Triangle object  → Triangle::area
```

同时调用者只依赖统一接口：

```cpp
void print_area(const Shape& s) {
    std::cout << s.area();
}
```

这意味着语言必须支持：

> **调用点只知道 Base 接口，但最终函数体由对象的动态类型决定。**

虚函数就是 C++ 为这件事提供的语言机制。

```cpp
struct Shape {
    virtual double area() const = 0;
};
```

现在：

```text
静态接口
Shape::area

运行时实现
Circle::area / Rectangle::area / ...
```

被拆成了两个层次。

这就是 runtime polymorphism 的核心。

---

# 2. 虚函数建立在什么对象模型之上

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

从 C++ 对象模型看，`Derived` complete object 内部包含一个真实的 `Base` subobject：

```text
Derived complete object
┌──────────────────────────────┐
│ Base subobject               │
│   b                          │
│   virtual interface state    │
├──────────────────────────────┤
│ Derived own part             │
│   d                          │
└──────────────────────────────┘
```

所以：

```cpp
Base* p = &obj;
```

不是“把 Derived 假装成 Base”。

更准确地说：

> `p` 指向 `Derived` complete object 中的 `Base` subobject。

但这个 Base subobject 所属的 most-derived object 仍然是：

```text
Derived
```

虚派发正是利用了这个事实：

```text
静态视图 = Base
动态完整对象 = Derived
```

---

# 3. 静态类型、动态类型与 most-derived object

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

可以这样理解：

```text
static type  = Base
dynamic type = Derived
```

对于：

```cpp
r
```

也是：

```text
static type  = Base
dynamic type = Derived
```

这里的“dynamic type”来自那个 Base subobject 所属的 most-derived object。

静态类型主要控制：

```text
name lookup
可见接口
overload resolution
默认参数
普通非虚调用
```

动态类型主要参与：

```text
virtual dispatch
dynamic_cast
typeid（多态 glvalue）
```

因此必须始终分清：

```text
“我能看到哪些函数？”
```

和：

```text
“选中的 virtual interface 最终执行哪个 override？”
```

这是两个阶段。

---

# 4. 什么叫 virtual function

一个非静态成员函数如果在类中声明为：

```cpp
virtual void f();
```

它就是 virtual function。

如果派生类中的成员函数正确 override 它，那么后续即使不再写 `virtual`，它仍然是 virtual：

```cpp
struct Base {
    virtual void f();
};

struct D1 : Base {
    void f();          // 仍是 virtual
};

struct D2 : D1 {
    void f();          // 仍继续 override
};
```

可以把“virtual 属性”理解成沿 override 链传播：

```text
Base::f    virtual interface
   ↑
D1::f      override
   ↑
D2::f      override
```

只要 Base 首次把这个接口声明为 virtual，这条 overriding chain 后面一直具有动态派发语义。

一个至少拥有一个 virtual function 的类通常称为：

```text
polymorphic class
```

这对 RTTI 的某些运行时能力也很重要。

---

# 5. 哪些函数可以是 virtual，哪些不可以

理解限制时，不要背表，要先问：

> 这个函数是否属于“通过一个对象实例调用的、可由派生对象替换实现的成员行为”？

## 5.1 普通非静态成员函数：可以

```cpp
struct Base {
    virtual void f();
};
```

这是最典型的情况。

## 5.2 destructor：可以

```cpp
struct Base {
    virtual ~Base();
};
```

这是多态生命周期协议的一部分。

## 5.3 conversion function：可以

非静态 conversion function 也可以参与 virtual dispatch：

```cpp
struct Base {
    virtual operator int() const;
};
```

虽然工程上较少这样设计，但语言上允许。

## 5.4 某些 member operator：可以

只要该 operator 是符合条件的非静态成员函数，也可以 virtual：

```cpp
struct Base {
    virtual Base& operator+=(int);
};
```

是否值得这样设计是另一回事。

## 5.5 constructor：不可以

```cpp
struct Base {
    virtual Base();   // ill-formed
};
```

原因不是语法任性，而是对象尚未完成建立，后面专门分析。

## 5.6 static member function：不可以

```cpp
struct Base {
    static virtual void f();   // ill-formed
};
```

因为 static member function 没有某个具体对象实例作为隐式对象参数，也就没有：

```text
this
most-derived object
dynamic type
```

可供虚派发使用。

## 5.7 free function：不可以

```cpp
virtual void f();   // 不能把普通非成员函数做成 virtual
```

虚派发是 class member / object model 机制。

## 5.8 member function template：不能是 virtual function

不能写：

```cpp
struct Base {
    template<class T>
    virtual void f(T);   // ill-formed
};
```

一个重要设计原因是：

```text
virtual interface
→ 继承体系需要一个稳定、可识别的函数槽位

template member
→ 具体实例集合取决于后续被哪些 T 实例化
```

后者无法自然对应传统意义上预先确定的 virtual slot 集合。

---

# 6. 为什么 constructor 不能是 virtual

这是理解虚函数设计最重要的问题之一。

假设语法上允许：

```cpp
virtual Base();
```

马上出现一个根本问题：

> 在 constructor 被调用之前，我们到底已经有一个什么动态类型的对象？

普通虚调用依赖：

```text
已经存在的对象
↓
找到它的 Base subobject
↓
知道 most-derived object
↓
找 final overrider
```

但是 constructor 的职责本身就是：

```text
让对象开始存在
```

在最外层构造逻辑开始之前，不能反过来要求“先有一个完整动态对象，再根据动态对象选择该调用哪个 constructor”。

这会形成循环依赖。

C++ 采用的模型是：

```text
先由调用表达式静态决定构造哪个类型
↓
分配/准备该类型需要的 storage
↓
按 Base → member → Derived 的顺序逐层建立对象
```

例如：

```cpp
Derived d;
```

编译器在编译期就知道：

```text
构造目标 = Derived
```

然后调用 Derived 的构造协议。

如果工程上需要“根据运行时类型决定创建哪个派生对象”，通常使用 factory：

```cpp
std::unique_ptr<Base> make(Type t);
```

factory 本身根据运行时信息选择：

```cpp
std::make_unique<DerivedA>()
std::make_unique<DerivedB>()
```

这才是“动态创建不同派生类型”的正确层次。

---

# 7. 为什么 destructor 可以且常常应该是 virtual

constructor 解决：

```text
我要创建哪一种对象？
```

在构造前就必须确定具体类型。

但 destructor 面对的是另一种场景：

```cpp
Base* p = new Derived;
delete p;
```

对象已经完整存在。

此时我们知道：

```text
static pointer type = Base*
most-derived object = Derived
```

所以完全可以根据动态对象决定销毁链。

如果 Base destructor 是 virtual：

```cpp
struct Base {
    virtual ~Base() = default;
};
```

那么：

```cpp
delete p;
```

可以正确形成：

```text
Derived::~Derived()
↓
Base::~Base()
```

这不是“普通函数 f() 的虚派发再套到 destructor 上”这么简单，而是：

> **Base 明确把“通过 Base 接口销毁整个 most-derived object”纳入其类型协议。**

如果一个基类准备被多态删除，virtual destructor 通常是必要的。

---

# 8. override 到底建立了什么关系

考虑：

```cpp
struct Base {
    virtual void f(int) const &;
};

struct Derived : Base {
    void f(int) const & override;
};
```

这里 `Derived::f` override `Base::f`。

关键不是“名字一样”这么简单。

核心匹配包括：

```text
同名
相同 parameter-type-list
相同 cv-qualification
相同 ref-qualifier
返回类型相同或满足协变规则
```

例如：

```cpp
struct Base {
    virtual void f() const;
};

struct Derived : Base {
    void f();   // NOT override
};
```

`const` 不同，因此不是同一条 virtual interface。

再例如：

```cpp
struct Base {
    virtual void f() &;
};

struct Derived : Base {
    void f() &&;
};
```

ref qualifier 不同，也不是 override。

C++17 中 override 的 exception specification 也必须满足兼容性要求，派生版本不能比基类虚函数承诺得更宽松。

工程上应当把：

```cpp
override
```

看作“编译器验证我的意图”的机制。

---

# 9. `virtual`、`override`、`final` 的职责分工

## `virtual`

建立动态派发接口：

```cpp
struct Base {
    virtual void f();
};
```

## `override`

验证当前函数确实覆盖了某条 Base virtual interface：

```cpp
struct Derived : Base {
    void f() override;
};
```

如果写错：

```cpp
void f(int) override;
```

而 Base 只有：

```cpp
virtual void f();
```

编译器直接报错。

## `final`

阻止后续 override：

```cpp
struct D : Base {
    void f() final;
};
```

或者阻止整个类继续派生：

```cpp
struct D final : Base {};
```

因此三者可以理解为：

```text
virtual  → 打开动态替换点
override → 验证我正在替换
final    → 从这里停止继续替换
```

---

# 10. 一次虚函数调用的完整语言流水线

考虑：

```cpp
Base* p = &derived;
p->f(1);
```

正确推理绝不是：

```text
p 指向 Derived
→ 去 Derived 里找 f
```

而应该是：

```text
Step 1  p 的静态类型是 Base*
        ↓
Step 2  对 Base 做 member name lookup
        ↓
Step 3  建立 overload candidate set
        ↓
Step 4  overload resolution
        ↓
Step 5  选中某个 Base::f(...) 接口
        ↓
Step 6  检查它是不是 virtual
        ↓
Step 7  若 virtual 且调用未被显式限定
        ↓
Step 8  查看 *p 所属 most-derived object
        ↓
Step 9  找这条 virtual interface 的 final overrider
        ↓
Step 10 调用最终实现
```

关键结论：

> **运行时 virtual dispatch 不会重新做 overload resolution。**

运行时只是在已经选定的 virtual interface 上找 final overrider。

---

# 11. 为什么 `Derived*` 可以变成 `Base*`

考虑 public、可访问、无歧义继承：

```cpp
struct Base {
    virtual void f();
};

struct Derived : public Base {
    void f() override;
};

Derived d;
Derived* pd = &d;
Base* pb = pd;
```

之所以能做这个隐式 upcast，根本原因是：

> 每一个 `Derived` complete object 内部一定包含对应的 `Base` subobject。

所以转换不是“猜测”。

而是：

```text
Derived complete object
┌──────────────────────┐
│ Base subobject       │ ◀── pb
├──────────────────────┤
│ Derived own part     │
└──────────────────────┘
        ▲
        pd
```

标准语义只要求：

```text
pb 指向正确的 Base subobject
```

至于机器地址是否需要改变，取决于布局和 ABI。

---

# 12. 为什么 `Base*` 仍能调用 `Derived` override

这是最容易被误解成“Base 指针偷偷访问了 Derived 成员”的地方。

例如：

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    void f() override;
    void g();
};

Derived d;
Base* p = &d;
```

下面：

```cpp
p->g();
```

不允许，因为：

```text
p 的 static type = Base*
Base 的静态接口中没有 g
```

但：

```cpp
p->f();
```

允许，因为：

```text
Base 的静态接口中有 f
```

并且 `f` 是 virtual。

于是：

```text
静态阶段：选中 Base::f 这条 interface
动态阶段：Derived 是 most-derived type
          ↓
          final overrider = Derived::f
```

因此执行：

```cpp
Derived::f()
```

所以真正的规则不是：

```text
Base* 可以访问 Derived
```

而是：

> **Base* 只能使用 Base 暴露的接口；virtual 允许 Derived 替换这些接口的实现。**

---

# 13. 编译器在单继承下通常怎么实现虚派发

下面进入实现层。

注意：

> 以下 vptr/vtable 描述是主流 ABI 的典型实现，不是 C++ 标准要求的唯一实现。

考虑：

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

一种典型布局可以想象为：

```text
Derived object
┌──────────────────────────┐
│ vptr ────────────────────┼────► Derived vtable
├──────────────────────────┤
│ Base::b                  │
├──────────────────────────┤
│ Derived::d               │
└──────────────────────────┘
```

vtable 概念上可能是：

```text
Derived vtable
┌──────────────────────────┐
│ slot f → Derived::f      │
├──────────────────────────┤
│ slot g → Base::g         │
└──────────────────────────┘
```

因为 Derived override 了 `f`，对应 slot 指向 Derived implementation。

`g` 没 override，所以仍指向 Base implementation。

---

# 14. vptr、vtable 与 virtual slot

可以把一条 virtual interface 想象成继承体系中一个稳定的“逻辑槽位”。

例如 Base 定义：

```cpp
virtual void f();
```

那么从语义上形成：

```text
virtual interface = Base::f
```

实现上典型 ABI 会为它安排某个 vtable slot。

Derived：

```cpp
void f() override;
```

不是增加一条完全无关的 runtime overload，而是替换/更新：

```text
同一 virtual slot 对 Derived dynamic type 的目标
```

所以虚派发的本质更接近：

```text
interface slot 固定
implementation 随 dynamic type 改变
```

而不是：

```text
运行时重新根据参数选函数
```

这再次解释了为什么：

```text
virtual dispatch != overload resolution
```

---

# 15. `this` 指针在虚调用里发生了什么

成员函数：

```cpp
void Derived::f();
```

执行时需要一个指向当前对象/子对象语义的隐式对象参数，通常用 `this` 理解。

对于简单单继承：

```cpp
Base* p = &derived;
p->f();
```

如果 Base subobject 正好位于 Derived 对象开头，`p` 的地址和完整对象起始地址在机器上可能相同。

虚表 slot 可以直接指向：

```text
Derived::f
```

调用时把 `p` 作为隐式对象地址传入。

但这只是最简单情况。

一旦 Base subobject 不是 offset 0，就需要进一步调整 `this`。

---

# 16. 多继承为什么需要 `this adjustment / thunk`

考虑：

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

典型布局可能是：

```text
D complete object
address + 0
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

现在：

```cpp
D d;
B2* p = &d;
p->g();
```

`p` 指向的是 B2 subobject，而 `D::g()` 的函数体通常希望以适合 D 对象的 `this` 语义运行。

因此 ABI 可能在 vtable slot 里放一个小型 thunk：

```text
B2 virtual slot
    ↓
this-adjusting thunk
    ↓
把 this 从 B2 subobject address 调整回 D 所需地址
    ↓
jump/call D::g
```

伪代码概念上类似：

```text
this = this - offset_of_B2_in_D
jump D::g
```

具体方向和数值取决于 ABI。

这就是为什么多继承的 virtual call 不能简单理解成：

```text
vtable 里直接存函数地址就结束了
```

有时还必须解决：

```text
哪个 subobject 视图进入调用？
callee 需要怎样的 this？
```

---

# 17. 虚继承下为什么更复杂

虚继承：

```cpp
struct D : virtual Base {};
```

解决的是：

```text
多条继承路径共享同一个 virtual Base subobject
```

它会让 Base subobject 的位置不再总能简单表示成某个静态固定 offset。

主流 ABI 可能需要在对象/虚表相关元数据中保存：

```text
virtual base offset
```

以便从当前 subobject 找到共享 virtual base。

当它和 virtual function、multiple inheritance 组合时，实现可能同时需要：

```text
vptr
vtable
virtual-base offset
this adjustment
thunk
```

但标准层面仍然只要求：

```text
正确找到对象关系
+
正确执行 final overrider
```

所以不要把复杂 ABI 细节反过来当作语言规则。

---

# 18. 标准语义与 ABI 实现必须分开

这张表要牢牢记住：

| 标准保证 | 主流实现常见做法 |
|---|---|
| virtual function | vtable slot |
| dynamic dispatch | vptr 间接取函数地址 |
| final overrider | slot 指向 override / thunk |
| Base subobject | 某段 object layout |
| Derived* → Base* | address adjustment |
| multiple inheritance | 多个 vptr / secondary vtable |
| virtual base | virtual-base offset metadata |
| 调整到正确对象视图 | thunk / this adjustment |

标准不要求成员名叫：

```text
vptr
vtable
```

甚至不要求实现必须使用这种数据结构。

真正可依赖的是语言行为。

---

# 19. 显式限定调用为什么关闭虚派发

考虑：

```cpp
struct Base {
    virtual void f();
};

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

已经由程序员明确指定：

```text
我要 Base::f 这个实现
```

所以不再进行 runtime virtual dispatch。

同样：

```cpp
p->Base::f();
```

也明确关闭动态选择。

实现层面通常意味着：

```text
不需要从 vptr/vtable 间接取 slot
```

编译器可以直接生成对 Base::f 的调用。

---

# 20. 默认参数为什么不是虚的

考虑：

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

结果等价于：

```cpp
Derived::f(1);
```

为什么？

因为调用点首先按静态类型做：

```text
p 的 static type = Base*
↓
可见声明 = Base::f(int = 1)
↓
缺失参数由编译器在调用点补成 1
```

得到：

```cpp
p->f(1);
```

然后才进入 virtual dispatch：

```text
dynamic type = Derived
↓
final overrider = Derived::f
```

所以：

```text
default argument → static
function body    → dynamic
```

这也是工程上通常不建议让 override 层级各自定义不同默认参数的原因。

---

# 21. 构造与析构期间的虚调用

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

构造 Derived 的顺序是：

```text
Base subobject
↓
Derived own part
```

执行：

```cpp
Base::Base()
```

时，Derived 部分尚未完成构造。

所以这里的虚调用不会跑到 Derived::f。

当前构造阶段只把对象看作当前已经构造到的类层级。

因此：

```cpp
Base() { f(); }
```

调用 Base 层级对应实现。

析构则相反：

```text
Derived destructor body
↓
Derived members/bases teardown
↓
进入 Base destructor
```

进入 Base destructor 时，更派生层已经不再是可安全依赖的完整状态，所以 Base destructor 内虚调用也不会重新跳回 Derived override。

这不是编译器“暂时关闭 vtable”这样简单的标准规则。

正确的语言模型是：

> **构造/析构期间，virtual dispatch 的有效动态层级受当前正在构造/析构的类限制。**

主流 ABI 为满足这条规则，确实会在构造/析构阶段安排对应的 vptr/vtable 状态，但具体策略属于实现细节。

工程结论：

```text
不要在 ctor/dtor 中依赖更派生类 virtual behavior
```

---

# 22. 纯虚函数与抽象类

```cpp
struct Shape {
    virtual double area() const = 0;
};
```

这里：

```text
= 0
```

把 `area` 声明为 pure virtual function。

如果一个类在其最终接口中仍存在未获得 concrete final overrider 的 pure virtual function，它就是 abstract class。

所以：

```cpp
Shape s;   // ill-formed
```

但：

```cpp
Shape* p;
Shape& r = derived;
```

完全可以。

派生类：

```cpp
struct Circle : Shape {
    double area() const override;
};
```

提供 concrete override 后，`Circle` 可以成为可实例化类。

pure virtual 的设计意义是：

```text
Base 定义 interface contract
但不允许 Base 自己成为一个完整 concrete object
```

---

# 23. 纯虚函数为什么仍然可以有定义

一个很容易被忽略的事实是：

> **pure virtual 并不等于“绝对不能有函数体”。**

例如可以在类外提供定义：

```cpp
struct Base {
    virtual void f() = 0;
};

void Base::f() {
    // common implementation
}
```

`Base` 仍然是 abstract，因为：

```text
pure status 来自声明中的 = 0
```

而不是来自“有没有函数体”。

派生类可以 override 它，同时在自己的实现里显式调用 Base implementation：

```cpp
struct Derived : Base {
    void f() override {
        Base::f();
        // extra work
    }
};
```

这里：

```cpp
Base::f();
```

是显式限定调用，所以不是虚派发，而是直接调用 pure virtual 的那个已提供定义。

一个特别重要的例子是 pure virtual destructor：

```cpp
struct Base {
    virtual ~Base() = 0;
};

Base::~Base() = default;
```

即使 destructor 是 pure virtual，它仍然必须有定义，因为销毁 Derived 时 Base destructor 最终仍会作为析构链的一部分执行。

注意：不要在 constructor/destructor 中依赖虚调用落到 pure virtual；如果当前构造/析构阶段的 final overrider 是 pure virtual，进行这种虚调用会进入未定义行为领域。

---

# 24. private virtual function 为什么仍可被 override

考虑：

```cpp
struct Base {
private:
    virtual void do_work();
};

struct Derived : Base {
    void do_work() override;
};
```

这是允许的。

原因是：

```text
access control
```

和：

```text
overriding relation
```

是两个不同机制。

`private` 表示：

```text
谁可以通过名字直接访问 Base::do_work
```

而 override 解决：

```text
Derived 的这个成员是否替换 Base virtual interface 的实现
```

所以一个 Base private virtual 完全可以被 Derived override。

这也是一个重要设计启示：

```text
接口访问权限
和
动态替换能力
可以分离
```

---

# 25. NVI：把接口权限和虚派发拆开

Non-Virtual Interface（NVI）是一种常见设计：

```cpp
class Base {
public:
    void run() {
        // common precondition check
        do_run();
        // common postcondition work
    }

private:
    virtual void do_run() = 0;
};
```

Derived：

```cpp
class Derived : public Base {
private:
    void do_run() override {
        // customization
    }
};
```

好处是：

```text
public non-virtual interface
→ Base 控制统一入口和不变量

private/protected virtual hook
→ Derived 只定制需要变化的部分
```

这样比把所有 public API 都直接 virtual 化更容易维护接口契约。

---

# 26. 协变返回类型

通常 override 返回类型要求一致。

但 class pointer/reference 有一个重要例外：covariant return。

```cpp
struct Base {
    virtual Base* clone();
};

struct Derived : Base {
    Derived* clone() override;
};
```

为什么允许？

因为：

```text
Derived* 可以向上转换成 Base*
```

所以通过 Base 静态接口调用时仍满足 Base contract；通过 Derived 静态接口调用时又能得到更精确的返回类型。

同理可用于相应 class reference。

但不能写：

```cpp
struct Base {
    virtual int f();
};

struct Derived : Base {
    double f() override;  // ill-formed
};
```

普通数值类型之间的“可转换”并不等于协变返回规则。

---

# 27. 虚函数与 overload / hiding 的边界

考虑：

```cpp
struct Base {
    virtual void f(int);
};

struct Derived : Base {
    void f(double);
};
```

这里：

```text
Derived::f(double)
```

会在 Derived 作用域中 hide Base::f，但它不 override Base::f(int)。

于是：

```cpp
Derived d;
d.f(1);
```

直接 lookup Derived::f(double)，把 `1` 转 double。

但：

```cpp
Base* p = &d;
p->f(1);
```

静态 lookup 在 Base，选中 Base::f(int)。

因为 Derived 没有对应 override，所以 final overrider 仍然是 Base::f(int)。

所以一定要分开：

```text
hiding
→ name lookup

override
→ virtual interface relation

overload
→ compile-time candidate selection

virtual dispatch
→ runtime final overrider
```

---

# 28. 对象切片为什么会让虚派发消失

```cpp
struct Base {
    virtual void f();
};

struct Derived : Base {
    void f() override;
};

void call(Base b) {
    b.f();
}

Derived d;
call(d);
```

这里传参时创建了一个新的 Base complete object：

```text
原 d：
Derived complete object
┌────────────────────┐
│ Base subobject     │
├────────────────────┤
│ Derived part       │
└────────────────────┘

参数 b：
Base complete object
┌────────────────────┐
│ Base               │
└────────────────────┘
```

进入 `call` 后：

```text
b 的 most-derived type = Base
```

因此 `b.f()` 的 final overrider 自然就是 Base::f。

虚函数不会“记住 b 原来是从 Derived copy 出来的”。

多态依赖的是当前对象身份，而不是历史来源。

---

# 29. `dynamic_cast` / `typeid` 与虚函数的关系

一个类只要存在 virtual function，就是 polymorphic class。

polymorphic class 的对象通常携带足够的运行时类型信息，使语言可以支持：

```cpp
dynamic_cast<Derived*>(p)
```

以及对 polymorphic glvalue：

```cpp
typeid(*p)
```

这里不要把因果关系简化成：

```text
“因为有 vtable，所以 dynamic_cast 才能工作”
```

标准只规定语义。

实现通常会把 RTTI 信息和虚表/相关 ABI metadata 关联起来，但具体组织方式由 ABI 决定。

虚派发回答：

```text
已知 virtual interface，当前对象执行哪个 implementation？
```

`dynamic_cast` 回答：

```text
当前动态对象是否满足某个运行时类型关系？
```

它们都利用 dynamic type，但解决不同问题。

---

# 30. 虚函数与 virtual inheritance 不是一回事

C++ 中 `virtual` 关键字还会出现在：

```cpp
struct D : virtual Base {};
```

这是 virtual inheritance。

它解决：

```text
多条继承路径如何共享同一个 Base subobject
```

而 virtual function 解决：

```text
Base interface 的 implementation 如何按 dynamic type 替换
```

两者完全是不同维度：

| 特性 | virtual function | virtual inheritance |
|---|---|---|
| 核心问题 | 动态行为派发 | 基类子对象共享 |
| 作用对象 | member function | inheritance graph |
| 关键概念 | final overrider | virtual base subobject |
| 常见实现 | vtable slot | virtual-base offset metadata |

虽然主流 ABI 可能都利用某类表结构和 offset metadata，但不能因为实现上都和“虚表”有关就把语义混为一谈。

---

# 31. 虚调用的性能成本从哪里来

语义上，virtual call 多了一层运行时选择。

典型实现中：

```cpp
p->f();
```

可能近似为：

```text
1. 从 object/subobject 读取 vptr
2. 根据固定 slot offset 读取函数地址
3. 间接 call
```

概念伪汇编可能类似：

```text
load  table = [object]
load  fn    = [table + slot]
call  fn
```

这可能带来：

```text
额外内存读取
间接分支
分支预测压力
限制普通内联机会
```

多继承下还可能加：

```text
this adjustment / thunk
```

对象本身通常也可能因为 vptr 增加一定空间开销。

但是不要机械地得出：

```text
virtual = 一定慢
```

现代编译器经常可以 devirtualize。

---

# 32. 编译器为什么可以 devirtualize

所谓 devirtualization，就是：

> 编译器虽然面对语义上的 virtual call，但能够证明最终实现唯一，于是直接生成普通调用甚至内联。

例如：

```cpp
Derived d;
d.f();
```

如果编译器明确知道：

```text
对象就是 Derived
```

并且没有可能的更派生动态对象参与当前表达式，那么 final overrider 可以在编译期确定。

于是实现可以从：

```text
vptr → slot → indirect call
```

优化成：

```text
direct call Derived::f
```

甚至：

```text
inline Derived::f body
```

LTO、whole-program analysis、局部类型传播等都可能帮助这种证明。

关键点是：

> **优化改变实现，不改变 virtual language semantics。**

程序员仍应先按标准动态派发规则推理结果。

---

# 33. `final` 为什么有时能帮助优化

例如：

```cpp
struct D final : Base {
    void f() override;
};
```

`final` 向编译器明确提供：

```text
D 不会再有派生类
```

或者：

```cpp
void f() final;
```

明确表示：

```text
这条 virtual interface 后面不会再被 override
```

在某些调用上下文中，这能让编译器更容易证明：

```text
final overrider 唯一
```

从而 devirtualize。

但不要把：

```text
final → 保证性能更快
```

当成语言规则。

它只是可能提供优化信息；最终是否优化取决于编译器和上下文。

---

# 34. 工程设计中的虚函数使用原则

## 34.1 多态删除必须设计 destructor contract

如果 Base 会被：

```cpp
Base* p = new Derived;
delete p;
```

那么 Base destructor 应设计为 virtual。

常见写法：

```cpp
virtual ~Base() = default;
```

## 34.2 override 一律显式写 `override`

```cpp
void f() override;
```

它能把签名错误从“静默 hiding”变成编译错误。

## 34.3 避免给 virtual override 层级设置不同默认参数

因为：

```text
默认参数静态
函数体动态
```

非常容易制造：

```text
Derived body + Base default value
```

这样的非直观行为。

## 34.4 不要在 ctor/dtor 依赖动态多态

如果需要派生类完成初始化工作，考虑：

```text
factory
post-construction step
NVI hook after complete construction
```

而不是 Base ctor 中调用 virtual 希望进入 Derived。

## 34.5 区分 public API 与 customization point

NVI 是一种常用方式：

```text
public non-virtual interface
+
private/protected virtual hook
```

## 34.6 能用值语义就不要为了“面向对象”强行 virtual

virtual polymorphism 适合：

```text
运行时需要替换行为
调用者只依赖稳定 Base interface
对象 identity 需要保持
```

如果类型天然适合值语义、variant、template/CRTP，未必需要 virtual hierarchy。

---

# 35. 完整规则表

| 问题 | 结论 |
|---|---|
| 普通非静态成员函数能 virtual 吗 | 可以 |
| constructor 能 virtual 吗 | 不可以 |
| destructor 能 virtual 吗 | 可以 |
| static member function 能 virtual 吗 | 不可以 |
| free function 能 virtual 吗 | 不可以 |
| member function template 能 virtual 吗 | 不可以 |
| Derived override 时必须重复写 `virtual` 吗 | 不需要 |
| 建议写 `override` 吗 | 强烈建议 |
| private virtual 可以被 override 吗 | 可以 |
| default argument 参与 override 匹配吗 | 不参与 |
| default argument 动态绑定吗 | 不，按调用点静态声明选择 |
| 返回类型可不同吗 | 仅特定 class pointer/reference 协变情况 |
| 显式 `Base::f()` 还会虚派发吗 | 不会 |
| ctor 中 virtual call 会到更派生类吗 | 不会 |
| dtor 中 Base 阶段 virtual call 会回到已析构 Derived 吗 | 不会 |
| pure virtual 能有定义吗 | 可以 |
| pure virtual destructor 需要定义吗 | 需要 |
| `Base*` 能调用 Derived-only 成员吗 | 不能，静态接口仍是 Base |
| `Base*` 能执行 Derived override 吗 | 可以，前提是先选中 Base virtual interface |
| virtual dispatch 会重新 overload resolution 吗 | 不会 |
| vptr/vtable 是标准强制的吗 | 不是，是主流实现方式 |
| multiple inheritance 可能需要 this adjustment 吗 | 典型 ABI 中可能需要 |
| virtual call 一定是间接 call 吗 | 不一定，编译器可 devirtualize |
| `final` 一定提高性能吗 | 不保证 |

---

# 36. 统一推导算法

以后遇到虚函数题，固定按下面顺序。

## Step 1：先确认对象有没有被 slicing

如果：

```cpp
Base b = derived;
```

`b` 已经是独立 Base complete object。

后面不要再把 dynamic type 当 Derived。

## Step 2：写出表达式 static type

例如：

```cpp
Base* p = &d;
```

对：

```cpp
p->f()
```

静态分析入口是 Base。

## Step 3：做 name lookup

Base interface 中到底看得到哪些 `f`？

## Step 4：做 overload resolution

从可见候选中选 best viable function。

## Step 5：处理 default arguments

缺失参数按调用点静态声明补齐。

## Step 6：检查选中的函数是不是 virtual

如果不是，到此结束。

## Step 7：检查是否显式 qualified

```cpp
Base::f()
p->Base::f()
```

会关闭 virtual dispatch。

## Step 8：确定当前 most-derived object

```cpp
Base* p = &derived;
```

通常：

```text
dynamic type = Derived
```

但 ctor/dtor 阶段需要按当前生命周期层级重新判断。

## Step 9：找 selected virtual interface 的 final overrider

不要运行时重新拿所有同名函数比参数。

只沿这条 virtual interface 的 override chain 找最终实现。

## Step 10：如果分析编译器实现，再问 ABI

```text
Base subobject 位于哪里？
Derived* → Base* 是否需要 address adjustment？
当前 subobject 有哪个 vptr？
slot 指向实现还是 thunk？
是否需要 this adjustment？
编译器能否 devirtualize？
```

这一步属于实现层，不能反推语言结论。

---

# 37. 常见误区

## 误区 1：有 `virtual` 就运行时去 Derived 里按参数找最佳函数

错误。

正确流程：

```text
静态 lookup + overload
↓
先选中 virtual interface
↓
运行时只找 final overrider
```

## 误区 2：Base* 能调用 Derived override，所以 Base* 能访问 Derived 全部成员

错误。

Base* 的静态接口仍然只有 Base。

## 误区 3：vtable 是 C++ 标准规定的对象成员

错误。

vtable/vptr 是主流实现策略。

## 误区 4：override 必须显式写 `virtual`

错误。

一旦 Base function 是 virtual，正确 override 后 virtual 属性继续存在。

## 误区 5：函数同名就是 override

错误。

```cpp
Base::f(int)
Derived::f(double)
```

可能只是 hiding。

## 误区 6：virtual destructor 只是“让 Base destructor 动态调用”

不够准确。

它建立的是：

```text
通过 Base 接口安全销毁 most-derived object
```

的生命周期协议。

## 误区 7：ctor 中调用 virtual 会自动进入 Derived

错误。

更派生部分还没完成构造。

## 误区 8：pure virtual 就绝对不能有实现

错误。

pure virtual 可以有类外定义；pure virtual destructor 甚至必须有定义。

## 误区 9：virtual call 一定慢且无法内联

错误。

如果编译器能够证明 final overrider，可能 devirtualize 甚至 inline。

---

# 38. 学习检查点

学完这一专题后，应该能不靠背答案解释下面问题。

### 检查点 1

为什么：

```cpp
Derived* pd = &d;
Base* pb = pd;
```

通常可以隐式成立？

应能从：

```text
Derived complete object 中存在 Base subobject
```

解释，而不是只记“子类指针可以给父类指针”。

### 检查点 2

为什么：

```cpp
pb->derived_only();
```

不允许，但：

```cpp
pb->virtual_f();
```

却可能执行 Derived implementation？

应能拆成：

```text
static interface
vs
runtime final overrider
```

### 检查点 3

为什么 constructor 不能 virtual，而 destructor 可以 virtual？

应能从：

```text
构造前尚无完整动态对象
vs
析构时动态对象已经存在
```

推导。

### 检查点 4

为什么：

```cpp
Base* p = &d;
p->f();
```

运行时不会重新拿 Derived 所有 overload 重新做 overload resolution？

应能说明：

```text
compile time selects interface
runtime selects final overrider
```

### 检查点 5

为什么：

```cpp
p->Base::f();
```

会关闭 virtual dispatch？

### 检查点 6

为什么：

```cpp
Base b = d;
b.f();
```

不能再调 Derived override？

应能从 object slicing 和 most-derived object identity 解释。

### 检查点 7

为什么多继承下：

```cpp
B2* p = &d;
p->g();
```

实现可能需要 thunk / this adjustment？

### 检查点 8

为什么 `final` 可能帮助 devirtualization，但不能承诺性能一定提升？

---

## 最终心智模型

虚函数专题最终应该收敛成下面这张图：

```text
                 Derived complete object
                         │
                 contains Base subobject
                         │
Derived* ── upcast ──> Base* / Base&
                         │
                         │ static type
                         ▼
                name lookup / overload
                         │
                         ▼
              selected Base virtual interface
                         │
                         │ runtime dynamic type
                         ▼
                  final overrider
                         │
            ┌────────────┴────────────┐
            │ language semantics      │
            │                         │
            │ implementation may use │
            ▼                         ▼
        vptr/vtable               thunk / this adjust
            │                         │
            └────────────┬────────────┘
                         ▼
                    target body
```

最值得保留的一句话是：

> **静态类型决定“能调用哪条接口”，动态类型只在已经选中的 virtual interface 上决定“执行哪个最终实现”；vptr/vtable 只是编译器实现这套语义的一种常见手段。**
