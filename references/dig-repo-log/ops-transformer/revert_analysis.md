# ops-transformer Revert专项分析

共发现10条Revert相关提交(含1条空提交)，涉及9个独立的回退事件。

## 原始提交追溯汇总

| Revert hash | 原始提交 | 间隔commit数 | 回退原因类别 |
|-------------|---------|-------------|------------|
| 613ae40b | 8df8fe65 | ~19 | API兼容性/编译依赖耦合 |
| 349083a7 | cc85a74e | ~20 | 测试框架迁移不成熟 |
| a6aa6f17 | 54df45bb | ~4 | 编译依赖不完整 |
| 8ade8c3e | 81ef4922 | ~7 | 并发同步正确性 |
| e5988e2b | 5c46cb51 | ~84 | 状态管理逻辑错误(revert of revert) |
| f8ab5505 | 59c9c74f+ec00dc96+f61275e1 | ~30+ | 示例代码集成问题 |
| c8f181eb | ff21f09b | ~1 | 框架API不可用 |
| 4fd755b6 | b3b1f377 | ~7 | 模板编译超时 |
| d6dd8838 | (空提交) | -- | 操作失误 |

---

## 逐条分析

### 613ae40b revert//新增ROPEV2接口支持辅助矩阵输入

- 原始提交: `8df8fe65` "新增ROPEV2接口支持辅助矩阵输入"
- revert原因: 原始提交是一个大型特性变更(27个文件, +2930/-47行)，引入了aclnnRotaryPositionEmbeddingV2新接口，支持辅助旋转矩阵(rotate)输入。关键问题：原始提交同时修改了interleave_rope的调用方式——将本地声明的`aclnnInnerRotaryPositionEmbedding`替换为通过头文件引入的公共API `aclnnRotaryPositionEmbedding`，并在CMakeLists中新增了对rotary_position_embedding模块的依赖(`set(interleave_rope_depends ...)`)。导致API层面的兼容性或编译/链接依赖问题。由cann-robot自动创建revert MR(!1862)，说明是CI门禁触发的紧急回退。
- 缺陷类型: 接口设计缺陷/编译依赖耦合
- 严重程度: 高
- 涉及文件: `posembedding/rotary_position_embedding/`下27个文件；`posembedding/interleave_rope/op_host/CMakeLists.txt`，`posembedding/interleave_rope/op_host/op_api/aclnn_interleave_rope.cpp`
- 审查规则:
  1. 新增公开API(aclnn接口)应与内部实现解耦，不应在同一MR中同时修改依赖方的调用方式
  2. 超过25个文件/3000行的大型特性应分阶段提交，降低回退代价
  3. 修改现有算子对其他算子的调用路径(Inner -> Public API)属于高风险变更，需全平台集成测试

---

### 349083a7 Revert "修改了all_gather_matmul算子ut的op_host组件的用例输入方式，改成csv表格"

- 原始提交: `cc85a74e` "修改了all_gather_matmul算子ut的op_host组件的用例输入方式，改成csv表格"
- revert原因: 原始提交将all_gather_matmul算子的UT用例从C++硬编码改为CSV表格驱动方式。改造引入了CSV解析框架依赖，新增了.csv数据文件和参数解析结构体(param.h)，同时删除了原有的infershape独立测试文件。CSV框架本身存在稳定性问题，或改造不完整(删除了原测试但CSV未完全覆盖所有场景)，导致UT运行失败或覆盖率下降。
- 缺陷类型: 测试基础设施迁移不成熟/测试覆盖回退
- 严重程度: 中
- 涉及文件: `mc2/all_gather_matmul/tests/ut/op_api/`和`mc2/all_gather_matmul/tests/ut/op_host/`下的测试文件、CSV文件、param.h等
- 审查规则:
  1. 测试框架迁移(硬编码 -> CSV驱动)应确保新框架经过充分验证后再推广
  2. 迁移过程中不应删除原有测试文件，应先并行运行验证新框架完全覆盖后再清理
  3. 涉及删除独立测试文件时，审查者应确认新框架中已包含等价的测试覆盖

---

### a6aa6f17 revert//优化头文件

- 原始提交: `54df45bb` "优化头文件"
- revert原因: 原始提交对FIA/IFA核心kernel代码进行了头文件细粒度化优化，将粗粒度的`kernel_operator.h`拆分为更精确的子头文件(`kernel_vec_intf.h`, `kernel_cube_intf.h`, `adv_api/activation/softmax.h`等)。拆分遗漏了某些隐式依赖(某些类型定义或宏通过kernel_operator.h间接引入，拆分后不再可达)，导致在某些编译配置下编译失败。由cann-robot自动创建revert MR(!1862)，CI编译失败触发。距原始提交仅4个commit。
- 缺陷类型: 编译依赖不完整/头文件管理错误
- 严重程度: 中
- 涉及文件: `attention/fused_infer_attention_score/op_kernel/`和`attention/incre_flash_attention/op_kernel/`和`attention/prompt_flash_attention/op_kernel/`下共8个头文件
- 审查规则:
  1. 头文件细粒度化(将kernel_operator.h拆分为子头文件)必须经过全平台、全编译配置的编译验证
  2. 格式清理(trailing whitespace)不应与功能性修改混在同一个commit中
  3. 同时修改多个核心算子(FIA + IFA + PFA)的头文件属于广影响范围变更，应逐模块提交

---

### 8ade8c3e Revert "dispatch优化syncall"

- 原始提交: `81ef4922` "dispatch优化syncall"
- revert原因: 原始提交对moe_distribute_dispatch_v2的同步机制和token计数逻辑进行了优化，将expertTokenNums的计算从单核(lastCore)改为FIRST_CORE(aivId == 0)执行，并使用ReduceSum+循环方式替代原来的直接累加，同时修改多处同步原语(PipeBarrier改为SyncFunc)。这些变更导致运行时正确性问题(数据竞争或结果错误)。距原始提交仅7个commit。
- 缺陷类型: 并发同步/正确性问题
- 严重程度: 高
- 涉及文件: `mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h`(主逻辑)，`mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_v2_constant.h`(常量定义)
- 详细分析:
  - 新增`SetExpertTokenNums()`方法，仅在FIRST_CORE执行，使用statusBuf_从windowInstatusFp32Tensor_复制状态，对每个localExpert逐个调用ReduceSum计算token数。相比原来在lastCore上通过直接读取sendCounts累加，依赖更多同步点
  - 同步修改：将`PipeBarrier<PIPE_MTE3>()`替换为`SyncFunc<AscendC::HardEvent::S_MTE3>()`和`SyncFunc<AscendC::HardEvent::MTE3_MTE2>()`
  - revert恢复了统一的`UpdateTokenNumsOut()`处理方式，在lastCore直接读取GM上的sendCounts值
- 审查规则:
  1. 修改多核同步机制(SyncAll/PipeBarrier/SyncFunc)必须有严格的并发正确性测试，包括多卡多专家场景
  2. 将计算从一个核移到另一个核时，需证明该核在执行时刻已拥有完整的数据依赖
  3. 同步原语的替换(PipeBarrier -> SyncFunc)改变了全局同步语义，需证明等价性

---

### e5988e2b revert

- 原始提交: `5c46cb51`(对`57884a9b` "support dfx: mc2_win"的不当回退)
- revert原因: commit message只写了"revert"一个词。本质上是"revert of revert"——`5c46cb51`尝试将fullmesh的窗口状态管理逻辑从使用公共基类改回手动实现，但这个改动本身是错误的，所以e5988e2b又把它revert回来。
- 缺陷类型: 状态管理逻辑反复 + commit message缺失
- 严重程度: 高
- 涉及文件: `mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h`
- 详细分析:
  - diff涉及MOE dispatch核心路径：移除基类头文件include，恢复本地UB_ALIGN常量定义(重复定义)
  - 将`SetDataStatus()`从使用`InitWinState()`函数改回手动读写方式
  - 后续提交(64e33c46 "fix win addr and sync bug", 662f162c "fix a synchronization issue")证明MOE dispatch v2 fullmesh的状态管理是持续痛点
- 审查规则:
  1. commit message必须说明回退什么、为什么回退
  2. 对"revert of revert"场景，说明原始变更反复的原因比代码本身更重要
  3. 窗口状态管理(SetDataStatus)涉及设备侧GM状态读写，属于高风险代码，修改前应有状态机文档

---

### f8ab5505 revert add_example

- 原始提交: 多个提交的累积效果：`59c9c74f`("add_example mod"), `ec00dc96`("add_example 编译修改"), `f61275e1`("add_example sup 910_93")
- revert原因: 新版add_example引入的图模式调用(GE IR)在开源仓场景下不适用(需GE图引擎依赖)，CMakeLists的BUILD_OPEN_PROJECT分支处理与构建系统冲突，且多轮修改导致路径管理和命名空间混乱。
- 缺陷类型: 示例代码过度扩展/编译集成问题/多轮修改积累的技术债
- 严重程度: 低
- 涉及文件: `examples/add_example/`下17个文件(+28/-658行)
- 审查规则:
  1. 示例代码中引入的能力(如图模式GE IR调用)应先确认是否适用于目标仓库(开源仓 vs 内部仓)
  2. 多轮修改(mod -> 编译修改 -> sup 910_93)说明原始方案不成熟，应在设计阶段明确需求再一次性实现
  3. 命名空间(Math vs Transformer)和include路径(相对 vs 绝对)的反复切换说明缺少统一编码规范

---

### c8f181eb revert//schedule

- 原始提交: `ff21f09b` "set schedule mode"
- revert原因: 原始提交为FIA非量化MLA分支添加了schedule mode设置功能，依赖框架的`context_->SetScheduleMode()`接口。该接口在当时的框架版本中不可用或未稳定。revert仅间隔1个commit，由turing_project1自行创建并合入，说明问题明确且紧急。MR名称中的"-auto"表明是自动触发。
- 缺陷类型: 框架API依赖/接口不成熟
- 严重程度: 中
- 涉及文件: `attention/common/op_host/arch32/fia_tiling_nonquant_mla.cpp`，`attention/common/op_host/arch32/fia_tiling_nonquant_mla.h`，`attention/common/op_host/fia_tiling_base.h`
- 详细分析:
  - 新增`ScheduleMode`枚举(NORMAL_MODE=0, BATCH_MODE=1, SYNC_MODE=2)和`SetScheduleMode()`方法
  - 硬编码`scheduleMode_ = ScheduleMode::BATCH_MODE`
  - 后续在多个提交中(0e1f7c2d, 9ac275b5, bc5e43c1)重新引入了此功能，说明功能本身是必要的，只是首次提交时机不对
- 审查规则:
  1. 依赖框架新API前应确认API已在目标框架版本中稳定发布
  2. 仅间隔1个commit即被revert，说明提交前没有经过集成编译验证

---

### 4fd755b6 FIA IFA PFA tilingkey revert

- 原始提交: `b3b1f377` "FIA PFA IFA模板化tiling key"
- revert原因: MR描述明确："FIA IFA PFA算子tilingkey整改回退, 编译超时导致线上ci阻塞"，关联Issue #48。模板化tilingkey导致C++编译器需实例化海量模板组合，编译时间超过CI限制。
- 缺陷类型: 编译性能问题——模板膨胀(template bloat)导致编译超时
- 严重程度: 高
- 涉及文件: 32个文件, +7717/-12346行，涉及`attention/fused_infer_attention_score/`、`attention/incre_flash_attention/`、`attention/prompt_flash_attention/`下的op_host和op_kernel
- 详细分析:
  - 核心目标：将FIA、IFA、PFA三大attention算子的tilingkey路由从硬编码if/else改为C++模板化
  - 问题：每个attention算子有数十个tilingkey维度(head_dim, seq_len, batch_size, quant_mode, mask_type, softmax_mode等)，模板参数的笛卡尔积导致实例化数量呈指数级增长
  - 后续通过`535a2749`("FIA IFA PFA tilingkey模版化整改")和`959643c6`("fia ifa pfa tilingkey整改 ut 适配")重新完成了模板化
- 审查规则:
  1. 大规模模板化重构必须在提交前测量编译时间，并设置编译时间预算作为CI门禁
  2. 超过10000行的diff必须分阶段提交(先FIA单独验证，通过后再IFA，最后PFA)
  3. 模板化tilingkey时应考虑使用显式实例化控制实例化数量，或使用tag dispatch等编译时间更友好的技术

---

### d6dd8838 tilingkey revert

- 原始提交: (空提交，tree hash与parent完全相同)
- revert原因: 这是`4fd755b6`的前置revert尝试。b3b1f377合入后CI编译超时，团队紧急组织回退。d6dd8838(MR!191)是第一次revert尝试，但merge后产生了空结果(没有任何文件变更)，随后4fd755b6(MR!195)才完成了真正的revert。
- 缺陷类型: 操作失误/空提交
- 严重程度: 低
- 涉及文件: 无(空提交，0个文件变更)
- 审查规则:
  1. revert操作前应先`git diff`确认回退内容非空
  2. 紧急回退应有标准操作手册(SOP)，避免在匆忙中产生无效操作
  3. "merge master into master"这种source/target相同的MR不应被允许合入

---

## 跨Revert模式总结

### 模式1：编译问题是首要revert原因(3/9)

- 头文件依赖不完整(a6aa6f17)
- 模板编译超时(4fd755b6)
- 框架API不可用(c8f181eb)

暗示代码审查中"全配置编译验证"环节薄弱。

### 模式2：并发/同步修改是高风险区域(2/9)

- dispatch的syncall优化(8ade8c3e)
- fullmesh状态管理(e5988e2b)

MOE dispatch v2在后续又有多个同步修复提交，说明是持续技术痛点。

### 模式3：大型变更缺乏分阶段策略

- ROPEV2: 27文件/3000行，一次性提交
- tilingkey模板化: 32文件/20000行，一次性提交

一次性提交导致回退代价极高，应拆分为多个独立可验证的小PR。

### 模式4：commit message质量差

- e5988e2b只写了"revert"
- d6dd8838是空提交
- 多个revert的MR描述使用默认模板未填写

给后续维护者带来不必要的考古成本。

### 模式5：CI门禁有效但不够前置

cann-robot自动创建revert说明有CI门禁机制，但"合入后发现问题再回退"的成本远高于"PR阶段就拦截"。应加强pre-merge编译检查和编译时间监控。

### 模式6：功能被revert后最终重新引入

- tilingkey模板化：b3b1f377被revert后通过535a2749重新引入(改进方案)
- schedule mode：ff21f09b被revert后通过0e1f7c2d/9ac275b5/bc5e43c1重新引入

说明功能本身是必要的，问题出在实现方式或提交时机。审查时应关注"这个变更是否经过充分验证"而非"这个功能是否需要"。
