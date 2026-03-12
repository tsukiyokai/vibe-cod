# Phase 5.4: Send/Recv Point-to-Point Slice Analysis

## Overview

本文档追踪点对点通信(Send/Recv/BatchSendRecv)从用户API到硬件执行的完整跨仓路径,
并与集合通信(AllReduce)路径做结构性对比。

P2P通信是集合通信的基础原语。理解P2P路径有助于理解:
- 最简化的跨仓调用链(无算法选择/多层分解的复杂性)
- Write vs Read两种DMA语义的差异
- 批量操作(BatchSendRecv)的死锁预防机制

端到端路径对比:
```
P2P (Send/Recv):
  User API (hccl)
    -> Op Entry: 三重门控
      -> Selector: 硬编码唯一算法
        -> Executor: 单层循环
          -> wrapper: SendWrite/RecvWrite (3个hcomm原语)
            -> [hccl/hcomm边界] C ABI
              -> Transport::WriteAsync/ReadAsync
                -> Dispatcher::MemcpyAsync
                  -> Hardware DMA

AllReduce (对比):
  User API (hccl)
    -> Op Entry: 三重门控 (相同)
      -> Selector: 22种算法5路分派 (复杂)
        -> Executor: 4种策略多层编排 (复杂)
          -> Template: 3种设备模型 (复杂)
            -> wrapper: 多种原语组合
              -> [hccl/hcomm边界] C ABI (相同)
                -> Transport (相同)
                  -> Dispatcher (相同)
```

---

## 1. hccl侧: Send完整路径

### 1.1 C ABI入口

文件: `hccl/src/ops/send/send_op.h:22-23`

```cpp
extern "C" HcclResult HcclSend(
    void *sendBuf, uint64_t count, HcclDataType dataType,
    uint32_t destRank, HcclComm comm, aclrtStream stream);
```

### 1.2 三重门控 (send_op.cc:41-51)

```
HcclSend()
  |-- CheckHCCLIndependentOp()    // 门控1: 独立操作模式?
  |-- hrtGetDeviceType()          // 门控2: 设备类型 == DEV_TYPE_910_95?
  |-- GetWorkflowMode()           // 门控3: WorkflowMode == OP_BASE?
  |
  |-- 全部通过 -> SendExec() (新路径, 910_95专用)
  |-- 任一不通过 -> HcclSendInner() (降级路径, 非910_95)
```

与AllReduce完全相同的门控模式。区别: AllReduce多一步enableDetour检查。

### 1.3 参数校验与OpParam构建 (send_op.cc:55-135)

```cpp
// 参数校验序列 (send_op.cc:55-70)
CheckSendInputPara(comm, sendBuf);  // comm和buf非空
count > 0;                           // 非零计数
tag = "Send_" + commName + "_" + userRank + "_" + destRank;
userRank != destRank;                // 不能自己发给自己
CheckCount(count); CheckDataType(dataType);

// OpParam构建 (send_op.cc:108-135)
OpParam param;
param.opType = HCCL_CMD_SEND;
param.inputPtr = sendBuf;        // Send有input
param.outputPtr = nullptr;       // Send无output
param.sendRecvRemoteRank = destRank;
param.enableDetour = false;      // P2P无绕路
param.opMode = OpMode::OPBASE;
```

### 1.4 算法选择 (send_auto_selector.cc:15-23)

```cpp
SelectorStatus SendAutoSelector::SelectAicpuAlgo(...) const override {
    selectAlgName = "InsSend";       // 硬编码唯一算法
    return SelectorStatus::MATCH;
}
REGISTER_SELECTOR_BY_OPTYPE(HCCL_CMD_SEND, 18, SendAutoSelector);
```

与AllReduce的22种算法5路分派形成鲜明对比。P2P无需算法选择。

### 1.5 Executor: InsSendExecutor

文件: `hccl/src/ops/send/executor/ins_send_executor.h/cc` (6文件/459行)

```cpp
class InsSendExecutor : public InsCollAlgBase {
    // CalcAlgHierarchyInfo(): 单层, 所有rank在一个group
    // CalcRes(): 创建1条channel到remoteRank
    // Orchestrate(): 分片循环SendWrite
};
```

#### CalcAlgHierarchyInfo (ins_send_executor.cc:39-60)

```
固定单层拓扑: infos[0][0] = [0, 1, ..., rankSize-1]
// 与AllReduce的多层hierarchy(level0/1/2)完全不同
```

#### CalcRes (ins_send_executor.cc:62-79)

```
请求1条thread + 1条channel到remoteRank
// 与AllReduce需要多条channel到多个rank不同
```

#### Orchestrate: OPBASE模式 (ins_send_executor.cc:134-159)

```
OPBASE模式数据流:
  input buffer --[SendWrite循环]--> 对端CCL buffer
                                      |
                              对端RecvWrite后LocalCopy
                                      |
                              对端output buffer

// 按 min(UB_MAX_DATA_SIZE, remoteCclMem.size) 分片
while (dataCountToSend > 0) {
    transferCount = min(remaining, maxLoopTransCount_);
    srcSlice = {inputPtr, currentOffset, transferSize};
    dstSlice = {dstBufferPtr, 0, transferSize};  // offset始终0
    SendWrite(sendDataInfo, thread);
    currentOffset += transferSize;
}
```

#### Orchestrate: OFFLOAD模式 (ins_send_executor.cc:108-133)

```
OFFLOAD模式数据流:
  input buffer --[SendWrite]--> 对端output buffer (直接)
// 图模式下buffer预分配, 无需CCL buffer中转
```

### 1.6 数据传输原语: SendWrite

文件: `hccl/src/ops/op_common/template/wrapper/alg_data_trans_wrapper.cc:15-37`

```cpp
HcclResult SendWrite(const DataInfo &sendInfo, const ThreadHandle &thread) {
    // 1. 等待对端ready (对端RecvWrite已发ACK)
    HcommChannelNotifyWaitOnThread(thread, channel, NOTIFY_IDX_ACK, timeout);
    // 2. 写数据到对端
    for (each slice)
        HcommWriteOnThread(thread, channel, dst, src, size);
    // 3. 通知对端数据已到
    HcommChannelNotifyRecordOnThread(thread, channel, NOTIFY_IDX_DATA_SIGNAL);
}
```

Notify同步协议(Send端):
```
Send:  Wait(ACK) -> Write(data) -> Record(DATA_SIGNAL)
Recv:  Record(ACK) -> Wait(DATA_SIGNAL) -> [LocalCopy]
```

---

## 2. hccl侧: Recv完整路径

### 2.1 C ABI入口

文件: `hccl/src/ops/recv/recv_op.h:21-22`

```cpp
extern "C" HcclResult HcclRecv(
    void *recvBuf, uint64_t count, HcclDataType dataType,
    uint32_t srcRank, HcclComm comm, aclrtStream stream);
```

### 2.2 门控与参数 (recv_op.cc:37-136)

与Send完全镜像:
```cpp
OpParam param;
param.opType = HCCL_CMD_RECEIVE;
param.inputPtr = nullptr;          // Recv无input (vs Send无output)
param.outputPtr = recvBuf;         // Recv有output
param.sendRecvRemoteRank = srcRank;
```

### 2.3 算法选择

```cpp
// recv_auto_selector.cc:21
selectAlgName = "InsRecv";  // 硬编码
```

### 2.4 Executor: InsRecvExecutor

文件: `hccl/src/ops/recv/executor/ins_recv_executor.cc` (6文件/473行)

#### Orchestrate: OPBASE模式 (ins_recv_executor.cc:140-169)

```
OPBASE模式数据流:
  对端input --[对端SendWrite]--> 本端CCL buffer
                                      |
                              [RecvWrite同步]
                                      |
                              [LocalCopy]
                                      |
                              本端output buffer

while (dataCountToRecv > 0) {
    RecvWrite(recvInfo, thread);     // 同步: Record ACK + Wait DATA_SIGNAL
    LocalCopy(thread, cclSlice, outputSlice);  // CCL buf -> output buf
}
```

### 2.5 数据传输原语: RecvWrite + LocalCopy

```cpp
// alg_data_trans_wrapper.cc:39-46
HcclResult RecvWrite(const DataInfo &recvInfo, const ThreadHandle &thread) {
    // 1. 通知对端本端ready
    HcommChannelNotifyRecordOnThread(thread, channel, NOTIFY_IDX_ACK);
    // 2. 等待对端数据到达
    HcommChannelNotifyWaitOnThread(thread, channel, NOTIFY_IDX_DATA_SIGNAL, timeout);
}

// alg_data_trans_wrapper.cc:361-376
HcclResult LocalCopy(const ThreadHandle &thread, ...) {
    HcommLocalCopyOnThread(thread, dstOut, srcIn, size);
}
```

### 2.6 Send/Recv协议时序图

```
Send端                          Recv端
  |                               |
  |  <--- Wait(ACK) ---          |  --- Record(ACK) --->
  |                               |
  |  --- Write(data) --->         |
  |                               |
  |  --- Record(DATA_SIGNAL) -->  |  <--- Wait(DATA_SIGNAL) ---
  |                               |
  |                               |  LocalCopy(CCL -> output)
```

---

## 3. hccl侧: BatchSendRecv路径

### 3.1 C ABI入口

文件: `hccl/src/ops/batch_send_recv/batch_send_recv_op.h`

```cpp
extern "C" HcclResult HcclBatchSendRecv(
    HcclSendRecvItem *sendRecvInfo, uint32_t itemNum,
    HcclComm comm, aclrtStream stream);

// HcclSendRecvItem (hccl_types.h:193-199):
typedef struct HcclSendRecvItemDef {
    HcclSendRecvType sendRecvType;  // HCCL_SEND or HCCL_RECV
    void *buf;
    uint64_t count;
    HcclDataType dataType;
    uint32_t remoteRank;
} HcclSendRecvItem;
```

### 3.2 算法选择

```cpp
// batch_send_recv_auto_selector.cc
selectAlgName = "InsBatchSendRecv";  // 硬编码
```

### 3.3 Executor: InsV2BatchSendRecvExecutor

文件: `hccl/src/ops/batch_send_recv/executor/ins_v2_batch_send_recv_executor.cc` (379+行)

这是P2P通信中最复杂的执行器。

#### CalcRes (行32-78)

```
与Send/Recv的区别: 为每个remoteRank创建2条channel (send和recv分离)
channelNumPerRankPair_ = 2
```

#### Orchestrate (行379-408)

```
Orchestrate()
  |-- GetPairWiseList()           // 死锁预防排序
  |-- ProcessSelfSendRecvTasks()  // 自发自收特殊处理
  |-- CalcSendSlices()            // 按maxTmpMemSize切分
  |-- CalcRecvSlices()
  |-- RunLoopSendRecv()           // 并行执行
```

### 3.4 死锁预防: Pair-Wise排序 (行139-207)

```
核心思想: 通过不对称排序打破循环等待

Send排序 (SortSendItems):
  aFlag = (remoteRank <= myRank_) ? (remoteRank + rankSize_) : remoteRank
  按aFlag降序 -> 优先向rank > myRank_的对端发送

Recv排序 (SortRecvItems):
  bFlag = (remoteRank < myRank_) ? (remoteRank + rankSize_) : remoteRank
  按bFlag升序 -> 优先从rank > myRank_的对端接收

注意不对称: Send用"<=", Recv用"<"
  -> 确保remoteRank == myRank_的任务被正确分离到selfDeque

例(4 rank, myRank=1):
  Send优先: rank2, rank3, rank0 (先高后低)
  Recv优先: rank2, rank3, rank0 (先高后低)
  -> rank1和rank2: rank1先Send给rank2, rank2先Recv from rank1 -> 不死锁
```

### 3.5 使用Read语义(vs Send/Recv的Write语义)

```
单Send/Recv使用Write语义:
  Send: Write数据到对端CCL buffer
  Recv: 被动等待数据到达, 然后LocalCopy

BatchSendRecv使用Read语义:
  Send端: LocalCopy(user buf -> local CCL buf) + SendRead()
    SendRead: Record(ACK告诉对端可读) -> Wait(DATA_SIGNAL等对端读完)
  Recv端: RecvRead()
    RecvRead: Wait(ACK等对端就绪) -> Read(从对端CCL buf读) -> Record(DATA_SIGNAL)

为什么BatchSendRecv选择Read语义?
  1. 接收端主动拉取, 天然流控(不会被发送端淹没)
  2. 多对多场景下, 一个发送端的CCL buffer可被多个接收端独立读取
  3. 配合pair-wise排序, 避免write语义下的buffer竞争
```

### 3.6 自发自收处理 (行80-104)

```
remoteRank == myRank_的任务:
  -> 直接LocalCopy(send buf -> recv buf)
  -> 无需网络通信
```

---

## 4. 边界穿越: hccl -> hcomm

### 4.1 P2P使用的hcomm原语

Send/Recv(Write语义)使用3个原语:
```
HcommWriteOnThread(thread, channel, dst, src, len)        // 数据写入
HcommChannelNotifyRecordOnThread(thread, channel, idx)     // 发送通知
HcommChannelNotifyWaitOnThread(thread, channel, idx, timeout)  // 等待通知
```

BatchSendRecv(Read语义)额外使用:
```
HcommReadOnThread(thread, channel, dst, src, len)          // 数据读取
HcommLocalCopyOnThread(thread, dst, src, len)              // 本地拷贝
```

### 4.2 与AllReduce边界接口的对比

```
AllReduce使用22个C ABI接口:
  10个数据面: Write/Read + WriteReduce/ReadReduce + LocalCopy + ...
  3个CCU面: HcommCcuSendNotify/Wait/...
  1个AIV面: HcommAivModeSet
  8个资源面: GetThread/GetChannel/MemcpyCtx/...

P2P(Send/Recv)仅使用5个C ABI接口:
  3个数据面: Write/Read/LocalCopy
  2个控制面: NotifyRecord/NotifyWait
  0个CCU/AIV面

原因: P2P仅有AICPU执行模式, 无CCU/AIV设备模板
```

### 4.3 参数传递方式

```
所有句柄均为uint64_t不透明指针(reinterpret_cast):
  ThreadHandle = reinterpret_cast<Thread*>
  ChannelHandle = reinterpret_cast<Transport*> (非A5)
                  reinterpret_cast<UbTransportLiteImpl*> (A5)
```

---

## 5. hcomm侧: P2P内部路径

### 5.1 C ABI实现位置

文件: `hcomm/src/framework/communicator/impl/independent_op/data_api/hccl_api_data_aicpu_ts.cc`

所有数据面C ABI函数在此文件实现, 采用A5/非A5分叉:

```cpp
// HcommWriteOnThread (行286)
if (threadPtr->IsDeviceA5()) {
    // A5路径: 直接UbTransportLiteImpl::Write
    auto *ubTransportLitePtr = reinterpret_cast<UbTransportLiteImpl*>(channel);
    ubTransportLitePtr->Write(locRmaBuf, rmtBuf, *streamLitePtr);
} else {
    // 非A5路径: HcclRemoteWrite -> Transport::WriteAsync
    HcclRemoteWrite(stream, transport, &rmtBuf, &locBuf);
}
```

### 5.2 非A5路径: 三层委托

```
C ABI函数
  -> HcclRemoteWrite/Read (platform/comm_primitive/hccl_primitive_remote.cc)
    -> Transport::WriteAsync/ReadAsync (resource/transport/)
      -> DispatcherPub::MemcpyAsync (task/dispatcher_common.cc)
        -> Hardware DMA
```

#### 中间适配层 (hccl_primitive_remote.cc:18-46)

```cpp
HcclResult HcclRemoteWrite(StreamHandle streamHandle, HcclMemTransport memTransport,
                           HcclBuf *rmtBuf, HcclBuf *locBuf) {
    Stream *stream = reinterpret_cast<Stream*>(streamHandle);
    return reinterpret_cast<Transport*>(memTransport)->WriteAsync(remoteBuf, localBuf, *stream);
}

HcclResult HcclRemoteRead(StreamHandle streamHandle, HcclMemTransport memTransport,
                          HcclBuf *locBuf, HcclBuf *rmtBuf) {
    return reinterpret_cast<Transport*>(memTransport)->ReadAsync(localBuf, remoteBuf, *stream);
}
```

#### Transport层 (transport_p2p_pub.h:1504-1581)

```cpp
HcclResult TransportP2p::WriteAsync(Buffer &remoteBuf, Buffer &localBuf, Stream &stream) {
    return HcclD2DMemcpyAsync(dispatcher_, remoteDevMem, localDevMem,
        stream, machinePara_.remoteWorldRank, transportAttr_.linkType);
}

HcclResult TransportP2p::ReadAsync(Buffer &localBuf, Buffer &remoteBuf, Stream &stream) {
    return HcclD2DMemcpyAsync(dispatcher_, dstDevMem, srcDevMem,
        stream, machinePara_.remoteWorldRank, transportAttr_.linkType);
}
```

Write和Read最终都调用HcclD2DMemcpyAsync, 仅src/dst参数顺序不同。
实际DMA方向由linkType和硬件决定。

#### Dispatcher层 (dispatcher_common.cc:120-128)

```cpp
HcclResult HcclD2DMemcpyAsync(HcclDispatcher dispatcherPtr, DeviceMem &dst, const DeviceMem &src,
    Stream &stream, const u32 remoteUserRank, const LinkType linkType) {
    return reinterpret_cast<DispatcherPub*>(dispatcherPtr)->MemcpyAsync(
        dst, src, stream, remoteUserRank, linkType);
}
```

Dispatcher根据linkType选择硬件驱动:
- LINK_ONCHIP: 芯片内
- LINK_HCCS: 华为芯片间总线
- LINK_PCIE: PCIe直通
- LINK_ROCE: RDMA over CE

### 5.3 Notify同步路径

```
Record: C ABI -> HcclRemoteNotifyRecord -> Transport::Post -> SignalRecord(远端notify内存)
Wait:   C ABI -> HcclRemoteNotifyWait   -> Transport::Wait -> dispatcher_->SignalWait(本地轮询)
```

Transport::Post/Wait (transport_p2p_pub.h:875-900):
```cpp
HcclResult TransportP2p::Post(u32 notifyIdx, Stream &stream) {
    return SignalRecord(userRemoteNotify_[notifyIdx],
                       userRemoteNotifyAddr_[notifyIdx],
                       userRemoteNotifyOffset_[notifyIdx], stream);
}

HcclResult TransportP2p::Wait(u32 notifyIdx, Stream &stream, const u32 timeOut) {
    return dispatcher_->SignalWait(userLocalNotify_[notifyIdx]->ptr(), stream,
        machinePara_.localUserrank, machinePara_.remoteWorldRank,
        INVALID_VALUE_STAGE, false, userLocalNotify_[notifyIdx]->notifyId_, timeOut);
}
```

### 5.4 A5(910_95)专用路径

文件: `hcomm/src/legacy/unified_platform/resource/transport/aicpu/ub_transport_lite_impl.h`

A5设备跳过Transport/Dispatcher层, 直接操作UB传输引擎:

```cpp
class UbTransportLiteImpl {
    void Post(u32 index, const StreamLite &stream);
    void Wait(u32 index, const StreamLite &stream);
    void Read(const RmaBufferLite &loc, const Buffer &rmt, const StreamLite &stream);
    void Write(const RmaBufferLite &loc, const Buffer &rmt, const StreamLite &stream);
};
```

A5路径的优势: 减少2层间接调用(Transport + Dispatcher), 更低延迟。

### 5.5 P2P与AllReduce在hcomm内共享的路径

```
                          P2P (Send/Recv)    AllReduce
C ABI入口层               hccl_api_data_aicpu_ts.cc    相同文件
中间适配层                HcclRemoteWrite/Read         + HcclReduceAsync
Transport层              WriteAsync/ReadAsync          + WriteReduceAsync
Dispatcher层             MemcpyAsync                   + ReduceAsync
硬件驱动                  相同                          相同

差异仅在: AllReduce额外使用WriteReduce/ReadReduce原语(带归约)
P2P仅使用纯数据搬运原语
```

---

## 6. 非910_95: 降级路径 (HcclSendInner)

### 6.1 降级触发条件

hccl侧send_op.cc:41-50的三重门控任一不通过:
```
非独立操作模式 OR 非DEV_TYPE_910_95 OR 非HCCL_WORKFLOW_MODE_OP_BASE
  -> HcclSendInner() (hcomm仓库实现)
```

### 6.2 hcomm侧降级路径

两条子路径:

#### 路径A: 直接单算子模式

```
HcclSendInner() [hccl weak符号, hcomm实现]
  -> op_base.cc: SendOutPlace() (行2663-2847)
    -> HcclCommunicator::ExecOp(HCCL_CMD_SEND)
      -> hcomm algorithm/: CollSendExecutor
        -> SendRunAsync: 完整6步握手
          TxAck -> RxAck -> TxAsync -> RxAsync -> RxWaitDone -> TxWaitDone
```

#### 路径B: Group批量模式

```
HcclGroupStart()
  HcclSend()  // 入队到hcclKernelPlanner
  HcclRecv()  // 入队
HcclGroupEnd()
  -> doLaunches() (hccl_group.cc:294-429)
    -> 去死锁排序: rank小的先Send后Recv, rank大的先Recv后Send
    -> KernelPlanner统一编排
      -> BatchSendRunAsync: 优化路径(无逐个握手)
        TxPrepare -> TxData
```

### 6.3 降级路径的结构差异

```
910_95新路径(hccl侧):
  3个hcomm原语 x 循环 = 极简
  Executor直接操作channel句柄

非910_95降级路径(hcomm侧):
  hcomm内部完整framework层
  CollSendExecutor继承体系
  6步握手协议(更安全但延迟更高)
```

---

## 7. P2P vs AllReduce: 结构性对比

### 7.1 层级复杂度对比

| 维度 | Send/Recv | BatchSendRecv | AllReduce |
|------|-----------|---------------|-----------|
| 源文件数 | 6 | 6 | 55+ |
| 代码行数 | ~460 | ~540 | ~9768 |
| 算法数量 | 1 (硬编码) | 1 (硬编码) | 22 (5路分派) |
| Executor策略 | 1 (单层循环) | 1 (pair-wise) | 4 (Sole/Parallel/Concurrent/Seq2Die) |
| Template数量 | 0 (无设备模板) | 0 | 14+ (AICPU/AIV/CCU) |
| 设备支持 | AICPU only | AICPU only | AICPU + AIV + CCU |
| 拓扑层级 | 1层 (flat) | 1层 (flat) | 多层 (level0/1/2) |
| 数据分解 | 线性分片 | 线性分片 | 树形分解(RS+AG) |

### 7.2 hcomm原语使用对比

| 原语 | Send/Recv | BatchSendRecv | AllReduce |
|------|-----------|---------------|-----------|
| Write | Y | - | Y |
| Read | - | Y | Y |
| WriteReduce | - | - | Y |
| ReadReduce | - | - | Y |
| LocalCopy | Recv only | Y | Y |
| NotifyRecord | Y | Y | Y |
| NotifyWait | Y | Y | Y |
| CCU系列 | - | - | Y |
| AIV系列 | - | - | Y |

### 7.3 DMA语义对比

```
Send/Recv: Write语义 (PUT模式)
  发送端主动推送数据到接收端CCL buffer
  优点: 简单, 延迟低
  缺点: 接收端被动, 流控依赖Notify握手

BatchSendRecv: Read语义 (GET模式)
  接收端主动从发送端CCL buffer拉取数据
  优点: 接收端控制节奏, 天然流控, 多对多友好
  缺点: 额外一步LocalCopy(发送端: user->CCL)

AllReduce: 主要Write语义 + 部分Read
  根据具体算法和拓扑选择
  DMAReduceFlag_控制: Read/Write/LocalReduce/InlineReduce
```

### 7.4 同步协议对比

```
Send/Recv (2步握手):
  ACK -> DATA_SIGNAL

BatchSendRecv (2步握手, 方向相反):
  ACK -> DATA_SIGNAL (Read方向)

AllReduce (4步握手):
  TxAck -> RxAck -> TxAsync -> RxAsync -> [Wait/Done]
  (hcomm内部CollCommExecutor的完整握手)

或简化为 (hccl侧template):
  ACK -> DATA_SIGNAL (与P2P相同的notify协议)
```

### 7.5 资源需求对比

```
Send/Recv:
  1 thread + 1 channel per remoteRank
  CCL buffer仅OPBASE模式需要

BatchSendRecv:
  1 thread(send) + 1 thread(recv) + 2 channels per remoteRank
  CCL buffer始终需要(Read语义: sender先copy到CCL)

AllReduce:
  N threads + M channels (依赖拓扑)
  CCL buffer: 200MB(input) + 200MB(output) + 1MB(workspace) 三区一体
  额外可能需要: Stream/Notify/AIV资源/CCU微码
```

---

## 8. 反模式与发现

### 8.1 确认bug

1. **Send OPBASE dstSlice offset始终为0** (ins_send_executor.cc:154)
   对端CCL buffer的offset始终为0, 意味着每次循环迭代都覆盖CCL buffer头部。
   对端必须在下一次Write前完成LocalCopy, 否则数据丢失。
   设计意图: Notify协议保证时序(ACK表示对端已完成LocalCopy)。
   不是bug, 但依赖协议正确性, 若notify丢失则数据损坏。

2. **BatchSendRecv: sendDataSilces_ 拼写错误** (ins_v2_batch_send_recv_executor.cc)
   Silces应为Slices, 但不影响功能。

### 8.2 设计观察

1. **P2P无CCU/AIV支持**: Send/Recv仅走AICPU路径。
   意味着P2P无法利用AIV的并行性或CCU的硬件加速。
   可能是设计选择(P2P场景下硬件加速收益小)或尚未实现。

2. **Write vs Read选择不一致**: Send/Recv用Write, BatchSendRecv用Read。
   对用户透明, 但内部实现差异大。两种语义在不同场景各有优势。

3. **降级路径完整性**: 非910_95走hcomm完整framework层, 包含CollSendExecutor的6步握手。
   910_95新路径仅3个原语+2步握手, 延迟更低但可靠性依赖简化的协议。

4. **BatchSendRecv pair-wise排序的不对称(<=/<)**: 确保自发自收被正确分离。
   排序键的设计巧妙但不直观, 缺少注释说明。

### 8.3 与AllReduce共享的代码路径

P2P和AllReduce在以下层级完全共享:
- op_common.cc: HcclExecOp执行框架
- alg_data_trans_wrapper.cc: SendWrite/RecvWrite/LocalCopy
- hccl->hcomm C ABI接口
- hcomm Transport/Dispatcher层

差异仅在:
- Selector: P2P硬编码 vs AllReduce复杂分派
- Executor: P2P单层 vs AllReduce多层编排
- Template: P2P无 vs AllReduce有14+种
- 原语: P2P纯数据搬运 vs AllReduce含Reduce
