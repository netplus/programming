# C++ 中高级模拟选择题（一）

> 默认按 **C++17** 规则。共 40 题，均为单选题。
>
> 建议先独立完成，再展开每题下方的“答案与解析”。

## 题目分布

- 1～5：函数重载、默认参数、`const`、名字查找
- 6～14：构造、析构、拷贝/移动、对象生命周期
- 15～22：继承、虚函数、多态、对象模型
- 23～30：指针、引用、值类别、`auto`、`decltype`、模板
- 31～36：STL 容器与算法
- 37～40：智能指针、RAII、RTTI、运算符规则

---

## 1. 整型提升与函数重载

```cpp
void f(int)  { }
void f(long) { }

short x = 1;
f(x);
```

调用的是哪一个重载？

A. `f(int)`  
B. `f(long)`  
C. 二义性，编译失败  
D. 取决于平台上 `short` 和 `long` 的大小

<details>
<summary>答案与解析</summary>

**答案：A**

`short` 传参时会发生**整型提升（integral promotion）**。只要 `int` 能表示 `short` 的全部取值，`short` 就优先提升为 `int`。在通常平台上满足这一条件，因此 `short -> int` 属于“提升”，而 `short -> long` 属于普通的整型转换。

重载决议中，标准转换序列的优先级大致可理解为：

1. 精确匹配；
2. 提升；
3. 普通转换。

所以 `f(int)` 胜出。

**考点：** 不要只看“两个目标类型都能装下 short”，重载决议关心的是**转换等级**。

</details>

---

## 2. 默认参数会不会影响重载二义性

```cpp
void f(int);
void f(int, int = 0);

f(10);
```

结果是：

A. 调用 `f(int)`  
B. 调用 `f(int, int)`  
C. 二义性，编译失败  
D. 只有链接阶段才能确定

<details>
<summary>答案与解析</summary>

**答案：C**

调用 `f(10)` 时：

- `f(int)` 可行，参数是精确匹配；
- `f(int, int = 0)` 也可行，第二个参数由默认实参补齐，第一个参数同样是精确匹配。

默认实参只负责让第二个函数成为**可行函数（viable function）**，并不会因为“少用了一个参数”而让第一个函数获得更高优先级。

两个候选在实际参与重载决议的第一个实参上同样好，因此无法选出唯一最佳函数，形成二义性。

**考点：** 默认参数不是函数签名的一部分，但它会改变一个重载是否可调用，从而制造二义性。

</details>

---

## 3. `const` 成员函数重载

```cpp
struct A {
    void f()       { std::cout << "N"; }
    void f() const { std::cout << "C"; }
};

A a;
const A ca;
a.f();
ca.f();
```

输出是：

A. `NN`  
B. `NC`  
C. `CC`  
D. 编译失败

<details>
<summary>答案与解析</summary>

**答案：B**

成员函数尾部的 `const` 会参与成员函数重载。

可以把两者粗略理解为隐藏的 `this` 参数不同：

- 非 `const` 成员函数接收 `A* this`；
- `const` 成员函数接收 `const A* this`。

非 `const` 对象 `a` 调用时，非 `const` 版本是更好的匹配，因此输出 `N`；`const A` 对象不能调用会修改对象状态的非 `const` 成员函数，只能调用 `f() const`，输出 `C`。

所以最终是 `NC`。

**考点：** `const` 对象只能调用 `const` 成员函数，但非 `const` 对象通常优先调用非 `const` 重载。

</details>

---

## 4. 派生类的同名函数隐藏基类重载

```cpp
struct Base {
    void f(int) { std::cout << "B"; }
};

struct Derived : Base {
    void f(double) { std::cout << "D"; }
};

Derived d;
d.f(1);
```

输出是：

A. `B`  
B. `D`  
C. 二义性  
D. 编译失败，因为 `int` 不能转为 `double`

<details>
<summary>答案与解析</summary>

**答案：B**

这里最容易误以为编译器会同时比较 `Base::f(int)` 与 `Derived::f(double)`，然后选择精确匹配的基类版本。

实际上在**名字查找**阶段，派生类中只要出现同名成员 `f`，就会隐藏基类中同名的 `f`，不管参数列表是否不同。因此 `Base::f(int)` 根本没有进入普通的重载候选集合。

最终只剩 `Derived::f(double)`，`int` 可以标准转换为 `double`，因此调用它并输出 `D`。

如果希望把基类重载重新引入候选集合，可以写：

```cpp
struct Derived : Base {
    using Base::f;
    void f(double);
};
```

**考点：** 先发生名字查找，再发生重载决议；“隐藏”和“重载”不是一回事。

</details>

---

## 5. `static` 成员函数的限制

以下哪一项是合法的 C++ 声明？

A. `virtual static void f();`  
B. `static void f() const;`  
C. `static void f();`  
D. `static virtual void f() const;`

<details>
<summary>答案与解析</summary>

**答案：C**

静态成员函数不属于某个具体对象，没有隐含的 `this` 指针，因此：

- 不能声明为 `virtual`，因为虚函数动态分派依赖对象；
- 不能带成员函数尾部的 `const`，因为这个 `const` 本质上修饰的是 `this` 所指向的对象。

所以只有 `static void f();` 合法。

**考点：** 看到成员函数尾部 `const`、`virtual`，都要想到它们依赖对象实例；而 `static` 成员函数没有实例级 `this`。

</details>

---

## 6. 构造顺序

```cpp
struct B {
    B() { std::cout << "B"; }
};

struct M {
    M() { std::cout << "M"; }
};

struct D : B {
    M m;
    D() { std::cout << "D"; }
};

D d;
```

输出是：

A. `DMB`  
B. `BDM`  
C. `BMD`  
D. `MDB`

<details>
<summary>答案与解析</summary>

**答案：C**

一个派生类对象的构造顺序是：

1. 虚基类（若有）；
2. 直接基类；
3. 数据成员，按**成员声明顺序**；
4. 当前类构造函数体。

因此这里先构造基类 `B`，再构造成员 `m`，最后进入 `D` 构造函数体，输出 `BMD`。

析构时顺序反过来。

**考点：** 构造函数体并不是对象构造的起点；进入函数体时，基类与成员已经构造完毕。

</details>

---

## 7. 初始化列表的书写顺序能否改变成员构造顺序

```cpp
struct X {
    X(int id) { std::cout << id; }
};

struct A {
    X x1;
    X x2;

    A() : x2(2), x1(1) {}
};

A a;
```

输出是：

A. `21`  
B. `12`  
C. 未定义行为  
D. 编译失败

<details>
<summary>答案与解析</summary>

**答案：B**

数据成员的初始化顺序**只由类中的声明顺序决定**，与构造函数初始化列表的书写顺序无关。

类中先声明 `x1`，后声明 `x2`，因此先构造 `x1(1)`，再构造 `x2(2)`，输出 `12`。

这也是一个常见工程陷阱：如果 `x2` 的初始化依赖 `x1`，或者反过来依赖，必须按成员实际声明顺序检查，而不能被初始化列表的视觉顺序误导。

**考点：** 最佳实践是让初始化列表顺序与成员声明顺序一致，减少误判和编译器警告。

</details>

---

## 8. `explicit` 构造函数

```cpp
struct A {
    explicit A(int) {}
};
```

以下哪一项合法？

A. `A a = 10;`  
B. `A a(10);`  
C. `void f(A); f(10);`  
D. `A a = {10};`

<details>
<summary>答案与解析</summary>

**答案：B**

`explicit` 禁止构造函数被用于大多数隐式转换和复制初始化。

- `A a = 10;`：复制初始化，需要隐式转换，失败；
- `A a(10);`：直接初始化，合法；
- `f(10)`：需要把 `int` 隐式转换为 `A`，失败；
- `A a = {10};`：复制列表初始化，如果选择到 `explicit` 构造函数也不允许。

**考点：** `explicit` 并不是“这个构造函数不能被调用”，而是“不能被隐式地用来做类型转换”。

</details>

---

## 9. C++17 强制复制消除

```cpp
struct A {
    A() = default;
    A(const A&) = delete;
    A(A&&) = delete;
};

A make() {
    return A{};
}

int main() {
    A a = make();
}
```

在 C++17 下结果是：

A. 必须调用拷贝构造，编译失败  
B. 必须调用移动构造，编译失败  
C. 合法，不要求存在可访问的拷贝/移动构造  
D. 是否合法取决于编译器是否开启优化

<details>
<summary>答案与解析</summary>

**答案：C**

从 C++17 开始，在这类 prvalue 场景中，语言规则保证对象可直接在最终存储位置构造，而不是先构造一个临时对象再“优化掉”拷贝。

因此这里并不是“编译器碰巧优化了两次拷贝”，而是语义上不需要调用被删除的拷贝/移动构造函数。

这与较早标准中的 NRVO/RVO“允许优化”概念要区分开。

**考点：** C++17 的某些 copy elision 已经是语言语义的一部分，不依赖 `-O2`。

</details>

---

## 10. 用户声明析构函数对移动操作的影响

```cpp
struct A {
    ~A() {}
};
```

在没有显式声明其他特殊成员函数的前提下，下列说法正确的是：

A. 编译器仍会自动生成移动构造函数  
B. 用户声明析构函数会阻止隐式生成移动构造函数  
C. 编译器不会生成任何拷贝操作  
D. `A` 自动变成不可复制类型

<details>
<summary>答案与解析</summary>

**答案：B**

在 C++11/14/17 的特殊成员函数生成规则中，**用户声明的析构函数会抑制隐式移动构造函数和移动赋值运算符的生成**。

拷贝构造/拷贝赋值在某些情况下仍可能隐式生成，因此并不会自动变成不可复制类型。

这正是 Rule of Five / Rule of Zero 背后的重要动机之一：一旦手工管理资源并自定义析构，通常也需要认真审视拷贝与移动语义。

**考点：** “写了析构函数”不只是析构阶段的事情，它会影响编译器生成其他特殊成员函数。

</details>

---

## 11. `std::move` 到底做了什么

下列说法正确的是：

A. `std::move(x)` 会立即把 `x` 中的资源搬走  
B. `std::move(x)` 本质上是把表达式转换成可绑定到右值引用的形式  
C. `std::move(x)` 会销毁 `x`  
D. `std::move(x)` 保证后续一定调用移动构造函数

<details>
<summary>答案与解析</summary>

**答案：B**

`std::move` 本身并不执行资源搬迁。它本质上是一个类型转换，把表达式转换为 xvalue，从而允许重载决议选择接收右值引用的函数。

真正“移动资源”的动作发生在后续被调用的移动构造函数、移动赋值运算符或其他接收右值的函数内部。

即使写了 `std::move(x)`，也不保证一定发生移动，例如：

- 类型根本没有移动构造；
- 目标重载只能拷贝；
- 对 `const` 对象使用 `std::move`，通常仍无法绑定到 `T&&` 移动构造，因为得到的是 `const T&&`。

**考点：** `std::move` 更准确地理解为“允许移动”，不是“执行移动”。

</details>

---

## 12. `new[]` 与 `delete`

```cpp
A* p = new A[10];
delete p;
```

这段代码的行为是：

A. 正确，`delete` 会自动识别数组长度  
B. 只销毁第一个元素，但仍是定义良好的行为  
C. 未定义行为  
D. 只有当 `A` 有虚析构函数时才正确

<details>
<summary>答案与解析</summary>

**答案：C**

由 `new[]` 分配的数组必须与 `delete[]` 配对；普通 `new` 与 `delete` 配对。

混用分配/释放形式属于未定义行为。其原因不仅仅是“是否调用多个析构函数”，实现还可能需要保存数组元素数量等额外元数据。

正确写法：

```cpp
A* p = new A[10];
delete[] p;
```

现代 C++ 更推荐直接使用容器或智能指针避免手工配对。

**考点：** `new/delete` 与 `new[]/delete[]` 必须严格成对。

</details>

---

## 13. 返回局部变量的引用

```cpp
const int& f() {
    int x = 42;
    return x;
}
```

下列说法正确的是：

A. 安全，因为返回的是 `const` 引用  
B. 安全，因为编译器会自动延长 `x` 生命周期  
C. 返回后引用悬空，继续使用会产生未定义行为  
D. 只有优化模式下才有问题

<details>
<summary>答案与解析</summary>

**答案：C**

`x` 是自动存储期局部变量，离开 `f()` 时生命周期结束。返回的引用仍然指向原来的存储位置，因此成为悬空引用。

`const` 只限制通过该引用修改对象，并不能延长普通局部变量的生命周期。

这与“临时对象绑定到局部 `const T&` 可以延长临时对象生命周期”的规则完全不同，不能混为一谈。

**考点：** 判断引用是否安全，首先看被引用对象的生命周期，而不是看有没有 `const`。

</details>

---

## 14. 临时对象绑定到局部 `const` 引用

```cpp
const std::string& s = std::string("hello");
std::cout << s;
```

在同一作用域中使用 `s`：

A. `s` 一开始就是悬空引用  
B. 临时 `std::string` 的生命周期延长到 `s` 的生命周期结束  
C. 只有 MSVC 支持，标准 C++ 不支持  
D. 编译器必须把它转换为静态对象

<details>
<summary>答案与解析</summary>

**答案：B**

临时对象直接绑定到局部 `const` 左值引用时，该临时对象的生命周期会被延长到引用变量生命周期结束。

因此本例安全。

但要特别注意，这种生命周期延长有边界。例如函数返回一个引用时，不能指望调用者继续延长函数内部临时对象或局部对象的生命周期。

**考点：** 生命周期延长是“特定绑定场景下的规则”，不是看到 `const&` 就自动永久安全。

</details>

---

## 15. 虚函数与非虚函数

```cpp
struct B {
    virtual void f() { std::cout << "Bf"; }
    void g()         { std::cout << "Bg"; }
};

struct D : B {
    void f() override { std::cout << "Df"; }
    void g()          { std::cout << "Dg"; }
};

B* p = new D;
p->f();
p->g();
```

输出是：

A. `BfBg`  
B. `DfDg`  
C. `DfBg`  
D. `BfDg`

<details>
<summary>答案与解析</summary>

**答案：C**

`p` 的**静态类型**是 `B*`，实际指向的对象动态类型是 `D`。

- `f()` 是虚函数，因此运行时根据动态类型分派，调用 `D::f()`；
- `g()` 不是虚函数，因此调用在编译期根据静态类型确定，为 `B::g()`。

所以输出 `DfBg`。

**考点：** 判断函数调用时先问两个问题：这个函数是不是 `virtual`？调用表达式的静态类型是什么？

</details>

---

## 16. 虚函数的默认参数

```cpp
struct B {
    virtual void f(int x = 1) { std::cout << x; }
};

struct D : B {
    void f(int x = 2) override { std::cout << x; }
};

B* p = new D;
p->f();
```

输出是：

A. `1`  
B. `2`  
C. 二义性  
D. 未定义行为

<details>
<summary>答案与解析</summary>

**答案：A**

这是非常高频的虚函数陷阱。

这里其实有两件事：

1. **调用哪个函数体**：虚函数动态绑定，根据对象动态类型，调用 `D::f`；
2. **默认参数取哪个值**：默认参数在编译期根据调用表达式的静态类型绑定，`p` 是 `B*`，所以使用 `B::f` 声明处的默认值 `1`。

最终等价于运行 `D::f(1)`，因此输出 `1`。

**考点：** 虚函数体是动态绑定，默认参数是静态绑定。

</details>

---

## 17. 构造函数中调用虚函数

```cpp
struct B {
    B() { f(); }
    virtual void f() { std::cout << "B"; }
};

struct D : B {
    void f() override { std::cout << "D"; }
};

D d;
```

构造 `d` 时输出：

A. `D`  
B. `B`  
C. 未定义行为  
D. 取决于编译器

<details>
<summary>答案与解析</summary>

**答案：B**

在 `B` 的构造函数执行期间，`D` 的派生部分还没有完成构造。C++ 不会把此时的虚调用分派到尚未构造完成的派生类版本。

因此 `B()` 中的 `f()` 调用 `B::f()`，输出 `B`。

析构阶段有类似规则：当派生部分已经析构后，在基类析构函数中进行虚调用，也不会再分派到已销毁的派生类实现。

**考点：** 构造/析构期间，动态类型可视为当前正在构造/析构的层级，虚分派不会“穿越”到未构造或已析构的派生部分。

</details>

---

## 18. 通过基类指针删除派生对象

```cpp
struct B {
    ~B() { }
};

struct D : B {
    ~D() { }
};

B* p = new D;
delete p;
```

结果是：

A. 一定依次调用 `D::~D()`、`B::~B()`  
B. 只调用 `B::~B()`，但行为完全合法  
C. 未定义行为  
D. 编译期禁止

<details>
<summary>答案与解析</summary>

**答案：C**

如果通过基类指针删除一个实际为派生类的对象，基类析构函数必须是虚函数，否则行为未定义。

正确设计通常是：

```cpp
struct B {
    virtual ~B() = default;
};
```

只要一个类准备作为**多态基类**使用，并可能通过基类指针拥有/删除派生对象，就应高度警惕非虚析构。

**考点：** 虚析构函数的核心目的不是“让析构函数变高级”，而是确保通过基类接口释放派生对象时能完整沿继承链析构。

</details>

---

## 19. 纯虚析构函数

```cpp
struct B {
    virtual ~B() = 0;
};
```

下列说法正确的是：

A. 纯虚析构函数绝对不能有定义  
B. 纯虚析构函数通常仍必须提供函数定义  
C. 声明纯虚析构后，派生类对象永远无法销毁  
D. 纯虚析构不能使类成为抽象类

<details>
<summary>答案与解析</summary>

**答案：B**

析构派生对象时，无论派生类析构函数做了什么，最终都需要调用基类析构函数。因此即使基类析构函数声明为纯虚函数，也通常必须提供定义：

```cpp
B::~B() {}
```

把析构函数声明为纯虚函数可以让基类成为抽象类，同时仍保留通过虚析构正确销毁派生对象的能力。

**考点：** “纯虚”表示类接口层面要求派生类提供最终覆盖/类不可直接实例化，并不等价于“函数永远没有定义”。析构函数是经典例外场景。

</details>

---

## 20. 对象切片

```cpp
struct B {
    virtual void f() { std::cout << "B"; }
};

struct D : B {
    void f() override { std::cout << "D"; }
};

void g(B b) {
    b.f();
}

D d;
g(d);
```

输出是：

A. `D`  
B. `B`  
C. 未定义行为  
D. 编译失败

<details>
<summary>答案与解析</summary>

**答案：B**

`g` 的参数是按值传递的 `B b`。把 `D` 对象复制到 `B` 对象时，只会保留其中的基类子对象，派生类部分被“切掉”，这就是**对象切片（object slicing）**。

进入 `g` 后，`b` 本身就是一个独立的 `B` 对象，其动态类型也为 `B`，因此即使 `f` 是虚函数，也只能调用 `B::f()`。

如果希望保留多态，应传引用或指针：

```cpp
void g(const B& b);
```

**考点：** `virtual` 不能拯救已经发生的对象切片。

</details>

---

## 21. `override` 能发现什么问题

```cpp
struct B {
    virtual void f() const;
};

struct D : B {
    void f() override;
};
```

结果是：

A. 正确覆盖  
B. 编译失败，因为派生类版本缺少 `const`  
C. 自动继承基类的 `const`  
D. 只有运行时才发现错误

<details>
<summary>答案与解析</summary>

**答案：B**

成员函数尾部 `const` 属于函数类型的一部分，`void f()` 与 `void f() const` 不是同一个虚函数签名。

派生类写 `override` 后，编译器必须验证该函数确实覆盖某个基类虚函数。如果签名不一致，直接编译报错。

如果没有 `override`，`D::f()` 只是定义了一个新的同名函数，很容易误以为已经完成覆盖。

**考点：** 现代 C++ 中，所有预期覆盖基类虚函数的实现都应优先写 `override`，它是非常有价值的编译期检查。

</details>

---

## 22. 虚继承解决什么问题

```cpp
struct A { int x; };
struct B : virtual A {};
struct C : virtual A {};
struct D : B, C {};
```

一个 `D` 对象中包含几个 `A` 基类子对象？

A. 0  
B. 1  
C. 2  
D. 由编译器决定

<details>
<summary>答案与解析</summary>

**答案：B**

这是经典菱形继承。

如果 `B`、`C` 都普通继承 `A`，那么 `D` 中会各自通过 `B`、`C` 带入一个 `A` 子对象，共两个。

使用虚继承后，最派生对象 `D` 只保留一个共享的虚基类 `A` 子对象，因此可消除重复的 `A` 子对象及相关二义性。

**考点：** 虚继承解决的是“共享虚基类子对象”问题；它与虚函数虽然都用了 `virtual` 关键字，但属于完全不同的机制。

</details>

---

## 23. `const` 指针辨析

以下声明中，哪一个表示“指针本身不能改变，但可以通过它修改所指向的 `int`”？

A. `const int* p`  
B. `int const* p`  
C. `int* const p`  
D. `const int* const p`

<details>
<summary>答案与解析</summary>

**答案：C**

阅读技巧可以从变量名向外看：

```cpp
int* const p;
```

`p` 先遇到 `const`，说明 `p` 本身是常量；再看到 `*`，说明它是一个指针；最终指向普通 `int`。

因此：

- `p` 不能改指向；
- `*p` 可以修改。

而 `const int* p` / `int const* p` 都表示“指向 const int 的可变指针”。

**考点：** `const` 修饰谁，关键看它与 `*` 的相对位置。

</details>

---

## 24. 左值与右值引用重载

```cpp
void f(const std::string&) { std::cout << "L"; }
void f(std::string&&)      { std::cout << "R"; }

std::string s = "x";
f(s);
f(std::move(s));
```

输出是：

A. `LL`  
B. `LR`  
C. `RR`  
D. `RL`

<details>
<summary>答案与解析</summary>

**答案：B**

命名变量 `s` 是左值，因此第一次调用不能直接绑定到 `std::string&&`，会调用 `const std::string&` 版本，输出 `L`。

`std::move(s)` 把表达式转换为 xvalue，可以绑定到右值引用重载，因此第二次输出 `R`。

注意：一个“名字本身”即使变量类型是 `T&&`，只要它有名字，在表达式中仍是左值；这是进阶题中常见陷阱。

**考点：** 类型是右值引用，不等于这个命名表达式本身就是右值。

</details>

---

## 25. `const` 对象与 `std::move`

```cpp
struct A {
    A(const A&) { std::cout << "C"; }
    A(A&&)      { std::cout << "M"; }
};

const A a;
A b(std::move(a));
```

最可能调用：

A. 移动构造，输出 `M`  
B. 拷贝构造，输出 `C`  
C. 二义性  
D. 必然编译失败

<details>
<summary>答案与解析</summary>

**答案：B**

`a` 是 `const A`，`std::move(a)` 得到的实际类型是 `const A&&`。

普通移动构造函数参数是 `A&&`，它不能绑定到 `const A&&`，因为那样意味着允许通过移动构造修改一个 `const` 源对象。

而 `const A&` 可以绑定右值，因此最终选择拷贝构造函数。

**考点：** 给 `const` 对象套上 `std::move` 往往仍然只能拷贝；移动通常需要修改源对象状态。

</details>

---

## 26. `auto` 的类型推导

```cpp
const int x = 10;
auto a = x;
auto& b = x;
```

正确的是：

A. `a` 是 `const int`，`b` 是 `int&`  
B. `a` 是 `int`，`b` 是 `const int&`  
C. `a` 是 `const int&`，`b` 是 `const int&`  
D. `a`、`b` 都是 `int`

<details>
<summary>答案与解析</summary>

**答案：B**

`auto a = x;` 是按值推导，顶层 `const` 会被丢弃，因此 `a` 是普通 `int`。

`auto& b = x;` 要保留引用绑定关系，因为 `x` 是 `const int`，所以 `b` 推导为 `const int&`。

可以类比模板类型推导：按值参数通常会去掉顶层 `const` 和引用；引用参数则保留被引用对象的 cv 属性。

**考点：** `auto` 是否保留 `const`，要看是按值还是按引用推导。

</details>

---

## 27. `decltype(x)` 与 `decltype((x))`

```cpp
int x = 0;
```

下列说法正确的是：

A. `decltype(x)` 和 `decltype((x))` 都是 `int`  
B. `decltype(x)` 是 `int&`，`decltype((x))` 是 `int`  
C. `decltype(x)` 是 `int`，`decltype((x))` 是 `int&`  
D. 两者均由编译器自行决定

<details>
<summary>答案与解析</summary>

**答案：C**

`decltype` 有一条非常重要的特殊规则：

- 对于未加括号的普通变量名 `x`，`decltype(x)` 直接得到该实体的声明类型，即 `int`；
- `(x)` 是一个表达式，并且是左值表达式，所以 `decltype((x))` 按值类别规则得到 `int&`。

这也是为什么“多一层括号”在 `decltype` 中可能真正改变类型。

**考点：** `decltype(变量名)` 与 `decltype((变量名))` 要分开记忆。

</details>

---

## 28. 转发引用

```cpp
template<class T>
void f(T&& x);
```

当调用：

```cpp
int a = 0;
f(a);
```

模板参数 `T` 推导为：

A. `int`  
B. `int&`  
C. `int&&`  
D. `const int&`

<details>
<summary>答案与解析</summary>

**答案：B**

这里的 `T&&` 位于模板类型推导上下文中，且 `T` 是待推导模板参数，因此它是**转发引用（forwarding reference）**。

当传入左值 `a` 时，`T` 推导为 `int&`。随后发生引用折叠：

```text
T&&
=> int& &&
=> int&
```

所以函数参数最终是左值引用。

**考点：** `T&&` 不总是“右值引用”；在特定模板推导上下文里它可能是转发引用。

</details>

---

## 29. 模板参数推导

```cpp
template<class T>
T add(T a, T b) {
    return a + b;
}

add(1, 2.0);
```

结果是：

A. `T` 自动推导为 `double`  
B. `T` 自动推导为 `int`  
C. 模板参数推导失败  
D. 返回类型自动推导为 `long double`

<details>
<summary>答案与解析</summary>

**答案：C**

模板参数推导通常不会像普通函数调用那样，先任意做数值转换再寻找一个“共同 T”。

第一个实参 `1` 要求 `T = int`，第二个实参 `2.0` 要求 `T = double`，推导结果冲突，因此模板参数推导失败。

可以显式指定：

```cpp
add<double>(1, 2.0);
```

此时 `1` 再转换为 `double`。

**考点：** 模板参数推导和普通重载转换规则不是一回事；先推导出模板，再进行允许的实参转换。

</details>

---

## 30. 通过基类引用向下转型

```cpp
struct B { virtual ~B() = default; };
struct D : B {};

B b;
B& r = b;
D& d = dynamic_cast<D&>(r);
```

运行到 `dynamic_cast` 时：

A. 返回空引用  
B. 抛出 `std::bad_cast`  
C. 编译失败，因为引用不能 `dynamic_cast`  
D. 未定义行为

<details>
<summary>答案与解析</summary>

**答案：B**

`B` 是多态类型，因为它含有虚函数（这里是虚析构）。因此可以使用 `dynamic_cast` 做运行时检查的向下转型。

实际对象动态类型只是 `B`，并不是 `D`，转换失败。

- 如果目标是指针 `dynamic_cast<D*>(p)`，失败返回 `nullptr`；
- 如果目标是引用 `dynamic_cast<D&>(r)`，失败抛出 `std::bad_cast`。

**考点：** 指针失败返回空，引用失败抛异常，是 RTTI 高频区分点。

</details>

---

## 31. `vector::reserve` 与 `vector::resize`

```cpp
std::vector<int> v;
v.reserve(100);
```

此时正确的是：

A. `v.size() == 100`  
B. 可以安全访问 `v[50]`  
C. `v.capacity() >= 100`，但 `v.size()` 仍为 0  
D. `reserve` 会创建 100 个值为 0 的元素

<details>
<summary>答案与解析</summary>

**答案：C**

`reserve(100)` 只保证容量至少达到 100，用于减少后续扩容次数；它**不会改变容器的元素个数**。

因此：

```cpp
v.size() == 0;
v.capacity() >= 100;
```

此时访问 `v[50]` 越过实际元素范围，属于未定义行为。

如果需要真正创建 100 个元素，应使用：

```cpp
v.resize(100);
```

**考点：** `size` 是已经存在的元素数量，`capacity` 是当前分配空间最多能容纳多少元素。

</details>

---

## 32. `vector` 扩容与迭代器失效

```cpp
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);
```

关于 `it`，正确的是：

A. 永远有效  
B. 只要添加的是尾部元素，旧迭代器一定有效  
C. 如果 `push_back` 触发重新分配，`it` 会失效  
D. `vector` 的迭代器从不失效

<details>
<summary>答案与解析</summary>

**答案：C**

`vector` 的元素存储要求连续。若 `push_back` 时容量不足，容器需要申请一块更大的连续内存，把现有元素迁移过去，再释放旧内存。

发生重新分配后，指向旧存储区的迭代器、指针、引用都会失效。

如果没有触发重新分配，尾部插入通常不会让所有旧元素迭代器都失效，但 `end()` 会变化。

**考点：** 看到 `vector` 修改操作，先判断是否可能发生 reallocation。

</details>

---

## 33. `map::operator[]`

```cpp
std::map<std::string, int> m;
int x = m["abc"];
```

执行后正确的是：

A. 抛出 `std::out_of_range`  
B. `m` 仍然为空  
C. `m` 中插入键 `"abc"`，对应值进行值初始化，`x` 为 0  
D. 行为未定义

<details>
<summary>答案与解析</summary>

**答案：C**

`std::map::operator[]` 不只是查找。如果键不存在，它会插入该键，并对 mapped value 做值初始化。

这里 value 类型是 `int`，值初始化得到 0，因此：

```cpp
m["abc"] == 0;
```

如果只是查询且不希望插入，可用 `find`、`contains`（C++20）或 `at`；其中 `at` 对不存在的键会抛异常。

**考点：** 在只读逻辑里滥用 `operator[]` 可能悄悄改变 `map`。

</details>

---

## 34. `unordered_map` rehash

`std::unordered_map` 发生 `rehash` 后，下列说法正确的是：

A. 所有迭代器失效，但元素的引用和指针通常仍保持有效  
B. 所有迭代器、引用、指针都必然失效  
C. 什么都不会失效  
D. 只有 `end()` 失效

<details>
<summary>答案与解析</summary>

**答案：A**

`unordered_map` 的 rehash 会重新组织 bucket，因此迭代器遍历结构发生变化，所有迭代器失效。

但标准容器保证 rehash 本身不会使指向元素的引用和指针失效；元素对象本身并没有因为换 bucket 就必须重新构造到另一个元素地址语义中。

这与 `vector` reallocation 的失效规则不同。

**考点：** 不同容器的迭代器/引用失效规则必须分别记，不能用一套直觉套所有 STL 容器。

</details>

---

## 35. `vector::erase` 后的迭代器

对于：

```cpp
std::vector<int> v = {1,2,3,4};
auto it = v.begin() + 1; // 指向 2
v.erase(it);
```

正确的是：

A. 原 `it` 仍指向元素 3，可继续安全使用  
B. 原 `it` 已失效；`erase` 的返回值可用于获得后继位置  
C. 所有早于 `it` 的迭代器也必然失效  
D. `erase` 不会移动任何元素

<details>
<summary>答案与解析</summary>

**答案：B**

对 `vector` 删除中间元素时，后续元素通常需要向前移动填补空洞。因此指向被删除位置以及其后的迭代器都会失效。

标准的安全遍历删除写法通常是：

```cpp
auto it = v.begin();
while (it != v.end()) {
    if (need_erase(*it))
        it = v.erase(it);
    else
        ++it;
}
```

`erase` 返回删除位置之后的新有效迭代器。

**考点：** 删除后不要理所当然地 `++it`，先看容器 API 返回了什么。

</details>

---

## 36. 排序比较器的要求

```cpp
std::sort(v.begin(), v.end(),
          [](int a, int b) { return a <= b; });
```

这段比较器的主要问题是：

A. lambda 不能传给 `sort`  
B. 比较器应满足严格弱序，`a <= b` 不满足该要求  
C. `sort` 只能升序  
D. `sort` 只能使用函数指针

<details>
<summary>答案与解析</summary>

**答案：B**

`std::sort` 要求比较器建立**严格弱序（strict weak ordering）**。

一个最直观的要求是比较器必须满足非自反性：

```cpp
comp(x, x) == false
```

但 `x <= x` 为真，因此这个比较器违反要求。

正确的升序比较器应使用：

```cpp
return a < b;
```

违反排序算法对比较器的契约会导致算法行为不再受标准保证。

**考点：** 排序比较器通常写 `<`，而不是 `<=`。

</details>

---

## 37. `unique_ptr` 的语义

```cpp
std::unique_ptr<int> p1(new int(1));
std::unique_ptr<int> p2 = p1;
```

结果是：

A. 两个指针共同拥有同一个对象  
B. `p1` 自动置空，等价于移动  
C. 编译失败，因为 `unique_ptr` 不可拷贝  
D. 运行时双重释放

<details>
<summary>答案与解析</summary>

**答案：C**

`std::unique_ptr` 表示**独占所有权**，因此复制构造和复制赋值被删除。

若要转移所有权，应显式移动：

```cpp
std::unique_ptr<int> p2 = std::move(p1);
```

移动后，`p2` 获得资源所有权，`p1` 处于有效但不再拥有该对象的状态，通常为空。

**考点：** `unique_ptr` 可以移动，不能复制；这正是类型系统对独占所有权的约束。

</details>

---

## 38. `shared_ptr` 循环引用

```cpp
struct B;
struct A { std::shared_ptr<B> b; };
struct B { std::shared_ptr<A> a; };
```

若 `A` 与 `B` 的对象彼此通过上述成员持有，主要风险是：

A. 编译失败  
B. 引用计数可能永远不能降到 0，造成资源泄漏  
C. `shared_ptr` 会自动检测环并回收  
D. 必然栈溢出

<details>
<summary>答案与解析</summary>

**答案：B**

`shared_ptr` 使用引用计数管理对象。如果两个对象互相持有对方的 `shared_ptr`，即使外部所有引用都释放，两边仍各自维持至少一个强引用，引用计数无法降到 0。

解决方式通常是在非所有权方向使用 `std::weak_ptr`：

```cpp
struct B { std::weak_ptr<A> a; };
```

`weak_ptr` 观察对象但不增加强引用计数。

**考点：** 引用计数不是垃圾回收器，它无法自动解决一般的环形所有权图。

</details>

---

## 39. RAII 与异常

```cpp
void f() {
    std::lock_guard<std::mutex> g(m);
    may_throw();
}
```

如果 `may_throw()` 抛出异常并沿调用栈传播，下列说法正确的是：

A. `g` 不会析构，因此互斥锁永久锁死  
B. 正常栈展开过程中会调用 `g` 的析构函数，从而释放锁  
C. C++ 异常传播时不会析构局部对象  
D. 必须手工写 `try/catch` 才能让 `g` 析构

<details>
<summary>答案与解析</summary>

**答案：B**

RAII 的核心思想是把资源生命周期绑定到对象生命周期。

`lock_guard` 构造时加锁，析构时解锁。发生异常时，C++ 会执行**栈展开（stack unwinding）**，已经完成构造的自动对象会按逆序析构，因此 `g` 的析构函数会运行，锁得到释放。

这也是 RAII 比“手工 acquire + release”更可靠的重要原因。

**考点：** RAII 不只管理内存，还可管理锁、文件描述符、socket、事务等任何需要成对获取/释放的资源。

</details>

---

## 40. 哪些运算符不能被重载

以下哪一个运算符**不能**被用户定义类型重载？

A. `+`  
B. `[]`  
C. `()`  
D. `.`

<details>
<summary>答案与解析</summary>

**答案：D**

C++ 允许重载大量运算符，例如 `+`、`[]`、`()`、`<<` 等，但有少数运算符不能被重载，常见的包括：

- `.` 成员访问；
- `::` 作用域解析；
- `?:` 条件运算符；
- `sizeof`；
- `typeid` 等语言级操作。

另外，即使一个运算符允许重载，也不能通过重载改变它的优先级、结合性或操作数个数这些语法规则。

**考点：** 运算符重载只是为已有语法形式提供用户类型语义，不是在创建新的语法规则。

</details>

---

# 答案速查

<details>
<summary>展开答案表</summary>

| 题号 | 答案 | 题号 | 答案 | 题号 | 答案 | 题号 | 答案 |
|---:|:---:|---:|:---:|---:|:---:|---:|:---:|
| 1 | A | 11 | B | 21 | B | 31 | C |
| 2 | C | 12 | C | 22 | B | 32 | C |
| 3 | B | 13 | C | 23 | C | 33 | C |
| 4 | B | 14 | B | 24 | B | 34 | A |
| 5 | C | 15 | C | 25 | B | 35 | B |
| 6 | C | 16 | A | 26 | B | 36 | B |
| 7 | B | 17 | B | 27 | C | 37 | C |
| 8 | B | 18 | C | 28 | B | 38 | B |
| 9 | C | 19 | B | 29 | C | 39 | B |
| 10 | B | 20 | B | 30 | B | 40 | D |

</details>

---

# 做完这套题后应该掌握的判断框架

面对中高级 C++ 选择题，不建议只靠记忆零散结论。可以优先按下面几个维度判断：

1. **先看编译期还是运行期。** 重载、默认参数、名字查找、模板推导主要发生在编译期；虚函数动态绑定才进入运行期。
2. **先看静态类型，再看动态类型。** 非虚调用主要由静态类型决定；虚调用再结合动态类型。
3. **先看对象生命周期。** 指针/引用题里，能不能访问往往不是类型问题，而是对象是否仍然存活。
4. **区分“对象个数”和“内存容量”。** `vector::size` 与 `capacity`、`reserve` 与 `resize` 是高频组合。
5. **区分资源所有权。** `unique_ptr` 是独占所有权，`shared_ptr` 是共享强所有权，`weak_ptr` 是非拥有观察者。
6. **看到特殊成员函数就联想到整组规则。** 自定义析构、拷贝、移动通常不能孤立分析，应联系 Rule of Five / Rule of Zero。
7. **看到模板和 `auto` 就按类型推导规则走，不要凭“会自动转换”的直觉。**

下一套题建议重点强化：

- 多重继承、虚继承的构造顺序；
- `operator=`、拷贝构造、移动构造的触发条件；
- `noexcept` 与容器移动策略；
- lambda 捕获与生命周期；
- `shared_ptr` 控制块与 `enable_shared_from_this`；
- STL 迭代器失效专题；
- 模板重载、SFINAE 基础；
- 异常安全保证（basic/strong/nothrow）。
