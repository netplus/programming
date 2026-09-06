# C++ 特殊成员函数：编译器什么时候会自动生成，`= default` 又是什么意思

> 语义基线：C++17
>
> 本文用于补充第 1 轮第 7 题：`std::move` 是否一定调用移动构造函数。

## 1. 先建立整体图像

C++ 类中最常见的特殊成员函数包括：

```cpp
class A {
public:
    A();                       // 默认构造函数
    A(const A&);               // 拷贝构造函数
    A(A&&);                    // 移动构造函数
    A& operator=(const A&);    // 拷贝赋值运算符
    A& operator=(A&&);         // 移动赋值运算符
    ~A();                      // 析构函数
};
```

考试时不要记成“没写，编译器就一定补”。

更准确的思维方式是：

```text
程序员是否声明过某些特殊成员函数？
        ↓
编译器是否还会隐式声明目标函数？
        ↓
如果声明了，它是否能被正常定义？
        ↓
如果参数是右值，重载决议最终选 move 还是 copy？
```

其中移动构造函数 `A(A&&)` 的隐式生成条件最容易考。

---

## 2. `= default` 到底是什么意思

例如：

```cpp
class A {
public:
    A() = default;
};
```

`= default` 的意思是：

> **这个特殊成员函数由我显式声明，但请编译器按默认规则生成它的实现。**

它不是“删除函数”，也不是“什么都不做”。

例如：

```cpp
A() = default;
```

通常表示：使用编译器本来会生成的默认构造行为。

类似地：

```cpp
A(const A&) = default;
A(A&&) = default;
A& operator=(const A&) = default;
A& operator=(A&&) = default;
~A() = default;
```

都表示要求编译器生成对应的默认语义。

### 与 `= delete` 对比

```cpp
A(const A&) = delete;
```

表示：

> 这个函数存在于重载集合中，但禁止使用。

所以：

```cpp
A(const A&) = default;
```

与：

```cpp
A(const A&) = delete;
```

含义完全相反。

### 一个考试很爱考的细节

```cpp
A() = default;
```

仍然属于**用户声明（user-declared）**的构造函数。

也就是说，`= default` 并不等于“我没有声明这个函数”。

这个区别会影响其它特殊成员函数是否还能被隐式声明。

---

## 3. 默认构造函数什么时候不会自动出现

默认构造函数是：

```cpp
A();
```

如果类中**没有任何用户声明的构造函数**，编译器才会考虑隐式声明默认构造函数。

例如：

```cpp
class A {
};
```

编译器会提供一个默认构造函数。

但如果写了：

```cpp
class A {
public:
    A(int);
};
```

那么：

```cpp
A a;
```

通常会编译失败，因为你已经声明了构造函数 `A(int)`，编译器不会再额外帮你生成无参的 `A()`。

如果还想要默认构造函数，可以显式写：

```cpp
class A {
public:
    A() = default;
    A(int);
};
```

### 快速记忆

```text
只要你声明了任意构造函数
→ 编译器就不会再“自动补一个无参默认构造函数”
```

---

## 4. 拷贝构造函数什么时候会出现

典型签名：

```cpp
A(const A&);
```

如果程序员没有显式声明拷贝构造函数，编译器通常会隐式声明一个。

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
A b = a;
```

这里通常使用编译器隐式生成的拷贝构造函数。

不过 C++11 以后，如果类中声明了某些移动操作，隐式拷贝操作可能会被定义为 deleted，因此“有声明”不等于“可正常调用”。

考试中更重要的是：

> **移动相关声明会影响拷贝，拷贝/析构相关声明也会影响移动。**

---

# 5. 最重要：什么时候编译器不会隐式声明移动构造函数

移动构造函数典型形式：

```cpp
A(A&&);
```

在 C++11 及以后，编译器只有在满足一组条件时，才会隐式声明移动构造函数。

可以直接记住：

> 如果下面五类函数中任意一个被用户声明，编译器通常就不会再隐式声明 `A(A&&)`。

### ① 用户声明了拷贝构造函数

```cpp
class A {
public:
    A(const A&);
};
```

此时不会自动再生成：

```cpp
A(A&&);
```

这正是第 1 轮第 7 题的情况。

---

### ② 用户声明了移动构造函数

```cpp
class A {
public:
    A(A&&);
};
```

当然不需要再隐式声明另一个移动构造函数。

---

### ③ 用户声明了拷贝赋值运算符

```cpp
class A {
public:
    A& operator=(const A&);
};
```

这也会抑制隐式移动构造函数的声明。

---

### ④ 用户声明了移动赋值运算符

```cpp
class A {
public:
    A& operator=(A&&);
};
```

同样会抑制隐式移动构造函数的声明。

---

### ⑤ 用户声明了析构函数

```cpp
class A {
public:
    ~A();
};
```

这也是非常高频的考试点。

只要析构函数是用户声明的，即使只是：

```cpp
~A() = default;
```

它仍然是 user-declared destructor，因此会影响移动构造函数的隐式声明。

### 可以压缩成一个记忆表

如果你声明过：

```text
copy ctor
move ctor
copy assignment
move assignment
destructor
```

那么：

```text
不要再默认认为编译器会自动提供 move ctor
```

这也是“Rule of Five”背后的重要背景之一。

---

## 6. `= default` 也会触发“用户已声明”规则

考虑：

```cpp
class A {
public:
    ~A() = default;
};
```

很多人会误以为：

```text
析构函数还是编译器生成的
→ 所以 move constructor 应该照常自动生成
```

这个判断不对。

`~A() = default;` 的确让编译器生成默认析构行为，但它仍然是：

```text
user-declared destructor
```

因此对于移动构造函数的隐式声明规则来说，它仍然算“程序员声明过析构函数”。

这就是为什么考试时必须区分：

```text
user-declared
```

和：

```text
user-provided
```

对于多数选择题，你先牢牢记住：

> `= default` 不等于“没有声明”。

就足够解决大量题目。

---

# 7. 右值传参时，为什么没有 move ctor 仍可能编译成功

回到第 1 轮第 7 题：

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

很多人看到：

```cpp
std::move(a)
```

就认为必然调用：

```cpp
A(A&&)
```

但实际上：

```cpp
std::move(a)
```

本质上只是把表达式转换为一个右值类别，近似理解为：

```cpp
static_cast<A&&>(a)
```

它做的是：

```text
把 a 变成一个适合参与右值重载匹配的表达式
```

而不是：

```text
直接调用移动构造函数
```

本题由于用户已经声明：

```cpp
A(const A&)
```

所以编译器不会自动提供普通的隐式移动构造函数 `A(A&&)`。

但是：

```cpp
const A&
```

可以绑定右值。

所以：

```cpp
A b = std::move(a);
```

仍可以匹配：

```cpp
A(const A&)
```

最终发生的是：

```text
copy
```

而不是 move。

### 一个非常重要的判断链

看到：

```cpp
A b = std::move(a);
```

不要直接写“move”。

应该按下面判断：

```text
std::move(a)
    ↓
产生右值/xvalue
    ↓
候选中有没有可用的 A(A&&)？
    ├─ 有 → 通常优先 move ctor
    └─ 没有 → A(const A&) 是否可用？
                ├─ 可用 → copy ctor
                └─ 不可用 → 编译失败
```

---

# 8. “不会隐式声明”和“隐式声明后被 deleted”不是一回事

这是中高级选择题常见的第二层陷阱。

## 情况 A：根本不会隐式声明 move ctor

例如：

```cpp
class A {
public:
    A(const A&);
};
```

因为已经有用户声明的 copy ctor，所以编译器不会再隐式声明普通的 move ctor。

---

## 情况 B：编译器本来会声明 move ctor，但它最终被定义为 deleted

例如：

```cpp
class M {
public:
    M(M&&) = delete;
};

class A {
public:
    M m;
};
```

`A` 自己没有声明 copy/move/destructor 等，因此编译器会考虑隐式声明：

```cpp
A(A&&)
```

但要生成它时，需要移动成员：

```cpp
m
```

而 `M` 的移动构造被删除：

```cpp
M(M&&) = delete;
```

于是 `A` 的隐式移动构造函数也可能被定义为 deleted。

因此：

```text
隐式声明了
≠
一定能使用
```

---

# 9. 哪些成员会让默认的 move ctor 变成 deleted

考试不一定要求穷举标准措辞，但应掌握基本规律：

如果一个类的某个基类或成员无法按要求完成移动构造，那么该类默认生成的移动构造函数可能被定义为 deleted。

例如成员：

```cpp
class M {
public:
    M(M&&) = delete;
};
```

那么包含它的类：

```cpp
class A {
    M m;
};
```

其默认移动能力会受到限制。

类似问题也可能来自：

- 基类不可移动；
- 成员的移动构造不可访问；
- 成员的移动构造被删除；
- 某些成员类型本身无法按默认规则构造。

考试时看到成员中有：

```cpp
X(X&&) = delete;
```

一定要继续追踪外层类的默认 move 是否还能成立。

---

# 10. 右值引用参数本身还有一个高频陷阱

考虑：

```cpp
void f(A&& x) {
    A y = x;
}
```

虽然 `x` 的**声明类型**是：

```cpp
A&&
```

但表达式：

```cpp
x
```

本身是一个**左值表达式**，因为它有名字。

因此：

```cpp
A y = x;
```

通常会尝试调用拷贝构造函数，而不是移动构造函数。

如果你确实想继续把它当右值转发/移动，需要：

```cpp
A y = std::move(x);
```

所以必须区分：

```text
变量的类型是 A&&
```

和：

```text
表达式 x 的值类别是左值
```

这是中高级选择题非常爱考的一点。

### 记忆句

> **有名字的变量，即使类型是 `T&&`，这个名字作为表达式使用时仍然是左值。**

---

# 11. 一个完整对照例子

```cpp
class A {
public:
    A() { cout << "default\n"; }
    A(const A&) { cout << "copy\n"; }
    A(A&&) { cout << "move\n"; }
};
```

### 左值

```cpp
A a;
A b = a;
```

输出：

```text
copy
```

### 显式转换成右值

```cpp
A c = std::move(a);
```

输出：

```text
move
```

### 右值引用变量有名字

```cpp
A&& r = A{};
A d = r;
```

这里 `r` 是有名字的表达式，因此：

```text
copy
```

而：

```cpp
A e = std::move(r);
```

才会优先：

```text
move
```

---

# 12. `= default` 最常见用途

## 明确要求默认构造函数存在

```cpp
class A {
public:
    A() = default;
    A(int x) : x_(x) {}

private:
    int x_{};
};
```

因为写了 `A(int)`，编译器本来不会再自动补无参 `A()`；此时显式写：

```cpp
A() = default;
```

可以把默认构造函数要回来。

## 明确保留默认拷贝语义

```cpp
A(const A&) = default;
```

## 明确保留默认移动语义

```cpp
A(A&&) = default;
```

## 明确使用默认析构语义

```cpp
virtual ~Base() = default;
```

这在多态基类中非常常见：

```cpp
class Base {
public:
    virtual ~Base() = default;
};
```

意思是：

```text
析构必须是 virtual
但析构函数体不需要自己写资源清理逻辑
让编译器生成默认实现即可
```

---

# 13. 考试速记表

## 默认构造函数

```text
声明了任意构造函数
→ 不要再假设编译器会自动补 A()
```

## 移动构造函数

如果用户声明过以下任意一种：

```text
copy ctor
move ctor
copy assignment
move assignment
destructor
```

则：

```text
不要再假设编译器会隐式声明 A(A&&)
```

## `std::move`

```text
std::move
= 值类别转换
≠ 真正的移动操作
```

## 右值引用变量

```cpp
A&& r = ...;
```

使用名字：

```cpp
r
```

时它仍是左值表达式。

## `= default`

```text
我显式声明这个特殊成员函数
+
要求编译器按默认规则生成实现
```

并且：

```text
= default
仍然属于“声明过”
```

---

# 14. 一句话总框架

碰到 copy/move 选择题时按这个顺序判断：

```text
1. 这个类到底有哪些特殊成员函数？
2. 哪些是用户声明的？
3. 编译器还会不会隐式声明 move/copy？
4. 隐式声明后是否会变成 deleted？
5. 当前实参是左值还是右值？
6. 注意：有名字的 T&& 变量本身是左值表达式。
7. 最后才做重载决议。
```

不要看到 `std::move` 就直接判断“调用移动构造”。
