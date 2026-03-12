## ops-nn-dev项目缺陷模式与审查规则

基于ops-nn-dev仓库2571次提交的完整git历史分析，从612条缺陷提交（占比23.8%）中提炼的34条高价值审查规则。每条规则均有commit证据和实际代码支撑。

ops-nn-dev是NN算子开发仓库（ops-nn的dev分支），覆盖matmul、conv、pooling、norm、quant、index、foreach等模块。与ops-nn主仓相比，dev分支的BUILD_CMAKE(14.3% vs ~5%)和INTERFACE_API(9.6% vs ~4%)占比显著更高，反映开发分支构建基础设施更不稳定、接口变更更频繁。本文档的18个缺陷类别从456条实际分析的缺陷数据中自然涌现，未套用其他仓库框架。

### 严重等级定义

| 等级 | 含义 | 影响范围 |
|------|------|----------|
| P0 | 致命 | CoreDump、数据损坏、内存越界、死锁/无限循环 |
| P1 | 严重 | 特定配置崩溃、静默精度错误、功能不可用 |
| P2 | 一般 | 边界条件异常、构建失败、测试不通过 |
| P3 | 建议 | 代码质量、可维护性、潜在隐患 |

### 缺陷分布总览

| 序号 | 缺陷类别 | 频次 | 占比 | 严重度 | 规则ID前缀 |
|------|---------|------|------|--------|-----------|
| 1 | BUILD_CMAKE 构建系统/CMake | 65 | 14.3% | 中 | BUILD |
| 2 | BOUNDARY 边界条件缺失 | 47 | 10.3% | 高 | BOUND |
| 3 | TILING_BUFFER Tiling/Buffer/Workspace计算 | 44 | 9.6% | 高 | TILE |
| 4 | INTERFACE_API 接口/API变更传播 | 44 | 9.6% | 中 | API |
| 5 | CALC_LOGIC 计算逻辑错误 | 35 | 7.7% | 高 | CALC |
| 6 | COMPILE_WARN 编译告警/代码规范 | 34 | 7.5% | 低-中 | WARN |
| 7 | CONDITION_LOGIC 条件判断/控制流 | 31 | 6.8% | 高 | COND |
| 8 | PLATFORM 平台/SoC适配 | 29 | 6.4% | 高 | PLAT |
| 9 | COPY_PASTE 复制粘贴错误 | 24 | 5.3% | 中 | COPY |
| 10 | CONFIG_MISSING 配置遗漏 | 22 | 4.8% | 中 | CONF |
| 11 | INT_OVERFLOW 整数溢出/类型截断 | 18 | 3.9% | 高 | INT |
| 12 | PIPELINE_SYNC 流水线同步/硬件事件 | 16 | 3.5% | 高 | SYNC |
| 13 | LOG_ERROR 日志/错误处理 | 11 | 2.4% | 低 | LOG |
| 14 | DATA_TYPE 数据类型 | 10 | 2.2% | 中 | DTYPE |
| 15 | RESOURCE 资源管理/状态初始化 | 8 | 1.8% | 高 | RES |
| 16 | UT_TEST 单元测试 | 8 | 1.8% | 低 | UT |
| 17 | NULLPTR 空指针解引用 | 6 | 1.3% | 高 | NULL |
| 18 | REVERT 性能/功能回退 | 4 | 0.9% | 高 | REV |
| - | 跨类别系统性风险 | - | - | - | SYS |

模块缺陷密度: matmul模块贡献最多缺陷，热点Top 30文件中matmul占20席，是绝对缺陷震中。quant/conv/norm依次为第二至四热点模块。

---

### 类别一：BUILD_CMAKE 构建系统/CMake（65条，14.3%）

最高频类别，dev分支占比远高于主仓。多为机械性错误，可通过CI自动化大幅减少。子类型：CMake依赖/链接声明缺失(~18)、include路径错误(~12)、打包/安装配置遗漏(~10)、编译选项缺失(~8)、构建脚本逻辑错误(~8)、文件命名/目录结构错误(~5)。

#### 规则 BUILD-01: 新增算子打包配置遗漏

严重等级: P2

缺陷描述: 新增算子未在ascendc_config.json中注册，导致kernel二进制不被打包，运行时找不到预编译kernel触发JIT异常或直接失败。

典型代码示例:

```json
// 缺陷 — c00e940bfa
// HardSwishGradV2算子目录和代码都已就绪，但ascendc_config.json中缺少注册条目
// 运行时：kernel binary not found

// 修复 — 在ascendc_config.json中添加算子条目
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

#### 规则 BUILD-02: CMake DEPENDENCIES列表不完整

严重等级: P2

缺陷描述: CMakeLists.txt中target的DEPENDENCIES未包含前置依赖，导致并行构建时序不确定，表现为本地编译通过但CI偶发失败。

典型代码示例:

```cmake
# 缺陷 — fb3b8365b9
# op_api UT编译缺少OPAPI_INCLUDE路径
target_include_directories(test_xxx PRIVATE ${SOME_PATH})
# 缺少: ${OPAPI_INCLUDE} 导致头文件找不到

# 修复 — 补充include路径
target_include_directories(test_xxx PRIVATE ${SOME_PATH} ${OPAPI_INCLUDE})
```

审查检查方法:
- CMakeLists.txt变更时检查DEPENDENCIES和include_directories是否完整
- 对比同族算子的CMake配置确认无遗漏

关联commit: `fb3b8365b9`, `5f3a97918e`

---

#### 规则 BUILD-03: 构建脚本缺少错误处理

严重等级: P2

缺陷描述: 构建shell脚本缺少`set -e`或文件操作前不检查存在性，构建失败被静默忽略，下游得到不完整或过期的产物。

典型代码示例:

```bash
# 缺陷 — 9add7e2316
# build.sh中某步骤失败后继续执行后续步骤
cp $SRC $DST       # SRC不存在时cp失败但脚本继续
make -j$(nproc)    # 使用了过期文件

# 修复 — 添加错误处理
set -e
set -o pipefail
[ -f "$SRC" ] && cp "$SRC" "$DST" || { echo "ERROR: $SRC not found"; exit 1; }
```

审查检查方法:
- 构建脚本首行是否有`set -e`/`set -o pipefail`
- 文件操作前是否做`[ -f ]`检查

关联commit: `9add7e2316`

---

### 类别二：BOUNDARY 边界条件缺失（47条，10.3%）

第二高频且严重度高。算子对非典型输入（空tensor、小shape、大shape、退化维度）的处理普遍不充分。子类型：空tensor处理缺失(~12)、尾块/尾核处理不完整(~8)、大shape越界(~6)、退化维度(~5)、输入校验遗漏(~5)、小shape(~4)、format变体(~4)、off-by-one(~3)。

#### 规则 BOUND-01: 空tensor输入未正确处理

严重等级: P1

缺陷描述: 空tensor输入时算子未正确early return并设置空输出shape，导致后续计算对零长度维度执行非预期操作。输入空和输出空语义不同，不能用`||`合并处理。

典型代码示例:

```cpp
// 缺陷 — aclnn_batch_matmul.cpp (hotspot, 9次触及)
// CreateBatchMatmulGraphImpl中先判断空tensor创建BatchmmEmptyTensorGraph
// 但紧接着无条件覆盖为BatchMatmulExecBmmOpGraph，缺少else
auto graph = CreateBatchmmEmptyTensorGraph(...);  // 空tensor分支
graph = CreateBatchMatmulExecBmmOpGraph(...);      // 无条件覆盖！死代码

// 修复 — 补充else分支
if (IsEmptyTensor(...)) {
    graph = CreateBatchmmEmptyTensorGraph(...);
} else {
    graph = CreateBatchMatmulExecBmmOpGraph(...);
}
```

审查检查方法:
- 空tensor判断后是否有return或else保护后续逻辑
- 输入空和输出空是否分别处理（不要用`||`合并）
- 空tensor分支是否正确设置输出shape

关联commit: hotspot `aclnn_batch_matmul.cpp`, 缺陷类`95c2fbb6bf`

---

#### 规则 BOUND-02: 多核任务分配未区分tail core和idle core

严重等级: P0

缺陷描述: group卷积/分核场景下，`>=`判断将合法尾部core和超范围idle core合并处理，导致超范围core使用错误数据量计算，产生精度问题或越界访问。

典型代码示例:

```cpp
// 缺陷 — e2fc1fecb1
// group卷积 >= 判断合并了两种不同语义的core
if (coreIdx >= normalCoreNum) {
    // 这里同时处理了：
    // 1. coreIdx == normalCoreNum (合法tail core，有少量数据)
    // 2. coreIdx > normalCoreNum (idle core，无数据)
    ProcessData(tailDataSize);  // idle core不应执行此操作
}

// 修复 — 拆分为两个分支
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

#### 规则 BOUND-03: 维度扩展后判断条件未适配

严重等级: P1

缺陷描述: 2D场景tensor扩展为5D(D=1)后，CheckOutputAllZero要求所有维度为0才推断输出shape，D=1导致返回false，infershape流程失败。维度扩展引入的默认值影响所有后续判断。

典型代码示例:

```cpp
// 缺陷 — 08b9ebb112
// CheckOutputAllZero遍历所有5个维度，要求全部为0
// 但2D扩展为5D后D维度=1（非0），函数返回false
bool CheckOutputAllZero(const Shape& shape) {
    for (int i = 0; i < shape.GetDimNum(); i++) {
        if (shape.GetDim(i) != 0) return false;  // D=1时返回false
    }
    return true;
}

// 修复 — 新增CheckOutputAllZeroFrom2D处理扩展维度
bool CheckOutputAllZeroFrom2D(const Shape& shape) {
    // 跳过扩展维度(D维度)，只检查原始有效维度
    ...
}
```

审查检查方法:
- 同时支持2D和3D且通过维度扩展统一处理时，所有依赖维度值的判断需考虑扩展维度默认值
- 输入索引用命名常量不硬编码

关联commit: `08b9ebb112`

---

### 类别三：TILING_BUFFER Tiling/Buffer/Workspace计算（44条，9.6%）

核心运行时缺陷，直接导致OOM、精度错误、性能劣化。子类型：workspace大小计算公式错误(~10)、tiling参数遗漏/计算不一致(~8)、UB空间分配不足(~7)、buffer数量/double buffer策略不一致(~5)、tiling key选择错误(~5)、对齐粒度错误(~4)、L1内存预留不足(~3)、tiling结构体不一致(~2)。

#### 规则 TILE-01: tiling侧buffer计算与kernel侧InitBuffer不一致

严重等级: P0

缺陷描述: tiling侧用公式估算buffer大小，但kernel侧实际buffer分配策略不同（buffer数量、double buffer策略），导致UB溢出或数据覆盖。这是tiling/buffer类缺陷中最具代表性的模式。

典型代码示例:

```cpp
// 缺陷 — e25e6e60c7
// tiling侧公式
bufferSize = ubSize_ / DOUBLE_BUFFER / bufferCount;
// 假设所有buffer均匀分配

// 但kernel侧实际：x和y各占2份buffer(double buffer)，
// scale/shift又有不同策略(NO_DOUBLE_BUFFER)
// 两端计算逻辑不一致，导致tiling算出的buffer大小 > kernel实际可用

// 修复 — 统一tiling侧公式
bufferSize = ubSize_ / (X_Y_BUFFER_NUM * DOUBLE_BUFFER + 1 + hasScaleShift);
// kernel侧scale/shift统一NO_DOUBLE_BUFFER
```

审查检查方法:
- tiling侧buffer计算必须与kernel侧InitBuffer做交叉比对
- 检查buffer数量和double buffer策略是否两端一致
- 建议用共享常量定义buffer数量

关联commit: `e25e6e60c7`

---

#### 规则 TILE-02: workspace计算公式中错误应用双buffer系数

严重等级: P1

缺陷描述: workspace大小计算时对每项无差别乘BUFFER_NUM(双buffer系数)，但部分buffer实际只需单份空间，导致workspace过大浪费内存或（更危险地）在修改后去掉系数时不足。

典型代码示例:

```cpp
// 缺陷 — e32a5eaae3
// MemFriendlyTilingFunc计算workspace
userWorkspaceByteSize += BT * H * IN_BYTE_SIZE * BUFFER_NUM;  // BT*H只需单份
userWorkspaceByteSize += V * H * IN_BYTE_SIZE * BUFFER_NUM;   // V*H只需单份
// 多分配了 (BT*H + V*H) * IN_BYTE_SIZE 的空间

// 修复 — 去掉不需要双buffer的项的BUFFER_NUM
userWorkspaceByteSize += BT * H * IN_BYTE_SIZE;  // 单份即可
userWorkspaceByteSize += V * H * IN_BYTE_SIZE;   // 单份即可
```

审查检查方法:
- workspace计算中每项乘BUFFER_NUM时逐项注释说明为什么需要双buffer
- 代码审查时逐行核对公式各项物理含义

关联commit: `e32a5eaae3`

---

#### 规则 TILE-03: tiling参数未区分normal核和tail核

严重等级: P0

缺陷描述: 多核tiling中normal核和tail核使用相同参数（如buffer大小），但tail核的数据量通常小于normal核。用normal核的参数分配tail核的workspace可能不足（tail核需要不同的内存布局），导致OOM。

典型代码示例:

```cpp
// 缺陷 — MaxPoolGradWithArgmaxV3 (参考b99dcb02c5类)
// CalcGradArgmaxInner对normal核和tail核使用相同highAxisInner
void CalcGradArgmaxInner(int highAxisInner) {
    bufferSize = highAxisInner * elementSize;  // tail核highAxisTail可能远小
}

// 修复 — 拆分为normal版和tail版
void CalcGradArgmaxInner(int highAxisInner) { ... }     // normal核
void CalcGradArgmaxInnerTail(int highAxisTail) { ... }   // tail核独立计算
```

审查检查方法:
- 多核tiling中normal核和tail核的buffer尺寸是否分别计算
- tiling key是否区分normal/tail模式

关联commit: `90649ac299`(MaxPoolGrad OOM fix)

---

#### 规则 TILE-04: TilingData结构体host-kernel不一致

严重等级: P0

缺陷描述: TilingData结构体的字段布局、类型在host和kernel端不完全一致，导致数据解析错位。字段重排、类型缩窄(uint64->uint32)、value字段冗余等都会引发精度错误。

典型代码示例:

```cpp
// 缺陷 — 0d32207040
// Host端TilingData
struct TilingData {
    int64_t scale;   // 应为float，int64_t导致精度丢失
    int64_t value;   // 冗余字段，kernel端不使用
};

// Kernel端期望
struct TilingData {
    float scale;     // 类型不匹配！
};
// 解析时字段偏移错位，所有后续字段读取值错误
```

审查检查方法:
- TilingData结构体定义在host和kernel端是否完全一致（字段数、字段类型、字段顺序）
- 建议使用共享头文件定义TilingData

关联commit: `0d32207040`, revert `74cdae15d`(NLLLossGrad重构因TilingData布局不匹配被回退)

---

### 类别四：INTERFACE_API 接口/API变更传播（44条，9.6%）

dev分支接口变更频繁，占比远高于主仓。子类型：API迁移不完整(~12)、接口签名变更未同步(~8)、废弃接口未清理(~6)、命名空间迁移(~5)、API误用(~5)、参数传递错误(~4)。

#### 规则 API-01: 修改Init签名/模板参数后调用点未全量同步

严重等级: P1

缺陷描述: 修改kernel类Init签名或模板参数后，未全局搜索所有调用点同步更新，导致部分调用点编译失败或静默传参错误。

典型代码示例:

```cpp
// 缺陷 — b14f5a03ca
// GENERAL_OP_IMPL宏调用op.Init()缺少workspace参数
GENERAL_OP_IMPL(op, OpType<T>);
// 宏展开后: op.Init(tilingData);
// 但Init签名已改为: Init(tilingData, workspace);  // 缺workspace

// 同时：模板类只传1个类型参数
OpType<float> op;
// 但OpType已改为: template<typename T, KernelMode M> class OpType;
// 缺少第二个kernel mode参数
```

审查检查方法:
- 修改Init签名/模板参数后，全局搜索所有调用点确认同步更新
- 宏展开后的实际调用是否与当前函数签名匹配

关联commit: `b14f5a03ca`, `22979c4afe`

---

#### 规则 API-02: TilingParse注册类型与实际CompileInfo类型不一致

严重等级: P1

缺陷描述: TilingParse<T>的模板参数T使用旧类型，但tiling实际依赖新类型，导致tiling数据解析失败。

典型代码示例:

```cpp
// 缺陷 — 368607bd
// TilingParse注册使用旧类型
REGISTER_TILING_PARSE(CTCLossV2GradCompileInfo, ParseFunc);
// 但tiling实际操作的是新类型
TilingPrepare<CTCLossV2GradForCompileInfo>(...);  // 类型不匹配

// 修复 — 统一为新类型
REGISTER_TILING_PARSE(CTCLossV2GradForCompileInfo, ParseFunc);
```

审查检查方法:
- TilingParse<T>的T与TilingPrepare操作的CompileInfo类型是否一致
- 重构后grep旧类型名确认无残留引用

关联commit: `368607bd`

---

### 类别五：CALC_LOGIC 计算逻辑错误（35条，7.7%）

算子核心计算逻辑中的数学/算法错误。子类型：tiling参数计算公式错误(~8)、维度/索引计算方向错误(~5)、数学公式实现错误(~4)、特殊值(NaN/Inf)处理(~3)、精度损失(~3)、多核初始化竞争(~3)、量化逻辑错误(~3)。

#### 规则 CALC-01: CeilDiv/CeilAlign结果缺少上界clamp

严重等级: P1

缺陷描述: tiling参数通过CeilDiv计算后可能超过实际shape维度值，未做`std::min(..., actualDim)`上界约束，导致L1 size校验产生错误判断或内存分配过大。

典型代码示例:

```cpp
// 缺陷 — a70d8b16d5
// 计算L1缓存中最小feature map高度
hoAL1min = CeilDiv(m0, wo);  // 当wo < m0时，结果可能超过实际ho
hiAL1min = (hoAL1min - 1) * strideH + dilateH * (kh - 1) + 1;
// hiAL1min超出实际输入高度，L1 size校验错误

// 修复 — 添加上界约束
hoAL1min = std::min(CeilDiv(m0, wo), ho);  // 不超过实际ho
```

审查检查方法:
- CeilDiv/CeilAlign结果是否需要与实际shape维度做min(上界clamp)
- 有符号整数做位运算前应校验非负

关联commit: `a70d8b16d5`

---

#### 规则 CALC-02: 多核场景全局内存初始化存在竞争

严重等级: P0

缺陷描述: 多个核独立对全局内存做零初始化存在竞争条件。跨循环累积计数器语义混淆（全局累积 vs 每次迭代独立）导致后续loop有效数据量不正确。

典型代码示例:

```cpp
// 缺陷 — fa03888b0d
// 空bag时每个核独立对全局内存做零初始化 — 竞争！
if (isEmpty) {
    for (int i = 0; i < size; i++) {
        globalMem[i] = 0;  // 多核同时写，结果不确定
    }
}

// 修复 — 指定核初始化 + 同步屏障
if (isEmpty && GetBlockIdx() == 0) {
    for (int i = 0; i < size; i++) {
        globalMem[i] = 0;
    }
}
SyncAll();  // 等所有核到达此点
```

审查检查方法:
- 多核场景全局内存写入是否由指定核(core 0)完成并加SyncAll
- 跨循环计数器确认是全局累积还是每次迭代独立

关联commit: `fa03888b0d`

---

#### 规则 CALC-03: splitK场景Bias加载时机错误

严重等级: P1

缺陷描述: splitK模式下Bias应在第一次K迭代(kIndex==0)加载参与累加，但错误地放在最后一次迭代加载，导致Bias与部分和的累加顺序错误。

典型代码示例:

```cpp
// 缺陷 — e7048cff4f
// Bias在最后一次迭代加载
if (kIndex == splitKRound - 1) {
    LoadBias();  // 错误：Bias应在首次累加时加入
}

// 修复 — Bias在第一次迭代加载
if (kIndex == 0) {
    LoadBias();  // 正确：首次累加时加入Bias
}
```

审查检查方法:
- splitK场景Bias/残差加载是否在kIndex==0
- 累加顺序是否符合数学定义

关联commit: `e7048cff4f`

---

### 类别六：COMPILE_WARN 编译告警/代码规范（34条，7.5%）

多为可通过编译器flag自动拦截的问题，但部分告警修复中隐含真实逻辑bug。

#### 规则 WARN-01: 删除"未使用变量"前检查是否原本应该被使用

严重等级: P2

缺陷描述: 删除编译器报告的"未使用变量"时，该变量可能原本应该被使用——真正的bug是上游代码copy-paste后忘记替换变量名。盲目删除变量会掩盖真实逻辑缺陷。

典型代码示例:

```cpp
// 缺陷 — d818b4d4
// 编译器报告sourceStorageFormat未使用
// 实际bug是条件判断copy-paste错误：
if (maskStorageFormat != FORMAT_ND || maskStorageFormat != FORMAT_ND) {
    //                                ^^^^^^^^^^^^
    //                                应为sourceStorageFormat, copy-paste重复了mask
}
// 该commit只删除了sourceStorageFormat变量来消除告警
// 但条件中的copy-paste错误并未修正，逻辑bug被掩盖
// 正确修复应为：
if (maskStorageFormat != FORMAT_ND || sourceStorageFormat != FORMAT_ND) {
    ...
}
```

审查检查方法:
- 删除"未使用变量"前检查该变量是否原本应该被使用
- 搜索同一代码块中是否有copy-paste导致的变量名重复
- 注意：d818b4d4中此bug实际未被修复，仅消除了告警

关联commit: `d818b4d4`

---

#### 规则 WARN-02: 链式比较`a < b < c`语义陷阱

严重等级: P1

缺陷描述: C++中`a < b < c`被解析为`(a < b) < c`即`bool < c`，不等同于数学含义的范围比较。

典型代码示例:

```cpp
// 缺陷 — 7ad3e89e
if (lower < value < upper) {
    // 编译器解析为: (lower < value) < upper
    // 即: bool_result < upper
    // 当upper > 1时永远为true！
}

// 修复
if (lower < value && value < upper) { ... }
```

审查检查方法:
- grep `< .* <` 模式检查链式比较
- 建议启用 -Wparentheses 编译选项

关联commit: `7ad3e89e`

---

### 类别七：CONDITION_LOGIC 条件判断/控制流（31条，6.8%）

条件分支遗漏、控制流路径错误、return缺失。子类型：分支遗漏(~10)、设置状态后缺少return(~4)、条件过宽/过严(~4)、循环终止条件错误(~3)、除零保护缺失(~3)、逻辑运算符优先级(~3)。

#### 规则 COND-01: 设置特殊状态后缺少return

严重等级: P0

缺陷描述: SetUnknownRank/SetUnknownShape后没有return语句，代码继续执行后续shape推导逻辑，对UnknownRank的shape调用GetDimNum可能返回异常值导致越界访问。

典型代码示例:

```cpp
// 缺陷 — 2d424df8e6
if (condition) {
    outputDesc->SetUnknownRank();
    // 缺少 return ge::GRAPH_SUCCESS;
    // 代码继续往下执行！
}
// 后续代码对UnknownRank shape调用GetDimNum -> 异常值 -> 越界
int dimNum = outputDesc->GetShape().GetDimNum();
for (int i = 0; i < dimNum; i++) { ... }  // dimNum可能是垃圾值

// 修复
if (condition) {
    outputDesc->SetUnknownRank();
    return ge::GRAPH_SUCCESS;  // 必须return
}
```

审查检查方法:
- 设置特殊状态(UnknownRank/UnknownShape)的分支必须包含return
- InferShape函数应做静态分析确保所有路径正确返回

关联commit: `2d424df8e6`

---

#### 规则 COND-02: 除法运算的除数缺少非零保护

严重等级: P1

缺陷描述: 除法运算的除数来自外部参数或计算结果，未做非零保护，特定输入下触发除零异常。

典型代码示例:

```cpp
// 缺陷 — 22f84c0e
result = totalSize / blockDim;  // blockDim可能为0

// 修复
if (blockDim == 0) {
    return ERROR_INVALID_PARAM;
}
result = totalSize / blockDim;
```

审查检查方法:
- 除法运算的除数是否有非零保护
- CeilDiv/整数除法的分母是否可能为0

关联commit: `22f84c0e`, hotspot `quant_batch_matmul_v3_tiling.cpp`

---

#### 规则 COND-03: 宏定义中隐藏return语句

严重等级: P2

缺陷描述: 宏中使用return语句隐藏控制流，调用者不知道宏可能提前退出函数，导致不同分支行为不一致。

典型代码示例:

```cpp
// 缺陷 — 6beab61dd0
#define INVOKE_FLAT_QUANT_IMPL(impl) \
    do { \
        auto ret = impl.Process(); \
        return ret;  // 隐藏的return！
    } while(0)

// 调用侧
TILING_KEY_IS(3) { INVOKE_FLAT_QUANT_IMPL(implA); }
TILING_KEY_IS(5) { INVOKE_FLAT_QUANT_IMPL(implB); }  // 提前退出
// 后续代码不会执行，但读者可能不知道

// 修复 — 内联展开消除宏中的return
```

审查检查方法:
- 宏定义中是否包含return语句
- 是否有更清晰的替代方案（内联函数、直接展开）

关联commit: `6beab61dd0`

---

### 类别八：PLATFORM 平台/SoC适配（29条，6.4%）

多平台支持(910/910_93/910_95/310P/5102/3510)下的适配遗漏。

#### 规则 PLAT-01: `__CCE_AICORE__ == xxx`精确匹配应改为范围比较

严重等级: P1

缺陷描述: 平台版本判断使用精确匹配`==`，新平台上线时指令集是向下兼容的，但精确匹配导致新平台不匹配任何分支走入default错误路径。

典型代码示例:

```cpp
// 缺陷 — 4f87bea9ab
#if __CCE_AICORE__ == 220
    // 仅匹配220，230/240等新平台被排除
    UseAdvancedInstruction();
#else
    UseFallback();
#endif

// 修复 — 范围比较
#if __CCE_AICORE__ >= 220
    UseAdvancedInstruction();  // 所有220+平台都支持
#else
    UseFallback();
#endif
```

审查检查方法:
- `__CCE_AICORE__ == xxx`和`__NPU_ARCH__ == xxx`精确匹配是否应改为`>=`
- 新增平台时有checklist检查所有相关算子的条件分支

关联commit: `4f87bea9ab`

---

#### 规则 PLAT-02: compute_units列表缺少目标平台

严重等级: P1

缺陷描述: ascendc_config.json中算子的compute_units列表缺少某个目标平台，该平台二进制不被预编译，运行时触发JIT异常。

典型代码示例:

```json
// 缺陷 — db31f7273 (TransQuantParamV2)
{
    "op_name": "TransQuantParamV2",
    "compute_units": ["ascend910b", "ascend910_93"]
    // 缺少 "ascend310p" -> 310P平台无预编译kernel
}

// 修复 — 补充平台
{
    "compute_units": ["ascend910b", "ascend910_93", "ascend310p"]
}
```

审查检查方法:
- 对照平台支持矩阵逐一确认compute_units覆盖所有目标平台
- 新增平台时全局搜索ascendc_config.json中遗漏的算子

关联commit: `db31f7273`(TransQuantParamV2 310p二进制异常)

---

### 类别九：COPY_PASTE 复制粘贴错误（24条，5.3%）

高度可审查的机械性错误。子类型：变量名混淆(~8)、日志引用错误变量(~5)、参数位置重复(~4)、拼写错误(~4)、示例代码错误(~3)。

#### 规则 COPY-01: 错误日志引用的变量与判断条件不一致

严重等级: P2

缺陷描述: CHECK/LOG调用中，报错信息引用的变量与实际校验的变量不一致，导致日志误导调试。根因是copy-paste后修改了条件但忘记修改日志中的变量名。

典型代码示例:

```cpp
// 缺陷 — d1832c87c9
// 检查scale的维度
if (scaleShape->GetStorageShape().GetDim(0) != expectedDim) {
    // 报错时误用offset的shape！
    LOG("dim mismatch: %d", offsetShape->GetStorageShape().GetDim(0));
    //                      ^^^^^^^^^^^^ 应为scaleShape
}

// 修复 — 统一为scaleShape
LOG("dim mismatch: %d", scaleShape->GetStorageShape().GetDim(0));
```

审查检查方法:
- 错误日志中引用的变量是否与上文条件判断中的变量一致
- 连续相似CHECK/LOG调用中，逐行对比检查对象

关联commit: `d1832c87c9`, hotspot `quant_batch_matmul_v4_tiling.cpp`

---

#### 规则 COPY-02: 函数调用中同类型参数重复

严重等级: P1

缺陷描述: 函数调用中多个同类型参数(batch1/batch2, src/dst)出现重复，第二个参数错误地与第一个相同。导致只检查了一个输入而忽略另一个。

典型代码示例:

```cpp
// 缺陷 — dec58d1b24
// aclnnAddbmmGetWorkspaceSize中
isAddBmmProcessEmptyTensor(batch1, batch1);
//                                 ^^^^^^ 应为batch2

// aclnnBaddbmmGetWorkspaceSize中同样错误
// 导致只检查batch1是否为空tensor而忽略batch2

// 修复
isAddBmmProcessEmptyTensor(batch1, batch2);
```

审查检查方法:
- 函数调用中有多个同类型参数时检查是否存在重复
- 对称操作(batch1/batch2, input/output, src/dst)的参数位置是否正确

关联commit: `dec58d1b24`

---

### 类别十：CONFIG_MISSING 配置遗漏（22条，4.8%）

算子注册/编译/运行时配置文件的条目遗漏或参数错误。

#### 规则 CONF-01: 平台特定编译选项遗漏

严重等级: P1

缺陷描述: 特定平台(如ascend910_95)需要额外编译选项(如`-mllvm -cce-aicore-dcci-before-kernel-end=false`)，但大量算子配置中遗漏该选项，导致kernel在目标平台运行异常。

典型代码示例:

```json
// 缺陷 — ac64e3d38b
// 约20个算子在ascend910_95平台缺少dcci编译选项
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

#### 规则 CONF-02: output paramType(REQUIRED/OPTIONAL)配置错误

严重等级: P1

缺陷描述: def.cpp中output的paramType标记(REQUIRED/OPTIONAL)与binary.json不一致，或与算子实际行为矛盾（实际为可选输出但标记为REQUIRED），导致框架层校验逻辑异常。

典型代码示例:

```cpp
// 缺陷 — 9c46dc5f19
// 910_95平台配置中rstd和x输出被标记为REQUIRED
.OUTPUT(rstd, TensorType({DT_FLOAT}))     // 标记为REQUIRED
.OUTPUT(x, TensorType({DT_FLOAT16}))      // 标记为REQUIRED
// 但实际这两个输出是可选的，某些模式下不产出

// 修复 — 改为OPTIONAL
.OPTIONAL_OUTPUT(rstd, TensorType({DT_FLOAT}))
.OPTIONAL_OUTPUT(x, TensorType({DT_FLOAT16}))
```

审查检查方法:
- def.cpp的paramType与binary.json是否双向一致
- REQUIRED输出是否在所有代码路径中都有值

关联commit: `9c46dc5f19`

---

### 类别十一：INT_OVERFLOW 整数溢出/类型截断（18条，3.9%）

频次不是最高，但后果严重（OOM、越界访问、精度错误），且当前代码中仍有12处残留。

#### 规则 INT-01: shape维度相乘强制使用int64_t

严重等级: P0

缺陷描述: shape维度相乘的表达式使用int32，大shape下乘积超过INT32_MAX溢出。典型场景：排序索引硬编码DT_INT32、buffer偏移计算、batch维度累乘。

典型代码示例:

```cpp
// 缺陷 — fa1305c56e
// 排序索引硬编码为DT_INT32
auto sortIndicesType = DT_INT32;  // 大shape下索引溢出

// 修复 — 动态选择类型
auto sortIndicesType = GetSortIndicesType(shape);  // 根据shape选int32/int64
ge::DataType GetSortIndicesType(const Shape& shape) {
    return (shape.GetDim(-1) > INT32_MAX) ? DT_INT64 : DT_INT32;
}
```

```cpp
// 缺陷 — e0ddd962
// gmOffset_计算，两个int32相乘
int32_t offset = addUbSize.GetValue(8) * shapeDim_;  // 溢出！

// 修复 — 显式类型提升
uint64_t offset = static_cast<uint64_t>(addUbSize.GetValue(8)) * shapeDim_;
```

审查检查方法:
- shape维度相乘表达式是否使用int64_t
- GetValue()返回值参与乘法时，操作数是否显式cast为64位

关联commit: `fa1305c56e`, `e0ddd962`, `6ac2bba4`

---

#### 规则 INT-02: 硬件指令参数位宽限制校验缺失

严重等级: P0

缺陷描述: 硬件指令的参数有位宽限制（如load3d的kStartPt字段上限65535/16位），超限时溢出翻转指向错误内存位置。这类缺陷可审查性低，需要硬件手册知识。

典型代码示例:

```cpp
// 缺陷 — 952313ace0
// load3d指令kStartPt字段为16位，上限65535
int kStartPt = k0 * hk * wk;  // 超过65536时溢出翻转！

// 修复 — 添加边界校验
const int LOAD3D_KSTART_MAX = 65535;
if (load3dK > LOAD3D_KSTART_MAX) {
    // 走超大kernel切分路径
    UseLargeKernelSplit();
}
```

审查检查方法:
- 涉及硬件指令参数时确认所有字段位宽限制
- tiling代码中对硬件参数做边界校验

关联commit: `952313ace0`

---

#### 规则 INT-03: uint64_t减法下溢

严重等级: P0

缺陷描述: size_t/uint64_t做减法时被减数小于减数，结果下溢为极大正整数，后续用于内存分配或循环控制导致OOM或越界。

典型代码示例:

```cpp
// 缺陷 — hotspot group_norm_silu_tiling.cpp
uint64_t remainUbSize = totalUbSize - usedSize;
// usedSize > totalUbSize时下溢为巨大正整数
// 导致maxProcessSize过大，UB溢出

// 修复 — 减法前检查
if (totalUbSize <= usedSize) {
    return ERROR_INSUFFICIENT_UB;
}
uint64_t remainUbSize = totalUbSize - usedSize;
```

```cpp
// 缺陷 — hotspot aclnn_matmul.cpp
size_t loopDims = dimNum - 2;  // dimNum < 2时下溢为极大值

// 修复
if (dimNum < 2) { return ERROR; }
size_t loopDims = dimNum - 2;
```

审查检查方法:
- size_t/uint64_t做减法前是否判断被减数>=减数
- CeilAlign/CeilDiv返回值参与二次乘法时结果类型是否足够宽

关联commit: hotspot `group_norm_silu_tiling.cpp`, `aclnn_matmul.cpp`

---

### 类别十二：PIPELINE_SYNC 流水线同步/硬件事件（16条，3.5%）

NPU硬件同步原语使用错误，后果为精度错误或硬件异常。

#### 规则 SYNC-01: SetFlag/WaitFlag必须成对出现且无条件执行

严重等级: P0

缺陷描述: WaitFlag被条件化（if ni>0等），导致某些迭代路径跳过必要的同步等待，DMA写出未完成就被覆盖。SetFlag后必须有对应WaitFlag，且不能被任何条件分支跳过。

典型代码示例:

```cpp
// 缺陷 — a79b527fe8
// WaitFlag被条件化
if (ni > 0) {
    WaitFlag<MTE3_MTE2>(eventId);  // ni==0时跳过等待！
}
SetFlag<MTE3_MTE2>(eventId);       // 但SetFlag无条件执行
// 不平衡导致硬件事件状态错误

// 修复 — 无条件执行WaitFlag
WaitFlag<MTE3_MTE2>(eventId);  // 始终等待
SetFlag<MTE3_MTE2>(eventId);
```

审查检查方法:
- SetFlag和WaitFlag是否成对出现且无条件执行
- 不应在循环边界条件中跳过同步事件

关联commit: `a79b527fe8`

---

#### 规则 SYNC-02: CopyOut/CopyIn前的同步事件与实际DMA通道匹配

严重等级: P0

缺陷描述: CopyOut走MTE3通道，但前置同步使用了S_MTE2事件（搬入通道），导致同步了错误的通道，CopyOut可能在Vector计算未完成时执行。

典型代码示例:

```cpp
// 缺陷 — 51f8247aee
// CopyOut前使用MTE2同步 — 错误通道！
WaitFlag<S_MTE2>(eventId);
CopyOut(dst, src, count);  // CopyOut走MTE3通道

// 修复 — 使用正确的MTE3通道 + V_MTE3双重同步
WaitFlag<S_MTE3>(eventId);
WaitFlag<V_MTE3>(eventId2);  // 确保Vector计算完成
CopyOut(dst, src, count);
```

审查检查方法:
- CopyOut前的同步是否用MTE3通道（不是MTE2）
- CopyIn前的同步是否用MTE2通道
- Vector计算后紧跟CopyOut需等待V_MTE3

关联commit: `51f8247aee`

---

#### 规则 SYNC-03: 连续matmul操作间必须插入PipeBarrier

严重等级: P1

缺陷描述: 两次matmul操作间缺少PipeBarrier<PIPE_ALL>()同步，matmulR结果可能尚未完成matmulL就读取，导致使用未完成的计算结果。

典型代码示例:

```cpp
// 缺陷 — 6beab61dd0
matmulL.Iterate();
// 缺少 PipeBarrier<PIPE_ALL>();
matmulR.Iterate();  // 可能读取matmulL的不完整结果

// 修复
matmulL.Iterate();
PipeBarrier<PIPE_ALL>();  // 确保matmulL完成
matmulR.Iterate();
```

审查检查方法:
- 连续matmul操作间是否插入PipeBarrier
- 同一kernel中eventID是否有冲突

关联commit: `6beab61dd0`

---

### 类别十三：LOG_ERROR 日志/错误处理（11条，2.4%）

#### 规则 LOG-01: 错误码区分PARAM_*和INNER_*

严重等级: P3

缺陷描述: 对用户传入参数的校验失败使用INNER_*错误码（内部错误），应使用PARAM_*错误码。错误码语义不正确会误导问题定位。

典型代码示例:

```cpp
// 缺陷 — 797589989a
// 用户参数校验使用内部错误码
CHECK_NOT_NULL(userInput, ACLNN_ERR_INNER_NULLPTR);
// 应为用户参数错误码
CHECK_NOT_NULL(userInput, ACLNN_ERR_PARAM_NULLPTR);
```

审查检查方法:
- 错误码区分PARAM_*(用户输入)和INNER_*(内部不可达状态)
- 错误日志中条件描述是否与实际判断条件一致

关联commit: `797589989a`, `dc3a8850e7`

---

### 类别十四：DATA_TYPE 数据类型（10条，2.2%）

#### 规则 DTYPE-01: 新增数据类型支持时三处同步更新

严重等级: P1

缺陷描述: 新增数据类型支持时，tiling层dtype校验、kernel层预编译条件、binary配置三处必须同步更新，遗漏任一处导致新类型走入错误分支或不被编译。

典型代码示例:

```cpp
// 缺陷 — 9150a7cca8
// tiling层新增了int8支持
if (dtype == DT_INT8) { ... }  // tiling已更新

// 但kernel层预编译条件未更新
#if TILING_KEY_IS(1)  // 仅float/float16
    ...
// int8无对应kernel，运行时找不到实现

// binary配置也未更新
// ascendc_config.json缺少int8变体
```

审查检查方法:
- 新增数据类型时同步检查tiling层、kernel层、binary配置三处
- 选择计算模板/优化路径时检查是否对所有支持的数据类型兼容

关联commit: `9150a7cca8`, `84ab237ad3`

---

### 类别十五：RESOURCE 资源管理/状态初始化（8条，1.8%）

#### 规则 RES-01: 禁止在op_host/*.cpp中使用非const全局容器

严重等级: P0

缺陷描述: 在.cpp文件作用域定义全局std::vector等容器，多实例并发调用时共享导致数据竞争和AIC错误。

典型代码示例:

```cpp
// 缺陷 — 2a15cd4d
// index_tiling_no_continuous.cpp 全局作用域
std::vector<gert::Stride> indexstrideList;  // 全局！并发不安全

void CalcTiling() {
    indexstrideList.push_back(...);  // 多实例并发写 -> AIC错误
}

// 修复 — 移入类成员
class IndexNonContinuousTiling {
private:
    std::vector<gert::Stride> indexstrideList_;  // 每个实例独立
};
```

审查检查方法:
- op_host/*.cpp中是否有非const全局变量/容器声明
- tiling相关状态必须封装在类成员中

关联commit: `2a15cd4d`, revert `2b00825d3`(addRmsNorm因全局变量被回退)

---

#### 规则 RES-02: L1全载优化在循环中的状态失效检查

严重等级: P1

缺陷描述: L1/L0缓存全载优化在多batch/多group循环中，当参数（如B矩阵偏移）变化时未切换全载状态，之前全载加载的数据已失效但仍被使用。

典型代码示例:

```cpp
// 缺陷 — 46f5817ef1
// B矩阵偏移变化但全载状态未切换
for (int batch = 0; batch < batchNum; batch++) {
    if (enableFullLoad) {
        UseL1CachedData();  // B偏移已变，L1数据失效！
    }
}

// 修复 — 跟踪偏移变化并重置全载状态
if (curOffsetB != preOffsetB_) {
    enableFullLoad = false;   // 偏移变化，关闭全载
    ReleaseL1Buffer();
    preOffsetB_ = curOffsetB;
}
```

审查检查方法:
- L1全载优化在循环中，检查全载条件是否因参数变化而失效
- 状态变量(enableFullLoad)是否在循环迭代间正确更新

关联commit: `46f5817ef1`

---

### 类别十六：UT_TEST 单元测试（8条，1.8%）

#### 规则 UT-01: 禁止注释EXPECT_*/ASSERT_*断言

严重等级: P3

缺陷描述: 大量EXPECT_EQ断言被批量注释掉，UT失去验证能力沦为编译检查。

典型代码示例:

```cpp
// 缺陷 — 91c29129
// EXPECT_EQ(result, expected);    // 被注释！
// EXPECT_EQ(shape[0], 128);       // 被注释！
// EXPECT_EQ(shape[1], 64);        // 被注释！
// UT运行全通过但实际什么都没验证
```

审查检查方法:
- grep检测`// EXPECT_`和`// ASSERT_`模式
- UT中不同阶段的推理应使用独立context避免状态污染
- tiling参数应由tiling函数自动生成而非手工硬编码

关联commit: `91c29129`, `ee685a90`, `9a18f2ca`

---

### 类别十七：NULLPTR 空指针解引用（6条，1.3%）

#### 规则 NULL-01: 可选输出tensor(outputMask控制)访问前必须检查null

严重等级: P0

缺陷描述: 反向传播中dx/dw/dbias由outputMask控制是否计算，为null时仍被解引用取shape或dtype导致crash。

典型代码示例:

```cpp
// 缺陷 — e49d5d27c0
// 只需计算dw不需dx时gradInput为nullptr
auto mmDwOutShape2d = CalcShape(gradInput->GetShape());  // crash!

// 修复 — 改用非空tensor的shape
auto mmDwOutShape2d = CalcShape(gradWeight->GetShape());  // gradWeight非空
```

```cpp
// 缺陷 — 59a561e5
// outputMask可能指示gradInput不需要计算(为null)
PreConv1DBackwardTo2D(gradInput, ...);  // 无条件调用 -> crash

// 修复 — 用outputMask守护
if (outputMask[0]) {
    PreConv1DBackwardTo2D(gradInput, ...);
}
```

审查检查方法:
- 反向传播中dx/dw/dbias解引用前需null检查或对应mask位检查
- dtype兼容性判断优先使用必选张量(output)而非可选张量

关联commit: `e49d5d27c0`, `59a561e5`, `2463697d99`

---

### 类别十八：REVERT 性能/功能回退（4条，0.9%）

#### 规则 REV-01: 大规模重构应分步提交

严重等级: P2

缺陷描述: 大规模算子重构（同时修改TilingData布局、kernel模板化、分核策略）一次性合入，host与device端数据解析不匹配导致全面回退。拆分为多个小提交可逐步验证。

典型代码示例:

```
# 缺陷 — 74cdae15d7 (NLLLossGrad重构被回退)
# 原始提交同时做了3件事：
# 1. TilingData字段重排 + 类型缩窄(uint64->uint32)
# 2. kernel模板化拆分
# 3. 分核策略变更
# host与device端TilingData解析不匹配 -> 全面回退

# 应拆分为：
# PR1: TilingData字段变更(host+kernel同步)
# PR2: kernel模板化(不改数据结构)
# PR3: 分核策略变更
```

审查检查方法:
- 单次合入超过3000行的PR应要求拆分
- 大规模重构不应同时改变数据结构布局和计算逻辑

关联commit: revert `74cdae15d7`, revert `fe16c557`

---

#### 规则 REV-02: 移除binary.json算子变体前确认无下游调用

严重等级: P1

缺陷描述: 过早移除binary.json中的算子变体或编译配置，但下游场景仍有调用，运行时找不到对应kernel。

典型代码示例:

```json
// 缺陷 — e682154870
// 移除了MatMulV3的bf16 bias编译配置变体
// 但下游仍有bf16 bias场景需要此变体
// 运行时：kernel not found for bf16 bias config
```

审查检查方法:
- 移除binary.json变体前grep所有下游场景确认无调用
- 强制禁用已有模板(`return false`)必须注释原因

关联commit: `e682154870`

---

### 跨类别系统性风险

#### SYS-01: matmul是绝对缺陷震中

热点Top 30文件中matmul占20席。Revert事件中matmul系列占45%(9/20)。量化矩阵乘(quant_batch_matmul_v3/v4, weight_quant_batch_matmul_v2)尤为集中。当前热点文件仍存在多处严重风险：

- `aclnn_batch_matmul.cpp`(9次触及): 空tensor分支死代码
- `matmul_v3_base_tiling.cpp`(8次触及): 空数组越界 + 日志字段不匹配
- `matmul_util.cpp`(8次触及): std::ceil对整数除法无效 + 索引越界
- `quant_batch_matmul_v4_tiling.cpp`(7次触及): 日志打印错误dtype + batch累乘溢出
- `weight_quant_batch_matmul_v2_tiling.cpp`(6次触及): uint64->int64转换溢出 + batch累乘溢出

审查建议: matmul相关PR需要额外一轮专项审查。

证据: revert `9b3d2d76b`, `2e1ef4b41`, `e1fdbe6e8`, `ac976d4b5`, qbmm事件(3次循环revert)

---

#### SYS-02: 整数溢出是最普遍残留风险

当前代码中仍存在11处整数溢出/精度损失风险，横跨matmul/conv/quant/norm四个模块。根因：int32类型的tiling参数相乘、batch维度累乘、buffer偏移计算缺少溢出保护，以及int64_t转float精度丢失。

具体残留位置:
- `matmul_util.cpp`: std::ceil(mDim/align128)整数除法截断(mDim和align128均为uint64_t，整除后ceil无效)
- `quant_batch_matmul_v3_tiling.cpp`: baseM*baseK*dtypeSize三个int32相乘
- `quant_batch_matmul_v4_tiling.cpp`: batch累乘无保护
- `weight_quant_batch_matmul_v2_tiling.cpp`: uint64->int64转换 + batch累乘
- `flat_quant_cube.h`: shape.K*shape.M结果转float(23位尾数)精度丢失，大值场景除法结果不精确
- `group_norm_silu_tiling.cpp`: Lcm中a*b溢出 + uint64_t下溢
- `aclnn_scatter.cpp`: int64->int32窄化截断
- `aclnn_matmul.cpp`: size_t减法下溢 + 维度乘积溢出
- `adaptive_max_pool3d_big_pool.h`: startIndex乘法溢出
- `dequant_swiglu_quant_tiling_base.cpp`: uint32_t乘法溢出
- `aclnn_convolution.cpp`: 负维度乘uint64_t

审查建议: 编写lint规则，强制shape维度算术运算使用int64_t。

---

#### SYS-03: 提交粒度过大导致质量失控

- qbmm事件：6.5万行一次性合入导致3次循环revert
- weight_quant事件：5.7万行合入修改了公共error_util.h导致其他算子编译失败
- layernormv4：3000行功能补齐被回退
- 70%的revert在3天内发生，说明合入前review不充分

审查建议: 单次合入超过3000行的PR必须拆分。

证据: revert `79623db1a`(qbmm), `9de027499`(weight_quant), `fe16c5570`(layernormv4)

---

#### SYS-04: 公共基础设施耦合在算子PR中

5次revert源于公共头文件(op_util.h, error_util.h, op_api_def.h)和cmake修改耦合在算子PR中。公共修改影响面远超单算子，但review时容易被算子逻辑掩盖。

典型事件:
- qbmm合入修改了公共op_util.h和matmul_common_infershape.cpp
- weight_quant合入修改了公共error_util.h(CUBE_INNER_ERR_REPORT宏格式)
- identity迁移修改了公共cmake/symbol.cmake
- 枚举值插入修改了公共op_api_def.h(USE_HIGH_PREC_MODE从4改为5)

审查建议: 公共文件修改必须独立PR先行验证。

证据: revert `79623db1a`, `9de027499`, `61a1f1583`, `e1fdbe6e8`

---

#### SYS-05: foreach模块高触及但低风险

聚合触及次数最高(1452次)，但本质是批量模板代码同步修改。单个算子结构性风险低，真正需重点review的是matmul和quant的kernel/tiling层。不应被热点文件列表误导分配不当的审查资源。

---

### 审查规则优先级矩阵

#### P0 — 必须检查（高频高危，可直接拦截严重缺陷）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| INT-01 | shape维度相乘/buffer偏移计算强制使用int64_t | INT_OVERFLOW | fa1305c56e, e0ddd962 |
| TILE-01 | tiling侧buffer计算必须与kernel侧InitBuffer做交叉比对 | TILING_BUFFER | e25e6e60c7 |
| BOUND-02 | 多核任务分配区分tail core和idle core | BOUNDARY | e2fc1fecb1 |
| SYNC-01 | SetFlag/WaitFlag必须成对出现且无条件执行 | PIPELINE_SYNC | a79b527fe8 |
| COND-01 | 设置特殊状态(UnknownRank)后必须包含return | CONDITION_LOGIC | 2d424df8e6 |
| TILE-03 | tiling参数区分normal核和tail核 | TILING_BUFFER | 90649ac299 |
| TILE-04 | TilingData结构体host-kernel完全一致 | TILING_BUFFER | 0d32207040 |
| SYNC-02 | CopyOut/CopyIn前同步事件与DMA通道匹配 | PIPELINE_SYNC | 51f8247aee |
| NULL-01 | 可选输出tensor访问前必须检查null | NULLPTR | 59a561e5, e49d5d27c0 |
| CALC-02 | 多核场景全局内存初始化由指定核完成+同步屏障 | CALC_LOGIC | fa03888b0d |

#### P1 — 重点检查（中频高危或高频中危）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| API-01 | 修改Init签名/模板参数后全局搜索调用点同步更新 | INTERFACE_API | b14f5a03ca |
| BUILD-01 | 新增算子checklist: ascendc_config.json + binary.json + CMake | BUILD_CMAKE | c00e940bfa |
| COPY-02 | 函数调用中同类型参数(batch1/batch2)是否重复 | COPY_PASTE | dec58d1b24 |
| PLAT-01 | `__CCE_AICORE__==xxx`精确匹配改为范围比较 | PLATFORM | 4f87bea9ab |
| COND-02 | 除法运算除数非零保护 | CONDITION_LOGIC | 22f84c0e |
| DTYPE-01 | 新增数据类型时同步tiling/kernel/binary三处 | DATA_TYPE | 9150a7cca8 |
| RES-01 | 禁止op_host中非const全局容器 | RESOURCE | 2a15cd4d |
| INT-02 | 硬件指令参数位宽限制校验 | INT_OVERFLOW | 952313ace0 |
| CONF-01 | 平台特定编译选项(dcci等)完整性 | CONFIG_MISSING | ac64e3d38b |
| SYNC-03 | 连续matmul操作间插入PipeBarrier | PIPELINE_SYNC | 6beab61dd0 |

#### P2 — 常规检查（中危或可部分自动化）

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| SYS-04 | 公共文件修改必须独立PR | 跨类别 | revert分析 |
| SYS-03 | 单次合入超过3000行必须拆分 | 跨类别 | qbmm事件 |
| WARN-01 | 删除"未使用变量"前检查是否原应被使用 | COMPILE_WARN | d818b4d4 |
| CONF-02 | 同族算子配置一致性对比 | CONFIG_MISSING | 9c46dc5f19 |
| LOG-01 | 错误码区分PARAM_*和INNER_* | LOG_ERROR | 797589989a |
| UT-01 | 禁止注释EXPECT_*/ASSERT_*断言 | UT_TEST | 91c29129 |
| COPY-01 | 日志引用变量与判断条件一致 | COPY_PASTE | d1832c87c9 |
| REV-01 | 大规模重构分步提交 | REVERT | 74cdae15d7 |

#### P3 — 建议自动化（可通过工具/CI拦截）

| 规则ID | 规则 | 实现方式 |
|--------|------|---------|
| AUTO-01 | 启用编译器flags: -Werror -Wshadow -Wconversion -Wformat -Wsign-compare -Wparentheses | CMake配置 |
| AUTO-02 | shape维度算术运算lint规则(检测int32乘法) | 自定义lint |
| AUTO-03 | ascendc_config.json与算子目录交叉校验 | CI脚本 |
| AUTO-04 | SetFlag/WaitFlag配对检测 | 静态分析 |
| AUTO-05 | "// EXPECT_"注释断言检测 | grep规则 |
| AUTO-06 | compile_options平台差异检测 | CI脚本 |

---

### 附录：与ops-nn主仓缺陷模式对比

ops-nn-dev (612缺陷/2571提交=23.8%) vs ops-nn (380缺陷/1474提交=25.8%)

相同点：
- 整数溢出、tiling/buffer计算、边界条件缺失、平台适配是两者共有的高频高危类别
- matmul模块在两个仓库都是最高危模块
- 可审查性均在99%以上

差异点：
- ops-nn-dev的BUILD_CMAKE占比(14.3%)远高于ops-nn(~5%)，dev分支构建基础设施更不稳定
- ops-nn-dev的INTERFACE_API占比(9.6%)高于ops-nn(~4%)，dev分支接口变更更频繁
- ops-nn-dev有更多UT_TEST类缺陷，dev分支测试基础设施迁移频繁
- ops-nn-dev的Revert事件(20条)显著多于主仓，70%在3天内回退，说明dev分支CI门禁更宽松

### 附录：Revert事件摘要（17个独立事件）

| 日期 | 事件 | 模块 | 根因 | 存活时间 |
|------|------|------|------|---------|
| 2026-02-05 | mmv3小case优化 | matmul | workspace脏数据 | 5个月 |
| 2026-01-08 | NLLLossGrad重构 | loss | TilingData布局不匹配 | 1天 |
| 2026-01-06 | conv C04 innerbatch | conv | L1对齐计算顺序错误 | 当天 |
| 2025-12-22 | baddbmm精度修复 | matmul | 多平台验证不足 | 3天 |
| 2025-12-10 | layernormv4功能补齐 | norm | 一次性禁用已有模板 | 1天 |
| 2025-12-08 | dlopen legacy ophost | matmul | 基础设施重构兼容性 | 2天 |
| 2025-12-04 | matmulv3分组累加 | matmul | 枚举值插入破坏ABI | 1天 |
| 2025-12-02 | addRmsNorm | norm | 全局变量+结构体布局 | 当天 |
| 2025-11-26 | ascendc_assert整改 | matmul | 函数->宏语义不等价 | 1天 |
| 2025-11-21 | transdata shape修改 | matmul | 10%shape性能劣化 | 3天 |
| 2025-11-19 | gather_v2 l0 | index | OP_ATTR变更不兼容 | 20天 |
| 2025-11-13 | addRmsNormDynamicQuant | norm | binary配置不匹配 | 1天 |
| 2025-09-30 | mmv3/bmmv3 tilingkey | matmul | tiling-kernel符号不同步 | 5天 |
| 2025-09-27 | identity迁移 | control | 公共cmake修改影响面 | 1天 |
| 2025-09-12~23 | qbmm三次循环 | matmul | 6.5万行+公共代码耦合 | 3天~11天 |
| 2025-08-26 | weight_quant | matmul | 5.7万行+error_util.h | <16小时 |
