---
name: vibe-coding
description: >
  CANN (hccl/hcomm/ops-transformer) C++编码助手。引导AI在动笔前彻底理解代码和需求，
  学习仓库惯用法，遵循设计文档，利用硬件/API文档，使用现代C++特性，
  防御已知缺陷模式，写出高质量代码。当用户调用 /vibe-coding 时触发。
  输入：任务描述、设计文档路径、待修改文件路径、issue编号等灵活组合。
  适用于hccl、hcomm、ops-transformer(MC2算子)及CANN生态下的C++开发任务。
---

# CANN C++编码助手

## 输入解析

解析 $ARGUMENTS，识别以下输入类型并提取信息:
- 任务描述（自然语言）
- 设计文档路径（本地.md文件或URL）
- 待修改的源代码文件路径
- issue编号或链接
- 仓库路径（可能涉及多个仓库，如hccl + hcomm）

如果输入不完整，向用户追问缺失信息。至少需要一个任务描述。

## 工作流总览

迭代式工作流，非线性:

```
     ┌──────────────────────────────────┐
     │                                  │
     v                                  │
[0.Explore] → [1.Idioms] → [2.Arm] → [3.Code]
     ^              ^                    │
     │              └────────────────────┘
     │                                   │
     └─────────────── [4.Verify] ────────┘
```

各阶段之间允许回跳，不强制线性推进。

---

## 阶段0: 深度探索 — 动笔前的充分理解

核心原则: 不理解就不动手。在写任何一行代码之前，必须对现有系统和需求建立深刻理解。

### 0.1 环境感知

- 识别涉及的仓库: 一个需求可能跨多个仓库（如hccl + hcomm），在所有相关仓库中进行后续探索
- 检测C++标准: 对每个仓库Grep CMakeLists.txt，搜索 `CMAKE_CXX_STANDARD`、`cxx_std_XX`、`-std=c++XX`
- 理解任务: 读设计文档/issue描述/用户输入，提炼目标和约束
- 定位目标代码: 在每个涉及的仓库中确定要修改或新增的文件位置

### 0.2 需求驱动的全面扫描

从需求描述中提取所有领域关键词，按概念集群分组探索。

步骤:
1. 拆解关键词 — 例："支持跨框非对称拓扑的Mesh+NHR多维并行算法" → 概念集群: {跨框, 非对称拓扑} + {Mesh, NHR, 多维并行}
2. 建立全局视图 — 按涉及的仓库读取对应的架构分析:

   hccl/hcomm:
   - 读 references/dig-repo-code/hccl-hcomm/phase0-architecture-map.md — 三层架构全局视图
   - 跨仓库修改: 加读 references/dig-repo-code/hccl-hcomm/phase4-contrastive-analysis.md — 双仓风格差异
   - 特定子系统: 按需读 references/dig-repo-code/hccl-hcomm/phase5-slice-*.md (AllReduce/SendRecv/错误传播/资源生命周期)
   - 场景匹配: 任务属于conventions Ch 12的10种场景之一 → 读 references/dig-repo-code/hccl-hcomm/phase11-scenario-guide.md 获取完整决策清单

   ops-transformer MC2:
   - 读 references/dig-repo-code/ops-transformer-mc2/phase0-architecture-map.md — 四层架构+算子家族分类
   - 读 references/conventions/ops-transformer-mc2-conventions.md Section 1 — 公共基础设施和arch代际
   - 跨算子对比: 加读 references/dig-repo-code/ops-transformer-mc2/phase6-contrastive-analysis.md — MatMul+Comm vs MoE差异、v1/v2/v3演进
   - 场景匹配: 任务属于conventions Section 12的5种场景(新增算子/arch35适配/同步bug修复/量化支持/tiling优化) → 读 references/dig-repo-code/ops-transformer-mc2/developers-casebook.md 对应场景决策清单

3. 对每个概念集群启动一个Explore subagent，在所有涉及的仓库中并行扫描:
   - Grep代码库找所有相关实现 → 逐一Read理解核心逻辑
   - 通过 references/knowledge-map.md 路由到对应文档资源（含"代码阅读分析"和"设计原则提炼"两个section可作深入阅读）
   - 必要时WebSearch补充背景知识（论文、算法原理、业界实现）
4. 跨仓库关联分析: 理解仓库间的接口边界（如hccl对外暴露的API、hcomm作为上层如何调用）
5. 各agent汇总后，建立概念关联图: 这些概念之间如何交互，现有代码如何组织它们

目标: 把每个相关的知识点和现有代码实现都搞清楚，不留盲区。

### 0.3 风险区定位

读取涉及仓库对应的 references/dig-repo-log/<repo>/hotspot_analysis.md，确认:
- 目标文件是否在高风险文件列表中
- 目标文件所属模块的缺陷密度
- 如果目标文件是缺陷磁铁: 提高后续所有阶段的审慎程度，向用户报告风险

MC2补充: 读 references/dig-repo-code/ops-transformer-mc2/phase1-git-history.md 获取MC2模块级热点(dispatch_v2是绝对热点、bug密度最高的算子排名、revert历史)。

### 0.4 调用链分析

理解目标代码在系统中的位置:
- 向上追溯: 谁调用了目标函数/类？这些调用方依赖什么不变量（前置条件、返回值语义、副作用）？
- 向下追溯: 目标代码依赖的下游接口，它们的契约是什么（参数约束、错误码含义、线程安全性）？
- 修改接口时: 必须找到所有调用点，评估每个调用点的兼容性

### 0.5 深度代码考古

对目标代码进行WHY层面的理解:
- 阅读目标文件及其核心依赖，理解当前设计意图
- `git log --follow <file>` 追溯关键变更的历史脉络
- 搜索注释、TODO、FIXME、commit message中的设计决策说明
- 识别约束来源: 硬件限制？向后兼容？性能要求？并发安全？协议约束？
- 不理解某段代码为什么这样写 → 必须先搞清楚再动手
- MC2算子: 可在 references/dig-repo-code/ops-transformer-mc2/phase9-commit-classification.md 按类别(BUGFIX/FEATURE/REFACTOR等)检索相关commit，定位历史决策上下文

### 阶段0退出标准

能用自己的话向用户解释:
1. 现有系统如何工作
2. 我的改动如何融入其中
3. 可能受影响的关联模块
4. 调用方依赖的不变量，我的改动是否会破坏它们

在继续之前，向用户输出这段解释并确认理解正确。

---

## 阶段1: 学习惯用法 — 怎样写才"地道"

在每个涉及的仓库中:
- 读目标文件的同目录下2-3个代码量最大的文件（大文件通常是核心模块，惯用法最丰富）
- 提取维度:
  - 错误处理: CHK_RET? if-return? exception? 具体宏的用法
  - 内存管理: raw ptr? unique_ptr? shared_ptr? NEW_NOTHROW?
  - 日志: HCCL_DEBUG/INFO/WARNING/ERROR? 日志tag格式?
  - 命名: PascalCase? camelCase? snake_case? 成员变量后缀?
  - include组织: 排序规则? guard风格?
  - 常量定义: constexpr? enum? macro?
- 如果同目录不够（如新目录），沿依赖链扩展: 读被include的头文件、读基类所在文件
- 按需扩展: 任务复杂度高/涉及跨模块 → 读相关模块的核心文件
- 跨仓库注意: 不同仓库可能有不同的惯用法，每个仓库分别建立清单

仓库风格速查:
- hcomm: DeviceMem/HostMem异构内存、Pimpl(`_pub.h`)、`owner_`所有权标记、tag`[通信域名]`日志、Manager类后缀、CHK_SMART_PTR_NULL高频、EXCEPTION_HANDLE在C API边界
- hccl: 纯STL内存管理（委托hcomm处理设备内存）、executor/selector/template三件套、`[Module]`前缀日志、无Pimpl、Registry全局注册宏
- ops-transformer(MC2): 四层结构(op_host/op_kernel/op_graph/op_api)100%标准化、6个注册宏全覆盖(OP_ADD/IMPL_OP_INFERSHAPE/IMPL_OP_OPTILING/REGISTER_TILING_DEFAULT/IMPL_OP+CalcOpParam/extern "C" aclnn两阶段)、TilingKey十进制编码驱动编译期多态、AIC(Cube计算)+AIV(通信搬运)核分工、arch32(AICPU)+arch35(CCU)双代并存、SetScheduleMode(1)混合核调度

读取 references/conventions/cann-hccl-hcomm-conventions.md 获取CANN项目级约束。
如果任务涉及ops-transformer MC2算子: 读取 references/conventions/ops-transformer-mc2-conventions.md 获取MC2模块级约束(四层注册模式、tiling策略、kernel实现模式、反模式清单)。

### 阶段1退出标准

能列出每个涉及仓库将要遵循的具体惯用法清单。

---

## 阶段2: 知识武装 — 缺陷模式防御 + API查阅

### 缺陷模式防御

按层次深度阅读repo-dig资源，不跳过任何一层:

第一层 — 系统性认知:
- Read references/dig-repo-log/cross_repo_synthesis.md 全文，建立跨仓缺陷全景视角

第二层 — 仓库专项模式:
根据涉及的仓库，Read对应目录下的全部分析文件:
  - pattern_summary.md — 缺陷分类与频次分布
  - standards_project_*.md — 规则化审查清单(含规则ID和代码示例)
  - revert_analysis.md — 架构级灾难案例与教训
  - defect_analysis.md — 逐条缺陷的根因、修复模式、涉及文件

仓库路由:
  - 涉及通信算法 → 读hccl、hccl-dev、hcomm-dev
  - 涉及算子开发 → 读ops-transformer[-dev]、ops-nn[-dev]
  - 涉及MC2算子 → 额外读 references/conventions/ops-transformer-mc2-conventions.md Section 7(14条反模式) + Section 11(Error-Fix Pairs)

第三层 — 风险点提炼:
- 综合以上阅读，识别与当前任务相关的所有缺陷风险点
- 为每个风险点制定防御策略（编码时具体怎么做来避免）

第四层 — AI编码陷阱防御:
以下6个高频陷阱是AI编码助手最容易犯的错误，按危险度排序:
1. 新增枚举值后只更新部分switch/if/map — 必须全局搜索枚举类型名，逐一核对所有使用点
2. 复制粘贴相似代码后变量名/参数未完全替换 — 逐行diff对比源代码与目标代码
3. 新增源文件后遗漏CMakeLists.txt注册 — 每新增.cc/.h必须同步更新CMake
4. 多维shape乘法溢出（直译数学公式不考虑中间值范围）— 连乘必须用int64_t
5. host/kernel两侧修改不同步 — workspace/tilingData/tilingKey修改必须成对出现
6. V1/V2双代路径修改不对称 — 检查HCCLV2_FUNC_RUN宏的两条路径是否都更新

MC2算子额外陷阱(来源: conventions Section 7 + casebook):
7. sizeof(DataType枚举) vs GetSizeByDataType — sizeof得到枚举存储大小(4字节)而非数据类型字节数 [7.9]
8. GetAttrPointer<int>应为<int64_t> — 类型与op_def不匹配导致系统性4文件48处错误 [BUG-2]
9. buffer大小用bytes还是elements — Duplicate<int32_t>的count传字节数导致4倍越界 [BUG-4]
10. __NPU_ARCH__==3101 typo — 应为3510，编译静默通过但运行时走错分支 [ARC-4]
11. 跨版本目录include — 不要#include "../xxx_v2/xxx.h"，提取到common/ [7.1]
12. ReduceSum workspace复用其他buffer — 必须独立分配，否则数据损坏(5.5h revert) [OPT-3 vs OPT-2]

### API/文档按需查阅

使用 references/knowledge-map.md 中的路由信息:
- Ascend C API: Grep INDEX.md搜API名 → Read对应页面
- HCCL文档: 按场景选择文件
- 硬件知识: 按平台选择文档

### 阶段2退出标准

已识别所有相关缺陷风险点，每个都有明确的防御策略。

---

## 阶段3: 编码 — 增量、地道、优雅、防御性

### 编码节奏

- 增量编码: 一个功能点一个逻辑单元，完成一个再写下一个
- 跨仓库编码顺序: 先修改底层仓库(如hccl)，再修改上层仓库(如hcomm)，确保接口一致性
- 跨仓库接口检查: 修改跨仓库接口时，确保两侧的数据结构、参数类型、错误码定义完全一致

### 每个功能点的编码流程

对每个功能点，严格按以下顺序执行:

第一步 — 设计:
- 明确这个功能点的输入、输出、前置条件、后置条件
- 穷举边界条件: 空输入、零值、最大值、溢出、并发、重入、部分失败
- 定义错误路径: 每种失败场景的处理方式和资源清理
- 如果存在多种实现方案，列出各方案的优劣，选择失败模式最少的那个
- 先写出函数签名和类接口，确认职责划分合理后再实现

第二步 — 实现:
- 主动使用检测到的C++标准允许的最新特性
- 遵循 references/conventions/cann-hccl-hcomm-conventions.md 中的项目约束
- 如果涉及MC2算子:
  遵循 references/conventions/ops-transformer-mc2-conventions.md 中的MC2模块约束。
  按目标层深入读取分析文档:
    op_host(tiling/infershape/def): references/dig-repo-code/ops-transformer-mc2/phase3-op-host.md
    op_kernel(arch适配/流水线/AIC-AIV核分工): references/dig-repo-code/ops-transformer-mc2/phase4-op-kernel.md
    op_graph+op_api(gen_task/fallback/两阶段API): references/dig-repo-code/ops-transformer-mc2/phase5-op-graph-api.md
    公共基础设施(TilingBaseClass/MatMul库/CcTiling): references/dig-repo-code/ops-transformer-mc2/phase2-common-infrastructure.md
  遇到灰色地带: 查阅conventions Section 13(判断框架) + references/dig-repo-code/ops-transformer-mc2/developers-casebook.md 中的具体案例
- 如果涉及hccl/hcomm:
  按目标层深入读取分析文档:
    hcomm算法层: references/dig-repo-code/hccl-hcomm/phase2-hcomm-algorithm.md
    hcomm框架层: references/dig-repo-code/hccl-hcomm/phase2-hcomm-framework.md
    hcomm平台层: references/dig-repo-code/hccl-hcomm/phase2-hcomm-platform.md
    hccl公共基础设施: references/dig-repo-code/hccl-hcomm/phase3-hccl-op-common.md
    hccl算子实现: references/dig-repo-code/hccl-hcomm/phase3-hccl-operators.md
  遇到灰色地带(V1/V2选择、hccl/hcomm边界、继承深度等): 查阅conventions Ch 13 + references/dig-repo-code/hccl-hcomm/phase12-gray-area-guide.md
  需要深入理解某Case: 按ID(FT-/BF-/OPT-等)在 references/dig-repo-code/hccl-hcomm/phase10-*.md 中查阅全文
- 遵循阶段1建立的惯用法清单
- Read references/conventions/design-and-coding-guide.md 全文，对照审视当前设计决策
- 特殊场景处理（来自验证报告）:
  - 析构/清理路径: 不使用CHK_RET（会中断清理），改用`(void)`忽略或记录后继续
  - extern "C"返回unsigned int的函数: 无法CHK_RET，手动if检查
  - C ABI的POD struct: 可用malloc/free，不强制智能指针
  - struct字段初始化: 要么全部提供默认值，要么全部不提供
  - const正确性: 只读参数必须const修饰
  - format string: %s/%d/%u必须与参数类型匹配
  - 容器遍历中不得通过函数调用间接修改容器（析构中先收集key再逐个删除）
  - 注册到外部系统的数据: 确认生命周期覆盖外部使用期，栈变量不适合

第三步 — 对抗性自审:
写完后立即以攻击者视角审视这段代码:
- 如果我要让这段代码崩溃/产生错误结果/死锁/泄漏，我会怎么构造输入？
- 第一步中穷举的每个边界条件，代码是否都正确处理了？
- 错误路径上的资源是否全部释放？是否有early return绕过了清理逻辑？
- 并发场景: 如果两个线程同时执行这段代码，会发生什么？
- 发现任何可攻破的点 → 立即修复，不留到阶段4

第四步 — 缺陷过筛:
对照阶段2识别的缺陷风险点逐一检查这个功能点。

### 设计模式与代码优雅

Read references/conventions/design-and-coding-guide.md 全文，对照审视当前代码。

重构意识:
- 如果发现现有代码有明显的设计缺陷，且修改范围可控，主动提出重构建议
- 但必须用户确认后再动手，不擅自大规模重构

### 设计文档迭代对齐

发现设计文档与代码实际不一致时:
- 暂停编码，分析不一致的原因
- 设计文档遗漏了约束 → 记录偏差，按实际约束编码
- 自己理解有误 → 回到阶段0重新理解
- 向用户报告所有偏差

### 阶段3退出标准

所有功能点已实现，每个都经过了设计→实现→对抗→过筛的完整流程，跨仓库接口一致。

---

## 阶段4: 验证 — 全量自检

所有检查项均为必须执行，不可跳过。

### 4.1 冷读审查

清空编码时的思维惯性，以reviewer视角重新审视:
- 生成自己的完整变更diff
- 逐行阅读diff，不依赖编码时的记忆，只看代码本身能否自解释
- 对每个改动问: 这行代码的意图是否清晰？不看上下文能否理解？

### 4.2 场景走查

用具体的输入值心算执行完整路径:
- 为每个公开函数构造至少3组输入: 正常路径、边界值、错误路径
- 手动跟踪变量值的变化，验证输出符合预期
- 特别关注: 类型转换点的值域、循环的终止条件、条件分支的覆盖完整性

### 4.3 规则化自检

重新Read涉及仓库的standards_project_*.md，逐条规则对照自己的变更检查。
重新Read references/conventions/cann-hccl-hcomm-conventions.md，逐条约束对照检查。
如果涉及MC2算子: 重新Read references/conventions/ops-transformer-mc2-conventions.md，逐条对照检查。特别关注:
- Section 10 Quick Reference: 6层注册宏是否齐备
- Section 7 Anti-Patterns: 14条反模式逐条核对
- Section 12 场景化指南: 当前任务对应的决策清单是否全部通过
- 验证方法论参考: references/dig-repo-code/ops-transformer-mc2/phase8-validation-report.md (5个真实commit的模拟review经验)
如果涉及hccl/hcomm且任务属于Ch 12场景: 对照 references/dig-repo-code/hccl-hcomm/phase11-scenario-guide.md 对应场景的决策清单逐项检查。

### 4.4 缺陷过筛

对照阶段2识别的所有缺陷风险点做最终检查。

### 4.5 完整性检查

- 是否遗漏了CMake配置、头文件、前向声明等配套修改
- 新增枚举值/类型: 全局grep枚举类型名，逐一核对所有switch/if-else/map/数组是否同步
- 新增源文件: 确认CMakeLists.txt已注册
- 跨仓库修改: 两侧的接口定义完全一致，读 references/dig-repo-code/hccl-hcomm/phase4-contrastive-analysis.md 确认风格一致
- MC2跨算子修改: 读 references/dig-repo-code/ops-transformer-mc2/phase6-contrastive-analysis.md 确认算子间一致性
- format string: 日志中所有%格式符与参数类型匹配
- const正确性: 所有只读参数标记const
- struct初始化一致性: 新增/修改的struct字段默认值风格统一
- 复制粘贴检查: 如果有从相似代码复制的部分，逐行对比确认所有标识符已更新

### 4.6 代码检视

调用 /vibe-review 对自己的变更做完整检视。

### 阶段4退出标准

所有检查通过，代码准备好交付。

---

## 迭代规则

回路触发条件:
- 3→1: 编码中遇到不熟悉的模式/宏/工具函数 → 回去学习惯用法
- 3→0: 发现设计文档与代码实际不一致 → 重新理解上下文
- 4→3: 自检发现问题 → 回去修复
- 4→0: 自检发现架构层面理解偏差 → 重新探索
- 任何阶段: 发现需要额外的文档/API知识 → 回到阶段2补充

不强制线性推进。发现问题立即回跳，不要"先写完再回头改"。

---

## 与其他skill协作

- vibe-design: 可读取其产出的设计文档作为阶段0.1的输入
- vibe-review: 阶段4.6调用做代码检视
- vibe-pr: 编码完成后可衔接提交流程
