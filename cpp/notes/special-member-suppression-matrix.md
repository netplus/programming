# C++ 特殊成员函数之间的抑制关系：从语言规则到设计逻辑

> 语义基线：C++17。
>
> 本文是 `object-lifetime-and-special-members.md` 的配套章节，专门回答：默认构造、拷贝构造、移动构造、拷贝赋值、移动赋值、析构函数之间到底如何相互影响。

## 1. 先纠正一个容易误导的直觉

不要把规则理解成：

```text
“只要程序员开始管理资源，编译器就停止生成 copy/move。”
```

这不准确。

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

虽然默认构造函数里显然发生了资源获取，但 C++ 不会分析函数体里的 `new`，并据此决定是否生成 move/copy。

真正的规则是：

> **特殊成员函数的自动生成是一套“基于类声明的对象语义协议推导机制”，而不是“资源检测机制”。**

编译器主要观察类声明中出现了哪些特殊成员函数，以及 base/member 是否支持相应操作；它不会尝试猜测函数体中的 `new`、`open()`、`malloc()`、句柄、锁等到底代表什么 ownership。

这有三个直接原因：

1. **资源是程序语义，不是语法概念。** `int fd` 可能是普通整数，也可能是文件描述符；`T*` 可能 owning，也可能 non-owning。
2. **C++ 支持 separate compilation。** 类型的静态语义不能依赖编译器去分析另一个 `.cpp` 中的函数体。
3. **规则必须局部、机械、可预测。** 只看 class declaration，就应该大致能判断它的 copy/move 能力。

---

# 2. 六类函数先分组，而不是先背表

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

它们并不是完全对称的一组六个开关。

可以先分成三类问题：

```text
A. 对象如何开始
   └─ default constructor

B. 一个已有对象的值/状态如何复制或转移到另一个对象
   ├─ copy constructor
   ├─ move constructor
   ├─ copy assignment
   └─ move assignment

C. 对象如何结束生命周期
   └─ destructor
```

其中 move 的隐式生成规则尤其保守：如果程序员已经显式介入了 B/C 组中的任一协议，语言就不再自动假设应当提供 move。

但 **default constructor 属于 A 组**，只说明“新对象如何建立初始状态”，因此它本身不会抑制 move。

---

# 3. 必须区分四个词

## 3.1 implicitly-declared

程序员没写，编译器根据规则声明出来：

```cpp
T(T&&);  // 可能由编译器隐式声明
```

## 3.2 user-declared

只要你在类中写出了声明，就属于 user-declared，包括：

```cpp
~T() = default;
T(T&&) = delete;
T(const T&);
```

`= default` 不等于“没声明”。

## 3.3 defaulted

```cpp
T(T&&) = default;
```

表示：

> 这个函数是我明确声明的，但请编译器按默认成员逐一操作的规则生成实现。

## 3.4 deleted

```cpp
T(T&&) = delete;
```

表示：

> 这个函数存在于重载集合中，但禁止调用。

此外，即使程序员写 `= default`，如果 base/member 的相应操作不成立，这个 defaulted 函数仍可能最终被定义为 deleted。

---

# 4. 总表：C++17 下主要抑制关系

下面的“抑制”主要指：**使目标特殊成员函数不再被编译器隐式声明**。

| 你显式声明了… | 隐式 default ctor | 隐式 copy ctor | 隐式 move ctor | 隐式 copy assignment | 隐式 move assignment | 隐式 destructor |
|---|---|---|---|---|---|---|
| 任意 constructor（如 `T(int)`） | **抑制** | 不因它本身抑制 | 不因普通 ctor 本身抑制 | 不因它本身抑制 | 不因它本身抑制 | 不因它本身抑制 |
| copy ctor `T(const T&)` | 已无关 | 自己已声明 | **抑制** | 通常仍可隐式声明 | **抑制 move assignment** | 通常仍可隐式声明 |
| move ctor `T(T&&)` | 已无关 | **copy ctor 会被定义为 deleted**（若没有显式 copy ctor） | 自己已声明 | **copy assignment 会被定义为 deleted**（若没有显式 copy assignment） | **抑制隐式 move assignment** | 通常仍可隐式声明 |
| copy assignment | 不影响 | 通常仍可隐式声明 | **抑制** | 自己已声明 | **抑制** | 通常仍可隐式声明 |
| move assignment | 不影响 | **copy ctor 会被定义为 deleted**（若没有显式 copy ctor） | **抑制隐式 move ctor** | **copy assignment 会被定义为 deleted**（若没有显式 copy assignment） | 自己已声明 | 通常仍可隐式声明 |
| destructor `~T()` | 不影响 | C++17 下仍可隐式声明，但这种组合在标准演进中被视为不理想/有 deprecated 背景 | **抑制** | C++17 下仍可隐式声明，但同样有 deprecated 背景 | **抑制** | 自己已声明 |

> 这张表不是说“所有特殊成员函数都完全互相关闭”。最重要的是：**move 的隐式生成条件最严格；copy 与 move 的规则并不镜像。**

---

# 5. 默认构造函数的生成规则

默认构造函数：

```cpp
T();
```

如果类中**没有任何 user-declared constructor**，编译器才会考虑隐式声明默认构造函数。

```cpp
class A {
};

A a;  // 可以，编译器提供 A()
```

但：

```cpp
class A {
public:
    A(int);
};

A a;  // 编译失败：不会再自动补 A()
```

如果需要显式恢复：

```cpp
class A {
public:
    A() = default;
    A(int);
};
```

### 为什么“普通 constructor”不抑制 move？

例如：

```cpp
class A {
public:
    A(int x) : x_(x) {}
private:
    int x_;
};
```

`A(int)` 只定义“一个新 A 如何从参数建立”。它没有表达“已有 A 如何复制/转移/销毁”。因此普通 constructor 本身不构成抑制隐式 move 的条件。

---

# 6. 隐式 move constructor 的规则：最应该掌握的一条

编译器只有在以下条件同时满足时，才会隐式声明 move constructor：

```cpp
T(T&&);
```

核心条件可记成：类中没有 user-declared 的

```text
copy constructor
move constructor
copy assignment operator
move assignment operator
destructor
```

注意：

```text
default constructor
普通有参 constructor
```

不在这五项里。

原因不是“它们没管理资源”，而是它们没有显式介入 **已有对象的 copy/move/assignment/destruction 协议**。

---

# 7. 隐式 move assignment 的规则

move assignment：

```cpp
T& operator=(T&&);
```

它的隐式声明条件与 move constructor 高度类似：如果存在 user-declared 的

```text
copy constructor
move constructor
copy assignment
move assignment
destructor
```

通常都不会再隐式声明 move assignment。

因此：

```cpp
class A {
public:
    ~A() = default;
};
```

虽然析构行为仍然可以是编译器默认实现，但因为 destructor 是 **user-declared**，所以隐式 move constructor / move assignment 都被抑制。

---

# 8. copy constructor：不要套用 move 的“五项口诀”

copy constructor：

```cpp
T(const T&);
```

其规则与 move constructor 不对称。

最重要的 C++11+ 规则是：

> 如果类声明了 move constructor 或 move assignment，那么编译器隐式声明的 copy constructor 会被定义为 deleted。

例如：

```cpp
class A {
public:
    A(A&&) = default;
};

A a;
A b = a;  // copy 不可用
```

这里不能说“因为 move 抑制了 copy ctor 的声明”这么简单；更准确地说，相关 copy ctor 语义会成为 deleted。

另一方面，用户声明 destructor 并不会像对 move 那样直接让 copy constructor 消失。C++17 中它仍可能存在，只是这种依赖隐式 copy 的设计历史上被逐渐视为不理想。

---

# 9. copy assignment：与 copy constructor 类似，也会受 move 声明影响

```cpp
T& operator=(const T&);
```

如果声明了 move constructor 或 move assignment，而没有显式提供 copy assignment，则隐式 copy assignment 会被定义为 deleted。

例如：

```cpp
class A {
public:
    A(A&&) = default;
};

A a;
A b;
b = a;  // copy assignment 不可用
```

所以可以形成一个非常有用的方向性理解：

```text
程序员声明 move
    ↓
语言认为“这个类型可能是 move-only”
    ↓
隐式 copy 不再安全地假定可用
    ↓
相关 copy 操作变成 deleted
```

`std::unique_ptr` 就属于典型的 move-only 类型设计。

---

# 10. destructor 为什么抑制 move，但不简单等价于“禁止 copy”

```cpp
class A {
public:
    ~A() = default;
};
```

这里 destructor 是 user-declared，因此：

```text
隐式 move ctor      → 不再自动声明
隐式 move assignment → 不再自动声明
```

但不能直接推出：

```text
copy ctor 一定不存在
copy assignment 一定不存在
```

C++17 下 copy 仍可能由编译器提供。

这正好说明：

> **特殊成员函数不是六个完全对称的开关。**

move 的自动生成规则是后加入语言的，所以为了兼容旧代码，它采用了更保守的“只有类看起来完全没有自定义 copy/move/destruction 协议时才自动提供 move”的策略。

---

# 11. `= default` 不会绕过抑制关系

```cpp
class A {
public:
    ~A() = default;
};
```

`~A() = default` 仍是 user-declared destructor，因此仍抑制隐式 move。

如果你明确知道默认 move 语义就是你想要的，可以显式要求：

```cpp
class A {
public:
    ~A() = default;
    A(A&&) = default;
    A& operator=(A&&) = default;
};
```

这里的意思不是“恢复隐式生成”，而是：

> **程序员明确声明 move，并要求编译器按 defaulted 规则实现。**

这两个概念要分开：

```text
implicitly-declared
```

和：

```text
user-declared + defaulted
```

最终函数体可能类似，但语言分类不同。

---

# 12. `= default` 后仍可能变成 deleted：base/member 能力传播

考虑：

```cpp
class M {
public:
    M(M&&) = delete;
};

class A {
public:
    A(A&&) = default;

private:
    M m;
};
```

你明确写了：

```cpp
A(A&&) = default;
```

不代表一定可移动。

编译器要实现 A 的 defaulted move constructor，就必须移动：

```cpp
m
```

但：

```cpp
M(M&&) = delete;
```

因此 A 的 defaulted move constructor 也会被定义为 deleted。

统一模型是：

```text
A 能 default construct
    ← base/member 能否 default construct

A 能 copy construct
    ← base/member 能否 copy construct

A 能 move construct
    ← base/member 能否 move construct

A 能 copy assign
    ← base/member 能否 copy assign

A 能 move assign
    ← base/member 能否 move assign

A 能 destruct
    ← base/member 的 destruction 是否可用/可访问
```

这比背诵大量孤立规则更重要。

---

# 13. `= delete` 与“不存在”一定要区分

## move 不存在

```cpp
class A {
public:
    A(const A&);
};
```

用户声明 copy ctor 抑制了隐式 move ctor，所以 `A(A&&)` 根本没有作为普通隐式候选出现。

于是：

```cpp
A b = std::move(a);
```

可能退而匹配：

```cpp
A(const A&)
```

因为 `const A&` 可以绑定右值。

## move 存在但 deleted

```cpp
class A {
public:
    A(const A&);
    A(A&&) = delete;
};
```

此时 `A(A&&)` 参与重载决议，并且对右值是更优匹配；选中后发现 deleted，于是编译失败。

编译器不会自动“退回第二名”调用 copy ctor。

---

# 14. Rule of Three / Five / Zero 应该如何从这些规则推出来

## Rule of Three（C++11 以前的核心经验）

如果你需要手写：

```text
destructor
copy constructor
copy assignment
```

中的一个，通常意味着类在直接管理资源，那么另外两个往往也需要认真定义。

## Rule of Five（加入 move 后）

再加入：

```text
move constructor
move assignment
```

形成五项资源/值语义协议。

它不是说“必须永远五个都写”，而是提醒：

> 一旦你开始直接定义 ownership semantics，就应该系统审视五个操作，而不是只修一个。

## Rule of Zero（现代 C++ 更推荐）

尽量不要让业务类直接管理裸资源：

```cpp
class A {
    std::string s;
    std::vector<int> v;
    std::unique_ptr<Resource> r;
};
```

让成员类型自己正确表达：

```text
copy 能不能做
move 能不能做
destruct 怎么做
```

然后外层 A 的特殊成员能力由这些成员递归组合出来。

这就是现代 C++ 对象设计真正希望达到的状态。

---

# 15. 最终不要背“谁关掉谁”，而是使用三层判断

遇到一个类时，按下面顺序判断。

## 第一层：程序员声明了什么？

```text
有没有任意 constructor？
有没有 copy ctor？
有没有 move ctor？
有没有 copy assignment？
有没有 move assignment？
有没有 destructor？
```

这一步决定哪些函数还会被 **implicitly-declared**。

## 第二层：即使声明了/defaulted，它能不能生成？

检查：

```text
base class
member object
```

对应操作是否可用、可访问、未 deleted。

这一步决定函数是否最终成为 **deleted**。

## 第三层：调用表达式最终选谁？

例如：

```cpp
T b = std::move(a);
```

再做正常 overload resolution：

```text
move ctor 是否存在？
是否 deleted？
copy ctor 是否能绑定？
哪个转换序列更优？
```

因此完整模型是：

```text
类声明
  ↓
决定 implicit declaration
  ↓
base/member 能力
  ↓
决定 defaulted/implicit function 是否 deleted
  ↓
具体表达式
  ↓
overload resolution
  ↓
最终调用 / 编译错误
```

掌握这个推导链，比记任何一套选择题答案都更可靠。
