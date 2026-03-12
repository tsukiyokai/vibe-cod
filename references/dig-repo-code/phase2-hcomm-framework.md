# Phase 2.2: hcomm Framework Layer Analysis

## 2.2.1 communicator/ — IndependentOp Sub-module Deep Dive

### Module Overview

`independent_op/` 是 hcomm communicator 下最大的子模块(52个文件)，位于:
`src/framework/communicator/impl/independent_op/`

IndependentOp 是面向"自定义算子"(用户编写的 AICPU/Host 侧算子)提供通信资源管理的独立子系统。
它与主 communicator 的"内建集合通信"(AllReduce/AllGather等)并列，
专门服务于用户需要自行编排通信逻辑的场景。

整体定位:
- 主communicator: 内建集合通信，框架自动编排 transport/algorithm/task
- IndependentOp: 用户自定义算子的通信资源池，暴露线程/notify/channel/context/memory等原语给用户

### Key Files

核心入口:
- `independent_op.h/.cc` — IndependentOp 类，资源管理器聚合入口
- `independent_op_context_manager.h/.cc` — ContextManager，engine context 生命周期管理
- `manager_common.h` — 回调函数结构体定义(ManagerCallbacks, ChannelManagerCallbacks)
- `hccl_independent_common.h` — CommEngine/NotifyType 验证工具函数

C API 胶水层(把 HcclComm 句柄分派到 IndependentOp 或 V2 路径):
- `hccl_independent_op_engine.cc` — 线程/notify 分配 API
- `hccl_independent_op_ctx.cc` — engine context CRUD API
- `hccl_independent_op_mem.cc` — 内存注册/反注册/buffer获取 API
- `hccl_independent_op_channel.cc` — channel 创建/销毁/查询 API
- `hccl_independent_rank_graph.cc` — rank graph 查询 API(最大，453行，18个API函数)

子管理器:
- `comm_mem_manager.h/.cc` — CommMemMgr，用户内存注册/反注册，CCL buffer 访问
- `reg_mem_manager.h` — RegMemMgr，空壳占位(尚未实现)
- `channel/channel_manager.h/.cc` — ChannelManager，通信通道创建/管理
- `rank_graph/rank_graph_base.h` — RankGraph 抽象基类
- `rank_graph/rank_graph.h/.cc` — RankGraphV1 实现(完整拓扑计算)
- `rank_graph/rank_graph_v2.h/.cc` — RankGraphV2 实现(pImpl 委托到 IRankGraph)
- `resource/engine/comm_engine_res_manager.h/.cc` — CommEngineResMgr，线程/notify 资源管理
- `resource/engine/thread_manager.h` — ThreadMgr，线程池管理
- `resource/engine/notify_manager.h` — NotifyManager，notify 分配/释放

### Architecture: IndependentOp 的组合式设计

```
IndependentOp (聚合器)
  |-- RegMemMgr          (空壳，预留)
  |-- CommMemMgr          (内存注册/CCL buffer)
  |-- CommEngineResMgr    (线程+notify资源)
  |     |-- ThreadMgr     (线程池，区分 CPU_TS/AICPU_TS)
  |     |-- NotifyManager (notify 分配/释放)
  |-- ContextManager      (engine context CRUD)
  |-- ChannelManager      (通信通道生命周期)
```

IndependentOp 本身没有复杂逻辑，仅做两件事:
1. 在 SetIndependentOpConfig 中初始化所有子管理器
2. 提供 GetXxxMgr() 访问器，让外部 C API 函数取到子管理器引用

这是典型的 Facade + Composition 模式:
- Facade: 一个入口点聚合多个子系统
- Composition over Inheritance: 所有子管理器都是值成员，不用继承

### 核心发现 1: V1/V2 双路径分派模式

这是 independent_op 模块最突出的架构特征。每个 C API 函数都有相同的分派结构:

```cpp
// hccl_independent_op_ctx.cc:20-51 (典型代表)
HcclResult HcclEngineCtxCreate(HcclComm comm, ...) {
    // 1. 参数校验
    CHK_PTR_NULL(comm);
    CHK_PTR_NULL(ctxTag);
    ...
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
    HcclResult ret = HCCL_SUCCESS;

    // 2. V2路径: 通过 CollComm 获取 Manager
    if (hcclComm->IsCommunicatorV2()) {
        CollComm* collComm = hcclComm->GetCollComm();
        CHK_PTR_NULL(collComm);
        ContextManager* contextMgr = collComm->GetContextManager();
        CHK_PTR_NULL(contextMgr);
        ret = contextMgr->CreateCommEngineCtx(...);
    }
    // 3. V1路径: 通过 IndependentOp 获取 Manager
    else {
        auto& contextMgr = hcclComm->GetIndependentOp().GetContextManager();
        ret = contextMgr.CreateCommEngineCtx(...);
    }

    // 4. 统一错误处理和日志
    if (ret != HCCL_SUCCESS) { HCCL_ERROR(...); return ret; }
    HCCL_RUN_INFO(...);
    return HCCL_SUCCESS;
}
```

关键观察:
- V1: Manager 是 IndependentOp 的值成员，通过引用(auto&)访问，用 `.` 调用
- V2: Manager 通过 CollComm 间接获取指针，用 `->` 调用
- 两路径调用的是同一个 Manager 类的同名方法，接口完全相同
- 这说明 V2 只是改变了 Manager 的持有者和生命周期管理方式，逻辑不变

此外，hccl_independent_rank_graph.cc 中还有第三种分派:

```cpp
// hccl_independent_rank_graph.cc:70-99 (典型代表)
HcclResult HcclRankGraphGetLinks(HcclComm comm, ...) {
    // HCCLV2_FUNC_RUN 宏: 检测 SoC 类型，若支持V2则直接 return V2实现
    HCCLV2_FUNC_RUN(
        [&]() -> HcclResult {
            // 还有环境变量二次判断
            const char *indOp = getenv("HCCL_INDEPENDENT_OP");
            if (indOp == nullptr || strcmp(indOp, "") == 0) {
                CHK_RET(HcclGetLinksV2(comm, ...));
                return HCCL_SUCCESS;
            }
            // V2 + INDEPENDENT_OP 路径
            RankGraph* rankGraph = nullptr;
            CHK_RET(GetRankGraphFromComm(comm, &rankGraph));
            CHK_RET(rankGraph->GetLinks(...));
            return HCCL_SUCCESS;
        }());
    // V1 fallback
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
    ret = hcclComm->GetLinks(...);
    ...
}
```

HCCLV2_FUNC_RUN 宏定义(param_check_basic_v2.h:14-21):
```cpp
#define HCCLV2_FUNC_RUN(func, ...) \
    do { \
        const char *socNamePtr = aclrtGetSocName(); \
        CHK_PTR_NULL(socNamePtr); \
        if (IsSupportHCCLV2(socNamePtr)) { \
            return func; \
        } \
    } while (0)
```

这里形成三层分派:
1. SoC 类型检测 (HCCLV2_FUNC_RUN 宏) → 硬件级分支
2. 环境变量 HCCL_INDEPENDENT_OP → 特性开关
3. IsCommunicatorV2() → 运行时对象类型检查

### 核心发现 2: ContextManager — Engine Context 的生命周期管理

ContextManager 管理不同 CommEngine 类型的执行上下文内存。

CommEngine 类型体系(hccl_independent_common.h:22-32):
- COMM_ENGINE_CPU / COMM_ENGINE_CPU_TS / COMM_ENGINE_CCU → Host 内存 (malloc/free)
- COMM_ENGINE_AICPU / COMM_ENGINE_AICPU_TS / COMM_ENGINE_AIV → Device 内存 (hrtMalloc/hrtFree)

核心数据结构(independent_op_context_manager.h:53-56):
```cpp
// 三级索引: tag → engine → HcclMem
std::unordered_map<std::string,
    std::unordered_map<CommEngine, HcclMem, CommEngineHash>> contextMap_;
// 反向索引 (用于 Destroy)
std::unordered_map<HcclMem, std::string, HcclMemHash, HcclMemEqual> tagMap_;
std::unordered_map<HcclMem, CommEngine, HcclMemHash, HcclMemEqual> engineMap_;
```

操作:
- CreateCommEngineCtx: 按 tag+engine 创建，根据 engine 类型决定 host/device 内存
- GetCommEngineCtx: 按 tag+engine 查询
- CopyCommEngineCtx: Host→Device 或 Host→Host 的内存拷贝
- DestroyCommEngineCtx: 通过反向索引找到 tag+engine，清理三个 map + 释放内存

惯用法 — 自定义 Hash 和 Equal:
```cpp
// independent_op_context_manager.h:20-29
struct HcclMemHash {
    static constexpr size_t MEM_ADDR_SHIFT = 1;
    static constexpr size_t MEM_SIZE_SHIFT = 2;
    size_t operator()(const HcclMem& mem) const noexcept {
        size_t hashMemType = std::hash<uint32_t>{}(static_cast<int>(mem.type));
        size_t hashMemaddr = std::hash<void*>{}(mem.addr);
        size_t hashMemSize = std::hash<uint64_t>{}(mem.size);
        return hashMemType ^ (hashMemaddr << MEM_ADDR_SHIFT) ^ (hashMemSize << MEM_SIZE_SHIFT);
    }
};
```

这是把 POD 结构体用作 unordered_map key 的标准做法:
自定义 Hash functor + Equal functor + 显式模板参数。

线程安全: 所有公共方法均使用 `std::lock_guard<std::mutex>`。

### 核心发现 3: RankGraph — V1 全量实现 vs V2 pImpl 委托

继承体系:
```
RankGraph (abstract base, rank_graph_base.h)
  |-- RankGraphV1 (rank_graph.h/.cc, 706行 — 完整实现)
  |-- RankGraphV2 (rank_graph_v2.h/.cc, 101行 — 纯委托)
```

RankGraph 基类(rank_graph_base.h:22-45):
- 定义 HCCL_NETLAYER_0/1/2 三层网络常量
- 纯虚接口: GetRankGraphInfo, GetLinks, GetNetLayers, GetInstTopoTypeByNetLayer, GetInstSizeByNetLayer, GetInstRanksByNetLayer, GetInstSizeListByNetLayer
- 部分默认实现: Init(空), GetRankId/GetRankSize(返回NOT_SUPPORT), GetHeterogMode(空)

RankGraphV1 (rank_graph.h/rank_graph.cc, 706行):
完整的网络拓扑模型，核心职责:
1. 从 RankTable_t 解析所有 rank 的网络信息(IP、devicePhyId、serverId、superPodId)
2. 根据设备类型(910B/910_93/310P 等)确定 CommProtocol(HCCS/RoCE/PCIe/SIO)
3. 构建 rank 间的 CommLink(源端点→目的端点+链路协议)
4. 按 NetLayer 分层组织 rank 列表

关键设计:
- 三层网络: L0(server 内) / L1(superPod 内/跨server) / L2(跨superPod)
- 协议选择逻辑(GetCommProtocolFromRankInfo, 行206-248): 根据 netLayer + 同/跨 server + 设备类型 的组合决定协议
- 同 server 内: HCCS(HCCS_TYPE/HCCS_SW_TYPE) > SIO > PCIe > RoCE(特殊条件)
- 跨 server: 910B→RoCE, 910_93同superPod→HCCS, 跨superPod→RoCE, 310P→PCIe
- Link 缓存: `std::map<tuple<netLayer,src,dst>, vector<CommLink>>` 避免重复构建
- Endpoint 构建: 每个 rank 按设备类型生成 1-3 种协议的 EndpointDesc(ROCE/HCCS/PCIE)

硬件知识编码(rank_graph.cc):
- DEV_TYPE_910B: L0=HCCS mesh, L1=RoCE CLOS; 标卡或A+X跨mesh时L1可用RoCE
- DEV_TYPE_910_93: L0=全互连, L1=HCCS(superPod内), L2=RoCE(跨superPod)
- DEV_TYPE_310P1/P3: L0=PCIe/HCCS
- DEVICE_PER_MODULE 常量: 910B 的 mesh 分界线(phyId < 4 = 下半mesh, >= 4 = 上半mesh)

异构检测(InitHeterogMode):
- 1种芯片 → HOMOGENEOUS
- 910B + 910_93 → MIX_A2_A3
- 其他组合 → INVALID

RankGraphV2 (rank_graph_v2.h/rank_graph_v2.cc, 101行):
纯 pImpl 委托模式:
```cpp
class RankGraphV2 : public RankGraph {
private:
    std::unique_ptr<Hccl::IRankGraph> pImpl;  // 外部接口
};
// 构造: pImpl = std::make_unique<Hccl::IRankGraph>(rankGraphPtr);
// 所有方法: return pImpl->XXX(...);
```
V2 不做任何计算，直接委托到 Hccl::IRankGraph(外部库提供的接口)。
这是标准的 Bridge/pImpl 模式，用于解耦具体拓扑实现。

V1 vs V2 的区别:
- V1: 自己实现拓扑计算，内存中维护完整的 rank 索引和 link 缓存
- V2: 委托到外部接口(可能是新版拓扑服务)
- V2 额外方法: GetTopoInstsByLayer, GetTopoType, GetRanksByTopoInst, GetEndpointNum, GetEndpointDesc, GetEndpointInfo — 这些在 V1 基类中不存在
- hccl_independent_rank_graph.cc 中通过 `static_cast<RankGraphV2*>(rankGraph)` 向下转型调用 V2 特有方法

在 hccl_independent_rank_graph.cc 中的 V2 专有 API:
- HcclRankGraphGetTopoInstsByLayer
- HcclRankGraphGetTopoType
- HcclRankGraphGetRanksByTopoInst
- HcclRankGraphGetEndpointNum
- HcclRankGraphGetEndpointDesc
- HcclRankGraphGetEndpointInfo

这些 V2 专有 API 在 V1 path 下返回 HCCL_E_NOT_SUPPORT，注释"only A5 is supported"
(A5 = Ascend 950/910_95/910_96 新一代硬件)。

### 核心发现 4: ChannelManager — 通信通道的完整生命周期

ChannelManager 管理"通道"(Channel)，即自定义算子用来与远端 rank 通信的链路资源。

通道标识:
```
channelKey = tag + ":" + engine + ":" + remoteRank + ":" + channelProtocol
```

数据结构(channel_manager.h:96-109):
```cpp
std::unordered_map<std::string, ChannelHandle> channelHandleMap_;  // key → handle
std::unordered_map<ChannelHandle, std::string> keyMap_;            // handle → key (反向)
std::unordered_map<ChannelHandle, CommEngine> engineMap_;          // handle → engine
std::unordered_map<ChannelHandle, ChannelHandle> channelD2HMap_;   // device handle → host handle
std::vector<LINK> channelLinks_;                                    // 保持 transport link 活跃
```

ChannelHandle 本质:
- CPU/CPU_TS engine: `ChannelHandle = reinterpret_cast<ChannelHandle>(link.get())`，直接是 Transport* 指针
- AICPU/AICPU_TS engine: Device 侧 handle(通过 kernel 返回)，另存 D2H 映射到 host Transport*

创建流程(ChannelCommCreate, channel_manager.cc:650-708):
1. CheckChannelParam: 校验 notifyNum/memHandleNum/remoteRank/protocol/engine
2. PrepareHandleArray: 复用已有 channel(幂等)，分离出 needCreateDescs
3. BuildChannelRequests: 构建 OpCommTransport 请求
4. channelCallbacks_.indOpTransportAlloc: 回调上层建链
5. AICPU 路径:
   - 确保 AicpuCommInit(通过 callbacks_)
   - AicpuChannelInit: 深拷贝参数到 Device 内存 → 下 kernel → 拷贝 channelList 回 Host
6. CPU 路径: 直接用 Transport* 指针作为 ChannelHandle
7. RegisterHandle + RegisterHandleHDPair(AICPU) / RegisterHandle(CPU)

AICPU Channel Init 的深拷贝链:
```
Host struct → DeepCopyH2DchannelParam
  → 每个 HcclIndOpChannelRemoteResV2:
    → DeepCopyH2DChannelRemoteResV2
      → P2P: DeepCopyH2DChannelP2p (remoteUserMem)
      → RoCE: DeepCopyH2DChannelRoce (remoteUserHostMem, remoteUserDeviceMem)
  → H2D copy 整个数组
  → 下 AicpuAclKernelLaunch("RunAicpuIndOpChannelInit")
  → D2H copy channelList 回 Host
  → ReleaseChannelParam (释放所有临时 malloc 内存)
```

关键惯用法:
- channelParamMemVector_: `vector<shared_ptr<DeviceMem>>` 跟踪所有临时 Device 内存
- channelParamMemList_: `list<DeviceMem>` 跟踪 notify list 的 Device 内存
- ReleaseChannelParam 中 free() 释放 malloc 的 host 内存，clear() 释放 DeviceMem

通道复用:
- PrepareHandleArray 检查 channelHandleMap_，已存在则直接复用
- 仅创建不存在的通道，实现幂等性

协议支持:
- COMM_PROTOCOL_HCCS (P2P通道)
- COMM_PROTOCOL_ROCE (RoCE通道)
- 其他协议返回 HCCL_E_PARA

### 核心发现 5: CommMemMgr — 区间树 + 引用计数的内存管理

CommMemMgr 管理两类内存:
1. CCL Buffer: 框架预分配的通信缓冲区(通过 CCLBufferManager)
2. 用户注册内存: 用户通过 HcclCommMemReg 注册的自有内存

用户内存管理的核心设计(comm_mem_manager.h:31-65):
```cpp
using Handle = std::shared_ptr<HcclMemoryHandle>;
using MemKey = hccl::BufferKey<uintptr_t, uint64_t>;
using Table  = hccl::RmaBufferMgr<MemKey, Handle>;

struct TagRegistry {
    Table table;  // 区间树 + ref 语义
};

std::unordered_map<std::string, TagRegistry> tagRegs_;
std::unordered_map<std::string, std::vector<Handle>> opBindings_;
```

这里使用了 RmaBufferMgr(在 platform/resource 层定义的区间树)来做:
- 区间冲突检测: 同 tag 下不允许 overlap 的内存注册
- 引用计数: 同一 (addr, size) 可被多次 Add，Del 时引用计数减1

注册流程(CommRegMem, comm_mem_manager.cc:33-81):
1. 构造 HcclMemoryHandle(shared_ptr)
2. MakeKey: (uintptr_t(addr), uint64_t(size))
3. table.Add(key, handle):
   - 成功 → 新区间
   - 失败但精确匹配 → 复用(ref++)
   - 失败且区间冲突 → 返回 HCCL_E_PARA
4. HcclRegMemAttr 不同时允许更新(WARNING 级别日志)
5. 幂等加入 opBindings_ (检查指针相同性避免重复)

反注册流程(CommUnregMem, comm_mem_manager.cc:85-132):
1. 查找 opBindings_[memTag]
2. erase-remove 模式遍历:
   - 匹配 memHandle → table.Del(key) 引用计数减1
   - try-catch std::out_of_range (防止已删除的 key)
3. 若 tag 下无绑定 → 清理 opBindings_ 和 tagRegs_

### 核心发现 6: CommEngineResMgr — 线程和 Notify 资源管理

CommEngineResMgr 是 ThreadMgr + NotifyManager 的 Facade:

```cpp
// comm_engine_res_manager.h:21-39
class CommEngineResMgr {
    std::unique_ptr<ThreadMgr> threadMgr_;
    std::unique_ptr<NotifyManager> notifyMgr_;
    std::mutex mtx_;
};
```

Init 使用了 lazy initialization + unique_ptr 的幂等模式:
```cpp
// comm_engine_res_manager.cc:16-31
HcclResult CommEngineResMgr::Init(...) {
    std::lock_guard<std::mutex> lock(mtx_);
    if (!threadMgr_) {
        EXECEPTION_CATCH(threadMgr_ = std::make_unique<ThreadMgr>(...), return HCCL_E_PTR);
    }
    if (!notifyMgr_) {
        EXECEPTION_CATCH(notifyMgr_ = std::make_unique<NotifyManager>(...), return HCCL_E_PTR);
    }
    return HCCL_SUCCESS;
}
```

EXECEPTION_CATCH 宏: 捕获 make_unique/make_shared 的 bad_alloc，
在分配失败时执行 fallback 动作(return HCCL_E_PTR)。
出现 29 次，覆盖 6 个文件，是 independent_op 模块特有的惯用法。

ThreadMgr (thread_manager.h:25-58):
- 管理 CPU_TS/AICPU_TS 两种线程
- HcclThreadAcquire: 创建指定数量线程，每线程分配 notifyNumPerThread 个 notify
- HcclThreadAcquireWithStream: 绑定已有 stream 到线程
- HcclThreadExportToCommEngine: 跨 engine 导出线程(AICPU↔CPU)
- threadHandleOthersToCpu_: 维护跨 engine 的 ThreadHandle 映射

NotifyManager (notify_manager.h:21-75):
- 管理硬件同步原语(notify/event)
- NotifyInfo 结构: {CommEngine, NotifyType, isAicpu, NotifyHandle}
- handleBlocks_: `vector<unique_ptr<NotifyHandle[]>>` 托管所有分配的 handle 数组
- 支持 4 种 NotifyType: RESERVED / RTS_NOTIFY / RTS_EVENT / DEVICE_MEM
- CPU engine 只能用 RTS_NOTIFY，AICPU/AIV engine 只能用 DEVICE_MEM

### Design Patterns Summary

1. Facade + Composition (IndependentOp)
   - IndependentOp 聚合 5 个 Manager，每个 Manager 独立管理一类资源
   - 外部通过 GetXxxMgr() 直接访问子管理器，IndependentOp 本身无业务逻辑

2. Strategy (V1/V2 双路径)
   - 每个 C API 函数通过 IsCommunicatorV2() 选择不同的 Manager 持有者
   - V1: IndependentOp 值成员; V2: CollComm 指针成员
   - HCCLV2_FUNC_RUN 宏提供 SoC 级别的策略选择

3. Template Method (RankGraph 继承体系)
   - RankGraph 基类定义接口，RankGraphV1 完整实现，RankGraphV2 委托到外部

4. Bridge/pImpl (RankGraphV2)
   - unique_ptr<Hccl::IRankGraph> pImpl，解耦接口与实现

5. Callback/DI (ManagerCallbacks / ChannelManagerCallbacks)
   - IndependentOp 通过 lambda 注入 callbacks
   - ChannelManager 通过 callbacks 调用上层的建链/rank列表功能
   - 避免了子管理器对上层的直接依赖

6. Registry (ChannelManager 的 handle 管理)
   - channelHandleMap_ / keyMap_ / engineMap_ / channelD2HMap_ 四个 map 构成双向注册表
   - channelKey 做唯一标识，实现 channel 复用

7. Lazy Initialization (CommEngineResMgr::Init)
   - unique_ptr + if(!ptr) + make_unique 模式
   - 配合 EXECEPTION_CATCH 处理 bad_alloc

### Error Handling Patterns

1. C API 函数入口: CHK_PTR_NULL 校验所有参数 → V1/V2分派 → 统一错误处理/日志

2. Manager 内部: CHK_RET 链式传播，lock_guard 保证异常安全

3. 特殊模式 — CHK_PRT_RET 用于条件错误:
   ```cpp
   CHK_PRT_RET(channelNum == 0,
       HCCL_ERROR("[%s]Invalid channelNum, channelNum[%u]", __func__, channelNum),
       HCCL_E_PARA);
   ```

4. 设备内存错误 — 使用 HCCL_E_PTR:
   ```cpp
   CHK_PRT_RET(size && !buffer,
       HCCL_ERROR("[Create][WorkSpace]..."), HCCL_E_PTR);
   ```

5. Destroy 的防御性: DestroyCommEngineCtx 通过反向索引 tagMap_ 查找，
   找不到返回 HCCL_E_PARA 而非 crash

### Anti-Patterns and Code Smells

1. ContextManager 的内存泄漏风险
   文件: independent_op_context_manager.cc:36-60
   CreateCommEngineCtx 中 host 内存用 malloc/free，device 内存用 hrtMalloc/hrtFree。
   但 contextMap_ 析构时没有遍历释放所有 context。
   ContextManager 析构函数为空(`~ContextManager() {}`)。
   如果 ContextManager 在所有 context 被 Destroy 之前销毁，会泄漏内存。
   正确做法: 析构函数中遍历 contextMap_ 释放所有未销毁的 context。

2. getenv() 在每次 API 调用时执行
   文件: hccl_independent_rank_graph.cc (18处), hccl_independent_op_mem.cc (2处), hccl_independent_op_channel.cc (2处)
   每个 HCCLV2_FUNC_RUN + getenv("HCCL_INDEPENDENT_OP") 都在热路径上调用 getenv。
   getenv 不是线程安全的(C标准)，且在高频调用路径上有性能开销。
   正确做法: 启动时读取一次，缓存到 static 变量。

3. static_cast<RankGraphV2*> 向下转型无安全检查
   文件: hccl_independent_rank_graph.cc:260, 282, 305, 328, 349, 373
   ```cpp
   RankGraphV2* rankGraphV2 = static_cast<RankGraphV2*>(rankGraph);
   ```
   RankGraph* 可能是 RankGraphV1 (没有这些方法)，static_cast 不做运行时检查。
   如果调用方传入了 V1 的 comm 句柄，会导致 UB。
   正确做法: dynamic_cast + nullptr 检查，或通过基类虚方法避免向下转型。

4. RegMemMgr 空壳
   文件: reg_mem_manager.h
   完全空的类，只有默认构造/析构。IndependentOp 持有它作为成员但从不使用。
   这是预留接口的占位，但增加了阅读困惑。

5. ChannelManager 缺少整体锁保护
   ChannelManager 的 channelHandleMap_/keyMap_/engineMap_ 等 map 都没有 mutex 保护。
   如果多线程并发调用 ChannelCommCreate/ChannelCommDestroy，会有数据竞争。
   对比: ContextManager 每个方法都有 lock_guard，CommMemMgr 也有 memMutex_/bufferMutex_。

6. 手动 malloc/free 与 DeviceMem::alloc 混用
   文件: channel_manager.cc:296, 535-561
   ParseChannelRemoteDataToMem 中用 malloc 分配 remoteResV2 数组，
   ReleaseChannelParam 中用 free 释放。
   但同一函数中 DeviceMem 通过 RAII 管理。
   两种内存管理风格混用增加了忘记释放的风险。

7. 日志级别不一致
   hccl_independent_op_ctx.cc:76 对 Get 失败用 HCCL_WARNING，
   hccl_independent_op_ctx.cc:43 对 Create 失败用 HCCL_ERROR。
   这两种情况的严重程度可能相同，但用了不同级别。

### Notable Patterns

1. H2D 深拷贝链
   ChannelManager 展示了将复杂嵌套 Host 结构体拷贝到 Device 侧的完整模式:
   - 递归深拷贝每个指针字段
   - 用 channelParamMemVector_ 跟踪所有临时 DeviceMem
   - 最终 ReleaseChannelParam 统一释放
   这是 AICPU kernel 通信的核心模式: Host 准备参数 → 深拷贝到 Device → kernel 执行 → 结果拷回

2. Stream 创建惯用法
   ```cpp
   Stream localStream(StreamType::STREAM_TYPE_ONLINE);
   constexpr u32 aicpuStreamMode = 1;
   CHK_RET(hrtStreamSetMode(localStream.ptr(), aicpuStreamMode));
   ```
   在 IndependentOp::KernelLaunchAicpuCommInit 和 ChannelManager::AicpuChannelInit 中重复出现。
   Stream 是 RAII 包装，离开作用域自动销毁。

3. Kernel Launch 惯用法
   ```cpp
   uint64_t beginTime = hrtMsprofSysCycleTime();
   // ... 准备参数 ...
   CHK_RET(AicpuAclKernelLaunch(stream, param, size, binHandle, kernelName, true, timeOut));
   CHK_RET(hcclStreamSynchronize(stream, timeout));
   HcommProfilingReportKernel(beginTime, profName);
   ```
   固定模式: 记时 → launch → sync → 上报 profiling

4. CommEngine 类型分叉
   整个模块的核心分叉逻辑是 CommEngine 类型:
   - CPU/CPU_TS/CCU → Host 侧资源 (malloc, memcpy_s, RTS_NOTIFY)
   - AICPU/AICPU_TS/AIV → Device 侧资源 (hrtMalloc, hrtMemSyncCopy, DEVICE_MEM)
   这个分叉渗透到 ContextManager/ChannelManager/CommEngineResMgr 所有子系统。

---

## 2.2.1 communicator/ — 主Communicator类 + 子模块 + group/

### Module Overview

本节覆盖 communicator/ 中除 independent_op/ 外的所有内容，包括:
- 核心Communicator类层次(hcclComm / HcclCommunicator / HcclCommunicatorAttrs)
- 8个impl子模块(one_sided_service/ resource_manager/ symmetric_memory/ zero_copy/ aclgraph/ task_abort_handler comm_topo_desc)
- 根目录配置文件(comm_config.cc / comm_configer.cc)
- group/模块(组操作)

模块总规模: ~100文件，37628行。

### Communicator类层次

#### 三层委托链

```
外部 API (HcclAllReduce等)
  ↓ hcclComm* → communicator_->Xxx()
hcclComm (hccl_comm.cc, 3168行)
  持有: shared_ptr<HcclCommunicator> communicator_
  职责: 输入参数校验 + 委托
  ↓
HcclCommunicator (impl/, 9152行host+3168行共享+1640行device)
  持有: HcclCommunicatorAttrs attrCollector_ + 100+成员
  职责: 核心协调(Init/算法选择/流管理/资源编排)
  ↓
HcclCommunicatorAttrs (impl/, 1108行host+705行共享+248行device)
  职责: 拓扑属性提取(rank info/server/superPod/NIC)
```

每层都遵循 host/device 编译分裂模式:
- xxx.cc: 共享实现(构造/析构/简单getter)
- xxx_host.cc: Host侧完整实现
- xxx_device.cc: Device侧stub(返回HCCL_SUCCESS)

佐证: 6个实现文件共16021行。

#### hcclComm: 外部句柄包装

hcclComm是用户可见的通信域句柄(typedef void* HcclComm)对应的内部类:

```cpp
// hccl_comm.cc
struct hcclComm {
    shared_ptr<HcclCommunicator> communicator_;
    DeviceMem indirectInCCLbuffer_, indirectOutCCLbuffer_;
    u64 inCCLbufferSize_, outCCLbufferSize_;
    shared_ptr<hcclKernelPlanner> planner;  // Group操作用
    string identifier_, cclBuffName_;
    bool isHeterogComm_, isResetDevice_, isSpecialType_;
};
```

Host侧方法(hccl_comm_host.cc)按两种模式组织:

1. 集合通信: 参数校验链 + 委托
```cpp
HcclResult hcclComm::AllReduce(tag, input, output, count, dataType, op, stream) {
    CHK_RET(communicator_->CheckCount(count));
    CHK_RET(communicator_->CheckDataType(dataType, true));
    CHK_RET(communicator_->CheckReduceDataType(dataType, op));
    CHK_RET(communicator_->CheckReductionOp(op));
    return communicator_->AllReduce(tag, input, output, count, dataType, op, stream);
}
```

2. 初始化: 多步序列
```cpp
HcclResult hcclComm::InitCollComm(tag) {
    // 加载二进制 → AICPU初始化 → 创建CollComm对象
    CHK_RET(communicator_->LoadBinary());
    CHK_RET(communicator_->InitAicpu());
    CHK_RET(communicator_->CreateCollComm(tag));
}
```

佐证: hccl_comm_host.cc AllReduce方法; hccl_comm.cc 构造函数。

#### HcclCommunicator: God Class (100+方法)

HcclCommunicator是整个communicator模块的核心协调器。头文件(hccl_communicator.h)声明了:

公共接口(100+虚方法):
- 集合通信: AllReduce/AllGather/ReduceScatter/AlltoAll/Broadcast/Scatter/Reduce + OutPlace变体
- 点对点: Send/Receive/BatchSendRecv + OutPlace变体
- 资源管理: CreateCommResource/SetWorkspaceResource/CreateOpBasedResources
- AICPU: AiCpuKernelLaunch/AicpuUnfoldKernelLaunch/Mc2AiCpuStreamAllocAndGet
- 运维: Stop/Resume/Suspend/SwitchNic/Break
- 单边通信: GetOneSidedService/InitOneSidedServiceNetDevCtx
- 零拷贝: InitZeroCopyMemoryAgent/SetMemoryRange/ActivateCommMemory
- 对称内存: RegisterWindow/DeregisterWindow/InitSymmetricMemory
- 图模式: GetConfigAclGraphZeroCopyEnable
- Group: SetGroupMode/SetSendIndex/SetRecvIndex/GroupPrepareStreamAndNotify
- 配置: SetDeterministicConfig/SetAivModeConfig/SetAlgoConfig

私有方法(100+):
- Init序列(20+步): InitNetResource/InitDispatcher/InitStreamManager/InitSocketManager/...
- 资源清理: ReleaseCommInfos/DestroyNetworkResources/DestroyAlgResource/...
- AICPU编排: AicpuKfcTilingDataLaunch/AicpuInitOpTilingDataBuf/...
- 传输管理: SetTransportStatusImpl/ReAllocTransports/...

成员变量(100+):
- 基础属性: userRank_/userRankSize_/deviceType_/devicePhyId_/...
- 子系统: dispatcher_/notifyPool_/callbackTask_/cclBufferManager_/...
- 传输: transportResInfo_/mrManager_/rankInfoList_/...
- 状态: initializedFlag_(atomic_flag)/nicInitialized_(atomic)/...
- 缓存: hcclCacheMap_(OpParam→HcclCacheInfo)/tagCommInfo_/tagStreamInfo_/...

佐证: hccl_communicator.h 行108-900; hccl_communicator_host.cc 9152行。

#### HcclCommunicatorAttrs: 拓扑属性提取

HcclCommunicatorAttrs封装了从RankTable中提取拓扑属性的逻辑:

```
Init(params, rankTable)
  ↓ InitCommParams → 基础参数
  ↓ InitRankInfo(rankTable) → rank列表/NIC列表/拓扑类型
    ↓ SetServerId → serverId_
    ↓ SetServerNum → serverNum_
    ↓ SetModuleInfo → moduleNum_
    ↓ SetSuperPodInfo → superPodNum_/superPodId_
    ↓ InitTopoInfo → topoInfoParse_/pairLinkInfo_/meshAggregationRankSize_
    ↓ SetNiclistInfo → nicList_
    ↓ GenUsedRdmaLevel0 → isUsedRdmaLevel0_
    ↓ GenSupportRdmaLite → isSupportRdmaLite_
```

60+个Getter方法提供只读属性访问(GetServerNum/GetSuperPodId/GetDevicePhyId/...)。
HcclCommunicator通过 `attrCollector_.GetXxx()` 读取这些属性。

佐证: hccl_communicator_attrs.h 全文(236行); hccl_communicator_attrs_host.cc InitTopoInfo()。

### Sub-module: one_sided_service/ (单边RDMA通信)

8个文件，三层设计:

```
IHcclOneSidedService (interface)
  ├── Config/DeInit/SetNetDevCtx/GetNetDevCtx
  ├── socketManager_ + notifyPool_ (引用)
  ├── netDevCtx: RDMA和IPC双通道
      ↓
HcclOneSidedService (service)
  ├── Prepare(): fullmesh连接(thread-per-connection)
  ├── RegMem/DeregMem: 远程内存注册(最大256)
  ├── BatchPut/BatchGet: AICPU kernel编排
  ├── oneSidedConns_ map / desc2HcclBufMap (IPC/RoCE分离)
  ├── hasErrorFlag_ / hasTimeoutErrorFlag_ (atomic error propagation)
      ↓
HcclOneSidedConn (per-connection)
  ├── Connect(): 确定性tag(smaller rank=SERVER)
  ├── BatchWrite/BatchRead: 逐descriptor RDMA操作
  ├── socket_ / rdmaSocket_ / transportMemPtr_
```

C API层(one_sided_service_adapt.cc):
- EXCEPTION_HANDLE_BEGIN/END包裹每个函数
- HCCLV2_FUNC_RUN(V2函数)在顶部做V2分派
- RPT_INPUT_ERR + CHK_PTR_NULL双重校验

V2层(one_sided_service_adapt_v2.h):
- 所有V2函数`__attribute__((weak))`声明
- 链接V2库时覆盖，未链接时为nullptr

注意: 全局static `g_launchStream`/`g_launchMutex` 管理AICPU launch流。
注意: 析构器中 `do-while(HCCL_E_AGAIN)` 循环重试HcclMemDereg。
注意: ProcessInfo{pid, sdid, serverId} 用于RDMA连接的端点标识。

佐证: i_hccl_one_sided_service.h/hccl_one_sided_service.h/hccl_one_sided_conn.h;
one_sided_service_adapt.cc EXCEPTION_HANDLE模式。

### Sub-module: resource_manager/ (传输链路管理)

8个文件:

1. TransportManager (transport_manager.h/.cc, ~78KB):
   - 核心类: 20+构造函数参数
   - TransportData struct + std::hash特化 → unordered_map<TransportData, LINK>实现链路复用
   - SubCommLinkPara: RAII析构中join线程
   - 当连接数超过MASSIVE_IBV_CONNECTION_COUNT(1000)时切换链路类型
   - Copy/move显式deleted

2. MulQpInfo (multi_qpInfo_manager.h/.cc):
   - Strategy Pattern: 4个Cache子类(DevCfg/DevNslb/EnvConfigPath/EnvPerConnection)
   - 优先级队列: init时按优先级入队，查询时第一个可用胜出
   - EnvConfigPathCache: 文件配置 + IP对匹配 + 0.0.0.0/::通配
   - Thread-safe: initLock_ mutex

3. PreemptPortManager (preempt_port_manager.h/.cc):
   - Singleton-per-device: `static PreemptPortManager instance[MAX_MODULE_DEVICE_NUM]`
   - 引用计数端口复用: 同IP多communicator共享端口
   - EnumClassHash: enum class做unordered_map key的helper
   - 端口扫描: 顺序遍历端口范围找空闲

佐证: transport_manager.h TransportData+hash; multi_qpInfo_manager.h MulQpInfoCacheBase继承体系;
preempt_port_manager.cc GetInstance()。

### Sub-module: symmetric_memory/ (VMM对称虚拟内存)

4个文件:

SymmetricMemory (symmetric_memory.h/.cc):
- VMM-based对称VA空间: targetStartTB=40TB固定起始地址
- 每rank stride大小的VA段，总空间=stride*rankSize
- SimpleVaAllocator (Pimpl): 内部free-list分配器(对齐+合并+重叠检测)
- PaMappingInfo: PA引用计数，支持PA句柄复用
- SymmetricWindow: 窗口描述符，含devWin(设备侧副本)
- std::once_flag懒初始化
- RegisterInternal中用goto INTERNAL_ERROR/MAP_ERROR做清理(非RAII)

SymmetricMemoryAgent (symmetric_memory_agent.h/.cc):
- Ring-based allgather: 左邻居接收，右邻居发送，转发非本地数据
- Packet: {MsgType, rankId, data[144]}
- Background thread + condition_variable完成通知
- WaitForCollectionComplete: CV wait + configurable timeout
- DealWithRequest: SaluSleep(1000) spin-wait

佐证: symmetric_memory.h targetStartTB/SimpleVaAllocator/PaMappingInfo;
symmetric_memory_agent.cc DealWithRequest。

### Sub-module: zero_copy/ (零拷贝地址管理)

7个文件:

ZeroCopyAddressMgr (zero_copy_address_mgr.h/.cc):
- AddressRange: 自定义operator<将重叠范围视为相等(set-based lookup)
- RingBuffer: Host→Device地址更新同步
- ZeroCopyReserveAddrMap: 嵌套map devicePhyId→remoteAddr→LocalIpc2RemoteAddr
- ProcessOneAddrMap: switch on ZeroCopyItemType(SET/UNSET/ACTIVATE/DEACTIVATE)
- weak symbol分裂: InitRingBuffer/PushOne
  - zero_copy_address_mgr_host.cc: 完整实现(device内存分配+H2D/D2H拷贝)
  - zero_copy_address_mgr_device.cc: 空实现(no-op)

ZeroCopyMemoryAgent (zero_copy_memory_agent.h/.cc, ~54KB):
- 异步通信agent: RequestType enum含ACK对(SET/SET_ACK, ACTIVATE/ACTIVATE_ACK等)
- ZeroCopyMemoryAgentSendMgr: 2-slot队列(ACK优先于Request)
- ZeroCopyMemoryAgentRecvMgr: 双缓冲接收
- static addressMgr_: 所有实例共享地址管理器
- Snapshot pause/resume支持
- ConstructData模板做byte buffer序列化
- Background InnerThread + barrier-based close protocol

佐证: zero_copy_address_mgr.h AddressRange/RingBuffer;
zero_copy_memory_agent.h RequestType/SendMgr/RecvMgr。

### Sub-module: aclgraph/ (图模式零拷贝)

2个文件:

ZeroCopyAclGraph (zero_copy_acl_graph.h/.cc):
- 只支持DEV_TYPE_910_93
- algoSet_: 白名单过滤支持的操作
- tagResourceIndex_: atomic计数器生成unique tag
- 临时切换workflow mode进行算法选择
- 反模式: `if (!isReduceOps == true)` 混淆的双重否定

佐证: zero_copy_acl_graph.cc DEV_TYPE_910_93检查。

### Sub-module: task_abort_handler (任务中止)

3个文件:

TaskAbortHandler:
- Static Init/DeInit接口
- 全局static: commVector(所有comm列表)/ref_(引用计数)/mutex_
- Init: 第一个comm注册回调到runtime; DeInit: 最后一个comm注销
- ProcessTaskAbortHandleCallback: C回调函数签名
- 两阶段abort: PRE→Suspend, POST→StopExec+Clean+Stop
- TaskAbortResult: Success/Fail/TimeOut
- 超时/非超时分支有大量重复代码

佐证: task_abort_handler.cc全文。

### Sub-module: comm_topo_desc (拓扑缓存)

2个文件:

CommTopoDesc:
- Thread-safe singleton: 缓存per-comm的rankSize和L0 topo type
- Simple mutex-protected map: Save/Get操作
- Delete copy constructor/assignment

佐证: comm_topo_desc.h/.cc。

### Root-level: comm_config.cc + comm_configer.cc

CommConfig (comm_config.cc):
- 版本化配置加载: magic word验证 + 级联版本检查(v1→v10)
- v1: bufferSize/deterministic
- v2: commName
- v3-v4: netDevCtxNum/commBufferSize
- v5: hcclAlgo (per-optype算法配置，支持"others"通配)
- v6: retryEnable ("L1:0,L2:0"字符串解析)
- v7: opExpansionMode (HOST/AICPU/AIV/ONLY_AIV)
- v8: aclGraphZeroCopyEnable
- v9: mc2_communication_config
- v10: QoS/symmetricMemoryStride

CommConfiger (comm_configer.cc):
- Singleton: per-communicator配置管理(timeout/algo/retry)
- 一致模式: lock→find identifier→fallback to env config
- UnRegisterToCommConfiger: check initialized_ flag before lock(析构顺序安全)

佐证: comm_config.cc SetConfigByVersion(); comm_configer.cc。

### group/ (组操作)

3个文件:

hcclKernelPlanner (hccl_group_utils.h):
- 组操作的调度计划器
- peers[rankSize]: 每peer的send/recv队列
- collTaskQueue: 集合通信任务队列
- sendStreamTasks/recvStreamTasks: 分流后的P2P任务(MAX_CONCURRENT=16)
- sendIdx2Byte/recvIdx2Byte: 每流的累计字节数
- hcclOpInfo: 统一的算子入参结构(coll/sendbuff/recvbuff/count/type/op/root/comm/stream)

hcclAsyncJob (hccl_group_utils.h):
- 异步任务节点(链表结构)
- 派生: hcclCommInitAsyncJob / hcclCommInitConfigAsyncJob / hcclCommDestroyAsyncJob
- 每个job含: thread(unique_ptr)/func(函数指针)/state/result/mtx

HcclGroupStart/HcclGroupEnd (hccl_group.cc):
- 嵌套计数: hcclGroupDepth++/--
- 只有最外层GroupEnd触发执行
- groupLaunch: asyncJobLaunch(初始化jobs) → doLaunches(通信操作) → 流同步

P2P调度算法:
```
SortSendTasks: 从myRank开始顺时针遍历所有peer的sendQueue
SortRecvTasks: 从myRank开始逆时针遍历所有peer的recvQueue
doLaunches: 每轮从16个流中取一个task，按rank号决定send/recv顺序
  myRank < peer → 先send后recv
  myRank >= peer → 先recv后send
syncThread: 并行执行GroupSyncMainstream(stream同步)
```

佐证: hccl_group.cc全文(492行)。

### Idioms Summary (惯用法汇总)

| ID | 惯用法 | 出现次数/位置 | 类型 |
|----|--------|---------------|------|
| IC-1 | 三层委托(hcclComm→HcclCommunicator→子模块) | 整个模块 | 架构级 |
| IC-2 | Host/Device编译分裂(xxx.cc/xxx_host.cc/xxx_device.cc) | 6组文件 | 项目级 |
| IC-3 | CHK_RET链式错误检查 | 2758处/100文件 | 项目级强制 |
| IC-4 | EXCEPTION_HANDLE_BEGIN/END | 34处(C API层) | 公共API |
| IC-5 | weak symbol版本化 | 9处 | V2/设备分裂 |
| IC-6 | 版本化配置加载(v1-v10) | comm_config.cc | 模块级 |
| IC-7 | RPT_INPUT_ERR结构化错误报告 | 46处 | 公共API |
| IC-8 | static_cast优先 | 783处 | 项目级 |
| IC-9 | EXECEPTION_CATCH(bad_alloc处理) | 29处 | independent_op |
| IC-10 | 自定义Hash+Equal做unordered_map key | 5+处 | 模块级常见 |
| IC-11 | atomic_flag做初始化守卫 | HcclCommunicator | 模块级 |
| IC-12 | ALGCFG_TO_NAME字符串到Executor映射 | hccl_communicator.cc | 模块级 |

### Anti-Patterns Summary (反模式汇总)

| ID | 反模式 | 位置 | 影响 |
|----|--------|------|------|
| AP-1 | God Class(100+方法) | HcclCommunicator | 可维护性 |
| AP-2 | 缺失break(fall-through) | hccl_group.cc:383 | 功能bug |
| AP-3 | Group全局可变状态无锁 | hccl_group.cc:18-21 | 线程安全 |
| AP-4 | usleep轮询 | hccl_group.cc:180 | 性能 |
| AP-5 | TaskAbort全局static+重复代码 | task_abort_handler.cc | 可维护性 |
| AP-6 | goto做错误清理 | symmetric_memory.cc | C++风格 |
| AP-7 | SaluSleep spin-wait | symmetric_memory_agent.cc | 性能 |
| AP-8 | operator<违反strict weak ordering | zero_copy_address_mgr.h | 理论UB |
| AP-9 | 双重否定(!x == true) | zero_copy_acl_graph.cc | 可读性 |
| AP-10 | 参数超5个(自标注) | hccl_one_sided_conn.h | 接口设计 |
| AP-11 | 构造函数初始化列表重复40行 | hccl_communicator.cc | DRY违反 |
| AP-12 | ContextManager析构不释放 | independent_op_context_manager | 内存泄漏 |
| AP-13 | getenv热路径调用 | hccl_independent_rank_graph | 线程安全+性能 |
| AP-14 | static_cast向下转型无检查 | hccl_independent_rank_graph | 类型安全 |
| AP-15 | ChannelManager无锁保护 | channel_manager | 线程安全 |

### Architecture Patterns Summary (架构模式汇总)

| Pattern | 位置 | 说明 |
|---------|------|------|
| Composition | HcclCommunicator | 聚合10+子系统(非继承) |
| Three-layer Inheritance | one_sided_service | Interface→Service→Connection |
| Strategy (4-backend) | MulQpInfo | 优先级队列选择配置源 |
| Singleton-per-Device | PreemptPortManager | static数组[MAX_DEVICE_NUM] |
| Ring Allgather | SymmetricMemoryAgent | 环形handle交换 |
| NCCL-style Group | group/ | GroupStart/End+嵌套+批量调度 |
| Hash-based Reuse | TransportManager | TransportData hash→link复用 |
| VMM Symmetric VA | SymmetricMemory | 40TB起始+per-rank stride |
| Custom operator< | ZeroCopyAddressMgr | 重叠=相等的set查找 |
| Facade+Composition | IndependentOp | 5个Manager聚合器 |
| Bridge/pImpl | RankGraphV2 | 委托到IRankGraph |
| Double-buffer Async | ZeroCopyMemoryAgent | 2-slot send + 双缓冲recv |

### Domain Knowledge Summary

- 通信域两层: World Group(全局) / Sub Group(子集)，通过WorldGroupInfo共享全局拓扑
- Rank拓扑属性(60+): 直接驱动算法选择和transport创建
- 算法配置: `OpType=levelN:algorithm` 格式，字符串→Executor映射
- 单边通信: Put/Get语义，fullmesh连接，MemDesc描述符交换
- 零拷贝: 用户内存直接通信，address range管理+异步同步
- 对称内存: VMM统一VA空间，消除运行时地址交换
- 版本化配置(v1-v10): 每个版本对应一个功能上线
- Group语义: 类NCCL的GroupStart/GroupEnd批量调度

### Hardware Knowledge Summary

- 设备类型决定RDMA边界: 910B=server, 910_93=superPod
- 链路类型层次: HCCS > SIO > PCIe > RoCE
- 910_93是主力目标: ZeroCopyAclGraph/SuperPod/DoubleRing等专属优化
- NIC部署: Device侧(default) vs Host侧，影响端口和IP选择
- 40TB VA起始: 对称内存的固定虚拟地址预留

---

## 2.2.1 Legacy Communicator Deep Dive

### Module Overview

`src/legacy/framework/communicator/` 是hcomm legacy层的核心模块(59个文件)，
包含三个子目录:
- 根目录(23个文件): 主通信域管理(CommunicatorImpl)、HDC通道、CCU Super Fast Load、故障恢复等
- aicpu/(21个文件): AICPU侧communicator(CommunicatorImplLite)、kernel入口、设备侧资源管理
- hostdpu/(15个文件): DPU执行路径、TaskService、RDMA Flush管理

本模块的职责:
1. 管理通信域(communicator)的完整生命周期: 创建、初始化资源、下发算子、故障恢复、销毁
2. 协调Host/Device/DPU三端的通信执行
3. 提供CCU Super Fast Load缓存加速
4. 管理HDC(Host-Device Communication)通道用于KFC协议

### Key Files

核心类:
- `communicator_impl.h/.cc` — CommunicatorImpl，God Object，~3200行实现，管理20+子系统
- `hccl_communicator.cc` — HcclCommunicator(legacy)，Pimpl门面，委托到CommunicatorImpl
- `hccl_communicator.h`(legacy/include/) — HcclCommunicator公共接口，unique_ptr pimpl

HDC通道:
- `hdc.h/.cc` — HDCommunicate，host侧共享内存通道，head_cnt/tail_cnt一致性协议
- `hdc_param.h` — Direction常量(H2D=1, D2H=0)和HDCommunicateParams结构体

CCU加速:
- `ccu_super_fast_load.h/.cc` — CcuSFLMappingKey、CachedCCUParams、SuperFastLoad()

故障恢复:
- `diff_rank_updater.cc` — DiffRankUpdater，64+1替换和Pod替换两种策略

参数检查:
- `op_params_checker.h/.cc` — DataTypeBitmap、OpType x DataType验证矩阵

辅助:
- `comm_topo_desc.h/.cc` — Singleton缓存rankSize和L0 topoType
- `op_task_config.h` — QoS配置和notify等待时间(default 27*68=1836)
- `hccl_params_pub.cc` — CollOpParams::Describe()格式化输出

AICPU子目录:
- `aicpu/communicator_impl_lite.h/.cc` — CommunicatorImplLite，设备侧communicator
- `aicpu/communicator_impl_lite_manager.cc` — Singleton管理map<u32, unique_ptr<CommunicatorImplLite>>
- `aicpu/kernel_entrance.cc` — extern "C" HcclKernelEntrance，AICPU kernel入口
- `aicpu/hdc_lite.h/.cc` — HDCommunicateLite，设备侧HDC镜像
- `aicpu/kfc.h` — KfcCommand/KfcStatus/KfcExecStatus枚举定义

HostDPU子目录:
- `hostdpu/task_service.h/.cc` — TaskService，共享HBM轮询服务
- `hostdpu/dpu_kernel_entrance.cc` — extern "C" RunDpuRpcSrvLaunch，DPU kernel入口
- `hostdpu/flush_handle.h/.cc` — FlushHandle，RDMA loopback QP + MR资源
- `hostdpu/flush_manager.h/.cc` — FlushManager，Singleton管理RDMA Flush

### Legacy核心发现 1: CommunicatorImpl — God Object模式

CommunicatorImpl是整个legacy communicator的核心，也是最大的代码异味。

文件: communicator_impl.h (585行), communicator_impl.cc (~3200行)

类成员统计:
- 20+ unique_ptr子系统成员(DataBufManager, StreamManager, QueueNotifyManager, RmaConnManager, MemTransportManager, MirrorTaskManager, ProfilingReporter, HDCommunicate x2, CollAlgComponent等)
- 10+ shared_ptr buffer成员(cclBuffer, aivTagBuffer, inCclBuffer, outCclBuffer, barrierInMemory等)
- 5+ 标志位(initFlag, devModeFlag, isSuspended, isCleaned, isAicpuKernelLaunched, isDpuKernelLaunched等)
- 4个Init()重载 + 28个InitXxx私有方法
- 50+ include头文件

God Object特征(communicator_impl.h:393-418):
```cpp
unique_ptr<DataBufManager>                 dataBufferManager;
unique_ptr<LocalRmaBufManager>             localRmaBufManager;
unique_ptr<RemoteRmaBufManager>            remoteRmaBufManager;
unique_ptr<QueueNotifyManager>             queueNotifyManager;
unique_ptr<QueueWaitGroupCntNotifyManager> queueWaitGroupCntNotifyManager;
unique_ptr<QueueBcastPostCntNotifyManager> queueBcastPostCntNotifyManager;
unique_ptr<ConnLocalNotifyManager>         connLocalNotifyManager;
unique_ptr<ConnLocalCntNotifyManager>      connLocalCntNotifyManager;
unique_ptr<StreamManager>                  streamManager;
unique_ptr<AicpuStreamManager>             aicpuStreamManager;
unique_ptr<SocketManager>                  socketManager;
unique_ptr<RmaConnManager>                 rmaConnectionManager;
unique_ptr<MemTransportManager>            memTransportManager{};
unique_ptr<MirrorTaskManager>              mirrorTaskManager;
unique_ptr<UbMemoryTransportMgr>           ubMemoryTransportMgr{};
unique_ptr<ProfilingReporter>              profilingReporter;
unique_ptr<HDCommunicate>                  kfcControlTransferH2D;
unique_ptr<HDCommunicate>                  kfcStatusTransferD2H;
unique_ptr<HcclOneSidedService>            oneSidedService;
unique_ptr<CcuStreamSyncNotifyManager>     ccuStreamSyncNotifyManager;
```

29步初始化序列(communicator_impl.cc:124-159):
```cpp
void CommunicatorImpl::InitCommResource(const CommParams &commParams)
{
    HrtSetDevice(devLogicId);       // 0. 设置当前设备
    InitHccpHdc();                  // 1. HCCP HDC通道
    // if (IsNeedDpu()) InitHccpPeer();  // 2. DPU peer模式(条件)
    AppendLocalDieIdForLinks();     // 3
    InitCcuSuperFastLoad();         // 4
    InitNotifyManager();            // 5
    InitStreamManager();            // 6
    InitSocketManager();            // 7
    // SocketManager::SetDeviceServerListenPortMap  // 8(条件)
    InitRmaConnManager();           // 9
    InitDataBufferManager();        // 10
    InitNotifyFixedValue();         // 11
    InitMemTransportManager();      // 12
    InitHostDeviceSyncNotifyManager(); // 13
    InitUbMemoryTransportMgr();     // 14
    CollAlgComponentInit();         // 15 (Builder模式)
    InitCollService();              // 16 (3个Service)
    InitTraceManager();             // 17
    DlProfFunction::...Init();      // 18
    InitMirrorTaskManager();        // 19
    InitProfilingReporter();        // 20
    InitTaskExceptionHandler();     // 21
    InitHDCommunicate();            // 22 (H2D + D2H通道)
    notifyTimeoutCfg.Init();        // 23
    status = CommStatus::COMM_READY; // 24
    SnapShotParser::...Serialize...(); // 25
    InitOneSidedService();          // 26
    RegisterKernel();               // 27
    InitDpuKernel();                // 28
}
```

Pimpl门面层(hccl_communicator.cc):
```cpp
// 构造: 创建pimpl + 注册TaskAbortHandler
HcclCommunicator::HcclCommunicator(const CommParams &commParams) {
    pimpl = std::make_unique<CommunicatorImpl>();
    RegistTaskAbortHandler();
}
// 析构: DECTOR_TRY_CATCH宏吞异常
~HcclCommunicator() {
    DECTOR_TRY_CATCH("HcclCommunicator", { UnRegistTaskAbortHandler(); pimpl = nullptr; ... });
}
// 所有方法: 纯委托
HcclResult Init(const std::string &ranktableM) { return pimpl->Init(commParams, ranktableM, config); }
```

### Legacy核心发现 2: HDC (Host-Device Communication) 机制

HDC是Host和Device之间的单向共享内存通道，用于KFC协议。

内存布局(hdc.cc:31-43):
```
+---------------------+
|      content        |
+---------------------+
|    head_cnt[u32]    |
+---------------------+
|    tail_cnt[u32]    |
+---------------------+
```

一致性协议:
- 写入方(Put/Write): head_cnt++ → 写content → tail_cnt++ (三步)
- 读取方(Get/Read): 比较cache中的tail_cnt与实际tail_cnt，不一致则UpdateCache
- UpdateCache: cache tail → cache data → cache head，然后检查head==tail，不等则重试
- 超时: 默认10秒

编译器优化阻止(hdc.cc:104-105):
```cpp
#pragma GCC push_options
#pragma GCC optimize("O0")    // 防止编译器重排序head/tail/content
// Put, Get, Read, Write, UpdateCache 全部在O0区域内
#pragma GCC pop_options        // 行262
```

两种传输路径(hdc.h:60):
```cpp
bool supportDevMemReg{ true }; // PCIe BAR映射(高性能) vs drvMemcpy(回退)
```
- supportDevMemReg=true: Device内存映射到Host地址空间，memcpy_s直接操作
- supportDevMemReg=false: 需要调用HrtDrvMemCpy走驱动路径

初始化(communicator_impl.cc:1383-1391):
```cpp
void CommunicatorImpl::InitHDCommunicate() {
    kfcControlTransferH2D = make_unique<HDCommunicate>(devLogicId, HCCLV2_HDC_TYPE_H2D, sizeof(KfcCommand));
    kfcControlTransferH2D->Init();
    kfcStatusTransferD2H = make_unique<HDCommunicate>(devLogicId, HCCLV2_HDC_TYPE_D2H, sizeof(KfcExecStatus));
    kfcStatusTransferD2H->Init();
}
```
H2D通道传输KfcCommand(控制命令), D2H通道传输KfcExecStatus(状态回报)。

设备侧镜像(aicpu/hdc_lite.h/.cc):
HDCommunicateLite是HDCommunicate的设备侧对应物，使用相同的一致性协议，
但简化了实现(无supportDevMemReg路径，直接memcpy_s)。

### Legacy核心发现 3: KFC协议 — Host与AICPU的命令/状态交互

KFC定义了Host和AICPU kernel之间的通信协议。

命令枚举(aicpu/kfc.h):
```cpp
enum class KfcCommand : u32 {
    NONE = 0,              NS_STOP_LAUNCH = 1,      // 暂停算子下发
    NS_CLEAN = 2,          DESTROY_AICPU_COMM = 3   // 销毁通信域
};
enum class KfcStatus : u32 {
    NONE = 0,              STOP_LAUNCH_DONE = 1,
    CLEAN_DONE = 2,        DESTROY_AICPU_COMM_DONE = 3,
    ERROR = 4
};
```

NsRecovery故障恢复协议流程:

Suspend (communicator_impl.cc:1775-1812):
1. 检查isSuspended防止重复暂停
2. 通过H2D通道发送NS_STOP_LAUNCH
3. 轮询D2H通道等待STOP_LAUNCH_DONE(超时返回HCCL_E_TIMEOUT)

Clean (communicator_impl.cc:1814-1892):
1. 前置: 必须已Suspend
2. Host侧: CCU路径→CcuTransportMgr.Clean(), 非CCU→rmaConnectionManager+memTransportManager.Clear()
3. Device侧: 等STOP_LAUNCH_DONE → 发NS_CLEAN → 等CLEAN_DONE

Resume (communicator_impl.cc:1894-1921):
1. 不能是COMM_ERROR
2. collService->Resume()
3. 重置flags (isSuspended=false, isCleaned=false)
4. HOST场景不支持Resume

销毁(communicator_impl.cc:2392-2410):
```
Host发DESTROY_AICPU_COMM → 轮询等DESTROY_AICPU_COMM_DONE → 超时处理
```

### Legacy核心发现 4: CCU Super Fast Load — 参数缓存加速

CCU Super Fast Load缓存CCU算子参数以避免重复编排。

数据结构(ccu_super_fast_load.h):
- CcuSFLMappingKey = array<u32, 3> (三元素缓存键)
- CachedCCUParams: posix_memalign分配，move-only，存储rtCcuTaskInfo_t数组

缓存键构建(communicator_impl.cc:427-446):
```
ALLTOALL:   {reduceOp, sendType, sendCount}
BROADCAST:  {root, dataType, count}
其他:       {reduceOp, dataType, count}
```
支持的OpType: ALLREDUCE, ALLGATHER, REDUCESCATTER, BROADCAST, ALLTOALL

缓存碰撞处理(communicator_impl.h:342-359):
保守策略 — 同一key如果产生不同参数(碰撞):
1. 从缓存移除: `ccuParamsMapping.erase(ccuParamsMappingKey)`
2. 加入永久黑名单: `ccuParamsNotCacheKey.emplace(ccuParamsMappingKey)`
后续该key永不再缓存。

LoadOpbasedCollOp中的快速路径(communicator_impl.cc:609-628):
```cpp
if (rankSize == 1) { SingleRankProc(...); return SUCCESS; }  // 单rank快速路径
if (TryFastCcuLaunch(opParams, stream)) { return SUCCESS; }  // CCU缓存命中
// 正常路径: 算法选择 → Service选择 → collService->LoadWithOpBasedMode
```

### Legacy核心发现 5: AcceleratorState — 6种执行模式路由

初始化3种Service(communicator_impl.cc:1346-1365):
```cpp
auto ccuCollService   = make_shared<CollServiceDeviceMode>(this);   // CCU/AIV
auto aiCpuCollService = make_shared<CollServiceAiCpuImpl>(this);    // AICPU
auto hostCollService  = make_shared<CollServiceDefaultImpl>(this);  // Host图模式

collServices[AIV]        = ccuCollService;   // host展开 + AIV执行
collServices[AIV_ONLY]   = ccuCollService;
collServices[CCU_MS]     = ccuCollService;   // host展开 + CCU执行
collServices[CCU_SCHED]  = ccuCollService;
collServices[AICPU_TS]   = aiCpuCollService; // AICPU展开和执行
collServices[HOSTCPU_TS] = hostCollService;  // host展开 + host执行(910_95不支持)
```

SetAccelerator约束(communicator_impl.cc:2608-2650):
- isLoadOp后禁止修改(已下发过算子)
- 标卡(MAINBOARD_PCIE_STD)不支持CCU_MS
- HOSTCPU_TS在910_95上不支持
- CCU_MS不可用时回退到CCU_SCHED

### Legacy核心发现 6: AICPU Communicator (CommunicatorImplLite)

CommunicatorImplLite是CommunicatorImpl的设备侧对应物，运行在AICPU上。

Host ↔ Device对比:

| 方面 | CommunicatorImpl (Host) | CommunicatorImplLite (AICPU) |
|------|------------------------|------------------------------|
| 继承 | 无基类 | ResMgrFetcher |
| 子系统 | 20+ unique_ptr | 10+ "Lite"版管理器 |
| 资源来源 | 直接创建 | 序列化数据反序列化恢复 |
| 执行 | 选择CollService下发 | 直接UnfoldOp展开 |
| 生命周期 | 外部创建销毁 | Singleton懒加载 |
| 实现行数 | ~3200 | ~400 |

资源恢复:
```
Host打包 → HDC传输 → AicpuResPackageHelper.ParsePackedData → 恢复7类资源:
StreamLite / QueueNotifyLite / Cnt1tonNotifyLite / CntNto1NotifyLite /
ConnectedLink / HostDeviceSyncNotifyLite / MemTransportLite
```

CommunicatorImplLiteManager:
- Singleton管理 map<u32, unique_ptr<CommunicatorImplLite>>
- 构造函数注册daemon service + 启动后台线程
- Get()按idIndex懒加载

AICPU Kernel入口(kernel_entrance.cc):
```cpp
extern "C" uint32_t HcclKernelEntrance(void *args)       // 算子执行
extern "C" uint32_t HcclUpdateCommKernelEntrance(void *args) // NsRecovery更新
```

### Legacy核心发现 7: DPU执行路径

DPU(Data Processing Unit)用于Host网卡场景的集合通信。

NPU/DPU上下文切换(communicator_impl.cc:3104-3147):
```cpp
aclrtGetCurrentContext(&npuContext);       // 保存NPU ctx
rtSetXpuDevice(RT_DEV_TYPE_DPU, 0);       // 切到DPU
aclrtGetCurrentContext(&dpuContext);       // 获取DPU ctx
// ... DPU操作(PrepareDpuKernelResource + LaunchDpuKernel) ...
aclrtSetCurrentContext(npuContext);        // 切回NPU
```

TaskService — DPU侧共享HBM轮询(hostdpu/task_service.h/.cc):
```
共享HBM布局(每半100MB):
  NPU->DPU: [flag(1B) | taskType(1B) | msgId(4B) | data(...)]
  DPU->NPU: [flag(1B) | taskType(1B) | msgId(4B) | data(...)]

Flag: TASK_UNSET(0)=空闲, TASK_OK(1)=有任务, TASK_TERMINATE(2)=退出
TaskRun(): 无限轮询flag，分发到注册callback
```

DPU终止(communicator_impl.cc:2358-2390):
写flag=2(TERMINATE) → 轮询等flag=3(确认) → 超时10秒

### Legacy核心发现 8: RDMA Flush机制

FlushManager/FlushHandle确保PCIe排序一致性。

设计:
Host NIC通过RDMA Write写入Device HBM后，需确保PCIe写入对Device可见。
通过loopback QP的RDMA Read利用Read ordering语义强制刷新。

FlushHandle初始化(hostdpu/flush_handle.h/.cc):
```
GetRdmaHandle → AllocHostMem → AllocDeviceMem →
CreateLoopbackQp → RegisterLocalMr → RegisterRemoteMr
```

FlushManager(hostdpu/flush_manager.h/.cc):
- Singleton, 按IP管理FlushHandle
- Flush(): 对每个loopback QP执行RDMA READ
- ExecuteRdmaRead: ibv_post_send + ibv_poll_cq, 30秒超时

### Legacy核心发现 9: DiffRankUpdater — 故障恢复Rank Table更新

两种替换策略(diff_rank_updater.cc):
- 64+1替换(Check64Plus1Replace): 同一R0 ID下只有1个rank变化
- Pod替换(CheckPodReplace): 同一R0 ID下变化数等于snapshot中rank数

决策: changeCount==0→跳过, ==1→64+1, ==snapshotSize→Pod, 其他→错误

### Legacy核心发现 10: 三层异常捕获模式

```cpp
// communicator_impl.cc:98-122 (Init), 611-697 (LoadOpbasedCollOp)
try {
    // 业务逻辑
} catch (HcclException &e) {           // 项目自定义异常(含错误码+backtrace)
    HCCL_ERROR(e.what()); PrintBackTrace(e); return e.GetErrorCode();
} catch (exception &e) {               // 标准C++异常
    HCCL_ERROR(e.what()); return HCCL_E_INTERNAL;
} catch (...) {                         // 未知异常
    HCCL_ERROR("Unknown error occurs!"); return HCCL_E_INTERNAL;
}
```

变体:
- NsRecovery方法: TRY_CATCH_RETURN宏包装
- 析构: DECTOR_TRY_CATCH宏(吞异常不传播)

### Legacy核心发现 11: CommStatus状态机

```
COMM_IDLE ─Init()→ COMM_READY ─LoadOp开始→ COMM_INUSE ─LoadOp结束→ COMM_READY
                                                                     ↓(异常)
                                                                  COMM_ERROR
```

CheckCommStatus(communicator_impl.cc:700-712):
```cpp
if (status == COMM_ERROR) return HCCL_E_INTERNAL;
if (isSuspended) return HCCL_E_SUSPENDING;
return HCCL_SUCCESS;
```

### Legacy Design Patterns

| Pattern | 位置 | 说明 |
|---------|------|------|
| Pimpl | HcclCommunicator→CommunicatorImpl | unique_ptr持有，纯委托 |
| Strategy | collServices map | AcceleratorState→3种CollService |
| Builder | CollAlgComponentBuilder | 链式.Set().Build() |
| Command | KFC协议 | KfcCommand枚举+HDC传输+polling |
| State | CommStatus | IDLE→READY→INUSE→READY |
| Singleton | FlushManager/LiteManager/CommTopoDesc | GetInstance() |
| Mirror/Lite | Host↔Device | CommunicatorImpl↔CommunicatorImplLite, HDCommunicate↔HDCommunicateLite |

### Legacy Anti-Patterns

| ID | 反模式 | 位置 | 说明 |
|----|--------|------|------|
| LAP-1 | God Object | communicator_impl.h | 20+子系统, 585行头文件, ~3200行实现 |
| LAP-2 | #pragma GCC optimize("O0") | hdc.cc:104-105 | 应用atomic/fence替代关闭优化 |
| LAP-3 | 全局可变状态 | communicator_impl.cc:78 hostArgsTemp | 全局DpuKernelLaunchParam |
| LAP-4 | busy-wait无退避 | communicator_impl.cc:1793+ | KFC轮询无sleep/退避 |
| LAP-5 | std::max组合HcclResult | flush_handle.cc | 错误码数值大小无语义 |
| LAP-6 | LoadOp不受serialMutex保护 | hccl_communicator.cc | 最热路径无互斥(可能有意) |
| LAP-7 | 析构CHK_RET跳过清理 | communicator_impl.cc:2306+ | (void)忽略但内部可能提前return |
| LAP-8 | NPU/DPU ctx切换无RAII | communicator_impl.cc:3104+ | 异常路径可能漏切回NPU ctx |

### Legacy Notable Patterns

1. 序列化-反序列化桥接:
   Host打包7类资源 → HDC传输 → AICPU ParsePackedData恢复

2. CollAlgComponent Builder模式:
   ```cpp
   collAlgComponent = CollAlgComponentBuilder()
       .SetRankGraph(...).SetDevType(...).SetMyRank(...)
       .SetRankSize(...).SetDmaMode(PUT).Build();
   ```

3. DataTypeBitmap验证矩阵:
   `std::bitset<HCCL_DATA_TYPE_RESERVED>` x 7个SupportMap实例
   统一bitset查询替代if-else

4. DPU上下文切换显式管理:
   NPU ctx保存 → DPU切换 → DPU操作 → NPU切回
   缺少RAII上下文守卫

5. 缓存碰撞永久黑名单:
   saveCCUParams中碰撞的key永久加入ccuParamsNotCacheKey，不再尝试缓存

---

## 2.2.2 Legacy op_base_v2 + hcom_v2 深度分析

### 模块定位

Legacy entrance层包含两个核心子模块:
- `op_base/op_base_v2.{h,cc}` — NCCL风格的C API入口(HcclXxxV2系列)，面向单算子模式
- `hcom/hcom_v2.{h,cc}` — HCOM风格的C API入口(HcomXxxV2系列)，面向图模式/框架集成
- `hcom_comm/comm_manager.{h,cc}` — 两者共享的全局状态管理(HcclCommInfoV2/CommManager)

这三者构成了910_95(V2)芯片的legacy入口层。与framework层同名模块(src/framework/op_base, src/framework/hcom)是不同芯片的入口实现——framework层面向传统芯片(910/910B/910_93)，legacy/entrance层面向新芯片(910_95)。两者通过SoC类型分派共存，不是替代关系。

### 关键文件

| 文件 | 行数 | 职责 |
|------|------|------|
| legacy/entrance/op_base/op_base_v2.h | 194 | 85+个C函数声明，extern "C"包裹 |
| legacy/entrance/op_base/op_base_v2.cc | 2902 | 实现: 通信域生命周期 + 集合操作 + MC2 + Snapshot + RankGraph查询 |
| legacy/entrance/hcom/hcom_v2.h | 131 | 53个C函数声明，Hcom风格API |
| legacy/entrance/hcom/hcom_v2.cc | 1262 | 实现: Hcom集合操作 + Graph模式操作 + 算法选择 |
| legacy/entrance/hcom_comm/comm_manager.h | 117 | HcclCommInfoV2/HcclGroupParamsV2/CcuStatus/CommManager定义 |
| framework/inc/hccl_comm_pub.h | 464 | hcclComm类: framework层的Communicator外壳 |

### 1. op_base_v2.cc 功能分解 (2902行)

按功能域划分:

#### 1.1 通信域生命周期 (~750行, 行43-611 + 1510-1705)
- `HcclCommInitClusterInfoV2`: JSON ranktable → 创建Communicator → 注册到hcclGroupMap
- `HcclCommInitClusterInfoConfigV2`: 同上 + HcclCommConfig配置
- `HcclCommInitClusterInfoMemConfigV2`: 内存中传入ranktable字符串
- `HcclCommInitRootInfoV2/ConfigV2`: RootInfo模式，先做拓扑探测(RankInfoDetect)再建comm
- `HcclCommInitAllV2`: 多设备并行初始化(thread+future+async)
- `HcclCreateSubCommConfigV2`: 子通信域创建(从world group的rank子集)
- `HcclCommDestroyV2`: 销毁(从hcclGroupMap删除 + CCU状态清理)
- `CommInitRootInfo`: RootInfo的核心实现(探测→建comm→建链→更新ranktable)

关键模式:
```
CallSingletons() → 创建HcclCommunicator → Init() → RegisterAcceStateCallBack()
→ SetCommAcceleratorV2() → 注册到hcclGroupMap → RegisterPrintChannelInfoCallback()
```
这个6步初始化序列在3个初始化路径中重复出现(ClusterInfo/ClusterInfoConfig/RootInfoConfig)。

#### 1.2 集合通信操作 (~1200行, 行613-2246)
涵盖所有标准集合操作的NCCL风格接口:
- HcclAlltoAllV2 / HcclAlltoAllVV2 / HcclAlltoAllVCV2
- HcclReduceV2 / HcclAllReduceV2
- HcclBroadcastV2 / HcclScatterV2
- HcclAllGatherV2 / HcclAllGatherVV2
- HcclReduceScatterV2 / HcclReduceScatterVV2
- HcclSendV2 / HcclRecvV2 / HcclBatchSendRecvV2
- HcclBarrierV2 (用AllReduce 8个FP32元素实现)

#### 1.3 MC2资源管理 (~200行, 行1246-1498)
- HcclAllocComResourceByTilingV2: 按MC2 tiling分配通信资源
- HcclGetOpArgsV2/Free/SetXxx系列: MC2算子参数管理(malloc/free手动管理)
- HcclCommResPrepareV2: MC2模式资源预分配

#### 1.4 Snapshot快照 (~400行, 行2349-2630)
- HcclSnapshotSave: 序列化所有通信域到BinaryStream
- HcclSnapshotRecoverAllComms: 从快照恢复所有通信域(全局+子comm)
- SnapshotGenerate: 生成静态+动态快照数据 + CRC校验
- WaitAllCommReady: 恢复后推动式建链(轮询+10s超时)

#### 1.5 RankGraph查询 (~200行, 行2660-2898)
RankGraph拓扑查询API: GetNetLayers/GetInstSize/GetTopoType/GetRanks/GetLinks/GetEndpointInfo等。
这些是纯透传函数，直接委托给HcclCommunicator。

#### 1.6 配置和辅助 (~150行)
- HcclSetConfigV2/GetConfigV2: 确定性计算配置(910_95默认开启)
- HcclGetHeterogModeV2: 910_95只支持同构模式
- HcclCommSuspendV2: 910_95不支持
- HcclCommWorkingDevNicSetV2/SetMemoryRange/ActivateCommMemory等: 均返回NOT_SUPPORT
- 多个stub函数: 返回HCCL_E_NOT_SUPPORT或直接忽略

### 2. hcom_v2.cc 功能分解 (1262行)

#### 2.1 集合操作Hcom入口 (~600行)
标准Hcom API，额外参数group(字符串组名) + tag:
HcomAllGatherV2/VV2, HcomAllReduceV2, HcomReduceScatterV2/VV2,
HcomSendV2/ReceiveV2, HcomAlltoAllV2/VV2/VCV2, HcomBroadcastV2, HcomReduceV2

#### 2.2 Graph模式操作 (~200行)
HcclCommGraphAllGatherV2, HcomGraphAllReduceV2, HcomGraphReduceScatterV2等。
这些接收int64_t comm句柄(reinterpret_cast为HcclCommunicator*)，用于PyTorch图模式。

#### 2.3 资源查询和配置 (~300行)
HcomGetWorkspaceSubStreamNumV2, HcomGetWorkspaceMemSizeV2, HcomSetWorkspaceResourceV2,
HcomGetAlltoAllStagedWorkSpaceMemSizeV2, HcomCalcTaskNumV2, HcomCalcNumBlocksV2等。

#### 2.4 算法选择 (~100行)
HcomSelectAlgV2, HcomGraphSelectAlgV2, HcclGetAlgExecParamV2

#### 2.5 辅助和stub (~100行)
多个返回NOT_SUPPORT或WARNING的函数: HcomResetQosCfgV2, HcclCommSetQosCfgV2,
HcomGetCommCCLBufferSizeV2, HcomMc2AiCpuStreamAllocAndGetV2等。

### 3. op_base_v2 vs hcom_v2 的核心差异

#### 3.1 接口风格差异
| 维度 | op_base_v2 (NCCL风格) | hcom_v2 (HCOM风格) |
|------|------|------|
| 通信域标识 | HcclComm comm (void*句柄) | const char *group (字符串组名) |
| 操作标识 | 自动生成 "OpType_" + commId | 调用方传入tag字符串 |
| 执行方式 | communicator->LoadOpbasedCollOp() | hcclComm->LoadOffloadCollOp() |
| 日志详细度 | 条件日志(GetEntryLogEnable) + Trace.Save | 简单 HCCL_RUN_INFO |
| stream capture | 检查GetStreamCaptureInfo + modelId | 不处理 |
| opParams存储 | static thread_local (14处) | 栈上分配 |

#### 3.2 调用路径差异

op_base_v2路径:
```
HcclAllReduceV2(comm) → static_cast<HcclCommunicator*>(comm)
  → communicator->LoadOpbasedCollOp(opParams, stream)
```

hcom_v2路径:
```
HcomAllReduceV2(group) → GetHcclCommV2(group) → shared_ptr<HcclCommunicator>
  → hcclComm->LoadOffloadCollOp(tag, opParams, stream)
```

两者最终都调用HcclCommunicator，但方法不同:
- op_base用 `LoadOpbasedCollOp` (opParams已含opTag)
- hcom用 `LoadOffloadCollOp` (tag单独传入)

这表明backend的V2 Communicator有两套执行入口:
- opbased: 单算子即时执行模式
- offload: 卸载/延迟执行模式(更适合图模式)

### 4. 全局状态管理

#### 4.1 HcclCommInfoV2 结构 (comm_manager.h:57-72)
```cpp
struct HcclCommInfoCtxV2 {
    s32 devId{-1};
    shared_ptr<HcclCommunicator> pComm{nullptr};      // 全局(world)通信域
    CommParams commParams;                              // world comm的参数副本
    map<string, HcclGroupParamsV2> hcclGroupMap;        // 组名→通信域映射
    mutex groupParamsLock;                              // hcclGroupMap的锁
    bool isUsed{false};
    DeviceStatus status{DEVICE_IDLE};                  // IDLE/RECOVERED/READY
    u64 step{0};                                       // snapshot step计数
    CcuStatus ccuStatus;                               // CCU MS/Sched使用追踪
};
```

#### 4.2 CommManager单例 (per-device)
```cpp
CommManager::GetInstance(deviceLogicId)   // 按逻辑设备ID获取
  → commInfoV2  (HcclCommInfoV2实例)
  → ccuDriverHandle
```

GetCommInfoV2()是全局便捷函数: 获取当前线程设备对应的CommManager的commInfoV2引用。

#### 4.3 g_hcclCommunicators (op_base_v2.cc:43)
```cpp
std::map<std::string, Hccl::HcclCommunicator *> g_hcclCommunicators[MAX_MODULE_DEVICE_NUM + 1];
```
这是一个per-device的全局数组，但在op_base_v2.cc中没有被使用(声明了但无引用)。
这是一个遗留的全局状态，已被HcclCommInfoV2.hcclGroupMap取代。

#### 4.4 锁策略
- hcclGroupMap操作: 统一使用 `groupParamsLock` mutex保护
- hcom_v2的GetHcclGroupParams使用 lock_guard
- op_base_v2的初始化/销毁使用 unique_lock (允许提前unlock)
- thread_local g_hcclDeviceId: 无锁(per-thread天然安全)
- g_taskServiceMap: 无锁保护，仅在op_base_v2中find/访问

### 5. V2 vs V1 差异

#### 5.1 Architecture Shift
V1 (framework/op_base): 通过hcclComm类(hccl_comm_pub.h中定义)间接访问HcclCommunicator
```
HcclOpInfoCtx → HcclCommPtr(shared_ptr<hcclComm>) → hcclComm::communicator_ → HcclCommunicator
```

V2 (legacy/entrance/op_base): 直接操作HcclCommunicator，跳过hcclComm包装层
```
HcclCommInfoV2 → shared_ptr<HcclCommunicator> → HcclCommunicator
```

V2消除了一层间接(hcclComm类)，直接将HcclCommunicator*作为HcclComm句柄返回给调用方。

#### 5.2 Communicator获取方式
V1: HcclOpInfoCtx.pComm → 通过hcclComm的public方法间接操作
V2: `static_cast<HcclCommunicator*>(comm)` 直接cast

#### 5.3 初始化模式
V1: RankTable JSON解析 → hcclComm::init(params, rankTable) → HcclCommunicator::Init
V2: RankInfoDetect拓扑探测 → HcclCommunicator(commParams) → Init(rankTable)
    新增: RootInfoUpdate(二次更新ranktable，建链端口交换)

#### 5.4 设备能力差异
V2文件中多处硬编码 `DEV_TYPE_910_95`，以及多个函数返回 `HCCL_E_NOT_SUPPORT`:
- HcclCommSuspendV2 (不支持)
- HcclGetRemoteIpcHcclBufV2 (不支持)
- HcclCommWorkingDevNicSetV2 (不支持)
- HcclCommSetMemoryRangeV2/Unset/Activate/Deactivate (全部不支持)
- HcclGetHeterogModeV2: 只支持同构模式

这些NOT_SUPPORT函数暴露了910_95相对传统芯片的功能差异。

### 6. 惯用法

#### 6.1 static thread_local CollOpParams (op_base_v2.cc特有)
op_base_v2.cc中14处使用:
```cpp
static thread_local Hccl::CollOpParams opParams;
opParams.opType = ...;
opParams.sendBuf = ...;
// 每次调用覆盖字段
communicator->LoadOpbasedCollOp(opParams, stream);
```
目的: 避免CollOpParams的重复构造/析构开销。在hcom_v2.cc中不使用此模式(使用栈分配或工厂函数GetHcclOpParams)。

风险: 字段不完全覆盖可能导致残留状态。例如opParams.opTag有时设置有时不设置。

#### 6.2 条件日志模式 (op_base_v2.cc的Hccl接口)
```cpp
char stackLogBufferV2[LOG_TMPBUF_SIZE];
if (EnvConfig::GetInstance().GetLogConfig().GetEntryLogEnable()) {
    s32 ret = snprintf_s(stackLogBufferV2, ...);
    CHK_PRT_CONT(ret == -1, ...);
    std::string logInfo = "Entry-Xxx:" + std::string(stackLogBufferV2);
    if (isCapture) { ... }
    communicator->GetTrace().Save(logInfo);
}
// ... 执行操作 ...
if (EnvConfig::GetInstance().GetLogConfig().GetEntryLogEnable()) {
    std::string endInfo = "Xxx:success,take time: " + ...;
    communicator->GetTrace().Save(endInfo);
}
```
这个模式在所有Hccl风格接口中重复(AllReduce/Broadcast/Scatter等)。hcom_v2的接口更简洁，仅用HCCL_RUN_INFO。

#### 6.3 GetHcclOpParams工厂函数 (hcom_v2.cc特有)
```cpp
inline CollOpParams GetHcclOpParams(void *inputPtr, void *outputPtr, u64 count,
    HcclDataType dataType, OpType opType, HcclReduceOp op = HCCL_REDUCE_RESERVED, ...)
```
参数组装统一到一个函数，减少重复。但op_base_v2.cc没有使用，而是每个函数手动组装。

#### 6.4 CallSingletons()临时规避 (6处)
```cpp
CHK_RET(CallSingletons()); // 临时规避，在初始化通信域前声明单例保证时序
```
在所有初始化路径中调用，用于确保某些单例在通信域初始化前已构造。
注释明确标记"临时规避"，暗示存在初始化顺序依赖问题。

#### 6.5 通信域初始化代码重复
CreateCommConfig(行125-177)和CreateCommConfigRootInfo(行179-230)几乎完全相同，
只有Init调用的参数不同(string vs RankTableInfo)。应提取公共流程。

同样，HcclCommInitClusterInfoV2(行248-313)和HcclCommInitClusterInfoConfigV2(行342-397)
大量重复JSON解析和Communicator创建代码。

### 7. 反模式

#### AP-2.2.2-1: 未使用的全局数组 g_hcclCommunicators
文件: op_base_v2.cc:43
```cpp
std::map<std::string, Hccl::HcclCommunicator *> g_hcclCommunicators[MAX_MODULE_DEVICE_NUM + 1];
```
声明了但在整个文件中未使用。这是从V1代码遗留的全局状态，功能已被HcclCommInfoV2.hcclGroupMap取代。
应删除此声明以避免混淆。

#### AP-2.2.2-2: HcclGetOpArgsV2的null检查错误
文件: op_base_v2.cc:1296-1310
```cpp
HcclOpArgs *opArgsMem = (HcclOpArgs *)malloc(sizeof(HcclOpArgs));
if (opArgs == nullptr) {   // BUG: 应检查 opArgsMem == nullptr
    HCCL_ERROR("[HcclGetOpArgs] malloc HcclOpArgs mem fail, please check.");
    return HCCL_E_INTERNAL;
}
```
检查的是入参opArgs而非malloc返回的opArgsMem，malloc失败时将产生空指针解引用。

#### AP-2.2.2-3: HcclFreeOpArgsV2的局部置null无效
文件: op_base_v2.cc:1312-1320
```cpp
free(opArgs);
opArgs = nullptr;  // 无效: 形参赋值不影响调用方
```
经典C错误——给参数(按值传递的指针)赋nullptr不影响调用方变量。

#### AP-2.2.2-4: malloc/free与C++混用
文件: op_base_v2.cc:1301
HcclOpArgs使用malloc分配，但HcclOpArgs有Init()方法暗示可能需要构造函数。
应使用new/delete或unique_ptr，与项目其他地方的new(nothrow)惯用法一致。

#### AP-2.2.2-5: static thread_local opParams残留状态风险
文件: op_base_v2.cc (14处)
static thread_local对象跨调用保持状态，某些字段可能不被每次覆盖。
例如: HcclScatterV2(行1753)没有设置opParams.opTag，但后续函数可能依赖此字段。
HcclSendV2(行1955)设置了opParams.opTag = tag，但HcclAllGatherV2(行1812)也设置了。
如果同一线程先调用AllGather再调用Scatter，opTag会残留AllGather的值。

#### AP-2.2.2-6: 日志信息copy-paste错误
文件: op_base_v2.cc 多处
HcclSendV2(行1969)、HcclRecvV2(行2028)、HcclReduceScatterV2(行2091)、
HcclReduceScatterVV2(行2183)、HcclBatchSendRecvV2(行2240)的结束日志全部写:
```cpp
std::string endInfo = "HcclAllGatherVV2:success,take time: " + ...
```
错误使用了HcclAllGatherVV2的名字，应分别使用各自函数名。
这是典型的copy-paste错误，代码复制后未修改标识符。

#### AP-2.2.2-7: IS_SET_DEVICE_MASK重复定义
文件: hcom_v2.cc:1224 和 1240
```cpp
#define IS_SET_DEVICE_MASK 0xfffffffe  // 在两个函数内部各定义了一次
```
宏定义在函数体内，且在两个函数中重复。应提取到文件头部或匿名namespace。

#### AP-2.2.2-8: HcomGetAlltoAllStagedWorkSpaceMemSizeV2注释中的疑问
文件: hcom_v2.cc:553
```cpp
u64 dataSize = 0; // ??
```
注释"??"表明开发者不确定此值是否正确。代码中传入0可能导致资源计算不准确。

#### AP-2.2.2-9: 初始化代码大量重复
CreateCommConfig / CreateCommConfigRootInfo / HcclCommInitClusterInfoV2 三者的初始化序列:
new HcclCommunicator → Init → RegisterAcceStateCallBack → SetCommAcceleratorV2
→ 保存到hcclGroupMap → RegisterPrintChannelInfoCallback
在三个函数中几乎逐行重复(每处约30行)。应提取为CreateAndRegisterComm()公共函数。

#### AP-2.2.2-10: reinterpret_cast无安全检查(hcom_v2 Graph接口)
文件: hcom_v2.cc 行781, 789, 798, 807, 815, 828, 852, 879, 906, 934, 958, 982, 1106
Graph模式接口接收int64_t/s64参数，直接reinterpret_cast为HcclCommunicator*:
```cpp
Hccl::HcclCommunicator* hcclComm = reinterpret_cast<Hccl::HcclCommunicator *>(opBaseHcom);
```
无类型安全检查，无magic number验证。任何非法值都会导致未定义行为。
与op_base_v2的static_cast<HcclCommunicator*>(comm)相比更危险。

### 8. op_base_v2与hcom_v2的关系: 功能对比

两者提供同一套集合操作的两种API风格:

| 集合操作 | op_base_v2 (NCCL风格) | hcom_v2 (HCOM风格) |
|----------|----------------------|-------------------|
| AllGather | HcclAllGatherV2 | HcomAllGatherV2 |
| AllReduce | HcclAllReduceV2 | HcomAllReduceV2 |
| Broadcast | HcclBroadcastV2 | HcomBroadcastV2 |
| Reduce | HcclReduceV2 | HcomReduceV2 |
| ReduceScatter | HcclReduceScatterV2 | HcomReduceScatterV2 |
| Send/Recv | HcclSendV2/RecvV2 | HcomSendV2/ReceiveV2 |
| AlltoAll | HcclAlltoAllV2 | HcomAlltoAllV2 |
| AlltoAllV | HcclAlltoAllVV2 | HcomAlltoAllVV2 |
| AlltoAllVC | HcclAlltoAllVCV2 | HcomAlltoAllVCV2 |
| Barrier | HcclBarrierV2 | (无对应) |
| BatchSendRecv | HcclBatchSendRecvV2 | (无对应) |

hcom_v2额外提供:
- Graph模式API(HcomGraphXxxV2系列，13个函数)
- 资源查询(WorkspaceSubStreamNum/MemSize/TaskNum)
- 算法选择(SelectAlg/CalcNumBlocks/GetAlgExecParam)
- CCL Buffer管理(CreateCommCclBuf/GetIn/Out/Indirect)

op_base_v2额外提供:
- 通信域完整生命周期(Init/Destroy/SubComm/Resume)
- MC2资源管理(AllocByTiling/OpArgs系列)
- Snapshot快照(Save/Recover/GetBufSize)
- RankGraph查询(15+个接口)
- 配置管理(SetConfig/GetConfig)

### 9. 与hccl_comm_pub.h(hcclComm类)的关系

hcclComm类(framework/inc/hccl_comm_pub.h)是传统芯片(V1)的包装层:
- 内部持有 `unique_ptr<HcclCommunicator> communicator_`
- 提供AllReduce/Broadcast等高级方法，内部委托给communicator_
- 还持有IndependentOp、CollComm等V1特有子系统
- 管理CCL buffer、barrier memory等设备资源

V2入口层完全绕过hcclComm类，直接操作HcclCommunicator:
- V2: `HcclComm comm` → `static_cast<HcclCommunicator*>(comm)` → communicator方法
- V1: `HcclComm comm` → `hcclComm*` → `communicator_->xxx()`

这意味着V2的HcclComm句柄和V1的HcclComm句柄虽然类型相同(void*)，实际指向不同类型的对象。
混用会导致严重问题(类型混淆)。两者通过SoC类型(910_95 vs其他)在编译或运行时分隔。

### 10. Snapshot机制(op_base_v2独有)

Snapshot是910_95的CRIU式checkpoint/restore机制:

保存流程:
```
HcclSnapshotGetBufSize(step) → SnapshotGenerate() →
  GetAllSnapShotStaticBuf(全局comm + 所有子comm的静态状态) +
  GetAllSnapShotDynamicBuf(全局comm + 所有子comm的动态状态) +
  GetSnapShotCcuStatusBuf(CCU MS/Sched使用记录)
→ HcclSnapshotSave(snapshotBuf, size, step) →
  BinaryStream序列化 + CRC32校验 + memcpy到用户buffer
```

恢复流程:
```
HcclSnapshotRecoverAllComms(clusterInfo, changedInfo, snapshotBuf) →
  ParseSnapshotToLocalBuff(反序列化) →
  new HcclCommunicator(savedParams) → RecoverComm(静态数据, step, changedInfo) →
  循环恢复子通信域: RecoverSubComm() →
  RecoverSnapshotCcuStatus(CCU状态恢复) →
  WaitAllCommReady(推动式建链, 10s超时)
```

值得注意: WaitAllCommReady内部是tight loop(无sleep/yield)，持有groupParamsLock的同时轮询所有comm的IsCommReady()，10秒超时。这种持锁忙等待在超时前会完全阻塞其他hcclGroupMap操作。

### 11. 总结

#### 架构角色
Legacy entrance层是910_95芯片的API入口薄层，职责是:
1. C ABI暴露(extern "C")
2. 参数校验和类型转换
3. 全局状态(hcclGroupMap)管理
4. 日志和计时
5. 委托给HcclCommunicator执行

不包含任何通信算法或协议逻辑。是典型的Facade模式。

#### 质量观察
- 代码重复严重: 初始化序列3处重复，日志模板12+处复制
- Copy-paste错误: 至少5处日志使用了错误的函数名
- Null检查bug: HcclGetOpArgsV2检查了错误的变量
- 遗留代码: g_hcclCommunicators全局数组未使用
- 风格不一致: op_base用static thread_local + 条件日志，hcom用栈分配 + 简单日志
- NOT_SUPPORT函数多: 至少8个函数只返回不支持，占接口总量的~10%

#### 设计模式
- Facade: 两个文件都是Communicator的薄包装
- Factory Method: 多种Init路径(ClusterInfo/RootInfo/RootInfoConfig)
- Adapter: hcom_v2将group字符串适配为Communicator查找
- 全局注册表: hcclGroupMap是运行时通信域的中央注册表


## 2.2.2 op_base/ — HCCL API Entry Layer Deep Analysis

### Module Overview

op_base/ 是hcomm的最外层API入口模块，直接实现了HCCL公共API (HcclAllReduce, HcclBroadcast, HcclSend等)。
它是C/C++对外接口与内部hcclComm对象之间的桥梁层，承担以下职责:
1. 通信域生命周期管理 (Init/Destroy/Suspend/Resume)
2. 集合通信算子入口 (AllReduce/AllGather/ReduceScatter/Broadcast/Scatter/Reduce等)
3. 点对点通信入口 (Send/Recv/BatchSendRecv)
4. AlltoAll系列入口 (AlltoAll/AlltoAllV/AlltoAllVC)
5. MC2资源分配 (HcclAllocComResourceByTiling)
6. 辅助API (GetRootInfo/GetConfig/SetConfig/GetTopoDesc/CommRegister等)

在架构中的位置: op_base是hcomm面向AI训练框架(PyTorch/MindSpore)的对外窗口，
向下通过hcclComm对象委托到communicator模块。

### Key Files (7个文件)

| File | Lines | Role |
|------|-------|------|
| op_base.h | 112 | 头文件: HcclOpInfoCtx结构体定义 + 函数声明 |
| op_base.cc | 4695 | 核心实现: 所有API入口函数 |
| op_base_v2.h | 190 | V2版本函数声明(全部weak符号) |
| op_base_host.cc | 261 | Host编译目标: 完整的host侧实现(AllReduceInner/Barrier) |
| op_base_device.cc | 55 | Device编译目标: stub实现(仅WARNING返回) |
| op_base_mc2.cc | 261 | MC2相关功能: tiling解析/资源分配/OpArgs系列 |
| op_base_pub.h | 33 | 公共辅助结构: OpBaseMemPara/GatherPara |

### Core Data Structures

#### HcclOpInfoCtx (op_base.h:26-44)
```cpp
using HcclOpInfoCtx = struct HcclInfoTag {
    HcclCommPtr pComm;                    // 主通信域指针(shared_ptr)
    hccl::HcclCommParams params;          // 通信参数
    hccl::RankTable_t rankTable;          // rank表
    bool cloudFlag = false;               // 实验室(0) vs 云场景(1)
    bool isUsed;                          // 槽位是否已使用
    std::mutex opGroupMapMutex;           // 保护opGroup2CommMap
    std::unordered_map<std::string, std::shared_ptr<hccl::hcclComm>> opGroup2CommMap; // group名→comm映射
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectServer; // 拓扑检测server
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectAgent;  // 拓扑检测agent
};
```
每个设备对应一个HcclOpInfoCtx槽位，通过全局数组g_opHcomInfos按deviceId索引。

### Global State Design

#### 全局变量清单 (op_base.cc)
```
thread_local s32 g_hcclDeviceId = INVALID_INT;       // L83: 当前线程的设备ID
std::mutex g_opHcomInfosMutex{};                      // L84: 保护g_opHcomInfos
std::mutex g_opHcomOneSideMutex{};                    // L85: 保护单边通信
HcclOpInfoCtx g_opHcomInfos[MAX_MODULE_DEVICE_NUM+1]; // L86: 设备→OpInfoCtx数组
std::unordered_map<s32, std::unordered_map<std::string, shared_ptr<HcclOpInfoCtx>>>
    g_oneSidedCommHcomInfos;                           // L268: 单边通信域管理
std::set<HcclComm> g_oneSidedCommSet;                 // L269: 单边comm集合
std::unordered_map<CommSymWindow, HcclComm> winHandle2comm; // L4628: 对称窗口→comm映射
std::mutex g_winHandleMtx;                            // L4629: 保护winHandle2comm
```

#### thread_local g_hcclDeviceId 使用模式 (27次引用)
- HcclGetDeviceId(): 懒初始化模式，INVALID_INT时调用hrtGetDevice获取
- GetHcclExistDeviceOpInfoCtx(): 用g_hcclDeviceId索引g_opHcomInfos数组
- GetHcclOpInfoCtx(): 两层fallback: 先尝试获取deviceId → 扫描已使用槽位 → 使用末尾兜底槽位
- HcclDeviceRefresh(): 刷新thread_local值 (27次引用，每个API入口必调)

g_hcclDeviceId是整个op_base模块最核心的线程局部状态。它解决了一个根本问题:
同一进程中多张NPU卡各自有独立的通信域，需要区分当前线程操作的是哪张卡。

#### g_opHcomInfos 全局数组 (13次引用)
固定大小 MAX_MODULE_DEVICE_NUM+1 的数组(最后一个是兜底槽位)。
通过isUsed标志管理槽位占用，mutex保护并发访问。
这是一个简单但不够灵活的设计: 设备数固定上限，兜底槽位可能被多个无法获取deviceId的线程共享。

### V1/V2 Version Dispatch Mechanism

#### 三层分派架构

1. HCCLV2_FUNC_RUN宏 (param_check_basic_v2.h:14-21):
   ```cpp
   #define HCCLV2_FUNC_RUN(func, ...) \
       do { \
           const char *socNamePtr = aclrtGetSocName(); \
           CHK_PTR_NULL(socNamePtr); \
           if (IsSupportHCCLV2(socNamePtr)) { \
               return func; \
           } \
       } while (0)
   ```
   运行时按SoC名称判断: Ascend950开头的芯片走V2路径，否则继续执行V1路径。

2. op_base_v2.h: 75个V2函数声明，全部标记`__attribute__((weak))`。
   weak符号意味着: 如果V2库(next/)没有链接，这些函数解析为nullptr，
   IsSupportHCCLV2返回true时尝试调用会崩溃。
   实际运行时通过动态库加载保证V2实现存在。

3. 条件编译守护: `#if (!defined (HCCD)) && (!defined (CCL_KERNEL_AICPU))`
   V2分派代码被条件编译包围，Device侧(HCCD)和AICPU(CCL_KERNEL_AICPU)编译时完全跳过。

#### HCCLV2_FUNC_RUN在op_base.cc中的使用频率: 40次
每个API入口函数在执行V1逻辑前先尝试V2分派。关键特征:
- return语义: HCCLV2_FUNC_RUN内部直接return，V2成功则V1代码不执行
- 不可回退: 一旦走V2，即使V2失败也不会fallback到V1
- 位置不一致: 部分在参数校验前(如HcclBatchSendRecvInner)，部分在校验后(如HcclAllReduceInner)

### Host/Device Compilation Split

op_base_host.cc 和 op_base_device.cc 是同一接口的两个编译目标:

| 函数 | Host实现 | Device实现 |
|------|---------|-----------|
| GetCaptureInfo | 完整stream capture检测 | 返回WARNING+SUCCESS |
| HcclAllReduceInner | 完整算子入口逻辑(180行) | 返回WARNING+SUCCESS |

Device侧是stub: 所有函数打印WARNING后返回SUCCESS。
这意味着Device侧(AICPU算子)不通过op_base入口，而是通过其他路径(aicpu_operator)触发通信。

op_base_host.cc中的HcclAllReduceInner展示了完整的算子入口模式:
group操作支持 → capture检测 → profiling → 参数校验 → V2分派 →
operatorlock_ → StateGuard → tag构造 → 日志 → SetWorkflowMode →
PrintMemoryAttr → 实际操作 → profiling上报 → 结束日志

### API Entry Function Pattern (Collective Ops)

所有集合通信API入口(AllReduce/Broadcast/ReduceScatter/AllGather/Reduce/Scatter/AlltoAll等)
遵循高度一致的模式，以HcclAllReduceInner为标准参考(op_base_host.cc:72-179):

#### Step 1: Group Check (hcclGroupDepth > 0) — 18次出现
```cpp
if (hcclGroupDepth > 0) {
    struct hcclOpInfo info;
    info.coll = HcclCMDType::HCCL_CMD_ALLREDUCE;
    info.sendbuff = static_cast<const void *>(sendBuf);
    // ... fill info ...
    CHK_RET(taskAppend(comm, info));
    return HCCL_SUCCESS;
}
```
如果在HcclGroupStart/End块内，将操作延迟到taskAppend，不立即执行。
这是类NCCL的group语义支持。

#### Step 2: Timing + Capture Detection
```cpp
HcclUs startut = TIME_NOW();
bool isCapture;
aclmdlRICaptureStatus captureStatus = ...;
uint64_t modelId = 0xFFFFFFFF;
CHK_PRT(GetCaptureInfo(stream, captureStatus, modelId, isCapture));
if (!isCapture) { HcclSetIfProfile(); }
uint64_t beginTime = hrtMsprofSysCycleTime();
```
Stream capture检测: 如果在图模式capture中，跳过profiling。

#### Step 3: Early Return for Zero Count
```cpp
CHK_PRT_RET(count == 0, HCCL_WARNING("input count is 0, return AllReduce success"), HCCL_SUCCESS);
```

#### Step 4: Parameter Validation (RPT_INPUT_ERR + CHK_PTR_NULL pair)
```cpp
RPT_INPUT_ERR(comm == nullptr, "EI0003",
    std::vector<std::string>({"ccl_op", "parameter", "value", "tips"}),
    std::vector<std::string>({"HcclAllReduceInner", "comm", "nullptr", "please check comm"}));
CHK_PTR_NULL(comm);
```
每个参数先RPT_INPUT_ERR上报错误信息，再CHK_PTR_NULL实际检查。
RPT_INPUT_ERR是面向用户的错误上报(IDE集成)，CHK_PTR_NULL是实际的null检查+return。
这个模式在op_base.cc中出现68次(RPT_INPUT_ERR)。

#### Step 5: V2 Dispatch
```cpp
HCCLV2_FUNC_RUN(HcclAllReduceV2(sendBuf, recvBuf, count, dataType, op, comm, stream));
```

#### Step 6: Lock + State Guard
```cpp
hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
const std::lock_guard<std::mutex> lock(hcclComm->operatorlock_);
StateGuard<hccl::hcclComm, HcclCommState> guard(hcclComm, HcclCommState::INUSE);
```
operatorlock_: 保证同一comm不被并发调用(8次出现)。
StateGuard: RAII设置comm状态为INUSE，函数返回时自动恢复(13次出现)。
这确保HcclCommDestroy能检测到comm正在使用中(返回HCCL_E_AGAIN)。

#### Step 7: Tag Construction
```cpp
const string tag = "AllReduce_" + hcclComm->GetIdentifier();
```
Tag = "算子名_" + comm标识符。同一通信域同一算子共享tag，用于资源复用。
Send/Recv特殊: tag包含src和dst rank: "worldCommSendRecv_src_dst_identifier"

#### Step 8: Specific Validation (per-op)
```cpp
CHK_RET_AND_PRINT_IDE(HcomCheckOpParam(...), tag.c_str());
CHK_RET_AND_PRINT_IDE(HcomCheckReductionOp(op), tag.c_str());
CHK_RET_AND_PRINT_IDE(HcomCheckReduceDataType(dataType, op, devType), tag.c_str());
```
使用CHK_RET_AND_PRINT_IDE: 失败时不仅return，还打印IDE友好的错误信息。

#### Step 9: Entry Log (conditional)
```cpp
if (GetExternalInputHcclEnableEntryLog()) {
    // snprintf_s to stackLogBuffer
    // hcclComm->SaveTraceInfo(logInfo)
}
```
32次出现GetExternalInputHcclEnableEntryLog检查。
使用栈上buffer(char stackLogBuffer[LOG_TMPBUF_SIZE])避免堆分配。
snprintf_s是安全格式化(华为securec库)。

#### Step 10: SetWorkflowMode + PrintMemoryAttr
```cpp
CHK_RET_AND_PRINT_IDE(SetWorkflowMode(HcclWorkflowMode::HCCL_WORKFLOW_MODE_OP_BASE), tag.c_str());
CHK_RET_AND_PRINT_IDE(PrintMemoryAttr(sendBuf), tag.c_str());
```

#### Step 11: Actual Operation (delegate to hcclComm)
```cpp
CHK_RET_AND_PRINT_IDE(hcclComm->AllReduceOutPlace(tag, sendBuf, recvBuf, count, dataType, op, stream), tag.c_str());
```
所有实际工作委托给hcclComm的*OutPlace方法。

#### Step 12: Profiling Report
```cpp
CHK_RET(CallMsprofReportHostApi(hcclComm, HcclCMDType::HCCL_CMD_ALLREDUCE, beginTime, count, dataType, tag));
if (!isCapture) { HcclResetIfProfile(); }
ProfilingManagerPub::DeleteThreadCaptureStatus(threadID);
```

#### Step 13: End Log (conditional)
```cpp
if (GetExternalInputHcclEnableEntryLog()) {
    HcclUs endut = TIME_NOW();
    std::string endInfo = "HcclAllReduceInner:success,take time: " + ...;
    CHK_RET_AND_PRINT_IDE(hcclComm->SaveTraceInfo(endInfo), tag.c_str());
}
```

### Comm Init Pattern: do-while(0) + errorFlag

通信域初始化函数(InitCommClusterInfo, InitCommRootInfo, HcclCreateSubCommConfigInner)
使用特征性的 do{...}while(0) + errorFlag 模式:
```cpp
bool errorFlag = false;
do {
    ret = step1();
    CHK_PRT_BREAK(ret != HCCL_SUCCESS, HCCL_ERROR(...), errorFlag = true);
    ret = step2();
    CHK_PRT_BREAK(ret != HCCL_SUCCESS, HCCL_ERROR(...), errorFlag = true);
    // ... 10+ steps
} while (0);
if (errorFlag) {
    (void)HcclCommDestroy(pComm.get());
    return ret;
}
```
这是模拟goto清理的C++惯用法: break跳出do块进入统一清理逻辑。
InitCommClusterInfo (L396-531) 有13步设置步骤全用此模式。

### Comm Init Config Pipeline

通信域初始化后的配置步骤是固定序列(至少出现3次，逻辑完全重复):
1. SetDeterministicConfig
2. SetQpQosAttr (TC/SL)
3. SetAivModeConfig
4. SetOnlyAivModeConfig
5. SetAicpuUnfoldConfig
6. SetExecTimeOutConfig
7. SetAlgoConfig
8. SetIndependentOpConfig

这些步骤在InitCommClusterInfo、InitCommRootInfo、HcclCreateSubCommConfigInner中
几乎逐字重复(每处约60行)，是明显的代码重复反模式。

### Group Async Pattern (hcclGroupDepth)

当hcclGroupDepth > 0时(在HcclGroupStart/End块内)，API调用采用延迟执行模式:
```cpp
if (hcclGroupDepth > 0) {
    // 对于通信域Init: 创建AsyncJob → commInitTaskAppend → return
    // 对于集合通信: 填充hcclOpInfo → taskAppend → return
    // 对于Destroy: 创建AsyncJob → commInitTaskAppend → return
}
```
AsyncJob通过std::shared_ptr管理，包含Wrapper函数指针+参数。
Wrapper函数先hrtSetDevice恢复设备上下文，再执行实际逻辑。

### MC2 Special Path (op_base_mc2.cc)

MC2 (Multi-Chip 2) 是AICPU融合算子的资源预分配机制。
关键函数:

HcclGetInitTilingList (L32-51): 解析MC2 tiling数据
- tiling是二进制结构: version(u32) + count(u32) + Mc2ServerCfg + N个Mc2HcommCfg
- version >= 2 按固定offset计算每个HcommCfg位置
- version < 2 用旧格式(offset数组)

HcclMc2ComResourceByTiling (L53-94): 按tiling信息批量预分配资源
- 遍历tilingList，匹配当前comm的groupName
- 为每个匹配的tiling创建OpParam并调AllocComResourceByTiling
- commEngine==0 表示启用AICPU引擎

HcclAllocComResourceByTiling (L99-164): MC2资源分配入口(extern "C")
- 兼容老版本: version < 2 或非910_93芯片走旧路径
- 新路径: 创建aicpuStream → 按tiling分配资源

OpArgs系列 (L167-255): 面向MC2的算子参数设置API
- 全部是简单的 CHK_PTR_NULL → HCCLV2_FUNC_RUN → return HCCL_SUCCESS
- 所有函数V1路径直接返回SUCCESS(不做任何事)
- 说明OpArgs仅V2路径有实际实现

HCCL_INDEPENDENT_OP环境变量 (6次查询):
在Init和某些查询API中出现: getenv("HCCL_INDEPENDENT_OP")。
当设置此环境变量时，V2 comm上层再包裹一层CollComm(HcclCommInitCollComm)。
这是"独立算子"模式的切换开关。

### One-Sided Communication Management

单边通信是独立的通信域管理子系统(非集合通信):
- g_oneSidedCommHcomInfos: 二层map (deviceId → commName → HcclOpInfoCtx)
- g_oneSidedCommSet: 快速判断comm是否是单边通信域
- 通过HcclCommInitClusterInfoMemConfig入口创建
- HcclOneSidedCommDestroy单独的销毁路径
- 不支持创建子通信域(SubCommIsOneSidedComm检查)

### Profiling Integration

Profiling深度集成在每个API入口中，包含多个层面:

1. HcclSetIfProfile/HcclResetIfProfile (L1886-1897):
   根据workflow mode和profiling全局状态控制profiling开关

2. ProfilingManagerPub::SetThreadCaptureStatus:
   线程级capture状态管理(用于stream capture场景)

3. CallMsprofReportHostApi (L56-81):
   每个算子结束后上报Host API信息到msprof

4. HCCL_PROFILER_ADD_GROUP_UDI / HCCL_PROFILER_DEL_GROUP_UDI:
   在comm Init/Destroy时管理group与UDI(User Device Info)的映射

5. hrtMsprofSysCycleTime():
   使用硬件cycle计时器获取高精度时间戳

### NSLB Data Plane Integration

NSLB (Network Service Level Balance Data Plane) 出现在多个算子入口的尾部:
```cpp
if (hcclNslbDp::GetInstance().GetGlobalCommTaskId() != 0) {
    // 查询算法类型 → 填充NSLB表6 → 发送
}
```
仅在Broadcast/Scatter/Reduce三个有root rank的算子中出现(SetNslbDpRootRank)。
这是运行时网络负载均衡数据采集的hook点。

### Idioms & Conventions

#### 1. RPT_INPUT_ERR + CHK_PTR_NULL 配对 (68处)
每个对外参数的空指针检查都是双重的:
- RPT_INPUT_ERR: 向IDE/错误中心上报参数错误(错误码"EI0003")
- CHK_PTR_NULL: 实际的null检查+return
两者在逻辑上冗余(都检查nullptr)，但目的不同(上报 vs 控制流)。

#### 2. CHK_RET_AND_PRINT_IDE (大量使用)
CHK_RET的增强版: 失败时额外打印tag信息供IDE展示。
在op_base.cc中是最高频的错误处理宏(与CHK_RET一起共607处)。

#### 3. stackLogBuffer + snprintf_s (16处)
用栈上固定大小buffer构造日志字符串，避免堆分配。
```cpp
char stackLogBuffer[LOG_TMPBUF_SIZE];
s32 ret = snprintf_s(stackLogBuffer, LOG_TMPBUF_SIZE, LOG_TMPBUF_SIZE - 1U, ...);
CHK_PRT_CONT(ret == -1, HCCL_WARNING("Failed to build log info"));
```
使用CHK_PRT_CONT(非CHK_PRT_RET): 日志构造失败不中断主流程。

#### 4. EXECEPTION_CATCH 异常捕获 (22处)
包裹所有可能抛异常的表达式(make_shared, insert, erase等):
```cpp
EXECEPTION_CATCH((job = std::make_shared<...>()), return HCCL_E_PARA);
EXECEPTION_CATCH(opBaseHcom.opGroup2CommMap.erase(group), return HCCL_E_MEMORY);
```

#### 5. TIME_NOW() + DURATION_US() 计时 (每个API)
```cpp
HcclUs startut = TIME_NOW();
// ... operation ...
HCCL_RUN_INFO("... take time [%lld]us ...", DURATION_US(TIME_NOW() - startut));
```

#### 6. HCCL_RUN_INFO Entry/Exit 日志对
```cpp
HCCL_RUN_INFO("Entry-%s: ...", __func__, ...);
// ... operation ...
HCCL_RUN_INFO("[HCCL_TRACE]%s success, take time [%lld]us, ...", __func__, ...);
```

#### 7. new (std::nothrow) + CHK_PTR_NULL/CHK_PRT_RET
```cpp
pComm.reset(new (std::nothrow) hccl::hcclComm(...));
CHK_PTR_NULL(pComm);
```

### Architecture Patterns

#### 1. Thin Delegation Layer
op_base不包含任何业务逻辑，所有实际工作委托给hcclComm:
- 集合通信: hcclComm->AllReduceOutPlace / BroadcastOutPlace / ...
- 资源管理: hcclComm->CreateCommResource / AllocComResourceByTiling
- 生命周期: hcclComm->init / Suspend / Resume / SetStopFlag
op_base仅负责: 参数校验 + 版本分派 + 线程安全 + profiling + 日志

#### 2. C API Facade with C++ Internal
对外是C API (extern "C"块包裹):
- HcclAllocComResourceByTiling, HcclCreateComResource, HcclGetAicpuOpStreamNotify等
内部使用C++: static_cast<hccl::hcclComm*>(comm) 恢复类型。
HcclComm = void* 是C ABI的opaque handle。

#### 3. Multiple Init Paths Converging
```
HcclCommInitClusterInfo      ─┐
HcclCommInitClusterInfoConfig ─┤─→ InitCommClusterInfo ─→ pComm->init
HcclCommInitClusterInfoMemConfig ─┘    (do-while清理模式)

HcclCommInitRootInfo         ─┐
HcclCommInitRootInfoConfig   ─┤─→ InitCommRootInfo ─→ pComm->init
                              ┘    (拓扑检测+层级/扁平模式选择)
```
两大初始化路径: ClusterInfo(rankTable文件) vs RootInfo(分布式发现)。
每条路径都有Config/非Config变体和Group异步变体(Wrapper函数)。

#### 4. Barrier = AllReduce on 1 Element
HcclBarrier (op_base_host.cc:181-260) 直接用AllReduceOutPlace实现:
```cpp
HcclDataType dataType = HCCL_DATA_TYPE_FP32;
HcclReduceOp op = HCCL_REDUCE_SUM;
hcclComm->AllReduceOutPlace(tag, hcclComm->barrierSendBuf, hcclComm->barrierRecvBuf,
    HCCL_BARRIER_DEFAULT_COUNT, dataType, op, stream, SyncMode::UNLIMITED_TIMEWAITSYNCMODE);
```

### Anti-Patterns

#### AP1: 4695行超大文件
op_base.cc包含40+个函数，4695行代码。集合通信API入口(13个)模板高度一致但不共享代码，
每个约100-150行。应提取公共模板或使用代码生成。

#### AP2: 通信域初始化配置代码重复(3处x60行)
InitCommClusterInfo、InitCommRootInfo、HcclCreateSubCommConfigInner中
SetDeterministicConfig到SetIndependentOpConfig的8步配置序列几乎逐字重复。
应提取为独立的ApplyCommConfig函数。

#### AP3: getenv热路径调用(6处)
```cpp
const char *indOp = getenv("HCCL_INDEPENDENT_OP");
if (indOp == nullptr || strcmp(indOp, "") == 0) { ... }
```
getenv在每次API调用时执行(在V2分派的lambda内)。应在初始化时读取并缓存。

#### AP4: g_oneSidedCommHcomInfos/g_oneSidedCommSet无锁保护
这两个全局容器的访问(IsOneSidedComm、IsCommNameExistInOneSidedComms等)
没有mutex保护。虽然g_opHcomOneSideMutex存在(L85)，但仅在HcclCommDestroy中使用。
InitOneSidedHcomInfo和DeInitOneSidedHcomInfo等函数直接操作g_oneSidedCommHcomInfos无锁。

#### AP5: Wrapper函数大量代码复制
HcclCommInitClusterInfoWrapper vs HcclCommInitClusterInfo
HcclCommInitClusterInfoConfigWrapper vs HcclCommInitClusterInfoConfig
HcclCommDestroyWrapper vs HcclCommDestroy
每对Wrapper几乎完整复制了主函数逻辑(仅多了hrtSetDevice)。
应提取共享实现。

#### AP6: HcclCommDestroy竞态窗口
```cpp
HcclCommState state = hcclComm->GetState();
if (state == HcclCommState::INUSE) {
    return HCCL_E_AGAIN;  // 检查与后续操作之间有竞态
}
```
GetState和后续操作之间没有原子保护。虽然operatorlock_在算子入口加锁，
但Destroy函数本身没有获取operatorlock_。

#### AP7: 条件编译导致的V2分派位置不一致
部分API在参数校验前分派V2(HcclBatchSendRecvInner):
```cpp
HCCLV2_FUNC_RUN(HcclBatchSendRecvV2(sendRecvInfo, itemNum, comm, stream));
// 后面才做 CHK_PTR_NULL(comm)
```
V2函数可能收到nullptr参数。其他API在校验后才分派V2。

#### AP8: stackLogBuffer作用域问题
```cpp
char stackLogBuffer[LOG_TMPBUF_SIZE]; // 在if外声明
if (GetExternalInputHcclEnableEntryLog()) {
    snprintf_s(stackLogBuffer, ...);  // 在if内填充
}
// ...
if (GetExternalInputHcclEnableEntryLog()) {
    std::string endInfo = ... + std::string(stackLogBuffer); // 在第二个if内使用
}
```
如果entryLog在两次检查间被关闭，stackLogBuffer可能未初始化。
虽然概率极低(运行时配置不太可能改变)，但逻辑上不严谨。

#### AP9: HcclSetConfig大括号错位
```cpp
if (config == HCCL_DETERMINISTIC) {
    // ... 30行逻辑
    }  // <- 这个大括号闭合的是if
return HCCL_SUCCESS;  // <- 此行不在if内，但格式暗示在内
```
(op_base.cc:1837-1871) 缩进与实际逻辑不匹配，极易误读。

### Relationship with Communicator Module

op_base → hcclComm 的调用关系:

1. 通信域创建: hcclComm::init(params, commConfig, rankTable)
2. 集合通信:
   - AllReduceOutPlace(tag, sendBuf, recvBuf, count, dataType, op, stream)
   - BroadcastOutPlace(tag, buf, count, dataType, root, stream)
   - ReduceScatterOutPlace(tag, sendBuf, recvBuf, recvCount, dataType, op, stream)
   - AllGatherOutPlace(tag, sendBuf, recvBuf, sendCount, dataType, stream)
   - ReduceOutPlace(tag, sendBuf, recvBuf, count, dataType, op, root, stream)
   - ScatterOutPlace(tag, sendBuf, recvBuf, recvCount, dataType, root, stream)
   - AlltoAll/AlltoAllV/AlltoAllVC/AlltoAllVOutPlace(...)
   - SendOutPlace / ReceiveOutPlace / BatchSendRecv
3. 资源管理:
   - GetInCCLbuffer / GetOutCCLbuffer
   - CreateCommResource / CreateOpBasedResources
   - AllocComResourceByTiling / Mc2AiCpuStreamAllocAndGet
   - GetCommResource / SetAicpuCommEngine
4. 状态管理:
   - GetState / SetState / SetStopFlag
   - Suspend / Resume
   - GetIdentifier / GetUserRank / GetGroupRank / GetRankSize
5. 辅助:
   - GetAlgType / SetDeterministicConfig / SetAivModeConfig / ...
   - SaveTraceInfo / GetAicpuOpStreamNotify
   - RegisterCommUserMem / DeregisterCommUserMem / ExchangeCommUserMem
   - RegisterWindow / DeregisterWindow / GetCommSymWin

### Notable Patterns

#### 1. Hierarchical Topo Detect (32k threshold)
```cpp
if (nRanks > TOPO_HIERARCHICAL_ENABLE_THRESHOLD) {
    // 分层拓扑检测: Agent → GroupLeader → Member
} else {
    // 扁平拓扑检测: 所有rank直连root
}
```
32k rank以上自动切换为分层发现，减少root的连接压力。
分层路径: Agent连Root → 获取GroupLeader → GroupLeader开启监听 →
Member连GroupLeader → 获取集群信息。

#### 2. ReduceScatterLoop / ReduceLoop 分片执行
当数据量超过CCL buffer大小时，自动分片循环执行:
```cpp
u64 maxCountPerLoop = commInputSize / (rankSize * unitSize);
for (u64 countLeft = count; countLeft > 0; countLeft -= curCount) {
    curCount = ((countLeft * unitSize * rankSize) > commInputSize) ? maxCountPerLoop : countLeft;
    // memcpy到中转buffer → 执行 → memcpy回
}
```
这种分片发生在op_base层而非communicator层内部。

#### 3. GatherAlltoAllV 多线程内存拷贝
RunGather使用16个线程并行拷贝数据:
```cpp
const u32 GATHER_THREAD_NUM = 16;
for (u32 num = 0; num < GATHER_THREAD_NUM; num++) {
    threads[num].reset(new (std::nothrow) std::thread(&GatherMemCopyThread, ...));
}
```
这是图模式动态shape专用路径，不对外公开。

#### 4. 错误码到字符串映射 (HcclGetErrorString)
使用static local map实现，包含24种错误码的字符串描述。
这是面向用户的诊断接口。

#### 5. Symmetric Window API (最新API)
HcclCommSymWinRegister / Deregister / Get 是最新加入的对称内存窗口API:
- 用全局map(winHandle2comm)管理窗口句柄到comm的映射
- 支持flag: HCCL_WIN_COLL_SYMMETRIC (flag=1)，flag=0尚未支持

---

## 2.2.2 hcom/ — 对外接口层深度分析

### Module Overview

hcom/是hcomm对外暴露的最高层API接口模块，直接面向AI框架(PyTorch/MindSpore/GE)提供集合通信服务。
它是一个纯C函数API表面(extern "C")，不包含任何类，所有状态都通过全局变量管理。

在架构中的位置:
```
AI Framework (PyTorch/MindSpore/GE)
        |
    hcom/ (C API surface) <--- 本模块
        |
    communicator/ (hcclComm)
        |
    op_base/ (PyTorch单算子路径)
```

hcom/和op_base/是两个并列的"入口"模块:
- hcom/: 服务GE(图引擎)场景，基于group name查找通信域
- op_base/: 服务PyTorch/HCCL原生API场景，基于HcclComm句柄直接操作

两者最终都调用hcclComm的同名方法(AllReduce/AllGather等)。

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| hcom.cc | 4076 | 核心: 所有对外C API的实现 |
| hcom_common.cc | ~1614 | 全局状态管理、group操作、辅助函数 |
| hcom_common.h | 129 | 核心数据结构定义(HcomInfo/HcclGroupParams) |
| hcom_pub.h | 170 | 对外API声明(public header) |
| hcom_private.h | 60 | 内部函数声明 |
| hcom_common_v2.h | 27 | V2 init/group弱符号声明(8个) |
| hcom_private_v2.h | 116 | V2通信操作弱符号声明(~60个) |
| hcom_host_profiling.h | 63 | Host侧profiling结构和接口 |
| hcom_host_profiling.cc | 114 | Host侧profiling实现 |
| hcom_device_profiling.h | 62 | Device侧profiling结构和接口 |
| hcom_device_profiling.cc | 126 | Device侧AICPU profiling实现 |
| gradient_segment/gradient_segment.h | 69 | 梯度切分类定义 |
| gradient_segment/gradient_segment.cc | 309 | 梯度切分算法实现 |

### Core Data Structures

#### HcomInfo — 通信域全局上下文

```cpp
// hcom_common.h:80-107
using HcomInfo = struct HcomInfoTag {
    HcclCommPtr pComm;                    // WORLD_GROUP的通信域 (shared_ptr<hcclComm>)
    void *psComm;                          // 备用
    hccl::HcclCommParams params;           // 通信参数(rank, totalRanks, serverId等)
    std::unordered_map<std::string, HcclGroupParams> hcomGroupMap;  // group name -> 子通信域
    std::mutex groupParamsLock;            // 保护hcomGroupMap
    hccl::RankTable_t rankTable;           // rank表
    s32 devId;
    bool cloudFlag;                        // 云 vs 实验室场景
    bool isHcomInit;                       // PyTorch单算子通信域复用标记
    std::mutex backloggedGroupLock;
    std::map<std::string, std::vector<u32>> backloggedGroup;  // 待创建的group(init前预注册)
    // TopoInfoDetect用于重连
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectServer;
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectAgent;
    std::mutex groupRankNumMapLock;
    std::unordered_map<std::string, u32> groupRankNumMap;  // 用于topoInfo设置
};
```

#### HcomInfoCtx — 设备级上下文包装

```cpp
// hcom_common.cc:84-95
using HcomInfoCtx = struct HcomInfoCtxTag {
    HcomInfo hcomInfo;                     // 主通信信息
    shared_ptr<RemoteAccess> remoteAccess; // 远程内存访问
    vector<MemRegisterAddr> remoteAddrInfos;
    HcomOpTagInfo opTagInfo;               // 按op tag索引的计数器
    bool isUsed;                           // 是否已激活
};
```

全局实例: `HcomInfoCtx g_hcomInfoCtx[MAX_MODULE_DEVICE_NUM + 1]`，下标即设备逻辑ID，最后一个元素是兜底槽位。

#### HcclGroupParams — 子通信域参数

```cpp
// hcom_common.h:67-78
using HcclGroupParams = struct TagHcclGroupParamsInfo {
    u32 worldRank;
    u32 groupRank;
    u32 serverNum;
    u32 totalRanks;
    std::vector<u32> groupRanks;  // 下标=groupRank, 值=worldRank
    HcclCommPtr pSubComm;         // 子通信域
    u32 refCounter = 0;           // 引用计数(用于安全销毁)
    bool destroyFlag = false;     // 销毁标记
};
```

#### HcomProInfo — Profiling信息结构(重复定义)

```cpp
// hcom_device_profiling.h:24-42 和 hcom_host_profiling.h:24-42 (两处完全相同)
typedef struct HcomProInfo {
    uint8_t dataType;
    uint8_t cmdType;
    uint64_t dataCount;
    uint32_t rankSize;
    uint32_t userRank;
    uint32_t blockDim = 0;
    uint64_t beginTime;
    uint32_t root;
    uint32_t slaveThreadNum;
    uint64_t commNameLen;
    uint64_t algTypeLen;
    char tag[MAX_LENGTH];       // MAX_LENGTH=128
    char commName[MAX_LENGTH];
    char algType[MAX_LENGTH];
    bool isCapture = false;
    bool isAiv = false;
    uint8_t reserved[MAX_LENGTH];
} HcomProInfo;
```

### Two Parallel API Families

hcom/对外暴露两套并行的API，服务不同的AI框架集成路径:

1. Group-Based API (GE/MindSpore路径):
   - 函数签名: `Hcom*(..., const char *group, rtStream_t stream)`
   - 通信域查找: `HcomGetCommByGroup(group, hcclComm)`
   - 代表: HcomAllReduce, HcomAllGather, HcomBroadcast, HcomSend, HcomReceive
   - group可为nullptr，默认使用HCCL_WORLD_GROUP

2. OpBaseHcom API (PyTorch单算子通信域复用路径):
   - 函数签名: `HcclCommGraph*(..., s64 opBaseHcom, rtStream_t stream)`
   - 通信域获取: `reinterpret_cast<hccl::hcclComm*>(opBaseHcom)` (直接转型)
   - 代表: HcclCommGraphAllReduce, HcclCommGraphAllGather, HcclCommGraphSend
   - opBaseHcom是由PyTorch通过HcclCommInitRootInfo创建的hcclComm指针强转为s64

量化对比:
- group-based API调用HcomGetCommByGroup: 36次
- opBaseHcom API使用reinterpret_cast: 27次
- 两者最终都调用相同的hcclComm方法(AllReduce, AllGather等)

### Communication API Implementation Template

hcom.cc中的通信API函数遵循高度统一的模板，所有API(AllReduce/AllGather/Broadcast/Send/Receive/ReduceScatter/AlltoAllV等)都是同一模板的变体:

```cpp
HcclResult HcomXxx(const char *tag, void *inputPtr, void *outputPtr, u64 count,
    HcclDataType dataType, ..., const char *group, rtStream_t stream)
{
    // 1. 双时间戳: 用户侧耗时 + profiling侧耗时
    HcclUs startut = TIME_NOW();                    // 52处
    uint64_t beginTime = hrtMsprofSysCycleTime();   // 21处

    // 2. 获取当前设备ID(用于日志)
    s32 deviceLogicId = 0;
    CHK_RET(hrtGetDevice(&deviceLogicId));

    // 3. count==0 早返回(WARNING级别，非ERROR)
    CHK_PRT_RET(count == 0, HCCL_WARNING("input count is 0, return Xxx success"), HCCL_SUCCESS);  // 12处

    // 4. 参数校验: RPT_INPUT_ERR + CHK_PTR_NULL 双重检查
    RPT_INPUT_ERR(inputPtr == nullptr, "EI0003", ...);  // 53处
    CHK_PTR_NULL(inputPtr);
    RPT_INPUT_ERR(outputPtr == nullptr, "EI0003", ...);
    CHK_PTR_NULL(outputPtr);
    RPT_INPUT_ERR(stream == nullptr, "EI0003", ...);
    CHK_PTR_NULL(stream);

    // 5. 获取stream ID(用于日志)
    s32 streamId = 0;
    CHK_RET(hrtGetStreamId(stream, streamId));

    // 6. group默认值处理
    std::string strGroup = (group == nullptr) ? HCCL_WORLD_GROUP : group;

    // 7. 入口日志(HCCL_USER_CRITICAL_LOG = 最高可见度)
    HCCL_USER_CRITICAL_LOG("Entry-HcomXxx:tag[%s], ..., streamId[%d], deviceLogicId[%d]", ...);  // 13处

    // 8. 内存属性打印(设备内存/主机内存/统一内存)
    CHK_RET(PrintMemoryAttr(inputPtr));     // 36处
    CHK_RET(PrintMemoryAttr(outputPtr));

    // 9. V2分派(核心机制 —— 见下一节)
    HCCLV2_FUNC_RUN(HcomXxxV2(tag, inputPtr, outputPtr, count, dataType, ..., group, stream));  // 47处

    // 10. 参数校验(业务级)
    CHK_RET(HcomCheckOpParam(tag, count, dataType, group, stream));  // 22处

    // 11. 获取通信域
    std::shared_ptr<hccl::hcclComm> hcclComm;
    CHK_RET(HcomGetCommByGroup(strGroup.c_str(), hcclComm));

    // 12. 调用核心通信操作
    HcclResult ret = hcclComm->Xxx(tag, inputPtr, outputPtr, count, dataType, ..., stream);
    CHK_PRT_RET(ret != HCCL_SUCCESS, HCCL_ERROR("[Xxx][Result]errNo[0x%016llx] hcclComm Xxx error, ..."), ret);

    // 13. Profiling上报
    CHK_RET(CallMsprofReportHostApi(hcclComm.get(), HcclCMDType::HCCL_CMD_XXX, beginTime, count, dataType));  // 24处

    // 14. 成功日志(HCCL_RUN_INFO)
    HCCL_RUN_INFO("hcom Xxx success,take time [%lld]us, tag[%s], ...",
        DURATION_US(TIME_NOW() - startut), tag, ...);  // 47处

    return HCCL_SUCCESS;
}
```

OpBaseHcom API变体的差异:
- 不调用HcomGetCommByGroup，而是: `hccl::hcclComm* hcclComm = reinterpret_cast<hccl::hcclComm*>(opBaseHcom);`
- 额外调用: `CHK_RET(SetWorkflowMode(HcclWorkflowMode::HCCL_WORKFLOW_MODE_OPS_KERNEL_INFO_LIB));`
- 入口日志用HCCL_RUN_INFO而非HCCL_USER_CRITICAL_LOG

### V1/V2 Dispatch Mechanism

#### 机制原理

V2分派是hcom/的核心版本演进机制，允许在不修改V1代码的情况下渐进式地引入V2实现。

宏定义(param_check_basic_v2.h:14-21):
```cpp
#define HCCLV2_FUNC_RUN(func, ...) \
    do { \
        const char *socNamePtr = aclrtGetSocName(); \
        CHK_PTR_NULL(socNamePtr);                   \
        if (IsSupportHCCLV2(socNamePtr)) {           \
            return func;                              \
        }                                             \
    } while (0)
```

工作流程:
1. 获取当前芯片型号(SoC name)
2. IsSupportHCCLV2判断该芯片是否支持V2(基于SoC类型表)
3. 如果支持: 直接`return func;`调用V2函数并返回其结果，V1代码不会执行
4. 如果不支持: 宏结束，继续执行V1代码

V2函数通过`__attribute__((weak))`声明:
- hcom_common_v2.h: 8个init/group相关V2弱符号
- hcom_private_v2.h: ~60个通信/资源操作V2弱符号

弱符号含义: 如果链接时V2实现的.so被加载，弱符号解析为实际函数；否则为nullptr。
但注意: HCCLV2_FUNC_RUN宏并不检查nullptr，而是用IsSupportHCCLV2作为gate。

AICPU编译路径下V2完全禁用:
```cpp
// aicpu/param_check_basic_v2.h:14
#define HCCLV2_FUNC_RUN(func, ...)  // 空宏
```

#### 分布

V2弱符号覆盖了几乎所有公共API:
- 集合通信: AllReduce, AllGather, Broadcast, Reduce, ReduceScatter, Send, Receive, AlltoAll系列
- Group操作: CreateGroup, DestroyGroup, GetRankSize, GetRankId, GetWorldRankFromGroupRank
- 资源查询: GetWorkspaceSubStreamNum, GetWorkspaceMemSize, GetAlltoAllStagedWorkSpaceMemSize
- CCL Buffer: CreateCommCclBuf, GetInCclBuf, GetOutCclBuf (group和graph两套)
- 算法选择: SelectAlg (group和graph两套)
- 初始化: InitByFile, InitByString, Destroy

### HcomInit / HcomDestroy Complete Flow

#### HcomInit (hcom.cc:67-162)

```
HcomInit(rankTableM, identify, commWorkMode)
  |-- CHK_PTR_NULL(rankTableM, identify)             // 参数校验
  |-- CHK_PRT_RET(pComm != nullptr)                  // 防重复初始化
  |-- do { ... } while(0) 伪循环:                    // 使用CHK_PRT_BREAK实现go-to-end
  |     |-- InitHcomMiscInfo(params, rankTable)       // 解析rank表基本信息
  |     |-- hrtGetDevice(&logicDevId)                 // 获取当前设备
  |     |-- CfgGetClusterInfo(...)                    // 解析集群拓扑信息
  |     |-- [310P多服务器] InitExternalInputHeterog() // 异构场景初始化
  |     |-- new(nothrow) hcclComm(0, 0, WORLD_GROUP)  // 创建通信域
  |     |-- pComm->init(params, commConfig, rankTable)// 核心初始化
  |     |-- ShowRanktableConfigInfo(...)               // 日志输出拓扑
  |     |-- InitWorkflowMode(OPS_KERNEL_INFO_LIB)     // 设置工作流模式
  |     |-- HcomFlushBackloggedGroups()                // 刷新预注册的group
  |     \-- HcomSetGroupTopoInfo(...)                  // 注册topo信息
  |-- if (errorFlag) { HcomDestroy(); return ret; }   // 失败时清理
  \-- return HCCL_SUCCESS
```

关键设计:
- do-while(0) + CHK_PRT_BREAK + errorFlag: C风格的"try-finally"模式，避免goto但实现等价的清理逻辑
- 失败时调用HcomDestroy()进行完整清理，保证状态一致性
- Backlogged groups: 允许在HcomInit之前调用CreateGroup(预注册)，Init时自动flush

#### HcomDestroy (hcom_common.cc:987-1050)

```
HcomDestroy()
  |-- HCCLV2_FUNC_RUN(HcomDestroyV2())                // V2优先
  |-- lock(g_destroyDeviceLock)                        // 全局销毁锁
  |-- for (i = 0..MAX_MODULE_DEVICE_NUM):              // 遍历所有设备槽位
  |     |-- if (!isHcomInit):
  |     |     |-- pComm = nullptr
  |     |     \-- g_hcomDestroyCallback(hcomInfo) if exists
  |     |-- else:
  |     |     |-- HcomUnSetGroupTopoInfo(...)
  |     |     |-- [heterog] 设置正确的device context
  |     |     |-- HcomDestroyOneDevice(hcomInfo):
  |     |     |     |-- ClearCheckInfo()
  |     |     |     |-- hcomGroupMap.clear()
  |     |     |     |-- backloggedGroup.clear()
  |     |     |     |-- rankTable clear
  |     |     |     |-- g_segmentIdxMap.clear()
  |     |     |     |-- g_segmentSizeMap.clear()
  |     |     |     \-- ProfilingManagerPub::ClearStoragedProfilingInfo()
  |     |     |-- g_hcomDestroyCallback(hcomInfo) if exists
  |     |     \-- pComm = nullptr; topoInfoDetect.clear()
  \-- return HCCL_SUCCESS
```

关键设计:
- 遍历所有设备槽位(MAX_MODULE_DEVICE_NUM+1)而非仅当前设备
- 全局销毁锁保证多线程安全
- group资源在world group之前销毁(注意hcomGroupMap.clear()在pComm=nullptr之前)
- 回调函数模式: g_hcomDestroyCallback允许异构场景挂载自定义清理逻辑

### Group Management (Communication Domain)

#### HcomGetCommByGroup (hcom_common.cc:234-257)

核心路由逻辑，区分WORLD_GROUP和子group:
```cpp
HcclResult HcomGetCommByGroup(const char *group, shared_ptr<hcclComm> &hcclComm)
{
    HcomInfo &hcomInfo = HcomGetCtxHomInfo();
    string strGroup = (group == nullptr) ? HCCL_WORLD_GROUP : group;
    if (strGroup == HCCL_WORLD_GROUP) {
        hcclComm = hcomInfo.pComm;        // 直接返回world comm
    } else {
        lock(hcomInfo.groupParamsLock);     // 加锁
        auto iter = hcomInfo.hcomGroupMap.find(strGroup);
        if (iter == end) {
            // 降级: 查找op_base的opGroup2CommMap
            ret = HcclGetCommHandle(strGroup.c_str(), hcclComm);
        } else {
            hcclComm = iter->second.pSubComm;
        }
    }
}
```

两级查找:
1. hcomInfo.hcomGroupMap — hcom自己创建的group
2. HcclGetCommHandle — op_base管理的通信域(PyTorch HcclCommInitRootInfo创建的)

#### Group Create (hcom.cc中HcomCreateGroupImpl)

流程: 参数校验 -> 检查重复 -> GetRankList -> pComm->CreateGroup -> 插入hcomGroupMap
线程安全: groupParamsLock保护map的读写

#### Group Destroy (DestroyFlag + busy-wait)

```
DestroyFlag(group, true)   // 设置destroyFlag
  |
  \-- busy-wait on refCounter == 0
      |
      \-- pComm->DestroyGroup(...)
          |
          \-- hcomGroupMap.erase(group)
```

refCounter机制: 通信操作进入时++，退出时--，确保正在进行的通信不被中途销毁。
注意: busy-wait无退避(反模式)。

#### Backlogged Groups (延迟创建机制)

允许在HcomInit之前调用HcomCreateGroup:
1. HcomStoreBackloggedGroup: 将group和rankIds存入backloggedGroup map
2. HcomInit -> HcomFlushBackloggedGroups: 遍历map，逐个调用CreateGroup
3. 场景: GE图构建时需要创建group，但此时HcomInit可能尚未调用

### Global State Management

#### 全局变量清单 (hcom_common.cc)

```cpp
std::mutex g_hcomInfoCtxMutex;                          // 保护g_hcomInfoCtx
HcomInfoCtx g_hcomInfoCtx[MAX_MODULE_DEVICE_NUM + 1];   // per-device上下文数组
std::mutex g_backloggedGroupLock;
std::map<std::string, std::vector<u32>> g_backloggedGroup;
std::mutex g_destroyDeviceLock;                          // 销毁全局锁
static std::mutex g_taskNumCalModeMutex;
static bool g_isAutoTuneModeOpen = false;
static bool g_notSupportSecAddrCopyWithOffset = false;
```

回调函数指针(hcom_common.cc:74-82):
```cpp
HcomCreateGroupCallback g_hcomCreateGroupCallback = nullptr;
HcomCallBackGroupIsInit g_hcomCallBackGroupIsInit = nullptr;
HcomDestroyGroupCallback g_hcomDestroyGroupCallback = nullptr;
HcomDestroyCallback g_hcomDestroyCallback = nullptr;
HcomSetGroupTopoInfoPtr g_hcomSetGroupTopoInfo = nullptr;
HcomUnsetGroupTopoInfoPtr g_hcomUnsetGroupTopoInfo = nullptr;
```

梯度切分全局变量(hcom_common.cc:98-104):
```cpp
namespace hccl {
    std::map<std::string, std::vector<u32>> g_segmentIdxMap;
    std::map<std::string, std::vector<float>> g_segmentSizeMap;
    std::mutex g_segmentIdxMapLock;
    std::mutex g_segmentSizeMapLock;
    std::mutex g_setTaskNumCalModeLock;
}
```

#### 线程安全分析

| 全局状态 | 保护方式 | 评估 |
|----------|----------|------|
| g_hcomInfoCtx | g_hcomInfoCtxMutex | HcomGetCurHcomCtx加锁；但拿到引用后无锁保护 |
| hcomGroupMap | groupParamsLock | 读写都加锁 |
| backloggedGroup | backloggedGroupLock | 加锁 |
| g_segmentIdxMap/SizeMap | g_segmentIdx/SizeMapLock | 加锁 |
| 回调函数指针(6个) | 无保护 | 依赖初始化顺序(constructor属性设置) |
| g_isAutoTuneModeOpen | 无保护 | 单一布尔值，实践中单线程设置 |

HcomGetCurHcomCtx的设备查找策略(hcom_common.cc:119-148):
1. 有deviceId且在范围内 -> g_hcomInfoCtx[deviceId]
2. 首次使用该deviceId -> 检查兜底槽位是否已配置
3. 无deviceId -> 线性扫描找已使用的槽位
4. 都没有 -> 使用兜底槽位[MAX_MODULE_DEVICE_NUM]

### Profiling Integration

#### 双侧分离设计

Host侧和Device侧使用完全相同的HcomProInfo结构但不同的上报机制:

Host侧(hcom_host_profiling.cc):
- HcommProfilingReportKernel: ProfilingManagerPub::CallMsprofReportNodeInfo (kernel时间)
- HcommProfilingReportOp: ProfilingManagerPub::CallMsprofReportHostApi (API调用)
- HcommProfilingRegThread: HCCL_PROFILER_ADD_TAG/STREAM宏注册stream
- HcommProfilingUnRegThread: HCCL_PROFILER_DEL_TAG/STREAM宏注销

Device侧(hcom_device_profiling.cc, #ifdef CCL_KERNEL_AICPU):
- HcommProfilingInit: 记录各stream的起始SQE索引
- HcommProfilingReportMainStreamAndFirstTask/LastTask: 上报主流首尾task
- HcommProfilingReportDeviceHcclOpInfo: dfx::ProfilingManager::ReportHcclOpInfo
- HcommProfilingEnd: dfx::ProfilingManager::ReportTaskInfo + 清理

hcom.cc中通信API的profiling调用链路:
```
HcomAllReduce(...)
  |-- beginTime = hrtMsprofSysCycleTime()             // 进入时记录
  |-- ... (实际通信) ...
  \-- CallMsprofReportHostApi(hcclComm, CMD_ALLREDUCE, beginTime, count, dataType)  // 退出时上报
```

#### HcomProInfo结构重复定义问题

hcom_device_profiling.h和hcom_host_profiling.h各自独立定义了完全相同的HcomProInfo和MAX_LENGTH。
这两个头文件不会在同一编译单元中同时include(device用前者，host用后者)，因此不会直接产生编译错误。
但这是一个维护风险 —— 若修改一处忘记同步另一处，会导致隐蔽的二进制不兼容问题。

### Gradient Segment

GradientSegment类实现深度学习梯度的融合切分策略。

#### 三级策略优先级

```
GetGradientSegmentExecutor
  |-- 1. GetSegmentBySize (用户配置的按数据量切分)
  |     \-- 查g_segmentSizeMap: 用户通过HcomSetGradFusionBySize预设
  |-- 2. GetSegmentByIndex (用户配置的按层数切分, 仅在非FORCE_SIZE时)
  |     \-- 查g_segmentIdxMap: 用户通过HcomSetGradFusionByIndex预设
  \-- 3. GetSegmentByDefaultRatio (内置默认策略)
        |-- KNOWN_SHAPE: 两段切分(96.54% + 3.46%)
        \-- UNKNOWN_SHAPE: 固定大小切分(受CCL buffer限制)
```

96.54/3.46比率来源: ResNet-50模型的梯度分布特征(代码注释明确说明)。

#### 二分查找算法

GetIdxByBinarySearch在累积梯度大小的有序列表中找到目标大小对应的层索引:
- 精确匹配: fabs差值 < 1e-6
- 未精确匹配: GetNearIdxByDataSize找最近的
- UNKNOWN_SHAPE特殊处理: 如果二分极限值仍小于目标，直接返回当前idx(防溢出)

### hcom.cc 文件组织分析 (4076行)

#### 功能区块

| 行号范围 | 行数 | 功能 |
|----------|------|------|
| 1-66 | 66 | Includes + CallMsprofReportHostApi |
| 67-350 | 284 | Init系列: HcomInit, HcomInitByString, HcomInitByMasterInfo, HcomNormalInit |
| 351-855 | 505 | Group-based通信API: AllGather/AllReduce/Broadcast/Reduce/ReduceScatter/Send/Receive + V变体 |
| 856-1270 | 415 | OpBaseHcom通信API: HcclCommGraph系列 |
| 1271-1600 | 330 | Group管理/查询/配置 |
| 1600-1900 | 300 | Workspace/资源查询/CCL Buffer |
| 1900-2450 | 550 | AlltoAll系列(group和graph两套) |
| 2450-2800 | 350 | 杂项: UnloadTask, OverFlow, QoS, AIV, Deterministic |
| 2800-4076 | 1276 | 离线编译Task计算: CalcTaskNum系列(最大的功能块) |

Task计算区块(2800-4076)占文件31%，包含CalcTaskNum/GetStreamNumOfflineComp/GetIntraComTaskNum等~30个函数，
用于离线编译时精确计算所需的task数量和stream数量。这部分逻辑高度依赖算法类型(AlgType)和设备类型(DevType)，
涉及大量分支和常量。

#### 文件臃肿问题

4076行的单文件包含了:
- 初始化/销毁
- ~20个集合通信API(group-based)
- ~15个集合通信API(opBaseHcom-based)
- Group管理
- 资源查询
- Task数量计算(离线编译)

这些功能按职责至少应拆分为4-5个文件。当前的组织使得任何改动都有高概率与其他开发者冲突。

### hcom与op_base的关系

hcom/和op_base/是两个平行的"入口"层，不存在调用关系:

```
          AI Framework
         /            \
   GE(graph mode)   PyTorch(eager mode)
        |                |
    hcom/ API        op_base/ API
        |                |
        +--- hcclComm ---+    <-- 共享同一个通信域对象
```

hcom/ -> op_base/的唯一依赖:
- hcom.cc:29 `#include "../op_base/src/op_base.h"` — 获取op_base的HcclOpInfoCtx类型
- HcomGetCommHandleByGroup中查询op_base的opGroup2CommMap作为降级路径

op_base/ -> hcom/: 无依赖

### Idioms & Conventions (惯用法)

#### 1. RPT_INPUT_ERR + CHK_PTR_NULL 双重检查 (53处 RPT_INPUT_ERR)

每个指针参数都先报告错误(RPT_INPUT_ERR)再检查null(CHK_PTR_NULL):
```cpp
RPT_INPUT_ERR(inputPtr == nullptr, "EI0003",
    std::vector<std::string>({"ccl_op", "parameter", "value", "tips"}),
    std::vector<std::string>({"HcomAllReduce", "inputPtr", "nullptr", "please check inputPtr"}));
CHK_PTR_NULL(inputPtr);
```
RPT_INPUT_ERR是错误日志上报(IDE/运维系统可追踪)，CHK_PTR_NULL是防御性返回。
两者必须成对出现，不能只有其一。

#### 2. count==0 早返回模式 (12处)

```cpp
CHK_PRT_RET(count == 0, HCCL_WARNING("input count is 0, return AllReduce success"), HCCL_SUCCESS);
```
count==0视为合法输入(WARNING而非ERROR)，直接返回成功。
这是NCCL的语义: 空操作应成功而非报错。

#### 3. group nullptr默认值 (一致模式)

```cpp
std::string strGroup = (group == nullptr) ? HCCL_WORLD_GROUP : group;
```
所有API中group参数可为nullptr，默认映射到HCCL_WORLD_GROUP。

#### 4. 入口-出口日志对

入口: HCCL_USER_CRITICAL_LOG("Entry-HcomXxx:tag[%s], ..., streamId[%d], deviceLogicId[%d]")
出口: HCCL_RUN_INFO("hcom Xxx success,take time [%lld]us, tag[%s], ...")

USER_CRITICAL_LOG用于入口(高可见度，运维可追踪)，RUN_INFO用于出口(常规运行日志)。

#### 5. do-while(0) + errorFlag 初始化模式

HcomInit使用do-while(0) + CHK_PRT_BREAK + errorFlag实现C风格的异常处理:
```cpp
bool errorFlag = false;
do {
    ret = Step1(); CHK_PRT_BREAK(ret != HCCL_SUCCESS, LOG, errorFlag = true);
    ret = Step2(); CHK_PRT_BREAK(ret != HCCL_SUCCESS, LOG, errorFlag = true);
    ...
} while (0);
if (errorFlag) { Cleanup(); return ret; }
```
优点: 避免goto
缺点: 所有步骤挤在一个do块中，可读性差

#### 6. 回调函数指针安装模式

```cpp
HcomGroupCallbackFuncInstall(p1, p2, p3, p4)
  -> g_hcomCreateGroupCallback = p1;
     g_hcomCallBackGroupIsInit = p2;
     ...
```
通过__attribute__((constructor))在库加载时自动注册回调，实现异构场景的扩展:
```cpp
__attribute__((constructor))
void RegisterHeterogCallbacks() {
    HcomGroupCallbackFuncInstall(...);
}
```

### Anti-Patterns (反模式)

#### AP-1: HcomProInfo重复定义 (ODR风险)

位置: hcom_device_profiling.h:24-42 和 hcom_host_profiling.h:24-42
问题: 完全相同的struct定义在两个头文件中各写一遍
风险: 修改一处忘记同步另一处，导致Host/Device间profiling数据序列化/反序列化不兼容
正确做法: 提取到公共头文件(如hcom_profiling_common.h)

#### AP-2: static map在header中定义 (ODR违规)

位置: hcom_common.h:55-60
```cpp
static std::unordered_map<s32, u64> OFFLINE_BUILD_SUB_STEAM_NUM = {...};
```
问题: static变量在每个包含此头文件的编译单元中创建一份独立副本。
如果有N个.cc文件include此头文件，就有N个独立的map实例，浪费内存且不一致。
正确做法: 声明为inline constexpr(C++17)或extern声明+单一.cc定义

#### AP-3: reinterpret_cast<hcclComm*>(opBaseHcom) 无校验 (27处)

位置: hcom.cc所有HcclCommGraph*函数
```cpp
hccl::hcclComm* hcclComm = reinterpret_cast<hccl::hcclComm*>(opBaseHcom);
```
问题: opBaseHcom是AI框架传入的s64值，如果传入错误值(野指针)会直接crash
正确做法: 至少添加nullptr检查; 理想情况下维护一个有效句柄注册表做校验

#### AP-4: 4076行单文件 (God File)

位置: hcom.cc
问题: 单文件包含初始化、通信API、group管理、资源查询、task计算五类功能
影响: 合并冲突概率高、代码导航困难、编译时间长
正确做法: 按功能拆分为hcom_init.cc, hcom_comm_ops.cc, hcom_graph_ops.cc, hcom_group.cc, hcom_task_calc.cc

#### AP-5: Group destroy busy-wait无退避

位置: hcom.cc中DestroyFlag相关逻辑
问题: busy-wait等待refCounter降为0，无sleep/退避/超时
风险: 如果通信操作hang住，destroy线程会永久自旋，浪费CPU

#### AP-6: 回调函数指针无锁保护

位置: hcom_common.cc:74-82 (6个全局函数指针)
问题: g_hcomCreateGroupCallback等指针在constructor中设置、在API中读取，无内存屏障保护
实践中安全(constructor在main之前执行)，但理论上有可见性问题

#### AP-7: HcomGetCurHcomCtx中拿到引用后无锁保护

位置: hcom_common.cc:119-148
问题: 函数内部持锁找到正确的HcomInfoCtx&，但返回引用后锁释放。
调用者对返回的引用进行读写时不再持有g_hcomInfoCtxMutex。
如果多线程同时操作不同设备的HcomInfo则安全(不同数组元素)，
但如果操作同一设备(通过不同线程)则存在竞争。

#### AP-8: HCCLV2_FUNC_RUN在参数校验之后调用

位置: hcom.cc多数通信API
问题: V2分派(HCCLV2_FUNC_RUN)在RPT_INPUT_ERR/CHK_PTR_NULL之后、HcomCheckOpParam之前调用。
如果V2接管执行，部分参数校验(HcomCheckOpParam)会被跳过。
V2实现需要自行做完整校验，否则可能接受非法参数。

### Notable Patterns

#### 1. 异构场景回调函数安装

通过__attribute__((constructor))在库加载时自动注册异构场景的回调:
```
hcom_common.cc定义回调指针(初始为nullptr)
  -> HcomGroupCallbackFuncInstall设置回调
  -> HcomInit/CreateGroup/Destroy中检查并调用回调
```
这种模式允许异构集合通信(如CPU+NPU混合)在不修改hcom/核心代码的情况下扩展行为。

#### 2. 双查找降级: hcom group -> op_base comm

HcomGetCommByGroup中的降级路径:
```
hcomGroupMap.find(group) -> 失败 -> HcclGetCommHandle(group, hcclComm)
```
HcclGetCommHandle会去op_base的opGroup2CommMap中查找，
这允许PyTorch通过HcclCommInitRootInfo创建的通信域也能被hcom API使用。

#### 3. Task计算公式体系 (离线编译)

CalcTaskNum等函数封装了复杂的task数量计算逻辑，涉及:
- 算法类型(Ring/Mesh/HD/NHR等)决定通信步数
- 设备数决定AllReduce/AllGather的内部task
- 服务器数决定server间通信task
- Pipeline slicing决定stream间同步task
- DFX预留task数(16个)

这些计算用于GE图编译阶段预分配资源，不在运行时执行。

#### 4. HcomInfoCtx兜底查找机制

HcomGetCurHcomCtx的三级降级:
1. 当前设备ID -> 对应槽位
2. 首次使用且兜底槽位有配置 -> 兜底槽位
3. 无设备ID -> 线性扫描已使用槽位
4. 全空 -> 激活兜底槽位

这种设计处理了"HcomInit在线程A调用、通信API在线程B调用(可能未setDevice)"的场景。

#### 5. HcomProInfo 固定大小C结构

HcomProInfo使用固定大小的char数组(MAX_LENGTH=128)而非std::string:
- 目的: 支持Host-Device间通过HDC/共享内存传输profiling数据
- 设备侧(AICPU)运行时无std::allocator，不能使用堆分配的string
- reserved[128]字段预留未来扩展，保持二进制兼容


---

## 2.2.2 op_base/ — HCCL原生API入口层分析

### Module Overview

op_base/是hcomm面向HCCL原生API(HcclAllReduce/HcclCommInitRootInfo等)的入口模块，
服务PyTorch等通过HcclComm句柄直接操作通信域的场景。

与hcom/的关系:
- hcom/: GE图引擎路径，基于group name
- op_base/: HCCL原生API路径，基于HcclComm句柄
- 两者结构高度镜像: 都是平坦C函数，都用全局数组管理per-device状态，都有V2分派

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| op_base.cc | 4695 | 核心: HCCL原生API实现 |
| op_base_host.cc | 260 | Host侧辅助功能 |
| op_base_device.cc | 55 | Device侧(AICPU)辅助 |
| op_base_mc2.cc | 260 | MC2相关通信资源分配 |
| op_base.h | 112 | 头文件: HcclOpInfoCtx定义 |
| op_base_v2.h | 189 | V2弱符号声明(~70个) |

### HcclOpInfoCtx — op_base的全局上下文

```cpp
// op_base.h:26-44
using HcclOpInfoCtx = struct HcclInfoTag {
    HcclCommPtr pComm;
    hccl::HcclCommParams params;
    hccl::RankTable_t rankTable;
    bool cloudFlag = false;
    bool isUsed;
    std::mutex opGroupMapMutex;
    std::unordered_map<std::string, std::shared_ptr<hccl::hcclComm>> opGroup2CommMap;
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectServer;
    std::map<std::string, std::shared_ptr<hccl::TopoInfoDetect>> hcclCommTopoInfoDetectAgent;
};
```

与HcomInfo的核心区别:
- op_base不需要hcomGroupMap(无group概念)
- op_base通过opGroup2CommMap保存以comm name为key的通信域
- op_base使用thread_local g_hcclDeviceId而非每次hrtGetDevice

全局实例: `HcclOpInfoCtx g_opHcomInfos[MAX_MODULE_DEVICE_NUM + 1]` (与hcom相同的per-device数组模式)

### op_base与hcom的结构镜像

两个模块遵循近乎相同的实现模式:

| 维度 | hcom/ | op_base/ |
|------|-------|----------|
| 核心文件 | hcom.cc (4076行) | op_base.cc (4695行) |
| 上下文数据结构 | HcomInfo + HcomInfoCtx | HcclOpInfoCtx |
| 全局状态 | g_hcomInfoCtx[MAX+1] | g_opHcomInfos[MAX+1] |
| 状态互斥 | g_hcomInfoCtxMutex | g_opHcomInfosMutex |
| 设备ID查找 | HcomGetCurHcomCtx | GetHcclExistDeviceOpInfoCtx |
| V2分派 | hcom_private_v2.h (~60弱符号) | op_base_v2.h (~70弱符号) |
| Profiling上报 | CallMsprofReportHostApi(无tag) | CallMsprofReportHostApi(带tag) |
| API数量 | ~35个通信+管理 | ~40个通信+管理 |
| 通信域获取 | HcomGetCommByGroup(group) | 直接使用HcclComm句柄 |

### op_base的V2弱符号

op_base_v2.h定义了~70个V2弱符号，覆盖:
- 通信域生命周期: CommInitClusterInfo, CommInitRootInfo, CommInitAll, CommDestroy, CommSuspend, CommResume
- 集合通信: AllReduce, AllGather, Broadcast, Reduce, ReduceScatter, Send, Recv, BatchSendRecv, AlltoAll系列, Scatter, Barrier
- 资源管理: AllocComResourceByTiling, DevMemAcquire, GetHcclBuffer, CommResPrepare, SetMemoryRange, ActivateCommMemory
- 拓扑查询: GetNetLayers, GetInstSizeByNetLayer, GetRanksByNetLayer, GetLinks, GetTopoType (V2新增大量拓扑查询API)
- 配置: SetConfig, GetConfig, SetOpAlgConfig, SetOpCommEngine

V2在op_base中的覆盖面更广，特别是拓扑查询和对称内存API是V2独有的。

### op_base的thread_local设备ID

```cpp
// op_base.cc:83
thread_local s32 g_hcclDeviceId = INVALID_INT;
```

op_base使用thread_local缓存设备ID(首次查询后缓存):
- 优势: 避免每次API调用都hrtGetDevice
- 风险: 如果同一线程切换了device(SetDevice)，缓存不更新
- HcclDeviceRefresh()函数强制刷新缓存

### op_base与hcom的交叉依赖

hcom/ -> op_base/:
- hcom.cc include op_base.h
- HcomGetCommByGroup降级查找op_base的opGroup2CommMap
- HcomGetCommHandleByGroup优先查op_base再查hcom

op_base/ -> hcom/:
- op_base.cc include hcom_common.h
- op_base使用HcomInfo的部分工具函数

两者共同依赖的公共组件:
- hcclComm (communicator)
- ProfilingManagerPub
- ExternalInput (环境变量)
- WorkflowMode


---

## 2.2.3 next/comms — 新框架通信端点抽象 (endpoints + endpoint_pairs + channels)

### Module Overview

`framework/next/comms/` 是hcomm新框架(V2/next)的通信连接管理核心模块，207个C++文件，
负责通信端点(Endpoint)、端点对(EndpointPair)、通道(Channel)的创建、连接建立和资源管理。

与V1（legacy/communicator、framework/communicator）的核心区别:
- V1: Transport = 一个完整的点对点连接（conn + buffer + notify打包）
- V2(next): Endpoint = 通信设备抽象, EndpointPair = 两个Endpoint的配对, Channel = EndpointPair上特定引擎的通道

这个分层让新框架可以在同一对Endpoint之间创建多种引擎的Channel(Host/AICPU/AIV/CCU)。

### 子模块结构

```
next/comms/
  endpoints/           # Endpoint基类 + 3种实现 + 注册内存管理
    reged_mems/         # 3种注册内存管理器(RoCE/UB/URMA)
    server_socket/      # ServerSocket全局管理
  endpoint_pairs/       # EndpointPair + Manager
    channels/           # Channel基类 + 5种通道实现
      host/             # HostCpuRoceChannel (完整RDMA实现)
      aicpu/            # AicpuTsUrmaChannel (AICPU+URMA)
      aiv/              # AivUbMemChannel (AIV+UB共享内存)
      ccu/              # CcuUrmaChannel (CCU+URMA)
      slaves/           # Device侧kernel桩
    sockets/            # SocketMgr
  api_c_adpt/           # C API适配层 (全局状态 + 句柄管理)
  adpt/                 # HCCP/RTS适配器
  common/               # 工具函数(类型转换、TP管理、EID管理)
  comm_engine_res/      # 通信引擎资源管理(线程、Notify、上下文)
  ccu/                  # CCU子系统(微码、表示层、Transport、设备管理)
```

### Key Files Table

| 文件 | 行数 | 职责 |
|------|------|------|
| endpoints/endpoint.h | 78 | Endpoint抽象基类，定义Init/RegisterMemory/MemoryExport等虚接口 |
| endpoints/endpoint.cc | 40 | 工厂方法CreateEndpoint，按protocol+locType选择具体Endpoint类 |
| endpoints/endpoint_mgr.h/cc | 54+107 | EndpointMgr: EndpointDesc→EndpointHandle映射+注册内存管理 |
| endpoints/cpu_roce_endpoint.h/cc | 44+133 | Host CPU + RoCE协议端点，通过RdmaHandleManager获取RDMA上下文 |
| endpoints/urma_endpoint.h/cc | 53+149 | Device + URMA协议端点，含CcuChannelCtxPool |
| endpoints/ub_mem_endpoint.h/cc | 39+72 | Device + UB共享内存端点，MemoryExport/Import不支持 |
| endpoint_pairs/endpoint_pair.h | 90 | EndpointPair类 + FNV-1a hash + std::hash特化 |
| endpoint_pairs/endpoint_pair.cc | 47 | Init/GetSocket/CreateChannel委托 |
| endpoint_pairs/endpoint_pair_mgr.h/cc | 36+37 | EndpointPairMgr: EndpointDescPair→EndpointPair去重缓存 |
| endpoint_pairs/channels/channel.h | 67 | Channel抽象基类，禁拷贝允许移动，定义控制面/数据面接口分区 |
| endpoint_pairs/channels/channel.cc | 69 | 工厂方法按CommEngine分派到5种Channel实现 |
| endpoint_pairs/channels/host/host_cpu_roce_channel.h/cc | 109+739 | 最完整的Channel实现，含RDMA QP生命周期+数据面verbs调用 |
| endpoint_pairs/channels/host/host_rdma_connection.h/cc | 81+210 | RDMA连接管理，QP创建/修改/交换/销毁 |
| endpoint_pairs/channels/aicpu/aicpu_ts_urma_channel.h/cc | 69+251 | AICPU通道，BuildConnection/BuildNotify/BuildBuffer+H2D资源打包 |
| endpoint_pairs/channels/aiv/aiv_ub_mem_channel.h/cc | 46+87 | AIV UB共享内存通道，通过AivUbMemTransport管理 |
| endpoint_pairs/channels/ccu/ccu_urma_channel.h/cc | 60+275 | CCU通道，CcuChannelCtxPool分配CCU资源+CcuTransport建链 |
| endpoint_pairs/sockets/socket_mgr.h/cc | 47+164 | Socket创建/复用管理，区分DEV_NET/HOST_NET |
| api_c_adpt/hcomm_c_adpt.h | 63 | C API声明(18个函数)，extern "C" |
| api_c_adpt/hcomm_c_adpt.cc | 605 | C API实现，4个全局map + WithChannelByHandleLocked模板 |
| api_c_adpt/endpoint_map.h/cc | 42+73 | HcommEndpointMap: EndpointHandle→unique_ptr<Endpoint>，mutex保护 |
| common/orion_adpt_utils.h/cc | 30+144 | Orion类型系统到next类型系统的适配(CommAddr↔IpAddress等) |
| adpt/hcomm_adapter_hccp.h | 165 | HCCP URMA适配层定义(Jetty/JFC/TP概念) |

### Architecture: Three-Layer Abstraction

```
Endpoint (通信设备)
  |-- owns --> RegedMemMgr (注册内存)
  |-- owns --> ServerSocket (监听)
  |-- via RdmaHandleManager --> ctxHandle_ (RDMA/URMA上下文)
  
EndpointPair (连接)
  |-- refs --> local Endpoint + remote EndpointDesc
  |-- owns --> SocketMgr (数据通道socket)
  |-- creates --> Channel(s) (可多个不同引擎)

Channel (通道)
  |-- refs --> EndpointHandle (本地端点句柄)
  |-- owns --> Connection(s) (QP/Jetty)
  |-- owns --> LocalRmaBuffer(s) (本地注册内存)
  |-- owns --> Notify(s) (同步信号)
  |-- exchanges --> RemoteRmaBuffer(s), RemoteNotify(s)
```

### Architecture: Factory Pattern × 2

两层工厂:
1. Endpoint::CreateEndpoint (endpoint.cc L22-38):
   - COMM_PROTOCOL_ROCE + HOST → CpuRoceEndpoint
   - COMM_PROTOCOL_UBC_TP/CTP + DEVICE → UrmaEndpoint
   - COMM_PROTOCOL_UB_MEM + DEVICE → UbMemEndpoint

2. Channel::CreateChannel (channel.cc L22-63):
   - COMM_ENGINE_CPU + ROCE → HostCpuRoceChannel
   - COMM_ENGINE_AICPU/AICPU_TS → AicpuTsUrmaChannel
   - COMM_ENGINE_AIV → AivUbMemChannel
   - COMM_ENGINE_CCU → CcuUrmaChannel

值得注意: Endpoint按protocol×location分派，Channel按engine×protocol分派，形成矩阵:

| Engine\Protocol | RoCE | UBC_TP/CTP | UB_MEM | HCCS |
|-----------------|------|------------|--------|------|
| CPU (Host)      | HostCpuRoceChannel | - | - | - |
| AICPU_TS        | (头文件空壳) | AicpuTsUrmaChannel | - | (头文件空壳) |
| AIV             | - | - | AivUbMemChannel | (头文件空壳) |
| CCU             | - | CcuUrmaChannel | - | - |

4个空壳头文件(CpuRoceChannel/AicpuTsHccsChannel/AicpuTsRoceChannel/AivHccsChannel)只有类声明无实现，
说明对应组合尚未实现。

### Architecture: Handle-Based C API Layer

全局状态(hcomm_c_adpt.cc):
- g_EndpointMap: EndpointHandle → unique_ptr<Endpoint> (mutex保护)
- g_ChannelMap: ChannelHandle → unique_ptr<Channel> (g_ChannelMapMtx保护)
- g_ChannelD2HMap: ChannelHandle → ChannelHandle (Device→Host句柄映射)
- g_ThreadMap: ThreadHandle → shared_ptr<Thread>

核心设计: 句柄 = reinterpret_cast<Handle>(ptr.get())
- 创建时: unique_ptr持有对象，指针值作为句柄
- 查找时: 通过句柄在map中查找，返回裸指针
- 销毁时: 从map中erase，unique_ptr析构释放资源

WithChannelByHandleLocked模板(L46-79):
- 单锁同时保护g_ChannelD2HMap和g_ChannelMap
- D2H映射: device上的channel handle通过kernel launch获得，需映射回host上的channel handle
- lambda在锁内执行: 防止并发destroy释放对象

### Architecture: RDMA Connection State Machine

HostCpuRoceChannel实现了完整的RDMA连接状态机(host_cpu_roce_channel.cc):

```
INIT → [CheckSocketStatus] → SOCKET_OK → [CreateQp] → QP_CREATED
    → [ExchangeData] → DATA_EXCHANGE → [ModifyQp] → QP_MODIFIED → CONN_OK
```

ExchangeData流程(L187-222):
1. Pack: NotifyVec + BufferVec + ConnVec → BinaryStream → vector<char>
2. Send: socket_->Send(data)
3. Recv: socket_->Recv(data) (阻塞等待对端)
4. Unpack: BinaryStream → NotifyVec + RmtBufferVec + ConnVec

数据面接口直接调用ibv verbs:
- WriteWithNotify: ibv_post_send(IBV_WR_RDMA_WRITE_WITH_IMM)
- NotifyWait: ibv_poll_cq轮询(带超时)
- ChannelFence: ibv_poll_cq等待所有WC

### Architecture: H2D Resource Serialization

AicpuTsUrmaChannel.H2DResPack (aicpu_ts_urma_channel.cc L242-248):
将Host侧构建的channel资源序列化为二进制，通过H2D memcpy传给Device侧AICPU kernel。

HcommChannelKernelLaunch (hcomm_c_adpt.cc L272-366):
1. 对每个channel调用H2DResPack → hostPackBuffers
2. CombineHostMemory: 多段序列化数据合并为连续HostMem
3. hrtMemSyncCopy: H2D
4. AicpuAclKernelLaunch: 下发AICPU kernel初始化通道
5. hrtMemSyncCopy: D2H取回device侧channel handle
6. FillChannelD2HMap: 建立device→host句柄映射

### Idioms & Conventions

1. FNV-1a Hash for struct (endpoint_pair.h L29-50):
   对EndpointDesc做字节级FNV-1a hash，同时适配32/64位:
   - sizeof(size_t)==8 → 64-bit FNV offset/prime
   - sizeof(size_t)==4 → 32-bit FNV offset/prime
   EndpointDescPair hash用boost式组合: h ^= h2 + 0x9e3779b97f4a7c15 + (h<<6) + (h>>2)

2. memcmp做==比较 (endpoint_pair.h L25-27):
   inline bool operator==(const EndpointDesc& a, const EndpointDesc& b) noexcept {
       return std::memcmp(&a, &b, sizeof(EndpointDesc)) == 0;
   }
   前提: EndpointDesc是POD类型，无padding差异。

3. EXECEPTION_CATCH + make_unique/make_shared (48处/18文件):
   所有涉及STL分配的地方包裹EXECEPTION_CATCH宏防止内存分配异常。

4. MAKE_ENUM (18处):
   next/comms大量使用MAKE_ENUM宏生成枚举类型，自带Describe()方法:
   MAKE_ENUM(ChannelStatus, INIT, SOCKET_OK, SOCKET_TIMEOUT, READY, FAILED)

5. Channel::CreateChannel后立即Init (channel.cc L61):
   CHK_RET(channelPtr->Init()); // 工厂创建后立即初始化
   每个Channel子类的Init遵循固定子步骤: ParseInputParam→BuildXxx→...

6. EXCEPTION_HANDLE_BEGIN/END (42处/11文件):
   不同于EXECEPTION_CATCH(单行)，EXCEPTION_HANDLE用于多行代码块的异常保护。
   主要出现在CCU子系统和SocketMgr中。

7. 注册内存引用计数 (RoceRegedMemMgr/RmaBufferMgr):
   - 重复RegisterMemory返回HCCL_E_AGAIN(非错误)
   - UnregisterMemory引用计数>1时返回HCCL_E_AGAIN
   - 重叠内存返回HCCL_E_INTERNAL(真错误)

### Anti-Patterns

AP-1: GetStatus内fall-through (host_cpu_roce_channel.cc L138):
```cpp
case RdmaStatus::DATA_EXCHANGE:
    CHK_RET(ModifyQp());
    rdmaStatus_ = RdmaStatus::QP_MODIFIED;
case RdmaStatus::QP_MODIFIED:   // <-- 无break，fall-through
    // TODO: Prepare Rqe
default:
    rdmaStatus_ = RdmaStatus::CONN_OK;
```
DATA_EXCHANGE执行完ModifyQp后直接fall-through到QP_MODIFIED再到default，
跳过了可能需要的Prepare Rqe步骤。

AP-2: 变量遮蔽bug (orion_adpt_utils.cc L27-30):
```cpp
int32_t family = AF_INET6;      // L27: 外层声明
if (commAddr.type == COMM_ADDR_TYPE_IP_V4) {
    binAddr.addr = commAddr.addr;
    int32_t family = AF_INET;   // L30: 内层重新声明，遮蔽外层
    ipAddr = Hccl::IpAddress(binAddr, family);
```
虽然当前行为恰好正确(IPv4分支内用的是局部AF_INET)，但内层重新声明是bug写法。

AP-3: 全局static map无清理机制 (共6个):
- g_EndpointMap, g_EndpointMapMutex (endpoint_map.cc L18-19)
- g_ChannelMap, g_ChannelD2HMap, g_ThreadMap, g_ChannelMapMtx (hcomm_c_adpt.cc L31-35)
- ccuDrvHandleMap, ccuDrvHandleMutex (ccu_dev_mgr_imp.cc L25-26)
进程级全局状态，析构顺序不可控。EndpointMgr析构时会调用HcommEndpointDestroy/HcommMemUnreg，
但g_EndpointMap是static，析构时已可能被销毁。

AP-4: g_ThreadMap无mutex保护:
g_ChannelMap/g_ChannelD2HMap有g_ChannelMapMtx保护，
但g_ThreadMap的HcommThreadAlloc/HcommThreadFree操作无锁保护。

AP-5: ServerSocket全局static + 无mutex (cpu_roce_endpoint.cc L91-96, urma_endpoint.cc L102-106):
```cpp
static std::unordered_map<...> serverSocketMap;  // 函数内静态
```
GetServerSocketMap返回引用给调用者使用，无锁保护并发访问。

AP-6: 混用new(nothrow)和make_unique (channel.cc):
```cpp
case COMM_ENGINE_CPU:
    EXECEPTION_CATCH(channelPtr = std::make_unique<...>(), return HCCL_E_PARA);  // make_unique
case COMM_ENGINE_AICPU_TS:
    channelPtr.reset(new (std::nothrow) AicpuTsUrmaChannel(...));  // new(nothrow)
case COMM_ENGINE_AIV:
    channelPtr.reset(new (std::nothrow) AivUbMemChannel(...));     // new(nothrow)
```
同一个工厂函数内两种分配风格混用，EXECEPTION_CATCH+make_unique和new(nothrow)的错误处理路径不一致。

AP-7: reinterpret_cast<Endpoint*>(endpointHandle) (109处/20文件):
EndpointHandle/ChannelHandle本质是void*，通过reinterpret_cast还原:
```cpp
Endpoint* localEpPtr = reinterpret_cast<Endpoint*>(endpointHandle_);  // 无类型安全检查
```
如果传入错误的handle值将导致未定义行为。HostCpuRoceChannel和AicpuTsUrmaChannel都直接做此转换。

AP-8: 13个TODO表明模块不成熟:
- "TODO: 通过引擎 + 协议" (channel.cc L27)
- "TODO: 处理抛异常" (aicpu_ts_urma_channel.cc L163)
- "TODO: 成员变量全部初始化" (aicpu_ts_urma_channel.h L51)
- "TODO: to be deleted" (aicpu_ts_urma_channel.cc L120)
- "TODO: Prepare Rqe" (host_cpu_roce_channel.cc L139)
等，说明模块处于早期开发状态，部分功能路径尚未完善。

AP-9: NotifyWait忙轮询无退避 (host_cpu_roce_channel.cc L574-594):
```cpp
while (true) {
    auto actualNum = ibv_poll_cq(qpInfo[0].recvCq, 1, &wc);
    // ... 无sleep/yield/backoff
    if ((std::chrono::steady_clock::now() - startTime) >= waitTime) {
        return HCCL_E_TIMEOUT;
    }
}
```
纯忙等待消耗CPU，应加入指数退避或event通知机制。

AP-10: 硬编码端口 (socket_mgr.cc L20):
```cpp
constexpr uint32_t TempServerListenPort = 60001;    // 临时固定监听端口，用于功能验证
```
端口号硬编码，注释明确标注"临时"和"功能验证"。

### Domain Knowledge

1. 通信协议矩阵:
   - COMM_PROTOCOL_ROCE: 传统RDMA over Converged Ethernet (Host CPU发起)
   - COMM_PROTOCOL_UBC_TP: URMA基础通信-TP模式 (Device发起)
   - COMM_PROTOCOL_UBC_CTP: URMA基础通信-CTP模式 (Device发起，可能更高效)
   - COMM_PROTOCOL_UB_MEM: UB共享内存模式 (Device内/Device间共享)
   - COMM_PROTOCOL_HCCS: 高速芯片互联 (片间互联，未实现)

2. 通信引擎矩阵:
   - COMM_ENGINE_CPU: Host CPU直接发起RDMA操作
   - COMM_ENGINE_AICPU_TS: AICPU通过TaskScheduler发起
   - COMM_ENGINE_AIV: AI Vector处理器发起
   - COMM_ENGINE_CCU: Communication Control Unit发起(硬件卸载)

3. Channel D2H映射机制:
   Host侧创建Channel → 序列化资源 → H2D拷贝 → AICPU kernel初始化 → Device侧生成handle
   → D2H取回device handle → g_ChannelD2HMap建立 device→host 映射
   查询/操作Channel时先通过D2H map找到host上的真实Channel对象。

4. Jetty概念 (hcomm_adapter_hccp.h):
   URMA协议的通信端点，类似RDMA的QP。有6种模式:
   STANDARD / HOST_OFFLOAD / HOST_OPBASE / DEV_USED / CACHE_LOCK_DWQE / CCU_CCUM_CACHE
   不同模式对应不同的SQ资源分配和发送方式。

5. TP(Transport Path)管理 (tp_mgr.h):
   TpMgr维护per-device的TP信息缓存(CTP和RTP两种)，引用计数管理:
   - 首次GetTpInfo: 异步请求HCCP → 缓存结果
   - 再次GetTpInfo: 直接从缓存返回+引用计数+1
   - ReleaseTpInfo: 引用计数-1

### Hardware Abstractions

1. ServerSocket区分NicType:
   - CpuRoceEndpoint → NicType::HOST_NIC_TYPE (Host网卡)
   - UrmaEndpoint → NicType::DEVICE_NIC_TYPE (设备网卡)
   SocketMgr也区分DEV_NET(SocketHandleManager)和HOST_NET(HostSocketHandleManager)。

2. 910_95设备限制 (host_rdma_connection.cc L38-45):
   HostRdmaConnection::Init仅支持DEV_TYPE_910_95:
   ```cpp
   if (devType == DevType::DEV_TYPE_910_95) {
       qpMode = Hccl::OPBASE_QP_MODE;
   } else {
       return HCCL_E_NOT_SUPPORT;
   }
   ```
   说明Host CPU RoCE通道是910_95专用功能。

3. CCU资源固定为16 (ccu_urma_channel.cc L143):
   "当前建链不支持资源扩容，CCU资源默认固定为16"
   每个CCU Channel固定使用16个CKE(Notify)资源。

4. DPU Notify (host/dpu_notify/dpu_notify_manager):
   Host CPU RoCE Channel使用DPU Notify ID进行信号通知，
   通过ibv_send WR的imm_data字段传递notify ID。

### Notable Patterns

1. 新旧共存: next/comms大量引用legacy代码:
   ```cpp
   #include "../../../../../../legacy/unified_platform/resource/socket/socket.h"
   #include "../../../../../../legacy/unified_platform/resource/buffer/local_rdma_rma_buffer.h"
   ```
   6层../相对路径，Channel实现重度依赖legacy的Socket、RmaBuffer、Connection等基础设施。

2. 空壳Channel保留扩展点:
   AicpuTsHccsChannel、AicpuTsRoceChannel、AivHccsChannel、CpuRoceChannel
   4个头文件只有类声明，无虚函数override、无实现文件。
   为未来protocol×engine组合预留了扩展位置。

3. 双层Manager去重:
   - EndpointMgr: EndpointDesc → EndpointHandle (同一设备只创建一个Endpoint)
   - EndpointPairMgr: EndpointDescPair → EndpointPair (同一对设备只创建一个pair)
   - SocketMgr: SocketConfig → Socket (同一配置只创建一个socket)
   三层都实现了"按需创建，复用已有"的模式。

4. BinaryStream序列化协议:
   ExchangeData使用Hccl::BinaryStream做序列化，格式:
   [notifyNum][notify_id_0]...[notify_id_n] + [bufferNum][pos][dto]... + [connNum][pos][dto]...
   收发双方必须按相同顺序pack/unpack，无版本字段或自描述能力。

5. WithChannelByHandleLocked函数模板:
   统一的"查找+持锁执行"模式，避免每个API函数都重复D2H映射+查Channel+加锁的样板代码。
   设计精巧但注释提醒"func内不要做长耗时/阻塞操作"。

---

## 2.2.4-supplement: next/ 子模块补充分析

本节覆盖 2.2.3 中未完全分析的 next/ 子模块:
- comm_engine_res/ (执行引擎资源管理)
- endpoint_pairs/channels/slaves/ (Device侧kernel实现)
- comm_primitives/ (通信原语C适配层)
- common/enum_factory.h (枚举工厂宏)

### Part 1: comm_engine_res/ — 执行引擎资源管理

路径: next/comms/comm_engine_res/
文件数: 23 (含 CMakeLists.txt)
子目录: threads/, engine_ctxs/, notifys/, ccu/

#### 模块职责

comm_engine_res 管理不同通信引擎(CPU_TS/AICPU_TS/AIV/CCU)下的执行资源:
- Thread: 通信引擎的并行执行单元，封装 Stream + LocalNotify
- EngineCtx: 引擎内存上下文(引擎可访问的内存区域)
- Notify: 跨 kernel/stream 的同步信号
- CcuResPack: CCU 引擎专有资源包

层次结构:
```
CommEngineResMgr (管理所有引擎资源)
  └── CommEngineRes (单个引擎的资源)
        ├── threads (并行执行单元)
        ├── engineCtxs (内存上下文)
        └── (notifys 通过 thread 管理)
```

#### 核心类分析

##### Thread 抽象基类 (thread.h, hccl namespace)

注意: 尽管位于 next/ 目录下，Thread 类属于 `hccl` namespace 而非 `hcomm` namespace。
这是因为 Thread 需要与 legacy 层的类型(如 Stream, LocalNotify, HcclResult)直接交互。

Thread 是所有引擎执行单元的抽象基类，定义了两类接口:

1. 资源访问接口:
   - Init()/DeInit(): 生命周期管理
   - GetUniqueId(): 序列化为二进制字符串(用于 H2D 传输)
   - GetNotifyNum()/GetNotify(index): 同步信号访问
   - GetStream()/GetStreamLitePtr(): 流访问(A3/A5 两代硬件)
   - IsDeviceA5(): 硬件代际判断

2. Local Data Plane 接口(在 kernel 内使用):
   - LocalNotifyRecord()/LocalNotifyWait(): 线程间同步
   - LocalCopy(): SDMA 数据复制
   - LocalReduce(): SDMA 规约操作

关键设计 — ThreadHandle 映射:
```cpp
// thread.h:57
std::unordered_map<CommEngine, ThreadHandle> threadHandleMap_;
// CPU_TS上的ThreadHandle与其他引擎上的ThreadHandle的映射
```
同一个物理 Thread 可以在多个引擎上有不同的 Handle，通过 AddThreadHandleToMap/FindThreadByCommEngine 管理。
ThreadHandle 实际是 `uint64_t`，通过 `reinterpret_cast<Thread*>` 还原为指针。

全局工具函数:
```cpp
// thread.h:60-78
inline Stream *GetStream(uint64_t thread)   // 从 ThreadHandle 获取 Stream
inline LocalNotify *GetNotify(uint64_t thread, uint32_t index)  // 从 ThreadHandle 获取 Notify
```
这两个函数用 UNLIKELY 保护 nullptr 检查，是热路径上的快捷访问。

##### CreateThread 工厂函数 (thread.cc:16-31)

```cpp
HcclResult CreateThread(CommEngine engine, StreamType streamType,
    uint32_t notifyNum, NotifyLoadType loadType, std::shared_ptr<Thread>& out_thread)
{
    out_thread = nullptr;  // 初始化出参
    if (engine == COMM_ENGINE_CPU_TS || engine == COMM_ENGINE_CPU || engine == COMM_ENGINE_CCU) {
        EXECEPTION_CATCH(out_thread = std::make_shared<CpuTsThread>(...), return HCCL_E_PTR);
    } else if (engine == COMM_ENGINE_AICPU_TS || engine == COMM_ENGINE_AICPU) {
        EXECEPTION_CATCH(out_thread = std::make_shared<AicpuTsThread>(...), return HCCL_E_PTR);
    } else {
        return HCCL_E_NOT_SUPPORT;
    }
    return HCCL_SUCCESS;
}
```

惯用法:
- 初始化出参为 nullptr
- EXECEPTION_CATCH 包裹内存分配
- CommEngine 类型决定具体实现类

三组映射函数:
- CommEngineToNotifyLoadType: CPU系→HOST_NOTIFY, AICPU系→DEVICE_NOTIFY
- CommHostEngineToNotifyLoadType: 仅接受 CPU 系引擎
- CommEngineToStreamType: CPU系→STREAM_TYPE_ONLINE, AICPU系→STREAM_TYPE_DEVICE, AIV→不支持

注意注释: `// 暂不支持AIV` (thread.cc:79) — AIV 引擎暂未集成到 Thread 体系。

##### AicpuTsThread (aicpu_ts_thread.h/.cc)

最完整的 Thread 实现，330+ 行。核心特征:

1. Host/Device 双侧初始化:
```cpp
HcclResult Init() {
    CHK_RET(GetRunSideIsDevice(isDeviceSide_));
    if (!isDeviceSide_) {
        return HostInit();    // Host侧: 申请资源
    } else {
        return DeviceInit();  // Device侧: 反序列化恢复资源
    }
}
```
这是 H2D 资源传递模式的核心: Host 侧创建资源 → GetUniqueId() 序列化 → H2D memcpy → Device 侧 DeviceInit() 反序列化。

2. GetUniqueId() 序列化协议:
```
[streamType_][notifyLoadType_][devId_][notifyNum_]
[HcclStreamParam{streamIds, sqIds, cqIds, logicCqIds, sqCqContextAddr, sqCqContextSize}]
[HcclSignalInfo_0][HcclSignalInfo_1]...[HcclSignalInfo_n]
```
使用 ostringstream + write(reinterpret_cast<const char_t*>(&field), sizeof(field)) 做二进制序列化。
协议无版本号，收发双方必须严格匹配字段顺序。

3. A3 vs A5 硬件代际适配:
```cpp
bool IsDeviceA5() const { return devType_ == DevType::DEV_TYPE_910_95; }
```
- A3 (非910_95): 使用传统 Stream + InitStream + SqCqeContext
- A5 (910_95): 使用 StreamLite + InitStreamLite + IAicpuTsThread pImpl

DeviceInit() 中 (aicpu_ts_thread.cc:322-328):
```cpp
if (devType_ == DevType::DEV_TYPE_910_95) {
    CHK_RET(InitStreamLite(streamParam.streamInfo, hostPhyId));
} else {
    CHK_RET(InitStream(streamParam));
}
```

4. Local Data Plane 委托给 IAicpuTsThread (pImpl):
```cpp
HcclResult LocalCopy(void *dst, const void *src, uint64_t sizeByte) const {
    CHK_PTR_NULL(pImpl_);
    uint64_t dstAddr = reinterpret_cast<uint64_t>(dst);
    uint64_t srcAddr = reinterpret_cast<uint64_t>(src);
    return pImpl_->SdmaCopy(dstAddr, srcAddr, sizeByte);
}
```
所有 Local Data Plane 操作都通过 pImpl_ (IAicpuTsThread) 委托，
IAicpuTsThread 封装了 StreamLite API (rtsq 系列接口)。

5. InitStream (A3 路径, #ifdef CCL_KERNEL_AICPU):
- hrtHalResourceIdRestore 恢复 stream 资源
- custom 进程 vs aicpu 进程差异处理
- BuildComStreamInfo: 查询 SQ base addr、SQ depth
- InitSqAndCqeContext: 读取 sqHead/sqTail，初始化 SQ/CQ 上下文

6. HostInit (aicpu_ts_thread.cc:271-303):
```cpp
HcclResult AicpuTsThread::HostInit() {
    CHK_PRT_RET(!uniqueIdStr_.empty(), ...);  // Host侧不接受uniqueId
    // 获取设备物理ID
    CHK_RET(hrtGetDevice(&deviceLogicId));
    CHK_RET(hrtGetDevicePhyIdByIndex(..., devId_));
    CHK_RET(hrtGetDeviceType(devType_));
    // 创建Stream
    stream_.reset(new (std::nothrow) Stream(streamType_));
    // 创建Notify数组
    for (idx = 0; idx < notifyNum_; idx++) {
        notifys_[idx].reset(new (std::nothrow) LocalNotify());
        CHK_RET(notifys_[idx]->Init(notifyLoadType_));
        if (devType_ != DevType::DEV_TYPE_910_95) {
            CHK_RET(notifys_[idx]->SetIpc());  // 非A5需要IPC
        }
    }
    // A3: 申请SqCqeContext设备内存
    if (streamType_ == StreamType::STREAM_TYPE_DEVICE && devType_ != DevType::DEV_TYPE_910_95) {
        sqCqeContext_ = DeviceMem::alloc(sizeof(SqCqeContext));
    }
}
```

##### CpuTsThread (cpu_ts_thread.h/.cc)

与 AicpuTsThread 结构相似但功能简化:
- Host 侧初始化完整，Device 侧返回 NOT_SUPPORT (CpuTsThread 只在 Host 上运行)
- 支持从已有 rtStream_t 构造(复用外部 stream)
- Local Data Plane 全部返回 HCCL_E_NOT_SUPPORT (CPU_TS 不支持 SDMA 操作)
- GetStreamLitePtr() 返回 nullptr (不支持 A5 StreamLite)
- LaunchTask() 为空操作

GetUniqueId() 的特殊处理 (cpu_ts_thread.cc:73-135):
CpuTsThread 在 Host 上运行，但需要将其 Thread 信息传递到 Device 端让 AicpuTsThread 使用。
因此 GetUniqueId() 额外申请了一条 DEVICE 类型的 stream (streamDevice_) 和 sqCqeContext_。

注意: GetUniqueId() 中的日志标签写的是 "[AicpuTsThread]" 而非 "[CpuTsThread]" (cpu_ts_thread.cc:121-131)。
这是 copy-paste 遗留的错误标签。

##### AivThread (aiv_thread.h/.cc)

最简实现，只是骨架:
```cpp
class AivThread : public Thread {
public:
    uint32_t GetNotifyNum() const override;
private:
    uint32_t notifyNum_{};
};
```
仅实现了 GetNotifyNum()，其余 Thread 基类的 11 个 virtual 方法均未 override。
对应 thread.cc 中的注释 `// 暂不支持AIV`。

##### CommEngineRes / CommEngineResMgr (hcomm namespace)

上层资源管理器，属于 `hcomm` namespace:

CommEngineRes: 管理单个引擎类型下的所有资源
```cpp
class CommEngineRes {
    CommEngineType engineType_{};
    std::vector<std::shared_ptr<Thread>> threads_{};
    std::vector<std::unique_ptr<EngineCtx>> engineCtxs_{};
};
```

CommEngineResMgr: 管理所有引擎类型的资源
```cpp
class CommEngineResMgr {
    std::unordered_map<CommEngineType, std::shared_ptr<CommEngineRes>> engineResources_{};
    std::mutex mutex_{};
};
```

注意: .cc 文件均为空实现 (仅包含空 namespace 体)。
CommEngineRes.h 声明了 AllocateThreads/ReleaseThreads/AcquireEngineCtx/ReleaseEngineCtx，
但 comm_engine_res.cc 无任何方法实现。这说明资源管理器目前由调用方直接操作 Thread/EngineCtx，
CommEngineRes 作为组织容器存在但尚未封装逻辑。

##### EngineCtxMgr (engine_ctxs/)

引擎内存上下文管理器:
```cpp
class EngineCtxMgr {
    std::unordered_map<std::string, void *> engineCtxs_{};  // 注意: .h 声明为 void*
    std::mutex mutex_{};
};
```

实现特点 (engine_ctx_mgr.cc):
1. GenerateCtxKey: 用 `to_string(engineType) + "_" + to_string(opTag)` 生成字符串 key
2. AcquireEngineCtx: try-catch 包裹，支持 HcclException 和通用异常
3. ReleaseEngineCtx: 线性遍历所有 entry 查找匹配指针
4. FindEngineCtx: 按 key 查找

注意 .h 和 .cc 的不一致:
- .h 声明 `std::unordered_map<std::string, void*>`
- .cc 调用 `it->second.get()` 和 `std::make_unique<EngineCtx>` 和 `std::move(newCtx)`
  这些操作需要 `unique_ptr<EngineCtx>` 类型，但 .h 里是 `void*`
  推测 .h 的类型声明已过时或简化过，实际编译应使用不同的头文件版本。
  此处存在头文件声明与实现不匹配的问题。

异常处理风格:
```cpp
try { ... }
catch (const HcclException& e) { return e.GetResult(); }
catch (...) { return HcclResult::HCCL_ERROR_COMMUNICATION; }
```
这是 comm_engine_res 模块唯一使用 try-catch 的地方，与其他模块普遍使用 CHK_RET 不同。
另外错误码使用 `HcclResult::HCCL_ERROR_INVALID_PARAM` 形式(全限定枚举)，
而非其他模块的 `HCCL_E_PARA` 短名称，两种风格在同一模块中混用。

##### Notify 体系 (notifys/)

抽象层极简:
```cpp
class Notify { public: virtual ~Notify() = default; };
```
只有虚析构函数，无任何方法。

AicpuTsRtsNotify (aicpu_ts_rts_notify.h):
```cpp
class AicpuTsRtsNotify : public Notify {
    uint32_t GetNotifyNum() const override;  // 注意: Notify基类没有声明这个方法
    std::vector<Notify> notifys_{};
    uint32_t notifyNum_{};
};
```
问题: GetNotifyNum() 声明为 override，但基类 Notify 没有这个虚方法。
这会导致编译错误，说明此文件可能未被实际编译或使用不同的基类版本。

Kernel-side 定义 (slaves/aicpu_ts_rts_notify_kernel.h):
仅有注释，空 namespace 体。与 aicpu_ts_thread_kernel.h 一样是预留占位。

##### CcuResContainer (ccu/)

CCU 引擎专有资源管理:
```cpp
class CcuResContainer {
    uint32_t opExpansionMode_{0};         // 运行模式
    int32_t devLogicId_{INT32_MAX};       // 设备逻辑ID
    std::shared_ptr<CcuDrvHandle> ccuDrvHandle_{};  // CCU驱动句柄(共享)
    std::unique_ptr<CcuResPack> resPack_{};          // CCU资源包(独占)
    std::vector<CcuKernelHandle> kernelHandles_{};   // 所有注册的kernel
    std::vector<CcuKernelHandle> untranslatedKernelHandles_{};  // 未翻译的kernel
};
```

关键逻辑:

1. OpExpansionMode → CcuEngine 映射 (ccu_res_container.cc:27-39):
```cpp
inline CcuEngine OpExpansionModeToCcuEngine(const uint32_t opExpansionMode) {
    if (mode == DEFAULT_MODE || mode == CCU_SCHE_MODE) return CcuEngine::CCU_SCHE;
    if (mode == CCU_MS_MODE)  return CcuEngine::CCU_MS;
    return CcuEngine::INVALID;
}
```
三种模式: DEFAULT(0)→调度引擎, CCU_MS(5)→微服务引擎, CCU_SCHE(6)→调度引擎

2. 析构顺序严格 (ccu_res_container.cc:41-58):
```cpp
~CcuResContainer() {
    // 主动释放资源保证时序，不得随意调整顺序
    for (auto &kernelHandle : kernelHandles_) {
        (void)CcuKernelMgr::GetInstance(devLogicId_).UnRegister(kernelHandle);
        kernelHandle = 0;
    }
    kernelHandles_.clear();
    resPack_ = nullptr;          // 先释放资源包
    if (ccuDrvHandle_) {
        ccuDrvHandle_ = nullptr; // 再释放驱动句柄
        (void)CcuDeinitFeature(devLogicId_);  // 最后尝试关闭CCU功能
    }
}
```
注释强调 "不得随意调整顺序" — 典型的资源释放时序敏感模式。
(void) 忽略 UnRegister 和 CcuDeinitFeature 的返回值 — 析构中不传播错误。

3. EXCEPTION_HANDLE_BEGIN/END 宏:
```cpp
HcclResult SaveCcuKernel(const CcuKernelHandle kernelHandle) {
    EXCEPTION_HANDLE_BEGIN           // = try {
    kernelHandles_.push_back(kernelHandle);
    untranslatedKernelHandles_.push_back(kernelHandle);
    EXCEPTION_HANDLE_END             // = } catch (...) { ... }
    return HcclResult::HCCL_SUCCESS;
}
```
保护 vector::push_back 的 bad_alloc 异常。

4. Init() 惰性初始化:
```cpp
HcclResult Init() {
    if (ccuEngine == CcuEngine::INVALID) return HCCL_SUCCESS;  // 无效模式静默成功
    if (!ccuDrvHandle_) CHK_RET(CcuInitFeature(devLogicId_, ccuDrvHandle_));
    if (!resPack_) { resPack_.reset(new (nothrow) CcuResPack(ccuEngine)); CHK_RET(resPack_->Init()); }
}
```
check-then-create 的惰性模式。

5. untranslatedKernelHandles_ 双列表设计:
   - kernelHandles_: 所有曾注册的 kernel (析构时全部 UnRegister)
   - untranslatedKernelHandles_: 尚未翻译的 kernel (ResetResPack 时清空)
   这支持增量编译: 每次 Reset 后只翻译新增的 kernel。

#### Thread 相关的常量和限制

```cpp
constexpr u32 HCOMM_NOTIFY_MAX_NUM = 64;       // 每个Thread最多64个Notify
constexpr u32 HCOMM_THREADNUM_MAX_NUM = 1000;   // 每个引擎最多1000个Thread
```

### Part 2: slaves/ — Device侧 Kernel 实现

路径: next/comms/endpoint_pairs/channels/slaves/
文件数: 4 (含 CMakeLists.txt)

#### 模块职责

slaves/ 目录存放运行在 Device 端(AICPU kernel)的入口函数，
是 Host → Device 跨空间调用的 Device 侧落地点。

#### AicpuTsUrmaChannelKernel (唯一有实质实现)

头文件 (aicpu_ts_urma_channel_kernel.h):
```cpp
extern "C" {
__attribute__((visibility("default"))) uint32_t RunAicpuIndOpChannelInitV2(void *args);
}
```

注意 header guard 有 typo: `AICPU_TS_URMA_CAHNNEL_KERNEL_H` (CAHNNEL 应为 CHANNEL)。

实现 (aicpu_ts_urma_channel_kernel.cc):
```cpp
extern "C" {
__attribute__((visibility("default"))) uint32_t RunAicpuIndOpChannelInitV2(void *args) {
    HCCL_RUN_INFO("RunAicpuIndOpChannelInitV2 start.");
    CHK_PRT_RET(args == nullptr, HCCL_ERROR("[%s]args is null.", __func__), HCCL_E_PARA);
    struct InitTask {
        u64 context;
        bool isCustom;
    };
    InitTask *ctxArgs = reinterpret_cast<InitTask *>(args);
    HcclChannelUrmaRes *commParam = reinterpret_cast<HcclChannelUrmaRes *>(ctxArgs->context);
    return AicpuHcclProcess::AicpuIndOpChannelInitV2(commParam);
}
}
```

Kernel 函数的标准模式:
1. extern "C" + __attribute__((visibility("default"))): 使符号对动态加载可见
2. void *args 通用入口: 所有 AICPU kernel 统一签名
3. 两层 reinterpret_cast 解包:
   - args → InitTask (获取 context 地址和 isCustom 标志)
   - InitTask::context → HcclChannelUrmaRes (获取通道资源参数)
4. 委托给 AicpuHcclProcess 静态方法

与 ccl_kernel.ini 注册的关系:
```ini
[RunAicpuIndOpChannelInitV2]
opInfo.functionName=RunAicpuIndOpChannelInitV2
```
kernel 函数通过 .ini 文件注册，runtime 通过函数名字符串动态查找。

HcclChannelUrmaRes 结构体 (channel_param.h:71-78):
```cpp
struct HcclChannelUrmaRes {
    char  hcomId[HCOMID_MAX_LENGTH];  // 通信域ID
    void* channelList;                // 反序列化后返回给host侧的device侧handle地址
    u32   listNum;                    // 建链channel的总数量
    void* uniqueIdAddr;              // 序列化后device侧地址
    u32   uniqueIdSize;              // 序列化后总地址长度
    u32   singleUniqueIdSize;        // 单个channel内序列化后地址长度
};
```
这是 H2D 交互的核心数据结构: Host 把序列化数据写入 device memory，
然后通过 kernel 调用让 Device 侧反序列化并建立通道。

#### 空壳文件

- aicpu_ts_roce_channel_kernel.h: 仅注释 "AicpuTsRoceChannel的Device Aicpu kernel的数据结构、函数声明"，空体。
- aicpu_ts_thread_kernel.h (threads/slaves/): 仅定义空结构体 `AicpuTsThreadKernel`。

这两个空壳说明 ROCE 通道的 Device 侧逻辑和 Thread 的 Device 侧逻辑尚未从 legacy 迁移到 next 框架。

#### CMake 构建配置

```cmake
# slaves/CMakeLists.txt
if(NOT BUILD_OPEN_PROJECT OR (BUILD_OPEN_PROJECT AND KERNEL_MODE))
    target_sources(ccl_kernel PRIVATE ${src_list})
endif()
```
slaves/ 代码编译到 `ccl_kernel` 目标(而非 `hcomm` 目标)，
这是因为 kernel 代码运行在 Device 端的 AICPU 上。

threads/CMakeLists.txt 同理:
```cmake
target_sources(hcomm PRIVATE ${src_list})       # Host 目标
if(NOT BUILD_OPEN_PROJECT OR (BUILD_OPEN_PROJECT AND KERNEL_MODE))
    target_sources(ccl_kernel PRIVATE ${src_list})  # Kernel 目标(复用相同源码)
endif()
```
Thread 的源码被同时编译到 hcomm(Host) 和 ccl_kernel(Device) 两个目标，
通过 `#ifdef CCL_KERNEL_AICPU` 和 `GetRunSideIsDevice()` 区分行为。

### Part 3: comm_primitives/ — 通信原语 C 适配层

路径: next/comm_primitives/api_c_adpt/
文件数: 4 (2对 .h/.cc)

#### 模块职责

为 AICPU 和 AIV 通信引擎提供数据面接口(通信原语)的 C 到 C++ 适配。
目前全部为空实现。

#### AICPU 适配 (aicpu/)

```cpp
// aicpu_primitives_c_adpt.h
#ifdef __cplusplus
extern "C" {
#endif
// 职责：aicpu通信引擎的数据面接口（通信原语）的C接口声明（暂未对外的接口）
#ifdef __cplusplus
}
#endif
```

```cpp
// aicpu_primitives_c_adpt.cc
#include "aicpu_primitives_c_adpt.h"
// 职责：aicpu通信引擎的数据面接口（通信原语）的C到C++适配
```

#### AIV 适配 (aiv/)

结构与 AICPU 完全相同，也是空实现。

#### 分析

这4个文件是纯占位: 头文件声明了 extern "C" 块但内部无函数，.cc 文件仅 include 头文件。
从注释可知:
- 这些是"暂未对外的接口"，计划中的但尚未实现
- 对应 Platform 层 comm_primitive/ 中已有的完整实现(HcclLocalCopy, HcclRemoteRead 等)
- next 框架的通信原语最终会通过这些 C 适配层暴露，但目前 next 框架的 Thread 类直接委托给 IAicpuTsThread/pImpl

### Part 4: common/enum_factory.h — 枚举工厂宏

路径: next/common/enum_factory.h
引用位置: 5 个文件

#### MAKE_ENUM 宏分析

MAKE_ENUM 是一个强大的宏，用于生成"增强枚举类":

```cpp
#define MAKE_ENUM(enumClass, ...)
    class enumClass {
    public:
        enum Value : uint8_t { __VA_ARGS__, __COUNT__, INVALID };
        // 构造、比较运算符...
        std::string Describe() const;                      // 枚举值→字符串
        friend std::ostream& operator<<(std::ostream&, const enumClass&);
    private:
        Value value{INVALID};
        static std::vector<std::string> InitStrs();        // 编译时字符串化
    };
```

使用方式 (18处实例):
```cpp
MAKE_ENUM(CcuEngine, CCU_MS, CCU_SCHE);
MAKE_ENUM(ChannelStatus, INIT, SOCKET_OK, SOCKET_TIMEOUT, READY, FAILED)
MAKE_ENUM(RequestResult, RESULT_OK, RESULT_FAIL, ...);
```

设计优点:
1. 类型安全: 包装为 class 而非裸 enum，防止隐式整数转换
2. 自动 Describe(): 通过 `#__VA_ARGS__` 宏字符串化 + 运行时解析，自动生成 "ClassName::ValueName"
3. 内置 __COUNT__ 和 INVALID 哨兵值
4. 支持 operator<< 直接打印
5. 底层 uint8_t: 节省存储

InitStrs() 实现细节:
```cpp
static std::vector<std::string> InitStrs() {
    std::string s = #__VA_ARGS__;  // 将参数列表字符串化
    // 按 ',' 和 ' ' 分隔解析出各枚举名
    // 拼接 "enumClass::token" 格式
}
```
static 局部变量保证只初始化一次(线程安全，C++11)。

设计缺陷:
1. Describe() 中: `if (value > m.size())` — 应为 `>=`，当 value == m.size() 时会越界。
   但因为 __COUNT__ 和 INVALID 的存在，实际 value 通常不会等于 m.size()，
   不过 INVALID 值(= __COUNT__ + 1)可能触发此边界问题。
2. 默认值 INVALID: 未初始化的枚举值为 INVALID，增加了防御成本
3. uint8_t 限制: 最多 253 个枚举值(256 - __COUNT__ - INVALID - 1)
4. 运行时开销: 每次 Describe() 都检查 static 变量(但实际只初始化一次)

EnumClassHash:
```cpp
namespace std {
struct EnumClassHash {
    template <typename T> std::size_t operator()(T t) const {
        return static_cast<std::size_t>(t);
    }
};
}
```
放在 `namespace std` 中为 MAKE_ENUM 生成的类型提供 hash 支持，
这样可以直接用作 unordered_map 的 key。

注意: 直接向 `namespace std` 注入非标准类型的 hash 特化是不推荐的做法。
标准建议是为自定义类型提供 std::hash 特化或在容器模板参数中指定 hash 类型。

### 惯用法总结

1. Host/Device 双编译 + 运行时分派:
   - 同一源文件编译到 hcomm 和 ccl_kernel 两个目标
   - `GetRunSideIsDevice()` 运行时判断当前执行环境
   - `#ifdef CCL_KERNEL_AICPU` 条件编译设备特有逻辑
   - 出现位置: AicpuTsThread::Init(), CpuTsThread::Init()

2. 二进制序列化 + UniqueId 模式:
   - 用 ostringstream::write 做 POD 结构体的二进制序列化
   - 序列化结果作为 "UniqueId" 字符串用于 H2D 传递
   - Device 侧用 istringstream::read 按相同顺序反序列化
   - 无版本号，无自描述能力 (与 2.2.3 中发现的模式一致)

3. A3/A5 硬件代际分叉:
   - `DevType::DEV_TYPE_910_95` 是 A5 代际标识
   - A5: StreamLite + NotifyLite + IAicpuTsThread
   - A3: 传统 Stream + SqCqeContext + 直接 SQ/CQ 操作
   - 分叉贯穿 Init/Notify/Stream 所有子系统

4. AICPU Kernel 入口函数标准签名:
   - `extern "C" __attribute__((visibility("default"))) uint32_t FuncName(void *args)`
   - 两层 reinterpret_cast 解包 (args → 上下文结构 → 资源参数)
   - 委托给 AicpuHcclProcess 静态方法
   - 通过 .ini 文件注册，运行时按名字查找

5. 析构时序敏感资源释放:
   - CcuResContainer 注释强调 "不得随意调整顺序"
   - 先 UnRegister kernel → 再释放 resPack → 最后释放 DrvHandle
   - (void) 吞掉析构中的错误

6. MAKE_ENUM 枚举工厂:
   - 18处使用，覆盖所有状态机枚举(ChannelStatus, CcuConnStatus, RdmaStatus等)
   - 提供类型安全 + 自动字符串化 + operator<<

### 反模式

1. Thread 基类方法通过 reinterpret_cast<Thread*>(uint64_t) 恢复指针:
   - thread.h:63, thread.cc:102
   - 无类型安全检查，如果 ThreadHandle 值被损坏将导致未定义行为
   - 影响: 通过 ThreadHandle 的所有操作

2. CpuTsThread::GetUniqueId() 日志标签错误:
   - cpu_ts_thread.cc:121,125,131 都使用 "[AicpuTsThread]" 而非 "[CpuTsThread]"
   - 典型 copy-paste 错误，增加调试时的混淆

3. AicpuTsRtsNotify 声明 `GetNotifyNum() const override` 但基类 Notify 无此虚方法:
   - aicpu_ts_rts_notify.h:25
   - 可能导致编译错误(如果实际编译)
   - 推测: 基类可能在不同编译配置下有不同版本

4. EngineCtxMgr .h 和 .cc 类型不一致:
   - .h: `std::unordered_map<std::string, void*>`
   - .cc: 使用 .get() 和 std::move 操作 unique_ptr
   - 推测: .h 声明过时或简化，但造成了接口定义不准确

5. MAKE_ENUM::Describe() 边界问题:
   - `if (value > m.size())` 应为 `if (value >= m.size())`
   - 当 value == m.size() 时可能越界访问 vector

6. MAKE_ENUM 向 namespace std 注入:
   - enum_factory.h:104 直接在 std namespace 中定义 EnumClassHash
   - 不符合标准建议的 hash 特化方式

7. header guard typo:
   - aicpu_ts_urma_channel_kernel.h: `AICPU_TS_URMA_CAHNNEL_KERNEL_H` (CAHNNEL→CHANNEL)
   - 不影响功能但影响代码质量

### 硬件抽象知识

1. 通信引擎类型 (CommEngine):
   - CPU_TS / CPU: Host 侧调度引擎 (Task Scheduler)
   - AICPU_TS / AICPU: Device 侧 AI CPU 调度引擎
   - AIV: AI Vector 引擎 (未完全支持)
   - CCU: Communication Control Unit (独立通信控制单元)

2. Notify 类型映射:
   - CPU 系 (CPU/CPU_TS/CCU) → HOST_NOTIFY
   - AICPU 系 (AICPU/AICPU_TS) → DEVICE_NOTIFY

3. Stream 类型映射:
   - CPU 系 → STREAM_TYPE_ONLINE (单算子用 online，图模式用 offline)
   - AICPU 系 → STREAM_TYPE_DEVICE

4. SQ/CQ 硬件概念:
   - SQ (Submission Queue): 提交队列，Host 写入命令
   - CQ (Completion Queue): 完成队列，Device 写入完成状态
   - SqCqeContext: SQ/CQ 运行时上下文(头尾指针、深度等)
   - 每个 Thread 对应一个 SQ/CQ 对

5. CCU 运行模式:
   - DEFAULT(0) / CCU_SCHE(6) → CCU_SCHE 调度引擎模式
   - CCU_MS(5) → CCU_MS 微服务引擎模式
   - kernel 有"未翻译"状态，需要额外的翻译步骤

6. A3 vs A5 架构差异:
   - A3 (910B/910_93等): 传统 Stream + SQ/CQ 直接操作 + Notify IPC
   - A5 (910_95): StreamLite API (rtsq) + NotifyLite + 无需 IPC


---

## 2.2.4 next/comms/ccu/ — CCU (Communication Control Unit) Deep Dive

### Module Overview

CCU(Communication Control Unit)是华为Ascend NPU上的硬件通信加速器，
next/comms/ccu/ 子模块是其软件栈的完整实现，包含5个层次:

1. ccu_device/ — 硬件资源发现、分配、管理(驱动交互层)
2. ccu_transport/ — CCU连接建立、握手、通道管理(传输层)
3. ccu_kernel/ — CCU程序编写API和生命周期管理(编程模型)
4. ccu_representation/ — 中间表示(IR)，指令集抽象(编译器前端)
5. ccu_microcode/ — 二进制指令生成(编译器后端)

文件规模: 50个.h + 69个.cc = 119个文件(不含pub_inc/)

在整体架构中的位置:
- 位于 framework/next/comms/ 下，是 next 新框架的子模块
- 通过 pub_inc/ 向上层暴露 C 函数 API (CcuInitFeature/CcuAllocChannels等)
- 向下依赖 hccp/RaCustomChannel 与硬件驱动交互
- 通过 endpoint_pairs/channels/ccu/ccu_urma_channel 与 next 通信框架集成
- 与 legacy/unified_platform/ccu/ 存在并行实现关系

### Key Files (按层次)

**pub_inc/ (公共接口)**
- ccu_dev_mgr_pub.h — 9个C函数API + CcuEngine/CcuJettyType枚举 + CcuChannelPara/CcuJettyInfo结构
- ccu_res_pack.h — CcuResPack RAII资源包装器

**ccu_device/ (驱动+资源层, 18个.h/.cc对)**
- ccu_dev_mgr_imp.h/.cc — 静态API代理类(CcuDevMgrImp, 全static方法)
- ccu_drv_handle.h/.cc — 驱动句柄RAII包装(CcuDrvHandle)
- ccu_res_specs.h/.cc — 硬件能力寄存器解析(CcuResSpecifications)
- ccu_res_batch_allocator.h/.cc — 块/离散/连续三种资源分配(CcuResBatchAllocator, 806行)
- ccu_comp/ccu_comp.h/.cc — 中央协调器(CcuComponent, 884行)
- ccu_comp/ccu_res_allocator.h/.cc — First-fit ID分配器(CcuResIdAllocator)
- ccu_comp/ccu_channel/ — Channel/Jetty/PFE/WQE管理(10个文件)

**ccu_transport/ (传输层, 5个.h/.cc对)**
- ccu_transport_.h/.cc — 12态状态机(CcuTransport, 557行)
- ccu_conn.h/.cc — 双层状态机(CcuConnection, 539行)
- ccu_jetty_.h/.cc — 异步Jetty创建(CcuJetty)
- ccu_channel_ctx_pool.h/.cc — 通道上下文池管理(CcuChannelCtxPool)

**ccu_kernel/ (编程模型, 2个.h/.cc对)**
- ccu_kernel_mgr.h/.cc — 内核生命周期管理(CcuKernelMgr, 694行)
- ccu_kernel.h/.cc — 编程API基类(CcuKernel, 670行)

**ccu_representation/ (中间表示, 29个.h + 46个.cc)**
- context/ccu_rep_context.cc — Rep上下文(mainBlock/activeBlock)
- reps/common/ — 基础Rep: Nop/Load/LoadArg/LoadVar/Store (5+5对)
- reps/arithmetic/ — 算术Rep: Add/Assign (2+3对)
- reps/control/ — 控制流Rep: FuncBlock/FuncCall/Jump/JumpLabel (0+4对)
- reps/data/ — 数据传输Rep: Read/Write/LocCpy/BufRead/BufWrite/BufReduce/BufLocRead/BufLocWrite/RemMem (9+9对)
- reps/loop/ — 循环Rep: SetLoop/Loop/LoopGroup/LoopBlock/LoopCall (3+5对)
- reps/sync/ — 同步Rep: RemPostSem/RemPostVar/RemWaitSem/LocRecordEvent/LocWaitEvent/LocWaitNotify/RecordSharedNotify (7+7对)
- reps/translator/ — Rep→指令翻译器(CcuRepTranslator, 2对)
- interface/ — 高级接口: FuncBlock/FuncCall/LoopBlock/LoopCall/LoopGroupCall/Condition/Repeat/DataType/InterfaceAssist (0+9对)

**ccu_microcode/ (指令生成, 3个.cc)**
- ccu_microcode.cc — V1指令编码(~500行)
- ccu_microcode_v2_.cc — V2指令编码(~426行)
- ccu_assist.cc — 辅助函数(数据类型/Reduce类型映射, Token查询)

---

### 1. CCU设备通信模型

#### IO Die双die架构

CCU硬件以IO Die为单位组织，每颗NPU最多包含2个IO Die(CCU_MAX_IODIE_NUM = 2)。
所有资源管理都是per-die的，资源规格通过能力寄存器per-die查询。

```
// ccu_res_specs.h:71
std::array<bool, CCU_MAX_IODIE_NUM> dieEnableFlags_{}; // per-die使能标记
std::array<CcuResSpecInfo, CCU_MAX_IODIE_NUM> resSpecs_{}; // per-die资源规格
```

关键设计约束:
- Die间通信通过Loopback Channel(自发自收)实现: dstDieId = 1 - dieId (ccu_comp.cc:485)
- 这个 `1 - dieId` 假设硬编码了"恰好2个die"的前提
- A+X形态(PCIe连接到IOdie0)需要特殊处理: CCUA0不可用，分配MS时跳过 (ccu_res_batch_allocator.cc:171)

#### URMA通信协议

CCU使用URMA(Unified Remote Memory Access)协议进行通信，
关键概念映射:

| CCU概念 | URMA对应 | 用途 |
|---------|---------|------|
| Jetty | Queue Pair | 发送/接收队列 |
| PFE | Port Function Engine | 前端ID→Jetty上下文映射 |
| WQE/WQEBB | Work Queue Entry | 提交队列条目(64字节, 每SQE含4个WQEBB) |
| Channel | 通信链路 | 封装了Jetty+PFE+Channel Context的完整链路 |

通信路径:
1. 本地CCU资源空间(72MB)→ URMA Jetty → 远端CCU资源空间
2. 本地HBM内存 → CCU MS buffer → URMA传输 → 远端MS buffer → 远端HBM内存

#### 资源空间布局

CCU资源空间总大小72MB(CCU_RESOURCE_SIZE = 72 * 1024 * 1024)，V1版本固定布局:
```
// ccu_res_specs.h
Offset 0x000000: CCUA区域
Offset 0x800000: CCUM区域 (CCU_V1_CCUM_OFFSET)
Offset 0x1000000: WQE Basic Block区域 (CCU_V1_WQE_BASIC_BLOCK_OFFSET)
```

每个资源类型有硬件寄存器定义的最大数量:
- Loop Engine: ~16个/die
- MS(Memory Space): ~64个/die (每个4KB, CCU_MS_SIZE=4096)
- CKE(Completed Key Event): ~64个/die
- XN(Variable): ~256个/die
- GSA(Address): ~32个/die
- Instruction: 1MB预留空间(CCU_RESOURCE_INS_RESERVE_SIZE = 0x100000)
- Mission: ~8个/die
- Jetty: ~128个/die (CCU_PER_DIE_JETTY_RESERVED_NUM = 128)
- PFE: 16个/die (CCU_V1_PER_DIE_PFE_RESERVED_NUM = 16)
- WQEBB: 4096个 (CCU_WQEBB_RESOURCE_NUM)

---

### 2. 核心类层次和继承关系

#### 2.1 Per-Device单例模式 (Static Array Singleton)

CCU模块有5个类使用完全相同的单例模式:

```cpp
// 模板: static T instances[MAX_MODULE_DEVICE_NUM + 1]
static CcuComponent &GetInstance(const int32_t deviceLogicId) {
    static CcuComponent ccuComponent[MAX_MODULE_DEVICE_NUM + 1];
    int32_t devLogicId = deviceLogicId;
    if (devLogicId < 0 || static_cast<uint32_t>(devLogicId) >= MAX_MODULE_DEVICE_NUM) {
        HCCL_WARNING(...);
        devLogicId = MAX_MODULE_DEVICE_NUM; // 使用备份设备(最后一个槽位)
    }
    ccuComponent[devLogicId].devLogicId_ = devLogicId;
    return ccuComponent[devLogicId];
}
```

使用此模式的类(77处GetInstance调用覆盖20个文件):
1. CcuResSpecifications — 硬件能力规格(per-device)
2. CcuComponent — 中央协调器(per-device)
3. CcuResBatchAllocator — 资源批量分配器(per-device)
4. CcuPfeCfgMgr — PFE配置管理(per-device)
5. CcuKernelMgr — 内核管理(per-device)

特征:
- 备份槽位(MAX_MODULE_DEVICE_NUM)用于兜底非法deviceLogicId
- 所有实例在程序生命周期内不销毁(static局部变量)
- Init/Deinit成对调用，initFlag_防止重复初始化

#### 2.2 CcuDevMgrImp: 纯静态代理类

```cpp
// ccu_dev_mgr_imp.h
class CcuDevMgrImp {
    CcuDevMgrImp() = delete;
    ~CcuDevMgrImp() = delete;
    // 全部 static 方法，转发到对应的单例实例
    static HcclResult CcuAllocEngineResHandle(...);
    static HcclResult CcuReleaseResHandle(...);
    // ...
};
```

CcuDevMgrImp不持有任何状态，每个static方法内部调用 XxxClass::GetInstance(devLogicId).Method()。
这是一种Facade模式——提供统一的调用入口，隐藏内部5个单例之间的依赖关系。

#### 2.3 CcuDrvHandle: 驱动句柄RAII

```
CcuDrvHandle::Init() → TLV HDC初始化 → 发送CCU_INIT请求 → 初始化子组件
CcuDrvHandle::Deinit() → 逆序释放(所有错误用(void)忽略，不中断释放流程)
```

全局管理: `static std::map<uint64_t, std::shared_ptr<CcuDrvHandle>> ccuDrvHandleMap` + mutex
多实例引用计数: CcuInitFeature只在首次调用时创建，CcuDeinitFeature在use_count==1时销毁。

#### 2.4 Channel/Jetty上下文管理(虚方法体系)

```
CcuChannelCtxMgr (abstract)       CcuJettyCtxMgr (abstract)
    │                                  │
    └── CcuChannelCtxMgrV1             └── CcuJettyCtxMgrV1
```

- CcuChannelCtxMgr: 5个virtual方法(Init/Alloc/Config/Release + Deinit)
  CcuChannelCtxMgrV1: 64字节packed struct ChannelCtxDataV1用于硬件寄存器编程
- CcuJettyCtxMgr: 5个virtual方法(Init/Alloc/Config/Release/Deinit)
  CcuJettyCtxMgrV1: 32字节packed struct LocalJettyCtxData，bit field精确匹配硬件寄存器
- CcuChannelCtxPool: 非继承，管理ResourceBatch(批量分配的通道组)
- 虚方法总数: 11个(3个.h文件, 覆盖Channel和Jetty两个维度)
- V1/V2版本通过CcuVersion选择: CreateChannelCtxMgrByVersion工厂函数(ccu_comp.cc)

#### 2.5 CcuRepBase: 表示层类层次(26个子类)

```
CcuRepBase (abstract)
    ├── CcuRepBlock (abstract)
    │   ├── CcuRepFuncBlock
    │   └── CcuRepLoopBlock
    │
    ├── Common: CcuRepNop, CcuRepLoad, CcuRepLoadArg, CcuRepLoadVar, CcuRepStore
    ├── Arithmetic: CcuRepAdd, CcuRepAssign
    ├── Data: CcuRepRead, CcuRepWrite, CcuRepLocCpy, CcuRepBufRead, CcuRepBufWrite,
    │         CcuRepBufReduce, CcuRepBufLocRead, CcuRepBufLocWrite, CcuRepRemMem
    ├── Loop: CcuRepSetLoop, CcuRepLoop, CcuRepLoopGroup
    └── Sync: CcuRepRemPostSem, CcuRepRemPostVar, CcuRepRemWaitSem,
              CcuRepLocRecordEvent, CcuRepLocWaitEvent, CcuRepLocWaitNotify,
              CcuRepRecordSharedNotify
```

每个Rep子类实现两个虚方法:
- Translate(CcuInstr *&instr, uint16_t &instrId, const TransDep &dep) → 生成微码指令
- Describe() → 返回人类可读描述字符串

共26个具体Rep类，覆盖CCU所有可编程操作类型。

#### 2.6 CcuConnection协议子类

```
CcuConnection
    ├── CcuRtpConnection  (tpProtocol_ = TP_RC)
    └── CcuCtpConnection  (tpProtocol_ = TP_CTP)
```

极简子类——仅在构造函数中设置协议类型字段，无方法覆盖。

---

### 3. CCU Kernel/Microcode抽象

#### 3.1 编程模型概览

CCU的编程模型类似GPU shader编程:
1. 用户继承CcuKernel，实现Algorithm()虚方法
2. Algorithm()内调用编程API(Load/ReadNb/WriteNb/ReduceNb等)构建Rep序列
3. CcuKernelMgr::Register() 分配资源 → Translate() 翻译Rep为微码 → LoadInstruction()加载到硬件

```
用户代码                    CcuKernel API           Representation            Microcode
┌─────────┐ Algorithm() ┌─────────────┐ 构建Rep ┌──────────────┐ Translate ┌────────────┐
│ MyKernel ├────────────>│ ReadNb()    ├────────>│ CcuRepRead   ├──────────>│ TransRmt.. │
│          │             │ WriteNb()   │         │ CcuRepWrite  │           │ TransLoc.. │
│          │             │ ReduceNb()  │         │ CcuRepBuf..  │           │ Reduce..   │
│          │             │ NotifyWait()│         │ CcuRepRem..  │           │ SetCKE..   │
└─────────┘             └─────────────┘         └──────────────┘           └────────────┘
```

#### 3.2 CcuKernel编程API (ccu_kernel.cc)

资源创建API:
- CreateVariable() → Variable(对应XN寄存器)
- CreateAddress() → Address(对应GSA寄存器)
- CreateLocalNotify() → LocalNotify(对应CKE)
- CreateCompletedEvent() → CompletedEvent(同CKE)
- CreateCcuBuf() → CcuBuf(对应MS)
- CreateExecutor() → Executor(对应Loop Engine)
- CreateLocalAddr/CreateRemoteAddr → 地址封装
- CreateBlockXxx → 块资源变体

数据传输API:
- ReadNb(channel, loc, rem, len, sem, mask) → CcuRepRead
- WriteNb(channel, loc, rem, len, sem, mask) → CcuRepWrite
- ReadReduceNb(..., dataType, opType) → CcuRepRead(带reduce)
- WriteReduceNb(..., dataType, opType) → CcuRepWrite(带reduce)
- LocalCopyNb(src, dst, len, sem, mask) → CcuRepLocCpy
- LocalReduceNb(..., dataType, opType) → CcuRepBufReduce

同步API:
- NotifyRecord(sem, mask) → SetCKE指令
- NotifyWait(sem, mask) → 等待CKE
- RecordEvent(event) → CcuRepLocRecordEvent
- WaitEvent(event) → CcuRepLocWaitEvent

控制流:
- FuncCall(label) → CcuRepFuncCall + CcuRepFuncBlock
- LoopCall(label, count, offset, contextId) → CcuRepLoopCall
- Load/LoadVariable/StoreVariable → 数据加载到XN/GSA

#### 3.3 CcuKernelMgr生命周期管理 (ccu_kernel_mgr.cc)

Register → Translate → UnRegister 三阶段:

**Register(kernel)**:
1. AllocRes(resReq) → 从CcuResBatchAllocator分配资源
2. 分配kernelId
3. MoveResInfo: 将CcuResPack中的资源信息转移到CcuKernel
4. 存入kernelMap_[kernelId]

**Translate(kernelId)**:
1. TransRepResToPhyRes: 将Rep层虚拟资源ID映射为物理资源ID
2. TransRepSequenceToMicrocode: 调用CcuRepTranslator翻译Rep序列为CcuInstrInfo
3. LoadInstruction: Host→Device memcpy + RaCustomChannel SET_INSTRUCTION

**Inter-kernel resource sharing**:
ProcessInterCtxRes: 通过tag匹配，在多个kernel之间共享CKE/XN等通知资源。
MergeExportedNotifyByTag/ResetImportedNotifyByTag实现导出/导入语义。

#### 3.4 CcuRepTranslator翻译流程 (ccu_rep_translator.cc)

翻译按4阶段有序执行:
1. 翻译LoopBlock — filter: rep->Type() == LOOP_BLOCK
2. 翻译FuncBlock — filter: rep->Type() == FUNC_BLOCK
3. 翻译LoadArg — filter: rep->Type() == LOAD_ARG
4. CommonProcess(插入3条初始化指令)
5. 翻译主体 — filter: true (所有剩余Rep)
6. FinishMainBlock(插入终止指令)

翻译器自身需要4个XN + 3个GSA + 2个CKE固定资源(BindResource)。

多轮翻译策略: 如果某个Rep依赖其他Rep的结果(如FuncCall依赖FuncBlock的地址)，
Translate()返回false表示"未完成"，翻译器最多重试10次(maxTryCount = 10)。

默认指令容量: 32K条(defaultInstrCapacity = 32 * 1024)。

#### 3.5 Microcode指令格式

CcuInstr = CcuInstrHeader(2字节: 11位code + 4位type + 1位reserved) + 28字节payload

四种指令类型:
- LOAD_TYPE(0x0): 数据加载/算术 — 7个V1子码 + 16个V2子码
- CTRL_TYPE(0x1): 控制流 — 5个V1子码 + 8个V2子码
- TRANS_TYPE(0x2): 数据传输 — 14个V1子码 + 6个V2子码
- REDUCE_TYPE(0x3): 规约运算 — 3个V1子码 + 3个V2子码

V1→V2演进:
- V1使用GSA(64位地址)和XN(64位变量)分离的设计
- V2统一为X寄存器，增加了ALU指令(Sub/Mul/And/Or/Not/Xor/Shl/Shr/Popcnt)
- V2增加了TransMem统一传输指令和缓存配置(CacheConfig: allocHint + victimHint)
- V2的Loop指令参数从packed XN寄存器改为独立字段
- V2新增SyncWtX/SyncAtX远端写/原子写指令

数据类型映射(ccu_assist.cc):
- CCU SUM: FP32(0)/FP16(1)/BFP16(2)/HIF8(3)/FP8E4M3(4)/FP8E5M2(5)/INT8(6)/UINT8(7)/INT16(8)/INT32(9)
- CCU MAX/MIN: 同上但不支持HIF8/FP8E4M3/FP8E5M2

---

### 4. CCU Transport层设计

#### 4.1 CcuTransport 12态状态机

```
INIT → SEND_ALL_INFO → RECV_ALL_INFO → SEND_TRANS_RES → RECV_TRANS_RES
     → SEND_FIN → RECVING_FIN → RECV_FIN → READY
```

每步通过 Socket(TCP)发送/接收 BinaryStream 序列化数据。
FINISH_MSG: 128字节 "Transport exchange data ready!" 作为完成握手标记。

交换的信息:
- 连接信息(ConnectionInfo): buffer地址/大小/JettyKey/TP参数
- 传输资源(TransRes): CKE ID列表 + XN ID列表
- 缓冲区信息(CclBufferInfo): HBM地址/大小/TokenId/TokenValue

#### 4.2 CcuConnection 双层状态机

外层: CcuConnStatus(8态 — INIT/CONNECTING/CHANNEL_CONFIGED/CONNECTED/...)
内层: InnerStatus(6态 — INIT/JETTY_CREATING/TP_INFO_GETTING/EXCHANGEABLE/JETTY_IMPORTING/CONNECTED)

异步操作:
- CreateJetty: 通过 HccpUbCreateJettyAsync 异步创建，HandleAsyncRequest轮询完成
- ImportJetty: 通过 HccpUbTpImportJettyAsync 异步导入远端Jetty信息

#### 4.3 CcuJetty异步创建模型

```cpp
// ccu_jetty_.cc
HcclResult CcuJetty::HandleAsyncRequest() {
    if (!isCreated_) {
        // 首次调用: 触发异步创建
        EXCEPTION_HANDLE_BEGIN
        HccpUbCreateJettyAsync(handle, &requestHandle_);
        EXCEPTION_HANDLE_END
        isCreated_ = true;
    }
    // 后续调用: 轮询完成状态
    EXCEPTION_HANDLE_BEGIN
    bool isDone = false;
    HccpUbHandleRequest(requestHandle_, &isDone, &qpCreateInfo_);
    if (!isDone) return HCCL_E_AGAIN;
    EXCEPTION_HANDLE_END
    ParseCreateInfo(qpCreateInfo_);
    return HCCL_SUCCESS;
}
```

特征: isError_标记防止失败后重试(一旦失败永不再尝试)。

#### 4.4 CcuChannelCtxPool批量管理

ResourceBatch: 按源IP分组批量分配的通道集合。
PrepareCreate: 批量预创建，复用已存在的batch。
UnconfirmedRecord: 跟踪确认前的分配记录，支持回滚。
自定义hash: ResIdHash使用Hccl::HashCombine。

---

### 5. CCU资源管理体系

#### 5.1 三种分配策略

CcuResBatchAllocator实现了三种资源分配策略:

1. **Block分配**: Loop/MS/CKE三种资源按固定块大小(默认8/64/8)预分配，
   分配时在预分配块中做first-fit查找(HandleBlockRes)
2. **连续分配**: XN等资源需要连续ID(true参数)，直接从CcuResIdAllocator连续分配
3. **离散分配**: Loop/MS/CKE/XN/GSA可以非连续分配(false参数)

资源分配顺序(TryAllocResHandle):
Block → Mission → Consecutive → Discrete

分配失败时返回 HCCL_E_UNAVAIL 而非 HCCL_E_INTERNAL，调用方可据此做降级处理。

#### 5.2 CcuResIdAllocator: First-fit with Coalescing

```cpp
// ccu_res_allocator.cc
HcclResult CcuResIdAllocator::Alloc(uint32_t reqNum, bool needConsecutive, vector<ResInfo> &resInfos) {
    // 在allocated_列表的间隙中寻找足够空间
    // Sentinel trick: 临时插入尾部元素简化边界处理
    allocated_.emplace_back(ResInfo{totalNum_, 0});
    // Sequential scan...
    allocated_.pop_back();
}
```

Release支持部分释放(split/merge): 可以释放一个已分配块的中间部分，
自动将块一分为二或从两端缩减。

#### 5.3 CcuResRepository: 完整资源仓库

```cpp
struct CcuResRepository {
    // per-die, 3种block资源
    array<vector<ResInfo>, CCU_MAX_IODIE_NUM> blockLoopEngine, blockMs, blockCke;
    // per-die, 5种离散资源
    array<vector<ResInfo>, CCU_MAX_IODIE_NUM> loopEngine, ms, cke, xn, gsa;
    // per-die, 连续XN
    array<vector<ResInfo>, CCU_MAX_IODIE_NUM> continuousXn;
    // Mission(跨die共享)
    MissionResInfo mission;
};
```

Handle管理: 使用raw pointer的uintptr_t值作为handleMap_的key，
CcuResHandle = void*(通过reinterpret_cast双向转换)。

#### 5.4 Mission资源特殊性

Mission是CCU的执行上下文，具有跨die约束:
- FUSION_MULTIPLE_DIE模式: 两个die必须分配相同startId的mission
- 如果die0和die1的startId不一致，直接报错(不降级)
- 这是当前唯一支持的Mission分配模式(其他类型会被强制降级到FUSION_MULTIPLE_DIE)

---

### 6. 与channels/ccu/ccu_urma_channel的关系

CcuUrmaChannel(位于endpoint_pairs/channels/ccu/)是CCU模块与next通信框架的桥接点:

```
CcuUrmaChannel : public Channel
    ├── 持有 unique_ptr<CcuTransport> impl_
    ├── 提供 GetChannelId() / GetDieId() 给Rep层
    ├── 提供 GetLocCke/GetRmtCke/GetLocXn/GetRmtXn 给Kernel层
    └── 提供 GetRmtBuffer() 给Transport层
```

关键交互点:
1. Rep层通过dynamic_cast获取CcuUrmaChannel:
   ```cpp
   // ccu_rep_read.cc:49 (8个data/sync文件都有同样的模式)
   auto *channelImpl = dynamic_cast<CcuUrmaChannel *>(static_cast<Channel *>(channelPtr));
   ```
   然后调用 channelImpl->GetChannelId() 生成对应的传输指令

2. CcuKernel层同样通过dynamic_cast:
   ```cpp
   // ccu_kernel.cc:71, :181
   auto *channelImpl = dynamic_cast<CcuUrmaChannel *>(static_cast<Channel *>(channelPtr));
   ```

3. 依赖方向: ccu_representation/ → ccu_urma_channel.h (通过相对路径include)
   ccu_kernel/ → ccu_urma_channel.h (同样通过相对路径include)

---

### 7. 惯用法

#### 7.1 错误处理

**CHK_RET宏**: 214次出现，覆盖19个文件。是CCU模块最基本的错误检查模式。

**双轨错误处理**: CCU模块存在两种并行的错误处理风格:
- CHK_RET(返回错误码路径): 用于ccu_device/和ccu_transport/的所有代码
- THROW(异常路径): 用于ccu_representation/和ccu_microcode/的所有代码(86次THROW, 32个文件)

分界线清晰: device/transport层用错误码，representation/microcode层用异常。
这反映了两个不同团队的编码习惯或两个不同开发阶段的产物。
佐证: ccu_assist.cc:13 的TODO注释: `// todo: 需要统一整改为不抛异常`

**EXCEPTION_HANDLE_BEGIN/END**: 24次出现，6个文件。用于桥接异常和错误码:
在调用可能抛异常的URMA/HCCP API时包裹代码，catch后转为HcclResult。

**(void)忽略错误**: 44次出现，17个文件。主要用于Deinit/Release流程中:
```cpp
// 释放流程不中断原则
(void)ccuResSepcs.GetLoopEngineNum(dieId, loopNum);  // 查询可能失败但不阻塞释放
(void)ReleaseTransRes();  // 析构函数中释放
```

#### 7.2 Packed Struct硬件寄存器映射

CCU模块大量使用`#pragma pack(push, 1)`定义的packed struct，精确匹配硬件寄存器布局:

- ChannelCtxDataV1: 64字节
- LocalJettyCtxData: 32字节
- PfeCtx: 8字节
- CcuInstr: 30字节(2字节header + 28字节payload)
- CcuData/CcuDataUnion: 2048字节(驱动通信buffer)

特征:
- 使用bit field精确定义每个寄存器位域
- 硬件维护字段(如pi/ci/maxCi)在struct中声明但标记为"硬件回写"
- doorbell地址拆为4个16位字段(hardware format)

#### 7.3 MAKE_ENUM类型安全枚举

9处使用:
- CcuEngine, CcuJettyType(pub_inc/)
- CcuVersion, ResType(ccu_dev_mgr_imp.h)
- TransStatus, CcuConnectionType(ccu_transport_.h)
- CcuConnStatus, InnerStatus(ccu_conn.h)
- HcclMainboardId(ccu_res_specs.cc)

提供: Describe()字符串化 + operator<<日志输出 + 类型安全比较。

#### 7.4 constexpr函数指针表

```cpp
// ccu_res_specs.h:127-135
using GetResSpecFunc = HcclResult (CcuResSpecifications::*)(const uint8_t, uint32_t&) const;
constexpr ResSpecFuncPair GET_RES_SPEC_FUNC_ARRAY[] = {
    {ResType::LOOP, &CcuResSpecifications::GetLoopEngineNum},
    {ResType::MS, &CcuResSpecifications::GetMsNum},
    // ...
};
```

这是一种编译期多态: 通过constexpr数组+成员函数指针避免runtime switch/if。
CcuResAllocator在初始化时遍历此数组，按ResType查询对应的资源规格。

#### 7.5 BinaryStream序列化

Transport层使用BinaryStream进行Pack/Unpack:
- CcuTransport: 序列化ConnectionInfo, TransRes, CclBufferInfo
- CcuConnection: 序列化JettyKey, TpHandle, PSN, BufferInfo

模式固定: Pack写入 → Socket发送 → Socket接收 → Unpack读取。

#### 7.6 RAII with FuncBlock

CcuRepContext使用activeBlock栈管理"当前代码块":
```cpp
// ccu_funcblock.cc
FuncBlock::FuncBlock(context, label, callLayer) {
    repFuncBlock = make_shared<CcuRepFuncBlock>(label);
    AppendToContext(context, repFuncBlock);
    curActiveBlock = CurrentBlock(context);  // 保存
    SetCurrentBlock(context, repFuncBlock);  // 切换
}
FuncBlock::~FuncBlock() {
    SetCurrentBlock(context, curActiveBlock);  // 恢复
}
```

构造函数切换当前block，析构函数恢复——典型的RAII scope guard模式。

---

### 8. 反模式

#### AP-CCU-1: 深层相对路径include

10处使用 `../../../../` 级别的相对路径include:

```cpp
// ccu_rep_read.cc:17 (8个data/sync Rep文件都有)
#include "../../../../endpoint_pairs/channels/ccu/ccu_urma_channel.h"

// ccu_transport_.h:18
#include "../../../../../legacy/unified_platform/resource/socket/socket.h"

// ccu_comp.cc:15
#include "../../../../../legacy/common/sal.h"
```

问题: 路径脆弱，目录重构即断裂; 暴露了模块间依赖未通过CMake include path管理。
出现频次: 10处/5个不同路径模式

#### AP-CCU-2: CcuUrmaChannel的double-cast模式

所有需要获取CcuUrmaChannel的地方都使用相同的double-cast:
```cpp
auto *channelImpl = dynamic_cast<CcuUrmaChannel *>(static_cast<Channel *>(channelPtr));
```

问题:
- channelPtr是void* (ChannelHandle = void*)，先static_cast到Channel*再dynamic_cast
- 如果channelPtr不是Channel*，static_cast是未定义行为
- 13处完全相同的代码(13个文件)，应提取为公共函数

#### AP-CCU-3: std::rand()用于安全相关的随机数

```cpp
// ccu_comp.cc:473
uint32_t randNum = std::rand();
// ccu_conn.cc:193
uint32_t randNum = std::rand();
```

std::rand()不是线程安全的，也不是密码学安全的。
用途: 生成PSN(Packet Sequence Number)的初始值。
虽然PSN不需要密码学安全，但std::rand()的线程安全问题在多线程初始化场景中可能导致冲突。

#### AP-CCU-4: 变量拼写错误 "stragtegy"

```cpp
// ccu_res_batch_allocator.h:56
uint32_t stragtegy_{0};
```

应为 "strategy_"。该拼写错误贯穿整个MissionMgr实现(8处引用)，
说明代码审查可能没有覆盖此文件。

#### AP-CCU-5: hardcoded "恰好2个die" 假设

```cpp
// ccu_comp.cc:485
const uint32_t dstDieId = 1 - dieId; // 当前仅存在最多两个die
```

虽然有注释说明假设前提，但如果未来硬件扩展到3+个die，此逻辑将静默地产生错误结果。
更安全的做法是使用显式查表或断言 CCU_MAX_IODIE_NUM == 2。

#### AP-CCU-6: TODO注释未清理

4处TODO(仅CCU模块):
```
ccu_kernel_mgr.cc:149: // todo: 优化为遍历数组
ccu_kernel_mgr.cc:219: // todo: 建议改成dieId
ccu_assist.cc:13:       // todo: 需要统一整改为不抛异常
ccu_channel_ctx_pool.cc:140: // todo: 需要检查资源管理是否存在泄露可能
```

其中 ccu_channel_ctx_pool.cc 的TODO暗示存在资源泄漏风险。

#### AP-CCU-7: V1/V2微码重复代码

ccu_microcode.cc(V1, ~500行) 和 ccu_microcode_v2_.cc(V2, ~426行) 存在大量结构重复:
- 相同的指令类型常量定义(LOAD_TYPE/CTRL_TYPE/TRANS_TYPE/REDUCE_TYPE)
- 相似的函数签名和实现模式
- 差异仅在于struct字段名(v1.xxx vs v2.xxx)

应通过模板或代码生成减少重复。

#### AP-CCU-8: CcuKernelMgr::Init()被注释掉

```cpp
// ccu_drv_handle.cc:62 (根据前轮分析记录)
// CcuKernelMgr::GetInstance(devLogicId_).Init();  // 被注释掉
```

CcuKernelMgr的Init在驱动初始化流程中被跳过，
意味着kernel管理器可能在未初始化的状态下被使用，或者初始化在其他地方延迟发生。

#### AP-CCU-9: DoReleaseNonBlockTypeRes中的erase-in-loop

```cpp
// ccu_res_batch_allocator.cc:595
for (uint32_t i = 0; i < reqSize; i++) {
    // ...
    resInfos.erase(resInfos.begin() + i);  // erase后i递增跳过下一个元素
}
```

在遍历vector时erase元素但没有调整索引，会导致跳过erase后紧邻的元素。
这是一个经典的off-by-one bug。

---

### 9. 硬件抽象

#### 9.1 CCU硬件版本

当前支持: CcuVersion::CCU_V1(唯一)
CCU_V2已在微码层有实现(ccu_microcode_v2_.cc)，但设备层尚未接入(CheckCcuVersion硬编码返回V1)。

V1→V2差异:
- V1: GSA(地址寄存器) + XN(变量寄存器) 分离
- V2: 统一X寄存器 + 新增ALU指令集(Sub/Mul/And/Or/Not/Xor/Shl/Shr/Popcnt) + CacheConfig

#### 9.2 主板形态适配

CcuResSpecifications::Init() 通过 HccpMacGetMainboardId() 检测主板形态:
```
HcclMainboardId: POD / A_K_SERVER / A_X_SERVER / PCIE_STD / EQUIPMENT / EVB
```

A+X Server(armX86Flag_=true): 特殊处理
- IOdie0的CCUA0不可用(PCIe占用)，分配MS时跳过
- MS交织粒度使用 MSID_CONFIG_ARMX86_MAINBOARD = 6 (其他形态用0)

#### 9.3 TLV驱动通信协议

CCU通过TLV(Type-Length-Value)协议与驱动交互:

```
CcuOpcodeType (20+操作码):
- Kernel操作(0-100): SET_INSTRUCTION, SET_SQE, SET_CCU_CONFIG, SQE_DOORBELL, SET_LOOP_PARAM, ...
- 资源空间操作(200-300): READ_RES, WRITE_RES, READ_CAP, ...
- 管理操作: CCU_INIT, CCU_QUERY
```

每次操作通过CcuData联合体(2048字节buffer)传递数据。

#### 9.4 SQE任务参数格式

GeneTaskParam (ccu_kernel.cc):
- 每个SQE最多携带 CCU_SQE_ARGS_LEN 个参数
- 超过时拆分为多个SQE(每个SQE只存部分参数)
- SQE字段: missionKey, startInstId, instNum, argNum, args[]

#### 9.5 Token安全机制

CCU使用Token机制保护内存访问:
- TokenId(20位) + TokenValue(32位) + TokenValid(1位)
- 每次RDMA传输指令中携带源/目的Token
- Token通过 RtsUbDevQueryInfo(QUERY_PROCESS_TOKEN) 查询runtime获取

---

### 10. 模式量化统计

| 模式 | 出现次数 | 覆盖文件数 | 说明 |
|------|---------|-----------|------|
| CHK_RET | 214 | 19 | 错误检查宏 |
| THROW | 86 | 32 | 异常抛出(rep/microcode层) |
| GetInstance | 77 | 20 | 单例访问 |
| (void)忽略返回值 | 44 | 17 | 释放流程不中断 |
| reinterpret_cast | 33 | 10 | 类型强转(多为void*) |
| EXCEPTION_HANDLE | 24 | 6 | 异常→错误码桥接 |
| dynamic_cast | 13 | 5 | CcuUrmaChannel获取(+kernel) |
| MAKE_ENUM | 9 | 5 | 类型安全枚举 |
| TODO/todo | 4 | 3 | 待处理项 |
| CcuRepBase子类 | 26 | 26 | 表示层指令集 |
| V1微码操作 | ~30 | 1 | ccu_microcode.cc |
| V2微码操作 | ~25 | 1 | ccu_microcode_v2_.cc |
| 相对路径include | 10 | 5 | 跨模块依赖 |

---

### Notable Patterns

#### NP-1: 编译器式架构

CCU模块的整体设计是一个微型编译器:
- 前端(CcuKernel API) → IR(CcuRepBase hierarchy) → 后端(ccu_microcode) → 目标码(CcuInstr)
- CcuRepTranslator是"代码生成器"，有4阶段翻译pass
- CcuRepReferenceManager是"符号表"，管理FuncBlock/LoopBlock的名称解析
- CcuRepContext是"编译上下文"，维护当前activeBlock(类似编译器的scope栈)

#### NP-2: 异常vs错误码的清晰分界

CCU模块是hcomm中少有的异常使用密集区:
- Device/Transport层: 纯错误码(CHK_RET + HcclResult)
- Representation/Microcode层: 纯异常(Hccl::THROW + CcuApiException)
- 桥接点: EXCEPTION_HANDLE_BEGIN/END (Transport和KernelMgr中使用)

这种分界暗示: representation/microcode层可能最初是独立开发的，
后来集成到hcomm的错误码体系时通过桥接宏适配。

#### NP-3: 资源分配的三层抽象

资源从硬件到用户经过3层:
1. 硬件层: 能力寄存器 → CcuResSpecifications(查询每die的各类资源总量)
2. ID分配层: CcuResIdAllocator(first-fit算法分配资源ID)
3. 批量管理层: CcuResBatchAllocator(block/consecutive/discrete三种策略)

用户不直接操作ID，而是通过CcuResHandle(void*)间接持有资源。

#### NP-4: SQE参数拆分策略

当kernel参数数量超过单个SQE容量时:
```cpp
// ccu_kernel.cc GeneTaskParam
if (argIdx + paraNum > CCU_SQE_ARGS_LEN) {
    // 当前SQE装不下所有参数
    // 拆分: 当前SQE装满 → 下一个SQE继续装
}
```

这是硬件约束驱动的软件设计——CCU SQE有固定的参数槽位数限制。


---

## 2.2.5 next/coll_comms/ — 新框架集合通信管理 (30文件)

### Module Overview

`next/coll_comms/` 是hcomm新框架(代号"A5"/91095芯片)的集合通信域管理模块，30个文件(15 .h + 15 .cc)。
核心职责是为每个通信域提供统一的上下文对象(CollComm)，管理拓扑信息、本rank资源、通信内存和通道创建。

架构位置:
```
hcclComm (framework/communicator, 入口层)
  +-- communicator_ (旧路径: HcclCommunicator, legacy架构, 91092/91093)
  +-- collComm_ (新路径: CollComm, A5架构, 91095)     <-- 本模块
  |     +-- rankgraph_ (RankGraphV2, 拓扑抽象)
  |     +-- myRank_ (MyRank, 本rank资源)
  |     |     +-- commMems_ (CommMems, 通信内存)
  |     |     +-- rankPairMgr_ (RankPairMgr, rank对管理)
  |     |     +-- endpointMgr_ (EndpointMgr, 端点管理, from comms/)
  |     |     +-- ccuResContainer_ (CCU资源)
  |     +-- commEngineResMgr_ (通信引擎资源)
  |     +-- contextMgr_ (上下文管理)
  +-- independentOp_ (旧路径: 独立算子, 91092/91093)
```

新旧分水岭: `IsCommunicatorV2()` — 新路径走CollComm.GetMyRank().CreateChannels()，旧路径走IndependentOp.GetChannelManager()。

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| coll_comm.h/cc | ~200 | CollComm上下文: 聚合5个子管理器的Facade |
| coll_comm_mgr.h/cc | ~50 | 通信域管理器(空壳预留) |
| rank_graphs/rank_graph.h/cc | ~100 | 拓扑抽象基类 + 静态工厂 |
| rank_graphs/rank_graph_v1.h/cc | ~80 | V1拓扑(skeleton) |
| rank_graphs/rank_graph_v2.h/cc | ~80 | V2拓扑(skeleton, 与communicator/的同名类不同!) |
| rank_pairs/rank_pair.h/cc | ~120 | (localRank, remoteRank)通信关系 + EndpointPair |
| rank_pairs/rank_pair_mgr.h/cc | ~100 | 懒创建缓存: RankIdPair→RankPair |
| rank/my_rank.h/cc | ~350 | 核心工作类: 通道创建三阶段 + 资源管理 |
| rank/comm_mems/comm_mems.h/cc | ~400 | 区间冲突检测的内存注册 |
| api_c_adpt/coll_comm_c_adpt.h/cc | ~150 | C API薄适配(大部分空壳) |
| api_c_adpt/coll_comm_res_c_adpt.h/cc | ~400 | 通道获取+ABI兼容+CCU管理(实际工作代码) |
| api_c_adpt/rank_graph_c_adpt.h/cc | ~80 | 拓扑查询C API(空壳) |
| common/loggers/*.h/cc | 6文件 | 诊断日志: channel/endpoint/addr三层Logger |

### Idioms & Conventions

1. 显式析构顺序控制(my_rank.cc:41-48):
```cpp
MyRank::~MyRank() {
    rankPairMgr_ = nullptr;     // 先销毁channel，可能返还endpoint与ccu资源
    endpointMgr_ = nullptr;     // 销毁endpoint，可能返回ccu资源
    ccuResContainer_ = nullptr; // 清理CCU资源，关闭CCU通道
    commMems_ = nullptr;
}
```
当unique_ptr成员有依赖时，手动显式置nullptr控制析构序(不依赖C++默认逆声明序)。

2. 懒创建缓存(RankPairMgr::Get):
```cpp
auto iterPtr = rankPairMap_.find(rankIdPair);
if (iterPtr != rankPairMap_.end()) { out = iterPtr->second.get(); return; }
// 缓存未命中: 创建+Init+emplace
```

3. namespace std中hash特化(rank_pair.h, comm_mems.h):
```cpp
namespace std {
template<> struct hash<RankIdPair> {
    size_t operator()(const RankIdPair& p) const {
        return std::hash<uint64_t>()((uint64_t(p.first) << 32) | uint64_t(p.second));
    }
};
}
```
项目中多处出现的hash注入惯用法。

4. ABI版本兼容(ProcessHcclResPackReq):
header.size比较实现前向/后向兼容; header.magicWord校验; header.version分派;
ROCE参数使用INVALID_UINT/0xFF哨兵值表示"未设置"，运行时从环境变量读默认值。

5. 三阶段建链(MyRank::CreateChannels):
BatchCreateSockets → BatchCreateChannels → BatchConnectChannels(轮询等待)

6. 区间冲突检测(CommMems::CommRegMem):
同tag内先尝试Add，失败则Find判断: 完全重复(幂等复用) vs 部分重叠(冲突报错)。

### Architecture & Patterns

1. CollComm是Facade: 真正工作在MyRank中完成，CollComm聚合5个子管理器但CommMemMgr和ChannelManager未初始化(预留)
2. 保留void* comm_反向引用旧HcclCommunicatorV2 — 新旧架构桥接痕迹
3. 两个同名RankGraphV2共存: next/coll_comms/版(skeleton, string构造) vs communicator/版(工作代码, void*构造)
4. 静态工具类Logger: 构造函数=delete, 全static方法, 无状态

### Anti-Patterns

1. 两个同名RankGraphV2(不同基类、不同API、不同实现) — 新旧架构并存产物，导致代码理解困难
2. rank_graph基类private成员被子类"访问" — skeleton代码可能编译不过
3. CollComm的unique_ptr::get()冗余三元检查(5处): `return ptr_ != nullptr ? ptr_.get() : nullptr`
4. my_rank.h相对路径include: `#include "../../comms/comm_engine_res/ccu/ccu_res_container.h"` (注释说明临时措施)
5. Logger头文件在namespace内用extern "C"包裹class声明(无意义，可能copy-paste)
6. ChannelGetHcclBuffer硬编码vector大小10无溢出检查
7. CommUnregMem中const_cast(注释自己标注"需要增加校验")

### Domain Knowledge

1. 通信协议矩阵: HCCS(片内)/PCIe(节点内)/SIO/UBC_CTP/UB_MEM/ROCE(节点间)
2. 引擎类型: AICPU/AICPU_TS(需kernel launch)/CCU(限3种展开模式)/AIV/CPU
3. 端点位置模型: DEVICE(devPhyId/superDevId/serverIdx/superPodIdx四级) vs HOST(id)
4. 地址类型: IPv4/IPv6/ID(32位设备ID)/EID(128位扩展ID, subnet prefix + interface id)
5. 芯片代际: 91092/91093 → V1旧架构, 91095 → V2新架构(A5)

### Module Maturity

早期开发阶段: CollCommMgr空壳、rank_graphs/为skeleton、C API大部分空壳、多处TODO。
实际工作代码集中在: my_rank.cc(三阶段建链) + comm_mems.cc(内存注册) + coll_comm_res_c_adpt.cc(C API+ABI兼容)。


---

## 2.2.6 device/ + nslbdp/ + cluster_maintenance/ — 设备执行、网络负载均衡、集群维护

### 2.2.6.1 device/ — 设备侧(AICPU)执行框架 (~92文件)

#### Module Overview

device/模块是hcomm在设备侧(AICPU)的执行框架，负责在华为Ascend芯片的AICPU核上运行集合通信算子。
是host侧通信框架在device侧的对应物，通过HDC通道接收指令、调度SQE、执行通信算法。

子目录: framework/(communicator/HDC/process/zero-copy/cache), aicpu_kfc/(算法/调度/RPC),
common/(AicpuComContext/SQE), debug/(profiling/trace), utils/, inc/

#### Architecture & Patterns

1. God Class HcclCommAicpu(~580行头文件, 150+方法, 60+成员): 管理流/链路/算法/重执行/零拷贝/缓存/DFX/独立算子
2. KFC算法层Strategy模式: AicpuAlgorithm基类 → AicpuAllreduce/ReduceScatter/Allgather/AllToAll/DmyCalAllreduce
3. 三层分发: AicpuKfcProcess(入口) → AicpuKfcRpcServer(消息解析) → TaskOrchestrator(SQE编排) → AicpuDispatcher(底层操作)
4. 重执行FSM(10态): INIT→LAUNCH→WAIT_END→STOPPING→STOPPED→CHANGE_LINK→WAIT_RETRY→RETRY→END/ERROR
5. extern "C" + visibility("default") ABI导出: 设备侧动态加载标准接口
6. RAII状态管理(KfcState): 构造/析构自动管理isRunning标志

#### Idioms

1. CHK_*宏 1151次/36文件 — 严格"每次调用必检查"
2. 全局Context: AicpuComContext(~80字段大struct), AicpuUpdatComContextMumber通过offset+reinterpret_cast批量更新
3. volatile+内存屏障: FlagData的volatile成员+MemFence(), 注释解释字段顺序约束(flag必须最后以配合SDMA)
4. __attribute__((weak))延迟绑定: profiling等可选功能
5. CCL_LLT条件编译(5处): 区分本地测试和真实设备环境

#### Hardware Abstractions

- AICPU核: 8核/cluster x 2 cluster, 最多8核并行
- AIV核: 最多64个
- SQE: 64字节/条, 每流最多2048条
- 通信算法: one-shot(1/4/8流)/two-shot(8流)/HD/单环, 按数据量选择
- AC_MAX_RANK_NUM=32(单机最多32卡)
- MC2融合: 矩阵计算+通信同时进行(inputA→outputC+commOut)
- Master-Slave架构: root核编排, 其他核执行BatchWrite

#### Anti-Patterns

1. God Class: HcclCommAicpu
2. AicpuComContext: 80字段大struct未封装，reinterpret_cast<T*>按offset批量更新(类型安全极差)
3. 拼写错误散布: AicpuUpdatComContextMumber/CommandToBackGroud/directlySendMainSteramSqe/notift
4. yxg-debug注释残留(aicpu_zero_copy_exchanger.h:37)
5. TODO 9处/4文件(aicpu_cache_manager.cc有6处)

### 2.2.6.2 nslbdp/ — 网络服务负载均衡数据面 (5文件)

#### Module Overview

NSLB(Network Service Load Balance) DP模块，向网络管理平面上报集合通信域的拓扑信息、算子信息、算法邻接表，
供网络侧做负载均衡决策。通过TLV序列化+H2D通道发送到网络协处理器(NetCo)。

#### Key Design

1. 六张信息表: CommConfig/Operator/Algorithm/GlobalRank/GlobalDisRank/RootRank
2. Singleton: hcclNslbDp::GetInstance()
3. TLV序列化 + 分包发送(NSLBDP_RANKTOTALNUM_BLOCK_FIR=1024阈值)
4. MD5校验: 通信域信息变更检测

#### Anti-Patterns

1. 魔法数字机械命名: NSLB_MD5_4=4, NSLB_MD5_5=5...NSLB_MD5_25=25 — 无语义信息
2. 类名hcclNslbDp首字母小写，违反项目PascalCase规范
3. 大量public成员变量直接暴露

### 2.2.6.3 cluster_maintenance/ — 集群维护 (~22文件, 4子功能)

#### Module Overview

四个子功能: detect/(连接异常检测), health/heartbeat/(心跳), recovery/operator_retry/(算子重执行), snapshot/(快照)

#### Architecture & Patterns

1. 心跳机制(Heartbeat, Singleton per-device):
   - 50ms间隔广播心跳帧
   - 支持7种异常状态(OK/LOST/NOTIFY/CQE_ERR/OPRETRY_NOT_SUPPORT/STUCK/INCONSISTENT)
   - 5分钟stuck检测
   - 支持算子一致性检查(跨rank验证op类型/数据类型/reduce类型/root/count)
   - ReferenceMap: 引用计数管理多通信域共享rank连接

2. 算子重执行(OpRetry, Server-Agent双状态机):
   - Server(root rank) 22态状态机, Agent(non-root)对称执行
   - State模式: 每个状态一个派生类, RetryContext持有当前状态指针
   - 完整流程: 停AICPU→停流→清流→停transport→检查链路→恢复transport→重置notify→retry
   - 最多重试3次(OP_RETRY_MAX_CNT=3)
   - 支持NIC切换(主备链路)

3. 连接异常检测(DetectConnectionAnomalies, Singleton):
   vnic/nic socket探测网络连通性, 异步线程模型, 白名单(每IP最大16连接)

4. 快照控制(SnapshotControl, Singleton):
   4态: DEFAULT→PRE_SNAPSHOT→POST_SNAPSHOT→RESTORE_SNAPSHOT, 快照期间暂停心跳

#### Idioms

1. Per-Device Singleton: 所有单例以deviceLogicID为参数 — 一个进程管理多设备
2. 状态→字符串映射表: 所有enum配std::map<State, string> + GetXxxStr()
3. CHK_*宏 370次/12文件
4. RingBuffer: 手写环形缓冲区(new/delete管理)

#### Anti-Patterns

1. HeartBeatFrame和HeartBeatFrameWithOpCheck结构体代码重复(应继承或组合复用)
2. RetryState用typedef enum而非enum class(与device/模块的enum class不一致)
3. const map在头文件中非inline定义 — 每个翻译单元生成副本
4. TODO残留(detect_connect_anomalies.h:24)

---

## 2.2.7 Framework层综合分析

本节综合2.2.1-2.2.6全部分析，提取Framework层(含legacy entrance和next新框架)的
跨子模块惯用法、架构模式、业务领域知识、硬件抽象知识和反模式清单。

分析覆盖范围:
- framework/communicator/ (125文件, 19子目录)
- framework/op_base/ (7文件)
- framework/hcom/ (13文件)
- framework/group/ (3文件)
- framework/device/ (~92文件)
- framework/nslbdp/ (5文件)
- framework/cluster_maintenance/ (~22文件)
- framework/next/comms/ (endpoints/endpoint_pairs/channels/coll_comms/CCU等, 200+文件)
- legacy/framework/communicator/ (59文件)
- legacy/entrance/ (op_base_v2 + hcom_v2)

### 一、跨子模块惯用法 (Cross-Module Idioms)

#### FW-ID-1: CHK_RET 链式错误检查 [项目级强制]

量化: framework/ 4357次/152文件 + legacy/ 451次/28文件 + next/ 425次/48文件
总计 5233次/228文件，覆盖率接近100%。

每个可失败调用的返回值都用CHK_RET检查:
```cpp
CHK_RET(Init());
CHK_RET(CreateResource());
CHK_RET(Connect());
```

变体:
- CHK_RET: 基础版，失败return错误码
- CHK_RET_AND_PRINT_IDE: 失败额外打印tag供IDE展示 (op_base.cc 607处)
- CHK_PRT_RET: 失败打印自定义信息后return
- CHK_PRT_BREAK: 失败打印+break(do-while清理模式)
- CHK_PTR_NULL: 空指针检查特化版

唯一例外: CCU representation/microcode层用THROW(见FW-ID-10)。

#### FW-ID-2: RPT_INPUT_ERR + CHK_PTR_NULL 配对 [API入口层强制]

量化: 282次/26文件

对外API的每个参数校验是双重的:
```cpp
RPT_INPUT_ERR(comm == nullptr, "EI0003",
    std::vector<std::string>({"ccl_op", "parameter", "value", "tips"}),
    std::vector<std::string>({"HcclAllReduceInner", "comm", "nullptr", "please check comm"}));
CHK_PTR_NULL(comm);
```
RPT_INPUT_ERR面向用户(IDE错误中心)，CHK_PTR_NULL负责控制流。

#### FW-ID-3: C ABI 门面模式 [项目级强制]

所有对外接口都是extern "C"函数，内部通过类型转换恢复C++对象:
```cpp
extern "C" HcclResult HcclAllReduceInner(..., HcclComm comm, ...) {
    hccl::hcclComm *hcclComm = static_cast<hccl::hcclComm *>(comm);
    // ...
}
```

出现位置:
- op_base.cc: 40+ extern "C" 函数
- hcom.cc: 50+ extern "C" 函数
- legacy/entrance/: 85 + 53 个C函数
- next/ api_c_adpt/: Handle-based C API (4个全局static map)
- device/: extern "C" + visibility("default") 内核入口

#### FW-ID-4: HCCLV2_FUNC_RUN V1/V2 分派 [framework层强制]

量化: 145次/11文件 (仅framework + legacy，next/ 0次)

```cpp
HCCLV2_FUNC_RUN(HcclAllReduceV2(sendBuf, recvBuf, count, dataType, op, comm, stream));
```
宏检查SoC类型(IsSupportHCCLV2)，条件满足则直接调用V2函数并return。
V2函数通过__attribute__((weak))声明，运行时条件链接。

分水岭: IsCommunicatorV2()
- V1(91092/91093): hcclComm → HcclCommunicator → IndependentOp
- V2(91095): 直接HcclCommunicator / CollComm路径

#### FW-ID-5: new(std::nothrow) + CHK_PTR_NULL [推荐]

量化: 127次/41文件

```cpp
auto *ptr = new (std::nothrow) HcclCommunicator(params);
CHK_PTR_NULL(ptr);
```
替代 try/catch(bad_alloc) 的轻量内存分配模式。
但next/部分子模块混用make_unique(见AP-FW-14)。

#### FW-ID-6: EXECEPTION_CATCH 异常捕获 [推荐]

量化: 206次/55文件

包裹所有STL分配操作防止内存异常:
```cpp
EXECEPTION_CATCH((job = std::make_shared<AsyncJob>()), return HCCL_E_PARA);
EXECEPTION_CATCH(opBaseHcom.opGroup2CommMap.erase(group), return HCCL_E_MEMORY);
```

#### FW-ID-7: EXCEPTION_HANDLE_BEGIN/END 异常→错误码桥接 [CCU/next层]

量化: 89次/18文件 (54次在next/)

用于调用可能抛异常的代码(URMA/HCCP API):
```cpp
EXCEPTION_HANDLE_BEGIN
    ccuTransport_.Init();
    ccuConnection_.Connect();
EXCEPTION_HANDLE_END(ret)
```
catch后转为HcclResult错误码。

#### FW-ID-8: Host/Device 编译分裂 [项目级]

同一逻辑有host和device两个编译目标:
- xxx.cc / xxx_host.cc / xxx_device.cc (communicator 6组)
- #ifdef CCL_KERNEL_AICPU 条件编译 (device/ 5处CCL_LLT)
- device stub: 仅打WARNING+return SUCCESS (op_base_device.cc)

GetRunSideIsDevice() 运行时判断当前执行环境。

#### FW-ID-9: unique_ptr + 显式析构顺序 [推荐]

量化: unique_ptr 335次/117文件

资源管理首选unique_ptr，有依赖时手动控制析构序:
```cpp
MyRank::~MyRank() {
    rankPairMgr_ = nullptr;     // 先销毁channel
    endpointMgr_ = nullptr;     // 再销毁endpoint
    ccuResContainer_ = nullptr; // 最后清理CCU资源
    commMems_ = nullptr;
}
```
位置: MyRank、CcuResContainer、CommunicatorImpl析构等。

#### FW-ID-10: 双轨错误处理 — 错误码 vs 异常的清晰分界 [CCU模块]

CCU模块独有:
- device/transport层: 纯CHK_RET错误码 (214次/19文件)
- representation/microcode层: 纯THROW异常 (86次/32文件)
- EXCEPTION_HANDLE桥接两者 (24次/6文件)

ccu_assist.cc TODO: "需要统一整改为不抛异常"

#### FW-ID-11: MAKE_ENUM 类型安全枚举 [next/层推荐]

量化: 19次/11文件 (仅next/层)

```cpp
MAKE_ENUM(ChannelStatus, INIT, SOCKET_OK, SOCKET_TIMEOUT, READY, FAILED)
```
提供: Describe()字符串化 + operator<<日志输出 + EnumClassHash。
覆盖所有状态机枚举(ChannelStatus, CcuConnStatus, TransStatus, RdmaStatus等)。

#### FW-ID-12: Entry/Exit 日志对 + TIME_NOW() 计时 [API入口层强制]

所有API入口函数的标准日志模式:
```cpp
HcclUs startut = TIME_NOW();
HCCL_RUN_INFO("Entry-%s: count[%llu] ...", __func__, count, ...);
// ... operation ...
HCCL_RUN_INFO("[HCCL_TRACE]%s success, take time [%lld]us", __func__, DURATION_US(TIME_NOW()-startut));
```

变体:
- op_base.cc: GetExternalInputHcclEnableEntryLog()条件日志 + stackLogBuffer (16处)
- hcom.cc: HCCL_RUN_INFO直接日志 (更简洁)
- legacy/: 条件日志模式(同op_base)

#### FW-ID-13: (void)忽略返回值 [析构/释放路径]

析构和释放路径中吞掉错误，确保清理不中断:
```cpp
(void)ReleaseTransRes();
(void)ccuResSepcs.GetLoopEngineNum(dieId, loopNum);
```
位置: CCU 44次/17文件，communicator析构路径，resource释放路径。

#### FW-ID-14: BinaryStream序列化 [跨子模块]

多处使用BinaryStream做POD结构体二进制序列化:
- CCU Transport: ConnectionInfo/TransRes/CclBufferInfo
- H2D资源传递: Channel资源序列化→H2D memcpy→Device反序列化
- Snapshot: 通信域状态序列化+CRC32校验

特征: 无版本号、无自描述能力，收发必须按相同顺序pack/unpack。

#### FW-ID-15: Per-Device Singleton [推荐]

```cpp
static XxxManager instances[MAX_MODULE_DEVICE_NUM + 1];
static XxxManager& GetInstance(uint32_t deviceLogicId) {
    return instances[deviceLogicId];
}
```

出现: 54次/54文件。位置: CCU(5个类)、Heartbeat、OpRetry、DetectConnectionAnomalies、
SnapshotControl、PreemptPortManager、hcclNslbDp等。

#### FW-ID-16: do-while(0) + errorFlag 清理模式 [通信域初始化]

```cpp
bool errorFlag = false;
do {
    ret = step1(); CHK_PRT_BREAK(ret != HCCL_SUCCESS, ..., errorFlag = true);
    ret = step2(); CHK_PRT_BREAK(ret != HCCL_SUCCESS, ..., errorFlag = true);
} while (0);
if (errorFlag) { (void)HcclCommDestroy(pComm.get()); return ret; }
```
模拟goto清理的C++惯用法。位置: InitCommClusterInfo(13步)、InitCommRootInfo等。

#### FW-ID-17: Packed Struct 硬件寄存器映射 [CCU/device]

```cpp
#pragma pack(push, 1)
struct ChannelCtxDataV1 {  // 64字节，精确匹配硬件布局
    uint32_t sqStartAddr : 20;
    uint32_t pi : 12;
    // ...
};
#pragma pack(pop)
```
CCU大量使用，device/的SQE格式也遵循此模式。

### 二、架构模式 (Architecture Patterns)

#### FW-AP-1: 三层委托链 + God Class中心

Framework层的核心调用链:
```
API入口(op_base/hcom) → hcclComm(句柄) → HcclCommunicator(核心) → 子模块
```

HcclCommunicator是God Class:
- 100+方法、100+成员、9152行host实现
- 持有10+个unique_ptr子系统
- Legacy的CommunicatorImpl同样是God Object(3789行)

这是Framework层最突出的架构特征和最大的技术债务。

#### FW-AP-2: Facade + Composition (聚合优于继承)

Framework层广泛使用Facade聚合而非深度继承:

| Facade | 聚合的子系统 | 位置 |
|--------|-------------|------|
| HcclCommunicator | 10+子系统(IndependentOp/ResourceMgr/OneSided/...) | communicator/ |
| IndependentOp | 5个Manager(RegMem/CommMem/EngineRes/Context/Channel) | communicator/independent_op/ |
| CollComm | 5个子管理器(MyRank/RankGraph/EngineResMgr/ContextMgr/CommEngineResMgr) | next/coll_comms/ |
| HcclCommAicpu | 60+成员(流/链路/算法/重执行/零拷贝/缓存/DFX) | device/ |

#### FW-AP-3: C API表面 + C++内部实现

对外暴露纯C接口，内部是C++:
- HcclComm = void* 是opaque handle
- 对外: extern "C" 函数群
- 内部: static_cast/reinterpret_cast恢复类型

三套并行C API表面:
1. op_base (NCCL风格): HcclAllReduce/HcclBroadcast/...
2. hcom (HCOM风格): HcomAllReduce/HcomBroadcast/... + Graph模式
3. next/ api_c_adpt (Handle-based): HcommChannelCreate/HcommEndpointCreate/...

#### FW-AP-4: V1/V2 双代架构并存

| 维度 | V1 (91092/91093) | V2 (91095/A5) |
|------|-----------------|---------------|
| 通信域 | hcclComm → HcclCommunicator → IndependentOp | HcclCommunicator → CollComm |
| 拓扑 | RankGraphV1(706行完整拓扑计算) | RankGraphV2(pImpl委托IRankGraph) |
| API入口 | op_base.cc通过hcclComm | op_base_v2通过V2 weak symbols |
| 资源管理 | ChannelManager + CommMemMgr | MyRank + CommMems |
| 建链 | 单步建链 | 三阶段(Socket→Channel→Connect) |
| CCU | 无 | 完整5层CCU软件栈 |
| Snapshot | 无 | CRIU式checkpoint/restore |

分派机制: HCCLV2_FUNC_RUN宏 + IsCommunicatorV2() + __attribute__((weak))

#### FW-AP-5: 编译器式CCU架构

CCU子模块是一个微型编译器:
```
CcuKernel API (前端) → CcuRepBase IR (26子类) → CcuRepTranslator (代码生成) → CcuInstr (目标码)
```
- CcuRepContext = 编译上下文(scope栈)
- CcuRepReferenceManager = 符号表
- FuncBlock RAII = scope guard
- 4阶段翻译pass

#### FW-AP-6: Strategy模式 (多处)

| 位置 | Strategy接口 | 具体策略 |
|------|-------------|---------|
| resource_manager/ | MulQpInfo | 4个backend按优先级选配置 |
| device/aicpu_kfc/ | AicpuAlgorithm | AllReduce/ReduceScatter/AllGather/AllToAll/DmyCalAllreduce |
| communicator/ | AcceleratorState | 3种CollService(DeviceMode/AiCpuImpl/DefaultImpl) |
| CCU resource/ | 分配策略 | block/consecutive/discrete |
| next/channels/ | Channel | 5种通道(CPU_RoCE/AICPU_TS_URMA/AIV_UB_MEM/CCU_URMA + Host) |

#### FW-AP-7: 状态机驱动

Framework层有多个显式状态机:

| 状态机 | 状态数 | 位置 |
|--------|--------|------|
| CommStatus | 4态(IDLE→READY→INUSE→READY) | communicator/ |
| RDMA Connection | 6态 | next/channels/host/ |
| CCU Transport | 12态 | next/ccu/transport/ |
| CCU Connection | 6态 | next/ccu/conn/ |
| OpRetry Server | 22态 | cluster_maintenance/ |
| Snapshot | 4态 | cluster_maintenance/ |
| 重执行FSM | 10态 | device/ |

MAKE_ENUM为状态机提供类型安全和字符串化支持。

#### FW-AP-8: 三代实现并存

Legacy(V1) / Framework(V1增强) / Next(V2) 三代架构在Framework层并存:

1. Legacy: communicator_impl(God Object) + HDC/KFC/CCU快加载
2. Framework: HcclCommunicator(另一个God Object) + IndependentOp + 子模块
3. Next: CollComm(Facade) + EndpointPairMgr + CCU 5层栈

Next层大量通过深层相对路径(6层../)引用Legacy代码，说明尚未解耦。

#### FW-AP-9: H2D资源序列化桥接

Host侧创建资源 → BinaryStream序列化 → H2D memcpy → Device侧反序列化初始化

出现:
- Legacy CommunicatorImplLite: 7类资源打包传递
- next/channels/: Channel资源 → AICPU kernel Init
- next/comm_engine_res/: Thread HostInit → UniqueId → DeviceInit

#### FW-AP-10: API入口函数固定模板 (13步)

所有集合通信API入口遵循高度统一的13步模板:

1. Group检查(hcclGroupDepth > 0 → 延迟执行)
2. 计时 + Capture检测
3. Zero-count early return
4. RPT_INPUT_ERR + CHK_PTR_NULL 参数校验
5. HCCLV2_FUNC_RUN V2分派
6. operatorlock_ + StateGuard(INUSE)
7. Tag构造("算子名_" + identifier)
8. 算子特定校验(HcomCheckOpParam等)
9. 条件Entry日志
10. SetWorkflowMode + PrintMemoryAttr
11. 委托hcclComm->XxxOutPlace
12. Profiling上报
13. 条件Exit日志

op_base.cc中13个集合通信函数各100-150行，模板几乎相同但未共享代码。

### 三、业务领域知识 (Domain Knowledge)

#### FW-DK-1: 通信域层次

```
World Group (全局通信域)
  ├── Sub Group (子通信域, SubComm)
  │     └── 通过HcclCreateSubCommConfigInner创建
  ├── One-Sided Group (单边通信域)
  │     └── Put/Get语义, fullmesh连接
  └── Group (NCCL语义)
        └── GroupStart/End批量调度
```

每个通信域核心属性:
- groupName: 唯一标识
- rank/rankSize: 本rank编号和总数
- tag: "算子名_" + identifier，用于资源复用
- comm对象: HcclCommunicator(V1) 或 CollComm(V2)

#### FW-DK-2: 两套API风格

| 维度 | op_base (NCCL风格) | hcom (HCOM风格) |
|------|-------------------|----------------|
| 通信域标识 | HcclComm comm (void*) | const char *group (字符串) |
| 执行方式 | LoadOpbasedCollOp (直接执行) | LoadOffloadCollOp (卸载执行) |
| 额外功能 | MC2/Snapshot/RankGraph | Graph模式/算法选择/梯度切分 |
| 使用方 | PyTorch (单算子路径) | GE/MindSpore (图路径) |

#### FW-DK-3: 集合通信操作矩阵

全量支持: AllReduce/AllGather/ReduceScatter/Broadcast/Reduce/Scatter/
AlltoAll/AlltoAllV/AlltoAllVC/Send/Recv/BatchSendRecv/Barrier

特殊实现:
- Barrier = AllReduce on 1 element (FP32 SUM)
- ReduceScatter/Reduce 在数据量超CCL buffer时自动分片循环
- AlltoAllV 有Staged版本(工作空间预计算)
- GatherAlltoAllV 使用16线程并行内存拷贝

#### FW-DK-4: 通信域初始化两大路径

```
ClusterInfo路径: 静态rankTable → 所有rank信息已知
  变体: ClusterInfo / ClusterInfoConfig / ClusterInfoMemConfig

RootInfo路径: 分布式发现
  32k rank阈值: 以上自动切换分层拓扑检测
  分层: Agent→Root→GroupLeader→Member
```

两条路径都收敛到pComm->init()，后接8步配置序列。

#### FW-DK-5: 通信协议矩阵

| 协议 | 含义 | 场景 |
|------|------|------|
| HCCS | 高速芯片互联 | 片内/节点内(未完全实现) |
| SIO | Serial IO | 节点内PCIe替代 |
| PCIe | PCIe总线 | 节点内 |
| RoCE | RDMA over Converged Ethernet | 节点间(Host CPU发起) |
| UBC_TP | URMA基础通信-TP模式 | 节点间(Device发起) |
| UBC_CTP | URMA基础通信-CTP模式 | 节点间(Device发起，更高效) |
| UB_MEM | UB共享内存 | 设备内/设备间共享 |
| TCP | TCP socket | 控制面/fallback |

#### FW-DK-6: 通信引擎矩阵

| 引擎 | 含义 | 使用场景 |
|------|------|---------|
| CPU/CPU_TS | Host CPU (Task Scheduler) | Host侧RDMA操作 |
| AICPU/AICPU_TS | AI CPU | Device侧调度(8核/cluster x 2cluster) |
| AIV | AI Vector | 向量处理器(未完全支持) |
| CCU | Communication Control Unit | 硬件通信加速 |

Engine x Protocol 矩阵实际实现: CPU_RoCE / AICPU_TS_URMA / AIV_UB_MEM / CCU_URMA
其余组合为空壳(扩展预留)。

#### FW-DK-7: Group异步语义 (NCCL-style)

```cpp
HcclGroupStart();
// 以下操作延迟执行，不立即通信
HcclAllReduce(comm1, ...);  // → taskAppend
HcclSend(comm2, ...);       // → taskAppend
HcclGroupEnd();              // → KernelPlanner调度 → 批量执行
```
- hcclGroupDepth > 0 时API只做taskAppend
- 支持嵌套(depth递增)
- KernelPlanner调度P2P和集合通信
- Init/Destroy也支持Group异步(AsyncJob+Wrapper)

#### FW-DK-8: 故障恢复机制

心跳检测(50ms间隔) → 异常分类(7种状态) →
算子重执行(OpRetry, Server-Agent双状态机, 最多3次) →
流程: 停AICPU→停流→清流→停transport→检查链路→恢复→重试

NIC切换(主备链路): DiffRankUpdater处理rank表更新
CRIU快照(V2): SaveSnapshot(PRE/POST两阶段) → RestoreSnapshot

#### FW-DK-9: MC2融合算子

MC2(Multi-Chip 2)是AICPU融合算子的资源预分配机制:
- tiling二进制数据: version + count + ServerCfg + N个HcommCfg
- 批量预分配通信资源(buffer/stream/notify)
- 运行时: 矩阵计算 + 通信同时进行 (inputA→outputC+commOut)
- 仅V2路径有实际实现，V1为空函数

### 四、硬件抽象知识 (Hardware Abstractions)

#### FW-HW-1: 芯片代际演进

| 代际 | SoC | 架构代号 | Framework路径 |
|------|-----|---------|--------------|
| 传统 | 910B/910_93 | A3 | V1: HcclCommunicator + IndependentOp |
| 当前 | 910_95 | A5 | V2: CollComm + CCU + Snapshot |
| 未来 | (950/910_96) | - | UDMA协议预留 |

A3 vs A5的差异贯穿所有子系统:
- Stream: 传统Stream+SQ/CQ直操作 vs StreamLite(rtsq)
- Notify: Notify IPC vs NotifyLite
- CCU: 无 vs 完整5层软件栈
- RankGraph: V1自主拓扑计算 vs V2 pImpl委托
- RDMA: IBVerbs vs URMA

#### FW-HW-2: 拓扑层次与链路选择

RankGraph V1编码了完整的华为芯片网络拓扑知识:
- L0 (片内): HCCS
- L1 (节点内): PCIe / SIO
- L2 (节点间): RoCE

910B vs 910_93 vs 310P各有不同的L0-L2协议映射。
链路类型决定Transport选择: HCCS→P2P, RoCE→IBVerbs/TCP, PCIe→DirectNpu。

#### FW-HW-3: AICPU执行模型

- 8核/cluster x 2 cluster, 最多8核并行
- AIV: 最多64个向量处理核
- SQE: 64字节/条, 每流最多2048条
- Master-Slave架构: root核编排, 其他核执行BatchWrite
- AC_MAX_RANK_NUM=32 (单机最多32卡)

#### FW-HW-4: CCU硬件模型

- IO Die双die架构, A+X Server形态CCUA0被PCIe占用
- V1: GSA(地址寄存器) + XN(变量寄存器)分离
- V2: 统一X寄存器 + 扩展ALU指令集
- Token安全: TokenId(20位) + TokenValue(32位) + TokenValid(1位)
- SQE参数槽位有限，超过时拆分多个SQE
- 资源: Loop引擎 + MS微服务引擎 + 3种分配策略(block/consecutive/discrete)

#### FW-HW-5: 主板形态适配

HccpMacGetMainboardId()检测: POD/A_K_SERVER/A_X_SERVER/PCIE_STD/EQUIPMENT/EVB
- A+X Server: IOdie0的CCUA0不可用(PCIe占用), MS交织粒度特殊
- 不同形态影响CCU资源分配策略

#### FW-HW-6: NIC部署模型

- Device侧NIC(default): 设备网卡, NicType::DEVICE_NIC_TYPE
- Host侧NIC: 主机网卡, NicType::HOST_NIC_TYPE
- 影响: 端口选择、IP选择、Socket创建方式
- 910_95的Host CPU RoCE通道是专用功能

#### FW-HW-7: 硬件限制常量

| 常量 | 值 | 含义 |
|------|----|------|
| SDMA 4GB切分 | 4GB | 大数据传输必须按4GB分段(platform层) |
| CCU资源固定 | 16 | 每Channel固定16个CKE资源 |
| AC_MAX_RANK_NUM | 32 | 单机最多32卡 |
| SQE深度 | 2048条/流 | AICPU SQE队列深度 |
| SQE大小 | 64字节 | 单条SQE |
| CCU CcuData | 2048字节 | 驱动通信buffer |
| 心跳间隔 | 50ms | 心跳广播周期 |
| OpRetry上限 | 3次 | 最多重试次数 |
| 分层拓扑阈值 | 32k rank | 超过切换分层发现 |

### 五、反模式清单 (Anti-Patterns)

#### AP-FW-1: God Class [影响: 可维护性] [严重]

| 文件 | 行数 | 方法数 | 成员数 |
|------|------|--------|--------|
| hccl_communicator_host.cc | 9152 | 100+ | 100+ |
| op_base.cc | 4695 | 40+ | - |
| hcom.cc | 4076 | 50+ | - |
| communicator_impl.cc (legacy) | 3789 | - | 20+ unique_ptr |
| HcclCommAicpu (device/) | ~580行头文件 | 150+ | 60+ |

这是Framework层最普遍的反模式。已尝试提取(IndependentOp, Attrs)但核心仍高度耦合。

#### AP-FW-2: 代码重复 [影响: 可维护性] [严重]

多处大规模代码重复:
- op_base.cc: 13个集合通信API各100-150行，模板几乎相同未共享
- 8步配置序列重复3次(每次~60行): InitCommClusterInfo/InitCommRootInfo/SubComm
- 通信域初始化3处重复(~30行): CreateCommConfig/CreateCommConfigRootInfo/ClusterInfo
- Wrapper函数完整复制主函数逻辑(仅多hrtSetDevice)
- HeartBeatFrame/HeartBeatFrameWithOpCheck结构体重复
- CCU V1/V2微码大量结构重复

#### AP-FW-3: 全局可变状态 + 锁缺失 [影响: 线程安全] [严重]

| 全局状态 | 位置 | 锁保护 |
|---------|------|--------|
| g_oneSidedCommHcomInfos/g_oneSidedCommSet | op_base.cc | 部分操作无锁 |
| g_ThreadMap | hcomm_c_adpt.cc | 无锁 |
| serverSocketMap(函数static) | cpu_roce_endpoint.cc | 无锁 |
| hcclGroupDepth + g_groupInfo | hccl_group.cc | 无锁 |
| thread_local g_hcclDeviceId(27次引用) | op_base.cc | thread_local |
| 6个全局static map | next/api_c_adpt/ | 部分有锁 |
| g_callBackResult | task/(platform层遗留) | 无锁 |

#### AP-FW-4: reinterpret_cast 无类型安全 [影响: 类型安全] [普遍]

量化: 1376次/183文件

主要场景:
- void* handle → 具体类型 (C API入口)
- uint64_t → 指针 (Thread handle)
- AicpuComContext offset + reinterpret_cast 批量更新(类型安全极差)
- double-cast: static_cast<Channel*>(void*) → dynamic_cast<CcuUrmaChannel*> (13处)

无magic number或类型标签验证。

#### AP-FW-5: Copy-paste日志/代码错误 [影响: 正确性]

- op_base_v2.cc: 5处结束日志都写成"HcclAllGatherVV2"(应为各自函数名)
- cpu_ts_thread.cc: "[AicpuTsThread]"日志标签(应为"[CpuTsThread]")
- nslbdp/: 魔法数字机械命名 NSLB_MD5_4=4...NSLB_MD5_25=25
- CCU: 变量拼写 "stragtegy_" (贯穿8处引用)
- device/: AicpuUpdatComContextMumber/CommandToBackGroud/directlySendMainSteramSqe

#### AP-FW-6: switch缺失break (fall-through) [影响: 功能bug]

两处确认的fall-through bug:
1. hccl_group.cc:383 — REDUCE_SCATTER缺break，fall-through到ALLREDUCE
2. host_cpu_roce_channel.cc:138 — DATA_EXCHANGE fall-through跳过Prepare Rqe

#### AP-FW-7: null检查/资源管理错误 [影响: 功能bug]

- HcclGetOpArgsV2: 检查了错误变量(opArgs而非opArgsMem) — malloc失败时空指针解引用
- HcclFreeOpArgsV2: 形参赋null不影响调用方 — 经典C错误
- ContextManager析构不释放context — 泄漏风险
- DispatcherCtx创建不注册到全局map — 泄漏风险(platform层)
- KernelMgr::Init()被注释掉 — 可能未初始化使用
- CCU erase-in-loop: vector遍历中erase后索引未调整 — off-by-one bug

#### AP-FW-8: 忙等待无退避 [影响: 性能]

多处pure busy-wait:
- hccl_group.cc:180: usleep轮询
- symmetric_memory_agent.cc: SaluSleep spin-wait
- host_cpu_roce_channel.cc: ibv_poll_cq忙轮询无退避
- legacy communicator: KFC轮询无sleep/退避
- WaitAllCommReady: 持锁tight loop(10s超时)
- task/(platform层): HostNIC 200us忙轮询

#### AP-FW-9: 深层相对路径include [影响: 可维护性]

next/层大量6层../的相对路径include:
```cpp
#include "../../../../../../legacy/unified_platform/resource/socket/socket.h"
```
10+处/5种模式。暴露模块间依赖未通过CMake include path管理。

#### AP-FW-10: 同名类混淆 [影响: 可理解性]

两个RankGraphV2并存:
- next/coll_comms/rank_graphs/rank_graph_v2.h (skeleton, string构造)
- communicator/independent_op/rank_graph_v2.h (工作代码, void*构造)
不同基类、不同API、不同实现。

#### AP-FW-11: getenv热路径调用 [影响: 性能+线程安全]

```cpp
const char *indOp = getenv("HCCL_INDEPENDENT_OP");  // 每次API调用时
```
op_base.cc 6处 + independent_op多处。getenv不是线程安全的(POSIX)。
应在初始化时读取并缓存。

#### AP-FW-12: V2分派位置不一致 [影响: 正确性]

部分API在参数校验前就分派V2(HcclBatchSendRecvInner):
V2函数可能收到nullptr参数。其他API在校验后才分派V2。不一致增加V2实现的防御负担。

#### AP-FW-13: operator<违反strict weak ordering [影响: 理论UB]

zero_copy_address_mgr.h中AddressRange的operator<:
将"重叠"视为"相等"用于set查找(巧妙但违反strict weak ordering要求)。
如果std::set实现依赖strict weak ordering的传递性，可能产生未定义行为。

#### AP-FW-14: new(nothrow) vs make_unique 混用 [影响: 一致性]

channel.cc同一个工厂函数内:
```cpp
case COMM_ENGINE_CPU:
    EXECEPTION_CATCH(channelPtr = std::make_unique<...>(), ...);  // make_unique+catch
case COMM_ENGINE_AICPU_TS:
    channelPtr.reset(new (std::nothrow) AicpuTsUrmaChannel(...)); // new(nothrow)
```
两种分配风格的错误处理路径不一致。

#### AP-FW-15: std::rand()非线程安全 [影响: 线程安全]

CCU模块 ccu_comp.cc:473, ccu_conn.cc:193 使用std::rand()生成PSN初始值。
std::rand()不是线程安全的，多线程初始化场景可能冲突。

#### AP-FW-16: 模块成熟度差异大

| 模块 | 成熟度 | 证据 |
|------|--------|------|
| communicator/ | 高 | 完整实现，1个TODO |
| op_base/ + hcom/ | 高 | 完整实现，少量重复 |
| device/ | 中高 | 完整实现，9个TODO/拼写问题 |
| next/coll_comms/ | 低 | CollCommMgr空壳，rank_graphs skeleton |
| next/channels/ | 中 | 13个TODO，4个空壳类 |
| next/CCU | 中高 | V2实现未接入，4个TODO |
| cluster_maintenance/ | 中高 | 完整4子功能 |

next/层TODO 16个(全framework层17个中占16个)，说明新框架处于活跃开发期。

### 六、Framework层特征总结

| 维度 | 关键特征 |
|------|---------|
| 规模 | ~500+文件，4个God Class(9152/4695/4076/3789行) |
| 错误处理 | CHK_RET 5233次(100%覆盖)，双轨(错误码+异常)仅CCU |
| 类型安全 | reinterpret_cast 1376次(普遍)，void* handle模式 |
| 代码重复 | API模板/配置序列/初始化流程/V1-V2微码 多处大规模重复 |
| 架构演进 | Legacy/Framework/Next 三代并存，V1/V2 运行时分派 |
| 硬件耦合 | A3/A5芯片代际分叉贯穿所有子系统 |
| 成熟度 | 核心模块高成熟度，next/层活跃开发中 |
| 主要债务 | God Class、代码重复、全局状态+锁缺失、reinterpret_cast |
