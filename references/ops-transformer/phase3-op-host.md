# Phase 3: op_host Layer Analysis

## 3.1.1 matmul_all_reduce/op_host/ -- 全量分析

### 文件结构 (27个C++文件)

```
op_host/
  matmul_all_reduce_def.cpp        -- 算子定义(OpDef): 输入/输出/属性声明
  matmul_all_reduce_infershape.cpp -- InferShape + InferDataType
  op_tiling/
    matmul_all_reduce_tiling.cpp   -- tiling入口: 路由到arch级TilingRegistry
    matmul_all_reduce_tiling_base.h  -- 基类: MatmulAllReduceTilingBase (继承TilingBaseClass)
    matmul_all_reduce_tiling_base.cpp -- 基类实现(~800行): 公共tiling逻辑
    all_reduce_formulaic_tiling.h    -- 公式化tiling: MMPlusAllReduce, MMPlusQuantAllReduce
    all_reduce_formulaic_tiling.cpp  -- 公式化tiling实现
    arch31/ (4对 .h+.cpp)           -- Ascend310P: 4个tiling变体
      matmul_all_reduce_tiling_310_general  -- 通用310 (priority=3, fallback)
      unquant_matmul_all_reduce_tiling_310  -- 非量化310 MatmulV3路径 (priority=2)
      quant_matmul_all_reduce_tiling_310_general -- A8W8全量化 (priority=1)
      weight_quant_matmul_all_reduce_tiling_310p -- A16W8权重量化 (priority=0, 最高)
    arch32/ (3对 .h+.cpp)           -- Ascend910B: 3个tiling变体
      matmul_all_reduce_tiling_910  -- FP A16W16 (priority=2, fallback)
      quant_matmul_all_reduce_tiling -- A8W8全量化 (priority=0, 最高)
      weight_quant_matmul_all_reduce_tiling -- A16W8权重量化 (priority=1)
    arch35/ (3对 .h+.cpp)           -- Ascend950: 3个tiling变体
      matmul_all_reduce_tiling_950  -- FP A16W16 (priority=2, fallback)
      quant_matmul_all_reduce_tiling_950  -- A8W8全量化 (priority=0, 最高)
      weight_quant_matmul_all_reduce_tiling_950 -- A16W8/A16W4权重量化 (priority=1)
```

### 1. 架构总览

MatmulAllReduce op_host层的核心架构是一个两级继承+两级路由体系:

继承层级: TilingBaseClass -> MatmulAllReduceTilingBase -> 场景+arch专用子类
路由层级: MatmulAllReduceTilingFunc(NpuArch路由) -> TilingRegistryArch/TilingRegistryNew(priority链式选择)

路由逻辑(matmul_all_reduce_tiling.cpp:29-38):
```cpp
if (npuArch == NpuArch::DAV_3510) {
    return TilingRegistryArch::GetInstance().DoTilingImpl(context);  // arch35注册
}
return TilingRegistryNew::GetInstance().DoTilingImpl(context);       // SocVersion注册
```

arch35使用REGISTER_TILING_TEMPLATE_WITH_ARCH宏注册，arch31/32使用REGISTER_TILING_TEMPLATE_WITH_SOCVERSION宏注册。两套注册表独立运行。

### 2. 注册优先级体系

完整的tiling类注册及其优先级(priority越小越优先):

arch35 (REGISTER_TILING_TEMPLATE_WITH_ARCH, NpuArch::DAV_3510):
- priority=0: QuantMatmulAllReduceTilingA5 (A8W8全量化)
- priority=1: WeightQuantMatmulAllReduceTilingA5 (A16W8/A16W4权重量化)
- priority=2: MatmulAllReduceTilingA5 (A16W16 FP fallback)

arch32 (REGISTER_TILING_TEMPLATE_WITH_SOCVERSION, ASCEND910B):
- priority=0: QuantMatmulAllReduceTiling (A8W8全量化)
- priority=1: WeightQuantMatmulAllReduceTiling (A16W8权重量化)
- priority=2: MatmulAllReduceTiling910 (A16W16 FP fallback)

arch31 (REGISTER_TILING_TEMPLATE_WITH_SOCVERSION, ASCEND310P):
- priority=0: WeightQuantMatmulAllReduceTiling310P (A16W8权重量化)
- priority=1: QuantMatmulAllReduceTiling310General (A8W8全量化)
- priority=2: UnQuantMatmulAllReduceTiling310 (MatmulV3非量化)
- priority=3: MatmulAllReduceTiling310General (legacy fallback)

设计意图: 量化场景(Quant/WeightQuant)优先尝试，IsCapable()检查输入是否满足量化条件；不满足返回false(GRAPH_PARAM_INVALID)让框架继续尝试下一个。A16W16类总是最低优先级作为兜底。

观察: arch31有4个注册类(最多)，因为310P把unquant(MatmulV3路径)和general(legacy路径)拆成了两个类。

### 3. Op定义模式 (matmul_all_reduce_def.cpp)

核心模式: 继承OpDef，构造函数中声明全部输入/输出/属性。

输入列表(按op_mc2.h中MC2InputIdx枚举顺序):
- x1 (REQUIRED): 矩阵A
- x2 (REQUIRED): 矩阵B
- bias (OPTIONAL)
- x3 (OPTIONAL): 残差加
- antiquant_scale (OPTIONAL): 反量化缩放
- antiquant_offset (OPTIONAL): 反量化偏移
- dequant_scale (OPTIONAL): 解量化缩放
- pertoken_scale (OPTIONAL): per-token缩放
- comm_quant_scale_1 (OPTIONAL): 通信量化缩放1
- comm_quant_scale_2 (OPTIONAL): 通信量化缩放2

输出: y (REQUIRED)

关键特征:
- 数据类型枚举极其庞大: x1有65种dtype组合(覆盖fp16/bf16/int8/fp8/hif8/mxfp4/mxfp8)
- 格式: 绝大多数为FORMAT_ND，仅x2在int8量化场景支持FORMAT_FRACTAL_NZ(6种组合)
- 每个输入都有.Format()和.UnknownShapeFormat()两组格式声明(后者用于动态shape)
- 全部使用.IgnoreContiguous()标记
- 注册宏: `OP_ADD(MatmulAllReduce);`

属性(通过MmAllReduceAttrIdx枚举索引):
- group, reduce_op, is_trans_a, is_trans_b, comm_turn, antiquant_group_size, dtype_y, comm_quant_mode, group_size

### 4. InferShape模式 (matmul_all_reduce_infershape.cpp)

结构: 两个静态函数 + IMPL_OP_INFERSHAPE注册宏

InferShapeForMatmulAllReduce:
1. 通过MC2InputIdx枚举获取输入shape
2. 通过MmAllReduceAttrIdx枚举获取属性
3. 约束: x1维度2或3，x2维度必须2，transA不支持
4. K维度一致性检查(跳过动态shape: k=-1/0/1)
5. antiquant_scale shape校验(CheckScaleShape)
6. 输出shape: [s,m,n]或[m,n]

InferDataTypeForMC2:
1. 优先使用y_dtype属性
2. int8+int8+bf16_dequant -> bf16; int8+int8+other -> fp16
3. fallback到原始y_type

注册: `IMPL_OP_INFERSHAPE(MatmulAllReduce).InferShape(...).InferDataType(...);`

关键发现:
- AllReduce不改变M轴大小(与AllGather/ReduceScatter不同)
- 使用OPS_CHECK + CUBE_INNER_ERR_REPORT组合报错
- 属性访问使用枚举索引(如MmAllReduceAttrIdx::K_TRANS_X1)而非字符串

### 5. Tiling基类 (MatmulAllReduceTilingBase)

继承TilingBaseClass，是所有arch-specific tiling类的公共基类。

构造函数三种形式(体现融合算子复用):
1. `(context)` -- 独立使用，自有mmrCtxInfo和tilingData
2. `(context, mmrCtxInfo*)` -- 被融合算子使用，外部传入context信息
3. `(context, mmrCtxInfo*, tilingData*)` -- context和tiling数据都外部提供

"引用绑定+实体备份"惯用法:
```cpp
MMRCtxInfo mmrCtxInfoSelf_{};      // 实体(独立使用时的数据存储)
MMRCtxInfo& mmrCtxInfo_;           // 引用(统一访问接口)
```
独立构造时引用绑定自有实体；复合算子构造时引用绑定外部数据。
注释原文: "tilingdata作为类的成员随构造函数实例化，使用引用来统一数据访问接口，同时用实体对象来管理数据的生命周期"

模板方法override:
- GetPlatformInfo(): socVersion/npuArch/aicCoreNum/ubSize/libApiWorkSpaceSize + supportL0c2Out检测 + rankSize平台支持检查
- GetShapeAttrsInfo(): ContextTransfer::AssembleMMRCtxInfoFromMMRCtx + AnalyzeShapeAttr
- DoLibApiTiling(): 空实现(子类按需override)
- GetWorkspaceSize(): libApiWorkSpaceSize + biasLen + softSyncSize + mmOutInt32Len + gmcFloat(950专用)
- PostTiling(): memcpy_s tilingData到rawTilingData + SetBlockDim + 8字节对齐检查

子类必须覆盖: IsCapable(), DoOpTiling(), GetTilingKey()

Virtual MutableXxxData()方法族:
- MutableRCSTilingData() -> Mc2Tiling::RCSTiling&
- MutableMc2MsgData() -> Mc2Tiling::Mc2Msg&
- MutableTCubeTileTilingData() -> TCubeTiling&
- MutableTCubeTailTilingData() -> TCubeTiling&

子类override这些方法指向自己更大的专用tilingData中的对应字段，让基类代码可以统一操作不同arch的tilingData。

核心tiling方法:
- DoRCSTiling(): 填充RCSTiling结构(rankDim, transpose, commtype, rankM/N/K, isAdd, commQuantScale等)
- DoSplitMTiling(): M轴公式化切分入口(MMPlusAllReduce/MMPlusQuantAllReduce)
- DoAllReduceTiling(): 填充Mc2Msg通信参数(sendOff/recvOff/sendCnt/recvCnt/总轮次/buffer类型等)
- DoMatmulTiling(): matmul_tiling::MultiCoreMatmulTiling + MatmulFormulaicTiling::GetCubeTiling
- DoL2CacheTiling(): L2 cache tile参数计算
- SetQuantData()/SetAntiQuantData()/SetCommQuantScale(): 量化参数设置
- CheckInput()/CheckA16W16()/CheckA8W8()/CheckA16W8(): 输入校验链

典型DoOpTiling子类实现流程(MatmulAllReduceTilingA5为例):
1. CheckA16W16() -- 校验非量化场景输入约束
2. CheckInput() -- 通用校验 + 子类特有校验
3. SetMc2Hcomm() -- 获取通信tiling(arch35特有)
4. DoRCSTiling() -- RCS参数填充
5. DoSplitMTiling() -- M轴公式化切分
6. Do910Tiling()/DoMatmulTiling() -- 矩阵乘tiling计算
7. DoAllReduceTiling() -- 通信参数填充

### 6. M轴公式化切分 (all_reduce_formulaic_tiling)

MMPlusAllReduce和MMPlusQuantAllReduce继承自OneCalcOneCommBase(mc2/3rd中的性能模型框架)。

核心算法:
1. 计算totalMatmulTime和totalCommTime
2. 判断计算bound(shortTileAtBack=true)还是通信bound(shortTileAtBack=false)
3. 根据通算比选择切分策略:
   - 计算bound: ShortAtEndCalcBoundBalancing -- 尾部短块填充通信等待
   - 通信bound: ShortAtFrontCommBoundBalancing -- 首部短块让通信提前启动
4. GenerateInitialPartition -> FitTileLengthContinuous -- 生成并调整

MMPlusAllReduce(A16W16):
- arch35专门CommTimeFactor: SetCommTimeFactorForA5() (ALLREDUCE_COMMTIME_FACTOR + rank比例调整)
- 310P非对齐场景: 强制shortTileAtBack=true(首地址非对齐时通信很慢)
- 通算均衡且shape较小时鼓励切分(MinLen减半)

MMPlusQuantAllReduce(A8W8):
- 通信因子: QUANT_COMM_GROWTH_FACTOR_SOC910B=0.6, all2all因子=0.5
- 三分支:
  - mm < a2a=ag: ShortAtFrontCommBoundBalancing
  - a2a=ag < mm < (a2a+ag): 均匀切分(UniformCutSetShort)
  - (a2a+ag) < mm: ShortAtEndCalcBoundBalancing
- SmallShortCheck: 规避"大短块"(remainLen > shortTileLen * 1.25时调整)
- SMALL_BACKTILE_RATIO=0.25(短块尽量短，给aiv留时间)

### 7. ContextTransfer模式

MMRCtxInfo是扁平化的context快照结构(context_transfer.h)，包含:
- 10个输入tensor的CompileTimeTensorDesc和StorageShape指针
- 1个输出tensor的desc和shape指针
- group/reduceOp/isTransA/isTransB/commTurn等属性指针

设计意图(基类注释): "根据ctx填充mmrCtxInfo，这个函数结束之后从context读的操作应该都从mmrCtxInfo读"

好处:
1. 避免多处重复调用context getter
2. null检查集中处理
3. 复合算子(MatmulAllReduceAddRmsNorm)从不同context索引组装同一MMRCtxInfo

类比: ARNCtxInfo(AddResNorm), MRNCtxInfo(MatMulAllReduceAddResNorm = MMR + ARN), IMRNCtxInfo也使用相同模式。

### 8. Arch差异化设计

| 维度 | arch31 (310P) | arch32 (910B) | arch35 (950/A5) |
|------|---------------|---------------|-----------------|
| 注册方式 | WITH_SOCVERSION | WITH_SOCVERSION | WITH_ARCH |
| tiling类数 | 4 | 3 | 3 |
| MatMul库 | MatmulV3 + NZ格式强制 | MatmulV3 + MultiCoreMatmulTiling | MatmulV3 + ASW + RegBase |
| 通信tiling | KFC消息手动填充 | KFC消息手动填充 | Mc2CcTilingConfig新接口 |
| PostTiling | 标准 | 标准 | +SetScheduleMode(1)独占全核 |
| 量化约束 | transB=true, K%32=0, N%16=0, NZ格式 | 宽松 | 宽松 |
| Workspace | 标准 | 标准 | +rankM*rankN*outputDtypeSize(mm输出) |
| TilingData | MatmulAllReduceTilingData | MatmulAllReduce910TilingData | MatmulAllReduce910TilingDataA5 |
| TransferHelper | 内部类(class-in-class) | 独立友元类 | 独立友元类 |

### 9. TransferHelper模式 (Adapter Pattern)

每个arch-specific量化子类都有配套TransferHelper，桥接MC2参数到底层matmul tiling库:

arch35:
- QuantTilingTransferHelperA5 : Mc2AdaptiveSlidingWindowTiling
- WeightQuantTilingTransferHelperA5 : Mc2WeightQuantBatchMatmulV2RegBase
- WeightQuantAsTilingTransferHelper : Mc2WeightQuantBatchMatmulV2TilingAS
- TilingTransferHelper(910B) : mc2_matmul_v3::Mc2MatmulV3BaseTiling

arch32:
- TilingTransferHelper : mc2_matmul_v3::Mc2MatmulV3BaseTiling
- QuantTilingTransferHelper : Mc2QuantBatchMatmulV3Tiling
- WeightQuantTilingTransferHelper : Mc2WeightQuantBatchMatmulV2TilingCustom

每个TransferHelper至少override:
1. GetShapeAttrsInfo() -- 从MC2 args_/mmrCtxInfo_提取参数填入底层inputParams_
2. PostTiling() -- 从底层获取workspaceSize回填MC2 myWorkSpaceSize_

### 10. 通信buffer类型选择 (setUseBufferType)

严格条件链:
1. NpuArch == DAV_2002 -> OUTPUT
2. debugMode == MC2_DEBUG_ONLY_AICPU -> OUTPUT
3. reuseMode == 0 -> OUTPUT
4. isKZero_ -> OUTPUT
5. tileSendOff+tailSendOff >= maxWindowSize(默认200MB) -> OUTPUT
6. 否则 -> WINDOW_IN

低bit通信时M需pad到rankDim整数倍。maxWindowSize可通过HCCL_BUFFSIZE环境变量调整。

### 11. 反模式和维护风险

1. dtype组合爆炸: def.cpp中65种dtype组合 x 10个输入 = 数百行重复代码。
   任何新增dtype需在所有输入中同步修改。(matmul_all_reduce_def.cpp全文)

2. TransferHelper逐字段拷贝: weight_quant_matmul_all_reduce_tiling_950.h:128-165
   约30行data_.matmulTiling.X = tilingData_->matmulTiling.X的逐字段赋值。
   底层tiling库新增字段时此处可能遗漏。

3. 疑似copy-paste bug: weight_quant_matmul_all_reduce_tiling_950.h:163和239行:
   `data_.matmulTiling.shareL1Size = tilingData_->matmulTiling.shareMode;`
   左值shareL1Size但右值shareMode，几乎确定是复制粘贴错误。

4. getenv直接使用: matmul_all_reduce_tiling_base.cpp:201-210
   `getenv(HCCL_BUFFSIZE)` + try-catch stoi，C风格配置获取，无线程安全保证。

5. D_TYPE_MAP/D_TYPE_SIZE_MAP/HCCL_DATA_TYPE三重类型映射:
   新增dtype需同步更新3个map(matmul_all_reduce_tiling_base.cpp:64-87)。

### 12. 值得注意的模式

1. SetScheduleMode(1): 仅arch35 PostTiling调用。注释: "独占全核，设置以后会让所有核空闲以后才启动，有多核同步指令需要设置避免出现网络挂死"。950架构特有。

2. 空tensor(K=0)特殊路径: isKZero_标记控制，跳过matmul tiling，每个子类要实现DoEmptyTensorTiling()。

3. PostTiling 8字节对齐检查: `tilingDataSize % sizeof(uint64_t) != 0`时报错，硬件要求。

4. 310P内联TransferHelper: arch31的3个量化类都把TransferHelper定义为class内嵌class，而910B/950定义为独立友元类。310P文件更大但自包含。

5. 双Registry路由: arch35走TilingRegistryArch，其他走TilingRegistryNew。新增arch需修改tiling入口函数。

6. TilingParse空实现: MatmulAllReduceCompileInfo是空结构体，TilingParseForMatmulAllReduce直接返回SUCCESS。MC2算子不需要编译信息解析(所有信息runtime可用)。

---

## 3.1.2 all_gather_matmul/op_host/ -- 全量分析

### 文件结构 (5个C++文件)

```
op_host/
  all_gather_matmul_def.cpp         -- 算子定义(OpDef)
  all_gather_matmul_infershape.cpp  -- InferShape(委托给公共函数)
  op_tiling/
    all_gather_matmul_tiling.cpp    -- 完整tiling逻辑(单文件, ~630行)
    all_gather_formulaic_tiling.h   -- 公式化tiling: AllGatherPlusMM
    all_gather_formulaic_tiling.cpp -- 公式化tiling实现
```

与matmul_all_reduce (27个文件)对比: all_gather_matmul极度精简。所有tiling逻辑在单一文件中完成，无TilingBaseClass继承，无arch目录分层，无量化变体类。

### 1. 架构差异: 无继承 vs 深继承

matmul_all_reduce: TilingBaseClass -> MatmulAllReduceTilingBase -> 10个arch+场景子类
all_gather_matmul: 无继承体系，纯静态函数组织

all_gather_matmul没有使用TilingBaseClass模板方法模式，tiling入口函数AllGatherMatmulTilingFunc直接包含完整流程:
1. 参数校验(AllGatherParamsCheck)
2. 属性解析(group/transA/transB/gather_index/comm_turn)
3. SoC参数设置(SetSocParam)
4. 通信算法设置(SetCommAlg)
5. rank有效性校验(VALID_RANK map)
6. matmul tiling计算(SetMatmulTilingAllGatherMatmul -> MCSpliteM)
7. tiling key计算(UpdateTilingKey)
8. 通信参数初始化(InitHcclParam)

这可能因为all_gather_matmul:
- 只支持FP16/BF16(无量化)，不需要场景分支
- 没有arch31特有逻辑，arch分支通过isA3标志处理(而非独立类)

### 2. Op定义对比

all_gather_matmul vs matmul_all_reduce:

| 特征 | all_gather_matmul | matmul_all_reduce |
|------|-------------------|-------------------|
| 输入dtype数 | 2(fp16/bf16) | 65 |
| 输入数 | 3(x1,x2,bias) | 10 |
| 输出数 | 2(y, gather_out) | 1(y) |
| 量化输入 | 无 | 7个optional |
| OpAICoreConfig | 有(详细) | 无(在def.cpp中) |
| HcclGroup声明 | `this->MC2().HcclGroup("group")` | 无(在def.cpp中) |
| .IgnoreContiguous | 仅x2 | 全部输入 |
| AddConfig | ascend910b/ascend910_93/ascend950 | 无 |

all_gather_matmul的def.cpp:
- 包含OpAICoreConfig配置块: DynamicCompileStaticFlag/DynamicFormatFlag等
- 使用ExtendCfgInfo设置"aclnnSupport"/"jitCompile"/"multiKernelSupportDynamicGraph"
- 使用this->MC2().HcclGroup("group")声明HCCL通信域
- gather_out是第二个输出(AllGather的中间结果)

特有属性:
- gather_index: 收集哪个矩阵(当前只支持0，即A矩阵)
- rank_size: 可选的rank数
- is_gather_out: 是否需要gather输出

### 3. InferShape对比

all_gather_matmul使用公共函数AllGatherMatmulCommonInferShape(定义在mc2/common/src/mc2_common_infershape.cpp)。

InferDataType: 直接输入dtype透传到两个输出。

对比matmul_all_reduce的自包含infershape逻辑，all_gather_matmul的infershape极简(7行实质代码)。这是因为MC2公共层抽取了AllGather场景的通用shape推导。

### 4. Tiling逻辑

all_gather_matmul_tiling.cpp是一个大的单文件(~630行)，所有逻辑用静态函数组织:

关键静态函数:
- AllGatherParamsCheck(): 参数校验(K范围[256, 65535)，transA=false，gather_index=0)
- SetSocParam(): 通过GetCommSets判断isA3(0=fullmesh/910B，1=doublering/910_93)
- SetCommAlg(): 固定fullmesh
- SetMatmulTilingAllGatherMatmul(): 主tiling逻辑
- MCSpliteM(): M轴切分(三分支: enableSplitK / commTurn!=0 / 公式化)
- CalcMatmulTiling(): 直接调用matmul_tiling::MultiCoreMatmulTiling
- GetAllGatherFormulateTileCnt(): 公式化tiling入口
- UpdateTilingKey(): 使用APT tiling key宏
- MC2SetWorkspace(): workspace计算
- InitHcclParam(): Mc2CcTilingConfig获取通信tiling

M轴切分特殊性:
- args.rankTileNum = args.rankDim - 1 (因为AllGather场景local先算，其他rank的数据通信后算)
- 支持三种切分: enableSplitK不切 / commTurn指定切 / 公式化切分
- MC2_Splite(): 简单的等分+对齐(128/32字节对齐)

AllGather通信特有:
- localTiling: 本卡数据的matmul tiling(先计算)
- tileTiling/tailTiling: 其他卡数据的matmul tiling(通信后计算)
- DoubleRing下: args.mValue /= DOUBLE_RING_FACTOR
- gather_out: 可选，控制是否需要workspace存储gather数据

### 5. APT Tiling Key

使用ASCENDC_TPL_ARGS_DECL/ASCENDC_TPL_SEL宏声明模板参数:
- ALL_GATHER_MM_FULL_MESH: bool
- ALL_GATHER_MM_ND2NZ_OPT: bool
- ALL_GATHER_MM_BIAS_CAST: bool

只有4种合法组合(fullmesh=1时的2x2组合)。

### 6. 公式化Tiling (AllGatherPlusMM)

继承OneCalcOneCommBase(与MMPlusAllReduce同源):
- commShapeLen = K (AllGather收集的是A矩阵的K方向数据)
- 而MMPlusAllReduce的commShapeLen = N (AllReduce归约的是结果矩阵的N方向)
- 新增: frontMMTime_(local matmul先行计算的时间)
- hasLocalAtFront_ = true (AllGather特有: local先计算)
- noCutFlag_: totalTpTime < frontMMTime时不切(通信比local计算还快，不需要通算重叠)

A5/A3 CommTimeFactor区分:
- SOC950: ALLGATHERMM_COMMTIME_FACTOR
- SOC910_93: 额外乘0.6
- SOC910_B: 复杂逻辑(大K大N膨胀处理 + 最大tile数限制8)

FitTileLengthDiscrete (vs AllReduce的FitTileLengthContinuous): AllGather使用离散tile长度调整。

### 7. 反模式和代码质量问题

1. 属性索引硬编码: all_gather_matmul_tiling.cpp中使用`context->GetAttrs()->GetAttrPointer<bool>(1)`等数字索引，而非枚举。与matmul_all_reduce使用MmAllReduceAttrIdx枚举的做法不一致。

2. VALID_RANK硬编码map: `{0, {2, 4, 8}}, {1, {2, 4, 8, 16, 32}}`直接写在匿名namespace中。matmul_all_reduce使用Mc2TilingUtils::CheckRankSize(npuArch_)的通用方式。

3. const_cast: AllGatherParamsCheck中`const_cast<gert::TilingContext*>(context)`将const参数强转为非const传给CheckOutputParamDim0。这是因为GetTilingData等API需要非const context。

4. K值范围硬编码: `valueTwo < KVALUE_MIN || valueTwo >= KVALUE_MAX`，常量定义在其他头文件中。

5. workspace魔数: `16 * 1024 * 1024`(16MB)未附注释解释为何需要额外16MB。

6. 代码风格混合: 花括号位置不一致(有的`else {`同行，有的`else\n{`下行)，缩进混合(tab和空格)。

### 8. TilingData结构

AllGatherMatmulTilingData(定义在op_kernel/all_gather_matmul_tiling.h):

```
Mc2InitTiling mc2InitTiling    -- CCU通信初始化参数
Mc2CcTiling mc2CcTiling        -- CCU通信tiling参数
TCubeTiling tileTiling         -- 长tile的matmul tiling
TCubeTiling tailTiling         -- 短tile的matmul tiling
TCubeTiling localTiling        -- 本卡local matmul tiling (AllGather特有)
TileL2Tiling tileL2Tiling     -- 长tile L2参数
TileL2Tiling tailL2Tiling     -- 短tile L2参数
TileL2Tiling localL2Tiling    -- 本卡local L2参数
RCSTiling param               -- RCS通信参数
AllGatherSoc socParam          -- SoC参数(commAlg/isA3/isStep/isND2NZ)
```

关键区别: AllGatherMatmul比MatmulAllReduce多一套localTiling+localL2Tiling。
AllGather先算本卡数据(local)，再算通信到来的远端数据(tile/tail)。
AllReduce没有local先算的概念，是先通信再归约。

AllGatherSoc结构: 直接用uint32标志位区分平台(isA3)，比matmul_all_reduce的NpuArch枚举粗糙。

### 9. Config文件

- binary.json: ascend910b和ascend910_93内容完全相同，各两个二进制(FP16+BF16)
- 没有ascend950目录(950走all_gather_matmul_v2)
- simplified_key.ini: default=0(框架默认)
- shape全部使用[-2](动态shape支持)

### 10. v1 vs v2确认

all_gather_matmul_v2(21个文件)已采用标准架构:
- AllGatherMatmulTilingBase : TilingBaseClass(继承模板方法模式)
- arch32/all_gather_matmul_tiling_v2.cpp: REGISTER_TILING_TEMPLATE_WITH_ARCH
- arch35/: AllGatherQuantBmmTiling(量化BMM) + AllGatherFitBalanceTiling(配平)
- 有独立的tiling_func.h(arch32 tiling函数声明)
- 有ascend950 config目录

这证实: v1扁平实现是"早期模式"，v2继承实现是"成熟模式"。新增算子应遵循v2模式。

### 11. 关键结构差异总结

all_gather_matmul vs matmul_all_reduce的根本差异:

通信语义差异:
- AllGather: 通信输入数据(A矩阵, K轴)，先算local再等通信
- AllReduce: 通信输出数据(结果矩阵, N轴)，先算全部再通信归约

InferShape差异:
- AllGather: M轴放大rankSize倍 (M_out = M_in * rankSize)
- AllReduce: M轴不变
- ReduceScatter: M轴缩小rankSize倍 (M_out = M_in / rankSize)

架构差异:
- v1(all_gather_matmul): 扁平单文件，isA3标志位，无继承，3 bool tiling key
- matmul_all_reduce: 深继承(10类)，双Registry路由，10+参数tiling key
- v2(all_gather_matmul_v2): 已迁移到标准继承模式

代码质量差异:
- 属性访问: v1用magic index，matmul_all_reduce用枚举
- SoC区分: v1用isA3标志，matmul_all_reduce用arch目录+注册
- 通信算法: v1固定fullmesh(SetCommAlg)，matmul_all_reduce通过GetCommSets动态查询

反模式汇总(v1特有，v2已修正):
1. 属性magic index访问(应使用枚举)
2. VALID_RANK硬编码(应使用Mc2TilingUtils::CheckRankSize)
3. const_cast绕过API限制
4. workspace魔数(16MB)缺注释
5. 代码风格不一致(花括号/缩进)

---

## 3.1.3 moe_distribute_combine/op_host/ -- 全量分析

### 文件结构 (4个C++文件 + 3x2 config文件)

```
op_host/
  moe_distribute_combine_def.cpp              -- 算子定义(OpDef)
  moe_distribute_combine_infershape.cpp       -- InferShape + InferDataType
  op_tiling/
    moe_distribute_combine_tiling.cpp         -- 完整tiling逻辑(~935行)
    moe_distribute_combine_tiling_a2a3.h      -- MoeTilingBase子类声明
  config/
    ascend910b/   (binary.json + simplified_key.ini)
    ascend910_93/ (binary.json + simplified_key.ini)
    ascend950/    (binary.json + simplified_key.ini) -- 注意: 多了950配置
```

### 1. 架构: 第三种模式 -- MoeTilingBase + 混合路由

moe_distribute_combine使用了不同于matmul_all_reduce和all_gather_matmul的第三种tiling架构:

继承关系: TilingBaseClass -> MoeTilingBase -> MoeDistributeCombineTilingA2A3
注册方式: REGISTER_OPS_TILING_TEMPLATE(MoeDistributeCombine, MoeDistributeCombineTilingA2A3, 0)
路由: TilingRegistry::GetInstance().DoTilingImpl(context) -- 使用第三种Registry

与前两者对比:
- matmul_all_reduce: TilingBaseClass -> MatmulAllReduceTilingBase -> 10个子类，双Registry(Arch + New)
- all_gather_matmul: 无继承，直接IMPL_OP_OPTILING注册
- moe_distribute_combine: MoeTilingBase -> 1个子类，REGISTER_OPS_TILING_TEMPLATE + TilingRegistry

MoeTilingBase(mc2/common/inc/tiling/moe_tiling_base.h)继承TilingBaseClass，是MoE系列算子的公共基类。

关键区别: MoeDistributeCombineTilingA2A3的DoOpTiling()直接调用静态函数MoeDistributeCombineTilingFunc，
函数内部通过socVersion字符串分派到A2或A3/A5路径:
```cpp
if (socVersion == "Ascend910B") {
    ret = MoeDistributeCombineA2TilingFuncImpl(context);
} else {
    ret = MoeDistributeCombineA3A5TilingFuncImpl(context);
}
```
这是"继承框架+过程式分派"的混合模式: 用了TilingBaseClass的框架(获得模板方法流程)，
但DoOpTiling()内部又退化为过程式的SoC版本字符串比较。

GetTilingKey()也是在DoOpTiling()中已经SetTilingKey，GetTilingKey只是context->GetTilingKey()读回来。
这偏离了TilingBaseClass的设计意图(GetTilingKey应该是独立计算tilingKey的)。

### 2. Op定义 (moe_distribute_combine_def.cpp)

MoE领域特有的输入集:

输入(11个):
- expand_x (REQUIRED): bf16/fp16/int32/int32 -- 4种dtype组合
- expert_ids (REQUIRED): int32
- expand_idx (REQUIRED): int32
- ep_send_counts (REQUIRED): int32
- expert_scales (REQUIRED): float32
- tp_send_counts (OPTIONAL): int32
- x_active_mask (OPTIONAL): bool
- activation_scale (OPTIONAL): float32
- weight_scale (OPTIONAL): float32
- group_list (OPTIONAL): int64
- expand_scales (OPTIONAL): float32

输出: x (REQUIRED): bf16/fp16/bf16/fp16

属性(14个 -- 远多于MatMul+Comm类):
REQUIRED: group_ep, ep_world_size, ep_rank_id, moe_expert_num
OPTIONAL: group_tp, tp_world_size, tp_rank_id, expert_shard_type, shared_expert_num,
          shared_expert_rank_num, global_bs, out_dtype, comm_quant_mode, group_list_type

关键差异:
1. 双通信域: group_ep(EP域) + group_tp(TP域) vs MatMul+Comm类的单group
   `this->MC2().HcclGroup({"group_ep", "group_tp"})` -- 注意传入vector而非单string
2. MoE特有属性: moe_expert_num, expert_shard_type, shared_expert_num, shared_expert_rank_num
3. 不使用.IgnoreContiguous()，而使用.AutoContiguous() -- 所有输入都标记AutoContiguous
4. 输入类型有int32/bool/float32/int64 -- 远比MatMul类多样(不只是矩阵计算的dtype)
5. 两种AICore配置:
   - ascend910b(A2): jitCompile.flag=static_false
   - ascend950/ascend910_93(A3/A5): jitCompile.flag=static_true

### 3. InferShape (moe_distribute_combine_infershape.cpp)

自包含的70行InferShape:
- 输出shape = [bs, h]
- bs来自expert_ids.dim0，h来自expand_x.dim1
- 若输入维度为1则设为-1(动态shape)
- InferDataType: 直接取expand_x的dtype

与MatMul+Comm类的根本差异:
- MatMul类: shape由M/N/K矩阵维度+rankSize确定
- MoE: shape由bs(batch size)/h(hidden size)确定，与MatMul无关

### 4. Tiling: A2路径 vs A3/A5路径

A2路径(Ascend910B -- MoeDistributeCombineA2TilingFuncImpl):
1. SetScheduleMode(1) -- 批量模式，全核同时启动
2. 属性校验: epWorldSize必须是8的倍数且<=256，moeExpertNum<=512
3. 环境变量探测: MoeDistributeCombineA2IsLayered()
   - 检查HCCL_INTRA_PCIE_ENABLE=1且HCCL_INTRA_ROCE_ENABLE=0 -> layered模式
   - layered模式支持INT8通信量化
4. TilingKey: GET_TPL_TILING_KEY(tp, quantMode, layeredMode, TILINGKEY_TPL_A2)
5. 通信: opType=18(BatchWrite), algConfig="BatchWrite=level0:fullmesh"
6. Workspace: SYSTEM_NEED_WORKSPACE(16MB) + moeExpertNum*sizeof(uint32_t)*2

A3/A5路径(Ascend910_93/Ascend950 -- MoeDistributeCombineA3A5TilingFuncImpl):
1. 属性校验: epWorldSize必须是8的倍数(A5放宽到2的倍数), <=288
   - epWorldSize有效值: A3={8,16,32,64,128,144,256,288}, A5={2,4,...,288}
2. 双通信域tiling: SetHCommCfg配置EP(AlltoAll)+TP(ReduceScatter)
   - EP: opType=AllToAll, algConfig="AlltoAll=level0:fullmesh;level1:pairwise"
   - TP: opType=ReduceScatter, algConfig="ReduceScatter=level0:ring"
3. mc2CcTilingConfig.SetCommEngine(AIV_ENGINE) -- 不拉起AICPU，提高退出性能
4. Window Size校验: 详细计算HCCL buffer是否足够
   - EP: epWorldSize * maxBs * h * 2 * 2 * localMoeExpertNum
   - TP: A * (h * 2 + 128) * 2
5. TilingKey: GET_TPL_TILING_KEY(tp, quantMode, layeredMode=MTE, archTag)
   - archTag: DAV_3510 -> TILINGKEY_TPL_A5, 否则 -> TILINGKEY_TPL_A3
6. Workspace: 固定16MB(SYSTEM_NEED_WORKSPACE)

### 5. TilingKey设计

4个参数(比all_gather_matmul多，但仍比matmul_all_reduce少):
- TILINGKEY_TP: bool (0/1) -- 是否有TP域
- TILINGKEY_QUANT_MODE: uint2 (NO_QUANT=0 / INT8_QUANT=1)
- TILINGKEY_LAYER_MODE: uint2 (MTE=0 / AICPU=1) -- A2 layered通信
- ARCH_TAG: uint2 (A2=0 / A3=1 / A5=2)

ASCENDC_TPL_SEL中定义了4组合法组合:
- A5: tp=0/1, quant=0/1, layer=MTE, arch=A5 (用MoeDistributeCombineTilingData)
- A3: tp=0/1, quant=0/1, layer=MTE, arch=A3 (用MoeDistributeCombineTilingData)
- A2无量化: tp=0, quant=0, layer=MTE/AICPU, arch=A2 (用MoeDistributeCombineA2TilingData)
- A2量化: tp=0, quant=INT8, layer=AICPU, arch=A2 (用MoeDistributeCombineA2TilingData)

关键: A2和A3/A5使用不同的TilingData结构体(通过ASCENDC_TPL_TILING_STRUCT_SEL指定)。
这是matmul_all_reduce和all_gather_matmul都没有的特性 -- tiling key与tiling数据结构绑定。

### 6. 与v2的关系

moe_distribute_combine直接#include了v2的头文件:
- `../../../moe_distribute_combine_v2/op_kernel/moe_distribute_combine_tiling.h`
- `../../../moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_tiling_helper.h`

TilingData定义在v2的op_kernel中(被v1复用)。
MoeDistributeCombineTilingHelper校验函数在v2的op_host中(被v1复用)。

这说明v1和v2共享数据结构和部分校验逻辑，但tiling策略不同。

### 7. 关键差异总结

与matmul_all_reduce/all_gather_matmul对比:

| 维度 | matmul_all_reduce | all_gather_matmul | moe_distribute_combine |
|------|------------------|-------------------|----------------------|
| 继承深度 | 3层(10个类) | 无继承 | 2层(1个类) |
| 注册方式 | WITH_SOCVERSION/ARCH | IMPL_OP_OPTILING | REGISTER_OPS_TILING_TEMPLATE |
| Registry | 双(Arch+New) | 无 | 单(TilingRegistry) |
| arch分派 | Registry层面 | isA3标志 | socVersion字符串 |
| 通信域 | 单(group) | 单(group) | 双(group_ep+group_tp) |
| 输入个数 | 10 | 3 | 11 |
| 属性个数 | 9 | 7 | 14 |
| dtype组合 | 65 | 2 | 4 |
| matmul | 有(核心) | 有(核心) | 无(纯通信+数据重排) |
| 公式化tiling | MMPlusAllReduce | AllGatherPlusMM | 无(不需要通算配平) |
| TilingData数量 | 3(per arch) | 1 | 2(A2/A3分开) |
| AutoContiguous | IgnoreContiguous | IgnoreContiguous(仅x2) | AutoContiguous(全部) |
| jitCompile | 未配置 | static_false | A2=static_false, A3/A5=static_true |

### 8. 反模式和代码质量问题

1. socVersion字符串比较(line 889): `socVersion == "Ascend910B"` -- 硬编码字符串而非NpuArch枚举。
   Phase 1发现"socVersion硬编码"是被大规模替换的反模式。
   同一文件中也有正确写法: `mc2tiling::GetNpuArch(context) == NpuArch::DAV_3510`(line 643/852)。
   两种方式混用说明代码在迁移过程中。

2. H值硬编码(line 542): `expandXDim1 != 7168` -- hidden size硬编码为7168。
   这是极强的业务假设(特定模型的hidden size)，限制了算子的通用性。

3. 环境变量判断通信模式(line 254-269):
   `getenv("HCCL_INTRA_PCIE_ENABLE")` + `getenv("HCCL_INTRA_ROCE_ENABLE")`
   直接用环境变量判断是否layered，C风格，无线程安全保证。

4. 属性索引使用constexpr常量(line 52-63):
   `constexpr uint32_t ATTR_EP_WORLD_SIZE_INDEX = 1;`
   比all_gather_matmul的magic index好，但不如matmul_all_reduce的枚举。
   且ATTR_COMM_QUANT_MODE_INDEX=12跳过了11(out_dtype)，容易出错。

5. v1和v2之间的跨目录#include(line 38/20):
   `"../../../moe_distribute_combine_v2/op_kernel/moe_distribute_combine_tiling.h"`
   相对路径跨三级目录引用v2代码，紧密耦合。

6. GetTilingKey()非独立计算(line 917-924):
   `return context_->GetTilingKey()` -- 依赖DoOpTiling()已经SetTilingKey。
   偏离TilingBaseClass的设计意图(GetTilingKey应独立计算)。

### 9. MoE域特有模式

1. 双通信域配置:
   EP域使用AlltoAll(fullmesh+pairwise), TP域使用ReduceScatter(ring)。
   需要两个Mc2CcTilingConfig实例，mc2CcTiling1/mc2CcTiling2分别存储。

2. 共享专家(Shared Expert)逻辑:
   epRankId < sharedExpertRankNum的rank承担共享专家角色。
   共享专家不做token dispatch，只做combine(从所有rank收集结果)。

3. A2 vs A3 TilingData分离:
   A2使用MoeDistributeCombineA2TilingData(轻量)。
   A3/A5使用MoeDistributeCombineTilingData(含mc2CcTiling1/mc2CcTiling2)。
   通过ASCENDC_TPL_TILING_STRUCT_SEL在tiling key中绑定。

4. Window Size精确计算:
   MoE需要根据globalBs * moeExpertNum * h计算所需buffer空间。
   layered模式需额外的flag区域(6MB)和per-token附带信息(expert_id/topk_weight/quant_scale/flag)。

5. config/目录(MoE特有):
   binary.json枚举预编译kernel二进制(文件名+输入输出类型), simplified_key.ini控制opc编译器key模式。
   ascend910b只有2个bin(BF16/FP16), ascend910_93和ascend950有4个bin(多INT32量化变体)。
   matmul_all_reduce和all_gather_matmul都没有config/目录。

6. CombineV2Config索引映射表(v2 helper):
   87个uint32_t字段记录v1/v2/v3三版本算子的input/output/attr位置映射。
   v1的TilingCheckMoeDistributeCombine委托给v2的helper实现，通过config对象适配不同原型。
   设计意图: 一套校验代码适配多版本算子原型，但87字段本身也是维护负担。

7. A3路径expandX dim1硬编码检查(line 542):
   `OP_TILING_CHECK((expandXDim1 != 7168), ...)` -- hidden_size=7168是特定模型(如Deepseek)的值。
   A2路径也有MAX_HIDDEN_SIZE_A2=7168但作为上限而非精确值，两套约束语义不同。

8. ATTR_COMM_QUANT_MODE_INDEX=12跳过了11(out_dtype attr):
   属性索引序列为0-10,12(跳过11)。out_dtype的index=11但在tiling中通过attrOutDTypeIndex(v2)访问。
   跳跃索引容易出错，是hard-to-maintain模式。

---

## 3.1.4 三者对比: 共性模式和差异点

### 1. 共性模式(强制规范候选)

a) Op定义(def.cpp)模式:
- 所有算子: 继承OpDef + 构造函数声明 + OP_ADD()宏注册
- 所有输入/输出: 必须同时声明.Format()和.UnknownShapeFormat() (出现3/3)
- 属性: group/group_ep是REQUIRED String (出现3/3)
- config: binary.json + simplified_key.ini per SoC (出现3/3)

b) InferShape模式:
- 所有算子: 两个静态函数(InferShape + InferDataType) + IMPL_OP_INFERSHAPE注册 (3/3)
- InferDataType通常极简(透传输入dtype到输出)
- 使用OPS_CHECK_NULL_WITH_CONTEXT做null检查

c) Tiling入口:
- 所有算子: IMPL_OP_OPTILING注册 + TilingParse空实现(CompileInfo空结构体) (3/3)
- 所有算子: matmul使用MultiCoreMatmulTiling + MatmulFormulaicTiling::GetCubeTiling做matmul部分tiling (2/2 matmul类)

d) TilingKey:
- 所有算子: 使用GET_TPL_TILING_KEY宏生成 (3/3)
- 参数都是bool/uint组合的编译期多态key

e) 通信Tiling:
- A3/A5路径: 都使用Mc2CcTilingConfig获取mc2InitTiling + mc2CcTiling (3/3)
- A2路径: 各有不同实现

f) SetScheduleMode(1):
- MoE的A2路径和matmul_all_reduce的A5路径都调用(需全核同步的算子)

g) Workspace:
- 所有算子: GetWorkspaceSizes(1)获取workspace指针 + 精确计算 (3/3)
- SYSTEM_NEED_WORKSPACE=16MB固定预留是MoE和AllGather的共同模式

### 2. 差异点(需区分的规范层级)

a) Tiling架构(三种模式):
- 深继承(matmul_all_reduce): TilingBaseClass->Base->子类, 适合多量化场景+多arch
- 浅继承(moe_distribute_combine): MoeTilingBase->单子类, 适合单场景多SoC
- 无继承(all_gather_matmul v1): 纯静态函数, 适合简单算子(已被v2淘汰)

b) Registry注册维度:
- REGISTER_TILING_TEMPLATE_WITH_SOCVERSION: matmul_all_reduce arch31/32
- REGISTER_TILING_TEMPLATE_WITH_ARCH: matmul_all_reduce arch35
- REGISTER_OPS_TILING_TEMPLATE: moe系列
- 直接IMPL_OP_OPTILING: all_gather_matmul v1

c) 属性访问方式(质量递减):
- 枚举常量索引(matmul_all_reduce): MmAllReduceAttrIdx::K_TRANS_X1 -- 最安全
- constexpr常量索引(moe_distribute_combine): ATTR_EP_WORLD_SIZE_INDEX=1 -- 次安全
- magic index++(all_gather_matmul v1): index++ -- 最脆弱

d) SoC分派方式(质量递减):
- arch目录+Registry(matmul_all_reduce): 编译期分派，每arch独立文件 -- 最佳
- NpuArch枚举判断(moe_distribute_combine部分): GetNpuArch() == DAV_3510 -- 可接受
- socVersion字符串比较(moe_distribute_combine部分): socVersion == "Ascend910B" -- 反模式
- isA3标志位(all_gather_matmul v1): 布尔值，无法扩展 -- 反模式

e) 通信域数量:
- 单域: matmul_all_reduce(group), all_gather_matmul(group)
- 双域: moe_distribute_combine(group_ep + group_tp)
  -> def.cpp中MC2().HcclGroup传入vector vs string

f) IgnoreContiguous vs AutoContiguous:
- MatMul类使用.IgnoreContiguous()
- MoE类使用.AutoContiguous()
- 区别: IgnoreContiguous跳过连续性检查(输入数据可能不连续)
         AutoContiguous自动确保连续(框架会做copy如果需要)

g) jitCompile配置:
- all_gather_matmul: static_false(动态shape复用二进制)
- moe_distribute_combine A2: static_false, A3/A5: static_true
- matmul_all_reduce: 不在def.cpp中配置

### 3. 演进方向

从三个算子的对比可以提取明确的演进方向:

1. tiling架构: 从扁平静态函数(v1) -> TilingBaseClass继承(v2) -> 带arch子目录的深继承(成熟算子)

2. 注册方式: 从直接IMPL_OP_OPTILING -> REGISTER_OPS_TILING_TEMPLATE -> REGISTER_TILING_TEMPLATE_WITH_ARCH

3. 属性访问: 从magic index -> constexpr常量 -> 枚举(最安全)

4. SoC分派: 从标志位/字符串 -> NpuArch枚举 -> arch目录+Registry自动分派

5. 通信配置: 从手动填充KFC消息 -> Mc2CcTilingConfig新接口(已全面迁移)

### 4. 规范候选提取

强制规范:
- def.cpp: OP_ADD宏注册，Format/UnknownShapeFormat双声明
- infershape: IMPL_OP_INFERSHAPE注册，静态函数形式
- tiling: IMPL_OP_OPTILING注册，TilingParse空实现
- tilingKey: GET_TPL_TILING_KEY宏生成
- 通信: Mc2CcTilingConfig获取mc2InitTiling/mc2CcTiling

推荐规范:
- 新增算子使用TilingBaseClass继承模式(非v1扁平模式)
- 属性访问使用枚举或constexpr常量(非magic index)
- SoC分派使用NpuArch枚举或arch目录(非socVersion字符串)
- 多arch算子使用REGISTER_TILING_TEMPLATE_WITH_ARCH注册

可选规范:
- OpAICoreConfig在def.cpp中配置(有些算子不配置也可以)
- AutoContiguous vs IgnoreContiguous取决于业务场景

---

## 3.2 Cross-Operator Pattern Extraction

### 3.2.1 Tiling策略跨算子统计

#### 注册宏使用统计

三种注册宏在mc2/下的分布:

1. IMPL_OP_OPTILING: 37处/37文件(100%覆盖)
   - 每个MC2算子(含3rd)恰好一处，是强制的tiling入口注册
   - 总是配对TilingParse(78处/35文件)

2. REGISTER_OPS_TILING_TEMPLATE: 33处/26文件
   - MoE系列: moe_distribute_combine/dispatch/v2 + setup/teardown (12处)
   - GMM系列: allto_allv_grouped/quant_grouped (3处)
   - 3rd库: weight_quant_batch_matmul_v2(10处!最多) + mat_mul_v3/batch_mat_mul_v3/quant_batch_matmul_v3 (5处)
   - 其他: inplace_matmul_all_reduce_add_rms_norm(3处), matmul_all_reduce_add_rms_norm(3处)
   - 特征: 算子内部自行做arch分派(而非框架分派)

3. REGISTER_TILING_TEMPLATE_WITH_ARCH: 13处/13文件
   - matmul_all_reduce arch35 (3处: FP/Quant/WeightQuant)
   - all_gather_matmul_v2 (2处: arch32/arch35)
   - matmul_reduce_scatter_v2 (2处: arch32/arch35)
   - matmul_allto_all (3处: arch35 FP/KcQuant/MxQuant)
   - allto_all_matmul (3处: arch35 FP/KcQuant/MxQuant)
   - 特征: arch35(DAV_3510)专用，与TilingRegistryArch配合

4. REGISTER_TILING_TEMPLATE_WITH_SOCVERSION: 11处/11文件
   - matmul_all_reduce arch31+arch32 (7处: 310P 4处 + 910B 3处)
   - matmul_allto_all arch32 (2处)
   - allto_all_matmul arch32 (2处)
   - 特征: 旧arch(非DAV_3510)专用，与TilingRegistryNew配合

分布规律:
- MatMul+Comm类算子(matmul_all_reduce/matmul_allto_all/allto_all_matmul)使用WITH_ARCH + WITH_SOCVERSION
- MoE类算子使用OPS_TILING_TEMPLATE
- v2版算子(all_gather_matmul_v2/matmul_reduce_scatter_v2)已迁移到WITH_ARCH
- v1版算子(all_gather_matmul/matmul_reduce_scatter)使用直接IMPL_OP_OPTILING(无中间注册)

#### TilingKey统计

GET_TPL_TILING_KEY宏: 99处/77文件(跨越op_host和op_kernel两层)
ASCENDC_TPL_ARGS_DECL: 36个tiling_key.h文件(每个算子一个)

TilingKey定义位置: 全部在op_kernel/<算子名>_tiling_key.h中
- 有些算子有apt_tiling_key.h(matmul_all_reduce, all_gather_matmul_v2, matmul_reduce_scatter_v2)
- 新的apt格式和旧的手动格式共存

#### SetScheduleMode(1)统计

39处/31文件 -- 几乎所有MC2算子都调用
- 含义: "独占全核，batch mode模式，所有核同时启动"
- 注释一致: "涉及SyncAll/多核同步指令，需要做此设置避免影响整网其他算子"
- 这是MC2模块级的强制规范(因为MC2算子都有跨核同步)

#### Mc2CcTilingConfig统计

103处/47文件 -- MC2通信配置的标准API
- 核心用法: 构造(group, opType, algConfig) + GetTiling(tiling)
- 出现在所有有通信的算子中(100%覆盖)

#### HcclGroup统计

38处，两种形式:
- 单组: HcclGroup("group") -- 18处(MatMul+Comm类: matmul_all_reduce/matmul_reduce_scatter等)
- 多组: HcclGroup({"group_ep", "group_tp"}) -- 8处(MoE类: combine/dispatch/v2/add_rms_norm/bmm_rs_a2a)
- 单EP组: HcclGroup({"group_ep"}) / HcclGroup("group_ep") -- 4处(setup/teardown)
- 注意: vector形式{"group"}和string形式"group"混用(技术上等价但风格不一致)

#### IgnoreContiguous vs AutoContiguous

- IgnoreContiguous: 16处/7文件 -- 仅MatMul+Comm类(matmul_all_reduce/matmul_reduce_scatter_v2/all_gather/inplace)
- AutoContiguous: 230处/23文件 -- MoE类/GMM类/AlltoAll类/FFN类/attention类
- 规律: 矩阵乘法算子用IgnoreContiguous(允许非连续输入,NZ格式), 其他用AutoContiguous
- 比例: AutoContiguous是主流(23/30算子)，IgnoreContiguous是少数特例(7/30)

### 3.2.2 InferShape模式跨算子统计

IMPL_OP_INFERSHAPE: 33处/33文件(100%覆盖，与OP_ADD数量一致)

所有InferShape文件遵循统一模式:
1. 静态函数InferShape*: 从input shape推导output shape
2. 静态函数InferDataType*: 确定输出dtype(通常透传)
3. IMPL_OP_INFERSHAPE(OpName).InferShape(...).InferDataType(...)注册

InferShape的两大类:
- MatMul类: 需要计算M/N/K矩阵维度，涉及rankSize缩放
  - AllReduce: output_m = input_m (N轴归约，M不变)
  - AllGather: output_m = input_m * rankSize (K轴gather，M放大)
  - ReduceScatter: output_m = input_m / rankSize (M轴scatter，M缩小)
- MoE/非MatMul类: 直接从输入shape推导(bs/h/expert_num等)

### 3.2.3 Op定义模式跨算子统计

OP_ADD: 35处/35文件(100%覆盖)

def.cpp统一模式:
1. class OpName : public OpDef
2. 构造函数中声明Input/Output/Attr
3. OpAICoreConfig配置
4. MC2().HcclGroup()声明通信组
5. AICore().AddConfig()注册SoC配置
6. OP_ADD(OpName)宏注册

AddConfig的SoC分布:
- ascend910b: 几乎所有算子
- ascend910_93: 大部分算子
- ascend950: 较新算子

ExtendCfgInfo常见配置项:
- aclnnSupport.value = support_aclnn (标准)
- jitCompile.flag = static_true / static_false
- multiKernelSupportDynamicGraph.value = multi_kernel
- prebuildPattern.value = Opaque

### 3.2.4 config/子目录结构

仅部分算子有config/子目录(包含binary.json + simplified_key.ini):
- moe_distribute_combine, moe_distribute_dispatch, moe_distribute_combine_v2, moe_distribute_dispatch_v2
- 不是所有算子都需要(matmul_all_reduce/all_gather_matmul没有)
- binary.json描述预编译kernel二进制列表(按dtype分)
- simplified_key.ini控制opc编译器的tiling key简化模式(通常default=0)

### 3.2.5 Phase 3 总结: op_host层规范提取

基于3.1深度阅读(3个代表性算子)和3.2跨算子统计(Grep全量扫描)，提取op_host层编码规范:

#### 强制规范(100%覆盖，违反即不可编译/不可运行)

1. OP_ADD注册: 每个算子恰好一个def.cpp，恰好一处OP_ADD(OpName)。(35/35)
2. IMPL_OP_INFERSHAPE注册: 每个算子恰好一个infershape.cpp，恰好一处IMPL_OP_INFERSHAPE(OpName).InferShape().InferDataType()。(33/33)
3. IMPL_OP_OPTILING注册: 每个算子恰好一处tiling入口注册。(37/37)
4. GET_TPL_TILING_KEY: 100%的MC2算子用此宏在tiling和kernel两层编码tiling key。(99处/77文件)
5. SetScheduleMode(1): 所有MC2算子必须在tiling中设置此模式(独占全核+批量启动)。(39处/31文件)
6. Mc2CcTilingConfig: 所有有通信的MC2算子用此API获取通信tiling。(103处/47文件)
7. Format双声明: def.cpp中Input/Output必须同时声明Format和UnknownShapeFormat。

#### 推荐规范(主流做法，新算子应遵循)

1. TilingBaseClass继承: 新增算子应使用TilingBaseClass继承模式(v2+算子100%使用)，而非v1扁平函数。
2. REGISTER_TILING_TEMPLATE_WITH_ARCH: arch35算子使用此宏进行框架级arch分派，而非算子内部分派。(13处)
3. 属性访问用枚举/constexpr: 而非magic index。(matmul_all_reduce已迁移到constexpr，all_gather_matmul v1仍用index++)
4. SoC分派用NpuArch枚举: 而非socVersion字符串比较。(moe_distribute_combine仍混用字符串)
5. AutoContiguous为默认: 除非算子明确需要处理非连续输入(NZ格式的MatMul类)才用IgnoreContiguous。(AutoContiguous 23算子 vs IgnoreContiguous 7算子)
6. InferShape共用函数: MatMul+Comm类算子应使用mc2_common_infershape.cpp中的公共函数。

#### 可选规范(取决于业务场景)

1. OpAICoreConfig(jitCompile/DynamicCompileStaticFlag): 根据算子shape动态性选择。
2. config/子目录(binary.json + simplified_key.ini): 仅需要预编译二进制或tiling key简化的算子需要。
3. 通信域数量: 单域用HcclGroup("group")，双域用HcclGroup({"group_ep", "group_tp"})。

#### 反模式(已在代码中发现的错误做法)

1. magic index++属性访问: all_gather_matmul v1用int index递增访问属性，极易越界。正确: 用枚举或constexpr。
2. socVersion字符串硬编码: moe_distribute_combine:889行 `socVersion == "Ascend910B"`。正确: 用NpuArch枚举。
3. hidden_size=7168硬编码: moe_distribute_combine:542行。正确: 从输入shape获取或作为属性传入。
4. VALID_RANK硬编码: all_gather_matmul用map固定合法rankSize集合。正确: 基于通信域动态查询或用范围检查。
5. 跨版本引用v2代码: moe_distribute_combine引用v2版的代码。正确: 提取公共代码到common/。
6. const_cast绕过: all_gather_matmul用const_cast修改本应不可变的数据。
7. shareL1Size=shareMode赋值错误: matmul_all_reduce weight_quant tiling中的copy-paste bug(163/239行)。

#### 架构演进方向(从git历史和v1/v2/v3对比总结)

1. tiling架构: 扁平函数(v1) → TilingBaseClass继承(v2) → arch子目录+Registry自动分派(成熟算子)
2. 注册方式: IMPL_OP_OPTILING → REGISTER_OPS_TILING_TEMPLATE → REGISTER_TILING_TEMPLATE_WITH_ARCH
3. 属性访问: magic index → constexpr常量 → 枚举
4. SoC分派: 标志位/字符串 → NpuArch枚举 → arch目录+Registry
5. 通信配置: 手动填充KFC消息 → Mc2CcTilingConfig统一接口
