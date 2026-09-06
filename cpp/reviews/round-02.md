# C++ 中高级选择题训练：第 2 轮复盘

> 语义基线：C++17

## 成绩

用户答案：`1A, 2B, 3A, 4C, 5C, 6D, 7B, 8C`

正确答案：`1B, 2B, 3B, 4C, 5C, 6D, 7A, 8C`

成绩：**5 / 8，62.5%**

### 本轮结论

本轮多态相关题目全部答对，说明以下知识已经明显稳定：

- 名字隐藏与 override 的区别；
- 虚函数与默认参数的静态/动态绑定分裂；
- 虚析构与正确的析构链。

当前主要薄弱点集中在：

1. `= default` 与“user-declared”的关系；
2. move constructor“不存在”与“存在但 deleted”的区别；
3. C++17 guaranteed copy elision（强制拷贝消除）。

---

# 第 1 题：`= default` 与隐式移动构造

```cpp
class A {
public:
    A() = default;
    ~A() = default;
};
```

用户答案：**A**  
正确答案：**B**  
结果：❌

## 正确结论

```cpp
~A() = default;
```

虽然要求编译器生成默认析构行为，但这个析构函数仍然是**用户声明的（user-declared）析构函数**。

在 C++11 及以后，编译器只有在以下五类特殊成员函数都没有被用户声明时，才会隐式声明普通的 move constructor：

```text
copy constructor
move constructor
copy assignment operator
move assignment operator
destructor
```

因此本题由于存在：

```cpp
~A() = default;
```

编译器不会再隐式声明：

```cpp
A(A&&);
```

所以 B 正确。

## `= default` 到底是什么意思

```cpp
A() = default;
```

不是：

```text
“这个函数不存在”
```

也不是：

```text
“当作没写过”
```

而是：

> 程序员显式声明这个特殊成员函数，并要求编译器按照语言规则生成默认实现。

所以要区分两个概念：

```text
user-declared
```

和：

```text
user-provided
```

`= default` 至少明确属于 **user-declared**。

这正是本题的关键。

## 为什么 A 错

A 认为：

```text
= default 不算用户声明
```

这是错误的。

只要你明确写出了：

```cpp
~A() = default;
```

这个析构函数就是 user-declared destructor。

而 user-declared destructor 会抑制隐式 move constructor 的声明。

## 为什么 C 错

```cpp
A() = default;
```

通常正是在明确要求编译器生成默认构造函数。

它不会因为 `= default` 就自动被删除。

当然，如果类的成员或基类导致默认构造本身无法成立，最终默认构造函数可能被定义为 deleted；但本题没有这种条件。

## 为什么 D 错

用户声明析构函数会影响 move 的隐式生成，但并不等价于“不能拷贝”。

本题的 copy constructor 仍然可以由编译器隐式声明。

因此：

```cpp
A a;
A b = a;
```

仍然可以成立。

## 考试快速判断

看到：

```cpp
~T() = default;
```

不要把它当成“没写析构函数”。

应立即标记：

```text
user-declared destructor
→ suppress implicit move ctor / move assignment
```

---

# 第 2 题：有名字的右值引用

```cpp
class A {};

void f(const A&) {
    cout << "L";
}

void f(A&&) {
    cout << "R";
}

void g(A&& x) {
    f(x);
    f(std::move(x));
}

g(A{});
```

用户答案：**B**  
正确答案：**B**  
结果：✅

输出：

```text
LR
```

## 最重要的规则

虽然变量 `x` 的类型是：

```cpp
A&&
```

但表达式：

```cpp
x
```

本身是一个**左值表达式**。

这是 C++ 中极高频的陷阱：

> “变量的类型是右值引用”不等于“这个有名字的变量表达式仍然是右值”。

只要引用有名字：

```cpp
x
```

使用这个名字时，它就是左值表达式。

## 第一次调用

```cpp
f(x);
```

`x` 是左值。

因此：

```cpp
f(A&&)
```

不能绑定左值。

而：

```cpp
f(const A&)
```

可以绑定。

所以输出：

```text
L
```

## 第二次调用

```cpp
f(std::move(x));
```

`std::move(x)` 将 `x` 转换成 xvalue。

于是：

```cpp
f(A&&)
```

成为更好的匹配。

输出：

```text
R
```

总输出：

```text
LR
```

## 为什么这个规则很重要

移动构造函数本身就经常需要显式 `std::move` 成员：

```cpp
T(T&& other)
    : member(std::move(other.member))
{}
```

因为函数体里的：

```cpp
other
```

虽然类型是：

```cpp
T&&
```

但 `other` 这个表达式仍然是左值。

## 考试快速判断

看到：

```cpp
void f(T&& x)
```

立刻写：

```text
类型：T&&
表达式 x：lvalue
```

看到：

```cpp
std::move(x)
```

再改成：

```text
xvalue
```

---

# 第 3 题：deleted move constructor

```cpp
class A {
public:
    A() = default;

    A(const A&) {
        cout << "copy";
    }

    A(A&&) = delete;
};

A a;
A b = std::move(a);
```

用户答案：**A**  
正确答案：**B**  
结果：❌

## 这是本轮最关键的一题

它和第 1 轮的：

```cpp
class A {
public:
    A(const A&);
};
```

完全不是同一种情况。

### 情况一：move constructor 根本不存在

```cpp
class A {
public:
    A(const A&);
};
```

因为用户声明了 copy constructor，编译器不会隐式声明 move constructor。

于是当：

```cpp
A b = std::move(a);
```

发生重载决议时，候选中没有：

```cpp
A(A&&)
```

但还有：

```cpp
A(const A&)
```

而 `const A&` 可以绑定右值。

所以最终发生 copy。

### 情况二：move constructor 存在，但被 delete

本题：

```cpp
A(A&&) = delete;
```

意味着：

```text
这个函数确实被声明了
```

只是：

```text
禁止真正调用它
```

进行：

```cpp
A b = std::move(a);
```

时，候选函数包括：

```cpp
A(const A&)
A(A&&)
```

对于右值/xvalue：

```cpp
A(A&&)
```

是更好的匹配。

于是重载决议首先选择：

```cpp
A(A&&)
```

然后才发现：

```text
selected function is deleted
```

所以程序编译失败。

## 关键原则

> `= delete` 不会让函数从重载集合中消失。

这是选择题中非常重要的一条。

正确分析顺序是：

```text
1. 函数是否参与重载决议？
2. 哪个候选是 best viable function？
3. 最后再检查它是否 deleted / inaccessible。
```

不是：

```text
看到 = delete → 直接忽略这个函数
```

## 为什么 A 错

A 相当于假设：

```text
deleted move = no move
```

这是错误的。

本题的 move constructor 是存在的，而且因为参数是 `A&&`，对 `std::move(a)` 比 `const A&` 匹配更好。

它被选中后，程序才因为 deleted 而编译失败。

## 为什么 C 错

编译器不会因为更优候选被 deleted，就“退回去”重新选择次优 copy constructor。

## 为什么 D 错

这是编译期错误，而不是运行期 UB。

## 考试快速判断

必须区分：

```text
没有 T(T&&)
```

和：

```text
T(T&&) = delete
```

前者：

```text
可能 fallback 到 const T& copy
```

后者：

```text
move 仍参与重载
若它是最佳匹配 → 编译失败
```

---

# 第 4 题：名字隐藏与虚函数

```cpp
class Base {
public:
    virtual void f(int) {
        cout << "B";
    }
};

class Derived : public Base {
public:
    void f(double) {
        cout << "D";
    }
};

Derived d;
Base* p = &d;

d.f(1);
p->f(1);
```

用户答案：**C**  
正确答案：**C**  
结果：✅

输出：

```text
DB
```

这题同时考两个机制：

1. name hiding（名字隐藏）；
2. virtual override（虚函数覆盖）。

## 第一次调用

```cpp
d.f(1);
```

查找从 `Derived` 开始。

`Derived` 中存在：

```cpp
void f(double);
```

一旦派生类声明了同名 `f`，基类中的同名函数：

```cpp
Base::f(int)
```

会被名字隐藏。

这和参数列表是否相同无关。

所以 `d.f(1)` 只看到：

```cpp
Derived::f(double)
```

`int 1` 转换成 `double`，调用 Derived 版本：

```text
D
```

## 第二次调用

```cpp
p->f(1);
```

`p` 的静态类型是：

```cpp
Base*
```

名字查找看到：

```cpp
Base::f(int)
```

然后检查虚函数动态分派。

但：

```cpp
Derived::f(double)
```

并没有 override：

```cpp
Base::f(int)
```

因为参数类型不同。

所以虚函数表中没有 Derived 对这个槽位的覆盖。

最终调用：

```cpp
Base::f(int)
```

输出：

```text
B
```

最终：

```text
DB
```

## 高频误区

很多人把：

```text
同名
```

错误理解成：

```text
override
```

实际上 override 要求函数签名满足覆盖规则。

本题：

```cpp
f(int)
f(double)
```

不是 override。

## 如何重新引入 Base 重载

可以写：

```cpp
class Derived : public Base {
public:
    using Base::f;
    void f(double);
};
```

这样直接对 `Derived` 对象调用时，Base 的重载集合会重新参与。

---

# 第 5 题：虚函数 + 默认参数

```cpp
class Base {
public:
    virtual void f(int x = 1) {
        cout << "B" << x;
    }
};

class Derived : public Base {
public:
    void f(int x = 2) override {
        cout << "D" << x;
    }
};

void call(Base& b) {
    b.f();
}

Derived d;
call(d);
```

用户答案：**C**  
正确答案：**C**  
结果：✅

输出：

```text
D1
```

这一题你已经稳定掌握。

需要继续坚持把一个调用拆成两个问题。

## 问题 1：默认参数取哪个

在：

```cpp
void call(Base& b)
```

内部，表达式：

```cpp
b
```

的静态类型是：

```cpp
Base&
```

默认参数是编译期绑定，因此使用：

```cpp
Base::f(int x = 1)
```

所以参数确定为：

```text
1
```

## 问题 2：函数体执行哪个

实际传入：

```cpp
Derived d;
call(d);
```

`b` 引用的动态对象是 `Derived`。

由于 `f` 是 virtual，函数体动态分派到：

```cpp
Derived::f
```

最终相当于：

```cpp
Derived::f(1);
```

输出：

```text
D1
```

## 一句话规则

```text
默认参数：静态绑定
虚函数体：动态绑定
```

---

# 第 6 题：虚析构

```cpp
class Base {
public:
    virtual ~Base() {
        cout << "B";
    }
};

class Derived : public Base {
public:
    ~Derived() {
        cout << "D";
    }
};

Base* p = new Derived;
delete p;
```

用户答案：**D**  
正确答案：**D**  
结果：✅

输出：

```text
DB
```

## 核心规则

基类析构函数是：

```cpp
virtual ~Base()
```

因此：

```cpp
Base* p = new Derived;
delete p;
```

是合法的多态删除。

析构顺序始终与构造顺序相反。

构造：

```text
Base → Derived
```

析构：

```text
Derived → Base
```

所以输出：

```text
DB
```

## 和上一轮第 4 题对比

上一轮：

```cpp
~Base();
```

非 virtual：

```text
通过 Base* delete Derived → UB
```

本轮：

```cpp
virtual ~Base();
```

于是：

```text
Derived::~Derived()
Base::~Base()
```

正常执行。

## 工程规则

只要一个基类打算用于多态，并允许：

```cpp
delete base_ptr;
```

通常就应提供 virtual destructor。

---

# 第 7 题：C++17 强制拷贝消除

```cpp
class A {
public:
    A() {
        cout << "D";
    }

    A(const A&) {
        cout << "C";
    }

    A(A&&) {
        cout << "M";
    }
};

A make() {
    return A{};
}

A a = make();
```

用户答案：**B**  
正确答案：**A**  
结果：❌

实际只输出：

```text
D
```

## 为什么不是 `DM`

如果用较老的“先构造临时对象，再 move”的思路，很容易得到：

```text
D → M
```

但 C++17 对同类型 prvalue 初始化引入了 guaranteed copy elision 语义。

本题：

```cpp
return A{};
```

中的 `A{}` 是 prvalue。

在 C++17 语义下，它不需要先创建一个独立临时对象再 move 到返回值对象。

结果对象可以直接在最终存储位置构造。

然后：

```cpp
A a = make();
```

同样不需要再从一个独立的返回临时对象 move 到 `a`。

最终实际只需要一次：

```cpp
A()
```

所以输出：

```text
D
```

## 一个更有冲击力的例子

在 C++17：

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

仍然可以成立。

因为这个场景并不需要真正调用 copy ctor 或 move ctor。

## 必须区分 guaranteed copy elision 和 NRVO

### Guaranteed copy elision

例如：

```cpp
return A{};
```

C++17 中是语言保证的同类型 prvalue 场景。

### NRVO

例如：

```cpp
A make() {
    A x;
    return x;
}
```

这里返回的是一个有名字的局部变量：

```cpp
x
```

这是 NRVO 场景。

NRVO 仍然是允许但不强制的优化。

如果 NRVO 没发生，才可能进一步涉及 move/copy。

这是非常常见的选择题区分点。

## 考试快速判断

看到：

```cpp
return T{};
```

且考试基线为 C++17：

优先想到：

```text
guaranteed copy elision
```

不要自动脑补：

```text
临时对象 → move → 返回对象 → move → 目标对象
```

这是旧式对象模型直觉。

---

# 第 8 题：声明有参构造后是否还有默认构造

```cpp
class A {
public:
    A(int x) {
    }
};

A a;
```

用户答案：**C**  
正确答案：**C**  
结果：✅

## 核心规则

如果类没有任何用户声明的构造函数，编译器才可能隐式声明默认构造函数。

本题已经有：

```cpp
A(int x)
```

这是用户声明的构造函数。

因此编译器不会再自动补：

```cpp
A();
```

所以：

```cpp
A a;
```

找不到无参构造函数，编译失败。

## 如果希望两种构造都支持

可以显式写：

```cpp
class A {
public:
    A() = default;
    A(int x) {}
};
```

这也是 `= default` 的典型用途。

## 为什么 A 错

“有了有参构造，编译器仍自动生成无参构造”是错误的。

用户声明任意构造函数以后，隐式默认构造函数就不再自动声明。

## 为什么 B 错

```cpp
A(int x)
```

没有写：

```cpp
A(int x = 0)
```

所以并不存在默认实参。

编译器不会自动给参数补零。

## 为什么 D 错

这是编译期错误，不是 UB。

---

# 本轮知识结构总结

## 1. `= default` 不等于“没写”

```cpp
~T() = default;
```

仍然是：

```text
user-declared destructor
```

会影响 move special members 的隐式生成。

## 2. “move 不存在”与“move = delete”必须严格区分

### move 不存在

```text
std::move(x)
→ 没有 T(T&&)
→ const T& 可能接管
→ 可能发生 copy
```

### move 被 delete

```text
std::move(x)
→ T(T&&) 参与 overload resolution
→ 它通常是最佳匹配
→ 选中后发现 deleted
→ 编译失败
```

## 3. 右值引用变量的名字本身是左值

```cpp
T&& x = ...;
```

则：

```text
x             → lvalue
std::move(x)  → xvalue
```

## 4. C++17 `return T{}` 不要再机械数 move 次数

```cpp
T make() {
    return T{};
}
```

同类型 prvalue 场景属于 guaranteed copy elision。

## 5. 多态部分本轮表现稳定

你已经连续正确处理：

```text
name hiding
virtual dispatch
default argument static binding
virtual destructor
```

下一轮可以减少纯规则复测，开始加入：

- Rule of Three / Five / Zero；
- copy/move assignment；
- 成员导致 special member 被 implicitly deleted；
- `noexcept` 对 `std::vector` 搬迁策略的影响；
- `explicit`、转换构造与重载；
- 引用生命周期与临时对象；
- C++17 copy-elision / NRVO 对比。
