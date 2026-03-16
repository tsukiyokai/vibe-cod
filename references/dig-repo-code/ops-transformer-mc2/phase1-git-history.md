# Phase 1: Git History Mining

## 1.1 高频修改文件统计

mc2/目录共有247个涉及C++文件修改的commit（取top 500 commit hash去重后）。
总commit数: 361。

### Top 30 高频修改文件

按文件被修改的commit次数排序：

| 排名 | 修改次数 | 文件路径 | 层 | 算子 |
|------|----------|----------|-----|------|
| 1 | 37 | mc2/moe_distribute_dispatch_v2/op_host/op_tiling/moe_distribute_dispatch_v2_tiling.cpp | op_host/tiling | dispatch_v2 |
| 2 | 34 | mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp | op_host/tiling | combine_v2 |
| 3 | 28 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h | op_kernel | dispatch_v2 |
| 4 | 24 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h | op_kernel | dispatch_v2 |
| 5 | 19 | mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.h | op_kernel | combine_v2 |
| 6 | 16 | mc2/allto_all_matmul/op_api/aclnn_allto_all_quant_matmul.cpp | op_api | allto_all_matmul |
| 7 | 14 | mc2/moe_distribute_dispatch/op_host/op_tiling/moe_distribute_dispatch_tiling.cpp | op_host/tiling | dispatch_v1 |
| 7 | 14 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.cpp | op_kernel | dispatch_v2 |
| 9 | 13 | mc2/grouped_mat_mul_allto_allv/op_host/op_tiling/grouped_mat_mul_allto_allv_tiling.cpp | op_host/tiling | gmm_alltoallv |
| 10 | 12 | mc2/moe_distribute_combine/op_host/op_tiling/moe_distribute_combine_tiling.cpp | op_host/tiling | combine_v1 |
| 10 | 12 | mc2/grouped_mat_mul_allto_allv/op_kernel/grouped_mat_mul_allto_allv.h | op_kernel | gmm_alltoallv |
| 12 | 11 | mc2/moe_distribute_dispatch_v2/op_host/moe_distribute_dispatch_v2_infershape.cpp | op_host/infershape | dispatch_v2 |
| 12 | 11 | mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.cpp | op_kernel | combine_v2 |
| 12 | 11 | mc2/common/inc/tiling/mc2_tiling_utils.h | common | - |
| 15 | 10 | mc2/moe_distribute_combine_add_rms_norm/op_kernel/moe_distribute_combine_add_rms_norm.h | op_kernel | combine_arn |
| 15 | 10 | mc2/moe_distribute_combine_add_rms_norm/op_host/op_tiling/moe_distribute_combine_add_rms_norm_tiling.cpp | op_host/tiling | combine_arn |
| 15 | 10 | mc2/allto_allv_grouped_mat_mul/op_host/op_tiling/allto_allv_grouped_mat_mul_tiling.cpp | op_host/tiling | alltoallv_gmm |
| 18 | 9 | mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch.h | op_kernel | dispatch_v1 |
| 18 | 9 | mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.cpp | op_host/tiling | alltoall_mm |
| 20 | 8 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_a2_layered.h | op_kernel | dispatch_v2 |
| 20 | 8 | mc2/moe_distribute_combine_add_rms_norm/op_kernel/moe_distribute_combine_add_rms_norm.cpp | op_kernel | combine_arn |
| 20 | 8 | mc2/matmul_allto_all/op_api/aclnn_quant_matmul_allto_all.cpp | op_api | matmul_alltoall |
| 20 | 8 | mc2/matmul_allto_all/op_api/aclnn_matmul_allto_all.cpp | op_api | matmul_alltoall |
| 20 | 8 | mc2/grouped_mat_mul_allto_allv/op_kernel/grouped_mat_mul_allto_allv.cpp | op_kernel | gmm_alltoallv |
| 20 | 8 | mc2/distribute_barrier/op_kernel/distribute_barrier.h | op_kernel | barrier |
| 20 | 8 | mc2/attention_to_ffn/op_kernel/attention_to_ffn.h | op_kernel | attn2ffn |
| 20 | 8 | mc2/allto_all_matmul/op_kernel/arch32/allto_all_matmul.h | op_kernel | alltoall_mm |

### 分析要点

热点集中度分析:
- MoE算子(dispatch/combine v1/v2)占据前5名中的4个，是修改最频繁的算子家族
- dispatch_v2是绝对热点：top 30中占7个文件（tiling 37次, kernel.h 28次, fullmesh.h 24次, kernel.cpp 14次, infershape 11次, a2_layered.h 8次, tiling.h 7次）
- op_host/op_tiling层（tiling文件）是修改最频繁的层: top 10中占5个
- op_kernel层: kernel头文件(.h)修改频率高于实现文件(.cpp)，说明接口和模板代码变动频繁

层分布统计(top 30):
- op_host/tiling: 8个文件
- op_kernel: 14个文件
- op_api: 3个文件
- op_host/infershape: 1个文件
- common: 1个文件

公共文件: mc2/common/inc/tiling/mc2_tiling_utils.h 被修改11次，是MC2级tiling工具的核心文件

## 1.2 Revert Commit 分析

mc2/目录共发现3个revert commit。

### Revert列表

| Commit | 原始变更描述 | 涉及文件 | 分析 |
|--------|------------|----------|------|
| 349083a7 | Revert "修改了all_gather_matmul算子ut的op_host组件的用例输入方式，改成csv表格" | all_gather_matmul ut文件 | UT输入方式变更被revert，说明csv表格方式不可行或引入问题 |
| 8ade8c3e | Revert "dispatch优化syncall" | moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h (132行变更), moe_distribute_v2_constant.h | 同步优化被revert，dispatch的PipeAll/SyncAll优化是高风险操作。涉及58行新增75行删除的重构 |
| e5988e2b | revert (无详细描述) | moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h (33行变更) | fullmesh模板的变更被revert，14行新增19行删除 |

### Revert分析要点

数量少(3个/361总commits = 0.83%)，但全部集中在两个领域:
1. dispatch_v2的kernel同步机制（2/3个revert）: syncall优化和fullmesh模板都被revert，印证了dispatch_v2的op_kernel层是最脆弱的代码区域
2. UT输入格式变更（1/3）: 测试基础设施的变更也有风险

反模式提取:
- 对dispatch kernel的PipeAll/SyncAll同步机制做优化需要格外谨慎，历史上优化尝试被revert
- fullmesh模板的变更也有被revert的记录

## 1.3 Bug-fix Commit 分析

### 统计概览

mc2/目录共74个bugfix commit（匹配 fix/bug/修复/bugfix/hotfix）。
其中约15个为纯文档/资料修复，约59个为代码bugfix。
总commit数361，代码bugfix占比约16%。

### 按算子子目录的Bug密度

排除3rd和广撒网式commit(如096526e6修复全部算子ut的单个commit)后，
以"bugfix涉及的文件数/总修改文件数"近似bug密度:

| 排名 | 算子子目录 | bugfix文件数 | 总修改文件数 | bug密度 |
|------|-----------|-------------|-------------|---------|
| 1 | allto_all_matmul | 25 | 181 | 13.8% |
| 2 | moe_distribute_dispatch_v2 | 24 | 275 | 8.7% |
| 3 | matmul_all_reduce | 21 | 467 | 4.5% |
| 4 | matmul_allto_all | 14 | 207 | 6.8% |
| 5 | moe_distribute_combine_v2 | 12 | 183 | 6.6% |
| 6 | moe_distribute_combine | 12 | 114 | 10.5% |
| 7 | common | 12 | 256 | 4.7% |
| 8 | moe_distribute_dispatch | 9 | 126 | 7.1% |
| 9 | grouped_mat_mul_allto_allv | 8 | 149 | 5.4% |
| 10 | batch_mat_mul_reduce_scatter_allto_all | 7 | 86 | 8.1% |

(注: bugfix文件数来源于精确grep匹配，但096526e6等单个commit修改全部算子的情况会略微膨胀各子目录的数字)

### Bug类型分类

从74个bugfix commit的message中提取bug类型分布:

1. 同步/Sync相关 (约10个):
   - f8eb3c97 "fix bug:big shape sync fail"
   - 662f162c "fix a synchronization issue of dispatch v2 fullmesh"
   - d76a776a "fix pipe V_S"
   - 4b6e703c "ffn2attn fix Sync bug"
   - c2d880ba "Fix SyncFunc EventId Type"（跨11个算子子目录）
   - 3f21ad63 "dispatch v2 fix winIn addr and sync bug"
   - 5dce387d "MatmulAllReduce CommFp8 Sync Fix"
   → 同步是MC2算子最大的bug来源，涉及pipe同步、事件同步、通信同步

2. 编译/类型错误 (约8个):
   - 73e01410 "Bugfix for Dispatch/Combine V3 compile"
   - 27cbe00d "fix error compile A5"
   - 65c132d8 "fix: compile fail"
   - 78282635 "fixup bug uint64_t to int64_t for quant_all_reduce"
   - 5a217f2d "fix type bug when using GetAttrPointer"
   → 多arch编译和类型不匹配是常见问题

3. 量化相关 (约6个):
   - a6afc73e "修复allToAllMatmul场景，smoothQuant功能"
   - a882f8c0 "alltoallvgmm 修复量化 模板转置问题 & aclnn 共享转置问题"
   - ad0c0a9b "alltoallmatmul A16W4和A16W8性能修复"
   - 151ca802 "fix combineARN精度问题"
   → 量化场景的正确性和性能是持续修复的领域

4. Fullmesh/分层模板相关 (约4个):
   - 662f162c "fix a synchronization issue of dispatch v2 fullmesh"
   - 9691bcc3 "dispatch v2 fullmesh 2-dim mask fix bug"
   - ac0860f7 "Fix A3 fullmesh bug"
   - 99755a1b "Layered Dispatch And Combine Bug Fix"
   → fullmesh和分层拓扑是复杂度和bug的高发区

5. 参数校验/空指针 (约5个):
   - 35e60a61 "当mmX和mmWeight相关参数都不输入时，mmXQuantMode无需判空"
   - 2bd3ff8c "修复静态图mmy空tensor拦截报错"
   - aaae3128 "fix some null pointer risks"
   - 44338c14 "修复alltoallmatmul的api校验"
   → 参数边界条件处理是持续改进的领域

6. 性能相关fix (约4个):
   - 60550452 "alltoallmatmul 修复int4 8卡性能"
   - ad0c0a9b "alltoallmatmul A16W4和A16W8性能修复"
   - d1d95dfb "fix matmul_all_reduce ub conflict"
   → 性能问题常表现为UB冲突、同步开销

### 重点Commit详解

096526e6 "修复mc2算子ut的opapi因打桩变更引起的问题":
- 一次性修改了24个算子子目录的ut文件
- 说明op_api层的打桩(mock)接口变更会波及所有算子的UT，是一个脆弱点

c2d880ba "Fix SyncFunc EventId Type":
- 一次性修改了11个算子子目录的代码（attention_to_ffn, common, distribute_barrier, ffn_to_attention, matmul_all_reduce, matmul_allto_all, moe_distribute_combine, moe_distribute_combine_add_rms_norm, moe_distribute_dispatch, moe_distribute_dispatch_v2, moe_update_expert）
- EventId的类型错误影响面极广，说明同步事件ID的类型定义是全局性的

4f5aeb89 "mc2 ut kernel fix":
- 修改了13个算子子目录
- UT kernel层的修复也是广撒网式的

### 初步反模式清单

1. 同步机制变更需要全链路验证: MC2算子的同步bug是最大类别，dispatch的syncall优化曾被revert (8ade8c3e)
2. 类型定义变更影响面广: EventId类型错误(c2d880ba)一次波及11个子目录，公共类型定义必须严格review
3. 量化场景需要独立验证: smoothQuant/A16W4/int4等量化模式频繁出现bug，说明量化路径的测试覆盖不足
4. fullmesh模板是bug高发区: dispatch_v2的fullmesh相关bug有4个独立fix，且有1个revert
5. op_api打桩接口变更是脆弱点: 一次打桩变更导致24个算子UT失败(096526e6)

## 1.4 重构Commit分析

mc2/共约338个含重构关键词(refactor/rework/重构/优化/clean/整改)的commit(包括body匹配)。

### 重构主题分类(按commit message关键词)

| 主题 | Commit数 | 说明 |
|------|----------|------|
| Arch/Platform适配 | 47 | socVersion→npuArch迁移、新Arch(950/3510)支持 |
| 头文件/依赖清理 | 30 | aclnn前缀重命名、未使用头文件删除 |
| 命名整改 | 6 | json命名、checker.h重命名、aclnn_*_base→*_base |
| TilingKey整改 | 5 | tilingkey模板化、tilingkey迁移 |
| 告警/Codecheck | 5 | 编译告警清理、shadow warning |
| 超大函数拆分 | 2 | combineV2 tiling拆分、GMM超大函数清理 |

### 关键重构事件详解

#### 1. socVersion → npuArch 迁移 (全仓级)

- 核心Commits: 17de35a6 "(tiling+gentask+infershape+register)replace socVersion with npuArch", 36391f24 "(register)replace socversion with npuArch"
- 范围: 贯穿四层(tiling+gentask+infershape+register)
- 意义: 将平台判断从具体SoC版本号切换到更抽象的Arch编码，提高可扩展性。新算子不应再使用socVersion做分支判断。

#### 2. aclnn前缀文件重命名 (MC2模块级)

- Commit: a851b9d3 `将aclnn_*_base.h/cpp文件统一重命名为*_base.h/cpp`
- 日期: 2026-03-05
- 意义: 去除aclnn框架前缀，使文件命名更简洁统一。说明aclnn是历史遗留的命名风格，新代码不应使用此前缀。

#### 3. combineV2 tiling大函数拆分

- Commit: 916aa93c `combineV2 tiling big function split`
- 日期: 2026-03-12
- 影响: mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp (单文件)
- 意义: tiling函数膨胀到需要拆分的程度，反映MoE算子tiling逻辑的复杂性，也是代码质量治理的信号。

#### 4. GMM超大函数清理 (度量驱动)

- Commit: 05088f5b `Cmertric_clean-GMM 超大函数清理`
- 日期: 2026-01-27
- 描述: "Cmertric_clean-GMM 超大函数清理，拆解了一些超大函数"
- 影响: allto_allv_grouped_mat_mul和grouped_mat_mul_allto_allv的tiling/infershape/graph三层各一个文件(共6个文件)
- 意义: 使用Cmetric工具做度量驱动的代码质量改进。

#### 5. MoE MC2 tiling重构

- Commit: 2e23c04c `re_tiling_moe_mc2`
- 日期: 2026-03-11
- 影响: mc2/moe_distribute_dispatch_v2/op_host/op_tiling/moe_distribute_dispatch_v2_tiling.cpp
- 意义: dispatch_v2的tiling逻辑重构(该文件是全仓修改次数最多的文件, 43次)

#### 6. cleancode: 删除分层Dispatch的PipeAll

- Commit: 0889195e `【cleancode】删除分层Dispatch的PipeAll`
- 日期: 2026-03-12
- 意义: 清理不再需要的PipeAll同步，反映kernel层同步策略从保守的全局同步向细粒度同步演进。

### 架构演进方向总结

1. 平台抽象化: SoC版本号 → Arch编码 (具体芯片型号 → 架构代)
2. 命名去框架化: aclnn_*_base → *_base (去除框架前缀)
3. 函数治理: 大函数拆分(Cmetric度量驱动)
4. 同步精细化: 从保守的全局同步(PipeAll/SyncAll)向细粒度同步演进
5. Tiling持续重构: tiling是最复杂也最活跃的模块，持续被优化和重构

### 被重构最多的模块

| 模块 | 重构commit | 总commit | 重构活跃度 |
|------|-----------|----------|------------|
| moe_distribute_dispatch_v2 | 113 | 127 | 最高 |
| moe_distribute_combine_v2 | 84 | 98 | 次高 |
| matmul_all_reduce | 56 | 64 | 高 |
| allto_all_matmul | 49 | 54 | 高 |
| moe_distribute_dispatch | 52 | 64 | 高 |

注：高数字包含了grep匹配到merge commit body中关键词的情况，实际纯重构commit占比较低，大部分是功能迭代中的改进。

---

## 1.5 算子新增时间线

### Wave 0: 仓库初始化 (2025-09-28)

commit b122ed22 `init` — 一次性引入16个算子(仓库创建时已存在于内部开发):

基础MatMul+Comm算子:
- matmul_all_reduce (MatMul + AllReduce融合)
- all_gather_matmul (AllGather + MatMul融合)
- matmul_reduce_scatter (MatMul + ReduceScatter融合)
- matmul_all_reduce_add_rms_norm (MatMul + AllReduce + Add + RMSNorm融合)
- inplace_matmul_all_reduce_add_rms_norm (Inplace变体)

MoE专用算子:
- moe_distribute_dispatch (v1, MoE专家分发)
- moe_distribute_dispatch_v2 (v2, 多拓扑支持)
- moe_distribute_combine (v1, MoE专家合并)
- moe_distribute_combine_v2 (v2, 多拓扑支持)
- moe_distribute_combine_add_rms_norm (合并 + Add + RMSNorm)
- moe_update_expert (专家更新)
- distribute_barrier (分布式屏障)

GMM + AlltoAll算子:
- allto_allv_grouped_mat_mul (AlltoAllv + GroupedMatMul)
- grouped_mat_mul_allto_allv (GroupedMatMul + AlltoAllv)
- allto_all_all_gather_batch_mat_mul (AlltoAll + AllGather + BatchMatMul)
- batch_mat_mul_reduce_scatter_allto_all (BatchMatMul + ReduceScatter + AlltoAll)

### Wave 1: Ascend950支持 + 新算子 (2025-12-25 ~ 2026-01-29)

| 日期 | Commit | 新增算子 | 动机 |
|------|--------|---------|------|
| 2025-12-25 | f33d3cec | all_gather_matmul_v2, matmul_reduce_scatter_v2, grouped_mat_mul_all_reduce | 为Ascend950新增v2版本 |
| 2025-12-29 | 24239d77 | attention_to_ffn, ffn_to_attention | Attention到FFN的数据重排 |
| 2026-01-22 | 04eedcaf | matmul_allto_all | MatMul + AlltoAll融合 |
| 2026-01-26 | 60a38fbb | quant_reduce_scatter | 量化ReduceScatter |
| 2026-01-28 | e767c3fb | quant_all_reduce | 量化AllReduce |
| 2026-01-29 | d79d1792 | allto_all_matmul | AlltoAll + MatMul融合(反向) |

Wave 1特征: v2版本为950 arch适配，量化变体独立成算子，数据重排算子(attn↔ffn)新增。

### Wave 2: V3版本 + Setup/Teardown分离 (2026-02-28 ~ 2026-03-14)

| 日期 | Commit | 新增算子 | 动机 |
|------|--------|---------|------|
| 2026-02-28 | feat commit | moe_distribute_combine_setup, moe_distribute_combine_teardown | AIV直驱URMA通信方式，setup/teardown拆分 |
| 2026-02-28 | host commit | moe_distribute_dispatch_setup, moe_distribute_dispatch_teardown | host侧实现 |
| 2026-03-04 | bb44d77a | moe_distribute_dispatch_v3, moe_distribute_combine_v3 | 新增context作为必选输入 |
| 2026-03-14 | 29c51384 | allto_allv_quant_grouped_mat_mul, quant_grouped_mat_mul_allto_allv | 最新量化GMM AlltoAll算子 |

Wave 2特征: MoE算子的架构演进(context传递/setup-teardown解耦)和更多量化组合。

### 完整目录覆盖

mc2/共35个算子目录 + common/ + 3rd/ + tools/。

截至2026-03-15各波次引入的算子数:
- Wave 0 (init): 16个
- Wave 1 (2025-12~2026-01): 7个
- Wave 2 (2026-02~03): 6个
- 来源覆盖: 29个算子目录已确认首次commit时间

### 演进脉络

```
Wave 0 (2025-09, init):
  基础MC2算子族 = MatMul+Comm(4个) + MoE(6个) + GMM+AlltoAll(4个) + 辅助(2个)
       |
Wave 1 (2025-12~2026-01, Ascend950):
  v2版本(arch35适配) + 量化独立算子 + 数据重排算子
       |
Wave 2 (2026-02~03, MoE架构升级):
  v3(动态context) + setup/teardown解耦 + 量化GMM组合
```

---

## 1.6 v1 → v2 → v3 演进分析

### all_gather_matmul / matmul_reduce_scatter: v1 → v2

v2引入动机 (commit 6b31281b, 2026-01-28):
- 描述: "add new ops: all_gather_mamtul_v2、matmul_reduce_scatter_v2；update all_gather_mamtul、matmul_reduce_scatter、matmul_all_reduce、moe_update_expert to support Ascend950"
- v2版本专为Ascend950(arch35)新增，v1保持对旧arch的支持
- 同一commit还更新了v1版本以"support Ascend950"，说明v1也做了950适配但以v2为主要950载体

早期commit显示(f33d3cec "update docs" → 8c34a873 "文档格式整改" → 6b31281b实际代码):
v2目录先创建了docs骨架，再填入实现。

### moe_distribute_combine / dispatch: v1 → v2 → v3

#### v1 → v2

v1和v2都在仓库init时已存在(b122ed22)，说明v2在开源前就已开发完成。

v2相比v1的关键差异(从文件结构和修改历史推断):
- v2新增fullmesh通信拓扑: dispatch_v2_full_mesh.h (31次修改，v1无此文件)
- v2新增layered分层拓扑: dispatch_a2_layered.h (v1有a2_layered.h但修改频率低)
- v2 tiling复杂度显著更高: dispatch_v2 tiling 43次 vs v1的15次
- v2的commit总量显著更多: dispatch_v2 127次 vs v1 64次，combine_v2 98次 vs v1 59次

v2的核心价值: 支持ring + fullmesh + layered三种通信拓扑，适配更多网络互联配置。

#### v2 → v3

v3引入时间: 2026-03-04 (commit bb44d77a)

v3引入动机(来自commit message和关联Issue):
- bb44d77a: "op kernel of mc2Context" — "更新kernel侧代码，新增接口和必选输入用于传递context信息，在kernel内部增加对新路径的判断并获取context信息。"
- 8a88dd6f: "Add context as an input to dispatch/combine" — "新增MoeDistributeDispatchV3/CombineV3原型，完善host侧适配，支持传入context。"
- 关联Issue #1030: "新增动态获取context输入的方法，以便实时更新"

v3的核心变化: 新增context作为必选输入tensor，支持在运行时动态传递上下文信息(而非v1/v2的编译期/启动期静态配置)。

v3当前状态(截至2026-03-15):
- dispatch_v3: 8个commit, 3个bug fix — 早期开发阶段
- combine_v3: 7个commit, 3个bug fix — 早期开发阶段
- 典型的早期问题: 编译失败(73e01410 "Bugfix for Dispatch/Combine V3 compile"), 图构建失败(d0fc7865 "Fix D/C v3 graph failure")

#### setup/teardown分离 (2026-02-28)

新模式:
- moe_distribute_combine_setup / moe_distribute_combine_teardown
- moe_distribute_dispatch_setup / moe_distribute_dispatch_teardown

引入动机: "feat: 新增combine setup&teardown 950 AIV直驱URMA通信方式"

含义: 将dispatch/combine的初始化(RDMA buffer注册、通信域建立)和清理(资源释放)逻辑从主kernel中分离，专门用于Ascend950的AIV直驱URMA通信。这是一种架构解耦——主kernel只负责数据搬运和计算，通信连接的生命周期管理独立出去。

### 版本演进总结

```
v1 (Wave 0, 2025-09):
  - 基础ring拓扑通信
  - 静态配置
  - 仍在维护，但非主力

v2 (Wave 0, 2025-09):
  - 多拓扑: ring + fullmesh + layered
  - 最复杂、最活跃的版本(全仓修改量最高)
  - 当前主力版本

v3 (Wave 2, 2026-03):
  - 动态context传递
  - 早期开发阶段
  - 编译/图构建问题仍在修复中

setup/teardown (Wave 2, 2026-02):
  - 通信生命周期管理解耦
  - Ascend950 AIV直驱专用
  - 与v3并行发展
```

版本演进的核心驱动力:
1. 新硬件支持(Ascend950/arch35) → v2增加更多通信拓扑 + setup/teardown分离
2. 通信拓扑扩展(ring → fullmesh → layered) → v2的核心价值
3. 运行时灵活性(静态配置 → 动态context) → v3的核心价值
4. 架构解耦(初始化/清理从主kernel分离) → setup/teardown的价值

---

## 综合反模式清单 (Phase 1汇总)

### 反模式1: 贸然优化kernel层同步机制
- 错误做法: 直接优化dispatch kernel中的SyncAll/PipeAll，未在所有通信拓扑(ring/fullmesh/layered)下验证
- 正确做法: 修改同步逻辑必须在全部拓扑配置下进行端到端验证
- git证据: 8ade8c3e Revert "dispatch优化syncall", e5988e2b Revert fullmesh变更

### 反模式2: tiling函数无限膨胀
- 错误做法: 在单个tiling函数中堆积所有场景(量化/非量化/各arch/各拓扑)的参数计算逻辑
- 正确做法: 按功能模块拆分(shape校验/参数计算/config填充)，使用dispatch或策略模式处理不同场景
- git证据: 916aa93c "combineV2 tiling big function split", 05088f5b "GMM超大函数清理"

### 反模式3: 新arch支持时遗漏条件编译分支
- 错误做法: 新增arch支持时只修改主路径，遗漏其他条件编译分支和模板特化
- 正确做法: 新arch支持需要系统性检查所有#ifdef分支、模板特化、tiling key、register注册
- git证据: 27cbe00d "fix error compile A5", 65c132d8 "fix: compile fail", 73e01410 "Bugfix for D/C V3 compile"

### 反模式4: socVersion硬编码做平台分支
- 错误做法: 在代码中直接使用SoC版本号(如"Ascend910B")做if/else分支
- 正确做法: 使用npuArch抽象层(如arch32/arch35)做平台分支判断
- git证据: 17de35a6 "replace socVersion with npuArch" — 全仓四层联动修改

### 反模式5: 公共类型定义变更未做全量影响评估
- 错误做法: 修改公共头文件中的类型定义(如EventId类型)时只验证当前算子
- 正确做法: 公共类型变更必须在所有依赖算子中验证
- git证据: c2d880ba "Fix SyncFunc EventId Type" — 波及11个算子子目录

### 反模式6: op_api打桩接口变更破坏全量UT
- 错误做法: 修改op_api层的mock/打桩接口时不考虑下游UT兼容性
- 正确做法: 打桩接口变更需要同步更新所有消费方UT，或提供向后兼容层
- git证据: 096526e6 "修复mc2算子ut的opapi因打桩变更引起的问题" — 波及24个算子子目录
