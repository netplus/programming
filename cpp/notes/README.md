# C++ 专题学习导航

当前对象生命周期 / 特殊成员函数专题建议按以下顺序学习：

1. `object-lifetime-and-special-members.md`
   - 从对象生命周期、RAII、copy/move/destruction 的设计逻辑建立整体模型。
2. `special-member-suppression-matrix-v2.md`
   - **当前精确版**：C++17 下 default ctor / copy ctor / move ctor / copy assignment / move assignment / destructor 的隐式声明、抑制、deleted、deprecated 关系。
3. `special-member-functions.md`
   - 结合 `std::move`、右值引用、`= default`、`= delete` 与 overload resolution 做具体推导。
4. `object-lifetime-special-members-study-guide.md`
   - 学习导引和统一思维模型。

> 注意：`special-member-suppression-matrix-v2.md` 是特殊成员函数关系的当前事实基线；分析时务必区分“未隐式声明”“声明后 deleted”“仍可隐式生成但存在 deprecated 语义背景”三种情况。
