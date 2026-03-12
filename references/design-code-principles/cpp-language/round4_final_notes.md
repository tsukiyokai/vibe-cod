# Round 4: 物理设计、工程实践 + 综合提炼

## 来源

- $MAIN L388-L468 (翻译单元/链接性/extern/static)
- $MAIN L1402-L1643 (主题: 性能/编译/物理设计/ABI/回调/反射/异常)
- round1_polymorphism.md (多态与接口实现机制, 12个知识点)
- round2_value_memory.md (值语义、对象模型与内存管理, 12个知识点)
- round3_c_generic_stl.md (C底层技术、泛型编程与STL, 9大主题30+知识点)
- WebFetch: Abseil Tips #45/#76/#119/#123/#130/#153/#161/#163/#171/#173/#181/#186/#197
- WebFetch: C++兼容C回调(juejin)、C++物理设计(jianshu)

---

## 综合提炼决策

### 新增子章节(8个)

写入位置: design-principles.md "三、编码中遵循: 接口与实现"章节，在"现代C++特性优先级"之后、"四、编码后回看"之前。

1. 多态实现方式选择 — 从round1的12个知识点中提炼决策树+对比表，聚焦何时用哪种
2. 值语义与移动语义 — 从round2的移动语义5大误区、noexcept性能数据、Rule of Five综合
3. 对象模型与内存布局 — 从round2的struct packing、sizeof规则、false sharing
4. 泛型编程 — 从round3的两阶段名字查找、SFINAE、auto推导、ADL
5. C底层技术 — 从round3的do-while(0)、container_of、位操作核心
6. 翻译单元与物理设计 — 从$MAIN本轮新读取的链接性、extern/static + 物理设计 + Abseil namespace tips
7. 错误处理与参数传递 — 从Abseil Tips #76/#181/#171/#123/#173/#163/#234/#45综合
8. STL容器实战 — 从round3的排序谓词、lower/upper_bound、emplace vs push、容器陷阱

### 深化已有内容(2处)

1. "RAII与资源管理": 添加placement new析构责任(3行)
2. "现代C++特性优先级": 添加C++11条目(noexcept/enum class/初始化三板斧/static线程安全)

### 内容取舍决策

详写(高CANN实用性):
- 多态决策树: CANN中virtual/CRTP/Type Erasure都有使用，需要明确选择标准
- 移动语义误区: 代码审查中高频出现的问题
- 翻译单元与物理设计: 大型C++项目的核心工程实践
- C底层技术: hccl/hcomm底层C代码直接使用

简写(一行带过):
- ABI: 仅在对象模型节提及Itanium ABI概念
- 反射: C++26尚未普及，且CANN暂无使用场景
- 异常: 原文仅一条"虚假安全感"，已通过契约设计节覆盖
- 柯里化: 在C++中实用性有限，round3已详细分析
- 华为CSTL优先队列: 过于具体，不适合放入通用参考资料
- 编译优化工具(ccache/distcc/PCH): 在物理设计节一行提及

### 避免重复

以下内容已在design-principles.md中存在，新增内容仅交叉引用:
- Pimpl: 已在"信息隐藏与模块化"节 → 翻译单元节引用
- Type Erasure: 已在DIP节 → 多态选择节引用
- container_of: 已在"OOP in C"节 → C底层技术节引用
- 工厂/观察者/状态机: 已在"编码前审视"节 → 不重复

### 风格一致性

- 所有新增内容遵循: 信号+机制+代码示例+陷阱 格式
- 不使用markdown加粗语法(**)
- 中英文混排紧凑
- 决策表/对比表优先于长段文字
- 代码示例简短，展示核心用法

---

## 产出统计

- 新增内容: ~335行(8个新子章节 + 2处深化 + C++11条目)
- 文件总行数: 1085 → 1420
- WebFetch: 13个Abseil Tips + 2个外部链接

## 未WebFetch的链接(内容已从源材料覆盖)

- C++ ABI (zhuanlan.zhihu.com): 核心概念已在对象模型节从ESR文章和round2覆盖
- C++26反射 (open-std.org): CANN暂无使用场景，一行带过
- ESR Structure Packing: round2已WebFetch，本轮复用结论
