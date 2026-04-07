## HCCL/HCOMM项目缺陷模式与审查规则

数据来源: hccl(hcomm仓库, 84条缺陷中提炼48条规则) + hccl-dev(10条缺陷中提炼9条规则) + hcomm-dev(162条缺陷中提炼40条规则)
合并后: 去重整合为统一规则集

### 严重等级定义

| 等级 | 含义 | 影响范围                                 |
|------|------|------------------------------------------|
| P0   | 致命 | CoreDump/Hang/数据损坏/内存越界/UAF      |
| P1   | 严重 | 特定配置崩溃/功能不可用/静默精度错误/OOM |
| P2   | 一般 | 构建失败/边界条件异常/日志误导           |
| P3   | 建议 | 代码质量/可维护性/潜在隐患               |

### 规则总览

| 规则ID    | 类别     | 严重等级 | 规则名                                   | 来源                 |
|-----------|----------|----------|------------------------------------------|----------------------|
| ALG-01    | 算法正确性 | P0     | 变量名遮蔽导致成员未赋值                 | [hccl]               |
| ALG-02    | 算法正确性 | P0     | 变量自比较(tautological compare)         | [hccl]               |
| ALG-03    | 算法正确性 | P1     | 结构体/参数赋值遗漏                      | [hccl]               |
| ALG-04    | 算法正确性 | P1     | 同族executor/子类一致性缺陷              | [hccl]               |
| ALG-05    | 算法正确性 | P1     | 边界条件缺失                             | [hccl]               |
| ALG-06    | 算法正确性 | P1     | Get函数不应有Set副作用                   | [hccl]               |
| ALG-07    | 算法正确性 | P1     | 多阶段/多版本流程变量混淆               | [hccl]               |
| ALG-08    | 算法正确性 | P1     | 非均匀集合通信per-rank偏移逻辑           | [hccl]               |
| BUILD-01  | 构建     | P2       | CMakeLists源文件遗漏                     | [hcomm-dev][hccl-dev]|
| BUILD-02  | 构建     | P2       | 条件编译/编译目标不兼容                  | [hcomm-dev]          |
| BUILD-03  | 构建     | P1       | ABI兼容性/符号导出                       | [hcomm-dev]          |
| BUILD-04  | 构建     | P1       | 环境/配置适配遗漏                        | [hcomm-dev]          |
| BUILD-05  | 构建     | P2       | CMake版本兼容性与参数格式                | [hccl-dev]           |
| BUILD-06  | 构建     | P2       | 大变更合入须有充分pre-merge验证          | [hccl]               |
| CONC-01   | 并发     | P0       | 共享数据写后更新标志须有内存屏障         | [hccl][hcomm-dev]    |
| CONC-02   | 并发     | P0       | Record/Wait同步顺序                      | [hccl]               |
| CONC-03   | 并发     | P0       | publish ordering                         | [hccl]               |
| CONC-04   | 并发     | P0       | 并发delete须加锁保护                     | [hccl][hcomm-dev]    |
| CONC-05   | 并发     | P1       | atomic变量的所有读取必须用.load()        | [hccl]               |
| CONC-06   | 并发     | P1       | 单例Init须区分once-only和every-time      | [hccl]               |
| CONC-07   | 并发     | P1       | thread_local变量须在线程入口点统一设置   | [hccl]               |
| CONC-08   | 并发     | P0       | 全局/静态变量无锁访问                    | [hcomm-dev]          |
| CONC-09   | 并发     | P1       | TOCTOU竞态                               | [hcomm-dev]          |
| INIT-01   | 初始化   | P1       | 多分支初始化路径不对称                   | [hcomm-dev]          |
| INIT-02   | 初始化   | P0       | 成员变量未初始化/默认值错误              | [hcomm-dev]          |
| INIT-03   | 初始化   | P0       | 执行顺序/依赖错误                        | [hcomm-dev]          |
| INIT-04   | 初始化   | P1       | offload/lambda变量作用域泄漏             | [hcomm-dev]          |
| BRANCH-01 | 分支     | P1       | 枚举/类型分支遗漏                        | [hcomm-dev][hccl-dev]|
| BRANCH-02 | 分支     | P1       | V2/offload适配路径缺失                   | [hcomm-dev]          |
| BRANCH-03 | 分支     | P0       | 条件方向错误/条件编译覆盖运行时分支      | [hcomm-dev]          |
| BRANCH-04 | 分支     | P1       | 设备类型判断遗漏组合条件                 | [hccl-dev]           |
| TYPE-01   | 类型     | P0       | 枚举值语义误用                           | [hcomm-dev]          |
| TYPE-02   | 类型     | P1       | 类型隐式截断/溢出                        | [hcomm-dev][hccl]    |
| TYPE-03   | 类型     | P0       | 参数对象混淆/引用错误                    | [hcomm-dev]          |
| TYPE-04   | 类型     | P0       | map::at()对未收录key抛异常               | [hcomm-dev]          |
| TYPE-05   | 类型     | P1       | 通信引擎枚举变体混淆                     | [hccl-dev]           |
| TYPE-06   | 类型     | P1       | 平台字符串字面量拼写错误                 | [hccl-dev]           |
| TYPE-07   | 类型     | P2       | API参数名不同步                          | [hccl-dev]           |
| CFG-01    | 配置     | P0       | 跨版本协议OpCode修改须保持向后兼容       | [hccl]               |
| CFG-02    | 配置     | P1       | 设备类型分支须全面覆盖                   | [hccl]               |
| CFG-03    | 配置     | P2       | 常量不应承担多重语义                     | [hccl]               |
| CFG-04    | 配置     | P1       | v1/v2接口分发完整性                      | [hccl]               |
| CFG-05    | 配置     | P1       | 结构体/类字段变更未同步到所有使用方      | [hccl-dev]           |
| CFG-06    | 配置     | P2       | 公共API函数名与头文件声明不匹配          | [hccl-dev]           |
| RES-01    | 资源     | P1       | early return跳过必要初始化/清理          | [hccl]               |
| RES-02    | 资源     | P1       | map/cache的key必须能唯一标识资源         | [hccl]               |
| RES-03    | 资源     | P1       | context切换附近的内存操作须审查执行context | [hccl]             |
| RES-04    | 资源     | P1       | 析构路径须完整                           | [hccl]               |
| RES-05    | 资源     | P0       | 资源释放顺序错误                         | [hcomm-dev]          |
| RES-06    | 资源     | P1       | 资源重复管理/泄漏                        | [hcomm-dev]          |
| RES-07    | 资源     | P1       | 错误路径资源泄漏                         | [hcomm-dev]          |
| RES-08    | 资源     | P0       | 禁止暴露局部容器内部指针                 | [hccl]               |
| RES-09    | 资源     | P1       | IPC共享内存注册须检查重复注册            | [hccl]               |
| RES-10    | 资源     | P1       | 可选初始化路径的成员指针使用前必须判空   | [hccl]               |
| LOG-01    | 日志     | P2       | 格式化字符串占位符须与实参类型匹配       | [hccl][hcomm-dev]    |
| LOG-02    | 日志     | P3       | 日志前缀/tag必须与所在类名/函数名匹配   | [hccl]               |
| LOG-03    | 日志     | P2       | 快速路径必须与标准路径保持功能对等       | [hccl]               |
| LOG-04    | 日志     | P3       | 日志级别不当                             | [hcomm-dev]          |
| LOG-05    | 日志     | P3       | 日志洪泛/变量引用错误                    | [hcomm-dev]          |
| ERR-01    | 错误处理 | P2       | 异常上报操作须受前置条件保护             | [hccl]               |
| ERR-02    | 错误处理 | P2       | 项目有统一异常处理宏时不允许裸try-catch  | [hccl]               |
| ERR-03    | 错误处理 | P0       | C接口函数不应抛C++异常                   | [hccl]               |
| ERR-04    | 错误处理 | P2       | CHK_RET调用顺序须与业务逻辑匹配         | [hccl]               |
| ERR-05    | 错误处理 | P2       | 预期内返回值不应用error日志级别          | [hccl]               |
| ERR-06    | 错误处理 | P0       | 空指针检查顺序错误                       | [hcomm-dev]          |
| ERR-07    | 错误处理 | P1       | API入口参数校验缺失                      | [hcomm-dev]          |
| ERR-08    | 错误处理 | P1       | 可选指针/新增参数校验缺失                | [hcomm-dev]          |
| STATE-01  | 状态     | P1       | cache key须覆盖所有影响执行路径的状态维度 | [hccl]              |
| STATE-02  | 状态     | P1       | 缓存复用前须逐字段确认是否需要刷新       | [hccl][hcomm-dev]   |
| STATE-03  | 状态     | P1       | 函数副作用污染成员变量                   | [hccl]               |
| STATE-04  | 状态     | P1       | 运行时分支条件与缓存key必须同数据源      | [hccl]               |
| STATE-05  | 状态     | P1       | 超时值硬编码/配置不一致                  | [hcomm-dev]          |
| STATE-06  | 状态     | P2       | 环境变量/运行时状态一致性                | [hcomm-dev]          |
| QUAL-01   | 质量     | P1       | 算法名称/变量名拼写错误                  | [hcomm-dev]          |
| QUAL-02   | 质量     | P2       | 全局重命名同步不完整                     | [hcomm-dev]          |
| QUAL-03   | 质量     | P3       | 全代码库拼写错误渗入API                  | [hcomm-dev]          |
| QUAL-04   | 质量     | P1       | 复制粘贴后变量名未替换                   | [hccl-dev]           |
| API-01    | 接口     | P2       | 抢跑依赖未发布API                        | [hcomm-dev]          |
| API-02    | 接口     | P1       | 返回值语义差异未适配                     | [hcomm-dev]          |
| API-03    | 接口     | P2       | 封装层能力不完整                         | [hcomm-dev]          |
| HW-01     | 硬件     | P0       | 硬件协议乘数因子遗漏                     | [hcomm-dev]          |
| HW-02     | 硬件     | P1       | 设备类型特殊路径未处理                   | [hcomm-dev]          |
| HW-03     | 硬件     | P0       | 硬件寄存器/地址映射宏定义冲突            | [hcomm-dev]          |
| CPP-01    | C++语言  | P1       | 跨SO边界虚析构不应使用= default          | [hccl]               |
| SYS-01    | 系统性   | P1       | 缺陷修复提交须额外审查是否引入新缺陷     | [hccl]               |
| SYS-02    | 系统性   | P2       | 修复一处须搜索重复代码段                 | [hccl][hcomm-dev]    |
| SYS-03    | 系统性   | P3       | God Object文件修改须加强审查             | [hccl][hcomm-dev]    |
| SYS-04    | 系统性   | P1       | 返回值系统性忽略                         | [hccl]               |
| SYS-05    | 系统性   | P2       | 编译器可捕获的缺陷须在CI中强制开启       | [hccl]               |
| SYS-06    | 系统性   | P1       | 热点文件已知高危风险                     | [hccl][hcomm-dev]    |
| SYS-07    | 系统性   | -        | 并发安全缺乏系统性设计                   | [hcomm-dev]          |
| SYS-08    | 系统性   | -        | 公共API/ABI设计债务                      | [hcomm-dev]          |
| SYS-09    | 系统性   | -        | 大爆炸提交掩盖关键变更                   | [hcomm-dev]          |

---

### 类别: 算法正确性 (ALG)

集合通信算法实现中逻辑错误的各种表现形式，是HCCL项目最高频的缺陷类别(hccl仓库25.0%)。

#### ALG-01: 变量名遮蔽导致成员未赋值 [hccl]

严重等级: P0

缺陷描述: 构造函数或成员函数中，局部变量与成员变量同名，赋值写入局部变量而非成员变量。成员变量保持未初始化状态，后续使用产生未定义行为。

典型代码示例:

```cpp
// 缺陷代码 -- src/platform/common/hccl_ip_address.cc
HcclIpAddress::HcclIpAddress(const Eid &eidInput)
{
    family = AF_INET6;        // local var shadows member, this->family unset
}

// 修复代码
HcclIpAddress::HcclIpAddress(const Eid &eidInput)
{
    this->family = AF_INET6;  // explicit member access
}
```

审查检查方法:
- 编译选项开启 `-Wshadow -Werror=shadow`，CI强制执行
- 构造函数体内赋值语句逐一确认目标是否为成员变量
- 优先使用初始化列表代替构造函数体内赋值

关联commit: `ef766683`

---

#### ALG-02: 变量自比较(tautological compare) [hccl]

严重等级: P0

缺陷描述: 条件表达式中同一变量与自身比较(如 `x > x`)，结果恒为常量，边界保护形同虚设。

典型代码示例:

```cpp
// 缺陷代码 -- src/platform/task/dispatcher_aicpu.cc
CHK_PRT_RET(sqeContextBuffer->tailSqeIdx > sqeContextBuffer->tailSqeIdx,  // tautology, always false
    HCCL_ERROR("[DispatcherAicpu][MemcpyRtsq] tailSqeIdx[%u] > HCCL_SQE_MAX_CNT[%u]",
        sqeContextBuffer->tailSqeIdx, HCCL_SQE_MAX_CNT),
    HCCL_E_INTERNAL);

// 修复代码
CHK_PRT_RET(sqeContextBuffer->tailSqeIdx > HCCL_SQE_MAX_CNT,
    HCCL_ERROR("[DispatcherAicpu][MemcpyRtsq] tailSqeIdx[%u] > HCCL_SQE_MAX_CNT[%u]",
        sqeContextBuffer->tailSqeIdx, HCCL_SQE_MAX_CNT),
    HCCL_E_INTERNAL);
```

审查检查方法:
- 编译选项开启 `-Wtautological-compare`
- CHK_PRT_RET/CHK_RET条件表达式须与错误消息字符串交叉验证: 消息说比较A和B，条件也应是A和B的比较

关联commit: `e1880dc1`

---

#### ALG-03: 结构体/参数赋值遗漏 [hccl]

严重等级: P1

缺陷描述: 填充参数结构体时遗漏了某个字段的赋值。特别是成对字段(dataType/outputDataType、sendType/recvType)，只赋值了一个而遗漏另一个。遗漏字段保持零值或默认值，导致运行时行为错误。

典型代码示例:

```cpp
// 缺陷代码 -- src/legacy/framework/communicator/aicpu/aicpu_utils.cc
// FillKernelParam only sets outputDataType, missing dataType
kernelParam_->op.algOperator.outputDataType = HcclDataTypeToDataType(data->outputDataType);
CHECK_DATA_TYPE(kernelParam_->op.algOperator.outputDataType);
// dataType not assigned! MC2 precision error when input/output types differ

// 修复代码
kernelParam_->op.algOperator.dataType = HcclDataTypeToDataType(data->dataType);
CHECK_DATA_TYPE(kernelParam_->op.algOperator.dataType);
kernelParam_->op.algOperator.outputDataType = HcclDataTypeToDataType(data->outputDataType);
CHECK_DATA_TYPE(kernelParam_->op.algOperator.outputDataType);
```

审查检查方法:
- 填充参数结构体时，列出结构体全部字段，逐一对比赋值语句，标记未赋值字段
- 成对字段(input/output、src/dst、send/recv前缀)确保不遗漏其一
- if/else分支中填充参数时，检查每个分支是否都做了完整赋值

关联commit: `7bc9e850`(dataType遗漏), `1d171a92`(ipcMemDataSize遗漏)

---

#### ALG-04: 同族executor/子类一致性缺陷 [hccl]

严重等级: P1

缺陷描述: 继承体系中N个子类共享某段逻辑，但有1个子类遗漏。典型表现: N-1个executor有某个调用而1个没有; 同一业务条件在不同子类中判断逻辑不一致。

典型代码示例:

```cpp
// 缺陷代码 -- coll_all_to_all_v_direct_fullmesh_executor.cc
// All other executors have LaunchTaskExtend at the end of Orchestrate
// AlltoAllDirectFullmesh is the only one missing it -> aicpu cache malfunction

// 修复代码
CHK_RET(LaunchTaskExtend(dispatcher_, param.stream, algResResp_->slaveStreams));
```

审查检查方法:
- 新增executor子类时，用同族其他executor作为checklist逐项对比
- N-1个子类有某段逻辑而1个没有，应标记为"遗漏"而非"不需要"，除非有明确注释说明原因
- 日志字符串中的类名/tag前缀必须与当前文件名或类名匹配(检测copy-paste错误)

关联commit: `f7183c87`(LaunchTaskExtend遗漏), `bb681c5c`(preloadCopyOpt条件不一致), `71fd0b86`(numBlocks校验不一致), `12f3680c`(日志类名copy-paste错误)

---

#### ALG-05: 边界条件缺失 [hccl]

严重等级: P1

缺陷描述: 算法选择或执行路径未覆盖所有有效配置。典型场景: pipeline算法假设至少3卡但未排除2卡场景; 参数校验上限值与实际类型范围不一致。

典型代码示例:

```cpp
// 缺陷代码 -- src/algorithm/impl/operator/all_reduce_operator.cc
// Pipeline needs >=3 devices, but selector allows 2-device case
if (isOpbase && algType_.algoLevel1 == AlgTypeLevel1::ALG_LEVEL1_PIPELINE) {
    algName = "AllReduceDeterPipelineExecutor";  // 2-device enters pipeline path
}

// 修复代码
if (isOpbase && algType_.algoLevel1 == AlgTypeLevel1::ALG_LEVEL1_PIPELINE
    && deviceNumPerAggregation_ > DEVICE_TWO) {
    algName = "AllReduceDeterPipelineExecutor";
}
```

审查检查方法:
- 算法选择分支必须覆盖所有设备拓扑配置(1卡/2卡/多卡)，每种配置标注预期行为
- 参数校验上限值须与参数实际类型范围一致(u8上限255、u16上限65535)
- Builder/fluent API构造对象时确认所有必要setter都被调用

关联commit: `666b6ab5`(2卡边界), `7990e3f3`(repeat上限), `8959e766`(selector遗漏SetRankSize)

---

#### ALG-06: Get函数不应有Set副作用 [hccl]

严重等级: P1

缺陷描述: 命名为Get*/Query*的查询函数内部调用了Set/Update/Insert操作，违反查询函数的纯净性约束。调用方无法预期查询操作会修改系统状态。

审查检查方法:
- Get*前缀函数体内不应包含Set/Update/Insert/Modify调用
- 方向对称性检查: `dst.*GetSource()` 或 `src.*GetTarget()` 模式标记为可疑

关联commit: `3323cecd`(GetEndpointDesc调用SetEndpointToIface), `e4f59213`(查询函数修改成员变量curAlgName)

---

#### ALG-07: 多阶段/多版本流程变量混淆 [hccl]

严重等级: P1

缺陷描述: 多阶段流水线或多版本兼容代码中，阶段标识变量名相似导致混淆(intra vs inter、prepare vs launch、v1 vs v2)。典型表现: 用错误的阶段变量作为条件判断。

审查检查方法:
- 同一函数通过bool参数区分完全不同行为时，应拆分为两个独立函数
- 流水线多阶段的变量名须有明确区分前缀，审查条件是否指向正确阶段
- 多版本兼容路径中检查初始化顺序是否符合版本优先级

关联commit: `9bcb1bdc`(intra/inter混淆), `af109f94`(prepare/launch阶段字段未区分), `1a840065`(1.0/2.0初始化顺序)

---

#### ALG-08: alltoallv等非均匀集合通信的per-rank偏移逻辑 [hccl]

严重等级: P1

缺陷描述: 非均匀集合通信(alltoallv/allgatherv)中，每个rank的发送/接收偏移量和数据量不同。repeat模式下偏移量须逐轮累加，不能用通用标量运算代替。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/device/aicpu_kfc/decoupler/comm_kfc_aicpu_server.cc
// alltoallv repeat: offset not accumulated per round
for (u32 i = 0U; i < repeatCnt; ++i) {
    FormatOpData(msg, extMsg, i, data);  // offset unchanged
}

// 修复代码
for (u32 i = 0U; i < repeatCnt; ++i) {
    FormatOpData(msg, extMsg, rankNum_, i, data);
    // FormatOpData accumulates per-rank offsets:
    // extMsg.sendOffset[j] += extMsg.sendCounts[j];
    // extMsg.recvOffset[j] += extMsg.recvCounts[j];
}
```

审查检查方法:
- alltoallv/allgatherv等非均匀算子的repeat/分片逻辑须逐rank独立处理偏移
- 检查循环体内是否正确更新了所有per-rank的状态变量

关联commit: `fab6dbb7`

---

### 类别: 构建/编译/链接 (BUILD)

hcomm-dev中频次最高的缺陷类别(14.8%)。根本原因是构建系统复杂度随多编译目标(host/device/kernel/daemon)和多构建模式(open/closed/HCCD/CCL_KERNEL)增长而爆炸。

#### BUILD-01: CMakeLists源文件遗漏 [hcomm-dev][hccl-dev]

严重等级: P2

缺陷描述: 新增.cc文件后忘记在CMakeLists.txt中添加，导致功能不被编译链接。另有源文件同时被添加到两个CMakeLists.txt中导致符号重复定义。hccl-dev中也出现构建目标缺少依赖源文件，链接阶段undefined symbol。

典型代码示例:

```cmake
# 缺陷(hcomm-dev) -- alltoallv continuous pipeline .cc files missing
set(SOURCES
    ...
    coll_all_to_all_v_executor.cc
    # missing: alltoallv_continuous_pipeline.cc
    # missing: coll_all_to_all_v_continuous_pipeline_executor.cc
    ...
)

# 缺陷(hccl-dev) -- scatter_aicpu_kernel missing 5 dependency .cc files
add_library(scatter_aicpu_kernel SHARED
    ${CMAKE_CURRENT_SOURCE_DIR}/common/utils.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/common/adapter_acl.cc
    # missing: config_log.cc, sal.cc, log.cc,
    #          adapter_error_manager_pub.cc, alg_env_config.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/ops/aicpu/kernel_launch.cc
    ...
)
```

审查检查方法:
- 新增.cc文件的PR是否同步修改了CMakeLists.txt?
- 同一.cc文件是否出现在多个CMakeLists.txt中(符号重复)?
- 新增add_library时，逐个检查源文件中的#include和函数调用是否有对应.cc文件在target中
- 关注间接依赖: A.cc include了B.h，B.h的实现在B.cc中，B.cc是否在target里
- CI是否覆盖所有编译目标的构建验证?

关联commit: `8c424d41`, `20125a56`(hcomm-dev), `beb0ed54`(hccl-dev)

---

#### BUILD-02: 条件编译/编译目标不兼容 [hcomm-dev]

严重等级: P2

缺陷描述: 在特定编译目标下引入不可用的外部依赖，或条件编译块覆盖不完整。7条案例。常见模式: 函数调用了ACL API但CCL_KERNEL_AICPU/HCCD目标下ACL不可用。

典型代码示例:

```cpp
// 缺陷 -- ParseCannVersion()调用ACL API，HCCD目标下不可用
HcclResult ParseCannVersion() {
    const char *version = aclGetVersion();  // link error under CCL_KERNEL_AICPU/HCCD
    ...
}

// 修复 -- 条件编译包裹
HcclResult ParseCannVersion() {
#if !defined(CCL_KERNEL_AICPU) && !defined(HCCD)
    const char *version = aclGetVersion();
    ...
#else
    return HCCL_E_NOT_SUPPORT;
#endif
}
```

审查检查方法:
- 新增外部API依赖时，是否检查了所有编译目标(host/device/kernel/daemon)的可用性?
- 条件编译块(BUILD_OPEN_PROJECT/KERNEL_MODE等)的open和closed路径是否对称?

关联commit: `89fd99e0`, `947460b2`

---

#### BUILD-03: ABI兼容性/符号导出 [hcomm-dev]

严重等级: P1

缺陷描述: 公共头文件中使用内部类型、导出C++符号到C linkage、动态库导出函数使用std::string参数等。5条案例。

典型代码示例:

```cpp
// 缺陷 -- 公共头文件hcomm_primitives.h中uint32_t改为内部typedef u32
// 外部用户编译时找不到u32定义，ABI破坏
void HcommFunc(u32 param);  // pkg_inc/ uses internal typedef

// 修复 -- 公共头文件仅使用标准C/C++类型
void HcommFunc(uint32_t param);

// 另一缺陷 -- 动态库导出函数用std::string&参数
HcclResult HcomSelectAlg(const char *tag, std::string &algName);  // ABI-incompatible

// 修复 -- 改为C兼容POD类型
HcclResult HcomSelectAlg(const char *tag, char *algName, uint32_t algNameLen);
```

审查检查方法:
- 公共头文件(pkg_inc/)中是否仅使用标准C/C++类型，禁止内部typedef?
- 动态库导出函数(extern "C")的参数和返回值是否为POD类型?
- stub符号与实际函数签名是否匹配?

关联commit: `30e25e50`, `7fccc101`, `e6d12932`

---

#### BUILD-04: 环境/配置适配遗漏 [hcomm-dev]

严重等级: P1

缺陷描述: 环境变量解析、设备可见性配置等边界条件处理不足。4条案例。

典型代码示例:

```cpp
// 缺陷 -- ASCEND_RT_VISIBLE_DEVICES只配1个设备时backup设备不在可见列表
HcclResult ret = hrtGetDeviceIndexByPhyId(backupDevId, &logicId);
// backupDevId not in visible list, call fails

// 修复 -- 移到条件分支内部按需调用
if (needBackupDevice) {
    HcclResult ret = hrtGetDeviceIndexByPhyId(backupDevId, &logicId);
    CHK_RET(ret);
}
```

审查检查方法:
- 涉及设备可见性的API调用是否考虑了单设备/部分设备可见的场景?
- 环境变量缺失时的默认行为是否安全?

关联commit: `f6e1c1c2`

---

#### BUILD-05: CMake版本兼容性与参数格式 [hccl-dev]

严重等级: P2

缺陷描述: CMake脚本中使用了高版本特性(如CMAKE_CURRENT_FUNCTION_LIST_DIR需要3.17+)，或`-D`参数后加空格导致解析失败。

典型代码示例:

```cmake
# 缺陷(commit 1f13573a, cmake/func.cmake)
# Problem 1: CMAKE_CURRENT_FUNCTION_LIST_DIR requires CMake 3.17+
-P "${CMAKE_CURRENT_FUNCTION_LIST_DIR}/_pack_stage.cmake"

# Problem 2: space after -D causes parse failure
set(manifest_arg "-D _MANIFEST_FILE=${staging_dir}/${ARG_MANIFEST}")

# 修复
# Save path at file scope (compatible with older CMake)
set(_FUNC_CMAKE_DIR "${CMAKE_CURRENT_LIST_DIR}")
-P "${_FUNC_CMAKE_DIR}/_pack_stage.cmake"

# No space after -D
set(manifest_arg -D_MANIFEST_FILE=${staging_dir}/${ARG_MANIFEST})
```

审查检查方法:
- 对照项目cmake_minimum_required确认使用的CMake特性版本要求
- `-D`后紧跟变量名，不加空格
- 函数内引用路径变量时，确认变量在函数作用域内可用

关联commit: `1f13573a`

---

#### BUILD-06: 大变更合入须有充分的pre-merge验证 [hccl]

严重等级: P2

缺陷描述: 24文件的构建系统重构合入1小时后被revert; 硬件常量修改同时修改了UT期望值适配变更。两次PR描述均为空模板。

审查检查方法:
- PR描述为空模板的变更不应合入主干
- 同时修改源码和对应UT期望值时，审查UT修改的合理性 -- UT应独立验证而非适配变更
- 硬件常量修改须注明取值依据(spec文档引用)
- 构建系统大变更(>5个文件)应分步合入，每步独立验证
- 一个commit只做一件事: 逻辑变更和风格变更必须分离

关联commit: `72cdf80e`(revert fb56d64b, CCU loop count常量修改无spec依据, 合入6小时后撤回), `753ba8c2`(revert 05b38411, 24文件构建重构, 合入1小时后撤回)

---

### 类别: 并发/线程安全 (CONC)

通信库最核心的缺陷类型，涵盖竞态、死锁、原子性违反、同步时序。hccl仓库10次(11.9%)，hcomm-dev仓库11条(6.8%)。18个热点文件中15个存在并发安全问题。

#### CONC-01: 共享数据写后更新标志须有内存屏障 [hccl][hcomm-dev]

严重等级: P0

缺陷描述: "先写数据、后更新标志位"的生产者-消费者模式中，缺少memory fence。CPU或编译器可能重排store指令，导致消费者看到标志位更新但数据还未写入。两个仓库均出现此缺陷模式。

典型代码示例:

```cpp
// 缺陷代码 -- src/platform/resource/mem/hdc.cc
// HDCommunicate::Write
CHK_RET(memcpy_s(hostMem_.data() + offset, hostMem_.size(), value, length));
// missing memory fence!
u32 tail = *tailCntAddr_;
tail++;
*tailCntAddr_ = tail;   // consumer may see tail update before data is ready

// 修复代码
CHK_RET(memcpy_s(hostMem_.data() + offset, hostMem_.size(), value, length));
std::atomic_thread_fence(std::memory_order_seq_cst);  // ensure data visibility
u32 tail = *tailCntAddr_;
tail++;
*tailCntAddr_ = tail;
```

审查检查方法:
- "先写数据、后更新标志位"模式必须在两步之间插入memory fence
- 跨线程/跨进程的共享内存通信，标志位更新前须有acquire-release语义保证

关联commit: `0947b660`(hccl), `a014e919`(hcomm-dev)

---

#### CONC-02: Record/Wait同步顺序 [hccl]

严重等级: P0

缺陷描述: 集合通信broadcast中，中继rank先Record通知root"数据已取走"，然后才Wait其他rank完成。root可能在下游未完成时就覆盖buffer。

典型代码示例:

```cpp
// 缺陷代码 -- src/algorithm/base/alg_aiv_template/aiv_broadcast_910b_bigdata.h
// Relay rank: Record before Wait
Record(tag, root, AivNotifyType::DataSignal, 0, ifPingpong);  // notify root first
PipeBarrier<PIPE_ALL>();
for (uint32_t remoteRank = 0; remoteRank < rankSize_; remoteRank += 1) {
    if (remoteRank != root && remoteRank != rank_) {
        Wait(tag, remoteRank, AivNotifyType::DataSignal, 0, ifPingpong);  // wait later
    }
}

// 修复代码 -- Wait all downstream first, Record root last
for (uint32_t remoteRank = 0; remoteRank < rankSize_; remoteRank += 1) {
    if (remoteRank != root && remoteRank != rank_) {
        Wait(tag, remoteRank, AivNotifyType::DataSignal, 0, ifPingpong);
        PipeBarrier<PIPE_ALL>();
    }
}
PipeBarrier<PIPE_ALL>();
Record(tag, root, AivNotifyType::DataSignal, 0, ifPingpong);  // notify root last
```

审查检查方法:
- Record/Wait顺序原则: 先Wait依赖方完成，再Record通知上游
- 集合通信中root/coordinator应是最后退出的角色

关联commit: `a0f05be2`

---

#### CONC-03: publish ordering [hccl]

严重等级: P0

缺陷描述: 多线程环境下先重置状态标记再清理资源。其他线程看到状态已重置就认为可以使用，但资源实际还未清理完成。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/device/framework/aicpu_communicator.cc
ret = ResetOpRetryException(param.opType);       // reset state first
dfxExtendInfo_.pollStatus = PollStatus::kDefault; // other threads see state reset
dfxExtendInfo_.cqeStatus = dfx::CqeStatus::kDefault;

// 修复代码 -- cleanup resources first, then reset state
dfxExtendInfo_.pollStatus = PollStatus::kDefault;
dfxExtendInfo_.cqeStatus = dfx::CqeStatus::kDefault;
ret = ResetOpRetryException(param.opType);        // reset state last
```

审查检查方法:
- publish ordering原则: 先完成实际清理/初始化工作，再修改对外可见的状态标志
- 状态重置函数调用应在所有资源清理操作之后

关联commit: `75659d24`

---

#### CONC-04: 并发delete须加锁保护 [hccl][hcomm-dev]

严重等级: P0

缺陷描述: 多线程可能并发调用含delete操作的销毁函数，无锁保护导致double-free。常见模式: check-then-act(先判空再delete)在并发下不安全。两个仓库均有此缺陷。

典型代码示例:

```cpp
// 缺陷代码 -- src/platform/resource/dispatcher_ctx/dispatcher_ctx.cc
HcclResult DispatcherCtx::Destroy()
{
    if (dispatcher_ != nullptr) {    // two threads may pass this check
        delete dispatcher;           // double-free
        dispatcher_ = nullptr;
    }
}

// 修复代码
HcclResult DispatcherCtx::Destroy()
{
    const std::lock_guard<std::mutex> lock(destroyMutex_);
    if (dispatcher_ != nullptr) {
        delete dispatcher;
        dispatcher_ = nullptr;
    }
}
```

```cpp
// 缺陷(hcomm-dev) -- reqHandle freed while async thread references it
free(reqHandle);      // async thread may be referencing -> UAF
reqHandle = NULL;

// 修复 -- unified deletion under lock
std::lock_guard<std::mutex> lock(reqMutex);
HdcAsyncDelResponse(reqHandle);
```

审查检查方法:
- delete后置nullptr的模式在多线程环境下不安全，必须加锁
- 审查所有Destroy/Cleanup函数是否可能被并发调用
- 有配套mutex的数据结构，其free/delete是否在锁保护下进行?

关联commit: `36994739`(hccl), `7be3b8b8`, `cd056a5e`(hcomm-dev)

---

#### CONC-05: atomic变量的所有读取必须用.load() [hccl]

严重等级: P1

缺陷描述: `std::atomic`变量用隐式转换(如 `==` 操作符)读取时，在某些编译器/优化级别下可能不产生原子load指令。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/communicator/impl/hccl_communicator_host.cc
if (g_enableBackupLinkCommCount == 0) {  // implicit non-atomic read

// 修复代码
if (g_enableBackupLinkCommCount.load() == 0) {  // explicit atomic load
```

审查检查方法:
- 所有`std::atomic`变量的读取点必须使用`.load()`显式调用

关联commit: `6b354394`

---

#### CONC-06: 单例/引用计数Init方法须区分once-only和every-time [hccl]

严重等级: P1

缺陷描述: 单例模式的Init方法中，部分逻辑只需首次执行(如注册回调)，部分逻辑每次创建通信域都需执行(如设置白名单)。引用计数>1时直接跳过整个Init，导致后续通信域缺少必要配置。

审查检查方法:
- 单例Init方法中明确标注哪些操作是once-only、哪些是every-time
- 引用计数递增路径也须检查是否有必须每次执行的逻辑

关联commit: `eb9be21b`

---

#### CONC-07: thread_local变量须在线程入口点统一设置 [hccl]

严重等级: P1

缺陷描述: `thread_local`变量(如`gDispatcherCtx`)的设置逻辑被嵌在某个业务函数深处，新线程如果没有经过该函数就调用了依赖此变量的其他函数，访问到未初始化的thread_local值。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/device/framework/aicpu_communicator.cc
// SetDispatcherCtx buried inside RegisterOpInfo
HcclResult HcclCommAicpu::RegisterOpInfo(void* opInfo, u32 size)
{
    CHK_RET(SetDispatcherCtx(dispatcherCtx_));   // thread_local set here
    CHK_RET(taskExecption_.RegisterOpInfo(opInfo, size));
}
// But HcommAcquireComm on another thread doesn't go through RegisterOpInfo

// 修复代码 -- extract to explicit method, call at thread entry point
HcclResult HcclCommAicpu::SetDispatcherCtxOnThread()
{
    CHK_RET(SetDispatcherCtx(dispatcherCtx_));
    return HCCL_SUCCESS;
}

// hccl_api_data.cc -- explicit call at thread entry
int32_t HcommAcquireComm(const char* commId)
{
    HcclCommAicpu *hcclComm = AicpuHcclProcess::AicpuGetCommbyGroup(commId);
    CHK_RET(hcclComm->SetDispatcherCtxOnThread());
    ...
}
```

审查检查方法:
- thread_local变量的设置须作为线程入口的第一步操作，不应埋在业务函数内部
- 新增线程入口时须审查是否依赖了已有的thread_local变量
- thread_local变量的所有使用点须确认该线程已经过设置路径

关联commit: `a6e4d199`

---

#### CONC-08: 全局/静态变量无锁访问 [hcomm-dev]

严重等级: P0

缺陷描述: static全局变量在多线程/多流场景下被并发读写。4条案例。

典型代码示例:

```cpp
// 缺陷 -- static global array written concurrently under MC2 multi-stream
static uint8_t g_expectPrepareId[MAX_QUE_NUM];  // multi-thread write same slot
// Thread A: g_expectPrepareId[queueId] = idA;
// Thread B: g_expectPrepareId[queueId] = idB;  // overwrites -> msg ID mismatch -> hang

// 修复 -- thread_local
static thread_local uint8_t g_expectPrepareId[MAX_QUE_NUM];
```

审查检查方法:
- static全局变量在多线程场景下是否需要thread_local或显式同步?
- "循环体外持写锁、循环内做IO/RPC"的模式是否存在持锁粒度过粗风险?

关联commit: `341e7893`

---

#### CONC-09: TOCTOU竞态 [hcomm-dev]

严重等级: P1

缺陷描述: 持锁检查后释放锁再操作，检查结果在释放锁后失效。2条案例。

典型代码示例:

```cpp
// 缺陷 -- check under lock, unlock, then create -> race window
{
    std::lock_guard<std::mutex> lock(groupMutex_);
    if (groupMap_.find(groupName) != groupMap_.end()) {
        return HCCL_E_EXIST;
    }
}  // unlock
// window: another thread also passes the check
CHK_RET(CreateGroup(groupName));  // both threads create same group

// 修复 -- check and create within same critical section
{
    std::lock_guard<std::mutex> lock(groupMutex_);
    if (groupMap_.find(groupName) != groupMap_.end()) {
        return HCCL_E_EXIST;
    }
    CHK_RET(CreateGroup(groupName));
    groupMap_[groupName] = group;
}
```

审查检查方法:
- 持锁检查结果是否在解锁后被使用(TOCTOU)? 操作和检查是否在同一个临界区内?

关联commit: `7b12579d`

---

### 类别: 初始化/赋值/时序 (INIT)

通信框架对象生命周期早期阶段的脆弱性。核心模式是多分支初始化路径中的遗漏和成员变量初始化缺失。hcomm-dev 17条(10.5%)。

#### INIT-01: 多分支初始化路径不对称 [hcomm-dev]

严重等级: P1

缺陷描述: 同一函数存在V1/V2、图模式/单算子模式等多条初始化分支，其中某条遗漏了必要的初始化步骤。7条案例。

典型代码示例:

```cpp
// 缺陷 -- V2 path missing HcomSetGroupTopoInfo after CreateCommunicator
// V2 path
auto lambda = [&]() {
    CHK_RET(CreateCommunicator(...));
    // missing: CHK_RET(HcomSetGroupTopoInfo(...));
    return HCCL_SUCCESS;
};
HCCLV2_FUNC_RUN(lambda);

// Non-V2 path (correct)
CHK_RET(CreateCommunicator(...));
CHK_RET(HcomSetGroupTopoInfo(...));

// 修复 -- V2 path matches non-V2
auto lambda = [&]() {
    CHK_RET(CreateCommunicator(...));
    CHK_RET(HcomSetGroupTopoInfo(...));  // added
    return HCCL_SUCCESS;
};
```

审查检查方法:
- 同一函数的多条初始化分支是否执行了相同的必要初始化步骤(对称性检查)?
- V2适配路径是否完整覆盖了V1路径的所有初始化步骤?
- 对比两个分支的函数调用序列，寻找不对称项

关联commit: `65f0c2ba`, `b54e3852`

---

#### INIT-02: 成员变量未初始化/默认值错误 [hcomm-dev]

严重等级: P0

缺陷描述: C++内置类型成员未显式初始化，运行时包含垃圾值导致非确定性行为。5条案例。

典型代码示例:

```cpp
// 缺陷 -- multiple built-in type members lack in-class initializers
class HcclCommunicator {
    bool isGroupMode_;       // garbage -> group mode check anomaly
    u32 iSend;               // garbage
    u32 iRecv;
    u32 nSend;
    u32 nRecv;
    u32 bufferSliceNum;
};

// 修复
class HcclCommunicator {
    bool isGroupMode_ = false;
    u32 iSend = 0;
    u32 iRecv = 0;
    u32 nSend = 0;
    u32 nRecv = 0;
    u32 bufferSliceNum = 0;
};
```

审查检查方法:
- C++类的内置类型成员(bool/int/u32/指针)是否有显式初始化(in-class initializer或构造函数初始化列表)?
- Init函数是否具备幂等性? 重复调用是否会覆盖外部配置?

关联commit: `2bad014b`, `103755111cff`, `6596fc1f`

---

#### INIT-03: 执行顺序/依赖错误 [hcomm-dev]

严重等级: P0

缺陷描述: 操作执行顺序错误，或状态变量清除在依赖该状态的调用之前。5条案例。

典型代码示例:

```cpp
// 缺陷 -- start thread before setting flag, thread reads false
AicpuHcclProcess::CallMC2MaintenanceThread(ctx);  // thread checks flag
indOpCommInitialized_ = true;                       // flag not yet set

// 修复 -- set flag before starting thread
indOpCommInitialized_ = true;
AicpuHcclProcess::CallMC2MaintenanceThread(ctx);
```

```cpp
// 缺陷 -- executor_ set to nullptr before CalBlockDim which needs it
executor_ = nullptr;                  // premature nullification
HcclResult ret = CalBlockDim(...);    // CalBlockDim needs executor_

// 修复 -- move nullification after CalBlockDim
HcclResult ret = CalBlockDim(...);
if (ret != HCCL_SUCCESS) {
    executor_ = nullptr;              // move to error handler
    return ret;
}
```

审查检查方法:
- 状态变量清除操作(置nullptr/置0)是否在所有依赖该状态的调用之后?
- 线程启动前，线程内读取的共享状态是否已完成初始化?

关联commit: `4d1a3a60`, `08b2658a`, `87da187b`

---

#### INIT-04: offload/lambda变量作用域泄漏 [hcomm-dev]

严重等级: P1

缺陷描述: HCCLV2_FUNC_RUN宏的lambda外部提前做了cast和使用，offload模式下控制流未正确进入V2分支，变量在错误作用域被访问。

典型代码示例:

```cpp
// 缺陷 -- cast and use outside lambda
auto *comm = static_cast<hcclComm*>(handle);  // outside lambda
comm->DoSomething();                            // should not execute in non-V2 mode
auto lambda = [&]() {
    HCCLV2_FUNC_RUN(comm->V2Method());
    return HCCL_SUCCESS;
};

// 修复 -- move cast and use inside lambda
auto lambda = [&]() {
    auto *comm = static_cast<hcclComm*>(handle);
    CHK_RET(comm->V2Method());
    return HCCL_SUCCESS;
};
HCCLV2_FUNC_RUN(lambda);
```

审查检查方法:
- HCCLV2_FUNC_RUN宏前后的变量使用是否明确在正确的作用域内?
- lambda捕获列表中的变量是否可能在外部被提前消费?

关联commit: `b54e3852`

---

### 类别: 条件判断/分支覆盖 (BRANCH)

通信框架中算法选择(SelectAlg)、设备类型分支、V1/V2路径等分支逻辑极为复杂。遗漏特定拓扑或运行模式的分支是高频缺陷来源。

#### BRANCH-01: 枚举/类型分支遗漏 [hcomm-dev][hccl-dev]

严重等级: P1

缺陷描述: 新增枚举值后未同步更新所有switch/if分支。hcomm-dev 7条，hccl-dev中也有类似模式(散布算子的通信引擎枚举全部用错)。

典型代码示例:

```cpp
// 缺陷(hcomm-dev) -- engine type check only allows AICPU, missing AICPU_TS
if (engineType != AICPU) {
    HCCL_ERROR("engine type[%u] is not support", engineType);
    return HCCL_E_NOT_SUPPORT;
}

// 修复
if (engineType != AICPU && engineType != AICPU_TS) {
    HCCL_ERROR("engine type[%u] is not support", engineType);
    return HCCL_E_NOT_SUPPORT;
}
```

```cpp
// 缺陷(hcomm-dev) -- AIV selection without asymmetric topo guard
if (aivCoreNum > 0) {
    return SelectAIVAlg(algName);  // asymmetric topo not supported -> hang
}

// 修复
if (aivCoreNum > 0 && !multiModuleDiffDeviceNumMode_) {
    return SelectAIVAlg(algName);
}
```

审查检查方法:
- 新增枚举值时，是否全局搜索了所有switch/if判断并同步更新?
- AIV算法选择函数的拓扑约束checklist是否完整覆盖?

关联commit: `802c0411`, `c6f8cc36`(hcomm-dev), `7347fee3`(hccl-dev)

---

#### BRANCH-02: V2/offload适配路径缺失 [hcomm-dev]

严重等级: P1

缺陷描述: 新增API函数后未在V2头文件中声明weak symbol和添加HCCLV2_FUNC_RUN分发。5条案例。

典型代码示例:

```cpp
// 缺陷 -- HcclGetHcclBuffer missing V2 dispatch
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
    // V1 logic only, V2 environment behavior incorrect
    ...
}

// 修复
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
#if defined(OPEN_BUILD_PROJECT) && defined(ORION_MODE)
    HCCLV2_FUNC_RUN(HcclGetHcclBufferV2(comm, buffer));
#endif
    // V1 fallback
    ...
}
```

审查检查方法:
- 新增Hccl*/Hcom*公共API时，是否在V2头文件中声明了weak symbol并在函数体内添加HCCLV2_FUNC_RUN?
- 搜索仓库中所有没有HCCLV2_FUNC_RUN的公共API入口

关联commit: `e25c6484`

---

#### BRANCH-03: 条件方向错误/条件编译覆盖运行时分支 [hcomm-dev]

严重等级: P0

缺陷描述: 条件判断写反，或#ifdef块无条件覆写了上方的运行时分支判断。4条案例。

典型代码示例:

```c
// 缺陷 -- #ifdef CONFIG_CONTEXT overwrites runtime branch unconditionally
// Above: correct runtime branch
phyId = (attr->protocol == PROTOCOL_RDMA) ?
        attr->dev.rdma.phyId : attr->ub.phy_id;

#ifdef CONFIG_CONTEXT
    phyId = attr->ub.phy_id;  // unconditional overwrite, wrong for RDMA
#endif

// 修复 -- preserve branch semantics inside #ifdef
#ifdef CONFIG_CONTEXT
    phyId = (attr->protocol == PROTOCOL_RDMA) ?
            attr->dev.rdma.phyId : attr->ub.phy_id;
#endif
```

审查检查方法:
- 条件表达式的方向(>= vs <=, && vs ||)是否与注释/上下文语义一致?
- 基类和子类存在相同语义条件判断时，是否通过virtual方法统一?
- #ifdef块是否无条件覆写了上方的运行时判断?

关联commit: `54e5e41b`, `32ff3df8`, `66f33b83`

---

#### BRANCH-04: 设备类型判断遗漏组合条件 [hccl-dev]

严重等级: P1

缺陷描述: 设备类型分支仅检查硬件型号，遗漏了运行模式/环境变量等组合约束，导致在非预期配置下走入错误路径。

典型代码示例:

```cpp
// 缺陷代码(commit 19d20206, src/ops/scatter/scatter_op.cc)
// Only checks device type, ignores runtime mode env var
if (deviceType != DevType::DEV_TYPE_910_93 && deviceType != DevType::DEV_TYPE_910B) {
    return HcclScatterInner(...);
}

// 修复代码
// Check both device type and HCCL_OP_EXPANSION_MODE env var
if (!RunIndependentOpExpansion(deviceType)) {
    return HcclScatterInner(...);
}
// RunIndependentOpExpansion checks device + mode combo:
// 910_93: supports HOST_TS and AI_CPU
// 910B:   supports HOST_TS only
// others: not supported
```

审查检查方法:
- 设备类型分支逻辑是否考虑了环境变量、运行模式等配置维度
- 新增设备类型时是否更新了所有设备类型判断点
- 提取设备能力判断为独立函数，避免多处硬编码同一逻辑

关联commit: `19d20206`

---

### 类别: 数据类型/参数/枚举 (TYPE)

通信框架中类型安全问题突出，包括枚举值误用为算术因子、类型隐式截断、map::at()异常、参数对象混淆、通信引擎枚举变体混淆等。

#### TYPE-01: 枚举值语义误用 [hcomm-dev]

严重等级: P0

缺陷描述: 将枚举值当作字节大小、数组索引等用于算术运算。5条案例。

典型代码示例:

```cpp
// 缺陷 -- dataType enum used as divisor
// reduceInfo.dataType is enum (e.g. HCCL_DATA_TYPE_FP32=3), not byte size (4)
u64 count = src->len / reduceInfo.dataType;  // wrong result

// 修复 -- lookup SIZE_TABLE
u64 count = src->len / SIZE_TABLE[reduceInfo.dataType];

// 缺陷 -- wrong enum constant in comparison
bool IsOverFlowInfNanMode() {
    aclrtFloatOverflowMode mode = GetOverflowMode();
    return (mode == ACL_RT_OVERFLOW_MODE_UNDEF);  // should be INFNAN, not UNDEF
}

// 修复
bool IsOverFlowInfNanMode() {
    aclrtFloatOverflowMode mode = GetOverflowMode();
    return (mode == ACL_RT_OVERFLOW_MODE_INFNAN);
}
```

审查检查方法:
- dataType枚举值是否被直接用作算术运算的操作数? 应通过SIZE_TABLE查表获取字节数
- 枚举常量比较时，常量名是否与期望语义一致?

关联commit: `5422c95d`, `20824546`

---

#### TYPE-02: 类型隐式截断/溢出 [hcomm-dev][hccl]

严重等级: P1

缺陷描述: 函数返回类型或变量类型比内部计算类型窄。hcomm-dev 4条，hccl 2条(INT-01/INT-02)。

典型代码示例:

```cpp
// 缺陷(hcomm-dev) -- u32 timeout truncated to u8, max 255 seconds
u8 GetKernelExecTimeoutFromEnvConfig() {
    u32 timeout = GetEnvConfig("KERNEL_EXEC_TIMEOUT");
    return timeout;  // u32 -> u8 truncation
}

// 修复
u32 GetKernelExecTimeoutFromEnvConfig() {
    u32 timeout = GetEnvConfig("KERNEL_EXEC_TIMEOUT");
    return timeout;
}
```

```cpp
// 缺陷(hccl) -- u64 count truncated to u32, buffer overflow when >4GB elements
u32 count = hcomOpParam->count;  // u64 -> u32

// 修复
u64 count = hcomOpParam->count;
```

```cpp
// 缺陷(hccl) -- addition then narrow cast, wrap-around to near-zero timeout
u16 timeOut = static_cast<u16>(notifyWaitTime + AICPU_KERNEL_TIMEOUT_INC);
// When sum > 65535, timeout wraps to tiny value

// 修复 -- saturating arithmetic
if (notifyWaitTime + AICPU_KERNEL_TIMEOUT_INC >= MAX_VALUE_U16) {
    timeOut = MAX_VALUE_U16;
} else {
    timeOut = notifyWaitTime + AICPU_KERNEL_TIMEOUT_INC;
}
```

审查检查方法:
- 函数返回类型是否与内部实际计算类型一致? -Wconversion可检出
- 赋值时窄类型接收宽类型值是否有显式范围检查?
- 加法/乘法后窄化的表达式须有饱和逻辑或溢出检查
- 涉及内存大小计算的表达式必须使用u64
- 编译选项开启 `-Wconversion -Werror=conversion`

关联commit: `4ebb11d1`, `7a3d6fa6`(hcomm-dev), `9c1f957b`, `6baf33c4`(hccl)

---

#### TYPE-03: 参数对象混淆/引用错误 [hcomm-dev]

严重等级: P0

缺陷描述: 传入的参数使用了错误的对象，或参数被声明但未使用。4条案例。

典型代码示例:

```cpp
// 缺陷 -- should get notify from dstThreadPtr, actually gets from threadPtr(source)
void HcommThreadNotifyRecordOnThread(Thread *threadPtr, Thread *dstThreadPtr) {
    auto notify = threadPtr->GetNotify();    // wrong: from source thread
    // dstThreadPtr passed but unused
    ...
}

// 修复
void HcommThreadNotifyRecordOnThread(Thread *threadPtr, Thread *dstThreadPtr) {
    auto notify = dstThreadPtr->GetNotify();  // correct: from destination thread
    ...
}
```

审查检查方法:
- 函数参数被声明但未使用时是否告警(unused parameter)?
- 多个同类型参数的函数，调用处传参顺序是否正确?

关联commit: `d89396c9`

---

#### TYPE-04: map::at()对未收录key抛异常 [hcomm-dev]

严重等级: P0

缺陷描述: std::map::at()在key不存在时抛std::out_of_range异常，C++异常未被捕获导致coredump。新增枚举值后未更新对应map。3条案例。

典型代码示例:

```cpp
// 缺陷 -- errorMap missing new error codes, at() throws -> coredump
std::string msg = errorMap.at(code);  // HCCL_E_SUSPENDING not in map -> exception

// 修复 -- use find()
auto it = errorMap.find(code);
if (it != errorMap.end()) {
    std::string msg = it->second;
} else {
    std::string msg = "unknown error code: " + std::to_string(code);
}
```

审查检查方法:
- 禁止在可能接收不可控输入的场景使用std::map::at()，应使用find()+迭代器检查
- 新增枚举值时，是否全量搜索所有使用该枚举的map/switch/数组并同步更新?

关联commit: `a84b98a0`

---

#### TYPE-05: 通信引擎枚举变体混淆 [hccl-dev]

严重等级: P1

缺陷描述: CANN平台存在多个语义相近的通信引擎枚举值(COMM_ENGINE_AICPU / COMM_ENGINE_AICPU_TS / COMM_ENGINE_CPU / COMM_ENGINE_CPU_TS / COMM_ENGINE_AIV等)，使用了错误的变体会导致执行路径、资源分配和launch模式全部错误。

典型代码示例:

```cpp
// 缺陷代码(commit 7347fee3, src/ops/scatter/scatter_op.cc 6处全部用错)
param.engine = CommEngine::COMM_ENGINE_AICPU;         // wrong variant
if (param.engine == COMM_ENGINE_AICPU) { ... }

// 修复
param.engine = CommEngine::COMM_ENGINE_AICPU_TS;      // correct variant
if (param.engine == COMM_ENGINE_AICPU_TS) { ... }
```

审查检查方法:
- 搜索所有CommEngine枚举使用点，确认变体选择与设计文档一致
- 同一文件/函数内的引擎类型应前后一致
- 新增引擎类型使用时，要求开发者说明选择理由

关联commit: `7347fee3`

---

#### TYPE-06: 平台字符串字面量拼写错误 [hccl-dev]

严重等级: P1

缺陷描述: 平台对launch mode等字符串做精确匹配，拼写不一致(如"AICPU" vs "AI_CPU")会导致功能静默失效。

典型代码示例:

```cpp
// 缺陷代码(commit 13b6032d, src/ops/scatter/scatter_op.cc)
launchMode = "AICPU";   // platform expects "AI_CPU" (with underscore)

// 修复
launchMode = "AI_CPU";
```

审查检查方法:
- 所有平台相关字符串字面量应有对应常量定义，禁止裸字符串
- 审查时将字符串字面量与平台文档/头文件中的定义逐字符比对
- 搜索相似但不同的字符串(如"AICPU"/"AI_CPU"/"AiCpu")是否混用

关联commit: `13b6032d`

---

#### TYPE-07: API参数名不同步 [hccl-dev]

严重等级: P2

缺陷描述: 上游API参数名变更后(如blockDim->numBlocks)，调用侧变量名未同步。虽不影响编译和运行，但导致代码与API定义不一致，增加维护成本。

典型代码示例:

```cpp
// 缺陷(commit 2d8d548a)
constexpr uint32_t blockDim = 1;   // API renamed to numBlocks
aclrtLaunchKernelWithConfig(funcHandle, blockDim, stream, &cfg, argsHandle, nullptr);

// 修复
constexpr uint32_t numBlocks = 1;
aclrtLaunchKernelWithConfig(funcHandle, numBlocks, stream, &cfg, argsHandle, nullptr);
```

审查检查方法:
- API参数名变更时，全局搜索旧名确保所有调用点、stub、示例代码同步
- 代码中的变量命名应与被调API的参数名保持一致

关联commit: `2d8d548a`

---

### 类别: 配置与兼容性 (CFG)

多设备类型、多编译环境、多版本协议的适配问题。hccl仓库13次(15.5%)。

#### CFG-01: 跨版本协议OpCode修改须保持向后兼容 [hccl]

严重等级: P0

缺陷描述: 修改已发布的OpCode对应的数据结构时，直接复用原OpCode编号。新版本发送新结构体给旧版本，旧版本按老结构体解析导致协议不兼容。

典型代码示例:

```cpp
// 缺陷代码 -- src/platform/hccp/inc/private/network/ra_rs_comm.h
// RA_RS_TLV_INIT = 87's data struct gained moduleType field, but OpCode unchanged
// Old version parses new data with old struct layout -> field misalignment

// 修复代码
RA_RS_TLV_INIT_V1 = 87,     // old version keeps original number
RA_RS_TLV_INIT = 110,       // new version gets new number

// Query peer version before sending
ret = RaHdcGetInterfaceVersion(phyId, RA_RS_TLV_INIT, &interfaceVersion);
if (ret == 0 && interfaceVersion >= RA_RS_OPCODE_BASE_VERSION) {
    opCode = RA_RS_TLV_INIT;        // peer supports new version
} else {
    opCode = RA_RS_TLV_INIT_V1;     // peer only supports old version
}
```

审查检查方法:
- 修改已发布OpCode的数据结构时，必须分配新OpCode编号
- 保留旧OpCode的处理函数(至少一个版本周期)
- 发送前须查询对端版本能力，选择兼容的OpCode

关联commit: `5ad2ced6`

---

#### CFG-02: 设备类型分支须全面覆盖 [hccl]

严重等级: P1

缺陷描述: 新增硬件资源初始化或引擎类型时，仅在部分设备类型分支中处理，其他设备类型分支遗漏。或者新增引擎类型后未全局检查所有引擎分支代码。

审查检查方法:
- 新增硬件资源初始化须审查是否适用所有设备类型(910_93/910_95/A3等)
- 新增引擎类型后须全局grep所有按引擎分支的代码确认归属
- 按设备类型分支时审查各分支功能等价性

关联commit: `099fe2a8`(心跳socket类型判断不一致), `07c52708`(910_95 profiling不兼容), `91fbd1d6`(A3 AIV跨节点回退1700行), `9d75f557`(AIV引擎归入CPU分支)

---

#### CFG-03: 常量不应承担多重语义 [hccl]

严重等级: P2

缺陷描述: 同一个常量既作为默认值又作为哨兵值(标识"未配置")，甚至其数值本身超出合法范围。

典型代码示例:

```cpp
// 缺陷代码 -- src/legacy/framework/topo/new_topo_builder/rank_table_info/new_rank_info.h
constexpr unsigned int MAX_VALUE_DEVICEPORT = 65536;  // default + sentinel + exceeds port max
u32 devicePort{MAX_VALUE_DEVICEPORT};

// 修复代码
constexpr unsigned int DEFAULT_VALUE_DEVICEPORT = 60001;  // explicit default
constexpr unsigned int MAX_VALUE_DEVICEPORT = 65535;       // valid port max
u32 devicePort{DEFAULT_VALUE_DEVICEPORT};
```

审查检查方法:
- 一个常量只承担一个语义
- 端口号上限为65535而非65536
- 硬件常量修改须在注释中标注取值依据(spec文档引用)

关联commit: `2bfa02d1`(端口默认值/哨兵值混淆), `fb56d64b`(CCU loop count常量值错误64->128)

---

#### CFG-04: v1/v2接口分发完整性 [hccl]

严重等级: P1

缺陷描述: op_base.cc中的公开API须同时有v1和v2版本的分发入口。新增或修改API时遗漏HCCLV2_FUNC_RUN分发，导致v2调用路径无法到达。

审查检查方法:
- op_base.cc中所有公开API须有HCCLV2_FUNC_RUN分发(可自动化扫描)
- 字符串到枚举映射表中key须与枚举值名称一致

关联commit: `6383b2bc`(HcclGetCommConfigCapability缺分发), `edf73e80`(协议名字符串映射不匹配)

---

#### CFG-05: 结构体/类字段变更未同步到所有使用方 [hccl-dev]

严重等级: P1

缺陷描述: 修改结构体定义(增删字段)后，部分使用方(特别是测试代码、mock、stub)未同步更新，导致编译失败。

典型代码示例:

```cpp
// 缺陷(commit c11a5289, test/.../sim_communicator.cc)
// HcclCommConfig removed commEngine/threadNum/notifyNumPerThread
// but test code still sets these fields
commConfig.commEngine = HCCL_COMM_ENGINE_CONFIG_NOT_SET;
commConfig.threadNum  = HCCL_COMM_THREADNUM_CONFIG_NOT_SET;
commConfig.notifyNumPerThread = HCCL_COMM_NOTIFY_NUM_PER_THREAD_CONFIG_NOT_SET;

// 修复: delete assignments to removed fields
```

审查检查方法:
- 修改struct/class定义时，全局grep所有使用点，范围必须包含test/目录
- 提交变更前在本地完成全量编译
- 注意同一struct可能在.cc和.h中都有使用

关联commit: `c11a5289`, `8222fcf8`

---

#### CFG-06: 公共API函数名与头文件声明不匹配 [hccl-dev]

严重等级: P2

缺陷描述: 公共API的.cc实现使用了错误的函数名(与内部函数同名)，与.h声明不一致，导致链接失败。

典型代码示例:

```cpp
// 缺陷(commit cae52923, src/ops/interface/hccl_collective_op.cc)
// Header declares HcclBatchSendRecv, but .cc implements as HcclBatchSendRecvInner
HcclResult HcclBatchSendRecvInner(HcclSendRecvItem* sendRecvInfo, ...) {
    return HcclBatchSendRecvInner(sendRecvInfo, itemNum, comm, stream);
    // also infinite recursion (calls itself)
}

// 修复
HcclResult HcclBatchSendRecv(HcclSendRecvItem* sendRecvInfo, ...) {
    return HcclBatchSendRecvInner(sendRecvInfo, itemNum, comm, stream);
}
```

审查检查方法:
- 公共API函数的.cc定义必须与.h声明的函数签名精确匹配
- 包装函数的名字不应与被包装函数相同(否则变成递归调用)

关联commit: `cae52923`

---

### 类别: 资源管理 (RES)

资源的创建、使用、销毁三阶段管理不当。hccl仓库8次(资源生命周期) + 4次(内存管理)，hcomm-dev仓库12条(7.4%)。

#### RES-01: early return跳过必要初始化/清理 [hccl]

严重等级: P1

缺陷描述: 函数包含多个early return路径，某个return跳过了后续必要的初始化或清理逻辑。

审查检查方法:
- 函数包含多个early return时，逐一审查每个return路径是否遗漏后续必要初始化/清理
- 调试用的`return HCCL_SUCCESS`须在提交前移除(lint可检测unreachable code)

关联commit: `82989927`(early return跳过TpManager::Init), `c5443da0`(调试遗留return跳过整个逻辑)

---

#### RES-02: map/cache的key必须能唯一标识资源 [hccl]

严重等级: P1

缺陷描述: map/cache的key设计不充分，未包含所有区分资源的关键属性。不同语义的资源请求命中同一key，复用了错误的资源。

审查检查方法:
- 设计map/cache的key时，问"同一key下是否可能出现不同语义的资源请求?"
- key须覆盖所有区分资源的关键属性(IP+端口、algName+remoteRank等)
- 资源销毁顺序须按依赖关系图反序

关联commit: `43069da0`(IP不含端口), `18752141`(key不含remote rank)

---

#### RES-03: context切换附近的内存操作须审查执行context [hccl]

严重等级: P1

缺陷描述: 代码中切换到不同的设备context后分配/释放内存，但操作应在原context下执行。

审查检查方法:
- context切换(SetDevice/SwitchContext)附近的内存分配须审查执行context是否正确
- 枚举值switch/if匹配需覆盖所有有效值

关联commit: `3b8ac218`(DPU context下错误分配NPU内存), `6b9278e4`(图模式remote memory更新遗漏)

---

#### RES-04: 析构路径须完整 [hccl]

严重等级: P1

缺陷描述: 引用计数递减到非零值时直接返回，但该路径上可能有局部获取的资源(如映射handle)需要释放。

审查检查方法:
- 分配/映射的资源在释放函数中须有明确释放路径
- 引用计数递减的每个分支都须检查是否有局部资源需释放
- 析构函数中的日志/DFX调用应在所有资源释放操作之前(避免日志依赖先被销毁)

关联commit: `a9fea200`(refCount>1分支遗漏释放handle), `7d914168`(析构顺序错误导致日志依赖先销毁)

---

#### RES-05: 资源释放顺序错误 [hcomm-dev]

严重等级: P0

缺陷描述: 组件间存在资源依赖，析构顺序不正确导致UAF或报错。4条案例。

典型代码示例:

```cpp
// 缺陷 -- communicator destroys AlgResource first,
// but ChannelManager's transport links depend on it
HcclCommunicator::~HcclCommunicator() {
    DestroyAlgResource();       // release alg resources
    // channelManager_ destructor accesses freed resources -> error
}

// 修复 -- inject releaseChannel_() callback before resource release
HcclCommunicator::~HcclCommunicator() {
    DestroyAlgResource();
    releaseChannel_();  // release channel transport first, avoid UAF
}
```

```cpp
// 缺陷 -- UnimportAllJetty frees handle but doesn't invalidate
void CcuComponent::UnimportAllJetty() {
    for (auto &jetty : jettyList_) {
        UrmaUnimportJetty(jetty.handle);
        // missing: jetty.handle = INVALID_HANDLE;
    }
}

// 修复 -- invalidate immediately after free
void CcuComponent::UnimportAllJetty() {
    for (auto &jetty : jettyList_) {
        UrmaUnimportJetty(jetty.handle);
        jetty.handle = INVALID_HANDLE;  // prevent double free on reentry
    }
}
```

审查检查方法:
- 当组件A持有组件B创建的资源引用时，析构流程是否保证A的引用先释放?
- 资源释放后是否立即将handle/指针置空?

关联commit: `f45581ee`, `e17cfff4`

---

#### RES-06: 资源重复管理/泄漏 [hcomm-dev]

严重等级: P1

缺陷描述: 同一段内存/资源被多个路径独立管理，导致重复映射或分配过量。4条案例。

典型代码示例:

```cpp
// 缺陷 -- P2P input/output already within CCL buffer range
// but still independently IPC-registered -> IPC leak -> OOM
void TransportP2P::Init() {
    SetMemoryName(input_);    // input_ within machinePara_.mem[0]
    SetMemoryName(output_);   // redundant registration -> IPC leak
    ...
}

// 修复 -- check address range inclusion
void TransportP2P::Init() {
    isMemInclude_ = IsAddrInRange(input_, machinePara_.mem[0]);
    if (!isMemInclude_) {
        SetMemoryName(input_);  // only register independent ranges
    }
    ...
}
```

审查检查方法:
- 同一段内存/资源是否存在"子区域被父区域管理但仍被独立管理"的重复路径?
- 内存分配逻辑的每个分支是否有算法文档或注释支撑其buffer大小计算?

关联commit: `f64f6498`, `9277e023`

---

#### RES-07: 错误路径资源泄漏 [hcomm-dev]

严重等级: P1

缺陷描述: 正常路径正确释放但CHK_RET提前返回时遗漏释放。4条案例。

典型代码示例:

```cpp
// 缺陷 -- hrtMalloc memory leaked on CHK_RET failure path
void *devMem = nullptr;
CHK_RET(hrtMalloc(&devMem, size));
CHK_RET(SomeOperation(devMem));  // failure leaks devMem
CHK_RET(AnotherOp(devMem));
hrtFree(devMem);

// 修复 -- explicit cleanup or RAII
void *devMem = nullptr;
CHK_RET(hrtMalloc(&devMem, size));
HcclResult ret = SomeOperation(devMem);
if (ret != HCCL_SUCCESS) {
    hrtFree(devMem);
    return ret;
}
```

```cpp
// 缺陷 -- buffer added to manager, Init() fails, stale entry remains
localRmaBufferMgr->Add(tempKey, buffer);
CHK_RET(buffer->Init());  // failure -> uninitialized buffer in manager -> coredump

// 修复 -- rollback on failure
localRmaBufferMgr->Add(tempKey, buffer);
HcclResult ret = buffer->Init();
if (ret != HCCL_SUCCESS) {
    localRmaBufferMgr->Del(tempKey);  // rollback
    return ret;
}
```

审查检查方法:
- 每个CHK_RET/提前return路径上，之前分配的资源是否已释放? (RAII优于手动管理)
- "先插入容器再初始化"的模式是否有Init失败的回滚逻辑?

关联commit: `d8400fd2`, `bc8c5036`

---

#### RES-08: 禁止暴露局部容器内部指针 [hccl]

严重等级: P0

缺陷描述: 将栈上局部vector的`.data()`指针通过输出参数返回。函数返回后vector析构，调用方拿到悬垂指针。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/hcom/hcom.cc
HcclResult HcomGetandClearOverFlowTasks(const char *group,
    hccl::HcclDumpInfo **hcclDumpInfoPtr, u32 *len)
{
    std::vector<hccl::HcclDumpInfo> hcclDumpInfo;  // local vector
    CHK_RET(hcclComm->GetandClearOverFlowTasks(hcclDumpInfo));
    *hcclDumpInfoPtr = hcclDumpInfo.data();  // returns internal pointer
    *len = hcclDumpInfo.size();
    return HCCL_SUCCESS;
}  // hcclDumpInfo destroyed, *hcclDumpInfoPtr is dangling

// 修复代码 -- malloc + memcpy
if (hcclDumpInfo.size() > 0) {
    *hcclDumpInfoPtr = static_cast<hccl::HcclDumpInfo*>(
        malloc(hcclDumpInfo.size() * sizeof(hccl::HcclDumpInfo)));
    CHK_PTR_NULL(*hcclDumpInfoPtr);
    memcpy_s(*hcclDumpInfoPtr, ..., hcclDumpInfo.data(), ...);
}
```

审查检查方法:
- 禁止将局部容器(vector/string/array)的内部数据指针通过输出参数或返回值暴露给外部
- 需要返回数据所有权时，使用malloc+memcpy或std::unique_ptr

关联commit: `6784944a`(此缺陷由`4e82ec25`的修复引入)

---

#### RES-09: IPC共享内存注册须检查重复注册 [hccl]

严重等级: P1

缺陷描述: P2P传输中同一块物理内存被重复注册IPC映射，大规模通信下资源耗尽OOM。

审查检查方法:
- IPC共享内存注册前检查是否已注册同一块内存
- bool成员变量默认false且在多处以`!flag`守护资源分配时，检查所有初始化路径是否都有flag设置

关联commit: `3ec7410b`(IPC重复注册OOM), `b2e74aee`(SetMemIncludeFlag遗漏调用)

---

#### RES-10: 可选初始化路径的成员指针使用前必须判空 [hccl]

严重等级: P1

缺陷描述: 成员变量仅在特定配置条件下创建(如`symmetricMemory_`仅在跨超节点场景初始化)，但使用该成员的函数没有判空保护，其他配置下调用时空指针解引用crash。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/communicator/impl/hccl_communicator_host.cc
bool HcclCommunicator::IsSupportSymmetricMemory(HcclCMDType opType, OpParam &opParam)
{
    // symmetricMemory_ is nullptr in non-cross-supernode scenarios
    // no null check -> crash
    HCCL_INFO("[%s] aicpuUnfold[%d], workflowMode[%d], ...", ...);
}

// 修复代码
bool HcclCommunicator::IsSupportSymmetricMemory(HcclCMDType opType, OpParam &opParam)
{
    CHK_PRT_RET(symmetricMemory_ == nullptr,
        HCCL_DEBUG("symmetricMemory_ is a nullptr"), false);  // early return
    HCCL_INFO("[%s] aicpuUnfold[%d], workflowMode[%d], ...", ...);
}
```

审查检查方法:
- 成员变量仅在特定条件下初始化时，所有使用点须有判空保护
- 新增对可选成员的使用时，须确认所有初始化路径是否都覆盖了该成员的创建
- 构造函数/Init中有条件分支跳过某些成员初始化时，标记这些成员为"可选"并在使用处强制判空

关联commit: `4e0bd97b`

---

### 类别: 日志/可观测性 (LOG)

hccl仓库8次(9.5%)，hcomm-dev仓库11条(6.8%)。日志格式串参数错误(如%s传int)可导致coredump。

#### LOG-01: 格式化字符串占位符须与实参类型匹配 [hccl][hcomm-dev]

严重等级: P2

缺陷描述: printf-style格式化中，`%s`传入int、`%u`传入u64、参数与占位符错位。可能导致日志输出垃圾值或crash。hcomm-dev中%s传int32_t导致将整数当指针解引用(UB)是P0级。

典型代码示例:

```cpp
// 缺陷(hcomm-dev) -- %s expects string but gets int32_t
HCCL_ERROR("[%s] operation failed, ret[%s]", __func__, ret);
//                                     ^^ dereferences int as pointer -> UB/segfault

// 修复
HCCL_ERROR("[%s] operation failed, ret[%d]", __func__, ret);
```

审查检查方法:
- 编译选项开启 `-Wformat -Werror=format`
- `std::string`传入C格式化函数须用`.c_str()`
- 禁止字符串字面量中用 `\` 续行(会将下一行前导空格纳入字符串)
- 日志中方括号是否配对?

关联commit: `35e32c7c`, `6a6eac0f`(hccl), `5e446705`, `f715f167`(hcomm-dev)

---

#### LOG-02: 日志前缀/tag必须与所在类名/函数名匹配 [hccl]

严重等级: P3

缺陷描述: copy-paste代码后未更新日志中的模块名/类名标识，误导定位。

审查检查方法:
- 日志前缀/tag必须与所在函数名或类名匹配
- 日志模块ID必须在枚举中有明确定义
- 可用脚本自动检测日志tag与代码位置的一致性

关联commit: `7ff92abc`, `112766de`(缺少[HCCL_ENV]前缀), `12f3680c`(日志类名copy-paste错误), `ed50e7eb`(日志模块ID未在枚举中定义)

---

#### LOG-03: 快速路径必须与标准路径保持功能对等 [hccl]

严重等级: P2

缺陷描述: cache hit或fast path跳过了profiling/tracing/DFX信息采集，导致性能分析和问题定位时信息缺失。

审查检查方法:
- 所有快速路径(cache hit/fast path)必须与标准路径保持profiling/tracing/logging功能对等
- 新增fast path时逐项对比标准路径的DFX调用

关联commit: `891665ac`(SQE缓存命中缺profiling), `3a9b61f2`(CCU DFX结构体缺字段)

---

#### LOG-04: 日志级别不当 [hcomm-dev]

严重等级: P3

缺陷描述: 可恢复场景使用ERROR级别，误导运维排查。CHK_RET宏隐式打印ERROR。3条案例。

典型代码示例:

```cpp
// 缺陷 -- AIV core not enough is recoverable, CHK_RET prints ERROR implicitly
CHK_RET(CalcAivBlockDim(coreNum));  // failure prints ERROR

// 修复 -- explicit control of log level
HcclResult ret = CalcAivBlockDim(coreNum);
if (ret != HCCL_SUCCESS) {
    HCCL_WARNING("AIV core not enough, fallback to non-AIV path");
    return ret;
}
```

审查检查方法:
- 可恢复/预期场景(如AIV fallback)是否使用了ERROR级别? 应降为WARNING/INFO
- CHK_RET宏包裹的调用失败是否确实应打ERROR? 有fallback路径时建议直接return

关联commit: `6972024c`, `2adb85d7`

---

#### LOG-05: 日志洪泛/变量引用错误 [hcomm-dev]

严重等级: P3

缺陷描述: 高频路径无日志抑制导致刷屏; 日志中引用已被修改的累积值而非原始值。4条案例。

典型代码示例:

```cpp
// 缺陷 -- high-frequency opcode prints debug log every call
HcclResult RaGetOpRight(u32 opcode) {
    HCCL_DEBUG("get op right for opcode[%u]", opcode);  // log flood
    ...
}

// 修复 -- log suppression
HcclResult RaGetOpRight(u32 opcode) {
    if (!RaIsOpcodeLogSuppressed(opcode)) {
        HCCL_DEBUG("get op right for opcode[%u]", opcode);
    }
    ...
}
```

```cpp
// 缺陷 -- log prints accumulated value instead of original
scratchBufSize += extraReserve;
HCCL_INFO("cclBufferSize[%llu]", scratchBufSize);  // prints post-accumulation value

// 修复 -- use original variable
HCCL_INFO("cclBufferSize[%llu]", cclBufferSize);   // original variable
scratchBufSize += extraReserve;
```

审查检查方法:
- 日志中引用的变量是否是当前值而非已被修改的累积值?
- 高频调用路径上的日志是否有抑制/采样机制?

关联commit: `4425a342`, `802c0411`

---

### 类别: 错误处理 (ERR)

合并hccl的ERR系列(7次, 8.3%)和hcomm-dev的PTR系列(8条, 4.9%)。

#### ERR-01: 异常上报操作须受前置条件保护 [hccl]

严重等级: P2

缺陷描述: 异常上报/错误通知在正常路径上也无条件执行，导致正常操作触发虚假的异常打印。

审查检查方法:
- 异常上报操作(SendTaskExceptionByMBox等)必须受前置条件判断保护
- 重试成功路径不应触发异常上报

关联commit: `96087ffb`(重试成功也触发异常打印)

---

#### ERR-02: 项目有统一异常处理宏时不允许裸try-catch [hccl]

严重等级: P2

缺陷描述: 项目提供了统一异常处理宏(如TRY_CATCH_PROCESS_THROW)，但开发者手写try-catch吞掉异常只打日志return，导致异常类型信息丢失。

典型代码示例:

```cpp
// 缺陷代码 -- src/legacy/unified_platform/resource/ccu_transport/ccu_transport.cpp
try {
    std::unique_lock<std::shared_timed_mutex> lock(transMutex);
    status = StateMachine();
} catch (HcclException &e) {
    HCCL_ERROR(e.what());
    return CcuTransport::TransStatus::CONNECT_FAILED;  // swallows exception
} catch (...) {
    HCCL_ERROR("Unknown error");
    return CcuTransport::TransStatus::CONNECT_FAILED;
}

// 修复代码 -- use project macro
auto lockAndStatuMachine = [&]() {
    std::unique_lock<std::shared_timed_mutex> lock(transMutex);
    status = StateMachine();
};
TRY_CATCH_PROCESS_THROW(InternalException, lockAndStatuMachine(),
    "CcuTransport GetStatus() Error", { transStatus = CONNECT_FAILED; });
```

审查检查方法:
- 搜索裸try-catch块，确认是否应使用项目统一宏
- `catch(...)` 块不应默默return，至少需要向上传播异常类型信息

关联commit: `69c73166`

---

#### ERR-03: C接口函数不应抛C++异常 [hccl]

严重等级: P0

缺陷描述: 对外接口函数内部调用了含`THROW`的inline函数(如`MC2DataType()`、`MC2ReduceType()`)，当参数非法时抛出C++异常。若调用方无法处理异常(如通过dlsym加载或跨语言调用)，异常穿越接口边界导致`std::terminate()`直接abort进程。

典型代码示例:

```cpp
// 缺陷代码 -- src/legacy/framework/entrance/op_base/op_base_v2.cc
DataType mc2DataType = MC2DataType(static_cast<HcclDataType>(srcDataType));
// MC2DataType() contains THROW<Hccl::CcuApiException>

// 修复代码 -- bounds check + table lookup + return error code
if (srcDataType >= (sizeof(MC2_DATA_TYPE) / sizeof(MC2_DATA_TYPE[0]))) {
    HCCL_ERROR("srcDataType[%u] invalid.", srcDataType);
    return HCCL_E_PARA;
}
opArgsPtr->srcDataType = MC2_DATA_TYPE[srcDataType];
```

审查检查方法:
- 对外接口函数体内不应有未catch的throw路径(含间接调用的inline函数中的THROW)
- 接口边界处使用错误码返回而非异常

关联commit: `9939a862`

---

#### ERR-04: CHK_RET调用顺序须与业务逻辑匹配 [hccl]

严重等级: P2

缺陷描述: 带CHK_RET宏包裹的函数调用放在条件判断之前，非相关场景的函数失败会阻断整个函数。

审查检查方法:
- CHK_RET包裹的函数调用应延迟到确认需要其返回值处
- if-else分支设置错误状态后检查是否遗漏return

关联commit: `9476c6df`(CHK_RET调用顺序不当), `e025b6c5`(if-else缺return导致fallthrough)

---

#### ERR-05: 预期内返回值不应用error日志级别 [hccl]

严重等级: P2

缺陷描述: 调用底层接口时，某些返回值(如`-ENOTSUPP`)是预期内的"功能不支持"语义，但代码统一用error级别日志打印，产生大量误导性错误日志。

典型代码示例:

```c
// 缺陷代码 -- src/platform/hccp/rdma_agent/peer/ra_peer.c
ret = RsSetQpLbValue(qpHandle->phyId, qpHandle->rdevIndex, qpHandle->qpn, lbValue);
if (ret != 0) {
    hccp_err("[set][lbValue]RsSetQpLbValue failed ret:%d", ret);  // -ENOTSUPP also ERROR
}

// 修复代码 -- distinguish expected vs real error
if (ret != 0) {
    if (ret == -ENOTSUPP) {
        hccp_run_warn("[set][lbValue]RsSetQpLbValue unsuccessful ret:%d", ret);
    } else {
        hccp_err("[set][lbValue]RsSetQpLbValue failed ret:%d", ret);
    }
}
```

审查检查方法:
- 调用可能返回"不支持/不可用"语义值的底层接口时，须区分日志级别
- `-ENOTSUPP`、`-EOPNOTSUPP`、`HCCL_E_NOT_SUPPORT`等返回值应用warn而非error
- error级别日志应保留给真正需要人工介入的错误

关联commit: `2fcde546`

---

#### ERR-06: 空指针检查顺序错误 [hcomm-dev]

严重等级: P0

缺陷描述: 先解引用再检查null，或日志中%s传可能为nullptr的指针(UB)。3条案例。

典型代码示例:

```cpp
// 缺陷 -- dereference before null check
auto val = ptr->GetValue();
if (ptr == nullptr) {
    return HCCL_E_PTR;
}

// 修复
if (ptr == nullptr) {
    return HCCL_E_PTR;
}
auto val = ptr->GetValue();
```

```cpp
// 缺陷 -- nullptr passed to %s is UB
void DestroyDispatcherCtx(const char *commId) {
    HCCL_INFO("[DestroyCtx] commId[%s]", commId);  // commId may be nullptr
    if (commId == nullptr) { return; }

// 修复
void DestroyDispatcherCtx(const char *commId) {
    if (commId == nullptr) { return; }
    HCCL_INFO("[DestroyCtx] commId[%s]", commId);
```

审查检查方法:
- 空指针检查是否在解引用之前(不是之后)?
- %s格式化的参数是否保证非空?

关联commit: `dd744786`, `384d78c0`

---

#### ERR-07: API入口参数校验缺失 [hcomm-dev]

严重等级: P1

缺陷描述: 公共API中reinterpret_cast后未检查null，用户传入无效句柄则crash。3条案例。

典型代码示例:

```cpp
// 缺陷 -- HcclGetHcclBuffer checks buffer but misses comm
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
    CHK_PRT_RET(buffer == nullptr,
        HCCL_ERROR("buffer is nullptr"), HCCL_E_PTR);
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
    // comm is nullptr -> hcclComm is nullptr -> crash on next line
    std::string identifier = hcclComm->GetIdentifier();

// 修复 -- add comm pointer check
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
    CHK_PRT_RET(buffer == nullptr,
        HCCL_ERROR("buffer is nullptr"), HCCL_E_PTR);
    CHK_PRT_RET(comm == nullptr,
        HCCL_ERROR("comm is nullptr"), HCCL_E_PTR);
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
```

审查检查方法:
- reinterpret_cast/static_cast后是否检查null? 特别是来自用户输入的句柄
- 公共API入口是否对所有指针参数做了null/有效性校验?

关联commit: `58d588bf`

---

#### ERR-08: 可选指针/新增参数校验缺失 [hcomm-dev]

严重等级: P1

缺陷描述: 可能为空的可选指针未做defensive check; 新增参数的操作缺少不支持的组合校验。2条案例。

典型代码示例:

```cpp
// 缺陷 -- commMem pointer may be null but used directly
void *buffer = commMem->GetBuffer();  // commMem may be null

// 修复
if (commMem == nullptr) {
    HCCL_ERROR("commMem is nullptr");
    return HCCL_E_PTR;
}
void *buffer = commMem->GetBuffer();
```

审查检查方法:
- 新增数据类型/操作类型组合时，是否有IsSupportXxx前置校验?

关联commit: `a2b86b3d`, `384d78c0`

---

### 类别: 状态/缓存/超时 (STATE)

缓存/复用机制中key生成不完整或缓存数据过期的问题。HCCL项目特有的高频模式(hccl 6次 + hcomm-dev 10条)。

#### STATE-01: cache key须覆盖所有影响执行路径的状态维度 [hccl]

严重等级: P1

缺陷描述: 缓存key的维度不够，新增的分支条件未反映在key中，导致不同执行路径命中同一缓存。

典型代码示例:

```cpp
// 缺陷代码 -- src/framework/device/framework/aicpu_cache_manager.cc
// AlltoAllV cache key missing isBigCount dimension
// big/small count SQE layout schemes share same cache

// 修复代码 -- encode isBigCount into cache key
bool isBigCountForAlltoallv = false;
CHK_RET(IsBigCountForAlltoallv(param, topoinfo, isBigCountForAlltoallv));
inputSize = isBigCountForAlltoallv ? 1 : 0;  // encode into cache key
```

审查检查方法:
- 新增if/switch分支逻辑如果影响执行路径，须检查是否反映在cache key中
- cache key维度checklist: opType、algName、dataType、count/size、拓扑信息、特殊标志位
- 分支条件与缓存key必须使用相同数据源

关联commit: `e0744b7b`(缺isBigCount维度), `18752141`(缺remote rank信息)

---

#### STATE-02: 缓存复用前须逐字段确认是否需要刷新 [hccl][hcomm-dev]

严重等级: P1

缺陷描述: 缓存命中后直接复用所有缓存字段，但某些字段(如stream handle)在每次调用时可能变化，复用过期值导致错误。两个仓库均出现此缺陷。

典型代码示例:

```cpp
// 缺陷代码(hccl)
// ExecOpCache reuses old stream instead of current OpParam stream

// 修复代码
cacheInfo.resourceArgs.stream = opParam.stream.ptr();  // refresh stream
```

```cpp
// 缺陷代码(hcomm-dev)
// ExecOpCache stream field not updated to current operation's stream
if (cacheHit) {
    LaunchKernel(cacheInfo.resourceArgs);  // stale stream -> execution order error
}

// 修复
if (cacheHit) {
    cacheInfo.resourceArgs.stream = currentStream;  // refresh
    LaunchKernel(cacheInfo.resourceArgs);
}
```

审查检查方法:
- 缓存复用前逐字段确认哪些可能变化，对可变字段加刷新逻辑(staleness checklist)
- 句柄类字段(stream、context、device handle)几乎总需要刷新
- 缓存用于恢复逻辑时检查是否保存了所有依赖的输入状态

关联commit: `43dab3e2`, `dd053d5b`(hccl), `3c24c0fe`, `b3975b85`(hcomm-dev)

---

#### STATE-03: 函数副作用污染成员变量 [hccl]

严重等级: P1

缺陷描述: 在调用某函数前后都使用了同一成员变量，但该函数有副作用会修改这个成员变量。后续代码使用的是被修改后的值而非预期的原始值。

典型代码示例:

```cpp
// 缺陷代码 -- src/legacy/framework/communicator/communicator_impl.cc
OpAcceleratorStateFallback();  // side effect: modifies curAlgName
// ...
opAcceStateCache.insert({{curOpParams.opType, curAlgName}, ...});  // uses modified value

// 修复代码
string needFallBackAlgName = curAlgName;   // save original
OpAcceleratorStateFallback();              // may modify curAlgName
// ...
opAcceStateCache.insert({{curOpParams.opType, needFallBackAlgName}, ...});  // use saved value
```

审查检查方法:
- 函数调用前后使用同一成员变量时，审查被调用函数是否修改该变量
- 有副作用的函数须在注释或命名中体现

关联commit: `e4f59213`

---

#### STATE-04: 运行时分支条件与缓存key必须使用相同数据源 [hccl]

严重等级: P1

缺陷描述: 运行时判断走哪条执行路径用了变量A，但生成缓存key时用了变量B。A和B在大多数情况下一致，但在边界情况(如最后一轮最后一张卡)下不一致，导致缓存复用时走错路径。

审查检查方法:
- 运行时分支条件和缓存key生成必须使用完全相同的数据源
- 用虚函数或统一getter抽象数据源，避免两处分别计算

关联commit: `b0e6a8b7`(运行时用curSize_判断，key用perRankAvgDataSize_)

---

#### STATE-05: 超时值硬编码/配置不一致 [hcomm-dev]

严重等级: P1

缺陷描述: 多层超时值未统一联动，内外层超时语义不一致。4条案例。

典型代码示例:

```cpp
// 缺陷 -- OpRetry timeout hardcoded 205s, user-configured larger value times out first
constexpr u32 OP_RETRY_SEND_RECV_TIMEOUT = 205;
future.wait_for(std::chrono::seconds(OP_RETRY_SEND_RECV_TIMEOUT));
// User HCCL_LINK_TIMEOUT=300, but OpRetry times out at 205s first

// 修复 -- take max(user_config, default) + margin
u32 timeout = std::max(GetExternalInputHcclLinkTimeOut(),
                       OP_RETRY_SEND_RECV_TIMEOUT) + OP_RETRY_WAIT_AICPU_TIMEOUT;
```

```cpp
// 缺陷 -- execTimeOut=0 means "no timeout", but code treats literally
u64 timeoutUs = execTimeOut * 1000000;  // 0 * 1000000 = 0us -> immediate timeout

// 修复 -- handle zero as special semantics
u64 timeoutUs = (execTimeOut == 0) ? UINT64_MAX : execTimeOut * 1000000;
```

审查检查方法:
- 超时参数是否从外层逐层传递到底层阻塞调用? std::async+future::wait_for是否有内层取消机制?
- 超时值0是否有"不限制"的特殊语义? 转换函数中是否显式处理了零值?
- 不同组件的超时值是否与用户可配置值联动(取max而非硬编码)?

关联commit: `b8de68b9`, `1479e7ba`, `f058f2d9`

---

#### STATE-06: 环境变量/运行时状态一致性 [hcomm-dev]

严重等级: P2

缺陷描述: 每次调用都getenv而非缓存初始化时读取，运行时修改环境变量导致行为不一致。3条案例。

典型代码示例:

```cpp
// 缺陷 -- getenv every call, runtime modification causes inconsistency
bool IsIndependentOp() {
    const char *env = getenv("HCCL_INDEPENDENT_OP");
    return (env != nullptr && strcmp(env, "1") == 0);
}
// return value may differ within same process lifetime

// 修复 -- cache at init time
static bool g_isIndependentOp = false;
void InitConfig() {
    const char *env = getenv("HCCL_INDEPENDENT_OP");
    g_isIndependentOp = (env != nullptr && strcmp(env, "1") == 0);
}
bool IsIndependentOp() { return g_isIndependentOp; }
```

审查检查方法:
- 环境变量是否在初始化时一次性读取缓存? 运行时重复读取是否有意为之?

关联commit: `578c16b4`

---

### 类别: 代码质量/命名 (QUAL)

hcomm-dev 15条(9.3%)。命名不一致和拼写错误在通信框架的算法名称匹配场景中可能造成严重功能故障。

#### QUAL-01: 算法名称/变量名拼写错误 [hcomm-dev]

严重等级: P1

缺陷描述: Copy-Paste导致的算法名称错误、变量名多/少字符。7条案例。通信框架通过字符串匹配选择executor，拼写错误直接导致选错算法。

典型代码示例:

```cpp
// 缺陷 -- ReduceScatterV algorithm name missing "V" suffix (copied from ReduceScatter)
algName = "AlignedReduceScatterDoubleRingFor91093Executor";

// 修复
algName = "AlignedReduceScatterVDoubleRingFor91093Executor";
//                            ^ V suffix added
```

```cpp
// 缺陷 -- member variable misspelled as numBlockss_ (extra s)
u32 numBlockss_;              // JSON key: "_const_num_blockss" mismatches upstream

// 修复
u32 numBlocks_;               // JSON key: "_const_num_blocks"
```

审查检查方法:
- ReduceScatterV相关文件中的算法名称是否包含"V"后缀?
- Copy-Paste新函数后，函数名/算法名/变量名中的标识符是否全部替换?
- 算法名称字符串是否有拼写检查机制(建议引入cspell)

关联commit: `5bfaa1e5`, `1820333425906`

---

#### QUAL-02: 全局重命名同步不完整 [hcomm-dev]

严重等级: P2

缺陷描述: 大规模重命名(如blockDim -> numBlocks, invaild -> invalid)后测试代码或其他模块未同步更新。5条案例。

典型代码示例:

```cpp
// 缺陷 -- HcclCalcBlockDim renamed to HcclCalcNumBlocks, device UT not updated
// test/ut/device_test.cc
HcclCalcBlockDim(param);  // compile error: function no longer exists

// 修复
HcclCalcNumBlocks(param);
```

审查检查方法:
- 大规模重命名后，测试代码是否已同步更新(CI编译即可拦截)?
- 全局重命名应通过自动化脚本+编译验证执行
- grep确认旧名称在全仓无残留

关联commit: `36c8449d`, `fd23d1b6`

---

#### QUAL-03: 全代码库拼写错误渗入API [hcomm-dev]

严重等级: P3

缺陷描述: 常见拼写错误(invaild/vaild, recevie, at lest等)渗入枚举值、函数名、日志，部分影响API兼容性。

典型代码示例:

```cpp
// 缺陷 -- enum value misspelled
enum OpType {
    OP_SEND = 0,
    OP_RECV,
    OP_INVAILD    // should be OP_INVALID
};
void SetAicpuNotifyInvaild();

// 修复
enum OpType {
    OP_SEND = 0,
    OP_RECV,
    OP_INVALID
};
void SetAicpuNotifyInvalid();
```

审查检查方法:
- CI中引入cspell或codespell工具扫描代码注释和字符串字面量
- 枚举值/函数名中的单词是否拼写正确?

关联commit: `345be6c4`, `02d62eef`

---

#### QUAL-04: 复制粘贴后变量名未替换 [hccl-dev]

严重等级: P1

缺陷描述: 从功能相似的代码块复制后，未将所有变量名替换为新的上下文对应值，导致逻辑完全错误。

典型代码示例:

```bash
# 缺陷代码(commit 8222fcf8 introduced, 11b7211a fixed, build.sh)
# run_st() copied from run_ut(), condition variable not replaced
function run_st() {
  if [[ "X$ENABLE_UT" = "Xon" ]]; then  # should be ENABLE_ST
    ...
  fi
}

# 修复
function run_st() {
  if [[ "X$ENABLE_ST" = "Xon" ]]; then
    ...
  fi
}
```

审查检查方法:
- 当新增函数与已有函数结构高度雷同时，要求开发者列出所有替换点
- 构建脚本中函数名后缀(_ut/_st)与检查的变量名后缀(ENABLE_UT/ENABLE_ST)必须对应
- 可用grep确认: 函数名中的关键字是否在函数体中一致出现

关联commit: `8222fcf8`(引入), `11b7211a`(修复)

---

### 类别: 接口/API设计 (API)

通信框架经历V1->V2迁移、ACL->RT接口替换、open/closed双模式等架构演进，接口适配层面的设计缺陷频繁暴露。hcomm-dev 8条(4.9%)。

#### API-01: 抢跑依赖未发布API [hcomm-dev]

严重等级: P2

缺陷描述: 在依赖的SDK正式发布新接口前就在消费端使用，链接失败。3条案例。12天内对同一组文件进行5次方向性变更(乒乓Revert)。

典型代码示例:

```cpp
// 缺陷 -- rtModelGetId not yet published in runtime SDK
uint64_t modelId;
rtModelGetId(rtModel, &modelId);  // link error: undefined symbol

// 修复 -- fallback to published API or dlopen probe
uint64_t modelId = reinterpret_cast<uint64_t>(rtModel);  // fallback

// Or: dlopen dual-path fallback
void *handle = dlopen("libruntime.so", RTLD_LAZY);
auto func = dlsym(handle, "rtModelGetId");
if (func) {
    func(rtModel, &modelId);
} else {
    modelId = reinterpret_cast<uint64_t>(rtModel);  // fallback
}
```

审查检查方法:
- 跨团队接口迁移是否前置确认目标API在所有受支持版本已发布?
- 是否有dlopen/weak symbol的兼容性降级方案?

关联commit: `30e25e50`, `1d8e2c14`

---

#### API-02: 返回值语义差异未适配 [hcomm-dev]

严重等级: P1

缺陷描述: 底层接口替换后返回值语义变化(bitmask vs enum)，调用方解析逻辑未同步适配。3条案例。

典型代码示例:

```cpp
// 缺陷 -- ACL returns bitmask, RT returns enum, parsing logic unchanged
// ACL: aclrtGetDevicesTopo returns link type bitmask (combinable)
// RT:  rtGetPairPhyDevicesInfo returns single enum value
u32 linkType = rtGetPairPhyDevicesInfo(devA, devB);
if (linkType & LINK_TYPE_HCCS) {  // bitmask parsing of enum -> logic error
    ...
}

// 修复 -- adapt to enum semantics
u32 linkType = rtGetPairPhyDevicesInfo(devA, devB);
if (linkType == LINK_TYPE_HCCS) {  // single-value comparison
    ...
}
```

审查检查方法:
- 返回值类型差异(bitmask vs enum, signed vs unsigned)在迁移时调用方解析逻辑是否同步适配?
- API返回裸指针时，所有权(谁分配谁释放)是否在注释中明确文档化?

关联commit: `1d8e2c14`

---

#### API-03: 封装层能力不完整 [hcomm-dev]

严重等级: P2

缺陷描述: 适配层未覆盖底层完整能力，或V2 weak symbol函数声明与实际签名不匹配。2条案例。

审查检查方法:
- 封装层新增接口后，是否覆盖了底层完整能力?
- V2 weak symbol函数的签名是否与实际函数完全一致?

关联commit: `47020bd5`, `26f5813f`

---

### 类别: 硬件/平台适配 (HW)

通信库需要适配多种芯片(910B/910_93/910_95/A5等)和多种连接协议(PCIE/RoCE/HCCS/URMA)，硬件协议约束和设备差异性是低可审查性缺陷的主要来源。hcomm-dev 6条(3.7%)。

#### HW-01: 硬件协议乘数因子遗漏 [hcomm-dev]

严重等级: P0

缺陷描述: URMA约束每个SQE含4个WQEBB，多处代码计算SQ buffer大小和VA映射长度时遗漏WQEBB_NUM_PER_SQE(=4)乘数因子，device侧VA空间只有实际需要的1/4，内存越界。3条案例。

典型代码示例:

```cpp
// 缺陷 -- sqDepth missing WQEBB_NUM_PER_SQE multiplier
UbConnLite::UbConnLite(..., u32 sqDepth, ...) {
    sqDepth_ = sqDepth;  // missing multiplier
}
u64 sqBufferSize = sqDepth_ * WQE_BB_SIZE;  // missing * WQEBB_NUM_PER_SQE
// VA space is only 1/4 of actual need -> out of bounds

// 修复 -- multiply uniformly
UbConnLite::UbConnLite(..., u32 sqDepth, ...) {
    sqDepth_ = sqDepth * WQEBB_NUM_PER_SQE;  // 4 WQEBB per SQE
}
u64 sqBufferSize = sqDepth_ * WQE_BB_SIZE;  // sqDepth_ already includes multiplier
```

```cpp
// 缺陷 -- macro value collision
#define URMA_JFS_PI_TYPE     0x0011
#define URMA_JFS_DB_STATUS   0x0011  // wrong: same as PI_TYPE
// Reading DB_STATUS actually reads PI_TYPE register

// 修复
#define URMA_JFS_DB_STATUS   0x000f  // correct value
```

审查检查方法:
- 涉及硬件协议约束的乘数因子是否在协议层统一封装为宏/函数? 避免多处手动乘算
- 同一组宏常量定义中是否存在重复值? (static_assert或编译期检查)
- SQ深度等协议参数的"单位"(SQE数/WQEBB数/字节)是否在类型或命名中体现?

关联commit: `df96667c`, `39ad6c07`

---

#### HW-02: 设备类型特殊路径未处理 [hcomm-dev]

严重等级: P1

缺陷描述: 新增设备类型的初始化/资源管理路径与已有设备不同。3条案例。

典型代码示例:

```cpp
// 缺陷 -- 910_95 does not need SetIpc()
CHK_RET(notify->SetIpc());  // fails on 910_95

// 修复 -- branch by device type
if (deviceType_ != DEV_TYPE_910_95) {
    CHK_RET(notify->SetIpc());
}
```

```cpp
// 缺陷 -- notify offset uses simple linear calc, wrong for slice-based address space
u64 offset = notifyId * notifySize;  // wrong for 910B/910_93 when notifyId>=512

// 修复 -- device-specific calculation
u64 offset;
if (devType == DEV_TYPE_910B || devType == DEV_TYPE_910_93) {
    u32 sliceId = notifyId / 512;
    u32 offsetInSlice = notifyId % 512;
    offset = offsetInSlice * notifySize + sliceId * 0x10000;  // 64KB per slice
} else {
    offset = notifyId * notifySize;
}
```

审查检查方法:
- 新增设备类型时，是否审查了所有资源初始化路径的分支处理?
- 是否有设备兼容性矩阵文档?

关联commit: `b9f46705`, `a0e420e9`

---

#### HW-03: 硬件寄存器/地址映射宏定义冲突 [hcomm-dev]

严重等级: P0

缺陷描述: 宏常量值重复导致寄存器访问错误。

审查检查方法:
- 同一组寄存器偏移宏是否通过static_assert确保无重复值?
- 新增寄存器映射宏时是否检查了已有定义?

关联commit: `39ad6c07`

---

### 类别: C++语言特性 (CPP)

#### CPP-01: 跨SO边界的多态类虚析构不应使用 = default [hccl]

严重等级: P1

缺陷描述: `= default`的虚析构函数由编译器在每个翻译单元隐式内联生成。当基类和派生类位于不同SO时，链接器可能找不到虚析构函数的符号。

典型代码示例:

```cpp
// 缺陷代码 -- pkg_inc/hcomm/ccu/ccu_kernel.h (included by multiple SOs)
~CcuKernel() override = default;  // inlined per TU, cross-SO symbol missing

// 修复代码
// ccu_kernel.h
~CcuKernel() override;

// ccu_kernel.cc
CcuKernel::~CcuKernel() {}  // explicit definition in single TU
```

审查检查方法:
- 在pkg_inc或公共头文件中导出的多态类，虚析构函数不应使用`= default`
- 应在.cc文件中提供显式定义

关联commit: `bb490dc2`

---

### 跨类别系统性风险 (SYS)

以下模式跨越多个缺陷类别，是code review时应优先关注的系统性风险。

#### SYS-01: 缺陷修复提交须额外审查是否引入新缺陷 [hccl]

严重等级: P1

缺陷描述: `4e82ec25`修复API参数方向问题时引入了use-after-free(将局部vector的data指针返回)，被`6784944a`再次修复。修复提交的审查重心往往只在"原缺陷是否被修"，忽略了修复代码本身的正确性。

审查检查方法:
- 缺陷修复提交须有额外审查关注点: 修复代码本身是否引入新问题
- 修复提交中新增的代码应按新代码标准审查，而非仅关注是否解决了原bug

关联commit: `4e82ec25` -> `6784944a`

---

#### SYS-02: 修复一处须搜索重复代码段 [hccl][hcomm-dev]

严重等级: P2

缺陷描述: 热点文件中存在大量代码重复(如ExecOp vs ExecOpAlltoAll、AllocTransportResource三路重复)，修复一处时忘记修复重复代码中的同样问题。hcomm-dev中op_base.cc的InitCommConfig序列、hccl_communicator.cc中Suspend/StopExec/Clean三函数均有大段重复。

审查检查方法:
- 修复一处缺陷后，全局搜索相似代码段是否有同样问题
- 对重复代码优先重构为共享函数，从根本上消除"修一漏一"
- 修改超大文件中的某一处逻辑时，必须搜索该文件内和跨文件的所有同模式代码，确保同步修改

关联证据: hccl_communicator_host.cc(ExecOp重复), aicpu_communicator.cc(AllocTransportResource三路重复), communicator_impl.cc(5个初始化路径), op_base.cc(InitCommConfig序列)

---

#### SYS-03: God Object文件修改须加强审查 [hccl][hcomm-dev]

严重等级: P3

缺陷描述: 多个超大文件集中了大量缺陷修复。hccl仓库三个热点文件占全部缺陷修复的36.9%:

| 文件                        | 规模              | 缺陷修复次数 |
|-----------------------------|-------------------|-------------|
| hccl_communicator_host.cc   | 312方法/9152行    | 13次(hccl)  |
| aicpu_communicator.cc       | 213函数/5550行    | 9次(hccl)   |
| communicator_impl.cc        | ~100函数/3789行   | 9次(hccl)   |

hcomm-dev同样: hccl_communicator_host.cc(8818行, 14次), hcom.cc(4068行, 14次), aicpu_communicator.cc(5809行, 12次), op_base.cc(4587行, 11次)。

审查检查方法:
- 任何对这些文件的修改应要求至少2个reviewer
- 长期策略: 拆分God Object为职责单一的小类

---

#### SYS-04: 返回值系统性忽略 [hccl]

严重等级: P1

缺陷描述: 热点文件中多处忽略关键函数返回值:
- `hrtMalloc`返回值未检查(hccl_communicator_host.cc L1803)
- `malloc` 100MB未检查返回值(communicator_impl.cc L3115)
- `getenv`返回nullptr直接解引用(communicator_impl.cc L3025)

审查检查方法:
- 内存分配函数(malloc/hrtMalloc)返回值必须检查
- getenv()返回值必须判空
- 编译选项开启 `-Wunused-result`

---

#### SYS-05: 编译器可捕获的缺陷须在CI中强制开启 [hccl]

严重等级: P2

缺陷描述: 以下缺陷本可由编译器警告捕获但仍进入主干:

| 编译选项                    | 可捕获的缺陷   | 关联commit         |
|-----------------------------|---------------|---------------------|
| `-Wshadow`                  | 变量遮蔽       | ef766683           |
| `-Wtautological-compare`    | 自比较         | e1880dc1           |
| `-Wformat`                  | 格式串不匹配   | 35e32c7c, 6a6eac0f |
| `-Wconversion`              | 整数截断       | 9c1f957b, 6baf33c4 |
| `-Werror=enum-conversion`   | 枚举隐式转换   | d953cbf3           |

审查检查方法:
- CI中强制开启上述警告并设为 `-Werror`
- 新增代码不允许suppress这些警告

---

#### SYS-06: 热点文件已知高危风险 [hccl][hcomm-dev]

严重等级: P1

hccl仓库(截至分析时点):

| 风险         | 位置                         | 描述                              |
|-------------|------------------------------|-----------------------------------|
| 悬空指针     | communicator_impl.cc L601    | 临时unique_ptr.get()传出          |
| malloc未检查 | communicator_impl.cc L3115   | malloc 100MB未检查返回值          |
| getenv空指针 | communicator_impl.cc L3025   | getenv返回nullptr直接解引用       |
| TLV死循环   | aicpu_communicator.cc L659   | length==0时while循环无法前进      |
| 锁策略不一致 | aicpu_communicator.cc resMap_ | 同一数据结构两种锁保护           |

hcomm-dev仓库(截至分析时点):

| 风险               | 位置                            | 描述                                                    |
|-------------------|---------------------------------|---------------------------------------------------------|
| head/tail赋值反转 | aicpu_communicator.cc:3385-3386 | UpdateSqStatus中SQ_TAIL写入head、SQ_HEAD写入tail        |
| 确定性UAF         | hcom.cc:2076                    | const_cast(str.c_str())返回局部变量内部指针              |
| 宏硬编码失效      | adapter_rts.cc:37-45            | result硬编码为0，event替换功能完全失效                   |

---

#### SYS-07: 并发安全缺乏系统性设计 [hcomm-dev]

18个热点文件中15个存在并发安全问题。模式包括:
- TOCTOU竞态: hcom.cc(HcomCreateGroupImplHeterog), hcom_common.cc(HcomGetCurHcomCtx)
- 无锁全局变量: op_base.cc(g_oneSidedCommHcomInfos), aicpu_communicator.cc(errMessageReport_), adapter_rts.cc(g_localDeviceType), hcom.cc(g_rankTableSetInfo)
- 锁范围不一致: op_base.cc中HcclSendInner/HcclRecvInner等6个函数缺少operatorlock_

根因: 锁的使用缺乏统一的层级设计和文档化约定。

审查规则: 对static/全局变量的新增或修改，必须标注线程安全声明(thread_local/互斥锁/仅单线程访问)。

---

#### SYS-08: 公共API/ABI设计债务 [hcomm-dev]

严重缺陷:
- hcom.h: extern "C"块内使用namespace、公共函数签名含C++默认参数、头文件中定义non-inline std::string/std::map全局变量
- hccl_comm_pub.h: 条件编译(CCL_KERNEL_AICPU/HCCD)改变类内存布局，跨模块传指针越界
- 多个API返回悬垂指针: hcom.cc:2076 `*algo = const_cast<char*>(str.c_str())`将局部变量内部指针返回(确定性UAF)

审查规则: 公共头文件(pkg_inc/)的任何修改必须通过ABI兼容性checklist审查。

---

#### SYS-09: 大爆炸提交掩盖关键变更 [hcomm-dev]

Revert分析揭示的最强信号:
- Revert#1原始提交: "update"(678文件)中隐藏心跳帧结构膨胀30倍
- Revert#4: "ars code"(51文件, 2000+行, 跨4层)仅两个单词commit message
- 功能变更混入大批量同步提交，审查者无法在海量无关变更中发现关键问题

审查规则: 超过20个文件或500行变更的PR必须拆分。功能变更不得混入同步/批量提交中。
