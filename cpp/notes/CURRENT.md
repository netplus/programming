# C++ 对象生命周期 / 特殊成员函数：当前事实基线

> 语义基线：C++17。

后续学习与复盘以以下顺序为准：

1. `object-lifetime-and-special-members.md`
   - 设计哲学、生命周期、RAII、Rule of Three/Five/Zero。
2. `special-member-suppression-matrix-v2.md`
   - **特殊成员函数抑制/隐式声明/deleted 关系的当前精确版。**
3. `special-member-rules-corrections.md`
   - 两个必须掌握的精度点：
     - deprecated copy 的标准条件使用 **user-declared**；
     - defaulted move 若在生成时被 defined as deleted，会被 overload resolution 忽略；这不同于显式 `T(T&&) = delete`。
4. `special-member-functions.md`
   - `std::move`、右值引用、`= default`、`= delete` 的案例推导。

## 统一模型

```text
类声明
  ↓
决定哪些 special member 会被 implicitly-declared
  ↓
base/member 能力
  ↓
决定 implicit/defaulted 函数是否成为 deleted
  ↓
具体表达式
  ↓
overload resolution
  ↓
最终 copy / move / compile error
```

不要再用“编译器检测到程序员在管理资源，因此关闭 copy/move”作为规则解释。C++ 不分析函数体来推断 ownership；ownership 最好通过类型本身表达。
