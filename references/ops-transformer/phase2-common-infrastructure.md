# Phase 2: Common Infrastructure Analysis

## 2.2 mc2/common/inc/ -- MC2公共头文件深度分析

### 概述

mc2/common/inc/ 包含53个 `.h` 头文件，分为三个层次:
- 顶层(19个): 算子定义、工具函数、日志、infershape、gen_task等
- tiling/(18个): tiling参数结构、性能模型、切分策略、tiling key生成
- kernel/(16个): kernel侧数据结构、基础块管理、计算模板、通信上下文

### 文件清单与职责

#### 顶层头文件(19个)

| 文件 | 职责 | 定义的关键实体 |
|------|------|----------------|
| op_mc2_def.h | MC2基础类型和HCCL参数结构 | HcclOperatorType枚举, HcclCombinOpParam/HcclAndAicpuResource结构, MOE_X_DTYPE/MOE_BIAS_DTYPE支持列表 |
| op_mc2.h | 算子输入/输出/属性索引枚举 | MC2Type, MC2InputIdx, MC2V2InputIdx, MC2OutputIdx, AllGatherMMAttrIdx等20+枚举, ApiParamDef |
| mc2_log.h | 错误处理宏和日志工具 | OPS_ERR_IF, OPS_CHECK, GE_ASSERT系列, ErrorResult(多类型转换), Mc2Log打印函数, CUBE_INNER_ERR_REPORT |
| ops_utils.h | 数学工具函数 | OpsUtils::Ceil/CeilAlign/CeilDiv/FloorDiv/Aligned/FloorAlign |
| math_util.h | 数学工具桥接层 | Ops::NN::MathUtil(转发到Ops::Base), ops::CeilAlign/FloorAlign/CeilDiv |
| aclnn_util.h | ACLNN API可见性宏 | ACLNN_API宏 |
| mc2_aclnn_util.h | MX Scale转换判断 | MC2Aclnn::IsNeedScaleTrans |
| context_transfer.h | tiling context解析 | MMRCtxInfo/ARNCtxInfo/MRNCtxInfo结构, ContextTransfer类(8个Assemble方法) |
| context_util.h | context空指针检查宏 | OPS_CHECK_NULL_WITH_CONTEXT / _RET |
| hccl_util.h | op_api层检查宏 | OP_API_CHECK |
| matmul_util.h | MatMul op_api层工具 | MmOpInfo/OpShapeInfo/OpBaseInfo/BmmNd2nzInfo结构, Exec*Op系列函数 |
| runtime_util.h | infershape工具函数 | StridedInterval, SubShapePara, BroadcastShape, SetAllUnknownDim等20+函数 |
| mc2_common_infershape.h | MC2公共infershape逻辑 | CommParas结构, AllGatherMatmulCommonInferShape/InferMatmulReduceScatterCommon |
| mc2_exception_dump.h | 异常dump功能 | Mc2Exception::Mc2ExceptionImpl(将HCCL window内容dump到文件) |
| mc2_gen_task_ops_utils.h | gen_task公共工具 | Mc2GenTaskOpsUtils类(IsComputationOnly, CommonKFCMc2GenTask, InsertHiddenInputs等) |
| mc2_gen_task_ops_utils_arch35.h | arch35 gen_task | Mc2Arch35GenTaskOpsUtils(CreateCCUFusionTask, GetCommAlg), GROUP_INFO_MAP_ARCH35 |
| mc2_hcom_topo_info.h | HCCL拓扑查询 | MC2HcomTopology(dlopen方式查询rankNum/topoType/cclBufferSize), COMM_MESH/SWITCH/RING常量 |
| matmul_all_reduce_gen_task_ops_utils.h | MatmulAllReduce gen_task | MatmulAllReduceGenTaskOpsUtils |
| mc2_a5_gen_task_utils.h | A5(950) gen_task | Mc2A5GenTaskUtils(CCU fusion task创建), COMM_ALG_FULLMESH_V1/V2/MTE/CCU |
| mc2_moe_gen_task_ops_utils.h | MoE gen_task | Mc2MoeGenTaskOpsUtils(多通信域task生成) |
| mc2_moe_utils.h | MoE infershape工具 | Mc2Moe::EpTpSizeCheck/DimNumCheck/GroupCheck等, OutShapeInfo |
| mc2_matmul_tiling_cfg.h | MatMul V3 tiling配置 | Mc2MatmulTilingCfg(继承Mc2MatMulTilingCfg, baseMLimit/rankDim/commCnt) |

#### tiling/子目录(18个)

| 文件 | 职责 | 定义的关键实体 |
|------|------|----------------|
| mc2_tiling_struct.h | tiling数据结构(TILING_DATA宏版本) | MC2ServerCfg/MC2HcommCfg/RCSTiling/Mc2Msg/TileL2Tiling/TileInfo/MC2MatmulV3TilingData(均用BEGIN_TILING_DATA_DEF宏定义) |
| formulaic_tiling_datatype.h | 公式化tiling的参数类型 | SocVersion/KernelType/MatmulCalcType枚举, MatmulParameters/HCCLInfo/HCCLFittingParameters/CutResult/TileArguments结构 |
| tiling_key.h | TilingKey生成 | RecursiveSum(constexpr递归), GET_TILINGKEY宏(TILINGKEYOFFSET=10^19 + 十进制编码) |
| matmul_formulaic_tiling.h | MatMul tiling算法 | MatmulFormulaicTiling类, TilingArgs/ChipArgs/BestArgs/MatmulRunInfo/SoCInfo结构, 硬件常量(BASE_BLOCK_M=128/N=256/K=64, L1=512KB, L0C=128KB) |
| hccl_formulaic_tiling.h | HCCL通信tiling + 单通信域融合 | FormPartition(切分对齐类), OneCalcOneCommBase(计算+通信配平基类, MAX_TILE_CNT=16) |
| matmul_performance.h | MatMul性能模型 | MatmulPerformanceModel(cube利用率预测, L2 cache估计, 时间反推), 性能常量(FP16 COMPUTES_PER_CYCLE=4096) |
| hccl_performance.h | HCCL通信性能模型 | HCCLPerformanceModel(分段拟合: 线性区+抛物线区, CommTime/InverseCommTime) |
| matmul_performance_arch35.h | arch35 MatMul性能模型 | MatmulPerformanceArch35(继承基类, baseM=256/N=256/K=128, 调和平均+指数衰减的cubeUtil模型) |
| hccl_performance_arch35.h | arch35 HCCL性能模型 | HCCLPerformanceArch35(继承基类, override InverseCommTime) |
| one_calc_two_comm_tiling.h | 双通信域tiling(MoE) | OneCalcTwoCommBase(EP+TP两个HCCLPerf), OneCalcTwoCommShardHBase(H轴切分变体) |
| mc2_fit_based_balance_tiling.h | arch35拟合配平tiling | Mc2FitBasedBalanceTiling(MatmulPerformanceArch35+HCCLPerformanceArch35, virtual方法可定制) |
| mc2_calc_num_blocks.h | AIC/AIV核数计算 | GetNumBlocks(按1:2 AIC:AIV比例取min) |
| mc2_opversion_manager.h | 算子版本管理 | OpVersionManager(线程安全单例, 存储opVersion) |
| mc2_tiling_common_var.h | tiling公共常量 | FP32_DATASIZE=4, FP16_DATASIZE=2, MAX_HCCL_HANDLE_LIMIT=32, MC2_DEBUG_ONLY_AICPU=4 |
| moe_tiling_base.h | MoE tiling基类 | MoeTilingBase(继承TilingBaseClass, 5个虚方法: GetShapeAttrsInfo/GetPlatformInfo/DoLibApiTiling/GetWorkspaceSize/PostTiling) |
| mc2_tiling_utils.h | tiling工具函数集 | Mc2TilingUtils类, 类型转换map(D_TYPE_MAP/HCCL_DATA_TYPE), supportedRankSizeSet按NpuArch分, GetTilingKey模板, Mc2QuantMode枚举, KFCMsgBody/KFCNotify |

#### kernel/子目录(16个)

| 文件 | 职责 | 定义的关键实体 |
|------|------|----------------|
| mc2_common_def.h | kernel侧通信算法枚举 | AscendC::CommAlgType(DEFAULT/FULL_MESH/DOUBLE_RING/SWITCH_WING), AscendC::HcclCombinOpParam(轻量版) |
| mc2_tiling_struct.h | kernel侧tiling结构(C struct版) | Mc2Tiling::MC2ServerCfg/MC2HcommCfg/RCSTiling/Mc2Msg/TileL2Tiling/MC2MatmulV3TilingData(与tiling/版本字段一一对应但用POD struct) |
| mc2_matmul_block.h | Matmul基础块管理 | MatmulBaseBlockMC2(Init/InitBlockIndex/UpdateBlockIndex/UpdateBlockParams/CalcGMOffset), BaseBlockOffset/BaseBlockArguments |
| mc2_matmul_block_l2cache.h | L2 Cache感知基础块 | MatmulBaseBlockL2Cache(继承MC2, L2切片分核), SplitType枚举(DEFAULT/L2CACHE), BlockType模板特化 |
| mc2_matmul_compute.h | 计算模板 | MatmulCompute<A,B,C,BIAS,SplitType>(封装MatmulImpl, Compute/ComputeWithL2Cache/ComputeWithoutIndex) |
| mc2_kernel_utils.h | kernel同步工具 | SyncFunc<HardEvent>(FetchEventID+SetFlag+WaitFlag三合一) |
| mc2_nd_to_nz.h | ND到NZ格式转换 | CopyPadNd2Nz, PrePaddingImplNd2Nz, MatrixND2NZ, MatrixBtoNZ, CastBFtoFloat, Gm2GmTrans |
| all_gather_matmul_tiling.h | AllGather tiling数据(kernel侧) | AllGatherMatmulTilingData(含mc2InitTiling+mc2CcTiling+3套TCubeTiling+3套L2Tiling+RCSTiling+AllGatherSoc) |
| moe_distribute_comm_ctx.h | A5 HCCL通信上下文 | HcclCombinOpParam(A5版, 64 rank支持, xnAddr/ckeAddr for CCU) |
| moe_distribute_base.h | MoE通信基础设施 | HcclOpResParam(A3版大结构, remoteRes数组), Mc2Kernel::GetRankId/GetRankDim/GetBaseWindAddrByRankId(A3/A5双实现), HcclA2CombineOpParam, HcclAiRMAInfo/WQ/CQ |
| mc2_moe_context.h | MoE上下文参数 | Mc2MoeContext(epRankId + kfcContextAddr + epHcclBuffer) |
| qbmm_asw_block_noncontiguous.h | 量化BMM非连续块偏移 | Mc2QuantBmmAswBlockNonContiguous(CalcGMOffset with batchWeight) |
| qbmm_mix_perblock_noncontiguous.h | 量化BMM逐块计算 | MatMulPerBlockASWNonContiguous(Init/Process/ProcessWithoutBatch, MAX_RANK_DIM=64) |
| qbmm_perblock_api_utils_noncontiguous.h | 量化BMM逐块API | MatMulPerBlockNonContiguous(Iterate/ProcessAivSingleK/UpdateUBOffsetC) |
| gm_ub_gm_copy.h | GM到UB到GM数据拷贝 | GmUbGmCopy(Init/Process, double buffer, 分核策略) |
| reduce_sum.h | AIV ReduceSum | ReduceSumForAlltoAll(Init/ExecuteReduceSum, 多rank累加+分核) |
| reduce_sum_cast_fp32.h | AIV ReduceSum(FP32累加版) | ReduceSumForAlltoAll(Cast到FP32做累加, 结果Cast回目标类型) |
| reduce_sum_utils.h | 数学工具(kernel侧) | AscendC::CeilDiv/FloorDiv/CeilAlign/FloorAlign/BlockAlignMod/MIN |

---

### 关键设计模式分析

#### 1. tiling数据结构的"双版本"设计

tiling/mc2_tiling_struct.h 用 `BEGIN_TILING_DATA_DEF` 宏定义结构(host侧序列化/反序列化):
```cpp
BEGIN_TILING_DATA_DEF(RCSTiling)
    TILING_DATA_FIELD_DEF(uint32_t, rankDim);
    ...
END_TILING_DATA_DEF;
REGISTER_TILING_DATA_CLASS(RCSTilingOp, RCSTiling)
```

kernel/mc2_tiling_struct.h 用纯C struct定义相同结构(设备侧直接内存映射):
```cpp
struct RCSTiling {
    uint32_t rankDim;
    ...
};
```

两者字段完全一一对应，但使用不同的定义方式。这是因为host侧需要自动序列化能力(TILING_DATA_FIELD_DEF生成getter/setter)，而kernel侧需要直接内存布局。

适用范围: MC2模块级强制
强制程度: 强制(两个文件字段必须同步)

#### 2. 性能模型驱动的计算通信配平

tiling层的核心设计: 用性能拟合模型预测计算时间和通信时间，然后配平。

完整链路:
1. MatmulPerformanceModel: 预测cube利用率(考虑shape/L2 cache/K对齐) → 预测matmul时间
2. HCCLPerformanceModel: 分段拟合通信时间(线性区+抛物线区) → 预测comm时间
3. OneCalcOneCommBase/OneCalcTwoCommBase: 配平 → 决定切分方案(CutResult)
4. FormPartition: 执行切分(离散对齐 or 连续对齐) → 输出tileCnt/tileLen/tailLen

关键约束:
- MAX_TILE_CNT = 16 (受FFTS队列大小限制)
- 切分对齐到baseM (默认128, arch35为256)
- 短块vs长块位置: 计算bound时短块后置, 通信bound时短块前置

三类性能模型:
- 基础版(910B/310P/910_93): MatmulPerformanceModel + HCCLPerformanceModel
- arch35版(950): MatmulPerformanceArch35 + HCCLPerformanceArch35(调和平均+指数衰减)
- 双通信域版(MoE): OneCalcTwoCommBase(EP AllToAll + TP AllGather/ReduceScatter)

适用范围: MC2模块级
强制程度: 强制(所有MC2算子tiling都经过此链路)

#### 3. 通信算法枚举的多处定义

CommAlg相关常量至少在4个位置定义:

a. tiling/mc2_tiling_struct.h (optiling namespace): `COMM_ALG_DEFAULT=0, FULL_MESH=1, DOUBLE_RING=2, SWITCH_WING=3`
b. kernel/mc2_common_def.h (AscendC namespace): `CommAlgType枚举: DEFAULT=0, FULL_MESH=1, DOUBLE_RING=2, SWITCH_WING=3`
c. kernel/mc2_tiling_struct.h (Mc2Tiling namespace): 同上值
d. mc2_tiling_utils.h (mc2tiling namespace): `COMM_ALG_DEFAULT=0, FULL_MESH=1, DOUBLE_RING=2, SWITCH_WING=3`

值完全一致，但分散在不同namespace和文件。这是因为host/kernel两侧不共享头文件，且各子系统有独立的namespace。

适用范围: MC2模块级
注意事项: 新增通信算法时必须同步修改所有位置

#### 4. SoC/Arch多版本适配模式

SocVersion枚举(formulaic_tiling_datatype.h):
```cpp
enum class SocVersion { SOC910_B, SOC310_P, SOC910_93, SOC910_B4, SOC950 };
```

NpuArch映射(mc2_tiling_utils.h中使用):
```
DAV_2002 → 310P系列 (rankSize: 1,2,4)
DAV_2201 → 910B/910_93系列 (rankSize: 1,2,4,8)
DAV_3510 → 950系列 (rankSize: 1,2,4,8,16,32,64)
```

关键分支点:
- 性能模型: 不同SoC有不同的cycle/频率/base block参数
- 核数: AIC_NUM_950=32, 默认=20
- 通信: A5(950)不支持某些HCCL接口(CommGetBufSizeCfg), 需要规避路径
- MTE状态区: A5需要前1MB作为状态区(MTE_STATE_ZONE_SIZE=1MB)
- Window地址: A3用remoteRes链表, A5用windowsIn数组+状态区偏移

适用范围: MC2模块级
强制程度: 强制(每个算子必须考虑arch适配)

#### 5. kernel侧基础块分配策略

MatmulBaseBlockMC2的分配逻辑:
1. 按baseM/baseN将矩阵划分为基础块网格(mBlockCnt x nBlockCnt)
2. 行优先(isRowOrder): 默认; 列优先: 当N > 5*M时切换
3. 均匀分核: blockCnt = total / usedCoreNum, 余数轮转分配(preCoreNum/preCoreStartIdx)
4. 尾块处理: mBaseTail/nBaseTail是最后一行/列的实际大小
5. L2 Cache版本: 在M/N方向增加L2 tile维度, 蛇形遍历(reverse N方向)

CalcGMOffset根据A/B矩阵格式(ND/NZ)和转置状态计算偏移:
- ND + no trans: offsetA = mBlockOffset * Ka
- NZ + no trans: offsetA = mBlockOffset * C0_SIZE (C0=16)
- C矩阵只支持ND格式

适用范围: MC2模块级
强制程度: 强制(所有matmul kernel使用此基础块管理)

#### 6. MoE通信资源管理的A3/A5双实现

moe_distribute_base.h用编译宏 `__NPU_ARCH__ == 3510` 切换两套实现:

A5(950): 使用HcclCombinOpParam(轻量, 直接windowsIn数组索引)
- GetRankId → winContext->rankId
- GetBaseWindAddrByRankId → windowsIn[rankId] + MTE_STATE_ZONE_SIZE

A3(910_93): 使用HcclOpResParam(重量级, localUsrRankId + remoteRes链表)
- GetRankId → winContext->localUsrRankId
- GetBaseWindAddrByRankId → 本卡localWindowsIn, 远端走remoteRes链表解引用

适用范围: MoE算子级
强制程度: 强制(不同arch的HCCL接口不同)

#### 7. TilingKey十进制编码

tiling_key.h定义了编译期递归编码:
```cpp
constexpr uint64_t TILINGKEYOFFSET = 10^19;
template <typename... Args>
constexpr uint64_t GET_TILINGKEY(Args... ids) {
  return TILINGKEYOFFSET + RecursiveSum(ids...);
}
// RecursiveSum: id0 + 10*id1 + 100*id2 + ...
```

mc2_tiling_utils.h中MC2特有编码:
```cpp
uint64_t tilingKey = optiling::RecursiveSum(castBias, 1, commAlg);
// 编码: castBias + 10*1 + 100*commAlg
```

每个数位代表一个配置维度(bias有无/ND2NZ/通信算法等)，kernel侧switch-case解码。

适用范围: MC2模块级
强制程度: 强制(tiling key是host-kernel协议的核心)

#### 8. 错误处理的层次化宏体系

mc2_log.h定义了完整的错误处理宏体系:

条件检查+日志+返回:
- OPS_CHECK(COND, LOG_FUNC, EXPR): 通用条件检查
- OPS_ERR_IF(COND, LOG_FUNC, EXPR): 带__builtin_expect的条件检查
- OP_TILING_CHECK(cond, log_func, expr): tiling专用(在optiling namespace)
- GE_ASSERT(exp, ...): 带error report的断言

空指针检查:
- OPS_CHECK_NULL_WITH_CONTEXT(context, ptr): 返回GRAPH_FAILED
- OPS_CHECK_NULL_WITH_CONTEXT_RET(context, ptr, ret): 自定义返回值

ErrorResult是特殊设计: 通过多个operator T()转换，使宏返回值能匹配各种函数签名(bool/graphStatus/unique_ptr/shared_ptr/vector/string等)。

适用范围: 仓库级
强制程度: 强制(所有host侧代码使用这套宏)

#### 9. 数学工具函数的多层定义

CeilDiv/CeilAlign/FloorDiv/FloorAlign至少在5处定义:
1. ops_utils.h (OpsUtils namespace): 基础版
2. math_util.h (Ops::NN/ops namespace): 桥接到Ops::Base
3. matmul_formulaic_tiling.h (mc2tiling namespace): MathCeil/AlignUp/AlignDown
4. reduce_sum_utils.h (AscendC namespace): kernel侧版本
5. 仓库common/的Ops::Base: 被math_util.h转发

这反映了MC2模块历史演进中各子系统独立发展，后来才逐步统一。

适用范围: 观察记录
注意事项: 新代码应优先使用Ops::Base版本

#### 10. Gen Task的arch分层

gen_task相关的类按arch分层:
- Mc2GenTaskOpsUtils: 通用基础(IsComputationOnly, CommonKFCMc2GenTask)
- Mc2A5GenTaskUtils: A5(950)特有(CCU fusion task, 非open project)
- Mc2Arch35GenTaskOpsUtils: arch35特有(CCU fusion task, open project兼容)
- MatmulAllReduceGenTaskOpsUtils: MatmulAllReduce专用
- Mc2MoeGenTaskOpsUtils: MoE专用(多通信域)

GROUP_INFO_MAP_ARCH35定义了每个算子的通信域数量:
- 单通信域(groupCnt=1): AllGatherMMV2, MMReduceScatterV2, MatmulAllReduce等
- 双通信域(groupCnt=2): MoeDistributeDispatch/Combine及其V2

适用范围: MC2模块级
强制程度: 强制(新算子必须在GROUP_INFO_MAP中注册)

---

### 依赖关系图

```
顶层:
  op_mc2_def.h ← hccl_util.h, mc2_exception_dump.h
  mc2_log.h ← context_util.h, mc2_common_infershape.h, mc2_moe_utils.h, mc2_exception_dump.h
  context_util.h ← runtime_util.h

tiling/:
  formulaic_tiling_datatype.h ← matmul_performance.h, hccl_performance.h
  matmul_formulaic_tiling.h ← hccl_formulaic_tiling.h, matmul_performance.h, hccl_performance.h
  matmul_performance.h ← matmul_performance_arch35.h
  hccl_performance.h ← hccl_performance_arch35.h
  hccl_formulaic_tiling.h ← one_calc_two_comm_tiling.h, mc2_fit_based_balance_tiling.h
  mc2_tiling_utils.h ← moe_tiling_base.h, mc2_exception_dump.h

kernel/:
  mc2_tiling_struct.h(kernel) ← mc2_matmul_block.h, mc2_nd_to_nz.h, all_gather_matmul_tiling.h
  mc2_matmul_block.h ← mc2_matmul_block_l2cache.h ← mc2_matmul_compute.h
  mc2_kernel_utils.h ← reduce_sum_utils.h ← reduce_sum.h, reduce_sum_cast_fp32.h, gm_ub_gm_copy.h
  moe_distribute_comm_ctx.h ← moe_distribute_base.h, mc2_exception_dump.h
  qbmm_asw_block_noncontiguous.h ← qbmm_mix_perblock_noncontiguous.h ← qbmm_perblock_api_utils_noncontiguous.h
```

---

### 关键常量汇总

硬件参数:
- BASE_BLOCK_M=128, BASE_BLOCK_N=256, BASE_BLOCK_K=64 (默认)
- arch35: baseM=256, baseN=256, baseK=128
- L1_SIZE=512KB-32, L0A/B=32KB(DB on), L0C=128KB(DB on), L2=192MB(full)
- C0_SIZE=16 (分形块边长)
- AIC_NUM_950=32, 默认AIC=20
- MAX_RANK_NUM=8(A2/A3), 64(A5 MTE), 768(A3 max), AICPU_MAX_RANK_NUM=128K

tiling约束:
- MAX_TILE_CNT=16 (FFTS队列限制)
- SHAPE_ALIGN_SIZE=256
- KVALUE_MIN=256, KVALUE_MAX=65535
- EXPERT_LOWER_LIMIT=2, EXPERT_UPPER_LIMIT=512

通信约束:
- HCCL_GROUP_NAME_MAX=128
- HCCL_MIN_TILE_LEN=64KB (细粒度), 2MB (粗粒度)
- MTE_STATE_ZONE_SIZE=1MB (A5状态区)
- ALL_GATHER_HCCL_MEM_LIMIT=256MB, NUM_LIMIT=16

性能模型:
- FP16_CUBE_CAL_POWER=4096 (每cycle计算量)
- CYCLE_PER_MICRO_SEC=1.8K(910B), 1.08K(310P), 1.65K(950)
- MAX_CUBE_UTIL=0.95, AVERAGE=0.75
- PART_L2_UTIL=0.85, FULL=0.75, NO_L2=0.65

---

## 2.1 ops-transformer/common/ -- 仓库级公共基础设施分析

### 概述

ops-transformer/common/ 是整个仓库共享的公共基础设施层，包含约97个.h头文件和16个.cpp实现文件。MC2模块大量依赖此层的tiling框架、fallback机制、kernel工具函数和错误报告宏。

目录结构:
- include/tiling_base/ (7个.h): tiling类注册框架、TilingKey编码、平台工具
- include/fallback/ (4个.h): aclnn fallback执行框架(dlopen动态加载op_api)
- include/kernel/ (16个.h): 仓库级kernel工具(hardware常量、迭代器、SIMD包装、内存管理)
- include/common/ (2个.h): op_api层常量和tensor工具
- include/err/ (1个.h): 错误报告宏(E89999 vector / E69999 cube)
- include/static/ (2个.h): 算子静态空间注册
- include/op_graph/ (1个.h): 算子proto定义(REG_OP)
- include/tiling_sink/ (2个.h): 设备侧tiling注册
- include/external/ (9个.h): 对外aclnn kernel头文件
- include/framework/ (1个.h): ONNX框架集成
- stub/ (~57个.h): op_api打桩实现(测试用)
- src/ (~16个.cpp): 实现文件(fallback_comm、tiling_util、tiling_sink、ONNX插件)

### 核心文件分析

#### tiling_base/tiling_base.h -- Tiling基类(仓库级强制)

定义 `Ops::Transformer::OpTiling::TilingBaseClass`，是所有算子tiling类的根基类。

DoTiling()模板方法定义了tiling的标准执行流程(8步):
1. GetShapeAttrsInfo() -- 获取输入/输出/属性信息
2. GetPlatformInfo() -- 获取平台硬件参数(核数/UB/L1/L0C)
3. IsCapable() -- 判断本tiling类是否适用(不适用返回GRAPH_PARAM_INVALID继续下一个)
4. DoOpTiling() -- 核心tiling计算(数据切分)
5. DoLibApiTiling() -- 高阶API tiling(如MatmulImpl的参数)
6. GetTilingKey() -- 计算tiling key
7. GetWorkspaceSize() -- 计算workspace大小
8. PostTiling() -- 保存tiling数据到context
(DumpTilingInfo()在最后做调试输出)

关键设计:
- IsCapable()返回false → DoTiling()返回GRAPH_PARAM_INVALID → 注册系统尝试下一个优先级的tiling类
- 这构成了"链式tiling选择"机制: 按优先级尝试多个tiling实现，第一个报告capable的胜出
- 保护成员: context_, ascendcPlatform_, numBlocks_, workspaceSize_, tilingKey_, aicoreParams_
- CalcTschBlockDim(): 按AIC:AIV比例计算TSCH block维度

硬件参数结构体有3个变体:
- AiCoreParams: 基础版(ubSize/numBlocks/aicNum/l1/l0a/l0b/l0c)
- CompileInfoCommon: 含socVersion + l2CacheSize + coreNum
- FACompileInfoCommon: FlashAttention专用版

MC2中使用: TilingBaseClass被30个文件、58处引用。MC2的MoeTilingBase继承它增加了MoE专用的5虚方法。

适用范围: 仓库级强制
MC2关联: mc2/common/inc/tiling/moe_tiling_base.h直接继承此类

#### tiling_base/tiling_templates_registry.h -- Tiling注册框架(仓库级强制)

定义了三层tiling注册机制和对应的宏:

1. TilingRegistryArch + RegisterArch: 按NpuArch注册
   宏: REGISTER_TILING_TEMPLATE_WITH_ARCH(op_type, class_name, archs, priority)
   MC2使用: 13处(arch35新算子: matmul_all_reduce_950, allto_all_*_base等)

2. TilingRegistryNew + RegisterNew: 按SocVersion注册
   宏: REGISTER_TILING_TEMPLATE_WITH_SOCVERSION(op_type, class_name, soc_versions, priority)
   MC2使用: 11处(arch31/32老算子: matmul_all_reduce_910/310等)

3. TilingRegistry + Register: 不区分平台
   宏: REGISTER_OPS_TILING_TEMPLATE(op_type, class_name, priority)
   MC2使用: 33处(MoE算子、grouped_matmul、matmul_all_reduce_add_rms_norm等)

执行逻辑(DoTilingImpl):
- 按priority从小到大遍历已注册的tiling类
- 实例化tiling对象 → 调用DoTiling()
- 返回GRAPH_PARAM_INVALID → 跳过，尝试下一个
- 返回GRAPH_SUCCESS/GRAPH_FAILED → 停止，返回结果

模式演进: arch31/32时代用SocVersion注册(REGISTER_TILING_TEMPLATE_WITH_SOCVERSION), arch35开始迁移到NpuArch注册(REGISTER_TILING_TEMPLATE_WITH_ARCH)。不区分平台的版本(REGISTER_OPS_TILING_TEMPLATE)用于MoE等自己在tiling内部做arch分支的算子。

适用范围: 仓库级强制
强制程度: 强制(所有算子tiling必须通过此框架注册)

#### tiling_base/tiling_key.h / tiling_type.h -- TilingKey编码(仓库级)

RecursiveSum + GET_TILINGKEY定义在两个文件中(tiling_key.h在Ops::Transformer::OpTiling命名空间，tiling_type.h在optiling命名空间)，实现完全一致。这是TilingKey十进制编码的仓库级定义。

tiling_type.h还定义了FlashAttention特有的枚举(AxisEnum, DtypeEnum, LayoutEnum, SparseEnum等)。MC2有自己的mc2/common/inc/tiling/tiling_key.h，与仓库级版本编码规则相同，但位域含义不同(MC2编码comm算法、bias、ND2NZ等)。

适用范围: 仓库级(编码机制一致), MC2模块级(位域含义专用)

#### fallback/ -- aclnn Fallback框架(MC2 op_graph层强制)

核心设计: 当MC2算子在计算only模式(无通信)下运行时，通过fallback机制将计算部分delegate给aclnn算子。

fallback.h: 基础fallback(单阶段)
- dlopen加载libopapi.so / libcust_opapi.so / libaclnn_*.so
- ConvertType()系列: gert::Tensor → aclTensor, ge::DataType → aclDataType, vector<int64_t> → aclIntArray
- EXEC_OPAPI_CMD宏: 完成 GetWorkspaceSize → MallocWorkspace → Execute 全流程
- ConvertedParams RAII包装: 确保转换后的参数在离开作用域时被释放

fallback_2stages.h: 两阶段fallback(prepare + launch分离)
- EXEC_OPAPI_PREPARE_CMD宏: 只执行GetWorkspaceSize阶段
- OpApiParams结构体通过context传递到第二阶段

fallback_comm.h / fallback_comm_2stages.h: 公共类型定义
- ToAclDataType(), OpApiAnyValue, OpApiParams, ExecuteOpLaunch()

MC2使用: EXEC_OPAPI_CMD在23个MC2算子的op_graph/fallback_*.cpp中使用(36处引用)。每个MC2算子都有对应的fallback实现。

注意: fallback.h中使用 `using namespace std` 和 `using namespace gert`，这在头文件中是反模式，但因为整个fallback命名空间只被op_graph层的cpp文件引用，影响可控。

适用范围: 仓库级(fallback框架), MC2 op_graph层强制(每个算子必须有fallback)
强制程度: 强制

#### kernel/common.h -- Kernel层同步宏(仓库级)

定义3个简化宏:
- SET_FLAG(trigger, waiter, e): 展开为 AscendC::SetFlag<AscendC::HardEvent::trigger_waiter>(e)
- WAIT_FLAG(trigger, waiter, e): 展开为 AscendC::WaitFlag<AscendC::HardEvent::trigger_waiter>(e)
- PIPE_BARRIER(pipe): 展开为 AscendC::PipeBarrier<PIPE_pipe>()
- FORCE_INLINE: inline __attribute__((always_inline))

MC2使用: SET_FLAG/WAIT_FLAG仅在mc2/common/inc/kernel/mc2_nd_to_nz.h中出现8次。MC2 kernel更多直接使用AscendC::SetFlag/WaitFlag原生API或mc2_kernel_utils.h中的SyncFunc封装。

注意: MC2有自己的同步封装(SyncFunc<HardEvent>)，比仓库级宏更灵活。

#### kernel/common_func.h -- Kernel层数学工具(仓库级)

定义了kernel侧的模板数学工具:
- RoundUp<ALIGN>(val): 编译期对齐
- RoundUp(val, align): 运行期对齐
- CeilDiv<DIVISOR>(dividend): 编译期向上取整除
- CeilDiv(dividend, divisor): 运行期版本
- BlockSize<Dtype>(): 32/sizeof(Dtype) -- 数据块大小
- MatrixSize<Dtype>(): 512/sizeof(Dtype) -- 矩阵块大小
- BlockSizeRoundUp/NumBlocksRoundUp/MatrixSizeRoundUp/NumMatrixsRoundUp
- L0HalfSize<Dtype>(): 32*1024/sizeof(Dtype)

MC2有自己的kernel数学工具(mc2/common/inc/kernel/reduce_sum_utils.h中的CeilDiv/FloorDiv/CeilAlign/FloorAlign)，这是数学工具函数重复定义的又一个实例。

#### kernel/hardware.h -- 硬件常量(仓库级)

定义ArchType枚举(ASCEND_V220/V200/M200)和模板化的HardwareInfo:
- l1Size=512KB, l0ASize/l0BSize=64KB, l0CSize=128KB
- ubSize=192KB, l2Size=192MB
- fractalSize=512, l1l0BlockSize=32, btBlockSize=64, fbBlockSize=128

注意: 这些值是FlashAttention等算子使用的默认硬件参数。MC2有自己的硬件参数体系(matmul_formulaic_tiling.h中per-SoC参数)，不直接使用此文件。

#### kernel/layout.h -- 数据格式枚举(仓库级)

DataFormatT: ND/NZ/ZN/ZZ/NN/VECTOR

MC2在kernel/mc2_common_def.h中也定义了自己的格式相关枚举，但NZ格式的C0_SIZE=16常量来自mc2_matmul_block.h。

#### kernel/util.h -- FlashAttention工具(仓库级)

定义了FA专用的:
- LayOutTypeEnum, TransposeLayoutEnum
- BoolCopyIn/Bit2Int8CopyIn: DataCopy+DataCopyPad的封装
- math::Ceil/Align: 又一组数学工具

MC2不直接使用此文件中的FA专用工具。

#### kernel/simd.h -- SIMD操作包装(仓库级)

对AscendC向量指令做了ArchType模板化的薄封装:
add_v, adds_v, cadd_v, brcb_v, cmax_v, conv_v, convr_v, div_v, exp_v, max_v, mul_v, muls_v, sub_v, maxs_v, mins_v, sqrt_v, ln_v, tranpose_v, cgmax_v, cgadd_v

MC2不直接使用此文件，而是在kernel中直接调用AscendC API。

#### kernel/iterator.h / gm_to_l1_iterator.h等 -- 数据搬运迭代器(仓库级)

iterator.h定义了数据搬运的模板骨架:
- gm_to_l1<ArchTag, DataType, FormatInGM, FormatInL1>
- l1_to_l0_a/l1_to_l0_b<ArchTag, DataType, IsTransPose, DFmtIn, DFmtOut>
- l0c_to_gm/l0c_to_l1<ArchTag, LayoutOut, ElementOut, ElementIn>

各gm_to_*_iterator.h文件提供具体格式(ND/NZ/ZN)的特化实现。

MC2不使用此迭代器框架，而是在mc2_matmul_compute.h中通过MatmulImpl封装计算流水线。

#### kernel/mem.h -- 内存管理(仓库级)

BufferType枚举(UB/CB/L0A/L0B/L0C)和AsdopsBuffer模板类。
按 `__DAV_C220_VEC__` / `__DAV_C220_CUBE__` 条件编译初始化不同的LocalTensor。

MC2不使用此文件。

#### kernel/mma.h -- 矩阵乘法包装(仓库级)

mmad模板结构体，封装AscendC::Mmad调用。
MC2使用MatmulImpl高阶API，不直接使用mmad。

#### err/ops_err.h -- 错误报告宏(仓库级强制)

- OPS_REPORT_VECTOR_INNER_ERR: 错误码E89999(向量单元)
- OPS_REPORT_CUBE_INNER_ERR: 错误码E69999(矩阵单元)

MC2使用: CUBE_INNER_ERR_REPORT在mc2_log.h中定义(类似但使用不同错误码E69999)，在39个文件中383处引用。

#### common/op_api_def.h -- op_api层常量

小文件，定义MAX_SUPPORT_DIMS_NUMS=8, KEEP_DTYPE=0等常量。MC2的op_api层偶尔使用。

#### static/static_space.h -- 静态空间初始化(仓库级)

StaticSpaceInitializer单例，在构造时注册OpImplSpaceRegistryV2。这是算子静态加载的入口。

#### tiling_sink/ -- 设备侧tiling(仓库级)

DeviceOpImplRegistry: 设备侧tiling函数注册表。MC2的MoE算子使用tiling_sink机制。

---

### MC2对仓库级公共设施的使用模式总结

| 公共设施 | MC2使用方式 | 使用频次 |
|---------|------------|---------|
| TilingBaseClass | MoeTilingBase继承它，其他MC2算子的tiling_base也继承 | 30文件/58处 |
| REGISTER_TILING_TEMPLATE_* | arch35用WITH_ARCH(13处), arch31/32用WITH_SOCVERSION(11处), MoE用OPS版(33处) | 57处/37文件 |
| fallback框架 | 每个MC2算子有对应的fallback_*.cpp | 23个算子/36处 |
| CUBE_INNER_ERR_REPORT | infershape和tiling中大量使用 | 39文件/383处 |
| RecursiveSum/GET_TILINGKEY | MC2有自己的独立副本，编码规则相同但位域含义不同 | -- |
| kernel/common.h的SET_FLAG | 仅mc2_nd_to_nz.h使用8次，MC2更多用SyncFunc | 1文件/8处 |
| kernel/common_func.h | MC2有自己的reduce_sum_utils.h，不直接使用 | 0处 |
| kernel/hardware.h | MC2不使用(有自己的硬件参数体系) | 0处 |
| kernel/simd.h | MC2不使用(直接调用AscendC API) | 0处 |
| kernel/iterator.h | MC2不使用(用MatmulImpl) | 0处 |

关键发现:
1. MC2强依赖仓库级tiling框架(TilingBaseClass + 注册宏)和fallback框架
2. MC2在kernel层几乎完全自给自足，不依赖仓库级的kernel工具(iterator/simd/mma/hardware)
3. MC2重新实现了数学工具函数(CeilDiv/CeilAlign等)，与仓库级版本重复
4. TilingKey编码机制仓库级统一(RecursiveSum+10^19偏移)，但MC2有自己的副本和位域定义
5. 注册宏的选择反映了arch演进: arch31/32用SocVersion注册 → arch35用NpuArch注册

### 仓库级设计模式(对MC2有影响的)

#### 1. 链式Tiling选择(Chain of Responsibility)

TilingRegistryArch/New/无版本三种注册表都实现了相同的选择逻辑:
- 按priority升序遍历已注册的tiling类
- 实例化 → DoTiling()
- GRAPH_PARAM_INVALID = "我不适用，跳过"
- GRAPH_SUCCESS = "成功"
- GRAPH_FAILED = "失败，中止"

这决定了IsCapable()的设计: 必须快速判断，不能有副作用。

适用范围: 仓库级强制
MC2表现: MC2算子的tiling_base中统一override IsCapable()做arch/shape/dtype检查

#### 2. Fallback = 计算only模式的标准实现

每个MC2算子的op_graph层都有一个fallback_*.cpp:
- 将MC2的融合算子拆解为"aclnn单算子调用"
- 通过dlopen动态加载libopapi.so，运行时查找aclnn函数指针
- 用EXEC_OPAPI_CMD宏编排: ConvertTypes → GetWorkspaceSize → MallocWorkspace → Execute

适用范围: MC2 op_graph层强制(每个算子一个fallback)
强制程度: 强制(CI要求fallback必须存在)

#### 3. 错误报告双通道(Log + Report)

OPS_INNER_ERR_STUB宏同时做两件事:
- OpLogSub(): 写运行时日志
- REPORT_INNER_ERR_MSG(): 写错误报告(持久化到错误文件)

MC2的mc2_log.h中CUBE_INNER_ERR_REPORT也遵循此模式。

适用范围: 仓库级强制

---

## 2.5 mc2/3rd/ -- 第三方算子库

### 概述

mc2/3rd/ 包含MC2依赖的"第三方"算子库(实际是仓库内维护的独立算子), 共431个C++文件:
- mat_mul_v3/: 矩阵乘(MM V3), MC2的核心matmul引擎
- batch_mat_mul_v3/: 批量矩阵乘(BMM V3), 量化算子使用
- common/: 共享工具(infershape, cube_util, tiling_cache, hash, lock)

### 关键架构点

#### mat_mul_v3 -- MC2 arch35的核心matmul引擎

op_host/op_tiling/arch35/ 是MC2 arch35路径主要使用的tiling:
- matmul_v3_asw_tiling/asw_basic_tiling/asw_full_load_tiling: ASW(自适应滑窗)三种模式
- matmul_v3_basic_streamk_tiling/stream_k_tiling: StreamK两种模式
- matmul_v3_tiling_advanced: 高级tiling入口(选择ASW/StreamK/FullLoad)
- matmul_tiling_cfg.h: Mc2MatMulTilingCfg基类定义(MC2的mc2_matmul_tiling_cfg.h继承它)

op_kernel/arch35/ 是MC2 arch35 kernel使用的matmul实现:
- mat_mul_asw_block.h/mat_mul_asw_kernel.h: ASW kernel(4块宽滑窗+蛇形扫描)
- mat_mul_fixpipe_opti.h: fixpipe优化(cube输出直接进fixpipe, 不经过UB)
- block_scheduler_aswt.h/block_scheduler_streamk.h: 块调度器

MC2通过继承Mc2MatMulTilingCfg来适配matmul_v3的tiling框架。

#### batch_mat_mul_v3 -- 量化BMM引擎

与mat_mul_v3结构类似但增加batch维度迭代:
- iterbatch_tiling/iterbatch_basicapi_tiling: 批量迭代模式
- dasw_block_advanced: 高级ASW块(D维度)
MC2的量化算子(quant_*)使用batch_mat_mul_v3。

#### common/ -- 共享工具

- matmul_common_infershape.cpp/h: matmul的公共infershape逻辑
- cube_util.cpp/h: cube资源检查
- tiling_cache.h: tiling结果缓存(key: shape+dtype hash, 避免重复计算)
- hash.cpp/h + lock.cpp/h: 线程安全的哈希和锁

---

## Phase 2 Cross-Cutting Summary (本轮源码验证后新增)

### CeilDiv/Align的零除行为差异(反模式)

直接阅读源码确认至少5处独立定义，b==0时行为各不相同:

| 位置 | 函数 | b==0行为 |
|------|------|---------|
| common/include/kernel/common_func.h:67 | CeilDiv(a,b) | return T_MAX (numeric_limits) |
| common/include/kernel/util.h:56 | math::Ceil(a,b) | return 0 |
| mc2/common/inc/ops_utils.h:35 | OpsUtils::CeilDiv(a,b) | return a |
| mc2/common/inc/tiling/matmul_formulaic_tiling.h | MathCeil(a,b) | return 0 |
| mc2/common/inc/kernel/reduce_sum_utils.h | AscendC::CeilDiv(a,b) | return a |

调用方必须在调用前保证b!=0，否则不同版本会产生不同结果。
这是潜在的bug来源，尤其是代码从一个模块搬到另一个模块时。

### TilingKey偏移值差异

仓库级(FlashAttention): TILINGKEYOFFSET = 10^19 (uint64_t(10000000000000000000UL))
MC2模块级: MC2_TILINGKEY_OFFSET = 10^18 (uint64_t(1000000000000000000UL))

两者差一个数量级，确保FA和MC2的tiling key空间不冲突。

### ErrorResult的设计精妙之处

mc2_log.h中ErrorResult通过operator重载实现"万能失败值":
```cpp
struct ErrorResult {
  operator bool() const { return false; }
  operator ge::graphStatus() const { return ge::GRAPH_PARAM_INVALID; }
  template <typename T> operator std::unique_ptr<T>() const { return nullptr; }
  template <typename T> operator std::vector<T>() const { return {}; }
  operator std::string() const { return ""; }
  template <typename T> operator T() const { return T(); }
};
```
使GE_ASSERT宏的`return ::ErrorResult()`能匹配任何函数返回类型。
注意: 返回ge::GRAPH_PARAM_INVALID而非GRAPH_FAILED -- 在链式tiling中意味着"跳过"而非"失败"。

### 双定义结构体的实际映射验证

对比kernel/mc2_tiling_struct.h和tiling/mc2_tiling_struct.h:
- 字段完全一致(Mc2Msg/RCSTiling/TileL2Tiling/MC2MatmulV3TilingData)
- tiling版额外有@deprecated标注和REGISTER_TILING_DATA_CLASS注册
- tiling版标注"结构体在后续版本中将不再支持"，说明正在向新tiling数据格式迁移

### fallback.h中的using namespace

fallback.h中`using namespace std; using namespace gert; using namespace ge;`
在头文件中通常是反模式，但fallback.h只被op_graph层cpp文件include，污染范围可控。

### 仓库级 vs MC2级 arch枚举体系

两套完全独立的arch枚举:
1. ArchType (hardware.h): ASCEND_V220/V200/M200 -- FlashAttention的kernel模板
2. NpuArch (platform_ascendc): DAV_2002/DAV_2201/DAV_3510/DAV_5102 -- tiling注册和运行时分发

MC2只使用NpuArch体系。新增MC2算子不需要关心ArchType。

### tiling_util.cpp中的Regbase架构标识

```cpp
const static std::set<NpuArch> regbaseArch = {NpuArch::DAV_3510, NpuArch::DAV_5102};
```
DAV_3510(950)和DAV_5102是"regbase"架构。MC2的arch35路径为这些regbase平台设计。
