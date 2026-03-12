# HCCL缺陷提交深度分析

## Batch 1: Commit #1-15 (2025-12-27 ~ 2026-01-19)

### 604667ea 解决MC2多流场景问题
- 根因类别：并发问题
- 涉及文件：src/framework/device/aicpu_kfc/framework/aicpu_kfc_process.cc
- 缺陷描述：`g_expectPrepareId`是一个全局静态数组，在MC2多流（多线程）场景下被不同线程共享读写，导致数据竞争。同时，`SetExpectPrepareId(0, 0)`的重置调用放在了`KfcClearMsgArea`这个通用清理路径中，而非MC2特有的suspend退出路径，导致suspend场景下prepareId没有被正确重置，后续消息序号校验可能出错。
- 修复模式：将`g_expectPrepareId`从`static`改为`static thread_local`，每线程独立副本消除竞争。将重置调用移到`AicpuRunRpcServerForMC2V2`和`AicpuRunRpcServerForMC2`的suspend退出分支中。
- 可审查性：中
- 审查规则建议：检查全局/静态变量在多线程环境下是否需要`thread_local`；审查共享可变状态的写入点是否有同步机制；状态重置操作应与语义场景对齐，避免放在通用清理路径中。

### 35e32c7c fix log issue
- 根因类别：日志与调试
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/device/debug/dfx/profiling/profiling_manager_device.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：`aicpu_communicator.cc`中`HCCL_ERROR("[%s] failed. ret = [%d]", ret)`格式化字符串用`%s`但传入`int`类型的`ret`，缺少`__func__`参数，参数不匹配导致未定义行为（可能崩溃或输出乱码）。另一处profiling的memcpy失败日志缺少缓冲区大小信息。
- 修复模式：补上`__func__`参数使`%s`正确输出函数名；增加`sizeof_data[%zu]`字段输出缓冲区大小。
- 可审查性：高
- 审查规则建议：启用`-Wformat`警告检查printf风格格式化参数匹配；审查日志宏中格式占位符与实际参数的对应关系；错误日志应包含足够上下文（缓冲区大小、函数名等）。

### 4e82ec25 HcomGetandClearOverFlowTasks bugfix
- 根因类别：算法正确性（API设计缺陷）
- 涉及文件：pkg_inc/hccl/hcom.h, src/framework/hcom/hcom.cc
- 缺陷描述：`HcomGetandClearOverFlowTasks`的输出参数`hcclDumpInfo`和`len`被设计为值传递而非指针传递，函数无法将结果返回给调用者，完全颠倒了"获取"接口的输入/输出语义。
- 修复模式：将签名改为二级指针`HcclDumpInfo **hcclDumpInfoPtr`和`s32 *len`，通过输出参数返回数据。注意：修复后局部vector在返回后析构，`data()`指针悬空，引入了新的use-after-free风险。
- 可审查性：高
- 审查规则建议：API设计审查确认参数输入/输出方向与实际使用一致；检查函数返回的指针是否指向局部变量/临时对象；返回容器内部指针时确保容器生命周期覆盖指针使用期。

### 0947b660 fix NsRecovery
- 根因类别：并发问题（内存序）
- 涉及文件：src/platform/resource/mem/hdc.cc
- 缺陷描述：HDC的`Write`方法中先写数据后更新tail计数器，缺少内存屏障，CPU/编译器可能重排序store操作，导致对端读到不完整数据。NsRecovery场景下可能触发通信异常。
- 修复模式：在memcpy_s和tail更新之间插入`std::atomic_thread_fence(std::memory_order_seq_cst)`防止store-store重排序。
- 可审查性：低
- 审查规则建议：共享内存通信代码（生产者-消费者模式）必须审查内存屏障；标记使用普通变量进行跨核/跨设备通信的代码为高风险；审查"先写数据、后更新标志位"模式是否有memory fence。

### 7ff92abc [Fix] Add HCCL_ENV
- 根因类别：日志与调试
- 涉及文件：src/framework/common/src/config/env_config.cc, src/platform/common/externalinput.cc
- 缺陷描述：大量环境变量解析函数的日志输出缺少`[HCCL_ENV]`前缀标识，无法快速过滤定位环境变量相关日志。涉及数十个环境变量，约60余处日志调用点。
- 修复模式：统一添加`[HCCL_ENV]`前缀。纯日志规范化改动。
- 可审查性：高
- 审查规则建议：建立日志规范要求特定模块的日志包含统一模块标签前缀；通过lint规则检查。

### bb681c5c fix reducescatter small count deter ffts graph bug
- 根因类别：算法正确性（继承体系中逻辑不一致）
- 涉及文件：src/algorithm/impl/coll_executor/coll_reduce_scatter/（多个executor文件）
- 缺陷描述：`preloadCopyOpt`条件在基类和small count deterministic子类中以内联表达式硬编码，逻辑不一致——基类额外检查了`!DMAReduceFlag_`，子类没有。FFTS图模式下子类走到错误的优化路径导致数据拷贝方式错误。
- 修复模式：将判断逻辑提取为基类virtual方法`IsPreloadCopyOptimizeCondition()`，子类override提供定制逻辑，避免散落的条件判断不一致。
- 可审查性：中
- 审查规则建议：同一业务逻辑条件在继承体系多个类中出现时检查一致性；可被子类覆盖的策略判断应提取为virtual方法。

### 6784944a fix hcom dumpinfo bug
- 根因类别：内存管理（use-after-free）
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：`HcomGetandClearOverFlowTasks`将局部`std::vector`的`data()`指针通过输出参数返回，函数返回后vector销毁，调用方持有悬空指针。这是commit 4e82ec25引入的新缺陷。
- 修复模式：vector非空时通过`malloc`分配堆内存，`memcpy_s`拷贝数据，返回堆内存指针。调用方需负责`free`。
- 可审查性：高
- 审查规则建议：禁止将局部容器的内部数据指针直接暴露给外部；静态分析可标记"将局部对象的.data()赋值给输出指针参数"的模式。

### d5cce87e OOM Patch
- 根因类别：错误处理（OOM未区分处理）
- 涉及文件：src/platform/common/adapter/adapter_rts.cc
- 缺陷描述：`hrtMalloc`对`ACL_ERROR_RT_MEMORY_ALLOCATION`(OOM)没有专门处理，与其他错误混用通用错误码`EI0007`，无法让用户明确识别OOM，上层也无法做针对性处理。
- 修复模式：新增OOM专用检查，上报`EI0011`错误码并返回`HCCL_E_OOM`，输出分配大小等诊断信息。
- 可审查性：中
- 审查规则建议：资源分配类API应区分"资源不足"和"其他失败"；OOM等关键错误需有专用错误码和诊断信息。

### f2f1c83e fix version check and cmake release flags
- 根因类别：配置与兼容性
- 涉及文件：CMakeLists.txt, scripts/package/common/sh/version_compatiable.inc, version.info
- 缺陷描述：两个问题：(1) CMake Release模式默认注入`-O3 -DNDEBUG`与项目自定义选项冲突；(2) 版本兼容性检查用`>=8.5.0`而非精确匹配，不匹配时ERROR+return 1阻断安装，应为WARNING且不阻断。
- 修复模式：清空`CMAKE_C/CXX_FLAGS_RELEASE`；将ERROR降为WARNING并删除`return 1`；版本约束改为精确匹配`8.5.0`。
- 可审查性：中
- 审查规则建议：CMake审查应关注`CMAKE_*_FLAGS_RELEASE`等内置变量管理；版本检查脚本确认约束策略和失败行为是否符合产品需求。

### c5443da0 fix aclgraph launch in order: fixing the issue of exhausted stream resources
- 根因类别：资源生命周期（stream资源泄漏）
- 涉及文件：src/framework/common/src/order_launch/order_launch.cc, order_launch.h, hccl_communicator_host.cc
- 缺陷描述：`OrderLaunch`为每个model分配独立stream，stream数量随model增长无限增加但不复用，最终耗尽硬件stream资源。另外`AclgraphLaunchInOrder`开头直接`return HCCL_SUCCESS`（调试遗留），整个顺序启动逻辑被跳过。析构函数未释放event资源。
- 修复模式：将per-model的stream map替换为全局唯一stream；拆分为`ToOrderStream`和`ToKernelStream`两个方法用event做同步闭环；新增`DestoryRes()`统一管理资源释放；移除调试遗留的提前return和条件编译。
- 可审查性：中
- 审查规则建议：硬件资源管理检查：(1)创建是否有对应销毁路径；(2)资源数量是否随输入规模无限增长；(3)是否存在提前return导致逻辑跳过的死代码；(4)条件编译是否导致关键逻辑缺失。

### 891665ac Fix issue of missing profiling info
- 根因类别：日志与调试（快速路径功能缺失）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc, src/platform/common/unfold_cache/op_unfold_cache_entry.cc/h, src/platform/task/dispatcher_aicpu.cc/h
- 缺陷描述：SQE缓存命中走快速路径时缺少profiling信息采集和上报，`LaunchNewTask`在缓存命中场景不传递profiling状态也不记录SQE时间戳，导致profiling工具无法获取缓存命中路径的性能数据，与cache miss路径不对等。
- 修复模式：缓存命中入口增加`InvokeKfcHandler(kSetProfTimeStart)`调用；将`profL1Enable`参数贯穿整条调用链；对每个SQE记录timestamp；新增ring buffer上报逻辑。
- 可审查性：中
- 审查规则建议：所有快速路径(cache hit/fast path)必须与标准路径保持功能对等，特别是可观测性相关的profiling/tracing/logging调用。新增快速路径时需逐一对照标准路径的辅助功能。

### 112766de evb log fix
- 根因类别：日志与调试
- 涉及文件：src/framework/common/src/config/env_config.cc
- 缺陷描述：`ParseDFSConfig`函数的日志前缀仅为`[Parse]`，缺少`[HCCL_ENV]`模块级前缀，影响日志检索效率。
- 修复模式：将`[Parse]`改为`[HCCL_ENV][Parse]`。
- 可审查性：低
- 审查规则建议：日志前缀命名规范要求二级前缀格式`[模块名][阶段]`。

### 6baf33c4 kernel launch timeout motify
- 根因类别：算法正确性（整数溢出/截断）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/communicator/impl/one_sided_service/hccl_one_sided_service.cc
- 缺陷描述：计算kernel launch timeout时用`static_cast<u16>(notifyWaitTime + AICPU_KERNEL_TIMEOUT_INC)`，当和超过65535时u16截断溢出，timeout变为极小值导致过早超时。4处kernel launch和1处one-sided service调用均受影响。
- 修复模式：引入`MAX_VALUE_U16 = 0xFFFF`常量做饱和截断：超过MAX则饱和到MAX，否则正常相加。
- 可审查性：高
- 审查规则建议：所有`static_cast`从宽类型向窄类型的转换必须检查溢出/截断风险；静态分析标记所有narrowing cast；加法后窄化的表达式要求饱和逻辑。

### f1e1d52a fix HcomSelectAlg bug
- 根因类别：算法正确性（API缺陷+空指针）
- 涉及文件：pkg_inc/hccl/hcom.h, src/algorithm/impl/operator/(多文件), src/framework/communicator/(多文件), src/framework/hcom/hcom.cc
- 缺陷描述：多个问题：(1)API缺少`counts`参数，变长算子无法传入count信息；(2)AlltoAll走特殊分支绕过通用`SelectAlg`流程无法获得AIV编译信息；(3)`IsBufferSatisfyAlltoAllAivCondition`中alltoall场景对空指针`sendCountMatrix`解引用；(4)algName为空时仍尝试查找executor导致失败。
- 修复模式：全链路增加`void* counts`参数；移除AlltoAll特殊分支统一走SelectAlg；优先用`sendCount`字段避免空指针；algName为空时用兜底executor。
- 可审查性：中
- 审查规则建议：API签名变更检查所有调用点是否同步更新；解引用指针前必须检查非空；审查"提前return"的特殊路径是否遗漏通用路径中的必要逻辑。

### 3ec7410b fix p2p OOM
- 根因类别：内存管理（IPC资源泄漏）
- 涉及文件：src/platform/resource/transport/host/transport_p2p.cc, transport_p2p_pub.h
- 缺陷描述：P2P传输中input/output为CCL buffer子区间时仍单独注册IPC内存映射，同一块物理内存被重复注册多次，大规模通信下IPC资源耗尽导致OOM。析构时也重复close/destroy。
- 修复模式：新增`isMemInclude_`标志检测子区间包含关系；包含时复用父区间的IPC映射通过offset计算地址；析构时跳过子区间的IPC释放；调整序列化顺序确保依赖可用。
- 可审查性：中
- 审查规则建议：IPC共享内存注册检查是否重复注册同一块内存，建议注册层面增加去重；析构与构造路径一一对应；内存区间包含场景检查子区间复用可能性。

## Batch 2: Commit #16-30 (2026-01-19 ~ 2026-02-03)

### 9c1f957b fix workspace calc scratch mem size overflow
- 根因类别：整数溢出/截断
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：`GetOpScratchMemSize`中，`hcomOpParam->count`类型为u64，但被赋值给局部变量`u32 count`，高32位被截断。后续计算`opMemSize = count * dataTypeSize * rankSize`全部在32位范围内进行，三个32位因子相乘极易溢出回绕为一个错误的小值，导致申请的scratch memory远小于实际需要，后续写入时产生buffer overflow。
- 修复模式：将局部变量`count`类型从u32改为u64，使后续乘法提升到64位运算。
- 可审查性：高
- 审查规则建议：结构体字段为u64时，接收其值的局部变量禁止隐式截断为u32；涉及内存大小计算的多因子乘法必须使用u64；开启`-Wconversion`并在CI中设为error。

### e1880dc1 fix bug of condition check
- 根因类别：算法正确性（恒假条件/copy-paste错误）
- 涉及文件：src/platform/task/dispatcher_aicpu.cc
- 缺陷描述：`DispatcherAiCpu::MemcpyRtsq`中边界检查条件写成`sqeContextBuffer->tailSqeIdx > sqeContextBuffer->tailSqeIdx`（同一变量自比较），恒为false，`CHK_PRT_RET`永远不触发。原意是检查`tailSqeIdx > HCCL_SQE_MAX_CNT`防止ring buffer越界。条件恒假时，当tailSqeIdx异常超过MAX，`HCCL_SQE_MAX_CNT - tailSqeIdx`在无符号类型下回绕为极大值，导致memcpy越界。错误信息字符串中已正确写了`tailSqeIdx > HCCL_SQE_MAX_CNT`，但条件表达式把`HCCL_SQE_MAX_CNT`误写为`sqeContextBuffer->tailSqeIdx`。
- 修复模式：将条件修正为`sqeContextBuffer->tailSqeIdx > HCCL_SQE_MAX_CNT`。
- 可审查性：高
- 审查规则建议：`x op x`（变量自比较）必须标记为缺陷；`CHK_PRT_RET`/`CHK_RET`的条件表达式须与错误消息字符串交叉验证一致性；集成`-Wtautological-compare`并设为error。

### 1d171a92 fix p2p transport
- 根因类别：资源生命周期（变量遗漏赋值）
- 涉及文件：src/platform/resource/transport/host/transport_p2p.cc
- 缺陷描述：`TransportP2p::FillExchangeDataTotalSize`中局部变量`ipcMemDataSize`初始化为0。进程内（Intra Proc）分支正确赋值为`sizeof(u64) + sizeof(u64)`，但跨进程（Inter Proc）分支遗漏赋值——跨进程需额外传输IPC name（65字节）加size和offset。`ipcMemDataSize`保持为0导致`exchangeInfoSize_.ipcMenSize`计算为0，IPC交换数据缓冲区分配大小不足，P2P通信失败。
- 修复模式：在Inter Proc分支开头添加`ipcMemDataSize = HCCL_IPC_MEM_NAME_LEN + sizeof(u64) + sizeof(u64)`。
- 可审查性：中
- 审查规则建议：if/else分支中使用的变量，检查每个分支是否都做了正确赋值；数据交换大小计算的跨进程/进程内路径应有注释说明各字段及其大小。

### 9845f06f fix step recalculation
- 根因类别：算法正确性/配置与兼容性
- 涉及文件：src/framework/cluster_maintenance/recovery/operator_retry/opretry_agent.cc, opretry_base.h, opretry_manager.cc/h, opretry_server.cc, hccl_communicator_host.cc
- 缺陷描述：operator retry状态机两个子问题：(1) 超时硬编码`OP_RETRY_SWITCH_WAIT_RESUM = 10`秒，但实际链路建连超时由`GetExternalInputHcclLinkTimeOut()`配置控制，硬编码10秒可能不够导致retry状态机提前超时退出；(2) 无备份链路（backup link）场景时，状态机仍进入等待恢复流程空等命令最终超时失败，应直接回到RUNNING状态。
- 修复模式：删除硬编码常量改用动态配置超时；新增`haveCommEnableBackupLink_`标志和全局原子计数器`g_enableBackupLinkCommCount`跟踪备份链路状态；无备份链路时直接创建RUNNING状态。
- 可审查性：低
- 审查规则建议：超时时间不应硬编码，应从统一配置源获取；状态机每个状态需明确列出所有前置条件和边界场景；全局引用计数器的加减操作必须与同一条件判断配对，建议RAII封装。

### 9476c6df fix one sided init bugs
- 根因类别：错误处理（调用顺序不当）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：`CheckOneSidedBackupAndSetDevId`中，`hrtGetDeviceIndexByPhyId(backupDevPhyId, backupDevLogicId)`被无条件调用，放在了判断`isOneSidedTaskAndBackupInitA3`之前。当设备不支持pair device或backup device不存在时，该调用返回错误导致`CHK_RET`使整个函数提前失败，即使当前通信域根本不需要one-sided backup功能。
- 修复模式：将`hrtGetDeviceIndexByPhyId`调用移到`if (isOneSidedTaskAndBackupInitA3)`条件内部，按需获取。
- 可审查性：中
- 审查规则建议：带`CHK_RET`的函数调用应延迟到确认需要其返回值处；条件性功能的资源获取操作应统一放在条件为true的代码块内。

### b2e74aee fix p2p OOM
- 根因类别：内存管理（遗漏初始化调用）
- 涉及文件：src/platform/resource/transport/host/transport_p2p.cc
- 缺陷描述：`TransportP2p::Init()`缺失对`SetMemIncludeFlag()`的调用。该函数检查input/outputMem是否包含在CCLbuf整块内存中，是则设置`isMemInclude_ = true`。由于`isMemInclude_`默认false，即使内存已在CCLbuf范围内仍走`!isMemInclude_`分支，额外创建IPC内存映射。大量P2P transport反复创建时冗余IPC映射累积导致OOM。
- 修复模式：在`CheckExchangeData()`之后调用`SetMemIncludeFlag()`。
- 可审查性：中
- 审查规则建议：bool成员变量默认false且在多处以`!flag`守护资源分配时，检查所有初始化路径是否都有flag设置逻辑；建立"flag-setting completeness"检查。

### a6e4d199 [Fix] Fix SetDispatcherCtx
- 根因类别：并发问题（thread_local变量初始化时机错误）
- 涉及文件：src/framework/communicator/impl/independent_op/data_api/hccl_api_data.cc, src/framework/device/framework/aicpu_communicator.cc/h, src/platform/comm_primitive/hccl_dispatcher_ctx.cc
- 缺陷描述：`gDispatcherCtx`是thread_local变量。原代码在`RegisterOpInfo()`中调用`SetDispatcherCtx()`，但`HcommAcquireComm`获取通信域时当前线程的`gDispatcherCtx`可能尚未设置。新线程调用`HcommAcquireComm`后通过`GetDispatcherCtx`获取dispatcher失败或拿到错误上下文。
- 修复模式：将`SetDispatcherCtx`从`RegisterOpInfo()`移除，新增`SetDispatcherCtxOnThread()`方法在`HcommAcquireComm`流程中立即调用。
- 可审查性：中
- 审查规则建议：thread_local变量需建立"thread-entry-point audit"——在每个线程入口点检查所有必需的thread_local变量是否已设置；设置逻辑不应嵌在业务函数深处。

### 36994739 fix dispatcher and queue notify core
- 根因类别：并发问题（double-free/use-after-free）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/platform/comm_primitive/hccl_dispatcher_ctx.cc, src/platform/resource/dispatcher_ctx/dispatcher_ctx.cc/h
- 缺陷描述：三个并发问题：(1) `DispatcherCtx::Destroy()`对`dispatcher_`的delete无锁保护，多线程同时销毁导致double-free；(2) `DestroyDispatcherCtx`全局函数对全局map`g_ctx`的操作无互斥保护；(3) `queueNotifyManager_`智能指针在销毁流程中未及时置空，网络资源销毁后仍被引用导致use-after-free。
- 修复模式：`Destroy()`增加`destroyMutex_`保护；`DestroyDispatcherCtx`增加static mutex；`queueNotifyManager_`在资源释放后及时置nullptr。
- 可审查性：中
- 审查规则建议：delete后置nullptr模式若存在并发访问必须加锁；智能指针成员在销毁函数中应按依赖逆序显式置空；shared_ptr多持有者场景需明确销毁顺序。

### 43dab3e2 fix aiv bug
- 根因类别：资源生命周期（缓存中stream引用过期）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：`ExecOpCache`使用缓存的`HcclCacheInfo`复用执行参数，但`cacheInfo.resourceArgs.stream`未随每次调用刷新。stream是可变的硬件资源句柄，不同调用可能使用不同stream。cache中保留过期stream指针导致kernel下发到错误队列或访问已释放资源。`buffersIn`和`buffersOut`已有刷新逻辑，唯独stream被遗漏。
- 修复模式：在`ExecOpCache`入口添加`cacheInfo.resourceArgs.stream = opParam.stream.ptr()`刷新stream。
- 可审查性：高
- 审查规则建议：缓存/复用模式需建立"staleness checklist"——逐字段确认cache结构体中哪些字段可能变化、复用前是否有刷新逻辑；句柄类字段（stream、context、device handle）几乎总需要刷新。

### 1535b1c4 读写锁改为基于原子变量的实现，减少冲突
- 根因类别：并发问题（读写锁性能瓶颈和死锁风险）
- 涉及文件：src/framework/common/src/read_write_lock_base.cc/h, src/framework/device/debug/dfx/trace/executor_tracer.cc
- 缺陷描述：原始读写锁基于mutex+condition_variable，两个问题：(1) 高频读多写少场景下内核态切换开销大，condition_variable的wait/notify涉及系统调用；(2) `HandleDestroyComm`在循环外获取写锁、循环内逐个销毁通信域，整个循环持锁阻塞所有读写者，若销毁过程需要读锁则死锁。
- 修复模式：重写为基于`std::atomic<uint64_t>`的无锁实现，高两位标识写者状态，低62位为读者计数。新增`tryReadLock()`/`fastReadUnlock()`接口。`HandleDestroyComm`写锁粒度缩小到循环内。潜在风险：多写者竞争时WAITING_BIT被提前清除可能破坏写者优先语义；忙等待无退避策略。
- 可审查性：低
- 审查规则建议：无锁数据结构变更需至少两名并发经验丰富reviewer交叉审查并附压力测试结果；CAS循环需枚举所有失败原因确认重试正确性；位操作状态机需绘制状态转移图；自旋锁需评估最坏CPU占用。

### 67b4ad3f [Fix] fix compile
- 根因类别：配置与兼容性
- 涉及文件：src/CMakeLists.txt, src/framework/CMakeLists.txt, src/platform/CMakeLists.txt
- 缺陷描述：仓库拆分（hcomm -> hcomm + hcomm-legacy）后，CMakeLists.txt中`target_include_directories`仍引用旧路径`platform/legacy/inc`（已迁移到hcomm-legacy仓库），编译时找不到头文件。3个CMakeLists.txt共4处include路径错误。
- 修复模式：将所有`platform/legacy/inc`路径改为`${TOP_DIR}/hcomm-legacy/src/platform/legacy/inc`。
- 可审查性：高
- 审查规则建议：仓库拆分/目录迁移时，必须全局搜索所有构建脚本中对迁移目录的引用确保零遗漏；CI中增加对不存在include路径的检测。

### 994390df 修复Step快恢失败问题
- 根因类别：并发问题/错误处理
- 涉及文件：src/framework/cluster_maintenance/recovery/operator_retry/opretry_manager.cc, src/framework/communicator/hccl_comm_host.cc, src/framework/communicator/impl/hccl_communicator.cc
- 缺陷描述：三个问题：(1) 状态机初始化缺失——`RegisterAgentRetryMachine`/`RegisterServerRetryMachine`创建`RetryContext`后设置`startExec = true`但未调用`SetRetryState()`初始化重试状态，状态机在"已启动但状态未定义"下运行；(2) `CheckExitWaitResumeState`被无条件调用，但只有AICPU Unfold/AICPU通信引擎场景才应执行，非AICPU场景误入导致异常；(3) 关键路径日志级别不足（HCCL_INFO应为HCCL_RUN_INFO），超时日志不含当前state信息。
- 修复模式：注册时立即设置初始状态；添加AICPU场景条件守卫；增强日志级别和内容。
- 可审查性：中
- 审查规则建议：状态机对象创建后必须在同一代码块内显式设置初始状态，禁止依赖默认值；涉及多执行模式的恢复路径需逐一确认每个模式下调用的合理性；错误日志必须包含足够上下文。

### 91fbd1d6 a3 aiv bugfix
- 根因类别：配置与兼容性/算法正确性
- 涉及文件：src/algorithm/impl/hccl_aiv.cc, src/algorithm/impl/operator/all_gather_v_operator.cc, broadcast_operator.cc, reduce_scatter_v_operator.cc，及被删除的5个crossnode头文件和3个executor实现文件（约1700行删除）
- 缺陷描述：910_93（A3）芯片的AIV跨节点通信实现整体不可用。三个集合通信算子（AllGatherV、Broadcast、ReduceScatterV）的跨节点AIV executor存在严重bug。修复采用完整回退策略：删除所有crossnode相关的executor实现、kernel注册、参数类型定义，跨节点场景回退到成熟的ring/double-ring算法。策略是"功能不成熟就不暴露"。
- 修复模式：删除crossnode AIV路径（约1700行），算子选择中移除跨节点分支，跨节点降级到传统算法。
- 可审查性：低
- 审查规则建议：新增硬件相关通信路径（尤其跨节点）必须有多节点集成测试覆盖；算子选择新增分支时须评估fallback路径可用性；同一功能不应有两套参数格式。

### 6b354394 修复借轨条件判断&重执行超时时间
- 根因类别：并发问题/算法正确性
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/device/aicpu_kfc/framework/aicpu_kfc_deprecated_process.cc, aicpu_kfc_retry_process.cc, src/framework/device/framework/aicpu_communicator.cc/h
- 缺陷描述：四个独立问题：(1) `IsEnableBackupLink()`遗漏`retryEnable_`核心条件，retry关闭时仍认为backup link可用并执行借轨；(2) `g_enableBackupLinkCommCount`原子变量用`==`比较而非`.load()`显式原子读取；(3) `HcclGetWaitStopExecCmdTimeout()`是static方法使用全局配置，但多实例需要不同超时；(4) `identifier_`（std::string）直接作为`%s`参数传入缺少`.c_str()`。
- 修复模式：IsEnableBackupLink增加retryEnable_检查；原子变量改用.load()；超时方法从static改为成员方法使用实例的linkTimeOut_；修复格式化输出。
- 可审查性：中
- 审查规则建议：功能使能判断函数修改时须对照设计文档检查所有前置条件完备性；atomic变量所有读取点必须用.load()；std::string传入C格式化函数须用.c_str()，启用`-Wformat`。

### f8d4b8e9 fix orion numblocks
- 根因类别：算法正确性（API参数语义不匹配）
- 涉及文件：涉及73个文件，核心包括src/legacy/framework/aiv/aiv_ins/hccl_aiv_utils.h/cc, src/legacy/framework/communicator/communicator_impl.cc/h, src/legacy/framework/device_mode/coll_service_device_mode.cpp, src/legacy/service/collective/coll_operator.h等
- 缺陷描述：`aclrtLaunchKernelWithHostArgs`第二参数语义是"numBlocks"（block数量），但代码中从结构体字段到函数参数全部使用"blockDim"命名。在Orion平台上，blockDim（每block线程数维度）与numBlocks（block数量）可能不同，语义不匹配导致实际启动的block数量不正确。此前在某些平台上两值恰好相等，问题被掩盖。
- 修复模式：横跨73个文件的全局语义重命名：blockDim->numBlocks, CalcBlockDim->CalcNumBlocks, MAX_BLOCK_DIM->MAX_NUM_BLOCKS, blockDimLimit->numBlocksLimit。
- 可审查性：中
- 审查规则建议：调用外部runtime API时参数命名须与API文档一致；涉及kernel launch参数的变量定义处须注释说明语义；同一概念在代码中存在多个别名时首次review应要求统一。

## Batch 3: Commit #31-45 (2026-02-03 ~ 2026-02-06)

### e025b6c5 修复重执行约束故障上报
- 根因类别：错误处理（错误码语义不精确+异常上报遗漏）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：AICPU重执行(OpRetry)流程中，约束违反场景存在三个问题：(1)错误码使用`HCCL_E_INTERNAL`而非专用的`HCCL_E_OPRETRY_FAIL`，KFC错误码使用`KfcError::kExec`而非`KfcError::kExecConstraint`，上层无法区分"执行错误"和"重试约束不满足"；(2)约束不满足时未调用`SendTaskExceptionByMBox(TS_ERROR_RETRY_CONSTRAINT)`向TS上报故障；(3)`HcclOpExecFsmStoppedProcess`中两个分支设置了`errorCode`和`fsmState`后缺少`return`语句，fallthrough导致状态不一致。
- 修复模式：将约束违反场景错误码改为`HCCL_E_OPRETRY_FAIL`/`KfcError::kExecConstraint`；增加`SendTaskExceptionByMBox`调用；补充`return`语句防止fallthrough。
- 可审查性：中
- 审查规则建议：错误码语义匹配检查——当引入新错误分类时确认所有路径使用正确错误码；异常上报完整性——所有需通知上层组件的错误路径应包含上报调用；if-else分支设置错误状态后检查是否遗漏return。

### b0e8318b 问题单修改
- 根因类别：其他（安全加固+输入校验细化+冗余清理）
- 涉及文件：src/legacy/framework/fault_recovery/snap_shot_parse.cc, src/legacy/framework/topo/new_topo_builder/rank_table_info/address_info.cc, src/legacy/framework/topo/new_topo_builder/rank_table_info/control_plane.cc, src/legacy/unified_platform/aiv/ub_memory_transport.cc/h
- 缺陷描述：四个子问题：(1)日志信息不精确——`SerializeCommConfigInfo`中打印`hcclBufferSize`缺少单位"MB"；(2)格式化字符串安全隐患——`AddressInfo::Deserialize`和`ControlPlane::Deserialize`中用`%s`直接打印超长address，改为`%.*s`配合`MAX_DISPLAY_LEN(128)`截断；(3)输入校验不精细——`EidToAddr`仅做格式校验无法区分"长度错误"和"格式错误"，修复后先检查长度再检查格式分别提示；(4)冗余成员清理——`UbMemoryTransport`中4个未使用的`char`数组被移除。
- 修复模式：使用`%.*s`精度限定符截断输出；拆分校验为长度+格式两步；删除dead code。
- 可审查性：高
- 审查规则建议：用户可控字符串通过`%s`格式化输出时必须限制长度（用`%.*s`或预先截断）；输入校验异常信息应区分具体的非法原因；日志涉及度量字段应附带单位。

### 8959e766 fix allreduce selector
- 根因类别：算法正确性（参数未传递+算法选择条件缺失）
- 涉及文件：src/legacy/service/collective/alg/interface/host/coll_alg_component.cc, src/legacy/service/collective/alg/selector/all_reduce_auto_selector.cc
- 缺陷描述：两个问题：(1)`CollAlgComponent`构造`ExecuteSelector`时只调用了`SetVirtualTopo(rankGraph)`未调用`SetRankSize(rankSize_)`，selector内部`rankSize_`保持默认值，后续算法选择中计算per-rank数据量出错；(2)MESH_1D场景无条件选择`CcuAllReduceMesh1DOneShot`，未考虑`dataSize_ / rankSize_`超过`AR_ONESHOT_1D_MAX_DATA_SIZE`(16KB)时OneShot不适用，应选普通`CcuAllReduceMesh1D`。
- 修复模式：构造链增加`.SetRankSize(rankSize_)`；MESH_1D分支增加数据量阈值判断。
- 可审查性：中
- 审查规则建议：Builder/fluent API构造对象时确认所有必要setter都被调用；算法选择逻辑应考虑数据规模边界条件；除法运算前确保除数非零。

### e3ea4ed1 ccu 2die sync fix
- 根因类别：资源生命周期/算法正确性（初始化时序+硬件拓扑资源归属错误）
- 涉及文件：src/legacy/unified_platform/ccu/ccu_device/ccu_component/ccu_component.cpp/h
- 缺陷描述：CCU双die同步场景四个问题：(1)初始化时序——`Init()`中`ChooseLoopEids()`依赖`dieEnableFlags`但在其被设置之前调用；(2)die使能标志——驱动返回使能不等于实际可用，`FindOneUsableEid`失败时仍标记为可用；(3)`ConfigLoopChannel`中用`ccuRmaBufferMap.find(dieId)`查本die的buffer，但双die同步需配对端die的buffer，应为`find(1 - dieId)`；(4)`CleanDieCkes`中使用`devLogicId`而非`devPhyId`构造`HRaInfo`。
- 修复模式：将`ChooseLoopEids`拆为per-die调用融入`CheckDiesEnable`循环；只在`ChooseLoopEid`成功后标记die可用；RMA buffer查找改为对端die；修正为物理ID。
- 可审查性：低
- 审查规则建议：初始化依赖顺序——函数A依赖B设置的状态时确保B先执行；双die/多die场景配置通信通道时确认资源归属是本端还是对端；逻辑ID vs物理ID不可混用；驱动使能不等于实际可用，资源获取成功后才标记可用。

### af109f94 fix alltoall op with mc2 on aicpu unfold
- 根因类别：算法正确性（两阶段参数有效性未区分）
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc/h
- 缺陷描述：`ConvertCollOperatorA2A`通过`isLaunch`区分准备资源(`false`)和实际下发(`true`)两个阶段，但两阶段的DataDes赋值逻辑完全相同。MC2 compile阶段`opParams`中`sendCount/recvCount`等字段尚未填充有效值，导致准备阶段使用了无效数据。另外`currentCollOperator`为nullptr时仅打印日志不返回错误。
- 修复模式：拆为`DefaultConvertCollOperatorA2A`（准备，使用默认值）和`LaunchConvertCollOperatorA2A`（下发，使用实际值）；nullptr改为抛异常。
- 可审查性：中
- 审查规则建议：同一函数通过bool参数区分完全不同行为时应拆分为两个函数；两阶段调用确认每阶段使用的参数是否已就绪；核心对象为nullptr应抛异常或返回错误码，不能仅打印日志。

### 82989927 solve the hccp error
- 根因类别：资源生命周期（early return跳过必要初始化）
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc
- 缺陷描述：`TryInitCcuFeature()`中，当环境变量`HCCL_INDEPENDENT_OP`被设置时函数提前return，跳过了后续的`TpManager::GetInstance(devLogicId).Init()`调用。独立算子模式下TpManager从未被初始化，后续代码路径依赖其就绪状态导致hccp错误。
- 修复模式：在`HCCL_INDEPENDENT_OP`分支提前return之前插入`TpManager::GetInstance(devLogicId).Init()`。
- 可审查性：中
- 审查规则建议：函数包含多个early return时，审查每个return路径是否遗漏了后续必要的初始化/清理操作；建立checklist逐一确认每个early return点跳过了什么。

### 71fd0b86 fix aiv bug
- 根因类别：算法正确性（新增模式未同步更新同族校验）
- 涉及文件：src/algorithm/impl/coll_executor/coll_all_gather/coll_all_gather_mesh_aiv_executor.cc, coll_all_reduce/coll_all_reduce_mesh_aiv_executor.cc, coll_reduce_scatter/coll_reduce_scatter_mesh_aiv_executor.cc
- 缺陷描述：AIV核心数量校验逻辑存在遗漏：(1)AllGather和AllReduce的`CalNumBlocks`中只校验了`numBlocks_ < rankSize`，在`isOpBase`模式下还需确保`numBlocks_ >= bestNumBlocks`；(2)ReduceScatter的条件`numBlocks_ < bestNumBlocks && deviceType != DEV_TYPE_910_93`在910_93上短路跳过校验，但`isOpBase`模式下任何设备都需校验。
- 修复模式：AllGather/AllReduce新增`isOpBase && numBlocks_ < bestNumBlocks`校验；ReduceScatter条件改为`(... || isOpBase)`。
- 可审查性：中
- 审查规则建议：新增运行模式时搜索所有同族executor确认校验逻辑一致性；设备类型豁免条件在新模式下是否仍成立需逐一验证。

### 099fe2a8 resolved known issues
- 根因类别：配置与兼容性（设备类型分支逻辑不等价）
- 涉及文件：src/framework/cluster_maintenance/health/heartbeat/heartbeat.cc
- 缺陷描述：`GetSocketTypeIn91093`中心跳socket类型判断依赖`isInterServer`。910_93设备通过`rtGetServerIDBySDID`精确判断，其他设备仅比较`serverId`字符串。超节点模式下非910_93设备也需通过SDID精确判断，字符串比较不够准确导致选择错误的socket类型(VNIC vs ROCE)。
- 修复模式：移除设备类型分支，统一使用`rtGetServerIDBySDID`路径。
- 可审查性：中
- 审查规则建议：按设备类型分支处理同一逻辑时，审查各分支功能等价性；如果某分支使用更精确方法应考虑统一采用。

### 7bc9e850 fix mc2 precision
- 根因类别：算法正确性（结构体字段赋值遗漏）
- 涉及文件：src/legacy/framework/communicator/aicpu/aicpu_utils.cc
- 缺陷描述：`FillKernelParam`函数设置了`outputDataType`但遗漏了`dataType`（输入数据类型）的赋值。MC2场景下输入和输出数据类型可能不同（如输入fp16输出fp32），缺少`dataType`导致AICPU使用未初始化的数据类型计算，引发精度问题。
- 修复模式：新增`kernelParam_->op.algOperator.dataType = HcclDataTypeToDataType(data->dataType)`赋值和校验；日志增加`dataType`字段。
- 可审查性：高
- 审查规则建议：填充参数结构体时对比字段定义与赋值语句标记未赋值字段；成对字段（dataType/outputDataType、sendType/recvType）确保不遗漏其一。

### 29eb1736 fix hccd CUSTOM_INTERFACE
- 根因类别：配置与兼容性（编译宏条件不完整）
- 涉及文件：src/platform/hccp/rdma_service/CMakeLists.txt
- 缺陷描述：`CUSTOM_INTERFACE`宏定义条件仅检查`PRODUCT_SIDE==device`，但实际应在`USE_ALOG=0`且device侧时才生效。`USE_ALOG=1`时也定义该宏导致hccd使用错误接口实现。
- 修复模式：CMake generator expression条件从单一`PRODUCT_SIDE==device`改为`AND(USE_ALOG==0, PRODUCT_SIDE==device)`。
- 可审查性：中
- 审查规则建议：编译宏定义条件变更需审查与其他编译选项的交叉影响；搜索宏被使用的所有位置验证行为正确性。

### 1a840065 bugfix, move 2.0process before 1.0process
- 根因类别：算法正确性（多版本流程初始化顺序错误）
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：`HcclCommInitRootInfoInner`和`HcclCommInitRootInfoConfigInner`中，1.0流程的`InitExternalInput()`/`InitEnvConfig()`被放在`HCCLV2_FUNC_RUN`（2.0流程）之前执行。2.0流程通过条件编译保护并可能提前return，执行顺序错误导致2.0流程下初始化行为异常。
- 修复模式：将`InitExternalInput()`/`InitEnvConfig()`从`HCCLV2_FUNC_RUN`之前移到之后，确保2.0流程优先执行。
- 可审查性：中
- 审查规则建议：多版本兼容路径中检查初始化顺序是否符合版本优先级；条件编译块内外代码依赖关系需明确。

### 6a6eac0f fix log bug
- 根因类别：日志与调试（批量日志缺陷）
- 涉及文件：src/algorithm/base/communicator/calc_nhr_transport_req.cc, src/algorithm/impl/operator/（all_gather、all_reduce、alltoall、coll_alg、reduce_scatter等多个operator文件）
- 缺陷描述：六类日志问题：(1)字符串字面量用`\`续行将下一行前导空格纳入字符串，应改为相邻字面量拼接；(2)日志tag写成`[SelectAlgforA2]`但所在函数是`SelectAlgfor910B`，复制粘贴未更新；(3)`scratchMemSize`(u64)和`dataSizeLimit`(s64)用`%u`占位符，应为`%llu`/`%lld`；(4)`;;`双分号、`capacityof`缺空格；(5)`HCCL_ERROR`中占位符`%llu`但传入`multiModuleDiffDeviceNumMode_`(u32)而非`dataSize`；(6)过长日志需拆分。
- 修复模式：逐一修正续行方式、tag名称、格式化占位符、多余分号、传参对应关系、过长日志拆分。
- 可审查性：高
- 审查规则建议：禁止字符串字面量中用`\`续行，强制相邻字面量拼接；启用`-Wformat`检查格式化占位符与实参类型一致性；日志tag必须与所在函数名一致；`;;`双分号lint检测。

### 6b9278e4 fix bug of ccl addr update in graph mode
- 根因类别：资源生命周期（图模式下地址更新路径遗漏）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：图模式+强制单算子展开场景下两个问题：(1)Host端通过`aicpuCacheEnable`值+10编码"强制单算子模式"传递给device端；(2)Device端`PrepareUserMemRanges`中remote memory range更新只处理`PARAM_INPUT/PARAM_OUTPUT`类型，遗漏`CCL_INPUT/CCL_OUTPUT`类型。当transport使用CCL buffer时地址未被更新，图模式下CCL地址传递失败。
- 修复模式：Host端+10编码传递模式信息；Device端扩展条件判断和`TransportMemType`匹配范围，增加`CCL_INPUT/CCL_OUTPUT`分支。
- 可审查性：低
- 审查规则建议：枚举值switch/if匹配需覆盖所有有效值；跨host/device模式传递应有明确编码协议，避免magic number；新增执行模式时系统性审查所有依赖`WorkflowMode`判断的路径。

### d953cbf3 fix: resolve compilation errors for GCC 10+
- 根因类别：配置与兼容性（GCC版本兼容性）
- 涉及文件：src/platform/hccp/rdma_service/ctx/rs_ub.c
- 缺陷描述：GCC 10+对整数到枚举类型的隐式转换执行更严格检查。3处赋值缺少显式类型转换：`cfg->trans_mode`到`urma_transport_mode_t`、`rjetty_cb->policy`到`urma_jetty_grp_policy_t`、`rjetty_cb->type`到`urma_target_type_t`。
- 修复模式：添加C风格显式类型转换。
- 可审查性：高
- 审查规则建议：启用`-Werror=enum-conversion`；CI矩阵覆盖多GCC版本（至少最低和最新）；整数到枚举赋值必须显式转换。

### 96087ffb 修复重执行失败无约束打印
- 根因类别：错误处理（异常上报缺少条件守卫）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：`BSRStopedProcess`中，无论`ret`结果如何（重试成功或失败），都无条件调用`SendTaskExceptionByMBox(TS_ERROR_RETRY_CONSTRAINT)`发送异常报告。重试成功时也会错误触发异常打印和可能的误报。
- 修复模式：将`SendTaskExceptionByMBox`包裹在`if (ret != HCCL_SUCCESS)`条件中。
- 可审查性：中
- 审查规则建议：异常上报/错误通知操作必须受前置条件判断保护，不可无条件执行；三元表达式赋值后紧跟的操作审查是否应区分两个分支。

## Batch 4: Commit #46-60 (2026-02-07 ~ 2026-02-11)

### 43069da0 fix one sided bugs
- 根因类别：资源生命周期、结构体字段遗漏、配置与兼容性（多个子缺陷）
- 涉及文件：src/algorithm/impl/resource_manager/hccl_socket_manager.cc, src/framework/common/src/onesided_memory_management/global_mem_manager.cc, src/framework/common/src/onesided_memory_management/global_mem_manager.h, src/platform/common/adapter/adapter_hccp.cc, src/platform/inc/adapter/adapter_hccp.h, src/platform/resource/transport/onesided/transport_roce_mem.cc
- 缺陷描述：包含三个独立子缺陷：(1) `netDevCtxMap_`的key类型错误——原先map的key是`HcclIpAddress`，但同一IP可能对应不同端口的设备上下文，同IP不同端口调用`GetNetDevCtx`时会错误复用已有NetDevCtx。`Destroy()`中`ServerDeInit`硬编码`HETEROG_CCL_PORT`而非实际端口，且关闭顺序错误（先`HcclNetCloseDev`再`ServerDeInit`，但后者依赖存活的net device）。(2) `CreateQp`和`CreateAiQp`中创建one-sided QP时没有显式设置`sendCqDepth`，默认32768对one-sided场景过大。(3) `TransportRdmaWithType`中RDMA写操作对每个WQE都设置`RA_SEND_SIGNALED`标志，多WQE时中间WQE不应signaled，否则提前收到completion通知误认为传输完成。
- 修复模式：(1) key从`HcclIpAddress`改为`PortInfo`（含IP和端口），修复Destroy参数和销毁顺序。(2) 新增`DEFAULT_MAX_ONE_SIDED_SEND_CQ_DEPTH = 512`显式赋值。(3) signaled标志改为仅最后一个WQE（`remainingBytes <= MAX_RDMA_WQE_SIZE`时）设置。
- 可审查性：中
- 审查规则建议：map/set的key是自定义类型时审查key是否能唯一标识资源（IP+端口场景不可仅用IP）；资源销毁顺序须按依赖关系图反序；RDMA多WQE传输中只有最后一个应设`RA_SEND_SIGNALED`；QP创建时所有深度参数应显式设置。

### eb9be21b Debug Get Socket Timeout
- 根因类别：并发问题（单例初始化时序）
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc, src/legacy/framework/communicator/communicator_impl.h, src/legacy/unified_platform/common/hccp_peer_manager.cc
- 缺陷描述：`HccpPeerManager::Init()`中`RaSocketSetWhiteListStatus(1)`放在单例的一次性初始化路径中。由于引用计数机制，第二次调用`Init`直接走引用计数增加分支返回，不会执行白名单设置。某些communicator的PEER模式socket连接因白名单未启用而在socket获取阶段超时。
- 修复模式：将`RaSocketSetWhiteListStatus(1)`从`HccpPeerManager::Init()`移到`CommunicatorImpl::InitHccpPeer()`中，每次communicator初始化都会调用。
- 可审查性：低
- 审查规则建议：单例/引用计数管理器的`Init`方法中区分"只需执行一次"和"每次都需执行"的操作；socket超时时首先检查白名单/权限/认证前置条件；`Init/DeInit`审查关注引用计数>1时跳过的逻辑是否遗漏必要副作用。

### 3a9b61f2 fix ccu dfx
- 根因类别：日志与调试、结构体字段遗漏、算法正确性
- 涉及文件：15个CCU DFX模块文件
- 缺陷描述：大型DFX改进，包含：(1) `CcuErrorInfo`结构体`transMem`/`bufTransMem`缺`len`字段，`loop`缺`loopEngineId`，所有`GenErrorInfo*`函数未收集`len`。(2) `CcuBuffer`ID包含DieId编码（bit15），生成错误信息时直接使用原始ID导致显示值偏移。(3) 异常处理流程未区分当前异常指令和上下文指令。(4) 新增依赖追踪机制（signal ID -> mask -> 依赖Rep列表），用于`LOC_WAIT_SEM`异常精确定位未完成的源操作。(5) 新增channel到IP地址对映射。(6) Block资源ID偏移修复（加`0x1000`偏移分割ID空间）。
- 修复模式：结构体增字段、函数补采集、引入`GetMSIdPerDie`屏蔽DieId bit、新增依赖追踪API、重构异常处理主流程。
- 可审查性：低
- 审查规则建议：DFX错误信息必须包含所有问题定位相关字段；硬件相关ID使用前检查是否含编码位；资源ID空间不同类型不得重叠；异常信息应包含通信双端标识。

### ef766683 修复局部变量导致的功能错误
- 根因类别：算法正确性（变量名称遮蔽）
- 涉及文件：src/platform/common/hccl_ip_address.cc
- 缺陷描述：`HcclIpAddress`的EID构造函数中，`family = AF_INET6;`的赋值目标被同名局部变量/参数遮蔽，成员变量`family`未被正确设置为`AF_INET6`。后续`SetBianryAddress(family, binaryAddr)`读取未正确赋值的成员`family`，导致IPoURMA场景下EID初始化的IP地址类型判断错误。
- 修复模式：改为`this->family = AF_INET6;`，确保赋值目标是成员变量。
- 可审查性：高
- 审查规则建议：编译选项必须开启`-Wshadow`并设为error；构造函数中赋值成员变量优先使用`this->`前缀或初始化列表；对裸名赋值成员变量的写法保持警觉。

### 666b6ab5 fix pipeline bug
- 根因类别：算法正确性（边界条件缺失）
- 涉及文件：src/algorithm/impl/operator/all_reduce_operator.cc, src/algorithm/impl/operator/coll_alg_operator.cc, src/algorithm/impl/operator/reduce_scatter_operator.cc
- 缺陷描述：910B AllReduce和ReduceScatter算法选择中，pipeline算法未正确处理server内2卡（`deviceNumPerAggregation_ == DEVICE_TWO`）场景。Pipeline算法需至少3卡，但原代码缺少设备数下界检查；确定性模式与pipeline的交叉条件不完整。
- 修复模式：三处pipeline选择条件加入`deviceNumPerAggregation_ > DEVICE_TWO`前置检查；ReduceScatter放宽非pipeline入口条件使2卡场景走正确路径。
- 可审查性：中
- 审查规则建议：算法选择分支必须覆盖所有设备拓扑配置（1卡/2卡/多卡）；Pipeline算法在所有选择分支中必须有设备数下界检查；确定性模式和设备数的交叉组合应有完整决策表；多operator共享类似算法选择逻辑时同步审查。

### 2bfa02d1 fix some detail with multi process in per rank
- 根因类别：配置与兼容性、错误处理
- 涉及文件：src/legacy/framework/resource_manager/socket/socket_manager.cc, src/legacy/framework/topo/new_topo_builder/rank_table_info/new_rank_info.cc, src/legacy/framework/topo/new_topo_builder/rank_table_info/new_rank_info.h, src/legacy/framework/topo/rank_info_detect/preempt_port_manager.cc
- 缺陷描述：(1) `devicePort`默认值与哨兵值混淆——`MAX_VALUE_DEVICEPORT`(65536)既用作默认值又用作"未配置"哨兵，且65536超出合法端口范围上限65535（off-by-one）。(2) `ServerInit`中`Listen()`不检查返回值，端口被占用时不报错。(3) 端口抢占失败时错误日志硬编码了错误的环境变量名。
- 修复模式：引入`DEFAULT_VALUE_DEVICEPORT = 60001`替代硬编码；修正MAX为65535；Listen返回值检查失败时抛异常；根据NIC类型动态选择环境变量名。
- 可审查性：中
- 审查规则建议：检查"默认值"和"哨兵值"是否混用同一常量；端口号最大值应为65535；网络`Listen`/`Bind`返回值必须检查；错误日志中环境变量名须与上下文匹配。

### a0f05be2 DTS2026020617801 broadcast bugfix
- 根因类别：并发问题（同步时序）
- 涉及文件：src/algorithm/base/alg_aiv_template/aiv_broadcast_910b_bigdata.h
- 缺陷描述：910B大数据broadcast算法的`WaitRecordSync`中，中转rank先`Record`通知root"数据已取走"再`Wait`等待下游。root可能在下游未完成数据拷贝时就覆盖发送buffer。典型的生产者-消费者同步顺序错误。
- 修复模式：将`Record`从循环前移到循环后（先Wait所有下游完成再Record通知上游）；`PipeBarrier`移到每次Wait后确保流水线同步。
- 可审查性：低
- 审查规则建议：集合通信`Record`/`Wait`顺序必须严格审查：先Wait依赖方完成再Record通知上游；多rank同步绘制信号依赖图验证；`PipeBarrier`位置变动需验证在数据依赖之间。

### 7990e3f3 修复对repeat参数错误的校验
- 根因类别：算法正确性（参数校验逻辑错误）、资源生命周期
- 涉及文件：src/framework/device/aicpu_kfc/decoupler/comm_kfc_aicpu_server.cc, src/framework/device/aicpu_kfc/framework/aicpu_kfc_deprecated_process.cc, test/ut/aicpu_kfc/aicpu/ut_aicpu_kfc_unfold_utest.cc
- 缺陷描述：(1) `repeatCnt > TILING_TURN_MAX`校验不正确——repeat实际取值范围应为0~255（u8），iota初始化也应用`UINT8_MAX + 1`而非`TILING_TURN_MAX + 1`。(2) `Finalize`中`HcclReleaseComm`在所有task完成前被调用，资源提前释放。
- 修复模式：删除错误的上限校验；iota改为UINT8_MAX+1；将释放逻辑从`Finalize`移到`IsAllTaskFinished`中。
- 可审查性：中
- 审查规则建议：参数校验上限值须与参数实际类型和语义一致；资源释放须在确认所有使用方完成后执行；初始化数组大小须与实际最大使用范围一致。

### 9939a862 fix some log and not throw with mc2 interface
- 根因类别：错误处理、内存管理
- 涉及文件：src/legacy/framework/entrance/op_base/op_base_v2.cc
- 缺陷描述：(1) 三个Set函数将外部传入整数直接强转为枚举索引数组，无越界检查，可致数组越界；MC2是C接口语义但原代码可能抛C++异常。(2) `HcclDevMemAcquireV2`对外部`memTag`(char*)直接`std::string()`构造，可能读取越界内存。
- 修复模式：改为数组下标访问前先做边界检查，越界返回`HCCL_E_PARA`不抛异常；memTag改用栈缓冲区+`strcpy_s`安全拷贝。
- 可审查性：高
- 审查规则建议：整数作数组索引前必须边界检查；C接口语义函数不应抛C++异常；外部C字符串禁止直接`std::string(ptr)`构造，须先长度校验或用安全函数。

### 4e0bd97b 修复跨超节点场景对称内存指针未判空问题
- 根因类别：内存管理（空指针解引用）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, test/ut/framework/communicator/impl/symmetric_memory/ut_hccl_communicator_host.cc
- 缺陷描述：`IsSupportSymmetricMemory`使用`symmetricMemory_`前未判空。跨超节点场景下该指针可能未初始化（某些设备类型不创建symmetric memory对象），触发空指针解引用导致崩溃。
- 修复模式：入口处增加`CHK_PRT_RET(symmetricMemory_ == nullptr, ..., false)`；UT全面重构增加初始化和非空断言。
- 可审查性：高
- 审查规则建议：可选初始化路径设置的成员指针使用前必须判空；UT必须覆盖"对象未初始化"场景；跨节点功能特性须验证不同硬件配置下的graceful降级。

### 2a4ec69d 修复函数参数名定义与声明不一致
- 根因类别：其他（代码规范性问题），附带内存管理缺陷（未初始化成员变量）
- 涉及文件：29个文件（大规模清理）
- 缺陷描述：头文件声明参数名与.cc实现参数名不一致（如`ranksPort` vs `nicRanksPorts`）；参数名使用成员变量命名风格（带`_`后缀）；参数名拼写错误（`indOpMemd`/`isBakup`）；使用`NULL`而非`nullptr`；`u64 cclBufferSize_;`未初始化。
- 修复模式：逐文件修正参数名一致性、拼写、`NULL`替换`nullptr`、成员变量添加默认值`= 0`。
- 可审查性：高
- 审查规则建议：CI引入clang-tidy规则`readability-inconsistent-declaration-parameter-name`；启用`modernize-use-nullptr`和`cppcoreguidelines-init-variables`；功能性修复与规范性清理应拆分为独立提交。

### 75659d24 修复重执行多线程下的时序问题
- 根因类别：并发问题（多线程操作顺序）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：两处"先重置状态标记再清理资源"的错误顺序。(1) `HcclOpExecFsmWaitRetryProcess`中先`ResetOpRetryException`再重置`pollStatus`/`cqeStatus`，其他线程可能在dfx状态保留旧值时开始新操作。(2) `CleanStream`中先`ResetStreamCqeExceptionStatus`再`ClearLocalBuff`/`UpdateSqStatus`，其他线程在资源未清理时就尝试使用。
- 修复模式：调换操作顺序——先完成实际清理再修改对外可见状态标记（publish ordering原则）。
- 可审查性：低
- 审查规则建议：对"状态重置"函数审查：重置后其他线程是否立即感知？重置前的依赖操作是否已完成？建立"先完成实际工作再修改对外可见状态"审查检查项；考虑引入TSan到CI。

### e4f59213 ccu fallback algname cache bug fix
- 根因类别：算法正确性（函数副作用污染缓存key）
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc
- 缺陷描述：`AcceleratorFallback()`中，`OpAcceleratorStateFallback()`会修改成员变量`curAlgName`，但缓存插入在此调用之后使用`curAlgName`作key。导致缓存映射为"fallback后的算法名 -> 加速模式"而非"原始算法名 -> 加速模式"，后续查询无法命中。
- 修复模式：调用前用局部变量`needFallBackAlgName = curAlgName`保存原始值，后续用该变量作key。
- 可审查性：中
- 审查规则建议：函数调用前后使用同一成员变量时须确认中间调用是否修改该变量；缓存插入的key语义须与查询时一致；对会修改成员变量的函数标注副作用文档。

### 07c52708 fix msg
- 根因类别：配置与兼容性（设备类型分支遗漏）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：`InitThreads`末尾`InitProfthreadResource`对所有设备类型无差别执行，但910_95设备不支持此初始化，导致初始化失败中断整个通信流程。
- 修复模式：添加`if (deviceType != DEV_TYPE_910_95)`条件守卫。
- 可审查性：中
- 审查规则建议：新增硬件资源初始化代码须审查是否适用所有设备类型；建立设备能力矩阵文档；设备类型排除条件须注释原因。

### fb56d64b fix loopnum = 128
- 根因类别：算法正确性（硬件参数常量值不正确）
- 涉及文件：src/legacy/unified_platform/ccu/ccu_microcode/ccu_microcode.h, test/legacy/ut/unified_platform/ccu/ccu_representation/ut_ccu_dfx.cpp
- 缺陷描述：`CCU_MS_DEFAULT_LOOP_COUNT`设为64但正确值为128。该常量控制CCU微码段默认循环次数，过小导致不必要的微码分段，增加通信延迟。UT中profiling段数期望值从3降为2印证了修复。
- 修复模式：常量从64改为128；更新8个UT测试用例的期望段数。
- 可审查性：高
- 审查规则建议：硬件相关常量值须在注释中说明取值依据；常量修改时全局搜索所有引用确认影响范围；考虑从硬件配置动态读取减少硬编码出错。

## Batch 5: Commit #61-75 (2026-02-11 ~ 2026-02-25)

### 72cdf80e Revert "fix loopnum = 128"
- 根因类别：配置参数修改引发行为回归（错误的"修复"被撤回）
- 涉及文件：src/legacy/unified_platform/ccu/ccu_microcode/ccu_microcode.h, test/legacy/ut/unified_platform/ccu/ccu_representation/ut_ccu_dfx.cpp
- 缺陷描述：对fb56d64b的完整revert，两者间隔仅6小时。原始commit将`CCU_MS_DEFAULT_LOOP_COUNT`从64改为128，同时将UT期望值从3降为2。该常量是全局常量，被多个算法模板引用（allgather_mesh_1D_detour、reduce_scatter_mesh_detour_1D、all_reduce_mesh_detour_1D等），loop count加倍影响CCU微码执行时序、内存带宽压力、流水线交叠。原始commit的PR描述完全空白（无关联Issue、无测试方案），且混入了大量无关的代码风格变更（花括号位置、空格），掩盖了真正的逻辑变更。
- 修复模式：完整git revert，恢复常量值和所有UT期望值。
- 可审查性：中
- 审查规则建议：全局常量修改（尤其硬件相关参数）必须有设计文档说明取值依据，PR描述不能为空；UT期望值大面积同步修改应触发审查警告；代码风格变更不应与逻辑变更混在同一commit中；影响多算法路径的基础参数修改应提供性能对比数据和多场景回归测试。

### b0e6a8b7 fix ffts bug for deter pipeline
- 根因类别：运行时路径选择与子图key计算数据源不一致
- 涉及文件：src/algorithm/base/alg_template/alg_template_multi_deter_pipeline.cc/.h, src/algorithm/base/alg_template/temp_all_reduce/all_reduce_multi_deter_pipeline.cc/.h, src/algorithm/base/alg_template/temp_reduce_scatter/reduce_scatter_multi_deter_pipeline.cc/.h
- 缺陷描述：`RunAsyncReduceScatterPipeline()`中用`curSize_`判断是否走串行算法（<2MB），但FFTS子图key用`perRankAvgDataSize_`（总量/卡数）生成。AllReduce场景下最后一轮最后一张卡的`curSize_`可能因不均分而<2MB走串行路径，但子图key对应的是pipeline路径（平均数据量>=2MB），导致子图无法复用。本质是"运行时路径选择"与"编译时子图key计算"使用了不同数据源做同一决策。
- 修复模式：引入多态方法`GetLocalReduceSerialThresh()`——AllReduce返回`perRankAvgDataSize_`（与子图key计算口径一致），ReduceScatter仍返回`curSize_`；附带将阈值类型从s64改为u64避免有符号/无符号比较。
- 可审查性：中
- 审查规则建议：分支条件与缓存/子图key必须使用相同数据源；基类模板方法中直接使用的成员变量如果在不同派生类中语义不同，应用虚函数封装；有符号/无符号混合比较应由`-Wsign-compare`捕获。

### fab6dbb7 解决alltoallv精度问题
- 根因类别：多轮迭代(repeat)场景下偏移量未累加——逻辑遗漏
- 涉及文件：src/framework/device/aicpu_kfc/decoupler/comm_kfc_aicpu_server.cc/.h
- 缺陷描述：MC2通信算子alltoallv在`repeatCnt > 1`时精度错误。`FormatOpData()`中alltoallv的`sdispls`和`recvOffset`指针只在`repeat == 0`时初始化，后续repeat轮次没有对per-rank的offset数组进行累加更新。每一轮都使用相同偏移量读写数据，导致重复读和覆盖写。函数末尾的通用偏移逻辑`data.input = msg.sendBuffer + offset * repeat`只调整基地址，但alltoallv内部还依赖sdispls/rdispls决定rank-pair间子区间偏移。
- 修复模式：`extMsg`去掉const限定允许原地修改；新增`u32 rankNum`参数；在`repeat > 0`时遍历所有rank对sendOffset和recvOffset累加对应的sendCounts和recvCounts值。
- 可审查性：中
- 审查规则建议：当函数对多种操作类型分支处理且有通用后处理逻辑时，应逐一验证通用逻辑对每种类型是否充分；`if (i == 0)`初始化的可变状态在`i > 0`时是否需要演进；非均匀集合通信算子（alltoallv/allgatherv/reducescatterv）的repeat/分片逻辑比均匀算子更复杂，审查应重点关注偏移计算。

### 3323cecd fix getEndpointdesc
- 根因类别：数据源错误 + 方向性错误（复合缺陷，3处独立问题）
- 涉及文件：src/legacy/framework/topo/new_topo_builder/rank_graph/rank_graph.cc, rank_gph.h, rank_graph_builder.cc/.h, src/legacy/interface/rank_graph_interface.cc, src/legacy/framework/communicator/communicator_impl.cc
- 缺陷描述：三个关联缺陷：(1) `GetEndpointDesc()`是查询函数却有副作用——内部调用`SetEndpointToIface()`注册映射，导致`endpointToIfaceMap_`内容取决于调用顺序；(2) `rank_graph_interface.cc`中获取dst endpoint的devPhyId时错误使用`peer2net->GetSourceNode()`而非`net2peer->GetTargetNode()`；(3) `AppendLocalDieIdForLinks()`只处理正向边的source端dieId，遗漏反向边的target端。
- 修复模式：将endpoint注册从查询函数移到构建阶段消除副作用；修正方向性API调用；用lambda统一处理正反向边。
- 可审查性：高
- 审查规则建议：`Get*`前缀函数体内不应包含`Set/Update/Insert`调用；方向对称性检查——`dst.*GetSource`或`src.*GetTarget`模式应标记为可疑；图边遍历`GetEdges(A,B)`时检查是否需要处理`GetEdges(B,A)`。

### f7183c87 vcache fix bug
- 根因类别：多缺陷混合（executor遗漏逻辑 + 空指针静默跳过 + 格式化字符串UB）
- 涉及文件：src/framework/device/framework/aicpu_cache_manager.cc, src/algorithm/impl/coll_executor/coll_all_to_all/coll_all_to_all_v_direct_fullmesh_executor.cc, 及8个executor文件
- 缺陷描述：三个独立子缺陷：(1) AlltoAllDirectFullmesh executor的Orchestrate末尾缺少`LaunchTaskExtend`调用，其他所有executor（AllGather/AllReduce/Broadcast/Reduce/ReduceScatter/Scatter/AlltoAll）都有该逻辑，导致cache miss路径中SQE序列不完整；(2) `PostProcessForCacheMiss`中用`if (entryPtr != nullptr)`宽松处理本应必然非空的entry指针，实际应用`CHK_PTR_NULL`硬断言；(3) `GetKeyString()`返回std::string直接传给`%s`是UB，缺少`.c_str()`。
- 修复模式：为AlltoAllDirectFullmesh补上LaunchTaskExtend并在所有executor添加"不要删除"警告注释；将宽松if改为CHK_PTR_NULL断言；补上.c_str()。
- 可审查性：高
- 审查规则建议：同一抽象层次的所有子类应具备相同结构模式——N-1个有某段逻辑而1个没有应标记为遗漏；明确post-condition路径上对指针做`if != nullptr`而非断言的标记为"静默吞错误"风险；`HCCL_INFO/ERROR`中`%s`参数必须是`const char*`不能是`std::string`。

### 12f3680c fix wrong info in selector
- 根因类别：复制粘贴错误（Copy-Paste Bug）
- 涉及文件：src/legacy/service/collective/alg/selector/all_reduce_auto_selector.cc, reduce_auto_selector.cc, scatter_auto_selector.cc
- 缺陷描述：多个selector类的日志打印中类名标识写错。`AllReduceAutoSelector`打印`[ReduceScatterAutoSelector]`，`ReduceAutoSelector`打印`[AllGatherAutoSelector]`，`ScatterAutoSelector`全部打印`[AllGatherAutoSelector]`——从其他selector文件复制代码后未修改日志中的类名。调试时错误日志把开发者引向错误代码路径。
- 修复模式：纯文本替换，3个文件10处字符串。
- 可审查性：高
- 审查规则建议：静态检查——日志字符串中`[XxxSelector]`模式的类名前缀应与当前文件名或类名匹配；可用clang-tidy自定义检查或grep脚本自动发现。

### b65526e9 fix device listen port not same
- 根因类别：架构设计缺陷——端口映射作用域和生命周期管理错误
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc/.h, hccl_communicator.cc/.h, op_base_v2.cc, socket_manager.cc/.h, new_rank_info.cc, rank_table_info.cc, rank_info_detect.cc/.h, rank_info_detect_client.cc/.h, rank_info_detect_service.cc/.h
- 缺陷描述：四个核心问题：(1) `rankListenPortMap_`是SocketManager实例成员，子通信域需手动拷贝父域端口映射，脆弱易漏；(2) HCCL_NPU_SOCKET_PORT_RANGE=auto端口抢占模式下，实际端口在初始化后才确定但无回传机制；(3) Check函数强制校验所有rank的devicePort必须相同；(4) CreateConnectedSocket查本地而非对端rank的监听端口。
- 修复模式：大架构重构——`rankListenPortMap_`改为static全局存储；新增两阶段初始化协议（探测拓扑 + 端口回传广播）；删除端口相同校验；改查全局端口映射。
- 可审查性：低
- 审查规则建议：分布式系统中涉及端口的协议设计须明确"各节点端口是否可不同"的假设；审查"实例状态 vs 全局状态"选择是否合理；多rank协调信息需完整同步机制而非依赖隐式假设。

### 3b8ac218 fix dpu kernel malloc devBuf
- 根因类别：执行上下文错误（Wrong Context for Resource Allocation）
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc/.h, src/legacy/framework/communicator/hostdpu/dpu_kernel_entrance.cc
- 缺陷描述：`LaunchDpuKernel`内同时做HBM内存分配和kernel下发。`InitAndLaunchDpuKernel`先切换到DPU context再调用它，导致`CreateWorkspaceBuf`在DPU context下执行而非NPU context。附带：`hostShareBuf`未初始化为nullptr（析构时free UB）；分配失败只打日志不return；`deviceId`用`HrtGetDevice()`取当前context（已切到DPU）而非使用成员变量`devLogicId`。
- 修复模式：将内存分配移到context切换前；`hostShareBuf`零初始化；分配失败加return；deviceId改用成员变量。
- 可审查性：中
- 审查规则建议：`aclrtSetCurrentContext`/context切换附近的内存分配调用须审查执行context是否正确；类成员裸指针必须初始化为nullptr（cppcoreguidelines-init-variables）；分配失败后须return/throw不能仅打日志。

### 69c73166 fix bug
- 根因类别：异常吞没（exception swallowing）
- 涉及文件：src/legacy/unified_platform/resource/ccu_transport/ccu_transport.cpp, src/legacy/framework/ccu/ccu_ins_preprocessor.cpp
- 缺陷描述：`CcuTransport::GetStatus()`手写try-catch捕获异常后只打印日志就return CONNECT_FAILED，异常被吞没不向上传播。与代码库统一的`TRY_CATCH_PROCESS_THROW`宏模式不一致。且catch块只设局部变量status未更新成员变量`transStatus`，导致内部状态与返回值不一致。
- 修复模式：替换为统一的`TRY_CATCH_PROCESS_THROW`宏，用lambda包装保护代码，异常时更新transStatus并由宏重新抛出。
- 可审查性：高
- 审查规则建议：当项目有统一异常处理宏时，不允许裸try-catch处理同类异常；catch中仅打印日志就return的模式标记为"异常吞没"风险。

### e0744b7b fix bug of alltoallv perf
- 根因类别：缓存key生成不完整——缺少影响行为的状态维度
- 涉及文件：src/framework/device/framework/aicpu_cache_manager.cc/.h, src/platform/common/unfold_cache/op_unfold_key.h, src/algorithm/base/alg_template/temp_alltoallv/alltoallv_direct_fullmesh.cc
- 缺陷描述：AlltoAllV的op-unfold cache key缺少`isBigCount`维度。`isBigCount`（maxSendLen > BIG_SIZE阈值）决定SQE编排是否并发执行，但cache key未区分该状态，导致数据量变化使isBigCount翻转后仍命中旧缓存，使用错误编排方案。
- 修复模式：在GetOpUnfoldKey的alltoallv分支中新增IsBigCountForAlltoallv()计算，将结果编码到cache key的inputSize字段（0或1）；在Prepare()添加注释提醒逻辑同步。
- 可审查性：中
- 审查规则建议：缓存机制的cache key是否覆盖所有影响行为的状态维度；新增分支逻辑如果影响执行路径须检查是否反映在cache key中；跨模块的逻辑重复应抽取公共函数。

### 2fcde546 HCCP fix log
- 根因类别：错误码处理粒度不足——日志级别不当
- 涉及文件：src/platform/hccp/rdma_agent/peer/ra_peer.c
- 缺陷描述：`RaPeerSetQpLbValue()`对底层`RsSetQpLbValue()`所有非零返回值统一用`hccp_err`打印。但`-ENOTSUPP`是预期内的合法返回值（硬件/驱动不支持），按error级别上报产生误导性错误日志和不必要告警。
- 修复模式：对`-ENOTSUPP`特殊处理，降级为`hccp_run_warn`且措辞从"failed"改为"unsuccessful"。
- 可审查性：高
- 审查规则建议：调用可能返回`-ENOTSUPP`等"功能不支持"错误码的接口时，日志级别不应为error；底层接口有多种合法失败语义时不应对所有非零返回值用同一日志级别。

### a9fea200 修复资源释放不完全问题
- 根因类别：资源泄漏（释放路径不完整） + 边界条件缺失
- 涉及文件：src/framework/communicator/impl/symmetric_memory/symmetric_memory.cc/.h, src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：三个关联问题：(1) `RegisterInternal()`中`aclrtMapMem`映射后未保存虚拟地址到handle映射，释放时用`aclrtMemRetainAllocationHandle`重新获取handle不可靠；(2) `refCount > 1`时减引用计数前未释放当前window的`paHandle`致物理内存泄漏；(3) 单卡场景（rankSize==1）直接报错而非优雅降级。
- 修复模式：引入`importAddrs_`(unordered_map)跟踪映射关系；补充refCount>1分支的paHandle释放；单卡场景返回SUCCESS并分配空壳SymmetricWindow。
- 可审查性：中
- 审查规则建议：分配/映射的资源必须在释放函数中有明确释放路径，不应依赖运行时API重新获取handle；引用计数递减分支也须检查是否有局部资源需释放；析构函数应清理所有容器类成员。

### e1e83e18 fix switch
- 根因类别：控制流逻辑错误——冗余条件守卫导致功能缺失
- 涉及文件：src/framework/hcom/hcom_host_profiling.cc
- 缺陷描述：`HcommProfilingReportOp`中`CallMsprofReportHostApi`前用`if (GetIfProfile())`做条件判断，但调用方已检查过profiling开关。此处重复检查导致某些场景下（开关状态变化或时序不一致）profiling数据不上报。
- 修复模式：删除冗余的`if (GetIfProfile())`守卫，让函数无条件执行上报。
- 可审查性：低
- 审查规则建议：条件守卫放错层级难以静态发现；需架构层面明确profiling开关检查点在哪一层，建立规范"profiling上报函数内部不应再检查开关状态"。

### dd053d5b ccu fallback algname cache bug fix
- 根因类别：缓存数据不完整导致fallback后算法选择错误
- 涉及文件：src/legacy/framework/communicator/communicator_impl.cc/.h
- 缺陷描述：CCU加速器fallback时将opType+algName映射到AcceleratorState并缓存。命中缓存恢复AcceleratorState后，`curAlgName`仍是fallback前的值，但不同数据量/类型下算法可能不同，导致选错算法。缓存只保存了AcceleratorState没保存fallback后的实际algName。
- 修复模式：缓存值从AcceleratorState扩展为`pair<AcceleratorState, string>`（加newAlgName），命中缓存时同时恢复curAlgName并递归调用ExecAlgSelect重新走算法选择。
- 可审查性：中
- 审查规则建议：缓存用于恢复/重放逻辑时，检查是否保存了该逻辑所有依赖的输入状态；命中缓存恢复状态后，检查是否还需重新执行依赖该状态的后续逻辑。

### 6383b2bc fix HcclGetCommConfigCapability
- 根因类别：v2适配遗漏——接口未分发到新版本实现
- 涉及文件：src/framework/op_base/src/op_base.cc, src/framework/op_base/src/op_base_v2.h, src/legacy/framework/entrance/op_base/op_base_v2.cc/.h
- 缺陷描述：`HcclGetCommConfigCapability()`在v1流程返回`HCCL_COMM_CONFIG_RESERVED`（支持所有配置项），但v2流程只到`HCCL_COMM_CONFIG_RETRY`。该函数没有走v2分发路径（缺少`HCCLV2_FUNC_RUN`宏），始终返回v1结果，导致v2场景下调用者误以为支持v1全部配置项。
- 修复模式：新增`HcclGetCommConfigCapabilityV2()`实现，在v1入口通过`HCCLV2_FUNC_RUN`宏分发。
- 可审查性：高
- 审查规则建议：`op_base.cc`中所有公开API函数体开头必须包含`HCCLV2_FUNC_RUN`分发调用，缺失即为lint错误；可写自动化脚本扫描op_base.cc中公开函数检查V2对应函数和分发调用。

## Batch 6（#76-84）

### 5ad2ced6 fix new driver pkg compatable to old can pkg for tlv
- 根因类别：配置与兼容性——跨版本协议OpCode不兼容
- 涉及文件：src/platform/hccp/inc/private/network/ra_rs_comm.h, ra_adp.c, ra_adp_tlv.c/.h, ra_hdc.c/.h, ra_hdc_tlv.c/.h, rs.c, dl_netco_function.c（共10文件）
- 缺陷描述：TLV init接口的OpCode（`RA_RS_TLV_INIT=87`）在新版驱动中数据结构发生变化（`OpTlvInitData`中`txData`的reserved从61变为62），但旧版CANN包仍发送旧格式数据。新驱动收到旧格式请求时由于OpCode相同但数据布局不匹配，导致功能异常。
- 修复模式：将旧OpCode87重命名为`RA_RS_TLV_INIT_V1`，新增OpCode110为`RA_RS_TLV_INIT`，并为V1版本创建兼容处理函数`RaRsTlvInitV1`（直接返回warn日志），新版本通过`RaHdcGetInterfaceVersion`检查版本后选择正确的OpCode。
- 可审查性：中
- 审查规则建议：涉及跨组件/跨版本接口的OpCode或数据结构修改时，必须保留旧版本OpCode和处理函数做兼容；新OpCode应分配新编号而非原地修改。审查所有跨进程/跨设备通信协议变更时检查是否有版本协商机制。

### 7d91416863 bugfix issues/63
- 根因类别：资源生命周期——析构顺序错误（日志依赖先于日志使用被销毁）
- 涉及文件：（空diff merge commit，实际修复代码在合入分支中）
- 缺陷描述：根据PR描述"先打印日志后再析构"，析构函数中在对象被部分销毁后仍尝试使用日志功能，此时日志依赖的成员已被析构，导致崩溃或未定义行为。
- 修复模式：调整析构顺序，在释放资源前先完成日志打印。
- 可审查性：中
- 审查规则建议：析构函数中的日志打印语句应在所有资源释放操作之前；审查析构函数时检查日志/DFX调用是否依赖即将被释放的成员变量。注意：此commit是空diff的merge commit（tree对象与parent相同），说明rebase/merge策略可能导致commit历史中出现空提交。

### bb490dc2 修复CcuKernel析构函数缺少符号问题
- 根因类别：C++语言特性——`= default`析构函数在跨编译单元场景下的符号缺失
- 涉及文件：pkg_inc/hcomm/ccu/ccu_kernel.h, src/framework/next/comms/ccu/ccu_kernel/ccu_kernel.cc
- 缺陷描述：`CcuKernel`类的虚析构函数声明为`~CcuKernel() override = default;`，当该类被用作跨动态库边界的多态基类时，`= default`的析构函数可能不会生成out-of-line定义，导致链接时找不到vtable或析构符号。
- 修复模式：将`= default`改为在.cc文件中提供空的显式定义`CcuKernel::~CcuKernel() {}`。
- 可审查性：高
- 审查规则建议：跨动态库边界的多态类（尤其是pkg_inc导出的头文件中的类），其虚析构函数不应使用`= default`，应在.cc中提供显式定义以确保符号在正确的编译单元中生成。这是C++ ODR和动态链接的经典陷阱。

### 9d75f557 CopyCommEngineCtx AIV bugfix
- 根因类别：算法正确性——引擎类型分支归属错误（memcpy方向错误）
- 涉及文件：src/framework/communicator/impl/independent_op/independent_op_context_manager.cc
- 缺陷描述：`CopyCommEngineCtx`函数根据engine类型选择内存拷贝方式：AICPU_TS/AICPU使用H2D拷贝（`hrtMemSyncCopy`），CPU/CPU_TS/CCU/AIV使用host端`memcpy_s`。但AIV引擎实际运行在Device端，其context内存应使用H2D拷贝而非host端memcpy。AIV被错误归入了CPU分支。
- 修复模式：将`COMM_ENGINE_AIV`从`memcpy_s`分支移到`hrtMemSyncCopy`(H2D)分支。
- 可审查性：高
- 审查规则建议：switch/if-else分支处理不同引擎/设备类型时，审查每个枚举值的归属是否与其实际运行位置（Host/Device）一致。新增引擎类型时，须逐一审查所有按引擎类型分支的代码点（可grep `COMM_ENGINE_`枚举使用处）。此类缺陷模式在Batch 3的099fe2a8（设备类型分支不等价）中也出现过。

### edf73e80 fix rdma to roce
- 根因类别：配置与兼容性——协议名称字符串与枚举映射不匹配
- 涉及文件：src/legacy/framework/topo/new_topo_builder/topo_info/edge_info.cc, 及3个测试文件
- 缺陷描述：拓扑信息解析中，JSON配置使用`"RDMA"`作为链路协议名，但映射目标枚举值已改名为`LinkProtocol::ROCE`。虽然map映射`{"RDMA", LinkProtocol::ROCE}`功能上正确，但语义上不一致，且如果外部配置文件已更新为使用`"ROCE"`名称则会解析失败。
- 修复模式：将映射key从`"RDMA"`改为`"ROCE"`与枚举名一致，同时更新所有测试中的JSON字符串。
- 可审查性：高
- 审查规则建议：字符串到枚举的映射表中，key字符串应与枚举值名称保持一致；重命名枚举值时须全局搜索对应字符串常量同步更新。配置文件/序列化中使用的字符串常量应有集中定义，避免散落在代码各处。

### ed50e7eb fix log module name
- 根因类别：日志与调试——日志模块ID与实际模块不匹配
- 涉及文件：src/platform/hccp/external_depends/inc/log/log_types.h, src/platform/hccp/inc/private/network/user_log.h, test/ut/depends/pkg_inc/base/log_types.h
- 缺陷描述：ROCE相关日志宏使用了`ROCE`作为模块ID，但日志系统的模块枚举中没有定义`ROCE`（或其值不正确）。修复后新增`SCC=2`、`CCU=5`、`NET=11`三个模块枚举，并将所有`roce_*`日志宏的模块ID从`ROCE`改为`NET`。
- 修复模式：在`log_types.h`中补充缺失的模块枚举定义（SCC/CCU/NET），将ROCE日志宏统一使用正确的模块ID `NET`。
- 可审查性：高
- 审查规则建议：日志模块ID必须在日志系统枚举中有明确定义；新增子系统时须同步在日志模块枚举中注册。日志宏中的模块ID应使用集中定义的常量，不应硬编码。可自动化检查：日志宏使用的模块ID是否都在枚举定义中存在。

### 18752141 fix batchsendrecv
- 根因类别：算法正确性——资源标识key不唯一导致不同算子实例复用错误资源
- 涉及文件：communicator_impl_lite.cc/.h, kernel_param_lite.h, socket_manager.cc, coll_service_ai_cpu_impl.cc/.h, hccl_one_sided_service.cc, coll_alg_component.cc, ut_aicpu_stream_manager.cc（共9文件）
- 缺陷描述：BatchSendRecv算子在AICPU执行时，资源（algTopoInfoMap、collOpLoadedMap）的key使用`algName`或`opTag`，但同一通信域中不同的BatchSendRecv调用可能有不同的remote rank组合却共享相同的algName/opTag。导致后发的BatchSendRecv算子复用了前一次分配的资源（拓扑信息、device内存），但实际通信对端不同，造成数据错乱。
- 修复模式：新增`tagKey`字段（= opTag/algName + remote rank集合的hash值），替代原来的algName/opTag作为资源唯一标识。新增`GetRemoteRankIdsHashValue()`对远端rank排序后做hash。同时修复BSR场景下`DevBuffer`内存未保持引用（改用`bsrItemsMem`列表持有shared_ptr）。
- 可审查性：中
- 审查规则建议：资源池/缓存的key设计必须能唯一标识资源使用者的所有关键属性。特别对于点对点和非均匀集合通信，通信对端信息必须参与key计算。审查map/cache的key时问"同一key下是否可能出现不同语义的资源请求？"。此缺陷与Batch 5中e0744b7b（cache key缺维度）和dd053d5b（cache数据不完整）属于同一模式：cache/资源池的标识不充分。

### 753ba8c2 Revert "[Build] Add support for offline compile [2/2]"
- 根因类别：构建系统——未经充分验证的构建脚本变更导致快速revert
- 涉及文件：CMakeLists.txt, cmake/hcomm_utils.cmake(deleted), cmake/utils.cmake(restored), 及22个cmake/third_party/*.cmake和CMakeLists.txt文件
- 缺陷描述：原始提交05b38411在合入后约1小时即被revert，说明离线编译支持的构建变更引入了严重问题。原提交将`cmake/utils.cmake`替换为`cmake/hcomm_utils.cmake`（新增347行，主要是从远程下载hcomm-utils预编译包的逻辑），同时大幅修改了多个third_party cmake文件的依赖获取方式和openssl编译逻辑。变更影响了24个文件（640+/499-行），构建流程的核心路径。
- 修复模式：完整revert原始提交，恢复`cmake/utils.cmake`，删除`cmake/hcomm_utils.cmake`。
- 可审查性：中
- 审查规则建议：构建系统的重大变更（特别是依赖获取方式、编译流程变更）应有独立的CI验证pipeline并在staging环境充分测试后再合入主干。单次合入不应同时修改20+个构建文件——应分步骤合入以便定位问题。跨1小时即revert说明可能连基本编译都未通过，需要更严格的pre-merge CI门禁。这是本仓库第2次revert（与72cdf80e构成全部2次revert记录）。

### 9bcb1bdc [Bugfix] fix bugs of CP algorithm
- 根因类别：算法正确性——CP(Continuous Pipeline) alltoallv在PCIe链路下多处逻辑错误
- 涉及文件：alltoallv_continuous_pipeline.cc, alltoallv_continuous_pipeline_pub.h, testcase_all_to_allv.cc（共3文件）
- 缺陷描述：AlltoallvContinuousPipeline算法在走PCIe链路（非SDMA）时存在多个缺陷：
  1. `PrepareSendRecvInfo`中本地recvCounts未拷贝到`intraRecvCounts_`数组，导致inter通信时recvMems为空（接收计数全0）。
  2. `InterSendAndReceive`中`needCollectInfo_`条件守卫导致SDMA路径下不填写recvMems（仅走InterSdmaTx写方向），但实际PCIe路径下应始终填充sendMems和recvMems。
  3. `RunAsync`主循环中等待接收信息的条件写成了`intraState.loopNum == 0`（intra循环）但实际应在`interState.loopNum == 0`（inter循环）前等待——在错误的循环阶段等待导致时序错乱。
  4. 存在冗余的`InterSdmaTx`函数（仅做写方向），在修复后被完全删除，改为统一使用`InterSdmaRx`（读方向）。
- 修复模式：补充`std::copy`初始化intraRecvCounts_；移除`needCollectInfo_`对recvMems填充的条件守卫；修正循环条件从intra改为inter；删除不再需要的InterSdmaTx函数。新增6个ST测试用例覆盖多种拓扑（2srv-2nodes/4nodes/8nodes、4srv-8nodes、1srv-16nodes、大数据量）。
- 可审查性：中
- 审查规则建议：流水线(pipeline)算法涉及多阶段（intra/inter）时，审查每个阶段的数据准备是否完整、条件守卫是否指向正确的阶段变量（intraState vs interState）。新增通信路径类型（如PCIe vs SDMA）时须覆盖所有已有逻辑分支，而非仅在部分分支添加条件守卫跳过。拥有send和receive两个方向时，审查两个方向的数据准备是否对称完整。
