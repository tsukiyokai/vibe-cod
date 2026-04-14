---
name: vibe-cod
description: >
  CANN (hccl/hcomm/ops-transformer) 及 torch_npu (PTA) 编码助手。引导AI在动笔前彻底理解代码和需求，
  学习仓库惯用法，遵循设计文档，利用硬件/API文档，使用现代C++特性，
  防御已知缺陷模式，写出高质量代码。当用户调用 /vibe-cod 时触发。
  输入：任务描述、设计文档路径、待修改文件路径、issue编号等灵活组合。
  适用于hccl、hcomm、ops-transformer(MC2算子)、torch_npu(PTA PyTorch适配层)及CANN生态下的C++/Python开发任务。
---

# CANN / torch_npu 编码助手

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

- 识别涉及的仓库: 一个需求可能跨多个仓库（如hccl + hcomm，或torch_npu + op-plugin），在所有相关仓库中进行后续探索
- 仓库类型判定:
  - hccl/hcomm: 集合通信算法/框架层，纯C++
  - ops-transformer MC2: 融合算子开发，纯C++（Ascend C）
  - torch_npu (PTA): PyTorch昇腾适配层，C++/Python混合（csrc/ 77K LOC C++ + Python 71K LOC）
- 检测C++标准: 对每个仓库Grep CMakeLists.txt，搜索 `CMAKE_CXX_STANDARD`、`cxx_std_XX`、`-std=c++XX`
- 检测Python版本(torch_npu): 检查setup.py/pyproject.toml中的python_requires
- 主动阅读领域文档:
  - hccl/hcomm开发: Read references/hccl-guide.md + references/ascendc-guide.md
  - CANN其他仓(ops-transformer等)开发: Read references/ascendc-guide.md
- 理解任务: 读设计文档/issue描述/用户输入，提炼目标和约束
- 定位目标代码: 在每个涉及的仓库中确定要修改或新增的文件位置

### 0.2 需求驱动的全面扫描

从需求描述中提取所有领域关键词，按概念集群分组探索。

步骤:
1. 拆解关键词 — 例："支持跨框非对称拓扑的Mesh+NHR多维并行算法" → 概念集群: {跨框, 非对称拓扑} + {Mesh, NHR, 多维并行}
2. 建立全局视图 — 按涉及的仓库读取对应的架构分析:

   hccl/hcomm:
   - 读 references/hccl/phase0-architecture-map.md — 三层架构全局视图
   - 跨仓库修改: 加读 references/hccl/phase4-contrastive-analysis.md — 双仓风格差异
   - 特定子系统: 按需读 references/hccl/phase5-slice-*.md (AllReduce/SendRecv/错误传播/资源生命周期)
   - 场景匹配: 任务属于conventions Ch 12的10种场景之一 → 读 references/hccl/phase11-scenario-guide.md 获取完整决策清单

   ops-transformer MC2:
   - 读 references/ops-transformer/phase0-architecture-map.md — 四层架构+算子家族分类
   - 读 references/conventions/ops-transformer-mc2-conventions.md Section 1 — 公共基础设施和arch代际
   - 跨算子对比: 加读 references/ops-transformer/phase6-contrastive-analysis.md — MatMul+Comm vs MoE差异、v1/v2/v3演进
   - 场景匹配: 任务属于conventions Section 12的5种场景(新增算子/arch35适配/同步bug修复/量化支持/tiling优化) → 读 references/ops-transformer/developers-casebook.md 对应场景决策清单

   torch_npu (PTA):
   - 读 references/torch-npu/phase0-architecture-map.md — 仓库全景: 目录树、模块LOC、csrc子系统(core/distributed/framework/aten/profiler/inductor)、Python层、初始化11步序列
   - 设计模式查阅: 读 references/torch-npu/phase2-design-patterns.md 对应章节 — 9大模式家族(注册/RAII/内存/OpCommand/Monkey-patch/异常/Stream/分布式/Inductor)
   - 按目标子系统深入:
     csrc/distributed/ → casebook Ch.1(分布式通信: stream lifecycle/rank mapping/heartbeat/P2P)
     csrc/core/ + framework/ → casebook Ch.2(内存管理) + Ch.6(初始化/生命周期) + Ch.7(ACL接口)
     csrc/aten/ → casebook Ch.3(算子注册) + design-patterns §4(OpCommand框架)
     Python _inductor/ → casebook Ch.8(Inductor后端) + Ch.16(边界适配)
     Python profiler/ → casebook Ch.5(Profiler) + Ch.14(数据管道兼容)
     Python utils/ → casebook Ch.4(Monkey-patch) + design-patterns §5(注入策略)
   - 场景匹配: 读 references/torch-npu/developers-casebook.md 对应章节获取完整决策上下文

3. 对每个概念集群启动一个Explore subagent，在所有涉及的仓库中并行扫描:
   - Grep代码库找所有相关实现 → 逐一Read理解核心逻辑
   - 通过 references/knowledge-map.md 路由到对应文档资源（含"代码阅读分析"和"设计原则提炼"两个section可作深入阅读）
   - 必要时WebSearch补充背景知识（论文、算法原理、业界实现）
4. 跨仓库关联分析: 理解仓库间的接口边界（如hccl对外暴露的API、hcomm作为上层如何调用）
5. 各agent汇总后，建立概念关联图: 这些概念之间如何交互，现有代码如何组织它们

目标: 把每个相关的知识点和现有代码实现都搞清楚，不留盲区。

### 0.3 风险区定位

识别目标文件是否在高风险区域，如果是，提高后续所有阶段的审慎程度，向用户报告风险。

hccl/hcomm: 读 references/hccl/phase1-hccl-git-history.md 或 phase1-hcomm-git-history.md 获取模块级热点(高频修改文件、变更集中度)。

MC2: 读 references/ops-transformer/phase1-git-history.md 获取MC2模块级热点(dispatch_v2是绝对热点、bug密度最高的算子排名、revert历史)。

torch_npu重灾区预警:
  - ProcessGroupHCCL.cpp: 91次bugfix, Bug/100L=1.27, 并发/GIL死锁/stream生命周期
  - NPUCachingAllocator.cpp: 84次bugfix, Bug/100L=2.15, 锁序/资源泄漏/event阻塞
  - npu_sys_ctrl.cpp: 63次bugfix, Bug/100L=15.59, 初始化无回滚/部分清理
  - dynamic_profile.py: 57次bugfix, Bug/100L=28.08(仓库最高), 非原子状态机
  目标文件命中以上任一 → 提高所有阶段审慎程度，向用户报告风险等级

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
- MC2算子: 可在 references/ops-transformer/phase9-commit-classification.md 按类别(BUGFIX/FEATURE/REFACTOR等)检索相关commit，定位历史决策上下文

理解失败协议:
当代码考古穷尽以下手段仍无法确定某段代码的设计理由时（git log/blame无解释性commit message、代码注释和文档无说明、模式无法从上下文推断）:
- 向用户报告具体的理解盲区，请求口头补充
- 如果用户也无法解释: 将该代码段标记为"Chesterton's fence"（不理解围栏的用途就不要拆它），修改时保持其行为不变，仅在其周围或之上构建新逻辑
- 在代码注释中记录: `// NOTE: 此段逻辑的设计理由未能确认，保持原行为不变。参见 [commit hash]`

### 阶段0退出标准

能用自己的话向用户解释:
1. 现有系统如何工作
2. 我的改动如何融入其中
3. 可能受影响的关联模块
4. 调用方依赖的不变量，我的改动是否会破坏它们
5. 这段代码最可能的维护者是谁？他们了解什么、不了解什么？我的代码在他们的视域中是否自解释？

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
  - 测试模式: 测试框架? mock/stub策略? 测试文件组织? (参考 references/hccl/phase2-hcomm-test-examples.md 或 phase3-hccl-test-examples.md)
- 如果同目录不够（如新目录），沿依赖链扩展: 读被include的头文件、读基类所在文件
- 按需扩展: 任务复杂度高/涉及跨模块 → 读相关模块的核心文件
- 跨仓库注意: 不同仓库可能有不同的惯用法，每个仓库分别建立清单

仓库风格速查:
- hcomm: DeviceMem/HostMem异构内存、Pimpl(`_pub.h`)、`owner_`所有权标记、tag`[通信域名]`日志、Manager类后缀、CHK_SMART_PTR_NULL高频、EXCEPTION_HANDLE在C API边界
- hccl: 纯STL内存管理（委托hcomm处理设备内存）、executor/selector/template三件套、`[Module]`前缀日志、无Pimpl、Registry全局注册宏
- ops-transformer(MC2): 四层结构(op_host/op_kernel/op_graph/op_api)100%标准化、6个注册宏全覆盖(OP_ADD/IMPL_OP_INFERSHAPE/IMPL_OP_OPTILING/REGISTER_TILING_DEFAULT/IMPL_OP+CalcOpParam/extern "C" aclnn两阶段)、TilingKey十进制编码驱动编译期多态、AIC(Cube计算)+AIV(通信搬运)核分工、arch32(AICPU)+arch35(CCU)双代并存、SetScheduleMode(1)混合核调度
- torch_npu(PTA): C++/Python混合仓。C++侧: 文件名PascalCase(73.7%)、内部函数camelCase(70%)、缩略词全大写NPU/HCCL/ACL(mandatory)、`#pragma once`(63%)、私有成员trailing underscore(>80%)、smart pointer优先(unique_ptr/shared_ptr)。Python侧: PEP 8(snake_case函数/变量)、monkey-patch通过setattr注入、idempotency guard(`_npu_patched`标志)。注册三层: TORCH_LIBRARY/TORCH_LIBRARY_IMPL + C10_REGISTER_CLASS + 自定义宏(REGISTER_OPTION 48处/GET_FUNC 276处)。RAII守卫: OptionalNPUGuard(设备上下文)、UnlockGuard(临时释放mutex)。OpCommand: 统一算子ACL调用封装(Adapter→Command→Execute)。异常: TORCH_CHECK/TORCH_WARN + NPUException层次。

读取 references/conventions/cann-hccl-hcomm-conventions.md 获取CANN项目级约束。
如果任务涉及ops-transformer MC2算子: 读取 references/conventions/ops-transformer-mc2-conventions.md 获取MC2模块级约束(四层注册模式、tiling策略、kernel实现模式、反模式清单)。
如果任务涉及torch_npu/PTA: 读取 references/conventions/torch-npu-conventions.md 获取PTA编码约束(39条规则, 9章, 覆盖命名/头文件/错误处理/内存/日志/并发/测试/构建)。

### 阶段1退出标准

能列出每个涉及仓库将要遵循的具体惯用法清单。

---

## 阶段2: 知识武装 — API查阅 + 编码陷阱防御

### AI编码陷阱防御
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

torch_npu(PTA)额外陷阱(来源: defect-analysis 1,316条 + casebook 16章):
7. recordStream batch遗漏 — batch_isend_irecv循环中只protect tensors[0]，跳过[1..n]导致多stream UAF [Case 1.1.1, D-1]
8. allocator_mutex → GIL锁序 — 反序持锁(先allocator_mutex再GIL)导致AB-BA死锁；MakeSureQueueEmpty某些路径持GIL+allocator锁 [D-879, Case 2.x]
9. monkey-patch from-import绑定 — `from torch import func`在patch前已绑定，后续setattr不生效；必须用`torch.func`点号访问或在import前完成patch [casebook Ch.4]
10. monkey-patch dispatch变体遗漏 — 只patch `.default`但漏掉`_`(inplace)/`_out`(out变体)，导致部分调用路径走原始实现 [design-patterns §5]
11. GET_FUNC运行时dlsym — GET_FUNC/GET_FUNCTION宏在首次调用时dlsym，若ACL/HCCL库未加载返回nullptr但无检查 → segfault [casebook Ch.7]
12. SoC版本守卫点排除vs范围 — `< Ascend950`会排除未来SoC，应改用`!= Ascend950`点排除 [validation-report, RS P2-4.2]
13. profiler state machine非原子 — dynamic_profile.py的状态转换非原子，并发enable/disable导致状态撕裂 [hotspot #4, Bug/100L=28.08]
14. 初始化无回滚 — npu_sys_ctrl初始化失败时不回滚已完成步骤，重试时状态不一致 [casebook Ch.6]
15. ACL资源描述符泄漏 — aclCreateTensor等创建的descriptor必须显式release，format cast路径上尤其容易遗漏 [D-122]

### API/文档按需查阅

需要查阅Ascend C API、HCCL接口、硬件规格等外部文档时，在仓库docs/目录下搜索相关关键词。

### 阶段2退出标准

已查阅所需API文档，已熟悉当前任务对应的编码陷阱。

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
    op_host(tiling/infershape/def): references/ops-transformer/phase3-op-host.md
    op_kernel(arch适配/流水线/AIC-AIV核分工): references/ops-transformer/phase4-op-kernel.md
    op_graph+op_api(gen_task/fallback/两阶段API): references/ops-transformer/phase5-op-graph-api.md
    公共基础设施(接口/注册模板): references/ops-transformer/phase2-common-infrastructure.md
    公共基础设施(实现/MatMul库/GenTask/性能模型): references/ops-transformer/phase2-common-src-analysis.md
  遇到灰色地带: 查阅conventions Section 13(判断框架) + references/ops-transformer/developers-casebook.md 中的具体案例
- 如果涉及torch_npu/PTA:
  读取 references/conventions/torch-npu-conventions.md 全文对照编码。
  按目标子系统深入读取设计模式:
    分布式(csrc/distributed/): references/torch-npu/phase2-design-patterns.md §8(ProcessGroupHCCL模式) + casebook Ch.1
    内存(csrc/core/): design-patterns §3(NPUCachingAllocator/StorageDesc/Block lifecycle) + casebook Ch.2
    算子(csrc/aten/ + framework/): design-patterns §4(OpCommand/OpAdapter/OpExecute三层) + casebook Ch.3
    Python注入(utils/): design-patterns §5(monkey-patch: 直接替换/wrapper嵌套/idempotency guard) + casebook Ch.4
    Profiler: casebook Ch.5 + Ch.14(数据管道兼容)
    ACL接口(csrc/core/interface/): design-patterns §6(异常安全) + §7(Stream/Event state machine) + casebook Ch.7 + Ch.15(版本守卫)
    Inductor(csrc/inductor/ + _inductor/): design-patterns §9(lowering registry/graph IR/DVM codegen) + casebook Ch.8 + Ch.16(边界适配)
  遇到决策困难: 查阅 references/torch-npu/developers-casebook.md 对应章节(Case X.Y.Z格式，含场景→约束→决策→替代方案→后果→可迁移经验)
  Python编码特别注意:
    - monkey-patch必须在被patch模块被其他代码import前执行
    - 使用点号访问(torch.func)而非from-import绑定
    - patch所有dispatch变体(.default / _inplace / _out)
    - 添加idempotency guard防止重复patch
    - profiler/inductor状态机修改必须考虑线程安全
- 如果涉及hccl/hcomm:
  按目标层深入读取分析文档:
    hcomm算法层: references/hccl/phase2-hcomm-algorithm.md
    hcomm框架层: references/hccl/phase2-hcomm-framework.md
    hcomm平台层: references/hccl/phase2-hcomm-platform.md
    hccl公共基础设施: references/hccl/phase3-hccl-op-common.md
    hccl算子实现: references/hccl/phase3-hccl-operators.md
  遇到灰色地带(V1/V2选择、hccl/hcomm边界、继承深度等): 查阅conventions Ch 13 + references/hccl/phase12-gray-area-guide.md
  需要深入理解某Case: 按ID(FT-/BF-/OPT-等)在 references/hccl/phase10-*.md 中查阅全文
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

第二步半 — 意图传达:
写完功能代码后，审视代码的人类可读性:
- 命名自检: 函数名是否完整表达了行为（含副作用）？变量名是否编码了语义而非仅编码类型？（deviceCount vs n，remoteRankAddr vs addr）
- 决策点注释: 对非显然的设计选择，用一行注释说明"为什么"而非"是什么"。触发条件: 存在两种以上合理写法时
- 结构注释: 对超过50行的函数，在逻辑段落之间用空行+注释标注阶段（如 `// Phase 1: validate inputs`）
- 不注释显然的代码: 如果注释只是用英文/中文重复了代码已经说的事情，删掉它
- 以阶段0确认的维护者画像为基准判断什么算"显然"、什么算"非显然"

第三步 — 对抗性自审:
写完后立即以攻击者视角审视这段代码:
- 如果我要让这段代码崩溃/产生错误结果/死锁/泄漏，我会怎么构造输入？
- 第一步中穷举的每个边界条件，代码是否都正确处理了？
- 错误路径上的资源是否全部释放？是否有early return绕过了清理逻辑？
- 并发场景: 如果两个线程同时执行这段代码，会发生什么？
- 发现任何可攻破的点 → 立即修复，不留到阶段4

### 设计模式与代码优雅

Read references/conventions/design-and-coding-guide.md 全文，对照审视当前代码。
如果指导粒度不足以做出设计决策，按需回溯到 references/design-code-principles/ 下的源材料:
  - 接口/多态选型(虚函数/CRTP/type erasure): cpp-language/round1_polymorphism.md
  - 内存/值语义: cpp-language/round2_value_memory.md
  - 模板/STL用法: cpp-language/round3_c_generic_stl.md
  - 物理布局/编译: cpp-language/round4_final_notes.md
  - C++惯用法速查: cpp-language/abseil-tips.md
  - 设计模式选型(SOLID/创建/结构/行为/模块化): software-design/round1~round4

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

所有功能点已实现，每个都经过了设计→实现→对抗的完整流程，跨仓库接口一致。

---

## 阶段4: 验证 — 全量自检

所有检查项均为必须执行，不可跳过。

### 4.1 冷读审查

清空编码时的思维惯性，以阶段0确认的维护者画像为视角重新审视:
- 生成自己的完整变更diff
- 逐行阅读diff，不依赖编码时的记忆，只看代码本身能否自解释
- 对每个改动问: 这行代码的意图是否清晰？不看上下文能否理解？

语义清晰度检查:
- 命名: 是否存在名字暗示的语义与实际行为不一致的情况？（如getX()有副作用、isValid()做了转换、count实际是index）
- 非显然选择: 每个非显然的设计选择是否有注释说明理由？以维护者画像为基准判断什么算"非显然"
- 模式一致性: 代码的组织结构是否暗示了一种设计模式而实际实现了另一种？
- 作用域误导: 局部修改是否给读者造成"这个方案适用于更广场景"的错误印象？（如仅适用于特定DevType但代码结构未体现此限制）

### 4.2 场景走查

用具体的输入值心算执行完整路径:
- 为每个公开函数构造至少3组输入: 正常路径、边界值、错误路径
- 手动跟踪变量值的变化，验证输出符合预期
- 特别关注: 类型转换点的值域、循环的终止条件、条件分支的覆盖完整性

### 4.3 规则化自检

重新Read references/conventions/cann-hccl-hcomm-conventions.md，逐条约束对照检查。
如果涉及MC2算子: 重新Read references/conventions/ops-transformer-mc2-conventions.md，逐条对照检查。特别关注:
- Section 10 Quick Reference: 6层注册宏是否齐备
- Section 7 Anti-Patterns: 14条反模式逐条核对
- Section 12 场景化指南: 当前任务对应的决策清单是否全部通过
- 验证方法论参考: references/ops-transformer/phase8-validation-report.md (5个真实commit的模拟review经验)
如果涉及hccl/hcomm:
  - 任务属于Ch 12场景: 对照 references/hccl/phase11-scenario-guide.md 对应场景的决策清单逐项检查
  - 验证方法论参考: references/hccl/phase7-validation-report.md (5个真实commit的模拟review经验)
如果涉及torch_npu/PTA: 重新Read references/conventions/torch-npu-conventions.md，逐条对照检查。特别关注:
  - Ch.1命名: PascalCase文件名(csrc/)、camelCase内部函数、缩略词全大写(NPU/HCCL/ACL)
  - Ch.4内存: smart pointer优先、RAII Guard模式、NPUStorageDesc格式桥接
  - Ch.6并发: 锁序(allocator_mutex → GIL → HCCL comm mutex)、check-then-act原子性、shared变量std::atomic或mutex保护
  - 验证方法论参考: references/torch-npu/validation-report.md (5个真实commit的交叉验证, 86.7%命中率)

### 4.4 完整性检查

- 是否遗漏了CMake配置、头文件、前向声明等配套修改
- 新增枚举值/类型: 全局grep枚举类型名，逐一核对所有switch/if-else/map/数组是否同步
- 新增源文件: 确认CMakeLists.txt已注册
- 跨仓库修改: 两侧的接口定义完全一致，读 references/hccl/phase4-contrastive-analysis.md 确认风格一致
- MC2跨算子修改: 读 references/ops-transformer/phase6-contrastive-analysis.md 确认算子间一致性
- torch_npu C++/Python跨层修改: csrc/与Python层修改必须同步(如新增C++ API必须在_C绑定和Python wrapper中注册; 修改OpCommand参数必须同步op-plugin侧)
- torch_npu monkey-patch修改: 确认patch在目标模块被import前执行、使用点号访问、覆盖所有dispatch变体、有idempotency guard
- torch_npu注册修改: 新增算子必须在TORCH_LIBRARY_IMPL + functionalization + DTensor + schema.json + dynamo中全部注册
- torch_npu SoC兼容: 新增SoC版本守卫使用点排除(!= AscendXXX)而非范围排除(< AscendXXX)
- format string: 日志中所有%格式符与参数类型匹配
- const正确性: 所有只读参数标记const
- struct初始化一致性: 新增/修改的struct字段默认值风格统一
- 复制粘贴检查: 如果有从相似代码复制的部分，逐行对比确认所有标识符已更新

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

## 注释边界原则

代码有两个听众：编译器和人。编译器只需形式正确；人需要理解意图。

需要注释（代码无法自表达的信息）:
- 设计决策的理由: 为什么选这个方案而非另一个
- 非显然的约束: 硬件限制、协议要求、向后兼容等代码中看不到的原因
- 临时妥协: TODO/HACK标注及计划的修复方式
- Chesterton's fence: 理解失败协议中标记的保持原行为的代码段

不需要注释（代码已经表达的信息）:
- 代码做了什么（好的命名已经说了）
- 函数参数的类型和名称（签名已经说了）
- 控制流的走向（if/else已经说了）

判断标准: 如果删掉这条注释，一个具有阶段0确认的维护者画像所描述背景的开发者是否仍能理解代码？能 → 删掉。不能 → 保留。

