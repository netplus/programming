# C++ 对象生命周期与特殊成员函数：学习导引

> 语义基线：C++17。
>
> 本页用于统一现有专题材料的学习顺序，并纠正一个容易误导的表述：C++ 特殊成员函数规则不是资源检测机制，而是基于类声明的对象语义协议推导机制。

## 推荐学习顺序

1. `object-lifetime-and-special-members.md`
   - 建立对象生命周期、RAII、copy/move/destruction 的整体模型。
2. `special-member-suppression-matrix.md`
   - 系统学习 default ctor / copy ctor / move ctor / copy assignment / move assignment / destructor 之间的抑制、deleted 与 defaulted 关系。
3. `special-member-functions.md`
   - 回看 `std::move`、右值引用、`= default`、`= delete` 与 overload resolution 的具体案例。

## 本专题统一采用的设计模型

不要再使用下面这个过度简化的说法：

```text
“程序员只要表现出在管理资源，编译器就会停止生成 move/copy。”
```

它无法解释：

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

默认构造函数中虽然发生了资源获取，但普通/default constructor 本身并不会抑制隐式 move constructor。

统一采用下面这个模型：

> **C++ 不分析函数体来判断你是否在管理资源；它根据类声明中出现了哪些特殊成员函数，以及 base/member 的相应能力，机械地推导这个类型的默认构造、copy、move、assignment 和 destruction 能力。**

这套模型分三层：

```text
第一层：类声明
  ↓
哪些特殊成员函数会被 implicitly-declared？

第二层：base/member 能力
  ↓
defaulted / implicitly-declared 函数是否最终变成 deleted？

第三层：具体调用表达式
  ↓
overload resolution 最终选择谁？
```

## 为什么默认构造函数不属于抑制 move 的“五项”

默认构造函数解决：

```text
一个新对象如何从无到有建立初始状态
```

而下面五项处理的是已有对象的值/状态/生命周期协议：

```text
copy constructor
move constructor
copy assignment
move assignment
destructor
```

因此普通/default constructor 即使在函数体里 `new`、`open()` 或取得句柄，也不会因为这些实现细节而改变隐式 move 的声明规则。

完整抑制关系、C++17 下 copy/move 的非对称规则，以及 `= default` / `= delete` 的区别，请以 `special-member-suppression-matrix.md` 为准。
