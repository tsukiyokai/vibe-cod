# Phase 5.1: AllReduce End-to-End Slice Analysis

## Overview

本文档追踪 AllReduce 操作从用户 API 调用到硬件执行的完整跨仓数据流。
AllReduce 是最核心的集合通信算子，其路径覆盖了 hccl 和 hcomm 的所有层次。

端到端路径:
```
User API (hccl)
  -> Op Entry (hccl)
    -> Selector (hccl)
      -> Executor (hccl)
        -> Template (hccl)
          -> [hccl/hcomm 边界] C ABI 接口
            -> Framework (hcomm)
              -> Platform (hcomm)
                -> Hardware Driver
```

---

## 1. hccl 侧: API 入口到算子入口

### 1.1 公共 API

文件: `hccl/include/hccl.h:35-36`

```cpp
extern HcclResult HcclAllReduce(void *sendBuf, void *recvBuf, uint64_t count,
    HcclDataType dataType, HcclReduceOp op, HcclComm comm, aclrtStream stream);
```

### 1.2 算子入口 (all_reduce_op.cc)

文件: `hccl/src/ops/all_reduce/all_reduce_op.cc`

HcclAllReduce 的执行分为两个阶段:

阶段 A: 前置检查 (行 39-76)
```
CheckHCCLIndependentOp()     // 判断是否走独立操作模式
hrtGetDeviceType()           // 获取设备类型 (910B/910_93/910_95)
GetWorkflowMode()            // 检查 HCCL_WORKFLOW_MODE_OP_BASE
InitEnvConfig()              // 读取 HCCL_OP_EXPANSION_MODE 等环境变量
CheckAllReduceInputPara()    // sendBuf/recvBuf/comm/stream 非空 + sendBuf != recvBuf
HcclGetRankSize/RankId()     // 获取通信域信息
HcclCheckTag/HcomCheckUserRank()  // tag 和 rank 校验
```

阶段 B: AllReduceOutPlace (行 104-156)
```
构造 OpParam:
  - inputPtr/outputPtr = sendBuf/recvBuf
  - DataDes.count/dataType
  - opType = HCCL_CMD_ALLREDUCE
  - reduceType = op
  - stream, commName, opMode = OPBASE

if rankSize == 1:
  -> SingleRankProc(param)  // 直接 memcpy, 跳过通信
else:
  -> Selector(comm, param, topoInfo, algName, opExecuteConfig)
  -> HcclExecOp(comm, param, topoInfo, algName)
```

关键数据结构: OpParam 是贯穿整个 hccl 侧的核心参数容器,
包含用户输入(buf/count/dataType/op) + 通信域信息(comm/rank) + 执行配置(engine/mode)。

---

## 2. hccl 侧: Selector 决策链

### 2.1 Selector 入口

文件: `hccl/src/ops/op_common/op_common.cc:47-72`

```
Selector(comm, param, topoInfo, algName, opExecuteConfig)
  1. HcclGetOpExpansionMode()  // 环境变量 -> opExecuteConfig 初值
     - HCCL_AICPU_UNFOLD=1 -> AICPU_TS
     - HCCL_AIV_MODE=1     -> AIV
     - HCCL_CCU_MS_MODE=1  -> CCU_MS
     - HCCL_CCU_SCHED_MODE=1 -> CCU_SCHED
  2. HcclCalcTopoInfo(comm)  // 计算拓扑信息
  3. ExecuteSelector::Run()  // 执行选择
```

### 2.2 ExecuteSelector 路由

文件: `hccl/src/ops/op_common/selector/execute_selector.cc:21-55`

```
如果 isMc2:
  -> 直接查找优先级 18 的 selector, 调用 Select()
否则:
  -> SelectorRegistry::GetSelectorsByOpType(HCCL_CMD_ALLREDUCE)
  -> 按优先级遍历, 第一个 MATCH 即返回
```

AllReduceAutoSelector 注册: 优先级 18 (硬编码)。
文件: `hccl/src/ops/all_reduce/selector/all_reduce_auto_selector.cc:348`

### 2.3 AutoSelectorBase 5 路分派

文件: `hccl/src/ops/op_common/selector/auto_selector_base.cc:16-66`

```
1. CheckHostDPUOnly() -> SelectDPUAlgo()      [DPU 专用]
2. if CCU_MS   -> SelectCcuMsAlgo()            [CCU 微码直发]
3. if CCU_SCHED -> SelectCcuScheduleAlgo()     [CCU 调度器]
4. if AIV      -> SelectAivAlgo()              [AIV 向量核]
5. fallback    -> SelectAicpuAlgo()            [AICPU 兜底]

降级链: CCU_MS 失败 -> CCU_SCHED -> AICPU (非 AIV)
```

### 2.4 AllReduce 算法决策矩阵

每路分派内部按 拓扑形状 x 数据量 x 层级数 选择具体算法:

CCU_MS 路径 (topoLevelNums <= 1, 排除 INT8/PROD/INT64/UINT64/FP64):
```
MESH_1D:
  TWO_DIE + small(<512KB)  -> CcuAllReduceMesh2Die
  TWO_DIE + large           -> CcuAllreduceMesh2DieBigMs
  SINGLE_DIE + small        -> CcuAllReduceMesh1DOneShot
  SINGLE_DIE + large        -> CcuAllReduceMesh1D
MESH_1D_CLOS (UBX):
  meshNum==closNum && rank<=4 + large -> CcuAllReduceConcurrentMs
  其他                                 -> CcuAllReduceMesh1D
```

CCU_SCHED 路径 (排除 PROD/INT64/UINT64/FP64):
```
topoLevelNums > 1:
  MESH_1D + deviceNumPerModule>1 -> CcuAllReduceParallelMesh1DNHR
  其他                            -> CcuAllReduceNHR1D
topoLevelNums <= 1 (dataSize <= 16MB):
  MESH_1D:
    TWO_DIE + small  -> CcuAllReduceMesh1DMem2Mem2DieOneShot
    TWO_DIE + large  -> CcuAllreduceMesh2DieBigSche
    SINGLE_DIE       -> CcuAllReduceMesh1DMem2Mem
  MESH_1D_CLOS:
    meshNum==closNum && rank<=4 + large -> CcuAllReduceConcurrentSche
    isClosMultiple + !small             -> CcuAllReduceParallelNHR1DMutiJetty
    其他                                -> CcuAllReduceNHR1DMem2MemMultiJetty
```

AICPU 路径 (排除 PROD):
```
topoLevelNums > 1:
  MESH_1D + deviceNumPerModule>1 -> InsAllReduceParallelMesh1DNHR
  其他                            -> InsAllReduceNHR
topoLevelNums <= 1:
  MESH_1D:
    dataSize <= 8MB  -> InsAllReduceMesh1DOneShot
    dataSize > 32MB  -> InsAllReduceMesh1DTwoShotMeshChunk
    8-32MB           -> InsAllReduceMesh1DTwoShot
  CLOS:              -> InsAllReduceNHR
  MESH_1D_CLOS:
    meshNum==closNum && rank<=4:
      small           -> InsAllReduceMesh1DOneShot
      large           -> InsAllReduceConcurrent
    isClosMultiple    -> InsAllReduceParallelMesh1DNHR
    其他              -> InsAllReduceNHR
```

AIV 路径 (仅 MESH_1D && topoLevelNums<=1, 排除 PROD/INT64/UINT64/FP64):
```
  small(<512KB) -> AivAllReduceMesh1DOneShot
  large         -> AivAllReduceMesh1DTwoShot
```

数据大小常量:
- SMALL_COUNT_512KB = 512*1024
- LARGE_COUNT_1024KB = 1024*1024
- AR_AICPU_1D_SMALL = 8MB
- AR_AICPU_1D_MAX = 32MB
- AR_M2M_1D_MAX = 16MB

Selector 输出: (algName: string, opExecuteConfig: enum) 二元组。
algName 直接用作 Registry 查找 key。

---

## 3. hccl 侧: Executor 编排

### 3.1 Executor 查找

文件: `hccl/src/ops/op_common/op_common.cc:74-139`

```
HcclExecOp(comm, param, topoInfo, algName)
  1. CollAlgExecRegistryV2::Instance().GetAlgExec(param.opType, algName)
     -> 返回 InsCollAlgBase* executor
  2. HcclGetAlgRes(comm, param, executor, topoInfo, resCtx, ...)
     -> 计算并分配资源
  3. executor->Orchestrate(param, resCtx)
     -> 编排执行
```

### 3.2 资源计算三步曲

文件: `hccl/src/ops/op_common/op_common.cc:280-341`

```
HcclGetAlgRes():
  Step 1: executor->CalcAlgHierarchyInfo(comm, topoInfo, algHierarchyInfo)
    - 委托给 TopoMatch::MatchTopo() 分析拓扑
    - 输出: 各级通信域的 rank 列表

  Step 2: executor->CalcRes(comm, param, topoInfo, algHierarchyInfo, resourceRequest)
    - 构造 template 并调用 template->CalcRes()
    - 输出: 线程数、通道描述、notify 数、scratch 倍数

  Step 3: HcclAllocAlgResource*(comm, resourceRequest, resCtx)
    - 根据 engine 类型调用不同的分配函数
    - AICPU: HcclGetHcclBuffer + HcclThreadAcquire + HcclChannelAcquire
    - CCU:   + HcclCcuKernelRegister
    - AIV:   + HcclChannelGetRemoteMems
    - 输出: AlgResourceCtxSerializable (序列化到 device)
```

### 3.3 四种 Executor 策略

AllReduce 有 4 种 Executor, 每种对应不同的拓扑和执行模式:

Sole (单 template):
```
文件: ins_v2_all_reduce_sole_executor.cc
场景: 单层拓扑 (MESH_1D)
逻辑: 单个 template, 按 UB 限制切分数据, 逐 loop 执行
  maxCountPerLoop = min(UB限制, scratch / multiplier)
  for loop in [0, loopTimes):
    template->KernelRun(curCount)
```

Concurrent (双 template 并行):
```
文件: ins_v2_all_reduce_concurrent_executor.cc
场景: UBX 拓扑 (MESH_1D_CLOS), 两个 template 并行
逻辑: 按端口比例分数据, 交替下发
  portNum0 = rankSize - 1  (mesh)
  portNum1 = CLOS_PORT_NUM (clos)
  data0 = counts * portNum0 / (portNum0 + portNum1)
  data1 = counts - data0
  while (data0 > 0 || data1 > 0):
    temp0->KernelRun(data0_chunk)
    temp1->KernelRun(data1_chunk)
```

Sequence2Die (双 template 串行):
```
文件: ins_v2_allreduce_sequence2die_executor.cc
场景: 双 die 拓扑
逻辑: AllReduce = ReduceScatter + AllGather, 顺序执行
  CalcSliceInfoAllReduce(currDataCount)  // 按 rankSize 均分
  for loop in [0, loopTimes):
    template_RS->KernelRun()  // Step 1: ReduceScatter
    template_AG->KernelRun()  // Step 2: AllGather
  资源: max(req0, req1) (非相加, 因为串行复用)
```

Parallel (双 template 并行+同步):
```
文件: ins_all_reduce_parallel_executor.cc
场景: 多层拓扑 (intra + inter), 两个 template 并行但需同步
逻辑: 50:50 分数据, 两阶段交替
  for loop in [0, loopTimes):
    Stage 0:
      PreSync()
      temp_intra->KernelRun(part0)
      temp_inter->KernelRun(part1)
      PostSync()
    Stage 1 (换轴):
      PreSync()
      temp_inter->KernelRun(part0)  // 角色对调
      temp_intra->KernelRun(part1)
      PostSync()
  资源: req0 + req1 + 2 notify (同步用)
```

### 3.4 Executor 注册

文件: `hccl/src/ops/all_reduce/executor/ins_v2_all_reduce_sole_executor.cc:171-200`

AllReduce 共注册 22 个 executor 变体:
- 14 个 REGISTER_EXEC_V2 (单 template)
- 8 个 REGISTER_EXECUTOR_BY_TWO_TEMPS (双 template)

注册格式:
```cpp
REGISTER_EXEC_V2(HCCL_CMD_ALLREDUCE, InsAllReduceMesh1DOneShot,
    InsV2AllReduceSoleExecutor, TopoMatch1D, InsTempAllReduceMesh1DOneShot)
// (操作类型, 算法名=selector返回值, Executor类, TopoMatcher, Template类)
```

---

## 4. hccl 侧: Template 任务生成

### 4.1 四代 Template 基类

```
AlgTemplateBase (V1/SDMA, 已过时)
InsAlgTemplateBase (V2/AICPU, 继承 AlgTemplateBase)
AivAlgTemplateBase (AIV, 独立)
CcuAlgTemplateBase (CCU, 独立)
```

四代完全独立不继承(不同处理器资源模型差异太大)。

### 4.2 AICPU Template 执行模型

以 OneShot 为例 (ins_temp_all_reduce_mesh_1D_one_shot.cc):

```
KernelRun(OpParam, TemplateDataParams, TemplateResource):
  获取 threads (主线程 + N-1 从线程)
  主线程: LocalCopy(usrIn -> usrOut)  // 本地数据先拷到输出
  从线程[1..rankSize-1]:
    // 与第 i 个 peer 交换数据
    HcommChannelNotifyRecordOnThread(ACK)      // 通知 peer 本端已就绪
    HcommChannelNotifyWaitOnThread(ACK)        // 等待 peer 就绪
    HcommWriteOnThread(peer_cclBuf, local_data) // 写数据到 peer 的 scratch buffer
    HcommChannelNotifyRecordOnThread(DATA_SIGNAL) // 通知 peer 数据已到
  主线程:
    for i in [1..rankSize-1]:
      HcommChannelNotifyWaitOnThread(DATA_SIGNAL) // 等待所有 peer 数据到达
    LocalReduce(cclBuf[1..N] -> usrOut)           // 规约所有 peer 的数据
```

TwoShot = ReduceScatter + AllGather 两步:
```
Step 1 RunReduceScatter(): 每 rank 与其他 rank 执行部分规约
Step 2 RunAllGather(): 将规约结果全局广播
需要 rankSize 个线程并行
```

### 4.3 AIV Template 执行模型

文件: `aiv_temp_all_reduce_mesh_1D_oneshot.cc`

```
KernelRun():
  构造 AivOpArgs:
    cmdType = HCCL_CMD_ALLREDUCE
    input/output = base_offset + ptr
    rank, rankSize, count, dataType, op
    topo[] = subCommRanks 列表
    numBlocks = min(rankSize+1, numBlocksLimit)  // AIV 核数
  ExecuteKernelLaunch(aivOpArgs)
    -> aclrtLaunchKernelWithHostArgs()  // 提交到 ACL Runtime
```

Device 侧 kernel 实现 (aiv_all_reduce_mesh_1d_oneshot.h):
```
Producer(): 将本 rank 数据写到所有 peer 的 GM buffer
  CpGM2GM(peer_output, local_input, len)
  Record(targetRank, myRank, tag)  // GM flag 通知

Consumer(): 接收所有 peer 数据并规约
  for waitRank in [1..rankSize-1]:
    WaitFlag(myRank, waitRank, tag)  // 等待 GM flag
    CpGM2GM(output, peer_data, len, reduceOp)  // 规约
```

### 4.4 CCU Template 执行模型

文件: `ccu_temp_all_reduce_mesh_1D_one_shot.cc`

```
CalcRes():
  创建 CcuKernel (微码 kernel)
  slaveThreadNum = 0 (CCU 不需要额外线程)

KernelRun():
  构造 CcuTaskArg (inputAddr, outputAddr, sliceSize, token)
  HcclCcuKernelLaunch(comm, thread, kernel, taskArg)
    -> [穿越 hcomm 边界]
```

CCU Kernel 内部 (ccu_kernel_all_reduce_mesh1d_one_shot.cc):
```
Algorithm():
  InitResource()    // 为每个 peer 创建 channel variable
  LoadArgs()        // Load(input, output, token, groupOpSize)
  Presync()         // NotifyRecord + NotifyWait (全局屏障)
  DoGroupReduce()   // CCU 硬件 Group Reduce 指令
  Postsync()        // NotifyRecord + NotifyWait (完成同步)
```

---

## 5. hccl -> hcomm 边界穿越

### 5.1 边界接口清单

共 22 个 C ABI 接口, 分 4 类:

AICPU 数据面 (10 个, 返回 int32_t):
```
声明: hcomm/include/hccl/hcomm_primitives.h
实现: hcomm/src/framework/communicator/impl/independent_op/data_api/hccl_api_data_aicpu_ts.cc

HcommChannelNotifyRecordOnThread(ThreadHandle, ChannelHandle, uint32_t notifyIdx)
HcommChannelNotifyWaitOnThread(ThreadHandle, ChannelHandle, uint32_t notifyIdx, uint32_t timeout)
HcommWriteOnThread(ThreadHandle, ChannelHandle, void* dst, const void* src, uint64_t len)
HcommReadOnThread(ThreadHandle, ChannelHandle, void* dst, const void* src, uint64_t len)
HcommWriteReduceOnThread(ThreadHandle, ChannelHandle, void* dst, const void* src, uint64_t len, HcommDataType, HcommReduceOp)
HcommReadReduceOnThread(ThreadHandle, ChannelHandle, void* dst, const void* src, uint64_t len, HcommDataType, HcommReduceOp)
HcommLocalCopyOnThread(ThreadHandle, void* dst, const void* src, uint64_t len)
HcommLocalReduceOnThread(ThreadHandle, void* dst, const void* src, uint64_t len, HcommDataType, HcommReduceOp)
HcommThreadNotifyRecordOnThread(ThreadHandle src, ThreadHandle dst, uint32_t notifyIdx)
HcommThreadNotifyWaitOnThread(ThreadHandle, uint32_t notifyIdx, uint32_t timeout)
```

CCU 控制面 (3 个, 返回 HcclResult):
```
声明: hcomm/pkg_inc/hcomm/ccu/hccl_ccu_res.h
实现: hcomm/src/framework/next/coll_comms/api_c_adpt/coll_comm_res_c_adpt.cc

HcclCcuKernelLaunch(HcclComm, ThreadHandle, CcuKernelHandle, void* taskArgs)
HcclCcuKernelRegister(HcclComm, CcuKernelHandle*, void* kernelArg, void* algResInfo)
HcclCcuKernelRegisterFinish(HcclComm)
```

AIV (1 个):
```
hccl 内部: ExecuteKernelLaunch(AivOpArgs)
  -> aclrtLaunchKernelWithHostArgs()  // 直接调用 ACL Runtime, 不经过 hcomm
```

资源分配 (8 个, 返回 HcclResult):
```
声明: hcomm/include/hccl/hccl_res.h

HcclGetHcclBuffer(HcclComm, void**, uint64_t*)          // 获取 CCL buffer
HcclThreadAcquire(HcclComm, CommEngine, u32, u32, ThreadHandle*)   // 获取线程
HcclThreadAcquireWithStream(HcclComm, CommEngine, aclrtStream, u32, ThreadHandle*)
HcclChannelAcquire(HcclComm, CommEngine, HcclChannelDesc*, u32, ChannelHandle*)  // 获取通道
HcclChannelGetHcclBuffer(HcclComm, ChannelHandle, void**, uint64_t*)  // 远端 buffer
HcclChannelGetRemoteMems(HcclComm, ChannelHandle, u32*, CommMem**, char***)  // 注册内存
HcclThreadExportToCommEngine(HcclComm, ThreadHandle, aclrtStream)
HcclMemcpyCtxHostToDevice(HcclComm, ...)  // 资源上下文 H2D 序列化
```

### 5.2 句柄类型

所有句柄都是 uint64_t 不透明类型:
- ThreadHandle: hcomm 侧 reinterpret_cast 为 Thread*
- ChannelHandle: hcomm 侧 reinterpret_cast 为 Channel* 或通过 map 查找
- CcuKernelHandle: hcomm 侧 reinterpret_cast 为 CcuKernel*
- HcclComm: hcomm 侧 reinterpret_cast 为 HcclCommunicator*

### 5.3 接口设计特点

1. 全部 C ABI (extern "C"), 无 C++ 类型暴露
2. 数据面函数 (Hcomm*) 返回 int32_t, 控制面函数返回 HcclResult
3. 所有远端操作通过 (ThreadHandle, ChannelHandle) 二元组定位: 线程绑定执行流, 通道绑定链路
4. 无回调, 纯同步语义 (操作提交到 stream 后返回, 但 stream 本身异步)
5. hccl 侧有独立的类型映射: HcclDataType <-> HcommDataType, HcclReduceOp <-> HcommReduceOp

---

## 6. hcomm 内部: Framework -> Platform

### 6.1 通道数据传输路径

```
HcommWriteOnThread(thread, channel, dst, src, len)
  |
  +- [A5 设备] -> UbTransportLiteImpl::Write()
  |                直接 UB/URMA 通道
  |
  +- [AICPU/AIV] -> HcclRemoteWrite()
       |
       +- hccl_primitive_remote.cc:18-31
       |  Transport::WriteAsync(dst, src, len)
       |
       +- transport_pub.h (TransportBase 虚函数)
          |
          +- TransportP2pBase   -> PCIe DMA (P2P 链路)
          +- TransportIbverbsBase -> ibv_post_send (RoCE/RDMA)
          +- TransportHccsBase   -> HCCS 链路
          +- TransportOnchipBase -> 片内通道
```

DMA 模式选择:
- P2P 默认 GET (目的端拉取)
- DEV_NET 默认 PUT (源端推送)
- HcommWrite 对应 PUT, HcommRead 对应 GET

### 6.2 本地操作路径

```
HcommLocalCopyOnThread(thread, dst, src, len)
  |
  +- [A5] -> Thread::LocalCopy()
  +- [AICPU/AIV] -> HcclLocalCopy()
       -> DispatcherPub::LocalCopyAsync()
       -> SDMA 任务下发

HcommLocalReduceOnThread(thread, dst, src, len, dataType, op)
  |
  +- [A5] -> Thread::LocalReduce()
  +- [AICPU/AIV] -> HcclLocalReduce()
       -> DispatcherPub::LocalReduceAsync()
       -> 支持: SUM/MAX/MIN x FP32/FP16/INT8/INT16/INT32/BFP16
       -> SDMA 或 InlineReduce 执行
```

InlineReduce 降级逻辑:
- 优先使用 InlineReduce (硬件加速)
- 若 DevCapability 不支持 -> 降级为 LocalReduce (两步: 搬运 + 规约)
- RoCE 链路不支持 ReadReduce

### 6.3 Notify 同步路径

```
HcommChannelNotifyRecordOnThread(thread, channel, notifyIdx)
  |
  +- [A5] -> UbTransportLiteImpl::Post()
  +- [AICPU/AIV] -> HcclRemoteNotifyRecord()
       -> Transport::Post(notifyIdx)
       -> 远端 Notify 寄存器写操作

HcommChannelNotifyWaitOnThread(thread, channel, notifyIdx, timeout)
  |
  +- [A5] -> UbTransportLiteImpl::Wait()
  |          (注: A5 强制忽略 timeout 参数)
  +- [AICPU/AIV] -> HcclRemoteNotifyWait()
       -> Transport::Wait(notifyIdx)
       -> 轮询本地 Notify 寄存器
```

Notify 三种实现 (按 deviceId 组合自动选择):
- RtsNotify: 硬件 Notify (本地/远端均可)
- BareNotify: 空实现 (仅占位)
- EschedNotify: 事件驱动 (新架构)

### 6.4 CCU 路径

```
HcclCcuKernelLaunch(comm, thread, kernel, taskArgs)
  -> coll_comm_res_c_adpt.cc:299-340
  -> Thread* 获取 stream
  -> CcuKernelMgr 获取 kernel 对象
  -> kernel->GeneTaskParam(taskArgs)  // 生成微码参数
  -> LaunchCcuTasks(stream, kernel)   // 提交到 CCU 硬件
```

### 6.5 CCL Buffer 数据流转

AllReduce 中数据在 3 个 buffer 间流转:

```
User Input Buffer
  | HcommLocalCopy (input -> output)
  v
User Output Buffer
  | HcommWrite (output -> peer_cclBuf)  [OneShot: 写到 peer 的 scratch]
  v
CCL Scratch Buffer (per-rank)
  | HcommChannelNotifyWait (等待 peer 数据到达)
  | HcommLocalReduce (cclBuf[1..N] -> output)  [规约所有 peer 数据]
  v
User Output Buffer (最终结果)
```

TwoShot 的流转更复杂:
```
Step 1 (ReduceScatter):
  input -> cclBuf (本 rank slice)
  cclBuf -> peer_cclBuf (交换)
  peer_cclBuf 规约 -> cclBuf

Step 2 (AllGather):
  cclBuf (结果 slice) -> peer_output (广播)
```

---

## 7. 端到端完整调用链

以 8 卡 AICPU OneShot AllReduce 为例:

```
用户调用:
  HcclAllReduce(sendBuf, recvBuf, count, FP32, SUM, comm, stream)

hccl API 层 (all_reduce_op.cc):
  CheckAllReduceInputPara()
  AllReduceOutPlace()
    OpParam{inputPtr=sendBuf, outputPtr=recvBuf, count, FP32, SUM, ALLREDUCE}

hccl Selector (op_common.cc + all_reduce_auto_selector.cc):
  HcclGetOpExpansionMode() -> AICPU_TS
  HcclCalcTopoInfo() -> MESH_1D, topoLevelNums=1, rankSize=8
  SelectAicpuAlgo():
    dataSize = count * 4 (FP32)
    dataSize <= 8MB -> algName = "InsAllReduceMesh1DOneShot"

hccl Executor (op_common.cc + sole_executor.cc):
  CollAlgExecRegistryV2::GetAlgExec(ALLREDUCE, "InsAllReduceMesh1DOneShot")
    -> InsV2AllReduceSoleExecutor<TopoMatch1D, InsTempAllReduceMesh1DOneShot>

  HcclGetAlgRes():
    executor->CalcAlgHierarchyInfo() -> [rank 0..7] 单层
    executor->CalcRes():
      template->CalcRes() -> 7 slave threads, 7 channels, 8 notifies
    HcclAllocAlgResourceAICPU():
      [穿越边界] HcclGetHcclBuffer(comm) -> cclBufAddr
      [穿越边界] HcclThreadAcquire(AICPU_TS, 0, 7) -> mainThread + 7 slaveThreads
      [穿越边界] HcclChannelAcquire(AICPU_TS, channelDescs, 7) -> 7 channels

  executor->Orchestrate(param, resCtx):
    RestoreChannelMap()
    OrchestrateLoop():
      maxCountPerLoop = min(UB_LIMIT, scratchSize / multiplier)
      loop 0 (假设一次够):
        template->KernelRun():
          主线程: [边界] HcommLocalCopyOnThread(sendBuf -> recvBuf)
          从线程 1..7 (并行):
            [边界] HcommChannelNotifyRecordOnThread(ACK)
            [边界] HcommChannelNotifyWaitOnThread(ACK)
            [边界] HcommWriteOnThread(peer_cclBuf, myData, sliceSize)
            [边界] HcommChannelNotifyRecordOnThread(DATA_SIGNAL)
          主线程:
            [边界] HcommChannelNotifyWaitOnThread(DATA_SIGNAL) x 7
            [边界] HcommLocalReduceOnThread(cclBuf[1..7] + recvBuf -> recvBuf)

hcomm Framework (hccl_api_data_aicpu_ts.cc):
  ThreadHandle -> Thread* (reinterpret_cast)
  ChannelHandle -> 链路查找

hcomm Platform (hccl_primitive_remote.cc + transport_*_pub.h):
  Write: Transport::WriteAsync() -> TransportIbverbsBase::PostSend() -> ibv_post_send
  Notify: Transport::Post() -> 远端寄存器写
  Wait: Transport::Wait() -> 本地寄存器轮询
  LocalCopy: DispatcherPub::LocalCopyAsync() -> SDMA
  LocalReduce: DispatcherPub::LocalReduceAsync() -> SDMA + InlineReduce

硬件:
  数据搬运: ibv WQE -> RoCE NIC -> 网络 -> 对端 NIC -> PCIe -> HBM
  通知同步: Notify 寄存器 (硬件原子操作)
  本地操作: SDMA 引擎 (DMA 拷贝 + 向量规约)
```

---

## 8. 关键设计观察

### 8.1 分层解耦

hccl 和 hcomm 通过 22 个 C ABI 函数完全解耦:
- hccl 负责"做什么" (算法选择 + 执行编排)
- hcomm 负责"怎么做" (通信原语实现 + 硬件驱动)
- 接口全部基于不透明句柄, 无类型泄漏

### 8.2 Selector-Executor-Template 三层分离

```
Selector: 策略决策 (拓扑 x 数据量 x 引擎 -> 算法名)
Executor: 资源编排 (资源计算 + 数据切分 + 循环控制)
Template: 任务生成 (单次通信的具体操作序列)
```

三层通过 Registry + 字符串名称松耦合:
Selector 返回 algName -> Registry 查找 Executor(绑定 Template 类型)

### 8.3 三种设备的编程模型差异

AICPU: 多线程模型, Host 侧编排, 通过 wrapper 函数调用 hcomm 原语
AIV: 单 kernel 模型, Host 构造参数, Device 侧 GM flag 同步
CCU: 编译器模型, Host 构造微码 IR, CCU 硬件执行

### 8.4 数据切分策略

所有 Executor 都处理 "数据量超过单次处理上限" 的场景:
- 按 maxCountPerLoop 切分 (受 UB 大小、scratch buffer、SDMA 4GB 限制)
- Loop 内 template 执行固定大小的通信
- Loop 间无依赖 (pipeline 化)

### 8.5 A5 设备分叉

hcomm 数据面的每个函数都有 IsDeviceA5() 分叉:
- A5 (910_95): UbTransportLiteImpl, 直接 UB/URMA 通道
- 非 A5: 传统 Transport 继承体系 (P2P/IBVerbs/HCCS 等)

---

## 9. 发现的问题

### 确认 Bug

1. Selector FP64 重复/INT64 遗漏: 3+ 个 selector 文件中 FP64 检查重复, INT64 遗漏
   (all_reduce_auto_selector.cc + reduce_scatter + all_gather)

2. NHR recv slice 用错 send 变量: ins_temp_all_reduce_nhr.cc 中 2 处

3. Parallel CalcSendDataSize 赋值错误: ins_all_reduce_parallel_executor.cc

4. CCU multi_jetty goSize 选择反了: ccu_kernel_all_reduce_mesh1d_multi_jetty.cc

5. A5 超时忽略: HcommChannelNotifyWait 中 A5 设备强制忽略 timeout 参数
   (可能导致死锁无法被超时检测)

### 设计问题

1. 22 种算法变体 x 4 种 Executor = 大量组合, 维护成本高

2. 环境变量决定执行模式 (HCCL_AICPU_UNFOLD 等) 意味着运行时无法动态切换

3. AIV 路径不经过 hcomm, 直接调用 ACL Runtime, 绕过了 hcomm 的抽象层
   (可能导致 AIV 路径缺少 hcomm 提供的监控/调试/容错能力)

4. reinterpret_cast 句柄转换无类型安全检查 (22 个边界接口全部如此)
