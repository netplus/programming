# C++17 特殊成员函数关系：精确版

> 本文替代性补充 `special-member-suppression-matrix.md` 中的速查关系，重点区分：
>
> 1. **不再隐式声明（not implicitly declared）**；
> 2. **仍会隐式声明，但被定义为 deleted**；
> 3. **仍可隐式定义，但这种隐式生成在 C++11 起属于 deprecated 方向**；
> 4. **显式 `= default` 与真正的 implicit declaration 不是一回事**。
>
> 语义基线：C++17。

## 1. 六个特殊成员函数

```cpp
class T {
public:
    T();                         // default ctor
    T(const T&);                 // copy ctor
    T(T&&);                      // move ctor
    T& operator=(const T&);      // copy assignment
    T& operator=(T&&);           // move assignment
    ~T();                        // destructor
};
```

先记住：它们不是六个完全对称的开关。

---

## 2. 第一张表：谁会阻止谁“被隐式声明”

| 你已经 user-declared 了… | 隐式 default ctor | 隐式 copy ctor | 隐式 move ctor | 隐式 copy assignment | 隐式 move assignment | 隐式 destructor |
|---|---|---|---|---|---|---|
| 任意 constructor（包括 default/copy/move/普通有参 ctor） | **不再隐式声明** | 见各自专门规则 | 普通/default ctor 本身不抑制；copy ctor 会抑制；move ctor 自己已存在 | 普通 ctor 本身不抑制 | 普通/default ctor 本身不抑制；copy/move ctor 会抑制 | 不因 ctor 本身抑制 |
| copy ctor | **抑制 default ctor** | 自己已声明 | **抑制** | 不因 copy ctor 本身阻止声明，但有 deprecated 关联见后文 | **抑制** | 不抑制 |
| move ctor | **抑制 default ctor** | 隐式 copy ctor仍会被声明，但随后 **defined as deleted** | 自己已声明 | 隐式 copy assignment仍会被声明，但随后 **defined as deleted** | **抑制隐式 move assignment** | 不抑制 |
| copy assignment | 不抑制 | copy ctor 仍可隐式声明；其隐式定义有 deprecated 关联 | **抑制** | 自己已声明 | **抑制** | 不抑制 |
| move assignment | 不抑制 | 隐式 copy ctor会被声明，但随后 **defined as deleted** | **抑制** | 隐式 copy assignment会被声明，但随后 **defined as deleted** | 自己已声明 | 不抑制 |
| destructor | 不抑制 | copy ctor 仍可隐式声明；其隐式定义有 deprecated 关联 | **抑制** | copy assignment 仍可隐式声明；其隐式定义有 deprecated 关联 | **抑制** | 自己已声明 |

这里最重要的是三种不同结局：

```text
A. 根本不隐式声明
B. 隐式声明了，但 defined as deleted
C. 隐式声明并仍可定义，只是这种依赖方式属于 deprecated 方向
```

考试和工程分析时一定不要混成“都叫抑制”。

---

# 3. default constructor：规则最独立

如果类中没有任何 user-declared constructor，编译器才会隐式声明 default constructor。

```cpp
class A {};
// implicit A()
```

只要声明任意 constructor：

```cpp
class A {
public:
    A(int);
};
```

就不再隐式提供：

```cpp
A();
```

同样：

```cpp
class A {
public:
    A(const A&);
};
```

copy constructor 本身也是 constructor，因此也会使隐式 default constructor 不再出现。

如果两者都想要：

```cpp
class A {
public:
    A() = default;
    A(const A&);
};
```

这里 `A() = default` 是 **user-declared + explicitly-defaulted**，而不是 implicitly-declared。

---

# 4. move constructor：为什么“五项”最值得记

典型签名：

```cpp
T(T&&);
```

C++11+ 中，编译器只在类没有显式介入 copy/move/assignment/destruction 协议时自动提供 move constructor。

实用记忆为：如果你声明了下面任一项，就不要再期待编译器额外自动提供普通的隐式 move ctor：

```text
copy ctor
move ctor（你已经自己声明了）
copy assignment
move assignment
destructor
```

而：

```text
default ctor
普通有参 ctor
```

本身不抑制隐式 move ctor。

## 为什么？

不是因为编译器判断“有没有资源”。

而是：

```text
default/普通 ctor
    → 表达“新对象如何建立”

copy/move/assignment/destructor
    → 表达“已有对象的值/状态如何复制、转移、替换、结束”
```

语言只根据声明层面的协议做机械判断，不分析函数体。

---

# 5. move assignment：和 move ctor 高度相似

```cpp
T& operator=(T&&);
```

如果已经 user-declared：

```text
copy ctor
move ctor
copy assignment
move assignment
destructor
```

则不会再自动隐式声明 move assignment。

因此：

```cpp
class A {
public:
    ~A() = default;
};
```

虽然析构实现是 defaulted 的，但 `~A()` 仍是 user-declared destructor，所以：

```text
implicit move ctor       → 不再产生
implicit move assignment → 不再产生
```

---

# 6. copy constructor：与 move 不对称

```cpp
T(const T&);
```

如果程序员没有提供 copy constructor，编译器通常仍会隐式声明一个 copy constructor。

但是如果类声明了：

```text
move constructor
或 move assignment
```

那么这个隐式 copy constructor 会被 **defined as deleted**。

这是：

```text
“声明了，但 deleted”
```

不是：

```text
“根本没有声明”
```

例如：

```cpp
class A {
public:
    A(A&&) = default;
};

A a;
A b = a;  // copy 不可用
```

---

# 7. copy assignment：同样受 move 声明影响

```cpp
T& operator=(const T&);
```

如果类声明了：

```text
move constructor
或 move assignment
```

那么隐式 copy assignment 会被 **defined as deleted**。

例如：

```cpp
class A {
public:
    A& operator=(A&&) = default;
};

A a, b;
a = b;  // copy assignment 不可用
```

---

# 8. destructor 对 copy 与 move 的影响不一样

```cpp
class A {
public:
    ~A();
};
```

对 move：

```text
implicit move ctor       → 不再隐式声明
implicit move assignment → 不再隐式声明
```

对 copy：

```text
implicit copy ctor       → C++17 下仍可存在
implicit copy assignment → C++17 下仍可存在
```

但标准演进对这种“自己定义 destructor，却继续依赖隐式 copy”的组合持保留态度。

更精确地说：

- 隐式定义 copy constructor，在类有 user-defined destructor 或 user-defined copy assignment 时属于 deprecated 方向；
- 隐式定义 copy assignment，在类有 user-declared destructor 或 user-declared copy constructor 时属于 deprecated 方向。

因此工程设计上不要依赖“我写了析构但 copy 仍然偷偷可用”来表达 ownership semantics。

---

# 9. `= default`：不会把 user-declared 变回 implicit

```cpp
class A {
public:
    ~A() = default;
};
```

含义是：

```text
user-declared destructor
+ explicitly-defaulted implementation
```

不是：

```text
“等价于没写 destructor”
```

所以仍会影响 move 的隐式声明。

如果你需要 move，可以明确写：

```cpp
class A {
public:
    ~A() = default;
    A(A&&) = default;
    A& operator=(A&&) = default;
};
```

这时 move 是：

```text
user-declared + defaulted
```

而不是 implicit move。

---

# 10. `= default` 以后，还要再过 base/member 能力检查

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

`A(A&&) = default` 不保证最终可用。

编译器要生成 A 的 memberwise move，就需要处理 `m`。

如果对应操作对成员/基类不可用、不可访问或 deleted，则 A 的 defaulted move ctor 也可能成为 deleted。

所以：

```text
= default
≠
保证函数可用
```

---

# 11. 为什么“默认构造里 new 了内存”不会改变规则

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

语言不会因为看到了 `new` 就停止生成 copy/move。

原因：

```text
new/open/malloc/handle/lock
```

这些实现细节并不能可靠表达 ownership。

例如：

```cpp
int fd;
T* p;
uint64_t handle;
```

都可能代表资源，也可能只是普通值或 borrowed handle。

因此 C++ 选择：

```text
不分析函数体语义
↓
只根据类型声明与成员类型能力做推导
```

这也是为什么现代 C++ 强调：

```cpp
std::unique_ptr
std::shared_ptr
std::vector
std::string
std::fstream
```

让 ownership 尽量体现在 **类型** 中，而不是隐藏在裸指针和函数体里。

---

# 12. 最终关系图

```text
                            ┌────────────────────┐
                            │ any user-declared  │
                            │ constructor        │
                            └─────────┬──────────┘
                                      │
                                      ▼
                         implicit default ctor
                              不再产生


copy ctor ───────────────┐
copy assignment ─────────┤
move assignment ─────────┤
destructor ──────────────┼──→ implicit move ctor 不再产生
move ctor(自己已声明) ────┘


copy ctor ───────────────┐
move ctor ───────────────┤
copy assignment ─────────┤
destructor ──────────────┼──→ implicit move assignment 不再产生
move assignment(自己已声明)┘


move ctor ───────────────┐
                         ├──→ implicit copy ctor 被 defined as deleted
move assignment ─────────┘


move ctor ───────────────┐
                         ├──→ implicit copy assignment 被 defined as deleted
move assignment ─────────┘
```

然后再加第二层：

```text
base/member 对应操作不可用
        ↓
implicit/defaulted special member
        ↓
可能 defined as deleted
```

最后第三层才是：

```text
具体调用表达式
        ↓
overload resolution
        ↓
最终选 copy / move / deleted candidate / compile error
```

---

# 13. 最重要的学习结论

不要背成：

```text
“某某函数会关掉某某函数。”
```

而要分清三种机制：

```text
1. declaration suppression
   → 编译器根本不隐式声明目标函数

2. implicit/defaulted function becomes deleted
   → 函数存在，但语义无法成立

3. overload resolution
   → 在实际调用点决定最终选择哪个候选
```

掌握这三层以后，特殊成员函数的大多数规则都可以推导，而不是靠背题。
