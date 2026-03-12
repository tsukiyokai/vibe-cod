# ops-transformer-dev 代码热点与结构性风险分析

## 一、缺陷热点文件 Top-30

按被缺陷修复提交触及次数排序（788条缺陷提交中各文件被触及次数）：

| 排名 | 文件 | 触及次数 | 模块 |
|------|------|---------|------|
| 1 | cmake/custom_build.cmake | 23 | Build |
| 2 | attention/prompt_flash_attention/.../tiling_v2.cpp | 19 | PFA |
| 3 | cmake/func.cmake | 17 | Build |
| 4 | mc2/matmul_all_reduce/op_kernel/matmul_all_reduce.cpp | 14 | MC2 |
| 5 | gmm/grouped_matmul/.../grouped_matmul_tiling.cpp | 14 | GMM |
| 6 | CMakeLists.txt | 14 | Build |
| 7 | mc2/moe_distribute_dispatch_v2/.../tiling.cpp | 13 | MC2/MoE |
| 8 | cmake/obj_func.cmake | 12 | Build |
| 9 | classify_rule.yaml | 12 | Build |
| 10 | build.sh | 12 | Build |
| 11 | tests/test_config.yaml | 11 | Test |
| 12 | mc2/moe_distribute_combine_v2/.../combine_v2.h | 11 | MC2/MoE |
| 13 | mc2/moe_distribute_combine_add_rms_norm/.../kernel.h | 11 | MC2/MoE |
| 14 | mc2/matmul_all_reduce/.../quant_tiling_910_95.cpp | 11 | MC2 |
| 15 | attention/flash_attention_score_grad/.../regbase.cpp | 11 | FAG |
| 16 | gmm/grouped_matmul/.../aclnn_grouped_matmul.cpp | 10 | GMM |
| 17 | attention/incre_flash_attention/.../tiling_v2.cpp | 10 | IFA |
| 18 | attention/common/op_kernel/arch32/fia_kernel_nonquant_mla.h | 10 | FIA/MLA |
| 19 | mc2/moe_distribute_dispatch_v2/.../dispatch_v2.h | 9 | MC2/MoE |
| 20 | mc2/moe_distribute_combine_v2/.../tiling.cpp | 9 | MC2/MoE |

模块分布: Build系统(6/20) > MC2/MoE(7/20) > Attention(5/20) > GMM(2/20)

## 二、构建系统热点 (5文件, 3308行)

### cmake/custom_build.cmake (23次)
- 847行
- 风险1: BUILD_OPEN_PROJECT与非BUILD_OPEN_PROJECT两条分支大量重复代码。aclnn/aclnnInner/aclnnExc三组target创建、aclnn源码生成循环、opbuild命令生成几乎是三份拷贝，修一处漏一处。
- 风险2: 第540-561行target_sources/target_link_libraries在if/else块外无条件执行，引用的`${OPHOST_NAME}_opapi_obj`等target仅在BUILD_OPEN_PROJECT=ON时存在，依赖`$<TARGET_EXISTS:...>`防护，去掉guard即崩溃。
- 风险3: 第330-346行硬编码`".*attention.*"`和`".*moe_inplace_index_add_with_sorted.*"`特殊算子名正则匹配，算子增减时容易遗忘更新。

### cmake/func.cmake (17次)
- 846行
- 风险1: `add_modules_sources`和`add_modules_sources_with_soc`两个macro代码重复率约80%。
- 风险2: 两者是macro而非function，局部变量(如SOURCE_DIR)会污染调用者作用域。
- 风险3: `add_bin_compile_target`(501-738行)237行长函数，含深层嵌套循环和无注释的"group"格式解析(`1-0`字符串拆分为step/start_index)。
- 风险4: `A5_OPS_BLACK_LIST`每个元素带末尾分号如`"mla_prolog;"`，不加分号的元素无法被`list(FIND ...)`匹配。

### CMakeLists.txt (14次)
- 607行
- 风险1(已确认bug): 第60行变量名拼写错误`INDXE`(应为INDEX)。`list(FIND ARCH_DIRECTORY "arch32" INDXE)`存入INDXE，但第61行`if(NOT INDEX EQUAL -1)`检查的是INDEX(上方foreach残留值)。arch22是否被追加完全取决于上次循环残留状态。
- 风险2: 第289-296行硬编码特定算子名和路径，绕过统一发现机制。
- 风险3: 第322-607行BUILD_OPS_RTY_KERNEL块(285行)与custom_build.cmake几乎平行实现，增减feature需两处同步。

### cmake/obj_func.cmake (12次)
- 697行
- 风险1: `add_opapi_modules`/`add_infer_modules`/`add_tiling_modules`三个function的include_directories大量重复，新增公共include需3处同步。
- 风险2: `AICPU_INCLUDE`变量在variables.cmake第252-264行通过`list(APPEND ...)`扩展后，被第267行`set(AICPU_INCLUDE ...)`无条件覆盖，append白做。
- 风险3: 所有GLOB调用(48/66/73/93-99行)不会在文件增减时自动重新配置。

### cmake/variables.cmake (7次)
- 311行
- 风险1: 第259行include路径使用通配符`*.h`，include_directories期望目录路径而非文件通配符，静默失效。
- 风险2: 第302-311行用`grep -Po`(Perl正则)获取版本号，macOS不支持`-P`选项。
- 风险3: OPAPI_INCLUDE/OP_TILING_INCLUDE/OP_PROTO_INCLUDE三组include列表存在大量重复路径，缺乏公共部分提取。

构建系统小结: 核心问题是custom_build.cmake与CMakeLists.txt(RTY)的平行实现、func.cmake两个macro的高重复、variables.cmake三组include列表的冗余。23次缺陷触及的根因是"修一处漏一处"的结构性问题。

## 三、Tiling文件热点 (5文件, ~15000行)

### prompt_flash_attention_tiling_v2.cpp (19次)
- 4762行
- 风险1: 巨型单文件4762行，`RunBigKernelTilingWithParams`含20+串行校验步骤。
- 风险2: workspace计算`batchSize * gSize * headNumSize * kvSplitPart * vHeadSize * sizeof(float)`六值连乘，大batch+大head场景溢出风险。
- 风险3: `CeilDivision`和`AlignUp`在除数为0时静默返回0，下游使用错误tiling参数可能导致kernel访存越界。
- 风险4: `dVBasicBlock`在dSize>512时保持为0(未赋值)，导致`bmm2ResBlockSize=0`，workspace计算偏小。
- 风险5: `PFATilingDataconvert`100+行逐字段手工搬运，新增字段容易遗漏。

### grouped_matmul_tiling.cpp (14次)
- 2103行
- 风险1: `SixteenAlign`的int64_t版本`a & ~15`对有符号数的行为在C++20前是实现定义的。
- 风险2: `AlignUp`对num2==0返回0而非报错，`ratio = float(curTaskNum) / AlignUp(uint32_t(curTaskNum), aicNum)`在aicNum=0时产生NaN。
- 风险3: `IsFixedAxisMoveCondition`中FIXAXISMOVE_K1/K2/N1/N2等magic number与kernel侧寄存器分配紧耦合。

### moe_distribute_dispatch_v2_tiling.cpp (13次)
- 1833行
- 风险1: window size计算`maxBs * tokenNeedSize * epWorldSize * localMoeExpertNum * DOUBLE_DATA_BUFFER`在大集群(epWorldSize=768)场景溢出风险。
- 风险2: `globalBs / epWorldSize`计算maxBs，epWorldSize=0时除零无防御。
- 风险3: socVersion通过字符串比较("Ascend910_95")路由，添加新硬件型号时极易遗漏分支。
- 风险4: if分支数(69)远大于else分支数(19)，大量条件缺少else fallthrough处理。

### incre_flash_attention_tiling_v2.cpp (10次)
- 4351行
- 风险1(已确认bug): `SetLayoutTypefaRun`中map有重复key: `IfaLayout::BSH_BSND`映射了两次(分别到LAYOUT_BSH和LAYOUT_BSND)，std::map覆盖前值，BSH_BSND实际映射到LAYOUT_BSND而非LAYOUT_BSH。
- 风险2: `RunBigKernelTiling`中`EmptyTensorProcess() != ge::GRAPH_SUCCESS`时才走正常tiling流程，"失败才继续"的控制流极易误读。
- 风险3: `CheckUbSpace()`返回`ge::graphStatus`但函数体`return false/true`，bool到graphStatus隐式转换导致语义反转(false=0=GRAPH_SUCCESS)。
- 风险4: `UpdateTilingKeyConfig()`的else分支只打LOGE但没return错误码，函数返回void，调用链无法感知不支持的参数组合。

### fused_infer_attention_score_tiling.cpp (8次)
- 2002行
- 风险1: `GetQueryD`中BSH layout分支`queryD = queryH / numHeads`，numHeads未校验是否为0。
- 风险2: layout字符串"BSH"/"BNSD"/"TND"等硬编码散布在20+个函数中，缺少集中enum映射，拼写错误无编译期保护。
- 风险3: 路由函数`TilingFusedInferAttentionScore`400+行mega-function同时处理IFA/PFA/FAI三条路径。

Tiling文件小结: 核心系统性风险: (1) 整数溢出(5个文件的workspace/buffer计算都涉及多维度连乘); (2) 静默失败的除零防御(AlignUp/CeilDivision返回0); (3) 手工字段搬运(TilingDataconvert); (4) magic number分散无集中定义。

## 四、Kernel/Host文件热点 (9文件)

### matmul_all_reduce.cpp (14次)
- 585行
- 风险1: 310P与910B两套分支通过`#if __CCE_AICORE__ == 220`隔离，各含20+个`if constexpr`分支。新增量化模式需同步修改两处。
- 风险2: 宏参数传递(`INVOKE_QUANT_BATCH_MATMUL_DEQUANT_BF16_IMPL_COMM_INT8`等)嵌套在条件编译内部，参数顺序/类型依赖调用点的`using`定义，跨文件一致性难以静态验证。

### moe_distribute_combine_v2.h (11次)
- 1318行
- 风险1: `WaitDispatch`中用float比较判断状态完成，浮点精度误差可能导致永不满足条件，且无超时保护，异常时死循环hang住。
- 风险2: `epOffset * hAlignWinSize_`地址计算，两个uint32乘积在大规模MoE场景可能溢出。
- 风险3: `LocalWindowCopy`循环体超130行，GM地址通过`base + offset1 + offset2`重计算，难以审计一致性。

### moe_distribute_combine_add_rms_norm.h (11次)
- 1321行
- 风险1: 与combine_v2.h约80%代码相似但存在关键差异(HcclOpResParam vs HcclOpParam, GetWinAddrByRankId实现不同)。修bug不同步到另一侧就会行为不一致。
- 风险2: 缺少v2中的`bufferNum_`和shared expert相关成员，若启用shared expert + AddRmsNorm组合将触发未定义行为。

### quant_matmul_all_reduce_tiling_910_95.cpp (11次)
- 839行
- 风险1: `GetDynamicQuantTempBuffSize`中`procRows = ubSize / ubDenomQuant`，ubDenomQuant为0时除零。
- 风险2: `GetWorkspaceSize`中`rankNum * M * N * sizeof(type)`在大模型场景可能溢出uint64。
- 风险3: `GetTilingKey`使用`GET_TPL_TILING_KEY`宏按位组装tilingKey，参数顺序必须与kernel侧严格一致，仅靠人工保证。

### flash_attention_score_grad_tiling_...regbase.cpp (11次)
- 4202行（本仓最大单文件）
- 风险1: 函数调用链深度超5层(CalcleDeterParam -> CalcleTNDDeterParam -> CalcleTNDCausalDeterParam -> ...)，每层含大量数学计算和条件分支。
- 风险2: `std::copy`到固定大小数组`deterPrefix0/1/2`，源vector大小取决于运行时batch数，可能写越界。
- 风险3: `GetTilingKey`组装18个参数到uint64 tilingKey，参数位域排列通过宏完成，增删参数必须同步kernel侧。
- 风险4: `(optiling::DtypeEnum)4/5/6`强转magic number，枚举定义变更将silent错误。

### fia_kernel_nonquant_mla.h (10次)
- 1026行
- 风险1: `Init`中workspace每个buffer的offset依赖前一个buffer的`sizeof(type) * size`(类型链half->float->half交替)。tiling侧修改任一buffer大小但kernel侧未同步，所有后续buffer地址错位且不会报错。
- 风险2: 模板参数爆炸(layout/dtype/PA/rope多维度)，PA+MLA+非标准layout交叉场景测试覆盖可能不足。

### memory_copy.h (8次)
- 2775行
- 风险1: 8+种格式的OffsetCalculator特化，新增格式需同时修改特化和所有Copy类分支。
- 风险2: `SafeStrideCopy`处理`dstStride > UINT32_MAX`的退化路径，假设dstStride是字节单位，非标准类型(int4b_t等)的sizeof可能不符合预期。
- 风险3: 语义混淆的接口命名: `GetStrideBlockSize()`返回的是stride而非blockSize。

### aclnn_grouped_matmul.cpp (10次)
- 2440行
- 风险1: V1/V2/V3/V4/V5/WeightNz共6个`GetWorkspaceSize`入口，参数子集不同但主体逻辑相似，新增参数/修check需同步6处。
- 风险2: `TransWeightToNz`中310P/非310P分支的break/continue控制流语义不直观(只检查第一个weight就break vs 检查所有)。

## 五、已确认的存量bug

1. CMakeLists.txt:60-61 -- `INDXE`变量名拼写错误导致arch32的检测逻辑使用了错误变量(上次循环残留值)
2. incre_flash_attention_tiling_v2.cpp -- `SetLayoutTypefaRun`中`BSH_BSND`重复key导致映射到错误layout
3. incre_flash_attention_tiling_v2.cpp -- `CheckUbSpace()`返回bool而声明为ge::graphStatus，语义反转
4. variables.cmake:267 -- `set(AICPU_INCLUDE ...)`无条件覆盖前序`list(APPEND ...)`结果

## 六、跨文件系统性风险模式

### 模式1: 代码克隆/平行实现
- cmake/custom_build.cmake vs CMakeLists.txt(RTY分支): 285行平行实现
- cmake/func.cmake两个macro: 80%重复
- moe_distribute_combine_v2.h vs _add_rms_norm.h: 80%相似但有关键差异
- aclnn_grouped_matmul.cpp 6个版本入口
- PFA/IFA各自独立的TilingDataconvert

影响: "修一处漏一处"是23次cmake缺陷和11次MC2 kernel缺陷的根因。

### 模式2: 整数溢出/除零
- workspace/buffer大小计算涉及4-6个维度连乘，普遍缺少溢出检查
- AlignUp/CeilDivision在除数为0时静默返回0
- 地址计算(uint32乘积)在大规模场景溢出

涉及文件: 几乎所有tiling文件和kernel文件。

### 模式3: 跨文件隐式一致性约束
- tilingKey按位组装(host) vs 模板参数解析(kernel): 顺序必须严格一致
- workspace内存布局(host计算offset) vs kernel读取offset: sizeof链必须匹配
- 宏参数顺序(matmul_all_reduce.cpp) vs 宏定义: 仅靠人工保证

影响: tilingKey不匹配是本仓高频缺陷模式之一(阶段2分析中~20+条)。

### 模式4: 硬件平台分支不完整
- socVersion字符串比较路由("Ascend910_95")
- `#if __CCE_AICORE__ == 220`预处理分支
- A2/A3/A5平台常量分散定义

影响: 新增硬件型号时多处遗漏是常见缺陷来源。
