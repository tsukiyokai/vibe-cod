# ops-nn-dev 缺陷提交深度分析

## 分析进度

- 总缺陷提交：612条
- 已处理：476条（第1-502行），其中分析393条，跳过83条
- 剩余：136条（第503-612行）

---

### a70d8b16d5e6800d796b98bf5b01999d98f70362 fix hoAL10 cal error
- 根因类别：计算逻辑错误
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_api_tiling_base.cpp, conv/conv2d_v2/op_host/op_tiling/arch35/ 多文件(共14文件)
- 缺陷描述：计算L1缓存中最小所需的feature map高度hoAL1min时，当wo < m0时通过CeilDiv(m0, wo)计算得到的值可能超过实际ho值，导致hiAL1min超出实际输入高度，L1 size校验产生错误判断。同时opImplMode做位运算前缺少非负校验，且使用了错误的k0来源(cubeInfo.k0而非按weight dtype查询的k0)
- 修复模式：对hoAL1min增加std::min(..., ho)上界约束；对opImplMode增加>=0校验并转uint32_t后做位操作；将k0改为从CUBE_MKN_TAB按实际weight dtype动态查询
- 可审查性：高
- 审查规则建议：1) tiling参数通过除法/向上取整计算时，检查结果是否需要用实际shape维度做上界clamp 2) 有符号整数做位运算前应校验非负或转无符号类型 3) 使用硬件参数表时检查是否与实际数据类型匹配

### 12e796c4fc0927379fa2590c5ec5b69f7b36c54e fix foreach_mul_scalar_v2 aclnn storage shape bug
- 根因类别：shape处理错误(storage shape未同步)
- 涉及文件：foreach/foreach_mul_scalar/op_host/op_api/aclnn_foreach_mul_scalar_v2.cpp
- 缺陷描述：CheckShape函数校验self和out的view shape一致性后，未将out tensor的storage shape同步为view shape，非连续tensor场景下后续计算使用错误的storage shape
- 修复模式：增加SetStorageShape(GetViewShape())同步
- 可审查性：高
- 审查规则建议：算子对输出tensor做shape校验时，检查是否需要同步更新storage shape，特别是foreach类算子

### fa03888b0d0144e8d59b0f7bcf7e1486b00768ea add embeddingBag bugFix
- 根因类别：多核初始化错误 + 计算逻辑错误 + 资源管理错误
- 涉及文件：index/embedding_bag/op_kernel/arch35/ 多文件(6文件)
- 缺陷描述：(1) 空bag时每个核独立对全局内存做零初始化存在竞争 (2) validIndicesFactorNumber跨loop累计导致后续loop有效行数不正确 (3) TQue不需要队列语义应用TBuf (4) DisposalBagSize直接写全局内存不安全 (5) tiling中UB大小错误除以DOUBLE_BUF
- 修复模式：全局初始化移到core 0+SyncAll；替换为循环内局部变量；TQue改TBuf；重写为DMA拷贝；去掉多余DOUBLE_BUF除法
- 可审查性：中
- 审查规则建议：1) 多核场景全局内存初始化应由指定核完成并加同步屏障 2) 跨循环累积计数器需确认语义是全局累积还是每次迭代独立 3) 不需要队列语义时用TBuf而非TQue

### 997a5518461caf365b6907df890cdaccdbb6609a fix dsq-aclnn资料
- 纯文档修复，跳过

### 93690b823fa9513deb3a373753722fe0599f9218 bugfix wqbmmv2 only support x contiguous
- 根因类别：平台特定约束校验遗漏
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_api/aclnn_weight_quant_batch_matmul_v2.cpp
- 缺陷描述：DAV_3510平台不支持x tensor转置，但校验逻辑transposeX || IsContiguous(x)对DAV_3510不正确，该平台要求x必须连续且不能转置
- 修复模式：增加平台判断分支，DAV_3510使用!transposeX && IsContiguous(x)
- 可审查性：高
- 审查规则建议：算子存在平台差异化约束时，检查校验逻辑是否按平台区分。添加新平台支持时审查所有输入约束

### 7004b8e99922340407fb24d9bb6c3fbef0d94f8d 修复QuantBatchMatmulV3纯CUBE模板低阶API的若干问题
- 根因类别：常量定义重复 + 条件编译逻辑错误 + 类型支持遗漏
- 涉及文件：matmul/common/cmct/block/ 多文件, quant_batch_matmul_constant.h, quant_batch_matmul_v3_apt.cpp 等(7文件)
- 缺陷描述：(1) 多文件重复定义相同常量(IDX_M_TILE_IDX等)与统一头文件命名不一致 (2) 条件编译#if != 应使用#else (3) IsCapable缺少DT_INT64 scale dtype支持 (4) CUBE模板使用条件宏不一致
- 修复模式：常量集中到统一头文件；#if改#else；添加DT_INT64判断；引入CUBE_TEMPLATE_ND宏
- 可审查性：中
- 审查规则建议：1) 常量定义集中单一头文件避免重复 2) 互斥条件编译分支用#else 3) 新增数据类型时检查所有能力判断/路由逻辑同步更新

### 08b9ebb1123e0e89001b5bc2fadf8e34a9089437 修复onnx算子ConvTranspose图通路动态shape在2D场景执行失败的问题
- 根因类别：维度场景适配遗漏(2D/3D混用)
- 涉及文件：conv/common/op_host/conv_backprop_infershape.cpp/.h, conv/conv3d_transpose_v2/op_host/conv3d_transpose_v2_infershape.cpp
- 缺陷描述：2D场景tensor扩展为5D(D=1)，但CheckOutputAllZero要求所有维度为0才推断输出shape，D=1导致返回false，infershape流程失败。另外输入索引硬编码为0而非正确常量kConv3DTransposeXIdx(=1)
- 修复模式：新增CheckOutputAllZeroFrom2D函数处理D=1的特殊情况；修复索引为正确常量
- 可审查性：高
- 审查规则建议：1) 同时支持2D和3D且通过维度扩展统一处理时，所有依赖维度值的判断需考虑扩展维度默认值 2) 输入索引用命名常量不硬编码

### f4ffed6948841b5acd20598ea9bf9dc38152da9e fix maxpoolwithargmaxv3 CHW HWC
- 根因类别：输入format支持遗漏(3D tensor未处理)
- 涉及文件：pooling/max_pool_with_argmax_v3/op_host/arch35/max_pool_with_argmax_v3_tiling_base.cpp
- 缺陷描述：tiling逻辑仅支持4维输入(NCHW/NHWC)，3维tensor(CHW/HWC)维度校验直接失败
- 修复模式：放宽维度校验允许3维；3维时推断format并将batch设为1，映射到已有4维逻辑
- 可审查性：高
- 审查规则建议：维度校验不应仅支持一种维度数，需考虑框架实际传入的维度变体(如batch维省略的3D输入)

### e2fc1fecb181aaea32809992a5da09615cf56ae1 group场景精度问题
- 根因类别：条件判断错误(边界条件)
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/conv3d_backprop_input_v2/conv3d_dx_rowc_block.h
- 缺陷描述：group卷积场景下，>=判断将合法尾部core和超范围core合并处理，超范围core使用错误数据量计算导致精度问题
- 修复模式：拆分为==处理尾部core和>直接return两个分支
- 可审查性：高
- 审查规则建议：多核任务分配中，必须同时处理"合法tail core"和"超范围idle core"，不能合并到同一分支

### dc3a8850e7817cd926e0aeba0c9ee350d81bb34a fix bug for DynamicQuant
- 根因类别：输入校验遗漏 + 日志错误
- 涉及文件：quant/dynamic_quant/op_host/dynamic_quant_tiling_arch35.cpp, op_api/aclnn_dynamic_quant.cpp
- 缺陷描述：(1) groupNum仅校验上界未校验<=0的非法值 (2) 错误日志"pertensor"与实际检查条件"perchannel"矛盾
- 修复模式：校验改为groupNum <= 0 || groupNum > MAX_EXPERT_NUM；日志修正为perchannel
- 可审查性：高
- 审查规则建议：1) 数值范围校验应同时包含上下界，特别是用于除法/索引的参数 2) 错误日志条件描述必须与实际判断条件一致

### db5a08d10f260616567d901e99fdca51e191e0fa 修改新包__bf16未定义的报错
- 根因类别：类型定义缺失/编译兼容性
- 涉及文件：pooling/adaptive_max_pool3d/op_kernel/adaptive_max_pool3d_big_pool.h
- 缺陷描述：CPU侧单元测试编译时(__CCE_KT_TEST__宏)，__bf16硬件内建类型未定义导致编译报错
- 修复模式：增加条件编译将__bf16定义为bfloat16_t别名
- 可审查性：高
- 审查规则建议：kernel代码使用硬件内建类型(__bf16等)时，检查是否在__CCE_KT_TEST__宏下提供CPU侧兼容定义

### 7b7999e9e003bd15617781e8a1980633a73b5664 [dx] fix K_M_EXT_STEP error
- 根因类别：计算逻辑错误/M维度上界不正确
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/conv3d_bp_func.h
- 缺陷描述：TPL_NO_SPLIT_KERNEL模式下baseUseM_可能超过实际有效M维度，load3d的padList[3]=255导致实际M小于tiling给出的baseM，访问超范围数据
- 修复模式：根据实际H/W/dilation/stride/padding重新计算有效M值，当M < baseUseM_且mIter_==1时修正baseUseM_
- 可审查性：低
- 审查规则建议：tiling参数直接用于计算时，审查是否存在tiling值大于实际有效数据范围的情况，需与实际值做min

### 7aa778e0c8d17aab28f98dca8374df22fe4a9ab3 group场景精度问题
- 根因类别：分组卷积尾块计算逻辑错误
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/conv3d_backprop_input_v2/conv3d_dx_rowc_block.h
- 缺陷描述：group convolution最后一个group的N维度与其他group不同(cin不整除group)，原代码统一使用cinG和nCnt_导致边界判断对最后group不正确，nGroupCoreTail_计算公式也有误(cin - group*cinG为负值)
- 修复模式：新增nTailCnt_表示最后group的N迭代次数，修正尾块判断条件和nGroupCoreTail_计算为tailN % singleShapeN_
- 可审查性：中
- 审查规则建议：group convolution中审查最后group的通道数是否单独处理(cin % group != 0的情况)

### 616b4699ee0054f0fc78a1e7894f81fc74b40637 aclnnInplaceIndexCopy大shape性能劣化修复
- 根因类别：条件判断遗漏/性能回退
- 涉及文件：index/scatter_update/op_host/op_api/aclnn_index_copy.cpp
- 缺陷描述：IndexCopy非连续场景无条件走ScatterUpdate路径，大shape时性能反而更差，缺少数据规模判断
- 修复模式：提取为独立函数，新增stride和数据量门控条件(stride*elemSize >= 128B 或 总数据量 < 512KB)
- 可审查性：中
- 审查规则建议：引入新算子分派路径时，审查启用条件是否考虑数据规模维度，性能优化路径需有回退机制

### 58f275c6dc75bbcf8ce1f879832ea62422d69df2 修复dynamicQuant 310p精度问题
- 根因类别：DMA参数溢出/硬件限制未处理
- 涉及文件：quant/dynamic_quant/op_kernel/dynamic_quant.cpp, dynamic_quant_unalign_large_310p.h 等
- 缺陷描述：310P芯片DMA搬运参数为uint16_t，stride值可能超过UINT16_MAX导致截断，数据搬运错误引发精度问题。scaleLocal在多处反复DeQue存在queue管理问题
- 修复模式：增加stride溢出检测，超限时降为逐行搬运；scaleLocal的DeQue提前到外层循环通过参数传递
- 可审查性：中
- 审查规则建议：DMA搬运指令的stride/nBurst等参数检查是否可能超过硬件限制(uint16_t上限65535)

### 46f5817ef1679e1ce818d6ef04e6f98f036b5fd5 [dx] fix precision error
- 根因类别：资源管理错误/全载模式状态未正确切换
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/conv3d_backprop_input_v2/conv3d_dx_rowc_block.h
- 缺陷描述：B矩阵偏移地址变化时未切换全载状态(enableFullLoad)，之前全载加载的L1数据已失效但仍被使用，导致精度问题
- 修复模式：新增preOffsetB_和preEnableFullLoad跟踪上一轮状态，偏移变化时释放L1并关闭全载
- 可审查性：低
- 审查规则建议：L1/L0缓存全载优化在多batch/多group循环中，检查全载条件是否因参数变化而失效

### 3e45851a13dbd55b95aea1028d67c532b56ff976 fix aclnnScatter支持0维tensor
- 根因类别：边界条件遗漏/0维tensor处理缺失
- 涉及文件：index/scatter_elements_v2/op_host/op_api/aclnn_scatter.cpp
- 缺陷描述：CheckTensorDim未处理0维tensor(标量)，GetDimNum()返回0与1维不匹配，标量scatter被拒绝
- 修复模式：dimSize为0时视作1维处理(赋值为1)再做一致性校验
- 可审查性：高
- 审查规则建议：输入校验中检查是否正确处理0维tensor(标量)，维度数用于比较/计算时0维是常见边界条件

### fa1305c56ef80179035f8ec7284875eb3c6126d2 修复aclnnKthValue在索引超过int32时的aicore问题
- 根因类别：整数类型溢出/索引类型选择不当
- 涉及文件：index/gather_v2/op_api/aclnn_kthvalue.cpp
- 缺陷描述：排序索引硬编码为DT_INT32，大shape下最后一维超过INT32_MAX时索引溢出
- 修复模式：新增GetSortIndicesType函数根据shape动态选择int32/int64
- 可审查性：高
- 审查规则建议：索引/offset类型为int32时，审查索引值是否可能超过int32范围，大shape下需动态选择int32/int64

### e7048cff4f29a467a6f285e72e497e6cd5ac1701 asw-kernel-fix
- 根因类别：计算逻辑错误/Bias加载时机错误
- 涉及文件：matmul/mat_mul_v3/op_kernel/arch35/mat_mul_asw_kernel.h
- 缺陷描述：splitK模式下Bias错误地在最后一次K迭代加载，应在第一次(kIndex==0)加载
- 修复模式：Bias加载条件从kIndex == splitKRound-1改为kIndex == 0
- 可审查性：高
- 审查规则建议：splitK场景Bias/残差加载审查是否在正确迭代轮次，Bias应在首次累加时加入(kIndex==0)

### e49d5d27c0877efd4b7707a2fd491b4870fd776e fix dx nullptr
- 根因类别：空指针解引用
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：只需计算dw不需dx时，gradInput为nullptr，但代码用gradInput的shape计算mmDwOutShape2d并调用ViewWithShape导致crash
- 修复模式：改用gradWeight的shape计算mmDwOutShape2d
- 可审查性：高
- 审查规则建议：可能为nullptr的tensor指针在解引用前需null检查，反向传播中dx/dw/dbias可能为nullptr

### e25e6e60c70c9f218b190e497dc065dfd4682522 fix modulate buffer bug
- 根因类别：UB内存分配计算错误（tiling与kernel的buffer计算不一致）
- 涉及文件：vfusion/modulate/op_host/modulate_regbase_tiling.cpp, vfusion/modulate/op_kernel/arch35/modulate_regbase_common.h, vfusion/modulate/tests/ut/op_host/test_modulate_tiling.cpp
- 缺陷描述：tiling侧用`ubSize_ / DOUBLE_BUFFER / bufferCount`公式估算buffer大小，但kernel侧实际对x和y各占2份buffer(double buffer)，scale/shift又有不同策略，导致tiling计算出的buffer大小与kernel实际分配不匹配，UB溢出或数据覆盖
- 修复模式：统一tiling侧公式为`ubSize_ / (X_Y_BUFFER_NUM * DOUBLE_BUFFER + 1 + hasScaleShift)`，kernel侧scale/shift统一NO_DOUBLE_BUFFER，消除两端不一致
- 可审查性：中
- 审查规则建议：tiling侧buffer计算必须与kernel侧InitBuffer做交叉比对，确认buffer数量和double buffer策略一致；建议用共享常量定义buffer数量

### c00e940bfab0b85cf9c0c5f21c4faa377afd269e fix package size
- 根因类别：配置遗漏（新算子未注册到二进制打包配置）
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：新增HardSwishGradV2算子未在ascendc_config.json中注册，导致kernel二进制不被打包
- 修复模式：在ascendc_config.json中添加HardSwishGradV2及其compute_units配置
- 可审查性：高
- 审查规则建议：新增算子时checklist需确认已更新打包配置；CI可自动校验新增算子目录是否在config中有对应条目

### b14f5a03cac76f0bb6e25d4b64e1ab30ca4f17df fix inplace_add_rms_norm kernelmode
- 根因类别：接口签名不匹配（缺少模板参数和函数参数）
- 涉及文件：norm/inplace_add_rms_norm/op_kernel/inplace_add_rms_norm.cpp, classify_rule.yaml
- 缺陷描述：GENERAL_OP_IMPL宏调用op.Init()缺少workspace参数；模板类只传1个类型参数但实际需要第二个kernel mode参数
- 修复模式：为Init添加workspace参数；为所有GENERAL_OP_IMPL模板类添加第二个参数`1`；更新classify_rule.yaml
- 可审查性：高
- 审查规则建议：修改kernel类Init签名或模板参数后，全局搜索所有调用点确认同步更新

### 2d424df8e624fd50b2911fd3f09a596343a43db9 fix repeatinterleave infershape
- 根因类别：控制流缺陷（设置状态后缺少return）
- 涉及文件：index/repeat_interleave/op_host/repeat_interleave_infershape.cpp
- 缺陷描述：两处SetUnknownRank后没有return语句，代码继续执行后续shape推导逻辑，对UnknownRank的shape调用GetDimNum可能返回异常值导致越界
- 修复模式：在两处SetUnknownRank后各添加`return ge::GRAPH_SUCCESS;`
- 可审查性：高
- 审查规则建议：设置特殊状态(UnknownRank/UnknownShape)的分支必须确认包含return；InferShape函数应做静态分析确保所有路径正确返回

### 245e9e45efcc611a6a3622265f333b80a1bdbb6e 修复EmbeddingHashTableApplyAdamW精度性能问题
- 根因类别：哈希表查找逻辑缺陷 + 类型精度问题 + 代码重复
- 涉及文件：hash/embedding_hash_table_apply_adam_w/op_kernel/arch35/ 多文件
- 缺陷描述：(1)线性探测循环内嵌套了embedding维度计算循环，导致未匹配key时也执行计算，且查找与grad计算的循环嵌套关系不正确 (2)b16和b32维护两个几乎相同的实现文件 (3)b32的PostProcess硬编码float类型转换
- 修复模式：删除重复b32版本，统一为模板实现；重写哈希表查找逻辑：先探测找key，found后再批量计算embedding维度
- 可审查性：低
- 审查规则建议：(1)哈希表probe逻辑和data处理应明确分离不交叉嵌套 (2)同一算法多类型版本应用模板替代代码复制 (3)kernel中间计算统一提升到float

### cbd1092fab563aa500c0a883ba6f8cb669032d41 fix hwnc tilingkey
- 根因类别：tilingkey定义不完整（宏参数缺失）
- 涉及文件：conv/conv2d_v2/op_kernel/arch35/conv2d_v2_tilingkey.h
- 缺陷描述：HWNC格式的tiling key宏定义缺少DisContinuous参数选择项，与其他格式的同类宏参数维度不一致，导致tiling key匹配错误
- 修复模式：在宏末尾追加DisContinuous参数选择项（默认CONV_DIS_CONTINUOUS_CLOSE）
- 可审查性：中
- 审查规则建议：存在多个变体宏（按format区分）时，逐一比对各变体参数列表是否完整一致

### 9c46dc5f1914fa2b01895682c679b9b342224bd0 addrmsnorm 95问题修复
- 根因类别：算子输出参数约束错误（REQUIRED vs OPTIONAL）
- 涉及文件：norm/add_rms_norm/op_host/add_rms_norm_def.cpp, norm/add_rms_norm/op_host/config/ascend910_95/add_rms_norm_binary.json
- 缺陷描述：910_95平台配置中rstd和x输出被标记为REQUIRED，但实际应为OPTIONAL，调用方不需要这些输出时框架强制校验导致失败
- 修复模式：def.cpp和binary.json中共8处paramType从REQUIRED改为OPTIONAL
- 可审查性：中
- 审查规则建议：新增平台配置时检查output的paramType是否与设计文档一致；def.cpp与binary.json的paramType声明必须双向一致

### 15138bbd86c161144966aafc17cc3457cac1f157 fix aclnnScatterAdd 注释
- 跳过：纯注释修改，仅更新头文件中API注释的数据类型支持列表描述

### db31f7273006cdde19bce89f2c4c0fc49d244002 修复TransQuantParamV2 310p上二进制增幅异常
- 根因类别：平台约束遗漏（编译配置缺少目标平台）
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：TransQuantParamV2算子的compute_units列表缺少ascend310p，该平台二进制不被预编译导致运行时JIT编译引发异常
- 修复模式：在compute_units数组中添加"ascend310p"
- 可审查性：高
- 审查规则建议：对照平台支持矩阵逐一确认compute_units覆盖所有目标平台

### 952313ace0bca58196fd11d67f59dafa7dd1ee4f 修正kStartPt溢出翻转问题
- 根因类别：整数溢出（硬件指令参数位宽限制）
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_basic_block_tiling.cpp/.h
- 缺陷描述：load3d指令kStartPt字段上限65535(16位)，当k0*hk*wk超过65536时发生溢出翻转，导致指向错误内存位置
- 修复模式：在checkLargeSpecs()中新增LOAD3D_KSTART_MAX=65535常量，当load3dK>65536时走超大kernel切分路径
- 可审查性：低
- 审查规则建议：涉及硬件指令参数时确认所有字段位宽限制，tiling代码中对这些字段的取值范围做边界校验

### 4f87bea9ab075b07008587aadf3a09041c4fd273 FusedCrossEntropyLossWithMaxSum算子回退
- 根因类别：平台约束遗漏 / 平台适配错误
- 涉及文件：loss/fused_cross_entropy_loss_with_max_sum/ 配置文件(删除910_95), activation/heaviside/, index/scatter_list/, norm/deep_norm/ 等多文件
- 缺陷描述：混合提交：(1)FusedCrossEntropyLossWithMaxSum在910_95平台有问题需回退 (2)多个算子平台版本判断过于严格，如`__CCE_AICORE__ == 220`应为`>= 220`
- 修复模式：回退问题算子的910_95配置；将精确匹配改为范围匹配或增加新平台枚举
- 可审查性：中
- 审查规则建议：(1)`__CCE_AICORE__ == xxx`精确匹配审查是否应用范围比较 (2)新增平台应有checklist检查所有相关算子

### 38bd1c5380231b94bcb301962808e9771a0a2432 aic问题修复
- 根因类别：边界条件缺失（数值越界未保护）
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/impl/dav_v310/conv_bp_sub_func.h
- 缺陷描述：CalcCutInWIndexForOnlyH函数中headWi_可能超过baseUseM_，后续使用headWi_的逻辑产生AIC错误
- 修复模式：headWi_计算后增加上界保护`if (headWi_ > baseUseM_) headWi_ = baseUseM_`
- 可审查性：中
- 审查规则建议：取模/取余计算的中间变量审查值域是否在后续使用的合法范围内；应在变量赋值处就做约束

### 30b91291450210a172c043704370cdd801f84406 fix mse_loss tilingdata bug
- 根因类别：数据结构定义不一致（自定义结构体与标准结构体字段布局不匹配）
- 涉及文件：loss/mse_loss/op_host/arch35/ 多文件, loss/mse_loss/op_kernel/arch35/mse_loss_tiling_def.h
- 缺陷描述：自定义ReduceOpTilingDataV2与标准ReduceOpTilingData字段布局不一致，手工逐字段拷贝导致kernel侧解析tiling数据出错
- 修复模式：删除自定义结构体和转换函数，直接使用标准ReduceOpTilingData
- 可审查性：高
- 审查规则建议：禁止手动镜像标准库数据结构定义，直接引用原始头文件；tiling数据结构host和kernel必须用同一份定义

### e71db0907d151071d3099fc8526e2df5bfe9d815 fix aclnnMedian
- 根因类别：边界条件缺失（空Tensor输入/输出场景混淆）
- 涉及文件：index/gather_v2/op_api/aclnn_median.cpp
- 缺陷描述：self->IsEmpty()和valuesOut->IsEmpty()用`||`合并处理，但输入空和输出空语义不同：输出空应直接返回不做任何操作
- 修复模式：拆分为两个独立分支，先判断输出空直接返回，再判断输入空走特殊处理
- 可审查性：高
- 审查规则建议：多条件用`||`合并时审查各条件是否确实需要相同处理；空Tensor的输入/输出应分别考虑

### e41444931b663806b8968942b424f52d64b1bd93 修复tilingkey计算错误的问题
- 根因类别：计算逻辑错误（非连续场景缺少条件守卫）
- 涉及文件：index/gather_elements/op_host/arch35/gather_elements_no_contiguous_tiling.cpp
- 缺陷描述：CoalesceGatherElements()被无条件调用，但index转置场景下维度压缩不适用，导致tiling key计算错误
- 修复模式：调用前增加`if (!isIndexTranspose_)`条件判断
- 可审查性：高
- 审查规则建议：算子存在多种执行路径时审查每个步骤是否对所有模式适用；bool模式标志应检查关键路径是否遗漏判断

### e2f856d387301b5daff7d382ca74e7971bef2e88 fix example & add aclnnadvancestepv2 atk
- 根因类别：示例代码错误（shape/data不匹配、头文件名大小写错误）
- 涉及文件：optim/advance_step/docs/aclnnAdvanceStepV2.md, optim/advance_step/examples/, optim/advance_step/tests/st/
- 缺陷描述：文档示例代码多处问题：头文件名V2大小写错、输入shape与API不匹配、data和shape张冠李戴、参数值不正确
- 修复模式：重新整理输入tensor的shape和data定义使其与API一一对应；新增独立示例和atk测试
- 可审查性：高
- 审查规则建议：示例代码必须通过实际编译运行验证；检查CreateAclTensor的data、shape、deviceAddr是否匹配

### 51f8247aee7db5da0f5697f09a909604ffb20d09 inplace_index_add_确定性问题fix
- 根因类别：流水线同步错误（硬件事件依赖类型错误）
- 涉及文件：index/inplace_index_add/op_kernel/arch35/inplace_index_add_determinstic.h, index/scatter_nd_add/op_kernel/arch35/scatter_nd_add_determinstic.h
- 缺陷描述：CopyOut前使用S_MTE2同步但CopyOut走MTE3通道，应使用S_MTE3和V_MTE3事件，原代码同步错误通道导致CopyOut可能在计算未完成时执行
- 修复模式：将S_MTE2替换为S_MTE3+V_MTE3双重同步；同步点移到Muls之后CopyOut之前
- 可审查性：低
- 审查规则建议：CopyOut/CopyIn前的同步事件必须与实际DMA通道匹配(MTE2搬入/MTE3搬出)；Vector计算后紧跟CopyOut需等待V_MTE3

### 0e5504b36d0806da206efdba14aecbaeb45c6ebd fix multi_add_rms_norm_dynamic_quant binary & cmake
- 根因类别：混合缺陷（构建配置错误 + 无符号整数比较 + 缺少return）
- 涉及文件：norm/multi_add_rms_norm_dynamic_quant/ 多文件(op_host/CMakeLists.txt, simplified_key.ini, infershape.cpp, tiling.cpp等)
- 缺陷描述：(1)ACLNNTYPE aclnn应为aclnn_exclude (2)section名全大写下划线与算子驼峰注册名不匹配 (3)size_t相减判断<0永远不成立(无符号下溢) (4)OP_CHECK_IF失败分支写了ge::GRAPH_FAILED但缺少return
- 修复模式：分别修复4个独立问题
- 可审查性：中
- 审查规则建议：size_t变量间做减法比较需警惕无符号下溢；OP_CHECK_IF失败分支确认是否需要return；simplified_key.ini的section名必须与算子注册名一致

### 0d47b59fdd5e766e52568a438869dba0448a94cf fix dynamic_quant_update_scatter_v2 aclnn
- 根因类别：构建配置错误（不应生成aclnn接口）
- 涉及文件：quant/dynamic_quant_update_scatter_v2/op_host/CMakeLists.txt
- 缺陷描述：ACLNNTYPE设为aclnn但该算子不应生成aclnn接口，导致编译失败或接口冲突
- 修复模式：ACLNNTYPE aclnn改为aclnn_exclude
- 可审查性：中
- 审查规则建议：新算子CMakeLists.txt的ACLNNTYPE应与设计文档中接口暴露策略一致

### 0d322070406a886cfb22218e615c0a9a80acfb72 SwishGrad精度修复
- 根因类别：数据结构定义不一致 + tiling key计算错误
- 涉及文件：activation/swish_grad/op_host/arch35/swish_grad_tiling_arch35.cpp, swish_grad_tiling_arch35.h, op_kernel/arch35/swish_grad_tilingdata.h
- 缺陷描述：SwishGradTilingData结构体中value字段冗余，scale字段类型应为float而非int64_t导致精度丢失；tiling key计算使用了未正确初始化的局部变量schMode，应从tiling->baseTiling.scheMode获取并转uint64_t
- 修复模式：修正TilingData结构体字段（删除冗余、修正类型），修正tiling key数据来源
- 可审查性：高
- 审查规则建议：TilingData结构体变更需与kernel端消费代码交叉审查确保字段布局和类型一致；tiling key计算应直接引用tiling结构体值而非外部变量

### c28b94f7bcb9b4a9febe030bec0dc1d2fee53e6a fix aclnnMedian empty
- 根因类别：边界条件处理缺失（空tensor）
- 涉及文件：index/gather_v2/op_api/aclnn_median.cpp, aclnn_median.h
- 缺陷描述：aclnnMedian在输入为空tensor时仅将workspaceSize设0返回，未对输出tensor填充值。对标PyTorch：整型应返回类型最小值，浮点应返回NaN
- 修复模式：新增DealMedianEmptyTensor函数按数据类型分别填充边界值（INT64填MIN_INT64，INT32填MIN_INT32，浮点填NAN）
- 可审查性：中
- 审查规则建议：所有aclnn算子必须显式处理空tensor输入且行为应与竞品对齐；新增算子review checklist应包含"空tensor行为"

### a3f3447463ef6556bfc6d7baa0c1946698f4b04a add leaky relu api check
- 根因类别：输入校验缺失（输出dtype兼容性未检查）
- 涉及文件：activation/leaky_relu/op_api/aclnn_leaky_relu.cpp
- 缺陷描述：CheckDtypeValid只检查输入self的dtype是否在支持列表内，未检查输出out的dtype能否由self类型转换而来，不兼容时静默产生错误结果
- 修复模式：扩展校验函数增加out参数，补充OP_CHECK_RESULT_DTYPE_CAST_FAILED校验
- 可审查性：高
- 审查规则建议：所有aclnn算子dtype校验必须同时检查输入和输出tensor的类型兼容性

### 6beab61dd0ee9c84809e45882dab39c835c35a4e fix change (flat_quant)
- 根因类别：混合缺陷（属性默认值错误 + 流水线同步缺失 + 宏隐藏控制流）
- 涉及文件：quant/flat_quant/op_graph/flat_quant_proto.h, op_host/flat_quant_def.cpp, op_kernel/arch35/flat_quant_high_v2.h, op_kernel/flat_quant_apt.cpp
- 缺陷描述：(1)dst_dtype默认值DT_INT32(3)应为DT_INT4(29)与算子实际输出类型矛盾 (2)两次matmul操作间缺少PipeBarrier<PIPE_ALL>()同步，matmulR结果可能尚未完成matmulL就读取 (3)INVOKE_FLAT_QUANT_IMPL宏中有return语句导致TILING_KEY_IS(5)分支提前退出，与其他分支不一致
- 修复模式：修正属性默认值 + 补充流水线同步 + 内联展开消除宏中隐藏控制流
- 可审查性：中
- 审查规则建议：OP_REG属性默认值须与OUTPUT数据类型定义兼容；连续matmul操作间必须插入PipeBarrier；禁止在宏中使用return语句

### 6657ff815962ceb355bee58290d5bf2407bfaa9c Fix slice but shape not valid
- 根因类别：条件判断不完整 + fallback路径逻辑缺失
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp, matmul_util.h, matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp
- 缺陷描述：IsSliceNonContiguous缺少NZ格式不支持判断和左右矩阵dtype必须相同的约束，导致不满足条件的tensor错误进入slice优化路径。此外slice不支持的3D tensor应先转连续再fold batch维度重算m/n/k，原代码缺少needFoldBatch的fallback逻辑
- 修复模式：将分散的前置校验集中到判断函数 + 新增needFoldBatch降级路径
- 可审查性：中
- 审查规则建议："是否支持某优化路径"的判断应封装在统一入口而非散布多处；matmul shape变更后必须重新刷新m/n/k信息

### 175e2720e5903d59d85fb1bd990134369e0f186b 解决qbmm空tensor处理的问题单
- 根因类别：边界条件缺失（多通路行为不一致）
- 涉及文件：aclnn_quant_matmul_v4.cpp, aclnn_quant_matmul_v5.cpp
- 缺陷描述：qbmm算子静态图通路中m/n=0能正确返回空tensor，但aclnn直调和torch动态图通路缺少对m=0/n=0的early return，导致被拦截报错。多通路行为不一致
- 修复模式：在函数前部增加m=0/n=0边界检查，根据NZ/ND格式分别提前return
- 可审查性：高
- 审查规则建议：多通路调用同一算子时，各入口对边界输入(空tensor/零维度)的处理必须一致

### 048a67c900053103d5f1daf7b93899438a5db221 conv soc fix
- 根因类别：硬编码SoC标识/平台抽象不足
- 涉及文件：conv模块下约60个文件(conv_base.h/cpp, conv_api_tiling_util.h, conv_api_tiling_base.cpp等)
- 缺陷描述：conv算子全面依赖SocVersion枚举(ASCEND910_95等)做平台判断，新SoC加入时无法扩展；UB切分vector/cube比例硬编码为常量VEC_NUM_PER_CUBE_910D=2
- 修复模式：SocVersion替换为NpuArch架构级枚举；硬编码参数改为运行时platformInfo查询(aivPerAic等)
- 可审查性：中
- 审查规则建议：禁止在tiling代码中硬编码具体芯片型号，统一通过NpuArch或平台API获取；硬件参数不允许使用magic number

### fb3b8365b964132fdd467ecc63724fb8996cc083 ut compile fix
- 根因类别：构建配置缺失（include路径遗漏）
- 涉及文件：cmake/ut.cmake
- 缺陷描述：op_api的UT编译缺少OPAPI_INCLUDE路径导致头文件找不到编译失败
- 修复模式：补全CMake include path
- 可审查性：高
- 审查规则建议：新增op_api模块时CI应跑完整UT编译验证；CMake审查checklist包含头文件路径是否同步

### e32a5eaae3d3988f184d92b1be14db26669a535d fix allocated oversize of FusedLinearCrossEntropyLossGrad
- 根因类别：内存分配过大（workspace计算公式错误）
- 涉及文件：fused_linear_cross_entropy_loss_grad_tiling.cpp
- 缺陷描述：MemFriendlyTilingFunc计算userWorkspaceByteSize时对BT*H和V*H两项错误乘了BUFFER_NUM(双buffer系数)，这两个buffer实际只需单份空间，多分配了(BT*H+V*H)*IN_BYTE_SIZE的量
- 修复模式：去掉两行的BUFFER_NUM乘子
- 可审查性：高
- 审查规则建议：workspace计算中每项乘以BUFFER_NUM时应逐项注释说明为什么需要双buffer；代码审查时逐行核对公式各项物理含义

### d1832c87c936cbf2bf352c7ef8e73e3e3c6d3858 fix log for check func
- 根因类别：复制粘贴错误（日志变量引用错误）
- 涉及文件：fused_quant_matmul_checker.cpp
- 缺陷描述：CheckDimValue中检查scale维度但报错时误用offsetShape->GetStorageShape().GetDim(0)（offset的shape），应为scaleShape的dim(0)，导致日志误导调试
- 修复模式：修正报错信息中的变量名使其与判断条件一致
- 可审查性：高
- 审查规则建议：错误信息/日志引用的变量应与上文条件判断中的变量一致；检测"条件用A但报错引用B"的模式

### ba224cb4c335a8a8883f9bda56b6b31d8e863726 mxfp4 bugfix
- 根因类别：buffer对齐粒度错误
- 涉及文件：dynamic_mx_quant_tail_axis.h
- 缺陷描述：Init中outQueue_和outBuffer_的buffer大小对齐基准使用UBBlockSize_(32B)，但fp4数据需按vlForHalfNumber_(更大粒度)对齐，使用较小粒度导致buffer不够大，vector操作可能越界
- 修复模式：将对齐计算基准从UBBlockSize_改为vlForHalfNumber_
- 可审查性：中
- 审查规则建议：buffer InitBuffer对齐粒度须与数据类型的vector处理粒度匹配；sub-byte数据类型(fp4等)需特别审查对齐常量

### 9af284829b26035a8a78732aa7059b80bf5c870d fix 负载均衡性能裂化
- 根因类别：条件判断边界遗漏
- 涉及文件：adaptive_sliding_window_tiling.cpp
- 缺陷描述：balanceAfterFixp条件原仅判断kSize<1024启用负载均衡优化，但kSize==1024且nCore>=8时N轴不均衡占比已很小也应启用M轴优化，遗漏此边界case导致性能裂化
- 修复模式：条件扩展为kSize<1024 || (kSize==1024 && nCore>=8)
- 可审查性：中
- 审查规则建议：tiling中<阈值判断应同时考虑==时的行为；性能相关条件分支修改应附带benchmark对比

### 8233cc99456acc404b3f1bda42689d7fe209db0b bugfix: 清除unused变量
- 根因类别：代码冗余/编译告警 + UT逻辑缺陷
- 涉及文件：index/repeat_interleave/op_host/op_api/aclnn_repeat_interleave.cpp, tests/ut/
- 缺陷描述：IntToTensor中AllocScalar返回值赋给repeatsScalar但从未使用产生unused告警；UT中BF16测试无条件走test_run_invalid但部分芯片(910_95/910B~910E)实际支持BF16
- 修复模式：删除未使用变量；UT按芯片版本选择test_run/test_run_invalid
- 可审查性：高
- 审查规则建议：启用-Werror=unused-variable将告警升级为错误

### 793745bf14d1f586bf57ac91e41c06bf662e4d6e fix RepeatInterleave config json
- 根因类别：配置遗漏
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：RepeatInterleave在ascend910_95的编译配置缺少compile_options(-mllvm -cce-aicore-dcci-before-kernel-end=false)，而同族的RepeatInterleaveGrad已有此配置
- 修复模式：补充JSON配置项
- 可审查性：低
- 审查规则建议：新增算子时强制对比同族算子(XXX与XXXGrad)的配置一致性；编写脚本检测同族算子间compile_options差异

### 597274964667176934a7dd2d2b1e075afd967e5c fix check bug
- 根因类别：复制粘贴错误（空指针检查对象错误）
- 涉及文件：conv/convolution_forward/op_host/op_api/aclnn_quant_convolution.cpp
- 缺陷描述：TransDataPreProcess中对weight执行TransData转换后，紧接着的CHECK_NULLPTR却检查input而非weight，是复制上一行检查代码后忘改变量名。weight返回nullptr将不被捕获导致后续空指针崩溃
- 修复模式：CHECK_NULLPTR(input,...)改为CHECK_NULLPTR(weight,...)
- 可审查性：高
- 审查规则建议：连续相似CHECK调用重点比对检查对象与赋值对象是否匹配

### 4ddbefef5897c5db30b388cbb21b7de34fc0c9be fix unstall package
- 根因类别：打包配置遗漏
- 涉及文件：scripts/package/ops_nn/ops_nn.xml
- 缺陷描述：卸载配置缺少python和python/site-packages目录声明导致卸载时未清理。此外新增行使用install_mode属性名但文件其余一致用install_mod（无e），拼写不一致可能是新bug
- 修复模式：补充XML目录条目（但属性名拼写不一致存疑）
- 可审查性：中
- 审查规则建议：对XML配置文件建立schema校验限定合法属性名；检测同文件内属性名拼写一致性

### 1fcd73b68091ed0b500af53cb81242f86f930a48 fix log
- 跳过-纯文档/注释：仅将日志字符串"y format is not support"改为"y format does not support"英文语法修正

### 1999ccf22ff67a7010a02fcb203ad655032e3187 qbmm int8_to_int32问题修复
- 根因类别：模板分支逻辑错误（多余代码路径）
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_apt.cpp
- 缺陷描述：BIASMODE==TPL_EXCLUDE且DTYPE_SCALE为FLOAT/BF16分支中，存在TPL_KERNELTYPE==TPL_NO_VEC_EPILOGUE_WITH_MMAPI的if constexpr分支直接实例化MatMulASWKernel，在int8->int32场景下被错误匹配执行且模板类型参数不正确，而下方MxType分支中已有该KernelType的正确处理
- 修复模式：直接删除多余且错误的模板特化分支
- 可审查性：低
- 审查规则建议：if constexpr模板分支需注释说明适用条件和数据类型约束；新增KernelType分支时需更新分派矩阵并在CR中对照

### 13f10ef11478af52d09b9cb25a159ece6c7f59f7 回退超大shape
- 根因类别：边界条件缺失（超大shape未拦截）
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：ConvBackpropInput在W=1,B=1情况下可转Matmul路径优化，但未限制shape大小。当(m*k+k*n+n*m)*typeSize超过L2 cache(128MB)时转Matmul会导致性能劣化或结果异常
- 修复模式：新增IsGreaterL2Cache函数，计算矩阵内存超128MB L2 cache时返回false阻止conv->matmul转换
- 可审查性：中
- 审查规则建议：conv/matmul路径转换的条件判断应包含shape×dtype size的上界校验；优化路径需考虑硬件资源(L2 cache)限制

### 90cd650db33769846904b07e515283758f8b82de aclnn_quant_matmul_v5 check bugfix
- 根因类别：维度索引取反 + scale shape计算遗漏
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：InferGroupSize中两个bug：(1)MicroScaling路径下scaleSizeK未乘2，e8m0 scale shape是[m,k/2,2]取dim(1)得到k/2而非k (2)非MicroScaling路径transX1=true时应取倒数第二维但取了最后一维，transX1=false反之，维度索引条件写反
- 修复模式：MicroScaling路径scaleSizeK乘2；非MicroScaling路径交换transX1的true/false分支维度索引
- 可审查性：高
- 审查规则建议：transX1/transX2条件分支中的维度索引审查时需逐一核对：转置时K在倒数第二维，非转置时K在最后一维

### 3bd8c6a2738d339fa9e477628f664f091f926b5c 3ddx tiling hk hkwk bugfix
- 根因类别：边界条件遗漏（tiling模式组合未覆盖退化场景）
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/conv3d_backprop_input_v2_inner_product_tiling.cpp, conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/conv3d_bp_large_attribute_func.h, conv/conv3d_backprop_input_v2/op_kernel/arch35/convolution_3d_backprop/impl/dav_v310/conv_bp_sub_func_load_gm_to_l1a.h
- 缺陷描述：Conv3D反向(dx)算子在tiling模式为TILING_HK且dedx_w==1时未被正确处理。原代码多处条件判断只考虑TILING_HK_WK模式，遗漏了TILING_HK && dedx_w==1这一等效场景，导致L1缓冲区大小计算错误。kernel侧ComputeForWkLoop中realWoSize_计算结果可能<=0但未做保护返回；ComputeForTilingHk中前放大补零的跳过逻辑缺失；unlikely()宏括号位置错误只包裹了第一个条件
- 修复模式：tiling侧3处条件扩展为TILING_HK_WK || (TILING_HK && dedx_w==1)；增加realWoSize_<=0的提前返回；增加补零部分的continue逻辑；修正unlikely()括号位置
- 可审查性：低
- 审查规则建议：多种tiling模式存在时，审查每个条件分支是否覆盖所有模式组合下的维度退化场景（如宽度=1）

### 10cab125fa9d6709b0f7b753dda6bdf8b389f6fd 修复算子缺失json simpledKey配置问题
- 根因类别：配置遗漏（impl_mode缺失）
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：ScatterAdd、ScatterNd、ScatterNdAdd三个算子的impl_mode字段为空字符串""，缺失"high_precision"配置，导致未使用高精度实现模式
- 修复模式：将impl_mode从""改为"high_precision"
- 可审查性：高
- 审查规则建议：scatter类算子配置时检查impl_mode是否需要配置为high_precision

### 0081270b594bade272e0839fceb4cf699da03593 fix maxpoolwithargmaxv3 bigkernel mulcore
- 根因类别：内存寻址方式错误 + 计算逻辑错误
- 涉及文件：pooling/max_pool_with_argmax_v3/op_kernel/arch35/max_pool_with_argmax_v3_big_kernel_mul_core.h
- 缺陷描述：两处bug：(1) Init中设置GlobalBuffer时使用workspace[VALUE_WORKSPACE_SIZE]下标取址，应改为指针加法workspace + VALUE_WORKSPACE_SIZE；(2) RealIndex中splitW==0分支对index做了CeilValue对齐后的alignBlockLen来计算行列坐标，但应直接用curkW做除法和取模，对齐操作引入错误的index映射
- 修复模式：改为指针加法；去掉alignBlockLen中间变量直接用curkW计算
- 可审查性：中
- 审查规则建议：kernel中GlobalBuffer地址计算应使用指针算术而非下标；index还原逻辑中不应错误引入对齐操作

### dec58d1b24e052b805c4af99fa25309109c63990 baddbmm问题
- 根因类别：复制粘贴错误（参数重复）
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp, matmul/batch_mat_mul_v3/op_host/op_api/aclnn_baddbmm.cpp
- 缺陷描述：aclnnAddbmmGetWorkspaceSize中调用isAddBmmProcessEmptyTensor(batch1, batch1)，第二个参数应为batch2却写成batch1。同样错误存在于aclnnBaddbmmGetWorkspaceSize中。导致只检查batch1是否为空tensor而忽略batch2
- 修复模式：将两处第二个参数从batch1改为batch2
- 可审查性：高
- 审查规则建议：函数调用中有多个同类型参数(batch1/batch2、input/output)时检查是否存在复制粘贴导致的参数重复

### ac64e3d38bde3e1976eaa10cf2c025b2e801bd88 fix ascendc_config.json
- 根因类别：编译选项配置遗漏
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：约20个算子(Relu/ReluV2/Sigmoid/Gelu/NLLLoss等)在ascend910_95平台缺少-mllvm -cce-aicore-dcci-before-kernel-end=false编译选项
- 修复模式：批量补充compile_options
- 可审查性：中
- 审查规则建议：ascend910_95平台算子需检查是否配置dcci相关编译选项

### 2dcfbd35aeb03cd5cb22bc97cf99fea0014a6fad 回退gather_v2算子docs中误删的内容
跳过：纯文档修改

### f70cd870918fb011bf0db1acaad8d7bf9b77c738 fix msdagrad SetScheduleMode
- 根因类别：调度模式配置遗漏
- 涉及文件：vfusion/multi_scale_deformable_attention_grad/op_host/multi_scale_deformable_attention_grad_tiling.cpp, vfusion/multi_scale_deformable_attn_function/op_host/multi_scale_deformable_attn_function_tiling.cpp
- 缺陷描述：MultiScaleDeformableAttentionGrad算子缺少SetScheduleMode(1)调用；MultiScaleDeformableAttnFunction在TilingKey==0分支下同样缺少
- 修复模式：在对应位置添加SetScheduleMode(1)
- 可审查性：中
- 审查规则建议：tiling函数中需要batch调度的多核算子应检查SetScheduleMode调用

### 84ab237ad39a9c734b519983e736e3a51b4452aa ScatterNdAddSimtTiling 修复int64精度问题
- 根因类别：数据类型约束遗漏
- 涉及文件：index/scatter_nd_add/op_host/arch35/scatter_nd_add_tiling_base.cpp, scatter_nd_add_tiling_base.h
- 缺陷描述：SelectTiling中判断是否进入排序模板的条件未排除int64数据类型，排序模板不支持int64精度，当updateDtype_为DT_INT64时进入排序路径导致精度错误
- 修复模式：排序条件中增加updateDtype_ != ge::DT_INT64过滤
- 可审查性：高
- 审查规则建议：选择计算模板/优化路径时检查是否对所有支持的数据类型兼容，特别是int64等特殊精度类型

### 5c796d90aab3e2a1c48896e0c0aedbbc0846d5c6 fix SetScheduleMode
- 根因类别：调度模式配置遗漏
- 涉及文件：quant/swi_glu_quant/op_host/swi_glu_quant_tiling.cpp
- 缺陷描述：SwiGluQuant的tiling函数缺少SetScheduleMode(1)调用
- 修复模式：定义BATCH_MODE=1常量，tiling函数入口调用context->SetScheduleMode(BATCH_MODE)
- 可审查性：高
- 审查规则建议：新增tiling函数时确认是否需要SetScheduleMode

### 5bb4b5c77d635e0f11685232786c88be9347d767 fix: a16f8 n=1 not supported
- 根因类别：边界条件处理不当（退化维度）
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/weight_quant_batch_matmul_v2_tiling.cpp, weight_quant_batch_matmul_v2_tiling.h
- 缺陷描述：A16F8量化场景下antiquant_scale shape为(n,)，当n=1时被识别为PER_TENSOR并报错"A16F8 only support perchannel"，但n=1的perchannel语义上合法
- 修复模式：新增ConfigureReuseScenarios方法，A16F8场景PER_TENSOR自动转换为PER_CHANNEL复用逻辑；CheckTempLimit扩展允许PER_TENSOR通过
- 可审查性：中
- 审查规则建议：量化算子的quantType判定逻辑应检查n=1等退化维度下的边界处理

### 56bd8fbcc314c7edd69d245b252e42820d83fb16 门禁ut修复
- 根因类别：Python类型转换逻辑错误
- 涉及文件：scripts/util/parse_changed_files.py, scripts/util/parse_compile_changed_files.py
- 缺陷描述：is_experimental = bool(sys.argv[2])对字符串参数做类型转换，Python中bool("FALSE")返回True（非空字符串均为True），导致传入"FALSE"时is_experimental被错误设为True
- 修复模式：改为sys.argv[2] == 'TRUE'精确字符串比较
- 可审查性：高
- 审查规则建议：Python中对命令行字符串参数不应使用bool()转换，应用字符串精确比较

### 09e5a53e8dda8c1d1871d1fde6dcce13e0363467 fix ascendc_config.json
- 根因类别：编译选项配置遗漏
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：约20个算子(ForeachNonFiniteCheckAndUnscale/AdamApplyOne/EluGrad等)在ascend910_95平台缺少dcci编译选项
- 修复模式：批量补充compile_options
- 可审查性：低
- 审查规则建议：新增算子到ascendc_config.json时检查目标平台compile_options是否完整

### d3a7b1956cb73744331a6bd2f918a059844eb52b relu_v2和layer_norm_v3算子代码格式化
跳过：纯代码格式化/文档修改

### d1382de350f6284edbba722f629a36ba221c502b DTS2026011320178：RmsNormGrad 修改日志错误信息
- 根因类别：日志消息错误（copy-paste）
- 涉及文件：norm/rms_norm_grad/op_host/rms_norm_grad_tiling.cpp
- 缺陷描述：CheckRstdShape中三处OP_LOGE日志将rstd与x的shape比较错误写成"dy first few dim"，实际检查的是x而非dy
- 修复模式：统一修正日志消息为"Input rstd shape invaild, shape is not equal x first few dim."
- 可审查性：高
- 审查规则建议：日志消息中引用的变量名/tensor名应与实际检查的对象一致

### aa95cab24383f060204b409a7b97f371ca872d1a 修复matmul目录下example内存泄漏问题
- 根因类别：资源泄漏（内存泄漏）
- 涉及文件：matmul/batch_mat_mul_v3/examples/test_aclnn_addbmm.cpp等共15个文件
- 缺陷描述：多个example中aclTensor和device内存在CHECK_RET失败提前return时不会执行到末尾的aclDestroyTensor/aclrtFree释放逻辑；Inplace调用复用同一workspaceAddr可能导致第一段分配被覆盖后无法释放
- 修复模式：使用std::unique_ptr搭配自定义deleter(aclDestroyTensor/aclrtFree)封装，RAII确保异常路径资源释放；Inplace操作引入独立workspace变量
- 可审查性：高
- 审查规则建议：ACL API代码中aclrtMalloc/aclCreateTensor分配的资源应使用RAII封装确保所有return路径正确释放

### 9150a7cca8b69b222d68e8a9f1f31845667f9b75 fix scatterElementsV2 支持double/bool
- 根因类别：数据类型支持遗漏
- 涉及文件：index/scatter_elements_v2/op_host/scatter_elements_v2_asc_tiling.cpp, scatter_elements_v2_asc_tiling.h, op_kernel/scatter_elements_v2_apt.cpp, config/ascend910_95/scatter_elements_v2_binary.json
- 缺陷描述：ScatterElementsV2在tiling层的dtype集合中缺少DT_DOUBLE和DT_BOOL，kernel层预编译条件未将BOOL归入int8同族/DOUBLE归入int64同族处理，binary配置缺少分发条目
- 修复模式：dtype集合添加DT_DOUBLE/DT_BOOL；kernel中按字节宽度归入对应分支；bool的REDUCTION_ADD添加cast到half；补充binary配置和UT
- 可审查性：中
- 审查规则建议：新增数据类型支持时需同步更新tiling层dtype校验、kernel层预编译条件、binary配置三处

### 80e3f64102391e5f57659cdf0d6895ad90c7c0cb fix schedulemodel
- 根因类别：调度模式配置位置错误
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch32/conv3d_backprop_input_v2_base_tiling.cpp, pooling/adaptive_max_pool3d_grad/op_host/adaptive_max_pool3d_grad_normal_tiling.cpp, adaptive_max_pool3d_grad_tiling_base.cpp
- 缺陷描述：Conv3DBackpropInputV2的PostTiling中错误设置了SetScheduleMode(1)（该算子不需要）；AdaptiveMaxPool3DGrad将SetScheduleMode(1)放在PostTiling基类中但应在DoOpTiling的Normal分支
- 修复模式：从Conv3DBackpropInputV2移除；从AdaptiveMaxPool3DGrad基类移至Normal分支的DoOpTiling中
- 可审查性：中
- 审查规则建议：SetScheduleMode调用位置应确认在正确的算子和tiling分支中，避免copy-paste引入错误

### 601831d25c2c34ddbc2336720a3b40bd9848e949 layerNorm fix compile_options
- 根因类别：编译选项配置遗漏
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：LayerNormV4缺少ascend910_95平台compile_options和mc62cm12a平台支持；LayerNormV3条目完全缺失
- 修复模式：补充LayerNormV4的compile_options和平台支持，新增LayerNormV3完整配置
- 可审查性：高
- 审查规则建议：算子迁移到新平台时检查ascendc_config.json中compile_options和平台支持

### a66a28aa164f88cbb044dbef04ef5be2fc920e65 fix cmake file for max_pool_grad_with_argmax
- 根因类别：CMake构建配置缺失
- 涉及文件：pooling/max_pool_grad_with_argmax/CMakeLists.txt, pooling/max_pool_grad_with_argmax/op_host/CMakeLists.txt(新建), pooling/max_pool_grad_with_argmax_common/CMakeLists.txt, pooling/max_pool_grad_with_argmax_common/op_host/CMakeLists.txt(新建)
- 缺陷描述：顶层CMakeLists.txt缺少COMPUTE_UNIT和TILING_DIR参数；op_host子目录缺少CMakeLists.txt
- 修复模式：添加SUPPORT_COMPUTE_UNIT和SUPPORT_TILING_DIR变量；新建op_host/CMakeLists.txt
- 可审查性：高
- 审查规则建议：新增算子目录的CMakeLists.txt应包含COMPUTE_UNIT和TILING_DIR参数，op_host需独立CMakeLists.txt

### 87d126b21d8011cf0d264cf1171b3c33e3f2d796 MaxPoolV3 opapi告警修复
- 根因类别：编译告警（返回值const冗余）+ 注释错误
- 涉及文件：pooling/max_pool_v3/op_api/aclnn_max_pool.cpp
- 缺陷描述：PoolingOutShape和CheckOutputShape返回类型static inline const int64_t / static const bool中const修饰值类型返回值冗余触发编译器告警；View5Das4D/View5Das3D注释中维度编号和操作名与实际不一致
- 修复模式：移除返回类型冗余const；修正注释维度描述
- 可审查性：高
- 审查规则建议：函数返回基础值类型时不应添加const修饰；注释中维度编号应与代码逻辑一致

### ec83d69ca84b00a55f8b442e04629ee2a98b1156 【MatMul】修复fusedmatmul带bias时性能问题
- 根因类别：Tiling参数计算遗漏(bias占用L1空间)
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_asw_tiling.cpp, matmul/fused_mat_mul/tests/ut/op_host/test_fused_matmul_tiling.cpp
- 缺陷描述：CalL1Tiling计算stepKa/stepKb时未扣除bias table在L1中的占用空间(BIAS_TABLE_NUM*DATA_SIZE_FP32)，导致带bias时stepK过大，L1实际可用不足引起性能退化
- 修复模式：在CalL1Tiling后重新计算remainSizeForAL1BL1(有bias时减去bias table大小)，基于剩余空间重算stepKa/stepKb
- 可审查性：中
- 审查规则建议：Tiling计算涉及L1/UB空间分配时，检查是否遗漏了辅助数据结构(bias/scale/offset table)的空间占用

### dfe06e8bc1878cf540de03b90833503189fecfe3 解决BatchNormGrad ub空间不够问题
- 根因类别：UB空间计算公式错误(缺乘数+对齐缺失)
- 涉及文件：norm/batch_norm_grad_v3/op_host/batch_norm_grad_v3_base_tiling.cpp, batch_norm_grad_v3_base_tiling.h, batch_norm_grad_v3_splitload_crosscore_tiling.cpp, batch_norm_grad_v3_splitload_tiling.cpp
- 缺陷描述：CalcBubBlock中ubTensorNotRelateChannelBlock = tmpChannelNum * noRelateBlockNum缺少sizeof(float)乘数，计算出的UB占用偏小导致实际分配不足；同时tmpChannelNum未按dtype对齐
- 修复模式：补充sizeof(float)乘数；新增GetAlignValue方法按FLOAT_BLOCK_SIZE或HALF_BLOCK_SIZE对齐tmpChannelNum
- 可审查性：高
- 审查规则建议：UB/L1空间计算公式必须统一单位(字节vs元素)，所有维度参与空间计算前必须乘以sizeof(dtype)

### c625ae7a243ec57a5c008c445c2b39bd627c3e15 1952 perf fix
- 根因类别：AIC/AIV控制流不一致(AIV只执行一次Iterate)
- 涉及文件：conv/common/op_kernel/arch35/conv_common_func.h
- 缺陷描述：IterateAll中AIC分支在while循环中迭代，但AIV分支只调用一次Iterate(非循环)，导致AIV未正确迭代处理所有数据块
- 修复模式：将AIV的SetOutputGm提前到while循环前，统一AIC/AIV共用while(Iterate)循环，内部按ASCEND_IS_AIC_CONV/ASCEND_IS_AIV_CONV分支处理
- 可审查性：中
- 审查规则建议：AIC/AIV双核架构中，两个分支的循环结构和迭代次数必须对等审查

### aad25fdf620e5c3a270291f3a0feae3d99cdca6b 移除extendConvTranspose op def，消除编译报错
- 根因类别：废弃代码未清理导致编译错误
- 涉及文件：conv/extend_conv_transpose/op_host/extend_conv_transpose_def.cpp(删除)
- 缺陷描述：extendConvTranspose的op_def文件已不再使用，但其引用的类型或宏在新版本中变更导致编译报错
- 修复模式：删除整个废弃的op_def文件
- 可审查性：高
- 审查规则建议：算子废弃或重构时确保清理所有关联的def/registration文件

### 797589989afa9c31155a167e838c862e21e31fd7 修复aclnn错误码错误
- 根因类别：错误码语义使用错误
- 涉及文件：quant/trans_quant_param_v2/op_host/op_api/aclnn_trans_quant_param_v2.cpp
- 缺陷描述：CheckNotNull对用户传入参数的空指针检查返回ACLNN_ERR_INNER_NULLPTR(内部错误)，应返回ACLNN_ERR_PARAM_NULLPTR(参数错误)
- 修复模式：更正错误码为ACLNN_ERR_PARAM_NULLPTR
- 可审查性：高
- 审查规则建议：用户输入校验的错误码必须使用PARAM_*系列，INNER_*仅用于内部不可达状态

### 74cdae15d7d781c60ebb7bb80a1439e11c04013e Revert NLLLossGrad performence optimize
- 根因类别：性能优化引入功能回退(Revert)
- 涉及文件：loss/nll_loss_grad/op_host/arch35/nll_loss_grad_tiling_arch35.cpp, nll_loss_grad_tiling_arch35.h, loss/nll_loss_grad/op_kernel/arch35/nll_loss_grad.h, nll_loss_grad_4d.h(恢复合并), nll_loss_grad_base.h(恢复合并), nll_loss_grad_tiling_key.h(恢复合并), nll_loss_grad_apt.cpp
- 缺陷描述：NLLLossGrad的大规模性能优化重构(拆分为4d/base/tiling_key等多个文件)引入了问题，被完全回退到优化前版本
- 修复模式：全量Revert，恢复为单文件实现
- 可审查性：低(大规模重构难以审查)
- 审查规则建议：大规模算子重构应分步提交，每步可独立验证；性能优化PR必须附带完整精度对比测试

### 52bb1ef24a23d11d72216c775904cada9b42581d fix dynamicquant perf
- 根因类别：性能优化(VF计算重构)
- 涉及文件：quant/dynamic_quant/op_kernel/arch35/dynamic_quant_regbase_full_load.h
- 缺陷描述：原实现逐行调用ComputeVF，每行重复Scale/Y计算初始化开销大
- 修复模式：拆分为DataCopyInputVF/ComputeScaleVF/ComputeYVF，传入multiRow批量处理
- 可审查性：低
- 审查规则建议：[性能优化，非功能缺陷]

### 407b68e17356d7cb2d97b75e007e982362746aee [MMV3] 回退对齐场景下workspace申请逻辑，解决对齐场景内存膨胀
- 根因类别：workspace计算公式错误(对齐场景使用全局尺寸)
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp, matmul_v3_base_tiling.h
- 缺陷描述：DETERMINISTIC_SPLIT_K模式下workspace用alignedM*alignedN全局对齐尺寸计算singleSize，对齐场景下(M/N被pad到很大值)导致workspace内存申请远超实际需要
- 修复模式：新增GetDeterministicSplitKWorkspaceSize方法，BASE模式用singleCoreM*singleCoreN(实际每核尺寸)，非BASE模式保持alignedM*alignedN
- 可审查性：高
- 审查规则建议：workspace计算中涉及对齐后的维度时，区分全局对齐尺寸和每核实际尺寸，避免对齐膨胀被乘以核数

### 20d81cfa23cd4cc713842b0cca0bc3089ad02070 CCEC指令适配1952后的临时方案回退
- 根因类别：平台临时方案未及时清理
- 涉及文件：conv/common/op_kernel/arch35/conv_instr_hw_mode_impl.h, conv_instr_impl.h, conv_instr_m_mode_impl.h
- 缺陷描述：为Ascend910_95(5102)平台添加的#ifdef __NPU_ARCH__==5102条件编译临时方案(使用raw intrinsics如img2colv2_cbuf_to_ca/load_cbuf_to_cb代替LoadData API)，在API适配完成后应清理
- 修复模式：删除所有#ifdef __NPU_ARCH__==5102分支，统一使用LoadData/Load3DBitModeParam API
- 可审查性：高
- 审查规则建议：#ifdef __NPU_ARCH__临时平台适配代码必须附带清理计划和跟踪issue

### e147ca470a58cb58b1b8eb615e14ae946304a890 Fix the warnings when compiling for AddLayerNormQuant op.
- 根因类别：编译告警(未使用变量+类型混用+返回值丢失)
- 涉及文件：norm/add_layer_norm_quant/op_host/add_layer_norm_quant_empty_tiling.cpp, add_layer_norm_quant_tiling.h
- 缺陷描述：多个问题：1)x2StorageShape/betaStorageShape/y1StorageShape声明后未使用；2)rows_/cols_/usedCoreNum_/rowsPerCore_/rowsLastCore_用int64_t但实际不会为负(uint64_t更合适)；3)CalcUsedCoreNums返回void但内部调用CalcuTilingData的返回值被丢弃；4)maxOutputSize为uint64_t但被检查<0(永远为假)
- 修复模式：移除未使用变量；int64_t→uint64_t；CalcUsedCoreNums改返回ge::graphStatus并传递CalcuTilingData返回值；移除无意义的负数检查
- 可审查性：高
- 审查规则建议：函数返回值必须被使用或显式忽略；无符号类型不应与负数比较；声明的变量必须使用

### cb0f64cbcb0ce5c8ad9795bab760413165123d23 修复CMakeList以及补充二进制json
- 根因类别：二进制配置json参数错误
- 涉及文件：pooling/max_pool_v2/op_host/config/ascend910_95/max_pool_v2_binary.json
- 缺陷描述：ksize和strides的index都是0(应为0,1,2递增)；shape固定为[1,4]不支持动态shape(应为-2)；padding的value为空数组(应为"SAME")
- 修复模式：修正index为1和2；shape改-2；padding value设"SAME"
- 可审查性：高
- 审查规则建议：binary.json中多个input的index必须连续递增；动态shape算子shape应为-2；attrs必须有有效默认值

### b4076b657853f08242092a3632c79f84b8cd3e5f Fix fp32 align
- 根因类别：对齐粒度硬编码错误(half vs实际输出类型)
- 涉及文件：matmul/common/cmct/epilogue/block_epilogue_cv.h, matmul/common/cmct/epilogue/block_epilogue_elementwise.h
- 缺陷描述：AlignBlock<half>硬编码half类型进行16元素对齐，当输出类型为fp32时应按8元素对齐(32B/4B)，对齐粒度错误导致计算越界或精度异常
- 修复模式：AlignBlock<half>改为AlignBlock<DataTypeOut>，根据实际输出数据类型决定对齐粒度
- 可审查性：高
- 审查规则建议：AlignBlock/CeilAlign的模板参数必须与实际操作的数据类型一致，不应硬编码特定类型

### a79b527fe83c4f55f18936c1a724fec61987ac4f BNGV3 sync bugfix
- 根因类别：流水线同步事件管理缺陷(条件性WaitFlag导致同步不完整)
- 涉及文件：norm/batch_norm_grad_v3/op_kernel/arch35/batch_norm_grad_v3_split_r1_regbase.h
- 缺陷描述：多处WaitFlag<MTE3_MTE2>被条件化(if ni>0/basicBlockIdx<BUFFER_NUM等)，导致某些迭代路径跳过必要的同步等待；CopyOutDx后只SetFlag不WaitFlag就释放tensor，可能导致DMA写出未完成就被覆盖
- 修复模式：移除所有条件性WaitFlag；CopyOutDx后的SetFlag<MTE3_MTE2>后立即无条件WaitFlag<MTE3_MTE2>
- 可审查性：中
- 审查规则建议：SetFlag和WaitFlag必须成对出现且无条件执行；不应在循环边界条件中跳过同步事件

### 90649ac29984878c587dc0c9d1bdf5ad859d9e49 MaxPoolGradWithArgmaxV3 oom fix
- 根因类别：Tiling参数未区分normal/tail核(OOM)
- 涉及文件：pooling/max_pool_grad_with_argmax_v3/op_host/arch35/max_pool_grad_with_argmax_v3_nchw_tiling_scalar.cpp, max_pool_grad_with_argmax_v3_nchw_tiling_scalar.h, max_pool_grad_with_argmax_v3_simt_tiling.cpp, max_pool_grad_with_argmax_v3_ksize_one_tiling.cpp, op_kernel/arch35/max_pool_grad_with_argmax_v3_nchw_scalar.h
- 缺陷描述：原CalcGradArgmaxInner对normal核和tail核使用相同的argmaxInner参数，tail核的highAxisTail可能远小于highAxisInner，但使用highAxisInner计算的buffer尺寸分配，导致tail核实际处理时内存溢出
- 修复模式：拆分CalcGradArgmaxInner为normal版和CalcGradArgmaxInnerTail，分别用highAxisInner和highAxisTail计算；新增SetNormalInner和SetTailInner设置outer/tail参数
- 可审查性：高
- 审查规则建议：多核tiling中normal核和tail核的buffer尺寸必须分别计算，tail核的数据量通常小于normal核

### 8e56b958aaf22a70e9b3179b97b6dd86fda250a8 fix opkernel ut issue when run with ophost ut
- 根因类别：CMake构建配置(UT符号链接缺失)
- 涉及文件：cmake/ut.cmake
- 缺陷描述：kernel UT单独执行正常，但与ophost UT一起编译时legacy_common_manager.cpp不会被编入tiling obj，导致kernel UT符号缺失链接失败
- 修复模式：CMake条件编译中增加legacy_common_manager_stub.cpp桩代码源文件，在UT_TEST_ALL或OP_HOST_UT时自动链入
- 可审查性：中
- 审查规则建议：UT的CMake配置应确保独立运行和联合运行两种模式下符号链接均完整

### 79d5ee94de98881825423b4783610516017dee3b fix nonzero bug is datatype error
- 根因类别：非标准数据类型别名(平台不可移植)
- 涉及文件：index/non_zero/op_kernel/arch35/non_zero_big_mask.h
- 缺陷描述：使用uint64类型(非标准别名)而非标准的uint64_t，在某些编译环境下未定义导致编译错误或行为异常
- 修复模式：static_cast<uint64>改为static_cast<uint64_t>
- 可审查性：高
- 审查规则建议：使用标准C++类型(uint64_t/int64_t)，不使用非标准别名(uint64/int64)

### 6ec2c7aec4f6ceef082f05c4b468093cd36fb5c2 fix max pool argmax v3 nhwc kernel
- 根因类别：复制粘贴错误(stride变量) + UpdateMask参数被循环修改
- 涉及文件：pooling/max_pool_with_argmax_v3/op_kernel/arch35/max_pool_with_argmax_v3_nhwc_small_c.h, max_pool_with_argmax_v3_nhwc_big_c.h
- 缺陷描述：(1)small_c中wStride = hStride_复制粘贴错误，应为wStride_，导致W方向步长取值错误影响第二个输出精度；(2)big_c中FillPadNegInf的topCount/downCount在UpdateMask调用中被修改(UpdateMask会递减count)，循环第二次迭代起使用错误的count值
- 修复模式：(1)改为this->wStride_；(2)引入topCountTmp/downCountTmp临时变量避免原始参数被UpdateMask修改
- 可审查性：高(1) / 中(2)
- 审查规则建议：检查结构体成员赋值中h*/w*是否成对正确引用；UpdateMask等会修改输入参数的API在循环中使用时必须用临时变量保护原始值

### 2a15cd4d 解决index整网用例aic问题
- 根因类别：全局变量误用（作用域错误）
- 涉及文件：index/index/op_host/arch35/index_tiling_no_continuous.cpp, index_tiling_no_continuous.h
- 缺陷描述：indexstrideList被定义为.cpp文件作用域的全局std::vector，多实例并发调用时共享导致AIC错误。修复将其移入IndexNonContinuousTiling类的private成员，每个tiling实例拥有独立副本
- 修复模式：全局变量 -> 类成员变量（作用域收窄）
- 可审查性：中
- 审查规则建议：禁止在op_host/*.cpp中使用非const全局容器声明，tiling相关状态必须封装在类成员中

### e1a2f11f group_norm_grad算子在超长R轴下内存访问越界
- 根因类别：整数溢出（uint32溢出导致UB分配不足）
- 涉及文件：norm/group_norm_grad/op_host/group_norm_grad_tiling_arch35.cpp
- 缺陷描述：(1)mode0xDyDxSize为uint32_t，CPerG_*HxW_超长时CeilAlign()*tTypeBytes_*UB_COPIES_3乘积溢出，内存分配不足导致越界；(2)mode1UbCapCNum_分母计算中CeilAlign()返回值乘tTypeBytes_*UB_COPIES_3也溢出。修复改为int64_t和显式(int64_t)cast
- 修复模式：类型提升（uint32_t -> int64_t）
- 可审查性：高
- 审查规则建议：tiling代码中shape维度相乘的表达式强制使用int64_t；CeilAlign/CeilDiv返回值参与二次乘法时检查结果类型

### e0ddd962 fix nonzero operator B32/B16/B8 output shape product out of int32 boundary
- 根因类别：整数溢出（int32乘法溢出）
- 涉及文件：index/non_zero/op_kernel/arch35/non_zero_big_mask.h
- 缺陷描述：gmOffset_计算中addUbSize.GetValue(8)*shapeDim_均为int32，B32/B16/B8模式下output shape乘积超int32边界。修复对两个操作数做static_cast<uint64>
- 修复模式：类型提升（显式static_cast<uint64>）
- 可审查性：高
- 审查规则建议：GetValue()返回值参与乘法且结果赋给uint64/int64时，要求操作数显式cast为64位

### 8192a77f fix max pool v3 small kernel tiling
- 根因类别：UB空间计算遗漏
- 涉及文件：pooling/max_pool_v3/op_host/arch35/max_pool_v3_small_kernel_tiling.cpp
- 缺陷描述：计算availableUb_时只减了UB_RESVERVED_SIZE，遗漏了紧邻上方刚计算的indiceUbSize_，导致可用UB被高估，数据分块过大溢出indices区域
- 修复模式：补全资源预算减项（availableUb_ = ubSize - UB_RESVERVED_SIZE - indiceUbSize_）
- 可审查性：中
- 审查规则建议：计算availableUb/可用空间时，检查同作用域内所有*UbSize_变量是否纳入减项

### 502f8ac9 Revert "DTS2025120868273 c04 with innerbatch"
- 根因类别：特性引入缺陷（tiling + kernel全面错误）
- 涉及文件：conv/common/op_host/..., conv/common/op_kernel/..., conv/conv2d_v2/... (9个文件)
- 缺陷描述：C04(small channel)+innerbatch特性整体回退。原始提交三类问题：(1)L1空间计算公式错误(align与innerBatch乘法顺序不对)导致buffer越界；(2)kernel mmad指令参数(m/srcStride/srcBatchStride)计算公式错误；(3)265行tilingkey宏定义与错误kernel逻辑配套。全量Revert
- 修复模式：全量Revert（特性设计有根本性问题）
- 可审查性：低
- 审查规则建议：同时修改tiling(host)和kernel(device)超过5个文件的大特性，应拆分子PR并附带数值验证用例

### 4432d437 Fix M-MTE1 SetWait when kernel contains TPipe
- 根因类别：硬件同步flag编号冲突
- 涉及文件：matmul/common/cmct/block/block_mmad_pingpong_without_que.h
- 缺陷描述：M_MTE1事件的SetFlag/WaitFlag使用ZERO_FLAG(0)/FIRST_FLAG(1)，与TPipe内部使用的低编号flag冲突，导致同步信号互相践踏。修复将flag偏移到SIXTH_FLAG(6)/SEVENTH_FLAG(7)，三处循环体都修复
- 修复模式：flag编号偏移到高位区间，避开TPipe占用
- 可审查性：中
- 审查规则建议：硬件同步flag应有全局编号分配策略，禁止不同模块隐式假设flag不冲突；SetFlag/WaitFlag变更需注释flag分配方案

### ffed8d28 bugfix swi_glu_quant && inplace_add_rms_norm
- 根因类别：预处理条件宏逻辑错误
- 涉及文件：norm/inplace_add_rms_norm/op_kernel/inplace_add_rms_norm.cpp, quant/swi_glu_quant/op_kernel/swi_glu_quant.cpp
- 缺陷描述：(1)inplace_add_rms_norm中4处#if条件取反遗漏：`__NPU_ARCH__==3003`应为`!=3003`，arch3003不支持bfloat16_t却编译了该路径；(2)swi_glu_quant中#if/#endif嵌套层级错误，外层#endif过早关闭导致#elif变成孤立分支。修复取反条件+合并为单层#if
- 修复模式：条件宏取反修正 + 消除错误嵌套
- 可审查性：高
- 审查规则建议：平台架构排除宏应封装为统一宏(如ARCH_SUPPORTS_BF16)避免手写取反遗漏；#if嵌套超2层须在#endif注释对应条件

### e499fdc8 fix groupnormgrad warning
- 根因类别：类型安全缺陷（有符号/无符号比较 + 整数截断 + 返回值未检查）
- 涉及文件：norm/group_norm_grad/op_host/group_norm_grad_empty_tiling_arch35.cpp
- 缺陷描述：(1)ubSize_(uint64_t)强转int64_t做比较，超INT64_MAX时变负导致逻辑反转；(2)maxRowsNumDG_声明为int，存uint64_t除法结果可能截断；(3)CalcuTilingData()返回值未检查。修复去掉多余cast、改int64_t、合并声明赋值并加错误日志
- 修复模式：类型修正 + 返回值检查
- 可审查性：高
- 审查规则建议：buffer size计算统一用uint64_t/int64_t禁止截断到int；graphStatus返回值必须检查

### d818b4d4 修复告警
- 根因类别：编译告警修复（含潜在逻辑bug）
- 涉及文件：index/masked_scatter/op_api/aclnn_masked_scatter.cpp, masked_scatter_tiling_arch35.cpp, quant_update_scatter_tiling_arch35.cpp, smooth_l1_loss_grad_v2_tiling.cpp
- 缺陷描述：5处告警修复。值得注意：aclnn_masked_scatter.cpp删除未使用的sourceStorageFormat变量，但原始代码条件判断中重复检查maskStorageFormat两次而非source+mask各一次，极可能是copy-paste逻辑bug（此commit仅消除变量告警未修复逻辑）
- 修复模式：删除未使用变量、void cast、%d->%u格式修正
- 可审查性：高
- 审查规则建议：删除"未使用变量"时须检查该变量是否原本应该被使用（可能掩盖copy-paste逻辑bug）

### 685d55f9 fix a bug is the int64 data type of nonzero
- 根因类别：复制粘贴参数错误
- 涉及文件：index/non_zero/op_kernel/arch35/non_zero_base.h, non_zero_full_load_base.h
- 缺陷描述：CopyInBefore中int8/int16/int32三个分支调用VfPerCoreNonZeroNum使用(loopCore,beforeNum,...)，但int64分支错误使用(loopNum,tailNum,...)，语义完全不同（前序core总数 vs 当前core循环数），导致int64下计算结果错误
- 修复模式：参数纠正，与其他分支保持一致
- 可审查性：高
- 审查规则建议：同函数内多个if/else分支调用同签名函数时，检查各分支实参语义一致性

### 4453d4c8 Fix MaxPool3DGrad
- 根因类别：数学公式错误（池化输出维度计算）
- 涉及文件：pooling/max_pool3d_grad_with_argmax/op_host/max_pool3d_grad_with_argmax_tiling_base.cpp
- 缺陷描述：ceil mode池化输出维度公式错误，分子少了+stride且CeilDiv外多了+1，等价于floor((x-1)/s)+2而非正确的ceil(x/s)。当分子为负时CeilDiv行为异常导致整数溢出。修复为标准公式(input+2*pad+stride-dilation*(kernel-1)-1)/stride
- 修复模式：公式纠正（标准池化输出维度公式）
- 可审查性：低（bug fix混在36文件3000+行重构中）
- 审查规则建议：池化输出维度计算应作为公共工具函数不应多处手写；大PR应拆分重构和bug fix

### 368607bd CTCLossV2Grad算子搬迁修正
- 根因类别：搬迁遗漏（类型不匹配 + 空函数体）
- 涉及文件：loss/ctc_loss_v2_grad/op_host/arch35/ctc_loss_v2_grad_tiling_arch35.cpp, .h
- 缺陷描述：(1)TilingParse注册旧类型CTCLossV2GradCompileInfo(字段coreNum)，实际tiling依赖新类型CTCLossV2GradForCompileInfo(字段totalCoreNum)，类型不匹配导致参数取值错误；(2)TilingPrepare函数体为空直接return SUCCESS，缺失获取平台core数的初始化逻辑，totalCoreNum始终为0
- 修复模式：类型统一 + 补全缺失逻辑
- 可审查性：高
- 审查规则建议：TilingParse<T>的模板参数T必须与TilingPrepare操作的CompileInfo类型一致；TilingPrepare为空函数体应标记告警

### 121c1683 convtbc aicore问题修复
- 根因类别：tiling模式选择缺陷
- 涉及文件：conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_v2_base_tiling_fast_tiling.cpp
- 缺陷描述：GetC04TilingSplitMode()中conv1d(ConvTBC)场景下wo>128时不应进入M-split模式应走HW-split，旧代码缺少此条件判断导致大wo下选错tiling模式产生计算错误
- 修复模式：增加前置条件检查(IS_CONV1D_FLAG && wo>128时forceHWSplitModeFlag=true)
- 可审查性：中
- 审查规则建议：conv各变体(conv1d/conv2d/convtbc)共用tiling逻辑时，须标注各变体适用范围

### f0cf6d56 fix tiling error
- 根因类别：错误的校验逻辑（过早拒绝合法输入）
- 涉及文件：quant/dequant_swiglu_quant/op_host/dequant_swiglu_quant_tiling_base.cpp
- 缺陷描述：DoOpTiling()中isPerformanceBranch()判断后调用CheckKernelUBUpBound()，该检查将合法tiling场景也拒绝，导致本应成功的算子返回GRAPH_FAILED。修复删除错误的检查调用
- 修复模式：移除错误校验
- 可审查性：高
- 审查规则建议：新增return GRAPH_FAILED校验点须附UT说明触发条件和原因

### a6a1aff4 flatquant编译问题优化
- 根因类别：运算符优先级不明确（编译告警）
- 涉及文件：quant/flat_quant/op_kernel/flat_quant_vec_one.h
- 缺陷描述：Nceil>>LOG2_16-1和Nceil>>LOG2_128-1中>>与-优先级关系不明确，虽然C++中-优先级高于>>所以运行时行为正确，但代码意图不明确触发-Wparentheses告警。修复添加显式括号
- 修复模式：添加括号明确运算优先级
- 可审查性：高
- 审查规则建议：移位与算术运算符混用须加括号；启用-Werror=parentheses

### 282382f9 fix ini
- 根因类别：CMake构建依赖缺失
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：add_custom_command生成ops-info.ini时缺少DEPENDS opbuild_custom_gen_aclnn_all，导致并行构建时aclnn生成目标可能尚未完成就开始合并ini文件，产生构建竞态
- 修复模式：在两处add_custom_command中添加DEPENDS opbuild_custom_gen_aclnn_all
- 可审查性：中
- 审查规则建议：add_custom_command必须声明所有输入依赖，特别是跨模块的生成目标

### d44f47c2 fix conv warning
- 根因类别：编译告警（参数名遮蔽成员变量 + 未使用变量）
- 涉及文件：conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_v2_api_tiling.cpp, .h, conv3d_v2/.../conv3d_v2_base_tiling_check_attrs.cpp
- 缺陷描述：(1)SetQuantScale(bool hasScale)参数名与成员变量this->hasScale同名，-Wshadow告警；(2)conv3d ParseGroupLegal()中fMapDesc获取后未使用
- 修复模式：参数重命名为hasScaleFlag；删除未使用变量
- 可审查性：高
- 审查规则建议：函数参数名不得与类成员变量同名；启用-Wshadow -Werror

### 7a917555 修复A5 qbmmv4相关warning
- 根因类别：编译告警（未使用参数/变量）
- 涉及文件：matmul/quant_batch_matmul_v3/.../quant_batch_matmul_v3_checker_base.h, quant_batch_matmul_v4/...多文件
- 缺陷描述：基类虚函数CheckShape的参数未标注/* unused */触发-Wunused-parameter；qbmmv4中offsetShape获取后未使用；CalL1TilingDepth4MmadS8S4的leftL1Size参数未使用
- 修复模式：参数加/* name */注释标注；删除未使用变量
- 可审查性：高
- 审查规则建议：虚函数基类中无操作的参数须用/* name */标注；启用-Wunused-parameter -Werror

### 73e48d80 修复matmulv3在特定shape下的精度问题
- 根因类别：边界条件遗漏（尾块处理不完整）
- 涉及文件：matmul/mat_mul_v3/op_kernel/mat_mul_deterministic_splitk_kernel.h
- 缺陷描述：ReduceKInUbNzL2cache中双循环遍历M和N分块时，(1)mCoreUse/nCoreUse未在每次循环开始时重置为singleCoreM/singleCoreN，导致前次循环的尾块值被后续使用；(2)actualN尾块条件仅检查orderNMFlag==true时的outIndex尾块，遗漏orderNMFlag==false时inIndex尾块对应N轴的情况
- 修复模式：循环体内重置mCoreUse=tiling.singleCoreM/nCoreUse=tiling.singleCoreN；扩展条件为(orderNMFlag && outIndex==outCnt-1) || (!orderNMFlag && inIndex==inCnt-1)
- 可审查性：高
- 审查规则建议：双循环中涉及尾块的变量必须在每次迭代开始时重置；遍历顺序flag改变时须同步更新所有依赖该flag的条件分支

### 6287806e fix compilation alarms
- 根因类别：编译告警（隐式类型转换 + 浮点比较不精确）
- 涉及文件：conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_v2_api_tiling.cpp, conv2d_v2_base_tiling_basic_block.cpp
- 缺陷描述：(1)多处scaleSize计算中channelWiseCoeff(float)*整数乘积赋值给int64_t/uint64_t时隐式截断；(2)calCut1/calCut2为float但后续用作uint32_t循环变量，缺显式转换；(3)BasicBlockSortFWDimScores中浮点数直接用==比较，应改为abs(a-b)<epsilon；(4)minCost从float赋值给uint32_t隐式截断；(5)sqrt(cores)返回double赋值给float
- 修复模式：添加static_cast显式转换；引入epsilon做浮点近似比较
- 可审查性：高
- 审查规则建议：浮点数不得用==比较，须用epsilon；混合类型运算须显式cast；启用-Wconversion

### 6246f56c fix ut
- 根因类别：UT测试环境缺陷（缺少平台版本设置）
- 涉及文件：pooling/avg_pool3_d_grad/tests/ut/op_api/test_aclnn_avgpool2d_backward.cpp
- 缺陷描述：测试用例ascend310P_test_avgpool2dbackwardbackward_global_avg_pool_bf16中缺少SocVersionManager设置为ASCEND310P，导致平台信息为默认值，测试结果不可靠
- 修复模式：添加op::SocVersionManager versionManager(op::SocVersion::ASCEND310P)
- 可审查性：中
- 审查规则建议：指定特定平台的UT用例须在开头设置SocVersionManager

### 35f22362 SparseTensorDenseMatMul修复编译告警
- 根因类别：编译告警（printf格式符不匹配）
- 涉及文件：matmul/sparse_tensor_dense_mat_mul/op_host/op_tiling/sparse_tensor_dense_mat_mul_tiling_arch35.cpp
- 缺陷描述：OP_LOGE中对int64_t变量使用%lld格式符，在某些平台上int64_t是long而非long long，应使用%ld。约12处告警
- 修复模式：全部%lld改为%ld
- 可审查性：高
- 审查规则建议：int64_t使用PRId64宏或%ld，不要硬编码%lld；启用-Wformat

### 1e38bfac LayerNormV3 tiling duplicate registrations bug fixed
- 根因类别：tiling重复注册
- 涉及文件：norm/layer_norm_v4/op_host/layer_norm_v4_regbase_two_pass_perf_tiling.cpp, layer_norm_v4_regbase_two_pass_tiling.cpp, layer_norm_v4_tiling.h, layer_norm_v4_tiling_base.cpp
- 缺陷描述：LayerNormV4的tiling文件中同时注册了LayerNormV3和LayerNormV4的REGISTER_OPS_TILING_TEMPLATE/REGISTER_TILING_DATA_CLASS，以及V3专用的CompileInfo结构体和GetV3PlatformInfo函数。V3已有独立tiling实现，V4文件中的V3注册是冗余的，导致重复注册冲突
- 修复模式：删除V4文件中所有V3相关的注册、结构体定义和函数
- 可审查性：中
- 审查规则建议：每个算子版本的tiling注册应在各自文件中完成，不得在其他版本文件中交叉注册

### 1a267243 fix: l1 size constant inconsistent
- 根因类别：常量运算语义错误（乘vs除）
- 涉及文件：matmul/common/cmct/prologue/block_prologue_b_antiquant_scmc_nd_kn.h, block_prologue_b_antiquant_scmc_nd_nk_nz_kn.h
- 缺陷描述：weightF16L1DbOffset_计算中使用L1_SIZE * KB_ELEM<half>，但KB_ELEM<half>的语义是"每个half元素占的字节比例"（即sizeof(half)），这里应该是L1_SIZE / sizeof(half)得到half元素数量再减去weight空间。L1_SIZE*KB_ELEM结果远大于实际L1大小，导致UB_TO_L1目的地址越界AIC_ERROR
- 修复模式：L1_SIZE * KB_ELEM<half> → L1_SIZE / sizeof(half)
- 可审查性：高
- 审查规则建议：地址偏移计算中乘除运算须明确单位语义（字节vs元素数）；L1/UB/GM地址偏移须做边界检查

### f02308a2 qbmmv3/sparse4to2 sk sync bugfix
- 根因类别：AIC/AIV同步阈值错误（流水线事件Wait条件）
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3_bf16_basic.h, quant_batch_matmul_v3_pertoken_basic.h, matmul/sparse4to2quant_matmul/op_kernel/sparse4to2quant_matmul.h
- 缺陷描述：SplitK结束时AIC补齐之前跳过的WaitEvent，原始代码条件为loop_>2补ping、loop_>3补pong。但实际skipWait只跳过了第0次和第1次（ping/pong各一次），所以补齐条件应为loop_>0和loop_>1。错误阈值导致loop_<=2或loop_<=3时缺少WaitEvent，C2V数据可能尚未就绪就开始使用
- 修复模式：loop_>2→loop_>0，loop_>3→loop_>1，三处文件同步修复
- 可审查性：中
- 审查规则建议：ping-pong双buffer的WaitEvent/SetEvent必须成对出现，跳过的Wait在结束时必须补齐且阈值与跳过次数一致

### d0790662 fix dynamic block quant kernel bug
- 根因类别：复制粘贴错误（行/列变量名混用）
- 涉及文件：quant/dynamic_block_quant/op_kernel/arch35/dynamic_block_quant_b8_kernel.h, dynamic_block_quant_bf16_b8_kernel.h
- 缺陷描述：CopyOut中计算scaleGmOffset时使用rowIdx * rowBlockLoopNum_，但该偏移是沿列方向递增的，应为rowIdx * colBlockLoopNum_。错误导致scale数据写入GM的位置偏移不正确
- 修复模式：rowBlockLoopNum_ → colBlockLoopNum_，两个文件同步修复
- 可审查性：高
- 审查规则建议：行/列相关变量命名须清晰区分；GM偏移计算中的stride须与数据布局一致

### cd7e67d6 添加资料及修复代码告警
- 根因类别：编译告警（参数名遮蔽成员变量）
- 涉及文件：activation/sigmoid_grad/op_host/arch35/sigmoid_grad_tiling_arch35.cpp, activation/silu_grad/op_host/arch35/silu_grad_tiling.cpp
- 缺陷描述：GetComputeMap(uint64_t opKey)和ToString(TilingData &tilingData)中参数名与成员变量/类型名冲突，触发-Wshadow告警
- 修复模式：参数重命名为opKeyParam、tilingDataParam
- 可审查性：高
- 审查规则建议：函数参数名不得与成员变量或类型名相同

### b47cf616 LayerNormV4: Fix_lastBinaryBlock
- 根因类别：向量运算mask作用域错误 + ReduceSum精度缺陷
- 涉及文件：norm/layer_norm_v4/op_kernel/arch35/layer_norm_v4_two_pass_perf.h
- 缺陷描述：LAST_LOOP_NUMS==2分支中：(1)pregLast在if constexpr分支外定义但仅在特定分支使用，LAST_LOOP_NUMS==1时pregLast基于lastBinaryAddNum，而LAST_LOOP_NUMS==2时实际需要的mask是lastBinaryAddNum-VL_B32（仅覆盖第二块的有效元素）；(2)直接将两块Add后用pregLast做ReduceSum，但pregLast对应的是整体lastBinaryAddNum，第二块的超出部分包含了脏数据。修复改为：先用ShiftLefts将x2中有效元素左移对齐，再与x1做Add，最后用pregFull做ReduceSum
- 修复模式：pregLast移入分支内部重新定义为lastTailNum；引入ShiftLefts对齐第二块数据；ReduceSum改用pregFull
- 可审查性：低
- 审查规则建议：ReduceSum的mask必须精确覆盖有效数据范围；多块归约时须处理尾块对齐问题

### b139245e fix rms_norm infershape
- 根因类别：InferShape边界条件缺失（unknown rank未处理 + 输出shape未初始化 + 维度校验缺失）
- 涉及文件：norm/add_rms_norm/op_host/add_rms_norm_infershape.cpp, norm/rms_norm/op_host/rms_norm_infershape.cpp
- 缺陷描述：(1)未处理unknown rank(-2)输入，直接对GetDimNum()结果做循环，unknown rank时DimNum返回异常值；(2)rstdShape输出未提前获取和null检查；(3)缺少xDimNum < gammaDimNum的合法性校验，可能导致循环下标溢出
- 修复模式：添加IsUnknownRank()提前返回；提前GetOutputShape(1)并做null检查；添加维度数比较校验
- 可审查性：高
- 审查规则建议：InferShape必须处理unknown rank/unknown dim场景；所有输出shape须提前获取和null检查

### a0fd9959 fix_elu_grad_v2_tiling
- 根因类别：搬迁遗漏（CompileInfo类型不匹配 + 缺少tiling初始化）
- 涉及文件：activation/elu_grad/op_host/arch35/elu_grad_tiling_arch35.cpp/.h, activation/elu_grad_v2/op_host/arch35/elu_grad_v2_tiling_arch35.cpp/.h
- 缺陷描述：(1)EluGrad和EluGradV2使用通用ElewiseCompileInfo类型，但该类型字段与实际需要的coreNum/ubSize不匹配，需改用专用的EluGradCompileInfo/EluGradV2CompileInfo；(2)EluGradV2Tiling::RunTiling中缺少GetTilingData<EluGradV2TilingData>()初始化，tiling指针为null导致后续操作崩溃
- 修复模式：新建专用CompileInfo结构体；TilingParse/Tiling/TilingPrepare全部替换为专用类型；RunTiling开头添加tiling初始化
- 可审查性：高
- 审查规则建议：算子tiling的CompileInfo类型须与IMPL_OP_OPTILING注册的TilingParse<T>模板参数一致；RunTiling入口必须初始化tiling数据指针

### 99a2052c 修复部分算子编译过程报错的问题
- 根因类别：构建脚本glob匹配过宽
- 涉及文件：scripts/util/ascendc_impl_build.py
- 缺陷描述：get_ops_info_files()使用glob(aic-*-ops-info.ini)匹配所有SoC的ini文件，在多SoC构建环境中加载了不相关SoC的配置导致编译报错。修复改为按--soc参数按指定SoC精确匹配
- 修复模式：增加soc参数，按aic-{soc}*-ops-info.ini精确glob
- 可审查性：中
- 审查规则建议：构建脚本中的glob匹配应尽可能精确，避免通配符过宽匹配到不相关文件

### 9572f313 精度fail场景修改
- 根因类别：量化matmul多处计算逻辑错误（对齐/条件/偏移/模板分支）
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_api/aclnn_weight_quant_batch_matmul_v2.cpp, op_tiling/arch35/weight_quant_batch_matmul_v2_reg_base_tiling.cpp, op_tiling/weight_quant_batch_matmul_v2_tiling.cpp, op_kernel/arch35/weight_quant_batch_matmul_v2_reg_base_common.h
- 缺陷描述：(1)perGroup场景N轴对齐检查在910_95平台不应生效，缺少平台排除条件；(2)kBubSize双buffer切分未考虑groupSize对齐，直接CeilDiv(kBl1Size,2)导致切分点不在group边界上，反量化参数错位；(3)4bit权重NZ格式的DMA参数bubKLen未CeilAlign到BLOCK_CUBE导致搬运不完整；(4)weightOutStride公式错误dataBlockStride*(vfElemB16-BLOCK_CUBE)+BLOCK_CUBE应为dataBlockStride*vfElemB16-bubKLen*BLOCK_CUBE；(5)pergroup32/64的VF_CALL中innerExtend==1||outerExtend==1时使用false模板参数分支，实际该条件判断有误应统一走true分支；(6)整体删除了K/N维度32B对齐的过严校验
- 修复模式：多点修复，涉及计算公式、平台判断、DMA对齐、模板分支简化
- 可审查性：低
- 审查规则建议：量化matmul的buffer切分边界须与groupSize对齐；DMA参数中的blockLen须与实际数据类型对齐粒度一致

### 5c65b17d LayerNormV3: Fix_lastBinaryBlock
- 根因类别：向量运算mask作用域错误 + ReduceSum精度缺陷（同b47cf616的V3版本）
- 涉及文件：norm/layer_norm_v3/op_kernel/arch35/layer_norm_v3_two_pass_perf.h
- 缺陷描述：与b47cf616完全相同的缺陷在LayerNormV3中的对应实现。LAST_LOOP_NUMS==2分支中pregLast作用域和ReduceSum的mask不正确，导致尾块归约包含脏数据
- 修复模式：与b47cf616相同的修复手法
- 可审查性：低
- 审查规则建议：同一算法的不同版本(V3/V4)须同步修复同类缺陷

### 44162b1b fix ksize one tiling
- 根因类别：API命名空间迁移不完整
- 涉及文件：pooling/max_pool_grad_with_argmax_v3/op_host/arch35/max_pool_grad_with_argmax_v3_ksize_one_tiling.cpp
- 缺陷描述：KsizeOne分支tiling代码使用了旧命名空间API(`platform::GetVRegSize`/`ops::CeilDiv`)，导致编译失败或运行时行为不正确。同时tiling key从301改为900，说明分支路由也有错误
- 修复模式：`platform::GetVRegSize` → `Ops::Base::GetVRegSize`，`ops::CeilDiv` → `Ops::Base::CeilDiv` 等8处命名空间替换；UT中tiling key 301→900
- 可审查性：高
- 审查规则建议：API命名空间迁移时需全局搜索旧命名空间，确保所有分支（包括不常触发的ksize=1特殊路径）都已迁移

### 47e5dfc0 scatter_add、scatter_nd_add、scatter_elements、smooth_l1_loss_grad_v2修复问题
- 根因类别：算子注册架构搬迁不完整(graph_infer → infershape合并)
- 涉及文件：index/scatter_add/op_host/scatter_add_infershape.cpp, index/scatter_elements/CMakeLists.txt, loss/smooth_l1_loss_grad_v2/op_host/smooth_l1_loss_grad_v2_infershape.cpp 等13个文件
- 缺陷描述：InferDataType函数原先在独立的graph_infer.cpp中注册，需迁移到infershape.cpp中与InferShape合并注册(`.InferShape().InferDataType()` 链式调用)。scatter_elements的CMake还缺少对tf_scatter_add/scatter_nd_add的依赖声明
- 修复模式：删除独立的graph_infer.cpp，将InferDataType函数移入infershape.cpp，使用链式注册；补充CMake DEPENDENCIES
- 可审查性：中
- 审查规则建议：算子infershape/infertype注册架构变更时，需确认所有算子都已完成迁移，InferDataType不应存在于独立的graph_infer文件中

### 16275685 去掉chamferdistancegrad的pipeall同步
- 根因类别：流水线同步过粗(PipeBarrier<PIPE_ALL>滥用)
- 涉及文件：loss/chamfer_distance_grad/op_kernel/chamfer_distance_grad.h
- 缺陷描述：kernel中约20处使用`PipeBarrier<PIPE_ALL>`作为万能同步，这既是正确性风险（过度序列化可能掩盖实际数据依赖关系），也导致严重性能问题。正确的方法是根据实际的数据流依赖使用精确的HardEvent同步
- 修复模式：新增`PipeSync<HardEvent>`模板方法封装SetFlag/WaitFlag，用精确的事件类型替换所有PipeBarrier<PIPE_ALL>：V_MTE3(向量写完后DMA出)、MTE2_MTE3(搬入搬出依赖)、MTE2_V(搬入后计算)、V_S(计算后标量)、S_V(标量后计算)、MTE3_S(搬出后标量)等
- 可审查性：高
- 审查规则建议：PipeBarrier<PIPE_ALL>在kernel中应被视为代码异味，审查时应要求开发者证明无法用更精确的HardEvent替代

### f5606d69 修复空tensor提示bug
- 根因类别：CMake依赖声明缺失
- 涉及文件：matmul/mat_mul_v3/op_host/CMakeLists.txt
- 缺陷描述：mat_mul_v3缺少对batch_mat_mul_v3的CMake DEPENDENCIES声明，导致空tensor场景下符号解析失败
- 修复模式：CMakeLists.txt添加`DEPENDENCIES batch_mat_mul_v3`
- 可审查性：高
- 审查规则建议：算子模块间存在符号引用时必须在CMakeLists.txt中声明DEPENDENCIES

### 81d3a62e 修复qbmmv4 mxa8w4 ND精度问题
- 根因类别：多buffer策略下循环控制遗漏
- 涉及文件：common/act/prologue/block_prologue_b_cast_scsc.h
- 缺陷描述：ND场景下2-buffer模式时nL1Len可能不等于nUbSize，但VectorProcess直接调用ProcessL1，内部仅处理4-buffer的单次搬运。缺少对nL1Len>nUbSize时的多轮循环控制，导致weight数据只处理了第一个UB大小的部分，引起精度错误
- 修复模式：将原ProcessL1拆分为ProcessL1NK4Buffer(4-buffer快速路径)和ProcessL1NK(通用循环路径，含bUbNFactor×bUbKFactor双层循环)；VectorProcess根据l1BufNum_分发；offset变量从Aiv1专用(nL1Aiv1Offset_)重命名为通用(nL1Offset_)
- 可审查性：中
- 审查规则建议：多buffer模式切换时(2buffer/4buffer)，必须验证L1→UB搬运是否需要多轮循环；当L1数据量>UB容量时，必须有循环控制

### 4cd640c2 DynamicBlockQuant代码告警修复
- 根因类别：参数名遮蔽成员变量(-Wshadow)
- 涉及文件：quant/dynamic_block_quant/op_host/dynamic_block_quant_i8_tiling.cpp, .h
- 缺陷描述：RowTilingData构造函数参数`coreNum`与成员变量`coreNum`同名，构造函数内`this->coreNum = coreNum`虽然功能正确但触发-Wshadow告警。类似地DynamicBlockQuantI8构造函数参数`context`与成员`context`同名
- 修复模式：参数重命名：`coreNum` → `curCoreNum`，`context` → `ctx`
- 可审查性：高
- 审查规则建议：构造函数参数不应与成员变量同名，启用-Wshadow编译选项

### 21797e26 fix conv forward aclnn cmake
- 根因类别：CMake依赖声明缺失
- 涉及文件：conv/convolution_backward/CMakeLists.txt, conv/convolution_forward/op_host/CMakeLists.txt
- 缺陷描述：convolution_backward缺少对avg_pool3_d_grad/batch_mat_mul_v3/convolution_forward的依赖；conv相关源文件缺少include
- 修复模式：补充CMake DEPENDENCIES声明；添加缺失的include
- 可审查性：高
- 审查规则建议：新增算子模块间调用时必须同步更新CMake DEPENDENCIES

### 0c96cd51 修复ScatterNdAdd符号未定义的问题
- 根因类别：API迁移不完整 + 文件命名冲突
- 涉及文件：index/scatter_nd_add/op_host/arch35/scatter_nd_add_tiling.cpp, scatter_util→scatter_tiling_util重命名
- 缺陷描述：scatter_util.h/cpp文件名与其他模块冲突导致符号未定义，同时内部使用旧API `GetCompileInfoPtr<>` 需迁移为 `context->GetCompiledInfo<>()`，TilingPrepare4Scatter中大量TIK旧路径代码已废弃需清理
- 修复模式：文件重命名scatter_util→scatter_tiling_util；API替换GetCompileInfoPtr→GetCompiledInfo；移除TIK旧代码路径
- 可审查性：中
- 审查规则建议：工具函数文件命名应包含算子名前缀避免跨模块冲突；API迁移时同步清理废弃代码路径

### 0ac42f18 修复非连续
- 根因类别：条件判断遗漏
- 涉及文件：matmul/common/op_host/op_api/batch_matmul_util.cpp
- 缺陷描述：CheckTransNonContiguousShapeSupport函数检查L0a/L0b/L0c/L1容量约束时，计算了lessThanL1变量但在early return条件中遗漏了对它的检查。导致L1放不下的场景仍然走了非连续转置优化路径，产生错误结果
- 修复模式：在`if (!batchEqual || !batchLargerThanAicNum)`条件中追加`||!lessThanL1`
- 可审查性：高
- 审查规则建议：当函数计算了多个条件变量(lessThanL0a/L0b/L0c/L1)后，审查需确认每个变量都被后续逻辑使用；未使用的计算结果是强烈的遗漏信号

### 940066dd fix EnsureNotScalar
- 根因类别：头文件依赖断裂
- 涉及文件：activation/elu_grad/op_host/arch35/elu_grad_tiling_arch35.cpp
- 缺陷描述：elu_grad依赖tiling_base/tiling_util.h中的EnsureNotScalar函数和Ops::NN::OpTiling命名空间，但头文件路径或符号已不可用，导致编译失败
- 修复模式：移除对tiling_util.h的include和using namespace，在本地文件中重新实现EnsureNotScalar(标量→shape{1}转换)
- 可审查性：中
- 审查规则建议：公共工具函数被删除或移动时，需检查所有使用方是否已更新

### 503a451c fix aclnn bng bug
- 根因类别：空tensor的dtype未设置导致Cast失败
- 涉及文件：norm/batch_norm_grad_v3/op_host/op_api/aclnn_batch_norm_backward.cpp
- 缺陷描述：当saveMean/saveInvstd为空tensor时，Contiguous后tensor的dtype可能未正确设置，直接调用`l0op::Cast(tensor, DT_FLOAT)`失败。空tensor虽然没有实际数据，但Cast操作仍需要合法的源dtype
- 修复模式：在Cast前检查IsEmpty()，若为空tensor则用const_cast显式设置DT_FLOAT dtype再Cast
- 可审查性：中
- 审查规则建议：对tensor执行Cast/类型转换前，需考虑空tensor的dtype是否已正确初始化；空tensor不等于可以跳过类型检查

### 0e5dadce 修复告警信息
- 根因类别：隐式bool转换告警(unsigned int → bool)
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/quant_batch_matmul_v3_iterbatch_tiling.cpp
- 缺陷描述：`!aicoreParams_.aicNum`对uint64使用逻辑非运算符，虽然语义正确(检查是否为0)但触发编译告警(implicit conversion from uint64 to bool)
- 修复模式：`!aicoreParams_.aicNum` → `aicoreParams_.aicNum == 0`，同理处理l1Size、l0cSize
- 可审查性：高
- 审查规则建议：对非bool类型变量禁止使用`!`运算符做零值检查，使用显式`== 0`比较

### dcb8f328 dynamic_quant 910c问题修复
- 根因类别：平台case遗漏
- 涉及文件：quant/dynamic_quant/op_host/dynamic_quant_tiling.cpp, CMakeLists.txt
- 缺陷描述：TilingForDynamicQuant的switch语句中缺少`case ASCEND910_93`(910C平台)，导致910C平台无法走入正确的tiling分支。同时CMake缺少对dynamic_quant_v2的依赖
- 修复模式：在ASCEND910B case前添加`case platform_ascendc::SocVersion::ASCEND910_93:`使其fall-through；补充CMake DEPENDENCIES
- 可审查性：高
- 审查规则建议：平台switch语句新增SoC版本时，须全局搜索所有SocVersion switch确认无遗漏；每个switch应有default分支处理未知平台

跳过(第160-179行)：7条
- 062c7b64: 回退cmake修改(构建配置revert)
- f72dda48: fix tiling ut noexec(UT基础设施/cmake)
- 9119a756: ExtendConv2d support leaky relu(新功能)
- 8c22fef1: 支持leaky_relu/swish算子A5实现(新功能)
- 71cd018e: fix link(纯文档链接修复)
- 50e3f514: 静态库脚本修复(Python 3.7兼容性)
- d92c383f: docs_aclnnBMM_fix(纯文档)

### cfe148f860bcacb29ca21ed256fa49cce56ab93d 修复C04的性能问题&更新flagInfo
- 根因类别：状态同步缺失(组合对象状态未传递)
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_base.cpp, conv_base.h, conv_base_utils.h, conv2d_v2_base_tiling_fast_tiling.cpp, conv3d_v2_base_tiling_fast_tiling.cpp
- 缺陷描述：ConvBase对象内部的flagInfo_成员在fast tiling流程中未被更新。Conv2d/Conv3d的tiling类持有各自的flagInfo_，但在调用convBase_的方法(如GetWeightBandWidthCoeff)之前没有将flag信息同步给convBase_。导致convBase_不知道C04模式已启用，GetWeightBandWidthCoeff()无法命中C04+NHWC分支，返回错误的带宽系数(BW_COEFF_UB=1而非BW_COEFF_C04=10)，造成性能劣化
- 修复模式：新增UpdateFlagInfo()方法在fast tiling入口同步flagInfo_ + 补充C04+NHWC格式的带宽系数分支
- 可审查性：中
- 审查规则建议：组合对象的状态同步检查——若父类持有子组件对象并调用其方法，需验证子组件所依赖的状态字段是否已从父类同步

### a4ea013a7083035071fee5d39cc9e3969a0172a7 fix al1 fulload perf problem&fusematmul problem
- 根因类别：(1)条件判断缺失(性能缺陷) (2)继承层级错误(功能缺陷)
- 涉及文件：matmul/fused_mat_mul/op_host/op_tiling/arch35/fused_matmul_asw_basic_tiling.cpp/.h, matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_basic_aswt_tiling.cpp
- 缺陷描述：两个独立问题。(1) CheckAL1FullLoad()缺少对l0C2Out_模式的检查，当输出不是ON_THE_FLY模式时全载策略不适用但被错误启用，引发性能劣化。(2) FusedMatMulAswBasicApiTiling错误继承自MatMulV3BasicAswtTiling(应为MatMulV3AswTiling)，导致走了Basic级别tiling路径
- 修复模式：(1) CheckAL1FullLoad()入口增加模式前置条件过滤 (2) 更换基类 + 覆写GetTilingKey()
- 可审查性：中
- 审查规则建议：新增子类时应说明基类选择理由，特别是存在多个相似基类时

### 850c7eb03a354a1ce34418509e833e38e4fcb533 修复A L1全载的判断条件
- 根因类别：条件判断不完整(边界条件遗漏)
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/arch35/adaptive_sliding_window_basic_api_tiling.cpp
- 缺陷描述：IsAFullLoad()函数判断A矩阵是否可在L1中全载时，缺少对batch维度的约束。A L1全载仅在batch==1时适用，但原代码没有检查inputParams_.batchC==1。多batch时L1空间不足或复用逻辑不正确，导致错误启用全载
- 修复模式：在isAFullLoad_条件表达式末尾追加 && inputParams_.batchC == 1
- 可审查性：高
- 审查规则建议：全载策略启用条件必须包含batch维度检查，审查tiling策略启用条件时逐条对照M/N/K/Batch约束

### 741ae7a013526702b0bf98d36dd7cf986f4592ed 修改输出日志错误DTS2025121125810
- 根因类别：日志/字符串错误(缩进错误+拼写错误)
- 涉及文件：matmul/quant_batch_matmul_v3/op_api/quant_matmul_checker.cpp
- 缺陷描述：(1)错误信息字符串C++续行符\后下一行开头有多余缩进空格，被包含在最终日志中导致输出不自然。(2)"scenarion"拼写错误，正确为"scenario"
- 修复模式：修正字符串续行缩进 + 修正拼写错误
- 可审查性：高
- 审查规则建议：检查C++字符串续行(\)后的下一行是否有非预期前导空白；引入codespell等工具检查日志字符串拼写

### 12b41a6438acef94a717dce9c227da8b96e6ff44 公式化weight全载开启db错误
- 根因类别：分支逻辑错误(条件组合遗漏导致double buffer决策错误)
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_api_tiling_algorithm_Mmode.cpp/.h
- 缺陷描述：InitABL1TilingMode()在fmap占主导且启用innerBatch(batch>1)时直接降级为NONE_FULL_LOAD，但正确逻辑应先尝试weight全载(FULL_LOAD_BL1)作为fallback。原if-else结构未覆盖此组合路径，丢失double buffer优化机会
- 修复模式：重构条件分支，拆分为4个语义辅助函数(IsAllFullLoadPossible等)，补充innerBatch时fallback到weight全载的路径
- 可审查性：低
- 审查规则建议：涉及多个互斥/组合条件的tiling决策逻辑应提供决策矩阵(truth table)；超过3层嵌套if-else应拆分为命名清晰的辅助函数

### 017f5007edd5cb64e6329c1f20412c8d737aa8ae ISA fix
- 根因类别：ISA接口变更/废弃接口调用
- 涉及文件：norm/batch_norm_grad_v3/op_kernel/arch35/ 下5个头文件
- 缺陷描述：三类问题。(1)使用已废弃的bcnt1()（popcount），应替换为ScalarGetCountOfValue<1>()。(2)调用已废弃的set_mov_pad_val(0)裸指令接口，需删除。(3) PipeBarrier<PIPE_ALL>();;多余分号
- 修复模式：替换bcnt1→ScalarGetCountOfValue<1>(4处) + 删除set_mov_pad_val(0) + 去除多余分号
- 可审查性：中
- 审查规则建议：建立废弃ISA函数清单(bcnt1、set_mov_pad_val等)，CI grep检查禁止使用；linter规则检测连续分号;;

### a592f6fa0921454b49f2ebc15f8f677d514059e7 bugfix for bmm util
- 根因类别：接口参数类型不匹配(bool→int64_t隐式转换导致语义错误)
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_batch_matmul.cpp, matmul/common/op_host/op_api/batch_matmul_util.cpp
- 缺陷描述：TransBmm2Mm的第4个参数是bool enableHf32，但底层MatMulV3Nd期望int64_t opImplModeEnum(位标志枚举)。bool true隐式转换为1，而HF32模式对应0x40，完全不同。enableHf32=false时传入0x0，正确默认值应为0x1。导致矩阵乘法精度模式设置错误
- 修复模式：参数类型改为int64_t，调用方根据enableHf32/enableForceGrpAccForFp32计算正确枚举值(0x40/0x4/0x1)
- 可审查性：中
- 审查规则建议：启用-Wconversion编译警告检测bool→int隐式转换；使用enum class替代raw int避免隐式转换

### 626804ab7ca9ed650b9087ab4a8f349ff306e89f nullptr input error code modify
- 根因类别：错误码语义混用(INNER vs PARAM)
- 涉及文件：matmul/sparse4to2quant_matmul/op_host/op_api/aclnn_sparse4to2quant_matmul_weight_nz.cpp
- 缺陷描述：OP_CHECK_NULL(sparseWeight)检查用户传入参数空指针时使用ACLNN_ERR_INNER_NULLPTR(内部错误)，应为ACLNN_ERR_PARAM_NULLPTR(参数错误)。错误码会误导问题定位
- 修复模式：ACLNN_ERR_INNER_NULLPTR → ACLNN_ERR_PARAM_NULLPTR
- 可审查性：高
- 审查规则建议：函数入口参数校验区域内OP_CHECK_NULL必须搭配PARAM_NULLPTR；对内部指针搭配INNER_NULLPTR

### 3ac83c4ebfca1d81c7546139ffdcbdb8b9386bdc fix: 91095 op single compile error
- 根因类别：文件存在性检查缺失(脚本健壮性)
- 涉及文件：scripts/kernel/binary_script/build_binary_op_exe_task.sh, build_binary_op_exe_task_out.sh
- 缺陷描述：构建脚本在读取任务文件(opc_cmd_file/out_cmd_file)前未检查文件是否存在。单算子编译时可能不生成这些文件，sed/cat读取不存在的文件报错，set -o pipefail导致整个编译失败
- 修复模式：增加 if [ -f "${opc_cmd_file}" ] 前置判断
- 可审查性：高
- 审查规则建议：Shell脚本对变量路径文件操作(cat/sed/source)前必须做[ -f ]检查

### 2790615c7989ac8cb747d8a7299d119a835c9a4d 裸指令回退
- 根因类别：API语义混淆(block级ID vs subblock级ID)
- 涉及文件：conv/common/op_kernel/arch35/conv_opt_group_init_impl.h, conv/conv2d_v2/op_kernel/arch35/ 下3个impl头文件
- 缺陷描述：之前有人把get_subblockid()替换为GetBlockIdx()，两者语义完全不同：前者返回AI Core内部子块(subblock) ID用于区分向量执行单元，后者返回全局block ID用于区分不同AI Core。vecId字段应使用get_subblockid()获取子块ID。错误替换导致卷积计算任务分配和数据寻址错误
- 修复模式：4个文件中GetBlockIdx()全部回退为get_subblockid()
- 可审查性：中
- 审查规则建议：变量名含vec/subblock时赋值使用GetBlockIdx()应触发review警告；底层API批量替换需专项review

### 1d65758e162c8ccc60fa7cbe3ccffdbb4c020646 kernel大小校验过程防止溢出
- 根因类别：整数溢出(int32_t中间结果溢出)
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_basic_block_tiling.cpp
- 缺陷描述：CheckKernelSize()中totalPadingD/H/W使用int32_t存储两个pad值之和，后续与di/hi/wi维度值相加计算kdMax等值时int32_t中间结果会溢出，导致kernel大小校验产生错误结论。旧代码还有不规范的static_cast<int>
- 修复模式：所有中间变量和结果类型从int32_t改为int64_t，第一个加数显式static_cast<int64_t>
- 可审查性：高
- 审查规则建议：多个int32_t值参与加法/乘法且结果仍存int32_t时标记溢出风险，特别关注shape/padding/dilation等用户输入参数的算术运算

### 1059e41e2df11bf745f2896e01448ce5b51072f0 Fix Nz n=1 tranpose
- 根因类别：边界条件缺失(输入校验遗漏)
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp, matmul/mat_mul_v3/docs/aclnnMatmulWeightNz.md
- 缺陷描述：aclnnMatmulWeightNz在mat2为FRACTAL_NZ格式时未校验最后两根轴(k和n)是否为1。n=1或k=1时NZ格式内存布局退化，转置操作产生错误结果。旧代码仅文档声明"不保证精度"但未代码拦截
- 修复模式：CheckWeightNzShapeValid中新增前置校验，mat2Shape.GetDim(0)==1或GetDim(1)==1时返回false拒绝执行
- 可审查性：中
- 审查规则建议：算子声明不支持某些输入场景时必须在代码中有对应参数校验，仅文档说明不足以防止误用

### c866c4defe096d8c1f047553355a24e5368ac4c9 修复fake_quant_affine_cachemask精度问题
- 根因类别：计算逻辑错误(NaN/Inf特殊值处理不当)
- 涉及文件：quant/fake_quant_affine_cachemask/op_kernel/fake_quant_affine_cachemask_fp16.h, fake_quant_affine_cachemask_fp32.h
- 缺陷描述：NaN处理三个问题。(1)将NaN位置替换为quantMin是语义错误，fake quant期望NaN→0。(2)fp32版本第二步Compare比较yLocal而非xLocal，此时yLocal尚未正确赋值，读未初始化数据。(3)Inf检测冗余，Inf经Maxs/Mins clamp已被正确处理
- 修复模式：删除Inf检测冗余逻辑，NaN替换值从quantMin改为0.0f
- 可审查性：低
- 审查规则建议：NaN/Inf特殊浮点值处理必须有对应单测覆盖；对未初始化tensor的读取应作静态分析标记

### 9a1e4b840f2bc31f8c926ff8b4ac7beb6d623971 fix aclnn empty tensor bug
- 根因类别：空tensor边界条件处理不完整
- 涉及文件：norm/group_norm_grad/op_host/op_api/aclnn_group_norm_backward.cpp
- 缺陷描述：910_95平台处理empty tensor两个问题。(1)检查!gamma->IsEmpty()用的是原始gamma而非转换后的gammaContiguous，可能empty状态不一致。(2)缺少gamma本身为empty的处理分支，既不走empty分支也无法正常走后续reshape逻辑
- 修复模式：判断对象从gamma改为gammaContiguous + 新增else if分支处理gammaContiguous为empty时设workspaceSize=0
- 可审查性：中
- 审查规则建议：对tensor进行Contiguous/转换后，后续条件判断应使用处理后的tensor而非原始tensor；每个输入的empty组合都需确认处理路径

### 3570e96d90a0dc0f58e157ccabcf845739e316cc fix dequant_swiglu_quant tiling error
- 根因类别：常量误用(复制粘贴错误)
- 涉及文件：quant/dequant_swiglu_quant/op_host/dequant_swiglu_quant_tiling_base.cpp
- 缺陷描述：CheckKernelUBUpBound()在dynamic模式(quantMode==1)下使用NUMBER_OF_INPUT_SIZE(10)作为buffer数量乘数，正确应为DYNAMIC_BF16_INT16_TBUF_NUM_HALF(6)。同文件另一处计算singleDataSize已正确使用该常量。高估UB内存需求(10 vs 6)导致合法tiling方案被错误判定为超限
- 修复模式：将NUMBER_OF_INPUT_SIZE替换为DYNAMIC_BF16_INT16_TBUF_NUM_HALF
- 可审查性：高
- 审查规则建议：同一文件中对同一计算场景应使用相同常量；常量仅使用一次而附近存在语义相似但值不同的常量被多次使用时标记为可疑

跳过(第180-199行)：5条
- 7e63c460: fix aicpu ut(UT构建脚本修复)
- 581bed62: aclnnMatmulWeightNz资料错误修改(纯文档)
- 2e99c5ad: fix DQUSV2 md(纯文档)
- 856183cd: fix ut case(纯UT测试)
- 884c0ce1: fix ut(纯UT测试)

### 342539b0 修复baddbmm在95平台的算子bug和910b平台的精度bug
- 根因类别：条件判断缺失 / 平台适配遗漏
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp, aclnn_batch_matmul.cpp, batch_matmul_util.cpp
- 缺陷描述：enableFp16Bf16InFp32Out特性在910B/910_93平台对所有fp16/bf16输入都开启，但只应在Baddbmm接口调用时才使能fp32输出，普通BatchMatMul不应走此路径导致精度异常。同时fp16/bf16转fp32路由BatchMatMulV3NdFp16Bf162Fp32没有限制平台，在910_95等不支持平台上被错误命中导致算子执行失败
- 修复模式：将isBaddbmm标志透传到GetBatchMatmulOpInfo/CreateBatchMatmulOpInfo，路由分支增加SocVersion平台判断
- 可审查性：中
- 审查规则建议：平台特定逻辑需检查条件守卫是否足够，共享工具函数的新增特性开关是否在所有调用路径正确传递

### ab0ef822 fix (conv backward 1D-to-2D转换和白名单扩充)
- 根因类别：维度转换逻辑错误 + 边界条件缺失
- 涉及文件：conv/convolution_backward/op_api/aclnn_convolution_backward.cpp, convolutionbackward.cpp, conv3d_backprop_*_tiling_key.h
- 缺陷描述：PreConv1DBackwardTo2D中groups>1时conv1d到conv2d升维变换无条件用params.groups作为维度交换参数，但在dilation==45（magic number标识）时应使用exchangeDim=1跳过groups维度变换。OutputPostProcess中needSpecialCast缺少storageShapeDimSize>1检查
- 修复模式：增加条件分支控制维度交换逻辑，增加维度数量边界检查，扩充白名单
- 可审查性：低
- 审查规则建议：使用magic number(dilation==45)作为逻辑控制标志是代码异味，应用具名常量替代。数组索引操作应检查维度数量前置条件

### 7a1ea034 fix_expand_into_jagged_permute
- 根因类别：配置遗漏
- 涉及文件：scripts/kernel/binary_config/ascendc_config.json
- 缺陷描述：ExpandIntoJaggedPermute算子缺少AscendC二进制编译配置注册，在ascend910b和ascend910_93平台无法使用
- 修复模式：在ascendc_config.json中新增算子配置条目
- 可审查性：高
- 审查规则建议：新增算子应有checklist确保所有配置文件已更新

### 43503a16 conv 2d msg fix
- 根因类别：错误上报缺失
- 涉及文件：conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_v2_tiling_utils.h
- 缺陷描述：tiling工具宏检测到不支持的参数时仅OP_LOGE_WITHOUT_REPORT打印日志但无REPORT_INNER_ERR_MSG上报错误码，上层调用方和错误监控系统无法感知内部错误
- 修复模式：在日志打印后增加REPORT_INNER_ERR_MSG调用，补充头文件引入
- 可审查性：高
- 审查规则建议：所有OP_LOGE_WITHOUT_REPORT应审查是否需要配套REPORT_*_ERR_MSG

### 3d7b9c0d Fix aFullLoad with bias
- 根因类别：计算公式使用错误维度变量
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_basic_aswt_tiling.cpp, .h
- 缺陷描述：MatMulV3 AFullLoad tiling中biasSize计算错误使用mAlignedValue/nAlignedValue，bias大小应基于N维度的实际L1 block大小min(CeilAlign(nValue,16), baseN*DB_SIZE)。导致L1内存占用估算不准确，tiling策略选择错误
- 修复模式：新增GetAFullLoadBasicNL1()方法封装正确的N方向L1计算逻辑
- 可审查性：中
- 审查规则建议：tiling内存计算公式中每个size变量引用的维度是否正确，bias与特定维度绑定需使用对应维度值

### fe6f8610 adaptive_max_pool3d_small_pool编译报错问题
- 根因类别：平台兼容性 / 类型硬编码
- 涉及文件：pooling/adaptive_max_pool3d/op_kernel/adaptive_max_pool3d.cpp, _big_pool.h, _small_pool.h
- 缺陷描述：mask类型硬编码为uint16_t，但310平台Compare指令mask输出为uint8_t，导致该平台编译失败
- 修复模式：引入条件编译using maskdType（310平台uint8_t/其他uint16_t），作为模板参数传递到所有相关类
- 可审查性：中
- 审查规则建议：NPU指令相关代码应检查是否存在平台相关的类型差异（如mask宽度），是否通过条件编译或模板参数适配

### f1bb8177 instance_norm异常处理修复
- 根因类别：错误处理不当 / 缺乏错误日志
- 涉及文件：norm/instance_norm_v3/op_host/op_api/aclnn_instance_norm.cpp
- 缺陷描述：参数校验函数使用CHECK_RET宏，校验失败时静默返回错误码无日志输出，用户无法定位失败原因
- 修复模式：将CHECK_RET替换为OP_CHECK配合OP_LOGE日志输出，每个校验失败点增加具体错误描述
- 可审查性：高
- 审查规则建议：对外API参数校验失败路径必须有OP_LOGE日志输出，禁止使用静默返回的CHECK_RET宏

### 3e9e2acd fix conv3d kernel bug
- 根因类别：状态初始化位置错误
- 涉及文件：conv/conv3d_v2/op_kernel/arch35/conv3d_v2_instr_impl.h
- 缺陷描述：aL1Pos=0初始化放在带参数的重载函数中而非无参数版本的入口。调用流程上无参数版本先计算偏移再调用带参数版本，但aL1Pos在带参数版本中才重置，前序计算依赖的aL1Pos可能保留上一轮迭代脏值
- 修复模式：将this->aL1Pos=0从两个带参数重载函数移到无参数版本LoadAl1Data()入口处
- 可审查性：高
- 审查规则建议：状态变量初始化应放在最早的调用入口而非被调用的子函数中。循环体内复用函数的状态重置时机是否正确

### ec448458 修复aclnnbaddbmm接口精度不通过问题
- 根因类别：数据类型处理缺陷 / 精度损失
- 涉及文件：batch_mat_mul_v3相关17个文件(op_host/op_api, stub, def, config)
- 缺陷描述：baddbmm接口fp16/bf16输入时中间计算以fp16/bf16输出导致精度损失。正确行为在A2/A3平台(910B/910_93)下应输出fp32。ifKEqual1分支也缺少fp32 cast。算子定义缺少fp16->fp32和bf16->fp32类型组合
- 修复模式：新增enableFp16Bf16InFp32Out标志位，新增BatchMatMulV3NdFp16Bf162Fp32函数走fp32输出路径，K=1分支增加Cast处理，算子定义增加类型映射
- 可审查性：低
- 审查规则建议：矩阵乘法算子应验证所有输入dtype×输出dtype组合是否完整覆盖，退化路径(K=1)是否保持主路径一致精度行为

### d5228e13 fix big shape run error for dequantSwigluQuant
- 根因类别：资源预分配校验缺失
- 涉及文件：quant/dequant_swiglu_quant/op_host/dequant_swiglu_quant_tiling_base.cpp
- 缺陷描述：大shape输入(如96000000×3072)时tiling计算出的UB预分配大小超出硬件实际UB空间，代码无校验导致kernel运行时UB分配失败(561002)
- 修复模式：新增CheckKernelUBUpBound()方法，在tiling计算后按kernel实际UB分配逻辑逐项累加所需空间与ubSize比较，超限返回GRAPH_FAILED
- 可审查性：高
- 审查规则建议：tiling阶段必须对kernel所需的关键硬件资源(UB/L1/L0)进行上界校验

### 8a55ce05 fix conv matmul ophost ut
- 根因类别：构建/UT基础设施缺陷
- 涉及文件：cmake/func.cmake, tests/ut/common/CMakeLists.txt, legacy_common_manager_stub.cpp
- 缺陷描述：UT场景下LegacyCommonMgr通过当前so相对路径dlopen libophost_comm_legacy.so，但UT环境部署位置不同导致dlopen失败，conv/matmul的ophost UT编译链接失败
- 修复模式：cmake增加UT场景判断避免引入生产实现，新建stub打桩文件，补充头文件路径和链接依赖
- 可审查性：高
- 审查规则建议：引入新外部依赖(dlopen)时必须同步考虑UT场景的mock/stub方案

### 5c3d6119 fix quantize file name
- 根因类别：构建配置 / 文件命名错误
- 涉及文件：quant/ascend_quant_v2/op_host/CMakeLists.txt, quant/quantize/op_kernel/quantize.cpp→quantize_apt.cpp
- 缺陷描述：kernel源文件quantize.cpp不符合构建系统对apt后缀的命名约定，CMakeLists缺少DEPENDENCIES quantize声明
- 修复模式：重命名文件为quantize_apt.cpp，补充CMakeLists依赖声明
- 可审查性：高
- 审查规则建议：新增kernel文件名是否符合命名约定（_apt后缀），CMakeLists是否声明所有必要DEPENDENCIES

### 5401f5ec fix opsnn bug
- 根因类别：构建依赖缺失
- 涉及文件：norm/add_rms_norm_dynamic_quant, add_rms_norm_quant, group_norm_grad, layer_norm_grad_v3, rms_norm 各自的CMakeLists.txt
- 缺陷描述：5个norm算子的CMakeLists缺少DEPENDENCIES声明(norm_common, rms_norm等)，导致编译链接找不到公共函数
- 修复模式：在add_modules_sources调用末尾追加DEPENDENCIES参数
- 可审查性：高
- 审查规则建议：源码include了公共模块头文件时，CMakeLists是否已声明对应DEPENDENCIES

### 3d307c27 [bugfix] eliminate the risk of nullptr in qbmmv3 tiling
- 根因类别：空指针解引用风险
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/quant_batch_matmul_v3_tiling_base.cpp, .h, fused_quant_matmul_asw_tiling.cpp
- 缺陷描述：CheckFusionBatchA接收const gert::Shape&参数，调用方先对biasShape指针调用->GetStorageShape()再传入。当hasBias为false时指针为nullptr，解引用发生在传参时（函数调用前），函数内的hasBias条件守卫无法保护
- 修复模式：参数改为const gert::StorageShape*指针类型，解引用延迟到函数体内hasBias条件判断之后
- 可审查性：中
- 审查规则建议：传参表达式中对可能为null的指针做解引用，特别是指针有效性依赖于另一个布尔条件时，确保解引用在条件判断之后

### d014e1c3 Fix ACLNN interception for 2D EmbeddingBag cases
- 根因类别：平台适配缺失 / 多层缺陷
- 涉及文件：index/embedding_bag/下12个文件(op_api, op_host tiling, op_kernel)
- 缺陷描述：EmbeddingBag原只支持1D indices，910_95平台需支持2D indices。多处缺陷：(1)ACLNN维度校验对indices硬编码1维；(2)offset2bag shape用GetDim(0)而非GetShapeSize()，2D下只取第一维；(3)kernel空bag初始值用(T)(-1)应为(T)(0)；(4)tiling缺inclueLastOfst字段传递；(5)sum模式kernel循环用错indicesFactor应为weightRowFactor
- 修复模式：引入SocVersion::ASCEND910_95分支走不同校验和计算逻辑，tiling新增字段，kernel修正初始值和循环变量
- 可审查性：低
- 审查规则建议：新平台适配应有独立checklist明确各层修改点。GetDim(0) vs GetShapeSize()使用应有明确规范

### 1d6b119a scatterElementsV2 fp16/bf16 bugfix
- 根因类别：复制粘贴错误
- 涉及文件：index/scatter_elements_v2/op_kernel/scatter_elements_v2.h
- 缺陷描述：Cast(updatesTemp, updatesLocal, ..., indicesAlign)中第4参数应传updatesAlign而非indicesAlign。上行Cast用inputLocal配inputAlign，下行复制粘贴后未将indicesAlign改为updatesAlign，导致fp16/bf16时Cast长度参数错误
- 修复模式：将indicesAlign改为updatesAlign
- 可审查性：高
- 审查规则建议：Cast/Copy等批量操作的目标buffer与长度参数应有命名对应关系，名称不匹配是red flag

### d221fb68 bugfix for wqbmm 310p kernel config
- 根因类别：配置遗漏
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/config/ascend310p/weight_quant_batch_matmul_v2_binary.json
- 缺陷描述：310P平台kernel配置缺少dtype和inner_precise两个参数，导致kernel无法正确获取数据类型和精度控制参数
- 修复模式：在attrs数组补充dtype(默认-1)和inner_precise(默认0)参数项
- 可审查性：中
- 审查规则建议：新增kernel参数时检查所有平台配置文件是否同步更新，建立配置一致性校验脚本

### b946a8f5 修复图模式需要int64的场景
- 根因类别：数据类型支持不完整
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/conv3d_backprop_input_v2_def.cpp
- 缺陷描述：Conv3dBackpropInputV2算子定义中input_size只声明INT32类型，图模式下框架可能传入INT64导致类型匹配失败
- 修复模式：将数据类型组合从5组扩展到10组，新增INT64和FLOAT类型支持
- 可审查性：高
- 审查规则建议：算子定义各输入输出DataType/Format列表长度必须一致。图模式和单算子模式的类型需求差异应在设计文档中明确

### 9624d01c fix perf downgrade
- 根因类别：性能回退 / 架构设计
- 涉及文件：matmul/weight_quant_batch_matmul_v2/下39个文件(tiling + kernel)，删约3216行新增约766行
- 缺陷描述：ACT代码中L0/L1 K列表包含1024配置导致tiling不优，MTE2配置多级分支引入开销，事件同步标识命名错误(MTE2_TO_MTE2应为MTE1_TO_MTE2)，A16W4 pergroup单独路径未合并到通用模板导致代码膨胀
- 修复模式：缩减K_LIST去掉1024，简化MTE2配置，修正事件标识，删除专用路径统一合并到ScmcKernel模板
- 可审查性：低
- 审查规则建议：性能相关改动必须附带benchmark数据。事件同步标识命名应与硬件事件语义一致

### 6650178c FlatQuant修复inf、nan场景精度不一致问题
- 根因类别：数值计算 / 矩阵维度错误
- 涉及文件：quant/flat_quant/op_host/flat_quant_tiling.cpp, op_kernel/flat_quant_cube.h, flat_quant_vec.h, tensor_utils.h
- 缺陷描述：MM_DOUBLE_MODE下多处维度计算错误：(1)workspace用4*Mceil^2应为Mceil^2；(2)P2分配用calM*calM但calM=2*Mceil导致4倍空间且越界；(3)缺calN/calFractalM/calFractalN导致fractal维度计算错误；(4)ProcessP1错误合并两块P1数据；(5)流水线事件x2p1ready未正确复用l0ready
- 修复模式：修正workspace公式，P2改Mceil*Mceil，新增calN/calFractalM/calFractalN字段，重写ProcessP1逻辑，修正流水线事件
- 可审查性：低
- 审查规则建议：double/倍增模式下所有维度变量需逐个确认是否需翻倍。workspace大小公式应有注释说明推导过程

跳过的提交(本轮行200-224):
- 8a6214d6: fix tiling ut for fusedquantmatmul(纯UT测试)
- d966e6ea: fix aclnn md(纯文档)
- 0570c89a: fix examples and geir(构建脚本适配/非代码缺陷)
- 0a1e2f6c: examples_fix(纯文档/注释修正)
- 03ea5201: fix aclnnFusedLinearOnlineMaxSum.md(纯文档)

### 5fd1fb95 修改lstm整包路径识别错误
- 根因类别：路径/引用错误
- 涉及文件：rnn/bidirection_lstmv2/op_kernel/bidirection_lstmv2.cpp
- 缺陷描述：`#include`路径写错，原路径`../../bidirection/op_kernel/lstm_bidir_fp16.cpp`指向错误目录，应该是`../bidirection_lstm/lstm_bidir_fp16.cpp`。整包编译场景下目录结构与开发时不同，导致include找不到正确文件或链接到错误实现
- 修复模式：单行路径字符串替换，将相对路径修正为正确的相邻目录
- 可审查性：高
- 审查规则建议：检查`#include`中使用`../../`等多级回退的相对路径引用，特别是include .cpp文件（而非.h文件）的非常规用法，应标记为高风险

### 4a423e45 fix dw acc
- 根因类别：特定shape下tiling参数缺失
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_base_tiling.cpp
- 缺陷描述：conv3d反向滤波器算子在特定输入shape（dilation kernel场景）下走入默认tiling初始化分支，缺少stepN=390的特殊配置，导致计算精度异常。修复通过新增TILINGDATA_MAP_TMP硬编码映射表匹配该shape
- 修复模式：硬编码特定shape的tiling参数（workaround风格），新增UT用例覆盖
- 可审查性：低
- 审查规则建议：关注tiling参数中使用硬编码map+全零初始值的模式，通常是针对特定case的临时补丁，需检查是否有更通用的计算逻辑来取代hardcode

### 3e156100 gng 读越界
- 根因类别：内存读越界（DataCopy对齐多读）
- 涉及文件：norm/group_norm_grad/op_kernel/arch35/下6个文件（group_norm_grad_base.h, _c_full_load.h, _g_full_load.h, _recompute.h, _small_ng_c_full_load.h, _apt.cpp）
- 缺陷描述：GroupNormGrad算子（arch35平台）使用DataCopy搬运数据时，count参数经CeilAlign对齐到block大小。当实际数据量不是block整数倍时，对齐后的count超出GM tensor边界，造成读越界
- 修复模式：将`DataCopy(dst, src, CeilAlign(count))`替换为`DataCopyPad(dst, src, {blockLen=count*sizeof(T)})`，消除对齐导致的越界读取。系统性修复同一问题在CFullLoad/GFullLoad/ReCompute/SmallNG多个模式中的所有出现
- 可审查性：中
- 审查规则建议：搜索所有DataCopy调用中使用CeilAlign对齐count参数的地方，检查是否存在GM边界越界风险；当tensor的实际元素数不是block size整数倍时，应优先使用DataCopyPad

### e52bc834 bugfix for wqbmm
- 根因类别：模板参数声明错误（多余的TILING_STRUCT_SEL参数）
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/weight_quant_batch_matmul_v2_kernel_tiling_key.h
- 缺陷描述：wqbmm v2算子的kernel tiling key模板声明中，所有分支（约20处）都多包含了一行`ASCENDC_TPL_TILING_STRUCT_SEL(WeightQuantBatchMatmulV2*TilingData)`，导致编译或运行时异常
- 修复模式：批量删除所有tiling key模板声明中多余的ASCENDC_TPL_TILING_STRUCT_SEL行
- 可审查性：中
- 审查规则建议：ASCENDC_TPL_ARGS_DECL宏的参数列表变更时，检查所有使用该宏的分支是否保持一致

### a6b67225 修复上边界问题
- 根因类别：整数溢出（int32不足）
- 涉及文件：vfusion/modulate/op_host/modulate_tiling.cpp, op_api/aclnn_modulate.cpp
- 缺陷描述：CalcTilingParam函数的totalElements参数类型为int（32位），当tensor较大时totalElements超过INT_MAX导致溢出，后续除法计算totalElements/coreNum结果错误
- 修复模式：参数类型从int改为int64_t消除溢出
- 可审查性：高
- 审查规则建议：tiling计算函数中涉及元素数量/字节数的参数和中间变量，应使用int64_t而非int/int32_t，防止大shape场景溢出

### 9aec1447 fix addmm bug
- 根因类别：平台相关dtype校验缺失
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp
- 缺陷描述：A2(910B)/A3(910_93)平台下调用aclnnInplaceAddmm时走入CheckGemmV3Support，但该函数缺少对这些平台上数据类型的校验——gemmv3只支持fp16*fp16和bf16*bf16，其他dtype（如fp32）被放行导致执行异常
- 修复模式：在两个重载的CheckGemmV3Support函数中各增加平台判断+dtype白名单检查
- 可审查性：高
- 审查规则建议：新增算子平台支持时，需同步审查dtype校验是否覆盖完整；当同一函数有多个重载版本时，修复需确保所有重载同步更新

### 890af289 问题单修复
- 根因类别：数据格式分支处理不完整（NZ格式stride计算错误）
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/下2个文件
- 缺陷描述：conv3d反向输入算子在arch35平台上，不使用vecTrans且filter为NZ格式时，kernelSplitStrideB_的计算仍按NDHWC逻辑乘cin，导致stride偏大访问错误GM地址
- 修复模式：将else分支拆分为NDHWC和非NDHWC两种情况，NZ格式下stride只取wk不乘cin。新增LoadToB1Dn2NzTransposeForKernelSplitH函数处理kernel在H维度split时的权重搬运
- 可审查性：中
- 审查规则建议：当代码中出现二分分支且else分支隐含某种格式假设时，检查是否需要按实际格式细分

### 08515924 修复tilingdata从host memcpy_s npu错误
- 根因类别：API误用 / 指针语义错误
- 涉及文件：quant_batch_matmul_v4/op_host/op_tiling/quant_batch_matmul_v4_msd_tiling.cpp
- 缺陷描述：memcpy_s的第三个参数tilingData_本身已是指针类型，但代码使用&tilingData_（取指针的地址），导致拷贝的是指针变量自身的内存而非所指向的tiling数据结构体，NPU侧拿到完全错误的tiling数据
- 修复模式：将&tilingData_改为tilingData_，去掉多余的取地址操作
- 可审查性：高
- 审查规则建议：对memcpy_s/memcpy调用进行静态检查，当源参数对指针类型变量再次取地址时发出警告

### 53c9881a fix single op opgraph
- 根因类别：构建系统依赖顺序错误
- 涉及文件：cmake/gen_ops_info.cmake, cmake/symbol.cmake
- 缺陷描述：merge_graph_headers这个cmake target定义在gen_ops_info.cmake中，但它实际应作为opgraph shared library构建的依赖。single op场景下不走gen_ops_info流程时graph header合并缺失，opgraph构建失败
- 修复模式：将merge_graph_headers从gen_ops_info.cmake移到symbol.cmake中gen_opgraph_symbol函数内
- 可审查性：中
- 审查规则建议：CMake中add_dependencies(A B)的target B应在同一函数或更早的作用域中定义

### 4cd50276 fix group_norm_silu small shape case
- 根因类别：小shape边界条件处理不当
- 涉及文件：norm/group_norm_silu/op_host/group_norm_silu_tiling.cpp, op_kernel/group_norm_silu_hw1_b32.h
- 缺陷描述：hwNum==1且numGroups==shapeC（每个channel一个group的小shape场景）时：(1) tiling侧未按N维度重新分核，多核场景核间数据划分错误；(2) kernel侧gmOffset计算缺少*shapeC系数，ProcessYWithEqualC中数据搬运逻辑未正确按C维度循环
- 修复模式：tiling侧增加hwNum==1 && numGroups==shapeC特殊分支；kernel侧重写ProcessYWithEqualC为按C维度分块循环的正确实现
- 可审查性：中
- 审查规则建议：算子kernel中涉及多维tensor的偏移量计算时，审查gmOffset是否正确包含所有维度的stride因子

### 29a25677 matmulv3使能分组累加，kernel config文件修改
- 根因类别：配置参数名/类型不匹配
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp, matmul/mat_mul_v3/op_host/config/各平台binary.json
- 缺陷描述：matmulv3引入分组累加功能后，kernel config中配置项名称仍用旧的enable_hf32(bool)，代码侧已改为opImplMode(int枚举)，名称和类型不一致导致功能无法正确使能
- 修复模式：将所有平台json配置中enable_hf32(bool/false)替换为opImplMode(int/1)
- 可审查性：中
- 审查规则建议：kernel config json中的参数名和dtype应与代码中的枚举/结构体定义保持一致

### 0f9c8ebc 修复ascend_quant_v2算子310p上编译失败的问题
- 根因类别：配置遗漏 — 平台特定kernel参数缺失
- 涉及文件：quant/ascend_quant_v2/op_host/config/ascend310p/ascend_quant_v2_binary.json
- 缺陷描述：310p平台binary配置json中缺少axis参数的声明，kernel代码读取该参数时编译失败
- 修复模式：在310p的两个kernel binary配置段中补充axis参数项
- 可审查性：高
- 审查规则建议：同一算子的不同平台binary json配置应具有一致的参数列表

### 1072e625 fix cross_entropy_loss_full_load bug
- 根因类别：ReduceMax分支处理不统一 / 未初始化寄存器参与归约
- 涉及文件：loss/cross_entropy_loss/op_kernel/arch35/cross_entropy_loss_full_load.h
- 缺陷描述：求行最大值（ReduceMax）分为cNum<vfLen和cNum>=vfLen两个分支，小分支中UnPack后直接cast再ReduceMax，对tailNum的mask处理不完整，未初始化寄存器区域参与了Max计算可能引入脏数据；边界case（cNum恰好等于vfLen时tailNum为0）行为不正确
- 修复模式：统一为一个实现：先用Duplicate(minValue)初始化寄存器，对tail部分用mask Max确保padding区域不影响结果，消除分支不一致性
- 可审查性：低
- 审查规则建议：ReduceMax/ReduceMin等归约操作前，检查寄存器是否已用安全值初始化，防止未初始化区域参与归约

跳过的提交(本轮行225-244):
- b508a0e0: fix example(纯example代码取消注释)
- 80461871: fix example(纯example代码取消注释)
- 22415d57: 修正arch35下example的License(纯License头+example warning)
- e4a0caad: fix coverage script(纯构建/CI脚本)
- a782c40c: add_layer_norm_quant fix_Warning(纯格式化/尾部空白)
- 5e8f5e7e: fix op_list info(纯文档链接修复)
- 482195e2: fix GroupNormSilu V1 V2 md(纯文档)

## 第11批 (defect_commits.txt #245-264)

### 758202d9 日志打印问题修复
- 根因类别：格式化字符串类型不匹配 + 算子定义表数据类型/格式列表数量多余
- 涉及文件：conv/conv3d_backprop_filter_v2 tiling, conv3d_backprop_input_v2 def, convolution_backward checker
- 缺陷描述：(1) OP_LOGE使用%ld打印int类型的params_.groups，32位int下格式不匹配，修复为%d。(2) conv3d_backprop_input_v2_def中910_95平台的DataType/Format列表多出DT_HIFLOAT8和DT_FLOAT8_E4M3FN对应条目，导致各列表长度不一致，删除多余条目修复。
- 修复模式：格式符修正(%ld->%d) + 列表条目删除对齐
- 可审查性：中
- 审查规则建议：启用-Wformat警告；对算子定义中DataType/Format/UnknownShapeFormat列表编写CI脚本校验长度一致性

### fe16c557 Revert "layernormv4功能补齐"
- 根因类别：功能补齐提交引入质量问题，需整体回退
- 涉及文件：norm/layer_norm_v4/ 下22个文件（tiling/kernel/UT）
- 缺陷描述：原始提交做了三件事：(1)新增CommonTiling和MergeNTiling两个tiling模板及kernel实现；(2)在SingleReadTiling::IsCapable()首行插入return false强制禁用；(3)扩大V4路径芯片适配范围(新增910B和910_93)。Revert表明新增代码质量问题，需回退到仅910_95支持。UT中tiling_key从600恢复到100(SingleRead)，说明原提交改变了tiling路由。
- 修复模式：全量Revert
- 可审查性：低
- 审查规则建议：大型功能补齐应拆分独立PR；强制禁用已有模板(return false)需注释原因；扩大芯片适配前需对应平台测试覆盖

### 538b635a fix group_norm_silu hw=1 C=G case
- 根因类别：边界条件下tiling和kernel计算逻辑错误
- 涉及文件：norm/group_norm_silu tiling.cpp, hw1_b32.h kernel, UT
- 缺陷描述：hw=1且C=G(每group只有1个channel)时两类错误。(1) tiling: SetBlockTiling中C=G特殊分支逻辑错误导致多核数据分配错乱，修复删除整个特殊分支统一走通用逻辑。(2) kernel: gmOffset计算多乘了shapeC，C=G时每组只有1个元素偏移应去掉乘shapeC；ProcessYWithEqualC中按shapeC循环但C=G时shapeD=1不应按shapeC循环；ProcessMean/Rstd中groups*shapeC修正为groups。
- 修复模式：删除tiling错误特化分支 + 修正kernel偏移和循环逻辑
- 可审查性：中
- 审查规则建议：算子tiling/kernel对边界条件(C=G,hw=1,shapeD=1)必须有UT覆盖；gmOffset偏移计算review时检查各维度乘法因子与内存布局一致性

### 41690e1c fix aclnnbng check
- 根因类别：参数校验遗漏 + 数据类型推导缺陷 + 日志级别错误 + 空指针风险
- 涉及文件：norm/batch_norm_grad_v3 tiling.cpp, aclnn_batch_norm_backward.cpp
- 缺陷描述：四个独立问题。(1) OP_LOGE用于正常调试信息，修正为OP_LOGD。(2) inference场景不允许saveMean/saveInvstd为shape[0]，但inference时可以为空，放宽校验。(3) 混合dtype场景(gradOut=fp32,weight=fp16)直接传给L0算子类型不匹配，修复显式Cast到fp32。(4) 多处L0操作结果未检查空指针(outputMask指示不需要时可能为nullptr)。
- 修复模式：日志级别修正 + 校验条件放宽 + 显式Cast + 空指针检查
- 可审查性：中
- 审查规则建议：L0/L1算子调用前所有输入tensor dtype必须显式对齐；可能为nullptr的tensor指针使用前必须检查；OP_LOGE仅用于错误路径

### 2eba3d75 fix expand indices 路由 scatterUpdate 校验
- 根因类别：整数类型截断 + 路由条件校验不完整
- 涉及文件：index/scatter_elements_v2 aclnn_scatter.cpp
- 缺陷描述：IsMeetUpdateShape函数两个问题。(1) 维度值比较用static_cast<int32_t>截断，大shape时比较结果错误，修复为int64_t。(2) 循环体只校验updatesShape==selfShape，遗漏updatesShape==indexShape的校验，可能导致不满足条件的case错误路由到scatterUpdate。
- 修复模式：类型提升(int32->int64) + 补充缺失校验条件
- 可审查性：高
- 审查规则建议：shape维度值比较一律使用int64_t禁止强转int32_t；路由条件函数需UT覆盖正例和反例

### e3e7fe45 修复特定shape下matmulv3精度问题
- 根因类别：splitK reduce阶段输出偏移计算未区分切分方向
- 涉及文件：matmul/mat_mul_v3 mat_mul_deterministic_splitk_kernel.h, UT
- 缺陷描述：ReduceKNzInUb中currOutCOffset固定为按M维度切分的偏移公式(index*N*singleCoreM)，但orderFlag=true时应按N维度切分(index*singleCoreN)。错误偏移导致输出写入位置错误产生精度问题。
- 修复模式：根据orderFlag条件分支选择不同偏移计算
- 可审查性：中
- 审查规则建议：涉及多种切分模式的kernel，偏移/stride计算必须对每种模式分别验证；UT应覆盖不同iterateOrder组合

### b694cfda 1952_ccec_bugfix
- 根因类别：特定架构(5102)不支持参数化加载接口
- 涉及文件：conv/common/op_kernel/arch35/ conv_instr_hw_mode_impl.h, conv_instr_impl.h, conv_instr_m_mode_impl.h
- 缺陷描述：arch35的Load3DBitModeParam/Load2DBitModeParam参数化接口在NPU_ARCH==5102上不支持(CCEC编译器/指令集限制)。原代码统一使用参数化调用，5102上编译错误。修复用#if条件编译在5102上改用直接调用img2colv2_cbuf_to_ca/load_cbuf_to_cb。
- 修复模式：条件编译(#if __NPU_ARCH__)区分架构指令调用方式
- 可审查性：低
- 审查规则建议：新增架构适配时系统性扫描所有参数化接口使用位置；条件编译分支需对应架构CI验证

### 7da21af9 fix aclnnindex aicpu to aicore
- 根因类别：平台适配遗漏(910_95特性分支未正确处理)
- 涉及文件：index/index aclnn_index.cpp
- 缺陷描述：aclnnIndex判断aicore路径时对910_95平台缺少特殊处理：(1)indices连续性检查对910_95不应生效但未做平台区分；(2)small tail transpose优化对910_95不适用但无条件启用；(3)is91095变量声明位置在使用之后需提前。
- 修复模式：添加平台版本条件分支用if(!is91095)守护
- 可审查性：中
- 审查规则建议：新增SoC平台支持时系统性搜索所有SocVersion条件分支确认新平台归属

### 6ea06eb1 fix SparseSoftmaxCrossEntropyWithLogits bug
- 根因类别：寄存器级向量操作逻辑错误(UnPack丢失高半部分数据)
- 涉及文件：loss/sparse_softmax_cross_entropy_with_logits arch35 full_load.h
- 缺陷描述：half精度路径中UnPack默认只提取LOWEST半数据，丢失HIGHEST部分。ReduceMax只在一半数据上求最大值导致结果偏小，后续softmax数值错误。修复分别对LOWEST/HIGHEST做UnPack+Cast后取Max合并再ReduceMax。
- 修复模式：补全数据通路——单次UnPack拆为LOWEST/HIGHEST两次，增加Max合并
- 可审查性：低
- 审查规则建议：UnPack操作review时强制要求注释数据宽度关系，确认是否需处理HIGHEST部分

### 62b86225 修复大shape超过int表示范围
- 根因类别：整数溢出(int32用于shape计算) + 输出目标tensor错误
- 涉及文件：vfusion/modulate/op_kernel/modulate.cpp
- 缺陷描述：(1) Init中baseOffset/scaleShiftOffset/bufferSize等声明为int(32位)，大tensor时乘法溢出int32导致地址计算错误。(2) Mul结果写入xLocal(源tensor)而非yLocal(目标tensor)，in-place写入破坏后续读取。
- 修复模式：int替换为int64_t + 修正Mul输出目标tensor
- 可审查性：高
- 审查规则建议：kernel代码禁止裸int做shape/offset计算，lint强制int64_t；向量计算API检查输出参数是否误与输入同buffer

### 24c2eab0 bugfix: miss check on 910_93
- 根因类别：平台适配遗漏(910_93未加入功能准入判断)
- 涉及文件：matmul/common matmul_util.cpp, matmul/mat_mul_v3 aclnn_mm.cpp
- 缺陷描述：Split-K和K=1特殊路径准入函数只检查ASCEND910B，遗漏ASCEND910_93。910_93本应支持但被排除，导致性能劣化。
- 修复模式：SoC版本判断增加ASCEND910_93
- 可审查性：高
- 审查规则建议：维护SoC能力矩阵配置表，所有特性准入用查表实现避免散落硬编码

### 1b26459e fix calssify_rule
- 根因类别：配置文件重复条目(YAML分类规则路径重复/错位)
- 涉及文件：classify_rule.yaml
- 缺陷描述：matmul-c分组包含本属于matmul和matmul-quant的路径条目，同一路径被多个owner匹配，可能导致CI流水线或审批人错误。
- 修复模式：删除重复YAML条目重新归类
- 可审查性：高
- 审查规则建议：CI校验classify_rule.yaml中同一path不允许出现在多个分组中

### e95284a2 fix adaptivemaxpool small模板
- 根因类别：buffer InitBuffer调用顺序错误导致内存布局与预期不符
- 涉及文件：pooling/adaptive_max_pool3d small_pool.h, UT
- 缺陷描述：mulWBuffer(64KB)和mulWIdxBuffer(32KB)的InitBuffer调用顺序错误。TPipe中buffer按InitBuffer顺序连续分配，交换顺序意味着起始地址互换，后续使用有基于顺序的隐含假设时数据错乱或越界。
- 修复模式：调换两行InitBuffer顺序
- 可审查性：低
- 审查规则建议：TPipe::InitBuffer调用顺序具有语义，buffer声明处应注释分配顺序依赖关系

### d34b41cb 修复图模式reg_base报错
- 根因类别：模板框架配置与编译工具链不兼容
- 涉及文件：matmul/weight_quant_batch_matmul_v2 arch35 tiling_key.h
- 缺陷描述：tiling_key中使用ASCENDC_TPL_TILING_STRUCT_SEL宏，但tbe编译器compile_op.py将opParaSize错误设为0，导致图模式下无法获取正确tilingData。去掉STRUCT_SEL后默认AS结构体(280B)大于RegBase(256B)仍能正确获取。
- 修复模式：删除所有ASCENDC_TPL_TILING_STRUCT_SEL行(28处)及相关include
- 可审查性：低
- 审查规则建议：新增ASCENDC模板框架宏时需在CI添加图模式端到端编译验证

### 9cc8c69c revert 2691
- 根因类别：不兼容重构引入编译/链接问题
- 涉及文件：matmul/fused_quant_mat_mul op_tiling arch35 fused_quant_matmul_tiling.cpp
- 缺陷描述：MR !2691(68个文件重构dlopen legacy ophost)在fused_quant_matmul_tiling.cpp新增using Ops::NN::TilingPrepareForOpCache，该符号在部分编译环境不可用导致编译/链接失败。本次仅删除该using声明(局部revert)。
- 修复模式：删除不兼容的using声明
- 可审查性：中
- 审查规则建议：大规模重构MR应分阶段提交验证，68文件单MR风险过高

跳过：
- cf8b43f1: fix assert aicpu_kernel json(纯json配置文件补充input/dynamic_input字段，无逻辑代码)
- a85a9e5e: fix weight nz demo(纯文档markdown修复demo代码示例)
- a1535094: [dx] fix aclnnConvTbcBackward introduction: Stride=1(纯文档markdown修复公式和排版)
- 93af994f: ziliao fix(纯文档/API注释修改，无代码逻辑变更)
- dbd0b626: fix dts(纯文档修改aclnnIndexCopy的md文件)

### 25d1b6bb FlatQuant修复小值域精度问题
- 根因类别：算法/精度缺陷 -- half精度不足导致小值域量化误差
- 涉及文件：quant/flat_quant/op_kernel/ flat_quant_cube.h, flat_quant_high.h, flat_quant_vec.h, tensor_utils.h
- 缺陷描述：量化计算路径直接用half做乘法再cast到int4b_t，小值域下half精度不足导致量化结果偏差大。修复改为先cast到float做乘法再CAST_RINT取整回half。同时CalReduceMax使用两次WholeReduceMax引入冗余中间buffer，NaN检测用手写`tmpMax != tmpMax`。L0C buffer position从TPosition::C2误用改为TPosition::CO1。
- 修复模式：提升中间计算精度(half->float->half路径) + 简化reduce算法 + 修正buffer位置枚举
- 可审查性：低
- 审查规则建议：量化算子中涉及scale乘法的路径应检查中间精度是否足够；避免手写NaN检测应使用标准API；Buffer TPosition枚举值应与硬件手册对应

### 06d1309e add_rms_norm_cast overflow check
- 根因类别：整数溢出 -- workspace大小计算无溢出保护
- 涉及文件：norm/add_rms_norm_cast/op_host/add_rms_norm_cast_tiling_arch35.cpp
- 缺陷描述：GetWorkspaceSize()计算workspaceSize = WORKSPACE_COUNT * usedCoreNum * numN * sizeof(float)时，numN极大(如2147483648)导致uint64_t乘法溢出，产生错误的workspace大小。修复前完全没有溢出检查。
- 修复模式：乘法前逐步除以各因子计算maxAllowed上界，超出返回GRAPH_FAILED（"先除后比"溢出检测模式）
- 可审查性：高
- 审查规则建议：tiling/workspace大小计算中涉及多变量连乘的表达式必须有溢出检查；可提取通用SafeMultiply工具函数

### 3466ffb6 fix level2
- 根因类别：拼写错误(typo) -- cmake安装路径拼写错误
- 涉及文件：cmake/variables.cmake
- 缺陷描述：ACLNN_OP_INC_INSTALL_DIR路径值中level2被拼写为levle2，导致built-in模式下头文件安装到错误路径
- 修复模式：单字符修正 levle2 -> level2
- 可审查性：高
- 审查规则建议：路径字符串可用CI脚本校验安装后目标目录是否存在；cmake安装路径建议集中管理

### ebed9d6e 修正scatter_list的proto.h
- 根因类别：语法错误 -- 缺少闭合花括号
- 涉及文件：index/scatter_list/op_graph/scatter_list_proto.h
- 缺陷描述：ScatterList算子的proto.h在OP_END_FACTORY_REG宏调用后缺少闭合}，导致编译错误
- 修复模式：添加缺失的}闭合括号
- 可审查性：高
- 审查规则建议：proto.h应有编译CI验证；新增算子proto.h应有模板校验确保namespace正确闭合

### db4b7dc3 fix adapitvemaxpool3d indices错误
- 根因类别：边界条件判断错误(off-by-one)
- 涉及文件：pooling/adaptive_max_pool3d/op_host/adaptive_max_pool3d_small_pool_tiling.cpp
- 缺陷描述：IsCapable()中判断kernel尺寸是否在限制范围内时使用<=，应为<。当kernelDMax*kernelHMax*kernelWMaxAlign恰好等于KERNEL_SIZE_LIMIT时，原代码错误认为有能力处理，但实际已达上限应走其他分支
- 修复模式：将<=改为<，收紧边界判断
- 可审查性：高
- 审查规则建议：涉及资源容量/尺寸上限的比较操作需明确"等于边界值"时的预期行为；比较操作符与LIMIT/MAX/SIZE常量组合时应确认<=与<的选择

### 3aa3ec75 算子kernel大小校验问题--C++风格问题整改
- 根因类别：编码风格/类型规范
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_basic_block_tiling.cpp
- 缺陷描述：CheckKernelSize()使用C风格int类型和(int)floor()强制转换。改为int32_t确保类型宽度，static_cast替代C风格cast，去掉整数除法上下文中多余的floor()
- 修复模式：int->int32_t + C cast->static_cast + 去冗余floor
- 可审查性：高
- 审查规则建议：lint规则禁止C风格强制类型转换；整数运算中调用floor()/ceil()应发出警告

### e2cb86e7 [bugfix] Get L1 size by platform info
- 根因类别：硬编码常量/平台适配缺陷
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_api_tiling_util.h, conv/conv2d_v2/op_host/op_tiling/arch35/conv2d_api_tiling.cpp
- 缺陷描述：L1 cache大小被硬编码为常量L1_SIZE_VALUE=524488，不同硬件平台L1大小不同，导致tiling策略选择错误。同时日志格式串%u与uint64_t类型不匹配
- 修复模式：删除硬编码常量改为platformInfo.l1Size动态获取；修正printf格式符
- 可审查性：高
- 审查规则建议：硬件参数相关常量应优先通过platform API获取；搜索L1/L2 size、core count等硬编码常量标记审查

### 5b7c5689 fix masked_scatter l2
- 根因类别：多重逻辑缺陷(冗余转换/指针类型/控制流/分支处理)
- 涉及文件：common/stub/op_api/op_api_stub.cpp, index/masked_scatter/op_host/op_api/aclnn_masked_scatter.cpp
- 缺陷描述：(1)Cast位置错误：ProcessBroadcast内部和外部重复对mask做Cast(DT_BOOL)；(2)指针类型不安全：输出参数aclTensor** out用const_cast去const赋值；(3)循环缺break：设isBA=false后未break继续无意义迭代；(4)910_95分支传参错误：传入selfRefContiguous而非selfRef，mask未传已cast的maskBool
- 修复模式：重构函数接口(const正确性)、消除冗余操作、修正控制流、统一分支后处理
- 可审查性：低
- 审查规则建议：const_cast使用应触发审查警告；循环中设flag后不break应触发lint告警；同一数据类型转换在调用链中出现多次应检查冗余

### 261c36f2 fix AddRmsNormDynamicQuant infershape bug
- 根因类别：条件校验逻辑缺陷(动态shape场景未覆盖)
- 涉及文件：norm/add_rms_norm_dynamic_quant/op_host/add_rms_norm_dynamic_quant_infershape.cpp
- 缺陷描述：infershape中用*gammaShape != *smooth1Shape直接比较shape一致性，但动态shape场景下shape值可能是-1/-2等未知值，直接比较会错误拒绝合法输入
- 修复模式：删除不适用于动态场景的静态shape一致性校验
- 可审查性：中
- 审查规则建议：shape校验代码必须考虑动态shape场景(-1/-2等未知维度值)

### e1fdbe6e Revert "matmulv3支持选择分组累加方式计算"
- 根因类别：功能设计/接口变更回退
- 涉及文件：matmul/mat_mul_v3及batch_mat_mul_v3 多文件(proto.h/tiling/api/UT)
- 缺陷描述：将MatMulV3的enable_hf32(bool)属性改为opImplMode(int64)以支持FORCE_GRP_ACC_FOR_FP32模式的功能被完整回退，说明该特性存在问题。涉及算子属性类型变更、tiling计算公式修改等
- 修复模式：全量Revert
- 可审查性：低
- 审查规则建议：算子接口属性类型变更(bool->int)需完整兼容性测试；Revert提交应说明回退原因

### cbc33130 修复 wqbmmv2 和 qbmmv4 在子包编译失败的问题
- 根因类别：配置遗漏/依赖错误
- 涉及文件：matmul/quant_batch_matmul_v4/op_kernel/arch35/ reg_base_common.h, matmul/weight_quant_batch_matmul_v2/op_host/config/ binary.json
- 缺陷描述：(1)qbmmv4引用不存在的头文件../../common/anti_quant.h改为使用AscendC::IsSameType；(2)wqbmmv2的binary.json全部22个变体缺少inner_precise参数导致子包编译失败
- 修复模式：补齐配置参数；修正头文件依赖
- 可审查性：高
- 审查规则建议：binary.json修改应引入schema校验自动检查必填字段；引用外部头文件应验证所有编译目标下可达

### efe09d11 fix aclnn_hardsigmoid CheckParams
- 根因类别：参数校验过严(过度约束)
- 涉及文件：activation/hard_sigmoid/op_host/op_api/aclnn_hardsigmoid.cpp
- 缺陷描述：CheckParams对format校验同时要求输入输出format相同且必须为ND格式，实际算子应支持非ND格式(如NZ)，只需保证输入输出format一致即可。过严校验导致合法非ND格式输入被拒绝
- 修复模式：放宽校验条件，删除"必须为ND"约束，仅保留输入输出format一致性检查
- 可审查性：高
- 审查规则建议：参数校验中硬编码特定format(!= FORMAT_ND)时应review是否有充分理由排除其他format

### ece9f87e fix: sc of wqbmmv2
- 根因类别：编译告警/代码规范违规
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/arch35/ 多文件(.cpp/.h)
- 缺陷描述：(1)map<vector>大initializer_list导致栈帧过大触发-Wframe-larger-than告警；(2)basic_block_table.h超500行规范上限；(3)reg_base_common.h超2000行含冗余代码
- 修复模式：运行时map<vector>重构为编译期constexpr扁平数据+偏移表；大块数据从.h移到.cpp；删除冗余代码
- 可审查性：中
- 审查规则建议：大型initializer_list应在CI通过-Wframe-larger-than告警拦截；头文件行数应有lint规则检查上限

跳过：
- 81e8f37d: 【YAML】fix ops/ops-nn/common(纯YAML配置classify_rule添加路径条目，无代码逻辑)
- 50ced237: aclnnConvDepthwise2d文档类型错误修正(纯文档修改kernelSize类型描述INT32->INT64)
- aff4487f: fix gemmv2 readme(纯README修改昇腾910_95支持状态)
- 809ef7e4: aclnnapplyadamwv2的step参数(纯文档修改.md参数约束描述)
- 27bdc238: ModulateGrad fix(纯文档修改公式/链接/约束说明)
- 1d933547: fix: redundant code(仅删除一行注释// #include "op_util.h")
- 21ebd05b: 修复act仓fusedmatmul冗余整改(大规模删除重复目录fusedmatmul_act/，代码仓库整理非缺陷修复)

### 8137de26 vendor_name fix
- 根因类别：CMake变量名混淆
- 涉及文件：cmake/variables.cmake
- 缺陷描述：custom vendor包安装路径中使用了`${VENDOR_NAME}`而非`${VENDOR_PACKAGE_NAME}`，两个变量含义不同(VENDOR_NAME是原始名，VENDOR_PACKAGE_NAME是包名)，导致impl安装路径错误
- 修复模式：将IMPL_INSTALL_DIR和IMPL_DYNAMIC_INSTALL_DIR路径中的VENDOR_NAME替换为VENDOR_PACKAGE_NAME
- 可审查性：高
- 审查规则建议：CMake中多个相似命名变量(VENDOR_NAME vs VENDOR_PACKAGE_NAME)需要在review时逐一确认使用场景是否匹配

### 77dd5b52 conv forward quant mode bug fix
- 根因类别：多通路行为不一致/量化模式分派缺陷
- 涉及文件：conv/common/op_kernel/arch35/conv_instr_impl.h
- 缺陷描述：GetQuantPre函数中hif8/fp8输入类型路径无条件返回vector quant模式(VQ)，未区分extendConv2d(支持scalar quant和vector quant)与普通quant conv(只需vector quant)。当extendConv2d使用scalar quant时，错误地选择了VQ模式
- 修复模式：将单一GetQuantPre拆分为GetQuantPreHif8Fp8/GetQuantPreInt32/GetQuantPreFp32三个子函数，每个子函数内部对extendConv2d场景根据runtime quantMode选择正确的quant模式
- 可审查性：中
- 审查规则建议：量化模式分派函数中每条类型路径需验证是否覆盖所有算子变体(extendConv2d/quantConv/fixedPoint)的差异行为

### 357bb5ed fix apply_adam_w_quant
- 根因类别：复制粘贴错误/配置文件section名错误
- 涉及文件：optim/apply_adam_w_quant/op_host/config/ascend910_93/apply_adam_w_quant_simplified_key.ini (2个平台配置)
- 缺陷描述：simplified_key.ini的section名为[AdvanceStep]，应为[ApplyAdamWQuant]。从另一个算子的配置文件复制时未修改section名
- 修复模式：[AdvanceStep]→[ApplyAdamWQuant]
- 可审查性：高
- 审查规则建议：新增算子的配置文件section名必须与算子名一致，review时应检查.ini文件section名是否匹配所在目录的算子名

### 321c4fcd 算子kernel大小校验问题
- 根因类别：参数校验缺失
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_basic_block_tiling.cpp/.h
- 缺陷描述：conv3d_backprop_filter_v2缺少对kernel尺寸(kd/kh/kw)的合法性校验。当kernel尺寸超过输入+padding允许的最大值时，不报错直接执行导致计算结果错误
- 修复模式：新增CheckKernelSize()函数，根据公式`kdMax = floor((di + padF + padB - 1) / dilationD + 1)`计算各维度kernel上界并校验
- 可审查性：高
- 审查规则建议：conv类算子tiling阶段必须校验kernel/stride/dilation/padding组合的合法性，防止超出输入维度范围

### 2942cc8c 修复降低精度计算的场景问题
- 根因类别：算子调用顺序错误/类型转换时序
- 涉及文件：conv/convolution_backward/op_host/op_api/aclnn_convolution_backward.cpp
- 缺陷描述：convolution_backward的needSpecialCast路径中，先Cast到outputTensor->GetDataType()再TransData。当输出类型不是float16时TransData可能失败或精度异常。正确顺序应为Cast(→fp16)→TransData→Cast(→output dtype)
- 修复模式：将CastOnlyForConvBackward的目标类型改为DT_FLOAT16，TransData后再Cast到最终输出类型。同时增加TransData返回值null检查
- 可审查性：中
- 审查规则建议：数据搬运流水线中多步类型转换的顺序需逐步验证中间类型是否满足下游算子的输入约束

### 26abb542 norm compile warning fix
- 根因类别：编译告警(未使用参数 + signed/unsigned比较)
- 涉及文件：index/gather_v2/op_host/op_api/aclnn_embedding_renorm.cpp, norm/renorm/op_host/op_api/renorm.cpp
- 缺陷描述：(1)CheckParams声明了maxNorm参数但未使用；(2)RenormInferShape中`dim >= real_dim_num`为signed/unsigned比较，dim为int64_t而real_dim_num为size_t
- 修复模式：(1)删除未使用参数；(2)先检查dim<0再用static_cast<size_t>(dim)比较
- 可审查性：高
- 审查规则建议：signed/unsigned混合比较必须先检查signed值非负再cast比较；函数参数需review是否全部使用

### f1827a03 修复910A未cast问题
- 根因类别：平台条件判断不完整
- 涉及文件：index/embedding_dense_grad_v2/op_host/op_api/aclnn_embedding_dense_backward.cpp
- 缺陷描述：embedding_dense_backward中needCast仅考虑scaleGradByFreq或deterministic条件，但在非910B平台(如910A)上，无论这些条件如何都需要Cast到float32。遗漏平台约束导致910A上精度错误
- 修复模式：增加平台判断：910B/910_93上沿用原条件，其他平台(包括910A)无条件设needCast=true
- 可审查性：中
- 审查规则建议：涉及精度转换的逻辑需审查是否在所有支持平台上行为正确，特别注意新增平台支持时原有条件是否仍成立

### a6360067 解决测试用例unique接口精度问题
- 根因类别：tensor别名/就地操作导致精度错误
- 涉及文件：index/scatter_elements/op_host/op_api/aclnn_unique.cpp, aclnn_unique2.cpp
- 缺陷描述：ScatterElements(sumIdx, sortedIndices, sumIdx, ...)中data和updates使用同一个tensor sumIdx。ScatterElements的scatter操作会修改data，而updates也指向同一内存，导致部分元素读到被修改后的值
- 修复模式：单独AllocTensor分配newData作为ScatterElements的data参数，避免别名
- 可审查性：高
- 审查规则建议：ScatterElements/GatherElements等in-place scatter操作中，data和updates参数不能引用同一tensor

### 9887d805 fix sc for qbmmv3 mdc
- 根因类别：编译告警/代码规范(未使用参数 + 命名规范)
- 涉及文件：matmul/quant_batch_matmul_v3/..., matmul/quant_batch_matmul_v4/... 多文件
- 缺陷描述：(1)变量名DimValueOfMKN违反camelCase规范；(2)CheckShape的scaleShape/offsetShape参数传入但未使用；(3)CheckX2TableShape的x2TableShape参数未使用(数据从成员变量获取)
- 修复模式：重命名变量；删除多余参数；清理调用点
- 可审查性：高
- 审查规则建议：静态分析工具应检测未使用的函数参数(-Wunused-parameter)

### 58f1502e 修复foreach
- 根因类别：CMake构建逻辑顺序错误
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：binary.json的存在性检查在compile_from_config之后执行，但实际应在op_type判断之前执行。当算子有自定义binary.json时不需要走get_op_type_from_op_name和check_op_supported逻辑，否则会错误跳过该算子
- 修复模式：将binary.json检查提前到foreach循环开头，存在binary.json时直接使用，否则走原有op_type获取和supported检查流程
- 可审查性：中
- 审查规则建议：CMake构建脚本中条件判断的执行顺序需确保early-exit路径在正确位置

### 3fcda4f9 修复ci出包引用错同名头文件的问题
- 根因类别：include路径歧义/同名头文件冲突
- 涉及文件：conv/convolution_forward/op_host/op_api/aclnn_convolution.cpp
- 缺陷描述：`#include "matmul/common/op_host/op_api/matmul_util.h"`在CI出包时因include搜索路径中存在另一个同名matmul_util.h，导致引用到错误的头文件
- 修复模式：改用相对路径`../../../../matmul/common/op_host/op_api/matmul_util.h`消除歧义
- 可审查性：中
- 审查规则建议：跨模块include应使用显式相对路径或namespace前缀目录，避免同名头文件在不同include路径中产生歧义

### 3721c17f fix_conv3d_no_innerbatch
- 根因类别：条件判断遗漏(format维度)
- 涉及文件：conv/common/op_host/op_tiling/arch35/conv_api_tiling_algorithm_Mmode.cpp
- 缺陷描述：CalFormulaicInnerBatch函数的early return条件只检查了isC04Flag、groups>1、isDmaFlag，未检查3D卷积格式(NCDHW/NDHWC)。conv3d场景不应使用inner batch优化(仅适用于2D)，遗漏导致错误切分
- 修复模式：在early return条件中增加NCDHW和NDHWC格式检查
- 可审查性：高
- 审查规则建议：tiling优化策略的适用条件必须明确排除不支持的format/维度场景

### 01b268c9 batchNormGrad迁移遗漏
- 根因类别：日志格式化错误/迁移不完整
- 涉及文件：norm/batch_norm_grad_v3/op_host/batch_norm_grad_v3_tiling.cpp
- 缺陷描述：dtype校验的错误日志使用`%d`格式打印enum值(如`actual %d`, dxDtype)，不可读且遗漏了对比的另一个dtype。迁移到新API时未同步更新日志格式
- 修复模式：改用`Ops::Base::ToString(dtype).c_str()`输出可读的dtype名称，并同时打印两个不匹配的dtype值
- 可审查性：高
- 审查规则建议：错误日志中的enum/类型值应使用ToString转换为可读字符串；类型不匹配错误应同时打印expected和actual两个值

### d9448715 fix mixType error
- 根因类别：tiling变量计算遗漏(条件分支)
- 涉及文件：norm/rms_norm/op_host/rms_norm_tiling.cpp
- 缺陷描述：rms_norm的mixType路径中计算了blockFactor和useCoreNum但遗漏了latsBlockFactor(lastBlockFactor)的计算。最后一个核的行数因使用未初始化/上一路径残留的值而错误
- 修复模式：在SetBlockDim后增加`latsBlockFactor = numRow - blockFactor * (useCoreNum - 1)`
- 可审查性：高
- 审查规则建议：多核tiling中blockFactor和lastBlockFactor必须成对计算；每个分支路径都需独立设置完整的tiling参数

### cd81d431 修复代码问题
- 根因类别：static局部变量在运行时状态依赖场景中错误使用
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_host/op_tiling/arch35/weight_quant_batch_matmul_v2_adaptive_split_tiling.cpp
- 缺陷描述：(1)`static const uint64_t BLOCK_DIM_M_MAX`依赖运行时值compileInfoPtr_->aicNum，但static只初始化一次，后续调用使用首次的值；(2)`static const std::vector<uint64_t> L0_BASE_K_LIST`依赖运行时标志weightMxFp4Flag_和nzSceneFlag_，同样只初始化一次
- 修复模式：去掉static关键字，改为普通局部变量
- 可审查性：高
- 审查规则建议：依赖运行时状态(成员变量/参数)初始化的局部变量禁止使用static修饰，否则多次调用共享首次初始化值

### 7bc3d081 fix addRmsNorm
- 根因类别：硬件事件ID生命周期管理错误
- 涉及文件：norm/add_rms_norm/op_kernel/add_rms_norm_single_n.h
- 缺陷描述：使用FetchEventID获取事件ID后未Release，导致事件ID耗尽或复用冲突。在bf16路径中MTE2_V事件被多次Fetch但从不Release
- 修复模式：将关键路径的FetchEventID改为AllocEventID，使用后调用ReleaseEventID归还。变量重命名以区分不同生命周期的事件
- 可审查性：中
- 审查规则建议：AllocEventID/ReleaseEventID必须配对使用(类似malloc/free)；review时检查每个Alloc是否有对应Release

### 59a561e5 修复aten直调功能使得npu行为跟cpu一致
- 根因类别：空指针解引用/outputMask未检查
- 涉及文件：conv/convolution_backward/op_host/op_api/aclnn_convolution_backward.cpp, convolution_backward_checker.cpp
- 缺陷描述：(1)PreConv1DBackwardTo2D无条件对gradInput/gradWeight调用View4dWithGroups，但outputMask可能指示它们不需要计算(为null)；(2)InterceptConvFor8bit中对gradInput/gradWeight调用GetDataType但未检查null；(3)CheckDtypeValidForBpFilter8bit中对gradBias调用GetDataType但可能为null
- 修复模式：(1)用outputMask[0]/[1]守护gradInput/gradWeight的View4d调用；(2)InterceptConvFor8bit增加null检查；(3)gradBias先判null再取DataType
- 可审查性：高
- 审查规则建议：可选输出tensor(由outputMask控制)的任何访问前必须检查null或对应mask位

### 328daa2f layerNormGrad精度修复
- 根因类别：ReduceSum的src与tmp buffer别名导致数据污染
- 涉及文件：norm/layer_norm_grad_v3/op_kernel/layer_norm_grad_v3_single_read.h, layer_norm_grad_v3_transpose.h, layer_norm_grad_v3_workspace.h
- 缺陷描述：ReduceSum(dst, src, tmp, count)中src和tmp使用同一tensor。ReduceSum实现会将中间结果写入tmp区域，覆盖了src的原始数据，导致归约结果错误(精度问题)
- 修复模式：为ReduceSum分配独立的tmp buffer(通过queue.AllocTensor或额外参数传入)，修改ReduceMeanModeOne/DoLastAxisReduce签名增加tmp参数
- 可审查性：高
- 审查规则建议：ReduceSum/ReduceMax等归约操作的src和tmp参数禁止指向同一tensor；review时检查归约调用是否存在buffer别名

跳过：
- 4ac05898: fix chmod(shell脚本权限修改，非代码)
- 3036f6d9: fix AdaMaxPool2\3d aclnnMD(纯文档markdown修改)
- f6d85bd0: fix repackage(构建打包脚本增加filelist.csv复制)
- 5f563eff: 回退swigluquant手写aclnn接口(行政性代码回退，删除659行)
- 2b00825d: revert//addRmsNorm Change(行政性revert，14文件大规模回退)
- ae903f83: 修复子包卡死问题(example代码注释化，配合addRmsNorm回退)
- 5c29093a: 新增extendConvTranspose ut用例(UT新增+CodeCheck修复，非缺陷)

### 5df40a02 修复matmulv3 BL1全载模板tiling卡死问题
- 根因类别：循环终止条件缺失
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp
- 缺陷描述：DoBL1FullLoadTilingBase()中while循环不断将baseM减半以使loadSize适配L1，但没有下界保护。当B矩阵本身就超过L1容量时，baseM会被无限减半趋近于0但永远无法使loadSize <= l1Size，导致死循环(tiling卡死)。
- 修复模式：添加前置guard check，即使baseM取最小值16仍超L1则直接返回false放弃fullLoad路径
- 可审查性：中
- 审查规则建议：包含x=x/2或x>>=1的while循环必须有明确的下界终止条件；对缩减变量的循环需检查是否存在不可满足的退出条件

### cbcf639d fix compile error in A16W8 kn A16W4 Nz
- 根因类别：复制粘贴错误(变量名+stride参数)
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/act/prologue/block/block_prologue_b_antiquant_scmc_nd_kn.h, block_prologue_b_antiquant_scmc_nd_nk_nz_kn.h
- 缺陷描述：(1) A16W8 ND kn场景中CeilDiv参数使用了antiQuantNOffset/antiQuantKOffset，但该代码路径中正确变量名是nOffset/kOffset——从另一场景复制代码时未替换变量名。(2) A16W4 NZ场景中stride设置用了复杂动态计算表达式，实际NZ格式kn布局stride应是固定的MakeStride(_128{}, _1{})。
- 修复模式：修正变量名引用；修正stride常量值
- 可审查性：高
- 审查规则建议：新增kernel代码路径应有编译验证用例；copy-paste代码段需逐个检查变量名是否适配新上下文

### ab87ed7b ascend_quant_v2算子修复上边界error
- 根因类别：整数类型溢出(int32用于int64场景)
- 涉及文件：quant/ascend_quant_v2/op_kernel/ascend_quant_v2_nz.h
- 缺陷描述：kernel中多个变量(N, blockIdx, needCoreNum, 循环变量kloop/nloop/offset_n/offset_k等)声明为int32_t，但参与的运算(如K*i*64, 16*16*needCoreNum*j)结果可能超过int32上限，导致整数溢出。当输入tensor维度较大时offset计算错误，产生越界访问。
- 修复模式：扩大整数类型宽度(int32 -> int64)
- 可审查性：高
- 审查规则建议：kernel代码中参与地址/偏移量计算的变量应统一使用int64_t；对int32_t变量参与乘法运算的表达式应检查溢出可能

### 5ccfc384 fix SparseSoftmaxCrossEntropyWithLogits bug
- 根因类别：SIMT资源配置参数不当(LAUNCH_BOUND过大)
- 涉及文件：loss/sparse_softmax_cross_entropy_with_logits/op_kernel/arch35/sparse_softmax_cross_entropy_with_logits_full_load.h, sparse_softmax_cross_entropy_with_logits_split_r.h
- 缺陷描述：SIMT kernel的LAUNCH_BOUND从2048改为1024。LAUNCH_BOUND控制并发线程数上限，设为2048时每个线程分配到的栈/寄存器资源减半，可能导致栈溢出或寄存器spilling引发运行时错误。
- 修复模式：降低并发线程数以保证每线程资源充足
- 可审查性：低
- 审查规则建议：SIMT kernel的LAUNCH_BOUND值应有资源计算依据(栈大小/寄存器数量)，不应随意设置

### 5a2316b5 fix multi_add_rms_norm_dynamic_quant safeDiv &+
- 根因类别：复合缺陷(copy-paste + SafeDiv符号逻辑 + 空指针防护缺失)
- 涉及文件：norm/multi_add_rms_norm_dynamic_quant/op_host/multi_add_rms_norm_dynamic_quant_infershape.cpp, multi_add_rms_norm_dynamic_quant_tiling.cpp
- 缺陷描述：(1) infershape中获取yShape后空指针检查写成了OP_CHECK_NULL_WITH_CONTEXT(context, xShape)——检查的是xShape而非yShape，yShape为空时不会被拦截。(2) SafeDiv当除数b为负数且绝对值小于EPSINON时将b替换为正的EPSINON，导致除法结果符号反转。(3) epsPtr为nullptr时静默跳过不设置eps，导致eps使用未初始化值。
- 修复模式：修正copy-paste变量名；修正SafeDiv符号逻辑(b<0时用-EPSINON)；增加空指针防护
- 可审查性：高
- 审查规则建议：OP_CHECK_NULL参数应与前一行获取的变量名一致；SafeDiv需处理负数near-zero场景；可选参数nullptr分支应有显式fallback或报错

### d48e63bf bugfix
- 根因类别：缺少namespace声明
- 涉及文件：optim/apply_ftrl/op_kernel/apply_ftrl_apt.cpp
- 缺陷描述：文件使用ApplyFtrlOpTiling命名空间中的类型/常量，但缺少using namespace ApplyFtrlOpTiling声明，导致编译时找不到符号。新增tiling数据结构后忘记在kernel文件中引入对应命名空间。
- 修复模式：添加缺失的using namespace声明
- 可审查性：高
- 审查规则建议：kernel cpp文件中引用的tiling结构体所在namespace必须显式声明；CI编译应覆盖所有kernel目标

### ce6faf51 回退公共方法导致算子用例执行失败问题
- 根因类别：API兼容性/平台适配不完整
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/tool.h
- 缺陷描述：将ASCENDC_ASSERT(大写宏)整改为ascendc_assert(小写函数)，但ascendc_assert在部分芯片上不支持，导致算子用例执行失败。未验证全部目标芯片的API可用性。
- 修复模式：API调用回退为ASCENDC_ASSERT宏形式
- 可审查性：中
- 审查规则建议：涉及公共工具函数的API变更须在所有支持芯片平台上验证兼容性

### ada7984a support noalign and fix k>65535 kernel error
- 根因类别：同名函数行为不一致导致整数溢出
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/act/下多个文件
- 缺陷描述：K维度>65535时kernel出现AIC ERROR和TASK Timeout。代码中AscendC::CeilDiv/CeilAlign与Act::CeilDiv/CeilAlign的实现不同——处理大数值(uint64_t)时前者存在隐式类型截断或溢出。
- 修复模式：统一使用Act命名空间下的正确实现，部分调用处加static_cast<uint64_t>
- 可审查性：中
- 审查规则建议：同一项目中不应存在同名但行为不同的工具函数；维度参数运算需检查是否支持超过16位整数范围

### 75468d69 fix aclnn_avgpool2d_backward
- 根因类别：分支逻辑缺失(维度处理不完整)
- 涉及文件：pooling/avg_pool3_d_grad/op_host/op_api/aclnn_avgpool2d_backward.cpp
- 缺陷描述：获取cDims时只处理gradOutput维度为kNCHWDIM(4维)情况，缺少else分支。3维输入(NCL)时cDims保持初始值1而非从正确维度索引(kCDimNCLIdx)获取。
- 修复模式：补全else分支覆盖3维输入场景
- 可审查性：高
- 审查规则建议：条件分支处理多种shape格式时确保所有维度情况都有对应逻辑；对有默认值的变量检查是否存在遗漏的赋值路径

### a881847a kv_rms_norm_rope_cache修复host
- 根因类别：复合缺陷(量化模式判断逻辑错误 + 空指针检查缺失 + copy-paste日志变量名)
- 涉及文件：norm/kv_rms_norm_rope_cache/op_host/下多个tiling文件
- 缺陷描述：(1) GetQuantMode用scale和offset组合区分量化模式，逻辑过于复杂且存在错误(offset为空但scale非空时返回-1而非有效模式)。(2) gammaShapePtr缺少空指针检查。(3) 错误日志中k_rope_offset误写为k_rope_scale、c_kv_offset误写为c_kv_scale。
- 修复模式：简化量化模式枚举(合并为QUANT_MODE)，增加空指针校验，修正日志
- 可审查性：高
- 审查规则建议：可选输入的组合判断应有明确状态机定义；GetInputShape返回值使用前必须做空指针检查；错误日志变量名应与实际校验变量一致

### 9fd32a44 fix adaptiveavgpool3dgrad bug
- 根因类别：复制粘贴错误(input/output buffer搞混)
- 涉及文件：pooling/adaptive_avg_pool3d_grad/op_kernel/adaptive_avg_pool3d_grad_cast.h
- 缺陷描述：atomicAdd模式下清零workspace时使用了inputLocalFloat作为清零源和DMA拷贝源，应为outputLocalFloat。inputLocalFloat是输入tensor的local buffer，用它做清零源会导致输入数据被破坏且清零值不正确。
- 修复模式：将inputLocalFloat替换为outputLocalFloat
- 可审查性：高
- 审查规则建议：清零操作的目标buffer应与后续写回的目标buffer一致；同一函数中多个相似命名buffer需特别关注使用正确性

### 811e50f8 修复deepseek-r1主线dsq执行报错问题
- 根因类别：循环控制逻辑错误(break位置不当)
- 涉及文件：quant/dequant_swiglu_quant/op_kernel/dequant_swiglu_quant_cut_group.h
- 缺陷描述：group切分循环中，realDimx_<=0时的break逻辑放在if(groupIdx==cuGroupIdx)分支内部。当中间存在空group但还未轮到当前core处理时循环不会退出，导致后续使用非法的groupOffset_/realDimx_值。
- 修复模式：将终止条件从嵌套分支内提升到循环顶层
- 可审查性：高
- 审查规则建议：循环中的提前退出条件应在循环体最早位置检查，不应被嵌套在无关条件分支内；多core并行遍历时空数据的跳过/终止逻辑应独立于core分配逻辑

### 7f3cb998 fix accuracy
- 根因类别：数学公式实现错误(操作数搞混)
- 涉及文件：norm/deep_norm_grad/op_kernel/deep_norm_grad_cut_d.h, deep_norm_grad_merge_n.h
- 缺陷描述：DeepNorm反向传播梯度计算中多处向量运算的source/destination tensor角色混淆。计算dvar时使用了inputGamma(dy*gamma)而非inputGx(x_hp-mean)；dgx和dgamma计算中也有操作数组合顺序错误。
- 修复模式：重新梳理数学推导步骤，修正每步向量运算的输入/输出tensor指向
- 可审查性：低
- 审查规则建议：复杂数学公式的kernel实现应附带伪代码或公式注释；精度相关改动需有参考实现(如PyTorch)做对比验证

### 113df1fc fix ut
- 根因类别：CMake include路径过宽
- 涉及文件：cmake/ut.cmake
- 缺陷描述：UT编译的include目录使用项目根目录(PROJECT_SOURCE_DIR/)，过于宽泛可能导致头文件解析到错误同名文件。
- 修复模式：将include路径精确化到具体的tests/ut/common目录
- 可审查性：高
- 审查规则建议：CMake的target_include_directories不应将项目根目录加入include搜索路径，应指定到具体子目录

### f27cf969 修复问题单dts2025101722772
- 根因类别：复合缺陷(InferShape平台特定逻辑缺失 + kernel流水线同步缺失)
- 涉及文件：quant/trans_quant_param_v2/op_host/trans_quant_param_v2_infershape.cpp, quant/trans_quant_param_v2/op_kernel/trans_quant_param_v2.h
- 缺陷描述：(1) InferShape输出shape推导缺少芯片型号(Ascend910_95)特定的分支逻辑，当offset dim(0) > scale dim(0)时应取offset shape。(2) Kernel中多处GM到UB的数据搬运与计算之间缺少PipeBarrier<PIPE_ALL>同步屏障，数据搬运未完成就开始计算。
- 修复模式：增加芯片判断的InferShape分支；插入流水线屏障
- 可审查性：中
- 审查规则建议：InferShape需覆盖所有目标芯片的shape推导规则；GM/UB间数据搬运必须在使用数据前设置流水线同步屏障

### 95d3488d fix adaavgpool2dbackward
- 根因类别：输入校验不完整
- 涉及文件：pooling/adaptive_avg_pool3d_grad/op_host/op_api/aclnn_adaptive_avg_pool2d_backward.cpp
- 缺陷描述：CheckInputOutputShape只校验out和self的batch/channel维度一致性，遗漏gradOutput的维度校验。gradOutput前几维与self不匹配时不报错直接进入计算。
- 修复模式：扩展校验函数参数和范围，增加gradOutput的shape一致性检查
- 可审查性：高
- 审查规则建议：backward算子的输入校验必须覆盖grad tensor的shape与前向输入的一致性

---

跳过的提交(第310-329批)：
- 567eb75b: fix aclnn资料(纯文档修改)
- 12d88c6c: 修复卸载脚本问题(shell脚本路径修复)
- 9b5fdc31: 修复静态库出包(打包/命名空间符号导出)
- 31b74f3a: 修复子包安装问题(安装脚本/打包配置)

## 第330-349批 (2025-11-24 ~ 2025-11-26)

### 930d93a8 【FIX】AdaLayerNormV2
- 根因类别：条件判断逻辑错误
- 涉及文件：norm/ada_layer_norm/op_host/ada_layer_norm_tiling.cpp
- 缺陷描述：用OR逻辑判断isWeightFloat，weight/bias任一为float就设true，但当weight不存在而bias是float时会误判。修复改为逐个检查存在且非float时显式置false
- 修复模式：复合OR条件拆分为独立if块
- 可审查性：中
- 审查规则建议：对含多个可选输入(optional tensor)的布尔条件判断进行review，确保null情况下逻辑正确

### 76593c3a revert//ascendc_assert整改
- 根因类别：API名称大小写不兼容
- 涉及文件：多个matmul相关kernel头文件
- 缺陷描述：之前将ASCENDC_ASSERT改为ascendc_assert（小写），但小写版本与当前SDK不兼容，需回退为大写
- 修复模式：批量文本替换ascendc_assert -> ASCENDC_ASSERT
- 可审查性：高
- 审查规则建议：API名称变更前应确认目标版本兼容性

### 4ee704f8 修复arch35 qbmmv4 tiling base类中tilingData赋值行为
- 根因类别：API误用(序列化方法)
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/op_tiling/arch35/quant_batch_matmul_v4_tiling.cpp
- 缺陷描述：SaveToBuffer()行为不符合预期（可能只拷贝部分数据或格式不匹配），修复改为memcpy_s直接拷贝struct原始内存并加EOK检查
- 修复模式：高层API替换为底层安全内存拷贝
- 可审查性：中
- 审查规则建议：tiling数据序列化应有统一规范，审查所有SaveToBuffer调用点确认语义

### 4e794227 修复910B不支持的算子编译问题
- 根因类别：构建配置错误(错误级别过高)
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：compute_unit无对应算子时用FATAL_ERROR中断编译，实际应为WARNING跳过
- 修复模式：FATAL_ERROR降级为WARNING
- 可审查性：高
- 审查规则建议：CMake中FATAL_ERROR应仅用于不可恢复错误，可选组件缺失用WARNING

### 362fed23 fix avgpool2d
- 根因类别：分支选择缺陷(维度边界)
- 涉及文件：pooling/avg_pool3_d/op_host/op_api/aclnn_avgpool2d.cpp
- 缺陷描述：C维度>=65536时仍走AvgPool(Cube)分支，但该分支不支持大C维度。修复增加cDims<cMaxDims条件约束
- 修复模式：增加维度阈值边界检查
- 可审查性：中
- 审查规则建议：算子分支选择逻辑应对所有维度参数做范围校验

### 29ea53b8 fix wrong var issue
- 根因类别：变量名拼写错误
- 涉及文件：scripts/package/ops_nn/scripts/ops_nn_custom_install.sh
- 缺陷描述：$src_kernel_path误写为$ops_kernel_path（不存在的变量），glob匹配失败
- 修复模式：变量名修正
- 可审查性：高
- 审查规则建议：shell脚本应开启set -u；用shellcheck捕获未定义变量引用

### d9034ea4 maskedsoftmaxwithrelposbias算子修正tiling逻辑错误
- 根因类别：整数溢出 + 索引常量错误 + 零值检查缺陷
- 涉及文件：norm/masked_softmax_with_rel_pos_bias/op_host/masked_softmax_with_rel_pos_bias_tiling.cpp
- 缺陷描述：三类问题：(1)多个size变量用uint32_t，大shape溢出；(2)输出shape获取用了输入索引X_INPUT_INDEX(=0)而非Y_OUTPUT_INDEX，且Y_OUTPUT_INDEX定义也错(3应为0)；(3)零值检查b_*w_*n_*s1_*s2_==0乘积本身也可能溢出
- 修复模式：类型提升uint32->uint64、索引常量修正、零值判断拆分为逐项检查
- 可审查性：中
- 审查规则建议：tiling计算中尺寸变量应默认uint64_t；零值检查应逐项而非乘积

### d84cb711 modulategrad fix
- 根因类别：API误用(可选输入检测方法)
- 涉及文件：vfusion/modulate_grad/op_host/modulate_grad_tiling.cpp
- 缺陷描述：通过GetOptionalInputShape()返回值是否为null判断可选输入是否存在，但shape可能在输入不存在时仍返回非null。修复改用GetOptionalInputTensor()检查tensor本身
- 修复模式：shape存在性检查替换为tensor存在性检查
- 可审查性：高
- 审查规则建议：可选输入存在性判断应统一使用GetOptionalInputTensor()!=nullptr

### c1262c16 topKtopPSample selLogitsOut sync fix
- 根因类别：初始值错误 + 多核同步缺陷
- 涉及文件：index/top_k_top_p_sample/op_kernel/top_k_top_p_sample.h
- 缺陷描述：(1)logitsTopKPSelect输出默认值初始化为0.0，应为-inf；(2)InitGlobalMemory写入GM后缺少MTE3->MTE2同步，后续读取可能拿到脏数据
- 修复模式：初始值常量修正 + 增加硬件同步屏障
- 可审查性：低
- 审查规则建议：GM初始化后必须有对应同步屏障；输出tensor默认值应与文档约定一致

### b74a1dce 修复ACT代码名称错误
- 根因类别：标识符拼写错误
- 涉及文件：common/act/matmul/kernel/kernel_qgmm_inplace_add.h等
- 缺陷描述：类名KernelQGmmInpaceAdd中Inpace是Inplace的拼写错误，涉及类名/构造函数/析构函数批量修正
- 修复模式：标识符重命名Inpace->Inplace
- 可审查性：高
- 审查规则建议：新增标识符应做拼写检查，可集成cspell到CI

### 9ee6a9ef fix aclnn
- 根因类别：复制粘贴错误(注释)
- 涉及文件：多个pooling算子API头文件
- 缺陷描述：doxygen注释中引用了错误的函数名（如MaxPool2D应为AdaptiveMaxPool2d），copy-paste导致
- 修复模式：修正注释中的函数名引用
- 可审查性：高
- 审查规则建议：API头文件注释中函数名引用应与实际声明一致

### 86c08aea fix custom pkg issue
- 根因类别：构建配置错误(安装路径)
- 涉及文件：cmake/gen_ops_info.cmake
- 缺陷描述：ENABLE_CUSTOM模式下安装路径硬编码附加了ops_nn子目录，导致自定义算子包文件安装到错误位置
- 修复模式：增加ENABLE_CUSTOM条件分支，自定义模式下子目录设为空
- 可审查性：高
- 审查规则建议：install路径应支持可配置子目录，自定义包构建需独立集成测试

### 68c878cf BatchNormGradV3算子精度问题修改
- 根因类别：参数传递错误(值传递应为指针传递)
- 涉及文件：norm/batch_norm_grad_v3/op_host/op_api/aclnn_fast_batch_norm_backward.cpp
- 缺陷描述：函数接收const aclTensor*参数并在内部Transpose()后重新赋值，但值传递导致调用者指针未更新，后续计算使用未转置tensor，造成精度问题
- 修复模式：改为const aclTensor**（二级指针）实现出参语义
- 可审查性：高
- 审查规则建议：函数若需修改调用者的指针变量必须使用二级指针或引用

### 16168cae bug fix (dequant_swiglu_quant)
- 根因类别：数据类型对齐计算错误
- 涉及文件：quant/dequant_swiglu_quant/op_kernel/dequant_swiglu_quant_static_base.hpp等
- 缺陷描述：BiasType为bfloat16/half时，偏移量计算使用了int8_t对齐的alignColNum，但不同类型对齐要求不同，导致DataCopyPad偏移错误
- 修复模式：引入biasAlignColNum按BiasType字节宽度独立计算对齐
- 可审查性：低
- 审查规则建议：混合精度数据搬运中对齐偏移量必须按实际数据类型计算

### 1257005b fix sc修复编译告警问题
- 根因类别：预处理宏未定义导致告警
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/arch35/quant_batch_matmul_v3_apt_tiling_key.h
- 缺陷描述：#if条件中直接使用ORIG_DTYPE_SCALE==DT_FLOAT，宏未定义时预处理器视为0可能意外为true
- 修复模式：增加defined()前置检查
- 可审查性：高
- 审查规则建议：#if中使用自定义宏前必须先用defined()检查；开启-Wundef编译选项

### f9c776ed fix 95 kernel build
- 根因类别：构建脚本错误(exit vs return + 缓存失效)
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op.sh, gen_opcinfo_for_socversion.sh
- 缺陷描述：(1)单算子生成失败时exit 1中断整个进程，应为return跳过；(2)opcinfo文件存在时exit 0跳过但不处理缓存过期
- 修复模式：exit 1改为return + 删除错误的缓存跳过逻辑
- 可审查性：高
- 审查规则建议：构建脚本中单组件失败不应中断整体流程，用return而非exit

### f7c3a307 修改 index_select ut 错误
- 根因类别：UT代码不兼容(废弃API调用)
- 涉及文件：index/gather_v2/tests/ut/op_host/test_aclnn_embedding.cpp等
- 缺陷描述：UT调用TestPrecision()方法在当前框架版本不可用；废弃的test_aclnn_gather_v3.cpp未清理；build.sh缺少return
- 修复模式：删除不兼容API调用、删除废弃文件、增加提前返回
- 可审查性：高
- 审查规则建议：UT框架升级后应CI自动验证所有UT编译通过

---

跳过的提交(第330-349批)：
- 67829d07: fix aclnnConvTbcBackward introduction(纯文档/数学公式修正)
- cf468aea: fix readme(文档修复)
- c8883dc4: fix aclnn ziliao(文档修复)

---

### c94f6927d9a41cdf23ebbe3e4d9420a097b1c7d5 fix api
- 根因类别：构建脚本变量未初始化
- 涉及文件：build.sh
- 缺陷描述：build_ut函数中enable_cov变量在CI模式循环内使用但未在循环前初始化，导致后续coverage收集逻辑行为不确定
- 修复模式：在循环前添加`local enable_cov=FALSE`初始化，循环内匹配时设为TRUE
- 可审查性：高
- 审查规则建议：shell脚本中条件分支内使用的标志变量应在分支外先初始化默认值

### b529184f7abfa0859a6c3ce26c7fc753db5545be fix rmsNorm
- 根因类别：流水线同步事件类型错误
- 涉及文件：norm/rms_norm/op_kernel/rms_norm_merge_n.h
- 缺陷描述：DataCopyCustom从GM→Local使用MTE2通道，但SetFlag/WaitFlag错误使用了HardEvent::MTE3_S（MTE3是Local→GM方向）。gamma数据搬入后用错误事件同步，可能导致Cast操作读到未搬运完的数据
- 修复模式：HardEvent::MTE3_S → HardEvent::MTE2_S
- 可审查性：高
- 审查规则建议：SetFlag/WaitFlag的HardEvent类型必须与实际DMA搬运方向一致：MTE2=GM→Local，MTE3=Local→GM

### b065c36c89707ad4fb54cf9be12e101b24e18036 fix L0C verification in AdjustSmallCaseBaseBlock
- 根因类别：资源溢出校验缺失
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_stream_k_tiling.cpp
- 缺陷描述：AdjustSmallCaseBaseBlock为提升核利用率尝试扩大blockBaseM，但循环中未校验调整后的singleShapeM * blockBaseN * L0C_DTYPE_BYTE是否超出L0C_SIZE。扩大后L0C溢出导致计算错误。同时dbL0C仅在条件满足时设为DB_ON，不满足时缺少显式设为DB_OFF
- 修复模式：循环中增加L0C溢出检查并break；else分支显式设dbL0C=DB_OFF
- 可审查性：高
- 审查规则建议：调整tiling参数后必须重新校验所有相关buffer(L0A/L0B/L0C/UB)是否溢出；双buffer标志应有显式的else分支

### 8ece91366f7dfb28800b5065bc2bccd6893740da fix adaptiveavgpool2d bug
- 根因类别：API访问方式错误
- 涉及文件：pooling/adaptive_avg_pool3d/op_host/op_api/aclnn_adaptive_avg_pool2d.cpp
- 缺陷描述：使用outputShape[i]和inputShape[i]通过下标运算符访问shape维度，而正确的API是GetDim(i)。下标运算符可能返回错误结果或编译通过但语义不同
- 修复模式：outputShape[i] → outputShape.GetDim(i)，inputShape[i] → inputShape.GetDim(i)
- 可审查性：高
- 审查规则建议：shape对象维度访问应统一使用GetDim()方法，禁止使用operator[]

### 65fef269563dbf822f2a0fc12a51e0b34056fa20 修改review意见，修改变量未初始化红线问题
- 根因类别：变量初始化顺序/静态分析告警
- 涉及文件：common/act/matmul/kernel/kernel_qbmm_pertile.h
- 缺陷描述：QuantMmBatchPertile的ProcessMainLoop中cacheID = GetCacheID(i)声明在VFBinaryReduceSumWithoutTail之后，静态分析工具标记为潜在未初始化风险。同时重构ProcessSingleBatch拆分为ProcessWithoutBatch和ProcessWithBatch两个函数，分离batch=1和多batch逻辑
- 修复模式：将cacheID声明移至函数体前部（CopyInputsToUB之前），确保在所有使用路径上已初始化
- 可审查性：中
- 审查规则建议：变量声明应尽早在使用前定义，避免跨越复杂流水线操作导致静态分析误报

### 163f7220a1e817bb733cc4563246f787cc624d4e 安全扫描问题修改
- 根因类别：安全扫描多类问题(类型双关/整数溢出/数组越界)
- 涉及文件：norm/deep_norm/op_host/deep_norm_tiling.cpp, norm/rms_norm/op_host/rms_norm_tiling.cpp, norm/rms_norm_grad/op_host/rms_norm_grad_tiling.cpp, norm/batch_norm_grad_v3/tests/ut/op_kernel/test_batch_norm_grad_v3.cpp
- 缺陷描述：(1) deep_norm中reinterpret_cast<uint32_t*>(&eps)进行float→uint32类型双关违反strict aliasing规则；(2) FindPowerTwo参数类型int32_t无法处理大于2^31的值，且缺少64位移位；(3) rms_norm_grad中未校验dyDimNum >= gammaDimNum就用差值做索引可能越界；(4) GetShapeSize非inline导致多编译单元ODR违规
- 修复模式：(1) 改用memcpy_s安全转换；(2) 参数改uint64_t并增加n|=n>>32；(3) 增加前置校验dyDimNum > gammaDimNum；(4) 添加inline修饰
- 可审查性：高
- 审查规则建议：(1) 禁止reinterpret_cast做类型双关，用memcpy_s替代 (2) 位运算工具函数参数类型应匹配实际使用场景的值域 (3) 用维度差做数组索引前必须校验不会为负

### ee7a3e343929b21498b4983be3e9473e0cde4c7a fix ci
- 根因类别：构建脚本路径查找不完整
- 涉及文件：build.sh, activation/clipped_swiglu/tests/ut/op_host/test_clipped_swiglu_tiling.cpp
- 缺陷描述：build_single_example中检查libascendcl.so只查找EAGER_LIBRARY_PATH，未查找EAGER_LIBRARY_OPP_PATH备选路径，导致OPP部署场景下编译失败
- 修复模式：添加`|| [ -f ${EAGER_LIBRARY_OPP_PATH}/libascendcl.so]`条件（注：原修复中`]`前缺空格可能引入新问题）
- 可审查性：高
- 审查规则建议：库文件路径查找应覆盖所有部署场景的可能路径；shell test语句`]`前必须有空格

### 21f78d5a0cac90dba72b15b88a5adede374afc1a fix kernel error
- 根因类别：构建脚本错误传播缺失
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op.sh, scripts/kernel/binary_script/build_binary_single_op_gen_task.sh
- 缺陷描述：build_binary_single_op_gen_task.sh的多个失败路径exit 0（成功码）或不检查子脚本返回值。OPC信息为空时exit 0导致上层脚本以为成功，binary_config_file不存在时同样静默成功。主脚本build_binary_single_op.sh不检查gen_task子脚本返回值
- 修复模式：各失败路径使用不同exit code(2/3/4/5)区分错误类型；主脚本检查子脚本返回值并报错退出；函数末尾添加显式exit 0
- 可审查性：高
- 审查规则建议：(1) shell脚本中WARNING+exit 0是危险模式，失败路径必须exit非零 (2) 调用子脚本后必须检查$? (3) 不同失败场景应使用不同exit code便于定位

### e68215487077fca2c7b32109ca6d13962ecf7bc8 回退//remove mmv3 bf16 bias compile
- 根因类别：Revert(过早移除配置)
- 涉及文件：matmul/mat_mul_v3/op_host/config/ascend910b/mat_mul_v3_binary.json
- 缺陷描述：之前的MR移除了MatMulV3的bf16 bias编译配置(binary.json中BF16_BF16_BF16_BF16变体)，但该配置仍有使用场景，移除导致对应数据类型组合的MatMul无法binary编译
- 修复模式：Revert整个移除操作，恢复bf16 bias的binary.json配置条目
- 可审查性：中
- 审查规则建议：移除binary.json中的算子变体前，必须确认该变体在所有下游场景中已无调用

### ccfe9770d0d2242a5dc1862b65ba8a2833293158 fix GroupNormSilu Tiling
- 根因类别：整数溢出 + 错误传播缺失
- 涉及文件：norm/group_norm_silu/op_host/group_norm_silu_tiling.cpp
- 缺陷描述：(1) SetBlockTiling中shapeN * numGroups直接相乘可能uint32溢出（两个较大的uint32乘积超出32位范围），导致numPerCore/realCoreNum计算错误；(2) SetProcessSize和SetTilingSD返回void，内部检测到除零等错误时OP_LOGE后return但不返回错误码，调用方无法感知失败继续使用无效tiling
- 修复模式：(1) 显式cast为uint64_t后再乘：uint64_t totalGroups = static_cast<uint64_t>(shapeN) * static_cast<uint64_t>(numGroups)；(2) 返回类型改ge::graphStatus，OP_CHECK_IF失败返回GRAPH_FAILED，调用方检查返回值
- 可审查性：高
- 审查规则建议：(1) 两个32位值相乘用于64位变量时必须先提升至少一个操作数为uint64_t (2) tiling辅助函数不应返回void，检测到异常必须通过返回值传播错误

### a4223bfa1b21a999345aca86592e041e4559ed93 解决910编译问题
- 根因类别：平台配置遗漏
- 涉及文件：activation/hard_swish_grad_v2/op_host/hard_swish_grad_v2_def.cpp
- 缺陷描述：HardSwishGradV2算子的ascend910配置缺少DynamicCompileStaticFlag/DynamicRankSupportFlag/DynamicShapeSupportFlag三个编译标志，导致910平台上动态shape/rank场景编译失败
- 修复模式：在config910A上添加三个.Flag(true)调用
- 可审查性：高
- 审查规则建议：新增算子或新增平台配置时，对照同类算子检查是否遗漏Dynamic*Flag系列编译标志

### 6f5cf633357228244d86c4775528f9772c5b8c9e fix rmsnormgrad bug
- 根因类别：流水线同步屏障缺失 + 变量声明顺序
- 涉及文件：norm/rms_norm_grad/op_kernel/arch35/rms_norm_grad_regbase_dgamma.h
- 缺陷描述：(1) dgamma二分累加Level2完成后，缺少VEC_STORE→VEC_LOAD的LocalMemBar屏障，Level1累加可能读到未完成写入的数据（load等store依赖同步缺失）；(2) cacheID = GetCacheID(i)在两处函数中声明位置过晚，移至函数顶部确保资源ID在流水线操作前获取
- 修复模式：(1) 在Level2循环后添加MicroAPI::LocalMemBar<VEC_STORE, VEC_LOAD>()；(2) cacheID声明移至CopyInputsToUB之前
- 可审查性：中
- 审查规则建议：二分累加/归约操作的不同level之间必须插入对应方向的MemBar；向量Store后Load同一地址必须有VEC_STORE→VEC_LOAD屏障

### 2d6f9cb4996f63280870af28ccfd211afb09c8c5 Revert "[MatMul] modify range of shape to transdata"
- 根因类别：Revert(条件判断过于宽松)
- 涉及文件：matmul/common/op_host/op_api/matmul_util.cpp
- 缺陷描述：原始修改将IsNdToNzOnTheFly中innerAxis的判断简化为仅检查上界(<=65535)，移除了下界和16对齐检查。但transdata要求小于128的innerAxis必须16对齐，否则数据排布错误。Revert后恢复完整判断：>=128时检查上界，<128时检查16对齐
- 修复模式：恢复kInnerAxisMinLimit=128和分段条件：(axis >= 128 && axis <= 65535) || (axis < 128 && (axis & 0xF) == 0)
- 可审查性：中
- 审查规则建议：NdToNz transdata的轴长度需同时满足上界和对齐约束，简化条件判断时不能丢弃对齐检查

### 2463697d9914e180d7105870f100e27b9f89f6ec fix back
- 根因类别：可选输入空指针风险
- 涉及文件：norm/group_norm_silu/op_host/group_norm_silu_tiling_arch35.cpp
- 缺陷描述：CheckMixRegBase中通过GetOptionalInputDesc获取gamma/beta的dtype来判断是否支持regbase，但gamma/beta是可选输入，可能为null。虽然有OP_CHECK_NULL_WITH_CONTEXT，但不同场景下可选输入的存在性不可靠。修改为用GetOutputDesc获取output1/output2的dtype判断，output始终存在
- 修复模式：GetOptionalInputDesc(INPUT_IDX_GAMMA/BETA) → GetOutputDesc(INPUT_IDX_GAMMA/BETA)，变量名从gamma/betaDtype改为out1/out2Dtype
- 可审查性：高
- 审查规则建议：dtype兼容性判断优先使用必选张量(output)而非可选张量(optional input)，避免null风险

### 14a737503ff9b9ad98a8019cecc3799c9f42e3fd MinUsedDim fix from being zero
- 根因类别：除零错误
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp
- 缺陷描述：CalcTile中outerMinUseDim和innerMinUseDim通过aicNum/maxConflictDim计算，当maxConflictDim >= aicNum时整数除法结果为0，后续用这两个值做除数触发除零异常
- 修复模式：std::max(aicNum / maxConflictDim, 1UL)确保最小值为1
- 可审查性：高
- 审查规则建议：整数除法结果用作后续除数时，必须用std::max(..., 1)保护

### 9716a600fadec6cc86efb9f7960a5bb1d1914f28 修复fused_linear_online_max_sum在910_93上偶现确定性计算精度的问题
- 根因类别：多核同步屏障缺失
- 涉及文件：matmul/fused_linear_online_max_sum/op_kernel/fused_linear_online_max_sum.h
- 缺陷描述：CVProcess()（向量后处理）之前缺少SyncAll<false>()全局同步，Matmul结果可能未全部写入L0C/GM就被CVProcess读取，在910_93多核场景下偶现精度错误（确定性计算要求所有核完成后再后处理）
- 修复模式：在CVProcess()前增加SyncAll<false>()全局屏障
- 可审查性：中
- 审查规则建议：Matmul→后处理(CVProcess/Fixpipe)之间必须有全局同步屏障，确保所有核的Cube计算完成

---

跳过的提交(第350-369批)：
- d7740e93: fix kongge(日志字符串尾部空格修复，非代码缺陷)
- cd942771: conv文档拼写错误修改(纯文档)
- 8ffc8129: 修复matmul目录下README低错问题(纯文档)
- 8cc6fd68: fix aclnnziliao(纯文档修复)

## 第370-391批分析

### 7ab2a2cd fix sub pkg compile error
- 根因类别：构建配置缺陷/平台支持遗漏
- 涉及文件：CMakeLists.txt, build.sh, cmake/gen_ops_info.cmake, 新增3个common头文件
- 缺陷描述：ASCEND_ALL_COMPUTE_UNIT列表缺少多个SoC型号(ascend031/035/310b/910_55/mc62cm12a)，导致A5等平台的算子无法编译。同时构建脚本中act_copy目标改名common_copy，增加公共头文件拷贝逻辑解耦编译依赖。
- 修复模式：配置补全+资源拷贝路径修正
- 可审查性：中
- 审查规则建议：新增SoC平台时，检查CMakeLists.txt、build.sh等多处SoC列表是否一致同步

### 5bd40ca6 addLayerNorm上边界问题修复
- 根因类别：整数溢出/类型截断缺陷
- 涉及文件：norm/add_layer_norm/op_host/add_layer_norm_tiling.cpp, add_layer_norm_tiling.h
- 缺陷描述：tiling计算核心变量(numRow/numCol/rowPerTime等)使用uint32_t/int32_t，tensor维度超2^31时溢出。中间计算x1Size=numRow*numCol在int32下易溢出。TILING_DATA_FIELD_DEF字段类型也需同步改int64_t，否则host侧正确但传device时截断。
- 修复模式：系统性类型提升uint32_t/int32_t→int64_t，消除static_cast截断
- 可审查性：高
- 审查规则建议：tiling代码中shape/size变量强制int64_t；检查所有static_cast到窄类型的转换

### 097251e0 bug fix
- 根因类别：多类缺陷(参数校验缺失+编译警告+符号可见性)
- 涉及文件：activation/hard_sigmoid, activation/softshrink_grad, quant/dequant_swiglu_quant
- 缺陷描述：(1)hardsigmoid缺少输入输出dtype/format一致性检查；(2)softshrink_backward对lambd参数过度约束类型检查；(3)dequant_swiglu_quant常量从头文件移到cpp解决链接可见性问题
- 修复模式：增强参数校验+移除过度约束+调整符号作用域
- 可审查性：高
- 审查规则建议：算子API层必须检查输入输出dtype/format一致性；头文件避免定义非inline全局常量(ODR violation)

### 8f0ebc3e 修复DequantSwigluQuant算子在group_index值存在零时出现的精度问题
- 根因类别：控制流逻辑缺陷/零值边界条件处理不当
- 涉及文件：quant/dequant_swiglu_quant/op_kernel/dequant_swiglu_quant_cut_group.h
- 缺陷描述：遍历group时realDimx_<=0直接break跳出整个循环，但group_index存在零值只表示空group不应终止遍历。修复将break移入groupIdx==cuGroupIdx条件内部，区分speGroupType场景。
- 修复模式：边界条件修正，"遇零即停"改为"遇零跳过继续"
- 可审查性：中
- 审查规则建议：循环中break语句应审查是否覆盖"部分数据为零/空"的边界case

### 7bc74e32 修复dynamic_quant算子kernel ut报错
- 根因类别：UT与生产代码不同步/结构体字段缺失
- 涉及文件：quant/dynamic_quant/tests/ut, quant/dynamic_quant_v2/tests/ut
- 缺陷描述：生产代码tiling结构体新增ubSize字段，UT侧结构体未同步更新，内存布局不一致导致UT读取错误tiling参数。
- 修复模式：UT结构体补上ubSize字段
- 可审查性：高
- 审查规则建议：tiling结构体应只定义一次由UT include复用，不应拷贝维护

### 1fb3ffe1 [DTS2025101528926] Conv2DV2 TF groups fix
- 根因类别：框架兼容性缺陷/groups参数推导逻辑缺失
- 涉及文件：conv/common/op_host/conv_forward_infershape.cpp
- 缺陷描述：TF框架Conv2DV2的groups默认1，但TF语义通过in_channels/kernel_channels隐式推导groups。原代码未处理groups=1但ic!=kc的隐式分组卷积场景，直接报错。
- 修复模式：增加条件分支在groups=1且ic%kc==0时自动推导groups值
- 可审查性：中
- 审查规则建议：多框架适配时关键属性应有明确的框架语义转换逻辑并加注释

### 0799defc Revert "change gather_v2 l0"
- 根因类别：接口变更回退
- 涉及文件：index/gather_v2/op_host/op_api/gather_v2.cpp
- 缺陷描述：之前的commit移除了gather_v2的isPreprocessed参数导致接口不兼容。Revert恢复该参数。
- 修复模式：完整Revert
- 可审查性：低
- 审查规则建议：L0 API参数变更应有严格兼容性审查；Revert message应明确回退原因

### 05108b11 BatchNormGradV3算子精度问题修改
- 根因类别：多核同步缺陷+参数传递错误+条件分支遗漏
- 涉及文件：norm/batch_norm_grad_v3/op_kernel/ (infer/train/common 3个头文件)
- 缺陷描述：(1)GetCrossCoreR1Param传入cIndex应为coreIndex，语义完全不同导致跨核参数计算错误；(2)ComputeCrossCoreDBias/DWeight缺少TPipeSetWaitFlag<HardEvent::MTE3_MTE2>()，MTE3写出和MTE2读入间无同步屏障，跨核拷贝读未写完数据；(3)wsReduceInputOffset计算遗漏moreMultiChannel条件；(4)blockIdx>needCoreNum比较方向应为<
- 修复模式：参数修正+硬件同步补全+条件分支补全
- 可审查性：中
- 审查规则建议：跨核同步函数参数应明确区分channel index和core index；跨核数据拷贝前必须设置硬件事件等待标志

### 91c29129 解决apiUT报错
- 根因类别：UT断言被批量注释(anti-pattern)
- 涉及文件：12个UT文件(add_rms_norm_quant_v2, batch_norm_grad_v3, batch_norm_v3等)
- 缺陷描述：大量EXPECT_EQ断言被注释掉，原验证TestGetWorkspaceSize返回值的断言被禁用。不是真正修复而是掩盖问题。
- 修复模式：注释掉失败断言(anti-pattern)
- 可审查性：高
- 审查规则建议：禁止直接注释EXPECT_*/ASSERT_*断言，需附带原因；可grep检测"// EXPECT_"模式

### 42bc1795 修复RNNV2 TILING ut
- 根因类别：API变更适配/UT编译错误
- 涉及文件：rnn/dynamic_rnnv2/tests/ut/
- 缺陷描述：UT使用旧ge::AnyValue::CreateFrom接口需迁移到Ops::NN::AnyValue::CreateFrom。删除依赖废弃ge::OpDescUtils的测试。
- 修复模式：命名空间迁移+删除废弃API依赖
- 可审查性：中
- 审查规则建议：检测ge::AnyValue、ge::OpDescUtils等已废弃API使用

### fd41c335 修复性能劣化问题
- 根因类别：优化路径边界条件缺失
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_api/, op_tiling/arch35/
- 缺陷描述：BatchMatMulToMul优化路径缺少N=1排除条件。N=1时matmul转element-wise mul反而性能劣化。修复在op_api层和tiling层同时增加nDim==1的返回false判断。
- 修复模式：增加边界条件短路返回阻止不适用场景进入优化路径
- 可审查性：高
- 审查规则建议：优化路径(算子替换/融合)需完整适用性条件检查；op_api层和tiling层相同逻辑需保持同步

### d0df69d1 celoss tiling bug fix
- 根因类别：UB内存计算不准确
- 涉及文件：cross_entropy_loss_tiling_arch35.cpp
- 缺陷描述：CrossEntropyLoss全载模板的tiling计算UB空间时未扣除ReduceSum临时buffer(maxValue)大小，导致实际运行时UB不足。修复通过GetReduceSumMaxMinTmpSize获取临时空间需求后扣除。
- 修复模式：精确计算UB资源用量，引入API获取中间buffer需求
- 可审查性：高
- 审查规则建议：tiling中UB空间分配必须包含所有中间临时buffer(reduce/cast等)；使用ReduceSum等API时应调用GetXxxTmpSize获取临时空间

### a2c4cbec fix rmsnormquant
- 根因类别：运行时类型判断vs编译期模板分支
- 涉及文件：norm/rms_norm_quant/op_kernel/rms_norm_quant.cpp
- 缺陷描述：判断输出是否INT4用运行时dstType==DT_INT4选分支，模板实例化时不匹配类型的分支仍被编译导致类型错误。修复增加yDtype模板参数，改用if constexpr(IsSameType<yDtype,int4b_t>::value)编译期消除分支。
- 修复模式：运行时if→if constexpr编译期分支消除
- 可审查性：高
- 审查规则建议：模板类中不同类型tensor操作应用if constexpr而非运行时if

### 6f830ee8 Revert "modify name of addRmsNormDynamicQuant and the binary patch"
- 根因类别：配置回退(rename导致binary配置不匹配)
- 涉及文件：add_rms_norm_quant_binary.json
- 缺陷描述：之前的rename操作导致算子二进制配置不匹配。Revert恢复正确的dtype组合和算子变体配置。
- 修复模式：git revert恢复配置
- 可审查性：低
- 审查规则建议：binary patch JSON修改应通过自动化测试验证配置与实际二进制一致性

### b63f9f0e fix ut
- 根因类别：UT基础设施与工程配置批量修复
- 涉及文件：classify_rule.yaml, 9个foreach算子UT, dynamic_rnnv2 UT, generate_cpp_cov.sh, test_op_host_main.cpp
- 缺陷描述：(1)foreach系列UT的include路径层级多一层(../../../../→../../../)；(2)classify_rule.yaml组件名ops_nn→ops-nn；(3)覆盖率脚本硬编码/home/jenkins/Ascend/→环境变量；(4)_exit(RUN_ALL_TESTS())→return，_exit跳过析构和覆盖率数据写入
- 修复模式：路径修正+环境适配
- 可审查性：中
- 审查规则建议：include路径层级应与目录结构匹配；脚本不应硬编码绝对路径；_exit()不应用于UT main

### aff96527 uniqueConsecutive算子ACLNN单输出场景修改
- 根因类别：API适用性缺陷(OP_OUTSHAPE不支持多输出)
- 涉及文件：index/unique_consecutive/op_host/op_api/unique_consecutive.cpp
- 缺陷描述：UniqueConsecutive有3个输出，但OP_OUTSHAPE在多输出场景不适用。修复对returnCounts==false时走单独注册路径只传第一个output shape。
- 修复模式：按条件分支规避框架限制
- 可审查性：中
- 审查规则建议：使用OP_OUTSHAPE时需确认输出个数满足适用条件

### b8815b6e stride code fix
- 根因类别：逻辑条件不完整
- 涉及文件：matmul/quant_matmul_v4/op_host/op_api/aclnn_quant_matmul_v4.cpp
- 缺陷描述：CheckSpecialCase只检查firstLastDim==secondLastDim但条件过宽，只有两维度都==1时才是特殊case。缺少==1约束导致不该跳过transpose设置的case被错误跳过。
- 修复模式：收紧条件增加GetDim(secondLastDim)==1约束
- 可审查性：高
- 审查规则建议：矩阵维度特判条件应明确约束具体值(如==1)，不应仅依赖维度间相等关系

### 3ed03487 fix rmsNormQuant
- 根因类别：自定义DataCopy替换为标准API
- 涉及文件：norm/rms_norm_quant/op_kernel/rms_norm_quant.cpp
- 缺陷描述：删除自定义DataCopyCustom模板函数(含复杂对齐处理+运行时类型判断)，替换为AscendC标准DataCopyPad API。原函数内部用运行时dstType==DT_INT4判断在模板实例化时可能编译错误。
- 修复模式：删除冗余手写逻辑，使用框架标准API
- 可审查性：高
- 审查规则建议：优先使用框架DataCopy/DataCopyPad API，避免自行实现对齐和分块拷贝

### 130060de 修改之前因处理告警产生的错误
- 根因类别：处理编译告警引入逻辑错误
- 涉及文件：norm/kv_rms_norm_rope_cache/op_host/kv_rms_norm_rope_cache_regbase_full_load_tiling.cpp
- 缺陷描述：三个broadcast flag常量值被错误统一为1。CONST_BRCFLAG_ZERO应=0、ONE=1、TWO=2，之前"处理告警"时全改成1导致broadcast标志位无法区分不同模式。
- 修复模式：恢复常量正确值
- 可审查性：高
- 审查规则建议：命名XXX_ZERO/ONE/TWO的常量值应与名称数字含义一致；处理编译告警不应修改常量实际值

---

跳过的提交(第370-391批)：
- 1a2b46a1: oplist整改；v-fusion目录修复(纯文档)
- 5cb632a9: fix-group_norm文档(纯文档)
- 973d8414: fix readme(纯文档)

### a77634169c fix_version_script
- 根因类别：脚本/配置错误
- 涉及文件：scripts/package/ops_nn/scripts/opp_install.sh, version.info
- 缺陷描述：(1) shell脚本中在非函数体内使用local关键字声明变量，某些shell环境下报错或行为未定义。(2) version.info依赖包版本号写死为"8.3"，缺少>=前缀，版本兼容性检查过严只允许精确匹配。
- 修复模式：删除local关键字；版本号加>=前缀
- 可审查性：高
- 审查规则建议：shell lint检查local关键字只在函数内使用；版本依赖配置应有schema校验支持范围语义

### 5f7c1ea4 foreach_minimum_list fix duplicate REG_OP
- 根因类别：复制粘贴错误(算子注册定义)
- 涉及文件：foreach/foreach_minimum_list/op_graph/foreach_minimum_list_proto.h
- 缺陷描述：foreach_minimum_list的算子原型错误复制了ForeachMinimumScalarList的REG_OP注册代码，算子名错误为MinimumScalarList，输入从双DYNAMIC_INPUT(x1,x2)变成DYNAMIC_INPUT(x)+INPUT(scalars)——List版本的双tensor list输入被错误定义为ScalarList版本的tensor+scalar输入。
- 修复模式：REG_OP名改为ForeachMinimumList，输入改为正确的双DYNAMIC_INPUT(x1, x2)
- 可审查性：高
- 审查规则建议：检查op_graph proto文件中REG_OP名称是否与文件名/目录名一致；检测同一仓库中是否存在重复的REG_OP注册名

### 225c12dd flatquant精度问题修复
- 根因类别：内存偏移计算缺少sizeof(T) + 核间同步缺失
- 涉及文件：quant/flat_quant/op_kernel/flat_quant_cube.h, flat_quant_vec.h, tensor_utils.h
- 缺陷描述：(1) workspace指针偏移计算时遗漏* sizeof(T)，导致doubleP1GM的GlobalBuffer设置在错误偏移位置(按元素数而非字节偏移)，与outnzGM区域重叠造成数据踩踏精度问题。(2) vec端缺少核间同步(缺TWO_VEC_SYNC_ID的CrossCoreSetFlag/WaitFlag)。
- 修复模式：偏移计算补乘sizeof(T)；增加核间同步屏障
- 可审查性：中
- 审查规则建议：所有GlobalBuffer偏移计算需审查是否混淆"元素数"和"字节数"；workspace划分区域时需以字节为单位统一计量

### bd7975a8 处理大Wi场景L0C搬出时读取数据溢出问题
- 根因类别：边界条件处理不完整(大shape下溢)
- 涉及文件：conv/conv3d_backprop_input_v2/op_kernel/arch35/.../conv_bp_sub_func_mix.h
- 缺陷描述：CalcCutInWIndex函数中计算Wi切分时，遗漏curWiPos > doubleBaseUseM的情况(大Wi场景)，headWi_被赋超大值，导致leftBaseUseM = doubleBaseUseM - headWi_下溢。
- 修复模式：增加curWiPos > doubleBaseUseM判断分支，headWi_置0
- 可审查性：中
- 审查规则建议：涉及数据搬运偏移计算的代码需覆盖边界case；减法结果可能为负时需检查保护

### 68500442 fix include
- 根因类别：头文件include路径错误
- 涉及文件：optim/apply_adam_w_v2/op_host/apply_adam_w_v2_tiling_arch35.h
- 缺陷描述：#include "elewise/elewise_tiling.h"路径缺少atvoss/前缀，应为"atvoss/elewise/elewise_tiling.h"，导致某些编译环境下找不到头文件。
- 修复模式：修正include路径
- 可审查性：高
- 审查规则建议：CI编译所有target确保include路径正确

### 3fc12f1c MatMul修复l1iterbatchAicError
- 根因类别：K维度对齐逻辑错误(A/B不必要分离引入bug)
- 涉及文件：matmul/batch_mat_mul_v3/op_host/op_tiling/arch35/batch_matmul_v3_iterbatch_basicapi_tiling.cpp/.h
- 缺陷描述：原代码将K对齐分为alignKaValue_(A矩阵)和alignKbValue_(B矩阵)，根据转置状态分别对齐。导致iterbatch场景下L1/L0A/L0B容量计算使用不同K对齐值，baseK取min(Ka,Kb)，某些shape组合下tiling结果不正确引发AIC error。K维度在MatMul中应统一。
- 修复模式：合并alignKaValue_/alignKbValue_为单一alignKValue_，统一对齐到BASIC_BLOCK_SIZE_16
- 可审查性：中
- 审查规则建议：tiling参数计算中同一维度不应有多个对齐变体；新增tiling路径需覆盖极端shape组合

### bb4b4662 fix compile (模板类型参数错误)
- 根因类别：复制粘贴错误(A/B类型混淆)
- 涉及文件：common/act/matmul/block/block_quant_with_tile_mmad_multi_block.h
- 缺陷描述：CopyCubeInB模板实例化中，MatmulInputBType<InputBType, typename InputAType::T>错误引用InputAType::T。B矩阵应使用InputBType::T。当A/B数据类型不同时(如A为fp16、B为int8量化场景)，导致编译错误或运行时类型不匹配。
- 修复模式：InputAType::T改为InputBType::T
- 可审查性：高
- 审查规则建议：模板类型参数中A/B应明确区分，review重点关注对称结构中的A/B互换错误

### c6e269cf matmulv3 infershape对齐修改+addMM示例缺陷修改
- 根因类别：(1)对齐公式off-by-one (2)示例代码内存泄漏
- 涉及文件：matmul/mat_mul_v3/op_host/mat_mul_v3_infershape.cpp, matmul/mat_mul_v3/examples/test_aclnn_addmm_aclnninplace_addmm.cpp
- 缺陷描述：(1) hidden_size对齐公式(*hidden_size + BLOCK_SIZE) / BLOCK_SIZE * BLOCK_SIZE多加1个BLOCK_SIZE，正确应为+BLOCK_SIZE-1。(2) 示例代码aclnnInplaceAddmm复用workspaceAddr变量做aclrtMalloc，前一次workspace地址被覆盖无法释放。
- 修复模式：(1)修正对齐公式；(2)使用独立变量并正确释放
- 可审查性：高
- 审查规则建议：向上对齐标准写法(x+align-1)/align*align应作为静态检查规则

### ba467b19 quantbatchmatmul 修复修改编译警告导致的不一致
- 根因类别：修复编译warning时引入逻辑变更
- 涉及文件：matmul/quant_batch_matmul_v4/op_host/op_api/aclnn_quant_matmul_v5.cpp
- 缺陷描述：修改编译警告时，CheckSpecialCase的条件判断被错误改写，误删了&& dim(secondLast) == 1条件，变成仅检查两个维度相等。导致非1x1情况(如2x2)也被判定为特殊case，跳过transpose属性设置引起计算错误。
- 修复模式：恢复被误删的== 1判断条件
- 可审查性：高
- 审查规则建议：修复编译warning时不应改变条件逻辑，review应对比修复前后语义等价性

### 5626757109 算子包脚本修复
- 根因类别：硬编码平台名导致多平台不兼容
- 涉及文件：scripts/package/ops_nn/scripts/ops_nn_custom_install.sh, ops_nn_custom_remove_softlink.sh
- 缺陷描述：安装/卸载脚本中将平台目录硬编码为ascend910b，仅支持910b一个平台。新增其他ascend平台时软链接不会创建/删除。
- 修复模式：硬编码改为PATTERN="ascend*"通配遍历所有ascend平台目录
- 可审查性：高
- 审查规则建议：安装/部署脚本中不应硬编码平台型号，应使用通配或配置驱动

### 8681104f fix tilingkey compile error
- 根因类别：条件编译宏未同步定义
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/weight_quant_batch_matmul_v2_apt.cpp
- 缺陷描述：A16W4(int4b_t)场景下，#ifdef块重定义了DTYPE_WEIGHT和S4宏，但遗漏重定义ORIG_DTYPE_WEIGHT为DT_INT4。后续代码依赖ORIG_DTYPE_WEIGHT进行条件编译分支选择，缺少此宏导致tilingkey对应kernel编译失败。
- 修复模式：增加#undef ORIG_DTYPE_WEIGHT + #define ORIG_DTYPE_WEIGHT DT_INT4
- 可审查性：高
- 审查规则建议：条件编译块中重定义类型相关宏时，需检查所有关联宏是否同步更新

### a0a2246c 修复gather_elements_v2 超INT_MAX shape的bug
- 根因类别：整数溢出(int32不足以表示大shape偏移)
- 涉及文件：index/gather_elements_v2/op_kernel/gather_elements_v2_last_dim.h, index/gather_v2/op_host/op_api/aclnn_gather_v3.cpp
- 缺陷描述：(1) kernel中xGmBaseOffset超INT_MAX时static_cast<int32_t>截断溢出，Adds指令索引计算错误。(2) gather_v3 op_api仅检查self tensor size阈值决定精度模式，遗漏index tensor size过小也需高精度模式。
- 修复模式：(1)增加INT_MAX边界检查分支，大偏移用逐元素加法；变量升级int64_t (2)增加index size阈值检查
- 可审查性：中
- 审查规则建议：kernel中涉及shape乘积/偏移计算的变量应默认使用int64_t

### 96f1a29b MaxPool3DWithArgmaxV2修复非常规数错判NAN值
- 根因类别：NaN判定算法错误(基于范围比较而非IEEE754标准)
- 涉及文件：pooling/adaptive_max_pool3d/op_kernel/adaptive_max_pool3d_big_pool.h, pooling/max_pool3d_with_argmax_v2/op_kernel/max_pool3d_with_argmax_big_kernel.h
- 缺陷描述：原NaN判定用数值范围比较(nan > INF)，对denormalized number(如1e-45)的二进制表示恰好落在NaN范围内，正常极小浮点数被误判为NaN导致索引错误。正确做法按IEEE754：指数位全1且尾数位非0才是NaN。
- 修复模式：改用位掩码检测：fp16检查(nan & 0x7C00)==0x7C00 && (nan & 0x03FF)!=0，fp32检查(nan & 0x7F800000)==0x7F800000 && (nan & 0x007FFFFF)!=0
- 可审查性：高
- 审查规则建议：NaN/Inf判定必须基于IEEE754位模式而非数值范围比较；此类工具函数应抽取为公共库

---

跳过的提交(第392-411批)：
- 19216d51: fix_ut_1107(纯UT/示例代码修复，数据类型int32->int64/删除死函数/修正CMake宏)
- ee1806a6: masked_softmax算子修正ut(纯UT整改，kernel侧仅头文件include和常量定义)
- 7e28bd1a + 2a728a65: fix foreach host ut + foreach op fix(互逆操作，净效果为零)
- 730babc2: 修复quant类算子kernel ut报错(纯UT修复)
- 92968c7a: fix norm kernel ut error(纯UT修复，38个文件全在tests/ut目录)
- 02bd44c9: gelu_quant等warning修复(纯printf格式符修正+删除未使用变量，无逻辑变更)

---

## 第412-431批 (20条)

### 2c18ae1e 修改SwiGluQuant大shape精度问题
- 根因类别：数据类型溢出(参数类型截断)
- 涉及文件：quant/swi_glu_quant/op_kernel/swi_glu_quant.h, swi_glu_quant_static.h
- 缺陷描述：ProcessCoreMultiUbMultiAlign函数的offsetRow参数声明为uint16_t(最大65535)，但调用处传入uint32_t。大shape时行偏移超过65535发生截断，导致访问错误地址产生精度问题。
- 修复模式：参数类型从uint16_t提升为uint32_t
- 可审查性：高
- 审查规则建议：检测函数参数窄类型接收宽类型的隐式截断(uint16_t <- uint32_t)，-Wconversion可捕获

### 1a684711 修复wqbmmv2 310P kernel入口条件判断
- 根因类别：条件分支逻辑错误(过度约束)
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/weight_quant_batch_matmul_v2.cpp
- 缺陷描述：310P平台kernel入口if constexpr条件中包含多余的HAS_ANTIQUANT_OFFSET == 0约束，但kernel内部已通过模板参数处理两种场景。多余条件导致带antiquant offset时无法进入310P分支。对比相邻BATCH_ONE分支无此约束可佐证。
- 修复模式：删除if constexpr中多余的HAS_ANTIQUANT_OFFSET == 0条件
- 可审查性：中
- 审查规则建议：同一函数内多个if constexpr分支条件应对称检查；当模板参数已在宏调用中传入时，入口条件不应对该参数做硬编码值约束

### a984e3f8 fix group_norm_silu ut
- 跳过(纯UT修复)

### a6686eda 修复FlatQuant功能报错
- 根因类别：workspace内存布局错误(buffer地址计算)
- 涉及文件：quant/flat_quant/op_kernel/flat_quant_high.h
- 缺陷描述：workspace被x1GM(类型T)和x2GM(类型float)共享。原代码x1GM在前x2GM在后，但x2GM需要更大空间(K_DOUBLE_VEC vs K_PER_VEC)，导致两者内存区域重叠。同时偏移量sizeof转换方向错误。
- 修复模式：交换两个buffer在workspace中的分配顺序，修正偏移量计算的sizeof方向
- 可审查性：中
- 审查规则建议：多buffer共享workspace时，审查每个buffer的offset + size不得超过下一个区域的offset；关注sizeof类型转换方向

### 8ee73052 修复infershape中先使用后校验nullptr的问题
- 根因类别：空指针解引用(use-before-null-check)
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：先对shape_x1/shape_x2执行解引用(CheckIsUnknownDimNum(*shape_x1))，然后才检查nullptr。CWE-476典型模式。
- 修复模式：将nullptr校验提前到解引用之前；attrs的null check单独拆分到实际使用前
- 可审查性：高
- 审查规则建议：静态分析规则use-before-null-check，Coverity/Clang SA可直接检出

### 4203e836 celoss ut fix
- 跳过(纯UT修复)

### 11bd363e quantbatchmatmul 修复编译警告
- 根因类别：编译警告 + 隐藏逻辑缺陷
- 涉及文件：matmul/quant_batch_matmul_v3/..., matmul/quant_batch_matmul_v4/...多个文件
- 缺陷描述：(1) printf格式说明符不匹配(%ld打印uint64_t应为%lu)；(2) 未使用函数参数；(3) 实际逻辑缺陷：CheckSpecialCase只检查firstLastDim == secondLastDim就跳过transpose设置，缺少&& secondLastDim == 1约束，导致两维度相等但非1时错误跳过transpose。
- 修复模式：格式符修正 + 未使用参数处理 + 条件收紧增加==1校验
- 可审查性：中
- 审查规则建议：条件判断只比较"相等"时需审视是否还需约束具体值；逻辑修复不应混在warning修复提交中

### 0d4c358c fix compile warnings [dynamic_quant_update_scatter_v2]
- 根因类别：编译警告(printf格式说明符)
- 涉及文件：quant/dynamic_quant_update_scatter_v2/op_host/dynamic_quant_update_scatter_v2_tiling.cpp
- 缺陷描述：%d打印需要%ld的类型。修复用(long)强转+%ld，不如PRId64规范。
- 修复模式：格式说明符修正 + (long)强制类型转换
- 可审查性：高
- 审查规则建议：日志打印应使用inttypes.h的PRId64/PRIu64宏

### cf6e64b9 调整1952 bias非全载条件、新增scale非全载拦截，修复extend conv transpose fp16精度问题
- 根因类别：L1内存分配未预留硬件buffer空间 + 全载条件缺失 + scale尺寸校验缺失
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/arch35/多个tiling文件, common/conv_backprop_input_context_utils.*
- 缺陷描述：(1) L1容量校验在IsSocVersionFuse下未扣除bias buffer(4KB)和scale buffer(6KB)预留空间，导致tiling参数偏大、数据越界覆盖引发fp16精度问题；(2) 缺少isBiasFullLoad标志，bias超大仍按全载处理导致截断；(3) INT8下缺少scale尺寸上限校验。
- 修复模式：L1容量校验扣除预留空间 + 新增bias全载判断 + 新增scale尺寸拦截
- 可审查性：低
- 审查规则建议：L1/UB容量校验必须考虑所有硬件预留buffer(特别是多SoC版本适配时)；"是否全载"判断必须基于实际容量与数据量比较

### 58fccb11 修复dynamic_quant_update_scatter
- 跳过(纯UT修复)

### 7e5f88dc issue 修复
- 根因类别：脚本健壮性缺陷
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op_gen_task.sh, gen_output_json.py
- 缺陷描述：(1) shell脚本中dos2unix命令未检查文件存在性、未检查工具可用性、未检查CRLF格式；(2) Python脚本编译失败时错误信息不充分缺排查指引。
- 修复模式：前置条件校验(文件存在→CRLF检测→工具可用) + 错误信息增强
- 可审查性：高
- 审查规则建议：shell脚本中外部命令调用前需command -v可用性检查；文件操作前需-f存在性判断

### ecf7f382 [convbp] fix bug for transpose padding<0
- 根因类别：输入校验缺失(负值边界)
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/common/conv_backprop_input_context_utils.cpp
- 缺陷描述：CheckTranspose函数处理output_padding时只检查全零判断transpose模式，未检查负数值。负数padding在Arch35+上导致未定义行为。
- 修复模式：新增outputPaddingAllNonNegative标志位，负值时报错返回
- 可审查性：高
- 审查规则建议：卷积算子的padding/stride/dilation等整型参数需检查负值或越界校验

### 46ba9c58 修改问题单
- 跳过(纯文档，仅补充API注释)

### 40759d9b Fix Dw Problem
- 根因类别：多重缺陷(tiling结构体bool类型 + 返回值未检查 + 函数定义顺序)
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/..., op_kernel/..., tests/ut/...
- 缺陷描述：(1) tiling数据结构中isSplitKernelHW字段类型为bool，host-device间按字节传输时bool大小依赖编译器实现可能不一致，改为int32_t并调整字段排列顺序；(2) SetStridesAttr/SetDilationsAttr返回值被忽略，非法参数继续传递；(3) kernel中模板函数定义在调用方之后导致编译告警。
- 修复模式：bool→int32_t类型修正 + OP_CHECK_IF包装返回值检查 + 函数定义顺序调整
- 可审查性：中
- 审查规则建议：tiling数据结构禁用bool(应使用固定宽度整数)；属性设置函数返回值必须检查

### 8c98bd58 FlatQuant修复大shape偏移溢出，高阶matmul模板修复精度问题
- 根因类别：多重缺陷(int32溢出 + workspace寻址错误 + 变量名笔误 + 尾块搬运缺失)
- 涉及文件：quant/flat_quant/op_host/flat_quant_tiling.cpp, op_kernel/flat_quant_cube.h, flat_quant_high.h, flat_quant_vec.h, tensor_utils.h
- 缺陷描述：(1) FlatQuantShapeInfo所有字段为int32_t，k*M*N偏移计算溢出int32范围，全部改为int64_t；(2) workspace寻址用k*shape.Mceil*shape.N导致不同core写入同一区域，改为GetBlockIdx()*K_PER_VEC + k%K_DOUBLE_VEC做per-core隔离；(3) Nceil计算错误用M而非N：shape.Nceil = (shape.M + CEIL_SIZE - 1) / CEIL_SIZE * CEIL_SIZE；(4) 新增尾块慢路径搬运(invalidK判断)避免读越界。
- 修复模式：int32→int64 + per-core workspace寻址 + M→N笔误修正 + 尾块搬运路径
- 可审查性：低
- 审查规则建议：shape维度相乘偏移用int64_t；workspace寻址必须含blockIdx隔离；ceil变量名须与目标维度一致

### 6b39aa33 fix SMS ut;add SMS/SMSG example
- 根因类别：Tensor尺寸信息缺失 + Tiling字段遗漏
- 涉及文件：vfusion/scaled_masked_softmax_v2/op_kernel/scaled_masked_softmax_v2.h(生产代码), tests/ut/...(UT)
- 缺陷描述：生产代码中LocalTensor使用前未调用SetSize()设置有效数据长度，后续Softmax等shape-aware API可能读未初始化区域。UT的tiling解析宏遗漏width字段。
- 修复模式：在使用Tensor前补充SetSize()调用；tiling宏补充width字段
- 可审查性：中
- 审查规则建议：AllocTensor和shape-dependent API之间必须存在SetSize调用；tiling解析宏应与结构体定义做字段一致性比对

### 395e3673 fixbug: WeightQuantBatchMatmulV2 vf中u16和u32比较会出现循环异常
- 根因类别：隐式类型提升导致硬件行为异常
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/weight_quant_batch_matmul_v2_vf.h
- 缺陷描述：vf kernel中for循环迭代变量为uint16_t，上界为uint32_t。C++隐式将uint16_t提升为uint32_t在CPU上安全，但vf硬件对u16 vs u32比较指令有偶发异常，导致循环次数不正确产生精度错误。
- 修复模式：在5个for循环上界表达式处显式static_cast<uint16_t>()
- 可审查性：高
- 审查规则建议：vf kernel代码中循环变量和上界必须类型一致；正则for\s*\(\s*uint16_t.*<\s*p\.可扫描疑似位点

### c760190c fix addrmsnorm splitd bug
- 根因类别：控制流逻辑错误(条件分支外的累加器更新)
- 涉及文件：norm/add_rms_norm/op_kernel/arch35/add_rms_norm_regbase_split_d.h
- 缺陷描述：ComputeFormer函数stage2的两个分支(tail <= ubFactor/2 和 tail > ubFactor/2)执行ComputeSum后，level1 += 1和ComputeMultiLevelReduce放在分支外面。当tail == 0时两个分支都不进入但仍执行level递增和多余reduce，影响RMSNorm精度。
- 修复模式：将level1递增和reduce调用移入各自if/else if分支内部
- 可审查性：高
- 审查规则建议：多阶段累加/规约模式中，level递增和reduce必须与ComputeSum严格配对在同一条件分支内

### 794c8451 DynamicBlockQuant算子UT修复
- 跳过(纯UT/文档修复)

### 77556df0 fix msda msdag ut
- 跳过(纯UT修复)

---

跳过的提交(第412-431批)：
- a984e3f8: fix group_norm_silu ut(纯UT修复)
- 4203e836: celoss ut fix(纯UT修复)
- 58fccb11: 修复dynamic_quant_update_scatter(纯UT修复)
- 46ba9c58: 修改问题单(纯文档，仅补充API注释)
- 794c8451: DynamicBlockQuant算子UT修复(纯UT/文档)
- 77556df0: fix msda msdag ut(纯UT修复)

## 第23轮分析 (defect_commits.txt 第432-451行)

跳过的提交：
- 1a33848b: fix scatter_elements_v2 原型注释(纯注释修改)
- 6b59bbdb: 修改大模型排查问题(纯文档修复，104个README/docs文件)
- 5334965f: swi_glu_quant kernal ut error修复(纯UT修复)
- f8561b1e: fix test_geir_kv_rms_norm_rope_cache(纯测试用例重写)
- 168d3bcd: fix tbmm mmad compile warning(重构清理，用官方API替换自定义封装)
- 48d33953: fix lisence(纯license声明修复)
- af539669: fix AdaptiveMaxPool3d\Grad example(纯示例代码shape/data修正)

### bf173212 修复kernel文件有后缀的情况下编译无法打包.o和json文件的问题
- 根因类别：构建脚本文件名处理逻辑缺陷
- 涉及文件：cmake/gen_ops_info.cmake, scripts/kernel/binary_script/build_binary_single_op_gen_task.sh
- 缺陷描述：当kernel Python文件带有后缀(如_apt、_910b)时，构建脚本在两个子函数中各自对op_file_name做后缀剥离，但仅修改局部变量不影响调用方后续构造.o和.json输出路径。CMake侧用精确文件名匹配install(FILES ...json)，当实际json文件名带额外后缀时匹配失败。
- 修复模式：(1)shell脚本将后缀剥离提取到main函数统一处理；(2)CMake改用file(GLOB *.json)+install通配符匹配
- 可审查性：中
- 审查规则建议：同一变量转换逻辑在多个函数中重复时应提取到公共调用点；CMake install(FILES)用硬编码文件名时检查是否有变体未覆盖

### 83f360f0 docxfix (原型定义部分)
- 根因类别：类型声明错误(复制粘贴)
- 涉及文件：matmul/batch_mat_mul_v3/op_graph/batch_mat_mul_v3_proto.h
- 缺陷描述：BatchMatMulV3算子bias输入的TensorType声明中，将DT_BF16误写为DT_FLOAT，导致{DT_FLOAT16, DT_FLOAT, DT_FLOAT}出现重复DT_FLOAT而缺少DT_BF16支持。x1和x2都声明了DT_BF16但bias遗漏。
- 修复模式：将重复的DT_FLOAT替换为DT_BF16
- 可审查性：高
- 审查规则建议：检测同一算子注册宏中TensorType列表内重复枚举值；当输入tensor声明了BF16但关联参数未声明时警告

### 09c38a47 fix eigen compilation using cuda keyword
- 根因类别：第三方库构建配置缺失
- 涉及文件：cmake/third_party/eigen.cmake
- 缺陷描述：ExternalProject_Add引入eigen时未设置CONFIGURE_COMMAND ""和BUILD_COMMAND ""，导致CMake用默认行为编译eigen，eigen源码含CUDA关键字在非CUDA环境下编译失败。实际只需下载头文件(header-only库)。
- 修复模式：添加CONFIGURE_COMMAND ""和BUILD_COMMAND ""跳过配置和构建
- 可审查性：高
- 审查规则建议：header-only第三方库的ExternalProject_Add应同时设置CONFIGURE/BUILD/INSTALL_COMMAND为空

### c26eedd7 fix workspace
- 根因类别：workspace大小计算遗漏
- 涉及文件：norm/add_layer_norm_quant/op_host/add_layer_norm_quant_tiling_arch35.cpp
- 缺陷描述：workspace大小计算两个问题：(1)初始值设为1字节过小，应为32字节DEFAULT_WORKSPACE_SIZE；(2)当rowsPerCore_==1&&isDynamicQuant_且非WELFORD策略时，缺少sysWorkspaceSize_累加，导致workspace分配不足可能内存越界。
- 修复模式：补全条件分支中遗漏的workspace累加逻辑
- 可审查性：中
- 审查规则建议：workspace/buffer大小计算时检查所有条件分支是否完整覆盖所有内存组件

### d97951a7 拦截GeluQuant空tensor输入问题
- 根因类别：空tensor输入校验缺失
- 涉及文件：activation/gelu_quant/op_host/gelu_quant_tiling_arch35.cpp
- 缺陷描述：GeluQuant算子tiling阶段未校验输入tensor各维度是否为0，传入空tensor时后续tiling计算(对齐、除法)产生未定义行为。
- 修复模式：在tiling入口遍历输入shape各维度，发现0维度时提前返回GRAPH_FAILED
- 可审查性：高
- 审查规则建议：所有算子tiling入口应检查输入shape是否包含0维度

### 941e12f0 fix qbmm
- 根因类别：平台硬件参数被上层错误修改 + 基类方法非virtual
- 涉及文件：matmul/quant_batch_matmul_v3/op_host/op_tiling/quant_batch_matmul_v3_tiling_base.h, matmul/quant_batch_matmul_v4/op_host/op_tiling/quant_batch_matmul_v4_pergroup_tiling.cpp/.h
- 缺陷描述：两层缺陷：(1)父类SetPlatformInfoForTiling()未声明virtual，子类无法override定制；(2)B4平台L2 cache大小被上层从168MB错误改为96MB，tiling基于错误L2容量计算。
- 修复模式：父类方法改为virtual支持override；子类override中硬编码修正L2大小(96->168MB)
- 可审查性：中
- 审查规则建议：基类中被子类可能需定制的方法应声明virtual；硬编码平台参数修正是临时workaround应跟踪上游修复

### 5cf2e333 解决geluQuant输入为scalar时报错问题
- 根因类别：边界条件缺失(scalar/0维tensor)
- 涉及文件：activation/gelu_quant/op_host/gelu_quant_tiling_arch35.cpp/.h
- 缺陷描述：geluQuant输入tensor为scalar(0维)时，直接对shape调用GetDimNum()/GetDim(0)/GetShapeSize()，scalar语义与普通tensor不同导致tiling计算异常。
- 修复模式：引入EnsureNotScalar()方法将scalar shape映射为{1}，统一处理
- 可审查性：中
- 审查规则建议：对tensor shape调用GetDimNum()/GetDim()时检查是否处理了scalar(0维)情况

### f0e7c19b bugfix_ctcloss
- 根因类别：条件分支优先级错误
- 涉及文件：loss/ctc_loss_v2/op_host/op_api/ctc_loss_v2.cpp
- 缺陷描述：CtcLossV2中V3 AiCore路径判断被放在V2之前，当输入同时满足V2和V3条件时V3优先匹配，导致本应走V2的case错误走V3。V2条件更严格应优先判断。
- 修复模式：交换if-else分支顺序，将更特化的V2判断放前面
- 可审查性：低
- 审查规则建议：多个互斥IsXxxSupport判断链中，更严格/特化的条件应排在前面

### eb5334f2 GeGluGradV2 kernel generate failed. code modify
- 根因类别：constexpr与预编译机制不兼容
- 涉及文件：activation/ge_glu_grad_v2/op_kernel/ge_glu_grad_v2_apt.cpp
- 缺陷描述：kernel中tiling key值声明为constexpr int32_t，在预编译(codegen)场景中constexpr变量无法被编译器作为宏展开进行模板/分支消除，导致kernel编译失败。
- 修复模式：将18个constexpr改为#define宏定义
- 可审查性：高
- 审查规则建议：kernel代码(op_kernel目录)中用于tiling key的常量必须用#define而非constexpr/const

### a0ccff51 [conbp]fix bug of batchdout
- 根因类别：计算逻辑错误(切分粒度公式方向反)
- 涉及文件：conv/conv3d_backprop_filter_v2/op_kernel/arch35/conv3d_backprop_filter_v2/conv3d_dw_v2_basic_block.h
- 缺陷描述：batchdoutPerBlock_计算公式为Ceil(totalAddSize, THRESHOLD)混淆了"切分块数"和"每块大小"的语义。正确应为Ceil(THRESHOLD, ho*wo)计算每块处理量，然后按偏移步进遍历。
- 修复模式：修正切分粒度计算方向+循环改为按偏移步进+修正尾块大小计算
- 可审查性：中
- 审查规则建议：Ceil(A,B)切分计算需确认A和B语义——"总量/块大小=块数"还是"阈值/单元大小=每块处理量"

### 3715ff61 fix celossgrad
- 根因类别：流水线同步屏障遗漏
- 涉及文件：loss/cross_entropy_loss_grad/op_kernel/arch35/cross_entropy_loss_grad_base.h
- 缺陷描述：WeightReduceSumTailN中Vector计算(Adds)完成后直接调用DataCopyPad将Local搬到GM，缺少PipeV2M()同步屏障。Vector结果尚未写入Local buffer时MTE3就开始搬运，非全载mean场景偶现精度错误。
- 修复模式：在DataCopyPad前插入PipeV2M()
- 可审查性：中
- 审查规则建议：DataCopyPad/DataCopy(Local->GM)前必须有PipeV2M()/PipeS2M()同步

### e44ac0fd fix aicpu example code issue
- 根因类别：变量名错误 + 循环变量类型不匹配
- 涉及文件：examples/add_example_aicpu/op_kernel_aicpu/add_example_aicpu.cpp
- 缺陷描述：(1)日志打印"Num of elements"但传入data_size而非num_elements；(2)循环变量int i但边界num_elements是int64_t，超INT_MAX时截断。
- 修复模式：(1)日志参数改为num_elements；(2)循环变量改为int64_t
- 可审查性：高
- 审查规则建议：printf日志参数与格式串语义匹配；循环边界为int64_t时循环变量也应为int64_t

### bf34b10d fix error code
- 根因类别：缺少return语句(错误路径未返回) + 校验逻辑位置错误
- 涉及文件：matmul/mat_mul_v3/op_host/op_api/aclnn_addmm.cpp, matmul/mat_mul_v3/op_host/op_api/aclnn_matmul.cpp
- 缺陷描述：(1)CheckMatmulWeightNz中检测dtype非法并打印错误日志后没有return false，错误路径fall through；(2)dtype校验放在Shape校验函数中违反职责分离，且CheckDtypeValid末尾直接return true跳过检查。
- 修复模式：(1)补充return false；(2)将dtype校验从Shape函数移到独立的CheckWeightNzDtype函数
- 可审查性：高
- 审查规则建议：if分支中有错误日志(OP_LOGE)但没有return的代码路径应发出警告

### 9bddf3ad Fix aclnnQuantConvolution Conv3DV2 tiling kAL1 upper bound
- 根因类别：算法逻辑错误 -- 上界计算公式有误
- 涉及文件：conv/conv3d_v2/op_host/op_tiling/conv3d_api_tiling_algorithm.cpp
- 缺陷描述：kAL1上界的计算公式为`(POSTK_LIMIT + k0) / ci0HkWk`，但正确语义应是"kAL1乘以ci0HkWk后不能超过POSTK_LIMIT(65535)"。原公式多加了一个k0，导致上界偏大，选出的kAL1值在乘回ci0HkWk后可能超过load3d指令的postk硬件限制65535，引发精度或功能问题。
- 修复模式：将公式从`(POSTK_LIMIT + k0) / ci0HkWk`修正为`POSTK_LIMIT / ci0HkWk`
- 可审查性：中
- 审查规则建议：涉及硬件指令限制常量的边界计算，应验证"计算结果 x 乘回因子 <= 限制值"的不变量是否成立

### 9a18f2ca ge_glu_v2算子kernal ut修复error
- 根因类别：UT测试用例参数与算子实现不匹配
- 涉及文件：activation/ge_glu_v2/tests/ut/op_kernel/test_ge_glu_v2.cpp
- 缺陷描述：多个测试用例存在多处问题：(1)tiling参数与kernel逻辑不一致(loopNum/nLastTailGroup值颠倒，blockSize/ny值错误)；(2)输出buffer大小未考虑GLU算子输出维度减半；(3)workspace分配过小；(4)缺少SetKernelMode(AIV_MODE)调用；(5)blockDim=48与realCoreNum=40不一致
- 修复模式：修正所有tiling参数、输出buffer大小、workspace大小，统一blockDim=40，补充SetKernelMode调用
- 可审查性：中
- 审查规则建议：UT中的tiling参数应由tiling函数自动生成而非手工硬编码

### 4805e45e 修复aclnn conv日志打印有歧义
- 根因类别：日志消息不准确/有歧义
- 涉及文件：conv/convolution_forward/op_host/op_api/aclnn_convolution.cpp
- 缺陷描述：输入shape经过pad和dilation后小于0时，原错误日志仅打印"expect input shape[i] >= 0"，没有说明是输入尺寸小于kernel尺寸的根本原因，日志表述有歧义
- 修复模式：将日志改为更详细的描述包含完整计算公式和维度索引
- 可审查性：高
- 审查规则建议：错误日志应包含完整的计算公式和相关变量值；OP_LOGE的format string参数个数与实际参数应匹配

### 3aabd2e3 fixbug: 使用未初始化的值导致精度错误的问题
- 根因类别：使用未初始化变量 + 数据类型与硬件不兼容
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/weight_quant_batch_matmul_v2_reg_base_common.h, matmul/weight_quant_batch_matmul_v2/op_kernel/arch35/weight_quant_batch_matmul_v2_vf.h
- 缺陷描述：(1)条件判断`if (tiling_->kBubSize >= params.groupSize)`中params.groupSize尚未初始化(在if体内才被赋值)，导致条件结果不确定引发精度错误；(2)vf硬件不支持uint32_t类型的循环变量
- 修复模式：(1)将条件中的params.groupSize改为已有值tiling_->groupSize；(2)将循环变量从uint32_t改为uint16_t适配vf硬件
- 可审查性：高
- 审查规则建议：静态分析可检测use-before-def模式；对vf硬件代码应建立禁用类型清单(如uint32_t)

### 9609a1a4 fix argument gmaddr problem
- 根因类别：结构体初始化列表缺少成员导致位置偏移
- 涉及文件：matmul/mat_mul_v3/op_kernel/arch35/mat_mul_streamk_basic_act.h
- 缺陷描述：Params结构体gm addr初始化列表`{aGM, bGM, cGM, biasGM, workspaceGM}`缺少一个字段，导致workspaceGM被赋值到错误的结构体字段位置
- 修复模式：在biasGM和workspaceGM之间插入nullptr，修正为`{aGM, bGM, cGM, biasGM, nullptr, workspaceGM}`
- 可审查性：高
- 审查规则建议：C++聚合初始化应使用designated initializers或注释标明字段名；初始化列表元素数量应与结构体字段数量匹配；开启-Wmissing-field-initializers

### cc407a02 fix gather_v2 oom
- 根因类别：内存阈值常量设置过小导致OOM
- 涉及文件：index/gather_v2/op_host/op_api/aclnn_gather_v3.cpp
- 缺陷描述：MIN_SELF_SIZE常量61440过小，导致某些场景workspace不足触发OOM
- 修复模式：将MIN_SELF_SIZE从61440调大到102400
- 可审查性：低
- 审查规则建议：魔数常量应配合注释说明推导过程和约束条件

### a4e4225f 修复头文件找不到
- 根因类别：include路径错误 -- 目录结构变更后未同步更新
- 涉及文件：activation/ge_glu_grad_v2/examples/*.cpp, activation/ge_glu_v2/examples/*.cpp (4个文件)
- 缺陷描述：include路径使用了aclnnop/level2/aclnn_geglu*.h，但头文件已不在level2子目录下，导致编译找不到头文件
- 修复模式：将include路径去掉中间的level2/
- 可审查性：高
- 审查规则建议：目录结构重构时应全仓搜索受影响的include路径；CI编译应覆盖examples目录

### 7dfb2a69 上边界tensor int类型越界修改
- 根因类别：整数类型溢出 -- int存储shape乘积
- 涉及文件：index/index_fill_d/op_host/op_api/aclnn_index_fill_tensor.cpp
- 缺陷描述：GenerateAssistMatrix函数中blocksize、blocknum、n三个变量为int(32位)，通过shape各维度乘法计算总元素数，超大shape时溢出
- 修复模式：将blocksize、blocknum、n和循环变量i从int改为int64_t
- 可审查性：高
- 审查规则建议：涉及tensor shape维度乘积的变量一律使用int64_t

### 0866cced [conbp]fix bug for streamkType=0
- 根因类别：控制流逻辑错误 -- 设置streamkType=0后未阻止后续streamk计算
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch35/conv3d_backprop_filter_v2_stream_k_tiling.cpp, conv/conv3d_backprop_filter_v2/tests/ut/op_host/test_conv3d_backprop_filter_v2_arch35_tiling.cpp
- 缺陷描述：设置streamkType=NO_STREAMK_CALC后没有return或else分支，代码继续执行streamk分核逻辑；同时缺少batchDout=1或howo=1(分核数为1)的退化处理
- 修复模式：将streamk分核逻辑放入else分支；新增UT覆盖退化场景
- 可审查性：中
- 审查规则建议：设置"跳过"标志后应确保后续逻辑不被意外执行；对分核数为1的退化场景应有显式处理

### fe0c08a2 fix ut fail in arch64 env
- 根因类别：构建配置问题 -- ARM环境下RPATH导致加载桩函数而非真实实现
- 涉及文件：tests/ut/op_host/CMakeLists.txt
- 缺陷描述：UT在aarch64环境下因RPATH优先从stub加载so而非真实实现，导致MM和Conv UT失败
- 修复模式：设置SKIP_BUILD_RPATH TRUE跳过构建时RPATH
- 可审查性：低
- 审查规则建议：跨平台UT应在x86和ARM环境下同时CI验证

### f5121124 fix bmmv3 weightnz example bug
- 根因类别：API调用错误 -- 使用旧版API缺少参数
- 涉及文件：matmul/batch_mat_mul_v3/examples/test_aclnn_batchmatmul_weight_nz.cpp
- 缺陷描述：示例代码调用旧版API aclnnCalculateMatmulWeightSize，缺少aclDataType参数，导致weight NZ格式的size计算不正确
- 修复模式：替换为V2版本aclnnCalculateMatmulWeightSizeV2并传入ACL_FLOAT16
- 可审查性：中
- 审查规则建议：对已废弃API建立deprecated列表；示例代码也需纳入CI编译验证

### b5127a2d aclnngather fix bug
- 根因类别：整数溢出 -- int存储shape维度乘积
- 涉及文件：index/gather_elements_v2/op_host/gather_elements_v2_last_dim_tiling.h
- 缺陷描述：MergeAxis函数中累乘shape各维度的变量val声明为int(32位)，多维度乘积超过INT32_MAX时溢出
- 修复模式：将int val = 1改为int64_t val = 1
- 可审查性：高
- 审查规则建议：涉及shape维度乘积的变量必须使用int64_t

### f678bf6e 解决n轴取错的问题
- 根因类别：复制粘贴错误 -- n轴和k轴维度索引条件表达式相同
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：UpdateX2NewShape中k_x2_dim和n_x2_dim的条件表达式完全相同，都写成trans_x2 ? (x2_dim_num - 1UL) : (x2_dim_num - 2UL)，但n轴和k轴在转置/非转置下索引应互反
- 修复模式：将n_x2_dim的条件表达式修正为trans_x2 ? (x2_dim_num - 2UL) : (x2_dim_num - 1UL)
- 可审查性：高
- 审查规则建议：相邻两行代码结构相同但语义应不同时必须逐字比对；可检测"连续赋值语句右侧表达式完全相同"

### ee685a90 修复conv infershape ut中context复用问题
- 根因类别：测试代码中context对象复用导致状态污染
- 涉及文件：conv/common/tests/ut/op_host/test_quant_conv3d_dyn_infershape.cpp
- 缺陷描述：UT中InferShape和InferDataType使用同一holder对象获取context，InferShape执行后修改holder内部状态导致InferDataType的context被污染
- 修复模式：为每个InferDataType调用单独创建InferDataTypeContextFaker
- 可审查性：中
- 审查规则建议：UT中不同阶段的推理应使用独立context

### e4c9639a [qbmmv4][a4w4pergroup] fix mem illegal read
- 根因类别：越界内存读取 -- CeilAlign导致DMA读取超出实际数据范围 + 参数传递遗漏
- 涉及文件：matmul/quant_batch_matmul_v4/op_kernel/quant_batch_matmul_v4_pergroup.h
- 缺陷描述：(1)DataCopyPad的blk_len参数使用CeilAlign后的长度，在M/N非对齐整数倍时DMA越界读取；(2)CopyInX1INT8的blockCount硬编码ubCalcM_而非当前块实际值curAivM
- 修复模式：(1)去掉CeilAlign直接使用实际大小；(2)新增curAivM参数替代ubCalcM_
- 可审查性：中
- 审查规则建议：DMA拷贝长度不应使用CeilAlign值(CeilAlign适用于buffer分配非实际拷贝)；函数中用类成员变量替代应传入的局部变量时需警惕尾块场景

### e308f13d fix compile warning
- 根因类别：无符号类型与有符号常量比较 / printf格式符不匹配
- 涉及文件：norm/group_norm_silu/op_host/group_norm_silu_tiling.cpp, norm/group_norm_silu/op_host/group_norm_silu_tiling_arch35.cpp, norm/group_norm_swish_grad/op_host/group_norm_swish_grad_tiling.cpp
- 缺陷描述：(1)uint64_t类型channel/gammaDtypeSize与<0比较恒false无意义；(2)OP_LOGE用%ld打印uint64_t应用%lu
- 修复模式：删除无意义的<0检查；将%ld改为%lu
- 可审查性：高
- 审查规则建议：开启-Wsign-compare和-Wformat并treat as error

### dd6cddd1 修复avgPool3dGrad超大shape报错
- 根因类别：整数溢出 -- int32不足以表示超大shape
- 涉及文件：pooling/avg_pool3_d_grad/op_host/avg_pool_3d_grad_tiling.cpp, pooling/avg_pool3_d_grad/op_kernel/avg_pool3_d_grad_base.h等5个文件
- 缺陷描述：DHWParam结构体的n/d/h/w字段为int(32位)，超大shape时溢出；min/max函数参数、循环变量也是int类型
- 修复模式：全面将int提升为int64_t，包括结构体字段、函数签名、循环变量、日志格式串
- 可审查性：高
- 审查规则建议：与shape/尺寸相关的变量一律使用int64_t

### cf971b07 maxpool修正L2_DFX_PHASE_1
- 根因类别：DFX日志宏放置位置错误
- 涉及文件：pooling/max_pool3d_with_argmax_v2/op_host/op_api/aclnn_max_pool2d_with_indices.cpp
- 缺陷描述：L2_DFX_PHASE_1宏被放在if分支内部，kH==1且kW==1或SoC版本不匹配时走else分支不记录DFX日志
- 修复模式：将L2_DFX_PHASE_1调用移到if判断之前
- 可审查性：高
- 审查规则建议：DFX/日志宏应放在条件分支之前确保全路径覆盖

### 9392a5db Tbmm Init Buffer Fix
- 根因类别：buffer初始化方式与执行模型不匹配
- 涉及文件：matmul/transpose_batch_mat_mul/op_kernel/pp_matmul_ein_sum_kernel.h等3个文件
- 缺陷描述：通过TPipe.InitBuffer分配片上缓存，但算子不使用TPipe流水线调度，pipe作为局部变量销毁后buffer引用可能悬空
- 修复模式：用OnChipBuffer替代AsdopsBuffer，直接通过LocalTensor构造函数手动映射片上存储，移除TPipe依赖
- 可审查性：低
- 审查规则建议：片上buffer初始化方式需与算子执行模型匹配；不使用TPipe流水线时不应通过TPipe分配buffer

### 7435f6a6 [conbp]fix singleShapebatch bug
- 根因类别：计算表达式错误 -- 切分数计算分子取错 + 整数溢出
- 涉及文件：conv/conv3d_backprop_filter_v2/op_kernel/arch35/conv3d_backprop_filter_v2/conv3d_dw_v2_basic_block.h
- 缺陷描述：(1)totalAddSize中batch_*dout_*ho_*wo_乘法缺少uint64_t类型提升可能溢出；(2)singleShapeBatch_计算使用Ceil(singleCoreBatch_, batchdoutPerBlock_)但应使用总量batch_*dout_
- 修复模式：(1)对batch_做static_cast<uint64_t>；(2)将Ceil参数从singleCoreBatch_改为batch_*dout_
- 可审查性：中
- 审查规则建议：Ceil(总量, 块大小)的"总量"参数需确认语义——是总任务量而非单核任务量

### 5984e340 fix gen_op command bug
- 根因类别：路径拼写错误(typo)
- 涉及文件：build.sh
- 缺陷描述：gen_op函数中复制模板CMakeLists.txt时源目录写成example/而非examples/(缺少s)，两处重复错误
- 修复模式：将两处example/CMakeLists.txt改为examples/CMakeLists.txt
- 可审查性：高
- 审查规则建议：shell脚本中硬编码路径应核实与实际目录结构匹配

### 4d562156 修复gemmv3UT
- 根因类别：构建依赖缺失 + 测试期望值错误
- 涉及文件：matmul/gemm_v3/op_host/CMakeLists.txt, matmul/gemm_v3/tests/ut/op_host/test_gemm_v3_tiling.cpp
- 缺陷描述：(1)CMakeLists.txt缺少DEPENDENCIES mat_mul_v3声明导致链接失败；(2)tiling UT的golden data中一个数值错误(1应为2)
- 修复模式：补充DEPENDENCIES声明；修正UT期望值
- 可审查性：低
- 审查规则建议：超长magic number字符串形式的golden data应重构为结构化数据

### f6165622 fix compile warning
- 根因类别：未使用参数导致编译告警
- 涉及文件：foreach/foreach_non_finite_check_and_unscale/op_host/foreach_non_finite_check_and_unscale_base_tiling.cpp, foreach/foreach_utils/op_host/foreach_tiling_func.cpp
- 缺陷描述：两个TilingPrepare函数的context参数完全未使用，开启-Wunused-parameter时产生告警
- 修复模式：对context参数添加[[maybe_unused]]属性
- 可审查性：高
- 审查规则建议：启用-Wunused-parameter -Werror

(跳过) aa794631 回退//qbmm - Revert提交，留待阶段3分析
(跳过) 7a065a17 fix AvgPool3dGrad README - 纯文档修复

### bb5aad58 增加越界保护
- 根因类别：边界条件缺失
- 涉及文件：loss/cross_entropy_loss/op_host/cross_entropy_loss_tiling_arch35.cpp
- 缺陷描述：countOnes(uint64_t num)函数在num=0时进入while循环后返回count-1=-1，语义不正确。0的二进制中有0个1，减1后变为-1，若后续用作数组索引或除数会导致越界或未定义行为。
- 修复模式：前置guard clause，在函数入口对num==0提前返回0
- 可审查性：高
- 审查规则建议：对位操作/数学函数，静态检查是否处理了零值和边界输入(0、UINT64_MAX等)

(跳过) ac976d4b mmv3, bmmv3, gemmv2 tiling key revert - Revert提交，留待阶段3分析

### 8c91e2a0 fix conv ut
- 根因类别：头文件include路径错误
- 涉及文件：6个conv相关UT文件(test_aclnn_convolutin_backward.cpp等)
- 缺陷描述：UT文件中#include使用裸文件名，依赖构建系统的隐式include path配置。当include path变化后编译失败。
- 修复模式：将隐式include路径改为显式相对路径(../../../op_host/op_api/...)
- 可审查性：高
- 审查规则建议：UT文件中#include应使用相对路径或明确的项目内路径，禁止依赖隐式include path定位项目内部头文件

### 5506cf24 ophost warning fix
- 根因类别：编译警告(printf格式符不匹配 + 未使用变量/函数 + UT断言注释掉)
- 涉及文件：17个matmul相关文件
- 缺陷描述：批量消除编译警告：(1)大量%zu用于打印int64_t类型，修复为%ld/%lu；(2)删除3个未被调用的静态辅助函数(39行死代码)；(3)注释掉一个UT断言ASSERT_EQ(tiling_data_result, golden_tiling_data)——这意味着tiling正确性不再被验证，可能掩盖回归缺陷
- 修复模式：格式说明符修正 + 死代码清理 + trailing whitespace清理
- 可审查性：中
- 审查规则建议：启用-Wformat -Wformat-signedness；UT中注释掉ASSERT_EQ断言是危险操作，review时应重点关注

### 4aba3129 fastgelu和elugrad错误examples代码修改
- 根因类别：示例代码复制粘贴错误
- 涉及文件：activation/elu_grad_v2/examples/test_aclnn_elu_backward.cpp、activation/fast_gelu/examples/test_aclnn_fast_gelu.cpp、activation/fatrelu_mul/tests/ut/op_kernel/fatrelu_mul_tiling_def.h
- 缺陷描述：两个算子的example文件内容被交换了——elu_backward的example里写的是fast_gelu的调用代码，反之亦然。从创建之日起就是错误的。另外fatrelu_mul_tiling_def.h的include guard名称与文件名不匹配。
- 修复模式：内容互换修正 + include guard命名规范化
- 可审查性：高
- 审查规则建议：示例代码中#include的头文件名应与所在目录的算子名一致；include guard应与文件名匹配；example代码应有CI编译验证

### e00d9aa0 UT bug fix
- 根因类别：构建系统逻辑缺陷(CMake文件查找逻辑错误)
- 涉及文件：cmake/ut.cmake
- 缺陷描述：原代码使用execute_process+stat在CMake configure阶段检测生成文件是否存在，但这些文件标记为GENERATED TRUE，configure阶段尚不存在，检测结果取决于上次构建残留文件。
- 修复模式：将运行时文件探测替换为编译期静态白名单匹配(IN_LIST)
- 可审查性：高
- 审查规则建议：CMake中对GENERATED文件不应使用execute_process做存在性检测

### cfd79187 fix group_norm_swish ut
- 根因类别：UT框架迁移不完整
- 涉及文件：norm/group_norm_swish/tests/下9个文件
- 缺陷描述：UT框架整体迁移修复：(1)CMake GLOB RELATIVE路径解析失败改用LIST_DIRECTORIES true；(2)add_modules_llt_sources迁移到add_modules_ut_sources；(3)ge::AnyValue迁移到Ops::NN::AnyValue；(4)include路径迁移；(5)删除重复的TilingData结构体定义；(6)using namespace std改为显式std::；(7)op_kernel CMake从return()改为AddOpTestCase
- 修复模式：框架迁移适配(批量API/命名空间/构建宏替换)
- 可审查性：低
- 审查规则建议：框架迁移应有checklist逐项验证；UT中手动复制结构体定义应触发告警

(跳过) c86c4af4 fix pooling README - 纯文档修复

### c515aa98 matmul compile warning fix
- 根因类别：编译警告中包含实际逻辑bug(链式比较 + 运算符优先级 + 类型转换方向错误)
- 涉及文件：5个matmul相关文件(aclnn_batch_matmul.cpp、aclnn_gemm.cpp、aclnn_quant_matmul_checker.cpp等)
- 缺陷描述：(1)链式比较a==b==1在C++中等价于(a==b)==1即判断a和b是否相等，而非"两个都等于1"；(2)if条件中&&和||混合使用缺少括号导致优先级错误；(3)static_cast<uint64_t>(int64_t_value)将有符号转为无符号，负值会溢出为巨大数值；(4)删除5处未使用变量
- 修复模式：修正运算符优先级/链式比较逻辑 + 修正类型转换方向(统一转int64_t) + 清理dead code
- 可审查性：高
- 审查规则建议：连续使用==运算符(a==b==c)视为高危模式；if条件混合&&和||时必须显式括号(-Wparentheses)；static_cast<uint64_t>(signed)应触发审查

(跳过) adebf652 fix aclnnAvgPool3DGrad.md - 纯文档修复
(跳过) 9ac31cc5 回退//【1952mm回合主线】wqbmm - Revert提交，留待阶段3分析

### 7d532e6c fix ut
- 根因类别：UT测试数据与tiling参数不匹配
- 涉及文件：vfusion/scaled_masked_softmax_grad_v2/tests/ut/下3个文件
- 缺陷描述：(1)测试shape参数过大(S=512,D=2048)改为合理值(S=64/32,D=1536/256)，coreNum从48改为40匹配实际硬件；(2)tiling数据从代码直接构造改为从bin文件反序列化，使UT更接近真实运行场景
- 修复模式：缩小测试shape + 新增tiling数据生成脚本 + 修改tiling数据读取方式
- 可审查性：低
- 审查规则建议：UT中kernel的tiling数据应通过与生产代码相同的序列化路径生成

### 584223ca fixed torch dynamic graph compile fail
- 根因类别：宏定义顺序错误/头文件include顺序依赖
- 涉及文件：matmul/quant_batch_matmul_v4/op_kernel/quant_batch_matmul_v4_apt.cpp
- 缺陷描述：float32场景下，#define DTYPE_X2 fp4x2_e2m1_t放在文件末尾，但消费该宏的catlass_convertor.h在文件开头已被include，导致头文件中DTYPE_X2仍为原始值，类型不匹配编译失败
- 修复模式：调整宏定义和头文件include顺序，确保宏在被消费前已正确定义
- 可审查性：高
- 审查规则建议：当代码中存在#undef/#define覆盖宏时，审查需检查所有消费该宏的头文件是否在宏重定义之后才被include

### ed08f259 fix tilingkey
- 根因类别：条件编译分支遗漏/平台适配缺陷
- 涉及文件：matmul/quant_batch_matmul_v3/op_kernel/quant_batch_matmul_v3.cpp、quant_batch_matmul_v3_tiling_key.h
- 缺陷描述：quant_batch_matmul_v3一个特定kernel分支的代码不在任何__CCE_AICORE__保护下，导致AI Core 220平台缺失该分支的tiling key注册，运行时找不到对应tiling key而失败
- 修复模式：将平台无关代码正确归入各平台条件编译分支，为AI Core 220补充kernel实例化和tiling key注册
- 可审查性：中
- 审查规则建议：多平台条件编译(__CCE_AICORE__ == 200/220)应系统性对比两个平台分支是否覆盖所有相同kernel模板组合

(跳过) ec96530d 回退【1952mm回合主线】mm/bmm - Revert提交，留待阶段3分析

### cf4f58ff qbmm wqbmm ut fix
- 根因类别：UT stub函数残留 + 测试数据不匹配
- 涉及文件：matmul/quant_batch_matmul_v3/tests/ut/、weight_quant_batch_matmul_v2/tests/ut/、tests/ut/common/ut_op_common.cpp
- 缺陷描述：(1)ut_op_common.cpp中硬编码了CheckSupportConditionQbmm和GenWqbmmTiling两个stub函数(总返回true)，让依赖它们的UT永远走不到真实逻辑；(2)删除4条与算子实际tiling结果不匹配的测试参数
- 修复模式：删除无效stub函数 + 删除不匹配的测试参数
- 可审查性：中
- 审查规则建议：公共UT工具文件中不应包含特定算子的stub实现；删除测试用例时必须说明原因

### 8e9b137c fix example bugs of conv3d_transpose_v2
- 根因类别：CMake依赖声明错误
- 涉及文件：conv/conv3d_transpose_v2/op_host/CMakeLists.txt
- 缺陷描述：DEPENDENCIES字段写成convolution_backward，正确的应为convolution_forward。conv3d_transpose(反卷积)的前向计算依赖convolution的forward路径而非backward路径。
- 修复模式：单词替换 convolution_backward -> convolution_forward
- 可审查性：高
- 审查规则建议：CMake DEPENDENCIES变更时验证依赖关系的语义正确性

### 35c6053f [convbp]fix compile warning logs
- 根因类别：类型不匹配 + 函数调用缺少参数
- 涉及文件：conv/conv3d_backprop_input_v2/op_host/op_tiling/下2个文件
- 缺陷描述：(1)NUM_THREE常量声明为uint32_t但与int32_t混合运算导致有符号/无符号比较警告；(2)两个局部变量声明后未使用；(3)IsArchAfter35调用缺少context参数——不传context可能导致架构判断错误
- 修复模式：类型修正 + 删除未使用变量 + 补全函数调用参数
- 可审查性：高
- 审查规则建议：函数调用缺少参数通常是重构时遗漏，对IsArchAfter35调用点做全局搜索确认一致性

### 15851ff1 fix UT bug
- 根因类别：UT基础设施迁移不完整 + proto头文件语法错误 + tiling结构不匹配
- 涉及文件：index/scatter_elements_v2/下4个文件
- 缺陷描述：(1)scatter_elements_v2_proto.h中OP_END_FACTORY_REG后缺少namespace闭合花括号；(2)infershape UT注册被注释掉导致不会编译执行；(3)测试代码引用错误的tiling头文件和类型名；(4)gen_tiling.py中tiling字段数量与实际结构体不一致(旧14字段vs新30+字段)；(5)删除5个冗余测试用例
- 修复模式：语法修复 + 路径修正 + 类型名对齐 + tiling数据结构同步
- 可审查性：中
- 审查规则建议：tiling结构体变更时必须同步更新所有引用该结构的UT数据生成脚本

### 66100677 SMS/SMSG ut fix
- 根因类别：UT框架迁移(LLT→新UT框架)多处适配问题
- 涉及文件：vfusion/scaled_masked_softmax_grad_v2/和scaled_masked_softmax_v2/下16个文件
- 缺陷描述：大规模UT框架迁移：(1)CMake条件从add_modules_llt_sources改为add_modules_ut_sources；(2)旧版CMake以return()开头直接跳过所有构建(kernel UT从未执行)；(3)头文件路径迁移；(4)删除硬编码的#define __CCE_AICORE__ 220；(5)gen_data.py中使用未定义变量dtype；(6)调试日志残留——softmax模块打印了is_finite的日志(复制粘贴未修改)
- 修复模式：框架迁移适配(构建系统 + 头文件路径 + API名称 + 数据生成逻辑)
- 可审查性：低
- 审查规则建议：CMake中return()直接跳过构建是极大红旗；调试日志模块名应与实际模块匹配

(跳过) 61a1f158 revert identity/identity_n/rank/shape/shape_n - Revert提交，留待阶段3分析

### a87c18f0 qbmm opensource fix
- 根因类别：架构重构/开源合规(硬编码白名单暴露)
- 涉及文件：common/stub/op_tiling/op_cache_def_tiling.h、quant_batch_matmul_v3相关tiling文件、ut_op_common.cpp
- 缺陷描述：QuantBatchMatmulV3的tiling逻辑中大量硬编码shape白/黑名单直接写死在cpp文件里。开源时这些与特定硬件平台绑定的shape list不应暴露在源码中。
- 修复模式：抽象接口封装——新增QuantBatchMatmulRunParas结构体和QbmmType枚举，将所有白/黑名单判断统一收敛到CheckSupportConditionQbmm() weak symbol函数中，stub固定返回true
- 可审查性：中
- 审查规则建议：源码中硬编码的std::set<std::tuple<...>>形式的shape常量应抽象为配置或接口；weak symbol stub返回值需审查是否符合预期行为

### a6be209a fix issues
- 根因类别：文档路径错误 + 示例代码硬编码
- 涉及文件：docs/context/quick_op_invocation.md、matmul/quant_batch_matmul_v4/examples/test_aclnn_quant_matmul_v5_at2_at3.cpp
- 缺陷描述：(1)文档中pip3 install路径tests/requirements.txt错误，应为requirements.txt；(2)示例代码deviceId=1硬编码非零设备号，单卡环境直接失败
- 修复模式：值修正——路径修正 + deviceId从1改为0
- 可审查性：高
- 审查规则建议：文档中引用的文件路径应自动校验是否存在；示例代码deviceId默认值应为0

### 5bb873eb fix cleancode
- 根因类别：代码规范/const正确性
- 涉及文件：cmake/gen_ops_info.cmake、conv/convolution_backward/op_host/op_api/convolution_backward_checker.cpp/.h
- 缺陷描述：IsConv8bit方法不修改成员状态但未声明为const，导致const上下文中无法调用。另删除cmake调试日志。
- 修复模式：const修饰符补全 + 冗余日志清理
- 可审查性：高
- 审查规则建议：不修改成员的方法应声明为const(clang-tidy readability-make-member-function-const)

### 2e985117 解决group_quant再910_93编译问题
- 根因类别：文件命名错误导致编译失败
- 涉及文件：quant/group_quant/op_host/config/ascend910_93/group_quant_simplified_key_mode.ini(重命名为group_quant_simplified_key.ini)
- 缺陷描述：配置文件名group_quant_simplified_key_mode.ini多了_mode后缀，构建系统查找的是group_quant_simplified_key.ini，文件名不匹配导致910_93平台编译失败
- 修复模式：文件重命名去掉多余_mode后缀
- 可审查性：高
- 审查规则建议：新增配置文件时CI应在所有目标平台上验证编译；配置文件命名应有规范文档

### 2aced33a fix foreach compile error
- 根因类别：宏接口签名不一致
- 涉及文件：foreach/foreach_lerp_scalar_def.cpp、foreach_pow_scalar_and_tensor_def.cpp、foreach_round_off_number_def.cpp、foreach/foreach_utils/op_host/foreach_proto_utils.h
- 缺陷描述：头文件中宏FOREACH_OPDEF_PARAM_SCALAR接受3个参数，但调用侧与内部变量名存在耦合。另外FOREACH_BINARY_LIST_ALPHA_PARAM中参数名scalar应为alpha。
- 修复模式：宏接口重构——减少宏参数个数消除耦合；参数名语义修正
- 可审查性：中
- 审查规则建议：宏定义变更时全量搜索所有调用点确认参数个数同步更新；宏内部依赖外部作用域局部变量是脆弱设计

### 0ebd992a 修复example编译找不到头文件的问题
- 根因类别：构建系统include路径缺失
- 涉及文件：build.sh, cmake/opbuild.cmake, cmake/variables.cmake
- 缺陷描述：头文件安装路径不正确，导致example编译时找不到aclnn头文件。build.sh中缺少`-I ${INCLUDE_PATH}/aclnnop`，cmake中未定义`ACLNN_OP_INC_INSTALL_DIR`且未将头文件安装到`aclnnop`子目录
- 修复模式：在构建系统中新增include路径变量并同步安装头文件到多个路径（兼容不同引用方式）
- 可审查性：高
- 审查规则建议：cmake install配置变更时检查是否覆盖了所有必要路径；build脚本中`-I`参数需包含完整搜索路径

### b2f36968 fix group-norm include path
- 根因类别：include路径错误
- 涉及文件：norm/group_norm/op_host/op_api/aclnn_group_norm.cpp
- 缺陷描述：`#include "../../../norm/group_norm_silu/..."`使用过深相对路径引用，路径不正确。修复为`#include "norm/group_norm_silu/..."`使用项目include路径
- 修复模式：#include相对路径`../../../`错误 -> 改用基于项目根的include路径
- 可审查性：高
- 审查规则建议：检测`#include`中使用3层以上`../`的过深相对路径引用，推荐使用基于项目根目录的路径

### 86c5fb8f kernelut&readme fix
- 根因类别：CMake变量引用错误
- 涉及文件：cmake/func.cmake, cmake/ut.cmake, 多个CMakeLists.txt, gen_tiling_head_file.py
- 缺陷描述：`cmake/func.cmake`中`set(OP_TYPE ...)`应为`set(${OP_TYPE} ...)`——函数参数作为变量名传入`set`时需要解引用，否则设置的是字面量"OP_TYPE"而非调用者传入的变量名，导致函数返回值丢失。另外ut.cmake中opType推断从下划线分词大小写转换改为从binary.json读取，避免非规则命名出错
- 修复模式：CMake `set(VAR_NAME ...)` vs `set(${VAR_NAME} ...)`解引用遗漏，导致输出变量名写死
- 可审查性：高
- 审查规则建议：CMake函数中`set()`第一个参数如果来自函数参数，必须用`${}`解引用

### 7cc5ae81 算子注册原型bugfix
- 根因类别：语法结构不完整
- 涉及文件：control/assert/op_graph/assert_proto.h
- 缺陷描述：`OP_END_FACTORY_REG(Assert)`后缺少namespace闭合`}`导致编译失败
- 修复模式：proto.h文件缺少右大括号——namespace/struct闭合不完整
- 可审查性：高
- 审查规则建议：proto.h文件中检查namespace/struct/class的大括号匹配（此类缺陷在ops-nn-dev中多次出现）

### 723a43ae fix warning for conv3dv2 0924
- 根因类别：编译告警修复（含实质性改动）
- 涉及文件：conv3d_api_tiling_algorithm_hw_mode.cpp/.h, aclnn_convolution.cpp, aclnn_quant_convolution.cpp
- 缺陷描述：(1)头文件include依赖传播过广，从header移到cpp；(2)缺少override关键字约20处；(3)C风格强制转换未改static_cast；(4)magic number 4未替换为CONV_2D_DIM_SIZE常量；(5)未使用的stride参数未移除
- 修复模式：大规模override/const/C-cast/magic number告警修复
- 可审查性：高
- 审查规则建议：虚函数重写必须带override；禁止C风格cast，使用static_cast/reinterpret_cast

### 63bee42e fix maxpool3dwithargmaxv2 kernel select
- 根因类别：模板类型安全
- 涉及文件：max_pool3d_with_argmax_v2_base.h, *_nosplit.h, *_splitd.h, *_splith.h, *_splitw.h
- 缺陷描述：`Select<T>()`调用中scalar参数使用`1.f * kernelDHW`或`-1.f`（float字面量），但模板T可能是half等非float类型。T为half时传float给Select的scalar参数导致类型不匹配
- 修复模式：模板函数scalar参数使用固定float字面量而非(T)显式转换，当T!=float时类型不匹配
- 可审查性：高
- 审查规则建议：模板函数中数值字面量（1.f, -1.f等）传给模板类型T参数时，须显式转换`(T)value`

### 092b884e fix cleancode
- 根因类别：命名空间污染
- 涉及文件：conv3d_backprop_filter_v2_base_tiling.h, conv3d_backprop_filter_v2_basic_block_tiling.h
- 缺陷描述：头文件中`using namespace`位于全局作用域，会污染所有包含此头文件的编译单元的命名空间，可能导致名称冲突
- 修复模式：将`using namespace`从全局作用域移到具体命名空间内部
- 可审查性：高
- 审查规则建议：头文件(.h)中禁止全局作用域的`using namespace`声明

### c6726c69 修复CCE裸指令问题，clean code问题
- 根因类别：CCE裸指令使用（硬件兼容性风险）
- 涉及文件：17个文件，涉及chamfer_distance_grad, cross_entropy_loss, ctc_loss_v3_grad, advance_step, apply_adam_w_v2, multi_scale_deformable_attn_function
- 缺陷描述：kernel代码中使用`pipe_barrier(PIPE_ALL)`等CCE裸指令（底层硬件指令），替换为AscendC高级API `PipeBarrier<PIPE_ALL>()`。裸指令可移植性差，不同硬件版本行为可能不一致
- 修复模式：`pipe_barrier()` -> `PipeBarrier<>()`，CCE裸指令统一替换为AscendC API
- 可审查性：高
- 审查规则建议：kernel代码中检测CCE裸指令（pipe_barrier, set_flag等），应使用AscendC高级API替代

### b58b066f LayerNormGradV3日志等级错误修复
- 根因类别：日志等级误用
- 涉及文件：layer_norm_grad_v3_common_tiling.cpp, layer_norm_grad_v3_single_read_tiling.cpp, layer_norm_grad_v3_transpose_tiling.cpp
- 缺陷描述：`IsCapable()`方法中当某tiling模板不适用时使用`OP_LOGE`(ERROR级别)，但这不是错误——只是当前模板不适用会继续尝试下一个。ERROR日志误导用户
- 修复模式：`OP_LOGE` -> `OP_LOGI`，正常执行路径中ERROR日志应降级为INFO
- 可审查性：高
- 审查规则建议：IsCapable()/capability check方法中return false路径使用OP_LOGE是常见错误，应为OP_LOGI或OP_LOGD

### a9eab151 swi_glu_grad arch35 kernel fix
- 根因类别：C++模板语法错误
- 涉及文件：activation/swi_glu_grad/op_kernel/arch35/swi_glu_grad_ub_rearrange.h
- 缺陷描述：`template <typename T> struct VciTypeGet<> {};`语法错误——主模板声明后跟了空特化参数列表`<>`，应为`template <typename T> struct VciTypeGet;`
- 修复模式：模板主声明与部分特化语法混淆
- 可审查性：高
- 审查规则建议：检测`template <typename T> struct/class Name<> {}`模式——这是无效的空特化参数列表

### 305aac24 fix build.sh
- 根因类别：构建脚本参数校验过严
- 涉及文件：build.sh
- 缺陷描述：build.sh对非识别参数的错误处理逻辑过于严格，将合法参数误判为非法并报错退出
- 修复模式：条件判断逻辑过严误拒合法输入（此模式在ops-nn-dev多次出现：错误校验逻辑过早拒绝合法输入）
- 可审查性：中
- 审查规则建议：参数校验的error分支应列举所有合法情况，避免未来新增参数时被误拒

### ed871d87 修复conv编译告警
- 根因类别：编译告警修复（含实质性缺陷）
- 涉及文件：conv_forward_infershape.cpp, conv3d_api_tiling_algorithm.h, aclnn_convolution.cpp, aclnn_quant_convolution.cpp, convolution.cpp
- 缺陷描述：(1)`OpParamIdx`结构体成员未初始化（添加`= 0`），可防止未定义行为；(2)`dimIdx < 0`对`size_t`(unsigned)永远为false，删除无意义检查；(3)删除未使用模板函数`All<T,Func>()`；(4)大量unused变量清理；(5)CheckPad/CheckPadDepthwise2d签名去掉未使用的stride参数
- 修复模式：POD结构体成员未初始化 + unsigned与0比较 + unused参数/变量
- 可审查性：高
- 审查规则建议：POD/聚合结构体成员必须初始化；unsigned类型与`< 0`比较是逻辑错误（编译器-Wsign-compare可检测）

### d87304ad fix sc
- 根因类别：编译告警修复（批量）
- 涉及文件：27个文件，涉及activation, common, index, loss, matmul, pooling, quant
- 缺陷描述：(1)头文件static函数缺inline导致ODR violation；(2)cross_entropy_loss_tiling中`while((a >>= 1ULL) && a!=0ULL)`隐式bool转换告警，改为显式`static_cast<bool>`；(3)cross_v2_tiling添加nullptr检查——防御性编程改进；(4)函数参数添加const修饰；(5)未使用参数移除
- 修复模式：头文件static函数缺inline(ODR) + 隐式bool转换 + nullptr防御
- 可审查性：中
- 审查规则建议：头文件中static函数应加inline避免ODR问题

### d108eaae fix warning for conv3dv2 0922
- 根因类别：编译告警修复（大规模）
- 涉及文件：8个conv3d相关文件
- 缺陷描述：与723a43ae(0924)属同一批次修复的前序版本。(1)include依赖重组——header中移除不必要include；(2)约20处虚函数重写缺override；(3)约18处PreProcess/Impl/PostProcess缺override；(4)删除未使用的模板函数；(5)大量const修饰添加
- 修复模式：大规模override缺失修复（同一文件aclnn_convolution.cpp在2天内分2次修复，说明问题范围大且遗漏多）
- 可审查性：高
- 审查规则建议：同723a43ae——虚函数重写必须带override（编译器-Wsuggest-override可检测）

跳过：239b38a2(示例变量名错误), dc1d5a96(日志文本补充), d8774743(README/文档修复), 9d40cbc4(文档链接修复), 68c6a0ad(示例修复), 1fcd5463(示例修复), 1f3a2268(示例修复), 1462b675(代码重构-文件重命名无逻辑变更), 979d049c(构建系统占位注释workaround), 7539638d(清理删除空文件), 38b87118(license修复), cccda0de(示例文件重命名)

### 3e7d1fc4 tbmm warning问题修复
- 根因类别：内存管理API不规范
- 涉及文件：transpose_batch_mat_mul下pp_matmul_ein_sum_kernel.h, transpose_batch_mat_mul.cpp, utils/mem.h
- 缺陷描述：tbmm算子通过InitBuffer/address_.logicPos手动初始化tensor内存，绕过TPipe管理。新版编译器产生warning且存在内存管理不规范问题。修复为通过TPipe::InitBuffer+TBuf标准API，并添加pipe.Destroy()生命周期管理
- 修复模式：裸内存初始化 -> TPipe标准管道管理
- 可审查性：中
- 审查规则建议：检测kernel中直接操作address_.logicPos的非标准内存初始化模式

### f121a9e4 fix kernel ut source files dependencies issue
- 根因类别：CMake构建依赖缺陷
- 涉及文件：cmake/ut.cmake, tests/ut/op_host/CMakeLists.txt
- 缺陷描述：(1)kernel UT中file(GLOB)在cmake配置阶段查找.cpp文件，但该文件是构建阶段生成的，配置时不存在导致依赖缺失。(2)op_host UT硬编码链接static_lib target，当无用例时target不存在导致增量编译失败
- 修复模式：file(GLOB)替换为显式set+GENERATED属性+target_sources延迟指定；用$<TARGET_EXISTS:>守卫条件链接
- 可审查性：高
- 审查规则建议：禁止file(GLOB)获取构建时生成的源文件；链接可选target用$<TARGET_EXISTS:>守卫

### d490c4b4 fix soc package
- 根因类别：打包系统配置冗余
- 涉及文件：cmake/makeself_built_in.cmake, cmake/package.cmake, cmake/runtimeKB.cmake, scripts/package下多个XML和安装脚本
- 缺陷描述：每个soc_version独立维护runtimeKB cmake、打包XML和安装脚本，新增soc需大量复制。合并为统一参数化配置
- 修复模式：消除soc维度的重复配置，合并为参数化统一配置
- 可审查性：中
- 审查规则建议：检测scripts/package下按soc维度冗余的配置文件

### 839fce05 fix warning for conv3dv2
- 根因类别：编译告警修复
- 涉及文件：conv相关9个文件
- 缺陷描述：(1)模板constexpr比较宏展开优先级需加括号；(2)虚函数覆盖缺override；(3)static const在header应为inline const；(4)内部链接函数缺匿名namespace
- 修复模式：加括号/override/inline/匿名namespace
- 可审查性：高
- 审查规则建议：-Wsuggest-override, -Wparentheses, header中static const非constexpr变量应改inline

### 5ed81216 修复增量编译失败bug
- 根因类别：增量编译依赖解析不完整
- 涉及文件：scripts/util/parse_changed_files.py, parse_compile_changed_files.py
- 缺陷描述：反向依赖(reverse_op_dependencies)获取后直接附加到编译依赖列表，但反向依赖本身的编译依赖(二级依赖)未被检索，导致编译时缺少必要依赖
- 修复模式：两步依赖解析——先获取反向依赖，再以反向依赖为输入获取其编译依赖
- 可审查性：高
- 审查规则建议：依赖解析需传递闭包，不能只取一级依赖

### 743fc6ac addrmsnormquantv2精度修复
- 根因类别：模板参数组合下的精度损失
- 涉及文件：norm/add_rms_norm_quant/op_kernel/add_rms_norm_quant.h
- 缺陷描述：ComputePart2中当TX为half时执行fp32->fp16降精度Cast，但模板参数PT=true(高精度路径)时不应降精度。条件`is_same<TX, half>::value`缺少`&& !PT`判断
- 修复模式：`is_same<TX, half>::value` -> `is_same<TX, half>::value && !PT`
- 可审查性：高
- 审查规则建议：新增模板参数后需review所有conditioned-on-type的Cast分支

### c4f2f27e compile mmv3 david fix
- 根因类别：算子配置遗漏
- 涉及文件：matmul/mat_mul_v3/op_host/mat_mul_v3_def.cpp
- 缺陷描述：mat_mul_v3在ascend910_95配置中缺少opFile.value和aclnnSupport.value扩展配置，导致编译找不到正确kernel文件
- 修复模式：添加ExtendCfgInfo("opFile.value", ...).ExtendCfgInfo("aclnnSupport.value", ...)
- 可审查性：高
- 审查规则建议：新增soc配置时检查是否包含完整的ExtendCfgInfo(opFile, aclnnSupport等)

### c097c981 bugfix_crossv2
- 根因类别：算子注册遗漏
- 涉及文件：loss/cross_v2/op_host/cross_v2_tiling.cpp
- 缺陷描述：CrossV2算子tiling注册缺少TilingParse，IMPL_OP_OPTILING只注册了Tiling函数未链式调用.TilingParse<>()，运行时找不到CompileInfo解析入口
- 修复模式：补充空TilingPrepare函数并在注册链追加.TilingParse<CrossV2CompileInfo>(...)
- 可审查性：高
- 审查规则建议：所有IMPL_OP_OPTILING注册必须包含.TilingParse调用（此模式在ops-nn-dev多次出现）

### 6f6bdfdc addprermsnormquant精度问题修复
- 根因类别：模板参数组合下的精度损失
- 涉及文件：norm/add_rms_norm_quant/op_kernel/add_rms_norm_quant.h, add_rms_norm_quant_split_d.h
- 缺陷描述：与743fc6ac同一算子同类问题。bf16输入有beta场景中，PT=true时数据已是fp32却仍执行bf16->fp32 Cast(先truncate再cast)导致精度损失
- 修复模式：`if (hasBeta)` -> `if (hasBeta && !PT)`
- 可审查性：高
- 审查规则建议：同743fc6ac

### 4753a05e rms_norm_quant解决内存检测问题
- 根因类别：内存越界 + 流水线同步缺失
- 涉及文件：norm/rms_norm/op_kernel/rms_norm_base.h, norm/rms_norm_quant/op_kernel/rms_norm_quant.cpp
- 缺陷描述：三重根因：(1)DataCopy/Cast/Mul等操作使用对齐后大小num_col_align_f16而非实际num_col_，读写越界；(2)手动repeat参数计算不处理尾部对齐，改用单参数API；(3)Cast/Mul之间缺PipeBarrier<PIPE_V>，ReduceSum后缺SetFlag/WaitFlag<V_S>
- 修复模式：用实际大小替换对齐大小；简化向量API；补充pipeline同步
- 可审查性：高
- 审查规则建议：DataCopy元素数量不应用align后的大小；向量操作间需PipeBarrier同步

### 45f4afd6 修复transpose 511 case错误
- 根因类别：边界常量设置过小
- 涉及文件：conv3d_backprop_input_v2的tiling/kernel/test等文件
- 缺陷描述：filter维度H/W上限kFilterDimHWUp设为255，实际应为511，导致filter 256-511范围的case被错误拒绝
- 修复模式：constexpr kFilterDimHWUp = 255 -> 511
- 可审查性：高
- 审查规则建议：边界常量修改需与硬件spec/算子协议文档对齐验证

### f14030e7 fmm opapi ut fix
- 根因类别：UT include路径错误
- 涉及文件：matmul/fused_mat_mul/tests/ut/op_host/test_aclnn_fused_matmul.cpp
- 缺陷描述："common/op_api_def.h"不正确，应为"op_api/op_api_def.h"
- 修复模式：修正UT include路径
- 可审查性：高
- 审查规则建议：UT的include路径应与产品代码保持一致

### 7876dbde pkg打包报错修复以及增量编译修复
- 根因类别：构建脚本多重缺陷
- 涉及文件：build.sh, cmake/package.cmake, scripts/util/parse_changed_files.py, parse_compile_changed_files.py
- 缺陷描述：(1)CI模式下parse脚本输出未过滤就添加到TRIGER_UTS导致触发不存在的target；(2)package.cmake对所有soc都include runtimeKB但只有部分soc有文件；(3)os.path.exists缺strip()导致检查失败
- 修复模式：过滤op_*前缀；soc条件判断；strip()处理
- 可审查性：高
- 审查规则建议：文件路径字符串处理前必须strip()；CI脚本输出解析需有格式验证

### 1344a3b3 修复没有aclnn代码编译失败的问题
- 根因类别：CMake target不存在时未守卫
- 涉及文件：build.sh, cmake/opbuild.cmake, cmake/package.cmake, cmake/symbol.cmake
- 缺陷描述：(1)无aclnn代码时opapi target不存在，package.cmake/symbol.cmake无条件引用导致cmake报错；(2)opbuild.cmake未收集带版本号的生成文件(如aclnn_xxx_v2.cpp)
- 修复模式：if(TARGET ...)守卫；file(GLOB)匹配版本号变体
- 可审查性：高
- 审查规则建议：cmake引用target前用if(TARGET)或$<TARGET_EXISTS:>守卫

### 05ea548c fix bias optional range
- 根因类别：optional输入API误用
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：bias是optional输入，但使用GetInputShapeRange按物理索引取值，bias不存在时取到错误输入。应使用GetOptionalInputShapeRange
- 修复模式：GetInputShapeRange -> GetOptionalInputShapeRange
- 可审查性：高
- 审查规则建议：optional输入必须使用GetOptionalInput*/GetOptionalInputShapeRange API

### ebdf8b70 aclnn ut 修复不带asan卡死问题
- 根因类别：UT框架依赖真实硬件初始化
- 涉及文件：tests/ut/op_api下43个文件，ut_main.cpp，新增stub/opdev/platform.h/cpp
- 缺陷描述：ut_main.cpp的SetUp调用PlatformInfoManager初始化，纯CPU UT无硬件则阻塞卡死
- 修复模式：移除真实平台初始化，替换为stub实现
- 可审查性：中
- 审查规则建议：UT框架不应依赖真实硬件平台初始化

### d79d3c4b fix aicpu include path bug
- 根因类别：跨仓路径引用错误
- 涉及文件：cmake/variables.cmake
- 缺陷描述：AICPU_INCLUDE列表中使用${OPS_CV_DIR}(CV仓路径)，在NN仓中应使用${OPS_NN_DIR}
- 修复模式：${OPS_CV_DIR} -> ${OPS_NN_DIR}
- 可审查性：高
- 审查规则建议：cmake include路径不应引用其他仓库的目录变量

### d60227e1 bugfix
- 根因类别：tiling路径条件不完整
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp
- 缺陷描述：GetMixNd2nzType判断V_PARALELL_ND2NZ路径时缺少tilingEnableSplitCore和tilingEnableFixOpti的约束，非BASE状态走该路径计算错误
- 修复模式：条件追加 && tilingEnableSplitCore == BASE && tilingEnableFixOpti == BASE
- 可审查性：高
- 审查规则建议：tiling路径选择条件需覆盖所有相关配置参数组合

### ce3b325f 修复删除算子，增量编译不通过的情况
- 根因类别：增量编译脚本未处理已删除文件
- 涉及文件：scripts/util/parse_changed_files.py, parse_compile_changed_files.py
- 缺陷描述：changed_files中的文件路径可能已不存在(算子被删除)，后续os.path.relpath因文件不存在报错
- 修复模式：处理前添加os.path.exists()跳过已删除文件
- 可审查性：高
- 审查规则建议：处理git diff文件列表时必须考虑文件已删除

### a1c019e8 修复核数少于6个时的问题
- 根因类别：硬编码核数参数
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp
- 缺陷描述：L2切分参数maxConflictDim=6, minConflictDim=3硬编码，低端设备核数<6时分配到不存在的核
- 修复模式：std::min(compileInfo_.aicNum, 6UL) / std::min(compileInfo_.aicNum, 3UL)
- 可审查性：高
- 审查规则建议：tiling中核数相关参数必须以实际aicNum为上界；检测核数字面量

### 6e0e4681 fix triger ut
- 根因类别：UT触发逻辑缺陷
- 涉及文件：build.sh, scripts/util/dependency_parser.py
- 缺陷描述：增量UT触发时对TRIGER_UTS中每个entry无条件执行cmake构建，但有些target当前配置中不存在导致失败
- 修复模式：触发前检查target是否在当前构建目标中
- 可审查性：高
- 审查规则建议：CI中cmake --build --target前验证target存在性

### 34428023 fix rnn 裸指令
- 根因类别：CCE裸指令误用
- 涉及文件：rnn/dynamic_rnn/op_kernel/dynamic_rnn_910b.cpp, rnn/dynamic_rnnv2/op_kernel/dynamic_rnn_v2_910b.cpp
- 缺陷描述：kernel入口调用set_mask_norm()裸指令，可能与框架mask管理冲突
- 修复模式：删除裸指令调用
- 可审查性：高
- 审查规则建议：检测kernel入口中set_mask_norm/set_mask_count等裸指令

### 1f9512ca fix bias len 0
- 根因类别：边界条件遗漏
- 涉及文件：matmul/common/op_host/matmul_common_infershape.cpp
- 缺陷描述：InferRangeBias中num_dim_bias为0(bias不存在)时仍初始化vector并校验，可能越界或推导错误
- 修复模式：num_dim_bias==0时提前返回
- 可审查性：高
- 审查规则建议：GetDimNum()返回值为0需边界检查；可选输入shape推导需覆盖"空输入"

### eb5c3bd2 dynamic_quant_fix
- 根因类别：constexpr与device编译不兼容
- 涉及文件：quant/dynamic_quant/op_kernel/v35/dynamic_quant_struct.h
- 缺陷描述：constexpr uint32_t常量在device端编译环境中可能因ODR问题或不支持导致链接冲突，改用#define
- 修复模式：constexpr -> #define
- 可审查性：中
- 审查规则建议：AscendC kernel头文件中用于模板参数的常量优先使用#define

### a28cf7c7 fix nz copy bug
- 根因类别：NZ格式K维度对齐计算错误
- 涉及文件：matmul/mat_mul_v3/op_kernel/mat_mul_sc_splitk_kernel_gm_to_l1.h
- 缺陷描述：splitk kernel中B矩阵为ND格式时不需按c0Size对齐K，但使用了对齐后的realK导致拷贝越界。NZ格式才需对齐
- 修复模式：引入realKb变量，ND格式用实际K，NZ格式用对齐值
- 可审查性：高
- 审查规则建议：同一算子支持ND/NZ格式时，所有维度对齐操作需对格式做区分

### 5b4f67ce fix bug
- 根因类别：隐式类型窄化
- 涉及文件：index/non_zero/op_host/op_api/nonzero.cpp
- 缺陷描述：size_t加1后传入Shape构造函数(期望int64_t)，隐式转换可能窄化
- 修复模式：添加static_cast<int64_t>()
- 可审查性：高
- 审查规则建议：size_t到有符号整型的隐式转换应显式cast

### 45320686 fix bug
- 根因类别：API层级依赖错误
- 涉及文件：index/embedding_dense_grad_v2/op_host/op_api/aclnn_embedding_dense_backward.cpp
- 缺陷描述：使用上层ACL接口aclrtCtxGetSysParamOpt(acl/acl.h)，应使用底层runtime接口rtCtxGetSysParamOpt(runtime/context.h)，解除不必要的上层依赖
- 修复模式：API调用和头文件替换
- 可审查性：高
- 审查规则建议：算子host代码不应直接依赖acl/acl.h，应使用runtime层接口

### f955a59b 空指针检验
- 根因类别：空指针解引用
- 涉及文件：index/scatter_elements_v2/op_host/scatter_elements_v2_tiling.cpp
- 缺陷描述：GetAttrPointer<int>(0)返回值直接解引用*(attrs->GetAttrPointer<int>(0))，未检查nullptr
- 修复模式：改为指针接收+OP_CHECK_NULL_WITH_CONTEXT检查后再解引用
- 可审查性：高
- 审查规则建议：GetAttrPointer返回值必须先nullptr检查再解引用

### f86b5d37 fix_TBMM_910b_91093_重复部分
- 根因类别：配置冗余
- 涉及文件：transpose_batch_mat_mul_binary.json(ascend910_93和ascend910b)
- 缺陷描述：binary config中存在重复配置条目
- 修复模式：删除JSON中重复的simplified_key条目
- 可审查性：中
- 审查规则建议：binary config JSON不应存在相同simplified_key的重复条目

### f65088be fix foreach compilation failure
- 根因类别：目录结构不符合构建规范
- 涉及文件：976个文件(foreach系列算子graph_plugin→op_graph重命名)
- 缺陷描述：foreach系列算子目录结构使用旧名graph_plugin而非新规范op_graph，导致编译失败
- 修复模式：批量目录重组+CMakeLists路径修正
- 可审查性：低
- 审查规则建议：新增算子目录结构必须遵循op_graph命名约定

### 19456c2f 解决伪量化S4数据类型不匹配INT32问题
- 根因类别：宏条件判断顺序错误
- 涉及文件：matmul/weight_quant_batch_matmul_v2/op_kernel/weight_quant_batch_matmul_v2_apt.cpp
- 缺陷描述：框架传入INT32表示packed INT4，但先检查DT_INT4定义S4宏再处理INT32重定义，此时ORIG_DTYPE_WEIGHT仍是DT_INT32不等于DT_INT4，S4宏永远不定义，S4量化路径完全失效
- 修复模式：调整宏条件顺序，先INT32→INT4重定义再判断S4
- 可审查性：高
- 审查规则建议：数据类型映射的宏定义链需检查定义顺序确保下游宏正确匹配

### b09f2321 fix api_stub
- 根因类别：CMake链接依赖错误
- 涉及文件：cmake/symbol.cmake, common/stub/op_api/CMakeLists.txt, tests/ut/op_api/CMakeLists.txt
- 缺陷描述：target_link_directories引入对已安装CANN包的隐式依赖，改为add_dependencies确保先编译本地target
- 修复模式：link_directories -> add_dependencies + IMPORTED target
- 可审查性：中
- 审查规则建议：避免target_link_directories，优先target-based依赖

### a9384107 fix gather_elements bug
- 根因类别：维度校验缺失 + 循环条件顺序错误
- 涉及文件：index/gather_elements_v2/op_host/gather_elements_v2_last_dim_tiling.h, op_api/aclnn_gather.cpp
- 缺陷描述：(1)MergeAxis中xDimNum和indexDimNum不等未校验导致越界；(2)IfUseGatherElementsV2中dtypeAndSocCheckFlag设置位于break之后，break时未被正确设置
- 修复模式：增加维度相等校验；调整条件判断到break前
- 可审查性：高
- 审查规则建议：多tensor维度操作前校验维度数一致；循环break前后的状态设置需审查是否被跳过

### a49477aa readme&code fix
- 根因类别：返回值类型语义错误
- 涉及文件：matmul/mat_mul_v3/op_host/mat_mul_v3_infershape.cpp
- 缺陷描述：函数返回ge::graphStatus，`return false`隐式转为0即GRAPH_SUCCESS，语义完全相反——本应报错却返回成功
- 修复模式：return false -> return ge::GRAPH_FAILED
- 可审查性：高
- 审查规则建议：返回ge::graphStatus的函数禁止return false/true，必须用枚举值

### 9ab56a34 ut fix
- 根因类别：UT期望值错误
- 涉及文件：conv/conv3d_backprop_input_v2/tests/ut/op_host/test_conv3d_backprop_input_v2_infershape.cpp
- 缺陷描述：动态shape场景infershape结果应为[-1,-1,-1,-1,-1]，UT错误地期望[32,16,8,8,24](静态shape值)
- 修复模式：修正UT期望值
- 可审查性：高
- 审查规则建议：动态shape场景UT应验证输出shape包含-1维度

### 7b4afa89 mmv3 wqbmmv2 code fix
- 根因类别：整数溢出
- 涉及文件：matmul/mat_mul_v3/op_host/mat_mul_v3_infershape.cpp, matmul/weight_quant_batch_matmul_v2/op_host/op_api/aclnn_weight_quant_batch_matmul_v2.cpp
- 缺陷描述：(1)infershape中*input_size+BLOCK_SIZE-1可能溢出int64_t；(2)wqbmmv2中mX*kX可能溢出int64_t后与MAX_MK_VALUE比较不可靠
- 修复模式：前置溢出检查input_size>max-BLOCK_SIZE+1；新增CheckMultiplyOverflow函数
- 可审查性：高
- 审查规则建议：shape维度算术运算(加法/乘法)应添加溢出检查

### 66d2beb0 修复avgpool3d 310p迁移问题
- 根因类别：算子注册遗漏
- 涉及文件：pooling/avg_pool3_d/op_host/avg_pool3_d_tiling.cpp
- 缺陷描述：AvgPool3DV2算子缺少IMPL_OP_OPTILING注册，310P平台无法正确执行tiling
- 修复模式：补充IMPL_OP_OPTILING(AvgPool3DV2)注册
- 可审查性：高
- 审查规则建议：新增V2/V3版本算子时检查是否遗漏IMPL_OP_OPTILING注册

### fe7c0913 conv use base func fix
- 根因类别：API迁移/依赖清理
- 涉及文件：conv/common/op_host/conv_forward_infershape.cpp, 删除aclnn_mm_white_list.h和op_util.h
- 缺陷描述：conv infershape使用ops::Shape2String(被删除的op_util.h)，需替换为基础库Ops::Base::ToString
- 修复模式：API调用替换+冗余文件清理
- 可审查性：中
- 审查规则建议：op_host层代码不应依赖op_api层工具函数

### 8a4c75d1 fix advance_step atk
- 根因类别：tiling参数未设置
- 涉及文件：optim/advance_step/op_host/advance_step_tiling.cpp
- 缺陷描述：tilingKey==1分支中needCoreNum未设置，kernel侧使用未初始化值
- 修复模式：补充set_needCoreNum(this->aivNum_)
- 可审查性：高
- 审查规则建议：所有tiling分支需检查必要参数是否都已设置

### e6415492 kv_rms_norm_rope_cache bug fix
- 根因类别：平台适配文件缺失
- 涉及文件：norm/kv_rms_norm_rope_cache/op_kernel/arch35/platform.h(新增)
- 缺陷描述：缺少arch35平台适配头文件，导致编译失败或使用错误的平台参数
- 修复模式：新增arch35/platform.h
- 可审查性：中
- 审查规则建议：新增算子时检查是否覆盖所有目标平台的platform适配文件

### 73f7ece2 celossgrad fix
- 根因类别：命名空间迁移适配
- 涉及文件：loss/cross_entropy_loss_grad/op_kernel/arch35/cross_entropy_loss_grad_weight_not_none.h
- 缺陷描述：Ops::Base::CeilAlign/CeilDiv调用需改为ops::CeilAlign/CeilDiv
- 修复模式：命名空间前缀更新
- 可审查性：低
- 审查规则建议：基础库API命名空间变更时全量扫描调用方

### 0cb33856 fix rmsnormgrad oom
- 根因类别：tail block DMA拷贝长度错误导致OOM
- 涉及文件：norm/rms_norm_grad/op_kernel/arch35/rms_norm_grad_regbase_dgamma.h
- 缺陷描述：tail block处理时CopyInputsToUB使用rowsPerUB_(最大行数)而非实际尾部行数rows_%rowsPerUB_，DMA多拷数据导致UB溢出
- 修复模式：tailRowsNum计算提前到DMA拷贝阶段
- 可审查性：高
- 审查规则建议：主循环+tail模式中，tail block的DMA长度必须使用tail尺寸而非主循环尺寸

### b98f37a9 PR98遗留问题解决
- 根因类别：逻辑运算符优先级缺陷
- 涉及文件：matmul/batch_mat_mul_v3/op_kernel/batch_mat_mul_v3_block.h等5个文件
- 缺陷描述：`nd2nzFlag_ == ONLY_A || BOTH_AB`缺少第二个==，|| BOTH_AB永远为true(非零值)
- 修复模式：`nd2nzFlag_ == ONLY_A || nd2nzFlag_ == BOTH_AB`
- 可审查性：高
- 审查规则建议：检测`var == VALUE1 || VALUE2`模式，几乎总是逻辑错误

### 791575cd 修复rmsNormQuant的UB越界问题
- 根因类别：tiling切片常量过大
- 涉及文件：norm/rms_norm_quant/op_host/rms_norm_quant_common_tiling.cpp
- 缺陷描述：无bias场景SLICE_SIZE=12288在某些shape下超出UB容量限制，统一为8192
- 修复模式：缩小SLICE_SIZE使其不超过UB容量上限
- 可审查性：高
- 审查规则建议：SLICE_SIZE常量应有注释说明与UB容量的关系，需静态断言slice_size*element_size*buffer_count<=UB_SIZE

### 57a8c64f 修复aclnnGroupNormBackward接口fuzz泛化个别用例aicore异常507015
- 根因类别：workspace大小计算int32溢出
- 涉及文件：norm/group_norm_grad/op_host/group_norm_grad_tiling.cpp
- 缺陷描述：workSpaceSize是int32，乘法在int32域内执行，n和c较大时溢出。需先static_cast<int64_t>再乘
- 修复模式：乘法操作数提升为int64_t
- 可审查性：高
- 审查规则建议：size/workspace/buffer计算中int32*int32必须至少一个提升为int64_t

### 0c1846c9 bugfix stride和dilation pad d维度拦截
- 根因类别：参数校验上限常量混用
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch32/conv_backprop_filter_context_utils.cpp
- 缺陷描述：Conv3D反向的stride_d/dilation_d/pad_f/pad_b使用H/W维度上限(STRIDE_UPPER等)校验，但D维度合法范围更大应用SHAPE_UPPER
- 修复模式：D维度校验改用通用上限常量
- 可审查性：高
- 审查规则建议：3D卷积D/H/W三维度参数校验应使用各自对应的上限常量

### 06d13589 opkernel ut json fix
- 根因类别：header-only库链接方式错误
- 涉及文件：tests/ut/op_kernel/CMakeLists.txt
- 缺陷描述：json是header-only库不需链接，从target_link_libraries改为add_dependencies
- 修复模式：header-only库从link依赖改为构建顺序依赖
- 可审查性：中
- 审查规则建议：header-only库不应出现在target_link_libraries中

### af90af68 bugfix
- 根因类别：printf格式符类型不匹配
- 涉及文件：conv/conv3d_backprop_filter_v2/op_host/op_tiling/arch32/conv_backprop_filter_context_utils.cpp
- 缺陷描述：OP_LOGE中%s对应std::string对象而非const char*，导致未定义行为
- 修复模式：添加.c_str()
- 可审查性：高
- 审查规则建议：printf风格宏的%s参数检查是否传入std::string而非const char*（-Wformat检测）

### 23cdff84 fix bianyi
- 根因类别：结构体成员类型不匹配
- 涉及文件：conv/convolution_backward/op_host/op_api/convolutionbackward.h
- 缺陷描述：groups字段int64_t与下层API期望int不匹配
- 修复模式：int64_t -> int
- 可审查性：中
- 审查规则建议：结构体参数类型应与传递目标API参数类型一致

### cbf1d6eb fix foreach_non_finite include head file failed
- 根因类别：目录结构不符合构建规范
- 涉及文件：foreach/foreach_non_finite_check_and_unscale下9个文件
- 缺陷描述：使用旧名graph_plugin而非新规范op_graph，并缺少ascend910_95平台配置
- 修复模式：目录重命名+补充平台配置
- 可审查性：中
- 审查规则建议：目录遵循op_graph命名约定

### a89df355 fix:单核切K
- 根因类别：splitK场景bias残留
- 涉及文件：matmul/mat_mul_v3/op_kernel/arch35/mat_mul_asw_block.h, mat_mul_asw_kernel.h
- 缺陷描述：splitK多轮K循环中只在最后一轮SetBias，其他轮未ClearBias，残留bias数据影响累加结果导致精度失败
- 修复模式：新增UpdateBias方法：最后一轮SetBias，其他轮ClearBias
- 可审查性：高
- 审查规则建议：splitK/多轮迭代中检查bias是否每轮正确设置或清除；SetBias有条件调用则必须有ClearBias分支

### 16371af6 fix_same_funtion
- 根因类别：重复函数定义冲突
- 涉及文件：conv/common/op_host/conv_forward_infershape.cpp, cube_util.h
- 缺陷描述：cube_util.h中IsUnknownShape/IsUnknownRank与ops命名空间同名函数冲突
- 修复模式：删除重复定义，使用公共库版本
- 可审查性：中
- 审查规则建议：自定义utility头文件检查是否有与公共库同名函数

### b8bfebd5 fix single_op script error
- 根因类别：Shell脚本语法错误
- 涉及文件：scripts/kernel/binary_script/build_binary_single_op_gen_task.sh
- 缺陷描述：(1)[! -z ...]缺空格，正确为[ ! -z ... ]；(2)命令存入变量后$(${cmd})执行不可靠
- 修复模式：修复方括号空格；直接内联$(find ...)
- 可审查性：高
- 审查规则建议：推荐shellcheck静态检查；[ ]内侧必须有空格

### 68a44620 修复kernel找不到act头文件的问题
- 根因类别：编译include路径缺失
- 涉及文件：scripts/util/ascendc_impl_build.py
- 缺陷描述：AscendC kernel编译缺少act头文件目录的-I路径
- 修复模式：补充-I指向tikcpp/../ascendc/act
- 可审查性：中
- 审查规则建议：新增kernel依赖头文件目录时同步更新构建脚本include路径

### 20877724 修复非量化mmv3 fp32场景k很大精度失败
- 根因类别：tiling策略选择不当
- 涉及文件：matmul/mat_mul_v3/op_host/op_tiling/arch35/matmul_v3_asw_basic_tiling.cpp
- 缺陷描述：fp32+大K(>8192)+ND格式时basic api tiling无法正确处理导致精度问题。需在IsCapable中排除该场景回退到更合适的策略
- 修复模式：新增FP32_SPLIT_K_THRESHOLD=8192UL，IsCapable返回false让该场景回退
- 可审查性：高
- 审查规则建议：tiling IsCapable应覆盖所有已知的精度风险shape组合

### bef75e8a DTS2025082812507 问题单修复
- 根因类别：编译告警批量修复
- 涉及文件：11个文件(op_legacy_api, matmul_util, aclnn_matmul等)
- 缺陷描述：未使用参数/变量告警、有符号/无符号比较告警
- 修复模式：移除unused参数/变量，static_cast消除sign compare
- 可审查性：中
- 审查规则建议：-Wunused-parameter -Wunused-variable -Wsign-compare

### 5113cf57 fix_kernel_build_fail_but_return_build_success
- 根因类别：构建失败被静默忽略
- 涉及文件：scripts/kernel/binary_script/gen_output_json.py
- 缺陷描述：kernel编译未生成输出json时仅WARNING+continue，构建返回成功。失败被吞掉
- 修复模式：WARNING改ERROR，continue改raise FileNotFoundError()
- 可审查性：高
- 审查规则建议：构建脚本检测产物缺失时不应continue/pass跳过，必须返回非零或抛异常

### 156fbaeb fix new ge so
- 根因类别：上游依赖拆分后链接缺失
- 涉及文件：cmake/func.cmake, cmake/modules/Findmetadef.cmake, 多个CMakeLists.txt
- 缺陷描述：新版GE SO拆分后需额外查找并链接error_manager/rt2_registry_static/metadef库
- 修复模式：补充CMake中新版GE依赖的库查找和链接
- 可审查性：中
- 审查规则建议：上游依赖拆分/重组时全量检查下游CMakeLists链接依赖

### 366657c4 修复子包打包控制变量未生效问题
- 根因类别：Shell变量初始化逻辑错误
- 涉及文件：CMakeLists.txt, build.sh, cmake/package.cmake
- 缺陷描述：build.sh用$OPTIND判断是否有参数($OPTIND是getopts内部变量初始值为1不代表无参数)，任何场景都触发子包打包
- 修复模式：$OPTIND改为$#判断参数个数；分离构建控制路径
- 可审查性：高
- 审查规则建议：shell中用$#而非$OPTIND判断参数存在性

### f9c92cc8 修复子包和自定义算子包cmake target重名问题
- 根因类别：CMake target命名冲突
- 涉及文件：CMakeLists.txt
- 缺陷描述：子包打包和自定义算子包使用相同target名称导致冲突
- 修复模式：给子包target添加pkg_前缀
- 可审查性：高
- 审查规则建议：CMake中不同功能模块target应有命名前缀约定防止重名

跳过：6e514413(clean code-static→inline), 1c32f798(代码重构), 63b25963(新特性迁移), 4f9d6758(Revert留阶段3), 9fbd3595(Revert留阶段3), 0861ed29(Revert留阶段3), 80df2809(README), 5b4f67ce→已分析, 1b9d36af(文档修复), 79623db1(Revert留阶段3), 43baa6f8(Revert留阶段3), 1536ebfc(文档/注释), a93b20d7(新特性), cf43fc65(目录重组), 0ded3d43(README), 0c283c09(README), ca466912(代码迁移), c2b0c9ff(示例), a7973807(新功能), 3c583402(拼写), 785e2572(Revert留阶段3), a5ca9cf8(OAT license), 9de02749(Revert留阶段3), 8558f785(README)
