# hcomm-dev 缺陷深度分析

全量分析162条缺陷提交的diff，每条记录根因、涉及文件、缺陷描述、修复模式、可审查性和审查规则建议。

---

## 第1轮 (commits 1-20)

### d89396c950503a69ae7deffc229052e1aaf5d55d [Fix] A5 HcommThreadNotifyRecordOnThread
- 根因类别：对象混淆 + 缺少异常捕获
- 涉及文件：src/framework/communicator/impl/independent_op/data_api/hccl_api_data_aicpu_ts.cc
- 缺陷描述：`HcommThreadNotifyRecordOnThread`函数中，从`threadPtr`（源线程）获取notify，但语义上应从`dstThreadPtr`（目标线程）获取。参数`dstThread`被传入却未使用，导致record操作写入了错误的notify对象。此外A5设备路径下所有数据面操作（LocalCopy、LocalReduce、NotifyRecord、NotifyWait、LaunchTask）均缺少异常捕获，抛出的异常会导致未定义行为或进程崩溃。
- 修复模式：将`GetNotify(dstNotifyIdx)`的调用对象从`threadPtr`改为`dstThreadPtr`，并增加`dstThreadPtr`的空指针检查。对A5路径下7处直接调用统一包裹`EXECEPTION_CATCH`宏。
- 可审查性：高
- 审查规则建议：当函数参数被声明但未被使用时应告警（unused parameter检查）；数据面API中与设备交互的调用应统一使用异常捕获宏包裹。

### b9f467057aecb5ff3b4f53fda065ffa5d0b2cb98 修复notify申请失败
- 根因类别：平台适配缺失（910_95设备特殊路径未处理）
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_op_mem.cc, src/framework/communicator/impl/independent_op/resource/engine/thread_manager.cc, src/framework/next/comms/comm_engine_res/threads/aicpu_ts_thread.cc, src/framework/next/comms/comm_engine_res/threads/cpu_ts_thread.cc, src/platform/resource/notify/local_notify.cc, src/pub_inc/local_notify.h
- 缺陷描述：910_95设备类型的notify初始化流程与其他设备不同——不需要`SetIpc()`操作，设备侧也不需要完整的`Init`流程而应使用轻量级的`InitNotifyLite`。原代码对所有设备类型走相同路径，910_95上调用`SetIpc()`会失败导致notify申请失败。同时`HcclGetHcclBuffer`中`commMem`指针未做空指针检查。
- 修复模式：在AicpuTsThread和CpuTsThread的Init中对910_95设备跳过`SetIpc()`调用。新增`LocalNotify::InitNotifyLite`方法仅设置notifyId而不申请新资源。补充`commMem`空指针检查。
- 可审查性：中
- 审查规则建议：新增设备类型时应审查所有资源初始化路径是否需要分支处理；对平台抽象层函数应有设备类型兼容性矩阵。

### d004332ed58db033ba759fe96357bb8750cee440 aiv bugfix
- 根因类别：错误的初始化操作（不必要的内存清零）
- 涉及文件：src/orion/framework/aiv/aiv_mc2/aiv_mc2_compont.cpp
- 缺陷描述：`GenerateCommContext`函数中在获取CclBuffer地址后，使用`HrtMemcpy`将整个buffer清零。分配了一个与`commCclBufferSize`等大的`vector<char>`作为零源，通过`RT_MEMCPY_HOST_TO_DEVICE`拷贝到设备端。这一操作不必要，且可能破坏已有数据、引入性能开销，或在buffer已被其他操作使用时导致数据竞争。
- 修复模式：直接删除4行清零代码（vector分配 + HrtMemcpy调用）。
- 可审查性：中
- 审查规则建议：对设备内存的清零/初始化操作应审查其必要性和时序正确性，避免在共享buffer上执行无条件的memcpy清零。

### df96667c9453f796bcebe9bae23d46ad8843e296 fix va merge
- 根因类别：硬件协议乘数因子遗漏
- 涉及文件：src/orion/unified_platform/resource/connection/aicpu/ub_conn_lite.cc, src/orion/unified_platform/resource/connection/dev_ub_connection.cc, src/platform/hccp/orion/hcomm_dev/rdma_service/ctx/product/ascend910_95/rs_ub_jetty.c, src/platform/hccp/orion/hcomm_dev/rdma_service/ctx/product/rs_ub_jetty.h
- 缺陷描述：URMA约束每个SQE包含4个WQEBB，但host侧创建jetty时指定的`sqDepth`是以SQE为单位的，device侧需要感知WQEBB粒度的深度。多处代码在计算SQ buffer大小和VA映射长度时只乘了`WQE_BB_SIZE`而遗漏了`WQEBB_NUM_PER_SQE`(=4)这个乘数因子，导致device侧分配的VA空间只有实际需要的1/4，造成内存访问越界。
- 修复模式：在UbConnLite构造函数中将`sqDepth_`乘以4。在DevUbConnection的构造函数和CreateJetty中增加`WQE_NUM_PER_SQE`因子。在C层VA映射长度计算增加`WQEBB_NUM_PER_SQE`因子。新增常量定义。
- 可审查性：低
- 审查规则建议：涉及硬件协议约束的乘数因子应在协议层统一封装为计算宏/函数，而非在多处手动乘算；SQ深度的"单位"应在类型或命名中体现。

### 4ebb11d13e11f550c7db74523507664332917dd2 GetKernelExecTimeoutFromEnvConfig Debug
- 根因类别：返回值类型截断
- 涉及文件：src/orion/unified_platform/resource/stream/aicpu/sqe_build_a5.cc, src/orion/unified_platform/resource/stream/aicpu/sqe_build_a5.h
- 缺陷描述：`GetKernelExecTimeoutFromEnvConfig`函数内部从env配置获取的timeout值类型为`u32`（32位），但函数返回类型声明为`u8`（8位），最大只能表示255。当配置的超时值超过255秒时，返回值被隐式截断，导致实际生效的超时时间远小于预期。
- 修复模式：将函数声明和定义的返回类型从`u8`改为`u32`。
- 可审查性：高
- 审查规则建议：编译器-Wconversion或静态分析工具应能直接检出从宽类型到窄类型的隐式截断；函数返回类型应与内部实际计算类型一致。

### 1820333425906d42255be5d180e493e8a209e372 fix calnumblocks
- 根因类别：联动计算变量对齐遗漏 + 变量名typo
- 涉及文件：src/algorithm/impl/coll_executor/coll_all_reduce/coll_all_reduce_mesh_aiv_for_910_93_executor.cc, src/algorithm/impl/coll_executor/coll_all_to_all/coll_all_to_all_mesh_aiv_for_910_93_executor.cc, src/algorithm/impl/coll_executor/coll_reduce_scatter/coll_reduce_scatter_mesh_aiv_for_910_93_executor.cc, src/platform/legacy/hccl_tbe_task/eletwise_v1.cc, src/platform/legacy/hccl_tbe_task/eletwise_v1.h, src/platform/legacy/hccl_tbe_task/eletwise_v2.cc, src/platform/legacy/hccl_tbe_task/eletwise_v2.h, src/platform/legacy/hccl_tbe_task/eletwise_v3.h
- 缺陷描述：`CalNumBlocks`函数在缩减`numBlocks`时只对`numBlocks`做了对齐因子的向下取整，但遗漏了对`minNumBlocks`做对应的向上取整，导致缩减后的`numBlocks`可能小于`minNumBlocks`触发错误返回。另外变量名误拼为`numBlockss_`（多一个s），JSON key `_const_num_blockss`与上游不一致。
- 修复模式：在三个executor的CalNumBlocks中，缩减numBlocks时同步将minNumBlocks按同一对齐因子向上取整。将成员变量`numBlockss_`重命名为`numBlocks_`，JSON key修正为`_const_block_dims`。
- 可审查性：高
- 审查规则建议：对同一函数中联动计算的变量（如numBlocks和minNumBlocks），审查时应检查所有分支是否对两者做了一致的变换。

### 39ad6c078385773b5fbc6c864657acf097a0960a bugfix URMA_JFS_DB_STATUS
- 根因类别：硬件寄存器地址常量错误
- 涉及文件：src/platform/hccp/orion/hcomm_dev/external_depends/ubengine/urma_opcode.h
- 缺陷描述：宏`URMA_JFS_DB_STATUS`的值被错误地定义为`0x0011`，与紧邻的`URMA_JFS_PI_TYPE`（也是`0x0011`）重复。两个不同语义的寄存器操作码映射到了同一个地址，导致读取DB_STATUS时实际访问的是PI_TYPE寄存器。正确值应为`0x000f`。
- 修复模式：将`URMA_JFS_DB_STATUS`的宏值从`0x0011`改为`0x000f`。
- 可审查性：高
- 审查规则建议：同一组宏常量定义中不应存在重复值，可编写静态检查规则检测同一define块中的值重复。

### b54e38524b88bed6f96e882b896559b40e4303f0 fix offload
- 根因类别：V1/V2分支逻辑错误（offload路径遗漏）
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_rank_graph.cc, src/framework/op_base/src/op_base.cc
- 缺陷描述：多个函数在函数入口处将`comm`转型为`hcclComm`并访问其方法，但这些操作应该在`HCCLV2_FUNC_RUN`宏内部的lambda中执行。原代码在lambda外部提前做了cast和使用，导致在offload模式下控制流未正确进入V2分支。此外op_base.cc中V2初始化路径缺少`HcomSetGroupTopoInfo`调用。
- 修复模式：将hcclComm的cast操作从函数顶部移入lambda内部。在op_base.cc的V2初始化路径中补充HcomSetGroupTopoInfo调用。
- 可审查性：中
- 审查规则建议：当函数存在多条件分支（如V1/V2双路径），审查时应检查每个分支是否独立完备，是否有变量的作用域泄漏到错误的分支。

### 802c0411b4e189955467040ea0a0600bd028b6f1 fix unfold op in aicpu log and engine condition
- 根因类别：条件判断不完整 + 日志变量引用错误
- 涉及文件：src/orion/framework/communicator/communicator_impl.cc
- 缺陷描述：`AllocCollOpResource`函数中engine类型检查只允许`AICPU`，但遗漏了`AICPU_TS`这种合法的engine类型，导致`AICPU_TS`类型的unfold op请求被错误拒绝并返回`HCCL_E_NOT_SUPPORT`。同时日志打印使用了已被累加额外预留空间的`scratchBufSize`而非原始的`cclBufferSize`。
- 修复模式：将条件增加`AICPU_TS`的判断。将日志中的`scratchBufSize`替换为`cclBufferSize`。
- 可审查性：高
- 审查规则建议：枚举值新增时应全局搜索所有对该枚举的条件判断确认是否需要同步更新。

### 1479e7baedb611df9c5970ee34bed633cf3b4d6e one side rmda timeout bugfix
- 根因类别：超时机制架构缺陷（async+future超时无法传导）
- 涉及文件：src/platform/common/adapter/adapter_hccp.cc, src/platform/inc/adapter/adapter_hccp.h, src/platform/resource/transport/onesided/transport_roce_mem.cc, src/platform/resource/transport/onesided/transport_roce_mem.h
- 缺陷描述：原来`ConnectImplWithTimeout`使用`std::async`+`std::future::wait_for`来实现超时控制，但内层的`QpConnect`和`ExchangeNotifyValueBuffer`本身没有接收timeout参数，使用全局默认超时值。外层future超时后内层线程仍在阻塞等待。当one-sided RDMA场景下一端已超时退出但另一端仍在等待时，出现单边超时挂起。
- 修复模式：移除整个`ConnectImplWithTimeout`方法及其async/future超时包装；改为将timeout参数逐层传递到各子操作中。
- 可审查性：中
- 审查规则建议：使用std::async+future::wait_for做超时控制时，必须确认被调用方支持取消/中断机制；优先将timeout传递到底层阻塞调用中。

### d07099d4a5ef8c3ccf662500bb06364de1726aa7 优化hcomm仓当前read & write lock
- 根因类别：并发/锁设计缺陷
- 涉及文件：src/framework/common/src/read_write_lock_base.cc, src/framework/common/src/read_write_lock_base.h, src/framework/device/debug/dfx/trace/executor_tracer.cc
- 缺陷描述：原有读写锁基于`std::mutex`+`std::condition_variable`实现，读路径也需要抢互斥锁导致读-读互斥。`executor_tracer.cc`中在循环外部持有写锁、循环内部执行`AicpuDestoryCommbyGroup`，持锁粒度过粗，可能导致长时间阻塞读者甚至死锁。
- 修复模式：将读写锁从mutex+CV方案重写为基于`std::atomic<uint64_t>`的无锁(spin-lock)方案。将写锁从循环外移到循环内，缩小临界区。
- 可审查性：中
- 审查规则建议：对"循环体外持有写锁、循环体内做IO/RPC"的模式进行自动检测，标记为持锁粒度过粗风险。

### f715f16679b7646f3105610eed55066aa684f848 fix rankGraph
- 根因类别：通信协议选择逻辑缺陷 + 日志格式错误
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_rank_graph.cc, src/framework/communicator/impl/independent_op/rank_graph/rank_graph.cc, src/framework/communicator/impl/independent_op/rank_graph/rank_graph.h, test/ut/framework/communicator/impl/independent_op/ut_hccl_independent_rank_graph.cc
- 缺陷描述：`RankGraph`在构建link时存在多个问题：日志格式字符串方括号不匹配；`GetLinks`返回的`CommLink`中endpoint的protocol未被设置为实际选定的协议；`GetCommProtocolInSameServer`中PXI_TYPE场景下对910B A+X异构且非对称rank映射的情况缺少判断，错误返回PCIE协议；同server的NETLAYER_1场景缺少RoCE协议判定逻辑；日志中用`%u`格式打印指针。
- 修复模式：修正日志格式字符串；设置link的endpoint protocol；新增`IsRoceInSameServer`方法；修正异构场景判断逻辑。
- 可审查性：高
- 审查规则建议：静态分析工具可检测printf格式说明符与参数类型不匹配，以及方括号不配对的日志字符串。

### dd7447862df754340561634dd6b29af05eae142c fix aicpu issue problem
- 根因类别：多场景适配缺陷（空指针检查顺序错误、错误码吞没、局部变量生命周期、参数校验缺失）
- 涉及文件：include/hcomm_res.h, src/framework/communicator/impl/independent_op/data_api/hccl_api_data_aicpu_ts.cc, src/framework/communicator/impl/independent_op/hccl_independent_op_channel.cc, src/framework/communicator/impl/independent_op/hccl_independent_op_mem.cc, src/framework/communicator/impl/independent_op/hccl_independent_rank_graph.cc, src/framework/inc/hccl_comm_pub.h, src/framework/next/coll_comms/api_c_adpt/coll_comm_res_c_adpt.cc, src/framework/next/coll_comms/rank/comm_mems/comm_mems.cc, src/framework/next/coll_comms/rank/my_rank.cc, src/framework/next/comms/api_c_adpt/hcomm_c_adpt.cc, src/framework/next/comms/endpoint_pairs/channels/aicpu/aicpu_ts_urma_channel.cc, src/framework/next/comms/endpoint_pairs/channels/host/host_cpu_roce_channel.cc, src/framework/next/comms/endpoint_pairs/sockets/socket_mgr.cc, src/framework/next/comms/endpoints/endpoint.h, src/framework/next/comms/endpoints/reged_mems/roce_mem.cc, src/framework/next/comms/endpoints/reged_mems/urma_mem.cc, src/framework/op_base/src/op_base.cc, src/orion/unified_platform/resource/transport/ub_mem_transport.cc, src/platform/resource/notify/rts_notify.cc
- 缺陷描述：综合性修复。核心问题包括：(1) 空指针检查在解引用之后；(2) lambda内`return HCCL_SUCCESS`覆盖了实际的错误码`ret`，导致错误被吞没；(3) `GetRemoteMem`中`remoteMemsPtr_`是局部变量导致返回后内存被释放；(4) Reduce操作缺少对不支持的dataType/reduceOp组合的参数校验；(5) 内存注册在重复注册时缺少已存在判断；(6) 日志参数传错；(7) 默认构造函数未初始化成员。
- 修复模式：修正空指针检查顺序；将`return HCCL_SUCCESS`改为`return ret`透传错误码；将局部变量提升为类成员变量；新增`IsSupportReduce`参数校验；在RegisterMemory中先Find再构造；修正日志参数；显式初始化成员。
- 可审查性：高
- 审查规则建议：检测"空指针解引用在null-check之前"模式、lambda中return HCCL_SUCCESS忽略局部ret变量、以及printf参数与格式不匹配。

### e17cfff4be9e01114ffa2f69736edbfad137d0f0 Three Bugfix
- 根因类别：异常安全 + 资源释放时use-after-free + 错误的服务对象获取
- 涉及文件：src/orion/common/utils/exception_util.h, src/orion/framework/communicator/communicator_impl.cc, src/orion/framework/dfx/task_exception/task_exception_handler.cpp, src/orion/unified_platform/ccu/ccu_device/ccu_component/ccu_component.cpp
- 缺陷描述：三个独立缺陷：(1) `TaskExceptionHandler::Process`作为异常回调入口，内部抛出C++异常会导致进程coredump，缺少顶层try-catch保护；(2) `CcuComponent::UnimportAllJetty`和`DestroyAllJetty`中释放资源后handle仍保留旧值，重入时会double free导致coredump；(3) `GetMC2AlgTaskParam`和`GetRankIdByChannelId`中获取了错误的服务对象（应使用CCU_SCHED对应的collService）。
- 修复模式：新增`TRY_CATCH_PRINT_ERROR`宏包裹Process全部逻辑；释放资源后立即将handle置0；新增`GetCcuCollService()`方法获取正确的服务对象；在Init和Deinit中添加CleanDieCkes调用。
- 可审查性：高
- 审查规则建议：异常回调/析构函数中应有顶层try-catch保护；资源释放后应立即将handle/指针置空。

### 65f0c2ba34fb7e583c283d8c88b4d323744269d0 fix get topo failed
- 根因类别：初始化时序遗漏
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：`HcclCommInitRootInfoConfigInner`中V2初始化路径创建communicator后，没有调用`HcomSetGroupTopoInfo`设置拓扑信息，导致后续通过独立算子路径查询topo时失败。另一条非V2路径已正确调用了该函数，V2分支遗漏。
- 修复模式：在V2路径的lambda中补充`HcomSetGroupTopoInfo`调用。
- 可审查性：高
- 审查规则建议：当同一函数存在多条初始化分支时，审查是否所有分支都执行了相同的必要初始化步骤（对称性检查）。

### 3c24c0fea333a02a01d5ab1e718bb20a1d611c57 fix aiv bug
- 根因类别：状态未刷新
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：在cache复用路径（`ExecOpCache`）中，`cacheInfo.resourceArgs`的`stream`字段未被更新为当前操作的stream。导致cache命中后使用旧的stream下发任务，AIV kernel可能在错误的stream上执行，造成执行序错乱或功能异常。
- 修复模式：新增一行将当前stream写入cache的资源参数；同时在日志中增加stream id打印。
- 可审查性：中
- 审查规则建议：当结构体字段在cache/复用路径中被部分更新时，审查是否所有运行时上下文字段都已刷新。

### 36c8449df5e44f7ea7068705f670b1c59db39d25 [Fix] Fix device ut
- 根因类别：API重命名后测试未同步
- 涉及文件：test/ut/device/ut_communicator_device_utest.cc
- 缺陷描述：`HcclCalcBlockDim`接口被重命名为`HcclCalcNumBlocks`（对应commit fd23d1b中的大规模重命名），但device侧的UT测试代码未同步修改，导致编译失败。
- 修复模式：将测试代码中的`HcclCalcBlockDim`调用改为`HcclCalcNumBlocks`，变量名`blockDim`改为`numBlocks`。
- 可审查性：高
- 审查规则建议：全局重命名后应自动检查所有调用点（包括测试代码）是否已同步更新，CI编译即可拦截。

### 7be3b8b8814caed0634f1a72c2cc296e2f629e0d fix dispatcher and queue notify core
- 根因类别：并发资源释放导致coredump
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/platform/comm_primitive/hccl_dispatcher_ctx.cc, src/platform/resource/dispatcher_ctx/dispatcher_ctx.cc, src/platform/resource/dispatcher_ctx/dispatcher_ctx.h
- 缺陷描述：三处并发安全问题：(1) `DispatcherCtx::Destroy()`中对`dispatcher_`指针的delete操作没有加锁保护，多线程并发析构时会double free导致coredump；(2) `DestroyDispatcherCtx`全局函数同样缺乏互斥保护；(3) `queueNotifyManager_`资源未在communicator销毁时显式置空。
- 修复模式：在Destroy中新增destroyMutex_加锁；在DestroyDispatcherCtx中新增static mutex加锁；在communicator析构路径中显式将queueNotifyManager_置空。
- 可审查性：中
- 审查规则建议：对象析构路径中涉及裸指针delete的操作，审查是否存在多线程并发调用的可能。

### 9277e0236f73991da26d0e23bfb8dfa00b13f1b0 broadcast memory bugfix
- 根因类别：内存多算（过度分配）
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：`GetOpScratchMemSize`函数在计算broadcast操作所需scratch memory时，对大数据量（count * dataTypeSize > 32MB）的分支多分配了`count * dataTypeSize`的额外内存。broadcast大数据量场景不需要额外的scratch buffer，导致显存浪费甚至OOM。
- 修复模式：删除else分支中多余的内存分配代码。
- 可审查性：高
- 审查规则建议：审查内存分配逻辑时，确认每个分支的buffer大小计算有对应的算法文档或注释支撑。

### fd23d1b6ac3640b8f66ced148d657979665ccbaf fix blockdim -> numblocks
- 根因类别：术语/命名规范化（大规模重命名）
- 涉及文件：175个文件，核心涉及externel_depends/tsch/task_struct.h, src/algorithm/pub_inc/hccl_aiv.h, src/algorithm/impl/coll_executor/coll_executor_base.cc等
- 缺陷描述：整个代码库中`blockDim`/`block_dim`这一命名与上游硬件/runtime接口的术语`numBlocks`/`num_blocks`不一致，造成语义混淆。`blockDim`在CUDA生态中表示block的维度，而这里实际含义是block的数量。
- 修复模式：全局机械性替换：blockDim -> numBlocks, block_dim -> num_blocks, CalBlockDim -> CalNumBlocks等。175个文件，纯机械替换无逻辑变更。
- 可审查性：低
- 审查规则建议：大规模重命名应通过自动化脚本执行并验证编译通过；重点审查是否有遗漏。

---

## 第2轮 (commits 21-40)

### 5422c95de1da3a343e404e0f2ddb0de297d7ccee 修复reduce精度
- 根因类别：数据类型误用 / 算术计算错误
- 涉及文件：src/platform/comm_primitive/hccl_primitive_local.cc, src/framework/communicator/impl/independent_op/data_api/hccl_api_data.cc, src/framework/communicator/impl/independent_op/hccl_independent_op_channel.cc, src/framework/common/src/config/env_config.cc, env_config.h
- 缺陷描述：`HcclLocalCopyReduce`函数中，计算reduce元素个数时使用`src->len / reduceInfo.dataType`。`reduceInfo.dataType`是枚举值（如`HCCL_DATA_TYPE_FP32`对应的枚举整数），而非该数据类型的字节大小。正确做法是`src->len / SIZE_TABLE[reduceInfo.dataType]`通过查表获取实际字节数。用枚举值代替字节大小做除法，导致计算出错误的元素个数，造成reduce结果精度错误。此外函数签名参数类型从`HcclDataType/HcclReduceOp`改为`HcommDataType/HcommReduceOp`，在构造`HcclReduceInfo`时需要`static_cast`，之前直接传入不同枚举类型，类型不安全。同时新增`IsSupportReduce`校验函数，防止不支持的类型组合走进reduce路径。
- 修复模式：将`reduceInfo.dataType`替换为`SIZE_TABLE[reduceInfo.dataType]`；函数签名参数类型修正+显式static_cast；新增IsSupportReduce前置校验。
- 可审查性：高
- 审查规则建议：当dataType枚举值被用作除数或乘数时，检查是否应通过SIZE_TABLE或sizeof查表获取实际字节数。对`/ dataType`这类直接用枚举做算术运算的代码发出警告。

### e25c648461c962e8add590a257a18fdb32ac47de V2 sync patch from open
- 根因类别：功能缺失 / V2适配层遗漏
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_op_mem.cc, src/framework/hcom/hcom.cc, src/framework/hcom/hcom_private_v2.h, src/framework/op_base/src/op_base.cc, src/framework/op_base/src/op_base_mc2.cc, src/framework/op_base/src/op_base_v2.h
- 缺陷描述：多个API函数（`HcclGetHcclBuffer`、`HcclCommGraphUnloadTask`、`HcclGetAicpuOpStreamAndNotify`）及一组MC2相关接口缺少V2版本的分发调用。在`OPEN_BUILD_PROJECT && ORION_MODE`编译条件下，这些函数应优先通过`HCCLV2_FUNC_RUN`宏调用V2实现（weak symbol动态分发），否则在V2运行时环境下会走到旧的V1逻辑，功能或行为不正确。
- 修复模式：在受影响函数入口处通过条件编译块插入`HCCLV2_FUNC_RUN(XxxV2(...))`调用；在V2头文件中声明对应的weak symbol函数。
- 可审查性：中
- 审查规则建议：当新增public API函数时，检查是否需要在V2头文件中声明对应的weak symbol并在函数体内添加`HCCLV2_FUNC_RUN`分发。每个新增`Hccl*`/`Hcom*`外部接口都需确认V2适配。

### b8de68b9997be7c5b251e13296ce5068da99bceb fix OpRetry timeout
- 根因类别：超时值硬编码导致配置不一致
- 涉及文件：src/framework/cluster_maintenance/recovery/operator_retry/opretry_agent.cc, opretry_base.cc, opretry_base.h, opretry_server.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：OpRetry机制中所有发送/接收/等待操作的超时时间硬编码为`OP_RETRY_SEND_RECV_TIMEOUT`(205秒)，而用户可通过`HCCL_LINK_TIMEOUT`环境变量配置链路超时。当用户配置的链路超时大于默认200秒时，OpRetry超时会先触发，误判超时失败。AICPU端也是硬编码200秒，不跟随配置。
- 修复模式：将超时计算改为`std::max(GetExternalInputHcclLinkTimeOut(), OP_RETRY_SEND_RECV_TIMEOUT) + OP_RETRY_WAIT_AICPU_TIMEOUT`，取用户配置和默认值的较大者再加裕量。
- 可审查性：中
- 审查规则建议：当系统存在可配置超时参数时，检查依赖该超时的所有下游超时是否也随之调整。搜索所有硬编码超时常量，检查是否应从全局配置获取。

### 4425a342cca7fee37a3f64120b2e1214b176272d Fix RaGetOpRight log flooding
- 根因类别：日志刷屏 / 高频操作无日志抑制
- 涉及文件：src/platform/hccp/rdma_agent/adapter/ra_adp.c, ra_adp.h, async/ra_adp_async.h
- 缺陷描述：`RaGetOpRight`函数每次调用都打印debug日志。对于高频opcode（如`RA_RS_GET_SOCKET`）会导致日志大量刷屏，影响可读性和性能。
- 修复模式：引入日志抑制机制。在`RaHdcOpSec`中新增`lastOpcode`和`lastOpcodeCnt`字段跟踪连续相同opcode计数。新增`RaIsOpcodeLogSuppressed`判断是否抑制，opcode切换时补打汇总日志。
- 可审查性：高
- 审查规则建议：在高频调用路径（循环内、中断处理、轮询函数）中出现无条件debug日志打印，应审查是否需要日志抑制（采样/计数/限频）。

### 103755111cffd92cf0e99341d9d49b9ab1951ab4 [Fix] Fix SetDispatcherCtx
- 根因类别：上下文设置时机错误 / TLS变量未正确初始化
- 涉及文件：src/framework/communicator/impl/independent_op/data_api/hccl_api_data.cc, src/framework/device/framework/aicpu_communicator.cc, aicpu_communicator.h, src/platform/comm_primitive/hccl_dispatcher_ctx.cc
- 缺陷描述：`SetDispatcherCtx`将`DispatcherCtxPtr`设置到TLS变量`gDispatcherCtx`中。原来放在`RegisterOpInfo`内调用，但`RegisterOpInfo`并非每次线程操作入口——AICPU工作线程通过`HcommAcquireComm`获取通信域时不经过`RegisterOpInfo`，导致该线程TLS未设置，后续通信操作找不到正确的dispatcher上下文。
- 修复模式：从`RegisterOpInfo`中删除`SetDispatcherCtx`调用，新增`HcclCommAicpu::SetDispatcherCtxOnThread()`方法，在`HcommAcquireComm`的AICPU路径中显式调用。
- 可审查性：中
- 审查规则建议：对TLS变量的初始化，审查所有可能的线程入口点是否都正确设置了TLS。识别TLS变量的所有写入点和读取点，验证每个线程执行路径在读取前必经至少一个写入点。

### 54e5e41bade695fbcebb00c5305a703e02d90eb5 RPING phyId bugfix
- 根因类别：条件分支逻辑被条件编译无条件覆写
- 涉及文件：src/platform/hccp/rdma_service/rs_ping.c — `RsPingInit`函数
- 缺陷描述：`#ifdef CONFIG_CONTEXT`条件编译块中，`phyId`被无条件赋值为`attr->ub.phy_id`，覆盖了上方根据`attr->protocol`做的RDMA/非RDMA分支判断。当`CONFIG_CONTEXT`开启且`protocol == PROTOCOL_RDMA`时，本应使用`attr->dev.rdma.phyId`却错误使用了`attr->ub.phy_id`，导致后续`RsGetRsCb`拿到错误的物理ID。
- 修复模式：将条件编译块内的无条件赋值改为与上方一致的三元表达式，恢复protocol分支语义。
- 可审查性：高
- 审查规则建议：当`#ifdef`块内对同一变量做赋值时，检查是否无意覆盖了上方已有的条件分支逻辑。

### 68b477c593eab5691b1e3abcc10f3b856af5f181 fix calblockdim log
- 根因类别：日志格式不一致 / 拼写错误（前序修复的遗留问题）
- 涉及文件：17个coll_executor下的CalBlockDim相关executor实现文件
- 缺陷描述：上一轮修复(4310539)把日志从"is less than need"改为"is invalid, at least need[%u]"，但`need`和`[%u]`间缺少空格。3个910_93文件中存在拼写错误"at lest"及格式不统一。
- 修复模式：所有文件`need[%u]` -> `need [%u]`加空格；910_93文件修正"at lest" -> "at least"并统一格式。
- 可审查性：高
- 审查规则建议：日志字符串格式化参数前应有空格分隔；同一函数簇的错误日志应使用统一模板。

### 8d96755e4e51f8b5a94536732df21c1d9953764b fix fail
- 根因类别：日志文案不规范（`fail` vs `failed`时态不一致）
- 涉及文件：src/platform/hccp/common/dl_hal_function.c, file_opt.c, network_comm.c
- 缺陷描述：多处错误日志使用`fail`（动词原形），不符合英文日志规范，应为`failed`。涉及`DlHalInit`、`ReadFileToBuf`等函数共10处。
- 修复模式：批量将日志字符串中的`fail`替换为`failed`，纯文本替换无逻辑变更。
- 可审查性：高
- 审查规则建议：日志用词lint规则：检测`\bfail[^e]`模式提示应使用`failed`。

### 0cce66771de52f74eee898692e9b6ed5189e7924 bugfix ICSL recv_sge_idx out of range
- 根因类别：Off-by-one边界检查错误
- 涉及文件：src/platform/hccp/rdma_service/rs_ping_roce.c — `RsPongPostSend`函数
- 缺陷描述：`recvSgeIdx`访问sge数组，边界检查条件为`recvSgeIdx > sgeNum`，但数组索引有效范围是`[0, sgeNum-1]`，当`recvSgeIdx == sgeNum`时越界但未被拦截。经典off-by-one错误。
- 修复模式：`>` 改为 `>=`。
- 可审查性：高
- 审查规则建议：对数组索引边界检查使用`index > array_size`模式时应发出警告，建议使用`index >= array_size`。

### 4310539da0224d71934e1b42a833950eb29ef321 fix calblocidim log
- 根因类别：日志文案不清晰（语义模糊）
- 涉及文件：15个coll_executor下的CalBlockDim函数实现文件
- 缺陷描述：CalBlockDim函数中日志"aivCore[%u] is less than need[%u]"语义模糊，运维人员难以快速理解问题。
- 修复模式：统一改为"is invalid, at least need[%u]"，使错误描述更明确。（此次遗留need[%u]缺少空格问题被后续68b477c修复）
- 可审查性：中
- 审查规则建议：错误日志模板应包含"原因+期望值+实际值"三要素；同一函数簇使用统一日志模板。

### 6da8976f379cfd7b6baaf3e2512e6166bb2bc2e3 Delete debug logs to avoid log flooding
- 根因类别：日志过量(log flooding)
- 涉及文件：src/platform/hccp/rdma_agent/hdc/async/ra_hdc_async.c, src/platform/hccp/rdma_service/rs.c, src/platform/hccp/rdma_service/rs_socket.c
- 缺陷描述：4处`hccp_dbg()`调用位于高频执行路径上（RDMA消息收发、context切换、socket连接建立），正常运行时产生海量debug日志，导致日志洪泛影响性能和可用性。
- 修复模式：纯删除——移除热路径上的debug日志和仅服务于日志的临时变量声明，删7行无新增。
- 可审查性：中
- 审查规则建议：高频调用路径中的debug日志应受采样/限频保护，或要求"预期QPS > N的函数中禁止无条件debug日志"。

### 66f33b831447db1b4db72cfa833b216b629dc2ec fix step recalculation
- 根因类别：状态机逻辑缺陷 / 超时常量硬编码 / 缺少条件分支
- 涉及文件：opretry_agent.cc, opretry_base.h, opretry_manager.cc, opretry_manager.h, opretry_server.cc, hccl_communicator_host.cc
- 缺陷描述：operator retry的"等待恢复"状态机存在多个问题：(1)超时时间用硬编码常量10秒，实际应从外部配置获取；(2)`ExitWaitResumeState`未区分"是否存在开启backup link的通信域"条件，无backup link时状态机会卡死。
- 修复模式：新增条件分支(early return)；硬编码常量替换为动态配置值；新增`haveCommEnableBackupLink_`状态追踪字段和全局原子计数器。
- 可审查性：低
- 审查规则建议：超时值禁止硬编码、必须从配置获取；状态机每个状态的ProcessEvent应显式处理所有可能的前置条件组合。

### 6596fc1f1fa7894eb6d4c020c125eff74089cf21 fix p2p transport
- 根因类别：变量未初始化 / 遗漏赋值
- 涉及文件：src/platform/resource/transport/host/transport_p2p.cc — `FillExchangeDataTotalSize`函数
- 缺陷描述：P2P跨进程传输场景下，`ipcMemDataSize`变量在进入跨进程分支后未被赋值就用于`exchangeInfoSize_.ipcMenSize`计算。缺少`ipcMemDataSize = HCCL_IPC_MEM_NAME_LEN + sizeof(u64) + sizeof(u64)`赋值。导致IPC内存交换数据大小计算错误。
- 修复模式：在条件分支内新增一行变量赋值语句。
- 可审查性：高
- 审查规则建议：静态分析检测"变量在某条件分支中使用前未赋值"；审查变量声明与首次使用之间所有路径是否都有赋值。

### 8f030cdbcffbaafae6a6c9465d0ad4e095052194 fix bug of condition check
- 根因类别：条件表达式拼写错误（自身比较）
- 涉及文件：src/platform/task/dispatcher_aicpu.cc — `MemcpyRtsq`函数
- 缺陷描述：边界检查条件写成`sqeContextBuffer->tailSqeIdx > sqeContextBuffer->tailSqeIdx`（变量与自身比较），恒为false，导致边界检查完全失效。正确应为`> HCCL_SQE_MAX_CNT`。
- 修复模式：右侧操作数从自身改为`HCCL_SQE_MAX_CNT`。
- 可审查性：高
- 审查规则建议：经典"自身比较"缺陷，大多数静态分析工具(clang-tidy `misc-redundant-expression`、Coverity、cppcheck)可自动检测`x > x`恒false表达式。建议CI启用此类检查。

### 7a3d6fa691b2f497a7893f0caad45290148447b8 fix workspace calc scratch mem size overflow
- 根因类别：整数溢出 / 类型宽度不足
- 涉及文件：src/framework/hcom/hcom.cc — `GetOpScratchMemSize`函数
- 缺陷描述：`count`变量声明为`u32`，但`hcomOpParam->count`在大规模集合通信下可能超过32位范围。用`count`参与scratch memory大小计算时整数溢出，导致内存分配不足。
- 修复模式：`count`类型从`u32`改为`u64`。
- 可审查性：高
- 审查规则建议：所有用于内存大小计算的变量应使用`u64`/`size_t`类型；静态分析检测"窄类型接收宽类型值"的隐式截断。

### f64f64985da9e7a86fe6afa9ed0ca3393ebdb34b fix p2p OOM
- 根因类别：资源重复管理——子内存区域被父区域已覆盖仍被独立做IPC注册/映射/释放，导致内存泄漏最终OOM
- 涉及文件：src/platform/resource/transport/host/transport_p2p.cc, transport_p2p_pub.h
- 缺陷描述：P2P传输中，当input/output内存区域已包含在`machinePara_.mem[0]`（CCL buffer整块大内存）范围内时，原代码仍对input/output分别做独立IPC注册（SetMemoryName）、跨进程交换（ConstructIpcMemInfoForSend）和析构释放（CloseIpcMem/DestroyIpcMem），同一段物理内存被重复映射和管理，造成IPC内存资源泄漏最终OOM。
- 修复模式：引入`isMemInclude_`标志位检测地址范围包含关系。若子区域在父区域内：析构跳过子区域释放；交换数据只传size+offset而非独立ipcName；解析端通过offset从已映射的远端指针计算。
- 可审查性：中
- 审查规则建议：当同一资源存在多种生命周期管理路径时，审查是否存在"子区域已被父区域管理但仍被独立管理"的重复管理。

### 486ab87aadb58fb39bd07d939a846dd4694a0b94 fix aiv core
- 根因类别：回归缺陷——前序commit在fallback路径中错误添加不适用的初始化调用
- 涉及文件：src/algorithm/impl/operator/coll_alg_operator.cc
- 缺陷描述：`SelectAlg`的DefaultExecutor分支（无匹配算法执行器时的fallback），前序commit 87da187b3添加了`SetExecutorAttr(param)`调用。但DefaultExecutor路径的executor状态与正常路径不同，调用SetExecutorAttr会访问不正确的状态，在AIV场景下触发coredump。
- 修复模式：删除DefaultExecutor分支中的`CHK_RET(SetExecutorAttr(param))`。
- 可审查性：高
- 审查规则建议：在error/fallback/default路径中新增函数调用时，需验证被调用函数对当前对象状态的前置条件是否满足。正常路径和fallback路径的executor可能处于不同状态，不可盲目统一添加调用。

### f6e1c1c2597aed61c1db3ecdc3087a42ac1c1564 fix one sided init bugs
- 根因类别：无条件调用在特定环境配置下会失败的函数
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：`CheckOneSidedBackupAndSetDevId`中，`hrtGetDeviceIndexByPhyId`被无条件调用。当`ASCEND_RT_VISIBLE_DEVICES`只配置一个设备时，backup设备物理ID无法转换为逻辑ID（设备不在可见列表中），调用返回错误导致one-sided backup检查失败。实际该调用只在`isOneSidedTaskAndBackupInitA3`为true时才需要。
- 修复模式：将调用移到条件分支内部，按需调用。
- 可审查性：高
- 审查规则建议：若API调用结果只在特定条件下被使用，该调用应放在该条件分支内部（延迟执行/按需调用原则）。

### a84b98a0274795d52601cabad6f984c38947d425 fix errorMap coredump
- 根因类别：`std::map::at()`对不存在key抛异常导致coredump
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：`HcclGetErrorString`使用`errorMap.at(code)`查找错误码字符串，但errorMap未覆盖所有HcclResult值（`HCCL_E_SUSPENDING`、`HCCL_E_OPRETRY_FAIL`、`HCCL_E_OOM`未收录）。传入未收录code时at()抛异常导致coredump。
- 修复模式：补充缺失错误码；将`at()`替换为`find()`+迭代器判空，找不到时返回`"unknown err"`。
- 可审查性：高
- 审查规则建议：禁止在可能接收不可控输入的场景使用`std::map::at()`，应使用find()+迭代器检查；每当新增枚举值时检查所有使用该枚举的map/switch是否同步更新。

### 87da187b365238c4425bb30c99167976ca4d87c3 fix calblockdim log & fix word spell
- 根因类别：(1)executor属性未初始化——查找executor后遗漏SetExecutorAttr调用；(2)executor_过早置空——在依赖它的CalBlockDim调用之前无条件执行`executor_ = nullptr`；(3)拼写错误
- 涉及文件：src/algorithm/impl/operator/coll_alg_operator.cc（核心逻辑），src/algorithm/base/alg_template/temp_alltoallv/alltoallv_direct_fullmesh.cc等（拼写修正约110个文件）
- 缺陷描述：`CalBlockDim`、`SelectAlg`、`SelectAlgFor91093WithCoreLimit`三个函数中，通过名字查找executor后缺少`SetExecutorAttr(param)`调用，导致executor属性未正确设置。`SelectAlgFor91093WithCoreLimit`中`executor_ = nullptr`在CalBlockDim调用之前无条件执行，但CalBlockDim内部需要executor_。注意此commit在SelectAlg的DefaultExecutor分支也加了SetExecutorAttr，后被486ab87aa回滚。
- 修复模式：补充`CHK_RET(SetExecutorAttr(param))`；将`executor_ = nullptr`从无条件位置移入错误处理分支；大规模拼写修正。
- 可审查性：中（逻辑）/ 高（拼写）
- 审查规则建议：对象被创建/查找后检查是否调用了所有必要的配置函数；状态变量清除操作不应在依赖该状态的调用之前无条件执行。

---

## 第3轮 (commits 41-60)

### 6b7ca887dcac4a97e0cf4a121312379cffc2afb9 c25 bugfix合入主线
- 根因类别：多类混合——(1)资源计算不精确 (2)图模式/单算子模式适配缺失 (3)API命名不规范
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc（核心修复：图模式AIV展开精确计算subStreamNum）, src/framework/hcom/hcom.cc, src/algorithm/impl/coll_alg_utils.cc, pkg_inc/hccl/hcom.h, 多个API重命名文件
- 缺陷描述：包含多个独立修复：(1) Hcom WorkflowMode(图编译模式)下AIV展开时，subStreamNum的计算是粗放的（没有考虑AIV算法实际需要的stream数量），导致资源请求不精确。(2) alltoallv在selectAlg阶段，GetStreamCaptureInfo在非acl graph场景(isOpbase为false)被调用会失败，需要用isOpbase条件保护。(3) 对不支持AIV选算的opType（REDUCE_SCATTER_V、ALLGATHER_V等）缺少提前返回路径。(4) HcclGetNetLayers等6个API重命名（非缺陷修复）。
- 修复模式：增加ifAiv和algName参数在整条调用链中透传，通过CalcResRequest精确计算streamNum；离线编译AIV模式下streamNum设为0；增加isOpbase条件判断保护GetStreamCaptureInfo调用。
- 可审查性：低（聚合提交混入API重命名，极难审查）
- 审查规则建议：聚合提交应拆分为独立bugfix和重构提交；函数签名变更涉及跨层透传时检查调用链每一层是否正确传递新参数；图模式与单算子模式的行为差异应有明确条件分支。

### 578c16b47d51468c2c1e051dca74577caec5f739 alg info print bugfix
- 根因类别：状态覆盖导致条件丢失（use-after-overwrite）
- 涉及文件：src/algorithm/impl/operator/coll_alg_operator.cc, src/algorithm/pub_inc/coll_alg_operator.h
- 缺陷描述：CollAlgOperator::SelectAlg中，先执行`algDesc = executor_->GetAlgDesc()`整体覆盖algDesc，然后用`algDesc.isLastSelect`判断是否打印维测日志。但isLastSelect本应取覆盖前调用者传入的值，覆盖后读到的是executor内部状态，导致日志打印条件错误。
- 修复模式：在algDesc被覆盖前先保存`bool isLastSelect = algDesc.isLastSelect`，后续用保存值判断。将展开模式字符串获取逻辑抽取为独立方法GetOpExpansionStr。
- 可审查性：高
- 审查规则建议：当结构体被整体赋值（覆盖）后，检查后续代码是否依赖覆盖前的旧字段值；"先覆盖聚合状态，再读子字段做判断"是典型的use-after-overwrite缺陷模式。

### dad59061245de2024339447cea1a48757ef1a145 fix_hccp_init
- 根因类别：编译宏替代运行时检测
- 涉及文件：src/platform/hccp/rdma_service/rs.c（核心）, src/platform/hccp/hccp_service/main.c, src/platform/hccp/rdma_service/dl_ibverbs_function.c, rs_drv_socket.c, rs_epoll.c, rs_socket.c, rs.h
- 缺陷描述：大量使用编译宏(CONFIG_CONTEXT, CONFIG_SSL, NOT_INIT_IBVERBS)条件编译RDMA和SSL相关代码。某些芯片型号(如310p)不支持RDMA，但原代码用编译时宏而非运行时检测来区分，导致：(1)在不支持RDMA的芯片上执行RsApiInit会失败；(2)SSL功能被编译时排除而非运行时控制。
- 修复模式：新增RsGetIsRdmaSupported(int devId)函数，通过DlHalGetChipInfo获取芯片型号做运行时判断；移除所有CONFIG_SSL/NOT_INIT_IBVERBS宏包裹，使代码始终编译进去由运行时sslEnable控制。
- 可审查性：中
- 审查规则建议：编译宏不应用于控制与硬件型号相关的运行时行为，应改为运行时检测；fnmatch模式匹配芯片名称容易遗漏新型号，关注枚举完整性和默认分支处理。

### 3d42ffb53ef8a3a3654047051df475fd7ae2b6e3 fix HcomSelectAlg bug
- 根因类别：多类混合——(1)空指针解引用 (2)算子类型未覆盖 (3)非AIV算法缺乏提前退出路径
- 涉及文件：src/algorithm/impl/operator/alltoall_operator.cc, src/algorithm/impl/operator/coll_alg_operator.cc, src/framework/communicator/impl/hccl_communicator_host.cc, pkg_inc/hccl/hcom.h, src/framework/hcom/hcom.cc, 多个头文件
- 缺陷描述：(1) IsBufferSatisfyAlltoAllAivCondition中直接对param.All2AllDataDes.sendCountMatrix解引用，但alltoallv场景下sendCountMatrix为nullptr导致crash。(2) HcclSelectAlg只处理HCCL_CMD_ALLTOALL一种opType，ALLTOALLV/ALLTOALLVC/ALLGATHER_V/REDUCE_SCATTER_V等未覆盖。(3) algName为空（非AIV算法）时仍尝试用空名查找executor导致失败。(4) 图编译选择AIV阶段不应执行完整alltoall算法选择流程。
- 修复模式：先取sendCount再按条件判断sendCountMatrix非空时才解引用；扩展OpParam填充逻辑覆盖所有opType；algName为空时用"SendExecutor"兜底；增加ifCompileForAiv标记让AlltoAll提前返回。
- 可审查性：中
- 审查规则建议：解引用指针前必须做null检查；switch/if-else覆盖opType时检查是否覆盖所有枚举值；图编译阶段与运行时执行阶段需要明确的边界标记。

### 3a23fa6f6892d25f5cb34d2790f3e0c884a1b4d8 Fix issue of missing profiling info
- 根因类别：缓存命中路径遗漏副作用（profiling上报）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc, src/platform/common/unfold_cache/op_unfold_cache_entry.cc/h, src/platform/task/dispatcher_aicpu.cc, src/platform/task/dispatcher_aicpu_pub.h
- 缺陷描述：OpUnfoldCache缓存命中路径直接从缓存取SQE下发RTSQ，跳过了正常展开流程中的profiling信息采集和上报。缓存命中路径没有调用InvokeKfcHandler(kSetProfTimeStart)、没有记录SQE刷新时间戳、没有触发TaskProfilingCallBack上报，导致profiling L1信息丢失。
- 修复模式：在缓存命中入口添加InvokeKfcHandler调用保持与cache miss路径一致；UpdateAndGetSqeArray中增加profTimestamps记录；MemcpyRtsq中增加完整profiling上报逻辑（SQE ring buffer写入+callback上报）。注意代码中疑似存在自比较bug（tailSqeIdx > tailSqeIdx恒false）。
- 可审查性：低
- 审查规则建议：引入缓存机制时，必须列出正常路径(cache miss)中的所有副作用（side effects），逐项检查缓存路径是否都覆盖；profiling是典型被遗漏的副作用。

### 1844b8232208fcffe2d5e2676282ddf4ad700d0a Revert "ars code"
- 根因类别：特性代码质量不达标导致整体回退（回归缺陷）
- 涉及文件：51个文件，删除2283行，恢复179行。核心涉及算法executor、拓扑提取器、算法配置器
- 缺陷描述：原始"ars code"提交引入完整ARS(Adaptive Ring Size)通信算法特性：新增3个ARS executor、修改ring executor核心路径KernelRun（通信域获取从局部变量改为成员变量缓存模式）、修改SetTopoInfoForLevel2拓扑判断条件。大量侵入已有ring executor核心路径导致原有场景出现回归。成员变量缓存通信域模式引入状态污染风险。
- 修复模式：完全revert，撤销ARS特性全部修改。
- 可审查性：低（50+文件2000+行的大型特性提交难以充分review）
- 审查规则建议：大型特性提交(50+文件)必须拆分为独立可审查的子PR；对已有executor核心路径的修改必须有集成测试覆盖原有场景；将通信域缓存到成员变量需警惕状态生命周期问题。

### 7fccc101b5a3469e7161174c47e8db52cfd33ff2 fix version check and cmake release flags
- 根因类别：构建配置错误 + 版本兼容性检查策略过严
- 涉及文件：CMakeLists.txt, scripts/package/common/sh/version_compatiable.inc, version.info
- 缺陷描述：两个独立问题：(1) CMake在Release模式下自动注入`-O3 -DNDEBUG`等默认flags，但项目自管编译选项，未清空CMAKE_C_FLAGS_RELEASE/CMAKE_CXX_FLAGS_RELEASE导致flags冲突。(2) version_compatiable.inc中版本不兼容检查用ERROR+return 1阻止安装，但依赖包版本不完全匹配时应只警告。
- 修复模式：(1) 在CMakeLists.txt中显式set清空release flags。(2) ERROR降级为WARNING，删除return 1。(3) version.info中版本约束从范围改为精确匹配。
- 可审查性：高
- 审查规则建议：项目自管编译选项时应检查是否处理了CMake内置的默认flags；安装脚本中ERROR+return 1会直接阻断安装，修改前需确认是否应为硬性约束。

### 8a3e9b4712185de2e12e5c3877cc2d7d1ea1395b OOM Patch
- 根因类别：错误码区分不足 / OOM场景未特化处理（非严格缺陷，属错误处理增强）
- 涉及文件：src/platform/common/adapter/adapter_rts.cc
- 缺陷描述：hrtMalloc调用aclrtMallocWithCfg后，只有通用错误上报(EI0007)，未区分OOM(ACL_ERROR_RT_MEMORY_ALLOCATION)场景，统一返回通用错误码。调用者无法针对OOM做特殊处理（如释放缓存重试），也不利于故障定位。
- 修复模式：在通用错误检查之前新增OOM专用检查：RPT_ENV_ERR上报EI0011错误（携带memory_size）；CHK_PRT_RET在OOM时返回HCCL_E_OOM专用错误码。
- 可审查性：高
- 审查规则建议：外部内存分配API应区分OOM与其他分配失败，返回不同错误码；错误上报应包含请求的资源大小信息。

### 7da764406d0538065ec6ffc8862b72807d68da6f fix HcomSelectAlg bug (revert of a2b86b3d)
- 根因类别：回退引入bug的提交（revert a2b86b3d的MR !609）
- 涉及文件：与a2b86b3d完全相同的13个文件
- 缺陷描述：a2b86b3d引入的问题被此commit完全revert：(1)空指针解引用——IsBufferSatisfyAlltoAllAivCondition中无条件解引用sendCountMatrix；(2)isLastSelect状态被executor_->GetAlgDesc()覆盖导致维测日志条件丢失；(3)公开API增加void* counts参数破坏兼容性。
- 修复模式：完全revert，从13个文件中删除void* counts参数、ifCompileForAiv标记，恢复null检查和isLastSelect保存逻辑。
- 可审查性：中
- 审查规则建议：公开API签名变更需评估兼容性；声称"fix"但包含大量feature修改的提交应被review退回要求拆分。

### a2b86b3d5e69293bc431bc1de9e795a94bd92efd fix HcomSelectAlg bug (引入新bug后被revert)
- 根因类别：空指针解引用 + 状态覆盖（声称fix但引入新问题）
- 涉及文件：pkg_inc/hccl/hcom.h, src/algorithm/impl/operator/alltoall_operator.cc, coll_alg_operator.cc, hccl_communicator_host.cc, hcom.cc等13个文件
- 缺陷描述：这个commit声称修复HcomSelectAlg bug但实际引入了新问题：(1) IsBufferSatisfyAlltoAllAivCondition中将原有的先取sendCount再条件解引用sendCountMatrix改为无条件解引用sendCountMatrix，非alltoallvc场景下sendCountMatrix为nullptr导致crash；(2) algDesc整体覆盖后isLastSelect字段丢失调用者语义；(3) 公开API签名变更未经兼容性评审。最终被7da76440 revert。
- 修复模式：N/A（此提交本身引入问题，被后续revert）
- 可审查性：中
- 审查规则建议：删除null检查guard是高危操作，review时必须验证被解引用指针在所有调用路径上不会为null；标记为"fix"但包含API签名变更的commit应被要求拆分。

### 8b16d98e70a1970f821c6702d3af5853690be2b4 fix aclgraph launch in order: fixing the issue of exhausted stream resources
- 根因类别：资源泄漏 / 流资源管理策略错误
- 涉及文件：src/framework/common/src/order_launch/order_launch.cc/h, src/framework/communicator/impl/hccl_communicator_host.cc, order_launch/CMakeLists.txt
- 缺陷描述：原实现aclgraphStreamMap_用unordered_map<u64, Stream>按modelId维护流，每个modelId创建新Stream并AddStreamToModel绑定，modelId数量增多时流资源耗尽。此外AclgraphLaunchInOrder函数开头直接return HCCL_SUCCESS（死代码跳过），LaunchInOrder被#ifndef CCL_KERNEL_AICPU包裹导致aicpu模式下函数体为空。
- 修复模式：将按modelId创建流的map替换为全局唯一的unique_ptr<Stream>；用event机制(aclrtRecordEvent+aclrtStreamWaitEvent)替代AddStreamToModel实现流间同步；拆分为双向同步两个函数；新增DestoryRes()集中管理资源销毁。
- 可审查性：中
- 审查规则建议：检查stream/event等硬件资源是否有数量上限，避免按ID无限增长的map管理有限资源；审查是否存在return HCCL_SUCCESS等死代码跳过整个函数逻辑。

### 7999a2c540a0268f7d5fb00e20b3490007cbb1dc fix cleancode & dfx enhance
- 根因类别：(1)代码风格(C-style cast) (2)错误诊断信息不足
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_op_channel.cc, hccl_independent_rank_graph.cc（cleancode）, src/platform/common/adapter/adapter_rts.cc（dfx）
- 缺陷描述：(1) 使用C-style的(uint8_t *)指针转换，应为reinterpret_cast。参数命名channelDescList/listNum不如channelDescs/channelNum语义清晰。(2) aclrtIpcMemGetExportKey失败时错误信息无可能原因提示，用户无法定位根因。
- 修复模式：C-style cast改为reinterpret_cast；变量重命名；错误日志末尾追加"Possible Cause: The memory addr or size is not aligned with pagesize."
- 可审查性：高
- 审查规则建议：C++代码应使用reinterpret_cast/static_cast而非C-style cast；错误日志应包含可能原因提示(Possible Cause)。

### d8400fd235598badfc652fb0f1be01a8e673d773 fix hcom dumpinfo bug
- 根因类别：悬垂指针 / 生命周期管理错误
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：HcomGetandClearOverFlowTasks中将局部vector<HcclDumpInfo>的.data()指针赋给输出参数*hcclDumpInfoPtr。函数返回后局部vector析构，内部buffer释放，调用方通过输出指针访问的是已释放内存——经典悬垂指针。
- 修复模式：改为malloc分配堆内存，memcpy_s将vector数据拷贝到堆内存，再将堆内存指针赋给输出参数。（注：修复引入了新问题——malloc出的内存由谁free？可能存在泄漏。）
- 可审查性：高
- 审查规则建议：永远不要将局部vector的.data()指针通过输出参数传给调用方；如果必须通过指针输出数据，要么使用malloc(调用方负责free)，要么使用外部传入的buffer，并在文档/注释中明确所有权转移。

### 58d588bffa3ede0ed9e10911931d12709dbdb635 bug fix
- 根因类别：空指针检查缺失 / 入参校验遗漏
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_op_mem.cc
- 缺陷描述：HcclGetHcclBuffer函数检查了buffer == nullptr，但没检查comm == nullptr。下一行立即static_cast<hccl::hcclComm*>(comm)并调用hcclComm->GetIdentifier()，comm为nullptr时导致崩溃。
- 修复模式：在buffer空检查之后、static_cast之前增加CHK_PRT_RET(comm == nullptr, ...)空指针检查。
- 可审查性：高
- 审查规则建议：所有公开API函数的指针参数都必须在入口做空指针检查，且在任何解引用之前完成；commit message不应只写"bug fix"，应描述具体修复内容。

### 345be6c4a1b96a1e5071fb826f35236b8c574e40 fix vaild
- 根因类别：拼写错误(typo) — invaild → invalid
- 涉及文件：src/platform/resource/transport/host/transport_roce_pub.h（生产代码枚举值OP_INVAILD），20个测试文件
- 缺陷描述：全代码库中"invalid"被拼写成"invaild"（a和l位置颠倒），包括生产代码中的枚举值OP_INVAILD和SetAicpuNotifyInvaild函数名。commit message本身也拼错了（"fix vaild"应为"fix invalid spelling"）。
- 修复模式：全局搜索替换invaild→invalid。注意测试代码中重命名后可能与已有invalidIp变量命名冲突。
- 可审查性：高
- 审查规则建议：在CI中集成拼写检查工具(如cspell)；公开标识符修改需评估API兼容性影响。

### 6efad9efdb58f63970fa89aab0c0a3b1e6f627cc fix hcomm tlv
- 根因类别：新消息类型处理缺失（非严格缺陷，属功能增加）
- 涉及文件：src/platform/hccp/rdma_service/tlv/rs_tlv.c
- 缺陷描述：TLV请求处理模块缺少对TLV_MODULE_TYPE_CCU模块类型的支持。原来CCU类型的请求会走到default分支报"module type error"。新增rs_ccu_request()处理MSG_TYPE_CCU_INIT和MSG_TYPE_CCU_UNINIT消息。在CONFIG_CONTEXT宏保护下。
- 修复模式：在switch-case中新增TLV_MODULE_TYPE_CCU分支，封装独立的rs_ccu_request()处理函数。
- 可审查性：中
- 审查规则建议：新增枚举值或消息类型时，检查所有相关switch-case是否同步更新；commit message应区分"fix"和"feature"。

### 02d62eefaa701397c4fab7a3119e9fbf61c611f2 fix log syntax error
- 根因类别：日志拼写错误 / 函数名拼写错误
- 涉及文件：11个文件，包括comm_base.cc, coll_comm_executor.cc, comm_config.cc, hccl_communicator.h, hccl_communicator_attrs_host.cc, hccl_communicator_host.cc/device.cc, network_manager.cc, ra_hdc_ping.c, graph_ctx_mgr.cc/common.cc
- 缺陷描述：批量修正两类拼写问题：(1) invaild→invalid（日志和函数名SetAicpuNotifyInvaild→SetAicpuNotifyInvalid）；(2) viable→variable（"set by environment viable."应为"set by environment variable."，语义错误）。
- 修复模式：全局搜索替换，同步修正函数声明和定义。
- 可审查性：高
- 审查规则建议：日志字符串应通过拼写检查；函数命名应在定义时即通过拼写检查。注意viable→variable不仅是拼写错误更是语义错误。

### 543628cf426ac1485eaaeca41c386028cc6b5ce2 [Fix] Add HCCL_ENV
- 根因类别：日志可观测性不足 / 日志前缀规范化（非严格缺陷，属增强）
- 涉及文件：src/framework/common/src/config/env_config.cc, src/platform/common/externalinput.cc
- 缺陷描述：约65处环境变量解析相关日志缺少统一前缀标签。无法通过grep快速过滤所有环境变量相关日志，影响问题定位效率。
- 修复模式：批量在HCCL_RUN_INFO/HCCL_RUN_WARNING宏调用的日志消息前添加"[HCCL_ENV]"前缀。
- 可审查性：高
- 审查规则建议：日志应遵循统一的前缀/标签规范；commit message中[Fix]标签名不实——这是enhancement。

### 5c72f6aaf3c3ecdab315927c9c784f84688d9db4 fix local windows addr in attn ffn scene
- 根因类别：map key与value语义混淆导致地址获取错误
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：attn/ffn分离场景下使用用户注册内存时，localWindowsIn/Out的赋值使用了`userMemMap_.begin()->first`（map的key=handle指针），应使用`userMemMap_.begin()->second->ptr()`（value指向的DeviceMem实际设备内存地址）。将handle指针值当作设备内存地址使用，导致通信操作读写错误内存地址。
- 修复模式：将.first(key/handle)改为.second->ptr()(value指向的实际内存地址)；增加isUseUserMem/winSize/localWindows等关键变量的日志输出。
- 可审查性：中
- 审查规则建议：使用std::map时明确区分.first(key)和.second(value)语义，特别是key和value都是指针类型时极易混淆；reinterpret_cast<u64>将指针转地址值时必须确认指向的是实际硬件内存而非管理对象handle。

### a014e91918eeecf35d8de4622d459d71e2b21445 fix NsRecovery
- 根因类别：并发安全 / 内存屏障缺失（编译器/CPU指令重排序）
- 涉及文件：src/platform/resource/mem/hdc.cc
- 缺陷描述：HDCommunicate::Write()中先memcpy_s将数据写入共享内存，然后更新尾计数器(tailCntAddr_)通知读取端有新数据。编译器/CPU可能将尾计数器更新重排序到数据写入之前，读取端看到尾计数器已更新但数据尚未写入，读到脏数据。经典的store-store重排序问题。
- 修复模式：在数据写入和尾计数器更新之间插入`std::atomic_thread_fence(std::memory_order_seq_cst)`全序内存屏障。
- 可审查性：低
- 审查规则建议："先写数据、再写标志位/计数器"的生产者-消费者模式，必须在两步之间插入内存屏障（至少release语义）；建议将tailCntAddr_改为std::atomic<u32>类型使用store(tail, memory_order_release)更安全。

---

## 第4轮 (commits 61-80)

### f45581ee0fb12f4c46f1e28f055cdbae1e0d19af 修复跨超transport销毁报错
- 根因类别：资源释放顺序错误
- 涉及文件：src/framework/communicator/impl/independent_op/channel/channel_manager.cc, src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/communicator/hccl_comm_host.cc
- 缺陷描述：跨超节点场景下，HcclCommunicator析构时销毁算法资源(DestroyAlgResource)，但ChannelManager中持有的transport link(channelLinks_)尚未释放。这些transport依赖的底层资源可能已被部分清理，导致channelLinks析构时访问已释放资源而报错。channel层的transport生命周期未与communicator的销毁流程正确协调。
- 修复模式：在ChannelManager中新增ReleaseChannel()方法，通过回调机制(std::function)注入到HcclCommunicator的析构流程中，确保在DestroyAlgResource之后显式释放所有channel transport。
- 可审查性：中
- 审查规则建议：当组件A持有组件B创建的资源引用时，审查析构/销毁流程是否保证资源释放顺序正确；跨层回调注册处应检查是否存在对称的注销逻辑。

### 32ff3df859db49f67ea90d2b40243b9c4300e733 fix reducescatter small count deter ffts graph bug
- 根因类别：多态覆写缺失 / 条件逻辑不一致
- 涉及文件：src/algorithm/impl/coll_executor/coll_reduce_scatter/coll_reduce_scatter_executor.cc, coll_reduce_scatter_mesh_opbase_small_count_deterministic_executor.cc/.h
- 缺陷描述：基类CollReduceScatterExecutor中preloadCopyOpt的判断条件是`(!DMAReduceFlag_) && (count == execMem.count)`，子类SmallCountDeterministicExecutor直接硬编码为`(count == execMem.count)`缺少DMAReduceFlag_检查。deterministic FFTS图模式下条件不匹配导致图构建错误。
- 修复模式：将条件判断提取为virtual方法IsPreloadCopyOptimizeCondition()，基类和子类各自override实现正确语义。
- 可审查性：高
- 审查规则建议：基类和子类存在相同语义的条件判断时，应通过virtual方法统一，避免内联条件在不同位置各自演化导致不一致。

### 89fd99e02e07d6d50c06c150dea9a1bffe576581 fix ccl_kernel hccd not found acl.h API
- 根因类别：编译条件缺失 / 平台兼容性
- 涉及文件：src/platform/common/externalinput.cc
- 缺陷描述：ParseCannVersion()调用ACL头文件(acl.h)中的API获取版本，但CCL_KERNEL_AICPU和HCCD编译目标下ACL不可用，直接编译导致找不到函数符号、编译失败。
- 修复模式：用#if !defined(CCL_KERNEL_AICPU) && !defined(HCCD)条件编译包裹，不支持的目标返回HCCL_E_NOT_SUPPORT。
- 可审查性：高
- 审查规则建议：引入新的外部API依赖时，必须检查所有编译目标(host/device/kernel/daemon)是否都能正确链接；CI应覆盖所有编译目标的构建验证。

### 8c424d41f66418cb675de678c75d4b7a464a3987 [DEBUG] fix continuous pipeline missing in cmakelist
- 根因类别：构建配置遗漏
- 涉及文件：src/framework/CMakeLists.txt
- 缺陷描述：新增的alltoallv continuous pipeline功能对应的两个.cc源文件在CMakeLists.txt中遗漏，导致不被编译链接，功能运行时缺失。
- 修复模式：在CMakeLists.txt的源文件列表中补充遗漏的alltoallv_continuous_pipeline.cc和coll_all_to_all_v_continuous_pipeline_executor.cc。
- 可审查性：高
- 审查规则建议：新增源文件的PR必须同步检查CMakeLists.txt是否已更新；可在CI中加入脚本扫描.cc文件是否都被构建系统引用。

### 5e446705486ff9b2c4a4db9b5251c51a2c4ce3cf fix log issue
- 根因类别：日志格式串参数错误 + 日志信息不足
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc, src/framework/device/debug/dfx/profiling/profiling_manager_device.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：三处问题。(1) profiling_manager_device.cc: memcpy失败日志缺少目标buffer大小，无法判断溢出原因。(2) aicpu_communicator.cc: HCCL_ERROR格式串%s期望字符串但传入int32_t类型的ret，%s将ret当指针解引用读取字符串，导致段错误或垃圾输出——未定义行为。(3) hccl_communicator_host.cc: 纯代码风格调整。
- 修复模式：补充sizeof_data日志字段；将%s对应参数改为__func__，ret作为%d的参数。
- 可审查性：高
- 审查规则建议：所有printf风格日志应启用-Wformat编译警告捕获格式串参数不匹配；建议对自定义日志宏添加__attribute__((format(printf,...)))标注。

### 47020bd546892ee89c86ebc27b9b415158576cc5 HcomGetandClearOverFlowTasks bugfix
- 根因类别：接口设计错误 / 输出参数语义错误
- 涉及文件：pkg_inc/hccl/hcom.h, src/framework/hcom/hcom.cc
- 缺陷描述：HcomGetandClearOverFlowTasks函数语义是"获取并清除溢出任务"，但原签名用值传递(HcclDumpInfo*, s32 len)，函数内部构造局部vector填充结果后无法回传给调用者——调用者永远拿不到溢出任务信息。
- 修复模式：改为二级指针和指针参数(HcclDumpInfo**, s32*)作为输出。注意修复后仍存在潜在悬垂指针风险：局部vector的.data()在函数返回后被释放。
- 可审查性：高
- 审查规则建议：当函数语义是"获取/查询"时，检查输出参数是否通过指针/引用回传结果而非仅作输入；对返回指向局部容器.data()的指针，检查生命周期安全性。

### 41de1fe5cd8f54b7e4c14dc537bdce3ad81a8deb Bug Fix
- 根因类别：初始化逻辑错误 / 默认值覆盖外部配置
- 涉及文件：src/algorithm/impl/topo_matcher.h
- 缺陷描述：HcclExternalEnableDef::SetDefaultAlgo()将所有算法配置硬编码为HCCL_ALGO_TYPE_DEFAULT，忽略了通过环境变量配置的算法选择策略。调用SetDefaultAlgo后用户的算法配置被丢弃。
- 修复模式：替换为调用GetExternalInputHcclAlgoConfig()从外部配置加载算法选择，而非硬编码默认值。
- 可审查性：中
- 审查规则建议："设置默认值"逻辑需检查是否存在应优先的外部配置源(环境变量/配置文件)；"Default"语义需明确——是系统内置默认还是用户未配置时的回退。

### 20125a56ba772d2798601bc52bad0fb4051a1397 fix bug
- 根因类别：构建配置错误 / 重复编译单元
- 涉及文件：src/CMakeLists.txt, src/framework/CMakeLists.txt
- 缺陷描述：error_code.cc同时被添加到两个CMakeLists.txt的源文件列表中，链接到同一二进制时导致符号重复定义(multiple definition)链接错误。
- 修复模式：从两处CMakeLists.txt中移除重复的error_code.cc引用。
- 可审查性：中
- 审查规则建议：CMake中添加新源文件时检查是否已在其他target中被引用，避免同一编译单元被多个target包含。

### 293b3f921fd57e0238fa30a8a47c9976d9d95881 hcom log fix
- 根因类别：字符串字面量拼接错误
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：HCCL_ERROR日志使用反斜杠(\)续行，下一行开头的缩进空格被原样包含在字符串中，导致日志输出中出现约8个多余空格。
- 修复模式：将反斜杠续行改为C/C++字符串字面量自动拼接("str1" "str2")。
- 可审查性：高
- 审查规则建议：禁止在字符串字面量中使用\续行，统一使用相邻字符串字面量自动拼接方式。

### 1d8e2c143905a5f2f035df01d7e0b12304e3fc13 [Master] Revert rts interfaces
- 根因类别：接口迁移回退 / API封装层能力不足（非典型缺陷，属接口重构）
- 涉及文件：src/platform/common/adapter/adapter_rts.cc, src/platform/common/p2p_mgmt/p2p_mgmt.cc, adapter_rts.h, 多个UT stub文件
- 缺陷描述：之前某次提交将底层RTS调用替换为ACL封装调用，但ACL层抽象引入功能限制(hrtGetPhyDeviceInfo只支持查询masterID)和不必要的ID转换。本次"回退"恢复对RTS原生接口(rtGetPhyDeviceInfo/rtEnableP2P等)的直接调用，并新增dlsym动态加载机制实现向后兼容。
- 修复模式：ACL封装调用替换回RTS原生接口，调整函数签名匹配RTS语义，新增动态加载回退路径。
- 可审查性：低
- 审查规则建议：封装层变更需检查是否覆盖被封装接口的完整能力；当封装层与底层参数语义不对齐时，不应强制使用封装层。

### 2bad014b2c249cb31c91d5bc83f95065837930fa try to fix core problem
- 根因类别：成员变量未初始化
- 涉及文件：src/framework/communicator/impl/hccl_communicator.h, src/framework/inc/hccl_comm_pub.h
- 缺陷描述：isGroupMode_、iSend、iRecv、nSend、nRecv、bufferSliceNum等10个内置类型成员变量缺少类内初始化器。C++中未初始化的bool/u32包含垃圾值，isGroupMode_的随机值导致group模式判断异常，计数变量的随机值可能引发coredump。
- 修复模式：为所有未初始化成员添加C++11类内默认初始化器(bool=false, u32=0, shared_ptr=nullptr, vector={})。
- 可审查性：高
- 审查规则建议：所有C++类的内置类型成员变量(bool/int/unsigned/指针)必须有类内默认初始化器或在构造函数初始化列表中显式初始化。

### f73986b9b83667d1da57916c6a25f9fed544efdd fix a+x netlayer
- 根因类别：逻辑错误 / 恒真条件(自身==自身) + 过度抽象
- 涉及文件：src/framework/communicator/impl/independent_op/rank_graph/rank_graph.cc, rank_graph.h
- 缺陷描述：InitServerRankInfo()中存在恒真条件bug：`rankInfoList[index].superPodId == rankInfoList[index].superPodId`自身与自身比较永远为true，superPod过滤逻辑完全失效——本应只记录本superPod的server到rank映射，实际所有rank被无条件加入。同时GetModuleIdx()间接层引入不必要复杂性，直接用serverIdx才正确。
- 修复模式：删除GetModuleIdx()及恒真条件分支，serverToRank_的key统一改为serverIdx。
- 可审查性：高
- 审查规则建议：静态分析应标记所有自身与自身比较的条件(a==a模式)，几乎总是bug。引入间接层时审查每个使用点是否真需要该间接。

### f597dd475132e5b269110e2434276273b2accd98 fix get retry server
- 根因类别：参数传递错误 / map查找key不匹配
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：HcclGetCommConnections()用identifier去hcclCommTopoInfoDetectServer map中查找，但map插入时使用的key是rootHandle.identifier，两者不同。查找永远匹配不到，isRoot永远false，retry时无法识别root节点。
- 修复模式：新增rootHandle参数，将map查找key从identifier改为rootHandle.identifier。
- 可审查性：高
- 审查规则建议：map的insert和find操作应确认使用相同语义的key；当函数参数名与map key语义不同时重点检查混淆。

### 26f5813f32cbe4c3872b14c223ceb4773221aec6 fix opretry
- 根因类别：API语义错误 / 上层API无法区分子操作
- 涉及文件：src/framework/cluster_maintenance/recovery/operator_retry/opretry_base.cc, 多个test stub文件
- 缺陷描述：StreamClear()函数两个分支(STREAM_STOP和STREAM_CLEAR)都调用aclrtStreamStop()——step参数形同虚设。aclrtStreamStop只能执行stop语义，无法执行clear语义。算子重试时stream没被真正清理，残留状态可能导致重试失败。
- 修复模式：替换为底层rtStreamClear()调用，根据step传入RT_STREAM_STOP或RT_STREAM_CLEAR枚举值。通过extern "C"前向声明引入。
- 可审查性：中
- 审查规则建议：函数有分支逻辑但所有分支执行相同操作时应标记可疑——通常意味着分支语义未被正确实现；使用上层封装API需确认其是否支持所需的操作区分。

### fa053984be459b3a9fa4c694e96f8fef2dc55d8b 修复N秒快恢问题
- 根因类别：状态重置时机错误
- 涉及文件：src/framework/device/aicpu_kfc/framework/aicpu_kfc_process.cc
- 缺陷描述：SetExpectPrepareId(0,0)重置放在KfcClearMsgArea()中——每次清理消息区域都会重置prepareId，但真正需要重置的时机是检测到suspending状态进入N秒快恢流程时。原位置触发频率过高且在真正需要的路径反而缺失。
- 修复模式：从KfcClearMsgArea()中删除重置调用，移到AicpuRunRpcServerForMC2V2()和AicpuRunRpcServerForMC2()的suspend分支中。
- 可审查性：中
- 审查规则建议：状态重置操作应审查执行时机是否与业务语义一致；放在通用清理函数中vs特定业务流程中，触发频率和条件可能完全不同。

### 7b12579d092891e82c52bcc50bae2db1e1d154dd 修复zerocopy锁问题
- 根因类别：并发安全 / 锁作用域错误
- 涉及文件：src/framework/communicator/impl/zero_copy/zero_copy_address_mgr_host.cc
- 缺陷描述：processRingBufferLock_互斥锁放在InitRingBuffer()中(只初始化一次的函数)，真正的并发写入点PushOne()缺少锁保护。多线程场景下PushOne()的数据竞争可能破坏ring buffer一致性或引发崩溃。
- 修复模式：将lock_guard从InitRingBuffer()移到PushOne()函数开头。
- 可审查性：高
- 审查规则建议：审查互斥锁时验证锁保护的临界区是否覆盖所有实际并发读写路径。关注"只初始化一次"的函数上锁但"频繁调用"的写入函数未上锁的模式。

### 08b2658a6804140724e16ca5b18210725894d9be [BUGFIX]fix repeat launch
- 根因类别：重复执行 / 流程控制错误
- 涉及文件：coll_all_gather_executor.cc, coll_all_reduce_executor.cc, coll_all_reduce_mesh_small_count_executor.cc, coll_all_reduce_smallcount_for_910_executor.cc, coll_all_to_all_executor.cc, coll_broadcast_executor.cc, coll_native_executor_base.cc, coll_reduce_executor.cc, coll_reduce_scatter_executor.cc, coll_scatter_executor.cc (10个文件)
- 缺陷描述：各集合通信executor的Orchestrate()末尾无条件调用LaunchTaskExtend()，但某些执行路径(KernelRun()/RunLoop())内部已调用过LaunchTask()，导致任务被重复launch，引起硬件重复执行或stream状态异常。
- 修复模式：引入needLaunchAtTheEnd标志变量，已内部launch的分支将其置false，末尾改为条件调用。统一launch逻辑下沉到基类PostSyncWithoutSubstream()。
- 可审查性：中
- 审查规则建议："收尾操作"既可能在子流程内完成又在外层统一调用时，必须有标志位防止重复执行。更优模式是收敛到单一出口。

### af89e3a8635a4ecbc4ec5421208490770fd2ded9 fix A5 adaptor
- 根因类别：复制粘贴错误 / 参数传递错误
- 涉及文件：src/framework/communicator/impl/independent_op/hccl_independent_rank_graph.cc, src/framework/hcom/hcom.cc
- 缺陷描述：3处V2适配调用存在copy-paste后未修改的错误：HcclGetInstSizeByNetLayer()内调错函数名为HcclGetNetLayersV2且参数全错，CommGetInstTopoTypeByNetLayer()和CommGetInstSizeByNetLayer()参数同样对不上。因在条件编译块内(OPEN_BUILD_PROJECT+ORION_MODE)，常规构建不触发。另外hcom.cc中头文件名hcomm_private_v2.h多了一个m。
- 修复模式：纠正3处V2调用的函数名和参数；修正头文件名拼写。
- 可审查性：高
- 审查规则建议：条件编译块内的代码须与主路径同等严格审查；结构相似的函数中V2/适配层调用是copy-paste bug高发区，必须逐一验证函数名和参数对应关系。

### 5d1dc13b8e76cbc8fdb63c50353a2f59a338ebe5 Rsv log fix
- 根因类别：参数传递错误 / input/output内存混淆
- 涉及文件：src/algorithm/impl/coll_executor/coll_reduce_scatter_v/coll_reduce_scatter_v_deter_executor.cc
- 缺陷描述：ReduceScatterV的Level1阶段Prepare()第三个参数(output memory)错误传入execMem.outputMem，但Level1是在input buffer上做原地reduce，应传execMem.inputMem。中间结果写入错误内存区域，后续Level0读取得到错误数据。注意commit message称"log fix"实际是计算正确性bug。
- 修复模式：将Prepare()第三个参数从execMem.outputMem改为execMem.inputMem。
- 可审查性：高
- 审查规则建议：集合通信算法中Prepare()调用须验证input/output/scratch memory参数与当前算法阶段的数据流方向一致；多级流水线中间阶段的output不一定是最终output buffer。

### 4515f86f5682e29e0dad8d66d00fc86a4469c028 fix memory alloc for TbeCrackCleard
- 根因类别：内存分配不足 / 缓冲区复用缺陷
- 涉及文件：src/platform/legacy/hccl_tbe_task/tbe_crack_cleared.cc, tbe_vector_reduce.h
- 缺陷描述：tilingDataHostPtr_是成员级缓冲区，原分配逻辑为ptr==nullptr时才分配(仅首次)。但每次Run()的crackAddr.size()可变，后续调用若元素增多则所需buffer超过首次分配大小，导致堆缓冲区溢出。
- 修复模式：新增tilingDataHostSize_记录已分配大小，分配条件改为size<required时delete[]旧缓冲区并重新分配。
- 可审查性：高
- 审查规则建议：成员缓冲区被复用时，仅用ptr==nullptr判断是否分配是危险模式——应同时记录已分配大小并与实际需求比较。

---

## 第5轮 (commits 81-100)

### 2661d00faae29f757f858fa33f95d318fcf1bf09 fix
- 根因类别：架构设计缺陷——全局单例耦合导致配置传递路径不正确
- 涉及文件：57个文件，核心包括 src/algorithm/impl/hccl_aiv.cc, src/algorithm/impl/topo_matcher.h/cc, src/algorithm/pub_inc/hccl_aiv.h, src/algorithm/impl/alg_configurator.cc, src/algorithm/impl/coll_alg_utils.cc/h, src/algorithm/base/communicator/topo_info_extractor.cc/h, src/framework/communicator/impl/hccl_communicator_*.cc, 26个coll_executor文件, src/framework/op_base/src/op_base.cc
- 缺陷描述：多处通过`CommConfiger::GetInstance()`全局单例获取算法配置(algoConfig)和超时配置(execTimeOut)。全局单例配置按identifier查询，但不同通信域实例应各自持有独立配置。AIV kernel launch时通过CommConfiger查询超时配置而非从调用链传递，导致配置可能不一致。Barrier和ClearAivSyncBuf函数内部自行查询CommConfiger获取超时参数，而非从调用方传入。
- 修复模式：依赖注入替换全局单例——将隐式全局状态查询改为显式参数传递。AivAlgArgs结构体新增execTimeOut/execTimeOutSet字段，26个AIV executor显式设置；Barrier/ClearAivSyncBuf函数签名增加参数；TopoInfoExtractor::Init()增加algoConfig参数；TopoMatcher新增algoConfig存储。
- 可审查性：低
- 审查规则建议：禁止在算法执行层(executor)直接调用`CommConfiger::GetInstance()`，配置应通过参数链传入；对`GetInstance()`模式的调用进行审查，评估是否应改为依赖注入。

### d3e2df653448cb509ebc3a77937e9474a8997c1f fix ffts perf
- 根因类别：多因素——FFTS图模式下notify地址重复查询 + alltoallv算法冗余计算/日志
- 涉及文件：src/platform/legacy/graph_ctx_mgr/graph_ctx_mgr.cc/h, graph_ctx_mgr_common.cc, src/platform/task/dispatcher_graph.cc, src/algorithm/base/alg_template/temp_alltoallv/alltoallv_direct_fullmesh.cc/pub.h, src/algorithm/impl/operator/alltoall_operator.cc
- 缺陷描述：(a) `ConstructFftsNotifyRecordRemoteCtx`每次调用`NotifyGetAddr(signal, &notifyAddr)`获取notify地址，在图模式下是昂贵的runtime调用。(b) alltoallv direct fullmesh算法中`CheckIsHaveZeroLength()`、`CalMaxRecvLen()`已不再需要，高频路径上的HCCL_DEBUG日志有性能开销。RunSDMA的opMeta参数内部重复构造。
- 修复模式：(a) 新增signalAddr参数允许调用方传入已知地址，避免重复查询。(b) 删除冗余函数和日志，RunSDMA签名改为外部传入opMeta。
- 可审查性：中
- 审查规则建议：FFTS图模式热路径中的runtime查询调用(如NotifyGetAddr)应审查是否可缓存；高频循环路径中的HCCL_DEBUG日志需评估性能影响。

### 341e789326ed1570029436c30625b3b90fe19aee MC2多流场景问题修复
- 根因类别：并发/线程安全——全局变量在多线程场景下的数据竞争
- 涉及文件：src/framework/device/aicpu_kfc/framework/aicpu_kfc_process.cc
- 缺陷描述：`g_expectPrepareId`是静态全局数组(`static uint8_t g_expectPrepareId[MAX_QUE_NUM]`)，用于跟踪每个队列期望的prepare消息ID。MC2多流场景下多个线程并发写同一位置，一个线程写入的值被另一个线程覆盖，造成消息ID不匹配，引发通信hang或数据错误。prepare ID的语义本身就是per-thread的。
- 修复模式：`static` -> `static thread_local`，使每个线程拥有独立副本。
- 可审查性：高
- 审查规则建议：AICPU KFC框架中的static全局变量需审查是否在多线程/多流场景下被并发访问；涉及per-stream/per-queue状态的变量必须使用thread_local或显式同步。

### f058f2d9fee1c10f8129c51346a26776c68e9c96 fix aiv timeout err
- 根因类别：边界条件处理缺失——超时值为0时的特殊语义未处理
- 涉及文件：src/algorithm/impl/hccl_aiv.cc
- 缺陷描述：`GetAivTimeout`函数将用户配置的超时时间(秒)转换为AIV kernel超时(微秒)。`execTimeOut = 0`的语义应是"使用最大超时时间"(不超时)，但代码按字面值0处理，计算`0 * TIME_S_TO_US = 0`微秒，导致AIV kernel启动后立即报超时错误。
- 修复模式：函数入口增加`execTimeOut == 0`特判，直接返回`AIV_TIMEOUT_MAX_US`。
- 可审查性：高
- 审查规则建议：所有接受超时参数的函数须审查零值和负值处理逻辑；当配置项存在"0 = 不限制"的语义约定时，必须在转换函数中显式处理。

### c6f8cc36589a785f915bb48eff456bb9c7c434c1 fix aiv alltoall selectalg
- 根因类别：条件判断遗漏——AIV算法选择未排除不支持的硬件拓扑
- 涉及文件：src/algorithm/impl/operator/alltoall_operator.cc
- 缺陷描述：`IsSatisfyAlltoAllAivCondition`缺少`!multiModuleDiffDeviceNumMode_`检查。该标志为true表示多模组场景下各模组device数量不一致(非对称拓扑)，AIV alltoall算法依赖对称device数量假设，强行走AIV路径导致计算错误或hang。注意：次日commit d3e2df6又删除了此条件，两者存在矛盾。
- 修复模式：在isSupportAiv条件链中添加`&& !multiModuleDiffDeviceNumMode_`。
- 可审查性：高
- 审查规则建议：AIV算法的IsSatisfy*Condition函数需建立checklist覆盖所有拓扑约束；新增拓扑模式时须联动审查所有算法选择函数。

### 5bfaa1e5b3547eb653731223a3daa20fe6ec3162 6link reducescatterVfailed bugfix
- 根因类别：算法名称拼写错误(Copy-Paste)
- 涉及文件：src/algorithm/impl/operator/reduce_scatter_v_operator.cc
- 缺陷描述：`SelectAlgfor91093`中选择的算法名称写成`AlignedReduceScatterDoubleRingFor91093Executor`，缺少"V"后缀。当前操作是ReduceScatterV，应选择`AlignedReduceScatterVDoubleRingFor91093Executor`。从ReduceScatter代码复制过来后忘记加"V"，导致6-link拓扑下选错executor。
- 修复模式：单字符修复，算法名称中插入"V"。
- 可审查性：中
- 审查规则建议：ReduceScatterV相关operator文件中的所有算法名称字符串须检查是否包含"V"后缀；可建立算法名称到operator类型的映射校验规则。

### cd056a5eb154dfee01ab006f98b246ffd2f81d86 Fix reqHandle UAF issue
- 根因类别：Use-After-Free(UAF) / 并发资源管理缺陷
- 涉及文件：src/platform/hccp/rdma_agent/hdc/async/ra_hdc_async.c
- 缺陷描述：`RaHdcAsyncSessionClose`中session close超时等待结束后，直接`free(reqHandle)`释放请求句柄。reqHandle是异步请求句柄，可能正被其他线程(异步接收线程、工作线程池)引用，直接free导致UAF。
- 修复模式：将`free(reqHandle); reqHandle = NULL;`替换为在reqMutex锁保护下调用`HdcAsyncDelResponse(reqHandle)`，确保删除操作与其他访问互斥，通过统一的删除函数处理关联资源清理。
- 可审查性：中
- 审查规则建议：有配套mutex的数据结构，其生命周期管理(free/delete)须在锁保护下进行；如果项目中有配套的释放函数(如XxxDelResponse)，不应出现对同类资源的裸free()调用。

### 6972024c8243a63c2eeb911d3ceb65c264fae833 fix aiv log
- 根因类别：控制流/宏语义问题——CHK_RET宏的隐式ERROR日志不符合场景需求
- 涉及文件：src/algorithm/impl/operator/coll_alg_operator.cc
- 缺陷描述：`CalBlockDim`函数中使用`CHK_RET(executor_->CalBlockDim(...))`，CHK_RET宏在失败时隐式打印ERROR级别日志并return。但AIV core不足是可恢复场景(系统回退到非AIV路径)，不应打ERROR。修复将`CHK_RET(...)`改为`return executor_->CalBlockDim(...)`，让调用方控制日志级别。
- 修复模式：`CHK_RET(expr)` -> `return expr`，消除宏的隐式日志打印行为。
- 可审查性：低
- 审查规则建议：当CHK_RET的调用结果之后紧跟return HCCL_SUCCESS且两者间无其他逻辑时，建议直接使用return；caller有fallback逻辑时，被调函数不应被CHK_RET包裹。

### 2adb85d7d36f29005e1eaa2f92f36f518fee3a3c fix aiv error log
- 根因类别：日志级别不当——可恢复场景使用ERROR级别
- 涉及文件：27个文件，涵盖AllGather/AllReduce/AllToAll/ReduceScatter/Broadcast等AIV executor实现和hccl_communicator_host.cc
- 缺陷描述：各executor的CalBlockDim函数中aivCore数量不足时用HCCL_ERROR打印，但这是可恢复场景(回退到非AIV路径)，不应用ERROR。各KernelRun中用CHK_RET宏调用CalBlockDim也会产生隐式ERROR日志。
- 修复模式：(1) HCCL_ERROR -> HCCL_WARNING批量修改所有executor的CalBlockDim；(2) CHK_RET -> CHK_PRT_RET显式控制错误日志内容和返回码。
- 可审查性：中
- 审查规则建议：可恢复错误场景的日志不应用ERROR应用WARNING；CHK_RET宏因隐式打印ERROR，在需精细控制日志级别的场景下应避免使用。

### 947460b264e6c3a8da37ea2482d68acd6ec238fa fix gcov print
- 根因类别：构建配置错误——interface library传播了不当的编译选项
- 涉及文件：CMakeLists.txt, src/CMakeLists.txt, src/algorithm/CMakeLists.txt, src/framework/CMakeLists.txt, src/platform/CMakeLists.txt, src/platform/legacy/CMakeLists.txt
- 缺陷描述：`intf_pub`是interface library，通过CMake传播机制将GCC native/gcov编译选项传递到所有链接它的target。在BUILD_OPEN_PROJECT构建模式下这些选项不应启用，导致异常的gcov输出。
- 修复模式：移除`include(cmake/intf_pub_gccnative.cmake)`及所有target的`$<BUILD_INTERFACE:intf_pub>`链接依赖。
- 可审查性：低
- 审查规则建议：CMake中interface library的编译选项传播应有文档说明；多构建模式项目中须检查interface library在所有模式下是否合理。

### cabdb5db01b4938705faa8d3edc52362f23d6dc5 fix A5 identifier param with foo string.
- 根因类别：条件编译分支漏改——接口签名变更后调用点未同步更新
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：`HcclCommInitRootInfoV2`调用时传入`identifier`参数，但在当前条件编译分支(`#if !defined(OPEN_BUILD_PROJECT)`)中该变量已不存在(上游commit bdbc655移除了identifier参数)。修复创建空`std::string fooidentifier`局部变量占位。
- 修复模式：创建空占位变量替代不存在的变量引用。
- 可审查性：高
- 审查规则建议：函数签名变更时应自动检查所有调用点包括条件编译分支内的调用；变量名含"foo"/"tmp"/"hack"的应重点关注。

### bdbc65541499e5006d20936228604e9fba5cf892 fix topo detect server and agent map key inconsistency
- 根因类别：map查找key不一致——server端和agent端使用不同key索引同一逻辑实体
- 涉及文件：src/framework/op_base/src/op_base.cc
- 缺陷描述：拓扑检测中server端map以`rootHandle.identifier`为key插入，但通信标识符经协商后变为`commIdentifier`。当`rootHandle.identifier != commIdentifier`时，agent端insert和后续查找(`HcclGetCommConnections`)使用的key不一致，导致找不到。
- 修复模式：(1) HcclGetCommConnections参数从HcclRootHandle改为std::string identifier解耦；(2) agent端用commIdentifier作key；(3) InitCommRootInfo结束前若key不一致则替换server端map的key；(4) 移除无用的全局thread_local map。
- 可审查性：中
- 审查规则建议：同一逻辑概念使用多个map存储时，审查所有map的key是否始终一致；同一函数内对多个map的insert/find操作key来源应相同。

### 30e25e509b127c077358877259046992f7111d34 [Master] Revert Runtime interfaces
- 根因类别：接口回退——ACL封装层能力不足，回退到RT底层接口
- 涉及文件：include/hccl/hcomm_primitives.h, src/common/stream/stream_utils.cc, src/framework/op_base/src/op_base_host.cc, src/platform/common/adapter/adapter_rts.cc, src/platform/inc/adapter/adapter_rts.h, src/pub_inc/adapter_rts_common.h, 及UT相关文件
- 缺陷描述：之前将RT接口替换为ACL封装引入问题：(1) GetModelId被改成`reinterpret_cast<uint64_t>(rtModel)`(把指针地址当ID)，无法获取真实model ID；(2) 任务失败回调改用ACL的aclrtSetExceptionInfoCallback后丢失按模块名注册能力("HCCL"模块标识丢失)。
- 修复模式：全面回退ACL接口适配，恢复RT接口调用。
- 可审查性：中
- 审查规则建议：替换底层接口为高层封装时须验证高层接口是否提供底层的全部能力；`reinterpret_cast`将指针转为整数ID的模式应设置告警。

### bd7fffbc24e98fe64170c96a221104bd5dfd187e fix code review
- 根因类别：混合缺陷——printf格式说明符与变量类型不匹配 + 函数返回值未检查
- 涉及文件：src/algorithm/base/alg_aiv_template/aiv_crossnode_91093_base.h, 多个coll_executor文件, src/algorithm/impl/hccl_aiv.cc
- 缺陷描述：(1) count/blockOffset是uint64_t却用%u打印；minNpuSchedTimeout/maxNpuSchedTimeout是u64用%u打印；userRank/localRank是u32用%d打印。(2) CalReduceScatterVSliceData()和RunReduceScattervLevel1()返回HcclResult但调用处未用CHK_RET检查。
- 修复模式：格式说明符修正(%u->%llu, %d->%u)；裸调用改为CHK_RET()包裹。
- 可审查性：高
- 审查规则建议：开启-Wformat编译警告可自动检测格式符不匹配；返回HcclResult的函数调用须用CHK_RET或显式检查返回值。

### 98b099e463df7b172066fa6037e3e83c479e32af fix 910C batchSendRecv bug
- 根因类别：缺少链路类型过滤条件——对不应建链的远端rank执行了建链操作
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：910C的BatchSendRecv中ReAllocTransportResource和IncreAllocTransportResource为远端rank创建通信链路，但缺少过滤：当远端rank使用DirectNpu链路时不应在这里CreateLink(DirectNpu有独立建链路径)，导致重复或错误建链引发功能异常。
- 修复模式：在CreateLink调用前增加判断：若当前操作是HCCL_CMD_BATCH_SEND_RECV且远端rank是DirectNpu类型则continue跳过。两个对称函数做了相同修复。
- 可审查性：低
- 审查规则建议：对称/镜像函数的diff应联合审查；新增芯片/链路类型时应有checklist确保所有资源分配路径考虑了新类型过滤。

### 61aa0aaab6b9d0ad375165fd37b303a0953efbc4 add asan gcov and fix upgrade
- 根因类别：构建配置缺陷——ASAN/GCOV编译选项判断条件不一致 + 安装脚本逻辑缺陷
- 涉及文件：CMakeLists.txt, build.sh, cmake/intf_pub_gccnative.cmake, src/CMakeLists.txt, src/algorithm/CMakeLists.txt, src/framework/CMakeLists.txt, src/platform/CMakeLists.txt, src/platform/legacy/CMakeLists.txt, test/ut/多个CMakeLists.txt, scripts/package/hcomm/scripts/install.sh
- 缺陷描述：ENABLE_ASAN的编译选项判断用`STREQUAL ON`而链接选项用`STREQUAL true`，build.sh传入的是`true`导致ASAN在非测试构建中不生效。升级脚本中对atc/fwkacllib目录的处理逻辑不适用于当前组件。
- 修复模式：统一使用`$<$<BOOL:${ENABLE_ASAN}>>`生成器表达式；升级脚本删除不适用分支。
- 可审查性：中
- 审查规则建议：CMake中ENABLE_*开关变量同一文件内判断方式须一致；build.sh传入CMake的布尔值应统一使用ON/OFF。

### 8fa9c33715fc63ed520f0857bdddb1ac50d91b07 aiv bugfix
- 根因类别：AIV核函数同步屏障缺失 + 使用过时API
- 涉及文件：src/algorithm/base/alg_aiv_template/aiv_all_gather_910b_bigdata.h
- 缺陷描述：910B大数据量AllGather的AIV kernel中，所有`pipe_barrier(PIPE_ALL)`被替换为`PipeBarrier<PIPE_ALL>()`(旧API->新API)。关键修复点在MemcpyWithFlagWrap：CpGM2GM(GM到GM拷贝)之后缺少PipeBarrier同步屏障，直接更新processedBatchCount，导致数据拷贝可能未完成就进入下一轮批次处理，产生数据竞争/不一致。
- 修复模式：pipe_barrier -> PipeBarrier API替换 + 补充缺失的GM2GM后同步屏障。
- 可审查性：低
- 审查规则建议：全局搜索禁止使用已废弃的pipe_barrier函数；GM2GM拷贝操作后必须有显式pipeline barrier。

### 20824546dfb45e2c1554cab6c4a261cf48bff152 [BUGFIX] fix wrong comparison of overflow mode
- 根因类别：枚举值比较错误(语义混淆)
- 涉及文件：src/platform/common/device_capacity.cc
- 缺陷描述：`IsOverFlowInfNanMode()`函数本意判断设备是否处于InfNan溢出模式，但实际比较的是`ACL_RT_OVERFLOW_MODE_UNDEF`(未定义模式)而非`ACL_RT_OVERFLOW_MODE_INFNAN`。导致UNDEF模式时错误返回true，真正的INFNAN模式下返回false——逻辑完全反了。影响通信算子的溢出处理路径选择。
- 修复模式：单行修复，将ACL_RT_OVERFLOW_MODE_UNDEF替换为ACL_RT_OVERFLOW_MODE_INFNAN。
- 可审查性：高
- 审查规则建议：函数名包含特定枚举语义时审查其内部是否确实比较了对应枚举值；可编写静态规则检测函数名与内部使用枚举值的匹配性。

### 107247d150ba91de9e25f9f4d707462de1a081cc Split header files to fix cleancode
- 根因类别：头文件组织不当(cleancode问题，非功能缺陷)
- 涉及文件：src/platform/hccp/rdma_service/rs_inner.h, rs_rdma.h, rs_rdma_inner.h(新建), rs_drv_rdma.c, rs_rdma.c
- 缺陷描述：rs_inner.h是通用内部头文件，但包含了仅RDMA子模块使用的宏定义(RS_WC_NUM, RS_QP_ATTR_*等)和数据结构(RsCmdOpcode, RsMrInfo)。cleancode工具报告头文件中定义了不属于其职责范围的符号。
- 修复模式：提取头文件——将RDMA相关宏和结构体从通用头文件移到专用rs_rdma_inner.h。
- 可审查性：高
- 审查规则建议：头文件中的宏和类型定义应与其职责范围匹配；仅被特定子模块使用的定义应放在该子模块专属头文件中。注意：此commit是纯重构，不修复运行时行为缺陷。

### 77633b7c6d5ded06a4dfd91b5187e48731592b4c ProcessTaskAbortHandleCallback bugfix
- 根因类别：任务中止恢复流程缺少关键步骤——StopExec未被调用
- 涉及文件：src/framework/communicator/impl/task_abort_handler.cc/h, src/framework/communicator/impl/hccl_communicator.cc
- 缺陷描述：`ProcessTaskAbortHandleCallback`的POST阶段处理中只调用Clean()清理通信器，但缺少先调用StopExec()停止执行。NsRecovery流程应为Suspend->StopExec->Clean三阶段，缺少StopExec导致执行未停止就被清理，造成资源泄漏或状态不一致。附带修复：printf格式符%u改%d匹配int32_t，%zu改%u；CHK_RET包裹简化为直接return。
- 修复模式：在Clean()之前为每个communicator添加StopExec()调用，有超时/无超时两个分支均做相同处理。
- 可审查性：中
- 审查规则建议：任务中止回调的POST阶段应包含完整恢复步骤序列(Suspend/StopExec/Clean)，审查时对照设计文档核对；printf格式符须与参数类型匹配。

## 第6轮：commits 101-120

### 4d1a3a60258cd18a25592d2d1ba4df98062210f7 InitAicpuIndOp fix backThread.251212
- 根因类别：竞态条件/初始化时序错误
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：`HcclCommAicpu::InitAicpuIndOp`函数中，原代码先调用`AicpuHcclProcess::CallMC2MaintenanceThread(ctx)`拉起背景线程，然后才将`indOpCommInitialized_`设为true。背景线程启动后可能立即检查该标志判断初始化是否完成，但此时标志还是false，导致背景线程误判初始化未完成而跳过处理或进入异常分支。
- 修复模式：调整语句执行顺序，先设置状态标志再启动依赖该标志的线程。
- 可审查性：中
- 审查规则建议：当标志位赋值与线程启动在同一函数中出现时，检查线程是否依赖该标志位，若是则标志位赋值必须先于线程启动（happens-before关系）。

### be994b3253d11ab532f20af61c7cc3a8c66d13c0 fix onesided bugs
- 根因类别：多类缺陷（场景遗漏/printf参数错误/数据结构操作缺失）
- 涉及文件：src/framework/communicator/impl/hccl_communicator.cc, src/platform/resource/mem/mem_mapping_manager.cc, src/platform/resource/transport/mem_name_repository.cc
- 缺陷描述：3个独立bug：(1) `InitMemoryManager()`缺少对onesided communicator的判断，onesided不需要初始化MemoryManager但原代码没有跳过，导致资源不匹配而失败。(2) `GetDevVA`日志`%p`格式传了`&devVA`（指针的地址）而非`devVA`本身，日志输出栈地址而非设备VA。(3) `SetIpcMem`构造`ipcMemInfo`后缺少插入`alignPtrMap_`的操作，后续`FindIpcMem`找不到导致IPC内存注册失败。
- 修复模式：添加场景分支跳过 / 修正printf参数 / 补充遗漏的map插入操作。
- 可审查性：中
- 审查规则建议：新增communicator类型时审查所有初始化流程是否需要特殊处理；`%p`参数应为指针值本身不应取地址；构造数据对象后检查是否遗漏了容器插入步骤。

### 67317132a4d55315e8fee6b069b475eb7601b46e fix rs bug
- 根因类别：多类缺陷（调试代码残留/运算符优先级歧义/状态泄漏）
- 涉及文件：src/algorithm/base/alg_aiv_template/aiv_all_gather_910b_smalldata.h, src/algorithm/impl/coll_executor/多个executor文件, src/framework/hcom/hcom.cc
- 缺陷描述：(1) `aiv_all_gather_910b_smalldata.h`残留调试断言`AIV_ERROR(false, "[ASSERT] tag is [%d]\n", tag)`，生产环境中触发错误输出。(2) 多个executor的日志"at lest"拼写错误。(3) `reduce_scatter_operator.cc`中`SelectAlgfor91093`条件`PIPELINE || DEFAULT && size >= MIN`，由于&&优先级高于||需要显式括号消除歧义。(4) `CallMsprofReportHostApi`结尾缺少`SetAivCoreLimit(0)`重置调用，上一次操作设置的AIV core limit泄漏到下一次操作。
- 修复模式：删除调试代码 / 修正拼写 / 添加括号 / 补充状态重置。
- 可审查性：中
- 审查规则建议：CI扫描`AIV_ERROR(false,`模式禁止合入；启用`-Wlogical-op-parentheses`警告；SetXxxLimit类调用检查是否有配对的reset。

### 2121cd32b29338d88260577b14c711a410038f9f fix buildcheck
- 根因类别：构建规范回退（与92fe99a6互为反转操作）
- 涉及文件：build.sh, build_third_party.sh, cmake/scripts/parser_ini.py, examples/build.sh, python/hccl/manage/api.py等16个文件
- 缺陷描述：完全回退commit 92fe99a6的改动（移除shebang行、编码声明、set-e、版权头），同时引入危险改动：`build.sh`中移除了`rm -rf`前的变量非空检查，当变量为空时可能误删文件。两个commit构成"引入-回退"对，说明buildcheck规则存在矛盾。
- 修复模式：批量文件头格式调整以通过构建检查工具。
- 可审查性：低
- 审查规则建议：`rm -rf`命令前的变量必须有非空检查；buildcheck工具的规则应明确文档化避免反复修改。

### 92fe99a626ea8fcae74b5de535235dc9528904c9 fix buildcheck
- 根因类别：构建规范修复（被2121cd32回退）
- 涉及文件：与2121cd32完全相同的16个文件
- 缺陷描述：正向的构建合规修复：为shell脚本添加shebang和set-e，为Python脚本添加编码声明，`$(cmd)`替代反引号，为`rm -rf`添加变量非空检查。但被后续commit完全回退，说明与某个buildcheck规则冲突。
- 修复模式：批量添加文件头声明、修正shell语法风格、增加安全检查。
- 可审查性：高
- 审查规则建议：shell脚本必须以`#!/bin/bash`开头；`rm -rf`接变量必须有非空判断；这对commit说明buildcheck流程存在矛盾规则需统一。

### 3ebc92916ca7d3be2a28d1d95b89f0a06e5acfbb [Master] Build: Fix makeself
- 根因类别：构建配置缺陷（路径硬编码+cmake API误用）
- 涉及文件：cmake/third_party/makeself-fetch.cmake
- 缺陷描述：三个问题：(1) 拷贝目标路径硬编码为`${CMAKE_BINARY_DIR}/makeself`而非使用已定义的`${MAKESELF_PATH}`变量，导致后续EXISTS判断永远找不到已拷贝文件，每次触发下载。(2) `execute_process(COMMAND cmake -E copy ...)`将多源文件作为同一COMMAND参数传递，cmake -E copy只接受单源文件。(3) 冗余双层if且外层内层路径不一致。修复统一使用`${MAKESELF_PATH}`，改用`file(COPY)`命令。
- 修复模式：路径统一 + API替换 + 死代码删除。
- 可审查性：高
- 审查规则建议：cmake中路径变量已定义时检查同作用域是否有硬编码路径；`cmake -E copy`参数个数检查。

### b3975b8596bcf207f12f954fc12acdc18be444f4 fix rs precision
- 根因类别：缓存键维度不完整导致跨模式缓存命中错误
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc, src/platform/common/unfold_cache/op_unfold_cache.cc, op_unfold_cache_entry.cc, op_unfold_key.cc, op_unfold_key.h, src/platform/task/dispatcher_aicpu.cc
- 缺陷描述：算子展开缓存OpUnfoldCache的缓存键`OpUnfoldKey`缺少`workflowMode`（图模式/单算子模式）维度。同一算子在图模式和单算子模式下走不同算法路径，但缓存键不区分导致跨模式错误复用，造成ReduceScatter精度错误。附带：缓存跳过条件缺少`HCCL_CMD_REDUCE_SCATTER`枚举值。
- 修复模式：缓存键扩展（增加区分维度）+ 遗漏枚举值补全。
- 可审查性：中
- 审查规则建议：缓存键结构体新增字段时检查hash/operator==/拷贝构造是否同步更新；枚举遍历/条件分支检查是否涵盖所有操作类型。

### de46e91aa4951638dc6e956efb33513df0828906 fixRepeatedAicpuComm251211
- 根因类别：缺少幂等性保护导致重复初始化失败
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc/h, aicpu_hccl_process.cc/h, 及st/ut测试文件
- 缺陷描述：`HcclCommAicpu`的`Init()`和`InitAicpuIndOp()`缺少幂等性保护。`AicpuCreateCommbyGroup()`在commMap中发现group已存在时返回`HCCL_E_INTERNAL`错误，导致正常的重复调用场景直接失败。修复新增`initialized_`/`indOpCommInitialized_`标志做init guard，将`AicpuCreateCommbyGroup()`重构为`AcquireAicpuComm()`实现"获取或创建"语义。同时将初始化完成标志赋值移到函数末尾确保所有步骤成功后才标记。
- 修复模式：幂等性保护（init guard）+ 语义重构（create -> acquire-or-create）。
- 可审查性：中
- 审查规则建议：Init/Setup类函数应包含幂等性检查；初始化完成标志应在所有步骤成功后设置。

### 48752a04f77290bf6154d3f8c961ea946bceca65 A3多通信域并发场景问题修复
- 根因类别：固定大小数组+全局/局部索引混淆+并发竞态（系统性设计缺陷）
- 涉及文件：src/framework/device/aicpu_kfc/aicpu_kfc_interface.cc, aicpu_kfc_process.cc/h, aicpu_kfc_rpc_serverv2.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：A3芯片多通信域并发场景下的系统性缺陷：(1) `g_commInst[MAX_COMM_CTX_NUM]`固定数组（MAX=3），通信域超过3个时数组越界，改为unordered_map。(2) `TimeOutCheckInfo`中`msgFlag`/`msgStartTime`等固定数组同样改为map。(3) 全局groupIdx与局部localGroupIdx混淆，`BarrierProcess()`等函数传入全局索引但BarrierInfo按局部索引组织，引入localGroupIdx区分。(4) `g_commIdMap`/`g_commTypeInfoMap`缺少读写锁保护，增加ReadWriteLock。(5) 多处初始化/检查时序重排。
- 修复模式：动态容器替换固定数组 + 索引体系重构 + 并发锁保护 + 时序重排。
- 可审查性：低
- 审查规则建议：全局固定大小数组存储可变数量实例时应告警；函数参数中多种语义索引需明确命名区分；共享可变全局状态必须有锁保护。

### 384d78c0fea07d2dc5f6a4503c69280c63e91495 fix dispatcher_ctx
- 根因类别：空指针解引用 + 资源获取失败未处理
- 涉及文件：src/platform/comm_primitive/hccl_dispatcher_ctx.cc, hccl_primitive_local.cc, src/pub_inc/new/hccl_dispatcher_ctx.h
- 缺陷描述：(1) `DestroyDispatcherCtx()`和`GetDispatcherCtx()`中，`commId`可能为nullptr但先用`%s`格式化打印再做空指针检查，nullptr传给`%s`是UB可能崩溃。(2) `GetPubDispatcher()`调用`GetDispatcherCtx()`返回nullptr时直接失败无创建逻辑。修复新增`AcquireDispatcherCtx()`实现获取或创建语义：先尝试获取，为空则创建新实例并初始化。(3) 重复绑定同一commId日志级别从ERROR降为WARNING。
- 修复模式：空指针检查前置 + 懒初始化（acquire-or-create）+ 日志级别修正。
- 可审查性：高
- 审查规则建议：日志`%s`打印指针参数前必须确保已通过空指针检查；获取资源返回nullptr时调用方应有创建或回退逻辑。

### 017142fbc6b6f749e62df4a0cb9f16da8cfc8f76 fix rping init failed issue
- 根因类别：初始化依赖链断裂（前置初始化遗漏）
- 涉及文件：src/platform/hccp/hccp_service/main.c
- 缺陷描述：`llt_main`函数在调用`HccpInit`之前遗漏了`RsApiInit()`调用。rping功能依赖Rs API层初始化，但原代码直接进入HccpInit，导致Rs API未就绪rping初始化失败。修复在HccpInit前插入RsApiInit()，失败时goto清理路径，并在hccp_init_fail标签处增加对应的RsApiDeinit()。
- 修复模式：补充缺失的前置初始化调用 + 对应的反初始化清理。
- 可审查性：中
- 审查规则建议：Init/Deinit配对函数检查所有调用路径上是否成对出现；模块初始化函数的前置依赖需文档化并在调用点验证。

### a6782e9c33626990e410a2c34bdc4971384f7f8d fix ag precision
- 根因类别：地址空间混淆（远端VA vs 本端映射VA）
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc, aicpu_zero_copy_exchanger.cc/h
- 缺陷描述：`PrepareRemoteMemRanges`函数构建remote rank的memory range时，直接使用`inAddrs_[remoteRank % deviceNumPerAggregation_]`作为baseAddr。该数组保存的是远端virtual address，但OpUnfoldCache需要本端映射后的VA（通过RDMA映射的地址）。修复改为通过`current_->links`中的`curLink->GetRemoteMem()`获取正确的本端映射地址。同时日志格式从`%u`修正为`0x%016llx`匹配64位地址。这是allgather精度错误的根因——zero copy场景下使用了错误地址导致数据读写位置错误。
- 修复模式：修正数据源——从错误的远端地址数组切换到通过transport link获取正确的本端映射地址。
- 可审查性：低
- 审查规则建议：RDMA/zero-copy场景中对remote memory地址需明确标注地址空间归属（本端VA vs 远端VA）；日志打印地址类型应为`%p`或`0x%016llx`而非`%u`。

### ae3c0ec71d1f6c75b074fa67db770711c0cc7473 fix invalid tls status
- 根因类别：非缺陷修复（日志增强+等价代码重构）
- 涉及文件：src/platform/common/adapter/adapter_hccp.cc, src/platform/resource/socket/hccl_network.cc
- 缺陷描述：实际逻辑变更为零。将三目运算符拆成if-else分支（语义等价），增加HCCL_INFO日志打印phyId和tlsEnable/tlsStatus值用于排查"invalid tls status"问题。commit message说的"fix"实际是增加可观测性协助定位，而非修复逻辑缺陷。
- 修复模式：增加诊断日志。
- 可审查性：高
- 审查规则建议：此commit本身无缺陷修复，但暴露一个潜在问题：当HrtRaGetTlsEnable返回非NOT_SUPPORT的错误码时，tlsEnable保持初始值false导致错误设置为DISABLE而非UNKNOWN。

### 3d83a22d51dcc6203e59f1c26654fcd9e24fabfc [Master] Fix acl interfaces
- 根因类别：API废弃/迁移（上游接口变更适配）
- 涉及文件：src/platform/legacy/common/legacy_common.cc, test/ut/stub/llt_hccl_stub.cc
- 缺陷描述：`FftsPlusTaskLaunchWithFlag`原先调用`rtFftsPlusTaskLaunchWithFlag`接口，该接口被废弃需迁移到`rtGeneralCtrl`。新接口需将fftsPlusTaskInfo和stm打包到`uintptr_t input[2]`数组再调用`rtGeneralCtrl(input, 2, RT_GNL_CTRL_TYPE_FFTS_PLUS)`。同时清理UT stub中废弃函数的桩定义。
- 修复模式：API迁移——旧接口替换为新接口，适配参数传递约定。
- 可审查性：高
- 审查规则建议：上游SDK发布breaking changes时CI构建应自动检测链接失败；建立接口迁移checklist确保所有调用点同步更新。

### e1e70e6e63c0fb0a5a0a4bade2d03fa038953f12 fix dead lock issue
- 根因类别：死锁（锁持有范围过大，嵌套获取互斥锁）
- 涉及文件：src/framework/hcom/hcom_common.cc
- 缺陷描述：`HcomGetCommHandleByGroup`通过unique_lock获取`opGroupMapMutex`后，缓存未命中时继续持有该锁调用`HcomGetCommByGroup`。而后者内部获取`groupParamsLock`并可能调用`HcclGetCommHandle`等函数。如果其他线程以相反顺序获取这两把锁就会死锁。
- 修复模式：缩小锁持有范围——缓存未命中分支在调用外部函数前显式`lock.unlock()`释放锁。
- 可审查性：中
- 审查规则建议：unique_lock/lock_guard作用域内调用外部函数时检查被调函数是否可能获取其他锁；建立锁获取顺序约定；"缓存查询未命中则回退到重量级操作"模式应在查询完成后立即释放锁。

### bc8c503673ed7c8a2d8cf4b640991a71a4ea5fbf fix onesided memreg coredump
- 根因类别：错误处理路径资源泄漏（缺少rollback）
- 涉及文件：src/platform/resource/transport/onesided/hccl_mem.cc
- 缺陷描述：`HcclMemRegIpc`和`HcclMemRegRoce`中，将localBuffer插入localRmaBufferMgr后调用`Init()`初始化。如果Init()失败，原代码直接返回错误码但没有`Del(tempKey)`清理已插入的未初始化buffer。残留的不完整对象在后续访问时导致coredump。
- 修复模式：Init()失败后先执行Del(tempKey)删除已插入但初始化失败的buffer再返回错误。
- 可审查性：高
- 审查规则建议：容器insert后的初始化如果可能失败，失败路径必须有对应的erase/Del回滚操作。

### f9bd259967f26e5e615c02e199c8117efd905f9e Fix RsGetHccnCfg ENOENT issue
- 根因类别：错误码语义错误（可选配置不存在被当致命错误）
- 涉及文件：src/platform/hccp/rdma_service/rs.c
- 缺陷描述：`RsGetHccnCfg`调用`FileReadCfg`读取HCCN配置文件，当返回`FILE_OPT_INNER_PARAM_ERR`或`FILE_OPT_SYS_READ_FILE_ERR`时原代码返回`-ENOENT`。但这两种错误本质上是"配置文件不存在或无法读取"，在某些部署环境下是正常情况，返回-ENOENT导致调用方将其视为致命错误。修复改为返回0与其他非致命读取失败保持一致。
- 修复模式：错误码语义修正。
- 可审查性：中
- 审查规则建议：配置文件读取函数的错误处理需区分"可选配置不存在"和"必选配置读取异常"；同一函数内相似错误条件检查返回值是否一致。

### 28f18ab5715f0b8e6964c79c4996d7fcc69b45b8 fix changelink for OpRetry WaitResume
- 根因类别：全局变量竞态 + 条件分支过度区分
- 涉及文件：opretry_base.h, opretry_manager.cc, opretry_server.cc, hccl_communicator.cc/h, task_abort_handler.cc, aicpu_kfc_def.h, aicpu_hccl_sqcq.cc/h
- 缺陷描述：原设计通过全局变量`g_isRdmaError`传递RDMA错误状态。多group并发时存在竞态条件：一个group的RDMA错误状态影响另一个group的判断；task_abort_handler中重置时机与读取存在竞态。且非RDMA错误场景下不执行changelink，但实际OpRetry WaitResume后都应执行。修复彻底移除全局变量，`isChangedLink`无条件设为true，合并两条分支为统一changelink路径。
- 修复模式：消除全局状态 + 简化条件分支。
- 可审查性：中
- 审查规则建议：跨线程通信审查全局变量是否存在竞态，优先使用上下文对象传递状态；多条件分支最终执行相同逻辑时考虑合并为无条件执行。

### 8d2d8157588ba983e94c7ff27a6fc9b036bcd3bc fix alltoallV 8server not Success
- 根因类别：host-device参数传递内存布局不匹配
- 涉及文件：src/framework/common/src/launch_aicpu.cc/h, src/framework/communicator/impl/hccl_communicator_host.cc, hccl_one_sided_service.cc, src/framework/device/hccl_aicpu_interface.cc, src/platform/common/launch_aicpu.cc/h
- 缺陷描述：AlltoAllV 8server场景下，`AicpuAclKernelLaunch`通过`aclrtLaunchKernelWithConfig`传递参数，但AICPU侧按`KFCTaskComm`结构体（context指针+tilingData指针）解析。aclrtLaunchKernelWithConfig传递的是设备侧地址引用，而AICPU需要连续的host内存布局。修复新增`AicpuAclKernelLaunchV2`使用`aclrtLaunchKernelWithHostArgs`接口，在host侧将context(u64)和tilingData拼接到连续thread_local缓冲区中一次性传递，AICPU侧改为偏移解析。
- 修复模式：新增适配函数 + 修改内存布局匹配方式（host拼接连续buffer + device偏移解析）。
- 可审查性：低
- 审查规则建议：host与device间参数传递时审查两端对数据结构内存布局假设是否一致；审查reinterpret_cast和指针偏移确认双方解析方式匹配。

### 036e1118898536ae3d2b8b06c7c1fce9cbb4fa76 [Master] Fix seninfo in test
- 根因类别：UT桩文件与源码头文件不同步
- 涉及文件：test/ut/depends/include/platform/platform_info.h, platform_infos_lite_def.h
- 缺陷描述：UT桩代码中定义了`PlatFormInfosLite`相关接口和类型（SocVersion枚举、TOTAL_SOC_COUNT、SOC_VERSION_STR等），但上游源码已移除或重构这些接口导致UT编译失败。修复删除桩文件中过时的定义使其与当前源码对齐。
- 修复模式：删除过时的测试桩定义，与上游接口对齐。
- 可审查性：高
- 审查规则建议：修改公共头文件接口时CI应同时编译UT桩代码确保同步；可建立lint规则检查test/ut/depends/include/下头文件与源码头文件接口一致性。

---

## 第7轮：commits #121-140

### 5294bc21 310p memtype bugfix
- 根因类别：接口/类型误用（ACL层枚举 vs RT层枚举混用）
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：在310P芯片场景下，HcomGetMemType函数为内存类型赋值时使用了ACL层枚举（ACL_MEM_TYPE_LOW_BAND_WIDTH | ACL_MEM_MALLOC_NORMAL_ONLY_P2P），但实际应使用Runtime层的RT_MEMORY_P2P_DDR。ACL层枚举与RT层枚举的数值和语义不一致，导致分配到错误类型的内存。
- 修复模式：将错误的ACL枚举组合替换为正确的RT层常量RT_MEMORY_P2P_DDR。
- 可审查性：中
- 审查规则建议：对memType赋值时检查所使用的枚举常量是否与该变量最终传递给的API层级(ACL/RT)一致；禁止在RT层上下文中混用ACL层枚举。

### 241bb1af seninfo bugfix
- 根因类别：敏感信息泄露/命名规范违规
- 涉及文件：src/algorithm/impl/operator/all_gather_operator.cc, src/algorithm/impl/operator/reduce_scatter_operator.cc, test/(多个测试文件)
- 缺陷描述：代码注释中包含内部产品版本号"1520"（//待1520出版本），属于敏感信息外泄风险。同时测试用例中使用了内部芯片代号"910C"作为标识符，应统一为脱敏后的"910_93"。
- 修复模式：替换注释中的敏感版本号为通用描述；将测试函数名中的"910C"统一重命名为"910_93"。
- 可审查性：高
- 审查规则建议：设置静态扫描规则禁止代码中出现内部产品代号/版本号（正则匹配已知敏感代号列表）；提交前自动扫描。

### 6fd8a56b fix graph mode bug to refresh memory
- 根因类别：执行模式差异导致的逻辑遗漏
- 涉及文件：src/framework/device/framework/aicpu_communicator.cc, aicpu_communicator.h
- 缺陷描述：PrepareUserMemRanges在图模式下仅设置了当前rank的user memory range，未通过transport link获取remote ranks的user memory地址。导致图模式下算子展开缓存中的remote user memory信息缺失，缓存命中后replay时使用过期内存地址。原先只处理了ZeroCopy场景，遗漏了图模式场景。
- 修复模式：新增图模式分支，遍历opTransportResponse获取remote ranks的user memory地址；将AlgResourceResponse参数逐层透传到PrepareUserMemRanges。
- 可审查性：低
- 审查规则建议：新增workflow mode时审查所有按mode分支处理的函数确认各模式下数据准备逻辑完整；对涉及缓存key/value构建的函数要求覆盖所有执行模式的集成测试。

### 1bd3e2b5 bugfix of undefined symbol HcomSelctAlg
- 根因类别：ABI兼容性/跨动态库符号导出问题
- 涉及文件：src/framework/hcom/hcom.cc
- 缺陷描述：HcomSelectAlg函数参数algName使用std::string &类型，但该函数作为动态库导出符号被外部调用。std::string在不同编译器版本/ABI下内存布局可能不同，跨动态库边界传递std::string引用导致undefined symbol或ABI不兼容。
- 修复模式：将参数从std::string &改为char *，内部用临时std::string接收结果后memcpy_s拷贝到调用方buffer。
- 可审查性：高
- 审查规则建议：动态库导出函数参数和返回值禁止使用STL类型（std::string, std::vector等），必须使用C兼容的POD类型；静态分析工具在编译期检查导出符号签名。

### 3b673cf8 修复资源释放
- 根因类别：资源泄露/设备释放遗漏
- 涉及文件：examples/01_communicators/03_one_device_per_pthread/main.cc
- 缺陷描述：示例代码释放了Device内存、Host内存、Stream等资源后，未调用aclrtResetDevice重置设备，导致设备资源（Device Context等）未被正确释放，多线程多设备场景下资源泄露。
- 修复模式：在资源释放末尾追加aclrtResetDevice调用。
- 可审查性：高
- 审查规则建议：对ACL设备生命周期建立配对检查——aclrtSetDevice必须有对应的aclrtResetDevice；lint规则扫描资源获取/释放配对完整性。

### 89faf71c fix seninfo
- 根因类别：命名不一致/标识符重命名遗漏
- 涉及文件：test/llt/st/, test/llt/ut/, test/stub/ (7个文件)
- 缺陷描述：设备类型从旧命名(910c/910C/910D)迁移到新命名(910_93/910_95)时，测试代码中的test case名称和变量名未同步更新。内部逻辑已使用DEV_TYPE_910_93，但test case命名仍沿用旧名。
- 修复模式：批量字符串替换旧设备标识符为新标识符，纯重命名。
- 可审查性：高
- 审查规则建议：设备类型枚举重命名时自动扫描全仓库（含测试和stub）中引用旧名称的字符串字面量；CI lint规则检测已废弃设备类型名称。

### 18eabd74 fix aiv bug
- 根因类别：多处逻辑缺陷组合修复（接口参数遗漏、条件判断错误、资源管理缺陷）
- 涉及文件：src/algorithm/base/alg_aiv_template/(6个头文件), src/algorithm/impl/coll_executor/(2个文件), src/algorithm/impl/operator/all_reduce_operator.cc, src/framework/communicator/impl/hccl_communicator.cc, hccl_communicator_host.cc, src/framework/op_base/src/op_base.cc
- 缺陷描述：至少5个独立子缺陷：(1) 6个AIV模板中SyncAll调用缺少blockdim_参数，使用错误的默认block维度；(2) all_reduce_operator.cc中条件逻辑取反错误（multiModuleDiffDeviceNumMode_前漏写!），导致多模块异构场景错误启用AIV确定性算法；(3) CalBlockDim校验不区分910_93和其他设备；(4) SetAivCoreLimit误拒绝合法的0值输入；(5) ExecOp中aivCoreLimit获取逻辑错误，且缺少操作结束后的重置导致核数限制泄漏。
- 修复模式：接口签名补参、条件逻辑修正、设备类型分支细化、资源状态重置。
- 可审查性：低
- 审查规则建议：单个commit应只修复一个独立缺陷（此commit混合5个修复增加review难度）；函数签名变更时CI应自动检测所有调用点；布尔条件取反(!)需显式注释语义。

### a0e420e9 [BUGFIX] fix notify offset calculation error
- 根因类别：硬件地址映射计算错误
- 涉及文件：src/platform/common/adapter/adapter_rts.cc, adapter_rts.h
- 缺陷描述：hrtNotifyGetOffset中使用offset = notifyId * notifySize线性计算，但910B和910_93设备上notify地址空间是分slice组织的（每512个notify一个slice，每slice 64KB）。正确计算为offset = (notifyId % 512) * notifySize + (notifyId / 512) * 64KB。notifyId >= 512时原计算产生错误偏移。
- 修复模式：提取hrtCalNotifyOffset函数，按设备类型区分线性/分段计算。
- 可审查性：高
- 审查规则建议：硬件地址偏移计算应有UT覆盖slice边界（notifyId = 511, 512, 1023等）；新设备引入时检查所有硬件地址计算逻辑；常量应附带硬件spec来源注释。

### e77fe1b9 fix tbe_reduce deinit double free
- 根因类别：并发安全缺陷 + double free
- 涉及文件：src/platform/legacy/hccl_tbe_task/tbe_crack_cleared.cc/.h, tbe_vector_reduce.cc/.h
- 缺陷描述：TbeCrackCleard和TbeVectorReduce的Init/DeInit通过isInit_/isDeInit_标志位短路返回但无锁保护，多线程并发时可能重复初始化或重复释放。同时opNameStubFuncsMap_遍历释放时未检查iter->second是否为nullptr，多次DeInit导致double free。另外日志中CrackInit误写为"TbeVectorReduce had been initialized"（复制粘贴错误）。
- 修复模式：为Init/DeInit添加互斥锁；释放资源前增加nullptr检查；修正日志信息。
- 可审查性：高
- 审查规则建议：所有"init标志位短路返回"模式检查是否有并发场景并加锁；遍历容器释放资源后应clear()而非仅置nullptr；检查日志中的类名与当前类一致。

### 17364815 fix cmake bug
- 根因类别：CMake变量作用域 + 参数格式错误
- 涉及文件：cmake/func.cmake
- 缺陷描述：(1) function内使用CMAKE_CURRENT_LIST_DIR在被其他目录调用时解析为调用方目录而非func.cmake所在目录；(2) "-D _MANIFEST_FILE=..."中-D和变量名之间有空格且被引号包裹，某些CMake版本会解析为两个参数。
- 修复模式：在文件顶层缓存CMAKE_CURRENT_LIST_DIR到普通变量；修正-D参数格式为无空格形式。
- 可审查性：中
- 审查规则建议：CMake function/macro中禁止直接使用CMAKE_CURRENT_LIST_DIR应在文件顶部缓存；-D参数统一使用无空格格式(-DVAR=value)。

### 583d07c7 fix bug
- 根因类别：初始化顺序错误
- 涉及文件：src/framework/communicator/impl/hccl_communicator.cc, hccl_communicator_host.cc, src/platform/comm_primitive/hccl_dispatcher_ctx.cc
- 缺陷描述：dispatcher和dispatcherCtx初始化放在InitTransportManager()中，但TransportManager构造依赖dispatcher已就绪，导致使用时尚未初始化。同时DestroyDispatcherCtx中缺少已销毁的判断可能重复销毁，日志tag从[CreateCtx]误写为destroy场景。
- 修复模式：将dispatcher初始化提前到InitTransportManager()之前；增加防御性检查避免重复销毁；修正日志标签。
- 可审查性：低
- 审查规则建议：review时检查被构造函数使用的成员是否已在调用前完成初始化；销毁函数应具备幂等性。

### 27f313ac [UT] Fix acl_rt.h
- 根因类别：UT mock头文件与实际SDK不同步
- 涉及文件：test/ut/CMakeLists.txt, test/ut/depends/include/acl/acl_rt.h, test/ut/platform/resource/dispatcher/ut_aicpu_dispatcher_utest.cc
- 缺陷描述：UT mock acl_rt.h缺少生产代码依赖的新增API声明（aclrtFreeWithDevSync等），枚举值缺少显式赋值导致隐式值与生产代码不一致，aclrtStreamAttr缺少新成员。UT依赖已变更的profiling接口导致编译失败。
- 修复模式：同步mock头文件到最新SDK版本，补齐缺失声明；移除过时接口依赖；补充构建include路径。
- 可审查性：中
- 审查规则建议：mock/stub头文件应有自动化机制确保与真实SDK同步；枚举成员应始终显式赋值避免隐式值漂移。

### 88c6c63e [Fix] Fix reduce op enum
- 根因类别：枚举映射表与枚举定义不匹配（数组下标越界/错位）
- 涉及文件：src/pub_inc/aicpu/aicpu_hccl_sqcq.h
- 缺陷描述：RK_MAP_TABLE以HcclReduceOp枚举值为下标但只有3个元素（SUM=0, MAX=2, MIN=3），遗漏了PROD=1和RESERVED=4的条目。导致下标访问错位：RK_MAP_TABLE[2]本应映射MAX实际取到MIN；访问[3]或[4]越界。
- 修复模式：为不支持的枚举值填充ACL_RT_MEMCPY_INVALID占位元素，保证下标对齐。
- 可审查性：高
- 审查规则建议：枚举到数组的映射表元素数量必须与枚举范围完全匹配，每个元素应注释对应枚举值；建议static_assert校验数组大小等于枚举最大值+1。

### 79c68964 [Build] Fix pack_targets_and_files
- 根因类别：CMake变量使用错误
- 涉及文件：cmake/func.cmake
- 缺陷描述：pack_targets_and_files函数中使用CMAKE_CURRENT_FUNCTION_LIST_DIR引用脚本路径，该变量在跨文件调用时语义不直观且有版本兼容性问题。
- 修复模式：替换为CMAKE_CURRENT_LIST_DIR加显式子路径拼接。
- 可审查性：高
- 审查规则建议：CMake脚本中优先使用CMAKE_CURRENT_LIST_DIR而非CMAKE_CURRENT_FUNCTION_LIST_DIR；引用cmake脚本路径时确保在不同调用上下文中路径稳定。

### 500ab5f6 fix msprof profiling issue
- 根因类别：资源销毁时无条件设备上下文切换
- 涉及文件：src/framework/hcom/hcom_common.cc
- 缺陷描述：HcomDestroy中无条件调用hrtSetDevice(logicId)设置设备，但当进程中已有活跃设备上下文时，无条件SetDevice可能导致msprof profiling异常（切换上下文导致profiling数据丢失或状态错乱）。
- 修复模式：先hrtGetDevice检查是否已有设备上下文，仅在没有时才SetDevice。
- 可审查性：中
- 审查规则建议：SetDevice调用前应检查当前设备状态避免不必要的上下文切换；销毁路径中的设备操作需评估对其他子系统的副作用。

### a13340fb fix hdc nullptr and dispatcher unsupport
- 根因类别：空指针解引用 + 条件判断缺失
- 涉及文件：src/framework/communicator/impl/hccl_communicator.cc, hccl_communicator.h, hccl_communicator_host.cc
- 缺陷描述：(1) InitTransportManager中未根据设备类型和ffts开关做前置判断，不支持ffts的设备也尝试以ffts模式创建dispatcher；(2) GetHDCommunicate直接对未初始化的kfcControlTransferH2D_/kfcStatusTransferD2H_智能指针做空检查并返回错误，但不支持HD通信的设备上这些指针本就为nullptr应跳过而非报错。
- 修复模式：增加前置条件守卫(guard clause)；提取GetSupportHDCommunicate方法统一判断逻辑。
- 可审查性：中
- 审查规则建议：对智能指针执行空检查前确认该指针在所有代码路径上是否被初始化；同一条件表达式多处重复时应提取为独立方法。

### 90610a52 fix error log for nslb
- 根因类别：资源释放时序错误 + 功能守卫缺失
- 涉及文件：src/framework/op_base/src/op_base.cc, src/platform/hccp/rdma_agent/client/ra_tlv.c
- 缺陷描述：(1) HcclCommDestroy在comm销毁完成后调用DeinitNetCo()/DeinitHccpChannel()，但此时通信资源可能已释放，产生nslb错误日志；(2) RaTlvRequest中未拦截不支持的TLV_MODULE_TYPE_NSLB类型请求。
- 修复模式：移除不当的析构调用；增加不支持类型的前置拦截返回-EINVAL。
- 可审查性：高
- 审查规则建议：资源销毁调用顺序需与初始化顺序严格相反；TLV/RPC分发器对所有module type应有明确的支持/不支持分支。

### 859549ac fix verson
- 根因类别：ABI破坏性变更回退
- 涉及文件：inc/hccl/hccl_comm.h, hccl_types.h, src/framework/communicator/(多个文件)
- 缺陷描述：先前在公共头文件hccl_types.h的HcclCommConfigDef结构体中新增了commEngine/threadNum/notifyNumPerThread字段，改变了内存布局导致与旧版本二进制不兼容（ABI break）。同时rank_graph.cc中DEV_TYPE_910和910_95落入不支持的default分支（实际910应走ROCE、910_95应走UB_CTP）。
- 修复模式：回退公共API的ABI破坏性变更，将字段下沉到内部模块作为局部常量；补全switch-case设备类型分支。
- 可审查性：高
- 审查规则建议：公共头文件结构体修改必须做ABI兼容性审查，禁止在已发布结构体中间插入字段；switch-case中device type枚举落入default应触发告警。

### a37e6cf1 Revert HB check opinconsistent
- 根因类别：功能设计缺陷回退(Revert)
- 涉及文件：src/framework/cluster_maintenance/health/heartbeat/heartbeat.cc/.h, src/framework/communicator/(多个文件)
- 缺陷描述：心跳算子一致性校验功能多方面问题：(1) OpInfoTagQueueFrame二维结构使心跳帧体积膨胀；(2) 发送逻辑存在busy-wait循环(120次loop每次sleep 100us)可能阻塞心跳线程；(3) inconsistentOpMap_/srTagMap_引入额外mutex和内存开销缺清理机制；(4) CheckOpInconsistentError接口设计有问题，上层处理逻辑不完备。
- 修复模式：Revert + 重构简化为扁平OpInfoDesc数组，移除busy-wait和冗余接口。
- 可审查性：低
- 审查规则建议：心跳等后台线程禁止busy-wait循环应使用非阻塞I/O；跨节点传输帧结构变更需评估带宽影响；新增对外接口需确保上层有完整调用和错误处理路径。

### cb448283 fix netlayer err
- 根因类别：输出参数未赋值（遗漏赋值语句）
- 涉及文件：src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：函数通过指针参数netLayers向调用方返回网络层信息，某分支中设置了netLayer_[0]和*netLayerNum = 1，却遗漏了*netLayers = netLayer_赋值，调用方拿到野指针。
- 修复模式：补全遗漏的输出参数赋值。
- 可审查性：高
- 审查规则建议：函数所有通过指针/引用返回的输出参数在每条返回路径上都必须被显式赋值；静态分析工具检测"函数返回前输出参数可能未初始化"的路径。

## 第8轮：commits #141-162 (22条，全量分析完结)

### 6770b07c fix the occasional cache failure issue
- 根因类别：逻辑缺陷 / 不恰当的缓存策略
- 涉及文件：src/algorithm/impl/coll_executor/coll_all_gather_v/*.cc, src/algorithm/impl/coll_executor/coll_reduce_scatter_v/*.cc, src/algorithm/pub_inc/coll_alg_param.h
- 缺陷描述：AllGatherV和ReduceScatterV的AIV executor在KernelRun中调用了SetOpCache设置操作缓存，同时在coll_alg_param.h的OpParam::operator<中为这两类操作实现了基于memcmp比较counts/displs数组的缓存key逻辑。由于V类操作的counts/displs每次可能不同，导致偶发的缓存失效或错误命中。
- 修复模式：移除V类操作的缓存设置调用和对应的缓存key比较分支，不再对variable-length集合操作做缓存。
- 可审查性：中
- 审查规则建议：对于参数中包含动态长度数组的操作，审查是否存在基于可变参数构建缓存key的逻辑；对memcmp做缓存key比较时，检查被比较的内存区域生命周期和有效性。

### 4a8aa636 fix aicpu comm destory timeout
- 根因类别：状态判断缺陷 / 异步初始化标志时序不一致
- 涉及文件：src/framework/communicator/hccl_comm_host.cc, src/framework/communicator/impl/hccl_communicator.cc/.h, src/framework/communicator/impl/hccl_communicator_device.cc, src/framework/communicator/impl/hccl_communicator_host.cc
- 缺陷描述：AICPU通信域销毁时使用布尔成员isIndOpCommInit_判断是否需要执行销毁。该标志在GetHDCommunicate被调用时设为true，但实际AICPU通信域初始化是异步的，标志仅表示"获取了HD通信参数"而非"实际初始化完成"。导致销毁流程误判需要destroy，发送kDestroyComm命令后等待响应超时。
- 修复模式：将静态布尔标志替换为std::function<bool()> getAicpuCommState_回调函数，绑定到IndependentOp::GetAicpuCommState()，在销毁前动态查询实际状态。
- 可审查性：高
- 审查规则建议：当布尔标志用于表示异步初始化完成状态时，审查标志设置时机是否确实在初始化完成之后；资源destroy流程中的前置条件判断，检查"设置标志"与"实际完成"之间是否存在时间窗口不一致。

### 2e95c956 [UT] Fix stub symbols
- 根因类别：编译链接缺陷 / 缺失stub符号
- 涉及文件：test/ut/stub/CMakeLists.txt, test/ut/stub/llt_hccl_stub_platform.cc(新增)
- 缺陷描述：UT编译链接时缺少platform模块相关符号(HcclTbeTaskInit、HcclTbeReduce、GraphMgrInit、LaunchGraph等)。新增stub文件提供空实现，并通过CMake FILTER排除legacy和typical目录源文件避免符号冲突。
- 修复模式：补充缺失的stub函数定义 + CMake过滤规则排除不需要参与UT编译的平台源文件。
- 可审查性：高
- 审查规则建议：新增源码模块或接口函数时，检查UT stub目录是否同步补充了符号定义；CMake中使用GLOB收集源文件时，检查是否有需要排除的目录。

### 4e59da23 fix graph err
- 根因类别：架构缺陷 / 初始化时序错误 + 数据类型不匹配
- 涉及文件：src/framework/communicator/(多个文件), src/framework/communicator/impl/independent_op/(多个文件), src/framework/hcom/hcom.cc, src/framework/inc/hccl_comm_pub.h
- 缺陷描述：RankGraph由IndependentOp拥有并初始化，通过间接访问。但HcclApi路径(非hcom)不会调用SetIndependentOpConfig，导致rankGraph未初始化。同时RankGraph内部使用旧数据类型RankInfo_t(含rankId字段)而非新类型RankInfo(含userRank字段)，排序函数使用了错误字段。
- 修复模式：将RankGraph从IndependentOp移到HcclCommunicator中，在communicator初始化时直接初始化；统一数据类型从RankInfo_t到RankInfo。
- 可审查性：中
- 审查规则建议：当某模块的初始化依赖于另一模块的配置步骤时，检查所有调用路径是否都会执行该配置；数据结构重构后检查所有引用点是否使用了新字段名。

### f7cdd1d8 LLT fix
- 根因类别：编译缺陷 / struct类型重命名后stub未同步
- 涉及文件：test/ut/CMakeLists.txt, test/ut/stub/llt_hccl_stub_gdr.cc, test/ut/stub/llt_hccl_stub_pub.h
- 缺陷描述：HCCP RDMA Agent库的头文件中约30处struct类型名从snake_case重命名为PascalCase(如mr_info→MrInfoT、tlv_init_info→TlvInitInfo等)，UT stub中的函数签名未同步更新导致编译失败。同时CMakeLists.txt中两个add_subdirectory被注释掉未恢复。
- 修复模式：批量更新stub函数签名中的struct类型名以匹配上游重命名。
- 可审查性：高
- 审查规则建议：上游依赖库进行struct/type重命名时应同步检查所有stub/mock文件；CMake中被注释掉的add_subdirectory应标注原因。

### 3a8b5e7f LLT fix
- 根因类别：编译缺陷 / API签名不匹配 + 构建脚本短路
- 涉及文件：CMakeLists.txt, build.sh, 约66个测试文件
- 缺陷描述：(1) build.sh中UT构建分支有一行exit 0导致UT编译完全被跳过。(2) CMakeLists.txt中有trailing whitespace。(3) test/stub/和test/llt/下60+个文件中struct名称未随上游HCCP库重命名同步更新。
- 修复模式：删除build.sh中的短路exit 0；批量更新所有测试文件中的struct类型名。
- 可审查性：高
- 审查规则建议：构建脚本中不应出现无条件exit 0跳过后续步骤，CI应检测UT是否实际执行；依赖库struct重命名时需全仓扫描所有引用点。

### b47f5c96 llt fix
- 根因类别：构建基础设施缺陷 / UT框架缺失
- 涉及文件：CMakeLists.txt, build.sh, build_third_party.sh, cmake/(新增3个文件), test/ut/(大量新增和迁移), 约300个文件变更
- 缺陷描述：UT构建框架从旧的test/llt/迁移到新的test/ut/。原有UT CMakeLists.txt几乎为空无法编译。本次重建CMake配置、迁移stub文件、内置外部依赖头文件、完善构建脚本。
- 修复模式：UT构建基础设施重建——迁移目录结构、补全CMake配置、内置外部依赖头文件。
- 可审查性：低
- 审查规则建议：UT构建框架变更应确保CI中UT能实际编译运行；stub文件迁移时验证旧路径不再被引用。

### 19e2e2a4 fix GetLogicIdByPhyId
- 根因类别：API调用错误 / 函数名调反
- 涉及文件：src/platform/common/adapter/adapter_rts.cc
- 缺陷描述：在hrtGetDeviceIndexByPhyId函数中，需要根据物理设备ID获取逻辑设备ID，但代码错误地调用了aclrtGetPhyDevIdByLogicDevId(根据逻辑ID获取物理ID)，传参方向完全相反。
- 修复模式：将调用的API从aclrtGetPhyDevIdByLogicDevId改为aclrtGetLogicDevIdByPhyDevId。
- 可审查性：高
- 审查规则建议：当函数名包含方向性语义(如ByPhyId/ByLogicId)时，审查调用处是否与当前上下文的转换方向一致。

### 43ab234d fix legacy include
- 根因类别：构建配置错误 / 路径硬编码 + 架构特定路径
- 涉及文件：build.sh, src/CMakeLists.txt, src/framework/CMakeLists.txt, src/algorithm/base/alg_aiv_template/CMakeLists.txt
- 缺陷描述：(1) build.sh中CPU_NUM计算使用*2导致并行编译job数过大。(2) legacy头文件include路径指向已不存在的源码目录，需改为构建产物目录。(3) alg_aiv_template/CMakeLists.txt中将x86_64-linux和aarch64-linux两套架构路径同时硬编码(共16行重复x9个target)。
- 修复模式：将硬编码架构路径替换为${CMAKE_HOST_SYSTEM_PROCESSOR}变量；将legacy头文件路径改为构建输出目录。
- 可审查性：高
- 审查规则建议：检测CMakeLists.txt中硬编码的x86_64-linux或aarch64-linux路径，提示使用CMake变量替代。

### aca1fa1e rsv and hb fix
- 根因类别：逻辑错误 / 三目运算符分支写反 + 格式化字符串类型不匹配
- 涉及文件：src/algorithm/impl/coll_executor/coll_reduce_scatter_v/coll_reduce_scatter_v_deter_executor.cc, src/framework/cluster_maintenance/health/heartbeat/heartbeat.cc, src/framework/common/src/topo/topoinfo_exchange_base.cc, src/framework/op_base/src/op_base.cc
- 缺陷描述：(1) CalcTransportMemType函数在scratchMemFlag_为true时，mesh拓扑和非mesh拓扑的outputType赋值反了，三目运算符true/false分支写颠倒。(2) heartbeat.cc中HCCL_INFO用%s打印uid_但uid_不是C字符串。(3) topoinfo中"failed"拼写为"faild"。(4) op_base.cc中多余的CommCheckOpInconsistentError调用。
- 修复模式：交换三目运算符两个分支值；对非字符串类型使用正确格式化转换函数；修正拼写；移除不当调用。
- 可审查性：高
- 审查规则建议：三目运算符的true/false分支进行语义一致性检查，特别是条件名和分支值存在命名对应关系时；检测日志宏中%s格式符对应的参数是否为const char*。

### d16c139a [FIX] when system nic ip not set, get backupIpList core
- 根因类别：空容器未校验导致越界访问(coredump)
- 涉及文件：src/framework/common/src/onesided_memory_management/global_mem_manager.cc
- 缺陷描述：CheckOneSidedBackupAndSetDevId函数调用hrtRaGetDeviceAllNicIP获取设备IP列表后，未检查chipDeviceIPs是否为空就直接用下标访问。当系统NIC IP未配置时，空vector下标访问触发越界导致coredump。
- 修复模式：在下标访问前增加empty()空判断，为空时提前返回。
- 可审查性：高
- 审查规则建议：对vector/数组执行下标访问前必须有空/长度校验；特别是数据来源于外部接口时必须做防御性检查。

### 4a0c9385 TaskException bugfix
- 根因类别：C++性能缺陷 / range-for值拷贝代替引用
- 涉及文件：src/common/debug/profiling/task_exception_handler.cc
- 缺陷描述：for循环遍历ctxInfoArray时循环变量声明为std::vector<CtxInfo>值类型而非引用，导致每次迭代拷贝副本，reserve(100)只作用于副本而非原始容器。
- 修复模式：将循环变量从值类型改为引用类型&。
- 可审查性：高
- 审查规则建议：range-based for循环中对元素进行修改时循环变量必须声明为引用；可配置clang-tidy的performance-for-range-copy规则检测。

### c6141265 修复hccd缺符号
- 根因类别：条件编译缺失 / 链接符号缺失
- 涉及文件：src/platform/resource/transport/transport_base.cc
- 缺陷描述：TransportBase构造函数中调用了GetDispatcherCtx()和GetDispatcher()，但这些函数在hccd构建场景下没有符号定义，导致链接失败。
- 修复模式：用#if条件编译包裹dispatcher_相关的初始化代码块，在hccd编译模式下跳过。
- 可审查性：中
- 审查规则建议：新增使用外部符号的代码时，检查该符号在所有编译target中是否有定义。

### d74718ce fix aclrtMemcpyAsyncWithOffset
- 根因类别：API参数类型错误 / 指针语义混淆
- 涉及文件：src/platform/common/adapter/adapter_rts.cc
- 缺陷描述：hrtMemcpyAddrAsync函数调用aclrtMemcpyAsyncWithOffset时，将dst和src使用&取地址传入，但API期望void**类型。&dst传的是栈上局部变量的地址而非设备内存指针的间接引用。
- 修复模式：移除&取地址操作，改为使用reinterpret_cast将void*转换为API要求的void**/const void**类型。
- 可审查性：中
- 审查规则建议：调用外部API时检查参数是否通过&取局部变量地址传递给二级指针参数；对reinterpret_cast的使用需审查目标类型与API签名是否一致。

### 41853d3d fix openssl compile fail without cache
- 根因类别：构建配置缺陷 / CMake变量作用域 + 多余依赖
- 涉及文件：cmake/third_party/openssl.cmake, src/platform/hccp/rdma_service/CMakeLists.txt, src/platform/hccp/inc/network/(删除2个头文件)
- 缺陷描述：(1) openssl.cmake中OPENSSL_SRC_DIR变量仅在if(EXISTS)分支内设置，else分支(无缓存需下载源码)使用不同路径，导致无缓存时编译失败。(2) rdma_service链接了实际不需要的ssl_static库。
- 修复模式：将变量定义提升到条件分支之前统一赋值；移除多余的ssl_static链接；清理无用头文件。
- 可审查性：高
- 审查规则建议：CMake变量在条件分支中定义时，检查所有分支是否都对该变量一致赋值；链接库变更时确认所有编译场景均能通过。

### 35b1d12c fix utils version wrong
- 根因类别：配置/版本号硬编码错误
- 涉及文件：cmake/utils.cmake
- 缺陷描述：utils依赖包版本号硬编码为8.5.0.alpha001，但实际发布版本已更新为8.5.0。导致GLOB匹配包名、SIMULATOR_VERSION变量、下载URL三处均使用过期版本号。
- 修复模式：将3处alpha版本号统一替换为正式版本号。
- 可审查性：高
- 审查规则建议：cmake中硬编码版本号字符串在多处出现时检查是否一致；包含alpha/beta/rc等预发布标识时在正式分支上应告警。

### e6d12932 HCCE A5 comm interface bugfix
- 根因类别：接口签名与跨层调用不匹配(C++ ABI兼容性)
- 涉及文件：pkg_inc/hccl/hcom.h, src/framework/hcom/hcom.cc, src/framework/hcom/hcom_common.cc, src/framework/hcom/hcom_pub.h, test/llt/(多个测试文件)
- 缺陷描述：(1) HcomExecSelectAlg使用bool&和std::string&(C++引用)，但需被C侧/AICPU侧调用，改为指针类型保证ABI兼容。(2) 调用处引用了未定义的algNameStr变量而非algName参数。(3) 公共头文件暴露了未实现/不应暴露的接口声明。(4) CCL Buffer的A5芯片特殊处理逻辑需内联到通用流程。
- 修复模式：将C++引用参数改为指针；删除公共头文件中不应暴露的接口；将芯片特定逻辑按设备类型分支内联。
- 可审查性：中
- 审查规则建议：公共头文件(pkg_inc/)中extern函数签名不应使用std::string&或C++引用类型参数；检测函数调用中是否引用了未声明的变量名。

### 5922b099 add Release/Debug
- 根因类别：构建系统缺陷 / 缺少Debug构建类型支持
- 涉及文件：build.sh, cmake/config.cmake, src/CMakeLists.txt, 约12个子目录CMakeLists.txt
- 缺陷描述：构建系统不支持Debug构建类型。build.sh缺少--build-type参数解析；config.cmake没有设置CMAKE_BUILD_TYPE默认值；所有CMakeLists.txt的target_compile_options未包含Debug模式下的-g选项。
- 修复模式：添加--build-type参数解析；设置默认构建类型为Release；在所有target编译选项中添加$<$<CONFIG:Debug>:-g>生成器表达式。
- 可审查性：高
- 审查规则建议：检测CMakeLists.txt中是否缺少CMAKE_BUILD_TYPE的设置；所有target_compile_options应包含Debug配置下的-g选项。

### c16c28ab fix indpt ROCE bug
- 根因类别：设备类型适配缺失 + 日志信息不完整
- 涉及文件：src/framework/communicator/impl/independent_op/independent_op.cc, src/framework/device/framework/aicpu_communicator.cc
- 缺陷描述：(1) InitAicpuIndOp中notifySize_未根据设备类型初始化，910_93/910B应使用4字节notify，其他设备(如910_95)应使用8字节，缺少分支导致ROCE通信时notify大小错误。(2) SetIndependentOpConfig日志打印位置在参数赋值之前且缺少关键诊断信息。
- 修复模式：添加基于deviceType的notifySize_初始化分支；将日志打印移到赋值后并补充关键字段。
- 可审查性：中
- 审查规则建议：使用硬件相关常量时必须按设备类型做分支初始化；日志语句应位于相关变量赋值之后。

### a5cb0451 independent Op channel bugfix
- 根因类别：多个子缺陷——空指针检查条件反转 + 缺少本地内存信息 + 硬编码临时值
- 涉及文件：src/framework/communicator/impl/independent_op/channel/channel_manager.cc, channel_param.h, src/framework/device/framework/aicpu_communicator.cc, src/platform/resource/transport/host/transport_ibverbs.cc
- 缺陷描述：(1) CHK_PRT_RET(remoteMem, ...)条件反转——remoteMem非空时条件为true会误返回错误，应为remoteMem == nullptr。(2) BuildOpRemoteChannelRoceResParam缺少GetLocalMemDetails调用。(3) AicpuChannelInit中deviceLogicId/devicePhyId/deviceType使用硬编码0赋值且deviceLogicId被重复赋给devicePhyId。(4) localInputMem/localOutputMem未赋值(仅有"待确认"注释)。
- 修复模式：修正空指针检查条件；添加localHcclbuffer字段并填充本端内存；删除冗余临时硬编码字段。
- 可审查性：高
- 审查规则建议：CHK_PRT_RET宏的第一个参数应为错误条件(指针检查必须用ptr == nullptr而非ptr)；结构体字段不应有"临时赋值"注释和硬编码值0；TODO/待确认注释不应合入正式代码。

### 28c19c7a TemplateType Bugfix
- 根因类别：枚举值定义顺序错误
- 涉及文件：src/algorithm/base/alg_template/alg_template_base_pub.h
- 缺陷描述：TemplateType枚举中TEMPLATE_ALL_GATHER_V_GRAPH_PIPELINE = 101被放在TEMPLATE_ALL_2_ALL_V_CONTINUOUS_PIPELINE = 100之前，破坏枚举值递增顺序，可能影响依赖枚举顺序的区间判断逻辑。
- 修复模式：交换两个枚举项定义顺序，保持按数值递增排列。
- 可审查性：高
- 审查规则建议：枚举定义中显式赋值的枚举项应按数值严格递增排列。

### 830d044c HCCE bugfix
- 根因类别：头文件层级划分错误——内部函数声明暴露在公共头文件
- 涉及文件：pkg_inc/hccl/hcom.h, src/framework/hcom/hcom_pub.h
- 缺陷描述：GenerateCclOpTag、HcomCalcTaskNum、HcclCommCalcTaskNum、HcomExecSelectAlg等内部实现函数被声明在公共对外头文件pkg_inc/hccl/hcom.h中，违反接口封装原则。
- 修复模式：将内部函数声明从pkg_inc/hccl/hcom.h移至src/framework/hcom/hcom_pub.h。
- 可审查性：高
- 审查规则建议：pkg_inc/目录下的公共头文件不应包含内部实现函数的声明；新增函数到pkg_inc/时审查是否确实属于对外API。
