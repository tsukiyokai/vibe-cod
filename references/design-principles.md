# 设计模式与Clean Code速查

本文件聚焦C++系统编程场景下最实用的子集。编码时对照使用。

---

## 编码前审视: 这个功能点适合什么模式？

### 消除条件分支膨胀

Strategy模式:
- 信号: switch/if-else超过3个分支且可能继续增长
- 做法: 定义接口基类，每个分支实现为独立的Strategy类
- 收益: 新增行为不改已有代码（开闭原则）
- C++实现: 虚函数 + unique_ptr<Base>，或std::function + lambda

State模式:
- 信号: 对象行为随内部状态变化，状态转换逻辑复杂
- 做法: 每个状态一个类，状态转换封装在状态类内部
- 收益: 消除散布在多处的状态判断代码

### 统一流程骨架

Template Method:
- 信号: 多个类有相似的处理流程但个别步骤不同
- 做法: 基类定义算法骨架（public non-virtual），变化步骤为protected virtual
- 收益: 避免流程代码的复制粘贴
- 注意: CANN中常见的Init/Process/Cleanup流程适合此模式

### 解耦创建与使用

Factory Method:
- 信号: 调用方不需要知道具体类型，或创建逻辑可能变化
- 做法: 工厂函数返回unique_ptr<Base>
- 收益: 新增类型不改调用方代码

Builder:
- 信号: 构造参数过多(>4个)或有多种合法组合
- 做法: 链式调用设置各参数，最后Build()
- 收益: 可读性好，避免长参数列表

### 解耦通知与响应

Observer:
- 信号: 一个事件需要通知多个不相关的模块
- 做法: 事件源维护回调列表，事件发生时遍历通知
- C++实现: std::function回调 + vector存储

### 管理资源生命周期

RAII (C++最核心的模式):
- 信号: 任何需要配对的获取/释放操作
- 做法: 构造函数获取，析构函数释放
- 收益: 异常安全，不会忘记释放
- CANN场景: stream/event/buffer/lock的管理

Scope Guard:
- 信号: 需要在作用域退出时执行清理，但不值得单独建类
- C++实现: 利用析构函数或lambda + RAII wrapper

---

## 编码后回看: 代码有哪些smell？

### 函数级

过长(>40行):
- 拆分为子函数，每个函数做且只做一件事
- 拆分标准: 一段代码能用一个动词+名词短语命名 → 提取为函数

过深嵌套(>3层):
- 用early return / guard clause扁平化
- 反转if条件，把错误情况提前return

参数过多(>4个):
- 用结构体/Config对象打包相关参数
- 考虑Builder模式

重复代码:
- 两处以上相似代码 → 提取公共函数
- 相似但有微小差异 → 考虑模板或Strategy

### 命名级

函数名不能自解释:
- 重命名，让代码读起来像英语句子
- 好: `CalculateGcdGroupSize(serverCardCounts)`
- 差: `calc(v)`

魔数:
- 所有字面量用constexpr命名常量
- 好: `constexpr uint32_t kMaxPipelineDepth = 4;`
- 差: `if (depth > 4)`

bool参数:
- 调用处不可读: `CreateComm(true, false)` → 什么意思？
- 改用enum: `CreateComm(Mode::kAsync, Verify::kDisabled)`

### 设计级

上帝类(>500行或>10个public方法):
- 拆分职责到多个类
- CANN中的典型: hccl_communicator_host.cc(8818行)就是教训

Feature Envy(频繁访问另一个类的数据):
- 考虑将方法移动到数据所在的类

数据泥团(同组参数反复出现):
- 提取为结构体/类
- 好: `struct TopoConfig { uint32_t serverCount; uint32_t cardPerServer; ... };`
- 差: `void Setup(uint32_t servers, uint32_t cards, uint32_t level0, uint32_t level1, ...)`

---

## SOLID原则速查

S — 单一职责: 一个类/函数只做一件事。改变它的理由应该只有一个。
O — 开闭: 对扩展开放，对修改关闭。用多态/模板而非修改已有代码来扩展行为。
L — 里氏替换: 子类可以替换父类而不破坏行为。子类不应收窄前置条件或放宽后置条件。
I — 接口隔离: 不强迫依赖不需要的接口。宁可多个小接口也不要一个臃肿的大接口。
D — 依赖反转: 高层模块不依赖低层模块，两者都依赖抽象。

---

## 重构手法速查

Extract Method: 提取代码块为独立函数 — 最常用的重构，提高可读性
Rename: 让名字说出意图 — 最简单但最有效的重构
Replace Conditional with Polymorphism: 用Strategy/State消除switch — 分支>3时考虑
Introduce Parameter Object: 用结构体打包参数 — 参数>4时使用
Replace Magic Number with Named Constant: constexpr — 所有字面量
Split Loop: 一个循环做了两件事 → 拆成两个 — 除非性能关键路径
Extract Class: 类职责过多时拆分 — 类>500行时考虑
Move Method: 方法放错了类 → 移到数据所在的类

---

## 现代C++特性优先级

根据检测到的C++标准，主动使用以下特性:

C++14 (基本都可用):
- auto返回类型推导
- 泛型lambda
- [[deprecated]]属性

C++17 (检测到后使用):
- structured bindings: `auto [key, value] = pair;`
- if constexpr: 编译期分支消除
- std::optional: 替代裸指针表示"可能为空"
- std::string_view: 零拷贝字符串引用
- [[nodiscard]]: 防止忽略返回值
- [[maybe_unused]]: 消除未使用变量警告
- 折叠表达式: 替代递归模板
- class template argument deduction

C++20 (检测到后使用):
- concepts: 约束模板参数
- std::span: 替代(ptr, size)参数对
- designated initializers: `.field = value` 初始化
- [[likely]]/[[unlikely]]: 分支预测提示
- ranges: 管道式算法组合
