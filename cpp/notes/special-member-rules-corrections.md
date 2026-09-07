# 特殊成员函数规则：重要精度补充（C++17）

> 配合 `special-member-suppression-matrix-v2.md` 阅读。

## 1. deprecated copy 的标准措辞是 user-declared

C++17 `[depr.impldec]` 的精确规则：

- 如果类有 **user-declared copy assignment** 或 **user-declared destructor**，隐式 copy constructor 的 defaulted 定义属于 deprecated；
- 如果类有 **user-declared copy constructor** 或 **user-declared destructor**，隐式 copy assignment 的 defaulted 定义属于 deprecated。

因此 `= default` / `= delete` 这样的声明仍属于 user-declared，不能简单用“函数体是否由用户手写”判断。

## 2. defaulted move 被定义为 deleted，与显式 `= delete` 还要再区分

```cpp
struct M {
    M(M&&) = delete;
    M(const M&) = default;
};

struct A {
    M m;
};
```

A 的隐式/defaulted move constructor 在生成时可能因为成员 `M` 没有可用的 move 而被定义为 deleted。

但标准对这种 **defaulted move constructor that is defined as deleted** 有特殊规则：它会被 overload resolution 忽略，以免妨碍从右值通过 copy constructor 初始化。

这与程序员直接写：

```cpp
A(A&&) = delete;
```

不同。显式 deleted 的 move candidate 会参与重载决议；如果它是最佳匹配，调用会报错，不会自动退回 copy。

因此分析 deleted move 时至少分：

```text
显式写 T(T&&) = delete
        ↓
候选存在并参与 overload resolution
        ↓
选中后报错

implicit / explicitly-defaulted move
在生成过程中被 defined as deleted
        ↓
该 defaulted deleted move 被 overload resolution 忽略
        ↓
可能由 copy ctor 接管右值
```

这是理解 `= delete` 与“编译器生成失败”区别的重要第二层规则。
