# 知识路由表

本文件是知识资源的索引，所有资源已收纳在skill的references/目录下。按需查阅，不要一次性加载。

路径说明: 下文所有路径均相对于skill根目录（即SKILL.md所在目录）。

---

## 1. 最终产物 (conventions/)

编码时直接对照使用的规范文档。

cann-cpp-conventions.md:
  路径: references/conventions/cann-cpp-conventions.md
  内容: 代码惯用法规范，从hccl/hcomm代码阅读中提炼
  来源: dig-repo-code分析的最终产物
  何时读: 编码阶段1(学习惯用法) + 阶段4.3(逐条自检)

design-and-coding-guide.md:
  路径: references/conventions/design-and-coding-guide.md
  内容: C++系统编程设计原则（SOLID、设计模式、模块化等）
  来源: design-code-principles分析的最终产物
  何时读: 做架构决策或设计类/接口时

---

## 2. 缺陷模式知识库 (dig-repo-log/)

来源: 基于git log的缺陷挖掘
基于9237条提交、2279条确认缺陷的系统性分析。

### 跨仓通用模式 (8个)
路径: references/dig-repo-log/cross_repo_synthesis.md
内容: 8个跨仓共性缺陷模式的详细分析（整数溢出、分支不完整、CMake遗漏、复制粘贴、流水线同步、Host-Kernel不一致、空指针、Revert）
何时读: 开始任何编码任务前，通读与任务相关的模式章节
查阅方式: Grep "共性-N" 或关键词跳转到对应章节

### 仓库专项分析

hccl:
  根目录: references/dig-repo-log/hccl/
  pattern_summary.md — 缺陷模式总结
  standards_project_hccl.md — 审查规则集
  defect_analysis.md — 缺陷逐条分析
  hotspot_analysis.md — 热点文件风险分析
  revert_analysis.md — Revert提交分析

hccl-dev:
  根目录: references/dig-repo-log/hccl-dev/
  pattern_summary.md — 5个缺陷类别: 枚举混淆(30%)、接口变更传播(30%)、复制粘贴(20%)、条件不完整(10%)、CMake(10%)
  standards_project_hccl-dev.md — 审查规则集
  defect_analysis.md — 10条缺陷逐条分析（需要看具体案例时读）

hcomm-dev:
  根目录: references/dig-repo-log/hcomm-dev/
  pattern_summary.md — 12个缺陷类别(162条缺陷): 构建(14.8%)、初始化时序(10.5%)、条件分支(9.9%)、数据类型(9.9%)、命名拼写(9.3%)、资源管理(7.4%)、并发安全(6.8%)、日志(6.8%)、状态缓存超时(6.2%)、空指针(4.9%)、API设计(4.9%)、硬件适配(3.7%)
  standards_project_hcomm-dev.md — 审查规则集
  defect_analysis.md — 162条缺陷逐条分析
  hotspot_analysis.md — 18个热点文件风险分析

ops-transformer[-dev]:
  根目录: references/dig-repo-log/ops-transformer/ 和 references/dig-repo-log/ops-transformer-dev/
  pattern_summary.md — 算子开发缺陷模式
  适用场景: 涉及算子tiling、kernel开发时参考

ops-nn[-dev]:
  根目录: references/dig-repo-log/ops-nn/ 和 references/dig-repo-log/ops-nn-dev/
  pattern_summary.md — 算子开发缺陷模式
  适用场景: 涉及NN算子开发时参考

---

## 3. 代码阅读分析 (dig-repo-code/)

来源: 系统性阅读hccl/hcomm C++代码
根目录: references/dig-repo-code/

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

---

## 5. Ascend C文档 (docs/ascendc/)

### API参考 (661页)
索引: references/docs/ascendc/api_reference/INDEX.md
页面目录: references/docs/ascendc/api_reference/pages/
查阅方式: Grep INDEX.md搜API名 → 获取页面文件名 → Read pages/下对应.md

高频API直接路径:
  AllReduce     → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0872.md
  AllGather     → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0873.md
  ReduceScatter → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0874.md
  AlltoAll      → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0875.md
  HCCL InitV2   → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_10221.md
  HCCL Finalize → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0881.md
  DataCopy(概览) → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0101.md
  DataCopy(基础) → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0103.md
  SetFlag/WaitFlag   → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0270.md
  CrossCoreSetFlag   → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0273.md
  CrossCoreWaitFlag  → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0274.md
  PipeBarrier   → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0271.md
  Muls          → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0055.md
  TPipe         → references/docs/ascendc/api_reference/pages/atlasascendc_api_07_0108.md

### 开发指南 (146页 = 基础知识92 + 最佳实践54)
索引: references/docs/ascendc/developer_guide/INDEX.md
页面目录: references/docs/ascendc/developer_guide/pages/
查阅方式: Grep INDEX.md搜关键词 → Read pages/下对应.md

高频主题:
  核函数          → references/docs/ascendc/developer_guide/pages/atlas_ascendc_10_0014.md
  TPipe和TQue编程 → references/docs/ascendc/developer_guide/pages/atlas_ascendc_10_0016.md
  编程模型设计原理  → references/docs/ascendc/developer_guide/pages/atlas_ascendc_10_00015.md
  DoubleBuffer    → references/docs/ascendc/developer_guide/pages/atlas_ascendc_10_00016.md
  Scalar读写数据   → references/docs/ascendc/developer_guide/pages/atlas_ascendc_10_00030.md

---

## 6. HCCL文档 (docs/hccl/)

根目录: references/docs/hccl/

按场景选择:
  通信整体流程、拓扑、算法   → references/docs/hccl/user-guide.md (3911行，用Grep搜索关键词)
  C语言API用法              → references/docs/hccl/api-c.md (1514行)
  Python API               → references/docs/hccl/api-python.md (888行)
  AscendC + HCCL集成开发    → references/docs/hccl/ascendc-hccl.md (1009行，15个核心接口)
  环境变量配置              → references/docs/hccl/env-vars.md (1051行)
  性能测试工具              → references/docs/hccl/hccl-test.md (712行)
  故障排查                  → references/docs/hccl/faq.md
  版本迁移                  → references/docs/hccl/migration.md
