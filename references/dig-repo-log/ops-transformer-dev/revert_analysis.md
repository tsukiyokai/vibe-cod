# ops-transformer-dev Revert专项分析

共35个Revert提交，覆盖2025-09-11至2026-01-16。

## 一、Revert事件分类统计

| 类别 | 数量 | 占比 |
|------|------|------|
| TilingKey模板化整改失败 | 6 | 17% |
| 大型新算子/功能premature merge | 10 | 29% |
| 跨仓迁移协调失败 | 3 | 9% |
| 性能优化引入正确性问题 | 4 | 11% |
| 大规模重构一次性合入 | 4 | 11% |
| Cleancode/文案改动引入语义变更 | 3 | 9% |
| 构建系统/CI配置错误 | 3 | 9% |
| 功能开关批量开启 | 2 | 6% |

## 二、逐条分析

---

### 事件群1: TilingKey模板化整改系统性失败 (6条)

同一架构方案(将tilingKey从手动常量改为`GET_TPL_TILING_KEY()`模板宏)在多个MC2算子上连续失败，是本仓库最显著的系统性缺陷逃逸事件。

#### 2001261f7c Revert "MatmulAllreduce tilingKey整改【A2+310P】"
- 原始提交: 3ec314f89 (MR !2611, 2025-11-28)
- 涉及模块: mc2/matmul_all_reduce (23个文件, +2232/-7449行)
- 问题描述: 将MatmulAllReduce的tilingKey从展开式模板声明(大量ASCENDC_TPL_SEL宏枚举)重构为紧凑模板形式，新增`matmul_all_reduce_tiling_key.h`(711行)。revert时恢复了7000+行模板展开代码。合入约1天后revert。
- 缺陷逃逸信号: 7000+行架构性重构影响A2/310P两个平台多个量化/非量化路径，MR测试栏为空。

#### 8ead68ccfa Revert "mc2 tilingkey rectification for A2/A3（combine，dispatch）"
- 原始提交: 5114e8562 (MR !2803, 2025-11-29)
- 涉及模块: mc2/moe_distribute_combine, moe_distribute_combine_v2 (6个文件)
- 问题描述: 将combine/dispatch两算子的A2/A3版本tilingKey改为模板宏。引入`moe_distribute_combine_tiling_key.h`。合入不到1天被revert。
- 缺陷逃逸信号: 与下条属同一作者(cqk1107)同批重构，分别提交但未联合验证。

#### 20f4d5412b Revert "mc2 tilingkey rectification（MatmulReduceScatter V2）"
- 原始提交: f8f038c90 (MR !2437, 2025-11-29)
- 涉及模块: mc2/matmul_reduce_scatter_v2
- 问题描述: 将tilingKey从magic number累加重构为`GET_TPL_TILING_KEY()`宏。同批次第3个失败。
- 缺陷逃逸信号: 3个算子同一方案连续失败，说明方案设计阶段验证不足。

#### 58a7ef184e Revert "mc2 tilingkey rectification for A2/A3 and A5_isolation"
- 原始提交: bce10843f (MR !3100, 2025-12-03)
- 涉及模块: mc2/moe_distribute_combine/dispatch v1/v2 (20个文件, +1189/-601行)
- 问题描述: 4个算子的APT tilingkey整改，引入`*_apt.cpp`和`*_tiling_key.h`。合入同日revert。
- 缺陷逃逸信号: 4个算子合在一个PR中，一个出问题全部回退。

#### a2b032dcb2 Revert "tilingkey rectification for redecescatterv2_and_A5_isolation"
- 原始提交: 25318dd92 (MR !3126, 2025-12-03)
- 涉及模块: mc2/matmul_reduce_scatter_v2 (def, tiling, kernel)
- 问题描述: 同日同批次整改。新增apt文件和tiling_key头文件。同日revert。

#### 02505c922e revert unquant mmv3 tilingkey
- 原始提交: aa0feec38 + b69e14702 (2025-09-29/30)
- 涉及模块: mc2/matmul_all_reduce (310P非量化场景)
- 问题描述: 将310P非量化tilingkey从旧格式迁移到mmv3格式，但host下发的key与kernel中TILING_KEY_IS判断值不一致，导致无法命中正确kernel分支。两次fix均未解决，次日revert。
- 缺陷逃逸信号: tilingkey的host/kernel匹配是跨文件强约束，缺乏自动化一致性检查。

事件群小结: tilingKey模板化是一个反复失败的架构重构尝试。11-28至12-03的5天内连续5次revert(来自同一作者或同一方案)，后续12月又有MC2 TilingData整改(aee3fb63e)再次失败。根因是方案在第一次失败后没有暂停评审，而是继续推进同一方案到其他算子。

---

### 事件群2: 大型新算子/功能premature merge (10条)

#### 1788aaceea revert MoeInitRoutingV2
- 原始提交: b45b42cf5 (2025-09-11, 40个文件, ~7300行)
- 涉及模块: moe/moe_init_routing_v2
- 问题描述: 首次迁入MoeInitRoutingV2全套代码(arch35 SIMT、多核排序、aclnn接口等)。1小时内revert，次日通过"911_commit"分支重新合入。构建配置问题或文件遗漏。

#### 8611973812 revert (MoeComputeExpertTokens)
- 原始提交: 06f2d9d1a (2025-09-14, 31个文件, ~3400行)
- 涉及模块: moe/moe_compute_expert_tokens
- 问题描述: 迁入moe_compute_expert_tokens全套代码。2小时内revert，中间有修复尝试但失败。可能与同日MoeInitRoutingV2 revert有共享依赖关联。

#### d1396cb6fa Revert "open unpermute"
- 原始提交: 7992c245f (2025-09-18, moe_token_unpermute全套, ~5000行)
- 涉及模块: moe/moe_token_unpermute + moe/moe_token_permute_grad
- 问题描述: 引入moe_token_unpermute新算子并重构permute_grad。48分钟内revert。同日16:32再次合入并修复。
- 缺陷逃逸信号: 同日3次提交(open-revert-reopen)暴露本地验证不充分。

#### d769324960 revert v2 grad
- 原始提交: (2025-09-19前, 36+个文件, ~5000行)
- 涉及模块: moe/moe_init_routing_v2_grad
- 问题描述: MoeInitRoutingV2反向算子全套代码被整体删除。3天后重新合入。可能依赖前向算子尚不稳定。

#### 5bc15d9ef9 Revert "add alibi mask feature for split fuse"
- 原始提交: 88b9ba5a7 (MR !2977, 2025-12-09, 21个文件, +3268行)
- 涉及模块: attention/fused_infer_attention_score (splitfuse路径)
- 问题描述: 为FIA的splitfuse路径新增alibimask能力。包含V5接口文档(2292行)、3个新参数、tiling修改、online_softmax大幅改动。合入1天后同一作者自行revert。大型新特性验证不充分。

#### 8c0fff581c Revert "add GMM定轴算法"
- 原始提交: 117127d1c (2025-11-15, 50个文件, +8726行)
- 涉及模块: gmm/grouped_matmul (含gmm_infra基础设施)
- 问题描述: GMM定轴搬移算法的第二次尝试。第一次(c51db5b7, 11-14)当日revert，第二次(117127d1, 11-15)两天后再次revert。引入了完整gmm_infra底层模块(arch抽象、cross_core_sync、GEMM pipeline等)。
- 缺陷逃逸信号: 同一feature 3天内被revert两次，暴露"fix-then-re-merge"仓促模式。

#### 1ff006a430 Revert "aclnn接口调试"
- 原始提交: b088b3e41 (2025-11-07, 7个文件, ~1008行)
- 涉及模块: mc2/quant_reduce_scatter (aclnn接口层)
- 问题描述: commit message"接口调试"明确表明代码处于调试阶段不应合入主干。约18小时后revert。
- 缺陷逃逸信号: "调试"代码进入master，review未拦截。

#### 90bd1c6440 revert//add attention_update
- 原始提交: 394a1be77 (MR !2581, 2025-11-26, 30个文件, +4493行)
- 涉及模块: attention/attention_update (全新算子)
- 问题描述: AttentionUpdate算子全套代码(CMake、API、host、kernel、UT)合入1小时16分钟后revert。次日重新合入成功。

#### a9e3525e86 revert: moe_finalize_routin单算子编译
- 原始提交: 7f967fb95 (2025-11-25)
- 涉及模块: moe/moe_finalize_routing_v2, moe_finalize_routing_v2_grad
- 问题描述: 将include路径从本地相对路径改为全局路径，命名空间从platform::/ops::迁移到Ops::Base::。单算子编译环境下API不可用，次日revert。

#### dfa78b4b93 revert//MLAProlog fp8全量化
- 原始提交: 53377fd68 (MR !2298)
- 涉及模块: attention/mla_prolog (fp8全量化支持)
- 问题描述: 为MLA Prolog新增fp8(FLOAT8_E4M3FN)量化支持，含2个新VF内核模块。MR描述完全空白，branch名"merge master into master"暗示匆忙合入。约3天后revert。

事件群小结: 10个premature merge中有7个在合入后2小时内revert，说明问题在本地编译/CI即可发现。核心原因是缺乏pre-merge CI gate和迁仓前的集成验证checklist。

---

### 事件群3: 跨仓迁移协调失败 (3条)

#### 1337ec1579 Revert "add scatter_pa_kv_cache"
- 原始提交: fc96a5ec1 (2025-11-24, 49个文件, ~14986行)
- 涉及模块: attention/scatter_pa_kv_cache
- 问题描述: 从gitee canndev迁移scatter_pa_kv_cache算子。PR描述明确提到"需要和gitee删除PR同时合入避免打断"，但实际协调失败，合入19小时后revert。12月2日重新成功添加。
- 缺陷逃逸信号: 跨仓迁移的"同时合入"约束仅靠人工协调，缺乏技术保障。

#### 3140d5a245 Revert "mc2_3rd与nn仓解耦"
- 原始提交: a7cc8dc61 (2025-10-30, 402个文件, +40125/-11629行)
- 涉及模块: mc2/3rd, cmake/, common/stub/, scripts/及大量mc2算子
- 问题描述: 解除transformer仓对nn-dev仓的依赖，涉及构建系统、stub头文件、include路径、打包配置。14小时后revert。4个co-author来自不同团队，协调复杂度高。
- 缺陷逃逸信号: 402个文件的超大跨仓变更一次性合入，缺乏分阶段策略。

#### 05c16e686c Revert "fix include"
- 原始提交: bb90fa511 (2025-09-15, common/mc2/moe多模块include路径)
- 涉及模块: common/CMakeLists.txt, mc2/3rd/, moe/多个算子
- 问题描述: 将`tiling/tiling_base.h`统一改为`tiling_base/tiling_base.h`并移除common/inc目录。4小时后revert。上游tiling库路径重组在本仓构建环境未就绪。

事件群小结: 跨仓操作的核心问题是缺乏原子性保障机制和依赖就绪检查。

---

### 事件群4: 性能优化引入正确性问题 (4条)

#### a478f78879 Revert "FAG L1 Policy"
- 原始提交: (release分支r1.25.0, 17个文件, +4289行)
- 涉及模块: attention/flash_attention_score_grad
- 问题描述: 将FAG的matmul数据路径从GM改为TSCM(L1)，引入完整的TSCM buffer管理系统(ping-pong双缓冲)和L1 matmul配置。同时放宽head dimension支持范围(d==64||128 -> d<=128)。在release分支上被完整revert。
- 缺陷逃逸信号: 4000+行架构级优化在release分支合入，测试未覆盖所有head dimension组合。

#### 014e93e772 Revert "MoeInitRoutingV2 & MoeFinalizeRoutingV2性能优化PR3935"
- 原始提交: 0f71dbfab (MR !3935, 2025-12-30)
- 涉及模块: moe/moe_finalize_routing_v2, moe/moe_init_routing_v2
- 问题描述: 为k==1场景(单expert)引入专门的buffer分配和内存计算路径。合入8天后revert。k==1场景的buffer公式与通用路径不一致，可能在某些shape下导致溢出。
- 缺陷逃逸信号: 特定参数值的优化分支增加了代码复杂度，参数空间大难以穷举测试。

#### 8b58f7c1f3 Revert "optimize gmm split_k"
- 原始提交: 9198c2d82 (MR !3007, 2025-12-13)
- 涉及模块: gmm/grouped_matmul (host tiling + kernel)
- 问题描述: 为split_k场景添加L2 Cache策略优化(输出超L2容量时跳过)和专用base参数。合入后3小时紧急revert。后来fe2689e8e重新成功合入。
- 缺陷逃逸信号: L2 Cache策略是硬件行为相关优化，需硬件实测而非仅code review。TilingData新增字段可能与kernel侧不兼容。

#### 336b4e33ff Revert "Optimized cross core"
- 原始提交: 8a9cdc5ee (2025-11-10, 14个文件)
- 涉及模块: attention/common/op_kernel (flash_attention_score buffer管理)
- 问题描述: 将BMM1/BMM2输出从LocalTensor/GlobalTensor改为Buffer<UB/GM, CROSS_CORE_SYNC>抽象，修改了SyncType模板参数。合入1.3小时后同一作者自行revert。
- 缺陷逃逸信号: 核心基础设施(buffer.h)的接口签名变更影响面广，PR描述为空。

事件群小结: 性能优化类变更有最高的缺陷密度。特征：引入新的条件分支、改变数据路径/缓存策略、放宽参数范围。

---

### 事件群5: 大规模重构一次性合入 (4条)

#### aee3fb63e8 Revert "MC2 TilingData整改"
- 原始提交: 68f5fcb70 (MR !3198, 2025-12-03, 161个文件, +4480/-5172行)
- 涉及模块: mc2下多个算子的tiling层
- 问题描述: MC2模块全局TilingData结构整改。161个文件一次性变更。合入约1天revert。
- 缺陷逃逸信号: 横切面重构应分模块逐步合入，而非全量推送。

#### 0d4ebf9463 Revert "delete quant_reduce_scatter & quant_all_reduce"
- 原始提交: b52bc9c1c (MR !3757, r1.25.0分支, 59个文件, -8894行)
- 涉及模块: mc2/quant_reduce_scatter, mc2/quant_all_reduce
- 问题描述: 在release分支上删除两个完整算子模块(8894行)。1小时19分钟后revert。这些算子在发布版本中仍然需要，属于错误清理。
- 缺陷逃逸信号: 大规模删除操作缺乏依赖影响分析，PR描述空白。

#### 423572ba6c revert//修复gmm若干问题
- 原始提交: ce09ff7c5 (MR !2632, 2025-11-27)
- 涉及模块: gmm/grouped_matmul (infershape, tiling, op_api)
- 问题描述: 标题说"修复若干问题"但实际是多个不相关修改打包(infershape逻辑变更、函数重命名、越界保护、op_api错误处理)。"大杂烩MR"反模式，无法隔离问题。不到1天revert。

#### ef0947575c Revert "tiling模板代码优化"
- 原始提交: 65f75abdd (2025-10-13)
- 涉及模块: attention/common/op_kernel/arch-310/ (FlashAttentionScore)
- 问题描述: 将tiling模板选择从hardcode条件泛化为计算公式，合并多条TPL_ARGS_SEL，添加constexpr裁剪。泛化后某些D/Dv组合错误选择buffer策略。同日revert(10小时)。

---

### 事件群6: Cleancode/文案改动引入语义变更 (3条)

#### 5474fdf4a6 revert//cleancode
- 原始提交: d77a07323 (MR !2989, 2025-12-03)
- 涉及模块: attention/fused_infer_attention_score/op_host/infershape
- 问题描述: 以"代码规范"为名，将`GetQueryBSND/GetQueryTND/InferAttentionOutShape/InferLseOutShape`返回类型从`ge::graphStatus`改为`void`，删除了不支持layout时的else分支错误处理(`return ge::GRAPH_FAILED`)。导致非法layout静默通过。
- 缺陷逃逸信号: cleancode PR被视为低风险review门槛低，但实际是语义变更(删除错误处理、改变函数签名)。

#### 31ead8dec2 Revert "修改拦截报错信息"
- 原始提交: c8cfb861c (MR !2727, 2025-12-01)
- 涉及模块: attention/incre_flash_attention (IFA tiling)
- 问题描述: 将PageAttention场景的错误提示改为描述MLA/int8场景的提示，但实际校验逻辑仍是PageAttention。错误消息与校验条件不匹配。后续被来回折腾多次(revert->重提->再revert->最终重提)。
- 缺陷逃逸信号: 错误消息修改缺乏与校验逻辑的交叉核对。

#### 1eb3f0ce04 revert//修改拦截报错信息
- 原始提交: (同上系列后续提交, 2025-12-01)
- 涉及模块: attention/fused_infer_attention_score, incre_flash_attention
- 问题描述: 在FIA中新增value tensor维度校验(DimNum != 3/4报错)，在IFA中增加MLA场景特殊拦截(int8+keyRope+headDim=512要求NZ)。拦截条件过严，误拦合法使用路径。
- 缺陷逃逸信号: 防御性编程(新增校验)的review应反向思考"是否会误拦合法场景"。

---

### 事件群7: 构建系统/CI配置错误 (3条)

#### 422548259b Revert "修改分包json文件合并问题"
- 原始提交: 86a006ad1 (MR !4385, 2025-12-23)
- 涉及模块: cmake/func.cmake, scripts/ci/operator_list.yaml, scripts/package/merge_binary_info_config.py
- 问题描述: 同时修改cmake(添加_apt后缀处理)、CI分包yaml(重组算子分组格式)、打包脚本(dict.update替换为60+行深度合并)。跨层改动在36小时后revert。
- 缺陷逃逸信号: 跨cmake/yaml/python三层改动难以评估完整影响。

#### 330ff015fa revert add py
- 原始提交: (2025-12-18)
- 涉及模块: CMakeLists.txt, cmake/custom_build.cmake
- 问题描述: 将py脚本安装从按名称精确安装(`${_op_name}.py`)改为glob通配(`*.py`)。导致foreach循环中每次迭代安装所有py文件N次。
- 缺陷逃逸信号: cmake中glob替代精确匹配破坏了按算子隔离安装语义。

#### ad447f9501 Revert "gmm tilingKey tpl rectify"
- 原始提交: cf9a84d0f (2025-11-03, 9个文件, +2224行)
- 涉及模块: gmm/grouped_matmul (tiling, kernel, tests)
- 问题描述: 重构GMM tilingKey系统，新增tiling_key.h头文件。host代码直接`#include "../../op_kernel/grouped_matmul_tiling_key.h"`引入了host-kernel间的不当编译依赖。约24小时revert，次日修复后重新合入。

---

### 事件群8: 功能开关批量开启 (2条)

#### f1fa74193b revert example
- 原始提交: 8316623b6 "开启example" (2026-01-06)
- 涉及模块: tests/test_config.yaml (gmm子模块)
- 问题描述: 批量移除`examples: False`开关，但gmm的swiglu_quant/swiglu_quant_v2/grouped_matmul_add的example测试实际跑不通。

#### 3b689fd480 revert-gmm-example
- 原始提交: 同上 8316623b6
- 涉及模块: tests/test_config.yaml (grouped_matmul, rope_with_sin_cos_cache)
- 问题描述: 同一根因的不同批次修复。需要两人分两天做两次revert才解决完。
- 缺陷逃逸信号: 批量开启功能开关时缺乏逐个算子验证。

---

## 三、Revert时间线热力图

```
2025-09: |||||||  (7个: MoE系列迁仓+include路径)
2025-10: |||      (3个: tiling模板+mc2解耦+tilingkey)
2025-11: ||||||||||| (11个: GMM定轴x2+crosscore+aclnn+tilingkey系列+新算子系列)
2025-12: |||||||||||| (12个: tilingKey整改高峰+MC2重构+cleancode+分包+split_k)
2026-01: ||          (2个: example开关+FAG L1+MoE性能)
```

12月是revert高峰(12个)，对应tilingKey模板化整改的集中推进期。

## 四、核心发现与审查启示

### 发现1: TilingKey模板化是反复失败的系统性工程

11月下旬至12月上旬，tilingKey模板化方案在MatmulAllReduce、MatmulReduceScatterV2、MoeDistributeCombine/Dispatch共4组算子上连续失败，累计6次revert。方案在第一次失败后没有暂停评审，而是继续推进到其他算子。

审查规则: 当一个架构方案在首个算子上失败并revert后，reviewer应要求方案进行根因分析和方案修正，再继续推进到其他算子。

### 发现2: Premature merge是最大的revert来源

10个(29%)revert属于功能未完成就合入。特征：
- 大量新文件(30-50个文件，数千行)一次性加入
- PR描述/测试栏为空
- 合入后极短时间(48分钟~几小时)revert
- 通常是构建级别的问题(编译失败/链接错误)

审查规则: 对于新增算子(>10个新文件)的PR，必须填写测试栏并提供CI通过证据。

### 发现3: 性能优化是最危险的变更类型

4个性能优化revert中3个涉及缓存策略(L1/L2/TSCM)变更。这类改动修改了数据在硬件存储层级间的流动路径，正确性高度依赖硬件行为，纯code review难以验证。

审查规则: 涉及缓存策略(L1/L2/TSCM/GM路径)的性能优化，必须附带硬件实测精度对比数据。

### 发现4: Cleancode伪装下的语义变更

3个表面是"代码规范/报错信息"修改的PR，实际删除了错误处理分支、改变了函数签名、引入了不匹配的错误消息。Review时被当作低风险变更放行。

审查规则: reviewer应区分"纯格式cleancode"和"改变函数签名/删除错误处理的语义变更"，后者需要与功能变更同等审查标准。

### 发现5: 跨仓操作缺乏原子性保障

3个跨仓迁移/解耦的revert都因为两个仓库的变更不能原子性同步而导致过渡期构建失败。

审查规则: 跨仓变更必须有明确的上下游就绪检查，不能仅靠PR描述中的文字约定。

### 发现6: PR描述缺失是系统性问题

35个revert中，绝大多数原始提交的MR描述、测试字段为空或仅模板占位。这使得reviewer无法仅从PR信息判断变更正确性。
