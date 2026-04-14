# Phase 4: op_kernel Layer Analysis

## 4.1 Representative Operators Deep Dive

### 4.1.1 matmul_all_reduce/op_kernel/ (51个文件)

#### 目录结构

```
op_kernel/
  matmul_all_reduce.cpp          -- kernel入口(590行)
  matmul_all_reduce_tiling_key.h -- arch31/32 tiling key定义(745行!)
  matmul_all_reduce_apt_tiling_key.h -- arch35 APT tiling key(342行)
  matmul_all_reduce_apt.cpp      -- arch35 APT kernel入口
  common.h                       -- 算子级公共定义
  common/                        -- 算子级公共工具(element_wise_add)
  arch31/ (10个文件)             -- Ascend 310P (AICore type 200)
  arch32/ (14个文件)             -- Ascend 910B (AICore type 220)
  arch35/ (21个文件)             -- Ascend 950  (NPU_ARCH 3510)
```

#### Kernel入口文件: matmul_all_reduce.cpp

核心设计模式:

1. 条件编译层次: 外层用`__CCE_AICORE__`区分arch32(220) vs arch31(非220)，内层用#define宏(MC2_QUANT/MC2_WEIGHT_QUANT/MC2_NON_QUANT)区分量化模式。这些宏在common.h中根据ORIG_DTYPE_X1/X2的组合自动推导。

2. 三级kernel函数:
   - `matmul_all_reduce()`: 顶层入口，负责REGISTER_TILING_DEFAULT/FOR_TILINGKEY + TPipe创建 + workspaceGM空指针检查 + 按MM_TYPE分派到三个子函数
   - `fp_matmul_all_reduce()`: 非量化路径
   - `quant_matmul_allreduce()`: 全量化路径
   - `weight_quant_matmul_allreduce()`: 伪量化路径

3. REGISTER_TILING机制:
   - REGISTER_TILING_DEFAULT: 注册最大的TilingData结构体(WeightQuantMatmulAllReduceTilingData)
   - REGISTER_TILING_FOR_TILINGKEY: 按tiling key条件注册不同TilingData结构体
   - 目的: 让编译器分配足够大的tiling内存空间
   - 注释明确说明: 当前编译器不能自动从ASCENDC_TPL_TILING_STRUCT_SEL识别，需要手动注册

4. if constexpr分派: 所有运行时分支都用`if constexpr`，在编译期消除不需要的代码路径

5. INVOKE宏封装: 每种量化模式有专门的INVOKE_*宏(INVOKE_MC2_910_OP_IMPL, INVOKE_MC2_QUANT_910_OP_IMPL等)，封装TilingData获取+类实例化+Init+Process的完整流程

#### common.h分析

职责: 算子级公共定义，不是MC2模块级的common/

关键内容:
- 类型别名: A_DTYPE/B_DTYPE/C_DTYPE从编译器传入的DTYPE_X1/X2/Y宏推导
- HCCL通信结构: HcclCombinOpParam(含rankId/rankDim/windowsIn/windowsOut/signalInfo)
- 枚举: AntiQuantType(NONE/PER_TENSOR/PER_CHANNEL/PER_GROUP), DebugMode(5种), MC2_BUFFER_TYPE(7种), Mc2CoreType(CUBE_AND_VECTOR/VECTOR/CUBE)
- 条件推导宏: 从ORIG_DTYPE组合推导MC2_QUANT/MC2_WEIGHT_QUANT等宏
- 同步封装: Mc2SyncAll<Mc2CoreType>根据核类型选择不同同步方式
  - ON_CUBE_AND_VECTOR: SyncAll<false>() (不等待AIV完成)
  - ON_VECTOR: SyncAll() (标准全同步)
  - ON_CUBE: PipeBarrier<PIPE_ALL>() + CrossCoreSetFlag/WaitFlag(仅cube间同步)
- OOMInit: 检查workspace和window地址范围
- CastBFtoFloatOnAiv0: bf16->float32的cast工具，仅在AIV core 0上执行

#### Tiling Key设计对比: 旧版 vs APT版

旧版(matmul_all_reduce_tiling_key.h, 745行):
- 命名: MatmulAllReduce
- 16个模板参数: AICORE_TYPE(2bit) + MM_TYPE(2bit) + EMPTY_INPUT(bool) + INT8_COMM(bool) + ENABLE_L2_CACHE(bool) + SHARE_MM(bool) + MIXND2NZ(4bit) + TRANS(2bit) + PERTOKEN(2bit) + SUB_ALGORITHM_CUSTOM(4bit) + HAS_ANTIQUANT_OFFSET(bool) + ANTIQUANT_TYPE(4bit) + TRANS_A(bool) + TRANS_B(bool) + FORMAT_B(bool)
- ASCENDC_TPL_SEL: 用#if __CCE_AICORE__条件编译区分310P和910B的选择集
- 每个ASCENDC_TPL_ARGS_SEL都附带ASCENDC_TPL_TILING_STRUCT_SEL选择对应TilingData
- 注释中标注了每组选择的tiling key数量和值

APT版(matmul_all_reduce_apt_tiling_key.h, 342行):
- 命名: Mc2MatmulAllReduceApt
- 12个模板参数: APT_MM_TYPE(4bit) + TPL_TRANS_B(bool) + TPL_EXIST_BIAS(bool) + MATMUL_WITH_ADD(bool) + KERNEL_TYPE(4bit) + COMMDTPYE(2bit) + SCENARIO_MXFP8(bool) + TEMPLATE_CUSTOM(4bit) + ANTIQUANT_TYPE(2bit) + QUANTTYPE(2bit) + HAS_ANTIQUANT_OFFSET(bool) + BAIS_IS_FP32(bool) + WEIGHTFORMAT(2bit)
- 无条件编译: APT版的ASCENDC_TPL_SEL不按__CCE_AICORE__分支
- 使用ASCENDC_TPL_KERNEL_TYPE_SEL指定AIC/AIV核类型(旧版在旧arch不支持)
- 使用arch35专用TilingData(MatmulAllReduce910TilingDataA5等)

关键差异: APT版参数更面向业务语义(EXIST_BIAS/MATMUL_WITH_ADD/COMMDTPYE/SCENARIO_MXFP8)，旧版更面向实现细节(MIXND2NZ/SHARE_MM)。这是tiling key设计的演进方向: 从实现驱动到业务驱动。

#### arch32 vs arch35 Base类对比

同名文件MatmulAllReduceBase，但有关键差异:

1. HCCL Server类型:
   - arch32: `Hccl<HCCL_SERVER_TYPE_AICPU> hccl_` (通过AICPU代理通信)
   - arch35: `Hccl<HcclServerType::HCCL_SERVER_TYPE_CCU> hccl_` (CCU直接驱动通信)
   这是arch31/32到arch35最核心的架构变化

2. 初始化方式:
   - arch32: `hccl_.Init(GetHcclContext<0>())`
   - arch35: `hccl_.InitV2(GetHcclContext<0>(), tilingData_)` + `hccl_.SetCcTilingV2(offsetof(MC2TilingHeader, mc2CcTiling))`
   arch35使用V2版初始化，传入tilingData中的mc2CcTiling偏移量

3. 输出缓冲区选择:
   - arch32: 通过`msgInTiling_->useBufferType`和`determinism`判断是否使用WindowIn
   - arch35: `addrs_->cGM = addrs_->workspaceGM + nd2NzWorkLen + biasLen` (直接从workspace偏移)
   arch35不再依赖AICPU的窗口地址，直接在workspace中分配

4. Wait语义:
   - arch32: `hccl_.Wait(handleId)` 一次Wait等待所有tile
   - arch35: 循环`for (i=0; i<tileCnt; ++i) hccl_.Wait(handleId)` 逐片Wait
   arch35的CCU需要per-tile的显式等待

5. 数据类型支持:
   - arch35增加fp4x2_e2m1_t/fp4x2_e1m2_t支持(4-bit场景)
   - arch35增加isOneTileFlag_优化(单tile时用rankM替代tile M)

6. MatmulOp使用:
   - arch32: 使用mat_mul_v3的MatmulBaseBlock (Mc2MatmulBaseBlock)
   - arch35: 使用mat_mul_v3/arch35的MatmulAswBlock (Mc2MatmulV3Advanced::Mc2MatmulAswBlock)
   ASW = Adaptive Sliding Window

### 4.1.2 all_gather_matmul/op_kernel/ (5个文件)

#### 目录结构

```
op_kernel/
  all_gather_matmul.cpp              -- kernel入口(82行)
  all_gather_matmul_tiling.h         -- TilingData结构体(49行)
  all_gather_matmul_tiling_key.h     -- 3-bool tiling key(51行)
  all_gather_matmul_base.h           -- Base类(155行)
  all_gather_matmul_full_mesh.h      -- FullMesh实现(230行)
```

总计仅5个文件、~567行，与matmul_all_reduce的51个文件/数千行形成鲜明对比。

#### Kernel入口

比matmul_all_reduce简洁得多:
- 无条件编译分支(不需要区分arch)
- 仅3个bool模板参数: IsFullMesh, IsNd2Nz, IsBias
- `#if __CCE_AICORE__ != 310`直接排除A5(注释说明"不支持A5")
- 4个constexpr分支覆盖所有组合
- INVOKE_ALL_GATHER_MATMUL_OP_IMPL宏: 实例化模板类 + Init + Process

#### 通信模式差异

与matmul_all_reduce的关键差异:

1. 通信操作: AllGather(收集) vs AllReduce(规约)
   - AllGather: hccl_.AllGather<true>(aGM, gatherGM, ...) -- gather输入A矩阵
   - AllReduce: hccl_.AllReduce(cGM, outputGM, ...) -- reduce输出C矩阵

2. 计算通信重叠策略:
   - AllGather: "先算本卡 + 等通信 + 算远端" -- 本地计算和通信可以重叠
     - MatmulKernelLocal(): 计算本卡的A*B(无需等待通信)
     - MatmulKernelGather(): 等待gather完成后，逐rank计算远端数据
   - AllReduce: "算 + 同步 + 提交通信" -- 计算和通信串行
     - InnerProcess(): 每个tile先做matmul，PostProcEachTurn中Commit通信

3. Gather特有: gather_out输出(通信中间结果)
   - gatherGM_空间管理: 如果gatherLen != 0或gatherGM为空，从workspace分配

4. 多rank计算调度:
   - 使用`(rankId_ + j + GetBlockIdx()) % rankDim_`的轮询模式
   - 每个core从不同rank开始计算，负载均衡

#### Base类分析

AllGatherMatmulBase不继承任何外部类(vs matmul_all_reduce继承链)。
使用mc2/common的MatmulCompute<>模板类做矩阵乘，支持两种模式:
- SplitType::DEFAULT: 标准计算
- SplitType::L2CACHE: L2 cache优化计算

### 4.1.3 moe_distribute_combine/op_kernel/ (3个文件)

#### 目录结构

```
op_kernel/
  moe_distribute_combine.cpp         -- kernel入口(111行)
  moe_distribute_combine.h           -- MoeDistributeCombine类(880行)
  moe_distribute_combine_tiling_key.h -- 4维tiling key(75行)
```

#### 架构差异: 无MatMul

MoE Combine与MatMul+Comm算子有本质区别:
- 不做矩阵乘: 核心操作是AlltoAll数据重排 + Scale加权累加 + ReduceScatter
- 不使用mat_mul_v3/3rd库: 完全自己实现数据搬运和向量计算
- 不继承MatmulAllReduceBase: 独立的类MoeDistributeCombine

#### 通信模式: Window-Based AlltoAll

与MatMul+Comm的HCCL API调用不同，MoE使用基于窗口的AlltoAll:

1. 双通信域: EP域(epWinContext_) + TP域(tpWinContext_)
   - EP AlltoAll: 通过RDMA窗口直接写对端内存
   - TP ReduceScatter: 类似的窗口机制

2. 状态同步: 自实现的状态位轮询
   - SetStatus(): 向对端窗口写入状态标志
   - WaitDispatch(): 轮询读取对端状态，用GatherMask+Sum判断所有rank就绪
   - InitStatusTargetSum(): 双缓冲状态切换(0↔1)，通过int32_t存储float的IEEE754表示(0x3F800000=1.0)

3. 数据搬运: 逐token的GM->UB->Window拷贝
   - 按专家ID和rank分配token
   - 每个token: expandX → UB → 对端window

4. 加权累加: 在本地window中完成
   - 从各rank的window收集数据
   - Cast到float → Muls(scale) → Add → Cast回原类型
   - bf16特殊处理: 先Cast到float再Add(精度要求)

#### 量化支持

INT8量化模式:
- QuantProcess(): per-block量化(BlockReduceMax → scale计算 → Cast → int8)
- DequantProcess(): 反量化(int8 → float → scale恢复 → 原类型)
- 量化数据布局: 前H字节存int8数据，后scaleNum_个float存scale参数

#### 流水线

Process()的执行顺序:
1. ReduceScatterTrans() -- TP域: 把本卡数据发送到对端TP window
2. BuffInit() -- UB buffer分配
3. SetWaitTpStatusAndDisPatch() -- 等TP域就绪 + EP域AlltoAll发送
4. AlltoAllBuffInit() -- 重新分配UB buffer(更大的)
5. SetStatus() -- EP域: 写状态通知对端
6. WaitDispatch() -- EP域: 等所有rank完成发送
7. LocalWindowCopy() -- 从window读取+加权累加+写出

### 4.1.4 三者对比

#### 复杂度

| 维度 | matmul_all_reduce | all_gather_matmul | moe_distribute_combine |
|------|-------------------|-------------------|----------------------|
| 文件数 | 51 | 5 | 3 |
| arch目录 | 3(31/32/35) | 0 | 0 |
| 入口文件行数 | 590 | 82 | 111 |
| tiling key参数 | 16(旧)+12(APT) | 3 | 4 |
| HCCL类型 | AICPU(32)+CCU(35) | AICPU | Window-based(自实现) |
| 量化变体 | 3种(FP/Quant/WeightQuant) | 0 | 1种(INT8) |
| MatMul库 | mat_mul_v3 | mc2/common/MatmulCompute | 无(自实现向量) |

#### 通信类型决定kernel架构

1. AllReduce(matmul_all_reduce):
   - 计算→通信串行: 必须先算完C再提交AllReduce
   - 按tile提交: 每个tile算完立即Commit，不等所有tile完成
   - 通信对象: 输出C矩阵(N轴)

2. AllGather(all_gather_matmul):
   - 本地计算与通信并行: 先算本卡A*B，同时gather远端A
   - 等gather完成再算远端: Wait(handleId) → 逐rank计算
   - 通信对象: 输入A矩阵(K轴)

3. AlltoAll(moe_distribute_combine):
   - 完全不同的范式: 窗口写入+状态轮询
   - 无HCCL高级API: 直接操作RDMA window
   - 通信对象: token级别的数据重排

#### arch适配的三种策略

1. matmul_all_reduce: arch目录隔离
   - arch31/: 310P的RAC Server + HcclServer
   - arch32/: 910B的AICPU Server + Base类继承
   - arch35/: 950的CCU Server + ASW MatMul + 新TilingData
   - 入口文件通过__CCE_AICORE__条件编译选择

2. all_gather_matmul: 单一实现
   - 仅支持arch32(910B)
   - `#if __CCE_AICORE__ != 310`排除A5
   - 无arch目录，无CCU支持

3. moe_distribute_combine: 运行时分派
   - 入口文件通过`__NPU_ARCH__ == 3510`区分A5
   - A3/A5使用相同的MoeDistributeCombine类
   - A2使用v2版实现(跨目录引用)

#### REGISTER_TILING模式

三种方式:
1. REGISTER_TILING_DEFAULT + REGISTER_TILING_FOR_TILINGKEY:
   - matmul_all_reduce: 7个FOR_TILINGKEY(按MM_TYPE/AICORE_TYPE/FORMAT_B条件)
   - moe_distribute_combine: 3个FOR_TILINGKEY(按ArchTag条件)

2. 仅REGISTER_TILING_DEFAULT:
   - all_gather_matmul: 只有一种TilingData结构体

规则: 如果算子只有一种TilingData，用DEFAULT即可；多种则必须FOR_TILINGKEY。

---

## 4.2 Cross-Operator Pattern Extraction

### 4.2.1 HCCL Server类型跨算子统计

HCCL_SERVER_TYPE_AICPU: 64处/55文件
- 覆盖: op_kernel + op_api两层
- arch32及以下的标准通信方式
- 出现在所有MatMul+Comm v1算子 + MoE v1/v2 + GMM

HCCL_SERVER_TYPE_CCU(HcclServerType::HCCL_SERVER_TYPE_CCU): 63处/50文件
- 覆盖: op_kernel + op_api两层
- arch35(950)专用的CCU直驱通信
- 出现在所有v2算子 + arch35目录 + matmul_allto_all/allto_all_matmul APT

关键发现: AICPU和CCU几乎等量出现，说明当前处于过渡阶段。新算子(v2/APT)全部使用CCU，但旧算子仍保留AICPU路径。

### 4.2.2 同步机制统计

SyncAll: 302处/81文件(*.h)
- MC2模块中最普遍的同步原语
- 含`SyncAll<true>`, `SyncAll<false>`, `SyncAll()` 三种形式

PipeBarrier<PIPE_ALL>: 310处/52文件(*.h)
- 与SyncAll几乎等量
- 更细粒度: 仅同步当前核的所有流水线

区别:
- SyncAll: 跨核同步(所有AIV+AIC全部到达)
- PipeBarrier<PIPE_ALL>: 核内同步(当前核所有pipe完成)

### 4.2.3 条件编译统计

`__CCE_AICORE__`: 21处/8个.cpp文件
- 仅出现在kernel入口文件(.cpp)
- 用于顶层arch分支: ==220(910B), ==200(310P), ==310(950)

`__NPU_ARCH__`: 94处/34文件
- 出现在.h和.cpp中
- 主要用法: `__NPU_ARCH__ == 3510` (Ascend 950/DAV_3510)
- 也出现在op_host的tiling代码中

模式: __CCE_AICORE__用于kernel入口的顶层分派，__NPU_ARCH__用于具体实现中的细粒度条件编译。

### 4.2.4 初步规范提取

#### 强制规范

1. Kernel入口格式: `__global__ __aicore__ void OpName<TplParams>(GM_ADDR inputs..., GM_ADDR workspaceGM, GM_ADDR tilingGM)`
   - 固定参数: workspaceGM和tilingGM必须是最后两个
   - 模板参数: 从tiling key中来

2. REGISTER_TILING: 每个kernel必须在入口函数开头注册TilingData结构体
   - 至少一个REGISTER_TILING_DEFAULT
   - 多TilingData时用REGISTER_TILING_FOR_TILINGKEY

3. workspaceGM空指针检查: `if (workspaceGM == nullptr) return;`
   - matmul_all_reduce:374行有此检查

4. GetUserWorkspace: 从系统workspace获取用户空间
   - `GM_ADDR userWS = GetUserWorkspace(workspaceGM);`

5. if constexpr: 模板参数分支必须用if constexpr而非if

#### 推荐规范

1. arch35新算子使用CCU Server(HCCL_SERVER_TYPE_CCU)而非AICPU
2. arch35使用APT tiling key格式(更面向业务语义)
3. 使用Mc2SyncAll<CoreType>模板封装同步(而非裸调SyncAll)
4. 使用ASW(Adaptive Sliding Window) MatMul块(arch35 mat_mul_v3/arch35/)

#### 反模式

1. 跨版本引用: moe_distribute_combine.cpp引用v2版kernel代码
   - `#include "../moe_distribute_combine_v2/moe_distribute_combine_a2.h"`
   - 正确做法: 提取公共代码到common/

2. 巨型入口文件: matmul_all_reduce.cpp 590行密集条件编译
   - 大量重复的INVOKE宏调用，仅参数不同
   - 可考虑用模板元编程或代码生成减少重复

3. Magic number in状态同步: moe_distribute_combine用0x3F800000表示1.0f
   - 语义不清晰，应定义为命名常量

### 4.2.5 公共kernel工具复用分析

#### 算子级common.h(非mc2/common/)

仅3个算子有独立的op_kernel/common.h:
1. matmul_all_reduce/op_kernel/common.h (357行) -- 最完整
2. all_gather_matmul_v2/op_kernel/common.h (100行) -- 精简版
3. moe_distribute_dispatch_v2/op_kernel/common.h (184行) -- MoE专用工具

共同内容(copy-paste模式):
- 类型别名: `using A_DTYPE = DTYPE_X1` -- 3个文件完全一致
- Mc2CoreType枚举: `ON_CUBE_AND_VECTOR/ON_VECTOR/ON_CUBE` -- matmul_all_reduce和all_gather_matmul_v2完全一致
- Mc2SyncAll<CoreType>模板: 同步封装 -- matmul_all_reduce和all_gather_matmul_v2完全一致
- HcclCombinOpParam结构体: 2个文件定义(matmul_all_reduce版更完整，含windowsIn/windowsOut)

差异内容(算子特有):
- matmul_all_reduce: AntiQuantType枚举、DebugMode枚举、MC2_BUFFER_TYPE枚举、OOMInit、CastBFtoFloatOnAiv0
- all_gather_matmul_v2: BiasType<T>偏特化、MAX_HANDLE/EVENT_ID常量
- moe_distribute_dispatch_v2: Ceil/Align系列对齐工具函数、Histograms频率统计(MicroAPI)、ReduceSum工具

观察: Mc2CoreType + Mc2SyncAll是跨MatMul类算子的共性代码，但未提取到mc2/common/，而是各自copy了一份。这是一个典型的代码重复反模式。MoE系列有自己独立的工具集(对齐函数、直方图统计)，与MatMul系列无交集。

统计佐证: Mc2SyncAll出现12处/6文件，全部在matmul_all_reduce和all_gather_matmul_v2的common.h及arch实现中。

#### mc2/common/kernel工具被引用情况

mc2/common/inc/kernel/下的kernel工具:
- mc2_matmul_compute.h (MatmulCompute<>): 9文件引用 -- all_gather_matmul + matmul_reduce_scatter + arch31算子
- mc2_matmul_block.h (Mc2MatmulBaseBlock): 13文件引用 -- arch31/32的标准MatMul block
- mc2_matmul_block_l2cache.h: l2cache优化的block

mc2/common/new_mc2_mm/kernel/下的新版工具(arch35):
- mc2_mat_mul_asw_block.h (Mc2MatmulAswBlock): arch35 ASW block
- mc2_mat_mul_asw_kernel.h: arch35 ASW kernel
- mc2_quant_batch_matmul.h: 量化BMM
- mc2_quant_batch_matmul_asw_block.h: 量化ASW block

关键模式: arch31/32使用mc2/common/inc/kernel/的MatmulBaseBlock，arch35使用mc2/common/new_mc2_mm/kernel/的AswBlock。这是MatMul执行引擎的arch代际升级。

#### INVOKE宏模式

INVOKE_*_OP_IMPL宏: 43处/32文件
- 封装标准流程: GET_TILING_DATA + 模板类型绑定 + 类实例化 + Init + Process
- 每个算子定义自己的INVOKE宏(不共享)
- 宏参数: (MatMulKernelClass, CoreType, ...)
- 目的: 减少kernel入口文件中的重复代码

宏定义位置:
- matmul_all_reduce: 在arch下各实现.h文件中定义(如arch32/matmul_all_reduce_910_general.h)
- all_gather_matmul: 在入口.cpp中定义
- all_gather_matmul_v2: 在APT入口.cpp中定义(更长更复杂，含mc2InitTiling/mc2CcTiling)

APT版INVOKE宏的新模式:
```
auto tiling = (__gm__ TilingDataType*)tilingGM;
__gm__ void* mc2InitTiling = (__gm__ void*)(&(tiling->mc2InitTiling));
__gm__ void* mc2CcTiling = (__gm__ void*)(&(tiling->mc2CcTiling));
```
arch35 CCU需要从TilingData中提取mc2InitTiling和mc2CcTiling指针，这是AICPU版不需要的额外步骤。

### 4.2.6 计算通信融合编排模式

#### 三种流水线架构

1. 手工编排(matmul_all_reduce家族):
   - Base类定义Process()/InnerProcess()虚方法
   - 每个arch/量化变体重写InnerProcess
   - 典型流程: per-tile循环 { MatMul → PostProc(含Commit通信) → Wait }
   - AIC核做MatMul，AIV核做通信后处理
   - 跨核同步: 用SyncAll/CrossCoreSetFlag+WaitFlag

2. 手工编排 + 本地优先(all_gather_matmul家族):
   - MatmulKernelLocal()先算本卡 → HcclPrepare()发起AllGather → Wait → MatmulKernelGather()算远端
   - 本地计算和通信有overlap机会
   - 轮询调度: `(rankId + j + blockIdx) % rankDim` 保证各core负载均衡

3. mc2_templates框架(matmul_allto_all/allto_allv_quant_grouped_mat_mul):
   - 最新的模板化流水线框架
   - 三种pipeline模板:
     - CommTransCompute: 通信→转置→计算(AlltoAll→Matmul场景)
     - ComputeTransComm: 计算→转置→通信(Matmul→AlltoAll场景)
     - CommTransQuantizeCompute: 通信→转置→量化→计算(带量化的AlltoAll场景)
   - Stage抽象: CommunicationType + TransposeType + ComputationType
   - PipelineBuilder组装各Stage
   - 目前仅matmul_allto_all和allto_allv_quant_grouped_mat_mul使用(2个算子目录)

mc2_templates的核心交互模式(pipeline_template_comm_trans_compute.h):
```
// AIV核: 通信 + 转置
if ASCEND_IS_AIV {
    commStage_->Process(index);
    SyncAll<true>();
    transStage_->Process(index);
    CrossCoreSetFlag + WaitFlag  // 通知AIC
}
// AIC核: 等通知后计算
if ASCEND_IS_AIC {
    CrossCoreWaitFlag(9);
    computeStage_->Process(index);
}
```
这是MC2算子的核心范式: AIV驱动通信和数据搬运，AIC执行MatMul计算。

4. 窗口自实现(moe_distribute_combine/dispatch家族):
   - 不使用HCCL高级API
   - 直接操作RDMA窗口写入 + 状态位轮询
   - 流程: SetStatus → WaitDispatch → LocalWindowCopy
   - 无MatMul计算，仅数据重排 + 向量运算(Scale/Add/Cast)

#### AIC/AIV核分工模式

ASCEND_IS_AIC/ASCEND_IS_AIV: 出现399处/126文件(*.h)
- 是MC2模块kernel代码中最普遍的模式
- AIC核: Cube计算(MatMul)、PipeBarrier<PIPE_ALL>
- AIV核: 通信发起、数据搬运(MTE)、向量后处理(Vector)
- 混合核: SyncAll同步AIC+AIV → 协作完成一个tile

#### HCCL初始化模式

GetHcclContext: 93处/60文件
- 100%覆盖所有涉及通信的kernel

arch32初始化: `hccl_.Init(GetHcclContext<0>())`
arch35初始化: `hccl_.InitV2(GetHcclContext<0>(), tilingData_)` + `hccl_.SetCcTilingV2(offsetof(TilingHeader, mc2CcTiling))`

InitV2+SetCcTilingV2: 35处/16文件 -- 全部在arch35路径

#### 通信等待模式

hccl_.Wait(): 46处/24文件
- arch32: 单次Wait (一次等所有tile完成)
- arch35: per-tile循环Wait (每tile单独等待CCU完成)

### 4.2.7 APT(Ascend Programming Templates)使用模式

#### APT双入口架构

支持APT的算子同时有两个kernel入口文件:
- `OpName.cpp` -- 旧版入口(arch31/32)
- `OpName_apt.cpp` -- APT版入口(arch35)

当前有APT入口的算子(6个):
1. matmul_all_reduce -- apt.cpp + apt_tiling_key.h
2. matmul_reduce_scatter_v2 -- apt.cpp + apt_tiling_key.h
3. all_gather_matmul_v2 -- apt.cpp + apt_tiling_key.h
4. matmul_allto_all -- apt.cpp (无独立apt_tiling_key.h)
5. allto_all_matmul -- apt.cpp (无独立apt_tiling_key.h)
6. allto_allv_quant_grouped_mat_mul -- apt.cpp (无独立apt_tiling_key.h)

有独立apt_tiling_key.h的(3个): matmul_all_reduce, matmul_reduce_scatter_v2, all_gather_matmul_v2

此外3rd库也有APT版:
- mat_mul_v3/op_kernel/mat_mul_v3_apt.cpp
- batch_mat_mul_v3/op_kernel/batch_mat_mul_v3_apt.cpp
- quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_apt.cpp

#### APT tiling key格式差异

旧版tiling key特征:
- 宏: ASCENDC_TPL_ARGS_DECL(OpName, ...) + ASCENDC_TPL_SEL
- `#if __CCE_AICORE__`条件编译分arch的选择集
- 无ASCENDC_TPL_KERNEL_TYPE_SEL
- 参数偏向实现细节: AICORE_TYPE, MIXND2NZ, SHARE_MM

APT tiling key特征:
- 宏: ASCENDC_TPL_ARGS_DECL(Mc2OpNameApt, ...) + ASCENDC_TPL_SEL
- 无条件编译(APT只用于arch35)
- 使用ASCENDC_TPL_KERNEL_TYPE_SEL指定AIC/AIV核类型: 7文件
- 参数偏向业务语义: COMMDTPYE, SCENARIO_MXFP8, KERNEL_TYPE

APT入口文件特征:
- 不用__CCE_AICORE__条件编译(只在arch35运行)
- ASC_DEVKIT_MAJOR >= 9时用basic_api/kernel_basic_intf.h替代kernel_operator.h
- KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_AIC_1_2)声明混合核类型
- 直接GetHcclContext获取通信上下文(不通过AICPU窗口)

#### REGISTER_TILING统计

REGISTER_TILING_DEFAULT: 59处/44个.cpp文件 -- 100%覆盖率
REGISTER_TILING_FOR_TILINGKEY: 26处/10个.cpp文件 -- 仅多TilingData算子需要

多TilingData的算子:
- matmul_all_reduce: 7个FOR_TILINGKEY (最多)
- matmul_reduce_scatter_v2_apt: 4个
- moe_distribute_combine: 3个
- moe_distribute_dispatch: 3个
- matmul_allto_all_apt: 3个
- allto_all_matmul_apt: 3个

#### GET_TILING_DATA模式

GET_TILING_DATA/GET_TILING_DATA_WITH_STRUCT: 240处/91文件
- 是kernel中获取tiling数据的标准方式
- GET_TILING_DATA: 较旧的宏，直接按指定类型cast tilingGM
- GET_TILING_DATA_WITH_STRUCT: 较新的宏，指定结构体类型 + 变量名

### 4.2.8 Phase 4总结

#### 强制规范(MC2 op_kernel层)

1. Kernel入口签名: `__global__ __aicore__ void OpName<TplParams>(..., GM_ADDR workspaceGM, GM_ADDR tilingGM)`
   - workspaceGM + tilingGM固定为最后两个参数
   - 出处: 44个kernel入口.cpp文件100%遵循

2. REGISTER_TILING_DEFAULT: 每个kernel入口必须注册至少一个TilingData结构体
   - 59处/44文件, 100%覆盖
   - 注册最大的结构体以确保编译器分配足够内存

3. if constexpr分派: 模板参数的所有运行时分支必须用if constexpr
   - 确保编译期消除不需要的代码路径，否则编译报错(NPU不支持运行时模板分支)

4. GetHcclContext<GroupId>(): 获取通信上下文的标准方式
   - 93处/60文件, 100%覆盖

5. AIC/AIV核分工: AIC做Cube计算, AIV做通信+数据搬运+向量运算
   - ASCEND_IS_AIC/AIV: 399处/126文件
   - 这是Ascend架构的基本编程模型

6. GET_TILING_DATA: 从tilingGM获取tiling结构体
   - 240处/91文件

#### 推荐规范

1. arch35使用CCU Server(HCCL_SERVER_TYPE_CCU) + InitV2 + SetCcTilingV2
   - 35处/16文件(arch35路径)

2. arch35使用APT tiling key格式
   - 面向业务语义而非实现细节
   - 6个算子已有APT入口

3. 新算子使用mc2_templates框架编排流水线
   - 当前仅2个算子使用(matmul_allto_all, allto_allv_quant_grouped_mat_mul)
   - 但这是架构演进方向: 从手工编排到模板化编排

4. 使用Mc2SyncAll<CoreType>封装同步
   - 12处/6文件, 限于MatMul类算子
   - 避免裸调SyncAll时遗漏核类型判断

5. workspaceGM空指针检查
   - APT入口中普遍存在: `if (workspaceGM == nullptr) return;`
   - 旧版入口中也存在(matmul_all_reduce.cpp:374)

6. arch35使用ASW MatMul Block替代旧版BaseBlock
   - MatmulAswBlock: 15文件(mc2/common/new_mc2_mm + 3rd/mat_mul_v3/arch35/)
   - MatmulBaseBlock: 13文件(mc2/common/inc/kernel + 3rd/mat_mul_v3/)

#### 反模式

1. Mc2SyncAll/Mc2CoreType跨算子copy-paste
   - matmul_all_reduce/common.h 和 all_gather_matmul_v2/common.h中完全相同的定义
   - 正确做法: 提取到mc2/common/inc/kernel/

2. 跨版本include
   - moe_distribute_combine.cpp: `#include "../moe_distribute_combine_v2/..."`
   - 正确做法: 提取公共代码到common/或建立明确的复用关系

3. 巨型kernel入口文件
   - matmul_all_reduce.cpp: 590行, 密集的#if/#elif + INVOKE宏
   - matmul_all_reduce_apt.cpp: 258行, 类似结构
   - 根因: 量化变体数量爆炸(FP/INT8/FP8/HIF8/MXFP/W4W8 x 通信量化 x pertoken/perblock)

4. 对齐/工具函数重复定义
   - moe_distribute_dispatch_v2/common.h定义了CeilN/AlignN系列
   - mc2/common/中也有类似工具
   - 仓库级common/中也有CeilDiv
   - 三级重复定义

5. HcclCombinOpParam定义不一致
   - matmul_all_reduce/common.h版: 含windowsIn/windowsOut/signalInfo (完整版)
   - all_gather_matmul_v2/common.h版: 仅WorkSpace/WorkSpaceSize/rankId/rankDim (精简版)
   - 同一结构体不同算子有不同字段，可能导致跨算子复用时出错

#### 演进方向

1. 手工编排 → mc2_templates模板化编排
   - 旧算子: 手工Init/Process/Wait
   - 新算子(matmul_allto_all): PipelineBuilder + CommStage + ComputeStage

2. AICPU → CCU
   - 通信控制从CPU代理变为硬件直驱

3. 旧tiling key → APT tiling key
   - 从实现导向(MIXND2NZ/SHARE_MM)到业务导向(COMMDTPYE/SCENARIO_MXFP8)

4. BaseBlock → AswBlock
   - MatMul执行从标准block到Adaptive Sliding Window

5. 单.cpp入口 → 双.cpp入口(旧+apt)
   - arch31/32使用旧入口, arch35使用APT入口
   - 最终目标可能是全面迁移到APT
