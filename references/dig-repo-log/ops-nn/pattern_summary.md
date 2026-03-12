# ops-nn 缺陷模式归纳与分类

基于阶段2(380条缺陷提交全量分析)、阶段3(5个Revert提交/4个事件)、阶段4(热点文件结构性风险)的综合汇总。

## 统计概览

- 总提交: 1474 (全部非merge)
- 确认缺陷提交: 380 (25.8%)
- 实际代码缺陷: ~321 (排除纯文档/UT/清理约59条)
- Revert事件: 4个独立事件(5条revert提交)
- 热点确认bug: P0 x8, P1 x8, P2 x8

## 缺陷类别总览 (按频次降序)

| # | 类别 | 频次 | 占比 | 可审查性 |
|---|------|------|------|----------|
| 1 | 文档与示例代码 | ~36 | 11.2% | 高 |
| 2 | 构建系统与配置缺陷 | ~35 | 10.9% | 高 |
| 3 | 条件判断与边界处理 | ~29 | 9.0% | 高 |
| 4 | 整数溢出与类型安全 | ~25 | 7.8% | 高 |
| 5 | 空指针与资源管理 | ~22 | 6.9% | 高 |
| 6 | 编译告警与代码规范 | ~21 | 6.5% | 高 |
| 7 | 输入校验与接口一致性 | ~18 | 5.6% | 高 |
| 8 | Shape与Tiling计算 | ~16 | 5.0% | 中 |
| 9 | 复制粘贴与变量引用 | ~16 | 5.0% | 高 |
| 10 | 多核/流水线并发 | ~15 | 4.7% | 低-中 |
| 11 | 平台适配与硬件约束 | ~10 | 3.1% | 中 |
| 12 | 功能回退与搭车提交 | ~8 | 2.5% | 中 |

注: 部分提交涉及多个类别，频次总和略超实际缺陷数。其余~20条为低频杂项(硬编码常量4、UT缺陷3等)。

---

## 类别1: 文档与示例代码 (~36次)

非纯文档修复(纯文档已在阶段1过滤)，指代码仓中的示例代码bug、文档与代码不一致、op_list条目遗漏等。

### 典型案例

- `501d38ab` A16W4示例代码重复函数定义(AclnnWeightQuantBatchMatmulV2Test出现两次)，用户拷贝编译直接报重定义错误
- `4063f8f9` aclnnTransposeBatchMatMul示例输出全0，FP16数据初始化错误，示例从未被实际运行验证
- `964be4dd` deep_norm_grad示例vector<int32_t>但传ACL_FLOAT创建tensor；deep_norm示例缺aclrtFree导致内存泄漏

### 审查检查点

1. 示例代码应在CI中编译验证(至少编译通过)
2. 示例中vector<T>的T必须与aclDataType匹配
3. aclrtMalloc/aclrtFree必须配对出现
4. 新增算子时op_list.md条目名称必须与算子名一致

---

## 类别2: 构建系统与配置缺陷 (~35次)

CMakeLists依赖声明缺失/错误、binary.json平台配置遗漏、build.sh语法错误、ascendc_config.json重复/冲突条目。

### 典型案例

- `640a1683` ascendc_impl_gen的DEPENDS未包含前置target(ops_info_gen_*)，构建时序不确定导致间歇失败
- `d6acf37e` unique_consecutive缺少ascend950 binary.json，该平台完全无法使用
- `adec83fb` build.sh中`[ $status -ne 0]`缺空格，条件判断失效，编译失败时CI仍显示成功

### 热点补充 (阶段4 P0确认)

- build.sh:1278 `]`前缺空格导致条件永远false
- build.sh:1060-1065 数组备份`${BUILD_LIBS}`只取首元素，恢复时变成字符串
- cmake/gen_ops_info.cmake:654 循环后遗留变量作为target名
- cmake/ut.cmake:571/587 CUSTOM_TILING_DATA_KEYS先用后赋值

### 审查检查点

1. CI脚本用shellcheck静态分析，`[`和`]`前后必须有空格
2. CMake custom_command/custom_target的DEPENDS必须包含所有输入依赖
3. 新增算子/新平台时同步添加所有目标平台的binary配置
4. ascendc_config.json中检查同名算子是否有重复/冲突配置条目

---

## 类别3: 条件判断与边界处理 (~29次)

逻辑运算符错误(||/&&混淆)、恒真/恒假表达式、off-by-one、尾块处理遗漏、条件分支覆盖不全。ops-nn中此类缺陷高频出现，尤其是`!=A || !=B`恒真模式。

### 典型案例

- `67c665fd` avg_pool_v2_grad: `inputDimNum != CHW_DIMS || inputDimNum != NCHW_DIMS`用||恒为true，合法输入被拒绝，应改为&&
- `692f43a9` adaptive_pool3d_tiling: `npuArch!=DAV_3510 || npuArch!=DAV_5102`恒真条件，所有平台返回错误。修复：删除第二个条件
- `4d315436` QuantBatchMatmulV4: 新增int4类型后k0默认值条件反转，int4变成了默认路径而非特殊分支

### 审查检查点

1. `x!=A || x!=B`是经典恒真逻辑错误(De Morgan)，应作为静态检查规则(clang-tidy always-true)
2. 新增类型/分支时，else(默认分支)必须保持原有行为不变
3. 三元表达式的true分支应对应"特殊情况"而非"常规情况"
4. 边界条件: 除零防护、尾块处理、动态shape的零维度

---

## 类别4: 整数溢出与类型安全 (~25次)

int32溢出(大shape/大元素数)、有符号/无符号混比、类型转换截断(uint64->uint32)、GM偏移在小类型域计算溢出。

### 典型案例

- `8f6ccaea` addlayernormgrad: uint32_t的roundUpNumLastDimFloat在ROUND_UP后乘sizeof(float)溢出(numLastDim>2^31-1)
- `acadb4c7` repeat_interleave: repeatSum超INT32_MAX时int32计算溢出，未区分是否需int64
- `50df91e7` foreach算子: GlobalTensor下标`index * maxDataCount`中index为uint16_t/uint32_t，乘法在小类型域溢出，10个文件涉及8-9个算子(foreach_copy/lerp/norm/pow/sign等)，修复用`1ULL*`前缀强制64位提升

### 审查检查点

1. shape维度乘法变量必须使用int64_t/uint64_t
2. GM内存偏移计算必须使用int64/uint64，乘法表达式中至少一个操作数为64位
3. tiling中应有大shape的int32溢出保护
4. 开启-Wsign-compare并视为错误，常量类型与比较对象类型一致

---

## 类别5: 空指针与资源管理 (~22次)

工厂函数返回值未检查、先解引用后判空、结构体成员未初始化(野指针)、PipeBarrier与FreeTensor时序颠倒。

### 典型案例

- `7b4a1b53` aclnnNanMedian: OP_CHECK宏第三参数写成裸`nullptr`而非`return nullptr`，失败时不return继续执行用空指针，14处同一问题
- `15e40a48` addmm: PromoteResult.logMessage指针未初始化，某路径为野指针，后续无条件OP_LOGE("%s", result.logMessage)崩溃
- `cf84e222` AddRmsNorm: PipeBarrier<PIPE_V>放在FreeTensor之后，Cast未完成就释放输入buffer

### 热点补充 (阶段4 P0确认)

- aclnn_addmm.cpp:298-299 空指针先用后查: `auto x = Contiguous(self, exec); if (self != nullptr ...)`
- aclnn_quant_matmul_v5.cpp:907-908 CHECK_RET检查错误变量: `reformatedX1 = TransData(...); CHECK_RET(x1 != nullptr)`应检查reformatedX1

### 审查检查点

1. OP_CHECK宏第三参数必须包含return语句，静态分析检测`, nullptr)`模式
2. C++结构体指针成员必须初始化为nullptr，所有成员声明处初始化
3. PipeBarrier必须在FreeTensor之前(释放前确保计算完成)
4. 指针解引用必须在nullptr检查之后(先检查后使用)

---

## 类别6: 编译告警与代码规范 (~21次)

编译warning中隐藏真bug(无符号>=0恒真循环、链式比较==a==b、运算符优先级)、有符号/无符号比较、未使用变量。此类别特殊之处在于"修复warning"的提交中经常同时发现warning背后的真实逻辑bug。

### 典型案例

- `0e00f88c` `for(uint64_t i=singleCoreWo; i>=0UL; i--)`无符号整数>=0恒真导致无限循环 --- 一个warning修复中发现的真bug
- `50854088` `a==b==1`链式比较被解析为`(a==b)==1`; `FORMAT_NZ && dim2==1 || dim1==1`优先级错误(&&高于||)
- `941d3c7f` uint64_t常量与int64_t shape维度比较触发-Wsign-compare

### 审查检查点

1. 无符号整数循环中`>=0`条件是无限循环bug(-Wtautological-compare)
2. `a==b==c`链式比较在C++中语义错误，应写成`a==b && b==c`
3. `&&`和`||`混合使用时必须加括号(-Wlogical-op-parentheses)
4. 建议全局开启`-Wall -Werror -Wsign-compare -Wshadow`

---

## 类别7: 输入校验与接口一致性 (~18次)

dtype默认参数与实际不一致、通过硬编码索引获取属性(不同算子attr顺序不同)、变体接口复用通用校验函数遗漏特有限制、nullptr入参未拦截致core dump。

### 典型案例

- `c8ca6bec` AdaptiveMaxPool3d: 调用MaxPool3D接口时未传indicesDtype，默认int32但用户可能传int64
- `07e77ddd` InplaceAddmm走GemmV3时: GetAttrPointer<bool>(0)/(1)读transpose，但GemmV3的attr顺序不同(transpose在索引2/3)
- `0bd3fe0b` WeightNz接口: 复用通用dtype校验允许了不支持的int8/fp32，nullptr入参未拦截致core dump

### 审查检查点

1. 具有默认参数的接口调用，检查实际数据类型是否可能与默认值不一致
2. 通过索引获取算子属性时，确认不同算子类型的attr排列顺序是否一致；优先用属性名
3. 变体接口(WeightNz等)应使用独立dtype校验而非复用通用函数
4. aclnn函数入口第一步应做所有必选输入的nullptr检查

---

## 类别8: Shape与Tiling计算 (~16次)

StorageShape/ViewShape混淆、tiling key参数错误、Shape校验维度不全、dead code(return后的SetAtomicNone)。算子库特有的高风险类别。

### 典型案例

- `6b52fdcf` LSTM: CheckFormatValid统一用FORMAT_ND但实际为FORMAT_NCL; ValidateInputShape用GetStorageShape()但非连续张量StorageShape != ViewShape
- `28ab6d70` thnn_fused_lstm_cell_backward: StorageShape获取维度，经view操作后与逻辑形状不符
- `6eee5478` tiling key参数传递错误选择错误策略; SetAtomicNone在return之后永远不执行

### 审查检查点

1. shape校验优先使用GetViewShape()，仅在需要物理布局时用GetStorageShape()
2. 非连续张量场景必须区分StorageShape和ViewShape
3. return语句后不应有可执行代码(dead code检测)
4. tiling key参数应有枚举约束而非裸字符串/魔法数

---

## 类别9: 复制粘贴与变量引用 (~16次)

函数调用参数重复(f(a,a)应为f(a,b))、维度索引copy-paste错误(k/n互补关系)、偏移量计算公式套错维度步长、先用后查时序颠倒。

### 典型案例

- `b2e2cada` addbmm: `isAddBmmProcessEmptyTensor(batch1, batch1)`第二参数应为batch2，copy-paste遗漏
- `1c2de786` infershape: n_x2_dim与k_x2_dim公式完全相同(粘贴错误)，矩阵k/n维度索引在转置/非转置时必须互补
- `910dac63` QuantUpdateScatter: gmVarOffset_计算将不同维度索引合并后统一乘同一步长，公式语义错误

### 热点补充 (阶段4 P0确认)

- mat_mul_deterministic_splitk_kernel.h:404-405 rowBlockNum/colBlockNum公式完全相同(疑似copy-paste)

### 审查检查点

1. 函数调用中两个参数相同`f(a,a)`时必须确认是否为copy-paste错误
2. 矩阵k/n维度索引在转置/非转置情况下必须互补
3. 多维偏移量应逐维展开(各维索引 x 对应步长)，避免合并表达式
4. 相邻行的变量名只差一个字符(如x1/x2, row/col)时重点核对

---

## 类别10: 多核/流水线并发 (~15次)

SetScheduleMode缺失、SetFlag/WaitFlag顺序错误、DataCopy前后缺少MTE/V/S同步事件、workspace脏数据(并发竞争)。此类缺陷可审查性最低，多为运行时随机表现。

### 典型案例

- `afb09c78` dynamic_mx_quant: 缺少SetScheduleMode(1)，多核间存在数据依赖但未设batch mode同步
- `ed0bd5a1` QBMMv4: WaitFlag放在SetFlag之前，等待的是上一轮flag而非当前轮
- `74d43ef3` scatter_sub: DataCopy(MTE2)前缺MTE3_MTE2同步，向量运算前缺MTE2_V同步，相同输入产生不同输出

### Revert补充 (阶段3)

- 事件4 (`e248762f7`/`a39e18548`): MatmulV3 small shape优化中workspace脏数据 --- vector和cube并行操作workspace时buffer残留上次计算数据，被cube管道读取导致精度错误

### 审查检查点

1. 所有tiling函数必须显式设置SetScheduleMode
2. SetFlag/WaitFlag必须成对且顺序为Set -> Wait -> Use
3. DataCopy(MTE2)前确保前序MTE3完成; 向量运算(V)前确保MTE2数据就绪
4. vector/cube并行场景中workspace buffer必须确认初始化/清零策略

---

## 类别11: 平台适配与硬件约束 (~10次)

DMA/datacopy指令参数超16位上限(65535)、uint16溢出截断、新芯片端云兼容性、硬件宏条件编译遗漏。

### 典型案例

- `34217db7` Conv3DBackpropInput: cin>65535时datacopy stride参数超16位上限导致精度失败
- `cf9ea8c9` dynamic_quant: DMA参数uint16_t类型，stride超UINT16_MAX时截断溢出
- `c1469b68` kirin适配: 三个算子端云无法兼容需删除kirin配置; Fixpipe指令需增加__NPU_ARCH__条件

### 审查检查点

1. datacopy/DMA stride参数使用前校验不超过65535(16位上限)
2. 硬件指令参数有位宽限制时必须做溢出保护
3. 新芯片适配时应有端云兼容性检查清单
4. 特定芯片指令需用__NPU_ARCH__等硬件宏条件编译保护

---

## 类别12: 功能回退与搭车提交 (~8次)

大feature合入后紧急revert、MR中混入不相关改动(搭车)、revert时造成附带伤害。

### 典型案例

- `d91e1757` unsorted_segment_sum迁移: 38文件/8892行新增，但op_list中错误添加scatter条目(非uss)，CI失败后整体revert
- `852f21d6` batchmatmul非连续输入: 变量遮蔽(auto shadowing) + 空指针检查缺失，影响正常路径，1天内revert
- `a39e1854`/`e248762f7` MatmulV3 small shape: workspace脏数据导致静默精度错误

### Revert跨事件模式 (阶段3)

1. 搭车提交(2/4事件): 不相关改动混入MR，revert时造成附带伤害
2. 变量遮蔽/初始化缺陷(2/4): 数据非预期值
3. 大规模提交测试不充分(3/4): 跨多层改动应拆分为小MR

### 审查检查点

1. MR中每个文件改动应与标题描述的功能相关，不相关改动拆分为独立MR
2. 大feature合入(>20文件或>1000行)前需全流程CI + 集成测试
3. 新增优化路径需验证workspace初始化/清理逻辑覆盖所有场景
4. 功能开关(feature flag)控制新功能上线，降低revert影响范围

---

## 跨类别系统性风险

### 风险1: build.sh系统性脆弱

build.sh被10条缺陷提交触及(热点第1)，当前代码仍存在多个P0/P1 bug:
- `set +u`从行35开始从未恢复，整个脚本未定义变量静默为空
- `]`前缺空格(P0)、数组备份只取首元素(P0)
- ASCEND_HOME_PATH未设置时路径变成`/include`等根目录(P1)
- CMAKE_ARGS非数组，参数含空格时错误拆分
- main通过管道传给gawk，exit在subshell中无法正确传播

### 风险2: matmul模块缺陷密度最高

matmul/bmm贡献67条缺陷(17.6%)，6个matmul文件进入热点列表。缺陷跨越host/kernel/op_api三层，涵盖空指针(P0)、CHECK_RET检查错变量(P0)、copy-paste公式(P0)、除零(P1)、溢出截断(P2)。matmul代码复杂度高，需重点审查。

### 风险3: 编译warning隐藏真bug

21条编译告警修复中至少3条包含真正的逻辑bug(无符号无限循环、链式比较、运算符优先级)。开启`-Wall -Werror`可系统性拦截此类缺陷。

### 风险4: ascendc_config.json配置混乱

多个算子存在重复条目(AdaptiveAvgPool3d行2/5, LayerNormQuant行571/600/601三份)和同名冲突配置(compute_units含/不含特定平台)，构建行为不确定。

### 风险5: 先用后查反模式在op_api层流行

空指针解引用后再判空、OP_CHECK后先LOG再CHECK、工厂函数返回值直接使用 --- 这些"先用后查"模式在op_api层多处存在，是crash的高发源。

---

## 模块分布与缺陷密度

高缺陷模块(按绝对频次):
1. matmul/bmm: 67 (17.6%)
2. other(杂项): 122 (32.1%) --- 分散在多个小模块
3. norm: 24 (6.3%)
4. conv: 22 (5.8%)
5. quant: 20 (5.3%)
6. scatter/gather: 19 (5.0%)
7. build/compile: 18 (4.7%)
8. pooling: 16 (4.2%)

matmul是当之无愧的缺陷热点，其缺陷类型覆盖了12个类别中的9个(除文档、构建配置、平台适配外)。
