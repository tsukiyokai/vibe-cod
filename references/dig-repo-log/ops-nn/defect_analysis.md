# ops-nn 缺陷提交深度分析

## 第1轮 (commit 1-20)

### d91e1757ab41c3620057477c28adc62d395c2e00 revert//uss算子迁移
- 根因类别：功能缺陷导致回退（新特性引入问题）
- 涉及文件：38个文件，涵盖 index/unsorted_segment_sum/ 下的全部实现（tiling、kernel、host、proto、文档、示例），以及 docs/zh/op_list.md
- 缺陷描述：原始commit 1962c3991（新增unsorted_segment_sum算子支持Ascend950）被整体revert。该MR在合入unsorted_segment_sum算子的同时，在op_list.md中错误地新增了一个scatter算子的条目（而非unsorted_segment_sum的条目），属于文档修改错误。此外从revert行为来看，原始提交可能存在编译/集成问题导致紧急回退。
- 修复模式：完整revert原始commit，删除所有38个新增/修改的文件
- 可审查性：中
- 审查规则建议：新增算子时检查op_list.md中新增条目是否与实际算子名称一致；大规模特性合入前确保CI全流程通过

### 21127847202d552b85c4b62eefbb81219897ffcd depthwise preload 遗留问题修正
- 根因类别：模式适配不全 / 条件判断逻辑错误
- 涉及文件：conv_api_tiling_algorithm_HWmode.cpp/h, conv_api_tiling_algorithm_base.cpp/h, conv_common_func.h, conv_iterate_impl.h, conv2d_v2_base_tiling_tilingkey.cpp, conv2d_v2_instr_base_impl.h, conv2d_v2_instr_impl.h, convolution.cpp（共10个文件）
- 缺陷描述：depthwise preload功能此前仅适配了M模式，未适配HW模式。具体包括：(1) InferWiL1方法定义在HWmode子类中，基类无法使用，需上移；(2) ResetOptGroupDoubleBuffer中AL1 size计算使用固定的cubeInfo.m0/k0，未区分M/HW模式下不同tiling参数；(3) CheckOptGroupPreload中otherFlag强制要求outputOrder==M，排除了HW模式，multiMFlag未处理HW模式条件，错误排除了FLOAT8/HIFLOAT8/INT8/UINT8数据类型；(4) kernel侧SetLoad3dFMatrixForOptPreload硬编码M模式参数传入方式；(5) padLeftL1/padRightL1/wiLoadL1定义为子类私有成员，基类无法访问；(6) bias空间计算对齐参数错误
- 修复模式：将InferWiL1从子类上移到基类；按M/HW模式分支计算AL1 size；去掉M模式限制和数据类型限制，增加HW模式条件；将私有成员移到基类；SetLoad3dFMatrixForOptPreload改为无参按outputOrder分支处理
- 可审查性：中
- 审查规则建议：新增特性时检查是否覆盖了所有执行模式分支（M模式/HW模式/不同数据类型）

### afb09c7856c63e1158c681bf96f2fda127b70ecf fix dynamic_mx_quant & map_index tiling
- 根因类别：多核同步模式配置遗漏
- 涉及文件：index/map_index/op_host/map_index_tiling_arch35.cpp, quant/dynamic_mx_quant/op_host/arch35/dynamic_mx_quant_tiling_arch35.cpp
- 缺陷描述：两个算子的tiling代码中缺少SetScheduleMode(1)调用。map_index完全没有设置schedule mode。dynamic_mx_quant在blockSizeNumInAxis为奇数时多核存在数据依赖但未设置batch mode同步。
- 修复模式：在tiling函数中增加context->SetScheduleMode(1)设置batch mode
- 可审查性：高
- 审查规则建议：所有tiling函数中检查是否正确设置了SetScheduleMode，特别是多核场景下的同步模式

### 7b4a1b53c3f564330e91daa974a182a32d2be48a 修复aclnnNanMedian空指针校验
- 根因类别：错误处理宏参数误用（return遗漏）
- 涉及文件：index/gather_v2/op_api/aclnn_median.cpp
- 缺陷描述：OP_CHECK宏的第三个参数（失败时的动作）写成了裸值nullptr而非return nullptr。OP_CHECK在条件不满足时执行第三参作为失败动作，但原代码写的是, nullptr)，意味着失败时不会return而继续向下执行使用空指针。共14处OP_CHECK调用存在此问题。
- 修复模式：将14处OP_CHECK(..., nullptr)统一改为OP_CHECK(..., return nullptr)
- 可审查性：高
- 审查规则建议：OP_CHECK宏的第三个参数必须包含return语句；可用静态分析检测, nullptr)模式

### eda6863e4c1d7aa0d4420215ccd21703e4bea4d4 aclnnIndexCopy增加空指针校验
- 根因类别：空指针校验缺失
- 涉及文件：index/scatter_update/op_host/op_api/aclnn_index_copy.cpp
- 缺陷描述：ExecIndexCopyGetWorkspaceSize函数中，多个l0op算子调用（Contiguous、Reshape、ScatterUpdate、TransposeBySpecifiedAxis、ViewCopy）的返回值使用CHECK_RET宏校验（多线程不安全），且Reshape后的结果在scalar输入路径下可能为nullptr但无校验，TransposeBySpecifiedAxis返回值也无空指针检查。
- 修复模式：将CHECK_RET替换为OP_CHECK并附带return和错误日志；将Contiguous+Reshape+校验抽取到独立函数；对所有算子返回值增加OP_CHECK空指针检查
- 可审查性：高
- 审查规则建议：所有l0op/算子调用返回值必须进行空指针校验，且校验失败时必须return错误码；多线程场景优先使用线程安全的错误处理宏

### ce96300c400aff997559ffccaad74d3936c07069 修复codecheck告警问题
- 根因类别：变量遮蔽 + 函数声明与定义不一致
- 涉及文件：norm/add_rms_norm_cast/op_host/add_rms_norm_cast_tiling_arch35.cpp, norm/add_rms_norm_quant/op_host/add_rms_norm_quant_tiling.h, norm/add_rms_norm_quant/op_host/add_rms_norm_quant_tiling_arch35.cpp, norm/add_rms_norm_quant/op_kernel/arch35/add_rms_norm_quant_regbase_perf.h
- 缺陷描述：(1) 头文件中CalFullLoadBaseM函数声明参数名为baseM，但实际定义中语义是baseN，声明与定义不一致。(2) 局部变量tmpPower与外层同名变量产生遮蔽。(3) 多处多余空行。
- 修复模式：修正函数声明参数名；重命名局部变量为tmpPowerSize消除遮蔽；删除多余空行
- 可审查性：高
- 审查规则建议：启用编译器-Wshadow告警检测变量遮蔽；CI集成codecheck自动拦截声明/定义不一致

### fe7d6ee3136d22b358f9011940125ba59b928902 修复 mat_mul_v3 kernel代码中的错误注释说明
- 根因类别：注释与代码逻辑不一致
- 涉及文件：matmul/mat_mul_v3/op_kernel/mat_mul_deterministic_splitk_kernel.h
- 缺陷描述：funcParamsMK（preload参数值为2）注释写成"preload左矩阵"，funcParamsNK（preload参数值为1）注释写成"preload右矩阵"，实际参数值2表示N方向preload，1表示M方向preload，注释方向完全相反。
- 修复模式：纯注释修正
- 可审查性：低
- 审查规则建议：对关键算法参数使用枚举或命名常量代替magic number以减少注释歧义

### 6b52fdcf2fb7b85c53a56d4f71e72b27444d1513 FixViewShape
- 根因类别：张量格式校验逻辑错误（FORMAT_ND vs FORMAT_NCL）+ shape获取方式错误（StorageShape vs ViewShape）
- 涉及文件：rnn/single_layer_lstm_grad/op_host/op_api/aclnn_lstm_backward.cpp, 及对应examples/tests/docs
- 缺陷描述：(1) CheckFormatValid中所有张量统一使用FORMAT_ND校验，但隐藏状态h/c、门控张量i/j/f/o等实际格式为FORMAT_NCL。(2) ValidateInputShape使用GetStorageShape()获取shape校验，但非连续张量的StorageShape与ViewShape不同，导致合法输入被错误拦截。
- 修复模式：修正format校验逻辑按张量实际格式区分NCL和ND；将GetStorageShape()改为GetViewShape()
- 可审查性：中
- 审查规则建议：format校验检查是否所有张量被强制同一格式而未考虑实际差异；shape校验优先使用GetViewShape()，仅在需要物理布局时使用GetStorageShape()

### 0253078d581f0a93fb3c83ce4dd02f02372f1533 fix scatter_elements_v2 VF计算偏移时数据类型统一
- 根因类别：数据类型不一致导致计算错误（int32 vs int64隐式转换）
- 涉及文件：index/scatter_elements_v2/op_host/scatter_elements_v2_asc_tiling.cpp, index/scatter_elements_v2/op_kernel/scatter_elements.h
- 缺陷描述：(1) Tiling层GetTilingKey判断int64路径时仅检查allAxis_>MAX_INT32_NUM，遗漏dataAxis_和updatesAxis_检查。(2) Kernel层SimtComputeDim2到Dim8中stride数组声明为uint64_t而非模板参数COMP_T，COMP_T为int32时发生不必要的类型提升导致VF性能退化。
- 修复模式：扩展溢出判断覆盖所有axis；将stride数组类型从uint64改为COMP_T
- 可审查性：中
- 审查规则建议：使用模板参数COMP_T的函数中检查是否存在与COMP_T不一致的局部变量类型；tiling key数值范围判断应覆盖所有相关维度

### e9bed79c4ec2a0866e1cc8f4a78721e8c40dd9e1 修复MxA8W4 shape校验错误
- 根因类别：StorageShape vs ViewShape混用 + 错误日志不准确 + 条件分支遗漏
- 涉及文件：matmul/quant_batch_matmul_v3/op_api/aclnn_quant_matmul_v4.cpp, matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：(1) MicroScaling分支直接用GetViewShape().GetDim()而未使用已提取的变量，非连续tensor校验不一致。(2) 错误日志期望shape写groupDimK但实际校验groupDimK/2，日志误导调试。(3) MxScaleContiguousProcess调用条件缺少isA8W4Float分支，A8W4场景下scale未被正确转连续。
- 修复模式：统一使用已提取的ViewShape维度变量；修正错误日志期望值；补充A8W4条件分支
- 可审查性：中
- 审查规则建议：OP_LOGE中format参数值应与校验条件严格一致；新增数据类型时检查所有条件分支是否同步更新

### dc01cd867b83ae6a690507cc9f7068e9840ac169 Atlas推理系列pertoken量化模式下batch>1精度修复
- 根因类别：多维偏移量计算错误 + 多核配置错误
- 涉及文件：quant_batch_matmul_v3_tiling_arch20.cpp, quant_batch_matmul_v3_kernel_tiling_data.h, quant_batch_matmul_v3_pertoken_arch20.h
- 缺陷描述：三个独立问题。(1) tiling层hardcode SetBlockDim(1)，batch>1时未使用多核。(2) bias偏移量计算未区分bias是否带batch维度——固定使用b_idx*n_+n_idx*n0_，但bias shape为[n]（无batch维）时偏移计算错误。(3) scale的GM偏移量错误乘了b_idx*n_，scale是pertensor/perchannel级别不应随batch变化。另外将GetInputDesc改为GetOptionalInputDesc以正确处理可选输入。
- 修复模式：修正偏移量计算公式 + 增加biasWithBatch维度信息传递 + 修正多核blockDim配置
- 可审查性：中
- 审查规则建议：检查SetBlockDim是否使用硬编码1；可选输入使用GetOptionalInputDesc；GM偏移量出现batch_idx乘法时验证tensor是否确实有batch维度

### 15e40a4867a2f2d7c1b17fbb1e7f136057c489f7 修复addmm
- 根因类别：未初始化变量 + 空指针解引用风险
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp, matmul/common/op_host/op_api/matmul_util.h
- 缺陷描述：(1) PromoteResult结构体的logMessage指针未初始化，某些GetUpperDtypeByLookUpTable返回路径中为野指针，后续无条件OP_LOGE("%s", result.logMessage)导致崩溃。(2) TensorInfo结构体的tensor指针、dataType、format也未初始化。
- 修复模式：结构体成员默认初始化 + 使用前判空
- 可审查性：高
- 审查规则建议：C++结构体指针成员必须初始化为nullptr；所有成员变量应在声明处初始化；OP_LOGE中%s前必须判空

### 910dac63b4adf28dd9a9a6b98e9f85b23b194e2a fix bug for QuantUpdateScatter & optimize compile time
- 根因类别：地址偏移量计算公式错误
- 涉及文件：quant_update_scatter_large_batch_little_quant_regbase.h（核心bug），及7个文件（编译优化）
- 缺陷描述：gmVarOffset_计算公式错误。原式(axisOffset + innerLoopIdx) * innerLoopEle将两个语义不同的变量相加后统一乘innerLoopEle，但axisOffset应乘varDim3（axis维度步长），innerLoopIdx应乘innerLoopEle。
- 修复模式：拆分为actualBsIdx * dstBsStride + axisOffset * varDim3 + innerLoopIdx * innerLoopEle
- 可审查性：低
- 审查规则建议：多维数组偏移量计算应逐维展开，每个维度索引乘以对应步长，避免将不同维度索引合并为一个表达式

### 941d3c7f88b8040a0e9da984a5947390c9e06791 修复tbmm的warning报错信息
- 根因类别：有符号/无符号比较类型不匹配
- 涉及文件：transpose_batch_mat_mul_base_tiling.cpp
- 缺陷描述：常量kSupportedInnerAxis声明为uint64_t，但与int64_t类型的shape维度值比较，编译器产生有符号/无符号比较warning。
- 修复模式：将类型从uint64_t改为int64_t
- 可审查性：高
- 审查规则建议：开启-Wsign-compare并视为错误；与shape维度比较的常量应声明为int64_t

### a810b9c0a7f3418dd32b91221c8f457283c228c1 applyTopkTopP算子950性能劣化修复及TopP分支增加至少保留一个值机制
- 根因类别：平台适配缺失（核数配置） + 功能遗漏（保底值机制）
- 涉及文件：apply_top_k_top_p_with_sorted_tiling.cpp, apply_top_p_with_sorted.h
- 缺陷描述：(1) 950平台核数多于A2，单TopP分支scatter搬出操作数量不均匀导致性能回退。(2) 单TopP分支缺少"至少保留一个值"的保底机制，当所有cumsum值未达阈值p时输出可能为全零。
- 修复模式：950平台特判限制核数上限为48；新增CopyOutLastValue函数保底机制
- 可审查性：中
- 审查规则建议：新增算子对照标准参考实现检查边界case；多平台算子审查核数分配策略是否考虑不同soc版本

### c8ca6becf24497e9499339d741bce760bd2e19de 解决aclnnAdaptiveMaxPool3d索引值类型为int64时出现的类型不匹配
- 根因类别：接口参数缺失/默认值不当
- 涉及文件：pooling/adaptive_max_pool3d/op_api/aclnn_adaptive_max_pool3d.cpp
- 缺陷描述：调用MaxPool3DWithArgmaxV2Ncdhw接口时未传入indices的实际数据类型（indicesDtype），接口默认使用int32，但当用户传入int64时底层计算与输出tensor类型不匹配。
- 修复模式：Regbase模式下从indices tensor获取真实dtype并显式传递
- 可审查性：高
- 审查规则建议：调用具有默认参数的接口时，检查实际数据类型是否可能与默认值不一致，涉及dtype参数应显式传递

### 8583f0f3b8ca33c3ffd6eba1cf0f6e258ac4106b quantbatchmatmul a8w4kernel修复，修复workspace偏移，修复ub多申请了大小
- 根因类别：硬编码常量错误 + sizeof语义误用
- 涉及文件：matmul/quant_batch_matmul_v4/op_kernel/quant_batch_matmul_v4_msd.h
- 缺陷描述：(1) workspace偏移使用硬编码MM_BASE_BLOCK_OFFSET=32768，实际应随baseM*baseN动态变化。(2) UB buffer申请用alignKSize_*sizeof(int4b_t)，但sizeof(int4b_t)返回1（非预期的0.5），导致int4类型buffer多申请一倍。
- 修复模式：硬编码替换为动态值baseN_*baseM_；用CeilDiv(alignKSize_, INT4_SIZE)替代sizeof误用
- 可审查性：中
- 审查规则建议：检测内存分配/偏移相关硬编码魔数；sub-byte类型(int4)的sizeof调用需特别审查

### 5da80998bd992fe9ca241b54589ee7373fe1eef2 fix scatter_list aclnn & repeat_inter_leave_grad warning
- 根因类别：配置错误 + printf格式符不匹配
- 涉及文件：index/repeat_interleave_grad/op_host/repeat_interleave_grad_int_repeat_tiling.cpp, index/scatter_list/op_host/CMakeLists.txt
- 缺陷描述：(1) 日志用%ld输出uint64_t类型的CACHE_BUF_SIZE，应为%lu。(2) scatter_list的CMakeLists.txt中ACLNNTYPE配置为aclnn_inner，实际应为aclnn。
- 修复模式：%ld改为%lu；aclnn_inner改为aclnn
- 可审查性：高
- 审查规则建议：CI门禁拦截printf格式符不匹配；CMakeLists.txt中ACLNNTYPE审查是否符合接口设计意图

### 67c665fd267c68937ddb872c9f54e0d97e1f4580 avg_pool_v2_grad 原型提交及infershape漏洞修复
- 根因类别：逻辑运算符错误 + 输出shape未赋值
- 涉及文件：pooling/avg_pool_v2_grad/op_graph/avg_pool_v2_grad_proto.h, pooling/avg_pool_v2_grad/op_host/avg_pool_v2_grad_infershape.cpp
- 缺陷描述：(1) 维度校验条件 inputDimNum != CHW_DIMS || inputDimNum != NCHW_DIMS 使用||（逻辑或），恒为true，合法输入也被拒绝，应改为&&。(2) infershape设置了输出shape的维度数SetDimNum但从未对每个维度赋值SetDim，输出shape未初始化。
- 修复模式：||改为&&修正逻辑；新增循环从输入数据读取shape值并写入输出
- 可审查性：高
- 审查规则建议：!= A || != B模式（恒真表达式）应作为静态检查规则；infershape中SetDimNum后必须有对应SetDim赋值

### 8f737553ce98e0ecab841fcf3d4c2b6dd50e90aa tiling性能优化 fmap可全载实际未全载问题修复
- 根因类别：条件判断逻辑错误导致分支策略不优
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_api_tiling_algorithm_BBmode.cpp
- 缺陷描述：L1缓存加载策略函数将fmap+weight全载、fmap全载都置于batch全载条件之下，多核场景batch不全载时fmap即使可全载也无法进入对应分支，退化为weight全载路径。
- 修复模式：重构条件分支：fmap+weight全载作为最高优先级无条件判断；按迭代顺序分支调整fmap全载与weight全载优先级
- 可审查性：低
- 审查规则建议：tiling策略多维度交叉判断时审查前置条件是否过于严格；性能相关分支变更应配套性能基准测试

## 第2轮 (commit 21-40)

### a71c47f707d42e9b5cec4c39a26d89d3780be2ac 修复TBMM算子资料scale的维度错误
- 根因类别：文档错误
- 涉及文件：matmul/transpose_batch_mat_mul/docs/aclnnTransposeBatchMatMul.md
- 缺陷描述：scale参数维度描述为2维，实际tbmm的scale只支持1维。文档表格中维度值从2修正为1。
- 修复模式：纯文档修正，修改md表格中的维度值
- 可审查性：高
- 审查规则建议：算子文档中参数维度描述应与代码中shape校验逻辑一致

### de7c68326010644a7a4998fe73e6e72ca15cc51c 修复matmulv3减少编译耗时导致后冒烟打断问题
- 根因类别：C++模板类型作用域错误（using声明在if-else块内无效）
- 涉及文件：matmul/mat_mul_v3/op_kernel/mat_mul_deterministic_splitk_kernel.h, mat_mul_unaligned_deterministic_splitk_kernel.h
- 缺陷描述：原代码试图通过if-else分支内的using声明来选择不同模板类型（aType/bType），但C++中块作用域内的using声明不会影响块外的后续代码。导致所有分支都使用外层默认的NZ格式类型，isNzA/isNzB为false时仍使用NZ格式模板参数调用MatMul函数，引发精度问题。这是之前一个编译耗时优化PR引入的回退。
- 修复模式：将类型选择和函数调用合并到每个if-else分支内部，确保每个分支使用正确的模板类型参数直接调用函数
- 可审查性：高
- 审查规则建议：if/else块内using类型别名声明不会传播到块外，检测此模式应作为静态审查规则；模板类型选择逻辑必须与调用点在同一作用域

### 07e77ddd07a00a13e0e71f8efa6aca8e6abe0d79 修复aclnnInplaceAddmm接口走入gemmv3算子时不支持输入转置的问题
- 根因类别：算子属性索引偏移错误（不同算子的attr排列不同）
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp
- 缺陷描述：GetShape函数通过固定索引GetAttrPointer<bool>(0)和GetAttrPointer<bool>(1)读取transposeX1/transposeX2属性。但GemmV3算子的attr顺序为[alpha, beta, transposeX1, transposeX2, ...]，transpose属性在索引2和3处而非0和1。当aclnnInplaceAddmm走入GemmV3路径时，读到的是alpha/beta而非transpose标志，导致转置信息丢失。
- 修复模式：增加NodeType判断，GemmV3取index 2/3，其他取index 0/1
- 可审查性：高
- 审查规则建议：通过索引获取算子属性时，必须确认不同算子类型的attr排列顺序是否一致；优先使用属性名而非硬编码索引获取attr

### fe95ffda590b30a132877ac88378ca60d0a1d10a 删除classify_rule.yaml中不存在的文件路径
- 根因类别：构建配置错误（引用已废弃算子路径）
- 涉及文件：classify_rule.yaml
- 缺陷描述：vfusion-c模块的unrelease->test_code配置中引用了norm_rope_concat和norm_rope_concat_grad的tests/examples路径，但这两个算子已废弃不再维护，实际路径不存在，造成构建干扰。
- 修复模式：从yaml配置中删除4条不存在的路径
- 可审查性：高
- 审查规则建议：yaml/cmake配置中引用的文件路径应在CI中校验是否实际存在

### f24f9dba9dba3d980a8e6f2fdd7022e7981adc54 fix: infershape ut failed
- 根因类别：构建配置错误（重复符号定义）
- 涉及文件：cmake/ut.cmake, tests/ut/op_host/CMakeLists.txt, 2个ut文件（空行变更）
- 缺陷描述：opbase的源码被tiling和infershape的UT target分别编译链接，导致同一符号在链接时重复定义。此外opbase_util_objs/opbase_infer_objs/opbase_tiling_objs直接嵌入到多个target的$<TARGET_OBJECTS>中导致冲突。
- 修复模式：新增add_opbase_ut_common()函数将opbase编译为公共的static library（opbase_ut_common），tiling和infershape的UT通过target_link_libraries引用，避免重复符号
- 可审查性：中
- 审查规则建议：CMake中同一组object files不应通过TARGET_OBJECTS嵌入多个target，应封装为独立static library

### 04e45bfc0b99f30ea6a11123163043b51795aad0 aclnnBatchMatMulWeightNz.md 确定性说明fix
- 根因类别：文档错误
- 涉及文件：matmul/batch_mat_mul_v3/docs/aclnnBatchMatMulWeightNz.md
- 缺陷描述：确定性说明中列出了特定产品系列（Atlas训练/推理系列），但引入了不支持的版本，需改为通用说明。同时缺少950D的确定性说明。
- 修复模式：删除特定产品系列限定，改为通用描述
- 可审查性：高
- 审查规则建议：文档中涉及产品系列/版本兼容性的描述应与实际代码支持矩阵一致

### 640a1683e4f647a4ddf8d745abf4fd463a80c320 fix: ascendc depend
- 根因类别：构建配置错误（CMake target缺少前置依赖）
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：ascendc_impl_gen target生成py文件时，其依赖的ops_info_gen_*等target尚未构建完成。原代码先创建ascendc_impl_gen target再通过foreach生成其依赖target并add_dependencies，但custom_command的DEPENDS参数中未包含这些前置target。且add_ops_impl_target函数未接受DEPENDS参数。
- 修复模式：(1) 函数签名增加DEPENDS参数传递到custom_command。(2) 将ascendc_impl_gen target的创建移到foreach循环之后，确保依赖target已定义。(3) 将依赖列表通过DEPENDS参数传入custom_command
- 可审查性：中
- 审查规则建议：CMake custom_command/custom_target的DEPENDS必须包含所有输入依赖；target创建顺序应在其依赖target之后

### 4f594168b090538019bac42de84424f294f5950f 增加空指针校验
- 根因类别：空指针校验缺失
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp, matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp
- 缺陷描述：aclnn_addbmm和aclnn_matmul中，ConvertToTensor、AllocScalar、AllocIntArray、Fill、ReduceSumOp、SetTensorToNZFormat等关键操作的返回值未校验，在内存分配失败时会产生空指针解引用。共新增约13处CHECK_RET校验。
- 修复模式：在每个可能返回空指针的操作后增加CHECK_RET(ptr != nullptr, error_code)
- 可审查性：高
- 审查规则建议：所有返回指针的API调用（ConvertToTensor/AllocScalar/AllocIntArray/Fill/ReduceSumOp等）后必须有空指针校验

### a8c4157ac9910b74ba82cead877c7d3fc59da847 fix: 修复UT编译失败
- 根因类别：构建配置错误（opbase对象链接缺失 + 变量传递遗漏）
- 涉及文件：build.sh, cmake/ut.cmake, cmake/variables.cmake, tests/ut/op_host/CMakeLists.txt
- 缺陷描述：(1) ut.cmake中optiling_ut和infershape_ut的static_lib缺少opbase_util_objs/opbase_tiling_objs/opbase_infer_objs依赖。(2) variables.cmake中硬编码CANN_3RD_LIB_PATH默认值被删除后，build.sh未将该变量传递到cmake命令行。(3) tests/ut/op_host/CMakeLists.txt中opbase对象被从顶层链接移除后下层未补充。
- 修复模式：ut.cmake中补充opbase对象到static_lib；build.sh增加-DCANN_3RD_LIB_PATH传递；调整CMakeLists.txt链接
- 可审查性：中
- 审查规则建议：删除默认值或移除依赖时，检查所有引用点是否同步更新；构建变量的传递链完整性

### b3aeb54fe71b06376c6e09c29b3098ca9c01a029 fix dynamic quant copyout bug
- 根因类别：数据搬运尾块越界（固定大小搬运 vs 实际大小搬运）
- 涉及文件：quant/dynamic_quant/op_kernel/dynamic_quant_unalign_310p.h, dynamic_quant_unalign_large_310p.h
- 缺陷描述：dynamic_quant算子在310p后端copyout阶段，尾块搬运时使用固定的numCopyRow_长度而非实际剩余行数。当输入不对齐时，尾块实际数据量小于numCopyRow_，但DataCopy按numCopyRow_搬运，越界写入踩踏后续内存。
- 修复模式：(1) 新增CopyOutUnalign方法，尾块搬运时按实际calCount计算realNumRowAlign。(2) isTailLoop_分支中用realNumRowAlign替代固定numCopyRow_
- 可审查性：中
- 审查规则建议：DataCopy的长度参数在尾块处理时必须使用实际剩余数据量而非固定值；循环+尾块模式下审查尾块搬运长度

### 6feeae0a83c340973f4db5c2e78ffffb95ad7267 修改EmbeddingDenseGradV2 idxOffset翻转导致indices取值越界导致aic
- 根因类别：数组越界访问 + 条件判断逻辑不完善
- 涉及文件：index/embedding_dense_grad_v2/op_kernel/v35/embedding_dense_grad_v2_regbase.h
- 缺陷描述：(1) 循环中idxOffset在每次迭代累加interval后可能超过indices数组大小，但无边界检查，下次循环indices(idxOffset)越界导致AIC错误。(2) 负索引判断idx < 0与idx >= numWeights分开写，但idx为有符号类型转uint64_t比较时，负数会变成极大正数自然>=numWeights，原来的idx < 0检查冗余且不如统一用unsigned比较直接。
- 修复模式：循环内增加idxOffset >= indices.GetSize()的边界检查及break；将idx < 0 || idx >= numWeights合并为static_cast<uint64_t>(idx) >= numWeights
- 可审查性：高
- 审查规则建议：循环中递增的索引变量访问数组前必须检查边界；有符号索引与无符号上界比较应统一为unsigned比较

### 0700b4fdee8bb96a4cbfba9d4b42365d752a1d00 fix: compile with opbase source
- 根因类别：构建配置重构（从库依赖改为源码编译）
- 涉及文件：CMakeLists.txt, cmake/func.cmake, 及多个cmake文件
- 缺陷描述：opbase之前作为外部库依赖（ops_base_util_objs/ops_base_infer_objs），现改为下载源码本地编译。原有target名不一致（ops_base_*改为opbase_*），函数名/链接关系需同步更新。
- 修复模式：新增add_opbase_modules()函数管理opbase源码编译；统一target名称；调整include/link依赖
- 可审查性：中
- 审查规则建议：重命名target时全局搜索确认所有引用点同步更新

### 07bec2430e0fcc03bedabc846849e3802929e8dd Fix repeat_interleave kernel102 functional issue
- 根因类别：计数器累加位置错误导致数组越界
- 涉及文件：index/repeat_interleave/op_kernel/arch35/repeat_interleave.h
- 缺陷描述：copyFromXNum_在CopyOneCpToRepeatOut函数末尾按dataCount累加，但实际应在外层CopyXToMatchOut循环中按mergedDims[2]累加。在CopyOneCpToRepeatOut内累加会导致尾块处理时copyFromXNum_超过实际数据量，后续访问越界。
- 修复模式：将copyFromXNum_ += dataCount从CopyOneCpToRepeatOut移除，在CopyXToMatchOut的循环体末尾改为copyFromXNum_ += tilingData_.mergedDims[2]
- 可审查性：中
- 审查规则建议：跨函数共享的计数器/偏移量变量，其累加位置应与实际数据消费粒度匹配；嵌套函数调用中避免在底层函数更新上层状态

### 22b4d33516f496aa833cac3b21921f131cf1eaf4 fix aclnnprelubackward bug
- 根因类别：Shape引用错误（使用中间转换后的shape而非原始输入shape）
- 涉及文件：activation/p_relu_grad_update/op_api/aclnn_prelu_backward.cpp
- 缺陷描述：PReLU backward中gradInput需要reshape到与原始self相同的shape。但代码使用contiguousSelf->GetViewShape()，而contiguousSelf是经过内部Contiguous/Reshape转换后的tensor，其shape可能与原始输入不同（如1维张量被扩展为多维）。reshape使用错误的shape导致输出与预期不符。
- 修复模式：新增originalSelfShape参数，在reshape时使用原始self的shape而非转换后的contiguousSelf的shape
- 可审查性：中
- 审查规则建议：Contiguous/Reshape操作后的tensor shape与原始输入shape可能不同，需要保持原始shape时应另存变量

### 8f6ccaea4116bed1b4a62f6541561c7215780587 addlayernormgrad上边界问题修复
- 根因类别：整数溢出（uint32不足以表示大shape计算结果）
- 涉及文件：norm/add_layer_norm_grad/op_host/add_layer_norm_grad_tiling.cpp/h, op_kernel/add_layer_norm_grad_cut_d.h, tests/ut
- 缺陷描述：roundUpNumLastDimFloat字段声明为uint32_t，ROUND_UP宏计算numLastDim对齐后乘sizeof(float)，当numLastDim超过2^31-1（如2147483649）时乘法溢出。同理deterministicWorkSpaceSize也声明为uint32_t不够。
- 修复模式：将roundUpNumLastDimFloat和deterministicWorkSpaceSize从uint32_t改为uint64_t；ROUND_UP前先static_cast<uint64_t>避免中间溢出
- 可审查性：高
- 审查规则建议：涉及shape维度乘法的变量必须使用int64_t/uint64_t；ROUND_UP宏的输入在乘法前检查类型是否足够

### 0958e639537066becf8556011f3305adfdeccd8e fix: 同一台机器同时编译kernel导致卡死
- 根因类别：构建脚本错误（shell重定向导致多进程死锁）
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：custom_target命令末尾追加echo $(MAKE) &> /dev/null，&>在某些shell环境下将echo和后续MAKE命令的输出都重定向到/dev/null，多进程同时编译时可能因MAKE变量展开和文件描述符竞争导致卡死。
- 修复模式：去掉&> /dev/null重定向，改为直接echo $(MAKE)
- 可审查性：中
- 审查规则建议：CMake custom_command中避免shell重定向到/dev/null，特别是涉及$(MAKE)变量展开的场景

### 3cf703d7e1247de3abf87eeea099abbd0e3477aa aclnn_add_rms_norm_dynamic_quantv2 fix int4 support pta
- 根因类别：条件判断缺失（outputMask为空时对空输出执行操作）
- 涉及文件：norm/add_rms_norm_dynamic_quant/op_host/op_api/aclnn_add_rms_norm_dynamic_quant_v2.cpp
- 缺陷描述：int4类型输出分支中，当存在outputMask且某个输出被mask掉时（PTA传入空指针），代码仍无条件对该输出执行AddRmsNormDynamicQuantV2Int42Int32PackedTensor和ViewCopy操作，访问空指针导致内部接口校验失败。
- 修复模式：增加outputMask判断，processOut1/processOut2根据outputMask决定是否执行对应输出的计算和拷贝
- 可审查性：高
- 审查规则建议：带outputMask参数的算子，所有输出处理分支必须检查对应mask位；空指针输出不应参与任何计算

### c1924396cd30039bbf7d7d657c677bbf0bac49fe fix noaicpu option
- 根因类别：构建脚本参数命名不一致
- 涉及文件：build.sh
- 缺陷描述：build.sh中长选项定义为no_aicpu（带下划线），但其他ops仓库统一使用noaicpu（不带下划线），导致跨仓库一致性问题。
- 修复模式：将no_aicpu改为noaicpu
- 可审查性：高
- 审查规则建议：构建参数命名应与其他仓库保持一致；新增构建选项时检查已有仓库的命名规范

### bba9de57ca6c64633393810a44397ad3c6d53fcd FixLstmBackwardMd
- 根因类别：文档错误（参数描述笔误、shape说明不准确、示例代码语法错误）
- 涉及文件：rnn/single_layer_lstm_grad/docs/aclnnLstmBackward.md
- 缺陷描述：(1) input参数shape描述未说明与batchSizesOptional的关联。(2) params中bias描述未区分bias_ih和bias_hh。(3) "政协"笔误应为"正向"。(4) 示例代码多余分号。
- 修复模式：修正shape说明、参数描述、笔误、语法错误
- 可审查性：高
- 审查规则建议：API文档中参数shape描述应覆盖所有输入组合条件；使用spell check工具

### 40f9a0d42541dfe69feb821514661e67e84338e0 fix conv warning
- 根因类别：代码格式问题（多余空行产生warning）
- 涉及文件：conv/convolution_forward/op_host/op_api/aclnn_quant_convolution.cpp
- 缺陷描述：函数间多余空行导致编译warning。
- 修复模式：删除多余空行
- 可审查性：高
- 审查规则建议：启用clang-format自动格式化

## 第3轮 (commit 41-60)

### 84a249a2 fix sizeof(T) to sizeof(float) in test demo
- 根因类别：测试代码内存分配错误
- 涉及文件：test demo中的内存分配代码
- 缺陷描述：测试demo中使用sizeof(T)分配内存，但实际数据类型应为float，模板参数T与实际类型不匹配导致分配大小错误。
- 修复模式：将sizeof(T)替换为sizeof(float)
- 可审查性：高
- 审查规则建议：sizeof参数应与实际使用的数据类型一致，模板代码中尤其注意类型参数是否被正确使用

### 52f9ba38 fix float implicit conversion to int64 + UT coverage
- 根因类别：隐式类型转换
- 涉及文件：算子host代码 + UT测试
- 缺陷描述：float值隐式转换为int64_t，可能导致精度丢失或未定义行为。同时UT覆盖不足未能发现此问题。
- 修复模式：添加static_cast显式转换 + 补充UT用例
- 可审查性：高
- 审查规则建议：编译器warning(-Wfloat-conversion)应作为error处理；CR时关注浮点到整型的隐式转换

### 1ca42282 fix eigen 5.0.0 download URL line endings
- 根因类别：构建配置错误
- 涉及文件：eigen依赖下载配置
- 缺陷描述：eigen 5.0.0下载URL因行尾换行符问题导致下载失败。
- 修复模式：修正URL字符串中的换行符
- 可审查性：高
- 审查规则建议：构建脚本中的URL字符串应避免跨行拼接，或使用strip处理

### ea9b02ef fix document format (cubeMathType term tag)
- 根因类别：文档格式错误
- 涉及文件：cubeMathType相关文档
- 缺陷描述：文档中cubeMathType使用了`<term>`标签，渲染异常。
- 修复模式：移除`<term>`标签
- 可审查性：高
- 审查规则建议：文档中避免使用可能被解析器误解的HTML/XML标签

### 8cef5510 fix multi-core index offset missing in sort operator
- 根因类别：多核并行索引偏移缺失(严重)
- 涉及文件：sort算子的simd_sort和simt_sort路径
- 缺陷描述：indicesOffset计算缺少`eachCoreIndexCount * GetBlockIdx()`项。在多核场景下，每个核输出的indices没有加上该核对应的全局偏移量，导致所有核输出的索引都从0开始而非各自的全局起始位置。
- 修复模式：在indicesOffset计算中加入`eachCoreIndexCount * GetBlockIdx()`
- 可审查性：中 — 需要理解多核分片逻辑
- 审查规则建议：多核算子中所有输出索引/偏移量必须包含GetBlockIdx()相关的全局偏移；多核场景必须有多核UT验证索引正确性

### 49bbb988 fix scatter_elements_v2 warnings
- 根因类别：编译warning(类型转换+未使用变量)
- 涉及文件：scatter_elements_v2算子代码
- 缺陷描述：size_t到int64_t的隐式转换warning、未使用变量warning、变量命名不规范。
- 修复模式：类型转换显式化、删除未使用变量、重命名变量
- 可审查性：高
- 审查规则建议：CI中开启-Wall -Werror；定期清理unused variable

### cce7a445 fix L1 bank conflict in buffer layout
- 根因类别：L1 bank冲突导致性能下降(严重)
- 涉及文件：矩阵运算kernel代码
- 缺陷描述：L1缓冲区布局为|A0|B1|...|B0|A1|，A和B的交错排列导致同bank访问冲突。
- 修复模式：将布局调整为|A0|B0|...|A1|B1|，使A/B连续排列避免bank冲突
- 可审查性：低 — 需要深入理解硬件bank结构
- 审查规则建议：L1 buffer分配时A/B矩阵应连续排列，避免交错；性能敏感kernel应做bank冲突检测

### 0a2fb0d7 fix RNN unused parameter + DFX_OUT macro missing parameter
- 根因类别：接口参数遗漏
- 涉及文件：RNN算子代码
- 缺陷描述：1) RNN中存在未使用参数；2) DFX_OUT宏调用缺少outputMask参数，导致调试输出信息不完整。
- 修复模式：删除未使用参数 + 补充DFX_OUT宏的outputMask参数
- 可审查性：高
- 审查规则建议：宏调用参数个数必须与宏定义一致；使用编译器warning检测未使用参数

### 28ab6d70 fix StorageShape to ViewShape in thnn_fused_lstm_cell_backward
- 根因类别：StorageShape/ViewShape混淆(严重)
- 涉及文件：thnn_fused_lstm_cell_backward算子host代码
- 缺陷描述：使用StorageShape获取tensor维度信息，但该tensor可能经过view操作，StorageShape反映的是底层存储形状而非逻辑视图形状，导致shape信息错误。
- 修复模式：将StorageShape替换为ViewShape
- 可审查性：中 — 需要理解Storage vs View语义
- 审查规则建议：获取tensor shape时默认使用ViewShape，仅在确认需要底层存储形状时才使用StorageShape

### f31988fa fix code-check issues (parameter name mismatch, unused header, implicit bool)
- 根因类别：代码规范问题(多项)
- 涉及文件：多个算子文件
- 缺陷描述：1) 函数声明与定义的参数名不一致；2) 包含未使用的头文件；3) 隐式bool比较。
- 修复模式：统一参数名 + 删除未使用头文件 + 显式bool比较
- 可审查性：高
- 审查规则建议：clang-tidy规则覆盖参数名一致性检查和隐式bool转换检查

### dc62e8dd fix CreateView null pointer check missing in batch_matmul
- 根因类别：空指针检查缺失(严重)
- 涉及文件：batch_matmul_util.cpp (3处)
- 缺陷描述：CreateView返回值未做null检查就直接使用，如果创建失败会导致空指针解引用崩溃。
- 修复模式：在3处CreateView调用后添加null检查
- 可审查性：高
- 审查规则建议：所有Create*/New*类工厂函数返回值必须做null检查后再使用

### d196e790 fix QuantMatmulWeightNz document format
- 根因类别：文档格式错误
- 涉及文件：QuantMatmulWeightNz算子文档
- 缺陷描述：文档格式问题影响渲染。
- 修复模式：修正文档格式
- 可审查性：高
- 审查规则建议：文档发布前预览渲染效果

### eb818ca6 fix GetAttrPointer int32_t should be int64_t for ignoreIndex
- 根因类别：属性类型不匹配(严重)
- 涉及文件：算子host代码
- 缺陷描述：使用GetAttrPointer<int32_t>获取ignoreIndex属性，但该属性实际类型为int64_t。类型不匹配导致只读取了64位值的低32位，高32位被截断。
- 修复模式：将GetAttrPointer<int32_t>改为GetAttrPointer<int64_t>
- 可审查性：高
- 审查规则建议：GetAttrPointer的模板类型参数必须与算子注册时声明的属性类型严格一致；建立属性类型映射表供CR参考

### 6eee5478 fix tiling key parameter error + SetAtomicNone after return
- 根因类别：tiling参数错误 + 同步位置错误(严重)
- 涉及文件：算子tiling代码 + kernel代码
- 缺陷描述：1) tiling key参数传递错误导致选择了错误的tiling策略；2) 在single-batch路径中SetAtomicNone被放在return语句之后，永远不会被执行，导致原子操作模式未被正确重置。
- 修复模式：修正tiling key参数 + 将SetAtomicNone移到return之前
- 可审查性：中
- 审查规则建议：return语句后不应有可执行代码(dead code检测)；tiling key参数应有枚举约束而非裸字符串

### 852f21d6 revert batchmatmul non-contiguous input support
- 根因类别：功能回退(feature引入问题)
- 涉及文件：batchmatmul算子代码
- 缺陷描述：之前添加的non-contiguous input支持引入了问题，需要整体回退。
- 修复模式：revert整个feature
- 可审查性：N/A — 回退操作
- 审查规则建议：大feature合入前应有充分的集成测试覆盖；考虑feature flag控制新功能上线

### edac4382 replace magic numbers with named constants
- 根因类别：代码可维护性(magic number)
- 涉及文件：算子代码
- 缺陷描述：代码中使用magic number，可读性差且容易出错。
- 修复模式：定义命名常量替换magic number
- 可审查性：高
- 审查规则建议：禁止裸数字常量(0/1/-1除外)，必须使用命名常量

### ed0bd5a1 fix QBMMv4 WaitFlag position wrong + A8W8GB tiling split missing
- 根因类别：同步时序错误 + tiling逻辑缺失(严重)
- 涉及文件：QBMMv4 kernel代码 + A8W8GB tiling代码
- 缺陷描述：1) WaitFlag位置错误 — 放在了SetFlag之前而非之后，导致等待的是上一轮的flag而非当前轮，数据可能未就绪就被使用；2) A8W8GB场景的tiling分片逻辑缺失，导致大shape场景无法正确分片。
- 修复模式：将WaitFlag移到SetFlag之后、数据使用之前 + 补充A8W8GB tiling split逻辑
- 可审查性：低 — SetFlag/WaitFlag时序需要理解AscendC流水线模型
- 审查规则建议：SetFlag/WaitFlag必须成对出现且顺序为Set→Wait→Use；tiling代码必须覆盖所有量化模式

### 67995b31 fix mmv2 Nz kernel bin not found (unstable nAxis value)
- 根因类别：变量值不稳定导致kernel匹配失败(严重)
- 涉及文件：mmv2算子aclnn层代码
- 缺陷描述：使用mat2的nAxis值来匹配kernel bin，但该值在aclnn处理过程中会被修改(如transpose/reshape操作改变axis含义)，导致后续查找kernel bin时使用了错误的值，找不到对应kernel。
- 修复模式：替换为稳定的mmOpInfo.shapeInfo.nDim值，该值在算子信息构建阶段就已确定且不会再变化
- 可审查性：低 — 需要理解aclnn处理流水线中tensor属性的变化
- 审查规则建议：kernel bin匹配参数应使用处理链早期确定的稳定值，避免使用中间过程中可能被修改的tensor属性

### 479d15b8 fix document link + classify_rule.yaml path cleanup
- 根因类别：文档链接错误 + 配置路径错误
- 涉及文件：文档文件 + classify_rule.yaml
- 缺陷描述：文档中的链接指向错误地址；classify_rule.yaml中的路径不正确。
- 修复模式：修正链接和路径
- 可审查性：高
- 审查规则建议：文档CI中加入链接有效性检查(dead link checker)

### a1cc187b fix EmbeddingDenseGradV2 idx upper bound check missing
- 根因类别：数组越界检查缺失(严重)
- 涉及文件：EmbeddingDenseGradV2算子kernel代码
- 缺陷描述：对idx只检查了`idx < 0`的下界，缺少`idx >= numWeights`的上界检查。当idx超出权重表大小时会导致越界访问。
- 修复模式：添加`idx >= numWeights`上界检查条件
- 可审查性：高
- 审查规则建议：数组/表索引校验必须同时包含上下界检查；Embedding类算子的index参数必须做range validation

## 第4轮 (commit 61-80)

### 92982b89 空tensor校验逻辑缺失 (yOffset)
- 根因类别：空tensor与空指针语义混淆(严重)
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：yOffset不为nullptr但IsEmpty()为true时（空tensor），仍对其做shape校验导致误报参数非法。空tensor应视为"无此输入"跳过校验。
- 修复模式：条件从`yOffset != nullptr`改为`yOffset != nullptr && !yOffset->IsEmpty()`
- 可审查性：中 — 需要理解空tensor语义
- 审查规则建议：可选tensor参数的null检查应同时考虑IsEmpty()状态；空tensor等价于空指针应作为编码规范

### a9e29824 mxfp8全载tiling中scaleFactor整除为0
- 根因类别：整除结果为零未防护(严重)
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/adaptive_sliding_window_tiling.cpp
- 缺陷描述：scaleFactor通过整除计算得到，当k,n较大时被除数小于除数导致结果为0，后续运算链中0值传播导致精度失败。
- 修复模式：增加`scaleFactorBMax != 0 && scaleFactorBBase != 0`保护条件；重组条件分支确保被除数>=除数时才使用优化搬运策略
- 可审查性：中
- 审查规则建议：整除运算结果必须做零值防护；tiling计算中的除法应有被除数<除数的边界处理

### 9141d377 fix code-check (conv3d_v2多项)
- 根因类别：代码规范问题(多项)
- 涉及文件：conv/conv3d_v2/多个头文件和实现文件
- 缺陷描述：1) 函数声明与定义参数名不一致(SetDilation/SetStride/SetHF32等)；2) CheckAlgorithmLimit/CheckBiasShape缺少const修饰；3) using namespace在头文件中；4) 未使用的头文件；5) C风格强转改static_cast；6) 变量命名不规范(aoeTiling→convRepoTiling)
- 修复模式：统一参数名 + 添加const + 删除using namespace + 删除未使用include + 规范命名
- 可审查性：高
- 审查规则建议：头文件禁止using namespace；函数声明和定义参数名必须一致

### 9c28c24d fix avgpoolv2grad int32溢出检查不完整
- 根因类别：溢出检查范围不足(严重)
- 涉及文件：pooling/avg_pool_v2_grad/op_host/arch35/avg_pool_v2_grad_tiling_base.cpp
- 缺陷描述：IsGreaterThanInt32Max函数仅检查H*W是否超int32上限，实际应检查batch*channels*H*W总大小。当batch或channels较大时，即使H*W未溢出，总元素数仍可能超int32。
- 修复模式：将`H*W > INT32_MAX`改为`batch*channels*H*W > INT32_MAX`
- 可审查性：高
- 审查规则建议：int32溢出检查应覆盖所有参与计算的维度，不能仅检查部分维度

### bf8e8ab5 打包脚本tar格式选择逻辑错误
- 根因类别：条件逻辑错误
- 涉及文件：scripts/package/common/py/packer.py
- 缺陷描述：原逻辑只要检测到bsdtar就使用ustar格式，但bsdtar不支持长文件名(>100字符)。应优先使用gtar(GNU tar)，仅在无gtar且有bsdtar时才退化到ustar格式。
- 修复模式：改为`if not gtar and bsdtar: tar_format = "ustar"`
- 可审查性：高
- 审查规则建议：条件优先级应与工具能力匹配；打包脚本应测试长文件名场景

### 5a5241b1 文档格式修复 (avgpool/lstm)
- 根因类别：文档格式错误
- 涉及文件：pooling/avg_pool3_d_grad/docs, rnn/single_layer_lstm_grad/docs
- 缺陷描述：avgpool文档产品名称缺少term标签；lstm文档details标签前缺少空行导致渲染异常。
- 修复模式：添加term标签 + 补充空行
- 可审查性：高
- 审查规则建议：Markdown中HTML标签前后需要空行以确保正确渲染

### a715c50a tiling文件位置错误导致编译错误版本
- 根因类别：构建配置/文件路径错误(严重)
- 涉及文件：embedding_dense_grad, gather_nd的tiling相关文件 + 文档
- 缺陷描述：embedding_dense_grad和gather_nd的tiling文件放置位置错误，编译系统找不到正确版本的tiling文件，导致多个用例执行失败。
- 修复模式：修正tiling文件路径 + 更新文档中的产品支持描述
- 可审查性：中
- 审查规则建议：算子tiling文件路径应遵循统一的目录规范；CI应验证tiling文件能被正确编译链接

### 8b2d85b4 matmul类算子文档链接失效
- 根因类别：文档链接错误
- 涉及文件：matmul下多个算子的README.md
- 缺陷描述：文档中链接使用URL编码(%26)导致Markdown渲染失败，应使用原始字符(&)。多个matmul算子README受影响。
- 修复模式：将%26替换为& + 修正错误的示例路径引用
- 可审查性：高
- 审查规则建议：Markdown文件中链接不应使用URL编码；文档CI应验证内部链接可达性

### ca565c0c 36核精度失败(return应为continue)
- 根因类别：控制流语句错误(严重)
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/conv3d_dx_rowc_block.h
- 缺陷描述：多核循环中当核索引超出有效范围时使用`return`直接退出整个函数，但实际应使用`continue`跳过当前迭代继续下一轮循环。在36核环境下，部分核被错误跳过导致输出不完整、精度失败。
- 修复模式：将`return`改为`continue`
- 可审查性：高
- 审查规则建议：循环体内的return语句需仔细审查是否应该是continue/break；多核循环中跳过无效核应用continue而非return

### c63abf32 aclnnRmsNormQuant输出shape校验不完整
- 根因类别：shape校验维度覆盖不全(严重)
- 涉及文件：norm/rms_norm_quant/op_host/op_api/aclnn_rms_norm_quant.cpp + 文档
- 缺陷描述：原有aclnn代码仅检查int32/int4类型输出的最后一维是否满足条件，缺少对前N-1个维度的shape校验。输入x和输出y的前面维度不一致时不会报错，导致运行时可能出现内存越界。
- 修复模式：新增CheckShapeDimWithFrontN函数，在INT32/INT4/其他dtype三个分支中分别添加前N维度的shape一致性校验
- 可审查性：中
- 审查规则建议：输出tensor的shape校验必须覆盖所有维度，不能仅校验尾维度

### 95c8dbac fix Third_Party_Open_Source_Software_List.yaml
- 根因类别：配置文件维护错误
- 涉及文件：Third_Party_Open_Source_Software_List.yaml
- 缺陷描述：三方依赖名称nlohmann/json应为json；缺少protobuf依赖声明。
- 修复模式：修正名称 + 添加protobuf条目
- 可审查性：高
- 审查规则建议：三方依赖清单应与实际使用的依赖保持同步更新

### 501d38ab 文档示例代码重复导致编译错误
- 根因类别：文档示例代码错误
- 涉及文件：matmul/weight_quant_batch_matmul_v2/docs/aclnnWeightQuantBatchMatmulV2.md
- 缺陷描述：A16W4示例代码中有重复的函数定义(AclnnWeightQuantBatchMatmulV2Test)，用户拷贝后编译会出现重定义错误。
- 修复模式：删除重复的代码片段(~112行)
- 可审查性：高
- 审查规则建议：文档示例代码应在CI中进行编译验证

### a7c5a771 回退QuantBatchMatmulV3 A矩阵全载+尾块切分
- 根因类别：功能回退(feature影响其他场景性能)
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/adaptive_sliding_window_tiling.cpp
- 缺陷描述：PR1762添加的A矩阵全载时尾块切分逻辑增加了`mBlockCnt == 1`限制条件，但该修改影响了950平台其他模板(非纯cube和mx模板)的A全载性能。
- 修复模式：回退整个条件限制，恢复原始的尾块切分逻辑
- 可审查性：N/A — 回退操作
- 审查规则建议：tiling策略修改需在所有受影响的平台和模板上做性能回归测试

### a7630f9f fix TopKTopPSample SetScheduleMode缺失
- 根因类别：多核调度模式未设置(严重)
- 涉及文件：index/top_k_top_p_sample/op_host, index/top_k_top_p_sample_v2/op_host
- 缺陷描述：TopKTopPSample/V2算子使用SyncAll进行核间同步，但未设置SetScheduleMode为BATCH_MODE(1)。缺少此设置可能导致核间同步行为不正确。
- 修复模式：在isNeedLogits/isNeedSampleResult条件下添加`context->SetScheduleMode(BATCH_MODE)`
- 可审查性：中
- 审查规则建议：使用SyncAll的算子必须设置SetScheduleMode为BATCH_MODE；CR checklist应包含同步原语与调度模式的匹配检查

### 2e1ef7e1 aclnnIndexAddV2 UT assert过时
- 根因类别：UT未跟随功能更新
- 涉及文件：index/inplace_scatter_add/tests/ut/op_host/test_aclnn_index_add_v2.cpp
- 缺陷描述：aclnnIndexAddV2接口功能扩展后已支持新的数据类型组合，但UT中期望仍是旧的错误码(ACLNN_ERR_INNER_NULLPTR)，实际应返回ACL_SUCCESS。原UT还被注释掉了。
- 修复模式：取消注释 + 将期望值改为ACL_SUCCESS
- 可审查性：高
- 审查规则建议：接口功能扩展时必须同步更新相关UT的期望值；UT中不应有被注释掉的断言

### 644f498b 文档demo路径错误 (extend_conv2d/quant_conv3d)
- 根因类别：文档路径错误
- 涉及文件：conv/extend_conv2d/README.md, conv/quant_conv3d/README.md
- 缺陷描述：README中demo路径指向`./examples/`但实际文件在`./examples/arch35/`下，导致链接失效。
- 修复模式：路径中添加arch35子目录
- 可审查性：高
- 审查规则建议：文档中的相对路径引用应在CI中验证文件存在性

### 853dcb95 fix codecheck of batch_norm_v3
- 根因类别：代码规范问题(C风格强转+参数名不一致)
- 涉及文件：norm/batch_norm_v3/op_host/多个文件
- 缺陷描述：1) 使用C风格强转`(int64_t)x`而非`static_cast<int64_t>(x)`；2) 函数声明参数名(theLeastAPerCore)与定义(aFactor)不一致。
- 修复模式：改为static_cast + 统一参数名
- 可审查性：高
- 审查规则建议：禁止C风格类型转换(使用clang-tidy google-readability-casting规则)

### 2045ef0a Conv3DTransposeV2 kernel split全载wi=2时AIC ERROR
- 根因类别：资源释放无效地址(严重)
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/conv3d_backprop_input_v2_kernel_split_tiling.cpp
- 缺陷描述：kernel split全载且wi=2场景下，存在单计算轮次中B1Tensor全部跳过无需加载的情况，但全载最后仍尝试释放B1Tensor地址，该地址无效导致AIC ERROR。
- 修复模式：在IsBaseShapeFitKernelSplitHW中增加`kSCoutFullLoad_ && wi <= 2`条件返回false，屏蔽该场景不走kernel拆分
- 可审查性：低 — 需要理解conv kernel split的buffer管理机制
- 审查规则建议：全载模式下跳过加载的buffer不应在最后被释放；kernel split的边界条件应有专门的UT覆盖

### 4482238c revert logsigmoid impl
- 根因类别：功能回退
- 涉及文件：activation/log_sigmoid/下全部文件(删除)
- 缺陷描述：logsigmoid算子实现被完整回退，删除了proto、tiling、kernel等所有文件。
- 修复模式：revert整个feature
- 可审查性：N/A — 回退操作
- 审查规则建议：新算子合入前应通过完整的集成测试验证

### f84fb52d indexputv2确定性分支broadcast函数调用错误
- 根因类别：函数调用错误(严重)
- 涉及文件：index/index_put_v2/op_api/aclnn_index_put_impl.cpp
- 缺陷描述：确定性分支中调用了IndicesBroadcast函数，但该函数是非确定性版本。确定性分支应调用IndicesBroadcastUndeter，两者在scalar tensor的broadcast处理逻辑上不同，导致进入broadcastto的infershape时报错。
- 修复模式：将IndicesBroadcast替换为IndicesBroadcastUndeter
- 可审查性：中 — 需要理解确定性/非确定性分支的差异
- 审查规则建议：确定性分支中只应调用带Undeter后缀的函数；代码审查时关注确定性/非确定性路径是否使用了正确的函数变体

## 第5轮 (commit 81-100)

### 6f8ad87c 头文件兼容性适配 (kernel_basic_intf.h)
- 根因类别：版本兼容性缺失(严重)
- 涉及文件：common/inc/op_kernel/platform.h + matmul多个kernel头文件
- 缺陷描述：`#include "kernel_basic_intf.h"`在CANN 8.5及以下版本中不存在，导致编译失败。需要通过宏`ASC_DEVKIT_MAJOR >= 9`条件编译，低版本退回`kernel_operator.h`。
- 修复模式：在所有include处添加`#if ASC_DEVKIT_MAJOR >= 9`条件编译
- 可审查性：高
- 审查规则建议：引入新SDK头文件时必须评估向后兼容性；使用条件编译保护非通用头文件

### 7fa2b4eb embedding_bag README列宽修复+链接失效
- 根因类别：文档错误
- 涉及文件：embedding_bag的README.md
- 缺陷描述：README表格列宽不当导致显示异常 + 文档链接失效。
- 修复模式：调整列宽 + 修正链接
- 可审查性：高
- 审查规则建议：文档CI中加入链接有效性检查

### 34217db7 Conv3DBackpropInputV2 cin超datacopy stride上限
- 根因类别：硬件指令参数上限校验缺失(严重)
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/conv3d_backprop_input_v2_inner_product_tiling.cpp
- 缺陷描述：当cin大于65535时，前置transpose的datacopy指令stride参数超过16位上限(65535)，导致精度失败。tiling未检查cin是否超出硬件指令限制。
- 修复模式：在CheckVecTransEnable中增加`runInfo_.dedx_cin > 65535`条件，超限时禁用前置transpose
- 可审查性：中 — 需要了解datacopy指令的参数上限
- 审查规则建议：使用datacopy的stride参数前必须校验不超过65535(16位上限)；tiling决策应考虑硬件指令参数范围

### c04e08dc DLQBMM算子日志缺少右括号
- 根因类别：日志格式错误
- 涉及文件：matmul/dual_level_quant_batch_matmul/op_host/op_tiling/dual_level_quant_batch_matmul_checker.cpp
- 缺陷描述：ToShapeString函数构造shape字符串时缺少闭合的']'字符，导致异常拦截时维测信息格式不完整。
- 修复模式：添加`shapeStr.push_back(']')`
- 可审查性：高
- 审查规则建议：字符串构造函数应确保配对符号(括号/方括号)完整

### f9e38e90 GroupNormSiluQuant文档格式修复
- 根因类别：文档格式错误
- 涉及文件：GroupNormSiluQuant算子文档
- 缺陷描述：文档中存在格式问题影响渲染。
- 修复模式：修正格式
- 可审查性：高
- 审查规则建议：文档发布前预览渲染效果

### adec83fb CI脚本shell语法错误导致编译失败不报错
- 根因类别：shell语法错误(严重)
- 涉及文件：scripts/ci/check_example.sh, check_kernel_ut.sh, check_pkg.sh (4处)
- 缺陷描述：`[ $status -ne 0]`缺少']'前的空格，bash语法错误导致条件判断失效。编译/UT执行失败时，CI流水线仍显示成功，掩盖了实际错误。
- 修复模式：改为`[ $status -ne 0 ]`(添加空格)
- 可审查性：高
- 审查规则建议：CI脚本应使用shellcheck静态分析；`[`和`]`前后必须有空格

### 305ad94a Embeddingbag paddingIdx重复处理导致精度失败
- 根因类别：上下游处理逻辑重复(严重)
- 涉及文件：index/embedding_bag/op_host/embedding_bag_regbase_tiling.cpp
- 缺陷描述：上层torch框架已将paddingIdx从负值转换为正值(如-1→numEmbeddings-1)，tiling中再次做`paddingIdx + numEmbeddings`导致索引超出范围，精度失败。
- 修复模式：删除tiling中重复的paddingIdx负值处理逻辑
- 可审查性：中 — 需要了解上下游职责边界
- 审查规则建议：算子内部不应重复上层框架已完成的参数预处理；paddingIdx语义应在接口文档中明确标注是否已预处理

### 7b870377 Conv3DTransposeV2 B矩阵全载场景释放无效地址
- 根因类别：资源释放无效地址(严重)
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/conv3d_bp_kernel_split_func.h
- 缺陷描述：B矩阵全载场景下，整轮计算都未加载B1Tensor但FreeB1Tensor仍尝试释放，导致释放无效地址引发AIC ERROR。与commit #78(wi=2场景)属同一类问题的不同触发路径。
- 修复模式：在FreeB1Tensor中增加`isLoadB1_ && !isFreeB1_`条件，未加载则跳过释放
- 可审查性：低 — 需要理解全载场景buffer生命周期
- 审查规则建议：Free操作前必须验证对应资源已被成功分配/加载；buffer释放逻辑应与加载逻辑对称

### f4ad97f3 conv nDim对齐计算与API侧不一致
- 根因类别：kernel与host侧对齐计算不一致(严重)
- 涉及文件：conv/common/op_kernel/arch35/conv_common.h + conv2d_v2多个kernel文件 + conv3d_v2 tiling
- 缺陷描述：kernel侧计算nDim时，N0对齐的方式与API侧不一致。原CalcNDimDataWeightNZ函数对齐逻辑错误(先CeilDiv再AlignB)，应该是对wholeDim先AlignB再CeilDiv，保证L1数据读取不会错位。
- 修复模式：重写为CalcNDimDataAlign函数，统一对齐逻辑：`AlignB(CeilDiv(AlignB(wholeDim, N0), dim), N0)`
- 可审查性：低 — 需要理解conv多核分片的对齐要求
- 审查规则建议：kernel与host的维度对齐计算必须使用相同公式；对齐计算应抽取为公共函数统一维护

### 79089b0e quant_conv3d dequant flag异常(biasType占位类型错误)
- 根因类别：占位类型选择不当(严重)
- 涉及文件：conv/quant_conv3d/op_kernel/quant_conv3d.cpp
- 缺陷描述：当DTYPE_BIAS未定义时(无bias场景)，biasType使用`half`作为编译占位类型，但half类型影响了dequant flag的判断逻辑，导致异常行为。
- 修复模式：将占位类型从`half`改为`int32_t`，消除对dequant flag的干扰
- 可审查性：中
- 审查规则建议：编译占位类型不应影响运行时逻辑判断；占位类型应选择与正常路径一致的默认类型

### 7e28fed7 QBMMv4文档错误修复
- 根因类别：文档错误
- 涉及文件：QBMMv4算子文档
- 缺陷描述：文档内容有误。
- 修复模式：修正文档
- 可审查性：高
- 审查规则建议：算子文档应由开发和测试双方审核

### 4d315436 aclnnQuantBatchMatmulV4 k0默认值条件反转
- 根因类别：条件判断逻辑反转(严重)
- 涉及文件：matmul/quant_batch_matmul_v3/op_api/aclnn_quant_matmul_v4.cpp (2处)
- 缺陷描述：加入int4类型后，k0值选择条件从`DT_INT8 ? INT8_K0 : INT4_K0`变为了默认走int4路径。正确逻辑应为`DT_INT4 ? INT4_K0 : INT8_K0`(int4是特殊分支，其他类型走int8默认值)。
- 修复模式：将条件从`== DT_INT8`反转为`== DT_INT4`
- 可审查性：高
- 审查规则建议：新增类型分支时，默认分支(else)应保持原有行为不变；三元表达式的条件应使"特殊情况"作为true分支

### e79d67ac AddLayerNormQuant bias为null时binary匹配失败
- 根因类别：simplified_key生成错误(严重)
- 涉及文件：norm/add_layer_norm_quant/op_host/add_layer_norm_quant_tiling.cpp + binary.json
- 缺陷描述：生成simplified_key时，bias字段直接使用x1的dtype代替(硬编码)，未检查bias是否实际存在。当bias为null时，key中bias类型应为-1而非x1的类型，导致匹配不到正确的binary。
- 修复模式：新增biasDtype变量，通过GetOptionalInputDesc检查bias是否存在来设置实际dtype或-1 + 补充bias_null场景的binary配置
- 可审查性：中
- 审查规则建议：simplified_key中可选参数的dtype应根据实际存在性设置，不能用其他参数替代；binary配置应覆盖所有可选参数组合

### df20f36e silu_mul算子定义与kernel不一致导致编译失败
- 根因类别：算子接口定义与kernel实现不匹配(严重)
- 涉及文件：experimental/activation/silu_mul/op_kernel/silu_mul.cpp, silu_mul.h + README.md
- 缺陷描述：算子从单输入input改为双输入(x,y)→输出z，但kernel入口函数仍为`silu_mul(input, output, ...)`，参数数量和语义均不匹配，无法编译出binary。内部处理也有错误：`d = lastDimSize / 2`应为`d = lastDimSize`。
- 修复模式：kernel入口改为`silu_mul(x, y, z, workspace, tiling)` + Init参数对应修改 + 修正d计算 + 更新README
- 可审查性：高
- 审查规则建议：算子定义(proto)修改后必须同步更新kernel入口函数签名和实现

### e3f1fdc5 C04下n轴绑核决策缺陷
- 根因类别：模式适配遗漏(严重)
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_base_blockdim_decision.cpp + conv2d_v2多个tiling文件
- 缺陷描述：1) CheckL1SizeLimitsKernelFullLoad未考虑C04模式下kAL1min/kBL1min应使用C04_CIN_SIZE而非k0，导致L1空间计算错误；2) n轴绑核mix策略在C04下不适用但未排除；3) 带宽系数未区分C04+NHWC场景。26个用例受影响。
- 修复模式：CheckL1SizeLimitsKernelFullLoad增加isC04参数区分计算 + C04下禁用BlockDimFactorMix + 添加C04+NHWC的带宽系数
- 可审查性：低 — 需要深入理解C04格式的特殊约束
- 审查规则建议：新增特殊格式(C04等)时，必须全面审查所有tiling决策路径是否需要适配

### 65140bdf 移除不支持的debug配置参数
- 根因类别：构建配置错误
- 涉及文件：cmake/gen_ops_info.cmake + scripts/kernel/binary_script/build_binary_opc_gen_task.sh
- 缺陷描述：`--op_debug_config=debug`参数不被opc支持，导致opc error时不显示报错信息。同时缺少`ASCEND_SLOG_PRINT_TO_STDOUT=1`环境变量。
- 修复模式：删除不支持的debug参数 + 添加日志输出环境变量
- 可审查性：高
- 审查规则建议：构建脚本中的工具参数应与工具文档保持同步；CI配置应确保错误信息可见

### 4512b92e op_api_list多余空行修复
- 根因类别：配置文件格式错误
- 涉及文件：op_api_list文件
- 缺陷描述：多余空行导致格式异常。
- 修复模式：删除多余空行
- 可审查性：高
- 审查规则建议：配置文件应有格式验证

### f89a1872 DequantSwigluQuant shape校验逻辑错误
- 根因类别：布尔逻辑错误(严重)
- 涉及文件：quant/dequant_swiglu_quant/op_host/dequant_swiglu_quant_tiling_arch35.cpp + 文档
- 缺陷描述：weight_scale与group_index匹配校验条件`(A != B) && (C != D)`应为`!(A == B && C == D)`。原条件在A!=B但C==D时仍会通过校验，未能正确拦截非法输入。同时bias维度为1时缺少group_index存在性检查。
- 修复模式：修正布尔条件为`!(A == B && C == D)` + 增加bias维度1与group_index互斥检查
- 可审查性：高
- 审查规则建议：复合否定条件建议使用De Morgan定律重写为肯定形式再取反(更易理解)；shape校验应有正/负用例UT覆盖

### affc1f58 magic numbers替换
- 根因类别：代码可维护性(magic number)
- 涉及文件：算子代码
- 缺陷描述：代码中使用magic number。
- 修复模式：定义命名常量替换
- 可审查性：高
- 审查规则建议：禁止裸数字常量

### cf84e222 AddRmsNorm同步问题+空指针检查
- 根因类别：流水线同步错误 + 空指针检查缺失(严重)
- 涉及文件：norm/add_rms_norm/op_host/op_api/aclnn_add_rms_norm.cpp + op_kernel/add_rms_norm_merge_n.h
- 缺陷描述：1) PipeBarrier<PIPE_V>位置错误 — 放在FreeTensor之后，导致Cast计算未完成就释放了输入buffer，数据竞争；2) BF16路径CopyOut缺少V_MTE3/MTE3_V事件同步，向量计算与数据传输之间无依赖保障；3) yComputeOut返回值未做空指针检查。
- 修复模式：PipeBarrier移到FreeTensor之前 + 添加V_MTE3/MTE3_V事件获取-等待-释放序列 + 添加CHECK_RET空指针检查
- 可审查性：低 — 需要理解AscendC流水线屏障和硬件事件模型
- 审查规则建议：PipeBarrier必须在FreeTensor之前；CopyOut(MTE3)前后需要对应的V_MTE3/MTE3_V事件同步；工厂函数返回值必须做null检查

## 第6轮 (commit 101-120)

### 36c945ec batch_norm_backward_reduce分支判断+workspace设置错误
- 根因类别：平台判断逻辑错误 + workspace设置遗漏(严重)
- 涉及文件：norm/sync_batch_norm_backward_reduce/op_host/op_api/aclnn_batch_norm_backward_reduce.cpp
- 缺陷描述：1) GetDtypeSupportList使用SocVersion范围判断(910B~910E)不准确，改为NpuArch枚举判断；2) 空tensor分支下workspaceSize被设为0，但executor实际有workspace需求，应使用GetWorkspaceSize()获取真实值。
- 修复模式：平台判断改为NpuArch枚举 + workspaceSize使用executor->GetWorkspaceSize()
- 可审查性：高
- 审查规则建议：平台判断应使用NpuArch而非SocVersion范围；空tensor路径不能跳过workspace设置

### d6acf37e unique_consecutive缺少ascend950 binary配置
- 根因类别：binary配置缺失
- 涉及文件：index/unique_consecutive/op_host/config/ascend950/(新增文件)
- 缺陷描述：unique_consecutive算子缺少ascend950平台的binary.json配置文件，导致该平台无法使用。
- 修复模式：新增ascend950平台的binary配置文件
- 可审查性：高
- 审查规则建议：新增算子或新平台时应同步添加所有目标平台的binary配置

### aa11f503 AdaptiveAvgPool3d缺少shape维度值<=0拦截
- 根因类别：输入校验缺失
- 涉及文件：pooling/adaptive_avg_pool3d/op_api/aclnn_adaptive_avg_pool3d.cpp + 文档
- 缺陷描述：self或out的shape某个维度值小于等于0时未拦截，可能导致后续计算异常。
- 修复模式：增加shape维度值>0的校验 + NC维度一致性检查 + 更新文档错误码说明
- 可审查性：高
- 审查规则建议：aclnn接口入口应校验所有shape维度值为正数

### 9b0eb33e cross_entropy_loss vfReduceMax偶现精度问题
- 根因类别：向量指令使用不当(严重)
- 涉及文件：loss/cross_entropy_loss/op_kernel/arch35/cross_entropy_loss_full_load.h
- 缺陷描述：vfReduceMax在R轴大于256byte时偶现取不到真正的最大值，导致精度问题。原实现区分cNum<vfLen和>=vfLen两种路径，逻辑复杂且有缺陷。
- 修复模式：重写ReduceMax逻辑，统一处理路径，使用更可靠的寄存器操作方式
- 可审查性：低 — 需要理解AscendC向量指令的行为细节
- 审查规则建议：ReduceMax等归约操作需要对大于256byte的场景做专门验证

### 624b4334 文档错别字("只有"→"只要")
- 根因类别：文档错误
- 涉及文件：matmul下多个算子文档
- 缺陷描述：确定性计算说明中"只有输入相同"应为"只要输入相同"，语义截然不同。
- 修复模式：修正措辞
- 可审查性：高
- 审查规则建议：技术文档中逻辑连接词(只有/只要/如果/只要)需要准确使用

### 2b10b6c4 embedding bag未初始化内存导致core dump
- 根因类别：未初始化内存(严重)
- 涉及文件：index/embedding_bag/op_host/embedding_bag_infershape.cpp
- 缺陷描述：mean模板使用了未初始化的内存区域，导致地址越界引发core dump。geir通路和精度测试场景下触发。
- 修复模式：重写infershape函数，在使用内存前先初始化；拆分为独立的InferShape4OutputSupport等子函数
- 可审查性：中
- 审查规则建议：所有内存区域使用前必须初始化；infershape函数应考虑动态shape(-1)场景

### 1de0b7a4 experimental算子冒烟环境rpath失效
- 根因类别：构建/部署配置错误
- 涉及文件：build.sh + experimental/activation/relu_v2/op_host/relu_v2_def.cpp
- 缺陷描述：冒烟环境存在自定义算子包的共享库，rpath失效导致链接到错误版本。需要显式设置LD_LIBRARY_PATH。
- 修复模式：在build_single_example中添加LD_LIBRARY_PATH设置 + 执行后恢复原值
- 可审查性：高
- 审查规则建议：自定义算子的example构建应显式设置库搜索路径，不应依赖rpath

### 06393c8e CtcLossBackward上边界用例aicore_error
- 根因类别：边界条件缺失(严重)
- 涉及文件：loss/ctc_loss_v2_grad/op_kernel/arch35/ctc_loss_v2_grad.h
- 缺陷描述：当targetLength==0时，代码仍访问`2*targetLength-1`即-1位置的targetPrime，导致数组越界引发aicore_error。
- 修复模式：在访问`2*targetLength-1`前增加`targetLength > 0`条件判断
- 可审查性：高
- 审查规则建议：涉及`length-1`下标访问时必须检查length>0；上边界用例(length=0/1)应有专门UT

### 74d43ef3 scatter_sub缺少流水同步导致竞态
- 根因类别：流水线同步缺失(严重)
- 涉及文件：index/scatter_add/op_kernel/arch35/scatter_add_simd_support_atomicadd.h
- 缺陷描述：ScatterSub操作中，从GM拷贝数据到UB前缺少MTE3_MTE2同步，执行减法前缺少MTE2_V同步。数据在MTE/V/S之间流动时序不当引发竞态，导致相同输入产生不同输出。
- 修复模式：在DataCopy前添加MTE3_MTE2事件同步 + 在NegateUpdate前添加MTE2_V事件同步
- 可审查性：低 — 需要理解AscendC流水线同步模型
- 审查规则建议：DataCopy(MTE2)前需确保前序MTE3完成；向量运算(V)前需确保MTE2数据就绪；ScatterAdd/Sub的非标量路径需要完整的事件同步链

### acadb4c7 repeat_interleave int32溢出
- 根因类别：整数溢出(严重)
- 涉及文件：index/repeat_interleave/op_host/arch35/repeat_interleave_tiling_normal.cpp/.h
- 缺陷描述：当repeatSum(输出总元素数)超过INT32_MAX时，使用int32计算导致溢出。原代码未区分是否需要int64。
- 修复模式：新增UseInt64()函数检查yShape总大小是否超INT32_MAX；根据结果选择int32或int64计算路径(不同tiling key)
- 可审查性：高
- 审查规则建议：涉及元素数量计算的变量应默认使用int64_t；tiling中应有大shape的int32溢出检查

### f992c37b 文档拼写错误(HFLOAT32)
- 根因类别：文档错误
- 涉及文件：aclnnConvTbc.md
- 缺陷描述：HFLOAT32拼写错误。
- 修复模式：修正拼写
- 可审查性：高
- 审查规则建议：文档spell check

### 05db401f 多项修复(simplified_key+indexSize计算+paramType)
- 根因类别：配置错误 + 计算逻辑错误(严重)
- 涉及文件：index_fill配置 + gather_v2 op_api + fused_cross_entropy_loss配置 + 文档
- 缺陷描述：1) index_fill的simplified_key.ini错误指向了FusedCrossEntropyLossWithMaxSum算子名；2) gather_v3的indexSize计算中多乘了GetSizeByDataType(应为元素个数而非字节数)；3) fused_cross_entropy_loss的json中weight参数paramType应为optional而非required。
- 修复模式：修正算子名 + 修正计算公式 + 修正paramType
- 可审查性：高
- 审查规则建议：simplified_key配置文件中的算子名必须与实际算子匹配；元素个数计算不应包含dtype大小

### 20350c7d 文档typo修复
- 根因类别：文档错误
- 涉及文件：aclnn返回码文档 + op_debug_prof文档
- 缺陷描述：文档中存在多处typo。
- 修复模式：修正typo
- 可审查性：高
- 审查规则建议：文档CI应集成spell check工具

### 420a1cdd 文档修复(FusedLinearCrossEntropyLossGrad)
- 根因类别：文档错误
- 涉及文件：FusedLinearCrossEntropyLossGrad文档
- 缺陷描述：文档内容有误。
- 修复模式：修正文档
- 可审查性：高
- 审查规则建议：文档修改应由算子开发者审核

### 695d75ef bmm opinfo fp16/bf16→fp32输出条件错误
- 根因类别：条件判断过宽(严重)
- 涉及文件：matmul/common/op_host/op_api/batch_matmul_util.cpp
- 缺陷描述：enableFp16Bf16InFp32Out标志对所有BatchMatMul调用都生效，但实际应仅限Baddbmm接口且仅限A2/A3平台。非Baddbmm接口走了fp32输出路径导致精度失败。另外K==1时不应先Cast到float再Mul。
- 修复模式：CreateBatchMatmulOpInfo增加isBaddbmm参数；条件中加入平台和isBaddbmm判断；K==1分支去除Cast
- 可审查性：中
- 审查规则建议：特定接口的优化路径应严格限定调用来源；新增条件分支时验证对现有调用链的影响

### 958f074a 统计耗时脚本变量命名错误
- 根因类别：变量名引用错误
- 涉及文件：scripts/ci/analyze_ops_time.py
- 缺陷描述：变量先赋值给main_func后续却用op变量名引用，导致Python NameError。
- 修复模式：统一变量名为op
- 可审查性：高
- 审查规则建议：Python脚本应启用pylint/flake8检查未定义变量

### d674d0da 编译warning修复(未使用变量+日志拼写)
- 根因类别：编译warning + 日志拼写错误
- 涉及文件：quant/dequant_swiglu_quant/op_host/dequant_swiglu_quant_tiling_arch35.cpp
- 缺陷描述：1) DoOpTiling中声明了未使用的`ge::graphStatus ret`变量；2) 日志中"exist"拼写为"exit"。
- 修复模式：删除未使用变量 + 修正拼写
- 可审查性：高
- 审查规则建议：编译warning应在CI中视为error

### c57524eb scatter_nd_add生成重复二进制文件
- 根因类别：binary配置错误
- 涉及文件：index/scatter_nd_add/op_host/config/ascend950/scatter_nd_add_binary.json
- 缺陷描述：binary.json中use_locking属性值设为false(固定值)，导致use_locking=true和false的情况生成同一个binary文件名。应设为null表示不区分该属性。
- 修复模式：将use_locking的value从false改为null
- 可审查性：高
- 审查规则建议：binary配置中不影响kernel行为的属性值应设为null；binary文件不应出现重复

### 557ea7a7 覆盖率脚本权限问题
- 根因类别：CMake配置错误
- 涉及文件：tests/ut/CMakeLists.txt
- 缺陷描述：CMake custom_command中直接执行脚本路径但未用bash调用，导致权限不足报错。同时缺少POST_BUILD关键字。
- 修复模式：命令前加bash + 添加POST_BUILD
- 可审查性：高
- 审查规则建议：CMake custom_command中调用shell脚本应使用bash显式调用

### 640c1939 文档demo内存分配size计算错误
- 根因类别：文档示例代码错误
- 涉及文件：quant/dynamic_quant_v2/docs/aclnnDynamicQuantV2.md
- 缺陷描述：CreateAclTensor中使用`sizeof(T)`计算内存大小，但T为uint16_t(fp16的host表示)而实际设备端需要按float大小分配，导致内存不足。
- 修复模式：将`sizeof(T)`改为`sizeof(float)`
- 可审查性：高
- 审查规则建议：文档示例代码中内存分配应注明host/device数据类型差异；示例代码应在CI中编译验证

## 第7轮 (commit 121-140)

### 1532f915a473f52a7fccc9968eca2cc20c61e0f2 inpalceindexadd 分核逻辑修复
- 根因类别：整数除法导致零值（边界条件缺陷）
- 涉及文件：index/inplace_index_add/op_host/arch35/inplace_index_add_simd_sort_tiling.cpp
- 缺陷描述：ubIndexFactor_通过整数除法halfUbSize/(...)计算，大shape场景分母大于分子导致结果为0。下方while(--ubIndexFactor_)循环中0先自减为-1，循环无法正常退出或产生错误行为。两处DoOpTilingSplitAfterSort和DoOpTilingSplitPreSort都有此问题。
- 修复模式：边界值补偿——在整数除法结果后+1，配合下方--循环形成"先试后退"的搜索模式
- 可审查性：中
- 审查规则建议：整数除法结果作为循环初始值时，需检查除法结果为0的情况是否有保护；关注while(--x)模式中x初始值可能为0的问题

### 90c51245d3d36ab4d7bf0705832c5678404817a5 fix opHost\opGraph UT
- 根因类别：UT修复（编译错误+构建配置缺陷）
- 涉及文件：tests/ut/op_graph/test_op_graph_main.cpp, tests/ut/op_host/CMakeLists.txt
- 缺陷描述：(1) 预编译宏误写#elif define(__x86_64__)应为#elif defined(__x86_64__)，缺d后缀导致编译错误；(2) CMakeLists中$<TARGET_EXISTS:...>生成器表达式在目标不存在时产生空源文件列表使add_library失败。
- 修复模式：(1) 拼写纠正 (2) 将运行时生成器表达式改为configure阶段条件判断+fallback
- 可审查性：高
- 审查规则建议：#elif define(应自动标记为疑似错误（正确形式是defined）；CMake中$<TARGET_EXISTS:>嵌套$<TARGET_OBJECTS:>需检查目标不存在时的空列表问题

### a39e185487260618620b98539b3f05c3f798db4a revert small shape optimization in mmv3
- 根因类别：优化引入的正确性缺陷（workspace脏数据）
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp, matmul/mat_mul_v3/tests/ut/op_host/test_matmul_v3_tiling.cpp
- 缺陷描述：先前为小shape添加V_PARALELL_ND2NZ优化分支，该模板可能导致workspace中存在脏数据参与计算产生精度问题。
- 修复模式：revert回退——删除有问题的优化路径，回退到V_HEAD_ND2NZ通用路径
- 可审查性：低
- 审查规则建议：新增优化路径需验证workspace初始化/清理逻辑是否覆盖所有场景；涉及ND2NZ格式转换的新模板应有workspace脏数据检查测试用例

### bf2ac324b478f0b492161646db1638a389b17e88 修复 aclnnaddmv 接口文档描述问题
- 根因类别：文档错误
- 涉及文件：matmul/addmv/docs/aclnnAddmv.md
- 缺陷描述：API文档中self/mat/vec/out维度列缺失具体维度数（用-代替），beta参数format/维度/是否必须列描述有误，out数据类型多列了BOOL。
- 修复模式：文档内容纠正
- 可审查性：高
- 审查规则建议：接口文档中tensor参数维度字段不应为-，应有明确维数描述；文档数据类型列表应与代码实际支持列表同步校验

### 9500a44b04e514bf5fce53460fee6377e4afc8f6 [MatMul] 修复大K场景因stepka错误调整引入的性能劣化问题
- 根因类别：优化代码作用域错误（未隔离API层级）
- 涉及文件：matmul/fused_mat_mul/op_host/op_tiling/arch35/fused_matmul_asw_basic_tiling.cpp/.h, matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_asw_tiling.cpp, matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_basic_aswt_tiling.cpp
- 缺陷描述：基类MatMulV3AswTiling::DoOpTiling()中调整了stepKa/stepKb计算方式，但此基类被基础API和高阶API(FusedMatMul)共同继承。高阶API有自己的depthA1/depthB1参数，基类的stepK调整改变了depthA1导致高阶API性能劣化。
- 修复模式：职责下沉——将特定于子类的逻辑从基类移至子类，通过override隔离
- 可审查性：中
- 审查规则建议：修改继承体系中基类的tiling逻辑时，必须验证所有子类场景的性能；stepK/depthA1/depthB1之间存在关联依赖，修改stepK时应同步检查depth是否需要更新

### 8d8b3dbbf62ff383a238e3f9868bfbdbebb484c8 修复优化ScatterNdAdd算子
- 根因类别：多类复合缺陷（芯片判断硬编码+数据类型传参错误+buffer计算错误+边界条件修复）
- 涉及文件：index/scatter_nd_add/op_api/scatter_nd_add.cpp, index/scatter_nd_add/op_host/arch35/scatter_nd_add_tiling_base.cpp/.h, index/scatter_nd_add/op_kernel/arch35/scatter_nd_add_common.h/.h
- 缺陷描述：(1) GetSocVersion()>=ASCEND950硬编码判断芯片改为IsRegbase()通用接口；(2) 多处将废弃字段indicesType_传给GetSortTmpSize()，应改为indiceDtype_；(3) buffer大小从halfUbSize改为UbSize；(4) deterministic分支indicesFactor_>indicesAxis_改为>eachCoreIndexCount_修正分核边界；(5) 新增排序条件排除INT64；(6) 新增isOpti_优化分支。
- 修复模式：综合修复——架构抽象化+参数纠正+内存计算修正+分核边界修正+新优化路径
- 可审查性：低
- 审查规则建议：芯片型号硬编码GetSocVersion()>=ASCENDXXX应统一使用架构抽象接口；dtype变量重命名/废弃时应全文搜索替换；单PR混合bugfix和新feature应拆分

### 7ff1df8c981ad22a41a34ff36c98db2c1e8e3b57 修复aclnnInplaceMaskedScatter算子不同芯片架构下分支判断逻辑错误
- 根因类别：条件判断取反（逻辑错误）
- 涉及文件：index/masked_scatter/op_api/aclnn_masked_scatter.cpp
- 缺陷描述：IsRegbase()判断写反——原代码if(IsRegbase())走l0op::MaskedScatter路径，else走ProcessBroadcast路径，实际应反转。同时修复else分支传入selfRef（未contiguous）改为selfRefContiguous。
- 修复模式：布尔条件取反 + 参数纠正
- 可审查性：高
- 审查规则建议：芯片架构分支选择逻辑必须有注释说明每个分支对应哪种架构；IsRegbase()/!IsRegbase()分支应在review中重点关注

### b4c5ba659f914fbf60a099b1b30b51e2b9bd424f 回退extern aclnn Inner修改
- 根因类别：构建系统改动引入的兼容性问题
- 涉及文件：CMakeLists.txt, cmake/opbuild.cmake, cmake/package.cmake, cmake/variables.cmake, 3个op_api .cpp文件
- 缺陷描述：将aclnnInner的extern声明改为通过自动生成头文件#include引入，但构建顺序或路径不正确导致编译失败。回退后恢复到extern显式声明方式。
- 修复模式：revert回退
- 可审查性：中
- 审查规则建议：将extern声明改为头文件include时需确保构建顺序正确；构建系统改动应有完整端到端编译验证（增量+全量）

### c1469b683950d49e2e5c583f4732442a256d08f4 bugfix for kirin：quant_batch_matmul_v4 && transpose_batch_mat_mul && weight_quant_batch_matmul_v2 && batch_norm_v3 && max_pool3d_with_argmax_v2
- 根因类别：新芯片平台兼容性缺陷（端云不兼容+编译宏不完整+配置冗余）
- 涉及文件：16个文件，涉及5个算子
- 缺陷描述：Kirin芯片适配中：(1) 三个算子端云无法兼容需删除kirin配置；(2) transpose_batch_mat_mul的Fixpipe指令需增加__NPU_ARCH__==3003||3113条件（6处）；(3) max_pool3d_with_argmax_v2的binary.json残留已删除attr。
- 修复模式：平台配置清理+编译条件扩展
- 可审查性：中
- 审查规则建议：新芯片适配时应有端云兼容性检查清单；__DAV_C220_CUBE__等硬件宏应有统一平台抽象层；binary.json的attr应与代码同步

### 33fcbea47ec66d3c4c6096aa07bd920d5e645d7d fix third_party download and install path
- 根因类别：构建配置缺陷（第三方依赖路径不统一）
- 涉及文件：CMakeLists.txt, build.sh, cmake/variables.cmake, 多个third_party cmake文件
- 缺陷描述：CANN_3RD_PKG_PATH和CANN_3RD_LIB_PATH两个变量用途重叠路径不统一；abseil-cpp.cmake硬编码路径而不用变量；protobuf缺少缓存路径支持；build.sh清理路径硬编码。
- 修复模式：路径统一化+变量归一化+冗余清理
- 可审查性：高
- 审查规则建议：构建脚本中第三方依赖路径应统一通过变量引用禁止硬编码；同一用途不应有多个变量

### 97caad29deca72b227bda63722a5a203c1ad8858 修复UT工程bug
- 根因类别：构建脚本缺陷
- 涉及文件：build.sh
- 缺陷描述：连续执行不同算子UT时，ai_core目录下json文件未被清理，导致核函数文件名读取到旧json编译出错。
- 修复模式：在build_ut()进入构建目录后增加rm -rf删除ai_core下所有json文件
- 可审查性：高
- 审查规则建议：构建脚本增量编译逻辑应检查跨任务状态污染；依赖生成文件的构建流程应有清理或版本校验机制

### 7651c0fd29deb39a45143b112c86eb76de3cb9ce msda修复numPoints=1时找不到kernel问题，增加numLevels维度一致性校验
- 根因类别：边界条件处理缺陷 + 输入校验缺失
- 涉及文件：multi_scale_deformable_attn_function_tiling.cpp, aclnn_multi_scale_deformable_attn_function.cpp, multi_scale_deformable_attn_function_def.cpp
- 缺陷描述：(1) numPoints=1时条件numPoints%2==0||numPoints==1导致走入noTranspose路径但该场景未适配tilingkey；(2) 三个输入tensor的numLevels维度未做一致性校验；(3) ascend910残留配置。
- 修复模式：修改条件判断 + 新增输入校验 + 删除无效配置
- 可审查性：中
- 审查规则建议：多条tiling路径时应对每条路径的边界值（如numPoints=1）有UT覆盖；多tensor同名维度应强制一致性校验

### 9a9bb866fb55f2d6f6d9223b74b5d0371f5b8bb9 修复instanceNormV3算子example不支持arch2002上调用的问题
- 根因类别：平台适配缺陷
- 涉及文件：build.sh, norm/instance_norm_v3/examples/arch20/test_aclnn_instance_norm.cpp(新增), examples/test_aclnn_instance_norm.cpp(删除)
- 缺陷描述：instanceNormV3是纯arch2002算子但example放在通用目录，build.sh的build_example只搜索通用和arch35目录缺少arch20。旧example内容全被注释掉临时规避CI。
- 修复模式：迁移example到arch20子目录 + 构建脚本增加ascend310p平台路径匹配
- 可审查性：高
- 审查规则建议：新增算子example时应确认支持设备列表与example存放路径一致；build.sh的build_example平台分支应覆盖所有已支持平台

### 75b0eb1e89f7e4ed69d95f745f34bca28cd0beef fix aclnnSigmoidBackward.md
- 根因类别：文档错误
- 涉及文件：activation/sigmoid_grad/docs/aclnnSigmoidBackward.md
- 缺陷描述：markdown格式问题（不必要的转义）；错误码场景描述笼统写"shape不一致"，修复后拆分为三个具体场景。
- 修复模式：文档内容修正和细化
- 可审查性：高
- 审查规则建议：API文档错误码描述应与代码中实际校验逻辑一一对应

### 37ae7c27cac15614eb123be0f2fa64ebdbc73fbd fix 负载均衡性能劣化
- 根因类别：Tiling策略条件判断不当（性能缺陷）
- 涉及文件：adaptive_sliding_window_tiling.cpp, test_quant_batch_matmul_v3.csv
- 缺陷描述：balanceAfterFixp条件原只在kSize<1024时为true，但kSize==1024且nCore>=8时也应启用边界优化。缺少此条件导致性能劣化。
- 修复模式：扩展条件判断增加kSize==1024&&nCore>=8分支
- 可审查性：低
- 审查规则建议：tiling策略条件阈值变更需配合性能基线测试；多维度交叉条件的tiling逻辑应在注释中明确每个阈值的物理含义

### c467ceed2c123df910bb7ccab1f167172a7be244 bmm ut补充修复性能问题
- 根因类别：Tiling策略优先级配置缺陷
- 涉及文件：batch_matmul_v3_tiling_strategy.h, 多个UT文件
- 缺陷描述：ASCEND950平台的BatchMatMulV3策略优先级列表缺少ITER_BATCH_BASICAPI策略，导致某些场景未匹配最优策略路径。
- 修复模式：调整策略优先级顺序 + 补充UT
- 可审查性：低
- 审查规则建议：策略优先级列表变更需附性能对比数据；新平台支持时应检查策略列表是否完整覆盖所有已实现策略

### e63cd956285308d58a1a141cb70948af7830b0cb fix scatterElementsV2 bare instructions
- 根因类别：代码规范（裸指令）
- 涉及文件：scatter_elements_v2_cache_scatter_add.h
- 缺陷描述：12处使用C风格裸指令pipe_barrier(PIPE_V)未使用AscendC封装模板API。
- 修复模式：将pipe_barrier(PIPE_V)替换为PipeBarrier<PIPE_V>()，机械式重构
- 可审查性：高
- 审查规则建议：可通过静态扫描禁止pipe_barrier等裸指令调用，强制使用PipeBarrier<>()模板API

### 34ad168fe776a1a89b643fc9ed0e9f0b44a17bf3 修正rms_norm_quant_v2的编译warning
- 根因类别：编译警告（参数遮蔽+有符号/无符号比较）
- 涉及文件：rms_norm_quant_v2_tiling_regbase_base.cpp, rms_norm_quant_v2_tiling_regbase_recompute.cpp
- 缺陷描述：(1) 函数参数名与类成员同名产生遮蔽warning，修复为加in前缀；(2) int64_t和uint64_t混合比较，修复为显式static_cast<uint64_t>。
- 修复模式：重命名参数消除遮蔽 + 显式类型转换
- 可审查性：高
- 审查规则建议：CI编译应开启-Wshadow和-Wsign-compare并设为error；函数参数命名应避免与类成员同名

### 4d1939f54fbdccb133dadf84b124993cc7e47a62 卷积反向dx算子正向流程补齐以及codecheck修复
- 根因类别：功能缺失 + 代码规范问题
- 涉及文件：conv/convolution_forward/op_host/op_api/convolution.cpp
- 缺陷描述：(1) 正向卷积(非转置)的N2H路径未补齐，只有转置卷积路径；(2) 使用magic number(63/1500/4096等)；(3) 条件不满足时用OP_LOGE应改为OP_LOGD；(4) 循环变量int64_t与无符号Size()比较；(5) output shape构造方式优化。
- 修复模式：功能补齐 + 代码重构（提取常量、拆分函数、修正日志级别、修复类型不匹配）
- 可审查性：中
- 审查规则建议：算子优化路径适用性检查应用调试级别日志而非错误级别；magic number应定义为命名常量

### a9951ec742db23c76ba1ceaa281e562d454ca6f3 Atlas推理 rmsnorm性能提升、layernormv4错误描述优化、addrmsnormquantv2校验修复
- 根因类别：校验返回值丢弃（代码逻辑缺陷）+ 性能优化 + 错误信息优化
- 涉及文件：aclnn_add_rms_norm_quant_v2.cpp, aclnn_fast_layer_norm.cpp, rms_norm_tiling.cpp, rms_norm_merge_n.h
- 缺陷描述：(1) addrmsnormquantv2中CheckParamsV2和SelfPreDealData返回值被丢弃，校验失败仍继续执行；(2) layernormv4错误信息不够清晰；(3) rmsnorm在310P平台norm轴=128时未走MODE_MERGE_N模板性能差10倍，SetBlockDim调用位置在模板选择前但310P的MODE_MERGE_N会修改useCoreNum。
- 修复模式：(1) 补充返回值检查 (2) 改进错误消息 (3) 扩展tiling条件+调整SetBlockDim位置
- 可审查性：中
- 审查规则建议：调用返回aclnnStatus的函数必须检查返回值，可用[[nodiscard]]或静态分析强制；SetBlockDim调用应在所有可能修改useCoreNum的逻辑之后

## 第8轮 (commit 141-160)

### 3e1ff64 convolution_backward 算子注释文档修正
纯文档修改（代码注释中的路径修正），跳过。

### a291109 softmaxGrad文档修正
纯文档修改（README/接口文档修正），跳过。

### 8af6f91 sparsesoftmaxcrossentropywithlogits AICERROR问题修复
- 根因类别：API配置遗漏（SIMT/SIMD混合编程场景未设置UB大小）
- 涉及文件：loss/sparse_softmax_cross_entropy_with_logits/op_host/sparse_softmax_cross_entropy_with_logits_tiling_arch35.cpp
- 缺陷描述：tiling的DoTiling()中设置了BlockDim和ScheduleMode，但漏掉SetLocalMemorySize(ubSize)调用。SIMT/SIMD混合编程模式要求显式设置UB内存大小，缺失导致AIC ERROR。
- 修复模式：单行补漏，在SetScheduleMode后添加SetLocalMemorySize调用
- 可审查性：高
- 审查规则建议：使用SIMT/SIMD混合编程的tiling文件，SetBlockDim/SetScheduleMode/SetLocalMemorySize三者必须同时出现

### d9f57c1 修正aclnnFusedMatmul资料
纯文档修改，跳过。

### 146f4f8 修复TopKTopPSampleV2 minP采样精度问题
- 根因类别：变量语义复用（同一变量在不同阶段承载不同含义导致错误引用）
- 涉及文件：index/top_k_top_p_sample_v2/op_kernel/top_k_top_p_sample_v2.h
- 缺陷描述：maxValue在softmax计算过程中从原始logit值变为概率值，但minP>=1.0的退化采样路径需要原始最大值来回填localValueRs。未保存原始值导致精度错误。
- 修复模式：引入oriMaxValue镜像变量保存原始状态，在6个分支点捕获原始最大值
- 可审查性：低
- 审查规则建议：当同一变量在赋值后被进一步变换再使用时，检查所有使用点是否期望变换前还是变换后的值；多分支状态设置应具有对称性

### 88f4ebf fix aclnnFastGeluBackward check
- 根因类别：参数校验配对顺序错误
- 涉及文件：activation/fast_gelu_grad/op_api/aclnn_fast_gelu_backward.cpp
- 缺陷描述：dtype和shape校验时，原代码将(gradOutput,gradInput)和(self,gradInput)配对校验，正确应为(self,gradOutput)和(gradOutput,gradInput)，原写法在传递性校验上存在逻辑漏洞
- 修复模式：调整校验配对顺序
- 可审查性：高
- 审查规则建议：3个以上tensor两两校验一致性时，检查配对是否覆盖正确的传递链(A==B, B==C)

### 20a9c63 embadding bag算子infer shape动态场景bug修改
- 根因类别：动态shape场景未处理（边界条件遗漏）
- 涉及文件：index/embedding_bag/op_host/embedding_bag_infershape.cpp, 对应UT
- 缺陷描述：infer shape只处理了unknown rank，未处理unknown shape（dim为-1/负值），导致负值直接参与计算产生错误shape，后续core dump
- 修复模式：在每个子shape推导函数中新增is_unknown_shape判断分支，调用SetUnknownShape
- 可审查性：中
- 审查规则建议：infer shape必须同时处理三种场景：静态shape、unknown rank、unknown shape(dim为负值)。检查了IsUnknownRank的代码必须同时检查IsUnknownShape

### 32e3a9f DLQBMM A4W4二级量化 修复尾块偏移错误导致的精度错误
- 根因类别：多核尾块偏移计算错误
- 涉及文件：matmul/dual_level_quant_batch_matmul/op_kernel/arch35/dual_level_quant_batch_matmul_vec_compute.h
- 缺陷描述：CopyUbToGm中realML0Size<128且l0Params.mL1Size>128的尾块场景，VEC0/VEC1核的数据偏移和大小计算错误（统一按CeilDiv(realML0Size,2)分配但尾块切分大小不同）
- 修复模式：新增尾块特殊处理逻辑，VEC0核最多64行，VEC1处理剩余，增加<=0提前返回保护
- 可审查性：低
- 审查规则建议：多核数据切分逻辑存在尾块场景时，验证尾块大小与标准块不同时的偏移计算正确性

### ca1626a 修正aclnnInplaceScatterValue相关描述
纯文档修改，跳过。

### 7c23775 inplaceindexadd deterministic tiling fix
- 根因类别：off-by-one边界错误（< 应为 <=）
- 涉及文件：index/inplace_index_add/op_host/arch35/inplace_index_add_determinstic_tiling.cpp
- 缺陷描述：条件indicesAxis_ < splitCoreNumThresh应为<=，当indicesAxis_恰好等于阈值时未进入有效tiling分支
- 修复模式：一字符修复 < -> <=
- 可审查性：高
- 审查规则建议：阈值比较的分支条件(<vs<=, >vs>=)需确认边界值归属哪个分支

### c32f0f8 修复CTCLossV2 Input ShapeSize大于INT32_MAX AIC_ERROR问题
- 根因类别：整数溢出（int32_t截断）
- 涉及文件：loss/ctc_loss_v2/op_kernel/arch35/ctc_loss_v2_fp16.h, ctc_loss_v2_fp32.h
- 缺陷描述：lpInputStride在函数签名和局部变量中硬编码int32_t，但lpBatchStride等已用模板参数ThreadType。大shape下lpInputStride溢出导致AIC ERROR
- 修复模式：类型提升，int32_t -> ThreadType，fp16/fp32两文件对称修改
- 可审查性：高
- 审查规则建议：同一计算上下文中同类语义变量（stride/offset）应使用一致的数据宽度，部分已升宽时检查剩余字段

### 1c68777 fix EmbeddingDenseGradV2 kernel apt
- 根因类别：kernel apt配置遗漏
- 涉及文件：index/embedding_dense_grad_v2/op_kernel/embedding_dense_grad_v2_apt.cpp
- 缺陷描述：缺少KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_AIV_1_0)声明，涉及核间同步的算子必须设置MIX_AIV类型
- 修复模式：单行添加宏声明
- 可审查性：高
- 审查规则建议：使用核间同步(SyncAll等)的算子kernel，apt入口必须包含KERNEL_TASK_TYPE_DEFAULT声明

### aebe3eb Fixed the bug that padIdx is used for calculation but none value is set in Interface
- 根因类别：接口语义不一致（复合型bug，横跨host/kernel/api三层）
- 涉及文件：index/embedding_bag/下12个文件
- 缺陷描述：(1) paddingIdx传入none但kernel仍用于计算 (2) GetAttrPointer缺空指针检查 (3) validIndicesFactorNumber统计范围错误（跨越整个bag而非当前批次） (4) outQueue从TQue改TBuf修复资源管理 (5) 空bag场景处理不当 (6) 平台判断从SocVersion改NpuArch
- 修复模式：接口-实现全链路对齐重构
- 可审查性：低
- 审查规则建议：单commit修改>5文件且跨host/kernel/api层应拆分；GetAttrPointer调用后必须空指针检查；循环内计数器作用域应与循环层级匹配

### 98b45b6 fix_pull_request
纯文档修改，跳过。

### a92b304 修复算子编译过程中的错误打印日志
- 根因类别：日志信息复制粘贴错误
- 涉及文件：scripts/kernel/binary_script/build_binary_opc.sh
- 缺陷描述：实际调用build_binary_opc_gen_task.sh但失败日志打印build_binary_single_op_gen_task.sh，误导排查
- 修复模式：修正日志字符串中的脚本名
- 可审查性：高
- 审查规则建议：错误日志中引用的命令名应与实际执行的命令匹配

### 2e273d9 [dx] [B702] fix MTE out of range
- 根因类别：条件分支过宽导致MTE越界
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/conv3d_backprop_input_v2_inner_product_tiling.cpp
- 缺陷描述：TILING_HK且dedx_w==1的场景也使用了固定baseM=256计算像素数量，但该场景不应走此路径，导致a1PixelNum偏小、L1空间校验通过但实际搬运越界
- 修复模式：收窄条件分支，删除多余OR条件
- 可审查性：高
- 审查规则建议：if条件含多个OR分支且涉及硬件资源边界计算时，审查每个分支是否满足后续计算的前提假设

### 89f41a2 修复opapi的打桩代码
- 根因类别：stub文件重复定义（与源码enum冲突）
- 涉及文件：tests/ut/op_api/stub/opdev/platform.h
- 缺陷描述：stub头文件手动定义NpuArch枚举，与上游platform/soc_spec.h定义冲突
- 修复模式：删除手写enum，改为#include引入官方定义
- 可审查性：高
- 审查规则建议：stub/mock文件不应手动复制源码enum/struct定义，应通过include引入

### aa742cf 同步脚本,修复出包不能合并构建产物的问题
- 根因类别：构建工具链缺失
- 涉及文件：scripts/package/common/py/merge_binary_info_config.py（新增）
- 缺陷描述：出包流程缺少binary_info_config.json配置合并脚本，导致构建产物无法合并
- 修复模式：新增Python合并脚本
- 可审查性：中
- 审查规则建议：dict.update()是浅合并，嵌套结构可能丢失base子字段；新增构建脚本应配套UT

### 1f9064e slice mm bugfix
- 根因类别：StreamK阈值计算错误 + NonContiguous tensor场景守护缺失
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp, matmul/mat_mul_v3/下4个文件
- 缺陷描述：(1) SK模式k轴阈值公式过大导致错误路径选择 (2) DPSK模式同样阈值过大 (3) createView产生的1D storageShape的3D tensor错误进入StreamK路径
- 修复模式：阈值修正 + 新增IsSelfNonContiguous守护函数
- 可审查性：低
- 审查规则建议：matmul tiling阈值常量变更需附带性能测试数据；commit混合阈值调整和场景守护应拆分

### 5e9bde5 fix mmv2nz to mmv3 nznznd
- 根因类别：算子路由缺陷（特定平台+format组合未切换正确实现）
- 涉及文件：matmul/common/和matmul/mat_mul_v3/下6个文件
- 缺陷描述：A2平台N轴对齐场景下MatMul V2 NZNZNZ路径需切换到V3 NZNZND；同时NZ格式输入从storageShape取k/m维度不正确，应从originShape取
- 修复模式：算子路由扩展 + tiling维度计算修正
- 可审查性：低
- 审查规则建议：功能变更和bug修复混合的大commit应拆分；两平台binary.json完全相同存在copy-paste风险

## 第9轮分析 (commit #161-#180)

### 7a1f3b33 修复QuantBatchMatmulV3纯CUBE模板低阶API的若干问题
- 根因类别：常量重复定义/命名不一致 + 条件分支遗漏枚举值
- 涉及文件：matmul/common/cmct/block/block_mmad_a8w8_fixpipe_quant.h, block_mmad_mx.h, kernel_qbmm_cube.h, quant_batch_matmul_constant.h, adaptive_sliding_window_basic_api_tiling.cpp, quant_batch_matmul_v3_apt.cpp
- 缺陷描述：三类子缺陷：(1) IDX_M_TILE_IDX等常量在三个文件中各自独立定义，与quant_batch_matmul_constant.h中统一常量命名不一致，修复收拢到统一头文件。(2) isCubePerChannel条件只检查DT_UINT64遗漏DT_INT64，INT64 scale类型的PerChannel场景无法进入CUBE模板路径。(3) kernel入口宏用排除式条件(!= DT_FLOAT8_E8M0)不精确，改为显式列举DT_UINT64/DT_INT64/DT_FLOAT/DT_BF16，同时#endif #if改为#else修复互斥分支逻辑。
- 修复模式：常量统一收拢 + 条件分支补全遗漏枚举值 + 排除式改为显式列举
- 可审查性：中
- 审查规则建议：枚举/类型新增成员时全局搜索所有条件判断分支确认覆盖；避免排除式条件选择模板路径

### c834b681 修复qbmmv3精度失败问题
- 根因类别：constexpr if条件逻辑错误 + 命名空间限定缺失
- 涉及文件：matmul/common/cmct/kernel/kernel_qbmm_pertile.h, matmul/common/cmct/utils/coord_utils.h
- 缺陷描述：(1) GetQuantOffset中load balance偏移修正用!(isTransA && !isTransB)将M和N修正耦合，当isTransA=false,isTransB=false时M修正应生效但N不应生效，原代码错误地同时执行两者。修复拆分为独立的if constexpr (!isTransA)和if constexpr (isTransB)。(2) kernel_qbmm_pertile.h中Get<IDX_A_OFFSET>缺少QuantBatchMatmul::命名空间限定，存在同名常量时解析到错误索引导致精度问题。
- 修复模式：拆分复合布尔条件为独立分支 + 显式命名空间限定
- 可审查性：低
- 审查规则建议：涉及transA/transB组合场景应对4种(TT/TN/NT/NN)逐一验证；多namespace中存在同名常量时必须显式限定

### 37855341 修复纯CUBE模板遗漏之处
- 根因类别：off-by-one边界条件
- 涉及文件：matmul/common/cmct/block/block_scheduler_qbmm.h
- 缺陷描述：判断是否需要tail tile split的条件为roundIdx_ < round_ - 1，最后一轮(roundIdx_==round_-1)错误进入tail split路径。修复为roundIdx_ < round_，使正常轮次(含最后一轮)都不做tail split。
- 修复模式：边界条件修正 < round_ - 1 改为 < round_
- 可审查性：高
- 审查规则建议：涉及roundIdx < round - 1与roundIdx < round的边界条件必须验证最后一轮行为

### 330382df 【bugfix】msdag编译告警修复
- 根因类别：未使用参数
- 涉及文件：vfusion/multi_scale_deformable_attention_grad/op_host/op_api/aclnn_multi_scale_deformable_attention_grad.cpp
- 缺陷描述：CheckFormat和CheckShape函数签名包含gradValue/gradLocation/gradAttnWeight三个未使用的输出梯度参数，产生unused parameter告警。修复移除这些参数。
- 修复模式：移除未使用的函数参数
- 可审查性：高
- 审查规则建议：开启-Werror=unused-parameter

### 7a69e382 fix dav_3510 x transpose problem
- 根因类别：平台特定约束缺失
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_api/aclnn_weight_quant_batch_matmul_v2.cpp
- 缺陷描述：ContiguousCheck对x tensor允许转置，但DAV_3510平台不支持x转置。修复增加平台判断：DAV_3510时要求!transposeX && IsContiguous(x)。
- 修复模式：增加平台特定参数约束校验分支
- 可审查性：低
- 审查规则建议：多硬件平台时应有平台能力矩阵，新增平台对照矩阵逐项校验

### 76626cce swish相关代码整改&quant类问题修复
- 根因类别：混合提交(空指针链式调用 + API迁移 + 新特性)
- 涉及文件：activation/silu_grad/op_host/arch35/silu_grad_tiling.cpp, activation/swish/ 多个文件, quant/ascend_quant/ 多个文件
- 缺陷描述：(1) silu_grad tiling中context_->GetOutputShape(0)->GetStorageShape()链式调用未检查GetOutputShape(0)是否为null。(2) swish算子将旧错误处理宏统一替换为新标准宏。(3) 新增dynamic_mx_quant相关算子。
- 修复模式：防御式编程(空指针检查) + API迁移
- 可审查性：低
- 审查规则建议：链式调用返回指针的方法时必须每一步检查null

### 1719d1ff fix foreach_mul_scalar_v2 aclnn storage shape bug
- 根因类别：storage shape与view shape不一致
- 涉及文件：foreach/foreach_mul_scalar/op_host/op_api/aclnn_foreach_mul_scalar_v2.cpp
- 缺陷描述：输出tensor的storageShape和viewShape不一致时，后续格式检查和计算基于错误的storageShape。修复在CheckShape后增加SetOutStorageShape步骤，将每个输出tensor的storageShape重设为viewShape。
- 修复模式：在校验流程中增加shape归一化步骤
- 可审查性：高
- 审查规则建议：输出tensor由调用方传入时，使用前必须确保storageShape与viewShape一致性

### 60a0233d slice matmul bugfix
- 根因类别：回退路径不完整(slice非连续处理)
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp, matmul_util.h
- 缺陷描述：(1) IsSliceNonContiguous未检查NZ格式和左右矩阵dtype是否相同，这些检查延后到调用处。(2) 当slice路径不支持需回退到连续路径时，缺少batch维度fold操作，3D tensor的batch维没被正确合并到M维导致shape不匹配。修复将约束检查前置+回退路径补全维度折叠。
- 修复模式：前置约束检查 + 补全回退路径维度折叠
- 可审查性：中
- 审查规则建议："乐观尝试+回退"模式中，回退路径必须完整恢复到连续路径等价语义

### cf9ea8c9 修复dynamic_quant精度问题
- 根因类别：硬件指令参数溢出(DMA参数uint16溢出)
- 涉及文件：quant/dynamic_quant/op_kernel/dynamic_quant_unalign_large_310p.h, dynamic_quant.cpp, test_dynamic_quant.cpp
- 缺陷描述：DMA搬运指令参数使用uint16_t类型，当stride值超过UINT16_MAX时截断溢出，导致搬运地址错误引发精度问题。修复为溢出时将safeCalRow降为1逐行处理。
- 修复模式：运行时溢出检测 + 降级为逐行处理
- 可审查性：高
- 审查规则建议：DMA/搬运指令参数转uint16_t前必须检查溢出；硬件指令参数有位宽限制时必须做溢出保护

### 7dbffe7c fix ascend config json
- 根因类别：编译配置缺失 + infershape提前返回缺失
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json, index/repeat_interleave/op_host/repeat_interleave_infershape.cpp
- 缺陷描述：(1) 约30个算子在ascend910_95上缺少-mllvm -cce-aicore-dcci-before-kernel-end=false编译选项。(2) repeat_interleave infershape中设置UnknownRank后没有return，继续执行后续shape推导导致非法维度访问。
- 修复模式：配置补全 + early return
- 可审查性：中
- 审查规则建议：infershape中设置UnknownRank/UnknownShape后必须立即return

### 7631a0fb bugfix (splitK bias添加时机)
- 根因类别：逻辑错误(条件判断值错误)
- 涉及文件：matmul/mat_mul_v3/op_kernel/arch35/mat_mul_asw_kernel.h
- 缺陷描述：splitK场景bias添加条件为kIndex == splitKRound-1(最后一轮)，但bias应在第一轮(kIndex==0)加入，否则与中间累加逻辑冲突导致精度偏差。
- 修复模式：条件值修正 splitKRound-1 -> 0
- 可审查性：中
- 审查规则建议：splitK/分块累加场景中，一次性addend(bias/残差)应在首轮注入而非末轮

### b7b3a900 fix kernel ut bug when depends multi ops
- 根因类别：构建系统逻辑缺陷(CMake参数解析 + Shell健壮性)
- 涉及文件：build.sh, cmake/ut.cmake
- 缺陷描述：(1) ut.cmake中AddOpTestCase用硬编码位置解析可变参数，UT依赖多个算子时丢弃多余依赖。改为foreach循环遍历。(2) build.sh无条件执行pre_op_kernel_ut target，不存在时报错。改为先检查target是否存在。
- 修复模式：参数解析从固定位置改为循环遍历；构建命令加target存在性检查
- 可审查性：高
- 审查规则建议：CMake处理可变参数应用foreach或cmake_parse_arguments，禁止硬编码index

### ad433211 修复qbmm编译问题
- 根因类别：标识符大小写拼写错误
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_apt.cpp
- 缺陷描述：宏内调用QbmmCmctPerTileKernel(T大写)但实际类名为QbmmCmctPertileKernel(t小写)，导致编译失败。
- 修复模式：单字符大小写修正
- 可审查性：高
- 审查规则建议：宏展开中引用的类名应与声明完全一致；该路径之前未被编译覆盖说明测试覆盖不足

### 7b9edfcb fix marndq binary, remove opapi
- 根因类别：构建配置错误(CMake参数 + ini section名)
- 涉及文件：norm/multi_add_rms_norm_dynamic_quant/op_host/CMakeLists.txt, simplified_key.ini
- 缺陷描述：(1) ACLNNTYPE为aclnn导致生成多余op_api代码与手写实现冲突，改为aclnn_exclude。(2) ini section名MULTI_ADD_RMS_NORM_DYNAMIC_QUANT(全大写)与框架期望的CamelCase名MultiAddRmsNormDynamicQuant不匹配。
- 修复模式：CMake参数值修正 + ini section名大小写规范修正
- 可审查性：高
- 审查规则建议：simplified_key.ini的section名必须与算子CamelCase注册名一致；CI加校验脚本

### dea19dca elu、gelu、sigmoid等问题修复
- 根因类别：混合修复(平台判断硬编码 + 结构体声明顺序 + 变量名遮蔽)
- 涉及文件：activation/elu/fast_gelu/sigmoid_grad/silu_grad等约50+文件
- 缺陷描述：(1) BF16支持判断用GetSocVersion()逐芯片枚举，改为GetCurNpuArch()==DAV_2201||IsRegbase()按架构族判断。(2) CompileInfo结构体声明顺序调整。(3) 函数参数名与外层作用域同名标识冲突。
- 修复模式：平台判断从枚举SoC改为架构族判断；结构体声明顺序调整；变量重命名消除遮蔽
- 可审查性：中
- 审查规则建议：禁止op_api中用GetSocVersion()逐个枚举判断特性支持，应使用NpuArch架构族判断

### b95a1b31 修复sc告警信息
- 根因类别：隐式布尔转换告警
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/quant_batch_matmul_v3_iterbatch_tiling.cpp
- 缺陷描述：!aicoreParams_.aicNum对无符号整型做隐式布尔转换判零，改为aicoreParams_.aicNum == 0。
- 修复模式：隐式布尔转换改为显式比较
- 可审查性：高
- 审查规则建议：无符号整型判零应使用== 0而非!运算符

### ec1602f4 fix assert aicpu ut
- 根因类别：UT构建未统一接入框架
- 涉及文件：cmake/ut.cmake, control/assert/CMakeLists.txt, control/assert/tests/ut/op_kernel_aicpu/
- 缺陷描述：assert算子的aicpu UT用独立CMakeLists而非仓库统一AddAicpuOpTestCase宏，多UT并行构建时冲突。修复删除独立CMake，改用统一宏注册。
- 修复模式：构建配置统一到框架标准流程
- 可审查性：低
- 审查规则建议：新增算子UT必须使用仓库统一的AddAicpuOpTestCase宏注册

### b0803c3a index/scatter_add/README.md中失效链接修正
- 根因类别：非代码缺陷(文档链接失效)
- 涉及文件：index/scatter_add/README.md
- 缺陷描述：README中引用文档的相对路径从common/改为../../docs/zh/context/。
- 修复模式：文档路径修正
- 可审查性：高
- 审查规则建议：CI加入文档链接有效性检查

### ce4d538f fix mse_loss tilingdata bug
- 根因类别：host/kernel tiling数据结构不一致
- 涉及文件：loss/mse_loss/op_host/arch35/mse_loss_tiling_arch35.cpp, mse_loss_tiling_arch35.h, mse_loss_tiling_def.h
- 缺陷描述：kernel侧自定义ReduceOpTilingDataV2结构体与框架标准ReduceOpTilingData字段布局不同，host侧手动逐字段拷贝转换容易遗漏导致tiling数据错乱。修复删除自定义结构体，统一使用框架标准类型。
- 修复模式：消除冗余自定义数据结构，统一使用框架标准类型
- 可审查性：中
- 审查规则建议：禁止算子侧重新定义框架已提供的标准数据结构；host和kernel的tiling结构体必须来自同一头文件

### 6a789b7c fix dx nullptr
- 根因类别：空指针解引用
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：计算dw时gradInput可能为nullptr(不需要dx)，但用gradInput->GetViewShape()计算mmDwOutShape2d导致core dump。修复改用gradWeight->GetViewShape()，语义上也更正确。
- 修复模式：将null对象引用替换为正确的非null对象
- 可审查性：高
- 审查规则建议：可选输出tensor使用前必须做空指针检查；计算某输出的shape应引用语义正确的tensor

### 52a4c5f2 fix dynamic_quant_update_scatter_v2 aclnn & readme
- 根因类别：接口/配置冗余
- 涉及文件：README.md及aclnn相关文件（纯文档/配置删除，无代码文件变更）
- 缺陷描述：算子无aclnn调用模式，但自动生成了aclnn接口文件，导致接口冗余和用户困惑
- 修复模式：删除自动生成的无效aclnn接口
- 可审查性：低
- 审查规则建议：新算子上线前确认是否需要aclnn接口，避免自动生成无效接口

### 4063f8f9 修复aclnnTransposeBatchMatMul.md中的样例精度问题
- 根因类别：文档demo代码错误
- 涉及文件：matmul/transpose_batch_mat_mul/examples/arch35/test_aclnn_transpose_batch_mat_mul.cpp
- 缺陷描述：示例代码输出全0，原因是FP16数据处理和初始化有误。修复完全重写了示例，添加了正确的FP16转换逻辑(Fp16ToFloat)
- 修复模式：重写示例代码，使用正确的数据类型处理
- 可审查性：中
- 审查规则建议：算子示例代码必须经过实际运行验证输出正确性

### b39a4d45 fix package size
- 根因类别：binary配置缺失
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：HardSwishGradV2算子未在ascendc_config.json中注册，导致包大小异常
- 修复模式：添加算子配置项(含compute_units和auto_sync设置)
- 可审查性：中
- 审查规则建议：新增算子必须同步更新binary配置文件

### 38bb9260 修复TransQuantParamsV2二进制增幅问题
- 根因类别：binary配置平台覆盖不全
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：TransQuantParamV2的compute_units缺少ascend310p，该平台编译时使用了默认(auto_sync=true)配置，导致二进制文件异常增大
- 修复模式：在compute_units列表中添加ascend310p
- 可审查性：高
- 审查规则建议：算子新增平台支持时必须同步更新binary配置中的compute_units列表；auto_sync设置对二进制大小影响显著

### ab465d7d 增加SetScheduleMode多核同步，修复logit算子NaN bug，增加SmoothL1LossGradV2 inferdtype
- 根因类别：(1)多核同步缺失 (2)特殊值(NaN)处理缺失 (3)inferdtype缺失
- 涉及文件：loss/logit/op_kernel/logit.h, loss/chamfer_distance_grad/op_host/chamfer_distance_grad_tiling.cpp, loss/ctc_loss_v2/op_host/arch35/ctc_loss_v2_tiling_arch35.cpp, loss/mse_loss_v2/op_host/mse_loss_v2_tiling.cpp, loss/smooth_l1_loss_grad_v2/op_graph/smooth_l1_loss_grad_v2_proto.h 等
- 缺陷描述：
  (1) chamfer_distance_grad/ctc_loss_v2/mse_loss_v2等算子使用多核但缺少SetScheduleMode(BATCH_MODE)设置，导致核间同步问题
  (2) logit算子在输入包含NaN时未保留NaN语义，输出结果错误。修复添加Compare(EQ, self, self)检测NaN(NaN!=NaN)，再用Select保留NaN值
  (3) SmoothL1LossGradV2缺少proto定义导致inferdtype功能缺失
- 修复模式：(1)添加SetScheduleMode调用 (2)添加NaN检测mask和Select操作 (3)新增proto头文件
- 可审查性：高
- 审查规则建议：多核算子必须设置SetScheduleMode；处理浮点数据的算子必须考虑NaN/Inf输入场景；NaN检测模式 = Compare(x, x, EQ)

### 3d35698f ascend_quant、ascend_quant_v2、dynamic_quant 问题修复
- 根因类别：平台判断方式不当（SocVersion枚举 -> NpuArch架构族）
- 涉及文件：quant/ascend_quant/op_api/aclnn_ascend_quant.cpp, quant/ascend_quant_v2/op_host/ascend_quant_v2_tiling.cpp, quant/ascend_quant_v2/op_host/op_api/aclnn_ascend_quant_v3.cpp
- 缺陷描述：使用SocVersion::ASCEND910B等具体芯片型号判断平台能力，新芯片加入时需逐一添加case。迁移到NpuArch::DAV_2201等架构族判断后，同架构的新芯片自动支持
- 修复模式：将SocVersion switch替换为NpuArch switch；常量名从平台名(ASCEND910B)改为能力名(WITH_BF16)；使用IsRegbaseSocVersion()工具函数
- 可审查性：高
- 审查规则建议：禁止使用SocVersion枚举判断平台能力，改用NpuArch架构族；变量名应反映能力而非平台名

### dd15791e 修复experimental脚本问题
- 根因类别：CI脚本工作目录错误
- 涉及文件：scripts/ci/check_experimental_example.sh, scripts/ci/check_experimental_pkg.sh
- 缺陷描述：脚本先cd到BUILD_PATH执行cmake，之后调用${BASE_PATH}/scripts下的python脚本时仍在BUILD_PATH下，导致相对路径解析错误
- 修复模式：在cmake步骤后添加cd "${BASE_PATH}"恢复工作目录
- 可审查性：高
- 审查规则建议：shell脚本中cd后的后续命令需要检查工作目录假设是否成立；优先使用pushd/popd或绝对路径

### 647010ea 增加fixpipe优化
- 根因类别：性能优化/API迁移（非缺陷修复）
- 涉及文件：matmul/common/cmct/block/block_mmad_builder.h, block_mmad_iterbatch.h, kernel_matmul_without_que.h
- 缺陷描述：为MatmulIterBatch添加ND_FIXPIPE_1_2 dispatch策略支持，新增L0C到UB的Fixpipe CopyOut路径。同时在kernel执行前后添加SetMMLayoutTransform(true/false)调用
- 修复模式：扩展模板特化，合并重复代码，新增fixpipe输出路径
- 可审查性：低
- 审查规则建议：新增dispatch策略时检查所有相关Builder类是否已适配

### 6e5da556 修复 wqbmmv2 和 qbmmv4 在子包编译失败的问题
- 根因类别：binary配置属性缺失
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/config/ascend910_95/weight_quant_batch_matmul_v2_binary.json
- 缺陷描述：binary.json中所有配置段落均缺少inner_precise属性，导致opp_kernel编译失败。需要在每个属性列表末尾添加inner_precise字段
- 修复模式：在每个kernel配置段添加 {"name": "inner_precise", "dtype": "int", "value": 0}
- 可审查性：高
- 审查规则建议：binary.json配置模板应包含所有必需属性的检查清单；新增平台配置时必须包含inner_precise等必需属性

### 1641e3af fix offset range
- 根因类别：整数溢出（uint32->uint64）
- 涉及文件：norm/add_rms_norm_cast/op_kernel/add_rms_norm_cast.h, norm/add_rms_norm_cast/op_kernel/add_rms_norm_cast_multi_n.h, norm/add_rms_norm_dynamic_quant/op_host/op_api/aclnn_add_rms_norm_dynamic_quant_v2.cpp
- 缺陷描述：gm_bias使用uint32_t类型，当i_o * rowFactor * numCol乘积超过2^32时溢出，导致GM地址计算错误。修复将gm_bias改为uint64_t，所有乘法操作数都static_cast<uint64_t>
- 修复模式：类型提升uint32_t -> uint64_t，贯穿整个调用链(CopyIn/Compute/CopyOut参数)
- 可审查性：高
- 审查规则建议：GM offset/地址计算必须使用uint64_t；检查所有uint32_t offset与tensor维度的乘积是否可能溢出

### 864d2df1 fix group_norm_silu
- 根因类别：文档/示例代码错误
- 涉及文件：norm/group_norm_silu/examples/test_aclnn_group_norm_silu_v2.cpp
- 缺陷描述：GroupNormSiluV2示例代码使用了旧版API(aclnn_group_norm_silu而非V2接口)，且校验信息不完整
- 修复模式：重写示例代码使用正确的V2 API
- 可审查性：中
- 审查规则建议：算子版本升级后必须同步更新示例代码中的API引用

### f90f38c0 fix gng copy problem for new interface
- 根因类别：DMA搬运接口使用不当导致OOM
- 涉及文件：norm/group_norm_grad/op_kernel/arch35/group_norm_grad_recompute.h
- 缺陷描述：DataCopy要求数据大小block对齐，当mainReduceNum/mode2UbTailNum_不满足对齐要求时，会读取超出buffer范围的内存导致OOM。修复改用DataCopyPad接口，显式设置blockLen=实际字节数，自动处理非对齐情况
- 修复模式：DataCopy替换为DataCopyPad(配合DataCopyExtParams和DataCopyPadExtParams)
- 可审查性：高
- 审查规则建议：非block对齐的数据搬运必须使用DataCopyPad而非DataCopy；审查DataCopy调用时检查数据量是否保证block对齐

### ab4ce144 aclnn_quant_matmul_v5 infer groupsize bugfix
- 根因类别：维度索引混淆（转置场景）
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：InferGroupSize中非MicroScaling场景，transX1时应取penultimate dim但取了last dim，非transX1时反之。MicroScaling场景还漏乘了2(e8m0格式scale shape为[m,k/2,2])
- 修复模式：交换transX1/非transX1的维度索引取值；MicroScaling场景scaleSizeK乘2
- 可审查性：高
- 审查规则建议：转置场景的维度索引取值必须逐case验证；scale shape推导需考虑数据格式编码(e8m0等)

### 8277682 修复Matmul切换低阶API导致的性能劣化
- 根因类别：API迁移遗漏（SetMMLayoutTransform调用缺失）
- 涉及文件：matmul/common/cmct/kernel/kernel_matmul_without_que.h
- 缺陷描述：从高阶Matmul API切换到低阶API实现后，未调用SetMMLayoutTransform(true)设置Fixpipe UnitFlag搬运方向，导致搬运效率降低、并行度变差、性能下降
- 修复模式：在Init前添加SetMMLayoutTransform(true)，在计算完成后添加SetMMLayoutTransform(false)
- 可审查性：中
- 审查规则建议：API迁移时必须对比原API内部行为，确认所有隐式设置已在新代码中显式调用

### ab60dc45 generate basic block table in compile time
- 根因类别：代码质量改进（硬编码消除，非缺陷修复）
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/arch35/weight_quant_batch_matmul_v2_basic_block_table.h
- 缺陷描述：~1000行硬编码的BasicBlock表替换为constexpr编译期生成，消除了维护困难和不一致风险
- 修复模式：使用constexpr模板函数在编译期计算block表
- 可审查性：低
- 审查规则建议：可通过算法生成的常量表应使用constexpr或代码生成，避免手工维护

### e62e7b4d 修复fusedmatmul带bias场景的性能问题
- 根因类别：tiling计算未考虑bias占用L1内存
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_asw_tiling.cpp, matmul_v3_basic_aswt_tiling.cpp
- 缺陷描述：stepK计算使用完整L1 size，但bias场景下L1中已有bias table占用空间(BIAS_TABLE_NUM * DATA_SIZE_FP32)。stepK过大导致实际搬运超出可用L1空间，性能劣化
- 修复模式：计算remainSizeForAL1BL1时扣除bias table占用，用此值重新计算stepKa/stepKb
- 可审查性：高
- 审查规则建议：tiling参数计算必须考虑所有内存占用(bias/workspace/临时buffer)，不应假设整个buffer都可用

### b2e2cada addbmm&baddbmm空tensor入参错误修复
- 根因类别：复制粘贴错误（参数重复）
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp, aclnn_baddbmm.cpp
- 缺陷描述：isAddBmmProcessEmptyTensor(batch1, batch1)和isProcessEmptyTensor(batch1, batch1)中第二个参数应为batch2。当batch2有空维度时无法检测到，继续参与计算导致错误
- 修复模式：将第二个batch1改为batch2
- 可审查性：高
- 审查规则建议：函数调用中两个参数相同(f(a,a))时必须确认是否为复制粘贴错误；代码审查时重点检查参数名相似的调用

### 68a1137d fix norm common cmake change and fix addRmsNormCast infershape
- 根因类别：infershape未支持unknown rank
- 涉及文件：norm/add_rms_norm/tests/ut/op_host/test_AddRmsNorm_infershape.cpp, test_add_rms_norm_tiling.cpp, norm/add_rms_norm_cast相关文件
- 缺陷描述：AddRmsNorm infershape不支持unknown rank场景(shape={-2})，新增UT覆盖此场景。同时修复cmake构建配置
- 修复模式：添加unknown rank UT用例，修复infershape逻辑
- 可审查性：中
- 审查规则建议：infershape实现必须覆盖unknown rank(-2)和unknown dim(-1)场景

### e76fa0ee fix matmul bug
- 根因类别：(1)错误日志变量引用错误 (2)硬件同步API不规范 (3)AFullLoad条件过严
- 涉及文件：matmul/fused_quant_mat_mul/op_host/.../fused_quant_matmul_checker.cpp, fused_quant_matmul_swiglu_tiling.cpp, fused_quant_mat_mul_swiglu.h, matmul/quant_batch_matmul_v3/op_host/.../adaptive_sliding_window_tiling.cpp
- 缺陷描述：
  (1) 错误日志中打印scaleShape的维度值时使用了offsetShape变量，输出错误的维度信息
  (2) FusedQuantMatmulSwiglu使用raw指令(set_flag/wait_flag)做FIX_V同步，不符合API规范
  (3) AFullLoad判断强制要求mBlockCnt<=nBlockCnt，在supportMmadS8S4平台上过于严格，限制了优化机会
- 修复模式：(1)修正变量名 (2)替换为AscendC::SetFlag/WaitFlag + FetchEventID (3)s8s4平台跳过blockCnt比较条件
- 可审查性：高
- 审查规则建议：日志和错误信息中的变量引用必须与上下文一致；使用封装API而非raw指令；条件判断需考虑平台差异

### eaebf6db SwishGrad算子精度修复
- 根因类别：(1)未初始化变量/变量引用错误 (2)TilingData类型不匹配
- 涉及文件：activation/swish_grad/op_host/arch35/swish_grad_tiling_arch35.cpp, swish_grad_tiling_arch35.h, activation/swish_grad/op_kernel/arch35/swish_grad_tilingdata.h
- 缺陷描述：
  (1) tilingKey计算使用成员变量schMode，但schMode未从tiling数据中正确赋值。修复改用tiling->baseTiling.scheMode
  (2) TilingData中scale使用int64_t类型，但实际数据是float，类型不匹配导致精度错误
  (3) TilingData中存在无用的int64_t value字段，浪费空间
- 修复模式：(1)直接引用tiling结构体字段而非中间变量 (2)scale类型从int64_t改为float (3)删除无用字段
- 可审查性：高
- 审查规则建议：TilingData结构体字段类型必须与实际数据语义匹配；tilingKey计算的每个输入都必须来自可靠数据源

## 第11轮 (commit 201-220)

### 8e6718595bb841d0239f3a5b5ab877b329d219cd wqbmm kernel config file fix in master
- 根因类别：API迁移不完整 + binary配置缺失
- 涉及文件：matmul/common/op_host/op_api/batch_matmul_util.cpp, matmul/weight_quant_batch_matmul_v2/op_host/config/ascend310p/weight_quant_batch_matmul_v2_binary.json
- 缺陷描述：(1) TransBmm2Mm函数参数enableHf32(bool)需更新为opImplModeEnum(int64_t)以适配MatMulV3Nd新接口，旧代码传入bool导致语义错误；(2) binary配置JSON缺少所有input/output的paramType字段(required/optional)
- 修复模式：参数类型从bool改为int64_t枚举，调用点构造枚举值(enableHf32?0x40:enableForceGrp?0x4:0x1)；JSON补全paramType字段
- 可审查性：中 —— API变更需查阅接口文档，但binary配置缺失paramType可通过模板对比发现
- 审查规则建议：L0 API升级时全量检查所有调用点的参数类型和语义；binary配置JSON必须包含所有paramType字段

### 3b93f190f8461c109edd6af5e7a36904dbef1219 fix: a16f8 n=1 not supported
- 根因类别：边界条件未覆盖
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/weight_quant_batch_matmul_v2_tiling.cpp
- 缺陷描述：A16F8模式仅支持per-channel量化，但当antiquant_scale的元素数量为1(即n=1)时，tiling将其判定为per-tensor场景并报不支持。文档要求per-channel的scale shape为(n,)，n=1是合法输入但被错误拒绝
- 修复模式：新增ConfigureReuseScenarios()方法，检测A16F8+per-tensor组合时自动复用per-channel逻辑
- 可审查性：高 —— 边界值n=1是典型的边界测试case
- 审查规则建议：量化模式切换逻辑必须覆盖参数为最小合法值(如n=1)的场景

### e24af7fe239d16c46ece22e2fbdc666bbab956b4 fix warning log + fix SetScheduleMode
- 根因类别：多核调度模式遗漏 + API误用
- 涉及文件：activation/gelu_quant/op_kernel/gelu_quant.cpp, quant/swi_glu_quant/op_host/swi_glu_quant_tiling.cpp
- 缺陷描述：(1) gelu_quant kernel入口多余调用SetSysWorkspace(workspace)，实际只需GetUserWorkspace；(2) swi_glu_quant tiling缺少SetScheduleMode(BATCH_MODE)调用，导致多核调度策略不正确
- 修复模式：删除多余SetSysWorkspace调用；tiling函数开头添加SetScheduleMode(BATCH_MODE)
- 可审查性：高 —— SetScheduleMode是多核算子的必备配置，可通过checklist检查
- 审查规则建议：多核算子tiling必须显式调用SetScheduleMode，kernel入口不应调用SetSysWorkspace(由框架管理)

### 0e10d3883cc0bd5a3ed6cd78af1617c32d5ca86e dynamic_block_quant、trans_quant_param_v2算子问题修复
- 根因类别：功能参数未应用 + 错误码语义错误
- 涉及文件：quant/dynamic_block_quant/op_host/dynamic_block_quant_i8_tiling.cpp, quant/dynamic_block_quant/op_kernel/dynamic_block_quant.h, quant/trans_quant_param_v2/op_host/op_api/aclnn_trans_quant_param_v2.cpp
- 缺陷描述：(1) dynamic_block_quant新增minScale参数但kernel未实际应用，量化时未对scale做min/max clamp导致极小scale产生溢出；(2) trans_quant_param_v2参数校验返回ACLNN_ERR_INNER_NULLPTR(内部错误)而非ACLNN_ERR_PARAM_NULLPTR(参数错误)，错误码语义不匹配
- 修复模式：kernel中添加hasMinScale判断及Min/Max clamp逻辑；修正错误码为ACLNN_ERR_PARAM_NULLPTR
- 可审查性：高 —— 新增参数必须端到端验证(host tiling -> kernel使用)；错误码应与实际错误类型匹配
- 审查规则建议：新增算子参数时，验证host->tiling->kernel全链路均已使用该参数；错误码必须与错误类型(参数/内部/资源)匹配

### 93f6d5b0563624db3ab90cb4501cad38e07fa65e fix layer norm v4 last reduce add mask problem
- 根因类别：尾块mask逻辑错误导致精度问题
- 涉及文件：norm/layer_norm_v4/op_kernel/arch35/layer_norm_v4_two_pass_perf.h
- 缺陷描述：LAST_LOOP_NUMS==2时，第二块x2与x1直接Add后用pregLast做ReduceSum，但pregLast掩码基于lastBinaryAddNum计算，对两块场景掩码范围错误。实际上x2可能包含超出有效范围的脏数据，直接Add会污染结果
- 修复模式：用ShiftLefts将x2的有效部分(lastTailNum = lastBinaryAddNum-VL_B32)移入寄存器并零填充无效位，再与x1做Add，最后用pregFull做ReduceSum
- 可审查性：中 —— 需要理解两轮归约的数据布局和掩码语义
- 审查规则建议：多块归约的尾块处理必须确保无效数据不参与计算，mask范围需与实际有效数据量一致

### 15530399173292672c6356ac5f4d3efef56ae5a4 fix conv 精度问题
- 根因类别：条件分支覆盖不全 + 数据类型提升缺失
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：(1) needSpecialCast条件缺少storageShapeDimSize > 1检查，当维度为1时访问dim(storageShapeDimSize-1)虽合法但逻辑不正确；(2) Conv1D->2D转换时dilation=45需特殊处理exchangeDim；(3) 特定SoC+case组合需要cast到float提升精度但未做
- 修复模式：添加storageShapeDimSize>1条件；新增DILATION_45常量和isConv2dToCastFloat白名单检查；exchangeDim根据groups和dilation条件选择
- 可审查性：中 —— 需要了解卷积反向的精度特性和平台差异
- 审查规则建议：卷积反向的数据类型提升逻辑需覆盖所有SoC平台；Conv1D->2D转换时groups和dilation的组合场景需穷举

### 9031da227ce5a92427ddf6fcda302f4ffcace286 fix allocated oversize of FusedLinearCrossEntropyLossGrad
- 根因类别：workspace大小计算错误(过大)
- 涉及文件：matmul/fused_linear_cross_entropy_loss_grad/op_host/fused_linear_cross_entropy_loss_grad_tiling.cpp
- 缺陷描述：workspace计算中BT*H和V*H两项各乘了BUFFER_NUM(=2)，但这两个buffer不需要双缓冲，导致workspace申请量是实际需求的2倍，浪费内存
- 修复模式：去掉两项的*BUFFER_NUM，添加日志打印workspace大小
- 可审查性：高 —— BUFFER_NUM的使用应与实际缓冲策略(单缓冲/双缓冲)一致，可通过审查每个buffer的用途判断
- 审查规则建议：workspace各项的BUFFER_NUM倍数必须与实际缓冲策略匹配，过度申请虽不影响正确性但浪费资源

### ecd6c75824b458d0e2ea624a3eeb73be1a801815 8.5 bug fix 合入ops-nn
- 根因类别：tiling workspace维度限制未细化 + 内存限制缺失
- 涉及文件：index/index_put_v2/op_api/aclnn_index_put_impl.cpp, index/index_put_with_sort/op_host/op_api/index_put_with_sort.cpp, index/scatter_elements_v2/op_host/op_api/aclnn_scatter.cpp, tests
- 缺陷描述：(1) index_put_v2对高维(5-8维)大索引数量，原60M上限过大导致tiling内部失败，需按维度细化限制(54M/48M/44M/40M)；(2) index_put_with_sort对fp16数据升精度后总内存可能超250MB但未检查；(3) scatter的reduce参数非法值无告警；(4) UT类名重复导致编译冲突
- 修复模式：按维度分级设置indices数量上限；添加内存上限检查(250MB)；添加reduce参数校验告警；重命名UT类
- 可审查性：高 —— tiling限制应与实际workspace容量匹配，内存限制应有上界检查
- 审查规则建议：高维场景的tiling限制必须按维度递减；数据类型提升(fp16->fp32)需检查内存膨胀是否超限

### 82c38b544653771df3ec12edcafa47ef5563896f 修复ConvTbcBackward算子文档错误
- 根因类别：文档错误
- 涉及文件：纯文档修改
- 缺陷描述：文档内容修正
- 可审查性：低
- 审查规则建议：N/A

### c51de510ac71135691fbdbdfa638af2e97a9e0a7 修复matmul目录下example内存泄漏问题
- 根因类别：Example代码资源泄漏
- 涉及文件：matmul/batch_mat_mul_v3/examples/test_aclnn_addbmm.cpp 等多个example文件
- 缺陷描述：example代码中aclTensor、device memory、workspace等资源在错误路径上未释放，存在内存泄漏。用户参考example编写应用时会继承同样的问题
- 修复模式：使用unique_ptr+自定义deleter(aclDestroyTensor, aclrtFree)实现RAII
- 可审查性：高 —— 裸指针分配后无RAII包装是明显的泄漏模式
- 审查规则建议：example代码中所有acl资源分配必须使用RAII或配对释放

### 6623da3fdfea838416be8bd6c0bed20384fd3807 修复experimental存在同名算子导致编译失败的问题
- 根因类别：构建系统搜索路径错误
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：CMake中find命令搜索CMAKE_CURRENT_SOURCE_DIR(整个源码目录)而非OP_DIR(当前算子目录)，当experimental目录存在同名算子时找到错误的def.cpp，导致编译逻辑判断失败
- 修复模式：搜索路径从CMAKE_CURRENT_SOURCE_DIR改为OP_DIR
- 可审查性：高 —— CMake搜索路径应限定在目标目录，全局搜索存在同名碰撞风险
- 审查规则建议：CMake find/搜索命令的路径必须限定为目标算子目录，禁止全局搜索

### 42e15c0bb4be44af8e6020b3d0aaca513f891b0e 问题修复，leaky_relu输入为float16报错
- 根因类别：tiling key未设置
- 涉及文件：activation/leaky_relu/op_host/arch35/leaky_relu_tiling_arch35.cpp
- 缺陷描述：leaky_relu算子当输出dtype为FLOAT16时，进入fp16->fp32 cast处理分支，但dType变量未被设置为TPL_FP16，导致tiling key错误，运行时找不到匹配的kernel
- 修复模式：在fp16分支开头添加dType = static_cast<uint64_t>(TPL_FP16)
- 可审查性：高 —— 每个dtype分支必须设置对应的tiling key，遗漏可通过分支检查发现
- 审查规则建议：tiling中每个dtype分支必须设置对应的tilingKey/dType变量

### 179ab8a17bfa59e5076d99f58952ea183e04ebae 修复卷积反向bf16精度问题
- 根因类别：BF16数据类型路径处理不当
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：卷积反向的needSpecialCast路径原始逻辑统一cast到fp16再transdata再cast回目标dtype，但BF16输出时中间经fp16会丢失精度。需区分float和bf16输出：float走fp16中转再cast回；bf16直接cast到目标dtype再transdata
- 修复模式：在needSpecialCast分支内按outputDtype是float还是其他(bf16)分两条路径
- 可审查性：高 —— 数据类型中转路径必须考虑所有支持的输出dtype
- 审查规则建议：Cast中转路径需确保中间dtype的精度不低于目标dtype；BF16与FP16是不同精度范围，不可互相中转

### f06cb5dbc7ba729cba65bfe1f35847880625dd99 修复kernel编译报错问题
- 根因类别：构建系统搜索路径错误 + 存在性检查缺失
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：(1) 与#211相同的搜索路径问题(CMAKE_CURRENT_SOURCE_DIR -> OP_DIR)；(2) 遍历算子目录时未检查op_host/{name}_def.cpp和op_kernel目录是否同时存在，缺一时执行错误的编译逻辑
- 修复模式：修正搜索路径；添加def.cpp和op_kernel目录的存在性检查，缺少任一则跳过
- 可审查性：高 —— 构建脚本应对文件存在性做防御性检查
- 审查规则建议：构建系统遍历算子时必须验证def+kernel成对存在

### 14b9cdb8b509095c1c43692f8a39029f8a420789 quant_matmul hard sync bugfix
- 根因类别：多核sub-block守护缺失 + 事件同步时序错误
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_bf16.h, quant_batch_matmul_v3_bf16_basic.h, quant_batch_matmul_v3_pertoken.h
- 缺陷描述：(1) SplitK场景下GetSubBlockIdx()==1的sub-block应跳过执行，但Init和Process函数只检查blockIdx_>=usedCoreNum_未检查sub-block；(2) AIC事件等待条件loop_>2/loop_>3过于宽松，在loop_<=2时跳过了必要的WaitEvent，导致C2V同步丢失
- 修复模式：添加GetSubBlockIdx()==1守护到Init和Process；将WaitEvent条件改为loop_>0和loop_>1
- 可审查性：高 —— SplitK场景必须检查sub-block index；事件同步的Wait/Set必须配对
- 审查规则建议：SplitK/多sub-block场景中，非主sub-block必须在Init和Process入口处跳过；AIC/AIV事件同步的Wait条件必须确保不跳过任何Set

### 98254481e5d40449491bdee6da23217b09104652 修正MaxPoolV3精度问题和opapi告警
- 根因类别：大kernel精度劣化 + 编译告警
- 涉及文件：pooling/max_pool_v3/op_api/aclnn_max_pool.cpp
- 缺陷描述：MaxPoolV3在kernel size > 1000时存在精度问题(数值范围过大导致比较/归约精度下降)。另有函数返回值const修饰多余的告警
- 修复模式：ksize>1000时路由到MaxPool3dWithArgmaxV2(精度更好)；移除函数返回值多余的const
- 可审查性：中 —— 需要了解不同pooling实现的精度特性
- 审查规则建议：大参数范围(kernel size/stride极大)需验证精度是否满足要求

### 2e389cbb0e6d1eaf31abc8aba165cc25d4b0d9aa aclnnSwiglu.md文档修复
- 根因类别：文档错误
- 涉及文件：纯文档修改
- 缺陷描述：文档内容修正
- 可审查性：低
- 审查规则建议：N/A

### 83c7e7c55d8994cbacc405e2378489b3644f5609 修复fake_quant_affine_cachemask算子精度问题
- 根因类别：特殊值(NaN)处理逻辑错误
- 涉及文件：quant/fake_quant_affine_cachemask/op_kernel/fake_quant_affine_cachemask_fp16.h, fake_quant_affine_cachemask_fp32.h
- 缺陷描述：原代码对NaN输入的处理有误：(1) Compare(x,x,EQ)利用NaN!=NaN特性检测NaN，但后续Select将NaN位置替换为quantMin值，语义错误(应为0)；(2) 额外的Compare(y,infTensor,NE)检测Inf逻辑多余且与quantMin的Select组合效果不正确
- 修复模式：简化为单次Compare(x,x,EQ)+Select(result,0.0f)，NaN位置输出0而非quantMin，删除多余的Inf检测
- 可审查性：高 —— NaN/Inf处理逻辑应有明确的数学语义说明
- 审查规则建议：特殊值处理(NaN/Inf/零)的Select替换值必须与数学定义一致；每个Compare+Select组合需注明其语义

### aa4a6df285df70bfbfde226fdbe7247d87274c8e Delete invalid versions
- 根因类别：无效平台配置残留
- 涉及文件：matmul/batch_mat_mul_v3/op_host/batch_mat_mul_v3_def.cpp, matmul/fused_quant_mat_mul/op_graph/fused_quant_mat_mul_proto.h
- 缺陷描述：batch_mat_mul_v3的def.cpp中残留无效的mc62cm12a SoC配置(与ascend910_95完全重复)；fused_quant_mat_mul的proto.h文件已废弃未删除
- 修复模式：删除无效SoC配置块和废弃文件
- 可审查性：高 —— 重复的SoC配置块和废弃文件可通过代码审查发现
- 审查规则建议：新增SoC配置前检查是否存在同内容的重复配置；废弃的proto/header文件应及时清理

### f63ed7eb39b2c1bbe79f22c7f35ea9b11f5c514e 修复lng的tiling告警
- 根因类别：编译告警(未使用变量 + 有符号/无符号比较)
- 涉及文件：norm/layer_norm_grad_v3/op_host/layer_norm_grad_v3_grouped_reduce_big_m_tiling.cpp, layer_norm_grad_v3_recompute_tiling.cpp
- 缺陷描述：maxBlocks和mCacheBufferCountStg2变量定义后未使用；uint64_t与int64_t直接比较产生有符号/无符号比较告警
- 修复模式：删除未使用变量；添加static_cast<uint64_t>转换
- 可审查性：高 —— 编译器告警应零容忍
- 审查规则建议：所有变量定义后必须使用；有符号与无符号比较需显式转换

## 第12轮 (commit 221-240)

### eb531f0bce57 fix index_put_v2 opapi UT
- 根因类别：构建配置/依赖缺失
- 涉及文件：index/index_put_v2/CMakeLists.txt
- 缺陷描述：index_put_v2的CMakeLists中DEPENDENCIES只声明了index，缺少index_put_with_sort和linear_index_v2，导致opapi UT编译链接失败
- 修复模式：在add_modules_sources的DEPENDENCIES中补充完整依赖项
- 可审查性：中（需要了解算子间依赖关系）
- 审查规则建议：新增算子CMakeLists时验证DEPENDENCIES是否包含所有被引用的算子模块

### 5ca5cc755f08 修复最新包编译DeformableConv2d编译报错的问题
- 根因类别：头文件依赖缺失
- 涉及文件：conv/deformable_conv2d/op_kernel/deformable_conv2d_base.h
- 缺陷描述：deformable_conv2d_base.h只include了lib/matmul_intf.h，缺少kernel_operator.h。上游包更新后该隐式依赖断裂，导致编译报错
- 修复模式：显式添加#include "kernel_operator.h"
- 可审查性：高（检查每个头文件是否自包含所有直接依赖）
- 审查规则建议：头文件必须显式include所有直接使用的依赖，不依赖间接传递

### 50df91e72ce9 修复foreach算子使用GlobalTensor offset数据类型不匹配
- 根因类别：整数溢出/类型不一致(int32->uint64)
- 涉及文件：foreach/foreach_copy/op_kernel/foreach_copy.h, foreach/foreach_lerp_list/op_kernel/foreach_lerp_list.h, foreach/foreach_lerp_scalar/op_kernel/foreach_lerp_scalar.h, foreach/foreach_non_finite_check_and_unscale/op_kernel/*.h, foreach/foreach_norm/op_kernel/*.h, foreach/foreach_pow_list/op_kernel/foreach_pow_list.h, foreach/foreach_pow_scalar_and_tensor/op_kernel/foreach_pow_scalar_and_tensor.h
- 缺陷描述：GlobalTensor的下标表达式`index * maxDataCount`中index为uint16_t/uint32_t、maxDataCount为int64_t，但乘法在较小类型域执行可能溢出。大tensor偏移超过32位范围时产生越界访问。涉及7个foreach算子的约20处相同模式
- 修复模式：在乘法前插入`1ULL *`强制提升为uint64_t运算：`inTensorsGM[1ULL * index * maxDataCount]`
- 可审查性：高（机械检查GM[]下标表达式是否包含64位提升）
- 审查规则建议：GM内存偏移计算必须使用int64/uint64类型，乘法表达式中至少一个操作数为64位

### 1a87986e53f9 [MMV3] 回退对齐场景下workspace申请逻辑，解决对齐场景内存膨胀
- 根因类别：workspace内存计算公式错误
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp, matmul_v3_base_tiling.h
- 缺陷描述：deterministic splitk场景的workspace计算统一使用alignedM*alignedN作为singleSize，但对齐场景(TilingEnableFixOpti::BASE)下应使用singleCoreN*singleCoreM（未对齐值），导致workspace过度膨胀
- 修复模式：提取GetDeterministicSplitKWorkspaceSize函数，区分BASE场景使用singleCore值、其他场景使用aligned值
- 可审查性：中（需理解对齐策略差异）
- 审查规则建议：workspace大小计算需区分不同tiling策略的对齐需求，避免过度分配

### d6c44e809a4a fix muls cast
- 根因类别：数据类型转换路径错误
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_batch_matmul.cpp
- 缺陷描述：baddbmm算子在混精度路径(fp16/bf16 input + CubeMathType==KEEP_DTYPE + 910B/910C)下，transdata输出类型已是FP32，但Cast仍使用out->GetDataType()目标类型(可能为fp16/bf16)，导致精度丢失
- 修复模式：增加enableFp16Bf16InFp32Out条件判断，混精度路径下Cast目标改为DT_FLOAT
- 可审查性：高（混精度路径中检查Cast目标类型是否与实际中间结果类型一致）
- 审查规则建议：混精度/高精度计算路径中的Cast操作，目标类型必须与上游实际输出类型匹配

### 2098f3961e0c fix run_example
- 根因类别：构建脚本路径过宽
- 涉及文件：build.sh
- 缺陷描述：build_example函数中find命令搜索example cpp文件时，会误匹配opgen/template目录下的模板文件，导致模板example也被编译执行
- 修复模式：find命令添加`-not -path "*/opgen/template/*"`排除模板目录
- 可审查性：高（审查find/glob命令的搜索范围是否精确）
- 审查规则建议：构建脚本中的文件搜索命令需显式排除模板/生成代码目录

### 7a453b33047d 修复图模式example cmakelist并且对齐文本描述
- 备注：纯文档修改(docs/zh/invocation/op_invocation.md)，无代码缺陷，跳过

### 78fe4d709a7d fix:auto default true
- 根因类别：shell脚本默认值/条件逻辑错误
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op_gen_task.sh
- 缺陷描述：auto_sync变量默认值设为"false"，但系统约定auto_sync默认应为true(不传即为true)。同时条件判断`!= "true"`过于宽泛（空值也会触发），应改为`== "false"`精确匹配
- 修复模式：默认值从"false"改为""(空)，条件判断从`!= "true"`改为`== "false"`
- 可审查性：高（检查默认值语义是否与系统约定一致）
- 审查规则建议：shell脚本的布尔变量默认值必须与系统约定一致；字符串比较优先使用精确匹配(== "value")而非否定匹配(!= "other_value")

### 27a2a8923a34 修复 convert_weight_to_int4_pack 编译 opapi 时的告警问题
- 根因类别：有符号/无符号比较 + 未使用参数告警
- 涉及文件：matmul/convert_weight_to_int4_pack/op_host/op_api/aclnn_convert_weight_to_int4_pack.cpp
- 缺陷描述：循环中int64_t变量g、k*n与size_t类型循环变量比较产生有符号/无符号告警；aclnnConvertWeightToINT4Pack函数参数未使用产生告警
- 修复模式：添加static_cast<size_t>()转换；未使用参数添加[[maybe_unused]]
- 可审查性：高（编译器告警即可发现）
- 审查规则建议：编译-Wsign-compare -Wunused-parameter告警视同错误处理

### 955f7dd5111a fix check ini
- 根因类别：CMake构建依赖缺失
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：add_ops_info_target_v1和merge_ini_files函数中，自定义命令缺少DEPENDS opbuild_custom_gen_aclnn_all声明，导致ini文件可能在aclnn生成完成前就被处理，产生时序依赖问题
- 修复模式：在add_custom_command中添加DEPENDS opbuild_custom_gen_aclnn_all
- 可审查性：高（检查CMake自定义命令是否声明了所有上游依赖）
- 审查规则建议：CMake add_custom_command必须在DEPENDS中声明所有前序生成步骤

### 02c0c42c21a6 修复命令bash build.sh -u --ophost报错问题
- 根因类别：目录结构重构后路径不一致
- 涉及文件：activation/swish/op_host/op_tiling/arch35/ -> activation/swish/op_host/arch35/ (重命名), cmake/func.cmake, 多个UT include路径
- 缺陷描述：tiling文件从op_host/op_tiling/arch35迁移到op_host/arch35后，UT中的相对include路径未同步更新。同时func.cmake中OP_HOST_UT条件下的GLOB_RECURSE扫描路径与新目录结构不兼容
- 修复模式：更新UT include路径(../../../ -> ../../../../)，删除cmake中不再适用的GLOB_RECURSE逻辑
- 可审查性：高（文件移动后检查所有引用路径）
- 审查规则建议：目录结构重构后必须全局搜索更新所有相对路径引用

### 611e857246e1 bugfix swi_glu_quant && inplace_add_rms_norm
- 根因类别：预编译宏条件反转
- 涉及文件：norm/inplace_add_rms_norm/op_kernel/inplace_add_rms_norm.cpp, quant/swi_glu_quant/op_kernel/swi_glu_quant.cpp
- 缺陷描述：inplace_add_rms_norm中bfloat16分支的条件写成`#if (defined(__NPU_ARCH__) && __NPU_ARCH__ == 3003)`，含义是"仅在3003上编译bf16"，但3003(Ascend310P)不支持bf16，应为"除3003外都编译bf16"即`#if !(...)`. swi_glu_quant中#if/#endif嵌套层级错误导致条件分支覆盖不全
- 修复模式：添加`!`取反条件；修正swi_glu_quant的#if条件合并为单一条件表达式
- 可审查性：高（平台条件宏审查：确认该平台是否实际支持该数据类型）
- 审查规则建议：平台条件编译宏中"排除某平台"应使用`#if !(defined(__NPU_ARCH__) && __NPU_ARCH__ == XXX)`模式，审查时验证条件语义是否与平台能力匹配

### d4a43b6f0cd5 fix warning
- 根因类别：未使用变量告警
- 涉及文件：matmul/quant_batch_matmul_v3/op_api/quant_matmul_checker.cpp
- 缺陷描述：x1Shape和x2Shape变量声明后未使用（相关逻辑可能已移至其他位置），产生编译告警
- 修复模式：删除未使用的变量声明
- 可审查性：高（编译器告警）
- 审查规则建议：代码重构后检查被移走逻辑的残留变量声明

### 8d718f35977c 修复了matmulv3在特定shape下的精度问题
- 根因类别：多核循环变量未重新初始化 + 条件分支不完整
- 涉及文件：matmul/mat_mul_v3/op_kernel/mat_mul_deterministic_splitk_kernel.h
- 缺陷描述：ReduceKInUbNzL2cache函数的双层循环中，mCoreUse/nCoreUse只在尾块条件`if (mIndex == mCnt-1 || nIndex == nCnt-1)`内赋值，非尾块迭代时沿用上一次尾块的残留值，导致非尾块的数据搬运范围错误。同时actualN的尾块判断只覆盖了orderNMFlag==true的情况，遗漏了orderNMFlag==false时的n轴尾块
- 修复模式：在循环体顶部无条件重置mCoreUse=tiling.singleCoreM和nCoreUse=tiling.singleCoreN；actualN尾块条件扩展为`(orderNMFlag && outIndex == outCnt-1) || (!orderNMFlag && inIndex == inCnt-1)`
- 可审查性：高（循环内变量是否每次迭代都被正确初始化；条件分支是否覆盖所有flag组合）
- 审查规则建议：循环体内使用的变量必须在每次迭代开始时初始化，不得依赖条件分支内的赋值；多路遍历(orderNMFlag)的尾块判断需覆盖所有排列方式

### eb3755e5b853 SparseTensorDenseMatMul修复Tiling内打印占位符误用
- 根因类别：printf格式占位符与参数类型不匹配
- 涉及文件：matmul/sparse_tensor_dense_mat_mul/op_host/op_tiling/sparse_tensor_dense_mat_mul_tiling_arch35.cpp
- 缺陷描述：OP_LOGE日志中int64_t类型参数使用%lld占位符，但在LP64系统(Linux 64位)上int64_t为long而非long long，应使用%ld。约20处日志语句受影响
- 修复模式：全部%lld改为%ld
- 可审查性：高（静态分析/编译器-Wformat即可发现）
- 审查规则建议：日志格式字符串中int64_t使用PRId64宏或%ld(LP64系统)，避免%lld

### 2052949f785d fix env.sh
- 根因类别：环境变量设置脚本残留代码清理
- 涉及文件：scripts/package/common/sh/script_operator.inc
- 缺陷描述：add_setenv和del_setenv函数中的环境变量设置逻辑已转移到上游包管理，但函数体中仍保留旧实现代码。清空函数体只保留return 0
- 修复模式：删除函数体中的旧逻辑，仅保留return 0
- 可审查性：中（需了解上下游职责变更）
- 审查规则建议：上下游职责迁移后及时清理下游残留实现

### 44886a3fea96 fix mse_loss index unique_consecutive dynamic_mx_quant doc bug
- 备注：纯文档修改(多个README.md)，无代码缺陷，跳过

### cfe65e1f9cef 修复PR 518 多删的tiling策略
- 根因类别：策略优先级列表条目遗漏
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_tiling/arch35/batch_matmul_v3_tiling_strategy.h
- 缺陷描述：先前PR 518在重构BatchMatMulV3PrioritiesMap时误删了ASCEND910_95平台的ITER_BATCH_BASICAPI策略项，导致该平台下特定batch场景无法匹配到正确的tiling策略
- 修复模式：在ASCEND910_95的优先级vector中恢复strategy::ITER_BATCH_BASICAPI条目
- 可审查性：高（PR改动策略列表时diff对比前后条目完整性）
- 审查规则建议：策略/路由表的增删改必须逐条对比变更前后的完整列表，防止误删

### 059ff49b6f3e 【bugfix】去掉chamferdistancegrad的pipeall
- 根因类别：流水线同步粒度过粗(PipeBarrier<PIPE_ALL>)
- 涉及文件：loss/chamfer_distance_grad/op_kernel/chamfer_distance_grad.h
- 缺陷描述：ChamferDistanceGrad kernel中约20处使用PipeBarrier<PIPE_ALL>进行全流水线同步，严重阻塞性能。实际只需特定硬件事件间的同步(如V_MTE3, MTE2_MTE3, MTE3_S等)
- 修复模式：引入PipeSync<HardEvent>模板函数，将所有PIPE_ALL替换为精确的硬件事件同步。部分场景用PIPE_MTE2替代
- 可审查性：高（搜索PipeBarrier<PIPE_ALL>用法，逐处分析实际数据依赖）
- 审查规则建议：禁止在性能敏感kernel中使用PipeBarrier<PIPE_ALL>，必须分析实际数据流确定最小同步粒度

### de3fce33252b TransposeBatchMatmul ut bugfix
- 根因类别：stub类型重复定义冲突
- 涉及文件：matmul/transpose_batch_mat_mul/op_kernel/pp_matmul_ein_sum_kernel.h
- 缺陷描述：UT测试宏__CCE_KT_TEST__下，手写的`using __bf16 = bfloat16_t`与随后include的stub_def.h中的同名定义冲突，导致编译报错
- 修复模式：删除手写的using声明，依赖stub_def.h提供定义
- 可审查性：高（类型别名/using声明不应与框架stub头文件重复）
- 审查规则建议：UT stub环境中的类型定义统一由stub头文件提供，禁止在业务代码中重复声明

## 第13轮 (commit 241-260)

### #241 433615f8 修复aicpu ut编译头文件和链接报错
- 根因类别：构建配置缺陷
- 涉及文件：cmake/ut.cmake, tests/ut/op_kernel_aicpu/CMakeLists.txt
- 缺陷描述：CMake缺少头文件搜索路径`${ASCEND_DIR}/pkg_inc/base`导致编译找不到头文件；链接顺序错误，`-ldl`放在`--whole-archive`区域内，应在`--no-whole-archive`之后
- 修复模式：添加缺失include路径；调整链接库顺序
- 可审查性：中
- 审查规则建议：`--whole-archive`和`--no-whole-archive`之间只应包含静态库(.a)，不应出现`-l`系统库

### #242 bf3a91df 修复dynamic_quant_update_scatter文档
- 纯文档修复，跳过

### #243 6b56f111 修复baddbmm接口bug + mmv3/bmmv3 example整改
- 根因类别：条件分支Guard缺失
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_batch_matmul.cpp, common/stub/op_api/level0/batch_matmul_v2tov3.h
- 缺陷描述：三个问题：(1)`enableFp16Bf16InFp32Out`标志在`GetBatchMatmulOpInfo`中未区分isBaddbmm参数，导致普通bmm也错误走fp16转fp32路径；(2)`GetBatchMatmulOp`中fp16/bf16转fp32路径缺少平台检查(需910B/910_93)；(3)K==1场景多余Cast到DT_FLOAT再Mul，不必要的精度损失和性能浪费
- 修复模式：增加`isBaddbmm`参数透传；收紧条件判断；移除K==1多余Cast
- 可审查性：高
- 审查规则建议：当某flag只在特定调用场景下生效时，检查是否有足够Guard条件区分调用上下文

### #244 db6c5243 fix ut case (where算子宏定义)
- 根因类别：宏定义与链式调用模式冲突
- 涉及文件：index/where/tests/ut/op_kernel_aicpu/test_where.cpp
- 缺陷描述：`CREATE_NODEDEF`宏使用`do { ... } while(0)`包裹，但宏体末尾是链式builder调用(.Output())需要后续代码继续追加(.Build())，`do-while(0)`使语句闭合后无法继续链式调用
- 修复模式：移除`do {} while(0)`包裹，改为直接展开语句
- 可审查性：高
- 审查规则建议：宏内有链式builder模式调用时，不应使用`do-while(0)`包裹

### #245 301e3067 fix multi_add_rms_norm_dynamic_quant readme
- 纯文档+license格式修复，跳过

### #246 8a2d45c9 裸指令回退 (GetBlockIdx vs get_subblockid)
- 根因类别：API语义混淆（block级vs sub-block级索引）
- 涉及文件：conv/common/op_kernel/arch35/conv_opt_group_init_impl.h, conv/conv2d_v2/op_kernel/arch35/conv2d_v2_c04_impl.h, conv/conv2d_v2/op_kernel/arch35/conv2d_v2_dma_impl.h, conv/conv2d_v2/op_kernel/arch35/conv2d_v2_weight_ub_trans_impl.h
- 缺陷描述：4个`VecInit`函数中，`self->ctx.vecId`被错误赋值为`GetBlockIdx()`(核级索引)，实际需要`get_subblockid()`(sub-block内ID)。多核/多sub-block场景下vecId错误导致向量计算地址偏移错误
- 修复模式：4处`GetBlockIdx()`改为`get_subblockid()`
- 可审查性：高
- 审查规则建议：arch35 kernel中`vecId`/subblock相关初始化必须使用`get_subblockid()`；`vecId = GetBlockIdx()`为可疑模式

### #247 36b7c87b fix DQUSV2md
- 纯文档修复，跳过

### #248 4cfb4526 fix ut (ApplyAdamWV2 tiling UT)
- 根因类别：UT与实现不同步
- 涉及文件：optim/apply_adam_w_v2/tests/ut/op_host/test_apple_adam_w_v2_arch35_tiling.cpp
- 缺陷描述：tiling逻辑改动（模板数据不参与tilingdata比对）后，UT期望值未同步更新，且有被注释掉的测试用例
- 修复模式：修改`TilingData2Str`只取tiling data尾部关键字段；更新UT期望值；取消注释被禁用的测试用例
- 可审查性：中
- 审查规则建议：修改tiling data结构/序列化逻辑时，必须同步更新UT期望值

### #249 9311a867 修复代码中的warning
- 根因类别：编译告警（类型不匹配+printf格式符+未使用变量）
- 涉及文件：index/目录和vfusion/目录下多个tiling.cpp（10个文件）
- 缺陷描述：uint64_t/int不匹配、printf格式符错误(%d对unsigned等)、未使用变量未删除、未使用参数缺少[[maybe_unused]]
- 修复模式：修改类型声明、printf格式符、删除未使用变量、添加[[maybe_unused]]
- 可审查性：高
- 审查规则建议：开启`-Werror -Wunused-variable -Wformat -Wsign-compare`

### #250 784cb847 修复代码注释中的硬件型号平台表述问题
- 根因类别：注释规范（内部代号泄露）
- 涉及文件：matmul/batch_mat_mul_v3/tests/ut/, matmul/common/op_host/op_api/
- 缺陷描述：代码注释中使用内部芯片代号(1980/1971/1951)而非对外产品型号(910/910B/310P)
- 修复模式：全局搜索替换注释中的代号
- 可审查性：高
- 审查规则建议：预提交钩子禁止代码中出现内部代号

### #251 243a86dc fix examples and geir
- 根因类别：构建配置路径错误
- 涉及文件：build.sh, cmake/func.cmake, scripts/util/dependency_parser.py
- 缺陷描述：build.sh中include/library路径多了一层`/compiler/`子目录；geir编译缺少`-I`和`-lge_compiler`链接选项；cmake未将examples子目录纳入构建；dependency_parser.py中`all_ops`初始列表为空导致example算子未被解析
- 修复模式：路径修正+编译选项补全+构建入口注册
- 可审查性：中
- 审查规则建议：构建脚本路径变量应有存在性检查；CI应包含examples编译验证

### #252 fa496af1 修复conv aclnn op_api ut问题
- 根因类别：CMake依赖声明缺失
- 涉及文件：conv/convolution_backward/op_host/CMakeLists.txt, conv/convolution_forward/op_host/CMakeLists.txt
- 缺陷描述：convolution_backward DEPENDENCIES缺少conv3d_v2和mat_mul_v3；convolution_forward完全没有声明DEPENDENCIES
- 修复模式：补全`DEPENDENCIES conv3d_v2 mat_mul_v3 matmul.common`
- 可审查性：中
- 审查规则建议：新增算子CMakeLists的DEPENDENCIES须完整覆盖实际依赖

### #253 e43ef817 add_rms_norm_cast overflow check
- 根因类别：整数溢出（多维度乘法无溢出检查）
- 涉及文件：norm/add_rms_norm_cast/op_host/add_rms_norm_cast_tiling_arch35.cpp, 对应UT
- 缺陷描述：`GetWorkspaceSize()`中`WORKSPACE_COUNT * usedCoreNum * numN * sizeof(float)`多个uint64_t相乘可能溢出UINT64_MAX，输入shape极大(2^31)时静默溢出算出错误workspaceSize
- 修复模式：乘法前逐步除法计算maxAllowed，超限则报错GRAPH_FAILED；新增overflow UT
- 可审查性：高
- 审查规则建议：多个维度/核数相乘计算size时必须做溢出检查

### #254 f2b9564f fix AdaMaxPool3d pipeAll
- 根因类别：PipeBarrier<PIPE_ALL>粗粒度同步
- 涉及文件：pooling/adaptive_max_pool3d/op_kernel/adaptive_max_pool3d_big_pool.h
- 缺陷描述：bf16类型转换前使用PipeBarrier<PIPE_ALL>()做全流水线同步，过于粗粒度，实际只需S->V同步
- 修复模式：替换为精确事件同步FetchEventID(HardEvent::S_V) + SetFlag + WaitFlag
- 可审查性：高
- 审查规则建议：禁止kernel代码中使用PipeBarrier<PIPE_ALL>()，必须使用精确event-based同步

### #255 4c95f212 修复pr模板自动关联issue
- 纯文档修复（PR模板），跳过

### #256 572a5695 修复linear_index README的错误
- 纯文档修复，跳过

### #257 8f8ded2d swigluQuant fix __restrict__
- 根因类别：编译器优化/别名分析问题
- 涉及文件：quant/swi_glu_quant/op_kernel/swi_glu_quant.cpp
- 缺陷描述：GET_TILING_DATA宏展开后tiling_data是栈上副本，通过&取地址传给op.Init()时，编译器无法做alias分析优化，可能产生错误代码生成
- 修复模式：引入`__restrict__`指针 + 全文`.`改`->`访问
- 可审查性：低
- 审查规则建议：所有GET_TILING_DATA后应统一使用`__restrict__`指针访问模式

### #258 a13fd663 修复foreach问题 (CMake binary_json优先级)
- 根因类别：CMake构建逻辑分支优先级错误
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：对有自定义binary_json的算子，原逻辑先走name-based查找op_type，若返回空则continue跳过，导致有自定义配置的算子永远无法编译。binary_json存在性检查应优先于name查找
- 修复模式：调整条件分支顺序，binary_json优先级提升到name查找之前
- 可审查性：高
- 审查规则建议：多配置来源的优先级应明确；审查"先检查低优先级来源、失败即退出"导致高优先级来源被跳过的逻辑

### #259 7f3c12e8 fix foreach_mul_scalar opapi ut failure
- 根因类别：UT头文件路径错误 + 不支持平台用例
- 涉及文件：foreach/foreach_mul_scalar/tests/ut/op_host/test_aclnn_foreach_mul_scalar_v2.cpp
- 缺陷描述：include路径多了一层`../`导致编译失败；包含Ascend910(910A)测试用例但算子不支持该平台
- 修复模式：修正include相对路径 + 删除不适用平台的测试用例
- 可审查性：高
- 审查规则建议：UT平台型号应与算子支持矩阵一致；相对路径include应有编译验证

### #260 5e21afdb fix TLS_VERIFY
- 根因类别：TLS证书验证失败（构建环境问题）
- 涉及文件：cmake/third_party/makeself-fetch.cmake
- 缺陷描述：FetchContent_Declare下载makeself时TLS证书校验失败导致构建中断
- 修复模式：添加`TLS_VERIFY OFF`关闭证书校验（临时规避方案）
- 可审查性：中
- 审查规则建议：任何`TLS_VERIFY OFF`必须附带说明和后续修复计划

---
## 第14轮 (commit #261-#280)

跳过纯文档/UT/example修复：#262(example拼写), #265(aclnn资料), #266(文档), #268(UT补齐), #270(foreach example+geir去重), #271(foreach opapi ut路径/用例), #273(groupnorm文档), #274(index算子README), #275(Conv3DV2 README), #276(描述错误), #279(kernel ut), #280(注释错误)

### #261 f43375ca fix GroupNormSilu Tiling
- 根因类别：整数溢出 + 错误传播缺失
- 涉及文件：norm/group_norm_silu/op_host/group_norm_silu_tiling.cpp
- 缺陷描述：(1) `shapeN * tilingData.get_numGroups()`在uint32范围内可能溢出，需提升为uint64; (2) `SetProcessSize`和`SetTilingSD`原本返回void，内部除零检查只打日志不返回错误码，调用方无法感知失败; (3) `remainUbSize == 0`未检查导致后续除法异常
- 修复模式：引入`uint64_t totalGroups`避免中间溢出; 将void函数改为返回`ge::graphStatus`，用`OP_CHECK_IF`宏链式传播错误
- 可审查性：高
- 审查规则建议：(1) 多维度乘法必须检查溢出或使用uint64; (2) tiling内部函数必须返回错误码，禁止void+日志的错误处理模式

### #263 e85c3d01 topKtopPSample selLogitsOut sync fix
- 根因类别：默认值语义错误 + DMA同步缺失
- 涉及文件：index/top_k_top_p_sample/op_kernel/top_k_top_p_sample.h
- 缺陷描述：(1) 可选输出logitsTopKPSelect的默认值为0.0，但文档规定应为-inf，导致未选中的logit值被误认为有效; (2) InitGlobalMemory写入GM后缺少MTE3→MTE2同步(SetWaitFlag)，多核场景下可能脏写; (3) SetGlobalBuffer缺少size参数限制，无边界保护
- 修复模式：定义FP32_NEG_INF_BITS常量用于-inf初始化; 添加SetWaitFlag<MTE3_MTE2>同步; SetGlobalBuffer添加coreEle_参数
- 可审查性：高
- 审查规则建议：(1) InitGlobalMemory默认值必须与算子语义文档一致; (2) GM写入后必须在跨核SyncAll前添加pipe同步

### #264 3a682df5 masked_softmax_with_rel_pos_bias算子修正逻辑错误
- 根因类别：索引常量错误 + 整数溢出 + 零检查溢出
- 涉及文件：norm/masked_softmax_with_rel_pos_bias/op_host/masked_softmax_with_rel_pos_bias_tiling.cpp
- 缺陷描述：(1) Y_OUTPUT_INDEX定义为3(与输入索引混淆)，实际输出只有1个应为0，导致CheckOutShape取错tensor; (2) 零值检查`b_ * w_ * n_ * s1_ * s2_ == 0`本身可能在uint32乘法中溢出，应逐个检查; (3) 多处buffer size计算使用uint32_t(bf16AttenAndBiasCastTempSize, xSize, totalSize等)，大shape时溢出; (4) 中间乘法如`w_ * n_ * s1_ * s2AlignedSize * dtypeSize`缺少uint64 cast
- 修复模式：Y_OUTPUT_INDEX改为0; 乘积零检查改为逐个`==0`; 十余处uint32→uint64升级 + static_cast<uint64_t>; printf格式%u→%lu
- 可审查性：高
- 审查规则建议：(1) 输入/输出索引常量必须与算子proto定义一致; (2) 多维度乘积不应用于零值检查(溢出风险); (3) buffer size变量统一用uint64_t

### #267 607c1717 FixApplyTopkToppNN
- 根因类别：UB size截断 + workspace不足 + TopP路径重构
- 涉及文件：index/apply_top_k_top_p_with_sorted/op_host/apply_top_k_top_p_with_sorted_tiling.cpp, op_kernel/*.h, *.cpp
- 缺陷描述：(1) `platformUbSize`为uint64_t但直接`static_cast<uint32_t>`截断后使用，大UB场景数据丢失; (2) TopP场景workspace只分配了固定syncWorkspaceSize，缺少数据buffer空间; (3) TopP路径实现作为TopK类的方法耦合度高且有逻辑缺陷，需独立类; (4) 多核分配逻辑batchSize_<=coreNum_时走特殊分支，统一为通用逻辑
- 修复模式：platformUbSize_存为uint64; TopP场景workspace增加`batchSize_ * vocabSize_ * 4`字节; TopP路径抽离为ApplyTopPWithSorted类; 添加iterateTimes_=ceil(log2(vocabSize_))
- 可审查性：中
- 审查规则建议：(1) 平台参数(UB/L1 size)禁止截断为uint32; (2) workspace分配必须覆盖所有tilingKey路径的实际需求

### #269 11b557fe Fix Conv3DV2 tiling kAL1 upper bound
- 根因类别：硬件指令参数上限公式错误
- 涉及文件：conv/conv3d_v2/op_host/op_tiling/conv3d_api_tiling_algorithm.cpp
- 缺陷描述：load3d指令postk参数限制为16bit(max 65535)。原公式`(POSTK_LIMIT + k0) / ci0HkWk`计算kAL1上界，但kAL1 * ci0HkWk可能>65535(因为加了k0再除法向上偏移)。正确公式应为`POSTK_LIMIT / ci0HkWk`确保不超限
- 修复模式：移除`+ k0`项，改为直接整除
- 可审查性：高
- 审查规则建议：硬件指令参数上限计算必须用下界公式(floor除法)，不得添加额外余量导致超限

### #272 7a9a7a5a 修复ophost ut以及解决run example链接顺序问题
- 根因类别：链接顺序错误
- 涉及文件：build.sh, tests/ut/op_host/CMakeLists.txt
- 缺陷描述：g++链接时`-lopapi -lcust_opapi`顺序错误，ELF链接器要求提供符号的库在引用符号的库之后。cust_opapi依赖opapi中的符号，所以cust_opapi必须排在opapi之前。同理`-lopapi -lopapi_nn`应为`-lopapi_nn -lopapi`
- 修复模式：调换链接库顺序: `-lcust_opapi -lopapi`, `-lopapi_nn -lopapi`
- 可审查性：高
- 审查规则建议：自定义库(-lcust_*)必须排在基础库(-lopapi)之前；链接顺序变更需验证所有example编译通过

### #277 9d1d765e fix_ut
- 根因类别：类名拼写错误 + UT匹配逻辑不完整 + _exit绕过清理
- 涉及文件：scripts/util/parse_changed_files.py, tests/ut/op_host/test_op_host_main.cpp
- 缺陷描述：(1) `UtMathcer`拼写错误应为`UtMatcher`; (2) OpApiUt只匹配`op_api`路径，遗漏了`op_host`下的`test_aclnn_*`文件(这些文件也是opapi ut); (3) UT main函数用`_exit(RUN_ALL_TESTS())`，_exit不执行atexit回调和全局析构，可能导致资源泄漏和覆盖率数据丢失
- 修复模式：修正拼写; OpApiUt增加`op_host + test_aclnn_`匹配; `_exit`→`return`
- 可审查性：高
- 审查规则建议：(1) CI脚本中的路径匹配逻辑变更需覆盖所有目录结构; (2) 测试入口禁止使用_exit，应用return确保正常清理

### #278 c7819ec4 fix for issue 89 and 86
- 根因类别：TPipe作用域错误 + 编译器alias分析缺失
- 涉及文件：norm/layer_norm_grad_v3/op_kernel/*.h, *.cpp; norm/add_rms_norm/op_kernel/add_rms_norm.h
- 缺陷描述：(1) layer_norm_grad_v3的4个实现类(SingleRead/Workspace/Transpose/Common)各自创建TPipe成员但由宏在函数体内实例化，TPipe管理硬件pipe资源，在类内部创建可能导致生命周期与硬件资源不匹配; (2) add_rms_norm的tiling指针缺少`__restrict__`，编译器无法做alias优化，可能生成低效代码
- 修复模式：在entry函数layer_norm_grad_v3中创建`TPipe pipe`并通过Init参数传递给所有实现类; add_rms_norm tiling指针添加`__restrict__`
- 可审查性：中
- 审查规则建议：(1) TPipe必须在kernel入口函数创建并传递，不得在类内部隐式创建; (2) GET_TILING_DATA指针应标记__restrict__

## 第15轮 (commit 281-300)

### #281 fd3a3d11 fix build warning
- 根因类别：编译告警 - printf格式说明符与参数类型不匹配
- 涉及文件：quant/dynamic_quant_update_scatter_v2/op_host/dynamic_quant_update_scatter_v2_tiling.cpp
- 缺陷描述：OP_LOGE中%llu格式化非unsigned long long类型的ubSize，%u格式化有符号int类型的vectorCoreNum
- 修复模式：修正格式说明符(%llu->%lu, %u->%d)
- 可审查性：高
- 审查规则建议：开启-Wformat编译选项自动检测格式化字符串与参数类型不匹配

### #282 23736aa6 修复kernel文件有后缀的情况下编译无法打包.o和json文件的问题
- 根因类别：构建脚本逻辑缺陷 - 文件名后缀处理位置不当
- 涉及文件：cmake/gen_ops_info.cmake, scripts/kernel/binary_script/build_binary_single_op_gen_task.sh
- 缺陷描述：kernel文件名带后缀(_apt/_910b)时，后缀剥离逻辑分散在子函数内，但main函数中用于拼接.o和json打包路径的变量仍保留后缀，路径与编译产物不一致；cmake中install(FILES)硬编码了${OP_NAME}.json无法匹配带后缀文件
- 修复模式：后缀剥离逻辑上移到main统一处理；cmake改用file(GLOB)模式匹配
- 可审查性：中
- 审查规则建议：构建脚本中文件名转换逻辑应集中在一处；install(FILES)应考虑文件名变体

### #283 21a1f29d fix InplaceScatterAdd readme
- 跳过：纯文档修改

### #284 54768d07 软链接修复
- 根因类别：硬编码平台路径
- 涉及文件：scripts/package/ops_nn/scripts/ops_nn_custom_install.sh, ops_nn_custom_remove_softlink.sh
- 缺陷描述：软链接脚本硬编码ascend910b目录，只为910b平台创建/删除软链接，其他平台(310p等)不被处理
- 修复模式：硬编码平台名替换为ascend*通配符遍历
- 可审查性：高
- 审查规则建议：部署脚本不应硬编码特定硬件平台名称，应使用通配符或配置文件枚举

### #285 1c6efd0d fix codecheck bug
- 根因类别：多类缺陷 - 资源泄漏 + 类型错误 + 魔法数
- 涉及文件：activation/elu/op_host/op_api/aclnn_elu.cpp, activation/erfinv/examples/, activation/fatrelu_mul/examples/, activation/gelu_mul/op_host/, activation/hard_sigmoid/op_host/op_api/, activation/selu_grad/examples/
- 缺陷描述：(1) aclnn_elu.cpp特殊路径分支创建zeroScalar和negAlphaScalar未释放(资源泄漏)；(2) 多个example中分配内存后未释放；(3) hardsigmoid中魔法数0.16666666f改为1.0f/6.0f；(4) selu_grad example梯度数据类型应为float而非int
- 修复模式：添加缺失的资源释放；删除重复检查；命名常量替代魔法数；修正数据类型
- 可审查性：高
- 审查规则建议：每个aclCreate*/aclrtMalloc必须有对应的aclDestroy*/aclrtFree；禁止魔法数

### #286 c489b93d fix rnnv2ut
- 根因类别：构建配置缺陷 - 符号重复定义
- 涉及文件：cmake/ut.cmake, rnn/dynamic_rnnv2/op_host/CMakeLists.txt, rnn/dynamic_rnnv2/tests/ut/op_kernel/CMakeLists.txt
- 缺陷描述：全量编译kernel UT时，dynamic_rnnv2的tiling UT通过特殊分支硬编码引入dynamic_lstm_tiling.cpp，该文件已通过OPHOST_NAME的tiling_obj目标包含，导致符号重复定义链接失败
- 修复模式：删除特殊case硬编码，通过DEPENDENCIES声明统一处理模块间依赖
- 可审查性：中
- 审查规则建议：CMake中不应为特定算子名添加if/else特殊分支，模块间依赖通过构建系统依赖声明机制表达

### #287 a4905db5 [Bugfix] 修复addmm example中的问题，修复matmulv3 infershape向上对齐实现方式不同的问题
- 根因类别：内存泄漏 + 对齐计算公式错误
- 涉及文件：matmul/mat_mul_v3/examples/test_aclnn_addmm_aclnninplace_addmm.cpp, matmul/mat_mul_v3/op_host/mat_mul_v3_infershape.cpp
- 缺陷描述：(1) example中aclnnInplaceAddmm复用了aclnnAddmm的workspaceSize/workspaceAddr变量，第二次aclrtMalloc覆盖第一次地址导致内存泄漏；(2) hidden_size向上对齐公式(*hidden_size + BLOCK_SIZE) / BLOCK_SIZE * BLOCK_SIZE缺少-1，恰好对齐时多对齐一个BLOCK_SIZE
- 修复模式：引入独立变量避免复用覆盖；修正对齐公式为(x + align - 1) / align * align
- 可审查性：高
- 审查规则建议：向上对齐公式必须使用(x + align - 1) / align * align标准模式；不应复用同名变量指向不同的动态分配内存

### #288 0f483ee6 修改quant目录下存在空指针、格式化漏洞等问题
- 根因类别：printf格式说明符错误 + 业务逻辑校验错误
- 涉及文件：quant/ascend_anti_quant_v2/examples/, quant/ascend_quant_v2/op_host/op_api/aclnn_ascend_quant_v3.cpp
- 缺陷描述：(1) example中resultData为int16_t但LOG_PRINT用%f格式化(应为%hd)，导致未定义行为；(2) aclnn_ascend_quant_v3.cpp中对float8类型的roundMode校验错误允许了不支持的"hybrid"模式
- 修复模式：修正格式说明符；收紧输入校验条件
- 可审查性：高
- 审查规则建议：printf格式说明符必须与参数类型严格匹配；参数校验允许值列表变更需与算子规格文档同步

### #289 422098e3 修正masked_softmax_with_rel_pos_bias算子tiling、infershape对应ut用例
- 跳过：纯UT修改

### #290 ed943785 fix ut error
- 根因类别：kernel代码数组越界 + tensor操作偏移冲突
- 涉及文件：loss/ctc_loss_v3/op_kernel/ctc_loss_v3.h
- 缺陷描述：(1) Duplicate初始化logAlpha时使用alphaLengthAlign(完整长度)，应只初始化尾部alpahTailSizeAlign，错误长度导致越界写入；(2) Log的结果写入expLogTensor偏移0处，但被后续Add操作覆盖，GetValue(0)读到的是Add结果而非Log结果。修复后Log写入FLOAT_NUM_PER_BLOCK偏移处
- 修复模式：修正Duplicate长度参数；修正Log输出偏移避免与前序操作重叠
- 可审查性：低
- 审查规则建议：Tensor操作的读写偏移和长度参数需逐行审查确保不重叠；涉及INFINITY特殊值的分支需专门边界测试

### #291 511503b1 修复conv目录下存在逻辑、重复代码问题
- 根因类别：错误处理不当 + 数学函数边界条件 + 整数溢出
- 涉及文件：conv/common/op_host/conv_backprop_infershape.cpp, conv/common/op_host/op_tiling/math_util.cpp, conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch32/conv3d_backprop_filter_v2_base_tiling.cpp
- 缺陷描述：(1) OP_LOGE_IF在nullptr时仅打日志不返回错误，改为OP_CHECK_IF正确返回GRAPH_FAILED；(2) GetGcd函数param2==0时返回0，GCD(n,0)应返回n；(3) CeilDiv结果int64_t直接static_cast<int32_t>缺少溢出检查
- 修复模式：错误处理宏替换(LOGE_IF->CHECK_IF)；GCD边界条件修正；窄化转换前置溢出校验
- 可审查性：高
- 审查规则建议：OP_LOGE_IF不应用于需中断执行的场景(应用OP_CHECK_IF)；数学工具函数必须覆盖零值边界；窄化类型转换前必须有溢出校验

### #292 7bcebf20 quantbatchmatmul 修复编译警告
- 根因类别：编译警告 + C++链式比较逻辑陷阱
- 涉及文件：matmul/quant_batch_matmul_v3/ (3文件), matmul/quant_batch_matmul_v4/ (5文件)
- 缺陷描述：(1) %ld/%zu格式说明符与int64_t/uint64_t不匹配；(2) 函数参数未使用；(3) 产品逻辑bug: a == b == 1在C++中被解析为((a==b)==1)即bool==1，不等价于a==1&&b==1
- 修复模式：格式说明符修正；删除/void标记未使用参数；修复链式比较
- 可审查性：高
- 审查规则建议：禁止使用a==b==c链式比较写法，在C++中几乎永远是bug

### #293 4384b110 【bugfix】kernel ut 整改
- 跳过：纯UT修改

### #294 1c2de786 修复infershape中先使用后校验nullptr的问题，修复n_x2_dim位置
- 根因类别：空指针解引用风险 + 复制粘贴错误
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：(1) InferShapeForBatchMatMul中先解引用shape_x1/shape_x2(调用CheckIsUnknownDimNum)再检查nullptr，时序颠倒导致空指针解引用崩溃；(2) n_x2_dim计算公式与k_x2_dim完全相同(trans_x2 ? dim-1 : dim-2)，复制粘贴错误，n_x2_dim应为trans_x2 ? dim-2 : dim-1
- 修复模式：空指针校验前置(use-before-check -> check-before-use)；修正维度索引互补关系
- 可审查性：高
- 审查规则建议：指针nullptr校验必须在首次解引用之前；矩阵k/n维度索引在转置/非转置情况下必须互补(一个dim-1另一个dim-2)

### #295 2cc78fa3 修复quant类算子kernel ut不执行的问题
- 跳过：纯UT修改

### #296 c391912 修复norm类算子opkernel UT error
- 根因类别：无效校验代码(unsigned类型与0比较)
- 涉及文件：norm/group_norm_grad/op_host/group_norm_grad_tiling.cpp
- 缺陷描述：uint32_t sysWorkspaceSize做<0检查、uint64_t ubSizePlatForm做<=0检查，unsigned类型永远不可能<0，条件恒假为死代码
- 修复模式：删除无符号类型与负数/零的无意义比较
- 可审查性：高
- 审查规则建议：对unsigned类型做<0检查应作为编译错误(开启-Werror=type-limits)

### #297 9eb1b2c7 rms_norm_quant opkernel UT error fix
- 跳过：纯UT修改

### #298 b45ac62b fix compliation warning
- 根因类别：编译警告(未使用的函数参数)
- 涉及文件：foreach/foreach_non_finite_check_and_unscale/op_host/, foreach/foreach_utils/op_host/
- 缺陷描述：TilingPrepare4Foreach*函数的context参数未使用触发-Wunused-parameter
- 修复模式：[[maybe_unused]]属性标记
- 可审查性：高
- 审查规则建议：回调/接口函数中不使用的参数应用[[maybe_unused]]标记或省略参数名

### #299 8595d59b 修复SwiGluQuant大shape精度问题
- 根因类别：整数类型溢出(uint16截断)
- 涉及文件：quant/swi_glu_quant/op_kernel/swi_glu_quant.h, swi_glu_quant_static.h
- 缺陷描述：ProcessCoreMultiUbMultiAlign的offsetRow参数类型为uint16_t(最大65535)，大shape时超出范围截断溢出，数据拷贝地址计算错误导致精度问题
- 修复模式：参数类型从uint16_t扩展为uint32_t
- 可审查性：高
- 审查规则建议：表示shape/偏移量/元素数量的变量不应使用uint16_t等窄类型，至少uint32_t；开启-Wconversion检测窄化转换

### #300 49589d34 ctcloss_v3grad,kernelut_fix
- 跳过：纯UT修改

## 第16轮 (commit 301-320)

### #301 7bf556ae mse_loss_v2,advance_step等kernel_ut_fix
- 跳过：纯UT修复

### #302 4882c709 修复pooling目录下存在资源泄露
- 根因类别：资源泄露/遗漏释放
- 涉及文件：pooling/adaptive_avg_pool3d/examples/test_aclnn_adaptive_avg_pool2d.cpp, test_aclnn_adaptive_avg_pool3d.cpp
- 缺陷描述：示例代码创建的outputSize(aclIntArray)对象未调用aclDestroyIntArray释放，释放阶段只释放了input和out两个tensor，遗漏了outputSize
- 修复模式：补充遗漏的aclDestroyIntArray(outputSize)调用
- 可审查性：高
- 审查规则建议：检查所有aclCreate*/aclDestroy*配对完整性，静态分析扫描同作用域内分配与释放的对应关系

### #303 f8f5e723 fix opapiUT for pooling
- 根因类别：构建配置遗漏
- 涉及文件：cmake/ut.cmake, 多个pooling UT CMakeLists.txt
- 缺陷描述：cmake/ut.cmake中supportedCategory列表缺少"pooling"类别，导致pooling算子UT编译流程无法执行；多个pooling算子UT CMakeLists.txt缺少标准编译配置
- 修复模式：在supportedCategory中补充pooling，补全各UT CMakeLists.txt配置
- 可审查性：高
- 审查规则建议：新增算子类别时应检查cmake构建系统的类别注册列表是否已更新

### #304 cd9c0049 fix group_norm_silu ut
- 跳过：纯UT修复

### #305 0e00f88c fix ops-nn_compile_ophost_warnings
- 根因类别：编译告警含真bug（无限循环、格式符、无效检查）
- 涉及文件：conv/conv3d_backprop_input_v2 tiling, index/apply_top_k_top_p_with_sorted tiling, pooling/adaptive_avg_pool3d_grad tiling, pooling/avg_pool3_d tiling+grad, pooling/max_pool3d_with_argmax_v2 tiling
- 缺陷描述：多处编译告警中隐含实际bug：(1) avg_pool3_d_grad中`for(uint64_t i=singleCoreWo; i>=0UL; i--)`无符号整数>=0恒真导致无限循环；(2) apply_top_k_top_p_with_sorted中printf格式说明符%ld用于size_t应为%zu；(3) adaptive_avg_pool3d_grad中无符号返回值做<0检查永远为false属无效代码；(4) conv3d_backprop_input_v2中有符号/无符号比较
- 修复模式：循环条件改为i>0UL、格式说明符修正、删除无效检查、static_cast类型转换
- 可审查性：高
- 审查规则建议：无符号整数循环中>=0条件是经典无限循环bug，应通过-Wtautological-compare或静态分析检测；启用-Werror将告警升级为错误

### #306 1074ec43 fix-DynamicQuantUpdateScatter-UT
- 根因类别：API接口变更适配
- 涉及文件：tests/ut/op_host/test_op_host_main.cpp, tests/ut/op_kernel/test_op_kernel_main.cpp
- 缺陷描述：OppSoDesc构造函数接口变更，原接受单个AscendString参数，现需要花括号初始化列表{AscendString(...)}。影响所有UT公共入口文件导致全部UT编译失败
- 修复模式：适配新的初始化列表构造函数
- 可审查性：高
- 审查规则建议：上游依赖库API签名变更时应有自动化API兼容性检测

### #307 3bf4267d 修复batch_mat_mul_v3等约束描述
- 跳过：纯文档修改

### #308 925b1ba8 nn-dev仓同步修复issue18
- 根因类别：工具脚本健壮性不足
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op_gen_task.sh, gen_output_json.py
- 缺陷描述：shell脚本无条件调用dos2unix但未检查工具是否安装、文件是否存在；Python脚本kernel编译产物json生成失败时错误信息不够详细
- 修复模式：添加前置条件检查(command -v dos2unix, -f file)和防御性编程
- 可审查性：高
- 审查规则建议：shell脚本调用外部命令前应先检查command -v确认工具存在

### #309 039383d3 修复matmul下gemm_v2等UT用例缺失报错
- 根因类别：构建配置遗漏/迁仓同步不完整
- 涉及文件：matmul/gemm_v2/op_host/CMakeLists.txt(生产代码), 多个matmul UT CMakeLists.txt
- 缺陷描述：(1) gemm_v2的op_host CMakeLists.txt缺少DEPENDENCIES mat_mul_v3声明导致编译失败；(2) quant_batch_matmul_v3 UT使用错误的模块名(OP_API_MODULE_NAME而非OP_TILING_MODULE_NAME)；(3) quant_matmul_reduce_sum UT CMakeLists为空，迁仓时PR未同步
- 修复模式：补充CMake依赖声明和编译配置
- 可审查性：高
- 审查规则建议：迁仓操作后应有自动化构建验证确保所有算子UT能编译运行

### #310 4023e4c2 修改rnn doc修复
- 跳过：纯文档修改

### #311 87a66d70 fix issue 11
- 跳过：纯文档修复（将文档中错误引用的算子名修正）

### #312 12fbd5c8 [swi_glu/swi_glu_grad] kernel ut修复；host ut修复；增加example
- 根因类别：构建配置缺陷 + 无符号整数比较bug
- 涉及文件：activation/swi_glu/op_host/CMakeLists.txt, activation/swi_glu_grad/op_host/CMakeLists.txt, activation/swi_glu/op_host/swi_glu_tiling.cpp(生产代码)
- 缺陷描述：(1) CMakeLists缺少DEPENDENCIES声明，swi_glu与swi_glu_grad互相依赖但未声明；(2) swi_glu_tiling.cpp中totalCore是uint32_t类型，做<0比较永远为false
- 修复模式：补充CMake依赖 + cast为int再比较
- 可审查性：中
- 审查规则建议：-Wtype-limits检测无符号整数与0的<比较

### #313 50854088 ophost warning fix
- 根因类别：编译告警含多类真bug（链式比较、运算符优先级、类型转换）
- 涉及文件：matmul/下17个生产代码文件
- 缺陷描述：(1) `a==b==1`链式比较误用，C++中被解析为(a==b)==1而非a==1&&b==1；(2) `FORMAT_NZ && dim2==1 || dim1==1`运算符优先级错误，&&高于||导致逻辑意图被改变，应加括号；(3) printf格式说明符%zu用于int64_t应为%ld；(4) CeilDiv调用中int64_t强转uint64_t负值变大正数；(5) 多处未使用变量/参数
- 修复模式：链式比较重写、加括号明确优先级、格式说明符修正、类型转换方向修正、删除死代码
- 可审查性：高
- 审查规则建议：-Wlogical-op-parentheses检测&&/||混合；静态分析检测==链式比较；-Wformat -Wsign-compare -Wunused全量启用

### #314 c9b03748 msda ut fix
- 根因类别：构建配置遗漏
- 涉及文件：cmake/ut.cmake
- 缺陷描述：supportedCategory列表缺少"vfusion"类别，导致vfusion目录下算子UT无法被构建系统识别
- 修复模式：补充supportedCategory条目
- 可审查性：高
- 审查规则建议：CI检查扫描顶层目录与supportedCategory列表的差异

### #315 b17c8b62 dequant_swiglu_quant ut error修复
- 跳过：纯UT修复

### #316 c80e6362 fix pooling ophost opkernel UT
- 跳过：纯UT修复

### #317 8ee323a6 fix SMS/SMSG ut
- 跳过：纯UT修复

### #318 ad8129da kernal ut error 修复
- 根因类别：代码风格缺陷（冗余分号）
- 涉及文件：optim/advance_step/op_kernel/advance_step_spec.h(生产代码)
- 缺陷描述：10处PipeBarrier<PIPE_V>()后有多余分号`;;`，虽不影响运行时行为但表明copy-paste错误
- 修复模式：删除冗余分号
- 可审查性：高
- 审查规则建议：-Wextra-semi检查多余分号

### #319 338f8b9a fix eigen compilation using cuda keyword
- 根因类别：第三方库构建配置缺陷
- 涉及文件：cmake/third_party/eigen.cmake
- 缺陷描述：Eigen作为header-only库通过ExternalProject_Add引入，但未设置CONFIGURE_COMMAND ""和BUILD_COMMAND ""，默认会尝试编译含CUDA关键字的源码导致非CUDA环境编译失败
- 修复模式：显式禁用ExternalProject的configure和build步骤
- 可审查性：中
- 审查规则建议：header-only第三方库通过ExternalProject引入时应显式禁用configure和build

### #320 301b0cdd host ut error 修复
- 跳过：纯UT修复

## 第17轮 (commit 321-340)

### #321 c1f867025f 修复日志打印有歧义问题
- 跳过：纯日志文案优化，将error message从模糊描述改为更详细的提示

### #322 402244ae71 fix rnn ut
- 跳过：纯UT修复（cmake构建系统新增rnn类别支持+UT测试文件更新）

### #323 9a2ee5e935 fix lisence
- 跳过：纯license修复（第三方软件声明拆分+文件头版权声明添加）

### #324 6593ca1b89 fix run example
- 根因类别：CI脚本逻辑缺陷（空数组误判为失败）
- 涉及文件：build.sh
- 缺陷描述：build_example函数中，当某个算子在指定mode下没有example用例时（files数组为空），脚本走到`failed_example != "" || success_example == ""`判断分支，误报为"Run example failed"并exit 1，导致整个构建流程异常中断
- 修复模式：新增`${#files[@]} -eq 0`前置条件分支，当没有example时仅输出提示信息而非报错退出
- 可审查性：高
- 审查规则建议：shell脚本中对空数组/空列表的边界条件处理，避免将"无数据"误判为"失败"

### #325 578fecacdb add fix AdaptiveMaxPool3d\Grad example
- 跳过：纯新增example文件（4个新增文件）

### #326 b49c1c1eb6 fix quant_matmul_reduce_sum_weight_nz example bug
- 跳过：纯新增example文件（1个新增文件）

### #327 9e27b020f1 fix ut fail in arm env
- 根因类别：构建配置缺陷（RPATH导致链接到不完整桩函数so）
- 涉及文件：tests/ut/op_host/CMakeLists.txt
- 缺陷描述：ophost UT可执行文件构建时默认嵌入了BUILD_RPATH，在ARM环境下运行时优先链接到构建目录中不完整的桩函数共享库导致UT失败
- 修复模式：设置CMake属性SKIP_BUILD_RPATH TRUE，使运行时从环境变量查找正确的共享库
- 可审查性：中
- 审查规则建议：CMake构建配置中检查RPATH设置对跨平台（x86 vs ARM）部署的影响

### #328 71da36591b fix conv ut
- 跳过：纯UT修复（头文件引用路径从简单文件名改为相对路径）

### #329 760c32539e FixInstructions
- 根因类别：废弃API未迁移
- 涉及文件：norm/masked_softmax_with_rel_pos_bias/op_kernel/masked_softmax_with_rel_pos_bias_BW.h
- 缺陷描述：3处使用了小写函数风格的pipe_barrier(PIPE_V)调用，这是AscendC已废弃的旧版同步原语写法，在新版本编译环境下可能导致编译失败或行为不正确
- 修复模式：替换为模板函数风格PipeBarrier<PIPE_V>()
- 可审查性：高
- 审查规则建议：全局搜索pipe_barrier(调用，批量替换为PipeBarrier<>模板写法

### #330 0e183e0410 修改avgPoolV2Grad proto原型注释，解决geir通路失败
- 根因类别：proto注释规范错误导致功能通路失败
- 涉及文件：pooling/avg_pool_v2_grad/op_graph/avg_pool_v2_grad_proto.h
- 缺陷描述：ksize和strides属性注释描述不准确（"length is 1, 2 or 4"实际仅支持长度4），proto注释中的@li标注被工具链解析为算子约束，错误的维度描述导致geir通路校验失败
- 修复模式：修正注释从"length is 1, 2 or 4"为"length is 4"
- 可审查性：高
- 审查规则建议：proto头文件中的@li注释会被工具链解析为算子约束规范，修改时需验证geir通路

### #331 adee08cd65 二进制发布重复场景整改
- 跳过：JSON配置文件清理（删除重复的binary.json配置项），非代码逻辑缺陷

### #332 124cfa7b2e 修改string类型直接赋值给char*导致的告警
- 根因类别：类型安全缺陷（const correctness违反）
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：将字符串字面量（const char[]）赋值给constexpr char*（非const指针），在C++11及以后标准中不合法，可能导致编译告警且通过该指针写入会产生UB
- 修复模式：将`constexpr char* FUSED_OP_TYPE_SWIGLU = "swiglu"`改为`constexpr char FUSED_OP_TYPE_SWIGLU[] = "swiglu"`
- 可审查性：高
- 审查规则建议：检测constexpr char*或char*直接用字符串字面量初始化的模式

### #333 d2bf0d115c 拦截bias变化导致进入matmul v3kernel bin找不到的问题
- 根因类别：逻辑错误（使用了错误的数据源进行比较）
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp
- 缺陷描述：CheckMMV3NzNzNdSupport中检查混精度时，bias类型比较使用了ori_info（原始输入类型），但bias的dtype可能已被修改（如fp16提升为fp32），应使用support_info（实际参与kernel计算的类型）
- 修复模式：将比较对象从mmOpInfo.ori_info改为mmOpInfo.support_info
- 可审查性：中
- 审查规则建议：当存在ori_info和support_info两套数据源时，审查条件判断中引用的字段是否来自正确的数据源

### #334 33b30e51b9 aclnnAddmmGetWorkspaceSize接口bias不支nz格式未对异常format拦截
- 根因类别：输入校验缺失（format未检查）
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_addmm.cpp
- 缺陷描述：CheckMatmul函数只检查了mat1和mat2的维度和format，但未检查self（bias）的format。当self的storage format为FRACTAL_NZ时后续kernel不支持，导致执行报错
- 修复模式：在CheckMatmul中增加self参数，添加对self->GetStorageFormat() == FORMAT_FRACTAL_NZ的拦截判断
- 可审查性：高
- 审查规则建议：aclnn接口的Check函数应对所有输入tensor的format进行合法性校验，不能只校验部分tensor

### #335 4fed4582df matmul相关算子编译告警消除，避免除0风险
- 根因类别：除零风险 + 未使用参数
- 涉及文件：matmul/transpose_batch_mat_mul/op_host/op_tiling/transpose_batch_mat_mul_base_tiling.cpp, matmul/quant_matmul/op_host/op_api/aclnn_quant_matmul.cpp, matmul/matmul_compress/op_host/op_api/aclnn_matmul_compress.cpp
- 缺陷描述：(1) ResetBasicBlock函数未校验tempBaseM和tempBaseN为0的情况，后续有除法运算；(2) BaseLoadBalance中使用普通除法/而非安全的FloorDiv；(3) ProcessEmptyTensor函数有未使用参数x2
- 修复模式：添加零值前置检查+替换为ops::FloorDiv安全除法+删除未使用参数
- 可审查性：高
- 审查规则建议：检测整数除法运算前是否有零值检查；有安全除法工具函数时检测直接使用/的场景

### #336 c4d5efbd1c gather_v2丢失分支补充
- 根因类别：遗漏分支（tiling key分支不完整）
- 涉及文件：index/gather_v2/op_kernel/gather_v2_apt.cpp
- 缺陷描述：TILING_KEY_VAR分支处理中缺少SIMD_LAST_GATHER_B16_TILING_KEY（int16_t类型），已有B8和B32但B16被遗漏，导致16-bit数据类型用例精度输出为0
- 修复模式：在B8和B32分支之间补充B16分支
- 可审查性：高
- 审查规则建议：使用TILING_KEY进行多分支模板特化时，检查B8/B16/B32/B64等类型分支是否完整覆盖

### #337 00e890438a 解决ConvBackward性能重复上报问题
- 根因类别：DFX宏放置位置不当导致重复采集
- 涉及文件：conv/convolution_backward/op_api/convolutionbackward.cpp, conv/convolution_forward/op_host/op_api/convolution.cpp
- 缺陷描述：L0_DFX性能采集宏放在函数开头，但后续存在"小算子拼接"分支路径内部也有独立的性能采集，导致走拼接case的性能数据被重复采集
- 修复模式：将L0_DFX宏从函数入口移到参数适配/转换逻辑之后、实际kernel调用之前的位置
- 可审查性：中
- 审查规则建议：DFX性能采集宏应放在算子实际执行路径入口处，确保不会对分支路径重复采集

### #338 240a589531 aclnnAddbmm异常场景bias未拦截
- 根因类别：输入校验缺失（broadcast shape校验不完整）
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp
- 缺陷描述：CheckBroadCast函数计算出broadcastShape和bmmLastTwoShape后，缺少两者是否相等的校验。当bias broadcast后的shape与batch matmul输出最后两维shape不一致时，算子未拦截给出错误结果
- 修复模式：增加broadcastShape != bmmLastTwoShape的判断，不匹配时返回false
- 可审查性：高
- 审查规则建议：包含broadcast逻辑的Check函数应验证广播推导后的结果shape与预期shape的一致性

### #339 07a8e44ed4 解除conv2dv2和common目录的互相依赖问题
- 跳过：代码架构重构（解耦循环依赖），非运行时缺陷修复

### #340 ad9d2ab2da modify GroupedDynamicBlockQuant InferShape动态shape失败问题
- 根因类别：动态shape推导不完备 + 属性类型不匹配
- 涉及文件：quant/grouped_dynamic_block_quant/op_host/grouped_dynamic_block_quant_infershape.cpp
- 缺陷描述：(1) InferShape只考虑了dim1未知情况，groupNum也可能未知（动态shape下GroupList长度不确定），groupNum为-1时计算结果错误；(2) 属性指针类型用int32_t获取但实际是int64_t；(3) 缺少rowBlockSize/colBlockSize空指针和非法值校验
- 修复模式：重构为SetShapeDimTwo/SetShapeDimThree独立判断每个维度是否UNKNOWN；属性类型从int32_t*改为int64_t*；添加空指针和<=0校验
- 可审查性：高
- 审查规则建议：InferShape函数中涉及动态shape时检查所有参与维度计算的变量是否都考虑了UNKNOWN情况；GetAttrPointer模板类型必须与注册时属性类型一致

## 第18轮 (commit 341-380, 最终轮)

### #341 cdb5ba5f Conv3DTranspose算子db场景添加同步解决用例多次批跑偶现AICError的问题
- 根因类别：并发同步缺失（double buffer无ping-pong机制）
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/conv3d_bp_func.h, conv3d_bp_impl_base.h, conv_bp_sub_func.h
- 缺陷描述：Conv3DTranspose在db场景下L0C缓冲区使用单队列outQueL0C_管理，无ping-pong双缓冲同步，多次批跑时AIC/AIV核间读写冲突导致偶发AICError
- 修复模式：单队列拆分为l0cPing_/l0cPong_双队列，新增l0cPingPongFlag_标志位交替切换，AllocTensor/EnQue/FreeTensor/DeQue全部增加ping-pong分支
- 可审查性：低
- 审查规则建议：使用double buffer(Pbuffer>1)时检查是否有对应ping-pong同步机制；TQue深度为1但Pbuffer>1可能存在同步缺陷

### #342 4dbc8196 解决fusedmatmul mnk维度超过边界的问题
- 根因类别：整数类型溢出(int32截断shape维度)
- 涉及文件：matmul/fused_mat_mul/op_host/fused_mat_mul_infershape.cpp
- 缺陷描述：InferShapeForFusedMatMul中a_m/b_n/a_k/b_k声明为int(32位)，大矩阵维度超INT32_MAX时溢出
- 修复模式：int改为int64_t
- 可审查性：高
- 审查规则建议：shape/维度变量禁止使用int/int32_t，统一用int64_t；扫描GetDim()返回值赋给int类型的模式

### #343 9f307dfd 误开放的aclnn接口整改
- 根因类别：构建配置错误（接口暴露控制）
- 涉及文件：norm/dua_quantize_add_layer_norm/op_host/CMakeLists.txt, norm/inplace_add_layer_norm/op_host/CMakeLists.txt, norm/quantize_add_layer_norm/op_host/CMakeLists.txt
- 缺陷描述：三个不应有aclnn公开接口的算子ACLNNTYPE误设为aclnn，暴露了不应存在的头文件
- 修复模式：ACLNNTYPE aclnn改为aclnn_exclude
- 可审查性：高
- 审查规则建议：无对应docs/目录的算子ACLNNTYPE不应为aclnn；建立算子接口清单与CMake配置一致性检查

### #344 0bd3fe0b aclnnAddmmWeightNz/aclnnMatmulWeightNz异常拦截
- 根因类别：输入校验缺失（多项：dtype过宽/nullptr/维度/cubeMathType）
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_addmm.cpp, aclnn_matmul.cpp
- 缺陷描述：(1)WeightNz接口复用通用dtype校验允许了不支持的int8/fp32 (2)必选输入nullptr未拦截致core dump (3)维度校验缺失 (4)cubeMathType异常值未拦截
- 修复模式：新增CheckWeightNzDtypeValid独立校验(仅fp16/bf16)；维度!=2检查；空指针检查提前；新增socRule->CheckInput
- 可审查性：高
- 审查规则建议：不同变体接口应使用独立dtype校验函数而非复用通用函数；aclnn函数第一步应做nullptr检查

### #345 b7e91a6e Ascend950上ctcloss算子对标竞品不做尾轴向上8对齐
- 跳过-纯文档

### #346 49de9c80 冗余.o文件修改
- 跳过-纯资源清理（删除冗余binary json配置文件）

### #347 d8b73e6e 算子文档与算子具体代码不一致
- 跳过-纯文档

### #348 f43f60ec 清理冗余的二进制文件和修改cmake中错误的aclnntype
- 根因类别：构建配置错误（接口暴露控制）+ 冗余资源
- 涉及文件：norm/batch_norm_elemt/op_host/CMakeLists.txt等5个norm算子CMakeLists.txt + group_norm_grad binary json
- 缺陷描述：同#343模式，多个不应对外暴露aclnn接口的算子ACLNNTYPE误设为aclnn；group_norm_grad存在冗余binary配置
- 修复模式：ACLNNTYPE aclnn改为aclnn_exclude，删除冗余json
- 可审查性：高
- 审查规则建议：同#343

### #349 af87e30d aclnnGemm算子异常场景未拦截
- 根因类别：输入校验缺失（格式校验）
- 涉及文件：matmul/gemm/op_host/op_api/aclnn_gemm.cpp
- 缺陷描述：aclnnGemm缺少Format校验，NZ格式输入本应拦截但能跑通
- 修复模式：新增CheckFormat函数校验所有输入输出必须为FORMAT_ND
- 可审查性：高
- 审查规则建议：每个aclnn算子CheckParams应包含Format校验步骤

### #350 4e5a8b39 修改unused variable和unused parameter问题
- 根因类别：控制流缺陷（非void函数缺return）+ 编译警告
- 涉及文件：norm/ada_layer_norm_v2/op_host/op_api/aclnn_ada_layer_norm_v2.cpp等4个文件
- 缺陷描述：AdaLayerNormV2Calculate声明返回aclnnStatus但缺return语句，且调用处未检查返回值(control reaches end of non-void function)；其他是unused variable/parameter
- 修复模式：添加return ACLNN_SUCCESS；调用处增加CHECK_RET；移除未使用变量
- 可审查性：高
- 审查规则建议：开启-Werror=return-type将此类问题升级为编译错误

### #351 42f47c17 aclnnEinsum算子资料改正、aclnnMatmulWeightNz算子异常场景错误码改正
- 根因类别：错误码语义不匹配
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp
- 缺陷描述：BuildMatMulWeightNzGraph返回nullptr时使用ACLNN_ERR_INNER_NULLPTR错误码，但实际是参数非法导致，应返回ACLNN_ERR_PARAM_INVALID
- 修复模式：错误码从ACLNN_ERR_INNER_NULLPTR改为ACLNN_ERR_PARAM_INVALID
- 可审查性：中
- 审查规则建议：用户参数导致的校验失败应返回PARAM相关错误码而非INNER错误码

### #352 d303c3a0 删除qbmmv4算子def错误描述
- 根因类别：算子定义配置多余（不支持平台的配置）
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/quant_batch_matmul_v4_def.cpp
- 缺陷描述：def中包含不支持的310p平台配置config_310p(约82行)，可能导致图编译阶段错误匹配
- 修复模式：删除整段config_310p配置代码
- 可审查性：高
- 审查规则建议：def中AddConfig平台应与binary config的compute_units列表一致

### #353 5baade56 SegmentSum算子SIMT模板tiling显式设置workspace解决aic error问题
- 根因类别：框架约定遗漏（必须override的方法未实现）
- 涉及文件：index/segment_sum/op_host/arch35/segment_sum_simt_tiling.cpp, .h
- 缺陷描述：SIMT模板不用workspace但未实现GetWorkspaceSize()显式设为0，框架要求必须显式设置否则aic error
- 修复模式：新增GetWorkspaceSize() override，workspace[0]=0
- 可审查性：中
- 审查规则建议：所有tiling类必须override GetWorkspaceSize()；扫描继承tiling基类但未实现该方法的子类

### #354 bd286614 处理avgpool3d算子不生成二进制文件的问题
- 根因类别：头文件路径未同步（目录重命名后）
- 涉及文件：pooling/avg_pool3_d/op_kernel/avg_pool3_d_apt.cpp
- 缺陷描述：pool_3d_common目录从v35重命名为arch35后，10个#include路径仍引用旧路径，编译失败
- 修复模式：../pool_3d_common/v35/改为../pool_3d_common/arch35/
- 可审查性：高
- 审查规则建议：目录重命名PR应自动扫描所有引用旧路径的#include语句

### #355 692f43a9 处理adaptivemaxpool3d算子tiling判断错误
- 根因类别：逻辑运算符错误（||应为&&，恒真条件）
- 涉及文件：pooling/adaptive_pool3d_common/op_host/arch35/adaptive_pool3d_tiling.cpp
- 缺陷描述：条件npuArch!=A || npuArch!=B恒为true（De Morgan），所有平台都返回GRAPH_PARAM_INVALID
- 修复模式：移除多余条件，简化为只检查npuArch!=DAV_3510
- 可审查性：高
- 审查规则建议：x!=A||x!=B是经典恒真逻辑错误，可用clang-tidy/cppcheck的always-true检测

### #356 961b48b6 修改quantbatchmatmulinplaceadd算子soc version命名问题
- 根因类别：配置字符串拼写错误
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：compute_units中soc名写为ascend910_95（错误），正确为ascend950
- 修复模式：修正字符串
- 可审查性：高
- 审查规则建议：维护合法soc version枚举白名单，CI校验所有json配置的compute_units值

### #357 a84247e8 uniqueConsecutive算子return_idx为true拦截信息修改
- 根因类别：能力路由缺失（aicore不支持的场景未自动fallback到aicpu）
- 涉及文件：index/unique_consecutive/op_host/arch35/unique_consecutive_tiling_arch35.cpp, unique_consecutive_def.cpp
- 缺陷描述：return_idx=true时aicore不支持但缺少CheckSupport机制路由到aicpu，仅在tiling阶段拦截
- 修复模式：def中新增CheckSupport回调+NeedCheckSupportFlag(true)实现自动fallback
- 可审查性：中
- 审查规则建议：aicore有条件支持的算子应通过CheckSupport声明而非仅在tiling拦截

### #358 10bdb67b 修改aclnnDequantSwigluQuant.md文档
- 跳过-纯文档

### #359 300efeb1 aclnnConvolution测试Transpose1D带bias场景走matmul导致功能报错失败
- 根因类别：算子路由条件不完整
- 涉及文件：conv/convolution_forward/op_host/op_api/aclnn_convolution.cpp
- 缺陷描述：IsSupportConvTranspose1DToBmm未考虑weight L维>1且带bias的情况，此时Transpose导致维度不匹配matmul报错
- 修复模式：新增条件weight.shape[L_DIM]!=1&&bias!=nullptr时返回false不走matmul路径
- 可审查性：中
- 审查规则建议：算子优化路由中需确保转换后shape约束在所有可选输入(bias)存在时仍成立

### #360 ce7bfa39 添加qbmminplaceadd aclnn文件及tiling异常拦截
- 根因类别：tiling校验缺失（多项）
- 涉及文件：matmul/quant_batch_matmul_inplace_add/op_host/op_tiling/, op_api/等
- 缺陷描述：(1)未校验transA/transB合法组合 (2)未校验输出dtype必须DT_FLOAT (3)未校验shape维度和k轴一致性 (4)mx quant下scale shape未校验 (5)pertokenShape可能nullptr未拦截
- 修复模式：新增CheckShapeVaild/CheckParamsForMxQuant等校验方法
- 可审查性：高
- 审查规则建议：tiling的AnalyzeInputs/AnalyzeDtype/AnalyzeAttrs应对所有输入完整校验

### #361 f98df9bc 解决qbmm空tensor处理问题单
- 根因类别：边界条件缺失/多通路行为不一致
- 涉及文件：matmul/quant_batch_matmul_v3/op_api/aclnn_quant_matmul_v4.cpp, matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：静态图通路m/n=0返回空tensor正常，但aclnn直调/动态图/geir图通路拦截报错，行为不一致
- 修复模式：aclnn入口添加平台判断+维度为0的early return，区分NZ/ND格式分别处理
- 可审查性：中
- 审查规则建议：检查算子不同调用通路对边界输入(零维/空tensor)的处理是否一致

### #362 3da6cbf9 删除fastkernel冗余代码文件
- 跳过-纯代码清理

### #363 0e254120 删除多余日志
- 跳过-UT桩代码日志级别调整

### #364 4469304e BatchNormGrad算子修改UB空间计算错误
- 根因类别：数值计算错误（缺sizeof乘数+缺对齐）
- 涉及文件：norm/batch_norm_grad_v3/op_host/batch_norm_grad_v3_splitload_tiling.cpp等3个tiling文件
- 缺陷描述：CalcBubBlock()中ubTensorNotRelateChannelBlock=tmpChannelNum*noRelateBlockNum缺少*sizeof(float)，UB空间计算偏小；tmpChannelNum未做block对齐可能越界
- 修复模式：乘上sizeof(float)；新增GetAlignValue()对齐处理
- 可审查性：高
- 审查规则建议：UB/内存空间计算必须乘sizeof(dtype)；偏移/大小计算必须考虑对齐

### #365 908f0227 资料中yoffset描述有误
- 跳过-纯文档

### #366 f0eb3726 修改aclnn问题
- 跳过-纯文档

### #367 a17e0ee9 补齐A5算子op_graph下面的cmakelist、同步修改apply_adam_w算子geir通路动态用例失败问题
- 根因类别：构建配置缺失+冗余InferShape导致geir通路失败
- 涉及文件：多个op_graph/CMakeLists.txt(新增), optim/apply_adam_w/op_host/apply_adam_w_infershape.cpp(删除)
- 缺陷描述：(1)多个A5算子缺少op_graph/CMakeLists.txt导致graph plugin无法编译 (2)apply_adam_w冗余空InferShape与框架默认行为冲突致geir通路失败
- 修复模式：补齐CMakeLists.txt；删除冗余infershape.cpp
- 可审查性：中
- 审查规则建议：新增算子检查op_graph/CMakeLists.txt是否补齐；空InferShape应审视必要性

### #368 118b959d aclnnMatmulWeightNz资料错误修改
- 跳过-纯文档

### #369 6b9a0c39 修改代码扫描的问题
- 根因类别：整数溢出(GM偏移uint32) + 测试代码自比较bug
- 涉及文件：norm/add_rms_norm/op_kernel/add_rms_norm_multi_n.h, add_rms_norm_split_d.h, norm/group_norm_grad/op_kernel/group_norm_grad.h, norm/add_layer_norm/tests/ut/op_kernel/compare_data.py
- 缺陷描述：(1)kernel GM偏移用uint32_t计算乘法，大数据量溢出 (2)compare_data.py中np.isclose(tmp_gold,tmp_gold)自比较（第一参数应为tmp_out），精度比对永远PASS
- 修复模式：(1)GM偏移改为int64_t (2)修正np.isclose参数为tmp_out
- 可审查性：高
- 审查规则建议：(1)GM偏移计算禁止uint32乘法 (2)测试比对函数两参数不应相同（自比较检测）

### #370 1f044298 修改addmm接口参数和文档描述不符问题
- 根因类别：API签名与文档不一致(const修饰符)
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_addmm.cpp, aclnn_addmm.h
- 缺陷描述：stream参数声明为const aclrtStream与文档和其他算子不一致
- 修复模式：去掉stream参数的const修饰符
- 可审查性：高
- 审查规则建议：API声明参数修饰符应与文档和同类接口保持一致

### #371 e9ef7d29 renorm异常校验及ScaledMaskedSoftmaxGradV2增加拦截信息
- 根因类别：输入校验缺失(dim越界) + 错误信息缺失
- 涉及文件：norm/renorm/op_host/op_api/renorm.cpp, vfusion/scaled_masked_softmax_grad_v2/op_host/op_api/aclnn_scaled_masked_softmax_backward.cpp
- 缺陷描述：(1)renorm InferShape未校验dim越界/负值直接使用导致越界访问 (2)RenormAiCore返回值未检查 (3)CheckFormat失败无OP_LOGE信息
- 修复模式：添加dim范围校验；检查返回值；增加错误日志
- 可审查性：高
- 审查规则建议：用户输入参数(dim/axis)必须做边界校验；校验失败必须输出错误信息

### #372 a947b6ea 106 135 162 issue问题修改
- 根因类别：构建配置缺失(类别白名单) + 编译器指令缺失(__restrict)
- 涉及文件：cmake/ut.cmake, loss/mse_loss_v2/op_kernel/mse_loss_v2_base.h等
- 缺陷描述：(1)supportedCategory列表缺少loss类别致loss UT不被发现 (2)tilingData指针缺__restrict限定符
- 修复模式：添加loss到类别列表；指针加__restrict
- 可审查性：高
- 审查规则建议：新增算子类别目录时检查构建系统类别白名单是否同步更新

### #373 bf19309a 修改md（易用性问题）
- 跳过-纯文档

### #374 1d67b94e 修改opapi UT引用路径问题
- 跳过-纯UT路径修改

### #375 31501829 添加index类算子部分opapi UT及修改aclnnEmbeddingBag中的错误注释
- 根因类别：注释复制粘贴错误 + 构建配置缺陷
- 涉及文件：index/embedding_bag/op_host/op_api/aclnn_embedding_bag.h, cmake/gen_ops_info.cmake
- 缺陷描述：(1)aclnn头文件注释中函数名写成aclnnInplaceIndexCopyGetWorkspaceSize应为aclnnEmbeddingBagGetWorkspaceSize (2)cmake中有不合理的skip逻辑导致部分算子binary不被编译
- 修复模式：修正注释函数名；删除cmake不合理skip逻辑
- 可审查性：高
- 审查规则建议：API注释中引用的函数名必须与实际函数名匹配

### #376 92060846 pool目录头文件错误修改
- 根因类别：注释复制粘贴错误（大面积函数名引用错误）
- 涉及文件：pooling/adaptive_max_pool3d_grad/..aclnn_adaptive_max_pool2d_backward.h等4个pool头文件
- 缺陷描述：多个pool算子头文件注释中"由第一段接口xxx获取"的函数名写错（如写成aclnnMaxPool2DWithIndicesBackwardGetWorkspaceSize、aclnnAtan2GetWorkspaceSize等）
- 修复模式：修正注释中函数名引用
- 可审查性：高
- 审查规则建议：两段式接口头文件注释应自动生成或有校验脚本确保函数名一致性

### #377 3d71222b 修改有误的注释内容
- 根因类别：注释复制粘贴错误（大面积，11个文件）
- 涉及文件：norm/add_rms_norm_dynamic_quant/.., quant/ascend_quant_v2/.., quant/dynamic_quant/.., quant/quantize/.., quant/trans_quant_param_v2/..等11个头文件
- 缺陷描述：批量注释错误，如aclnnAscendQuant写成aclnnAscendAntiQuantGetWorkspaceSize，aclnnQuantize写成aclnnTopkGetWorkspaceSize等
- 修复模式：批量修正注释中的函数名引用
- 可审查性：高
- 审查规则建议：同#376，需自动化校验

### #378 964be4dd 修改norm下样例代码问题
- 根因类别：示例代码缺陷（数据类型不匹配+资源泄漏）
- 涉及文件：norm/deep_norm/examples/test_aclnn_deep_norm.cpp, norm/deep_norm_grad/examples/test_aclnn_deep_norm_grad.cpp
- 缺陷描述：(1)deep_norm_grad示例vector<int32_t>但创建tensor传ACL_FLOAT，类型不匹配 (2)deep_norm示例缺aclrtFree(betaDeviceAddr)内存泄漏
- 修复模式：int32_t改float；补充aclrtFree调用
- 可审查性：高
- 审查规则建议：示例代码vector类型必须与aclDataType匹配；aclrtMalloc/aclrtFree必须配对

### #379 e14c4eee 修改index类算子linear_index_v2的kernel UT中的error日志误报
- 根因类别：UT代码多处bug（tiling定义不匹配/数据路径错误/GmBuffer未指定大小）
- 涉及文件：index/linear_index_v2/op_kernel/linear_index_v2.h, 多个UT文件
- 缺陷描述：(1)SetGlobalBuffer缺dataNum_参数致error日志 (2)UT tiling结构体与实际不匹配 (3)数据生成脚本和路径引用错误
- 修复模式：修正SetGlobalBuffer参数；重命名结构体；修正路径；补齐tiling参数
- 可审查性：中
- 审查规则建议：SetGlobalBuffer应始终指定buffer大小参数；UT tiling结构体应与产品代码同步

### #380 c1d89aa2 修改index类算子kernel UT中的error日志误报
- 跳过-纯UT修改
