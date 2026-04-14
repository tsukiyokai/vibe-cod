# Phase 2.1: Platform Layer — Communication Primitives & Resources

## 2.1.1 comm_primitive/ — 通信原语接口和实现

### Module Overview

comm_primitive 是 hcomm platform 层的数据面编程接口，提供本地和远端通信原语。
它是算法层与硬件之间的桥梁：上层（算法/框架）通过这些原语描述数据搬运意图，
底层通过 DispatcherPub/Transport 将意图翻译为硬件指令。

模块在架构中的位置：
```
hccl ops → hcomm framework → hcomm algorithm → [comm_primitive] → DispatcherPub/Transport → 硬件
```

三代实现并存：
1. Platform 新接口 (src/platform/comm_primitive/): C ABI 平坦函数，委托 DispatcherPub/Transport
2. Legacy 类体系 (src/legacy/service/collective/primitive/): OOP Primitive 继承链 + PrimTranslator 流水线
3. Framework/next 适配层 (src/framework/next/comm_primitives/): AICPU/AIV 的 C 适配桩，当前为空

### Key Files

Platform 新接口（3 个实现 + 4 个头文件）:
- `src/pub_inc/new/hccl_primitive.h` — 基础类型: HcclReduceInfo, HcclBufPair, HcclBatchTransferInfo
- `src/pub_inc/new/hccl_primitive_local.h` — 本地原语声明 (11 个函数)
- `src/pub_inc/new/hccl_primitive_remote.h` — 远端原语声明 (12 个函数)
- `src/pub_inc/new/hccl_dispatcher_ctx.h` — Dispatcher 上下文管理 (7 个函数)
- `src/platform/comm_primitive/hccl_primitive_local.cc:1-207` — 本地原语实现
- `src/platform/comm_primitive/hccl_primitive_remote.cc:1-198` — 远端原语实现
- `src/platform/comm_primitive/hccl_dispatcher_ctx.cc:1-232` — Dispatcher 上下文管理实现

外部 API (hccl 消费):
- `include/hccl/hcomm_primitives.h` — 数据面编程接口 (~30 个函数，供 hccl 调用)

Legacy 类体系 (10 个文件):
- `src/legacy/service/collective/primitive/primitive.h` — Primitive 基类 + 9 个子类
- `src/legacy/service/collective/primitive/primitive.cc` — 子类实现
- `src/legacy/service/collective/primitive/prim_queue.h/.cc` — PrimQueue (HierarchicalQueue 特化)
- `src/legacy/service/collective/primitive/prim_translator.h/.cc` — PrimTranslator (原语→指令翻译器)
- `src/legacy/service/collective/primitive/prim_rules.h/.cc` — 翻译规则 (622 行，核心逻辑)
- `src/legacy/service/collective/primitive/dma_mode.h` — DmaMode 枚举 (DEFAULT/PUT/GET)

### Idioms & Conventions (惯用法)

#### 1. C ABI 边界的 extern "C" 包装

所有 platform 新接口头文件使用 extern "C" 包装，确保 C 链接兼容性:

```cpp
// src/pub_inc/new/hccl_primitive_local.h:20-22
#ifdef __cplusplus
extern "C" {
#endif // __cplusplus
```

注意: 实现文件 (.cc) 本身不需要 extern "C"，因为头文件已声明。
remote 实现使用 `using namespace hccl;`，local 实现不使用(保持完全限定名)。

出处: 4 个头文件全部使用此模式 (hccl_primitive.h, hccl_primitive_local.h, hccl_primitive_remote.h, hccl_dispatcher_ctx.h)

#### 2. CHK_PTR_NULL/CHK_RET 的100%覆盖

每个函数入口对所有指针参数做 CHK_PTR_NULL，每个 HcclResult 返回值用 CHK_RET 检查:

```cpp
// src/platform/comm_primitive/hccl_primitive_local.cc:32-36
HcclResult HcclLocalCopy(StreamHandle streamHandle, HcclBuf *dst, HcclBuf *src)
{
    CHK_PTR_NULL(src);        // 每个指针参数
    CHK_PTR_NULL(dst);
    CHK_PTR_NULL(streamHandle);
    // ...
    CHK_RET(GetPubDispatcher(&dispatcherPtr));  // 每个函数调用
```

量化: 3 个 .cc 文件中共 99 次 CHK_RET/CHK_PTR_NULL/CHK_PRT_RET 调用。
其中 CHK_PRT_RET 用于需要自定义错误消息的场景:
```cpp
// hccl_primitive_local.cc:165
CHK_PRT_RET(streamNum < 1, HCCL_ERROR("[HcclTaskLaunch]threadNum is less than 1"), HCCL_E_PARA);
```

#### 3. void* 句柄 + reinterpret_cast 的委托模式

模块大量使用 void* 作为跨层不透明句柄，在实现内部用 reinterpret_cast 还原类型:

```cpp
// hccl_primitive_local.cc:41
hccl::Stream *stream = reinterpret_cast<hccl::Stream*>(streamHandle);

// hccl_primitive_remote.cc:30
return reinterpret_cast<Transport*>(memTransport)->WriteAsync(remoteBuf, localBuf, *stream);

// hccl_dispatcher_ctx.cc:25
hccl::DispatcherCtx* ctx_temp = reinterpret_cast<hccl::DispatcherCtx *>(ctx);
```

量化: 3 个文件中共 35 次 reinterpret_cast，平均每个函数 1-2 次。

类型映射关系:
- `StreamHandle (void*)` ↔ `hccl::Stream*`
- `HcclMemTransport (void*)` ↔ `hccl::Transport*`
- `DispatcherCtxPtr (void*)` ↔ `hccl::DispatcherCtx*`

#### 4. 日志级别使用规范

- HCCL_INFO: 本地操作入口、上下文管理操作 (8+30 次)
- HCCL_DEBUG: 远端操作入口 (12 次)
- HCCL_WARNING: 非致命异常路径 (dispatcher_ctx.cc 中 5 次)
- HCCL_ERROR: 致命错误 (dispatcher_ctx.cc 中 5 次)

模式: 远端操作因调用频率高用 DEBUG 避免日志风暴，本地操作/管理操作用 INFO。

日志格式惯用法 — 函数名 + 关键参数:
```cpp
// hccl_primitive_remote.cc:24-25
HCCL_DEBUG("[HcclRemoteWrite]streamHandle[%p], memTransport[%p], locBuf addr[%p], rmtBuf addr[%p], len[%llu].",
    streamHandle, memTransport, locBuf->addr, rmtBuf->addr, rmtBuf->len);
```

#### 5. new (std::nothrow) + CHK_PTR_NULL 的堆分配模式

```cpp
// hccl_dispatcher_ctx.cc:80-81
hccl::DispatcherCtx *Ctx_tmp = new (std::nothrow) hccl::DispatcherCtx(devPhyId);
CHK_PTR_NULL(Ctx_tmp);
```

不使用 std::make_unique/make_shared，因为需要 nothrow 语义 + 手动管理生命周期 (Init/Destroy 模式)。
出处: hccl_dispatcher_ctx.cc:80, 220 (两处 new)

#### 6. LIKELY/UNLIKELY 分支预测提示

用于热路径的性能优化:

```cpp
// hccl_dispatcher_ctx.cc:176,181
if (UNLIKELY(commId == nullptr)) { ... }     // 异常路径
if (LIKELY((gDispatcherCtx != nullptr))) { ... }  // 快速路径
if (LIKELY(FindDispatcherByCommId(...))) { ... }   // 正常路径
```

出处: hccl_dispatcher_ctx.cc 中 3 处，集中在 GetDispatcherCtx/DestroyDispatcherCtx 热函数。

#### 7. Legacy 层: MAKE_ENUM 宏 + THROW<T> 异常

Legacy 代码使用两个项目特有的基础设施:

```cpp
// primitive.h:31
MAKE_ENUM(PrimType, POST_TO, WAIT_FROM, WAIT_GROUP, LOCAL_COPY, LOCAL_REDUCE, SEND, RECV, GROUP, SEND_REDUCE, RECV_REDUCE)

// primitive.cc:23
THROW<NullPtrException>("queue");
THROW<InvalidParamsException>("parent Qid is equal to queue Qid");
```

MAKE_ENUM 生成类型安全的枚举类，自带 Describe() 方法。
THROW<T> 是项目自定义的异常抛出宏。
量化: primitive 目录中 MAKE_ENUM 4 次，THROW 30 次。

注意: Legacy 层用异常处理错误，Platform 新接口用 HcclResult 返回码。这是两代代码的核心风格差异。

### Architecture & Patterns (架构模式)

#### 1. 两条委托路径: Local vs Remote

本地原语和远端原语走完全不同的委托路径:

Local 路径 (同芯片内):
```
HcclLocal*() → GetPubDispatcher() → AcquireDispatcherCtx()
             → DispatcherCtx::GetDispatcher()
             → DispatcherPub::MemcpyAsync / InlineReduceAsync / SignalRecord / SignalWait
```
核心中介: DispatcherPub (通过 thread_local 上下文获取)

Remote 路径 (跨芯片):
```
HcclRemote*() → reinterpret_cast<Transport*>(memTransport)
              → Transport::WriteAsync / ReadAsync / Post / Wait
```
核心中介: Transport (由调用方传入 memTransport 句柄)

设计意图: Local 操作依赖当前线程的执行上下文 (哪个 dispatcher 在执行)，
Remote 操作依赖特定的传输通道 (哪条链路连接对端)。

#### 2. thread_local + 全局 map 的双层查找

Dispatcher 上下文管理使用两层存储:

```
thread_local gDispatcherCtx  ← 快速路径 (每线程一个)
      ↓ (miss)
g_ctx (unordered_map<string, void*>)  ← 慢速路径 (按 commId 查找)
      ↓ (miss)
自动创建新 DispatcherCtx  ← AcquireDispatcherCtx 兜底
```

- `SetDispatcherCtx`: 直接设置 thread_local
- `GetDispatcherCtx`: 先查 thread_local，miss 则查 g_ctx map
- `AcquireDispatcherCtx`: GetDispatcherCtx + 自动创建兜底
- `g_mtx` (std::mutex) 保护 g_ctx 的并发访问

出处: hccl_dispatcher_ctx.cc:19-21 (全局变量声明)，174-192 (GetDispatcherCtx 实现)

#### 3. Legacy: Primitive → Instruction 编译流水线

Legacy 层实现了一个"原语编译器"，将高层通信原语翻译为底层硬件指令:

```
PrimQueue (原语队列，层次化)
    ↓ PrimTranslator::Translate()
InsQueue (指令队列，层次化)
```

翻译规则注册 (Strategy + Map):
```cpp
// prim_translator.cc:17-27
primTranslateRuleMap({
    {PrimType::POST_TO, GetRule<PrimPostTo>()},
    {PrimType::SEND, GetRule<PrimSend>()},
    ...
})
```

翻译过程根据链路类型和设备能力做策略选择:
```
PrimSend → DmaMode?
    PUT → PrimSendInWriteMode → link type?
        P2P → NormalWriteMode (WaitReady + Write*N + PostFin)
        DEV_NET → WriteWithNotify 支持? → WriteWithNotifyMode / NormalWriteMode
    GET → PrimSendInReadMode (PostReady + WaitFin)
    DEFAULT → P2P? GET : PUT
```

指令排序通过优先级 map 实现:
```cpp
// prim_rules.cc:15-31
constexpr u32 INSTRUCTION_PRI_LOCAL_COPY = 100;   // 最高
constexpr u32 INSTRUCTION_PRI_LOCAL_POST_TO = 90;
constexpr u32 INSTRUCTION_PRI_WAIT_FIN_ACK = 40;  // 最低
```

出处: prim_rules.cc (622 行，翻译逻辑核心)，prim_translator.cc (52 行，编排)

#### 4. Primitive 类继承体系

```
Primitive (abstract)
├── PrimPostTo      — 通知生产者 (向 queue 发送通知)
├── PrimWaitFrom    — 通知消费者 (等待来自 queue 的通知)
├── PrimWaitGroup   — 多源等待 (等待多个 queue)
├── PrimLocalCopy   — 本地内存拷贝
├── PrimLocalReduce — 本地归约
├── PrimSend        — 远端发送 (单边写)
├── PrimRecv        — 远端接收 (单边读)
├── PrimSendReduce  — 远端发送+归约
├── PrimRecvReduce  — 远端接收+归约
└── PrimGroup       — 原语组合 (多个 Send/Recv 并行)
```

关键设计: PrimQueue 继承 HierarchicalQueue，支持 master/slave 层次结构。
PrimGroup 强制只能包含 Send/Recv 类原语 (通过 Append 中的类型检查)。

出处: primitive.h:27-465

#### 5. 外部 API: 双层接口设计

hcomm 对外暴露两层通信原语接口:

内部接口 (pub_inc/new/): 使用 StreamHandle + HcclBuf 抽象
- HcclLocalCopy, HcclRemoteWrite 等
- 返回 HcclResult

外部接口 (include/hccl/hcomm_primitives.h): 使用 ThreadHandle + ChannelHandle + 裸指针
- HcommLocalCopyOnThread, HcommWriteOnThread 等
- 返回 int32_t
- 另有 Nbi (非阻塞) 变体: HcommWriteNbi, HcommReadNbi

两层接口的参数抽象层次不同:
- 内部: StreamHandle, HcclBuf (结构体封装 addr+len), HcclMemTransport
- 外部: ThreadHandle, ChannelHandle (uint64_t), void* + uint64_t len

外部接口有自己的类型系统: HcommReduceOp, HcommDataType (独立枚举，非复用 Hccl 类型)。
部分外部 API 标记 "WARNING: experimental API"。

出处: hcomm_primitives.h:1-448, hccl_primitive_local.h:1-104

### Domain Knowledge (业务知识)

#### 1. 本地 vs 远端通信原语

本地原语 (同芯片内, LINK_ONCHIP):
- LocalCopy: 芯片内存拷贝 (SDMA)
- LocalCopyReduce: 拷贝 + 归约 (InlineReduce)
- LocalNotifyRecord/Wait: 芯片内同步 (生产者-消费者)
- BareNotifyRecord/Wait: 基于 notifyId 的轻量通知 (无 aclrtNotify 对象)

远端原语 (跨芯片):
- RemoteWrite: 单边 RDMA 写 (推送数据到对端)
- RemoteRead: 单边 RDMA 读 (从对端拉取数据)
- RemoteWriteReduce/ReadReduce: 带原子归约的读写
- RemoteNotifyRecord/Wait: 跨芯片同步
- RemoteFence: 内存访问顺序屏障 (UB 执行序/完成序)
- RemoteBatchWrite/Read: 批量操作 (多个 bufPair 一次提交)

#### 2. DMA 传输模式 (PUT/GET/DEFAULT)

Legacy prim_rules.cc 中的策略揭示了核心业务逻辑:

- PUT (写模式): 发送方主动将数据推送到接收方内存
  - 流程: WaitReady → Write*N → PostFin (发送侧)
  - 流程: PostReady → WaitFin (接收侧)

- GET (读模式): 接收方主动从发送方内存拉取数据
  - 流程: PostReady → WaitFin (发送侧)
  - 流程: WaitReady → Read*N → PostFin (接收侧)

- DEFAULT (自动选择):
  - P2P (HCCS) 链路: 默认用 GET (读模式)
  - DEV_NET (RoCE) 链路: 默认用 PUT (写模式)

原因推测: P2P 链路 (HCCS) 支持高效的远端读；RoCE 网络中远端读需要额外往返，写更高效。

出处: prim_rules.cc:438-478 (Translate PrimSend/PrimRecv 的策略选择)

#### 3. InlineReduce vs 软件 Reduce

硬件是否支持 InlineReduce 取决于:
- 数据类型 (DevCapability::GetInlineReduceDataTypeMap)
- 归约操作 (DevCapability::GetInlineReduceOpMap)
- 链路类型 (DEV_NET 需要额外的 IsSupportDevNetInlineReduce 能力)

不支持 InlineReduce 时的降级策略:
- SendReduce: 先 Write 到 remoteSrc，再在远端做 LocalReduce
- RecvReduce: 先 Read 到 localSrc，再在本地做 LocalReduce

出处: prim_rules.cc:66-79 (IsSupportInlineReduce), 480-570 (降级逻辑)

#### 4. Notify 同步协议

通信原语使用三轮握手协议:
1. PostReady / WaitReady — "准备好了/等待对方准备好"
2. Write/Read — 数据传输
3. PostFin / WaitFin — "传输完成"
4. (可选) PostFinAck / WaitFinAck — DEV_NET 链路的额外确认 (IsSupportStarsPollNetCq 为 false 时需要)

WriteWithNotify 是优化: 将 Write + PostFin 合并为一个硬件操作，减少一次通知开销。
仅部分平台支持 (DevCapability::IsSupportWriteWithNotify)。

出处: prim_rules.cc:130-178 (Write 模式的三轮握手)

#### 5. 批量模式

外部 API 提供 HcommBatchModeStart/End 括号，中间的操作被批量下发:
```
HcommBatchModeStart(batchTag)
  HcommWriteOnThread(...)  // 不立即执行
  HcommWriteOnThread(...)
HcommBatchModeEnd(batchTag)  // 一次性提交
```

内部接口的 HcclRemoteBatchWrite/Read 是循环调用单次 Write/Read:
```cpp
// hccl_primitive_remote.cc:158-163
for (uint32_t i = 0; i < bufPairNum; i++) {
    CHK_RET(transport->WriteAsync(remoteBuf, localBuf, *stream));
}
```

### Hardware Abstractions (硬件知识)

#### 1. 链路类型决定行为

代码中可见的链路类型层次:

```
LinkType (传输层)
├── LINK_ONCHIP  — 芯片内 (本地原语使用)
└── LINK_ROCE    — RoCE 网络 (ReadReduce 不支持)

PortDeploymentType (部署层)
├── P2P          — 点对点直连 (HCCS 总线)
└── DEV_NET      — 设备网络 (需要进一步看 LinkProtocol)

LinkProtocol (协议层)
├── ROCE         — RoCE v2
├── UB_CTP       — Unified Bus CTP
└── UB_TP        — Unified Bus TP
```

关键约束:
- HcclRemoteReadReduce 仅支持 P2P，不支持 RoCE:
  ```cpp
  // hccl_primitive_remote.cc:78-80
  if (reinterpret_cast<Transport*>(memTransport)->GetLinkType() == LinkType::LINK_ROCE) {
      HCCL_ERROR("[HcclRemoteReadReduce]ROCE is not supported.");
      return HCCL_E_NOT_SUPPORT;
  }
  ```

#### 2. 执行引擎多样性

CMakeLists.txt 显示同一套源码编译到三个目标:
```cmake
target_sources(hccl_plf PRIVATE ${src_list})        # 主库
target_sources(ccl_kernel_plf PRIVATE ${src_list})   # kernel 变体
target_sources(ccl_kernel_plf_a PRIVATE ${src_list}) # kernel_a 变体
```

外部 API 的 ThreadHandle 对应不同执行引擎:
- AICPU 引擎: 通过 DispatcherAiCpu 执行
- AIV 引擎: 通过 aiv_primitives_c_adpt (框架尚未实现)
- Host CPU: 通过 host 侧数据面接口

DispatcherCtx 支持的类型:
```
CtxDispatcherType:
├── DISPATCHER_NORMAL
├── DISPATCHER_VIRTURAL
├── DISPATCHER_AICPU
└── DISPATCHER_FFTS
```

#### 3. UB (Unified Buffer) Fence 语义

```cpp
// hccl_primitive_remote.h:120
extern HcclResult HcclRemoteFence(StreamHandle streamHandle, HcclMemTransport memTransport, uint32_t orderFlag);
// orderFlag 支持 UB 的执行序、完成序等
```

Fence 用于保证 UB (Unified Buffer) 上的内存访问顺序，
是 Ascend 芯片特有的硬件抽象 (类似 GPU 的 memory barrier)。

#### 4. Graph 缓存机制

Task 管理涉及 Graph 缓存 (FFTS 模式):
```cpp
// hccl_primitive_local.cc:92-112
HcclResult HcclLocalInitTask(aclrtStream stream, const bool enableCache, const std::string &key, bool useGraphConstructorV2)
{
    // ...
    CHK_RET(dispatcherPtr->ResetGraphCtx(enableCache, key, useGraphConstructorV2));
    if (ctx_temp->GetInitTaskCallback() != nullptr) {
        CHK_RET(ctx_temp->GetInitTaskCallback()(dispatcherPtr, stream_temp));
    }
}
```

enableCache + key 参数实现 Graph 复用: 相同 key 的任务序列可以缓存编译结果，
避免重复编译 (类似 CUDA Graph capture/replay)。

### Anti-Patterns (反模式)

#### 1. CreateDispatcherCtx 绑定失败时静默复用

```cpp
// hccl_dispatcher_ctx.cc:90-103
ret = BindDispatcherCtxWithComm(Ctx_tmp, commId);
if (ret != HCCL_SUCCESS) {
    Ctx_tmp->Destroy();
    delete Ctx_tmp;
    if (!FindDispatcherByCommId(ctx, commId)) {
        return HCCL_E_NOT_FOUND;
    }
    gDispatcherCtx = *ctx;
    HCCL_WARNING("[CreateCtx] CTX bind fail, reuse existing ctx[%p] commId[%s]", *ctx, commId);
    return HCCL_SUCCESS;  // 返回 SUCCESS 但用了别人的 ctx
}
```

问题: 绑定失败后返回 HCCL_SUCCESS 并复用已有 ctx，调用方无法区分"新建成功"和"复用已有"。
这种静默降级可能掩盖并发创建的竞态问题。

#### 2. AcquireDispatcherCtx 创建但不绑定到全局 map

```cpp
// hccl_dispatcher_ctx.cc:213-231
HcclResult AcquireDispatcherCtx(DispatcherCtxPtr *ctx, const char* commId)
{
    // ... GetDispatcherCtx miss 后:
    hccl::DispatcherCtx *Ctx_tmp = new (std::nothrow) hccl::DispatcherCtx(devPhyId);
    // ...
    *ctx = Ctx_tmp;
    gDispatcherCtx = Ctx_tmp;  // 只设了 thread_local，没加入 g_ctx map!
}
```

问题: 新创建的 ctx 只存在于 thread_local，不注册到全局 map。
如果线程结束或 thread_local 被覆盖，这个 ctx 就泄漏了(无法通过 commId 找回)。

#### 3. DestroyDispatcherCtx 使用函数级 static mutex

```cpp
// hccl_dispatcher_ctx.cc:129-130
HcclResult DestroyDispatcherCtx(DispatcherCtxPtr ctx, const char* commId)
{
    static std::mutex deleteMutex_;
    const std::lock_guard<std::mutex> lock(deleteMutex_);
```

问题: 函数级 static mutex 是全局单例，所有通信域的销毁操作互斥。
这不同于 g_mtx (保护 g_ctx map) 的职责分离方式。
注释 `// 考虑已有的universal_map，或读写锁` (第20行) 也暗示开发者认识到锁粒度需要优化。

#### 4. HcclRemoteReadReduce 疑似 bug: 除以 dataType 枚举值

```cpp
// hccl_primitive_remote.cc:87-88
return dispatcherPtr->InlineReduceAsync(rmtBuf->addr, rmtBuf->len / reduceInfo.dataType,
    reduceInfo.dataType, reduceInfo.reduceOp, ...);
```

对比同模块 HcclLocalCopyReduce 的正确写法:
```cpp
// hccl_primitive_local.cc:64
return dispatcherPtr->InlineReduceAsync(src->addr, src->len / SIZE_TABLE[reduceInfo.dataType], ...);
```

Local 版本用 `SIZE_TABLE[reduceInfo.dataType]` 查表获取字节数。
Remote 版本直接用 `reduceInfo.dataType` (枚举值) 做除法，这很可能是 bug。
HcclDataType 枚举值 (如 INT8=0, INT16=1, FP16=3) 不等于实际字节数。

Phase 1 发现: hccl_primitive_local.cc 曾被 revert (commit f0666214)，
原始 commit 0e5be69c 添加了 host 侧 SDMA 支持但被回退。

#### 5. 未实现的 API 返回 HCCL_E_NOT_SUPPORT

4 个远端原语函数已声明但返回 NOT_SUPPORT:
- HcclRemoteWriteWithNotify (hccl_primitive_remote.cc:123)
- HcclRemoteWriteReduceWithNotify (hccl_primitive_remote.cc:136)
- HcclRemoteBatchTransfer (hccl_primitive_remote.cc:197)

这些是预留接口，等待硬件能力就绪后实现。

### Notable Patterns (值得注意的模式)

#### 1. 新旧两代错误处理哲学的对比

Platform 新接口: 返回码模式
- CHK_RET/CHK_PTR_NULL 宏自动返回错误码
- 不抛异常
- 适合 C ABI 边界

Legacy 类体系: 异常模式
- THROW<ExceptionType>("描述") 抛出自定义异常
- 构造函数中做参数校验 (如 PrimLocalCopy 检查 src/dst 大小一致、地址不重叠)
- 需要调用方 catch

这是代码从 C++ OOP 风格向 C ABI 平坦接口演进的典型痕迹。

#### 2. Legacy PrimGroup::CheckValid 中的 const_cast

```cpp
// primitive.cc:366
PrimSend *primSend = dynamic_cast<PrimSend *>(const_cast<Primitive *>(&(*iter)));
```

需要 const_cast 来从 const 迭代器获取可 dynamic_cast 的指针。
更好的做法是让迭代器返回非 const 引用，或提供类型查询虚方法。

#### 3. 外部 API 重复定义数据类型

hcomm_primitives.h 定义了独立的 HcommReduceOp / HcommDataType 枚举，
与 hccl_types.h 中的 HcclReduceOp / HcclDataType 存在映射关系但不共享定义。
包含条件编译: `#ifndef OPEN_BUILD_PROJECT` 控制部分 FP8 数据类型的可见性。

#### 4. NPU-DPU 通信接口

hcomm_primitives.h 包含 NPU ↔ DPU 的消息传递接口:
```cpp
extern int32_t HcommSendRequest(MsgHandle handle, const char* msgTag, const void *src, size_t sizeByte, uint32_t *msgId);
extern int32_t HcommWaitResponse(MsgHandle handle, void *dst, size_t sizeByte, uint32_t *msgId);
extern int32_t HcommFlush();
```

这些是 experimental API，用于 NPU 通过 HBM 共享内存向 DPU 发送同步消息。

#### 5. 指令优先级排序

Legacy PrimGroup 翻译时会对所有指令按优先级排序:
```
LOCAL_COPY(100) > LOCAL_POST_TO(90) > LOCAL_WAIT_FROM(85) > POST_READY(80)
> WAIT_READY(70) > WRITE/READ(60) > POST_FIN/WAIT_FIN(50) > WAIT_FIN_ACK(40)
```

设计意图: 同一步中的多个操作按依赖顺序执行 — 先本地拷贝，再发通知，再等通知，再传数据，最后确认完成。
同优先级内按原始顺序 (稳定排序)。

出处: prim_rules.cc:15-31 (优先级定义), 572-621 (排序逻辑)

---

## 2.1.2 resource/ — 通信资源的分配、管理、回收机制

### Module Overview

resource/ 是 platform 层最大的子模块(~110 C++文件)，负责通信所需的全部底层资源管理。
8个子目录覆盖: 内存(mem)、通知/同步(notify)、远程内存访问缓冲(rma_buffer)、
传输通道(transport)、分发器上下文(dispatcher_ctx)、网络设备(netdev)、
套接字(socket)、流(stream)。

在整体架构中的位置: platform 层的资源基础设施，被 framework 层和 algorithm 层
通过 Transport/Notify/Stream 等抽象间接使用。

### Key Files

| 子模块 | 文件数 | 核心文件 |
|-------|-------|---------|
| mem/ | 10 | mem_device.h/cc, mem_host.h/cc, hdc.cc, heterog_mem_blocks_manager.h/cc, mem_mapping_manager*.cc |
| notify/ | 16 | notify_base.h, rts_notify.h/cc, bare_notify.h/cc, esched_notify.h/cc, notify_pool_impl.h/cc, local_notify_impl.h/cc, remote_notify_impl.h/cc |
| rma_buffer/ | 12 | local_ipc_rma_buffer_impl.h/cc, local_rdma_rma_buffer_impl.h/cc, remote_ipc_rma_buffer_impl.h/cc, remote_rdma_rma_buffer_impl.h/cc |
| transport/ | ~62 | transport.h/cc, transport_base_pub.h, transport_base.cc, transport_p2p*.h/cc, transport_ibverbs*.h/cc, transport_roce*.h/cc, transport_tcp*.h/cc |
| dispatcher_ctx/ | 2 | dispatcher_ctx.h/cc |
| netdev/ | 4 | hccl_net_dev_v1.h/cc, hccl_net_dev_v2.h, hccl_net_dev.cc |
| socket/ | 3 | hccl_network.h/cc, hccl_socket.cc |
| stream/ | 2 | stream.h/cc |

---

### 子模块 A: mem/ — 内存资源管理

#### 设计概览

三种独立的内存管理机制:
1. DeviceMem/HostMem: 值语义RAII类，owner_标志区分所有权
2. MemMappingManager: 主机-设备地址映射缓存，引用计数管理
3. HeterogMemBlocksManager: 固定大小块的队列式预分配
4. HDCommunicate: Host-Device单向通道，基于共享内存+双计数器

#### 所有权模型: owner_ + move semantics

DeviceMem和HostMem采用相同的所有权模式(非shared_ptr)，通过owner_布尔标志+移动语义实现:

```cpp
// mem_device.cc:15-40
DeviceMem::DeviceMem(void *ptr, u64 size, bool owner)
    : ptr_(ptr), size_(size), owner_(owner) {}

// 复制: 副本不拥有所有权
DeviceMem::DeviceMem(const DeviceMem &that)
    : ptr_(that.ptr()), size_(that.size_), owner_(false) {}

// 移动: 转移所有权，清空源
DeviceMem::DeviceMem(DeviceMem &&that) noexcept
    : ptr_(that.ptr()), size_(that.size_), owner_(that.owner_) {
    that.ptr_ = nullptr; that.size_ = 0; that.owner_ = false;
}

// 析构: 仅owner释放
DeviceMem::~DeviceMem() {
    if (owner_ && ptr_) {
        HcclResult ret = hrtFree(ptr_);
        if (ret != HCCL_SUCCESS) { HCCL_WARNING(...); }
    }
}
```

API语义区分:
- `alloc(size)`: 创建拥有所有权的对象(owner_=true)
- `create(ptr, size)`: 包装已有指针，不拥有(owner_=false)
- `range(offset, size)`: 切割子块，不拥有(owner_=false)

出处: mem_device.cc:15-130, mem_host.cc:15-125

设计决策理由:
- 避免shared_ptr引用计数开销(DeviceMem对象很轻，仅3个字段)
- 与C API(hrtMalloc/hrtFree)兼容
- alloc/create的显式语义比构造函数更安全

#### HostMem 双分配源

HostMem额外维护isRtsMem_标志，析构时自动选择释放器:

```cpp
// mem_host.cc:47-62
if (isRtsMem_) {
    hrtFreeHost(ptr_);      // RTS分配的内存
} else {
    delete[] static_cast<u8 *>(ptr_);  // 标准C++ new
}
```

出处: mem_host.cc:15-86

#### MemMappingManager: 映射缓存 + 引用计数

单例模式(per device)，缓存主机→设备虚拟地址映射:

```cpp
// mem_mapping_manager.cc:49-66
bool MemMappingManager::IsRequireMapping(void *addr, u64 size, void *&devVA) {
    auto iter = SearchMappingMap(userAddr, userSize);
    if (iter != mappedHostToDevMap_.end()) {
        // 缓存命中: 计算偏移后的设备VA
        u64 tmpDva = reinterpret_cast<u64>(iter->second.devVA) + userAddr - iter->first.addr;
        devVA = reinterpret_cast<void*>(static_cast<uintptr_t>(tmpDva));
        iter->second.ref.Ref();  // 引用计数++
        return false;
    }
    return true;  // 需要新建映射
}
```

设备类型适配:
- 910B/910_93: HOST_MEM_MAP_DEV_PCIE_TH (PCIe TH参数)
- 其他设备: HOST_MEM_MAP_DEV (默认)

出处: mem_mapping_manager.cc:18-77, mem_mapping_manager_device.cc:18-58, mem_mapping_manager_host.cc:18-66

#### HDCommunicate: 共享内存双计数器协议

Host-Device单向通道，内存布局:

```
+---------------------+
|      content        |  ← buffLen_
+---------------------+
|    head_cnt[u32]    |  ← 发送端写入前+1
+---------------------+
|    tail_cnt[u32]    |  ← 发送端写入后+1
+---------------------+
```

接收端检查 head==tail 确保数据完整性。
写入前使用 std::atomic_thread_fence(std::memory_order_seq_cst) 保证可见性。

设备内存映射支持检测(hdc.cc:57-90):
- 查询CTRL_SUPPORT_PCIE_BAR_MEM_MASK
- 不支持时降级为hrtDrvMemCpy方式
- 共享内存4K页对齐(HCCL_SHM_ALIGN)

出处: hdc.cc:32-317

#### HeterogMemBlocksManager: 队列式块分配

预分配固定大小内存块，通过队列管理:
- Init: 分配总内存(块数*块大小+1页)，4K对齐，所有块入队
- Alloc: 从队列Pop，mutex保护
- Free: 归还到队列Push
- 幂等初始化: isinited_标志防止重复

出处: heterog_mem_blocks_manager.cc:30-71

#### mem/ 反模式

1. DeviceMem vs HostMem的range()实现不一致:
   - DeviceMem: 显式if-return提前退出(mem_device.cc:117-130)
   - HostMem: 单个if-else条件(mem_host.cc:116-125)
   - 建议统一为DeviceMem的显式提前返回风格

2. Referenced引用计数初始化为0而非1(referenced.h:19)，调用者必须显式Ref()，
   容易遗漏。MemMappingManager中使用正确(有mutex保护)。

3. mem_mapping_manager_device.cc和mem_mapping_manager_host.cc的MapMem/ReleaseDevVA
   几乎完全重复，仅910B设备处理不同。可统一。

---

### 子模块 B: notify/ — 通知/同步机制

#### 设计概览

三层结构:
1. NotifyBase: 纯虚基类，定义Alloc/Destroy/Wait/Post/SetIpc/Grant等接口
2. 三种具体实现: RtsNotify(硬件notify)、BareNotify(空操作)、EschedNotify(事件调度)
3. Wrapper层(Pimpl): LocalNotifyImpl→LocalIpcNotify, RemoteNotifyImpl→RemoteNotify, NotifyPoolImpl→NotifyPool

#### Notify类型选择规则

根据local/remote设备ID组合自动选择(local_notify_impl.cc:42-71):

| local | remote | Notify类型 | 原因 |
|-------|--------|-----------|------|
| Device | Device | RtsNotify(RUNTIME_NOTIFY) | 需要硬件信号 |
| Device | Host | BareNotify | Host无真正notify资源 |
| Host | Device | EschedNotify | Host侧事件驱动 |
| any | any (DEVICE_NOTIFY) | RtsNotify(RUNTIME_NOTIFY_MC2) | MC2专用 |

出处: local_notify_impl.cc:42-71

#### RtsNotify: 硬件Notify + IPC

核心生命周期:
1. Alloc: hrtNotifyCreate/hrtNotifyCreateWithFlag创建硬件资源
2. SetIpc: hrtIpcSetNotifyName生成IPC名称，withIpc=true
3. Grant: hrtSetIpcNotifyPid/hrtSetIpcMemorySuperPodPid设置进程白名单
4. Open: 远端通过IPC名打开
5. Destroy: 释放硬件资源

recvId编码: 高32位=SDID, 低32位=PID (notify_pool_impl.cc:475-497)

出处: rts_notify.cc:112-183

#### NotifyPool: 三层资源管理

三个独立管理器(notify_pool_impl.h:33-45):
- NOTIFY_NORMAL(0): 常规算子
- NOTIFY_A2A(1): AllToAll算子
- NOTIFY_ALIGN(2): 需要8字节对齐的atomic write

分配策略:
1. 查找tag在registeredOpMap中的位置
2. 遍历对应remote的notify列表，找满足offset对齐要求的
3. 命中: reuse并Grant()更新pid白名单
4. 未命中: CreateNotify()申请新的

对齐检查(notify_pool_impl.cc:274-288):
- NOTIFY_OFFSET_ALIGN_EIGHT: offset % 8 == 0
- NOTIFY_OFFSET_ALIGN_FOUR: offset % 4 == 0

出处: notify_pool_impl.cc:69-497

#### notify/ 反模式

1. LocalIpcNotify::SetEventIdAndTid()创建临时LocalNotifyImpl对象调用方法(local_ipc_notify.cc:116-120):
   ```cpp
   void LocalIpcNotify::SetEventIdAndTid(const u32 eventId, const u32 tid) {
       LocalNotifyImpl impl;  // 临时对象！
       return impl.SetEventIdAndTid(eventId, tid);  // 调用后销毁
   }
   ```
   应直接调用静态方法。

2. EschedNotify大量空实现(esched_notify.cc:84-117): Alloc/Destroy/ThreadIdCreate等
   全部返回HCCL_SUCCESS，可能是功能未实现或编译时禁用。

3. NotifyPool的CreateNotify()中对齐重试循环(notify_pool_impl.cc:69-122)逻辑复杂:
   不满足对齐的notify存入tmpNotifys，循环结束后clear()销毁。
   shared_ptr保证安全但意图不够清晰。

---

### 子模块 C: rma_buffer/ — 远程内存访问缓冲

#### 设计概览

2x2矩阵设计:

|  | IPC方式 | RDMA方式 |
|--|--------|---------|
| Local | LocalIpcRmaBuffer | LocalRdmaRmaBuffer |
| Remote | RemoteIpcRmaBuffer | RemoteRdmaRmaBuffer |

全部采用Pimpl模式: 公开类在pub_inc/inner/，Impl类在src/platform/resource/rma_buffer/。

#### 生命周期对比

Local端:
1. 构造(netDevCtx, addr, size, memType)
2. Init → 注册内存(IPC: SetIpcMem / RDMA: RegGlobalMr)
3. Serialize → 生成字节序列(缓存，首次生成后复用)
4. Grant → 设置远端进程白名单
5. Destroy → 注销(析构函数中调用，RAII)

Remote端:
1. 构造(无地址)
2. Deserialize → 从字节序列重建
3. Open → 打开远端内存(IPC: OpenIpcMem)
4. Close → 关闭

#### RDMA MR引用计数保护

全局计数表防止同一内存重复注册/注销MR(local_rdma_rma_buffer_impl.cc:34-84):

```cpp
std::unordered_map<s32, std::unordered_map<std::string, u32>> g_devAddrIdentifierMap;

// 注销时仅在计数归零时真正释放
if (!IsDevAddrExistInDevAddrIdentifierMap(deviceLogicId, devAddrID)) {
    ret = hrtRaDeRegGlobalMr(rdmaHandle, mrHandle);  // 计数为0才注销
}
```

MR注册参数(local_rdma_rma_buffer_impl.cc:109-112):
```cpp
info.access = RA_ACCESS_REMOTE_WRITE | RA_ACCESS_LOCAL_WRITE | RA_ACCESS_REMOTE_READ;
```

出处: local_rdma_rma_buffer_impl.cc:34-190

#### 二进制序列化

使用ostringstream进行二进制(非文本)序列化:
```cpp
std::ostringstream oss;
oss.write(reinterpret_cast<const char_t *>(&type), sizeof(type));
oss.write(reinterpret_cast<const char_t *>(&addr), sizeof(addr));
// ... size, devAddr, memType, ipcName/rkey
```

出处: local_ipc_rma_buffer_impl.cc:57-77, local_rdma_rma_buffer_impl.cc:129-147

#### rma_buffer/ 反模式

1. 全局g_devAddrIdentifierMap(local_rdma_rma_buffer_impl.cc:34-35):
   测试困难，无法隔离单个buffer。注释暗示曾发生过重复释放MR的bug。

2. IPC_NOTIFY_PID_ARRAY_SIZE硬编码为1(local_ipc_rma_buffer_impl.cc:79):
   不可扩展到多进程通知场景。

---

### 子模块 D: transport/ — 传输通道抽象

#### 设计概览

两层包装器模式:
1. Transport(Facade): 持有TransportBase*，根据TransportType动态创建具体实现
2. TransportBase(虚基类): 70+虚函数，定义完整传输接口

Transport工厂(transport.cc:25-71):

| TransportType | 实现类 | 通信场景 |
|--------------|--------|---------|
| TRANS_TYPE_P2P | TransportP2p | 同节点IPC/HCCS/PCIe |
| TRANS_TYPE_HOST_TCP | TransportTcp | 跨节点TCP |
| TRANS_TYPE_IBV_EXP | TransportIbverbs | 跨节点InfiniBand RDMA |
| TRANS_TYPE_DEVICE_DIRECT | TransportDirectNpu | 直连NPU RDMA+AICPU |
| TRANS_TYPE_DEVICE_P2P | TransportDeviceP2p | 设备侧P2P |
| TRANS_TYPE_DEVICE_IBVERBS | TransportDeviceIbverbs | 设备侧IBVerbs |
| TRANS_TYPE_ROCE_HOST_* | TransportRoce | RoCE跨节点 |
| TRANS_TYPE_SHM_EVENT | TransportShmEvent | Host-Device共享内存事件 |
| TRANS_TYPE_VIRTURAL | TransportVirtural | 虚拟(仅记录任务逻辑) |

出处: transport.cc:25-83, transport_base_pub.h:52-349

#### TransportBase公共功能

1. 交换数据(Exchange Data): 连接建立时交换自定义数据，MAX_EXCHANGE_DATA_LEN限制
   出处: transport_base.cc:879-942

2. 五类通知信号:
   - localSendReadyNotify_ / remoteSendReadyNotify_ — 发送就绪
   - localSendDoneNotify_ / remoteSendDoneNotify_ — 发送完成
   - Device侧: localSendReadyDeviceNotify_ / localSendDoneDeviceNotify_
   - User自定义: userLocalNotify_[] / userRemoteNotify_[]
   出处: transport_base_pub.h:266-328, transport_base.cc:409-845

3. TGID/SDID交换: 910_93设备填充SuperPodInfo(pid, sdid, serverPhyIdx)
   出处: transport_base.cc:409-450

4. 原子写支持: useAtomicWrite_(transport_base_pub.h:347)

#### Host Transport实现矩阵

| Transport | 链路 | QP管理 | Reduce | 多QP | 特殊 |
|-----------|------|--------|--------|------|------|
| P2P | IPC/HCCS/SIO/PCIe | 无 | 否 | 否 | SDMA信号记录 |
| TCP | 以太网 | 无 | 否 | 否 | 5MB缓冲区 |
| IBVerbs | InfiniBand | 单/多QP | 是(硬件加速) | 是 | WQE批量、CQE错误收集 |
| RoCE | RoCE | CQ+QP | 否 | 否 | 3条独立链路、任务编排 |
| ShmEvent | 共享内存 | 无 | 否 | 否 | Worker/PS架构、嵌套锁 |
| DirectNpu | 直连RDMA | 单QP | 否 | 否 | 动态加载AICPU内核 |
| Virtual | 无 | 无 | 否 | 否 | 仅记录TaskLogicInfo |

#### Device Transport特有设计

1. P2P(transport_device_p2p.cc): 跨节点场景用SDMA信号记录(STARS可检出链路异常触发重执行)
2. IBVerbs(transport_device_ibverbs.cc):
   - 内存屏障: HCOMM_DSB()宏(x86=compiler fence, aarch64=dsb st)
   - WR id全局自增: static std::atomic<u64> wrIdOffset_(memory_order_relaxed)
   - 零字节消息: 不下发WR，opRsp设为INVALID_UINT
   - 多QP阈值: multiQpThreshold_根据数据大小自适应选择

出处: transport_device_p2p.cc:148-156, transport_device_ibverbs.cc:31-38, 504-526, 600

#### Heterog Transport特有设计

1. P2P异构(transport_heterog_p2p.cc): 四阶段连接状态机:
   INIT → SOCKET_CONNECT → EXCHANGE_PID → EXCHANGE_TRANSPORT_INFO → DONE
   带自适应重试和超时控制。

2. RoCE异构(transport_heterog_roce.cc): 3条独立链路:
   - SOCKET_FOR_TAG_QP(0): 控制消息
   - SOCKET_FOR_DATA_QP(1): 数据
   - SOCKET_FOR_SENDRECV_QP(2): 收发

3. Event RoCE(transport_heterog_event_roce.cc): 全局CQE计数器+try_lock避免活锁

出处: transport_heterog_p2p.cc:143-176, transport_heterog_roce.cc:25-32

#### Onesided Transport特有设计

1. RoCE单边(transport_roce_mem.cc): 超2G数据自动分片(MAX_RDMA_WQE_SIZE=2G)
2. Fence机制: AddOpFence通过RDMA READ实现全序化
3. RmaBufferMgr: 引用计数管理远端内存访问

出处: transport_roce_mem.cc:25-26, 351-391, 481-501

#### TCP线程模型

接收端: 事件驱动单例(TcpRecvTask)
- 静态回调RecvDataCb，通过fdTransportMap_路由到对应Transport
- 二元状态机: 未收到信封→接收HcclEnvelope → 接收数据体
- 引用计数(initCount_)管理生命周期

出处: tcp_recv_task.cc:42-164

发送端: 线程池(TcpSendThreadPool)
- HOST平台2线程, Device平台1线程
- 双队列切换: 主队列收任务，备队列执行
- 基于tag分片的负载均衡
- 绑核优化: pthread_setaffinity_np + cgroup

出处: tcp_send_thread_pool.cc:56-294

#### MemNameRepository: IPC内存去重

单例(per device)，核心操作:
- SetIpcMem: 页对齐→去重检查→hrtIpcSetMemoryName注册→设置PID白名单
- OpenIpcMem: 去重缓存查找→hrtIpcOpenMemory→加offset
- 引用计数: setNameMapRef_ / openedNameMapRef_ 确保最后一个关闭时才调用系统清理

出处: mem_name_repository.cc:30-152

#### transport/ 反模式

1. Transport全局静态map(transport.cc:23-24):
   ```cpp
   static std::unordered_map<TransportBase*, Transport*> transportMap_;
   ```
   多个Transport同时析构可能竞争。析构中delete pimpl_后野指针风险。

2. TCP发送双队列切换(tcp_send_thread_pool.cc:260-264):
   三元运算符嵌套逻辑晦涩:
   ```cpp
   TaskQueueManager_[threadSerial].threadTaskQueuePtr =
       &...taskQueues.first == ...threadTaskQueuePtr ?
       &...taskQueues.second : &...taskQueues.first;
   ```
   应拆分为if-else。

3. TCP发送SaluSleep(500)(tcp_send_thread_pool.cc:294):
   一次发送超过1MB时sleep 500us重试，可能影响延迟。

4. TransportDirectNpu的虚拟Notify填充(transport_direct_npu.cc:85-114):
   GetLocalNotifyValueAddrKey()重复push三个相同的AddrKey，注释说明是临时方案。

5. Heterog Event全局状态(transport_heterog_event_roce.cc:49-52):
   gQpnToTransportMap等全局map使用普通map而非ConcurrentMap，多连接时潜在竞态。

---

### 子模块 E: dispatcher_ctx/ — 分发器上下文

#### 设计概览

DispatcherCtx根据平台信息动态选择Dispatcher实现(dispatcher_ctx.cc:30-58):
- Host侧 + 910B + FFTS开关 → DispatcherGraph(FFTS模式)
- Host侧(其他) → DispatcherPub(NORMAL模式)
- Device侧 → DispatcherAiCpu(AICPU模式)
- 虚拟 → DispatcherVirtural

工厂模式创建(dispatcher_ctx.cc:85-138): switch语句+new(std::nothrow)+失败delete清理。

出处: dispatcher_ctx.cc:30-138

---

### 子模块 F: netdev/ — 网络设备抽象

#### 设计概览

v1/v2版本切换:
- v1: 强符号，所有函数直接实现
- v2: __attribute__((weak))弱符号，允许被覆盖
- 适配层: DEV_TYPE_910_95用v2，其他用v1

出处: hccl_net_dev.cc:18-83, hccl_net_dev_v1.cc:24-174, hccl_net_dev_v2.h:22-26

NetDevContext(hccl_network.h:23-126): 管理单个网络设备上下文，包含:
- 设备标识(devicePhyId_, deviceLogicId_)
- 网络配置(localIp_, backupIp_, nicDeployment_)
- 协议类型(protoType_: ROCE/BUS/TCP)
- RMA buffer管理器(localIpcRmaBufferMgr_, localRdmaRmaBufferMgr_)

InitV2按协议+部署位置创建handle(hccl_network.cc:82-172):
- Device+ROCE → CreateRdmaHandle
- Device+BUS → CreateVnicSocketHandle
- Device+TCP → CreateNicSocketHandle
- HOST+TCP → CreateHostSocketHandle
- HOST+ROCE → CreateRdmaHandle

#### netdev/ 反模式

GetBusAddr/GetNicAddr每次临时InitV2 → 查询 → DeInitV2，低效。
出处: hccl_net_dev_v1.cc:135-174

---

### 子模块 G: socket/ — Socket通信

#### HcclSocket设计

两种模式:
- 客户端: 构造(tag, remoteIp, remotePort) → Connect
- 监听端: 构造(localPort) → Listen → Accept

状态机: INIT → CONNECTING → OK / TIMEOUT / ERROR

关键惯用法:
- VNIC_TYPE使用强制断链(forceClose_=true)
- Accept带超时循环(对比GetExternalInputHcclLinkTimeOut)
- 50倍采样日志打印避免日志爆炸
- ISend/IRecv返回已传大小(半异步)
- SendAsync/RecvAsync完全异步

出处: hccl_socket.cc:80-505

#### socket/ 反模式

hrtRaSocketBatchClose失败返回WARNING而非Error(hccl_socket.cc:284)，可能隐藏问题。

---

### 子模块 H: stream/ — 流管理

#### Stream所有权模型

与DeviceMem/HostMem相同的owner_模式(stream.cc):
- 从StreamType构造: owner=true, 调用hrtStreamCreate
- 从rtStream_t包装: owner=false
- 拷贝: owner=false
- 移动: 转移所有权
- 析构: 仅owner调用hrtStreamDestroy

#### SQE环形缓冲

工作流任务通过SQE缓冲提交(stream.cc:293-368):
- tailSqeIdx达到HCCL_SQE_MAX_CNT(2048)时清空
- taskId到UINT16_MAX时翻转flipNum并递增
- profTimestamp记录每个SQE的CPU时间戳(性能分析集成)

#### TaskLogicInfo队列

std::queue<TaskLogicInfo>支持FIFO追踪任务执行逻辑类型。
TransportVirtural通过PushTaskLogicInfo记录任务而非实际执行。

出处: stream.cc:218-368

#### stream/ 反模式

移动赋值(stream.cc:155-174)中"that.taskLogicInfo_ = taskLogicInfo_"逻辑可疑:
移动后应清空that，但此处将this的队列赋给了that。

---

### Cross-Cutting Idioms (跨子模块惯用法)

#### 1. owner_ + move语义的所有权管理

出现于: DeviceMem, HostMem, Stream, LocalNotifyImpl(notifyOwner_)

模式: 布尔标志区分"拥有资源"vs"引用资源"，析构时仅owner释放。
替代shared_ptr的理由: 轻量、与C API兼容、显式语义。

强制程度: 项目级强制(所有值语义资源类)
出处: mem_device.cc, mem_host.cc, stream.cc

#### 2. Pimpl模式

出现于: Transport, LocalIpcNotify, RemoteNotify, NotifyPool, 所有RmaBuffer

模式: 公开类持有unique_ptr<Impl>或raw Impl*，隐藏实现。
公开头文件在pub_inc/，Impl在src/。

强制程度: 项目级强制(所有对外暴露的资源类)
出处: transport.cc, local_ipc_notify.cc, local_ipc_rma_buffer.cc 等

#### 3. 引用计数去重

出现于: MemMappingManager, MemNameRepository, g_devAddrIdentifierMap, NotifyPool

模式: 同一底层资源被多个上层对象共享时，用引用计数延迟释放。
共同特征: Ref()/Unref()接口, Unref()返回新计数值, ==0时执行清理。

强制程度: 平台层强制(资源共享场景)
出处: mem_mapping_manager.cc, mem_name_repository.cc, local_rdma_rma_buffer_impl.cc

#### 4. 静态资源 + 实例引用计数

出现于: TransportP2p, TransportIbverbs (notifyValueMem_, notifyValueMutex_, instanceRef_)

模式: 同一设备的多个Transport实例共享静态资源(如notify值缓冲区)。
第一个实例创建时分配，最后一个析构时释放。

```cpp
static std::array<DeviceMem, MAX_MODULE_DEVICE_NUM> notifyValueMem_;
static std::array<Referenced, MAX_MODULE_DEVICE_NUM> instanceRef_;
// 构造: instanceRef_[devId].Ref()
// 析构: if (instanceRef_[devId].Unref() == 0) notifyValueMem_.free()
```

强制程度: 平台层推荐
出处: transport_p2p_pub.h, transport_ibverbs_pub.h

#### 5. 工厂选择(runtime条件)

出现于:
- Transport: 根据TransportType枚举创建具体实现
- LocalNotifyImpl: 根据localDeviceId/remoteDeviceId选择Notify类型
- DispatcherCtx: 根据DevType+平台+配置选择Dispatcher
- NetDev: 根据DevType选择v1/v2
- NetDevContext::InitV2: 根据protoType+deployment选择handle

强制程度: 项目级强制(所有多实现场景)

#### 6. 二进制序列化(ostringstream)

出现于: RmaBuffer的Serialize/Deserialize, NotifyBase的Serialize/Deserialize

模式: 使用ostringstream写入POD结构体的原始字节，istringstream读出。
序列化结果缓存(serializeStr_)避免重复生成。

强制程度: 平台层推荐
出处: local_ipc_rma_buffer_impl.cc:57-77, notify_base.h:91-106

#### 7. 异步重试 + 超时控制

出现于: HcclSocket::Accept, TransportHeterogP2p连接状态机

模式: 循环 + SaluSleep + GetExternalInputHcclLinkTimeOut()比较。
日志采样(每N次打印一次)避免日志爆炸。

出处: hccl_socket.cc:372-427, transport_heterog_p2p.cc:119-134

---

### Architecture & Patterns (架构模式)

#### 1. Transport继承体系

```
TransportBase (70+ virtual methods)
├── TransportP2p (同节点共享内存)
├── TransportNet → TransportTcp, TransportIbverbs, TransportDirectNpu
├── TransportRoce (RoCE + TransportHeterogRoce双继承)
├── TransportShmEvent (共享内存 + TransportHeterog双继承)
└── TransportVirtural (任务记录代理)
```

Device侧独立继承链:
- TransportDeviceP2p ← TransportP2p
- TransportDeviceIbverbs ← TransportIbverbs

Heterog独立继承链:
- TransportHeterogP2p, TransportHeterogRoce, TransportHeterogEventRoce/Tcp

#### 2. 资源管理统一模式

所有资源类遵循: 分配 → 初始化 → 使用 → 销毁 → 释放

| 资源 | 分配 | 初始化 | 销毁 | 释放 |
|------|------|--------|------|------|
| DeviceMem | alloc() | - | free() | ~DeviceMem() |
| Transport | new | Init() | DeInit() | ~Transport() |
| Notify | CreateNotify | Init() | Destroy() | ~LocalNotifyImpl() |
| RmaBuffer | new | Init() | Destroy() | ~Impl() |
| Stream | StreamType构造 | InitStream() | DestroyStream() | ~Stream() |
| QP | CreateQp() | ConnectQp() | DestroyQP() | (随Transport) |
| MR | RegUserMem() | - | DeRegMR() | (随Transport) |

---

### Hardware Abstractions (硬件抽象)

#### 1. 设备类型适配

关键设备类型及其影响:
- DEV_TYPE_910B / DEV_TYPE_910_93: PCIe TH参数、FFTS模式、多QP扩展模式
- DEV_TYPE_910_95: NetDev v2接口
- HOST_DEVICE_ID: BareNotify、EschedNotify场景

#### 2. 通信链路分类

```
同节点:
  HCCS (High-speed Chip-to-Chip)  ← P2P/DeviceP2p
  HCCS_SW (HCCS via Switch)       ← A3 DIE间
  SIO (Serial I/O)                ← 低速备选
  PCIe                            ← Host-Device

跨节点:
  RoCE (RDMA over Converged Ethernet) ← IBVerbs, RoCE, DeviceIbverbs
  TCP                                  ← TransportTcp

异构:
  共享内存 + 事件驱动               ← ShmEvent
  IPC (进程间通信)                 ← P2P(同进程flag)
```

#### 3. 内存层次

```
Device Memory (hrtMalloc/hrtFree):
  - DeviceMem管理
  - 需要VA映射才能从Host访问
  - 4K页对齐约束

Host Memory:
  - RTS分配 (hrtMallocHost/hrtFreeHost): 运行时管理的DMA-safe内存
  - 标准new: 普通主机内存
  - HostMem通过isRtsMem_标志区分

映射:
  - MemMappingManager: Host→Device VA映射
  - 910B: HOST_MEM_MAP_DEV_PCIE_TH
  - 其他: HOST_MEM_MAP_DEV

RDMA Memory Region:
  - MR注册: hrtRaRegGlobalMr
  - lkey(本地访问) / rkey(远程访问)
  - g_devAddrIdentifierMap引用计数防止重复注销
```

#### 4. 内存屏障

```cpp
// Device IBVerbs (transport_device_ibverbs.cc:31-38)
#if defined(__x86_64__)
#define HCOMM_DSB() asm volatile("" ::: "memory")     // compiler fence
#elif defined(__aarch64__)
#define HCOMM_DSB() asm volatile("dsb st" ::: "memory") // store barrier
#endif

// HDCommunicate (hdc.cc:187)
std::atomic_thread_fence(std::memory_order_seq_cst);   // full fence

// 原子操作 (transport_device_ibverbs.cc:600, transport_roce_mem.cc:31)
wrIdOffset_.fetch_add(1, std::memory_order_relaxed);   // 高频计数用relaxed
```

---

### Anti-Patterns (反模式汇总)

1. 全局可变状态: g_devAddrIdentifierMap, transportMap_, gQpnToTransportMap等
   多处使用全局static map + mutex，测试隔离困难，并发风险。
   正确做法: 考虑将状态绑定到对象生命周期或使用ConcurrentMap。

2. 临时初始化-查询-销毁: NetDev的GetBusAddr每次临时InitV2→查询→DeInitV2
   出处: hccl_net_dev_v1.cc:135-174
   正确做法: 缓存初始化结果或延迟销毁。

3. TCP发送队列切换: 三元运算符嵌套(tcp_send_thread_pool.cc:260-264)
   正确做法: 拆分为if-else。

4. 移动赋值bug: Stream::operator=(Stream&&)中that.taskLogicInfo_被赋值为this的内容
   出处: stream.cc:155-174
   正确做法: std::move(taskLogicInfo_)到this，清空that。

5. 空操作stub: EschedNotify的大量空实现、LocalIpcNotify::SetEventIdAndTid的临时对象
   可能代表未完成的功能或设计缺陷。

6. Socket close错误吞掉: hrtRaSocketBatchClose失败仅打WARNING
   出处: hccl_socket.cc:284
   正确做法: 至少记录ERROR级别日志。

---

## 2.1.3 hccp/ — HCCP集合通信协议栈

### Module Overview

hccp/ (HCCP = HCCN Communication Protocol) 是一个纯C模块(非C++)，实现Host与Ascend NPU之间的
RDMA/UB通信协议栈。它是hcomm platform层中最大的子模块之一(237文件/50目录)，
承担了所有网络相关操作的底层协议实现。

在架构中的位置:
```
hccl → hcomm framework → hcomm platform → [hccp] → HDC/Peer → Device Driver
```

核心职责:
1. 抽象Host-Device通信通道(HDC/Peer双路径)
2. 实现基于opcode的RPC协议(110+操作码)
3. 管理RDMA和UB(UDMA)双协议栈
4. 提供Socket/QP/MR/CQ等网络资源的生命周期管理

与resource/子模块的关系: resource/中的Transport类(如TransportIbverbs, TransportRoCE)
最终调用hccp/的公共API(如hRaQpCreate, hRaSendWr)来完成实际硬件操作。

### Key Files

公共头文件 (7个，定义外部API):
- `inc/network/hccp.h` (1088行): ~60个导出函数，覆盖Socket/QP/MR/Init/Epoll/Ping/CRIU
- `inc/network/hccp_common.h` (664行): 类型定义、错误码、枚举(NetworkMode, ProtocolTypeT等)
- `inc/network/hccp_ctx.h` (1089行): 新统一API，union-based RDMA/UB协议抽象
- `inc/network/hccp_async.h` (139行): 异步操作包装器
- `inc/network/hccp_async_ctx.h` (216行): 异步ctx操作
- `inc/network/hccp_tlv.h` (83行): TLV消息(NSLB/CCU模块)
- `inc/network/hccp_ping.h` (279行): 网络诊断Ping

私有头文件 (5个，定义内部协议):
- `inc/private/network/ra_rs_comm.h` (371行): 核心协议定义，110+ opcode枚举 + 内部消息结构
- `inc/private/network/ra_rs_err.h` (24行): 自定义errno扩展(ESAFEFUNC=256等)
- `inc/private/network/ra_rs_ctx.h`: ctx操作的内部类型
- `inc/private/network/user_log.h` (134行): 三路编译时日志选择 + CHK_PRT_RETURN宏
- `inc/private/network/hccp_msg.h` (75行): #pragma pack(1)紧凑消息结构

rdma_agent/ (Host/NPU客户端):
- `client/ra_init.c` (243行): 全局状态gRaRdevHandle[64]，引用计数多实例管理
- `hdc/ra_hdc.h` (244行): HDC通道类型定义(HdcInfo, HdcAsyncInfo, MsgHead, HdcOps vtable)
- `hdc/ra_hdc.c` (1037行): HDC通道核心实现(消息收发、初始化、版本协商)
- `hdc/async/ra_hdc_async.c`: 异步HDC通道(请求/响应列表管理)
- `adapter/ra_adp.c` (~2055行): 设备侧消息分发(gRaOpHandle[118], RaHandle(), RsOps vtable, token bucket限流)
- `adapter/ctx/ra_adp_ctx.h` (98行): ctx操作的vtable(RsCtxOps, 30+函数指针)
- `peer/ra_peer.h` (195行): Peer直连路径(NETWORK_PEER_ONLINE模式)
- `inc/ra.h` (219行): 客户端核心类型(RaRdmaHandle, RaSocketHandle, RaQpHandle, RaListHead)

rdma_service/ (Device侧服务端):
- `rs.h` (364行): 服务端API + ProductType枚举 + 内联helpers
- `rs.c` (~2191行): 服务端实现(`__thread struct rs_cb *gRsCb`, gInterfaceInfoList版本表)
- `ctx/rs_ctx.h` (90行): ctx服务端API

hccp_service/:
- `main.c` (162行): 设备侧服务入口(HccpInit/HccpDeinit)
- `param.h`: 配置参数 + STATIC宏定义

### Idioms & Conventions (惯用法)

#### 1. CHK_PRT_RETURN — hccp的错误检查宏

hccp使用CHK_PRT_RETURN而非hcomm C++层的CHK_RET。这是纯C模块的等价物:

```c
// inc/private/network/user_log.h (推测定义)
// 用法: CHK_PRT_RETURN(条件, 日志语句, 返回值)
CHK_PRT_RETURN(reqHandle == NULL || reqResult == NULL,
    hccp_err("[get][async]req_handle or req_result is NULL"),
    ConverReturnCode(OTHERS, -EINVAL));
```

与CHK_RET的区别:
- CHK_RET: 条件为true时返回错误码，自动打日志 (C++层)
- CHK_PRT_RETURN: 条件为true时执行自定义日志语句后返回指定值 (C层)

量化: 1546次使用，覆盖61个文件 (hccp模块内最高频宏)。

出处: 几乎所有.c文件均大量使用

#### 2. securec安全函数

全面替代标准C库的内存/字符串操作:

```c
// ra_hdc.c (RaHdcProcessMsg函数)
ret = memcpy_s(sendBuf + sizeof(struct MsgHead),
    sendLen - sizeof(struct MsgHead), data, dataSize);   // 替代 memcpy
ret = memset_s(sendBuf, sendLen, 0, sendLen);            // 替代 memset
ret = snprintf_s(buf, len, len - 1, "%s", src);          // 替代 snprintf
ret = strcpy_s(dst, dstLen, src);                        // 替代 strcpy
```

所有securec调用都必须检查返回值:
```c
if (ret != EOK) {
    hccp_err("memcpy_s failed ret[%d]", ret);
    // 释放资源后返回 -ESAFEFUNC (256)
}
```

量化: 463次使用，覆盖49个文件。
出处: 整个hccp模块。自定义errno: ESAFEFUNC=256 (ra_rs_err.h:16)

#### 3. STATIC宏 — 测试可见性控制

三处独立定义(完全一致):
```c
// rs.h:66-70, ra.h:25-29, param.h:15-19
#if defined(HNS_ROCE_LLT) || defined(DEFINE_HNS_LLT)
#define STATIC
#else
#define STATIC static
#endif
```

- 生产构建: STATIC = static (内部链接，符号不导出)
- 测试构建(LLT): STATIC = 空 (全局链接，测试可直接调用)

模式: 所有需要单元测试覆盖的static函数都用STATIC替代，
测试代码通过编译宏HNS_ROCE_LLT/DEFINE_HNS_LLT暴露它们。

出处: rs.h:66-70, ra.h:25-29, param.h:15-19

#### 4. union-based tx/rx数据协议

每个opcode对应一个union，txData定义请求参数，rxData定义响应参数:

```c
// ra_hdc.h:95-107
union OpSetPidData {
    struct {
        unsigned int phyId;
        pid_t pid;
        unsigned int rsvd[RA_RSVD_NUM_2];
        char pidSign[PROCESS_RA_SIGN_LENGTH];
        char resv[PROCESS_RA_RESV_LENGTH];
    } txData;          // 客户端填充，发送到设备

    struct {
        unsigned int rsvd[RA_RSVD_NUM_4];
    } rxData;          // 设备端填充，返回给客户端
};
```

设计特点:
- union确保tx和rx共享同一块内存(节省分配)
- 每个union的sizeof就是该opcode的协议数据大小
- gRaOpHandle[]表中用sizeof(union OpXxxData)记录每个opcode的期望数据大小

出处: ra_hdc.h (6个union), ra_rs_comm.h (大量内部union)

#### 5. 函数指针vtable(C语言多态)

三个核心vtable实现C语言的多态分发:

a) HdcOps (18个函数指针，HDC通道操作):
```c
// ra_hdc.h:183-204
struct HdcOps {
    hdcError_t (*getCapacity)(struct drvHdcCapacity *capacity);
    hdcError_t (*clientCreate)(HDC_CLIENT *client, int maxSessionNum, int serviceType, int flag);
    hdcError_t (*sessionConnect)(int peerNode, int peerLogicid, HDC_CLIENT client, HDC_SESSION *session);
    hdcError_t (*send)(HDC_SESSION session, struct drvHdcMsg *msg, unsigned long long flag, unsigned int timeout);
    hdcError_t (*recv)(HDC_SESSION session, struct drvHdcMsg *msg, int bufLen, unsigned long long flag,
                       int *recvBufCount, unsigned int timeout);
    // ...
};
```

b) RsOps (43个函数指针，RDMA服务操作):
```c
// ra_adp.c:44-109
struct RsOps {
    int (*rdevInit)(struct rdev rdevInfo, unsigned int notifyType, unsigned int *rdevIndex);
    int (*qpCreate)(struct QpCreateParam *param, struct QpCreateResp *resp);
    int (*qpDestroy)(unsigned int qpn);
    int (*mrReg)(struct MrRegParam *param, struct MrRegResp *resp);
    // ...43个函数指针
};
```

c) RsCtxOps (30+函数指针，ctx操作):
```c
// ra_adp_ctx.h:25-97
struct RsCtxOps {
    int (*ctxInit)(struct CtxInitAttr *attr, unsigned int *devIndex, struct DevBaseAttr *devAttr);
    int (*ctxQpCreate)(struct RaRsDevInfo *devInfo, struct CtxQpAttr *qpAttr, struct QpCreateInfo *qpInfo);
    // ...
};
```

初始化方式 — C99指定初始化器:
```c
// ra_adp.c:111-159
struct RsOps gRaRsOps = {
    .rdevInit = RsRdevInit,
    .rdevInitWithBackup = RsRdevInitWithBackup,
    .qpCreate = RsQpCreate,
    .qpDestroy = RsQpDestroy,
    // ...
};
```

调用方式:
```c
// ra_adp.c:233
*opResult = gRaRsOps.rdevInit(rdevInitData->txData.rdevInfo, NOTIFY, &rdevIndex);
```

出处: ra_hdc.h:183-204, ra_adp.c:44-159, ra_adp_ctx.h:25-97

#### 6. 全局数组 + phyId索引

整个模块以物理设备ID (phyId) 为索引，使用全局数组管理per-device状态:

```c
// ra_init.c: 客户端侧
struct RaRdmaHandle gRaRdevHandle[RA_MAX_PHY_ID_NUM];  // RA_MAX_PHY_ID_NUM=64
struct RefInstances gRefInstances[RA_MAX_PHY_ID_NUM];   // 引用计数

// ra_hdc.c: HDC通道状态
struct HdcInfo gRaHdc[RA_MAX_PHY_ID_NUM];               // 同步HDC信息
struct HdcAsyncInfo gRaHdcAsync[RA_MAX_PHY_ID_NUM];     // 异步HDC信息

// rs.c: 服务端侧
struct rs_cb *gRsCbList[RS_MAX_DEV_NUM];                 // RS_MAX_DEV_NUM=64
int gInitCounter[RS_MAX_DEV_NUM];

// ra_adp.c: 适配器层
struct HdcServer gHdcServer[RA_MAX_PHY_ID_NUM];
```

模式: 几乎所有函数的第一个参数都是 unsigned int phyId，用于定位全局数组中的设备上下文。
这是C语言中的"对象"管理方式 — 全局数组替代对象池。

出处: 覆盖整个模块

#### 7. pthread_mutex保护 + RA_PTHREAD_MUTEX_LOCK宏

自定义mutex操作宏，失败时打warning而不abort:

```c
// ra_hdc.h:206-218
#define RA_PTHREAD_MUTEX_LOCK(mutex) do { \
    int ret_lock = pthread_mutex_lock(mutex); \
    if (ret_lock) { \
        hccp_warn("pthread_mutex_lock unsuccessful, ret[%d]", ret_lock); \
    } \
} while (0)

#define RA_PTHREAD_MUTEX_UNLOCK(mutex) do { \
    int ret_ulock = pthread_mutex_unlock(mutex); \
    if (ret_ulock) { \
        hccp_warn("pthread_mutex_unlock unsuccessful, ret[%d]", ret_ulock); \
    } \
} while (0)
```

设计选择: mutex失败时记录warning而非abort，这允许系统在异常状态下继续运行
(容错优先于fail-fast)。与内核驱动风格一致。

量化: 192次使用，覆盖24个文件。
出处: ra_hdc.h:206-218

#### 8. HCCP_ATTRI_VISI_DEF 导出符号

```c
// 定义 (推测在公共头文件中)
#define HCCP_ATTRI_VISI_DEF __attribute__((visibility("default")))
```

所有公共API函数都使用此宏标记，配合 -fvisibility=hidden 编译选项，
仅导出显式标记的符号，减少DSO加载开销和符号冲突。

出处: hccp.h, hccp_async.h 中所有公共函数声明

#### 9. 日志体系: 三路编译时选择

```c
// user_log.h: 根据编译环境选择日志后端
#ifdef DRV_HOST
    // 路径1: 驱动侧 — 使用驱动日志宏
#elif defined(LOG_HOST)
    // 路径2: 主机侧 — 使用 HCCPDlogForC
#else
    // 路径3: 默认 — 使用 hccp_dlog
#endif
```

两个模块ID: HCCP(带tid)和NET/roce(不带tid)。
RUN_LOG_MASK区分操作日志和诊断日志。

量化: 日志宏(hccp_info/hccp_err/hccp_warn/hccp_debug) 共3330次使用，覆盖64个文件。
出处: user_log.h:1-134

#### 10. HCCP_CLOSE_RETRY_FOR_EINTR — 安全关闭fd

```c
// ra_peer.h:20-29
#define HCCP_CLOSE_RETRY_FOR_EINTR(fd) do { \
    int ret_; \
    do { \
        ret_ = close(fd); \
        if (ret_ < 0) { \
            hccp_warn("close filedscp[%d] unsuccessful, errno:%d", fd, errno); \
        } \
    } while ((ret_ < 0) && (errno == EINTR)); \
    fd = -1; \
} while (0)
```

处理close()被信号中断的情况，重试直到非EINTR错误，最后置fd=-1防止双关闭。
这是POSIX编程中的标准模式，但在用户态程序中close()返回EINTR后
fd的状态是undefined的(不同于read/write)，所以重试close()可能有问题。

出处: ra_peer.h:20-29

### Architecture & Patterns (架构模式)

#### 1. Client-Server 双侧架构

```
Host/NPU (rdma_agent)                    Device (rdma_service)
+------------------+                     +------------------+
| ra_init.c        |                     | rs.c             |
|  RaInit()        |                     |  RsInit()        |
|  gRaRdevHandle[] |                     |  gRsCbList[]     |
+------------------+                     +------------------+
        |                                        |
        v                                        v
+------------------+    HDC Channel     +------------------+
| ra_hdc.c         |==================>| adapter/         |
|  RaHdcProcessMsg |   MsgHead+Data    |  ra_adp.c        |
|  gRaHdc[phyId]   |                   |  RaHandle()      |
+------------------+                   |  gRaOpHandle[118]|
        |                              +------------------+
        |  OR (PEER_ONLINE)                     |
        v                                       v
+------------------+                   +------------------+
| ra_peer.c        |                   | RsOps vtable     |
|  RaPeerInit()    |                   |  gRaRsOps.xxx()  |
|  Direct path     |                   |  → Rs* functions |
+------------------+                   +------------------+
```

路径选择基于NetworkMode:
- NETWORK_OFFLINE: 走HDC通道(Host-Device Communication driver)
- NETWORK_PEER_ONLINE: 走Peer直连(跳过HDC，直接调用Rs*函数)

出处: ra_init.c (RaInit路由), ra_peer.h (Peer路径声明)

#### 2. Opcode-based RPC协议

消息格式:
```
+------------------------------+
| MsgHead (28 bytes)           |
|   opcode (4)                 |  ← enum OpType (110+ 外部 + 内部)
|   ret (4)                    |  ← 返回值(设备端填充)
|   asyncReqId/rsvd (4)        |  ← 异步请求ID
|   msgDataLen (4)             |  ← 数据部分长度
|   hostTgid (4)               |  ← 进程组ID
|   pidSign[PROCESS_RA_SIGN_LENGTH] |  ← 进程签名(安全)
+------------------------------+
| union OpXxxData (变长)        |
|   txData / rxData            |  ← 请求/响应共享内存
+------------------------------+
```

协议版本协商:
- 服务端维护 gInterfaceInfoList[] (rs.c:72-195，~130条目)，记录每个opcode的当前版本
- 客户端维护 gRaInterfaceInfoList[] (ra_hdc.c)，记录已协商的版本
- 初始化时 RaHdcGetAllOpcodeVersion() 批量查询所有opcode版本
- 版本号用于向后兼容(如RA_RS_SOCKET_CONN: 版本1→2, 接口增强)

出处: ra_hdc.h:60-70 (MsgHead), ra_rs_comm.h:22-145 (OpType枚举),
rs.c:72-195 (服务端版本表), ra_hdc.c (客户端版本查询)

#### 3. 消息分发: 线性查找 + 函数指针表

设备侧适配器使用全局分发表:

```c
// ra_adp.c:1680-1798
struct RaOpHandle gRaOpHandle[] = {
    {RA_RS_SOCKET_CONN, RaRsSocketBatchConnect, sizeof(union OpSocketConnectData)},
    {RA_RS_SOCKET_CLOSE, RaRsSocketBatchClose, sizeof(union OpSocketCloseData)},
    {RA_RS_QP_CREATE,    RaRsQpCreate,          sizeof(union OpQpCreateData)},
    // ... 共118个条目
};
```

RaHandle()分发流程 (ra_adp.c:1841-1882):
1. RaCheckParam(): 校验消息长度 + opcode有效性 + 数据大小匹配
2. RaGetOpRight(): Token bucket限流检查
3. calloc分配响应缓冲区
4. for循环线性查找gRaOpHandle[].opcode
5. 调用匹配的opHandle(recvBuf, sendBuf, sndBufLen, &opRet, rcvBufLen)
6. MsgHeadBuildUpHw()填充响应头

参数检查细节 (ra_adp.c:1800-1839):
- 检查 rcvBufLen >= sizeof(struct MsgHead)
- 检查 msgDataLen + sizeof(MsgHead) == rcvBufLen (长度一致性)
- 检查 opcode在有效范围内(0-110外部 或 1000+内部)
- 查表验证 msgDataLen == gRaOpHandle[i].dataSize (RA_RS_SOCKET_RECV除外)

注意: 118个条目的线性查找 O(n)，没有使用hash或二分。
对于设备侧RPC这个规模(每次调用涉及HDC传输开销)，线性查找的开销可以忽略。

出处: ra_adp.c:1680-1882, ra_adp.h:49-53

#### 4. Token Bucket限流

```c
// ra_adp.h:19-21
#define BUCKET_DEPTH    (1024 * 2)  // 桶容量: 2048
#define TOKEN_RATE      400         // 每毫秒补充400个token
```

RaGetOpRight() (ra_adp.c:1627-1678):
```c
// 1. 计算距上次操作的时间间隔(毫秒)
RaTimeInterval(&tCur, &opSec->tLast, &timeInterval);
// 2. 补充token (时间间隔 * TOKEN_RATE)
opSec->tokenNum += timeInterval * TOKEN_RATE;
// 3. 检查并消耗1个token
if (opSec->tokenNum == 0) {
    *right = HAVE_NOT_OP_RIGHT;  // 拒绝
    return;
}
opSec->tokenNum--;
// 4. 限制桶容量不超过BUCKET_DEPTH
opSec->tokenNum = (opSec->tokenNum > BUCKET_DEPTH) ? BUCKET_DEPTH : opSec->tokenNum;
```

每个设备(gHdcServer[chipId])有独立的token bucket(opSec字段)。
初始化: RaHdcInitOpSec(&gHdcServer[chipId].opSec, BUCKET_DEPTH, false)

出处: ra_adp.c:1612-1678, ra_adp.h:19-21

#### 5. 同步/异步双通道

同步路径 (ra_hdc.c):
```
RaHdcProcessMsg(opcode, phyId, data, dataSize)
  → calloc(MsgHead + data)
  → MsgHeadBuildUp()
  → memcpy_s(data)
  → HdcSendRecvPkt()  // mutex保护, 阻塞等待响应
  → MsgHeadCheck()     // 校验响应
  → memcpy_s(结果)
  → free()
```

异步路径 (ra_hdc_async.c):
```
RaHdcSendMsgAsync(opcode, phyId, data, dataSize, reqHandle)
  → RaHdcIsAsyncOp(opcode)     // 查找异步操作表
  → calloc(MsgHead + data)
  → reqId = gRaHdcAsync[phyId].reqId++  // 分配单调递增ID
  → HdcAsyncSetRequest(reqHandle)        // 初始化请求句柄
  → HdcAsyncSendPkt()                   // 发送(不等待)

// 接收线程:
  → HdcAsyncRecvPkt()                   // 在独立线程中接收响应
  → HdcAsyncSetReqDone(reqHandle)       // isDone=true, 加入rspList

// 调用方轮询:
RaGetAsyncReqResult(reqHandle, &reqResult)
  → if (!isDone) return OTHERS_EAGAIN (128301)
  → *reqResult = opRet
  → HdcAsyncDelResponse()  // 释放句柄
```

异步请求句柄生命周期:
1. calloc分配 (isDone=false)
2. HdcAsyncSetRequest: 设置reqId, opHandle, phyId
3. 发送后等待接收线程完成
4. HdcAsyncSetReqDone: isDone=true, opRet赋值
5. RaGetAsyncReqResult: 返回结果并free

出处: ra_hdc.c (同步), ra_hdc_async.c (异步), ra_async.h:17-33 (RaRequestHandle),
hccp_async.h:21-30 (RaGetAsyncReqResult API)

#### 6. 引用计数多实例管理

```c
// ra_init.c
struct RefInstances {
    int refCount;
    pthread_mutex_t mutex;
};
struct RefInstances gRefInstances[RA_MAX_PHY_ID_NUM];

// 首次初始化检查
int RaIsFirstUsed(unsigned int phyId) {
    // mutex保护下检查 refCount==0
}

// 最后一个释放检查
int RaIsLastUsed(unsigned int phyId) {
    // mutex保护下检查 refCount减到0
}
```

多进程/多线程同时使用同一设备时，通过引用计数确保:
- 第一个使用者执行完整初始化(RaInitHdc或RaInitPeer)
- 后续使用者仅增加引用计数
- 最后一个释放者执行完整清理

出处: ra_init.c (RaIsFirstUsed/RaIsLastUsed)

#### 7. __thread TLS for Server Context

```c
// rs.c:50-51
__thread struct rs_cb *gRsCb = NULL;       // 线程本地
struct rs_cb *gRsCbList[RS_MAX_DEV_NUM] = {0};  // 全局

void RsSetCtx(unsigned int phyId) {
    gRsCb = gRsCbList[phyId];  // 绑定当前线程到特定设备
}
```

每个HDC会话在独立线程中处理，线程进入前调用RsSetCtx()绑定设备上下文。
后续所有Rs*函数通过gRsCb直接访问当前设备状态，无需传递参数。

出处: rs.c:50-70

### Domain Knowledge (业务知识)

#### 1. 双协议抽象: RDMA vs UB(UDMA)

hccp_ctx.h通过union实现RDMA和UB(Unified Bus)的协议抽象:

```c
// hccp_ctx.h: CtxInitAttr
struct CtxInitAttr {
    union {
        struct { /* RDMA 初始化参数 */ } rdma;
        struct { /* UB 初始化参数 */ } ub;
    };
};

// QP创建属性也是union
struct CtxQpAttr {
    union {
        struct { /* RDMA QP 属性 */ } rdma;
        struct { /* UB Jetty 属性 */ } ub;
    };
};
```

协议类型选择基于ProductType:
- RDMA (RoCE): 910, 910B, 910_93 → ProtocolTypeT::PROTOCOL_RDMA
- UB (UDMA): 950, 910_96 → ProtocolTypeT::PROTOCOL_UB

内联helpers (rs.h):
```c
static inline bool RsIsRdmaSupported(enum ProductType type) {
    return (type == PRODUCT_910 || type == PRODUCT_910B || type == PRODUCT_910_93);
}
static inline bool RsIsUdmaSupported(enum ProductType type) {
    return (type == PRODUCT_950 || type == PRODUCT_910_96);
}
```

Jetty modes (UB特有):
- URMA_NORMAL: 普通模式
- CACHE_LOCK_DWQE: 缓存锁定+直接WQE
- CCU: CCU引擎模式
- USER_CTL_NORMAL: 用户控制模式
- CCU_TA_CACHE: CCU TA缓存模式

出处: hccp_ctx.h (union-based抽象), rs.h (ProductType + helpers),
hccp_common.h (ProtocolTypeT枚举)

#### 2. 操作码分类 (110+ 外部 + 内部)

按功能分组:
- Socket操作 (0-8): CONN, CLOSE, ABORT, LISTEN_START/STOP, GET_SOCKET, SEND, RECV
- QP操作 (9-14, 82+): CREATE, DESTROY, CONNECT, STATUS, INFO, MODIFY, AI_QP_CREATE, TYPICAL_QP_CREATE
- MR操作 (11-13): REG, DEREG, SEND_WR, TYPICAL_MR_REG
- 初始化 (15-16): INIT, DEINIT, RDEV_INIT
- Epoll操作: NOTIFY_CREATE/DESTROY/OPEN, EPOLL_ADD/DEL/WAIT
- Ctx操作 (60+): CTX_INIT/DEINIT, CTX_QP_CREATE/DESTROY/BIND, CTX_MR_REG/DEREG, CTX_CQ_CREATE
- 异步批量: SOCKET_BATCH_CONNECT_ASYNC, QP_CONNECT_ASYNC
- 诊断: PING相关, CQE_ERR_INFO, GET_VERSION
- 内部 (1000+): HDC_SESSION_CLOSE, GET_VNIC_IP, NOTIFY_CFG_SET/GET, SET_PID, ASYNC_HDC_SESSION_*

版本化: 每个opcode有独立版本号(1-3)，支持协议的向后兼容升级。
如RA_RS_GET_SOCKET从版本1升级到版本3，表示接口经历了两次增强。

出处: ra_rs_comm.h:22-145

#### 3. 错误码编码方案

6位数错误码 = 模块前缀(3位) + 标准errno(3位):

```c
// hccp_common.h
#define SOCK_EAGAIN     128201  // Socket模块 + EAGAIN
#define SOCK_ENOENT     228200  // Socket模块(特殊重试) + ENOENT
#define ROCE_EAGAIN     128101  // RoCE模块 + EAGAIN
#define HCCP_EAGAIN     128001  // HCCP通用 + EAGAIN
#define OTHERS_EAGAIN   128301  // 其他模块 + EAGAIN

// ConverReturnCode() 将模块+errno转换为6位码
// 前三位: 128=需要重试, 228=特殊重试, 328=严重错误, 528=不支持
```

自定义errno扩展 (ra_rs_err.h):
- ESAFEFUNC=256: securec函数失败
- EFILEOPER=257: 文件操作失败
- ESYSFUNC=258: 系统调用失败
- EOPENSRC=259: 开源组件失败
- EDEFAULT=260: 默认错误
- ESOCKCLOSED=261: Socket已关闭
- EINVALIDIP=262: 无效IP
- ENOTSUPP=524: 不支持的操作

出处: hccp_common.h:60-80, ra_rs_err.h:16-24

#### 4. CRIU快照(Checkpoint/Restore)

支持进程级快照保存和恢复(用于容错/迁移):

SaveSnapshot两阶段:
```c
// PRE_PROCESSING: 保存session到backup，清空活跃session
action == SAVE_SNAPSHOT_ACTION_PRE_PROCESSING:
    gRaHdc[phyId].snapshotSession = gRaHdc[phyId].session;
    gRaHdc[phyId].session = NULL;

// POST_PROCESSING: 从backup恢复session
action == SAVE_SNAPSHOT_ACTION_POST_PROCESSING:
    gRaHdc[phyId].session = gRaHdc[phyId].snapshotSession;
    gRaHdc[phyId].snapshotSession = NULL;
```

RestoreSnapshot:
```c
// 设置restoreFlag，后续操作检查此flag跳过实际执行
gRaHdc[phyId].restoreFlag = 1;
// 异步通道也设置restoreFlag
gRaHdcAsync[phyId].restoreFlag = 1;
```

异步通道的snapshot需要同时锁定sendMutex和recvMutex，
防止快照期间有正在进行的消息收发。

出处: ra_hdc.c:956-1004 (同步), ra_hdc_async.c:684-735 (异步)

#### 5. HDC通道初始化

RaHdcInit() 流程 (ra_hdc.c):
1. DlHalInit(): 加载HAL动态库
2. pthread_mutex_init(): 初始化per-device锁
3. clientCreate(maxSession=2): 创建HDC客户端(最多2个session)
4. sessionConnect(): 连接到设备侧服务
5. setSessionReference(): 设置session引用(热迁移支持)
6. sendPid(): 发送进程标识到设备侧(安全认证)
7. RaHdcGetAllOpcodeVersion(): 批量协商所有opcode版本

断线检测:
```c
// ra_hdc.h:220-223
static inline bool RaHdcIsBroken(int lastRecvStatus) {
    return lastRecvStatus == DRV_ERROR_SOCKET_CLOSE;
}
```
每次收发后更新lastRecvStatus，后续操作先检查是否已断线。

出处: ra_hdc.c (RaHdcInit, HdcSendRecvPkt)

### Hardware Abstractions (硬件知识)

#### 1. HDC (Host-Device Communication)

HDC是Ascend芯片提供的Host-NPU通信驱动接口。
HdcOps vtable封装了18个驱动级操作:

关键概念:
- HDC_CLIENT: 客户端句柄(per进程/per模块)
- HDC_SESSION: 会话句柄(per连接)
- drvHdcMsg: 驱动级消息结构(HDC API定义)
- serviceType: 区分不同的HDC服务类型
- peerNode/peerDevid: 标识目标设备

流量控制:
- send/recv的flag参数控制阻塞/非阻塞模式
- timeout参数支持超时控制
- HdcSendRecvPkt在DRV_ERROR_WAIT_TIMEOUT时自动重试

出处: ra_hdc.h:183-204 (HdcOps)

#### 2. 产品型号与协议映射

```c
// rs.h
enum ProductType {
    PRODUCT_310P,     // 推理卡
    PRODUCT_910,      // 训练卡(初代)
    PRODUCT_910B,     // 训练卡(二代)
    PRODUCT_910_93,   // 训练卡(三代)
    PRODUCT_950,      // 新一代(UB协议)
    PRODUCT_910_96,   // 新架构(UB协议)
};
```

产品与通信协议:
- 910/910B/910_93: RDMA (RoCE v2)
- 950/910_96: UB (UDMA, Unified DMA)

产品目录: hccp/下有ascend910_95/, ascend910_96/, stub/子目录，
包含产品特定的代码路径。

出处: rs.h (ProductType枚举)

#### 3. x86/aarch64平台差异

```c
// hccp_common.h
#ifdef __x86_64__
#define EPOLL_PACKED __attribute__((packed))
#else
#define EPOLL_PACKED
#endif
```

x86上Epoll相关结构需要packed属性确保布局一致，
aarch64上自然对齐即满足要求。

出处: hccp_common.h

#### 4. SocketHdcInfo vs Peer路径

```c
// ra_hdc.h:23-28
struct SocketHdcInfo {
    unsigned int phyId;
    int fd;
    void *socketHandle;
    uint32_t rsv;  // consistent with socket_peer_info
};
```

HDC路径: 所有网络操作通过HDC RPC传递到设备侧执行
Peer路径: 直接在当前进程空间调用Rs*函数(跳过HDC序列化/反序列化)

Peer路径适用于: 同一设备上的用户态进程直接访问RDMA资源

出处: ra_hdc.h:23-28, ra_peer.h

### Anti-Patterns (反模式)

#### 1. 线性查找118条目的dispatch表

```c
// ra_adp.c:1862-1874
for (i = 0; i < num; i++) {
    if (gRaOpHandle[i].opcode == recvMsgHead->opcode) {
        // ...
    }
}
```

118个元素的线性查找，每次消息分发都遍历。
虽然HDC传输开销远大于查找开销，但可以用hash map或直接用opcode做数组索引。

正确做法: 如果opcode是连续的(0-110)，可以直接用opcode做数组下标O(1)查找。

出处: ra_adp.c:1862-1874

#### 2. RaCheckParam中的双重线性查找

```c
// ra_adp.c:1825-1836
// RaCheckParam()内部也线性查找gRaOpHandle获取dataSize
for (i = 0; i < num; i++) {
    if (gRaOpHandle[i].opcode == recvMsgHead->opcode) {
        dataSize = gRaOpHandle[i].dataSize;
        break;
    }
}
```

RaHandle()先调用RaCheckParam()(线性查找一次获取dataSize)，
然后自己再线性查找一次分发handler。同一个opcode被查找两次。

正确做法: RaCheckParam返回找到的index，RaHandle直接用index分发。

出处: ra_adp.c:1800-1839 (RaCheckParam), 1862-1874 (RaHandle)

#### 3. HCCP_CLOSE_RETRY_FOR_EINTR的close重试问题

```c
// ra_peer.h:20-29
do {
    ret_ = close(fd);
} while ((ret_ < 0) && (errno == EINTR));
```

POSIX标准中close()返回EINTR后fd的状态是undefined:
- Linux内核: close()总是释放fd，即使返回EINTR
- 重试close()可能关闭了其他线程新打开的fd (race condition)

正确做法(Linux): close()不需要重试EINTR，只需关闭一次。

出处: ra_peer.h:20-29

#### 4. 三处重复的STATIC宏定义

rs.h:66-70, ra.h:25-29, param.h:15-19 三处完全相同的#define STATIC。

正确做法: 放在公共头文件中定义一次。可能是因为三个子模块独立编译，
没有共享的公共头文件，但仍然是代码重复。

出处: rs.h:66-70, ra.h:25-29, param.h:15-19

#### 5. 全局数组硬编码上限

```c
struct HdcInfo gRaHdc[RA_MAX_PHY_ID_NUM];  // RA_MAX_PHY_ID_NUM=64
struct rs_cb *gRsCbList[RS_MAX_DEV_NUM];    // RS_MAX_DEV_NUM=64
```

多处全局数组使用硬编码上限64，没有运行时检查phyId是否越界。
如果物理设备ID超过64(未来硬件扩展)，将导致越界访问。

正确做法: 所有使用phyId索引的地方应有边界检查(CHK_PRT_RETURN)。

出处: ra_init.c, ra_hdc.c, rs.c

#### 6. RA_PTHREAD_MUTEX_LOCK失败仅打warning

```c
// ra_hdc.h:206-211
if (ret_lock) {
    hccp_warn("pthread_mutex_lock unsuccessful, ret[%d]", ret_lock);
}
// 继续执行! 不返回错误
```

mutex加锁失败后继续执行临界区代码，可能导致数据竞争。
在容错场景下可以理解(避免死锁挂起整个系统)，
但应该至少标记当前操作为不可靠。

出处: ra_hdc.h:206-218

### Notable Patterns (值得注意的模式)

#### 1. C语言 vs C++层的风格对比

hccp是纯C模块，与hcomm其他C++模块存在显著风格差异:

| 维度 | hccp (C) | hcomm其他 (C++) |
|------|----------|-----------------|
| 错误检查 | CHK_PRT_RETURN | CHK_RET / CHK_PTR_NULL |
| 多态 | 函数指针vtable | 虚函数+继承 |
| 内存管理 | calloc/free | new(std::nothrow)/delete, DeviceMem RAII |
| 状态管理 | 全局数组[phyId] | 对象实例 |
| 测试适配 | STATIC宏 | 虚函数+mock |
| 符号导出 | HCCP_ATTRI_VISI_DEF | DLLEXPORT / RS_ATTRI_VISI_DEF |
| 链表 | RaListHead (kernel-style) | std::vector/std::list |
| 字符串 | securec (memcpy_s等) | std::string |
| TLS | __thread | thread_local |

这种差异反映了hccp更接近内核/驱动风格的设计哲学，
强调可预测性和确定性(无异常、无隐式构造/析构)。

#### 2. 客户端-服务端版本协商

初始化时的版本协商确保了协议的向前兼容:

```
Client                           Server
  |                                |
  |-- RaHdcGetAllOpcodeVersion --->|
  |                                | 查 gInterfaceInfoList[opcode].version
  |<--- version for each opcode ---|
  |                                |
  | 存入 gRaInterfaceInfoList      |
  |                                |
  |-- 后续操作按协商版本发送 ------>|
```

这使得:
- 新客户端可以与旧服务端通信(降级到服务端支持的版本)
- 新服务端可以与旧客户端通信(旧客户端只用旧opcode)
- 新opcode不影响已有功能

出处: ra_hdc.c (RaHdcGetAllOpcodeVersion), rs.c:2171-2191 (RsGetInterfaceVersion)

#### 3. 断线检测与恢复

同步通道:
```c
// HdcSendRecvPkt() 中 (ra_hdc.c)
if (RaHdcIsBroken(gRaHdc[phyId].lastRecvStatus)) {
    // 通道已断，立即返回错误，不尝试发送
    return -DRV_ERROR_SOCKET_CLOSE;
}
// 发送消息...
// DRV_ERROR_WAIT_TIMEOUT → 重试
// 其他错误 → 更新lastRecvStatus，后续调用快速失败
```

异步通道:
```c
// RaHdcSendMsgAsync() 中 (ra_hdc_async.c)
CHK_PRT_RETURN(RaHdcIsBroken(gRaHdcAsync[phyId].lastRecvStatus), ...);
```

模式: "fail-fast + last-status cache" — 一旦检测到连接断开，
所有后续调用立即失败，避免重复等待超时。

#### 4. extern "C" 包装

所有公共头文件使用:
```c
#ifdef __cplusplus
extern "C" {
#endif
// ... C API declarations ...
#ifdef __cplusplus
}
#endif
```

确保hccp的C函数可以被hcomm的C++代码正确链接。
这是纯C模块嵌入C++项目的标准做法。

出处: hccp.h, hccp_common.h, hccp_ctx.h等所有公共头文件

#### 5. 产品特定代码的目录组织

```
hccp/
├── ascend910_95/     // 910_95特定实现
├── ascend910_96/     // 910_96特定实现
├── stub/             // 桩实现(编译时替换)
├── rdma_agent/       // 通用客户端
└── rdma_service/     // 通用服务端
```

产品特定代码通过目录隔离+编译选择，而非运行时分支，
避免产品间的代码污染。stub/目录提供空实现，
用于不支持特定功能的产品型号。

#### 6. RaListHead — 内核风格链表

```c
// ra.h
struct RaListHead {
    struct RaListHead *prev;
    struct RaListHead *next;
};
```

配合container_of宏实现侵入式链表，与Linux内核list_head完全一致。
用于异步请求列表(reqList/rspList)的管理。

出处: ra.h, ra_async.h (RaRequestHandle.list)

#### 7. hccp与resource/层的接口关系

hccp的公共API(hccp.h, hccp_ctx.h)是resource/层Transport实现的底层依赖:
```
TransportIbverbs.cc → hRaQpCreate(), hRaSendWr(), hRaMrReg()
TransportRoCE.cc    → hRaSocketBatchConnect(), hRaSocketSend()
NetDevContext.cc    → hRaInit(), hRaRdevInit()
```

hccp对上层完全透明: resource/层只看到hccp的C API函数，
不知道底层是走HDC通道还是Peer直连。
这种抽象使得通信路径的切换对上层完全无感。

---

## 2.1.4 task/ — 任务下发、调度、执行模型

### Module Overview

task/ 是 platform 层的任务下发子模块 (17 files, 2 dirs)，负责将集合通信算法
产生的原子操作 (memcpy, reduce, signal, rdma send) 编排为硬件可执行的任务序列，
通过不同的执行后端 (Runtime Stream / FFTS Graph / AICPU RTSQ) 下发到硬件。

在整体架构中位于 algorithm → framework → task → hardware 的关键路径上，
是算法层"算什么"到硬件层"怎么做"的翻译层。

### Key Files

```
dispatcher.h            — C ABI 外部接口(HcclDispatcherInit/Destroy/MemcpyAsync等)
dispatcher_pub.h        — DispatcherPub 基类定义(70+ 方法), 公共数据结构
dispatcher_graph_pub.h  — DispatcherGraph (FFTS模式), SQE位域定义
dispatcher_aicpu_pub.h  — DispatcherAiCpu (AICPU下沉模式), SQE函数指针表
dispatcher_virtural_pub.h — DispatcherVirtural (虚拟模式, 仅记录不执行)
task_logic_info_pub.h   — TaskLogicInfo 结构体, 用于Virtual模式的逻辑任务记录
dispatcher_common.cc    — C ABI门面函数实现, reinterpret_cast<DispatcherPub*>转发
dispatcher.cc           — DispatcherPub 实现 (1333行), 含HostNIC TCP/RDMA线程
dispatcher_aicpu.cc     — DispatcherAiCpu 实现 (1100+行), SQE编排+RTSQ下发
dispatcher_graph.cc     — DispatcherGraph 实现 (693行), FFTS图模式
dispatcher_virtural.cc  — DispatcherVirtural 实现 (79行), 逻辑记录模式
task_logic_info.cc      — TaskLogicInfo 构造函数实现
rtsq_interact/          — AICPU SQE格式化函数 (v1/v2两版本)
```

### Architecture: Strategy Pattern — Three Dispatch Backends

```
                    DispatcherPub (base)
                    /       |        \
         DispatcherGraph  DispatcherAiCpu  DispatcherVirtural
         (FFTS Graph)    (AICPU RTSQ)     (Virtual/Logic)
```

DispatcherType enum 控制创建:
- DISPATCHER_NORMAL → DispatcherPub (runtime stream) 或 DispatcherGraph (FFTS)
- DISPATCHER_VIRTURAL → DispatcherVirtural
- DISPATCHER_AICPU → DispatcherAiCpu (独立初始化路径 HcclDispatcherAicpuInit)

创建工厂在 dispatcher_common.cc:HcclDispatcherInit():
```cpp
// dispatcher_common.cc:50-87
if (type == DispatcherType::DISPATCHER_NORMAL) {
    if (GetExternalInputHcclEnableFfts()) {
        pDispatcher = new (std::nothrow) DispatcherGraph(deviceLogicId);  // #ifndef HCCD
    } else {
        pDispatcher = new (std::nothrow) DispatcherPub(deviceLogicId);
    }
} else if (type == DispatcherType::DISPATCHER_VIRTURAL) {
    pDispatcher = new (std::nothrow) DispatcherVirtural(deviceLogicId);
}
```

关键设计决策:
1. C ABI 门面: dispatcher.h 用 extern "C" 包裹, HcclDispatcher 是 void* 句柄,
   dispatcher_common.cc 全部用 reinterpret_cast<DispatcherPub*>(dispatcherPtr) 转发,
   与 comm_primitive/ 的 DispatcherPub(thread_local context) 是两套不同的 Dispatcher 概念
2. 运行时决策 FFTS vs Runtime: GetExternalInputHcclEnableFfts() 环境变量决定
3. 设备侧条件编译: #ifndef HCCD 排除 host-only 代码(Graph模式、TBE reduce等)

### Idioms & Conventions

#### 1. C ABI 门面 + reinterpret_cast 转发模式

dispatcher_common.cc 中 26 个 C ABI 函数全部遵循同一模式:
```cpp
HcclResult HcclSomeOperation(HcclDispatcher dispatcherPtr, ...) {
    CHK_PTR_NULL(dispatcherPtr);
    return reinterpret_cast<DispatcherPub*>(dispatcherPtr)->SomeMethod(...);
}
```
出现 26 处, 覆盖 dispatcher_common.cc 全文。
特例: HcclSetOpExecStatusCallback 等 AICPU-specific 接口直接
reinterpret_cast<DispatcherAiCpu*> (dispatcher_common.cc:254), 无运行时类型检查。

#### 2. 4GB SDMA 切分惯用法

所有大数据传输必须按 4GB (HCCL_SDMA_MAX_COUNT_4GB = 0x100000000) 切分:
```cpp
// dispatcher.cc:389-421, dispatcher_aicpu.cc:377-413, dispatcher.cc:729-764
uint64_t spiltLoop = 0;
if (countSize > HCCL_SDMA_MAX_COUNT_4GB) {
    spiltLoop = (countSize % HCCL_SDMA_MAX_COUNT_4GB) ?
        (countSize / HCCL_SDMA_MAX_COUNT_4GB) : ((countSize / HCCL_SDMA_MAX_COUNT_4GB) - 1);
}
for (uint64_t index = 0; index <= spiltLoop; index++) {
    addrOffset = index * HCCL_SDMA_MAX_COUNT_4GB;
    contSplit = (index == spiltLoop) ? (countSize - index * HCCL_SDMA_MAX_COUNT_4GB) : ...;
    // ... 逐片下发
}
```
该模式在 task/ 模块出现 47 次 (含相关变量引用), 覆盖 3 个文件。
注意: LLT 构建时 HCCL_SDMA_MAX_COUNT_4GB = 200MB (dispatcher_pub.h:24)。

#### 3. Profiling 回调保护模式

每个任务下发方法都包含 callback_ 非空检查后的 profiling 回调:
```cpp
uint64_t beginTime = GetMsprofSysCycleTime();
// ... 执行实际操作 ...
if (callback_ != nullptr) {
    hccl::TaskPara taskPara(...);
    callback_(callBackUserPtr_, (void *)&taskPara, sizeof(struct TaskPara));
}
```
出现 34 处 callback_ 检查。DispatcherGraph 中更加冗余, 因为需要分别检查
IsProfSubscribeAdditionInfo() 和 GetExternalInputTaskExceptionSwitch() == 1
两种条件, 导致相同的回调代码重复两次 (例如 dispatcher_graph.cc:238-264)。

#### 4. PLF_CONFIG_INFO 详细日志

每个操作结束后都有 PLF_CONFIG_INFO(PLF_TASK, ...) 格式化日志:
```cpp
PLF_CONFIG_INFO(PLF_TASK,
    "%s para: dst[%p] destMax[%llu] src[%p] count[%llu] ...", __func__, ...);
```
出现 259 次日志调用, 覆盖 9 个文件。使用 __func__ 自动填充函数名。

#### 5. NotifyID 编码: rank<<32 | offset

全模块统一的 notifyID 编码方式:
```cpp
// 出现 8+ 处
u64 NotifyID = userRank;
NotifyID = (NotifyID << 32) | (offset & 0x00000000FFFFFFFF);
```
高 32 位存 userRank, 低 32 位存 offset, 用于 profiling 和 DFX 追踪。

### Architecture & Patterns

#### 1. 三种执行模式的关系

DispatcherPub (Runtime Stream模式):
- 直接调用 hrtMemAsyncCopy / hrtNotifyRecord / hrtReduceAsync 等 Runtime API
- 每个操作立即提交到硬件 stream
- 最简单的路径, 但不支持图缓存和优化

DispatcherGraph (FFTS 图模式):
- 先通过 GraphAdd*Task() 系列函数将操作积累到 FFTS context (fftsCtxsPtr)
- LaunchTasksEx() 时一次性通过 LaunchGraph() 提交整个图
- 支持图缓存 (enableCache + key): ResetGraphCtx → GetGraphCtx/GetGraphCtxV2
- 回退机制: disableFfts_ = true 时自动调用父类 DispatcherPub 的方法
- FFTS 超时特殊处理: 0→65535(永不超时), 65535→65534(避免被误认为永不超时)

DispatcherAiCpu (AICPU下沉模式):
- 设备侧执行, 通过 SQE(Submission Queue Entry) ring buffer 与硬件交互
- 每个操作编排为一个 SQE, 攒满 128 个(HCCL_PER_LAUNCH_SQE_CNT)后批量下发
- SQE 格式 v1/v2 由设备类型决定 (310P1/P3 用 v2, 其余用 v1)
- 支持算子展开动态缓存 (OpUnfoldCache): 首次执行缓存SQE, 后续命中时直接更新地址并下发
- LaunchTask() 内含自旋等待 RTSQ 消费空间的逻辑, 带超时和异常检查

DispatcherVirtural (虚拟模式):
- 不执行任何硬件操作, 仅通过 stream.PushTaskLogicInfo() 记录逻辑任务
- 用于算法层的 dry-run 或资源计算

#### 2. AICPU SQE 版本化策略

```cpp
// dispatcher_aicpu.cc:75-105
if (devType == DevType::DEV_TYPE_310P1 || devType == DevType::DEV_TYPE_310P3) {
    addOneNotifyWaitSqe_ = AddOneNotifyWaitSqeV2;  // 13个函数指针用V2版本
    ...
} else {
    addOneNotifyWaitSqe_ = AddOneNotifyWaitSqeV1;  // 13个函数指针用V1版本
    addOneRdmaDbSendSqe_ = AddOneRdmaDbSendSqeV1;  // V1独有
    addOneCacheMemcpyPlaceHolderSqe_ = ...;          // V1独有(缓存placeholder)
    ...
}
```
通过函数指针数组实现SQE格式的运行时多态, 避免每个方法内部 if-else。
V1 有 13 个函数指针 (含缓存 placeholder), V2 只有 7 个基础函数指针。

#### 3. AICPU 算子展开缓存 (OpUnfoldCache)

DispatcherAiCpu 内建 SQE 级缓存:
```
首次执行: SetLaunchContext → 各操作编排SQE → LaunchTask(缓存SQE) → ClearLaunchContext
缓存命中: LaunchNewTask(entryPtr) → UpdateAndGetSqeArray → WaitRtsq → MemcpyRtsq
```
缓存 key = OpUnfoldKey, 缓存内容 = SQE数组 + 类型 + DfxInfo。
刷新时只更新地址相关字段, 避免重新编排。
特殊处理: alltoallv 算子的 placeholder 机制 (isPlaceholder_ + addOneCachePlaceHolderSqe_)。

#### 4. HostNIC TCP 单线程发送模型

```
主线程:  HostNicTcpSend → hrtCallbackLaunch(StartHostNicTcpSendThread) → 回调设置 hostNicTcpSendThreadParam_
发送线程: HostNicTcpSendThreadTask() { while(state) { poll param; HostNicCallbackTcpSend(param); param=nullptr; } }
等待完成: HostNicTcpWaitSendCompletion → hrtCallbackLaunch(WaitHostNicTcpSendDone) → 自旋等待 param==nullptr
```
单线程序列化发送, 通过 hostNicTcpSendThreadParam_ (unique_ptr) 传参。
如果上一个发送未完成就设置新参数, 只打 HCCL_ERROR 但不返回错误 (dispatcher.cc:1076)。

### Hardware Abstractions

1. SDMA 4GB 限制: 硬件 SDMA 引擎单次传输上限 4GB, 软件层统一切分
2. Notify 大小: 910B/910_93 为 4 字节, 其他芯片为 8 字节 (dispatcher_aicpu.cc:116-117)
3. FFTS 超时: 硬件用 u16 表示超时(秒), 65535 被 RTS 解释为永不超时
4. SQE 格式: FftsSdmaSqeHeader 位域定义反映硬件寄存器布局
   (opcode:4, datatype:4, ie2:1, sssv:1, dssv:1, sns:1, dns:1, qos:4, ...)
5. 设备类型分支: 910B/910_93/310P1/310P3 各有不同的 notify/SQE/超时行为

### Anti-Patterns & Issues

#### 1. 全局可变状态

```cpp
// dispatcher_common.cc:23-24
FftsCounterCallBack g_InitTaskCallback = nullptr;
FftsCounterCallBack g_LaunchTaskCallback = nullptr;

// dispatcher.cc:28
HcclResult g_callBackResult = HCCL_SUCCESS;
```
三个全局变量在多进程/多设备场景下可能产生竞争。
g_callBackResult 在 HostNIC 回调中被赋值 (dispatcher.cc:817), 无锁保护。

#### 2. HostNIC TCP 发送线程的 busy-wait

```cpp
// dispatcher.cc:1087-1098
while (hostNicTcpSendThreadState_) {
    if (hostNicTcpSendThreadParam_ == nullptr) {
        SaluSleep(TCP_SEND_THREAD_SLEEP_TWO_HUNDRED_MICROSECOND);  // 200us 轮询
    } else { ... }
}
```
无 condition_variable, 空闲时 200us 间隔轮询, CPU 浪费。
WaitHostNicTcpSendTaskDone() 也是类似 busy-wait (dispatcher.cc:179-182)。

#### 3. HcclDispatcherDestroy 的 dispatcherPtr 参数问题

```cpp
// dispatcher_common.cc:89-97
HcclResult HcclDispatcherDestroy(HcclDispatcher dispatcherPtr) {
    if (dispatcherPtr != nullptr) {
        DispatcherPub* dispatcher = reinterpret_cast<DispatcherPub*>(dispatcherPtr);
        delete dispatcher;
        dispatcherPtr = nullptr;  // 无效: 这只清了局部副本
    }
    return HCCL_SUCCESS;
}
```
`dispatcherPtr = nullptr` 清除的是按值传入的局部变量, 调用方的指针不受影响。

#### 4. DispatcherGraph profiling 回调重复代码

每个 DispatcherGraph 方法中, profiling 和 exception 两种模式的回调代码几乎相同,
只有 ProfilerType 不同, 但整段代码被完整复制。
例如 dispatcher_graph.cc:238-264 与 251-264, MemcpyAsync/ReduceAsync/InlineReduceAsync
等 6 个方法均存在此问题。应提取为私有方法消除重复。

#### 5. reinterpret_cast 无类型安全

dispatcher_common.cc 中 AICPU-specific 接口直接用
reinterpret_cast<DispatcherAiCpu*>(dispatcherPtr) (共 4 处, :254/:262/:270/:280),
如果传入非 AICPU 类型的 dispatcher 句柄会导致未定义行为。
虽有 CheckRunSideIsDevice() 前置检查, 但只验证了运行环境而非对象类型。

### Notable Patterns

#### 1. T_DESC 宏的代码组织

```cpp
#if T_DESC("DispatcherPub", true)
// ... 整个类的实现 ...
#endif
```
dispatcher.cc 和 dispatcher_pub.h 使用 T_DESC 宏将代码分段, 值始终为 true,
仅用于 IDE 折叠和代码组织, 不影响编译。

#### 2. Callback 机制的两层设计

外部注册: RegisterLoadTaskCallBack(dispatcherPtr, userPtr, callback) — 外部 profiling 系统注册
内部全局: RegisterInitTaskCallBack / RegisterLaunchTaskCallBack — 全局 Init/Launch 钩子

两层独立, 外部 callback_ 是实例级的 (每个 dispatcher 对象), 全局钩子是进程级的。

#### 3. Stream 作为 SQE 载体

AICPU 模式下 Stream 对象不只是执行上下文, 还是 SQE ring buffer 的物理载体:
```cpp
stream.GetNextSqeBufferAddr(sqeBuffer, sqeTypeAddr, sqeDfxInfoAddr, taskId);
stream.GetSqeContextPtr() → SqeRingBuffer {localBuff, sqeType, dfxInfo, sqHead, sqTail, sqeCnt}
```
一个 Stream 对应一个 SQ (Submission Queue), 多 Stream 多 SQ 并行下发。

---

## 2.1.5 debug/ + ping_mesh/ — 维测功能、网络探测

### Module Overview

debug/ (1文件) 和 ping_mesh/ (3文件) 是 platform 层的辅助子模块:
- debug/traceinfo/: 运行时 trace 日志系统, 对接 atrace 基础设施
- ping_mesh/: 网络连通性探测工具 (RDMA/UB ping), 对外暴露 HCCN C API

两者均为功能独立的工具模块, 不在集合通信数据路径上。

### Key Files

```
debug/traceinfo/hccl_trace_info.cc — HcclTraceInfo 类实现(trace写入/分段提交/flush)
ping_mesh/ping_mesh.h              — PingMesh 类定义(RDMA/UB rping 管理)
ping_mesh/ping_mesh.cc             — PingMesh 实现(~800行, init/addTarget/batchPing/getResult)
ping_mesh/hccn_rping.cc            — HCCN C API 入口(HccnRpingInit/Deinit/AddTarget等)
```

### Idioms & Conventions

#### 1. Trace 分段提交

```cpp
// hccl_trace_info.cc:95-105
u32 len = TRACE_MAX_MSG_SIZE;  // 111 字节/段
while (pos < totLen) {
    submitLen = (pos + len <= totLen) ? len : (totLen - pos);
    CHK_RET(hrtTraceSubmit(handle, startPos, submitLen));
    startPos += submitLen;
    pos += submitLen;
}
```
atrace 单条日志上限 111 字节 (TRACE_MAX_MSG_SIZE), 长消息自动分段提交。

#### 2. HCCN C API 模式

ping_mesh/ 对外暴露 extern "C" 接口, 内部用 PingMesh 类管理状态:
```cpp
HccnRpingCtx = void*
HccnRpingInit(..., HccnRpingCtx *rpingCtx) {
    PingMesh *rping = new (std::nothrow) PingMesh();
    ...
    *rpingCtx = rping;
}
HccnRpingDeinit(HccnRpingCtx rpingCtx) {
    PingMesh *rping = static_cast<PingMesh*>(rpingCtx);
    ...
    delete rping;
}
```
与 dispatcher 的 C ABI 门面模式一致: void* 句柄 + static_cast 转换。

#### 3. 设备 ID 一致性校验

ping_mesh/ 每个 API 入口都进行设备 ID 一致性检查:
```cpp
s32 devLogicId = 0;
CHK_RET(hrtGetDeviceRefresh(&devLogicId));
if (devLogicId != rping->GetDeviceLogicId()) {
    HCCL_ERROR("curr devId[%d] don't match record logicId[%d].", ...);
    return HCCN_E_PARA;
}
```
出现 8+ 次, 防止跨设备误用同一句柄。

### Hardware Abstractions

ping_mesh/ 支持两种网络模式:
- RoCE (LINK_TYPE_MODE_ROCE = 3): IPv4/IPv6 地址, 传统 RDMA 网络
- UB/UDMA (LINK_TYPE_MODE_UB = 7): EID 地址, 950/910_96 芯片新通信协议

通过 IsSupportHCCLV2(socNamePtr) 判断芯片是否支持 UB 模式。
RpingPayloadHead / RpingIpHead / RpingEidHead 分别定义 RoCE 和 UB 的报文头。

### Anti-Patterns

#### 1. hccn_rping.cc 的 new/delete 手动内存管理

```cpp
// hccn_rping.cc:270-294
RpingInput *input = new (std::nothrow) RpingInput[targetNum];
...
if (res != HCCN_SUCCESS) { delete[] input; return ...; }
...
delete[] input;
```
多处 new[]/delete[] 手动配对, 错误路径容易泄漏。
例如 HccnRpingGetResult():540-551 中 output 分配后如果后续失败需要同时清理 input 和 output,
代码正确但脆弱。应使用 unique_ptr<T[]> 或 vector。

#### 2. Trace handle 初始化失败的静默处理

```cpp
// hccl_trace_info.cc:45-48
if (hrtTraceCreateWithAttr(logInfo.c_str(), handle) != HCCL_SUCCESS) {
    HCCL_RUN_INFO("[HcclTraceInfo]atrace create handle failed, Save logs to run info");
    handle = -1;  // 后续 SaveTraceInfo 会走 WARNING 降级路径
}
```
handle 创建失败只打 info 日志, 不返回错误, 后续操作静默降级。
设计意图是 trace 不应阻塞主逻辑, 但可能导致关键 trace 信息丢失。

---

## 2.1.6 Platform 层综合分析

本节综合 2.1.1-2.1.5 五个子模块的分析结果，提取跨子模块的惯用法、架构模式、
业务知识、硬件抽象和反模式。每条结论标注覆盖范围、强制程度和量化证据。

### 一、Platform 层全景

Platform 层是 hcomm 的底座，由 5 个功能域组成:

```
+-------------------------------------------------------------------+
| Platform Layer (~400 C++ files + ~237 C files)                    |
|                                                                   |
| comm_primitive/   resource/        hccp/         task/   debug/   |
| (数据面原语)       (资源管理)       (协议栈)      (任务下发) (维测) |
|  7 files           ~110 files       237 files     17 files  4 files|
|  C ABI facade      C++ OOP          纯C/内核风格   Strategy  工具  |
|                                                                   |
| 关键依赖链:                                                       |
| algorithm -> comm_primitive -> task(Dispatcher) -> hardware       |
|                             -> resource(Transport) -> hccp -> HDC |
+-------------------------------------------------------------------+
```

两大编码风格分区:
- C++区 (comm_primitive + resource + task + debug + ping_mesh): 继承/虚函数/RAII/CHK_RET
- C区 (hccp): 函数指针vtable/全局数组/calloc-free/CHK_PRT_RETURN/securec

### 二、跨子模块惯用法 (Idioms)

#### I1. 错误检查宏的100%覆盖

[适用范围]: Platform层强制
[强制程度]: 强制

C++区: 每个函数入口对所有指针参数 CHK_PTR_NULL，每个 HcclResult 返回值 CHK_RET。
C区(hccp): 对应使用 CHK_PRT_RETURN(条件, 日志语句, 返回值)。

```cpp
// C++ 区 (good)
HcclResult SomeFunc(void* ptr, Stream* stream) {
    CHK_PTR_NULL(ptr);
    CHK_PTR_NULL(stream);
    CHK_RET(stream->DoSomething());
    return HCCL_SUCCESS;
}

// C 区 (good)
int SomeFunc(unsigned int phyId, void *data) {
    CHK_PRT_RETURN(data == NULL, hccp_err("data is NULL"), -EINVAL);
    CHK_PRT_RETURN(phyId >= RA_MAX_PHY_ID_NUM, hccp_err("phyId overflow"), -EINVAL);
    return 0;
}

// bad — 遗漏指针检查
HcclResult SomeFunc(void* ptr) {
    return reinterpret_cast<Stream*>(ptr)->DoSomething();  // ptr 可能为空
}
```

量化: CHK_RET/CHK_PTR_NULL 覆盖 93/73 个 C++ 文件; CHK_PRT_RETURN 1546 次覆盖 61 个 hccp 文件。
出处: 整个 platform 层

#### I2. C ABI 门面 + void* 句柄 + cast 转发

[适用范围]: Platform层强制 (所有对外接口)
[强制程度]: 强制

所有 Platform 层对上层暴露的接口均使用 C ABI (extern "C") + void* 句柄模式:

```cpp
// good — C ABI 门面统一模式
// 头文件 (extern "C")
extern HcclResult HcclSomeOp(HcclDispatcher dispatcherPtr, ...);

// 实现文件
HcclResult HcclSomeOp(HcclDispatcher dispatcherPtr, ...) {
    CHK_PTR_NULL(dispatcherPtr);
    return reinterpret_cast<DispatcherPub*>(dispatcherPtr)->SomeMethod(...);
}

// bad — 在 C ABI 边界暴露 C++ 类型
extern HcclResult HcclSomeOp(DispatcherPub* dispatcher, ...);  // C 不识别 DispatcherPub
```

出现位置:
- comm_primitive: StreamHandle/HcclMemTransport/DispatcherCtxPtr (35 次 reinterpret_cast)
- task: HcclDispatcher (26 个 C ABI 门面函数)
- ping_mesh: HccnRpingCtx (void* → PingMesh*)
- hccp: 公共 API 全部用 HCCP_ATTRI_VISI_DEF 标记可见性

量化: extern "C" 覆盖 58 个文件 (platform 43 + pub_inc 15); reinterpret_cast 覆盖 71 个 C++ 文件。

#### I3. owner_ + move 语义的轻量所有权管理

[适用范围]: Platform层强制 (值语义资源类)
[强制程度]: 强制

用布尔标志 owner_ 替代 shared_ptr 管理资源所有权:

```cpp
// good — owner_ 模式
class DeviceMem {
    void* ptr_;
    u64 size_;
    bool owner_;               // 是否拥有底层资源
public:
    static DeviceMem alloc(u64 size);    // owner=true, 分配新资源
    static DeviceMem create(void* p, u64 s); // owner=false, 包装已有指针
    DeviceMem range(u64 off, u64 s);     // owner=false, 切割子块
    DeviceMem(DeviceMem&& that) noexcept; // 转移所有权
    ~DeviceMem() { if (owner_ && ptr_) hrtFree(ptr_); }
};

// bad — 对轻量资源句柄使用 shared_ptr
std::shared_ptr<void> ptr(hrtMalloc(size), hrtFree);  // 引用计数开销不必要
```

出现于: DeviceMem, HostMem, Stream, LocalNotifyImpl(notifyOwner_) — 共 4 个核心资源类。
设计理由: 轻量(仅 3 字段)、与 C API 兼容、alloc/create 的显式语义比构造函数更安全。
出处: mem_device.cc, mem_host.cc, stream.cc, local_notify_impl.cc

#### I4. new(std::nothrow) + CHK_PTR_NULL 的堆分配

[适用范围]: Platform层强制
[强制程度]: 强制

禁用 new 抛异常，改用 nothrow + 空指针检查:

```cpp
// good
auto* obj = new (std::nothrow) SomeClass();
CHK_PTR_NULL(obj);

// bad — 裸 new (可能抛异常)
auto* obj = new SomeClass();

// bad — make_unique (隐含异常)
auto obj = std::make_unique<SomeClass>();
```

量化: 覆盖 27 个文件。
设计理由: 与 C ABI 边界的无异常设计保持一致; Init/Destroy 生命周期模式不适合 RAII 智能指针。
出处: dispatcher_common.cc, hccl_dispatcher_ctx.cc, hccl_network.cc 等

#### I5. Pimpl 模式隐藏实现

[适用范围]: Platform层强制 (所有对外暴露的资源类)
[强制程度]: 强制

公开类在 pub_inc/，Impl 类在 src/platform/，头文件隔离实现细节:

```cpp
// pub_inc/inner/transport.h (对外)
class Transport {
    TransportBase* pimpl_;
public:
    HcclResult WriteAsync(...);
};

// src/platform/resource/transport/transport.cc (实现)
HcclResult Transport::WriteAsync(...) { return pimpl_->WriteAsync(...); }
```

出现于: Transport, LocalIpcNotify, RemoteNotify, NotifyPool, 所有 RmaBuffer (7+ Impl 文件)。
出处: pub_inc/inner/ → src/platform/resource/

#### I6. 引用计数去重 (延迟释放共享底层资源)

[适用范围]: Platform层强制 (资源共享场景)
[强制程度]: 强制

```cpp
// good — Ref/Unref + 归零释放
if (IsDevAddrExistInDevAddrIdentifierMap(logicId, addrID)) {
    DecrementRefCount(logicId, addrID);
} else {
    hrtRaDeRegGlobalMr(rdmaHandle, mrHandle);  // 计数归零才释放
}
```

出现于 5 处独立实现:
1. MemMappingManager — Host→Device 地址映射缓存 (Referenced 对象)
2. MemNameRepository — IPC 内存名注册去重 (setNameMapRef_/openedNameMapRef_)
3. g_devAddrIdentifierMap — RDMA MR 注册去重 (全局 map + 计数)
4. NotifyPool — Notify 资源复用 (registeredOpMap)
5. hccp RefInstances — 多进程共享设备初始化 (gRefInstances[phyId].refCount)

#### I7. 工厂选择 (运行时条件分派)

[适用范围]: Platform层强制 (所有多实现场景)
[强制程度]: 强制

每个需要多实现的抽象都通过运行时条件选择具体类型:

| 抽象 | 选择依据 | 实现数 |
|------|----------|--------|
| Transport | TransportType 枚举 | 9 种 |
| Notify | local/remote deviceId 组合 | 3 种 |
| Dispatcher | DevType + 平台 + FFTS 开关 | 4 种 |
| NetDev | DEV_TYPE_910_95 → v2, 其他 → v1 | 2 版本 |
| NetDevContext | protoType + deployment | 4 种 handle |
| hccp 通信路径 | NetworkMode | HDC / Peer 2 条 |
| AICPU SQE | DevType (310P1/P3 → v2) | v1/v2 函数指针 |

模式: switch/if-else + new(std::nothrow)，不使用注册表或反射。

#### I8. 日志级别纪律

[适用范围]: Platform层推荐
[强制程度]: 推荐

```
HCCL_ERROR   — 不可恢复错误，操作必定失败
HCCL_WARNING — 可恢复异常，静默降级或重试
HCCL_INFO    — 管理操作入口、生命周期事件、低频路径
HCCL_DEBUG   — 数据面高频操作 (远端原语入口、逐条 SQE 等)
```

特殊惯用法:
- 远端原语用 DEBUG 避免日志风暴 (comm_primitive)
- 50 倍采样打印: 每 50 次只打 1 次 (socket)
- PLF_CONFIG_INFO(PLF_TASK, ...): 任务下发专用格式化日志 (task, 259 次)
- hccp 用独立日志后端: hccp_info/err/warn/debug (3330 次, 64 文件)

量化: HCCL 日志宏覆盖 115 个 C++ 文件。

#### I9. LIKELY/UNLIKELY 分支预测提示

[适用范围]: Platform层推荐 (热路径)
[强制程度]: 推荐

```cpp
if (UNLIKELY(ptr == nullptr)) { return HCCL_E_PTR; }     // 异常路径
if (LIKELY(cache.Find(key))) { return cached_result; }    // 快速路径
```

量化: 覆盖 21 个文件，集中在 transport、dispatcher_ctx、comm_primitive 的热函数。

#### I10. securec 安全函数 (hccp C 区)

[适用范围]: hccp 模块强制
[强制程度]: 强制

```c
// good — securec + 返回值检查
ret = memcpy_s(dst, dstSize, src, srcSize);
if (ret != EOK) { hccp_err("memcpy_s failed"); return -ESAFEFUNC; }

// bad — 标准 C 库函数
memcpy(dst, src, size);  // 无边界检查
```

量化: 463 次，覆盖 49 个 hccp 文件。自定义 errno: ESAFEFUNC=256。

#### I11. 4GB SDMA 切分

[适用范围]: task 模块强制
[强制程度]: 强制

所有大数据传输必须按 HCCL_SDMA_MAX_COUNT_4GB (0x100000000) 切分后逐片下发:

```cpp
if (countSize > HCCL_SDMA_MAX_COUNT_4GB) {
    spiltLoop = (countSize % HCCL_SDMA_MAX_COUNT_4GB) ?
        (countSize / HCCL_SDMA_MAX_COUNT_4GB) : (countSize / HCCL_SDMA_MAX_COUNT_4GB - 1);
}
for (uint64_t i = 0; i <= spiltLoop; i++) { /* 逐片下发 */ }
```

量化: task 模块出现 47 次。LLT 构建时降低为 200MB 便于测试。

### 三、架构模式 (Architecture Patterns)

#### A1. C/C++ 双风格分区

Platform 层存在明显的 C/C++ 风格分界线:

| 维度 | C++ 区 (comm_primitive/resource/task) | C 区 (hccp) |
|------|--------------------------------------|-------------|
| 错误处理 | CHK_RET/CHK_PTR_NULL (HcclResult) | CHK_PRT_RETURN (int + errno) |
| 多态 | 虚函数 + 继承 (TransportBase 70+ virtual) | 函数指针 vtable (HdcOps/RsOps/RsCtxOps) |
| 内存管理 | new(std::nothrow)/delete, DeviceMem RAII | calloc/free, 手动生命周期 |
| 状态管理 | 对象实例 + Pimpl | 全局数组 gXxx[phyId] |
| 同步 | std::mutex, thread_local | pthread_mutex, __thread |
| 测试适配 | 虚函数 + mock | STATIC 宏暴露 static 函数 |
| 字符串 | std::string | securec (memcpy_s 等) |
| 链表 | std::vector/list | RaListHead (内核风格侵入式) |
| 日志 | HCCL_INFO/DEBUG/WARNING/ERROR | hccp_info/err/warn/debug (三路编译选择) |

分界原因: hccp 更接近内核/驱动风格，强调可预测性和确定性 (无异常、无隐式构造/析构)。

#### A2. 资源生命周期统一模式

所有 Platform 层资源类遵循: 分配 → 初始化 → 使用 → 销毁 → 释放

```
                alloc/new          Init()         使用          Destroy/DeInit    delete/~dtor
                   |                 |              |                |                |
DeviceMem:    alloc(size)           -           ptr()/range()        -            ~DeviceMem()
Transport:    new Transport     Init(links)     WriteAsync()     DeInit()       ~Transport()
Notify:       CreateNotify      Init()          Post/Wait        Destroy()      ~NotifyImpl()
RmaBuffer:    new RmaBuffer     Init()          Serialize()      Destroy()      ~Impl()
Stream:       StreamType ctor   InitStream()    GetSqeBuffer     DestroyStream  ~Stream()
Dispatcher:   new Dispatcher    Init(devLogId)  MemcpyAsync()    Destroy()      delete
hccp设备:     RaInit(phyId)     RaHdcInit()     RaHdcProcessMsg  RaHdcDeinit    RaDeinit(phyId)
```

关键约束: Init() 可以失败并返回错误码; 构造函数不做复杂初始化。
这与"两阶段初始化"模式一致，避免构造函数抛异常。

#### A3. Transport 继承体系 (Platform 层最大的类层次)

```
TransportBase (70+ virtual methods, transport_base_pub.h)
├── TransportP2p (同节点 IPC/HCCS/PCIe)
│   └── TransportDeviceP2p (设备侧 P2P, SDMA 信号记录)
├── TransportNet
│   ├── TransportTcp (跨节点 TCP, 事件驱动接收+线程池发送)
│   ├── TransportIbverbs (InfiniBand RDMA, 单/多 QP, WQE 批量)
│   │   └── TransportDeviceIbverbs (设备侧 IBVerbs, HCOMM_DSB 内存屏障)
│   └── TransportDirectNpu (直连 RDMA + AICPU 动态内核)
├── TransportRoce (RoCE, 3 条独立链路, 异构双继承)
├── TransportShmEvent (共享内存 + 事件驱动, Worker/PS 架构)
└── TransportVirtural (仅记录 TaskLogicInfo, dry-run)

Heterog 独立链:
├── TransportHeterogP2p (4 阶段连接状态机)
├── TransportHeterogRoce (3 条独立 Socket)
└── TransportHeterogEventRoce/Tcp (全局 CQE 计数)
```

设计特点:
- TransportBase 定义完整接口 (70+ 虚方法)，子类按需覆写
- Transport (Facade) 持有 TransportBase*，根据 TransportType 工厂创建
- 5 类通知信号: sendReady/sendDone (local+remote) + device 侧 + user 自定义

#### A4. 三种执行后端 (Strategy Pattern)

```
                 DispatcherPub (base, Runtime Stream)
                 /            |              \
    DispatcherGraph     DispatcherAiCpu    DispatcherVirtural
    (FFTS 图缓存)      (AICPU SQE 下沉)    (逻辑记录)
```

| 后端 | 执行方式 | 优势 | 适用场景 |
|------|----------|------|----------|
| DispatcherPub | 逐条提交到 runtime stream | 简单直接 | 通用 host 侧 |
| DispatcherGraph | 积累到 FFTS context，一次性提交图 | 图缓存复用 | 910B + FFTS |
| DispatcherAiCpu | SQE ring buffer 批量下发 (128/批) | 设备侧执行 | AICPU 下沉 |
| DispatcherVirtural | 仅记录 TaskLogicInfo | 无硬件开销 | dry-run / 资源计算 |

选择逻辑: DevType + 平台(host/device) + FFTS 开关 → 运行时决策。

#### A5. hccp Client-Server + Opcode RPC

```
Host (rdma_agent)              Device (rdma_service)
  RaHdcProcessMsg() ──────────> RaHandle() ──> gRaOpHandle[118]
  MsgHead(28B) + OpXxxData       线性查找 opcode → 函数指针分发
```

- 110+ 外部 opcode + 内部 opcode (1000+)
- 每个 opcode: union OpXxxData { txData; rxData; } 共享内存
- 版本协商: 初始化时批量查询所有 opcode 版本，支持向后兼容
- Token bucket 限流: BUCKET_DEPTH=2048, TOKEN_RATE=400/ms
- 同步 (mutex + 阻塞) / 异步 (reqId + isDone 轮询) 双通道
- HDC / Peer 双路径: NetworkMode 决定走驱动通道还是直连

#### A6. 三代实现并存

comm_primitive 模块展示了代码演进的三代共存:
1. Platform 新接口 (src/platform/): C ABI 平坦函数，CHK_RET 返回码
2. Legacy 类体系 (src/legacy/): OOP Primitive 继承链 + PrimTranslator 编译流水线 + THROW 异常
3. Framework/next 适配桩 (src/framework/next/): 空实现，未来方向

教训: 新代码应采用 Platform 新接口风格 (C ABI + 返回码)，不使用 Legacy 的异常模式。

### 四、业务领域知识 (Domain Knowledge)

#### D1. 通信原语分类

```
本地原语 (同芯片, LINK_ONCHIP):
  LocalCopy         — SDMA 内存拷贝
  LocalCopyReduce   — 拷贝 + InlineReduce
  LocalNotify       — 硬件通知 (Record/Wait)
  BareNotify        — 轻量通知 (notifyId, 无 aclrt 对象)

远端原语 (跨芯片):
  RemoteWrite/Read        — 单边 RDMA 写/读
  RemoteWriteReduce       — RDMA 写 + 原子归约
  RemoteReadReduce        — RDMA 读 + 原子归约 (仅 P2P, 不支持 RoCE)
  RemoteNotify            — 跨芯片同步
  RemoteFence             — UB 内存访问顺序屏障
  RemoteBatch*            — 批量操作
```

#### D2. DMA 传输模式策略

```
DmaMode::DEFAULT 自动选择:
  P2P (HCCS) 链路 → GET (读模式, 接收方主动拉)
  DEV_NET (RoCE) 链路 → PUT (写模式, 发送方主动推)

原因: P2P/HCCS 支持高效远端读; RoCE 网络中远端读需要额外往返。
```

#### D3. Notify 三轮握手同步协议

```
发送侧                    接收侧
  PostReady ──────────> WaitReady    "我准备好了"
  Write/Read  ~~~~~~~~  (数据传输)
  PostFin   ──────────> WaitFin      "传输完成"
  (WaitFinAck) <────── (PostFinAck)   DEV_NET 可选额外确认
```

WriteWithNotify 优化: 将 Write + PostFin 合并为一个硬件操作 (需设备支持)。

#### D4. InlineReduce 降级策略

```
是否支持 InlineReduce?
  ← 检查: 数据类型 + 归约操作 + 链路类型 (DevCapability)
  支持: 直接硬件归约
  不支持:
    SendReduce → 先 Write 到 remoteSrc，再远端 LocalReduce
    RecvReduce → 先 Read 到 localSrc，再本地 LocalReduce
```

#### D5. 双协议栈: RDMA vs UB(UDMA)

```
产品型号        协议          QP 概念
910/910B/910_93  RDMA (RoCE)   标准 RDMA QP
950/910_96       UB (UDMA)     Jetty (5 种模式: NORMAL/DWQE/CCU/USER_CTL/CCU_TA_CACHE)
```

hccp_ctx.h 通过 union 实现协议抽象: 同一 struct 的 rdma/ub 分支。

### 五、硬件抽象 (Hardware Abstractions)

#### H1. 通信链路分层

```
同节点:
  HCCS (High-speed Chip-to-Chip)    ← TransportP2p, DEFAULT→GET
  HCCS_SW (HCCS via Switch)         ← A3 DIE 间
  SIO (Serial I/O)                  ← 低速备选
  PCIe                              ← Host-Device, MemMappingManager

跨节点:
  RoCE (RDMA over Converged Ethernet) ← TransportIbverbs/RoCE, DEFAULT→PUT
  TCP                                  ← TransportTcp

异构:
  共享内存 + 事件驱动              ← TransportShmEvent
  IPC (进程间通信)                 ← P2P 同进程模式
```

#### H2. 设备类型对行为的影响

| 设备 | 影响的行为 |
|------|-----------|
| 910B/910_93 | PCIe TH 参数、FFTS 模式、4 字节 Notify、多 QP 扩展 |
| 910_95 | NetDev v2 接口 |
| 950/910_96 | UB(UDMA) 协议、Jetty 模式 |
| 310P1/310P3 | AICPU SQE v2 格式 |
| HOST_DEVICE_ID | BareNotify、EschedNotify |

#### H3. 内存层次与屏障

```
Device Memory (hrtMalloc/hrtFree):
  DeviceMem 管理, 4K 页对齐, 需 VA 映射才能从 Host 访问
  910B: HOST_MEM_MAP_DEV_PCIE_TH / 其他: HOST_MEM_MAP_DEV

Host Memory:
  RTS 分配 (hrtMallocHost) — DMA-safe, isRtsMem_=true
  标准 new — 普通内存, isRtsMem_=false

RDMA Memory Region:
  hrtRaRegGlobalMr → lkey(本地) / rkey(远端)
  g_devAddrIdentifierMap 引用计数防重复注销

内存屏障:
  x86:     asm volatile("" ::: "memory")              // compiler fence
  aarch64: asm volatile("dsb st" ::: "memory")        // store barrier
  通用:    std::atomic_thread_fence(memory_order_seq_cst) // full fence
  高频计数: memory_order_relaxed                       // 无序足够
```

#### H4. 硬件限制常量

```
HCCL_SDMA_MAX_COUNT_4GB = 0x100000000   // SDMA 单次传输上限
MAX_RDMA_WQE_SIZE = 2GB                 // RDMA WQE 上限
HCCL_SQE_MAX_CNT = 2048                 // SQE ring buffer 容量
HCCL_PER_LAUNCH_SQE_CNT = 128           // AICPU 批量下发阈值
TRACE_MAX_MSG_SIZE = 111                // atrace 单条上限 (字节)
HCCL_SHM_ALIGN = 4096                  // 共享内存页对齐
BUCKET_DEPTH = 2048                     // hccp token bucket 容量
TOKEN_RATE = 400                        // hccp 每毫秒补充 token 数
```

### 六、反模式清单 (Anti-Patterns)

#### AP1. 全局可变状态 (4+ 文件, 8 处 static mutex)

问题: 全局 static map + mutex 导致测试隔离困难、并发风险、生命周期不可控。

实例:
| 位置 | 全局变量 | 风险 |
|------|----------|------|
| transport.cc:23 | transportMap_ (static unordered_map) | 多 Transport 析构竞争 |
| local_rdma_rma_buffer_impl.cc:34 | g_devAddrIdentifierMap | 测试无法隔离 |
| transport_heterog_event_roce.cc:49 | gQpnToTransportMap (普通 map) | 非线程安全 |
| dispatcher_common.cc:23 | g_InitTaskCallback (全局函数指针) | 多设备竞争 |
| dispatcher.cc:28 | g_callBackResult (全局变量) | 无锁保护 |
| hccp 全模块 | gRaRdevHandle[64], gRaHdc[64], gRsCbList[64] | 硬编码上限 |

正确做法: 将状态绑定到对象生命周期; 全局 map 使用 ConcurrentMap 或收窄锁粒度。

#### AP2. reinterpret_cast 无类型安全

问题: void* 句柄 + reinterpret_cast 无运行时类型检查，传入错误类型导致 UB。

实例:
- dispatcher_common.cc:254/262/270/280 直接 reinterpret_cast<DispatcherAiCpu*>
  (仅有 CheckRunSideIsDevice 检查运行环境，不检查对象类型)
- comm_primitive 所有函数 (35 处 reinterpret_cast)

正确做法: 在 debug 构建中加 type tag 校验; 或使用 static_cast + 基类指针。

#### AP3. HcclRemoteReadReduce 疑似 bug

```cpp
// bad — 用枚举值做除法 (hccl_primitive_remote.cc:87)
rmtBuf->len / reduceInfo.dataType           // HcclDataType 枚举值 ≠ 字节数

// good — 查表获取字节数 (hccl_primitive_local.cc:64)
src->len / SIZE_TABLE[reduceInfo.dataType]   // 正确查表
```

出处: hccl_primitive_remote.cc:87 vs hccl_primitive_local.cc:64

#### AP4. AcquireDispatcherCtx 泄漏风险

```cpp
// bad — 创建 ctx 只存 thread_local, 不注册到全局 map (hccl_dispatcher_ctx.cc:213-231)
*ctx = Ctx_tmp;
gDispatcherCtx = Ctx_tmp;  // 线程结束后无法通过 commId 找回
```

正确做法: 新创建的 ctx 同时注册到 g_ctx map。

#### AP5. Stream 移动赋值 bug

```cpp
// bad — 移动后 that 被赋值为 this 的内容 (stream.cc:155-174)
that.taskLogicInfo_ = taskLogicInfo_;  // 应该是 taskLogicInfo_ = std::move(that.taskLogicInfo_)
```

#### AP6. HcclDispatcherDestroy 局部变量置空

```cpp
// bad — 清空局部副本无效 (dispatcher_common.cc:89-97)
HcclResult HcclDispatcherDestroy(HcclDispatcher dispatcherPtr) {
    delete reinterpret_cast<DispatcherPub*>(dispatcherPtr);
    dispatcherPtr = nullptr;  // 无效: 按值传入的局部变量
}
```

正确做法: 参数改为 HcclDispatcher*，或返回后由调用方置空。

#### AP7. close() EINTR 重试 (hccp)

```c
// bad — Linux 下 close() 返回 EINTR 后 fd 已释放, 重试可能关闭其他线程的 fd
do { ret_ = close(fd); } while ((ret_ < 0) && (errno == EINTR));
```

正确做法(Linux): close() 不需要重试 EINTR，只需关闭一次。

#### AP8. HostNIC TCP busy-wait

```cpp
// bad — 200us 轮询无 condition_variable (dispatcher.cc:1087-1098)
while (state) {
    if (param == nullptr) { SaluSleep(200us); }  // CPU 空转
    else { /* 处理 */ }
}
```

正确做法: 用 std::condition_variable 替代 busy-wait。

#### AP9. 手动 new[]/delete[] 配对

```cpp
// bad — 错误路径容易泄漏 (hccn_rping.cc:270-294)
RpingInput *input = new (std::nothrow) RpingInput[targetNum];
if (res != HCCN_SUCCESS) { delete[] input; return ...; }
```

正确做法: 使用 std::unique_ptr<T[]> 或 std::vector。

#### AP10. CreateDispatcherCtx 绑定失败静默复用

```cpp
// bad — 返回 SUCCESS 但复用了别人的 ctx (hccl_dispatcher_ctx.cc:90-103)
if (BindFail) { reuse existing ctx; return HCCL_SUCCESS; }
```

调用方无法区分"新建成功"和"复用已有"，掩盖并发竞态。

#### AP11. hccp 双重线性查找

```c
// bad — 同一 opcode 在 RaCheckParam + RaHandle 中被线性查找两次
RaCheckParam() → for 循环查 dataSize
RaHandle()     → for 循环查 handler
```

正确做法: RaCheckParam 返回找到的 index，RaHandle 直接用 index 分发。

#### AP12. TCP 发送队列切换的三元运算符嵌套

```cpp
// bad — 晦涩 (tcp_send_thread_pool.cc:260-264)
ptr = &queues.first == ptr ? &queues.second : &queues.first;
```

正确做法: 拆分为 if-else，或用 bool 标志 toggle。

### 七、Platform 层特征总结

| 特征 | 描述 |
|------|------|
| 代码规模 | ~400 C++ 文件 + ~237 C 文件 |
| 错误处理 | 100% 返回码检查，零异常 (C++ 新代码) |
| 内存管理 | owner_+move 轻量 RAII; new(std::nothrow); 无 shared_ptr 依赖 |
| 多态机制 | C++区虚函数继承; C区函数指针 vtable |
| 接口设计 | C ABI + void* 句柄 + Pimpl 隐藏实现 |
| 资源生命周期 | 两阶段初始化 (构造 + Init)，引用计数延迟释放 |
| 硬件适配 | 运行时工厂选择，编译时条件编译 (#ifdef HCCD / DevType) |
| 并发模型 | std::mutex/pthread_mutex + thread_local/__thread; 无 lock-free 容器 |
| 代码卫生 | 0 个 TODO/FIXME/HACK (platform C++ 区); hccp 零 TODO |
| 主要风险 | 全局可变状态 (8处 static mutex)、reinterpret_cast 无类型检查 |
