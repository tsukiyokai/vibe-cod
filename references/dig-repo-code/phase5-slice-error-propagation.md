# Phase 5.3: Error Propagation (Cross-Repo Slice)

## Overview

追踪错误从hcomm platform层最底部到hccl API返回值的完整路径,
分析各层的错误码转换/日志记录/诊断机制,以及健康检查和故障恢复系统。

---

## 5.3.1 Error Propagation Path

### HcclResult错误码体系

所有层共享同一个错误码枚举(hccl_types.h:23-51):

```
HCCL_SUCCESS           = 0
HCCL_E_PARA            = 1    参数错误
HCCL_E_PTR             = 2    空指针
HCCL_E_MEMORY          = 3    内存错误
HCCL_E_INTERNAL        = 4    内部错误(不应到达)
HCCL_E_NOT_SUPPORT     = 5    不支持
HCCL_E_NOT_FOUND       = 6    未找到
HCCL_E_UNAVAIL         = 7    资源不可用
HCCL_E_TIMEOUT         = 9    超时
HCCL_E_TCP_CONNECT     = 11   TCP连接失败
HCCL_E_ROCE_CONNECT    = 12   RoCE连接失败
HCCL_E_ROCE_TRANSFER   = 14   RoCE传输错误
HCCL_E_RUNTIME         = 15   Runtime API失败
HCCL_E_NETWORK         = 19   网络错误
HCCL_E_AGAIN           = 20   重试(特殊: CHK_RET走WARNING而非ERROR)
HCCL_E_REMOTE          = 21   远端CQE错误
HCCL_E_SUSPENDING      = 22   正在挂起
```

### 核心传播机制: CHK_RET

```cpp
// pub_inc/log.h:188 (两仓定义一致)
#define CHK_RET(call) do {
    HcclResult hcclRet = call;
    if (UNLIKELY(hcclRet != HCCL_SUCCESS)) {
        if (hcclRet == HCCL_E_AGAIN) {
            HCCL_WARNING("[%s]call trace: hcclRet -> %d", __func__, hcclRet);
        } else {
            HCCL_ERROR("[%s]call trace: hcclRet -> %d", __func__, hcclRet);
        }
        return hcclRet;  // 原样透传,不做任何转换
    }
} while(0)
```

关键语义:
- HCCL_E_AGAIN走WARNING而非ERROR(重试语义特殊处理)
- 错误码原封不动向上传递,CHK_RET永远不改变错误码值
- 自动追加`[__func__]call trace`日志,形成调用栈追踪

量化: hccl CHK_RET 1448次, hcomm CHK_RET 14097次

### CHK_PRT_RET: 可定制的错误传播

```cpp
#define CHK_PRT_RET(result, exeLog, retCode) do {
    if (UNLIKELY(result)) {
        exeLog;         // 先打自定义日志
        return retCode; // 可返回与原错误不同的码
    }
} while(0)
```

这是错误码发生"转换"的唯一地方。hcomm中1131次使用。

### 错误产生的三类源头

1. 底层driver/runtime调用失败 -> CHK_PRT_RET赋新错误码
2. 参数校验失败 -> return HCCL_E_PARA/HCCL_E_PTR
3. 超时检测 -> return HCCL_E_TIMEOUT

### 完整传播路径(AllReduce为例, 9层)

```
层9 [hcomm/platform] driver/runtime API失败
  |  hrtRaQpCreate() 失败
  |  -> CHK_PRT_RET(..., HCCL_E_ROCE_CONNECT)     [第一次错误码产生]
  v
层8 [hcomm/platform] Transport层
  |  TransportIbverbs::Init() -> InitQpConnect() -> CreateQp() -> CreateSingleQp()
  |  每层 CHK_RET 透传,不转换
  v
层7 [hcomm/platform] comm_primitive C ABI层
  |  HcclRemoteWrite() / HcclLocalCopy() 等
  |  仅CHK_PTR_NULL参数校验,其余原样透传Transport返回值
  v
层6 [hcomm/platform] dispatcher层
  |  DispatcherPub::MemcpyAsync() / SignalRecord() / SignalWait()
  |  CHK_RET透传runtime API(hrt*)返回值
  v
层5 [hccl/template] wrapper层 -- 跨仓边界
  |  alg_data_trans_wrapper.cc
  |  CHK_RET(static_cast<HcclResult>(HcommWriteOnThread(...)))
  |  int32_t -> HcclResult 强制转换后透传
  v
层4 [hccl/template] KernelRun
  |  ins_temp_all_reduce_*.cc
  |  CHK_RET透传wrapper返回值
  v
层3 [hccl/executor] Orchestrate
  |  ins_v2_all_reduce_sole_executor.cc
  |  CHK_RET透传template返回值
  v
层2 [hccl/op_common] HcclExecOp
  |  op_common.cc: executor->Orchestrate(param, *resCtxHost)
  |  CHK_RET透传executor返回值
  v
层1 [hccl/op] AllReduceOutPlace
  |  all_reduce_op.cc: HcclExecOp(comm, param, topoInfo, algName)
  |  CHK_RET透传
  v
层0 [hccl/API] HcclAllReduce
  |  CHK_RET_AND_PRINT_IDE(AllReduceOutPlace(...), tag.c_str())
  |  返回给用户
```

规律: 9层中只有层9(driver调用点)产生新错误码,层8~层0全部CHK_RET透传。
用户拿到的HcclResult就是platform层最底部产生的那个值。

### hccl-hcomm边界的双返回类型体系

控制面接口(返回HcclResult, hccl_res.h/hccl_comm.h):
- HcclThreadAcquire, HcclChannelAcquire, HcclEngineCtxCreate等
- hccl侧直接`CHK_RET(HcclXxx(...))`无需转换

数据面接口(返回int32_t, hcomm_primitives.h):
- HcommWriteOnThread, HcommReadOnThread, HcommLocalCopyOnThread等22个
- hccl侧必须`CHK_RET(static_cast<HcclResult>(HcommXxx(...)))`

量化:
- `CHK_RET(static_cast<HcclResult>(Hcomm...))`: 165次/27文件
- `CHK_RET(HcclThread/Channel/Engine...)` 控制面直接调用: 36次/3文件
- `CHK_RET(HcclGetRankSize/Id/CommName)`: 71次/16文件

潜在风险: static_cast<HcclResult>是裸转换,依赖两边错误码数值一致。
如果hcomm数据面返回的int32_t值有一天与HcclResult枚举不对齐,
CHK_RET仍会触发但打印的错误码含义会错误。

### Transport层具体错误产生点

IBVerbs(RoCE):
- QP创建失败: hrtRaQpCreate()失败 -> HCCL_E_ROCE_CONNECT
- QP建链超时: 轮询超timeout_ -> HCCL_E_TIMEOUT
- 内存拷贝失败: memcpy_s!=EOK -> HCCL_E_MEMORY
- 非法wqeType: switch default -> HCCL_E_INTERNAL
- CQE运行时错误: 异步路径,通过GetTransportErrorCqe()填充CqeInfo vector,
  上层健康检查转为HCCL_E_REMOTE

TCP:
- NIC部署无效: -> HCCL_E_PARA
- sockets为空: -> HCCL_E_INTERNAL
- Send/Recv失败: 原样透传dispatcher返回值

P2P:
- socket收发失败: 原样透传socket层错误码
- 链路类型不支持: -> HCCL_E_NOT_SUPPORT

comm_primitive:
- 不产生新错误码,仅CHK_PTR_NULL(->HCCL_E_PTR)后透传Transport返回值
- 唯一例外: HcclRemoteReadReduce对ROCE链路返回HCCL_E_NOT_SUPPORT

dispatcher:
- Runtime API失败: CHK_RET透传hrt*返回值
- 参数校验失败: HCCL_E_PARA/HCCL_E_PTR

### CQE异步错误检测路径(非同步传播)

```
硬件RDMA CQE出错
  -> hrtRaGetCqeErrInfoList() 检测到错误CQE
  -> ProcessCqeInfo(): 通过 g_qpn2IbversLinkMap_[devicePhyId<<32|qpn]
     定位TransportIbverbs实例
  -> GetTransportErrorCqe() 填充 CqeInfo vector(时间戳/status/remoteIp)
  -> Heartbeat::OpRetryCQEHandle() 每次最多获取128个CQE
  -> 存入remoteIpMap+rankMapForRetryAgent
  -> OpRetry Agent通过GetQpnErr()查询
  -> 触发OpRetry重执行流程
```

---

## 5.3.2 Error Code Conversion & Logging Strategy

### 错误码转换汇总

整个系统中只有4个错误码转换/映射点:

1. CHK_PRT_RET(hcomm platform, 1131次)
   driver/runtime调用失败 -> 语义化HcclResult
   例: hrtRaQpCreate失败 -> HCCL_E_ROCE_CONNECT

2. CHK_SAFETY_FUNC_RET(hcomm framework, 261处)
   安全函数memcpy_s/memset_s的s32返回值 -> HCCL_E_INTERNAL

3. ExceptionHandler::HandleException(hcomm next/框架C接口边界, 39处)
   C++异常类型 -> HcclResult映射表:
   - HcclException -> e.code()原样传出
   - out_of_range -> HCCL_E_NOT_FOUND
   - runtime_error -> HCCL_E_RUNTIME
   - logic_error -> HCCL_E_INTERNAL
   - std::exception -> HCCL_E_INTERNAL
   - ... -> HCCL_E_INTERNAL

4. ExceptionInfo::errorCodeMap(hcomm legacy CCU层)
   ExceptionType -> HcclResult映射表:
   - CCU_API_EXCEPTION -> HCCL_E_INTERNAL
   - SOCKET_EXCEPTION -> HCCL_E_TCP_CONNECT
   - RMA_CONN_EXCEPTION -> HCCL_E_ROCE_CONNECT
   - TIMEOUT_EXCEPTION -> HCCL_E_TIMEOUT
   - SUSPENDING_EXCEPTION -> HCCL_E_SUSPENDING
   - NULL_PTR_EXCEPTION -> HCCL_E_PTR

其余所有层(CHK_RET 15545次)都是原样透传。

### 双轨错误处理: CHK_RET vs THROW

分界线: CCU微码层用THROW, 其余用CHK_RET。
转换点: EXCEPTION_HANDLE_BEGIN/END在C API边界捕获异常并映射为HcclResult。

两套异常类并存:
- `Hccl::HcclException`(旧, ExceptionType+errorCodeMap)
- `hccl::HcclException`(新, 直接持有HcclResult code_)
CCU ccu_assist.cc注释"todo: 需要统一整改为不抛异常"印证迁移中。

### 日志宏体系

三层调用链:
```
HCCL_ERROR(fmt, ...)
  -> HcclCheckLogLevel(HCCL_LOG_ERROR)  // LIKELY提示
     -> HCCL_ERROR_LOG_PRINT(fmt, ...)
        -> IsErrorToWarn() ?
           ├ true  -> LOG_FUNC(HCCL_LOG_MASK, WARN, "ErrToWarn: " fmt)
           └ false -> LOG_FUNC(HCCL, ERROR, fmt)
```

每条日志自动携带`[文件名:行号] [线程tid]`前缀。

ErrToWarn降级机制: 通过thread_local `g_hcclErrToWarn`变量,
hcomm用LogControl RAII类管理(拓扑探测重试时开启,避免噪音日志)。

### 日志级别分支预测

- HCCL_DEBUG/INFO/WARNING: UNLIKELY(假设通常不打)
- HCCL_ERROR: LIKELY(假设通常会打)

### RPT上报宏(与日志独立)

RPT_INPUT_ERR/RPT_ENV_ERR是纯上报宏,不打日志不return,
向错误管理系统写入结构化的用户可见错误信息(EI0003等标准码)。
必须配合CHK_PTR_NULL/CHK_PRT_RET才能拦截执行流。

量化: hcomm RPT_INPUT_ERR 303次 + RPT_ENV_ERR 68次,
hccl RPT_INPUT_ERR 57次 + RPT_ENV_ERR 8次。

### HCCL_ERROR_CODE(64位诊断码)

```
bit[63:32] = SYSTEM_RESERVE_ERROR
bit[31:24] = HCCL_MODULE_ID(5=HCCL)
bit[23:16] = HcclSubModuleID
bit[15:0]  = HcclResult枚举值
```

仅用于日志打印中的`errNo[0x%016llx]`格式,函数返回值仍是HcclResult枚举。
hcomm含`errNo[0x%016llx]`的日志1273处, hccl仅71处。

### hcomm独有: 三级结构化日志关键词

在`hccl_common.h`中定义,专用于日志过滤和故障定位工具:

一级(故障阶段): TaskExecStage / InitGroupStage / InitChannelStage / LinkInfo
二级(故障类型): Timeout / RunFailed / HeartbeatAbnormal / EnvConfig / InvalidArgument / Not Supported
三级(执行位置): HOST / HOST_TS / AIV / AICPU / ROCE CQE ERROR

使用方式:
```cpp
HCCL_ERROR("[%s][%s]errNo[0x%016llx] data type[%s] not supported",
    LOG_KEYWORDS_TASK_EXEC.c_str(),
    LOG_KEYWORDS_INVALID_ARGUMENT.c_str(),
    HCCL_ERROR_CODE(HCCL_E_NOT_SUPPORT), ...);
```

hccl没有这套体系。

### DFX诊断注册

hcomm: RegisterDfxInfo/UnRegisterDfxInfo在算子执行前后注册/注销诊断信息(826处)
hccl: DFX仅2处(src/ops/op_common/template/aicpu/dfx/), 序列化算子信息

### 日志量化对比

| 宏 | hccl | hcomm | 比例 |
|----|------|-------|------|
| HCCL_ERROR | 594 | 8531 | 1:14 |
| HCCL_WARNING | 188 | 1336 | 1:7 |
| HCCL_INFO | 927 | 7579 | 1:8 |
| HCCL_DEBUG | 488 | 3059 | 1:6 |
| HCCL_RUN_INFO | 29 | 977 | 1:34 |
| CHK_RET | 1448 | 14097 | 1:10 |
| EXECEPTION_CATCH | 1 | 229 | 1:229 |

HCCL_RUN_INFO差异最悬殊(1:34): hcomm面向用户的运行时诊断投入更多。
HCCL_TRACE标记配合HCCL_RUN_INFO记录关键里程碑(初始化完成/建链成功等, 38处)。

---

## 5.3.3 Health Check & Fault Recovery

### 心跳机制(hcomm独有)

核心文件: src/framework/cluster_maintenance/health/heartbeat/

时间常数:
```
BROADCAST_INTERVAL              = 50 ms     背景线程周期(不带算子检查)
BROADCAST_INTERVAL_WITH_CHECK   = 25 ms     带算子一致性检查时
HEARTBEAT_INTERVAL              = 1000 ms   心跳帧发送间隔
HEARTBEAT_COUNT                 = 20        每20个loop发一帧
lostThreshold_                  = 30        30次未收到=30s,判定丢失
STUCK_INTERVAL                  = 300000 ms 卡住检测默认周期5分钟
stuckDetectTime_                = max(execTimeout/3, 60s)
```

心跳状态枚举:
```
HEARTBEAT_OK           正常
HEARTBEAT_LOST         lostNum >= 30(30s未收到心跳)
HEARTBEAT_NOTIFY       预留(广播错误)
HEARTBEAT_CQE_ERR      RDMA CQE错误
HEARTBEAT_OPRETRY_NOT_SUPPORT  算子重试不支持
HEARTBEAT_STUCK        算子执行计数器无变化(进程/设备卡住)
HEARTBEAT_INCONSISTENT 各rank调用的集合通信算子不一致
```

心跳帧不只是存活ping: HeartBeatFrameWithOpCheck携带每个rank正在执行的
算子信息(opType/dataType/reduceOp/root/count), 用于检测算子入参不一致。

卡住检测: 通过OpExeCounter::GetCounter()读取算子执行计数器前后两次值,
完全相同则视为卡住, 广播HEARTBEAT_STUCK。检测到后周期指数退避。

关键事件过滤(IsKeyEvent): 事件到达时间距HCCL_EXEC_TIMEOUT在+-300s内
才认为是"关键事件"触发上报, 防止短暂网络抖动误报。

心跳失败传播路径:
```
lostNum >= 30
  -> SetStatus(crimer, informer, HEARTBEAT_LOST)
  -> errStatusQueue_(最多5000条)
  -> ProcessExceptionEvent(): 向所有peer广播裁决帧
  -> GetErrStatusVec(): 过滤关键事件, 返回可读错误字符串
```

### OpRetry重执行机制(hcomm独有)

核心文件: src/framework/cluster_maintenance/recovery/operator_retry/

时间常数:
```
OP_RETRY_MAX_CNT                    = 3        最大重试次数
OP_RETRY_WAIT_AICPU_TIMEOUT         = 5 s      等待Aicpu停止
OP_RETRY_WAIT_AGENT_AICPU_TIMEOUT   = 10 s     等待Agent+Aicpu合计
OP_RETRY_POLL_AICPU_ERROR_INTERVAL  = 1 s      正常态轮询Aicpu错误
OP_RETRY_POLL_RDMA_ERROR_INTERVAL   = 1 s      正常态轮询RDMA CQE
OP_RETRY_SEND_RECV_TIMEOUT          = 200 s    socket send/recv超时
OP_RETRY_KEEP_INTERVAL              = 1 s      agent->server保活
OP_RETRY_RUNNING_POLL_INTERVAL      = 100000us 状态机正常轮询
```

触发条件:
- SDMA错误: KfcError::kSdma -> 触发重执行
- RDMA CQE错误(借轨使能时): QPN状态码0x0C -> 触发重执行

Server-Agent双状态机(22态):

Server正常恢复流程:
```
SERVER_RUNNING
  -> CMD_STOP_AICPU      向所有Agent发RETRY_CMD_STOP_AICPU
  -> WAIT_AICPU_STOPED
  -> CMD_STOP_STREAM
  -> WAIT_STREAM_STOPED
  -> CMD_CLEAR_STREAM
  -> WAIT_STREAM_CLEARED
  -> CMD_STOP_TRANSPORT
  -> WAIT_STOP_TRANSPORT
  -> CMD_CHECK_LINK      借轨决策
  -> WAIT_LINK_CHECKED -> CHECK_ALL_LINK 汇总所有Agent网口状态
  -> CMD_RESUME_TRANSPORT
  -> WAIT_RESUME_TRANSPORT
  -> CMD_RESET_NOTIFY
  -> WAIT_NOTIFY_RESETED
  -> CMD_CHECK           校验所有节点算子一致
  -> WAIT_CHECK_INFO -> CHECK_OP
  -> CMD_CAN_RETRY
  -> WAIT_CAN_RETRY -> SERVER_RUNNING 重执行完成
```

设计模式: "聚合分发" -- Server拥有所有Agent的socket连接,
所有命令由Server统一发出、汇总响应后才推进状态;
Agent是被动响应方,只轮询本地Aicpu状态和RDMA CQE。

### KFC故障恢复协议

KfcCommand(aicpu_operator_pub.h:294-312):
```
kNone           空命令
kStopLaunch     停止算子下发
kStopExec       停止算子执行
kClear          清理算子状态和资源
kChangeLink     切换主备链路(借轨)
kRetry          重新执行算子
kExit           退出算子执行
NsStopLaunch    N秒快恢专用(4个Ns*命令)
NsStopExec
NsClear
NsChangeLink
kDestroyComm    销毁AICPU通信域
kSwitchNic      发起主动借轨
kWaitSwitchNic  等待主动借轨完成
kAllSwitched    所有卡完成
kSwitchFail     借轨失败回退
kReportRetryErr 退出但支持step重计算
```

KfcStatus(18种状态):
kNull/kRuning/kEnd/kStoplaunch/kStopExec/kClear/kChanged/kError/
kDestroyComm/kPlanSwitch/kSwitchError/kWaitSwitchRes/kSwitchSuccess/
kSwitchFail/kRetryError/kResumeError/kResumeChanged

KfcError(8种错误):
kNone/kSdma/kRdma/kTimeout/kInner/kExec/kExit/kExecConstraint

Host-Device通信通道: HDC(Host-Device Channel)
Agent通过hdcPtr->Put()写KfcCommand, hdcPtr->Get()读KfcStatus。

双通道恢复路径:
- 标准OpRetry: kStopLaunch -> kStopExec -> kClear -> kChangeLink -> kRetry
- N秒快恢: NsStopLaunch -> NsStopExec -> NsClear -> NsChangeLink (快速通道)

### 借轨(ChangeLink)机制

当CHECK_ALL_LINK发现有网口故障时:
```
Server发CMD_CHANGE_LINK -> 向Agent发ChangeLinkInfo(remoteRankList+isUseDefaultPort)
  -> Agent通过SetOpChangeLinkInfo()写入device
  -> Aicpu执行借轨
  -> 上报kChanged状态
  -> Server收到所有Agent确认后推进到下一步
```

### 超时配置汇总

| 层次 | 配置变量 | 默认值 |
|------|---------|--------|
| 建链超时 | HCCL_CONNECT_TIMEOUT | 120s |
| 算子执行超时 | HCCL_EXEC_TIMEOUT | 1836s(27*68) |
| AIV执行超时 | HCCL_EXEC_TIMEOUT | 1091s |
| 心跳丢失阈值 | 硬编码lostThreshold_ | 30s |
| 卡住检测 | max(EXEC_TIMEOUT/3, 60s) | ~612s |
| OpRetry send/recv | 硬编码 | 200s |
| OpRetry等Aicpu停止 | 硬编码 | 5s |
| RDMA层timeout | HCCL_RDMA_TIMEOUT(指数编码) | 20(约4s) |

### 链路故障检测

RDMA CQE错误检测(heartbeat.cc:1691-1700):
- OpRetryCQEHandle()每次最多获取RETRY_CQE_ARRAY_SIZE=128个CQE
- 通过HcclCommunicator::GetTransportCqeErrors()从驱动获取
- 按identifier分组存入remoteIpMap

独立连通性探测(DetectConnectionAnomalies):
- CQE错误后主动发起TCP握手验证
- 在VNIC和NIC两个通道上分别拉起client/server
- 广播时间窗口broadCastTime=10s

Transport层连接状态:
- SOCKET_OK/CONNECTING: 正常
- SOCKET_TIMEOUT/ERROR: 关闭socket
- HCCL_E_INTERNAL: 加入errorSocket_队列, 下一个loop的DelErrorSocket()统一清理

### Snapshot暂停机制

SnapshotControl在快照期间通过isPaused_暂停心跳收发和OpRetry状态机,
避免快照过程中误触发故障检测。

---

## Key Findings Summary

### 错误传播的核心规律

1. 透传为主: 9层调用栈中只有底层(driver调用点)产生新错误码,
   其余8层全部CHK_RET透传。用户拿到的HcclResult就是platform层最底部产生的值。

2. 转换点极少: 整个系统仅4个错误码转换点(CHK_PRT_RET/CHK_SAFETY_FUNC_RET/
   ExceptionHandler/ExceptionInfo), 其余15545次CHK_RET全部原样透传。

3. 边界类型不匹配: hccl-hcomm边界数据面用int32_t, 控制面用HcclResult,
   hccl侧通过static_cast<HcclResult>()桥接(165处)。

4. 日志层层追加: 每层CHK_RET自动追加`[__func__]call trace`日志,
   错误发生时可从日志重建完整调用栈。

5. 异步错误独立通道: RDMA CQE运行时错误不走同步return路径,
   通过Heartbeat+OpRetry异步处理。

### 健康检查与故障恢复的核心设计

1. 心跳不只是ping: 携带算子信息做一致性检查,是业务语义的健康检查。

2. Server-Agent聚合分发: Server统一指挥所有Agent,确保多rank一致性恢复。

3. 双通道恢复: 标准OpRetry和N秒快恢并存,快恢用于时间敏感场景。

4. 关键事件过滤: IsKeyEvent用+-300s时间窗口过滤非关键时段的事件,降低误报。

5. 全部hcomm独有: 心跳/OpRetry/KFC/DetectConnectionAnomalies/SnapshotControl
   全在hcomm中实现, hccl仓库无任何健康检查代码。

### 反模式

1. AP-EP-1: int32_t->HcclResult裸转换(static_cast)无值域校验,
   如果两边枚举不同步会导致语义错误。

2. AP-EP-2: 两套异常类(Hccl::HcclException vs hccl::HcclException)并存,
   映射表不同步,CCU层正在迁移中(TODO注释)。

3. AP-EP-3: g_qpn2IbversLinkMap_全局static map用于CQE错误定位,
   无锁保护,存在并发风险。

4. AP-EP-4: HCCL_E_INTERNAL作为通用兜底(132次/36文件),
   丢失了具体错误原因的信息。

5. AP-EP-5: CHK_PRT_RET中手动指定的错误码有时与底层真实错误不匹配,
   例如QP连接失败统一映射为HCCL_E_ROCE_CONNECT,丢失了具体失败原因。

6. AP-EP-6: OpRetry状态机22态,所有状态转换通过enum值比较+switch,
   无编译期完整性检查(漏处理某状态不会报错)。

7. AP-EP-7: 心跳帧中OpCheck信息通过socket发送,
   每帧最多500条算子信息,大规模场景可能截断。

8. AP-EP-8: HCCL_E_AGAIN走WARNING而非ERROR是通过CHK_RET宏硬编码的,
   不是调用方可配置的行为,如果其他错误码也需要类似降级则需修改宏。
