## ops-nn项目缺陷模式与审查规则

数据来源: ops-nn(380条缺陷中提炼42条规则) + ops-nn-dev(612条缺陷中提炼46条规则)
合并后: 去重整合为统一规则集

### 严重等级定义

| 等级 | 含义 | 影响范围                                 |
|------|------|------------------------------------------|
| P0   | 致命 | CoreDump/数据损坏/内存越界/死锁/无限循环 |
| P1   | 严重 | 特定配置崩溃/功能不可用/静默精度错误     |
| P2   | 一般 | 边界条件异常/构建失败/测试不通过         |
| P3   | 建议 | 代码质量/可维护性/潜在隐患               |

### 规则总览

| 规则ID     | 类别          | 严重等级 | 规则名                                              | 来源                     |
|------------|---------------|----------|-----------------------------------------------------|--------------------------|
| BUILD-01   | 构建            | P2       | 新增算子打包配置遗漏                                          | [ops-nn-dev]             |
| BUILD-02   | 构建            | P2       | CMake DEPENDENCIES列表不完整                             | [ops-nn][ops-nn-dev]     |
| BUILD-03   | 构建            | P1       | shell脚本语法/逻辑错误                                      | [ops-nn][ops-nn-dev]     |
| BUILD-04   | 构建            | P2       | 新平台binary配置遗漏                                       | [ops-nn][ops-nn-dev]     |
| BUILD-05   | 构建            | P1       | ascendc_config.json重复/冲突配置                          | [ops-nn]                 |
| BOUND-01   | 边界条件          | P1       | 空tensor输入未正确处理                                      | [ops-nn-dev]             |
| BOUND-02   | 边界条件          | P0       | 多核任务分配未区分tail core和idle core                        | [ops-nn-dev]             |
| BOUND-03   | 边界条件          | P1       | 维度扩展后判断条件未适配                                        | [ops-nn-dev]             |
| BOUND-04   | 边界条件          | P0       | !=A || !=B恒真逻辑错误                                    | [ops-nn]                 |
| BOUND-05   | 边界条件          | P1       | 尾块/边界条件处理遗漏                                         | [ops-nn]                 |
| TILE-01    | Tiling/Buffer | P0       | tiling侧buffer计算与kernel侧InitBuffer不一致                | [ops-nn-dev]             |
| TILE-02    | Tiling/Buffer | P1       | workspace计算公式中错误应用双buffer系数                         | [ops-nn-dev]             |
| TILE-03    | Tiling/Buffer | P0       | tiling参数未区分normal核和tail核                            | [ops-nn-dev]             |
| TILE-04    | Tiling/Buffer | P0       | TilingData结构体host-kernel不一致                         | [ops-nn-dev]             |
| TILE-05    | Tiling/Buffer | P1       | StorageShape/ViewShape混淆                            | [ops-nn]                 |
| TILE-06    | Tiling/Buffer | P1       | tiling key参数传递错误与dead code                          | [ops-nn]                 |
| API-01     | 接口            | P1       | 修改Init签名/模板参数后调用点未全量同步                              | [ops-nn-dev]             |
| API-02     | 接口            | P1       | TilingParse注册类型与实际CompileInfo类型不一致                  | [ops-nn-dev]             |
| API-03     | 接口            | P1       | 接口调用默认参数与实际数据不一致                                    | [ops-nn]                 |
| API-04     | 接口            | P1       | 属性索引硬编码顺序依赖                                         | [ops-nn]                 |
| API-05     | 接口            | P1       | 变体接口复用通用dtype校验                                     | [ops-nn]                 |
| CALC-01    | 计算逻辑          | P1       | CeilDiv/CeilAlign结果缺少上界clamp                        | [ops-nn-dev]             |
| CALC-02    | 计算逻辑          | P0       | 多核场景全局内存初始化存在竞争                                     | [ops-nn-dev]             |
| CALC-03    | 计算逻辑          | P1       | splitK场景Bias加载时机错误                                  | [ops-nn-dev]             |
| WARN-01    | 编译告警          | P2       | 删除"未使用变量"前检查是否原本应该被使用                               | [ops-nn-dev]             |
| WARN-02    | 编译告警          | P1       | 链式比较语义陷阱                                            | [ops-nn][ops-nn-dev]     |
| WARN-03    | 编译告警          | P0       | 无符号整数>=0恒真导致无限循环                                    | [ops-nn]                 |
| COND-01    | 条件逻辑          | P0       | 设置特殊状态后缺少return                                     | [ops-nn-dev]             |
| COND-02    | 条件逻辑          | P1       | 除法运算的除数缺少非零保护                                       | [ops-nn-dev]             |
| COND-03    | 条件逻辑          | P2       | 宏定义中隐藏return语句                                      | [ops-nn-dev]             |
| COND-04    | 条件逻辑          | P1       | 新增类型/分支改变默认路径行为                                     | [ops-nn]                 |
| PLAT-01    | 平台适配          | P1       | `__CCE_AICORE__ == xxx`精确匹配应改为范围比较                  | [ops-nn-dev]             |
| PLAT-02    | 平台适配          | P1       | compute_units列表缺少目标平台                               | [ops-nn-dev]             |
| PLAT-03    | 平台适配          | P0       | DMA/datacopy参数超16位上限                                | [ops-nn]                 |
| PLAT-04    | 平台适配          | P2       | 硬件宏条件编译遗漏                                           | [ops-nn]                 |
| COPY-01    | 复制粘贴          | P2       | 错误日志引用的变量与判断条件不一致                                   | [ops-nn-dev]             |
| COPY-02    | 复制粘贴          | P1       | 函数调用中同类型参数重复                                        | [ops-nn][ops-nn-dev]     |
| COPY-03    | 复制粘贴          | P0       | 矩阵k/n维度索引粘贴错误                                       | [ops-nn]                 |
| COPY-04    | 复制粘贴          | P1       | GM偏移量公式维度混淆                                         | [ops-nn]                 |
| CONF-01    | 配置            | P1       | 平台特定编译选项遗漏                                          | [ops-nn-dev]             |
| CONF-02    | 配置            | P1       | output paramType(REQUIRED/OPTIONAL)配置错误             | [ops-nn-dev]             |
| INT-01     | 整数溢出          | P0       | shape维度相乘强制使用int64_t                                | [ops-nn][ops-nn-dev]     |
| INT-02     | 整数溢出          | P0       | GM偏移量在小类型域计算溢出                                      | [ops-nn]                 |
| INT-03     | 整数溢出          | P0       | 硬件指令参数位宽限制校验缺失                                      | [ops-nn-dev]             |
| INT-04     | 整数溢出          | P0       | uint64_t减法下溢                                        | [ops-nn-dev]             |
| INT-05     | 整数溢出          | P2       | 有符号/无符号混比与类型不一致                                     | [ops-nn]                 |
| SYNC-01    | 流水线同步         | P0       | SetFlag/WaitFlag必须成对出现且无条件执行                        | [ops-nn-dev]             |
| SYNC-02    | 流水线同步         | P0       | CopyOut/CopyIn前的同步事件与实际DMA通道匹配                      | [ops-nn-dev]             |
| SYNC-03    | 流水线同步         | P1       | 连续matmul操作间必须插入PipeBarrier                          | [ops-nn-dev]             |
| SYNC-04    | 流水线同步         | P1       | SetScheduleMode缺失                                   | [ops-nn]                 |
| SYNC-05    | 流水线同步         | P0       | SetFlag/WaitFlag顺序错误                                | [ops-nn]                 |
| SYNC-06    | 流水线同步         | P0       | DataCopy前后缺少流水线同步事件                                 | [ops-nn]                 |
| LOG-01     | 日志            | P3       | 错误码区分PARAM_*和INNER_*                                | [ops-nn-dev]             |
| DTYPE-01   | 数据类型          | P1       | 新增数据类型支持时三处同步更新                                     | [ops-nn-dev]             |
| RES-01     | 资源管理          | P0       | 禁止在op_host/*.cpp中使用非const全局容器                       | [ops-nn-dev]             |
| RES-02     | 资源管理          | P1       | L1全载优化在循环中的状态失效检查                                   | [ops-nn-dev]             |
| RES-03     | 资源管理          | P0       | PipeBarrier与FreeTensor时序颠倒                          | [ops-nn]                 |
| RES-04     | 资源管理          | P0       | 结构体指针成员未初始化                                         | [ops-nn]                 |
| UT-01      | 单元测试          | P3       | 禁止注释EXPECT_*/ASSERT_*断言                             | [ops-nn-dev]             |
| NULL-01    | 空指针           | P0       | 可选输出tensor(outputMask控制)访问前必须检查null                 | [ops-nn-dev]             |
| NULL-02    | 空指针           | P0       | OP_CHECK宏缺少return语句                                 | [ops-nn]                 |
| NULL-03    | 空指针           | P0       | 先解引用后判空(先用后查)                                       | [ops-nn]                 |
| REV-01     | 回退            | P2       | 大规模重构应分步提交                                          | [ops-nn-dev]             |
| REV-02     | 回退            | P1       | 移除binary.json算子变体前确认无下游调用                           | [ops-nn-dev]             |
| REV-03     | 回退            | P2       | 搭车提交造成revert附带伤害                                    | [ops-nn]                 |
| REV-04     | 回退            | P2       | 大规模提交测试不充分                                          | [ops-nn]                 |
| DOC-01     | 文档            | P2       | 示例代码未经编译/运行验证                                       | [ops-nn]                 |
| DOC-02     | 文档            | P1       | 示例代码数据类型与API不匹配                                     | [ops-nn]                 |
| DOC-03     | 文档            | P2       | op_list条目遗漏或名称不一致                                   | [ops-nn]                 |
| SYS-01     | 系统性           | P1       | build.sh系统性脆弱                                       | [ops-nn]                 |
| SYS-02     | 系统性           | P0       | matmul模块缺陷密度最高                                      | [ops-nn][ops-nn-dev]     |
| SYS-03     | 系统性           | P1       | 编译warning隐藏真bug                                     | [ops-nn][ops-nn-dev]     |
| SYS-04     | 系统性           | P2       | 提交粒度过大导致质量失控                                        | [ops-nn][ops-nn-dev]     |
| SYS-05     | 系统性           | P2       | 公共基础设施耦合在算子PR中                                      | [ops-nn][ops-nn-dev]     |
| SYS-06     | 系统性           | P0       | 先用后查反模式在op_api层流行                                   | [ops-nn]                 |
| SYS-07     | 系统性           | P2       | ascendc_config.json配置混乱                             | [ops-nn]                 |
| SYS-08     | 系统性           | -        | foreach模块高触及但低风险                                    | [ops-nn][ops-nn-dev]     |

---

### 类别: 构建 (BUILD)

最高频类别。dev分支占比(14.3%)远高于主仓(~10.9%), 反映开发分支构建基础设施更不稳定。多为机械性错误, 可通过CI自动化大幅减少。子类型: CMake依赖/链接声明缺失(~28)、include路径错误(~12)、打包/安装配置遗漏(~10)、编译选项缺失(~8)、构建脚本逻辑错误(~16)、binary.json平台配置遗漏(~8)、文件命名/目录结构错误(~5)、ascendc_config.json重复/冲突(~5)。

#### BUILD-01: 新增算子打包配置遗漏 [ops-nn-dev]

严重等级: P2

缺陷描述: 新增算子未在ascendc_config.json中注册, 导致kernel二进制不被打包, 运行时找不到预编译kernel触发JIT异常或直接失败。

典型代码示例:

```json
// 缺陷 -- c00e940bfa [dev]
// HardSwishGradV2算子目录和代码都已就绪, 但ascendc_config.json中缺少注册条目
// 运行时: kernel binary not found

// 修复 -- 在ascendc_config.json中添加算子条目
{
  "op_name": "HardSwishGradV2",
  "compute_units": ["ascend910b", "ascend910_93", "ascend910_95"],
  ...
}
```

审查检查方法:
- 新增算子PR必须包含ascendc_config.json/binary.json条目变更
- CI脚本可自动交叉校验算子目录与配置文件

关联commit: `c00e940bfa`, `0d47b59fdd`

---

#### BUILD-02: CMake DEPENDENCIES列表不完整 [ops-nn][ops-nn-dev]

严重等级: P2

缺陷描述: CMakeLists.txt中target的DEPENDENCIES未包含前置依赖, 导致并行构建时序不确定, 表现为本地编译通过但CI偶发失败。

典型代码示例:

```cmake
# 缺陷 -- 640a1683 [主仓]
# ascendc_impl_gen的DEPENDS未包含前置target(ops_info_gen_*)
add_custom_target(ascendc_impl_gen
    DEPENDS ${impl_gen_outputs}  # 缺少对ops_info_gen_*的依赖
)

# 修复
add_custom_target(ascendc_impl_gen
    DEPENDS ${impl_gen_outputs} ops_info_gen_${compute_unit}
)
```

```cmake
# 缺陷 -- fb3b8365b9 [dev]
# op_api UT编译缺少OPAPI_INCLUDE路径
target_include_directories(test_xxx PRIVATE ${SOME_PATH})
# 缺少: ${OPAPI_INCLUDE} 导致头文件找不到

# 修复
target_include_directories(test_xxx PRIVATE ${SOME_PATH} ${OPAPI_INCLUDE})
```

热点补充(阶段4 P0确认, 主仓):

```cmake
# gen_ops_info.cmake:654 -- 循环后遗留变量作为target名
# foreach循环结束后调用compile_from_config
compile_from_config(TARGET ascendc_bin_${compute_unit}_${op_name} ...)
# ${op_name}仅为最后一次迭代的值, 非预期target
```

审查检查方法:
- CMake custom_command/custom_target的DEPENDS必须包含所有输入依赖
- foreach循环结束后不应使用循环变量(取值为最后一次迭代的值)
- 间歇性构建失败首先排查DEPENDS是否完整
- 对比同族算子的CMake配置确认无遗漏

关联commit: `640a1683`, `fb3b8365b9`, `5f3a97918e`

---

#### BUILD-03: shell脚本语法/逻辑错误 [ops-nn][ops-nn-dev]

严重等级: P1

缺陷描述: shell脚本中`[`/`]`前后缺空格导致条件判断永远false/true、数组变量未正确展开、选项flag赋值错误。此类bug在本地运行时可能被掩盖, 在CI中才暴露。

典型代码示例:

```bash
# 缺陷 -- adec83fb [主仓] (check_example.sh)
[ $status -ne 0]
# ']'与'0'粘连, test命令寻找名为"0]"的值, 条件永远false
# 编译失败时CI仍显示成功

# 修复
[ $status -ne 0 ]
```

热点补充(阶段4 P0确认, 主仓):

```bash
# build.sh:1060-1065 -- 数组备份只取首元素
local TMP_BUILD_LIBS=${BUILD_LIBS}     # ${BUILD_LIBS}只取数组第一个元素
# 恢复时
BUILD_LIBS=${TMP_BUILD_LIBS}           # 变成字符串, 多target构建数据丢失

# build.sh:796 -- 启用选项却赋值FALSE
mssanitizer) ENABLE_MSSANITIZER=FALSE ;;  # 应为TRUE

# build.sh:779 -- sed空操作(替换前后相同)
COMPUTE_UNIT=$(echo "$COMPUTE_UNIT" | sed 's/ascend950/ascend950/g')
```

```bash
# 缺陷 -- 9add7e2316 [dev]
# build.sh中某步骤失败后继续执行后续步骤
cp $SRC $DST       # SRC不存在时cp失败但脚本继续
make -j$(nproc)    # 使用了过期文件

# 修复
set -e
set -o pipefail
[ -f "$SRC" ] && cp "$SRC" "$DST" || { echo "ERROR: $SRC not found"; exit 1; }
```

审查检查方法:
- CI脚本必须通过shellcheck静态分析
- `[`和`]`前后必须有空格
- 数组变量使用`"${arr[@]}"`展开, 而非`${arr}`
- 赋值语句的值与意图一致(启用=TRUE, 禁用=FALSE)
- 构建脚本首行是否有`set -e`/`set -o pipefail`

关联commit: `adec83fb`, `9add7e2316`

---

#### BUILD-04: 新平台binary配置遗漏 [ops-nn][ops-nn-dev]

严重等级: P2

缺陷描述: 新增算子或新增硬件平台时, 遗漏目标平台的binary.json配置文件, 导致该平台完全无法使用该算子。编译阶段不报错, 运行时找不到kernel。

典型代码示例:

```
# 缺陷 -- d6acf37e [主仓]
# unique_consecutive算子缺少ascend950的binary.json
# 该平台上算子完全不可用

# 修复
# 新增 index/unique_consecutive/op_host/config/ascend950/binary.json
```

审查检查方法:
- 新增算子时检查是否为所有目标平台提供了binary.json
- 新增平台支持时检查现有算子是否需要同步添加binary配置
- binary.json中的compute_units列表应与实际支持的平台一致

关联commit: `d6acf37e`

---

#### BUILD-05: ascendc_config.json重复/冲突配置 [ops-nn]

严重等级: P1

缺陷描述: ascendc_config.json中存在完全重复的条目或同名但配置内容冲突的条目(如compute_units列表不同)。哪一条生效取决于JSON解析器实现, 构建行为不确定。

典型代码示例:

```json
// 缺陷 -- 完全重复条目 [主仓]
// AdaptiveAvgPool3d在行2和行5各有一份完全相同的配置
// LayerNormQuant在行571/600/601有三份

// 同名但冲突的配置
// AdaptiveAvgPool3dGrad: 一份compute_units含ascend950, 另一份不含
// STFT: 一份含ascend910_93, 另一份不含

// 无效配置
// compile_options中引用"ascend910_95"(行283/363)
// 不在compute_units列表中且非已知平台标识
```

审查检查方法:
- 修改ascendc_config.json时检查同名算子是否已有条目
- 同名算子的配置(compute_units、compile_options等)必须一致
- 平台标识符应与已知平台列表匹配, 避免拼写错误

关联commit: 热点分析(hotspot_analysis.md)

---

### 类别二: BOUNDARY 边界条件缺失(~62条)

高频且严重度高。算子对非典型输入(空tensor、小shape、大shape、退化维度)的处理普遍不充分。子类型: 空tensor处理缺失(~15)、尾块/尾核处理不完整(~11)、大shape越界(~8)、退化维度(~7)、输入校验遗漏(~5)、小shape(~4)、format变体(~4)、off-by-one(~5)、除零防护(~3)。

---

### 类别: 边界条件 (BOUND)

高频且严重度高。算子对非典型输入(空tensor、小shape、大shape、退化维度)的处理普遍不充分。子类型: 空tensor处理缺失(~15)、尾块/尾核处理不完整(~11)、大shape越界(~8)、退化维度(~7)、输入校验遗漏(~5)、小shape(~4)、format变体(~4)、off-by-one(~5)、除零防护(~3)。

#### BOUND-01: 空tensor输入未正确处理 [ops-nn-dev]

严重等级: P1

缺陷描述: 空tensor输入时算子未正确early return并设置空输出shape, 导致后续计算对零长度维度执行非预期操作。输入空和输出空语义不同, 不能用`||`合并处理。

典型代码示例:

```cpp
// 缺陷 -- aclnn_batch_matmul.cpp [dev] (hotspot, 9次触及)
auto graph = CreateBatchmmEmptyTensorGraph(...);  // 空tensor分支
graph = CreateBatchMatmulExecBmmOpGraph(...);      // 无条件覆盖! 死代码

// 修复
if (IsEmptyTensor(...)) {
    graph = CreateBatchmmEmptyTensorGraph(...);
} else {
    graph = CreateBatchMatmulExecBmmOpGraph(...);
}
```

审查检查方法:
- 空tensor判断后是否有return或else保护后续逻辑
- 输入空和输出空是否分别处理(不要用`||`合并)
- 空tensor分支是否正确设置输出shape

关联commit: hotspot `aclnn_batch_matmul.cpp`, `95c2fbb6bf`

---

#### BOUND-02: 多核任务分配未区分tail core和idle core [ops-nn-dev]

严重等级: P0

缺陷描述: group卷积/分核场景下, `>=`判断将合法尾部core和超范围idle core合并处理, 导致超范围core使用错误数据量计算, 产生精度问题或越界访问。

典型代码示例:

```cpp
// 缺陷 -- e2fc1fecb1 [dev]
if (coreIdx >= normalCoreNum) {
    ProcessData(tailDataSize);  // idle core不应执行此操作
}

// 修复 -- 拆分为两个分支
if (coreIdx == normalCoreNum) {
    ProcessData(tailDataSize);   // tail core正常处理
} else if (coreIdx > normalCoreNum) {
    return;                       // idle core直接返回
}
```

审查检查方法:
- 多核任务分配中`>=`条件是否应拆分为`==`(tail)和`>`(idle)
- idle core是否有明确的early return

关联commit: `e2fc1fecb1`

---

#### BOUND-03: 维度扩展后判断条件未适配 [ops-nn-dev]

严重等级: P1

缺陷描述: 2D场景tensor扩展为5D(D=1)后, CheckOutputAllZero要求所有维度为0才推断输出shape, D=1导致返回false, infershape流程失败。

典型代码示例:

```cpp
// 缺陷 -- 08b9ebb112 [dev]
bool CheckOutputAllZero(const Shape& shape) {
    for (int i = 0; i < shape.GetDimNum(); i++) {
        if (shape.GetDim(i) != 0) return false;  // D=1时返回false
    }
    return true;
}

// 修复 -- 新增处理扩展维度的变体
bool CheckOutputAllZeroFrom2D(const Shape& shape) {
    // 跳过扩展维度(D维度), 只检查原始有效维度
}
```

审查检查方法:
- 同时支持2D和3D且通过维度扩展统一处理时, 所有依赖维度值的判断需考虑扩展维度默认值
- 输入索引用命名常量不硬编码

关联commit: `08b9ebb112`

---

#### BOUND-04: !=A || !=B恒真逻辑错误 [ops-nn]

严重等级: P0

缺陷描述: `x != A || x != B`(当A != B时)恒为true, 属于De Morgan律的经典误用。在ops-nn主仓中多次独立出现, 导致合法输入被拒绝或错误代码路径被选中。

典型代码示例:

```cpp
// 缺陷 -- 67c665fd [主仓] (avg_pool_v2_grad)
if (inputDimNum != CHW_DIMS || inputDimNum != NCHW_DIMS) {
    return GRAPH_PARAM_INVALID;  // 恒为true, 所有输入被拒绝
}
// 修复
if (inputDimNum != CHW_DIMS && inputDimNum != NCHW_DIMS) {
    return GRAPH_PARAM_INVALID;
}
```

```cpp
// 缺陷 -- 692f43a9 [主仓] (adaptive_pool3d_tiling.cpp)
if (npuArch != NpuArch::DAV_3510 || npuArch != NpuArch::DAV_5102) {
    return GRAPH_PARAM_INVALID;  // 恒为true, 所有平台返回错误
}
// 修复 -- 删除第二个条件
if (npuArch != NpuArch::DAV_3510) {
    return GRAPH_PARAM_INVALID;
}
```

审查检查方法:
- `x != A || x != B`是经典恒真逻辑错误, 应作为clang-tidy静态检查规则
- 条件表达式中多个!=用||连接时99%应该用&&

关联commit: `67c665fd`, `692f43a9`

---

#### BOUND-05: 尾块/边界条件处理遗漏 [ops-nn]

严重等级: P1

缺陷描述: 数据不能被tile/block整除时尾块处理遗漏、除零防护缺失、动态shape维度为0时的特殊处理遗漏。

典型代码示例:

```cpp
// 缺陷(通用模式) [主仓]
uint32_t tileCount = totalSize / tileSize;  // 不能整除时尾块丢失
for (uint32_t i = 0; i < tileCount; i++) { ... }

// 修复
uint32_t tileCount = CeilDiv(totalSize, tileSize);
uint32_t lastTileSize = totalSize - (tileCount - 1) * tileSize;
```

审查检查方法:
- tile/分块循环中最后一块的大小是否与其他块不同
- 所有除法的除数是否可能为零
- 动态shape场景中维度值可能为0时是否有提前拦截

关联commit: 多条(主仓defect_analysis.md)

---

### 类别三: TILING_BUFFER Tiling/Buffer/Workspace计算(~60条)

核心运行时缺陷, 直接导致OOM、精度错误、性能劣化。子类型: workspace大小计算公式错误(~14)、tiling参数遗漏/计算不一致(~8)、UB空间分配不足(~7)、buffer数量/double buffer策略不一致(~5)、tiling key选择错误(~8)、对齐粒度错误(~4)、L1内存预留不足(~3)、tiling结构体不一致(~2)、StorageShape/ViewShape混淆(~4)、dead code(~2)。

---

### 类别: Tiling/Buffer (TILE)

核心运行时缺陷, 直接导致OOM、精度错误、性能劣化。子类型: workspace大小计算公式错误(~14)、tiling参数遗漏/计算不一致(~8)、UB空间分配不足(~7)、buffer数量/double buffer策略不一致(~5)、tiling key选择错误(~8)、对齐粒度错误(~4)、L1内存预留不足(~3)、tiling结构体不一致(~2)、StorageShape/ViewShape混淆(~4)、dead code(~2)。

#### TILE-01: tiling侧buffer计算与kernel侧InitBuffer不一致 [ops-nn-dev]

严重等级: P0

缺陷描述: tiling侧用公式估算buffer大小, 但kernel侧实际buffer分配策略不同(buffer数量、double buffer策略), 导致UB溢出或数据覆盖。

典型代码示例:

```cpp
// 缺陷 -- e25e6e60c7 [dev]
// tiling侧公式
bufferSize = ubSize_ / DOUBLE_BUFFER / bufferCount;
// 假设所有buffer均匀分配

// 但kernel侧实际: x和y各占2份buffer(double buffer),
// scale/shift又有不同策略(NO_DOUBLE_BUFFER)
// 两端计算逻辑不一致, 导致tiling算出的buffer大小 > kernel实际可用

// 修复 -- 统一tiling侧公式
bufferSize = ubSize_ / (X_Y_BUFFER_NUM * DOUBLE_BUFFER + 1 + hasScaleShift);
```

审查检查方法:
- tiling侧buffer计算必须与kernel侧InitBuffer做交叉比对
- 检查buffer数量和double buffer策略是否两端一致
- 建议用共享常量定义buffer数量

关联commit: `e25e6e60c7`

---

#### TILE-02: workspace计算公式中错误应用双buffer系数 [ops-nn-dev]

严重等级: P1

缺陷描述: workspace大小计算时对每项无差别乘BUFFER_NUM, 但部分buffer实际只需单份空间。

典型代码示例:

```cpp
// 缺陷 -- e32a5eaae3 [dev]
userWorkspaceByteSize += BT * H * IN_BYTE_SIZE * BUFFER_NUM;  // 只需单份
userWorkspaceByteSize += V * H * IN_BYTE_SIZE * BUFFER_NUM;   // 只需单份

// 修复 -- 去掉不需要双buffer的项的BUFFER_NUM
userWorkspaceByteSize += BT * H * IN_BYTE_SIZE;
userWorkspaceByteSize += V * H * IN_BYTE_SIZE;
```

审查检查方法:
- workspace计算中每项乘BUFFER_NUM时逐项注释说明为什么需要双buffer
- 代码审查时逐行核对公式各项物理含义

关联commit: `e32a5eaae3`

---

#### TILE-03: tiling参数未区分normal核和tail核 [ops-nn-dev]

严重等级: P0

缺陷描述: 多核tiling中normal核和tail核使用相同参数, 但tail核的数据量通常小于normal核。用normal核的参数分配tail核的workspace可能不足, 导致OOM。

典型代码示例:

```cpp
// 缺陷 -- MaxPoolGradWithArgmaxV3 [dev] (参考b99dcb02c5类)
void CalcGradArgmaxInner(int highAxisInner) {
    bufferSize = highAxisInner * elementSize;  // tail核highAxisTail可能远小
}

// 修复 -- 拆分为normal版和tail版
void CalcGradArgmaxInner(int highAxisInner) { ... }     // normal核
void CalcGradArgmaxInnerTail(int highAxisTail) { ... }   // tail核独立计算
```

审查检查方法:
- 多核tiling中normal核和tail核的buffer尺寸是否分别计算
- tiling key是否区分normal/tail模式

关联commit: `90649ac299`

---

#### TILE-04: TilingData结构体host-kernel不一致 [ops-nn-dev]

严重等级: P0

缺陷描述: TilingData结构体的字段布局、类型在host和kernel端不完全一致, 导致数据解析错位。

典型代码示例:

```cpp
// 缺陷 -- 0d32207040 [dev]
// Host端
struct TilingData {
    int64_t scale;   // 应为float, int64_t导致精度丢失
    int64_t value;   // 冗余字段, kernel端不使用
};
// Kernel端期望
struct TilingData {
    float scale;     // 类型不匹配!
};
// 解析时字段偏移错位
```

审查检查方法:
- TilingData结构体定义在host和kernel端是否完全一致(字段数、字段类型、字段顺序)
- 建议使用共享头文件定义TilingData

关联commit: `0d32207040`, revert `74cdae15d`

---

#### TILE-05: StorageShape/ViewShape混淆 [ops-nn]

严重等级: P1

缺陷描述: 非连续张量的StorageShape(物理内存布局)与ViewShape(用户视角的逻辑shape)不同。shape校验使用GetStorageShape(), 但非连续张量的StorageShape可能大于ViewShape。

典型代码示例:

```cpp
// 缺陷 -- 6b52fdcf [主仓] (LSTM)
auto shape = input.GetStorageShape();  // 非连续张量StorageShape != ViewShape
if (shape.GetDim(0) != expectedDim) { return INVALID; }

// 修复
auto shape = input.GetViewShape();    // 优先使用逻辑shape
```

```cpp
// 缺陷 -- 28ab6d70 [主仓] (thnn_fused_lstm_cell_backward)
auto dim = tensor.GetStorageShape().GetDim(1);  // 经view操作后物理shape与逻辑shape不符

// 修复
auto dim = tensor.GetViewShape().GetDim(1);
```

审查检查方法:
- shape校验优先使用GetViewShape(), 仅在需要物理布局时用GetStorageShape()
- 非连续张量场景(view/transpose/slice后)必须区分StorageShape和ViewShape
- format校验应按张量实际格式区分, 不应统一强制FORMAT_ND

关联commit: `6b52fdcf`, `28ab6d70`

---

#### TILE-06: tiling key参数传递错误与dead code [ops-nn]

严重等级: P1

缺陷描述: tiling key参数传递错误导致kernel选择了错误的计算策略; return语句后的SetAtomicNone等代码永远不执行。

审查检查方法:
- tiling key参数应有枚举约束而非裸字符串/魔法数
- return语句后不应有可执行代码(开启dead code检测)
- tiling key值与kernel侧的switch-case必须一一对应

关联commit: `6eee5478`

---

### 类别四: INTERFACE_API 接口/API变更传播(~62条)

dev分支接口变更频繁, 占比远高于主仓。子类型: API迁移不完整(~12)、接口签名变更未同步(~8)、废弃接口未清理(~6)、命名空间迁移(~5)、API误用(~5)、参数传递错误(~4)、dtype默认参数不一致(~5)、硬编码索引获取属性(~5)、变体接口复用通用校验(~4)。

---

### 类别: 接口 (API)

dev分支接口变更频繁, 占比远高于主仓。子类型: API迁移不完整(~12)、接口签名变更未同步(~8)、废弃接口未清理(~6)、命名空间迁移(~5)、API误用(~5)、参数传递错误(~4)、dtype默认参数不一致(~5)、硬编码索引获取属性(~5)、变体接口复用通用校验(~4)。

#### API-01: 修改Init签名/模板参数后调用点未全量同步 [ops-nn-dev]

严重等级: P1

缺陷描述: 修改kernel类Init签名或模板参数后, 未全局搜索所有调用点同步更新。

典型代码示例:

```cpp
// 缺陷 -- b14f5a03ca [dev]
// GENERAL_OP_IMPL宏调用op.Init()缺少workspace参数
GENERAL_OP_IMPL(op, OpType<T>);
// 宏展开后: op.Init(tilingData);
// 但Init签名已改为: Init(tilingData, workspace);

// 同时: 模板类只传1个类型参数
OpType<float> op;
// 但OpType已改为: template<typename T, KernelMode M> class OpType;
```

审查检查方法:
- 修改Init签名/模板参数后, 全局搜索所有调用点确认同步更新
- 宏展开后的实际调用是否与当前函数签名匹配

关联commit: `b14f5a03ca`, `22979c4afe`

---

#### API-02: TilingParse注册类型与实际CompileInfo类型不一致 [ops-nn-dev]

严重等级: P1

缺陷描述: TilingParse<T>的模板参数T使用旧类型, 但tiling实际依赖新类型, 导致tiling数据解析失败。

典型代码示例:

```cpp
// 缺陷 -- 368607bd [dev]
REGISTER_TILING_PARSE(CTCLossV2GradCompileInfo, ParseFunc);
// 但tiling实际操作的是新类型
TilingPrepare<CTCLossV2GradForCompileInfo>(...);

// 修复
REGISTER_TILING_PARSE(CTCLossV2GradForCompileInfo, ParseFunc);
```

审查检查方法:
- TilingParse<T>的T与TilingPrepare操作的CompileInfo类型是否一致
- 重构后grep旧类型名确认无残留引用

关联commit: `368607bd`

---

#### API-03: 接口调用默认参数与实际数据不一致 [ops-nn]

严重等级: P1

缺陷描述: 调用下层接口时未传入实际数据类型, 使用了接口的默认参数值。当用户传入的数据类型与默认值不同时产生错误结果。

典型代码示例:

```cpp
// 缺陷 -- c8ca6bec [主仓] (AdaptiveMaxPool3d)
MaxPool3D(input, output, indices);  // indicesDtype默认int32
// 但用户可能传int64的indices

// 修复
MaxPool3D(input, output, indices, indicesDtype);  // 显式传入实际类型
```

审查检查方法:
- 调用具有默认参数的接口时, 检查实际数据类型是否可能与默认值不同
- 显式传参优于依赖默认值

关联commit: `c8ca6bec`

---

#### API-04: 属性索引硬编码顺序依赖 [ops-nn]

严重等级: P1

缺陷描述: 通过`GetAttrPointer<T>(index)`硬编码索引获取算子属性, 但不同算子类型的attr排列顺序不同。

典型代码示例:

```cpp
// 缺陷 -- 07e77ddd [主仓] (InplaceAddmm走GemmV3路径)
auto transA = GetAttrPointer<bool>(0);  // 期望读transpose_a
auto transB = GetAttrPointer<bool>(1);  // 期望读transpose_b
// 但GemmV3的attr顺序是: [alpha, beta, transpose_a, transpose_b]
// 索引0/1实际读到了alpha/beta

// 修复
auto transA = GetAttrPointer<bool>(2);
auto transB = GetAttrPointer<bool>(3);
```

审查检查方法:
- 通过索引获取算子属性时, 确认不同算子类型的attr排列顺序是否一致
- 优先使用属性名获取(GetAttrByName)而非索引

关联commit: `07e77ddd`

---

#### API-05: 变体接口复用通用dtype校验 [ops-nn]

严重等级: P1

缺陷描述: 变体接口(如WeightNz版本)复用通用dtype校验函数, 但变体接口有更严格的类型限制。通用校验放行了变体不支持的类型, 运行时crash。同时入参nullptr未拦截。

典型代码示例:

```cpp
// 缺陷 -- 0bd3fe0b [主仓]
bool isValid = CommonDtypeCheck(input);  // 通用函数允许int8
// 但WeightNz接口不支持int8 -> crash
// 同时 input 为 nullptr 时未拦截

// 修复
if (input == nullptr) { return ACLNN_ERR_INNER_NULLPTR; }
bool isValid = WeightNzDtypeCheck(input);  // 独立的变体校验
```

审查检查方法:
- 变体接口应使用独立dtype校验而非复用通用函数
- aclnn函数入口第一步应做所有必选输入的nullptr检查

关联commit: `0bd3fe0b`

---

### 类别五: CALC_LOGIC 计算逻辑错误(~35条)

dev分支独有细分类别。算子核心计算逻辑中的数学/算法错误。

---

### 类别: 计算逻辑 (CALC)

dev分支独有细分类别。算子核心计算逻辑中的数学/算法错误。

#### CALC-01: CeilDiv/CeilAlign结果缺少上界clamp [ops-nn-dev]

严重等级: P1

缺陷描述: tiling参数通过CeilDiv计算后可能超过实际shape维度值, 未做`std::min(..., actualDim)`上界约束。

典型代码示例:

```cpp
// 缺陷 -- a70d8b16d5 [dev]
hoAL1min = CeilDiv(m0, wo);  // 当wo < m0时, 结果可能超过实际ho
hiAL1min = (hoAL1min - 1) * strideH + dilateH * (kh - 1) + 1;
// hiAL1min超出实际输入高度, L1 size校验错误

// 修复
hoAL1min = std::min(CeilDiv(m0, wo), ho);  // 不超过实际ho
```

审查检查方法:
- CeilDiv/CeilAlign结果是否需要与实际shape维度做min(上界clamp)
- 有符号整数做位运算前应校验非负

关联commit: `a70d8b16d5`

---

#### CALC-02: 多核场景全局内存初始化存在竞争 [ops-nn-dev]

严重等级: P0

缺陷描述: 多个核独立对全局内存做零初始化存在竞争条件。

典型代码示例:

```cpp
// 缺陷 -- fa03888b0d [dev]
if (isEmpty) {
    for (int i = 0; i < size; i++) {
        globalMem[i] = 0;  // 多核同时写, 结果不确定
    }
}

// 修复 -- 指定核初始化 + 同步屏障
if (isEmpty && GetBlockIdx() == 0) {
    for (int i = 0; i < size; i++) { globalMem[i] = 0; }
}
SyncAll();
```

审查检查方法:
- 多核场景全局内存写入是否由指定核(core 0)完成并加SyncAll
- 跨循环计数器确认是全局累积还是每次迭代独立

关联commit: `fa03888b0d`

---

#### CALC-03: splitK场景Bias加载时机错误 [ops-nn-dev]

严重等级: P1

缺陷描述: splitK模式下Bias应在第一次K迭代(kIndex==0)加载, 但错误地放在最后一次迭代加载。

典型代码示例:

```cpp
// 缺陷 -- e7048cff4f [dev]
if (kIndex == splitKRound - 1) { LoadBias(); }  // 错误

// 修复
if (kIndex == 0) { LoadBias(); }  // 首次累加时加入Bias
```

审查检查方法:
- splitK场景Bias/残差加载是否在kIndex==0
- 累加顺序是否符合数学定义

关联commit: `e7048cff4f`

---

### 类别六: COMPILE_WARN 编译告警/代码规范(~55条)

部分告警修复中隐含真实逻辑bug。

---

### 类别: 编译告警 (WARN)

部分告警修复中隐含真实逻辑bug。

#### WARN-01: 删除"未使用变量"前检查是否原本应该被使用 [ops-nn-dev]

严重等级: P2

缺陷描述: 删除编译器报告的"未使用变量"时, 该变量可能原本应该被使用, 真正的bug是上游copy-paste后忘记替换变量名。

典型代码示例:

```cpp
// 缺陷 -- d818b4d4 [dev]
// 编译器报告sourceStorageFormat未使用
// 实际bug是条件判断copy-paste错误:
if (maskStorageFormat != FORMAT_ND || maskStorageFormat != FORMAT_ND) {
    //                                ^^^^ 应为sourceStorageFormat
}
// 该commit只删除了变量来消除告警, 但条件中的copy-paste错误并未修正
```

审查检查方法:
- 删除"未使用变量"前检查该变量是否原本应该被使用
- 搜索同一代码块中是否有copy-paste导致的变量名重复

关联commit: `d818b4d4`

---

#### WARN-02: 链式比较语义陷阱 [ops-nn][ops-nn-dev]

严重等级: P1

缺陷描述: C++中`a < b < c`被解析为`(a < b) < c`即`bool < c`, 不等同于数学含义。`a == b == 1`解析为`(a == b) == 1`。

典型代码示例:

```cpp
// 缺陷 -- 7ad3e89e [dev]
if (lower < value < upper) { ... }
// 编译器解析为: (lower < value) < upper
// 当upper > 1时永远为true!

// 修复
if (lower < value && value < upper) { ... }
```

```cpp
// 缺陷 -- 50854088 [主仓] (约14个文件)
if (tensor->GetViewShape().GetDim(firstLastDim) ==
    tensor->GetViewShape().GetDim(secondLastDim) == 1) { ... }
// 解析为 (dim1 == dim2) == 1
```

审查检查方法:
- 链式比较`a < b < c`/`a == b == c`在C++中语义错误
- `&&`和`||`混合使用时必须加括号
- 建议启用 -Wparentheses 编译选项

关联commit: `7ad3e89e`, `50854088`

---

#### WARN-03: 无符号整数>=0恒真导致无限循环 [ops-nn]

严重等级: P0

缺陷描述: `for(uint64_t i = N; i >= 0UL; i--)`中`i >= 0`对无符号整数恒为true。

典型代码示例:

```cpp
// 缺陷 -- 0e00f88c [主仓] (avg_pool3d_grad等5处)
for (uint64_t i = singleCoreWo; i >= 0UL; i--) {
    ProcessTile(i);  // 无限循环
}

// 修复
for (uint64_t i = singleCoreWo + 1; i > 0; i--) {
    ProcessTile(i - 1);
}
```

审查检查方法:
- 无符号整数循环中`>=0`条件是无限循环bug
- 开启`-Wtautological-compare`可自动检测

关联commit: `0e00f88c`

---

### 类别七: CONDITION_LOGIC 条件判断/控制流(~45条)

条件分支遗漏、控制流路径错误、return缺失。

---

### 类别: 条件逻辑 (COND)

条件分支遗漏、控制流路径错误、return缺失。

#### COND-01: 设置特殊状态后缺少return [ops-nn-dev]

严重等级: P0

缺陷描述: SetUnknownRank/SetUnknownShape后没有return语句, 代码继续执行后续shape推导逻辑, GetDimNum可能返回异常值导致越界。

典型代码示例:

```cpp
// 缺陷 -- 2d424df8e6 [dev]
if (condition) {
    outputDesc->SetUnknownRank();
    // 缺少 return ge::GRAPH_SUCCESS;
}
int dimNum = outputDesc->GetShape().GetDimNum();
for (int i = 0; i < dimNum; i++) { ... }  // dimNum可能是垃圾值

// 修复
if (condition) {
    outputDesc->SetUnknownRank();
    return ge::GRAPH_SUCCESS;
}
```

审查检查方法:
- 设置特殊状态(UnknownRank/UnknownShape)的分支必须包含return
- InferShape函数应做静态分析确保所有路径正确返回

关联commit: `2d424df8e6`

---

#### COND-02: 除法运算的除数缺少非零保护 [ops-nn-dev]

严重等级: P1

缺陷描述: 除法运算的除数来自外部参数或计算结果, 未做非零保护。

典型代码示例:

```cpp
// 缺陷 -- 22f84c0e [dev]
result = totalSize / blockDim;  // blockDim可能为0

// 修复
if (blockDim == 0) { return ERROR_INVALID_PARAM; }
result = totalSize / blockDim;
```

审查检查方法:
- 除法运算的除数是否有非零保护
- CeilDiv/整数除法的分母是否可能为0

关联commit: `22f84c0e`

---

#### COND-03: 宏定义中隐藏return语句 [ops-nn-dev]

严重等级: P2

缺陷描述: 宏中使用return语句隐藏控制流, 调用者不知道宏可能提前退出函数。

典型代码示例:

```cpp
// 缺陷 -- 6beab61dd0 [dev]
#define INVOKE_FLAT_QUANT_IMPL(impl) \
    do { auto ret = impl.Process(); return ret; } while(0)

TILING_KEY_IS(3) { INVOKE_FLAT_QUANT_IMPL(implA); }
TILING_KEY_IS(5) { INVOKE_FLAT_QUANT_IMPL(implB); }  // 提前退出
```

审查检查方法:
- 宏定义中是否包含return语句
- 是否有更清晰的替代方案(内联函数、直接展开)

关联commit: `6beab61dd0`

---

#### COND-04: 新增类型/分支改变默认路径行为 [ops-nn]

严重等级: P1

缺陷描述: 在条件判断中新增一个类型分支时, if-else结构的默认路径原本服务于已有类型, 新分支插入改变了条件匹配顺序。

典型代码示例:

```cpp
// 缺陷 -- 4d315436 [主仓] (QuantBatchMatmulV4)
int k0 = (dtype == DT_INT4) ? INT8_K0 : INT4_K0;
// 条件搞反了: int4应该用INT4_K0

// 修复
int k0 = (dtype == DT_INT4) ? INT4_K0 : INT8_K0;
```

审查检查方法:
- 新增类型/分支时, 三元表达式的true分支应对应"新增的特殊情况"
- else(默认分支)必须保持原有行为不变

关联commit: `4d315436`

---

### 类别八: PLATFORM 平台/SoC适配(~39条)

多平台支持(910/910_93/910_95/310P/5102/3510)下的适配遗漏。

---

### 类别: 平台适配 (PLAT)

多平台支持(910/910_93/910_95/310P/5102/3510)下的适配遗漏。

#### PLAT-01: `__CCE_AICORE__ == xxx`精确匹配应改为范围比较 [ops-nn-dev]

严重等级: P1

缺陷描述: 平台版本判断使用精确匹配`==`, 新平台上线时指令集是向下兼容的, 但精确匹配导致新平台不匹配任何分支。

典型代码示例:

```cpp
// 缺陷 -- 4f87bea9ab [dev]
#if __CCE_AICORE__ == 220
    UseAdvancedInstruction();
#else
    UseFallback();
#endif

// 修复
#if __CCE_AICORE__ >= 220
    UseAdvancedInstruction();
#else
    UseFallback();
#endif
```

审查检查方法:
- `__CCE_AICORE__ == xxx`和`__NPU_ARCH__ == xxx`精确匹配是否应改为`>=`
- 新增平台时有checklist检查所有相关算子的条件分支

关联commit: `4f87bea9ab`

---

#### PLAT-02: compute_units列表缺少目标平台 [ops-nn-dev]

严重等级: P1

缺陷描述: ascendc_config.json中算子的compute_units列表缺少某个目标平台, 该平台二进制不被预编译, 运行时触发JIT异常。

典型代码示例:

```json
// 缺陷 -- 15138bbd86 [dev], db31f7273 [dev]
{
    "op_name": "TransQuantParamV2",
    "compute_units": ["ascend910b", "ascend910_93"]
    // 缺少 "ascend310p"
}
```

审查检查方法:
- 对照平台支持矩阵逐一确认compute_units覆盖所有目标平台
- 新增平台时全局搜索ascendc_config.json中遗漏的算子

关联commit: `15138bbd86`, `db31f7273`

---

#### PLAT-03: DMA/datacopy参数超16位上限 [ops-nn]

严重等级: P0

缺陷描述: datacopy/DMA指令的stride/blockLen参数为uint16_t类型, 最大值65535。当cin或数据量超过65535时参数截断。

典型代码示例:

```cpp
// 缺陷 -- 34217db7 [主仓] (Conv3DBackpropInput)
uint16_t stride = cin * sizeof(float);  // cin > 65535时截断

// 修复 -- tiling层校验并降级
if (cin > MAX_DATACOPY_STRIDE) {
    useFallbackPath = true;
}
```

审查检查方法:
- datacopy/DMA stride参数使用前校验不超过65535(16位上限)
- 大数据量场景应使用Ext版本API或分块处理

关联commit: `34217db7`, `cf9ea8c9`

---

#### PLAT-04: 硬件宏条件编译遗漏 [ops-nn]

严重等级: P2

缺陷描述: 特定芯片专有指令(如Fixpipe)未用`__NPU_ARCH__`等硬件宏条件编译保护, 在不支持该指令的芯片上编译/运行失败。

典型代码示例:

```cpp
// 缺陷 -- c1469b68 [主仓]
FixPipe(dst, src, params);  // 某些芯片不支持

// 修复
#if defined(__NPU_ARCH__) && (__NPU_ARCH__ >= 910)
    FixPipe(dst, src, params);
#else
    // fallback路径
#endif
```

审查检查方法:
- 特定芯片指令需用`__NPU_ARCH__`等硬件宏条件编译保护
- 新芯片适配时应有端云兼容性检查清单

关联commit: `c1469b68`

---

### 类别九: COPY_PASTE 复制粘贴错误(~40条)

高度可审查的机械性错误。子类型: 变量名混淆(~13)、日志引用错误变量(~5)、参数位置重复(~7)、拼写错误(~4)、示例代码错误(~3)、维度索引copy-paste(~4)、偏移量公式套错维度(~3)。

---

### 类别: 复制粘贴 (COPY)

高度可审查的机械性错误。子类型: 变量名混淆(~13)、日志引用错误变量(~5)、参数位置重复(~7)、拼写错误(~4)、示例代码错误(~3)、维度索引copy-paste(~4)、偏移量公式套错维度(~3)。

#### COPY-01: 错误日志引用的变量与判断条件不一致 [ops-nn-dev]

严重等级: P2

缺陷描述: CHECK/LOG调用中, 报错信息引用的变量与实际校验的变量不一致。

典型代码示例:

```cpp
// 缺陷 -- d1832c87c9 [dev]
if (scaleShape->GetStorageShape().GetDim(0) != expectedDim) {
    LOG("dim mismatch: %d", offsetShape->GetStorageShape().GetDim(0));
    //                      ^^^^^^^^^^^^ 应为scaleShape
}
```

审查检查方法:
- 错误日志中引用的变量是否与上文条件判断中的变量一致
- 连续相似CHECK/LOG调用中, 逐行对比检查对象

关联commit: `d1832c87c9`

---

#### COPY-02: 函数调用中同类型参数重复 [ops-nn][ops-nn-dev]

严重等级: P1

缺陷描述: 函数调用中两个参数使用了同一个变量, 应使用不同变量。

典型代码示例:

```cpp
// 缺陷 -- dec58d1b24 [dev], b2e2cada [主仓]
bool isEmpty = isAddBmmProcessEmptyTensor(batch1, batch1);
// 第二参数应为batch2

// 修复
bool isEmpty = isAddBmmProcessEmptyTensor(batch1, batch2);
```

审查检查方法:
- 函数调用中两个参数完全相同`f(a, a)`时必须确认是否为copy-paste错误
- 尤其关注名称只差一个字符的参数(batch1/batch2, self/other)

关联commit: `dec58d1b24`, `b2e2cada`

---

#### COPY-03: 矩阵k/n维度索引粘贴错误 [ops-nn]

严重等级: P0

缺陷描述: 矩阵维度索引的k和n在转置/非转置时必须互补。copy-paste后k维度的公式和n维度完全相同。

典型代码示例:

```cpp
// 缺陷 -- 1c2de786 [主仓] (matmul_common_infershape.cpp)
int64_t k_x2_dim = transB ? x2Shape.GetDim(dimNum - 1) : x2Shape.GetDim(dimNum - 2);
int64_t n_x2_dim = transB ? x2Shape.GetDim(dimNum - 1) : x2Shape.GetDim(dimNum - 2);
// n_x2_dim公式与k_x2_dim完全相同!

// 修复
int64_t n_x2_dim = transB ? x2Shape.GetDim(dimNum - 2) : x2Shape.GetDim(dimNum - 1);
```

热点补充(P0确认):
```cpp
// mat_mul_deterministic_splitk_kernel.h:404-405 [主仓]
uint64_t rowBlockNum = alignedN / 16;  // 应为alignedM / 16?
uint64_t colBlockNum = alignedN / 16;  // 两行公式完全相同
```

审查检查方法:
- 矩阵k/n维度索引在转置/非转置情况下必须互补
- 相邻行的赋值公式完全相同时高度疑似copy-paste错误
- row/col相关变量应使用不同的维度(M vs N)

关联commit: `1c2de786`

---

#### COPY-04: GM偏移量公式维度混淆 [ops-nn]

严重等级: P1

缺陷描述: 多维偏移量计算将不同维度索引合并后统一乘同一步长, 公式语义错误。

典型代码示例:

```cpp
// 缺陷 -- 910dac63 [主仓] (QuantUpdateScatter)
gmVarOffset_ = (batchIdx * outerDim + axisOffset) * stride;
// axisOffset和outerDim的步长不同, 不能合并乘

// 修复
gmVarOffset_ = batchIdx * outerDim * varDim3 + axisOffset * varDim3;
```

审查检查方法:
- 多维偏移量应逐维展开(各维索引 x 对应步长), 避免合并表达式

关联commit: `910dac63`

---

### 类别十: CONFIG_MISSING 配置遗漏(~22条)

dev分支独有细分类别。

---

### 类别: 配置 (CONF)

dev分支独有细分类别。

#### CONF-01: 平台特定编译选项遗漏 [ops-nn-dev]

严重等级: P1

缺陷描述: 特定平台(如ascend910_95)需要额外编译选项(如`-mllvm -cce-aicore-dcci-before-kernel-end=false`), 但大量算子配置中遗漏。

典型代码示例:

```json
// 缺陷 -- ac64e3d38b [dev]
{
    "op_name": "SomeOp",
    "ascend910_95": {
        "compile_options": ""
        // 缺少: "-mllvm -cce-aicore-dcci-before-kernel-end=false"
    }
}
```

审查检查方法:
- ascend910_95平台算子是否配置dcci相关编译选项
- 新增算子时对比同族算子的compile_options确认一致性

关联commit: `ac64e3d38b`

---

#### CONF-02: output paramType(REQUIRED/OPTIONAL)配置错误 [ops-nn-dev]

严重等级: P1

缺陷描述: def.cpp中output的paramType标记与binary.json不一致, 或与算子实际行为矛盾。

典型代码示例:

```cpp
// 缺陷 -- 9c46dc5f19 [dev]
.OUTPUT(rstd, TensorType({DT_FLOAT}))      // 标记为REQUIRED
// 但实际这个输出是可选的

// 修复
.OPTIONAL_OUTPUT(rstd, TensorType({DT_FLOAT}))
```

审查检查方法:
- def.cpp的paramType与binary.json是否双向一致
- REQUIRED输出是否在所有代码路径中都有值

关联commit: `9c46dc5f19`

---

### 类别十一: INT_OVERFLOW 整数溢出/类型截断(~43条)

后果严重(OOM、越界访问、精度错误), 当前代码中仍有12处残留。

---

### 类别: 整数溢出 (INT)

后果严重(OOM、越界访问、精度错误), 当前代码中仍有12处残留。

#### INT-01: shape维度相乘强制使用int64_t [ops-nn][ops-nn-dev]

严重等级: P0

缺陷描述: shape维度相乘的表达式使用int32, 大shape下乘积超过INT32_MAX溢出。

典型代码示例:

```cpp
// 缺陷 -- fa1305c56e [dev]
auto sortIndicesType = DT_INT32;  // 大shape下索引溢出

// 修复
auto sortIndicesType = GetSortIndicesType(shape);
ge::DataType GetSortIndicesType(const Shape& shape) {
    return (shape.GetDim(-1) > INT32_MAX) ? DT_INT64 : DT_INT32;
}
```

```cpp
// 缺陷 -- 8f6ccaea [主仓] (addlayernormgrad)
uint32_t roundUpNumLastDimFloat = ROUND_UP(numLastDim, ...);
// numLastDim > 2^31时已溢出, 后续再乘sizeof(float)二次溢出

// 修复
uint64_t roundUpNumLastDimFloat = ROUND_UP(static_cast<uint64_t>(numLastDim), ...);
```

审查检查方法:
- shape维度乘法变量必须使用int64_t/uint64_t
- tiling结构体中shape/元素数相关字段的类型是否足够宽
- ROUND_UP/CeilAlign等宏的输入值和输出值类型一致性

关联commit: `fa1305c56e`, `8f6ccaea`, `acadb4c7`, `e0ddd962`

---

#### INT-02: GM偏移量在小类型域计算溢出 [ops-nn]

严重等级: P0

缺陷描述: GlobalTensor下标表达式`index * maxDataCount`中, 当index为uint16_t/uint32_t时乘法在小类型域完成后溢出。此模式在ops-nn的foreach算子系列中批量出现(10个文件涉及8-9个算子)。

典型代码示例:

```cpp
// 缺陷 -- 50df91e7 [主仓] (foreach系列10个文件)
DataCopy(dataLocal, inTensorsGM[index * maxDataCount], dataCount);
// index为uint32_t, 乘法在uint32_t域完成后溢出

// 修复
DataCopy(dataLocal, inTensorsGM[1ULL * index * maxDataCount], dataCount);
// 1ULL *强制提升为uint64_t后再乘
```

审查检查方法:
- GM内存偏移计算表达式中至少一个操作数必须为64位类型
- 批量审查: 搜索`Gm[expr * expr]`或`inputGm[expr * expr]`模式

关联commit: `50df91e7`

---

#### INT-03: 硬件指令参数位宽限制校验缺失 [ops-nn-dev]

严重等级: P0

缺陷描述: 硬件指令的参数有位宽限制(如load3d的kStartPt字段上限65535/16位), 超限时溢出翻转。

典型代码示例:

```cpp
// 缺陷 -- 952313ace0 [dev]
int kStartPt = k0 * hk * wk;  // 超过65536时溢出翻转!

// 修复
const int LOAD3D_KSTART_MAX = 65535;
if (load3dK > LOAD3D_KSTART_MAX) {
    UseLargeKernelSplit();
}
```

审查检查方法:
- 涉及硬件指令参数时确认所有字段位宽限制
- tiling代码中对硬件参数做边界校验

关联commit: `952313ace0`

---

#### INT-04: uint64_t减法下溢 [ops-nn-dev]

严重等级: P0

缺陷描述: size_t/uint64_t做减法时被减数小于减数, 结果下溢为极大正整数, 后续用于内存分配或循环控制导致OOM或越界。

典型代码示例:

```cpp
// 缺陷 -- hotspot [dev] group_norm_silu_tiling.cpp
uint64_t remainUbSize = totalUbSize - usedSize;
// usedSize > totalUbSize时下溢为巨大正整数

// 修复
if (totalUbSize <= usedSize) { return ERROR_INSUFFICIENT_UB; }
uint64_t remainUbSize = totalUbSize - usedSize;
```

审查检查方法:
- size_t/uint64_t做减法前是否判断被减数>=减数
- CeilAlign/CeilDiv返回值参与二次乘法时结果类型是否足够宽

关联commit: hotspot `group_norm_silu_tiling.cpp`, `aclnn_matmul.cpp`

---

#### INT-05: 有符号/无符号混比与类型不一致 [ops-nn]

严重等级: P2

缺陷描述: 无符号常量与有符号shape值比较引发-Wsign-compare告警; 模板参数COMP_T为int32时局部变量声明为uint64_t导致不必要的类型提升。

典型代码示例:

```cpp
// 缺陷 -- 941d3c7f [主仓]
// kSupportedInnerAxis为uint64_t常量, shape维度值为int64_t
if (shape.GetDim(i) > kSupportedInnerAxis) { ... }

// 修复
static constexpr int64_t kSupportedInnerAxis = ...;  // 类型一致
```

审查检查方法:
- 开启`-Wsign-compare`并视为错误
- 常量类型与比较对象类型一致
- 使用模板参数COMP_T的函数中, 局部变量类型应与COMP_T一致

关联commit: `941d3c7f`, `0253078d`

---

### 类别十二: PIPELINE_SYNC 流水线同步/硬件事件(~31条)

NPU硬件同步原语使用错误, 后果为精度错误或硬件异常。

---

### 类别: 流水线同步 (SYNC)

NPU硬件同步原语使用错误, 后果为精度错误或硬件异常。

#### SYNC-01: SetFlag/WaitFlag必须成对出现且无条件执行 [ops-nn-dev]

严重等级: P0

缺陷描述: WaitFlag被条件化(if ni>0等), 导致某些迭代路径跳过必要的同步等待。SetFlag后必须有对应WaitFlag, 且不能被任何条件分支跳过。

典型代码示例:

```cpp
// 缺陷 -- a79b527fe8 [dev]
if (ni > 0) {
    WaitFlag<MTE3_MTE2>(eventId);  // ni==0时跳过等待!
}
SetFlag<MTE3_MTE2>(eventId);       // 但SetFlag无条件执行

// 修复
WaitFlag<MTE3_MTE2>(eventId);  // 始终等待
SetFlag<MTE3_MTE2>(eventId);
```

审查检查方法:
- SetFlag和WaitFlag是否成对出现且无条件执行
- 不应在循环边界条件中跳过同步事件

关联commit: `a79b527fe8`

---

#### SYNC-02: CopyOut/CopyIn前的同步事件与实际DMA通道匹配 [ops-nn-dev]

严重等级: P0

缺陷描述: CopyOut走MTE3通道, 但前置同步使用了S_MTE2事件(搬入通道), 导致同步了错误的通道。

典型代码示例:

```cpp
// 缺陷 -- 51f8247aee [dev]
WaitFlag<S_MTE2>(eventId);
CopyOut(dst, src, count);  // CopyOut走MTE3通道

// 修复
WaitFlag<S_MTE3>(eventId);
WaitFlag<V_MTE3>(eventId2);  // 确保Vector计算完成
CopyOut(dst, src, count);
```

审查检查方法:
- CopyOut前的同步是否用MTE3通道(不是MTE2)
- CopyIn前的同步是否用MTE2通道

关联commit: `51f8247aee`

---

#### SYNC-03: 连续matmul操作间必须插入PipeBarrier [ops-nn-dev]

严重等级: P1

缺陷描述: 两次matmul操作间缺少PipeBarrier<PIPE_ALL>()同步。

典型代码示例:

```cpp
// 缺陷 -- 6beab61dd0 [dev]
matmulL.Iterate();
matmulR.Iterate();  // 可能读取matmulL的不完整结果

// 修复
matmulL.Iterate();
PipeBarrier<PIPE_ALL>();
matmulR.Iterate();
```

审查检查方法:
- 连续matmul操作间是否插入PipeBarrier
- 同一kernel中eventID是否有冲突

关联commit: `6beab61dd0`

---

#### SYNC-04: SetScheduleMode缺失 [ops-nn]

严重等级: P1

缺陷描述: tiling函数中缺少SetScheduleMode(1)调用, 多核间存在数据依赖但未设置batch mode同步。

典型代码示例:

```cpp
// 缺陷 -- afb09c78 [主仓]
void TilingFunc(TilingContext* context) {
    context->SetBlockDim(blockDim);  // 缺少SetScheduleMode

    // 修复
    context->SetScheduleMode(1);
    context->SetBlockDim(blockDim);
}
```

审查检查方法:
- 所有tiling函数必须显式设置SetScheduleMode
- SetScheduleMode的参数值(0/1/2)是否与实际同步需求匹配

关联commit: `afb09c78`

---

#### SYNC-05: SetFlag/WaitFlag顺序错误 [ops-nn]

严重等级: P0

缺陷描述: WaitFlag放在SetFlag之前, 等待的是上一轮flag而非当前轮。同时提前return路径遗漏event平衡。

典型代码示例:

```cpp
// 缺陷 -- ed0bd5a1 [主仓] (QBMMv4)
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
// ... 多步操作 ...
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);  // WaitFlag位置太靠后
// 同时提前return路径缺少SetFlag<V_MTE2>

// 修复
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);  // 紧跟Set后
// 提前return路径补充SetFlag + ping-pong翻转
```

审查检查方法:
- SetFlag/WaitFlag必须成对且顺序正确
- pipe方向必须与实际数据流向一致(MTE2=搬入, V=向量计算, MTE3=搬出)

关联commit: `ed0bd5a1`

---

#### SYNC-06: DataCopy前后缺少流水线同步事件 [ops-nn]

严重等级: P0

缺陷描述: DataCopy(MTE2)前缺少MTE3_MTE2同步, 向量运算前缺少MTE2_V同步。运行时表现为相同输入产生不同输出。

典型代码示例:

```cpp
// 缺陷 -- 74d43ef3 [主仓] (scatter_sub)
DataCopy(localBuffer, gmSrc, dataLen);     // MTE2搬入
// 缺少WaitFlag<PIPE_MTE2, PIPE_V>
VecAdd(result, localBuffer, localOther);   // 可能读到未就绪数据

// 修复
DataCopy(localBuffer, gmSrc, dataLen);
SetFlag<PIPE_MTE2, PIPE_V>(eventId);
WaitFlag<PIPE_MTE2, PIPE_V>(eventId);
VecAdd(result, localBuffer, localOther);
```

审查检查方法:
- DataCopy(MTE2)前确保前序MTE3完成
- 向量运算(V)前确保MTE2数据就绪

关联commit: `74d43ef3`

---

### 类别十三: LOG_ERROR 日志/错误处理(~11条)

---

### 类别: 日志 (LOG)

#### LOG-01: 错误码区分PARAM_*和INNER_* [ops-nn-dev]

严重等级: P3

缺陷描述: 对用户传入参数的校验失败使用INNER_*错误码(内部错误), 应使用PARAM_*错误码。

典型代码示例:

```cpp
// 缺陷 -- 797589989a [dev]
CHECK_NOT_NULL(userInput, ACLNN_ERR_INNER_NULLPTR);
// 应为
CHECK_NOT_NULL(userInput, ACLNN_ERR_PARAM_NULLPTR);
```

审查检查方法:
- 错误码区分PARAM_*(用户输入)和INNER_*(内部不可达状态)
- 错误日志中条件描述是否与实际判断条件一致

关联commit: `797589989a`, `dc3a8850e7`

---

### 类别十四: DATA_TYPE 数据类型(~10条)

---

### 类别: 数据类型 (DTYPE)

#### DTYPE-01: 新增数据类型支持时三处同步更新 [ops-nn-dev]

严重等级: P1

缺陷描述: 新增数据类型支持时, tiling层dtype校验、kernel层预编译条件、binary配置三处必须同步更新。

典型代码示例:

```cpp
// 缺陷 -- 9150a7cca8 [dev]
// tiling层新增了int8支持
if (dtype == DT_INT8) { ... }
// 但kernel层预编译条件未更新
#if TILING_KEY_IS(1)  // 仅float/float16
// int8无对应kernel
// binary配置也未更新
```

审查检查方法:
- 新增数据类型时同步检查tiling层、kernel层、binary配置三处
- 选择计算模板/优化路径时检查是否对所有支持的数据类型兼容

关联commit: `9150a7cca8`, `84ab237ad3`

---

### 类别十五: RESOURCE 资源管理/状态初始化(~15条)

---

### 类别: 资源管理 (RES)

#### RES-01: 禁止在op_host/*.cpp中使用非const全局容器 [ops-nn-dev]

严重等级: P0

缺陷描述: 在.cpp文件作用域定义全局std::vector等容器, 多实例并发调用时共享导致数据竞争。

典型代码示例:

```cpp
// 缺陷 -- 2a15cd4d [dev]
std::vector<gert::Stride> indexstrideList;  // 全局! 并发不安全

void CalcTiling() {
    indexstrideList.push_back(...);  // 多实例并发写 -> AIC错误
}

// 修复
class IndexNonContinuousTiling {
private:
    std::vector<gert::Stride> indexstrideList_;  // 每个实例独立
};
```

审查检查方法:
- op_host/*.cpp中是否有非const全局变量/容器声明
- tiling相关状态必须封装在类成员中

关联commit: `2a15cd4d`, revert `2b00825d3`

---

#### RES-02: L1全载优化在循环中的状态失效检查 [ops-nn-dev]

严重等级: P1

缺陷描述: L1/L0缓存全载优化在多batch/多group循环中, 当参数(如B矩阵偏移)变化时未切换全载状态。

典型代码示例:

```cpp
// 缺陷 -- 46f5817ef1 [dev]
for (int batch = 0; batch < batchNum; batch++) {
    if (enableFullLoad) {
        UseL1CachedData();  // B偏移已变, L1数据失效!
    }
}

// 修复
if (curOffsetB != preOffsetB_) {
    enableFullLoad = false;
    ReleaseL1Buffer();
    preOffsetB_ = curOffsetB;
}
```

审查检查方法:
- L1全载优化在循环中, 检查全载条件是否因参数变化而失效

关联commit: `46f5817ef1`

---

#### RES-03: PipeBarrier与FreeTensor时序颠倒 [ops-nn]

严重等级: P0

缺陷描述: PipeBarrier<PIPE_V>放在FreeTensor之后, 导致Cast/向量运算未完成就释放了输入buffer。

典型代码示例:

```cpp
// 缺陷 -- cf84e222 [主仓] (AddRmsNorm)
FreeTensor(inputBuffer);     // 先释放
PipeBarrier<PIPE_V>();       // 后等待

// 修复
PipeBarrier<PIPE_V>();       // 先等待Cast完成
FreeTensor(inputBuffer);     // 后释放
```

审查检查方法:
- PipeBarrier必须在FreeTensor之前
- 搜索FreeTensor调用, 确认前面都有PipeBarrier

关联commit: `cf84e222`

---

#### RES-04: 结构体指针成员未初始化 [ops-nn]

严重等级: P0

缺陷描述: C++结构体中指针成员未在声明处初始化, 某些代码路径下成员值为野指针。

典型代码示例:

```cpp
// 缺陷 -- 15e40a48 [主仓] (addmm matmul_util.cpp)
struct PromoteResult {
    const char* logMessage;  // 未初始化 -> 野指针
};
OP_LOGE("%s", result.logMessage);  // 崩溃

// 修复
struct PromoteResult {
    const char* logMessage = nullptr;
};
if (result.logMessage != nullptr) { OP_LOGE("%s", result.logMessage); }
```

审查检查方法:
- C++结构体指针成员必须在声明处初始化为nullptr
- OP_LOGE中`%s`对应的参数必须保证非空

关联commit: `15e40a48`

---

### 类别十六: UT_TEST 单元测试(~8条)

---

### 类别: 单元测试 (UT)

#### UT-01: 禁止注释EXPECT_*/ASSERT_*断言 [ops-nn-dev]

严重等级: P3

缺陷描述: 大量EXPECT_EQ断言被批量注释掉, UT失去验证能力。

典型代码示例:

```cpp
// 缺陷 -- 91c29129 [dev]
// EXPECT_EQ(result, expected);    // 被注释!
// EXPECT_EQ(shape[0], 128);       // 被注释!
```

审查检查方法:
- grep检测`// EXPECT_`和`// ASSERT_`模式
- UT中不同阶段的推理应使用独立context避免状态污染
- tiling参数应由tiling函数自动生成而非手工硬编码

关联commit: `91c29129`, `ee685a90`, `9a18f2ca`

---

### 类别十七: NULLPTR 空指针解引用(~21条)

---

### 类别: 空指针 (NULL)

#### NULL-01: 可选输出tensor(outputMask控制)访问前必须检查null [ops-nn-dev]

严重等级: P0

缺陷描述: 反向传播中dx/dw/dbias由outputMask控制是否计算, 为null时仍被解引用取shape或dtype导致crash。

典型代码示例:

```cpp
// 缺陷 -- e49d5d27c0 [dev]
auto mmDwOutShape2d = CalcShape(gradInput->GetShape());  // gradInput为nullptr时crash

// 修复
auto mmDwOutShape2d = CalcShape(gradWeight->GetShape());  // 使用非空tensor
```

```cpp
// 缺陷 -- 59a561e5 [dev]
PreConv1DBackwardTo2D(gradInput, ...);  // outputMask可能指示不需要计算

// 修复
if (outputMask[0]) { PreConv1DBackwardTo2D(gradInput, ...); }
```

审查检查方法:
- 反向传播中dx/dw/dbias解引用前需null检查或对应mask位检查
- dtype兼容性判断优先使用必选张量(output)而非可选张量

关联commit: `e49d5d27c0`, `59a561e5`, `2463697d99`

---

#### NULL-02: OP_CHECK宏缺少return语句 [ops-nn]

严重等级: P0

缺陷描述: OP_CHECK宏的第三参数写成裸值`nullptr`而非`return nullptr`。OP_CHECK失败时不会return而继续向下执行, 使用空指针导致crash。此模式在单个文件中可批量出现(如aclnn_median.cpp中14处)。

典型代码示例:

```cpp
// 缺陷 -- 7b4a1b53 [主仓] (aclnnNanMedian)
OP_CHECK(condition, "error msg", nullptr);
// 失败时执行 nullptr; 作为表达式语句 -> 什么都不做, 继续执行

// 修复
OP_CHECK(condition, "error msg", return nullptr);
```

审查检查方法:
- OP_CHECK宏的第三参数必须包含return语句
- 静态分析: 检测`, nullptr)`和`, nullptr;)`模式
- 批量审查: 全文搜索`OP_CHECK.*,\s*nullptr\)`

关联commit: `7b4a1b53`

---

#### NULL-03: 先解引用后判空(先用后查) [ops-nn]

严重等级: P0

缺陷描述: 指针先被解引用使用(Contiguous/Cast/TransData等操作), 之后才检查是否为nullptr。此模式在op_api层多处存在。

典型代码示例:

```cpp
// 缺陷 -- hotspot [主仓] (aclnn_addmm.cpp:298-299)
auto selfContiguous = l0op::Contiguous(addmmTensor.self, uniqueExecutor);
if (addmmTensor.self != nullptr && ...)  // 检查太晚

// 缺陷 -- hotspot [主仓] (aclnn_quant_matmul_v5.cpp:907-908)
reformatedX1 = l0op::TransData(reformatedX1, Format::FORMAT_FRACTAL_NZ, 0, executor);
CHECK_RET(x1 != nullptr, ACLNN_ERR_INNER_NULLPTR);
// 赋值给reformatedX1但检查的是x1 -- 检查错误变量
```

审查检查方法:
- 指针解引用(包括作为函数参数传入)必须在nullptr检查之后
- CHECK_RET中检查的变量必须是刚被赋值的变量
- 搜索模式: `= l0op::Xxx(ptr); if (ptr != nullptr)` 或 `CHECK_RET(oldVar != nullptr)`

关联commit: hotspot aclnn_addmm.cpp, aclnn_quant_matmul_v5.cpp

---

### 类别十八: REVERT 性能/功能回退(~12条)

---

### 类别: 回退 (REV)

#### REV-01: 大规模重构应分步提交 [ops-nn-dev]

严重等级: P2

缺陷描述: 大规模算子重构(同时修改TilingData布局、kernel模板化、分核策略)一次性合入, host与device端数据解析不匹配导致全面回退。

典型代码示例:

```
# 缺陷 -- 74cdae15d7 [dev] (NLLLossGrad重构被回退)
# 原始提交同时做了3件事:
# 1. TilingData字段重排 + 类型缩窄(uint64->uint32)
# 2. kernel模板化拆分
# 3. 分核策略变更
# host与device端TilingData解析不匹配 -> 全面回退

# 应拆分为:
# PR1: TilingData字段变更(host+kernel同步)
# PR2: kernel模板化(不改数据结构)
# PR3: 分核策略变更
```

审查检查方法:
- 单次合入超过3000行的PR应要求拆分
- 大规模重构不应同时改变数据结构布局和计算逻辑

关联commit: revert `74cdae15d7`, revert `fe16c557`

---

#### REV-02: 移除binary.json算子变体前确认无下游调用 [ops-nn-dev]

严重等级: P1

缺陷描述: 过早移除binary.json中的算子变体或编译配置, 但下游场景仍有调用。

审查检查方法:
- 移除binary.json变体前grep所有下游场景确认无调用
- 强制禁用已有模板(`return false`)必须注释原因

关联commit: `e682154870`

---

#### REV-03: 搭车提交造成revert附带伤害 [ops-nn]

严重等级: P2

缺陷描述: MR中混入与标题功能不相关的改动, revert时不相关改动被一并回滚。

典型案例:

```
# 事件1 -- d91e1757a [主仓] (unsorted_segment_sum)
# 搭车改动: op_list.md中添加了scatter条目(非本MR功能)
# Revert附带伤害: scatter条目也被删除

# 事件3 -- 4482238c0 [主仓] (LogSigmoid)
# 搭车改动: 修改了binary_cross_entropy的平台判断逻辑
# Revert附带伤害: binary_cross_entropy的修改也被回滚
```

审查检查方法:
- MR中每个文件改动应与标题描述的功能直接相关
- 不相关改动必须拆分为独立MR

关联commit: `d91e1757a`, `4482238c0`

---

#### REV-04: 大规模提交测试不充分 [ops-nn]

严重等级: P2

缺陷描述: 大体量提交(>20文件/>1000行)一次性合入后紧急revert。常见问题: 仅声称"冒烟测试通过"但缺乏全面集成测试。

审查检查方法:
- 大feature合入(>20文件或>1000行)前需全流程CI + 集成测试
- 跨三层(op_api/tiling/kernel)的变更需要额外的回归验证
- 开启`-Wshadow`拦截变量遮蔽

关联commit: `852f21d6b`, `d91e1757a`

---

### 附录A: [主仓] 文档与示例代码(~36条)

主仓独有类别, 非纯文档修复, 指代码仓中的示例代码bug、文档与代码不一致、op_list条目遗漏等。

---

### 类别: 文档 (DOC)

主仓独有类别, 非纯文档修复, 指代码仓中的示例代码bug、文档与代码不一致、op_list条目遗漏等。

#### DOC-01: 示例代码未经编译/运行验证 [ops-nn]

严重等级: P2

缺陷描述: 示例代码包含编译错误(重复函数定义)或运行时输出全0/垃圾数据。

典型代码示例:

```cpp
// 缺陷 -- 501d38ab [主仓]
void AclnnWeightQuantBatchMatmulV2Test() { /* 第一份 */ }
void AclnnWeightQuantBatchMatmulV2Test() { /* 第二份, 重复定义 */ }
```

审查检查方法:
- 示例代码PR必须在CI中包含编译验证步骤
- 示例代码中不应有重复的函数/类定义

关联commit: `501d38ab`, `4063f8f9`

---

#### DOC-02: 示例代码数据类型与API不匹配 [ops-nn]

严重等级: P1

缺陷描述: 示例中host数据容器类型(如`vector<int32_t>`)与创建tensor时传入的aclDataType(如ACL_FLOAT)不匹配。

典型代码示例:

```cpp
// 缺陷 -- 964be4dd [主仓]
std::vector<int32_t> hostData = {...};
aclTensor* tensor = aclCreateTensor(..., ACL_FLOAT, ...);
// 整数比特被解释为浮点数

// 修复
std::vector<float> hostData = {...};
```

审查检查方法:
- 示例中`vector<T>`的T必须与`aclDataType`对应
- 检查aclrtMalloc/aclrtFree是否配对出现

关联commit: `964be4dd`

---

#### DOC-03: op_list条目遗漏或名称不一致 [ops-nn]

严重等级: P2

缺陷描述: 新增算子时op_list.md中条目遗漏或名称与实际算子名不一致。

审查检查方法:
- 新增算子时op_list.md条目名称必须与算子注册名完全一致
- PR中对op_list.md的改动应只包含与PR标题功能直接相关的条目

关联commit: `d91e1757a`

---

### 跨类别系统性风险

---

### 类别: 系统性 (SYS)

#### SYS-01: build.sh系统性脆弱 [ops-nn]

严重等级: P1(跨BUILD/COMPILE_WARN/REVERT, 热点第1, 10条缺陷触及)

缺陷描述: build.sh当前代码仍存在多个P0/P1 bug:
- `set +u`从行35开始从未恢复, 整个脚本期间未定义变量静默为空
- `]`前缺空格(P0)、数组备份只取首元素(P0)、--mssanitizer赋FALSE(P1)
- ASCEND_HOME_PATH未设置时路径变成`/include`等根目录(P1)
- CMAKE_ARGS非数组, 参数含空格时错误拆分
- main通过管道传给gawk, exit在subshell中无法正确传播退出码

审查检查方法:
- build.sh修改必须通过shellcheck验证
- 恢复`set -u`(或至少在关键变量使用处加`${VAR:?}`保护)
- 数组操作使用`"${arr[@]}"`而非`${arr}`

关联commit: `adec83fb`及热点分析

---

#### SYS-02: matmul模块缺陷密度最高 [ops-nn][ops-nn-dev]

严重等级: P0(跨9+个类别)

缺陷描述: matmul/bmm模块两仓合计贡献最多缺陷。dev热点Top 30中占20席, Revert中占45%。当前代码仍存在的确认bug:
- 空指针先用后查(P0): aclnn_addmm.cpp:298-299
- CHECK_RET检查错变量(P0): aclnn_quant_matmul_v5.cpp:907-908
- copy-paste公式相同(P0): mat_mul_deterministic_splitk_kernel.h:404-405
- 除零风险(P1): matmul_v3_base_tiling.cpp:605-606, 80-81
- 溢出截断(P2): matmul_v3_base_tiling.cpp:909-912

审查检查方法:
- matmul相关PR应提高审查标准(多人review)
- 重点检查空指针、copy-paste、维度索引、类型溢出

关联commit: 见热点分析中matmul部分

---

#### SYS-03: 编译warning隐藏真bug [ops-nn][ops-nn-dev]

严重等级: P1(跨CONDITION_LOGIC/COMPILE_WARN)

缺陷描述: 两仓合计55条编译告警修复中至少6条包含真正的逻辑bug: 无符号>=0恒真无限循环、链式比较语义错误、运算符优先级、变量遮蔽掩盖copy-paste、Cast方向反转。

审查检查方法:
- 开启`-Wall -Werror -Wsign-compare -Wshadow -Wtautological-compare`
- warning修复提交应认真review, 可能暴露新的bug

关联commit: `0e00f88c`, `50854088`, `d818b4d4`, `7ad3e89e`

---

#### SYS-04: 提交粒度过大导致质量失控 [ops-nn][ops-nn-dev]

严重等级: P2(跨REVERT)

缺陷描述:
- dev: qbmm 6.5万行导致3次循环revert; weight_quant 5.7万行修改公共error_util.h
- 主仓: unsorted_segment_sum 38文件/8892行; batchmatmul跨三层
- 两仓合计70%的revert在3天内发生

审查检查方法:
- 单次合入超过3000行的PR必须拆分

证据: revert `79623db1a`, `9de027499`, `d91e1757a`, `852f21d6b`

---

#### SYS-05: 公共基础设施耦合在算子PR中 [ops-nn][ops-nn-dev]

严重等级: P2(跨BUILD_CMAKE/REVERT)

缺陷描述: dev分支5次revert + 主仓build.sh系统性脆弱, 均源于公共文件(op_util.h, error_util.h, op_api_def.h, cmake)修改耦合在算子PR中。公共修改影响面远超单算子, 但review时容易被算子逻辑掩盖。

审查检查方法:
- 公共文件修改必须独立PR先行验证

证据: revert `79623db1a`, `9de027499`, `61a1f1583`, `e1fdbe6e8`

---

#### SYS-06: 先用后查反模式在op_api层流行 [ops-nn]

严重等级: P0(跨NULLPTR/COPY_PASTE)

缺陷描述: 空指针先解引用后判空、OP_CHECK后先LOG再CHECK、工厂函数返回值直接使用不检查。热点分析中P0级别的8个确认bug中有3个属于此模式。

典型表现:
- `l0op::Contiguous(ptr)` -> `if (ptr != nullptr)`(先解引用后判空)
- `l0op::Cast(...)` -> `OP_LOGD(result->ToString())` -> `OP_CHECK(result != nullptr)`(先用后查)
- `reformatedX1 = TransData(...)` -> `CHECK_RET(x1 != nullptr)`(检查错误变量)

审查检查方法:
- op_api代码中搜索`= l0op::` / `= Contiguous(` / `= Cast(`后紧跟的if/CHECK_RET是否检查了正确的变量
- 工厂函数/算子调用返回值必须先检查再使用

关联commit: hotspot aclnn_addmm.cpp, aclnn_quant_matmul_v5.cpp

---

#### SYS-07: ascendc_config.json配置混乱 [ops-nn]

严重等级: P2(跨BUILD_CMAKE/CONFIG_MISSING)

缺陷描述: ascendc_config.json当前存在的问题:
- 完全重复条目: AdaptiveAvgPool3d(2份)、DeformableOffsetsGrad(2份)、LayerNormQuant(3份)、GetPaddingOffset(3份)
- 同名冲突配置: compute_units/compile_options/impl_mode不一致
- 无效引用: compile_options中引用`ascend910_95`非已知平台标识
- 哪一条生效取决于JSON解析器实现, 构建行为不确定

审查检查方法:
- 修改ascendc_config.json的PR应先搜索是否已有同名条目
- CI中应加入重复条目检测脚本

关联commit: 热点分析(hotspot_analysis.md)

---

#### SYS-08: foreach模块高触及但低风险 [ops-nn][ops-nn-dev]

说明: 聚合触及次数最高(1452次), 但本质是批量模板代码同步修改。单个算子结构性风险低, 真正需重点review的是matmul和quant的kernel/tiling层。不应被热点文件列表误导分配不当的审查资源。

---

### 审查规则优先级矩阵

#### P0 -- 必须检查(高频高危, 可直接拦截严重缺陷)

| 规则ID   | 规则                                                    | 覆盖类别                | 典型案例hash                         |
|----------|--------------------------------------------------------|------------------------|-------------------------------------|
| INT-01   | shape维度相乘/buffer偏移强制使用int64_t                  | INT_OVERFLOW           | fa1305c56e, 8f6ccaea, e0ddd962      |
| TILE-01  | tiling侧buffer计算必须与kernel侧InitBuffer交叉比对      | TILING_BUFFER          | e25e6e60c7, b99dcb02c5              |
| BOUND-02 | 多核任务分配区分tail core和idle core                     | BOUNDARY               | e2fc1fecb1                          |
| SYNC-01  | SetFlag/WaitFlag必须成对出现且无条件执行                  | PIPELINE_SYNC          | a79b527fe8, 51f8247aee              |
| COND-01  | 设置特殊状态(UnknownRank)后必须包含return                | CONDITION_LOGIC        | 2d424df8e6                          |
| TILE-03  | tiling参数区分normal核和tail核                           | TILING_BUFFER          | 90649ac299                          |
| TILE-04  | TilingData结构体host-kernel完全一致                      | TILING_BUFFER          | 0d32207040                          |
| SYNC-02  | CopyOut/CopyIn前同步事件与DMA通道匹配                    | PIPELINE_SYNC          | 51f8247aee                          |
| NULL-01  | 可选输出tensor访问前必须检查null                          | NULLPTR                | 59a561e5, e49d5d27c0                |
| CALC-02  | 多核场景全局内存初始化由指定核完成+同步屏障                | CALC_LOGIC             | fa03888b0d                          |
| RES-01   | 禁止op_host中非const全局容器                             | RESOURCE               | 2a15cd4d                            |
| INT-02   | [ops-nn] GM偏移量在小类型域计算溢出, 用1ULL*强制64位        | INT_OVERFLOW           | 50df91e7                            |
| INT-03   | 硬件指令参数位宽限制校验                                  | INT_OVERFLOW           | 952313ace0                          |
| INT-04   | uint64_t减法前检查被减数>=减数                           | INT_OVERFLOW           | hotspot group_norm_silu_tiling.cpp  |
| BOUND-04 | [ops-nn] !=A\|\|!=B恒真逻辑错误(De Morgan)                | BOUNDARY               | 67c665fd, 692f43a9                  |
| NULL-02  | [ops-nn] OP_CHECK宏第三参数必须包含return语句               | NULLPTR                | 7b4a1b53                            |
| NULL-03  | [ops-nn] 先检查后使用, 禁止先解引用后判空                   | NULLPTR                | hotspot aclnn_addmm.cpp             |
| WARN-03  | [ops-nn] 无符号>=0恒真无限循环                              | COMPILE_WARN           | 0e00f88c                            |
| COPY-03  | [ops-nn] 矩阵k/n维度索引粘贴错误                           | COPY_PASTE             | 1c2de786                            |
| SYNC-05  | [ops-nn] SetFlag/WaitFlag顺序错误                          | PIPELINE_SYNC          | ed0bd5a1                            |
| SYNC-06  | [ops-nn] DataCopy前后缺少流水线同步事件                     | PIPELINE_SYNC          | 74d43ef3                            |
| PLAT-03  | [ops-nn] DMA/datacopy参数超16位上限                        | PLATFORM               | 34217db7, cf9ea8c9                  |
| RES-03   | [ops-nn] PipeBarrier与FreeTensor时序颠倒                   | RESOURCE               | cf84e222                            |
| RES-04   | [ops-nn] 结构体指针成员必须初始化为nullptr                  | RESOURCE               | 15e40a48                            |
| SYS-02   | matmul模块缺陷密度最高                                    | 跨类别                 | hotspot matmul部分                  |
| SYS-06   | [ops-nn] 先用后查反模式在op_api层流行                       | 跨类别                 | hotspot aclnn_*.cpp                 |

#### P1 -- 重点检查(中频高危或高频中危)

| 规则ID   | 规则                                                    | 覆盖类别                | 典型案例hash                         |
|----------|--------------------------------------------------------|------------------------|-------------------------------------|
| API-01   | 修改Init签名/模板参数后全局搜索调用点同步更新              | INTERFACE_API          | b14f5a03ca                          |
| BUILD-01 | 新增算子checklist: ascendc_config.json + binary.json     | BUILD_CMAKE            | c00e940bfa                          |
| COPY-02  | 函数调用中同类型参数是否重复                              | COPY_PASTE             | dec58d1b24, b2e2cada                |
| PLAT-01  | `__CCE_AICORE__==xxx`精确匹配改为范围比较                | PLATFORM               | 4f87bea9ab                          |
| COND-02  | 除法运算除数非零保护                                     | CONDITION_LOGIC        | 22f84c0e                            |
| DTYPE-01 | 新增数据类型时同步tiling/kernel/binary三处               | DATA_TYPE              | 9150a7cca8                          |
| TILE-02  | workspace计算公式逐项确认双buffer需求                    | TILING_BUFFER          | e32a5eaae3                          |
| CONF-01  | 平台特定编译选项(dcci等)完整性                            | CONFIG_MISSING         | ac64e3d38b                          |
| SYNC-03  | 连续matmul操作间插入PipeBarrier                          | PIPELINE_SYNC          | 6beab61dd0                          |
| BOUND-01 | 空tensor输入正确early return                             | BOUNDARY               | 95c2fbb6bf                          |
| BOUND-03 | 维度扩展后判断条件适配                                    | BOUNDARY               | 08b9ebb112                          |
| CALC-01  | CeilDiv结果与实际shape做min(上界clamp)                   | CALC_LOGIC             | a70d8b16d5                          |
| CALC-03  | splitK场景Bias加载时机(kIndex==0)                        | CALC_LOGIC             | e7048cff4f                          |
| WARN-02  | 链式比较语义陷阱                                         | COMPILE_WARN           | 7ad3e89e, 50854088                  |
| RES-02   | L1全载优化循环中状态失效检查                              | RESOURCE               | 46f5817ef1                          |
| API-02   | TilingParse注册类型与CompileInfo类型一致                  | INTERFACE_API          | 368607bd                            |
| PLAT-02  | compute_units列表覆盖所有目标平台                         | PLATFORM               | 15138bbd86, db31f7273               |
| CONF-02  | output paramType(REQUIRED/OPTIONAL)配置正确              | CONFIG_MISSING         | 9c46dc5f19                          |
| TILE-05  | [ops-nn] StorageShape/ViewShape混淆                        | TILING_BUFFER          | 6b52fdcf, 28ab6d70                  |
| TILE-06  | [ops-nn] tiling key参数传递错误与dead code                  | TILING_BUFFER          | 6eee5478                            |
| API-03   | [ops-nn] 接口调用默认参数与实际数据类型一致                  | INTERFACE_API          | c8ca6bec                            |
| API-04   | [ops-nn] 属性索引硬编码顺序依赖                             | INTERFACE_API          | 07e77ddd                            |
| API-05   | [ops-nn] 变体接口复用通用dtype校验                          | INTERFACE_API          | 0bd3fe0b                            |
| BOUND-05 | [ops-nn] 尾块/边界条件处理遗漏                              | BOUNDARY               | 多条                                |
| COND-04  | [ops-nn] 新增类型/分支改变默认路径行为                       | CONDITION_LOGIC        | 4d315436                            |
| COPY-04  | [ops-nn] GM偏移量公式维度混淆                               | COPY_PASTE             | 910dac63                            |
| SYNC-04  | [ops-nn] SetScheduleMode必须显式设置                        | PIPELINE_SYNC          | afb09c78                            |
| REV-02   | 移除binary.json变体前确认无下游调用                       | REVERT                 | e682154870                          |
| DOC-02   | [ops-nn] 示例代码数据类型与API匹配                          | 文档                   | 964be4dd                            |
| BUILD-05 | [ops-nn] ascendc_config.json重复/冲突配置                   | BUILD_CMAKE            | hotspot_analysis.md                 |
| SYS-01   | [ops-nn] build.sh系统性脆弱                                 | 跨类别                 | adec83fb, hotspot                   |
| SYS-03   | 编译warning隐藏真bug                                      | 跨类别                 | 0e00f88c, 50854088, d818b4d4        |

#### P2 -- 常规检查(中危或可部分自动化)

| 规则ID   | 规则                                                    | 覆盖类别                | 典型案例hash                         |
|----------|--------------------------------------------------------|------------------------|-------------------------------------|
| SYS-04   | 单次合入超过3000行必须拆分                                | 跨类别                 | qbmm事件                            |
| SYS-05   | 公共文件修改必须独立PR                                    | 跨类别                 | revert分析                          |
| WARN-01  | 删除"未使用变量"前检查是否原应被使用                       | COMPILE_WARN           | d818b4d4                            |
| COPY-01  | 日志引用变量与判断条件一致                                 | COPY_PASTE             | d1832c87c9                          |
| BUILD-02 | CMake DEPENDENCIES列表完整                                | BUILD_CMAKE            | 640a1683, fb3b8365b9                |
| BUILD-04 | 新平台binary配置遗漏                                     | BUILD_CMAKE            | d6acf37e                            |
| COND-03  | 宏定义中隐藏return语句                                    | CONDITION_LOGIC        | 6beab61dd0                          |
| PLAT-04  | [ops-nn] 硬件宏条件编译遗漏                                 | PLATFORM               | c1469b68                            |
| INT-05   | [ops-nn] 有符号/无符号混比                                  | INT_OVERFLOW           | 941d3c7f, 0253078d                  |
| REV-01   | 大规模重构分步提交                                        | REVERT                 | 74cdae15d7                          |
| REV-03   | [ops-nn] 搭车提交造成revert附带伤害                         | REVERT                 | d91e1757a, 4482238c0                |
| REV-04   | [ops-nn] 大规模提交测试不充分                               | REVERT                 | 852f21d6b                           |
| DOC-01   | [ops-nn] 示例代码未经编译/运行验证                          | 文档                   | 501d38ab, 4063f8f9                  |
| DOC-03   | [ops-nn] op_list条目遗漏或名称不一致                        | 文档                   | d91e1757a                           |
| SYS-07   | [ops-nn] ascendc_config.json配置混乱                        | 跨类别                 | hotspot_analysis.md                 |

#### P3 -- 建议自动化(可通过工具/CI拦截)

| 规则ID   | 规则                                                    | 实现方式                |
|----------|--------------------------------------------------------|------------------------|
| AUTO-01  | 启用: -Werror -Wshadow -Wconversion -Wsign-compare -Wparentheses | CMake配置     |
| AUTO-02  | shape维度算术运算lint规则(检测int32乘法)                  | 自定义lint              |
| AUTO-03  | ascendc_config.json与算子目录交叉校验                     | CI脚本                 |
| AUTO-04  | SetFlag/WaitFlag配对检测                                 | 静态分析               |
| AUTO-05  | "// EXPECT_"注释断言检测                                  | grep规则               |
| AUTO-06  | compile_options平台差异检测                                | CI脚本                 |
| LOG-01   | 错误码区分PARAM_*和INNER_*                                | 代码审查               |
| UT-01    | 禁止注释EXPECT_*/ASSERT_*断言                             | grep规则               |

---

### 数据来源

- 主仓: ~/repo/cann/ops-nn/ (1474次非merge提交, 380条缺陷, 25.8%)
- dev分支: ~/repo/cann/ops-nn-dev/ (2571次非merge提交, 612条缺陷, 23.8%)
- 合计: 4045次提交, 992条缺陷(24.5%)
- 实际分析: 主仓~321条 + dev 456条 = ~777条(排除纯文档/UT/清理)
- Revert事件: 主仓4个 + dev 17个 = 21个独立事件(25条revert提交)
- 分析周期: 全量git历史(2025-08 ~ 2026-03)
