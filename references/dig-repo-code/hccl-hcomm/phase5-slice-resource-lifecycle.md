# Phase 5.2: Resource Lifecycle (Cross-Repo Slice)

## 5.2.1 通信域创建: HcclCommInit* 的完整跨仓流程

### Overview

通信域创建100%在hcomm仓库中实现。hccl仓库通过`#include <hccl/hccl_comm.h>`引用hcomm的声明，
两仓编译为同一个`libhccl.so`(`target_link_libraries(hccl PRIVATE hcomm)`, hccl/src/CMakeLists.txt:408)。

三种API入口:
- HcclCommInitRootInfo — 基于RootInfo(网络拓扑发现)
- HcclCommInitClusterInfo — 基于RankTable文件(静态配置)
- HcclCommInitAll — 单进程多卡(封装RootInfo模式)

所有API声明带`__attribute__((weak))`(HCOMM_WEAK_SYMBOL)，允许V2实现覆盖。

### API入口层

文件: `hcomm/src/framework/op_base/src/op_base.cc`

三种入口最终都汇聚到相同的初始化核心:

```
HcclCommInitRootInfo(nRanks, rootInfo, rank, comm)          :1679
  |-- Group模式(hcclGroupDepth > 0): 追加到异步任务队列
  |-- 正常模式: HcclCommInitRootInfoInner()                  :1579
        |-- HcclDeviceRefresh()                              # thread_local g_hcclDeviceId
        |-- 参数校验: nRanks>0, rank<nRanks, comm/rootInfo非空
        |-- [V2分派] HCCLV2_FUNC_RUN(HcclCommInitRootInfoV2) # 910_95芯片直接return
        |-- [V1路径]
        |-- InitExternalInput() / InitEnvConfig()
        |-- memcpy_s(rootInfo->internal -> rootHandle)
        |-- CommConfig commConfig(identifier)
        |-- InitCommRootInfo()                               :1359  ← 核心
        |-- HCCL_PROFILER_ADD_GROUP_UDI
        |-- NslbDp::SendGlobalDisRankTable (可选)

HcclCommInitClusterInfo(clusterInfo, rank, comm)            :586
  |-- [V2分派] → HcclCommInitClusterInfoV2
  |-- [V1路径]
  |-- HcomLoadRanktableFile(clusterInfo -> rankTableM, realFilePath)
  |-- InitCommClusterInfo(rankTableM, rank, commConfig, ...)  :396
        |-- HcomCheckRankTable(rankTableM)                    # 校验JSON格式
        |-- CfgGetClusterInfo(rankTableM, rank -> params, rankTable) # 解析RankTable
        |-- pComm->init(params, commConfig, rankTable)        # 同下

HcclCommInitAll(ndev, devices, comms)                       :223
  |-- [V2分派] → HcclCommInitAllV2
  |-- [V1路径]
  |-- std::async -> HcclGetCommAll()                         :168
        |-- HcclGetRootInfo(&rootHandle)                     :1124
        |     |-- TopoInfoDetect->SetupServer(rootHandle)    # Root rank启动Server
        |-- N个线程并行: GetDeviceComm()                      :147
              |-- HcclCommInitRootInfo(ndev, &rootHandle, rank, &comm)  # 递归调用
```

两种入口的关键差异:

| 维度 | HcclCommInitRootInfo | HcclCommInitClusterInfo |
|------|---------------------|------------------------|
| 拓扑发现 | TopoInfoDetect网络交换 | CfgGetClusterInfo文件解析 |
| Identifier来源 | rootHandle.identifier | 默认HCCL_WORLD_GROUP |
| RankTable获取 | 从拓扑发现结果自动生成 | 从JSON文件加载 |

### 拓扑发现: TopoInfoDetect

文件: `hcomm/src/framework/common/src/topo/topoinfo_detect.h/cc`
      `hcomm/src/framework/common/src/topo/topoinfo_exchange_server.h/cc`
      `hcomm/src/framework/common/src/topo/topoinfo_exchange_agent.h/cc`
      `hcomm/src/framework/common/src/topo/topoinfo_exchange_base.h/cc`

星形拓扑发现协议:
1. Root rank调用HcclGetRootInfo() → TopoInfoDetect::SetupServer()
   - GenerateRootInfo() 生成 HcclRootHandle(ip+port+nicDeploy+identifier+rankId)
   - identifier格式: `ip_port_devPhyId_timestamp` (会话唯一标识)
   - 启动TCP Server监听
2. 用户将RootInfo通过MPI/NCCL等机制广播给所有rank
3. 其他rank调用HcclCommInitRootInfo() → TopoInfoDetect::SetupAgent()
   - 连接到Root Server的(ip, port)
4. 信息交换使用TCP socket + JSON序列化:
   - Agent发送握手(agentID + rankSize)
   - Agent发送本地RankTable JSON(只含自己一条记录)
   - Server合并所有rank的信息
   - Server广播全局RankTable JSON给所有Agent
   - 带step同步校验确保一致性

分层模式(nRanks > 32768):
- SetupHierarchical() 引入GroupLeader中间层
- 每2048个rank选一个GroupLeader(按rank顺序，每组第一个)
- 三阶段: 全量连root选GL → GL开监听 → 组内收集+全局合并
- 避免root直接管理几万连接的瓶颈
- GroupLeader复用grpLeaderToRoot_ socket连接与root通信

HcclRootHandle POD结构体:
```cpp
struct HcclRootHandle {
    HcclIpAddress ip;       // Server监听IP
    uint32_t port;          // Server监听端口
    uint32_t nicDeploy;     // NIC部署模式
    char identifier[128];   // 会话标识(ip_port_devPhyId_timestamp)
    uint32_t rankId;        // Root rank ID
};
```

### V1路径核心: InitCommRootInfo

文件: `hcomm/src/framework/op_base/src/op_base.cc:1359-1577`

```
InitCommRootInfo(nRanks, rank, rootHandle, commConfig, comm)
  |
  |-- 1. 前置检查
  |   |-- GetHcclOpInfoCtx()                # per-device全局上下文
  |   |-- 检查identifier是否已在opGroup2CommMap中(防重复)
  |   |-- hrtGetDeviceType() -> devType
  |   |-- 计算retryEnable(仅910_93+非AIV+配置了retry)
  |
  |-- 2. 创建hcclComm对象
  |   |-- new hcclComm(bufferSize, identifier, bufferName)
  |
  |-- 3. 拓扑发现
  |   |-- nRanks > 32K: SetupHierarchical() (分层)
  |   |-- nRanks <= 32K: SetupAgent() (扁平)
  |   |-- GetTopoDetectInfo() -> params, rankTable
  |
  |-- 4. 初始化hcclComm
  |   |-- InitWorkflowMode(HCCL_WORKFLOW_MODE_OP_BASE)
  |   |-- InitOtherInfo(params, nullptr)
  |   |-- pComm->init(params, commConfig, rankTable)    ← 进入核心初始化
  |
  |-- 5. 后置配置(10+项环境变量驱动的配置)
  |   |-- SetDeterministicConfig / SetQpQosAttr / SetAivModeConfig
  |   |-- SetOnlyAivModeConfig / SetAicpuUnfoldConfig / SetIndependentOpConfig
  |   |-- SetExecTimeOutConfig / SetAlgoConfig / SetHcclQos
  |
  |-- 6. 全局注册
  |   |-- *comm = pComm.get()
  |   |-- opGroup2CommMap[identifier] = pComm
  |   |-- if HCCL_WORLD_GROUP: HcomGetCtxHomInfo().pComm = pComm
  |
  |-- 7. 错误处理: do-while(0)+errorFlag → HcclCommDestroy(pComm.get())
```

### hcclComm::init 包装层

文件: `hcomm/src/framework/communicator/hccl_comm.cc:102-150`

```cpp
hcclComm::init(params, commConfig, rankTable)
  |-- UpdateIsHaveCpuRank(rankTable)                    // 检查CPU rank
  |-- InitImpl(deviceType, commConfig)                  :933
  |     |-- communicator_.reset(new HcclCommunicator(commConfig))
  |     |-- RegistTaskAbortHandler()
  |-- CommConfiger::GetInstance().SetCommConfig(...)
  |-- communicator_->AtomicInitSet()                    // 原子防重入(失败时Clear)
  |-- communicator_->Init(params, rankTable)            // ← 核心28步初始化
  |-- ShareCCLbufferMgr::RecordShareCCLbuffer()
  |-- communicator_->InitCCLbuffer(in, out)             // 创建CCL buffer
  |-- InitIndependentOp()                               // 独立算子
```

### HcclCommunicator::Init 28步初始化序列

文件: `hcomm/src/framework/communicator/impl/hccl_communicator_host.cc:330-380`

```
HcclCommunicator::Init(params, rankTable)

  === Phase A: 基础信息 ===
  Step 1:  InitCommParams(params)                :758   -- rank/deviceId/通信模式
  Step 2:  attrCollector_.Init(params, rankTable) :333  -- 属性收集器(解析拓扑)
  Step 3:  InitRankInfo(rankTable)               :151   -- Rank信息
             |-- InitTcpMode()    -- TCP/RDMA通信模式
             |-- SetAttrs()       -- 从attrCollector提取拓扑属性
             |-- SetRanksPort()   -- rank->port映射

  === Phase B: 网络资源 ===
  Step 4:  InitNetResource(rankTable)            :207
             |-- InitPreResource()  :1255  -- P2P使能/设备发现
             |-- InitRaResource()   :1334  -- RDMA/网络核心初始化
                   |-- HcclNetInit(DEVICE)       -- device侧网络
                   |-- InitSocketManager()       -- SocketManager
                   |-- VNIC初始化/IP获取
                   |-- socketManager_->ServerInit() -- 启动监听(建链Server端)
                   |-- InitNic()     :2334  -- 网卡初始化(打开NIC设备+启动监听)

  === Phase C: 运行时基础设施 ===
  Step 5:  InitDebug()                           :214   -- Profiling/Atrace
  Step 6:  InitNotifyManager()                   :275   -- 事件同步(Notify池)
  Step 7:  InitStreamManager()                   :334   -- Stream管理
  Step 8:  InitProfiler()                        :2060  -- Profiler
  Step 9:  InitDispatcher()                      :297   -- 任务派发器
             |-- HcclDispatcherInit()
             |-- CreateDispatcherCtx(commId)
  Step 10: InitTransportManager()                :349   -- 传输管理(normal+independentOp)
  Step 11: InitMemoryManager()                   :379   -- MR/接收buffer/内存块
  Step 12: InitCombinOpara()                     :7665  -- 组合操作参数
  Step 13: RegisterRanksToDca()                         -- DCA注册

  === Phase D: 算法引擎(加锁区, g_hcomInitMutex) ===
  Step 14: RegistTaskExceptionHandler()          :471
  Step 15: GenCollectiveId()                            -- 集合通信ID
  Step 16: InitPara()                            :1132  -- 核心算法参数(加锁)
             |-- attrCollector_.CheckLocalRankInfo()
             |-- OpExeCounter::InitCounter()
             |-- NotifyPool初始化
             |-- HcclCallbackTask创建
             |-- WorkspaceResource创建
             |-- implAlg_->Init()                       -- HcclAlg算法引擎初始化

  === Phase E: Kernel和通信通道 ===
  Step 17: RegisterKernel()                      :361   -- CCU/AIV内核注册
  Step 18: LoadAICPUKernel()                     :414   -- AICPU算子二进制加载
  Step 19: LoadCustomKernel()                           -- 自定义算子加载
  Step 20: InitHDCommunicate()                   :601   -- H2D/D2H通信通道(KFC协议)
  Step 21: InitOpRetry()                         :1180  -- 操作重试管理器
  Step 22: InitOpResPara()                       :124   -- 操作资源参数
  Step 23: InitOneSidedService(rankTable)        :440   -- 单边通信服务
  Step 24: OrderLaunch::RegisterOrderLaunch()           -- 有序下发注册
  Step 25: rankGraph_.Init(rankTable, topoAttr)         -- RankGraph初始化
  Step 26: SaveTopoDesc()                               -- 拓扑描述保存
  Step 27: RegisterToSnapshot()                         -- 快照注册(910B/910_93)
  Step 28: InitSymmetricMemory()                 :9105  -- 对称内存初始化
```

### HcclAlg 算法引擎初始化

文件: `hcomm/src/framework/communicator/impl/alg/hccl_alg.cc`

```
HcclAlg::Init()                                         :55-101
  |-- AlgConfigurator::Init()                            -- 三级算法选择(L0/L1/L2)
  |     |-- GetLevel0AlgType()                           -- HCCS/SIO内算法
  |     |-- GetLevel1AlgType()                           -- 节点间算法
  |     |-- GetLevel2AlgType()                           -- SuperPod间算法
  |-- TopoInfoExtractor::Init()                          -- 拓扑信息提取
  |     |-- 计算各级通信域的rank分组(CommPlaneVector)
  |     |-- 建立rank映射关系
  |-- TopoMatcher::Init()                               -- 拓扑匹配
  |     |-- 持有TopoInfoExtractor的输出
  |     |-- 运行时计算transport需求
  |-- ParallelTaskLoader初始化                            -- 并行任务加载器
  |-- TinyMem初始化                                      -- 小内存管理
```

关键: HcclAlg::Init阶段不建链。只准备建链所需的拓扑分组和算法配置。
实际建链是lazy的 -- 在首次执行集合通信算子时触发(通过transportManager_->Alloc())。

### V2路径(910_95芯片)初始化流程

文件: `hcomm/src/legacy/framework/entrance/op_base/op_base_v2.cc`
      `hcomm/src/legacy/framework/communicator/communicator_impl.cc`

V2路径入口:
```
HcclCommInitRootInfoV2(nRanks, rootInfo, rank, comm)       :1618
  |-- CommInitRootInfo(nRanks, rootInfo, rank, comm)        :1557
        |-- 1. CallSingletons()                             # 预初始化11个全局单例
        |-- 2. RootInfoDetect(rank, nRanks, rootHandle)     :1510
        |     |-- RankInfoDetect::Client connect + 交换rank信息
        |     |-- 获取commId/rankTable
        |-- 3. new Hccl::HcclCommunicator(deviceType)
        |-- 4. comm->Init(commParams, ranktableM, config)
        |-- 5. 注册到hcclGroupMap/allComm
```

CommunicatorImpl::InitCommResource (V2核心):
```
CommunicatorImpl::InitCommResource(commParams)              :124-159
  |-- HrtSetDevice(devLogicId)
  |-- InitHccpHdc()               -- HCCP HDC通道(V2首先执行, V1按需lazy)
  |-- InitHccpPeer()              -- DPU场景(V2特有)
  |-- AppendLocalDieIdForLinks()
  |-- InitCcuSuperFastLoad()      -- CCU快速下发(V2特有)
  |-- InitNotifyManager()
  |-- InitStreamManager()
  |-- InitSocketManager()
  |-- InitRmaConnManager()        -- RMA连接管理(V2特有)
  |-- InitDataBufferManager()     -- 数据缓冲区管理(V2特有)
  |-- InitNotifyFixedValue()
  |-- InitMemTransportManager()
  |-- InitHostDeviceSyncNotifyManager()
  |-- InitUbMemoryTransportMgr()  -- UB内存传输管理(V2特有)
  |-- CollAlgComponentInit()
  |-- InitCollService()           -- 3种Service(DeviceMode/AiCpuImpl/Default)
  |-- InitTraceManager()
  |-- DlProf初始化
  |-- InitMirrorTaskManager()
  |-- InitProfilingReporter()
  |-- InitTaskExceptionHandler()
  |-- InitHDCommunicate()
  |-- notifyTimeoutCfg.Init()
  |-- status = COMM_READY
  |-- SnapShotParser序列化
  |-- InitOneSidedService()
  |-- RegisterKernel()
  |-- InitDpuKernel()             -- DPU内核(V2特有)
```

### V1 vs V2 关键差异

| 维度 | V1 (HcclCommunicator) | V2 (CommunicatorImpl) |
|------|----------------------|----------------------|
| 适用芯片 | 910/910B/910_93 | 910_95 (Ascend950) |
| HCCP交互 | 按需lazy (H2DTlv) | Init时直接初始化HccpHdcManager |
| 错误处理 | CHK_RET逐步返回 | try-catch(HcclException) |
| 拓扑解析 | attrCollector_+rankTable | InitRankGraph(RankTableInfo) |
| 算法引擎 | HcclAlg(统一) | CollAlgComponent |
| 内存管理 | MrManager+TransportManager | RmaConnManager+DataBufferManager+UbMemoryTransportMgr |
| CCU | 后续lazy | InitCcuSuperFastLoad立即 |
| DPU支持 | 不支持 | InitHccpPeer+InitDpuKernel |
| 全局单例 | 按需创建 | CallSingletons预初始化11个 |
| 回滚 | AtomicInitClear+HcclCommDestroy | initFlag防重入(无显式回滚) |

### 全局状态管理

```cpp
// op_base.cc:83-86
thread_local s32 g_hcclDeviceId;                            // 线程级设备ID
std::mutex g_opHcomInfosMutex;                              // 全局互斥锁
HcclOpInfoCtx g_opHcomInfos[MAX_MODULE_DEVICE_NUM + 1];    // per-device上下文

// HcclOpInfoCtx (op_base.h:26)
struct HcclInfoTag {
    HcclCommPtr pComm;                                      // 主通信域(shared_ptr)
    HcclCommParams params;
    RankTable_t rankTable;
    bool cloudFlag;
    bool isUsed;
    std::mutex opGroupMapMutex;
    unordered_map<string, shared_ptr<hcclComm>> opGroup2CommMap;  // group名→通信域
    map<string, shared_ptr<TopoInfoDetect>> ...Server/Agent;
};
```

设备上下文获取(GetHcclOpInfoCtx :127):
1. RT获取当前设备ID → g_opHcomInfos[deviceId]
2. RT失败 → 遍历找已用slot
3. 兜底 → g_opHcomInfos[MAX_MODULE_DEVICE_NUM]

### hccl→hcomm 接口分层

通信域创建完成后，hccl仓库通过三层接口调用hcomm:

第一层 -- 控制面/拓扑查询 (hccl_rank_graph.h, hccl中72处调用/11文件):
- HcclGetRankSize/HcclGetRankId — rank信息
- HcclRankGraphGetLayers — 网络层级
- HcclRankGraphGetTopoType/ByLayer — 拓扑类型
- HcclRankGraphGetRanks/ByTopoInst — rank列表
- HcclRankGraphGetLinks — 链路信息
- HcclRankGraphGetEndpointNum/Desc/Info — 端点信息

第二层 -- 资源管理 (hccl_res.h, hccl中91处调用/17文件):
- HcclGetHcclBuffer — CCL通信buffer
- HcclThreadAcquire/WithStream — 执行线程
- HcclChannelAcquire — 通信通道
- HcclEngineCtxCreate/Get/Copy — 引擎上下文
- HcclCommMemReg — 内存注册
- HcclDevMemAcquire — 设备内存

第三层 -- 数据面原语 (hcomm_primitives.h, hccl中149处调用/17文件):
- HcommWrite/ReadOnThread — RDMA Write/Read
- HcommLocalCopy/ReduceOnThread — 本地拷贝/归约
- HcommWriteReduce/WithNotifyOnThread — 远程归约
- HcommChannelNotifyRecord/Wait — 通道同步
- HcommThreadNotifyRecord/Wait — 线程间同步
- HcommBatchModeStart/End — 批量下发

所有句柄类型:
- HcclComm (void*): 通信域句柄
- ChannelHandle (uint64_t): 通信通道句柄
- ThreadHandle (uint64_t): 执行线程句柄
- HcclMemHandle: 内存注册句柄

### 建链时机: Init时准备，首次操作时实际建链

Init阶段只做准备:
- InitRaResource: 打开NIC设备 + 启动监听(ServerInit)
- InitTransportManager: 创建管理器实例(空容器)
- HcclAlg::Init: 计算拓扑分组和算法配置(CommPlaneVector)

实际建链发生在首次集合通信操作时:
- V1: CreateCommResource() (hccl_communicator_host.cc:6894)
  - CreateCommAndStreamRes(newTag, stream)
  - TransportManager在AllocComResource时按需创建Transport连接
- V2: LoadOpbasedCollOp/AllocCollOpResource时建立连接

这是lazy initialization策略 -- 避免在init时建立可能用不到的连接。

### Idioms & Conventions

ID-RL-1: API入口固定模板(V1)
```
HcclCommInit*(...)
  |-- HcclDeviceRefresh()           // thread_local设备ID
  |-- 参数校验                       // CHK_PTR_NULL + RPT_INPUT_ERR
  |-- HCCLV2_FUNC_RUN(V2版本)       // V2分派
  |-- InitExternalInput/EnvConfig   // 环境变量
  |-- InitComm*()                   // 核心初始化
  |-- 后置配置(Set*Config)           // 10+项配置
  |-- 全局注册                       // opGroup2CommMap
```

ID-RL-2: 二级防重复初始化
- API层: 检查identifier是否已在opGroup2CommMap中
- Communicator层: AtomicInitSet()/AtomicInitClear() 原子操作

ID-RL-3: do-while(0)+errorFlag清理模式
```cpp
bool errorFlag = false;
do {
    // ... 初始化步骤 ...
    if (ret != HCCL_SUCCESS) { errorFlag = true; break; }
    // ...
} while(0);
if (errorFlag) { HcclCommDestroy(pComm.get()); }
```
出现3处(op_base.cc中通信域初始化函数)。

ID-RL-4: per-device全局上下文数组
- g_opHcomInfos[MAX_MODULE_DEVICE_NUM + 1] — 静态数组，不用动态分配
- 通过hrtGetDevice获取索引，有兜底slot

ID-RL-5: 拓扑发现的Server-Agent模式
- Root rank做Server(GenerateRootInfo → listen → accept → 汇总)
- 其他rank做Agent(connect → 上报 → 接收全局RankTable)
- 大规模(>32K)引入GroupLeader中间层

### Anti-Patterns

AP-RL-1: 全局可变状态无锁保护
- g_hcclDeviceId (thread_local, 本身线程安全)
- g_opHcomInfos (全局数组, g_opHcomInfosMutex保护不完全)
- GetHcclOpInfoCtx中isUsed字段读取无锁

AP-RL-2: 拓扑发现Server/Agent的全局Map
- hcclCommTopoInfoDetectServer/Agent(per-device map)
- 在HcclGetRootInfo中创建并存入全局map
- TopoInfoDetect对象生命周期由map管理，但map清理时机不明确

AP-RL-3: 后置配置代码大量重复
- SetDeterministicConfig/SetQpQosAttr/SetAivModeConfig等10+项配置
  在InitCommRootInfo/InitCommClusterInfo/InitCommRootInfoConfig中重复出现
  (每处~80行配置代码)
- 应提取为单独的PostInitConfig()函数

AP-RL-4: V2分派位置不一致
- 部分API在参数校验前V2分派(跳过校验)
- 部分API在校验后V2分派(V2接收已校验参数)
- 导致V1/V2行为不一致

AP-RL-5: HcclCommInitAll的并发安全问题
- N个线程并行调用HcclCommInitRootInfo
- 每个线程设置hrtSetDevice(不同设备)
- 但全局状态访问(opGroup2CommMap等)的锁粒度可能不够
- 失败时"销毁所有已创建comm"逻辑可能与仍在初始化的线程竞争

AP-RL-6: HcclAlg::Init无回滚
- 28步初始化中Step 16(HcclAlg::Init)内部初始化了5个子系统
- 任一子系统失败，已初始化的子系统不做清理
- 依赖析构函数最终清理(RAII)，但中间状态可能不一致

AP-RL-7: 拓扑发现使用JSON序列化
- Agent和Server之间通过JSON字符串交换RankTable
- 大规模(数万rank)时JSON解析性能可能是瓶颈
- 没有二进制协议选项

### Domain Knowledge

DK-RL-1: 通信域创建是同步阻塞操作(除非在GroupStart/End中)
- 所有rank必须同时参与拓扑发现
- 拓扑发现通过TCP socket交换，有超时机制
- Group模式允许异步追加(commInitTaskAppend)，但GroupEnd时仍同步执行

DK-RL-2: 三种通信域初始化模式的适用场景
- RootInfo模式(多进程): 适用于分布式训练，每进程一个rank
- ClusterInfo模式(文件): 适用于静态配置的集群环境
- InitAll模式(单进程多卡): 适用于单机多卡推理/训练

DK-RL-3: 通信域标识(identifier)的唯一性
- RootInfo模式: identifier = `ip_port_devPhyId_timestamp`(自动生成)
- ClusterInfo模式: identifier = HCCL_WORLD_GROUP(固定)
- 子通信域: identifier可由用户指定(HcclCreateSubCommConfig)

DK-RL-4: Init完成并不意味着所有连接就绪
- Init只准备了网络基础设施(NIC打开、监听端口)
- Transport连接在首次操作时lazy建立
- CCL Buffer在init返回前已分配(hcclComm::init最后一步)

### Notable Patterns

NP-RL-1: 拓扑发现与通信域初始化的解耦
- 拓扑发现(TopoInfoDetect)是独立模块，在op_base层执行
- 产出RankTable传入hcclComm::init
- HcclCommunicator不关心拓扑是如何发现的(网络/文件/手动)

NP-RL-2: 28步初始化的依赖关系
- Phase A(1-3)无依赖，可独立执行
- Phase B(4)依赖Phase A的rank信息
- Phase C(5-13)依赖Phase B的网络资源
- Phase D(14-16)加全局锁，依赖Phase C
- Phase E(17-28)依赖Phase D的算法配置

NP-RL-3: weak符号的V2覆盖机制
- 所有公共API用__attribute__((weak))声明
- V2实现用强符号定义同名函数
- 链接时自动选择强符号(V2)覆盖弱符号(V1声明)
- 运行时通过IsSupportHCCLV2()决定是否使用V2路径

---

## 5.2.2 资源分配与绑定: Buffer, Transport, Thread, Channel

### CCL Buffer 生命周期

文件: `hcomm/src/framework/communicator/hccl_comm.cc` (InitCCLbuffer)
      `hcomm/src/framework/communicator/impl/hccl_communicator_host.cc` (AllocCCLBuffer)
      `hcomm/src/framework/common/src/share_cclbuffer_mgr/share_cclbuffer_mgr.cc`

Lazy allocation — Init时只记size，首次集合通信才真正分配HBM:
```
hcclComm::init()
  |-- InitCCLbuffer(inCCLbufferSize_, outCCLbufferSize_)  // 记录大小
  |-- (此时不分配内存)

首次ExecOp()
  |-- AllocCCLBuffer()                                     // 真正分配
        |-- hrtMalloc(inCCLbufferSize + outCCLbufferSize + expSize)
        |-- input区 = base + 0                              // 200MB默认
        |-- output区 = base + inSize                        // 200MB默认
        |-- exp区 = base + inSize + outSize                 // 1MB固定
```

三区一体布局:
- 一次hrtMalloc连续内存，偏移切分为Input(200MB默认) + Output(200MB默认) + Exp(1MB固定)
- Scratch不是预分配区 — 图模式复用Output区域地址，单算子模式独立alloc
- 大小可通过环境变量或HcclCommConfig配置

跨通信域共享(ShareCCLbufferMgr):
- 同一设备上同bufferName的通信域共享同一块CCL Buffer
- per-device单例管理，引用计数控制生命周期
- 约束: 共享Buffer的通信域必须在同一stream上执行(避免数据竞争)
- 通过bufferName做key查找: 有则复用(引用计数+1)，无则新建

AIV buffer独立体系:
- 40MB data+flag + 32KB comm info
- 与CCL Buffer分开管理(不走ShareCCLbufferMgr)

### Transport连接 Lazy建链

文件: `hcomm/src/framework/communicator/impl/hccl_communicator_host.cc`
      `hcomm/src/platform/resource/transport/transport_base_pub.h`
      `hcomm/src/platform/resource/transport/*/`

触发路径:
```
ExecOp(tag, ...)
  |-- resMap_.find(tag)                                    // 按tag查缓存
  |-- (缓存miss)
  |-- AllocAlgResource(tag)
        |-- transportManager_->Alloc(tag, transportRequest)
              |-- 对每个(srcRank, dstRank)对:
              |     |-- 检查缓存(transportMap_)
              |     |-- CreateLink线程(并行建链)
              |           |-- SetMachinePara()              // socket握手
              |           |-- GetTransportType()            // 选择Transport类型
              |           |-- TransportInit()               // 创建Transport实例
              |           |-- transport->Init()             // 建立连接
              |-- join所有CreateLink线程
              |-- 将结果存入resMap_[tag]                    // 缓存
```

四种Transport类型的建链差异:

| Transport | 适用场景 | 建链过程 |
|-----------|---------|---------|
| P2P | HCCS(同module) | DeviceMem映射+IPC handle交换 |
| IBVerbs | RoCE(跨节点RDMA) | QP创建→RTR→RTS→MR注册→地址交换 |
| TCP | 退化路径 | Socket connect/accept |
| DirectNpu | 设备直通 | 设备端IBVerbs初始化 |

IBVerbs建链详细步骤:
1. ibv_open_device() — 打开网卡
2. ibv_alloc_pd() — 分配Protection Domain
3. ibv_create_cq() — 创建Completion Queue
4. ibv_create_qp(IBV_QPT_RC) — 创建QP(Reliable Connection)
5. QP状态转换: RESET → INIT → RTR → RTS
6. ibv_reg_mr() — 注册Memory Region
7. Socket交换(QP号, GID, LID, rkey, addr)

连接缓存策略:
- resMap_(tag级): 同tag复用已建链的Transport(启用)
- transportMap_(hash级): per-(srcRank,dstRank)级缓存(代码存在但未启用)
- remoteTransportMap_: BSR(BatchSendRecv)类型一致性保证
- 1000连接阈值: 超过时从IBV_EXP模式切换到DEVICE_DIRECT模式

910_93分批交错建链:
- 非全并行，而是分批(batch)交错创建
- 每批内并行，批间串行(避免网络风暴)
- 其他设备: 全并行CreateLink

### Thread/Channel/Notify 资源获取

文件: `hcomm/src/framework/communicator/impl/independent_op/hccl_independent_op_engine.cc`
      `hcomm/src/framework/next/comms/comm_engine_res/threads/thread.h/cc`
      `hcomm/src/framework/next/comms/endpoint_pairs/channels/channel.h/cc`
      `hcomm/src/platform/resource/notify/notify_pool_impl.cc`

Thread资源获取(HcclThreadAcquire):
```
HcclThreadAcquire(comm, engine, threadNum, notifyNumPerThread, threads)
  |-- hcclComm->GetIndependentOp()->GetEnginRes()
  |-- ThreadManager::AcquireThreads(engine, threadNum, notifyPerThread)
        |-- for each threadNum:
        |     |-- Thread::Create(engine)                    // 工厂方法
        |     |     |-- engine==CPU/CPU_TS → new CpuTsThread()
        |     |     |-- engine==AICPU_TS  → new AicpuTsThread()
        |     |     |-- engine==AIV       → new AivThread() (空壳)
        |     |-- thread->HostInit(notifyNum)               // Host侧初始化
        |     |     |-- 创建Stream
        |     |     |-- 分配notifyNum个Notify
        |     |     |-- (AICPU) 序列化资源→H2D memcpy
        |     |-- 返回ThreadHandle(uint64_t, reinterpret_cast)
```

Thread继承体系:
- Thread基类: 11个virtual方法(Init/HostInit/DeviceInit/Record/Wait/Synchronize等)
- CpuTsThread: Host-only执行(Runtime Stream)
- AicpuTsThread: Host+Device双侧(H2D资源序列化+kernel launch)
- AivThread: 空壳(AIV不走Thread模型)

Channel资源获取(HcclChannelAcquire):

V1路径(IndependentOp):
```
HcclChannelAcquire(comm, engine, channelDescs, channelNum, channels)
  |-- IndependentOp::ChannelManager::AcquireChannels()
        |-- for each channelDesc:
        |     |-- 根据(remoteRank, protocol)查找/创建Transport
        |     |-- 包装为Channel句柄(ChannelHandle)
        |     |-- 创建Host→Device深拷贝链:
        |           |-- 序列化channel资源到BinaryStream
        |           |-- H2D memcpy
        |           |-- AICPU kernel初始化
        |           |-- D2H取回device handle
```

V2路径(CollComm/next):
```
HcclChannelAcquire(comm, engine, channelDescs, channelNum, channels)
  |-- 路由到CollComm::MyRank
  |-- EndpointPairMgr::CreateEndpointPairs()
        |-- for each channelDesc:
        |     |-- Endpoint::Create(protocol, location)     // 4种Endpoint
        |     |-- EndpointPair::Create(local, remote)
        |     |-- Channel::Create(engine)                   // 5种Channel
        |           |-- engine×protocol矩阵: 4个实际实现+4个空壳
        |     |-- 存入全局static map(handle→Channel)
        |     |-- 返回ChannelHandle
```

Channel与Transport的关系:
- V1: Channel是Transport的1:1包装(ChannelManager内部持有TransportManager引用)
- V2(next): Channel是EndpointPair的包装，EndpointPair持有两个Endpoint
  Endpoint封装了底层协议(RoCE/HCCS/UB等)
- 两套路径通过IsCommunicatorV2()分叉

Notify资源:

NotifyPool三层管理:
```
NotifyPoolImpl
  |-- normalPool_    — 通用Notify(集合通信)
  |-- a2aPool_       — AllToAll专用(大量通知需求)
  |-- alignPool_     — 对齐Notify(特殊硬件要求)
```

Notify分配:
- 每个Thread在HostInit时预分配notifyNum个Notify(从Pool中取)
- Transport建链时也分配Notify(Record/Wait用)
- TransportBase内部: localNotify_ + remoteNotify_ 一对(用于同步)
- Notify的物理实现取决于设备: RtsNotify(硬件) / BareNotify(空) / EschedNotify(事件)

### 资源间关联关系图

```
HcclComm (void*)
  |
  +-- HcclCommunicator
        |
        +-- ShareCCLbufferMgr --> CCL Buffer (Input+Output+Exp)
        |
        +-- TransportManager
        |     |-- resMap_[tag] --> Transport[] (lazy创建)
        |     |-- transportMap_[(src,dst)] --> Transport缓存
        |
        +-- NotifyPoolImpl
        |     |-- normalPool_ / a2aPool_ / alignPool_
        |
        +-- StreamManager --> Stream[]
        |
        +-- Dispatcher --> DispatcherCtx
        |
        +-- IndependentOp (V1) / CollComm (V2)
              |
              +-- ThreadManager --> Thread[] --> Stream + Notify[]
              +-- ChannelManager --> Channel[] --> Transport + Notify
```

### Idioms & Conventions

ID-RL-6: Lazy资源分配惯用法
- CCL Buffer: init记size，首次ExecOp时分配
- Transport: init时只ServerInit(监听)，首次操作时CreateLink
- 好处: 避免分配可能用不到的资源(如某些拓扑路径)

ID-RL-7: 句柄化(Handle-based)资源管理
- 所有资源通过uint64_t句柄暴露给hccl层
- hcomm内部通过reinterpret_cast在句柄和指针间转换
- 全局static map维护(handle→对象)映射(V2路径)
- 好处: C ABI兼容，隐藏内部类型

ID-RL-8: 连续内存切分模式
- CCL Buffer: 一次alloc → 偏移切分Input/Output/Exp
- AICPU SQE: 连续分配 → 按64B步长切分
- 好处: 减少alloc次数，保证局部性

ID-RL-9: 引用计数共享
- ShareCCLbufferMgr: 同bufferName引用计数+1
- MR(Memory Region): 全局引用计数防重复注销
- TransportManager: hash-based连接复用

### Anti-Patterns

AP-RL-8: Transport缓存代码存在但未启用
- transportMap_(hash级缓存)的Put/Get逻辑已实现
- 但调用处被注释或条件跳过
- 导致相同(src,dst)对可能重复建链

AP-RL-9: CCL Buffer共享的隐式约束
- 共享Buffer的通信域必须在同一stream上执行
- 这个约束只在注释中说明，没有运行时校验
- 违反时会导致silent数据竞争

AP-RL-10: Thread句柄的reinterpret_cast无类型安全
- ThreadHandle = reinterpret_cast<uint64_t>(thread_ptr)
- 反向转换时无类型检查
- 传入错误句柄会导致未定义行为

AP-RL-11: 全局static map管理Channel(V2路径)
- 4个全局static map存储handle→对象映射
- 没有大小限制(OOM风险)
- 没有清理机制描述(何时remove)

AP-RL-12: 建链线程join无超时
- CreateLink线程启动后用thread.join()等待
- 如果远端rank不响应(网络故障)，join永久阻塞
- 应有超时机制(但依赖底层socket timeout)

---

## 5.2.3 通信域销毁与异常清理

### HcclCommDestroy 正常销毁流程

文件: `hcomm/src/framework/op_base/src/op_base.cc:2989`

API入口层:
```
HcclCommDestroy(HcclComm comm)                             :2989
  |-- CHK_PTR_NULL(comm)
  |-- HcclDeviceRefresh()
  |-- [V2分派] HCCLV2_FUNC_RUN(HcclCommDestroyV2(comm))    :567
  |-- [V1路径]
  |-- HcclCloseCommConnections(hcclComm*)                   :1253
  |     |-- operatorlock_.lock()                            // 等待所有算子完成
  |     |-- HcclCommunicator::SetOverFlowAddr                // 溢出地址设置
  |     |-- communicator_析构(unique_ptr reset)               // 触发~HcclCommunicator
  |-- ShareCCLbufferMgr::UnRecordShareCCLbuffer()           // 引用计数-1
  |-- GetHcclOpInfoCtx().opGroup2CommMap.erase(identifier)  // 全局注销
  |-- DestroyTopoInfoDetect(identifier)                     // 拓扑发现对象清理
  |-- MayResetIsUsed()                                      // 如果map为空,重置isUsed
```

### HcclCommunicator析构(V1): 43步逆序清理

文件: `hcomm/src/framework/communicator/impl/hccl_communicator_host.cc:171-317`

```
~HcclCommunicator()
  |
  |== Phase E逆序: Kernel和通信通道 ==
  |-- Step 43: SnapshotCleanUp()
  |-- Step 42: UnInitSymmetricMemory()
  |-- Step 41: DestroyOneSidedService()
  |-- Step 40: DestroyOpRetry()
  |-- Step 39: DestroyHDCommunicate()            // H2D/D2H通道
  |-- Step 38: UnloadCustomKernel()
  |-- Step 37: UnloadAICPUKernel()
  |-- Step 36: UnRegisterKernel()                // CCU/AIV内核
  |
  |== Phase D逆序: 算法引擎 ==
  |-- Step 35: implAlg_.reset()                  // HcclAlg析构
  |-- Step 34: DestroyWorkspaceResource()
  |-- Step 33: DestroyNotifyPool()
  |-- Step 32: UnRegisterTaskExceptionHandler()
  |
  |== Phase C逆序: 运行时基础设施 ==
  |-- Step 31: DestroyOpTransportResponse()      // Transport链路
  |       |-- transportManager_->DestroyAllLinks()
  |       |-- 遍历resMap_中所有tag的Transport:
  |             |-- transport->DeInit()           // 关闭QP/释放MR/关Socket
  |-- Step 30: DestroyNetworkResources()          :1600
  |       |-- socketManager_->DeInit()            // 关闭所有监听Socket
  |       |-- HcclNetCloseDev(每个NIC)            // 关闭网卡设备
  |       |-- HcclNetDeInit()                     // 网络层清理
  |-- Step 29: DestroyMemoryManager()
  |       |-- MrManagerDeInit()                   // MR注销
  |       |-- 释放recv buffer内存
  |-- Step 28: DestroyDispatcher()
  |       |-- HcclDispatcherDestroy()
  |-- Step 27: DestroyStreamManager()
  |-- Step 26: DestroyNotifyManager()
  |-- Step 25: DestroyProfiler()
  |-- Step 24: DestroyDebug()
  |
  |== Phase B逆序: 网络资源 ==
  |-- (已在Step 30处理)
  |
  |== Phase A逆序: 基础信息 ==
  |-- Step 23: DestroyRankInfo()
  |-- Step 22: attrCollector_.DeInit()
  |-- Step 21: DestroyCommParams()
  |-- Step 20: OrderLaunch::UnRegister()
  |-- Step 19: DCA注销
  |-- Step 18: BackgroundThread注销
  |-- Step 17: AICPU comm销毁
  |
  |-- (unique_ptr成员自动析构)
```

关键点: 析构顺序是init的严格逆序。每步都用CHK_RET检查返回值(但析构中CHK_RET失败只记日志不中断)。

### V2路径销毁(CommunicatorImpl)

文件: `hcomm/src/legacy/framework/communicator/communicator_impl.cc:2306-2390`

```
~CommunicatorImpl()
  |-- DestroyDpuKernel()                          // DPU特有
  |-- DestroyOneSidedService()
  |-- DestroyHDCommunicate()
  |-- DestroyCollService()                        // 3种Service
  |-- DestroyTransportManager()
  |-- DestroyNotifyManager()
  |-- DestroyStreamManager()
  |-- DestroySocketManager()
  |-- DestroyRmaConnManager()                     // V2特有
  |-- DestroyDataBufferManager()                  // V2特有
  |-- DestroyUbMemoryTransportMgr()               // V2特有
  |-- HccpHdcManager::GetInstance().DeInit()       // HCCP HDC通道
  |-- status = COMM_DESTROYED
```

### ShareCCLbuffer 引用计数销毁

文件: `hcomm/src/algorithm/impl/resource_manager/share_ccl_buffer_manager.cc:88`

```
UnRecordShareCCLbuffer(bufferName, deviceLogicId)
  |-- refCount_[bufferName]--
  |-- if refCount == 0:
  |     |-- hrtFree(buffer_)                      // 释放HBM
  |     |-- buffer_ = nullptr
  |     |-- map_.erase(bufferName)
```

### Init失败时的清理

二级保护机制:

1. AtomicInitSet/Clear(communicator层):
```
hcclComm::init()
  |-- communicator_->AtomicInitSet()       // 标记"正在init"
  |-- communicator_->Init(params, ...)     // 如果失败:
  |     |-- (异常) communicator_->AtomicInitClear()  // 清除标记,允许重试
  |-- (成功后标记不会被清除，防止二次init)
```
AtomicInitSet/Clear是`hccl_communicator_host.cc:1669-1681`中的原子操作。

2. do-while(0)+errorFlag(op_base层):
```cpp
// op_base.cc 典型模式(3处)
bool errorFlag = false;
do {
    ret = InitStep1(); if (ret) { errorFlag = true; break; }
    ret = InitStep2(); if (ret) { errorFlag = true; break; }
    // ...
} while(0);
if (errorFlag) {
    HcclCommDestroy(pComm.get());  // 走正常销毁路径清理
}
```

已知问题: init部分完成时的中间状态:
- 28步初始化中，每步失败后直接返回错误码
- 已初始化的子系统依赖RAII(unique_ptr析构)自动清理
- HcclAlg::Init内部5个子系统没有显式回滚
- 但op_base层的errorFlag会触发HcclCommDestroy，进而析构整个对象

### 运行时故障恢复: 五步协议

文件: `hcomm/src/framework/communicator/impl/hccl_communicator.cc:1190-1377`

当集合通信操作检测到错误(如RDMA超时、设备异常)时:
```
Suspend()                                                   :1190
  |-- 标记comm为"暂停"状态
  |-- 等待所有正在执行的算子完成
  |-- 冻结新的算子提交

StopExec()                                                  :1240
  |-- 停止所有执行中的任务
  |-- 发送abort信号给device侧

Clean()                                                     :1280
  |-- 重置Transport链路状态
  |-- 重新注册Notify
  |-- 清理残留的中间数据

Resume()                                                    :1340
  |-- 重新建链(如果需要)
  |-- 恢复comm到"就绪"状态
  |-- 允许新的算子提交
```

OpRetryManager 状态机:
- Server-Agent双角色，每个通信域有一个实例
- 22个状态(RetryState枚举)
- State模式: 每种状态一个派生类
- 状态转换通过Host-Device通信(HDCommunicate)触发

KFC故障恢复(legacy, V2路径):
```
KFC协议: Suspend → Clean → Resume
  |-- Suspend: head_cnt/tail_cnt一致性等待
  |-- Clean: 清理设备侧缓冲区
  |-- Resume: 重新初始化KFC通道
  |-- 超时: 7秒(硬编码)
```

### TaskAbortHandler

文件: `hcomm/src/framework/communicator/impl/task_abort_handler.cc`

- 在hcclComm::init时通过RegistTaskAbortHandler注册
- 当runtime检测到设备异常(如NPU reset)时触发回调
- 回调内容: 标记comm为error状态 + 唤醒所有等待的线程
- N秒快恢复: 注册N秒内的快速恢复回调(不等待完整reset)

### TaskExceptionHandler

文件: `hcomm/src/common/debug/profiling/task_exception_handler.cc`

- 异常诊断(非恢复): 记录异常现场信息(task ID、stream状态、错误码)
- 与Profiling集成: 异常信息上报到分析系统
- 在HcclCommunicator::Init的Step 14注册

### Idioms & Conventions (销毁相关)

ID-RL-10: 严格逆序销毁
- 43步析构顺序是28步初始化的精确逆序
- 确保依赖关系的正确性(先销毁依赖方，后销毁被依赖方)
- 每步销毁都尝试继续(不因某步失败而中断后续清理)

ID-RL-11: operatorlock_等待模式
```cpp
// HcclCloseCommConnections
operatorlock_.lock();     // 等待所有正在执行的算子完成
// ... 开始销毁 ...
```
- 不使用condition_variable，直接锁等待
- 所有算子执行前acquire，完成后release
- 销毁时acquire意味着"等待所有算子完成"

ID-RL-12: 两层状态标记
- atomicInit_(Communicator层): 防止init/destroy并发
- status_(CommunicatorImpl, V2): COMM_READY/COMM_DESTROYED枚举

### Anti-Patterns (销毁相关)

AP-RL-13: 析构函数中CHK_RET
- ~HcclCommunicator 43步中每步都用CHK_RET
- CHK_RET在析构中失败只记WARN日志，不抛异常
- 但如果某步失败，后续步骤可能操作已失效的资源
- 正确做法: 析构中捕获所有异常，best-effort清理

AP-RL-14: Transport DeInit的顺序依赖
- TransportManager::DestroyAllLinks遍历所有tag
- 但Transport之间可能有隐式依赖(如共享的QP/MR)
- 销毁顺序不确定(unordered_map遍历)

AP-RL-15: ShareCCLbuffer引用计数与销毁竞态
- UnRecordShareCCLbuffer减引用计数
- 如果另一个comm正在使用buffer(不同stream)
- 引用计数到0时释放，但另一comm可能还在读

AP-RL-16: HcclCommDestroy后全局状态残留
- opGroup2CommMap.erase成功
- 但其他全局状态(如TopoInfoDetect的Server/Agent map)
  可能有残留引用
- MayResetIsUsed只在map为空时重置，否则保留isUsed=true

AP-RL-17: 五步故障恢复协议的复杂性
- Suspend/StopExec/Clean/Resume 四步需要严格顺序执行
- 跨Host-Device的状态同步(HDCommunicate)可能超时
- OpRetryManager 22个状态的状态机过于复杂，难以测试所有路径
- KFC的7秒硬编码超时可能在高负载下不够

### Domain Knowledge (资源生命周期)

DK-RL-5: 资源生命周期的三个阶段
```
Init阶段        → 准备(NIC打开, 监听, 管理器创建)
                  CCL Buffer记录size(不分配)
                  Transport管理器创建(空容器)

首次操作阶段    → Lazy分配(CCL Buffer HBM分配, Transport建链)
                  资源缓存到resMap_/transportMap_

Destroy阶段     → 逆序清理(Transport DeInit, Buffer释放, NIC关闭)
                  全局状态注销
```

DK-RL-6: 故障恢复的两种策略
- 算子级重执行(OpRetry): 单个操作超时后重试(不销毁通信域)
- 通信域级恢复(Suspend→Resume): 整个通信域暂停、清理、恢复
- 两者可以嵌套: OpRetry失败 → 触发通信域级恢复

### Notable Patterns (资源生命周期)

NP-RL-4: 销毁的"best-effort"策略
- 析构函数中的每步操作都不会因失败而中断后续清理
- 即使某个Transport DeInit失败，仍继续清理其他资源
- 这是C++析构函数的最佳实践(不抛异常)

NP-RL-5: 共享资源的引用计数归零模式
- ShareCCLbufferMgr: 最后一个comm销毁时释放HBM
- MR引用计数: 最后一个使用者注销时ibv_dereg_mr
- 单例(per-device): 最后一个comm销毁时重置isUsed

NP-RL-6: Init和Destroy的对称性
- 28步init ↔ 43步destroy (destroy步骤更多因为包含init时不存在的运行时资源)
- V1和V2各自保持独立的对称性
- 所有子系统都有对应的Init*/Destroy*方法对
