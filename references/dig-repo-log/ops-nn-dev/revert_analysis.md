# ops-nn-dev Revert专项分析

## 概览

ops-nn-dev仓库共发现20个Revert提交（含3个qbmm循环revert），时间跨度2025-08 ~ 2026-02。

| 统计项 | 数值 |
|--------|------|
| Revert总数 | 20 |
| 独立事件数 | 17（qbmm 3次算1个事件） |
| 24h内回退 | 8 |
| 72h内回退 | 13 |
| 5个月+才回退 | 1 |
| 涉及matmul系列 | 9 |
| 涉及norm系列 | 4 |
| 修改公共基础设施 | 5 |

## 逐条分析

### 9b3d2d76b - revert small case optimization in mmv3
- 日期: 2026-02-05
- 原始提交: 3695e56b1 (cv并行方案matmul性能优化, 2025-09-16)
- 涉及模块: matmul/mat_mul_v3 tiling决策逻辑
- Revert原因: 小shape场景走入V_PARALELL_ND2NZ优化模板，workspace初始化不完整导致脏数据参与计算
- 缺陷逃逸类型: 功能缺陷
- 存活时间: ~5个月
- 教训: 性能优化新增快速路径时必须确保workspace等共享资源初始化完整；小shape边界条件需覆盖内存残留场景

### 74cdae15d - Revert NLLLossGrad performence optimize
- 日期: 2026-01-08
- 原始提交: e67aa277f (NLLLossGrad performence optimize, 2026-01-07, MR!3666)
- 涉及模块: loss/nll_loss_grad tiling+kernel大规模重构
- Revert原因: 同时修改TilingData结构体布局(字段重排+类型缩窄uint64->uint32)、kernel模板化、分核策略变更，导致host与device端数据解析不匹配
- 缺陷逃逸类型: 功能缺陷/设计缺陷
- 存活时间: 1天
- 教训: 大规模重构应拆分为多个小提交逐步验证；"性能优化"不应同时改变数据结构布局和计算逻辑

### 8aa5be33e - revert (文档同步)
- 日期: 2026-01-08
- 原始提交: 0725e9c2f (sync nn, 2026-01-07, MR!4344)
- 涉及模块: activation/elu_grad_v2, activation/softmax_v2, quant/quantize 的aclnn文档
- Revert原因: 文档同步将产品支持表格过度精简，遗漏了Atlas推理/训练系列等仍在支持的产品信息
- 缺陷逃逸类型: 需求变更/文档准确性
- 存活时间: 1天
- 教训: 文档同步需对照完整产品支持矩阵；"sync nn"这类模糊commit message不利于审查

### 502f8ac92 - Revert "DTS2025120868273 c04 with innerbatch"
- 日期: 2026-01-06
- 原始提交: 8bab37ba4 (DTS2025120868273 c04 with innerbatch, 2026-01-06, MR!4018)
- 涉及模块: conv/common, conv/conv2d_v2 (9文件, +381/-40行)
- Revert原因: C04模式下L1内存对齐计算顺序错误 -- `AlignB(size, C0) * innerBatch`应为`AlignB(size * innerBatch, C0)`，先对齐再乘会导致L1空间分配不足
- 缺陷逃逸类型: 功能缺陷
- 存活时间: 当天
- 教训: 特殊数据排布(C04)上叠加新特性(innerbatch)时内存对齐计算需格外小心；DTS问题单修复也需充分测试

### 2e1ef4b41 - revert baddbmm dts
- 日期: 2025-12-22
- 原始提交: ec4484582 + c10991e38 (修复aclnnbaddbmm接口精度问题, 2025-12-19, MR!2888/!3012)
- 涉及模块: matmul/batch_mat_mul_v3, matmul/common (11文件, -415行)
- Revert原因: fp16/bf16->fp32精度修复路径实现不完整，在910_95/910b等不同平台分别出现功能错误和精度不达标
- 缺陷逃逸类型: 功能缺陷 + 性能回退
- 存活时间: 3天
- 教训: 精度修复需全平台验证；改变算子输出dtype是侵入性变更，对上下游有连锁影响

### fe16c5570 - Revert "layernormv4功能补齐"
- 日期: 2025-12-10
- 原始提交: c3b8b716d (layernormv4功能补齐, 2025-12-09, MR!2839)
- 涉及模块: norm/layer_norm_v4 (22文件, -2917行)
- Revert原因: 3个新tiling模板+完整kernel一次性合入，同时禁用SingleReadTiling(`return false`)并收窄TransposeTiling路由条件，导致部分场景无模板可选
- 缺陷逃逸类型: 设计缺陷
- 存活时间: 1天
- 教训: 大规模功能补齐应分批合入(先加新模板不启用路由，再逐步切换)；一次性禁用现有模板+扩展平台支持是高风险操作

### 9cc8c69c5 - revert 2691 (dlopen legacy ophost)
- 日期: 2025-12-08
- 原始提交: 23144a91d (dlopen legacy ophost, 2025-12-06, MR!2691)
- 涉及模块: matmul/fused_quant_mat_mul (仅1行)
- Revert原因: dlopen重构中为fused_quant_matmul添加的`TilingPrepareForOpCache`依赖与该算子不兼容，导致编译/链接错误
- 缺陷逃逸类型: 编译失败
- 存活时间: 2天
- 教训: 大规模基础设施重构需对每个受影响模块单独验证编译链接

### ce7121e1f - revert addRmsNorm (发布分支同步)
- 日期: 2025-12-04
- 原始提交: 5af6b8741 (addRmsNorm Change, r1.25.0分支cherry-pick)
- 涉及模块: norm/add_rms_norm
- Revert原因: master上2b00825d3已revert的改动被cherry-pick到r1.25.0发布分支，需同步回退
- 缺陷逃逸类型: 功能缺陷(同2b00825d3)
- 存活时间: N/A(发布分支保护)
- 教训: 合入发布分支前需确认对应master提交的稳定性

### e1fdbe6e8 - Revert "matmulv3支持选择分组累加方式计算"
- 日期: 2025-12-04
- 原始提交: b94bb78a0 (matmulv3支持选择分组累加方式计算, 2025-12-03, MR!2513)
- 涉及模块: common/inc/op_api/op_api_def.h, matmul/batch_mat_mul_v3, matmul/common
- Revert原因: 新增枚举值`FORCE_GRP_ACC_FOR_FP32=4`时将`USE_HIGH_PREC_MODE`从4改为5，破坏了已有调用方的ABI；接口参数语义从bool改为int64_t造成不兼容
- 缺陷逃逸类型: 设计缺陷
- 存活时间: 1天
- 教训: 公共API枚举值扩展应追加而非插入；接口参数语义变更需评估所有下游影响

### 2b00825d3 - revert//addRmsNorm Change
- 日期: 2025-12-02
- 原始提交: 4efcade5a (addRmsNorm Change, 2025-12-02, MR!2592)
- 涉及模块: norm/add_rms_norm (tiling+kernel+UT)
- Revert原因: 引入全局变量`addRmsNormSocVersion`(线程不安全)；新增13个tiling字段改变数据结构布局；新增bfloat16的multi_n路径未充分测试
- 缺陷逃逸类型: 功能缺陷/设计缺陷
- 存活时间: 当天
- 教训: 避免全局变量存储运行时状态；大量tiling参数变更需确保与所有kernel路径兼容

### 76593c3a2 - revert//ascendc_assert整改
- 日期: 2025-11-26
- 原始提交: 8c1c70474 (matmul ascendc_assert整改, 2025-11-25, MR!2159)
- 涉及模块: matmul/mat_mul_v3, quant_batch_matmul_v3/v4, weight_quant_batch_matmul_v2
- Revert原因: `ascendc_assert`(函数)统一替换为`ASCENDC_ASSERT`(宏)，但两者行为不等价(编译条件/错误处理逻辑差异)，导致多个算子用例执行失败
- 缺陷逃逸类型: 编译/功能缺陷
- 存活时间: 1天
- 教训: API名称大小写变更(函数->宏)语义可能不同，必须确认等价性；跨模块统一整改需全量测试

### 2d6f9cb49 - Revert "[MatMul] modify range of shape to transdata"
- 日期: 2025-11-21
- 原始提交: c9dcb2f95 ([MatMul] modify range of shape to transdata, 2025-11-18, MR!2007)
- 涉及模块: matmul/common/op_host/op_api/matmul_util.cpp (1文件)
- Revert原因: IsNdToNzOnTheFly中修改transdata插入条件(内轴<128且非16对齐时跳过)，虽平均12%收益但10%shape劣化，生产环境命中了劣化case
- 缺陷逃逸类型: 性能回退
- 存活时间: 3天
- 教训: 性能优化存在明确劣化比例时不应直接合入；"平均有收益"不等于"所有场景都有收益"

### 0799defc7 - Revert "change gather_v2 l0"
- 日期: 2025-11-19
- 原始提交: a7055b2e5 (change gather_v2 l0, 2025-10-30, MR!1691)
- 涉及模块: index/gather_v2 (op_api + UT)
- Revert原因: 新增`isPreprocessed` OP_ATTR改变了算子属性签名，AICPU侧属性名列表变更是breaking change，与已有tiling/kernel不兼容。且代码风格重构与功能变更混在同一提交
- 缺陷逃逸类型: 功能缺陷(接口不兼容)
- 存活时间: 20天
- 教训: OP_ATTR变更是接口变更，需tiling/kernel同步修改；格式重构与功能变更不应混在同一提交

### 6f830ee8c - Revert "modify name of addRmsNormDynamicQuant and the binary patch"
- 日期: 2025-11-13
- 原始提交: e8491f1a2 (modify name of addRmsNormDynamicQuant, 2025-11-12, MR!1739)
- 涉及模块: norm/add_rms_norm_quant binary.json (1文件, +1736/-560行)
- Revert原因: binary配置JSON中的bin_filename或simplified_key与实际编译产出的二进制文件不匹配，导致算子无法正确加载预编译kernel
- 缺陷逃逸类型: 功能缺陷
- 存活时间: 1天
- 教训: binary配置修改必须与实际编译产出严格对应；算子名称变更需端到端验证加载链路

### ac976d4b5 - mmv3, bmmv3, gemmv2 tiling key revert
- 日期: 2025-09-30
- 原始提交: 729189a56 + 5400ae82f (mat_mul_v3/gemm_v2/batch_mat_mul_v3 tilingkey重构, 2025-09-25, MR!547/!737)
- 涉及模块: matmul/mat_mul_v3, batch_mat_mul_v3, gemm_v2 (11文件)
- Revert原因: tiling key从magic number重构为枚举常量时，kernel侧符号名变化(extern "C"->模板函数)与binary_config注册不兼容；tiling侧和kernel侧对tilingkey理解未同步
- 缺陷逃逸类型: 设计缺陷
- 存活时间: 5天
- 教训: 跨模块重构(tiling+kernel)需双方完全对齐后一起合入；后续10-14通过0bad81147成功重做

### 61a1f1583 - revert identity/identity_n/rank/shape/shape_n
- 日期: 2025-09-27
- 原始提交: c5752277c (move ge_local, 2025-09-26, MR!558)
- 涉及模块: control/identity, identity_n, rank, shape, shape_n + cmake/symbol.cmake (53文件, -2690行)
- Revert原因: 从ge_local迁移5个control算子到nn仓时修改了公共cmake/symbol.cmake和UT基础设施，影响其他算子编译/测试。后续10-10重新添加时不再改symbol.cmake
- 缺陷逃逸类型: 功能缺陷(迁移不完整)
- 存活时间: 1天
- 教训: 算子迁移需评估对公共基础设施的影响；公共cmake修改需充分回归

### qbmm三次Revert循环

时间线:
```
09-12  8c2157971  qbmm原始提交 (MR!113) - 新增quant_batch_matmul_v3/v4, +65519行
09-13  f7188ff70  test delete - 提交后立即发现测试问题
09-15  79623db1a  第一次Revert "qbmm" (MR!410) - 功能问题，从master删除
09-15  7b4afa892  mmv3 wqbmmv2 code fix - revert后修复关联代码
09-21  4f9d6758c  Revert "Revert "qbmm"" (MR!532) - bug修复后恢复到master
09-23  03fe5cfbe  Revert "qbmm" (MR!837) - 从r1.23.0发布分支回退("蓝区回退")
```

#### 79623db1a - Revert "qbmm" (第一次)
- 日期: 2025-09-15
- 原始提交: 8c2157971 (qbmm, MR!113, 2025-09-12)
- 涉及模块: matmul/quant_batch_matmul_v3/v4 + 公共模块(op_util.h, infershape, cmake) - 161文件
- Revert原因: 6.5万行超大提交，修改了公共op_util.h和matmul_common_infershape.cpp，测试不通过
- 缺陷逃逸类型: 功能缺陷
- 存活时间: 3天

#### 4f9d6758c - Revert "Revert "qbmm"" (恢复)
- 日期: 2025-09-21
- 恢复操作，bug修复后在master重新启用，附带约220行修复代码
- 缺陷逃逸类型: N/A

#### 03fe5cfbe - Revert "qbmm" (蓝区回退)
- 日期: 2025-09-23
- 从r1.23.0发布分支删除qbmm，master保留。功能未达发布标准不应包含在release中
- 缺陷逃逸类型: 流程问题

qbmm事件根因:
1. 6.5万行一次性合入无法有效review
2. 公共代码(op_util.h, infershape)与新算子耦合在同一PR
3. master到release分支缺乏feature gate，未成熟功能流入发布分支

### 9de027499 - revert pr 32.
- 日期: 2025-08-26
- 原始提交: 602c72811 (add weight_quant_batch_matmul, 2025-08-25, MR!32)
- 涉及模块: matmul/weight_quant_batch_matmul_v2 + common/include/error_util.h (136文件, -57059行)
- Revert原因: 5.7万行新算子合入时修改了公共error_util.h(改变CUBE_INNER_ERR_REPORT宏格式)，导致其他算子编译失败
- 缺陷逃逸类型: 编译失败
- 存活时间: <16小时
- 教训: 新算子不应修改公共头文件已有接口格式；公共基础设施变更应独立PR先行验证

---

## 模式归纳

### 按缺陷逃逸类型分布

| 类型 | 次数 | 占比 | 典型案例 |
|------|------|------|----------|
| 功能缺陷 | 9 | 45% | workspace脏数据、L1对齐错误、接口不兼容 |
| 设计缺陷 | 4 | 20% | 大规模合入、枚举值插入、全局变量 |
| 编译失败 | 3 | 15% | 公共头文件修改、符号解析、assert等价性 |
| 性能回退 | 2 | 10% | transdata条件、精度修复连锁 |
| 需求/流程 | 2 | 10% | 文档精简、蓝区回退 |

### 五大系统性问题

1. 公共基础设施修改耦合在算子PR中 (5次: qbmm, weight_quant, identity迁移, dlopen, assert整改)
   - 公共头文件(op_util.h, error_util.h, op_api_def.h)和cmake修改影响面远超算子本身
   - 应独立PR先行验证

2. 提交粒度过大 (4次: qbmm 6.5万行, weight_quant 5.7万行, layernormv4 3000行, identity 2690行)
   - 万行级提交无法有效review
   - 应分批合入，先基础设施后算子逻辑

3. 性能优化验证不足 (4次: mmv3 small case, NLLLossGrad, transdata, baddbmm)
   - 性能优化新增快速路径时遗漏边界条件
   - workspace/内存初始化不完整
   - 存在劣化case时仍强行合入

4. 接口变更兼容性评估缺失 (4次: 枚举值插入, OP_ATTR变更, TilingData布局, bool->int64_t)
   - 公共API枚举值插入而非追加
   - 算子属性签名变更未同步tiling/kernel
   - 数据结构字段重排导致ABI不兼容

5. Release分支准入门控缺失 (2次: qbmm蓝区回退, addRmsNorm发布分支同步)
   - master上未充分验证的功能被cherry-pick到release分支
   - 需要独立的feature readiness评审

### 高危模块

| 模块 | Revert次数 | 涉及提交 |
|------|-----------|----------|
| matmul系列 | 9 | mmv3小case, NLLLoss, baddbmm, matmulv3累加, transdata, tilingkey, qbmm×3, fused_quant |
| addRmsNorm | 3 | 全局变量(2b008), 发布分支(ce712), binary配置(6f830) |
| conv2d | 1 | C04 innerbatch |
| gather_v2 | 1 | OP_ATTR变更 |
| layernormv4 | 1 | 3000行功能补齐 |

matmul系列占Revert总量的45%，是最高危模块。

### 回退速度分布

| 存活时间 | 次数 | 说明 |
|----------|------|------|
| 当天 | 4 | 合入后立即发现问题 |
| 1天 | 6 | 次日冒烟测试发现 |
| 2-5天 | 5 | 集成测试/下游发现 |
| 20天 | 1 | gather_v2(延迟发现) |
| 5个月 | 1 | mmv3小case(长尾隐蔽缺陷) |

70%的revert在3天内发生，说明合入前的review和测试门禁形同虚设 -- 问题在合入后而非合入前被发现。
