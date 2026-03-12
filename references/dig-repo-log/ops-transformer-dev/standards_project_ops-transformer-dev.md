# ops-transformer-dev 项目缺陷模式与审查规则

基于2822条提交(2820条非merge)的全量分析，确认788条缺陷提交(27.9%)，其中528条代码缺陷经阶段2逐条diff分析。提炼15个缺陷类别和4个跨类别系统性风险，产出42条审查规则。

本仓库是ops-transformer的开发分支，提交历史与主仓完全独立(零共享hash)。缺陷类别从数据中自然涌现，未套用主仓框架。与主仓(243条缺陷/11类)相比，本仓缺陷规模更大(528条/15类)，新增了TKEY(tilingKey一致性)、PLAT(硬件平台兼容)、REVERT(功能回退)三个主仓不显著的类别，反映dev分支迭代速度更快、跨平台适配更密集的特征。

### 严重等级定义

| 等级 | 含义 | 影响范围 |
|------|------|----------|
| P0 | 致命 | CoreDump、数据损坏、内存越界、死锁 |
| P1 | 严重 | 特定配置崩溃、静默精度错误、功能不可用 |
| P2 | 一般 | 边界条件异常、构建失败、测试不通过 |
| P3 | 建议 | 代码质量、可维护性、潜在隐患 |

### 缺陷分布总览

| 序号 | 缺陷类别 | 频次 | 占比 | 严重度 | 规则ID前缀 |
|------|---------|------|------|--------|-----------|
| 1 | 条件分支/校验逻辑 | ~75 | 14.2% | P0-P1 | COND |
| 2 | CMake/构建配置 | ~65 | 12.3% | P1-P2 | BUILD |
| 3 | Buffer/内存/地址 | ~60 | 11.4% | P0-P1 | BUF |
| 4 | 同步/流水线时序 | ~35 | 6.6% | P0 | SYNC |
| 5 | 初始化/赋值遗漏 | ~30 | 5.7% | P1 | INIT |
| 6 | 整数溢出/类型转换 | ~30 | 5.7% | P0-P1 | OVFL |
| 7 | Shape/Layout处理 | ~28 | 5.3% | P1 | SHAPE |
| 8 | 接口/返回值 | ~25 | 4.7% | P0-P1 | API |
| 9 | 参数值/顺序/魔数 | ~25 | 4.7% | P1-P2 | PARAM |
| 10 | 硬件平台兼容性 | ~25 | 4.7% | P1-P2 | PLAT |
| 11 | TilingKey一致性 | ~20 | 3.8% | P0-P1 | TKEY |
| 12 | Tiling参数/计算 | ~20 | 3.8% | P1 | TPARAM |
| 13 | 对齐计算 | ~18 | 3.4% | P1 | ALIGN |
| 14 | 复制粘贴/拼写 | ~18 | 3.4% | P1-P2 | CPASTE |
| 15 | 功能回退(Revert) | ~16 | 3.0% | - | REVERT |
| - | 跨类别系统性风险 | - | - | P0-P1 | SYS |

---

## 类别1: 条件分支/校验逻辑 (COND, ~75条, 14.2%)

本仓频次最高的缺陷类别。集中在op_host的tiling层和参数校验层。子类型分布: 条件覆盖不全(~20)、校验过严误拦(~15)、逻辑取反/运算符错(~12)、校验缺失(~10)、early return跳过逻辑(~8)、零值/边界(~10)。

---

#### 规则 COND-01: OP_CHECK_IF/OP_CHECK条件语义反转

严重等级: P0

缺陷描述: OP_CHECK_IF宏的设计语义是"参数为true时触发失败"，但开发者常将"合法条件"直接传入(不加取反)，导致合法输入被拒绝、非法输入被放行。528条缺陷中此模式出现约12次，是本仓条件类缺陷的第一大子类。

典型代码示例:
```cpp
// 缺陷代码 (b6ec4bfe86)
// CheckSameShape返回true表示shape合法
OP_CHECK_IF(CheckSameShape(qShape, kShape), ...);  // true=合法 -> 触发失败 -> 合法被拒

// 修复代码
OP_CHECK_IF(!CheckSameShape(qShape, kShape), ...);  // true=不合法 -> 触发失败 -> 正确
```

审查检查方法:
- 逐个核对OP_CHECK_IF参数的语义方向: 条件为true应表示"错误状态"
- 当参数是函数调用(如CheckXxx())时，确认函数返回true的含义是否与CHECK宏的fail语义一致
- 搜索diff中所有OP_CHECK_IF/OP_CHECK调用，特别关注直接传入bool函数(未取反)的用法

关联commit: b6ec4bfe86, 479084fc, c8cf3f011d

---

#### 规则 COND-02: 逻辑运算符OR/AND混淆导致恒真/恒假

严重等级: P0

缺陷描述: `(x != A || x != B)` 恒为true、`(x == A && x == B)` 恒为false是经典逻辑错误。本仓多次出现在多枚举值校验场景中。

典型代码示例:
```cpp
// 缺陷代码 (67f1fffc)
if (dimNum != DIM3 || dimNum != DIM4) {  // 恒true: 不等于3或不等于4，任何值都满足其一
    return GRAPH_FAILED;
}

// 修复代码
if (dimNum != DIM3 && dimNum != DIM4) {  // 既不是3也不是4时才报错
    return GRAPH_FAILED;
}
```

```cpp
// 缺陷代码 (b6bdcb97)
if ((D <= 256) || (D == otherD)) {  // 应为&&: D<=256且D相等

// 修复代码
if ((D <= 256) && (D == otherD)) {
```

审查检查方法:
- 搜索diff中`!= ... ||`和`== ... &&`模式，逐一验证逻辑意图
- 多枚举值OR连接时确认每个值是否属于同一约束集
- 两个不等式用OR连接时考虑是否应为AND

关联commit: 67f1fffc, b6bdcb97

---

#### 规则 COND-03: 新增枚举/Layout/模式时分支覆盖不全

严重等级: P1

缺陷描述: 新增枚举值(sparseMode、quantMode、Layout等)或运行模式后，代码中多处if/switch分支未覆盖新值，导致走入错误的默认路径。约20条缺陷属此模式。

典型代码示例:
```cpp
// 缺陷代码 (c8cf3f011d)
// NTD全量化拦截范围过宽: 只有Prefill MLA不支持, 但对所有NTD生效
if (layoutType == NTD) {
    return GRAPH_FAILED;  // 误拦NTD下的合法Decode路径
}

// 修复代码
if (layoutType == NTD && isPrefillMLA) {
    return GRAPH_FAILED;
}
```

审查检查方法:
- 新增枚举值时全局搜索该枚举类型的所有switch/if-else，确认新值已覆盖
- 检查default/else分支是否会对新值产生意外行为
- 新增模式(如quantMode)时同步审查所有引用该字段的条件分支

关联commit: c8cf3f011d, b4b9198148, a0077ae9, 82dbcd14

---

#### 规则 COND-04: 校验过严误拦合法输入

严重等级: P1

缺陷描述: 防御性校验的条件范围过宽，将合法但非典型的输入路径拒绝。表现为新功能/新数据类型上线后被已有校验误拦。约15条。

审查检查方法:
- 新增校验时反向思考"是否会误拦合法场景"
- 新增数据类型/格式时检查已有校验是否需要扩展白名单
- 校验条件涉及多个维度时逐一确认各维度独立性

关联commit: c8cf3f011d, 1eb3f0ce04

---

#### 规则 COND-05: 零值/除零/边界未防御

严重等级: P1

缺陷描述: 除数为0未检查、==0 vs <=0 混淆、off-by-one错误。AlignUp/CeilDivision在除数为0时静默返回0，下游使用错误参数。约10条。

典型代码示例:
```cpp
// 缺陷代码 (grouped_matmul_tiling.cpp结构性风险)
ratio = float(curTaskNum) / AlignUp(uint32_t(curTaskNum), aicNum);
// aicNum=0时AlignUp返回0, 除零产生NaN

// 修复方向: 调用前校验 aicNum > 0
```

审查检查方法:
- 每个除法/取模表达式核查除数是否可能为0
- AlignUp/CeilDivision调用的第二参数必须>0
- 无符号减法(a-b)检查a>=b，否则下溢为极大值

关联commit: b7a0e9fced, 479084fc

---

## 类别2: CMake/构建配置 (BUILD, ~65条, 12.3%)

本仓最具结构性的缺陷来源。cmake/custom_build.cmake(23次触及)和cmake/func.cmake(17次触及)是Top1和Top3缺陷热点文件。核心根因是平行实现导致的"修一处漏一处"。

---

#### 规则 BUILD-01: CMake OP_NAME/target名复制粘贴错误

严重等级: P2

缺陷描述: 新增算子时从已有算子复制CMakeLists.txt，OP_NAME等关键标识符未替换为新算子名。约12条。

典型代码示例:
```cmake
# 缺陷代码 (5508fbb6c9)
# moe_init_routing/op_host/CMakeLists.txt
set(OP_NAME MoeFinalizeRouting)  # 从finalize复制未改, 应为MoeInitRouting

# 修复代码
set(OP_NAME MoeInitRouting)
```

审查检查方法:
- CMakeLists中OP_NAME、OPTYPE与所在目录名/算子实际名称逐字核对
- 新增算子PR中搜索diff是否包含其他算子名(复制来源的残留)
- 检查target名是否与source文件名、namespace一致

关联commit: 5508fbb6c9, d69eec6dab

---

#### 规则 BUILD-02: 新平台编译选项/分支未覆盖

严重等级: P1

缺陷描述: 新增硬件平台(如ascend910_95/A5)时，多处CMakeLists/cmake中的平台条件分支(ASCEND_COMPUTE_UNIT)未添加新平台。约10条。8个moe CMakeLists一次性遗漏A5平台分支(eeba666256)是典型案例。

典型代码示例:
```cmake
# 缺陷代码 (eeba666256)
# 8个moe相关CMakeLists均缺少ascend910_95分支
if(ASCEND_COMPUTE_UNIT STREQUAL "ascend910b")
    target_compile_options(...)
elseif(ASCEND_COMPUTE_UNIT STREQUAL "ascend310p")
    target_compile_options(...)
# 缺少: elseif(ASCEND_COMPUTE_UNIT STREQUAL "ascend910_95")

# 修复: 添加910_95分支及其编译选项
```

审查检查方法:
- 新平台PR: 全局搜索ASCEND_COMPUTE_UNIT/socVersion条件，逐一确认新平台已覆盖
- 统计搜索命中数与实际修改数是否一致(遗漏=命中但未修改)
- 检查新平台的编译选项是否与最近似的已有平台一致

关联commit: eeba666256, e9ec0142

---

#### 规则 BUILD-03: auto-sync编译选项与kernel代码冲突

严重等级: P1

缺陷描述: --cce-auto-sync=on时编译器自动插入同步屏障，但kernel代码中已手动插入同步指令。两者叠加导致重复同步(性能下降)或auto-sync被关闭但手动同步不完整(数据竞争)。

典型代码示例:
```cmake
# 缺陷代码 (38606b2b)
# auto-sync=on, 但kernel中手动调用SetFlag/WaitFlag
target_compile_options(... --cce-auto-sync=on)
# kernel: SetFlag<HardEvent::V_MTE3>(event); WaitFlag(event);
# 冲突: 编译器也会插入同步, 可能导致死锁或不必要停顿

# 修复: 手动同步的kernel必须设置 --cce-auto-sync=off
```

审查检查方法:
- diff中出现auto-sync设置变更时，检查对应kernel是否有手动SetFlag/WaitFlag
- kernel中新增手动同步时，检查CMakeLists中auto-sync是否为off
- 同一算子的不同平台分支auto-sync设置是否一致

关联commit: 38606b2b, e9ec0142

---

#### 规则 BUILD-04: include路径变更未同步

严重等级: P2

缺陷描述: 头文件路径重组后，引用该头文件的cmake/源码未同步更新。include guard名冲突、依赖缺失也属此类。约12条。

审查检查方法:
- include路径变更时grep所有引用该头文件的文件，确认路径同步更新
- 新增include guard时搜索同名guard是否已存在
- 跨仓头文件路径变更须确认上游变更已就绪

关联commit: 05c16e686c, bb90fa511

---

#### 规则 BUILD-05: 黑名单/分包/注册配置未更新

严重等级: P2

缺陷描述: 新增算子后operator_list.yaml、classify_rule.yaml、A5_OPS_BLACK_LIST等配置文件未同步更新。约10条。

审查检查方法:
- 新增算子PR中检查: classify_rule.yaml、operator_list.yaml是否已添加新算子
- 算子从黑名单中移除时确认对应平台编译/测试已通过
- A5_OPS_BLACK_LIST元素末尾分号格式是否正确(func.cmake中list(FIND)依赖此格式)

关联commit: eeba666256, 422548259b

---

## 类别3: Buffer/内存/地址 (BUF, ~60条, 11.4%)

kernel层和tiling层最危险的缺陷类别。可导致OOM、精度错误、静默数据损坏。fia_kernel_nonquant_mla.h(10次触及)中的sizeof链是典型结构性风险。

---

#### 规则 BUF-01: workspace偏移/大小计算溢出

严重等级: P0

缺陷描述: workspace大小 = batch * head * seq * dim * sizeof(type)等多维度连乘，int32乘法溢出导致workspace分配不足、后续写入越界。offset累加链中sizeof类型交替(half->float->half)时任一类型错误导致后续全部错位。约15条。

典型代码示例:
```cpp
// 缺陷场景 (prompt_flash_attention_tiling_v2.cpp结构性风险)
// 六值连乘, 大batch+大head场景int32溢出
uint32_t wsSize = batchSize * gSize * headNumSize * kvSplitPart * vHeadSize * sizeof(float);

// 修复方向: 至少一个操作数提升到uint64
uint64_t wsSize = (uint64_t)batchSize * gSize * headNumSize * kvSplitPart * vHeadSize * sizeof(float);
```

```cpp
// 缺陷代码 (fia_kernel_nonquant_mla.h结构性风险)
// workspace offset链: 每个buffer的offset依赖前一个的sizeof(type)*size
// 修改buffer[2]的类型从half->float, buffer[3-6]全部错位, 不报错
gmOffset[3] = gmOffset[2] + sizeof(half) * buf2Size;  // 若buf2实际是float, 此处sizeof错误
```

审查检查方法:
- workspace/buffer大小计算: 涉及3个以上维度连乘时，至少一个操作数cast到uint64
- sizeof链: 核对每段sizeof(type)与实际数据类型是否匹配(type交替half/float/half时尤其注意)
- offset累加: 验证每个offset = 前一个offset + 前一个size，链条无断裂

关联commit: e5817c02, 29edf728

---

#### 规则 BUF-02: DataCopy参数/边界越界

严重等级: P0

缺陷描述: DataCopy搬运长度超buffer分配大小、stride超uint16/uint32上限截断、尾块搬运用固定块大小而非实际剩余大小。约12条。

典型代码示例:
```cpp
// 缺陷代码 (2efc569c)
// 尾块搬运: 用固定blockSize而非实际剩余, 超界覆写相邻scale数据
DataCopy(dst, src, blockSize);

// 修复代码: 尾块用实际剩余大小
uint32_t copyLen = (isLastBlock) ? remainSize : blockSize;
DataCopy(dst, src, copyLen);
```

```cpp
// 缺陷代码 (1dbd7856)
// 按RoundUp(headSize)搬运, 非32B对齐时超出GM连续空间
DataCopy(ubBuf, gmBuf, RoundUp(headSize, 32));

// 修复: 使用DataCopyPad处理非对齐尾部
```

审查检查方法:
- DataCopy搬运长度 <= 目标buffer分配大小
- stride参数不超uint16(65535)/uint32上限; 超出时用DataCopyExtParams
- 尾块/边界块搬运需用min(blockSize, remainSize)

关联commit: 2efc569c, 1dbd7856, cefdc8776e

---

#### 规则 BUF-03: GM访问前缺边界检查(check-before-access)

严重等级: P0

缺陷描述: GetValue/SetValue等GM访问操作在边界检查之前执行，index越界时直接访问非法地址。约10条。

典型代码示例:
```cpp
// 缺陷代码 (af72ccb938)
auto val = blockTablesGm_.GetValue(offset);  // 先读
if (offset >= blockTableWidth_) { break; }   // 后检查 -> 已越界

// 修复代码
if (offset >= blockTableWidth_) { break; }   // 先检查
auto val = blockTablesGm_.GetValue(offset);  // 后读
```

审查检查方法:
- 每个GM GetValue/SetValue调用前是否有边界检查(index < size)
- 边界检查与GM访问之间无条件跳转(不会被跳过)
- 循环中的GM访问: 循环条件是否包含边界约束

关联commit: af72ccb938

---

#### 规则 BUF-04: UB buffer复用缺同步/清零

严重等级: P1

缺陷描述: 两个逻辑buffer映射同一物理UB区间时缺同步屏障; buffer复用前未清零导致脏数据残留; 清零长度小于分配大小。约12条。

典型代码示例:
```cpp
// 缺陷代码 (f55d2a35)
// statusTensor清零长度 < buffer大小, 尾部脏数据
Duplicate(statusTensor, (half)0, partialSize);  // partialSize < bufferSize

// 修复: 清零整个buffer
Duplicate(statusTensor, (half)0, bufferSize);
```

审查检查方法:
- buffer复用: 新用途开始前是否已清零或完全覆写
- 清零范围: Duplicate/memset的长度 >= buffer实际分配大小
- 同一物理UB被两个逻辑buffer使用时，中间必须有PipeBarrier

关联commit: f55d2a35, 64f59aae, 949f8193fd

---

## 类别4: 同步/流水线时序 (SYNC, ~35条, 6.6%)

NPU kernel中最难调试的缺陷类型。表现为偶现nan/精度抖动/hang死。子类型: 流水线屏障缺失(~10)、HardEvent类型错误(~8)、SetFlag/WaitFlag不配对(~6)、多核同步遗漏(~5)。

---

#### 规则 SYNC-01: 流水线屏障缺失(RAW hazard)

严重等级: P0

缺陷描述: 数据在管道A(如V/MTE2)产生后被管道B(如MTE3/V)消费，中间缺少对应管道的PipeBarrier或HardEvent同步，导致读到未完成的数据(RAW数据竞争)。约10条。

典型代码示例:
```cpp
// 缺陷代码 (d2601de86d)
DataCopyPad(gmDst, v0ValidSizeUb_, ...);  // MTE3写GM
// 缺少: WaitFlag<MTE3_S>
// 后续代码复用v0ValidSizeUb_ -> 数据竞争

// 修复代码
DataCopyPad(gmDst, v0ValidSizeUb_, ...);
SetFlag<HardEvent::MTE3_S>(eventId);
WaitFlag<HardEvent::MTE3_S>(eventId);
// 安全复用v0ValidSizeUb_
```

```cpp
// 缺陷代码 (37105bc0)
Cast(vOutput, input, ...);       // V管道产出
DataCopyPad(gmDst, vOutput, ...); // MTE3消费 -> 缺V->MTE3同步

// 修复: 中间加 PipeBarrier<PIPE_V>
```

审查检查方法:
- 每个DataCopy/Duplicate后: 在消费者读取同一buffer前是否有对应管道的barrier/event
- 确认生产者管道(写)和消费者管道(读)不同时，中间有跨管道同步
- 特别注意: buffer被写入GM后立即复用的场景

关联commit: d2601de86d, 949f8193fd, 37105bc0

---

#### 规则 SYNC-02: HardEvent类型与数据流方向不匹配

严重等级: P0

缺陷描述: HardEvent类型指定了生产者->消费者的管道对，类型选错导致同步的是错误的管道对，实际数据流无同步保护。

典型代码示例:
```cpp
// 缺陷代码 (d5c79c44c7)
SetFlag<HardEvent::V_MTE2>(eventId);    // 同步V->MTE2
// 实际数据流: MTE3产出 -> MTE2消费, 应为MTE3_MTE2

// 修复代码
SetFlag<HardEvent::MTE3_MTE2>(eventId); // 正确同步MTE3->MTE2
```

审查检查方法:
- HardEvent类型: 核对模板参数中的管道对是否与实际数据流方向一致
- 生产者是DataCopy(MTE2/MTE3)还是Duplicate/Cast(V): 决定了事件类型的第一个管道
- 消费者是DataCopy(MTE2/MTE3)还是算术运算(V): 决定第二个管道

关联commit: d5c79c44c7

---

#### 规则 SYNC-03: SetFlag/WaitFlag不配对

严重等级: P0

缺陷描述: 未Set即Wait导致死锁; 多余Wait导致流水线不必要停顿; Set后遗漏对应Wait导致同步缺失。

典型代码示例:
```cpp
// 缺陷代码 (ee07e5dd25)
// sink场景分配了独立的eventIdVToMte2Sink, 与非sink路径的eventIdVToMte2A形成两套事件
// 条件分支下Set/Wait路径不对称, 导致同步语义混乱
SetFlag<HardEvent::V_MTE2>(eventIdVToMte2Sink);  // sink独立事件
WaitFlag<HardEvent::V_MTE2>(eventIdVToMte2Sink);  // 在循环头部wait, 但首次迭代未set

// 修复: 删除独立的sink事件, 统一复用eventIdVToMte2A, 用!hasSink条件避免重复set
SetFlag<HardEvent::V_MTE2>(eventIdVToMte2A);  // 统一到同一事件
```

审查检查方法:
- 全局搜索SetFlag/WaitFlag配对，确认每个Wait都有对应的Set
- 同一eventId的Set和Wait之间无条件跳转(某分支只Set不Wait或反之)
- 条件分支中的event使用: if分支Set了event则else分支也必须处理

关联commit: ee07e5dd25

---

## 类别5: 初始化/赋值遗漏 (INIT, ~30条, 5.7%)

子类型: 条件分支遗漏赋值(~10)、构造/基类初始化缺失(~6)、清零/reset不完整(~5)、初始化时序错误(~5)、UB脏数据残留(~4)。

---

#### 规则 INIT-01: 条件分支遗漏赋值(if赋值else未赋值)

严重等级: P1

缺陷描述: 变量仅在if分支赋值，else分支保持默认值(通常为0)，后续公共路径使用该变量。unsigned类型的0减1下溢为UINT32_MAX是典型后果。

典型代码示例:
```cpp
// 缺陷代码 (b7a0e9fced)
uint32_t kvSLoopNumTotal = 0;
if (condition) {
    kvSLoopNumTotal = calcValue;
}
// else分支未赋值, kvSLoopNumTotal保持0
uint32_t loopEnd = kvSLoopNumTotal - 1U;  // 0-1 = UINT32_MAX -> 循环爆炸
```

审查检查方法:
- if赋值的变量: 检查else/default分支是否也赋值
- unsigned变量: 减法操作前确认变量>0
- 新增分支(如新模式/新Layout)时确认所有输出变量都被赋值

关联commit: b7a0e9fced, 3aacdac4, 66337ddb6b

---

#### 规则 INIT-02: 构造/基类初始化缺失

严重等级: P1

缺陷描述: 子类构造函数未调用基类InitCompileInfo()等初始化方法，或结构体字段声明时未给初始值，导致使用未初始化数据。

典型代码示例:
```cpp
// 缺陷代码 (1fab7bab45)
class DerivedTiling : public BaseTiling {
    DerivedTiling() {
        // 未调用 BaseTiling::InitCompileInfo()
        // platformInfo等基类成员未初始化
    }
};

// 修复: 构造函数中调用 InitCompileInfo()
```

审查检查方法:
- 新增子类: 是否调用基类Init方法
- 结构体新增字段: 是否有默认初始值或在所有构造路径被赋值
- 成员变量声明: 是否在类定义处赋默认值

关联commit: 1fab7bab45, 3aacdac4

---

#### 规则 INIT-03: 循环/batch切换时状态残留

严重等级: P1

缺陷描述: 多batch/多循环迭代间，上一轮的中间状态(偏移量、累加值、标志位)未被正确reset，导致后续迭代使用残留值。

典型代码示例:
```cpp
// 缺陷代码 (1d256572)
// batch切换时ROPE偏移量未更新, 使用上一batch的残留值
for (int b = 0; b < batchSize; b++) {
    // ropeOffset未在batch切换时重新计算
    ProcessBatch(b, ropeOffset);  // 第二个batch用了第一个batch的offset
}

// 修复: 每个batch开始时重新计算offset
```

审查检查方法:
- 循环体: 上一迭代的中间变量是否在新迭代开始时reset
- batch处理: 状态变量(offset/flag/accumulator)是否在batch切换时清零/重算
- buffer复用: 跨迭代buffer是否需要清零

关联commit: 1d256572, 64f59aae

---

## 类别6: 整数溢出/类型转换 (OVFL, ~30条, 5.7%)

子类型: 字节数/元素数混淆(~8)、uint16/uint32截断(~8)、int32乘法溢出(~6)、有符号/无符号比较(~4)。

---

#### 规则 OVFL-01: 字节数/元素数混淆

严重等级: P0

缺陷描述: API期望元素数但传入字节数(或反之)，导致操作范围成倍偏差。Duplicate的count传字节偏移(4倍越界)是典型案例。

典型代码示例:
```cpp
// 缺陷代码 (798597da0a)
Duplicate(dst, value, byteOffset);  // API期望元素数, 传入字节偏移
// 实际写入量 = byteOffset * sizeof(type) = 4倍越界 -> OOM

// 修复代码
Duplicate(dst, value, elementCount);  // 传入元素数
```

```cpp
// 缺陷代码 (45cfad32)
InitOutput(buf, byteSize);  // API参数语义为元素数, 传入字节数
// 初始化范围翻倍越界

// 修复: byteSize / sizeof(elementType)
```

审查检查方法:
- 每个API调用核对文档确认参数单位(字节数需除以sizeof, 元素数不需要)
- Duplicate/InitOutput/DataCopy: 参数名含"count/num"则为元素数, 含"size/bytes"则为字节数
- sizeof出现的位置: workspace分配用字节, kernel API用元素

关联commit: 798597da0a, 45cfad32

---

#### 规则 OVFL-02: DataCopy stride/count截断溢出

严重等级: P0

缺陷描述: DataCopyParams的srcStride/dstStride为uint16_t(上限65535)，当输入shape较大时stride超限被截断，数据搬运错位。

典型代码示例:
```cpp
// 缺陷代码 (cefdc8776e)
DataCopyParams params;
params.srcStride = (uint16_t)srcStride;  // srcStride > 65535时截断

// 修复代码
if (srcStride <= UINT16_MAX) {
    DataCopy(dst, src, params);
} else {
    DataCopyExtParams extParams;  // 使用Ext版本支持更大stride
    extParams.srcStride = srcStride;
    DataCopyExt(dst, src, extParams);
}
```

审查检查方法:
- DataCopy的stride/count参数: 检查输入shape是否可能导致超uint16上限
- CeilDivision返回值: 确认与接收变量类型匹配(uint16 vs uint64)
- 大shape场景(seq > 65535): 优先使用DataCopyExtParams

关联commit: cefdc8776e, 025c1734

---

#### 规则 OVFL-03: 多维度连乘int32溢出

严重等级: P1

缺陷描述: 地址偏移 = uint32 * uint32在大数据场景(>4GB)溢出。workspace = batch * head * seq * dim * sizeof(type)在int32范围内不足。

典型代码示例:
```cpp
// 缺陷代码 (29edf728)
uint32_t offset = tndBatch * seqLen * headDim;  // TND场景溢出

// 修复代码
uint64_t offset = (uint64_t)tndBatch * seqLen * headDim;
```

审查检查方法:
- 3个以上维度连乘: 至少一个操作数cast到uint64/int64
- GM地址计算: 结果变量必须为uint64_t
- 特别关注TND/大batch/大seq场景

关联commit: 29edf728, e5817c02

---

## 类别7: Shape/Layout处理 (SHAPE, ~28条, 5.3%)

子类型: 新Layout分支遗漏(~10)、Layout专属逻辑范围错误(~6)、维度索引/来源错误(~6)。

---

#### 规则 SHAPE-01: 新Layout分支遗漏

严重等级: P1

缺陷描述: 新增Layout(NTD/TND/BSND等)后，代码中多处if/switch未覆盖新Layout，走入错误的默认处理路径。三处遗漏NTD(b4b9198148)是典型案例。

典型代码示例:
```cpp
// 缺陷代码 (b4b9198148)
// 三处遗漏NTD layout处理: dstStride / IFAMask / isGqa
if (layout == BSH) { ... }
else if (layout == BNSD) { ... }
// 缺少: else if (layout == NTD) { ... }
// NTD走入else的默认路径, 逻辑不正确
```

审查检查方法:
- 新增Layout PR: 搜索所有引用现有Layout枚举的条件分支，逐一确认新Layout已覆盖
- 统计搜索命中数与实际修改数(遗漏 = 命中但未修改的处)
- 使用if constexpr限制Layout专属逻辑只在目标Layout下执行

关联commit: b4b9198148, a0077ae9, 82dbcd14

---

#### 规则 SHAPE-02: Layout专属逻辑在错误Layout下执行

严重等级: P1

缺陷描述: 某些计算逻辑仅对特定Layout有效(如累加batch偏移仅BSH需要)，但条件范围过宽导致对所有Layout执行。

典型代码示例:
```cpp
// 缺陷代码 (0166982e)
// BSND不需累加batch偏移, 但对所有layout都执行了
gmOffset += batchIdx * seqLen * headDim;  // BSND下batchIdx已编码在shape中

// 修复: 仅BSH/BNSD下执行
if (layout == BSH || layout == BNSD) {
    gmOffset += batchIdx * seqLen * headDim;
}
```

审查检查方法:
- 偏移量/stride计算: 确认计算公式是否对当前Layout成立
- 不同Layout下batch/seq/head维度的位置不同，核对取值来源
- TND下tSize应用GetTSize()而非batchSize*qSeqSize(ec9278e0)

关联commit: 0166982e, ec9278e0

---

## 类别8: 接口/返回值 (API, ~25条, 4.7%)

子类型: 返回值未检查(~8)、函数签名不匹配(~6)、错误码语义误用(~5)。

---

#### 规则 API-01: Check函数返回值被丢弃

严重等级: P0

缺陷描述: 调用CheckXxx()函数后不检查返回值(ge::graphStatus)，即使返回GRAPH_FAILED也继续执行后续逻辑，非法输入不被拦截。

典型代码示例:
```cpp
// 缺陷代码 (0f363800)
CheckFAIIsTND(context);  // 返回值被完全丢弃
// GRAPH_FAILED也继续返回SUCCESS

// 修复代码
auto ret = CheckFAIIsTND(context);
if (ret != ge::GRAPH_SUCCESS) { return ret; }
```

```cpp
// 缺陷代码 (e7278f18)
CheckNonQuantMatmulDataType(dtypeA, dtypeB);  // 返回值被丢弃
// 非法dtype不被拦截
```

审查检查方法:
- diff中所有Check/Verify函数调用: 返回值是否被检查并传播
- 考虑对Check函数添加[[nodiscard]]属性
- LOGE后必须紧跟return错误码(否则仅打日志不生效)

关联commit: 0f363800, e7278f18

---

#### 规则 API-02: return false在graphStatus函数中(语义反转)

严重等级: P1

缺陷描述: 函数声明返回ge::graphStatus(整型)，但函数体中return false/true。bool->int隐式转换导致false=0=GRAPH_SUCCESS，本意返回失败实际返回成功。CheckUbSpace()是已确认存量bug。

典型代码示例:
```cpp
// 缺陷代码 (479084fc, incre_flash_attention_tiling_v2.cpp存量bug)
ge::graphStatus CheckUbSpace(...) {
    if (condition) {
        return false;  // 意图返回失败, 但 false=0=GRAPH_SUCCESS
    }
    return true;  // 意图返回成功, 但 true=1=GRAPH_FAILED
}

// 修复: 使用正确的枚举值
ge::graphStatus CheckUbSpace(...) {
    if (condition) {
        return ge::GRAPH_FAILED;
    }
    return ge::GRAPH_SUCCESS;
}
```

审查检查方法:
- 返回ge::graphStatus的函数中搜索return true/false
- 静态分析规则: 检测graphStatus函数中的bool返回值
- LOGE后检查是否紧跟return ge::GRAPH_FAILED(而非缺少return)

关联commit: 479084fc

---

#### 规则 API-03: 函数签名/参数不匹配

严重等级: P1

缺陷描述: extern声明与实际定义参数列表不一致; 指针值传递应为引用; SetOutputDataType参数写反。

典型代码示例:
```cpp
// 缺陷代码 (0c941da3)
void ProcessData(SomeType* ptr) {  // 值传递指针
    ptr = newPtr;  // 修改局部副本, 调用方不可见
}

// 修复: 引用传递
void ProcessData(SomeType*& ptr) {
    ptr = newPtr;  // 修改调用方的指针
}
```

审查检查方法:
- extern声明与实际函数定义参数列表逐一比对
- 函数内修改指针本身(非指向内容)时: 参数应为指针引用或二级指针
- API版本升级: V(N+1)新增参数不应在V(N)接口中调用

关联commit: 0c941da3, 7eee21c7, 051f878c

---

## 类别9: 参数值/顺序/魔数 (PARAM, ~25条, 4.7%)

---

#### 规则 PARAM-01: API参数语义/顺序混淆

严重等级: P1

缺陷描述: 函数参数位置调换(如rowOuterIdx与rowInnerIdx); 参数语义与实际传值相反(elementLengths和srcList赋值互换)。参数>3个时尤其易错。

典型代码示例:
```cpp
// 缺陷代码 (e4b3ba6d)
// elementLengths[i]与srcList.src[i+1]的Dst/Src对应关系反了
params.elementLengths[0] = mrgDstNum;  // 应为mrgSrcNum(对应src1)
params.elementLengths[1] = mrgSrcNum;  // 应为mrgDstNum(对应src2)
srcList.src1 = mrgDst;                 // 应为mrgSrc
srcList.src2 = mrgSrc;                 // 应为mrgDst

// 修复: Dst/Src的index顺序互换, 使elementLengths与srcList对应
params.elementLengths[0] = mrgSrcNum;
params.elementLengths[1] = mrgDstNum;
srcList.src1 = mrgSrc;
srcList.src2 = mrgDst;
```

```cpp
// 缺陷代码 (7075b6143f)
ProcessRow(rowInnerIdx, rowOuterIdx);  // 实际定义是(outer, inner) -> 顺序反了
```

审查检查方法:
- API调用核对参数与函数声明中参数名的语义对应(不仅看类型)
- 函数参数>3个: 考虑用结构体封装避免位置混淆
- 同名参数(如两个int): 特别注意顺序

关联commit: e4b3ba6d, 7075b6143f

---

#### 规则 PARAM-02: 常量名值不匹配/魔法数字

严重等级: P2

缺陷描述: 常量名与值矛盾(ZERO_DIM=1, TWO_DIM=3); 边界值硬编码无推导来源注释; 非标准无穷大表示(3e+99而非IEEE 754)。

典型代码示例:
```cpp
// 缺陷代码 (cfe618f7)
constexpr int ZERO_DIM = 1;  // 名为ZERO实际值为1
constexpr int TWO_DIM = 3;   // 名为TWO实际值为3

// 修复: 名称与值匹配
constexpr int ZERO_DIM = 0;
constexpr int TWO_DIM = 2;
```

审查检查方法:
- 常量定义: 名称与值必须语义一致
- 魔法数字: 边界值/阈值需定义为命名常量并附注推导来源
- 无穷大: 使用std::numeric_limits<T>::infinity()而非字面量

关联commit: cfe618f7, cb1c8346, fb58b43e98

---

## 类别10: 硬件平台兼容性 (PLAT, ~25条, 4.7%)

---

#### 规则 PLAT-01: 新平台运行时/编译期分支未覆盖

严重等级: P1

缺陷描述: socVersion字符串比较路由("Ascend910_95")、`#if __CCE_AICORE__ == 220`预处理分支、ASCEND_COMPUTE_UNIT条件——新增硬件型号时多处遗漏。

典型代码示例:
```cpp
// 缺陷代码 (4bc16847)
constexpr int cvRatio_ = 2;  // 硬编码c:v比例
// A5平台c:v=1:1, 此constexpr在编译期确定, 运行时无法区分

// 修复: 运行时获取
int cvRatio_ = GetSubBlockNum();  // API返回实际硬件比例
```

审查检查方法:
- 新平台适配PR: 搜索socVersion/ASCEND_COMPUTE_UNIT/__CCE_AICORE__所有出现位置
- 硬编码硬件参数(cvRatio/对齐粒度/核数): 应通过API获取而非hardcode
- constexpr用于平台参数时确认是否需要改为运行时获取

关联commit: 4bc16847, bf74834, b85709b25e

---

#### 规则 PLAT-02: 平台特有寄存器/配置遗漏

严重等级: P2

缺陷描述: 新arch适配时遗漏平台特有的SPR/控制寄存器设置; InitGlobalMemory对齐要求跨平台不同。

审查检查方法:
- 新arch kernel: 检查是否需要特殊SPR/控制寄存器设置(参考已有arch适配)
- 平台切换时: InitGlobalMemory/对齐参数是否需要调整

关联commit: b85709b25e

---

## 类别11: TilingKey一致性 (TKEY, ~20条, 3.8%)

host侧tiling生成的key值与kernel侧模板匹配常量不一致。本仓最具特征性的缺陷模式，也是Revert事件群1(6次tilingKey模板化整改连续失败)的根因。

---

#### 规则 TKEY-01: host/kernel tilingKey常量值不一致

严重等级: P0

缺陷描述: host侧通过GET_TPL_TILING_KEY宏或手动计算生成tilingKey，kernel侧通过TILING_KEY_IS/ASCENDC_TPL_SEL匹配。两侧值不一致导致kernel走入错误的模板分支或无法匹配任何分支。约8条。

典型代码示例:
```cpp
// 缺陷代码 (02505c922e系列)
// host侧: tilingKey = ... (新编码方式)
// kernel侧: TILING_KEY_IS(oldValue) -> 无法匹配host下发的newValue
// 结果: 无法命中正确kernel分支, 可能走default fallback或行为未定义
```

审查检查方法:
- tilingKey新增/修改的diff中: 必须同时出现host侧(tiling.cpp)和kernel侧(kernel.h/.cpp)的对应修改
- GET_TPL_TILING_KEY宏参数顺序与kernel侧ASCENDC_TPL_SEL严格一致
- 建议: host/kernel共享tilingKey常量定义头文件，避免手动维护两份

关联commit: 02505c922e, ee638d9b, 31044c91

---

#### 规则 TKEY-02: tilingKey编码维度缺失

严重等级: P1

缺陷描述: 新增feature/配置参数时未编入tilingKey，不同配置映射到同一key，共用同一kernel模板分支。

典型代码示例:
```cpp
// 缺陷代码 (31044c91)
// tilingKey未编码mte2Config参数
// 不同vec_config映射到同一key, kernel无法区分配置差异

// 修复: 将mte2Config编入key的某一位
tilingKey |= (mte2Config << BIT_OFFSET);
```

```cpp
// 缺陷代码 (aba3bd8f)
GetTilingKey(..., /*bit15=*/0, ...);  // 第15位传0而非tndS1Pingpong
// 不同pingpong配置生成相同key
```

审查检查方法:
- 新增配置参数时: 检查是否需要编入tilingKey(否则不同配置共用同一kernel)
- 位域顺序: 核对每一位的含义注释与实际传值
- tilingKey位数: 确认未溢出uint64范围(b41315b3: UINT64_MAX溢出值)

关联commit: 31044c91, aba3bd8f, b41315b3, 13ae078c

---

## 类别12: Tiling参数/计算 (TPARAM, ~20条, 3.8%)

---

#### 规则 TPARAM-01: tiling计算逻辑/函数替换错误

严重等级: P1

缺陷描述: CeilAlign被错误替换为CeilDiv(语义完全不同); early return跳过tiling计算导致字段未赋值; 校验魔数与tiling实际输出范围不匹配。

典型代码示例:
```cpp
// 缺陷代码 (586493dc)
ubFactor = CeilDiv(rawSize, alignment);   // CeilDiv: rawSize/alignment向上取整
// 原为CeilAlign: rawSize对齐到alignment的倍数
// CeilDiv结果远小于CeilAlign: 130016 vs 4063

// 修复: 恢复CeilAlign
ubFactor = CeilAlign(rawSize, alignment);
```

```cpp
// 缺陷代码 (66337ddb6b)
if (mode == DUAL_IN_OUT) {
    return;  // early return跳过后续tiling计算
    // vLoopNum_, sLoopNum_等未赋值, 保持0
}
```

审查检查方法:
- CeilAlign vs CeilDiv: 两者语义完全不同(对齐 vs 除法)，替换需验证结果
- early return: 确认return前所有输出字段都已被赋值
- tiling校验边界值: 魔数上下界需标注推导来源，变更后与tiling实际输出范围对齐

关联commit: 586493dc, 66337ddb6b, fb58b43e98, 1fab7bab45

---

## 类别13: 对齐计算 (ALIGN, ~18条, 3.4%)

---

#### 规则 ALIGN-01: 数据搬运对齐不满足硬件要求

严重等级: P1

缺陷描述: DataCopy/DataCopyPad搬运长度未满足32B(256bit)对齐; NZ格式行数未2行对齐; Mmad的m/n/k维度未BLOCK_SIZE对齐。

典型代码示例:
```cpp
// 缺陷代码 (3cfe2f3b64)
copyTotalS = rawTotalS;  // 未32对齐, NZ格式scale搬运失败

// 修复
copyTotalS = CeilAlign(rawTotalS, ALIGN_32);
```

```cpp
// 缺陷代码 (c439c4d7)
// Mmad的m维度未BLOCK_SIZE对齐, L0C矩阵乘精度问题
Mmad(dst, src, m, n, k);  // m未对齐

// 修复: m = CeilAlign(m, BLOCK_SIZE)
```

审查检查方法:
- DataCopy/DataCopyPad: 搬运长度是否32B对齐; 不满足时用DataCopyPad(自动padding)
- NZ格式: 行数需2行(16元素行)对齐
- Mmad: m/n/k需BLOCK_SIZE对齐

关联commit: 3cfe2f3b64, 446d15c9, c439c4d7

---

#### 规则 ALIGN-02: 不必要的对齐破坏地址计算

严重等级: P2

缺陷描述: 对不需要对齐的值执行对齐操作，对齐后值偏大，破坏后续偏移量计算。

典型代码示例:
```cpp
// 缺陷代码 (73a82c36)
perTokenSize_ = CeilAlign(rawPerTokenSize, 512);  // 不需要512对齐
// 对齐后偏大 -> 后续地址 = base + perTokenSize_ * idx 偏移错误

// 修复: 去掉不必要的对齐
perTokenSize_ = rawPerTokenSize;
```

审查检查方法:
- 每个CeilAlign/RoundUp: 确认对齐是否确实需要(有硬件/DMA要求)
- 对齐后值作为偏移量乘数时: 偏大会导致后续所有地址错位

关联commit: 73a82c36, ac141d52

---

## 类别14: 复制粘贴/拼写 (CPASTE, ~18条, 3.4%)

---

#### 规则 CPASTE-01: 复制粘贴后标识符未替换

严重等级: P1

缺陷描述: 从已有文件/函数复制代码后，关键标识符(变量名、函数名、buffer索引)未替换为新上下文的正确值。

典型代码示例:
```cpp
// 缺陷代码 (a02041efea)
// 从另一校验复制, 用了valueShapeInfo.b而非.n
OP_CHECK_IF(valueShapeInfo.b != expectedN, ...);  // .b应为.n

// 缺陷代码 (120106d9)
// L0A/L0B/L0C索引全写成ASCEND_CB, 从一行复制未改
InitBuffer(inQueueL0A_, ASCEND_CB, ...);  // 应为ASCEND_L0A
InitBuffer(inQueueL0B_, ASCEND_CB, ...);  // 应为ASCEND_L0B
InitBuffer(inQueueL0C_, ASCEND_CB, ...);  // 应为ASCEND_L0C
```

审查检查方法:
- 新增文件/函数: 如果是从已有代码复制，逐行检查所有标识符是否已替换
- 相似行(initA/initB/initC等): 逐行核对差异部分
- grep源算子/函数名: diff中不应残留复制来源的标识符

关联commit: a02041efea, e8bdf8f2, 120106d9

---

#### 规则 CPASTE-02: 平行函数组修改不同步

严重等级: P1

缺陷描述: V1/V2/V3等平行函数或combine_v2.h与_add_rms_norm.h等克隆文件，仅修改一处而未同步到平行代码。aclnn_grouped_matmul.cpp的6个版本入口是典型热点。

审查检查方法:
- diff中只修改了某函数的一个版本: 检查平行版本是否也需同样修改
- combine_v2.h与_add_rms_norm.h: 同一bug修复必须同步到两个文件
- aclnn接口V1-V5: 新增参数/check需同步6处入口

关联commit: 5508fbb6c9, e8bdf8f2

---

## 类别15: 功能回退/Revert (REVERT, ~16条)

35个Revert提交归为8个事件群。详见附录: Revert事件摘要。此处提炼核心审查规则。

---

#### 规则 REVERT-01: 架构方案首次失败后应暂停评审

严重等级: P1

缺陷描述: tilingKey模板化整改在MatmulAllReduce首次失败后未暂停，继续推进到其他MC2算子(MatmulReduceScatterV2、MoeDistributeCombine/Dispatch)，5天内连续6次Revert。

审查检查方法:
- 同一架构方案在首个模块失败并revert后: 要求根因分析和方案修正再继续
- 跨模块推广: 确认已有至少一个模块稳定运行

关联commit: 2001261f7c, 8ead68ccfa, 20f4d5412b, 58a7ef184e, a2b032dcb2, 02505c922e

---

#### 规则 REVERT-02: 大型新算子合入门禁

严重等级: P2

缺陷描述: 10个premature merge中7个在合入后2小时内revert，问题在本地编译/CI即可发现。commit message"接口调试"、MR描述空白是预警信号。

审查检查方法:
- 新增算子(>10个新文件): 必须填写测试栏并提供CI通过证据
- commit message含"调试"/"临时"/"WIP": 不应合入主干
- MR描述/branch名异常(如"merge master into master"): 需要额外审查

关联commit: 1788aaceea, 8611973812, 1ff006a430, dfa78b4b93

---

## 跨类别系统性规则 (SYS)

---

#### 规则 SYS-01: 代码克隆/平行实现"修一处漏一处"

严重等级: P1

缺陷描述: 本仓23次cmake缺陷和11次MC2 kernel缺陷的根因是"修一处漏一处"。已识别克隆对: custom_build.cmake vs CMakeLists.txt(RTY) 285行平行; func.cmake两个macro 80%重复; combine_v2.h vs _add_rms_norm.h 80%相似; aclnn_grouped_matmul.cpp 6个版本入口; PFA/IFA各自独立的TilingDataconvert。

审查检查方法:
- diff仅修改一个克隆体时: 搜索对应克隆体是否需同步修改
- cmake/custom_build.cmake修改时: 检查CMakeLists.txt(RTY分支)是否需同步
- func.cmake: add_modules_sources修改时检查add_modules_sources_with_soc
- MC2 MoE: combine_v2.h修改时检查_add_rms_norm.h

关联commit: 本仓高频模式，贯穿多个类别

---

#### 规则 SYS-02: 跨文件隐式一致性约束

严重等级: P0

缺陷描述: 无编译期保证的跨文件约束是本仓高频缺陷源。tilingKey按位组装(host) vs 模板参数解析(kernel): ~20条; workspace内存布局(host offset) vs kernel读取offset: ~8条; 宏参数顺序 vs 宏定义: ~5条。

审查检查方法:
- tilingKey修改: diff必须同时包含host和kernel两侧
- workspace offset变更: 验证host侧sizeof链与kernel侧读取逻辑一致
- 宏调用参数>5个: 逐一比对宏定义的参数名
- 建议: host/kernel共享常量定义(如tiling_key.h)，减少手动同步

关联commit: 涵盖TKEY全部案例和BUF-01部分案例

---

#### 规则 SYS-03: 整数溢出/除零静默失败

严重等级: P0

缺陷描述: 几乎所有tiling文件和kernel文件都涉及多维度连乘或除法，普遍缺少溢出/除零检查。workspace六值连乘(PFA tiling)、AlignUp/CeilDivision除数为0返回0(GMM tiling)、uint32地址溢出(MoE kernel)是已确认风险。

审查检查方法:
- 多维度连乘: 至少一个操作数提升到int64/uint64
- 除法/取模: 除数为0时应报错而非静默返回0
- 变量类型: 地址/偏移量统一使用uint64_t

关联commit: 涵盖OVFL全部案例和BUF-01部分案例

---

#### 规则 SYS-04: 硬件平台分支不完整

严重等级: P1

缺陷描述: 新增硬件型号时多处遗漏是常见缺陷来源。散布在: socVersion字符串比较路由、#if __CCE_AICORE__预处理分支、CMake ASCEND_COMPUTE_UNIT条件、平台常量(cvRatio/对齐粒度)。

审查检查方法:
- 新平台适配PR: 搜索所有平台分支入口(socVersion/ASCEND_COMPUTE_UNIT/__CCE_AICORE__)
- 平台参数: 优先用API获取(GetSubBlockNum())而非hardcode
- checklist: 编译选项、运行时分支、寄存器配置、对齐要求四项逐一确认

关联commit: 涵盖PLAT和BUILD-02全部案例

---

## 附录

### 审查规则优先级矩阵

P0 (致命, 10条):

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| COND-01 | OP_CHECK_IF条件语义反转 | COND | b6ec4bfe86 |
| COND-02 | 逻辑运算符OR/AND混淆 | COND | 67f1fffc |
| BUF-01 | workspace偏移/大小计算溢出 | BUF | e5817c02 |
| BUF-02 | DataCopy参数/边界越界 | BUF | 2efc569c |
| BUF-03 | GM访问前缺边界检查 | BUF | af72ccb938 |
| SYNC-01 | 流水线屏障缺失(RAW hazard) | SYNC | d2601de86d |
| SYNC-02 | HardEvent类型错配 | SYNC | d5c79c44c7 |
| SYNC-03 | SetFlag/WaitFlag不配对 | SYNC | ee07e5dd25 |
| OVFL-01 | 字节数/元素数混淆 | OVFL | 798597da0a |
| API-01 | Check函数返回值被丢弃 | API | 0f363800 |

P1 (严重, 18条):

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| COND-03 | 新增枚举/Layout分支覆盖不全 | COND | c8cf3f011d |
| COND-04 | 校验过严误拦合法输入 | COND | 1eb3f0ce04 |
| COND-05 | 零值/除零/边界未防御 | COND | b7a0e9fced |
| BUILD-02 | 新平台编译选项分支遗漏 | BUILD | eeba666256 |
| BUILD-03 | auto-sync与kernel代码冲突 | BUILD | 38606b2b |
| BUF-04 | UB buffer复用缺同步/清零 | BUF | f55d2a35 |
| INIT-01 | 条件分支遗漏赋值 | INIT | b7a0e9fced |
| INIT-02 | 构造/基类初始化缺失 | INIT | 1fab7bab45 |
| INIT-03 | 循环/batch切换状态残留 | INIT | 1d256572 |
| OVFL-02 | DataCopy stride/count截断 | OVFL | cefdc8776e |
| OVFL-03 | 多维度连乘int32溢出 | OVFL | 29edf728 |
| SHAPE-01 | 新Layout分支遗漏 | SHAPE | b4b9198148 |
| SHAPE-02 | Layout专属逻辑范围错误 | SHAPE | 0166982e |
| API-02 | return false在graphStatus函数中 | API | 479084fc |
| API-03 | 函数签名/参数不匹配 | API | 0c941da3 |
| PARAM-01 | API参数语义/顺序混淆 | PARAM | e4b3ba6d |
| PLAT-01 | 新平台分支未覆盖 | PLAT | 4bc16847 |
| TKEY-01 | host/kernel tilingKey不一致 | TKEY | 02505c922e |

P1 (严重, 续):

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| TKEY-02 | tilingKey编码维度缺失 | TKEY | 31044c91 |
| TPARAM-01 | tiling计算逻辑/函数替换错误 | TPARAM | 586493dc |
| ALIGN-01 | 数据搬运对齐不满足硬件要求 | ALIGN | 3cfe2f3b64 |
| CPASTE-01 | 复制粘贴后标识符未替换 | CPASTE | a02041efea |
| CPASTE-02 | 平行函数修改不同步 | CPASTE | 5508fbb6c9 |
| REVERT-01 | 架构方案首次失败后应暂停 | REVERT | 2001261f7c |
| SYS-01 | 代码克隆"修一处漏一处" | 跨类别 | - |
| SYS-02 | 跨文件隐式一致性约束 | 跨类别 | - |
| SYS-04 | 硬件平台分支不完整 | 跨类别 | - |

P2 (一般, 8条):

| 规则ID | 规则 | 覆盖类别 | 典型案例hash |
|--------|------|---------|-------------|
| BUILD-01 | CMake OP_NAME复制粘贴错误 | BUILD | 5508fbb6c9 |
| BUILD-04 | include路径变更未同步 | BUILD | 05c16e686c |
| BUILD-05 | 黑名单/分包配置未更新 | BUILD | eeba666256 |
| PARAM-02 | 常量名值不匹配/魔法数字 | PARAM | cfe618f7 |
| PLAT-02 | 平台特有寄存器/配置遗漏 | PLAT | b85709b25e |
| ALIGN-02 | 不必要的对齐破坏地址计算 | ALIGN | 73a82c36 |
| REVERT-02 | 大型新算子合入门禁 | REVERT | 1788aaceea |
| SYS-03 | 整数溢出/除零静默失败 | 跨类别 | - |

P3 (建议自动化, 6条):

| 规则ID | 规则 | 实现方式 |
|--------|------|---------|
| AUTO-01 | graphStatus函数中return bool检测 | 静态分析: 函数返回ge::graphStatus但含return true/false |
| AUTO-02 | Check函数返回值丢弃检测 | clang-tidy: 添加[[nodiscard]]或检测未使用返回值 |
| AUTO-03 | LOGE后缺return检测 | 静态分析: LOGE调用后下一条非return语句 |
| AUTO-04 | 窄类型截断检测 | -Wconversion/-Wnarrowing: DataCopy参数赋值的隐式截断 |
| AUTO-05 | 常量名值一致性检查 | lint: 命名含数字的常量其值应匹配(ZERO=0等) |
| AUTO-06 | CMake OP_NAME与目录名一致性 | CI脚本: 验证OP_NAME == 所在目录名 |

---

### 附录: 已确认存量bug

| 文件 | 缺陷 | 发现途径 |
|------|------|---------|
| CMakeLists.txt:60-61 | INDXE变量名拼写错误，arch32检测使用循环残留值 | 阶段4热点分析 |
| incre_flash_attention_tiling_v2.cpp | SetLayoutTypefaRun中BSH_BSND重复key | 阶段4热点分析 |
| incre_flash_attention_tiling_v2.cpp | CheckUbSpace()返回bool而声明graphStatus，语义反转 | 阶段4热点分析 |
| cmake/variables.cmake:267 | set(AICPU_INCLUDE ...)无条件覆盖前序list(APPEND)结果 | 阶段4热点分析 |

---

### 附录: Revert事件摘要

| 事件群 | 次数 | 代表hash | 模块 | 根因 | 存活时间 |
|--------|------|---------|------|------|---------|
| TilingKey模板化整改 | 6 | 2001261f7c | MC2(4组算子) | 方案设计阶段验证不足 | 1天内 |
| 大型新算子premature merge | 10 | 1788aaceea | MoE/GMM/IFA | 缺pre-merge CI gate | 48分钟~数小时 |
| 跨仓迁移协调失败 | 3 | 1337ec1579 | scatter_pa/mc2_3rd | 缺原子性保障 | 4-19小时 |
| 性能优化引入正确性问题 | 4 | a478f78879 | FAG/MoE/GMM | 缓存策略/buffer公式 | 3小时~8天 |
| 大规模重构一次性合入 | 4 | aee3fb63e8 | MC2/GMM | 缺分阶段策略 | 1天内 |
| Cleancode伪装语义变更 | 3 | 5474fdf4a6 | FIA/IFA | 删错误处理/改签名 | 1天内 |
| 构建系统/CI配置错误 | 3 | 422548259b | cmake/yaml/py | 跨层改动 | 24-36小时 |
| 功能开关批量开启 | 2 | f1fa74193b | tests(gmm) | 批量开启未逐个验证 | 1-2天 |

Revert时间线: 12月为高峰(12个)，对应tilingKey模板化整改集中推进期。

---

### 附录: 与ops-transformer主仓缺陷模式对比

共性(两仓均显著):
- COND(条件分支错误)两仓均为Top-3类别，OP_CHECK_IF语义反转是共性高频模式
- BUF(buffer/内存)两仓均为高危类别，workspace计算溢出和DataCopy越界是共性
- BUILD(构建配置)两仓均受cmake平行实现结构性问题困扰
- CPASTE(复制粘贴)两仓均存在从已有算子复制代码后标识符未替换的模式

差异:
- TKEY(tilingKey一致性): dev仓独有的显著类别(~20条)，主仓仅零星出现。反映dev分支tilingKey重构活跃度更高
- REVERT(功能回退): dev仓35个Revert(占非merge的1.2%)，主仓无显著Revert事件。反映dev分支"快速合入-快速回退"的开发模式
- PLAT(硬件平台): dev仓更显著(~25条)，主仓不突出。反映dev分支承担新平台(A5/910_95)首发适配
- SYNC(同步/流水线): dev仓更突出(~35条 vs 主仓较少)，MLA/sparse attention等新算子引入更多流水线同步挑战
- 缺陷密度: dev仓27.9%(788/2820) vs 主仓18.4%(243/1323)，dev分支缺陷率高约50%，符合开发分支迭代速度更快、质量门禁相对宽松的预期

---

### 附录: 数据来源

- 仓库路径: /Users/shanshan/repo/cann/ops-transformer-dev/
- 分析周期: 2025-08至2026-01
- 总提交: 2822 (非merge: 2820)
- 确认缺陷: 788条 (阶段1关键词初筛)
- 代码缺陷: 528条 (阶段2全量diff分析确认)
- 排除: 435条 (阶段1排除178条纯文档 + 阶段2排除257条非代码缺陷/重复)
- Revert事件: 35条 (阶段3专项分析)
- 热点文件: Top-30 (阶段4分析)
- 缺陷类别: 15个 + 4个跨类别风险 (阶段5归纳)
- 审查规则: 42条 (10条P0 + 27条P1 + 8条P2 + 6条P3自动化，含SYS规则)
