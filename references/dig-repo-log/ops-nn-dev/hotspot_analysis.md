# ops-nn-dev 代码热点与结构性风险分析

## 一、缺陷热点统计

### 1.1 总览

- 612条缺陷提交共触及3844个唯一文件
- 80.5%的文件(3095个)仅被1个缺陷提交触及
- 749个文件被多次触及(>=2次)，是真正的缺陷热点

### 1.2 按模块聚合

| 模块 | 缺陷触及总次数(文件*提交) | 特征 |
|------|--------------------------|------|
| foreach | 1452 | 58个算子子目录，热点极度分散，单文件<=4次 |
| matmul | 1185 | 集中在batch_matmul/quant_matmul的tiling和api层 |
| conv | 562 | 集中在conv2d/conv3d的infershape和tiling层 |
| norm | 435 | group_norm_silu和rms_norm_quant为热点 |
| activation | 246 | 分散 |
| index | 233 | scatter和embedding为热点 |
| quant | 192 | flat_quant的kernel代码为热点 |
| pooling | 183 | adaptive_max_pool3d为热点 |

### 1.3 Top 30 业务代码热点文件

排除构建/配置文件(build.sh, cmake/*, classify_rule.yaml, ascendc_config.json等)后：

| 触及次数 | 文件路径 | 模块 |
|---------|---------|------|
| 9 | matmul/batch_mat_mul_v3/op_host/op_api/aclnn_batch_matmul.cpp | matmul |
| 8 | matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp | matmul |
| 8 | matmul/common/op_host/op_api/matmul_util.cpp | matmul |
| 8 | matmul/common/op_host/matmul_common_infershape.cpp | matmul |
| 8 | conv/convolution_forward/op_host/op_api/aclnn_quant_convolution.cpp | conv |
| 7 | matmul/quant_batch_matmul_v4/op_host/op_tiling/arch35/quant_batch_matmul_v4_tiling.cpp | matmul |
| 7 | matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp | matmul |
| 7 | matmul/quant_batch_matmul_v3/op_host/op_tiling/quant_batch_matmul_v3_tiling.cpp | matmul |
| 7 | matmul/mat_mul_v3/tests/ut/op_host/test_matmul_v3_tiling.cpp | matmul |
| 7 | conv/convolution_forward/op_host/op_api/aclnn_convolution.cpp | conv |
| 6 | quant/flat_quant/op_kernel/flat_quant_cube.h | quant |
| 6 | matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/weight_quant_batch_matmul_v2_tiling.cpp | matmul |
| 6 | matmul/quant_batch_matmul_v3/op_host/op_tiling/quant_batch_matmul_v3_tiling_base.h | matmul |
| 6 | matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp | matmul |
| 6 | conv/common/op_host/conv_forward_infershape.cpp | conv |
| 5 | quant/flat_quant/op_kernel/tensor_utils.h | quant |
| 5 | matmul/weight_quant_batch_matmul_v2/op_kernel/weight_quant_batch_matmul_v2_apt.cpp | matmul |
| 5 | matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/weight_quant_batch_matmul_v2_reg_base_common.h | matmul |
| 5 | matmul/weight_quant_batch_matmul_v2/op_host/op_api/aclnn_weight_quant_batch_matmul_v2.cpp | matmul |
| 5 | matmul/quant_batch_matmul_v4/op_host/op_tiling/quant_batch_matmul_v4_msd_tiling.cpp | matmul |
| 5 | matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_apt.cpp | matmul |
| 5 | matmul/quant_batch_matmul_v3/op_host/op_tiling/quant_batch_matmul_v3_tiling_base.cpp | matmul |
| 5 | matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/adaptive_sliding_window_tiling.cpp | matmul |
| 5 | conv/conv3d_backprop_input_v2/op_host/op_tiling/conv3d_backprop_input_v2_tiling.cpp | conv |
| 5 | conv/conv3d_backprop_input_v2/op_host/op_tiling/common/conv_backprop_input_context_utils.cpp | conv |
| 5 | conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/conv3d_backprop_input_v2_inner_product_tiling.cpp | conv |
| 5 | conv/conv3d_backprop_input_v2/op_host/op_tiling/arch32/conv3d_backprop_input_v2_base_tiling.cpp | conv |
| 5 | conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_basic_block_tiling.cpp | conv |
| 5 | conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_v2_api_tiling.cpp | conv |
| 5 | matmul/batch_mat_mul_v3/tests/ut/op_host/test_batch_mat_mul_v3_tiling.cpp | matmul |

matmul模块占Top 30中的20席，是绝对的缺陷震中。

### 1.4 根因类别频次(从defect_analysis.md提取，归一化后)

| 排名 | 频次 | 占比 | 根因类别 |
|---:|---:|---:|:---|
| 1 | 37 | 7.3% | 条件判断/控制流缺陷 |
| 2 | 37 | 7.3% | 构建系统/CMake缺陷 |
| 3 | 37 | 7.3% | 编译告警/代码规范 |
| 4 | 29 | 5.7% | 数据类型缺陷 |
| 5 | 28 | 5.5% | 内存/buffer管理缺陷 |
| 6 | 24 | 4.7% | 边界条件处理缺陷 |
| 7 | 24 | 4.7% | 整数溢出 |
| 8 | 23 | 4.5% | 计算逻辑错误 |
| 9 | 21 | 4.1% | 平台适配缺陷 |
| 10 | 20 | 3.9% | 配置遗漏 |
| 11 | 18 | 3.5% | API/接口缺陷 |
| 12 | 17 | 3.3% | 流水线/硬件同步缺陷 |
| 13 | 17 | 3.3% | UT/测试缺陷 |
| 14 | 16 | 3.1% | 复制粘贴错误 |
| 15 | 14 | 2.8% | 日志/错误处理缺陷 |
| 16 | 13 | 2.6% | 空指针/资源安全 |
| 17 | 13 | 2.6% | tiling逻辑缺陷 |
| 18 | 11 | 2.2% | include路径错误 |
| 19 | 10 | 2.0% | 输入校验缺陷 |
| 20 | 7 | 1.4% | 功能回退/Revert |
| 21 | 7 | 1.4% | 命名/拼写错误 |
| 22 | 5 | 1.0% | 代码搬迁/迁移缺陷 |
| 23 | 4 | 0.8% | 脚本缺陷 |

Top 3并列(各37次, 7.3%)：条件判断/控制流、构建系统/CMake、编译告警/代码规范。

---

## 二、热点文件结构性风险审查

### 2.1 matmul模块 (Top缺陷震中)

#### aclnn_batch_matmul.cpp (9次触及)

[严重] 死代码 -- 空tensor分支永不生效(约282-293行)
- `CreateBatchMatmulGraphImpl`中先判断空tensor创建`BatchmmEmptyTensorGraph`，但紧接着无条件覆盖为`BatchMatmulExecBmmOpGraph`，缺少`else`
- 空tensor输入走正常计算路径，可能产生非预期行为

[中等] 数据类型支持遗漏
- DTYPE_SUPPORT_LIST仅含FLOAT/FLOAT16/BF16，缺INT8等量化类型

#### matmul_v3_base_tiling.cpp (8次触及)

[低等] GetTotalSize(约1764-1770行)溢出风险已消除
- `m * k * aDtype`所有操作数均为uint64_t，乘法在uint64_t域完成，不存在int32截断问题

[中等] tiling参数边界 -- baseMNK为空时越界(约1556-1585行)
- `minLoadSize = -1`(UINT64_MAX哨兵值)，但`calBaseMNK`为空时`calBaseMNK[0]`越界

[低等] 复制粘贴 -- CheckMMTilingDataIsVaild(约2666-2668行)
- 日志字符串与实际校验字段不匹配：`CheckNumberIsValid(runInfo_.stepM, ..., "runInfo_.baseK")`

#### matmul_util.cpp (8次触及)

[中等] 整数除法后ceil无效(约356行)
- `std::ceil(mDim / align128)` -- 整数除法先截断再ceil，结果等于`mDim / align128`，应用CeilDiv

[中等] CheckBatchDimBroadcast索引越界(约1600行)
- 用`batch1DimNum`作为`batch2`的索引偏移，`batch1DimNum != batch2DimNum`时可能越界

[低等] 代码重复 -- ExecMmOpWithBias vs MatmulCommonProcess(约1094-1366行)
- 大段重复逻辑，修改一处易遗漏另一处

#### quant_batch_matmul_v4_tiling.cpp (7次触及)

[严重] 复制粘贴 -- 错误日志打印错误dtype(约170-172行)
- 校验y(output)的dtype时，错误信息打印`bDtype`(B矩阵类型)而非`cDtype`(输出类型)

[中等] batch累乘无溢出保护(约445行)

#### aclnn_quant_matmul_v5.cpp (7次触及)

[中等] 平台扩展性 -- A8W4Float硬编码仅支持DAV_3510(约538-548行)
- 新NPU架构需手动添加else分支

[低等] const_cast修改输入tensor类型(约345-346行)
- 校验函数中通过`const_cast`修改输入tensor的dtype，违反函数语义契约

#### quant_batch_matmul_v3_tiling.cpp (7次触及)

[中等] 整数溢出 -- int32乘法计算L1/L0C空间(约1072-1074行)
- `mt.baseM * mt.baseK * dtypeSize`三个int32相乘，base块较大时溢出

[中等] SpiltForWorkSpaceLimit除零(约1296行)
- `WORKSPACE_LIMIT / blockDim`无blockDim==0保护

[低等] CalcUsedL1AndUBSize参数类型为int32_t(约1115行)
- L1大小在某些平台可能超INT32_MAX

#### weight_quant_batch_matmul_v2_tiling.cpp (6次触及)

[严重] 类型转换导致比较错误(约248行)
- `static_cast<int64_t>(fusedDimValue)` -- fusedDimValue是uint64_t，超INT64_MAX时转为负数，比较`> MAX_INT32`结果为false，漏检

[中等] GetBatchSize中batch累乘无溢出检查(约96行)

#### aclnn_matmul.cpp (6次触及)

[中等] size_t减法下溢(约147行)
- `size_t loopDims = dimNum - 2` -- dimNum < 2时下溢为极大值

[中等] CheckWeightNzStorageShape维度乘积溢出(约418-428行)

### 2.2 conv模块

#### aclnn_convolution.cpp (7次触及)

[严重] All函数实现调用了Any(行266-274)
- 模板函数`All`递归调用`Any`而非`All`自身
- `CHECK_PARAM_ALL_EQ`和`CHECK_PARAM_ALL_GTE`宏语义错误：只检查第一个参数和任一后续参数

[中等] ConstructPad静默丢弃异常pad值(行604-628)
- `oldPad.size()`不符合预期时pad被设为0，无warning/error

[中等] PointWiseKernelBeyondLimits中负维度乘uint64_t(行790-798)
- 动态shape场景下dim=-1乘以uint64_t产生巨大正数

#### aclnn_quant_convolution.cpp (8次触及)

[中等] 平台分支覆盖 -- TemporaryLimitChecker(行910-933)
- switch/case仅覆盖ASCEND910B和ASCEND910_93，新平台直接报错

[中等] 输出shape计算中stride除零依赖前置check(行345-359)
- check流程与计算流程耦合，重构时除零风险暴露

#### conv_forward_infershape.cpp (6次触及)

[中等] GetConvOutShapeRangeNeedInfer中stride<=0时outRange未赋值(行1087-1134)
- 只打log并return，outRange[idx]保留默认pair(0,0)，后续推导出错误shape range

### 2.3 quant模块

#### flat_quant_cube.h (6次触及)

[中等] 精度损失(行89-94)
- `shape.K * shape.M`结果转float(23位尾数)再除法，值超2^24时精度丢失。int64_t溢出风险在flat_quant实际场景中较低

[中等] L1 buffer总量无运行时校验(行38-45)
- 5个tensor分配在L1中，无check总和是否超出L1_SIZE=512KB
- MM_DOUBLE_MODE下calM = 2*Mceil，buffer需求翻倍

[中等] 硬件同步事件ID复用(行466-471)
- 所有DEvent使用同一对EVENT_ID4/EVENT_ID5

#### tensor_utils.h (5次触及)

[中等] UB/L1大小硬编码(行44-54)
- `UB_SIZE = 192*1024`, `L1_SIZE = 512*1024`仅适配特定芯片

[中等] CalReduceMax尾部数据丢失(行177-188)
- `repeatTimes = len >> 7`截断余数，`len % 128 != 0`时尾部元素未参与ReduceMax

[中等] CalReduceMaxOne中mask为0(行230)
- `colSize % BASE_SIZE == 0`时Max的mask参数为0，硬件行为未定义

### 2.4 norm模块

#### group_norm_silu_tiling.cpp (3次触及)

[严重] Lcm中a*b溢出(行73-76)
- `return a * b / Gcd(a, b)` -- 应改为`a / Gcd(a, b) * b`避免中间结果溢出

[严重] remainUbSize uint64_t下溢(行372-373)
- 减法结果为负时下溢成巨大正整数，导致maxProcessSize过大、UB溢出

[中等] loopNum用整数除法而非CeilDiv(行409)
- elemNum不能被processSize整除时尾块数据漏算

[中等] numGroups字段语义被复用(行388)
- `set_numGroups(gammaPerCoreRoundUp)`覆盖原始numGroups值

#### rms_norm_quant.cpp (3次触及)

[严重] beta搬入缺MTE2->V同步(行108-111)
- gamma搬入有完整SetFlag/WaitFlag，beta搬入缺少，可能读到未完成搬入的数据

[中等] 平台宏条件不一致(行188 vs 245)
- FastCompute用`__NPU_ARCH__ == 3003`，ComputeSquareSum用`__CCE_AICORE__ == 100`，覆盖不同芯片组合

[中等] calc_buf_偏移越界风险(行77, 181)
- OFFSET_WORKSPACE=3的偏移已超出BUF_FACTOR=3分配区域，仅靠+32字节余量

### 2.5 foreach模块

foreach热点极度分散(58个算子子目录，单文件<=4次)，本质是批量模板代码同步修改。

[低等] infershape注册ops::前缀不一致(foreach_infershape.cpp)
- 前52个算子用`ops::InferShape4ForeachCommon`，后5个(ForeachSigmoid等)缺少`ops::`前缀
- 风格不一致反映批量添加时审查缺失

[低等] tiling测试占位断言
- test_foreach_abs/cos/atan_tiling.cpp中只断言tiling_key值，有`// todo check tiling result`注释，未验证tiling数据正确性

### 2.6 index模块

#### aclnn_scatter.cpp (3次触及)

[中等] int64->int32窄化截断(行483等)
- `static_cast<int32_t>(shape.GetDim(0))`比较时，dim超INT32_MAX会截断导致比较错误

[低等] CheckDimRange边界逻辑与前置扩维有交叉依赖(行248-256)

#### aclnn_embedding_dense_backward.cpp (3次触及)

[中等] IsComputeByV2多阈值组合判断(行194-218)
- 涉及芯片版本/gradRow/embeddingDim/numWeights/scaleGradByFreq的复杂决策树，是边界条件bug温床

### 2.7 pooling模块

#### adaptive_max_pool3d_big_pool.h (3次触及)

[中等] startIndex乘法溢出(行140-153)
- `idx * inLen`在高维场景下可能溢出int64_t

[中等] NaN处理复杂度高
- isnan通过reinterpret_cast手动拆解浮点位模式，fp16和fp32/bf16走不同分支

#### dequant_swiglu_quant_tiling_base.cpp (3次触及)

[中等] baseRowLen_*baseColLen_ uint32_t溢出(行415)
- 乘法在uint32_t域完成后才提升到uint64_t

[中等] GetBufferNumAndDataLenPerUB中quantMode/dtype分支不完整(行343-378)
- 不匹配任何已知dtype时singleDataSize默认为1，导致dataLenPerUB过大

---

## 三、结构性风险汇总

### 3.1 按严重程度统计

| 严重程度 | 数量 | 分布 |
|---------|------|------|
| 严重 | 7 | matmul(2), conv(1), quant(0), norm(3), index(0), pooling(0), foreach(0) |
| 中等 | 28 | matmul(12), conv(4), quant(5), norm(3), index(2), pooling(2) |
| 低等 | 8 | matmul(4), conv(1), foreach(2), index(1) |

### 3.2 风险模式聚合

| 风险模式 | 出现次数 | 典型文件 |
|---------|---------|---------|
| 整数溢出/精度损失(int32/uint32乘法、累乘无保护、float截断) | 11 | quant_batch_matmul_v3_tiling, flat_quant_cube.h, group_norm_silu_tiling |
| 条件分支不完整(平台/dtype/else缺失) | 6 | aclnn_quant_matmul_v5, aclnn_quant_convolution, dequant_swiglu_quant_tiling_base |
| 复制粘贴错误(日志字段/函数调用错误) | 4 | matmul_v3_base_tiling, quant_batch_matmul_v4_tiling, aclnn_convolution(All调Any) |
| buffer/内存边界(L1/UB无校验、偏移越界) | 4 | flat_quant_cube.h, rms_norm_quant, group_norm_silu_tiling |
| 硬件同步缺失/不一致 | 3 | rms_norm_quant(beta同步), flat_quant_cube.h(事件ID复用), rms_norm_quant(平台宏不一致) |
| 类型转换错误(窄化截断、uint->signed) | 3 | weight_quant_batch_matmul_v2_tiling, aclnn_scatter, aclnn_matmul |
| 除零保护不足 | 3 | conv_forward_infershape, quant_batch_matmul_v3_tiling, group_norm_silu_tiling |
| 死代码/语义错误 | 2 | aclnn_batch_matmul(空tensor分支), aclnn_convolution(All/Any混淆) |

### 3.3 关键发现

1. matmul是ops-nn-dev的绝对缺陷震中：Top 30热点文件中占20席，量化矩阵乘(quant_batch_matmul_v3/v4, weight_quant_batch_matmul_v2)尤为集中。这与revert分析中matmul占45%的发现一致。

2. 整数溢出是当前代码中最普遍的残留风险(12处)，横跨matmul/conv/quant/norm四个模块。根因：int32类型的tiling参数相乘、batch维度累乘、buffer偏移计算均缺少溢出保护。

3. 8个严重风险中有4个可被静态分析工具发现：All调Any(类型检查)、Lcm溢出(模式匹配)、uint64_t下溢(unsigned减法检查)、beta同步缺失(配对分析)。

4. foreach模块虽然聚合触及次数最高(1452次)，但缺陷本质是"批量模板代码同步修改"，单个算子的结构性风险较低。真正需要重点review的是matmul和quant的kernel/tiling层。

5. 硬编码硬件参数(UB_SIZE, L1_SIZE)在quant/flat_quant中出现，是跨平台移植的定时炸弹。
