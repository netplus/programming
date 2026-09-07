# C++17 特殊成员函数抑制关系速查

> 本页只做速查，不替代原理学习。详细解释见 `special-member-suppression-matrix.md`。

| 已声明项 | default ctor | copy ctor | move ctor | copy assign | move assign | destructor |
|---|---|---|---|---|---|---|
| 任意 constructor | 抑制隐式 default ctor | — | 普通 ctor 本身不抑制 | — | — | — |
| copy ctor | — | 自己已声明 | 抑制 | 通常仍可隐式声明 | 抑制 | — |
| move ctor | — | 隐式 copy ctor 会 deleted | 自己已声明 | 隐式 copy assign 会 deleted | 抑制 | — |
| copy assignment | — | 通常仍可隐式声明 | 抑制 | 自己已声明 | 抑制 | — |
| move assignment | — | 隐式 copy ctor 会 deleted | 抑制 | 隐式 copy assign 会 deleted | 自己已声明 | — |
| destructor | — | C++17 下仍可能隐式存在 | 抑制 | C++17 下仍可能隐式存在 | 抑制 | 自己已声明 |

判断时始终分三层：

```text
类声明
→ 是否 implicitly-declared
→ base/member 是否让它 deleted
→ 调用点 overload resolution 最终选谁
```
