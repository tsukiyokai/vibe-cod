# Phase 10.2: BUGFIX + LIFECYCLE 类 Landmark Case 深度分析

日期: 2026-03-16
Phase: 10.2 (决策点深度分析 - BUGFIX + LIFECYCLE)

---

## BUGFIX Cases (9)

### Case BF-1: 重执行多线程下的时序问题(Error-Fix Pair)

[Error Commit]: hcomm:d17b1f3a ("supoprt aicpu cache for alltoallv")
[Fix Commit]: hcomm:75659d24 ("修复重执行多线程下的时序问题")
[类别]: BUGFIX
[涉及层]: hcomm:framework (device/framework/aicpu_communicator.cc)
[变更规模]: Error: 34 files, +3684/-725 lines; Fix: 1 file, +2/-2 lines
[维度]: D(协议) + E(并发)

[场景]:
d17b1f3a 是一个大型特性commit，将aicpu_communicator.cc中约400行的OpUnfoldCache逻辑
提取到独立的aicpu_cache_manager.cc。这是一个典型的"特性+重构"混合commit。
同日(2026-02-10)，另一开发者发现重执行(retry)路径在多线程下出现时序问题。

[约束]:
- aicpu_communicator.cc是设备侧AICPU通信器的核心文件(Phase 1: 18次BUGFIX修改)
- 重执行FSM(有限状态机)中，收到retry命令后需要: (1)重置DFX状态 (2)重置异常状态 (3)清理stream
- 这三个操作之间存在数据依赖: ResetOpRetryException()内部调用CleanStream()，
  CleanStream()会清空stream的本地buffer并更新SQ状态，然后重置CQE异常状态
- 在多线程下，如果先调用ResetOpRetryException(它会清理stream)，
  而dfxExtendInfo_的pollStatus/cqeStatus还是旧值，另一个线程可能读到不一致的状态

[决策]:
Fix commit仅做了两处语句顺序调换:

1. HcclOpExecFsmWaitRetryProcess: 将dfxExtendInfo_状态重置移到ResetOpRetryException之前
   (原: Reset -> dfxReset; 改: dfxReset -> Reset)

2. CleanStream: 将ResetStreamCqeExceptionStatus移到ClearLocalBuff+UpdateSqStatus之后
   (原: ResetCqe -> ClearBuff -> UpdateSq; 改: ClearBuff -> UpdateSq -> ResetCqe)

两处修改的共同逻辑: 先完成物理层面的清理(buffer/SQ/DFX状态)，再清除异常标记。
确保"状态一致性窗口"最小化——在异常标记被清除时，底层已经是干净的状态。

[替代方案]:
- 加互斥锁保护整个retry路径: 成本太高，设备侧AICPU对性能极敏感
- 用原子变量标记"正在retry": 增加复杂度，且本质上语句重排已足够解决问题
- 不改，认为时序窗口极小: 风险不可接受(重执行是容错关键路径)

[后果]:
后续aicpu_communicator.cc有7个commit继续修改(scatter/告警清理/A5适配等)，
但没有再出现retry时序问题。fix有效且稳定。

[可迁移经验]:
在设备侧FSM中修改状态重置顺序时，必须遵循"先清理物理资源，后清除异常标记"的原则。
大型feature commit引入的间接时序问题，定位极难(34文件中找2行)——
feature review时需特别审查retry/异常/并发路径是否被间接影响。

[教学价值 - Error-Fix对分析]:
- Error commit(d17b1f3a)本身没有直接修改CleanStream或retry路径的语句，
  但它重构了cache逻辑，使得新的cache miss/hit路径引入了更多并发访问
- 这说明"不直接修改X，但修改X的调用环境"同样可以引入X的bug
- Error commit的MR描述几乎为空(所有字段未填)，review时可能因此跳过了深入审查
- Fix commit同一天提交，说明问题在集成测试中很快暴露

---

### Case BF-2: 读写锁改为基于原子变量的实现

[Commit]: hcomm:1535b1c4 ("读写锁改为基于原子变量的实现，减少冲突")
[类别]: BUGFIX
[涉及层]: hcomm:framework (common/src/read_write_lock_base.cc/.h, device/debug/dfx/trace/executor_tracer.cc)
[变更规模]: 3 files, +100/-46 lines
[维度]: E(并发)

[场景]:
项目使用了基于std::mutex+condition_variable的读写锁实现(ReadWriteLockBase)。
在高并发DFX trace场景下(executor_tracer.cc中的HandleDestroyComm)，
mutex版读写锁产生严重的线程冲突。

[约束]:
- 通信框架天然多线程: 每个通信域有独立的executor线程
- DFX trace(executor_tracer)是后台诊断功能，不能阻塞主通信路径
- 锁的粒度需精确: 太粗阻塞太多，太细又可能遗漏保护
- 设备侧(AICPU)代码: 不能使用std::mutex(开销大)，但可用硬件原子操作

[决策]:
1. 将ReadWriteLockBase从mutex+CV实现改为纯atomic实现:
   - 使用单个atomic<uint64_t> state_编码所有状态:
     bit63=WRITING_BIT, bit62=WAITING_BIT, bit0-61=reader count
   - readLock: CAS循环递增reader count(仅在无WRITING/WAITING时)
   - writeLock: 先fetch_or(WAITING_BIT)阻断新读者，再CAS等待所有reader退出
   - CPU_PAUSE宏: x86上用pause指令，ARM上用yield
   - 新增tryReadLock: 非阻塞尝试获取读锁(用于轮询场景)
   - 新增fastReadUnlock: 简化的读锁释放(用于后台线程)

2. 同时修改executor_tracer.cc中的锁粒度:
   将"整个循环外加writeLock"改为"每次迭代内加writeLock"
   (原: writeLock -> for循环 -> writeUnlock; 改: for循环 { writeLock -> destroy -> writeUnlock })

[替代方案 — 与被Revert的c546d353对比]:
hcomm-dev中有一个被revert的并发锁commit:
- c546d353原始commit(6c3f191a): "add lock in HcclAllocComResourceByTiling for mul-thread"
  在op_base_mc2.cc的HcclAllocComResourceByTiling中加std::lock_guard<std::mutex>
- 这个commit只加了一行锁，随后被merge-revert-mr回退

c546d353(加全局锁)失败 vs 1535b1c4(原子变量锁)成功的根本区别:
- c546d353在高层API(HcclAllocComResourceByTiling)加全局mutex，锁粒度太粗，
  导致整个资源分配序列化，性能退化严重
- 1535b1c4在底层(ReadWriteLockBase)替换锁实现，保持了读写分离特性:
  多读者并发、读者不被写者饥饿(WAITING_BIT机制)

教训: 并发问题的修复不能简单地"加一把大锁"，需要分析读写模式后选择合适粒度。

[后果]:
- 2周后(6d23a648)另一开发者将read_write_lock_base从framework/common迁移到platform层，
  并应用于hdc.cc的内存操作——说明这个原子锁实现被认可并推广
- 该锁最终被用于platform/resource/mem/hdc.cc的关键路径

[可迁移经验]:
通信框架的并发控制应优先使用原子操作(CAS+spinloop)而非mutex，
特别是读多写少场景。加锁必须分析读写比例和持锁时间——
粗粒度mutex在通信框架中几乎必然导致性能退化，这是c546d353被revert的根因。

---

### Case BF-3: 修复Step快恢失败问题

[Commit]: hcomm:994390df ("修复Step快恢失败问题")
[类别]: BUGFIX
[涉及层]: hcomm:framework (recovery/opretry_manager.cc, communicator/hccl_communicator.cc, hccl_comm_host.cc)
[变更规模]: 3 files, +12/-8 lines
[维度]: D(协议)

[场景]:
Step快恢(NsRecovery)是通信框架的容错恢复机制: 当某步骤(step)失败时，
框架不需要重建整个通信域，而是恢复当前步骤继续执行。
快恢过程中出现两个问题:
1. Agent/Server的retry状态机在注册时没有正确初始化状态
2. CheckExitWaitResumeState在非AICPU环境下也被调用，导致状态检查失败

[约束]:
- 快恢是异步协议: agent和server通过KFC(Kernel Function Call)命令通信
- 状态机有多个状态(RUNNING/WAIT_RESUME/CHANGE_LINK等)，必须在每个转换点正确设置
- 非AICPU/非MC2环境(如纯CPU通信)不支持快恢，但代码路径没有区分

[决策]:
1. opretry_manager.cc: 在RegisterAgentRetryMachine和RegisterServerRetryMachine中
   添加SetRetryState调用——注册时就设为RUNNING状态，
   而非依赖后续流程隐式初始化(之前retryCtx创建后状态为未定义)

2. hccl_communicator.cc: 将CheckExitWaitResumeState调用包在条件判断中:
   if (GetAicpuUnfoldFlag() || GetAicpuCommEngine()) { CHK_RET(CheckExitWaitResumeState(...)); }
   只有AICPU环境才执行快恢状态检查

3. 多处HCCL_INFO改为HCCL_RUN_INFO并加group标识——增强日志可追溯性

[替代方案]:
- 在CheckExitWaitResumeState内部判断环境: 调用者太多，不如在入口处判断
- 用默认初始状态替代显式SetRetryState: 容易遗漏，显式设置更安全

[后果]:
opretry_manager.cc后续只有1个commit(子包适配)，没有再出现快恢失败。
快恢路径趋于稳定。

[可迁移经验]:
状态机的初始状态必须在构造/注册时显式设置，不能依赖默认值或后续流程隐式初始化。
设备侧路径(AICPU/CCU)的容错逻辑必须有环境判断门控，避免在不支持的环境中执行。

---

### Case BF-4: fix bugs of CP algorithm (PCIe链路结果错误)

[Commit]: hcomm:9bcb1bdc ("[Bugfix] fix bugs of CP algorithm")
[类别]: BUGFIX
[涉及层]: hcomm:algorithm (base/alg_template/temp_alltoallv/alltoallv_continuous_pipeline.cc)
[变更规模]: 3 files, +201/-60 lines (含181行新增测试)
[维度]: B(Host-Device)

[场景]:
AlltoAllV的Continuous Pipeline(CP)算法在走PCIe链路(跨NUMA/跨socket)时，
结果错误且性能差。CP算法将alltoallv分为intra(节点内)和inter(节点间)两层，
inter层使用SDMA或RDMA搬移数据。

[约束]:
- CP算法需要在inter阶段知道每个rank的receive信息(recvCounts/recvDispls)
- needCollectInfo_标志控制是否需要阻塞等待receive信息
- inter和intra两层的启动顺序不同(intra先还是inter先)

[决策]:
三个关键修复:

1. 初始化遗漏: PrepareSendRecvInfo中设置needCollectInfo_=true后，
   没有初始化intraRecvCounts_——新增std::copy填充本rank的recvCounts
   (缺少此初始化导致recvCount全为0，读取长度为0)

2. inter/intra顺序颠倒: RunAsync中"第一轮等待receive信息"的条件从
   intraState.loopNum==0改为interState.loopNum==0
   (等待应在inter之前，而非intra之前——因为inter才需要远端信息)

3. SDMA路径简化: 删除整个InterSdmaTx函数和needCollectInfo_的SDMA特殊路径，
   统一使用InterSdmaRx(读模式)替代InterSdmaTx(写模式)
   (原设计: 在收到receive信息前用SDMA写，收到后用SDMA读——
    这增加了不必要的复杂性，且写模式的同步协议更复杂)

4. isSDMALink条件清理: 移除"!isSDMALink || needCollectInfo_"中的needCollectInfo_条件，
   简化为仅按链路类型判断

[替代方案]:
- 修复InterSdmaTx而非删除: 但TX(写)模式天然比RX(读)模式更复杂(需要额外的前后同步)
  且在有receive信息的情况下RX完全够用，所以删除更安全
- 只修inter/intra顺序: 不够——intraRecvCounts_初始化也是独立bug

[后果]:
后续无再修改(只有1个子包适配commit)。新增了181行测试用例验证PCIe链路场景。

[可迁移经验]:
分层pipeline算法中，层间的数据依赖(哪层先获得信息、哪层先启动)必须显式建模。
"同时修复多个独立bug"的commit应确保每个fix有独立测试——
这里3个fix虽在同一commit中，但根因各不相同(初始化遗漏/顺序错误/冗余路径)。

---

### Case BF-5: CCU NHR template bugfix (GetThreadNum返回值)

[Commit]: hccl:992fd5b ("ccu nhr template bugfix")
[类别]: BUGFIX
[涉及层]: hccl:ops (reduce_scatter/template/ccu/ccu_temp_reduce_scatter_nhr_1D_mem2mem.cc)
[变更规模]: 1 file, +1/-1 lines (仅改一个数字)
[维度]: B(Host-Device)

[场景]:
CCU(Communication Control Unit)模式的ReduceScatter NHR(Non-Hierarchical Ring) 1D模板，
GetThreadNum()返回值错误。

[约束]:
- CCU模式下，kernel在设备端运行，host端通过template设置参数
- GetThreadNum()返回kernel需要的线程数，影响CCU调度器的资源分配
- NHR 1D mem2mem模式需要2个线程(一个做计算/搬移，一个做通信同步)
- 错误的线程数会导致CCU调度死锁或资源不足

[决策]:
将GetThreadNum返回值从1改为2。

[替代方案]: 无，这是一个明确的参数错误。

[后果]:
后续该文件有4个commit(another bugfix f623ff7, ccu algorithm 9dd181d, new selector 549c453, CleanCode 84ddec9)，
说明该CCU模板仍在活跃开发中，GetThreadNum=2是正确的。

[可迁移经验]:
CCU template的参数(threadNum/loopNum等)影响设备侧调度，
错误的参数值导致的症状(死锁/超时)与根因(参数错误)距离很远。
新增CCU template时必须对每个GetXxxNum方法做单元验证。
这也印证了Phase 1结论: "CCU template是bug集中区"。

---

### Case BF-6: fix missing signals (AICPU .so缺符号)

[Commit]: hccl:b779699 ("[Bugfix] fix missing signals")
[类别]: BUGFIX
[涉及层]: hccl:ops (CMakeLists.txt, op_common/executor/channel/channel.cc)
[变更规模]: 2 files, +5/-0 lines
[维度]: B(Host-Device) + D(协议)

[场景]:
scatter算子的AICPU侧.so文件编译时缺少符号(topo_match_ubx.cc)，
同时channel.cc中的GetTopoTypeByLink函数在AICPU编译时链接了不可用的Host-only API。

[约束]:
- hccl代码需要编译为两个目标: host侧(.so)和AICPU侧(.so)
- Host侧API(如HcclRankGraphGetTopoInstsByLayer)在AICPU编译环境中不存在
- AICPU_COMPILE宏区分编译目标
- 链接时才发现符号缺失(编译时不报错)

[决策]:
1. CMakeLists.txt: 在scatter_aicpu_kernel的源文件列表中添加topo_match_ubx.cc
2. channel.cc: 用#ifndef AICPU_COMPILE条件编译包裹GetTopoTypeByLink的host-only实现，
   AICPU编译时直接返回HCCL_SUCCESS

[替代方案]:
- 在AICPU编译时提供stub实现: 更规范但需要维护额外的stub文件
- 重构GetTopoTypeByLink使其不依赖host API: 改动太大，且topo查询本身不需要在AICPU侧执行

[后果]:
修复后scatter AICPU.so可以正确编译链接。后续无回退。

[可迁移经验]:
Host/Device双目标编译时，新增的.cc文件必须同时检查是否需要添加到两个CMakeLists.txt。
引用Host-only API的函数必须有AICPU_COMPILE条件编译保护——
链接错误是编译环节最后才暴露的问题，发现时已延误。

---

### Case BF-7: fix A3 bug (IndependentOp路由 + topo解析隔离)

[Commit]: hccl:3e9fbbd ("fix a3 bug")
[类别]: BUGFIX
[涉及层]: hccl:ops (scatter/scatter_op.cc, op_common/topo/topo_host.cc)
[变更规模]: 2 files, +11/-11 lines
[维度]: B(Host-Device)

[场景]:
A3芯片(910_93)运行scatter算子时，在没有设置IndependentOp环境变量的情况下
错误地走到了老流程(HcclScatterInner)。同时A3和A5的topo解析没有隔离，
CalcTopoLevelNums/CalcLevel0TopoShape调用了仅910_95支持的API。

[约束]:
- scatter_op.cc有复杂的5级fallback路由:
  CheckHCCLIndependentOp → deviceType → RunIndependentOpExpansion → AclGraph → WorkflowMode
- A3(910_93)应该走新流程(IndependentOp)，但旧代码在检查IndependentOp环境变量之前
  就做了fallback判断
- CalcTopoLevelNums/CalcLevel0TopoShape是910_95专用的拓扑解析函数

[决策]:
1. scatter_op.cc: 将IndependentOp环境变量检查移到deviceType获取之后，
   并增加条件: "!CheckHCCLIndependentOp() && deviceType != DEV_TYPE_910_93"
   (A3芯片即使没设环境变量也走新流程)
   同时将InitEnvConfig()调用移到后面，并注释"调用位置有特殊要求，不要变化"

2. topo_host.cc: 将CalcTopoLevelNums+CalcLevel0TopoShape包在
   if (deviceType == DEV_TYPE_910_95) 条件内

3. 指针偏移修复(scatter_op.cc:299): 修正了AlgResourceCtx内部结构的sizeof计算，
   添加了 sizeof(uint32_t)*AICPU_CONTROL_NOTIFY_NUM + sizeof(void*)
   (原代码少算了两个字段，导致指针偏移错位)

[替代方案]:
- 为A3专门增加一个分支: 代码重复，且未来新芯片(A5)也需要类似处理
- 在Selector中处理芯片差异而非Op入口: Selector的介入太晚，Op入口需要先决定走新/旧流程

[后果]:
scatter_op.cc后续有多个commit继续完善(说明scatter是最活跃算子——Phase 1: 20次修改)。
InitEnvConfig的注释"调用位置有特殊要求，不要变化"暗示此处已经是经验教训沉淀。

[可迁移经验]:
多芯片适配时，fallback路由的条件顺序决定了哪些芯片走哪条路径——
新增芯片支持时必须逐行审查整个fallback链。
结构体指针偏移计算(reinterpret_cast+sizeof)是高危操作，
结构体新增字段后所有手动偏移计算都需要同步更新。

---

### Case BF-8: Hcomm Thread API bugfix (跨层thread生命周期重构)

[Commit]: hcomm:fa61b9fc ("Hcomm Thread API bugfix")
[类别]: BUGFIX
[涉及层]: hcomm:framework (next/comms/ + communicator/impl/independent_op/ + device/)
[变更规模]: 18 files, +642/-182 lines
[维度]: B(Host-Device) + E(并发)

[场景]:
HcommThreadAlloc和HcommThreadFree是V2(next)架构的线程管理API，
用于在host侧分配/释放AICPU侧的通信线程。
原实现存在多个功能缺陷:
- Kernel binary未按需加载(需要AICPU线程时才加载ccl_kernel.json)
- 线程handle的D2H(Device-to-Host)映射在销毁时残留
- 线程创建/初始化不够健壮(缺少参数校验拆分)

[约束]:
- Thread API是hcomm对外的C API(extern "C")，跨越Host↔Device边界
- AICPU线程的创建需要: (1)host侧分配 (2)kernel binary加载 (3)device侧launch (4)handle映射
- 销毁时需要: (1)查D2H映射 (2)调device侧destroy kernel (3)清理映射 (4)释放host对象
- 线程handle在g_ThreadMap(shared_ptr)和g_ThreadD2HMap(handle映射)中双重管理
- bin handle需要全局唯一(g_BinHandle)，且线程安全

[决策]:
这是一个跨层重构式bugfix:

1. hcomm_c_adpt.cc(C API适配层):
   - 新增EnsureKernelBinLoaded: 懒加载ccl_kernel.json，用g_BinHandleMtx保护
   - 新增g_BinHandle全局变量(单例模式)
   - HcommThreadAlloc/Free的逻辑委托到thread.cc中的新函数

2. thread.cc(线程管理核心):
   - 新增ValidateThreadParams: 参数校验独立函数
   - 新增CreateAndInitThreads: 创建+初始化循环
   - 新增SaveThreads: 线程handle写入g_ThreadMap和g_ThreadD2HMap
   - 新增FillThreadD2HMap: 处理device handle到host handle的映射
   - 新增FreeThreadHandlesLocked: 销毁时的映射清理(遍历D2HMap删除所有指向被销毁handle的映射)
   - 新增FreeThreads: 整合销毁流程(映射清理 + device kernel launch destroy)

3. aicpu_launch_manager.cc(AICPU启动管理):
   - 简化KernelLaunchAicpuCustom: 删除冗余的InitTask结构体
   - 将ThreadKernelLaunch拆分为多个职责清晰的函数(PrepareThreadMgrParam, CreateLocalStream)

4. aicpu_thread_process.cc(新增): device侧线程处理逻辑

[替代方案]:
- 只修ThreadAlloc/Free的bug不重构: 映射残留问题需要重新设计数据结构，补丁修复不够
- 使用RAII/智能指针自动管理thread: g_ThreadD2HMap的handle是device指针，
  不能直接用C++ RAII管理(需要kernel launch销毁)

[后果]:
新增ccl_kernel.ini配置文件(设备侧kernel入口信息)。
修改了test/st和test/ut的相关测试桩。
这是一个典型的"修复不如重写"的case。

[可迁移经验]:
跨Host-Device边界的资源(线程/stream/notify)需要双向映射管理(D2H Map)。
销毁时必须完整清理所有映射条目——残留的映射条目是use-after-free的根源。
对外C API层应做thin wrapper，核心逻辑放在内部模块中。

---

### Case BF-9: CCU模式scatter_mesh1d修改地址设置时机

[Commit]: hccl:80a70729 ("[Fix] ccu模式scatter_mesh1d算子修改每张卡设置地址的时机")
[类别]: BUGFIX
[涉及层]: hccl:ops (scatter/template/ccu/kernel/ccu_kernel_scatter_mesh1d.cc, op_common/op_common.cc + 删除reduce_mesh_2D相关12个文件)
[变更规模]: 14 files, +12/-1265 lines (含1254行删除废弃的reduce_mesh_2D模板)
[维度]: B(Host-Device)

[场景]:
CCU模式的scatter_mesh1d算子，每张卡的输入/输出地址设置在DoScatter函数内部的CCU_IF块中。
这意味着地址设置只在CCU_WHILE循环的首次迭代中执行。
但问题是: 当需要多轮重复scatter(DoRepeatScatter → CCU_WHILE循环)时，
地址设置在循环体内部，每次循环都会被重新设置——
而CCU_IF(flag_ != 0)分支中的增量偏移逻辑依赖地址已经在循环外正确初始化。

[约束]:
- CCU kernel中的代码是微码(microcode)，执行语义与普通C++不同:
  CCU_WHILE/CCU_IF是CCU控制指令，不是普通的while/if
- 地址设置(token/addr赋值)是在host侧编排时确定的，device侧不能动态改
- rankSize_和token_是编排参数，在CCU_WHILE前就已确定

[决策]:
将地址设置循环(for curId = 0 to rankSize_)从DoScatter()移到DoRepeatScatter()中，
放在CCU_WHILE循环之前。这确保:
1. 地址在循环开始前正确初始化
2. CCU_WHILE内部的CCU_IF(flag_ != 0)分支只做增量偏移

同时:
- op_common.cc: 将param.hcclComm赋值从HcclExecOp移到Selector(更早设置)
- 删除了12个废弃的reduce_mesh_2D模板文件(1254行)——乘此commit清理技术债

[替代方案]:
- 在CCU_WHILE内部每次迭代都重新计算完整地址(非增量): 可行但性能差(CCU微码中循环体应最小化)
- 通过CCU寄存器保存初始地址: CCU寄存器资源有限，不适合保存全rank的地址

[后果]:
scatter_mesh1d的后续只有1个commit(ccu algorithm 9dd181d)继续开发，
说明地址时机问题已修复。

[可迁移经验]:
CCU微码中，初始化代码(地址设置/参数计算)必须放在CCU_WHILE循环外部。
CCU_WHILE内部应只包含每轮需要变化的增量逻辑。
这是CCU编程模型与普通C++的关键区别——CCU控制流不等同于CPU控制流。

---

## LIFECYCLE Cases (4)

### Case LC-1: AlltoAll Symmetric Memory (对称内存设计)

[Commit]: hcomm:e63840b6 ("alltoall symmetric memory")
[类别]: LIFECYCLE (FEATURE + 资源管理模式)
[涉及层]: hcomm:algorithm + hcomm:framework
[变更规模]: 20 files, +879/-18 lines

[场景]:
为AlltoAll算子在超节点内AICPU模式下增加对称内存(Symmetric Memory)支持。
对称内存是一种分布式内存抽象: 所有rank上都有相同大小/偏移的内存区域，
可以直接通过对端rank的地址计算访问(不需要额外的内存注册/交换)。

[约束]:
- 对称内存的生命周期: 由HcommSymWin管理(window-based API)
- 每个link需要知道对端的输入/输出地址(remoteIn/remoteOut)
- 只有特定链路类型支持(HCCS/SIO/HCCS_SW)
- 与现有的alltoallv direct_fullmesh算法共存

[决策]:
三层扩展:

1. algorithm层:
   - 新增AlltoAllFullMeshSymmetricMemory类(继承AlgTemplateBase)
   - 新增CollRunAlltoAllFullMeshSymmetricMemory executor
   - executor标记desc_.isZeroCopy = true(零拷贝模式)

2. framework/device层:
   - 修改PrepareSymmetricMemory: 从仅遍历COMM_LEVEL0扩展为遍历所有level，
     但只处理isZeroCopy==true的transport
   - 通过HcommSymWinGetPeerPointer获取对端地址
   - 通过link->UpdateRemoteAddr设置远端地址

3. 多个现有executor:
   - ring_zerocopy系列(allgather/allreduce/broadcast/reduce_scatter)
     都新增了isZeroCopy = true标记
   - 说明对称内存不只是alltoall专用，而是一个通用特性

[替代方案]:
- 不用对称内存，依赖传统的RegisterMem+ExchangeMemDesc: 需要额外的控制面交互
- 只在alltoall中支持: 但isZeroCopy标记说明架构设计考虑了通用性

[后果]:
aicpu_communicator.cc后续有6个commit继续修改(包括scatter/symmetric相关)，
说明对称内存是活跃开发中的新特性。

[可迁移经验]:
对称内存的关键设计: 用isZeroCopy标记区分transport是否使用对称内存路径。
PrepareSymmetricMemory必须遍历所有topology level(不只是LEVEL0)，
因为超节点内的拓扑可能跨多个level。
新增内存管理路径时，必须同时考虑所有使用该路径的算子。

---

### Case LC-2: memDesc生命周期管理 (内存描述符所有权转移)

[Commit]: hcomm:3315532a ("memDesc生命周期管理")
[类别]: LIFECYCLE
[涉及层]: hcomm:framework (next/comms/endpoints/reged_mems/ + platform/resource/buffer/)
[变更规模]: 16 files, +78/-45 lines

[场景]:
V2(next)架构的内存注册/导出流程中，MemoryExport生成的memDesc(内存描述符)
的生命周期管理有问题:
1. RegisterMemory重复注册时(HCCL_E_AGAIN)不返回memHandle，导致调用者无法获取句柄
2. MemoryExport将序列化后的数据memcpy到调用者提供的buffer中——
   这意味着调用者需要知道buffer大小(TRANSPORT_EMD_ESC_SIZE)，且buffer由调用者管理

[约束]:
- RoceRegedMemMgr和UbRegedMemMgr是两个并行的内存管理器(RoCE vs URMA)
- memHandle是void*(实际指向LocalRdmaRmaBuffer/LocalUbRmaBuffer)
- memDesc是序列化后的二进制数据(包含endpoint信息+buffer信息)
- 调用者(endpoint/channel)需要将memDesc发送给对端，对端反序列化后建链

[决策]:
将memDesc的所有权从调用者转移到buffer对象本身:

1. RegisterMemory: 重复注册时(HCCL_E_AGAIN)也返回memHandle
   ```
   // before: 重复注册直接return HCCL_E_AGAIN，memHandle未设置
   // after: *memHandle = static_cast<void*>(localRegisteredBuffer.get());
   //        然后return HCCL_E_AGAIN
   ```

2. MemoryExport: 不再memcpy到外部buffer，而是将序列化结果存储在buffer对象的Desc成员中
   ```
   // before: memcpy_s(*memDesc, TRANSPORT_EMD_ESC_SIZE, data, size)
   // after:  localRdmaRmaBuffer->Desc = std::move(tempLocalMemDesc);
   //         *memDesc = static_cast<void*>(localRdmaRmaBuffer->Desc.data());
   //         *memDescLen = localRdmaRmaBuffer->Desc.size();
   ```

3. 新增GetMemDesc辅助函数: 将序列化逻辑从MemoryExport中提取出来

4. const void* -> void*: MemoryExport的memHandle参数从const去掉
   (因为现在需要修改buffer对象的Desc成员)

5. Buffer类新增Desc成员(local_rdma_rma_buffer.h, local_ub_rma_buffer.h)

6. GetParamsFromMemDesc从自由函数变为RoceRegedMemMgr的成员函数

[替代方案]:
- 让调用者管理memDesc的生命周期: 原方案就是这样，但调用者需要知道buffer大小(耦合)
- 用shared_ptr<vector<char>>管理memDesc: 可行但引入额外的引用计数开销
- move语义直接存储在buffer对象中: 这就是最终选择(零拷贝)

[后果]:
RoCE和URMA两个管理器做了相同的修改(对称修改模式)，
说明这是一个架构级的生命周期决策，而非单点修复。

[可迁移经验]:
序列化产物(memDesc/config/metadata)的所有权应属于产生它的对象本身，
而非由调用者提供buffer——这减少了调用者对buffer大小的耦合。
重复注册路径(HCCL_E_AGAIN)也必须返回有效的handle，
否则调用者无法继续操作已注册的资源。

---

### Case LC-3: Judge resource Update From host to kernel

[Commit]: hcomm:9167c142 ("Judge resource Update From host to kernel")
[类别]: LIFECYCLE
[涉及层]: hcomm:legacy (framework/communicator/aicpu/ + service/) + test
[变更规模]: 13 files, +84/-133 lines (净减少49行)
[维度]: B(Host-Device)

[场景]:
AICPU kernel的资源刷新(UpdateRes)判断逻辑原本在host侧:
host决定"是否需要更新资源"(needUpdateRes flag)，然后通过kernelParam传递给device侧。
但在aclgraph replay场景(同一rank用不同对象重放同一图)下，
host无法准确判断device侧是否真的需要刷新。

[约束]:
- aclgraph replay: 同一个图被不同HcclComm对象重放，
  每次重放时图的拓扑相同但资源(stream/notify/transport)可能不同
- host侧的needUpdateRes=false在replay时可能错误: host认为"已经加载过"但device侧的资源已过期
- legacy层(communicator_impl_lite.cc)和service层(coll_service_ai_cpu_impl.cc)
  都依赖needUpdateRes flag

[决策]:
将"是否需要更新资源"的判断从host侧移到device侧(kernel内):

1. coll_service_ai_cpu_impl.cc(host侧service):
   - OpBasedCollProcess: 删除needUpdateRes出参，改为直接返回已有的DevBuffer指针
     (如果已加载过，返回已有的buffer; 否则新分配)
   - 删除所有needUpdateRes相关逻辑和AicpuKernelEntranceLaunch的needUpdateRes参数

2. communicator_impl_lite.cc(device侧kernel):
   - 删除kernelParam->needUpdateRes的检查
   - 新增CheckNeedUpdateRes: 用loadedOpSet(set<string>)在device侧记录已加载的tag
   - 首次看到某tag时返回true并加入loadedOpSet，后续返回false
   - UpdateTransports和UpdateRes都改用CheckNeedUpdateRes判断

3. kernel_param_lite.h: 删除needUpdateRes字段

[替代方案]:
- host侧在replay时强制设needUpdateRes=true: 性能退化(每次都刷新)
- device侧用版本号而非set判断: 更精确但实现复杂
- 不改，依赖host侧correctness: replay场景必定出错

[后果]:
修改了UT(ut_communicator_impl_lite.cc, ut_aicpu_mc2_handler.cc等)验证新逻辑。
净减少49行代码(删除了跨层传递的needUpdateRes管道)。

[可迁移经验]:
资源是否需要刷新的判断应放在资源的使用方(device kernel)而非提供方(host service)——
因为使用方最清楚自己的当前状态。
跨Host-Device边界传递flag来控制行为是脆弱的——flag可能在任何中间环节被错误设置。
更好的模式是: 使用方自治判断(idempotent check)。

---

### Case LC-4: fix aclgraph launch: exhausted stream resources

[Commit]: hcomm:c5443da0 / hcomm-dev:6faf63c9
("fix aclgraph launch in order: fixing the issue of exhausted stream resources")
[类别]: LIFECYCLE
[涉及层]: hcomm:framework (common/src/order_launch/ + communicator/)
[变更规模]: hcomm: 5 files, +91/-48 lines; hcomm-dev: 7 files, +100/-48 lines
[维度]: A(跨仓)

[场景]:
aclgraph模式下的OrderLaunch(顺序执行控制)机制为每个model分配独立的host order stream，
导致stream资源耗尽。原设计: aclgraphStreamMap_[modelId]为每个model创建一个Stream，
并通过AddStreamToModel将其绑定到model——model数量增长时stream资源枯竭。

[约束]:
- aclgraph模式: 多个通信算子图化后顺序执行，需要order stream控制执行顺序
- stream是硬件资源(有限): AICPU stream数量受设备限制
- order stream只做同步(signal/wait)不做计算，理论上可以所有model共享
- Event机制比AddStreamToModel更灵活(event不绑定model)

[决策]:
用全局唯一的order stream + Event对替代per-model的stream:

1. 将aclgraphStreamMap_(map<modelId, Stream>)替换为aclgraphStream_(unique_ptr<Stream>)
   所有model共享同一条order stream

2. 用aclgraphEvents_[2]代替stream绑定:
   - event[0]: kernel stream → order stream (Record在kernel stream上, Wait在order stream上)
   - event[1]: order stream → kernel stream (Record在order stream上, Wait在kernel stream上)
   这样order stream不需要AddStreamToModel

3. AclgraphLaunchInOrder拆分为两个函数:
   - AclgraphLaunchInOrderToOrderStream: kernel→order(执行前同步)
   - AclgraphLaunchInOrderToKernelStream: order→kernel(执行后恢复)

4. 新增DestoryRes: 集中销毁order stream + event资源
   UnRegisterOrderLaunch在最后一个group注销时调用DestoryRes

5. 删除#ifndef CCL_KERNEL_AICPU保护(LaunchInOrder函数):
   原来在AICPU编译时跳过整个函数，现在始终编译

[替代方案]:
- 增加stream池上限: 治标不治本，model数量仍可能超限
- 每N个model共享一个stream: 增加调度复杂度
- 不用order stream，用纯notify: notify也是有限资源，且不如event灵活

[后果]:
hcomm和hcomm-dev两仓同步修复(同一作者，hcomm-dev先提交2小时)。
后续只有1个merge commit，修复本身稳定。

[可迁移经验]:
硬件资源(stream/event/notify)是有限的——设计时必须考虑最坏情况下的资源消耗。
"每个实例分配独立资源"的模式在通信框架中往往不可行(实例数=model数*rank数)。
优先使用共享资源+同步原语(event)而非独占资源(stream per model)。

---

## 交叉分析: BUGFIX模式总结

### 根因分类

| 模式 | Cases | 比例 |
|------|-------|------|
| 时序/顺序错误 | BF-1, BF-4, BF-9 | 3/9 (33%) |
| 并发/锁设计 | BF-2, BF-8 | 2/9 (22%) |
| 环境/编译条件遗漏 | BF-6, BF-7 | 2/9 (22%) |
| 参数错误 | BF-5 | 1/9 (11%) |
| 状态机初始化 | BF-3 | 1/9 (11%) |

### 修复策略分类

| 策略 | Cases | 特征 |
|------|-------|------|
| 语句重排(最小改动) | BF-1, BF-9 | 改动<5行，但定位成本极高 |
| 底层替换(换实现) | BF-2 | 替换整个锁机制，100+行 |
| 条件门控(加判断) | BF-3, BF-6, BF-7 | 加if/ifdef保护，通常<10行 |
| 逻辑简化(删冗余) | BF-4 | 删除冗余路径+修正逻辑 |
| 跨层重构 | BF-8 | 18文件重写，属于"修复不如重写" |
| 参数修正 | BF-5 | 改1个数字 |

### LIFECYCLE模式总结

| 模式 | Cases | 核心教训 |
|------|-------|----------|
| 资源共享 vs 独占 | LC-4 | stream等硬件资源必须共享设计 |
| 所有权转移 | LC-2 | 序列化产物的所有权属于产生者 |
| 判断权下放 | LC-3 | "是否需要刷新"的判断放在使用方 |
| 新内存模式引入 | LC-1 | 对称内存需要标记(isZeroCopy)贯穿所有层 |

### Host-Device边界是bug富矿

9个BUGFIX中有7个涉及Host-Device边界(BF-4到BF-9 + BF-1间接涉及)。
LIFECYCLE的4个case中3个涉及Host-Device边界(LC-1/LC-3/LC-4)。
这完全验证了Phase 1的结论: 设备侧代码的bug密度显著高于host侧。

原因分析:
1. 编译目标分裂: 同一代码需要编译为host.so和aicpu.so，条件编译容易遗漏
2. 语义分裂: CCU微码的控制流(CCU_WHILE/CCU_IF)不等同于C++控制流
3. 调试困难: 设备侧bug的表现(超时/死锁)与根因(参数错误/地址时机)距离远
4. 状态同步: host和device之间的flag/状态传递是脆弱的(LC-3的教训)
