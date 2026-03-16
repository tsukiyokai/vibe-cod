# Phase 8: Validation Report

## 8.1 选取的真实commit

从mc2/目录2026-03-12~03-14的近期commit中选取5个，覆盖feature/bugfix/refactor三种类型:

| # | Commit | Type | Description | Files |
|---|--------|------|-------------|-------|
| 1 | e8d52ffb | refactor | matmul_all_reduce clean code: 消减重复代码和无用文件 | 14 files, -916 +150 |
| 2 | 29c51384 | feature | 新增AlltoAllvQuantGroupedMatmul和QuantGroupedMatmulAlltoAllv算子 | 105 files, +5698 -4819 |
| 3 | f8eb3c97 | bugfix | fix bug: big shape sync fail (HCCL handle数组上限16→63) | 1 file, +2 -2 |
| 4 | 916aa93c | refactor | combineV2 tiling big function split | 1 file, +477 -416 |
| 5 | 809fc00e | bugfix | fix output limit check of AllGatherMMV2 and MMReduceScatterV2 | 5 files, +21 -15 |

## 8.2 模拟Code Review

### Commit 1: e8d52ffb (refactor - clean code)

变更概要:
- 将mc2_gen_task_ops_utils_arch35.cpp中的函数实现内联到.h头文件
- 消除mc2_gen_task_ops_utils_arch35.cpp和stub.cpp中的重复代码(常量+数据结构+函数实现完全相同)
- 将6个op_api文件中重复的NnopbaseHcclServerType枚举和include集中到matmul_all_reduce_util.h
- 将v3/v4中重复的IsWeightNZFormat/IsAclnnPreTransposed/ProcessTransposedX2提取到util函数

conventions能检出的问题:
1. [7.8 Weak Symbol Duplication] NnopbaseHcclServerType在6个op_api文件中重复声明 -> 此commit正是在修复这个反模式，说明conventions准确识别了此问题
2. [7.4 Mc2SyncAll Copy-Paste] 此commit属于同类修复(消除重复定义) -> conventions的反模式描述适用
3. [6.1 Naming Conventions] 提取到util.h的函数使用了MatmulAllReduceIsWeightNZFormat和QuantMatmulAllReduceIsAclnnPreTransposed这样的超长前缀命名，conventions中没有关于"公共工具函数命名"的具体规范

conventions检出结果: 2/3 relevant rules applied

conventions遗漏:
- 缺少"header-only vs cpp分离"的规范。此commit将.cpp实现移到.h，这是一个有争议的做法(减少了文件但增加了编译依赖)，conventions未涵盖
- 缺少"stub文件管理"规范。mc2_gen_task_ops_utils_arch35_stub.cpp和正式版完全一样的代码被消除，但conventions未解释stub文件的用途和何时需要

### Commit 2: 29c51384 (feature - 新增算子)

变更概要:
- 新增AlltoAllvQuantGroupedMatmul和QuantGroupedMatmulAlltoAllv两个量化GMM算子
- 完整的四层结构: op_host(def+infershape+tiling) + op_kernel(apt.cpp+tiling.h+tiling_key.h) + op_graph(gen_task+fallback+proto) + op_api
- 使用mc2_templates框架
- 同时重构了allto_allv_grouped_mat_mul(非量化版本)

conventions能检出的问题:
1. [1.1 Four-Layer Structure] 新算子100%遵循四层目录结构 -> PASS
2. [2.1 OP_ADD] def.cpp中使用class AlltoAllvQuantGroupedMatMul : public OpDef + 标准Input/Output声明 -> PASS
3. [2.8 HcclGroup] 声明了group属性 -> PASS
4. [3.1 Kernel Entry] workspaceGM+tilingGM作为最后两个参数 -> PASS
5. [3.2 REGISTER_TILING] REGISTER_TILING_DEFAULT(QuantAlltoAllvGroupedMatmulTilingData) -> PASS
6. [3.6 Workspace Null Check] userWorkspace == nullptr检查 -> PASS
7. [4.1 gen_task Dual Version] BUILD_OPEN_PROJECT双版本IMPL_OP/IMPL_OP_CT -> PASS
8. [4.2 CalcOpParam] CommonKFCMc2CalcParamFunc + NpuArch分派 -> PASS
9. [4.3 Fallback] EXEC_OPAPI_CMD(未直接使用但fallback结构正确) -> PASS(使用了自定义fallback逻辑而非EXEC_OPAPI_CMD)
10. [5.1 Two-Phase API] GetWorkspaceSize + Execute两阶段 -> 无法确认(op_api使用了已有的aclnn文件)
11. [7.3 Magic Number] fallback中使用constexpr size_t命名常量(ATTR_K_GROUP等) -> 这是一个好实践，但conventions中只在op_api层提到了magic number反模式

conventions检出结果: 8/8 mandatory rules all PASS

conventions遗漏:
- op_api中仍然内联了NnopbaseHcclServerType枚举定义(和commit 1修复的问题一样)，说明新算子开发时未参考已有的修复方向
- fallback中使用了枚举式的constexpr常量来索引属性(ATTR_K_GROUP = 0等)，这比magic number好但conventions未明确推荐这种模式
- mc2_templates框架的使用模式(A2avGmmScheduler/HcclA2avOp/QuantGroupedMatmul)在conventions中仅被提及为"2个算子试点"，缺乏具体的使用指南
- 新算子的tiling_key.h缺少conventions中描述的标准ASCENDC_TPL_ARGS_DECL模式(使用了不同的模板参数命名)
- config/目录下有binary.json和simplified_key.ini，conventions提到了"仅MoE系列有"但实际上量化GMM也有

### Commit 3: f8eb3c97 (bugfix - big shape sync fail)

变更概要:
- 将hccl_impl.h中MAX_HCCL_HANDLE_从16改为63
- 修复大shape场景下HCCL并行任务数不够导致的同步失败

conventions能检出的问题:
1. [7.6 Hasty Sync Optimization] 这正是一个同步相关的bug修复。conventions警告"kernel同步机制的优化变更需要全链路验证"，但此commit是修复而非优化
2. [3.9 Mc2SyncAll] 此bug在mc2_templates的通信层，不在Mc2SyncAll中
3. [8.2 Evolution: mc2_templates] 此文件在mc2_templates框架中，conventions提到该框架仅2个算子使用

conventions检出结果: 1 partially relevant rule (7.6的精神适用:通信机制变更需谨慎)

conventions遗漏:
- 缺少关于mc2_templates框架内部实现细节的规范(如HCCL handle并行数上限)
- 缺少关于"硬编码magic number"的kernel层规范 -- MAX_HCCL_HANDLE_ = 16是一个硬编码常量，注释说"hccl只支持最多16个任务并行"但实际上限是63
- 缺少"注释与代码一致性"规范 -- 旧注释说16，实际改为63但新注释被去掉了

### Commit 4: 916aa93c (refactor - tiling big function split)

变更概要:
- 将moe_distribute_combine_v2_tiling.cpp中的GetAttrAndSetTilingData大函数拆分为:
  - GetExpertsAttrAndSetTilingData
  - GetSharedAttrAndSetTilingData
  - GetTpAndEpAttrAndSetTilingData
  - 保留GetAttrAndSetTilingData作为入口，调用上述子函数

conventions能检出的问题:
1. [7.5 Tiling Function Bloat] 完美匹配! conventions明确警告"单个tiling函数超过200行应考虑拆分"，此commit正是在修复此反模式
2. [2.3 Tiling Registration] 重构后IMPL_OP_OPTILING入口不变 -> PASS

conventions检出结果: 1 rule directly applicable, 拆分方向正确

conventions遗漏:
- 缺少拆分后子函数的命名规范指导。当前拆分使用了Get*AttrAndSetTilingData模式，这是一个合理的命名但conventions未提供指导
- 缺少关于"attr指针判空检查顺序"的规范。此commit中epRankIdPtr和tpRankIdPtr的判空检查被放在了使用之后(先用后判空)，这是一个潜在bug，conventions未覆盖

### Commit 5: 809fc00e (bugfix - output limit check)

变更概要:
- AllGatherMatmulV2和MatmulReduceScatterV2的CheckHCCLSize()修复:
  1. sizeof(args_.geAType) -> ge::GetSizeByDataType(args_.geAType): sizeof取的是枚举类型大小(4字节)而非数据类型字节数
  2. 第二个OP_TILING_CHECK中sizeOfSingleM -> sizeOfSplitM: copy-paste导致的变量名错误
  3. OP_LOGE增加了实际值打印(sizeOfSingleM/sizeOfSplitM的值)
  4. 文档修正: "不支持空Tensor" -> "支持空Tensor"

conventions能检出的问题:
1. [6.2 Error Handling] op_host返回GRAPH_FAILED -> 格式正确，但OP_LOGE缺少实际数值打印 -> 此commit正在修复
2. [7.3 Magic Number] sizeof(args_.geAType)的使用不是magic number反模式，但属于"API误用" -> conventions未覆盖此类型错误
3. [2.7 Mc2CcTilingConfig] 不直接相关

conventions检出结果: 0 rules directly catch the bug

conventions遗漏:
- 缺少"sizeof vs GetSizeByDataType"的区分规范。这是MC2中一个常见错误来源: 对ge::DataType枚举使用sizeof得到的是枚举的存储大小(通常4字节)，而非该数据类型的实际字节数
- 缺少"copy-paste代码审查"规范。两处CheckHCCLSize()中第二个检查使用了sizeOfSingleM而非sizeOfSplitM，这是典型的copy-paste错误
- 缺少"OP_LOGE应包含实际运行时数值"的规范。原始代码只打印了字符串描述但没有打印实际的size值
- 缺少"文档与代码一致性"规范。"不支持空Tensor"与实际行为矛盾

## 8.3 规则用到/缺失/矛盾分析

### 被用到的规则(按频次排序)

| 规则 | 使用次数 | 场景 |
|------|---------|------|
| 1.1 Four-Layer Structure | 1 | Commit 2(新算子验证) |
| 2.1 OP_ADD | 1 | Commit 2 |
| 3.1 Kernel Entry | 1 | Commit 2 |
| 3.2 REGISTER_TILING | 1 | Commit 2 |
| 3.6 Workspace Null Check | 1 | Commit 2 |
| 4.1 gen_task Dual Version | 1 | Commit 2 |
| 4.2 CalcOpParam | 1 | Commit 2 |
| 7.4 Copy-Paste | 1 | Commit 1 |
| 7.5 Tiling Function Bloat | 1 | Commit 4 |
| 7.6 Hasty Sync Optimization | 1 | Commit 3(间接) |
| 7.8 Weak Symbol Duplication | 1 | Commit 1 |

总计: 11条规则在5个commit中被使用，覆盖了conventions 30+条中的约1/3

### 缺失的规则(应该有但conventions未写)

1. sizeof vs GetSizeByDataType区分
   - 重要性: 高(直接导致计算错误)
   - 适用范围: op_host层(tiling计算)
   - 建议: 新增反模式条目

2. OP_LOGE/OP_LOGD应包含运行时数值
   - 重要性: 中(影响调试效率)
   - 适用范围: 全层
   - 建议: 新增推荐规范

3. Copy-paste代码审查检查项
   - 重要性: 高(commit 5的两个bug都是copy-paste导致)
   - 适用范围: 全层
   - 建议: 在反模式7.4中扩展

4. mc2_templates框架使用指南
   - 重要性: 中(目前仅2+2=4个算子使用)
   - 适用范围: op_kernel层
   - 建议: 随框架推广补充

5. Stub文件管理规范
   - 重要性: 低(仅影响gen_task层)
   - 适用范围: op_graph层
   - 建议: 新增可选规范

6. Header-only vs cpp分离策略
   - 重要性: 低(编译策略选择)
   - 适用范围: mc2/common层
   - 建议: 记录当前两种做法并存的现状

7. attr指针使用前必须先判空
   - 重要性: 高(防止空指针解引用)
   - 适用范围: op_host层
   - 建议: 新增强制规范

8. 文档与代码一致性
   - 重要性: 中
   - 适用范围: docs/
   - 建议: 新增推荐规范

9. config/binary.json适用范围修正
   - 重要性: 低(事实性修正)
   - 当前conventions写"仅MoE系列有"，实际量化GMM也有
   - 建议: 修正描述

### 矛盾或不可操作的规则

1. [4.3 Fallback] conventions写EXEC_OPAPI_CMD是标准模式(36处/23文件)，但新算子(commit 2)的fallback未使用EXEC_OPAPI_CMD，而是自行实现了参数解析+API调用逻辑。两种做法都存在且都是合理的 -> 建议: 区分"使用EXEC_OPAPI_CMD的标准路径"和"自实现fallback的场景"

2. [7.8 Weak Symbol] conventions建议"提取到公共头文件"，commit 1确实这样做了，但commit 2的新算子op_api中又重新定义了NnopbaseHcclServerType -> 说明此规范虽然正确但还未被全面采纳，需要在conventions中强调"新算子必须复用已有的公共定义"

## 8.4 回补修正清单

基于验证结果，以下内容需要回补到conventions:

### 新增反模式

A. sizeof(ge::DataType) vs ge::GetSizeByDataType()
   - Severity: Error
   - Scope: op_host层(tiling计算)
   - 代码:
     ```cpp
     // bad: sizeof取的是枚举存储大小(4字节)，不是数据类型字节数
     uint64_t size = args_.kValue * sizeof(args_.geAType);
     // good: 使用API获取实际数据类型大小
     uint64_t size = args_.kValue * ge::GetSizeByDataType(args_.geAType);
     ```
   - Git evidence: 809fc00e

B. attr指针先用后判空
   - Severity: Error
   - Scope: op_host层
   - 代码:
     ```cpp
     // bad: 先解引用再判空
     int64_t epWorldSize = *epWorldSizePtr;
     OP_TILING_CHECK(epWorldSizePtr == nullptr, ...);
     // good: 先判空再解引用
     OP_TILING_CHECK(epWorldSizePtr == nullptr, ...);
     int64_t epWorldSize = *epWorldSizePtr;
     ```
   - Git evidence: 916aa93c (combineV2 tiling split中可见此模式)

### 新增推荐规范

C. OP_LOGE应包含关键运行时数值
   - Severity: Recommended
   - Scope: 全层
   - 代码:
     ```cpp
     // bad: 只有描述没有数值
     OP_LOGE(opName_, "Unsupported x1 size. Size exceeds 256MB.");
     // good: 包含实际数值便于调试
     OP_LOGE(opName_, "Unsupported x1 size %lu. Exceeds 256MB limit.", sizeOfSplitM);
     ```
   - Git evidence: 809fc00e

D. 新算子op_api必须复用公共枚举/工具定义
   - Severity: Recommended
   - Scope: op_api层
   - 说明: NnopbaseHcclServerType等枚举已在matmul_all_reduce_util.h中集中定义，新算子不应重新定义

### 事实性修正

E. config/binary.json + simplified_key.ini适用范围
   - 原文: "仅MoE系列有binary.json + simplified_key.ini"
   - 修正: MoE系列和量化GMM系列都有(allto_allv_quant_grouped_mat_mul等)

F. Fallback两种模式
   - 原文: "EXEC_OPAPI_CMD: 36处/23文件"暗示所有fallback都用此宏
   - 修正: 多数fallback使用EXEC_OPAPI_CMD，但部分算子(如量化GMM)的fallback自行实现参数解析和API调用

## 8.5 验证结论

conventions文档的整体质量:
- 对"新增算子"场景(commit 2): 8个强制规范全部能指导并验证通过，说明conventions对标准开发流程的覆盖是充分的
- 对"重构/清理"场景(commit 1, 4): conventions的反模式描述准确预测了实际被修复的问题(weak symbol重复、tiling函数膨胀)
- 对"bugfix"场景(commit 3, 5): conventions的覆盖较弱，仅能间接提示风险，无法直接捕获sizeof误用和copy-paste错误这类具体bug

需要补强的方向:
1. op_host层的"防御性编程"规范(判空顺序、sizeof陷阱、日志数值)
2. 反模式条目中增加更多具体的代码级anti-pattern(不仅是架构级)
3. mc2_templates框架的使用指南(随框架推广而完善)
