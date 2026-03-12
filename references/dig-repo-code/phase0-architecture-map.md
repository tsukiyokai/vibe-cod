# Phase 0: Global Architecture Survey + Common Infrastructure

## 0.1 hcomm 目录树扫描

hcomm 仓库约 4466 个 C++ 文件，采用三层架构。

### src/ 模块统计

| 模块路径 | 直接子目录数 | C++ 文件数 | 描述 |
|---------|-----------|----------|------|
| src/algorithm | 3 | 882 | 算法层: base(算法模板) + impl(具体实现) + pub_inc(头文件)。覆盖 AIV 和通用算法的 AllGather/AllReduce/AllToAll 等 |
| src/framework | 10 | 567 | 框架层: 集群维护、通信资源管理、通信域管理、AICPU 实现、网络负载均衡等 |
| src/platform | 10 | 394 | 平台层: 通信原语、资源管理、硬件抽象、HCCP 协议栈、远程访问、调试维测等 |
| src/legacy | 7 | 1221 | 遗留代码: 旧版 AICPU/CCU/通信器实现，向后兼容 |
| src/common | 5 | 48 | 通用工具: 调试配置、分析器、健康检查、任务管理 |
| src/pub_inc | 3 | 60 | 公开接口头文件: aicpu/inner/new 三个命名空间 |
| src/hccd | 0 | 11 | 通信守护进程 |

### src/ 外的顶级目录

| 目录 | 用途 |
|-----|------|
| include/ | 对外公开 C++ 头文件 |
| pkg_inc/ | 包间接口定义 |
| test/ | 单元测试(ut/) + 系统测试(st/) |
| examples/ | 示例代码 |
| docs/ | 项目文档 |
| python/ | Python 绑定和脚本工具 |
| scripts/ | 编译部署脚本 |
| cmake/ | CMake 构建配置 |
| externel_depends/ | 外部依赖 |

### 关键观察

1. legacy 模块文件数最多(1221)，说明大量历史代码仍在维护
2. 三层架构清晰: algorithm(算法) → framework(框架) → platform(平台)
3. algorithm 层(882 文件)复杂度高，体现通信算法的多样性

---

## 0.2 hccl 目录树扫描

hccl 仓库约 570 个 C++ 文件，结构为 common + 13 个算子。

### src/ 模块统计

| 模块路径 | 直接子目录数 | C++ 文件数 | 描述 |
|---------|-----------|----------|------|
| src/common/ | 0 | 22 | 通用基础设施: 日志、参数检查、ACL 适配、类型定义、SAL |
| src/ops/op_common/ | 5 | 70 | 算子公共组件: Executor/Registry/Selector/Topo/Template/Wrapper |
| src/ops/all_reduce/ | 3 | 55 | AllReduce(Mesh/RHD/2-die 拓扑) |
| src/ops/all_to_all_v/ | 3 | 50 | AllToAll/AllToAllV |
| src/ops/reduce_scatter/ | 3 | 49 | ReduceScatter |
| src/ops/all_gather/ | 3 | 46 | AllGather |
| src/ops/scatter/ | 4 | 46 | Scatter(Mesh/Ring/NHR/NB) |
| src/ops/reduce/ | 3 | 30 | Reduce |
| src/ops/broadcast/ | 3 | 28 | Broadcast |
| src/ops/all_gather_v/ | 3 | 8 | AllGatherV |
| src/ops/reduce_scatter_v/ | 3 | 8 | ReduceScatterV |
| src/ops/send/ | 2 | 6 | Send 点对点通信 |
| src/ops/recv/ | 2 | 6 | Recv 点对点通信 |
| src/ops/batch_send_recv/ | 2 | 6 | 批量 Send/Recv |

### src/ 外的顶级目录

| 目录 | 用途 |
|-----|------|
| include/ | 对外公开头文件(hccl.h, hccl_mc2.h) |
| test/ut/ | 单元测试 |
| test/st/ | 系统集成测试 |
| examples/ | 4 类示例(点对点/集合通信/AI框架集成/自定义算子) |
| cmake/ | CMake 构建脚本 |
| docs/ | 文档和架构图 |
| scripts/ | 辅助脚本 |

### 关键统计

- 源码文件: common(22) + ops(408) = 430
- 测试文件: 114
- 示例文件: 24

### 每个算子的统一目录模式

```
ops/<algo>/
  ├── *_op.h/cc          # 算子接口
  ├── executor/          # 执行器(并发/序列等策略)
  ├── selector/          # 算法选择器
  └── template/          # 硬件特定模板(aicpu/aiv/ccu)
```

### 关键观察

1. ops/ 下 13 个算子全部遵循 executor/selector/template 三件套模式
2. op_common/ 是基础设施枢纽，定义了 executor/selector/template/registry/topo 五大系统
3. common/ 只有 22 个文件，设计轻量
4. 深度面向硬件: aicpu/aiv/ccu 三套内核实现

---

## 0.3 hcomm 公共基础设施分析

### 1. 错误码体系

定义位置: `include/hccl/hccl_types.h:23-51` (HcclResult enum)

25 个错误码，分组:
- 0: HCCL_SUCCESS
- 1-10: 参数/内存/资源错误 (PARA, PTR, MEMORY, INTERNAL, NOT_SUPPORT, NOT_FOUND, UNAVAIL, SYSCALL, TIMEOUT, OPEN_FILE_FAILURE)
- 11-19: 网络/系统/API错误 (TCP_CONNECT, ROCE_CONNECT, TCP_TRANSFER, ROCE_TRANSFER, RUNTIME, DRV, PROFILING, CCE, NETWORK)
- 20-24: 重试/状态错误 (AGAIN, REMOTE, SUSPENDING, OPRETRY_FAIL, OOM)
- 1041: HCCL_E_IN_STATUS (特殊状态编码)

错误码编码策略 (`src/pub_inc/log.h:131-134`):
```cpp
#define HCCL_ERROR_CODE(error) \
    ((SYSTEM_RESERVE_ERROR << 32) + (HCCL_MODULE_ID << 24) + \
     ((HcclSubModuleID::LOG_SUB_MODULE_ID_HCCL) << 16) + error)
```
分层: [保留(32b) | 模块(8b) | 子模块(8b) | 错误号(16b)]

模块 ID: HCCL_MODULE_ID = 5，子模块: HCCL/HCOM/CLTM/CUSTOM_OP

### 2. CHK_RET 系列宏 (返回值检查)

定义位置: `src/pub_inc/log.h:139-279`

使用频次: CHK_RET 在 hcomm 中出现 14,173 次

| 宏名 | 行为 | 使用场景 |
|-----|------|---------|
| CHK_RET(call) | 执行 call，失败则 log + return 错误码。HCCL_E_AGAIN 记 warning，其余记 error | 最常用，主力错误传播 |
| CHK_RET_AND_PRINT_IDE(call, id) | 同 CHK_RET，额外打印通信域标识 | 通信域相关操作 |
| CHK_RET_NULL(call) | 失败则 log + return void | 析构/清理函数 |
| CHK_PTR_NULL(ptr) | 空指针则 return HCCL_E_PTR | 入口参数验证 |
| CHK_SMART_PTR_NULL(sp) | 智能指针空则 return HCCL_E_PTR | 内部管理指针 |
| CHK_PRT_RET(cond, log, ret) | 条件为真则执行 log + return ret | 自定义条件+错误码 |
| CHK_PRT_CONT(cond, log) | 条件为真则 log + continue | 循环中跳过错误项 |
| CHK_PRT_BREAK(cond, log, cmd) | 条件为真则 log + cmd + break | 循环中断 |
| CHK_SAFETY_FUNC_RET(call) | 检查 C 库安全函数(返回 EOK)，失败则 return HCCL_E_INTERNAL | memcpy_s/sprintf_s |
| CHK_PRT(call) | 执行 call，失败仅 log 不 return | 非关键路径 |
| EXECEPTION_CATCH(expr, ret) | try-catch std::exception | 异常边界 |
| NEW_NOTHROW(ptr, ctor, ret) | nothrow new + 空检查 | 堆分配 |

### 3. 日志宏体系

定义位置: `src/pub_inc/log.h:56-116`

使用频次: 日志宏在 hcomm 中共 21,460 次调用，覆盖 1,094 个文件

| 宏名 | 级别 | 特点 |
|-----|------|------|
| HCCL_DEBUG | DEBUG | 用 UNLIKELY() 优化分支预测，先检查级别再格式化 |
| HCCL_INFO | INFO | 标准信息 |
| HCCL_WARNING | WARNING | 警告 |
| HCCL_ERROR | ERROR | 支持 ErrToWarn 开关（可将 error 降级为 warning） |
| HCCL_RUN_INFO | INFO | 运行日志，用于操作级追踪 |
| HCCL_USER_CRITICAL_LOG | CRITICAL | 用户可见的关键日志 |

日志格式: `[文件:行号] [线程ID] 消息`

配置日志 (`src/common/debug/config/config_log.h:31-59`):
- HCCL_CONFIG_DEBUG/INFO: 按掩码控制细粒度日志 (ALG/TASK/RES/AIV_OPS_EXC)
- HCCL_ENTRY_INFO: 内核展开入口日志

### 4. 基础类型定义

定义位置: `src/pub_inc/hccl_common.h:37-49`, `pkg_inc/hccl/base.h:19-26`

```cpp
using s8/s16/s32/s64 = signed char/short/int/long long
using u8/u16/u32/u64 = unsigned char/short/int/long long
using char_t = char
```

通信句柄类型 (全部为 void*):
- HcclComm, HcclConn, HcclRtStream, HcclRtNotify, HcclRtEvent
- RdmaHandle, SocketHandle, FdHandle, QpHandle, MrHandle

### 5. 关键枚举定义

CommEngine (`include/hccl/hccl_res.h:72-80`): 6 种执行引擎
- CPU, CPU_TS, AICPU, AICPU_TS, AIV, CCU

CommProtocol (`include/hccl/hccl_res.h:98-107`): 7 种通信协议
- HCCS(片上), ROCE(RDMA), PCIE, SIO, UBC_CTP, UBC_TP, UB_MEM

TransportType (`src/pub_inc/hccl_common.h:320-332`): 10 种传输类型
- IBV_EXP, P2P, HOST_SHM, HOST_TCP, ROCE, HETEROG_P2P/ROCE, DEVICE_P2P/IBVERBS/DIRECT

DevType (`pkg_inc/hccl/dtype_common.h:22-32`): 7 种设备类型
- 910, 310P3(PG), 910B, 310P1(AG), 910_93, NOSOC, 910_95

HcclDataType (`include/hccl/hccl_types.h:90-109`): 18 种数据类型
- INT8/16/32/64, UINT8/16/32/64, FP16/32/64, BFP16, INT128, HIF8, FP8E4M3/E5M2/E8M0

HcclReduceOp (`include/hccl/hccl_types.h:78-85`): SUM, PROD, MAX, MIN

HcclCMDType (`include/hccl/hccl_types.h:201-228`): 20+ 集合操作类型
- BROADCAST, ALLREDUCE, REDUCE, SEND, RECEIVE, ALLGATHER, REDUCE_SCATTER, ALLTOALLV...

LinkTypeInServer (`src/pub_inc/hccl_common.h:248-261`): HCCS, PXI(PCIe), SIO, HCCS_SW

### 6. 关键常量

定义位置: `src/pub_inc/hccl_common.h:51-185`

系统限制:
- HCCL_AISERVER_DEVICE_NUM = 8 (单服务器最大设备数)
- MAX_MODULE_DEVICE_NUM = 32 (双模组最大设备数)
- HCCL_DEVICE_NIC_NUM = 3 (单设备最大网卡数)
- GROUP_NAME_MAX_LEN = 127
- HCCL_ALIGN_SIZE = 4096 (内存对齐)
- DEVICE_MEMORY_MAX_ALLOC_SIZE = 16GB

无效值哨兵:
- INVALID_VALUE_RANKID = 0xFFFFFFFF
- INVALID_VALUE_RANKSIZE = 0xFFFFFFFF
- INVALID_U64 = 0xFFFFFFFFFFFFFFFF
- HOST_DEVICE_ID = -1

通信端口:
- HETEROG_CCL_PORT = 16666
- HOST_CONTROL_BASE_PORT = 60000
- HOST_PARA_BASE_PORT = 60008

### 7. 资源相关结构体

定义位置: `include/hccl/hccl_res.h:54-221`

- CommMem: {type(DEVICE/HOST), addr, size} — 内存区域描述
- EndpointDesc: {protocol, commAddr(IPv4/v6/ID/EID), loc} — 网络端点
- HcclChannelDesc: {header(ABI兼容), remoteRank, protocol, endpoints, roceAttr} — 通道配置
- HcclBuf: {addr, len, handle} — 缓冲区描述

### 8. 工具类

Referenced (`src/pub_inc/hccl_common.h:334-369`): 引用计数基类 (Ref/Unref/Count/Clear/IsZero)

诊断关键字 (`src/pub_inc/hccl_common.h:146-176`):
- 一级: TaskExecStage, InitGroupStage, InitChannelStage
- 二级: Timeout, RunFailed, HeartbeatAbnormal, EnvConfig, RanktableConfig...
- 三级: HOST, HOST_TS, AIV, AICPU, ROCE CQE ERROR

一致性校验 (`src/common/health/rank_consistentcy_checker.h`):
- CRC 校验 ranktable/variable counts/displacements
- HcclCMDInfo 存储操作参数用于 rank 间一致性验证

---

## 0.4 hccl 公共基础设施分析

### 1. 公共 API (include/)

hccl.h — 集合通信 API:
- HcclAllReduce, HcclBroadcast, HcclReduceScatter(V), HcclScatter
- HcclAllGather(V), HcclSend, HcclRecv, HcclAlltoAll(V/VC)
- HcclReduce, HcclBatchSendRecv

hccl_mc2.h — MC2(Kfc) 扩展 API:
- HcclKfcAllocOpArgs/FreeOpArgs: 操作参数内存管理
- HcclKfcOpArgsSet*: 参数设置(数据类型、规约类型、算法配置、通信引擎)
- HcclCreateOpResCtx: 创建操作资源上下文

### 2. 日志系统 (src/common/log.h)

与 hcomm 共享同一套宏定义模式，但定义在 hccl 自己的 log.h 中:
- HCCL_DEBUG/INFO/WARNING/ERROR/RUN_INFO/RUN_WARNING/USER_CRITICAL_LOG
- 日志格式: `[文件:行号] [线程ID] 消息`
- UNLIKELY() 分支预测优化
- LOG_TMPBUF_SIZE = 512: 小于此值栈分配，否则堆分配
- 支持 ErrToWarn 开关 (SetErrToWarnSwitch/IsErrorToWarn)

### 3. CHK_RET 系列宏 (src/common/log.h)

与 hcomm 同套模式:
- CHK_RET, CHK_RET_NULL, CHK_PTR_NULL, CHK_SMART_PTR_NULL
- CHK_PRT_RET, CHK_PRT_CONT, CHK_PRT_BREAK
- CHK_SAFETY_FUNC_RET, EXECEPTION_CATCH, NEW_NOTHROW
- CHK_RET_AND_PRINT_IDE (带通信域标识)

### 4. 参数校验 (src/common/param_check.h/cc)

导出函数 (namespace ops_hccl):
- HcomCheckOpParam: 3 个重载，校验 tag/count/dataType/group/stream
- HcomCheckUserRank: 校验 userRank < totalRanks

校验规则:
- 组名: 0 < 长度 <= 127
- 标签: 0 < 长度 <= TAG_MAX_LEN
- 数据量: count <= SYS_MAX_COUNT (0x7FFFFFFFF)
- 数据类型: 必须在 HCCL_SUPPORT_DATA_TYPE 集合中
- 报错: RPT_INPUT_ERR 宏上报到错误管理系统 (错误码 "EI0003")

### 5. ACL 适配层 (src/common/adapter_acl.h/cc)

ACLCHECK 宏: 包装 ACL API 调用，失败返回 HCCL_E_RUNTIME，特殊处理内存分配错误

核心接口:
- haclrtGetPairDeviceLinkType: 获取两设备间链路类型 (HCCS/HCCS_SW/SIO/PXI)
- haclrtGetDeviceIndexByPhyId: 物理ID→逻辑ID，处理 NOSOC 特殊情况
- hcalrtGetDeviceInfo: 查询设备属性
- haclrtGetCaptureInfo: 查询 stream 是否在 RI 捕获中
- LoadBinaryFromFile: 加载二进制内核

条件编译: AICPU_COMPILE 区分 CPU 端编译和 NPU 端编译

### 6. SAL 系统抽象层 (src/common/sal.h/cc)

- SalStrToULong: 安全字符串→整数转换 (异常捕获 + 范围检查)
- IsAllDigit: 字符串全数字验证
- SalStrLen: 安全长度查询 (封装 strnlen)
- TIME_NOW()/DURATION_US(): 时间工具 (基于 steady_clock)
- weak_alias 宏: 弱符号别名

### 7. 算法配置 (src/common/alg_type.h/cc, alg_env_config.h/cc)

HcclAlgoType 枚举: DEFAULT/RING/PIPELINE/FULLMESH/HDR/PAIRWISE/NHR/NHR_V1/NB/AHC/AHC_BROKE

三层拓扑算法:
- Level 0: 节点内 (8P Ring/4P Mesh/2P Mesh/NP HD/Star/Pairwise...)
- Level 1: 节点间 (Ring/Pipeline/H-D/Star/NHR/AHC...)
- Level 2: 超节点间 (Ring/H-D/NHR...)
- AlgType struct 组合三层，默认 WHOLE_RING

环境变量配置 (AlgEnvConfig):
- HCCL_OP_EXPANSION_MODE, HCCL_DETERMINISTIC, HCCL_INTRA_PCIE/ROCE_ENABLE
- HCCL_ENTRY_LOG_ENABLE, HCCL_INTER_HCCS_DISABLE, HCCL_OP_RETRY_ENABLE
- 容错设计: 配置解析失败通过 RPT_ENV_ERR 上报但继续初始化

### 8. 工具类

BinaryStream (`src/common/binary_stream.h`):
- 通信数据结构序列化/反序列化
- 支持基础类型、vector、string、map
- operator<< / operator>> 接口

UniversalConcurrentMap (`src/common/universal_concurrent_map.h`):
- 线程安全并发映射 (std::shared_timed_mutex)
- Find(共享锁) / Emplace(独占锁) / EmplaceIfNotExist / EmplaceAndUpdate

utils.h/cc:
- HcclMemRange: 内存区间切割 (带边界检查)
- RoundUpWithDivisor: 向上对齐
- StringFormat: printf 风格格式化

### 9. 错误管理适配 (src/common/adapter_error_manager_pub.h/cc)

弱符号接口:
- RptInputErr / RptEnvErr: __attribute__((weak))
- RPT_INPUT_ERR / RPT_ENV_ERR 宏: 条件调用 + UNLIKELY 优化
- 底层调用 REPORT_PREDEFINED_ERR_MSG (来自 base/err_mgr.h)

### 10. 配置日志 (src/common/config_log.h/cc)

调试掩码位:
- HCCL_ALG (bit 0), HCCL_TASK (bit 1), HCCL_RES (bit 2), HCCL_AIV_OPS_EXC (bit 3)
- 环境变量 HCCL_DEBUG_CONFIG 初始化，支持 "ALG,TASK,RESOURCE" 格式，`^` 前缀取反

### 11. MC2 支持 (src/common/hccl_mc2.cc)

HcclOpArgs 结构体: {srcDataType, dstDataType, reduceType, count, algConfig[128], commEngine, reverse}
- HcclKfcAllocOpArgs/FreeOpArgs: malloc/free 生命周期
- HcclKfcOpArgsSet*: 参数设置接口
- HcclCreateOpResCtx: 调用弱符号 HcclCreateOpResCtxInner

---

## 0.3+0.4 交叉观察: hccl 与 hcomm 公共基础设施对比

1. 错误码共享: 两仓使用同一套 HcclResult 枚举 (定义在 hcomm 的 include/hccl/hccl_types.h)
2. 宏模式一致: CHK_RET/日志宏在两仓定义模式相同，说明是项目级强制规范
3. 类型共享: 基础类型(s32/u64等)、数据类型枚举、规约操作枚举均由 hcomm 定义，hccl 依赖
4. hccl 更轻量: common/ 只有 22 个文件 vs hcomm 的 48+60(pub_inc)，hccl 聚焦算子逻辑
5. 弱符号模式: 两仓都用 __attribute__((weak)) 做错误上报适配，允许上层框架注入回调
6. 配置驱动: 两仓都支持环境变量配置，容错设计(解析失败不阻塞初始化)

---

## 0.5 hccl→hcomm 调用接口分析

### 依赖方向: hccl 单向依赖 hcomm

证据:
- hccl 大量 #include hcomm 公共头文件
- hcomm 源码不包含 hccl 的实现头文件（仅引用共享类型定义）
- libhccl.so 链接 libhcomm.so (CMakeLists.txt 中 target_link_libraries)

### hcomm 暴露的 5 大接口类别

#### A. 通信域生命周期 (include/hccl/hccl_comm.h) — ~30 个导出函数

核心函数:
- HcclCommInitClusterInfo/Config: 从集群配置文件初始化
- HcclGetRootInfo + HcclCommInitRootInfo/Config: 跨进程初始化
- HcclCommInitAll: 多 NPU 单进程初始化
- HcclCreateSubCommConfig: 创建子域
- HcclCommDestroy: 销毁通信域
- HcclGetRankSize/Id/CommName: 查询基本信息
- HcclBarrier: 同步屏障
- HcclCommSuspend/Resume: 暂停/恢复
- HcclCommSetMemoryRange/Unset: 虚拟内存范围管理
- HcclGroupStart/End: 操作分组(实验 API)
- HcclCommSymWinRegister/Deregister/Get: 对称内存窗口

#### B. 资源管理 (include/hccl/hccl_res.h) — ~15 个导出函数

核心函数:
- HcclThreadAcquire/WithStream: 获取通信线程资源
- HcclChannelAcquire: 创建通信通道
- HcclChannelGetHcclBuffer: 获取通道通信缓存
- HcclEngineCtxCreate/Get/Copy: 通信引擎上下文管理
- HcclDevMemAcquire: 获取设备内存(MC2 场景)
- HcclThreadExportToCommEngine: 线程资源导出
- HcclCommMemReg: 内存注册
- HcclTaskRegister/UnRegister: 任务注册(实验 API)

#### C. 拓扑查询 (include/hccl/hccl_rank_graph.h) — ~10 个导出函数

核心函数:
- HcclRankGraphGetLayers: 获取网络层次
- HcclRankGraphGetRanksByLayer: 获取指定层 rank 列表
- HcclRankGraphGetRankSizeByLayer: 获取指定层 rank 数量
- HcclRankGraphGetTopoTypeByLayer: 获取拓扑类型
- HcclRankGraphGetInstSizeListByLayer: 拓扑实例大小列表
- HcclRankGraphGetLinks: 查询 rank 间链路信息
- HcclGetHeterogMode: 异构组网模式

#### D. 数据面通信原语 (include/hccl/hcomm_primitives.h) — ~30 个导出函数

核心函数:
- HcommLocalCopyOnThread: 本地内存拷贝
- HcommLocalReduceOnThread: 本地规约
- HcommWriteOnThread: 单边 RDMA 写
- HcommReadOnThread: 单边 RDMA 读
- HcommWriteReduceOnThread/ReadReduceOnThread: 规约读写
- HcommThreadNotifyRecord/Wait: 线程间同步
- HcommChannelNotifyRecord/Wait: 通道级通知
- HcommBatchModeStart/End: 批量模式
- HcommAcquireComm/ReleaseComm: 通信域锁定
- HcommSymWinGetPeerPointer: 对称内存对等端指针
- HcommThreadSynchronize: 线程同步
- HcommChannelFence: 通道内存屏障

#### E. CCU 资源 (pkg_inc/hcomm/ccu/hccl_ccu_res.h)

- HcclCcuKernelRegister/RegisterFinish: 注册 CCU kernel
- HcclCcuKernelLaunch: 启动 CCU kernel

### hccl 中的调用分布

| 接口类别 | 主要使用模块 | 调用频次 |
|---------|-----------|---------|
| hccl_comm.h | src/common/, examples/, test/ | <5 (初始化/销毁) |
| hccl_res.h | ops/op_common/op_common.cc, ops/*/op.cc | 30+ (资源获取) |
| hccl_rank_graph.h | ops/op_common/topo/, ops/op_common/selector/ | 40+ (拓扑查询) |
| hcomm_primitives.h | ops/op_common/template/, ops/*/template/ | 50+ (数据传输) |

### 关键调用链路

```
初始化: API → OpBase → AlgorithmExecutor
  → HcclEngineCtxCreate()     // 创建引擎上下文
  → HcclThreadAcquire()       // 获取执行线程
  → HcclChannelAcquire()      // 创建通信通道

执行: Algorithm Template Execute
  → HcclRankGraphGetLayers()             // 查询网络分层
  → HcclRankGraphGetRanksByLayer()       // 查询层内 rank
  → HcommWriteOnThread/ReadOnThread()    // 数据传输
  → HcommChannelNotifyWaitOnThread()     // 同步等待

销毁: HcclCommDestroy() → 自动回收所有资源
```

### 关键接口桥接文件

`src/ops/op_common/inc/alg_param.h` — 包含 6 个 hcomm 头文件，是 hccl→hcomm 的核心桥接

---

## 0.6 构建系统分析

### CMake 版本与 C++ 标准

- 两仓 CMake 最低版本: 3.16.0
- 两仓 C++ 标准: C++14 (-std=c++14)

### 编译选项 (两仓共通)

```
-Werror                    # 警告视为错误
-Wall                      # 全警告 (hcomm)
-fno-common                # 禁止 tentative definitions
-fno-strict-aliasing       # 禁止严格别名优化
-pipe                      # 管道编译
-O3                        # 优化级别
-fstack-protector-all      # 栈保护
-D_FORTIFY_SOURCE=2        # 缓冲区溢出检测
```

链接选项:
```
-Wl,-z,relro               # GOT 只读
-Wl,-z,now                 # 立即绑定
-Wl,-z,noexecstack         # 不可执行栈
-Wl,-Bsymbolic             # 优先绑定自身符号 (hcomm)
-Wl,--exclude-libs,ALL     # 不导出依赖库符号 (hcomm)
-s                         # Release 模式 strip
```

### 条件编译宏

| 宏名 | 定义位置 | 用途 |
|-----|---------|------|
| AICPU_COMPILE | hccl: scatter_aicpu_kernel 库 | AICPU kernel 编译 |
| HOST_COMPILE | hccl: hccl 主库 | Host 端编译 |
| HCCD | hcomm: hccd 库 | 通信守护进程编译 |
| CCL_KERNEL | hcomm: ccl_kernel_plf 库 | CCL kernel 编译 |
| CCL_KERNEL_AICPU | hcomm: ccl_kernel_plf 库 | AICPU kernel 编译 |
| USE_AICORE_REDUCESUM | hcomm: ccl_kernel_plf 库 | AI Core 规约求和 |
| OPEN_BUILD_PROJECT | hccl: BUILD_OPEN_PROJECT 时 | 开源构建模式 |

### 库产物

hccl:
- libscatter_aicpu_kernel.so — AICPU kernel 库 (链接 ccl_kernel)
- libhccl.so — 主库 (链接 hcomm, c_sec, unified_dlog, mmpa)

hcomm:
- libhccd.so — 通信守护进程 (链接 c_sec, mmpa, json, rt, dl, pthread)
- libccl_kernel_plf.so/.a — CCL kernel 平台库 (链接 c_sec, aicpu_sharder, mmpa, rt, dl, pthread)

### 链接依赖关系

```
libhccl.so
  └─> libhcomm.so (核心通信库)
  └─> libc_sec.so (安全 C 函数)
  └─> libunified_dlog.so (统一日志)
  └─> libmmpa.so (内存管理)

libhccd.so
  └─> libc_sec.so
  └─> libmmpa.so
  └─> libjson (nlohmann_json, 头文件库)
```

### 第三方依赖

| 库 | 版本 | 仓库 | 说明 |
|---|------|------|------|
| Google Test | - | hccl/hcomm (测试) | 单元测试框架 |
| OpenSSL | 3.0.9 | hcomm | TLS/加密 |
| nlohmann_json | 3.11.3 | hcomm | JSON 解析 (头文件库) |
| mockcpp | - | hcomm (测试) | Mock 框架 |
| protobuf | - | hcomm (测试) | 序列化 |
| Boost | - | hcomm (测试) | C++ 工具库 |
| abseil-cpp | - | hcomm (测试) | Google C++ 库 |

### Stub 库机制

两仓对无源码的外部库生成 stub 进行链接:
- hccl: ccl_kernel
- hcomm: ascend_hal, slog, aicpu_sharder, c_sec, unified_dlog, mmpa, ascendcl, tsdclient, runtime, acl_rt, metadef, opp_registry, error_manager

### 跨平台编译

- 支持 x86_64 和 aarch64
- aarch64 使用交叉编译工具链: TOOLCHAIN_DIR/hcc/
- ASCEND_CANN_PACKAGE_PATH 检测顺序: 环境变量 > 默认路径

### 版本管理

hccl 版本 9.0.0，构建依赖: hcomm(>=8.5), runtime(>=8.5), metadef(>=8.5), bisheng-compiler(>=8.5), asc-devkit(>=8.5)

### 设备侧构建

两仓支持 DEVICE_MODE 和 KERNEL_MODE:
- 生成 AICPU kernel tar.gz 包
- 签名机制: sign_file() / sign_aicpu_kernel()
- 设备侧产物独立打包和安装

---

## 0.7 核心基类与继承链分析

### 一、hccl 核心基类体系

#### 1. 执行器(Executor)层次

ExecutorBase (`ops/op_common/executor/executor_base.h:44`)
- 纯虚: `Orchestrate(const OpParam&, AlgResourceCtx*) = 0`
- 虚函数: `CalcResRequest(...)`, `KernelRun(const OpParam&, ExecMem&)`
- 设计模式: Template Method

InsCollAlgBase (`ops/op_common/executor/executor_v2_base.h:27`) — v2 版本
- 纯虚: `Orchestrate(const OpParam&, const AlgResourceCtxSerializable&) = 0`
- 纯虚: `CalcRes(...)`
- 子类 50+: InsReduceScatterParallelExecutor, InsV2AllGatherVSoleExecutor, InsBroadcastParallelExecutor...

ScatterExecutorBase (`ops/scatter/algo/scatter_executor_base.h:19`)
- 继承 ExecutorBase
- 保护虚函数: `KernelRunLevel1(...)`, `RunLoop(...)`

#### 2. 算法模板(Template)层次

AlgTemplateBase (`ops/op_common/template/alg_template_base.h:120`)
- 虚函数: `RunAsync(rank, rankSize, channels)`
- 虚函数: `Prepare(...)` — 多个重载(12参数版、PrepareData版等)
- 设计模式: Template Method

InsAlgTemplateBase (`ops/op_common/template/alg_v2_template_base.h:23`) — v2 版本
- 纯虚: `Describe() const = 0`
- 虚函数: `KernelRun(...)`, `DPUKernelRun(...)`, `CalcRes(...)`
- 虚函数: `GetNotifyIdxMainToSub/SubToMain()` — 线程间同步

设备特化模板:
- AivAlgTemplateBase (`template/aiv_alg_template_base.h:28`): 继承 AlgTemplateBase, AIV 设备
- CcuAlgTemplateBase (`template/ccu_alg_template_base.h:18`): 继承 AlgTemplateBase, CCU 设备
- CcuKernelAlgBase (`template/ccu/ccu_kernel_alg_base.h:21`): 继承 CcuKernel, CCU 内核级

#### 3. 选择器(Selector)层次

AutoSelectorBase (`ops/op_common/selector/auto_selector_base.h:70`)
- 虚函数: SelectCcuMsAlgo, SelectCcuScheduleAlgo, SelectAicpuAlgo, SelectAivAlgo, SelectDPUAlgo
- 设计模式: Strategy Pattern — 每个设备类型一个虚函数

子类: ReduceScatterAutoSelector, BroadcastAutoSelector, BatchSendRecvAutoSelector...

SelectorRegistry (`ops/op_common/selector/selector_registry.h:21`)
- 设计模式: Registry Pattern
- `static SelectorRegistry *Global()` — 全局单例
- `Register(u32 priority, AutoSelectorBase *selector)` — 带优先级注册
- `REGISTER_SELECTOR(priority, selector)` 宏 — __COUNTER__ 自动注册

#### 4. 拓扑匹配(Topo)层次

TopoMatchBase (`ops/op_common/topo/topo_match_base.h:54`)
- 纯虚: `Describe() const = 0`
- 纯虚: `MatchTopo(comm, topoInfo, algHierarchyInfo)`
- 设计模式: Strategy Pattern

子类: TopoMatch1D, TopoMatch2D, TopoMatchMultilevel, TopoMatchUbx

### 二、hcomm 核心基类体系

#### 1. 通信基类

CommBase (`algorithm/base/inc/comm_base_pub.h:38`)
- 构造函数: 13 参数
- 虚函数: Init, DeInit, CreateLinks, CalcLink, GetSocketsPerLink
- 虚函数: NeedDataReceivedAck, CreateIntraLinks, CreateInterLinks
- 虚函数: SetMachinePara, MakeClientInfo, MakeServerInfo
- 成员: `vector<shared_ptr<Transport>> transportInfo_` — 链路信息
- 设计模式: Template Method

#### 2. 集合执行器(新框架)

CollExecutorBase (`algorithm/impl/coll_executor/coll_executor_base.h:23`)
- 构造: 接收 dispatcher + unique_ptr<TopoMatcher>& (依赖注入)
- 纯虚: `CalcResRequest(param, resourceRequest) = 0`
- 纯虚: `Orchestrate(param, algRes) = 0`
- 虚函数: GetAivExecParam, PrepareCommInfoToDevice, CreatePairWiseList

CollNativeExecutorBase (`algorithm/impl/coll_executor/coll_native_executor_base.h:35`)
- 继承 CollExecutorBase
- 三级通信抽象: CalcLevel0CommInfo, CalcLevel1CommInfo, CalcLevel2CommInfo
- 虚函数: ParseParam, CalcCommInfo, CalcStreamNum, CalcScratchMemSize
- 虚函数: KernelRun, KernelRunInterServer, KernelRunIntraServerPre/Post
- 虚函数: SelectTempAlg
- 设计模式: Template Method 的分层应用

#### 3. 算法模板(旧框架)

AlgTemplateBase (hcomm 版, `algorithm/base/alg_template/alg_template_base_pub.h:248`)
- 150+ 种 TemplateType 枚举(行 56-150)
- 从 TEMPLATE_ALL_GATHER_HD_STAGE 到 TEMPLATE_REDUCESCATTER_HCCS_SIO
- 30+ 个具体实现: AllGatherRing, RecursiveHalvingDoublingBase, AlltoallPipelineBase...

AHCAlgTemplateBase (非对称分层连接, `asymmetric_hierarchical_concatenate_alg_template_base_pub.h:25`)
- 继承 AlgTemplateBase
- 子类: AllGatherAHCBase, AllReduceAHCBase, ReduceScatterAHCBase

#### 4. 重试状态机

OpRetryBase (`framework/cluster_maintenance/recovery/operator_retry/opretry_base.h:91`)
- 虚函数: Handle(RetryContext*)
- 纯虚: ProcessEvent(RetryContext*) = 0, ProcessError(RetryContext*) = 0
- 设计模式: State Pattern

状态子类:
- Agent 侧: OpRetryAgentRunning, OpRetryAgentResponse, OpRetryAgentWaitCmd, OpRetryAgentPollAicpuStop, OpRetryAgentRetryFail
- Server 侧: OpRetryServerRunning, OpRetryServerIssueCmd, OpRetryServerWaitResp

状态转移表: `RETRY_AGENT_RESP_STATE_LABEL` (map<RetryState, RetryState>)

#### 5. 平台层基类

TransportBase (`platform/resource/transport/transport_base_pub.h:52`) — 传输层基类
NotifyBase (`platform/resource/notify/notify_base.h:71`) — 设备通知基类
CcuRepBase (`legacy/unified_platform/ccu/ccu_representation/reps/common/ccu_rep_base.h:40`) — CCU 指令表示基类
- 子类: CcuRepRead, CcuRepWrite, CcuRepBufReduce, CcuRepLoopGroup, CcuRepLoop, CcuRepLocRecordEvent

TopoInfoExchangeBase (`framework/common/src/topo/topoinfo_exchange_base.h:48`) — 拓扑信息交换
- 子类: TopoInfoExchangeServer, TopoInfoExchangeAgent

### 三、设计模式汇总

| 模式 | 位置 | 实例 |
|-----|------|------|
| Template Method | 两仓核心 | ExecutorBase, AlgTemplateBase, CommBase, CollNativeExecutorBase |
| Strategy | hccl selector/topo | AutoSelectorBase(按设备分策略), TopoMatchBase(按拓扑分策略) |
| Registry | hccl selector | SelectorRegistry + REGISTER_SELECTOR 宏 |
| State | hcomm retry | OpRetryBase + Agent/Server 状态子类 |
| Dependency Injection | hcomm executor | CollExecutorBase(TopoMatcher), CollNativeExecutorBase |
| Factory | 两仓 | 大量 Create*/Build* 函数 |

### 四、v1 vs v2 架构并存

hccl 中存在两代执行框架:
- v1: ExecutorBase + AlgTemplateBase — 旧框架，RunAsync 驱动
- v2: InsCollAlgBase + InsAlgTemplateBase — 新框架，指令级(Instruction-based)执行
- 两代框架并存，v2 子类数量(50+)已远超 v1

### 五、资源管理模式

- ExecMem struct: 封装执行所需内存(输入/输出/暂存)
- PrepareData struct: 算法准备阶段参数打包
- RAII: unique_ptr 管理 TopoMatcher, shared_ptr 管理 Transport
