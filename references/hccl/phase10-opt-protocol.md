# Phase 10.3: OPTIMIZATION + PROTOCOL 类案例深度分析

日期: 2026-03-16
分析范围: 3个OPTIMIZATION + 6个PROTOCOL landmark cases
分析方法: git show --stat → 关键文件diff精读 → git log -5追踪后续 → 6维度框架

---

## Tier 1: PL-6 — host&device sync (唯一3维度landmark: A+B+D)

### Case PL-6: Host-Device双向同步机制从ACL Notify迁移到Thread Export

[Commit]: hccl:89a09db2 + hcomm:d813063d (host&device sync)
[类别]: PROTOCOL
[涉及层]: hccl:ops/op_common + hcomm:framework/next + hcomm:framework/communicator
[变更规模]: hccl 4 files +84/-58, hcomm 5 files +99/-11 (合计9 files, +183/-69)
[维度]: A(跨仓) + B(Host-Device) + D(协议)
[时间]: 2026-03-04 14:45:40(hcomm) → 14:46:01(hccl)，同作者间隔21秒

[场景]:
AICPU模式下，Host算子输入流需要与Device侧AICPU主thread做双向同步:
- Host→Device: "我准备好了，你可以开始执行算法"
- Device→Host: "我执行完了，你可以继续"
旧方案使用ACL层的Notify(aclrtRecordNotify / aclrtWaitAndResetNotify)，通过thread_local全局数组`g_notifies_host_with_device[2]`管理两个控制Notify，ID通过序列化写入Device Context。

[约束]:
1. Host和Device运行在不同的执行域，只能通过Notify做信号同步
2. 旧方案直接调用ACL接口(aclrtCreateNotify/aclrtRecordNotify)，绕过了hcomm的Thread资源管理体系
3. 910_95(A5)平台需要特殊的host类型notify用于host&device同步
4. 两个仓库(hccl + hcomm)需要原子性同步修改
5. Notify ID通过AlgResourceCtxSerializable序列化到Device Context中，序列化格式需要同步调整

[决策]:
将Host-Device同步机制从"裸ACL Notify"迁移到hcomm的Thread Export体系:

hccl侧(op_common.cc)核心变更:
1. 删除`g_notifies_host_with_device`全局数组和`HcclGetH2DNotify()`函数(约30行)
2. 删除`notifyIds[AICPU_CONTROL_NOTIFY_NUM]`序列化字段(alg_param.h)
3. 新增`SaveMainThreadInfo()`/`GetMainThreadInfo()`，将主流Thread信息保存到Engine Context中
4. 同步流程改为: `HcclThreadAcquireWithStream()` → `HcclThreadExportToCommEngine()` → `HcommThreadNotifyRecordOnThread()` / `HcommThreadNotifyWaitOnThread()`
5. 通过`param.opThread`将导出的Thread句柄传递给Device侧kernel

hcomm侧核心变更:
1. `hccl_independent_op_engine.cc`: `HcclThreadExportToCommEngine()`新增V1/V2双路径分派(IsCommunicatorV2()判断)，增加threadNum上限校验(MAX_EXPORT_THREAD_NUM=40)
2. `aicpu_ts_thread.cc`: A5平台在HostInit()中多申请一个host类型notify(`NotifyLoadType::HOST_NOTIFY`)，专用于host&device同步

Device侧(kernel_launch.cc)变更:
- 从`resCtx.notifyIds[0]`改为`HcommThreadNotifyWaitOnThread(thread, notifyNumOnMainThread, timeout)`
- 从`resCtx.notifyIds[1]`改为`HcommThreadNotifyRecordOnThread(thread, exportedAicpuTsThread, 0)`

[替代方案]:
1. 保持ACL裸Notify方案，只修补序列化 — 没选，因为绕过Thread管理体系导致资源无法统一追踪
2. 在hcomm platform层扩展comm_primitive接口 — 没选，因为这是框架层(Thread管理)的职责，不应下沉到platform层
3. 只修改hccl侧，hcomm不动 — 不可能，因为Thread Export是hcomm提供的能力

[后果]:
- 后续4个commit修改了aicpu_ts_thread.cc: `[Fix] Rename 950` → `channel复用` → `profiling_and_taskexception`，均未回滚同步机制本身
- op_common.cc后续5个commit(`fix CleanCode` → `ccu算法回退` → `channel protocol` → `upload dpu op`)未修改同步路径
- 未被revert，说明设计稳定

[可迁移经验]:
跨Host-Device边界的同步机制必须纳入框架的资源管理体系(Thread/Notify/Stream)，而非直接调用底层ACL接口。裸ACL调用虽然更快实现，但会导致资源泄漏风险和无法被框架统一追踪。跨仓原子性变更(同作者21秒内提交两仓)是保障双边协议一致性的必要手段。

---

## Tier 2: PL-1/PL-2 — HB check三阶段演进链

### Case PL-2: 心跳算子不一致检测加开关重做

[Commit]: hcomm-dev:ebfcde5c (HB check op insistent with switch)
[类别]: PROTOCOL
[涉及层]: hcomm:framework/cluster_maintenance + hcomm:framework/common + hcomm:framework/communicator + hcomm:framework/op_base
[变更规模]: 10 files, +499/-158
[维度]: D(协议)
[时间]: 2026-01-06 (dev仓)

[场景]:
心跳(Heartbeat)模块需要检测集群中不同节点是否在执行相同的集合通信算子(op inconsistent detection)。
三阶段演进:
- 阶段1(init code): 随初始化代码引入HB check op inconsistent功能，默认开启
- 阶段2(RV-7 a37e6cf1, 2025-12-03): 整体Revert，删除173行(7 files, +141/-249)
- 阶段3(PL-2 ebfcde5c, 2026-01-06): 用环境变量开关重新引入，默认关闭

[约束]:
1. 心跳帧(HeartBeatFrame)大小敏感——50ms广播周期内帧越大越影响网络
2. 原版将算子信息嵌入心跳帧(OpInfoTagQueueFrame含10个tag队列，每个队列500条opInfo)，帧膨胀严重
3. Revert原因推测: 功能无法关闭，一旦出问题只能回退整个commit
4. 不同节点的算子下发顺序/时序可能天然不一致(多线程环境)，误报风险高

[决策]:
PL-2相比被revert的原版，做了三个关键改进:

1. 环境变量开关(核心保护措施):
   - `EnvConfig.inconsistentCheckSwitch`默认为`false`
   - 通过`HCCL_DFS_CONFIG`环境变量中的`inconsistent_check:on/off`控制
   - `GetSendOpInfoList()`入口处直接`if (!GetExternalInconsistentCheckSwitch()) return;`短路
   - 心跳socket buffer大小根据开关动态选择: `sizeof(HeartBeatFrameWithOpCheck) : sizeof(HeartBeatFrame)`

2. 恢复tag级队列设计(与revert后的简化版不同):
   - Revert版: `OpInfoDesc`内嵌`identifier`字段，`opInfoQueue_`直接存`OpInfoDesc`
   - 重做版: 恢复`OpInfoTagQueue`+`OpInfoTagQueueFrame`结构，每个tag独立队列
   - 恢复`opInfoQueueForSend_`缓冲队列(跨轮次残留数据保存)
   - 队列上限从65536提升到131072

3. 新增错误上报路径:
   - `HcclGetCommAsyncError()`中新增`CommCheckOpInconsistentError()`调用
   - 仅在CQE检查通过(HCCL_SUCCESS)后才检查算子不一致
   - 新增`inconsistentOpMap_`记录不一致状态，支持按identifier查询

[替代方案]:
1. 不加开关，直接重新合入 — 没选，上次就是因为无法关闭而被revert
2. 只保留日志告警，不做错误上报 — 没选，失去了实际的故障检测价值
3. 缩小帧中的opInfo数量来控制帧大小 — revert版做了这个(OPINFO_SEND_NUM=16)，但重做版恢复了tag级细粒度设计

[后果]:
- PL-1(hcomm:93222924, 2026-01-12)是main仓最终版本，与PL-2 diff几乎完全相同(env_config部分diff逐字相同)
- PL-1比PL-2晚6天合入main，证实了"dev先验证→main后合入"的标准流程
- 后续commit(sc clean → 子包适配 → resolved known issues → adaptor高版本gcc → Err Code)均未修改核心逻辑
- 功能未被再次revert，说明"默认关闭+开关控制"策略有效

### Case PL-1: 心跳算子不一致检测main仓最终版

[Commit]: hcomm:93222924 (HB checkout op insistent)
[类别]: PROTOCOL
[涉及层]: 同PL-2
[变更规模]: 10 files, +499/-158 (与PL-2几乎相同)
[维度]: D(协议)
[时间]: 2026-01-12 (main仓)

PL-1与PL-2的diff对比:
- env_config.cc/env_config.h: 完全相同
- heartbeat.cc/heartbeat.h: 核心逻辑相同(tag级队列+开关控制+不一致记录)
- op_base.cc: 相同(新增CommCheckOpInconsistentError调用)
- 差异极小，仅可能是代码格式/注释微调

这确认了"dev验证→main合入"是hcomm的标准发布流程。

[可迁移经验]:
涉及通信协议变更的功能必须提供运行时开关(默认关闭)。不可关闭的协议功能在出问题时只能整体Revert，代价极高。"introduce→revert→add-switch-redo"三阶段模式是hcomm中反复出现的模式(Phase 8.9也在retry默认关闭中观察到c303e62a)，本质是: 通信框架的新协议功能需要渐进式上线，因为集群环境中的边界条件极难在开发阶段覆盖。

---

## Tier 2: PL-3 — scatter notify&wait (Notify协议扩展)

### Case PL-3: Thread跨通信引擎导出支持scatter的Notify同步

[Commit]: hcomm-dev:0e82e543 (scatter notify&wait)
[类别]: PROTOCOL
[涉及层]: hcomm:framework/communicator + hcomm:framework/next/comms + hcomm:platform/resource + include/hccl
[变更规模]: 14 files, +276/-25
[维度]: D(协议)
[时间]: 2026-01-29

[场景]:
scatter算子需要在Host侧(CPU_TS引擎)和Device侧(AICPU_TS引擎)之间做Thread级Notify同步。
现有的Thread资源管理只支持同引擎内的Thread操作，不支持跨引擎导出(Export)。
scatter是hccl中最活跃的算子(Phase 1: 20次修改)，正在经历从旧架构到新架构的迁移。

[约束]:
1. Thread句柄是engine-specific的，CPU_TS的ThreadHandle不能直接在AICPU_TS上使用
2. CpuTsThread原来不支持GetUniqueId()(直接return空字符串+ERROR日志)
3. AICPU侧的Thread展开需要通过AicpuLaunchMgr::ThreadKernelLaunch()
4. 导出操作需要双向映射: CPU↔AICPU 句柄互转

[决策]:
实现Thread跨通信引擎导出机制(3个核心变更):

1. 新增公共API `HcclThreadExportToCommEngine()` (hccl_res.h):
   - 将Thread资源从一个引擎导出到另一个引擎
   - 导出后在目标引擎直接引用，无需导入操作

2. ThreadMgr中实现双向导出(thread_manager.cc +120行):
   - `ThreadExportToCommEngineCpu()`: AICPU→CPU方向，通过`threadHandleOthersToCpu_`映射表查找
   - `ThreadExportToCommEngineAicpu()`: CPU→AICPU方向，需要调用`AicpuLaunchMgr::ThreadKernelLaunch()`做设备侧展开
   - `HcclThreadExportToCommEngine()`统一入口，按dstCommEngine分派(CPU_TS/CPU → Cpu路径, AICPU/AICPU_TS → Aicpu路径, AIV/CCU → 报错)
   - 新增`threadHandleOthersToCpu_`映射表和`threadMapMutex_`保护并发访问

3. CpuTsThread完整实现GetUniqueId()(cpu_ts_thread.cc +60行):
   - 从"不支持"改为完整实现: 序列化stream参数+notify信息+SqCqeContext
   - 临时申请device stream用于资源展开
   - SetIpc()调用从310P限定改为所有平台(去掉`Is310PDevice()`判断)

4. HcclThreadAcquireWithStream改为shared_ptr管理(thread_manager.cc):
   - unique_ptr → shared_ptr，允许多处持有引用
   - 新增流级缓存: `mainThread_[stream]`防止重复创建

[替代方案]:
1. 不做跨引擎导出，scatter在单引擎内完成所有同步 — 不可能，scatter天然需要Host-Device协同
2. 在platform层的comm_primitive中增加跨引擎Notify — 太底层，Thread级导出是更自然的抽象
3. 使用PL-6的旧方案(裸ACL Notify) — 已被PL-6替换掉，方向不对

[后果]:
- PL-6(host&device sync, 2026-03-04)在PL-3(2026-01-29)基础上进一步完善了Thread Export机制
- hccl在PL-3之后4天适配了scatter编排(d819cfc2)
- 后续commit未回滚，Thread Export成为scatter同步的标准方案

[可迁移经验]:
跨通信引擎的资源共享应在Thread管理层做抽象(Export/Import)，而非在每个使用点手动做句柄转换。双向映射表(threadHandleOthersToCpu_)加互斥锁是处理跨引擎句柄映射的标准模式。新增公共API(hccl_res.h)意味着这是对外暴露的接口变更，需要考虑向后兼容。

---

## Tier 3: PL-4 — topo detect retry

### Case PL-4: 拓扑发现建链增加重试机制

[Commit]: hcomm-dev:9ceead3b (topo detect retry)
[类别]: PROTOCOL
[涉及层]: hcomm:framework/common/topo + hcomm:platform
[变更规模]: 11 files, +83/-25
[维度]: D(协议)
[时间]: 2025-12-22

[场景]:
拓扑发现(TopoInfoExchange)过程中，Agent端连接Server后需要等待Server的确认消息。
在大规模集群中，Server端Accept所有client后才开始数据交换，如果某些节点网络抖动导致连接延迟，Agent端可能在超时前一直等待而非重试。
原始实现: `Connect()`一次连接，成功则等待Server Recv，超时则失败。

[约束]:
1. 拓扑发现是初始化阶段的关键路径，失败意味着整个集合通信无法启动
2. 总超时时间由`GetExternalInputHcclLinkTimeOut()`控制，不能增加
3. 重试时不应产生ERROR级日志(会误触告警系统)

[决策]:
在Agent端实现`ConnectWithRetry()`(最多3次重试):

1. 将总超时时间三等分: `timeout = GetExternalInputHcclLinkTimeOut() / AGENT_MAX_RETRY_TIME`
2. 每次重试: 重新创建socket → Init → Connect → GetConnection → TryRecvFromServer
3. 前2次重试失败时: 通过`SetErrToWarnSwitch(true)`将ERROR降级为WARNING，避免误触告警
4. 最后一次失败: 恢复ERROR级别，正常报错

Server端配合变更:
- Accept成功后立即Send确认消息(`TOPO_EXCHANGE_CHECK_MESSAGE`)
- 这样Agent可以快速判断连接是否真正建立

附带修正: `DisplayConnectionedRank` → `DisplayConnectedRank`(拼写修正)

[替代方案]:
1. 只增加总超时时间 — 没选，延长超时只是推迟失败，不解决网络抖动
2. 在Server端增加超时重试 — 没选，Server是被动Accept端，主动权在Agent
3. TCP层面的keepalive/重连 — 太底层，拓扑发现有自己的应用层协议

[后果]:
- 后续只有`fix spell err`修改了这两个文件(纯格式)
- 未被revert，重试机制稳定

[可迁移经验]:
分布式建链阶段的重试必须有两个保护: (1) 总超时不变(从中切分)，防止重试无限延长; (2) 非末次重试降低日志级别(SetErrToWarnSwitch)，防止中间态重试污染告警系统。Server和Agent的双边协议变更(Server加Send确认 + Agent加Recv验证)必须同时修改。

---

## Tier 3: PL-5 — fix changelink for OpRetry WaitResume

### Case PL-5: 简化重执行WaitResume的链路切换判断

[Commit]: hcomm-dev:28f18ab5 (fix changelink for OpRetry WaitResume)
[类别]: PROTOCOL (重执行协议修复)
[涉及层]: hcomm:framework/cluster_maintenance/recovery + hcomm:framework/communicator + hcomm:platform/task
[变更规模]: 9 files, +7/-32 (净减25行)
[维度]: D(协议) + E(并发)
[时间]: 2025-12-08

[场景]:
算子重执行(OpRetry)的WaitResume阶段需要决定是否做链路切换(changeLink)。
原设计中，链路切换只在RDMA错误时触发(`g_isRdmaError`全局变量控制)。
但实际场景中，非RDMA错误(如SDMA错误、约束违反)也可能需要链路切换来恢复。

[约束]:
1. `g_isRdmaError`是全局变量，存在多线程竞态风险(Phase 1已知并发控制薄弱)
2. WaitResume状态机有两条路径: `isRdmaError=true`走changelink，`isRdmaError=false`直接resume running
3. `isChangeLinkMap`的判断条件(`find != end && second == true`)过于严格

[决策]:
三处核心简化:

1. 删除`g_isRdmaError`全局变量(opretry_base.h + hccl_communicator.cc):
   - `extern bool g_isRdmaError;` 和 `bool g_isRdmaError = false;` 全部删除
   - 消除了全局状态带来的并发风险

2. OpRetryServerWaitResume状态机合并(opretry_server.cc):
   - 原来两条路径: `!isServerStateWaitResume_ && !isRdmaError`(直接resume) vs `!isServerStateWaitResume_ && isRdmaError`(走changelink)
   - 合并为单路径: `!isServerStateWaitResume_`统一走changelink路径
   - 删除约12行分支代码

3. ExitWaitResumeState中`isChangedLink`无条件设为true(opretry_manager.cc):
   - 原来: `if (retryCtx->isRdmaError) isChangedLink = true;`
   - 现在: `isChangedLink = true;`(无条件)

4. hccl_communicator.cc中放宽changeLink判断:
   - 原来: `isChangeLinkMap.find(rank) != end && isChangeLinkMap[rank] == true`(要求值为true)
   - 现在: `isChangeLinkMap.find(rank) != end`(只要在map中就认为需要changeLink)

5. 新增TS_ERROR_RETRY_CONSTRAINT错误码透传(aicpu_hccl_sqcq.cc):
   - `SwitchSdmaCqeErrCodeToTsErrCode()`新增`case TS_ERROR_RETRY_CONSTRAINT`

6. 删除task_abort_handler.cc中的`g_isRdmaError = false`重置(两处):
   - PRE和POST阶段都不再重置全局变量(因为变量已删除)

[替代方案]:
1. 保留g_isRdmaError但加锁保护 — 没选，本质问题不是并发而是逻辑: 非RDMA错误也需要changeLink
2. 按错误类型细分changeLink策略 — 没选，简化为"总是changeLink"更安全(宁可多切换不可漏切换)

[后果]:
- 后续2个commit: `fix step recalculation` → `fix OpRetry timeout`，都是OpRetry相关的后续修复
- 这说明重执行模块处于"密集调试稳定期"(Phase 8.9已知: 2025-12月下旬4天5个commit)
- 未被revert

[可迁移经验]:
用全局变量在多线程环境中传递状态机决策是反模式。重执行路径的错误分类(RDMA/SDMA/约束违反)不应影响恢复策略的骨干逻辑——"宁可多做恢复动作(changeLink)也不要漏做"是更安全的默认策略。净减25行的修复同时解决了正确性和并发安全两个问题。

---

## Tier 3: OPT-1 — groupSendRecv small data optimization

### Case OPT-1: GroupSendRecv小数据批量编排优化

[Commit]: hcomm:fd9d488c (groupSendRecv small data optimization)
[类别]: OPTIMIZATION
[涉及层]: hcomm:algorithm/coll_executor + hcomm:framework/communicator + hcomm:framework/group + hcomm:framework/op_base
[变更规模]: 28 files, +935/-711
[维度]: 无特殊维度标注

[场景]:
Group特性允许用户批量调用HcclSend/HcclRecv，但原实现是逐个执行Send/Recv算子。
对于小数据量场景(每条<128KB)，逐个执行的编排开销(每次都要经过communicator层调度)远大于数据传输本身。
MR描述明确指出: "本次改动较大，改变了算子执行的逻辑"。

[约束]:
1. 需要兼容大数据(>128KB)和小数据两种场景
2. 需要兼容A2单机/双机、A3单机/双机
3. 需要支持增量建链(逐步扩大通信规模)
4. 新执行器必须继承现有的CollBatchSendRecvExecutor框架

[决策]:
新增`CollBatchSendRecvGroupExecutor`，继承`CollBatchSendRecvExecutor`:

1. 数据量分类(128KB阈值):
   - `isGroupBigCount()`: 只要有一个item > BIG_DATA(128KB)就走big路径
   - Big路径: 按stream分组(`OrganizeSendItemByStream`/`OrganizeRecvItemByStream`)，每个remoteRank通过`%streamNum`分配到固定stream
   - Small路径: 不按stream分组，主流直接编排(`ProcessDataSliceSmall`)

2. 新编排入口:
   - `HcclBatchSendRecvGroup()` → `CollBatchSendRecvGroupExecutor::Orchestrate()`
   - 替代原来的逐个`HcclSend()`/`HcclRecv()`调用
   - 从framework/group层大幅删除逐个执行的代码(hccl_group.cc -243行, hccl_communicator_host.cc -230行)

3. 流水线同步模型:
   - Big: MainPostSubWait → 从流并行Send/Recv → MainWaitSubPost
   - Small: MainPostSubWaitSmall → 单从流Send+Recv → MainWaitSubPostSmall
   - 使用LocalNotify::Post/Wait做主从流同步

4. Slice切分:
   - `CalcBufferSliceSize()`: `cclInputMem.size() / alignSize / GROUP_MAX_CONCURRENT * alignSize`
   - SendRecvSlice结构体: `{addr, size, remoteRank}`

[替代方案]:
1. 只在framework层做batching，不改executor — 没选，编排开销在executor层面
2. 复用CollBatchSendRecvExecutor不做继承 — 没选，大小数据的编排逻辑差异太大
3. 单流串行编排所有小数据 — 没选，多rank情况下需要多流并发

[后果]:
- 后续2个commit: `const 修饰参数` → `clean code`，都是代码质量改进，未修改核心逻辑
- 28个文件的大规模变更没有被revert，说明测试覆盖充分(MR描述: A2/A3单机双机+增量建链)

[可迁移经验]:
小数据优化的核心手段是"批量编排替代逐个执行"——将N次单独的Send/Recv合并为1次批量executor调用。阈值(128KB)的选择需要在编排开销和数据传输时间之间权衡。大规模变更(28文件,+935/-711)需要充分的多平台测试(A2/A3)和多场景验证(增量建链)。

---

## Tier 3: OPT-3 — improve aicpu performance

### Case OPT-3: AICPU执行路径微优化(分支预测提示+日志精简)

[Commit]: hcomm:a98eb47a (improve aicpu performance)
[类别]: OPTIMIZATION
[涉及层]: hcomm:legacy/common + hcomm:legacy/framework + hcomm:legacy/unified_platform
[变更规模]: 15 files, +129/-130 (几乎纯替换)
[维度]: B(Host-Device)

[场景]:
AICPU执行路径(legacy层)的指令解释(ins_to_sqe_rule_v82.cc)和RTSQ任务下发(rtsq_a5.cc)存在大量错误检查分支。
这些分支在正常执行时几乎不会命中(空指针/零长度/溢出等防御性检查)，但编译器默认按50%概率预测，导致热路径的指令缓存效率下降。

[约束]:
1. 不能删除任何错误检查(100% CHK_PTR_NULL覆盖是强制规范)
2. 不能改变逻辑语义，只能改变编译器优化提示
3. 修改涉及15个文件，属于legacy层(bug密度最高区域)

[决策]:
两类微优化手段:

1. UNLIKELY宏标注(主要手段，~30处):
   - `if (ptr == nullptr)` → `if (UNLIKELY(ptr == nullptr))`
   - `if (size == 0)` → `if (UNLIKELY(size == 0))`
   - `if (ret != EOK)` → `if (UNLIKELY(ret != EOK))`
   - 覆盖文件: ins_to_sqe_rule_v82.cc(~20处), rtsq_a5.cc(~8处), ub_conn_lite.cc, data_type.h等

2. 分支结构优化(rtsq_a5.cc RefreshInfo):
   - 原: `if (pendingSqeCnt == perLaunchSqeCnt) { LaunchTask(); }`
   - 改: `if (pendingSqeCnt != perLaunchSqeCnt) { return; } LaunchTask();`
   - 将少数情况(达到128)的处理放在函数末尾，早退出是更常见路径

3. 日志增强(rtsq_a5.cc CopyLocBufToSq):
   - 增加sqId_、streamId_、sqHead_输出，辅助定位RTSQ满/回绕问题

4. 删除废弃代码:
   - rtsq_base.h删除6行未使用声明
   - st_lite_res.cc删除12行废弃测试代码
   - ut_aicpu_ub_conn_lite.cc删除13行废弃测试代码

[替代方案]:
1. PGO(Profile-Guided Optimization) — 编译器级别优化，但需要profiling基础设施支持
2. 将错误检查移到编译期(static_assert) — 不适用于运行时数据
3. 热路径单独提取为noinline函数 — 修改量更大，收益不一定更好

[后果]:
- 后续commit: `fix log` → `HcclEngineCtxDestroy API` → `hcommThreadJoin`
- `fix log`可能是修正本次新增日志的格式问题
- 未被revert

[可迁移经验]:
AICPU设备侧的热路径优化首选UNLIKELY宏标注错误检查分支——这是最低风险(不改语义)、最高收益(编译器分支预测)的微优化手段。注意: UNLIKELY只在热循环中有显著效果，对偶尔执行的初始化代码无意义。同时增加日志信息(sqId/streamId/head/tail)是"优化附带可观测性改进"的良好实践。

---

## Tier 3: OPT-4 — RS & AAV FastLoad

### Case OPT-4: ReduceScatter和AlltoallV的CCU快速下发

[Commit]: hcomm:8b0dc7b9 (RS & AAV FastLoad)
[类别]: OPTIMIZATION
[涉及层]: hcomm:legacy/framework/communicator + hcomm:legacy/service + hcomm:legacy/unified_platform
[变更规模]: 10 files, +215/-65
[维度]: B(Host-Device)

[场景]:
CCU(Communication Control Unit)快速下发(SuperFastLoad)机制已支持AllReduce/AllGather/Broadcast/AllToAll四种算子。
需要扩展到ReduceScatter和AlltoallV，使A5平台的这两种算子也能利用CCU快速下发减少Host侧编排延迟。

[约束]:
1. CCU快速下发需要缓存CCU参数(CachedCCUParams)，首次执行建立缓存，后续直接复用
2. AlltoallV的参数不是简单的send/recvBuf，需要特殊的args填充方式
3. ReduceScatter的2-die mesh拓扑需要额外的inline reduce操作
4. CachedCCUParams的move语义需要正确传递新增的`insType`字段

[决策]:

1. 扩展TryFastCcuLaunch的算子支持范围:
   - 原: AllReduce/AllGather/ReduceScatter/Broadcast/AllToAll
   - 新增: Reduce/Scatter/AlltoallV
   - AlltoallV使用新的ccuParamsMappingKey: `{reduceOp, sendType, 0}`(第三个字段为0)

2. 新增CcuInstType区分不同算子的CCU指令类型:
   - `CachedCCUParams`构造函数新增`CcuInstType insType`参数
   - AlltoallV检查: `params.insType != CcuInstType::CCU_ALLTOALLV_MESH_1D_DIRECT`时return false，不走快速路径

3. AlltoallV特殊的args填充(`FillAllToAllVArgs`):
   - 调用`CcuContextAllToAllVMesh1D::RefreshArgs()`获取参数向量
   - 跳过index=2(token info)，逐个填入ccuParams的args数组
   - 每RT_CCU_SQE_ARGS_LEN个参数后切换到下一个ccuParam

4. ReduceScatter 2-die特殊处理:
   - `CcuInstType::CCU_REDUCE_SCATTER_MESH_1D_2DIE`类型额外执行inline reduce
   - 使用`HrtReduceAsync()`完成scratch→dst的归约操作
   - 需要profiling和taskException的条件路径分支

5. DFX集成改进:
   - `FastCcuLaunchSaveDfxTaskInfo()`新增remoteRankId参数(默认INVALID_RANKID)
   - ReduceScatter的reduce任务使用`GetMyRank()`作为remoteRankId

[替代方案]:
1. ReduceScatter不走快速下发 — 没选，性能差距明显
2. AlltoallV使用和AllToAll相同的缓存key — 没选，V版本参数格式完全不同
3. 统一所有算子的args填充方式 — 不可能，不同算子的CCU上下文结构差异太大

[后果]:
- 后续commit: `profiling_and_taskexception` — 补充DFX相关的完善，说明快速下发的可观测性是持续改进的方向
- MR描述标注为"新特性"而非"性能优化"，但hccl_test性能测试是主要验证手段
- 未被revert

[可迁移经验]:
CCU快速下发的扩展模式是: (1) 在TryFastCcuLaunch中增加opType判断; (2) 为新算子设计ccuParamsMappingKey(缓存命中的关键); (3) 如果参数格式特殊(如AlltoallV的变长args)，新增专用的FillArgs函数; (4) 特殊拓扑(如2-die)需要额外的后处理步骤(inline reduce)。CachedCCUParams需要正确传递新增字段(insType)的move语义。

---

## 模式总结

### PROTOCOL类(6个case)的共性模式

1. 开关保护模式(PL-1/PL-2): 通信协议新功能必须提供运行时开关，默认关闭。"introduce→revert→add-switch-redo"是反复验证的安全模式。

2. 跨引擎资源导出模式(PL-3/PL-6): Host-Device同步从裸ACL接口迁移到框架的Thread Export体系，是hcomm 2026年1-3月的架构演进方向。PL-3(1月底)建立基础，PL-6(3月初)完成迁移。

3. 重试与容错模式(PL-4/PL-5):
   - PL-4: 建链阶段的重试 = 总超时切分 + 中间态降级日志
   - PL-5: 恢复阶段的简化 = 删除全局状态 + 无条件做最安全的操作(changeLink)

4. 跨仓原子性(PL-6): 同作者21秒内提交hccl+hcomm两仓，是双边协议变更的标准操作。

### OPTIMIZATION类(3个case)的共性模式

1. 编排层优化(OPT-1): 瓶颈在编排开销而非数据传输时，"批量executor替代逐个调用"是核心手段。

2. 微指令级优化(OPT-3): UNLIKELY宏 + 早退出分支重排是最低风险的热路径优化。

3. 快速下发扩展(OPT-4): CCU SuperFastLoad的算子扩展遵循固定模式(opType注册→key设计→args填充→特殊后处理)。

### 跨类别的通用教训

- 全局变量在通信框架中是反模式(PL-5删除g_isRdmaError，PL-6删除g_notifies_host_with_device)
- 大规模变更(OPT-1 28文件)需要多平台+多场景测试覆盖
- legacy层仍承载核心优化(OPT-3/OPT-4全在legacy/下)，不可忽视
