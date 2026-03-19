# ops-nn-dev 缺陷模式归纳与分类

## 概览

- 仓库：~/repo/cann/ops-nn-dev/
- 非merge提交：2571条
- 确认缺陷提交：612条（23.8%）
- 实际分析条目：456条（其余跳过：纯文档/配置搬迁/批量重构等）
- 可审查性：453/456为"高"（99.3%）
- Revert事件：20个提交，17个独立事件
- 缺陷热点：3844个唯一文件被触及，749个被多次触及

## 缺陷类别总览

从456条缺陷分析中自然涌现18个类别，按频次降序：

| 编号 | 类别 | 频次 | 占比 | 严重度 |
|------|------|------|------|--------|
| 1 | BUILD_CMAKE 构建系统/CMake | 65 | 14.3% | 中 |
| 2 | BOUNDARY 边界条件缺失 | 47 | 10.3% | 高 |
| 3 | TILING_BUFFER Tiling/Buffer/Workspace计算 | 44 | 9.6% | 高 |
| 4 | INTERFACE_API 接口/API变更传播 | 44 | 9.6% | 中 |
| 5 | COMPILE_WARN 编译告警/代码规范 | 34 | 7.5% | 低-中 |
| 6 | PLATFORM 平台/SoC适配 | 29 | 6.4% | 高 |
| 7 | CALC_LOGIC 计算逻辑错误 | 35 | 7.7% | 高 |
| 8 | CONDITION_LOGIC 条件判断/控制流 | 31 | 6.8% | 高 |
| 9 | COPY_PASTE 复制粘贴错误 | 24 | 5.3% | 中 |
| 10 | INT_OVERFLOW 整数溢出/类型截断 | 18 | 3.9% | 高 |
| 11 | CONFIG_MISSING 配置遗漏 | 22 | 4.8% | 中 |
| 12 | PIPELINE_SYNC 流水线同步/硬件事件 | 16 | 3.5% | 高 |
| 13 | LOG_ERROR 日志/错误处理 | 11 | 2.4% | 低 |
| 14 | DATA_TYPE 数据类型 | 10 | 2.2% | 中 |
| 15 | RESOURCE 资源管理/状态初始化 | 8 | 1.8% | 高 |
| 16 | UT_TEST UT测试 | 8 | 1.8% | 低 |
| 17 | NULLPTR 空指针解引用 | 6 | 1.3% | 高 |
| 18 | REVERT 性能/功能回退 | 4 | 0.9% | 高 |

按严重度×频次的审查优先级排序：
- P0（高频高危）：BOUNDARY(47), TILING_BUFFER(44), CALC_LOGIC(35), CONDITION_LOGIC(31), PLATFORM(29)
- P1（中频高危）：INT_OVERFLOW(18), PIPELINE_SYNC(16), RESOURCE(8), NULLPTR(6), REVERT(4)
- P2（高频中危）：BUILD_CMAKE(65), INTERFACE_API(44), COPY_PASTE(24), CONFIG_MISSING(22), DATA_TYPE(10)
- P3（低危/可自动化）：COMPILE_WARN(34), LOG_ERROR(11), UT_TEST(8)

---

## 类别1: BUILD_CMAKE 构建系统/CMake (65条, 14.3%)

最高频类别，但多为机械性错误，可通过CI自动化大幅减少。

### 子类型分布
- CMake依赖/链接声明缺失: ~18
- include路径错误: ~12
- 打包/安装配置遗漏: ~10
- 编译选项(compile_options)缺失: ~8
- 构建脚本逻辑错误: ~8
- 文件命名/目录结构错误: ~5
- 其他: ~4

### 典型案例

1. c00e940bfa - 新增HardSwishGradV2算子未在ascendc_config.json中注册，导致kernel二进制不被打包
2. fb3b8365b9 - op_api UT编译缺少OPAPI_INCLUDE路径导致头文件找不到
3. 0d47b59fdd - ACLNNTYPE设为aclnn但该算子不应生成aclnn接口
4. 5f3a97918e - CMake变量名混淆导致编译配置错误
5. 9add7e2316 - 构建失败被静默忽略(set -e缺失)

### 审查检查点
- [ ] 新增算子是否已更新ascendc_config.json/binary.json打包配置
- [ ] CMakeLists.txt的DEPENDENCIES列表是否完整
- [ ] include路径层级是否与实际目录结构匹配
- [ ] compile_options是否从同族算子继承并按平台检查
- [ ] 构建脚本是否有set -e / set -o pipefail，文件操作前是否检查存在性

---

## 类别2: BOUNDARY 边界条件缺失 (47条, 10.3%)

第二高频，且严重度高。算子对非典型输入(空tensor、小shape、大shape、退化维度)的处理普遍不充分。

### 子类型分布
- 空tensor/0维tensor处理缺失: ~12
- 尾块/尾核处理不完整: ~8
- 大shape越界/超限: ~6
- 退化维度(dim=1)场景: ~5
- 输入校验遗漏: ~5
- 小shape边界: ~4
- format/维度变体未覆盖: ~4
- off-by-one: ~3

### 典型案例

1. 08b9ebb112 - 2D场景tensor扩展为5D(D=1)，CheckOutputAllZero要求所有维度为0才推断输出shape，D=1导致返回false
2. e2fc1fecb1 - group卷积>=判断将合法tail core和超范围core合并处理，超范围core使用错误数据量
3. 7aa778e0c8 - group convolution最后group的cin不整除group，原代码统一使用cinG导致边界判断错误
4. 95c2fbb6bf - 空Tensor输入/输出场景混淆，输出为空时应设置空shape但未处理
5. e32ce0cbbb - 超大shape未拦截，tiling参数超过硬件寄存器表示范围

### 审查检查点
- [ ] 空tensor输入时算子是否正确early return并设置空输出shape
- [ ] 多核任务分配是否区分"合法tail core"和"超范围idle core"
- [ ] 维度扩展(2D→5D)后，所有依赖维度值的判断是否考虑了扩展维度默认值
- [ ] group卷积/分组操作的最后一组是否单独处理不整除余数
- [ ] tiling参数值是否有硬件位宽上限校验

---

## 类别3: TILING_BUFFER Tiling/Buffer/Workspace计算 (44条, 9.6%)

核心运行时缺陷，直接导致OOM、精度错误、性能劣化。

### 子类型分布
- workspace大小计算公式错误: ~10
- tiling参数遗漏/计算不一致: ~8
- UB空间分配不足/公式错误: ~7
- buffer数量/double buffer策略不一致: ~5
- tiling key/模式选择错误: ~5
- 对齐粒度错误: ~4
- L1内存预留不足: ~3
- tiling结构体host-kernel不一致: ~2

### 典型案例

1. e25e6e60c7 - tiling侧用ubSize_/DOUBLE_BUFFER/bufferCount估算，但kernel侧实际对x和y各占2份buffer(double buffer)，数量不一致导致越界
2. e32a5eaae3 - workspace计算对BT*H和V*H错误乘了BUFFER_NUM(双buffer系数)，这两个buffer实际只需单份空间
3. e1c10bf21e - UB空间计算缺少乘数且无对齐处理，实际数据超出分配的UB空间
4. b99dcb02c5 - tiling参数未区分normal/tail核，tail核分配的workspace不足导致OOM
5. 0d32207040 - TilingData结构体value字段冗余，scale字段类型应为float而非int64_t导致精度丢失

### 审查检查点
- [ ] tiling侧buffer计算是否与kernel侧InitBuffer做交叉比对(buffer数量、double buffer策略)
- [ ] workspace计算公式中每项乘BUFFER_NUM时是否逐项确认需要双buffer
- [ ] UB空间计算是否包含对齐(CeilAlign)且乘以正确的typeBytes
- [ ] tiling参数是否区分normal核和tail核的不同需求
- [ ] TilingData结构体字段布局、类型在host和kernel端是否完全一致

---

## 类别4: INTERFACE_API 接口/API变更传播 (44条, 9.6%)

接口变更后下游未同步更新，或API迁移不完整。

### 子类型分布
- API迁移/搬迁不完整: ~12
- 接口签名变更未同步: ~8
- 废弃接口/代码未清理: ~6
- 命名空间迁移: ~5
- 参数传递方式错误: ~4
- API误用(语义理解错误): ~5
- 其他兼容性问题: ~4

### 典型案例

1. b14f5a03ca - GENERAL_OP_IMPL宏调用op.Init()缺少workspace参数；模板类只传1个类型参数但实际需要第二个kernel mode参数
2. 368607bd - TilingParse注册旧类型CTCLossV2GradCompileInfo，实际tiling依赖新类型CTCLossV2GradForCompileInfo
3. 22979c4afe - API迁移不完整，graph_infer→infershape合并后注册宏未更新
4. 1524c8f0e5 - ISA接口变更，旧版Set_Vector_Mask接口被废弃但仍在使用
5. 1d3f8e2a - GetOptionalInputDesc/HasInput等可选输入检测API误用

### 审查检查点
- [ ] 修改kernel类Init签名或模板参数后，全局搜索所有调用点确认同步更新
- [ ] TilingParse<T>的模板参数T是否与TilingPrepare操作的CompileInfo类型一致
- [ ] 算子废弃或重构时是否清理所有关联的def/registration文件
- [ ] API迁移PR是否有完整的全局搜索确认所有旧调用已替换
- [ ] 可选输入使用HasInput()而非GetOptionalInputDesc()!=nullptr判断

---

## 类别5: COMPILE_WARN 编译告警/代码规范 (34条, 7.5%)

多为可通过编译器flag自动拦截的问题，但部分告警修复中隐含真实逻辑bug。

### 子类型分布
- 未使用参数/变量: ~10
- printf格式符不匹配: ~5
- 参数名遮蔽成员变量(-Wshadow): ~4
- 有符号/无符号比较: ~3
- 隐式类型转换: ~3
- 运算符优先级(-Wparentheses): ~2
- 告警修复引入逻辑错误: ~4
- 其他: ~3

### 典型案例

1. d818b4d4 - 删除"未使用变量"sourceStorageFormat时，发现原始代码条件判断重复检查maskStorageFormat两次而非source和mask各一次(copy-paste逻辑bug)
2. 7ad3e89e - 链式比较`a < b < c`被编译器解析为`(a < b) < c`即`bool < c`，非预期语义
3. 7f6cfb5a - 编译告警修复中将Cast方向从int32→float改为float→int32，引入精度截断逻辑变更

### 审查检查点
- [ ] 删除"未使用变量"前检查该变量是否原本应该被使用(可能掩盖copy-paste bug)
- [ ] 链式比较`a < b < c`是否为预期语义(C++中不等同于数学含义)
- [ ] 告警修复PR中的类型转换方向是否正确，是否改变了原有语义
- [ ] 建议启用: -Werror -Wshadow -Wunused-parameter -Wconversion -Wformat -Wsign-compare -Wparentheses

---

## 类别6: PLATFORM 平台/SoC适配 (29条, 6.4%)

多平台支持(910/910_93/910_95/310P/5102/3510)下的适配遗漏。

### 子类型分布
- 新平台case/分支遗漏: ~10
- 硬编码平台标识/常量: ~6
- __CCE_AICORE__精确匹配应为范围比较: ~4
- 条件编译宏同步缺失: ~3
- 硬件指令兼容性: ~3
- constexpr与device编译不兼容: ~3

### 典型案例

1. 997a551846 - DAV_3510平台不支持x tensor转置，但校验逻辑未按平台区分
2. 15138bbd86 - compute_units列表缺少ascend310p，该平台二进制不被预编译导致运行时JIT异常
3. 4f87bea9ab - `__CCE_AICORE__ == 220`精确匹配应为`>= 220`范围比较
4. 31cd4c17 - 硬编码平台名"ascend910"导致多平台不兼容

### 审查检查点
- [ ] `__CCE_AICORE__ == xxx`精确匹配是否应改为范围比较`>= xxx`
- [ ] compute_units列表是否覆盖所有目标平台
- [ ] 新增平台时是否有checklist检查所有相关算子的case/分支
- [ ] 硬件参数(UB_SIZE, L1_SIZE)是否从运行时或平台配置获取而非硬编码

---

## 类别7: CALC_LOGIC 计算逻辑错误 (35条, 7.7%)

算子核心计算逻辑中的数学/算法错误。

### 子类型分布
- tiling参数计算公式错误: ~8
- 维度/索引计算方向错误: ~5
- 数学公式实现错误: ~4
- 特殊值(NaN/Inf)处理不当: ~3
- 精度损失: ~3
- 多核初始化竞争: ~3
- 常量运算语义错误(乘vs除): ~2
- 量化逻辑错误: ~3
- 其他: ~4

### 典型案例

1. a70d8b16d5 - L1缓存最小feature map高度计算通过CeilDiv后可能超过实际ho值，未做上界clamp
2. fa03888b0d - (1)空bag时多核对全局内存做零初始化存在竞争 (2)validIndicesFactorNumber跨loop累计导致后续loop有效行数不正确
3. e7048cff4f - splitK模式下Bias在最后一次K迭代加载，应在第一次(kIndex==0)加载
4. 939f5b2d67 - 常量运算语义错误：乘法写成除法
5. 5eb0ba7e - NaN判定基于范围比较而非IEEE754标准，漏判denormalized number场景

### 审查检查点
- [ ] CeilDiv/CeilAlign结果是否需要与实际shape维度做min(上界clamp)
- [ ] splitK场景Bias/残差加载是否在正确迭代轮次(kIndex==0)
- [ ] 多核场景全局内存初始化是否由指定核完成并加同步屏障
- [ ] 跨循环累积计数器语义是全局累积还是每次迭代独立
- [ ] NaN判定是否符合IEEE754标准(x != x)

---

## 类别8: CONDITION_LOGIC 条件判断/控制流 (31条, 6.8%)

条件分支遗漏、控制流路径错误、return缺失。

### 子类型分布
- 分支遗漏/不完整: ~10
- 设置状态后缺少return: ~4
- 条件过于宽松/过于严格: ~4
- 循环终止条件错误: ~3
- 除零保护缺失: ~3
- 逻辑运算符优先级: ~3
- fallback路径逻辑缺失: ~2
- 其他: ~2

### 典型案例

1. 2d424df8e6 - SetUnknownRank后没有return语句，继续执行后续shape推导逻辑，GetDimNum返回异常值
2. 6657ff8159 - IsSliceNonContiguous缺少NZ格式不支持判断和左右矩阵dtype必须相同的约束
3. 616b4699ee - IndexCopy非连续场景无条件走ScatterUpdate路径，大shape时性能反而更差
4. 6beab61dd0 - 连续matmul操作间缺少PipeBarrier同步；宏中使用return语句隐藏控制流
5. 22f84c0e - 除零错误，分母未做非零保护

### 审查检查点
- [ ] 设置特殊状态(UnknownRank/UnknownShape)的分支是否包含return
- [ ] 优化路径的启用条件是否考虑数据规模维度(避免大shape性能劣化)
- [ ] switch/case是否覆盖所有平台架构和dtype，default分支是否返回错误而非静默通过
- [ ] 除法运算的除数是否有非零保护
- [ ] 宏定义中是否包含return语句(隐藏控制流anti-pattern)

---

## 类别9: COPY_PASTE 复制粘贴错误 (24条, 5.3%)

高度可审查的机械性错误，但在大规模代码中极易漏检。

### 子类型分布
- 变量名混淆(A/B, input/output, row/col): ~8
- 日志/错误信息引用错误变量: ~5
- 参数位置重复: ~4
- 示例代码/文档错误: ~3
- 拼写错误(typo): ~4

### 典型案例

1. d1832c87c9 - CheckDimValue检查scale维度但报错时误用offsetShape(offset的shape)
2. 5972749646 - TransDataPreProcess对weight执行TransData后，CHECK_NULLPTR却检查input而非weight
3. dec58d1b24 - aclnnAddbmmGetWorkspaceSize调用isAddBmmProcessEmptyTensor(batch1, batch1)，第二个参数应为batch2
4. 955dd52b65 - 复制粘贴错误，行/列变量名混用(n轴和k轴维度索引条件表达式相同)

### 审查检查点
- [ ] 连续相似CHECK/LOG调用中，检查对象与上文赋值/条件判断对象是否一致
- [ ] 函数调用中多个同类型参数(batch1/batch2, src/dst)是否存在重复
- [ ] 条件判断中引用的维度索引(row/col, m/n/k)是否与上下文匹配
- [ ] 错误日志引用的变量名是否与实际校验的变量一致

---

## 类别10: INT_OVERFLOW 整数溢出/类型截断 (18条, 3.9%)

频次不是最高，但后果严重(OOM、越界访问、精度错误)，且当前代码中仍有12处残留。

### 子类型分布
- shape维度乘积溢出int32: ~8
- tiling参数/buffer偏移计算溢出: ~4
- 硬件指令参数位宽限制: ~2
- 无符号整数下溢: ~2
- 隐式类型提升异常: ~2

### 典型案例

1. fa1305c56e - 排序索引硬编码DT_INT32，大shape下最后一维超过INT32_MAX时索引溢出
2. 952313ace0 - load3d指令kStartPt字段上限65535(16位)，k0*hk*wk超过65536时溢出翻转
3. e1a2f11f - mode0xDyDxSize为uint32_t，CPerG_*HxW_超长时CeilAlign()*tTypeBytes_*UB_COPIES_3乘积溢出
4. e0ddd962 - gmOffset_计算中addUbSize.GetValue(8)*shapeDim_均为int32，output shape乘积超int32边界
5. 6ac2bba4 - workspace大小计算int32溢出

### 审查检查点
- [ ] shape维度相乘的表达式是否使用int64_t(不是int32)
- [ ] 涉及硬件指令参数时确认所有字段位宽限制并做边界校验
- [ ] GetValue()返回值参与乘法且结果赋给uint64/int64时，操作数是否显式cast为64位
- [ ] size_t/uint64_t做减法前是否判断被减数>=减数(防止下溢)
- [ ] CeilAlign/CeilDiv返回值参与二次乘法时结果类型是否足够宽

---

## 类别11: CONFIG_MISSING 配置遗漏 (22条, 4.8%)

算子注册/编译/运行时配置文件的条目遗漏或参数错误。

### 子类型分布
- compile_options遗漏: ~6
- 算子注册/binary配置遗漏: ~5
- 算子属性(paramType/impl_mode)错误: ~4
- 配置文件section名/路径错误: ~3
- 平台特定kernel参数缺失: ~4

### 典型案例

1. ac64e3d38b - 约20个算子在ascend910_95平台缺少-mllvm -cce-aicore-dcci-before-kernel-end=false编译选项
2. 10cab125fa - ScatterAdd等3个算子impl_mode字段为空字符串，缺失high_precision配置
3. 9c46dc5f19 - 910_95平台配置中rstd和x输出被标记为REQUIRED但实际应为OPTIONAL

### 审查检查点
- [ ] ascend910_95平台算子是否配置dcci相关编译选项
- [ ] 新增算子时对比同族算子(XXX与XXXGrad)的配置一致性
- [ ] def.cpp中output的paramType(REQUIRED/OPTIONAL)与binary.json是否双向一致
- [ ] impl_mode字段是否按需配置(如scatter类需要high_precision)

---

## 类别12: PIPELINE_SYNC 流水线同步/硬件事件 (16条, 3.5%)

NPU硬件同步原语使用错误，后果为精度错误或硬件异常。

### 子类型分布
- SetFlag/WaitFlag不配对或条件化: ~5
- 同步事件通道类型错误(MTE2/MTE3): ~3
- PipeBarrier滥用或缺失: ~3
- DMA参数超硬件限制: ~2
- flag编号冲突: ~1
- 多核同步屏障缺失: ~2

### 典型案例

1. a79b527fe8 - 多处WaitFlag被条件化(if ni>0等)，导致某些迭代路径跳过必要的同步等待
2. 51f8247aee - CopyOut前使用S_MTE2同步但CopyOut走MTE3通道，应使用S_MTE3
3. 58f275c6dc - 310P芯片DMA搬运参数为uint16_t，stride值可能超过UINT16_MAX截断
4. 8f600f1a - flag编号冲突，两个不同的同步事件使用了相同的eventID

### 审查检查点
- [ ] SetFlag和WaitFlag是否成对出现且无条件执行
- [ ] CopyOut/CopyIn前的同步事件是否与实际DMA通道匹配(MTE2搬入/MTE3搬出)
- [ ] DMA搬运指令的stride/nBurst等参数是否可能超过硬件限制(uint16_t上限65535)
- [ ] 连续matmul操作间是否插入PipeBarrier
- [ ] 同一kernel中eventID是否有冲突

---

## 类别13: LOG_ERROR 日志/错误处理 (11条, 2.4%)

### 典型案例

1. dc3a8850e7 - 错误日志"pertensor"与实际检查条件"perchannel"矛盾
2. 797589989a - CheckNotNull对用户传入参数使用ACLNN_ERR_INNER_NULLPTR(内部错误)，应为PARAM_NULLPTR
3. 3ac83c4ebf - 构建脚本读取任务文件前未检查文件是否存在

### 审查检查点
- [ ] 错误码区分PARAM_*（用户输入）和INNER_*（内部不可达状态）
- [ ] 错误日志中条件描述是否与实际判断条件一致
- [ ] Shell脚本对文件操作前是否做[ -f ]检查

---

## 类别14: DATA_TYPE 数据类型 (10条, 2.2%)

### 典型案例

1. 30b9129145 - 自定义ReduceOpTilingDataV2与标准ReduceOpTilingData字段布局不一致
2. 84ab237ad3 - SelectTiling未排除int64进入排序模板，该模板不支持int64精度
3. 9150a7cca8 - 新增数据类型支持时tiling层/kernel层/binary配置三处未同步

### 审查检查点
- [ ] 禁止手动镜像标准库数据结构定义，直接引用原始头文件
- [ ] 新增数据类型支持时同步更新tiling层dtype校验、kernel层预编译条件、binary配置三处
- [ ] 选择计算模板/优化路径时检查是否对所有支持的数据类型兼容

---

## 类别15: RESOURCE 资源管理/状态初始化 (8条, 1.8%)

### 典型案例

1. 2a15cd4d - indexstrideList定义为.cpp文件全局std::vector，多实例并发调用共享导致AIC错误
2. 46f5817ef1 - B矩阵偏移变化时未切换全载状态(enableFullLoad)，之前L1数据已失效
3. aa95cab243 - ACL example中aclrtMalloc/aclCreateTensor在CHECK_RET失败early return时不释放

### 审查检查点
- [ ] 禁止在op_host/*.cpp中使用非const全局容器(并发不安全)
- [ ] L1/L0全载优化在多batch/多group循环中，检查全载条件是否因参数变化而失效
- [ ] 状态变量初始化是否放在最早的调用入口

---

## 类别16: UT_TEST 单元测试 (8条, 1.8%)

### 典型案例

1. 91c29129 - 大量EXPECT_EQ断言被批量注释掉(anti-pattern)
2. ee685a90 - UT中InferShape和InferDataType使用同一holder对象导致状态污染
3. 9a18f2ca - tiling参数手工硬编码与kernel逻辑不一致

### 审查检查点
- [ ] 禁止直接注释EXPECT_*/ASSERT_*断言(可grep检测"// EXPECT_"模式)
- [ ] UT中不同阶段的推理应使用独立context
- [ ] UT中的tiling参数应由tiling函数自动生成而非手工硬编码

---

## 类别17: NULLPTR 空指针解引用 (6条, 1.3%)

### 典型案例

1. e49d5d27c0 - 只需计算dw不需dx时gradInput为nullptr，但代码用gradInput的shape计算
2. 59a561e5 - PreConv1DBackwardTo2D无条件对gradInput/gradWeight调用View4dWithGroups，但outputMask可能指示它们不需要计算(为null)
3. 2463697d99 - 用可选输入gamma/beta的dtype做兼容性判断，但它们可能为null

### 审查检查点
- [ ] 可选输出tensor(由outputMask控制)的任何访问前必须检查null或对应mask位
- [ ] dtype兼容性判断优先使用必选张量(output)而非可选张量
- [ ] 反向传播中dx/dw/dbias可能为nullptr，解引用前需null检查

---

## 类别18: REVERT 性能/功能回退 (4条, 0.9%)

### 典型案例

1. 74cdae15d7 - NLLLossGrad大规模性能优化重构(拆分为4d/base/tiling_key等多个文件)被完全回退
2. fe16c557 - 功能补齐提交做了三件事(新增tiling模板+强制禁用已有模板+扩大芯片适配)，整体回退
3. e682154870 - 过早移除MatMulV3的bf16 bias编译配置(binary.json变体)

### 审查检查点
- [ ] 大规模算子重构应分步提交，每步可独立验证
- [ ] 移除binary.json中的算子变体前，必须确认所有下游场景已无调用
- [ ] 强制禁用已有模板(return false)必须注释原因

---

## 跨类别系统性风险（来自hotspot和revert分析）

### 风险1: matmul是绝对缺陷震中

- 热点Top 30文件中matmul占20席
- Revert事件中matmul系列占45%(9/20)
- 量化矩阵乘(quant_batch_matmul_v3/v4, weight_quant_batch_matmul_v2)尤为集中
- 建议: matmul相关PR需要额外一轮专项审查

### 风险2: 整数溢出是最普遍残留风险

- 当前代码中仍存在12处int32溢出风险
- 横跨matmul/conv/quant/norm四个模块
- 根因: int32类型的tiling参数相乘、batch维度累乘、buffer偏移计算均缺少溢出保护
- 建议: 编写lint规则，强制shape维度算术运算使用int64_t

### 风险3: 提交粒度过大导致质量失控

- qbmm事件：6.5万行一次性合入导致3次循环revert
- weight_quant事件：5.7万行合入修改了公共error_util.h导致其他算子编译失败
- 70%的revert在3天内发生，说明合入前review不充分
- 建议: 单次合入超过3000行的PR必须拆分

### 风险4: 公共基础设施耦合在算子PR中

- 5次revert源于公共头文件(op_util.h, error_util.h)和cmake修改耦合在算子PR中
- 公共修改影响面远超单算子，但review时容易被算子逻辑掩盖
- 建议: 公共文件修改必须独立PR先行验证

### 风险5: foreach模块高触及但低风险

- 聚合触及次数最高(1452次)，但本质是批量模板代码同步修改
- 单个算子结构性风险低，真正需重点review的是matmul和quant的kernel/tiling层
- 不应被热点文件列表误导分配不当的审查资源

---

## 与ops-nn主仓缺陷模式对比

ops-nn-dev (612缺陷/2571提交=23.8%) vs ops-nn (380缺陷/1474提交=25.8%)

相同点：
- 整数溢出、tiling/buffer计算、边界条件缺失、平台适配是两者共有的高频高危类别
- matmul模块在两个仓库都是最高危模块
- 可审查性均在99%以上

差异点：
- ops-nn-dev的BUILD_CMAKE占比(14.3%)远高于ops-nn(~5%)，dev分支构建基础设施更不稳定
- ops-nn-dev的INTERFACE_API占比(9.6%)高于ops-nn(~4%)，dev分支接口变更更频繁
- ops-nn-dev有更多UT_TEST类缺陷，dev分支测试基础设施迁移频繁
- ops-nn主仓的编译告警/代码规范类缺陷在分析时已由CI拦截更多，dev分支CI门禁更宽松

---

## 审查规则优先级矩阵

### P0 — 必须检查（高频高危，可直接拦截严重缺陷）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| R01 | shape维度相乘/buffer偏移计算强制使用int64_t | INT_OVERFLOW | fa1305c56e, e1a2f11f, e0ddd962 |
| R02 | tiling侧buffer计算必须与kernel侧InitBuffer做交叉比对 | TILING_BUFFER | e25e6e60c7, b99dcb02c5 |
| R03 | 空tensor输入时是否正确early return并设置空输出shape | BOUNDARY | 95c2fbb6bf, e32ce0cbbb |
| R04 | SetFlag/WaitFlag必须成对出现且无条件执行 | PIPELINE_SYNC | a79b527fe8, 51f8247aee |
| R05 | 设置特殊状态(UnknownRank)后必须包含return | CONDITION_LOGIC | 2d424df8e6 |
| R06 | CeilDiv/CeilAlign结果是否需要与实际shape做min(上界clamp) | CALC_LOGIC | a70d8b16d5, 7b7999e9e0 |
| R07 | 新增平台时checklist检查所有相关算子的case/分支和compute_units | PLATFORM | 15138bbd86, 4f87bea9ab |
| R08 | workspace计算公式中逐项确认是否需要双buffer | TILING_BUFFER | e32a5eaae3 |
| R09 | 可选输出tensor(outputMask控制)的访问前必须检查null | NULLPTR | 59a561e5, e49d5d27c0 |
| R10 | 多核场景全局内存初始化应由指定核完成并加同步屏障 | CALC_LOGIC+SYNC | fa03888b0d |

### P1 — 重点检查（中频高危或高频中危）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| R11 | 修改Init签名/模板参数后全局搜索所有调用点同步更新 | INTERFACE_API | b14f5a03ca |
| R12 | 新增算子checklist: ascendc_config.json + binary.json + CMake DEPENDENCIES | BUILD_CMAKE+CONFIG | c00e940bfa |
| R13 | 连续相似CHECK/LOG调用中检查对象与赋值对象是否匹配 | COPY_PASTE | 5972749646, d1832c87c9 |
| R14 | `__CCE_AICORE__ == xxx`精确匹配是否应改为范围比较 | PLATFORM | 4f87bea9ab |
| R15 | switch/case的default分支不能静默通过，必须返回错误 | CONDITION_LOGIC | 多处 |
| R16 | TilingData结构体字段布局和类型在host和kernel端完全一致 | TILING_BUFFER+DATA_TYPE | 0d32207040 |
| R17 | 新增数据类型支持时同步tiling层/kernel层/binary配置三处 | DATA_TYPE | 9150a7cca8 |
| R18 | 禁止在op_host/*.cpp中使用非const全局容器 | RESOURCE | 2a15cd4d |
| R19 | DMA搬运参数(stride/nBurst)是否可能超过硬件限制(uint16_t上限) | PIPELINE_SYNC | 58f275c6dc |
| R20 | 除法运算的除数必须有非零保护 | CONDITION_LOGIC | 22f84c0e |

### P2 — 常规检查（中危或可部分自动化）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| R21 | 公共文件(op_util.h, error_util.h)修改必须独立PR | BUILD_CMAKE | revert分析 |
| R22 | 单次合入超过3000行的PR必须拆分 | REVERT | qbmm事件 |
| R23 | 删除"未使用变量"前检查是否原本应该被使用(可能掩盖bug) | COMPILE_WARN | d818b4d4 |
| R24 | 同族算子(XXX与XXXGrad)配置一致性对比 | CONFIG_MISSING | 793745bf14 |
| R25 | 错误码区分PARAM_*(用户输入)和INNER_*(内部状态) | LOG_ERROR | 797589989a |
| R26 | 禁止直接注释EXPECT_*/ASSERT_*断言 | UT_TEST | 91c29129 |
| R27 | 示例代码必须通过实际编译运行验证 | COPY_PASTE | e2f856d387 |
| R28 | matmul相关PR需要额外一轮专项审查 | 跨类别 | revert分析 |

### P3 — 建议自动化（可通过工具/CI拦截）

| 规则ID | 规则 | 实现方式 |
|--------|------|---------|
| R29 | 启用编译器flags: -Werror -Wshadow -Wconversion -Wformat -Wsign-compare -Wparentheses | CMake配置 |
| R30 | shape维度算术运算lint规则(检测int32乘法) | 自定义lint |
| R31 | ascendc_config.json与算子目录交叉校验 | CI脚本 |
| R32 | SetFlag/WaitFlag配对检测 | 静态分析 |
| R33 | "// EXPECT_"注释断言检测 | grep规则 |
| R34 | compile_options平台差异检测 | CI脚本 |
