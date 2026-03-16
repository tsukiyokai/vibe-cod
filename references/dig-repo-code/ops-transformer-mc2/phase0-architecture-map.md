# Phase 0: Global Architecture Survey

## 0.1 ops-transformer 顶层结构

仓库根目录包含8个算子模块 + 支撑设施：

| 模块 | 算子子目录数 | C++文件数 | 主要职责 |
|-----|-----------|---------|--------|
| mc2 | 33+common+3rd | ~1429 | 计算通信融合算子(MatMul+AllReduce/AllGather/AllToAll, MoE分发) |
| attention | 42 | ~1406 | 注意力机制族(Flash/MLA/NSA/稀疏/KV缓存) |
| moe | 28 | ~698 | MoE前端(routing/token排列/expert计算) |
| gmm | 8 | ~411 | 分组矩阵乘法(GMM) |
| posembedding | 11 | ~272 | 位置编码(RoPE/KV缓存归一化) |
| common | 3 | ~113 | 仓库级公共基础设施(tiling基类/通信框架/工具) |
| ffn | 6 | ~105 | 前馈网络(FFN/worker batching) |
| mhc | 2 | ~35 | 多头选择(Sinkhorn) |

支撑目录: cmake/(73), scripts/(78), tests/ut/(100), examples/(34), docs/(78), experimental/(154)

关键顶层文件: CMakeLists.txt(22K), README.md, CHANGELOG.md, version.cmake, build.sh, classify_rule.yaml

## 0.2 MC2 目录树 — 33个算子四层完备度

所有33个算子目录都具备完整的四层结构(op_host/op_kernel/op_graph/op_api) — 100%完备率。

### 按功能分类

MatMul+通信融合(核心):
- matmul_all_reduce (97 files) — 最大最成熟的算子
- matmul_allto_all (65)
- all_gather_matmul_v2 (41), all_gather_matmul (15)
- matmul_reduce_scatter_v2 (40), matmul_reduce_scatter (14)
- allto_all_matmul (40)

MoE分发/组合:
- moe_distribute_dispatch_v2 (39), moe_distribute_dispatch (12), v3 (6)
- moe_distribute_combine_v2 (30), moe_distribute_combine (12), v3 (6)
- moe_distribute_combine_setup (19), moe_distribute_combine_teardown (17)
- moe_distribute_dispatch_setup (14), moe_distribute_dispatch_teardown (14)

融合增强:
- matmul_all_reduce_add_rms_norm (34)
- inplace_matmul_all_reduce_add_rms_norm (15)
- moe_distribute_combine_add_rms_norm (12)

量化通信:
- quant_all_reduce (12)
- quant_grouped_mat_mul_allto_allv (17)
- quant_reduce_scatter (17)

分组矩阵乘+通信:
- grouped_mat_mul_allto_allv (14), allto_allv_grouped_mat_mul (16)
- allto_allv_quant_grouped_mat_mul (23), grouped_mat_mul_all_reduce (12)
- allto_all_all_gather_batch_mat_mul (15), batch_mat_mul_reduce_scatter_allto_all (15)

跨模块:
- attention_to_ffn (11), ffn_to_attention (11)

辅助:
- distribute_barrier (14), moe_update_expert (11)

### 版本变体

| 基础算子 | v1(无后缀) | v2 | v3 |
|---------|-----------|----|----|
| all_gather_matmul | 15 files | 41 files | - |
| matmul_reduce_scatter | 14 files | 40 files | - |
| moe_distribute_combine | 12 files | 30 files | 6 files |
| moe_distribute_dispatch | 12 files | 39 files | 6 files |

v2版本文件数远大于v1 → v2是成熟版(多arch支持)，v1是早期/兼容版
v3版本文件数极小(6) → v3是最新迭代，可能正在开发中

### MC2 特殊目录

- common/ (86 files): inc/(56 headers), src/(22 sources), new_mc2_mm/(8 files)
- 3rd/ (~80+ files): 第三方/遗留实现(mat_mul_v3, batch_mat_mul_v3, add_rms_norm, rms_norm, template_linear_algebra, ops_legacy等)
- tools/: 工具目录
- CMakeLists.txt: 统一构建配置

### 文件规模分布

op_kernel通常是最大层(性能敏感的kernel实现)，op_host第二(tiling/infershape)，op_graph和op_api最小(接口定义)。

## 0.3 ops-transformer/common/ — 仓库级公共基础设施

97个头文件 + 16个实现文件，11个逻辑模块。

### 核心模块

1. tiling_base/ — Tiling参数化框架
   - TilingBaseClass虚基类：8步执行框架(IsCapable→GetPlatformInfo→GetShapeAttrsInfo→DoOpTiling→DoLibApiTiling→GetTilingKey→GetWorkspaceSize→PostTiling)
   - TilingKey十进制编码(10^19偏移)：从低到高位组装Ub0/Ub1/Block/DataType/Format/Sparse等维度
   - AiCoreParams(UB/L1/L0ABC大小)、CompileInfoCommon
   - 枚举体系：AxisEnum(B/N2/G/S1/S2/D), DtypeEnum(FP16/FP32/BF16), LayoutEnum(BSND/SBND/BNSD/TND), SparseEnum(ALL/NONE/CAUSAL/BAND/PREFIX...)

2. fallback/ — 动态库加载与Fallback框架
   - 运行时dlopen("libopapi.so"/"libcust_opapi.so")加载aclnn API
   - 1阶段：EXEC_OPAPI_CMD(aclnn_api, ...)
   - 2阶段：EXEC_OPAPI_PREPARE_CMD + ExecuteOpLaunch (准备期和启动期分离)
   - RAII资源管理：ConvertedParams自动释放aclTensor/aclIntArray

3. external/aclnn_kernels/ — 高频数据处理封装
   - Cast/Transpose/Reshape/Contiguous/Pad/Slice/TransData
   - ContiguousParam记录布局优化信息(mayBroadcast/mayTranspose)
   - IsRegbase()检测DAV_3510/DAV_5102架构

4. kernel/ — 硬件抽象与数据移动
   - ArchType枚举(ASCEND_V220/V200/M200)，HardwareInfo模板(L1=512KB,UB=192KB,L0A/B=64KB,L0C=128KB)
   - BufferType枚举(UB/CB/L0A/L0B/L0C)，AsdopsBuffer模板
   - 12个数据迭代器(gm_to_ub/l1_to_ub/l0c_to_gm/l0c_to_ub等)，全部模板化(ArchType+ElementType)
   - DataFormatT(ND/NZ/ZN/ZZ/NN/VECTOR)，NZ格式专项优化(绕过64K stride限制)
   - SET_FLAG/WAIT_FLAG/PIPE_BARRIER硬件事件同步宏

5. err/ — 统一错误上报
   - OPS_REPORT_VECTOR_INNER_ERR(E89999), OPS_REPORT_CUBE_INNER_ERR(E69999)

6. static/ — 资源注册
   - EXTERN_OP_RESOURCE/AUTO_GEN_OP_RESOURCE宏：声明和生成Tiling/InferShape/Tuning资源
   - StaticSpaceInitializer单例

7. op_graph/ — 图变换
   - REG_OP()宏注册算子到GE图(支持optional输入)

8. common/ — 通用定义
   - MAX_SUPPORT_DIMS_NUMS=8, 精度控制(KEEP_DTYPE/ALLOW_FP32_DOWN_PRECISION)

## 0.4 mc2/common/ — MC2模块级公共代码

86个文件(inc/56 + src/22 + new_mc2_mm/8)

### Tiling层(inc/tiling/, 16个头文件)

核心数据结构:
- mc2_tiling_struct.h：8个TILING_DATA结构体(112处宏展开)，含RCSTiling/Mc2Msg/MC2MatmulV3TilingData等
- formulaic_tiling_datatype.h：MatmulParameters/HCCLInfo参数结构体

性能拟合框架:
- MatmulPerformanceModel：K对齐比例 → Cube利用率模型
- HCCLPerformanceModel：rank维度/kernel类型/soc版本 → 时间-大小映射
- matmul_performance_arch35.h：Arch35(950芯片)专用(baseM/N/K=256/256/128)

计算通信融合编排:
- OneCalcOneCommBase(单通信域)、OneCalcTwoCommBase(EP+TP二层)
- FormPartition：离散(AllGather/ReduceScatter) vs 连续(AllReduce)切分对齐
- one_calc_two_comm_tiling.h：两层通信tiling支持

Tiling约束常量:
- K: [256, 65535]
- M分级: TINY_M=512, SMALL_M=2048, MEDIAN_M=4096
- baseM=128(默认)/256(L2Cache), baseN=256, baseK=64(默认)/128(L2Cache)
- L0_SIZE_DB_ON=32KB, L0C_SIZE_DB_ON=128KB, L1_SIZE=512KB-32B, L2_SIZE_FULL=192MB

多arch适配:
- DAV_2002(A2): CYCLE_PER_MICRO_SEC=1.8K, RankSize={1,2,4}
- DAV_2201(A3): CYCLE_PER_MICRO_SEC=1.8K, RankSize={1,2,4,8}
- DAV_3510(A5): CYCLE_PER_MICRO_SEC=1.65K, RankSize={1...64}

版本管理:
- mc2_opversion_manager.h：操作版本管理单例
- MC2ServerCfg/MC2HcommCfg/RCSTiling标记为@deprecated

### Kernel层(inc/kernel/, 18个头文件)

- mc2_common_def.h：CommAlgType枚举(DEFAULT/FULL_MESH/DOUBLE_RING/SWITCH_WING)
- mc2_matmul_block.h：MatmulBaseBlockMC2(基础块索引/偏移/行优先vs列优先)
- mc2_matmul_compute.h：MatmulCompute模板(全局张量初始化/L2缓存优化)
- mc2_nd_to_nz.h：ND→NZ格式转换kernel
- reduce_sum.h/reduce_sum_cast_fp32.h：AIV reduce_sum kernel
- gm_ub_gm_copy.h：GM→UB→GM双缓冲拷贝(AIV块分配)
- moe_distribute_base.h：HcclOpResParam/HcclOpParam/流和信号管理

### 通用工具(inc/顶层, 12个头文件)

- mc2_matmul_tiling_cfg.h：适配MatMulV3 tiling结果的Mc2MatmulTilingCfg类
- mc2_gen_task_ops_utils.h：KFC服务器初始化/AICPU参数插入
- mc2_common_infershape.h：AllGather/ReduceScatter通用推形
- mc2_hcom_topo_info.h：HCCL拓扑信息查询
- mc2_log.h：日志框架

### new_mc2_mm/(新版矩阵乘法模块, 8个文件)

- NewMc2MatmulTilingCfg类(继承自Mc2MatMulTilingCfg)
- mc2_mat_mul_asw_kernel.h：MC2MatmulAswKernelDerive(ASW块实现)
- mc2_mat_mul_asw_block.h：Mc2MatmulAswBlock(10K+行，最大单文件)
- 量化变体：mc2_quant_batch_matmul.h/mc2_quant_batch_matmul_asw_block.h

## 0.5 mc2/3rd/ — 第三方/遗留代码层

~321个源文件，是MC2的"库层"。

### 基础矩阵乘法库族(~270文件)

- mat_mul_v3 (~100): 标准MatMul，含arch35/ASW/StreamK/FullLoad等优化
- batch_mat_mul_v3 (~90): 批量MatMul，迭代批处理和自适应滑窗
- quant_batch_matmul_v3 (~50): 量化BatchMatMul，含Checker验证
- weight_quant_batch_matmul_v2 (~60): 权重量化v2，MSD/Custom/SplitK多种Tiling
- grouped_matmul: 分组MatMul(MoE场景)，CubeOnTheFly高级实现

### 归一化库族

- add_rms_norm: 残差+RMSNorm组合(op_host/add_rms_norm_tiling.h)
- rms_norm: RMSNorm kernel
- norm_common: 通用归约kernel库(ReduceSum等)

### 基础设施

- common/: matmul_common_infershape.h(通用InferShape)、tiling_cache.h(LRU缓存,500条上限)、hash/lock/cube_util
- template_linear_algebra: CATLASS式C++17模板元编程(Tensor/Layout/GEMM/GEMV)，编译期计算零运行时开销
- ops_legacy: 遗留代码兼容(dlsym动态加载，weak symbol降级)

### 依赖关系

算子目录 → 3rd/(mat_mul_v3等) → 3rd/common(tiling_cache, infershape) → 仓库级common/
CMake加载顺序：先算子目录，后3rd和common(MC2_COMPILE标志需在算子后设置)

## 0.6 构建系统分析

### 顶层CMakeLists.txt

- C++17标准(CMAKE_CXX_STANDARD 17)
- SOC→Arch映射: ascend310p→arch20, ascend910b→arch32, ascend910_93→arch32, ascend950→arch35, ascend950_c→arch38
- arch32自动包含arch22支持
- 构建模式: BUILD_OPEN_PROJECT(自定义包) vs BUILD_OPS_RTY_KERNEL(回黄host)

### MC2构建流程

两阶段加载:
1. 先加载所有算子目录(排除3rd/common) — 设置SUB_MC2_COMPILE/MC2_OPT标志
2. 后加载3rd和common — 通过MC2_COMPILE条件控制编译

关键构建宏:
- MC2_COMPILE: 全局MC2编译开关(mc2/CMakeLists.txt→向上传递顶层)
- SUB_MC2_COMPILE: 子模块开关(每个op_host/CMakeLists.txt→PARENT_SCOPE)
- OP_MC2_ENABLE: 触发opmaster_ct_gentask编译(gen_task.cpp)
- MC2_OPT: 优化开关
- MC2_EXCEPTION_HANDLER: 异常处理(moe_distribute_combine_v2/v3等)

### 算子级CMakeLists.txt标准模式

```cmake
# BUILD_OPEN_PROJECT路径:
target_sources(op_host_aclnnInner PRIVATE <op_name>_def.cpp)
add_modules_sources(OP_API_INDEPENDENT ON OP_MC2_ENABLE ON OPTYPE <op_name>)
set(SUB_MC2_COMPILE TRUE PARENT_SCOPE)
set(MC2_OPT ON PARENT_SCOPE)
set(<op_name>_depends mc2/common mc2/3rd PARENT_SCOPE)
```

### 架构特定编译选项

| SOC | 编译选项 | 说明 |
|-----|---------|------|
| ascend950 | --cce-auto-sync=off -DENABLE_CV_COMM_VIA_SSBUF=true | 禁自动同步，启SSBUF通信 |
| ascend910b | --cce-auto-sync=off | 禁自动同步 |
| ascend310p | --cce-auto-sync=off -mllvm -cce-aicore-jump-expand=true | 禁自动同步，启跳转展开 |

### 依赖声明

简单算子: `set(<op>_depends mc2/common mc2/3rd PARENT_SCOPE)`
复合算子: 链式依赖，如moe_distribute_combine_add_rms_norm依赖mc2/common + mc2/3rd + mc2/moe_distribute_dispatch + mc2/moe_distribute_combine + v2版本

### 产物

- op_host_aclnn/op_host_aclnnInner/op_host_aclnnExc (SHARED .so)
- kernel binary (per arch)
- 安装到 packages/vendors/${VENDOR_NAME}_transformer/

## 0.7 四层结构数据流映射

以matmul_all_reduce和all_gather_matmul_v2为例，追踪完整链路。

### 用户调用到kernel执行的完整数据流

```
用户代码
  |
  v
[op_api] aclnnMatmulAllReduceGetWorkspaceSize(x1, x2, bias, group, ...)
  | CheckParams()验证dtype/shape → 计算transposeX2
  | 调用 aclnnInnerMatmulAllReduceGetWorkspaceSize()
  v
[op_graph] MatmulAllReduceCalcParamFunc() → CommonKFCMc2CalcParamFunc()
  | REG_OP(MatmulAllReduce) 定义INPUT/OPTIONAL_INPUT/OUTPUT/ATTR
  | 触发tiling计算请求
  v
[op_host] MatmulAllReduceTilingFunc(gert::TilingContext* ctx)
  | 架构分发: DAV_3510→TilingRegistryArch / 其他→TilingRegistryNew
  | MatmulAllReduceTilingBase执行:
  |   GetPlatformInfo() → GetShapeAttrsInfo() → DoMatmulTiling()
  |   → DoAllReduceTiling() → GetWorkspaceSize() → PostTiling()
  | TilingKey十进制编码: MM_TYPE×EMPTY_INPUT×INT8_COMM×ENABLE_L2_CACHE×...
  | PostTiling() memcpy到 context_->GetRawTilingData()
  v
[op_kernel] __aicore__ void matmul_all_reduce(..., tilingGM)
  | GET_TILING_DATA_WITH_STRUCT(TilingDataType, tilingData, tilingGM)
  | #if MM_TYPE==FP_MM → fp_matmul_all_reduce<...>(...)
  | #elif MM_TYPE==QUANT_MATMUL → quant_matmul_allreduce<...>(...)
  | 条件编译+if constexpr双重分支
  | 计算结果 → AllReduce通信 → 写入cGM
  v
用户拿到output tensor
```

### TilingData结构体生命周期

1. 定义: op_kernel/arch32/*_tiling_data.h (如UnQuantMatmulAllReduceTilingData)
2. 初始化: op_host/op_tiling/*_tiling_base.h的Reset()
3. 填充: op_host/op_tiling/*.cpp的DoLibApiTiling()/DoMatmulTiling()/DoAllReduceTiling()
4. 写出: op_host PostTiling() memcpy → context_->GetRawTilingData()
5. 读取: op_kernel的GET_TILING_DATA_WITH_STRUCT宏展开

### 跨层接口约定

| 接口 | 传递方式 | 内容 |
|-----|---------|------|
| op_api→op_graph | aclOpExecutor指针(opaque handle) | 参数+executor |
| op_graph→op_host | gert::TilingContext | shape/attr/platform信息 |
| op_host→op_kernel | RawTilingData buffer + tiling_key | tiling参数+模板选择 |

### TilingKey多态选择机制

- op_host层: REGISTER_TILING_FOR_TILINGKEY("MM_TYPE==QUANT_MATMUL", QuantTilingData) 选择填充哪个TilingData结构
- op_kernel层: REGISTER_TILING_DEFAULT/REGISTER_TILING_FOR_TILINGKEY 选择解析哪个TilingData结构
- kernel内部: if constexpr(MM_TYPE==FP_MM) 编译期分支选择计算路径

### matmul_all_reduce vs all_gather_matmul_v2 对比

| 维度 | matmul_all_reduce | all_gather_matmul_v2 |
|-----|---|---|
| 通信操作 | AllReduce(规约) | AllGather(收集) |
| 量化支持 | 无量化/全量化/伪量化 | FP8/FP16/BF16 |
| 输出 | 1个(y) | 3个(y, gather_out, amax_out) |
| TilingKey维度 | ~8维 | ~3维(简化) |
| Arch分支 | arch31/32/35目录分离 | CCU/AIV模式分离 |
| Tiling模式 | MM+AR分离计算 | MM+AG协同计算 |

## 0.8 Phase 0 综合

### 架构全景

ops-transformer是一个面向Ascend NPU的高性能算子库，MC2模块是其中最复杂的子系统，实现计算(MatMul)+通信(HCCL)的融合。

三层公共基础设施(单向依赖):
```
算子目录(33个) → mc2/3rd(MatMul库族) → mc2/common(性能模型/融合编排) → 仓库级common(框架/硬件抽象)
```

四层代码分层(数据流):
```
op_api(用户入口,参数校验) → op_graph(图定义,任务生成) → op_host(tiling计算,shape推导) → op_kernel(设备执行)
```

### 关键设计决策

1. 四层强制完备: 33/33算子100%具备op_host/op_kernel/op_graph/op_api — 这是架构约束非建议
2. TilingKey驱动多态: 一个十进制编码贯穿tiling选择、TilingData类型、kernel模板分支
3. 编译期展开: 模板参数+if constexpr+条件编译(#if)三重手段，一个算子可产生100+种kernel变体
4. 两阶段API: GetWorkspaceSize(编译期) + Execute(运行期)分离
5. 性能拟合: Matmul和HCCL分别建模，通过性能模型驱动tiling切分的均衡点
6. 多arch适配: SOC→Arch映射 + arch目录分离 + 架构特定编译选项，而非运行时分支

### 算子分类体系

| 类别 | 算子数 | 典型代表 | 特征 |
|-----|-------|---------|------|
| MatMul+通信 | 10 | matmul_all_reduce, all_gather_matmul_v2 | 核心融合 |
| MoE分发/组合 | 10 | moe_distribute_dispatch_v2, _setup/_teardown | 多阶段管道 |
| 融合增强 | 3 | matmul_all_reduce_add_rms_norm | 计算后融合归一化 |
| 量化通信 | 3 | quant_all_reduce | 通信量化 |
| 分组MatMul+通信 | 6 | grouped_mat_mul_allto_allv | MoE/EP场景 |
| 跨模块 | 2 | attention_to_ffn | 模块间数据搬运 |
| 辅助 | 2 | distribute_barrier | 同步控制 |

### 版本演进规律

- v1(无后缀): 早期实现，文件少(12-15个)，少量arch支持
- v2: 成熟版，文件多(30-41个)，完整arch31/32/35支持
- v3: 最新迭代，文件极少(6个)，可能仅arch35，正在开发中
