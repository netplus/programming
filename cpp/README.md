# C++ 学习与能力训练

本目录用于系统学习 C++，默认以 **C++17** 为语义基线。学习目标不是记忆选择题答案，而是理解语言设计、对象模型、规则之间的因果关系，并能够从规则自行推导程序行为。

## 学习入口

### 1. 教材

- [`textbook/object-lifetime-and-special-member-functions.md`](./textbook/object-lifetime-and-special-member-functions.md)
  - 对象模型、生命周期与特殊成员函数专题。
  - 从 storage / lifetime / object representation / subobject 开始，系统覆盖 constructor/destructor、copy/move、`= default`、`= delete`、特殊成员函数隐式生成与抑制关系、RAII、Rule of Three/Five/Zero、C++17 copy elision 等。

- [`textbook/polymorphism-overload-and-dispatch.md`](./textbook/polymorphism-overload-and-dispatch.md)
  - 多态、重载、名字查找与虚派发专题。
  - 从 Base subobject、most-derived object、static type / dynamic type 开始，系统覆盖 name lookup、overload resolution、name hiding、`using Base::f`、override、virtual dispatch、default arguments、构造/析构期虚调用、object slicing、virtual destructor、RTTI、多继承二义性等。

后续新增专题统一放入 `textbook/`，避免同时维护多个内容重叠的 notes 版本。

### 2. 学习方法

- [`exam-workflow.md`](./exam-workflow.md)
  - 规定当前训练方式：**先建立专题模型，再用少量题目验收**。
  - 题目用于检测是否真正理解规则，而不是作为学习主线。

### 3. 学习复盘

- [`reviews/`](./reviews/)
  - 保存实际训练轮次的逐题复盘、错误原因和薄弱点。
  - 这些文件记录学习轨迹，不作为语言规则的最终事实基线。

### 4. 模拟验收

- [`mock-exam-01.md`](./mock-exam-01.md)
  - 第一套 40 题中高级 C++ 模拟选择题。
  - 建议在完成对应教材专题后用于阶段性验收。

## 建议学习顺序

```text
1. object-lifetime-and-special-member-functions.md
        ↓
   先建立 object / subobject / lifetime / value / identity 模型
        ↓
2. polymorphism-overload-and-dispatch.md
        ↓
   在对象模型上建立 static type / dynamic type / call dispatch 模型
        ↓
3. reviews + mock-exam
        ↓
   用少量题目验证模型是否稳定
```

## 目录职责

```text
cpp/
├── README.md          # 总导航
├── exam-workflow.md   # 学习与验收方法
├── textbook/          # 正式教材 / 事实基线
├── reviews/           # 实际训练复盘
└── mock-exam-*.md     # 阶段验收题
```

统一原则：

> **规则以 `textbook/` 中当前教材为准；`reviews/` 和模拟题服务于理解与验收，不反向成为规则事实基线。**