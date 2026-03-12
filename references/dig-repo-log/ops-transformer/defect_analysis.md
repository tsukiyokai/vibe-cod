# ops-transformer 缺陷提交深度分析

## 批次1: 提交 #1-#20 (2026-02-26 ~ 2026-03-02)

---

### fe0bee0d 修复dequant_quant_kvcache算子

非缺陷提交(代码清理)。删除一行已完成的TODO注释，不涉及运行时逻辑。从defect_commits中排除。

---

### 5079260e 修复pfa合轴精度问题
- 根因类别：边界条件缺失(tiling优化路径准入条件不完整)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：PFA合轴优化路径的`CheckPFAMerge`函数缺少关键前置校验。当query序列长度s>256且head维度d不能被64整除时(`d%64!=0`)，合轴路径产生精度问题，但原代码未拦截。
- 修复模式：新增条件`if (queryShapeInfo.s > 256U && (queryShapeInfo.d % 64) != 0) return false;`，不满足对齐约束时回退到不合轴路径。
- 可审查性：中(需理解PFA合轴的数学约束)
- 审查规则建议：tiling优化路径准入条件变更时，要求附带各维度参数的对齐要求文档，并用UT覆盖边界值(d=63/64, s=256/257)。

---

### 6a573aa2 修复ROPEV2中json的可选项设置
- 根因类别：算子描述配置与代码不一致
- 涉及文件：posembedding/rotary_position_embedding/op_host/config/kirin9030/rotary_position_embedding_binary.json, posembedding/rotary_position_embedding/op_host/config/kirinx90/rotary_position_embedding_binary.json, posembedding/rotary_position_embedding/op_host/op_api/aclnn_rotary_position_embedding.cpp
- 缺陷描述：ROPEV2新增`rotate`可选输入，但两个平台的binary JSON配置缺少index=3的`rotate`声明。同时aclnn API缺少`rotate!=nullptr`时的SOC架构校验(仅DAV_2201支持)。
- 修复模式：补全两个平台JSON中`rotate`可选输入声明；aclnn层新增SOC架构校验guard。
- 可审查性：高(JSON与代码参数列表不一致可自动化校验)
- 审查规则建议：新增/修改算子参数时，要求同时更新所有平台JSON配置。CI加入一致性校验。可选参数有平台限制时API入口必须有平台校验。

---

### a882f8c0 alltoallvgmm 修复量化模板转置问题 & aclnn共享转置问题
- 根因类别：变量遮蔽(variable shadowing) + 模板参数位置错误
- 涉及文件：mc2/allto_allv_grouped_mat_mul/op_api/aclnn_allto_allv_quant_grouped_mat_mul.cpp, mc2/allto_allv_grouped_mat_mul/op_kernel/allto_allv_grouped_mat_mul_apt.cpp
- 缺陷描述：两处独立bug。(1) if块内`auto mmWeightOptional = transMmWeightOptional`声明了同名局部变量遮蔽外层参数，if块结束后外层变量未被更新。(2) `QuantGroupedMatmul`模板实例化时`TILINGKEY_GMM_WEIGHT_TRANSPOSE`和`TILINGKEY_MM_WEIGHT_TRANSPOSE`两个模板参数位置填反。
- 修复模式：(1) 去掉`auto`关键字直接赋值。(2) 修正模板参数位置对应关系。
- 可审查性：高(编译器`-Wshadow`可捕获变量遮蔽；模板参数审查可对照签名)
- 审查规则建议：启用`-Werror=shadow`编译选项。多模板参数实例化时要求注释说明每个参数语义，或用具名常量代替裸bool。

---

### c3cf4d96 修复maskshape支持范围
- 根因类别：条件分支逻辑错误(硬编码覆盖动态shape值)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：`CheckMaskTypeAndShape`中当`enableIFAMask=true`时强制将`maskQsSize`硬编码为1，覆盖了从attenMask实际shape读取的值。IFA模式下mask的Q维度不一定为1，导致tiling数据与真实shape不匹配。
- 修复模式：删除`if (enableIFAMask) { maskQsSize = 1; }`整个条件块。
- 可审查性：中(需理解IFA模式下mask shape语义)
- 审查规则建议：看到"读取shape后又用常量覆盖"的模式应重点质疑。硬编码覆盖动态值必须附带注释说明原因。

---

### e8ffb3b9 sfa添加aclnn接口

非缺陷提交(新增接口)。为SparseFlashAttention新增aclnn外壳接口，处理softmaxMax/softmaxSum空指针场景。属于接口设计补全而非bug修复。从defect_commits中排除。

---

### 1216e10a grouped_matmul_finalize_routing：去掉pertoken_scale输入必须存在拦截
- 根因类别：参数校验过严(可选参数被当必选校验)
- 涉及文件：gmm/grouped_matmul_finalize_routing/op_host/grouped_matmul_finalize_routing_infershape.cpp
- 缺陷描述：`ValidateScaleAndBias`中当scale为三维(E,1,N)时，`OP_CHECK_IF`强制要求`pertoken_scale`不为nullptr，但规格定义其为可选输入。不传pertoken_scale时infershape直接报错。
- 修复模式：删除对pertoken_scale的强制判空检查(2行)。
- 可审查性：高(对照算子规格文档即可发现可选参数被强制校验)
- 审查规则建议：通过`GetOptionalInputShape`获取的输入，nullptr应为合法路径。凡对可选输入判空后返回GRAPH_FAILED，需检查是否与规格矛盾。

---

### 984d8a72 修复rotaryDim!=headSize时同步问题
- 根因类别：硬件流水线同步缺失
- 涉及文件：posembedding/rope_with_sin_cos_cache/op_kernel/rope_with_sin_cos_cache_f_bf16.h, posembedding/rope_with_sin_cos_cache/op_kernel/rope_with_sin_cos_cache_fp32.h
- 缺陷描述：RoPE kernel中处理完query转处理key时，DataCopy(MTE2搬入)前只调了`MTE3ToVSync()`缺少`VToMTE2Sync()`。当rotaryDim!=headSize时Vector单元可能仍在处理query，MTE2提前搬入key导致数据竞争。
- 修复模式：在key的DataCopy前、MTE3ToVSync()之后各增加一行`this->VToMTE2Sync()`。bf16和fp32各改2处共4行。
- 可审查性：中(需深入理解Ascend NPU MTE2/MTE3/Vector流水线同步模型)
- 审查规则建议：切换处理不同tensor并复用同一块local memory时，检查DataCopy前是否补全所有必要sync调用。建议lint规则：每个DataCopy前检查是否有覆盖所有写入来源的sync屏障。

---

### 53c21a1d npu_ops_transformer_ext算子与老写法算子隔离

非缺陷提交(构建工程改进)。引入CMake白名单机制控制打包范围，从"扫描所有子目录"改为"只打包白名单算子"。从defect_commits中排除。

---

### f14d0284 [FIA] pse,mask 非对齐场景拷贝越界修复
- 根因类别：非对齐场景DataCopy越界
- 涉及文件：attention/common/op_kernel/arch35/attenmask.h, attention/common/op_kernel/arch35/pse.h, attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h
- 缺陷描述：attenmask/pse的stride模式DataCopy仅检查`totalS2Size%blockBytes==0`但未检查`s2Size`是否也对齐blockBytes，blockLen向上取整后越界。kvcache中GQA场景下actualS1Size计算遗漏乘以gSize。
- 修复模式：(1) attenmask/pse增加`s2Size%blockBytes==0`检查，不满足走逐行拷贝fallback。(2) kvcache中GQA+s1Size==1+非BNSD场景使用`nextTokensPerBatch*gSize`修正计算。
- 可审查性：中(需理解DataCopy stride模式对齐要求和GQA的s1/gSize关系)
- 审查规则建议：使用DataCopy stride参数且涉及CeilDiv取整时，必须同时验证源数据实际宽度也满足blockBytes对齐。否则走逐行拷贝。

---

### 8e9fe171 fix antiquant BNSD_BSND bug
- 根因类别：数据传递链路断裂(结构体字段遗漏)
- 涉及文件：attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp, attention/incre_flash_attention/op_host/incre_flash_attention_tiling_context.h, attention/incre_flash_attention/op_host/incre_flash_attention_tiling_v2.cpp
- 缺陷描述：`IncreFlashAttentionContext`结构体缺少`transposeLayout`成员，`ConvertContextToParamsIFA`未调用`GetTransposeLayout`赋值。BNSD_BSND layout场景下tiling参数传递链断裂，kernel侧收到未初始化的layout信息。
- 修复模式：context新增`uint32_t transposeLayout=0`成员；ConvertContextToParamsIFA返回前赋值；IFATilingDataconvert中传递该值。补全数据传递链路。
- 可审查性：中(需理解tiling参数传递完整链路，单文件审查难发现)
- 审查规则建议：context/params结构体新增字段时，检查所有构造路径是否显式赋值，所有转换函数是否读取该字段。

---

### 151ca802 fix combineARN精度问题
- 根因类别：buffer空间分配不足
- 涉及文件：mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp, mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.h
- 缺陷描述：ReduceSum临时buffer分配`NUM_PER_REP_FP32*sizeof(float)`(256字节)，但H=8192时ReduceSum接口实际需要512字节。空间不足导致写越界，污染相邻buffer，表现为精度问题。
- 修复模式：buffer大小改为`NUM_PER_REP_FP32*sizeof(float)*2`，tiling侧和kernel侧同步修改。附带将gamma加载从循环内提到循环外。
- 可审查性：低(ReduceSum对buffer的最小空间需求是隐式的，需查API文档公式)
- 审查规则建议：调用ReduceSum/ReduceMax等归约API时，要求注释标明buffer大小计算依据(引用API文档公式)，并校验覆盖最大shape下的需求。建议封装CalcReduceSumBufferSize工具函数。

---

### 63f9fff6 fix gmm no quant l2 cache
- 根因类别：提前返回时状态未重置
- 涉及文件：gmm/grouped_matmul/op_host/op_tiling/arch35/grouped_no_quant_matmul_tiling.cpp
- 缺陷描述：`SetDisableL2Cache`中当totalSize<l2Size时直接return，但未将`weightNoL2Cache_`设为false。该字段可能在之前流程中被设为true，导致不需要禁用L2的场景误走双页表路径，950非量化性能劣化。
- 修复模式：early return前补加`weightNoL2Cache_=false;`。1行代码。
- 可审查性：高(Set类函数的early return路径遗漏状态更新，模式清晰)
- 审查规则建议：Set/Configure类函数的所有return路径必须对目标状态变量有显式赋值。审查early return是否遗漏状态重置。

---

### ae4c91e3 追加伪量化责任田

非缺陷提交(配置更新)。仅修改classify_rule.yaml添加文件归属配置。从defect_commits中排除。

---

### 91ef89b4 GroupedmatmulFinalizerouting pertoken量化，weightnz模式aclnn通路

非缺陷提交(新功能)。为GMR算子增加pertoken量化+weightNz模式aclnn通路。PR标签为"新特性"。从defect_commits中排除。

---

### 7fbc7bc3 GMMSQ算子异常场景提交

非缺陷提交(功能增强/参数校验补充)。PR标签"新特性"。但有防御性价值：补充了dequantMode和quantMode必须相等的校验。记录参数一致性校验模式供后续归纳。

---

### 108b4dd3 [experimental FIA] 编译报错修复&&cmakelist整改

非缺陷提交(构建系统整改)。CMakeLists简化、tiling注册统一化、kernel冗余条件分支清理。从defect_commits中排除。

---

### 234c1c8b rope_with_sin_cos_cache fix 950 devide 0 error
- 根因类别：除零错误(tiling计算除数为零)
- 涉及文件：posembedding/rope_with_sin_cos_cache/op_host/rope_with_sin_cos_cache_tiling.cpp
- 缺陷描述：`maxNPerLoopForUb`和`numHeadsForUb`在Ascend950+headDim>8192场景下可能为0。代码4处用这两个变量做除数进行向上取整计算，除零触发CoreDump。
- 修复模式：4处除法前增加三元表达式`var==0 ? 0 : (a+var-1)/var`零值保护。
- 可审查性：高(除零防护是基本审查项)
- 审查规则建议：所有除法运算必须确保除数非零，特别是UB大小计算得来的变量(不同硬件平台UB容量差异可能导致零值)。可自动扫描`/variable`形式除法检查零值保护。

---

### af418dfc 魔鬼数字问题修改

非缺陷提交(代码规范)。将字面量128替换为命名常量NUM_128。静态分析告警消除。从defect_commits中排除。

---

### c2250fc9 修复FAG aclnn非连续场景下获取ShapeSize问题
- 根因类别：API误用(非连续tensor的Size()语义错误)
- 涉及文件：attention/flash_attention_score_grad/op_api/aclnn_flash_attention_score_grad.cpp
- 缺陷描述：`GetInputShapeInfo`中用`query->Size()`获取tensor大小，但Size()返回StorageShape大小。非连续tensor的StorageShape可能大于逻辑元素数。调用发生在转contiguous之前，导致各轴维度推导有误。
- 修复模式：将`query->Size()`改为`queryShape.GetShapeSize()`(基于逻辑shape)，key同理。
- 可审查性：高(Size() vs GetShapeSize()语义差异明确)
- 审查规则建议：aclnn代码中获取tensor元素数优先用`GetViewShape().GetShapeSize()`而非`->Size()`。标记所有`->Size()`调用检查是否在contiguous转换之前。

---

## 批次1统计

总计20条提交:
- 实际缺陷修复: 12条
- 非缺陷(排除): 8条 (fe0bee0d, e8ffb3b9, 53c21a1d, ae4c91e3, 91ef89b4, 7fbc7bc3, 108b4dd3, af418dfc)

缺陷根因初步分类:
| 根因类别 | 数量 | commit |
|---------|------|--------|
| 边界条件缺失/tiling约束 | 2 | 5079260e, c3cf4d96 |
| 参数校验问题 | 1 | 1216e10a |
| 配置与代码不一致 | 1 | 6a573aa2 |
| 变量遮蔽/模板参数错误 | 1 | a882f8c0 |
| 硬件流水线同步缺失 | 1 | 984d8a72 |
| DataCopy非对齐越界 | 1 | f14d0284 |
| 结构体字段/数据传递遗漏 | 1 | 8e9fe171 |
| buffer分配不足 | 1 | 151ca802 |
| 提前返回状态未重置 | 1 | 63f9fff6 |
| 除零错误 | 1 | 234c1c8b |
| API误用(tensor语义) | 1 | c2250fc9 |

---

## 批次2: 提交 #21-#40 (2026-02-26 ~ 2026-02-28)

---

### 28671df1 ROPE 修复kernel侧拦截问题

非缺陷提交(新功能)。大规模新增约2956行，为ROPE算子添加rotate matrix功能支持和V2 API。新增rotate可选输入、aclnnRotaryPositionEmbeddingV2接口、rotate_matrix.h(681行)、tiling文件(497行)。本质是功能扩展+API重构。从defect_commits中排除。

---

### 8a09dcd0 修改classify_rule路径配置错误

非缺陷提交(CI配置修正)。classify_rule.yaml中路径缺少`ops/`前缀，从`ops-transformer/posembedding/...`改为`ops/ops-transformer/posembedding/...`。与ae4c91e3同类，属构建分类配置。从defect_commits中排除。

---

### 14a289de fix constraint of ratio between numHeads and numKeyValueHeads

非缺陷提交(文档修改)。所有修改均在.md文档文件中(共12个文件)，移除"numHeads与numKeyValueHeads比值不能大于64"的约束描述。无运行时代码变更。从defect_commits中排除。

---

### f73c0505 输入异常分析

非缺陷提交(新功能)。为mc2/tools/dump_analysis/诊断工具新增输入异常检测分析能力(check_dis_com、check_mask、check_topk等)，新增754行。属于工具层新功能开发。从defect_commits中排除。

---

### 831ab170 S1G invalidrows bug code review
- 根因类别：数据类型溢出(uint32乘法溢出)
- 涉及文件：attention/common/op_kernel/vector_common.h
- 缺陷描述：`DealInvalidRowsBelow`函数中`dealRowOffset`声明为`uint32_t`，但通过`s1BottomTok * params.gSize`计算，当两值较大时uint32溢出(最大~43亿)，导致偏移量错误、越界写入或逻辑异常。附带问题：while循环条件中`s1 >= s1BottomTok`对uint32_t恒成立，属冗余条件。
- 修复模式：将`dealRowOffset`从`uint32_t`提升为`uint64_t`；移除循环中冗余条件。
- 可审查性：中
- 审查规则建议：kernel代码中偏移量/索引计算变量，审查类型是否足以承载最大值。两个uint32_t相乘或结果用作数组偏移时，建议使用uint64_t。

---

### 967fed57 编译头文件包含路径bugfix

非缺陷提交(编译路径兼容性)。添加`__has_include`预处理条件判断，头文件路径不存在时回退到备选路径。编译期适配，不涉及运行时逻辑。从defect_commits中排除。

---

### f38d4a49 NTD_TND Dsize拦截信息修复

非缺陷提交(错误消息文案修正)。将错误报告中硬编码"NTD"替换为动态变量`layoutStr.c_str()`，仅影响日志输出内容，不改变校验逻辑。从defect_commits中排除。

---

### 613ae40b revert//新增ROPEV2接口支持辅助矩阵输入

非缺陷提交(功能revert)。cann-robot自动执行revert，撤销ROPEV2接口及rotate matrix功能。删除V2头文件、实现文件、tiling文件、rotate_matrix.h，从proto/json/kernel中移除rotate参数。此revert将在阶段3专项分析。从defect_commits中排除。

---

### da61be50 fa infershape修改
- 根因类别：InferShape边界条件逻辑错误
- 涉及文件：attention/flash_attention_score/op_host/flash_attention_score_infershape.cpp
- 缺陷描述：FlashAttentionScore的InferShape中，当key的N2维度计算结果为0时(h2/D1==0，发生在h2<D1场景)，输出attention_out的第3维被错误设置为0。正确值应为N1*D1。输出shape为0导致下游shape不匹配或计算异常。
- 修复模式：将边界条件(N2==0)下的输出shape从硬编码0修正为N1*D1。
- 可审查性：高
- 审查规则建议：InferShape函数中所有`SetDim(idx, 0)`代码路径需重点审查，特别是除法结果为0的边界条件。输出shape为0应极少见，出现时需确认是否确实应为空tensor。

---

### 1065427e fix_transformer_graph_extension

非缺陷提交(新功能/重构)。虽以"fix"开头，实际是为moe_distribute算子添加graph mode(torch.compile)支持：新增GE Converter、将op_module.load()提升到模块级、重写Meta函数签名。从defect_commits中排除。

---

### a44e3e7e 修复使能PA时获取VD错误
- 根因类别：多模式shape解析错误(未区分PA/非PA模式)
- 涉及文件：attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp
- 缺陷描述：PA(PagedAttention)模式下value tensor的storage shape格式不同(BBH 3维或BND1BD0 5维)，但各layout分支统一用`GetStorageShape().GetDim(VALUE_DIM_X)`取valueD，PA模式下得到错误值。
- 修复模式：前置检测是否启用PA(BLOCK_TABLE_INDEX是否存在)，若启用则根据V维度数(3维/5维)用不同公式计算正确valueD；后续各分支用条件选择。
- 可审查性：中
- 审查规则建议：处理多种tensor格式/模式时，所有`GetStorageShape().GetDim()`路径必须考虑各模式下shape语义差异。存在模式标志时，每处shape获取应有模式guard。

---

### 79a2aced fix: fix warnings

非缺陷提交(编译告警消除+代码规范化)。修改7个文件700+行，全部为：代码格式统一(花括号位置)、添加const限定符、删除未使用include/参数、static改static inline。从defect_commits中排除。

---

### 988a83a9 aclnnGroupedMatmulV3接口兼容int8输入,int8输出场景

非缺陷提交(新功能)。PR标签"新特性"。新增V3版本int8输入/输出校验方法(CheckInputAndOutputDtypeForV3Version等)、新增UT测试、更新文档。从defect_commits中排除。

---

### 28daf0ab feature: infer FA, Ascend950, deprecate api

非缺陷提交(新功能/API废弃拦截)。在Ascend950上拦截旧版API(FIAS V1-V4、IFA V1-V3、PFA V1-V2)，升级fallback路径到V5，更新接口文档。从defect_commits中排除。

---

### 5e8817a7 fix bug for TND
- 根因类别：条件判断逻辑错误(多模式统一条件不正确)
- 涉及文件：attention/flash_attention_score/op_host/arch35/flash_attention_score_tiling_varlen.cpp
- 缺陷描述：varlen tiling分核优化判断中，`LEFT_UP_CAUSAL`和`RIGHT_DOWN_CAUSAL`两种sparse mode使用统一条件`actualSeqLenData[i] >= thresholdS2Size && actualSeqLenKvData[i] > thresholdS2Size`。但LEFT_UP_CAUSAL模式下有效计算量取决于`min(seqLen, kvLen)`，不是两者都必须超阈值。导致特定序列长度组合下分核优化被错误跳过或错误开启。
- 修复模式：对LEFT_UP_CAUSAL取min(seqLen, kvLen)与阈值比较，对RIGHT_DOWN_CAUSAL仍用kvLen。分模式特化条件。
- 可审查性：高
- 审查规则建议：同一代码路径处理多个枚举/模式值时，审查每个模式的数学语义是否一致。不同模式语义不同则应设独立条件分支，不用统一条件覆盖。

---

### be6bcad7 【日志优化】gmm算子AlogCheckDebugLevel废弃日志接口替换

非缺陷提交(API迁移)。将4个文件中`AlogCheckDebugLevel`机械替换为`CheckLogLevel`，函数签名/参数/逻辑完全不变。从defect_commits中排除。

---

### 4d3bbf03 编译头文件包含路径bugfix

非缺陷提交(编译路径兼容性)。与967fed57同类，在两个头文件中添加`__has_include`预处理回退路径。从defect_commits中排除。

---

### 498342e0 GQA perblock全量化添加拦截
- 根因类别：输入校验缺失(多维度遗漏)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：`CheckPerblockQuantParams`存在两处遗漏：(1) 缺少对deqScale1、quantScale1、deqScale2、antiquantScale等不支持参数的非空拦截，传入不支持参数时不报错走入未定义行为；(2) 不支持layout列表不完整，原仅拦截TND，实际BNSD_NBSD、BSH_NBSD、BSH_BNSD、BSND_BNSD、BSND_NBSD、NTD也不支持。
- 修复模式：增加额外参数非空检查；将layout拦截从单一枚举扩展为不支持列表匹配。
- 可审查性：中
- 审查规则建议：多量化模式下所有输入参数合法性校验需逐项检查完备性。layout枚举拦截应使用白名单(支持列表)而非黑名单(不支持列表)以减少遗漏。

---

### 0104fc64 修复GQA非量化tiling下沉误拦截
- 根因类别：条件判断逻辑错误(模式标志位遗漏)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：两处逻辑缺陷。(1) `CheckPseShiftTypeAndShape`中pseShift shape校验未考虑`isMaxWorkspace`标志——MaxWorkspace模式下shape用最大值占位，校验条件不适用但被触发，导致合法请求被误拦截。(2) `GetMaxWorkspaceFlag`遗漏对`actualSharedPrefixLen`的空数据指针判断，该tensor存在但数据为空时应判定为MaxWorkspace模式，否则后续访问空指针。
- 修复模式：校验包裹在`!isMaxWorkspace`条件下；补充空数据指针判断。
- 可审查性：中
- 审查规则建议：存在"模式标志位"控制路径时，所有依赖该模式的校验需检查是否在正确模式分支下。新增tensor输入时检查所有引用"同类tensor列表"的地方是否需同步更新。

---

### f5f79a3e 【FAG】fix tnd s1s2 exist zero
- 根因类别：循环内布尔标志覆盖(应为累积)
- 涉及文件：attention/flash_attention_score_grad/op_host/arch35/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp
- 缺陷描述：`GetShapeAttrsInfo`循环中，`tndBaseInfo.isSeqExistZero`使用直接赋值`=`而非累积或`|=`。循环遍历多batch的qLen/kvLen检测零长度序列，但每次迭代覆盖前次结果，仅最后一个batch的判断生效。前面batch出现零长度序列信息被丢失。
- 修复模式：从`= (qLen == 0 || kvLen == 0)`改为`= (isSeqExistZero || (qLen == 0 || kvLen == 0))`。
- 可审查性：高
- 审查规则建议：循环内对布尔标志赋值，若语义为"是否存在某条件"必须用`|=`或`= (old || new)`。直接赋值`=`仅适用于"最终迭代即最终结果"。可编写静态规则：循环体内布尔变量直接赋值(非`|=/&=`)发出警告。

---

## 批次2统计

总计20条提交:
- 实际缺陷修复: 7条
- 非缺陷(排除): 13条 (28671df1, 8a09dcd0, 14a289de, f73c0505, 967fed57, f38d4a49, 613ae40b, 1065427e, 79a2aced, 988a83a9, 28daf0ab, be6bcad7, 4d3bbf03)

缺陷根因分类:
| 根因类别 | 数量 | commit |
|---------|------|--------|
| 数据类型溢出(uint32) | 1 | 831ab170 |
| InferShape边界条件 | 1 | da61be50 |
| 多模式shape解析错误 | 1 | a44e3e7e |
| 条件判断逻辑错误(多模式) | 1 | 5e8817a7 |
| 输入校验缺失(多维度) | 1 | 498342e0 |
| 模式标志位遗漏(误拦截) | 1 | 0104fc64 |
| 循环内布尔标志覆盖 | 1 | f5f79a3e |

## 批次3: 提交 #41-#60 (2026-02-14 ~ 2026-02-26)

---

### fcff1be7 [bugfix] 修复TND问题&修复资料问题
- 根因类别：workspace内存分配大小计算错误(多layout分支遗漏)
- 涉及文件：attention/dense_lightning_indexer_grad_kl_loss/op_kernel/dense_lightning_indexer_grad_kl_loss_base.h
- 缺陷描述：TND layout场景下，`dKeyIndexFloatSzie`计算使用了`bSize * s2Size`，但TND格式下token维度是连续展开的(T维度)，应从`actualSeqLengthsKeyGm`取实际序列长度`t2Size`来计算。错误的workspace大小可能导致内存越界或计算错误。
- 修复模式：增加`if constexpr (LAYOUT_T == DLILayout::TND)`编译期分支，TND场景用实际`t2Size`替代`bSize * s2Size`。
- 可审查性：中
- 审查规则建议：多layout算子中涉及shape维度乘积计算的地方，检查每种layout是否都有正确的处理分支。特别关注workspace/buffer大小计算是否遗漏layout分支。

---

### e46032f7 update error msg process

非缺陷(日志/可调试性改进)。文档示例代码中增加error message获取方法，不涉及运行时功能代码。

---

### 05e3ba28 ifa pfa kernel_operator.h 最后清理

非缺陷(编译优化/重构)。将`kernel_operator.h`大头文件替换为精确的`kernel_vec_intf.h`+`kernel_cube_intf.h`，无运行时行为变化。

---

### e48d4172 fix bug for eigen 5.0.0's line endings

非缺陷(构建配置)。修改eigen第三方库下载URL以解决换行符问题，不影响运行时行为。

---

### 89a58c3a 修复拦截问题
- 根因类别：参数校验(拦截)逻辑不完整，遗漏多种layout组合和功能互斥场景
- 涉及文件：attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.h
- 缺陷描述：三类bug：(1) `CheckNormalTensorList`中BSH检查遗漏`BSH_BNSD`和`BSH_NBSD`组合layout，BNSD检查遗漏`BNSD_NBSD`，导致这些layout下tensor list shape一致性校验被跳过。(2) 缺少`CheckRopeDataType`函数，rope tensor数据类型与query/key不一致时未拦截。(3) `CheckQuant`遗漏MLA不支持antiquant的拦截，`CheckPseShiftTypeAndShape`遗漏PFARope下MLA不支持pseShift的拦截。
- 修复模式：扩展layout字符串匹配范围，新增`CheckRopeDataType`校验函数，补充遗漏的互斥条件。
- 可审查性：中
- 审查规则建议：新增layout类型或功能特性时，搜索所有引用相关layout/flag的条件分支确保覆盖。建立layout与feature兼容性矩阵自动校验。

---

### 38c7162c 【PFA】删除keydim1 < blockNumValid的校验
- 根因类别：错误校验导致合法输入被拒绝(over-validation)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling.cpp
- 缺陷描述：PagedAttention模式下，`CheckPAWhenBaseApi`和`CheckPATypeAndShape`校验了`keyDim1 < blockNumValid`，但key的block pool可以共享/复用，keyDim1完全可以小于blockNumValid。多余校验导致合法输入被拒绝。
- 修复模式：直接删除两处`OP_CHECK_IF(keyDim1 < blockNumValid)`校验。
- 可审查性：中
- 审查规则建议：删除校验逻辑时检查是否存在对称位置的相同错误校验(copy-paste pattern)。新增校验时需附带该约束的技术依据。

---

### 68af3c15 aclnnGroupedMatmulV5接口支持量化场景

非缺陷(文档+测试新增)。文档补充量化场景参数描述，测试文件补充量化测试用例。

---

### 1ffefbce add inferGroupSize for mmar mmrs & agmm
- 根因类别：缺失参数自动推断逻辑 + 模板类型不匹配
- 涉及文件：mc2/all_gather_matmul_v2/op_host/op_tiling/arch35/all_gather_quant_bmm_tiling.cpp, mc2/common/inc/tiling/mc2_tiling_utils.h, mc2/common/src/mc2_tiling_utils.cpp, mc2/matmul_all_reduce/op_host/op_tiling/arch35/quant_matmul_all_reduce_tiling_950.cpp, mc2/matmul_all_reduce/op_host/op_tiling/matmul_all_reduce_tiling_base.cpp, mc2/matmul_all_reduce/op_host/op_tiling/matmul_all_reduce_tiling_base.h, mc2/matmul_reduce_scatter_v2/op_host/op_tiling/arch35/quant_bmm_reduce_scatter_tiling.cpp
- 缺陷描述：三个MC2算子处理量化groupSize参数(打包的uint64，M/N/K各占16bit)时，部分维度为0表示"由算子自动推断"，但原代码无推断逻辑，直接用0值做校验/计算导致合法输入被拒或tiling计算错误。另有`GetAttrPointer<uint64_t>`应为`int64_t`的类型不匹配。
- 修复模式：新增`Mc2TilingUtils::InferGroupSize()`公共函数做自动推断；三算子统一调用后再校验；修复类型不匹配。
- 可审查性：低
- 审查规则建议：输入参数部分字段可能为0/空时，检查是否有默认值推断或fallback逻辑。`GetAttrPointer`模板参数类型与属性注册类型一致性校验。

---

### 57a30f3a 修复GroupedMatmulAdd算子参数不完全匹配问题

非缺陷(纯API文档修正)。修正文档表格中的参数名和方向描述。

---

### a0733b24 repair S1G InvalidRows boundary condition
- 根因类别：嵌套循环边界条件逻辑错误
- 涉及文件：attention/common/op_kernel/vector_common.h
- 缺陷描述：`DealInvalidRowsBelow`函数用嵌套while+for循环对无效行清零，存在多处逻辑缺陷：(1) 内层while用`s1Stride`做步进判断不精确，可能跳过需清零的行或多清零有效行。(2) 内层`if (s1 == s1BottomTok) break`在`s1 < s1BottomTok`条件下是死代码。(3) 迭代变量`i`在内外层循环中被共同修改，难以保证正确覆盖所有case。
- 修复模式：重写为先算偏移再单层while循环的线性扫描模式，消除嵌套循环的边界条件错误。
- 可审查性：中
- 审查规则建议：内外层循环共同修改同一迭代变量的模式极易出错。while循环条件含`&&`但循环体首行break同一条件的死代码检测。

---

### 1a62a495 [FIA] aclnnFusedInferAttentionScoreV4 (splitfusepa) 算子MHA场景功能泛化

非缺陷(功能泛化)。放宽MHA场景支持条件，扩大算子支持范围，非bug修复。

---

### cf2a33d1 修复GQA perblock 全量化qs=1下的拦截信息不匹配问题
- 根因类别：输入校验缺失 + 错误信息不准确
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：(1) `CheckPerTensorQuantParams`缺少对rope的拦截 — pertensor量化场景下使用rope会导致功能异常，但未拦截，用户传入rope时静默产生错误结果。(2) perblock场景的错误消息写"PFARope"应为"Rope"，拦截信息与实际约束不匹配。
- 修复模式：新增`OP_CHECK_IF(enablePFARope, ...)`校验；修正错误消息文本。
- 可审查性：高
- 审查规则建议：新增输入约束时检查所有相关校验函数是否都覆盖；错误消息应与用户可见的参数名/概念一致。

---

### b0182bb3 [FIX]修复s1RealSize为0时没有特殊处理的问题
- 根因类别：除零错误(边界条件缺失)
- 涉及文件：attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h
- 缺陷描述：`ComputeParamS1`函数中，`s1RealSize`为0时`halfS1RealSize`也为0，导致`(runParam.nextTokensPerBatch * (-1)) / runParam.halfS1RealSize`除以零，device kernel中可能崩溃。
- 修复模式：条件判断最前面加入`runParam.s1RealSize == 0`短路判断(用`||`)，为0时直接返回true跳过后续计算。
- 可审查性：高
- 审查规则建议：作为除数的变量必须检查零值边界；静态分析时追踪分母变量取值范围确保不存在零值路径。

---

### 9081122a fix invalid link

非缺陷(纯文档链接修复)。修复markdown链接路径，不涉及运行时代码。

---

### 0162cad4 [mc2] fix wqmmar supported data type in pertensor scenario

非缺陷(纯文档描述修正)。修正数据类型支持场景的文档描述，不修改代码。

---

### 7af7c963 kv fix check shape
- 根因类别：数据类型枚举遗漏 + 空tensor未校验
- 涉及文件：posembedding/kv_rms_norm_rope_cache/op_host/kv_rms_norm_rope_cache_base_tiling.cpp, posembedding/kv_rms_norm_rope_cache/op_host/kv_rms_norm_rope_cache_regbase_full_load_tiling.cpp, posembedding/kv_rms_norm_rope_cache/op_host/kv_rms_norm_rope_cache_regbase_recompute_tiling.cpp, posembedding/kv_rms_norm_rope_cache/op_host/kv_rms_norm_rope_cache_tiling.h
- 缺陷描述：(1) 对齐校验中`kcacheDtype`/`vcacheDtype`判断仅覆盖`DT_INT8`，遗漏`DT_HIFLOAT8`/`DT_FLOAT8_E5M2`/`DT_FLOAT8_E4M3FN`三种量化类型，走错误的FP16对齐分支(16B vs 32B)导致数据对齐错误。(2) `DoOpTiling`入口缺少空tensor校验，传入空tensor会导致tiling访问无效shape。
- 修复模式：扩展dtype判断条件覆盖所有量化类型(四处同步修改)；新增`CheckInputShapeIsEmpty()`前置校验。
- 可审查性：高
- 审查规则建议：基于枚举值的switch/if分支检查是否覆盖所有有效值，特别关注后续新增的枚举成员。tensor输入算子检查是否有空tensor前置校验。

---

### b5d7c0fe [FAG] fix rope empty tensor bug
- 根因类别：空tensor场景多层联动处理缺失
- 涉及文件：attention/flash_attention_score_grad/op_api/aclnn_flash_attention_score_grad.cpp, attention/flash_attention_score_grad/op_host/flash_attention_score_grad_infershape.cpp, attention/flash_attention_score_grad/op_host/flash_attention_score_grad_tiling.cpp, attention/flash_attention_score_grad/op_kernel/arch35/flash_attention_score_grad_empty_tensor_regbase.h, attention/flash_attention_score_grad/op_kernel/arch35/flash_attention_score_grad_tiling_data_regbase.h, attention/flash_attention_score_grad/op_kernel/flash_attention_score_grad_apt.cpp
- 缺陷描述：三个关联bug：(1) BSH/SBH layout下`dDim`为0时直接报错，但rope空tensor场景下这是合法输入。(2) InferShape对rope shape的判断条件`!= nullptr && GetShapeSize() != 0`导致空tensor时输出shape未赋值。(3) empty tensor kernel缺少dqRope/dkRope的清零处理，输出含脏数据。
- 修复模式：op_api层增加D=0分支；infershape层放宽shape赋值条件；tiling/kernel层增加rope输出清零逻辑。跨4层联动修复。
- 可审查性：中
- 审查规则建议：empty tensor/边界case下检查所有输出tensor是否被正确初始化(清零)；可选输入的对应可选输出在各组合场景下的处理完备性。

---

### 3e056c8d 修复GroupedMatmulSwigluQuantV2、GroupedMatmulV4、GroupedMatmulV3算子问题
- 根因类别：示例代码中的VLA使用不当 + 空数组越界访问 + 非法内存释放
- 涉及文件：gmm/grouped_matmul/docs/aclnnGroupedMatmulV3.md, gmm/grouped_matmul/docs/aclnnGroupedMatmulV4.md, gmm/grouped_matmul_swiglu_quant_v2/docs/aclnnGroupedMatmulSwigluQuantV2.md
- 缺陷描述：文档示例代码中三处bug：(1) `aclTensor* tensors[size]`是C99 VLA，C++中不标准。(2) `aclCreateIntArray(data, 1)`但data为空vector，越界读取。(3) 对栈上数组指针调用`aclrtFree`属非法释放。
- 修复模式：VLA改`std::vector`；修正size参数；删除非法`aclrtFree`调用。
- 可审查性：高
- 审查规则建议：示例代码避免VLA；API调用size参数与实际数据长度匹配；`aclrtFree`参数确认为device侧分配的内存。
- 备注：此commit主体为文档示例代码修复，同时包含拼写修正等纯文档变更。

---

### 5c48d0be 补充groupListType描述

非缺陷(纯文档修改)。补充`aclnnGroupedMatmulWeightNz.md`中groupListType约束描述。

---

### 8f3a5747 修改整仓错误链接及错误产品名称

非缺陷(纯文档修改)。覆盖89个md文件修正链接路径、产品名称、术语统一。

---

## 批次3新增缺陷类别

| 类别 | 本批新增数 | hash |
|------|-----------|------|
| workspace/buffer大小计算错误(多layout) | 1 | fcff1be7 |
| 参数校验遗漏(layout组合/功能互斥) | 1 | 89a58c3a |
| 错误校验导致合法输入被拒(over-validation) | 1 | 38c7162c |
| 缺失参数自动推断逻辑 + 类型不匹配 | 1 | 1ffefbce |
| 嵌套循环边界条件逻辑错误 | 1 | a0733b24 |
| 输入校验缺失(量化+rope互斥) | 1 | cf2a33d1 |
| 除零错误(边界条件缺失) | 1 | b0182bb3 |
| 数据类型枚举遗漏 + 空tensor未校验 | 1 | 7af7c963 |
| 空tensor场景多层联动处理缺失 | 1 | b5d7c0fe |
| 示例代码缺陷(VLA/越界/非法释放) | 1 | 3e056c8d |

---

## 批次4: 提交 #61-#80 (2026-02-12 ~ 2026-02-14)

---

### 1f3291bf fix FA empty tensor n2=0 and d=0 div 0 problem
- 根因类别：除零错误(空tensor边界)
- 涉及文件：attention/flash_attention_score/op_api/aclnn_flash_attention_score.cpp
- 缺陷描述：AnalysisAxisForBsh和AnalysisAxisForSbh函数中，当n2=0时用n2做除数计算dk和dv会触发除零崩溃。原代码对dSize==0只做了提前return但没有保护n2==0的除法路径。
- 修复模式：去掉dSize==0的提前return，改为在每个除法前加三元运算符保护——当d==0时n2赋0，当n2==0时dk/dv回退为d值。
- 可审查性：高
- 审查规则建议：所有除法运算前检查除数是否可能为零；shape解析函数中空tensor(维度为0)的边界case是否被覆盖。

---

### 44e84d52 修复tensorlist场景B全为0被误拦截，添加非量化4维mask第2维异常校验
- 根因类别：校验逻辑顺序错误 + 校验条件缺失
- 涉及文件：attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp, attention/incre_flash_attention/op_host/incre_flash_attention_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：(1) CheckTensorList中遍历tensorlist读取shape的while循环内就检查Batch维是否为1，但B全为0是合法空tensor场景，会被误拦截(CheckEmptyTensorList在后面才调用)。(2) IFA和PFA的CheckMaskShape中，对非量化4维mask的第2维(N维)没有做!=1的校验。
- 修复模式：(1) 将Batch==1校验移到CheckEmptyTensorList之后。(2) 增加attenMaskN==1的校验条件。
- 可审查性：中
- 审查规则建议：输入校验逻辑是否在合法提前退出(如空tensor判断)之后才执行；mask shape每个维度是否都有对应校验。

---

### 0c90fb3c 修复CMake构建脚本中JSON库头文件包含路径设置错误的问题
- 根因类别：构建配置错误(include路径指向错误)
- 涉及文件：cmake/third_party/json.cmake
- 缺陷描述：原代码通过set_target_properties设置INTERFACE_INCLUDE_DIRECTORIES指向了安装路径，但该路径在ExternalProject构建完成前可能不存在或内容不正确。
- 修复模式：删除错误的set_target_properties调用。
- 可审查性：中
- 审查规则建议：CMake ExternalProject的头文件路径应指向实际源码/构建产物路径，而非尚未生成的安装路径。

---

### 8ceb72b6 grouped_matmul_swiglu_quant_v2 修复日志提示不合理的问题
- 根因类别：输入校验缺失(dtype白名单)
- 涉及文件：gmm/grouped_matmul_swiglu_quant_v2/op_host/op_api/aclnn_grouped_matmul_swiglu_quant_v2_utils.h
- 缺陷描述：量化场景下，当x或weight的dtype不在任何支持列表(FP8/MXFP4/Pertoken)中时，原代码没有提前拦截，会穿透到后续组合dtype匹配逻辑，产生未定义行为。
- 修复模式：在进入具体分支之前增加对x dtype和weight dtype的全量支持列表检查。
- 可审查性：高
- 审查规则建议：校验函数应在入口处对所有输入参数做白名单检查，而不仅在分支内部做局部检查。

---

### 368c7325 fix ScatterPaKvCache 同步问题
- 根因类别：硬件流水线同步缺失
- 涉及文件：attention/scatter_pa_kv_cache/op_kernel/arch35/scatter_pa_kv_cache_rope_fully_load.h, attention/scatter_pa_kv_cache/op_kernel/arch35/scatter_pa_kv_cache_rope_not_fully_load.h
- 缺陷描述：(1) fully_load中Cast运算后紧接DataCopyPad写出GM，缺少V_MTE3同步屏障。(2) not_fully_load中V_MTE3同步位置错误，放在Div之后而非Cast之后。(3) 循环内重复调用slotMappingLocal.GetValue(iter)。
- 修复模式：在Cast后DataCopyPad前插入V_MTE3同步；将同步位置修正到正确依赖点；提取循环不变量。
- 可审查性：低
- 审查规则建议：Vector计算结果写出GM(DataCopyPad)前必须有V_MTE3同步屏障；审查同步原语位置是否紧邻被保护的操作对。

---

### 349083a7 Revert "修改了all_gather_matmul算子ut的op_host组件的用例输入方式，改成csv表格"

非缺陷提交。被revert的内容是UT测试代码的输入方式重构(csv改回硬编码)，不涉及产品代码逻辑。

---

### 44338c14 修复alltoallmatmul的api校验
- 根因类别：校验逻辑条件分支错误(量化模式互斥)
- 涉及文件：mc2/allto_all_matmul/op_api/aclnn_allto_all_quant_matmul.cpp
- 缺陷描述：CheckAllDtypesValid中，DYN_PERTOKEN_QUANT场景的if块结束后，紧接着对所有x1ScaleOptional!=nullptr的情况都用SCALE_DTYPE_SUPPORT_LIST做校验，覆盖了DYN_PERTOKEN_QUANT的特殊处理。同时缺少PERTOKEN_QUANT场景的校验。
- 修复模式：将独立的if改为else if，使DYN_PERTOKEN_QUANT和PERTOKEN_QUANT各走独立校验路径。
- 可审查性：高
- 审查规则建议：多个量化模式的校验分支应用if-else if确保互斥，避免后续校验覆盖前面的特殊处理。

---

### bd65dfcc 修改GmmSwigluQuantV2算子aclnn通路调用宏函数输出报错信息不明确的异常

非缺陷提交。纯代码重构，将成员变量访问改为参数传递以改善宏展开时的报错信息可读性，校验逻辑行为完全不变。

---

### 4efbd43d 空tensor初始化添加同步
- 根因类别：硬件流水线同步缺失
- 涉及文件：attention/prompt_flash_attention/op_kernel/arch35/prompt_flash_attention_zero_output.h
- 缺陷描述：PromptFlashAttentionZeroOutPut::Process中，先对output做DuplicateZero/DuplicateInf初始化写出GM，再对softMaxLse做初始化写出GM。两段写出之间及之后都缺少MTE3_V同步屏障，可能导致数据竞争。
- 修复模式：在两段GM写出操作之间和最后一段之后各插入MTE3_V的SetFlag/WaitFlag同步。
- 可审查性：低
- 审查规则建议：所有对GM的写出操作之后，如果后续还有对GM的操作，必须插入对应的同步屏障。

---

### eb8f0ac7 MLA非量化mask bugfix
- 根因类别：赋值遗漏 + 错误的提前返回条件
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_kernel_noquant_mla.h, attention/common/op_kernel/arch35/flash_attention_score_block_vec_base.h
- 缺陷描述：(1) ComputeC函数初始化attenMaskInfo时遗漏attenMaskS1Size赋值，MLA非量化路径中mask的S1维度信息丢失。(2) MlaBoolCopyInRegbase中totalS2Size%blockBytes!=0时直接return，非对齐情况也应正常处理。
- 修复模式：补充attenMaskS1Size赋值；删除错误的提前返回判断。
- 可审查性：中
- 审查规则建议：初始化结构体时应逐字段检查是否有遗漏；提前返回条件是否真代表不可处理的情况。

---

### 9bb73d4c [FAG] error info modify

非缺陷提交。只修改了错误信息字符串中的数字(ceil(S2/128)->ceil(S2/256))，校验逻辑条件表达式完全没变，纯日志文本修正。

---

### 4573c2e ActSeqLen拦截信息修复
- 根因类别：校验逻辑条件错误(多余条件放行非法配置)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：MLA场景下ActSeqLen的layout校验条件 `inputLayout != InputLayout::TND && inputLayout != InputLayout::NTD` 等价于放行了TND和NTD，但NTD不应被放行——正确行为是只允许TND(以及TND_NTD变体)。
- 修复模式：将条件改为 `inputLayout != InputLayout::TND`，只允许TND通过校验。
- 可审查性：中
- 审查规则建议：当校验逻辑使用多个"不等于"条件组合时，逐一确认每个被排除的值是否确实应该被排除；检查错误提示文本与实际校验逻辑是否一致。

---

### 46bdb193 补充grouplistType描述

非缺陷提交。只修改了.md文档文件，补充groupListType说明，纯文档更新。

---

### 3881d5c gmm算子aclnn异常场景拦截打印日志修复
- 根因类别：校验逻辑不完整(缺少上界检查) + printf格式符类型不匹配
- 涉及文件：gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul.cpp
- 缺陷描述：(1) 对x tensor的维度数只检查了下界 `xDimNum >= MIN_FM_DIM`，缺少上界检查，超过MAX_FM_DIM(6)时不会被拦截。(2) weight维度日志中使用%d打印size_t类型的GetDimNum()返回值，64位平台可能打印错误值。
- 修复模式：增加上界校验 `xDimNum <= MAX_FM_DIM`；修正%d为%zu。
- 可审查性：高
- 审查规则建议：数值范围校验必须同时检查上界和下界；printf格式符必须与实参类型严格匹配。

---

### a6aa6f17 revert//优化头文件

非缺陷提交。被revert的原始PR内容是将细粒度头文件include替换为聚合头文件及尾部空格清理，纯重构/清理性质。

---

### 0c3b9b38 Fix Debug: Min HCCL_BUFFSIZE may result in cleaning bufferId
- 根因类别：硬编码常量导致缓冲区越界清理
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_a2_layered.h
- 缺陷描述：清理token end flag时使用硬编码常量MAX_BS_NUM=256，但实际最大batch size是运行时根据globalBs_/worldSize_动态计算的。当HCCL_BUFFSIZE为极限最小值时，buffer按实际maxBs分配，但cleanup固定按256清理，越界覆盖相邻区域的bufferId等元数据。
- 修复模式：删除硬编码MAX_BS_NUM=256，用运行时计算的maxBs_替换，确保清理范围与分配大小一致。
- 可审查性：中
- 审查规则建议：buffer分配大小和使用/清理大小必须来自同一数据源，禁止分配用动态值、清理用硬编码常量的模式。

---

### 1734ac98 修正pr_1711对moe算子的修改错误
- 根因类别：前序PR引入多处逻辑错误(CMake条件取反、芯片型号硬编码、batch size上限不区分模式、缺少调度模式设置)
- 涉及文件：mc2/moe_distribute_combine_v2/op_host/CMakeLists.txt, mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp, mc2/moe_distribute_dispatch_v2/op_host/CMakeLists.txt, mc2/moe_distribute_dispatch_v2/op_host/op_tiling/moe_distribute_dispatch_v2_tiling.cpp
- 缺陷描述：PR !1711引入多处错误：(1) CMake条件NOT取反写反；(2) 使用socVersion字符串比较判断芯片架构，应用GetNpuArch()枚举比较；(3) layered模式batch size上限应为512而非256；(4) dispatch/combine的A2 tiling缺少SetScheduleMode(batch_mode=1)，涉及SyncAll操作可能死锁。
- 修复模式：修正CMake条件方向；改用arch枚举比较；增加layered模式max batch size常量；添加batch mode调度设置。
- 可审查性：高
- 审查规则建议：CMake条件使用NOT时特别审查方向；芯片平台判断用架构枚举而非版本字符串；涉及SyncAll的算子必须检查batch mode设置。

---

### 974e5560 修复确定性场景FAG算子在SparseMode=1情况下的越界访问
- 根因类别：掩码尺寸硬编码导致缓冲区越界访问
- 涉及文件：attention/flash_attention_score_grad/op_kernel/arch32/basic_modules/vec_op_det.h
- 缺陷描述：SubGrapA函数中处理attention mask时，掩码张量尺寸硬编码为64x128，但sparseMode==1下实际尺寸由s1Extend和s2Extend决定，可能不同，导致CopyInAttenMaskBool和CalcAttenMaskBool越界读取。
- 修复模式：根据sparseMode动态计算掩码尺寸，sparseMode==1时用s1Extend和s2Extend，否则回退固定值；最后一轴做32字节对齐。
- 可审查性：中
- 审查规则建议：kernel中访问tensor时尺寸参数禁止与运行时配置无关的硬编码值；新增分支路径时须检查所有依赖参数是否在该路径下仍正确。

---

### d650ef8f 修复GMM V4问题

非缺陷提交。只修改了markdown文档文件中API签名的格式(两参数拆为两行显示)，纯文档格式调整。

---

### 5023d7a8 moe_finalize_routing_v2 md fix

非缺陷提交。修改.md文档中冗余分号和示例.cpp中的注释文本修正，不影响实际算子逻辑。

---

## 批次4新增缺陷类别

| 类别 | 本批新增数 | hash |
|------|-----------|------|
| 除零错误(空tensor边界) | 1 | 1f3291bf |
| 校验逻辑顺序错误(空tensor误拦截) | 1 | 44e84d52 |
| 构建配置错误(CMake include路径) | 1 | 0c90fb3c |
| 输入校验缺失(dtype白名单) | 1 | 8ceb72b6 |
| 硬件流水线同步缺失 | 2 | 368c7325, 4efbd43d |
| 校验逻辑条件分支错误(量化模式互斥) | 1 | 44338c14 |
| 赋值遗漏+错误的提前返回 | 1 | eb8f0ac7 |
| 校验逻辑条件错误(多余条件放行非法配置) | 1 | 4573c2e |
| 校验逻辑不完整(缺少上界)+格式符类型不匹配 | 1 | 3881d5c |
| 硬编码常量导致缓冲区越界清理 | 1 | 0c3b9b38 |
| 前序PR引入多处逻辑错误(回归缺陷) | 1 | 1734ac98 |
| 掩码尺寸硬编码导致越界访问 | 1 | 974e5560 |

---

## 批次5: 提交 #81-#99 (2026-02-10 ~ 2026-02-12)

---

### 853ce34a 【FAG】empty tensor bug
- 根因类别：整数类型错误(int64_t vs uint64_t) + 宏参数数量错误
- 涉及文件：attention/flash_attention_score_grad/op_host/flash_attention_score_grad_tiling.cpp, attention/flash_attention_score_grad/op_kernel/arch35/flash_attention_score_grad_kernel_base.h
- 缺陷描述：两处修复: (1) SetTilingKey调用的GET_TPL_TILING_KEY宏参数个数不对(18个→19个)，empty tensor场景tiling key生成错误导致走到错误kernel分支; (2) s2SparseLeft/s2SparseRight类型从int64_t改为uint64_t，这些变量参与位移运算(>>6 <<6)和Max(...,0)比较，有符号负值右移是算术右移保留符号，修复后用无符号类型配合Max确保稀疏窗口左边界为负时clip到0。
- 修复模式：数据类型修正(int64_t→uint64_t) + 宏参数数量修正
- 可审查性：中
- 审查规则建议：检测宏调用参数数量是否与宏定义一致；涉及位运算的变量检查有符号/无符号类型是否匹配语义预期。

---

### d3aa4960 修复matmulReduceScatter/matmulReduceScatterV2文档和参数校验更正
- 根因类别：空指针解引用(null pointer dereference)
- 涉及文件：mc2/matmul_reduce_scatter_v2/op_host/op_tiling/arch32/matmul_reduce_scatter_v2.cpp
- 缺陷描述：原校验逻辑将空指针检查和值检查合并为一个条件`commModePtr == nullptr || !(strcmp(commModePtr, "aiv") == 0)`，当commModePtr为nullptr时，OP_LOGE的%s格式化可能对null指针产生未定义行为(某些平台崩溃)。修复后拆分为两个独立的OP_TILING_CHECK: 先检查nullptr再检查值。
- 修复模式：空指针检查前置，避免对null pointer调用strcmp和printf %s
- 可审查性：高
- 审查规则建议：检测对可能为null的指针在同一条件中既检查null又调用strcmp/printf %s的模式——应将null检查拆分为独立前置校验。

---

### 54692072 gmm伪量化放开n=0场景拦截
- 根因类别：空tensor场景过度拦截(over-validation)
- 涉及文件：gmm/grouped_matmul/ 下9个文件(infershape/aclnn/tiling/kernel四层)
- 缺陷描述：GMM算子伪量化场景对N=0空tensor输入过度拦截(<=0改为<0)。多层面问题: (1) infershape/aclnn/tiling层用<=0拦截N和K，阻止了合法N=0空tensor; (2) 含batch轴时空tensor判断只看最后第二维未累乘batch轴; (3) bias等tensorlist为[(0)]时IsNonEmpty判断不正确导致nullptr解引用; (4) kernel层N=0未skip计算导致非法内存访问。
- 修复模式：放宽校验条件 + 修正空tensor判断逻辑 + 增加kernel层N=0跳过逻辑
- 可审查性：中
- 审查规则建议：维度校验中<=0拦截时检查是否存在合法的维度为0空tensor；含batch轴的tensor判空应累乘所有非K维度。

---

### 046aef30 FusedFloydAttention算子workspace大小修正
- 根因类别：workspace大小计算遗漏因子
- 涉及文件：attention/fused_floyd_attention/op_host/fused_floyd_attention_tiling_general.cpp
- 缺陷描述：GetWorkspaceSize计算stage2所需空间时，公式遗漏了n2BaseSize乘法因子。原公式`s1BaseSize * alignedD * calcTypeSize`应为`n2BaseSize * s1BaseSize * alignedD * calcTypeSize`。workspace估算偏小导致NZND/FP32场景内存越界。此外还删除了调试代码`workspaces[0] += 300*1024*1024;`(硬编码300MB)，说明之前用此临时掩盖了workspace不足。
- 修复模式：在workspace公式中补充遗漏的维度因子
- 可审查性：高
- 审查规则建议：workspace计算应覆盖所有参与维度；硬编码大数值内存偏移(如300MB)应作为code smell标记。

---

### 64dedf37 修复scatterPaKvCache算子aicore问题
- 根因类别：参数传递错误(分块处理量 vs 总大小)
- 涉及文件：attention/scatter_pa_kv_cache/op_kernel/arch35/scatter_pa_kv_cache_rope_not_fully_load.h
- 缺陷描述：CastToOrigin函数第三个参数(处理数据量)原传kHeadSize/vHeadSize，应传handleNum。在not fully load场景下每次处理量由handleNum决定(分块后实际处理量)，当kHeadSize较大时handleNum < kHeadSize，传错参数导致处理超出buffer范围触发aicore错误。
- 修复模式：函数参数从固定总大小改为实际分块处理量
- 可审查性：高
- 审查规则建议：分块处理模式下数据操作函数的长度参数应使用当前分块大小而非总大小。

---

### aa9faa8c 修复mla全量化场景qs>1输入sparsemode非3的报错提示
- 根因类别：校验条件范围过大(over-validation)
- 涉及文件：attention/incre_flash_attention/op_host/incre_flash_attention_tiling_check.cpp
- 缺陷描述：CheckMaskShapeWithQSeq中条件`if (antiQuantFlag_ || quantFlag_)`过于宽泛。quantFlag_不只MLA全量化还包含其他量化场景，而"qs>1时sparseMode必须为3"约束只适用于MLA场景。使用quantFlag_导致非MLA量化场景在qs>1且sparseMode!=3时被错误拦截。修复后条件改为精确匹配MLA场景(dequantScaleQuery != nullptr && ropeFlag_)。
- 修复模式：缩小条件范围，使校验仅作用于目标场景
- 可审查性：高
- 审查规则建议：输入校验guard condition应精确匹配适用场景，避免宽泛标志位；新增校验应验证所有已有场景兼容性。

---

### 021f8352 add error message for invalid quantmode parameters
- 根因类别：缺失错误处理/静默失败
- 涉及文件：mc2/allto_all_matmul/op_api/aclnn_allto_all_quant_matmul.cpp, mc2/matmul_allto_all/op_api/aclnn_quant_matmul_allto_all.cpp
- 缺陷描述：CheckDtypesValid函数中quantMode不匹配时存在两个问题: (1) allto_all_quant_matmul: quantMode不匹配时isAllDtypesValid保持false直接返回但无任何错误日志，用户无法得知失败原因; (2) quant_matmul_allto_all: 错误日志打印所有tensor数据类型，对quantMode不匹配场景产生误导。修复后在else分支添加OP_LOGE输出具体quantMode值。
- 修复模式：添加缺失错误日志 + 修正误导性错误信息
- 可审查性：高
- 审查规则建议：参数校验函数返回false的路径必须伴随OP_LOGE调用，禁止静默失败。

---

### 14824774 BugFix: KvRmsNormRopeCache算子大shape AicError问题修改
- 根因类别：整数溢出(循环索引类型不足)
- 涉及文件：posembedding/kv_rms_norm_rope_cache/op_kernel/arch35/kv_rms_norm_rope_cache_regbase_recompute.h
- 缺陷描述：5个Rope处理函数中for循环索引ubIdx声明为uint16_t(最大65535)，而循环上界ubFactorDkLoopCountCeil在D维度极大时超过uint16_t范围，导致循环索引溢出回绕产生AIC硬件错误。
- 修复模式：循环索引类型从uint16_t扩展为int64_t
- 可审查性：高
- 审查规则建议：循环索引类型应覆盖循环上界最大可能取值；当上界来源于外部shape计算时索引应用int64_t而非窄类型。

---

### b59cf016 [Fix] fix small s2 tail basic block
- 根因类别：尾块边界条件处理错误
- 涉及文件：attention/sparse_flash_attention_grad/basic_modules/cube_modules/cube1-5.h, cube_op.h (6个文件)
- 缺陷描述：sparse_flash_attention_grad处理s2维度尾部basic block时MM参数计算有误: (1) cube1/cube2的Sparse路径singleN固定为selectedBlockSize*blockOffset，未考虑最后block的lastBlockSize可能更小; (2) cube3中totalSel计算和isLastBasicBlock判断只在isLastLoop时生效导致非末次迭代singleK不正确; (3) cube4/5的singleM和GM拷贝长度未考虑lastBlockSize。s2不能被block大小整除时MM维度参数超出有效数据范围，偶现MTE1硬件报错。
- 修复模式：将lastBlockSize/isLastBasicBlock传入各cube函数，循环内用min()动态约束MM参数和数据搬运长度
- 可审查性：中
- 审查规则建议：分块处理时必须检查尾块实际大小是否被正确传递和使用；MM参数应取min(块大小, 剩余有效长度)。

---

### 7cab8a6a 修复MoeGatingTopKSoftmaxV2算子finished参数为true时expertIdxOut取值不对问题
- 根因类别：操作时序错误(EnQue位置不当导致break跳过入队)
- 涉及文件：moe/moe_gating_top_k_softmax_v2/op_kernel/arch35/moe_gating_top_k_softmax_v2_k_renorm.h
- 缺陷描述：indicesOutLocal的EnQue原放在ComputeSoftmax内部。当finished=true时代码在ComputeTopK之后ComputeSoftmax之前break跳出循环，直接进入CopyOut。break跳过了EnQue调用，CopyOut读到未入队的脏/旧数据，expertIdxOut输出不正确。
- 修复模式：将EnQue从ComputeSoftmax内部移到主循环CopyOut前，确保所有控制流路径都能正确入队
- 可审查性：高
- 审查规则建议：存在提前退出(break/continue/return)时，检查队列EnQue/DeQue是否在所有路径上正确配对；资源管理操作放在不会被跳过的位置。

---

### 非缺陷提交(本批次9条)

| hash | message | 排除原因 |
|------|---------|---------|
| d3460fbf | 回退 ROPEV2 文档 | 纯文档删除 |
| 7084b812 | 回退 ROPE V2接口 | 功能回退(revert)，非缺陷修复 |
| 2d497549 | 修复GMM问题，提升易用性 | 纯文档修改(docs/目录) |
| a9a25159 | fix alltoallquantmatmul A5 example | 示例代码修复(examples/目录) |
| af854dcd | fix GMMSQ && GMMweightNZ && GMMV5算子问题 | 纯文档修改(.md文件) |
| 6b173d1a | 将blockDim修改为numBlocks | 纯变量重命名重构 |
| 4143b07a | 修复 GMM && MOEInitRouting算子问题 | 纯文档修改(图片+链接) |
| 5980e181 | [FIA IFA PFA]将blockDim修改为numBlocks | 纯变量重命名重构 |
| 11e82892 | 修复GMMV4报错信息未打印数据类型 | DFX改进(错误信息增强) |

---

## 批次5缺陷类别统计

| 缺陷类别 | 数量 | 涉及hash |
|----------|------|---------|
| 整数类型错误/宏参数错误 | 1 | 853ce34a |
| 空指针解引用 | 1 | d3aa4960 |
| 空tensor过度拦截(over-validation) | 1 | 54692072 |
| workspace大小计算遗漏因子 | 1 | 046aef30 |
| 分块处理参数传递错误 | 1 | 64dedf37 |
| 校验条件范围过大(over-validation) | 1 | aa9faa8c |
| 缺失错误处理/静默失败 | 1 | 021f8352 |
| 整数溢出(循环索引类型不足) | 1 | 14824774 |
| 尾块边界条件处理错误 | 1 | b59cf016 |
| 操作时序错误(EnQue位置不当) | 1 | 7cab8a6a |

---

## 批次1+2+3+4+5累计统计

总计99条提交:
- 实际缺陷修复: 52条 (52.5%)
- 非缺陷(排除): 47条 (47.5%)

批次5: 10条缺陷 + 9条非缺陷 (52.6%)

累计非缺陷hash: fe0bee0d, e8ffb3b9, 53c21a1d, ae4c91e3, 91ef89b4, 7fbc7bc3, 108b4dd3, af418dfc, 28671df1, 8a09dcd0, 14a289de, f73c0505, 967fed57, f38d4a49, 613ae40b, 1065427e, 79a2aced, 988a83a9, 28daf0ab, be6bcad7, 4d3bbf03, e46032f7, 05e3ba28, e48d4172, 68af3c15, 57a30f3a, 1a62a495, 9081122a, 0162cad4, 5c48d0be, 8f3a5747, 349083a7, bd65dfcc, 9bb73d4c, 46bdb193, a6aa6f17, d650ef8f, 5023d7a8, d3460fbf, 7084b812, 2d497549, a9a25159, af854dcd, 6b173d1a, 4143b07a, 5980e181, 11e82892

---

## 批次6: 提交 #100-#118 (2026-02-07 ~ 2026-02-10)

---

### 80329f3a fix: scatter_pa_cache not compiling
- 根因类别：构建配置缺陷(新算子集成遗漏)
- 涉及文件：attention/scatter_pa_cache/op_host/CMakeLists.txt, attention/scatter_pa_cache/op_kernel/scatter_pa_cache.cpp(原名scatter_pa_cache_apt.cpp), scripts/ci/ascend950/ops_transformer_operator_list.yaml
- 缺陷描述：scatter_pa_cache算子存在三个问题导致无法编译：(1) CMakeLists.txt缺少`set(scatter_pa_cache_depends ...)`声明依赖关系；(2) kernel源文件命名为scatter_pa_cache_apt.cpp不符合约定(应为scatter_pa_cache.cpp)；(3) include路径仅写了一种相对路径未兼容另一种构建场景；(4) CI的operator_list.yaml未列入该算子。代码写好了但构建配置不完整，算子根本没有被编译进产物。
- 修复模式：重命名文件为标准名称；添加CMake依赖声明；用`#if __has_include()`做条件编译兼容两种路径；加入CI编译列表。
- 可审查性：中
- 审查规则建议：新增算子时应检查checklist——源文件命名是否符合约定、CMakeLists是否声明跨算子依赖、CI的operator_list是否包含新算子。

---

### 3c508d74 删除冗余函数，修复在非910b的CANN环境中ut用例执行失败的问题

非缺陷提交(UT测试修复)。修改tests/ut/目录下测试代码，将断言从`EXPECT_EQ(ACLNN_SUCCESS, ...)`改为`EXPECT_NE(ACLNN_ERR_PARAM_NULLPTR, ...)`，是对测试断言逻辑的修正。

---

### fd0ed91f Change Doc, single machine dc

非缺陷提交(纯文档更新)。所有8个文件均为README.md和API文档，扩展epWorldSize支持范围、删除约束说明。

---

### 3ea26544 fix(kv_rms_norm_rope_cache)：修改910B平台上，tilingkey判断分支

非缺陷提交(性能优化)。收紧DeepSeek优化分支的准入条件，增加batchSize > coreNum * BATCHES_FOR_EACH_CORE条件，避免非目标场景被错误分发到该分支导致性能损耗。

---

### 15ff1476 gmm_add aclnn description fix

非缺陷提交(纯文档注释修正)。修改.h文件中Doxygen注释的函数名引用笔误，不影响编译和运行时行为。

---

### 4e4f9168 【fix】moe_token_unpermute_with_routing_map_grad约束公式添加括号
- 根因类别：文档约束公式运算优先级错误(括号遗漏)
- 涉及文件：moe/moe_token_unpermute_with_routing_map_grad/docs/aclnnMoeTokenUnpermuteWithRoutingMapGrad.md
- 缺陷描述：约束公式缺少关键括号，运算符优先级导致语义错误。原公式`196608 - (probTypeLen + 1) * numExpertAlign-(tokenTypeLen + 8) * 256 / (6 * tokenTypeLen + 12) >= 1`按优先级`256/(6*tokenTypeLen+12)`会先计算。正确意图是分子为`196608 - (probTypeLen+1)*numExpertAlign - (tokenTypeLen+8)*256`，整体除以`6*tokenTypeLen+12`后`>=1`。用户按错误公式校验参数会得到错误结果。
- 修复模式：添加外层括号修正运算优先级。
- 可审查性：低
- 审查规则建议：API文档中约束公式应与代码实现中的校验逻辑交叉验证；review checklist中加入"验证公式括号是否正确反映运算优先级"。

---

### 613510ad 修复MoeInitRoutingV2 geir问题
- 根因类别：算子注册基础设施缺失(geir通路不可用)
- 涉及文件：moe/moe_init_routing_v2/op_graph/CMakeLists.txt(新增), moe/moe_init_routing_v2/op_graph/moe_init_routing_v2_proto.h(新增), moe/moe_init_routing_v2/op_host/moe_init_routing_v2_def.cpp(修改)
- 缺陷描述：MoeInitRoutingV2算子缺少op_graph目录下的proto头文件和CMakeLists.txt，导致geir通路(图引擎IR离线推理路径)无法运行。同时def.cpp缺少DynamicCompileStaticFlag、DynamicRankSupportFlag等配置，导致ascend950平台动态shape编译失败。
- 修复模式：补充缺失的proto.h注册文件和CMake配置；在def.cpp中添加动态编译配置项。
- 可审查性：低
- 审查规则建议：新增算子时检查op_graph目录是否包含proto.h和CMakeLists.txt；检查regbaseConfig是否配置动态shape相关flag。

---

### 08fbc029 [fix]自适应install_deps.sh的sudo, 避免找不到sudo报错
- 根因类别：环境兼容性缺陷(脚本硬编码sudo)
- 涉及文件：install_deps.sh
- 缺陷描述：install_deps.sh中所有包管理命令前硬编码了`sudo`，在没有安装sudo的环境(如精简Docker镜像)或已是root用户的环境下，脚本因"sudo: command not found"中断执行，依赖安装全部失败。
- 修复模式：脚本开头通过`command -v sudo`检测sudo是否存在，结合`$EUID`判断是否root，设置`try_sudo`变量替换硬编码的sudo。注意：修复不完整，有一处`sudo tee`未替换。
- 可审查性：高
- 审查规则建议：Shell脚本中不应硬编码sudo；应检测是否需要提权。

---

### 03f73405 sfag/slig infershape及校验修改
- 根因类别：输入校验缺失 + infershape输出dtype设置错误(多重缺陷)
- 涉及文件：attention/sparse_flash_attention_grad/op_host/arch35/sparse_flash_attention_grad_tiling_bs1_regbase.cpp, attention/sparse_flash_attention_grad/op_host/sparse_flash_attention_grad_infershape.cpp, attention/sparse_lightning_indexer_grad_kl_loss/op_host/arch35/sparse_lightning_indexer_grad_kl_loss_tiling_general_regbase.cpp, attention/sparse_lightning_indexer_grad_kl_loss/op_host/sparse_lightning_indexer_grad_kl_loss_infershape.cpp
- 缺陷描述：多重缺陷：(1) SFAG tiling中sparse_mode原本允许0和3但实际只支持3；head_dim校验条件不正确；缺少inputLayout校验、qRope/kRope维度校验、batchsize一致性校验。(2) SLIG infershape中loss输出shape未设置(缺SetDimNum/SetDim)；输出dtype只设了index 0，其他输出未设置类型。注意：修复中新引入bug——`dimDq != D_SIZE && dimDq != D_SIZE`两个条件相同，其中一个应为dimDk。
- 修复模式：添加参数范围校验；修正infershape输出shape计算和dtype映射。
- 可审查性：中
- 审查规则建议：infershape函数中所有output都必须有SetOutputDataType调用；注意检测相同子表达式的逻辑运算(copy-paste错误)。

---

### ac0860f7 Fix A3 fullmesh bug
- 根因类别：Pipeline同步缺失 + API参数类型错误
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：A3 16P fullmesh_v2通路下动态量化场景expandx精度错误。(1) PerToken动态量化函数中Cast操作使用PIPE_V，紧接着的Copy依赖该数据但中间没有PipeBarrier<PIPE_V>()同步屏障，Copy可能读到脏数据。(2) `DataCopyExtParams`被误用于`DataCopyPad`参数(应为`DataCopyParams`，字段类型从uint32_t改为uint16_t)；`DataCopyPadExtParams<T>`改为`DataCopyPadParams`。
- 修复模式：插入PipeBarrier<PIPE_V>()保证时序；修正API参数结构体类型匹配。
- 可审查性：低
- 审查规则建议：跨pipe数据依赖(PIPE_V产生→Copy消费)前必须有PipeBarrier；DataCopy系列API参数结构体类型应与API签名严格匹配。

---

### 1fa23e28 modify information for layout and blocksize constrain

非缺陷提交(纯文档修改)。只改了一个.md资料文件，更新blockSize和layout约束描述。

---

### 73c4fdaa FIA的learnablesink的host越界报错修复
- 根因类别：API误用(对可选输入使用必选输入访问接口)
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_tiling.cpp
- 缺陷描述：CheckFAILearnableSink函数中，learnableSink是可选输入(optional input)，但代码使用GetInputDesc和GetInputShape来获取其描述和形状。当该可选输入未提供时，通过必选接口访问导致越界访问，触发host侧报错。
- 修复模式：将GetInputDesc替换为GetOptionalInputDesc，将GetInputShape替换为GetOptionalInputShape，共2处。
- 可审查性：高
- 审查规则建议：对OPTIONAL参数索引的访问必须使用GetOptionalInputDesc/GetOptionalInputShape系列API。可建立静态规则扫描。

---

### 096526e6 修复mc2算子ut的opapi因打桩变更引起的问题

非缺陷提交(UT测试适配)。所有50个文件改动均在tests/ut/op_api/目录，适配打桩(mock)接口变更。

---

### bd4b2d3e 修复BNSD_NBSD尾块搬出异常
- 根因类别：计算逻辑错误(尾块DataCopy的blockLen和偏移量计算错误)
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_block_vec_base.h, attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：BNSD/NBSD layout转置输出场景中MlaTranspose2DataCopy函数存在多个bug：(1) subBlockIdx==1时curGIdx/curS1Idx计算公式有变量依赖错误；(2) 尾块blockLen使用了tailBlock*s1Size而非正确的tailBlock*dSizeV(维度用错)；(3) 尾块attentionOutOffset偏移计算公式错误。infer_flash_attention_kvcache.h中也有对应的偏移计算错误。
- 修复模式：删除错误的subBlockIdx条件分支；修正blockLen维度参数(s1Size→dSizeV)；修正偏移量累加公式；统一使用sOuterOffset代替cubeSOuterOffset。
- 可审查性：中
- 审查规则建议：DataCopy的blockLen涉及维度乘积时检查各维度变量是否与实际数据排布一致；转置layout变换的offset计算需逐维度验证步长。

---

### 71798b1b fix code spelling error

非缺陷提交(代码清理/变量重命名)。变量名拼写纠正(tmpRst→tmpRes, reduceGlobaLoop→reduceGlobalLoop等)、魔数替换为命名常量(3e+99→FLOAT_INF)、移除未使用头文件。所有改动不影响运行时行为。

---

### 6131adfc fix the inaccurate error description for alltoallmatmul&matmulalltoall

非缺陷提交(纯错误信息文案修正)。修改OP_LOGE中的字符串文案，MAX_GROUP_NAME_LEN从128改为127属于报错描述与实际行为对齐(C字符串末尾\0)。

---

### 5080903c 【fix】moe_token_unpermute_with_routing_map_grad补充参数描述

非缺陷提交(纯文档更新)。所有改动在README.md和docs/目录的.md文件，将参数名paddedMode统一替换为dropAndPad。

---

### 9aa05a4b fix gmmsqv2
- 根因类别：空指针解引用 + 入参校验缺失(场景未区分)
- 涉及文件：gmm/grouped_matmul_swiglu_quant_v2/op_host/op_api/aclnn_gmm_dsq_base.h, gmm/grouped_matmul_swiglu_quant_v2/op_host/op_api/aclnn_grouped_matmul_swiglu_quant_utils.h, gmm/grouped_matmul_swiglu_quant_v2/op_host/op_api/aclnn_grouped_matmul_swiglu_quant_v2.cpp
- 缺陷描述：grouped_matmul_swiglu_quant_v2算子在A4W4场景下，代码无条件对weightAssistMatrix进行解引用(`(*gmmDsqParams_.weightAssistMatrix)[i]`)，但A4W4场景weightAssistMatrix应为nullptr(只有A8W4才需要辅助矩阵)。导致A4W4场景传入nullptr时直接崩溃。原代码未区分A4W4和A8W4两种场景。
- 修复模式：新增isA8W4/isA4W4布尔标识通过SetScenario()自动判定场景；添加前置校验(A4W4要求weightAssistMatrix为nullptr)；所有解引用位置增加空值保护。
- 可审查性：中
- 审查规则建议：对可选参数(nullable指针)解引用前必须检查空值；当同一接口支持多种数据类型组合时校验逻辑应明确区分各场景对可选参数的要求。

---

### 0d2c731b sfag增加sparse_block_size校验拦截
- 根因类别：入参校验缺失(参数值未校验导致静默错误)
- 涉及文件：attention/sparse_flash_attention_grad/op_host/arch35/sparse_flash_attention_grad_tiling_bs1_regbase.cpp
- 缺陷描述：SFAG算子tiling中从属性读取selected_block_size参数但缺少合法性校验。当前只支持sparse_block_size=1，若传入非1值后续计算产生错误结果(静默错误)而非报错拦截。
- 修复模式：增加`if (selected_block_size != 1)`校验，不满足时OP_LOGE并返回GRAPH_FAILED。仅4行代码。
- 可审查性：高
- 审查规则建议：从属性/配置读取参数值后，应检查是否有合法范围校验；算子仅支持特定参数值时必须在入口显式拦截不支持的值。

---

## 批次6排除的非缺陷提交

| hash前8位 | commit message | 排除原因 |
|-----------|---------------|---------|
| 3c508d74 | 删除冗余函数，修复UT | UT测试修复 |
| fd0ed91f | Change Doc, single machine dc | 纯文档更新 |
| 3ea26544 | fix(kv_rms_norm_rope_cache) tilingkey分支 | 性能优化 |
| 15ff1476 | gmm_add aclnn description fix | 纯文档注释修正 |
| 1fa23e28 | modify information for layout and blocksize | 纯文档修改 |
| 096526e6 | 修复mc2算子ut的opapi | UT测试适配 |
| 71798b1b | fix code spelling error | 代码清理/重命名 |
| 6131adfc | fix inaccurate error description | 错误信息文案修正 |
| 5080903c | 补充参数描述 | 纯文档更新 |

---

## 批次6缺陷类别统计

| 缺陷类别 | 数量 | 涉及hash |
|----------|------|---------|
| 构建配置缺陷(算子集成遗漏) | 1 | 80329f3a |
| 文档约束公式运算优先级错误 | 1 | 4e4f9168 |
| 算子注册基础设施缺失 | 1 | 613510ad |
| 环境兼容性缺陷(脚本硬编码sudo) | 1 | 08fbc029 |
| 输入校验缺失+infershape输出dtype错误(多重) | 1 | 03f73405 |
| Pipeline同步缺失+API参数类型错误 | 1 | ac0860f7 |
| API误用(可选输入用必选接口访问) | 1 | 73c4fdaa |
| 计算逻辑错误(尾块搬出偏移/blockLen维度错误) | 1 | bd4b2d3e |
| 空指针解引用+场景未区分 | 1 | 9aa05a4b |
| 入参校验缺失(参数值未校验) | 1 | 0d2c731b |

---

## 批次7: 提交 #119-#138 (2026-02-05 ~ 2026-02-07)

---

### c4385241 [FIA] fix oom bug for antiquant
- 根因类别：tiling数据结构选择错误导致OOM
- 涉及文件：fused_infer_attention_score_template_tiling_key.h, incre_flash_attention_entry_regbase.h, incre_flash_attention_template_tiling_key.h
- 缺陷描述：FIA/IFA的antiquant场景tiling key配置中，`ASCENDC_TPL_TILING_STRUCT_SEL`选择了`IncreFlashAttentionTilingDataV2`，而实际应使用`FlashAttentionScoreSimplifiedTilingData`。两个struct内存大小/布局不匹配，导致kernel读取tiling数据时产生OOM。涉及两个tiling_key头文件约90处替换。
- 修复模式：将所有`IncreFlashAttentionTilingDataV2`替换为`FlashAttentionScoreSimplifiedTilingData`
- 可审查性：低
- 审查规则建议：tiling key中的struct类型必须与host侧tiling计算写入的struct一致。可建立自动化检查确保tiling_key.h中的struct与host侧填充的struct匹配。

---

### 1ceed275 fix debug for single machine
- 根因类别：运算符优先级错误(整数除法)
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_host/moe_distribute_dispatch_v2_infershape.cpp
- 缺陷描述：表达式`globalBsReal * 2 * k * (*epWorldSize) / RANK_NUM_PER_NODE`中，整数乘除左结合导致`/`最后执行。语义上`(*epWorldSize) / RANK_NUM_PER_NODE`应先计算，但实际`globalBsReal * 2 * k * (*epWorldSize)`先乘完再除，整数截断导致结果与预期不同。单机模式下infershape输出尺寸与PTA不一致，是PR#1229引入的回归bug。
- 修复模式：添加括号`((*epWorldSize) / RANK_NUM_PER_NODE)`强制先做除法
- 可审查性：高
- 审查规则建议：涉及整数除法的复合表达式，必须确认运算顺序是否符合语义。乘除混合时强制使用括号消除歧义。

---

### d452dde6 修复MLA非量化sparse0/4分核错误
- 根因类别：GQA维度缩放(gSize)遗漏
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：MLA非量化路径的`GetPreNextTokensLeftUp`函数中，preTokens/nextTokens/actualSeqLengthKV在计算时未乘以gSize。MLA模式下KV维度经过group-size压缩，Q侧与KV侧序列长度存在gSize倍数关系，直接比较维度不匹配导致分核数计算错误。sparse mode 0和4下均受影响。
- 修复模式：增加`enableIFAMLA`分支，MLA模式下对preTokens、nextTokens、actualSeqLengthKV乘以gSize
- 可审查性：中
- 审查规则建议：MLA/GQA场景中Q-KV维度交叉计算处都须检查gSize缩放。

---

### d78096ed fa_fag_add_input_error_log
非缺陷提交(纯日志增强)。将FA/FAG的aclnn API层`CHECK_RET`替换为`OP_CHECK`+`OP_LOGE`，增加参数名称日志，不改变运行时逻辑。

---

### 156446e3 MlaProlog fix tilingkey-not-found issue in per-tile scenario
- 根因类别：编译宏过度约束导致tiling key缺失
- 涉及文件：attention/mla_prolog/op_kernel/mla_prolog_template_tiling_key.h
- 缺陷描述：MlaProlog per-tile量化场景的tiling key编译宏中，对`ORIG_DTYPE_KR_CACHE`增加了`== DT_BF16`约束。per-tile场景下kr_cache dtype不一定是BF16，使用其他dtype时编译宏不满足，tiling key不被编译，运行时匹配失败。
- 修复模式：移除编译宏中对`ORIG_DTYPE_KR_CACHE`的dtype约束
- 可审查性：中
- 审查规则建议：tiling key的`#if`编译宏新增dtype约束时，必须枚举该输入所有合法dtype组合，避免遗漏。

---

### 81213d24 fix leftpadding
- 根因类别：GQA维度缩放(gSize)遗漏 + 初始化flag遗漏(双重缺陷)
- 涉及文件：attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：(1) `InitQueryLeftPaddingSize`中，GQA合轴场景下`actualS1Size`包含group维度倍数，直接用`s1Size - actualS1Size`计算padding大小得到错误值。(2) `SetAttributeInfo`中，`enableLeftPadding`为true时未设置`needInit = 1`，导致非量化left padding场景下output未刷零、LSE未初始化为inf，残留脏数据。
- 修复模式：(1) GQA场景下用`actualS1Size / constInfo.gSize`替代。(2) `enableLeftPadding`时显式设置`needInit = 1`。
- 可审查性：中
- 审查规则建议：多模式(GQA/MHA)共用计算路径时，审查维度语义一致性。涉及buffer初始化flag的新feature分支须检查初始化标志是否正确设置。

---

### 8adc6e89 AllGatherAdd bugfix
- 根因类别：函数重复定义导致编译错误
- 涉及文件：examples/mc2/all_gather_add/op_host/op_tiling/all_gather_add_tiling.cpp
- 缺陷描述：`GetWorkspaceSize`函数单独定义为static函数，与其他编译单元同名函数重复定义，导致CI编译失败。
- 修复模式：删除独立函数定义，将逻辑内联到调用处
- 可审查性：高
- 审查规则建议：新增函数时搜索全局是否已有同名定义。

---

### 3cbec10d mlaprologblocksize泛化
非缺陷提交(功能增强)。移除BlockSize硬编码限制{16,128}，泛化为16~1024且16的倍数。

---

### 17362b03 同步qbmmia算子规范资料到transfomer仓
非缺陷提交(纯文档更新)。仅修改.md文档。

---

### 62be3454 修改GQA伪量化支持prefix中的prefixloopcount判断条件
- 根因类别：循环索引偏移遗漏
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_antiquant_block_vec.h, flash_attention_score_antiquant_kernel.h
- 缺陷描述：GQA伪量化KV prefix场景下，判断`runInfo.s2LoopCount < constInfo.prefixLoopCount`未加起始偏移`runInfo.s2StartIdx / constInfo.s2BaseSize`。s2LoopCount是局部计数，不能直接与全局prefixLoopCount比较。遗漏偏移导致误判为prefix循环，读取错误的KV数据源。4处相同逻辑均受影响。
- 修复模式：将`runInfo.s2LoopCount`替换为`(runInfo.s2LoopCount + runInfo.s2StartIdx / constInfo.s2BaseSize)`
- 可审查性：中
- 审查规则建议：循环计数与全局阈值比较时，审查计数是局部还是全局语义，是否需要加起始偏移。同一模式多处出现时确认逻辑一致性。

---

### 8a852918 GQA非量化非BNSD格式actualS1Size计算bugfix
- 根因类别：条件分支遗漏(GQA模式组合未覆盖)
- 涉及文件：attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h
- 缺陷描述：`GetSingleCoreParam`中if-else只处理了hasRope+Aligned576+非BNSD和默认两种情况，缺少GQA+s1Size==1+非BNSD的分支。此场景下正确公式应为`(actualS2Size + preTokensPerBatch) * gSize`，缺失导致actualS1Size被截断为错误值。
- 修复模式：插入`else if`分支处理GQA+s1==1+非BNSD组合
- 可审查性：低
- 审查规则建议：多模式组合(GQA/MHA x BNSD/非BNSD x 量化/非量化)的分支逻辑须系统性枚举所有组合，检查是否遗漏。

---

### 49c60620 优化aclnnFusedInferAttentionScoreV4约束说明格式
非缺陷提交(纯文档格式调整)。将约束说明改为折叠格式。

---

### 8f4dbd41 修复GMM激活函数精度问题，补充tiling处acttype枚举值
- 根因类别：短路逻辑跳过了必要的状态设置
- 涉及文件：gmm/grouped_matmul/op_host/op_tiling/arch35/grouped_quant_matmul_tiling.cpp, .h
- 缺陷描述：`CheckActiveMode`中，`wScale`为2维+shape=(g,1)+nSize==1时，外层if短路跳过了所有scale维度校验。但这导致`bQuantMode`未被正确设置为`PERCHANNEL_MODE`。GMM激活函数N=1场景下量化模式错误导致精度出错。
- 修复模式：(1) 删除nSize==1时的短路分支，统一走校验逻辑；(2) nSize==1时显式设置`bQuantMode = PERCHANNEL_MODE`；(3) 补充`GMMActType`枚举定义
- 可审查性：中
- 审查规则建议："满足条件X则跳过校验"的逻辑，须追问跳过后是否有副作用(如状态设置)也被跳过。

---

### 15eccf03 测试程序环境变量警告信息指引性改善
非缺陷提交(日志/提示信息增强)。增加指引用户查看文档的LOG_PRINT。

---

### 11031c07 修复examples目录下示例执行失败问题
- 根因类别：构建配置遗漏(CMake列表未同步)
- 涉及文件：cmake/custom_build.cmake, docs/zh/develop/aicore_develop_guide.md
- 缺陷描述：CMake中`add_subdirectory`添加example后，缺少`list(APPEND OP_DIR_LIST ...)`，导致示例虽编译但执行阶段找不到构建产物。
- 修复模式：add_subdirectory后增加list(APPEND OP_DIR_LIST ...)
- 可审查性：高
- 审查规则建议：CMake中add_subdirectory与列表变量存在配对使用模式时，新增调用须同步维护相关列表。

---

### 7e8c30fa moe_gating_top_k_softmax精度问题处理
- 根因类别：向量mask merge模式隐式推导导致精度错误
- 涉及文件：moe/moe_gating_top_k_softmax/op_kernel/arch35/moe_gating_top_k_softmax_fullload_generalized_regbase.h
- 缺陷描述：调用`AscendC::MicroAPI::Max`时未显式指定`MaskMergeMode::MERGING`模板参数。带mask的Max归约中，非MERGING模式下mask为0的lane可能被置零或写入脏值，导致`reduceMidRreg`混入脏数据，softmax精度异常。
- 修复模式：显式添加`<float, AscendC::MicroAPI::MaskMergeMode::MERGING>`模板参数
- 可审查性：低
- 审查规则建议：所有带mask的向量intrinsic调用须显式指定MaskMergeMode。可建立lint规则：MicroAPI::Max/Min传入mask参数时必须显式指定模板参数。

---

### e747156a 【fix】moe_token_unpermute_with_routing_map_grad约束公式修复
非缺陷提交(纯文档修复)。修改README和API文档中约束公式括号位置和表格内容。

---

### 16bc59a1 [FIX] Modify the GQA per-block fullquant offset calculation
- 根因类别：per-block量化offset计算公式错误(CeilDiv不满足分配律)
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_block_vec_base.h, flash_attention_score_kernel_base.h, flash_attention_score_kernel_infer.h, util_regbase.h
- 缺陷描述：NTD layout下FP8 per-block fullquant场景，deScale offset用`runInfo.s1SizeAcc >> 7`(累积seqlen/128)计算。但GQA多batch场景下，per-block量化scale个数应为逐batch `CeilDiv(seqlen, blockSize)`后求和。例如batch0 seqlen=129,batch1 seqlen=127，正确结果=CeilDiv(129,128)+CeilDiv(127,128)=3，而原写法(129+127)>>7=2。
- 修复模式：新增`s1ScaleNumAcc`/`s2ScaleNumAcc`累积变量，逐batch做CeilDiv后累加，替代简单右移
- 可审查性：中
- 审查规则建议：`sum(a_i)/B`与`sum(CeilDiv(a_i,B))`不等价。向上取整/整数除法对求和不满足分配律。per-block分块场景中偏移量应逐元素CeilDiv后累加。

---

### c25b4a43 FIA splitFuse 场景算子scalar耗时增加问题修复
- 根因类别：性能回退(变量存储位置不当导致间接寻址开销)
- 涉及文件：attention/fused_infer_attention_score/op_kernel/flash_attention_regular.h
- 缺陷描述：splitFuse重构PR(!1175)中，16个`GlobalTensor`变量从局部变量改为类private成员变量。热循环中访问成员变量需通过`this`指针间接寻址，比局部变量多一层间接跳转。小batch短kvseqlen场景下scalar耗时显著劣化。
- 修复模式：将GlobalTensor恢复为局部变量，通过`GlobalTensorBundle`结构体引用传递
- 可审查性：中
- 审查规则建议：AscendC kernel热路径上的GlobalTensor等频繁访问对象优先用局部变量。"局部变量提升为成员变量"的重构须关注性能影响。

---

### 5b3eca4a aclnnGroupedMatmulWeightNz伪量化参数删除actType
非缺陷提交(文档修正)。从API文档中移除actType的"为空"描述。

---

### 批次7统计

批次7: 13条缺陷 + 7条非缺陷 (65.0%)

| 根因类别 | 频次 | commit hash |
|---------|------|-------------|
| GQA维度缩放(gSize)遗漏 | 3 | d452dde6, 81213d24, 8a852918 |
| 构建配置缺陷 | 2 | 8adc6e89, 11031c07 |
| tiling数据结构选择错误(OOM) | 1 | c4385241 |
| 运算符优先级错误(整数除法) | 1 | 1ceed275 |
| 编译宏过度约束(tiling key缺失) | 1 | 156446e3 |
| 初始化flag遗漏 | 1 | 81213d24(双重缺陷) |
| 循环索引偏移遗漏 | 1 | 62be3454 |
| 短路逻辑跳过必要状态设置 | 1 | 8f4dbd41 |
| 向量mask merge模式隐式推导 | 1 | 7e8c30fa |
| per-block量化offset公式错误(CeilDiv) | 1 | 16bc59a1 |
| 性能回退(变量存储位置) | 1 | c25b4a43 |

---

## 批次1-7累计统计

总计138条提交:
- 实际缺陷修复: 75条 (54.3%)
- 非缺陷(排除): 63条 (45.7%)

批次7: 13条缺陷 + 7条非缺陷 (65.0%)

## 批次8: 提交 #139-#158 (2026-02-04 ~ 2026-02-05)

---

### a5f8edd2637010fdbaeecd953d5cd04a62ea022e fix gmmsq sync
- 根因类别：硬件流水线同步缺陷
- 涉及文件：gmm/grouped_matmul_swiglu_quant/op_kernel/grouped_matmul_swiglu_quant_a8w4_msd_post.h
- 缺陷描述：A8W4后处理流水线中，多个函数共用同一个mmOutQueue，通过EnQue/DeQue在函数间传递tensor。问题：(1) customDataCopyIn中对同一buffer先做fp16 DataCopy再Cast为fp32，中间用EnQue/DeQue"切换"数据类型视图，但queue同步语义无法保证Cast写完再读，实际需要PIPE_V屏障；(2) MulPertokenScale中对同一HardEvent S_V连续做3组SetFlag/WaitFlag，其中2组完全冗余，消耗事件ID且引入不必要流水线停顿，极端情况下事件ID耗尽导致死锁。
- 修复模式：queue-based隐式同步改为PipeBarrier显式同步；tensor从函数局部提升到类成员变量；删除冗余SetFlag/WaitFlag
- 可审查性：低
- 审查规则建议：检测TQue在相邻函数间"EnQue后紧跟DeQue"且中间无其他生产者的模式——queue仅被用作同步屏障而非真正生产-消费；检测同一HardEvent重复SetFlag/WaitFlag超过2次

---

### 236a5fdb7a7d76cfcfcd307d0c831a106ca541c5 gmm add empty tensor check
- 根因类别：输入校验缺失（空tensor零维度未检查）
- 涉及文件：gmm/common/cgmct/kernel/kernel_grouped_matmul.h, gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul.cpp, gmm/grouped_matmul/op_host/op_tiling/arch35/grouped_no_quant_matmul_tiling.cpp/.h
- 缺陷描述：GMM算子三层均缺空tensor处理。kernel层：仅SPLIT_M检查M<=0、SPLIT_K检查K<=0，N维度完全未处理。host API层：CheckZeroShape只遍历x列表不查weight列表，weight为空时继续走matmul。tiling层：5个shape解析路径均未检查K=0，K=0进入matmul tiling计算导致除零或无效配置。
- 修复模式：三层防御——kernel统一"M/K/N任一<=0即跳过"；API新增CheckEmptyTensor入口校验；tiling新增kZero标志位在5条路径中追踪K=0
- 可审查性：中
- 审查规则建议：对矩阵运算算子检测GetDim()返回值未做零值校验就参与tiling/除法的路径；检测groupType分支中只对部分维度做零值检查而遗漏其他维度

---

### f8882f78bf19bf13726a2f11560c70d9e6d250b3 [FIA]fixbug:float8_e5m2拦截
- 根因类别：输入校验缺失（不支持数据类型静默通过）
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_infershape.cpp
- 缺陷描述：InferDataType通过TORCH_DTYPE_ENUM_VALUE_TO_GE_DTYPE_MAP做类型映射，float8_e5m2(enum=23)不在map中，find miss后代码以默认类型(fp16)静默继续，不报错。但float8_e5m2在当前算子不支持，用默认类型替代导致输出精度错误或后续算子类型不匹配崩溃，用户无感知。
- 修复模式：在映射查找前增加不支持类型黑名单校验，命中则返回GRAPH_FAILED
- 可审查性：高
- 审查规则建议：检测map.find()在miss分支没有显式错误返回而fall-through使用默认值的路径；对用户可控枚举输入应有白名单+黑名单双重校验

---

### 99a876f975e25dd848d5c332cee5fa4e0be13eee 修复计算attenOut的singleCoreSize时可能出现减翻的问题
- 根因类别：无符号整数下溢（uint32_t减法溢出）
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_block_vec_infer.h
- 缺陷描述：InitOutputSingleCore用`totalOutputSize - aivIdx * singleCoreSize`计算tailSize(uint32_t)。当输入shape较小(如[1,3,297])时后面核分不到数据，aivIdx*singleCoreSize > totalOutputSize，uint32_t减法下溢为接近2^32的巨大值。随后取min(tailSize, singleCoreSize)=singleCoreSize，该核在错误GM偏移处写入数据，触发aiverr硬件访存异常。
- 修复模式：减法前判断差值>0，否则clamp to 0
- 可审查性：高
- 审查规则建议：检测uint32_t/uint64_t减法未预先判断"被减数>=减数"的模式；多核场景特别关注"total - coreIdx * perCoreSize"形式

---

### 662f162cf8bf14a755e180c5503661cc30d536ee fix a synchronization issue of dispatch v2 fullmesh
- 根因类别：硬件同步事件方向错误
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：MoE dispatch v2 fullmesh中if/else两条路径处理expertIds。if分支DataCopyPad(MTE2)→SyncFunc<MTE2_V>()→Select(V)正确。else分支原代码用SyncFunc<V_MTE2>()，语义是"等V完成后启MTE2"，但实际需要的是"MTE2搬入完成后做V计算"，方向完全反。实际else分支复用已有UB数据，需要的是PipeBarrier<PIPE_V>()保证前序V计算完成。
- 修复模式：删除错误方向同步原语，替换为正确的PIPE_V屏障
- 可审查性：中
- 审查规则建议：检测SyncFunc模板参数方向是否与数据流一致；检测if/else分支同步原语类型不对称且缺少注释说明

---

### 65cafae00727f428756ac277a02ca4dc6e90a5d5 fix syncAll
- 根因类别：调度模式设置时序/路径遗漏（导致多流死锁）
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_tiling.cpp, attention/incre_flash_attention/op_host/incre_flash_attention_tiling.cpp, attention/incre_flash_attention/op_host/incre_flash_attention_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_arch38.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：IFA/PFA/FIA kernel使用SyncAll全核同步，要求batch mode调度。原代码SetScheduleMode(BATCH_MODE_SCHEDULE)存在：(1) 时序错误——放在DoSubOpTiling调用之后，但框架在DoSubOpTiling返回后即可能读取调度配置；(2) 路径遗漏——FIA SplitFuse和PFA arch38中完全没有SetScheduleMode调用。导致多流调度下核因调度延迟未启动，先到达SyncAll的核永久死锁。
- 修复模式：将SetScheduleMode移到DoSubOpTiling入口处；补全缺失路径
- 可审查性：中
- 审查规则建议：检测kernel使用SyncAll但host tiling缺少SetScheduleMode(BATCH_MODE_SCHEDULE)的路径；检测SetScheduleMode在DoSubOpTiling/SetTilingData之后（时序过晚）

---

### 7a03291632b83e4e592779c2cdceb88236d3cd79 [FIA] fix GQA per-block omm bug
- 根因类别：数组越界 + tiling struct类型不匹配
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_block_vec_base.h, attention/fused_infer_attention_score/op_kernel/fused_infer_attention_score_template_tiling_key.h, attention/prompt_flash_attention/op_kernel/arch35/prompt_flash_attention_entry_regbase.h, attention/prompt_flash_attention/op_kernel/arch35/prompt_flash_attention_template_tiling_key.h
- 缺陷描述：两个独立缺陷——(1) GQA per-block场景deScaleKvOffset为0时`deScaleKvOffset-1`产生下溢/越界访问；(2) FP8/HiFloat8 GQA非MLA场景使用PFAFullQuantTilingData而非FlashAttentionScoreSimplifiedTilingData，tiling数据结构不匹配引发OOM。
- 修复模式：(1) 增加边界值判断保护；(2) 更正模板tiling struct选择和tiling key分发条件
- 可审查性：中
- 审查规则建议：检查无符号整数减法是否有下溢保护；检查模板特化中tiling struct类型是否与实际场景一致

---

### 473010ebb375745d34258735b95978f616ccf5fd FIA constinfo struct optimize
- 非缺陷。结构体成员按类型大小重排(uint64_t→uint32_t→bool)减少编译器对齐填充的内存浪费，纯内存布局优化。

---

### 8c1f71ff535b6372d9d8ff197f0a85d70d99d9d8 修复moe_init_routing_v2_grad累加精度丢失问题
- 根因类别：数值精度缺陷（浮点累加顺序导致精度损失）
- 涉及文件：moe/moe_init_routing_v2_grad/op_kernel/arch35/moe_init_routing_v2_grad_base.h
- 缺陷描述：SequenceReduceSum中K维度顺序逐个累加(sum+=x[k])，K较大时累加器sum不断增大，后续小值浮点数加到大sum上发生精度截断（"大数吃小数"），导致moe_init_routing_v2_grad梯度精度不达标。
- 修复模式：顺序累加改为4路分块二分归约(binary tree reduction)——两两先加再合并，减少量级差异带来的精度损失
- 可审查性：中
- 审查规则建议：检查循环累加浮点数的场景是否使用精度友好归约方式（Kahan summation/tree reduction），特别在GPU/NPU向量化代码中

---

### 6978efa7716c1bcb7187420c91b007205cf2bfa8 整改L0接口内QuantGroupedMatmulInplaceAdd的Infershape入参顺序
- 根因类别：接口调用参数顺序错误
- 涉及文件：gmm/quant_grouped_matmul_inplace_add/op_host/op_api/quant_grouped_matmul_inplace_add.cpp
- 缺陷描述：INFER_SHAPE宏OP_INPUT参数顺序与算子注册input顺序不一致——scale1Optional(可选参数)被放在scale2前面，OOM框架按错误位置解析tensor shape，推断出错误的输出大小，OOM模式下触发越界检查报错。
- 修复模式：将scale1Optional移到OP_INPUT列表末尾，与算子注册顺序一致
- 可审查性：高
- 审查规则建议：自动比对INFER_SHAPE中OP_INPUT参数列表与算子proto定义的input顺序是否完全一致，可选参数应在末尾

---

### 17de35a6c2edd6f01c152a57c93679acb705a36c (tiling+gentask+infershape+register)replace socVersion with npuArch
- 非缺陷。大规模API迁移重构，mc2模块全域平台判断从socVersion替换为npuArch，涉及74个文件，提升平台抽象层级。

---

### fbf471b1a66c04e46f718697ec2c262ff9a53924 SFAG example问题修复
- 非缺陷（文档修复）。API文档示例代码中actSeqQLenshape/actSeqKvLenshape从int32_t改为int64_t，不影响生产代码。

---

### 6cce8b90633c999e6810531c758915f0302bbfc1 fix scatter_pa_kv_cache ub tiling
- 根因类别：buffer大小计算未对齐（RoundUp遗漏）
- 涉及文件：attention/scatter_pa_kv_cache/op_host/scatter_pa_kv_cache_tiling_arch35.cpp
- 缺陷描述：UB切分计算时numKHeadSize/numVHeadSize直接用原始headSize乘seqLen，但实际kernel运行时headSize需按dtypeByteSize做RoundUp对齐。reduceBuf/divideBuf/castBuf大小计算同理使用未对齐headSize。导致tiling阶段UB需求偏小，kernel执行时UB空间不够引发数据越界。
- 修复模式：在size计算路径中加入缺失的RoundUp对齐步骤
- 可审查性：高
- 审查规则建议：buffer大小计算涉及硬件对齐要求时，检查所有参与维度是否已正确对齐；搜索seqLen*headSize模式确认headSize是否需先对齐

---

### e61e5b81219ebc0bc532f4b9ca0ec20be6cffe8e fix tensorlist bug
- 根因类别：条件分支遗漏（多模式区分不当）
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：(1) tensorlist模式下emptyTensor判定检查keyInputShape/valueInputShape的StorageShape是否为0，但tensorlist首batch shape可能为0并非空tensor，误判导致跳过该batch计算；(2) 序列长度校验s<=0应为s<0，tensorlist首batch seq=0是合法场景被误拦截。
- 修复模式：tensorlist和非tensorlist场景分别判断emptyTensor；边界条件放宽允许seq=0
- 可审查性：高
- 审查规则建议：同一代码服务多种输入模式(tensorlist vs 普通tensor)时检查条件是否正确区分模式差异；输入校验边界值(<=0 vs <0)确认0是否合法

---

### 10009be89313c38b33a654b8e8b8b8317a68d993 fix license
- 非缺陷。OAT.xml许可证扫描配置维护，合并年份特定的license matcher为通用版本。

---

### ccd9bc9bcac4d6553327b792f005108c4cb54e9b fix rowinvalid
- 根因类别：边界条件保护缺失（负值未clamp）
- 涉及文件：attention/common/op_kernel/arch35/infer_flash_attention_kvcache.h
- 缺陷描述：FlashAttention推理KVCache路径中actualS1Size由原始值减去preToken/nextToken无效部分。当全部被裁剪时actualS1Size变负数，后续作为循环次数或内存偏移导致未定义行为/越界/kernel崩溃。
- 修复模式：减法后增加if(actualS1Size<0) actualS1Size=0下界保护
- 可审查性：高
- 审查规则建议：涉及减法得到的size/count/length变量检查是否有负值可能并做下界保护；关键模式：size=a-b后未检查size<0

---

### d76a776a5203587cbf393f1d7566385b1d972135 fix pipe V_S
- 根因类别：流水线同步缺失（Vector-Scalar pipeline间缺屏障）
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：DataCopyPad(Vector pipeline)将数据写入workspace后紧接调用UpdateTokenNumsOut(可能使用Scalar pipeline读取)，两个pipeline间缺少V_S同步屏障，Scalar侧可能读到Vector侧未写完的数据，产生数据竞争造成间歇性计算错误。
- 修复模式：在V写和S读之间插入SyncFunc<HardEvent::V_S>()硬件同步
- 可审查性：中
- 审查规则建议：DataCopyPad后紧跟非同pipeline数据消费时检查是否有对应SyncFunc/PipeBarrier

---

### 9691bcc3fb03ef0d564d44992976529884964c16 dispatch v2 fullmesh 2-dim mask fix bug
- 根因类别：buffer大小计算单位错误 + 分配不足
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h, mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：三个相关缺陷——(1) bsAlign256/bsKAlign256计算中Ceil(x*sizeof(half), ALIGNED_LEN_256)*ALIGNED_LEN_256已是字节数，再除sizeof(half)使单位变为元素数，后续比较和分配用错大小；(2) fullmesh MaxSizeCal缺少bsKAlign256对maxSize_的比较更新；(3) subExpBuf_ InitBuffer用expertIdsSize_但2-dim mask场景需更大空间，分配不足导致buffer越界。
- 修复模式：去除多余除法修正单位；buffer大小统一取max；InitBuffer改用maxSize_
- 可审查性：高
- 审查规则建议：buffer大小计算链路中检查对齐后值是否被多余类型大小除法破坏单位一致性；多用途复用buffer时检查是否取所有用途最大值

---

### e3ba65148240223012acda57a2a9c368d158a1c0 【bugfix/update】ffag支持D为特殊值/ffa examples修改
- 根因类别：输出格式配置错误 + 数据类型不匹配（混合提交）
- 涉及文件：attention/fused_floyd_attention_grad/op_host/fused_floyd_attention_grad_tiling_s1s2_bn2gs1s2.cpp, attention/fused_floyd_attention/examples/test_aclnn_fused_floyd_attention.cpp, attention/fused_floyd_attention/docs/aclnnFusedFloydAttention.md
- 缺陷描述：(1) FusedFloydAttentionGrad中mm2输出被条件配置为NZ格式(D=72/80/88/96时)，但NZ格式在这些case下产生不正确结果(5个case失败)。修复为统一ND格式。(2) example中attentionOut tensor创建类型ACL_FLOAT应为ACL_FLOAT16，类型不匹配导致内存大小错误和计算异常。(3) 文档重写属非缺陷部分。
- 修复模式：条件分支简化为固定安全值(统一ND格式)；数据类型修正
- 可审查性：中
- 审查规则建议：输出格式(NZ/ND)选择的条件分支中确认所有D维度值经过正确性验证；tensor创建时检查aclDataType是否与算子输出类型一致

---

### 69698404f178faa18bcbf848f6b924325737ab0e [FAG] fix old deter workspace size bug
- 根因类别：workspace大小计算错误（类型宽度+数量因子错误）
- 涉及文件：attention/flash_attention_score_grad/op_host/arch35/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp
- 缺陷描述：确定性计算路径workspace存3组int64偏移量(query/key/value GmOffset)。旧代码用maxValidBBLen*aicNum*FP32_BYTES*NUM_TWO计算大小，两处错误：(1) 偏移量是int64(8字节)但按FP32_BYTES(4字节)算，少一半；(2) 乘NUM_TWO(2)但实际3组偏移量，少一组空间。workspace申请不足导致AIC运行时内存越界引发硬件异常。
- 修复模式：修正数据类型宽度(4→8)、数量因子(2→3)和对齐因子
- 可审查性：高
- 审查规则建议：workspace大小计算中检查每个乘法因子语义是否对应实际数据类型和数据组数；关键模式：sizeof类型与实际写入数据类型不匹配

---

### 批次8统计

总计20条提交(#139-#158):
- 实际缺陷修复: 16条 (80.0%)
- 非缺陷(排除): 4条 (20.0%)

非缺陷: 473010eb(结构体优化), 17de35a6(API重构), fbf471b1(文档修复), 10009be8(license配置)

批次8新增缺陷类别分布:
- 硬件流水线同步缺陷(3): a5f8edd2, 662f162c, d76a776a
- workspace/buffer大小计算错误(3): 6cce8b90, 9691bcc3, 69698404
- 输入校验缺失(2): 236a5fdb, f8882f78
- 无符号整数下溢/边界保护缺失(2): 99a876f9, ccd9bc9b
- 调度模式设置时序/路径遗漏(1): 65cafae0
- GQA相关(数组越界+tiling struct不匹配)(1): 7a032916
- 数值精度缺陷(浮点累加)(1): 8c1f71ff
- 接口参数顺序错误(1): 6978efa7
- 条件分支遗漏/模式区分不当(1): e61e5b81
- 输出格式配置错误(1): e3ba6514

---

### 累计统计 (批次1-8)

总计158条提交:
- 实际缺陷修复: 107条 (60.5%)
- 非缺陷(排除): 70条 (39.5%)

批次8: 16条缺陷 + 4条非缺陷 (80.0%)
批次9: 16条缺陷 + 3条非缺陷 (84.2%)

累计非缺陷hash: fe0bee0d, e8ffb3b9, 53c21a1d, ae4c91e3, 91ef89b4, 7fbc7bc3, 108b4dd3, af418dfc, 28671df1, 8a09dcd0, 14a289de, f73c0505, 967fed57, f38d4a49, 613ae40b, 1065427e, 79a2aced, 988a83a9, 28daf0ab, be6bcad7, 4d3bbf03, e46032f7, 05e3ba28, e48d4172, 68af3c15, 57a30f3a, 1a62a495, 9081122a, 0162cad4, 5c48d0be, 8f3a5747, 349083a7, bd65dfcc, 9bb73d4c, 46bdb193, a6aa6f17, d650ef8f, 5023d7a8, d3460fbf, 7084b812, 2d497549, a9a25159, af854dcd, 6b173d1a, 4143b07a, 5980e181, 11e82892, 3c508d74, fd0ed91f, 3ea26544, 15ff1476, 1fa23e28, 096526e6, 71798b1b, 6131adfc, 5080903c, d78096ed, 3cbec10d, 17362b03, 49c60620, 15eccf03, e747156a, 5b3eca4a, 473010eb, 17de35a6, fbf471b1, 10009be8, 650bed1a, 12821635, b115d199

---

## 批次9: 提交 #159-#177 (2026-02-02 ~ 2026-02-04)

---

### e5dc9ee0 fix bug : empty tensor——init post quant output size
- 根因类别：计算逻辑错误(整数除法顺序)
- 涉及文件：attention/common/op_host/arch32/fia_tiling_empty_tensor.cpp
- 缺陷描述：FIA空tensor场景下，计算singleCoreSize时先用totalOutputSize除以2*usedCoreNum再除以2(量化缩半)，整数除法截断误差在先除后除的链路上被放大。正确做法是先缩半再分核。
- 修复模式：将isOutQuantEnable判断提前到singleCoreSize计算之前，先对totalOutputSize做/2UL处理，再执行分核的向上取整除法。
- 可审查性：中
- 审查规则建议：多步整数除法运算时，检查除法顺序是否影响精度/截断；先缩减再分配 vs 先分配再缩减的差异。

---

### 9b12d59b 【update/bugfix】ffag输入校验判断修复
- 根因类别：校验宏语义混淆(条件取反错误)
- 涉及文件：attention/fused_floyd_attention_grad/op_host/fused_floyd_attention_grad_tiling_common.cpp
- 缺陷描述：CheckSupportShape函数中4处OP_CHECK_IF调用传入CheckSameShape(...)无取反。OP_CHECK_IF语义是"条件为true时报错"，原代码导致shape相同时报错、不同时通过——逻辑完全反了。
- 修复模式：在4处条件参数前加!取反，即OP_CHECK_IF(!CheckSameShape(...),...)。
- 可审查性：高
- 审查规则建议：使用OP_CHECK_IF等断言宏时必须核实条件语义——条件为true时是"通过"还是"失败"？逐个确认每个调用点的条件方向。

---

### e233e106 修复PFA tiling侧accumOutSize计算公式不配套的问题
- 根因类别：workspace大小计算不一致(tiling侧对齐遗漏)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：GetPFAWorkSpaceSize中accumOutSize计算使用vHeadSize，但kernel侧实际用AlignUp(vHeadSize, BYTE_BLOCK)(32字节对齐)。tiling侧算出的workspace小于kernel侧实际使用量，导致潜在内存越界。
- 修复模式：将两处vHeadSize替换为AlignUp(vHeadSize, BYTE_BLOCK)即headDimAlign。
- 可审查性：高
- 审查规则建议：workspace/buffer大小计算必须与kernel侧实际使用保持一致，特别是对齐要求。建议将对齐后尺寸抽为共用常量或公共函数。

---

### dfd1838a fix isPerformanceFlag
- 根因类别：操作时序错误(循环内vs循环外) + 校验逻辑错误
- 涉及文件：mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.h, mc2/moe_distribute_combine_v2/op_host/op_tiling/moe_distribute_combine_v2_tiling.cpp
- 缺陷描述：(1) kernel侧isPerformanceFlag_的性能信息写回(SetAtomicMax+DataCopyPad)放在for循环内，每次迭代都执行sync+atomic+copy，浪费性能且可能产生中间不正确数据。(2) host tiling侧对performanceInfoStorageShape做"不为空则报错"的检查逻辑有误，在合法场景下会误拦截。
- 修复模式：将isPerformanceFlag_代码块从for循环体内移到循环之后；删除tiling侧错误的校验。
- 可审查性：高
- 审查规则建议：写回/输出操作是否应在循环内还是循环外，特别是涉及atomic操作和sync的代码；新增校验上线前确认所有合法path不会误拦截。

---

### 650bed1a gmm部分日志打印语法错误

非缺陷提交(日志文案修正)。将"inputs is not empty"修正为"inputs are not empty"，纯文案改动。

---

### 05f535a7 [FIA]回退代码
- 根因类别：前序PR引入多处回归错误(Revert)
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_infershape.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：被回退的PR1283引入三个问题：(1) post quant输出dtype推导删掉了默认fallback DT_INT8，map.at()在key不存在时抛异常崩溃；(2) PFA的CheckPrefix错误放宽了tensorlist场景拦截，s==1时也不支持；(3) 新增MLA不支持pseShift的约束是错误的，MLA实际可用pseShift。
- 修复模式：完整回退PR1283所有变更。
- 可审查性：中
- 审查规则建议：修改infershape/dtype推导时确认所有合法dtype都能处理；std::map::at()必须确保key存在否则用find()；放宽或收紧校验前与算子实际能力对齐。

---

### 3ca57b44 修复sink PSE场景下的精度问题
- 根因类别：workspace地址偏移计算遗漏(多段workspace地址重叠)
- 涉及文件：attention/flash_attention_score_grad/op_host/arch32/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_sab.cpp, attention/flash_attention_score_grad/op_kernel/arch32/flash_attention_score_grad_s1s2_bn2gs1s2_sab.h
- 缺陷描述：FlashAttentionScoreGrad在sink场景下workspace需额外分配dsinksum空间。Tiling侧正确推进了workspaceOffsets，但kernel侧计算pseAlibiAddr时完全未考虑dsinksum占用的空间，导致sink+PSE场景下地址重叠、数据互相覆盖。
- 修复模式：tiling数据结构新增sinkDataSize字段，tiling侧计算填充，kernel侧pseAlibiAddr加上该偏移量。
- 可审查性：高
- 审查规则建议：workspace多段使用时，每段起始地址必须考虑所有前序段的累积偏移。新增条件性workspace段时检查所有下游地址计算是否都累加了该段大小。

---

### 12821635 Modify the aclnn issues

非缺陷提交(纯文档修改)。仅修改aclnnQuantGroupedMatmulDequant.md的参数说明表格。

---

### c4fa9e4f 修复GQA非量化支持AttentionSink特性的拦截误改
- 根因类别：输入校验缺失(特性互斥约束遗漏)
- 涉及文件：attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：开发AttentionSink特性时遗漏不支持场景的拦截：(1) IFA伪量化模板不支持learnable sink但未拦截；(2) PFA路径缺少CheckLearnSink函数，未检查与量化模式/pse/alibi/leftpadding/prefix的互斥关系。
- 修复模式：IFA中增加OP_CHECK_IF校验；PFA中新增CheckLearnSink系统性检查互斥关系。
- 可审查性：高
- 审查规则建议：新增算子特性时，检查所有模板路径(IFA/PFA/GQA/MLA等)是否都添加了互斥特性拦截校验。

---

### d865fe6d MLA全量化FD负载均衡出口条件修复
- 根因类别：数值溢出/边界条件判断错误(硬编码精确值匹配)
- 涉及文件：attention/incre_flash_attention/op_host/incre_flash_attention_tiling.cpp
- 缺陷描述：FD负载均衡出口条件使用精确值匹配(s2!=55002)，无法覆盖整网场景s2的合理变化范围。s2过长时计算过程出现Inf(浮点溢出)。
- 修复模式：改为范围判断(SEQ_LEN_MIN_V2=37000, SEQ_LEN_MAX_V2=65536)，通过上界防止数值溢出。
- 可审查性：中
- 审查规则建议：精确值比较(!=某常量)作为动态参数校验时应审查是否改为范围判断；涉及浮点运算的tiling参数检查极端输入下的Inf/NaN风险。

---

### 6ed5ed33 fix call
- 根因类别：短路求值顺序错误
- 涉及文件：gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul.cpp
- 缺陷描述：GMM INT8量化校验中，CHECK_COND宏内先调用CheckIsEnabledActive(gmmParams)再OR isNoActivation。当isNoActivation为true时仍执行了不必要的CheckIsEnabledActive调用，该函数在前置条件不满足时可能行为异常。
- 修复模式：改为isNoActivation || CheckIsEnabledActive(gmmParams)，利用短路求值避免不必要的校验。
- 可审查性：高
- 审查规则建议：CHECK_COND宏中多条件OR组合时，轻量级/无副作用条件放前面，避免前置条件不满足时执行可能有问题的校验函数。

---

### 8c74b0bb fix path 6
- 根因类别：条件分支遗漏(模式豁免缺失)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：(1) CheckPerblockQuantParams中isMaxWorkspace为true时缺少提前返回，导致max workspace场景下误报shape不匹配；(2) CheckNTDLayoutCrossover中GQA head dim限制未豁免enablePerblockQuant && layout=="NTD_TND"组合，导致合法场景被错误拦截；(3) 错误信息硬编码"NTD"而非动态获取实际layout。
- 修复模式：增加isMaxWorkspace提前返回；增加perblock量化NTD_TND豁免条件；错误信息改用动态layoutStr。
- 可审查性：高
- 审查规则建议：多种workspace/layout模式的校验函数中，新增模式后逐一检查现有校验路径是否需要豁免或调整；错误信息参数值应动态获取。

---

### b115d199 mc2算子ut文件整改

非缺陷提交(代码风格整改)。命名风格、缩进、花括号统一，纯格式变更。

---

### a75a92e2 perblock量化场景串行修改
- 根因类别：条件分支遗漏(fallback路径缺失)
- 涉及文件：mc2/matmul_reduce_scatter_v2/op_host/op_tiling/arch35/quant_bmm_reduce_scatter_tiling.cpp, mc2/matmul_reduce_scatter_v2/op_kernel/arch35/quant_bmm_reduce_scatter_fp8_hif8.h
- 缺陷描述：perblock量化场景下orgMValue不满足PERBLOCK_SIZE*rankDim整数倍时，原代码仍走公式化切分路径，SetBatch()中无条件将batch4_设为rankDim。不满足对齐条件时tiling切分错误。
- 修复模式：新增isSerial_标志位，不满足对齐时回退串行模式；SetBatch()增加!isSerial_条件守卫。
- 可审查性：中
- 审查规则建议：有对齐/整除前提的计算路径是否有不满足条件的fallback处理；tiling参数设置是否有缺少条件守卫的情况。

---

### 8ade8c3e Revert "dispatch优化syncall"
- 根因类别：硬件流水线同步缺陷(优化引入正确性问题，Revert)
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h
- 缺陷描述：被revert的提交对MoE dispatch的syncall进行"优化"：(1) UpdateTokenNumsOut()从lastCore_改到FIRST_CORE执行并用ReduceSum重新聚合，改变执行核和数据读取时序；(2) 同步原语从PipeBarrier改为SyncFunc。导致多核场景下数据竞争或读取未就绪数据。
- 修复模式：完整revert，恢复到优化前实现。
- 可审查性：低
- 审查规则建议：多核/多流水线同步优化审查要点：修改执行核是否影响数据依赖；同步原语替换是否保证相同ordering语义；此类优化必须附多卡多专家场景回归测试。

---

### f91e8dee alltoallmatmul修复
- 根因类别：条件守卫缺失 + 多平台分支逻辑错误
- 涉及文件：mc2/allto_all_matmul/op_kernel/arch32/allto_all_matmul.h, mc2/allto_all_matmul/op_api/aclnn_allto_all_quant_matmul.cpp, mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.cpp
- 缺陷描述：(1) kernel中alltoallOut数据拷贝分支缺少isAlltoallOut条件判断，不需要输出时仍拷贝，访问未分配buffer导致越界；(2) aclnn层CheckAllDtypesValid缺少x1ScaleOptional的dtype校验，对nullptr做dtype检查导致空指针；(3) A2平台yDtype和all2AllOutFlag被错误放在DAV_3510分支内，A2平台上始终为默认值。
- 修复模式：增加isAlltoallOut条件判断；增加空指针守卫；将公共赋值提到平台判断之前。
- 可审查性：中
- 审查规则建议：可选输出/输入在kernel和host都必须有null/flag守卫；多平台代码中公共逻辑不应被错误放在特定平台分支内。

---

### f69aa254 fix bugs : PSE feature, qs==1 pseshifts1 > qs, copy falut
- 根因类别：边界条件处理缺失(qs==1索引错误)
- 涉及文件：attention/common/op_kernel/arch32/fia_block_vec_nonquant.h, attention/common/op_kernel/memory_copy.h
- 缺陷描述：PSE搬运逻辑中qs==1但PSE的s1维度大于qs时，使用GetDimS1()获取s1Size(大于1)导致索引计算错位。同时stride应使用GetStrideG()而非GetStrideS1()。
- 修复模式：新增qsEqualOne参数，为true时强制s1Size=1并使用GetStrideG()。
- 可审查性：中
- 审查规则建议：tensor维度与实际处理长度不一致时(broadcast场景)，检查索引计算和stride选择是否基于实际处理长度而非tensor维度。

---

### 4b6e703c ffn2attn fix Sync bug
- 根因类别：硬件流水线同步缺失(MTE3 barrier缺失)
- 涉及文件：mc2/ffn_to_attention/op_kernel/ffn_to_attention.h
- 缺陷描述：FFNToAttention::Process()中：(1) 首次迭代(tokenCnt==0)缺少SyncFunc<HardEvent::S_MTE3>()，scalar可能在MTE3写操作完成前准备下一次搬运；(2) DataCopyPad搬运token数据后缺少PipeBarrier<PIPE_MTE3>()，状态位可能在token数据落地前被更新。
- 修复模式：首次迭代增加SyncFunc<HardEvent::S_MTE3>()；token数据DataCopyPad后增加PipeBarrier<PIPE_MTE3>()。
- 可审查性：低
- 审查规则建议：连续DataCopyPad(MTE3)调用之间，后者作为前者的"完成信号"时必须插PipeBarrier<PIPE_MTE3>；循环首次迭代检查前序同步完整性。

---

### 78282635 fixup bug uint64_t to int64_t for quant_all_reduce
- 根因类别：整数类型错误(unsigned vs signed mismatch)
- 涉及文件：mc2/quant_all_reduce/op_host/quant_all_reduce_infershape.cpp
- 缺陷描述：QuantAllReduceShapeInfo中b/s/bs/hiddenSize/rankNum声明为uint64_t，但来自API的int64_t赋值时负值(-1表示动态shape)会隐式转为极大正数，后续shape计算和校验失效。
- 修复模式：将字段类型从uint64_t改为int64_t，与API类型一致。
- 可审查性：高
- 审查规则建议：infershape/tiling代码中存储shape的变量应使用int64_t，框架API普遍用有符号类型(-1表示动态维度)；审查所有uint64_t用于shape字段的场景。

## 批次10: 提交 #178-#196 (2026-01-30 ~ 2026-01-31)

---

### 93e71beb [FIA]fixbug:打印信息不清晰
非缺陷提交(日志格式修正)。将OP_LOGE中std::string对象改为.c_str()传给%s格式符。虽然传std::string给%s是UB，但属于日志打印层面修正，不影响业务逻辑正确性。

---

### cea3f28f [FIA] fix GQA per-block fullquant pse bug
- 根因类别：条件判断逻辑错误(指针非空检查 vs 布尔标志语义混淆)
- 涉及文件：attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：CheckPerblockQuantParams中per-block量化场景判断是否存在pse(位置偏移)时，使用了`pseType != nullptr`(指针非空检查)，但正确语义应检查`enablePseShift`(布尔标志)。当pseType指针非空但值为0(未启用pse)时，旧代码错误进入检查分支并可能拦截合法输入；反之enablePseShift为true但pseType指针为空时又跳过检查。
- 修复模式：将`pseType != nullptr`替换为语义正确的布尔标志`enablePseShift`
- 可审查性：高
- 审查规则建议：当存在语义明确的布尔标志(如enableXxx)时，优先使用布尔标志而非指针非空检查来判断特性是否启用；审查中应关注"指针非空 vs 功能开关"的语义差异。

---

### c7d7f6b9 [FIA]: fixbug pse\tensorlist\fp8e5m2相关场景拦截
- 根因类别：输入校验缺失 + 类型映射不完整 + 拦截条件不精确
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_infershape.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：三个子问题：(1) infershape中TORCH_DTYPE_ENUM_VALUE_TO_GE_DTYPE_MAP缺少{1, DT_INT8}映射，删除硬编码改为map查找，新增不支持类型(FP8E5M2)的错误返回；(2) MLA场景+pse/alibi未做拦截，缺少OP_CHECK_IF校验；(3) sys_prefix在Q_S=1时应支持tensorlist，拦截条件从enableTensorList改为enableTensorList && (queryShapeInfo.s > 1)，避免误拦截。
- 修复模式：补充类型映射、增加不支持场景的拦截校验、放宽合法场景的拦截条件
- 可审查性：中
- 审查规则建议：新增算子特性时需同步审查所有相关校验路径，确保不支持的组合被拦截、支持的组合不被误拦截；类型映射表变更时检查所有依赖路径的一致性。

---

### e5988e2b revert
- 根因类别：前序PR引入回归(Revert) + 初始化时序依赖
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：对MoeDistributeDispatchV2FullMesh进行实质性重构：(1) 移除对moe_distribute_v2_base.h的依赖及InitWinState辅助函数调用，改为内联直接读写GM状态(DataCacheCleanAndInvalid+直接赋值0/1翻转)；(2) 删除globalBS_成员变量，将axisMaxBS_计算从Init末尾提前到赋值epWorldSizeOriginal_之后修正初始化顺序依赖；(3) 移除函数末尾对selfDataStatusGMTensor_的DataCopyPad写状态操作及多个相关成员变量。
- 修复模式：简化GM状态读写逻辑、消除对外部基类工具函数的依赖、修正变量初始化时序
- 可审查性：中
- 审查规则建议：commit message为"revert"的提交需关注实际内容是否为简单回退还是包含新逻辑；涉及多核/通信场景的状态管理变更需审查并发安全性；成员变量的初始化顺序应与声明顺序和依赖关系一致。

---

### 33202735 修复GMM算子日志打印的若干问题
非缺陷提交(纯日志文本修饰)。仅修改aclnn_grouped_matmul.cpp中大量日志字符串的首字母大写(如"x[%lu] is null" -> "X[%lu] is null")，不涉及任何逻辑、条件或数据流变更。

---

### 60550452 alltoallmatmul 修复int4 8卡性能
- 根因类别：属性推断方式错误(shape推断 vs 属性驱动)
- 涉及文件：mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.cpp, mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.h
- 缺陷描述：x2的转置判断使用了`bool isTrans = info.K * info.rankSize == x2Dim1`(基于shape推断)，而非直接使用用户传入的x2Transpose属性标志。当shape恰好满足等式但实际未转置时(或反之)会导致N维度取值错误，影响后续tiling计算和tilingKey生成。
- 修复模式：将shape推断式转置判断替换为属性驱动判断；调整8卡int4特定shape下的核心分配参数
- 可审查性：高
- 审查规则建议：矩阵转置等属性应直接使用用户传入的属性值判断，而非通过shape反推(shape推断存在歧义性)；tiling参数中涉及核心数分配的变更应要求性能测试数据佐证。

---

### 99342fa4 win区dump解析工具bugfix
- 根因类别：API误用(logging格式占位符缺失)
- 涉及文件：mc2/tools/dump_analysis/dump_analysis.py
- 缺陷描述：`logging.info("... shape:", len(int32_dis_0_status))`使用print风格的逗号拼接传参，但logging.info的格式字符串中没有%占位符消费第二个参数，导致shape信息不出现在日志输出中。
- 修复模式：改为`logging.info("... shape:%d", len(...))`，使用%d格式化占位符正确输出整数值
- 可审查性：高
- 审查规则建议：logging.info/debug/error等调用中若有额外参数，格式字符串必须包含对应数量的%占位符；可通过pylint的logging-format-interpolation规则自动检测。

---

### 731c9344 GMMSwigluQuant pertoken量化模式 支持 aclnn通路
非缺陷提交(新功能开发)。为GMMSwigluQuant算子的pertoken量化模式增加aclnn通路适配，包含新增测试文件、参数校验逻辑扩展、文档更新等。

---

### 78e07300 修复算子MoeTokenUnpermuteWithRoutingMap在空tensor临界情况与标杆输出存在差异的问题
- 根因类别：空tensor场景处理缺失
- 涉及文件：moe/moe_token_unpermute_with_routing_map/op_host/op_api/aclnn_moe_token_unpermute_with_routing_map.cpp
- 缺陷描述：空tensor(如hidden_size==0)临界情况下，原代码空tensor判断条件不够精确(直接判断permutedTokens->IsEmpty() || sortedIndices->IsEmpty())，且空tensor路径下未对输出tensor做零值初始化，导致输出结果与标杆不一致。paddedMode为true时若permutedTokens为空，未跳过后续Reshape/Mul和InplaceIndexAdd操作。
- 修复模式：修正空tensor判断条件为`sortedIndices->IsEmpty() || (paddedMode == false && permutedTokens->IsEmpty())`；空tensor路径下对所有输出执行ZerosLike+ViewCopy初始化；paddedMode路径增加`!permutedTokens->IsEmpty()`守卫
- 可审查性：高
- 审查规则建议：对算子的空tensor/零维度输入路径进行专项审查，确保所有输出tensor在边界条件下都有确定性初始化值而非返回未定义内容。

---

### fcba7723 MLA非量化支持sparse 0,3,4，G泛化支持1,2,4,8,16,32,64,128
非缺陷提交(新功能开发)。大型特性提交(26个文件、1300+行变更)，为MLA非量化flash attention添加sparse mode 0/3/4支持、G参数泛化支持更多值、layout泛化支持转置。

---

### 0b621cb6 [FAG] fix bn2 sprasemode3 bug
- 根因类别：条件分支遗漏(边界变量clamp缺失)
- 涉及文件：attention/flash_attention_score_grad/op_kernel/arch35/flash_attention_score_grad_kernel_base.h
- 缺陷描述：FlashAttentionScoreGrad反向kernel在bn2多块分块模式下，sparseMode为RIGHT_DOWN_CAUSAL(mode 3)时，s2EndLen未被正确限制在s2Size范围内。原代码仅在有prefixN的分支中对s2EndLen做了Min clamp，但在else分支(无prefixN、sparseMode==3)遗漏了上界约束，导致反向计算中S2维度的end长度可能超出实际s2Size，产生越界访问或计算错误。
- 修复模式：在else分支添加`if (constInfo.sparseMode == RIGHT_DOWN_CAUSAL) { s2EndLen = Min(s2EndLen, constInfo.commonConstInfo.s2Size); }`
- 可审查性：高
- 审查规则建议：对含多个sparseMode分支的kernel逻辑，审查每个分支下的边界变量(s2EndLen、s1StartLen等)是否都做了合法范围clamp，尤其注意if/else分支中某一分支有约束而另一分支遗漏的情况。

---

### d1d95dfb [mc2]fix matmul_all_reduce ub conflict
- 根因类别：硬件流水线同步缺失(UB数据竞争)
- 涉及文件：mc2/matmul_all_reduce/op_kernel/arch35/matmul_all_reduce_quant_pertoken_comm_int8.h
- 缺陷描述：MatmulAllReduce的pertoken comm int8通路中，matmul计算和低bit通信都使用vec单元且共享UB空间，缺少全核同步屏障导致前一步vec计算结果可能被后一步通信操作覆盖，造成数据竞争和计算结果错误。
- 修复模式：在hccl_.Commit之后、下一轮matmul开始之前插入SyncAll<false>()全核同步调用，确保vec计算和通信操作之间的流水正确串行化
- 可审查性：高
- 审查规则建议：在涉及多流水(matmul+通信)共享UB/向量单元的算子kernel中，审查每个流水阶段切换点是否有恰当的同步屏障(SyncAll)，特别是当不同操作复用同一硬件单元或同一buffer区域时。

---

### 5dce387d MatmulAllReduce CommFp8 Sync Fix
- 根因类别：硬件流水线同步缺失(SyncAll模板参数错误+跨迭代同步缺失)
- 涉及文件：mc2/matmul_all_reduce/op_kernel/arch35/matmul_all_reduce_quant_commfp8_mixed_calc.h, mc2/matmul_all_reduce/op_kernel/arch35/matmul_all_reduce_quant_pertile_comm_fp8.h
- 缺陷描述：与d1d95dfb同类问题但发生在CommFp8通路。两处缺陷：(1) ElementWiseAdd之后SyncAll()使用默认模板参数(SyncAll<true>())，同步语义不正确应使用SyncAll<false>()；(2) StepOneTurn(matmul+quant+通信一个完整流水步骤)执行完毕后缺少SyncAll<false>()同步屏障，当前轮次的通信/量化操作可能与下一轮matmul产生UB冲突。
- 修复模式：将SyncAll()改为SyncAll<false>()修正模板参数；在StepOneTurn调用后新增SyncAll<false>()确保流水阶段间数据一致性
- 可审查性：高
- 审查规则建议：对SyncAll的模板参数使用进行审查，确认<true>和<false>的语义是否符合场景需求；在多阶段流水(matmul -> quant -> comm)的循环体中审查每轮迭代结束时是否有充分的同步屏障。

---

### a3a76213 kvrmsnormropecache support fp8 hif8 quant and recompute fix
非缺陷提交(新特性为主体)。为kvrmsnormropecache算子新增fp8/hifloat8量化类型支持，将原本硬编码的int8_t扩展为模板参数，涵盖proto注册、tiling校验、kernel计算逻辑等全链路改动。虽然包含若干recompute修复(tensor alloc/free位置调整、RopeWithoutQuant增加kCacheRowOffset参数等)，但这些修复与新特性代码深度耦合，整体归类为新特性。

---

### d9caa9d7 增加FIAv4 ATK工程看护用例，并扩展golden中的部分场景未适配问题
非缺陷提交(测试/CI维护)。改动仅涉及测试目录下的json配置文件和python测试执行器，属于ATK自动化测试框架的CI看护用例增补。

---

### 372b8e80 修改prefix拦截&&FIAV4mask资料修改
- 根因类别：参数传递错误 + 校验执行顺序错误
- 涉及文件：attention/fused_infer_attention_score/op_host/arch32/fused_infer_attention_score_tiling_check_consistency.cpp
- 缺陷描述：两个独立缺陷：(1) CheckSystemPrefixShape中创建FiaTilingShapeCompare时传入错误的name常量KEY_NAME，实际应为KEY_SHARED_PREFIX_NAME(校验的是prefix key而非普通key)，错误标识名导致校验日志和错误提示指向错误张量；(2) systemPrefixLen > systemPrefixMaxLen的长度校验被放在CompareShape之前执行，但shape校验通过是长度校验有意义的前提，存在逻辑依赖关系错误。
- 修复模式：修正KEY_NAME -> KEY_SHARED_PREFIX_NAME；调整校验执行顺序(先shape校验再长度校验)；显式返回成功状态码
- 可审查性：高
- 审查规则建议：函数参数是具有相似命名的常量时(KEY_NAME vs KEY_SHARED_PREFIX_NAME)审查应确认参数语义与上下文一致；多个校验步骤之间存在依赖关系时应按依赖顺序排列。

---

### c4ececa0 fix log and check
- 根因类别：校验条件范围错误(过度拦截)
- 涉及文件：gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul.cpp, gmm/grouped_matmul/tests/ut/op_host/test_grouped_matmu_tiling.cpp
- 缺陷描述：ASCEND950平台上，原代码在进入量化类型分支判断之前无条件执行CHECK_COND(isNoActivation, ...)拦截，导致任何带activation的场景被直接拒绝。然而INT8量化的pertoken-perchannel和pertensor-perchannel模式实际支持activation，这个过早的全局拦截导致合法的INT8+activation场景被错误拒绝。
- 修复模式：删除错误的提前全局拦截，让校验下沉到具体分支内执行精确判断；修正UT平台版本配置
- 可审查性：高
- 审查规则建议：校验逻辑不应在分支判断之前做全局拦截，当存在"某些子场景合法、某些子场景非法"时校验应下沉到具体分支内各自执行。

---

### 3f21ad63 dispatch v2 fix winIn addr and sync bug
- 根因类别：GM地址来源错误 + 硬件流水线同步缺失(多处HardEvent缺失) + PipeBarrier位置错误
- 涉及文件：mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h, mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2_full_mesh.h
- 缺陷描述：三类缺陷：(1) winIn地址错误：核间同步标志的读写使用windowInstatusFp32Tensor_(指向远端window状态地址)，但核间同步标志属于本地rank操作应使用本地rank的winIn地址。(2) 缺失硬件事件同步：SendToMoeExpert中CalTokenSendExpertCnt前缺S_V同步、DataCopyPad前缺V_MTE2同步、statusCleanFp32Tensor_写入前缺V_MTE3同步。(3) PipeBarrier<PIPE_ALL>位置错误：原放在循环结束后，实际需保护的是LocalWindowCopy(含reset操作)。
- 修复模式：修正GM地址来源(远端 -> 本地rank窗口地址)；补充缺失的硬件事件同步(S_V, V_MTE2, V_MTE3)；将PipeBarrier移至正确位置
- 可审查性：低
- 审查规则建议：多核/多rank通信场景中对窗口地址的读写操作必须确认目标是本地rank还是远端rank；所有DMA操作前应确认前序计算阶段已通过对应HardEvent完成同步；PipeBarrier应与其保护的操作紧邻。

---

### 35d0716d alltoallmatmul tiling与A4W4性能修复
- 根因类别：tiling参数缺陷(int4位宽未适配) + 维度校验错误 + GM地址偏移缺失 + 输入校验缺失
- 涉及文件：mc2/allto_all_matmul/op_host/op_tiling/arch32/allto_all_matmul_tiling_910b.cpp, mc2/allto_all_matmul/op_kernel/arch32/allto_all_matmul_a4w4.h
- 缺陷描述：五个缺陷：(1) A4W4的L1/L0 TileShape中k轴切分粒度过小，int4位宽仅fp16的1/4但原配置未利用此特性；(2) pValue(peermem通信buffer容量)未考虑int4的4倍压缩比，三个rank函数均存在；(3) x1Scale维度校验逻辑错误，perTokenScale的第一维应对应alltoall前的完整M轴而非缩小后的M/rankSize；(4) perTokenScaleGM_地址偏移缺失，未加当前rank偏移量导致所有rank读取同一段scale数据；(5) 缺少k轴tokenSize范围校验。
- 修复模式：调整TileShape适配int4位宽；增加pValue的4倍系数；修正x1Scale维度校验条件；增加GM地址rank偏移；新增tokenSize范围校验
- 可审查性：中
- 审查规则建议：tiling参数TileShape应根据数据类型实际位宽调整；通信buffer容量计算必须考虑数据类型压缩比；维度校验条件中涉及rankSize除法时确认是alltoall前还是操作后的维度；GM地址在多rank场景下必须加rank偏移。

## 批次11: 提交 #197-#215 (2026-01-28 ~ 2026-01-30)

---

### a8bc9667 GMMFR，文档格式修复

非缺陷提交(纯文档格式修改)。仅修改5个.md文件的中文格式和表格排布。

---

### c04bfe85 修复archname
- 根因类别：硬件架构标识符错误
- 涉及文件：gmm/grouped_matmul_swiglu_quant_v2/op_kernel/arch35/grouped_matmul_swiglu_quant_v2_pertoken_quant.h
- 缺陷描述：`GmmSwigluAswtPertokenKernel`中`TileCopy`模板参数使用了内部架构代号`Arch::DAV_3510`，应使用对外平台标识`Arch::Ascend950`。错误的架构标识符可能导致kernel编译或运行时选择了错误的tiling/copy策略。
- 修复模式：将`Arch::DAV_3510`替换为`Arch::Ascend950`
- 可审查性：高
- 审查规则建议：Arch枚举值使用时内部代号与对外名称不应混用，统一使用对外平台标识。

---

### 3bb75678 fix fia bug
- 根因类别：多处逻辑缺陷(GQA offset计算错误 + layout分支冗余 + 校验缺失 + 条件判断错误)
- 涉及文件：attention/common/op_kernel/arch35/attenmask.h, attention/common/op_kernel/arch35/flash_attention_score_block_cube.h, attention/common/op_kernel/arch35/flash_attention_score_block_vec_infer.h, attention/common/op_kernel/arch35/flash_attention_score_kernel_base.h, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.h
- 缺陷描述：复合修复，涉及Flash Attention多处独立缺陷：(1) attenmask.h中GQA场景`s1Offset`取模计算错误+多余模板参数；(2) flash_attention_score_block_cube.h中冗余layout分支使用了错误的queryOffset；(3) flash_attention_score_block_vec_infer.h中`PostQuant`的`perChannelQuantGQAOffset`遗漏`vec2S1BaseSize`和`subBlockIdx`维度偏移；(4) tiling层缺少n维度和nQ>256校验，GQA ratio检查逻辑冗余，actualSeqLengths解析条件误加enableIFA。
- 修复模式：删除错误GQA取模逻辑；统一用offsetCalculator替代冗余layout分支；补全多维偏移量计算；tiling层添加维度校验；简化GQA ratio检查；修正解析条件
- 可审查性：低(6文件batch fix)
- 审查规则建议：Flash Attention的offset计算应有UT覆盖GQA多head场景；tiling层对所有shape维度均需边界校验。

---

### 4f8bc97c fix matmulallReduce

非缺陷提交(功能调整/特性裁剪)。删除FP8_E5M2和FLOAT4_E1M2等数据类型支持，简化bias类型检查，属于接口收敛。

---

### f195e8f6 group list upper 128 and weight l2 cache on 950PR/DT

非缺陷提交(功能增强+性能优化)。950PR/DT非量化grouped_matmul支持tensorList长度从128放开至1024，新增weight L2 cache控制。

---

### c6d4e407 gather pa kv cache fix
- 根因类别：越界访问(GM读取在边界检查前) + DMA对齐缺失 + 类型精度丢失
- 涉及文件：attention/gather_pa_kv_cache/op_host/gather_pa_kv_cache_tiling_arch35.cpp, attention/gather_pa_kv_cache/op_kernel/arch35/gather_pa_kv_cache_nd.h, attention/gather_pa_kv_cache/op_kernel/arch35/gather_pa_kv_cache_nz.h
- 缺陷描述：GatherPaKvCache算子seqLens超长时aicore error(issue #608)。三处缺陷：(1) nd/nz kernel中`blockTableOffset`越界时仍先执行`blockTablesGm_.GetValue()`读取GM导致非法内存访问；(2) tiling层`maxUbHiddenSize`未对齐32B导致DMA传输错误；(3) `seqLensCopyParams.blockLen`缺少uint32_t转换，DataCopy系列API应使用带类型的Ext版本。
- 修复模式：将GM读取移至越界判断之后(越界时blockId=0)；maxUbHiddenSize做CeilAlign 32B对齐；修正类型转换和API参数类型
- 可审查性：高
- 审查规则建议：GM访问必须在边界检查之后执行；tiling中DMA相关size参数应保证对齐。

---

### acd150e2 GMM切K场景BASEM小于M时校验报错
- 根因类别：边界条件缺失(tiling base block size未与实际shape取min)
- 涉及文件：gmm/grouped_matmul/op_host/op_tiling/grouped_matmul_tiling.cpp
- 缺陷描述：GMM SPLIT_K场景tiling中`baseM_`硬编码为SPLITK_BASEM_256(256)，当实际`maxM_`<256时，后续矩阵分块调度因baseM>M导致校验报错或计算异常。
- 修复模式：baseM_赋值后增加判断：若baseM_>maxM_则调整为maxM_向上16对齐值
- 可审查性：高
- 审查规则建议：tiling中所有base block size赋值后应与实际shape维度做min/clamp，防止base超过实际大小。

---

### 826d92e0 修复后量化gm init
- 根因类别：数据类型大小计算错误(reinterpret_cast后size/offset不一致)
- 涉及文件：attention/common/op_kernel/arch32/fia_kernel_nonquant.h
- 缺陷描述：FIA非量化kernel中，输出类型为int8_t(后量化)时，output GM使用half指针初始化。`totalOutputSize`按元素数量计算但未换算为half元素个数(int8是1字节，half是2字节)，导致`singleCoreSize`过大。同时InitOutput调用处又错误地对offset/size除以2，结果只初始化了前半部分输出，后半部分含脏数据造成精度问题。
- 修复模式：计算totalOutputSize时提前除以2(换算为half元素数)，InitOutput调用处去掉多余除2，统一到half元素粒度
- 可审查性：高
- 审查规则建议：output GM使用reinterpret_cast改变数据类型时，所有基于该指针的size/offset必须统一到同一类型元素粒度。

---

### d0a870ea 修复MXFP8量化Cast时SAT_MODE不生效导致的精度差异
- 根因类别：硬件控制寄存器(SPR)配置遗漏
- 涉及文件：moe/moe_init_routing_v3/op_kernel/arch35/moe_v3_common.h, moe/moe_init_routing_v3/op_kernel/arch35/moe_v3_gather_mxfp8_quant.h, moe/moe_init_routing_v3/op_kernel/moe_init_routing_v3_apt.cpp
- 缺陷描述：Ascend 3101上MoE V3算子MXFP8量化Cast需要SAT_MODE(饱和模式)，但kernel未设置溢出控制寄存器`OVERFLOW_MODE_CTRL`(SPR #60)，导致Cast使用默认溢出模式，FP32->MXFP8转换产生精度差异。
- 修复模式：Init中通过SetCtrlSpr设溢出模式为0(saturation mode)；算子入口保存原始溢出模式，结束后恢复
- 可审查性：高
- 审查规则建议：低精度量化(FP8/MXFP8)kernel应检查是否需要显式设置溢出模式寄存器；修改SPR后必须在算子退出前恢复原值。

---

### 454df379 fix kernel select bugs of grouped_matmul

非缺陷提交(空提交)。tree与parent完全相同，无实际文件变更。

---

### 8b84d6df 解决MoeInitRouting/MoeInitRoutingQuantV2算子精度问题
- 根因类别：流水线同步缺失(SyncAll + PipeBarrier + TQueSync)
- 涉及文件：moe/moe_init_routing/op_kernel/moe_gather_out_small_activate_row.h, moe/moe_init_routing_quant_v2/op_kernel/arch35/moe_v2_gather_dynamic_quant_droppad.h, moe/moe_init_routing_quant_v2/op_kernel/arch35/moe_v2_gather_quant_simt.h, moe/moe_init_routing_quant_v2/op_host/moe_init_routing_quant_v2_tiling_base.cpp, moe/moe_init_routing/op_host/CMakeLists.txt, moe/moe_init_routing_quant_v2/op_host/CMakeLists.txt
- 缺陷描述：MoeInitRouting和MoeInitRoutingQuantV2在ascend910_95平台精度问题。三处：(1) 多核场景缺少SyncAll()导致核间数据竞争；(2) 量化kernel中Duplicate后缺PipeBarrier<PIPE_V>()导致Div读取未就绪数据；(3) bfloat16 Cast后缺TQueSync<PIPE_MTE3, PIPE_V>同步。
- 修复模式：在关键计算点插入SyncAll/PipeBarrier/TQueSync同步原语；为910_95平台条件添加编译选项
- 可审查性：中
- 审查规则建议：多核kernel和向量流水线连续操作中，检查是否缺少SyncAll/PipeBarrier等同步原语。

---

### 4b905603 fix ScatterPaKvCache tiling dual-in-out
- 根因类别：tiling逻辑分支遗漏(早期return跳过必要参数计算)
- 涉及文件：attention/scatter_pa_kv_cache/op_host/scatter_pa_kv_cache_tiling_arch35.cpp
- 缺陷描述：ScatterPaKvCache tiling在DUAL_IN_OUT模式下直接return，跳过了v维度(vHandleNumPerLoop/vLoopNum/vTailHandleNum)的tiling计算，导致该模式下v相关tiling参数未初始化。
- 修复模式：DUAL_IN_OUT分支内也执行v维度tiling计算后统一return
- 可审查性：高
- 审查规则建议：tiling函数中不同模式的早期return需确认所有必要tiling参数已被正确计算。

---

### 69f8dff2 fix GmmAddCgmct
- 根因类别：函数调用错误(重命名后调用点未同步更新)
- 涉及文件：gmm/grouped_matmul_add/op_kernel/grouped_matmul_add.cpp
- 缺陷描述：split_k场景tiling key对应的分支仍调用已重命名的`GmmAddAct`函数(已在f2173684中改为`GmmAddCgmct`)。
- 修复模式：将`GmmAddAct`调用替换为`GmmAddCgmct`
- 可审查性：高
- 审查规则建议：函数/类重命名后应全局搜索所有调用点确保一致性更新。

---

### b96fd79d fix g generalization
- 根因类别：变量名拼写错误(typo)
- 涉及文件：attention/common/op_kernel/arch35/attenmask.h
- 缺陷描述：GQA模式计算s1Offset时使用了`consInfo.s1Size`(少了t)，应为`constInfo.s1Size`，导致编译失败或运行时访问错误的结构体成员。
- 修复模式：将`consInfo`修正为`constInfo`
- 可审查性：高
- 审查规则建议：结构体/变量名引用应通过编译器严格检查，启用-Werror。

---

### f2173684 cgmct_fix

非缺陷提交(重构/重命名)。将grouped_matmul_add kernel从Act框架迁移到Cgmct框架，纯命名空间和文件名变更。

---

### 4eca5019 修复sink NZ场景下的精度问题
- 根因类别：计算逻辑错误(归约累加用赋值代替累加 + 符号错误)
- 涉及文件：attention/flash_attention_score_grad/op_kernel/arch32/flash_attention_score_grad_post.h
- 缺陷描述：FlashAttentionScoreGradPost在NZ格式处理dsink时两处算法错误：(1) 循环内`dsinkCalc = -vecOut.GetValue(0)`是赋值而非累加，多个数据块归约结果被覆盖；(2) 写入dsinkGm时应取负但直接写了dsinkCalc。两处共同导致NZ场景梯度缩放因子计算错误。
- 修复模式：赋值改累加`dsinkCalc += vecOut.GetValue(0)`；写入时取负`-dsinkCalc`；新增NZ格式sink场景完整处理分支
- 可审查性：中
- 审查规则建议：归约累加循环应检查是否误用赋值(=)代替累加(+=)；最终结果的符号正确性需交叉验证。

---

### b3ebb5b8 barrier的example运行出现问题，进行修复

非缺陷提交(测试示例参数调整)。仅修改example中K和moeExpertNum参数值。

---

### 79c5d19c 修正gmm activetype报错描述不清晰的问题

非缺陷提交(日志/错误信息改进)。改善错误提示信息的可读性。

---

### e21a3ec1 GMMFR MX aclnn通路提交

非缺陷提交(新功能开发)。GMMFR算子新增MX量化模式aclnn通路支持，+2043行新代码。

---

## 批次12: 提交 #216-#235 (2026-01-23 ~ 2026-01-28)

---

### 9cc118d5 fix: remove common/act

非缺陷提交(构建系统清理/重构)。删除common/act和common/groupedmatmul_act目录安装路径，替换为统一的common/cgmct路径。目录结构整理，非缺陷修复。

---

### bb1ca6f9 【gmm】【groupedmatmulinplaceadd】修改报错信息存在空行
- 根因类别：字符串字面量格式错误
- 涉及文件：gmm/grouped_matmul/op_host/grouped_matmul_infershape_quant_checker.cpp, gmm/grouped_matmul/op_host/op_api/aclnn_grouped_matmul_910_95_checker.cpp, gmm/grouped_matmul/op_host/op_tiling/grouped_matmul_tiling.cpp, gmm/quant_grouped_matmul_inplace_add/下2个文件
- 缺陷描述：C/C++中用`\`行续接时，下一行开头的缩进空格被视为字符串内容，导致OP_LOGE输出的报错信息中间嵌入大段空白（几十个空格），可读性极差。
- 修复模式：将`\`续接后下一行的缩进空格全部移除，让字符串内容紧贴行首。
- 可审查性：高
- 审查规则建议：检测字符串字面量中`\`续接后下一行是否以空白字符开头。

---

### e2ba8a5d 修改低错问题
- 根因类别：Git merge冲突标记残留
- 涉及文件：moe/moe_init_routing_v3/README.md, moe/moe_init_routing_v3/docs/aclnnMoeInitRoutingV3.md, moe/moe_gating_top_k/docs/aclnnMoeGatingTopK.md
- 缺陷描述：README.md和API文档中残留了Git merge冲突标记（`<<<<<<< HEAD`、`=======`、`>>>>>>>`），导致文档内容异常，产品支持状态出现矛盾（一处写`√`另一处写`×`）。
- 修复模式：删除merge冲突标记，保留正确分支内容。
- 可审查性：高
- 审查规则建议：CI/pre-commit hook应检测`<<<<<<<`/`=======`/`>>>>>>>`merge冲突标记。

---

### 9b60bd2a GMM 类算子blockdim相关概念统一为numBlocks

非缺陷提交(命名重构)。将blockDim、cubeBlockDimN等统一重命名为numBlocks、cubeNumBlocksN。纯术语统一的机械替换。

---

### 422763a5 fix gmmsqV1

非缺陷提交(文档增强)。补充产品支持矩阵、术语优化、补充groupListType=count示例说明等。虽然commit message带fix但实际全为文档勘误。

---

### c2d880ba Fix SyncFunc EventId Type
- 根因类别：API返回值类型错误
- 涉及文件：mc2/common/inc/kernel/mc2_kernel_utils.h(新建), mc2下19个头文件
- 缺陷描述：`SyncFunc`中`FetchEventID(event)`返回`AscendC::TEventID`，但原代码用`static_cast<int32_t>`强转为`int32_t eventID`。这是类型不匹配问题——直接cast为int32_t可能导致信息丢失或与SetFlag/WaitFlag的参数类型不匹配。
- 修复模式：用`AscendC::TEventID eventID`接收返回值，去掉static_cast。同时将19个文件中重复的SyncFunc定义统一提取到mc2_kernel_utils.h。
- 可审查性：中
- 审查规则建议：对FetchEventID返回值检查是否用原生TEventID类型接收；检测static_cast将SDK专用类型转为基础类型的模式。

---

### bc5e43c1 涉及核间同步的算子必须设置schedule_mode为1,修复空tensor场景
- 根因类别：调度配置遗漏 + 空tensor判断条件错误
- 涉及文件：moe/moe_init_routing/op_host/moe_init_routing_tiling.cpp, moe/moe_token_permute_with_routing_map/下3个文件
- 缺陷描述：两个独立缺陷：(1) moe_init_routing和moe_token_permute_with_routing_map涉及核间同步但tiling阶段没有SetScheduleMode(1)独占全核，调度器可能将核分配给其他算子破坏同步语义。(2) 空tensor判断条件`if (routingMap->IsEmpty() || permuteTokensOut->IsEmpty())`误将output的IsEmpty也纳入判断，正确逻辑应只在输入routingMap为空时提前返回。同时缺少probs与routingMap的shape一致性校验。
- 修复模式：添加SetScheduleMode(1) + 收紧空tensor判断(去掉output的IsEmpty) + 增加shape校验。
- 可审查性：高
- 审查规则建议：所有使用核间同步的算子检查是否有SetScheduleMode(1)；空tensor提前返回条件不应混淆输入/输出。

---

### e9830dee dispatch、combine算子文档添加performanceInfoOptional

非缺陷提交(纯文档更新)。为dispatch/combine V4算子添加performanceInfoOptional参数文档说明。

---

### ada9091d 修复MoeTokenUnpermute
- 根因类别：workspace大小计算错误/硬编码
- 涉及文件：moe/moe_token_unpermute/op_host/moe_token_unpermute_tiling.cpp
- 缺陷描述：TilingMoeTokenUnpermute函数硬编码16MB workspace(sysWorkspaceSize = 16*1024*1024)，但后续TilingCompute内通过GetLibApiWorkSpaceSize()动态获取正确大小并覆盖。两处都调用GetWorkspaceSizes(1)写入workspaces[0]，前者硬编码被后者覆盖。若动态值不正确或代码顺序变化，会导致workspace分配不正确。
- 修复模式：删除冗余硬编码workspace设置，统一由动态计算决定。
- 可审查性：高
- 审查规则建议：检测workspace大小是否存在硬编码magic number；检查GetWorkspaceSizes是否被多次写入导致覆盖。

---

### c03c66cd dispatch&combine性能打点问题修复
- 根因类别：轮询逻辑缺陷导致重复计数
- 涉及文件：mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch_a2.h
- 缺陷描述：WaitDispatch函数的while循环轮询各rank通信完成标志时，对已完成的rank也会重复执行DataCopy+SyncFunc读取状态。原代码用DCCI(DataCacheCleanAndInvalid)清零已完成rank标志位来避免重复累计recvFlagNum，但远端内存清零方式不可靠。
- 修复模式：引入isVisited布尔数组，已完成的rank标记为true并continue跳过。用本地visited标记替代远端内存清零。
- 可审查性：中
- 审查规则建议：轮询完成标志的循环中检查是否有对已完成项的重复处理；用远端内存清零防止重复计数标记为高风险模式。

---

### 5202e30b 修复MoeTokenPermuteWithRoutingMapGrad

非缺陷提交(功能增强)。为MoeTokenPermuteWithRoutingMapGrad算子新增BF16+FLOAT混合精度支持。虽然commit message写"修复"但实际是扩展数据类型支持。

---

### e44028b1 fix deter casual GQA
- 根因类别：GQA维度缩放/offset计算
- 涉及文件：attention/flash_attention_score_grad/op_host/arch35/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_regbase.cpp
- 缺陷描述：CalcleCausalDeterParam()计算rUpper时未考虑GQA场景（g!=1）的修正量。当fBaseParams.g!=1时需额外加上(m+m1+1)*t1修正项rm3。缺少此修正导致GQA+causal+deterministic模式下tiling划分不正确。
- 修复模式：根据g!=1条件计算rm3修正项并加入rUpper累加。覆盖ell==0和ell!=0两个分支。
- 可审查性：中
- 审查规则建议：tiling参数计算涉及MHA/MQA/GQA多模式组合时，检查是否所有模式都被覆盖，特别是g(group数)相关条件分支。

---

### 8a64e103 【FAG】swizzle block num optimize

非缺陷提交(性能优化)。将swizzle连续块数量从固定值改为根据S1大小动态计算。

---

### b481f7c3 fix sfa race bug
- 根因类别：硬件流水线同步缺失(DMA后缺同步屏障)
- 涉及文件：attention/sparse_flash_attention/op_kernel/sparse_flash_attention_service_vector_mla.h
- 缺陷描述：MergeKv函数中DataCopyPad将UB数据通过MTE3通道拷贝到GM，该操作是异步的，但之后没有SetFlag/WaitFlag同步。后续代码可能在DMA完成前读写源缓冲区，产生数据竞争。
- 修复模式：在DataCopyPad后添加SetFlag<MTE3_S>和WaitFlag<MTE3_S>配对，使用新flag编号SYNC_INPUT_V0BUF_FLAG=6。
- 可审查性：高
- 审查规则建议：所有DataCopy/DataCopyPad后，若存在对源或目标缓冲区的复用/读取，必须确保有对应SetFlag/WaitFlag同步对。

---

### 4217f6d3 bugfix
- 根因类别：循环边界条件错误(变量名用错)
- 涉及文件：attention/lightning_indexer/op_kernel/lightning_indexer_kernel.h
- 缺陷描述：SplitCore函数内层for循环使用s2BaseNum作为上界，但实际应使用s2Loop。s2Loop在isSparseCountOver2K为true时被设为0或1，使用s2BaseNum意味着本应跳过的循环仍执行s2BaseNum次，导致访问无效数据。
- 修复模式：将`s2Idx < s2BaseNum`改为`s2Idx < s2Loop`。
- 可审查性：高
- 审查规则建议：当变量被赋值后在紧邻循环中未使用（而使用了名称相似的另一变量），应标记为可疑。

---

### 685562cc 修复aclnn storageShape导致的问题
- 根因类别：变量作用域错误 + 条件判断对象混淆
- 涉及文件：gmm/grouped_matmul_finalize_routing/op_host/op_api/aclnn_grouped_matmul_finalize_routing.cpp
- 缺陷描述：两个独立bug：(1) storageShape变量声明在if(INT32)分支内，SetStorageShape也只在该分支执行，但语义上所有路径都应设置storageShape。(2) 空指针校验条件`x2->GetDataType()==DT_INT4`错误地检查了源tensor x2而非目标tmpWeight的数据类型。
- 修复模式：变量声明提升作用域到if之前，SetStorageShape移到if之后；条件判断对象从x2改为tmpWeight。
- 可审查性：高
- 审查规则建议：检查SetStorageShape等属性设置是否被不必要地限制在条件分支内；条件判断中引用的tensor对象是否与语义一致(源vs目标混淆)。

---

### d2f14888 sycn dev code
- 根因类别：多类混合(边界检查缺失+变量未初始化+溢出防护缺失)
- 涉及文件：gmm/grouped_matmul/下2个文件, gmm/grouped_matmul_swiglu_quant_v2/下3个文件
- 缺陷描述：三个可识别缺陷：(1) groupNum通过static_cast<int32_t>强转赋值但缺少上界校验，修复新增groupNum>GMM_MAX_GROUP_LIST_SIZE(1024)越界检查。(2) AnalyzeAttrs函数缺少inputParams_.groupType=SPLIT_M初始化，tiling计算依赖groupType决定L1分块策略。(3) kernel函数新增浮点溢出模式控制SetCtrlSpr<FLOAT_OVERFLOW_MODE_CTRL>(0)，防止nan/inf。
- 修复模式：边界检查增强 + 变量初始化补全 + 数值稳定性防护。
- 可审查性：中(多种改动混在sync dev code中)
- 审查规则建议：外部输入维度/数量值必须有上下界校验；tiling参数结构体所有字段使用前必须显式初始化。

---

### 78272790 fix AttentionToFFN/FFNToAttention aclnn demo and cmake
- 根因类别：API误用 + 构建配置错误
- 涉及文件：mc2/attention_to_ffn/examples/test_aclnn_attention_to_ffn.cpp, mc2/attention_to_ffn/op_host/CMakeLists.txt, mc2/ffn_to_attention/op_host/CMakeLists.txt
- 缺陷描述：三类问题：(1) HcclCommInitAll被错误地放在for循环中多次调用，但该函数语义是一次性初始化所有communicator。(2) CMakeLists.txt中OPTYPE和ACLNNTYPE参数存在重复值。(3) FFN Worker等待超时10秒不够，增加到30秒。
- 修复模式：去除冗余循环 + CMake参数去重 + 超时值调整。
- 可审查性：高
- 审查规则建议：InitAll/InitGroup语义的集合初始化函数不应在循环中调用；CMake宏参数列表检查重复项。

---

### b58c6cbf aclnnGroupedMatmul多多多说明

非缺陷提交(纯文档更新)。新增关于多tensor场景下shape约束说明。

---

### 96e8ecff sfag算子修复精度问题，算子性能优化，放开k=2048的限制
- 根因类别：多核并发写冲突 + 维度参数错用 + 尾块精度逻辑错误(compound fix)
- 涉及文件：attention/sparse_flash_attention_grad/下10个文件(约800行变更)
- 缺陷描述：多层面精度/正确性问题：(1) cube2中V矩阵读取使用错误stride维度dimDv应为dimDTotal，且数据源引用了错误的GM地址。(2) cube4/cube5中matmul结果通过ScatterFixOut直接scatter写到dkWorkspaceGm/dvWorkspaceGm，多核并行写同一地址产生竞争。isFixOut原为false导致Fixpipe未执行累加。修复改为每core独立workspace+SetAtomicAdd原子累加。(3) CalSoftmax/CalSoftmaxGrad的actualSelS2在尾块未选场景下未正确更新，修复为每次根据isLastBasicBlock和lastBlockSize重新计算。(4) dK的scaleValue乘法从post阶段移到ScatterAdd阶段，修正计算顺序。
- 修复模式：每core独立workspace+原子累加替代直接scatter + 维度参数修正 + 尾块逻辑局部化。
- 可审查性：低(精度修复与性能优化深度耦合，跨10个文件约800行变更)
- 审查规则建议：多核并行写同一输出地址必须使用原子操作或独立workspace再归约；scatter/gather维度参数必须与实际数据layout严格对应。

## 批次13: 提交 #236-#255 (2026-01-21 ~ 2026-01-23)

---

### 33d0b7bccf 修复MoeFinalizeRouting算子问题

非缺陷提交(纯文档格式修复)。diff仅修改aclnnMoeFinalizeRouting.md，将markdown标题行末尾误带的``` ``` ```删除，不影响任何运行时行为。

---

### e670c5622 代码规范修改，拦截遗漏修改
- 根因类别：条件分支遗漏(新layout/参数未同步更新)
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_kernel_noquant_mla.h, attention/fused_infer_attention_score/op_host/arch35/fused_infer_attention_score_tiling_v2.cpp, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：三处独立缺陷：(1) MLA kernel中constInfo.isTNDOut未从inputParamsRegbase.isTNDOut赋值，TND输出layout标志始终为默认值；(2) fused_infer_attention_score中layout判断只处理"NTD"而遗漏"NTD_TND"变体，导致Lse shape校验缺失；(3) per-block quant参数校验中dequant scale的s维度除数硬编码为128U/256U，应使用变量fp8QBlockSize/fp8KVBlockSize，当实际block size不等于128/256时校验结果错误。
- 修复模式：补充遗漏赋值 + 扩展条件分支覆盖新layout + 硬编码常量替换为运行时变量
- 可审查性：中
- 审查规则建议：引入新layout类型时全局搜索所有layout判断分支确保新类型被覆盖；参数校验中与配置相关的数值(如blockSize)不应硬编码，应使用对应的运行时变量。

---

### 26daba7c3 gmm add check on 950PR and gmm add v1 support 950PR

非缺陷提交(新增平台支持+输入校验增强)。为950PR平台添加empty tensor校验、放开weight transpose限制、更新文档。属于新特性支持。

---

### dcc515a44 优化GroupedMatmul算子A8W4,A4W4

非缺陷提交(性能优化+新功能)。新增动态tiling函数优化核间负载均衡、L2Cache禁用优化、A4W4右矩阵转置支持等。均为性能调优和新功能扩展。

---

### 99755a1bb Layered Dispatch And Combine Bug Fix
- 根因类别：并发通信DMA参数错误 + flag残留误判
- 涉及文件：mc2/moe_distribute_combine/op_kernel/moe_distribute_combine_a2_layered.h, mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch_a2_layered.h
- 缺陷描述：两处缺陷：(1) Combine侧写flag时DataCopy的offset乘4、copy长度为4均不正确(flag实际类型uint64_t=8字节)，导致flag写入位置和大小与读取端不匹配，大BS场景下部分token的flag写到错误地址，接收端永远读不到到达标志造成token丢失。(2) Dispatch侧前后两次dispatch若属性不一致，UB中残留的旧flag值可能恰好等于SHOULD_SEND_FLAG_VALUE，导致接收端误判token已到达。
- 修复模式：改用DataCopyPad并使用sizeof(uint64_t)精确控制拷贝大小 + flag值中叠加magicVal_(时间轮)防止残留误判
- 可审查性：低(需理解RDMA多卡通信协议和分层MoE token传输机制)
- 审查规则建议：DataCopy/DataCopyPad的copy长度必须与实际数据类型sizeof一致；多轮通信中的flag/信号量必须包含轮次标识防止残留数据被误读。

---

### 1c635788c fix-debug combine layered
- 根因类别：计算逻辑错误(IPC内存可用大小多减了无关偏移量)
- 涉及文件：mc2/moe_distribute_combine/op_kernel/moe_distribute_combine_a2_layered.h
- 缺陷描述：GM2IPC函数中计算每张卡IPC slice可存储的最大token数时，原代码`maxBsInRankSizeOnIpc = (ipcSliceSize - IPC_DATA_OFFSET) / localMoeExpertNum_ / tokenSize`中IPC_DATA_OFFSET与当前分配逻辑无关不应被减去，导致可用空间被低估，大BS场景下可能提前触发空间不足或数据截断。
- 修复模式：移除错误的偏移量减法
- 可审查性：高
- 审查规则建议：内存可用空间计算中的每一个减法/偏移量都需要有明确的对应用途；审查时重点关注被减去的常量是否确实属于当前分配上下文。

---

### a54af6d2e bugfix: fix libopai undefined symbol
- 根因类别：构建配置缺陷(CMakeLists.txt参数缺失和路径错误)
- 涉及文件：attention/attention_worker_scheduler/op_host/CMakeLists.txt, ffn/ffn_worker_scheduler/CMakeLists.txt, ffn/ffn_worker_scheduler/op_host/CMakeLists.txt
- 缺陷描述：三处构建配置问题：(1)(3) add_modules_sources()缺少OPTYPE和ACLNNTYPE参数，模块源文件未被正确注册到构建系统，符号不会编入libopai.so；(2) file(GLOB RELATIVE . ...)的RELATIVE基准路径使用`.`而非`${CMAKE_CURRENT_SOURCE_DIR}`，out-of-source构建时路径解析错误导致子目录未被发现。最终libopai.so缺少相关符号，运行时dlopen失败。
- 修复模式：补全CMake宏参数 + 修正file(GLOB)的RELATIVE基准路径
- 可审查性：高
- 审查规则建议：所有add_modules_sources()调用必须包含OPTYPE和ACLNNTYPE参数；file(GLOB RELATIVE ...)的基准路径应始终使用${CMAKE_CURRENT_SOURCE_DIR}，禁止使用`.`。

---

### ad627275f NTD 叠加特性修复
- 根因类别：条件分支遗漏(NTD layout在多个代码路径中未被正确处理)
- 涉及文件：attention/common/op_kernel/arch35/flash_attention_score_block_vec_infer.h, attention/prompt_flash_attention/op_host/prompt_flash_attention_tiling_v2.cpp
- 缺陷描述：三处缺陷：(1) SoftmaxLseCopyOut中Lse输出stride的GQA特殊处理仅在LAYOUT_TND分支生效，遗漏LAYOUT_NTD，NTD layout下Lse的dstStride计算走错分支产生数据输出错位；(2) CheckIO中当queryShapeInfo.s==1(decode场景)时无条件启用enableIFAMask和enableIFA，但NTD layout不应走IFA路径；(3) PFATilingDataconvert中isGqa设置条件人为排除NTD但NTD下isGqa也应根据isIFA设置。
- 修复模式：在条件分支中正确纳入或排除NTD layout的特殊处理
- 可审查性：中
- 审查规则建议：新增layout类型时逐一审查所有layout相关条件分支(特别是IFA/GQA/stride计算等关键路径)；decode场景(s==1)的特殊优化路径需枚举验证所有layout组合的正确性。

---

### e13d1ec19 modify datacopy overflow
- 根因类别：整数类型错误/溢出(uint16_t截断)
- 涉及文件：mc2/common/inc/kernel/mc2_nd_to_nz.h
- 缺陷描述：CastBFtoFloat函数中cpInLen(size*sizeof(bfloat16_t))和cpOutLen(size*sizeof(float))均声明为uint16_t，uint16_t最大值65535，当size超过32767个bfloat16元素或16383个float元素时长度被截断，DataCopy只拷贝部分数据导致计算结果错误。DataCopyParams的blockLen字段本身也是uint16_t。
- 修复模式：将uint16_t改为uint32_t，同时将DataCopyParams/DataCopyPadParams替换为Ext版本支持uint32_t的blockLen
- 可审查性：高
- 审查规则建议：DataCopy的长度计算结果不应赋值给uint16_t变量；当数据量可能超过64KB时必须使用Ext版本的DataCopy参数结构体。

---

### bfef435e6 remove blocksize and layout constrain for no quant

非缺陷提交(约束放宽/新特性支持)。为非量化场景放宽blockSize和layout约束条件(blockSize对齐要求从128降到16，范围扩展到[16,1024])，属于有意扩展算子支持范围。

---

### 2df5e03c7 fix pse datacopy error due to large s2
- 根因类别：整数类型错误/溢出(uint16_t截断)
- 涉及文件：attention/common/op_kernel/arch35/pse.h
- 缺陷描述：当actualS2Len很大时，计算出的srcStride可能超过uint16_t最大值65535，但DataCopyParams.srcStride字段是uint16_t类型。溢出后静默截断导致数据拷贝地址错误产生错误结果。
- 修复模式：先将srcStride计算为int64_t中间变量，增加srcStride <= UINT16_MAX的溢出检查条件，超出范围时走DataCopyExtParams慢路径
- 可审查性：高
- 审查规则建议：当赋值给宽度较小的整数类型字段(如uint16_t)时，检查右侧表达式是否可能超出目标类型范围；所有DataCopyParams的stride/blockLen字段赋值前应有显式范围检查。

---

### e5dbe5b5c moe_finalize_routing_v2 DB分支问题修复
- 根因类别：双缓冲(Double Buffer)条件分支逻辑错误
- 涉及文件：moe/moe_finalize_routing_v2/op_kernel/moe_finalize_routing_v2_fp_cuth_k2.h, moe/moe_finalize_routing_v2/op_kernel/moe_finalize_routing_v2_fp_cuth_k4.h
- 缺陷描述：两个问题：(1) k2版本中ISBIASEXIST=false时else分支执行Adds(x, x, 0, dataLen)，无意义的向量操作可能导致双缓冲DB0/DB1间数据竞争或pipeline同步问题；(2) k4版本中bias的Add操作被错误放在第二个DB分支统一处理DB0-DB3，但各DB的invalid row判断是独立的。当DB0有效但DB1无效(被Duplicate为0)时，DB0的bias加法被跳过。
- 修复模式：k2中删除无意义的Adds(x, x, 0) + k4中将bias Add拆分到各自DB的else分支中独立执行
- 可审查性：中
- 审查规则建议：双缓冲场景中每个buffer的计算操作应在其对应的条件分支中独立完成，不应合并不同buffer的操作；检查Adds(x, x, 0)或Muls(x, x, 1)这类no-op向量操作是否为无意义代码。

---

### 07dfd9c3e GQA per-block全量化支持NTD_TND

非缺陷提交(新功能)。为GQA per-block全量化增加NTD/TND layout支持，属于功能扩展。

---

### 3c413a90f bugfix: empty tensor tilingdata reset
- 根因类别：tiling数据结构不一致(host/kernel结构体不匹配)
- 涉及文件：attention/flash_attention_score/op_host/flash_attention_score_tiling.cpp, attention/flash_attention_score/op_kernel/arch32/flash_attention_score_template_tiling_key.h, attention/flash_attention_score/op_kernel/arch32/flash_attention_score_tiling.h
- 缺陷描述：empty tensor场景下host侧使用FlashAttentionScoreEmptyInputTilingData结构体(独立小结构)，kernel侧使用FlashAttentionScoreGeneralTilingData，两者内存布局不一致——host侧只填充少数字段，kernel侧按完整大结构体解析，读到未初始化垃圾值引发aicore error。
- 修复模式：删除独立的EmptyInputTiling类，改用完整TilingData结构体+reset()方法归零所有字段，确保host/kernel一致
- 可审查性：高
- 审查规则建议：tiling host侧写入的结构体类型必须与kernel侧读取的结构体类型严格一致；通过context->GetTilingData<T>()获取的类型T应与kernel侧声明的tiling struct匹配。

---

### 82013a396 sink tiling bug
- 根因类别：结构体内存布局/字段偏移错误
- 涉及文件：attention/flash_attention_score/op_kernel/arch32/flash_attention_score_tiling.h, attention/flash_attention_score_grad/op_kernel/arch32/flash_attention_score_grad_tiling.h
- 缺陷描述：InputParams结构体中needSinkOp字段位置不当，导致host侧写入和kernel侧读取的偏移量不匹配。梯度tiling结构体中padding数组大小错误(7应为3)，多余padding导致后续字段偏移错位。
- 修复模式：调整needSinkOp字段位置+修正padding数组大小，使内存布局与host侧一致
- 可审查性：高
- 审查规则建议：tiling结构体字段顺序和padding必须在host和kernel两侧严格对应；结构体添加新字段后应验证总大小和每个字段的offset是否与另一侧一致；建议使用static_assert或offsetof校验关键字段偏移。

---

### 70898673b FAG sink精度修复
- 根因类别：workspace偏移错误+初始化错误+DataCopyPad padding硬编码+索引计算错误+padding区域脏数据(compound fix)
- 涉及文件：attention/flash_attention_score_grad/op_host/arch32/flash_attention_score_grad_tiling_s1s2_bn2gs1s2_sab.cpp, attention/flash_attention_score_grad/op_kernel/arch32/下3个文件
- 缺陷描述：FAG Sink功能存在多个互相关联的缺陷：(1) dsinksum的workspace分配和偏移计算放在错误位置，offset与实际分配不匹配；(2) InitOutput初始化dsinksum时使用错误base地址且缺少核类型/核索引条件判断；(3) DataCopyPad的isPad和rightPadding硬编码，数据已8对齐时硬补8个float导致读取到workspace外脏数据；(4) dsinksum写入偏移计算公式错误；(5) Mul运算前未清零padding区域引入脏数据。
- 修复模式：重新组织workspace分配顺序 + 修正地址和条件 + 动态决定padding + 修正偏移公式 + padding区域清零
- 可审查性：低(涉及多个文件、多处关联修改、复杂的workspace内存布局)
- 审查规则建议：workspace的分配和使用必须严格按照相同顺序和条件；DataCopyPad的isPad和rightPadding应根据实际数据大小动态计算避免硬编码；向量运算前应确保padding区域已初始化为0。

---

### 03909dce5 Fix FAG Sink MM12 Nzout Bug
- 根因类别：操作时序错误(执行顺序/数据依赖)
- 涉及文件：attention/flash_attention_score_grad/op_kernel/arch32/flash_attention_score_grad_s1s2_bn2gs1s2_sab.h
- 缺陷描述：SubGrapB函数中将vecCopyOutBuffer拷贝到GM的DataCopyPad操作被放在Sink处理逻辑之前，但Sink逻辑会修改相关中间数据，在Nzout格式下数据拷贝和Sink计算产生竞争导致输出结果不正确。
- 修复模式：将DataCopyPad到GM的代码块整体移到Sink处理逻辑之后执行
- 可审查性：中
- 审查规则建议：在硬件pipeline架构下数据输出到GM的操作应放在所有依赖同一buffer的向量计算之后；代码块移动时应验证所有event信号的set/wait配对是否正确。

---

### 5f8f4d5fa fix gmmsqV1 ut
- 根因类别：UT测试数据类型错误
- 涉及文件：gmm/grouped_matmul_swiglu_quant/tests/ut/op_host/op_api/test_aclnn_grouped_matmul_swiglu_quant.cpp, gmm/grouped_matmul_swiglu_quant/tests/ut/op_host/op_api/test_aclnn_grouped_matmul_swiglu_quant_weight_nz.cpp
- 缺陷描述：UT中weightScale的tensor类型声明为ACL_INT64，但实际weight scale应为ACL_FLOAT类型，导致UT使用错误数据类型构造测试输入。
- 修复模式：将weightScale的TensorDesc类型从ACL_INT64改为ACL_FLOAT
- 可审查性：高
- 审查规则建议：UT中tensor的数据类型应与算子规格/接口文档中定义的类型一致；检查scale类tensor是否使用了浮点类型。

---

### ab7c48a6c fix scatter_pa_kv_cache oom aic
- 根因类别：DataCopy对齐要求导致越界读取(OOM/AIC异常)
- 涉及文件：attention/scatter_pa_kv_cache/op_kernel/arch35/下4个文件
- 缺陷描述：原代码DataCopy(dst, src, RoundUp(size))将size向上对齐到32B边界，但当源GM地址处的实际有效数据长度不足对齐后的长度时，DataCopy从GM读取超出有效数据范围的内存触发OOM或AIC异常。headSize不是32B对齐的倍数时必然发生。
- 修复模式：将所有DataCopy替换为DataCopyPad，使用DataCopyExtParams指定精确blockLen，通过DataCopyPadExtParams配置isPad=0避免越界读取
- 可审查性：高
- 审查规则建议：当GM数据的实际长度可能不满足DataCopy的对齐要求时应使用DataCopyPad替代DataCopy；检查所有使用RoundUp作为DataCopy长度参数的调用点。

---

### a248cefc1 fix CheckFeatureLse function

非缺陷提交(空提交)。该commit的tree与parent的tree完全一致，无任何文件变更。是merge产生的空commit。

---

## 批次14: 提交 #256-#275 (2026-01-15 ~ 2026-01-21)

---

### d0ede159b fix v3 infershape
- 根因类别：API误用/接口参数错误(GetInputShape vs GetOptionalInputShape)
- 涉及文件：moe/moe_init_routing_v3/op_host/moe_init_routing_v3_infershape.cpp
- 缺陷描述：对optional的scale和offset输入使用了GetInputShape()而非GetOptionalInputShape()。GetInputShape()在输入为空时返回异常结果或触发错误，而这两个输入在设计上允许为nullptr。
- 修复模式：将GetInputShape替换为GetOptionalInputShape
- 可审查性：高
- 审查规则建议：检测所有GetInputShape调用，对照算子定义文件中标记为optional的输入，确认是否应使用GetOptionalInputShape。

---

### ef22ac8f1 修复cust example头文件引用优先级问题
- 根因类别：构建配置缺陷(头文件include路径顺序)
- 涉及文件：build.sh
- 缺陷描述：g++编译命令中-I ${INCLUDE_PATH}在-I ${CUST_INCLUDE_PATH}之前，导致自定义算子的头文件被系统内置头文件覆盖。当cust算子与内置算子头文件同名时，编译器优先找到内置版本，产生链接或行为错误。
- 修复模式：交换两个-I参数的顺序，将${CUST_INCLUDE_PATH}放在${INCLUDE_PATH}前面
- 可审查性：高
- 审查规则建议：检测构建脚本中-I参数的顺序，cust/vendor路径应优先于系统路径。

---

### a58c8a0b9 FA算子B模板同步问题bugfix
- 根因类别：硬件流水线同步缺失(event信号使用错误导致数据竞争)
- 涉及文件：attention/flash_attention_score/op_kernel/arch32/flash_attention_score_bn2gs1s2_b.h
- 缺陷描述：FlashAttention算子B模板的hasSink场景中，原代码额外分配了独立的eventIdVToMte2Sink同步事件。hasSink时第一个if触发了eventIdVToMte2A的SetFlag，第二个if又触发了eventIdVToMte2Sink的SetFlag，但对应的WaitFlag只等了eventIdVToMte2Sink，没有消费eventIdVToMte2A，导致event信号积压引起死锁或数据竞争。
- 修复模式：删除冗余的event变量，统一使用eventIdVToMte2A；修正条件分支为互斥(!hasSink vs hasSink)
- 可审查性：低(需深入理解Ascend硬件event同步机制)
- 审查规则建议：检测同一代码块中分配多个相同类型HardEvent的情况(如多次AllocEventID<HardEvent::V_MTE2>)，可能暗示同步逻辑冗余或错误。

---

### 25c72d064 Fix RecurrentGatedDeltaRule UT

非缺陷提交(UT测试修复)。修复UT断言逻辑中||运算符优先级导致条件恒真的问题，以及CMakeLists中tiling文件路径引用。属于测试代码自身缺陷，不影响产品功能代码。

---

### e5aced81a RainFusion support BNSD kvcache nullptr && TND check

非缺陷提交(功能增强)。为RainFusionAttention算子新增BNSD格式下kvcache可传nullptr的支持能力，新增TND格式seqlen总和校验，是完整的特性开发而非缺陷修复。

---

### b48e75b7c AllReduceMM/MMReduceScatter_fix
- 根因类别：条件分支遗漏/缺陷(模板分支未覆盖所有参数组合导致ND/NZ格式混淆)
- 涉及文件：mc2/all_gather_matmul/op_kernel/all_gather_matmul.cpp, mc2/matmul_reduce_scatter/op_kernel/matmul_reduce_scatter.cpp
- 缺陷描述：原代码只有BIAS_CAST的true/false两个分支，两个分支内bType都固定使用NZ格式。当模板参数ND2NZ_OPT为false时(B矩阵本身是ND格式)，bType仍被设为NZ格式，导致Matmul引擎按NZ布局读取ND数据，产生静默的数值错误。同时使用普通if而非if constexpr，编译期无法剪枝。
- 修复模式：用if constexpr展开IsFullMesh x IsNd2Nz x IsBias的全部4种组合，每种组合正确设置bType(NZ vs ND)
- 可审查性：中
- 审查规则建议：检测模板函数中使用普通if分发模板布尔参数的代码；检测模板参数存在但分支内对应类型不随之变化的情况。

---

### b5023d8fe 拦截sink场景下QKV float32入参的情况
- 根因类别：输入校验缺失(sink场景下不支持float32但未做拦截)
- 涉及文件：attention/flash_attention_score_grad/op_api/aclnn_flash_attention_score_grad.cpp, 对应docs/
- 缺陷描述：FlashAttentionScoreGrad在sink场景(sinkInOptional != nullptr)下不支持float32数据类型的QKV输入，但aclnn层未做校验拦截，用户传入float32+sink时产生未定义的计算结果或崩溃。文档也错误地将FLOAT32列为支持类型。
- 修复模式：aclnn层新增当sinkInOptional非空且query dtype为DT_FLOAT时返回ACLNN_ERR_PARAM_INVALID；同步更新文档移除sink场景下FLOAT32类型声明
- 可审查性：高
- 审查规则建议：对每个aclnn算子的数据类型校验逻辑，检查是否覆盖了所有可选参数(optional tensor)非空时的类型约束；文档中声明的支持类型列表应与代码校验逻辑一致。

---

### c79715514 nsa compress with cache 校验修复
- 根因类别：输入校验缺失(空tensor和非ND格式未做拦截)
- 涉及文件：attention/nsa_compress_with_cache/op_host/op_api/aclnn_nsa_compress_with_cache.cpp, 对应docs/
- 缺陷描述：NsaCompressWithCache算子在aclnn层缺少两项关键校验：(1) input/weight/outputCache为空tensor(ShapeSize为0)时未拦截，会导致后续tiling或kernel阶段异常；(2) 输入tensor为非ND格式时未拦截，算子仅支持ND格式。文档中也有"使用该功能可传入nullptr"应为"不使用"的笔误。
- 修复模式：新增CheckIsEmptyTensor和CheckNDFormat校验函数，在GetWorkspaceSize入口处拦截；同步修正文档
- 可审查性：高
- 审查规则建议：对所有aclnn算子的GetWorkspaceSize入口，检查是否存在空tensor校验和格式校验。

---

### f736fef3f modify gsAxis performance debug

非缺陷提交(性能优化)。将循环内重复计算的表达式提取为局部变量，纯公共子表达式提取(CSE)，功能语义不变。

---

### 312d30359 Post tilingdata 结构体64位对齐
- 根因类别：赋值遗漏/结构体字段遗漏(tiling结构体未满足硬件64位对齐要求)
- 涉及文件：attention/flash_attention_score_grad/op_kernel/arch32/flash_attention_score_grad_tiling.h
- 缺陷描述：tiling数据结构中最后一个字段baseMN是uint32_t(4字节)，导致结构体总大小不是8字节整数倍。Ascend芯片DMA引擎要求传输数据大小8字节对齐，未对齐时DMA多搬或少搬字节，导致读取未初始化内存数据或覆盖相邻内存。
- 修复模式：在baseMN字段后添加4字节padding数组uint8_t PostParamsPH[4] = {}使结构体满足8字节对齐
- 可审查性：高
- 审查规则建议：对所有tiling数据结构体，静态检查sizeof是否为8的倍数；可编写编译期static_assert(sizeof(TilingData) % 8 == 0)。

---

### c3d301162 添加unlikely，减少sink分支可能带来的性能影响

非缺陷提交(性能优化)。将sink分支条件用unlikely()包裹，告诉编译器该分支大概率为false以优化分支预测。功能正确性不受影响。

---

### 23d4b8e93 select_attention_operators

非缺陷提交(新特性开发)。引入基于Quest论文的block-sparse mask predictor完整实现，约100个新增文件，不修改已有代码。

---

### 10f8a80b7 fix gather_pa_kv_cache ut

非缺陷提交(测试代码重构)。将UT从旧框架迁移到新框架，不涉及产品代码修改。

---

### 9be387c56 fix fia aclnn sample for opensource master

非缺陷提交(文档示例代码修正)。修改4个md文件中的示例代码(attenMask shape、变量名规范化)，不影响任何编译产出。

---

### edd2099ff 补充过滤脚本遗漏的内容

非缺陷提交(文档补充与产品名称整改)。约100个md文件变更，纯文档内容补齐和名称规范化。

---

### 7da5e5332 同步grouped_matmul_add代码

非缺陷提交(示例代码质量改进)。将示例代码的裸指针手动释放改为RAII智能指针模式，不涉及产品代码。

---

### 3639e55e7 fix moe_token_permute_with_routing_map example
- 根因类别：UT测试数据类型错误(示例代码CreateAclTensor参数与宿主数据类型不匹配)
- 涉及文件：moe/moe_token_permute_with_routing_map/examples/test_aclnn_moe_token_permute_with_routing_map.cpp
- 缺陷描述：三处数据类型不匹配：(1) indicesHostData为std::vector<int>(4字节)但CreateAclTensor指定ACL_BOOL(1字节)，内存布局完全不匹配；(2) x的宿主数据为std::vector<float>但创建tensor时指定ACL_BF16，float32与bfloat16内存表示不同导致数据截断；(3) expandedXOut同样从ACL_BF16改为ACL_FLOAT。
- 修复模式：统一宿主数据容器类型与aclDataType枚举值的语义映射
- 可审查性：高
- 审查规则建议：对所有CreateAclTensor调用，静态检查vector模板类型与aclDataType参数的一致性。

---

### c65fdcacb fix opapi ut bugs
- 根因类别：构建配置缺陷(测试退出码被后续清理命令覆盖 + 多芯片打桩不完整)
- 涉及文件：tests/ut/framework_normal/op_api/CMakeLists.txt, tests/ut/framework_normal/op_api/scripts/clean_opapi_stub.py, tests/ut/framework_normal/op_api/scripts/generate_opapi_stub.py
- 缺陷描述：两个独立缺陷：(1) CMake中bash -c用分号连接UT执行和清理脚本，整个bash -c退出码取决于最后一条命令(清理脚本)，当UT失败但清理成功时CMake接收到退出码0误认为UT通过，导致CI无法拦截UT失败；(2) generate_opapi_stub.py仅为当前芯片生成打桩，缺少其他芯片(ascend910b/ascend910_93/ascend910_95)路径下的符号链接。
- 修复模式：(1) 保存UT退出码并在最后返回: OP_API_UT_EXE_RET=$?; cleanup; exit $OP_API_UT_EXE_RET; (2) generate脚本为非当前芯片创建符号链接，clean脚本清理这些链接
- 可审查性：高
- 审查规则建议：CMake add_custom_command的bash -c中含测试执行后接清理命令时，检查是否正确保存并返回了测试退出码。

---

### 90cc4df64 Bugfix: A2 dispatch&combine add SetBlockDim for gentask
- 根因类别：赋值遗漏/结构体字段遗漏(aicpu task缺少SetBlockDim配置)
- 涉及文件：mc2/common/src/mc2_moe_gen_task_ops_utils.cpp
- 缺陷描述：A2平台上MoeDistributeDispatch/Combine及V2版本四种算子的gentask流程中，创建aicpu task时没有调用SetBlockDim设置block维度。缺少此设置导致aicpu任务使用默认配置，在A2平台上造成任务调度异常或性能严重劣化。462行改动中绝大部分是格式重排，实质性改动仅为新增NEED_SET_BLOCK_SET集合和SetBlockDim调用。
- 修复模式：在aicpu task创建后对特定算子类型调用SetBlockDim(默认6，可通过节点属性覆盖)
- 可审查性：中(大量格式重排掩盖真正逻辑改动)
- 审查规则建议：对创建aicpu task的代码路径检查是否根据平台/算子类型正确配置了blockDim；格式化和功能修复应拆分为独立提交。

---

### fbade778b fix experimental depends
- 根因类别：构建配置缺陷(ENABLE_EXPERIMENTAL模式下CMake路径拼接缺少前缀)
- 涉及文件：cmake/func.cmake
- 缺陷描述：两个函数受影响：(1) op_add_depend_directory中EXISTS检查使用不带experimental/前缀的路径，导致依赖存在但检查返回false被跳过；(2) add_bin_compile_target中源码拷贝路径同样缺少前缀。后果是开启ENABLE_EXPERIMENTAL编译时算子间依赖关系无法正确解析，构建失败。关联Issue #427。
- 修复模式：引入depend_info_update变量，在ENABLE_EXPERIMENTAL时统一加experimental/前缀
- 可审查性：高
- 审查规则建议：当编译模式开关影响源码目录结构时，静态检查所有file(EXISTS)、add_subdirectory、路径拼接是否都使用了带开关前缀的路径变量。

## 批次15: 提交 #276-#295 (2025-12-31 ~ 2026-01-14)

---

### c0224370 fix: opensource, fix mlaprolog A5 example
非缺陷提交(示例代码整理)。删除mla_prolog旧版example文件，替换为mla_prolog_v3新示例。示例/文档维护，不涉及production代码逻辑。

---

### 70e5be5f 修复UT批跑编译失败问题
- 根因类别：构建配置缺陷(CMake脚本缺少文件存在性检查)
- 涉及文件：cmake/custom_build.cmake, cmake/func.cmake
- 缺陷描述：执行`bash build.sh -u`批跑UT时，CMake脚本对每个算子目录无条件执行`add_subdirectory(${OP_UT_LIST}/tests)`和`file(READ "${OP_DIR}/tests/CMakeLists.txt" ...)`。当某些算子目录下不存在tests/CMakeLists.txt时，CMake直接报错导致整个编译流程失败。
- 修复模式：在3处关键位置添加`if (EXISTS "${...}/tests/CMakeLists.txt")`守卫检查，不存在时continue()跳过。
- 可审查性：高
- 审查规则建议：CMake中对add_subdirectory或file(READ)调用前，检查是否有路径存在性校验。

---

### b0bf9fa2 fix experimental depend bug
- 根因类别：构建配置缺陷(experimental模式下依赖路径硬编码错误)
- 涉及文件：cmake/func.cmake
- 缺陷描述：op_add_depend_directory函数中，ENABLE_EXPERIMENTAL开启时算子依赖目录应拼接为`${CMAKE_CURRENT_SOURCE_DIR}/experimental/${depend_info}`，但原代码始终使用`${CMAKE_CURRENT_SOURCE_DIR}/${depend_info}`，导致experimental模式下找不到依赖目录。
- 修复模式：添加if(ENABLE_EXPERIMENTAL)分支，根据编译模式选择不同路径前缀。
- 可审查性：高
- 审查规则建议：检查CMake中引用ENABLE_EXPERIMENTAL变量的代码块，是否在路径拼接逻辑中正确区分了experimental和非experimental路径。

---

### 611e9f1f fix spell problem for splitFuse at openSource code
非缺陷提交(标识符拼写修正)。将命名空间FaiKenel全局重命名为FaiKernel，所有引用处一致修正，不改变运行时行为。

---

### 7fb813a9 dispatch combine v4 demo fix
- 根因类别：API误用/接口参数错误(文档示例代码与API签名不匹配)
- 涉及文件：mc2/moe_distribute_combine_v2/docs/aclnnMoeDistributeCombineV4.md, mc2/moe_distribute_dispatch_v2/docs/aclnnMoeDistributeDispatchV4.md
- 缺陷描述：V4版本API新增performanceInfoOptional参数，但文档示例代码未传递该参数，同时include头文件名大小写错误。用户复制示例代码编译直接失败。
- 修复模式：补充nullptr参数，修正include头文件名大小写。
- 可审查性：中
- 审查规则建议：API签名变更时自动检测docs/examples目录下的示例代码是否同步更新了函数调用参数。

---

### c3c78777 修改version info
非缺陷提交(版本依赖配置变更)。修改version.info中运行时依赖包列表，正常迭代调整。

---

### 8d72712c fix ring_attention_update ophost ut
- 根因类别：UT测试与production代码不同步
- 涉及文件：attention/ring_attention_update/tests/ut/op_host/test_ring_attention_update_infershape.cpp, attention/ring_attention_update/tests/ut/op_host/test_ring_attention_update_tiling.cpp
- 缺陷描述：两处问题：(1) infershape UT的include头文件名从infershape_context_faker.h更名为infer_shape_context_faker.h，UT未同步导致编译失败；(2) tiling UT中命名空间从Ops::Math::AnyValue改为Ops::Transformer::AnyValue，且期望的tiling输出末尾缺少字段(production代码已新增tiling字段，UT断言未同步)。
- 修复模式：更新include路径、命名空间引用和期望输出字符串。
- 可审查性：中
- 审查规则建议：tiling数据结构新增字段时，检查UT断言是否覆盖；production代码重构后UT引用路径是否同步更新。

---

### 7ee592d1 fix: opensource, fix soc default value
非缺陷提交(文档修正)。补充soc_version参数默认值说明。

---

### 8a72fdf2 关闭cmake默认编译选项；安装校验失败ERROR级别改为WARNING级别
- 根因类别：构建/安装配置缺陷
- 涉及文件：CMakeLists.txt, scripts/package/common/sh/version_compatiable.inc
- 缺陷描述：两个独立问题：(1) CMake Release模式默认注入-O3 -DNDEBUG等选项，与项目自身编译配置冲突；(2) 安装脚本version_compatiable.inc中版本兼容性检查失败时ERROR+return 1导致安装中断，在某些兼容场景下过于严格。
- 修复模式：清空CMake默认Release编译选项；将校验失败从ERROR降为WARNING并移除return 1。
- 可审查性：中
- 审查规则建议：安装脚本中将校验失败从阻断降为警告时，需确认不会导致不兼容环境下的运行时问题。

---

### a740c5c8 fix: opensource, fix fia/ifa warnings
非缺陷提交(编译警告清理)。删除未使用的constexpr变量、删除不存在soc型号的AddConfig注册、重命名参数消除shadowing警告。

---

### aac27c3e fix:add conditional constraints in tnd module
- 根因类别：条件分支遗漏/缺陷(layout模式条件守卫缺失)
- 涉及文件：attention/nsa_selected_attention_infer/op_kernel/nsa_selected_attention_infer.h
- 缺陷描述：curTotalQSeqLenOffset的累加循环在所有layout模式下都会执行，但该偏移量计算仅对TND layout有意义。非TND模式下执行该循环会读取无意义的actualQSeqLengthsGm数据，导致偏移量计算错误，进而造成数据访问异常。
- 修复模式：用`if constexpr (LAYOUT_T == LAYOUT::TND)`包裹偏移量累加循环，编译期排除不适用路径。
- 可审查性：中
- 审查规则建议：当代码存在模板参数区分多种数据布局时，检查所有布局相关偏移计算是否有对应的条件守卫。

---

### 83be205f fix rt_kb
- 根因类别：安装/打包配置缺陷(覆盖安装误删其他组件文件)
- 涉及文件：scripts/package/module/ascend/OpsTransformer.xml
- 缺陷描述：transformer包覆盖安装时，rtkb相关目录的安装配置缺少install_type="all"和正确的install_mod权限设置，且使用了entity="true"属性，导致覆盖安装时知识库目录下其他仓库的文件被误删。
- 修复模式：添加install_type="all"，移除entity="true"，设置正确权限(550/555)。
- 可审查性：低
- 审查规则建议：打包配置中带entity="true"的目录项需确认覆盖安装场景下不会清除其他组件文件。

---

### 34a2a609 fix memory leak
非缺陷提交(空提交)。commit tree与parent完全相同，本仓库无实际文件变更。实际修改可能在关联dev子仓库中。

---

### d8681ad1 fix AttentionToFFN Addr
- 根因类别：GM地址来源/偏移错误(误用remote指针获取local地址)
- 涉及文件：mc2/attention_to_ffn/op_kernel/attention_to_ffn.h
- 缺陷描述：获取本rank地址时，原代码通过winContext_->userMemType判断走两条路径(remoteRes[rankId_].nextDevicePtr或userMemRes[rankId_].addr)，某些场景下取到的地址不正确。正确做法是直接使用winContext_->localWindowsIn获取本rank本地窗口输入地址。
- 修复模式：将条件分支简化为直接取localWindowsIn，消除类型判断导致的地址获取错误。
- 可审查性：高
- 审查规则建议：获取本rank地址时检查是否误用了remote资源指针代替local资源指针；条件分支中获取同一语义资源的不同路径应确保语义一致性。

---

### ab0dceb8 gpt-oss sink doc modify
非缺陷提交(纯文档修改)。在API文档中新增learnableSinkOptional输入参数说明。

---

### a911707e bugfix: ffn example support fp16
- 根因类别：UT测试数据类型错误(host数据类型与aclDataType不匹配)
- 涉及文件：ffn/ffn/docs/aclnnFFN.md, ffn/ffn/docs/aclnnFFNV2.md, ffn/ffn/docs/aclnnFFNV3.md
- 缺陷描述：FFN示例代码声明aclDataType为ACL_FLOAT16，但host端数据容器使用std::vector<float>(4字节)，CreateAclTensor中用sizeof(T)即sizeof(float)=4计算device内存大小而非实际fp16的2字节。导致aclrtMalloc分配2倍内存、aclrtMemcpy拷贝多余数据、结果读取时数据错误。
- 修复模式：将host容器从std::vector<float>改为std::vector<op::fp16_t>；内存大小计算从sizeof(T)改为aclDataTypeSize(dataType)。
- 可审查性：高
- 审查规则建议：CreateAclTensor的dataType参数与模板类型T的sizeof不一致时应报警；示例代码host数据类型应与aclDataType声明严格对应。

---

### b9e99acc fix: opensource, fix pfa warnings
非缺陷提交(编译警告修复)。变量重命名消除shadowing警告，不改变运行时行为。

---

### 181d25f5 fix issue 修复genop生成目录问题
- 根因类别：构建配置缺陷(代码生成工具链不完整)
- 涉及文件：scripts/opgen/opgen_standalone.py, cmake/custom_build.cmake, 及多个template文件
- 缺陷描述：genop脚本创建新算子工程时，若算子分类(op_class)是全新的，脚本只复制算子模板目录但不会生成上层CMakeLists.txt，导致新创建的算子工程无法被CMake发现和编译。此外模板代码本身是空壳无法编译运行。
- 修复模式：在_copy_template方法中增加逻辑——若目标目录上级不存在CMakeLists.txt则从模板复制一份；新增CMakeLists.txt模板文件；补全模板算子代码使其可编译。
- 可审查性：中
- 审查规则建议：代码生成工具在创建新目录结构时，应检查构建系统所需的所有入口文件是否已存在或需同步创建。

---

### 6b029d5f 修复GM偏移量的数据类型为int64
- 根因类别：整数类型错误/溢出(GM偏移用uint32导致大数据量溢出)
- 涉及文件：moe/moe_token_unpermute_with_routing_map/op_kernel/masked_select.h
- 缺陷描述：GM偏移量ind使用uint32_t类型，计算公式为progress*tileLength。当两者都较大时乘积超过uint32最大值(~42亿)，发生整数溢出导致GM地址偏移错误，访问错误内存位置。
- 修复模式：将ind类型从uint32_t改为uint64_t，CopyInData和CopyInMask两处均修改。
- 可审查性：高
- 审查规则建议：GM地址偏移计算变量不应使用32位整型；当偏移量由两个变量相乘得到时，结果类型应足够宽以避免溢出。

---

### 84ca56e9 修复moe_init_routing_v3/moe_re_routing子包编译问题
- 根因类别：API误用(CeilDiv与CeilAlign语义混淆)
- 涉及文件：moe/moe_re_routing/op_host/moe_re_routing_r_tiling.cpp, moe/moe_re_routing/op_host/moe_re_routing_re_tiling.cpp, moe/moe_re_routing/op_kernel/arch35/moe_re_routing_r_regbase.h, moe/moe_re_routing/op_kernel/arch35/moe_re_routing_re_regbase.h, moe/moe_re_routing/tests/ut/op_host/test_moe_re_routing_tiling.cpp
- 缺陷描述：多处混淆CeilDiv和CeilAlign语义。CeilDiv(a,b)=ceil(a/b)是向上取整除法，CeilAlign(a,b)=ceil(a/b)*b是向上对齐。在计算UB buffer大小、InitBuffer的size参数、shift size等需要对齐到block边界的场景中，错误使用CeilDiv(得到block数)而非CeilAlign(得到对齐后字节数)，导致分配buffer远小于实际需要，引发越界或数据错误。UT期望值从4063/4064改为130016/130048反映了修复后buffer大小的数量级变化。
- 修复模式：区分需要"对齐"和"除法"的场景，将应该对齐的地方从CeilDiv改为CeilAlign。
- 可审查性：高
- 审查规则建议：CeilDiv和CeilAlign调用需审查上下文语义——赋给buffer size的应用CeilAlign(对齐)，赋给loop count的应用CeilDiv(除法)；当buffer大小从tiling值直接传入InitBuffer时确认单位一致(字节vs block数)。

## 批次16 (#296-#314)

### 66b12bf2 fix cv1:1 sk
- 根因类别：条件分支遗漏/缺陷 + constexpr误用
- 涉及文件：attention/mla_prolog/op_host/mla_prolog_tiling.cpp, attention/mla_prolog/op_kernel/kernel_mla_prolog_split_n.h, attention/mla_prolog/op_kernel/service_dynamic_quant_qn_mul_qr.h
- 缺陷描述：MlaProlog算子CV1:1模式多个bug：(1)tiling校验逻辑仅在非EMPTY_QUERY路径执行，EMPTY_QUERY路径遗漏校验，且用黑名单方式（列举不支持的cacheMode），修复改为白名单。(2)cvRatio_是static constexpr编译态常量跟随模板参数，但CV1:1编译的kernel运行时可能跑在CV1:2硬件上。修复改为运行时变量通过GetSubBlockNum()获取。(3)两处if constexpr改为if运行时判断。(4)DynamicQuantQnWithMulQr的cvRatio从模板参数改为函数参数。
- 修复模式：编译期常量改运行时变量，增加运行时硬件配置检测；校验提前到公共路径并改用白名单
- 可审查性：中
- 审查规则建议：硬件配置参数（cvRatio等）应优先使用运行时查询而非constexpr；当编译态和运行态可能不一致时，检查所有constexpr/模板参数是否在所有运行场景下恒定

### d2ae9c6b all_gather_add编译修复
- 根因类别：构建配置缺陷 + 资源释放顺序错误
- 涉及文件：cmake/custom_build.cmake, examples/mc2/all_gather_add/op_host/op_api/aclnn_all_gather_add.h, examples/mc2/all_gather_add/examples/test_aclnn_all_gather_add.cpp
- 缺陷描述：(1)CMake构建系统缺少all_gather_add的add_subdirectory入口；(2)include路径缺少aclnnop/子目录前缀致编译找不到头文件；(3)HcclCommDestroy在main函数中aclrtResetDevice之后才调用，此时aclBinaryUnload找不到free内存（use-after-free类）。
- 修复模式：CMake增加构建入口；修正include路径；将HcclCommDestroy移到线程内device资源释放前执行
- 可审查性：高
- 审查规则建议：HCCL通信资源销毁必须在aclrtResetDevice之前；资源释放遵循LIFO顺序；新增算子必须确保CMake入口完整

### d27eab56 修复experimental下的attention目录
- 根因类别：构建配置缺陷
- 涉及文件：cmake/custom_build.cmake, cmake/func.cmake, experimental/attention/CMakeLists.txt
- 缺陷描述：ENABLE_EXPERIMENTAL模式下attention目录构建配置错误：custom_build.cmake无条件add_subdirectory(attention)未切换到experimental路径；func.cmake重复包含experimental/attention的glob与CMakeLists.txt自身遍历冲突；CMakeLists.txt缺少按ASCEND_OP_NAME过滤能力。
- 修复模式：增加ENABLE_EXPERIMENTAL条件分支；删除重复glob；重写CMakeLists支持按算子名过滤
- 可审查性：高
- 审查规则建议：experimental和非experimental路径应互斥且完整；构建目录遍历逻辑应与其他模块保持一致

### 9cff0e48 fix_gmm
- 分类：非缺陷
- 原因：纯文档修改（markdown内容清理和锚点修正）

### 1bc7f8dc fix: update version for FIA/IFA/PFA/MlaProlog readme && delete FIA_vx
- 分类：非缺陷
- 原因：文档更新 + 废弃VX版本算子代码清理

### b11cce04 fix FA compile error for arch32
- 根因类别：赋值遗漏/重命名不同步
- 涉及文件：attention/common/op_host/fia_tiling_info.h, attention/common/op_host/arch32/*.cpp, attention/common/op_kernel/arch32/*.h, attention/fused_infer_attention_score/op_host/arch32/*.cpp/*.h, common/src/tiling_sink/CMakeLists.txt
- 缺陷描述：attenMaskSize字段在主线代码已重命名为attenMaskBatchStride（语义更准确：batch stride而非size），但arch32分支代码未同步更新致编译失败。另tiling_sink的CMakeLists.txt缺少v2版本和arch35的3个源文件。
- 修复模式：全局重命名attenMaskSize→attenMaskBatchStride；CMakeLists补充缺失源文件
- 可审查性：高
- 审查规则建议：重命名字段时全局搜索所有引用点（含不同arch分支）；CI覆盖所有目标架构编译

### 10697b77 fix delete setenv.sh bug
- 根因类别：构建/部署配置缺陷
- 涉及文件：scripts/package/common/sh/script_operator.inc
- 缺陷描述：add_setenv()/del_setenv()函数包含对setenv.sh的完整操作逻辑，但ops-transformer包不应管理setenv.sh，会导致安装/卸载时错误操作setenv.sh。
- 修复模式：将两个函数体清空为return 0
- 可审查性：中
- 审查规则建议：安装/卸载脚本的环境变量管理逻辑需评估对用户环境的影响

### bfbbec83 修改moe_gating_top_k_softmax算子的BlockReduceSum/MaxIntrinsicsImpl接口
- 根因类别：API误用/底层接口误用
- 涉及文件：moe/moe_gating_top_k_softmax/op_kernel/moe_gating_top_k_softmax_perf.h
- 缺陷描述：使用底层intrinsics接口BlockReduceMaxIntrinsicsImpl/BlockReduceSumIntrinsicsImpl（直接操作裸指针__ubuf__ float*），不符合AscendC编程规范，存在类型安全风险。6处调用。
- 修复模式：替换为高层模板接口BlockReduceMax<float,false>/BlockReduceSum<float,false>，使用LocalTensor和MASK_PLACEHOLDER
- 可审查性：高
- 审查规则建议：禁止直接调用*IntrinsicsImpl底层接口；对直接操作__ubuf__指针的代码标记告警

### 32732476 fix nQueryIndex = 8 MTE address out of range bug
- 根因类别：DataCopy对齐越界
- 涉及文件：attention/sparse_lightning_indexer_grad_kl_loss/op_kernel/sparse_lightning_indexer_grad_kl_loss_vector.h
- 缺陷描述：gSizeQueryIndex==8时，DataCopy用AlignTo将长度从8对齐到16，但GM中实际只有8个元素，DMA引擎读取超出范围的后8个元素地址，造成MTE地址越界。
- 修复模式：对gSizeQueryIndex==8使用DataCopyPad替代DataCopy，通过DataCopyExtParams指定实际搬运字节数，DataCopyPadExtParams指定rightPadding=8填充值0
- 可审查性：高
- 审查规则建议：DataCopy搬运长度必须满足对齐要求（通常32字节）；不对齐场景应使用DataCopyPad；对MTE搬运长度参数做边界检查

### 0387ed9a 修复pfa example执行失败 && static脚本不支持3.7
- 根因类别：构建配置缺陷(链接库缺失 + Python版本兼容性)
- 涉及文件：build.sh, scripts/util/build_opp_kernel_static.py
- 缺陷描述：(1)example编译链接命令缺少-lc_sec致链接失败；(2)Path.unlink(missing_ok=True)是Python 3.8+才有的参数，Python 3.7下抛TypeError。
- 修复模式：g++命令添加-lc_sec；替换为try/except FileNotFoundError兼容3.7
- 可审查性：高
- 审查规则建议：新增依赖库引用同步所有编译命令；项目声明最低Python版本并检查新版API兼容性

### 19de45cc [MlaProlog] Fix precision issue when queryNormFlag is set to true
- 根因类别：workspace/buffer偏移计算错误
- 涉及文件：attention/mla_prolog/op_kernel/kernel_mla_prolog_split_n.h
- 缺陷描述：WorkspaceInit中，mmCqResGm_后的workspace偏移逻辑放在queryNormFlag条件判断外无条件执行。当queryNormFlag=true时，rmsNormCq结果直接输出到gm不经workspace，但后续mmQcQrResGm_等buffer仍需跳过mmCqResGm_空间。原代码此偏移仅在dtype不同时才执行，queryNormFlag=true下偏移缺失，导致workspace地址计算错误，后续buffer重叠引发精度问题。
- 修复模式：将偏移逻辑拆分到queryNormFlag==0和else两个分支，else分支无条件偏移mmCqOutputType大小
- 可审查性：中
- 审查规则建议：workspace中多buffer共享连续内存时，必须检查所有条件分支下偏移量计算一致性；review时应验证每个分支下的内存布局确保buffer不重叠

### f8ab5505 revert add_example
- 分类：非缺陷
- 原因：回退之前添加的add_example样例代码，代码整理

### 4138bd36 version.info add required_package info
- 分类：非缺陷
- 原因：版本配置/元信息更新，新增依赖包版本信息

### f05c0242 修复tiling下沉编译失败
- 根因类别：构建配置缺陷
- 涉及文件：common/src/tiling_sink/CMakeLists.txt
- 缺陷描述：CMakeLists.txt的src_files列表引用了4个已不存在的cpp文件（v2版本和arch38的tiling文件），文件在重构中被删除但CMakeLists未同步更新，tiling下沉模式编译失败。
- 修复模式：从src_files中移除4个不存在的源文件引用
- 可审查性：高
- 审查规则建议：删除/重命名源文件时必须同步更新所有CMakeLists.txt；CI应覆盖所有构建配置

### 8ab6568b aclnnFlashAttentionScoreV3部分链接修复
- 分类：非缺陷
- 原因：纯文档链接路径修正（docs/context → docs/zh/context）

### 2ff66c1c docs资料描述错误需要更改
- 分类：非缺陷
- 原因：纯文档修改（补充链接依赖说明、删除重复表格、修正安装路径）

### 4801ecb2 attentionupdate支持lse inf输入
- 根因类别：特殊值(inf/NaN)输入未处理
- 涉及文件：attention/attention_update/op_kernel/decode_update.h
- 缺陷描述：DecodeUpdate算子对lse输入执行max/exp/sum运算。多SP域场景下某些域无有效token，lse为+inf。exp(+inf - +inf) = exp(NaN) = NaN，污染最终输出。原代码未对+inf做特殊处理。
- 修复模式：新增ProcessLseInfReplacement函数，用CompareScalar检测+inf元素，Select替换为-inf。后续max运算中-inf不影响结果，exp(-inf - max)=0而非NaN。同时调整UB内存分配增加selMaskBuffer。
- 可审查性：中
- 审查规则建议：数学运算类算子review时应系统检查exp/log/div/sqrt在边界值(0/+inf/-inf/NaN)下的行为；UT应包含inf/NaN边界测试用例

### d8e8c382 opapi_math 替换opapi.so
- 分类：非缺陷
- 原因：上游依赖库名变更后的全局适配(opapi→opapi_math)，非代码逻辑缺陷

### 6d8abc8d fix:aclnnFusedInferAttentionScoreV4GetWorkspaceSize add learnableSinkOptional
- 分类：非缺陷
- 原因：纯文档修改，API函数原型声明补充遗漏参数

## 批次17: 提交 #315-#334 (2025-12-23 ~ 2025-12-01)

---

### 2b848593 修改version.info版本号
- 分类：非缺陷
- 原因：纯版本号变更(8.5.0.alpha001→8.5.0-beta.1)

---

### e40a4176 修复aclnn接口入参顺序
- 根因类别：API误用/接口参数错误
- 涉及文件：moe/moe_token_unpermute_with_ep_grad/op_host/moe_token_unpermute_with_ep_grad_def.cpp
- 缺陷描述：算子def.cpp中Attr属性注册顺序与aclnn接口参数顺序不匹配。range和topk_num排在padded_mode和restore_shape之前，但aclnn接口要求Attr注册顺序与接口入参顺序严格一致。
- 修复模式：交换Attr声明行顺序，使padded_mode/restore_shape在前，range/topk_num在后。
- 可审查性：中(需对照aclnn接口声明确认正确顺序)
- 审查规则建议：编写lint规则将_def.cpp中Attr注册顺序与对应aclnn接口声明自动比对

---

### 0a278590 修复FlashAttentionScore和FlashAttentionScoreGrad文件中aclnn文档问题
- 分类：非缺陷
- 原因：纯文档修复(12个md文件的参数描述补充)

---

### 88c2d46d 修复gmm和attention文件中aclnn文档问题
- 分类：非缺陷
- 原因：纯文档修复(清理git merge冲突标记残留、删除重复表格)

---

### befd8bcd fix bug for datacopysoftmaxlse bsnd layout
- 根因类别：多处关联缺陷(API误用+条件分支遗漏+功能限制误判)
- 涉及文件：attention/common/op_kernel/arch32/fia_block_vec_nonquant_mla.h, attention/fused_infer_attention_score/op_host/fused_infer_attention_score_tiling_check_feature.cpp, attention/fused_infer_attention_score/op_host/fused_infer_attention_score_tiling_v3.cpp
- 缺陷描述：三个关联问题。(1) BSND布局下调用DataCopySoftmaxLseBSND时多传了2个参数(qActSeqLensParser和info.bIdx)，该函数BSND布局仅需6参数。(2) CheckFeatureMlaNoquantUnsupported中错误阻止了BSND布局下的softmaxLse输出。(3) CheckGqaConstrain和CheckMlaConstrain函数体仅`return false`(占位符代码)，未实现完整的GQA/MLA约束判断逻辑，导致所有GQA/MLA场景均被错误拦截。
- 修复模式：删除多余参数；移除错误的功能限制检查；补全约束判断逻辑(+61行)，新增isNotLegacyGQA辅助函数区分legacy/non-legacy GQA模式。
- 可审查性：低(跨3文件3种修复类型，需深入理解FIA算子tiling策略)
- 审查规则建议：静态分析检测函数体仅含`return false/true`的约束检查函数，标记为可疑占位符代码；函数调用参数数量校验

---

### 881ed56a MlaPrologV3算子拦截问题修复
- 根因类别：多处校验缺陷(sizeof计算+InferShape维度+平台注册+空指针)
- 涉及文件：attention/mla_prolog/op_host/mla_prolog_tiling.cpp, mla_prolog_tiling.h, mla_prolog_tiling_check.cpp, attention/mla_prolog_v3/op_host/mla_prolog_v3_def.cpp, mla_prolog_v3_infershape.cpp, aclnn_mla_prolog_v2_weight_nz.cpp, aclnn_mla_prolog_v3.cpp
- 缺陷描述：(1) sizeof(char_array)-1导致strncmp比较长度不足，前缀相同的opType名称会误匹配。(2) InferShape将可选输出"不使用"标记为dim=1而非dim=0，下游误认为有有效数据。(3) V3算子未注册ascend910b平台，该平台完全无法调用。(4) actualSeqLen在特定cacheMode下可能为空指针直接解引用。(5) CheckTensorNotNullWhen/CheckTensorNullWhen两个独立判断存在逻辑不一致。
- 修复模式：修正sizeof计算、dim=1改dim=0、补全平台注册、加空指针保护、合并互斥校验函数。
- 可审查性：中
- 审查规则建议：sizeof(char_array)-1与strncmp配合检查是否正确处理终止符；算子def.cpp中AddConfig是否覆盖所有目标平台；InferShape中可选输出空语义应用dim=0而非dim=1

---

### e71e9b7d Fix cust example
- 根因类别：构建配置缺陷
- 涉及文件：build.sh
- 缺陷描述：构建example时CUST_LIBRARY_PATH/CUST_INCLUDE_PATH始终使用ASCEND_OPP_PATH下的路径。当用户设置了ASCEND_CUSTOM_OPP_PATH(自定义算子包非默认路径)时，脚本未使用该自定义路径，导致编译链接失败。
- 修复模式：增加ASCEND_CUSTOM_OPP_PATH环境变量判断，存在时优先使用。
- 可审查性：高
- 审查规则建议：涉及ASCEND_OPP_PATH路径拼接时检查是否需要考虑ASCEND_CUSTOM_OPP_PATH覆盖

---

### 88a950a2 修复quick_install描述
- 分类：非缺陷
- 原因：纯文档更新(安装指南md文件)

---

### 615303d6 move nn norm/kv_rms_norm_rope_cache to ops-transformer posembedding
- 分类：非缺陷
- 原因：代码迁移操作(37个新增文件，0删除)

---

### a8ac47bb issue of combineARN docs
- 分类：非缺陷
- 原因：纯文档修复(2个md文件的API参数表格调整)

---

### 1005700c rm version.info required_package
- 分类：非缺陷
- 原因：构建配置清理(移除version.info中内部依赖版本声明)，非实际构建问题修复

---

### 29b08607 修复moe_token_unpermute_grad和moe_token_unpermute_with_routing_map中op_api的UT
- 根因类别：UT测试代码缺陷+构建配置缺陷
- 涉及文件：moe/moe_token_unpermute_grad/op_host/CMakeLists.txt, tests/ut/op_host/op_api/test_aclnn_moe_token_unpermute_grad.cpp, tests/ut/op_kernel/test_moe_token_unpermute_grad.cpp, moe/moe_token_unpermute_with_routing_map/tests/ut/op_host/op_api/(新增CMakeLists.txt+test文件)
- 缺陷描述：(1) UT include路径引用了闭源仓库路径`level2/aclnn_moe_token_unpermute_grad.h`，开源仓库中不存在，UT编译失败。(2) UT通过system()调用拷贝闭源路径下的gen_data脚本，路径不存在导致运行失败。(3) CMakeLists.txt缺少依赖声明，moe_token_unpermute_with_routing_map完全缺少op_api级UT。
- 修复模式：路径改为相对路径；删除闭源路径的硬编码system调用；补充缺失的CMake配置和UT文件。
- 可审查性：高
- 审查规则建议：扫描UT中include路径和system()调用是否引用了仓库外路径

---

### a4e02b39 tp issue
- 根因类别：条件分支遗漏/缺陷
- 涉及文件：mc2/moe_distribute_combine/op_host/op_tiling/moe_distribute_combine_tiling.cpp, mc2/moe_distribute_combine_add_rms_norm同名文件, mc2/moe_distribute_combine_v2同名文件, mc2/moe_distribute_dispatch/op_host/op_tiling/moe_distribute_dispatch_tiling.cpp, mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch.h, mc2/moe_distribute_dispatch_v2同名文件
- 缺陷描述：MoE distribute/combine/dispatch系列算子在tpWorldSize==1(无需TP并行)时仍无条件初始化TP通信组和获取HCCL通信上下文。(1) tiling层无条件对不存在的TP通信组做SetGroupName/SetOpType/GetTiling，可能导致通信初始化异常。(2) kernel层无条件GetHcclContext<1>()，当只有一个通信组时该上下文无效，导致非法内存访问。(3) SetHCommCfg在获取tpWorldSize值之前就被调用，参数依赖未满足。
- 修复模式：给TP操作加`if (tpWorldSize > 1)`守卫；kernel中将GetHcclContext<1>()移入条件块；调整SetHCommCfg调用顺序。
- 可审查性：中
- 审查规则建议：HCCL通信上下文获取必须有worldSize>1守卫；函数参数依赖变量应在调用前完成初始化

---

### c362f74b modify fd invalidrows And fix compile issue
- 根因类别：多处缺陷(编译+运行时逻辑)
- 涉及文件：attention/common/op_kernel/arch32/fia_block_vec_flashdecode.h, fia_block_vec_nonquant.h, fia_block_vec_nonquant_mla.h, fia_kernel_nonquant.h, fia_kernel_nonquant_mla.h, memory_copy.h, vector_common.h
- 缺陷描述：综合性修复包含6类问题。(1) 头文件中`using namespace fa_base_vector`导致命名空间污染，引入新同名符号后编译二义性。(2) 变量在if块内声明但在块外使用，某些编译路径报"未声明标识符"。(3) DealInvalidRowsBelow循环条件`s1RealEnd > 0`漏处理第0行，应为`>= 0`。(4) FlashDecode的invalidRows处理时机错误，应在reduce之后、最终输出之前。(5) fdLseMaxUbBuf/fdLseSumUbBuf单buffer但按cntM%2做ping-pong切换导致数据覆盖，需改双buffer。(6) GetSafeActToken重复定义且mla版本用magic number，提取为公共函数。
- 修复模式：命名空间显式限定；修复变量作用域；循环边界>=0；调整处理时序；双buffer替代单buffer；公共函数提取。
- 可审查性：低(7个文件6种修复混在一个提交)
- 审查规则建议：禁止头文件using namespace；ping-pong双流模式共享buffer必须为双buffer；for循环处理index时注意0边界

---

### af587384 fix moe_token_unpermute_with_routing_map_grad md
- 分类：非缺陷
- 原因：纯文档修复(md文件公式符号修正+错误复制内容删除)

---

### 2adb2526 fix cmake for custom
- 根因类别：构建配置缺陷
- 涉及文件：27个op_host/CMakeLists.txt(gmm/moe/rope等多个算子目录)
- 缺陷描述：add_ops_compile_options被放在if(BUILD_OPS_RTY_KERNEL)分支下，与if(BUILD_OPEN_PROJECT)平级。CUSTOM包编译路径(BUILD_OPEN_PROJECT=true)时kernel编译选项(--cce-auto-sync/-Wno-deprecated-declarations/-Werror)不会被添加，导致CUSTOM包编译失败或行为异常。
- 修复模式：将add_ops_compile_options移入BUILD_OPEN_PROJECT条件块内，27个文件统一修改。
- 可审查性：中(单文件改动简单但需逐一确认27个文件一致性)
- 审查规则建议：add_ops_compile_options应在预期的编译目标条件作用域内

---

### 578ed128 fix: deqScale2 dataType name
- 分类：非缺陷
- 原因：纯文档拼写修正(FLOAT32r→FLOAT32)

---

### 054c31af nsacompressattentioninfer算子tiling侧actualQSeqLengths校验修复
- 根因类别：输入校验缺失(可选tensor判空逻辑错误)
- 涉及文件：attention/nsa_compress_attention_infer/op_host/nsa_compress_attention_infer_tiling.cpp
- 缺陷描述：SplitBN()和GetMaxQSeqlen()中用`actualQSeqLengths.tensor != nullptr`判断可选参数是否传入，但tensor对象可能存在(指针非null)但内部数据为空。应检查`tensor->GetData<int64_t>() != nullptr`。错误判断导致读取空数据，tiling计算错误。
- 修复模式：将判空从"检查tensor指针"改为"检查tensor内部数据指针"，2处同模式修改。
- 可审查性：高
- 审查规则建议：可选tensor参数判空应统一使用GetData()返回值；静态分析对`.tensor != nullptr`在可选参数上下文中发出警告

---

### 9f5283f9 fix ut and permute indices size
- 根因类别：workspace/buffer大小计算错误
- 涉及文件：moe/moe_token_permute/op_host/moe_token_permute_tiling.cpp, moe/moe_token_permute_with_ep/op_host/moe_token_permute_with_ep_tiling.cpp及其op_kernel/moe_index_copy.h和moe_index_copy_spilt_d.h, moe/moe_token_permute_with_routing_map/op_host/moe_token_permute_with_routing_map_tiling.cpp
- 缺陷描述：indices buffer大小计算时`UpAlign(onceIndices, ONE_BLOCK_BYTE)`按元素数量而非字节数对齐，应为`UpAlign(onceIndices * INT32_DTYPE_SIZE, ONE_BLOCK_BYTE)`。导致分配的字节数只有实际需要的1/4。kernel侧原来用`indicesUB * 4`做补偿，tiling/kernel间buffer大小语义不一致。另外moe_token_permute_with_routing_map中topK校验缺少paddedMode前置条件导致误报。
- 修复模式：tiling侧乘以INT32_DTYPE_SIZE统一按字节计算；kernel侧去掉补偿乘法；增加paddedMode前置条件。
- 可审查性：中
- 审查规则建议：UpAlign()第一个参数应始终为字节数；tiling和kernel之间buffer大小语义需保持一致，避免一侧元素另一侧字节的补偿模式

---

### 62ff8d25 修复首页md目录问题
- 分类：非缺陷
- 原因：纯文档更新(README目录树+开发指南md)

## 批次18: 提交 #335-#354 (2025-11-26 ~ 2025-11-10)

---

### 942d5acd 128P问题gmmalltoallv修复
- 根因类别：校验条件范围错误(硬编码白名单不完整)
- 涉及文件：mc2/grouped_mat_mul_allto_allv/op_host/op_tiling/grouped_mat_mul_allto_allv_tiling.cpp
- 缺陷描述：epWorldSize合法值白名单{8,16,32,64}缺少128，导致128P(128个EP节点)场景下tiling校验返回false，算子无法执行。
- 修复模式：扩展白名单枚举值加入128，同步更新错误日志文本。
- 可审查性：高
- 审查规则建议：硬编码白名单/枚举值校验应与上游规格文档一致；std::find模式的校验分支需验证白名单完备性

---

### eb8dda97 fix moe_init_routing md
- 分类：非缺陷
- 原因：纯文档格式修改(README.md表格布局调整)

---

### a46890ac fix moe_init_routing_quant md
- 分类：非缺陷
- 原因：纯文档格式修改(README.md表格布局调整)

---

### 64be6dd6 nsa_compress_attention_infer fix example by fixing mem.h InitBuffer
- 根因类别：赋值遗漏/结构体字段遗漏(复制粘贴错误)
- 涉及文件：common/include/kernel/mem.h
- 缺陷描述：AsdopsBuffer构造函数中初始化tensor数组时，L0A/L0B/L0C三种buffer类型的数组索引全部错误地写成了ASCEND_CB。连续3行`tensor[(uint32_t)BufferType::ASCEND_CB] = ...`不断覆盖同一个slot(tensor[1])，而tensor[2](L0A)、tensor[3](L0B)、tensor[4](L0C)从未被初始化。后续通过GetBuffer<ASCEND_L0A>等访问时得到未初始化tensor。两个条件编译分支各3处，共6处相同错误。
- 修复模式：将数组索引修正为对应的BufferType枚举(ASCEND_L0A/ASCEND_L0B/ASCEND_L0C)。
- 可审查性：高
- 审查规则建议：静态分析检测"同一数组相同索引被连续赋值多次"的模式，几乎总是复制粘贴遗漏修改的bug

---

### 0b12dd13 fix moe_gating_top_k_softmax_v2 md
- 分类：非缺陷
- 原因：纯文档修改(README.md参数分类和格式修正)

---

### cb655e0a fix moe_compute_expert_tokens md
- 分类：非缺陷
- 原因：纯文档修改(README.md表格格式调整)

---

### ce60c7d8 fix moe_finalize_routing_v2_grad md
- 分类：非缺陷
- 原因：纯文档修改(README.md表格样式+参数分类修正)

---

### d2e6b827 fix ut bug & support infer_datatype/infer_shaperange
- 根因类别：多处缺陷(UT框架+构建系统+API适配)
- 涉及文件：build.sh, cmake/ut.cmake, cmake/obj_func.cmake, common/stub/op_api/CMakeLists.txt, tests/ut/framework_normal/下多个文件，共32个文件
- 缺陷描述：包含多类修复。(1) build.sh中变量名拼写错误`UT_TARGES`应为`UT_TARGETS`(4处)，导致UT构建目标无法追加，UT不会被编译执行。(2) cmake/ut.cmake中`${CMAKE_CURRENT_SOURCE_DIR}`误用应为`${MODULE_DIR}`(多处)，路径解析错误。(3) cmake/ut.cmake中空字符串比较`""`应为`"ALL"`(5处)，--ops过滤逻辑失效。(4) UT框架中EXPECT_EQ被注释掉，infershape测试校验被跳过。(5) AscendString未加ge::命名空间限定。(6) op_kernel的tiling so路径硬编码错误。(7) 头文件guard从OPS_MATH_DEV改为OPS_TRANSFORMER_DEV(仓库复制遗留)。
- 修复模式：变量名拼写修正+路径修正+条件逻辑修正+取消注释断言+命名空间限定+API适配。
- 可审查性：低(32个文件，混合bug修复/API适配/新功能，未拆分)
- 审查规则建议：禁止提交被注释的EXPECT_EQ/ASSERT_EQ测试断言；shell变量名引用一致性检查；头文件guard应与当前仓库名一致

---

### 6e16f3dc nsa_selected_attention_infer fix example
- 根因类别：变量名typo(头文件名拼写错误)
- 涉及文件：attention/nsa_selected_attention_infer/examples/test_aclnn_nsa_selected_attention_infer.cpp
- 缺陷描述：include头文件名`aclnn_nsa_select_attention_infer.h`缺少`ed`后缀，应为`aclnn_nsa_selected_attention_infer.h`，导致example编译失败。
- 修复模式：修正头文件名拼写。
- 可审查性：高
- 审查规则建议：example代码提交应有编译验证CI gate

---

### 8b73026b fix example
- 根因类别：构建配置缺陷
- 涉及文件：build.sh
- 缺陷描述：build_example()和build_example_group_eager()中，-I ${EAGER_INCLUDE_OPP_ACLNNOP_PATH}头文件搜索路径原本只在MC2条件分支添加，但所有算子example编译都需要该路径。非MC2算子example编译时找不到aclnnop头文件报错。
- 修复模式：将-I路径从MC2条件分支移到通用g++编译命令行。
- 可审查性：高
- 审查规则建议：build脚本中通用头文件路径不应放入条件分支

---

### c29bc106 nsa_select_attention_infer: rename
- 分类：非缺陷
- 原因：纯重命名重构(select→selected批量替换)

---

### 47926d5a fix gmm_add_clean_code
- 根因类别：计算逻辑错误(L1 partA/B值与使用关系反转)
- 涉及文件：gmm/grouped_matmul_add/op_host/grouped_matmul_add_tiling.cpp
- 缺陷描述：原代码BEST_L1_PARTA=256K/PARTB=128K，但mmStepKa用PARTB计算、mmStepKb用PARTA计算，partA/B的值分配和使用关系是反的。修复交换值+纠正使用关系(stepKa用partA, stepKb用partB)。该缺陷被CleanCode标签掩盖，实际改变了stepKa和stepKb的计算结果。
- 修复模式：常量值纠正+变量使用关系纠正+显式类型转换增强。
- 可审查性：低(被CleanCode修改掩盖)
- 审查规则建议：常量定义值的变更应作为高优先级审查项；常量值和使用位置同时修改时需验证净效果

---

### 12461973 修复rope_with_sin_cos_cache example
- 根因类别：构建配置缺陷(头文件路径错误)
- 涉及文件：posembedding/rope_with_sin_cos_cache/examples/test_aclnn_rope_with_sin_cos_cache.cpp
- 缺陷描述：include路径`aclnnop/level2/aclnn_rope_with_sin_cos_cache.h`多了一层level2/子目录，应为`aclnnop/aclnn_rope_with_sin_cos_cache.h`。导致example编译失败。
- 修复模式：修正include路径。
- 可审查性：高
- 审查规则建议：include路径应与实际SDK头文件安装路径自动一致性校验

---

### 807caa4c NpuOpsTransformerExt 支持
- 分类：非缺陷
- 原因：纯新feature(新增experimental/npu_ops_transformer_ext工程模板)

---

### 4f5aeb89 mc2 ut kernel fix
- 根因类别：UT测试代码缺陷+构建配置缺陷
- 涉及文件：mc2目录下33个文件(all_gather_matmul, allto_all_all_gather_batch_mat_mul, matmul_all_reduce等多个算子)
- 缺陷描述：(1) 算子kernel中的相对include路径在UT构建目录下找不到头文件，通过__has_include宏增加备选路径fallback。(2) UT中tiling结构体名(如Mc2BatchMatmulTilingData)与主代码结构体名(BatchMatmulTilingData)不一致，UT编译失败。(3) 多个kernel入口函数在__CCE_KT_TEST__宏下缺少REGISTER_TILING_DEFAULT注册，UT运行时无法获取tiling数据。
- 修复模式：条件编译路径兼容+结构体命名同步+UT基础设施补全。
- 可审查性：中(33个文件但模式高度重复)
- 审查规则建议：UT中手写tiling_def.h应与主代码tiling结构体自动同步

---

### 7b44fa41 冒烟功能的看护优化
- 分类：非缺陷
- 原因：CI/CD测试看护优化(新增PR变更文件解析和自动关联UT/example)

---

### e96b7b2a 修复PFA的非32B对齐情况下的精度问题
- 根因类别：整数类型错误/溢出(硬件指令repeat参数溢出uint16)
- 涉及文件：attention/prompt_flash_attention/op_kernel/prompt_flash_attention_s1s2_bns1_x310_base.h
- 缺陷描述：CopyND2NZOnThe函数中对目标buffer做零初始化时，Duplicate的repeat参数=calcWidth*calcHeightAlign可能超过255(硬件InitConstValue的repeatTime字段上限为uint16)。当Query_seqlen在69~80之间且未16对齐时触发溢出，初始化不完整，buffer残留脏数据引发精度错误。
- 修复模式：将单次Duplicate改为循环调用InitConstValue，每次最多处理255个repeat(MAX_REPEATS_PER_BATCH)。
- 可审查性：中
- 审查规则建议：硬件intrinsic的repeat/repeatTime参数来源于运行时计算时，应检查是否可能超过255/65535上限，强制要求分批或加断言

---

### 55adcbeb 移除冗余输出
- 分类：非缺陷
- 原因：删除一行调试用std::cout输出，纯代码清理

---

### 24d5f4bb 回黄和builtin add_subdirectory统一，以attention为例
- 分类：非缺陷
- 原因：构建系统(CMake)重构统一，非运行时代码缺陷修复

---

### b4aada9f fix sequential bug in gqa antiquant kvnz
- 根因类别：硬件流水线同步缺失
- 涉及文件：attention/incre_flash_attention/op_kernel/incre_flash_attention_preload_dd.h
- 缺陷描述：DealAntiqBmm2ResBaseBlock函数中，DataCopy(bmm2ResPreUb, vec2ResUb, ...)读取Vector流水线的输出buffer，但之前没有PipeBarrier<PIPE_V>()。Vector写操作可能尚未完成，DataCopy读取到不完整/旧数据。GQA antiquant KV NZ场景下的RAW数据竞争。
- 修复模式：在DataCopy前插入PipeBarrier<PIPE_V>()。
- 可审查性：低
- 审查规则建议：DataCopy源操作数来自Vector计算结果时，必须确保前方存在PipeBarrier<PIPE_V>()

## 批次19: 提交 #355-#374 (2025-11-08 ~ 2025-10-31)

---

### 23b7944c support more ranknum and fix aivnum
- 根因类别：硬编码常量导致越界+调度模式配置错误
- 涉及文件：mc2/elastic_receivable_test/op_host/op_tiling/elastic_receivable_test_tiling.cpp, mc2/elastic_receivable_test/op_kernel/elastic_receivable_test.h
- 缺陷描述：(1) AIV_NUM_USED硬编码为6，但实际应为1(每个核独立处理)，导致sendRankNum_=rankNum_/aivNum_除法结果错误，任务分配不正确。(2) SetScheduleMode(1)(batch mode)应为SetScheduleMode(0)(normal mode)，调度模式不匹配算子执行场景。
- 修复模式：修正常量值6→1；修正调度模式1→0。
- 可审查性：中
- 审查规则建议：AIV_NUM_USED等硬编码核数常量应有取值依据注释；SetScheduleMode取值应与算子核间同步语义匹配

---

### c8f181eb revert//schedule
- 根因类别：前序PR引入回归错误(Revert)
- 涉及文件：attention/common/op_host/arch32/fia_tiling_nonquant_mla.cpp, fia_tiling_nonquant_mla.h, fia_tiling_base.h
- 缺陷描述：撤销MR!231(commit ff21f09b)引入的CalcScheduleMode功能。原始提交方法定义为FiaTilingNonQuant::CalcScheduleMode()但类声明在FiaTilingNonQuantMla中——类名不匹配，导致方法绑定到错误的类。
- 修复模式：Revert整个有缺陷的提交。
- 可审查性：高
- 审查规则建议：方法定义的类名与声明所在类必须一致

---

### 2090000e fix:修复mla_prolog_v3 资料 & exampe 执行报错
- 分类：非缺陷
- 原因：文档和example代码的数据类型/参数修正(BF16→INT8等)，不涉及算子运行时代码

---

### f66e305f fix md jump link
- 分类：非缺陷
- 原因：纯文档修改(22个md文件的相对路径跳转链接修正)

---

### ea47405a elastic_receivable_info_collect功能
- 分类：非缺陷
- 原因：纯新feature(全部19个文件均为新建，+2071行)

---

### 3037d282 修正mc2下部分example缺失打印信息的问题
- 分类：非缺陷
- 原因：example示例代码完善(补充打印信息、_exit改return等)

---

### 4fd31731 add:nsa_select_attention_infer_example
- 分类：非缺陷
- 原因：新增example调用示例

---

### 79cd3244 bugfix:nsa_select_infer_ut
- 根因类别：UT测试代码缺陷+构建配置缺陷
- 涉及文件：attention/nsa_select_attention_infer/tests/CMakeLists.txt, tests/utest/ts_nsa_select_attention_infer.h
- 缺陷描述：(1) CMakeLists.txt中file(GLOB_RECURSE)路径缺少算子目录前缀，找不到测试源文件。(2) message()和file()末尾多余反斜杠续行符导致CMake解析异常。(3) 测试类继承Ts_WithParam时缺少模板参数`<NsaSelectAttentionInferCase>`，编译失败。
- 修复模式：修正路径+删除多余续行符+补全模板参数。
- 可审查性：高
- 审查规则建议：CMake GLOB路径应包含完整算子目录前缀；模板类继承必须指定模板参数

---

### c05e6505 Barrier支持故障检测新增elasticInfo和timeOut可选输入
- 分类：非缺陷
- 原因：纯新feature(为DistributeBarrier算子新增可选输入)

---

### a7010fc9 fix ut for moe_init_routing_v3, moe_re_routing, moe_gating_top_k, interleave_rope
- 根因类别：多处生产代码缺陷(空指针校验对象写错+参数传递错误+API误用)
- 涉及文件：moe/moe_gating_top_k/op_host/moe_gating_top_k_tiling.cpp, moe_gating_top_k_tiling_arch35.cpp, moe_gating_top_k_proto.h, moe_gating_top_k_tiling.h
- 缺陷描述：虽标题为"fix ut"，但同时修了多个生产代码bug。(1) 获取expertIdxPtr后却校验了yShapePtr(copy-paste错误)，expertIdxPtr为空时不被捕获，后续解引用崩溃。(2) OP_LOGE的printf参数顺序与格式串不匹配(groupSelectMode_和perGroupExpertCount_传反)。(3) bias是optional输入却用了GetInputShape而非GetOptionalInputShape，未提供bias时异常。(4) 浮点字面量1e-20应为1e-20f(Float属性类型不匹配)。
- 修复模式：修正校验对象名+调换printf参数顺序+改用GetOptionalInputShape+加f后缀。
- 可审查性：高
- 审查规则建议：OP_CHECK_NULL校验对象应与上一行获取的指针变量名一致；optional输入必须用GetOptionalInputShape

---

### fc29fc0c fix:nsa_select_attention_infer_ut_example
- 分类：非缺陷
- 原因：UT框架搭建+example补充

---

### 0941b78c 修复GMMFR算子文档中变量名错误以及描述不通顺的问题
- 分类：非缺陷
- 原因：纯文档修改(5个md文件的变量名和描述修正)

---

### e03da471 [bugfix]NsaCompressAttentionWithCache UT 整改
- 分类：非缺陷
- 原因：UT工程结构整改(目录重组+CMake重写)

---

### 61ca689a nsa_compress_attention_infer fix ut
- 分类：非缺陷
- 原因：UT结构整改+tiling UT补充

---

### 644978ae nsa_compress_attention_infer and nsa_compress_with_cache example fix
- 分类：非缺陷
- 原因：example补全(新增2个example文件+文档)

---

### b9a02e9d fix rope_with_sin_cos_cache ut
- 根因类别：UT测试数据类型错误
- 涉及文件：posembedding/rope_with_sin_cos_cache/tests/ut/op_kernel/gen_data.py, test_case_fp32.cpp, test_case_fp16.cpp, op_host/test_rope_with_sin_cos_cache_infershape.cpp
- 缺陷描述：多处UT数据类型错误。(1) gen_data.py中position数据用传入的dtype(float)而非np.int64，位置索引必须是整数。(2) int(numQHeads * headSize)在字符串参数时做字符串重复而非数值乘法。(3) test_case_fp32中所有buffer size用sizeof(half)而非sizeof(float)，内存分配只有实际需要的一半，越界读写。(4) fp32/fp16测试都传了'bf16' dtype给gen_data.py。(5) infershape头文件名不匹配导致编译失败。
- 修复模式：修正数据类型+修正sizeof+修正dtype参数+修正头文件名。
- 可审查性：高
- 审查规则建议：UT中sizeof类型必须与测试数据类型一致；gen_data脚本的dtype参数应与测试用例对应

---

### 55ea73d2 fix_rope_quant_kvcache_ut
- 分类：非缺陷
- 原因：UT补充(取消注释启用tiling UT编译+新增tiling测试用例)

---

### 68b44c1e fix_moe_token_permute_with_routing_map_grad_ut
- 分类：非缺陷
- 原因：UT补充(取消注释启用UT编译+格式化)

---

### 1e927dbf [Bug-Report|缺陷反馈]: PromptFlashAttention缺少kernel部分UT用例
- 分类：非缺陷
- 原因：UT补充+CMake构建脚本重构(新增tiling/kernel/opapi测试用例)

---

### 8bfb5c84 修复算子的kernel_ut：moe_token_unpermute_grad等
- 根因类别：UT测试代码缺陷(路径依赖+类型错误)
- 涉及文件：moe/moe_token_unpermute_grad/tests/ut/op_kernel/, moe/moe_token_unpermute_with_routing_map/tests/ut/op_kernel/, moe/moe_token_unpermute_with_ep_grad/tests/ut/op_kernel/, common/include/kernel/masked_select.h
- 缺陷描述：(1) kernel UT通过system()调用依赖外部路径`ops/built-in/tests/ut/fast_op_test/`下的Python脚本生成测试数据，独立构建环境下路径不存在导致UT异常。(2) masked_select.h中模板类型约束错误(uint16_t应为half，int8_t应为bool)导致编译问题。
- 修复模式：移除外部路径依赖，改为C++内联tiling数据；修正模板类型。
- 可审查性：高
- 审查规则建议：UT不应依赖外部仓库绝对路径；模板特化的类型应与实际使用场景匹配

## 批次20: 提交 #375-#394 (2025-10-31 ~ 2025-10-23)

---

### 7bf5bee CMAKE_BUILD_MODE 调试
- 分类：非缺陷
- 原因：构建系统功能增强(debug/优化参数传递方式改进)

---

### 6aa4c33 解决magicValue整数反转问题
- 根因类别：整数类型错误/溢出
- 涉及文件：mc2/moe_distribute_combine/op_kernel/moe_distribute_combine_a2_layered.h, mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch_a2_layered.h, mc2/moe_distribute_dispatch/op_kernel/moe_distribute_dispatch_a2_layered_aicpu.h
- 缺陷描述：magicValue用于标识Dispatch/Combine算子调用轮次，每次递增1。类型为int32_t，超过INT32_MAX(约21亿次)后有符号溢出，值变负。下游通过magicValue+常量(12345/123/321)作为IPC同步标志区分轮次，溢出后标志值错乱导致核间同步失败，大模型推理输出胡言乱语或为空。
- 修复模式：类型扩宽int32→uint64；有符号改无符号；提取魔术数字为命名常量(GM2IPC_SYNC_FLAG等)。
- 可审查性：中
- 审查规则建议：持续递增的计数器变量应检查类型是否覆盖最大运行周期；IPC/核间同步标志应使用无符号类型

---

### 8dc40bd 修正example编译问题
- 分类：非缺陷
- 原因：文档和example代码修正(注释文件名+CMake示例补充链接库)

---

### 5cd9e41 fix UT of moeinitrouting moetokenpermute 系列算子
- 分类：非缺陷
- 原因：UT适配(更新期望tiling数据+修正路径+更新参数以匹配算子实现变化)

---

### 233466d bugfix_for_gmm_gragh
- 根因类别：边界条件/tiling约束(动态shape未处理)
- 涉及文件：gmm/grouped_matmul/op_host/grouped_matmul_infershape.cpp
- 缺陷描述：GetDim0函数循环累加每个tensor的第0维大小，当shape包含-1(动态/未知维度)时直接参与累加，结果变成无意义负数(多个-1累加得-3)，后续shape推断全部出错。
- 修复模式：遇到任何tensor的dim0为负值时立即返回该负值(传播-1表示"未知")，不再继续累加。
- 可审查性：高
- 审查规则建议：infershape中对维度值的算术操作前应检查是否为-1(动态shape标记)

---

### 8ef7993 修复ut报错
- 根因类别：UT测试代码缺陷(include路径+tiling未初始化+API签名+期望值错误)
- 涉及文件：moe/moe_finalize_routing_v2_grad/tests/ut/(op_host/op_kernel多文件), moe/moe_token_unpermute_with_ep/tests/ut/(多文件), moe/moe_compute_expert_tokens/tests/(新增)
- 缺陷描述：(1) include路径错误导致编译失败。(2) system()调用外部不存在路径。(3) infershape测试NodeOutputTd多传DType参数，API签名错误。(4) kernel测试tiling数据结构未初始化(GmAlloc后无赋值)，读取随机值crash。(5) tiling参数值错误(hidden从262144改为2621)。
- 修复模式：修正路径+删除外部依赖+修正API签名+补充tiling初始化+更新期望值。
- 可审查性：中
- 审查规则建议：GmAlloc后必须显式初始化所有字段；UT不应system()调用外部路径

---

### f1a24bd fix moe_token_unpermute_with_routing_map_grad ut
- 根因类别：UT测试代码缺陷(tiling数据未初始化+GET_TILING_DATA宏错误)
- 涉及文件：moe/moe_token_unpermute_with_routing_map_grad/tests/ut/op_kernel/test文件及头文件
- 缺陷描述：kernel UT通过GmAlloc分配tiling内存后未初始化任何字段，kernel读取随机值行为异常。GET_TILING_DATA宏使用memcpy从uint8_t*拷贝到结构体，存在对齐问题。
- 修复模式：补充21个字段显式初始化；重写宏为__ubuf__指针逐字段赋值。
- 可审查性：中
- 审查规则建议：GET_TILING_DATA宏应统一使用标准模板；tiling内存分配后所有字段必须初始化

---

### 5644a7b fix precision error under super-kernel
- 根因类别：硬件流水线同步缺失(同步ID冲突)
- 涉及文件：attention/mla_prolog/op_kernel/mla_prolog_comm.h
- 缺陷描述：super-kernel场景下MlaProlog算子的同步标志(sync flag)常量值(0x0/0x1)与前序算子同步ID冲突，导致错误同步行为引发精度问题。
- 修复模式：将所有同步常量值提升到0x6/0x7/0x8范围，避免ID冲突。
- 可审查性：低(需了解super-kernel场景所有算子的同步ID分配)
- 审查规则建议：同步ID分配应有全局统一管理机制而非各算子头文件硬编码

---

### b8cad11 PFA、IFA冗余模板删除
- 分类：非缺陷
- 原因：死代码清理(删除4个不可达的.h模板文件)

---

### 601bce1 fix_gmm_add_ut
- 分类：非缺陷
- 原因：UT适配(删除对外部路径的system调用依赖)

---

### b22f886 fix op-host cmake for [interleave_rope/moe_init_routing_v3/moe_re_routing]
- 根因类别：构建配置缺陷
- 涉及文件：posembedding/interleave_rope/op_host/CMakeLists.txt, moe/moe_init_routing_v3/op_host/CMakeLists.txt, moe/moe_re_routing/op_host/CMakeLists.txt
- 缺陷描述：add_modules_sources和add_ops_compile_options在if/else嵌套中被互斥化。"回黄kernel"模式只加编译选项不注册模块源码，"custom"模式反之。特定构建配置组合下编译失败或链接错误。
- 修复模式：提升add_ops_compile_options到BUILD_OPEN_PROJECT内无条件执行；add_modules_sources改为非BUILD_OPS_RTY_KERNEL时独立执行。
- 可审查性：中
- 审查规则建议：CMake条件分支中add_modules_sources和add_ops_compile_options不应互斥

---

### e677067 PFA IFA fix example
- 分类：非缺陷
- 原因：example代码改进(_exit改return)

---

### 4fd755b FIA IFA PFA tilingkey revert
- 根因类别：前序PR引入回归错误(Revert)
- 涉及文件：attention/fused_infer_attention_score/多文件, attention/incre_flash_attention/多文件, attention/prompt_flash_attention/多文件
- 缺陷描述：MR!99的tilingkey模板化整改引入大量模板化tilingkey头文件(4286行+585行+1662行)，导致编译时间爆炸，CI编译超时阻塞线上流水线。
- 修复模式：Revert恢复手动tilingkey计算方式。
- 可审查性：中
- 审查规则建议：模板化代码变更应评估编译时间影响；CI应有编译超时基线监控

---

### d6dd883 tilingkey revert
- 分类：非缺陷
- 原因：空提交(无文件变更，merge master操作)

---

### 6866957 解决部分算子编译告警问题
- 根因类别：参数传递错误(printf格式化说明符不匹配)
- 涉及文件：moe/moe_finalize_routing_v2/op_host/moe_finalize_routing_v2_tiling_arch35.cpp, posembedding/dequant_rope_quant_kvcache/op_host/dequant_rope_quant_kvcache_tiling.cpp, moe/moe_init_routing_v2_grad/op_host/moe_init_routing_v2_grad_tiling.cpp等
- 缺陷描述：多处OP_LOGE/OP_LOGD的printf格式化说明符与参数类型不匹配(%lu用于int64_t, %ld用于uint64_t, %zu用于int64_t等)。在C/C++标准层面构成未定义行为，错误路径触发时输出错误诊断信息。
- 修复模式：修正格式化说明符与参数类型一致。
- 可审查性：高
- 审查规则建议：CI加入-Wformat-security或同等静态分析

---

### b9d7361 修复GMM A8W4 tiling key路由错误
- 根因类别：条件分支遗漏/缺陷(tiling key路由优先级错误)
- 涉及文件：gmm/grouped_matmul/op_host/grouped_matmul_tiling.cpp, grouped_matmul_tiling.h
- 缺陷描述：isA8W4FakeA8W8_的tiling key判断被放在isA8W8_分支之后，且isA8W8_标志被isA8W4FakeA8W8_污染(通过`||`合并)。A8W4请求永远走进A8W8分支，被错误路由到A8W8的tiling key，无法到达TILING_KEY_A8W4_FAKE_A8W8。
- 修复模式：将isA8W4FakeA8W8_判断提到isA8W8_之前(优先匹配更特化路径)；去掉isA8W8_中的`||isA8W4FakeA8W8_`让两个标志独立。
- 可审查性：中
- 审查规则建议：多个互斥条件分支应先匹配更特化路径；布尔标志不应通过||合并不同语义

---

### a8c22c3 新增处理keyAntiquantScale和valueAntiquantScale输入(B, S)的情况
- 根因类别：条件分支遗漏/缺陷(输入shape维度处理遗漏)
- 涉及文件：attention/incre_flash_attention/op_host/incre_flash_attention_tiling.cpp
- 缺陷描述：GetAntiquantSeqLength()中antiquantSIdx只考虑了gqaKvNZFlag_一种情况。当kvAntiParamSplitFlag_=true且antiquantScale tensor为2维(B,S)时，序列长度维度index应为1，但代码默认走到index=2。2维tensor没有index=2，越界访问或返回错误值。
- 修复模式：条件中增加2维shape判断，index=1。
- 可审查性：中
- 审查规则建议：对tensor维度索引访问应校验dimNum是否足够

---

### 4251efa 修复tnd模板gs1合轴精度问题
- 根因类别：计算逻辑错误(向量乘法repeat长度缺乘数因子)
- 涉及文件：attention/prompt_flash_attention/op_kernel/prompt_flash_attention_s1s2_bns1_mla_baseapi.h
- 缺陷描述：MLA kernel中bmm2结果与softmax临时结果做Mul时，repeat count参数应为`vec2S1RealSize * gBaseSize`，但缺少`* gBaseSize`。tnd模板gs1合轴场景下group维度合并到数据维度，缺少乘数导致只处理了部分数据，精度错误。
- 修复模式：Mul的repeat count参数增加`* extraInfo.gBaseSize`。
- 可审查性：中
- 审查规则建议：合轴场景的向量操作repeat/长度参数应包含合轴维度因子

---

### b5ce77b PFA、IFA、FIA - fix warning
- 分类：非缺陷
- 原因：编译告警消除+防御性null检查(代码健壮性改进)

---

### 4aee01d 修正编译问题
- 根因类别：构建配置缺陷(路径依赖+cmake宏调用错误)
- 涉及文件：attention/mla_prolog_v2/op_host/CMakeLists.txt, attention/mla_prolog_v2/op_kernel/mla_prolog_v2.cpp, cmake/obj_func.cmake, cmake/custom_build.cmake
- 缺陷描述：(1) mla_prolog_v2的CMake依赖变量在错误的条件分支设置，开源构建时变量未设置。(2) include路径在不同构建模式下不同导致编译失败。(3) add_library直接调用缺少add_opapi_modules()宏内的必要初始化。(4) 依赖算子kernel文件未安装到目标目录。
- 修复模式：修正条件分支+__has_include条件编译+替换为正确的cmake宏+新增依赖安装逻辑。
- 可审查性：中
- 审查规则建议：跨算子依赖需在cmake/custom_build中同步安装

## 批次21: 提交 #395-#416 (2025-10-22 ~ 2025-09-30)

---

### 98579637 fix warn
- 分类：非缺陷
- 原因：参数名shadow告警消除(doubleBufferFlag→doubleBufferFlagLocal)

---

### 4d24f605 fix moeinitroutingv2 310p
- 根因类别：硬件平台兼容性缺陷(内存对齐+workspace偏移+废弃API)
- 涉及文件：moe/moe_init_routing_v2/op_kernel/moe_v2_sort_multi_core.h, moe_v2_sort_one_core.h
- 缺陷描述：310P平台三类问题。(1) InitGlobalMemory传入长度未按sizeof(int32_t)对齐，310P硬件要求内存操作对齐。(2) workspace中expert索引偏移量使用sortTotalLength计算，与310P上实际每核分配元素数perCoreElements不一致，多核间workspace地址重叠或越界。(3) pipe_barrier(PIPE_ALL)是310P废弃API，需替换为PipeBarrier<PIPE_ALL>()。
- 修复模式：条件编译增加310P平台专用逻辑(Align对齐+独立perCoreOffset+替换API)。
- 可审查性：低
- 审查规则建议：InitGlobalMemory长度参数应经过对齐处理；workspace多核偏移应与tiling的每核元素数一致

---

### c3933b09 weightMatmulAllreduceFixed
- 根因类别：硬件流水线同步缺失(barrier位置不正确)
- 涉及文件：mc2/matmul_all_reduce/op_kernel/common.h
- 缺陷描述：CastBFtoFloatOnAiv0Impl函数末尾做MTE3_V事件同步，但真正需要同步的是循环迭代间——前一次DataCopyPad写出的数据可能与下次迭代向量计算产生RAW冲突。最后一次迭代的同步是无效等待。
- 修复模式：将同步从Impl内部移到调用方循环中，仅在还有后续迭代时插入PipeBarrier<PIPE_ALL>()。
- 可审查性：中
- 审查规则建议：循环调用含DMA操作的函数时，检查迭代间是否正确设置pipeline barrier

---

### fe350c5a ut support infer_datatype/infer_shaperange
- 分类：非缺陷
- 原因：UT框架功能增强(新增infer_datatype/infer_shaperange支持)

---

### b4baa0d2 [SplitCore] Sync bugfix code
- 根因类别：多处逻辑缺陷(无符号整数下溢+边界条件+分核循环无终止保护)
- 涉及文件：attention/common/op_host/split_core.h
- 缺陷描述：SplitCore分核算法多处缺陷(DTS2025092902423)。(1) uint32_t减法下溢：batchLeftCost-=s1GLeftCost等操作被减数<减数时下溢为极大值，分配逻辑错乱。(2) 跳过空batch的while循环与后续bIdx++叠加可能多跳有效batch。(3) ==改>=：索引因跳过逻辑超过边界时无法终止。(4) 分核主循环缺核数上限保护：unassignedCost>0但uint32下溢后curCoreIdx超过coreNum数组越界。(5) coreUse为0时coreUse-1越界。(6) minMaxCost初始值用totalCost不合适。
- 修复模式：安全减法(a>b?a-b:0U)+isComplete状态标记+>=边界判断+核数保护+std::max(coreUse,1U)+UINT32_MAX初始值。
- 可审查性：中
- 审查规则建议：uint32_t减法操作必须检查下溢风险；循环对索引的边界判断优先用>=而非==

---

### ec503a50 fix mem.h InitBuffer
- 根因类别：API变更适配错误(InitBuffer接口废弃后复制粘贴引入新缺陷)
- 涉及文件：common/include/kernel/mem.h
- 缺陷描述：原InitBuffer+手动设logicPos模式改为LocalTensor构造函数。但修复本身引入复制粘贴错误：L0A/L0B/L0C的tensor全部赋值给tensor[ASCEND_CB]索引，后续被64be6dd6修复。
- 修复模式：替换为构造函数直接初始化(但引入了索引错误)。
- 可审查性：高
- 审查规则建议：数组索引与语义标识匹配检查

---

### 4bbc13de fix tilingKey
- 分类：非缺陷
- 原因：新增tilingkey模板文件(纯新增3786行)

---

### 50aaff27 graph修改
- 分类：非缺陷
- 原因：变量命名改善(file→files)，功能无变化

---

### b7ff6fa2 fix_add_2dims
- 根因类别：多处缺陷(buffer复用冲突+变量初始化顺序+UB内存不足+同步缺失)
- 涉及文件：mc2/moe_distribute_combine_add_rms_norm/op_kernel/moe_distribute_combine_add_rms_norm.h, mc2/moe_distribute_combine_v2/op_kernel/moe_distribute_combine_v2.h, mc2/moe_distribute_dispatch_v2/op_kernel/moe_distribute_dispatch_v2.h
- 缺陷描述：(1) 多个LocalTensor从同一TBuf Get不同类型数据导致覆盖(expertScalesBuf_和rowTmpFloatBuf_)。(2) epWorldSize_/moeExpertPerRankNum_只在!hasElasticInfoFlag_分支赋值，但后续无条件使用。(3) 总buffer超限未降级为单buffer。(4) DataCopyPad后缺SyncFunc同步。(5) WaitDispatch中对同一tensor先Duplicate再DataCopy存在竞态。
- 修复模式：新增独立buffer解除复用+提前赋值+动态降级+补同步+引入stateResetBuf消除竞态。
- 可审查性：低
- 审查规则建议：同一TBuf被Get为多种类型tensor时标记复用冲突；条件分支内赋值的变量不得在分支外无条件使用

---

### 214c47f5 modify GQA antiquant 8 buffer and fix bugs for gqa antiquant pertoken
- 根因类别：多处缺陷(参数校验缺失+循环逻辑错误+buffer大小不足)
- 涉及文件：attention/fused_infer_attention_score/op_host/fused_infer_attention_score_def.cpp, attention/incre_flash_attention/op_host/incre_flash_attention_tiling.cpp, attention/incre_flash_attention/op_kernel/incre_flash_attention_preload_dd.h
- 缺陷描述：(1) SetupPerfMode中gqaKvNZFlag_分支return导致后续ropeFlag_不可达(dead code)。(2) CopyInMm1AToL1中msdIterNum循环与地址偏移计算错误(mIter*copyIterNum+i应为i)。(3) CopyInMm1BToL1ForPA中baseN=128/sizeof(KV_T)应为256/sizeof(KV_T)，拷贝宽度不足。(4) GQA KV NZ场景缺少antiquantMode参数校验和维度校验。
- 修复模式：if-else链式判断修复dead code+循环重构+baseN修正+校验补全。
- 可审查性：低
- 审查规则建议：if分支中的return导致后续代码不可达应被静态分析标记

---

### 6cef41c6 fix ut bug
- 根因类别：UT测试代码缺陷(cmake配置+宏参数名typo+链接库缺失)
- 涉及文件：cmake/config.cmake, cmake/func_utest.cmake, cmake/ut.cmake, tests/ut/framework_normal/common/(多文件), tests/ut/framework_normal/op_api/CMakeLists.txt
- 缺陷描述：(1) ASCEND_OP_NAME的逗号→分号替换在条件分支内部，其他分支未执行导致多算子名list操作失败。(2) DO_TILING宏参数名tilingPara与宏体内tilingContextPara不一致，编译错误。(3) ut.cmake缺少json/gtest链接库。(4) socVersion硬编码Ascend910_95改为可配置。
- 修复模式：cmake配置提前执行+宏参数名修正+补全链接库+参数化。
- 可审查性：中
- 审查规则建议：C++宏定义形参名必须与宏体内使用的变量名一致

---

### 9c461b25 mc2单词拼写错误修改
- 分类：非缺陷
- 原因：纯拼写修正(AllGahter→AllGather, Exector→Executor等34个文件)

---

### 0708f156 build_lib支持-j, --jit支持自定义算子
- 分类：非缺陷
- 原因：新feature(构建脚本功能增强)

---

### 1a22fcb9 secure_c使用新版本，protobuf使用gitcode链接
- 分类：非缺陷
- 原因：第三方依赖升级和镜像切换

---

### 60a57476 修正custom包名
- 根因类别：构建配置缺陷(包命名格式错误)
- 涉及文件：cmake/custom_build.cmake
- 缺陷描述：自定义算子包文件名`-linux.${ARCH}`不符合CANN命名规范，应为`_linux-${ARCH}`。下游安装脚本或部署流程无法正确识别。
- 修复模式：修正命名格式为_linux-。
- 可审查性：高
- 审查规则建议：CPACK_PACKAGE_FILE_NAME应通过正则校验符合标准模式

---

### bd4cc701 修复example找不到头文件的错误
- 分类：非缺陷
- 原因：example编译脚本完善(补充include路径和链接库)

---

### 9c6d1284 fix interleave_rope,moe_gating_top_k,moe_re_routing,moe_init_routing_v3
- 根因类别：多类缺陷(格式化字符串不匹配+变量未赋值+平台信息获取错误+shadowing)
- 涉及文件：moe/moe_gating_top_k/op_host/(tiling.cpp, tiling_arch35.cpp), moe/moe_re_routing/op_host/moe_re_routing_tiling_base.cpp, moe/moe_init_routing_v3/op_host/(infershape.cpp, tiling.cpp, tiling_base.cpp, op_api/aclnn.cpp), posembedding/interleave_rope/op_host/interleave_rope_tiling.h
- 缺陷描述：(1) OP_LOGE中%lu用于int64_t等格式化说明符不匹配(UB)。(2) moe_init_routing_v3 infershape中quantMode声明后从未从attrs获取值(初始为-1)，InferDataType逻辑永远走错分支。(3) tiling中平台信息在错误阶段获取。(4) aclnn中不必要的V2回退逻辑导致错误路由。(5) 1e-20应为1e-20f。(6) 构造函数参数名shadow成员变量。
- 修复模式：格式符修正+补充attrs读取+平台信息获取前移+删除回退逻辑+加f后缀+改参数名。
- 可审查性：低(多个不相关缺陷混在一个提交)
- 审查规则建议：启用-Wformat和-Wshadow；infershape中从attrs获取的属性值应检查"声明后是否赋值再使用"

---

### 5c38f9c5 fa compile waring fix
- 根因类别：构建配置缺陷(参数shadow在-Werror下编译失败)
- 涉及文件：attention/flash_attention_score/op_host/flash_attention_score_tiling.cpp, attention/flash_attention_score_grad/op_kernel/flash_attention_score_grad_tiling.h
- 缺陷描述：setter函数参数名与类成员变量同名触发-Wshadow告警，在-Werror编译选项下导致编译失败。
- 修复模式：参数重命名加_val后缀+[[maybe_unused]]标注。
- 可审查性：高(但6000+行格式化diff掩盖实际修改)
- 审查规则建议：编译时启用-Wshadow -Werror；格式化变更应与逻辑变更分离提交

---

### 7d1f015a ut support noexec
- 根因类别：构建脚本参数解析缺陷
- 涉及文件：build.sh, cmake/func_utest.cmake
- 缺陷描述：(1) build.sh中--noexec分支缺少shift，后续参数全部错位。(2) cmake中变量名UT_NO_EXEC与build.sh传入的ENABLE_UT_EXEC不一致，noexec功能完全失效。
- 修复模式：补充shift+统一变量名为ENABLE_UT_EXEC。
- 可审查性：高
- 审查规则建议：shell脚本参数解析每个case必须包含shift；CMake -D变量名应与脚本传入名一致

---

### 7ed8fea3 -j命令取消空格
- 根因类别：构建脚本参数解析缺陷
- 涉及文件：build.sh
- 缺陷描述：-j参数只匹配精确字符串-j(空格分隔形式)，不支持-j8(紧跟数字)形式。用户输入-j8时匹配不到，并行编译数未被设置。
- 修复模式：匹配模式从-j改为-j*通配+提取数字部分。
- 可审查性：高
- 审查规则建议：带值的短选项应同时支持紧跟和空格两种形式

---

### f314395a aclnnGroupedMatmulFinalizeRoutingV3修复文档和报错信息
- 根因类别：输入校验缺失(多芯片场景参数校验未解耦+维度一致性校验缺失)
- 涉及文件：gmm/grouped_matmul_finalize_routing/op_host/op_api/aclnn_grouped_matmul_finalize_routing.cpp, aclnn_grouped_matmul_finalize_routing_MX_checker.h, docs/aclnnGroupedMatmulFinalizeRoutingV3.md
- 缺陷描述：(1) 910/95芯片路径多余CheckFormat调用，在不适用芯片上做错误format校验。(2) DAV_3510芯片weight非NZ格式处理逻辑不正确。(3) CheckDimRange未按芯片区分。(4) weight类型检查逻辑在非DAV_3510芯片上报错信息不准确。(5) 缺少x和weight的k维一致性前置校验。(6) weight/weightScale非ND场景无format校验。
- 修复模式：按芯片架构解耦校验+增加前置一致性校验+将format校验移入芯片专用checker。
- 可审查性：中
- 审查规则建议：多芯片校验应在架构层面解耦；维度一致性校验应作为shape check前置步骤

---

### ef8ac892 修复aclnn文档与头文件参数不匹配的问题
- 分类：非缺陷
- 原因：纯文档修复(const aclrtStream声明与头文件对齐)
