## ops-nn项目缺陷模式与审查规则

基于ops-nn仓库1474次提交的完整git历史分析，从380条缺陷提交（占比25.8%）中提炼的39条高价值审查规则。每条规则均有commit证据和实际代码支撑。

ops-nn是NN算子仓库，覆盖matmul、conv、pooling、norm、quant、scatter等模块。缺陷模式与通信库(hcomm/hccl)有本质差异——通信库侧重并发/协议/资源生命周期，算子仓库侧重精度/tiling/shape/数据类型/kernel参数。本文档的12个缺陷类别从380条缺陷数据中自然涌现，未套用其他仓库框架。

### 严重等级定义

| 等级 | 含义 | 影响范围 |
|------|------|----------|
| P0 | 致命 | CoreDump、数据损坏、内存越界、死锁/无限循环 |
| P1 | 严重 | 特定配置崩溃、静默精度错误、功能不可用 |
| P2 | 一般 | 边界条件异常、构建失败、测试不通过 |
| P3 | 建议 | 代码质量、可维护性、潜在隐患 |

### 缺陷分布总览

| 序号 | 缺陷类别 | 频次 | 占比 | 规则数 | 规则ID前缀 |
|------|---------|------|------|--------|-----------|
| 1 | 文档与示例代码 | ~36 | 11.2% | 3 | DOC |
| 2 | 构建系统与配置缺陷 | ~35 | 10.9% | 4 | BUILD |
| 3 | 条件判断与边界处理 | ~29 | 9.0% | 3 | LOGIC |
| 4 | 整数溢出与类型安全 | ~25 | 7.8% | 3 | INT |
| 5 | 空指针与资源管理 | ~22 | 6.9% | 4 | NULL |
| 6 | 编译告警与代码规范 | ~21 | 6.5% | 2 | WARN |
| 7 | 输入校验与接口一致性 | ~18 | 5.6% | 3 | VALID |
| 8 | Shape与Tiling计算 | ~16 | 5.0% | 2 | SHAPE |
| 9 | 复制粘贴与变量引用 | ~16 | 5.0% | 3 | COPY |
| 10 | 多核/流水线并发 | ~15 | 4.7% | 3 | SYNC |
| 11 | 平台适配与硬件约束 | ~10 | 3.1% | 2 | HW |
| 12 | 功能回退与搭车提交 | ~8 | 2.5% | 2 | REV |
| - | 跨类别系统性风险 | - | - | 5 | SYS |

模块缺陷密度: matmul/bmm模块贡献67条缺陷(17.6%)，是缺陷热点第一名，覆盖12个类别中的9个。

---

### 类别一：文档与示例代码（~36条，11.2%）

非纯文档修复（纯文档已在阶段1过滤），指代码仓中的示例代码bug、文档与代码不一致、op_list条目遗漏等。本类别特殊之处：示例代码中的bug会直接被用户复制使用，影响范围大。

#### 规则 DOC-01: 示例代码未经编译/运行验证

严重等级: P2

缺陷描述: 示例代码包含编译错误（重复函数定义、变量未声明）或运行时输出全0/垃圾数据，说明示例从未被实际编译和运行验证。用户拷贝示例代码后直接编译失败或得到错误结果。

典型代码示例:

```cpp
// 缺陷代码 — 501d38ab (aclnnWeightQuantBatchMatmulV2示例)
// AclnnWeightQuantBatchMatmulV2Test函数被定义了两次（copy-paste遗留）
void AclnnWeightQuantBatchMatmulV2Test() { /* 第一份 */ }
// ...
void AclnnWeightQuantBatchMatmulV2Test() { /* 第二份，重复定义 */ }
// 用户拷贝编译直接报redefinition错误

// 修复代码
// 删除重复的函数定义，保留一份
```

```cpp
// 缺陷代码 — 4063f8f9 (aclnnTransposeBatchMatMul示例)
// FP16数据初始化错误，示例输出全0
// 示例从未被实际运行验证

// 修复代码 — 修正数据初始化，确保输出正确
```

审查检查方法:
- 示例代码PR必须在CI中包含编译验证步骤
- 示例代码中不应有重复的函数/类定义
- 新增算子的示例应有至少一个非零输出的验证

关联commit: `501d38ab`, `4063f8f9`

---

#### 规则 DOC-02: 示例代码数据类型与API不匹配

严重等级: P1

缺陷描述: 示例中host数据容器类型(如`vector<int32_t>`)与创建tensor时传入的aclDataType(如ACL_FLOAT)不匹配，导致数据被错误解释。同时示例中aclrtMalloc/aclrtFree不配对导致内存泄漏。

典型代码示例:

```cpp
// 缺陷代码 — 964be4dd (deep_norm_grad示例)
std::vector<int32_t> hostData = {...};
// 但创建tensor时传ACL_FLOAT → 整数比特被解释为浮点数
aclTensor* tensor = aclCreateTensor(..., ACL_FLOAT, ...);
// 另外deep_norm示例缺少aclrtFree导致GPU显存泄漏

// 修复代码
std::vector<float> hostData = {...};  // 类型匹配ACL_FLOAT
aclTensor* tensor = aclCreateTensor(..., ACL_FLOAT, ...);
// 补充aclrtFree释放
```

审查检查方法:
- 示例中`vector<T>`的T必须与`aclDataType`对应: int32_t-ACL_INT32, float-ACL_FLOAT, half-ACL_FLOAT16
- 检查aclrtMalloc/aclrtFree是否配对出现
- 所有资源获取(Create)必须有对应释放(Destroy/Free)

关联commit: `964be4dd`

---

#### 规则 DOC-03: op_list条目遗漏或名称不一致

严重等级: P2

缺陷描述: 新增算子时op_list.md中条目遗漏或名称与实际算子名不一致。更严重的case：在添加算子A的MR中误添加了算子B的op_list条目（搭车），而A的条目反而遗漏。

典型代码示例:

```markdown
# 缺陷代码 — d91e1757a (unsorted_segment_sum迁移MR)
# MR标题是新增unsorted_segment_sum，但在op_list.md中
# 新增了scatter算子条目（不相关），unsorted_segment_sum自身条目遗漏

# 修复代码
# op_list.md新增条目名称必须与算子实际名称一致
```

审查检查方法:
- 新增算子时op_list.md条目名称必须与算子注册名完全一致
- PR中对op_list.md的改动应只包含与PR标题功能直接相关的条目
- 新增算子PR应包含对应的op_list条目（遗漏检查）

关联commit: `d91e1757a`(revert事件1)

---

### 类别二：构建系统与配置缺陷（~35条，10.9%）

CMakeLists依赖声明缺失/错误、binary.json平台配置遗漏、build.sh语法错误、ascendc_config.json重复/冲突条目。ops-nn构建系统(build.sh + cmake)是缺陷聚集度最高的区域，热点文件Top3全部是构建相关。

#### 规则 BUILD-01: CMake target依赖声明缺失

严重等级: P2

缺陷描述: CMake中custom_command/custom_target的DEPENDS未包含前置target，构建时序不确定导致间歇性构建失败。表现为本地编译通过但CI偶发失败（依赖并行构建调度顺序）。

典型代码示例:

```cmake
# 缺陷代码 — 640a1683
# ascendc_impl_gen的DEPENDS未包含前置target(ops_info_gen_*)
add_custom_target(ascendc_impl_gen
    DEPENDS ${impl_gen_outputs}  # 缺少对ops_info_gen_*的依赖
)

# 修复代码
add_custom_target(ascendc_impl_gen
    DEPENDS ${impl_gen_outputs} ops_info_gen_${compute_unit}
)
```

热点补充（阶段4 P0确认）:

```cmake
# gen_ops_info.cmake:654 — 循环后遗留变量作为target名
# foreach循环结束后调用compile_from_config
compile_from_config(TARGET ascendc_bin_${compute_unit}_${op_name} ...)
# ${op_name}仅为最后一次迭代的值，非预期target
```

审查检查方法:
- CMake custom_command/custom_target的DEPENDS必须包含所有输入依赖
- foreach循环结束后不应使用循环变量（取值为最后一次迭代的值）
- 间歇性构建失败首先排查DEPENDS是否完整

关联commit: `640a1683`

---

#### 规则 BUILD-02: shell脚本语法/逻辑错误

严重等级: P1

缺陷描述: shell脚本中`[`/`]`前后缺空格导致条件判断永远false/true、数组变量未正确展开、选项flag赋值错误。此类bug在本地运行时可能被掩盖（依赖环境变量/路径是否恰好存在），在CI中才暴露。

典型代码示例:

```bash
# 缺陷代码 — adec83fb (check_example.sh)
[ $status -ne 0]
# ']'与'0'粘连，test命令寻找名为"0]"的值，条件永远false
# 编译失败时CI仍显示成功

# 修复代码
[ $status -ne 0 ]
```

热点补充（阶段4 P0确认）:

```bash
# build.sh:1060-1065 — 数组备份只取首元素
local TMP_BUILD_LIBS=${BUILD_LIBS}     # ${BUILD_LIBS}只取数组第一个元素
# 恢复时
BUILD_LIBS=${TMP_BUILD_LIBS}           # 变成字符串，多target构建数据丢失

# build.sh:796 — 启用选项却赋值FALSE
mssanitizer) ENABLE_MSSANITIZER=FALSE ;;  # 应为TRUE

# build.sh:779 — sed空操作(替换前后相同)
COMPUTE_UNIT=$(echo "$COMPUTE_UNIT" | sed 's/ascend950/ascend950/g')
```

审查检查方法:
- CI脚本必须通过shellcheck静态分析
- `[`和`]`前后必须有空格
- 数组变量使用`"${arr[@]}"`展开，而非`${arr}`
- 赋值语句的值与意图一致（启用=TRUE，禁用=FALSE）

关联commit: `adec83fb`

---

#### 规则 BUILD-03: 新平台binary配置遗漏

严重等级: P2

缺陷描述: 新增算子或新增硬件平台时，遗漏目标平台的binary.json配置文件，导致该平台完全无法使用该算子。编译阶段不报错，运行时找不到kernel。

典型代码示例:

```
# 缺陷代码 — d6acf37e
# unique_consecutive算子缺少ascend950的binary.json
# 该平台上算子完全不可用

# 修复代码
# 新增 index/unique_consecutive/op_host/config/ascend950/binary.json
```

审查检查方法:
- 新增算子时检查是否为所有目标平台（ascend910/ascend950等）都提供了binary.json
- 新增平台支持时检查现有算子是否需要同步添加binary配置
- binary.json中的compute_units列表应与实际支持的平台一致

关联commit: `d6acf37e`

---

#### 规则 BUILD-04: ascendc_config.json重复/冲突配置

严重等级: P1

缺陷描述: ascendc_config.json中存在完全重复的条目或同名但配置内容冲突的条目（如compute_units列表不同）。哪一条生效取决于JSON解析器实现，构建行为不确定。

典型代码示例:

```json
// 缺陷 — 完全重复条目
// AdaptiveAvgPool3d在行2和行5各有一份完全相同的配置
// LayerNormQuant在行571/600/601有三份

// 缺陷 — 同名但冲突的配置
// AdaptiveAvgPool3dGrad: 一份compute_units含ascend950，另一份不含
// STFT: 一份含ascend910_93，另一份不含

// 无效配置
// compile_options中引用"ascend910_95"(行283/363)
// 不在compute_units列表中且非已知平台标识
// 疑似"ascend910_93"或"ascend950"的拼写错误
```

审查检查方法:
- 修改ascendc_config.json时检查同名算子是否已有条目
- 同名算子的配置（compute_units、compile_options等）必须一致
- 平台标识符应与已知平台列表匹配，避免拼写错误

关联commit: 热点分析(hotspot_analysis.md)

---

### 类别三：条件判断与边界处理（~29条，9.0%）

逻辑运算符错误(||/&&混淆)、恒真/恒假表达式、off-by-one、尾块处理遗漏、条件分支覆盖不全。ops-nn中`!=A || !=B`恒真模式尤其高频。

#### 规则 LOGIC-01: !=A || !=B恒真逻辑错误

严重等级: P0

缺陷描述: `x != A || x != B`（当A != B时）恒为true，属于De Morgan律的经典误用。本意是"x既不是A也不是B"（应写`x != A && x != B`），实际变成"x不是A，或者x不是B"（永远为真）。在ops-nn中多次独立出现，导致合法输入被拒绝或错误的代码路径被选中。

典型代码示例:

```cpp
// 缺陷代码 — 67c665fd (avg_pool_v2_grad)
if (inputDimNum != CHW_DIMS || inputDimNum != NCHW_DIMS) {
    return GRAPH_PARAM_INVALID;  // 恒为true，所有输入被拒绝
}
// 修复代码
if (inputDimNum != CHW_DIMS && inputDimNum != NCHW_DIMS) {
    return GRAPH_PARAM_INVALID;
}
```

```cpp
// 缺陷代码 — 692f43a9 (adaptive_pool3d_tiling.cpp)
if (npuArch != NpuArch::DAV_3510 || npuArch != NpuArch::DAV_5102) {
    return GRAPH_PARAM_INVALID;  // 恒为true，所有平台返回错误
}
// 修复代码 — 删除第二个条件，只保留单条件判断
if (npuArch != NpuArch::DAV_3510) {
    return GRAPH_PARAM_INVALID;
}
```

审查检查方法:
- `x != A || x != B`是经典恒真逻辑错误，应作为clang-tidy静态检查规则
- 逻辑运算符&&/||修改时画真值表验证
- 条件表达式中多个!=用||连接时99%应该用&&

关联commit: `67c665fd`, `692f43a9`

---

#### 规则 LOGIC-02: 新增类型/分支改变默认路径行为

严重等级: P1

缺陷描述: 在条件判断中新增一个类型分支时，if-else结构的默认(else)路径原本服务于已有类型，但新分支的插入改变了条件匹配顺序，导致原有类型走入错误路径。

典型代码示例:

```cpp
// 缺陷代码 — 4d315436 (QuantBatchMatmulV4)
// 新增DT_INT4后k0值选择逻辑反转
int k0 = (dtype == DT_INT4) ? INT8_K0 : INT4_K0;
// 条件搞反了: int4应该用INT4_K0，但实际用的是INT8_K0

// 修复代码
int k0 = (dtype == DT_INT4) ? INT4_K0 : INT8_K0;
```

审查检查方法:
- 新增类型/分支时，三元表达式的true分支应对应"新增的特殊情况"
- else(默认分支)必须保持原有行为不变——用原有case做回归测试
- if-else-if链中新增条件后，检查所有已有case是否仍匹配正确的分支

关联commit: `4d315436`

---

#### 规则 LOGIC-03: 尾块/边界条件处理遗漏

严重等级: P1

缺陷描述: 数据不能被tile/block整除时尾块处理遗漏（最后一块大小不同）、除零防护缺失、动态shape维度为0时的特殊处理遗漏。

典型代码示例:

```cpp
// 缺陷代码（通用模式）
uint32_t tileCount = totalSize / tileSize;  // 不能整除时尾块丢失
for (uint32_t i = 0; i < tileCount; i++) { ... }

// 修复代码
uint32_t tileCount = CeilDiv(totalSize, tileSize);  // 向上取整
uint32_t lastTileSize = totalSize - (tileCount - 1) * tileSize;  // 尾块特殊处理
```

审查检查方法:
- tile/分块循环中最后一块的大小是否与其他块不同
- 所有除法的除数是否可能为零
- 动态shape场景中维度值可能为0时是否有提前拦截

关联commit: 多条，详见defect_analysis.md中的边界处理类缺陷

---

### 类别四：整数溢出与类型安全（~25条，7.8%）

int32溢出(大shape/大元素数)、有符号/无符号混比、类型转换截断(uint64->uint32)、GM偏移在小类型域计算溢出。ops-nn中matmul/quant模块是此类缺陷的高发区。

#### 规则 INT-01: shape维度乘法int32溢出

严重等级: P0

缺陷描述: shape维度值或元素总数使用int32_t/uint32_t存储，当维度乘积超过2^31-1（约21亿）时溢出为负值或截断为小正数。在大shape场景(如大batch x 大sequence)下buffer分配不足或计算错误。

典型代码示例:

```cpp
// 缺陷代码 — 8f6ccaea (addlayernormgrad)
uint32_t roundUpNumLastDimFloat = ROUND_UP(numLastDim, ...);
// numLastDim > 2^31时ROUND_UP结果已溢出
// 后续 roundUpNumLastDimFloat * sizeof(float) 再次溢出

// 修复代码
uint64_t roundUpNumLastDimFloat = ROUND_UP(static_cast<uint64_t>(numLastDim), ...);
```

```cpp
// 缺陷代码 — acadb4c7 (repeat_interleave)
int32_t repeatSum = 0;
for (...) { repeatSum += repeats[i]; }  // 超INT32_MAX时溢出
// 未区分是否需要int64

// 修复代码
// 增加int64分支，超INT32_MAX时使用int64_t
```

审查检查方法:
- shape维度乘法变量必须使用int64_t/uint64_t
- tiling结构体中shape/元素数相关字段的类型是否足够宽
- ROUND_UP/CeilAlign等宏的输入值和输出值类型一致性

关联commit: `8f6ccaea`, `acadb4c7`

---

#### 规则 INT-02: GM偏移量在小类型域计算溢出

严重等级: P0

缺陷描述: GlobalTensor下标表达式`index * maxDataCount`中，当index为uint16_t/uint32_t时乘法在小类型域完成后溢出，结果才赋给uint64_t。此模式在ops-nn的foreach算子系列中批量出现（10个文件涉及8-9个算子，包括foreach_copy、foreach_lerp、foreach_norm、foreach_pow等）。

典型代码示例:

```cpp
// 缺陷代码 — 50df91e7 (foreach系列10个文件)
// index为uint16_t/uint32_t，maxDataCount为uint32_t
DataCopy(dataLocal, inTensorsGM[index * maxDataCount], dataCount);
// 乘法在uint32_t域完成后溢出，然后才被提升为指针偏移的uint64_t

// 修复代码
DataCopy(dataLocal, inTensorsGM[1ULL * index * maxDataCount], dataCount);
// 1ULL *强制提升为uint64_t后再乘
```

审查检查方法:
- GM内存偏移计算表达式中至少一个操作数必须为64位类型
- 乘法表达式在赋值之前就已溢出（赋给uint64_t也救不了）
- 批量审查: 搜索`Gm[expr * expr]`或`inputGm[expr * expr]`模式

关联commit: `50df91e7`

---

#### 规则 INT-03: 有符号/无符号混比与类型不一致

严重等级: P2

缺陷描述: 无符号常量与有符号shape值比较引发-Wsign-compare告警; 模板参数COMP_T为int32时局部变量声明为uint64_t导致不必要的类型提升; 类型不一致在特定输入组合下产生错误结果。

典型代码示例:

```cpp
// 缺陷代码 — 941d3c7f
// kSupportedInnerAxis为uint64_t常量，shape维度值为int64_t
if (shape.GetDim(i) > kSupportedInnerAxis) { ... }
// -Wsign-compare: 有符号/无符号混合比较

// 修复代码
static constexpr int64_t kSupportedInnerAxis = ...;  // 类型一致
```

```cpp
// 缺陷代码 — 0253078d (scatter_elements_v2)
// 模板COMP_T为int32_t时，stride数组声明为uint64_t
uint64_t stride[8];  // 与COMP_T不一致，导致VF性能退化

// 修复代码
COMP_T stride[8];    // 使用模板参数类型
```

审查检查方法:
- 开启`-Wsign-compare`并视为错误
- 常量类型与比较对象类型一致
- 使用模板参数COMP_T的函数中，局部变量类型应与COMP_T一致

关联commit: `941d3c7f`, `0253078d`

---

### 类别五：空指针与资源管理（~22条，6.9%）

工厂函数返回值未检查、先解引用后判空、结构体成员未初始化(野指针)、PipeBarrier与FreeTensor时序颠倒。ops-nn的op_api层是此类缺陷的高发区。

#### 规则 NULL-01: OP_CHECK宏缺少return语句

严重等级: P0

缺陷描述: OP_CHECK宏的第三参数（失败时的动作）写成裸值`nullptr`而非`return nullptr`。OP_CHECK失败时不会return而继续向下执行，使用空指针导致crash。此模式在单个文件中可批量出现（如aclnn_median.cpp中14处）。

典型代码示例:

```cpp
// 缺陷代码 — 7b4a1b53 (aclnnNanMedian)
OP_CHECK(condition, "error msg", nullptr);
// 失败时执行 nullptr; 作为表达式语句 → 什么都不做，继续执行

// 修复代码
OP_CHECK(condition, "error msg", return nullptr);
// 失败时return nullptr，函数正确返回
```

审查检查方法:
- OP_CHECK宏的第三参数必须包含return语句
- 静态分析: 检测`, nullptr)`和`, nullptr;)`模式
- 批量审查: 全文搜索`OP_CHECK.*,\s*nullptr\)`

关联commit: `7b4a1b53`

---

#### 规则 NULL-02: 结构体指针成员未初始化

严重等级: P0

缺陷描述: C++结构体中指针成员未在声明处初始化，某些代码路径下成员值为野指针。后续无条件使用（如OP_LOGE("%s", result.logMessage)）导致崩溃。

典型代码示例:

```cpp
// 缺陷代码 — 15e40a48 (addmm matmul_util.cpp)
struct PromoteResult {
    const char* logMessage;  // 未初始化 → 野指针
    aclDataType dtype;
};
// 某些GetUpperDtypeByLookUpTable返回路径不设置logMessage
OP_LOGE("%s", result.logMessage);  // 崩溃

// 修复代码
struct PromoteResult {
    const char* logMessage = nullptr;  // 声明处初始化
    aclDataType dtype = ACL_DT_UNDEFINED;
};
// 使用前判空
if (result.logMessage != nullptr) { OP_LOGE("%s", result.logMessage); }
```

审查检查方法:
- C++结构体指针成员必须在声明处初始化为nullptr
- 所有成员变量应在声明处初始化（C++11特性）
- OP_LOGE中`%s`对应的参数必须保证非空

关联commit: `15e40a48`

---

#### 规则 NULL-03: PipeBarrier与FreeTensor时序颠倒

严重等级: P0

缺陷描述: PipeBarrier<PIPE_V>放在FreeTensor之后，导致Cast/向量运算未完成就释放了输入buffer。运行时表现为随机精度错误或数据损坏。同时还可能缺少V_MTE3/MTE3_V同步事件。

典型代码示例:

```cpp
// 缺陷代码 — cf84e222 (AddRmsNorm)
FreeTensor(inputBuffer);                // 先释放
PipeBarrier<PIPE_V>();                  // 后等待 → Cast可能还没完成
// 缺少MTE3_V同步 → 向量运算可能读到未就绪的数据

// 修复代码
PipeBarrier<PIPE_V>();                  // 先等待Cast完成
FreeTensor(inputBuffer);                // 后释放
// 补充SetFlag<PIPE_MTE3, PIPE_V>() / WaitFlag<PIPE_MTE3, PIPE_V>()
```

审查检查方法:
- PipeBarrier必须在FreeTensor之前（释放前确保计算完成）
- 检查数据流: MTE2→V→MTE3的每个阶段转换是否有对应的同步事件
- 搜索FreeTensor调用，确认前面都有PipeBarrier

关联commit: `cf84e222`

---

#### 规则 NULL-04: 先解引用后判空(先用后查)

严重等级: P0

缺陷描述: 指针先被解引用使用（Contiguous/Cast/TransData等操作），之后才检查是否为nullptr。如果指针确实为空，已经崩溃在检查之前。此模式在op_api层多处存在。

典型代码示例:

```cpp
// 缺陷代码 — hotspot分析 (aclnn_addmm.cpp:298-299)
auto selfContiguous = l0op::Contiguous(addmmTensor.self, uniqueExecutor);
// self已被解引用（Contiguous内部使用self）
if (addmmTensor.self != nullptr && ...)  // 检查太晚

// 缺陷代码 — hotspot分析 (aclnn_quant_matmul_v5.cpp:907-908)
reformatedX1 = l0op::TransData(reformatedX1, Format::FORMAT_FRACTAL_NZ, 0, executor);
CHECK_RET(x1 != nullptr, ACLNN_ERR_INNER_NULLPTR);
// 赋值给reformatedX1但检查的是x1 — 检查错误变量
```

审查检查方法:
- 指针解引用（包括作为函数参数传入）必须在nullptr检查之后
- CHECK_RET中检查的变量必须是刚被赋值的变量（而非赋值前的旧变量）
- 搜索模式: `= l0op::Xxx(ptr); if (ptr != nullptr)` 或 `CHECK_RET(oldVar != nullptr)`

关联commit: 热点分析 aclnn_addmm.cpp, aclnn_quant_matmul_v5.cpp

---

### 类别六：编译告警与代码规范（~21条，6.5%）

此类别特殊之处：表面上是修复warning，但warning背后经常隐藏着真正的逻辑bug。21条编译告警修复中至少3条包含真正的逻辑错误。

#### 规则 WARN-01: 无符号整数>=0恒真导致无限循环

严重等级: P0

缺陷描述: `for(uint64_t i = N; i >= 0UL; i--)`中`i >= 0`对无符号整数恒为true。当i减到0后再减1会wrap around到UINT64_MAX，循环永不终止。此bug隐藏在-Wtautological-compare告警中。

典型代码示例:

```cpp
// 缺陷代码 — 0e00f88c (avg_pool3d_grad等5处)
for (uint64_t i = singleCoreWo; i >= 0UL; i--) {
    // i是uint64_t，>=0恒为true → 无限循环
    ProcessTile(i);
}

// 修复代码
for (uint64_t i = singleCoreWo + 1; i > 0; i--) {
    ProcessTile(i - 1);  // 避免无符号下溢
}
// 或改用int64_t
```

审查检查方法:
- 无符号整数循环中`>=0`条件是无限循环bug
- 开启`-Wtautological-compare`可自动检测
- 无符号递减循环应使用`i > 0`配合`i - 1`，或改用有符号类型

关联commit: `0e00f88c`

---

#### 规则 WARN-02: 链式比较与运算符优先级错误

严重等级: P1

缺陷描述: `a == b == 1`在C++中被解析为`(a == b) == 1`，即"a是否等于b"的布尔结果与1比较，语义完全偏离"a和b都等于1"的意图。同时`&&`优先级高于`||`，`X && Y || Z`实际为`(X && Y) || Z`而非`X && (Y || Z)`。

典型代码示例:

```cpp
// 缺陷代码 — 50854088 (aclnn_quant_matmul_v4.cpp等约14个文件)
if (tensor->GetViewShape().GetDim(firstLastDim) ==
    tensor->GetViewShape().GetDim(secondLastDim) == 1) { ... }
// 解析为 (dim1 == dim2) == 1，即dim1和dim2相等时为true

// 修复代码 — 去掉链式比较中的== 1
if (tensor->GetViewShape().GetDim(firstLastDim) ==
    tensor->GetViewShape().GetDim(secondLastDim)) { ... }

// 缺陷代码 — 50854088
if (static_cast<ge::Format>(...) == op::Format::FORMAT_FRACTAL_NZ &&
    (tensor->GetViewShape().GetDim(dim2) == 1) ||
    (tensor->GetViewShape().GetDim(dim1) == 1)) { ... }
// 解析为 (FORMAT_NZ && dim2 == 1) || dim1 == 1
// 意图是 (FORMAT_NZ && dim2 == 1) || dim1 == 1 的分组方式不同

// 修复代码 — 用括号明确分组
if ((static_cast<ge::Format>(...) == op::Format::FORMAT_FRACTAL_NZ &&
    (tensor->GetViewShape().GetDim(dim2) == 1)) ||
    (tensor->GetViewShape().GetDim(dim1) == 1)) { ... }
```

审查检查方法:
- `a == b == c`链式比较在C++中语义错误，应写成`a == b && b == c`
- `&&`和`||`混合使用时必须加括号（开启`-Wlogical-op-parentheses`）
- 建议全局开启`-Wall -Werror -Wsign-compare -Wshadow`

关联commit: `50854088`

---

### 类别七：输入校验与接口一致性（~18条，5.6%）

dtype默认参数与实际不一致、通过硬编码索引获取属性(不同算子attr顺序不同)、变体接口复用通用校验函数遗漏特有限制。

#### 规则 VALID-01: 接口调用默认参数与实际数据不一致

严重等级: P1

缺陷描述: 调用下层接口时未传入实际数据类型，使用了接口的默认参数值。当用户传入的数据类型与默认值不同时产生错误结果。

典型代码示例:

```cpp
// 缺陷代码 — c8ca6bec (AdaptiveMaxPool3d)
// 调用MaxPool3D接口时未传indicesDtype
MaxPool3D(input, output, indices);  // indicesDtype默认int32
// 但用户可能传int64的indices → 类型不匹配

// 修复代码
MaxPool3D(input, output, indices, indicesDtype);  // 显式传入实际类型
```

审查检查方法:
- 调用具有默认参数的接口时，检查实际数据类型是否可能与默认值不同
- 尤其关注dtype/format等有多种取值可能的参数
- 显式传参优于依赖默认值

关联commit: `c8ca6bec`

---

#### 规则 VALID-02: 属性索引硬编码顺序依赖

严重等级: P1

缺陷描述: 通过`GetAttrPointer<T>(index)`硬编码索引获取算子属性，但不同算子类型（如MatMul vs GemmV3）的attr排列顺序不同。在一种算子上正确的索引在另一种上读到完全不同的属性。

典型代码示例:

```cpp
// 缺陷代码 — 07e77ddd (InplaceAddmm走GemmV3路径)
auto transA = GetAttrPointer<bool>(0);  // 期望读transpose_a
auto transB = GetAttrPointer<bool>(1);  // 期望读transpose_b
// 但GemmV3的attr顺序是: [alpha, beta, transpose_a, transpose_b]
// 索引0/1实际读到了alpha/beta

// 修复代码
// GemmV3路径使用索引2/3，或优先使用属性名获取
auto transA = GetAttrPointer<bool>(2);
auto transB = GetAttrPointer<bool>(3);
```

审查检查方法:
- 通过索引获取算子属性时，确认不同算子类型的attr排列顺序是否一致
- 优先使用属性名获取（GetAttrByName）而非索引
- 复用通用代码但attr布局不同时必须做分支处理

关联commit: `07e77ddd`

---

#### 规则 VALID-03: 变体接口复用通用dtype校验

严重等级: P1

缺陷描述: 变体接口（如WeightNz版本）复用通用dtype校验函数，但变体接口有更严格的类型限制（如不支持int8/fp32）。通用校验放行了变体不支持的类型，运行时crash。同时入参nullptr未拦截直接传入导致core dump。

典型代码示例:

```cpp
// 缺陷代码 — 0bd3fe0b (WeightNz接口)
// 复用通用dtype校验允许了int8/fp32
bool isValid = CommonDtypeCheck(input);  // 通用函数允许int8
// 但WeightNz接口不支持int8 → 运行时crash
// 同时 input 为 nullptr 时未拦截

// 修复代码
if (input == nullptr) { return ACLNN_ERR_INNER_NULLPTR; }
bool isValid = WeightNzDtypeCheck(input);  // 独立的变体校验
```

审查检查方法:
- 变体接口应使用独立dtype校验而非复用通用函数
- aclnn函数入口第一步应做所有必选输入的nullptr检查
- 校验函数的允许列表应与接口实际支持的类型严格一致

关联commit: `0bd3fe0b`

---

### 类别八：Shape与Tiling计算（~16条，5.0%）

StorageShape/ViewShape混淆、tiling key参数错误、Shape校验维度不全、dead code。算子库特有的高风险类别，可审查性中等。

#### 规则 SHAPE-01: StorageShape/ViewShape混淆

严重等级: P1

缺陷描述: 非连续张量的StorageShape（物理内存布局的shape）与ViewShape（用户视角的逻辑shape）不同。shape校验、维度计算使用GetStorageShape()获取shape，但非连续张量的StorageShape可能大于ViewShape，导致合法输入被错误拦截或维度计算错误。

典型代码示例:

```cpp
// 缺陷代码 — 6b52fdcf (LSTM)
// CheckFormatValid统一用FORMAT_ND校验，但h/c/门控实际为FORMAT_NCL
CheckFormatValid(tensor, FORMAT_ND);  // 应区分FORMAT_NCL

// ValidateInputShape用GetStorageShape()
auto shape = input.GetStorageShape();  // 非连续张量StorageShape != ViewShape
if (shape.GetDim(0) != expectedDim) { return INVALID; }

// 修复代码
auto shape = input.GetViewShape();    // 优先使用逻辑shape
```

```cpp
// 缺陷代码 — 28ab6d70 (thnn_fused_lstm_cell_backward)
// 经view操作后StorageShape与逻辑形状不符
auto dim = tensor.GetStorageShape().GetDim(1);  // 物理shape，可能不对

// 修复代码
auto dim = tensor.GetViewShape().GetDim(1);
```

审查检查方法:
- shape校验优先使用GetViewShape()，仅在需要物理布局时用GetStorageShape()
- 非连续张量场景（view/transpose/slice后）必须区分StorageShape和ViewShape
- format校验应按张量实际格式区分，不应统一强制FORMAT_ND

关联commit: `6b52fdcf`, `28ab6d70`

---

#### 规则 SHAPE-02: tiling key参数传递错误与dead code

严重等级: P1

缺陷描述: tiling key参数传递错误导致kernel选择了错误的计算策略; return语句后的SetAtomicNone等代码永远不执行(dead code)。

典型代码示例:

```cpp
// 缺陷代码 — 6eee5478
// tiling key参数传递错误 → 选择错误策略
SetTilingKey(WRONG_KEY);  // 应为CORRECT_KEY

// return后的dead code
return;
SetAtomicNone();  // 永远不执行
```

审查检查方法:
- tiling key参数应有枚举约束而非裸字符串/魔法数
- return语句后不应有可执行代码（开启dead code检测）
- tiling key值与kernel侧的switch-case必须一一对应

关联commit: `6eee5478`

---

### 类别九：复制粘贴与变量引用（~16条，5.0%）

函数调用参数重复f(a,a)、维度索引copy-paste错误、偏移量计算公式套错维度步长。此类缺陷的共同特征：相邻代码行高度相似，只差一个变量名或索引值。

#### 规则 COPY-01: 函数调用参数重复f(a,a)

严重等级: P1

缺陷描述: 函数调用中两个参数使用了同一个变量，实际上应使用不同变量。最常见于从单输入版本复制到双输入版本时忘记改第二个参数。

典型代码示例:

```cpp
// 缺陷代码 — b2e2cada (addbmm)
bool isEmpty = isAddBmmProcessEmptyTensor(batch1, batch1);
// 第二参数应为batch2，copy-paste遗漏

// 修复代码
bool isEmpty = isAddBmmProcessEmptyTensor(batch1, batch2);
```

审查检查方法:
- 函数调用中两个参数完全相同`f(a, a)`时必须确认是否为copy-paste错误
- 尤其关注名称只差一个字符的参数（batch1/batch2, self/other, input/output）
- diff中新增代码与相邻已有代码高度相似时逐行比对变量名

关联commit: `b2e2cada`

---

#### 规则 COPY-02: 矩阵k/n维度索引粘贴错误

严重等级: P0

缺陷描述: 矩阵维度索引的k和n在转置/非转置时必须互补。copy-paste后k维度的公式和n维度完全相同，导致矩阵乘法的维度推导错误。同时还可能存在先解引用后判空的问题。

典型代码示例:

```cpp
// 缺陷代码 — 1c2de786 (matmul_common_infershape.cpp)
int64_t k_x2_dim = transB ? x2Shape.GetDim(dimNum - 1) : x2Shape.GetDim(dimNum - 2);
int64_t n_x2_dim = transB ? x2Shape.GetDim(dimNum - 1) : x2Shape.GetDim(dimNum - 2);
// n_x2_dim公式与k_x2_dim完全相同（粘贴错误）
// 应为互补索引: k取dimNum-1则n取dimNum-2，反之亦然

// 修复代码
int64_t k_x2_dim = transB ? x2Shape.GetDim(dimNum - 1) : x2Shape.GetDim(dimNum - 2);
int64_t n_x2_dim = transB ? x2Shape.GetDim(dimNum - 2) : x2Shape.GetDim(dimNum - 1);
```

热点补充（阶段4 P0确认）:

```cpp
// mat_mul_deterministic_splitk_kernel.h:404-405
uint64_t rowBlockNum = alignedN / 16;    // 应为alignedM / 16?
uint64_t colBlockNum = alignedN / 16;
// 两行公式完全相同，疑似copy-paste bug
```

审查检查方法:
- 矩阵k/n维度索引在转置/非转置情况下必须互补
- 相邻行的赋值公式完全相同时高度疑似copy-paste错误
- row/col相关变量应使用不同的维度(M vs N)

关联commit: `1c2de786`

---

#### 规则 COPY-03: GM偏移量公式维度混淆

严重等级: P1

缺陷描述: 多维偏移量计算将不同维度索引合并后统一乘同一步长，公式语义错误。正确做法是逐维展开：每个维度索引乘以对应维度的步长。

典型代码示例:

```cpp
// 缺陷代码 — 910dac63 (QuantUpdateScatter)
// gmVarOffset_计算: axisOffset应乘varDim3而非统一步长
gmVarOffset_ = (batchIdx * outerDim + axisOffset) * stride;
// axisOffset和outerDim的步长不同，不能合并乘

// 修复代码
gmVarOffset_ = batchIdx * outerDim * varDim3 + axisOffset * varDim3;
// 逐维展开
```

审查检查方法:
- 多维偏移量应逐维展开（各维索引 x 对应步长），避免合并表达式
- 括号中合并多个维度索引后乘以统一步长是危险模式
- 相邻行的变量名只差一个字符(如x1/x2, dim1/dim2)时重点核对

关联commit: `910dac63`

---

### 类别十：多核/流水线并发（~15条，4.7%）

SetScheduleMode缺失、SetFlag/WaitFlag顺序错误、DataCopy前后缺少MTE/V/S同步事件、workspace脏数据。此类缺陷可审查性最低，运行时表现为随机精度错误或间歇性崩溃。

#### 规则 SYNC-01: SetScheduleMode缺失

严重等级: P1

缺陷描述: tiling函数中缺少SetScheduleMode(1)调用，多核间存在数据依赖但未设置batch mode同步。运行时多核并行访问共享数据产生竞态。

典型代码示例:

```cpp
// 缺陷代码 — afb09c78 (dynamic_mx_quant, map_index)
void TilingFunc(TilingContext* context) {
    // 缺少SetScheduleMode
    context->SetBlockDim(blockDim);
    // 多核间存在数据依赖但无同步

    // 修复代码
    context->SetScheduleMode(1);  // 设置batch mode
    context->SetBlockDim(blockDim);
}
```

审查检查方法:
- 所有tiling函数必须显式设置SetScheduleMode
- 多核场景下是否存在数据依赖（共享内存读写、reduction操作）
- SetScheduleMode的参数值(0/1/2)是否与实际同步需求匹配

关联commit: `afb09c78`

---

#### 规则 SYNC-02: SetFlag/WaitFlag顺序错误

严重等级: P0

缺陷描述: SetFlag/WaitFlag配对位置不当或提前return路径遗漏event平衡。具体包括：WaitFlag与使用数据的操作距离过远导致同步不及时、提前return分支未补发SetFlag导致event计数失衡、ping-pong切换中WaitFlag位置不当。运行时表现为数据依赖顺序错误或event计数不平衡。

典型代码示例:

```cpp
// 缺陷代码 — ed0bd5a1 (QBMMv4 block_epilogue_pertile.h)
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
// ... 多步操作 ...
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);    // WaitFlag位置太靠后
// 同时提前return路径缺少SetFlag<V_MTE2> → event计数失衡

// 修复代码
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);    // 紧跟Set后立即Wait
// 提前return路径补充SetFlag<V_MTE2> + ping-pong翻转
```

审查检查方法:
- SetFlag/WaitFlag必须成对且顺序为: 数据操作 → SetFlag → (其他操作) → WaitFlag → 使用数据
- 同一eventId的Set和Wait必须在正确的顺序上
- pipe方向必须与实际数据流向一致(MTE2=数据搬入, V=向量计算, MTE3=数据搬出)

关联commit: `ed0bd5a1`

---

#### 规则 SYNC-03: DataCopy前后缺少流水线同步事件

严重等级: P0

缺陷描述: DataCopy(MTE2)前缺少MTE3_MTE2同步（上一轮搬出可能未完成就开始新搬入），向量运算前缺少MTE2_V同步（数据搬入未完成就开始计算）。运行时表现为相同输入产生不同输出（竞态条件）。

典型代码示例:

```cpp
// 缺陷代码 — 74d43ef3 (scatter_sub)
DataCopy(localBuffer, gmSrc, dataLen);     // MTE2搬入
// 缺少WaitFlag<PIPE_MTE2, PIPE_V>
VecAdd(result, localBuffer, localOther);   // V向量运算 → 可能读到未就绪数据

// 修复代码
DataCopy(localBuffer, gmSrc, dataLen);
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);      // 等待搬入完成
VecAdd(result, localBuffer, localOther);    // 数据已就绪
```

审查检查方法:
- DataCopy(MTE2)前确保前序MTE3完成
- 向量运算(V)前确保MTE2数据就绪
- 数据流每个阶段转换（MTE2→V→MTE3）必须有对应同步事件

关联commit: `74d43ef3`

---

### 类别十一：平台适配与硬件约束（~10条，3.1%）

DMA/datacopy指令参数超16位上限(65535)、uint16溢出截断、新芯片端云兼容性、硬件宏条件编译遗漏。

#### 规则 HW-01: DMA/datacopy参数超16位上限

严重等级: P0

缺陷描述: datacopy/DMA指令的stride/blockLen参数为uint16_t类型，最大值65535。当cin(输入通道)或数据量超过65535时参数截断，导致实际拷贝长度远小于预期，精度失败。

典型代码示例:

```cpp
// 缺陷代码 — 34217db7 (Conv3DBackpropInput)
// cin > 65535时stride参数超16位上限
uint16_t stride = cin * sizeof(float);  // 截断

// 修复代码 — tiling层校验并降级
if (cin > MAX_DATACOPY_STRIDE) {
    // 选择逐行处理策略而非大块DMA
    useFallbackPath = true;
}
```

```cpp
// 缺陷代码 — cf9ea8c9 (dynamic_quant)
// DMA参数uint16_t类型，stride超UINT16_MAX时截断溢出
uint16_t burstLen = dataSize;  // dataSize > 65535时截断

// 修复代码
// 运行时检测超限后降级为逐行处理
```

审查检查方法:
- datacopy/DMA stride参数使用前校验不超过65535（16位上限）
- 大数据量场景应使用Ext版本API或分块处理
- tiling层应有大shape的DMA参数溢出保护

关联commit: `34217db7`, `cf9ea8c9`

---

#### 规则 HW-02: 硬件宏条件编译遗漏

严重等级: P2

缺陷描述: 特定芯片专有指令（如Fixpipe）未用`__NPU_ARCH__`等硬件宏条件编译保护，在不支持该指令的芯片上编译/运行失败。端云兼容性问题导致多个算子在Kirin平台不可用。

典型代码示例:

```cpp
// 缺陷代码 — c1469b68 (Kirin适配)
// Fixpipe指令未加架构条件
FixPipe(dst, src, params);  // 某些芯片不支持

// 修复代码
#if defined(__NPU_ARCH__) && (__NPU_ARCH__ >= 910)
    FixPipe(dst, src, params);
#else
    // fallback路径
#endif

// 同时三个算子(quant_batch_matmul_v4, transpose_batch_mat_mul,
// max_pool3d_with_argmax_v2)需删除不兼容的Kirin binary配置
```

审查检查方法:
- 特定芯片指令需用`__NPU_ARCH__`等硬件宏条件编译保护
- 新芯片适配时应有端云兼容性检查清单
- binary.json中不支持的平台配置应删除而非留空

关联commit: `c1469b68`

---

### 类别十二：功能回退与搭车提交（~8条，2.5%）

大feature合入后紧急revert、MR中混入不相关改动(搭车)、revert时造成附带伤害。ops-nn共4个独立Revert事件暴露了系统性流程缺陷。

#### 规则 REV-01: 搭车提交造成revert附带伤害

严重等级: P2

缺陷描述: MR中混入与标题功能不相关的改动，revert时不相关改动被一并回滚。ops-nn的4个Revert事件中2个存在搭车问题。

典型案例:

```
# 事件1 — d91e1757a (unsorted_segment_sum)
# MR标题: 新增unsorted_segment_sum算子
# 搭车改动: op_list.md中添加了scatter条目(非本MR功能)
# Revert附带伤害: scatter条目也被删除

# 事件3 — 4482238c0 (LogSigmoid)
# MR标题: 新增LogSigmoid Ascend950 kernel
# 搭车改动: 修改了binary_cross_entropy的平台判断逻辑
# Revert附带伤害: binary_cross_entropy的修改也被回滚
```

审查检查方法:
- MR中每个文件改动应与标题描述的功能直接相关
- 不相关改动必须拆分为独立MR
- Review时对"顺便改了一下"的改动保持警惕

关联commit: `d91e1757a`, `4482238c0`

---

#### 规则 REV-02: 大规模提交测试不充分

严重等级: P2

缺陷描述: 大体量提交(>20文件/>1000行)一次性合入后紧急revert。常见问题：仅声称"冒烟测试通过"但缺乏全面集成测试；跨多层改动(op_api/tiling/kernel)的级联影响未充分验证。

典型案例:

```
# 事件1 — 38文件/8892行，revert间隔30小时
# 事件2 — 18文件/422行但跨三层，revert间隔1天
# 事件3 — 38文件/1190行，revert间隔13小时

# Revert事件2 — 852f21d6b (batchmatmul非连续输入)
# 核心缺陷: auto shadowing + 空指针检查缺失
# 影响: 不仅新功能出错，连正常路径也受影响
```

审查检查方法:
- 大feature合入(>20文件或>1000行)前需全流程CI + 集成测试
- 跨三层(op_api/tiling/kernel)的变更需要额外的回归验证
- 开启`-Wshadow`拦截变量遮蔽

关联commit: `852f21d6b`, `d91e1757a`, `4482238c0`

---

### 跨类别系统性风险

以下5个风险跨越多个缺陷类别，需要系统性应对而非逐条修复。

#### 规则 SYS-01: build.sh系统性脆弱

严重等级: P1（跨类别2/6/12，热点第1，10条缺陷触及）

缺陷描述: build.sh被10条缺陷提交触及，热点排名第1。当前代码仍存在多个P0/P1 bug且结构性问题严重：
- `set +u`从行35开始从未恢复，整个脚本期间未定义变量静默为空（拼写错误不报错）
- `]`前缺空格(P0)、数组备份只取首元素(P0)、--mssanitizer赋FALSE(P1)
- ASCEND_HOME_PATH未设置时路径变成`/include`等根目录(P1)
- CMAKE_ARGS非数组，参数含空格时错误拆分
- main通过管道传给gawk，exit在subshell中无法正确传播退出码

审查检查方法:
- build.sh修改必须通过shellcheck验证
- 恢复`set -u`（或至少在关键变量使用处加`${VAR:?}`保护）
- 数组操作使用`"${arr[@]}"`而非`${arr}`
- source使用`$SCRIPT_DIR/`前缀而非相对路径

关联commit: `adec83fb`及热点分析

---

#### 规则 SYS-02: matmul模块缺陷密度最高

严重等级: P0（跨类别3/4/5/9/10等9个类别，67条缺陷，17.6%）

缺陷描述: matmul/bmm模块贡献67条缺陷，是ops-nn缺陷热点第一名。6个matmul文件进入热点列表，缺陷跨越host/kernel/op_api三层。当前代码仍存在的确认bug：
- 空指针先用后查(P0): aclnn_addmm.cpp:298-299
- CHECK_RET检查错变量(P0): aclnn_quant_matmul_v5.cpp:907-908
- copy-paste公式相同(P0): mat_mul_deterministic_splitk_kernel.h:404-405
- 除零风险(P1): matmul_v3_base_tiling.cpp:605-606, 80-81
- 溢出截断(P2): matmul_v3_base_tiling.cpp:909-912

审查检查方法:
- matmul相关PR应提高审查标准（多人review）
- 重点检查空指针、copy-paste、维度索引、类型溢出
- matmul模块的代码复杂度高，新增功能需完整的回归测试矩阵

关联commit: 见热点分析中matmul部分

---

#### 规则 SYS-03: 编译warning隐藏真bug

严重等级: P1（跨类别3/6，至少3条warning修复中发现真逻辑bug）

缺陷描述: 21条编译告警修复提交中至少3条包含真正的逻辑bug：无符号>=0恒真无限循环(`0e00f88c`)、`a==b==1`链式比较语义错误(`50854088`)、`&&`/`||`优先级导致条件判断错误(`50854088`)。编译warning不是噪声，而是真bug的信号。

审查检查方法:
- 开启`-Wall -Werror -Wsign-compare -Wshadow -Wtautological-compare`
- 现有warning应逐一审查是否包含逻辑错误（不要机械性地消除warning）
- warning修复提交应认真review——它们可能暴露新的bug

关联commit: `0e00f88c`, `50854088`

---

#### 规则 SYS-04: ascendc_config.json配置混乱

严重等级: P2（跨类别2/11）

缺陷描述: ascendc_config.json当前存在的问题：
- 完全重复条目: AdaptiveAvgPool3d(2份)、DeformableOffsetsGrad(2份)、LayerNormQuant(3份)、GetPaddingOffset(3份)
- 同名冲突配置: AdaptiveAvgPool3dGrad/AdaptiveMaxPool3d/STFT/Split/SplitV/DeformableOffsets的compute_units/compile_options/impl_mode不一致
- 无效引用: compile_options中引用`ascend910_95`非已知平台标识
- 哪一条生效取决于JSON解析器实现，构建行为不确定

审查检查方法:
- 修改ascendc_config.json的PR应先搜索是否已有同名条目
- CI中应加入重复条目检测脚本
- 平台标识符应通过枚举/白名单约束

关联commit: 热点分析(hotspot_analysis.md)

---

#### 规则 SYS-05: 先用后查反模式在op_api层流行

严重等级: P0（跨类别5/9）

缺陷描述: 空指针先解引用后判空、OP_CHECK后先LOG再CHECK、工厂函数返回值直接使用不检查。这些"先用后查"模式在op_api层多处存在，是crash的高发源。热点分析中P0级别的8个确认bug中有3个属于此模式。

典型表现:
- `l0op::Contiguous(ptr)` → `if (ptr != nullptr)`（先解引用后判空）
- `l0op::Cast(...)` → `OP_LOGD(result->ToString())` → `OP_CHECK(result != nullptr)`（先用后查）
- `reformatedX1 = TransData(...)` → `CHECK_RET(x1 != nullptr)`（检查错误变量）

审查检查方法:
- op_api代码中搜索`= l0op::` / `= Contiguous(` / `= Cast(`后紧跟的if/CHECK_RET是否检查了正确的变量
- 工厂函数/算子调用返回值必须先检查再使用
- 日志输出不应在空指针检查之前调用返回值的方法

关联commit: 热点分析 aclnn_addmm.cpp, aclnn_quant_matmul_v5.cpp, aclnn_convolution_backward.cpp

---

### 附录：规则速查表

| 规则ID | 规则名称 | 严重等级 | 关联commit |
|--------|---------|---------|-----------|
| DOC-01 | 示例代码未经编译/运行验证 | P2 | 501d38ab, 4063f8f9 |
| DOC-02 | 示例代码数据类型与API不匹配 | P1 | 964be4dd |
| DOC-03 | op_list条目遗漏或名称不一致 | P2 | d91e1757a |
| BUILD-01 | CMake target依赖声明缺失 | P2 | 640a1683 |
| BUILD-02 | shell脚本语法/逻辑错误 | P1 | adec83fb |
| BUILD-03 | 新平台binary配置遗漏 | P2 | d6acf37e |
| BUILD-04 | ascendc_config.json重复/冲突配置 | P1 | hotspot_analysis.md |
| LOGIC-01 | !=A \|\| !=B恒真逻辑错误 | P0 | 67c665fd, 692f43a9 |
| LOGIC-02 | 新增类型/分支改变默认路径行为 | P1 | 4d315436 |
| LOGIC-03 | 尾块/边界条件处理遗漏 | P1 | 多条 |
| INT-01 | shape维度乘法int32溢出 | P0 | 8f6ccaea, acadb4c7 |
| INT-02 | GM偏移量在小类型域计算溢出 | P0 | 50df91e7 |
| INT-03 | 有符号/无符号混比与类型不一致 | P2 | 941d3c7f, 0253078d |
| NULL-01 | OP_CHECK宏缺少return语句 | P0 | 7b4a1b53 |
| NULL-02 | 结构体指针成员未初始化 | P0 | 15e40a48 |
| NULL-03 | PipeBarrier与FreeTensor时序颠倒 | P0 | cf84e222 |
| NULL-04 | 先解引用后判空(先用后查) | P0 | hotspot aclnn_addmm.cpp |
| WARN-01 | 无符号整数>=0恒真导致无限循环 | P0 | 0e00f88c |
| WARN-02 | 链式比较与运算符优先级错误 | P1 | 50854088 |
| VALID-01 | 接口调用默认参数与实际数据不一致 | P1 | c8ca6bec |
| VALID-02 | 属性索引硬编码顺序依赖 | P1 | 07e77ddd |
| VALID-03 | 变体接口复用通用dtype校验 | P1 | 0bd3fe0b |
| SHAPE-01 | StorageShape/ViewShape混淆 | P1 | 6b52fdcf, 28ab6d70 |
| SHAPE-02 | tiling key参数传递错误与dead code | P1 | 6eee5478 |
| COPY-01 | 函数调用参数重复f(a,a) | P1 | b2e2cada |
| COPY-02 | 矩阵k/n维度索引粘贴错误 | P0 | 1c2de786 |
| COPY-03 | GM偏移量公式维度混淆 | P1 | 910dac63 |
| SYNC-01 | SetScheduleMode缺失 | P1 | afb09c78 |
| SYNC-02 | SetFlag/WaitFlag顺序错误 | P0 | ed0bd5a1 |
| SYNC-03 | DataCopy前后缺少流水线同步事件 | P0 | 74d43ef3 |
| HW-01 | DMA/datacopy参数超16位上限 | P0 | 34217db7, cf9ea8c9 |
| HW-02 | 硬件宏条件编译遗漏 | P2 | c1469b68 |
| REV-01 | 搭车提交造成revert附带伤害 | P2 | d91e1757a, 4482238c0 |
| REV-02 | 大规模提交测试不充分 | P2 | 852f21d6b |
| SYS-01 | build.sh系统性脆弱 | P1 | adec83fb, hotspot |
| SYS-02 | matmul模块缺陷密度最高 | P0 | hotspot matmul部分 |
| SYS-03 | 编译warning隐藏真bug | P1 | 0e00f88c, 50854088 |
| SYS-04 | ascendc_config.json配置混乱 | P2 | hotspot_analysis.md |
| SYS-05 | 先用后查反模式在op_api层流行 | P0 | hotspot aclnn_*.cpp |

### P0规则汇总（致命级，需立即关注）

| 规则ID | 规则名称 | 典型commit |
|--------|---------|-----------|
| LOGIC-01 | !=A \|\| !=B恒真逻辑错误 | 67c665fd |
| INT-01 | shape维度乘法int32溢出 | 8f6ccaea |
| INT-02 | GM偏移量在小类型域计算溢出 | 50df91e7 |
| NULL-01 | OP_CHECK宏缺少return语句 | 7b4a1b53 |
| NULL-02 | 结构体指针成员未初始化 | 15e40a48 |
| NULL-03 | PipeBarrier与FreeTensor时序颠倒 | cf84e222 |
| NULL-04 | 先解引用后判空 | hotspot |
| WARN-01 | 无符号>=0恒真无限循环 | 0e00f88c |
| COPY-02 | 矩阵k/n维度索引粘贴错误 | 1c2de786 |
| SYNC-02 | SetFlag/WaitFlag顺序错误 | ed0bd5a1 |
| SYNC-03 | DataCopy前后缺少同步事件 | 74d43ef3 |
| HW-01 | DMA/datacopy参数超16位上限 | 34217db7 |
| SYS-02 | matmul模块缺陷密度最高 | hotspot |
| SYS-05 | 先用后查反模式 | hotspot |

### 数据来源

- 仓库: ~/repo/cann/ops-nn/
- 提交总数: 1474（全部非merge commit）
- 缺陷提交数: 380（占比25.8%，经二次过滤排除纯文档/清理提交）
- 实际代码缺陷: ~321（排除纯文档/UT/清理约59条）
- Revert事件: 4个独立事件（5条revert提交）
- 热点确认bug: P0 x 8, P1 x 8, P2 x 8
- 分析周期: 全量git历史(2025-09 ~ 2026-03)
- 中间产物: defect_analysis.md, revert_analysis.md, hotspot_analysis.md, pattern_summary.md, defect_stats.md
