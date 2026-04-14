# Phase 2.3: mc2/common/src/ 深度分析

## 文件清单与行数

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| matmul_formulaic_tiling.cpp | 351 | Matmul基本块tiling和L2 cache切分 |
| hccl_formulaic_tiling.cpp | 381 | 通信侧M轴切分(长块/短块配平) |
| matmul_performance.cpp | 273 | Matmul性能模型(cube利用率/时间预测) |
| hccl_performance.cpp | 209 | 通信性能模型(分段拟合/时间预测) |
| hccl_performance_arch35.cpp | 77 | Arch35(950)的通信性能拟合参数 |
| mc2_fit_based_balance_tiling.cpp | 49 | Arch35的拟合配平tiling |
| one_calc_two_comm_tiling.cpp | 372 | MoE双通信域(EP+TP)的E/C轴切分 |
| mc2_tiling_utils.cpp | 444 | 类型转换/rank校验/通信算法选择/groupSize推导 |
| mc2_matmul_tiling_cfg.cpp | 174 | MatmulV3 tiling数据到MC2自定义结构的适配 |
| mc2_common_infershape.cpp | 158 | AllGather/ReduceScatter的infershape公共逻辑 |
| mc2_gen_task_ops_utils.cpp | 260 | KFC任务生成(aicpu+aicore+record+wait) |
| mc2_gen_task_ops_utils_arch35.cpp | 92 | Arch35(950)的CCU Fusion任务生成 |
| mc2_gen_task_ops_utils_arch35_stub.cpp | 27 | Arch35 stub(非arch35平台返回SUCCESS) |
| mc2_a5_gen_task_utils.cpp | 339 | 910A5平台的GenTask(protobuf格式) |
| matmul_all_reduce_gen_task_ops_utils.cpp | 61 | MatmulAllReduce专用GenTask(310P特殊路径) |
| mc2_moe_gen_task_ops_utils.cpp | 346 | MoE系列算子的GenTask(双通信域) |
| mc2_hcom_topo_info.cpp | 283 | HCCL拓扑信息动态加载(dlopen/dlsym) |
| mc2_aclnn_util.cpp | 37 | MXFP scale tensor转置检测 |
| mc2_moe_utils.cpp | 201 | MoE算子公共校验(EP/TP/dtype/shape) |
| moe_tiling_base.cpp | 53 | MoE tiling基类(模板方法) |
| mc2_log.cpp | 237 | tiling数据结构打印(debug日志) |
| context_transfer.cpp | 238 | MRN/IMRN融合算子的context拆解 |

共22个文件, ~4602行代码。

---

## 核心文件深度分析

### 1. matmul_formulaic_tiling.cpp — Matmul基本块Tiling

路径: mc2/common/src/matmul_formulaic_tiling.cpp (351行)

职责: 为MC2融合算子中的Matmul部分计算基本块参数(baseM/baseN/baseK/depth)和L2 Cache切分。

函数列表:
- `CalcBaseBlockTiling()` — 小shape下重新计算baseM/baseN/baseK
- `UpdateDepth()` — 根据L1容量调整depthA1/depthB1
- `DoL2CacheTiling()` — L2 cache切分决策和参数计算
- `SetWeightFormat()` — 设置weight的NZ/ND格式
- `GetCubeTiling(TilingArgs&, TCubeTiling&, TileL2Tiling&)` — 主入口(含L2切分)
- `GetCubeTiling(TilingArgs&, TCubeTiling&)` — 主入口(无L2切分)
- `GetRankSize()` — 获取group的rank数
- `InitBaseBlockTiling()` — 设置平台相关默认基本块
- `GetBaseBlockParm()` — 冗余函数(兼容all_reduce,注释标注"待删除")
- `InitTilingArgs()` — 从外部参数初始化内部参数

核心算法 — baseM/baseN/baseK选择逻辑:

1. 默认值(910B): baseM=128, baseN=256, baseK=64, depthA1=8, depthB1=8
2. 310P特殊值: baseM=256, baseN=256, baseK=64, depthA1=6, depthB1=6; NZ格式另有一组
3. 小shape判定: `usedCoreNum <= aicCoreNum * 0.9` 时进入小shape调整
4. 小shape baseM: 若M < defaultBaseM, 则baseM = AlignUp(M, 16)
5. 小shape baseN: 从三个约束取最小值:
   - baseN1: N均分到core数(以aicCoreNum为参考)
   - baseN2: L0C容量约束: L0C_SIZE / FP32_SIZE / baseM
   - maxBaseN2: L0B容量约束: L0B_SIZE / bDtypeSize / 16(最小K值)
   - 如果有bias, 额外限制baseN <= 256(biasTable只有1024字节)
   - 用tailCoreNum启发式选择更优的baseN
6. baseK: min(L0A/baseM/aDtypeSize, L0B/baseN/bDtypeSize, K), 按64/16对齐

L2 Cache切分:
- 条件: totalSize >= L2Size 且各分量 >= 128MB
- tileSize范围: [15MB, 45MB](20核以上且小shape: [16MB, 64MB])
- 切分M和N两个方向, 要求 mTileBlocks * nTileBlocks >= coreNum
- L2切分只在2卡以上场景生效(MIN_SUPPORT_L2CACHE_DIM = 2)

关键常量:
- C0_SIZE = 16 — Cube单元的最小对齐粒度
- L0_SIZE_DB_ON = 32KB — L0A/L0B开双buffer后的可用容量
- L0C_SIZE_DB_ON = 128KB — L0C开双buffer后的可用容量
- L1_SIZE = 512KB - 32 — L1容量(保留32bit给allreduce mix)
- BASE_K_ALIGN_SIZE = 64 — baseK的对齐粒度
- MAX_BIAS_BASE_BLOCK_N = 256 — bias场景baseN上限(biasTable 1024B / 4B = 256)

值得注意的模式:
- double buffer (DB_ON=2) 是默认启用的，depth初始值为8，step = depth / DB_ON = 4
- dbL0C硬编码为1(关闭L0C双buffer), 代码注释: "需要适配打开"
- depth调整是同步减少A和B的，每次减MIN_DEPTH(=2)，直到总和 <= L1_SIZE

### 2. hccl_formulaic_tiling.cpp — 通信M轴切分

路径: mc2/common/src/hccl_formulaic_tiling.cpp (381行)

职责: 为MC2融合算子计算通信(AllGather/ReduceScatter)侧M轴的切分方案——长块(longTile)和短块(shortTile)的长度和数量。

核心类: FormPartition
核心数据结构: CutResult {shortTileAtBack, totalTileCnt, numLongTile, numShortTile, longTileLen, shortTileLen}

这是MC2计算-通信流水线配平的核心组件。其设计意图:
- 将M轴数据切成若干tile，每轮先做一块的matmul，同时传输另一块的数据
- longTile和shortTile的长度不同，用来匹配matmul和通信的时间差

函数列表(FormPartition类):
- `SetMinLenByMax/SetMinLenByMin()` — 更新最小tile长度约束
- `AlignTileLarge()` — 大tile对齐(>1KB时)
- `AlignTileSmall()` — 小tile对齐与再均衡
- `GenerateInitialPartition()` — 生成初始切分(对齐longTileLen，计算tileCnt)
- `FitTileLengthDiscrete()` — 离散模式切分(多步)
- `NoCutTiling()` — 不切分(整个M作为一块)
- `LargeBackTileCheck()` — 尾块过大时重新均分
- `CalcShortTile()/CalcBackTile()` — 计算尾块参数
- `SmallFrontTileCheck()` — 主块过小时合并(经验值: minTileLen * OPT_TILE_LEN_RATIO)
- `RedistributeTile()` — 剩余长度重分配
- `CommBoundShortAlign()` — 通信bound场景的短块对齐
- `FitTileLengthContinuous()` — 连续模式切分
- `SetShortTileLen()` — 设置短块初始长度
- `MaxTileCntUniform()` — 确保totalTileCnt <= maxTileCnt

函数列表(OneCalcOneCommBase类):
- `EstimateTotalMatmulTime()` — 预估整体matmul时间
- `EstimateTotalCommTime()` — 预估整体通信时间
- `ShortAtEndCalcBoundBalancing()` — 计算bound场景(短块在后)的配平
- `ShortAtFrontCommBoundBalancing()` — 通信bound场景(短块在前)的配平
- `GetTiling()` — 主入口

核心设计意图 — "一算一通"流水线配平:
1. 先用性能模型预测matmul总时间和通信总时间
2. 确定谁是瓶颈(calc-bound vs comm-bound)
3. 如果是calc-bound: 短块在后(shortTileAtBack=true)，短块做matmul，长块做通信
4. 如果是comm-bound: 短块在前(shortTileAtBack=false)，短块做通信，长块做matmul
5. 通过反函数(InverseCommTime/InverseMatmulTime)计算长块的精确长度，使matmul和通信时间匹配

关键常量:
- ALIGN_RATIO_CALC_BOUND = 0.5(计算bound的对齐阈值)
- ALIGN_RATIO_COMM_BOUND(通信bound的对齐阈值)
- SMALL_TILECNT_PAR(小切分数阈值)
- MAX_TILE_CNT(最大切分轮次)
- SOC310_BASE_M(310P平台的baseM)

### 3. matmul_performance.cpp — Matmul性能模型

路径: mc2/common/src/matmul_performance.cpp (273行)

职责: 根据shape和平台参数预测matmul的执行时间，为tiling配平提供决策依据。

核心类: MatmulPerformanceModel

性能预测公式:
```
matmulTime = M * rankTileNum * matmulGradient
matmulGradient = (N * K) / (computesPerCycle * cyclePerMicroSec * coreNum * cubeUtil)
```

cube利用率(cubeUtil)的三级模型(910B):
1. L2 part (memUsage <= 128MB): utilPartL2 = 0.85
2. L2 full (memUsage <= 192MB): utilFullL2 = 0.75
3. Out of L2: utilOutL2 = 0.65

950平台的cube利用率用调和平均数+乘方函数:
```
mnHarmonicMean = (M * rankTileNum * N) / (M * rankTileNum + N)
kExponentiated = min(1.4, pow(K, 0.7) / pow(1280, 0.7))
mnExponentiated = min(1.4, pow(mnHarmonicMean, 0.6) / pow(820, 0.6))
cubeUtil = min(0.95, 0.75 * kExponentiated * mnExponentiated)
```

K对齐惩罚:
- K字节数 % 512 != 0: cubeUtil *= 0.5(半cache line不对齐)
- K字节数 % 256 != 0: cubeUtil *= 0.7(cache line不对齐)
- 910B4: K字节数 % 256 != 0: cubeUtil *= 0.8
- 310P: K % 128 != 0: cubeUtil *= 0.8

拟合参数表(CUBE_CALC_PER_CYCLE_MAP):
- 默认: 4096 FP16 computes/cycle
- "1_1_1_1_2"等(910B, quant, int8, int8, fp16): 8192 computes/cycle
- "4_1_1_1_2"(950, quant): 8192

L2 cache参数表(L2_PARAMETER_MAP):
- 默认: sizePartL2=128, sizeFullL2=192, util=0.85/0.75/0.65
- "3_1_1_1_2"(910B4, quant): 96/96, 0.4/0.4/0.35
- "4_0_2_2_2"(950, fp16): 80/128, 0.95/0.85/0.75
- "4_1_1_1_2"(950, quant): 80/128, 0.95/0.85/0.75

maxTileLen计算: 基于L2 cache利用率变化的边界推导，如果增加一个rankTile的数据会导致L2利用率下降，则限制maxTileLen

线性阈值(GetLinearThresholdLen): matmul在何种数据量以下性能不是线性的
- 非BMM: max(4GB/(N*K), 6MB/(N*K/1024+N)) / rankTileNum, 底线512*baseM
- BMM: max(16GB/(N*K*rankTileNum), baseM) 或 max(24GB/(N*K*rankTileNum), baseM) (取决于K < 2048)
- 核心数适配: mmMinDataSize按 (coreNum/20) 比例缩放

### 4. hccl_performance.cpp — 通信性能模型

路径: mc2/common/src/hccl_performance.cpp (209行)

职责: 预测HCCL通信(AllGather/ReduceScatter/AllReduce)的执行时间。

核心类: HCCLPerformanceModel

性能预测公式 — 三段分段拟合:
```
commDataSize = mSize * commMatrixLen * lookUpTileNum * commDtypeSize (字节)
tmpSize = commDataSize / 1MB

if tmpSize > boundary2:        // 大数据量线性区
    result = gradient * tmpSize + offset
elif tmpSize > boundary1:      // 中等数据量抛物线区
    result = a * tmpSize^2 + b * tmpSize + c
else:                          // 小数据量常数区
    result = timeToSizeBoundary1
```

反函数InverseCommTime: 从目标时间反推mSize，用于流水线配平

拟合参数表(FITTING_PARAMETER_MAP):
- 默认(fullmesh): boundary=[64B/KB, 8MB], gradient=13.58, offset=61.5
- "1_3"(310P ring4): boundary=[64B/KB, 64B/KB], gradient=72.04, offset=6.69(纯线性)
- "1_4"(310P ring2): 含抛物线参数
- "2_1"(910_93): gradient=11.38, offset=248.15
- "4_0"(950 fullmesh): boundary=[32KB, 1.371MB], gradient=2.60, offset=5.89

commTimeFactor的调整:
- fullMesh: factor = 1.0 / (8 / rankDim) = rankDim / 8
- ring: factor = 1.0 / (currentRatio / fittingRatio), 其中ratio = (rankDim-1) / rankDim
- localReduce: factor /= (1 + LOCAL_REDUCE_FACTOR * rankDim / 8)

rankTileNum:
- AllGather: rankDim - 1
- ReduceScatter: rankDim
- AllReduce: 1

maxStepSize:
- DoubleRing + ReduceScatter: rankDim
- DoubleRing + AllGather: rankDim - 1
- 其他: 1

### 5. hccl_performance_arch35.cpp — Arch35通信性能

路径: mc2/common/src/hccl_performance_arch35.cpp (77行)

职责: 950平台(arch35/DAV_3510)的通信性能拟合参数。

继承: HCCLPerformanceArch35 : HCCLPerformanceModel

key格式: `socVersion_kernelType_topoType_rankDim`
- 默认("0_0_0_0"): 与主表的"4_0"相同参数
- "4_1_0_4"(950, AllGather, STANDARD_CARD, 4p): gradient=3.3586, offset=24.9879(纯线性)

AllGather特化: lookUpTileNum = rankDim - 1, commTimeFactor = 1

InverseCommTime: 与基类相同但增加了AllReduce + 线性反函数的特殊处理

### 6. mc2_fit_based_balance_tiling.cpp — 拟合配平Tiling

路径: mc2/common/src/mc2_fit_based_balance_tiling.cpp (49行)

职责: Arch35平台的计算-通信流水线配平tiling。相比OneCalcOneCommBase更简洁，直接用性能模型做配平。

核心类: Mc2FitBasedBalanceTiling

GetTiling()流程:
1. EstimateMMCommTime() — 虚函数，由子类实现
2. SetShortTileLen() — 虚函数，由子类实现
3. 若M太小(< 2*shortTileLen)，不切分
4. SetLongTileLen() — 根据短块时间反推长块长度
5. AdjustLongShortTileLen() — 虚函数，由子类微调

SetLongTileLen()的配平逻辑:
- shortTileAtBack(计算bound): 短块做matmul，长块做通信 → longTileLen = InverseCommTime(matmulTime(shortTileLen))
- shortTileAtFront(通信bound): 短块做通信，长块做matmul → longTileLen = InverseMatmulTime(commTime(shortTileLen))

MAX_TILE_CNT约束: 由于FFTS队列大小限制，最多切16轮

### 7. mc2_tiling_utils.cpp — Tiling工具函数

路径: mc2/common/src/mc2_tiling_utils.cpp (444行)

核心函数:

类型转换:
- `ConvertGeTypeToMmType()` — ge::DataType → matmul_tiling::DataType (bf16/fp16/fp32/hifloat8/fp8_e4m3/fp8_e5m2)
- `ConvertMmTypeToGeType()` — 反向转换
- `GetDataTypeSize()` — ge::DataType → 字节数 (bf16=2, fp16=2, fp32=4, 8bit类型=1)
- `ConvertGeTypeToHcclType()` — ge::DataType → HcclDataType

通信算法选择 `Mc2GetCommAlgo()`:
- arch20(310P): 固定FULL_MESH
- 2卡: 固定FULL_MESH
- rankDim <= 0: 错误返回
- 查询拓扑: TryGetGroupTopoType → commSets
  - COMM_MESH: 只能FULL_MESH
  - 其他: 默认DOUBLE_RING (需M为偶数且能被rankDim整除)
- 环境变量覆盖: ASCEND_MC2_DEBUG_COMM_ALG

RankSize校验 `CheckRankSize()`:
- DAV_2002(310P): 支持 {1, 2, 4}
- DAV_2201(910B): 支持 {1, 2, 4, 8}
- DAV_3510(950): 支持 {1, 2, 4, 8, 16, 32, 64}

GroupSize推导 `InferGroupSize()`:
- 从x1/x2的shape和scale的shape推导groupSizeM/N/K
- MXFP场景: scaleShape是3维[m, k/2, 2]
- 非MXFP: scaleShape是2维

通用参数检查 `CommonParamCheck()`:
- 校验A/B shape维度为2, format一致且为ND

debug环境变量:
- ASCEND_MC2_DEBUG_MODE: 0=正常, 1=只做计算不通信
- ASCEND_MC2_DEBUG_COMM_ALG: 强制指定通信算法
- ASCEND_MC2_DEBUG_STEP_SIZE: 强制指定步长
- HCCL_BUFFSIZE: 窗口大小(默认200MB)

MatmulV3参数适配:
- `UpdateMatmulV3Args()` — 填充Mc2MatMulV3Args结构
- `GetMatmulV3PriorityPolicy()` — arch35只有BASE策略

### 8. mc2_gen_task_ops_utils.cpp — KFC任务生成

路径: mc2/common/src/mc2_gen_task_ops_utils.cpp (260行)

职责: 为MC2算子生成GPU执行所需的KFC(Kernel Function Call)任务链。

任务链结构: [wait_task] → [aicpu_task] → [record_task] → [aicore_task]
- wait_task: 在attach stream上等待aicore完成
- aicpu_task: 通过KFC调用libccl_kernel.so的RunAicpuKfcSrvLaunch
- record_task: 在main stream上记录aicpu完成信号
- aicore_task: 实际的Cube计算

HcclCommParamDescTemp结构:
```
version: 4 bits (保留,固定1)
groupNum: 4 bits (group数,默认1)
hasFfts: 1 bit (aicpu不用)
tilingOff: 7 bits (tiling在args中的index)
isDyn: 48 bits (默认0)
```

aicpu参数顺序: {desc}{hcom}{INPUT0}...{INPUTN}{OUTPUT0}...{OUTPUTN}{WORKSPACE}{TILING}

aicore hidden input插入: 在IrInput之前插入kHcom类型的hidden input

平台判定:
- `IsTargetPlatformSocVersion()` — 按Short_SoC_version字符串匹配
- `IsTargetPlatformNpuArch()` — 按NpuArch数值匹配

### 9. mc2_common_infershape.cpp — 公共InferShape

路径: mc2/common/src/mc2_common_infershape.cpp (158行)

AllGather InferShape:
- y.shape = [M * rankSize, N]
- gatherOut.shape = isGatherOut ? [M * rankSize, K] : [0]

ReduceScatter InferShape:
- y.shape = [M / rankSize, N]

公共逻辑:
- 从attrs获取isTransA/isTransB/rankSize/group
- rankSize优先从attr获取，<=0时从拓扑信息查询
- 动态shape(M=-1)时不做rank倍增/缩减(rankSize设为1)
- K=0时报错

### 10. mc2_matmul_tiling_cfg.cpp — MatmulV3 Tiling适配

路径: mc2/common/src/mc2_matmul_tiling_cfg.cpp (174行)

职责: 将第三方MatmulV3 tiling引擎的输出适配到MC2自定义的Mc2MatMulV3TilingData结构。

核心类: Mc2MatmulTilingCfg

Update()流程:
1. DealBaseBlock() — 如果baseMLimit > 0且matmul给的baseM > baseMLimit, 截断为baseMLimit并对齐到16
2. SetMMTilingData() — 逐字段拷贝TCubeTiling的约40个字段
3. SetTailCntAndType() — 重算mTailCnt/nTailCnt(考虑MC2的多卡多片场景)

mTailCnt/nTailCnt重算逻辑:
```
mCnt = ceil(baseMLimit / baseM) * rankDim * commCnt
nCnt = ceil(N / baseN)
mnCnt = mCnt * nCnt
tailCnt = mnCnt % coreNum
// 贪心扩展: 只要 mTailCnt * nTailCnt * tailCnt <= coreNum
while ((mTailCnt+1) * nTailCnt * tailCnt <= coreNum) {
    mTailCnt++
    if (mTailCnt * (nTailCnt+1) * tailCnt <= coreNum) nTailCnt++
}
```
这是为了在最后一轮计算中更好地利用空闲core。

### 11. one_calc_two_comm_tiling.cpp — MoE双通信域切分

路径: mc2/common/src/one_calc_two_comm_tiling.cpp (372行)

职责: MoE场景下同时存在EP(Expert Parallel)和TP(Tensor Parallel)两个通信域，需要在E轴和C轴上做切分。

核心类:
- OneCalcTwoCommBase — E轴和C轴切分
- OneCalcTwoCommShardHBase — H维度分片场景

切分维度说明:
- E轴(Expert): 对应batchSize = E/Ep
- C轴(Capacity): 对应mValue
- H轴(Hidden): 对应kValue

E轴切分约束:
- E轴要求均匀切分(eSize % tmpCnt == 0)
- localCutE: 本地通信(TP域)的E轴切分
- cutE: 非本地通信(EP域)的E轴切分
- MAX_TILE_CNT_TWO_COMM: 总切分轮次上限

C轴切分:
- 只在E轴已切到最小(eTileLen==1)且还有切分预算时才切C
- C轴使用FitTileLengthContinuous做长短块配平

性能估算:
- Local AG/RS: tile_e_local * C * (H/Tp) * Tp * dTypeSize
- Non-local AG/RS: tile_e * ((Ep-1) * tile_c) * (H/Tp) * Tp * dTypeSize
- A2A: Ep * tile_c * (H/Tp) * dTypeSize

ShardH模式:
- 额外支持local和non-local两个branch的独立切分
- 通过AsignMaxCutNumForBranches分配切分预算
- 支持不均匀切分(unbalanceRatio控制长短比)

### 12. context_transfer.cpp — 融合算子Context拆解

路径: mc2/common/src/context_transfer.cpp (238行)

职责: 将MRN(MatmulReduceScatter+AddRmsNorm)、IMRN(Inplace MRN)等融合算子的TilingContext拆解为MMR(MatmulReduceScatter)和ARN(AddRmsNorm)两个子算子的Context。

适配场景:
- MRN → MMR + ARN
- IMRN → MMR + ARN (inplace场景)
- MMR独立 → MMR

MMRCtxInfo包含:
- attrs: group, reduceOp, isTransA, isTransB, commTurn, antiquantGroupSize, groupSize, yDtype, commQuantMode
- inputs: x1, x2, bias, x3, antiquant_scale, antiquant_offset, dequant_scale, pertoken_scale, comm_quant_scale_1, comm_quant_scale_2

ARNCtxInfo包含:
- attrs: epsilon
- inputs: x2(residual), gamma
- outputs: x, y

注意: MRN中的MMR没有x3(当前不支持融合带x3的mmr), optional input缺失时需要跳过index。

### 13. mc2_hcom_topo_info.cpp — HCCL拓扑信息动态加载

路径: mc2/common/src/mc2_hcom_topo_info.cpp (283行)

职责: 通过dlopen动态加载libhccl.so，获取通信拓扑信息(rank size, topo type, ccl buffer size)。

双编译路径:
- BUILD_OPEN_PROJECT: 使用HcomGetRankSizeEx/HcomGetL0TopoTypeEx(新接口)
- 非BUILD_OPEN_PROJECT: 使用CommGetNetLayers/CommGetInstTopoTypeByNetLayer/CommGetInstSizeByNetLayer + HcomTopoInfo

单例模式: MC2HcomTopology::GetInstance()返回static对象

加载路径:
- x86_64: ${ASCEND_HOME_PATH}/x86_64-linux/lib64/libhccl.so
- aarch64: ${ASCEND_HOME_PATH}/aarch64-linux/lib64/libhccl.so

---

## new_mc2_mm/ 目录分析

### 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| new_mc2_matmul_tiling_cfg.h | 57 | NewMc2MatmulTilingCfg头文件 |
| new_mc2_matmul_tiling_cfg.cpp | 165 | 新版MatmulV3 tiling适配(arch35) |
| tiling/new_mc2_tiling_utils.h | 40 | NewUpdateMatmulV3Args声明 |
| new_mc2_tiling_utils.cpp | 81 | 新版tiling工具(arch35) |
| kernel/mc2_mat_mul_asw_block.h | 240 | ASW(自适应滑窗)块管理 |
| kernel/mc2_mat_mul_asw_kernel.h | 127 | ASW Matmul Kernel |
| kernel/mc2_quant_batch_matmul.h | 243 | 量化BMM + ASW |
| kernel/mc2_quant_batch_matmul_asw_block.h | 325 | 量化BMM的ASW块管理 |

### NewMc2MatmulTilingCfg vs Mc2MatmulTilingCfg

新版与旧版的核心区别:
1. 新版使用setter方法(set_xxx)而非直接字段赋值，表明底层TilingData变成了序列化结构
2. 新版的baseMLimit_和tailCnt计算使用uint64_t而非int64_t/int32_t(修复了可能的类型安全问题)
3. 新版继承自Mc2MatMulTilingCfg(arch35 matmul tiling框架的基类)
4. 新版的DealBaseBlock不再做CeilAlign，直接设置baseMLimit

### ASW(自适应滑窗)块实现

ASW = Adaptive Sliding Window，MC2 Matmul Kernel中的核心调度算法。

关键概念:
- DEVICE_NUM = 64 — 最大group内卡数
- SLIDING_WINDOW_LEN = 4 — 滑窗M方向大小

MC2MatmulAswBlockDerive核心逻辑:

Init(): 从tiling数据读取baseM/baseN/nCnt/mTailCnt/nTailCnt

InitForMC2(): MC2特化初始化
- rankM = isGather ? cfg.rankM : cfg.rankM / rankDim (AllGather看全量M，RS看单卡M)
- headSliceM = (rankM - tailM * tailCnt) / tileCnt (头块的M)
- curSliceM = isTail ? tailM : headSliceM
- mSliceCnt = ceil(curSliceM / baseM) (当前slice的基本块数)
- mCnt = mSliceCnt * calRankNum (总M方向块数)
- mainWindow = min(4, mCnt) (主滑窗的M方向块数)
- mainRow = mCnt / mainWindow - 1 (主滑窗数)
- tailWindow = mCnt - mainRow * mainWindow (尾滑窗块数)

UpdateBasicIndex(roundIdx, blockIdx): 滑窗索引计算
- index = blockIdx + roundIdx * usedCoreNum - preCoreNum
- rowIdx = index / nCnt / mainWindow
- 主窗口区: mCntIndex = rowIdx * mainWindow + index % mainWindow
             nCntIndex = (index / mainWindow) % nCnt
- 尾窗口区: 从tailIndex计算
- 蛇形扫描: 奇数行nCntIndex取反(N方向反转)

CalcGMOffset(): 全局内存偏移
- offsetA = offsetsA[mc2MIdx] + mc2MRest * baseM * rankK + mSplitAddrOffset * rankK
- offsetB = nCntIndex * baseN (不转置) 或 nCntIndex * baseN * Ka (转置)
- offsetC = offsetsC[mc2MIdx] + nOffset + mOffset * N

UpdateOffset(idx, isTail): 更新多卡偏移数组
- 遍历rankDim个卡，跳过自己(AllGather时)
- offsetsA[cnt] = i * rankMK + offsetHeadAndTailK
- offsetsC[cnt] = i * rankMN + offsetHeadAndTailN

Process(): 执行Matmul
- SetAtomicNone
- SetHF32(isHf32, 1)
- set_ctrl(sbitset1(get_ctrl(), 51)) — fixp使用n搬出，cube和fixp并行
- 多轮迭代: 每轮遍历usedCoreNum个block
- 尾轮处理: tailSplit(mBaseSplitCnt * nBaseSplitCnt个子块)
- 结束: preCoreNum = totalCnt % usedCoreNum(传递给下一轮)

### 量化BMM ASW

QuantBatchMatmulAswBlock相比非量化版本的额外处理:
1. scale offset计算: offsetPerTokenScale/offsetScale
2. MXFP场景: scaleK = Align(CeilDiv(Ka, MXFP_GROUP_SIZE), MXFP_MULTI_BASE_SIZE)
3. deqScale处理: perTensor(scalar) / perToken(vector) / doubleScale(两个scalar相乘)
4. deqScale精度: fixpipe只取高19位(uint32Scale & 0xFFFFE000)
5. ScaleType模板特化: IsMxType<ScaleType>()控制是否使用MatmulWithScalePolicy

---

## 关键编码模式总结

### 1. 性能模型驱动的Tiling
MC2的tiling不是简单的shape对齐，而是建立在matmul/通信时间预测模型之上的配平优化。核心链路:
`性能模型参数表 → cube利用率 → matmulGradient → matmulTime/commTime → 反函数 → longTileLen/shortTileLen`

### 2. 参数表查表模式
性能相关参数都通过 `socType_calcType_dtypeA_dtypeB_dtypeC` 格式的key查map获取。命中时打OP_LOGW("HIT")。

### 3. 多平台条件分发
- 310P: 专用常量(BASE_BLOCK_M_L2CACHE=256)、专用拟合公式(1/(1.5+1.25*1024/K))
- 910B: 标准L2 cache三级利用率模型
- 950: 调和平均数+乘方的高精度模型

### 4. 长块短块配平
核心抽象: CutResult{longTileLen, shortTileLen, numLongTile, numShortTile, shortTileAtBack}
- shortTileAtBack=true: 计算瓶颈，短块做matmul(短→快→等通信)
- shortTileAtBack=false: 通信瓶颈，短块做通信(短→快→等计算)

### 5. 滑窗(ASW)调度
kernel层的块调度使用4块宽的滑窗+蛇形扫描，目的是提高L2 cache局部性(相邻块共享B矩阵数据)。

### 6. 新旧版tiling的过渡
mc2_matmul_tiling_cfg.cpp(旧): 直接字段赋值
new_mc2_matmul_tiling_cfg.cpp(新): setter方法
体现了从plain struct向序列化TilingData的架构迁移。

### 7. 环境变量Debug机制
三个环境变量(ASCEND_MC2_DEBUG_MODE/COMM_ALG/STEP_SIZE)提供运行时调试能力，可在不重编的情况下切换通信算法或禁用通信。
