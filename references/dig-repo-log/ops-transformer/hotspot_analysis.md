# ops-transformer 阶段4: 代码热点与结构性风险分析

## 一、缺陷热点文件统计

### 1.1 按文件频次排序 (Top 20)

| 排名 | 文件路径 | 缺陷触及次数 | 代码层 |
|------|---------|-------------|--------|
| 1 | attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp | 22 | tiling |
| 2 | cmake/custom_build.cmake | 7 | 构建 |
| 3 | build.sh | 7 | 构建 |
| 4 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h | 6 | kernel |
| 5 | attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp | 6 | tiling |
| 6 | attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h | 6 | kernel |
| 7 | mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h | 4 | kernel |
| 8 | gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul.cpp | 4 | API |
| 9 | cmake/func.cmake | 4 | 构建 |
| 10 | attention/incre_flash_attention/op_host/incre_flash_attention_tiling.cpp | 4 | tiling |
| 11 | attention/common/op_kernel/arch35/flash_attention_score_block_vec_base.h | 4 | kernel |
| 12 | mc2/moe_distribute_combine/op_kernel/moe_distribute_combine_a2_layered.h | 3 | kernel |
| 13 | mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.h | 3 | kernel |
| 14 | mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp | 3 | tiling |
| 15 | mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.cpp | 3 | tiling |
| 16 | mc2/allto_all_matmul/op_api/aclnn_allto_all_quant_matmul.cpp | 3 | API |
| 17 | attention/incre_flash_attention/op_host/incre_flash_attention_tiling_v2.cpp | 3 | tiling |
| 18 | attention/fused_infer_attention_score/op_host/fused_infer_attention_score_infershape.cpp | 3 | infershape |
| 19 | attention/flash_attention_score_grad/op_host/arch35/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp | 3 | tiling |
| 20 | attention/flash_attention_score_grad/op_api/aclnn_flash_attention_score_grad.cpp | 3 | API |

### 1.2 按模块聚合

| 模块 | 缺陷触及总次数 | 占比 |
|------|--------------|------|
| attention | 178 | 42.3% |
| mc2 | 78 | 18.5% |
| moe | 56 | 13.3% |
| gmm | 36 | 8.6% |
| cmake/build | 19 | 4.5% |
| posembedding | 16 | 3.8% |
| tests | 9 | 2.1% |
| scripts | 6 | 1.4% |
| common | 6 | 1.4% |
| ffn | 5 | 1.2% |
| 其他 | 12 | 2.9% |

### 1.3 按代码层聚合

| 代码层 | 缺陷触及次数 | 占比 | 说明 |
|--------|-------------|------|------|
| op_host (tiling层) | 180 | 42.9% | tiling参数计算是首要缺陷集中区 |
| op_kernel (算子核心) | 140 | 33.3% | kernel代码是第二大缺陷区 |
| 其他 (含docs/scripts) | 58 | 13.8% | |
| cmake (构建系统) | 19 | 4.5% | |
| op_api (接口层) | 15 | 3.6% | |
| common (公共库) | 5 | 1.2% | |
| tests (测试) | 3 | 0.7% | |

### 1.4 缺陷根因类别 Top 10

| 根因类别 | 出现次数 | 占比 |
|---------|---------|------|
| 构建配置缺陷 | 39 | 16.1% |
| 除零错误(边界条件缺失) | 30+ | 12.4% |
| 条件分支遗漏/缺陷 | 20+ | 8.3% |
| 整数类型错误/溢出 | 15+ | 6.2% |
| 输入校验缺失 | 12+ | 5.0% |
| workspace/buffer计算错误 | 10+ | 4.1% |
| 硬件流水线同步缺失 | 10+ | 4.1% |
| API误用/接口参数错误 | 8+ | 3.3% |
| 赋值遗漏/复制粘贴错误 | 6+ | 2.5% |
| 空tensor场景处理缺失 | 5+ | 2.1% |

---

## 二、热点文件结构性风险审查

### 2.1 prompt_flash_attention_tiling_v2.cpp (22次缺陷, 5163行)

核心功能: PromptFlashAttention算子的host侧tiling参数计算。根据输入shape/dtype/layout及20+个特性开关(PA/MLA/量化/Rope/Prefix等)计算分块策略、多核分配、MatMul配置、workspace大小。

这是整个仓库中缺陷密度最高的单文件，5163行承担了过多职责。

#### 高严重度风险

风险H1 -- int64_t到uint32_t截断导致数据错误 (行2901-2935)
- ParseActualSeqLengths中，actSeqLenData->GetData<int64_t>()的值通过static_cast<uint32_t>()各自截断后再做减法。TND layout下cumulative序列的差值计算会因先截断再减产生错误结果(应先减再截断)。负值输入被截断后变为巨大正数。
- 类别: 整数溢出/截断

风险H2 -- ifaBlockSizeBase被成员变量除法污染 (行2422)
- `ifaBlockSizeBase /= static_cast<int32_t>(dataTypeSize)`: 直接除以dataTypeSize，如果为0则除零。更严重的是如果类实例被复用，该成员变量会被反复除以dataTypeSize导致值越来越小直到为0。
- 类别: 除零 + 状态污染

风险H3 -- GetPFAWorkSpaceSize返回ge::GRAPH_FAILED作为size_t (行4120-4122)
- 函数返回类型是size_t，错误时返回ge::GRAPH_FAILED(枚举值通常为1)。调用方会把1当作合法workspace大小分配1字节，后续kernel执行时越界写入。
- 类别: 类型不匹配 / 错误传播

风险H4 -- 成员变量状态在多次调用间累积 (全类)
- 超过80个成员变量，没有统一的reset机制。如果实例被复用于不同参数的tiling调用，前次遗留状态影响后续调用。特别是enableXXX布尔标志和计算中间值(faRunFlag_, ifaBlockSizeBase等)。
- 类别: 状态污染 / 可重入性

#### 中严重度风险

风险M1 -- quantScale2ShapeSize / n 无零保护 (行1190)
- CheckPostQuantParams中`quantD = quantScale2ShapeSize / n`，n来自queryShapeInfo.n，异常输入路径下可能为0。

风险M2 -- int64_t到uint32_t截断(shape维度) (行545-550, 609-614)
- SetShape中h = n * d的结果是int64_t，当n和d都很大时超出uint32_t范围，截断后产生错误值。

风险M3 -- SetTilingData中4处除法依赖成员变量初始值 (行3130-3133)
- typeByteNum = BYTE_BLOCK / dataTypeSize等，如果CheckIODataType中inputType不在allowedDtypes中，dataTypeSize保持默认初始值(可能为0)。

风险M4 -- sinnerBlocknum除零 (行3880-3881)
- 空tensor场景下seqInnerSize可能为0，导致sinnerBlocknum为0，后续除法除零。

风险M5 -- gSize除零链 (行847, 2727, 5036)
- gSize = nQ / nKV在nQ本身为0时(前面只检查了nQ < 0)，gSize为0，后续多处用gSize做除数。

风险M6 -- GetMaxSeq无空/大小检查 (行398-405)
- 直接访问actualSeqLength->GetData<int64_t>()[0]，未检查GetShapeSize()>0和GetData()非空。

风险M7 -- workspace大小计算整数溢出 (行4127-4186)
- 多处大整数乘法链，uint32/int64混合运算，极端参数下中间结果可能溢出。

风险M8 -- innerPrecise被无条件覆盖导致死代码 (行2516-2529)
- innerPrecise = HIGH_PRECISION后，后续的innerPrecise == HIGH_PERFORMANCE分支永远为false，softmaxDataTypeSize永远不会被设为FLOAT16SIZE。

风险M9 -- allowedDtypes与dataTypeSizeArray长度不匹配 (行326-328, 1355, 1433)
- 5元素 vs 6元素，索引对应关系脆弱，新增类型时极易错位。此模式在文件中重复3次。

风险M10 -- CeilDiv与CeilDivision边界行为不一致 (行163-188)
- 两个功能相同的函数在除数为0时行为不同(返回n1 vs 返回0)，同一文件中共存是维护陷阱。

#### 低严重度风险

- blockTableStr.pop_back()在空字符串时的未定义行为 (行1502-1507)
- strideQ/ratio除法已有保护但负值未考虑 (行3288-3294)

#### 复杂度观察

- 5163行单文件，约80个成员函数
- 20+个互相关联的enableXXX布尔特性开关，交叉约束分散在多个Check函数中
- allowedDtypes查表模式重复3次未提取公共函数
- 5种layout(TND/NTD/BSH/BSND/BNSD)的条件分支在全文反复出现
- 最深嵌套4层(ComputeSplitNBSeq的三重循环+条件)

---

### 2.2 moe_distribute_dispatch_v2_full_mesh.h (6次缺陷, 1611行)

核心功能: MoE分布式dispatch的full mesh通信实现。通过RDMA窗口在多卡间进行token级all-to-all数据分发。

#### 高严重度风险

风险H1 -- moeExpertRankNum_为0时除零 (行366)
- `moeExpertNumPerRank_ = moeExpertNum_ / moeExpertRankNum_`，moeExpertRankNum_ = epWorldSize_ - sharedExpertRankNum_。当所有卡都部署共享专家时为0。elastic scaling场景下这些值来自runtime tensor，不受tiling静态保证约束。

风险H2 -- rankNumPerSharedExpert_为0时取模除零 (行363, 568)
- sharedExpertRankNum_ / sharedExpertNum_计算得到rankNumPerSharedExpert_，结果为0时在568行被用作取模运算除数。

风险H3 -- moeUsedAivNum_为0时除零 (行621)
- `sendNum_ = moeExpertNum_ / moeUsedAivNum_`，moeUsedAivNum_ = aivUsedAllToAll_ - sharedUsedAivNum_，当所有核分配给共享专家时为0。

风险H4 -- 3处无限循环无超时退出 (行933-944, 987-996, 1146-1155)
- GetCumSum、WaitDispatch、WaitCumSumFlag中的while(true)忙等循环没有超时机制。远端rank故障或通信异常时整个AIV核挂死。

#### 中严重度风险

- axisMaxBS_ = globalBS_ / epWorldSizeOriginal_ 缺零保护 (行350)
- 窗口地址偏移计算中uint32乘法可能在32位空间截断后再提升为uint64 (行404-405)
- startId_/endId_/sendNum_/statusCntAlign_ 4个成员变量未初始化 (行254-257)
- elastic scaling路径下epRankId_自引用索引无边界检查 (行321)

#### 复杂度观察

- 1611行单体header，80+成员变量
- 通信/计算/内存管理全混合在一个类中
- Process()三阶段流水线通过共享成员变量隐式通信
- 圈复杂度估计200+

---

### 2.3 fused_infer_attention_score_tiling_v2.cpp (6次缺陷, 1148行)

核心功能: FusedInferAttentionScore算子的host侧tiling。根据运行时参数决定走IFA还是PFA模板路径。

#### 高严重度风险

风险H1 -- GetAttrPointer返回nullptr时直接解引用 (行741-742, 760)
- `*attrs->GetAttrPointer<uint32_t>(ATTR_N_INDEX)` 未检查返回值。相比之下行466-478的对比代码只保存指针不解引用。lseFlag在行760也有同样问题。

#### 中严重度风险

- int64_t到uint32_t强转: GetDim返回-1(无效维度)时转为UINT32_MAX (行159, 213, 265)
- GetDynamicInputShape返回值未检查 (行439-440)
- layout字符串匹配仅覆盖BSH/BNSD/BSND三种，TND/NTD等未覆盖 (行994)

#### 复杂度观察

- DoOpTiling函数单体448行，14+种layout组合的if-else if链
- layout字符串比较分散在多处，没有统一的枚举映射

---

### 2.4 infer_flash_attention_kvcache.h (6次缺陷, 886行)

核心功能: Flash Attention推理阶段的KV cache参数计算。处理不同layout下的query/key/value内存偏移。

#### 高严重度风险

风险H1 -- constInfo.gSize在6处被用作除数无零保护 (行38, 451, 545, 546, 668, 670-671)
- GQA未启用时gSize理论上为1，但初始化异常时为0则多处除零。

风险H2 -- GetKeyCoreOffsetParam中NTD layout分支缺失 (行281-313)
- 只处理BSH、TND和else(BNSD)，NTD走else分支。但NTD的key布局与BNSD不同。对比GetValueCoreOffsetParam有NTD独立分支(行337-346)，两个函数不对称。

#### 中严重度风险

- actualSeqQlenAddr数组越界: bIdx-1在unsigned下溢时为UINT32_MAX (行104, 108, 476)
- 偏移量计算中int32*uint64的乘法链可能中间溢出 (行288, 306, 484)
- enableKVPrefix路径下TND/NTD/BNSD统一处理但布局语义不同 (行365-377)

#### 低严重度风险

- 自赋值语句runParam.actualS1Size = runParam.actualS1Size (行191)
- halfS1RealSize除零依赖||短路求值 (行628-629)

---

### 2.5 moe_distribute_dispatch_v2.h (4次缺陷)

核心功能: MoE分布式dispatch的kernel侧主逻辑。

#### 高严重度风险

- 行329/337: kernel侧无保护除法，tiling参数异常时导致NPU硬件异常

---

### 2.6 aclnn_grouped_matmul.cpp (4次缺陷)

核心功能: GroupedMatMul算子的aclnn API层实现。

#### 中严重度风险

- 输入tensor列表的校验分散在多处，空列表场景下部分路径缺少前置检查

---

### 2.7 incre_flash_attention_tiling.cpp (4次缺陷)

核心功能: Incremental Flash Attention的host侧tiling。

#### 高严重度风险

风险H1 -- increGcd函数b==0时除零 (行125-131)
- `a % b`在b为0时是未定义行为。该函数被tiling分块逻辑调用。

风险H2 -- GetMaxSeqLength未校验tensor数据有效性 (行327-333)
- 直接访问`actualSeqLength->GetData<int64_t>()[0]`，未检查GetData返回nullptr、GetShapeSize()为0。

---

### 2.8 flash_attention_score_block_vec_base.h (4次缺陷)

核心功能: Flash Attention前向的AIV核基础实现。

#### 高严重度风险

风险H1 -- deScaleKvOffset - 1的unsigned underflow (行933/942/948/956/1095/1100/1105/1110/1235/1240/1245/1250)
- 12处相同模式: 当deScaleKvOffset为0时，无符号减法下溢产生巨大值，影响FP8 attention精度。

---

### 2.9 incre_flash_attention_tiling_v2.cpp (3次缺陷, 2500+行)

核心功能: IFA V2的host侧tiling策略。覆盖从输入校验到tiling数据生成的完整流程。

#### 高严重度风险

- increGcd函数b==0除零 (行125-131)，与2.7同源
- GetMaxSeqLength未校验tensor空/null (行327-333)

#### 中严重度风险

- 循环变量int与GetShapeSize()的size_t类型不匹配 (行329)

---

### 2.10 fused_infer_attention_score_infershape.cpp (3次缺陷, 428行)

核心功能: FusedInferAttentionScore的InferShape与InferDataType。

#### 高严重度风险

风险H1 -- BSH layout下numHeadsPtr值为0时除零 (行130)
- 仅检查指针非null，未检查指向值是否为0。

风险H2 -- GetValueD中numKeyValueHeads为0时除零 (行180, 197)
- PA路径和非PA BSH路径都存在。

#### 中严重度风险

- GetQueryBSND返回值被忽略 (行222, 229, 235)，失败时b/s1/n1/d1保持默认值0，静默错误。

---

### 2.11 flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp (3次缺陷, 1500+行)

核心功能: Flash Attention Score反向梯度算子的tiling策略。

#### 高严重度风险

风险H1 -- queryRope在null检查前被解引用 (行597-598)
- 先调用->GetStorageShape()取地址，下一行才检查queryRope != nullptr。keyRope同理(行599-601)。

风险H2 -- headNum/g/n2除零传播链 (行614-631)
- queryDim2 / keyDim2 -> g, headNum / g -> n2, queryDim2 / headNum -> d, valueDim2 / n2 -> d1。上游任一维度为0导致下游连续多次除零。

#### 中严重度风险

- 手动new[]/delete[]存在异常安全问题 (行996)，中间return导致内存泄漏。

---

### 2.12 aclnn_flash_attention_score_grad.cpp (3次缺陷, ~1000行)

核心功能: aclnn API层的Flash Attention Score Grad实现。

#### 中严重度风险

- dDim为0后n2Dim除法链 (行348-362): 有部分保护但语义上dDim==0时继续执行不正确。
- GetSumIntArrayMaxValue未处理空数组 (行136-140): Size()==0时访问[0]越界。

---

### 2.13 flash_attention_score_block_vec_infer.h (3次缺陷, 988行)

核心功能: Flash Attention推理侧的AIV核实现，负责输出初始化、softmax LSE、Flash Decode多核合并。

#### 高严重度风险

风险H1 -- InitGlobalBuffer中workspace指针回退溢出 (行266)
- `workspace -= singleCoreOffset * preloadTimes * (aicIdx + 1)` 三个无符号整数乘积可能超uint64范围导致回绕。回退后指针无下界检查。

风险H2 -- InitOutputSingleCore中uint32减法下溢 (行561-562)
- `totalOutputSize - aivIdx * singleCoreSize`当后者更大时下溢为巨大正数。后续`> 0`判断对无符号数永远为true。

---

### 2.14 attenmask.h (3次缺陷, 607行)

核心功能: Attention Mask的kernel侧实现。处理各种compress模式的mask数据DMA拷入和偏移计算。

#### 中严重度风险

- BoolCopyInRegbase中blockBytes可能为0导致CeilDiv除零 (行69-72)
- MergeBandModeMask中s2BaseSize为0时除零 (行277)
- halfS1RealSize强转uint16后+1可能溢出(65535+1=0) (行280)

---

### 2.15 mc2模块其他热点文件

#### moe_distribute_combine_a2_layered.h (3次缺陷)
- 通信同步busy-wait无超时保护
- 除法运算缺零保护(与dispatch系列同类问题)

#### moe_distribute_combine_v2.h (3次缺陷)
- 空指针解引用风险(tensor数据访问前缺校验)
- 成员变量初始化不完整

#### moe_distribute_combine_v2_tiling.cpp (3次缺陷)
- 除法运算缺零保护
- workspace大小计算中整数溢出风险

#### allto_all_matmul_tiling_910b.cpp (3次缺陷)
- tiling参数计算中的除法链缺零保护
- 条件分支覆盖不完整

#### aclnn_allto_all_quant_matmul.cpp (3次缺陷)
- 输入校验分散，部分路径缺少前置检查
- 空指针解引用风险

---

## 三、跨文件系统性风险模式

### 3.1 除法零保护缺失 (系统性, 覆盖16/20热点文件)

这是整个仓库最突出的系统性风险。除数来源包括:
- shape维度(headNum, n, nKV, gSize, d): 来自用户输入tensor，异常输入可能为0
- tiling参数: 通过tiling data传递，理论上由tiling侧保证非零，但kernel侧无本地校验
- 计算中间结果(sinnerBlocknum, moeExpertRankNum_): 上游计算异常导致为0

典型传播链: H / headNum -> d, H / d -> n2, H / n2 -> d1。一个上游维度为0触发下游连续多次除零。

检查点: 每个除法运算的除数是否来自可信来源? 是否有本地零值校验?

### 3.2 整数类型截断与溢出 (系统性, 覆盖10/20热点文件)

三类子模式:
1. int64_t -> uint32_t截断: shape维度GetDim()返回int64_t，被static_cast<uint32_t>()截断。-1(无效维度)变为UINT32_MAX。
2. uint32_t无符号减法下溢: a - b当b > a时下溢为巨大正数而非负数。特别在kernel代码中因性能使用窄类型(uint16_t, uint32_t)时频发。
3. 乘法链中间溢出: 偏移量/workspace大小计算中多个uint32相乘，中间结果在32位空间溢出后再提升为64位。

检查点: 跨类型赋值是否有范围检查? 无符号减法是否可能下溢? 乘法链是否在最宽类型空间中计算?

### 3.3 条件分支不完整 (系统性, 覆盖8/20热点文件)

三类子模式:
1. Layout分支遗漏: 5种layout(BSH/BSND/BNSD/TND/NTD)的处理中NTD最容易被遗漏。GetKeyCoreOffsetParam vs GetValueCoreOffsetParam的不对称是典型案例。
2. 可选输入null检查时序: 先解引用后判空(flash_attention_score_grad_tiling中queryRope)。
3. 函数返回值忽略: GetQueryBSND等返回ge::graphStatus，调用方未检查导致错误静默传播。

检查点: 新增layout时是否同步更新了所有相关函数? 可选输入是否先判空再使用? 返回错误码的函数调用是否检查了返回值?

### 3.4 状态管理缺陷 (局部但高危, 覆盖3/20热点文件)

主要出现在大型tiling类中:
- prompt_flash_attention_tiling_v2.cpp: 80+成员变量无统一reset，复用实例导致状态污染
- moe_distribute_dispatch_v2_full_mesh.h: 80+成员变量，三阶段流水线通过共享状态隐式通信
- GetPFAWorkSpaceSize返回ge::GRAPH_FAILED(值=1)作为size_t，调用方误认为合法大小

检查点: tiling类实例是否被复用? 成员变量是否在每次调用前重置? 错误返回值的类型是否与函数签名匹配?

### 3.5 并发/通信同步无超时 (局部, 覆盖mc2模块)

mc2模块的分布式通信代码中，while(true)忙等循环缺乏超时退出机制。远端故障或通信异常时AIV核挂死。

涉及文件: moe_distribute_dispatch_v2_full_mesh.h, moe_distribute_combine_a2_layered.h

检查点: 分布式同步循环是否有超时退出? 超时后是否有错误上报机制?

---

## 四、热点文件复杂度指标

| 文件 | 行数 | 估计圈复杂度 | 最深嵌套 | 成员变量数 | 核心问题 |
|------|------|-------------|---------|-----------|---------|
| prompt_flash_attention_tiling_v2.cpp | 5163 | 300+ | 4层 | 80+ | 单文件过大, 职责过多, 20+特性开关交叉 |
| moe_distribute_dispatch_v2_full_mesh.h | 1611 | 200+ | 4层 | 80+ | 通信/计算/内存混合, 状态耦合极紧 |
| flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp | 1500+ | 150+ | 4层 | - | 除零传播链, 手动内存管理 |
| incre_flash_attention_tiling_v2.cpp | 2500+ | 200+ | 5-6层 | - | 多模式状态依赖, 分支嵌套深 |
| fused_infer_attention_score_tiling_v2.cpp | 1148 | 80+ | 3层 | - | 448行单函数, layout枚举爆炸 |
| infer_flash_attention_kvcache.h | 886 | 60+ | 4层 | - | 模板实例化组合数百种, 偏移计算分散 |

---

## 五、风险优先级排序 (建议修复顺序)

### P0 (立即修复 -- 可导致NPU硬件异常或数据错误)

1. flash_attention_score_block_vec_base.h: deScaleKvOffset-1 unsigned underflow (12处)
2. flash_attention_score_block_vec_infer.h: uint32减法下溢导致tailSize错误 (行561)
3. prompt_flash_attention_tiling_v2.cpp: GetPFAWorkSpaceSize返回ge::GRAPH_FAILED作为size_t (行4120)
4. moe_distribute_dispatch_v2_full_mesh.h: moeExpertRankNum_/rankNumPerSharedExpert_/moeUsedAivNum_除零 (行363-621)

### P1 (高优先级 -- 特定配置下触发崩溃)

5. flash_attention_score_grad_tiling: queryRope null检查前解引用 (行597)
6. flash_attention_score_grad_tiling: headNum/g/n2除零传播链 (行614-631)
7. fused_infer_attention_score_infershape: numHeadsPtr/numKeyValueHeads值为0除零 (行130, 180, 197)
8. fused_infer_attention_score_tiling_v2: GetAttrPointer返回nullptr解引用 (行741-742)
9. infer_flash_attention_kvcache.h: gSize在6处做除数无零保护 (行38-671)
10. incre_flash_attention_tiling: increGcd函数b==0除零 (行125)

### P2 (中优先级 -- 边界场景下的潜在问题)

11. prompt_flash_attention_tiling_v2: ParseActualSeqLengths类型截断 (行2901)
12. prompt_flash_attention_tiling_v2: 成员变量状态累积/无reset (全类)
13. prompt_flash_attention_tiling_v2: workspace大小计算溢出 (行4127-4186)
14. moe_distribute_dispatch_v2_full_mesh: busy-wait无超时 (3处)
15. aclnn_flash_attention_score_grad: GetSumIntArrayMaxValue空数组越界 (行136)

### P3 (低优先级 -- 代码质量/维护性)

16. prompt_flash_attention_tiling_v2: CeilDiv/CeilDivision行为不一致
17. prompt_flash_attention_tiling_v2: innerPrecise覆盖导致死代码
18. prompt_flash_attention_tiling_v2: allowedDtypes/dataTypeSizeArray长度不匹配
19. 各tiling文件: layout分支覆盖完整性
