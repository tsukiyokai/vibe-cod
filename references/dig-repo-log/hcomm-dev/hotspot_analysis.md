# hcomm-dev 代码热点与结构性风险分析

## 一、热点文件统计

数据来源: defect_analysis.md，共157条缺陷条目（含5条无涉及文件记录），涉及239个不同文件，总触及370次。

### 1.1 Top 20 缺陷磁铁文件

| 排名 | 触及次数 | 文件路径 |
|------|----------|----------|
| 1 | 14 | src/framework/communicator/impl/hccl_communicator_host.cc |
| 2 | 14 | src/framework/hcom/hcom.cc |
| 3 | 12 | src/framework/device/framework/aicpu_communicator.cc |
| 4 | 11 | src/framework/op_base/src/op_base.cc |
| 5 | 7 | src/platform/common/adapter/adapter_rts.cc |
| 6 | 6 | pkg_inc/hccl/hcom.h |
| 7 | 6 | src/CMakeLists.txt |
| 8 | 5 | src/framework/communicator/impl/independent_op/hccl_independent_rank_graph.cc |
| 9 | 5 | src/algorithm/impl/operator/coll_alg_operator.cc |
| 10 | 5 | src/framework/CMakeLists.txt |
| 11 | 5 | src/framework/communicator/impl/hccl_communicator.cc |
| 12 | 4 | src/framework/communicator/impl/independent_op/hccl_independent_op_mem.cc |
| 13 | 4 | src/platform/comm_primitive/hccl_dispatcher_ctx.cc |
| 14 | 4 | src/algorithm/impl/operator/alltoall_operator.cc |
| 15 | 3 | src/framework/communicator/impl/independent_op/hccl_independent_op_channel.cc |
| 16 | 3 | src/framework/inc/hccl_comm_pub.h |
| 17 | 3 | src/platform/hccp/rdma_service/rs.c |
| 18 | 3 | src/platform/task/dispatcher_aicpu.cc |
| 19 | 3 | src/framework/hcom/hcom_common.cc |
| 20 | 3 | (多个文件并列3次) |

- 频次 >= 2 的文件: 52个
- 频次 = 1 的文件: 187个
- 头部集中度: Top 4 文件合计被51次缺陷修复触及，占总触及次数的14%

### 1.2 模块维度聚合

| 触及次数 | 文件数 | 模块 | 平均触及/文件 |
|----------|--------|------|--------------|
| 143 | 64 | src/framework | 2.23 |
| 92 | 69 | src/platform | 1.33 |
| 36 | 26 | src/algorithm | 1.38 |
| 14 | 13 | test/ut | 1.08 |
| 13 | 11 | cmake | 1.18 |
| 11 | 10 | src/orion | 1.10 |
| 8 | 3 | pkg_inc | 2.67 |
| 4 | 4 | src/pub_inc | 1.00 |

framework模块的缺陷密度(2.23)远高于其他模块，是系统中最大的缺陷聚集区。

### 1.3 framework子模块细分

| 触及次数 | 文件数 | 子模块 | 平均触及/文件 |
|----------|--------|--------|--------------|
| 53 | 20 | src/framework/communicator | 2.65 |
| 23 | 11 | src/framework/device | 2.09 |
| 20 | 4 | src/framework/hcom | 5.00 |
| 14 | 4 | src/framework/op_base | 3.50 |
| 12 | 12 | src/framework/next | 1.00 |
| 7 | 6 | src/framework/cluster_maintenance | 1.17 |
| 6 | 5 | src/framework/common | 1.20 |

hcom子模块只有4个文件但平均触及5.0次/文件，是密度最高的缺陷聚集点。

### 1.4 platform子模块细分

| 触及次数 | 文件数 | 子模块 | 平均触及/文件 |
|----------|--------|--------|--------------|
| 31 | 26 | src/platform/hccp | 1.19 |
| 17 | 16 | src/platform/resource | 1.06 |
| 16 | 7 | src/platform/common | 2.29 |
| 11 | 10 | src/platform/legacy | 1.10 |
| 7 | 3 | src/platform/comm_primitive | 2.33 |
| 6 | 4 | src/platform/task | 1.50 |

platform的缺陷较为分散（hccp 31次/26文件 = 1.19），而common/adapter和comm_primitive密度偏高。

---

## 二、热点文件结构性风险审查

### 2.1 hccl_communicator_host.cc (8818行, 14次缺陷修复)

主要职责: HCCL通信域的host侧实现，负责集合通信算子的参数校验、资源分配、算法选择与执行编排。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 错误码吞没 | AicpuUnfold/AllReduceAicpuUnfold中Mc2AiCpuStreamAllocAndGet返回值被后续CreateCommResource覆盖，stream分配失败被静默忽略 | 2695, 2953 |
| 2 | 资源泄漏 | HcclGetAlgExecParam中hrtMalloc分配的device内存在CHK_RET失败提前返回时无法释放 | 1742-1812 |
| 3 | 函数复杂度 | ExecOp(230+行)和ExecOpAlltoAll(280+行)承担算法选择/资源创建/AIV处理/AICPU编排/DFX注册等多重职责 | 4280-4513, 4525-4800 |
| 4 | 错误处理 | SwitchNic中清理操作的CHK_RET可能丢弃原始错误码 | 8297-8302 |
| 5 | 并发安全 | SnapshotCheckPreProcess/PostProcess中busy-wait循环无sleep/yield，100%占用CPU | 8724-8806 |
| 6 | 死代码 | BatchSendRecv中IsAtomicInit()重复检查，第二次永远不触发（复制粘贴遗留） | 3864, 3878 |
| 7 | 并发安全 | g_enableBackupLinkCommCount的read-then-decrement非原子操作，并发析构可能underflow | 212-217 |
| 8 | 空指针 | implAlg_在AllGather/AllGatherOutPlace/AllReduceOutPlace等多个公共API中未做null检查就调用 | 2576, 2741, 3017等 |
| 9 | 资源管理 | 构造函数中mrManager_/zeroCopyAclGraph_的new(std::nothrow)分配失败仅记日志，对象带nullptr存活 | 124-132, 157-165 |
| 10 | 函数复杂度 | 析构函数超过140行，资源释放顺序依赖隐式假设 | 169-316 |
| 11 | 错误处理 | SwitchNic在ParseSwitchRanks失败后仍向设备发送可能包含垃圾值的changeLinkInfo | 8236-8303 |
| 12 | 并发安全 | Init函数的g_hcomInitMutex仅覆盖部分初始化操作，子通信域Init完全不使用该锁 | 344-405 |

### 2.2 hcom.cc (4068行, 14次缺陷修复)

主要职责: Hcom层入口，封装AllReduce/AllGather/Broadcast/Send/Receive等集合通信操作及通信域初始化、Group管理。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 并发安全(TOCTOU) | HcomCreateGroupImplHeterog: 持锁检查group不存在→解锁→CreateGroup→再加锁插入map，两个线程可同时通过检查导致重复创建 | 1488-1514 |
| 2 | 并发安全 | HcomReleaseSubComms: 直接遍历hcomInfo.hcomGroupMap而未持groupParamsLock，并发CreateGroup/DestroyGroup会导致迭代器失效 | 2089-2104 |
| 3 | 悬垂指针(确定性UAF) | HcomGetAlgorithm: `*algo = const_cast<char *>(str.c_str())`将局部变量str的内部指针返回给调用者，函数返回后即悬垂 | 2076 |
| 4 | 悬垂指针 | GetGroupNameByOpBaseHcom: `*groupname = const_cast<char *>(hcclComm->GetIdentifier().c_str())`若返回临时对象则立即悬垂 | 2800-2801 |
| 5 | 空指针(23处) | HcclCommGraph系列函数中reinterpret_cast<hcclComm*>(opBaseHcom)后大多不做null检查，仅2处有检查。用户传入0/无效句柄则crash | 888-2776(23处) |
| 6 | 内存泄漏 | HcomGetandClearOverFlowTasks: malloc后memcpy_s失败直接返回错误码，已分配内存未free | 2485-2494 |
| 7 | 内存泄漏 | HcomGetSplitStrategy: `*segmentIdxPtr = new u32[*len]`裸new交由调用者管理，无释放责任文档 | 1657 |
| 8 | 数据竞争 | g_rankTableSetInfo: 文件级static全局变量，Set/Get操作无任何锁保护，RankTable_t赋值非原子 | 2405 |
| 9 | 初始化不对称 | HcomInit: 部分backlogged groups已创建成功但HcomDestroy可能无法正确处理部分初始化状态 | 67-162 |
| 10 | 日志敏感信息 | HcomSetRankTableImpl和HcomInitByMasterInfo将完整rankTable/IP/端口以RUN_INFO级别明文打印 | 2408, 316-317 |

### 2.3 aicpu_communicator.cc (5809行, 12次缺陷修复)

主要职责: AICPU侧通信域管理器，负责初始化/销毁通信资源、编排集合通信算子执行、管理链路重试状态机和设备stream生命周期。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 数据错乱(确定性Bug) | UpdateSqStatus中SQ_TAIL查询结果写入head、SQ_HEAD查询结果写入tail，head/tail赋值反转导致重执行路径stream状态恢复错误 | 3385-3386 |
| 2 | 并发安全 | errMessageReport_作为static bool被多实例跨线程无锁读写（C++标准层面是UB） | 61, 2989, 4925, 4940 |
| 3 | 并发安全 | dfxExtendInfo_在背景线程HandleCqeException中写入、主线程CheckOpExecStatus中读取，无同步保护 | 4889-4892写, 4041读 |
| 4 | 并发安全 | isOpLaunch/endStopLaunch/needsResponseStopLaunch_/printTaskExceptionForErr_等bool标志无锁跨线程读写 | 2586, 2653, 2986-2988 |
| 5 | 并发安全 | GetAlgResponseRes中resMap_的double-check locking: 锁外读取resMap_时其他线程可能正在rehash | 2297-2318 |
| 6 | 资源管理 | 析构函数清理了dispatcher_但未考虑Transport对象析构对dispatcher_的依赖（析构顺序问题） | 78-106 |
| 7 | 资源管理 | opUnfoldCachePtr_使用裸new/delete而非RAII，Init()中间步骤失败可能导致泄漏 | 100-103, 160-161 |
| 8 | 函数复杂度 | Orchestrate状态机约190行，12-case switch内局部变量跨case隐式共享 | 2407-2594 |
| 9 | 错误处理 | ReportErrCqe中QuerySqStatusByType返回值被忽略，失败时head/tail保持初始值0导致后续越界 | 4904-4905 |
| 10 | 类型安全 | HcclOpSupportRetry返回bool但内部CHK_RET会返回HcclResult，非零错误码被隐式转为true（"支持重试"），语义完全相反 | 3230 |

### 2.4 op_base.cc (4587行, 11次缺陷修复)

主要职责: HCCL通信域的初始化/销毁及各类集合通信算子的单算子模式入口实现。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 并发安全 | g_oneSidedCommHcomInfos和g_oneSidedCommSet全局变量无锁访问（IsOneSidedComm/InitOneSidedHcomInfo/DeInitOneSidedHcomInfo等多处） | 267-334, 717, 2832 |
| 2 | 并发安全 | opGroup2CommMap的find/insert操作部分路径未持锁（CheckOpBasedHcom/HcclCreateSubCommConfigInner/InitCommRootInfo/HcclSetConfig） | 343, 900, 1353, 1829 |
| 3 | 并发安全 | HcclSendInner/HcclRecvInner/HcclAlltoAllInner/HcclAlltoAllVInner/HcclAlltoAllVCInner/HcclReduceInner缺少operatorlock_ | 2667-2794, 3247-3582 |
| 4 | 死代码 | HcclCommInitClusterInfoWrapper: return HCCL_SUCCESS后面还有return ret，永远不执行 | 582-584 |
| 5 | 状态泄漏 | HcclReduceInner/HcclAlltoAllInner/HcclAlltoAllVInner/HcclAlltoAllVCInner缺少HcclResetIfProfile()恢复调用，profiling开关泄漏 | 3287, 3409, 3527, 3634 |
| 6 | 代码重复 | InitCommRootInfo(212行)与InitCommClusterInfo(136行)的配置序列几乎完全重复，HcclCreateSubCommConfigInner亦然，三处拷贝 | 1344-1555, 395-530, 888-1025 |
| 7 | 并发安全 | GetHcclExistDeviceOpInfoCtx锁内修改thread_local的g_hcclDeviceId，后续无锁路径可能访问错误设备上下文 | 116-117 |
| 8 | 资源泄漏 | HcclOneSidedCommDestroy: opGroup2CommMap.find失败时提前return，isUsed标记永远不被清除，槽位无法复用 | 2843 |
| 9 | 悬垂引用 | HcclGetCommAll: 线程创建失败时已启动的线程未join，这些线程持有comms[]的引用，主线程return后引用失效 | 188-193 |
| 10 | 并发安全 | HcclCommDestroy和HcclCommDestroyWrapper对IsOneSidedComm的锁保护策略不一致（前者持锁后者无锁） | 3010-3014 vs 2905-2907 |

### 2.5 adapter_rts.cc (2487行, 7次缺陷修复)

主要职责: 封装ACL/Runtime底层API，为HCCL提供跨平台适配层。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 死代码(特性失效) | REPLACE_NOTIFY_WITH_EVENT宏中result硬编码为0，if(result!=0)永远不执行，event替换功能完全失效 | 37-45 |
| 2 | 并发安全 | g_localDeviceType/g_localDeviceLogicId/g_deviceSatMode等普通全局变量多线程无锁读写 | 94-96, 207, 449, 458 |
| 3 | 资源管理 | hrtDevMemAlignWithPage: hrtGetPointAttr失败且hrtFreeHost也失败时ptrAttr泄漏 | 1148-1196 |
| 4 | 函数复杂度 | hrtMalloc: 多层嵌套的设备类型判断+level2Address分支，76行，设备类型分支组合易遗漏 | 558-634 |

### 2.6 hccl_communicator.cc (3110行, 5次缺陷修复)

主要职责: HCCL通信域核心实现，包含初始化、通信操作执行、NS恢复(Suspend/StopExec/Clean)等全生命周期管理。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 并发安全(竞态) | DestroyOpTransportResponse: 持锁erase后释放锁，在无锁状态下逐个DeInit()，其他线程可能同时操作同一transport对象 | 634-678 |
| 2 | 初始化顺序缺陷 | InitOpResPara: memset_s清零后使用opResDeviceParaPtr_初始化链表，但该指针在下面CreateWorkSpace才赋值 | 123-139 |
| 3 | 代码重复 | Suspend/StopExec/Clean三个函数(约190行)包含几乎完全相同的while(true)轮询循环和超时处理 | 1166-1353 |
| 4 | 死代码 | DestroyAicpuComm: while(true)循环后的return HCCL_SUCCESS不可达 | 881 |
| 5 | 并发安全 | isSuspending成员变量在多个方法中设置为true，无atomic声明或锁保护 | 1176, 1236, 1299 |

### 2.7 hccl_independent_rank_graph.cc (490行, 5次缺陷修复)

主要职责: RankGraph查询的公共API入口，支持V1/V2双版本分发。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 错误处理 | HcclGetTopoInstsByLayer/HcclGetTopoType/HcclGetRanksByTopoInst: HCCLV2_FUNC_RUN后直接return HCCL_SUCCESS，V1路径永远不执行 | 251-323 |
| 2 | 日志格式错误 | 日志字符串"netLayer%u]"缺少左方括号 | 70, 80 |
| 3 | 状态一致性 | 每次调用都getenv("HCCL_INDEPENDENT_OP")而非缓存，运行时修改环境变量导致行为不一致 | 60, 102等11处 |

### 2.8 coll_alg_operator.cc (1081行, 5次缺陷修复)

主要职责: 集合通信算法算子基类，负责算法选择、executor管理、资源请求计算和执行编排。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 死代码 | CalNumBlocks: else分支后有永远不可达的return HCCL_SUCCESS | 97-100 |
| 2 | 状态管理 | executor_惰性初始化+SelectAlgFor91093WithCoreLimit中置nullptr重建，模式脆弱 | 67, 78, 140, 215 |
| 3 | 函数复杂度 | GetDefaultAlgoLevel1V2: 多个跨4行的复合布尔条件，可读性差 | 471-526 |

### 2.9 alltoall_operator.cc (712行, 4次缺陷修复)

主要职责: AlltoAll系列集合通信算子的算法选择、参数预处理和执行编排。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 空指针 | SetExecutorAttr/SetExcutorExtraInfo/CheckNeedRecreateComm: dynamic_cast结果未检查nullptr | 454, 458, 491 |
| 2 | 条件方向错误 | IsSatisfyAlltoAllAivCondition: CHK_PRT_RET(userRankSize_ > 1, ...)的条件方向与上下文逻辑矛盾 | 549 |
| 3 | 陈旧数据 | allMeshAggregationSendRecvInfo_在SelectAlg提前return时不会更新，后续SetExcutorExtraInfo使用上次遗留数据 | 243-349 |

### 2.10 hccl_independent_op_mem.cc (142行, 4次缺陷修复)

主要职责: 独立算子模式下的内存注册/注销和CCL Buffer获取。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 错误吞没 | HcclCommDeregMem: HCCL_E_NOT_FOUND时返回HCCL_SUCCESS，掩盖内存泄漏调试线索 | 90-91 |
| 2 | 控制流依赖宏 | HcclGetHcclBuffer: V1/V2路径选择依赖HCCLV2_FUNC_RUN宏内部实现，可读性差 | 105-142 |

### 2.11 hccl_dispatcher_ctx.cc (231行, 4次缺陷修复)

主要职责: DispatcherCtx生命周期管理，维护commId与DispatcherCtx的全局映射。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 资源泄漏 | AcquireDispatcherCtx: 创建新DispatcherCtx但不注册到g_ctx全局映射，无法被其他线程找到或按commId清理 | 205-231 |
| 2 | 并发安全(TOCTOU) | DestroyDispatcherCtx: deleteMutex_和g_mtx的嵌套获取+保护粒度不一致 | 127-161 |
| 3 | 悬垂指针 | DestroyDispatcherCtx: ctx=nullptr只修改局部参数副本，调用者持有的指针不会被置空 | 159 |

### 2.12 pkg_inc/hccl/hcom.h (387行, 6次缺陷修复)

主要职责: HCOM公共API头文件，定义集合通信对外接口和数据结构。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | ABI兼容性 | extern "C"块内使用namespace(hccl::HcclDumpInfo)，C编译器无法处理 | 28-39 |
| 2 | ABI兼容性 | 公共API函数签名包含C++默认参数，在C linkage中无意义且影响ABI稳定性 | 261, 275, 284, 317 |
| 3 | 符号膨胀 | 头文件中定义非inline的const std::string/std::map全局变量，每个翻译单元独立副本+静态初始化顺序问题 | 112-141 |
| 4 | 内部实现泄露 | 直接include了workflow.h/dtype_common.h/hccl_rank_graph.h等内部头文件 | 20-22 |
| 5 | 类型安全 | HcclRtStream和rtStream_t均为void*，编译器无法区分误用 | 143-144 |

### 2.13 src/CMakeLists.txt (697行, 6次缺陷修复)

主要职责: 顶层构建配置。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 路径硬编码 | HCCL_THIRD_PARTY_DIR使用7层../回溯 | 31 |
| 2 | 代码重复 | ccl_kernel_plf和ccl_kernel_plf_a的include/compile/link配置几乎完全重复 | 306-376, 493-573 |
| 3 | 编译选项冲突 | ccl_kernel_plf_a同时指定-O3和-O2，后者覆盖前者 | 515 |
| 4 | 安全选项不一致 | hccd用-fstack-protector-strong，ccl_kernel_plf/ccl_kernel_plf_a用-fstack-protector-all | 243, 502, 517 |
| 5 | 依赖管理 | GLOB_RECURSE动态收集源文件目录作为include路径 | 624-629 |

### 2.14 src/framework/CMakeLists.txt (997行, 5次缺陷修复)

主要职责: framework层构建配置。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 路径硬编码 | 8层../回溯，且与顶层CMakeLists同名变量值不同 | 16 |
| 2 | 代码重复 | ccl_kernel的include路径在BUILD_OPEN_PROJECT分支内外被添加两次 | 689-706, 835-846 |
| 3 | 全局副作用 | CMAKE_CXX_FLAGS在BUILD_OPEN_PROJECT分支中被清空，影响后续所有target | 927 |
| 4 | 硬编码构建产物路径 | open分支用硬编码.a路径引用ccl_kernel_plf | 943 |
| 5 | 条件分支不对称 | KERNEL_MODE=OFF时ccl_kernel target不存在但后续分支仍对其配置 | 267, 785 |

### 2.15 hccl_comm_pub.h (454行, 3次缺陷修复)

主要职责: hcclComm类定义，集合通信域的核心内部接口。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 封装泄露 | planner/barrierSendBuf/barrierRecvBuf/operatorlock_作为public成员暴露 | 338-341 |
| 2 | 条件编译导致内存布局变化 | 不同编译配置下hcclComm类的成员集合不同(CCL_KERNEL_AICPU/HCCD宏)，跨模块传递指针可能越界 | 366-375, 441-448 |
| 3 | 类职责过重 | hcclComm类承载约80个public方法，单类承担初始化/通信/资源/group/拓扑/profiling等全部功能 | 全文件 |

### 2.16 rs.c (2558行, 3次缺陷修复)

主要职责: RDMA服务核心实现，管理RDMA设备初始化、QP创建、连接管理。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | UAF | RsInit错误路径free(rscb)后gRsCb(thread-local)仍指向已释放内存 | 554-622 |
| 2 | 条件编译爆炸 | CONFIG_TLV/CONFIG_CONTEXT/CUSTOM_INTERFACE三层宏组合，每个变体初始化/反初始化路径不同 | 38-521 |
| 3 | 不可靠退出 | RsDeinitRscbCfg用usleep+tryAgain忙等待线程退出，超时后仅warning继续执行 | 529-549 |

### 2.17 dispatcher_aicpu.cc (1265行, 3次缺陷修复)

主要职责: AICPU任务分发器，负责SQE编排、下发和RTSQ管理。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 函数复杂度 | LaunchTask(216行)和LaunchNewTask(145行)包含多层嵌套的ring buffer索引计算 | 354-718 |
| 2 | 活锁风险 | LaunchTask忙等待RTSQ空间时遍历其他stream调LaunchTask(false)，互相等待可能活锁 | 529-567 |
| 3 | 代码重复 | LaunchTask和WaitRtsq+MemcpyRtsq的RTSQ等待/超时/拷贝逻辑大量重复 | 502-718 vs 902-1145 |
| 4 | 错误码误用 | 多处用HCCL_E_PTR表示"超出范围"错误，应为HCCL_E_PARA | 516, 913, 984 |

### 2.18 hcom_common.cc (1613行, 3次缺陷修复)

主要职责: HCOM通用功能实现，包括初始化、group管理、集合通信操作的入口封装。

| # | 风险类别 | 描述 | 行号 |
|---|---------|------|------|
| 1 | 并发安全(TOCTOU) | HcomGetCurHcomCtx: 加锁查找+返回引用后锁即释放，引用在锁外使用但内容可能被并发修改 | 119-148 |
| 2 | 无限阻塞 | HcomDestroyGroupImpl: while(ref!=0)忙等待轮询group引用计数，无超时保护 | 533-538 |
| 3 | 全局状态过多 | 文件中定义了10+个全局/static变量，锁获取顺序无明确约定 | 74-117 |
| 4 | 日志错误 | HcomCreateCommCCLbuffer: 错误报告中函数名写成了"HcomGetDevType" | 1468-1481 |

---

## 三、跨文件系统性风险总结

### 3.1 并发安全是最大的系统性风险

在审查的18个热点文件中，15个存在并发安全问题。模式包括：

- TOCTOU竞态: hcom.cc(HcomCreateGroupImplHeterog), hcom_common.cc(HcomGetCurHcomCtx), hccl_dispatcher_ctx.cc(DestroyDispatcherCtx)
- 无锁访问共享状态: op_base.cc(g_oneSidedCommHcomInfos), aicpu_communicator.cc(errMessageReport_), adapter_rts.cc(g_localDeviceType), hcom.cc(g_rankTableSetInfo)
- 锁范围不足/不一致: op_base.cc(operatorlock_缺失), hccl_communicator_host.cc(Init锁范围), hccl_communicator.cc(DestroyOpTransportResponse)
- 忙等待无超时/无yield: hccl_communicator_host.cc(SnapshotCheck), hcom_common.cc(HcomDestroyGroupImpl), rs.c(RsDeinitRscbCfg)

根因: 通信框架天然是多线程环境（多device、多通信域、主线程+背景轮询线程），但锁的使用缺乏统一的层级设计和文档化约定。

### 3.2 错误处理链断裂

- 返回值覆盖: hccl_communicator_host.cc中stream分配返回值被后续调用覆盖
- 错误码类型冲突: aicpu_communicator.cc中HcclResult被隐式转bool，语义反转
- 错误路径资源泄漏: hcom.cc(malloc后memcpy_s失败未free), hccl_communicator_host.cc(hrtMalloc失败路径), op_base.cc(isUsed标记未清除)
- 死代码掩盖逻辑错误: op_base.cc(return后的return), hccl_communicator.cc(while后的return)

### 3.3 God Class与代码重复

- hcclComm(80+方法)、hccl_communicator_host.cc(8818行)、aicpu_communicator.cc(5809行)、op_base.cc(4587行)、hcom.cc(4068行) 均为超大文件
- 三处InitCommConfig序列重复(op_base.cc)，两处RTSQ下发逻辑重复(dispatcher_aicpu.cc)，两处CMake target配置重复
- 代码重复导致"改一漏一"是hcomm-dev缺陷的高频来源

### 3.4 公共API/ABI设计缺陷

- hcom.h混用C/C++ linkage、暴露内部头、在头文件定义std容器全局变量
- hccl_comm_pub.h暴露public成员变量、条件编译改变类内存布局
- 多个API返回裸指针(悬垂指针: hcom.cc两处, 资源所有权不清: hcom.cc HcomGetSplitStrategy)

### 3.5 构建系统脆弱性

- 7-8层相对路径回溯
- GLOB_RECURSE动态收集include路径
- 编译选项/安全选项在不同target间不一致
- BUILD_OPEN_PROJECT/KERNEL_MODE条件分支不完整

---

## 四、缺陷热点分布可视化

```
src/framework/                          ████████████████████████████████████████ 143次
  communicator/                         ██████████████████████████ 53次
    impl/hccl_communicator_host.cc      ███████ 14次  ← 最高频
    impl/hccl_communicator.cc           ██ 5次
    impl/independent_op/                █████ 11次
  hcom/                                 ██████████ 20次
    hcom.cc                             ███████ 14次  ← 并列最高频
    hcom_common.cc                      █ 3次
  device/framework/                     ██████████ 23次
    aicpu_communicator.cc               ██████ 12次
  op_base/src/                          ███████ 14次
    op_base.cc                          █████ 11次

src/platform/                           ██████████████████████████ 92次
  hccp/                                 █████████ 31次(分散)
  common/adapter/                       ███ 7次
  comm_primitive/                       ██ 4次
  task/                                 ██ 3次

src/algorithm/                          ██████████ 36次
  impl/operator/                        █████████ 9次

cmake + CMakeLists                      █████ 13+6+5=24次
pkg_inc/hccl/                           ██ 8次
```
