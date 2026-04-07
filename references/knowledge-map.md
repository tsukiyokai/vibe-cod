# 知识路由表

本文件是知识资源的索引，所有资源已收纳在skill的references/目录下。按需查阅，不要一次性加载。

路径说明: 下文所有路径均相对于skill根目录（即SKILL.md所在目录）。

---

## 1. 最终产物 (conventions/)

编码时直接对照使用的规范文档。

cann-hccl-hcomm-conventions.md:
  路径: references/conventions/cann-hccl-hcomm-conventions.md
  内容: 代码惯用法规范，从hccl/hcomm代码阅读中提炼
  来源: dig-repo-code分析的最终产物
  何时读: 编码阶段1(学习惯用法) + 阶段4.3(逐条自检)

ops-transformer-mc2-conventions.md:
  路径: references/conventions/ops-transformer-mc2-conventions.md
  内容: MC2算子开发规范，从ops-transformer仓库mc2/模块的系统性代码阅读(Phase 0~9)中提炼
  来源: dig-repo-code MC2分析的最终产物(33个算子、991条commit、29个landmark case深度分析)
  结构: 13节 — 架构概览(S1) / op_host(S2) / op_kernel(S3) / op_graph(S4) / op_api(S5) / 跨层(S6) / 反模式14条(S7) / 演进方向8条(S8) / 测试(S9) / 注册宏速查(S10) / 案例索引(S11) / 场景化指南5个(S12) / 灰色地带判断(S13)
  何时读: 涉及ops-transformer MC2算子开发时，全流程使用(阶段1学惯用法 + 阶段2防缺陷 + 阶段3编码 + 阶段4.3自检)

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

hccl-hcomm:
  根目录: references/dig-repo-log/hccl-hcomm/
  数据来源: hccl(hcomm仓库, 84条缺陷) + hccl-dev(10条缺陷) + hcomm-dev(162条缺陷), 合计256条缺陷
  standards_project_hccl-hcomm.md — 统一审查规则集(含[来源]标签)
  supplementary_analysis.md — 热点文件风险 + Revert事件 + 跨类别系统性风险
  defect_analysis.md — 256条缺陷逐条分析
  何时读: 涉及通信算法(hccl/hcomm)编码时

ops-transformer:
  根目录: references/dig-repo-log/ops-transformer/
  数据来源: ops-transformer主仓(243条缺陷) + ops-transformer-dev(528条缺陷), 合计771条缺陷
  standards_project_ops-transformer.md — 审查规则集(72条, 含来源标签)
  supplementary_analysis.md — 热点文件风险 + Revert事件 + 跨类别系统性风险
  defect_analysis.md — 771条缺陷逐条分析
  何时读: 涉及算子tiling、kernel开发时

ops-nn:
  根目录: references/dig-repo-log/ops-nn/
  数据来源: ops-nn主仓(380条缺陷) + ops-nn-dev(612条缺陷), 合计992条缺陷
  standards_project_ops-nn.md — 审查规则集(80条, 含来源标签)
  supplementary_analysis.md — 热点文件风险 + Revert事件 + 跨类别系统性风险
  defect_analysis.md — 992条缺陷逐条分析
  何时读: 涉及NN算子开发时

---

## 3. 代码阅读分析 (dig-repo-code/)

根目录: references/dig-repo-code/
按仓库维度组织为子目录。

### hccl-hcomm/ — hccl + hcomm 双仓分析
来源: 系统性阅读hccl/hcomm C++代码
最终产物: conventions/cann-hccl-hcomm-conventions.md
根目录: references/dig-repo-code/hccl-hcomm/

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
根目录: references/dig-repo-code/ops-transformer-mc2/

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
  HCCL完整使用指南(中文)     → references/docs/hccl/HCCL集合通信库使用指南.md (4800+行，综合性中文指南)
  Ascend C算子开发(中文)     → references/docs/hccl/Ascend C算子开发指南.md (18000+行，算子开发全流程)
  通信整体流程、拓扑、算法   → references/docs/hccl/user-guide.md (3911行，用Grep搜索关键词)
  C语言API用法              → references/docs/hccl/api-c.md (1514行)
  Python API               → references/docs/hccl/api-python.md (888行)
  AscendC + HCCL集成开发    → references/docs/hccl/ascendc-hccl.md (1009行，15个核心接口)
  环境变量配置              → references/docs/hccl/env-vars.md (1051行)
  性能测试工具              → references/docs/hccl/hccl-test.md (712行)
  故障排查                  → references/docs/hccl/faq.md
  版本迁移                  → references/docs/hccl/migration.md
  目录总览                  → references/docs/hccl/README.md
