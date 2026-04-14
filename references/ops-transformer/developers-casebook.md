# Phase 9: Developer's Casebook

## 9.1: Commit Classification Summary

Total unique commits (message-deduped): 991 (Main 363 + Dev 680, 69 shared messages)

| Category | Count | % | Main | Dev |
|----------|-------|---|------|-----|
| NEW_OP | 15 | 1.5% | 9 | 6 |
| FEATURE | 198 | 20.0% | 83 | 115 |
| OPTIMIZATION | 20 | 2.0% | 7 | 13 |
| BUGFIX | 243 | 24.5% | 83 | 160 |
| REFACTOR | 188 | 19.0% | 66 | 122 |
| ARCH_ADAPT | 30 | 3.0% | 18 | 12 |
| INFRA | 68 | 6.9% | 28 | 40 |
| REVERT | 19 | 1.9% | 3 | 16 |
| DOC_TEST | 210 | 21.2% | 84 | 126 |

Phase 1 vs Phase 9 deltas:
- BUGFIX: 243 vs 74 (3.3x, Chinese keywords + dev repo)
- REFACTOR: 188 vs ~30 (6x, '整改/清理/重命名' patterns)
- OPTIMIZATION: 20 vs ~4 (5x, still only 2%, confirming perf changes are careful)
- REVERT: 19 vs 3 (dev has 16 reverts vs main's 3 -- dev is the "first draft")
- NEW_OP: 15 (1.5%) -- rare milestone events
- DOC_TEST: 210 (21.2%) -- 1 doc commit per 4 code commits

Dev repo pattern: proportionally more BUGFIX/REFACTOR, serving as the experimental branch where bugs are caught early.

## 9.2: Landmark Cases (29 total)

### 9.2.1: NEW_OP Cases (4)

#### NOP-1: Complete 4-Layer Operator Addition

Commit: MAIN:d79d1792
Message: add new ops: alltoallMM; update MMalltoall to support Ascend950
Scale: 107 files, +18579/-114 lines
Layers: op_api + op_graph + op_host + op_kernel (complete)
Operators: allto_all_matmul (AlltoAllMatmul + AlltoAllQuantMatmul)

Why selected:
- Most complete example of ground-up operator addition with full 4-layer structure
- Shows both non-quant and quant variants in a single commit
- Includes dual arch implementations (arch32 + arch35)
- Demonstrates the standard file layout and registration patterns

#### NOP-2: Complex Fusion Operator

Commit: DEV:40ce84bb
Message: Add AttentionToFFN
Scale: 33 files, +5205/-8 lines
Layers: op_api + op_graph + op_host + op_kernel (complete)

Why selected:
- Single complex fusion operator (not comm-related but shows the framework)
- Exceptionally detailed documentation (731-line API doc)
- 650-line tiling shows complex parameter derivation
- Complete test suite demonstrates expected UT coverage

#### NOP-3: Multi-Operator Framework Addition

Commit: DEV:fd55c092
Message: 添加Dispatch Setup&Teardown及Combine Teardown算子框架
Scale: 75 files, +2983 lines
Layers: op_api + op_graph + op_host + op_kernel (3 operators in parallel)
Operators: moe_distribute_dispatch_setup, moe_distribute_dispatch_teardown, moe_distribute_combine_teardown

Why selected:
- Batch addition of 3 related operators -- shows framework scaffolding pattern
- Each operator independently complete but sharing moe_distribute_base.h
- Demonstrates .gitkeep + CMakeLists.txt + binary.json skeletal structure
- Lifecycle decoupling pattern (setup/teardown separation from main compute)

#### NOP-4: Existing Operator Arch Extension

Commit: MAIN:a9209ec8
Message: 新增MatmulAlltoAllA3
Scale: 14 files, +1490/-44 lines
Layers: op_api + op_host + op_kernel

Why selected:
- Shows the minimal change set for adding arch support to an existing operator
- Tiling and kernel are the main work (CMakeLists, tiling_910_93.cpp, apt.cpp)
- op_api requires minimal modification (just a branch)
- Demonstrates arch-specific tiling implementation pattern

---

### 9.2.1: FEATURE Cases (5)

#### FEA-1: Communication Pattern Extension

Commit: MAIN:48da3541
Message: MatMulReduceScatter 支持all2all+ vec reducesum
Scale: 13 files, +1870 lines
Layers: op_host + op_kernel

Why selected:
- Adds new communication topology (all2all + vector reduce) to existing operator
- Shows double-buffer GM/UB/GM transfer template pattern
- Demonstrates how to extend tiling for new comm patterns

#### FEA-2: Full-Stack Quantization Feature

Commit: MAIN:648691a4
Message: ataKcQuantMatmul:alltoallMatmul动态量化（包含host与kernel提交）
Scale: 25 files, +2439 lines
Layers: op_host + op_kernel (full stack)

Why selected:
- Complete quantization feature across host+kernel
- 4-stage pipeline design (comm/transpose/quant/compute)
- Type matrix expansion pattern for dtype combinations
- Shows decision: embedded quant via tiling key vs separate operator

#### FEA-3: Tiling Key Template Evolution

Commit: MAIN:1d91a6c5
Message: alltoallvgmm/gmmalltoallv tilingkey template
Scale: 9 files, +402 lines
Layers: op_host + op_kernel

Why selected:
- Shows kernel parameterization evolution: hardcoded -> template
- Clear before/after: if-else branches eliminated via compile-time specialization
- Demonstrates the APT tiling key design philosophy

#### FEA-4: New Topology Template

Commit: MAIN:6c7079c1
Message: Add FullMesh Template for DispatchV2
Scale: 5 files, +1387 lines
Layers: op_kernel (primarily)

Why selected:
- Topology-driven memory layout design (fullmesh vs ring)
- 1086-line kernel implementation shows complete comm template
- Demonstrates how topology choice cascades into tiling formulas
- This commit was later reverted then re-applied (high-risk area)

#### FEA-5: Multi-Operator Feature Development

Commit: DEV:681d2b1a
Message: alltoallmatmul/matmulalltoall kernel实现
Scale: 37 files, +1461 lines
Layers: op_kernel

Why selected:
- Stage-based composition pattern for kernel development
- Covers symmetric operator pair (alltoall->matmul and matmul->alltoall)
- Shows how multiple operators share kernel infrastructure

---

### 9.2.1: OPTIMIZATION Cases (3)

#### OPT-1: Workspace Calculation Optimization

Commit: MAIN:2aad8b3e
Message: AttentionToFFN performance optimization
Scale: 2 files, +76/-66 lines
Layers: op_host (tiling)

Why selected:
- Dynamic workspace alignment calculation (vs static constants)
- Shows AIV count * alignment-aware sizing
- Demonstrates tiling-level performance tuning methodology

#### OPT-2: Fine-Grained Synchronization (Success)

Commit: MAIN:ea8f90e1
Message: 优化全核同步
Scale: 2 files, +85/-65 lines
Layers: op_kernel

Why selected:
- Replaces SyncAll() with selective HardEvent sync (S_V, S_MTE3, MTE3_MTE2)
- Shows the correct way to optimize synchronization
- Contrast with OPT-3 (failed attempt at same optimization)

#### OPT-3: Synchronization Optimization (Failed - Reverted)

Commit: MAIN:81ef4922 -> Reverted by MAIN:8ade8c3e (within 4 hours)
Message: dispatch优化syncall
Scale: 2 files, +71/-54 lines
Layers: op_kernel

Why selected:
- Anti-pattern: aggressive sync optimization broke correctness
- Same developer, same day as OPT-2 -- shows the thin line between success and failure
- Teaches: sync optimization requires exhaustive verification of all execution paths

---

### 9.2.1: BUGFIX Cases (5)

#### BUG-1: Communication Sync Constant Error

Commit: MAIN:f8eb3c97
Message: fix bug:big shape sync fail.
Scale: 1 file
Root cause: MAX_HCCL_HANDLE constant was 16 instead of 63

Why selected:
- Single constant error causing sync failures for large shapes
- HCCL supports 63 parallel handles, code hardcoded 16
- Teaches: always derive constants from spec, never guess

#### BUG-2: GetAttrPointer Type Mismatch

Commit: MAIN:5a217f2d
Message: fix type bug when using GetAttrPointer
Scale: 2 files (moe_distribute_combine, moe_distribute_combine_v2)
Root cause: GetAttrPointer<int> should be GetAttrPointer<int64_t>

Why selected:
- Systematic type bug affecting 10 attributes across MOE operators
- int64 pointer reinterpreted as int pointer -- platform-dependent failure
- Phase 8 identified this as a common anti-pattern (sizeof/type errors)

#### BUG-3: Multi-Layer Systemic Bug (rank < 8)

Commit: DEV:70a262ad
Message: Fix Dispatch and Combine, rank < 8
Scale: 5 files (infershape + tiling + kernel)
Root cause: Three concurrent issues: parenthesis order, BlockDim condition, alignment+Mask limit

Why selected:
- Most complex systemic bug -- single logic error manifesting in 3 layers
- InferShape bracket ordering: `a * b / c` vs `a * (b / c)` changed semantics
- Kernel alignment: RoundUp + DataCopyPad needed for edge case
- Teaches: multi-layer debugging methodology

#### BUG-4: API Size Unit Error (OOM)

Commit: DEV:798597da
Message: fix dispatch oom bug
Scale: 1 file
Root cause: Duplicate<int32_t>() size parameter in bytes vs elements

Why selected:
- Classic sizeof confusion: passed byte count where element count was expected
- Missing `/ sizeof(int32_t)` caused 4x memory over-allocation
- Teaches: always verify API parameter semantics (bytes vs elements vs count)

#### BUG-5: Tiling Constructor Missing Initialization

Commit: DEV:1fab7bab
Message: MatmulAllReduce伪量化Tiling GetPlatformInfo错误修复
Scale: 1 file
Root cause: Constructor missing InitCompileInfo() call

Why selected:
- Missing initialization in tiling class constructor
- GetPlatformInfo() returned garbage without InitCompileInfo()
- Teaches: tiling classes have mandatory initialization sequence

---

### 9.2.1: ARCH_ADAPT Cases (4)

#### ARC-1: Large-Scale Arch Macro Migration

Commit: MAIN:025e00f7
Message: adapt 3101 to 3510
Scale: 60 files, +145/-147 lines
Layers: cross-module (examples, gmm, mc2/common, mc2/3rd, mc2/matmul*)

Why selected:
- Mass migration: `__DAV_C310__` -> `__NPU_ARCH__ == 3510`
- 60 files across multiple modules -- shows cross-repo standardization
- Teaches: use standard NPU_ARCH macro, not legacy arch identifiers

#### ARC-2: Adapter Pattern for Arch-Specific Tiling

Commit: MAIN:57addf00
Message: MatmulAllReduce A5 Use Mc2InitTiling & Mc2CcTiling
Scale: 3 files, +2/-24 lines (net reduction!)

Why selected:
- Replaces arch35-specific tiling with generic Mc2InitTiling + Mc2CcTiling
- Net code reduction: adapter pattern eliminates arch-specific boilerplate
- Teaches: prefer common template + adapter over arch-specific implementations

#### ARC-3: Hardware Constraint Modeling

Commit: MAIN:84ef7bb3
Message: MatmulAllReduce CommFp8 Adapt CCU All2All DataCnt Limit
Scale: 2 files, +30/-0 lines

Why selected:
- Adds CCU hardware constraint to tiling parameters
- mMaxDataCnt = CCU_count * per_core_limit
- Teaches: hardware specs must be modeled as tiling constraints

#### ARC-4: NpuArch Chipset Mapping

Commit: MAIN:4da9f907
Message: alltoallmatmul A5npuarch芯片号更改
Scale: 2 files, +3/-3 lines

Why selected:
- Simple but error-prone: npuArch -> chipSetId mapping table update
- Cross-file consistency requirement (pipeline_builder.h + apt.cpp)
- Teaches: arch mapping must be synchronized across all references

---

### 9.2.1: REFACTOR Cases (5)

#### REF-1: Large-Scale Code Cleanup

Commit: MAIN:e8d52ffb
Message: [mc2]matmul_all_reduce clean code
Scale: 14 files, +150/-916 lines

Why selected:
- Net 766-line reduction across mc2/common and matmul_all_reduce
- Shows how to identify and remove cross-operator code duplication
- Phase 8 validated this commit against conventions

#### REF-2: Tiling Function Split (Complexity Reduction)

Commit: MAIN:916aa93c
Message: combineV2 tiling big function split
Scale: 1 file, +477/-416 lines

Why selected:
- Classic function decomposition: one 900+ line function -> 4 domain-specific functions
- GetExpertsAttrAndSetTilingData(), GetSharedAttrAndSetTilingData(), GetTpAndEpAttrAndSetTilingData()
- Teaches: when tiling functions exceed ~300 lines, split by parameter domain

#### REF-3: Cross-Module Normalization

Commit: MAIN:1bdd7212
Message: [aicore]整改
Scale: 26 files, +3/-55 lines

Why selected:
- 26 files with minimal change per file -- search-and-replace normalization
- Spans attention/, gmm/, mc2/ modules
- Teaches: how to execute repo-wide naming/API standardization

#### REF-4: V1 Module Reorganization

Commit: MAIN:f56a2f32
Message: D&C V1 Host侧cleancode
Scale: 6 files, +948/-625 lines

Why selected:
- Significant code reorganization of moe_distribute V1 (op_graph + op_host + op_api)
- Shows how to clean V1 code while maintaining backward compatibility
- Substantial code movement (not just deletion)

#### REF-5: API Parameter Removal (Cross-Layer)

Commit: MAIN:939b8c44
Message: quantgmmalltoallv para del offset
Scale: 11 files, +79/-409 lines

Why selected:
- API parameter deletion requires synchronized changes across op_api/proto/op_host
- Net 330-line reduction shows the cost of carrying unnecessary parameters
- Teaches: API simplification must touch all 4 layers

---

### 9.2.1: REVERT Cases (3)

#### REV-1: Sync Optimization Revert (4-hour turnaround)

Commit: MAIN:8ade8c3e (reverts 81ef4922)
Message: Revert "dispatch优化syncall"
Turnaround: 4 hours
Root cause: SetExpertTokenNums() sync ordering broke correctness

Why selected:
- Fastest revert in the codebase -- shows the risk of sync optimization
- Pairs with OPT-2 (successful sync optimization by same developer, same day)
- Teaches: sync changes need exhaustive path verification before merge

#### REV-2: Large-Scale Optimization Revert

Commit: DEV:f99a34dd
Message: 回退mm_rs优化
Scale: 13 files, +397/-3549 lines

Why selected:
- Massive code removal (3549 lines) of a performance optimization
- matmul_reduce_scatter_v2 tiling + kernel + CMakeLists affected
- Teaches: large-scale optimization rewrites need performance benchmarking before merge

#### REV-3: TilingKey Rectification Series (5 consecutive reverts)

Commits: DEV:a2b032dc, DEV:58a7ef18, DEV:20f4d541, DEV:8ead68cc, DEV:2001261f
Messages: Revert "tilingkey rectification for [various operators]"

Why selected:
- 5 sequential reverts of a tilingkey standardization initiative
- Each revert targets a different operator/arch combination
- Teaches: cross-cutting refactors (tilingkey format change) must be tested per-operator, not batch-applied

---

## 9.2.2: Error-Fix Commit Pairs

### Pair 1: Sync Optimization (4-hour cycle)
- Error: MAIN:81ef4922 (dispatch优化syncall, +71/-54)
- Fix: MAIN:8ade8c3e (Revert, same day)
- Pattern: aggressive sync optimization without exhaustive path testing

### Pair 2: TilingKey Rectification (batch failure)
- Error: DEV series of "tilingkey rectification" commits
- Fix: DEV:a2b032dc + 58a7ef18 + 20f4d541 + 8ead68cc + 2001261f (5 sequential reverts)
- Pattern: cross-cutting naming/format change applied without per-operator validation

### Pair 3: mm_rs Optimization (large rewrite)
- Error: DEV commit(s) optimizing matmul_reduce_scatter_v2
- Fix: DEV:f99a34dd (回退mm_rs优化, +397/-3549)
- Pattern: performance rewrite without sufficient regression testing

### Pair 4: TilingData Restructuring
- Error: DEV commit restructuring MC2 TilingData
- Fix: DEV:aee3fb63 (Revert "MC2 TilingData整改")
- Pattern: data structure change broke downstream consumers

### Pair 5: 3rd Party Decoupling
- Error: DEV commit decoupling mc2_3rd from nn repo
- Fix: DEV:3140d5a2 (Revert "mc2_3rd与nn仓解耦")
- Pattern: dependency decoupling broke build chain

---

## 9.2.3: Final Casebook Candidate List

29 landmark cases total:

| ID | Category | Repo:Hash | Message (short) | Files | Key Decision Point |
|----|----------|-----------|-----------------|-------|--------------------|
| NOP-1 | NEW_OP | MAIN:d79d1792 | add alltoallMM + 950 support | 107 | Complete 4-layer addition |
| NOP-2 | NEW_OP | DEV:40ce84bb | Add AttentionToFFN | 33 | Single complex fusion op |
| NOP-3 | NEW_OP | DEV:fd55c092 | Setup&Teardown framework | 75 | Multi-op batch scaffolding |
| NOP-4 | NEW_OP | MAIN:a9209ec8 | 新增MatmulAlltoAllA3 | 14 | Arch extension to existing op |
| FEA-1 | FEATURE | MAIN:48da3541 | MRS all2all+vec reducesum | 13 | New comm topology pattern |
| FEA-2 | FEATURE | MAIN:648691a4 | alltoallMatmul动态量化 | 25 | Full-stack quant feature |
| FEA-3 | FEATURE | MAIN:1d91a6c5 | tilingkey template | 9 | Parameterized kernel evolution |
| FEA-4 | FEATURE | MAIN:6c7079c1 | FullMesh Template DispatchV2 | 5 | Topology-driven kernel design |
| FEA-5 | FEATURE | DEV:681d2b1a | alltoallmatmul kernel实现 | 37 | Stage-based kernel composition |
| OPT-1 | OPTIM | MAIN:2aad8b3e | AttentionToFFN perf | 2 | Workspace alignment tuning |
| OPT-2 | OPTIM | MAIN:ea8f90e1 | 优化全核同步 | 2 | Fine-grained HardEvent sync |
| OPT-3 | OPTIM | MAIN:81ef4922 | dispatch syncall (REVERTED) | 2 | Failed sync optimization |
| BUG-1 | BUGFIX | MAIN:f8eb3c97 | big shape sync fail | 1 | HCCL handle constant error |
| BUG-2 | BUGFIX | MAIN:5a217f2d | GetAttrPointer type bug | 2 | int vs int64_t pointer cast |
| BUG-3 | BUGFIX | DEV:70a262ad | rank < 8 multi-layer bug | 5 | 3-layer systemic failure |
| BUG-4 | BUGFIX | DEV:798597da | dispatch OOM | 1 | sizeof unit confusion |
| BUG-5 | BUGFIX | DEV:1fab7bab | tiling init missing | 1 | Constructor init sequence |
| ARC-1 | ARCH | MAIN:025e00f7 | adapt 3101 to 3510 | 60 | Mass arch macro migration |
| ARC-2 | ARCH | MAIN:57addf00 | A5 Mc2InitTiling adapter | 3 | Common template + adapter |
| ARC-3 | ARCH | MAIN:84ef7bb3 | CCU DataCnt limit | 2 | Hardware constraint modeling |
| ARC-4 | ARCH | MAIN:4da9f907 | npuarch chipset mapping | 2 | Cross-file mapping sync |
| REF-1 | REFACTOR | MAIN:e8d52ffb | matmul_all_reduce cleanup | 14 | Cross-operator dedup |
| REF-2 | REFACTOR | MAIN:916aa93c | combineV2 tiling split | 1 | Function complexity reduction |
| REF-3 | REFACTOR | MAIN:1bdd7212 | aicore整改 | 26 | Repo-wide normalization |
| REF-4 | REFACTOR | MAIN:f56a2f32 | D&C V1 cleancode | 6 | V1 reorganization |
| REF-5 | REFACTOR | MAIN:939b8c44 | para del offset | 11 | Cross-layer API removal |
| REV-1 | REVERT | MAIN:8ade8c3e | Revert syncall | - | Sync optimization risk |
| REV-2 | REVERT | DEV:f99a34dd | 回退mm_rs优化 | 13 | Large rewrite risk |
| REV-3 | REVERT | DEV:5x commits | tilingkey rectification series | - | Batch refactor risk |

---

## 9.3: Decision Point Deep Analysis

### 9.3.1: NEW_OP Cases (4)

---

### Case NOP-1: 从零新增AlltoAllMatmul算子(含对称算子950扩展)

[Commit]: MAIN:d79d1792 (add new ops: alltoallMM; update MMalltoall to support Ascend950)
[类别]: NEW_OP
[涉及层]: op_host / op_kernel / op_graph / op_api (完整四层)
[变更规模]: 107 files, +18579/-114 lines

[场景]: 需要新增AlltoAllMatmul融合算子(先AlltoAll通信再MatMul计算)，同时把已有的MatmulAlltoAll算子扩展到Ascend950(arch35)。这是一对对称算子: AlltoAllMatmul = Comm -> Compute, MatmulAlltoAll = Compute -> Comm。

[约束]:
1. 必须同时支持arch32(910B)和arch35(950)两种架构，且kernel实现完全不同
2. arch32使用旧版tiling key(手动十进制编码+TILING_KEY_IS分支)，arch35使用APT框架(ASCENDC_TPL_ARGS_DECL)
3. AlltoAll通信涉及数据重排(permute/transpose)，需要额外workspace
4. 需要支持量化变体(int4/int8)，通过条件编译宏ORIG_DTYPE隔离
5. 两个算子的kernel共享mc2_templates基础设施(在matmul_allto_all/op_kernel/mc2_templates/下创建)

[决策]:
1. 四层结构完整并行创建: def.cpp(167行) + infershape.cpp(39行) + tiling_base.cpp/h(287+67行) + arch32 tiling(899行) + arch35 tiling(323行) + kernel双入口(.cpp+_apt.cpp) + proto.h(80行) + op_api(404行) + 全套tests/CMakeLists/docs/examples
2. op_def采用双AICore配置: 一个为ascend910_95(4种dtype组合)，另一个为ascend910b(10种dtype组合，含量化)，各自独立的DataType列表
3. kernel入口双版本: allto_all_matmul.cpp(旧版arch32，12模板参数) + allto_all_matmul_apt.cpp(新版arch35，3模板参数，通过宏展开)
4. mc2_templates框架创建: Stage-based流水线抽象(CommStage + ComputeStage + TransposeStage + PipelineTemplate)
5. tiling基类继承: AllToAllMatmulTilingBase继承TilingBaseClass的8步模板方法
6. HcclServerType在op_api层按arch分派: DAV_3510用CCU，ASCEND910B用MTE

[替代方案]:
- mc2_templates可以放在mc2/common/而非算子局部目录，但初期只有2个算子使用时放在局部更安全
- 可以把非量化和量化合成一个kernel入口，但ORIG_DTYPE的int4类型与非int4类型的Cast语义冲突，必须物理隔离

[后果]: 后续648691a4(FEA-2)在此基础上添加了动态量化支持，证明mc2_templates架构可扩展。mc2_templates后来被更多算子采用(从2到4+)。

[可迁移经验]:
1. 新增MC2算子必须同步创建完整四层+双arch(32+35)，单arch提交会导致兼容性问题
2. 当新增对称算子时，优先在其中一个算子目录下创建公共基础设施，待稳定后再考虑上浮到mc2/common/

---

### Case NOP-2: AttentionToFFN -- 不依赖MatMul的独立融合算子

[Commit]: DEV:40ce84bb (Add AttentionToFFN)
[类别]: NEW_OP
[涉及层]: op_host / op_kernel / op_graph / op_api (完整四层)
[变更规模]: 33 files, +5205/-8 lines

[场景]: 新增AttentionToFFN算子，实现Attention层到FFN层的数据派发(MoE路由)。纯AIV操作(无MatMul/Cube)，与MoE dispatch系列同属"数据路由"类。

[约束]:
1. 仅支持ascend910_93(A3)单一平台
2. 涉及RDMA窗口直驱通信，需要复用mc2/common/inc/kernel/moe_distribute_base.h
3. 多达8个输入(x, session_id, micro_batch_id, layer_id, expert_ids, expert_rank_table, scales, active_mask)
4. kernel单文件实现(608行)，不使用mc2_templates框架

[决策]:
1. 扁平化tiling: 650行单文件(不继承TilingBaseClass)，直接#include kernel侧tiling_key.h
2. jitCompile.flag = "static_true"(编译时确定shape，与MatMul系的"static_false"相反)
3. kernel直接操作RDMA窗口，而非使用HCCL高级API
4. 修改mc2/common中的moe_distribute_base.h将共享代码暴露给新算子
5. 环境变量ASCEND_ATTN_TO_FFN_WIN_TYPE用于运行时切换窗口类型(调试用)

[替代方案]:
- 可以继承MoeTilingBase做tiling，但语义差异大，复用收益低
- 可以使用HCCL AlltoAll做通信，但RDMA窗口直驱有更好的细粒度控制(逐token发送)

[后果]: 独立算子，后续未见重大修改。单平台设计简化了实现但限制了可移植性。

[可迁移经验]:
1. 当算子与现有系列编程范式不同时，不要强行复用公共基础设施，直接写扁平实现更清晰
2. RDMA窗口直驱的算子应复用moe_distribute_base.h中的窗口管理代码

---

### Case NOP-3: 三个Setup/Teardown算子的批量脚手架

[Commit]: DEV:fd55c092 (添加Dispatch Setup&Teardown及Combine Teardown算子框架)
[类别]: NEW_OP
[涉及层]: op_host / op_kernel / op_graph (框架骨架)
[变更规模]: 75 files, +2983/-0 lines

[场景]: 为v3架构(通信生命周期解耦)同时创建3个新算子目录框架: MoeDistributeDispatchSetup, MoeDistributeDispatchTeardown, MoeDistributeCombineTeardown。

[约束]:
1. 三个算子是v3架构的一部分(Setup -> 计算 -> Teardown)，必须同时创建
2. 仅限ascend910_95(arch35)
3. 本commit只创建骨架，实际逻辑后续填充

[决策]:
1. 批量创建: 一次commit创建75个文件覆盖3个算子的完整四层目录
2. 空kernel占位: .cpp只有GET_TILING_DATA一行，arch35/放.gitkeep
3. 空op_api/UT占位: .gitkeep和14行模板代码
4. jitCompile差异: Setup用"static_false"，CombineTeardown用"static_true"
5. CMakeLists中有拼写错误: CmakeLists.txt(小写m)

[替代方案]:
- 可以分三次commit分别创建，但同属一个架构特性，一起创建保证结构一致

[后果]: "框架先行"的开发模式，后续多次commit填充实际逻辑。

[可迁移经验]:
1. 同一架构特性的多个算子应一次性创建目录骨架，确保命名和结构一致
2. jitCompile.flag设定反映对算子运行时语义的提前规划

---

### Case NOP-4: 给已有算子扩展新arch(MatmulAlltoAll支持A3)

[Commit]: MAIN:a9209ec8 (新增MatmulAlltoAllA3)
[类别]: NEW_OP (实质是ARCH_ADAPT)
[涉及层]: op_host / op_kernel / op_api
[变更规模]: 14 files, +1490/-44 lines

[场景]: MatmulAlltoAll已有arch32(910B)和arch35(950)支持，要新增Ascend910_93(A3)支持。A3归属arch32但硬件能力不同(AICPU通信)。

[约束]:
1. A3归属arch32但与910B能力差异需独立处理(HcclServerType = AICPU)
2. IsCapable()需通过socVersion字符串判断(A3和A2共享arch32)
3. op_api需为A3设置不同的HcclServerType

[决策]:
1. 独立tiling类FpMatmulAllToAllTilingBaseA3: REGISTER_TILING_TEMPLATE_WITH_SOCVERSION注册
2. IsCapable()用socVersion字符串比较: `socVersionStr_ == "Ascend910_93"`
3. op_def新增ascend910_93 AICore配置(仅fp16/bf16)
4. op_api三路分派: DAV_3510->CCU, ASCEND910_93->AICPU, ASCEND910B->MTE
5. CMakeLists新增ascend910_93条件: --cce-auto-sync=off + -DAICORE_EXCEPTION_RESTART

[替代方案]:
- 可以在910B tiling中添加if/else，但A3的tiling参数差异大，独立文件更清晰

[后果]: 这种"同arch32但不同SoC"的适配模式后来在多个算子中推广。

[可迁移经验]:
1. 同一arch编码下的不同SoC适配，使用REGISTER_TILING_TEMPLATE_WITH_SOCVERSION + socVersion字符串是标准做法
2. op_api层HcclServerType三路分派(CCU/AICPU/MTE)是新增SoC支持时必须检查的点

---

### 9.3.1 NEW_OP综合: 四种新增模式

| 模式 | 代表 | 特征 | 典型文件数 |
|------|------|------|-----------|
| 完整四层新建 | NOP-1 | 对称算子+双arch+mc2_templates | 100+ |
| 独立融合算子 | NOP-2 | 单平台+扁平tiling+RDMA直驱 | 30+ |
| 骨架批量创建 | NOP-3 | 多算子框架先行+空占位 | 75+ |
| 已有算子arch扩展 | NOP-4 | 新SoC子路径+独立tiling | 14 |

---

### 9.3.2: FEATURE Cases (5)

---

### Case FEA-1: MatMulReduceScatter新增All2All+VecReduceSum通信路径

[Commit]: MAIN:48da3541 (MatMulReduceScatter 支持all2all+ vec reducesum)
[类别]: FEATURE
[涉及层]: op_host / op_kernel + mc2/common(新增公共kernel)
[变更规模]: 13 files, +1870/-29 lines

[场景]: MatMulReduceScatter原本只支持ReduceScatter通信拓扑。现在要新增All2All+VecReduceSum拓扑: 先AlltoAll交换数据，再用AIV做向量ReduceSum归约。

[约束]:
1. 仅arch35(950)
2. VecReduceSum需要AIV实现，涉及UB内存管理(double buffer + 32B对齐)
3. 需处理尾块(DataCopyPad防越界)
4. 公共实现需放在mc2/common/inc/kernel/供复用

[决策]:
1. 新增4个公共头文件到mc2/common/inc/kernel/: reduce_sum.h(258行核心算法), reduce_sum_cast_fp32.h(fp8变体), reduce_sum_utils.h, gm_ub_gm_copy.h
2. kernel入口通过APT tiling_key新增COMM_TYPE维度区分ReduceScatter vs All2All路径
3. AIV分核策略: 均分数据块到所有AIV核，每核独立处理rank-wise累加
4. ReduceSum采用"清零+逐rank累加"模式

[替代方案]:
- 公共代码可以放在算子局部，但考虑复用潜力放在mc2/common/更合理

[后果]: VecReduceSum公共代码后续被quant_bmm_reduce_scatter复用，验证了mc2/common/决策正确。

[可迁移经验]:
1. 新增通信拓扑通过APT tiling_key增加维度来选择路径
2. AIV侧向量计算工具应放mc2/common/inc/kernel/

---

### Case FEA-2: AlltoAllMatmul全栈动态量化(KC-Quant)

[Commit]: MAIN:648691a4 (ataKcQuantMatmul:alltoallMatmul动态量化)
[类别]: FEATURE
[涉及层]: op_host / op_kernel / op_graph (全栈)
[变更规模]: 25 files, +2436/-173 lines

[场景]: 给AlltoAllMatmul添加KC-Quant动态量化: 通信数据先做pertoken fp8量化再发送，接收后反量化再做MatMul。

[约束]:
1. 动态量化需运行时计算scale(pertoken)
2. mc2_templates需新增QuantizeStage和对应pipeline
3. tiling_key需新增QUANTMODE维度
4. 需新建arch35专用tiling文件(548行)

[决策]:
1. 独立tiling文件: allto_all_kc_quant_matmul_tiling_base(548+86行)，REGISTER_TILING_TEMPLATE_WITH_ARCH注册
2. mc2_templates扩展: 新增fp8_dynamic_quant_pertoken.h(413行) + quantize_stage.h + 四级流水线pipeline
3. tiling_key: QUANTMODE从{NON_QUANT}扩展为{NON_QUANT, KC_QUANT_FP8}

[替代方案]:
- 可以不扩展mc2_templates而直接在kernel写量化逻辑，但会让kernel难维护

[后果]: mc2_templates从2个pipeline扩展到3个，证明Stage-based架构支持功能扩展。

[可迁移经验]:
1. 量化标准扩展: 独立tiling文件(priority chain注册) + tiling_key新维度 + kernel模板分支
2. mc2_templates允许通过新增Stage扩展功能，无需修改已有Stage

---

### Case FEA-3: GMM算子TilingKey从手写十进制改为APT模板

[Commit]: MAIN:1d91a6c5 (alltoallvgmm/gmmalltoallv tilingkey template)
[类别]: FEATURE (实质是REFACTOR)
[涉及层]: op_host / op_kernel (双算子同步改造)
[变更规模]: 9 files, +403/-153 lines

[场景]: 两个GMM算子的tiling key从手写十进制编码(1000=FP16, 100=MM...)迁移到APT模板参数框架。kernel从12个TILING_KEY_IS分支缩减到2个if分支。

[约束]:
1. 必须两个对称算子同步改造
2. host和kernel两侧tiling_key严格匹配
3. host需#include kernel侧tiling_key.h(跨层引用)

[决策]:
1. 新建tiling_key.h: ASCENDC_TPL_ARGS_DECL定义4个模板参数
2. host改为调用GET_TPL_TILING_KEY()
3. kernel从12个分支缩减为2个dtype分支+模板自动展开

[后果]:
- 代码从12个分支缩减为2个(60行->17行)
- 后续tilingkey批量整改(REV-3)有5个连续revert，说明不能batch apply
- ASCENDC_TPL_DTYPE_DECL后来被FEA-5改为ASCENDC_TPL_UINT_DECL

[可迁移经验]:
1. TilingKey从手写到APT的迁移必须逐算子验证而非batch apply
2. tiling_key.h的canonical位置是op_kernel/，host通过相对路径#include

---

### Case FEA-4: MoE DispatchV2 FullMesh通信拓扑

[Commit]: MAIN:6c7079c1 (Add FullMesh Template for DispatchV2)
[类别]: FEATURE
[涉及层]: op_host(tiling) / op_kernel
[变更规模]: 5 files, +1387/-124 lines

[场景]: DispatchV2新增FullMesh拓扑: 每个rank直接向所有其他rank发送数据(RDMA窗口)。

[约束]:
1. FullMesh kernel 1086行巨型头文件，完全独立实现
2. RDMA窗口直驱 + 状态位管理 + CumSum + 核间软同步
3. dispatch_v2是bug密度最高的算子

[决策]:
1. 独立头文件(1086行)，从零实现不继承Ring/Hierarchy
2. tiling层379行变更: FullMesh独立窗口布局(WIN_STATE_OFFSET=500KB)
3. 复用moe_distribute_base.h公共基础设施

[后果]: FullMesh模板后来被revert(REV-1)，1086行巨型文件使bug定位困难。

[可迁移经验]:
1. 新通信拓扑作为独立头文件实现，不在已有拓扑中插入条件逻辑
2. RDMA窗口直驱kernel必须有充分UT覆盖

---

### Case FEA-5: mc2_templates框架首次落地

[Commit]: DEV:681d2b1a (alltoallmatmul/matmulalltoall kernel非量化实现)
[类别]: FEATURE
[涉及层]: op_host(微调) / op_kernel(核心)
[变更规模]: 37 files, +1461/-195 lines

[场景]: NOP-1的dev仓前身。首次创建mc2_templates流水线框架，实现两个对称算子的arch35 kernel。

[约束]:
1. 两个对称算子(Comm->Compute vs Compute->Comm)需共用基础设施
2. 流水线支持两种顺序

[决策]:
1. 三种Stage: CommStage(hccl_impl.h) + ComputeStage(fp_matmul.h) + TransposeStage(mc2_vec_transpose.h)
2. 两种Pipeline: CommTransCompute(AlltoAllMatmul) 和 ComputeTransComm(MatmulAlltoAll)
3. kernel入口通过宏展开(ALLTO_ALL_MATMUL_APT_FP_IMPL等)
4. tiling_key修正: DTYPE_DECL改为UINT_DECL
5. mc2_templates放在算子局部目录

[后果]: mc2_templates在main仓被原样合入(NOP-1)，并在FEA-2中成功扩展(Quantize pipeline)。

[可迁移经验]:
1. 对称算子对应通过Stage组合模式共享代码
2. 新框架先算子局部试点，成功后上浮到mc2/common/ -- "先局部后公共"

---

### 9.3.2 FEATURE综合: 五种扩展模式

| 模式 | 代表 | 扩展方法 |
|------|------|---------|
| 新通信路径 | FEA-1 | tiling_key增维+mc2/common新文件 |
| 全栈量化 | FEA-2 | 独立tiling+mc2_templates新Stage |
| TilingKey重构 | FEA-3 | tiling_key.h+kernel模板化 |
| 新通信拓扑 | FEA-4 | 独立头文件+tiling_key分支 |
| 框架首创 | FEA-5 | mc2_templates+宏展开 |

---

### 9.3.4: BUGFIX Cases (5)

---

### Case BUG-1: HCCL Handle常量错误导致大shape同步失败

[Commit]: MAIN:f8eb3c97 (fix bug:big shape sync fail.)
[类别]: BUGFIX
[涉及层]: op_kernel
[变更规模]: 1 file, +2/-2 lines

[场景]:
用户在大shape场景下运行MC2算子时出现通信同步失败。根因是hccl_impl.h中定义了
`static constexpr uint8_t MAX_HCCL_HANDLE_ = 16`，据此分配了HcclHandle数组和
taskSuccess数组各16个元素。但HCCL底层实际支持最多63个并行任务。当shape足够大时，
通信切分产生超过16个并行handle，写入越界导致同步失败。

[约束]:
- 这是一个magic number问题：HCCL spec定义的上限(63)没有以常量/宏的方式暴露给上层
- 代码中的注释"hccl只支持最多16个任务并行"是错误的，说明编码时对spec理解有误
- 该常量定义在mc2_templates/communication/communicator/hccl_impl.h中，是全局通信层共享代码，
  影响所有使用mc2_templates通信框架的算子(matmul_allto_all, matmul_all_reduce等)
- 小shape测试不会触发此bug，因为切分数量不会超过16

[决策]:
直接将常量从16修改为63，并同步更新注释。修改极其精简(2行)，但影响范围广泛。

[替代方案]:
1. 从HCCL头文件中获取宏定义(如果HCCL暴露了`HCCL_MAX_CONCURRENT_HANDLES`之类的常量)
   -- 更好，但HCCL可能没有暴露此常量
2. 运行时动态分配HcclHandle数组 -- 过度设计，编译期已知上限时静态数组更高效
3. 在kernel中添加运行时检查(handle数超限时报错而非越界)
   -- 属于防御性加固，但本次修复没做

[后果]:
修复后问题消失。但值得注意的是，mc2/common/inc/tiling/mc2_tiling_common_var.h中还有另一个
常量`MAX_HCCL_HANDLE_LIMIT = 32`(在tiling层使用)，与kernel层的63不一致。
这说明HCCL handle上限在项目中存在多处独立定义且数值不同，是潜在的定时炸弹。

[可迁移经验]:
硬件/底层库的容量常量必须从权威来源(spec文档、SDK头文件)获取，禁止凭经验"猜测"。
项目中同一语义的常量出现多处定义时，应统一为单一来源(mc2/common)并验证数值一致性。

[防御性编码建议]:
1. 所有与硬件能力相关的常量，在定义处附注spec出处(文档编号+页码)
2. 同一语义的常量在整个mc2/内只定义一次，其他地方通过include引用
3. 对固定大小数组的写入操作，添加编译期或运行时越界检查:
   `static_assert(idx < MAX_HCCL_HANDLE_)` 或 `ASCENDC_ASSERT(idx < MAX_HCCL_HANDLE_)`

---

### Case BUG-2: GetAttrPointer模板参数int/int64_t类型混用

[Commit]: MAIN:5a217f2d (fix type bug when using GetAttrPointer)
[类别]: BUGFIX
[涉及层]: op_host (op_tiling)
[变更规模]: 4 files, +48/-48 lines

[场景]:
MoE Dispatch/Combine算子的tiling层在获取算子属性时，大量使用了
`attrs->GetAttrPointer<int>(ATTR_XXX_INDEX)`，但实际属性存储类型是int64_t(8字节)。
GetAttrPointer<int>返回的是int指针(指向4字节)，解引用时只读取低4字节。
在little-endian平台上小数值碰巧能工作，但大数值或特定平台上会得到错误值。

修复覆盖4个文件(combine_v1, combine_v2, dispatch_v1, dispatch_v2)中的48处调用，
全部从`GetAttrPointer<int>`改为`GetAttrPointer<int64_t>`。

[约束]:
- 这是一个系统性copy-paste bug: 4个文件都犯了同样的错误，说明可能是从同一个
  初始实现copy出来的
- 在little-endian + 属性值 < 2^31的测试场景下，此bug完全不可见
- 属性的实际存储类型由op_def层的AddAttr<int64_t>()决定，tiling层必须与之匹配
- 同一文件中有些属性已经正确使用了int64_t(如zeroExpertNumPtr, copyExpertNumPtr)，
  说明是后期新增的属性用了正确类型，而早期属性沿用了错误模板

[决策]:
统一修复: 所有MoE算子的GetAttrPointer模板参数统一改为int64_t。
修复范围覆盖了moe_distribute_combine(v1+v2)和moe_distribute_dispatch(v1+v2)的全部属性获取。

[替代方案]:
1. 只修复已知会出错的属性 -- 不好，类型匹配是原则性问题，不应该选择性修复
2. 写一个类型安全的wrapper函数，从op_def推导正确类型
   -- 更好的长期方案，但当前框架不支持编译期类型推导
3. 在GetAttrPointer内部添加类型大小校验 -- 需要修改公共框架，超出算子开发者权限

[后果]:
修复后无后续问题。但Grep统计显示，修复后mc2/仓库中仍有29处`GetAttrPointer<int>`
(分布在14个文件中)，而int64_t版本有452处(55个文件)。
残留的int版本可能有些确实是int类型属性，但也可能存在遗漏的同类bug。

[可迁移经验]:
GetAttrPointer的模板参数必须与op_def中AddAttr的类型严格一致。
当发现一处类型不匹配时，应全量搜索同一仓库中所有同类调用(Grep pattern: `GetAttrPointer<int>`)，
批量检查而非只修当前报错点。

[防御性编码建议]:
1. 在code review中，GetAttrPointer<T>的T必须与op_def中的属性类型声明交叉核对
2. 建立MoE属性类型查找表(ATTR_INDEX -> type)，所有tiling文件从查找表获取正确类型
3. 新增属性时，在op_def和tiling中同时添加，不要先写tiling再补op_def
4. 如果框架支持，考虑用static_assert或编译期检查验证类型匹配

---

### Case BUG-3: 单机rank<8场景下Dispatch/Combine三层联动bug

[Commit]: DEV:70a262ad (Fix Dispatch and Combine, rank < 8)
[类别]: BUGFIX
[涉及层]: op_host (infershape + op_tiling) + op_kernel
[变更规模]: 5 files, +93/-46 lines

[场景]:
单机Dispatch/Combine场景(epWorldSize < 8，即卡数少于8张)下，出现三个并发问题:

问题1(infershape层 -- 括号优先级):
```
// 修复前: 先乘后除，整数除法丢精度方向不同
epRecvCountShape = *epWorldSize * localExpertNum + globalBsReal * 2 * k * (*epWorldSize) / RANK_NUM_PER_NODE
// 修复后: 先除后乘，语义正确(每个node的数量 * node数)
epRecvCountShape = *epWorldSize * localExpertNum + globalBsReal * 2 * k * ((*epWorldSize) / RANK_NUM_PER_NODE)
```
当epWorldSize < RANK_NUM_PER_NODE(8)时，`(*epWorldSize) / RANK_NUM_PER_NODE`等于0，
但`(*epWorldSize) * ... / RANK_NUM_PER_NODE`可能非零，导致PTA推导的shape与算子预期不一致。

问题2(tiling层 -- BlockDim条件):
单机场景下不需要layered通信，aicpuBlockDim应为1而非固定的AICPU_NUM_BLOCKS_A2。
同时，CommAlg检查函数需要提前判断epWorldSize <= 8时直接返回fullmesh。

问题3(kernel层 -- 对齐与Mask限制):
- dataSizePerRank_计算缺少UB_ALIGN对齐: `dataSize_ / worldSize_`改为`dataSize_ / worldSize_ / UB_ALIGN * UB_ALIGN`
- statusWorldSize缺少VEC_UB_ALIGN(256字节)对齐: 需`RoundUp(..., VEC_UB_ALIGN)`
- performanceInfo buffer缺少RoundUp对齐: `performanceInfoSize_ * sizeof(int64_t)`需对齐到B64_PER_BLOCK
- Add指令的Mask限制: 当localExpertNum超过32时，Mask参数(uint64_t的低32位)溢出。
  修复方案: 从直接传count模式改为按BITS_PER_U32分批处理，每批最多32个expert
- DataCopy改为DataCopyPad: 非对齐大小的数据搬运必须用Pad版本
- SingleServerDispatch/Combine中缺少sync和bufferId切换逻辑

[约束]:
- rank<8是罕见的部署配置(主流场景都是8卡一机)，开发和测试时容易被忽略
- 三个问题在rank=8时恰好都不触发:
  - 问题1: 8/8=1，乘法顺序无影响
  - 问题2: 8>8为false走正常路径
  - 问题3: worldSize=8时对齐恰好满足
- 这是一个典型的"边界条件叠加"bug: 每个问题单独看都很小，但必须同时修复才能正常工作

[决策]:
在三个层(infershape/tiling/kernel)同时修复，确保单机rank<8场景全链路正确:
- infershape: 修正括号优先级
- tiling: 添加epWorldSize<=8的提前返回+动态aicpuBlockDim
- kernel: 全面的对齐修复 + Mask分批处理 + DataCopyPad替代DataCopy + sync逻辑补全

[替代方案]:
1. 直接在入口层拒绝rank<8(添加check失败) -- 规避而非修复，不可接受
2. 只修一层(如只修kernel) -- 不行，三层问题独立存在，必须全部修复
3. 为rank<8创建独立的kernel实现路径 -- 过度设计，通过条件分支即可

[后果]:
修复后单机场景正常工作。但这个commit本身规模较大(+93/-46，跨5个文件)，
说明一个边界条件bug的修复成本可以很高。后续commit(DEV:798597da，即BUG-4)
继续修复了同一文件中的另一个相关bug(Duplicate size参数单位错误)。

[可迁移经验]:
当一个配置参数(如worldSize)改变了隐含的对齐/大小假设时，需要从infershape到kernel
全链路审查所有对该参数的除法、取模、对齐操作。
AscendC的Vec/DataCopy指令对buffer对齐有严格要求，任何非8倍数的count都要用RoundUp+Pad。

[防御性编码建议]:
1. 对所有整数除法`a / b`添加注释说明"a是否保证被b整除"，不能保证时必须用RoundUp
2. AscendC的Duplicate/Add/DataCopy的count参数，永远用RoundUp对齐到硬件要求粒度
3. 为每个通信域参数(epWorldSize, tpWorldSize, worldSize)建立边界测试:
   值为1, 2, 7, 8, 9, 16, 64 -- 覆盖"小于/等于/大于RANK_NUM_PER_NODE"三种情况
4. Mask参数超过32位时，必须分批处理(按BITS_PER_U32拆循环)
5. 使用DataCopyPad而非DataCopy作为默认选择，除非能证明数据量一定对齐

---

### Case BUG-4: Duplicate<int32_t>的size参数字节/元素单位混淆(OOM)

[Commit]: DEV:798597da (fix dispatch oom bug)
[类别]: BUGFIX
[涉及层]: op_kernel
[变更规模]: 1 file, +1/-1 lines

[场景]:
MoeDistributeDispatchA2的CleanUpFlags()函数中，使用Duplicate<int32_t>清零statusTensor_:
```cpp
// 修复前: worldSize_ * DATA_OFFSET 是字节数
Duplicate<int32_t>(statusTensor_, 0, worldSize_ * DATA_OFFSET);
// 修复后: 除以sizeof(int32_t)得到元素数
Duplicate<int32_t>(statusTensor_, 0, worldSize_ * DATA_OFFSET / sizeof(int32_t));
```
Duplicate<T>(tensor, value, count)的count参数语义是"T类型元素个数"，不是字节数。
传入字节数导致清零范围是实际需要的4倍(sizeof(int32_t)=4)，造成写入越界、内存踩踏，
最终表现为OOM。

[约束]:
- Duplicate<T>的API文档中count参数描述为"元素个数"，但在MC2代码中，
  很多buffer大小计算混合使用字节和元素两种单位，容易混淆
- DATA_OFFSET本身的单位是字节(用于指针偏移)，而worldSize_是无单位的数量
- 同一文件(moe_distribute_dispatch_a2.h)中其他Duplicate调用有的正确有的错误，
  没有统一的编码惯例
- BUG-3的修复(70a262ad)大规模改动了同一文件，但没有发现这个pre-existing bug

[决策]:
最小化修复: 仅在`worldSize_ * DATA_OFFSET`后追加`/ sizeof(int32_t)`。
一行代码修复，精确定位。

[替代方案]:
1. 将DATA_OFFSET重新定义为以int32_t元素为单位(DATA_OFFSET_ELEMENTS)
   -- 会引起连锁修改，风险大
2. 封装一个ClearTensor(tensor, byteSize)工具函数自动做单位转换
   -- 更好的长期方案，但本次只做最小修复
3. 使用memset语义的API(如果AscendC提供) -- 不确定是否存在

[后果]:
修复后OOM消失。值得注意的是，这个bug与BUG-3在同一文件中，BUG-3比BUG-4早2天提交。
BUG-3大规模改动了该文件的对齐和Duplicate调用，但没有发现CleanUpFlags中这个
已经存在的bug。这说明大改动的code review很难发现pre-existing的隐蔽bug。

[可迁移经验]:
AscendC的Duplicate/DataCopy等API的count参数单位必须与模板类型T一致(元素数，不是字节数)。
代码中涉及buffer大小的变量应在命名中显式标注单位(如dataOffsetBytes vs dataOffsetElements)。

[防御性编码建议]:
1. 变量命名显式标注单位: `dataSizeBytes`, `dataCountElements`, `dataBlockNum`
2. 或者统一使用"字节"作为内部表示，在调用API时显式除以sizeof(T):
   `Duplicate<int32_t>(tensor, 0, sizeInBytes / sizeof(int32_t))`
3. 对所有Duplicate/DataCopy调用做code review时，强制检查count参数的单位
4. 编写辅助宏: `#define ELEM_COUNT(bytes, T) ((bytes) / sizeof(T))`，消除手动计算

---

### Case BUG-5: Tiling子类构造函数遗漏InitCompileInfo()调用

[Commit]: DEV:1fab7bab (MatmulAllReduce伪量化Tiling GetPlatformInfo错误修复)
[类别]: BUGFIX
[涉及层]: op_host (op_tiling)
[变更规模]: 1 file, +6/-2 lines

[场景]:
MatmulAllReduce的伪量化(weight quant)tiling在arch35上运行时，
GetPlatformInfo()返回垃圾值，导致tiling参数计算全部错误。

根因: weight_quant_matmul_all_reduce_tiling_910_95.h中的两个Helper类
(WeightQuantTilingTransferHelperA5和WeightQuantTilingTransferHelperASA5)
继承了Mc2WeightQuantBatchMatmulV2RegBase / Mc2WeightQuantBatchMatmulV2TilingAS，
但构造函数体为空`{}`，没有调用基类的`InitCompileInfo()`。

基类Mc2WeightQuantBatchMatmulV2RegBase的构造函数中，有条件调用InitCompileInfo:
```cpp
explicit Mc2WeightQuantBatchMatmulV2RegBase(gert::TilingContext* context)
    : Mc2WeightQuantBatchMatmulV2Tiling(context) {
    if (context->GetCompileInfo() == nullptr) {
        InitCompileInfo(); // <-- 基类自己调用了
    }
    tilingSolver_.Init();
}
```
但子类WeightQuantTilingTransferHelperA5通过初始化列表调用基类构造时，
传入的是另一个对象的context_，且走的是不同的初始化路径(通过transfer helper间接构造)。
此时context->GetCompileInfo()可能不为nullptr(已有过期数据)，跳过InitCompileInfo()，
导致platformInfo等字段未正确初始化。

[约束]:
- C++继承链较深: WeightQuantTilingTransferHelperA5 -> Mc2WeightQuantBatchMatmulV2RegBase
  -> Mc2WeightQuantBatchMatmulV2Tiling -> TilingBaseClass
- 基类的InitCompileInfo()有条件判断(GetCompileInfo()==nullptr)，子类的使用场景不一定满足此条件
- 这是arch35特有的tiling路径(weight quant matmul all reduce)，非量化路径不受影响
- Transfer Helper模式(把一个tiling对象的结果转移到另一个)是MC2特有的设计，
  在这种场景下基类的自动初始化假设不成立

[决策]:
在两个Helper子类的构造函数中显式调用基类的InitCompileInfo():
```cpp
WeightQuantTilingTransferHelperA5(...) : Mc2WeightQuantBatchMatmulV2RegBase(...) {
    Mc2WeightQuantBatchMatmulV2RegBase::InitCompileInfo(); // 新增
}
```
不依赖基类构造函数中的条件判断，而是无条件调用。

[替代方案]:
1. 修改基类，去掉GetCompileInfo()==nullptr的条件判断，改为无条件调用
   -- 可能影响其他子类，风险大
2. 在Transfer Helper中先清空compileInfo再调用基类构造
   -- hack，不直观
3. 重新设计Transfer Helper模式，不通过继承实现
   -- 更好的长期方案，但重构成本高

[后果]:
修复后GetPlatformInfo()返回正确值。两个Helper类都做了同样的修复(对称性)。
这个bug暴露了MC2 tiling类层次设计的一个根本问题: 基类的"智能初始化"(条件调用)
在多层继承的Transfer Helper场景下假设不成立。

[可迁移经验]:
继承深层次的tiling类时，不能假设基类构造函数已完成所有初始化。
特别是Transfer Helper/Adapter模式下，必须显式检查每个初始化步骤是否被正确执行。
对于TilingBaseClass的8步DoTiling流程中的每一步(尤其是InitCompileInfo/GetPlatformInfo)，
在子类中要确认它们在当前构造路径上确实被调用了。

[防御性编码建议]:
1. 继承tiling基类时，构造函数checklist:
   - [ ] 基类构造函数是否被正确调用？
   - [ ] InitCompileInfo()是否在当前构造路径上被执行？
   - [ ] 如果是Transfer Helper，基类的条件初始化是否仍然有效？
2. 在GetPlatformInfo()/GetShapeAttrsInfo()入口添加断言:
   `ASSERT(compileInfo_ != nullptr, "InitCompileInfo not called")`
3. 考虑将InitCompileInfo()标记为final或在基类中用flag追踪初始化状态
4. Transfer Helper类应该有自己的构造checklist文档(哪些初始化是必须显式调用的)

---

### 9.3.4 BUGFIX综合分析: 五类bug根因模式

从5个BUGFIX案例中提炼出的根因模式和防御策略:

#### 根因模式1: 硬件/库常量猜测(BUG-1)
表现: 代码中的常量值与硬件spec不一致
根因: 编码时凭经验猜测而非查阅spec
防御: 常量定义处附注spec出处 + 全仓唯一定义

#### 根因模式2: 类型reinterpret(BUG-2)
表现: 模板参数类型与实际存储类型不匹配
根因: copy-paste传播 + little-endian小值场景掩盖bug
防御: GetAttrPointer<T>的T必须与AddAttr<T>交叉核对 + 全仓Grep扫描

#### 根因模式3: 边界条件叠加(BUG-3)
表现: 多个独立的边界假设在非主流配置下同时失效
根因: 开发测试只覆盖主流配置(rank=8) + 整数除法/对齐的隐含假设
防御: 参数边界测试矩阵(1,2,7,8,9,16,64) + 全链路审查除法/对齐

#### 根因模式4: API参数单位混淆(BUG-4)
表现: 字节数被当作元素数传入API
根因: 代码中字节和元素两种单位混用，命名不区分
防御: 变量命名显式标注单位(Bytes/Elements/Blocks) + 辅助宏ELEM_COUNT

#### 根因模式5: 继承链初始化假设失效(BUG-5)
表现: 子类构造后基类的某些字段未初始化
根因: 基类的条件初始化在子类的特殊构造路径上假设不成立
防御: 子类构造checklist + 关键字段的初始化断言 + 避免过深的继承链

---

### 9.3.3: OPTIMIZATION Cases (3)

#### Case OPT-1: AttentionToFFN动态workspace对齐优化

[Commit]: MAIN:2aad8b3e (AttentionToFFN performance optimization)
[类别]: OPTIMIZATION
[涉及层]: op_host(tiling) + op_kernel
[变更规模]: 2 files, +76/-66 lines

[场景]:
AttentionToFFN算子的异步通信模式(isSync)下，每个FFN节点需要通过workspace记录token发送状态。
旧实现中，每个FFN节点占固定WORKSPACE_ELEMENT_STRIDE=32个int32(128B)，导致workspace浪费且逻辑散乱:
- 每发一个token就要等MTE3完成再写状态(串行化)
- 状态初始化需要SyncAll全核同步
- SetFlagToFFN阶段需要逐FFN从GM读取状态(频繁MTE2_S同步)

[约束]:
1. workspace地址必须满足128B对齐(WORKSPACE_ELEMENT_OFFSET=128)
2. 多AIV核并行发送token，需保证各核状态数据不互相覆盖
3. FFN状态读回时需要跨AIV聚合(判断任意AIV是否向某FFN发了token)

[决策]:
选择了"本地累计+批量写出+向量ReduceSum聚合"的三步策略:
1. op_host层: workspace大小从`ffnNum * 128B`改为`ffnNumAlignSize * aivNum`，按AIV隔离
2. 每AIV维护本地ffnStatusTensor_(UB上)，发token时只`SetValue(toRankId - ffnStartRankId, 1)`标记，不写GM
3. 发送循环结束后一次性DataCopy到GM的`aivId * aivWorkspaceStride`位置
4. SetFlagToFFN阶段: 一次性把所有AIV的状态读回UB，用ReduceSum跨AIV聚合，再逐FFN检查sum>0

附带优化:
- FindExpertRank中移除逐行DataCacheCleanAndInvalid，改为循环一次性批量刷新expertRankTable Cache
- statusTensor_从固定32B改为按sendNum动态分配(若sendNum=0则只1个UB_ALIGN)
- SetFlagInAttn复用expertIdsBuf_作为attnStatusBuf_，减少buffer开销
- 彻底移除旧的SetFFNStatus函数及其Process入口处的SyncAll+PipeBarrier

[替代方案]:
- 保持旧方案(每token写GM + MTE3同步): 简单但串行化严重，每token有MTE3等待开销
- 使用atomic操作直接在GM上原子累加: 不可行，AscendC不支持AIV间的GM原子操作
- 用SyncAll全核同步后让单核汇总: 旧代码就是这样做的(SetFFNStatus用SyncAll)，但SyncAll开销大

[后果]:
此commit成功合入未被revert。

[可迁移经验]:
异步通信状态管理的核心原则: 写操作尽量在本地UB累积，读操作用向量ReduceSum批量聚合，避免在发送循环中频繁跨核同步。workspace布局应按AIV隔离(aivId * stride)而非按目标节点隔离。

---

#### Case OPT-2: 成功的细粒度HardEvent同步替代SyncAll

[Commit]: MAIN:ea8f90e1 (优化全核同步)
[类别]: OPTIMIZATION
[涉及层]: op_kernel
[变更规模]: 2 files, +85/-65 lines

[场景]:
moe_distribute_dispatch_v2的UpdateTokenNumsOut()函数需要汇总各核的expertTokenNums数据。
旧实现中，moeExpertNumPerRank > 1时需要SyncAll全核同步，等lastCore读取sendCountsGlobal再逐一从GM读各专家的cumsum。
SyncAll是全核barrier，阻塞所有AIV核直到最慢的核完成，浪费了其他核的空闲周期。

[约束]:
1. 各AIV核分别处理不同专家的token计数，写入window状态区
2. expertTokenNums要么是cumsum(expertTokenNumsType_==0)要么是per-expert(!=0)
3. 多核写的sendCountsGlobal数据需要被读取来计算总数
4. ReduceSum需要独立的workspace buffer(不能复用其他用途的buffer)

[决策]:
1. 用FIRST_CORE(aivId==0)替代lastCore做汇总，避免等待所有核到达barrier
2. 从window状态区的float值(windowInstatusFp32Tensor_)直接读取每个专家每个rank的状态
3. 用ReduceSum向量归约替代逐个GM读取: 按(localExpertNum * epWorldSize)批量DMA到statusBuf_，然后对每个专家的epWorldSize个值做ReduceSum
4. 新增tokenNumBuf_(UB)存放计算结果，DataCopyPad批量写出到expertTokenNumsOutGMTensor_
5. 新增独立workLocalBuf_作为ReduceSum的workspace

关键技术细节:
- 新增3个buffer: tokenNumBuf_(moeExpertNumPerRank * sizeof(int64_t)), statusBuf_(moeExpertNumPerRank * epWorldSize * UB_ALIGN), workLocalBuf_(epWorldSize对齐)
- statusBuf_大小: `moeExpertNumPerRank_ * epWorldSize_ * UB_ALIGN` (按实际维度精确分配)
- 同步改用细粒度HardEvent: S_V/V_S(ReduceSum前后), S_MTE3(写出前), MTE2_V(读入后)
- 拆分IsNeedAllgather路径: 非AllGather场景直接走SetExpertTokenNums，AllGather场景走AllGatherSetExpertTokenNumsAndTpRecvCount

[替代方案]:
- 保留SyncAll + lastCore汇总(旧方案): 安全但同步开销大
- 用PipeBarrier替代SyncAll: PipeBarrier只barrier当前core的pipeline，无法跨核同步
- 复用已有buffer(gatherMaskOutBuf_)做ReduceSum workspace: 这正是OPT-3的做法，导致buffer冲突被revert

[后果]:
此commit成功合入，3天后未被revert。同一开发者3天前的OPT-3失败后，修复了关键问题(buffer隔离+大小计算)后重新提交。

[可迁移经验]:
替代SyncAll的安全做法: (1)选择固定核(FIRST_CORE)做汇总而非lastCore，(2)用ReduceSum向量归约替代逐个标量读取，(3)为ReduceSum分配独立workspace buffer而非复用现有buffer。buffer大小必须精确匹配数据维度(expert_num * world_size)。

---

#### Case OPT-3: 失败的同步优化(5.5小时内被revert)

[Commit]: MAIN:81ef4922 (dispatch优化syncall) -> MAIN:8ade8c3e (Revert)
[类别]: OPTIMIZATION (失败)
[涉及层]: op_kernel
[变更规模]: 2 files, +71/-54 lines (提交) -> +58/-75 lines (revert)
[时间线]: 02-02 11:33提交, 02-02 16:58 revert (约5.5小时)

[场景]:
与OPT-2完全相同的场景: 替代moe_distribute_dispatch_v2中的SyncAll。
同一开发者(Davon14272/liuwenda4)，同一天(2026-02-02)提交。

[约束]:
同OPT-2。

[决策]:
采用了与OPT-2几乎相同的架构思路(ReduceSum替代SyncAll)，但实现上有三个关键差异:

差异1 -- statusBuf_大小计算不同:
```
// OPT-3 (失败): 按核数平分后对齐
uint32_t statusBufCntAlign = Ceil(Ceil(totalExpertNum_, aivNum_), 8) * 8;
uint32_t statusBufSize = statusBufCntAlign * UB_ALIGN;

// OPT-2 (成功): 按实际维度精确分配
uint32_t statusBufSize = moeExpertNumPerRank_ * epWorldSize_ * UB_ALIGN;
```
OPT-3的计算方式按"总专家数/核数"分配，与实际的数据搬运维度(localExpertNum * epWorldSize)不匹配，可能导致buffer溢出或数据错位。

差异2 -- ReduceSum workspace复用了gatherMaskOutBuf_:
```
// OPT-3 (失败): 复用已有buffer
LocalTensor<float> gatherMaskOutTensor = gatherMaskOutBuf_.Get<float>();
ReduceSum(..., gatherMaskOutTensor, ...);

// OPT-2 (成功): 新增独立buffer
tpipe_->InitBuffer(workLocalBuf_, workLocalBufSize);
LocalTensor<float> workLocalTensor = workLocalBuf_.Get<float>();
ReduceSum(..., workLocalTensor, ...);
```
gatherMaskOutBuf_在其他流程中可能还在使用(AllGather路径)，复用它做ReduceSum workspace可能导致数据污染。

差异3 -- tokenNumBuf_大小不同:
```
// OPT-3: moeExpertNumPerRank_ * UB_ALIGN (不考虑sizeof(int64_t))
// OPT-2: Ceil(moeExpertNumPerRank_ * sizeof(int64_t), UB_ALIGN) * UB_ALIGN
```

差异4 -- LocalWindowCopy末尾同步:
```
// OPT-3: SyncFunc<AscendC::HardEvent::MTE3_MTE2>() (替代PipeBarrier<PIPE_MTE3>)
// OPT-2: 直接删除此同步(不需要)
```
OPT-3用细粒度HardEvent替代了粗粒度PipeBarrier，但MTE3_MTE2可能不是正确的事件对。

[替代方案]:
就是OPT-2的做法——独立buffer + 精确大小计算。

[后果]:
提交于2026-02-02 11:33，同日16:58被revert(约5.5小时)。
3天后(2026-02-05)同一开发者提交OPT-2，修复了上述所有差异点，成功合入。

[可迁移经验]:
kernel buffer优化的三条铁律: (1)buffer大小必须精确匹配数据搬运的实际维度，不能用"近似对齐"凑; (2)ReduceSum等向量操作的workspace buffer必须独立分配，不能复用流水线中其他阶段的buffer; (3)同步事件(HardEvent)的选择必须匹配实际的数据依赖关系，而非机械替换。

---

#### OPT-2 vs OPT-3 对比分析

同一开发者、同一天、同一优化目标(替代SyncAll)，两次提交的成败对比:

| 维度 | OPT-3 (失败, 02-02) | OPT-2 (成功, 02-05) |
|------|---------------------|---------------------|
| statusBuf大小 | totalExpertNum/aivNum对齐 | moeExpertNumPerRank * epWorldSize精确 |
| ReduceSum workspace | 复用gatherMaskOutBuf_ | 独立workLocalBuf_ |
| tokenNumBuf大小 | moeExpertNumPerRank * UB_ALIGN | Ceil(moeExpertNumPerRank * sizeof(int64_t), UB_ALIGN) * UB_ALIGN |
| LocalWindowCopy同步 | SyncFunc(MTE3_MTE2) | 删除(不需要) |
| 结果 | 5.5小时内revert | 稳定合入 |

核心洞察:
1. Buffer大小计算必须基于"谁搬多少数据"的精确建模，而非"大概够用"的近似
2. Buffer复用是kernel优化中最危险的操作 -- 必须证明两个使用场景完全不重叠
3. 同步优化不是"把SyncAll换成HardEvent"那么简单，每个HardEvent必须精确对应一个真实的数据依赖
4. 开发者在3天内完成了从失败到成功的迭代，说明revert本身不是坏事，快速revert+修复比强行修补更安全

---

### 9.3.3 OPTIMIZATION综合分析: 三类优化模式的安全性谱系

从3个OPTIMIZATION案例中提炼的安全性判断框架:

#### 安全优化(高成功率):
- workspace布局重组(OPT-1): 改变"何时写GM"但不改变"写什么"，用向量操作替代标量循环
- 批量Cache刷新(OPT-1): 将逐行DataCacheCleanAndInvalid改为循环一次性刷新
- Buffer按需分配(OPT-1): 动态计算buffer大小而非固定大小

#### 高风险优化(需额外验证):
- SyncAll替代(OPT-2/3): 核心数据依赖分析必须精确，buffer必须独立
- 同步事件细化(PipeBarrier -> HardEvent): 必须证明新事件对匹配实际数据流

#### 判断准则:
1. 优化是否改变了数据依赖关系？(只改"何时"比改"什么"安全)
2. 是否引入了buffer复用？(必须证明生命周期不重叠)
3. buffer大小计算是否从实际数据维度推导？(不能用近似值)
4. 失败代价多大？(SyncAll优化失败=功能不正确，workspace优化失败=OOM)

---

### 9.3.5: ARCH_ADAPT Cases (4)

#### Case ARC-1: 大规模arch宏迁移(__DAV_C310__ -> __NPU_ARCH__ == 3510)

[Commit]: MAIN:025e00f7 (adapt 3101 to 3510)
[类别]: ARCH_ADAPT
[涉及层]: op_host + op_kernel + 3rd + common (跨全部层)
[变更规模]: 60 files, +145/-147 lines (跨examples/gmm/mc2三个顶层模块)

[场景]:
仓库中使用`__DAV_C310__`宏标记Ascend950(A5芯片)的条件编译分支。
`__DAV_C310__`是旧命名(DAV_C = 达芬奇C系列, 310 = 芯片号)，
需要迁移到标准化的`__NPU_ARCH__`宏体系(`__NPU_ARCH__ == 3510`表示arch35的10型芯片)。

[约束]:
1. 纯机械替换: 每个`#if defined(__DAV_C310__)`改为`#if defined(__NPU_ARCH__) && (__NPU_ARCH__ == 3510)`
2. 同一文件中可能有正向(`#if defined`)和反向(`#if !defined`)两种用法
3. 变更跨3个顶层模块(examples, gmm, mc2)和mc2内7个子模块(common, 3rd, matmul_all_reduce, moe_distribute_combine, moe_distribute_dispatch_v2, quant_all_reduce, quant_reduce_scatter, moe_init_routing系列)
4. 不能改变任何运行时行为，纯编译期宏替换

[决策]:
全量search-and-replace，一次性提交60个文件。替换规则:
- `#if defined(__DAV_C310__)` -> `#if defined(__NPU_ARCH__) && (__NPU_ARCH__ == 3510)`
- `#if !defined(__DAV_C310__)` -> `#if !(defined(__NPU_ARCH__) && (__NPU_ARCH__ == 3510))`
- `#ifdef __DAV_C310__` -> `#if defined(__NPU_ARCH__) && (__NPU_ARCH__ == 3510)`
- 注释中的`// __DAV_C310__` -> `// __NPU_ARCH__ == 3510`

统计: diff中约90处旧宏引用被替换为约198处新宏引用(因为新格式更长，涉及增删行)。

特殊处理:
- mc2/common/mc2_templates/communication/hccl_a2av_op.h: 将`#ifdef __DAV_C310__\n#include <...>`简化为无条件include(顺便清理了不必要的条件编译)
- matmul_all_reduce_tiling_base.cpp中的注释也同步更新: `// __DAV_C310__` -> `// __NPU_ARCH__ == 3510`
- 顺便清理了2行空注释(`// __DAV_C310__\n// end __DAV_C310__`只有注释没有实际ifdef)

[替代方案]:
- 分模块分批提交: 安全但增加review负担，且中间状态(部分旧/部分新)可能导致混淆
- 通过宏define兼容: `#define __DAV_C310__ (defined(__NPU_ARCH__) && __NPU_ARCH__ == 3510)`，不需要改60个文件。但这只是推迟问题
- 只改MC2模块: 留下gmm和examples不改。但不一致更危险

[后果]:
成功合入。但后续ARC-4(MAIN:4da9f907)暴露了遗漏: allto_all_matmul中有`__NPU_ARCH__ == 3101`(手动输入错误的arch编号)未被此次迁移覆盖。原因是allto_all_matmul在ARC-1时可能还未使用旧的`__DAV_C310__`宏(它直接写了错误的数值3101)，所以全局替换没有命中。

[可迁移经验]:
大规模宏迁移必须一次性完成(不要分批，避免中间不一致状态)，但"一次性"不等于"不会遗漏"。执行后应做全仓grep验证旧宏是否清零: `grep -r '__DAV_C310__' .`。更重要的是，也要grep新宏的值是否一致: `grep -r '__NPU_ARCH__.*3101' .` 来捕捉手动输入错误。标准化的arch宏(`__NPU_ARCH__`)优于产品线宏(`__DAV_C310__`)，因为前者的数值编码规则统一(arch位*1000 + 芯片型号)。

---

#### Case ARC-2: adapter模式清理arch特定旧API

[Commit]: MAIN:57addf00 (MatmulAllReduce A5 Use Mc2InitTiling & Mc2CcTiling)
[类别]: ARCH_ADAPT
[涉及层]: op_host(tiling) + op_kernel
[变更规模]: 3 files, +2/-24 lines (净减少22行)

[场景]:
MatmulAllReduce的arch35(A5)量化通信内核(quant_comm_int8, quant_pertoken_comm_int8)中，
初始化和通信配置使用了旧API:
- `hccl_.Init(GetHcclContext<0>())` — 旧的Init接口(arch32使用)
- `hccl_.SetReduceDataTypeAbility(...)` — 手动设置每次通信的数据类型能力

新的Mc2InitTiling + Mc2CcTiling接口(arch35标准)已经在InitV2+SetCcTilingV2中覆盖了这些配置。
旧API调用是冗余的dead code——配置已由tiling数据在InitV2阶段设置。

[约束]:
1. arch35的通信子系统(CCU)要求使用新API(InitV2/SetCcTilingV2)，旧API是兼容存根(no-op)
2. 旧API在每次ReduceScatter/AllGather前都要调用(循环内6处)，增加了代码噪音
3. 删除旧API调用不能改变通信行为

[决策]:
1. 删除kernel中冗余的`hccl_.Init(GetHcclContext<0>())` (1行，因为InitV2已经调用过)
2. 删除所有`hccl_.SetReduceDataTypeAbility(...)` 调用(6处，每处3行=18行)
3. 修复tiling文件中的一个语法小错误: `Mc2CcTilingConfig(...);\ ` -> `Mc2CcTilingConfig(...);` (行尾多余反斜杠)
4. 总计删除24行，增加2行(修正)

变更文件:
- quant_matmul_all_reduce_tiling_950.cpp: 修复行尾反斜杠
- matmul_all_reduce_quant_comm_int8.h: 删除Init + 4处SetReduceDataTypeAbility
- matmul_all_reduce_quant_pertoken_comm_int8.h: 删除4处SetReduceDataTypeAbility

[替代方案]:
- 保留旧API兼容层: 安全但代码冗余，维护者不知道哪些调用有实际效果
- 用#ifdef区分arch32和arch35调用: 增加条件编译复杂度

[后果]:
成功合入。commit描述标记为"旧代码清理"(linked issue #746)。这是arch35成熟化过程中的典型操作——新API稳定后清理旧API残留。

[可迁移经验]:
arch适配不仅是"加代码"(支持新arch)，还要"删代码"(清理旧arch兼容层)。当新arch引入了更高级的API(如Mc2CcTiling替代SetReduceDataTypeAbility)，必须及时清理旧API调用。判断标准: 如果新API的Init阶段已经覆盖了旧API的功能，旧API就是dead code。及时清理的好处: (1)代码可读性提升(循环内少6处冗余调用), (2)避免未来旧API行为变化引入bug。

---

#### Case ARC-3: CCU硬件约束建模(AlltoAll 256MB数据量上限)

[Commit]: MAIN:84ef7bb3 (MatmulAllReduce CommFp8 Adapt CCU All2All DataCnt Limit)
[类别]: ARCH_ADAPT
[涉及层]: op_host(tiling)
[变更规模]: 2 files, +30/-0 lines (纯新增)

[场景]:
arch35使用CCU(Communication Control Unit)替代AICPU执行通信。CCU对AlltoAll/AlltoAllv有单次最大数据传输量不超过256MB的硬件限制。
在CommFp8(通信FP8量化)模式下，如果tiling切出的单块数据量(tileM * rankN)超过此限制，CCU会卡死(hang)。

commit描述明确记录:
> CCU对于AlltoAll和AlltoAllv有单次最大数据传输量不高于256Mb的限制，需要在tiling侧做出修改进行规避，否则单个块数据量过大会导致卡死

[约束]:
1. CCU硬限制: 单次AlltoAll数据量 <= 256MB
2. 实际安全值定义为`CCU_ALLTOALL_MAX_DATACNT = 200 * 1024 * 1024` (约200MB，留约22%安全裕量)
3. 可用量 = CCU_ALLTOALL_MAX_DATACNT * rankSize (因为AlltoAll会分rankSize份)
4. 只影响CommFp8模式(isInputCommQuantScale == QUANT_MODE_FP8)
5. 必须在tiling阶段做约束(CCU hang时无错误码，无法在kernel运行时处理)

[决策]:
在DoOpTiling()的标准tiling流程(DoRCSTiling + DoSplitMTiling)之后，插入DoCommFp8ReTiling()做二次修正:
1. 条件守卫: 仅CommFp8模式(isInputCommQuantScale == 2)
2. 计算maxDataCnt = CCU_ALLTOALL_MAX_DATACNT * rankSize
3. 如果tileM * rankN > maxDataCnt 或 tailM * rankN > maxDataCnt:
   - 新tailM = maxDataCnt / rankN (确保每块不超限)
   - tailCnt = rankM / tailM (总块数)
   - tileMValue = rankM % tailM (余数块)
   - 处理整除(tileMValue==0)和非整除两种情况
4. 日志: "TileCnt Enter CommFp8. tileM=%u, tailM=%u, tileCnt=%u, tailCnt=%u"

[替代方案]:
- 在kernel层做分片(一次大AlltoAll拆成多次小的): 增加kernel复杂度和同步开销
- 在Mc2CcTiling初始化时自动限制: 需修改通信框架层(超出算子开发者职责)
- 全模式检查(不限CommFp8): 保守但可能影响非FP8模式性能(不必要地切小块)

[后果]:
成功合入。CCU_ALLTOALL_MAX_DATACNT作为硬编码常量(200MB)定义在文件顶部。
未来如果CCU版本升级改变限制，需手动更新此常量。

[可迁移经验]:
硬件约束必须在tiling层(host侧)建模为显式常量和检查逻辑，不能依赖kernel运行时处理。CCU等硬件单元hang时无错误码，只有tiling层预防才有效。建模惯用法: (1)定义命名常量(CCU_ALLTOALL_MAX_DATACNT)而非内联magic number, (2)只在受影响的模式下触发(条件守卫), (3)tiling的"二次修正"模式(先做标准tiling再做硬件约束re-tiling)是处理跨层硬件限制的惯用法, (4)添加调试日志记录re-tiling参数。

---

#### Case ARC-4: 跨文件arch映射同步(3101 -> 3510遗漏修复)

[Commit]: MAIN:4da9f907 (alltoallmatmul算子A5npuarch芯片号更改)
[类别]: ARCH_ADAPT
[涉及层]: op_kernel
[变更规模]: 2 files, +3/-3 lines

[场景]:
allto_all_matmul算子的两个文件中，`__NPU_ARCH__`比较值写的是3101而非正确的3510:
- allto_all_matmul_apt.cpp: 2处 `__NPU_ARCH__ == 3101`
- mc2_templates/scheduler/pipeline_builder.h: 1处 `__NPU_ARCH__ == 3101`

3101不是有效的NPU_ARCH编号(没有任何芯片对应)，导致arch35的条件编译分支永远不生效。

时间线分析:
- allto_all_matmul首次添加: 2026-01-29 (MAIN:d79d1792)
- ARC-1大规模宏迁移: 2026-03-04 (MAIN:025e00f7)
- 本次修复: 2026-03-04 (MAIN:4da9f907, 同日稍晚)

这说明allto_all_matmul在创建时就写了错误的3101(可能从模板中抄错)，而ARC-1的全局替换是搜索`__DAV_C310__`，不会命中已经使用`__NPU_ARCH__`但值写错的文件。

[约束]:
1. 3处修改必须同步(2个文件)
2. 3101不是有效arch编号，但编译器不会报错(只是条件永远为false)
3. 这是一个"静默失败": arch35分支(量化支持、pipeline量化stage)被跳过，可能回退到非arch35的通用路径

[决策]:
精确修复3处`3101` -> `3510`。

[替代方案]:
无。这是一个纯粹的typo修复。

[后果]:
修复了"静默失败"bug。在修复前，allto_all_matmul在arch35上运行时会跳过arch35专用的优化代码路径(量化stage、专用kernel)，可能使用了不正确的通用路径或编译出不完整的kernel。

[可迁移经验]:
arch编号(3510/3201等)是易错的magic number，且错误不会编译失败(条件永远为false的#if不报错)。应对策略:
(1) 新增算子时从已有算子复制arch条件编译，不要手打数值
(2) 大规模arch迁移后做全仓grep验证: 不仅搜旧宏(`__DAV_C310__`)，还要搜新宏的异常值(`__NPU_ARCH__.*3101`)
(3) 理想方案: 用命名常量(NPU_ARCH_A5=3510)替代数值字面量，但当前代码库尚未这样做
(4) 这个case证明: 大规模迁移后新增的文件可能引入新的不一致——arch迁移不是"做一次就结束"的事

---

### 9.3.5 ARCH_ADAPT综合分析: 四种arch适配模式

从4个ARCH_ADAPT案例中提炼出四种适配模式及其适用场景:

#### 模式1: 大规模宏迁移(ARC-1)
适用场景: 宏/标识符命名规范变更(如__DAV_C310__ -> __NPU_ARCH__)
执行策略: 一次性全仓search-and-replace，60+文件同批提交
风险控制: 提交后全仓grep验证旧标识符清零 + grep新标识符异常值
教训: 无法覆盖"已使用新标识符但值写错"的情况(ARC-4)

#### 模式2: 旧API清理(ARC-2)
适用场景: 新arch引入了更高级的API，旧API成为dead code
执行策略: 删除冗余调用(Init -> InitV2已覆盖, SetReduceDataTypeAbility -> CcTiling已覆盖)
判断标准: 如果新API的初始化阶段覆盖了旧API功能，旧调用就是dead code
净效果: 代码减少(+2/-24)

#### 模式3: 硬件约束建模(ARC-3)
适用场景: 新arch的硬件单元有新的容量/性能约束(如CCU AlltoAll 256MB限制)
执行策略: 在tiling层添加re-tiling逻辑，用命名常量+条件守卫
关键原则: 硬件hang场景无法在kernel层处理，必须在tiling层预防
惯用法: DoXxxReTiling()做二次修正

#### 模式4: 跨文件映射同步(ARC-4)
适用场景: arch编号/chipset ID等magic number在多个文件中需要一致
执行策略: 精确修复 + 全仓grep验证
根因: 手动输入magic number容易出错
长期方案: 用命名常量(NPU_ARCH_A5 = 3510)替代数值字面量

#### arch适配的决策清单(按时间顺序):
1. 新arch支持初期: 添加条件编译分支(`#if __NPU_ARCH__ == XXXX`)，在新分支中使用新API
2. 新arch稳定后: 清理旧API残留(ARC-2模式)，减少代码冗余
3. 发现硬件约束时: 在tiling层建模约束(ARC-3模式)，用re-tiling防止hang
4. 命名规范变更时: 大规模迁移(ARC-1模式) + 全仓验证
5. 持续维护: 新增文件时从已有算子复制arch条件编译，不手打magic number

---

### 9.3.6: REFACTOR/REVERT Cases (8)

---

#### Case REF-1: 跨算子公共代码去重(matmul_all_reduce clean code)

[Commit]: MAIN:e8d52ffb ([mc2]matmul_all_reduce clean code)
[类别]: REFACTOR
[涉及层]: op_graph(公共gen_task) / op_kernel / op_api
[变更规模]: 14 files, +150/-916 lines (净减766行)

[场景]:
matmul_all_reduce算子和mc2/common/公共文件之间存在大量重复代码。mc2_gen_task_ops_utils_arch35.cpp和其stub文件各有约120行完全相同的代码(常量定义、GroupInfo结构体、GetGroupInfo/GetCommAlg/Mc2Arch35GenTaskCallBack三个函数)——因为正式版和stub版需要保持同步，任何修改都要改两处。

[约束]:
1. mc2_gen_task_ops_utils_arch35有正式版(.cpp)和stub版(_stub.cpp)两个文件，内容必须同步
2. 头文件(.h)被多个算子的op_graph层include
3. kernel层有一个已不使用的pertile_comm_fp8.h(428行)需要清理
4. op_api层有工具函数从.cpp移到.h(便于inline)

[决策]:
核心手法是"从.cpp搬到.h": 将GetCommAlg()、Mc2Arch35GenTaskCallBack()、GetGroupInfo()三个函数从mc2_gen_task_ops_utils_arch35.cpp移入mc2_gen_task_ops_utils_arch35.h中作为inline类方法。同时将常量定义(MOE_DISTRIBUTE_DISPATCH_OP_TYPE等13个字符串 + GroupInfo结构体 + GROUP_INFO_MAP_ARCH35映射表)一并搬入头文件。

效果:
- .cpp和_stub.cpp各删约110行，只留CreateCCUFusionTask()实现
- 彻底消除正式版/stub版的同步维护负担
- 删除kernel层无用文件matmul_all_reduce_quant_pertile_comm_fp8.h(428行)
- op_api的matmul_all_reduce_util工具函数从.cpp移至.h

[替代方案]:
- 保持.cpp/.h分离，用宏或模板减少重复: 增加复杂度，且stub版仍需同步
- 只做stub版删除: 不解决根本的代码重复问题

[后果]:
成功合入。Phase 8验证时确认此commit符合conventions中"公共代码应定义在mc2/common/"的规范。净减766行是MC2历史上最大的单次cleancode。

[可迁移经验]:
当.cpp和_stub.cpp之间存在大量重复时，将共享逻辑搬入.h作为inline函数是标准做法。判断标准: 如果一个函数在正式版和stub版中实现完全相同，它就应该在头文件中定义。清理时同步检查: (1)是否有已废弃的kernel头文件可删除, (2)是否有工具函数可从.cpp移到.h提高可见性。

---

#### Case REF-2: tiling巨型函数拆分(combineV2 tiling big function split)

[Commit]: MAIN:916aa93c (combineV2 tiling big function split)
[类别]: REFACTOR
[涉及层]: op_host(tiling)
[变更规模]: 1 file, +477/-416 lines (净增61行，因为新增了函数签名和调用代码)

[场景]:
moe_distribute_combine_v2_tiling.cpp中的GetAttrAndSetTilingData()函数是一个约350行的巨型函数，负责从TilingContext中读取全部属性、做参数校验、设置TilingData。该函数违反了单一职责原则: 既处理expert参数、又处理shared expert参数、又处理TP/EP通信参数，认知负荷极高。

[约束]:
1. 拆分不能改变功能行为(纯重构)
2. 各子函数间存在数据依赖(isLayered由commAlg决定，后续校验依赖isLayered)
3. config结构体包含所有属性的index，需要传递给子函数
4. 多个属性校验有交叉关系(例如epWorldSize同时被TP校验和shared expert校验使用)

[决策]:
按"参数域"拆分为3个静态函数:
1. GetExpertsAttrAndSetTilingData(): 处理moeExpertNum/zeroExpertNum/copyExpertNum/constExpertNum/expertShardType
2. GetSharedAttrAndSetTilingData(): 处理sharedExpertNum/sharedExpertRankNum
3. GetTpAndEpAttrAndSetTilingData(): 处理epWorldSize/tpWorldSize/epRankId/tpRankId/groupTp

原函数GetAttrAndSetTilingData()缩减为: 读取公共属性(groupEp/commAlg/commQuantMode/isLayered) → 调用3个子函数 → 返回。

拆分保持了以下原则:
- 每个子函数只负责一组语义相关的属性
- 空指针检查(OP_TILING_CHECK)保留在各子函数内部，不集中到外层
- 子函数返回ge::graphStatus，调用方用OP_TILING_CHECK包装错误传播

[替代方案]:
- 按"操作类型"拆分(读取/校验/设置): 会打断同一属性的处理连续性
- 用类方法替代静态函数: 需要将整个tiling逻辑改为面向对象，变更范围过大
- 引入Builder模式: 过度设计

[后果]:
成功合入。Phase 8验证时确认此commit正是conventions中"tiling函数超过~300行时按参数域拆分"规则的实证。

[可迁移经验]:
tiling巨型函数的拆分信号: 函数超过300行、处理3+组语义无关的属性、需要滚动多屏才能看完。拆分策略: 按"参数域"而非"操作类型"拆分——把同一组属性的读取+校验+设置放在同一个子函数中，因为属性间的校验关系(如epWorldSize与epRankId的范围关系)属于同一认知单元。数据依赖通过参数传递解决(isLayered/commQuantMode等作为子函数入参)。

---

#### Case REF-3: 全仓__aicore__宏清理(编译器内部已定义)

[Commit]: MAIN:1bdd7212 ([aicore]整改)
[类别]: REFACTOR
[涉及层]: op_kernel(主要是tests/ut/)
[变更规模]: 26 files, +3/-55 lines (净减52行)

[场景]:
编译器内部已经定义了`__aicore__`宏，算子侧的`#define __aicore__`是冗余的。这些冗余定义主要出现在两种场景:
1. UT测试头文件: `#ifdef __CCE_KT_TEST__` + `#define __aicore__`(11个文件)
2. kernel实现头文件: 同样的`#ifdef __CCE_KT_TEST__` + `#define __aicore__`(2个文件)

commit描述明确说明: "编译器内部已经定义了`__aicore__`，算子侧需要去掉这个标识"。

[约束]:
1. 跨3个模块(attention/gmm/mc2/moe)，26个文件
2. 每个文件的修改都是2-5行的`#define __aicore__`删除
3. 不能影响编译和UT运行(编译器已保证`__aicore__`存在)

[决策]:
简单删除所有手动`#define __aicore__`。变更极其机械化: 每个文件删除2-5行条件宏定义，无逻辑变更。

[替代方案]:
无。这是编译器基础设施变更后的必要清理。

[后果]:
成功合入。消除了26个文件中的冗余宏定义。这是INFRA驱动REFACTOR的典型案例: 编译器(下游)更新了能力，算子侧(上游)需要同步清理。

[可迁移经验]:
当编译器/基础设施新增了内建定义(如`__aicore__`、`__CCE_AICORE__`)，算子侧的手动`#define`就成了潜在冲突源(重复定义warning或行为不一致)。清理策略: (1)全仓grep定位所有手动定义, (2)一次性删除所有instance, (3)跑全量UT验证无回归。注意: 这类清理的风险在于"以为编译器已定义但实际上某个编译路径没定义"——因此全量UT覆盖是必要的。

---

#### Case REF-4: V1 Host侧代码重组(D&C V1 cleancode)

[Commit]: MAIN:f56a2f32 (D&C V1 Host侧cleancode)
[类别]: REFACTOR
[涉及层]: op_graph(fallback) / op_host(infershape + tiling) / op_kernel
[变更规模]: 6 files, +948/-625 lines (净增323行，因为结构化重组引入了新结构体和函数)

[场景]:
moe_distribute_combine V1(最早的MoE Combine算子)的Host侧代码是"初始版本"风格: fallback中所有输入用裸指针index访问，infershape中参数校验散落在巨型函数中，tiling中缺少结构化的参数聚合。

[约束]:
1. V1仍在使用(非废弃)，不能改变功能
2. 重组只涉及Host侧(fallback/infershape/tiling)，不改kernel
3. op_kernel仅删除了一个未使用的头文件(.h中删6行声明)

[决策]:
三层并行重构:

1. op_graph/fallback: 引入结构体替代裸指针
   - 新增OpInput/OpOptionalInput/OpOutput/OpAttr 4个匿名namespace内结构体
   - 原来散落的`context->GetInput(0)`/`context->GetInput(1)`改为结构化访问
   - 变量命名从camelCase统一为snake_case
   - 从单一巨型函数拆分为结构化流程

2. op_host/infershape: 拆分巨型校验函数
   - 原来约200行的flat校验函数按参数域拆分
   - 格式化改善(缩进对齐、注释规范化)

3. op_host/tiling: 参数分组改善
   - 与REF-2(combineV2 tiling split)相似的思路
   - 将散落的校验逻辑按语义聚合

[替代方案]:
- 仅做格式化不做结构重组: 不解决认知负荷问题
- 直接废弃V1: V1仍有使用者，不可行

[后果]:
成功合入。虽然净增323行(因为引入了结构体定义和函数签名)，但代码可读性和可维护性大幅提升。

[可迁移经验]:
V1(早期版本)代码的cleancode模式: (1)引入结构体替代magic index裸指针访问(OpInput/OpOutput/OpAttr), (2)fallback层的结构化是关键——它是外部框架调用的入口，可读性直接影响调试效率, (3)净增行数不等于代码劣化——结构体定义和函数签名的引入是"好的增长"。判断标准: 如果一个函数内有5+个`context->GetInput(N)`用裸数字索引，就需要引入结构体。

---

#### Case REF-5: 跨四层API参数删除(quantgmmalltoallv para del offset)

[Commit]: MAIN:939b8c44 (quantgmmalltoallv para del offset)
[类别]: REFACTOR
[涉及层]: op_api / op_graph(proto) / op_host(def)
[变更规模]: 11 files, +79/-409 lines (净减330行)

[场景]:
新增的quantgmmalltoallv(量化GMM+AlltoAllv融合算子)在初始开发时预留了非对称量化的offset参数(gmmXOffset/gmmWeightOffset/mmXOffset/mmWeightOffset共4个)。代码评审(review)后判定: 这些offset参数暂不支持且无需求，应该删除以简化API。

commit描述明确标注: "新增的算子quantgmmalltoallv，评审后，删除未使用的非对称量化的offset参数"，关联Issue 1122。

[约束]:
1. 参数删除必须同步修改四层: op_api(外部接口签名) / op_graph(REG_OP原型) / op_host(def.cpp输入定义) / op_api(校验函数)
2. 非量化版本的aclnnGroupedMatMulAlltoAllv也有对应的Inner接口声明需同步修改
3. 删除的不仅是参数声明，还有: 校验函数(CheckNotSupportNull全函数删除)、文档注释、Input()定义
4. 量化版本的proto同时做了dtype扩展(DT_HIFLOAT8新增 + 量化属性新增)——这不是纯删除

[决策]:
四层同步修改:

1. op_api(2文件):
   - aclnn_grouped_mat_mul_allto_allv.cpp: Inner接口声明删4个offset参数，调用处对应删除
   - aclnn_quant_grouped_mat_mul_allto_allv.cpp: 公开接口签名删4个offset参数 + 删CheckNotSupportNull()整个函数(23行) + CheckParams()签名和调用删4个参数

2. op_graph(1文件proto.h):
   - REG_OP(GroupedMatMulAlltoAllv)删除offset相关OPTIONAL_INPUT
   - 同时新增: gmm_x_scale/gmm_weight_scale/mm_x_scale/mm_weight_scale/comm_quant_scale 5个量化输入 + 7个量化属性
   - dtype扩展: DT_HIFLOAT8新增

3. op_host(1文件def.cpp):
   - Input名称从"gmmXScaleOptional"/"gmmWeightScaleOptional"改为"gmm_x_scale"/"gmm_weight_scale"(匹配proto)
   - 删除gmmXOffsetOptional/gmmWeightOffsetOptional/mmXOffsetOptional/mmWeightOffsetOptional 4个Input定义块

[替代方案]:
- 保留offset参数但标记deprecated: 增加API复杂度，调用者困惑
- 只删op_api不改proto: 四层不一致，编译会报错
- 等功能需要时再加: 评审结论就是当前不需要，先删

[后果]:
成功合入。净减330行。此commit展示了一个重要模式: API参数删除的级联效应——删4个参数需要改11个文件，因为每个参数在四层各有对应的声明、校验、传递代码。

[可迁移经验]:
API参数增删是MC2中成本最高的重构操作之一——4个offset参数的删除需要同步修改11个文件330行代码。这解释了为什么MC2倾向于先保守(预留参数)再清理(评审后删除)。操作清单: (1)proto.h删除REG_OP中的OPTIONAL_INPUT, (2)def.cpp删除对应Input()定义, (3)op_api删除接口签名参数+校验函数+调用传参, (4)文档注释同步删除。漏改任何一层都会导致编译失败或运行时崩溃。注意: 此commit同时做了"删offset + 加量化属性"两件事，这不是最佳实践——理想情况应拆为两个独立commit。

---

#### Case REV-1: 同步优化revert(4小时内回退)

[Commit]: MAIN:8ade8c3e (Revert "dispatch优化syncall")
[类别]: REVERT
[涉及层]: op_kernel
[变更规模]: 2 files (revert of MAIN:81ef4922)

[场景]:
这是OPT-3的revert。原始commit(81ef4922)试图将dispatch_v2的SyncAll()替换为细粒度同步(HardEvent和ReduceSum)，仅4小时后就被revert。

[约束]:
1. revert必须完全恢复原始状态
2. 同一开发者在同日的OPT-2(ea8f90e1)对combine算子做了类似优化且成功

[决策]:
完全revert。从diff可以看到原始优化的三个关键问题:

1. CalTokenSendExpertCnt()中删除了S_V同步: 原代码有`SyncFunc<AscendC::HardEvent::S_V>()`，优化版删除了它。但Duplicate()是Vector操作，后续的Sub/Compare也是Vector操作——中间的S_V是确保Scalar→Vector流水线同步的关键点。

2. BufferInit()中新增的statusBuf_和tokenNumBuf_: 使用了近似对齐(`Ceil(Ceil(totalExpertNum_, aivNum_), 8) * 8`)而非精确维度。对比OPT-2的成功做法是精确按`moeExpertNumPerRank * epWorldSize * UB_ALIGN`分配。

3. LocalWindowCopy()末尾S_MTE3→MTE3_MTE2替换: 原代码`SyncFunc<AscendC::HardEvent::S_MTE3>()`被删除，替换后的`SyncFunc<AscendC::HardEvent::MTE3_MTE2>()`在AllGather路径中顺序不对。

4. SetExpertTokenNums()完全重写为基于ReduceSum的聚合: 但workspace复用了gatherMaskOutBuf_而非独立分配——与OPT-2的独立workLocalBuf_形成对比。

revert恢复了原始的UpdateTokenNumsOut()逻辑: 最后一个核(lastCore_)直接从sendCountsGlobal GM读取cumsum值，不使用ReduceSum。

[替代方案]:
- 修复而非revert: 风险太高，同步bug可能导致数据损坏且难以复现
- 部分revert(只回退有问题的部分): 增加合并冲突风险

[后果]:
4小时内快速revert。原始优化的思路后来被OPT-2(combine算子)以正确方式实现——关键差异是buffer独立分配和精确尺寸计算。dispatch_v2的syncall优化至今未重新提交，说明dispatch的执行路径比combine更复杂，同步优化的正确性验证更困难。

[可迁移经验]:
SyncAll()优化是MC2中风险最高的操作。成功和失败的分水岭在于: (1)ReduceSum workspace是否独立分配(不能复用流水线中其他buffer), (2)buffer尺寸是否精确建模(不能近似对齐), (3)是否保留了所有必要的流水线同步点(S_V/MTE3_MTE2等)。如果4小时内发现问题，果断revert而非尝试热修复。

---

#### Case REV-2: 大规模性能优化回退(mm_rs优化回退)

[Commit]: DEV:f99a34dd (回退mm_rs优化)
[类别]: REVERT
[涉及层]: op_host(tiling) / op_kernel
[变更规模]: 13 files, +397/-3549 lines (净减3152行)

[场景]:
matmul_reduce_scatter_v2(MRS V2)曾做了一次大规模性能优化，核心是引入SmallM模式——为小M(矩阵行数较小)场景做专门的tiling策略。这次优化引入了:
1. 一个基于决策树的SmallM tiling选择器(matmul_reduce_scatter_v2_aiv_mode_smallm_tiling.h, 1869行)——使用DecisionNode结构体编码的二叉决策树，按m/k/n/rankSize等特征选择m0/swizzlCount/commDataSplit等tiling参数
2. 配套的SmallM kernel实现(matmul_reduce_scatter_aiv_mode_smallM.h, 652行 + matmul_smallM.hpp, 377行)
3. tiling的决策树针对不同rankSize(2/4/8)和不同SoC(A2/A3)各有独立的规则集

[约束]:
1. 回退标记为"Bug修复"，说明SmallM模式引入了正确性问题
2. 涉及13个文件，需要完整回退(不能部分保留)
3. 回退后tiling层保留了微小的日志格式修正(%d -> %u)

[决策]:
完全回退SmallM优化。删除了:
- tiling层: 整个smallm_tiling.h(1869行的决策树)
- kernel层: 整个SmallM模式kernel(matmul_reduce_scatter_aiv_mode_smallM.h + matmul_smallM.hpp，合计约1029行)
- CMakeLists: 删除smallm_tiling.h的编译引用
- tiling_key: 删除SmallM相关分支

决策树是自动生成的(规则格式高度规律化，每个rankSize/SoC组合约200-300行规则)，说明这是一个数据驱动的autotuning方案。

[替代方案]:
- 修复SmallM逻辑: 如果决策树选择了错误的参数组合导致计算错误，修复需要重新训练/验证决策树——成本过高
- 仅禁用SmallM路径: 保留代码但不走，增加无用代码量
- 渐进修复: 先回退再分批重新引入——但最终选择了完全回退

[后果]:
回退了3549行代码(SmallM优化的全部内容)。这是MC2历史上最大的单次回退。SmallM优化的思路(基于决策树的自动tiling选择)未在后续commit中重新出现，说明:
1. 决策树方法可能不够鲁棒(训练数据覆盖不全的shape组合导致错误tiling)
2. 或者该特性在dev仓实验后被放弃(dev仓的实验性质)

[可迁移经验]:
大规模性能优化(1000+行的新路径)的风险管理: (1)决策树式的自动tiling选择器虽然看起来优雅，但每个叶节点的参数组合都需要独立验证——组合爆炸使得测试覆盖困难, (2)SmallM这样的特殊shape路径应该有明确的enable/disable开关(fallback到通用路径)而非硬切, (3)回退代价与引入代价成正比——3549行引入意味着3549行回退，优化应当分阶段合入(先基础框架再具体规则)以降低回退粒度。

---

#### Case REV-3: TilingKey整改批量失败(5次连续revert)

[Commit]: DEV仓5个连续revert:
- DEV:2001261f (Revert "MatmulAllreduce tilingKey整改【A2+310P】") — 23 files, +7467/-2250
- DEV:8ead68cc (Revert "mc2 tilingkey rectification for A2/A3 combine/dispatch") — 12 files, +337/-798
- DEV:20f4d541 (Revert "mc2 tilingkey rectification MatmulReduceScatter V2") — 3 files, +12/-54
- DEV:58a7ef18 (Revert "mc2 tilingkey rectification for A2/A3 and A5_isolation") — 20 files, +600/-1188
- DEV:a2b032dc (Revert "tilingkey rectification for reducescatterv2 and A5_isolation") — 5 files, +141/-201
[类别]: REVERT(批量)
[涉及层]: op_host(def + tiling) / op_kernel
[变更规模]: 累计63 files, 约+8557/-4491

[场景]:
MC2模块发起了一次"tilingkey整改"运动: 将多个算子的旧式十进制手工编码tilingkey迁移为APT模板化tilingkey。这是FEA-3(GMM tilingkey template化)的大规模推广。

原始commit(按时间顺序):
1. 3ec314f89: MatmulAllreduce tilingKey整改【A2+310P】
2. 5114e8562: mc2 tilingkey rectification for A2/A3(combine, dispatch)
3. f8f038c90: mc2 tilingkey rectification(MatmulReduceScatter V2)
4. bce10843f: mc2 tilingkey rectification for A2/A3 and A5_isolation
5. 25318dd92: tilingkey rectification for reducescatterv2 and A5_isolation

5个commit在2025-11-28~12-03期间全部被revert。

[约束]:
1. TilingKey是跨host-kernel的接口: tiling层编码(SET_TPL_TILING_KEY) → kernel层解码(GET_TPL_TILING_KEY分支)
2. 旧式手工编码: 12+个TILING_KEY_IS(xxx)分支，每个分支对应一种参数组合
3. APT模板化: 用ASCENDC_TPL_ARGS_DECL定义编译期模板参数，kernel通过if constexpr分支
4. MatmulAllreduce的整改最大: 23 files, 其中kernel入口文件(matmul_all_reduce.cpp)从APT模板缩回旧式992+行的TILING_KEY_IS分支，tiling_key.h(711行)被删除
5. MoE算子(combine/dispatch)的整改涉及4个算子8个入口(v1+v2各两个)

[决策]:
5个原始commit全部revert，恢复旧式手工编码。

从revert的diff可以看到整改方向:
- 旧式: kernel入口有12+个`TILING_KEY_IS(xxxx)`分支，每个分支展开不同的模板组合
- 新式: 引入tiling_key.h定义ASCENDC_TPL_ARGS_DECL，kernel入口缩减为2-3个if constexpr分支

revert恢复了旧式的以下特征:
- matmul_all_reduce.cpp回到992+行(12个TILING_KEY_IS分支)
- MoE combine/dispatch回到手工编码(63-290行不等的if-else链)
- 删除了所有新创建的_apt.cpp和_tiling_key.h文件

[替代方案]:
- 修复整改中的问题而非revert: 问题可能是编译期/运行时tiling key值不匹配——这是跨层接口问题，修复需要逐算子逐arch验证
- 部分保留(只revert有问题的算子): 但5个都出了问题，说明整改方法本身有系统性缺陷

[后果]:
这是MC2历史上最大的批量revert事件。5个原始commit的作者(caoqiku/jiangbo等)后来以更谨慎的方式逐个算子重新引入APT tilingkey——FEA-3(GMM算子)就是成功案例，它只改了1个算子9个文件。

失败根因分析:
1. batch apply风险: 一次性改5+个算子的tilingkey格式，任何一个算子出问题都需要全部revert
2. 跨层接口一致性: tilingkey值是host(编码)→kernel(解码)的隐式契约，批量修改后难以逐一验证每个算子的每种dtype/arch组合
3. 测试覆盖不足: dev仓的UT可能未覆盖所有TILING_KEY_IS分支的shape组合

[可迁移经验]:
跨算子的格式统一(如tilingkey从手工编码迁移到APT模板)是MC2中失败率最高的重构类型。原因: tilingkey是host→kernel的隐式跨层接口，每个算子有独特的分支组合，batch apply无法保证每个算子的正确性。正确做法: (1)逐个算子迁移(不超过1个算子/commit), (2)每个算子迁移后跑该算子的全量UT+ST, (3)先从最简单的算子开始(如all_gather_matmul只有3个bool分支), (4)MoE和MatmulAllReduce这样的复杂算子放最后。FEA-3(GMM的成功案例)正是"逐个算子+充分测试"策略的实证。

---

### 9.3.6 REFACTOR/REVERT综合分析: 五种重构模式和三种失败模式

#### 五种重构模式(按风险从低到高):

1. 宏/标识符清理(REF-3): 机械化删除，风险最低，编译验证即可
2. 公共代码去重(REF-1): .cpp搬.h，中等风险，需验证链接行为
3. 函数拆分(REF-2): 纯结构改善，中等风险，需验证功能等价
4. V1代码重组(REF-4): 引入新抽象(结构体)，中高风险，需全量UT
5. 跨四层API参数变更(REF-5): 级联改动11+文件，高风险，需四层联合验证

#### 三种REVERT失败模式(按影响从小到大):

1. 单点优化失败(REV-1): 1个算子1个优化，4小时内发现并revert，影响可控
2. 大规模特性失败(REV-2): 1个算子1个大特性(SmallM)，3549行回退，说明实验性特性应在dev仓充分验证
3. 批量迁移失败(REV-3): 5+个算子的格式统一，8500+行回退，这是系统性方法论失误——批量apply跨层接口变更必然失败

#### 重构的决策框架:

判断"是否适合batch重构":
- 变更是否局限在单层(如REF-3只删宏)? → 适合batch
- 变更是否涉及跨层接口(如REV-3改tilingkey)? → 必须逐个算子
- 变更是否有隐式契约(如tilingkey的host→kernel值映射)? → 必须逐个算子+全量验证
- 变更是否可编译验证(如REF-3删冗余define)? → 适合batch
- 变更是否需要运行时验证(如同步优化)? → 必须逐个场景验证

---

## 9.4: Scenario-Based Development Guide (场景化开发指南)

基于29个landmark case的决策点分析，重组为开发者面临具体任务时的决策清单。
每个场景包含: 决策清单(按顺序) + 每步引用相关case + 常见陷阱。

---

### 9.4.1 场景: "新增MC2算子"

适用: 从零创建一个全新的MC2融合算子(如AlltoAllMatmul、AttentionToFFN)。

引用案例: NOP-1(完整四层)、NOP-2(扁平独立)、NOP-3(批量骨架)、NOP-4(arch扩展)

#### 决策清单

Step 1: 确定算子类型和编程范式

- MatMul+Comm融合(如AlltoAllMatmul): 使用AIC(Cube)+AIV(通信)分工模型，考虑mc2_templates框架 [NOP-1]
- 纯数据路由(如AttentionToFFN): 仅AIV，RDMA窗口直驱，可用扁平实现 [NOP-2]
- 判断标准: 是否涉及MatMul? 是→继承TilingBaseClass+使用3rd/mat_mul_v3; 否→扁平tiling

Step 2: 确定arch支持范围

- 同时支持arch32+arch35: kernel双入口(.cpp旧版 + _apt.cpp新版)，tiling分arch文件 [NOP-1]
- 仅arch35: 简化，但要预留arch扩展空间 [NOP-2, NOP-3]
- 仅arch32+新增A3: REGISTER_TILING_TEMPLATE_WITH_SOCVERSION + socVersion字符串比较 [NOP-4]

Step 3: 创建目录骨架

- 一次commit创建完整四层: op_host/(def.cpp + infershape.cpp + tiling) + op_kernel/(各arch) + op_graph/(gen_task + fallback) + op_api/ [NOP-1]
- 多个相关算子: 一次性批量创建目录，用.gitkeep占位 [NOP-3]
- 命名: 小写蛇形(all_gather_matmul)，目录内文件与算子名一致

Step 4: 写op_host层(def + infershape + tiling)

- def.cpp: OP_ADD注册 + Input/Output声明 + 可选参数OPTIONAL_INPUT
  - HcclGroup: 单组("group") vs 双组({"group_ep","group_tp"})
  - AutoContiguous(主流) vs IgnoreContiguous(MatMul类需要)
  - jitCompile.flag: "static_false"(运行时shape) vs "static_true"(编译时shape)
- infershape.cpp: IMPL_OP_INFERSHAPE，输出shape推导
- tiling:
  - 复杂算子: 继承TilingBaseClass/MoeTilingBase，重写8步DoTiling [NOP-1]
  - 简单算子: 扁平单文件(AllGatherMatmul 634行示例) [NOP-2]
  - tiling_key.h: 定义ASCENDC_TPL_ARGS_DECL，将运行时参数编码为编译期模板参数
  - Mc2CcTilingConfig: 通信配置(100%覆盖率，强制) [Phase 3统计]

Step 5: 写op_kernel层

- REGISTER_TILING_DEFAULT: 绑定TilingData类型(强制) [Phase 4统计]
- AIC/AIV分工: ASCEND_IS_AIC执行MatMul，ASCEND_IS_AIV执行通信+搬运 [Phase 4模式]
- 通信初始化: GetHcclContext → HcclInit/HcclInitV2(arch35)
- 流水线: 手工编排 / mc2_templates框架 / 窗口自实现 -- 按算子特性选择
- 多arch kernel: .cpp(旧版arch32) + _apt.cpp(新版arch35) [NOP-1]

Step 6: 写op_graph层

- gen_task: IMPL_OP + CommonKFCMc2CalcParamFunc(标准化，100%覆盖) [Phase 5]
- fallback: EXEC_OPAPI_CMD宏包装dlopen调用(标准化) [Phase 2]
- arch分派: MatMul类按NpuArch，MoE类按SocVersion [Phase 5]

Step 7: 写op_api层

- 两阶段API: aclnnXxxGetWorkspaceSize() + aclnnXxx() [Phase 5]
- HcclServerType分派: arch35用CCU，arch32用AICPU/MTE [NOP-1, NOP-4]
- 输入校验: 在Execute入口做dtype/shape/rank校验

Step 8: 写tests + docs + examples

- op_host UT最优先(61个UT，覆盖率最高) [Phase 5]
- op_kernel UT通过#include .cpp包含源码 [Phase 5]
- 每个aclnn API对应一个.md文档

#### 常见陷阱

1. arch编号手打错误: __NPU_ARCH__ == 3101(应为3510)，编译不报错但静默跳过分支 [ARC-4]
2. tiling_key host→kernel不一致: SET_TPL_TILING_KEY的值与GET_TPL_TILING_KEY的分支不匹配 [REV-3]
3. CmakeLists.txt拼写错误: CMakeLists.txt中C和L必须大写 [NOP-3]
4. 公共代码先放局部: mc2_templates初期放算子目录下，稳定后再上浮到mc2/common [NOP-1]
5. 量化变体通过ORIG_DTYPE宏物理隔离，不要在同一文件内if-else [NOP-1]

---

### 9.4.2 场景: "给已有算子添加arch35支持"

适用: 已有arch32实现的算子需要适配Ascend950(arch35/CCU)。

引用案例: NOP-4(A3扩展)、ARC-1(宏迁移)、ARC-2(旧API清理)、ARC-3(硬件约束)、ARC-4(映射同步)

#### 决策清单

Step 1: 评估arch35与arch32的差异

arch35的5大变化(Phase 4总结):
- 通信控制: AICPU → CCU
- Tiling key: 手工十进制 → APT(ASCENDC_TPL_ARGS_DECL)
- MatMul Block: BaseBlock → ASW(Auto Split Work)
- Wait模式: 单次Wait → per-tile Wait
- 初始化: Init → InitV2 + SetCcTilingV2

Step 2: 改op_host

- tiling: 新增arch35 tiling文件(*_tiling_950.cpp 或 *_tiling_a5.cpp)
  - REGISTER_TILING_TEMPLATE_WITH_ARCH: arch35专用注册宏(13处) [Phase 3统计]
  - Mc2InitTiling + Mc2CcTiling: arch35标准通信配置 [ARC-2]
  - 硬件约束检查: CCU AlltoAll 256MB上限 → DoXxxReTiling()二次修正 [ARC-3]

Step 3: 改op_kernel

- 新增 _apt.cpp 入口文件: 使用ASCENDC_TPL_ARGS_DECL定义编译期参数
- 新增 arch35/ 目录: 放arch35专用的Base类和实现
- InitV2 + SetCcTilingV2: 替代旧Init [ARC-2]
- per-tile HcclWait: 在每个tile的通信完成后Wait(vs arch32的单次Wait) [Phase 4]

Step 4: 改op_api

- HcclServerType: 添加DAV_3510分支 → NnopbaseSetHcclServerType(CCU) [NOP-4]
- v1→v2自动重定向: 如果arch35不支持v1，在op_api层透传到v2 [Phase 5]

Step 5: 验证

- 全仓grep验证arch编号一致性: `grep -r '__NPU_ARCH__.*3101'` 捕捉typo [ARC-4]
- 清理旧API残留: 如果InitV2已覆盖旧Init，删除冗余调用 [ARC-2]

#### 常见陷阱

1. CCU hang无错误码: 超过256MB的AlltoAll数据量会导致CCU卡死，只能在tiling层预防 [ARC-3]
2. arch编号是magic number: 3510/3201等易错且编译不报错，建议从已有算子复制而非手打 [ARC-4]
3. 旧API是兼容存根: arch35的Init()是no-op，但不删除会增加代码噪音和误导 [ARC-2]
4. 宏迁移后验证: 不仅搜旧宏(__DAV_C310__)是否清零，还要搜新宏的异常值(__NPU_ARCH__==3101) [ARC-1→ARC-4]

---

### 9.4.3 场景: "修复通信同步bug"

适用: 遇到同步相关bug(数据损坏、死锁、大shape失败、OOM)的诊断和修复。

引用案例: BUG-1(常量错误)、BUG-3(三层联动)、BUG-4(sizeof混淆)、OPT-2/OPT-3(同步优化成功/失败)、REV-1(快速revert)

#### 诊断方法

Step 1: 确定bug层次

- 仅大shape触发 → 检查常量上限(如MAX_HCCL_HANDLE是否匹配HCCL实际限制) [BUG-1]
- 仅特定rank数触发 → 检查整数除法/取模/对齐的边界假设 [BUG-3]
- OOM → 检查buffer分配的字节/元素单位是否正确 [BUG-4]
- 数据损坏(偶发) → 检查SyncAll/HardEvent是否遗漏某个数据依赖 [OPT-3]

Step 2: 三层联动审查(从BUG-3学到的方法论)

同步bug往往在多层同时表现:
1. op_host/infershape: 检查括号优先级(`a * b / c` vs `a * (b / c)`)和整数截断
2. op_host/tiling: 检查blockDim条件分支(是否覆盖了所有合法rank数)
3. op_kernel: 检查对齐(RoundUp)、Mask位宽、DataCopyPad、同步事件匹配

Step 3: 参数边界测试矩阵

| 参数 | 测试值 | 原因 |
|------|--------|------|
| rank | 1, 2, 7, 8, 9, 16, 64 | 边界值和2的幂附近 [BUG-3] |
| shape M | 1, 小于tile, 等于tile, 大于tile | 对齐和余数 |
| dtype | fp16, bf16, fp8, int8 | sizeof差异 [BUG-4] |

#### 修复策略

1. 常量修复: 改值+附注spec出处(如"HCCL supports up to 63 parallel handles") [BUG-1]
2. 类型修复: 全仓Grep扫描同类错误(如GetAttrPointer<int>全改int64_t) [BUG-2]
3. 同步修复: 不要"修补"，要精确建模数据依赖并画出流水线图 [OPT-2 vs OPT-3]
4. 三层联动: 每层独立修复，但一次commit提交(避免中间态) [BUG-3]

#### 回归风险控制

- 修复同步bug后必须跑全量UT+多种rank配置 [BUG-3, OPT-3]
- 如果修复复杂度高(5+文件)，考虑先revert到安全状态再重做 [REV-1]
- 4小时内发现问题就果断revert，不要热修复 [REV-1]

#### 常见陷阱

1. 主流配置掩盖bug: rank=8时恰好不触发，非主流配置(rank<8)才暴露 [BUG-3]
2. little-endian掩盖类型bug: int64存小值时int32指针读到相同结果(仅LE平台) [BUG-2]
3. buffer复用导致数据污染: ReduceSum workspace不能复用流水线中其他buffer [OPT-3]
4. buffer近似对齐: 大小计算必须基于精确数据维度，不能用"大概够用"的近似值 [OPT-3]
5. sizeof(DataType枚举) != sizeof(实际数据): DT_FLOAT16的枚举值!=2字节 [Phase 8 rule 7.9]

---

### 9.4.4 场景: "添加量化支持"

适用: 给已有MC2算子添加量化模式(FP8/INT8/INT4/MX等)。

引用案例: FEA-2(全栈动态量化)、NOP-1(ORIG_DTYPE隔离)、REF-5(参数增删成本)

#### 决策清单

Step 1: 选择量化集成方式

两种模式(Phase 6总结):
- 内嵌变体(tiling key): 在tiling_key中增加QUANTMODE维度，kernel用if constexpr分支 [FEA-2]
  - 适用: 量化逻辑与主逻辑深度耦合(如量化发生在通信前后)
  - 优势: 共享主逻辑，维护一份代码
  - 劣势: tiling key组合爆炸
- 独立算子(quant_*): 创建新的quant_xxx算子目录 [Phase 6 quant对比]
  - 适用: 量化逻辑完全不同(如需要额外的Scale/Offset输入)
  - 优势: 互不干扰
  - 劣势: 代码重复

Step 2: 扩展tiling

- tiling_key新增量化维度: 在ASCENDC_TPL_ARGS_DECL中增加quantMode参数 [FEA-2]
- 量化参数传递: def.cpp中用OPTIONAL_INPUT声明Scale/Offset输入
- API参数增删的成本: 4个参数的删除需要11个文件330行 [REF-5]，所以初始API设计要谨慎

Step 3: 扩展kernel

- 量化Stage: 如果用mc2_templates，新增QuantizeStage + 扩展Pipeline [FEA-2]
- ORIG_DTYPE宏隔离: int4/int8与fp16/bf16的Cast语义冲突，必须用条件编译物理隔离 [NOP-1]
- 量化Scale/Offset的Buffer管理: 注意API参数是字节还是元素单位 [BUG-4]

Step 4: 验证

- 量化精度验证: 量化后的通信数据与非量化路径的结果比对
- dtype组合矩阵: 不要只测fp16→int8，还要测bf16→int8, fp8→int4等边界组合
- 参数空指针: 量化的Scale/Offset参数在非量化模式下应为nullptr，需要校验 [Phase 3分析]

#### 常见陷阱

1. 初始API预留过多参数后再删除代价很高(11文件/330行) [REF-5]
2. tiling key组合爆炸: 每增加一个bool维度，kernel分支翻倍
3. 量化常量与硬件spec不一致: 如CommFp8的256MB CCU限制 [ARC-3]
4. GetAttrPointer类型不匹配: 量化参数的int/int64_t混用是高频bug [BUG-2]

---

### 9.4.5 场景: "优化tiling性能"

适用: 优化MC2算子的workspace分配、同步机制、buffer管理等。

引用案例: OPT-1(workspace优化)、OPT-2(成功同步优化)、OPT-3(失败同步优化)、REV-2(SmallM回退)、REF-2(tiling拆分)

#### 优化安全性谱系(从安全到高风险)

1. workspace布局重组(安全): 改变"何时写GM"但不改变"写什么"
   - 本地UB累积 → 批量写出 → 向量ReduceSum聚合 [OPT-1]
   - 批量Cache刷新替代逐行刷新 [OPT-1]
   - buffer按需动态分配替代固定大小 [OPT-1]

2. tiling参数调优(中等风险): 改变切分策略
   - 验证方法: 对比优化前后的等效性(总数据量不变)
   - 陷阱: 决策树式自动tuning的覆盖性问题——未覆盖的shape组合可能选错参数 [REV-2]

3. SyncAll替代(高风险): 替换同步机制
   - 铁律1: buffer大小必须精确匹配数据搬运的实际维度(不能近似对齐) [OPT-2 vs OPT-3]
   - 铁律2: ReduceSum workspace必须独立分配(不能复用流水线中其他buffer) [OPT-2 vs OPT-3]
   - 铁律3: 每个HardEvent必须精确对应一个真实的数据依赖 [OPT-3失败点]

#### 决策清单

Step 1: 评估优化类型

- 仅改buffer布局/大小? → 安全，可以直接做
- 改写同步机制(SyncAll→HardEvent)? → 高风险，需要画数据依赖图
- 引入新的tiling策略(如SmallM)? → 中等风险，需要考虑覆盖性

Step 2: 如果是同步优化

1. 画出完整的数据依赖图: 哪个buffer被谁写、被谁读、在什么时序
2. 为每个HardEvent标注对应的数据依赖: S_V(Scalar→Vector), MTE3_MTE2(写出→读入)等
3. 为ReduceSum等向量操作分配独立workspace: `tpipe_->InitBuffer(workLocalBuf_, size)` [OPT-2]
4. buffer大小从精确维度计算: `moeExpertNumPerRank_ * epWorldSize_ * UB_ALIGN` [OPT-2]
5. 不要从近似值计算: `Ceil(Ceil(totalExpertNum_, aivNum_), 8) * 8` [OPT-3失败]

Step 3: 如果是新tiling策略

- 每个叶节点参数组合需独立验证(决策树的组合爆炸) [REV-2]
- 必须有fallback到通用路径的开关 [REV-2教训]
- 分阶段合入: 先基础框架 → 再具体规则，降低回退粒度 [REV-2教训]

Step 4: 验证

- 同步优化: 跑多种rank配置+多种shape+多种dtype
- tiling优化: 跑性能benchmark + 正确性验证
- 4小时内发现问题就revert，3天后修复再提交 [OPT-2 vs OPT-3的成功路径]

#### 常见陷阱

1. 同一开发者同一天可以提出成功和失败的同步优化 [OPT-2 vs OPT-3]——差异在实现细节不在思路
2. 3549行的SmallM优化被整体回退 [REV-2]——大规模优化应分阶段合入
3. tiling函数超过300行需拆分，但拆分本身也有风险(参数传递遗漏) [REF-2]
4. 优化后的代码比优化前更长是正常的(精确建模需要更多代码) [OPT-2]

---

## 9.5: Gray Area Judgment Guide (灰色地带指南)

conventions告诉你"做什么"(规则)，casebook告诉你"怎么做"(案例)，
灰色地带指南告诉你"怎么判断"(启发式)——在规则不明确时如何做出合理决策。

---

### 9.5.1 "何时拆分tiling函数"

从REF-2(combineV2 split)和整体tiling层分析中提炼。

#### 拆分信号(满足任意2个就应该拆分):

1. 行数超过300行(combineV2原函数约350行触发拆分) [REF-2]
2. 处理3组以上语义无关的参数(expert/shared/TP_EP) [REF-2]
3. 需要滚动3屏以上才能看完整个函数
4. 同一函数内有2+个SoC/arch分支路径(A2 vs A3A5) [Phase 3 moe_distribute_combine]

#### 拆分策略:

- 按"参数域"拆分(推荐): 同一组属性的读取+校验+设置放在一起 [REF-2]
  - GetExpertsAttrAndSetTilingData()
  - GetSharedAttrAndSetTilingData()
  - GetTpAndEpAttrAndSetTilingData()
- 不要按"操作类型"拆分(不推荐): 把所有读取放一起、所有校验放一起 — 打断属性的处理连续性

#### 不需要拆分的情况:

- AllGatherMatmul的tiling(634行单文件): 虽然长但逻辑线性、无分支、属性组少 [Phase 3]
- 单一arch的简单tiling: 如果只有一条执行路径，长≠复杂

#### 判断准则:

问自己: "如果要修改epWorldSize的校验逻辑，我需要阅读多少不相关的代码才能找到它?"
如果答案>100行，就该拆分。

---

### 9.5.2 "何时用继承 vs 扁平结构"

从matmul_all_reduce(深继承) vs all_gather_matmul(扁平) vs moe_distribute_combine(中间态)对比中提炼。

#### 三种tiling架构的适用场景:

1. 深继承(TilingBaseClass → Base → 10个具体类):
   - 适用: 5+个变体(量化/非量化/arch32/arch35/FP4等) [matmul_all_reduce]
   - 适用: 变体间共享大量公共逻辑(80%+代码相同)
   - 代价: 理解成本高，初始化链需要仔细管理(BUG-5的InitCompileInfo遗漏)

2. 扁平单文件(静态函数):
   - 适用: 变体少(3个bool参数) [all_gather_matmul]
   - 适用: 首次实现(快速迭代) [NOP-2]
   - 代价: 变体增多后函数膨胀

3. 中间态(MoeTilingBase单子类 + 内部分支):
   - 适用: arch分支需求明确但变体不多 [moe_distribute_combine]
   - 实际: 常是"扁平→继承"演进的中间阶段 [Phase 3演进方向]

#### 决策准则:

| 条件 | 推荐架构 |
|------|----------|
| 变体数 <= 3 | 扁平单文件 |
| 变体数 4-7，共享代码 > 50% | 中间态(Base + 2-3子类) |
| 变体数 >= 8，共享代码 > 70% | 深继承 |
| 新增算子首次实现 | 扁平(先跑通再重构) |
| 与MatMul+Comm范式不同的算子 | 扁平(不强行复用TilingBaseClass) [NOP-2] |

#### 演进方向:

MC2的明确趋势是"从扁平到继承": Phase 3的分析显示matmul_all_reduce从早期可能的扁平结构演进到了10子类继承，而较新的算子(如all_gather_matmul)仍保持扁平。这说明: 先扁平、后继承是安全的演进路径，但反过来(先继承后扁平)成本很高。

---

### 9.5.3 "何时复用公共代码 vs copy-paste"

从mc2/common/的演进和REF-1(去重)、NOP-1(局部创建mc2_templates)中提炼。

#### "先局部后公共"策略 [NOP-1的成功实践]:

1. 初始: 在算子局部目录创建(如matmul_allto_all/op_kernel/mc2_templates/)
2. 稳定: 第二个算子需要时，从局部目录复制到mc2/common/
3. 成熟: 3+个算子使用后，正式成为MC2公共基础设施

#### 判断"何时上浮到公共":

应该上浮:
- 2+个算子使用完全相同的代码(逐行一致) [REF-1: .cpp→.h去重的前提]
- 代码是"基础设施"性质(Stage/Pipeline/SyncFunc)而非"业务逻辑"
- 正式版(.cpp)和stub版(_stub.cpp)需要同步维护 — 必须上浮 [REF-1]

不应该上浮:
- 仅1个算子使用(上浮增加了不必要的间接层)
- 代码有算子特定的hardcode(如hidden_size=7168) [Phase 3反模式]
- 代码正在快速迭代(API不稳定时上浮会导致频繁的公共代码修改)

#### copy-paste可接受的场景:

- Mc2SyncAll/Mc2CoreType等3行以下的工具函数: 虽然3+处重复，但上浮到公共头文件需要解决include依赖 [Phase 4反模式]
- 算子级common.h: 仅3个算子有，说明大多数算子不需要算子级公共代码

#### 反模式:

- A2路径跨目录引用v2代码(#include "../moe_distribute_combine_v2/..."): 应提取到公共 [Phase 4反模式]
- CeilDiv在5+处定义且行为不一致(b==0返回值不同): 应统一 [Phase 2反模式]

---

### 9.5.4 "新功能放在哪一层"

从多层联动commit和Phase 5/6的四层分析中提炼。

#### 四层职责回顾:

- op_api: 参数校验 + 两阶段调用编排 + DFX打点 + HcclServerType分派
- op_graph: 图定义(REG_OP) + 任务生成(gen_task) + fallback路径
- op_host: Tiling计算(shape→切分参数) + InferShape(输出shape推导) + Op定义(属性声明)
- op_kernel: 设备执行(计算+通信流水线)

#### 职责归属判断:

| 新功能类型 | 放哪层 | 案例证据 |
|-----------|--------|----------|
| 新输入参数(Scale/Offset) | op_host(def.cpp) + op_graph(proto) + op_api(签名) | REF-5: 4个参数改11文件 |
| 新通信模式(FullMesh) | op_kernel(主要) + op_host(tiling_key新增维度) | FEA-4 |
| 新dtype/量化 | op_kernel + op_host(tiling_key) + op_api(校验) | FEA-2 |
| 新arch适配 | op_host(tiling) + op_kernel(arch目录) + op_api(ServerType) | NOP-4, ARC-1~4 |
| 硬件约束(CCU限制) | op_host(tiling层ReTiling) | ARC-3 |
| 输入校验/错误信息 | op_api(最外层) 或 op_host(tiling层) | BUG-1 |
| 调试/DFX | op_api(打点) + 环境变量(任意层) | NOP-2的ASCEND_ATTN_TO_FFN_WIN_TYPE |

#### 关键原则:

1. 硬件约束只能在tiling层处理: CCU hang无错误码，kernel层无法捕获 [ARC-3]
2. 跨层参数传递的代价很高: 增删一个API参数需要四层联动 [REF-5]
3. arch分派尽早做(op_api或op_graph): 不要在op_kernel中用if constexpr区分arch [Phase 5分析]
4. 新功能影响几层就改几层，不要"以防万一"多改 [REF-5的教训: 预留参数后要删除也很贵]

---

### 9.5.5 综合: 灰色地带判断框架

从29个案例中提炼出3条元准则，适用于conventions无法覆盖的所有灰色地带:

#### 元准则1: 可逆性优先

评估一个决策时，先问"如果做错了，回退成本是多少?":

| 回退成本 | 决策策略 | 案例 |
|---------|---------|------|
| 低(1-3文件) | 大胆尝试 | ARC-2(删旧API，净减22行) |
| 中(5-15文件) | 需要review | FEA-1(新增通信路径，13文件) |
| 高(20+文件) | 分阶段合入 | NOP-1(107文件，但一次创建是必要的) |
| 极高(跨层接口) | 逐个算子验证 | REV-3(tilingkey整改63文件全部revert) |

推论: batch apply适合单层内部变更，不适合跨层接口变更。

#### 元准则2: 精确建模优于近似猜测

在buffer大小、常量值、同步事件、参数类型等"精确性"问题上，永远选择从源头推导的精确值:

| 场景 | 精确做法 | 近似做法(反模式) |
|------|---------|-----------------|
| buffer大小 | moeExpertNumPerRank * epWorldSize * UB_ALIGN [OPT-2] | Ceil(totalExpertNum/aivNum, 8)*8 [OPT-3] |
| 硬件常量 | 查HCCL spec → 63 [BUG-1修复后] | 凭经验猜测 → 16 [BUG-1原始] |
| 类型参数 | 与op_def交叉核对 → int64_t [BUG-2修复后] | copy-paste → int [BUG-2原始] |
| 单位 | 显式标注Bytes/Elements → count/sizeof [BUG-4修复后] | 混用 → 字节当元素 [BUG-4原始] |

#### 元准则3: 先跑通再优雅

MC2的代码演进路径始终是: 扁平→继承 / 手工→模板 / 局部→公共。

不要在第一次实现时就追求"最终架构":
- 新算子先写扁平tiling [NOP-2, all_gather_matmul]，变体增多后再重构 [matmul_all_reduce]
- 公共代码先放局部 [NOP-1 mc2_templates]，稳定后再上浮 [mc2/common]
- tilingkey先手工编码 [FEA-3之前]，成熟后再迁移APT [FEA-3]——但不能batch迁移 [REV-3]

反面教训: SmallM决策树优化(REV-2)试图一步到位引入3549行的自动tuning框架，因覆盖性不足被整体回退。如果分阶段引入(先框架后规则)，回退粒度可控。

---

## 9.6: Cross-Validation Report (Phase 9.6.3)

用3个Phase 8/9未验证过的commit做交叉验证。

### 验证commit

1. MAIN:156765f3 (BUGFIX) — AlltoAllvGroupedMatmul TT量化问题修复, 11 files, +679/-72
2. MAIN:8709a41a (FEATURE) — groupSize checking移到tiling侧并支持自动推导, 12 files, +279/-64
3. MAIN:77411b7d (REFACTOR) — Dispatch/Combine HCCL Buffer地址计算重构, 3 files, +658/-450

### 覆盖率

conventions已有规则覆盖率约60-70%:
- 骨架层规则(Section 1-5: 注册宏/架构层次/API模式): 覆盖良好，所有commit均验证通过
- 反模式(Section 7): 7.5(函数膨胀)和7.11(日志缺值)在实际commit中被修复，验证了反模式的准确性
- 场景指南(Section 12-13): op_host层场景覆盖好，op_kernel层重构场景偏弱

### 发现的缺失规则(已回补到conventions)

1. 7.12 边界比较运算符off-by-one: >=和>混淆是实际bug来源 (156765f3)
2. 7.13 校验逻辑层级迁移规范: 校验从op_api下沉到op_host的决策准则 (8709a41a, 156765f3)
3. 7.14 kernel内存布局文档化: 超过3个地址偏移计算须附ASCII art布局图 (77411b7d)
4. 7.4扩展: 跨算子常量重复定义泛化(GROUP_MNK_BIT_SIZE等) (8709a41a)
5. 13.4扩展: 新增Validation migration和Kernel buffer refactor的层级指南

### 结构性发现

conventions的"决策层"指南主要面向op_host层，对op_kernel层的重构场景(内存布局、地址计算封装、Ping-Pong Buffer模式)覆盖不足。这是Phase 3-6分析时的结构性盲区——kernel层的分析主要关注了流水线和同步模式，而非代码组织和重构模式。

### 总体评估

经Phase 8(5 commits) + Phase 9.6.3(3 commits) = 8个真实commit的验证:
- 骨架层规则: 高可操作性，可直接用于code review checklist
- 反模式: 准确，Phase 8和Phase 9.6.3各发现2-4条缺失并回补
- 场景指南: op_host层覆盖好，op_kernel层需要后续补充(特别是MoE kernel的RDMA窗口/状态位协议/Ping-Pong Buffer)
