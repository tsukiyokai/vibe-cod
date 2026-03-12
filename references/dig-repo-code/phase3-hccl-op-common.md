# Phase 3.1: hccl op_common 深度分析

## 3.1.1 executor/ — 执行器设计，任务编排

### Module Overview

executor/ 是 hccl 算子执行的核心框架模块，位于 `src/ops/op_common/executor/`，
共 12 个文件(4 对 .h/.cc)，3 个子目录(根目录 + channel/ + registry/)。
总规模约 690 行(不含 channel 的 611 行)。

职责: 定义算子执行器的基类体系和注册工厂，为所有集合通信算子提供统一的
CalcRes → Orchestrate → KernelRun 三阶段执行框架。

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| executor_base.h | 69 | V1 执行器基类 ExecutorBase + ExecMem 结构体 |
| executor_base.cc | 77 | V1 基类实现(CalcResRequest/KernelRun/GetSubCommInfo/RefreshAlgType) |
| executor_v2_base.h | 69 | V2 执行器基类 InsCollAlgBase(完全独立，不继承 V1) |
| executor_v2_base.cc | 45 | V2 基类实现(Describe/RestoreChannelMap) |
| channel/channel.h | 51 | 通道计算函数声明(11 个自由函数) |
| channel/channel.cc | 487 | 通道计算实现(三层拓扑 + 5 种拓扑算法) |
| channel/channel_request.h | 28 | 连接计算函数声明(5 个自由函数) |
| channel/channel_request.cc | 124 | Ring/Mesh/NHR/NB/Mesh2D 连接计算 |
| registry/coll_alg_exec_registry.h | 51 | V1 Registry: tag → unique_ptr\<ExecutorBase\> |
| registry/coll_alg_exec_registry.cc | 43 | V1 Registry 实现 |
| registry/coll_alg_v2_exec_registry.h | 106 | V2 Registry: (HcclCMDType, tag) → shared_ptr\<InsCollAlgBase\> |
| registry/coll_alg_v2_exec_registry.cc | 44 | V2 Registry 实现 |

### 架构: V1/V2 双代执行器体系

#### V1: ExecutorBase (遗留，仅 scatter 使用)

```
ExecutorBase                       // executor_base.h:44
├── virtual ~ExecutorBase() = default
├── virtual CalcResRequest(...)    // 默认返回 SUCCESS(可选 override)
├── virtual Orchestrate(...) = 0   // 纯虚(必须实现)
├── virtual KernelRun(...)         // 默认 WARNING + SUCCESS
├── GetSubCommInfo(...)            // protected 工具方法
├── RefreshAlgType(...)            // protected 算法类型刷新
└── GetAlgDesc()                   // inline 非虚

protected 成员:
  tag_, workflowMode_, topoInfo_*, algResource_*, algType_, level0RankSize, desc_
```

V1 特点:
- 构造函数调用 GetWorkflowMode() 初始化 workflowMode_
- CalcResRequest 参数全部 `(void)param` 抑制(默认空实现)
- KernelRun 默认实现只打 WARNING 日志
- RefreshAlgType: 对 Level0/1/2 各检查 desc_.supportedAlgos，不支持的重置为 [0]
  (相同逻辑重复 3 次，仅 level 索引不同)
- 全仓仅 4 个 V1 注册(全在 scatter/algo/):
  ScatterCommExecutor, ScatterMeshExecutor, ScatterSingleExecutor, ScatterRingFor91093Executor

#### V2: InsCollAlgBase (主力，28 个子类)

```
InsCollAlgBase                     // executor_v2_base.h:28
├── virtual ~InsCollAlgBase()
├── virtual Describe() const       // 返回算法描述字符串
├── virtual CalcAlgHierarchyInfo(...) = 0  // 纯虚: 计算层级信息
├── virtual CalcRes(...) = 0       // 纯虚: 计算资源需求
├── virtual Orchestrate(...) = 0   // 纯虚: 编排算法执行
└── RestoreChannelMap(...)         // public 非虚: rank→channel 映射恢复

protected 成员:
  myRank_, rankSize_, devType_,
  reduceOp_, root_,
  dataType_, dataCount_, dataSize_, dataTypeSize_,
  maxTmpMemSize_,
  topoInfo_*(TopoInfoWithNetLayerDetails),
  algResource_&(AlgResourceCtxSerializable)
```

V2 特点:
- 完全独立类体系(不继承 ExecutorBase)
- 3 个纯虚方法(vs V1 的 1 个): 强制子类实现资源计算 + 层级信息 + 编排
- 拓扑信息升级: TopoInfoWithNetLayerDetails(含网络层细节) 替代 TopoInfo
- 资源上下文升级: AlgResourceCtxSerializable&(可序列化引用) 替代 AlgResourceCtx*
- 成员更完整: 包含 reduceOp/root/dataType 等集合通信语义字段
- 全仓 60 个 V2 注册，覆盖 all_reduce/all_gather/reduce_scatter/reduce/broadcast 等

#### V1 vs V2 对比

| 维度 | V1 (ExecutorBase) | V2 (InsCollAlgBase) |
|------|-------------------|---------------------|
| 使用量 | 4 个注册(仅 scatter) | 60 个注册(14 个文件) |
| 纯虚方法 | 1 个(Orchestrate) | 3 个(CalcAlgHierarchyInfo + CalcRes + Orchestrate) |
| 拓扑信息 | TopoInfo* | TopoInfoWithNetLayerDetails* |
| 资源上下文 | AlgResourceCtx*(裸指针) | AlgResourceCtxSerializable&(可序列化引用) |
| 返回容器 | unique_ptr | shared_ptr |
| 索引维度 | 单维(tag 字符串) | 双维(HcclCMDType + tag) |
| 设计风格 | 宽松(默认空实现) | 严格(纯虚强制实现) |
| 地位 | 遗留，仅 scatter 未迁移 | 主力，新算子全部使用 |

### ExecMem 结构体

```cpp
// executor_base.h:35-42
struct ExecMem {
    u64 count = 0;
    HcclMem inputMem;     // 单算子模式 = InCCLMem, 图模式 = InUserMem
    HcclMem outputMem;    // 单算子模式 = OutCCLMem, 图模式 = OutUserMem
    HcclMem scratchMem;
    void *inputPtr = nullptr;   // InUserMem 地址(图模式与 inputMem 同址)
    void *outputPtr = nullptr;  // OutUserMem 地址(图模式与 outputMem 同址)
};
```

区分单算子模式(CCL buffer)和图模式(User buffer)，通过注释说明语义。

### 硬件限制常量

```cpp
// executor_base.h:27-33
const u64 RDMA_SEND_MAX_SIZE = 0x80000000;   // 2GB, 节点间 RDMA 单个 WQE 最大数据量
const u64 SDMA_SEND_MAX_SIZE = 0x100000000;  // 4GB, 节点内单个 SDMA 任务最大数据量
constexpr u32 HCCL_INTERNODE_MAX_DATA_RATE = 1;
constexpr s32 LEVEL0_PLANE_NUM_IN_NPRING_SINGLE = 1;
constexpr s32 LEVEL0_PLANE_NUM_IN_NPRING_DOUBLE = 2;
```

与 hcomm platform 层的 4GB SDMA 切分惯用法一致(SDMA_SEND_MAX_SIZE = 0x100000000)。

### channel/ — 通道计算(过程式)

channel/ 不定义类，纯自由函数，为执行器的 CalcResRequest 阶段提供通道计算。

#### 三层通道计算

```cpp
// channel.h — 三层各一个入口函数
CalcLevel0ChannelRequest(OpParam, TopoInfo*, AlgHierarchyInfo&, AlgType&, channels)
CalcLevel1ChannelRequest(...)
CalcLevel2ChannelRequest(...)
```

每层根据 AlgType 中对应 level 的算法类型(Ring/Mesh/NHR 等)选择连接拓扑。

#### 拓扑特定通道计算(V2 专用)

```cpp
// 5 种拓扑: Mesh1D / Mesh2D / NHR / Mesh1DWithPriorityTopo / NHRWithPriorityTopo
CalcChannelRequestMesh1D(HcclComm, OpParam, TopoInfoWithNetLayerDetails*, ...)
CalcChannelRequestMesh2D(...)
CalcChannelRequestNhr(...)
CalcChannelRequestMesh1DWithPriorityTopo(..., CommTopo priorityTopo)
CalcChannelRequestNHRWithPriorityTopo(..., CommTopo priorityTopo)
```

Priority 变体: 多链路场景中优先选择特定拓扑类型的链路。

#### 连接拓扑算法(channel_request.cc)

```cpp
CalcRingChannelConnect(localRank, rankSize, root, connectRanks)   // r±1
CalcMeshChannelConnect(...)                                        // 全连接
CalcNHRChannelConnect(...)                                         // 2^k 距离
CalcNBChannelConnect(...)                                          // 同 NHR
CalcMesh2DChannelConnect(myRank, subcommInfo, connectRanks)       // 行+列环
```

所有算法输出 set<u32> connectRanks(set 去重，Ring 奇数 rank 时前后邻居可能相同)。

#### 条件编译

channel.cc 中 8 处 `#ifndef AICPU_COMPILE` 保护:
Mesh1D/Mesh2D/Nhr/GetTopoTypeByLink/ProcessLinksForChannel/Mesh1DWithPriority/NHRWithPriority/CreateChannelRequestByRankId
— AICPU 编译时这些函数直接返回 HCCL_SUCCESS(空操作)。

#### 协议选择逻辑

```cpp
// channel.cc 协议选择(简化)
// Level0: HCCS(节点内)
// Level1: 910B 多机 || 910_93 disableHccs → ROCE, 否则 HCCS
// Level2: ROCE(始终)
```

设备类型 + 层级 + 多机模式共同决定通信协议。

#### HcclChannelDesc 核心结构

```cpp
struct HcclChannelDesc {
    u32 remoteRank;
    EndpointDesc localEndpoint;
    EndpointDesc remoteEndpoint;
    CommProtocol channelProtocol;   // HCCS / ROCE / UBC_CTP / UB_MEM
    u32 notifyNum;                  // 通常 3 (NORMAL_NOTIFY_NUM)
};
```

#### Rank 映射惯用法

```cpp
// 全局 rank → 本地 rank (channel.cc 多处)
auto it = std::find(subcommInfo[0].begin(), subcommInfo[0].end(), myRank);
u32 localRank = std::distance(subcommInfo[0].begin(), it);
```

#### 多层拓扑查询(Early Exit)

```cpp
// channel.cc — 逐层查询，找到即停
for (auto netLayer : netLayersVector) {
    CHK_RET(HcclRankGraphGetLinks(comm, netLayer, myRank, rank, ...));
    if (listSize > 0) { break; }   // 在此层找到链路
}
```

### registry/ — 双 Registry 自注册工厂

#### V1 Registry

```cpp
class CollAlgExecRegistry {
    static CollAlgExecRegistry &Instance();                    // Meyer's singleton
    HcclResult Register(const string &tag, const CollExecCreator &creator);
    unique_ptr<ExecutorBase> GetAlgExec(const string &tag);
private:
    unordered_map<string, const CollExecCreator> execCreators_;
    mutable mutex mu_;
};

// 注册宏(1 种)
REGISTER_EXEC(tag, name, collExecBase)
// 展开: static HcclResult g_func_<name>_<__COUNTER__> =
//   CollAlgExecRegistry::Instance().Register(tag, DefaultExecCreator<collExecBase>);
```

#### V2 Registry

```cpp
class CollAlgExecRegistryV2 {
    static CollAlgExecRegistryV2 &Instance();                  // Meyer's singleton
    HcclResult Register(HcclCMDType type, const string &tag, const CollExecCreatorV2 &creator);
    shared_ptr<InsCollAlgBase> GetAlgExec(HcclCMDType type, const string &tag);
private:
    map<HcclCMDType, map<string, const CollExecCreatorV2>> execCreators_;
    mutable mutex mu_;
};

// 注册宏(7 种变体，支持 0/1/2/4 个模板参数)
REGISTER_EXECUTOR_IMPL(type, name, insCollAlgBase)                                          // 无模板
REGISTER_EXECUTOR_IMPL_NO_TOPOMATCH(type, name, insCollAlgBase, InsAlgTemplate)             // 1 模板
REGISTER_EXEC_V2(type, name, insCollAlgBase, AlgTopoMatch, InsAlgTemplate)                  // 2 模板
REGISTER_EXECUTOR_BY_TOPO(type, name, insCollAlgBase, AlgTopoMatch)                         // 仅拓扑
REGISTER_EXECUTOR_BY_TWO_TEMPS(type, name, ..., InsAlgTemplate0, InsAlgTemplate1)           // 2 算法模板
REGISTER_EXECUTOR_BY_FOUR_TEMPS(type, name, ..., InsAlgTemplate0..3)                        // 4 算法模板
```

所有宏使用三层嵌套(PUBLIC → _HELPER_1 → _HELPER)确保 `__COUNTER__` 正确展开。

#### 工厂函数

```cpp
// V1 + V2 相同模式
template <typename P>
static ExecutorBase *DefaultExecCreator() {
    static_assert(std::is_base_of<ExecutorBase, P>::value,
        "Executor type must derived from Hccl::ExecutorBase");
    return new (std::nothrow) P();
}
```

编译时类型检查(static_assert) + new(nothrow) 分配。

### Idioms & Conventions (惯用法)

#### EX-ID-1: Template Method 三阶段执行框架
V1: CalcResRequest(可选) → Orchestrate(必须) → KernelRun(可选)
V2: CalcAlgHierarchyInfo(必须) → CalcRes(必须) → Orchestrate(必须)
V2 更严格，3 个纯虚强制子类实现。
出处: executor_base.h:49-55, executor_v2_base.h:34-42

#### EX-ID-2: 双 Registry 自注册工厂
全局静态初始化期注册，运行时通过 tag 查找创建。
V1 单维(tag)，V2 双维(HcclCMDType + tag)。
量化: V1 4 个注册，V2 60 个注册。
出处: coll_alg_exec_registry.h:49, coll_alg_v2_exec_registry.h:51-104

#### EX-ID-3: (void)param 抑制未使用警告
CalcResRequest 默认实现中对所有参数执行 `(void)param`。
出处: executor_base.cc:20-28

#### EX-ID-4: new(nothrow) + static_assert 工厂
编译时基类检查 + 运行时不抛异常分配，与 hcomm 一致。
出处: coll_alg_exec_registry.h:25-30, coll_alg_v2_exec_registry.h:26-31

#### EX-ID-5: set<u32> 连接去重
连接计算函数输出 set(非 vector)，自动去重。
Ring 奇数 rank 时 (r+1)%n == (r-1+n)%n 可能相同。
出处: channel_request.h:20-26

#### EX-ID-6: std::find + std::distance rank 映射
全局 rank → 本地 rank 转换。
出处: channel.cc 多处

#### EX-ID-7: #ifndef AICPU_COMPILE 条件编译
Host 侧通道计算在 AICPU 编译时变为空操作。
出处: channel.cc 8 处

#### EX-ID-8: CHK_RET 100% 覆盖
executor/ 目录 CHK_RET 35 次/2 文件(channel.cc 34 次 + channel_request.cc 1 次)。
基类文件零 CHK_RET(因为逻辑简单，直接 return)。

#### EX-ID-9: 三层宏嵌套确保 __COUNTER__ 展开
所有注册宏都用 PUBLIC_MACRO → _HELPER_1 → _HELPER 三层。
原因: 如果直接在宏体内用 __COUNTER__，token 拼接会阻止展开。
出处: coll_alg_exec_registry.h:40-49, coll_alg_v2_exec_registry.h:43-104

### Architecture & Patterns (架构模式)

#### EX-AP-1: V1/V2 双代并存，V2 为主力
V1 仅 scatter 算子使用(4 个注册)，V2 覆盖所有其他算子(60 个注册)。
scatter 可能是最早实现的算子，尚未迁移到 V2。
与 hcomm 的 V1/V2 分派模式类似(HCCLV2_FUNC_RUN)。

#### EX-AP-2: 过程式通道计算(channel/)
channel/ 全部是自由函数(非类方法)，提供给执行器的资源计算阶段调用。
这与 hcomm algorithm 层的 CommBase + AlgTemplateBase 面向对象风格形成对比。
hccl 选择过程式可能是因为通道计算逻辑更简单直接。

#### EX-AP-3: 三层拓扑 + 五种算法的正交组合
Level0/1/2 三层，每层可选 Ring/Mesh/NHR/NB/Mesh2D。
与 hcomm algorithm 层的三级拓扑模型一致(COMM_LEVEL0/1/2)。

#### EX-AP-4: Registry V2 支持模板组合注入
7 种注册宏支持 0~4 个模板参数(AlgTopoMatch + InsAlgTemplate0..3)，
实现拓扑匹配策略 + 算法模板的编译时组合注入。
这是 hcomm REGISTER_TEMPLATE(仅 1 种)的增强版。

### Domain Knowledge (业务知识)

#### EX-DK-1: 三层通信层级
```
COMM_LEVEL0 (设备级) — HCCS 协议
COMM_LEVEL1 (主机级) — HCCS 或 ROCE(取决于设备类型和部署模式)
COMM_LEVEL2 (集群级) — ROCE 协议
```
出处: channel.h:19-24

#### EX-DK-2: 通道复用(channelRepeatNum)
CreateChannelRequestByRankId 支持同一 rank 对创建多个重复通道，
用于带宽聚合或容错。
出处: channel.cc:443+

#### EX-DK-3: Notify 数量固定为 3
NORMAL_NOTIFY_NUM = 3，每个通道固定分配 3 个 notify 资源。
出处: channel.h:24

#### EX-DK-4: RDMA 2GB vs SDMA 4GB 传输上限
节点间 RDMA 单次最大 2GB(WQE 限制)，节点内 SDMA 单次最大 4GB。
超过需要分段传输(与 hcomm platform 层 4GB 切分惯用法对应)。
出处: executor_base.h:27-28

### Hardware Abstractions (硬件知识)

#### EX-HW-1: 协议选择矩阵
| 层级 | 910B | 910_93 |
|------|------|--------|
| Level0 | HCCS | HCCS |
| Level1(单机) | HCCS | HCCS(默认) / ROCE(disableHccs) |
| Level1(多机) | ROCE | ROCE |
| Level2 | ROCE | ROCE |
出处: channel.cc Level1 选择逻辑

#### EX-HW-2: 四种通信协议
COMM_PROTOCOL_HCCS(节点内高速), COMM_PROTOCOL_ROCE(跨节点 RDMA),
COMM_PROTOCOL_UBC_CTP(AIV 2D mesh), COMM_PROTOCOL_UB_MEM(AIV 内存)
出处: channel.cc 协议引用

### Anti-Patterns (反模式)

#### AP-EX-1: Describe() 返回硬编码 "111" [低]
executor_v2_base.cc:26 — `std::string s = "111"; return s;`
明显是占位符实现，28 个子类可能都继承了这个无意义的默认值。
正确做法: 返回实际算法描述，或声明为纯虚。

#### AP-EX-2: namespace 注释错误 [低]
executor_v2_base.h:67 — `} // namespace Hccl` 但实际 namespace 是 `ops_hccl`(L25)。
正确做法: 注释应与实际 namespace 匹配。

#### AP-EX-3: V2 static_assert 错误消息 [低]
coll_alg_v2_exec_registry.h:29 — 消息写 "derived from Hccl::DefaultExecCreatorV2"
但实际检查的是 `InsCollAlgBase`。Copy-paste 错误。

#### AP-EX-4: malloc 无 free (内存泄漏) [高]
channel.cc:314 — GetTopoTypeByLink 中:
```cpp
EndpointDesc *endPointDescs = (EndpointDesc*)malloc(endPointNum * sizeof(EndpointDesc));
// ... 使用后无 free()
```
正确做法: 使用 std::vector<EndpointDesc> 或 RAII wrapper。

#### AP-EX-5: RestoreChannelMap 无输入验证 [中]
executor_v2_base.cc:30-43 — 未检查 resCtx.channels[level] 的 size 是否匹配
algHierarchyInfo.infos.size()，越界访问风险。

#### AP-EX-6: Registry 日志级别不一致 [低]
V1 重复注册用 HCCL_WARNING，V2 重复注册用 HCCL_ERROR。
同一错误条件不同日志级别。

#### AP-EX-7: GetAlgExec 不加锁读 map [中]
两个 Registry 的 Register() 加锁，但 GetAlgExec() 不加锁。
假设注册在启动期完成，运行时只读。如果有动态注册则存在 TOCTOU 竞态。
出处: coll_alg_exec_registry.cc:32-40, coll_alg_v2_exec_registry.cc:33-41

#### AP-EX-8: RefreshAlgType 重复代码 [低]
executor_base.cc:51-76 — Level0/1/2 的检查逻辑完全相同，仅 level 索引不同，
相同代码重复 3 次(~8 行 x 3)。
正确做法: 循环 level 0..2。

#### AP-EX-9: root 参数全部 (void) 抑制 [低]
channel_request.cc 中所有连接计算函数接受 root 参数但不使用。
接口预留但当前全部 unused，增加理解成本。

### Notable Patterns (值得注意的模式)

1. hccl 的 executor 模块相比 hcomm algorithm 层(882 文件/122,497 行)极其精简
   (12 文件/~1300 行)。核心框架只定义基类和注册机制，具体算法由各算子目录实现。

2. V1→V2 演进方向清晰:
   - 更严格的接口(纯虚 vs 默认空实现)
   - 更丰富的类型信息(可序列化资源上下文、网络层拓扑细节)
   - 更灵活的注册(双维索引 + 模板组合注入)
   - shared_ptr 替代 unique_ptr(支持共享所有权)

3. 零 TODO/FIXME/HACK: executor/ 目录完全干净，代码成熟。

4. channel/ 采用过程式风格而非 OO:
   所有函数都是自由函数(非类方法)，依赖 HcclComm 句柄和 RankGraph API。
   这与 hcomm communicator base/ 的面向对象风格(CommBase/CalcTransportReqBase 继承体系)
   形成鲜明对比。

---

## 3.1.2 selector/ -- 算法选择器，决策逻辑

### Module Overview

selector/ 是 hccl 算子框架的算法选择模块，负责在运行时根据拓扑、数据量、数据类型、
执行模式等条件为每种集合通信算子选择最优算法实现。

模块分为两部分:
- 核心基础设施: `src/ops/op_common/selector/` (6个文件)
- 算子特定选择器: 分布在 13 个算子目录下(各自的 `selector/` 子目录)

总规模: 34个源文件(17 .h + 17 .cc)，总约2500行。

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| op_common/selector/auto_selector_base.h | 113 | 基类 + 常量 + 算法名map(4张) |
| op_common/selector/auto_selector_base.cc | 282 | 基类实现: Select()入口 + 5个virtual默认 + 拓扑查询 |
| op_common/selector/execute_selector.h | 27 | ExecuteSelector类声明 |
| op_common/selector/execute_selector.cc | 58 | 选择器编排: MC2硬编码 + 按优先级遍历 |
| op_common/selector/selector_registry.h | 55 | SelectorRegistry单例 + 双注册宏 |
| op_common/selector/selector_registry.cc | 64 | Registry实现: Global() + Register() + Get() |
| all_reduce/selector/..cc | 350 | 最复杂: 22个算法名, 4种执行模式全覆盖 |
| reduce_scatter/selector/..cc | 353 | 次复杂: 19个算法名, 5种执行模式全覆盖(含DPU) |
| all_gather/selector/..cc | 271 | 19个算法名, 5种执行模式全覆盖(含DPU) |
| reduce/selector/..cc | 225 | 10个算法名 |
| broadcast/selector/..cc | 151 | 9个算法名 |
| all_gather_v/selector/..cc | 146 | 8个算法名 |
| reduce_scatter_v/selector/..cc | 205 | 7个算法名 |
| scatter/selector/..cc | 126 | 8个算法名 |
| alltoall/selector/..cc | 126 | 8个算法名 |
| alltoallv/selector/..cc | 112 | 7个算法名 |
| alltoallvc/selector/..cc | 75 | 2个算法名 |
| batch_send_recv/selector/..cc | 36 | 1个算法名(最简) |
| send/selector/..cc | 27 | 1个算法名 |
| recv/selector/..cc | 27 | 1个算法名 |

### 架构: Template Method + Strategy + Priority Registry

#### 1. SelectorRegistry -- 优先级自注册工厂

```cpp
// selector_registry.h
class SelectorRegistry {
    static SelectorRegistry *Global();                // 全局单例(new泄漏式)
    Register(u32 priority, AutoSelectorBase *selector);           // 全局注册(REGISTER_SELECTOR)
    RegisterByOpType(HcclCMDType opType, u32 priority, ...);     // 按算子类型注册
    map<u32, AutoSelectorBase*> GetAllSelectors();               // 获取全局map
    map<u32, AutoSelectorBase*> GetSelectorsByOpType(opType);    // 按算子类型获取
private:
    map<u32, AutoSelectorBase*> impls_;                          // 全局selector(priority→selector)
    map<HcclCMDType, map<u32, AutoSelectorBase*>> opTypeImpls_;  // 按算子类型索引
};
```

注册宏(2种):
- `REGISTER_SELECTOR(priority, selector)` -- 注册到全局impls_
- `REGISTER_SELECTOR_BY_OPTYPE(optype, priority, selector)` -- 注册到opTypeImpls_

量化: 全仓 14 个 REGISTER_SELECTOR_BY_OPTYPE 注册，优先级全部是 18。
REGISTER_SELECTOR 宏在代码中未被使用(impls_仅供MC2快速查找)。

#### 2. ExecuteSelector -- 选择器编排

```cpp
// execute_selector.cc
HcclResult ExecuteSelector::Run(OpParam &opParam, ...) {
    if (opParam.isMc2) {
        selectors.find(18);         // 硬编码优先级18! MC2直连CCU selector
        iter->second->Select(...);
        return;
    }
    selectors = GetSelectorsByOpType(opParam.opType);  // 按算子类型获取
    for (auto iter : selectors) {                       // 按优先级升序遍历
        if (iter.second->Select(...) == MATCH) return;  // 首个匹配即返回
    }
    return HCCL_E_NOT_SUPPORT;
}
```

关键设计: MC2路径硬编码 `selectors.find(18)` 跳过opType过滤，直接通过全局registry
以优先级18查找。这要求所有selector都注册到优先级18(当前确实如此)。

#### 3. AutoSelectorBase -- 5路执行模式分派

```cpp
// auto_selector_base.cc: Select()
SelectorStatus Select(OpParam, TopoInfo*, selectAlgName&, opExecuteConfig&) {
    // 1. DPU优先: 检查hostDPUOnly
    if (hostDPUOnly) → SelectDPUAlgo()
    // 2. CCU_MS模式
    if (CCU_MS) → SelectCcuMsAlgo(); 失败则降级到CCU_SCHED
    // 3. CCU_SCHED模式
    if (CCU_SCHED) → SelectCcuScheduleAlgo(); 失败则标记CCU_FAIL
    // 4. AIV模式
    if (AIV) → SelectAivAlgo(); 失败则标记CCU_FAIL
    // 5. AICPU兜底(Stars状态: AICPU_TS / HOSTCPU_TS / CCU_FAIL)
    if (IsStarsState) → SelectAicpuAlgo()
}
```

5个virtual方法的默认实现全部返回 NOT_MATCH(空壳)，子类按需override。
执行模式降级链: CCU_MS → CCU_SCHED → AICPU(兜底)。

#### 4. 算子特定Selector -- 13个子类

每个算子有一个selector子类，override不同数量的virtual方法:

| 算子 | 覆盖方法数 | 算法名数 | 复杂度 |
|------|-----------|---------|--------|
| AllReduce | 4 | 22 | 最高 |
| ReduceScatter | 5 | 19 | 高(含DPU) |
| AllGather | 5 | 19+ | 高(含DPU) |
| Reduce | 4 | 10 | 中 |
| ReduceScatterV | 4 | 7 | 中 |
| AllGatherV | 4 | 8 | 中 |
| Broadcast | 4 | 9 | 中 |
| Scatter | 4 | 8 | 中 |
| AlltoAll | 4 | 8 | 中(CcuMs空) |
| AlltoAllV | 3 | 7 | 低 |
| AlltoAllVC | 2 | 2 | 低 |
| BatchSendRecv | 1 | 1 | 最低 |
| Send | 1 | 1 | 最低 |
| Recv | 1 | 1 | 最低 |

### 算法选择决策树(通用结构)

所有selector共享统一的决策框架:

```
Level 1: 执行模式选择 (由基类Select()分派)
  ├─ DPU (hostDPUOnly → SelectDPUAlgo)
  ├─ CCU_MS → SelectCcuMsAlgo
  ├─ CCU_SCHED → SelectCcuScheduleAlgo
  ├─ AIV → SelectAivAlgo
  └─ AICPU → SelectAicpuAlgo (兜底)

Level 2: 前置排除 (各selector的方法开头)
  ├─ 不支持 HCCL_REDUCE_PROD → NOT_MATCH
  ├─ 不支持 64位数据类型(INT64/UINT64/FP64) → NOT_MATCH
  └─ 不支持 INT8 (仅CCU_MS部分算子) → NOT_MATCH

Level 3: 拓扑层级判断
  ├─ topoLevelNums > 1 (多层/跨节点)
  │   └─ 通常选 NHR 或 Parallel 算法
  └─ topoLevelNums == 1 (单层/节点内)
      └─ 进入拓扑形状判断

Level 4: 拓扑形状判断
  ├─ MESH_1D → 基础mesh算法
  │   ├─ TWO_DIE_REGULAR → 2Die特化算法
  │   ├─ TWO_DIE_NOT_REGULAR → NOT_MATCH
  │   └─ 其他 → 标准1D mesh算法
  ├─ MESH_1D_CLOS (UBX机型) → 复杂分支
  │   ├─ meshNum == closNum && rankSize <= 4 → Concurrent算法
  │   ├─ closNum是meshNum的整数倍 → Parallel算法
  │   └─ 其他 → NHR fallback
  ├─ CLOS → NHR算法
  └─ 其他 → NOT_MATCH

Level 5: 数据量分档
  ├─ small (< 512KB) → OneShot变体
  ├─ medium (512KB ~ 1MB) → TwoShot变体
  └─ large (>= 1MB) → MeshChunk / MultiMission变体
```

### 算法命名规范

算法名由四段构成: `{Engine}{OpName}{Topology}{Variant}`

Engine前缀:
- `Ccu` -- CCU引擎(MS/Schedule模式)
- `Ins` -- AICPU指令流引擎
- `Aiv` -- AIV向量核引擎
- `InsV2` -- V2 AICPU(仅DPU场景)
- (无前缀) -- Reduce的AICPU部分算法(命名不一致)

Topology:
- `Mesh1D` / `Mesh2D` / `Mesh2Die` -- mesh拓扑变体
- `NHR` / `NHR1D` -- Non-Hierarchical Ring
- `ParallelMesh1DNHR` -- 并行mesh+NHR组合
- `Concurrent` -- 并发算法(小rank数)

Variant后缀:
- `OneShot` / `TwoShot` -- 数据量级别
- `Mem2Mem` -- CCU Schedule模式特有
- `MultiMission` -- CCU MS大数据模式
- `MeshChunk` -- AICPU大数据分块
- `MultiJetty` -- UBX多Jetty
- `UBX` -- UBX机型特化

### 常量与阈值

```cpp
// auto_selector_base.h
constexpr uint64_t SMALL_COUNT_512KB = 512*1024;     // IsSmallData阈值
constexpr uint64_t LARGE_COUNT_1024KB = 1024*1024;   // IsLargeData阈值
constexpr int RANK_SIZE_EIGHT = 8;
constexpr u32 CCU_MS_MODE = 2;

// all_reduce_auto_selector.cc
constexpr u64 AR_M2M_1D_MAX_DATA_SIZE = 16*1024*1024;     // 16MB, CCU Schedule上限
constexpr u64 AR_AICPU_1D_SMALL_DATA_SIZE = 8*1024*1024;  // 8MB, AICPU oneshot/twoshot分界
constexpr u64 AR_AICPU_1D_MAX_DATA_SIZE = 32*1024*1024;   // 32MB, AICPU twoshot/meshchunk分界
constexpr u32 MAX_RANK_NUM_FOR_CONCURRENT_ALGO = 4;        // Concurrent算法rank上限

// reduce_scatter_auto_selector.cc 特有
double ratio = (8.0 / userRankSize) * (8.0 / userRankSize);  // 动态阈值, 8卡基线平方反比
```

不同执行模式使用不同阈值体系:
- CCU_MS/AIV: 基类的512KB/1MB(UB协议粒度)
- CCU_Schedule: 算子特定(如AllReduce 16MB)
- AICPU: 算子特定(如AllReduce 8MB/32MB)

### 全局算法名Map(基类预定义)

```cpp
// auto_selector_base.h: 4张opType→algName映射表
OP_TYPE_TO_AICPU_SOLE_ALG_MAP    // AICPU单机sole算法(6条)
OP_TYPE_TO_CCU_1D_ALG_MAP         // CCU 1D算法(7条, 含HALF_ALLTOALLV)
OP_TYPE_TO_CCU_2D_ALG_MAP         // CCU 2D算法(5条)
OP_TYPE_TO_DPU_ALG_MAP            // DPU算法(空map, 预留)

// 资源复用映射
RES_RESUSE_ALG = {                 // 6条, 共享资源的算法别名
    "InsReduceScatterMesh1D" → "InsReduceScatterMeshClass",
    "InsAllReduceMesh1DOneShot" → "InsAllReduceMeshClass",
    "InsSend" → "InsSendRecv",
    ...
};
```

### Idioms & Conventions (惯用法)

#### SEL-ID-1: Template Method + NVI变体
基类 Select() 是模板方法骨架(非virtual)，分派到5个virtual方法。
子类全部在private段override这些virtual方法(NVI惯用法)。
出处: auto_selector_base.h:72-96, 所有13个子类

#### SEL-ID-2: Priority-based自注册工厂
REGISTER_SELECTOR_BY_OPTYPE宏 + 静态初始化注册 + std::map<u32,...>自动按优先级排序。
量化: 14个注册，优先级全部18。
出处: selector_registry.h:35-53

#### SEL-ID-3: SelectorStatus二值返回
MATCH/NOT_MATCH简洁的二值枚举(非HcclResult错误码)，与CHK_RET体系分离。
量化: SelectorStatus出现366次/31文件。
出处: auto_selector_base.h:28

#### SEL-ID-4: (void)param抑制未使用警告
基类5个virtual默认实现中对所有参数执行(void)param。
出处: auto_selector_base.cc:90-138

#### SEL-ID-5: 执行模式降级链
CCU_MS失败→降级CCU_SCHED→失败→降级AICPU兜底。
决策在基类Select()中实现，子类无需关心降级逻辑。
出处: auto_selector_base.cc:16-66

#### SEL-ID-6: configAlgMap外部配置查询
每个Select方法开头从GetExternalInputHcclAlgoConfigAllType()获取用户配置。
但大多数selector查询后不使用(仅日志打印)，真正使用的只有部分CcuMs/CcuSchedule分支。
出处: auto_selector_base.cc:20, 各子类SelectXxx方法

#### SEL-ID-7: CHK_PRT_RET宏做前置排除
不支持的配置(PROD/64bit/INT8)通过CHK_PRT_RET + WARNING日志排除。
量化: CHK_RET/CHK_PRT_RET共53次/8个selector文件。
出处: 各子类SelectXxx方法开头

### Architecture & Patterns (架构模式)

#### SEL-AP-1: 五路执行模式分派
DPU → CCU_MS → CCU_SCHED → AIV → AICPU 的固定分派序。
每个子类选择性override(1~5个方法)，未override的返回NOT_MATCH自动降级。
这是Strategy模式的变体: 基类控制策略调用序，子类控制每个策略的实现。

#### SEL-AP-2: 拓扑驱动的算法选择
核心决策维度: topoLevelNums(层数) × level0Topo(形状) × meshType(子形状) × dataSize(数据量)。
所有selector共享相同的决策框架(拓扑→数据量→算法)，仅具体分支和算法名不同。

#### SEL-AP-3: Concurrent/Parallel/NHR三级扩展策略
- rank <= 4 且 meshNum==closNum → Concurrent(并发算法)
- closNum是meshNum整数倍 且大数据 → Parallel(并行mesh+NHR)
- 其他 → NHR(通用非层次Ring)
这是UBX(MESH_1D_CLOS)机型的标准三级选择模式。

#### SEL-AP-4: 算法名=注册key
selector输出的字符串(selectAlgName)直接作为ExecutorRegistry的查找key。
selector和executor通过字符串名耦合(非类型耦合)。

### Domain Knowledge (业务知识)

#### SEL-DK-1: 执行模式矩阵
| 模式 | 引擎 | 用途 |
|------|------|------|
| DPU | Host CPU | 仅跨节点，Host上的DPU加速 |
| CCU_MS | CCU | CCU Multi-Stream(内存共享)，单层拓扑为主 |
| CCU_SCHED | CCU | CCU Schedule(Mem2Mem)，支持多层拓扑 |
| AIV | AIV | 向量核，仅单层MESH_1D |
| AICPU_TS | AICPU | 指令流，最通用的兜底引擎 |
| HOSTCPU_TS | Host CPU | Host侧AICPU指令流 |

#### SEL-DK-2: 拓扑形状含义
- MESH_1D: 节点内设备全连接mesh(标准场景)
- MESH_1D_CLOS: UBX机型，CLOS交换网络 + mesh子拓扑
- TWO_DIE_REGULAR: 双Die对称(新硬件架构)
- TWO_DIE_NOT_REGULAR: 双Die非对称(不支持，全部NOT_MATCH)

#### SEL-DK-3: 算法与数据量的关系
AllReduce为例:
- AICPU: <=8MB OneShot, 8~32MB TwoShot, >32MB MeshChunk
- CCU_MS: <=512KB OneShot, >512KB 标准(或MultiMission)
- AIV: <=512KB OneShot, >512KB TwoShot

不同引擎的分档阈值不同，因为硬件特性不同(UB Buffer大小、SDMA带宽等)。

#### SEL-DK-4: PROD操作的普遍排除
所有涉及reduce操作的selector(AllReduce/ReduceScatter/Reduce/ReduceScatterV)
在所有执行模式下都排除PROD(乘法)。
原因: CCU/AIV硬件不支持乘法reduce，只能用AICPU软件实现(但AICPU也排除)。

#### SEL-DK-5: 64位类型的部分排除
INT64/UINT64/FP64 在CCU_MS、CCU_SCHED、AIV模式下被排除。
AICPU模式对64位类型有特殊算法(如 InsReduceScatterAicpuReduce)。
排除原因: CCU/AIV ALU不支持64位reduce运算。

### Hardware Abstractions (硬件知识)

#### SEL-HW-1: 执行引擎能力矩阵
| 能力 | CCU_MS | CCU_SCHED | AIV | AICPU |
|------|--------|-----------|-----|-------|
| INT8 | 部分不支持 | 支持 | 支持 | 支持 |
| 64位类型 | 不支持 | 不支持 | 不支持 | 特殊算法 |
| PROD | 不支持 | 不支持 | 不支持 | 不支持 |
| 多层拓扑 | 部分 | 支持 | 不支持 | 支持 |
| CLOS | 支持 | 支持 | 不支持 | 支持 |
| 2Die | 支持 | 支持 | 不支持 | 不支持 |

#### SEL-HW-2: UBX机型(MESH_1D_CLOS)特殊处理
UBX机型使用CLOS交换网络，带来特殊的算法选择维度:
- meshNum vs closNum的关系决定拓扑对称性
- rank <= 4时启用Concurrent算法(小规模利用全连接)
- 多Jetty(MultiJetty)算法利用UBX的多个传输通道

### Anti-Patterns (反模式)

#### AP-SEL-1: FP64重复/INT64遗漏 Bug [高]
3个selector(all_reduce, reduce, reduce_scatter_v)的CcuMs模式中:
```cpp
// all_reduce_auto_selector.cc:42-44
if (opParam.DataDes.dataType == HCCL_DATA_TYPE_FP64 ||     // FP64
    opParam.DataDes.dataType == HCCL_DATA_TYPE_UINT64 ||   // UINT64
    opParam.DataDes.dataType == HCCL_DATA_TYPE_FP64) {     // FP64重复! 缺INT64!
```
日志写"not support INT64, UINT64, FP64"但代码漏检INT64。
同时基类已提供Is64BitDataType()工具函数(正确覆盖三种类型)，部分方法使用了但CcuMs没用。
正确做法: 使用 Is64BitDataType() 替代手写条件。
出处: all_reduce_auto_selector.cc:42-44, reduce_auto_selector.cc:36-38, reduce_scatter_v_auto_selector.cc:38-40

#### AP-SEL-2: AllGatherV CcuMs fall-through Bug [高]
all_gather_v_auto_selector.cc: SelectCcuMsAlgo中，多层拓扑分支设置了
selectAlgName = "CcuAllGatherVParallelMeshNHR" 但没有return，代码继续执行到
SelectMeshAlgo覆盖了算法名。多层拓扑的Parallel算法实际永远不会生效。
出处: all_gather_v_auto_selector.cc:23-43

#### AP-SEL-3: 硬编码优先级18 [中]
ExecuteSelector::Run() 中 MC2路径用 `selectors.find(18)` 硬编码查找。
如果未来某个selector注册非18优先级，MC2路径会找不到。
正确做法: 通过opType查找，或声明常量。
出处: execute_selector.cc:28

#### AP-SEL-4: levle0Algo拼写错误 [低]
变量名 `levle0Algo`(应为level0Algo)出现在8个selector文件中共57处。
统一的copy-paste错误，说明从同一模板复制。
出处: reduce_scatter/all_gather/broadcast/scatter/reduce_scatter_v/all_gather_v/alltoallvc等

#### AP-SEL-5: namespace注释不匹配 [低]
所有selector .h文件关闭时注释 `} // namespace Hccl`，但实际namespace是 `ops_hccl`。
量化: 8个头文件有此错误。
出处: auto_selector_base.h:111, 各子类.h

#### AP-SEL-6: 算法名大小写/拼写不一致 [中]
- `CcuAllreduceMesh2DieBigMs` (小写r) vs `CcuAllReduceMesh1D` (大写R)
- `CcuAllReduceParallelNHR1DMutiJetty` (Muti) vs `Multi` (正确)
- `CcuReduceScatterNhr1DMem2Mem` (Nhr) vs `NHR` (其他地方)
- `CcuAllToAllMesh2Die` vs `CcuAlltoAllMesh1D` (toAll/ToAll)
- Reduce AICPU算法缺少"Ins"前缀: `ReduceMesh1D` vs `InsReduceMesh1D`
这些字符串是executor注册的查找key，拼写不一致增加维护风险。

#### AP-SEL-7: 死代码常量 [低]
- `RS_MAX_DATA_SIZE` (all_reduce_auto_selector.cc): 声明未使用
- `RS_2D_SMALL_DATA_SIZE` (reduce_scatter_auto_selector.cc): 声明未使用
- `AG_2D_SMALL_DATA_SIZE` (all_gather_auto_selector.cc): 声明未使用
3个selector都有声明但未使用的常量(可能是重构残留)。

#### AP-SEL-8: configAlgMap查询但结果未使用 [低]
大多数SelectAicpuAlgo/SelectAivAlgo方法从configAlgMap中提取算法配置并打印日志，
但实际选择逻辑不受配置影响。仪式性代码(从模板复制来)。

#### AP-SEL-9: include guard copy-paste错误 [低]
all_gather_v_auto_selector.h 使用了 `HCCLV2_REDUCESCATTTER_AUTO_SELECTOR`
(ReduceScatter的guard名 + SCATTER多了一个T)，而不是AllGatherV自己的guard。
出处: all_gather_v_auto_selector.h:11

#### AP-SEL-10: 全局const map在header中 [中]
auto_selector_base.h 中定义了4个 `const std::map<...>` 全局变量。
每个包含该header的翻译单元都会有一份副本(ODR风险)。
正确做法: 使用 `inline const` 或移到.cc文件。
出处: auto_selector_base.h:30-68

#### AP-SEL-11: SelectorRegistry内存泄漏 [低]
- Global()使用 `new SelectorRegistry` 且永不delete(intentional singleton leak)
- REGISTER宏使用 `new selector()` 创建对象，永不delete(进程生命周期)
单例+全局对象的标准做法，但依赖进程退出回收。

#### AP-SEL-12: GetSelectorsByOpType/GetAllSelectors返回副本 [低]
两个Get方法返回map副本(非const引用)，每次调用都复制整个map。
ExecuteSelector::Run()每次调用都会触发map复制。
出处: selector_registry.cc:49-61

#### AP-SEL-13: Register加锁但Get不加锁 [中]
Register()使用lock_guard保护，但GetSelectorsByOpType()和GetAllSelectors()不加锁。
与executor registry相同的假设: 注册在启动期完成，运行时只读。
出处: selector_registry.cc:26-61

### Notable Patterns (值得注意的模式)

1. 选择器复杂度与算子复杂度正相关:
   AllReduce(22算法) > ReduceScatter(19) > AllGather(19) >> Send/Recv(1)
   最复杂的三个算子(AR/RS/AG)覆盖全部5种执行模式，其余递减。

2. RS与AG的选择逻辑高度对称(互为逆操作)，但算法名并非简单替换。

3. P2P操作(Send/Recv/BatchSendRecv)只支持AICPU模式，反映了点对点通信
   不需要复杂的硬件加速(数据量小、无reduce计算)。

4. TWO_DIE_NOT_REGULAR始终返回NOT_MATCH:
   双Die非对称拓扑目前无任何算法支持(可能是新硬件形态尚未完善)。

5. DPU模式目前仅ReduceScatter和AllGather实现(各1个算法)，
   基类的OP_TYPE_TO_DPU_ALG_MAP为空map(预留)。

6. 零 TODO/FIXME/HACK: selector模块无技术债标记。

7. selector与executor通过字符串名耦合:
   selector输出字符串 → executor registry查找 → 创建executor
   没有编译时类型检查，拼写错误只能在运行时发现。

8. AlltoAll系列不支持多层拓扑(topoLevelNums > 1全部NOT_MATCH):
   AlltoAll的O(N^2)数据交换特性不适合层次化分解。

---

## 3.1.3 template/ 根目录基类体系分析

### Module Overview

template/根目录包含5对(10个)文件，构成hccl算子模板框架的四层基类体系:

| 基类 | 文件 | 行数(h+cc) | 处理器目标 | 子类数 |
|------|------|-----------|-----------|--------|
| AlgTemplateBase | alg_template_base.h/cc | 226+177=403 | SDMA(v1) | 5(scatter系列) |
| InsAlgTemplateBase | alg_v2_template_base.h/cc | 78+89=167 | AICPU(v2) | 20+ |
| AivAlgTemplateBase | aiv_alg_template_base.h/cc | 66+142=208 | AIV核心 | 9 |
| CcuAlgTemplateBase | ccu_alg_template_base.h/cc | 59+113=172 | CCU编排器 | 32+ |
| (工具) | template_utils.h/cc | 301+33=334 | 共享数据结构 | N/A |

总规模: 730行头文件 + 554行实现 = 1284行。

### 核心发现: 四代演进的基类体系

#### 1. AlgTemplateBase — V1代(SDMA路径)

文件: alg_template_base.h(226行) / alg_template_base.cc(177行)

类名: `AlgTemplateBase`
命名空间: `ops_hccl`
继承: 独立基类(无父类)
子类: 5个(ScatterRing/ScatterMesh/ScatterNB/ScatterRingDirect/NHRBase)，全部在scatter算子内

关键设计:
- Template Method模式的非经典变体: 所有virtual方法都有默认实现(非pure virtual)
- `RunAsync` 3个重载: (rank,rankSize,channels) / (+slaveThreads) / (dpuResCtx)
  全部返回HCCL_SUCCESS(空操作)，依赖子类override
- `Prepare` 4个重载: 12参数完整版 / PrepareData结构体版 / ScatterMesh 2参数版 / 4参数直接版
  仅12参数版有实际实现，其余返回HCCL_E_PARA
- Barrier同步: 5个ExecuteBarrier重载，基于Notify的Record/Wait协议
  两阶段同步: ACK同步(保证前一步完成) + DATA_SIGNAL同步(保证数据就位)

成员变量:
```
slicesDummy_/slices_    // Slice dummy+引用(避免拷贝)
inputMem_/outputMem_/scratchMem_  // 三区域buffer模型
count_/dataBytes_/dataType_/reductionOp_/root_  // 操作参数
unitSize_/disableDMAReduce_/baseOffset_  // 传输控制
thread_              // ThreadHandle(SDMA stream抽象)
barrierSwitchOn_     // Barrier开关
multRingsSlices_     // double ring切片
```

惯用法:
- (void)rank; 显式标记unused参数(alg_template_base.cc:79-81, 86-90, 96)
- CHK_RET + static_cast<HcclResult> 组合(所有Barrier实现)
- CHK_PRT_RET 用于需要打印错误日志的参数校验(alg_template_base.cc:160-164)
- `slices_`是`slicesDummy_`的引用(构造时绑定)，避免空slice时的特殊处理
- DataUnitSize通过SIZE_TABLE数组查找(内联实现在.h中)

值得注意的注释: L184 "// 下面这组是否需要？" — 对两个Barrier重载的自我怀疑

#### 2. InsAlgTemplateBase — V2代(AICPU路径)

文件: alg_v2_template_base.h(78行) / alg_v2_template_base.cc(89行)

类名: `InsAlgTemplateBase`
命名空间: `ops_hccl`
继承: 独立基类(不继承AlgTemplateBase)
子类: 20+个(InsTempAllReduceMesh1DTwoShot, InsTempReduceScatterNHR, InsTempBroadcastMesh1DTwoShot等)

关键设计:
- 与V1完全独立的类层次(不是继承演进而是并行替代)
- 纯虚方法: `Describe() const = 0`(唯一强制), `GetNotifyIdxMainToSub/SubToMain = 0`(2个)
- 核心虚方法: `KernelRun` / `DPUKernelRun` / `CalcRes` / `GetRes` / `CalcScratchMultiple` / `GetThreadNum`
  全部有默认实现: KernelRun/DPUKernelRun/CalcRes/GetRes返回HCCL_E_INTERNAL + HCCL_ERROR日志
  CalcScratchMultiple/GetThreadNum返回0
- 构造函数从OpParam提取: opMode, root, reduceOp, enableDetour
- templateRankSize_计算: 两层subCommRanks时取乘积(subCommRanks[0].size() * subCommRanks[1].size())

成员变量(与V1对比):
```
opMode_              // V2新增: 单算子/图模式区分
root_/myRank_/templateRankSize_/subCommRanks_  // rank信息(V1在Prepare中传入)
buffInfo_            // V2新增: BuffInfo结构体(V1用三个独立DeviceMem)
threadNum_           // V2新增: 多线程支持
reduceOp_/dataType_  // 操作参数(V1在Prepare中设置)
enableDetour_        // V2新增: 绕路开关
notifyIdxMainToSub_/notifyIdxSubToMain_  // V2新增: 主从线程通知索引
```

V1→V2关键差异:
- V1在Prepare中设置所有参数(两段式构造)，V2在构造函数中设置(一段式)
- V1以ThreadHandle为stream抽象，V2以TemplateResource(channels+threads+ccuKernels)
- V1无资源计算，V2有CalcRes/GetRes(编排前计算资源需求)
- V1无DPU支持，V2有DPUKernelRun
- V1的默认实现返回HCCL_SUCCESS(静默成功)，V2返回HCCL_E_INTERNAL(显式失败+日志)

#### 3. AivAlgTemplateBase — AIV核心计算路径

文件: aiv_alg_template_base.h(66行) / aiv_alg_template_base.cc(142行)

类名: `AivAlgTemplateBase`
命名空间: `ops_hccl`
继承: 独立基类
子类: 9个(AivTempAllReduceMesh1DOneShot/TwoShot, AivTempReduceScatterMesh1D, AivTempAllGatherMesh1D等)

与InsAlgTemplateBase的对比:
- 接口高度相似(CalcRes/KernelRun/CalcScratchMultiple/Describe)
- 无DPUKernelRun(AIV不在DPU上运行)
- 无GetRes/GetThreadNum(AIV资源模型不同)
- 多了CalNumBlocks(多核分配)和Pre/PostSync系列(AIV流间同步)
- 多了sliceId_(递增计数器，用于aivCountTag组装)

独特方法:
- `CalNumBlocks`: 计算实际使用的AIV核心数，默认=min(tempRankSize_, numBlocksLimit)
- `PreSync/PostSync`: 主从流同步协议
  PreSync: 主流向所有从流发Record(LOCAL_NOTIFY_IDX_ZERO) → 从流Wait
  PostSync: 从流向主流发Record(queIdx-1) → 主流Wait所有从流
- `PreSyncInterQueues/PostSyncInterQueues`: 批量同步(循环调用Pre/PostSync)

流间同步模型(aiv_alg_template_base.cc:103-140):
```
PreSync:
  主流(queIdx=0): for each 从流 → Record(目标流, LOCAL_NOTIFY_IDX_ZERO)
  从流(queIdx>0): Wait(LOCAL_NOTIFY_IDX_ZERO)

PostSync:
  从流(queIdx>0): Record(主流, queIdx-1)
  主流(queIdx=0): for each 从流 → Wait(qidx, CUSTOM_TIMEOUT)
```

值得注意的注释: L78 "// 可能用不到，预留" — PreSyncInterQueues自我怀疑
值得注意的bug: PostSyncInterQueues日志打印"[AivCollAlgFactory]"而非"[AivAlgTemplateBase]"
  (aiv_alg_template_base.cc:95, copy-paste错误)

#### 4. CcuAlgTemplateBase — CCU编排路径

文件: ccu_alg_template_base.h(59行) / ccu_alg_template_base.cc(113行)

类名: `CcuAlgTemplateBase`
命名空间: `ops_hccl`
继承: 独立基类
子类: 32+个(最多的子类)

与InsAlgTemplateBase的差异:
- 双构造: 默认构造函数 + 参数化构造函数(其他基类只有参数化)
- 额外的InitCcuAlgTemplate虚方法: 延迟初始化(先默认构造，后Init)
- templateRankSize_计算不同: 所有subCommRanks的size之和(而非乘积)
  — 这反映了CCU在物理die层面的资源分配逻辑
- 多了scratchBufferSize_成员: CCU需要额外scratch buffer
- 多了4个独特方法:
  - PointerToAddr: void* → uint64_t(reinterpret_cast, nullptr→0)
  - GetChannelDieId: 查询channel对应的die ID(通过HcclRankGraphGetEndpointInfo)
  - RestoreChannelMap: channelDescs → map<remoteRank, vector<channelDesc>>
  - 这3个是CCU特有的硬件层操作(die感知、channel重建)

templateRankSize_三种计算方式:
```
InsAlgTemplateBase:  subCommRanks[0].size() * subCommRanks[1].size()  // 二维乘积
AivAlgTemplateBase:  subCommRanks[0].size()                          // 仅第一维
CcuAlgTemplateBase:  sum(ranks.size() for ranks in subCommRanks)     // 所有维度之和
```
含义: AICPU看到的是逻辑rank(层级乘积), AIV看到的是同层rank(单级), CCU看到的是物理die(所有die之和)

#### 5. template_utils.h/cc — 共享数据结构和工具

文件: template_utils.h(301行) / template_utils.cc(33行)

namespace: `ops_hccl`

核心数据结构:
- `DataSlice`: addr + offset + size + count + Describe()
  — V2的切片抽象(V1用Slice)，比V1更完整(有count字段)
- `SlicesList`: srcSlices + dstSlices — 发送接收切片对
- `TxRxChannels`: txChannel + rxChannel — 发送接收通道对
- `SendRecvInfo`: TxRxChannels + TxRxSlicesList — 完整收发描述
- `DataInfo/DataReduceInfo`: 带channel的数据操作描述
- `A2ASendRecvInfo`: AlltoAll专用(sendLength/Offset/Counts/Displs)
- `BuffInfo`: input/output/hcclBuff三区域 + 类型 + 大小 + 基准偏移
- `TemplateDataParams`: 完整模板参数(BuffInfo + count + sliceSize + stride + repeat + 变长数据)
  带Serialize/DeSerialize方法(BinaryStream)
- `TemplateResource`: channels + threads + ccuKernels + shmem指针 + aivCommInfo
- `DPURunInfo`: DPU执行信息(templateName + tempAlgParams + channels + rank + subCommRanks)
  带Serialize/DeSerialize方法
- `AicpuNHRStepInfo`: NHR算法步骤描述(step/rank/sliceIdx)
- `BufferType`: INPUT/OUTPUT/HCCL_BUFFER/DEFAULT
- `SliceInfo`: offset + size (简单切片信息)
- `RankSliceInfo`: vector<vector<SliceInfo>> (按rank分组的切片)

工具函数:
- `GetAlgRank`: virtRank在rankIds中的位置(线性查找)
- `GetNHRStepNum`: rankSize的log2(用于NHR算法步数计算)
- `RoundUp`: 向上取整(inline)

序列化惯用法:
TemplateDataParams和DPURunInfo都实现了Serialize/DeSerialize，
使用BinaryStream << 和 >> 操作符，支持AICPU和DPU的跨进程数据传递。

### 与hcomm对应模块的关系

这是本次分析最关键的发现。hcomm中存在完全对应的基类:

| hccl | hcomm | 关系 |
|------|-------|------|
| AlgTemplateBase | ExecutorBase(别名AlgTemplateBase) | 同源分叉 |
| InsAlgTemplateBase | InsAlgTemplateBase | 同源分叉 |
| AivAlgTemplateBase | AivAlgTemplateBase | 同源分叉 |
| CcuAlgTemplateBase | CcuAlgTemplateBase | 同源分叉 |
| template_utils.h | template_utils.h | 同源分叉 |

关键差异:

1. hccl AlgTemplateBase vs hcomm ExecutorBase:
   - hcomm叫ExecutorBase，用`using AlgTemplateBase = ExecutorBase`做别名
   - hcomm使用DeviceMem/Stream/Transport(hcomm自有类型)
   - hccl使用HcclMem/ThreadHandle/ChannelInfo(hccl简化类型)
   - hcomm有HcclDispatcher成员(hccl没有)
   - hcomm的Prepare有约30个重载(682行头文件!)，hccl只有4个重载(226行)
   - hcomm有profiling支持(profilerInput_/RegisterProfiler), hccl没有
   - hcomm有nicRankList_/rankSliceLists_/algOpContext_等额外成员
   - hcomm多出方法: PrepareRunAsync, GetNslbAdjInfo, GetHcclOffsetDstRanksMap,
     ExecuteRxSync, ExecuteTxSync, RoundUpWithDivisor

2. hccl InsAlgTemplateBase vs hcomm InsAlgTemplateBase:
   - hcomm用InsQueue/InsQuePtr(指令队列)，hccl用ThreadHandle/ChannelInfo
   - hcomm有更丰富的接口: CalcSliceInfo, CalcResDetour, GetMaxSliceSize, CalcLoopMaxCount等
   - hcomm有DmaMode/OpMode/CollAlgOperator等额外控制
   - hcomm有rank2BitPos_/linkNumBtwPeers_/enableCounterNotify_等额外成员
   - hcomm有PreSync/PostSync(hccl的在AivAlgTemplateBase中)
   - hcomm用tempVTopo_/tempVirtRankMap_(虚拟拓扑), hccl用subCommRanks_

3. hccl AivAlgTemplateBase vs hcomm AivAlgTemplateBase:
   - 结构几乎一致(说明AIV路径较新，分叉时差异小)
   - hcomm多: CalcResDetour, GenExtIns, SetCollOp, SetDmaMode, SetDataType, SetRoot, InitReduceInfo
   - hcomm用tempVTopo_/tempVirtRankMap_, hccl用subCommRanks_
   - hcomm有linkNumBtwPeers_/dmaMode_, hccl没有
   - 常量完全相同: TOPO_LEN_Y_OFFSET=8, TOPO_LEN_Z_OFFSET=16, MAX_DIM_NUM=3

4. namespace差异: hccl用`ops_hccl`, hcomm用`hccl`(部分)和`Hccl`(部分)
   — namespace命名不统一是两仓分叉的遗留痕迹

分叉结论:
hccl的template基类是从hcomm分叉而来的精简版:
- 剥离了hcomm特有的类型依赖(DeviceMem→HcclMem, Stream→ThreadHandle, Transport→ChannelInfo)
- 删除了hcomm的多数Prepare重载(30→4)
- 删除了hcomm的profiling/NSLB等辅助功能
- 保留了核心算法框架(Template Method + virtual方法签名)
- 新增了DPU支持(DPUKernelRun)

### 设计模式总结

1. Template Method(非经典): 基类提供默认空实现(非pure virtual)
   - 优点: 子类只需override关心的方法
   - 缺点: 遗漏override不会编译报错(V1默认返回SUCCESS更危险)

2. Strategy: 4个基类代表4种执行路径(SDMA/AICPU/AIV/CCU)
   - 由selector选择 → executor选择 → 创建对应基类的子类

3. Two-Phase Construction:
   - V1: AlgTemplateBase() + Prepare(...) — 构造与初始化分离
   - V2/AIV/CCU: 构造函数即初始化 — 一段式(但CCU也有Init)

4. Serialize/Deserialize: template_utils中的TemplateDataParams/DPURunInfo
   用BinaryStream实现AICPU/DPU的跨进程数据传递

### 错误处理模式

四个基类的默认实现错误处理策略不同:

| 基类 | 未override的virtual方法返回值 | 风险 |
|------|------------------------------|------|
| AlgTemplateBase(V1) | HCCL_SUCCESS | 高: 静默成功，掩盖遗漏 |
| InsAlgTemplateBase(V2) | HCCL_E_INTERNAL + HCCL_ERROR | 低: 显式失败+日志 |
| AivAlgTemplateBase | HCCL_E_INTERNAL + HCCL_ERROR | 低 |
| CcuAlgTemplateBase | HCCL_E_INTERNAL + HCCL_ERROR | 低 |

V1的HCCL_SUCCESS默认返回是已知的设计缺陷(hcomm中也是如此)。

### 条件编译

这10个文件中没有#ifdef条件编译(干净)。
条件编译在调用方(executor, channel)中通过#ifndef AICPU_COMPILE控制。

### 常量定义

alg_template_base.h定义了大量切片对齐常量:
```
HCCL_MIN_SLICE_ALIGN = 128           // 通用最小对齐
HCCL_MIN_SLICE_ALIGN_910B = 16384    // 910B最小对齐(128倍)
HCCL_MIN_SLICE_ALIGN_910_93 = 16384  // 910_93最小对齐
HCCL_MIN_SLICE_ALIGN_ONCHIP = 512    // 片上最小对齐
HCCL_MIN_PIPLINE_SLICE_ALIGN = 512   // Pipeline最小对齐
HCCL_CHUNK_SIZE = 1GB                // 最大chunk
HCCL_NIC_MAX_NUM = 8                 // 最大NIC数
DOUBLE_RING_NUM = 2                  // 双ring数
HCCL_SPLIT_SIZE_INTER_SERVER = 8MB   // 跨机切分边界
BEST_SPLIT_VALUE_SR = 87             // Single Ring最佳切分比(87%)
BEST_SPLIT_VALUE_DR = 90             // Double Ring最佳切分比(90%)
```
这些常量与hcomm的alg_template_base_pub.h完全一致(同源)。

### 反模式

1. V1默认返回HCCL_SUCCESS: AlgTemplateBase的RunAsync/Prepare默认返回成功，
   如果子类忘记override则静默通过，无任何日志或错误信号。
   (V2/AIV/CCU已修正为HCCL_E_INTERNAL + HCCL_ERROR)

2. PostSyncInterQueues日志copy-paste错误:
   aiv_alg_template_base.cc:95 日志标签写成"[AivCollAlgFactory]"
   应为"[AivAlgTemplateBase]"。

3. "下面这组是否需要？"注释(alg_template_base.h:184):
   两个ExecuteBarrier(带notifyIdx)重载有自我怀疑注释，但已有被调用。
   这类注释应在确认后删除。

4. "可能用不到，预留" 注释(aiv_alg_template_base.cc:78):
   PreSyncInterQueues标记为可能用不到，属于代码债务。

5. slices_引用dummy模式潜在风险:
   alg_template_base.cc析构函数调用slices_.clear()，
   但slices_是slicesDummy_的引用，如果中途被rebind(不可能用引用)
   则不会有问题。但这个模式容易让后来者困惑。

6. CcuAlgTemplateBase::InitCcuAlgTemplate只设置了opMode_和root_，
   遗漏了myRank_/subCommRanks_/templateRankSize_
   (ccu_alg_template_base.cc:30-35)，与构造函数不对称。

7. templateRankSize_三种不同计算方式:
   AICPU(乘积) / AIV(仅第一维) / CCU(求和) — 语义不一致，
   没有文档解释为什么不同，容易在跨设备模板中产生混淆。

### 值得注意的模式

1. 四基类完全独立(不继承): SDMA/AICPU/AIV/CCU各自独立继承链，
   没有提取公共父类。原因可能是:
   - 不同处理器的资源模型差异太大
   - 避免多重继承的复杂度
   - 各路径独立演进(V1最老，CCU最新)
   但导致了大量接口签名重复(CalcRes/KernelRun/CalcScratchMultiple/Describe)。

2. Describe() = 0是唯一统一的纯虚方法:
   V2/AIV/CCU三个新基类都有`virtual std::string Describe() const = 0`，
   用于日志中标识算法名称。V1没有此接口(因为V1只有scatter系列)。

3. BuffInfo结构体的演进:
   V1用三个独立的HcclMem(inputMem_/outputMem_/scratchMem_)
   V2/AIV/CCU用统一的BuffInfo结构体(inputPtr/outputPtr/hcclBuff + 类型+大小+偏移)
   这是一个改进: 将分散的buffer相关参数内聚到一个结构体中。

4. TemplateResource统一资源模型:
   V2引入的TemplateResource(template_utils.h:230-237)统一了channels/threads/ccuKernels/shmem/aivCommInfo，
   所有V2子类的KernelRun都接收这个结构体(而非V1的散装参数)。

5. DPURunInfo的Serialize/DeSerialize:
   支持将完整的算法执行信息序列化到共享内存中，
   由NPU侧构造、DPU侧反序列化执行。这是NPU-DPU协同执行的关键基础设施。

## 3.1.3 template/aicpu/ — AICPU Kernel Launch与异常处理模块

### Module Overview

template/aicpu/ 是hccl中AICPU设备侧kernel入口和诊断基础设施。
这是运行在AICPU处理器上的代码，被编译为 libscatter_aicpu_kernel.so 独立共享库，
通过Runtime的aclrtLaunchKernelWithConfig API由Host侧下发到Device侧执行。

模块在整体架构中的位置：
```
Host侧 (hccl主库):
  Selector → 选算法 → SetCommEngine → LoadAICPUKernel → AicpuKernelLaunch
      |                                    |                    |
      v                                    v                    v
  算法选择              加载kernel .so+.json      序列化OpParam → launch到stream
                                                                    |
Device侧 (scatter_aicpu_kernel.so):                                v
  HcclLaunchAicpuKernel(OpParam*) → 反序列化 → 获取executor → Orchestrate → Notify完成
```

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| kernel_launch.h | 25 | 声明4个RestoreVarData函数 |
| kernel_launch.cc | 300 | 核心: AICPU kernel入口函数 HcclLaunchAicpuKernel |
| load_kernel.h | 24 | 声明LoadAICPUKernel + extern g_binKernelHandle |
| load_kernel.cc | 57 | kernel二进制加载(Host侧, 非Device侧) |
| dfx/task_exception_fun.h | 36 | ScatterOpInfo结构体 + 函数声明 |
| dfx/task_exception_fun.cc | 57 | 异常诊断信息的创建与序列化 |

总计: 499行, 6个文件。

### Kernel Launch的完整流程

#### 1. Host侧准备阶段 (op_common.cc)

Host侧流程在op_common.cc中实现(非aicpu/目录, 但构成完整链路的前半段):

```
Selector(comm, param, topoInfo, algName, opExecuteConfig)  // op_common.cc:47
  → HcclGetOpExpansionMode    // 获取算子展开模式
  → HcclCalcTopoInfo          // 计算拓扑信息
  → ExecuteSelector().Run()    // 算法选择(产出algName字符串)
  → SetCommEngine()            // 设置执行引擎(AICPU_TS/CPU/AIV)
  → LoadAICPUKernel()          // 条件加载kernel .so (load_kernel.cc:40)

HcclExecOp(comm, param, topoInfo, algName)  // op_common.cc:74
  → InsCollAlgBase executor = V2 Registry查找
  → AlgResourceCtxSerializable resCtxHost = 资源序列化
  → HcclThreadAcquireWithStream  // 获取cpuTsThread
  → HcclThreadExportToCommEngine // 导出到AICPU引擎空间
  → HcclGetAlgRes               // 获取算法资源(产出resCtxSequence)
  → GetMainThreadInfo            // 获取Device侧主线程信息
  → HcclAicpuKernelEntranceLaunch(...)  // op_common.cc:141
```

#### 2. Host侧Launch阶段 (op_common.cc:141-213)

```
HcclAicpuKernelEntranceLaunch:
  1. param.resCtx = resCtxSequence       // 将序列化context放入param
  2. sprintf_s(param.algName, algName)    // 将算法名放入param
  3. [DPU特殊路径] HcclTaskRegister       // DPU模式注册回调
  4. HcommThreadNotifyRecordOnThread      // Host stream → Device主thread发起通知
  5. AicpuKernelLaunch(param)             // 实际launch
  6. HcommThreadNotifyWaitOnThread        // Host stream等待Device完成通知
  7. aclrtSynchronizeStream               // 同步stream
```

#### 3. AicpuKernelLaunch ACL API序列 (op_common.cc:174-213)

```
AicpuKernelLaunch:
  1. aclrtBinaryGetFunction(g_binKernelHandle, "HcclLaunchAicpuKernel", &funcHandle)
  2. aclrtKernelArgsInit(funcHandle, &argsHandle)
  3. aclrtKernelArgsAppend(argsHandle, &param, sizeof(OpParam)+varMemSize, &paraHandle)
  4. aclrtKernelArgsFinalize(argsHandle)
  5. aclrtLaunchKernelCfg cfg:
     - NOTIFY_DEFAULT_WAIT_TIME = 27 * 68 = 1836 (超时配置)
     - numBlocks = 1 (单block)
  6. aclrtLaunchKernelWithConfig(funcHandle, 1, stream, &cfg, argsHandle, nullptr)
```

关键细节:
- paramSize = sizeof(OpParam) + param.varMemSize: 变长参数(如AlltoAllV的counts/displs)跟在OpParam后面
- 超时 NOTIFY_DEFAULT_WAIT_TIME = 1836 (单位不明确, 注释说"默认1836等待时长")
- numBlocks固定为1: AICPU kernel总是单block执行

#### 4. Device侧入口 (kernel_launch.cc:41)

这是AICPU设备上执行的C函数入口:

```c
extern "C" unsigned int HcclLaunchAicpuKernel(OpParam *param)
```

返回值: unsigned int (0=成功, 1=失败), 不使用HcclResult。

Device侧流程按设备类型分为两个完全不同的路径:

##### 路径A: DEV_TYPE_910_95 (V2路径)
```
1. AlgResourceCtxSerializable resCtx → DeSerialize(从param->resCtx)
2. RestoreVarData*(按opType分派): 恢复变长指针
3. HcommBatchModeStart(algTag)
4. HcommThreadNotifyWaitOnThread(thread, notifyNum, CUSTOM_TIMEOUT=1800)  // 等Host通知
5. CollAlgExecRegistryV2 → GetAlgExec(opType, algName) → shared_ptr<InsCollAlgBase>
6. executor->Orchestrate(param, resCtx)  // 执行算法
7. HcommThreadNotifyRecordOnThread(thread, exportedThread, 0)  // 通知Host完成
8. HcommBatchModeEnd(algTag)
```

##### 路径B: 非910_95 (V1路径)
```
1. 异常诊断注册:
   a. CreateScatter(param, &opInfo)           // 快照当前算子信息
   b. HcommAcquireComm(commName)              // 获取通信域引用
   c. HcommRegOpInfo(commName, &opInfo)       // 注册算子信息到hcomm
   d. HcommRegOpTaskException(commName, callback) // 注册异常回调
2. AlgResourceCtx* resCtx = reinterpret_cast<>(param->resCtx)  // 非序列化,直接cast
3. ThreadHandle* = reinterpret_cast<>(resCtx + sizeof(AlgResourceCtx))  // 手动偏移取变长数据
4. HcommBatchModeStart(algTag)
5. HcommProfilingInit(threads, slaveThreadNum+1)
6. HcommProfilingReportMainStreamAndFirstTask(thread)
7. HcommThreadNotifyWaitOnThread(...)         // 等Host通知
8. executor->Orchestrate(param, resCtx)       // V1 executor
9. HcomProInfo profInfo → 填充 → HcommProfilingReportDeviceHcclOpInfo
10. HcommThreadNotifyRecordOnThread(...)      // 通知Host完成
11. HcommProfilingReportMainStreamAndLastTask(thread)
12. HcommBatchModeEnd(algTag)
13. HcommProfilingEnd(threads, slaveThreadNum+1)
```

##### 公共尾部:
```
14. HcommReleaseComm(commName)
15. return 0
```

### V1 vs V2路径的关键差异

| 维度 | V1 (非910_95) | V2 (910_95) |
|------|---------------|-------------|
| 资源反序列化 | reinterpret_cast直接转换 + 手动偏移 | DeSerialize()正式反序列化 |
| 变长数据 | 紧跟AlgResourceCtx后的裸内存 | RestoreVarData*重建指针 |
| executor类型 | unique_ptr<ExecutorBase> (V1 Registry) | shared_ptr<InsCollAlgBase> (V2 Registry) |
| Orchestrate签名 | Orchestrate(param, AlgResourceCtx*) | Orchestrate(param, AlgResourceCtxSerializable&) |
| 异常诊断 | CreateScatter + RegOpInfo + RegOpTaskException | 无(V2不注册异常回调) |
| Profiling | 完整Init/First/Last/End四步 | 无(V2不做Device侧profiling) |
| 所有权语义 | unique_ptr(独占) | shared_ptr(共享, 来自Registry) |

### 与hcomm中task/模块的关系

AICPU kernel通过一系列Hcomm*前缀的C函数与hcomm交互:

| hccl调用 (kernel_launch.cc) | hcomm提供方 | 作用 |
|------|------|------|
| HcommAcquireComm/ReleaseComm | task/dispatcher层 | 通信域引用计数管理 |
| HcommBatchModeStart/End | task/dispatcher层 | SQE批量提交模式控制 |
| HcommThreadNotifyWaitOnThread | task/dispatcher层(DispatcherAiCpu) | Thread级notify等待 |
| HcommThreadNotifyRecordOnThread | task/dispatcher层 | Thread级notify记录(触发) |
| HcommRegOpInfo | communicator层 | 注册算子信息(异常诊断用) |
| HcommRegOpTaskException | communicator层 | 注册异常回调函数 |
| HcommProfilingInit/End | profiling子系统 | Device侧profiling生命周期 |
| HcommProfilingReportMainStreamAndFirst/LastTask | profiling子系统 | 主流+首尾task上报 |
| HcommProfilingReportDeviceHcclOpInfo | profiling子系统 | 算子附加信息上报 |

这些函数全部用 __attribute__((weak)) 声明(kernel_launch.cc:28-37)，
允许在没有hcomm链接时编译通过(在调用前检查nullptr)。

### AICPU的执行模型

从kernel_launch.cc代码可以推断出的AICPU执行模型:

1. **Thread模型**: AICPU有独立的Thread概念(ThreadHandle)，主thread + slave threads
   - V1: slaveThreadNum从AlgResourceCtx获取，threads数组紧跟在struct后面
   - V2: threads从AlgResourceCtxSerializable.threads vector获取

2. **Notify同步**: Thread间通过Notify机制同步
   - Host→Device: NotifyRecord(Host侧) → NotifyWait(Device侧) + timeout
   - Device→Host: NotifyRecord(Device侧) → NotifyWait(Host侧)
   - 超时机制: CUSTOM_TIMEOUT=1800秒(alg_param.h:50)

3. **BatchMode**: 通信操作通过BatchModeStart/End括号包裹
   - Start: 进入批量SQE模式(SQE不立即提交)
   - End: 退出批量模式(flush所有SQE)
   - 对应hcomm中DispatcherAiCpu的SQE缓存机制

4. **Stream**: kernel通过aclrtLaunchKernelWithConfig下发到stream执行
   - Host stream和AICPU thread之间需要Export/Import(跨引擎域)

5. **单block**: numBlocks固定为1, AICPU kernel始终单实例执行
   (与AIV的多核并行不同)

### load_kernel.cc — Kernel加载机制

LoadAICPUKernel()是Host侧代码(非Device侧):

```
GetKernelFilePath:
  1. 先查 ASCEND_HOME_PATH 环境变量
  2. 缺省 /usr/local/Ascend/cann/
  3. 追加 /opp/built-in/op_impl/aicpu/config/
  4. 加载 libscatter_aicpu_kernel.json

LoadAICPUKernel:
  1. 幂等检查: g_binKernelHandle != nullptr → 直接return
  2. LoadBinaryFromFile(jsonPath, CPU_KERNEL_MODE, 0, g_binKernelHandle)
  3. g_binKernelHandle保存加载结果
```

关键设计:
- g_binKernelHandle是全局变量(非thread_local): 所有线程共享一个handle
  (对比: examples/中用thread_local, 正式代码不用)
- 加载JSON不是SO: .json描述kernel元数据, Runtime据此加载.so
- .ini → .json转换: 构建时由parser_ini.py脚本完成
- 命名遗留: 虽然模块名叫"scatter_aicpu_kernel", 实际服务所有算子(op_common.cc:180注释确认)

### 异常诊断机制 (dfx/task_exception_fun.cc)

异常诊断仅在V1路径(非910_95)中使用:

#### ScatterOpInfo结构体
```cpp
struct ScatterOpInfo {
    char algTag[ALG_TAG_LENGTH];
    char commName[COMM_INDENTIFIER_MAX_LENGTH];
    void* inputPtr = nullptr;
    void* outputPtr = nullptr;
    u32 root = INVALID_VALUE_RANKID;
    u64 count;
    HcclDataType dataType;
    HcclCMDType opType = HcclCMDType::HCCL_CMD_INVALID;
};
```

结构体命名为"Scatter"但opType字段表明它被通用于所有算子类型。

#### CreateScatter()
从OpParam提取关键信息到ScatterOpInfo:
- 使用strncpy_s安全拷贝字符串
- CHK_PRT_RET检查返回值

#### GetScatterOpInfo() — 异常回调
当task异常时, hcomm调用此回调获取人可读的诊断信息:
```cpp
void GetScatterOpInfo(const void *opInfo, char *outPut, size_t size)
```
输出格式: "tag:xxx, group:xxx, count:xxx, dataType:xxx, opType:xxx, rootId:xxx, dstAddr:0xXXX, srcAddr:0xXXX."

### 条件编译

1. **AICPU_COMPILE宏**: 在CMakeLists.txt中scatter_aicpu_kernel目标定义`-DAICPU_COMPILE`
   - 影响channel.cc(8处): AICPU编译时跳过Host侧拓扑计算
   - 影响topo_match_*.cc(多处): AICPU编译时为空操作
   - kernel_launch.cc本身不使用此宏(它运行在AICPU上无需条件编译)

2. **DevType::DEV_TYPE_910_95**: 运行时分支(非编译时), 决定V1/V2路径

### 惯用法

1. **extern "C" unsigned int返回值**: kernel入口使用C ABI + unsigned int(0/1)
   而非HcclResult, 因为Runtime的kernel callback签名要求unsigned int返回

2. **weak符号注册hcomm回调**: 所有Hcomm*函数通过__attribute__((weak))声明
   + nullptr检查调用, 实现与hcomm的可选链接

3. **V1路径手动偏移取变长数据**:
   ```cpp
   ThreadHandle *ptr = reinterpret_cast<ThreadHandle *>(
       reinterpret_cast<u8 *>(resCtx) + sizeof(AlgResourceCtx));
   ```
   这是C风格的变长结构体惯用法: 固定header + 尾部变长数据区

4. **BatchMode括号**: HcommBatchModeStart/End成对出现, 类似事务begin/commit

5. **Notify Record+Wait对称**: Host Record→Device Wait, Device Record→Host Wait
   形成完整的同步握手

6. **RestoreVarData模式**: V2路径中4个RestoreVarData*函数结构一致:
   - 校验varMemSize == VECTOR_NUM * rankSize * sizeof(u64)
   - reinterpret_cast<u64*>(varData)
   - 按偏移恢复指针(counts, displs等)

7. **CHK_PRT_RET校验模式**: 参数校验 + 打印错误 + 提前返回, 与项目其他代码一致

8. **幂等加载**: LoadAICPUKernel内部检查g_binKernelHandle != nullptr防重复

### 反模式与Bug

#### AP-AICPU-1: GetScatterOpInfo日志缺少__func__
`task_exception_fun.cc:53`:
```cpp
HCCL_ERROR("%s strncpy_s fail, src size[%u], dst size[%u], sRet[%d]", strTmp.size(), size, sRet);
```
HCCL_ERROR的第一个%s应该传入__func__但实际传入了strTmp.size()(size_t类型,
与%s不匹配), 这是printf格式字符串bug, 会导致未定义行为或乱码。

正确做法:
```cpp
HCCL_ERROR("%s strncpy_s fail, src size[%zu], dst size[%zu], sRet[%d]", __func__, strTmp.size(), size, sRet);
```

#### AP-AICPU-2: dstAddr/srcAddr语义疑似颠倒
`task_exception_fun.cc:47-48`:
```cpp
ss << "dstAddr:0x" << std::hex << info->inputPtr << ", ";
ss << "srcAddr:0x" << std::hex << info->outputPtr << ".";
```
inputPtr标注为dstAddr, outputPtr标注为srcAddr — 这与直觉相反。
但考虑到某些集合操作(如reduce)中output确实是source(已聚合的数据来源),
这可能是命名歧义而非bug, 需要结合具体操作语义确认。

#### AP-AICPU-3: V2路径缺少异常诊断和Profiling
910_95 V2路径完全跳过了:
- CreateScatter + HcommRegOpInfo + HcommRegOpTaskException
- HcommProfilingInit/End + ReportMainStream + ReportDeviceOpInfo

对比V1路径有完整的异常诊断和profiling, V2路径在发生错误时无法获取算子现场信息,
profiling也无法覆盖Device侧执行。这可能是功能未完成的阶段性状态。

#### AP-AICPU-4: HcclLaunchAicpuKernel返回1而非错误码
Device侧kernel入口在所有错误分支都`return 1`, 丢失了具体的HcclResult错误码:
```cpp
if (CreateScatter(param, &opInfo) != HCCL_SUCCESS) {
    HCCL_ERROR("%s CreateScatter fail", __func__);
    return 1;  // 丢失具体错误码
}
```
共18处`return 1`, 调用方(Runtime)只能知道"失败了", 不知道失败原因。
这是因为Runtime kernel callback签名要求unsigned int, 但可以定义更丰富的错误码映射。

#### AP-AICPU-5: g_binKernelHandle全局变量无线程安全保护
`load_kernel.cc:17`:
```cpp
aclrtBinHandle g_binKernelHandle = nullptr;
```
LoadAICPUKernel中`if (g_binKernelHandle != nullptr)`检查和赋值没有任何同步原语,
多线程并发调用可能导致race condition(双重加载或使用半初始化的handle)。

对比: examples/中使用thread_local避免了这个问题, 但正式代码没有采用。

#### AP-AICPU-6: 环境变量调用模式
`load_kernel.cc:23-24`:
```cpp
char *getPath = getenv("ASCEND_HOME_PATH");
MM_SYS_GET_ENV(MM_ENV_ASCEND_HOME_PATH, getPath);
```
getenv先调用后又被MM_SYS_GET_ENV覆盖, 第一次调用结果被浪费。
且getenv在多线程环境下不是线程安全的(POSIX未保证)。

#### AP-AICPU-7: V1路径手动指针偏移脆弱
```cpp
ThreadHandle *threadHandlePtr =
    reinterpret_cast<ThreadHandle *>(reinterpret_cast<u8 *>(resCtx) + sizeof(AlgResourceCtx));
```
这假设AlgResourceCtx后面紧跟ThreadHandle数组, 没有任何编译时或运行时验证。
如果AlgResourceCtx布局变更(如添加新字段), 此处会静默读取错误数据。
V2使用AlgResourceCtxSerializable的正式序列化/反序列化避免了这个问题。

#### AP-AICPU-8: 命名遗留 "scatter" 贯穿整个模块
- 共享库名: libscatter_aicpu_kernel.so
- JSON: libscatter_aicpu_kernel.json
- INI: scatter_aicpu_kernel.ini
- 结构体: ScatterOpInfo
- 函数: CreateScatter, GetScatterOpInfo

但实际上此kernel服务所有算子(AllReduce/AllGather/AlltoAllV/Send/Recv等),
op_common.cc:180注释确认: "只实现了scatter的，先共用scatter的"。
这是早期scatter是唯一AICPU算子时的遗留命名。

### 编译模型

scatter_aicpu_kernel.so编译目标(CMakeLists.txt:214-316)包含:
- common/ 基础工具(6个文件)
- op_common/executor/ 完整执行器(6个文件)
- op_common/template/ 模板基础设施(7个文件, 含aicpu/kernel_launch.cc)
- op_common/topo/ 拓扑匹配(6个文件, 但AICPU_COMPILE下大部分为空操作)
- 13个算子的executor和aicpu template(约60个文件)

编译定义: `-DAICPU_COMPILE` (唯一特殊定义)

链接: ccl_kernel库

这意味着整个hccl的executor+template代码被编译了两次:
1. 主hccl库: 不定义AICPU_COMPILE, Host侧完整功能
2. scatter_aicpu_kernel.so: 定义AICPU_COMPILE, Device侧精简版

### Notable Patterns

1. **Host-Device同步协议的完整握手**:
   ```
   Host:  NotifyRecord(cpuTs → exportedAicpu, notifyNum)  // 告诉Device可以开始
   Host:  AicpuKernelLaunch(param)                        // 下发kernel到stream
   Host:  NotifyWait(cpuTs, 0, timeout)                   // 等Device完成
   Host:  SynchronizeStream                               // 同步stream

   Device: NotifyWait(thread, notifyNum, CUSTOM_TIMEOUT)  // 等Host通知
   Device: executor->Orchestrate(...)                     // 执行算法
   Device: NotifyRecord(thread, exportedThread, 0)        // 通知Host完成
   ```
   Host的NotifyRecord先于Launch执行, 这意味着当kernel被调度执行时,
   Notify已经ready, Device端Wait不会真正阻塞(在正常情况下)。

2. **双引擎线程域**: cpuTsThread和exportedAicpuTsThread的Export/Import
   揭示了昇腾芯片的多引擎架构 — CPU_TS(CPU Task Scheduler)和
   AICPU_TS(AICPU Task Scheduler)有独立的线程空间,
   跨引擎通信需要通过Export机制将句柄映射到对方域。

3. **序列化与反序列化的代际演进**:
   - V1(AlgResourceCtx): 固定header + 裸内存追加, 类似POD + flexible array member
   - V2(AlgResourceCtxSerializable): 正式的Serialize/DeSerialize, 支持vector等STL容器
   这反映了项目从C风格向C++风格的演进。

4. **ST测试中直接调用kernel**: test/st/hccl_proxy/aclrt_stub.cc将
   aclrtLaunchKernelWithConfig实现为直接调用HcclLaunchAicpuKernel(param),
   绕过Runtime调度, 实现无硬件的算法仿真测试。

5. **ALL_TO_ALL_V_VECTOR_NUM=4**: AlltoAllV需要4个per-rank数组
   (sendCounts, recvCounts, sdispls, rdispls),
   而ReduceScatterV和AllGatherV只需要2个(counts, displs)。
   这反映了MPI AlltoAllV的语义复杂度。

---

## 3.1.3 template/registry/ — 模板注册机制

### Module Overview

template/registry/提供两代模板注册工厂:
- V1 AlgTemplateRegistry: TemplateType枚举索引, vector存储, 仅scatter算子使用
- V2 InsAlgTemplateRegistry: string名索引, map存储, 仅DPU模板使用

总规模: 4文件, ~205行(含license头)。极其精简。

### Key Files

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| alg_template_register.h | 49 | AlgTemplateCreator typedef + DefaultTemplateCreator + AlgTemplateRegistry类 + REGISTER_TEMPLATE宏 |
| alg_template_register.cc | 59 | Singleton + Register(范围检查+去重+mutex) + GetAlgTemplate(范围检查+创建) |
| alg_v2_template_register.h | 50 | InsAlgTemplateCreator typedef + DefaultTemplateCreatorV2 + InsAlgTemplateRegistry类 + REGISTER_TEMPLATE_V2宏 |
| alg_v2_template_register.cc | 47 | Singleton + Register(去重+mutex) + GetAlgTemplate(map查找+创建) |

### 注册机制: 静态初始化 + 宏

两代Registry共享相同的Self-Registration Factory模式:

```cpp
// alg_template_register.h:43-47 — V1宏
#define REGISTER_TEMPLATE_HELPER(ctr, type, algTempBase)       \
    static HcclResult g_func_##algTempBase##_##ctr             \
        = AlgTemplateRegistry::Instance().Register(type, DefaultTemplateCreator<algTempBase>)

#define REGISTER_TEMPLATE(type, algTempBase) REGISTER_TEMPLATE_HELPER(__COUNTER__, type, algTempBase)
```

设计要点:
1. `__COUNTER__`生成唯一变量名(允许同文件多次注册)
2. `static HcclResult`全局变量 — 初始化时执行Register，返回值被丢弃
3. `DefaultTemplateCreator<P>()`使用`static_assert(std::is_base_of<AlgTemplateBase, P>)`编译时类型检查
4. 使用`new (std::nothrow)`分配(不抛异常)

V2宏结构完全对称(REGISTER_TEMPLATE_V2)，仅将TemplateType→string, AlgTemplateBase→InsAlgTemplateBase。

### V1: AlgTemplateRegistry — 枚举索引 + vector存储

```cpp
// alg_template_register.cc:23
tempCreators_.resize(TemplateType::TEMPLATE_CUSTOM_MAX_NUM, nullptr);  // 2000槽位
```

TemplateType枚举(定义在alg_template_base.h:48-62):
- 0~5: 内置(SCATTER_MESH/RING/NB/NHR/RING_DIRECT/REDUCE_SCATTER_MESH)
- 100~101: 特殊(HOST_DPU/LOCAL_REDUCE)
- 102(NATIVE_MAX_NUM): 内置边界
- 1000(CUSTOM_BEGIN): 自定义起始
- 2000(CUSTOM_MAX_NUM): 自定义最大
- 102~999之间是无效gap(Register/Get都做显式gap检查)

O(1)查找。预分配2000个槽位中仅使用8个(利用率0.4%)。

Register加mutex锁，不允许覆盖(已存在→HCCL_ERROR+返回错误)。
GetAlgTemplate不加锁(依赖"启动时注册完毕"前提)。

实际使用(Grep验证): 仅scatter的5个template注册，8个查找点(scatter_executor_base.cc和scatter_ring_executor.cc)。

### V2: InsAlgTemplateRegistry — string索引 + map存储

```cpp
// alg_v2_template_register.h:40
std::map<std::string, InsAlgTemplateCreator> tempCreators_;
```

O(log N)查找。无预分配。
Register: mutex锁 + find检查(map中重复则ERROR)。
GetAlgTemplate: 不加锁，find→nullptr检查→调用Creator。

实际使用(Grep验证): 仅2个DPU模板注册:
- ins_temp_reduce_scatter_mesh_1d_dpu.cc: REGISTER_TEMPLATE_V2("InsTempReduceScatterMesh1dDpu", ...)
- ins_temp_all_gather_nhr_dpu.cc: REGISTER_TEMPLATE_V2("InsTempAllGatherNHRDPU", ...)

### 与executor/registry/的关系和区别

executor/registry/有对等的两组注册工厂:
- CollAlgExecRegistry(V1): string→ExecutorBase, unordered_map
- CollAlgExecRegistryV2(V2): (HcclCMDType, string)→InsCollAlgBase, 双层map

核心区别:

| 维度 | template/registry V1 | template/registry V2 | executor/registry V1 | executor/registry V2 |
|------|---------------------|---------------------|---------------------|---------------------|
| 索引 | TemplateType(enum) | string | string | (HcclCMDType, string) |
| 存储 | vector[2000] | ordered map | unordered_map | map<map<>> |
| 查找 | O(1) | O(log N) | O(1)平均 | O(log M * log N) |
| 注册数 | 5 | 2 | 4 | 60+ |
| 返回 | unique_ptr | unique_ptr | unique_ptr | shared_ptr |
| 去重 | HCCL_ERROR | HCCL_ERROR | HCCL_WARNING | HCCL_ERROR |
| 宏种类 | 1种 | 1种 | 1种(+1 helper) | 7种(支持0~4模板参数) |

两级工厂关系:
- Executor是高层编排者(决定多步骤执行策略)
- Template是底层算法实现(一个具体通信步骤的执行)
- 一个Executor内部可通过GetAlgTemplate动态获取Template来组合执行
  (如AllReduce executor获取ReduceScatter+AllGather两个template)

executor/registry V2有7种注册宏的原因:
支持InsCollAlgBase模板类(C++意义上的模板)的多种参数组合注入:
```
REGISTER_EXECUTOR_IMPL — 无模板参数
REGISTER_EXECUTOR_IMPL_NO_TOPOMATCH — 1模板参数(InsAlgTemplate)
REGISTER_EXEC_V2 — 2模板参数(AlgTopoMatch, InsAlgTemplate)
REGISTER_EXECUTOR_BY_TOPO — 1模板参数(AlgTopoMatch)
REGISTER_EXECUTOR_BY_TWO_TEMPS — 3模板参数(AlgTopoMatch, InsAlgTemplate0, InsAlgTemplate1)
REGISTER_EXECUTOR_BY_FOUR_TEMPS — 5模板参数(AlgTopoMatch + 4个InsAlgTemplate)
```
template/registry的宏较简单因为template类层次更平坦(不需要模板参数注入)。

所有4个Registry的Get方法都不加锁 — 共同假设"注册在静态初始化时完成，查找在运行时执行"。

### 反模式

1. GetAlgTemplate不加锁:
   虽然"注册在启动时完成"是合理假设，但缺乏文档化的线程安全契约。
   如果静态初始化顺序跨编译单元不确定(SIOF)，可能出现竞态。

2. V1的gap检查代码在Register和GetAlgTemplate中完全重复(~4行):
   应提取为私有方法IsValidType(type)。

3. V1预分配2000槽位仅用8个:
   空间浪费。但vector<function>的nullptr元素开销极小(每个16~32字节)，
   总计~48KB，可以忽略。

---

## 3.1.3 template/wrapper/ — 数据传输包装函数

### Module Overview

wrapper/提供跨rank数据传输的高层包装函数(自由函数，非类方法)。
是Facade/Helper模式: 将底层hcomm通信原语(HcommWriteOnThread/ReadOnThread/ChannelNotifyXxx)
封装为面向"数据切片(DataSlice)传输"的高层操作，供各具体template的KernelRun实现调用。

### Key Files

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| alg_data_trans_wrapper.h | 65 | 18个AICPU wrapper函数声明(全部带ThreadHandle参数) |
| alg_data_trans_wrapper.cc | 606 | 18个函数实现 + AicpuReduceTemplate模板函数 |
| dpu_alg_data_trans_wrapper.h | 26 | 3个DPU wrapper函数声明(无ThreadHandle) |
| dpu_alg_data_trans_wrapper.cc | 44 | 仅SendRecvWrite实现(其余2个仅声明无实现) |

被60个文件include(Grep验证: alg_data_trans_wrapper.h被include 60处)。

### 函数矩阵: 三个正交维度的组合

12个远端传输函数 = 3(角色) x 2(模式) x 2(是否Reduce):

| 角色\模式+Reduce | Write | Write+Reduce | Read | Read+Reduce |
|-----------------|-------|-------------|------|-------------|
| Send(发方) | SendWrite | SendWriteReduce | SendRead | SendReadReduce |
| Recv(收方) | RecvWrite | RecvWriteReduce | RecvRead | RecvReadReduce |
| SendRecv(双向) | SendRecvWrite | SendRecvWriteReduce | SendRecvRead | SendRecvReadReduce |

加6个本地/同步操作:
- LocalCopy / LocalReduce / LocalCopySlices — 本地数据操作
- PreSyncInterThreads / PostSyncInterThreads — 主从线程同步
- AicpuReduce + AicpuReduceTemplate<T> — CPU侧reduce(64位/prod fallback)

### 通信协议: Notify驱动的两阶段同步

以SendWrite + RecvWrite为例:
```
Sender(调用SendWrite)                  Receiver(调用RecvWrite)
  Wait(sendCh, ACK)       <-- Record(recvCh, ACK)      // Phase 1: 接收方ready
  for slice: Write(data)  --> ...                       // 数据传输
  Record(sendCh, DATA_SIGNAL) --> Wait(recvCh, DATA_SIGNAL) // Phase 2: 发送完成
```

SendRecv(双向)增加了双向ACK和双向DATA_SIGNAL:
```
Rank0(调SendRecvWrite)              Rank1(调SendRecvWrite)
  Record(rxCh, ACK)    -->  Wait(txCh, ACK)     // 互相确认ready
  Wait(txCh, ACK)      <--  Record(rxCh, ACK)
  Write(data→Rank1)    -->  Write(data→Rank0)   // 双向写
  Record(txCh, SIGNAL) -->  Wait(rxCh, SIGNAL)  // 互相确认完成
  Wait(rxCh, SIGNAL)   <--  Record(txCh, SIGNAL)
```

Read语义与Write语义的角色反转:
- Write: 发方主动写数据到对方(RDMA Write语义)
- Read: 收方主动从对方读数据(RDMA Read语义)
但同步协议不变(都是ACK+DATA_SIGNAL两阶段)。

代码中的中文注释精确解释了SendRecv的notify语义(alg_data_trans_wrapper.cc:49-53)。

### Reduce路径分叉: 硬件限制导致的CPU fallback

LocalReduce(line 381-384)中:
```cpp
if (dataType == HCCL_DATA_TYPE_INT64 || dataType == HCCL_DATA_TYPE_UINT64 ||
    dataType == HCCL_DATA_TYPE_FP64 || reduceOp == HcclReduceOp::HCCL_REDUCE_PROD) {
    CHK_RET(AicpuReduce(thread, srcSlice, dstSlice, dataType, reduceOp));
```

64位类型(INT64/UINT64/FP64)和PROD操作走CPU(AicpuReduceTemplate<T>)逐元素循环，
其余走HcommLocalReduceOnThread(硬件DMA InlineReduce)。
原因: 昇腾SDMA/DMA InlineReduce仅支持16/32位类型的sum/max/min。

AicpuReduceTemplate<T>是纯host循环(无SIMD优化):
```cpp
for (u64 i = 0; i < count; ++i) {
    switch (reduceOp) {
        case SUM:  *(dst+i) = srcData + dstData; break;
        case PROD: *(dst+i) = srcData * dstData; break;
        case MAX:  *(dst+i) = std::max(srcData, dstData); break;
        case MIN:  *(dst+i) = std::min(srcData, dstData); break;
    }
}
```

### LocalCopySlices: 连续切片合并优化

关键性能优化 — 合并连续DataSlice为单次DMA传输:
```cpp
if (IsContinuousSlice(srcSlices[sliceIdx + 1], tmpSrcSlice) &&
    IsContinuousSlice(dstSlices[sliceIdx + 1], tmpDstSlice)) {
    // 连续: 扩展当前切片size
    u64 newTmpSize = tmpSrcSlice.size_ + srcSlices[sliceIdx + 1].size_;
} else {
    // 不连续: 执行当前切片拷贝 + 开始新片段
    HcommLocalCopyOnThread(thread, dst, src, tmpSrcSlice.size_);
}
```

IsContinuousSlice: 相同base addr + offset紧邻(nxt.offset == curr.offset + curr.size)。
减少DMA指令数，对小切片密集场景性能收益显著。

### AICPU vs DPU wrapper差异

| 维度 | AICPU版 | DPU版 |
|------|---------|-------|
| ThreadHandle | 每个函数都带 | 无(DPU单线程模型) |
| 底层API | HcommXxxOnThread | HcommXxx (直接版) |
| 函数数量 | 18个全部实现 | 3个声明，仅1个(SendRecvWrite)实现 |
| slice size=0检查 | 有(HCCL_WARNING+continue) | 无 |
| 数据传输 | Write后攒到最后统一signal | HcommWriteWithNotifyNbi(每次写自带notify) |
| 完成同步 | ACK + DATA_SIGNAL | ACK + DATA_SIGNAL + FIN_ACK + HcommFlush |
| 开发状态 | 成熟 | 早期("! 已编码完成"注释) |

DPU版SendRecvWrite (dpu_alg_data_trans_wrapper.cc:17-41):
- 每slice写入使用HcommWriteWithNotifyNbi(Nbi=Non-blocking immediate,写+notify一体)
- 每次写后立即Wait(而非AICPU版的攒到最后)
- 额外FIN_ACK + HcommFlush: DPU需要显式flush确保PCIe ordering

### 发现的Bug和反模式

#### B1: CHK_PRT_RET日志参数错误(3处重复传播)

alg_data_trans_wrapper.cc:101-106 SendWriteReduce:
```cpp
HCCL_ERROR("...src slice count [%u] is not mate to src slice size [%u]...",
    srcSlice.size_, srcSlice.size_),  // BUG: 第1个应为srcSlice.count_
```
同一错误在dst检查中也存在(line 108-113)。
且完整复制到SendRecvWriteReduce(line 159-164)、RecvReadReduce(line 280-285)、
SendRecvReadReduce(line 329-334) — 典型copy-paste错误传播。

#### B2: 日志tag不一致(遗漏重构)

- LocalReduce line 391: `[InsCollAlgFactory]` — 应为[AlgDataTransWrapper]
- LocalCopySlices line 410,426: `[InsCollAlgFactory] [AlgDataTrans]` — 混合tag
- 其余函数: `[AlgDataTransWrapper]` (正确)
说明这些函数最初定义在InsCollAlgFactory中，提取为独立wrapper时遗漏了日志tag更新。

#### B3: Reduce日志中函数名错误

RecvReadReduce(line 281): 日志写`SendWriteReduce`而非`RecvReadReduce`
SendRecvReadReduce(line 330): 日志写`SendWriteReduce`而非`SendRecvReadReduce`
copy-paste遗留。

#### B4: PostSyncInterThreads日志引用错误函数名

PostSyncInterThreads(line 502-509)中的HCCL_ERROR写`[PreSyncInterThreads]`:
```cpp
HCCL_ERROR("[AlgDataTransWrapper] [PreSyncInterThreads] subThreads size: ...")
```
应为`[PostSyncInterThreads]`。

#### B5: 参数名拼写错误(snedInfo)

alg_data_trans_wrapper.h中6处将"send"拼成"sned":
```cpp
HcclResult SendWrite(const DataInfo &snedInfo, const ThreadHandle &thread);     // "sned" → "send"
HcclResult SendWriteReduce(const DataReduceInfo &snedInfo, const ThreadHandle &thread);
...
```
.h中写"snedInfo"，.cc中写"sendInfo"(参数名不匹配，但C++允许)。

dpu_alg_data_trans_wrapper.cc:30也有拼写: `dstSlcie` → `dstSlice`。

#### B6: s8/u8类型不一致

远端传输函数用`static_cast<s8*>(addr) + offset`做地址偏移，
本地操作(LocalCopy/LocalReduce)用`static_cast<u8*>(addr) + offset`。
虽然指针算术不受符号影响，但风格不一致。

#### B7: int/u32比较

多处循环`for (int i = 0; i < sliceNum; i++)`其中sliceNum是u32。
signed/unsigned比较可能触发编译器warning(虽然实际数值不会溢出)。

### 设计模式

1. Facade/Helper(非GoF Decorator/Adapter):
   纯自由函数(无类)封装底层原语为高层操作。无状态，无继承。

2. Composable Synchronization Protocol:
   3x2x2=12种组合不是通过继承/模板生成，而是手写12个函数。
   优点: 每个函数逻辑清晰，易于独立调试。
   缺点: 大量sync代码重复(如ACK+DATA_SIGNAL在所有函数中都有)。

3. Strategy(隐式): LocalReduce中CPU vs 硬件Reduce路径选择。

### 惯用法

1. `CHK_RET(static_cast<HcclResult>(HcommXxxOnThread(...)))` — 所有hcomm C API调用模式
2. `static_cast<void*>(static_cast<s8/u8*>(addr) + offset)` — 地址偏移计算
3. `CHK_PRT_RET(condition, HCCL_ERROR(...), error_code)` — 前置条件检查
4. `if (srcSlice.size_ == 0) { HCCL_WARNING("..."); continue; }` — size=0跳过不报错
5. Notify Record/Wait必须成对出现
6. CUSTOM_TIMEOUT用于所有Wait操作(全局超时常量)

### 条件编译

4个wrapper文件中无条件编译(#ifdef仅用于include guard)。
与其他template子目录(aiv/ccu/)大量使用`#ifndef AICPU_COMPILE`形成对比。
wrapper函数同时适用于Host和AICPU编译路径。

---

## 3.1.3(续) template/aiv/, ccu/, dpu/ -- 设备侧子目录深度分析

### 概述

template/下的aiv/, ccu/, dpu/三个子目录分别对应三种设备执行引擎的kernel/微码编程基础设施。
它们在执行模型、编程抽象层次、与hcomm的依赖关系上有根本性差异。

文件统计:
- aiv/: 4个.h + 1个.cc = 5文件, 1045行 (设备侧头文件 + Host侧Launch工具)
- ccu/: 2个.h + 2个.cc = 4文件, 1017行 (Host侧编程框架)
- dpu/: 1个.h + 1个.cc = 2文件, 64行 (极简DPU入口)

### AIV子目录 (aiv/)

#### Key Files

| 文件 | 行数 | 角色 |
|------|------|------|
| aiv_communication_base_v2.h | 460 | 设备侧AIV kernel基类(V2版) |
| aiv_kernel_def.h | 140 | Host侧kernel注册表(名称x数据类型x算子) |
| hccl_aiv_utils.h | 155 | Host侧AIV工具函数声明和数据结构 |
| hccl_aiv_utils.cc | 263 | Host侧kernel注册/Launch/tag管理实现 |
| aiv_interface/sync_interface.h | 127 | 设备侧GM信号同步原语库 |

#### AivCommBase -- 设备侧AIV kernel基类(V2)

Init方法接受25个参数(GM地址表 + rank信息 + 数据参数 + counter + double buffer),
初始化GM_IN[]/GM_OUT[]地址表(从buffIn IPC共享区读取)和TOPO_[]拓扑信息。
所有成员全部public(设备侧代码无封装需求)。

关键方法:
- Record/WaitFlag: GM flag轮询同步协议(tag写入/busy-poll读取)
- BarrierAll: SyncAll(核间) + block 0做跨rank Record/WaitFlag
- SyncCoreAll: 等待所有MAX_NUM_BLOCKS(48)个block的flag
- DataCopyGM2UB/DataCopyUB2GM: 调用copy_gm_to_ubuf_align_v2 / copy_ubuf_to_gm_align_v2 intrinsic
- CpGM2GM: GM->UB->GM循环搬运(UB做中转, maxCountPerLoop按190KB/95KB限制)
- SetAtomicOp: SUM/MAX/MIN/None原子模式切换

AscendC SDK使用:
- TPipe: 流水线管理器(pipe_barrier)
- TBuf<>: UB缓冲区(localFlagBuf, 1024B)
- TQueBind<VECIN, VECOUT, 1>: 数据搬运队列(inOutQue, 190KB)
- GlobalTensor/LocalTensor: GM/UB张量抽象
- pipe_barrier(PIPE_ALL): 全流水线同步(17次出现)

硬件约束常量:
```
UB_MAX_DATA_SIZE      = 190KB    UB_DB_DATA_BATCH_SIZE = 95KB
UB_ALIGN_SIZE         = 32B      MAX_RANK_SIZE         = 8
MAX_NUM_BLOCKS        = 48       FLAG_SIZE             = 32B
ATOMIC_FLAG_SIZE      = 512B     AIV_FLAG_BUFFER_SIZE  = 3MB
MaxBufferSize         = 200MB    IPC_SYNC_OFFSET       = 500KB
BARRIER_OFFSET        = 900KB    SYNC_CORE_OFFSET      = 950KB
```

数据类型支持:
- Atomic(6种): float, half, int16_t, int32_t, int8_t, bfloat16_t
- Copy(11种): 以上 + uint16_t, uint32_t, uint8_t, uint64_t, int64_t

#### sync_interface.h -- GM信号同步原语

6个自由函数, 全部__aicore__ inline:
- SetSignalValue: UB->GM写入信号值(32字节DataCopy)
- AddSignalValue: 原子加(SetAtomicAdd + DataCopy)
- WaitSignalValue: busy-poll直到等于预期值
- WaitSignalGEValue: busy-poll直到大于等于预期值
- WaitSignalNotEqValue: busy-poll直到不等于预期值
- SetFlagBatchValue: 批量设置flag(一次DataCopy写多个flag)

所有Wait函数都是无超时的while(true)循环。
被注释掉的SyncFunc<HardEvent>暗示最初使用硬件事件, 后改为pipe_barrier(PIPE_ALL)。

#### Host侧Kernel管理 (hccl_aiv_utils.h/.cc)

AivOpArgs: 算子属性结构(cmdType, stream, counter, 25+字段)
AivSuperKernelArgs: SuperKernel模式参数(tag + clearEnable)
AivExtraKernelArgs: Launch时实际传递的POD结构(内部类型)

两阶段模型:
1. RegisterKernel: 从.o文件加载二进制 -> aclrtBinaryGetFunction提取函数句柄
   -> 存入g_aivFuncMap[合成key]。lock_guard + g_init保证全局单次。
2. ExecuteKernelLaunch: 组装AivExtraKernelArgs -> aclrtLaunchKernelWithHostArgs
   Launch配置: SCHEM_MODE=1, TIMEOUT=CUSTOM_TIMEOUT*1e6, ENGINE_TYPE=AIV

函数查找key: (cmdType<<20 + dataType)<<20 + argsType, 用reinterpret_cast<s8*>做map key。

Tag管理(GetAivCountTag): per-comm-per-rank递增, 到1000重置为1。
清空Buffer(ClearAivSyncBuf): aclrtMemcpy(D2D)从清零模板区到flag区。

#### aiv_kernel_def.h -- 注册表

8种算子80个kernel注册项, 统一组织在g_aivKernelInfoMap中。
每种算子一个binary(.o文件, 如hccl_aiv_all_reduce_op_910_95.o)。
问题: 所有static变量(17个vector+string)定义在header中。

疑似bug: allgather/broadcast/alltoall/alltoallv/scatter中INT64/UINT64映射交叉(5处):
```cpp
{"aiv_all_gather_uint64_t", HCCL_DATA_TYPE_INT64},   // uint64映射到INT64
{"aiv_all_gather_int64_t", HCCL_DATA_TYPE_UINT64},    // int64映射到UINT64
```

#### 与hcomm对应模块的关系

hccl aiv_communication_base_v2.h(460行, V2) vs hcomm aiv_communication_base.h(1052行, V1):
- V2更简洁: 直接AscendC SDK API, 无V1的52+参数宏和多层if-else分派树
- V2不依赖hcomm, V1不依赖AscendC SDK
- 二者共享GM flag轮询同步模型, 但V2的flag布局更简化
- V1有更丰富的同步原语(Record1vN/WaitNv1等)

#### 惯用法

AIV-ID-1: __aicore__ inline标注(32处)
AIV-ID-2: pipe_barrier(PIPE_ALL)(17处)
AIV-ID-3: TPipe数据搬运4步: AllocTensor -> EnQue -> DeQue -> FreeTensor
AIV-ID-4: 宏生成参数列表: KERNEL_ARGS_DEF / KERNEL_ARGS_CALL / KERNEL_CLASS_INIT
AIV-ID-5: X-macro类型列表: AIV_ATOMIC_DATA_TYPE_DEF(func) / AIV_COPY_DATA_TYPE_DEF(func)

#### 反模式

AIV-AP-1: WaitFlag死循环无超时(5处while(true) busy-poll)
AIV-AP-2: Header中static变量(aiv_kernel_def.h 17个static, ODR问题)
AIV-AP-3: INT64/UINT64映射交叉(5处)
AIV-AP-4: 合成指针做map key(reinterpret_cast<s8*>整数值)
AIV-AP-5: 全局可变状态(4个static全局, GetKernelFunc未加锁)
AIV-AP-6: CeilDiv(a,0)返回a(掩盖上层bug)
AIV-AP-7: using namespace AscendC/std在global scope

---

### CCU子目录 (ccu/)

#### Key Files

| 文件 | 行数 | 角色 |
|------|------|------|
| ccu_kernel_utils.h | 44 | 工具函数声明 |
| ccu_kernel_utils.cc | 166 | bit打包工具 + signature生成 + 类型映射 |
| ccu_kernel_alg_base.h | 104 | hccl侧CCU kernel算法基类 |
| ccu_kernel_alg_base.cc | 703 | GroupOp高阶操作(Broadcast/Reduce/Copy) |

#### CcuKernelAlgBase -- 继承hcomm::CcuKernel

继承链: CcuRep::CcuRepContext(hcomm) -> CcuKernel(hcomm) -> CcuKernelAlgBase(hccl)

内部结构:
- GroupOpConfig: { msInterleave, loopCount, memSlice }
- GroupOpSizeResource: { vector<CompletedEvent>, vector<CcuBuf>, vector<Executor> }
- GroupOpSize: { Variable addrOffset, loopParam, parallelParam, residual }

核心方法:
- AllocGoResource: 按parallelDim和msPerLoop分配Executor/Event/Buf (幂等, 首次分配)
- CalGoSize(size): 将数据分为m(整块)/n(部分loop)/p(尾块), 返回{offset, loopIterNum, loopExtendNum, tailSize}
- GroupBroadcast: src -> LocalCopyNb -> buf -> WriteNb -> remote dst (+ 本地dst)
- GroupReduce: remote src -> ReadNb -> bufs -> LocalReduceNb -> LocalCopyNb -> dst
- GroupCopy: src -> LocalCopyNb -> buf -> LocalCopyNb -> dst
- 每种操作都有WithoutMyRank变体(不含本rank的本地拷贝)
- LoopGroup: 将Loop组合为并行LoopGroup(由CCU硬件调度)

#### CCU编程模型

CCU是"编译器式"编程: Host侧代码构造微码IR, 非运行时执行。

关键特征:
1. CreateVariable/CreateLocalAddr等 -- 分配微码变量/地址(非C++变量)
2. CCU_IF -- 生成条件微码(非C++分支), 10次出现
3. Loop/LoopBlock/LoopGroupCall -- 硬件循环控制
4. CompletedEvent.mask位域 -- 细粒度异步完成等待
5. 双Loop ping-pong -- 所有操作固定2个Loop交替使用CcuBuf

三层数据切分:
- m = size / (loopCount * memSlice) -- LoopGroup0处理(CCU_IF loopParam != 0)
- n = 余量中能被memSlice整除的部分 -- LoopGroup1处理(CCU_IF parallelParam != 0)
- p = 最终尾块 -- 随n一起在LoopGroup1中处理

硬件约束:
- MS = 4KB (CcuRep::CCU_MS_SIZE)
- Loop最多4096次迭代(12bit)
- LoopGroup默认8个并行loop
- UB_MAX_TRANS_SIZE = 256MB

#### ccu_kernel_utils

bit打包工具:
- SetBits(start, end): 生成位掩码
- GetLoopParam: 将ctxId(8bit) + gsaOffset(32bit) + loopIterNum(13bit)打包到uint64_t
- GetParallelParam: repeatNum(7bit) + repeatLoopIndex(7bit) + totalLoopNum(7bit)
- GetOffsetParam: gsaOffset(32bit) + msOffset(11bit) + ckeOffset(10bit)
- GetExpansionParam: expansionNum编码到Bit[53-54]

GenerateCcuKernelSignature: 生成kernel缓存key,
追加: name + (reduceType + dataType + outputType if reduce类) + (root if rooted) + commSizes

GetReduceTypeStr: dataType + opType -> "fp32_sum"等字符串

GetReduceExpansionNum: 低精度(FP8/INT8)默认输出FP32, expansion = sizeof(output)/sizeof(input)

DataTypeSizeGet: 查SIZE_TABLE[type]

#### 与hcomm的关系

CcuKernel(hcomm/pkg_inc/hcomm/ccu/ccu_kernel.h)提供底层:
- Algorithm() = 0, GeneArgs() = 0 (纯虚, 子类必须实现)
- WriteNb/ReadNb/LocalCopyNb/LocalReduceNb (6种重载)
- WaitEvent/RecordEvent, NotifyRecord/NotifyWait
- CreateVariable/CreateLocalAddr/CreateRemoteAddr/CreateCcuBuf/CreateExecutor
- Load/LoadVariable/StoreVariable
- Loop/Func (控制流)
- GeneTaskParam (生成task参数)

hcomm还提供CcuKernelSignature类:
用ostringstream拼接, 实现operator== + hash + Describe, 用于kernel缓存key。

hccl中66个文件引用CcuKernelAlgBase, 覆盖7种算子。

#### 惯用法

CCU-ID-1: 双Loop ping-pong(5个CreateMultiOp方法)
CCU-ID-2: registeredLoop集合去重(set<string>)
CCU-ID-3: CreateBlock*返回vector包装(类型安全)
CCU-ID-4: CCU_IF条件微码(10次)
CCU-ID-5: Bit打包参数(constexpr位宽)
CCU-ID-6: 低精度expansion(FP8->FP32, dst.addr累加N次)
CCU-ID-7: using基类方法防遮蔽(2处)

#### 反模式

CCU-AP-1: reinterpret_cast<RemoteAddr*>(&localAddr)(7处)
CCU-AP-2: 哨兵值0xFFFFFFFF做"未配置"标记(应optional)
CCU-AP-3: HCCL_ERROR后返回SUCCESS(语义矛盾, L277)
CCU-AP-4: 代码重复(6对Xxx vs XxxWithoutMyRank方法)
CCU-AP-5: static局部map(GetReduceTypeStr)

---

### DPU子目录 (dpu/)

#### Key Files

| 文件 | 行数 | 角色 |
|------|------|------|
| kernel_launch.h | 20 | 单函数声明: int32_t HcclLaunchDPUKernel(uint64_t ptr, int32_t size) |
| kernel_launch.cc | 44 | 共享内存反序列化 + template查找 + DPUKernelRun |

#### 执行流程

1. 参数校验(ptr==0 || size<=0)
2. reinterpret_cast<char*>(ptr) -> vector<char>全量拷贝
3. DPURunInfo::DeSerialize(恢复templateName + tempAlgParams + channels + myRank + subCommRanks)
4. InsAlgTemplateRegistry::Instance().GetAlgTemplate(name) -> shared_ptr
5. templateIns->DPUKernelRun(params, channels, myRank, subCommRanks)

目前仅2个DPU算法模板:
- ins_temp_reduce_scatter_mesh_1d_dpu.cc
- ins_temp_all_gather_nhr_dpu.cc

调用方: op_common.cc中HcclLaunchDPUKernel被条件调用。

#### 反模式

DPU-AP-1: reinterpret_cast + 全量vector拷贝(ptr无效时崩溃)
DPU-AP-2: 返回int32_t而非HcclResult(需要static_cast)

---

### 三种设备执行模型对比

| 维度 | AIV | CCU | DPU |
|------|-----|-----|-----|
| 硬件 | AI Vector Core | Collective Comm Unit | Data Processing Unit |
| 代码位置 | 设备侧(AIV核心执行) | Host侧(构造微码IR) | DPU芯片(复用NPU模板) |
| 编程模型 | 显式GM<->UB搬运+SIMD | 声明式IR->微码->硬件 | 共享内存序列化->模板委托 |
| 同步机制 | GM flag轮询(busy) | CompletedEvent(硬件) | 共享内存+轮询 |
| buffer | UB 190KB/核 | MS 4KB/slice | 复用NPU |
| 错误处理 | 无 | CHK_RET | int32_t |
| hcomm依赖 | 独立 | 继承CcuKernel(深) | 无直接依赖 |
| 代码量 | 1045行 | 1017行 | 64行 |
| 算子覆盖 | 8种/80 kernel | 7种/66文件 | 2种 |
| 适用场景 | 小数据 | 大数据 | 跨节点卸载 |

关键洞察:

1. AIV+CCU互补: AIV小数据(UB高带宽低容量), CCU大数据(专用DMA+硬件loop)。
   selector 5路优先级: DPU > CCU_MS > CCU_SCHED > AIV > AICPU。

2. CCU的"编译器"模型: Host代码"构造程序"而非"运行程序",
   CCU_IF是硬件条件执行。需同时理解C++构造语义和微码执行语义。

3. DPU极简: 64行入口函数, 实际算法复用NPU模板, 仅2个实现。

4. hcomm依赖光谱: CCU(137次CcuRep::引用, 继承) > AIV(独立) > DPU(无依赖)。

5. templateRankSize_三种计算方式:
   AICPU(乘积=逻辑rank) / AIV(第一维=同层rank) / CCU(求和=物理die)。

---

## 3.1.3 综合: template/ 模块量化数据与惯用法总结

### Grep 量化统计

模块规模: 35个源文件(19 .h + 16 .cc), 共6199行代码。

| 模式 | 次数 | 文件数 | 密度(次/千行) | 解读 |
|------|------|--------|--------------|------|
| CHK_RET | 85 | 9 | 13.7 | 高防御性，集中在wrapper(49次)和base(11次) |
| virtual | 33 | 5 | 5.3 | 纯基类层，定义接口 |
| override | 0 | 0 | 0 | 子类不在此模块(在各算子目录) |
| reinterpret_cast | 38 | 9 | 6.1 | 底层硬件交互(kernel_launch 9次, sync_interface 7次) |
| static_cast | 125 | 8 | 20.2 | 安全转换优先，wrapper(84次)最密集 |
| HCCL_ERROR | 80 | 16 | 12.9 | 日志64%为ERROR，强调错误可观测 |
| HCCL_INFO | 18 | 6 | 2.9 | |
| HCCL_WARNING | 14 | 4 | 2.3 | |
| HCCL_DEBUG | 13 | 7 | 2.1 | |
| unique_ptr | 7 | 5 | 1.1 | Registry管理对象生命周期 |
| shared_ptr | 1 | 1 | 0.2 | 几乎不用共享所有权 |
| new/delete(裸) | 0 | 0 | 0 | 完全RAII化 |
| TODO/FIXME/HACK | 0 | 0 | 0 | 零技术债标记 |
| REGISTER | 4 | 2 | 0.6 | V1+V2两套注册宏 |
| #ifdef(业务) | 0 | 0 | 0 | 平台无关抽象层 |

关键比率:
- static_cast : reinterpret_cast = 3.3 : 1 (优先安全转换)
- unique_ptr : shared_ptr = 7 : 1 (强调独占所有权)
- ERROR : 其他日志 = 80 : 45 (错误路径可观测优先)

### 跨子模块惯用法(10条)

TPL-ID-1: CHK_RET 100%覆盖
  所有非trivial函数调用都用CHK_RET包裹，wrapper层49次最密集。
  出处: alg_data_trans_wrapper.cc 全文, alg_template_base.cc 全文

TPL-ID-2: 四基类独立不继承
  AlgTemplateBase / InsAlgTemplateBase / AivAlgTemplateBase / CcuAlgTemplateBase
  四个处理器基类完全独立(无公共父类)。
  原因: 不同处理器的资源模型差异太大(SDMA stream vs AICPU SQE vs AIV UB vs CCU microcode)。
  代价: CalcRes/KernelRun/Describe等接口签名重复。

TPL-ID-3: Describe() 统一纯虚
  V2/AIV/CCU三个基类都要求 `virtual std::string Describe() const = 0`。
  用于日志和调试，子类必须实现自描述。V1没有此约束(设计较早)。

TPL-ID-4: V2默认返回E_INTERNAL(安全)
  V2/AIV/CCU基类的未override方法默认返回HCCL_E_INTERNAL + HCCL_ERROR日志。
  V1默认返回HCCL_SUCCESS(危险: 子类遗漏override时静默成功)。

TPL-ID-5: Self-Registration Factory(双代)
  V1: TemplateType枚举索引(0~2000), vector存储, O(1)查找, 5个模板注册。
  V2: string名索引, ordered map, O(log N)查找, 2个DPU模板注册。
  两代都用__COUNTER__宏生成唯一静态变量名实现自注册。

TPL-ID-6: Notify双阶段同步协议
  ACK同步(保证前一步完成) + DATA_SIGNAL同步(保证数据就位)。
  wrapper层12个远端传输函数全部遵循此协议。

TPL-ID-7: Reduce路径硬件感知分叉
  64位类型/PROD走CPU fallback(硬件InlineReduce仅支持16/32位sum/max/min)。

TPL-ID-8: 连续切片合并优化
  LocalCopySlices检测连续切片并合并为单次DMA，减少指令数。

TPL-ID-9: 零裸new/delete
  整个35文件模块无裸new/delete，通过unique_ptr(7处)和shared_ptr(1处)管理。

TPL-ID-10: Host/Device双编译模型
  AICPU kernel(.so)由~80个文件双编译生成，AICPU_COMPILE宏使channel/topo退化为空操作。

### 反模式汇总(33条)

根基类(7条):
- TPL-AP-1: V1默认返回SUCCESS(掩盖遗漏) — alg_template_base.cc
- TPL-AP-2: PostSyncInterQueues日志copy-paste错误("AivCollAlgFactory") — aiv_alg_template_base.cc:95
- TPL-AP-3: "下面这组是否需要？"自我怀疑注释未清理 — alg_template_base.h:184
- TPL-AP-4: CcuAlgTemplateBase::InitCcuAlgTemplate遗漏myRank_/subCommRanks_初始化
- TPL-AP-5: templateRankSize_三种计算方式无文档(AICPU乘积/AIV第一维/CCU求和)
- TPL-AP-6: 四基类接口大量重复(无公共父类提取)
- TPL-AP-7: slices_引用dummy模式(容易困惑) — alg_template_base.h

AICPU(8条):
- TPL-AP-8: printf格式bug(%s传size_t) — task_exception_fun.cc:53
- TPL-AP-9: inputPtr/outputPtr语义疑似颠倒 — task_exception_fun.cc:47-48
- TPL-AP-10: V2路径跳过异常诊断和Profiling — kernel_launch.cc V2分支
- TPL-AP-11: 18处return 1丢失具体HcclResult错误码 — kernel_launch.cc
- TPL-AP-12: g_binKernelHandle全局变量无线程安全保护 — load_kernel.cc
- TPL-AP-13: getenv先调用后被MM_SYS_GET_ENV覆盖 — load_kernel.cc:23-24
- TPL-AP-14: V1 reinterpret_cast+sizeof偏移取变长数据 — kernel_launch.cc
- TPL-AP-15: libscatter_aicpu_kernel命名服务所有13个算子

AIV(5条):
- TPL-AP-16: GM flag轮询无超时(while(true), 5处) — sync_interface.h
- TPL-AP-17: header中定义static变量(ODR违规风险) — aiv_communication_base_v2.h
- TPL-AP-18: 合成指针key(高32位rank+低32位type) — hccl_aiv_utils.cc
- TPL-AP-19: INT64/UINT64映射交叉(5处疑似bug) — aiv_kernel_def.h
- TPL-AP-20: Tag重置阈值硬编码1000 — aiv_communication_base_v2.h

CCU(4条):
- TPL-AP-21: reinterpret_cast跨继承层次 — ccu_kernel_alg_base.cc
- TPL-AP-22: 哨兵值0xFFFFFFFF表示"无效" — ccu_kernel_utils.h
- TPL-AP-23: GroupBroadcast/GroupReduce代码重复(仅方向不同) — ccu_kernel_alg_base.cc
- TPL-AP-24: magic number硬编码 — ccu_kernel_utils.cc

DPU(2条):
- TPL-AP-25: reinterpret_cast+全量vector拷贝(ptr无效时崩溃) — dpu/kernel_launch.cc
- TPL-AP-26: 返回int32_t而非HcclResult — dpu/kernel_launch.cc

wrapper(7条):
- TPL-AP-27: CHK_PRT_RET日志参数错误(count_打成size_), copy-paste传播3处
- TPL-AP-28: 日志tag不一致([InsCollAlgFactory] vs [AlgDataTransWrapper])
- TPL-AP-29: 多处日志函数名写错(RecvReadReduce→SendWriteReduce等)
- TPL-AP-30: PostSyncInterThreads日志写成PreSyncInterThreads
- TPL-AP-31: 参数名拼写"snedInfo"出现6处
- TPL-AP-32: s8/u8类型不一致
- TPL-AP-33: DPU wrapper仅实现1/3函数(开发中)

### 特征总结表

| 维度 | template/ 模块特征 |
|------|-------------------|
| 规模 | 35文件/6199行, 7子目录 |
| 架构角色 | 为13个算子提供4种设备的执行模板基类 |
| 基类体系 | 4个独立基类(SDMA/AICPU/AIV/CCU) + 2个Registry |
| 错误处理 | CHK_RET 85次, V2+默认E_INTERNAL |
| 内存安全 | 零裸new/delete, unique_ptr主导 |
| 技术债 | 零TODO/FIXME/HACK |
| 条件编译 | 零业务级#ifdef(平台无关) |
| 日志 | ERROR 64%(错误可观测优先) |
| 反模式 | 33条(copy-paste日志错误最多, 占~30%) |
| 代码卫生 | 高(零技术债+零裸指针), 但copy-paste错误频发 |
| 与hcomm关系 | hccl是hcomm的精简分叉(Prepare从30重载砍到4) |
| 设备覆盖 | AICPU(主力) > CCU(大数据) > AIV(小数据) > DPU(早期) |

## 3.1.4 topo/ — 拓扑信息处理，拓扑感知的算法适配

### Module Overview

topo/ 是 hccl op_common 中负责拓扑感知的模块，位于 `src/ops/op_common/topo/`，
共 14 个文件(7 对 .h/.cc)，总规模 1870 行。

职责: 从 RankGraph 中提取结构化的拓扑信息(层级、形状、对称性)，
填充到 TopoInfo/AlgHierarchyInfo 结构体中，供算法选择器和执行器使用。

模块由两个功能区域组成:
- 拓扑信息提取(topo.h/cc + topo_host.h/cc): 纯函数式接口，从 RankGraph 查询并计算拓扑参数
- 拓扑匹配(topo_match_*.h/cc): Strategy 模式，根据拓扑形状构建算法分组(AlgHierarchyInfo)

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| topo.h | 35 | 6 个纯函数声明: A2/A3/Comm 拓扑计算 + rank 映射 + GCD |
| topo.cc | 146 | 混合基数(mixed-radix)rank 编解码 + 三种拓扑层级计算 |
| topo_host.h | 106 | 17 个函数声明: 拓扑初始化、server/module/superPod 信息计算 |
| topo_host.cc | 855 | 拓扑信息提取的完整实现(模块最大文件) |
| topo_match_base.h | 66 | TopoMatchBase 基类 + 910A 常量 + hash 函数 |
| topo_match_base.cc | 30 | 基类 MatchTopo() 默认实现(返回 E_INTERNAL) |
| topo_match_1d.h/cc | 39+54 | 1D Mesh 拓扑匹配: 所有 rank 打平为单组 |
| topo_match_2d.h/cc | 31+79 | 2D Mesh 拓扑匹配: rank 按 x/y 轴分成两组 |
| topo_match_multilevel.h/cc | 46+177 | 多级 Mesh 拓扑匹配: layer0 区分 x/y + layer1 同序号卡分组 |
| topo_match_ubx.h/cc | 42+164 | UBX 交叉互联拓扑匹配: layer0 不区分方向 + layer1 有条件构建 |

### 架构: 拓扑信息的计算和流转

#### 拓扑信息提取流程

```
HcclComm(通信域) → RankGraph(拓扑图查询)
    ↓
InitRankInfo()  [topo_host.cc:32]
    ├── CalcMyRankInfo()     — 基本 rank 信息(rankId, deviceType, serverIdx, superPodIdx)
    ├── GetPairLinkCounter() — 统计 server 内 rank 对的链路协议(HCCS/ROCE/PCIE/SIO)
    ├── SetServerModuleInfo()— module 划分(910B A+X 检测, moduleMap 构建)
    ├── SetSuperPodInfo()    — superPod 划分(仅 910_93)
    ├── CalcLinkInfo()       — HCCS_SW/SIO 链路关系(仅 910_93)
    └── CalcTopoShape()      — 拓扑形状识别(MESH_1D/CLOS/MESH_1D_CLOS/2-die)
    ↓
TopoInfo / TopoInfoWithNetLayerDetails (结构体)
    ↓
被 selector(算法选择) 和 executor(执行器) 消费
```

#### 拓扑匹配 Strategy 体系

```
TopoMatchBase  [topo_match_base.h:54]
├── virtual Describe() = 0       — 纯虚，返回描述字符串
├── virtual MatchTopo(...)       — 非纯虚，默认返回 E_INTERNAL
│
├── TopoMatch1D                  — 1D Mesh: 所有 rank 一个 flat 组
├── TopoMatch2D                  — 2D Mesh: 按 rank 间距区分 x/y 轴分两组
├── TopoMatchMultilevel          — 多级: layer0(x/y) + layer1(同序号卡)
└── TopoMatchUBX                 — UBX: layer0(不区分方向) + layer1(有条件)
```

使用方: V2 executor 在各算子中创建具体 matcher 实例并调用 MatchTopo()。
按 Grep 统计(97 处 include):
- topo_host.h: 被 12 个算子的 op.cc + 15+ executor .h 引用
- topo_match_1d.h: all_reduce/reduce_scatter/scatter/broadcast/reduce/all_gather/recv/all_to_all_v/all_gather_v/reduce_scatter_v
- topo_match_multilevel.h: all_reduce/reduce_scatter/scatter/broadcast/reduce/all_gather/all_to_all_v
- topo_match_ubx.h: all_reduce/reduce_scatter/all_gather/all_to_all_v
- topo_match_2d.h: scatter/all_to_all_v/broadcast/all_reduce

### Idioms & Conventions (惯用法)

1. 纯函数式拓扑计算(topo.h/cc): 6 个自由函数，无状态，无类。
   混合基数 rank 编解码是核心算法 — 全局 userRank 分解为各级 localRank:
   ```cpp
   // GetUserRankBySubCommRank: 重组各级 localRank 为全局 rank
   for (u32 i = 0; i < levels; i++) {
       userRank += (i == curLevel ? subCommRank : infos[i].localRank) * preLevelsRankSize;
       preLevelsRankSize *= infos[i].localRankSize;
   }
   // GetSubCommRankByUserRank: 从全局 rank 提取指定级的 localRank
   subCommRank = userRank / preLevelsRankSize % localRankSize;
   ```
   出处: topo.cc:116-145

2. (void) param 显式抑制未使用参数警告(项目级惯用法):
   ```cpp
   (void) comm;  // topo.cc:65
   ```
   出处: topo.cc:65,79,104

3. CHK_RET + CHK_PRT_RET 双宏错误处理:
   CHK_RET 用于子调用检查(42处 in topo_host.cc)，
   CHK_PRT_RET 用于条件校验(参数非法时打印日志+返回指定错误码)。
   出处: topo_host.cc 全文

4. 100% static_cast，零 reinterpret_cast:
   33 处 static_cast 全部用于 enum → u32 安全转换(26 处在 topo_host.cc)。
   这是 topo/ 模块最显著的代码卫生特征，与 op_common 其他子模块形成对比。
   出处: topo_host.cc 全文

5. 无状态 Strategy 对象:
   4 个 TopoMatch 子类均无成员变量(除 1D 有 myRank_ 和 rankIds_)，
   所有数据通过参数传入传出。
   出处: topo_match_multilevel.h, topo_match_ubx.h

6. #ifndef AICPU_COMPILE 条件编译:
   所有 MatchTopo() 实现都被此宏包裹，AICPU 编译时直接返回 SUCCESS(空操作)。
   拓扑匹配只在 Host 侧执行。
   出处: topo_match_1d.cc, topo_match_2d.cc, topo_match_multilevel.cc, topo_match_ubx.cc

7. Entry/Exit CONFIG_INFO 日志:
   InitRankInfo() 最后用 HCCL_RUN_INFO 打印所有关键拓扑参数(单行完整上下文):
   ```
   userRank[%u] rankSize[%u] deviceType[%s] serverIdx[%u] moduleIdx[%u] ...
   ```
   出处: topo_host.cc:47-56

### Architecture & Patterns (架构模式)

1. 三层拓扑模型(华为昇腾集群物理结构):
   - A2(2层): Level0=module 内(HCCS) → Level1=module 间(网络)
   - A3(3层): Level0=module 内 → Level1=superPod 内 server 间 → Level2=superPod 间
   - Comm(打平): Level0=单 rank → Level1=全局(非对称退化)
   AlgHierarchyInfo 使用固定 C 数组 SubCommInfo infos[HCCL_LOGIC_TOPO_LEVEL_NUM](最多 4 级)，
   非 vector，面向性能+可序列化到设备内存。
   出处: topo.cc:63-113

2. GCD 对称化策略:
   当 superPod 间 server 数不一致(非对称拓扑)时，取所有 superPod server 数的 GCD
   重新划分虚拟 superPod，使拓扑恢复对称，可使用 NHR-HCF 算法:
   ```cpp
   serverNumPerSuperPod = CalGCD(superPodToServerNum);
   superPodNum = serverNum / serverNumPerSuperPod;
   ```
   出处: topo_host.cc:178-184

3. 拓扑形状识别(Strategy 选择的前提):
   CalcLevel0TopoShape 从拓扑实例数和类型判断:
   - 1 个 MESH_1D → Level0Shape::MESH_1D
   - 1 个 CLOS → Level0Shape::CLOS
   - 2 个(1 CLOS + 1 MESH_1D) → Level0Shape::MESH_1D_CLOS
   出处: topo_host.cc:546-592

4. x/y 轴区分启发式:
   2D Mesh 中通过 rank 间距判断维度方向:
   `ranks[1] - ranks[0] == 1` → x 轴(rank 号连续)
   否则 → y 轴(rank 号有间隔，代表跨行)
   出处: topo_match_2d.cc:60, topo_match_multilevel.cc:69

5. 同序号卡(Layer1)分组:
   跨机通信时，同位置的卡组成跨机通信组(机器 A 的卡 N ↔ 机器 B 的卡 N)。
   判断方法: rankId % layer0Size == myRank % layer0Size，假设 rank 编号按机器连续编排。
   出处: topo_match_multilevel.cc:97-121, topo_match_ubx.cc:97-121

### Domain Knowledge (业务知识)

1. 华为昇腾拓扑层次:
   - Module: 一张物理板卡(通常 8 张 AI 芯片)
   - Server: 一台服务器(可含多个 module)
   - SuperPod: 多个 server 组成的聚合(L1 网络域)
   - 910B A+X 形态: 一个 server 内混合不同类型设备 module，
     通过 HCCS/非HCCS 链路区分(server 被拆分为 2 个 module)
   出处: topo_host.cc:189-211(IsDiffDeviceModule), :436-486(GetModuleIdxByRank)

2. 通信链路协议:
   - HCCS: 芯片间高速一致性连接(module 内)
   - SIO: 串行 I/O(module 内)
   - HCCS_SW: 通过交换机的 HCCS(module 间)
   - ROCE: RDMA over Converged Ethernet(跨机)
   - PCIE: PCI Express(本地设备间)
   CalcLinkInfo 中的 HCCS_SW/SIO 关系公式:
   `hccsSWNum == (deviceNumPerModule - 2) * deviceNumPerModule && sioNum == deviceNumPerModule`
   编码了设备无 HCCS_SW 自连接且同 SIO 伙伴也无 HCCS_SW 连接的物理约束。
   出处: topo_host.cc:213-238

3. Die 架构(910_93):
   - 2-die 全互连: 服务器内芯片有两个 die，每个 die 都有链路到对端
   - 单 die / 双 die regular / 双 die not-regular: 按 die 链路数差异分类
   - CalcLevel0MeshType 输出: NOT_MESH/SINGLE_DIE/TWO_DIE_REGULAR/TWO_DIE_NOT_REGULAR
   出处: topo_host.cc:730-855

4. 拓扑匹配的四种策略:
   - 1D: 适用于简单 Mesh(所有卡一维排列)
   - 2D: 适用于二维网格(x/y 并发通信)
   - Multilevel: 适用于多级 Mesh(layer0 x/y + layer1 跨机)
   - UBX: 适用于交叉互联(layer0 不区分方向，各实例对等)
   全部仅支持 DEV_TYPE_910_95。

5. 拓扑对称性对算法选择的影响:
   - 对称(所有 pod 大小相同): 可用 Ring/HD/NHR 等对称算法
   - 非对称但 GCD 可修正: CalGCD 重新划分 → NHR-HCF 算法
   - 完全非对称: 退化为打平拓扑(CalcGeneralTopoInfoForComm)，放弃分层优化

### Hardware Abstractions (硬件知识)

1. 芯片类型与拓扑功能矩阵:

   | 芯片 | superPod | A+X检测 | 链路信息 | 拓扑形状 | 多级匹配 |
   |------|----------|---------|----------|----------|----------|
   | 910_93 | Y | N | CalcLinkInfo | CalcTopoShape+die | Y |
   | 910B | N | IsDiffDeviceModule | N | N | N |
   | 910_95 | N | N | N | N | Y(MatchTopo) |

2. 硬编码硬件常量:
   - DEVICE_PER_MODULE_A2 = 8: A2 每 module 8 卡(topo_host.h:25)
   - dieNum = 2: 双 die 硬编码(topo_host.cc:750,805)
   - HcclNetLayer 三层: L0(server)/L1(superPod)/L2(更上层)

3. 网络拓扑类型:
   - COMM_TOPO_1DMESH: 一维 Mesh
   - COMM_TOPO_CLOS: 多级交换网络
   - COMM_TOPO_CUSTOM: 自定义拓扑(1D matcher 接受)

### Anti-Patterns (反模式)

TOPO-AP-1: GetCurrentServerStartRank/EndRank 返回 uint32_t 但内部使用 CHK_RET
  CHK_RET 展开为 `return hcclRet`(HcclResult 类型)，在返回 uint32_t 的函数中
  编译器隐式转换，错误码被当作 rank 值返回。确认 bug。
  出处: topo_host.cc:331-364

TOPO-AP-2: GetCurrentServerStartRank 与 GetCurrentServerEndRank 大量重复代码
  两函数前半部分完全相同(遍历+累加)，仅结尾差 1 行。应提取公共函数。
  出处: topo_host.cc:331-364

TOPO-AP-3: Is2DieFullMesh 与 CalcLevel0MeshType 大段逻辑重复
  两函数都遍历 rank 查链路统计 die 链路数，~50 行几乎完全相同。
  出处: topo_host.cc:730-855

TOPO-AP-4: ExtractTopoDetails 错误检查 bug
  HcclRankGraphGetRanksByTopoInst 返回值未赋给 ret，
  但后续 CHK_PRT_RET(ret != HCCL_SUCCESS, ...) 检查的仍是前一个 API 的 ret。
  出处: topo_host.cc:712-716

TOPO-AP-5: TopoMatch1D deviceType 校验不返回错误
  打印 HCCL_ERROR 后继续执行(无 return)。对比 2D 正确使用 CHK_PRT_RET 返回。
  ```cpp
  // 1D — bug: 缺少 return
  if (deviceType != DEV_TYPE_910_95) {
      HCCL_ERROR("...");
  }  // 继续执行!
  // 2D — 正确
  CHK_PRT_RET(deviceType != DEV_TYPE_910_95, HCCL_ERROR("..."), HCCL_E_PARA);
  ```
  出处: topo_match_1d.cc:30-32 vs topo_match_2d.cc:34-36

TOPO-AP-6: TopoMatch1D myRank_ 未初始化
  成员变量 myRank_ 在构造函数中未赋值，但在错误日志中被引用(使用未定义值)。
  对比 2D 正确调用 HcclGetRankId(comm, &myRank) 获取。
  出处: topo_match_1d.h:29, topo_match_1d.cc:31

TOPO-AP-7: TopoMatch2D ranks 越界访问
  当 rankNum == 1 时打印错误但不 return，
  后续 ranks[1] - ranks[0] 读越界内存(ranks.size()==1)。
  出处: topo_match_2d.cc:64-66

TOPO-AP-8: TopoMatchUBX Describe() copy-paste 错误
  返回与 Multilevel 完全相同的描述字符串，UBX 应有自己的描述。
  出处: topo_match_ubx.cc(Describe 返回 "Topo Match for combined Algorithm: layer 0 Mesh, layer 1 NHR.")

TOPO-AP-9: Multilevel 与 UBX 大量重复代码
  MatchTopo/TopoForLayer1/CheckVecElementAllSame/PrintCArray 几乎完全相同，
  仅 TopoForLayer0 的 mesh2d 分支和 layer1 调用条件有差异。
  应通过 Template Method 重构，把差异点提取为虚方法。
  出处: topo_match_multilevel.cc 全文 vs topo_match_ubx.cc 全文

TOPO-AP-10: 重复 include
  topo_host.cc 中 hccl_rank_graph.h 和 hcomm_primitives.h 各被 include 两次。
  topo.cc 中也有 hcomm_primitives.h 被重复 include。
  出处: topo_host.cc:13-17, topo.cc:13-15

TOPO-AP-11: 4 个 dead code 常量
  topo.cc 中 FACTOR_NUM_TWO/DEVICE_PER_MODULE/NET_LAYER_NUM_TWO/NET_LAYER_NUM_THREE
  全部未被引用。topo_host.cc 中 DEVICE_PER_MODULE 也未使用。
  出处: topo.cc:21-24, topo_host.cc:26

TOPO-AP-12: GetSubCommRankByUserRank curLevel 越界无检测
  如果 curLevel >= levels，循环不执行，subCommRank 不被赋值但函数返回 SUCCESS。
  出处: topo.cc:133-145

TOPO-AP-13: CalcGeneralTopoInfoForA3 除零风险
  serverNumPerSuperPod 未校验非零，直接用作除数。
  出处: topo.cc:86

TOPO-AP-14: topo_match_base.h 中 SERVER_910A_* 全局 const 容器
  vector 和 unordered_set 定义为 const 全局变量(非 constexpr/inline)，
  每个翻译单元独立初始化堆内存。应改为 inline const 或函数内 static。
  出处: topo_match_base.h:40-52

TOPO-AP-15: include guard / namespace 注释不一致
  - topo_match_1d.h guard 为 TOPO_MATCH_MESH(与文件名不符)
  - topo_match_multilevel.h guard 为 TOPO_MATCH_MESH_NHR(旧名称)
  - 多处 namespace 结尾注释写 "// namespace Hccl" 而实际是 ops_hccl
  - topo_match_base.cc 中拼写 "interfacce"(多一个 c)
  - topo_match_base.h hash 函数名 "HashFuc"(应为 HashFunc)
  出处: topo_match_1d.h:38, topo_match_multilevel.h:15, topo_match_base.cc:27, topo_match_base.h:38

TOPO-AP-16: 参数风格不一致
  同一组 API 中 TopoInfo 用裸指针，AlgHierarchyInfo 用引用，混用两种风格。
  出处: topo.h 全部函数签名

TOPO-AP-17: Multilevel TopoForLayer0 mesh2d 日志写 "Mesh 1D" 实际是 2D
  UBX 也有同样的误导性日志。
  出处: topo_match_multilevel.cc:49, topo_match_ubx.cc:47

### Notable Patterns (值得注意的模式)

1. 模块独立性极高:
   topo/ 不依赖 executor/selector/template 任何头文件，仅依赖通用基础设施(base.h, log.h,
   alg_param.h, hccl_rank_graph.h)。反向被所有算子和 executor 广泛引用。
   这是 op_common 中耦合度最低的子模块。

2. C/C++ 风格分区:
   topo.h/cc 是纯 C-style 自由函数(无类、无状态)，
   topo_match_*.h/cc 是 C++ Strategy 模式(虚函数+继承)。
   topo_host.h/cc 混合风格(自由函数但参数为 C++ 类型)。

3. 硬件知识集中度:
   topo_host.cc 的 855 行中，大量逻辑与芯片代际(910B/910_93/910_95)和物理拓扑强耦合。
   新增芯片类型时，此文件是必改文件(热点)。

4. 910_95 专属:
   4 个 TopoMatch 子类全部硬编码 DEV_TYPE_910_95。
   对其他芯片的拓扑匹配走传统路径(InitRankInfo → CalcGeneralTopoInfo)。

5. 量化特征:
   - CHK_RET: 71 次/4 文件(topo_host.cc 占 59%)
   - static_cast: 33 次(100%, 零 reinterpret_cast)
   - 日志: 101 次/7 文件
   - TODO/FIXME/HACK: 0 次
   - 智能指针: 0 次(栈分配+外部管理)
   - 反模式: 17 条(TOPO-AP-1~17)

## 3.1.5 op_common 综合分析

### 模块全景

op_common 是 hccl 所有集合通信算子的公共框架层，位于 `src/ops/op_common/`，
共 70 个 C++ 文件(~10,212 行)，4 个子模块:

| 子模块 | 文件数 | 行数 | 职责 |
|--------|--------|------|------|
| executor/ | 12 | ~1,301 | V1/V2 执行器基类 + 通道计算 + 注册工厂 |
| selector/ | 34 | ~2,500 | 算法选择: 执行模式→拓扑→数据量→算法名 |
| template/ | 35 | ~6,199 | 4 种设备的执行模板基类 + 注册 + wrapper |
| topo/ | 14 | ~1,870 | 拓扑信息提取 + 4 种拓扑匹配 Strategy |
| op_common.cc | 1 | ~342 | 顶层编排入口(OpCommon) |

依赖方向: topo → (无依赖) ← executor ← selector ← template ← 各算子
topo 是最底层(零依赖)，被所有其他子模块和算子引用。

### 跨子模块惯用法(OC-ID-1~12)

OC-ID-1: CHK_RET 100% 覆盖(项目级强制)
  259 次/17 文件。所有返回 HcclResult 的子调用都必须用 CHK_RET 检查。
  唯一例外: topo.cc 的纯计算函数(无子调用)和 selector 的布尔判断。

OC-ID-2: V1/V2 双代并存
  贯穿 executor(ExecutorBase vs InsCollAlgBase)、
  template(AlgTemplateBase vs InsAlgTemplateBase)、
  selector(SelectorRegistry 统一)三个子模块。
  V2 完全独立不继承 V1，是主力方向(V2: 60 注册 vs V1: 4 注册)。

OC-ID-3: Registry 自注册工厂(3 套)
  - ExecutorRegistry: V1(tag→unique_ptr) + V2((CMDType,tag)→shared_ptr, 7 种注册宏)
  - SelectorRegistry: 优先级注册(14 个 REGISTER_SELECTOR_BY_OPTYPE, 优先级全 18)
  - TemplateRegistry: V1(vector[2000] O(1)) + V2(ordered_map O(logN))
  三套 Registry 各自独立，通过静态初始化注册，运行时按 key 查找。
  共 46 个 REGISTER 宏调用(跨 5 个头文件定义)。

OC-ID-4: 纯函数式工具模块(topo.h/cc, channel/channel.h/cc)
  无状态自由函数。topo 6 个函数做 rank 编解码，channel 11 个函数做通道计算。
  优点: 可测试、无副作用。缺点: 参数列表较长。

OC-ID-5: #ifndef AICPU_COMPILE 条件编译(模块级)
  executor/channel(8 处)和 topo_match(4 处)的 Host 侧逻辑用此宏包裹。
  AICPU 编译时通道计算和拓扑匹配为空操作(返回 SUCCESS)。

OC-ID-6: static_cast 优先于 reinterpret_cast
  static_cast 172 次 vs reinterpret_cast 45 次(比值 3.8:1)。
  topo/ 达到 100% static_cast(零 reinterpret_cast)。
  reinterpret_cast 集中在 template/aicpu(kernel launch)和 template/wrapper。

OC-ID-7: 默认虚方法实现策略分化
  - V1 ExecutorBase: 默认返回 SUCCESS(危险 — 遗漏 override 不报错)
  - V2 InsCollAlgBase: 默认返回 E_INTERNAL(安全 — 遗漏立即暴露)
  - TopoMatchBase: 默认返回 E_INTERNAL + HCCL_ERROR 日志
  V2/topo 的"fail-closed"策略是正确方向。

OC-ID-8: 5 路执行模式分派(selector 统一框架)
  DPU → CCU_MS → CCU_SCHED → AIV → AICPU(兜底)
  降级链: CCU_MS 失败→CCU_SCHED→AICPU。
  14 个 selector 共享此框架，差异在拓扑→算法名映射。

OC-ID-9: 四代独立基类(template，不继承)
  AlgTemplateBase(V1/SDMA) / InsAlgTemplateBase(V2/AICPU) /
  AivAlgTemplateBase(AIV) / CcuAlgTemplateBase(CCU)。
  不同处理器的资源模型差异太大无法统一继承。

OC-ID-10: 零 TODO/FIXME/HACK(整个 op_common)
  70 个文件、10,212 行中零技术债注释。
  对比 hcomm 的 13+ TODO(next/ 框架)，hccl op_common 代码成熟度更高。

OC-ID-11: desc_ 描述符模式(executor V2 + template)
  所有 V2 executor 和 template 用 desc_ 成员(string)存储算法/模板名，
  Describe() 虚方法返回它，用于日志和 Registry 查找。

OC-ID-12: 协议选择矩阵(channel/)
  Level0→HCCS, Level1→设备/部署决定, Level2→ROCE。
  channel.cc 中按 (拓扑层级 x 拓扑类型) 组合选择通信协议。

### 架构模式(OC-AP-1~6)

OC-AP-1: 四子模块分工
  topo(是什么拓扑) → selector(选什么算法) → executor(怎么编排) → template(怎么执行)
  对应集合通信算子的四个决策阶段。单向依赖，topo 最底层。

OC-AP-2: 三阶段执行框架
  CalcRes(资源计算) → Orchestrate(任务编排) → KernelRun(核心执行)
  V1 和 V2 executor 都遵循此骨架，V2 增加了 Serialize/Deserialize。

OC-AP-3: Strategy + Registry 组合
  selector 用 Strategy 模式(AutoSelectorBase + 14 个具体 selector)，
  通过 SelectorRegistry 自注册。executor 和 template 同理。
  运行时多态(虚函数)+ 编译时注册(静态构造)。

OC-AP-4: 拓扑信息的计算-序列化-消费链
  Host: InitRankInfo()→TopoInfo(struct) →[BinaryStream序列化]→ H2D memcpy
  Device: 反序列化→TopoInfo(struct) → executor/template 消费
  AlgHierarchyInfo 用固定 C 数组(非 vector)，可直接二进制序列化。

OC-AP-5: 芯片代际分叉贯穿全栈
  910B(A+X 检测, topo)→910_93(superPod+die, topo)→910_95(MatchTopo, topo_match)
  selector 中按 SoC 选执行模式(AIV/CCU/AICPU)。
  template 中四代基类对应四种处理器。
  芯片类型是贯穿所有子模块的核心分叉维度。

OC-AP-6: 拓扑→算法的决策链
  物理拓扑(RankGraph)→ 拓扑参数(TopoInfo)→ 拓扑形状(Level0Shape)
  → 算法匹配(AlgHierarchyInfo)→ 算法名(selector)→ executor(Registry 查找)
  → template(Registry 查找)→ kernel launch
  7 步决策链，每步缩小选择范围。

### 业务领域知识(OC-DK-1~5)

OC-DK-1: 三层拓扑模型
  Level0 = module 内(HCCS, 8 卡) → Level1 = server 间/superPod 内 → Level2 = superPod 间
  混合基数 rank 编解码: 全局 rank ↔ 各级 localRank。
  GCD 对称化策略: 非对称拓扑 → CalGCD 重划分 → NHR-HCF 算法。

OC-DK-2: 5 种拓扑形状
  MESH_1D / CLOS / MESH_1D_CLOS(混合) / 2-die(regular/not-regular)
  由 CalcLevel0TopoShape + Is2DieFullMesh + CalcLevel0MeshType 识别。

OC-DK-3: 执行模式与处理器对应
  DPU(数据处理单元) / CCU(集合通信单元, MS/SCHED 两种调度) /
  AIV(AI 向量处理器) / AICPU(通用 CPU)
  每种处理器有独立的模板基类、编程模型和资源管理方式。

OC-DK-4: 硬件限制常量
  RDMA 2GB/WQE, SDMA 4GB/task(47 处 4GB 切分惯用法)。
  HCCS_SW/SIO 链路关系公式(编码物理拓扑约束)。
  910A 4 环拓扑序列(硬编码在 topo_match_base.h)。

OC-DK-5: UBX vs Mesh 拓扑
  UBX(交叉互联): layer0 所有实例对等，不区分方向，layer0Size 取最大值。
  Mesh 2D: layer0 按 rank 间距区分 x/y 方向，layer0Size = x * y。
  同序号卡跨机通信: rankId % layer0Size == myRank % layer0Size。

### 反模式汇总(跨子模块)

按严重程度分级:

严重(确认 bug):
- TOPO-AP-1: CHK_RET 在返回 uint32_t 的函数中使用(错误码当 rank 值)
- TOPO-AP-4: ExtractTopoDetails 检查错误 API 的返回值
- TOPO-AP-5: TopoMatch1D deviceType 校验不 return
- TOPO-AP-7: TopoMatch2D rankNum==1 时越界访问 ranks[1]
- SEL-AP-1: FP64 重复/INT64 遗漏(3 个 selector)
- SEL-AP-2: AllGatherV fall-through(算法名被覆盖)
- EXEC-AP-1: Describe() 返回 "111"(开发残留)
- EXEC-AP-3: static_assert 消息错误
- EXEC-AP-5: malloc 无 free(channel.cc:314)

中等(代码质量):
- TOPO-AP-9: Multilevel/UBX 大量重复代码
- TOPO-AP-3: Is2DieFullMesh/CalcLevel0MeshType 大段重复
- SEL-AP-4: levle0Algo 拼写错误固化(8 文件 57 处)
- TPL-AP-27~33: copy-paste 日志错误(~10 处)
- EXEC-AP-7: RefreshAlgType 重复代码(V1/V2 各有)

低级(风格/卫生):
- TOPO-AP-10: 重复 include
- TOPO-AP-11: dead code 常量
- TOPO-AP-15: include guard/namespace 注释不一致
- SEL-AP-5: namespace 注释错误
- EXEC-AP-2: namespace 注释错误

总计: executor 9 条 + selector 13 条 + template 33 条 + topo 17 条 = 72 条反模式。
copy-paste 错误是最突出的反模式类型(占 ~30%)。

### 特征总结表

| 维度 | op_common 模块特征 |
|------|-------------------|
| 规模 | 70 文件/10,212 行, 4 子模块 |
| 架构角色 | 所有算子的公共框架: 拓扑→选择→编排→执行 |
| 双代并存 | V1(遗留, 4 注册) / V2(主力, 60 注册), 完全独立不继承 |
| Registry | 3 套自注册工厂(executor/selector/template), 共 46 个注册宏 |
| 错误处理 | CHK_RET 259 次/17 文件, V2 默认 E_INTERNAL(安全) |
| 类型安全 | static_cast:reinterpret_cast = 3.8:1, topo 达到 100% |
| 技术债 | 零 TODO/FIXME/HACK(70 文件全部) |
| 条件编译 | #ifndef AICPU_COMPILE(12 处), Host/Device 分裂 |
| 反模式 | 72 条(copy-paste 占 ~30%, 确认 bug 9 条) |
| 代码卫生 | 高(零技术债), 但 copy-paste 和命名不一致频发 |
| 芯片耦合 | 芯片代际分叉贯穿全栈(topo→selector→template) |

### 与 hcomm 对比

| 维度 | hccl op_common | hcomm algorithm/ |
|------|---------------|-----------------|
| 规模 | 70 文件/10K 行 | 882 文件/122K 行 |
| 架构角色 | 框架+选择+拓扑 | 算法模板+执行+资源 |
| 错误处理密度 | 259 CHK_RET/70 文件 = 3.7/文件 | 5476 CHK_RET/301 文件 = 18.2/文件 |
| 技术债 | 零 | 零 |
| reinterpret_cast | 45 次(主要 template) | 62 次(主要 alltoallv) |
| Registry | 3 套, 46 个注册 | 2 套(Op+Exec), 268 个注册 |
| 关系 | hccl 消费 hcomm 的通信能力 | hcomm 提供通信实现 |
