# 知识路由表

本文件是知识资源的索引，所有资源已收纳在skill的references/目录下。按需查阅，不要一次性加载。

路径说明: 下文所有路径均相对于skill根目录（即SKILL.md所在目录）。

---

## 1. 最终产物 (conventions/)

编码时直接对照使用的规范文档。

cann-hccl-hcomm-conventions.md:
  路径: references/conventions/cann-hccl-hcomm-conventions.md
  内容: 代码惯用法规范，从hccl/hcomm代码阅读中提炼
  来源: 代码阅读分析的最终产物
  何时读: 编码阶段1(学习惯用法) + 阶段4.3(逐条自检)

ops-transformer-mc2-conventions.md:
  路径: references/conventions/ops-transformer-mc2-conventions.md
  内容: MC2算子开发规范，从ops-transformer仓库mc2/模块的系统性代码阅读(Phase 0~9)中提炼
  来源: 代码阅读 MC2分析的最终产物(33个算子、991条commit、29个landmark case深度分析)
  结构: 13节 — 架构概览(S1) / op_host(S2) / op_kernel(S3) / op_graph(S4) / op_api(S5) / 跨层(S6) / 反模式14条(S7) / 演进方向8条(S8) / 测试(S9) / 注册宏速查(S10) / 案例索引(S11) / 场景化指南5个(S12) / 灰色地带判断(S13)
  何时读: 涉及ops-transformer MC2算子开发时，全流程使用(阶段1学惯用法 + 阶段2防缺陷 + 阶段3编码 + 阶段4.3自检)

torch-npu-conventions.md:
  路径: references/conventions/torch-npu-conventions.md
  内容: torch_npu编码规范，从28,222条commit和14万行源码中提炼的39条规则(mandatory~optional分级)
  来源: 代码阅读 torch-npu分析的最终产物(C++: csrc/ 77,751 LOC + Python: 70,866 LOC)
  结构: 9章 — 命名(Ch.1) / 头文件与include(Ch.2) / 错误处理(Ch.3) / 内存管理(Ch.4) / 日志(Ch.5) / 并发(Ch.6) / 测试(Ch.7) / 构建(Ch.8) / 杂项(Ch.9)
  何时读: 涉及torch_npu/PTA开发时，全流程使用(阶段1学惯用法 + 阶段3编码 + 阶段4.3自检)

design-and-coding-guide.md:
  路径: references/conventions/design-and-coding-guide.md
  内容: C++系统编程设计原则（SOLID、设计模式、模块化等）
  来源: design-code-principles分析的最终产物
  何时读: 做架构决策或设计类/接口时

---

## 2. 代码阅读分析 (代码阅读/)

根目录: references/代码阅读/
按仓库维度组织为子目录。

### torch-npu/ — torch_npu (PyTorch Ascend Adapter) 分析
来源: 系统性阅读torch_npu C++核心(csrc/ 367文件 77,751 LOC)和Python层(342文件 70,866 LOC)
最终产物: conventions/torch-npu-conventions.md
根目录: references/torch-npu/

phase0 — 全局架构:
  phase0-architecture-map.md — 目录树、模块LOC统计、csrc子系统职责(core/distributed/framework/aten/profiler/inductor)、Python层(inductor/profiler/npu/utils/distributed/contrib)、初始化序列、全局单例

phase1 — git历史挖掘:
  phase1-git-history.md — 28,222 commits分类统计(NEW_OP/FEATURE/OPTIMIZATION/BUGFIX/REFACTOR等)、年度趋势、Top30热文件、REVERT聚类

phase2 — 设计模式目录:
  phase2-design-patterns.md — 9大模式家族: 注册/Registration(TORCH_LIBRARY/REGISTER_OPTION/GET_FUNC 276处)、RAII/Guard(OptionalNPUGuard/LockGuard)、内存管理(NPUCachingAllocator/StorageDesc)、OpCommand算子框架、Monkey-patch注入、错误处理/异常安全、异步任务队列/Stream管理、分布式通信(ProcessGroupHCCL)、Inductor编译器后端

  何时读: 需要理解torch_npu的既有模式如何做(注册三层机制/OpCommand封装/monkey-patch策略/RAII守卫)时

developers-casebook.md — 16章开发决策案例: 分布式通信/内存管理/算子注册/Monkey-patch/Profiler/初始化生命周期/ACL接口/Inductor/构建系统/序列化/测试基础设施/横切模式/Revert教训/Profiler兼容/ACL版本守卫/Inductor边界适配
  何时读: 遇到torch_npu编码决策时查阅对应章节(Case X.Y.Z格式, 含场景→约束→决策→替代→后果→可迁移经验)

validation-report.md — 用5个真实commit交叉验证conventions+review-standards, 有效覆盖率86.7%(26/30 applicable rules命中), 含规则gap修补记录
  何时读: 验证阶段参考规则实战表现

### hccl-hcomm/ — hccl + hcomm 双仓分析
来源: 系统性阅读hccl/hcomm C++代码
最终产物: conventions/cann-hccl-hcomm-conventions.md
根目录: references/hccl/

phase0 — 全局架构:
  phase0-architecture-map.md — hcomm/hccl目录树、三层架构、模块关系

phase1 — git历史概览:
  phase1-hccl-git-history.md — hccl提交历史分析
  phase1-hcomm-git-history.md — hcomm提交历史分析

phase2 — hcomm深度阅读:
  phase2-hcomm-algorithm.md — 算法层
  phase2-hcomm-framework.md — 框架层
  phase2-hcomm-platform.md — 平台层
  phase2-hcomm-test-examples.md — 测试与示例

phase3 — hccl深度阅读:
  phase3-hccl-op-common.md — 公共基础设施
  phase3-hccl-operators.md — 算子实现
  phase3-hccl-test-examples.md — 测试与示例

phase4 — 双仓对比:
  phase4-contrastive-analysis.md — hccl vs hcomm对比分析

phase5 — 切片深挖:
  phase5-slice-allreduce.md — AllReduce全链路
  phase5-slice-send-recv.md — Send/Recv链路
  phase5-slice-error-propagation.md — 错误传播
  phase5-slice-resource-lifecycle.md — 资源生命周期

phase7 — 验证:
  phase7-validation-report.md — 最终验证报告

phase10 — 深度案例分析(Developer's Casebook全文):
  phase10-feature-arch.md — FEATURE(9)+ARCH_ADAPT(4)类案例6维度深度diff分析
  phase10-bugfix-lifecycle.md — BUGFIX(9)+LIFECYCLE(4)类案例深度分析
  phase10-opt-protocol.md — OPTIMIZATION(3)+PROTOCOL(6)类案例深度分析
  phase10-refactor-revert.md — REFACTOR(3)+REVERT(10)类案例深度分析
  何时读: conventions Ch 11案例摘要不够时，按Case ID(FT-/BF-/OPT-/PL-/AA-/LC-/RF-/RV-)定位对应文件读全文

phase11 — 场景化开发指南(全文):
  phase11-scenario-guide.md — 10个开发场景完整决策清单+案例引用+陷阱(~1395行)
  何时读: 执行conventions Ch 12列出的场景任务时，需完整上下文

phase12 — 灰色地带指南(全文):
  phase12-gray-area-guide.md — 9+1个灰色地带完整正反案例分析(~813行)
  何时读: conventions Ch 13的压缩版判断标准不够时

### ops-transformer-mc2/ — ops-transformer MC2算子分析
来源: 系统性阅读ops-transformer仓库mc2/模块全部33个算子(991条commit, 29个landmark case)
最终产物: conventions/ops-transformer-mc2-conventions.md
根目录: references/ops-transformer/

phase0 — 全局架构:
  phase0-architecture-map.md — MC2四层架构、33个算子目录、三级公共基础设施、构建系统

phase1 — git历史挖掘:
  phase1-git-history.md — 高频修改文件、revert/bugfix/refactor分析、算子时间线、v1→v2→v3演进

phase2 — 公共基础设施:
  phase2-common-infrastructure.md — 仓库级common(TilingBaseClass/Fallback/注册模板) + mc2/common/inc + mc2/3rd
  phase2-common-src-analysis.md — mc2/common/src实现 + new_mc2_mm MatMul库

phase3 — op_host层:
  phase3-op-host.md — 3个代表算子深度阅读 + 跨算子tiling/infershape/def模式统计

phase4 — op_kernel层:
  phase4-op-kernel.md — 3个代表算子深度阅读 + 跨算子arch适配/流水线/同步模式统计

phase5 — op_graph + op_api层:
  phase5-op-graph-api.md — gen_task/fallback模式 + 两阶段API + tests/docs组织

phase6 — 跨算子对比:
  phase6-contrastive-analysis.md — MatMul+Comm vs MoE、v1/v2/v3演进、量化对比、四层一致性

phase8 — 验证:
  phase8-validation-report.md — 用5个真实commit模拟code review验证conventions可操作性

phase9 — 全量commit分析 + 案例教学:
  phase9-commit-classification.md — 991条commit按9类分类索引(BUGFIX/FEATURE/REFACTOR/...)
  developers-casebook.md — 29个landmark case深度diff分析 + 5个场景化开发指南 + 灰色地带判断框架(~2400行)
    何时读: 遇到MC2编码决策灰色地带时查阅具体案例(conventions Section 12引用的NOP-/BUG-/OPT-/ARC-/REF-/REV-编号对应本文)

---

## 4. 设计原则提炼 (design-code-principles/)

根目录: references/design-code-principles/

### 软件设计 (software-design/)
来源: 从Notion软件工程知识库提炼
最终产物: conventions/design-and-coding-guide.md

round1_solid.md — SOLID原则深化
round2_creational_structural.md — 创建型+结构型模式
round3_behavioral.md — 行为型模式
round4_modularity_contracts.md — 模块化与契约设计

### C++语言 (cpp-language/)
来源: 从Notion C++语言知识库提炼
最终产物: 同上(补充到接口与实现章节)

round1_polymorphism.md — 多态与接口实现机制(虚函数、CRTP、type erasure)
round2_value_memory.md — 值语义与内存模型
round3_c_generic_stl.md — C泛型、模板、STL
round4_final_notes.md — 杂项要点
abseil-tips.md — Abseil C++ Tips精选

