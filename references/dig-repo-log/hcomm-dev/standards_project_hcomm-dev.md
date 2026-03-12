## hcomm-dev项目缺陷模式与审查规则

基于hcomm-dev仓库488次提交(排除merge)的完整git历史分析，从162条缺陷提交(占比33.1%)中提炼的40条高价值审查规则。每条规则均有commit证据和实际代码支撑。

hcomm-dev是HCCL通信框架的开发仓库(2025-11 ~ 2026-01)，缺陷模式与算子仓库(ops-transformer)有本质差异。通信库侧重并发/资源生命周期/协议/跨设备适配，而非精度/tiling/kernel参数。本文档的缺陷类别从162条缺陷数据中自然涌现，未套用算子仓库框架。

### 严重等级定义

| 等级 | 含义 | 影响范围 |
|------|------|----------|
| P0 | 致命 | CoreDump/Hang/数据损坏/内存越界/UAF |
| P1 | 严重 | 特定配置崩溃/功能不可用/静默精度错误/OOM |
| P2 | 一般 | 构建失败/边界条件异常/日志误导 |
| P3 | 建议 | 代码质量/可维护性/潜在隐患 |

### 缺陷分布总览

| 序号 | 缺陷类别 | 频次 | 占比 | 规则数 | 规则ID前缀 |
|------|---------|------|------|--------|-----------|
| 1 | 构建/编译/链接缺陷 | 24 | 14.8% | 4 | BUILD |
| 2 | 初始化/赋值/时序缺陷 | 17 | 10.5% | 4 | INIT |
| 3 | 条件判断/分支覆盖不完整 | 16 | 9.9% | 3 | BRANCH |
| 4 | 数据类型/参数/枚举缺陷 | 16 | 9.9% | 4 | TYPE |
| 5 | 代码质量/命名/拼写 | 15 | 9.3% | 3 | QUAL |
| 6 | 资源管理缺陷 | 12 | 7.4% | 3 | RES |
| 7 | 并发/线程安全缺陷 | 11 | 6.8% | 4 | CONC |
| 8 | 日志/可观测性缺陷 | 11 | 6.8% | 3 | LOG |
| 9 | 状态/缓存/超时管理缺陷 | 10 | 6.2% | 3 | STATE |
| 10 | 空指针/入参校验缺失 | 8 | 4.9% | 3 | PTR |
| 11 | 接口/API设计与适配缺陷 | 8 | 4.9% | 3 | API |
| 12 | 硬件/平台适配缺陷 | 6 | 3.7% | 3 | HW |
| - | 跨类别系统性风险 | - | - | 5 | SYS |

可审查性分布: 高86条(53.1%), 中54条(33.3%), 低22条(13.6%)。超半数缺陷可在code review阶段拦截。

---

### 类别一: 构建/编译/链接缺陷(24条, 14.8%)

hcomm-dev中频次最高的缺陷类别。根本原因是构建系统复杂度随多编译目标(host/device/kernel/daemon)和多构建模式(open/closed/HCCD/CCL_KERNEL)增长而爆炸。

#### 规则 BUILD-01: CMakeLists源文件遗漏

严重等级: P2

缺陷描述: 新增.cc文件后忘记在CMakeLists.txt中添加，导致功能不被编译链接。8条案例。另有源文件同时被添加到两个CMakeLists.txt中导致符号重复定义。

典型代码示例:

```cmake
# 缺陷 — alltoallv continuous pipeline的两个.cc文件遗漏
set(SOURCES
    ...
    coll_all_to_all_v_executor.cc
    # 遗漏: alltoallv_continuous_pipeline.cc
    # 遗漏: coll_all_to_all_v_continuous_pipeline_executor.cc
    ...
)

# 修复 — 同步添加新增源文件
set(SOURCES
    ...
    coll_all_to_all_v_executor.cc
    alltoallv_continuous_pipeline.cc
    coll_all_to_all_v_continuous_pipeline_executor.cc
    ...
)
```

审查检查方法:
- 新增.cc文件的PR是否同步修改了CMakeLists.txt?
- 同一.cc文件是否出现在多个CMakeLists.txt中(符号重复)?
- CI是否覆盖所有编译目标的构建验证?

关联commit: `8c424d41`, `20125a56`

---

#### 规则 BUILD-02: 条件编译/编译目标不兼容

严重等级: P2

缺陷描述: 在特定编译目标下引入不可用的外部依赖，或条件编译块覆盖不完整。7条案例。常见模式: 函数调用了ACL API但CCL_KERNEL_AICPU/HCCD目标下ACL不可用。

典型代码示例:

```cpp
// 缺陷 — ParseCannVersion()调用ACL API，HCCD目标下不可用
HcclResult ParseCannVersion() {
    const char *version = aclGetVersion();  // CCL_KERNEL_AICPU/HCCD下链接失败
    ...
}

// 修复 — 条件编译包裹
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

#### 规则 BUILD-03: ABI兼容性/符号导出

严重等级: P1

缺陷描述: 公共头文件中使用内部类型、导出C++符号到C linkage、动态库导出函数使用std::string参数等。5条案例。

典型代码示例:

```cpp
// 缺陷 — 公共头文件hcomm_primitives.h中uint32_t改为内部typedef u32
// 外部用户编译时找不到u32定义，ABI破坏
typedef unsigned int u32;  // 内部头文件
void HcommFunc(u32 param);  // pkg_inc/中使用内部typedef

// 修复 — 公共头文件仅使用标准C/C++类型
void HcommFunc(uint32_t param);

// 另一缺陷 — 动态库导出函数用std::string&参数
HcclResult HcomSelectAlg(const char *tag, std::string &algName);  // 跨ABI不兼容

// 修复 — 改为C兼容POD类型
HcclResult HcomSelectAlg(const char *tag, char *algName, uint32_t algNameLen);
```

审查检查方法:
- 公共头文件(pkg_inc/)中是否仅使用标准C/C++类型，禁止内部typedef?
- 动态库导出函数(extern "C")的参数和返回值是否为POD类型?
- stub符号与实际函数签名是否匹配?

关联commit: `30e25e50`, `7fccc101`, `e6d12932`

---

#### 规则 BUILD-04: 环境/配置适配遗漏

严重等级: P1

缺陷描述: 环境变量解析、设备可见性配置等边界条件处理不足。4条案例。

典型代码示例:

```cpp
// 缺陷 — ASCEND_RT_VISIBLE_DEVICES只配1个设备时backup设备不在可见列表
HcclResult ret = hrtGetDeviceIndexByPhyId(backupDevId, &logicId);
// backupDevId不在可见列表中，调用失败

// 修复 — 移到条件分支内部按需调用
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

### 类别二: 初始化/赋值/时序缺陷(17条, 10.5%)

通信框架对象生命周期早期阶段的脆弱性。核心模式是多分支初始化路径中的遗漏和成员变量初始化缺失。

#### 规则 INIT-01: 多分支初始化路径不对称

严重等级: P1

缺陷描述: 同一函数存在V1/V2、图模式/单算子模式等多条初始化分支，其中某条遗漏了必要的初始化步骤。7条案例。

典型代码示例:

```cpp
// 缺陷 — V2路径创建communicator后遗漏HcomSetGroupTopoInfo
// V2路径
auto lambda = [&]() {
    CHK_RET(CreateCommunicator(...));
    // 遗漏: CHK_RET(HcomSetGroupTopoInfo(...));
    return HCCL_SUCCESS;
};
HCCLV2_FUNC_RUN(lambda);

// 非V2路径(已正确调用)
CHK_RET(CreateCommunicator(...));
CHK_RET(HcomSetGroupTopoInfo(...));

// 修复 — V2路径补充与非V2对称的调用
auto lambda = [&]() {
    CHK_RET(CreateCommunicator(...));
    CHK_RET(HcomSetGroupTopoInfo(...));  // 补充
    return HCCL_SUCCESS;
};
```

审查检查方法:
- 同一函数的多条初始化分支是否执行了相同的必要初始化步骤(对称性检查)?
- V2适配路径是否完整覆盖了V1路径的所有初始化步骤?
- 对比两个分支的函数调用序列，寻找不对称项

关联commit: `65f0c2ba`, `b54e3852`

---

#### 规则 INIT-02: 成员变量未初始化/默认值错误

严重等级: P0

缺陷描述: C++内置类型成员未显式初始化，运行时包含垃圾值导致非确定性行为。5条案例。

典型代码示例:

```cpp
// 缺陷 — 多个成员(含6个内置类型+4个容器类型)缺少类内初始化器
class HcclCommunicator {
    bool isGroupMode_;       // 垃圾值 -> group模式判断异常
    u32 iSend;               // 垃圾值
    u32 iRecv;
    u32 nSend;
    u32 nRecv;
    u32 bufferSliceNum;
};

// 修复 — 显式初始化
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

#### 规则 INIT-03: 执行顺序/依赖错误

严重等级: P0

缺陷描述: 操作执行顺序错误，或状态变量清除在依赖该状态的调用之前。5条案例。

典型代码示例:

```cpp
// 缺陷 — 先启动线程后设置标志，线程读到false
AicpuHcclProcess::CallMC2MaintenanceThread(ctx);  // 线程检查标志
indOpCommInitialized_ = true;                       // 标志尚未设置

// 修复 — 先设置标志再启动线程
indOpCommInitialized_ = true;
AicpuHcclProcess::CallMC2MaintenanceThread(ctx);
```

```cpp
// 缺陷 — executor_在CalBlockDim调用之前被置为nullptr
executor_ = nullptr;                  // 过早置空
HcclResult ret = CalBlockDim(...);    // CalBlockDim内部需要executor_

// 修复 — 置空操作移到CalBlockDim之后
HcclResult ret = CalBlockDim(...);
if (ret != HCCL_SUCCESS) {
    executor_ = nullptr;              // 移入错误处理分支
    return ret;
}
```

审查检查方法:
- 状态变量清除操作(置nullptr/置0)是否在所有依赖该状态的调用之后?
- 线程启动前，线程内读取的共享状态是否已完成初始化?

关联commit: `4d1a3a60`, `08b2658a`, `87da187b`

---

#### 规则 INIT-04: offload/lambda变量作用域泄漏

严重等级: P1

缺陷描述: HCCLV2_FUNC_RUN宏的lambda外部提前做了cast和使用，offload模式下控制流未正确进入V2分支，变量在错误作用域被访问。

典型代码示例:

```cpp
// 缺陷 — lambda外部使用了应在V2分支内的变量
auto *comm = static_cast<hcclComm*>(handle);  // lambda外部cast
comm->DoSomething();                            // 非V2模式下不应执行
auto lambda = [&]() {
    HCCLV2_FUNC_RUN(comm->V2Method());
    return HCCL_SUCCESS;
};

// 修复 — cast和使用都移入lambda内部
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

### 类别三: 条件判断/分支覆盖不完整(16条, 9.9%)

通信框架中算法选择(SelectAlg)、设备类型分支、V1/V2路径等分支逻辑极为复杂。遗漏特定拓扑或运行模式的分支是高频缺陷来源。

#### 规则 BRANCH-01: 枚举/类型分支遗漏

严重等级: P1

缺陷描述: 新增枚举值后未同步更新所有switch/if分支。7条案例。

典型代码示例:

```cpp
// 缺陷 — engine类型检查只允许AICPU，遗漏AICPU_TS
if (engineType != AICPU) {
    HCCL_ERROR("engine type[%u] is not support", engineType);
    return HCCL_E_NOT_SUPPORT;
}

// 修复 — 补充AICPU_TS
if (engineType != AICPU && engineType != AICPU_TS) {
    HCCL_ERROR("engine type[%u] is not support", engineType);
    return HCCL_E_NOT_SUPPORT;
}
```

```cpp
// 缺陷 — AIV算法选择未排除不支持的非对称拓扑
// multiModuleDiffDeviceNumMode_为true时强行走AIV路径导致hang
if (aivCoreNum > 0) {
    return SelectAIVAlg(algName);  // 非对称拓扑不支持AIV
}

// 修复 — 增加拓扑约束检查
if (aivCoreNum > 0 && !multiModuleDiffDeviceNumMode_) {
    return SelectAIVAlg(algName);
}
```

审查检查方法:
- 新增枚举值时，是否全局搜索了所有switch/if判断并同步更新?
- AIV算法选择函数的拓扑约束checklist是否完整覆盖?

关联commit: `802c0411`, `c6f8cc36`

---

#### 规则 BRANCH-02: V2/offload适配路径缺失

严重等级: P1

缺陷描述: 新增API函数后未在V2头文件中声明weak symbol和添加HCCLV2_FUNC_RUN分发。5条案例。

典型代码示例:

```cpp
// 缺陷 — HcclGetHcclBuffer等多个API缺少V2分发
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
    // 只有V1逻辑，V2环境下行为不正确
    ...
}

// 修复 — 添加V2分发
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
#if defined(OPEN_BUILD_PROJECT) && defined(ORION_MODE)
    HCCLV2_FUNC_RUN(HcclGetHcclBufferV2(comm, buffer));
#endif
    // V1 fallback逻辑
    ...
}
```

审查检查方法:
- 新增Hccl*/Hcom*公共API时，是否在V2头文件中声明了weak symbol并在函数体内添加HCCLV2_FUNC_RUN?
- 搜索仓库中所有没有HCCLV2_FUNC_RUN的公共API入口

关联commit: `e25c6484`

---

#### 规则 BRANCH-03: 条件方向错误/条件编译覆盖运行时分支

严重等级: P0

缺陷描述: 条件判断写反，或#ifdef块无条件覆写了上方的运行时分支判断。4条案例。

典型代码示例:

```c
// 缺陷 — #ifdef CONFIG_CONTEXT块无条件覆写运行时分支
// 上方已有正确的分支判断
phyId = (attr->protocol == PROTOCOL_RDMA) ?
        attr->dev.rdma.phyId : attr->ub.phy_id;

#ifdef CONFIG_CONTEXT
    phyId = attr->ub.phy_id;  // 无条件覆写，RDMA时使用了错误的phyId
#endif

// 修复 — 条件编译块内保持与上方一致的分支语义
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

### 类别四: 数据类型/参数/枚举缺陷(16条, 9.9%)

通信框架中类型安全问题突出，包括枚举值误用为算术因子、类型隐式截断、map::at()异常、参数对象混淆等。

#### 规则 TYPE-01: 枚举值语义误用

严重等级: P0

缺陷描述: 将枚举值当作字节大小、数组索引等用于算术运算。5条案例。

典型代码示例:

```cpp
// 缺陷 — dataType枚举值被直接用作除数
// reduceInfo.dataType是枚举值(如HCCL_DATA_TYPE_FP32=3)，不是字节数(4)
u64 count = src->len / reduceInfo.dataType;  // 用枚举值做除法，结果错误

// 修复 — 通过SIZE_TABLE查表获取字节数
u64 count = src->len / SIZE_TABLE[reduceInfo.dataType];  // 查表获取实际字节数
```

```cpp
// 缺陷 — 比较了错误的枚举常量
bool IsOverFlowInfNanMode() {
    aclrtFloatOverflowMode mode = GetOverflowMode();
    return (mode == ACL_RT_OVERFLOW_MODE_UNDEF);  // 错误: 应比较INFNAN而非UNDEF
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

#### 规则 TYPE-02: 类型隐式截断/溢出

严重等级: P1

缺陷描述: 函数返回类型或变量类型比内部计算类型窄。4条案例。

典型代码示例:

```cpp
// 缺陷 — u32超时值通过u8返回类型截断，最大只能表示255秒
u8 GetKernelExecTimeoutFromEnvConfig() {
    u32 timeout = GetEnvConfig("KERNEL_EXEC_TIMEOUT");
    return timeout;  // u32 -> u8隐式截断
}

// 修复 — 返回类型匹配内部计算类型
u32 GetKernelExecTimeoutFromEnvConfig() {
    u32 timeout = GetEnvConfig("KERNEL_EXEC_TIMEOUT");
    return timeout;
}
```

审查检查方法:
- 函数返回类型是否与内部实际计算类型一致? -Wconversion可检出
- 赋值时窄类型接收宽类型值是否有显式范围检查?

关联commit: `4ebb11d1`, `7a3d6fa6`

---

#### 规则 TYPE-03: 参数对象混淆/引用错误

严重等级: P0

缺陷描述: 传入的参数使用了错误的对象，或参数被声明但未使用。4条案例。

典型代码示例:

```cpp
// 缺陷 — 应从dstThreadPtr获取notify，实际从threadPtr(源线程)获取
void HcommThreadNotifyRecordOnThread(Thread *threadPtr, Thread *dstThreadPtr) {
    auto notify = threadPtr->GetNotify();    // 错误: 从源线程获取
    // 参数dstThreadPtr传入却未使用
    ...
}

// 修复 — 使用正确的参数
void HcommThreadNotifyRecordOnThread(Thread *threadPtr, Thread *dstThreadPtr) {
    auto notify = dstThreadPtr->GetNotify();  // 从目标线程获取
    ...
}
```

审查检查方法:
- 函数参数被声明但未使用时是否告警(unused parameter)?
- 多个同类型参数的函数，调用处传参顺序是否正确?

关联commit: `d89396c9`

---

#### 规则 TYPE-04: map::at()对未收录key抛异常

严重等级: P0

缺陷描述: std::map::at()在key不存在时抛std::out_of_range异常，C++异常未被捕获导致coredump。新增枚举值后未更新对应map。3条案例。

典型代码示例:

```cpp
// 缺陷 — errorMap未收录新增错误码，at()抛异常
std::string msg = errorMap.at(code);  // HCCL_E_SUSPENDING等未收录 -> 异常 -> coredump

// 修复 — 改用find()+迭代器检查
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

### 类别五: 代码质量/命名/拼写(15条, 9.3%)

累积频次高。命名不一致和拼写错误在通信框架的算法名称匹配场景中可能造成严重功能故障。

#### 规则 QUAL-01: 算法名称/变量名拼写错误

严重等级: P1

缺陷描述: Copy-Paste导致的算法名称错误、变量名多/少字符。7条案例。通信框架通过字符串匹配选择executor，拼写错误直接导致选错算法。

典型代码示例:

```cpp
// 缺陷 — ReduceScatterV的算法名写成ReduceScatter(缺少"V"后缀)
// 从ReduceScatter代码复制来后忘改
algName = "AlignedReduceScatterDoubleRingFor91093Executor";

// 修复 — 补全"V"后缀
algName = "AlignedReduceScatterVDoubleRingFor91093Executor";
//                            ^ 补全V后缀
```

```cpp
// 缺陷 — 成员变量误拼为numBlockss_(多一个s)
u32 numBlockss_;              // JSON key: "_const_num_blockss" 与上游不一致
// 修复
u32 numBlocks_;               // JSON key: "_const_num_blocks"
```

审查检查方法:
- ReduceScatterV相关文件中的算法名称是否包含"V"后缀?
- Copy-Paste新函数后，函数名/算法名/变量名中的标识符是否全部替换?
- 算法名称字符串是否有拼写检查机制(建议引入cspell)

关联commit: `5bfaa1e5`, `1820333425906`

---

#### 规则 QUAL-02: 全局重命名同步不完整

严重等级: P2

缺陷描述: 大规模重命名(如blockDim -> numBlocks, invaild -> invalid)后测试代码或其他模块未同步更新。5条案例。

典型代码示例:

```cpp
// 缺陷 — HcclCalcBlockDim重命名为HcclCalcNumBlocks后，device侧UT未同步
// test/ut/device_test.cc
HcclCalcBlockDim(param);  // 编译失败: 函数已不存在

// 修复 — 同步更新测试代码
HcclCalcNumBlocks(param);
```

审查检查方法:
- 大规模重命名后，测试代码是否已同步更新(CI编译即可拦截)?
- 全局重命名应通过自动化脚本+编译验证执行
- grep确认旧名称在全仓无残留

关联commit: `36c8449d`, `fd23d1b6`

---

#### 规则 QUAL-03: 全代码库拼写错误渗入API

严重等级: P3

缺陷描述: 常见拼写错误(invaild/vaild, recevie, at lest等)渗入枚举值、函数名、日志，部分影响API兼容性。

典型代码示例:

```cpp
// 缺陷 — 枚举值拼写错误
enum OpType {
    OP_SEND = 0,
    OP_RECV,
    OP_INVAILD    // invalid拼成invaild
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

### 类别六: 资源管理缺陷(12条, 7.4%)

通信框架中资源种类繁多(device memory, IPC handle, transport link, notify, stream, QP等)，生命周期跨越多个组件，释放时序和所有权归属是核心难题。

#### 规则 RES-01: 资源释放顺序错误

严重等级: P0

缺陷描述: 组件间存在资源依赖，析构顺序不正确导致UAF或报错。4条案例。

典型代码示例:

```cpp
// 缺陷 — communicator析构时先销毁AlgResource，
// 但ChannelManager中持有的transport link依赖底层资源已被清理
HcclCommunicator::~HcclCommunicator() {
    DestroyAlgResource();       // 先释放算法资源
    // channelManager_析构时transport link访问已释放资源 -> 报错
}

// 修复 — 新增releaseChannel_()回调注入析构流程，确保transport link在资源释放前清理
HcclCommunicator::~HcclCommunicator() {
    DestroyAlgResource();
    releaseChannel_();  // 通过回调释放channel transport，避免UAF
}
```

```cpp
// 缺陷 — UnimportAllJetty释放资源后handle仍保留旧值，重入时double free
void CcuComponent::UnimportAllJetty() {
    for (auto &jetty : jettyList_) {
        UrmaUnimportJetty(jetty.handle);
        // 遗漏: jetty.handle = INVALID_HANDLE;
    }
}

// 修复 — 释放后立即置空
void CcuComponent::UnimportAllJetty() {
    for (auto &jetty : jettyList_) {
        UrmaUnimportJetty(jetty.handle);
        jetty.handle = INVALID_HANDLE;  // 防止重入double free
    }
}
```

审查检查方法:
- 当组件A持有组件B创建的资源引用时，析构流程是否保证A的引用先释放?
- 资源释放后是否立即将handle/指针置空?

关联commit: `f45581ee`, `e17cfff4`

---

#### 规则 RES-02: 资源重复管理/泄漏

严重等级: P1

缺陷描述: 同一段内存/资源被多个路径独立管理，导致重复映射或分配过量。4条案例。

典型代码示例:

```cpp
// 缺陷 — P2P传输中input/output已在CCL buffer大块范围内
// 仍对其独立做IPC注册，重复映射导致IPC内存泄漏最终OOM
void TransportP2P::Init() {
    SetMemoryName(input_);    // input_在machinePara_.mem[0]范围内
    SetMemoryName(output_);   // 重复注册 -> IPC泄漏
    ...
}

// 修复 — 引入isMemInclude_检测地址范围包含关系
void TransportP2P::Init() {
    isMemInclude_ = IsAddrInRange(input_, machinePara_.mem[0]);
    if (!isMemInclude_) {
        SetMemoryName(input_);  // 仅独立区域需要注册
    }
    ...
}
```

审查检查方法:
- 同一段内存/资源是否存在"子区域被父区域管理但仍被独立管理"的重复路径?
- 内存分配逻辑的每个分支是否有算法文档或注释支撑其buffer大小计算?

关联commit: `f64f6498`, `9277e023`

---

#### 规则 RES-03: 错误路径资源泄漏

严重等级: P1

缺陷描述: 正常路径正确释放但CHK_RET提前返回时遗漏释放。4条案例。

典型代码示例:

```cpp
// 缺陷 — hrtMalloc分配的device内存在CHK_RET失败路径未释放
void *devMem = nullptr;
CHK_RET(hrtMalloc(&devMem, size));
CHK_RET(SomeOperation(devMem));  // 失败时devMem泄漏
CHK_RET(AnotherOp(devMem));
hrtFree(devMem);

// 修复 — 使用RAII或显式清理
void *devMem = nullptr;
CHK_RET(hrtMalloc(&devMem, size));
HcclResult ret = SomeOperation(devMem);
if (ret != HCCL_SUCCESS) {
    hrtFree(devMem);
    return ret;
}
```

```cpp
// 缺陷 — localRmaBufferMgr插入后Init()失败，残留未初始化buffer
localRmaBufferMgr->Add(tempKey, buffer);
CHK_RET(buffer->Init());  // Init失败 -> 已插入但未初始化的buffer残留 -> 后续访问coredump

// 修复 — Init失败后清理已插入条目
localRmaBufferMgr->Add(tempKey, buffer);
HcclResult ret = buffer->Init();
if (ret != HCCL_SUCCESS) {
    localRmaBufferMgr->Del(tempKey);  // 回滚插入操作
    return ret;
}
```

审查检查方法:
- 每个CHK_RET/提前return路径上，之前分配的资源是否已释放? (RAII优于手动管理)
- "先插入容器再初始化"的模式是否有Init失败的回滚逻辑?

关联commit: `d8400fd2`, `bc8c5036`

---

### 类别七: 并发/线程安全缺陷(11条, 6.8%)

通信框架天然是多线程环境(多device、多通信域、主线程+后台轮询线程+异步接收线程)，是最容易导致coredump/hang的缺陷类别。hotspot_analysis显示18个热点文件中15个存在并发安全问题。

#### 规则 CONC-01: 全局/静态变量无锁访问

严重等级: P0

缺陷描述: static全局变量在多线程/多流场景下被并发读写。4条案例。

典型代码示例:

```cpp
// 缺陷 — static全局数组在MC2多流场景下被多线程并发写
static uint8_t g_expectPrepareId[MAX_QUE_NUM];  // 多线程写同一位置
// 线程A: g_expectPrepareId[queueId] = idA;
// 线程B: g_expectPrepareId[queueId] = idB;  // 覆盖线程A的写入 -> 消息ID不匹配 -> hang

// 修复 — 改为thread_local
static thread_local uint8_t g_expectPrepareId[MAX_QUE_NUM];
```

审查检查方法:
- static全局变量在多线程场景下是否需要thread_local或显式同步?
- "循环体外持写锁、循环内做IO/RPC"的模式是否存在持锁粒度过粗风险?

关联commit: `341e7893`

---

#### 规则 CONC-02: 并发析构/UAF

严重等级: P0

缺陷描述: 多线程并发调用对象析构，裸指针delete缺乏互斥保护。3条案例。

典型代码示例:

```cpp
// 缺陷 — DispatcherCtx::Destroy()中delete无锁保护
void DispatcherCtx::Destroy() {
    delete dispatcher_;    // 多线程并发调用 -> double free -> coredump
    dispatcher_ = nullptr;
}

// 修复 — 新增destroyMutex_保护
void DispatcherCtx::Destroy() {
    std::lock_guard<std::mutex> lock(destroyMutex_);
    if (dispatcher_ != nullptr) {
        delete dispatcher_;
        dispatcher_ = nullptr;
    }
}
```

```cpp
// 缺陷 — reqHandle被异步线程引用时直接free
free(reqHandle);      // 异步线程可能正在引用 -> UAF
reqHandle = NULL;

// 修复 — 在reqMutex锁保护下通过统一删除函数处理
std::lock_guard<std::mutex> lock(reqMutex);
HdcAsyncDelResponse(reqHandle);  // 统一删除 + 关联资源清理
```

审查检查方法:
- 对象析构路径中涉及裸指针delete时，是否存在多线程并发调用的可能?
- 有配套mutex的数据结构，其free/delete是否在锁保护下进行?

关联commit: `7be3b8b8`, `cd056a5e`

---

#### 规则 CONC-03: 内存屏障/指令重排序

严重等级: P0

缺陷描述: 生产者-消费者模式中缺少store-store屏障，编译器/CPU重排序导致消费者读到脏数据。2条案例。

典型代码示例:

```cpp
// 缺陷 — 先memcpy数据再更新尾计数器，可能被重排序
void HDCommunicate::Write() {
    memcpy_s(sharedMem + offset, size, data, dataLen);  // store: 写数据
    *tailCntAddr_ = newTailCnt;  // store: 更新计数器
    // 两个store可能被CPU/编译器重排序 -> 消费者读到未写完的数据
}

// 修复 — 插入全序内存屏障
void HDCommunicate::Write() {
    memcpy_s(sharedMem + offset, size, data, dataLen);
    std::atomic_thread_fence(std::memory_order_seq_cst);  // 插入屏障
    *tailCntAddr_ = newTailCnt;
}
```

审查检查方法:
- "先写数据、再写标志位/计数器"的模式是否在两步之间插入了内存屏障?
- 共享内存通信中的生产者-消费者模式是否有acquire-release语义保证?

关联commit: `a014e919`

---

#### 规则 CONC-04: TOCTOU竞态

严重等级: P1

缺陷描述: 持锁检查后释放锁再操作，检查结果在释放锁后失效。2条案例。

典型代码示例:

```cpp
// 缺陷 — 持锁检查group不存在后解锁再创建，两线程可同时通过检查
{
    std::lock_guard<std::mutex> lock(groupMutex_);
    if (groupMap_.find(groupName) != groupMap_.end()) {
        return HCCL_E_EXIST;
    }
}  // 解锁
// 窗口期: 另一个线程也通过了检查
CHK_RET(CreateGroup(groupName));  // 两个线程都创建同名group
{
    std::lock_guard<std::mutex> lock(groupMutex_);
    groupMap_[groupName] = group;  // 覆盖
}

// 修复 — 检查和创建在同一临界区内
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

### 类别八: 日志/可观测性缺陷(11条, 6.8%)

非功能性缺陷中占比较大。日志格式串参数错误(如%s传int)可导致coredump。

#### 规则 LOG-01: 格式串参数类型不匹配

严重等级: P0

缺陷描述: printf/HCCL_ERROR的格式说明符与参数类型不匹配。%s传int导致将整数当指针解引用，未定义行为可能段错误。4条案例。

典型代码示例:

```cpp
// 缺陷 — %s期望字符串但传入int32_t类型的ret
HCCL_ERROR("[%s] operation failed, ret[%s]", __func__, ret);
//                                     ^^ %s将ret当指针解引用 -> UB/段错误

// 修复 — 使用正确的格式说明符
HCCL_ERROR("[%s] operation failed, ret[%d]", __func__, ret);
//                                     ^^ %d匹配int32_t
```

```cpp
// 缺陷 — %u格式打印指针值
HCCL_INFO("netLayer%u]", layerPtr);  // 缺少左方括号，%u打印指针

// 修复
HCCL_INFO("[netLayer%p]", layerPtr);
```

审查检查方法:
- printf/HCCL_ERROR的格式说明符与参数类型是否匹配? 启用-Wformat检查
- 日志中方括号是否配对?

关联commit: `5e446705`, `f715f167`

---

#### 规则 LOG-02: 日志级别不当

严重等级: P3

缺陷描述: 可恢复场景使用ERROR级别，误导运维排查。CHK_RET宏隐式打印ERROR。3条案例。

典型代码示例:

```cpp
// 缺陷 — AIV core不足是可恢复场景(回退到非AIV路径)，CHK_RET隐式打ERROR
CHK_RET(CalcAivBlockDim(coreNum));  // 失败时CHK_RET打印ERROR日志

// 修复 — 使用CHK_PRT_RET显式控制日志级别
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

#### 规则 LOG-03: 日志洪泛/变量引用错误

严重等级: P3

缺陷描述: 高频路径无日志抑制导致刷屏; 日志中引用已被修改的累积值而非原始值。4条案例。

典型代码示例:

```cpp
// 缺陷 — 高频opcode每次调用都打印debug日志
HcclResult RaGetOpRight(u32 opcode) {
    HCCL_DEBUG("get op right for opcode[%u]", opcode);  // 高频刷屏
    ...
}

// 修复 — 引入日志抑制机制
HcclResult RaGetOpRight(u32 opcode) {
    if (!RaIsOpcodeLogSuppressed(opcode)) {
        HCCL_DEBUG("get op right for opcode[%u]", opcode);
    }
    ...
}
```

```cpp
// 缺陷 — 日志打印使用了已被累加额外预留空间的scratchBufSize
scratchBufSize += extraReserve;
HCCL_INFO("cclBufferSize[%llu]", scratchBufSize);  // 打印的是累加后的值

// 修复 — 在累加前记录原始值
HCCL_INFO("cclBufferSize[%llu]", cclBufferSize);   // 使用原始变量名
scratchBufSize += extraReserve;
```

审查检查方法:
- 日志中引用的变量是否是当前值而非已被修改的累积值?
- 高频调用路径上的日志是否有抑制/采样机制?

关联commit: `4425a342`, `802c0411`

---

### 类别九: 状态/缓存/超时管理缺陷(10条, 6.2%)

通信框架中大量使用cache复用、重试超时、环境变量配置等机制。状态未刷新和超时值不一致是常见根因。

#### 规则 STATE-01: 缓存复用路径状态未刷新

严重等级: P0

缺陷描述: cache命中时结构体字段被部分更新，运行时上下文字段(如stream)遗漏。3条案例。

典型代码示例:

```cpp
// 缺陷 — ExecOpCache中stream字段未更新为当前操作的stream
if (cacheHit) {
    // cacheInfo.resourceArgs的stream还是上次缓存时的值
    // AIV kernel在旧stream上执行 -> 执行序错乱
    LaunchKernel(cacheInfo.resourceArgs);
}

// 修复 — cache命中后刷新运行时上下文字段
if (cacheHit) {
    cacheInfo.resourceArgs.stream = currentStream;  // 刷新stream
    LaunchKernel(cacheInfo.resourceArgs);
}
```

```cpp
// 缺陷 — 缓存key缺少workflowMode维度
// 图模式和单算子模式走不同算法路径，但缓存键不区分
OpUnfoldKey key(opType, dataType, count);  // 缺少workflowMode

// 修复 — 扩展缓存键
OpUnfoldKey key(opType, dataType, count, workflowMode);
```

审查检查方法:
- cache/复用路径中被部分更新的结构体，所有运行时上下文字段是否都已刷新?
- 缓存键是否包含所有影响执行路径的区分因子?

关联commit: `3c24c0fe`, `b3975b85`

---

#### 规则 STATE-02: 超时值硬编码/配置不一致

严重等级: P1

缺陷描述: 多层超时值未统一联动，内外层超时语义不一致。4条案例。

典型代码示例:

```cpp
// 缺陷 — OpRetry超时硬编码205秒，用户配置更大值时先超时误判
constexpr u32 OP_RETRY_SEND_RECV_TIMEOUT = 205;
future.wait_for(std::chrono::seconds(OP_RETRY_SEND_RECV_TIMEOUT));
// 用户HCCL_LINK_TIMEOUT=300时，OpRetry在205秒先超时

// 修复 — 取max(用户配置, 默认值) + 裕量
u32 timeout = std::max(GetExternalInputHcclLinkTimeOut(),
                       OP_RETRY_SEND_RECV_TIMEOUT) + OP_RETRY_WAIT_AICPU_TIMEOUT;
```

```cpp
// 缺陷 — execTimeOut=0语义为"不超时"，代码按字面值处理
u64 timeoutUs = execTimeOut * 1000000;  // 0 * 1000000 = 0微秒 -> 立即超时

// 修复 — 显式处理零值特殊语义
u64 timeoutUs = (execTimeOut == 0) ? UINT64_MAX : execTimeOut * 1000000;
```

审查检查方法:
- 超时参数是否从外层逐层传递到底层阻塞调用? std::async+future::wait_for是否有内层取消机制?
- 超时值0是否有"不限制"的特殊语义? 转换函数中是否显式处理了零值?
- 不同组件的超时值是否与用户可配置值联动(取max而非硬编码)?

关联commit: `b8de68b9`, `1479e7ba`, `f058f2d9`

---

#### 规则 STATE-03: 环境变量/运行时状态一致性

严重等级: P2

缺陷描述: 每次调用都getenv而非缓存初始化时读取，运行时修改环境变量导致行为不一致。3条案例。

典型代码示例:

```cpp
// 缺陷 — 每次调用都读环境变量，运行时修改导致行为不一致
bool IsIndependentOp() {
    const char *env = getenv("HCCL_INDEPENDENT_OP");
    return (env != nullptr && strcmp(env, "1") == 0);
}
// 在同一进程生命周期内，返回值可能前后不一致

// 修复 — 初始化时一次性读取并缓存
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

### 类别十: 空指针/入参校验缺失(8条, 4.9%)

集中在framework层的公共API入口和内部组件的指针使用。

#### 规则 PTR-01: 空指针检查顺序错误

严重等级: P0

缺陷描述: 先解引用再检查null，或日志中%s传可能为nullptr的指针(UB)。3条案例。

典型代码示例:

```cpp
// 缺陷 — 先解引用再检查null
auto val = ptr->GetValue();
if (ptr == nullptr) {
    return HCCL_E_PTR;
}

// 修复 — 检查前置于解引用
if (ptr == nullptr) {
    return HCCL_E_PTR;
}
auto val = ptr->GetValue();
```

```cpp
// 缺陷 — nullptr传给%s是UB
void DestroyDispatcherCtx(const char *commId) {
    HCCL_INFO("[DestroyCtx] commId[%s]", commId);  // commId可能为nullptr
    if (commId == nullptr) { return; }

// 修复 — 空指针检查前置
void DestroyDispatcherCtx(const char *commId) {
    if (commId == nullptr) { return; }
    HCCL_INFO("[DestroyCtx] commId[%s]", commId);
```

审查检查方法:
- 空指针检查是否在解引用之前(不是之后)?
- %s格式化的参数是否保证非空?

关联commit: `dd744786`, `384d78c0`

---

#### 规则 PTR-02: API入口参数校验缺失

严重等级: P1

缺陷描述: 公共API中reinterpret_cast后未检查null，用户传入无效句柄则crash。3条案例。

典型代码示例:

```cpp
// 缺陷 — HcclGetHcclBuffer检查了buffer但漏检comm
HcclResult HcclGetHcclBuffer(HcclComm comm, void **buffer) {
    CHK_PRT_RET(buffer == nullptr,
        HCCL_ERROR("buffer is nullptr"), HCCL_E_PTR);
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
    // comm为nullptr时hcclComm也是nullptr -> 下一行crash
    std::string identifier = hcclComm->GetIdentifier();

// 修复 — 补充comm指针校验
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

#### 规则 PTR-03: 可选指针/新增参数校验缺失

严重等级: P1

缺陷描述: 可能为空的可选指针未做defensive check; 新增参数的操作缺少不支持的组合校验。2条案例。

典型代码示例:

```cpp
// 缺陷 — commMem指针可能为空但直接使用
void *buffer = commMem->GetBuffer();  // commMem可能为空

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

### 类别十一: 接口/API设计与适配缺陷(8条, 4.9%)

通信框架经历V1->V2迁移、ACL->RT接口替换、open/closed双模式等架构演进，接口适配层面的设计缺陷频繁暴露。

#### 规则 API-01: 抢跑依赖未发布API

严重等级: P2

缺陷描述: 在依赖的SDK正式发布新接口前就在消费端使用，链接失败。3条案例。12天内对同一组文件进行5次方向性变更(乒乓Revert)。

典型代码示例:

```cpp
// 缺陷 — rtModelGetId和rtRegTaskFailCallbackByModule在runtime SDK中尚未发布
uint64_t modelId;
rtModelGetId(rtModel, &modelId);  // 链接失败: undefined symbol

// 修复 — 回退到已发布API或引入dlopen动态探测
uint64_t modelId = reinterpret_cast<uint64_t>(rtModel);  // 回退到hack写法

// 或: dlopen双路径fallback
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

#### 规则 API-02: 返回值语义差异未适配

严重等级: P1

缺陷描述: 底层接口替换后返回值语义变化(bitmask vs enum)，调用方解析逻辑未同步适配。3条案例。

典型代码示例:

```cpp
// 缺陷 — ACL返回bitmask，RT返回enum，切换后解析逻辑未改
// ACL: aclrtGetDevicesTopo返回link type bitmask (可组合多bit)
// RT:  rtGetPairPhyDevicesInfo返回单个enum值
u32 linkType = rtGetPairPhyDevicesInfo(devA, devB);
if (linkType & LINK_TYPE_HCCS) {  // 按bitmask解析enum -> 逻辑错误
    ...
}

// 修复 — 适配enum语义
u32 linkType = rtGetPairPhyDevicesInfo(devA, devB);
if (linkType == LINK_TYPE_HCCS) {  // 按enum单值比较
    ...
}
```

审查检查方法:
- 返回值类型差异(bitmask vs enum, signed vs unsigned)在迁移时调用方解析逻辑是否同步适配?
- API返回裸指针时，所有权(谁分配谁释放)是否在注释中明确文档化?

关联commit: `1d8e2c14`

---

#### 规则 API-03: 封装层能力不完整

严重等级: P2

缺陷描述: 适配层未覆盖底层完整能力，或V2 weak symbol函数声明与实际签名不匹配。2条案例。

审查检查方法:
- 封装层新增接口后，是否覆盖了底层完整能力?
- V2 weak symbol函数的签名是否与实际函数完全一致?

关联commit: `47020bd5`, `26f5813f`

---

### 类别十二: 硬件/平台适配缺陷(6条, 3.7%)

通信库需要适配多种芯片(910B/910_93/910_95/A5等)和多种连接协议(PCIE/RoCE/HCCS/URMA)，硬件协议约束和设备差异性是低可审查性缺陷的主要来源。

#### 规则 HW-01: 硬件协议乘数因子遗漏

严重等级: P0

缺陷描述: URMA约束每个SQE含4个WQEBB，多处代码计算SQ buffer大小和VA映射长度时遗漏WQEBB_NUM_PER_SQE(=4)乘数因子，device侧VA空间只有实际需要的1/4，内存越界。3条案例。

典型代码示例:

```cpp
// 缺陷 — sqDepth未乘WQEBB_NUM_PER_SQE
UbConnLite::UbConnLite(..., u32 sqDepth, ...) {
    sqDepth_ = sqDepth;  // 遗漏乘数因子
}
// VA映射长度也遗漏
u64 sqBufferSize = sqDepth_ * WQE_BB_SIZE;  // 缺 * WQEBB_NUM_PER_SQE
// VA空间只有实际需要的1/4 -> 内存越界

// 修复 — 统一乘WQEBB_NUM_PER_SQE
UbConnLite::UbConnLite(..., u32 sqDepth, ...) {
    sqDepth_ = sqDepth * WQEBB_NUM_PER_SQE;  // 每个SQE含4个WQEBB
}
u64 sqBufferSize = sqDepth_ * WQE_BB_SIZE;  // sqDepth_已包含乘数
```

```cpp
// 缺陷 — 宏URMA_JFS_DB_STATUS值与URMA_JFS_PI_TYPE重复
#define URMA_JFS_PI_TYPE     0x0011
#define URMA_JFS_DB_STATUS   0x0011  // 错误: 与PI_TYPE重复
// 读DB_STATUS实际读到PI_TYPE寄存器

// 修复
#define URMA_JFS_DB_STATUS   0x000f  // 正确值
```

审查检查方法:
- 涉及硬件协议约束的乘数因子是否在协议层统一封装为宏/函数? 避免多处手动乘算
- 同一组宏常量定义中是否存在重复值? (static_assert或编译期检查)
- SQ深度等协议参数的"单位"(SQE数/WQEBB数/字节)是否在类型或命名中体现?

关联commit: `df96667c`, `39ad6c07`

---

#### 规则 HW-02: 设备类型特殊路径未处理

严重等级: P1

缺陷描述: 新增设备类型的初始化/资源管理路径与已有设备不同。3条案例。

典型代码示例:

```cpp
// 缺陷 — 910_95设备notify初始化不需要SetIpc()
CHK_RET(notify->SetIpc());  // 910_95上SetIpc()失败 -> notify申请失败

// 修复 — 按设备类型分支处理
if (deviceType_ != DEV_TYPE_910_95) {
    CHK_RET(notify->SetIpc());
}
```

```cpp
// 缺陷 — notify偏移量使用简单线性计算，未适配分slice地址空间
u64 offset = notifyId * notifySize;  // 910B/910_93上notifyId>=512时错误

// 修复 — 按设备类型区分计算方式
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

#### 规则 HW-03: 硬件寄存器/地址映射宏定义冲突

严重等级: P0

缺陷描述: 宏常量值重复导致寄存器访问错误。

审查检查方法:
- 同一组寄存器偏移宏是否通过static_assert确保无重复值?
- 新增寄存器映射宏时是否检查了已有定义?

关联commit: `39ad6c07`

---

### 跨类别系统性风险

以下模式不属于单一缺陷类别，而是贯穿多个类别的系统性问题，来自hotspot_analysis和revert_analysis的交叉分析。

#### SYS-01: God Class与代码重复导致"改一漏一"

hccl_communicator_host.cc(8818行, 14次缺陷修复)、hcom.cc(4068行, 14次)、aicpu_communicator.cc(5809行, 12次)、op_base.cc(4587行, 11次)均为超大文件。hcclComm类承载约80个public方法。

代码重复的典型形式:
- op_base.cc中InitCommRootInfo(212行)/InitCommClusterInfo(136行)/HcclCreateSubCommConfigInner三处InitCommConfig序列几乎完全重复
- hccl_communicator.cc中Suspend/StopExec/Clean三个函数约190行包含几乎完全相同的while(true)轮询循环
- dispatcher_aicpu.cc中LaunchTask和WaitRtsq的RTSQ等待/超时/拷贝逻辑大量重复

审查规则: "改一漏一"是hcomm-dev缺陷的高频来源。修改超大文件中的某一处逻辑时，必须搜索该文件内和跨文件的所有同模式代码，确保同步修改。

---

#### SYS-02: 并发安全缺乏系统性设计

18个热点文件中15个存在并发安全问题。模式包括:
- TOCTOU竞态: hcom.cc(HcomCreateGroupImplHeterog), hcom_common.cc(HcomGetCurHcomCtx)
- 无锁全局变量: op_base.cc(g_oneSidedCommHcomInfos), aicpu_communicator.cc(errMessageReport_), adapter_rts.cc(g_localDeviceType), hcom.cc(g_rankTableSetInfo)
- 锁范围不一致: op_base.cc中HcclSendInner/HcclRecvInner等6个函数缺少operatorlock_

根因: 锁的使用缺乏统一的层级设计和文档化约定。

审查规则: 对static/全局变量的新增或修改，必须标注线程安全声明(thread_local/互斥锁/仅单线程访问)。

---

#### SYS-03: 公共API/ABI设计债务

严重缺陷:
- hcom.h: extern "C"块内使用namespace、公共函数签名含C++默认参数、头文件中定义non-inline std::string/std::map全局变量
- hccl_comm_pub.h: 条件编译(CCL_KERNEL_AICPU/HCCD)改变类内存布局，跨模块传指针越界
- 多个API返回悬垂指针: hcom.cc:2076 `*algo = const_cast<char*>(str.c_str())`将局部变量内部指针返回(确定性UAF)

审查规则: 公共头文件(pkg_inc/)的任何修改必须通过ABI兼容性checklist审查。

---

#### SYS-04: 大爆炸提交(Big Bang Commit)掩盖关键变更

Revert分析揭示的最强信号:
- Revert#1原始提交: "update"(678文件)中隐藏心跳帧结构膨胀30倍
- Revert#4: "ars code"(51文件, 2000+行, 跨4层)仅两个单词commit message
- 功能变更混入大批量同步提交，审查者无法在海量无关变更中发现关键问题

审查规则: 超过20个文件或500行变更的PR必须拆分。功能变更不得混入同步/批量提交中。

---

#### SYS-05: 确定性Bug仍存在于当前代码

hotspot_analysis中发现的确定性bug(截至分析时点):
- aicpu_communicator.cc:3385-3386 UpdateSqStatus中SQ_TAIL查询结果写入head、SQ_HEAD写入tail，head/tail赋值反转
- hcom.cc:2076 `*algo = const_cast<char*>(str.c_str())`返回局部变量内部指针，函数返回后即悬垂(确定性UAF)
- adapter_rts.cc:37-45 REPLACE_NOTIFY_WITH_EVENT宏中result硬编码为0，if(result!=0)永远不执行，event替换功能完全失效

审查规则: 热点文件的修改PR应要求整文件快速扫描已知风险模式。

---

### 附录: 审查规则速查表

| 规则ID | 严重等级 | 规则名称 | 类别 |
|--------|---------|---------|------|
| BUILD-01 | P2 | CMakeLists源文件遗漏 | 构建 |
| BUILD-02 | P2 | 条件编译/编译目标不兼容 | 构建 |
| BUILD-03 | P1 | ABI兼容性/符号导出 | 构建 |
| BUILD-04 | P1 | 环境/配置适配遗漏 | 构建 |
| INIT-01 | P1 | 多分支初始化路径不对称 | 初始化 |
| INIT-02 | P0 | 成员变量未初始化 | 初始化 |
| INIT-03 | P0 | 执行顺序/依赖错误 | 初始化 |
| INIT-04 | P1 | offload/lambda变量作用域泄漏 | 初始化 |
| BRANCH-01 | P1 | 枚举/类型分支遗漏 | 分支 |
| BRANCH-02 | P1 | V2/offload适配路径缺失 | 分支 |
| BRANCH-03 | P0 | 条件方向错误/条件编译覆盖运行时分支 | 分支 |
| TYPE-01 | P0 | 枚举值语义误用 | 类型 |
| TYPE-02 | P1 | 类型隐式截断/溢出 | 类型 |
| TYPE-03 | P0 | 参数对象混淆/引用错误 | 类型 |
| TYPE-04 | P0 | map::at()对未收录key抛异常 | 类型 |
| QUAL-01 | P1 | 算法名称/变量名拼写错误 | 质量 |
| QUAL-02 | P2 | 全局重命名同步不完整 | 质量 |
| QUAL-03 | P3 | 全代码库拼写错误渗入API | 质量 |
| RES-01 | P0 | 资源释放顺序错误 | 资源 |
| RES-02 | P1 | 资源重复管理/泄漏 | 资源 |
| RES-03 | P1 | 错误路径资源泄漏 | 资源 |
| CONC-01 | P0 | 全局/静态变量无锁访问 | 并发 |
| CONC-02 | P0 | 并发析构/UAF | 并发 |
| CONC-03 | P0 | 内存屏障/指令重排序 | 并发 |
| CONC-04 | P1 | TOCTOU竞态 | 并发 |
| LOG-01 | P0 | 格式串参数类型不匹配 | 日志 |
| LOG-02 | P3 | 日志级别不当 | 日志 |
| LOG-03 | P3 | 日志洪泛/变量引用错误 | 日志 |
| STATE-01 | P0 | 缓存复用路径状态未刷新 | 状态 |
| STATE-02 | P1 | 超时值硬编码/配置不一致 | 状态 |
| STATE-03 | P2 | 环境变量/运行时状态一致性 | 状态 |
| PTR-01 | P0 | 空指针检查顺序错误 | 空指针 |
| PTR-02 | P1 | API入口参数校验缺失 | 空指针 |
| PTR-03 | P1 | 可选指针/新增参数校验缺失 | 空指针 |
| API-01 | P2 | 抢跑依赖未发布API | 接口 |
| API-02 | P1 | 返回值语义差异未适配 | 接口 |
| API-03 | P2 | 封装层能力不完整 | 接口 |
| HW-01 | P0 | 硬件协议乘数因子遗漏 | 硬件 |
| HW-02 | P1 | 设备类型特殊路径未处理 | 硬件 |
| HW-03 | P0 | 硬件寄存器/地址映射宏定义冲突 | 硬件 |
| SYS-01 | - | God Class与代码重复 | 系统性 |
| SYS-02 | - | 并发安全缺乏系统性设计 | 系统性 |
| SYS-03 | - | 公共API/ABI设计债务 | 系统性 |
| SYS-04 | - | 大爆炸提交掩盖关键变更 | 系统性 |
| SYS-05 | - | 确定性Bug仍存在于当前代码 | 系统性 |
