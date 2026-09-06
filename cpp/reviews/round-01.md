# C++ 中高级选择题训练：第 1 轮复盘

> 语义基线：C++17

## 成绩

用户答案：`1B, 2C, 3A, 4B, 5A, 6C, 7B, 8D`

正确答案：`1B, 2C, 3A, 4C, 5A, 6C, 7A, 8C`

成绩：**5 / 8，62.5%**

### 本轮暴露出的薄弱点

1. 通过基类指针删除派生对象时，基类析构函数是否为 `virtual`；
2. `std::move` 只是值类别转换，不等于一定发生移动构造；
3. 虚函数是动态绑定，但默认参数是静态绑定。

---

## 第 1 题：`const` 与成员函数重载

```cpp
class A {
public:
    void f()       { cout << "1"; }
    void f() const { cout << "2"; }
};

const A a;
a.f();
```

你的答案：**B**  
正确答案：**B**  
结果：✅ 正确

### 核心规则

成员函数末尾的 `const` 不是装饰，它会参与重载。

可以把两个成员函数近似理解成：

```cpp
void f(A* this);
void f(const A* this);
```

实际语法当然不是这样，但这种理解非常适合考试判断。

对象 `a` 的类型是：

```cpp
const A
```

因此它的隐式 `this` 指针可以理解为：

```cpp
const A* const this
```

于是：

```cpp
void f()
```

要求一个可修改的 `A` 对象，不能作用于 `const A`；而：

```cpp
void f() const
```

可以作用于 `const A`。

最终调用：

```cpp
A::f() const
```

输出：

```text
2
```

### 为什么其它选项不对

- A：非 `const` 成员函数不能通过 `const` 对象调用；
- C：不存在二义性，因为非 `const` 版本甚至不是合法候选；
- D：代码可以正常编译。

### 常见误判

很多人只看到两个函数都叫 `f()`，以为签名完全相同。实际上成员函数的 cv 限定符会参与成员函数重载。

### 考试快速判断

看到：

```cpp
const A a;
a.f();
```

第一反应就是：

> `a` 是 const 对象，只能调用不会修改对象状态的 const 成员函数。

---

## 第 2 题：构造顺序

```cpp
struct B {
    B() { cout << "B "; }
};

struct M {
    M(int x) { cout << x << " "; }
};

struct D : B {
    M m1;
    M m2;

    D() : m2(2), m1(1), B() {
        cout << "D";
    }
};

D d;
```

你的答案：**C**  
正确答案：**C**  
结果：✅ 正确

输出：

```text
B 1 2 D
```

### 核心规则

构造顺序**不由初始化列表书写顺序决定**。

构造一个派生类对象时，顺序可以记成：

1. 虚基类；
2. 直接基类；
3. 成员变量，按**类中声明顺序**；
4. 构造函数体。

本题中：

```cpp
struct D : B {
    M m1;
    M m2;
```

因此顺序必然是：

```text
B -> m1 -> m2 -> D 构造函数体
```

虽然初始化列表写成：

```cpp
D() : m2(2), m1(1), B()
```

也不会改变真实构造顺序。

### 为什么标准这样设计

对象销毁时顺序是构造顺序的逆序。

如果初始化列表可以随意改变成员构造顺序，那么同一个类的对象布局与生命周期管理会变得难以推理。因此 C++ 固定使用声明顺序。

### 为什么其它选项不对

- A：错误地把初始化列表顺序当成构造顺序；
- B：基类正确，但 `m2` 不会先于 `m1`；
- D：成员不会先于基类构造。

### 常见误判

看到：

```cpp
: m2(2), m1(1)
```

就机械认为 `m2` 先构造。

### 考试快速判断

永远回到类定义：

```cpp
M m1;
M m2;
```

谁先声明，谁先构造。

---

## 第 3 题：构造函数中的虚函数调用

```cpp
class Base {
public:
    Base() { f(); }

    virtual void f() {
        cout << "Base";
    }
};

class Derived : public Base {
public:
    void f() override {
        cout << "Derived";
    }
};

Derived d;
```

你的答案：**A**  
正确答案：**A**  
结果：✅ 正确

### 核心规则

虚函数通常意味着运行时动态绑定，但有一个高频例外：

> 在构造函数和析构函数执行期间，虚调用不会分派到尚未构造完成或已经销毁的更派生类部分。

构造 `Derived d` 时：

```text
先执行 Base::Base()
再执行 Derived::Derived()
```

当执行：

```cpp
Base() { f(); }
```

时，`Derived` 子对象还没有完成构造。

因此此时：

```cpp
f();
```

调用的是：

```cpp
Base::f()
```

输出：

```text
Base
```

### 为什么不能调用 `Derived::f()`

因为派生类成员此时可能还没有初始化。

如果 C++ 允许基类构造函数动态调用派生类实现，那么派生类函数可能访问尚未初始化的数据成员，造成严重的生命周期问题。

### 为什么其它选项不对

- B：把普通虚调用规则机械套进了构造阶段；
- C：只调用一次 `f()`，不会连续调用两个版本；
- D：调用普通的非纯虚 `Base::f()` 是定义良好的行为。

### 常见误判

“只要有 `virtual` 就一定调用最派生版本”——这是错误的。

### 考试快速判断

看到：

```cpp
Base::Base() {
    virtual_function();
}
```

先把虚调用暂时理解成：

```cpp
Base::virtual_function();
```

析构阶段也要有类似警觉。

---

## 第 4 题：通过基类指针删除派生对象

```cpp
class Base {
public:
    ~Base() {
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

你的答案：**B**  
正确答案：**C**  
结果：❌ 错误

### 这是本轮第一个重要薄弱点

你选择 B，说明你的推理很可能是：

```text
p 的静态类型是 Base* -> delete 时调用 Base::~Base() -> 输出 B
```

这个推理看起来很自然，但漏掉了更高优先级的一条规则：

> 当通过基类指针删除实际类型为派生类的对象时，如果基类析构函数不是虚函数，行为是未定义的。

本题：

```cpp
Base* p = new Derived;
```

静态类型：

```text
Base*
```

动态类型：

```text
Derived
```

而：

```cpp
~Base()
```

不是虚析构函数。

因此：

```cpp
delete p;
```

是 **Undefined Behavior（未定义行为）**。

正确答案是 C。

### 为什么“可能只输出 B”也不能选 B

某些编译器、某些构建配置下，你实际运行可能真的看到：

```text
B
```

但考试问的是 C++ 标准语义，而不是某一次机器执行现象。

一旦行为属于 UB：

```text
标准不再保证任何结果
```

因此不能说“一定输出 B”。

### 正确设计

基类如果准备被多态删除，通常应该写：

```cpp
class Base {
public:
    virtual ~Base() = default;
};
```

这样：

```cpp
Base* p = new Derived;
delete p;
```

析构顺序才是：

```text
Derived::~Derived()
Base::~Base()
```

即输出：

```text
DB
```

### 为什么其它选项不对

- A：只有虚析构时才能保证 `Derived` 再 `Base`；
- B：某些实现可能呈现这个现象，但标准不保证；
- D：这是合法语法，不是编译错误，问题发生在运行语义层面。

### 考试快速判断

看到三件套：

```cpp
Base* p = new Derived;
delete p;
```

立即检查：

```cpp
virtual ~Base();
```

没有 `virtual`，优先想到 **UB**。

---

## 第 5 题：对象切片

```cpp
class Base {
public:
    virtual void f() {
        cout << "Base";
    }
};

class Derived : public Base {
public:
    void f() override {
        cout << "Derived";
    }
};

void g(Base b) {
    b.f();
}

Derived d;
g(d);
```

你的答案：**A**  
正确答案：**A**  
结果：✅ 正确

### 核心规则

关键不是 `f()` 是否是虚函数，而是参数：

```cpp
void g(Base b)
```

这里是**按值传递**。

调用：

```cpp
g(d);
```

时，会使用 `d` 中的 `Base` 子对象构造一个新的独立对象：

```cpp
Base b
```

`Derived` 特有的部分被丢掉，这就是：

> object slicing，对象切片。

进入 `g()` 后，`b` 已经不是一个“指向 Derived 的 Base 视图”，而是一个真正独立的 `Base` 对象。

因此：

```cpp
b.f();
```

动态类型也是 `Base`，最终调用：

```cpp
Base::f()
```

输出：

```text
Base
```

### 对比引用

如果函数改为：

```cpp
void g(Base& b)
```

再执行：

```cpp
g(d);
```

就不会发生切片。

此时：

```text
静态类型：Base
动态类型：Derived
```

调用虚函数会输出：

```text
Derived
```

### 为什么其它选项不对

- B：只有保持多态对象身份时才会调 `Derived::f()`；按值传递已经切片；
- C：不存在重载二义性；
- D：切片虽然常常不是程序员想要的，但它本身是合法行为，不是 UB。

### 考试快速判断

看到：

```cpp
void f(Base x)
```

然后：

```cpp
Derived d;
f(d);
```

先写下两个字：

```text
切片
```

再分析虚函数。

---

## 第 6 题：`nullptr` 与函数重载

```cpp
void f(int*);
void f(double*);

f(nullptr);
```

你的答案：**C**  
正确答案：**C**  
结果：✅ 正确

### 核心规则

`nullptr` 的类型不是：

```cpp
void*
```

也不是：

```cpp
int
```

它的类型是：

```cpp
std::nullptr_t
```

`std::nullptr_t` 可以通过标准空指针转换转换为各种指针类型。

因此：

```cpp
nullptr -> int*
```

合法；

```cpp
nullptr -> double*
```

也合法。

对于本题两个重载：

```cpp
void f(int*);
void f(double*);
```

两个转换序列没有谁明显优于另一个，因此重载决议无法选出唯一最佳函数。

结果是：

```text
ambiguous
```

编译失败。

### 为什么其它选项不对

- A：`int*` 没有比 `double*` 更特殊；
- B：同理；
- D：`nullptr` 设计出来的主要目的之一就是安全表示空指针。

### 一个高频对比

```cpp
void f(int);
void f(int*);

f(nullptr);
```

这里不会二义，因为 `nullptr` 可以转换成 `int*`，但不能像整数 `0` 那样隐式转换成 `int`。

### 考试快速判断

看到 `nullptr`：

1. 先记住类型是 `std::nullptr_t`；
2. 看有哪些“指针类型”重载；
3. 如果多个互不相关的指针类型都可行，经常会形成二义性。

---

## 第 7 题：`std::move` 是否一定调用移动构造

```cpp
class A {
public:
    A() = default;

    A(const A&) {
        cout << "copy";
    }
};

A a;
A b = std::move(a);
```

你的答案：**B**  
正确答案：**A**  
结果：❌ 错误

### 这是非常典型的中高级 C++ 考点

首先必须建立一个牢固认识：

> `std::move` 自己不移动任何东西。

它的核心作用可以近似理解为：

```cpp
static_cast<A&&>(a)
```

也就是：

> 把表达式转换成可以被移动语义利用的右值类别（更准确地说是 xvalue）。

真正“移动不移动”，取决于后续重载决议有没有找到合适的移动操作。

### 本题为什么没有隐式移动构造函数

类中显式声明了：

```cpp
A(const A&)
```

也就是用户声明的拷贝构造函数。

在这种情况下，编译器不会再自动生成通常意义上的隐式移动构造函数：

```cpp
A(A&&)
```

因此候选里实际上没有可用的移动构造函数。

### 那 `std::move(a)` 怎么还能构造 `b`

因为：

```cpp
A(const A&)
```

中的：

```cpp
const A&
```

可以绑定到右值。

于是：

```cpp
std::move(a)
```

虽然是右值表达式，但最终仍然可以匹配：

```cpp
A(const A&)
```

因此调用拷贝构造函数，输出：

```text
copy
```

正确答案 A。

### 为什么 B 错

B 的核心误区是：

```text
std::move(x) -> 一定调用移动构造
```

实际关系应该记成：

```text
std::move
   ↓
把表达式变成可被移动操作匹配的值类别
   ↓
进入正常重载决议
   ↓
有移动构造 -> 可能移动
没有移动构造但 const& 可用 -> 仍然拷贝
```

### 为什么其它选项不对

- B：用户声明拷贝构造函数会抑制隐式移动构造函数的生成；
- C：`const A&` 可以绑定右值，因此仍可编译；
- D：这是完全定义良好的行为。

### 考试快速判断

以后看到：

```cpp
std::move(x)
```

不要问：

```text
“是不是移动？”
```

而要问：

```text
“把它变成右值之后，重载决议最终选中了谁？”
```

这是更准确的思维方式。

---

## 第 8 题：虚函数 + 默认参数

```cpp
class Base {
public:
    virtual void f(int x = 1) {
        cout << "Base:" << x;
    }
};

class Derived : public Base {
public:
    void f(int x = 2) override {
        cout << "Derived:" << x;
    }
};

Derived d;
Base* p = &d;

p->f();
```

你的答案：**D**  
正确答案：**C**  
结果：❌ 错误

输出：

```text
Derived:1
```

### 这题实际上混合了两套完全不同的绑定规则

必须把：

```cpp
p->f();
```

拆成两个问题。

### 问题一：到底执行哪个函数体

`p` 的静态类型：

```text
Base*
```

但它实际指向：

```text
Derived
```

因为 `f` 是虚函数：

```cpp
virtual void f(...)
```

所以函数体采用**动态绑定**。

最终调用的是：

```cpp
Derived::f
```

因此输出前半部分一定是：

```text
Derived:
```

### 问题二：参数 `x` 使用哪个默认值

这里是本题真正的陷阱。

默认参数不是虚机制的一部分。

默认参数在编译期根据调用表达式的**静态类型**决定。

表达式是：

```cpp
p->f();
```

而：

```cpp
p
```

的静态类型是：

```cpp
Base*
```

因此编译器看到的是：

```cpp
Base::f(int x = 1)
```

于是调用点在编译期相当于补成：

```cpp
p->f(1);
```

之后运行时虚函数分派再决定函数体：

```cpp
Derived::f(1)
```

所以结果是：

```text
Derived:1
```

正确答案 C。

### 你的 D 为什么很有迷惑性

D：

```text
Derived:2
```

对应的是一种很自然但错误的整体思维：

```text
既然虚函数最后选了 Derived::f，那么默认参数也应该使用 Derived::f 的默认值 2。
```

问题就在于：

> 虚函数体和默认参数根本不属于同一个绑定阶段。

可以记成：

```text
函数体：运行期看动态类型
默认参数：编译期看静态类型
```

### 一个非常值得记住的变化

如果写：

```cpp
Derived* p = &d;
p->f();
```

那么静态类型变为 `Derived*`，默认参数就取 `2`；同时动态调用仍然是 `Derived::f`。

此时才输出：

```text
Derived:2
```

### 为什么其它选项不对

- A `Base:1`：默认参数对，但虚函数体错；
- B `Base:2`：函数体和默认参数都错；
- D `Derived:2`：函数体对，但错误地把默认参数也当成动态绑定。

### 考试快速判断

看到“虚函数 + 默认参数”，强制拆成两行：

```text
函数体看 dynamic type
默认值看 static type
```

这是一个非常高价值的考试口诀。

---

# 本轮知识结构诊断

## 已表现稳定

### 1. `const` 成员函数重载

你能正确利用对象的 cv 属性判断可调用的成员函数。

### 2. 构造顺序

你没有被初始化列表的书写顺序误导，说明已经掌握“基类优先、成员按声明顺序”的核心规则。

### 3. 构造阶段的虚调用

你已经知道构造期间不会向更派生类动态分派。

### 4. 对象切片

你能够识别按值传递造成的 Derived -> Base slicing。

### 5. `nullptr` 重载

你对 `std::nullptr_t` 和指针重载的二义性判断正确。

## 下一轮需要强化

### A. 析构与多态生命周期

重点继续考：

- virtual destructor；
- 析构顺序；
- pure virtual destructor；
- 基类指针删除；
- 构造/析构阶段的虚调用。

### B. 拷贝 / 移动语义

重点继续考：

- `std::move`；
- 用户声明 copy/move 对隐式生成规则的影响；
- `const T&&`；
- move constructor 与 copy constructor 的重载决议；
- moved-from object 的语义。

### C. 静态绑定与动态绑定混合题

重点继续考：

- virtual function；
- default argument；
- overload vs override；
- name hiding；
- static type vs dynamic type。

---

# 本轮最值得记住的三条规则

```text
1. Base* 指向 Derived，并通过 Base* delete：先检查 Base 析构是否 virtual。

2. std::move 不负责移动，只负责把表达式变成可参与移动重载的值类别。

3. 虚函数体看动态类型；默认参数看静态类型。
```

下一轮应在这三条规则上增加组合型陷阱，同时保留少量已掌握知识点用于确认稳定性。
