# torch_npu Developer Decision Casebook

Repository: `torch_npu` (PyTorch Ascend NPU Backend)
Source: 1316 defect analyses from git history (28,222 commits), architecture map, design patterns catalog
Method: bottom-up scenario clustering from defect root causes + decision dimension enrichment

本文档按开发者任务场景组织，每个case提取: 场景、约束、决策、替代方案、后果、可迁移经验。
Error-fix对和revert链显式标注。


## Table of Contents

- Ch.1  分布式通信 (HCCL / ProcessGroup)
- Ch.2  内存管理与显存分配
- Ch.3  算子实现与注册
- Ch.4  Python集成与Monkey-patch
- Ch.5  Profiler与可观测性
- Ch.6  初始化与生命周期
- Ch.7  ACL接口绑定
- Ch.8  Inductor编译器后端
- Ch.9  构建系统与跨平台兼容
- Ch.10 序列化与模型加载
- Ch.11 测试基础设施
- Ch.12 横切模式
- Ch.13 Revert-Driven决策教训(补遗)
- Ch.14 Profiler数据管道与兼容性
- Ch.15 ACL运行时接口兼容性与版本守卫
- Ch.16 Inductor编译器后端边界适配


---

## Ch.1 分布式通信 (HCCL / ProcessGroup)

ProcessGroupHCCL是torch_npu中缺陷触及次数最高的单文件(91次bugfix, 7147行)，
也是revert最密集的子系统(3个独立revert事件)。本章覆盖HCCL通信层的典型开发决策场景。


### 1.1 Stream管理与内存安全

HCCL集合通信使用专用stream异步执行，与计算stream并行。两类stream共享NPU显存池，
因此必须正确管理tensor在通信stream上的生命周期，否则计算stream可能复用尚在通信中的内存。
这一领域产生了3个独立revert事件，是torch_npu中最难做对的设计决策区。


#### Case 1.1.1: recordStream的batch遗漏 -- 只保护tensors[0]

Commits: c86a6dda3 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-1

Scenario: 开发者实现batch_isend_irecv的内存安全保护。collective()的pre callback
需要对参与DMA的tensor调用recordStream()，防止计算stream在通信完成前复用这些tensor的内存。

Constraints: HCCL的batch_isend_irecv接收tensor列表。4种内存复用模式
(CLOSE / AVOID_RECORD_STREAM / ERASE_RECORD_STREAM / ERASE_RECORD_STREAM_WITH_OPTIMIZE)
各有不同处理路径。

Decision: 原实现只对tensors[0]调用recordStream，遗漏了tensors[1..n]。
Fix: pre callback中循环遍历所有tensor，按模式分别处理。+20行。

Alternatives:
- 在collective()框架层统一处理所有tensor(而非在每个op的callback中) -- 更彻底但改动面大
- 对整个output tensor做device同步(粗粒度，性能差但安全)

Consequence: 修复前，多流并行场景下tensors[1..n]的内存可能被计算流覆盖，导致NaN。
单stream场景不触发(通信和计算串行)，因此开发阶段难以复现。

Transferable: 批量操作中，对容器内"所有元素"做资源管理处理，不只是第一个。
这是"off-by-one的语义变体"：不是数值差一，而是集合遗漏。Review时对照batch语义检查循环边界。


#### Case 1.1.2: recordStream的强引用 vs GC回收 -- 两次revert的教训

Commits: 674cbdaff (eraseStream架构), b27e5f091 (revert storage强引用),
  3f372302e (revert syncOp流选择优化)
Files: ProcessGroupHCCL.cpp, NPUCachingAllocator.cpp
Defect ref: D-267, D-198, D-568, D-569

Scenario: 开发者尝试优化HCCL通信后的内存回收效率。核心矛盾：
recordStream机制用event标记block归属stream，通信完成后需等event完成才释放内存。
但recorded_events集合的查询阻塞了释放路径，导致内存无法及时回收。

Constraints:
- 安全性: 通信完成前tensor内存不能被回收(否则DMA读到垃圾数据)
- 性能: event查询阻塞释放路径影响训练throughput
- GC兼容: Python侧tensor的引用计数归零后，storage应能被回收

Decision(演化过程):
1. 初始方案: 通过storage调用recordStream/eraseStream追踪tensor生命周期
   -> Revert(D-267): storage强引用阻止GC回收，长训练积累内存泄漏
2. 优化方案: sync op直接使用当前流，跳过syncStreams和recordStream
   -> Revert(D-198): 破坏了两个不变量(event同步+lifetime保护)，多stream场景数据竞争和UAF
3. 最终方案(D-568): 新增eraseStream()方法，在WorkHCCL::synchronize()中主动从block
   的stream_uses移除stream并lazy destroy event。用weak_intrusive_ptr追踪storage。
   环境变量MULTI_STREAM_MEMORY_REUSE控制开关。

Alternatives:
- 全局device同步(安全但性能极差)
- 引用计数自动管理(Python侧weak ref，但C++侧event生命周期无法用引用计数表达)
- 放弃异步释放，通信完成后同步释放(简单但阻塞计算)

Consequence: 经过两次revert，第三次方案(eraseStream)稳定运行。
关键技术转折: 从"被动等待event完成后释放"转为"主动在synchronize时移除stream标记"。

Transferable:
- 多流内存管理的三个不变量: (1)event同步保证数据可见性 (2)recordStream保证lifetime
  (3)不能阻止GC回收。三者同时满足极其困难。
- 引用强度选择: strong ref保安全但阻GC，weak ref允许GC但需额外的alive检查。
  HCCL场景选weak_intrusive_ptr是正确折中。
- 用环境变量控制高风险优化的开关，允许回退。


#### Case 1.1.3: stream ID获取时机 -- 异步上下文中的状态捕获

Commits: adc3e0d5a (fix) [+1 cherry-pick]
Files: ProcessGroupHCCL.cpp
Defect ref: D-305

Scenario: 开发者在约20处collective/p2p操作中获取stream ID用于profiler的MstxRange标注。

Constraints: collective()内部创建HCCL stream并传入lambda闭包。
stream ID只在stream创建后才有效。

Decision: 原实现在collective()调用之前调用getStreamId()，然后通过lambda capture传入。
首次调用时hcclStreams_中还没有对应entry，getStreamId返回-1。

Fix: 删除所有提前调用(约20处)，在lambda闭包内部改为stream.id()直接获取。
stream是collective()内部创建并传入lambda的NPUStream引用，此时ID必然有效。

Alternatives: 无更好方案。在执行上下文中获取状态是唯一正确做法。

Consequence: 修复前profiler记录的stream ID全部错误(-1)，影响性能分析准确性。
修改虽然是机械替换(20处相同模式)，但需要理解stream创建时序才能定位root cause。

Transferable: 异步/延迟执行的操作中，状态信息(stream ID、device ID等)应在实际执行
上下文中获取，不应在闭包外部提前捕获。这是"capture by value时序问题"的实例。


### 1.2 HCCL通信域创建与配置

HCCL通信域(communicator)是集合通信的基础资源。创建通信域涉及rank映射、
配置字段兼容性、collective语义等多维约束。


#### Case 1.2.1: ranktable场景下local rank与global rank混淆

Commits: 00572e020 (fix rank映射), related: D-235 (名称冲突)
Files: ProcessGroupHCCL.cpp
Defect ref: D-225, D-235

Scenario: 开发者为ranktable配置模式实现P2P通信域创建。HCCL需要全局rank ID定位对端设备，
但ProcessGroup内部使用local rank(从0开始的组内编号)。

Constraints:
- 非ranktable模式: local rank可直接用作HCCL rank(ProcessGroup和HCCL rank空间一致)
- ranktable模式: global_ranks_in_group非空，local rank和global rank不一致
- P2P通信域名称在所有group间必须唯一

Decision: 原实现直接用local rank传给HCCL。Fix: 检查global_ranks_in_group是否为空，
非空时通过映射表转换local->global rank，空时保持原逻辑。加bound check。
后续D-235进一步修复了P2P通信域名称跨group冲突(名称格式中加入group标识)和
子group条件限制(排除不参与P2P的rank)。

Alternatives:
- 统一在ProcessGroup初始化时建立完整的rank映射表(更彻底但改动大)
- 强制所有模式使用global rank(破坏非ranktable模式的向后兼容)

Consequence: 修复前，ranktable模式下P2P操作指向错误设备，通信数据送达wrong rank。
两个fix前后间隔不长，说明rank映射问题有隐藏的关联维度(名称唯一性)。

Transferable: 分布式系统中存在多个rank空间时，每次使用rank ID都要明确"这是哪个空间的rank"。
建议: 用类型区分(LocalRank / GlobalRank wrapper)或命名约定(local_rank / global_rank)。


#### Case 1.2.2: NSLB-DP配置项的copy-paste赋值错误

Commits: dbe7a336c [+5 cherry-picks]
Files: ProcessGroupHCCL.cpp
Defect ref: D-288

Scenario: 开发者为createHcclCommConfigWithOptions()添加NSLB(Network-level Scheduling
and Load Balancing)和DP(Data Parallelism)相关配置项。

Constraints: HcclCommConfig结构体有多个字段(hcclOpExpansionMode, hcclWorldRankID,
hcclJobID等)，从Python options字典读取后赋值到对应字段。

Decision: 开发者copy-paste了一段赋值代码，修改了dict key("hccl_world_rank_id",
"hccl_job_id")，但遗漏了赋值目标字段名: 两处都赋值给了config.hcclOpExpansionMode。
结果: hcclWorldRankID和hcclJobID始终为默认值，hcclOpExpansionMode被覆盖两次。

Fix: 修正目标字段名。2行改动。

Alternatives: 用data-driven方式(配置表mapping dict key -> struct field)替代逐行赋值，
从结构上消除copy-paste遗漏的可能。

Consequence: NSLB和DP功能因配置错误而静默失效。无crash、无报错，极难发现。
cherry-pick到5个版本分支说明影响面广。

Transferable: 结构相似的赋值代码块(从dict读取->赋值到struct)是copy-paste错误的高发区。
Review rule: 逐字段核对左侧目标变量名。更好的做法: 抽取为配置mapping表，消除重复。


#### Case 1.2.3: CommConfig字段的CANN版本兼容性

Commits: ddf7e7b4c [+9 cherry-picks]
Files: ProcessGroupHCCL.cpp
Defect ref: D-326

Scenario: 开发者为HCCL通信域创建设置hcclCommName字段。

Constraints: HcclCommConfig结构体随CANN版本演进，新版本新增字段。
旧版本CANN的结构体中该字段不存在或语义不同。

Decision: 原实现无条件设置config.hcclCommName。Fix: 在设置前检查
isHcclFeatureSupported(HcclCommConfigCapability::HCCL_COMM_CONFIG_COMM_NAME)。

Alternatives:
- 编译期#ifdef(需要CANN版本宏，但torch_npu是动态链接CANN)
- 运行期try-catch(性能差且掩盖真实错误)
- capability query(选择的方案，最适合动态链接场景)

Consequence: 旧版本CANN上写入不存在的字段导致内存越界或功能异常。
cherry-pick到9个版本分支(全部活跃版本)。

Transferable: 动态链接的依赖库(CANN)接口演进时，写入新增字段前必须做capability check。
这不是CANN版本号比较(版本号不精确)，而是feature flag查询(精确到字段级别)。
这与D-63/D-79/D-214/D-253/D-290构成同一模式族: "API可用性 = 函数存在性 + SoC范围 + CANN版本"。


#### Case 1.2.4: new_group的collective语义 -- StressDetect多worker hang

Commits: 482eb0ab9 [+5 cherry-picks]
Files: npu/utils.py, Stress_detect.cpp
Defect ref: D-301

Scenario: 开发者实现stress_detect(mode=1)，需要创建HCCL子通信域执行硬件压力检测。

Constraints: torch.distributed.new_group()是collective操作，要求所有参与
torch.distributed初始化的rank同步调用。不是"只有参与者调用"，而是"全部rank都必须调用"。

Decision: 原实现只对当前worker调用了一次new_group(local_ranks)。
多worker(多node)场景下，其他worker的rank没有参与调用 -> 全局hang。

Fix: 遍历所有worker(for worker_id in range(num_workers))，为每个worker的rank子集
都调用new_group，仅保留属于当前worker的group句柄。

Alternatives:
- 使用HCCL原生API直接创建子comm(绕过PyTorch的collective语义，但失去PyTorch的group管理)
- 所有rank在barrier后统一创建(简单但需要额外同步点)

Consequence: 多worker场景静默hang，单worker场景(开发环境)正常 -> 典型的"开发环境通过、
生产环境失败"模式。

Transferable: PyTorch分布式API中，new_group/barrier/init_process_group等都是collective。
"collective"意味着"全部rank必须参与"，不仅仅是"想加入的rank调用"。
开发时易遗忘这一语义(因为单机测试通常只有1个rank)。


### 1.3 错误处理与生命周期

分布式通信组件的错误处理和资源清理比常规代码复杂得多:
析构时通信状态可能已损坏，watchdog线程可能在检查状态，多个rank可能处于不同的错误阶段。


#### Case 1.3.1: 析构函数中的异常处理 -- 静默死亡 vs 有意crash

Commits: 24f51db28 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-32

Scenario: 用户未显式调用destroy_process_group()，ProcessGroupHCCL在析构时触发shutdown()。
shutdown失败(通信状态已损坏)时，异常被析构函数的隐式noexcept吞掉。

Constraints:
- C++标准: 析构函数默认noexcept，异常传播触发std::terminate()
- 诊断需求: shutdown失败表明通信状态已损坏，需要coredump分析
- 用户体验: 静默退出无错误信息 vs 有意crash产生coredump

Decision: Fix选择"有意crash": try-catch后LOG(ERROR)输出完整错误信息并rethrow。
rethrow触发std::terminate()产生coredump，确保有可分析的现场。

Alternatives:
- 只记日志不rethrow(安全但丢失现场，后续分析困难)
- 设全局error flag由上层检查(不crash但需要上层配合)
- 在析构前强制同步所有通信(可能死锁)

Consequence: 修复前，分布式训练中节点静默挂起，运维人员无法定位原因。
修复后，shutdown失败产生明确的error log和coredump。

Transferable: 分布式组件的析构函数是"最后的诊断机会"。
对于不可恢复的错误(通信状态损坏)，有意crash+coredump优于静默退出。
但普通组件的析构不应这样做 -- 这是分布式场景的特殊决策。


#### Case 1.3.2: Watchdog状态管理 -- 未初始化变量与日志变量引用

Commits: c3b1e3712 (未初始化), 01442bd42 (日志错误)
Files: ProcessGroupHCCL.hpp, ProcessGroupHCCL.cpp
Defect ref: D-258, D-255

Scenario: watchdog线程监控HCCL通信健康状态，检测超时和设备错误。
WatchdogStatus枚举控制watchdog的行为(RUN/STOP/RECOVERED等)。

Constraints:
- watchdogStatus是类成员变量，构造函数可能不在所有路径上执行初始化
- C++对非静态成员的枚举类型不保证零初始化
- watchdog线程在构造完成后立即启动

Decision(D-258): watchdogStatus声明时未给初值。编译器不保证零初始化，
watchdog线程启动时读到的状态可能是任意值。
Fix: 改为`WatchdogStatus watchdogStatus = WatchdogStatus::RUN`显式初始化。

Decision(D-255): workCleanupLoop中日志传入device_error_msg(赋值前旧值)而非当次
捕获的device_error。workEnqueue异常信息硬编码"UCE ERROR."丢失实际内容。
Fix: 修正变量引用；恢复场景日志从error降级为info。

Alternatives: 对D-258，将所有成员集中在构造函数初始化列表(不遗漏但维护成本高)。
对D-255，日志参数绑定到catch块的exception变量(结构上不可能引用错误变量)。

Consequence: D-258可能导致watchdog在任意状态启动，恢复逻辑误判。
D-255导致错误日志内容与实际错误不匹配，误导问题定位。

Transferable:
- 枚举/类类型非静态成员变量必须在声明处赋初值(C++11 in-class initializer)。
- catch块内格式化日志时，核查参数引用的是更新前还是更新后的值。


#### Case 1.3.3: heartbeat monitor与shutdown竞争

Commits: 0fcf48af1 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-364

Scenario: ProcessGroupHCCL支持可选的heartbeat monitor线程，检测节点存活。
shutdown()需要安全地停止monitor线程并释放所有HCCL comm资源。

Constraints:
- shutdown()中async abort会destroy所有comm
- 正常清理路径也会destroy comm -> double free风险
- monitor线程可能hang在waitForFutureOrTimeout
- 短生命周期进程(测试)中monitor增加不必要的开销

Decision: 原实现monitorThreadEnabled_默认true，所有场景都启动heartbeat。
Fix三方面: (1) 默认改为false; (2) 移除shutdown中的async abort;
(3) monitor启动加条件判断。

Alternatives:
- monitor线程用cooperative cancellation(设flag+定期检查，不强制终止)
- 将monitor改为lazy启动(首次通信时才创建)
- 保持默认开启但修复shutdown顺序(风险更高)

Consequence: 修复前，所有进程都启动monitor线程(包括不需要的测试场景)，
shutdown时double free或hang。修复后，monitor默认关闭，需要时显式启用。

Transferable: 可选的监控/诊断功能应默认关闭(opt-in而非opt-out)。
启用时需验证所有生命周期路径(创建、正常关闭、异常关闭、进程被kill)的资源安全。


### 1.4 资源门控与SoC平台兼容

Ascend NPU有多个SoC代际(910A/910B/910_93/910_95等)，
不同代际的HCCL API支持范围、P2P能力、核控制能力各不相同。
门控逻辑的完整性直接决定跨平台兼容性。


#### Case 1.4.1: 函数指针被当作bool -- is_core_control_enabled缺括号

Commits: 08d567865 (fix), ce153f7cd (一致性修复)
Files: ProcessGroupHCCL.cpp
Defect ref: D-37, D-105

Scenario: 开发者在allreduce/broadcast/reduce等10+处集合通信调用前，
检查core control功能是否启用。

Constraints: `c10_npu::is_core_control_enabled`是函数而非变量。
C++中函数名不加括号时求值为函数指针(非null即true)。

Decision(D-37): 多处写成`if (c10_npu::is_core_control_enabled)`(缺括号)，
导致UseStreamResInCurrentThread()被无条件调用。旧CANN版本不支持该API -> segfault。
Fix: 所有处加()。
Decision(D-105): 即使括号修复后，gather/scatter/alltoall中调用模式仍不一致
(部分遗漏、部分无条件)。Fix: 统一门控模式。

Alternatives: 将is_core_control_enabled改为全局bool变量(消除遗漏括号的风险，
但失去运行时动态查询能力)。或用宏封装调用(CORE_CONTROL_GUARD)。

Consequence: D-37的segfault在旧CANN版本上100%触发。D-105的不一致在特定op组合下触发。
编译器`-Waddress`可检测函数指针用于bool上下文，但未启用。

Transferable:
- 条件判断中的函数名后必须有调用括号。启用-Waddress。
- 特性门控(feature guard)的调用模式在所有同类函数中必须保持一致。
  一个函数中修复了门控，必须横向检查所有同类函数。


#### Case 1.4.2: HCCL线程核数同步 -- fix-revert cycle #4

Commits: 92a86938a (fix), e32c44c98 (revert)
Files: ProcessGroupHCCL.cpp, ProcessGroupHCCL.hpp
Defect ref: D-144, D-151

Scenario: 单算子控核(core control)通过线程变量(TLS)传递核数配置。
计算算子执行前会刷新TLS，但HCCL算子的执行路径不经过该刷新逻辑。

Constraints:
- TLS更新路径由计算算子触发，HCCL算子是独立路径
- 当HCCL算子前没有计算算子时，TLS中的核数是上次缓存值
- 修复需要在collective和collectiveCoalesced中增加核数更新逻辑
- 但stream资源限制的设置可能与HCCL stream的生命周期冲突

Decision: Fix在collective中新增核数更新逻辑。但在master分支被revert(e32c44c98)。
revert原因: 修复引入了其他副作用(具体副作用未在commit message中详述)。

Alternatives:
- 在HCCL stream创建时设置一次核数(避免每次collective都检查)
- 将核数配置从TLS改为stream属性(根本解决TLS覆盖遗漏问题)
- 在op dispatch层统一刷新TLS(不区分计算/通信op)

Consequence: fix-revert cycle说明TLS状态管理与HCCL stream的交互极其复杂。
至今该问题可能仍未完全解决。

Transferable: TLS(线程局部存储)的更新路径必须覆盖所有消费方，不仅是最常见的路径。
当TLS被多个独立子系统(计算op、通信op)消费时，在消费点检查TLS freshness，
而非依赖"前面一定有人刷新过"的假设。


#### Case 1.4.3: P2P连接数限制 -- SoC代际守卫与off-by-one

Commits: 0bc2a875e (SoC守卫), 309c3ccab [+4 cherry-picks] (off-by-one)
Files: NPUPeerToPeerAccess.cpp
Defect ref: D-202, D-300

Scenario: Ascend 910A/910B有P2P连接数上限(C10_P2P_ACCESS_MAX_NPUS)，
但A3(910_93)+已取消此限制。开发者需要在代码中正确实施限制。

Constraints:
- SoC代际能力差异: 910A/910B有限制，A3+无限制
- 自身->自身不是远程P2P连接，不应计入远程连接计数
- 达到限制时需要准确标识是source_dev还是dest_dev受限

Decision(D-202): 原实现对所有SoC无条件限制。Fix: 包裹在SoC版本守卫中，A3+跳过。
Decision(D-300): device_enabled_count_初始化为1(包含自身)，但自身不是远程连接。
Fix: 初始化为0；分别检查source和dest限制。

Alternatives:
- 运行时查询SoC P2P能力(最准确但需要CANN API支持)
- 配置文件指定P2P限制(灵活但增加配置复杂度)

Consequence: D-202导致A3+上多卡P2P通信失败(false negative -- 不该限制却限制了)。
D-300的off-by-one导致实际可用连接数是MAX_NPUS-1=7而非8(计数语义错误)。

Transferable:
- 硬件限制相关代码需添加SoC版本守卫。新SoC引入时，系统性审查所有版本守卫。
- 资源计数的初始值必须与"计数什么"的语义一致。自身不是远程连接 -> 计数从0开始。
- 达到限制时的错误信息必须准确指向受限资源(哪个设备、什么限制)。


### 1.5 性能优化与回退

分布式通信的性能优化往往涉及减少同步点或降低内存开销。
但优化可能破坏正确性不变量。本节记录三个典型的"优化 vs 正确性"决策。


#### Case 1.5.1: alltoall的compatible mode OOM

Commits: a6f79de99 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-34

Scenario: alltoall的默认路径对所有输入做flatten+cat再传给HCCL。
compatible_mode应该在支持native alltoall_v的SoC上走原生路径(不flatten)。

Constraints:
- compatible mode基于send/recv模拟集合操作，仅部分SoC支持
- flatten+cat产生等size的临时内存，大规模alltoall时OOM
- is_compatible_soc检查缺失导致支持native的SoC也走flatten

Decision: 原实现compatible判断遗漏SoC能力检查。
Fix: compatible mode仅在`use_compatible_impl && is_compatible_soc`同时为真时启用。

Alternatives:
- 完全移除compatible mode(简化代码但失去旧SoC兼容)
- 在HCCL层透明处理(由HCCL SDK判断是否需要分解)

Consequence: 大规模分布式训练(大模型alltoall)直接OOM。
开发/测试阶段通常用小模型，不触发内存压力，因此遗漏。

Transferable: compatible/default路径选择必须包含硬件能力检查。
"compatible"是fallback路径，默认应走native路径，只在不支持时才fallback。


#### Case 1.5.2: shutdown同步粒度 -- device级 vs stream级

Commits: 0b9d27722 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-57

Scenario: shutdown()需要等待当前PG的所有通信完成后才能安全释放资源。

Constraints:
- npuSynchronizeDevice()同步整个设备(所有stream)
- 多PG共享设备时，device同步会阻塞其他PG和计算stream
- stream级同步只等待特定stream，粒度更细

Decision: 原实现用device级同步。Fix: 收集当前PG的HCCL stream(hcclStreams_)，
只同步这些stream。无stream时跳过。

Alternatives:
- event级同步(更细粒度但实现复杂)
- 不同步直接释放(不安全)

Consequence: device级同步在多PG场景造成全局停顿，严重时deadlock
(PG_A等PG_B的stream完成，PG_B等PG_A的stream完成)。

Transferable: 避免device级同步，优先stream/event级同步。
设备级同步是"全局锁"的NPU等价物 -- 简单但不可扩展。


#### Case 1.5.3: register_work的OOM -- 条件性资源注册

Commits: 5d3160fe2 (fix)
Files: ProcessGroupHCCL.cpp
Defect ref: D-41

Scenario: batch_isend_irecv等操作无条件调用c10d::register_work。
注册的work引用阻止tensor被释放。

Constraints:
- register_work是c10d框架功能(用于graph input追踪)
- 仅在allow_inflight_collective_as_graph_input()为true时需要
- 缺少对应的unregister_work调用

Decision: 原实现无条件注册，缺少unregister。Fix: 用feature flag守卫register，
在synchronize()/wait()中添加unregister。

Alternatives:
- 用weak reference注册(不阻止GC但需要alive检查)
- 在work完成回调中自动unregister(结构更clean)

Consequence: 多次调用后OOM。只在graph capture场景外(即大多数训练场景)浪费内存。

Transferable: register/unregister必须成对出现。条件性注册资源时，
unregister路径也必须有相同的条件守卫(或无条件unregister -- 对不存在的注册是no-op)。


### 1.6 异步执行中的资源安全

HCCL通信操作通过lambda闭包提交到TaskQueue异步执行。闭包捕获的变量、
闭包内操作的stream归属、闭包持有的引用强度，三者共同决定异步执行的正确性。
这一领域的错误模式高度一致: 在同步上下文中"正确"的代码，放入异步上下文后失效。


#### Case 1.6.1: alltoall VLA引用捕获 -- 经典UAF

Commits: 117c6a8fe [+2 cherry-picks]
Files: ProcessGroupHCCL.cpp
Defect ref: D-473

Scenario: 开发者实现alltoall_base/alltoall，需要将inputCounts/outputCounts等数组
传递给异步执行的HCCL lambda。

Constraints:
- inputCounts等4个数组声明为C风格VLA(variable-length array)，分配在栈上
- lambda以&引用捕获后推入TaskQueue异步执行
- 函数返回后栈帧销毁，VLA内存被回收

Decision: 原实现用引用捕获栈上VLA。函数返回后lambda访问悬空引用 -> UAF。
Fix: VLA改为std::vector<uint64_t>; lambda改为值捕获; 调用hcclAlltoAllV时用.data()。

Alternatives:
- 用std::array(编译期大小固定，不适用于变长场景)
- 在lambda内部分配(不需要外部捕获，但代码结构变化大)

Consequence: UAF导致通信数据量参数随机错误，HCCL可能读/写错误长度的数据。
多卡训练中表现为偶发数据损坏或segfault，极难复现(取决于栈上VLA内存是否被覆盖)。

Transferable:
- 异步lambda禁止引用捕获局部数组/VLA。clang-tidy有规则检测此模式。
- 更一般地: 任何推入队列/线程池/callback的lambda，默认用值捕获。
  引用捕获只在能证明被引用对象的生命周期覆盖lambda执行完成时才允许。


#### Case 1.6.2: recorded_inputs强引用延长tensor生命周期

Commits: 0555aef8b [+7 cherry-picks]
Files: ProcessGroupHCCL.cpp, ProcessGroupHCCL.hpp
Defect ref: D-679

Scenario: WorkHCCL对象需要追踪通信操作的输入tensor，用于eraseStream时检查storage存活性。
同一类中recorded_outputs_已使用weak_intrusive_ptr，但recorded_inputs_使用了强引用。

Constraints:
- multi-stream memory reuse模式下，Work对象持有到synchronizeInternal调用
- Work对象可能被长期持有(存放在future queue中)
- 强引用阻止allocator回收input tensor的显存

Decision: recorded_inputs_用c10::Storage(强引用)保存。上层释放tensor引用后，
storage引用计数不归零，显存无法回收。
Fix: 改为weak_intrusive_ptr<c10::StorageImpl>，eraseStream前用.lock()检查存活。

Alternatives:
- 在WorkHCCL::synchronize完成后立即清空recorded_inputs_(时序控制复杂)
- 不记录inputs，只依赖recordStream机制(失去eraseStream优化能力)

Consequence: 长时间训练中显存持续增长。outputs用weak ref但inputs用strong ref，
这种不一致性本身就是明显的review信号。

Transferable: 异步Work对象中保存的tensor/storage引用应默认使用weak_ptr。
当同一类中存在weak/strong混用时，review应要求解释为什么需要不同的引用强度。
与Case 1.1.2(eraseStream架构)构成同一演化链。


#### Case 1.6.3: allreduce pre/post handler在错误的stream上执行

Commits: 86d1674ce [+2 cherry-picks]
Files: ProcessGroupHCCL.cpp
Defect ref: D-459

Scenario: allreduce/reduce对bool/byte类型tensor需要预处理(cast到int)和后处理(cast回原dtype)。
这些cast操作在collective()的pre/post lambda中执行。

Constraints:
- HCCL通信在专用hcclStream上执行
- pre/post lambda默认在调用方的computation stream上执行
- cast操作的输出tensor是HCCL通信的输入/输出buffer

Decision: 原实现pre/post handler在computation stream上执行dtype cast。
HCCL通信可能使用未cast完成的buffer(pre未完成就开始通信)，
或后处理在通信完成前执行(post在错误时机执行)。
Fix: pre/post handler lambda中加NPUStreamGuard guard(hcclStreams[0])，
确保dtype cast在HCCL stream上执行。

Alternatives:
- 在collective框架层统一管理pre/post的stream上下文(更彻底)
- 在pre/post前后插入stream同步event(安全但增加同步开销)

Consequence: 多stream场景下，bool/byte类型的allreduce结果随机错误。
单stream或非bool/byte类型不触发(cast被跳过)。

Transferable: 分布式通信的pre/post处理必须在对应的通信stream上执行。
更一般地: 当操作A的输出是操作B的输入且A、B在不同stream时，
A必须在B的stream上执行(或通过event建立依赖)。


### 1.7 DDP集成

torch_npu需要在DDP(DistributedDataParallel)初始化后插入NPU设备同步，
并修正DDP内部假设(bucket view shape、reducer行为等)。
扩展方式的选择(子类化 vs monkey-patch)是这一领域的核心架构决策。


#### Case 1.7.1: DDP全局类替换破坏API兼容性 -- 子类化 vs monkey-patch

Commits: 434dc97e5 [+9 cherry-picks] (D-471, 子类化方案),
  00a50a5b0 [+2 cherry-picks] (D-464, 重构为monkey-patch)
Files: __init__.py, distributed/__init__.py, distributed/distributed.py, utils/module.py
Defect ref: D-471, D-464

Scenario: NPU环境下DDP.__init__完成后设备侧操作可能尚未完成(异步执行)，
CPU侧已继续运行。需要在DDP初始化末尾插入npu.synchronize()。

Constraints:
- 全局替换class会破坏isinstance检查(用户代码检查isinstance(model, DDP)可能失效)
- 第三方库可能import了原始DDP类引用
- 需要在所有DDP实例化路径上生效

Decision(D-471): 第一版创建NPU的DDP子类，monkey-patch替换全局DistributedDataParallel。
Decision(D-464): 发现全局类替换破坏兼容性后，重构为只patch __init__方法:
在apply_module_patch()中替换DDP.__init__为包装函数(调原始init后再synchronize)。
删除distributed.py子类文件。

Alternatives:
- 注册DDP hook(如果DDP支持post-init hook -- 当时不支持)
- 在NPU backend层自动同步(不依赖DDP改动，但同步粒度过粗)
- 使用__init_subclass__机制(不适用于已存在的基类)

Consequence: 子类化方案上线后收到兼容性报告，快速迭代为monkey-patch方案。
关键教训: 扩展框架类的行为时，方法级patch优于类级替换。

Transferable:
- 扩展PyTorch框架类: monkey-patch方法 > 子类替换 > 全局类替换。
  方法patch保持isinstance兼容性，类替换破坏它。
- 这是torch_npu整体monkey-patch策略的缩影(参见design-patterns.md Ch.3)。


#### Case 1.7.2: DDP bucket view未恢复梯度shape

Commits: 757fd9ca2 [+6 cherry-picks]
Files: reducer.cpp
Defect ref: D-468

Scenario: DDP的gradient_as_bucket_view优化将梯度设为bucket内存的view(避免拷贝)。
initialize_bucket_views需要正确设置bucket_views_in的shape。

Constraints:
- gradient_as_bucket_view=True时，梯度tensor应保持参数的原始shape
- bucket内存是flat的1D tensor，narrow操作产出也是1D
- 后续操作(梯度更新、optimizer step)依赖梯度shape与参数shape一致

Decision: 原实现无论gradient_as_bucket_view是否开启，都用flat narrow作为bucket_views_in。
开启时梯度shape变成1D，与参数shape不匹配。
Fix: gradient_as_bucket_view开启时，对narrow结果追加.view(v.sizes())恢复原始shape。

Alternatives:
- 在DDP reducer的finalize_backward中恢复shape(但此时shape信息已丢失)
- 使用as_strided替代view(处理non-contiguous情况，但引入更多复杂性)

Consequence: gradient_as_bucket_view=True时，optimizer因shape不匹配报错或产生错误结果。
GPU上PyTorch原生实现已正确处理，NPU移植时遗漏了这个view步骤。

Transferable: 从PyTorch上游移植DDP/reducer代码时，bucket view相关逻辑是高频遗漏区。
原因: 该逻辑只在gradient_as_bucket_view=True时触发，默认路径不覆盖。
Review rule: 移植reducer代码时，逐行对比上游的bucket_views_in/out初始化。


#### Case 1.7.3: blocking wait超时后不abort communicator

Commits: f82a64769 [+1 cherry-pick]
Files: ProcessGroupHCCL.cpp, ProcessGroupHCCL.hpp
Defect ref: D-447

Scenario: HCCL_BLOCKING_WAIT模式下，synchronize()需要等待通信完成。
超时后需要安全地退出并报告错误。

Constraints:
- 超时后HCCL communicator可能处于不一致状态(部分rank已完成，部分未完成)
- 不abort communicator直接抛异常 -> 后续通信操作在损坏的comm上执行 -> 未定义行为
- abort后comm不可复用，需要重建

Decision: 原实现超时后直接checkAndThrowException抛异常，未先abort communicator。
Fix: 拆为公开synchronize()和私有synchronizeInternal(timeout);
超时时先记录日志+break，有异常则先abort() communicator再handleException(TearDown)。

Alternatives:
- 超时后尝试等待更长时间(延迟问题但不解决)
- 超时后全局barrier确认所有rank状态(在超时场景下barrier本身可能hang)

Consequence: blocking wait超时后后续所有通信在损坏的communicator上执行，
导致不可预测的失败模式(hang、数据损坏、segfault)。

Transferable: 分布式通信的错误处理三步骤: (1)abort communicator (2)记录诊断信息
(3)抛异常/触发TearDown。跳过任何一步都会导致后续行为不确定。
与Case 1.3.1(析构异常处理)共同构成HCCL错误处理的完整模式。


### 1.8 通信域身份与数值语义

通信域身份判定和通信操作中的数值计算看似简单，但错误在这两个区域的密度说明:
即使是基础的"谁是全局group"和"通信多少个元素"，在分布式场景下也需要精确推敲。


#### Case 1.8.1: 全局PG身份判定 -- 创建顺序不是语义

Commits: c25460d5d [+6 cherry-picks]
Files: ProcessGroupHCCL.cpp, ProcessGroupHCCL.hpp
Defect ref: D-373

Scenario: 代码中5处逻辑需要判断当前ProcessGroup是否是全局group
(comm初始化、rank查询、析构清理等)。

Constraints:
- 原实现用static std::atomic<size_t> process_group_id递增计数器
- 假设uid_==0的ProcessGroup就是global group(第一个创建的)
- ranktable场景下子group可能先于global group创建

Decision: 原实现依赖创建顺序。ranktable场景下uid=0被分配给子group，5处逻辑全部失效。
Fix: 移除process_group_id静态计数器和uid_成员，
改用options_->global_ranks_in_group.empty()判断(global group的ranks列表为空)。

Alternatives:
- 传入显式的is_global标志(侵入式但明确)
- 在创建时注册到全局注册表，用名称查找(过度设计)
- 用rank数量判断(不准确，子group可能有相同rank数)

Consequence: ranktable场景下所有依赖全局PG判断的功能失效。
cherry-pick到6个版本分支(影响面广)。

Transferable: 不要用全局递增ID推断对象语义。ID只保证唯一性，不保证语义。
用对象自身属性(如ranks列表是否为空)判断身份。
更一般地: 任何依赖"第一个创建的对象是特殊的"假设的代码都是脆弱的。


#### Case 1.8.2: barrier/reduce的数值语义错误三连

Commits: dfd3398c6 (D-1253), bad33087f (D-1258), 933c0f2d6 (D-1267)
Files: ProcessGroupHCCL.cpp, distributed_c10d.py
Defect ref: D-1253, D-1258, D-1267

Scenario: 三个独立的数值语义错误，分别影响barrier、reduce、barrier:
(1) barrier的设备ID计算错误 (2) reduce的元素数计算错误 (3) barrier的tensor值和后端名称。

Constraints:
- (D-1253) 使用rank_ % numNPUs计算deviceIdx，对device6/7等高编号设备不正确
- (D-1258) physical_numel可能大于logical numel(因storage padding)
- (D-1267) at::empty({1})创建的barrier tensor可能被HCCL优化跳过

Decision:
(D-1253): 改为at::Device(NativeDeviceType)(不指定index)，让运行时使用当前设备。
(D-1258): 将physical_numel(input)替换为getNumelForHCCL(input)，对标准格式用logical numel。
(D-1267): 改at::empty为at::ones确保tensor有值; Backend.NCCL改为Backend.HCCL。

Alternatives:
- (D-1253) 维护显式的rank->device映射表(复杂但精确)
- (D-1258) 统一所有通信操作使用相同的numel计算函数(已有getNumelForHCCL但未在所有处使用)
- (D-1267) 在HCCL层禁止empty tensor优化(不可控)

Consequence:
- D-1253: device6/7无法barrier，多卡训练死锁
- D-1258: reduce操作读/写越界或不完整，数据损坏
- D-1267: barrier不生效(被跳过) + HCCL后端传device_ids报错

Transferable:
- 设备ID计算: 不假设设备编号连续或从0开始。优先用"当前设备"而非计算设备。
- numel: HCCL通信必须区分physical_numel和logical_numel，NPU format有padding。
  有统一的getNumelForHCCL时应全局使用，不允许直接调physical_numel。
- 从NCCL移植的代码: 全文搜索替换NCCL引用为HCCL，包括字符串常量和Backend枚举。


---

## Ch.2 内存管理与显存分配

NPUCachingAllocator是torch_npu中缺陷密度排名第二的子系统(仅次于ProcessGroupHCCL)。
核心难点: allocator的mutex、NPU设备同步、Python GIL、TaskQueue消费者线程
四者的交互产生了多种死锁模式。内存泄漏的根源则集中在引用计数的多余持有。


### 2.1 CachingAllocator并发与死锁

NPUCachingAllocator使用recursive_mutex保护内部状态。但持锁期间调用的操作
(设备同步、GIL释放、event查询)可能触发其他线程回锁，形成跨线程死锁。
这一领域产生了3个独立的死锁修复(D-325/D-1086, D-1062, D-879)和1个竞态修复(D-862)。


#### Case 2.1.1: mutex-sync-GC三方死锁与UnlockGuard

Commits: c4fd1d9df [+9 cherry-picks] (D-325), a09ce7fd1 [+4 cherry-picks] (D-1086)
Files: NPUCachingAllocator.cpp
Defect ref: D-325, D-1086

Scenario: DeviceCachingAllocator的malloc/garbage_collect/release_available_cached_blocks
在持有recursive_mutex时调用npuSynchronizeDevice等待任务队列排空。

Constraints:
- 同步操作等待TaskQueue清空 -> TaskQueue子线程中CANN TBE算子编译可能触发Python GC
- GC尝试释放cached block时需要同一个recursive_mutex
- recursive_mutex只防止同线程重入，跨线程行为与普通mutex一致
- 死锁链: 主线程持锁等子线程 -> 子线程GC等锁 -> 死锁

Decision: 引入UnlockGuard(RAII风格的临时解锁)。在持锁期间需要调npuSynchronizeDevice的
所有位置用UnlockGuard临时释放mutex，同步完成后重新获取。
emptyCache改为在加锁前先同步。release_cached_blocks删除内部同步调用(前置条件改为调用方负责)。

Alternatives:
- 用try_lock替代阻塞获取(不解决根本问题，只降低概率)
- 将allocator改为lock-free设计(改动面极大)
- 在GC回调中跳过allocator操作(可能导致内存泄漏)

Consequence: 死锁在长时间训练中随机触发(取决于TBE编译与GC的时序)。
D-325先发现并修复，D-1086补充了遗漏的代码路径。cherry-pick到9个版本分支(全版本影响)。

Transferable:
- 持有mutex期间禁止调用可能触发GC或跨线程同步的函数。
- UnlockGuard模式: 必须临时解锁时，用RAII保证"解锁->操作->重新加锁"的原子性。
  解锁期间共享状态可能变化，重新加锁后必须重新验证不变量。
- recursive_mutex不能防止跨线程死锁。它只解决同线程重入，跨线程场景与普通mutex等价。


#### Case 2.1.2: GIL-mutex交叉死锁 -- AB-BA经典模式

Commits: 34f04920f (D-1062), 0d27919b9 [+3] (D-879)
Files: NPUCachingAllocator.cpp, NPUQueue.cpp, OpCommand.cpp
Defect ref: D-1062, D-879

Scenario: NPUQueue满时需要释放GIL等待消费者。但释放GIL的时机和条件不正确，
导致GIL与allocator mutex形成AB-BA死锁。

Constraints:
- 线程A: 持有allocator mutex -> 队列满 -> 释放GIL等消费者
- 线程B: 获得GIL -> 触发GC -> GC析构tensor -> 需要allocator mutex -> 死锁
- MakeSureQueueEmpty中循环内反复acquire/release GIL也有时序窗口
- ACL线程执行算子时可能触发TBE编译(需要GIL)

Decision(D-1062): 引入全局标志g_used_aclop，GIL释放路径增加条件判断:
非aclop路径不需要释放GIL给TBE编译器，避免死锁。这是过渡方案(aclop即将废弃)。
同时修复aclrtSetCurrentContext的隐式设备切换副作用(保存/恢复device index)。
Decision(D-879): 将GIL释放提升到整个等待循环之前(PyEval_SaveThread)，
循环结束后恢复(PyEval_RestoreThread)。消除循环内反复acquire/release的时序窗口。

Alternatives:
- 将allocator操作移出GIL保护范围(需要Python/C++边界重新设计)
- 使用条件变量替代eventfd_read(减少阻塞点但不消除锁交叉)
- 在GC回调中检测死锁风险并延迟释放(实现复杂且不可靠)

Consequence: 训练过程中随机卡死(无错误信息)。死锁链涉及GIL+mutex+GC+ACL四层交互，
调试极其困难。D-879的修复(整循环释放GIL)比D-1062的修复(条件标志)更彻底。

Transferable:
- 持有非GIL锁时禁止释放GIL(AB-BA死锁)。反之亦然: 持有GIL时避免获取长期持有的mutex。
- 等待NPU异步操作完成的阻塞路径上，检查是否持有GIL。
- aclrtSetCurrentContext有隐式副作用(切换当前设备)，调用后必须恢复device index。


#### Case 2.1.3: event record-query竞态 + NPUQueue循环退出死锁

Commits: 96abe60a8 [+2] (D-862), bac7acf61 (D-1157)
Files: NPUCachingAllocator.cpp, NPUQueue.cpp
Defect ref: D-862, D-1157

Scenario: 两个并发相关的bug:
(D-862) process_events()在一个线程中查询event完成状态，但event的record在另一个线程执行。
(D-1157) MakeSureQueueEmpty的need_empty赋值放在错误的作用域层级。

Constraints:
- (D-862) TaskQueue模式下event record在独立线程执行，process_events在allocator线程
- 如果process_events先于record执行，aclrtQueryEvent查询未record的event -> 未定义行为
- (D-1157) need_empty = false在外层花括号后，内层循环结束后未重置 -> 外层while永远true

Decision(D-862): 新增recorded_events set(mutex保护)。event被record后加入set，
process_events在处理前检查event是否在set中。不在set中且Queue模式开启时跳过。
Decision(D-1157): 将need_empty赋值移入内层花括号内。单行修复。

Alternatives:
- (D-862) event创建时标记为"pending record"状态(状态机方案，更复杂但更完整)
- (D-862) 在push event到deque前先同步等待record完成(阻塞但安全)
- (D-1157) 重构为单层循环(消除作用域嵌套问题)

Consequence:
- D-862: 提前释放block(数据损坏)或event查询返回错误(crash)。race window很窄，常规测试难以复现。
- D-1157: MakeSureQueueEmpty永远不返回，主线程hang。

Transferable:
- 跨线程共享的event/state需要explicit synchronization。event的record和query不能假设执行顺序。
- 嵌套循环中的退出条件变量赋值必须仔细核对作用域层级。
  这种bug(单行赋值位置错误)在code review中极难发现，建议用单层循环+显式退出条件重构。


### 2.2 引用计数与资源泄漏

NPU内存泄漏的根本原因集中在三类: (1)额外的强引用阻止GC (2)释放函数未实现
(3)std::move误用导致资源转移失败。这些模式在code review中都有明确的检查规则。


#### Case 2.2.1: NPUTensorImpl双份storage引用 -- 额外的intrusive_ptr

Commits: 5fd5bc92b [+1] (D-544), f806f6daf [+1] (D-860)
Files: NPUTensorImpl.cpp/h, TensorFactories.cpp
Defect ref: D-544, D-860

Scenario: torch_npu自定义NPUTensorImpl，构造函数接收Storage(传给基类)
和额外的intrusive_ptr<StorageImpl> storage_impl(存入私有_storage_impl成员)。

Constraints:
- 基类TensorImpl::storage_已持有一个intrusive_ptr<StorageImpl>
- _storage_impl是第二个intrusive_ptr指向同一个StorageImpl -> 引用计数+1
- tensor.data创建shallow_copy时再传入_storage_impl -> 引用计数再+1
- 析构函数中_storage_impl.reset()在基类析构之前执行

Decision: 原实现中NPUTensorImpl同时持有基类和自身的两份storage引用。
正常析构路径: 基类-1, _storage_impl-1, 归零。
tensor.data路径: shallow_copy每次+1，多次.data调用累积额外引用 -> 泄漏。
Fix(D-544/D-860): 删除_storage_impl成员和构造参数，析构函数置空，
shallow_copy_from用基类impl.get()，统一make_tensor签名。

Alternatives:
- 将_storage_impl改为raw pointer(不影响引用计数，但生命周期管理困难)
- 将_storage_impl改为weak_ptr(允许GC但需要lock检查)

Consequence: 长时间训练中NPU显存持续增长但不crash(引用计数不归零 -> block不回收)。
正常场景"工作"(析构最终减到0)，但.data路径泄漏。

Transferable:
- 不要在子类中重复持有基类已管理的资源。intrusive_ptr的ownership应该唯一。
- 移植/扩展TensorImpl时，review所有intrusive_ptr成员是否与基类重复。
- "正常路径正确、edge case泄漏"的模式需要长时间训练才能发现。
  Review rule: 自定义TensorImpl的引用计数行为必须有UT覆盖.data/.clone/.detach路径。


#### Case 2.2.2: std::move在循环中误用 -- OOM observer仅device 0生效

Commits: 5621a5b76 [+4 cherry-picks]
Files: NPUCachingAllocator.cpp
Defect ref: D-1097

Scenario: NpuCachingAllocator::attachOutOfMemoryObserver遍历所有device_allocator，
为每个allocator注册OOM observer(用于触发OOM snapshot dump)。

Constraints:
- observer是std::function对象，通过std::move传入
- std::move后原对象处于moved-from状态(有效但未指定，通常为空)
- 循环第一次move后，后续迭代的observer为空

Decision: 循环体内对循环外变量使用std::move(observer)。
第一次move后observer被移空 -> 仅device 0能触发OOM snapshot。
Fix: std::move(observer)改为observer(按值传递拷贝)。

Alternatives:
- 最后一次迭代用move，其余用copy(优化但增加复杂性)
- 将observer存为shared_ptr，循环中传递shared_ptr(避免拷贝开销)

Consequence: 多卡训练中，仅device 0的OOM能触发snapshot dump。
其他设备OOM时无诊断信息，问题定位极其困难。cherry-pick到4个版本分支。

Transferable:
- 循环体内禁止对循环外变量使用std::move(除非是最后一次迭代的特殊处理)。
- std::move后的变量只能赋值或销毁，不能再读取。这是C++ code review的基本检查项。
- 编译器不会警告moved-from变量的后续使用(不是UB，但逻辑错误)。


#### Case 2.2.3: 资源释放缺失 -- 空deleter与shutdown顺序

Commits: 8b4c3e678 (D-70), 322588027 (D-65), b9ea48873 (D-209)
Files: NPUSwappedMemoryAllocator.cpp, CachingHostAllocator.cpp, InitNpuBindings.cpp
Defect ref: D-70, D-65, D-209

Scenario: 三个独立的资源释放缺陷:
(D-70) swapped memory的svm_deleter函数体为空，所有主机内存永不释放。
(D-65) Python退出时GC触发tensor析构，CachingHostAllocator调用已teardown的NPU runtime。
(D-209) shutdown流程中host allocator清缓存时，record_stream尝试操作已销毁的stream。

Constraints:
- (D-70) DataPtr的deleter是唯一的资源释放入口。空实现 = 永久泄漏
- (D-65) Python进程退出顺序: GC先于NPU runtime teardown(不确定)
- (D-209) shutdown顺序: HCCL释放 -> allocator清缓存 -> 但清缓存触发record_stream需要stream

Decision:
(D-70): 实现完整释放流程: null检查 -> memBlocks查找 -> sync -> Unregister -> Free -> erase。
(D-65): 用NpuSysCtrl::GetInitFlag()守卫insertEvents()。runtime已关闭时跳过event操作，
允许block在进程退出时安全泄漏。
(D-209): 在HCCL释放后、allocator清空前设host_finalize_flag_=true;
CachingHostAllocator::record_stream检查该flag，为true时跳过event创建。

Alternatives:
- (D-70) 在allocator销毁时统一释放所有block(兜底但不及时)
- (D-65/D-209) 在atexit中注册统一的shutdown序列，严格控制析构顺序
- 引入全局phase枚举(RUNNING/SHUTTING_DOWN/FINALIZED)，所有设备操作检查phase

Consequence:
- D-70: swapped memory场景主机内存持续增长，最终host OOM
- D-65: 进程退出时core dump(析构函数内异常 -> std::terminate)
- D-209: shutdown时crash(操作已销毁的stream)

Transferable:
- deleter/destructor/cleanup函数不允许为空(除非显式注释说明不需要清理)。
  空函数体在code review中应立即被标记。
- 进程退出/shutdown路径中所有设备API调用前必须检查runtime是否仍然存活。
  这是"最后的安全网": 正常路径可以假设runtime存活，析构/shutdown路径不能。
- 资源释放的依赖顺序应有明确的层级定义:
  通信资源 -> host allocator缓存 -> device allocator缓存 -> NPU runtime。
  每一层释放前设置finalize flag，后续层检查该flag。


### 2.3 Shutdown容错: 异常传播与初始化中毒

设备异常(硬件故障、reset)和初始化失败是NPU环境特有的runtime级故障。
与2.2的"正常shutdown路径"不同，本节关注的是"异常状态下的恢复与容错"。
核心矛盾: C++标准库的某些机制(如std::call_once)在异常场景下的行为是平台相关的，
不能假设"抛异常后一切可重试"。


#### Case 2.3.1: event query异常击穿批量清理循环

Commits: bedc8509b (D-183)
Files: CachingHostAllocator.cpp
Defect ref: D-183

Scenario: CachingHostAllocator的processEvents循环逐个query已完成的event，
以释放其关联的host内存block。当NPU设备故障或reset后，event->query()抛C++异常。
未捕获的异常终止整个循环，导致后续所有event永远不被清理，host内存分配逐渐阻塞。

Constraints:
- event->query()底层调用acl API，设备异常时返回错误码或抛异常
- processEvents在while循环中按FIFO顺序处理event队列
- 单个event失败不意味着后续event也会失败(不同stream/device的event独立)
- 循环中break/异常穿透 = 后续所有event被跳过

Decision: 用try/catch包裹event->query()。异常时log错误并将isEventCompleted=true，
释放该event关联的资源，循环继续处理下一个event。

    CachingHostAllocator.cpp:222
    try {
        isEventCompleted = event->query();
    } catch (...) {
        ASCEND_LOGE("processEvents() query event failed!");
        isEventCompleted = true;  // 视为已完成，释放资源
    }

Alternatives:
- 异常时break退出循环，将未处理的event留到下一轮(延迟释放但不泄漏)
- 在processEvents入口先检查设备健康状态，不健康则批量释放所有event
- event->query()内部捕获ACL错误码，返回bool而非抛异常(需改接口)

Consequence: 设备故障后host内存持续泄漏。训练任务不会立即crash，
但随着时间推移host OOM。故障注入测试才能复现，常规UT无法覆盖。

Transferable:
- 批量操作的循环体内，硬件/runtime调用必须有独立的异常处理。
  "一个失败 -> 全部放弃"是最常见的错误传播过度(over-propagation)模式。
- 对设备故障场景的策略: 宁可"误释放一个未完成的event"也不要"阻塞整个清理管线"。
  异常时将资源标记为"可释放"是合理的降级策略。


#### Case 2.3.2: std::call_once异常中毒 -- 平台相关的永久死锁

Commits: 6b1946bc5 [+5 cherry-picks] (D-491)
Files: NPUStream.cpp
Defect ref: D-491

Scenario: NPU stream全局初始化通过std::call_once(init_flag, initGlobalStreamState)执行。
当设备不可用时initGlobalStreamState抛异常。C++标准规定此时once_flag可被新线程重试，
但glibc旧版本的实现中，第二次调用同一once_flag会永久阻塞(pthread_once的bug)。
结果: 第一次初始化失败后，所有后续线程永远卡在call_once上。

Constraints:
- C++标准(17.6.5.5/3): call_once的callable抛异常时，once_flag不翻转，下一次调用可重入
- glibc旧版本(2.17及以下)不遵守此语义: once_flag被置为"进行中"但不重置
- NPU stream初始化是全局唯一操作，失败后必须可重试(设备热插拔、延迟就绪)
- 不能用static局部变量替代(同样依赖pthread_once)

Decision: 用int flag + std::mutex手写double-checked locking替代std::call_once:

    NPUStream.cpp:196
    if (initialize_flag == 0) {
        mtx.lock();
        if (initialize_flag == 0) {
            initGlobalStreamState();
            initialize_flag = 1;
        }
        mtx.unlock();
    }

异常时flag保持0，下次重入可重新初始化。

Alternatives:
- 升级glibc到修复版本(不可控，生产环境glibc版本由OS决定)
- 用std::atomic_flag + CAS自旋(正确但性能不如mutex在无竞争场景)
- initGlobalStreamState内部捕获所有异常，返回错误码而非抛出(改变接口契约)

Consequence: NPU设备暂时不可用时(如driver重启)，第一次初始化失败后
所有后续PyTorch操作永久hang。cherry-pick到5个版本分支。

Transferable:
- std::call_once在callable可能抛异常时是危险的。必须验证目标平台的glibc版本。
  安全替代: mutex + flag的double-checked locking，或者C++11 magic static
  (但magic static也依赖__cxa_guard_* -> pthread_once)。
- code review中所有std::call_once使用处应标注: "callable是否可能抛异常?"
  如果可能，需要平台验证或替换为手写同步。
- 注意: 上述fix中initialize_flag应为std::atomic<int>才是严格正确的。
  非atomic int在x86上碰巧可用(强内存模型)，但在ARM(昇腾的host CPU)上有风险。


### 2.4 NPUQueue: 无锁队列的生产者-消费者协议

NPUQueue是torch_npu异步执行的核心: 主线程(producer)将算子参数入队，
consumer线程从队列取出并提交给ACL runtime。这是一个单生产者-单消费者(SPSC)的
环形缓冲区，用eventfd做线程间通知。围绕它的缺陷集中在三个方面:
同步时序、枚举演进、调度优化回退。


#### Case 2.4.1: TTL轮询优化引发竞态 -- 被revert的调度策略

Commits: a182e63ee [+5 cherry-picks] (D-866), 9636cb900 [+5] (D-867)
Files: NPUQueue.cpp, NPUQueue.h
Defect ref: D-866, D-867

Scenario: 原始设计中consumer线程空闲时通过eventfd阻塞等待producer通知。
优化方案"Second-work flow"引入TTL(Time-To-Live)机制: consumer空闲时不阻塞在eventfd，
而是用clock_gettime + busy-wait轮询MAX_TTL_TIME(10ms)，超时后才让出CPU。
意图是减少eventfd唤醒延迟，提升小算子吞吐。但引入了竞态窗口。

Constraints:
- 原始协议: producer写入 -> eventfd_write通知 -> consumer eventfd_read唤醒 -> 读取
- TTL协议: producer写入 -> consumer可能在TTL轮询中 -> 不看eventfd -> 延迟响应
- 竞态窗口: producer完成write_idx更新的瞬间，consumer恰好在TTL计时逻辑中
  检查IsEmptyQueue()，看到非空但TTL未到期 -> 继续等待而非立即处理
- 无锁队列的正确性依赖于producer-consumer之间严格的通知-响应协议

Decision: 完整revert TTL机制，恢复eventfd阻塞+通知模式。
核心变更: 删除MAX_TTL_TIME/GET_MSEC宏，删除StartConsume中的TTL轮询循环，
恢复Dequeue中的eventfd_read阻塞等待，恢复Enqueue中的eventfd_write通知。

    // revert后的producer通知(Enqueue):
    while (!IsReadWorking()) {
        s = eventfd_write(efd_read, u);  // 唤醒consumer
        ...
    }

    // revert后的consumer等待(Dequeue):
    SetReadWorking(false);
    __sync_synchronize();
    if (IsEmptyQueue()) {
        s = eventfd_read(efd_read, &u);  // 阻塞等待通知
        ...
    }

Alternatives:
- 修复TTL方案的竞态(在TTL轮询中也检查eventfd，但增加复杂度)
- 用futex替代eventfd(更低延迟，但更难调试)
- adaptive策略: 连续N个空闲周期后从spin切换到eventfd阻塞

Consequence: TTL优化在压力测试中暴露吞吐下降或hang。
D-866和D-867是同一"Second-work flow"方案的两部分，被一起revert。
此后NPUQueue的调度策略保持eventfd方案至今，后续优化更为谨慎。

Transferable:
- 无锁队列的线程调度优化必须经过形式化或至少模型检查验证。
  "看起来更快"的busy-wait/TTL方案极易引入不可复现的竞态。
- producer-consumer之间的通知机制是协议的一部分，不能单方面优化。
  如果consumer不再无条件等待通知，producer的通知时机也必须同步调整。
- revert是正确的决策: 无锁队列的调度bug一旦上线，在生产负载下才暴露，
  且极难复现。正确性 > 性能。


#### Case 2.4.2: 数据写入与索引更新的可见性竞态

Commits: 180377bc3 [+4 cherry-picks] (D-914)
Files: NPUQueue.cpp, AsyncTaskQueueInterface.h, OpParamMaker.cpp
Defect ref: D-914

Scenario: 环形缓冲区的经典竞态: producer更新write_idx后，consumer立即看到队列非空
并开始读取，但此时数据(paramVal)可能尚未完全写入内存。
特别是queueLen==1时(队列中只有一个待处理项)，consumer读到的可能是半初始化的数据。

Constraints:
- ARM内存模型(昇腾host CPU): store-store和load-load不保证顺序
- write_idx更新和paramVal写入是两个独立的store操作，可被乱序
- compiler barrier(__sync_synchronize)在x86上是full fence，但语义依赖编译器实现
- consumer的ReadQueue在queueLen>0时直接调用manager().Call(datas, read_idx, queueLen)

Decision: 三层防御:
(1) producer侧: paramVal写入完成后加__sync_synchronize()，再设paramCopyFinished=1
(2) consumer侧: ReadQueue中加__sync_synchronize()确保read_idx的读取先于数据读取
(3) queueLen==1时加usleep(2)，给producer额外的写入时间窗口

    NPUQueue.cpp:275  (consumer ReadQueue)
    __sync_synchronize();
    uint32_t queueLen = (write_idx.idx - read_idx.idx + kQueueCapacity) % kQueueCapacity;
    if (queueLen == 1) {
        usleep(2);  // 等待producer完成数据写入
    }

    OpParamMaker.cpp:356  (producer CopyPara)
    __sync_synchronize();
    dstPtr->paramCopyFinished = 1;

Alternatives:
- 用std::atomic<>的acquire/release语义替代__sync_synchronize(更精确但需重构)
- 引入per-slot的ready flag(paramCopyFinished)，consumer自旋等待直到ready(已部分实现)
- 使用lock-free SPSC queue库(如folly::ProducerConsumerQueue)替代手写实现

Consequence: consumer读到半初始化的CopyParas，传给aclrtMemcpyAsync的参数错误，
导致设备侧DMA拷贝地址/长度错误 -> 静默数据损坏或segfault。
极难复现: 依赖producer和consumer的精确时序交错。

Transferable:
- 无锁队列的数据写入和索引更新之间必须有memory barrier。
  在ARM(昇腾host)上这不是优化而是正确性要求。
- usleep(2)是"承认同步机制不完整"的临时方案。正确做法是per-slot ready flag +
  consumer spin-wait。但在性能敏感路径上，usleep的延迟代价可接受。
- 同一commit还修复了另一个问题: CopyParas的默认kind=RESERVED枚举值被ACL删除。
  教训: 依赖外部库的枚举默认值时，该枚举的删除/重编号必须触发编译错误而非静默行为变化。


#### Case 2.4.3: 枚举分支遗漏 -- 新增类型未更新所有消费者

Commits: b70ee2d8f [+4 cherry-picks] (D-5)
Files: NPUQueue.cpp
Defect ref: D-5

Scenario: 新增ExecuteParasOpApiV2算子类型后，NPUQueue中两个消费该类型的函数
(get_func_error_msg和Enqueue的日志分支)未添加EXECUTE_OPAPI_V2的处理分支。
V2类型的算子名称被错误解析为event名称(落入else分支的EventParas cast)。

Constraints:
- NPUQueue的类型判断使用if-else if链而非switch(无编译器-Wswitch保护)
- EXECUTE_OPAPI_V2与EXECUTE_OPAPI的参数结构不同(V2用opName string*，V1用opType char*)
- 错误的static_cast不会crash(内存布局碰巧兼容)，但解析出错误的名称

Decision: 在两个函数的if-else链中增加EXECUTE_OPAPI_V2分支:

    NPUQueue.cpp:194
    } else if (type == c10_npu::queue::EXECUTE_OPAPI_V2) {
        auto cur_paras = static_cast<at_npu::native::ExecuteParasOpApiV2*>(queueParam->paramVal);
        auto op_name = cur_paras->opName;
        result << "the current working operator name is " << *op_name;
    }

Alternatives:
- 将if-else链重构为switch + -Wswitch-enum(编译器强制覆盖所有枚举值)
- 用虚函数多态替代类型判断(每种Paras类型实现自己的getName())
- 统一V1/V2的名称字段为相同类型(消除分支需求)

Consequence: dispatch log中V2算子的名称显示为乱码或event名称，
影响问题诊断。不影响计算正确性但严重降低可观测性。

Transferable:
- 新增枚举值后，必须grep所有消费该枚举的switch/if-else链。
  if-else链比switch更危险: switch + -Wswitch-enum可以编译期捕获遗漏，if-else不能。
- 类型分派超过3个分支时，应重构为switch或多态。
  if-else链的维护成本随分支数线性增长，遗漏概率也线性增长。


### 2.5 OOM诊断: 异步错误传播链与allocator策略

OOM(Out of Memory)是NPU训练最常见的运行时错误。torch_npu的OOM snapshot机制
在分配失败时dump内存快照供事后分析。但异步执行路径(通过NPUQueue)的OOM
无法触发该机制，因为错误信息在传播链中断裂。


#### Case 2.5.1: GE异步OOM不触发snapshot -- 错误消息传播链断裂

Commits: 44d2aad41 [+5 cherry-picks] (D-338), d7ef5286c (D-1096)
Files: NPUException.cpp, NPUQueue.cpp, NPUQueue.h, NPUStream.cpp, NPUStream.h,
  OptionsManager.cpp
Defect ref: D-338, D-1096

Scenario: OOM snapshot仅在NPUCachingAllocator的同步分配失败时触发。
但GE(Graph Engine)通过task queue异步执行时的OOM错误不会触发snapshot。
根因: c10_npu_get_error_message()获取ACL错误消息后直接返回给调用方，
未传递到NPUQueue/Repository层。Repository在ERROR_EXIT状态下直接抛异常，
不检查错误内容是否为OOM。

Constraints:
- 同步路径: allocator分配失败 -> 直接throw -> OOM observer捕获 -> dump snapshot
- 异步路径: ACL返回错误 -> c10_npu_get_error_message() -> 消息只在调用栈中传递
  -> Repository设ERROR_EXIT -> 抛"Inner error" -> 消息丢失，observer无法触发
- Repository没有存储错误消息的成员变量
- OOM判断依赖字符串匹配: strstr(errmsg, "Failed to allocate memory")

Decision: 建立完整的异步错误消息传播链:
(1) NPUException.cpp: c10_npu_get_error_message()获取errmsg后调setRepoErrMsg()
    广播到所有stream的Repository
(2) Repository增加error_msg成员 + Set/GetQueueErrMsg接口
(3) MakeSureQueueEmpty和Enqueue的ERROR_EXIT分支检查IsOomSnapshotEnable()
    并strstr判断是否为OOM，是则调用oom_observer()

    NPUException.cpp:103
    auto errmsg = c10_npu::acl::AclGetErrMsg();
    c10_npu::setRepoErrMsg(errmsg);  // 广播到所有repo
    return errmsg;

    NPUQueue.cpp:284 (MakeSureQueueEmpty ERROR_EXIT分支)
    if (c10_npu::option::OptionsManager::IsOomSnapshotEnable()) {
        auto errmsg = GetQueueErrMsg();
        if (strstr(errmsg, "Failed to allocate memory") != nullptr) {
            c10_npu::option::oom_observer();
        }
    }

Alternatives:
- 将OOM判断下沉到ACL错误处理层(所有错误统一分类，而非在consumer端字符串匹配)
- Repository使用error code枚举而非const char*(避免字符串匹配的脆弱性)
- 异步路径的错误直接调用同步路径的OOM handler(需要统一错误处理入口)

Consequence: 异步执行的GE OOM无诊断信息。用户只看到"Inner error"，
无法定位是哪个算子、哪块tensor消耗了最多显存。
cherry-pick到5个版本分支(D-338)，D-1096在后续版本中进一步完善。

注意: error_msg成员(const char*)未初始化。GetQueueErrMsg()在首次SetQueueErrMsg
前调用会返回未定义值。这是fix本身引入的新缺陷。

Transferable:
- 异步错误路径必须与同步路径有等价的诊断能力。
  "同步能dump，异步不能dump"是可观测性的严重盲区。
- 错误消息传播链应在架构设计时显式定义，而非事后补丁。
  本fix跨6个文件3层(异常处理->队列->流管理)正是缺少前置设计的代价。
- 新增C++成员变量必须初始化(尤其是raw指针)。
  const char* error_msg未初始化是典型的"fix引入新bug"。


#### Case 2.5.2: allocator padding策略特化引发回归

Commits: 9c205684a (D-116)
Files: NPUCachingAllocator.cpp, test_allocator_envs.py
Defect ref: D-116

Scenario: 为Ascend910_95(SoC version >= 260)引入特化: AddPadSize()返回0(不加padding)，
其他SoC返回32字节。意图是910_95不需要32字节对齐padding。
但该策略在某些分配场景下导致内存对齐问题或显存碎片。最终被rollback为全SoC统一加32字节。

Constraints:
- allocator的padding影响所有tensor分配的实际大小: (size + padding) / 512 * 512
- 910_95的硬件是否真的不需要padding未在commit中给出明确依据
- 测试用例中按SoC区分了期望值，但未覆盖所有分配路径(大/小/对齐边界)
- padding=0时某些场景下分配的block刚好在512边界，但DMA可能读越界(硬件行为)

Decision: 删除AddPadSize()条件分支，硬编码constexpr size_t kPadSize = 32:

    // round_size和uncached malloc两处统一:
    constexpr size_t kPadSize = 32;
    size = ((size + kPadSize) / 512 + 1) * 512;

测试也统一为(size + 32) / 512 + 1) * 512计算，不再按SoC区分。

Alternatives:
- 保留AddPadSize()但修复具体的对齐问题场景(需要明确的硬件规格)
- 按场景(而非SoC)决定padding: 通信buffer加padding，计算tensor不加
- 用runtime的aclrtGetMemAlignSize()动态查询对齐要求(如果ACL提供)

Consequence: 910_95设备上每个tensor多分配32字节，总显存浪费可忽略。
但padding为0时的具体失败场景未在commit中记录(只说"回滚")。

Transferable:
- allocator的对齐/padding策略变更需要全SoC + 全分配路径的压力测试。
  "某个SoC不需要padding"是硬件假设，需要硬件团队确认而非开发者推测。
- SoC特化是高风险操作: 每增加一个SoC分支，测试矩阵乘以N。
  除非有明确的性能/正确性收益且有硬件规格文档支撑，否则统一策略更安全。
- "rollback"本身说明原始变更缺少充分验证。allocator变更应有独立的压力测试gate。


## Ch.3 算子实现与注册

torch_npu的算子层负责将PyTorch ATen算子映射到NPU硬件。这一层的核心矛盾:
PyTorch upstream假设tensor是"逻辑视图 + 连续存储"，而NPU有私有format(NZ/ND/5HD等)
和独立的stride/contiguous语义。每当upstream改变tensor语义(如Preserve MemoryFormat、
Copy-on-Write、negative stride)，torch_npu的算子层必须同步适配，
否则会出现静默的语义偏差。

本章按开发者任务场景组织: "tensor构造"、"format/dtype转换"、"upstream适配"。


### 3.1 Tensor构造与stride语义保持

PyTorch的Preserve MemoryFormat语义要求clone/empty_like等操作保留源tensor的
stride布局。torch_npu的默认tensor构造函数(apply_tensor_without_format)总是
生成连续tensor，导致Preserve语义失效。这是一类反复出现的bug。


#### Case 3.1.1: clone丢失stride -- apply_tensor_without_format的语义陷阱

Commits: 5a798156e (D-9), a260b2448 (D-92)
Files: CloneKernelOpApi.cpp
Defect ref: D-9, D-92

Scenario: tensor.clone(memory_format=torch.preserve_format)对channels-last(NHWC) tensor
应返回同样strides的tensor。但NPU实现中OpPreparation::apply_tensor_without_format(src)
总是创建连续(NCHW) tensor作为输出。channels-last模型(如ResNet + AMP)的中间结果
在clone后退化为NCHW，触发大量隐式format转换，训练性能下降30%+。

Constraints:
- apply_tensor_without_format是torch_npu算子的通用输出tensor创建函数
- 该函数内部用at::empty(sizes, options)创建tensor，不接受strides参数
- Preserve模式的语义: 输出tensor的stride layout应与输入"等价"(不要求完全相同)
- 只有non_overlapping_and_dense的tensor可以安全保留strides

Decision: Preserve模式下用at::empty_strided替代apply_tensor_without_format:

    CloneKernelOpApi.cpp
    if (memory_format == c10::MemoryFormat::Preserve
        && src.is_non_overlapping_and_dense()) {
        self = at::empty_strided_symint(  // D-9 用 symint 版本
            src.sym_sizes(), src.sym_strides(), src.options());
    } else {
        self = OpPreparation::apply_tensor_without_format(src);
    }

D-9和D-92是同一fix在不同版本分支的提交(D-9用sym_sizes/sym_strides，D-92用non-sym版本)。

Alternatives:
- 修改apply_tensor_without_format使其接受optional<strides>参数(影响面大)
- 在OpPreparation中增加apply_tensor_with_strides专用函数
- 在npu_format_cast输出侧做stride补偿(下游修复，不治本)

Consequence: channels-last训练场景性能严重退化(隐式format转换)。
用户看到的是"NPU比预期慢"而非明确错误。需要profiler才能定位到clone是瓶颈。

Transferable:
- apply_tensor_without_format是torch_npu的"默认选择"但不保留strides。
  凡是需要保留stride语义的算子(clone/empty_like/contiguous)都不能用它。
  审查规则: 搜索所有MemoryFormat::Preserve分支，检查输出tensor的创建方式。
- upstream的Preserve语义在NPU上必须显式实现，不会"自动继承"。
  每次upstream强化MemoryFormat语义时，torch_npu需要逐算子排查。


#### Case 3.1.2: empty_like丢失stride -- 另一个"默认连续"陷阱

Commits: 32033f92d (D-43)
Files: TensorFactories.cpp
Defect ref: D-43

Scenario: empty_like(src, memory_format=preserve_format)应返回与src相同stride布局的空tensor。
NPU实现中ApplyTensorWithFormat(self.sizes(), options, npu_format)只传入sizes，
输出tensor总是默认连续布局。与Case 3.1.1同一类问题，但发生在不同的代码路径。

Constraints:
- empty_like同时要处理NPU私有format(NZ/5HD)和standard format(ND)
- 只有base format(ND)时才能用at::empty_strided保留strides
- 非base format时tensor的物理布局由format决定，stride不可自定义
- support_as_strided()检查: NPU的某些format不支持as_strided视图操作

Decision: 分支处理: base format + strided layout + Preserve模式 → 用at::empty_strided;
其他情况 → 保持原ApplyTensorWithFormat:

    TensorFactories.cpp
    if (FormatHelper::IsBaseFormatType(npu_format)
        && self.unsafeGetTensorImpl()->support_as_strided()
        && self.layout() == c10::kStrided
        && (!optional_memory_format.has_value()
            || optional_memory_format.value() == c10::MemoryFormat::Preserve)) {
        std::vector<int64_t> strides = at::infer_dense_strides(self.sizes(), self.strides());
        result = at::empty_strided(self.sizes(), strides, options.memory_format(std::nullopt));
    } else {
        result = OpPreparation::ApplyTensorWithFormat(self.sizes(), options, npu_format);
    }

注意: 用at::infer_dense_strides推导而非直接复制src.strides()。
infer_dense_strides保证输出是密集的(无gap)，即使src有非标准stride。

Alternatives:
- 统一在ApplyTensorWithFormat内部处理stride保留(修改基础设施)
- 只在ChannelsLast/ChannelsLast3d时保留stride，其他format不处理(不完整)
- 在算子输出后补一次as_strided调整stride(额外开销)

Consequence: 与Case 3.1.1相同的性能退化。empty_like是PyTorch中最高频的tensor工厂函数，
影响范围比clone更广。

Transferable:
- torch_npu的tensor创建有两类函数: format-aware(ApplyTensorWithFormat)和
  format-free(apply_tensor_without_format)。两者都不保留stride。
  保留stride必须显式使用at::empty_strided。
- IsBaseFormatType是关键判断: 只有base format(ND)的tensor才能自由操作stride。
  NPU私有format(NZ/5HD/FZ等)的stride由format隐式决定，不能覆盖。


#### Case 3.1.3: as_strided自定义实现偏离upstream语义

Commits: 7492ad614 (D-93)
Files: ResizeNpu.h, TensorShape.cpp
Defect ref: D-93

Scenario: torch_npu重写了as_strided的底层实现setStrided(约60行代码)，
替换PyTorch原生的at::native::setStrided。自定义版本有两个语义偏差:
(1) overflow检测不完整(未覆盖所有边界条件)
(2) size/stride与当前值相同时提前return，跳过了update_contiguous()调用。
后者导致tensor的contiguous flag过期: 修改了storage但size/stride未变时，
is_contiguous()返回stale结果。

Constraints:
- upstream的setStrided在每次调用时都重新计算contiguous flag
- torch_npu的"优化": 相同size/stride时跳过更新(避免重新计算)
- 但contiguous flag还依赖storage_offset，不只是size/stride
- 自定义checkInBoundsForStorage的overflow检查漏了负数offset场景

Decision: 删除约60行自定义代码，直接调用at::native::setStrided:

    TensorShape.cpp
    // 删除自定义 setStrided 和 checkInBoundsForStorage
    at::native::setStrided(result, size, stride, storage_offset);

Alternatives:
- 修复自定义版本的两个bug(但维护成本 > 收益)
- 在NPU侧wrap upstream setStrided，仅在前后加NPU-specific逻辑

Consequence: as_strided操作后tensor的is_contiguous()返回错误结果，
下游算子基于stale flag做优化选择(如跳过不必要的copy)，导致计算结果错误。
这是静默的数据正确性问题。

Transferable:
- 不要重写upstream的基础设施函数，除非有不可替代的理由。
  "优化"(跳过contiguous重算)引入了语义偏差，且upstream演进时不会同步更新NPU版本。
- 凡是torch_npu中自定义的upstream替代函数，应建立同步检查机制:
  upstream版本变更时grep所有替代点。


#### Case 3.1.4: copy_对negative stride视图的判断不完整

Commits: d0a18e3e8 (D-123)
Files: CopyKernelOpApi.cpp
Defect ref: D-123

Scenario: tensor.neg()创建一个negative stride视图(通过TensorIterator的is_neg标志)。
copy_(self, src)在src是neg视图时，需要在拷贝后对self做neg_()补偿。
但原实现只检查src.is_neg()，遗漏了self.is_neg()的情况。
当self和src的neg状态不一致时才需要补偿。

Constraints:
- is_neg()是PyTorch的视图标志，表示该tensor是另一个tensor的negation视图
- copy_的语义: self的值应等于src的值(考虑所有视图标志)
- 补偿逻辑: 当且仅当self和src的neg状态不同时需要neg_()
- 原条件if(src.is_neg())在self也是neg时会错误地双重取反

Decision: 修正条件为XOR语义:

    CopyKernelOpApi.cpp
    if (self.is_neg() != src.is_neg()) {
        self.neg_();
    }

Alternatives:
- 在copy_入口统一materialize所有视图标志(消除视图，但可能引入额外拷贝)
- 用TensorIterator处理所有视图标志(upstream的做法，但NPU的copy_绕过了TensorIterator)

Consequence: copy_在neg视图间的结果符号错误。影响所有涉及neg视图的高阶操作。

Transferable:
- PyTorch的视图标志(is_neg, is_conj)是copy_/clone等操作必须处理的"隐藏状态"。
  NPU的自定义copy_绕过TensorIterator时，必须手动处理这些标志。
- XOR是正确的视图补偿逻辑: 只有"状态不一致"才需要补偿。
  单方检查(只看src)是不完整的。


### 3.2 Format与dtype转换

NPU的format转换(ND<->NZ/5HD)和dtype转换(npu_dtype_cast)是自定义算子，
不存在于upstream PyTorch中。这类算子的缺陷集中在"接口规范"和"资源管理"两方面。


#### Case 3.2.1: npu_format_cast参数声明与调用方式不一致

Commits: 5294f5ffe (D-27)
Files: wrapper_onnx_ops.py
Defect ref: D-27

Scenario: npu_format_cast(self, acl_format, customize_dtype=None)的ONNX wrapper
中customize_dtype未声明为keyword-only参数(缺少*分隔符)。
同时_NPUFormatCastOP.forward用*args/**kwargs透传，ONNX export时无法正确
trace参数绑定。

Constraints:
- Python 3的keyword-only参数需要*分隔符: def f(a, b, *, kw=None)
- ONNX export通过torch.autograd.Function.forward的签名推断参数
- *args/**kwargs透传使ONNX tracer无法确定参数数量和类型
- customize_dtype作为positional参数时，用户可能意外传入错误类型

Decision: (1) wrapper签名加*分隔符; (2) forward从透传改为显式声明:

    wrapper_onnx_ops.py
    def _wrapper_npu_format_cast(self, acl_format, *, customize_dtype=None):
        ...

    class _NPUFormatCastOP(torch.autograd.Function):
        def forward(ctx, self, acl_format, customize_dtype=None):
            return torch.ops.npu.npu_format_cast(
                self, acl_format, customize_dtype=customize_dtype)

Alternatives:
- 只加*分隔符不改forward(部分修复)
- 移除customize_dtype参数，在npu_format_cast内部推断dtype
- 用functools.wraps确保wrapper签名与底层C++函数一致

Consequence: ONNX export npu_format_cast时参数绑定错误，导出失败或模型语义错误。

Transferable:
- 自定义算子的Python wrapper必须与C++注册的schema参数类型、顺序、keyword-only
  完全一致。schema变更时wrapper必须同步更新。
- torch.autograd.Function.forward不要用*args/**kwargs。
  显式声明每个参数使ONNX/JIT tracer能正确推断。


#### Case 3.2.2: npu_dtype_cast缺少分布式tensor的策略注册

Commits: 6b4394105 (D-94)
Files: _pointwise_ops.py(新建), dtensor.py, distributed/tensor/__init__.py
Defect ref: D-94

Scenario: npu_dtype_cast是NPU自定义的dtype转换算子。当在DTensor(分布式tensor)上
调用时，PyTorch的DTensor dispatch需要查找该算子的分片策略(sharding strategy)。
npu_dtype_cast/npu_dtype_cast_backward未注册pointwise_strategy，
导致DTensor场景下dispatch失败(NotImplementedError)。

Constraints:
- PyTorch DTensor框架对每个算子需要注册分片策略(如何在多卡间分片计算)
- dtype_cast是element-wise操作，对应pointwise策略(每个元素独立转换)
- linearity=0: 输出的sharding与输入相同(dtype_cast不改变元素位置)
- 需要同时注册forward(npu_dtype_cast)和backward(npu_dtype_cast_backward)

Decision: 新建_pointwise_ops.py，为4个op注册pointwise_strategy:

    _pointwise_ops.py
    custom_pointwise_ops = {
        npu.npu_dtype_cast.default: 0,      # linearity=0
        npu._npu_dtype_cast.default: 0,
        npu.npu_dtype_cast_backward.default: 0,
        npu._npu_dtype_cast_backward.default: 0,
    }
    for op in custom_pointwise_ops:
        register_op_strategy(op, ...)(custom_pointwise_strategy)

Alternatives:
- 将dtype_cast注册到通用pointwise列表中(但需区分linearity参数)
- 在C++侧注册CompositeExplicitAutograd，让DTensor自动推断策略
- 不支持DTensor + dtype_cast组合(文档标注limitation)

Consequence: FSDP/DTensor训练中调用npu_dtype_cast时直接报错。
影响所有使用DTensor并需要混合精度的训练任务。

Transferable:
- 自定义算子(custom op)在新框架(DTensor/FX/TorchDynamo)中需要额外注册。
  C++ kernel注册不够，Python dispatch层的策略也必须覆盖。
- element-wise的自定义算子 → pointwise_strategy + linearity=0 是标准模式。
  审查清单: 新增custom op后，检查DTensor/FX/ONNX三个dispatch路径是否覆盖。


#### Case 3.2.3: aclCreateTensor返回的host侧描述符泄漏

Commits: c4f4b33a9 (D-122)
Files: FormatCastKernelNpu.cpp
Defect ref: D-122

Scenario: format转换调用链中ConvertType(src)将at::Tensor转换为aclTensor*描述符，
传给GetFormat()查询目标format。aclTensor*是ACL在host侧分配的元数据对象，
使用后必须Release。但代码中GetFormat(ConvertType(src), ...)直接在参数中调用
ConvertType，返回的指针无变量持有，无法释放。

Constraints:
- aclTensor*是ACL的不透明指针，内部含shape/dtype/format等元数据(host内存)
- 每次ConvertType调用分配约200-500字节host内存
- format转换在训练中每步每层都会调用(算子调度前的format检查)
- 不释放 → host内存泄漏，长时间训练后OOM

Decision: 将ConvertType返回值存入局部变量，使用后Release:

    FormatCastKernelNpu.cpp
    auto acl_src = ConvertType(src);
    auto api_ret = GetFormat(acl_src, acl_format, ...);
    Release(acl_src);  // 释放host侧描述符

两处分支(aclnnOnly和JitDisable)都做了相同修复。

Alternatives:
- 用RAII包装aclTensor*(如unique_ptr<aclTensor, decltype(&Release)>)
- 在GetFormat内部负责Release(改变所有权语义)
- ConvertType返回RAII wrapper而非裸指针

Consequence: 每步训练泄漏数百字节host内存。10万步后泄漏数十MB。
大模型训练中(百万步)可导致host OOM。不影响计算结果。

Transferable:
- ACL的Create/Convert系列API返回的指针都需要对应的Release/Destroy。
  在函数参数中直接调用Create类函数(如GetFormat(ConvertType(x), ...))是必然泄漏。
- 审查规则: grep所有aclCreate*/Convert*调用，检查返回值是否有对应的Release/Destroy。
  理想方案: 统一用RAII wrapper(类似std::unique_ptr)管理ACL对象生命周期。


### 3.3 Upstream特性适配: Copy-on-Write

PyTorch upstream不断引入新的tensor语义特性。torch_npu必须逐一适配，
否则新特性会在NPU设备上产生crash或静默错误。
COW(Copy-on-Write)是upstream在v2.1引入的lazy_clone机制。


#### Case 3.3.1: COW lazy_clone的NPU适配 -- allocator copy_data路径

Commits: f4e5747fb (D-67)
Files: TensorFactories.cpp, NPUStorageImpl.cpp, NPUCachingAllocator.cpp,
  NPUSwappedMemoryAllocator.cpp, NPUWorkspaceAllocator.cpp
Defect ref: D-67

Scenario: PyTorch的lazy_clone创建共享storage的tensor，写入时才copy(COW语义)。
该机制调用c10::impl::cow::lazy_clone_storage创建共享存储，然后在写入时
通过allocator的copy_data回调执行实际拷贝。
torch_npu的三个allocator(Caching/Swapped/Workspace)的copy_data使用
default_copy_data(即CPU memcpy)，但NPU tensor的数据在device上，
memcpy拷贝的是host侧的无效地址 → segfault。

Constraints:
- COW的copy_data由allocator提供，在tensor被写入时自动调用
- default_copy_data使用std::memcpy，只适用于host内存
- NPU tensor的DataPtr指向device内存，不能直接memcpy
- make_npu_storage_impl对size==0的StorageImpl要求data_ptr==nullptr，
  但COW场景下size==0的共享storage可能有非null data_ptr

Decision: 五处修改:
(1) 新增_lazy_clone实现: cow::lazy_clone_storage + CopyDesc同步NPU描述符
(2) 三个allocator的copy_data: default_copy_data → aclrtMemcpy(D2D)
(3) make_npu_storage_impl放宽size==0时的data_ptr检查

    NPUCachingAllocator.cpp (三个allocator相同修改)
    // before:
    default_copy_data(dest, src, count);
    // after:
    NPU_CHECK_ERROR(aclrtMemcpy(dest, count, src, count, ACL_MEMCPY_DEVICE_TO_DEVICE));

    NPUStorageImpl.cpp (放宽检查)
    // before:
    if (size == 0) {
        TORCH_CHECK(data_ptr == nullptr, "size=0时data_ptr必须为null");
    }
    // after:
    if (data_ptr == nullptr && size > 0) {
        // 分配device内存...
    }

Alternatives:
- 在NPU设备上禁用COW(注册一个reject-all的cow hook)
- 在COW层增加设备感知: copy_data前检查设备类型，自动选择memcpy/aclrtMemcpy
- 让PyTorch upstream的cow::lazy_clone_storage支持device回调(需upstream PR)

Consequence: 使用lazy_clone的代码(如Tensor.clone()的某些优化路径)在NPU上segfault。
影响所有使用PyTorch >= 2.1 + torch_npu的场景。

Transferable:
- upstream引入的新tensor语义(COW/NEX/vmap)都可能触及allocator回调。
  torch_npu自定义的allocator必须实现所有回调(不能依赖default实现)。
- 审查清单: 每次PyTorch升级后，diff allocator.h的虚函数列表，
  检查新增回调是否有NPU适配。default实现通常假设host内存。


### 3.4 二元算子的broadcast与dtype提升

PyTorch二元算子的标准契约:
(1) 输出shape = broadcast(self.shape, other.shape)
(2) 输出dtype = result_type(self, other)
(3) _out变体需要CheckOut校验输出tensor的shape/dtype合法性

torch_npu的算子实现经常遗漏这三步中的一到两步，因为底层CANN kernel
可能不做shape/dtype校验(直接crash或返回静默错误结果)。


#### Case 3.4.1: logaddexp缺broadcast shape推断与dtype提升

Commits: 37e48d896 (D-562)
Files: LogAddExpKernelNpuOpApi.cpp, LogAddExp2KernelNpuOpApi.cpp
Defect ref: D-562

Scenario: logaddexp(a, b)和logaddexp2(a, b)是逐元素二元算子。
当self和other的shape不同时(如[3,1]和[1,4])应做broadcast得到[3,4]。
当dtype不同时(如float32和float64)应提升到公共类型。
torch_npu的实现两个都没做。

Constraints:
- _out变体接收用户预分配的output tensor，必须校验其shape/dtype与broadcast结果匹配
- 非_out变体需要自行分配output，dtype应为result_type(self, other)
- 原实现直接用self.options()分配输出，dtype始终跟随self而非推断公共类型
- broadcast shape计算在torch_npu中有现成工具: broadcast_ops_npu_output_size

Decision: 四处修改(logaddexp + logaddexp2各两处):

    // _out变体: 增加shape校验
    auto output_size = broadcast_ops_npu_output_size(self, other);
    OpPreparation::CheckOut({self}, out, out, output_size);
    EXEC_NPU_CMD(aclnnLogAddExp, self, other, out);

    // 非_out变体: 用result_type推断dtype
    auto output_size = broadcast_ops_npu_output_size(self, other);
    at::ScalarType output_type = at::native::result_type(self, other);
    at::Tensor result = OpPreparation::ApplyTensorWithoutFormat(
        output_size, self.options().dtype(output_type));

Alternatives:
- 在CANN kernel侧做broadcast(性能更好但依赖kernel版本)
- 统一用TensorIterator处理broadcast+dtype(upstream方式，但torch_npu直调aclnn)
- 只做broadcast不做dtype提升(不完整修复)

Consequence: 不同shape/dtype的tensor做logaddexp时，结果shape错误或dtype被截断。
float64 + float32的精度损失是静默的，用户不会收到任何warning。

Transferable:
- torch_npu二元算子实现的标准清单:
  (1) broadcast_ops_npu_output_size(self, other)
  (2) at::native::result_type(self, other)
  (3) _out变体调用OpPreparation::CheckOut
  三步缺一就是bug。审查规则: grep所有二元算子实现，检查是否同时包含这三步。
- self.options()只反映self的dtype，不是公共dtype。
  正确写法: self.options().dtype(result_type(self, other))。


### 3.5 fix-revert cycle: memory layout变更的连锁反应

memory layout(stride/contiguous/format)是torch_npu中风险最高的修改领域。
一个看似正确的stride修复可能破坏下游算子的隐式layout假设。
本节记录两个典型的"修复正确但引发回归"场景。


#### Case 3.5.1: clone stride保持 → matmul精度回归 → 完整revert

Commits: 5a798156e (D-9 original fix), cebf05bcc (D-38 revert on master),
  ccbbde6d8 (D-38 revert on v2.11.0)
Files: CloneKernelOpApi.cpp
Defect ref: D-38 (revert of D-9)
Related: Case 3.1.1

Scenario: Case 3.1.1记录了D-9的修复: clone在Preserve模式下用empty_strided
保留stride。修复逻辑是正确的(对齐upstream语义)。但合入后matmul冒烟精度测试失败。

根本矛盾: torch_npu的matmul(及其他计算密集算子)在底层隐式依赖输入tensor是连续的。
clone返回非连续tensor后，matmul收到意外的stride layout，CANN kernel的计算路径
选择了错误的分支，精度偏差超过阈值。

Constraints:
- D-9的修复逻辑本身正确: upstream语义要求Preserve保留stride
- matmul的精度依赖输入的内存布局(连续 vs 非连续走不同CANN kernel路径)
- 精度回归只能通过端到端冒烟测试发现(单元测试只验证shape/dtype)
- 修复影响所有使用clone()的下游路径，不能只针对matmul打补丁

Decision: 完整revert D-9修改，clone恢复为无条件apply_tensor_without_format返回连续tensor:

    CloneKernelOpApi.cpp
    // D-9 (reverted):
    if (memory_format == Preserve && src.is_non_overlapping_and_dense()) {
        self = at::empty_strided_symint(src.sym_sizes(), src.sym_strides(), ...);
    } else {
        self = OpPreparation::apply_tensor_without_format(src);
    }

    // D-38 (revert → 回到原始):
    auto baseSelf = OpPreparation::apply_tensor_without_format(src);
    baseSelf.copy_(src);

revert在master和v2.11.0两个分支各做了一次(cebf05bcc, ccbbde6d8)。

Alternatives:
- 只revert Preserve分支，保留其他format模式的修复(风险: 可能还有其他隐式依赖)
- 在matmul入口加contiguous()强制转换(治标不治本，且影响性能)
- 先修复matmul对非连续输入的处理，再重新合入D-9(正确但工程量大)

Consequence: 这是torch_npu第二个layout相关的fix-revert cycle(第一个是Ch.2.4.1的TTL优化)。
NPU的算子实现链中存在大量对"输入是连续的"的隐式假设。在这些假设被系统性清理之前，
任何改变tensor layout的修复都是高风险操作。

Transferable:
- 涉及tensor memory layout变更的fix，必须运行全量精度回归测试(不能只跑UT)。
  冒烟测试应覆盖matmul/conv/attention等计算密集算子。
- fix-revert cycle的模式: 修复A在语义上正确 → 触发B的隐式假设 → revert A。
  真正的解决方案是修B(消除隐式假设)，但工程量决定了短期只能revert A。
  Case 3.1.1与本case合起来展示了这个矛盾的全貌。


#### Case 3.5.2: copy路径中empty_like的memory_format假设

Commits: c33d1b59d (D-15), b0d7dcb30 [cherry-pick]
Files: CopyKernel.cpp, CopyKernelOpApi.cpp, test/npu/test_copy.py (new)
Defect ref: D-15

Scenario: copy_h2d_baseformat()和copy_d2h_baseformat()在非连续dst tensor上
做host-device拷贝。步骤: 用empty_like(dst)创建连续临时buffer → aclrtMemcpy到buffer →
copy回dst。问题: upstream修改了empty_like的默认行为，使其默认保留src的stride(Preserve)
而非总是返回连续tensor。修改后empty_like(非连续dst)返回非连续临时buffer，
aclrtMemcpy期望连续内存但拿到非连续地址 → 拷贝结果静默错误。

Constraints:
- aclrtMemcpy是ACL的底层D2H/H2D拷贝接口，要求源和目标都是连续内存
- empty_like(x)的默认memory_format语义经历过upstream变更:
  早期 → 总是连续; 后来 → Preserve(跟随x的format)
- 调用方(copy_h2d/copy_d2h)隐式依赖empty_like返回连续tensor
- 四处调用: CopyKernel.cpp的h2d+d2h，CopyKernelOpApi.cpp的h2d+d2h

Decision: 在empty_like调用中显式传入LEGACY_CONTIGUOUS_MEMORY_FORMAT:

    CopyKernel.cpp (h2d和d2h各一处)
    // before:
    at::Tensor contiguous_dst = at::empty_like(dst);
    // after:
    at::Tensor contiguous_dst = at::empty_like(
        dst, dst.options(),
        at::MemoryFormat::Contiguous);  // LEGACY_CONTIGUOUS_MEMORY_FORMAT

CopyKernelOpApi.cpp中两处做相同修改。+138行copy测试覆盖11个场景(连续/非连续、
不同dtype、切片、广播、排列)。

Alternatives:
- 用at::empty(dst.sizes(), dst.options())替代empty_like(显式但失去_like的语义一致性)
- 在copy路径入口强制dst.contiguous()(改变用户看到的语义)
- 包装一个always_contiguous_empty_like工具函数(增加抽象但防止同类bug复发)

Consequence: 非连续tensor的H2D/D2H拷贝静默产生错误数据。
影响: 任何对slice/transpose/permute后的tensor做.cpu()或.npu()的场景。
用户看到的是"结果不对"但没有错误消息。

Transferable:
- empty_like/zeros_like/ones_like等_like API的默认memory_format语义可能被upstream改变。
  调用方如果依赖输出是连续的，必须显式传LEGACY_CONTIGUOUS_MEMORY_FORMAT。
  审查规则: grep所有empty_like调用，检查是否有隐式连续性假设。
- 与Case 3.5.1形成对照: 3.5.1是"改了layout导致下游出错"(主动变更引发回归)，
  3.5.2是"upstream改了语义导致调用方出错"(被动接受语义变更)。
  两者的共同根因: 隐式依赖tensor是连续的。


### 3.6 Upstream API适配与特性移植

torch_npu维护了一套与upstream PyTorch API对应的NPU实现。
当upstream演进API签名或引入新特性时，torch_npu面临两个选择:
(1) 跟进(适配成本 + 兼容性风险)  (2) 维持现状(技术债务累积)。
本节记录两个典型场景: 一个成功迁移，一个被迫放弃。


#### Case 3.6.1: custom_fwd/bwd签名落后 → 迁移到upstream device-generic API

Commits: 65fd39a25 (D-58)
Files: torch_npu/npu/amp/autocast_mode.py, test/torch_npu_schema.json
Defect ref: D-58

Scenario: torch_npu自行实现了custom_fwd/custom_bwd装饰器(用于自定义autograd函数的AMP)。
upstream PyTorch自2.4.0起提供了device-generic版本:
torch.amp.custom_fwd(device_type='npu')。torch_npu的自有实现签名已过时:
用**kwargs接收参数(vs upstream的keyword-only参数)，且不发出deprecation warning。
test_autocast_custom_deprecated_warning测试因此失败。

Constraints:
- torch_npu的custom_fwd有~50行自有逻辑(autocast状态保存、dtype cast、context管理)
- upstream的torch.amp.custom_fwd已覆盖这些逻辑且更完善(device-generic)
- 用户代码可能已经依赖torch_npu.npu.amp.custom_fwd的import路径
- 需要同时标记deprecated + 保持向后兼容

Decision: 从~90行自有实现缩减到thin wrapper + @deprecated:

    autocast_mode.py
    @deprecated(
        "`torch_npu.npu.amp.custom_fwd(args...)` is deprecated. "
        "Please use `torch.amp.custom_fwd(args..., device_type='npu')` instead.",
        category=FutureWarning,
    )
    def custom_fwd(fwd=None, *, cast_inputs=None):
        return functools.partial(
            torch.amp.custom_fwd, device_type="npu"
        )(fwd=fwd, cast_inputs=cast_inputs)

custom_bwd同理。-70行，+25行。自有逻辑全部删除，delegate到upstream。

Alternatives:
- 只更新签名(kwargs → keyword-only)，保留自有实现(治标不治本)
- 直接删除，强制用户迁移到torch.amp.custom_fwd(破坏向后兼容)
- 用__getattr__做import重定向到torch.amp(过于隐式)

Consequence: 自有实现与upstream功能重复且签名不同步。
test_autocast_custom_deprecated_warning失败只是表象，
本质是维护独立AMP实现的技术债务。

Transferable:
- upstream提供device-generic API时及时migrate并deprecate自有实现。
  维护独立实现的成本随upstream演进线性增长。
  判断标准: torch.amp/torch.distributed等命名空间下有device_type参数的API
  出现后，torch_npu.npu.*下的同名API应进入deprecation cycle。
- thin wrapper + @deprecated是最佳过渡模式:
  保持import路径兼容、发出warning引导迁移、零维护成本。


#### Case 3.6.2: deterministic empty填充 → NPU路径不兼容，选择放弃

Commits: a86817e1f (D-112)
Files: TensorFactories.cpp, ResizeNpu.cpp, .pytorch-disabled-tests.json
Defect ref: D-112

Scenario: PyTorch upstream引入"确定性模式下empty/resize填充NaN/MAX_INT"特性，
用于调试中发现使用未初始化内存的bug。torch_npu尝试对齐此特性，
在empty/empty_like/empty_strided/empty_out/resize_五处调用了upstream的
fill_empty_deterministic_和fill_resize_deterministic_。
但NPU的empty实现路径与CPU/CUDA不同(通过ACL分配device内存)，
upstream的fill函数假设tensor在host或CUDA device上 → 在NPU tensor上行为异常
→ 全dtype测试失败。

Constraints:
- upstream的fill_empty_deterministic_内部用at::native::fill_实现
- NPU的tensor数据在device上，at::native::fill_走的是NPU dispatch，
  但fill逻辑期望特定的内存布局(deterministic fill用memset-like操作)
- NPU的empty创建路径: ACL分配 → StorageDescHelper::SetDesc → restride，
  与CPU/CUDA的malloc → fill → view路径不同
- 修复五个入口点(empty/empty_like/empty_strided/empty_out/resize_)

Decision: 从所有五处入口完整删除deterministic填充逻辑:

    TensorFactories.cpp (四处删除):
    // 删除 should_fill_empty_deterministic() 辅助函数
    // 删除 empty/empty_like/empty_strided/empty_out 中的:
    if (C10_UNLIKELY(should_fill_empty_deterministic())) {
        at::native::fill_empty_deterministic_(tensor);
    }

    ResizeNpu.cpp (一处删除):
    // 删除 old_storage_nbytes 计算和 fill_resize_deterministic_ 调用

    // 删除两个include:
    // <ATen/native/ResizeCommon.h>
    // <ATen/native/TensorFactories.h>

相关测试加入disabled列表(+22行)。选择暂不支持该upstream特性。

Alternatives:
- 实现NPU版本的fill_empty_deterministic_(用aclrtMemset填充device内存)
- 在dispatch层做设备分流: CPU/CUDA用upstream实现，NPU用自定义实现
- 用c10::impl::device_guard在fill前做H2D拷贝(性能代价过高)

Consequence: NPU设备上确定性模式的empty/resize不做NaN填充。
对调试使用未初始化内存的场景没有保护。
实际影响较小: 该特性主要用于调试，生产环境不开启。

Transferable:
- 移植upstream特性前，须验证特性依赖的底层API在NPU上是否可用。
  fill_empty_deterministic_ → at::native::fill_ → NPU dispatch → 行为可能不同。
  不能假设upstream实现的调用链在NPU上"自动适配"。
- "选择放弃"是合法决策。不是所有upstream特性都值得移植。
  判断标准: 特性的用户价值(调试辅助 vs 核心功能) / 适配成本(重写底层 vs 一行调用)。
  本case中: 低价值(调试特性) + 高成本(五处入口 + 自定义fill实现) → 合理放弃。


### 3.7 Inductor codegen: 资源释放与stride校验

torch.compile在NPU上通过torch_npu._inductor模块工作。
NPUWrapperCodeGen继承upstream的PythonWrapperCodegen，覆写了若干方法
来适配NPU设备。覆写时容易引入两类问题:
(1) 过度覆写(删除关键逻辑)  (2) 不兼容的断言(NPU kernel的行为偏离upstream假设)。


#### Case 3.7.1: make_buffer_free空覆写导致tensor永不释放

Commits: c83bac42c (D-121)
Files: torch_npu/_inductor/__init__.py, codegen/wrapper.py, config.py, ir.py
Defect ref: D-121

Scenario: NPUWrapperCodeGen覆写了make_buffer_free(buffer)返回空字符串，
导致inductor生成的Python代码中不包含任何buffer释放语句(如del tensor)。
tensor生命周期被无限延长，显存占用持续增长。
同一commit还修复了另一个问题: _to_copy和reshape在NPU上的output stride
与eager模式不一致，aoti场景下的stride断言误报。

Constraints:
- make_buffer_free由upstream PythonWrapperCodegen定义，生成"del buf"语句
- NPU开发者可能因为"NPU的buffer释放由其他机制管理"而覆写为空(推测)
- 空覆写没有注释说明理由，无法判断是有意还是遗漏
- stride断言问题: NPU的_to_copy强制Contiguous format，
  compile-time(fake tensor)的stride与runtime(NPU tensor)不同

Decision: 两处修复:

(1) 删除make_buffer_free的空覆写，恢复框架默认的buffer释放逻辑:

    wrapper.py
    // 删除以下三行:
    # don't free anything
    def make_buffer_free(self, buffer):
        return ""

(2) 新增stride断言的配置化跳过:

    config.py
    skip_specific_stride_asserts = False

    ir.py
    def patch_extern_kernel_codegen_size_asserts():
        original_codegen_size_asserts = ExternKernel.codegen_size_asserts
        ops_with_variable_stride = (
            torch.ops.aten._to_copy.default,
            torch.ops.aten.reshape.default,
        )
        def npu_codegen_size_asserts(self, wrapper):
            fx_node = getattr(self, 'fx_node', None)
            should_skip = False
            if npu_config.skip_specific_stride_asserts and fx_node and fx_node.target:
                should_skip = fx_node.target in ops_with_variable_stride
            if should_skip:
                wrapper.writeline(f"# NPU: Skipping stride assertion for ...")
            else:
                original_codegen_size_asserts(self, wrapper)
        ExternKernel.codegen_size_asserts = npu_codegen_size_asserts

Alternatives:
- make_buffer_free: 用NPU特有的释放逻辑(如aclrtFree)替代del语句(过度复杂)
- stride断言: 全局禁用所有stride断言(范围过广，掩盖其他真实问题)
- stride断言: 在_to_copy的NPU实现中保留eager stride(需改CANN kernel)

Consequence:
- buffer不释放: torch.compile后的模型推理/训练中显存持续增长，
  直到OOM。不影响计算结果但限制batch size和序列长度。
- stride断言: aoti模式下正确的计算结果触发误报的assertion error。

Transferable:
- 覆写框架关键路径方法(尤其是资源释放)时须在注释中说明理由。
  无注释的空实现是code review红旗。
  审查规则: grep所有"return \"\""或"pass"的方法覆写，验证是否有意为之。
- NPU kernel的stride行为可能偏离upstream假设(compile-time vs runtime不一致)。
  解决方案应该是per-op配置化跳过，而非全局禁用断言。
  配置化允许逐步收敛: 当CANN kernel修复stride行为后，逐个从skip列表中移除。


---

## Ch.4 Python集成与Monkey-patch

torch_npu作为out-of-tree后端，无法直接修改PyTorch源码，因此大量依赖Python monkey-patch
来注入NPU支持。defect-analysis统计显示monkey-patch相关缺陷36条，是仅次于upstream同步缺失
(65条)的第二大缺陷类别。本章按反模式类型组织，提炼monkey-patch的决策边界。


### 4.1 粒度陷阱: 全局替换 vs 条件拦截

Monkey-patch最常见的错误是作用范围过广。开发者用全局替换实现NPU特化，
但副作用波及到不需要特化的路径(CPU、CUDA、fake backend)，
导致性能下降、功能破坏或编译失败。


#### Case 4.1.1: expr_fits_within_32bit全局禁用导致性能下降

Commits: db1de538e (fix)
Files: torch_npu/_inductor/utils.py, __init__.py, codegen/triton.py, lowering.py
Defect ref: D-16

Scenario: NPU inductor后端需要处理某些场景下的64bit索引需求。开发者选择monkey-patch
upstream的`expr_fits_within_32bit`函数，使其永远返回False，强制所有索引使用64bit。

Constraints:
- `expr_fits_within_32bit`被upstream三个模块引用(torch._inductor.utils, simd, triton_utils)
- NPU的索引需求只在特定tensor size场景下需要64bit
- onerec模型严重依赖32bit索引优化的性能收益

Decision: 完全删除`patch_expr_fits_within_32bit`及其调用。
同一commit还修复了cat constant场景(golden_var_list为None时crash):
在`cat_store`开头增加None guard，改用block_ptr索引存储。
并在cat lowering增加设备类型检查`inputs[0].get_device().type == "npu"`。

    torch_npu/_inductor/utils.py
    // 删除整个patch_expr_fits_within_32bit函数(11行):
    // 该函数将upstream三个模块的expr_fits_within_32bit替换为lambda: False

Alternatives:
- 条件化: 只在NPU设备+大tensor场景返回False(需要理解upstream的size阈值语义)
- 在NPU kernel侧支持32bit索引(改CANN侧，不改PyTorch侧)
- 用NPU自定义的IndexExpr包装器替代全局patch

Consequence:
- onerec模型性能下降被现场客户发现(ZJ site)
- 全局强制64bit索引影响所有经过inductor编译的NPU模型
- cat constant场景crash是附带发现，因为patch移除后暴露了新代码路径

Transferable:
- "永远返回False/True"的monkey-patch是最危险的模式。它消除了upstream优化路径，
  且影响范围等于被patch函数的全部调用者。
- Monkey-patch影响性能时极难诊断: 功能正确但速度下降，需要benchmark才能发现。
  Review rule: 任何修改upstream优化决策的patch必须附带性能benchmark结果。


#### Case 4.1.2: fake后端patch范围过宽

Commits: 0e4718038 (fix)
Files: torch_npu/__init__.py
Defect ref: D-114

Scenario: PyTorch的fake backend(用于测试的假ProcessGroup)不支持NPU设备。
开发者需要在fake backend注册时自动追加NPU支持。

Constraints:
- PyTorch的Backend.register_backend接受一个devices列表
- fake backend声明支持cuda，NPU需要搭载这个声明来获得注册
- nccl等真实cuda backend也声明支持cuda

Decision: 将patch条件从`'cuda' in devices`收窄为`backend_name == 'fake'`:

    torch_npu/__init__.py  _patch_backend_register_for_npu()
    // before: if 'cuda' in devices and 'npu' not in devices
    // after:  if backend_name == 'fake' and 'npu' not in devices

对已注册backend的补丁循环也做相同收窄。

Alternatives:
- 在fake backend源码中直接添加NPU支持(需修改upstream，out-of-tree做不到)
- 按backend capability细分: 只patch声明了'cuda'但没有真实device的backend
  (语义更精确但实现更复杂)

Consequence:
- 原条件`'cuda' in devices`命中了nccl等真实backend，导致这些backend被
  错误地追加NPU设备支持，引起分布式初始化时的兼容性问题。

Transferable:
- Monkey-patch第三方组件时，匹配条件应尽可能精确。用白名单(精确匹配backend_name)
  替代特征匹配('cuda' in devices)。特征匹配在新backend出现时容易误命中。
- "凡是支持X的都追加Y" → "只有Z需要追加Y"。前者是特征推断，后者是意图声明。


#### Case 4.1.3: Generator全局替换破坏CPU路径

Commits: 037a1eab9 (fix)
Files: codegen/templates/torch_funcs.py, torch_npu/__init__.py
Defect ref: D-613

Scenario: `torch._C.Generator`不支持NPU设备。之前的方案直接将
`torch.Generator`替换为`torch_npu._C.Generator`(全局替换整个类)。

Constraints:
- torch.Generator是基础设施类，CPU/CUDA/NPU都使用
- torch_npu._C.Generator只支持NPU设备
- 全局替换后CPU/CUDA用户的Generator也变成了NPU版本

Decision: 移除全局类替换，改为条件拦截函数:

    codegen/templates/torch_funcs.py
    def _generator(*args, **kwargs):
        if 'npu' in str(args) or 'npu' in str(kwargs):
            raise AssertionError(
                "Please use torch_npu._C.Generator for npu device.")
        return torch._C.Generator(*args, **kwargs)
    torch.Generator = _generator

    torch_npu/__init__.py
    // 从all_monkey_patches删除: ["_C.Generator", torch_npu._C.Generator]

Alternatives:
- 修改torch_npu._C.Generator使其同时支持CPU/CUDA/NPU(实现成本高)
- 用__torch_function__协议拦截Generator构造(更优雅但Generator未必支持)
- 不替换torch.Generator，只在NPU相关API内部使用torch_npu._C.Generator

Consequence:
- 修复了CPU/CUDA Generator被破坏的问题
- 但引入了新的副作用: torch.Generator从类变成了函数，
  isinstance(g, torch.Generator)将失败
- 检测方式`'npu' in str(args)`过于粗糙，有误报风险

Transferable:
- 全局替换一个类为另一个类是monkey-patch中最危险的操作:
  破坏isinstance检查、元类语义、子类继承。
- "条件拦截+透传"比"全局替换"安全，但仍需注意:
  替换类为函数会破坏类型系统的契约(isinstance/issubclass)。
- 字符串匹配检测(`'npu' in str(args)`)是反模式。应使用类型检查或设备属性。


#### Case 4.1.4: LayerNorm forward patch破坏torch.compile图捕获

Commits: 188009fe7 (fix)
Files: torch_npu/utils/module.py
Defect ref: D-497

Scenario: 开发者为推理场景优化LayerNorm，在非训练模式下将forward路由到
`torch_npu.npu_layer_norm_eval`自定义算子(CANN优化版本)。

Constraints:
- torch.compile需要trace整个forward方法生成计算图
- 被monkey-patch的forward包含`input.is_npu`设备检查，是graph break点
- 自定义算子`npu_layer_norm_eval`可能未注册到dynamo trace规则
- 该patch影响所有LayerNorm实例，包括CPU/CUDA上的

Decision: 完全删除layernorm_forward函数及其monkey-patch:

    torch_npu/utils/module.py
    // 删除:
    def layernorm_forward(self, input):
        if self.training or (not input.is_npu):
            return torch.nn.functional.layer_norm(...)
        else:
            return torch_npu.npu_layer_norm_eval(...)
    torch.nn.LayerNorm.forward = layernorm_forward

Alternatives:
- 注册npu_layer_norm_eval为TorchInGraphFunction并实现FakeTensor支持
  (使其可被compile正确trace)
- 使用torch.library.custom_op注册，让compile自动处理
- 在inductor的lowering规则中将LayerNorm路由到NPU kernel(不修改eager路径)

Consequence:
- torch.compile在包含LayerNorm的模型上无法正常工作(graph break或报错)
- 这是torch_npu引入compile支持早期(2023-09)发现的问题
- 删除patch后推理性能可能有轻微回退(失去CANN优化的LayerNorm)

Transferable:
- Monkey-patch nn.Module的forward方法与torch.compile天然冲突:
  compile需要trace Python代码生成图，而patch引入的设备检查和自定义算子调用
  都是graph break的根源。
- 正确的NPU算子加速路径应通过inductor lowering规则或torch.library注册，
  而非monkey-patch forward方法。这是eager优化(patch forward)向
  compiler优化(lowering rule)迁移的系统性趋势。
- Cross-ref: Ch.8将进一步讨论inductor lowering的正确注册方式。


### 4.2 from-import引用切断: Python绑定语义陷阱

Python的`from X import Y`在导入时创建Y的本地名字绑定(引用复制)。
后续对X.Y的monkey-patch不会传播到已创建的本地绑定。
这个Python语言特性导致了torch_npu中至少3个独立的bug(D-45, D-102, D-982)。


#### Case 4.2.1: npugraphs patch未传播到_graph_tree模块

Commits: 8343dcb87 (fix)
Files: torch_npu/_inductor/utils.py
Defect ref: D-45

Scenario: torch_npu需要替换`get_first_incompatible_cudagraph_node`函数，
使npugraphs后端正确识别NPU tensor的兼容性。patch已覆盖
`torch._inductor.compile_fx`和`torch._dynamo.backends.cudagraphs`两个模块。

Constraints:
- `torch_npu.utils._graph_tree`在模块加载时执行了
  `from torch._inductor.utils import get_first_incompatible_cudagraph_node`
- 该import在patch之前发生(模块加载顺序)
- _graph_tree持有原始函数的独立引用，不受后续patch影响

Decision: 在patch链末尾显式追加_graph_tree模块的patch:

    torch_npu/_inductor/utils.py  patch_get_first_incompatible_cudagraph_node()
    from torch_npu.utils import _graph_tree
    _graph_tree.get_first_incompatible_cudagraph_node = \
        get_first_incompatible_cudagraph_node

同时去掉了hasattr保护(torch_npu与torch版本锁定，不需要兼容检查)。

Alternatives:
- 让_graph_tree使用延迟导入(lazy import)，在调用时才解析引用
- 让_graph_tree通过模块引用访问(torch._inductor.utils.get_first_...)而非本地名
- 使用集中式patch注册表，自动遍历所有持有目标函数引用的模块

Consequence:
- npugraphs后端无法正确判断cudagraph节点兼容性，
  导致不兼容的节点被错误地纳入graph capture，运行时crash或产生错误结果。

Transferable:
- Monkey-patch一个被多模块from-import的函数时，必须列举所有import点。
  工具方法: `grep -r "from.*import.*target_func" .` 找出所有本地绑定。
- 架构建议: 被频繁patch的函数应通过模块级引用访问(M.func())而非本地名(func())，
  这样patch M.func自动传播到所有调用者。
- hasattr守卫在版本锁定的项目中是噪音。它掩盖了"函数不存在"的真实错误，
  应该用明确的版本断言替代。


#### Case 4.2.2: RPC BackendType枚举注册后引用未同步

Commits: 864e16ded (fix)
Files: torch_npu/distributed/rpc/backend_registry.py
Defect ref: D-102

Scenario: torch_npu注册NPU_TENSORPIPE RPC后端时，
`backend_registry.register_backend`重建了BackendType枚举(添加NPU条目)。

Constraints:
- `torch.distributed.rpc`模块在初始化时执行了
  `from .backend_registry import BackendType`
- register_backend创建了新的BackendType枚举对象(包含NPU后端)
- rpc模块的BackendType仍指向旧枚举(不包含NPU后端)
- `_validate_rpc_args`使用rpc.BackendType做isinstance检查

Decision: 注册完成后手动同步引用:

    torch_npu/distributed/rpc/backend_registry.py
    import torch.distributed.rpc as _rpc_module
    _rpc_module.BackendType = rpc.backend_registry.BackendType

Alternatives:
- 修改register_backend使其原地修改枚举而非重建(需改upstream)
- 在rpc模块中使用延迟访问: 每次通过backend_registry.BackendType而非本地名
- 注册NPU后端前先保存rpc.BackendType引用，注册后比对并同步

Consequence:
- RPC初始化时_validate_rpc_args的isinstance检查失败，
  因为传入的BackendType实例来自新枚举，而检查用的类型是旧枚举。
  TypeError: expected BackendType, got BackendType(同名但不同对象)。

Transferable:
- 枚举重建(而非原地修改)是from-import引用切断的特殊变体。
  Python的Enum一旦创建就不可变，register_backend必须创建新Enum，
  这使得引用同步成为必然需求。
- 与Case 4.2.1形成对比: 4.2.1是函数引用切断，4.2.2是类型(Enum)引用切断。
  两者的根因相同，但后果不同: 函数切断导致逻辑错误，类型切断导致类型检查失败。


### 4.3 Upstream签名漂移: Monkey-patch的维护成本

Monkey-patch本质上是对upstream内部API的非契约依赖。
当upstream修改函数签名、移除方法、或重命名模块时，
patch代码会静默失败(参数不匹配)或显式报错(AttributeError/KeyError)。
这是torch_npu中数量最多的缺陷类别(65条upstream同步缺失)的核心驱动因素。


#### Case 4.3.1: OrderedSet类型注解 + 函数签名缺参数

Commits: 7d08fde3d (fix, v2.8.0-26.0.0), 00da0b258 (v2.9.0), 747d94038 (v2.10.0), f12196ba3 (v2.7.1)
Files: torch_npu/utils/_graph_tree.py
Defect ref: D-17

Scenario: `npugraphify_impl`是upstream `cudagraphify_impl`的NPU适配版本(fork)。
upstream新增了`mutated_input_idxs`参数，torch_npu的复制版本未同步。

Constraints:
- npugraphify_impl是upstream函数的完整复制+修改(不是monkey-patch，是fork)
- upstream `align_inputs_from_check_idxs`新增了必填参数`mutated_input_idxs`
- 同一函数中还有Python类型注解的运行时副作用问题

Decision: 两处修复:

    torch_npu/utils/_graph_tree.py  npugraphify_impl()
    // (1) 移除类型注解(避免运行时求值副作用):
    // before: static_input_idxs: OrderedSet[int] = OrderedSet(...)
    // after:  static_input_idxs = OrderedSet(...)
    //
    // (2) 补充缺失参数:
    // before: return align_inputs_from_check_idxs(run, check_input_idxs)
    // after:  return align_inputs_from_check_idxs(
    //             run,
    //             inputs_to_check=check_input_idxs,
    //             mutated_input_idxs=OrderedSet(),
    //         )

Alternatives:
- 不fork函数，用monkey-patch注入差异(减少维护面积但增加patch复杂度)
- 建立自动化工具diff upstream函数签名变更(CI中的签名一致性检查)
- 使用**kwargs透传未知参数(容忍签名漂移但掩盖错误)

Consequence:
- 缺参数: align_inputs_from_check_idxs调用时TypeError，npugraphs完全不可用
- 类型注解: `OrderedSet[int]`在某些Python版本下触发运行时错误
  (OrderedSet未实现__class_getitem__)
- 同一fix被cherry-pick到4个release分支(v2.7.1, v2.8.0, v2.9.0, v2.10.0)

Transferable:
- Fork upstream函数的维护成本正比于upstream的变更频率。
  cudagraphify_impl是活跃开发区(PyTorch 2.x的核心优化路径)，
  fork它意味着每次torch升级都需要人工diff。
- Python类型注解`X: Type[T] = value`不只是文档，它在运行时被求值。
  如果Type[T]的__class_getitem__未实现，赋值语句本身就会报错。
  规则: 在非typing上下文中避免对第三方类使用泛型标注。


#### Case 4.3.2: TritonCSEVariable.__init__新增shape参数

Commits: 7650811e2 (fix)
Files: torch_npu/_inductor/codegen/triton.py, __init__.py
Defect ref: D-134

Scenario: torch_npu monkey-patch了`TritonCSEVariable.__init__`以添加`mask_vars`属性。
upstream新增了`shape: BlockShapeType`参数，patch中的__init__签名未同步。

Constraints:
- TritonCSEVariable是inductor内部类，签名不在公开API契约范围内
- torch_npu的patch需要添加mask_vars: OrderedSet[str]属性
- upstream __init__调用链需要透传shape参数到super().__init__

Decision: 重写patch函数，添加shape参数并透传:

    torch_npu/_inductor/codegen/triton.py
    def patch_TritonCSEVariable__init__(
        self, name, bounds, dtype,
        shape: BlockShapeType = None,
    ):
        super(TritonCSEVariable, self).__init__(
            name, bounds, dtype, shape=shape)
        self.mask_vars: OrderedSet[str] = OrderedSet()

Alternatives:
- 用*args, **kwargs捕获所有参数并透传(容忍未来签名变更但失去类型安全)
- 不patch __init__，改用__init_subclass__或post-init hook添加mask_vars
- 在TritonCSEVariable实例首次访问mask_vars时延迟初始化(__getattr__)

Consequence:
- __init__缺少shape参数时，upstream传入shape会触发TypeError，
  inductor编译流程中断。

Transferable:
- Monkey-patch __init__方法时，签名必须是upstream的超集(包含所有upstream参数)。
  但这意味着每次upstream新增参数都需要同步。
- 权衡: *args/**kwargs透传牺牲类型安全换取签名鲁棒性。
  对于upstream频繁变更的内部类，这个trade-off可能是值得的。


#### Case 4.3.3: Upstream移除group_fn后KeyError

Commits: 32b8fd81a (fix)
Files: torch_npu/_inductor/codegen/triton.py, __init__.py
Defect ref: D-201

Scenario: torch_npu monkey-patch了`TritonScheduling.group_fn`方法。
upstream后续移除了该方法。

Constraints:
- group_fn是TritonScheduling的内部实现细节
- 移除后patch代码`TritonScheduling.group_fn = group_fn`仍可执行
  (Python允许设置任意属性)，但upstream不再调用该方法
- patch的group_fn实现包含NPU特有的flatten/NumelList逻辑(13行)

Decision: 直接删除整个group_fn patch(函数定义 + monkey-patch行):

    torch_npu/_inductor/codegen/triton.py
    // 删除group_fn函数(13行) + __init__.py中的patch行

Alternatives:
- 将group_fn逻辑迁移到upstream新接口(需要理解upstream重构的意图)
- 保留代码但注释掉(累积死代码)

Consequence:
- group_fn被移除后，torch_npu的patch实际上变成了给TritonScheduling
  添加一个无人调用的方法。如果upstream同时重构了调用路径，
  NPU的flatten/NumelList逻辑不再被执行，导致调度行为静默退化。

Transferable:
- Upstream移除API时，patch代码不会报错(Python允许设置任意属性)，
  这比新增参数更危险: 新增参数会TypeError，移除API则静默无效。
- 防御措施: CI中添加"patch目标存在性检查":
  在monkey-patch之前assert hasattr(target, method_name)，
  upstream移除方法时立即报错而非静默失效。


### 4.4 Dynamo Trace规则: 入图分类决策

torch.compile/dynamo在trace Python代码时，需要知道每个函数调用是
"可以内联到图中"(InGraph)还是"必须断图"(Skip/UserDefined)。
torch_npu通过trace_rule.py注册NPU特有函数的分类。分类错误导致两种后果:
漏注册 → 不必要的graph break; 错误注册 → trace失败。


#### Case 4.4.1: NPU函数漏注册导致graph break

Commits: 1fbd67778 (fix)
Files: torch_npu/dynamo/trace_rule.py
Defect ref: D-113

Scenario: 三个NPU API未注册到dynamo trace规则，dynamo遇到这些调用时
graph break(退回eager模式)或产生非预期的trace行为。

Constraints:
- torch.npu._get_current_allocator和torch.npu.is_bf16_supported是非C绑定函数
- torch_npu._C._npu_resetAccumulatedMemoryStats是C绑定函数
- 两类函数注册到不同的字典(non_c_binding vs c_binding)

Decision: 将三个函数注册到对应的trace rule字典:

    torch_npu/dynamo/trace_rule.py
    // non_c_binding新增:
    "torch.npu._get_current_allocator": TorchInGraphFunctionVariable,
    "torch.npu.is_bf16_supported":     TorchInGraphFunctionVariable,

    // c_binding新增:
    "torch_npu._C._npu_resetAccumulatedMemoryStats":
        TorchInGraphFunctionVariable,

Alternatives:
- 自动发现: 扫描torch.npu和torch_npu._C的所有公开函数，批量注册
  (覆盖广但可能引入不应入图的函数)
- 延迟注册: 在函数定义处用装饰器标记入图属性(@in_graph_function)

Consequence:
- graph break导致torch.compile的模型无法生成完整的计算图，
  性能退化到接近eager模式。
- defect-analysis显示这个模式反复出现(D-113, D-703, D-740, D-784, D-1040):
  同一根因(NPU函数未注册)至少触发了5个独立的fix commit。

Transferable:
- 新增NPU公开API的checklist应包含: "是否需要注册到dynamo trace rule?"
  区分c_binding(torch_npu._C.*)和non_c_binding(torch.npu.*)。
- 该缺陷反复出现说明缺少机制性防护。建议:
  CI测试中torch.compile一个包含所有NPU API调用的模型，检测graph break数量。


#### Case 4.4.2: torch.device错误归类为InGraph导致compiled autograd失败

Commits: 638106a64 (fix)
Files: torch_npu/utils/_dynamo.py
Defect ref: D-104

Scenario: torch_npu将`torch.device`归类为`TorchInGraphFunctionVariable`。
但在原生PyTorch中torch.device是`SkipVariable`
(即dynamo不应尝试trace其内部实现)。

Constraints:
- compiled autograd的backward中使用torch.device("npu")构造设备对象
- dynamo尝试trace torch.device的__new__方法(因为被标记为InGraph)
- torch.device的C++实现不可被Python-level trace
- 错误分类使compiled autograd在所有包含device构造的backward中失败

Decision: 从`UserDefinedClassVariable__new__`的InGraph列表中移除torch.device:

    torch_npu/utils/_dynamo.py
    // 从InGraph类型列表中删除: torch.device

Alternatives:
- 为torch.device实现完整的InGraph支持(需要C++侧配合，工作量大)
- 在compiled autograd路径中特殊处理device构造(hack)
- 改用torch.device.type/index属性访问替代device构造(改用户代码)

Consequence:
- compiled autograd完全不可用于包含torch.device构造的backward函数。
- 这是一个"过度入图"的反例: 不是所有PyTorch对象都应该被trace，
  某些基础设施类(device, dtype, layout)应保持为常量而非图节点。

Transferable:
- 与Case 4.4.1形成对称: 4.4.1是漏注册(该入图的没入图)，
  4.4.2是误注册(不该入图的被强制入图)。两个方向的错误都有严重后果。
- 判断函数是否应入图的规则:
  (1) 函数是否有副作用(有 → Skip)
  (2) 函数是否可被Python trace(不可 → Skip)
  (3) 函数返回值是否参与计算图(不参与 → Skip)
  torch.device三条都不满足InGraph条件。


---

## Ch.5 Profiler与可观测性

torch_npu的profiler子系统(torch_npu/profiler/ + torch_npu/csrc/profiler/)
是仓库中缺陷最密集的模块之一。git-history显示profiler相关BUGFIX commit超过200条
(含多版本cherry-pick)，defect-analysis中独立缺陷约40个。

这些缺陷呈现清晰的结构性模式:
- 生命周期管理(状态机前置条件缺失、eager init副作用)
- 动态Profiling的配置解析(多维度组合未覆盖、步数追踪不一致)
- 解析器pipeline的数据流断裂(重排、配置传播、算法退化)
- CANN外部接口的防御式编程缺失
- mstx集成的C++/Python跨层问题

本章按这五个维度组织，每个维度下的cases揭示了可复用的防御策略。


### 5.1 生命周期管理: Profiler状态机缺陷

Profiler对象有明确的生命周期: 创建→启动(start/enter)→采集→停止(stop/exit)→解析。
状态机的前置条件缺失是此模块最基础的缺陷类型。


#### Case 5.1.1: stop-without-start空指针崩溃

Commits: 003752f5f (fix, +5 cherry-picks)
Files: torch_npu/csrc/profiler/npu_profiler.cpp:374,
  torch_npu/profiler/analysis/prof_common_func/_path_manager.py
Defect ref: D-223

Scenario: 用户创建`torch.profiler.profile()`对象后未调用`__enter__`/`start()`
即析构(或显式调用stop)。

Constraints:
- Python的`__del__`会在对象生命周期结束时调用，析构路径执行`stopNpuProfiler()`
- `stopNpuProfiler()`直接`_pop(PROFILER_STATE)`获取state，此时state为nullptr
- 后续对nullptr的操作触发段错误或异常
- 同时`get_realpath("")`对空路径调用`os.path.expanduser`返回空值导致二次错误

Decision: 在`stopNpuProfiler()`开头添加`profilerEnabled()`前置检查:

    // npu_profiler.cpp:374
    if (!torch::autograd::profiler::profilerEnabled()) {
        ASCEND_LOGE("Can't stop ... when it's not started.");
        return;
    }

同时在`get_realpath()`开头添加空path检查。

Alternatives:
- 在Python层__del__中检查self.started标志(更早拦截但依赖Python状态一致性)
- 使用RAII guard替代显式start/stop(根本性重构，工作量大)

Consequence:
- 修复总量极小(6行)，但需要cherry-pick到5个版本分支。
- 根因极其基础: 资源清理函数未检查对应的初始化是否完成。
  这是RAII/状态机编程的基本规则，但在手动管理生命周期时容易遗漏。

Transferable:
- 资源清理函数(stop/close/finalize/destroy)的第一行必须是"是否已初始化"的前置检查。
- 析构路径是异常路径的子集，需要与正常路径等价的防御。
  torch_npu中Ch.2.4的NPUQueue dispatch和Ch.1.7的stream析构也遵循此模式。


#### Case 5.1.2: eager init阻塞主功能链(三连击)

Commits: 957e6695d (D-336, singleton提取), ac8f4abfe (D-335, lazy init),
  3a359b8f2 (D-337, CannPackageManager lazy化)
Files: torch_npu/profiler/_dynamic_profiler/_dynamic_monitor_proxy.py:9-21,
  torch_npu/profiler/analysis/prof_common_func/_cann_package_manager.py:35-42,
  torch_npu/profiler/profiler_interface.py
Defect ref: D-335 + D-336 + D-337

Scenario: profiler模块的三个独立组件在import时或构造时执行外部依赖检查，
任一检查失败即阻塞整个profiler功能。

Constraints:
- D-336: `_call_dyno_monitor`每次调用都`from IPCMonitor import PyDynamicMonitorProxy`
  并创建新实例。IPCMonitor是可选dynolog依赖，未安装时每次import都尝试+失败。
- D-335: 上述singleton提取后，`__init__`仍然立即调用`_load_proxy()`，
  import失败无缓存机制，构造时每次重试。同时配置解析中`"false"`字符串未做
  BOOL_MAP转换，`"None"`被`float()`转换抛ValueError。
- D-337: `CannPackageManager.SUPPORT_EXPORT_DB = check_cann_package_support_export_db()`
  在类定义时(module import时)立即调用CANN包检查。容器环境中CANN未就绪时
  import即失败，profiler完全不可用。

Decision: 三个修复共同的模式 -- eager → lazy:

    // _dynamic_monitor_proxy.py (D-335)
    def __init__(self):
        self._proxy = None
        self._load_success = True  // 失败缓存标志
    def _load_proxy(self):
        if not self._proxy and self._load_success:
            try:
                from IPCMonitor import PyDynamicMonitorProxy
            except Exception:
                self._load_success = False  // 一次失败，永久缓存
                return
            self._proxy = PyDynamicMonitorProxy()
    def get_proxy(self):
        self._load_proxy()  // 延迟到首次使用
        return self._proxy

    // _cann_package_manager.py (D-337)
    class CannPackageManager:
        SUPPORT_EXPORT_DB = None  // 哨兵值替代立即求值
        @classmethod
        def is_support_export_db(cls) -> bool:
            if cls.SUPPORT_EXPORT_DB is None:
                cls.SUPPORT_EXPORT_DB = check_cann_package_support_export_db()
            return cls.SUPPORT_EXPORT_DB

Alternatives:
- 环境变量控制是否跳过检查(运维负担转移给用户)
- 全局try-except包裹整个profiler import(粒度太粗，掩盖其他问题)
- 编译时检测CANN版本(无法应对运行时环境变化)

Consequence:
- D-336→D-335是同一模块的两次修复(singleton提取后发现init仍然eager)。
  说明重构不彻底时容易产生"半修复"。
- D-335和D-337在同一轮提交中修复(各6个cherry-pick)，说明eager init是系统性问题。
  defect-analysis标注D-335/D-337与D-140/D-229/D-276"同族"，共5个独立模块
  犯了相同的错误。

Transferable:
- 可选外部依赖的import/检查必须lazy化且缓存失败状态。
  模式: `_loaded: Optional[bool] = None`, 首次调用时求值，失败设False永不重试。
- 类属性赋值中禁止调用可能失败的外部函数。改用`None`哨兵+`@classmethod`的lazy模式。
- profiler_interface.py中`__init__`过早校验DB导出能力也被D-337移除:
  校验应推迟到实际需要导出DB时，而非用户还未开始profiling就抛异常。


### 5.2 动态Profiling: 配置与步数追踪

torch_npu的动态Profiling(`_DynamicProfile`)允许在训练中途通过外部配置
启停profiler。这个子系统涉及多维度状态(rank、step、warmup、async_mode)，
每个维度的组合测试不充分是缺陷的根源。


#### Case 5.2.1: async_mode维度遗漏导致解析被错误关闭

Commits: de6c3901f (fix, +8 cherry-picks)
Files: torch_npu/profiler/_dynamic_profiler/_dynamic_profiler_config_context.py
Defect ref: D-175

Scenario: 动态Profiling指定部分rank采集时，异步解析模式下解析功能被错误关闭。

Constraints:
- `ConfigContext`在验证rank时，若当前rank在rank_set中，无条件设`_analyse = False`
- 同步模式下关闭解析合理(避免解析阻塞采集流程)
- 但异步模式(`async_mode=True`)解析在独立线程/进程中，不阻塞采集
- 原代码缺少对`_async_mode`的判断，导致异步模式也关闭了解析

Decision: 添加`if not self._async_mode`条件门控:

    if self._rank_id in self._rank_set:
        if not self._async_mode:  // 仅同步模式关闭解析
            self._analyse = False
            DynamicProfilerUtils.out_log(
                f"Rank {self._rank_id} ... and async_mode is false, "
                "profiler data analyse will be closed !")
        return True

Alternatives:
- 将async/sync模式的rank处理逻辑完全分离为两个方法(更清晰但代码重复)
- 通过策略模式注入不同的rank验证器(过度设计)

Consequence:
- 部分rank + 异步模式是一个合理的使用场景(大规模训练中只采集部分rank的profiling数据，
  异步解析避免阻塞训练)，但在初始实现中未被测试覆盖。
- cherry-pick到8个版本分支，说明此功能已广泛部署。

Transferable:
- 当功能有N个独立维度(rank过滤 x 同步/异步 x warmup x ...)时，
  N维组合中的非显然组合(如"部分rank + 异步")容易遗漏。
- 设计review时应要求列举维度组合矩阵，检查每个组合的预期行为。


#### Case 5.2.2: step offset两次才修完

Commits: d08ef6395 (D-312, 主修复), 49994df4c (D-307, 补漏)
Files: torch_npu/profiler/profiler.py,
  torch_npu/profiler/dynamic_profile.py
Defect ref: D-312 + D-307

Scenario: 动态profiler在训练中途启动时，`ProfilerStep#N`标记中的N从0开始，
与实际训练step不对应。同时缺少mstx range标记用于step粒度可视化。

Constraints:
- `_DynamicProfile`的`step_num`是采集窗口内的相对步数(0开始递增)
- `ProfilerStep#N`标记写入profiling数据，用户根据N定位训练步数
- N=0对应训练第10000步这样的错位使profiling数据难以关联训练日志

Decision(D-312): 新增`_step_num_offset`属性:

    // profiler.py
    self._step_num_offset = 0  // 新增

    // dynamic_profile.py: 启动时设置offset
    self.prof._set_step_num_offset_for_dynamic_prof(self.cur_step)
    self.prof.start()

    // profiler.py start()中:
    "ProfilerStep#" + str(self.step_num + self._step_num_offset)

但`step()`方法中构造`ProfilerStep#`标签时遗漏了offset:

    // profiler.py step()中(D-312修复后仍有bug):
    self.step_rec_fn = prof.record_function(
        "ProfilerStep#" + str(self.step_num))  // 缺 + self._step_num_offset

Decision(D-307): 补上遗漏的offset:

    "ProfilerStep#" + str(self.step_num + self._step_num_offset)

Alternatives:
- 在`_DynamicProfile`内部直接修改`step_num`的起始值(侵入性更强)
- 使用装饰器统一在所有构造`ProfilerStep#`的位置注入offset(防遗漏但复杂)

Consequence:
- D-312和D-307是典型的"一个commit包含两个feature，导致其中一个feature的实现不完整"。
  D-312同时引入了step offset和mstx range两个变更，offset在start()中实现了但
  step()中遗漏了。
- D-307在D-312之后独立提交修复，再cherry-pick到7个版本分支。
  如果D-312的review更彻底(grep所有构造`ProfilerStep#`的位置)，D-307可以避免。

Transferable:
- 影响record标识(如`ProfilerStep#N`)的状态变量，修改时必须grep所有构造该标识的位置。
- 一个commit应只包含一个逻辑单元。D-312混合了step offset和mstx range两个独立feature，
  增加了review难度和遗漏风险。


#### Case 5.2.3: warmup边界 -- 0不是无效值

Commits: ff314ef75 (fix, +5 cherry-picks)
Files: torch_npu/profiler/_dynamic_profiler/_dynamic_profiler_config_context.py
Defect ref: D-330

Scenario: 动态profiling的warmup参数设为0时被视为无效值，强制回退到默认值。

Constraints:
- warmup=0表示"不需要预热，直接开始采集"，是合法的用户意图
- 原代码: `if not isinstance(self._warmup, int) or self._warmup <= 0`
- `<= 0`将0归入无效范围，打印"Invalid parameter warmup, reset it to 0"后重置

Decision: `<= 0` → `< 0`:

    if not isinstance(self._warmup, int) or self._warmup < 0:
        // ... reset to 0

Alternatives:
- 允许负数作为"使用默认值"的信号(语义不直观)
- 使用None作为"未设置"的哨兵值(需要修改更多代码)

Consequence:
- 单字符修复(`<=` → `<`)，但影响所有warmup=0的动态profiling场景。
- 这是"边界值"类缺陷的教科书案例: 0既不是正数也不是负数，
  但对于warmup这个语义来说它是合法值。

Transferable:
- 参数校验中`<= 0`和`< 0`的选择取决于0是否有业务含义。
  对于warmup/timeout/retry类参数，0通常表示"立即/不等待/不重试"，是有效值。
- Review时对所有`<= 0`的边界检查追问: "0是什么含义?"


#### Case 5.2.4: 状态机残留导致日志刷屏

Commits: 7644f42b5 (fix, +8 cherry-picks)
Files: torch_npu/profiler/dynamic_profile.py
Defect ref: D-224

Scenario: 动态profiling配置无效(start_step不匹配当前step)时，
每个step都打印相同的警告日志，长训练任务中产生数万行重复日志。

Constraints:
- `_DynamicProfile.step()`每步检查`cfg_ctx`是否非空
- 配置无效时设状态为IDLE并打印警告，但`cfg_ctx`未置None
- 下一个step仍检查到非空`cfg_ctx`，再次进入异常分支

Decision: 在异常分支中清空`cfg_ctx`:

    print_warn_msg(f"Dynamic Profiler config is not effective. ...")
    self._dynamic_monitor.update_profiler_status(...IDLE)
    self.cfg_ctx = None  // 阻断后续重复触发

Alternatives:
- 用计数器限制同一警告的打印次数(治标不治本)
- 在IDLE状态中添加early return跳过所有检查(需要修改更多分支)

Consequence:
- 状态机转移到终态(IDLE)后残余字段未清理是经典bug。
  每一个状态转移都应有"exit action"清理该状态关联的所有上下文。
- 长训练任务(数十万step)中日志刷屏可能导致磁盘写满或日志系统过载。

Transferable:
- 状态机设计原则: 状态转移 = 退出旧状态(清理上下文) + 进入新状态(初始化上下文)。
  只做一半会产生"幽灵状态"。
- 与Case 5.1.1(stop-without-start)同源: 都是生命周期管理的前置/后置条件缺失。


### 5.3 解析器Pipeline: 数据流断裂

profiler的解析器(torch_npu/profiler/analysis/)是一个多stage pipeline:
配置加载→数据采集→CANN数据解析→事件匹配→视图生成→导出。
stage之间的数据依赖和配置传递是缺陷高发区。


#### Case 5.3.1: 解析器重排引入超时回归

Commits: 20357b76d (Revert)
Files: 11个文件(prof_common_func/_file_manager.py, prof_config/_parser_config.py,
  prof_view/_trace_view_parser.py, cann_parse/_cann_export.py等)
Defect ref: D-172

Scenario: 一个名为"memory optimaze"的优化提交试图将profiler解析器从串行执行
改为部分并行，但改变了数据流依赖导致动态Profiling场景下超时。

Constraints:
- 原始pipeline中CANNTimelineParser先于TraceViewParser执行，后者依赖前者的输出
- "memory optimaze"重排了执行顺序，TraceViewParser在CANN数据就绪前开始解析
- 动态Profiling场景下CANN数据异步生成，时序更敏感
- 同时`append_trace_json_by_path`签名变更(新增`new_name`参数)改变了json拼接逻辑

Decision: 完整revert该优化提交，恢复原始执行顺序。

Alternatives:
- 修复数据依赖(添加同步点)而非完整revert(复杂度高，风险大)
- 用DAG调度器管理解析器依赖(正确但重构量大)

Consequence:
- 11个文件的修改被完整revert。这是一个典型的"优化引入回归"案例。
- 核心教训: 修改多stage pipeline的执行顺序时，必须先绘制数据依赖DAG，
  验证新顺序满足拓扑排序约束。仅凭"看起来没有依赖"不够，需要分析数据流。

Transferable:
- Pipeline优化的前置条件: 数据依赖DAG分析。如果pipeline的stage之间通过
  文件系统/共享状态传递数据(而非显式的pipe/queue)，依赖关系是隐式的，
  容易在重排时断裂。
- 当优化提交引入回归且修复成本高于收益时，revert是正确选择。
  不要试图在broken的优化上打补丁。


#### Case 5.3.2: activities配置传播缺失(连环bug)

Commits: 88c1a9abb (D-303, +5 cherry-picks), 11610b99b (D-304, +5 cherry-picks)
Files: torch_npu/profiler/analysis/_profiler_config.py,
  torch_npu/profiler/analysis/_profiling_parser.py,
  torch_npu/profiler/analysis/prof_view/_memory_view_parser.py
Defect ref: D-303 + D-304

Scenario: 纯CPU profiling场景(不含NPU活动)下，profiler仍尝试解析NPU数据，
导致异常。根因是`activities`配置未在解析链中正确传播。

Constraints:
- `profiler_info.json`中记录了用户选择的activities(CPU/NPU)
- `ProfilerConfig.load_info()`从JSON中读取activities但未存储为属性
- `load_timediff_info()`每次重新从JSON解析activities(重复解析)
- 下游parser无法判断是否包含NPU活动，无条件尝试解析NPU数据
- 纯CPU场景下CANN目录不存在，解析触发异常

Decision(D-303): `ProfilerConfig`新增`_activities`属性:

    // _profiler_config.py
    self._activities = []

    def load_info(self, profiler_path):
        // ...
        self._activities = info_json.get(Config, {}).get(
            CommonConfig, {}).get(Activities, [])

    // _profiling_parser.py: 通过param_dict传递给下游
    param_dict[Constant.ACTIVITIES] = ProfilerConfig().activities

Decision(D-304): 下游解析器添加activities前置判断:

    // _memory_view_parser.py: NPU数据获取前检查
    if Constant.NPU_ACTIVITIES in self._activities:
        // ... 获取NPU内存数据

    // 同时将硬编码的"ProfilerActivity.NPU"提取为常量
    // 并对analyse_profiling_data加@no_exception_func()装饰器

Alternatives:
- 在pipeline入口统一判断activities，NPU不在列表中则完全跳过CANN解析链
  (更干净但需要重构pipeline调度)
- 每个parser自行检查activities(当前方案，分散但灵活)

Consequence:
- D-303和D-304是同一根因(activities传播缺失)的两个修复:
  D-303解决配置传递缺失，D-304解决下游未判断活动类型。
- 这揭示了profiler解析器的设计问题: 配置通过`param_dict`以松散字典形式传递，
  缺少类型约束，容易遗漏字段。

Transferable:
- 配置在pipeline中传递时，源头加载后必须存储为typed属性(而非每次重新解析JSON)。
- 解析NPU特有数据前必须先检查activities配置。这个规则在D-291(CPU活动路径遗漏)
  中再次被违反，说明需要机制性保障而非case-by-case修复。


#### Case 5.3.3: 双指针匹配算法O(n^2)退化

Commits: 75c77b902 (fix, +5 cherry-picks)
Files: torch_npu/profiler/analysis/prof_view/prof_db_parse/_fwk_api_db_parser.py
Defect ref: D-154

Scenario: profiler的事件匹配(dequeue↔node_launch、enqueue↔torch_op)
使用双指针算法，但缺少"已越过目标时间窗口"时的early-break。

Constraints:
- 两个有序序列(按时间戳排序)需要按时间窗口配对
- 标准双指针算法: 外指针遍历A，内指针在B中前进直到匹配或越过
- 原代码在"匹配成功"时break，但"内指针已越过目标窗口"时未break+更新last_index
- 导致内指针不更新，每次外循环内指针从旧位置重新扫描→O(n^2)

Decision: 在三处双指针循环中添加"越过目标"的break和last_index更新:

    // _fwk_api_db_parser.py (示例修复点之一)
    if dequeue_list[idx][START_NS] > node_launch_api[END_NS]:
        last_dequeue_index = idx  // 更新外层起始位置
        break                     // 提前终止内循环

Alternatives:
- 使用SortedList/BST替代线性扫描(常数因子更大但渐近更优)
- 将匹配逻辑改为merge-join(标准数据库技术，但需要两个序列都排序)

Consequence:
- 大规模profiling数据(数十万事件)下，O(n^2)退化导致解析超时。
  修复后恢复O(n)。
- 双指针算法的不变量是"内指针单调前进"。缺少early-break时内指针回退破坏了不变量。

Transferable:
- 有序序列的双指针匹配算法三要素: (1)匹配时break (2)越过时break+更新起点 (3)未到时continue。
  缺少(2)就退化为O(n^2)。
- profiler解析器处理的数据量与模型复杂度正相关。O(n^2)算法在小模型上不暴露，
  只有在大规模场景下才可观测。性能测试应覆盖大数据量场景。


### 5.4 CANN外部接口: 防御式编程

torch_npu的profiler依赖CANN(昇腾计算架构)提供的底层采集接口和数据路径。
CANN版本差异、安装状态、路径权限是持续的兼容性挑战。


#### Case 5.4.1: aclrtStreamGetId接口不存在的运行时探测

Commits: 85d2a49b9 (fix, +8 cherry-picks)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp,
  torch_npu/csrc/profiler/npu_profiler.cpp
Defect ref: D-214

Scenario: profiler采集内存数据时调用`aclrtStreamGetId`获取stream ID，
但该接口在低版本CANN中不存在。

Constraints:
- `reportMemoryDataToNpuProfiler()`直接调用`AclrtStreamGetId(data.stream, &stream_id)`
- 低版本CANN的动态库中无此符号，dlsym返回NULL，调用触发空指针崩溃
- profiler不应因为缺少一个可选的元数据采集接口就完全不可用

Decision: 添加运行时接口探测:

    // AclInterface.cpp: 新增探测函数
    bool IsExistRtGetStreamId() {
        typedef aclError(*Func)(aclrtStream, int32_t*);
        static Func func = (Func) GET_FUNC(aclrtStreamGetId);
        return func != nullptr;
    }

    // npu_profiler.cpp: 调用前检查
    int64_t stream_id = 0;
    if (c10_npu::acl::IsExistRtGetStreamId()) {
        int32_t stream_ptr = 0;
        if (AclrtStreamGetId(data.stream, &stream_ptr) != ACL_ERROR_NONE)
            stream_id = -1;
        else
            stream_id = stream_ptr;
    } else {
        stream_id = reinterpret_cast<int64_t>(data.stream);  // fallback
    }

Alternatives:
- 编译时宏控制(无法应对同一二进制在不同CANN版本上运行)
- CANN版本号比较(版本号不一定与接口可用性精确对应)

Consequence:
- `IsExistXxx()`模式是torch_npu中处理CANN接口差异的标准范式
  (AclInterface.h中有多个类似函数: IsExistMemcpyBatch, IsExistMemcpyBatchAsync等)。
- fallback方案用`reinterpret_cast<int64_t>(data.stream)`将stream指针当ID使用，
  虽然不是真正的stream ID但保证唯一性，满足profiler关联需求。

Transferable:
- CANN接口的可用性必须在运行时通过dlsym探测，不能假设静态链接时的接口清单在所有
  部署环境中一致。
- 探测失败时需要有意义的fallback而非crash。profiler场景下stream指针作为临时ID
  是合理的降级策略。


#### Case 5.4.2: CANN路径校验的三层防御演进

Commits: fd6927797 (D-82), 73a46b9f4 (D-238), d85991f31 (D-302)
Files: torch_npu/profiler/analysis/_profiler_config.py,
  torch_npu/profiler/analysis/prof_parse/_cann_file_parser.py,
  torch_npu/profiler/analysis/prof_common_func/_path_manager.py
Defect ref: D-82 + D-238 + D-302

Scenario: CANN profiling数据路径在不同部署环境下可能不存在/不可读/权限不匹配，
profiler需要在多个层级处理这些异常。

Constraints:
- D-82: `ProfilerConfig.load_info()`不校验CANN profiling路径是否存在，
  后续分析代码读取不存在的路径报出令人困惑的错误
- D-238: `CANNFileParser.__init__`的`_check_cann_path_valid()`在路径不可读时
  直接抛异常，但Level_none配置下CANN数据确实可能不存在
- D-302: `check_path_permission`权限不匹配时直接`raise PermissionError`，
  调用方无法做非中断处理

Decision: 三个修复逐步建立了多层防御:

    第1层(D-82): 配置加载时校验路径存在性
    ProfilerConfig.load_activities()校验FWK/CANN路径，不存在时移除activity

    ��2层(D-238): 解析器初始化时try-except
    CANNFileParser.__init__中path检查包裹在try-except，异常时打印error并继续

    第3层(D-302): 权限检查从异常改为返回值
    check_path_permission从raise改为返回bool，调用方自行决定处理策略

Alternatives:
- 在pipeline最外层统一catch所有路径异常(粒度太粗)
- 使用sentinel path对象封装路径状态(设计过度)

Consequence:
- 三个修复跨越不同时间提交，说明路径校验的防御策略是逐步演进的:
  先加载校验(D-82)→再初始化容错(D-238)→最后权限检查改API(D-302)。
- D-302将`raise`改为返回bool是一个API语义变更，需要检查所有调用方。
  defect-analysis对此标注"Reviewability: low"。

Transferable:
- profiler/监控工具初始化不应因可选数据源不存在而导致整体失败。
  原则: 数据源缺失 → 降级(跳过该数据源的解析) → 警告(告知用户)。
- 函数的错误处理策略(抛异常 vs 返回值)变更是高风险重构，
  必须grep所有调用方验证适配。


### 5.5 mstx集成: C++/Python跨层问题

mstx(MindStudio Tracing eXtension)是华为的profiling标记接口，
类似NVIDIA的NVTX。torch_npu通过C++层(mstx_mgr.cpp)和Python层
(_dynamic_profiler/)双层集成mstx，跨层问题是此模块的特有挑战。


#### Case 5.5.1: default domain空指针崩溃

Commits: 442016cf1 (fix, +5 cherry-picks)
Files: torch_npu/csrc/profiler/mstx_mgr.cpp,
  torch_npu/csrc/profiler/mstx_mgr.h,
  torch_npu/csrc/profiler/npu_profiler.h,
  torch_npu/profiler/experimental_config.py
Defect ref: D-320

Scenario: mstx的domain过滤机制在处理"default"域时触发三个层级的错误:
C++空指针解引用、Python空列表判断逻辑错误、系统保留名未拦截。

Constraints:
- "default"是mstx的系统内置domain，`createProfDomain("default")`返回不合法handle
- `MstxRange`析构时对nullptr domainHandle无条件调用`MstxDomainRangeEnd`
- Python层`_check_mstx_domain_params()`用`is not None`检查空列表`[]`，
  `[] is not None`为True，条件永真

Decision: 三层修复:

    // C++层: 拦截系统保留名
    // mstx_mgr.cpp
    mstxDomainHandle_t MstxMgr::createProfDomain(const string& name) {
        if (name == DOMAIN_DEFAULT) {  // "default"直接返回nullptr
            return nullptr;
        }
        // ...
    }

    // C++层: 析构增加nullptr guard
    // npu_profiler.h MstxRange::~MstxRange()
    // domainHandle检查后再调用DomainRangeEnd

    // Python层: truthy检查替代is not None
    // experimental_config.py
    if mstx_domain_include:  // 空列表[]为falsy
        // ...

Alternatives:
- 在系统内部将"default"映射为全局domain handle(需要mstx SDK支持)
- 禁止用户指定"default"为domain名(限制用户自由度)

Consequence:
- 三个独立bug在同一使用路径上串联: 用户配置domain过滤→Python层判断失败→
  C++层创建系统保留domain→析构时空指针崩溃。修复需要跨Python和C++两层。
- Python的`is not None`和truthy check的区别是经典陷阱:
  `[] is not None`为True但`bool([])`为False。

Transferable:
- handle/pointer创建函数的调用方必须检查返回值。这是C/C++编程的基本规则，
  但在"创建几乎总是成功"的代码路径中容易被忽略。
- "系统保留名"(如"default", "global", "root")需要在创建入口显式拦截。
- Python中`is not None`和truthy check有语义差异，选择应基于"None是否有特殊含义"。


#### Case 5.5.2: PythonTracer singleton的GIL死锁

Commits: 1b04447a8 (fix, +4 cherry-picks)
Files: torch_npu/csrc/profiler/profiler_python.cpp:219-266
Defect ref: D-284

Scenario: `PythonTracer::singleton()`使用C++11 static local变量实现线程安全单例，
但构造函数中`pybind11::gil_scoped_acquire`获取GIL。子线程首次访问时触发死锁。

Constraints:
- C++11保证static local变量的线程安全初始化(编译器生成的guard)
- `PythonTracer()`构造函数中`gil_scoped_acquire`请求Python GIL
- 子线程调用`call(kStartOne)`→`singleton()`→触发构造→请求GIL
- 如果主线程持有GIL并等待该子线程(如join)→经典死锁
- 多线程profiling场景(DataLoader worker中的profiling)触发此路径

Decision: 新增`instance_created_` atomic标志和`get_singleton_in_child_thread()`:

    // profiler_python.cpp
    static std::atomic<bool> instance_created_ = false;

    PythonTracer& PythonTracer::singleton() {
        static PythonTracer singleton_;
        instance_created_ = true;
        return singleton_;
    }

    PythonTracer* PythonTracer::get_singleton_in_child_thread() {
        if (instance_created_) {
            return &singleton();  // 已创建，安全访问
        } else {
            return nullptr;       // 未创建，跳过profiling
        }
    }

    // 子线程调用改为:
    case Command::kStartOne:
        if (auto tracer = get_singleton_in_child_thread()) {
            tracer->startOne();
        }
        break;

Alternatives:
- 在主线程初始化时强制创建singleton(需要找到可靠的初始化时机)
- 使用`std::call_once`+无GIL的初始化(需要重构构造函数)
- 子线程中不做Python profiling(功能裁剪)

Consequence:
- 这是C++ static init与Python GIL交互的微妙问题。
  C++11的线程安全static init机制(double-checked locking)在单一语言中是安全的，
  但当构造函数需要获取另一个语言运行时的锁(GIL)时，跨语言死锁成为可能。
- 修复策略是"子线程检查再访问": singleton未创建则跳过(不触发构造)，
  已创建则直接访问(不再触发GIL acquire)。代价是子线程在主线程完成初始化前
  不进行Python profiling，但这在语义上是可接受的。

Transferable:
- C++ singleton的构造函数若需要GIL，必须确保首次访问在主线程(持有GIL的线程)完成。
- 跨语言绑定(pybind11/cython)中，C++线程安全机制和Python GIL构成两层独立的锁，
  组合使用时需要分析死锁可能性。
- 多线程profiling是一个容易被忽略的测试维度:
  DataLoader worker线程中的profiling行为与主线程不同。


---

## Ch.6 初始化与生命周期

torch_npu作为动态加载的后端插件，其初始化与销毁逻辑横跨Python模块系统和C++ ACL
runtime两个层次。Python侧的`import torch_npu`触发`__init__.py`执行链；C++侧的
`aclInit`/`aclFinalize`管理设备资源。两层生命周期的交错是此领域缺陷的根本来源:
模块级副作用、初始化递归、析构时序竞争、资源所有权不明确。
本章覆盖的16个defect中，5个源于同一模式(模块级eager init)。


### 6.1 模块级初始化副作用

Python的`import`语句会执行模块顶层的所有代码。当顶层代码包含有副作用的函数调用
(设置环境变量、编译扩展模块、monkey-patch第三方库)时，`import torch_npu`就变成了
一个有状态操作。这违反了import的语义契约: 用户期望import只是"让名字可用"。
torch_npu中至少5个独立模块犯了同一错误，形成一个系统性模式。


#### Case 6.1.1: MLIR初始化的lazy化尝试与回退 -- 时序依赖不可见

Commits: beac5137b (D-145, 尝试lazy化), 79168bcce (D-140, 次日revert)
Files: torch_npu/_inductor/_lazy_init.py (已重构移除)
Defect ref: D-140, D-145

Scenario: 开发者认为`_lazy_init.py`顶层的`build_ascend_npu_ir_ext()`(编译MLIR扩展)
和`set_torch_npu_library_path()`(设置动态库搜索路径)是多余的eager init，
尝试将它们移到downstream的`import_npu_inductor_plugin`内部延迟执行。

Constraints:
- MLIR plugin的加载依赖于编译产物存在 + library path已设置
- 这两个前置条件在import链中由位置(先执行)隐式保证，无显式依赖声明
- downstream consumer `import_npu_inductor_plugin` 假设被调用时前置条件已满足

Decision: D-145移除了模块级调用，期望lazy化。次日D-140 revert，恢复原始行为。
"看起来可以延迟"的初始化实际有严格的时序要求，但这个要求只存在于执行顺序中，
不在任何接口契约中。

Alternatives:
- 将时序依赖显式化: `import_npu_inductor_plugin`内部检查前置条件并按需初始化
- 使用`importlib.resources`管理编译产物路径，消除对模块级path设置的依赖
- 在`_lazy_init.py`中添加注释说明顺序依赖(最低成本但最脆弱)

Consequence:
- 这是一个fix-revert cycle: 原始的eager init虽然不优雅，但是唯一正确的位置。
  revert证明了"在不理解隐式时序依赖的情况下做lazy化重构是危险的"。
- 后续D-106揭示了更深层问题: 即使保留eager init，局部变量guard也无法保证幂等性。

Transferable:
- 移除模块级初始化代码前，必须绘制完整的初始化依赖图，识别所有downstream consumer
  的隐式假设。如果假设无法显式化(通过接口契约)，就不应移除。
- fix-revert cycle是"理解不充分就动手"的典型信号。


#### Case 6.1.2: sanitizer的全局monkey-patch泄漏

Commits: f2e280ea6 (fix)
Files: torch_npu/__init__.py, torch_npu/npu/_sanitizer.py
Defect ref: D-276

Scenario: `torch_npu/__init__.py`顶层无条件调用`apply_sanitizer_patch()`。
该函数monkey-patch了`torch.Tensor.is_cuda`等方法，用于NPU sanitizer的内存检查。

Constraints:
- sanitizer是诊断工具，仅在用户设置`TORCH_NPU_SANITIZER`环境变量时才应激活
- `apply_sanitizer_patch()`在`__init__.py`顶层执行，早于环境变量检查
- monkey-patch一旦执行就不可逆(无un-patch机制)

Decision: 将`apply_sanitizer_patch()`从`__init__.py`顶层移入`enable_npu_sanitizer()`
内部，仅在用户显式启用时才执行。

Alternatives:
- 在patch函数内部检查环境变量(仍是模块级调用，但条件化)
- 用descriptor protocol替代直接monkey-patch(修改Tensor类属性为property)

Consequence:
- 修复前，所有torch_npu用户(不仅是使用sanitizer的)都受到monkey-patch影响。
  `is_cuda`返回值的变化可能干扰第三方库的设备检测逻辑。
- 这与D-140属于同一模式族: 诊断/可选功能的初始化代码"泄漏"到全局import路径。

Transferable:
- 可选功能的初始化代码不得放在无条件执行的import路径上。
  判断标准: "不使用此功能的用户是否受到影响？"如果是，就应该移到显式启用路径。
- monkey-patch的不可逆性使这个问题更严重: 不像变量赋值可以回退，
  被替换的方法引用一旦丢失就无法恢复。


#### Case 6.1.3: 测试文件的模块级副作用

Commits: 2cac358b2 (fix)
Files: test/_inductor/test_torch_mlir.py
Defect ref: D-229

Scenario: 测试文件顶层包含`os.environ['TORCHINDUCTOR_MAX_AUTOTUNE']='1'`和
`from torch_npu._inductor...utils import logger`。在无torch-mlir依赖的CI环境中
import即失败，阻塞整个测试发现流程。

Constraints:
- pytest的测试发现阶段会import所有测试文件
- import失败导致整个文件被跳过(不是单个测试跳过)
- 环境变量修改在模块级执行时会影响同进程的所有后续测试

Decision: 环境变量设置移入测试方法内部; 文件级添加`@skip("request torch-mlir")`。

Alternatives:
- 用`pytest.importorskip("torch_mlir")`做条件import(更标准的pytest惯用法)
- 将MLIR测试拆分到独立的requirements文件和CI job中

Consequence: 简单但系统性的问题。D-229 + D-140 + D-276 + D-335 + D-337共5个独立
模块犯了"模块级eager init"同一错误。这不是个案，而是缺乏仓库级lint规则的结果。

Transferable:
- 测试文件顶层禁止: (1)修改环境变量 (2)import可选依赖 (3)初始化设备/网络资源。
  可通过AST lint强制: 检测模块级`os.environ`赋值和非标准库import。
- 5个独立模块重复同一错误 → 需要的不是逐个修复，而是仓库级规则。


### 6.2 初始化递归

当错误处理路径本身依赖于尚未完成的初始化时，形成递归循环。
这是一个跨越"正常路径"和"异常路径"的设计缺陷。


#### Case 6.2.1: 错误日志触发设备初始化的无限递归

Commits: c90658d5b (fix)
Files: torch_npu/csrc/core/npu/NPUException.cpp:241
Defect ref: D-81

Scenario: `cacheDeviceErrorVerboseMsg()`在捕获设备错误时尝试获取当前设备ID
以生成更具体的错误信息。它调用`c10_npu::current_device()`。

Constraints:
- `current_device()`在设备未初始化时会触发设备初始化流程
- 设备初始化失败 → 触发错误处理 → 调用`cacheDeviceErrorVerboseMsg()`
  → 再次调用`current_device()` → 再次触发初始化 → stack overflow
- 典型触发场景: 首次设备操作失败(如驱动未安装)

Decision: 将`current_device()`替换为`GetLocalDevice()`:

    // Before: 触发初始化的"重"查询
    int32_t device = static_cast<int32_t>(c10_npu::current_device());

    // After: 只读缓存的"轻"查询
    int device = c10_npu::GetLocalDevice();
    if (device < 0) { return ""; }

`GetLocalDevice()`仅读取本地缓存的device ID，不触发任何初始化。
未初始化时返回-1，此时直接返回空字符串。

Alternatives:
- 在`cacheDeviceErrorVerboseMsg`入口设递归标志位(guard against re-entrance)
- 错误日志中不包含设备信息(功能裁剪但安全)

Consequence:
- 修复前，驱动未安装的环境上`import torch_npu`后首次操作触发stack overflow。
  用户看到的不是"设备未初始化"而是段错误，调试极其困难。
- 修复只改了1行(+5行含guard)，但需要理解`current_device()`的隐式初始化语义。

Transferable:
- 错误处理路径/日志路径禁止调用可能触发初始化或分配资源的函数。
  这是"异常路径必须比正常路径更简单"原则的具体实例。
- 区分"轻查询"(只读缓存)和"重查询"(可能触发副作用)。
  API命名应体现这一区别:`GetLocalDevice()`vs`current_device()`。


### 6.3 未初始化状态防护

设备未初始化时调用设备相关API的防护策略。


#### Case 6.3.1: memory API的五重防护缺失

Commits: 688839b8f (fix, +8 cherry-picks)
Files: torch_npu/npu/memory.py:352, torch_npu/csrc/core/npu/sys_ctrl/npu_sys_ctrl.cpp,
  torch_npu/csrc/distributed/ProcessGroupHCCL.cpp
Defect ref: D-413

Scenario: 用户在NPU未初始化时调用`torch_npu.npu.memory_allocated()`等内存查询API。

Constraints: 一个commit修复了5个独立缺陷(cherry-pick 8次到不同分支):
(1) `memory_stats_as_nested_dict()`直接调用C++扩展 → NPU未init时段错误
(2) `memory_allocated()`用`dict["key"]`访问空dict → KeyError
(3) `Initialize()`中`aclrtGetDeviceCount`在NPU不可用环境下被调用
(4) `repeat_init_acl_flag_`未在构造函数中初始化(UB)
(5) alltoall_base中int64_t到uint64_t隐式转换

Decision: 分层防护:

    # Python层: 初始化状态检查
    def memory_stats_as_nested_dict(device=None):
        if not is_initialized():
            return {}

    # Python层: dict安全访问
    return memory_stats(device=device).get("allocated_bytes.all.current", 0)

    // C++层: 构造函数显式初始化
    NpuSysCtrl::NpuSysCtrl() : repeat_init_acl_flag_(true), ... {}

Alternatives:
- 在C++扩展层统一做初始化检查(减少Python侧防护代码)
- 使用`@requires_initialization`装饰器批量保护所有设备API

Consequence:
- 8次cherry-pick说明该bug影响面广(多个release分支受影响)。
- 一个commit修5个bug反映了"未初始化防护"从未被系统性审查过。

Transferable:
- 所有面向用户的设备API入口必须有`is_initialized()`前置检查。
  这应该是一个checkable规则(decorator或lint)而非依赖开发者记忆。
- dict访问用`.get(key, default)`替代`[key]`是Python防御性编程基础，
  但在"正常路径永远有值"的假设下容易被忽略。


### 6.4 初始化保护机制失效

即使开发者意识到需要初始化保护，保护机制本身也可能因实现错误而失效。


#### Case 6.4.1: 函数局部变量做单例标志 -- Python作用域陷阱

Commits: 10bc06261 (fix)
Files: torch_npu/utils/_dynamo.py
Defect ref: D-106

Scenario: MLIR扩展采用运行时动态编译加载。`patch_inductor_wrapper()`中用局部变量
`_has_inited = False`做幂等保护，防止重复编译。

Constraints:
- Python函数每次调用都重新创建局部变量，所以`_has_inited`每次进入函数时都是False
- 保护完全失效，每次调用都触发`build_ascend_npu_ir_ext()`重新编译
- 编译是耗时操作(秒级)，且可能与并发的编译产生文件竞争

Decision: 直接移除动态编译路径，改为import预编译好的扩展模块:

    # Before: 局部变量guard(失效)
    def patch_inductor_wrapper():
        _has_inited = False
        if not _has_inited:
            _has_inited = True
            build_ascend_npu_ir_ext()  # 每次都执行

    # After: 直接import预编译模块
    from torch_npu._inductor.ascend_npu_ir... import npu_inductor_plugin

Alternatives:
- 用模块级变量做guard(`_has_inited = False`放在函数外)
- 用函数属性做guard(如D-147的`func._is_patched`)
- 用`functools.cache`装饰器(适合无参数的初始化函数)

Consequence:
- 这是Python新手和有经验的开发者都可能犯的作用域错误。
  C/C++的static局部变量语义(初始化一次，后续保持)不适用于Python。

Transferable:
- Python中，初始化标志的三种正确位置:
  (1) 模块级全局变量 (2) 类属性 (3) 函数属性(`func._flag`)
  绝对不能放在函数体内作为局部变量。
- 本质是C++背景开发者对Python作用域模型的误解:
  C++ `static local` ≠ Python `local variable`。


#### Case 6.4.2: monkey-patch的幂等性守卫

Commits: dfc2f5c99 (fix)
Files: torch_npu/_inductor/shape_handling.py (已重构移除)
Defect ref: D-147

Scenario: `patch_shape_handling()`在某些代码路径下被多次调用。
每次调用都执行`patch_dynamo_context()`，导致包装函数嵌套多层。

Constraints:
- monkey-patch的本质是替换引用: `torch.original = wrapper(torch.original)`
- 重复执行: `torch.original = wrapper(wrapper(torch.original))`
- 多层包装导致: (1)调用栈加深 (2)可能的语义变化 (3)性能退化

Decision: 用函数属性做幂等守卫:

    def patch_shape_handling():
        if getattr(patch_shape_handling, "_is_patched", False):
            return
        patch_dynamo_context()
        patch_shape_handling._is_patched = True

Alternatives:
- 模块级flag变量(更显式但多一个模块级名字)
- 在被patch的函数上检查是否已被包装(inspect调用栈)
- 确保调用方只调用一次(从源头防止，但脆弱)

Consequence:
- `getattr(func, attr, default)`是Python中最标准的幂等性守卫模式。
  D-106的局部变量方案是错误的; D-147的函数属性方案是正确的。
  两者出现在同一仓库中，说明团队内没有统一的monkey-patch规范。

Transferable:
- 所有monkey-patch函数必须有幂等性守卫。推荐模式:
  `if getattr(func, "_is_patched", False): return`
  这应该是code review的checkable规则。
- D-106 + D-147 + D-276形成一个三角: 初始化保护的三种失败模式
  (错误作用域、缺少保护、无条件执行)。仓库需要一个统一的"patch protocol"。


### 6.5 析构路径的隐式副作用

进程退出或资源销毁时，代码路径与正常运行时不同。
隐式类型转换在析构路径上可能触发意外的副作用。


#### Case 6.5.1: stream销毁时隐式转换触发队列排空

Commits: 0c7ba1f7d (fix, +4 cherry-picks)
Files: torch_npu/csrc/core/npu/NPUFunctions.cpp
Defect ref: D-314

Scenario: `DestroyUsedStreams()`在进程退出时销毁所有NPU stream。
它调用`acl::AclrtDestroyStreamForce(stream)`，传入的`stream`是`NPUStream`对象。

Constraints:
- `AclrtDestroyStreamForce`的参数类型是`aclrtStream`(原始指针)
- C++编译器通过`NPUStream::operator aclrtStream()`隐式转换
- 该转换调用无参`stream()`方法，在per-stream-queue模式下先执行
  `MakeSureQueueEmpty()`排空任务队列
- 进程退出时强制销毁stream，排空队列既不必要又可能卡死

Decision: 显式调用`stream.stream(false)`:

    // Before: 隐式转换，触发队列排空
    acl::AclrtDestroyStreamForce(stream);

    // After: 显式提取handle，跳过排空
    acl::AclrtDestroyStreamForce(stream.stream(false));

`false`参数告诉`stream()`跳过`MakeSureQueueEmpty()`。

Alternatives:
- 移除`operator aclrtStream()`的隐式转换(改为`explicit`，但影响面大)
- 在`MakeSureQueueEmpty()`中检查进程退出状态
- 在`DestroyUsedStreams`中先标记"退出模式"再销毁

Consequence:
- 修改只有"一字符差异"(添加`(false)`)，但需要理解:
  (1) NPUStream的两个`stream()`重载语义
  (2) 隐式类型转换的副作用链
  (3) 析构路径vs正常路径的语义差异
- 4次cherry-pick说明问题影响多个release分支。

Transferable:
- C++隐式类型转换不应携带副作用。如果`operator T()`做了超出类型转换的事情
  (如排空队列、同步等待)，就应该声明为`explicit`。
- 析构/finalize路径上的所有操作应使用"最小副作用"版本的API。
  销毁时不需要排空(数据即将丢弃); 不需要同步(进程即将退出)。


### 6.6 共享资源的init/finalize所有权

当多个框架共存时，"谁初始化谁清理"的所有权问题。


#### Case 6.6.1: aclInit/aclFinalize的所有权标记

Commits: 89da42786 (fix, +2 cherry-picks)
Files: torch_npu/csrc/core/npu/sys_ctrl/npu_sys_ctrl.cpp:162,
  torch_npu/csrc/core/npu/sys_ctrl/npu_sys_ctrl.h
Defect ref: D-423

Scenario: torch_npu(PTA)和MindSpore等框架可能在同一进程中共存。
ACL runtime通过`aclInit`/`aclFinalize`管理，但只需init一次。

Constraints:
- `aclInit`重复调用返回`ACL_ERROR_REPEAT_INITIALIZE`(非致命)
- `aclFinalize`会释放所有ACL资源(包括其他框架正在使用的)
- 原代码将`ACL_ERROR_REPEAT_INITIALIZE`视为错误; 无条件调用`aclFinalize`

Decision: 引入所有权标记`repeat_init_acl_flag_`:

    // Init时: 区分"自己初始化"和"他人已初始化"
    if (init_ret == ACL_ERROR_REPEAT_INITIALIZE) {
        repeat_init_acl_flag_ = false;  // 非PTA初始化
    }

    // Finalize时: 谁申请谁释放
    if (repeat_init_acl_flag_) {
        NPU_CHECK_WARN(aclFinalize());
    }

Alternatives:
- 引用计数方案: ACL runtime维护init计数，finalize递减(需ACL框架层支持)
- 约定PTA永远不调用aclFinalize(依赖进程退出自动清理)
- 提供全局配置让用户声明框架共存模式

Consequence:
- 修复遵循经典的"ownership semantics": 只有调用init成功的一方有权调用finalize。
  `ACL_ERROR_REPEAT_INITIALIZE`不是错误，而是"我不是owner"的信号。
- `repeat_init_acl_flag_`在构造函数中初始化为true(乐观假设自己是owner)，
  仅在收到`REPEAT_INITIALIZE`时翻转为false。

Transferable:
- 共享资源(runtime, context, device)的init/finalize必须成对管理。
  跨框架共存场景中，每个框架需要记录自己是否是owner。
- `ERROR_REPEAT_INITIALIZE`类返回码应被解读为"资源已就绪，你不是owner"，
  而非"初始化失败"。API设计中区分error和informational return code。


---

## Ch.7 ACL接口绑定

torch_npu通过`AclInterface.cpp`(~1700行)封装Ascend Computing Language (ACL) runtime
API的动态加载与版本适配。所有CANN API调用不直接静态链接，而是经由`GET_FUNC`宏通过
`dlsym`动态查找符号，再用`XxxExist()`函数做多维度版本守卫(驱动版本、CANN runtime版本、
SoC型号)。这个包装层是torch_npu与硬件解耦的关键架构决策，但也引入了独特的缺陷模式:
版本守卫维度遗漏、前向声明缺失、日志级别不当。
本章的defect呈现高度重复的模式族，说明需要的是机制化而非逐个修复。


### 7.1 API可用性的多维版本守卫

ACL API的可用性由三个独立维度决定: 驱动版本(driver)、CANN runtime版本、SoC芯片型号。
遗漏任一维度就可能在特定组合(如新CANN + 旧driver)上触发crash。
torch_npu中至少4个独立defect因遗漏driver版本维度而产生。


#### Case 7.1.1: aclrtPointerGetAttributes -- 遗漏驱动版本守卫

Commits: f8a0f174c (D-63 fix), b56a42282 (D-708, 同一问题的重复修复)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp:1674
Defect ref: D-63, D-708

Scenario: `AclrtPointerGetAttributesExist()`用于判断"查询指针所属设备"API是否可用。
该API在CANN runtime >= 8.5.0 时符号可能存在(dlsym成功)，但实际还依赖driver >= 25.5.0。

Constraints:
- CANN runtime和driver是两个独立可升级的组件
- 用户可能升级CANN但不升级driver(或反之)
- dlsym找到符号 ≠ 底层driver支持该功能
- 无法通过运行时调用来安全探测(调用失败可能是crash而非error返回)

Decision: 在runtime版本检查前增加driver版本前置检查。当前代码结构
(AclInterface.cpp:1674-1688):

    bool AclrtPointerGetAttributesExist() {
        const static bool result = []() -> bool {
            // 守卫1: driver版本
            if (!IsGteDriverVersion("25.5.0")) { return false; }
            // 守卫2: CANN runtime版本
            if (!IsGteCANNVersion("8.5.0", "RUNTIME")) { return false; }
            // 守卫3: 符号存在性
            auto func = GET_FUNC(aclrtPointerGetAttributes);
            return func != nullptr;
        }();
        return result;
    }

三层守卫顺序: driver → runtime → dlsym。`const static`保证lambda只执行一次。

Alternatives:
- 运行时try-call探测: 调用API，捕获error code(但某些失败模式是segfault而非返回错误)
- 将版本矩阵外置为配置文件(灵活但增加部署复杂度)
- CANN提供统一的feature capability查询API(理想方案但需上游支持)

Consequence:
- D-63和D-708是同一问题在不同分支上的独立发现(间隔约600个commit)，
  说明第一次修复未被系统性推广到所有`XxxExist()`函数。
- 修复只需+4行，但需要知道API的driver依赖关系(通常不在CANN文档中显式标注)。

Transferable:
- ACL API可用性检查的标准模板: driver版本 → runtime版本 → dlsym → SoC型号。
  四个维度缺一不可。新增API时应填写"版本依赖矩阵"而非逐个试错。
- 同一bug在不同分支独立出现 → 需要仓库级的lint/检查工具，
  而非依赖开发者手动对齐所有`XxxExist()`函数。


#### Case 7.1.2: aclrtMallocHostWithCfg -- 同一模式的重复

Commits: 503381ca1 (D-79 fix)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp:1237,
  torch_npu/csrc/core/npu/NPUAllocatorConfig.cpp
Defect ref: D-79, D-719

Scenario: `AclrtMallocHostWithCfgExist()`判断host内存分配的增强API是否可用。
与D-63完全相同的模式: 只检查CANN runtime版本，遗漏driver版本。

Constraints:
- 该API需要driver >= 25.5.2(注意比D-63的25.5.0更高)
- 每个API的driver版本要求不同，无法用统一的"最低driver版本"一刀切

Decision: 同D-63，增加`IsGteDriverVersion("25.5.2")`前置检查。
同时更新用户侧的warning消息，从"cann version too low"改为
"cann version or driver version too low"。

Alternatives: 同Case 7.1.1

Consequence:
- D-63(2024年修)和D-79(后续修)是前后脚的修复。D-63修的时候没有检查其他
  `XxxExist`函数是否有同样问题，导致D-79成为可预防但未预防的缺陷。
- 这是"修一个bug不检查同类bug"的经典反模式。

Transferable:
- 修复一个`XxxExist()`函数时，必须grep所有同模式函数做对齐检查。
  理想方案是将版本守卫提取为统一的宏/模板:
  `ACL_API_EXIST_GUARD(func_name, min_driver, min_runtime, soc_range)`。
- warning消息应同步更新。"版本太低"的消息如果不告诉用户哪个版本太低(driver/runtime)，
  等于没说。


### 7.2 SoC版本范围守卫

即使API符号存在且driver/runtime版本满足，特定SoC型号也可能不支持。
版本范围需要同时声明上界和下界。


#### Case 7.2.1: SoC版本只设下界忘记上界

Commits: cd5f447d5 (D-148 fix)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp:1256
Defect ref: D-148, D-756

Scenario: `AclrtMallocHostWithCfgExist()`的SoC版本检查最初只有下界
`>= Ascend910B1`。后来新增的Ascend910_95不支持该API的VA一致性实现，
但被下界条件错误放行。

Constraints:
- SoC版本枚举是有序的: 910B1 < 910B2 < ... < 910_95 < 950
- "下界成立"不等于"所有更高版本都成立"(功能可能在中间版本移除)
- 新SoC引入时，所有现存的版本范围检查都需要重新审查

Decision: 增加上界排除。当前代码(AclInterface.cpp:1256-1257):

    return c10_npu::GetSocVersion() >= c10_npu::SocVersion::Ascend910B1 &&
           c10_npu::GetSocVersion() < c10_npu::SocVersion::Ascend950;

注: 上界从最初的`Ascend910_95`已更新为`Ascend950`(随后续SoC引入而调整)。

Alternatives:
- 白名单方式: 显式列举支持的SoC，而非用范围(更安全但维护成本高)
- 运行时feature探测: 调用API后检查返回码(需要API支持graceful failure)
- SoC capability table: 维护一个{soc → [supported_apis]}的全局配置

Consequence:
- 单方向的范围检查(只有下界)在新硬件引入时必然break。
  这不是一次性bug，而是一个持续的维护负担: 每次引入新SoC都需要审查所有范围检查。
- D-148(Ascend910_95)和D-756(同一问题的重复)证明了这个审查机制不存在。

Transferable:
- SoC版本范围���查必须同时声明上下界: `[lower, upper)`。
  如果确实"无上界"(所有未来版本都支持)，需要显式注释说明原因。
- 新SoC引入是一个系统性事件: 需要grep所有`GetSocVersion()`调用并逐一确认。
  这应该是新SoC onboarding checklist的一项。


#### Case 7.2.2: D2H拷贝路径的API升级与SoC守卫

Commits: 8bb075835 (fix, +5 cherry-picks)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp:1545,
  torch_npu/csrc/framework/OpParamMaker.cpp
Defect ref: D-290, D-781, D-813

Scenario: Device-to-Host内存拷贝从`aclrtMemcpyAsync`升级到
`aclrtMemcpyAsyncWithCondition`。新API对非ACL分配的host内存(如malloc)
自动变为同步拷贝，避免异步模式下runtime内部隐式同步的性能问题。

Constraints:
- 新API需要SoC >= Ascend910B1 + 符号存在
- 最初的`Exist()`检查只看`func != nullptr`(dlsym成功)，缺SoC版本守卫
- 低版本SoC上符号可能存在(动态库版本更新)但功能不可用
- D2H拷贝路径需要区分新旧API并记录不同的错误日志

Decision: `AclrtMemcpyAsyncWithConditionExist()`增加三层守卫
(AclInterface.cpp:1545-1563):

    const static bool result = []() -> bool {
        // 守卫1: CANN version存在性
        if (GetCANNVersion("CANN") == "") { return false; }
        // 守卫2: driver版本
        if (!IsGteDriverVersion("25.0.RC1")) { return false; }
        // 守卫3: dlsym + SoC版本
        auto func = GET_FUNC(aclrtMemcpyAsyncWithCondition);
        if (func != nullptr) {
            return GetSocVersion() >= SocVersion::Ascend910B1;
        }
        return false;
    }();

OpParamMaker中D2H路径增加条件分支:

    if (AclrtMemcpyAsyncWithConditionExist()) {
        ret = AclrtMemcpyAsyncWithCondition(dst, ...);
    } else {
        ret = aclrtMemcpyAsync(dst, ...);
    }

Alternatives:
- 统一使用旧API(放弃性能优化)
- 在运行时try-call新API，失败则fallback(需要确保失败是error code而非crash)

Consequence:
- 5次cherry-pick说明D2H���贝性能影响广泛。
- D-290(首次引入) → D-781(D2H路径错误跳过同步) → D-813(驱动版本守卫)
  是一条三步演进链: 功能引入 → 功能bug → 版本守卫补全。
  这个演进模式在torch_npu中反复出现(见Case 7.1.1/7.1.2)。

Transferable:
- 新API引入的标准流程应是: (1)版本守卫(三维) (2)功能实现 (3)fallback路径 (4)错误日志区分。
  这四步缺一步就会产生后续defect。
- "功能引入 → 功能bug → 守卫补全"的三步演进说明第一步的checklist不完整。
  如果守卫在功能引入时就写全，后两步defect可以避免。


### 7.3 头文件的显式类型依赖

C/C++头文件的include顺序隐式决定了类型可见性。
当依赖链中的某一环变化时，隐式依赖就会断裂。


#### Case 7.3.1: AclInterface.h的隐式类型依赖

Commits: 313a0cf35 (D-220 fix), f685919a2 (D-221 fix)
Files: torch_npu/csrc/core/npu/interface/AclInterface.h,
  torch_npu/csrc/core/npu/NPUException.h
Defect ref: D-220, D-221

Scenario: `AclInterface.h`使用`aclrtMemUsageInfo`和`aclOpExecutor`类型但未前向声明，
依赖`acl.h`间接提供。`NPUException.h`使用`memset`但未include `<cstring>`，
依赖其他header间接提供。

Constraints:
- CANN版本更新可能调整acl.h内部的include结构
- 独立编译单元或不同的include顺序会暴露这些隐式依赖
- 头文件应该是"自包含的"(self-contained): 单独include即可编译

Decision:
D-220: 在AclInterface.h中添加C风格前向声明:

    typedef struct aclrtMemUsageInfo aclrtMemUsageInfo;
    typedef struct aclOpExecutor aclOpExecutor;

D-221: 在NPUException.h中添加`#include <cstring>`。

Alternatives:
- 在AclInterface.h中直接include定义这些类型的具体header(更重但更稳定)
- ���用`void*`+转换代替具体类型(牺牲类型安全)

Consequence:
- D-220和D-221属于同一族(D-171/D-218也是)，说明"隐式header依赖"是系统性问题。
- 这类问题只在特定CANN版本或特定编译配置下暴露，CI覆盖率不足时容易漏过。

Transferable:
- 头文件自包含原则: 每个.h文件应该能单独编译(include它自身就够了)。
  验证方法: CI中添加"header独立编译"测试。
- C风格`typedef struct X X;`是CANN API头文件的正确前向声明方式
  (CANN的struct不使用C++ class语法)。


### 7.4 动态加载的日志治理

dlsym找不到符号是旧CANN版本的常态，告警级别应与触发频率匹配。


#### Case 7.4.1: TORCH_WARN每次调用都打印导致日志刷屏

Commits: 8be844c47 (D-580 fix), 1b5858bc1 (D-581, 相关的API回退)
Files: torch_npu/csrc/core/npu/interface/AclInterface.cpp
Defect ref: D-580, D-581

Scenario: `AclrtSynchronizeStreamWithTimeout`、`AclrtDestroyStreamForce`、
`AclrtMallocAlign32`三个函数通过dlsym动态加载。找不到时通过`TORCH_WARN`告警。
旧CANN版本上这些函数不可用是常态(不是异常)，每次调用都warn产生海量日志。

Constraints:
- `TORCH_WARN`在每次调用时都输出(设计意图: 提醒用户注意)
- 动态加载的fallback路径可能被调用数万次(每次内存分配、每次stream同步)
- 日志量大到淹没真正有用的告警信息

Decision: 将`TORCH_WARN`/`TORCH_NPU_WARN`改为`TORCH_NPU_WARN_ONCE`:

    // Before: 每次调用都打印
    TORCH_NPU_WARN(func, "Failed to find function", "aclrtSynchronizeStreamWithTimeout");

    // After: 只打印一次
    TORCH_NPU_WARN_ONCE(func, "Failed to find function", "aclrtSynchronizeStreamWithTimeout");

Alternatives:
- 只在debug级别输出(完全静默对非debug用户)
- 在`XxxExist()`中做一次性检查并缓存结果(已有的`const static`模式)

Consequence:
- D-581揭示了更深层问题: `AclrtMallocAlign32`在某些CANN版本上即使dlsym成功，
  实际也不可用。被迫全量回退为标准`aclrtMalloc`。
  这说明"符号存在 ≠ 功能可用"(与Case 7.1.1同一教训)。

Transferable:
- 可预期的运行时fallback路径使用`WARN_ONCE`而非`WARN`。
  判断标准: 这个告警在正常使用中会出现吗？如果会，就应该是ONCE。
- 日志级别选择的经验法则:
  ERROR: 不可恢复的错误
  WARN_ONCE: 可恢复但需要用户知道的降级
  WARN: 不可预期的异常(每次出现都值得关注)
  INFO: 正常但值得记录的事件


### 7.5 设备查询的上下文依赖

设备查询API不应依赖"当前设备"上下文，而应查询具体的对象属性。


#### Case 7.5.1: getDeviceFromPtr忽略指针直接返回current device

Commits: 1341fbe93 (fix)
Files: torch_npu/csrc/core/npu/NPUHooksInterface.cpp,
  torch_npu/csrc/core/npu/interface/AclInterface.cpp,
  torch_npu/csrc/core/npu/interface/AclInterface.h
Defect ref: D-232

Scenario: `NPUHooksInterface::getDeviceFromPtr(void* data)`应返回指针data
所在的设备ID。原实现直接返回`c10_npu::current_device()`，完全忽略data参数。

Constraints:
- 单卡场景下current_device()碰巧正确(只有一个设备)
- 多卡场景: tensor在device 0分配，线程切到device 1后查询返回错误的device 1
- PyTorch框架层依赖此接口做tensor设备归属判断(影响自动数据迁移)

Decision: 引入`aclrtPointerGetAttributes()` API查询指针实际归属设备:

    // Before: stub实现，忽略参数
    return {at::DeviceType::PrivateUse1, c10_npu::current_device()};

    // After: 查询指针属性
    aclrtPtrAttributes attributes;
    NPU_CHECK_ERROR(acl::AclrtPointerGetAttributes(data, &attributes));
    TORCH_CHECK(attributes.location.type != ACL_MEM_LOCATION_TYPE_HOST, ...);
    return {at::DeviceType::PrivateUse1, attributes.location.id};

配套在AclInterface中用`GET_FUNC`动态加载`aclrtPointerGetAttributes`并封装wrapper。

Alternatives:
- 维护pointer → device的全局映射表(自己记录分配关系，不依赖ACL API)
- 在分配时将device ID存入tensor metadata(PyTorch已有此机制但NPU侧未正确接入)

Consequence:
- "函数签名有参数却不使用"是明显的code smell，但作为stub实现被长期容忍。
  单卡环境的测试无法暴露此问题(current_device碰巧正确)。
- 该修复引入了对`aclrtPointerGetAttributes`的依赖，后续D-63/D-708就是
  为这个API补充版本守卫。修复链: D-232(引入API) → D-63(补driver守卫)。

Transferable:
- "stub实现"(返回当前上下文而非查询具体对象)在单设备测试中不会暴露。
  多设备测试是必要的: 在device 0分配、切换到device 1后查询。
- 函数签名中未使用的参数是一个review信号: 要么实现不完整，要么接口设计有问题。
  编译器`-Wunused-parameter`可以自动检出。


---

## Ch.8 Inductor编译器后端

torch_npu的Inductor后端将PyTorch的编译图(FX Graph)转换为NPU可执行的Triton kernel。
这一层的核心难点在于: Triton DSL的tiling/mask/index语义是为CUDA GPU设计的，
NPU的dense tile layout和axis排列与GPU不同，codegen中每一个shape变换、mask过滤、
索引分析都需要NPU特化处理。本章覆盖的缺陷集中在codegen中间表示的语义误用。


### 8.1 Mask生成与过滤

Triton kernel通过mask保护越界访存。NPU codegen需要精确控制哪些axis生成mask:
多余的mask产生非法DSL代码，缺失的mask导致越界读写。
这一领域的缺陷根因一致: codegen对新feature/模式的mask需求建模不完整。


#### Case 8.1.1: persistent_reduction模式生成了多余的reduction axis mask

Commits: 41cd21e3b (fix)
Files: torch_npu/_inductor/codegen/triton.py
Defect ref: D-108

Scenario: 开发者实现persistent_reduction模式的Triton codegen。persistent_reduction
将reduction轴数据一次性全部加载到SRAM(不需要loop)，因此reduction axis不需要mask保护。

Constraints:
- persistent_reduction与普通loop reduction共用`masks`集合构建逻辑
- `masks`基于`sorted_axis`构建，无条件包含所有axis(含`r*`前缀的reduction维度)
- 单reduction轴场景下，多余mask导致Triton DSL验证错误或精度偏差

Decision: 原实现无条件将所有axis加入mask集合。Fix: 在`persistent_reduction`
且`numof_reduction_axis() == 1`时，过滤掉`node.name[0] == "r"`的维度。

Alternatives:
- 在mask集合构建时根据reduction模式(loop/persistent/split)走不同分支(更清晰但改动面大)
- 让Triton编译器自行消除冗余mask(依赖下游优化，不可控)

Consequence: 修复前，persistent_reduction的kernel在单reduction轴场景编译失败或精度错误。
多reduction轴场景不触发(多轴时需要per-axis mask)。

Transferable:
- codegen中不同execution模式(loop/persistent/split)对mask的需求不同，
  共用的mask构建逻辑必须按模式分支。每次新增execution模式，都要回归mask生成路径。
- "无条件包含所有维度"是codegen中的common anti-pattern。Review时检查哪些维度对当前模式是多余的。


#### Case 8.1.2: masked_subblock缺少配套的mask分析pass

Commits: 1043e2655 (fix)
Files: torch_npu/_inductor/codegen/__init__.py, ir.py, triton.py
Defect ref: D-149

Scenario: codegen新增`masked_subblock`特性(子块内独立mask)。但现有的
`filter_masks`函数不知道subblock的存在，对subblock内部的load/store操作，
把它们需要的mask也过滤掉了。

Constraints:
- `filter_masks`是全局过滤，按整个kernel的axis分析哪些mask可以省略
- subblock内部的index与外层block的index有映射关系，需要独立的mask依赖分析
- 缺少从subblock index到mask变量的关联建模

Decision: 新增`generate_masked_indexing`分析pass，在`filter_masks`中根据
`subblock_axis`精确判断mask保留条件。`triton.py`中跟踪`current_subblock`状态。

Alternatives:
- subblock无条件保留所有mask(安全但生成冗余代码，可能影响性能)
- 将subblock展开为flat kernel(消除subblock概念，但丧失了subblock的优化空间)

Consequence: 修复前，使用masked_subblock的模型pattern编译后产生非法kernel。
修复需要同时改动3个文件，说明新codegen节点类型对现有分析pass的侵入性很高。

Transferable:
- codegen中新增节点类型(如subblock)时，必须检查所有现有的分析pass是否需要感知新类型。
  mask过滤、permute分析、index重命名，每一个pass都可能对新类型做出错误假设。
- 标准检查: 新增IR节点类型后，`git grep filter_masks`/`analyze_permute`等分析函数，
  逐个确认它们是否处理了新类型。


### 8.2 IR类型混淆

Inductor IR中不同节点类型共用相同的标志位或分析路径，但语义不同。
codegen如果不区分这些类型差异，就会对错误的节点类型执行不该有的变换。


#### Case 8.2.1: Scan IR节点被误当Reduction处理导致编译崩溃

Commits: fcd98896b (fix)
Files: torch_npu/_inductor/codegen/scheduling.py, triton.py, test/_inductor/test_scan.py
Defect ref: D-109

Scenario: cumsum/cumprod lowering产生的是`ir.Scan`节点(保留前缀和语义)，
但codegen把它们当作`ir.Reduction`节点处理。两者共用`inside_reduction=True`标志，
`ReductionAnalysis`初始化时强制查找`ir.Reduction`节点，对Scan节点找不到就崩溃。

Constraints:
- `inside_reduction`是Scan和Reduction的共用标志，代表"当前在reduction上下文中"
- `decide_codegen_dims_in_kernel()`和`dense_size_list()`无条件触发`ReductionAnalysis`
- NPU dense tile layout中scan轴位置不固定(`[R,X...]`或`[X...,R]`)，
  不能像upstream GPU backend假设scan轴在最后

Decision: 四项修改:
1. scan kernel关闭cooperative reduction
2. `decide_codegen_dims_in_kernel()`先用`find_reduction_node()`判空
3. `dense_size_list()`同理加判空保护
4. 新增`NPUIndexTritonKernel.scan()`，动态推断scan轴位置

Alternatives:
- 给Scan节点设置独立的`inside_scan`标志(语义更清晰但侵入upstream设计)
- 让Scan lowering不设置`inside_reduction`(会破坏其他依赖该标志的逻辑)

Consequence:
- 修复前cumsum/cumprod模型无法用inductor编译。修复后新增test_scan_npu测试覆盖。
- 根因是`inside_reduction`标志的语义歧义: 它表达的是"reduction上下文"而非"Reduction节点"，
  但所有消费方都假设它意味着`ir.Reduction`存在。

Transferable:
- IR中共用标志位(inside_reduction/inside_scan)的语义必须文档化。
  每次新增IR节点类型，检查所有以该标志为条件的代码路径。
- "标志位语义膨胀"是编译器codegen中的系统性风险:
  一个标志从"Reduction专用"悄悄变成"Reduction或Scan"，但消费方未跟进。


#### Case 8.2.2: index_expr与普通索引共用分析路径导致4处错误

Commits: 51dfb9339 (fix)
Files: torch_npu/_inductor/codegen/kernel_analysis.py, triton.py
Defect ref: D-318

Scenario: codegen的`IndexAnalysis`没有区分`index_expr`(标量索引表达式)
和普通load/store索引，对index_expr执行了4类不该有的变换:
1. `analyze_permute_shape()`对index_expr做了permute shape分析(index_expr不参与tile permute)
2. `sympy_index_symbol`对index_expr变量做了后缀重命名(`x_0`→`x_1`)，破坏原始语义
3. `NPUTritonKernelOverrides`缺少`index_expr`覆写
4. store cache命中时对index_expr做了多余的permute分析

Constraints:
- index_expr与load/store索引在IR中的表示类似(都是sympy表达式)
- 分析pass按"所有索引"统一处理，没有类型区分机制
- permute分析和符号重命名对普通索引是正确的，但对index_expr是破坏性的

Decision: 新增`is_index_expr`标志全链路透传。各分析pass对index_expr分支处理:
- `analyze_permute_shape()`: index_expr直接return
- 变量重命名: index_expr保持原始符号名
- 新增`NPUTritonKernelOverrides.index_expr()`
- store cache命中时跳过index_expr的permute分析

Alternatives:
- index_expr用独立的数据结构(非sympy表达式)表示(类型安全但重构量大)
- 分析pass只处理显式注册的索引类型(白名单机制，更安全但需要维护注册表)

Consequence: 修复前，包含index_expr的模型(如`torch.arange`生成的索引)编译后
产生错误的kernel代码，表现为数值计算结果错误。4处错误分散在codegen核心路径的
不同层次，单个review很难一次性发现。

Transferable:
- 共用分析路径中引入新的索引类型时，必须审计所有现有pass是否需要区分处理。
  标准做法: `git grep analyze_permute\|index_symbol\|Overrides`检查所有消费方。
- "代码看起来对，因为所有索引都是sympy表达式"的推理方式是危险的。
  相同的数据表示不意味着相同的语义契约。


### 8.3 Triton DSL Shape变换语义

Triton DSL中`tl.broadcast_to`和`tl.reshape`的语义不同:
broadcast_to要求源维度与目标维度在非扩展轴上一致，reshape只要元素总数匹配即可。
选错了原语会在特定索引pattern下失败。


#### Case 8.3.1: reduction前shape提升误用broadcast_to

Commits: 7ee5d115d (fix)
Files: torch_npu/_inductor/codegen/triton.py, test/_inductor/test_pattern_44.py
Defect ref: D-278

Scenario: reduction前需要将输入tensor shape提升到与tile shape匹配。
codegen使用`tl.broadcast_to`做这一变换。但当输入来自非连续索引(需要permute/reshape)时，
broadcast_to的维度兼容性约束不满足(源维度与目标维度不对齐)。

Constraints:
- `tl.broadcast_to(x, shape)`: 要求x的每个维度要么等于shape对应维度，要么为1
- 非连续索引的load结果shape可能与tile shape不是简单的broadcast关系
- `tl.reshape(x, shape)`: 只要元素总数匹配即可，更宽松

Decision: 将`tl.broadcast_to`改为`tl.reshape`。新增从load/store索引反推
正确shape变换的逻辑(分析哪些维度是contiguous的)。

Alternatives:
- 在load时强制做contiguous化(确保broadcast总是合法，但可能引入多余拷贝)
- 根据索引pattern动态选择broadcast_to或reshape(更精确但逻辑更复杂)

Consequence: 修复前，特定pattern(非连续reduction输入)编译失败。
修复引入了更通用的shape变换逻辑，但增加了codegen的复杂度。

Transferable:
- Triton codegen中shape变换原语的选择不是随意的。`broadcast_to`有严格的维度约束，
  `reshape`更宽松但丧失了broadcast的语义信息。选错会导致运行时错误而非编译期错误。
- 测试必须覆盖非连续索引pattern: 这些pattern在简单模型中不出现，
  但在复杂的attention/convolution中很常见。


### 8.4 Codegen状态管理

codegen在遍历IR节点时积累状态(字典、集合)。循环中的状态更新如果缺少幂等性保护，
后续迭代会覆盖先前正确的值。


#### Case 8.4.1: rebuild_flattened_dim字典写入覆盖先前正确映射

Commits: 4d1bc797f (fix)
Files: torch_npu/_inductor/codegen/ir.py
Defect ref: D-316

Scenario: `rebuild_flattened_dims`遍历axis时，对`V.kernel.expr_substituted`字典写入
符号替换映射。flattened维度分解中，先遍历到的映射(最内层axis)才是正确的。
但代码直接赋值，后续迭代对同一key的写入覆盖了先前正确的值。

Constraints:
- `expr_substituted`字典记录sympy表达式到kernel符号的映射
- flattened维度分解的语义要求"first-write-wins": 最内层axis的映射优先
- 循环中相同expr可能被多个axis产生(维度共享sympy表达式)

Decision: 添加`if expr not in V.kernel.expr_substituted:`守卫。单行修改。

Alternatives:
- 反转循环顺序(从外层到内层)，让最后一次写入正确(但破坏其他依赖循环顺序的逻辑)
- 使用OrderedDict + setdefault(语义更显式，但对性能敏感的codegen路径有开销)

Consequence: 修复前，特定的高维flattened tensor codegen产生错误的索引计算。
单行修改，reviewability高。

Transferable:
- codegen遍历中对字典/集合的写入必须声明语义: "first-write-wins"还是"last-write-wins"。
  默认的`dict[key] = value`是last-write-wins，如果语义要求first-write-wins，
  必须加`if key not in dict`守卫或使用`setdefault`。
- 这是一个通用的循环不变量问题，不限于codegen:
  任何循环内的字典赋值都应检查是否需要幂等性保护。


### 8.5 CUDA→NPU Heuristics适配

Inductor的kernel tuning和grid计算使用CUDA-specific的heuristics基类。
NPU后端如果不替换这些基类，会触发CUDA runtime操作。


#### Case 8.5.1: user_autotune直接使用CUDA Grid/heuristics导致core dump

Commits: 7f4be1158 (fix)
Files: torch_npu/_inductor/codegen/triton.py, wrapper.py,
  npu_triton_heuristics.py, test/_inductor/test_user_autotune_npu.py
Defect ref: D-712

Scenario: PyTorch inductor的`user_autotune`允许用户自定义kernel tuning参数。
其实现使用CUDA的`triton_heuristics.user_autotune`和`PrecomputedGrid`基类，
这些类内部调用CUDA runtime API(如获取SM数量)。在NPU上调用导致core dump。
另外`gen_triton_ext_imports`缺少`@staticmethod`装饰器，wrapper codegen中
作为类方法调用时传入多余的self参数。

Constraints:
- user_autotune的Grid计算逻辑硬编码了CUDA的grid维度映射
- Inductor的codegen pipeline通过字符串模板生成Python代码，
  heuristics类名嵌入在模板字符串中
- NPU的grid计算使用不同的维度映射

Decision: 在`define_kernel`中做字符串替换:
`triton_heuristics.user_autotune(` → `npu_triton_heuristics.user_autotune_npu(`。
`PrecomputedGrid` → `PrecomputedGridNpu`。注入NPU ext imports。
补充`@staticmethod`装饰器。

Alternatives:
- 在Triton level做设备抽象(让Grid/heuristics接受设备参数，而非硬编码CUDA)
  -- 更优雅但需要修改upstream Triton
- 运行时检查设备类型并分派(增加运行时开销)

Consequence: 修复前user_autotune在NPU上直接core dump(CUDA API segfault)。
修复后可正常使用NPU的autotune配置。新增test_user_autotune_npu验证。

Transferable:
- Inductor codegen的扩展点: 每次upstream新增CUDA heuristics引用时，
  NPU后端必须确认替代路径存在。标准做法: `git grep triton_heuristics\\.`检查所有引用点。
- 字符串替换是codegen适配的pragmatic solution，但脆弱:
  upstream改变模板格式就会失效。长期应推动upstream支持设备参数化。
- `@staticmethod`遗漏在wrapper函数中不会编译报错，只在运行时因多余self参数而崩溃。


---

## Ch.9 构建系统与跨平台兼容

torch_npu的构建涉及多层依赖链: Python setuptools → CMake → GCC/Clang → CANN SDK(libhccl/libascendcl)。
链接属性、编译flag、动态库加载路径中的任何一环出错，都可能导致构建失败或运行时crash。
本章覆盖的问题多数在"主开发环境"中不可见，而在CI、客户环境、不同芯片平台上才暴露。


### 9.1 链接依赖管理

CMake的PUBLIC/PRIVATE/INTERFACE链接属性决定符号是否传递给下游consumer。
torch_npu被op_plugin等外部插件链接，链接可见性的变更影响面超出torch_npu自身的测试范围。


#### Case 9.1.1: 删除PUBLIC链接导致下游.so的符号丢失

Commits: 对应D-3
Files: CMakeLists.txt, setup.py
Defect ref: D-3

Scenario: 有人试图清理`libtorch_npu.so`的依赖列表，将`libhccl.so`从PUBLIC链接
改为PRIVATE(或直接移除)。在torch_npu自身的测试中一切正常，因为torch_npu内部
不直接调用hccl符号。但下游的op_plugin.so通过torch_npu间接链接hccl，
PUBLIC→PRIVATE后传递链接断裂，op_plugin运行时报undefined symbol。

Constraints:
- CMake的PUBLIC/PRIVATE/INTERFACE三种链接属性决定了是否传递给下游target
- 构建时不报错(链接器只检查直接依赖)
- 运行时dlopen时才触发符号解析失败

Decision: 恢复PUBLIC链接。同时回退setup.py中patchelf删除libhccl rpath的操作。
跨多个release分支cherry-pick修复。

Consequence: op_plugin.so在运行时报`undefined symbol`崩溃。构建阶段不暴露
(链接器只检查直接依赖)，仅在运行时dlopen时触发。需跨多个release分支cherry-pick修复。

Transferable:
- 修改shared library的链接可见性前，必须用`ldd -r`检查所有下游consumer的符号完整性。
- "构建通过≠运行通过"在动态链接场景中极为常见。CI应包含运行时符号完整性检查。


#### Case 9.1.2: 新CANN API直接静态链接导致旧环境运行崩溃

Commits: 对应D-24, D-111
Files: AclInterface.cpp, Module.cpp, third_party/acl headers
Defect ref: D-24, D-111

Scenario: 开发者在代码中直接调用新版CANN SDK的API(如`aclrtGetDeviceInfo`)，
并在头文件中声明了对应的函数原型和枚举值。在有新SDK的开发机上编译链接成功。
但部署到旧版CANN环境时，`dlopen`加载torch_npu.so时找不到符号，直接segfault。

D-111更严重: 新API声明被cherry-pick到6个release分支，导致所有分支都无法在旧CANN上编译。
需要在6个分支上同步回滚。

Decision: 所有CANN API通过`GET_FUNC`/`LOAD_FUNC`宏动态加载(dlsym)，
运行时检测API是否存在，不存在时走降级路径。

Consequence: D-24: 部署到旧版CANN环境时dlopen失败，segfault。
D-111: cherry-pick到6个release分支后所有分支在旧CANN上编译失败，需要批量回滚。
后者的影响面是前者的6倍，因为cherry-pick扩大了破坏半径。

Transferable:
- 对外部SDK的新API，永远通过dlsym动态加载而非编译时链接。
  编译环境的SDK版本不等于运行环境的SDK版本。
- cherry-pick前必须确认目标分支的SDK版本兼容性。


### 9.2 编译flag与平台兼容

torch_npu需要在多种gcc版本和硬件平台上编译。编译flag的硬编码是跨平台兼容性的常见陷阱。


#### Case 9.2.1: gcc版本差异导致Inductor JIT编译失败

Commits: 对应D-4
Files: Inductor codegen, cpp_builder.py
Defect ref: D-4

Scenario: Inductor C++ codegen后端编译kernel时传入`-march=native`等优化flag。
在gcc 12上正常，在A3平台的gcc 13上不被识别，编译失败。

Decision: 实现compiler feature detection:
运行`gcc -march=native -x c++ -fsyntax-only /dev/null`测试flag是否支持，
不支持的flag自动降级或移除。

Consequence: Inductor JIT编译在特定gcc版本+硬件组合上100%失败，NPU训练无法启动。
仅在A3平台gcc 13环境触发，主开发环境(gcc 12)无法复现。

Transferable:
- 编译flag不能硬编码，必须做运行时检测。
- 跨平台项目应在CI中覆盖所有目标gcc版本。


### 9.3 打包与版本号

setup.py负责源码打包和版本号生成。editable/非editable安装模式的差异、
无git环境、增量构建等场景容易被遗漏。


#### Case 9.3.1: setup.py遗漏MLIR源文件导致pip安装后JIT编译失败

Commits: D-139
Files: setup.py
Defect ref: D-139

Scenario: MLIR编译器的C++源文件未列入`package_data`。`pip install`后文件缺失，
Inductor JIT编译找不到头文件。开发者用`pip install -e .`(editable mode)不受影响
(直接引用源码目录)。

Decision: 将MLIR相关源文件和头文件添加到setup.py的`package_data`中。

Consequence: 非editable模式安装的用户使用Inductor JIT时报`FileNotFoundError`。
开发者本地用`pip install -e .`无法复现，因为editable模式直接引用源码目录。

Transferable:
- `pip install -e .`和`pip install .`的差异是常见的打包陷阱。
  CI应在非editable模式下跑完整测试。


#### Case 9.3.2: 无git环境下版本号不合法

Commits: D-850
Files: setup.py
Defect ref: D-850

Scenario: `get_sha()`在无`.git`目录的环境(Docker镜像、tarball)中返回`"Unknown"`，
拼接后版本号`2.1.0+gitUnknown`违反PEP 440。pip拒绝安装。

Decision: 检查`sha == "Unknown"`时不追加git后缀。

Consequence: Docker镜像和tarball构建环境中pip拒绝安装torch_npu，报版本号格式错误。
CI环境通常有git但生产部署环境通常没有，导致开发阶段无法发现。

Transferable:
- 版本号生成必须处理无git的场景。PEP 440对local version specifier有严格格式要求。


#### Case 9.3.3: setup.py增量构建因O_EXCL标记失败

Commits: D-1210
Files: setup.py
Defect ref: D-1210

Scenario: `generate_torch_npu_version()`用`os.O_CREAT | os.O_EXCL`创建version文件，
O_EXCL要求文件不存在。第二次构建时文件已存在，触发`FileExistsError`。

Decision: 改用`O_CREAT`(无O_EXCL)或先检查文件存在性。

Consequence: 增量构建(第二次`python setup.py`)即报`FileExistsError`，
开发者被迫每次清空build目录后全量重新构建，严重影响开发迭代效率。

Transferable:
- 构建脚本的文件操作必须幂等。`O_EXCL`适用于lock file，不适用于生成文件。


### 9.4 符号导出与API可见性

`-fvisibility=hidden`编译选项下，只有显式标记`TORCH_NPU_API`的函数才对外部.so可见。
遗漏标记的API在单元测试中正常(同一.so内)，在外部consumer编译时才暴露。


#### Case 9.4.1: 缺少TORCH_NPU_API宏导致外部plugin链接失败

Commits: D-46
Files: C++头文件
Defect ref: D-46

Scenario: `apply_tensor_without_format()`在头文件中声明但未标记`TORCH_NPU_API`。
torch_npu自身使用正常(同一.so内)。op_plugin作为独立.so链接时，找不到该符号。

Decision: 给所有公开API添加`TORCH_NPU_API`宏。

Consequence: op_plugin编译时报`undefined reference`。torch_npu自身测试全部通过
(同一.so内符号可见)，仅外部consumer编译时暴露。

Transferable:
- `-fvisibility=hidden`下，忘记标记导出宏的API在同.so内可用但跨.so不可见。
  这类bug只在外部consumer编译时暴露，单元测试无法覆盖。
  CI应包含"从外部.so链接所有public API"的集成测试。


### 9.5 CANN版本兼容

torch_npu需要支持多版本CANN SDK。直接链接新版API会破坏旧版运行环境，
是多版本支持的核心矛盾。


#### Case 9.5.1: 引入不存在的CANN API导致多分支回滚

Commits: D-111
Files: third_party/acl headers, CMakeLists.txt
Defect ref: D-111

Scenario: 开发者在acl头文件中声明了新版CANN的API符号(如`aclrtPointerGetAttributes`)
并在代码中直接调用。main分支编译通过(使用最新CANN)。cherry-pick到release分支后，
旧CANN版本没有这些符号，链接失败。6个分支都需要回滚。

Decision: 新CANN API一律通过dlsym运行时加载，不在编译期引入符号依赖。

Consequence: 6个release分支在旧版CANN环境上链接失败，需逐个分支手动回滚
cherry-pick的新API声明。回滚的人力成本远超原始修复。

Transferable:
- 多版本CANN支持的架构要求: 所有CANN API调用都应通过AclInterface的
  动态加载层(GET_FUNC宏)间接访问。直接调用=版本耦合。
- cherry-pick前必须在目标分支的CANN版本上验证编译。


### 9.6 头文件与间接include

C++头文件的间接include链随编译器版本变化。显式include所有直接使用的标准库头文件
是唯一可靠的策略。


#### Case 9.6.1: 缺失include导致概率性编译失败

Commits: D-171
Files: torch_npu/csrc/core/npu/NPUException.h
Defect ref: D-171

Scenario: `NPUException.h`使用`memset`但只include了`<string>`而非`<cstring>`。
在gcc 12中`<string>`间接include了`<cstring>`所以能编译通过，
gcc 13去掉了这个间接include，编译失败。

Decision: 显式`#include <cstring>`。

Consequence: gcc 12→13升级后编译失败，错误指向`memset`未声明。
根因是间接include链断裂，但错误信息不直接指向缺失的`#include <cstring>`。

Transferable:
- 头文件必须显式include所有直接使用的标准库头文件，不依赖间接include。
  间接include链会随编译器版本变化。


---

## Ch.10 序列化与模型加载

torch_npu对`torch.save`/`torch.load`进行了wrapper，处理NPU tensor的设备映射、
ACL内存格式、以及torch.compile(FakeTensor)场景下的序列化兼容。
这一层的bug在训练中不触发(训练不做序列化)，只在checkpoint保存/加载、
模型导出、分布式迁移等场景出现。


### 10.1 FakeTensor序列化兼容

PyTorch的FakeTensor是torch.compile中的trace-time占位符，没有实际设备storage。
torch_npu的序列化路径在多处假设tensor持有真实的NPU storage，
导致FakeTensor场景下系统性崩溃。


#### Case 10.1.1: _rebuild_npu_tensor在FakeTensor模式下访问不存在的设备storage

Commits: 对应D-12
Files: torch_npu/utils/storage.py
Defect ref: D-12

Scenario: `torch.load()`反序列化NPU tensor时调用`_rebuild_npu_tensor()`，
内部执行`storage.npu(device)`将数据搬到NPU。FakeTensor没有真实storage，调用失败。

Decision: 在`_rebuild_npu_tensor`入口检测FakeTensorMode，
fake模式下设置`fake_device`属性代替真实设备迁移。

Consequence: torch.compile流水线中torch.load反序列化NPU模型时crash。
影响所有使用torch.compile + checkpoint的训练流程。

Transferable:
- 序列化/反序列化路径必须考虑FakeTensor场景(torch.compile流水线会用fake tensor跑trace)。
  `isinstance(tensor, FakeTensor)`或`detect_fake_mode()`是标准检测方式。


#### Case 10.1.2: save()中访问fake tensor的真实storage导致crash

Commits: 对应D-192
Files: torch_npu/utils/serialization.py
Defect ref: D-192

Scenario: `_npu_save()`的逻辑先构造tensor的storage视图(访问真实数据)，
再检查是否skip_data。FakeTensor没有真实storage，先访问后检查的顺序导致crash。

Decision: 将skip_data/fake_mode检查提前到storage访问之前。

Consequence: torch.compile trace过程中执行checkpoint保存时crash。
"先访问后检查"的顺序在正常eager路径下无害，但FakeTensor路径下致命。

Transferable:
- "先检查后访问"是防御性编程的基本模式。序列化代码特别容易违反，
  因为正常路径总是有真实数据。


### 10.2 参数透传与API兼容

torch_npu的save/load wrapper需要将所有参数忠实地传递给upstream torch函数。
每次upstream新增参数，wrapper都需要同步更新。


#### Case 10.2.1: save()硬编码参数覆盖用户配置

Commits: 对应D-86
Files: torch_npu/utils/serialization.py
Defect ref: D-86

Scenario: `_npu_save()`内部调用`torch.save()`时，将`_use_new_zipfile_serialization`
参数硬编码为`True`，忽略用户传入的值。需要使用旧格式的场景(如兼容旧版本PyTorch)失效。

Decision: 将硬编码改为透传用户参数。

Consequence: 需要旧格式序列化的用户(跨PyTorch版本兼容)的参数设置被静默忽略，
保存格式始终为新版zipfile格式。用户无报错提示。

Transferable:
- wrapper/monkey-patch函数必须透传所有用户参数。
  用硬编码常量替代可配置参数是常见的copy-paste错误。
  Review时检查: wrapper的每个参数是否都被传递给底层函数。


#### Case 10.2.2: torch.load对weights_only/mmap组合处理不完整

Commits: D-520
Files: torch_npu/utils/serialization.py
Defect ref: D-520

Scenario: PyTorch 2.6+引入`weights_only`默认值变更、mmap加载等新特性。
torch_npu的`_npu_load()`只处理了基本case，多个参数组合未覆盖。

Decision: 重写load wrapper，完整支持weights_only/mmap/legacy格式矩阵。

Consequence: PyTorch 2.6+的weights_only默认值变更后，NPU模型加载行为与CPU/CUDA不一致，
触发unexpected keyword argument或行为静默偏差。

Transferable:
- 对upstream API的wrapper，每次upstream版本升级后必须review参数列表变更。
  使用`**kwargs`透传可减少同步负担。


### 10.3 序列化辅助功能

涉及Python对象的可序列化性(pickle)问题，影响分布式训练和torch.compile缓存。


#### Case 10.3.1: closure不可pickle导致torch.compile cache失效

Commits: D-49
Files: torch_npu/dynamo/__init__.py
Defect ref: D-49

Scenario: torch_npu注册给dynamo的backend函数是Python closure(捕获外部变量)。
分布式场景下pickle序列化backend函数时失败，因为closure不可pickle。

Decision: 将closure提升为模块级函数，用全局变量替代捕获变量。

Consequence: 分布式训练中torch.compile的backend函数无法跨进程pickle传递，
multi-GPU编译失败。单GPU场景不触发(无需pickle)。

Transferable:
- 注册到框架的callback必须是module-level函数，不能是closure或lambda。
  Python pickle不能序列化closure(因为closure绑定了自由变量的cell对象)。


---

## Ch.11 测试基础设施

测试相关的缺陷主要来源于: 与upstream PyTorch API的同步断裂、
测试覆盖度不足(缺少数据正确性断言)、以及skip/disable机制的管理失控。


### 11.1 Upstream测试API漂移

PyTorch upstream频繁重命名内部测试模块和函数。torch_npu直接import这些`_internal`模块，
每次rebase都有断裂风险。


#### Case 11.1.1: upstream改名/移动后import失败

Commits: D-39, D-47, D-61, D-173
Files: test/目录

Scenario: PyTorch upstream频繁重命名内部测试辅助函数和模块路径。
每次upstream rebase后，torch_npu的测试import链断裂，CI红。
例如`additional_module`从`torch.testing._internal`移到`torch.testing`。

Pattern: 典型的"追赶上游"问题。torch_npu直接import upstream的_internal模块，
没有抽象层缓冲。

Consequence: 每次rebase upstream后测试CI即红。故障表现为`ImportError`或`AttributeError`，
需要人工追踪upstream的rename/move记录并逐一修复import路径。

Transferable:
- 依赖upstream `_internal`模块的代码，必须在`try/except ImportError`中
  提供fallback import路径。或者将常用测试工具提取到torch_npu自己的test_utils中。


### 11.2 测试Skip机制的管理

`@skip`装饰器是必要的测试管理工具，但失控积累会降低测试覆盖率的真实性。
skip的生命周期管理是测试基础设施的核心问题。


#### Case 11.2.1: skip装饰器过期未清理

Commits: D-40
Files: test/目录

Scenario: 某bug被修复后，对应的`@skipIf(True, "bug #xxx")`未同步移除。
测试套件中积累大量已失效的skip，实际测试覆盖率低于表面数字。

Decision: 建立skip清理规范: 每个@skip必须附带issue URL，定期扫描已关闭issue对应的skip。

Consequence: 测试覆盖率表面数字高于实际。大量已修复bug对应的skip未移除，
回归保护实质缺失，已修复的bug有可能悄悄回归而不被发现。

Transferable:
- `@skip`是技术债。CI应有"skip audit"步骤:
  统计skip数量趋势，对新增skip要求issue链接。


#### Case 11.2.2: skipIfUnsupportMultiNPU用return而非raise，测试静默通过

Commits: 对应D-233
Files: test/npu/test_utils.py
Defect ref: D-233

Scenario: `skipIfUnsupportMultiNPU`装饰器在单卡环境中用`return`而非`raise SkipTest`。
unittest框架将`return`视为"测试通过"，不报skip也不报fail，测试静默通过。

Decision: 改为`raise unittest.SkipTest(...)`。

Consequence: 单卡环境中多卡测试全部显示"通过"而非"skip"，
掩盖了测试覆盖的真实空白。直到有人检查测试执行日志才发现异常。

Transferable:
- 自定义skip装饰器必须使用`raise SkipTest`而非`return`。
  `return`会导致测试框架误报"通过"。


### 11.3 数值断言陷阱

NPU算子测试中常见的反模式是"只测不crash，不测正确性"。
缺少数值断言的测试给人虚假的安全感。


#### Case 11.3.1: 测试只验证"不crash"但不验证数值正确性

Defect ref: D-68

Scenario: 开发者为算子编写测试用例，仅验证执行不抛异常，未校验计算结果的数值正确性。
典型模式: `output = model(input); torch.npu.synchronize()` 之后无任何数值断言。

Decision: 要求测试用例必须包含数值正确性断言，将NPU输出与CPU参考输出对比。

Pattern: `output = model(input); torch.npu.synchronize()` 没有任何assert。
验证"不crash"但不验证计算结果。一个返回全零的broken kernel也能通过。

Consequence: 返回全零的broken kernel也能通过测试。
bug只能在用户的训练workload中通过loss异常才被发现，此时距引入已很久。

Transferable:
- 每个算子测试必须包含数值正确性断言。
  至少: `torch.allclose(npu_output.cpu(), cpu_output, atol=..., rtol=...)`


---

## Ch.12 横切模式

跨章节的共性缺陷pattern，值得在code review中主动检查。


### 12.1 函数调用缺少括号

Defect refs: D-16, D-37, D-178

Pattern: `if is_supported:` 其中 `is_supported` 是函数而非变量。
函数对象是truthy的，条件永远为True。

Detection: `pylint --enable=using-constant-test`
或 `mypy`（如果有正确的类型标注）。


### 12.2 版本号比较错误

Defect ref: D-50

Pattern: `major >= X and minor >= Y` (应为 `(major, minor) >= (X, Y)`)

`major=9, minor=0` 对 `>= (8, 1)` 应为True，
但 `9 >= 8 and 0 >= 1` 为False。

用元组比较替代分量比较。


### 12.3 Copy-paste变量未修改

Defect ref: D-288, D-310

Pattern: 复制相邻行后修改了左侧但忘记修改右侧(或反之)。
D-288: `config_a.worldSize = worldSize` 复制为 `config_b.rankSize = worldSize`
(应为`rankSize`)。

Detection: 相邻赋值语句中右值完全相同时应review。


### 12.4 环境变量/字符串key拼写

Defect ref: D-993, D-310

Pattern: 字符串key(`os.environ["TOCRH_NPU"]`)拼写错误，
Python不报错(返回KeyError或None)，功能静默失效。

Mitigation: 环境变量名定义为常量，集中管理。


---

## Ch.13 Revert-Driven决策教训(补遗)

Ch.1-12的case主要从bugfix维度提取决策要素。本章补充完全由revert驱动的case:
开发者做出合理的优化/重构决策，但因隐性约束或副作用被迫回滚。
这类case的教育价值在于"决策弧"完整可见: 意图→实现→失败信号→回滚→教训。

本章覆盖的9个revert主题，横跨内存架构、算子实现、生命周期管理、编译器后端、
profiler功能添加五个领域。共性模式在13.5节总结。


### 13.1 内存架构重构的回滚

NPUCachingAllocator是torch_npu中改动最密集的单文件之一(919次touch)。
围绕它的架构级重构(非局部bugfix)产生了两个完整的revert cycle。


#### Case 13.1.1: per-device allocator重构 -- BlockComparator丢失device维度

Commits: ef47a4d8e (revert)
Files: NPUCachingAllocator.cpp/h, Module.cpp, memory.py, __init__.py, test_npu.py
Original: 将全局`THNCachingAllocator`重构为per-device `DeviceCachingAllocator`架构

Scenario: 开发者照搬CUDA CachingAllocator的per-device架构，为每个NPU设备创建独立的
DeviceCachingAllocator实例，支持`set_per_process_memory_fraction`按设备限制显存占比。
这是allocator架构的重大演进: 从全局单例+device字段筛选，到per-device实例+无需device字段。

Constraints:
- 原始全局架构中，`BlockComparator`(block排序函数)首先按device排序，保证跨设备block不混排
- per-device架构中，每个allocator只管自己设备的block，理论上不需要device比较
- 但全局block pool(free blocks)仍然共享，block在设备间迁移时仍需device区分

Decision(原始): 删除BlockComparator中的device比较，引入per-device stats。
Decision(revert): 恢复全局allocator + device字段排序。

    // revert后恢复的关键代码
    static bool BlockComparator(const Block* a, const Block* b) {
      if (a->device != b->device) {     // 跨设备排序
        return a->device < b->device;
      }
      if (a->stream != b->stream) { ... }

Alternatives:
- 彻底拆分: 每个device一个独立的block pool(不共享)，但浪费内存
- 保留device字段但使用per-device allocator实例(两层结构)
- 仅添加set_per_process_memory_fraction而不重构allocator架构

Consequence: 多设备场景下block排序错误，导致allocator的best-fit策略选到错误设备的block。
单设备环境不触发(只有一个device值)，多卡训练场景复现。
revert后set_per_process_memory_fraction API也一并移除。

Transferable:
- "理论上不需要"不等于"可以安全删除"。删除冗余字段前必须验证所有消费者。
  per-device架构确实使allocator实例不跨设备，但全局block pool仍然跨设备。
  架构重构的"理论分析"必须覆盖所有数据结构的使用点，而非仅覆盖主路径。
- 照搬CUDA allocator架构时需注意: CUDA CachingAllocator的per-device实例各自维护
  独立的block set(完全隔离)，NPU的实现可能共享了底层结构。


#### Case 13.1.2: event reconfiguration -- EventPool+引用计数的三位一体重构

Commits: 662d25462 (revert)
Original: 2220d272a "[feat] event reconfiguration"
Files: 15个文件(NPUCachingAllocator, NPUEvent, NPUEventManager, THNPUCachingHostAllocator,
  ProcessGroupHCCL, OpParamMaker, NPUGuardImpl, AsyncTaskQueueInterface, acl_rt.h等)

Scenario: 开发者尝试统一解决event生命周期管理的多个痛点:
(1) raw aclrtEvent创建/销毁开销 → 引入EventPool对象池
(2) npu_events的线性扫描 → 改为per-stream flat_hash_map分桶
(3) recorded_events集合的mutex争用 → 改为引用计数(IncreaseUnrecordedCount/DecreaseUnrecordedCount)
(4) 新增ACL_EVENT_CAPTURE_STREAM_PROGRESS flag

Constraints:
- event管理涉及allocator、ProcessGroup、EventManager三个子系统的交互
- recorded_events集合是异步task queue模式下判断"event是否已被ACL实际record"的关键门控
- ACL_EVENT_CAPTURE_STREAM_PROGRESS flag依赖底层ACL版本支持
- 15个文件的联动变更意味着回滚窗口极窄

Decision(原始): 用EventPool+引用计数+per-stream分桶替代raw event+recorded_events集合。
Decision(revert): 完整回滚至raw aclrtEvent + recorded_events集合。

    // revert恢复的核心数据结构
    std::deque<std::pair<aclrtEvent, Block*>> npu_events;  // 线性队列
    std::set<aclrtEvent> recorded_events;                   // recorded门控

    // 而非event reconfiguration的:
    ska::flat_hash_map<NPUStream,
      std::deque<std::pair<EventPool::Event, Block*>>> npu_events;

Alternatives:
- 分步重构: 先只做EventPool(减少创建开销)，不改数据结构
- 只做per-stream分桶(减少线性扫描)，不改recorded判断机制
- 保留recorded_events集合但用读写锁替代mutex

Consequence: 三个独立优化目标绑定在一个commit中，任何一个子目标的回归都导致全量回滚。
ACL_EVENT_CAPTURE_STREAM_PROGRESS flag不被当前ACL版本支持是直接触发因素。

Transferable:
- 大型重构的原子性困境: 15个文件的联动变更无法部分回滚。
  正确策略是拆成3个独立PR: EventPool → per-stream分桶 → 引用计数替代recorded_events。
  每个PR独立验证，独立回滚。
- 引入新的硬件/SDK feature flag时，必须确认目标SDK版本矩阵。
  ACL_EVENT_CAPTURE_STREAM_PROGRESS的可用性应在PR描述中声明最低SDK版本要求。
- 与Case 1.1.2(recordStream/eraseStream演化)同源: 都是allocator-event交互的复杂度管理问题。
  recorded_events集合后来被Case 2.4.x中的独立fix(f0bbf40d0)单独恢复，证明拆分策略的正确性。


### 13.2 算子路径分流与Format优化

算子实现中"添加快速路径"是常见优化手段，但快速路径的进入条件判断不完备时，
会导致部分输入走错路径，产生静默的精度错误。


#### Case 13.2.1: format_contiguous合并 -- copy_optimize的安全边界

Commits: c92556c2e (revert, +4 cherry-picks跨5分支)
Original: 4c0413536 "Combine format_contiguous with format_contiguous_add_copy_optimize"
Files: NpuUtils.cpp/h, AddKernelNpu.cpp, BmmKernelNpu.cpp, OpCommand.cpp
Author: chuboning (同一作者提交+revert)

Scenario: NPU框架存在两个功能相似的函数:
`format_contiguous`(安全路径，不做copy优化)和
`format_contiguous_add_copy_optimize`(快速路径，跳过冗余copy)。
开发者合并两者，让统一的format_contiguous默认走copy_optimize路径。

Constraints:
- copy_optimize的判断条件: `numel == product(base_sizes)` → 跳过拷贝
- 该条件在padding后的format转换场景下不成立(NZ format的base_sizes包含padding)
- 只有add/bmm/mm算子的调用方知道自己的tensor不会有padding
- 通用路径(OpCommand::Contiguous)的tensor可能带任意format

Decision(原始): 将copy_optimize逻辑合入format_contiguous，所有调用者默认享受优化。
Decision(revert): 恢复两个独立函数。通用路径用format_contiguous(无优化)，
  只有add/bmm/mm显式调用format_contiguous_add_copy_optimize。

    // revert后: 通用路径不做copy优化
    at::Tensor NpuUtils::format_contiguous(const at::Tensor &src) {
        return metadata_convert_match_without_copy_optimize(src);
    }

    // 只有特定算子使用带优化的版本
    at::Tensor NpuUtils::format_contiguous_add_copy_optimize(const at::Tensor &src) {
        return metadata_convert_match_with_copy_optimize(src);
    }

Alternatives:
- 在copy_optimize内部增加padding检测(检查actual storage size vs logical size)
- 用opt-in而非opt-out策略: 默认不优化，算子显式标记"我的输入是安全的"

Consequence: 合并后，经过NZ format padding的tensor在format_contiguous时
跳过了必要的拷贝，下游算子读到未初始化或错位的数据。
精度错误在特定shape(触发padding)下才复现，普通shape通过了测试。

Transferable:
- 函数合并的前置检查: 两个"相似"函数存在的理由往往是调用者的安全假设不同。
  合并前必须证明: 所有调用者都满足更强版本的前提条件。
- "同一作者提交+revert"模式表明这是一个"看起来显然正确"的重构。
  Review时对"消除冗余函数"类重构要追问: 为什么最初会有两个版本?


#### Case 13.2.2: Index/IndexPut的AiCore双路径分流

Commits: 57321fab5 (revert Index), 795bff936 (revert IndexPut adaptive),
  3bf1ddde1 (revert IndexPut adaptive logic)
Original: 14996a93f "Add Index/IndexPut aicore adapter"
Files: IndexKernelNpu.cpp, IndexPutKernelNpu.cpp

Scenario: Index/IndexPut算子在通用路径上性能不佳。开发者添加AiCore执行引擎的
快速路径: 当满足维度/dtype/shape条件时走AiCore(高性能)，否则fallback到通用路径
(设置`_exclude_engines=AiCore`属性)。

Constraints:
- AiCore引擎对Index/IndexPut有严格的输入约束(维度数、dtype、mask格式)
- 约束条件是枚举式的(dim>2&&kLong → false, dim>3&&kBool → false, kDouble → false...)
- 这些约束随CANN版本变化(新版本可能放宽或收紧AiCore支持范围)
- 测试覆盖依赖shape组合，维度/dtype的笛卡尔积太大

Decision(原始): 在IndexKernelNpu.cpp中添加`check_index_aicore()`条件判断函数。
Decision(revert): 删除双路径分流，统一走通用路径。

    // 被删除的分流逻辑
    bool check_index_aicore(const at::Tensor& self,
                            const at::TensorList& indices,
                            const at::IntArrayRef masks) {
      if ((self.dim() > 2 && indices[0].scalar_type() == at::kLong) ||
          (self.dim() > 3 && indices[0].scalar_type() == at::kBool))
        return false;
      ...
    }

Alternatives:
- 在CANN算子库侧做自适应引擎选择(让底层runtime决定，而非上层hardcode条件)
- 查表而非枚举: 维护一个(dim, dtype) → engine的映射表，随CANN版本更新
- 只对benchmark验证过的几个关键shape启用AiCore(保守策略)

Consequence: check_index_aicore的枚举条件无法覆盖所有边界case，
部分输入被错误路由到AiCore(不支持)或错误排除出AiCore(损失性能)。
三个独立revert(Index + IndexPut的两个分支)说明两个算子的分流逻辑都有问题。

Transferable:
- 枚举式条件分流的维护成本随输入空间指数增长。当约束来自外部(CANN引擎能力)时，
  应由外部提供能力查询API，而非在上层hardcode判断条件。
- 对应Ch.7(ACL接口绑定)的版本守卫模式: 硬件能力边界应通过API查询而非静态枚举。


### 13.3 初始化时序与生命周期

NPU设备的初始化(init)和清理(finalize)涉及多层资源: ACL context、stream、event、
allocator cache、GE session。这些资源的创建/销毁顺序构成隐式的时序契约。
违反此契约的常见方式: 在错误的时间点做正确的事。


#### Case 13.3.1: shutdown资源清理的删减与恢复

Commits: 6acebe4da (revert)
Original: c180c883914 "[Fix] During shutdown, removing operations for device resources"
Files: InitNpuBindings.cpp, npu_sys_ctrl.cpp
Author: wbigat (同一作者提交+revert, 间隔7天)

Scenario: 开发者发现npu_shutdown中有大量显式资源清理步骤(GraphExecutor Finalize,
TdtChannel Finalize, HostAllocator emptyCache, CachingAllocator emptyCache,
EventManager ClearEvent, stream destroy)。认为这些是冗余的:
既然aclFinalize/aclrtResetDevice会清理设备上下文，显式清理应该是不必要的。

Constraints:
- ACL资源清理的契约: aclrtResetDevice清理设备级资源，aclFinalize清理全局状态
- 但"设备级资源"不包括用户层分配的event和stream对象(这些是用户持有的handle)
- 调用顺序约束: aclrtResetDevice(设备) → aclFinalize(全局)
  原始fix把aclFinalize放在aclrtResetDevice之前，违反了顺序

Decision(原始): 删除显式清理步骤，依赖ACL隐式清理。同时调换finalize/reset顺序。
Decision(revert): 恢复全部显式清理 + 恢复正确的调用顺序。

    // revert恢复的显式清理
    at_npu::native::GraphExecutor::GetInstance().Finalize();
    at_npu::native::TdtChannelForPrint::GetInstance().Finalize();
    THNPUCachingHostAllocator_emptyCache();
    c10_npu::NPUCachingAllocator::emptyCache();

    // revert恢复的正确顺序
    C10_NPU_CHECK(aclrtResetDevice(device_id_));  // 先reset设备
    C10_NPU_CHECK(aclFinalize());                  // 再finalize全局

Consequence: 删除显式清理后:
(1) event handle泄漏(ACL不自动回收用户创建的event)
(2) aclFinalize→aclrtResetDevice的错误顺序触发driver异常
(3) 两个bug叠加: 即使顺序正确也会因event泄漏而段错误

Transferable:
- "SDK应该帮我清理"是危险假设。永远显式释放自己分配的资源。
  即使SDK文档声称finalize会清理一切，也要假设它只清理SDK自己创建的资源。
- 资源清理的调用顺序是API契约的一部分。调换顺序看似等价(都是清理)，
  但底层实现往往依赖严格顺序(reset设备→finalize全局，不可逆)。
- 7天内自我revert说明测试覆盖不足以在提交前捕获shutdown路径问题。
  建议: shutdown path需要专门的集成测试(多次init/finalize cycle)。


#### Case 13.3.2: aclrtDestroyStreamForce -- 引用未就绪的ACL API

Commits: b581b6c8c (revert)
Original: a243de6e2 "!2949 [Fix] fix core dump bug after npu shut down."
Files: acl_rt.h, acl.cpp, npu_sys_ctrl.cpp

Scenario: NPU关闭时stream的自动析构引发core dump。开发者添加显式的
`aclrtDestroyStreamForce`调用来在shutdown前强制销毁stream。
同时在third_party的acl_rt.h中自行添加了该API的声明。

Constraints:
- aclrtDestroyStreamForce是CANN SDK的新API，在目标版本中可能不存在
- 开发者在third_party的mock头文件中添加了函数声明和空实现
- 但实际链接时该符号可能在libascendcl.so中不存在(version-dependent)
- 同一commit还调换了aclFinalize/aclrtResetDevice的顺序(与Case 13.3.1同源)

Decision(原始): 在npu_sys_ctrl.cpp的release callback中添加aclrtDestroyStreamForce调用。
Decision(revert): 删除该调用，删除mock头文件中的声明，恢复原始顺序。

    // 被删除的强制销毁
    auto stream = c10_npu::getCurrentNPUStream();
    C10_NPU_CHECK(aclrtDestroyStreamForce(stream));  // API可能不存在

Alternatives:
- 使用现有的aclrtDestroyStream(非force版本，等待stream上所有任务完成后销毁)
- 在shutdown前先synchronize stream，再正常destroy
- 用dlsym动态查找aclrtDestroyStreamForce，不存在时fallback

Consequence: 在不支持aclrtDestroyStreamForce的CANN版本上链接失败或运行时crash。
mock头文件掩盖了编译期的API缺失(编译通过但链接/运行失败)。

Transferable:
- 永远不要在third_party mock头文件中声明实际SDK不提供的API。
  mock头文件的目的是编译隔离，不是创造不存在的功能。
  这与Ch.9.5(CANN版本兼容)的模式一致: 引入新API前必须确认SDK版本矩阵。
- 对应Ch.7.1(ACL API版本守卫): 使用新ACL API时应有版本守卫
  `if (IsGteCANNVersion(...))` 保护调用路径。


#### Case 13.3.3: _lazy_init注入device_guard热路径

Commits: 561a19dd1 (revert), 62660805f (另一分支的revert)
Original: PR !2695 "[Fix] add _lazy_init in torch_device_guard."
Files: device_guard.py, TensorMethods.cpp, tensor_methods.py, module.py, test_to.py

Scenario: `model.to("npu")`等device transfer操作要求NPU已初始化。
开发者在Python层device_guard入口注入`_lazy_init()`调用，
确保任何device操作前NPU上下文已就绪。同时删除了C++层TensorMethods.cpp中的
`maybe_initialize_npu(device)`调用(认为Python层的guard已覆盖)。

Constraints:
- device_guard是所有device操作的入口，是热路径(每次tensor操作都触发)
- `_lazy_init()`内部检查初始化标志，已初始化时快速返回，但仍有函数调用开销
- C++层的maybe_initialize_npu是精确注入(只在特定tensor方法如new_empty中调用)
- 删除C++层的初始化后，完全依赖Python层guard兜底

Decision(原始): 在device_guard.py import并调用_lazy_init，删除C++层初始化点。
Decision(revert): 删除device_guard中的_lazy_init，恢复C++层精确初始化。

    // revert后恢复的C++精确初始化
    static PyObject * THPVariable_new_empty(...) {
      auto device = parse_npu_device_with_default(r.args[4], self_.device());
      maybe_initialize_npu(device);  // 只在创建tensor时检查
    }

Alternatives:
- 在device_guard中用C级的`__slots__`或cdef快速检查(减少Python调用开销)
- 双层防护: 保留C++层初始化，Python层guard作为二级保障(但不删除C++层)
- 使用`__init_subclass__`或descriptor protocol做一次性初始化(init后自动移除guard)

Consequence: 热路径注入_lazy_init导致:
(1) 每次device操作多一次Python函数调用(对小tensor操作影响显著)
(2) 初始化时序变化: 某些C++层操作在Python guard之前执行，
    失去了C++层maybe_initialize_npu的保护

Transferable:
- 热路径上的"安全检查"必须是零开销或接近零开销。Python函数调用不满足此要求。
  初始化检查应在C++层用inline + 分支预测友好的方式实现。
- "一个统一入口"的诱惑: 在guard层统一注入看似最简，但guard是所有操作的咽喉，
  任何开销都被放大。精确注入(只在真正需要初始化的操作中检查)虽然分散但高效。


#### Case 13.3.4: Inductor AsyncCompile.warm_pool的调用时机

Commits: f17d4cc79 (revert, +5 cherry-picks)
Original: f685c01af "fix thread pool when inductor is imported"
Files: torch_npu/__init__.py, torch_npu/utils/_dynamo.py

Scenario: PyTorch Inductor的AsyncCompile模块在import时会预热线程池(`warm_pool`)。
在NPU环境下这导致线程池初始化竞态: inductor模块在torch_npu注册NPU设备之前就
启动了线程池，线程中尝试访问尚未就绪的NPU context。

Constraints:
- warm_pool触发时机由TORCH_WARM_POOL环境变量控制
- NPU设备注册(`_InductorNpuRegistry.register_inductor_npu`)发生在warm_pool之后
- 原始fix的策略: 先用env var禁用自动warm_pool，再在NPU注册完成后手动调用

Decision(原始): 在__init__.py设置`TORCH_WARM_POOL=0`，在register_inductor_npu中调用
`AsyncCompile.warm_pool()`。
Decision(revert): 删除TORCH_WARM_POOL=0和手动warm_pool调用。

    // 被删除的手动控制
    # torch_npu/__init__.py
    os.environ["TORCH_WARM_POOL"] = "0"  // 禁用自动预热

    # torch_npu/utils/_dynamo.py (register_inductor_npu内)
    AsyncCompile.warm_pool()  // NPU注册后手动预热

Alternatives:
- 不禁用自动warm_pool，而是让warm_pool本身对NPU context缺失做graceful fallback
- 延迟注册: 让NPU设备注册发生在module import阶段(早于warm_pool)
- 使用lazy proxy: warm_pool中创建的线程使用lazy NPU context(首次使用时初始化)

Consequence: 在register_inductor_npu阶段调用warm_pool引入了循环依赖或初始化顺序问题。
TORCH_WARM_POOL=0的全局设置可能影响非NPU的inductor路径(CUDA设备的线程池也被禁用)。

Transferable:
- 第三方模块的初始化时机不可控时，不要在自己的注册流程中调用它。
  应该让第三方模块的初始化在正确的时机自然发生，或通过hook/callback机制对接。
- 全局环境变量是粗粒度的控制手段。`TORCH_WARM_POOL=0`影响所有设备，
  但问题只出在NPU。设备特化的问题需要设备特化的解决方案。


### 13.4 Profiler功能添加的架构选型

Ascend Profiler是torch_npu中独立于上游PyTorch Profiler的功能模块。
功能添加时面临的核心决策: 自研实现 vs 对接上游架构(Kineto)。
选错架构意味着功能做对了但路径不对，需要完整重写。


#### Case 13.4.1: export_stacks自研实现 vs Kineto架构

Commits: 91895483d (revert, +3 cherry-picks跨4分支)
Original: e078c4bd6 "Add function profiler.export_stacks()"
Files: constant.py, view_parser_config.py, stack_view_parser.py(删除), profiler.py

Scenario: 开发者为Ascend Profiler添加`export_stacks()`接口，输出flamegraph格式的
调用栈数据。自研了`StackViewParser`类(29行)，在profiler.py中添加export_stacks方法(22行)，
支持按CPU时间或NPU时间聚合。功能本身正确。

Constraints:
- 上游PyTorch Profiler的export_stacks基于Kineto的_KinetoProfiler接口
- Ascend Profiler有自己的数据结构(torch_op_tree_node)，与Kineto不兼容
- `torch_op_tree_node.device_total_dur`在某些采集场景下不可靠
- 后续上游改造(profiling数据流重构)会与自研路径冲突

Decision(原始): 基于Ascend自有的torch_op_tree_node数据结构自研StackViewParser。
Decision(revert): 完整删除自研实现。13天后由另一作者通过Kineto接口重新实现(MR !7015-!7017)。

    // 被删除的自研常量定义
    EXPORT_STACK = "export_stack"
    METRIC_CPU_TIME = "self_cpu_time_total"
    METRIC_NPU_TIME = "self_npu_time_total"

    // 被删除的StackViewParser类 (29行)
    // 被删除的profiler.py中export_stacks方法 (22行)

Alternatives:
- 从一开始就走Kineto接口(需要先解决Kineto在NPU上的适配问题)
- 保留自研实现但标记为experimental，与Kineto路径并行
- 只导出raw数据，让外部工具(如py-spy)做flamegraph聚合

Consequence: 自研实现在13天后全面撤回，2个月后Kineto路径实现上线。
撤回原因不是功能bug，而是架构路径选择: 自研路径无法跟随上游profiling数据流的演进。

Transferable:
- 功能正确不等于架构正确。自研vs对接上游是torch_npu的永恒决策点:
  自研路径开发快但维护成本高(每次upstream改造都需要同步)，
  对接上游路径开发慢但维护成本低(随upstream自动演进)。
- 判断标准: 如果上游有对应的扩展点(如_KinetoProfiler)，优先对接;
  如果需要深度定制且上游无扩展点，才考虑自研。
- 这是torch_npu的核心架构trade-off: 作为downstream adapter，
  与upstream的耦合度决定了长期维护成本。


### 13.5 横切总结: Revert-Driven模式

9个case揭示了5个共性模式:


Pattern 1: 粗粒度手段解决细粒度问题

Case 13.2.1(format_contiguous合并), 13.3.3(_lazy_init热路径), 13.3.4(TORCH_WARM_POOL全局禁用)
都是用"全局/统一"方案解决"局部/特化"问题。合并函数→影响安全路径的调用者，
全局guard→影响热路径性能，全局env var→影响所有设备。

Detection: Review时对"统一/合并/简化"类PR追问:
"不需要这个优化/检查的调用者有多少? 它们被影响后会怎样?"


Pattern 2: 架构重构的原子性困境

Case 13.1.2(event reconfig, 15文件), 13.1.1(per-device allocator, 6文件)
大范围联动变更无法部分回滚。任一子目标回归导致全量撤销。

Mitigation: 拆分为独立可回滚的PR序列。每个PR解决一个子目标，独立验证。


Pattern 3: 隐式的时序/顺序契约

Case 13.3.1(finalize/reset顺序), 13.3.2(API版本依赖), 13.3.4(warm_pool调用时机)
资源清理顺序、API可用性、初始化时序都是隐式契约，代码中无显式声明。

Mitigation: 在代码注释中显式标注顺序依赖。
e.g., `// MUST: aclrtResetDevice before aclFinalize (ACL contract)`


Pattern 4: mock/stub掩盖真实依赖缺失

Case 13.3.2(mock acl_rt.h中的aclrtDestroyStreamForce)
在third_party mock中添加不存在的API声明，编译通过但运行失败。

Mitigation: mock头文件只包含当前SDK版本实际提供的API。
新API通过版本守卫(IsGteCANNVersion)保护，而非mock。


Pattern 5: 同一作者短期自我revert

Case 13.2.1(chuboning), 13.3.1(wbigat), 13.4.1(qkd→MooYeh重写)
说明: (1) code review未能捕获的问题在CI/集成测试中暴露;
(2) 开发者对自己代码的回滚比review更快，但失去了团队知识沉淀。

Mitigation: revert PR应包含root cause分析(为什么原方案不work)，
而非仅仅"revert commit xxx"。为后续开发者保留教训。


---

## Ch.14 Profiler数据管道与兼容性

torch_npu的profiler子系统维护了两条并行的数据输出路径(csv和db)、
一套多维度配置空间(level × aicore_metrics × activities × export_type × async_mode × rank_set)、
以及对CANN profiling API的版本兼容适配层。
本章覆盖的8个case揭示了一个共性: profiler的核心采集逻辑相对稳定，
问题集中在数据后处理的边界条件、配置状态管理、和CANN API兼容性三个层面。


### 14.1 数据后处理与配置状态


#### Case 14.1.1: step_trace_time减法溢出 -- 时间区间差值的语义越界

Commit: df8008c2f
Files: _trace_step_time_parser.py, _trace_step_time_db_parser.py

Scenario: profiling后处理解析step trace数据，计算通信时间与计算时间的重叠关系。
输出中`comunNotOverlpRec`(未重叠通信时间-排除接收时间)字段出现负数。

Constraints: `comunNotOverlpRec = comunNotOverlp - bubble`。当bubble时间(气泡/空闲时间)
大于非重叠通信时间时，减法结果为负。被减数和减数来自不同采集源(host vs device)，
时间戳精度和采集窗口不完全对齐。

Decision: 用`max(..., 0)`将结果钳位到非负。两条输出路径(csv parser和db parser)
做了相同修复。

Alternatives:
- 在采集层对齐host/device时间戳精度(改动面大，且不同硬件的时间戳精度差异是固有的)
- 将负值保留并标注为"数据精度不足以计算该指标"(语义更精确但增加下游处理负担)

Consequence: 修复了数据异常。但更深层的问题是: csv和db两条路径需要做相同修复，
说明数据后处理逻辑存在重复，bugfix时容易只修一条路径遗漏另一条。

Transferable: 涉及时间区间差值运算时，必须考虑减法结果的语义边界。
尤其当被减数和减数来自不同采集源时，负值是常态而非异常。
防御方式: clamp或标记为"不具备参考性"。


#### Case 14.1.2: proxy判断 vs 直接判断 -- L0级别shape列丢失

Commit: 2f9aa0594
Files: _profiler_config.py

Scenario: 用户使用profiler的L1级别，配置aicore_metrics为AicMetricsNone。
期望kernel_details.csv包含完整shape信息列，但shape列被错误过滤。

Constraints: profiler有两个独立的配置维度: profiler_level(L0/L1/L2)和
aicore_metrics(None/PipeUtil/Memory等)。`is_all_kernel_headers()`用于判断是否显示完整表头。

Decision: 原实现用`_ai_core_metrics != AicMetricsNone`作为"非L0"的proxy判断。
但L1 + AicMetricsNone是合法组合，此时proxy判断为False，导致L1被误判为L0。
Fix: 改为`_profiler_level != LEVEL0`直接判断。

Alternatives:
- 将profiler_level和aicore_metrics的组合空间显式建模为enum(类型安全但增加配置复杂度)
- 在配置层禁止L1+AicMetricsNone的组合(限制用户自由度)

Consequence: 修复了L1级别下shape信息的丢失。
同时暴露了配置层(`_profiler_config.py`)的脆弱性: 多个独立维度的配置之间存在隐式关联假设。

Transferable: 不要用间接属性(proxy)判断业务状态。
A和B是独立配置维度时，它们之间没有双射关系，用一个推断另一个必然产生盲区。
当配置维度超过3个时，组合空间的爆炸使得proxy判断更加危险。


#### Case 14.1.3: 默认值与文档不一致 -- export_type的无谓WARNING

Commit: 44e0d99f6
Files: experimental_config.py

Scenario: 用户不设置`export_type`参数时，profiler打印WARNING。
但文档明确写了默认值是"text"，用户看到WARNING会困惑。

Constraints: `_ExperimentalConfig.__init__`中`export_type`的默认值是None，
后续校验发现None时先打WARNING再fallback为text。

Decision: 将参数默认值从None改为ExportType.Text，消除"先留None后校验再WARNING再fallback"的迂回路径。

Alternatives:
- 保留None默认值但删除WARNING(区分"用户未设置"和"用户显式设了text"的能力被保留)
- 在文档中说明"不设置会WARNING但行为等同于text"(文档兜底，不改代码)

Consequence: 一行修改消除了用户困惑。

Transferable: 函数签名的默认值应当与文档描述一致。
"先留None后校验"的模式只适用于需要区分"用户未设置"和"用户显式设置了默认值"的场景。
如果不需要区分，就直接用有意义的默认值。


#### Case 14.1.4: 双指针匹配退化为O(n^2) -- profiler db解析性能

Commit: 75c77b902
Files: _fwk_api_db_parser.py

Scenario: profiler的db解析模式下，解析耗时异常长。
FwkApiDbParser中node_launch与dequeue、torch_op与enqueue的匹配算法使用双指针遍历。

Constraints: 双指针算法要求: 当dequeue的start_ns已超过node_launch的end_ns(时间窗口错过)时，
应记录当前指针位置并break，让外层推进到下一个node_launch。

Decision: 原代码在"不匹配"分支中没有更新指针位置也没有break，
导致双指针退化为O(n^2)暴力遍历。在三处匹配循环中补充了`last_xxx_index = idx`和`break`。

Alternatives:
- 改为基于时间戳的二分查找(降低到O(n log n)，但实现复杂度高)
- 用pandas的merge_asof做时间序列join(外部依赖重但代码简洁)

Consequence: 恢复了双指针的O(n)时间复杂度。大规模profiling数据的解析时间从分钟级降到秒级。

Transferable: 双指针/归并匹配算法中，"不匹配"分支的指针更新逻辑和"匹配"分支同样重要。
漏掉不匹配分支的break，双指针直接退化为暴力搜索。
review时必须检查所有分支是否都正确推进了指针。


### 14.2 CANN/PyTorch接口适配


#### Case 14.2.1: aclrtStreamGetId不存在时的fallback -- 跨CANN版本兼容

Commit: 85d2a49b9
Files: AclInterface.cpp, AclInterface.h, npu_profiler.cpp

Scenario: profiler采集内存数据时调用aclrtStreamGetId获取流ID。
在CANN旧版本中该接口不存在，导致训练直接中断。

Constraints: CANN SDK跨版本接口可用性不一致。aclrtStreamGetId是较新版本引入的。
torch_npu需要在编译时不静态链接该符号(否则旧版本上.so无法加载)，
同时在运行时检测并提供fallback。

Decision: 标准三件套模式:
(1) 新增IsExistRtGetStreamId()，通过GET_FUNC(dlsym)动态查找符号
(2) reportMemoryDataToNpuProfiler中先检查接口存在性
(3) 不存在时用reinterpret_cast<int64_t>(data.stream)将stream指针转为整数作为fallback标识

Alternatives:
- 编译期通过CANN版本宏条件编译(不支持同一二进制兼容多CANN版本)
- 在profiler层面直接跳过stream ID采集(丢失信息)

Consequence: 修复了旧CANN版本上的crash。fallback方案用stream指针值作为ID，
在同一进程生命周期内唯一，但跨进程不可比较。对profiling分析够用。

Transferable: torch_npu对CANN接口的标准兼容模式:
GET_FUNC(dlsym)动态查找 → IsExist*检查 → wrapper函数三件套。
这个模式在codebase中已有20+实例(IsExistMemcpyBatch等)，是最成熟的兼容性模式。


#### Case 14.2.2: dynamo EventVariable缺少python_type -- 上游接口适配

Commit: 652bc2ba4
Files: _dynamo.py

Scenario: torch.compile的fullgraph=False模式下使用profiler。
profiler用Event做计时，当torch._dynamo在graph break时遇到EventVariable，
需要获取其python_type，但该方法不存在。

Constraints: PyTorch上游的EventVariable类缺少python_type方法。
torch_npu作为下游适配层，在编译+profiling联合场景下触发该缺失。
上游的EventVariable没有预期到会在断图场景中被inspect。

Decision: 在torch_npu的dynamo monkey-patch机制中为EventVariable补丁注入python_type方法，
返回type(self.value)。用__dict__检查避免重复patch。

Alternatives:
- 向上游提交patch添加python_type方法(正确但等待上游合入周期长)
- 在torch_npu的graph break处跳过Event对象的inspect(可能遗漏其他需要python_type的场景)

Consequence: 修复了编译+profiling联合场景下的AttributeError。
这类monkey-patch需要在每次升级PyTorch版本时检查上游是否已修复。

Transferable: torch_npu维护了一套对torch._dynamo内部类的monkey-patch，
当上游新增Variable子类但缺少必要协议方法时，下游需要补齐。
Ch.4中已覆盖了monkey-patch的一般模式，此case是profiler特化场景。


#### Case 14.2.3: load_info幂等性与路径校验 -- CANN路径不存在时的优雅降级

Commit: fd6927797
Files: _profiler_config.py, _cann_file_parser.py

Scenario: 用户采集profiling数据后，CANN的PROF_XXX目录被删除或不存在
(只采集了CPU数据，或数据被部分清理)。解析时多次调用load_info，
导致重复打印路径校验报错信息。

Constraints: 两个问题叠加:
(1) load_info没有幂等保护，每次调用都重新校验路径并打印error
(2) 当CANN路径不存在时，_check_cann_path_valid直接raise RuntimeError，
中断整个解析流程，而不是优雅降级只跳过NPU解析

Decision: 三管齐下:
(1) 增加_is_load标志位使load_info幂等
(2) 新增load_activities方法在load_info阶段就检查fwk_path和cann_path是否存在，
不存在则从activities中移除对应项
(3) 删除不再需要的load_is_cluster方法

Alternatives:
- 在每个下游parser中各自检查路径(重复且容易遗漏)
- 将所有路径校验延迟到parse时(错误报告时机太晚)

Consequence: 修复了重复报错和crash。更重要的是建立了正确的层次:
"配置加载"阶段做完所有前置校验并裁剪后续处理scope，
而不是让每个下游parser各自发现问题再报错。

Transferable: 多阶段解析流程中，load阶段应做完所有前置校验并裁剪scope。
load函数加幂等保护(flag guard)是基本功。
这与Ch.6初始化生命周期中的lazy_init模式类似: 一次初始化，多次安全调用。


#### Case 14.2.4: 动态Profiling多rank异步解析 -- 配置维度交叉组合

Commit: 0e93c91c3
Files: _dynamic_profiler_config_context.py, _profiling_parser.py

Scenario: 动态profiling场景下，用户配置只采集部分rank的数据。
某些rank上的数据采集了但无法解析。

Constraints: 三个维度的交叉: 多rank采集、部分rank过滤、同步/异步解析模式。
原代码在rank_id命中rank_set时，无条件将_analyse设为False(关闭解析)。
但这只适用于同步解析模式。异步模式下，解析在独立进程中完成，不应被关闭。

Decision: 在设置`_analyse = False`前增加`if not self._async_mode`判断。
仅同步模式下关闭解析，异步模式保持开启。同时在日志中增加sync/async模式标识。

Alternatives:
- 将_analyse改为三值enum(ENABLED / SYNC_DISABLED / ASYNC_ENABLED)(类型安全但过度设计)
- 在异步解析进程中独立管理rank过滤(正确但需要跨进程传递rank_set配置)

Consequence: 修复了异步模式下部分rank数据无法解析的问题。

Transferable: feature flag(如_analyse)的设置必须考虑所有上下文维度。
多rank + 动态profiling + 异步解析构成了3维交叉，
在一个维度的逻辑中无条件修改另一个维度的配置，必然在某个交叉点上出错。
review时对"无条件赋值"追问: "所有上下文下这个赋值都正确吗?"


### 14.3 横切总结: Profiler子系统的缺陷模式

8个case揭示4个结构性问题:

Pattern A: 双路径维护负担
csv输出和db输出两条路径(Case 14.1.1)。bugfix时必须两条都修。
_profiler_config.py被Case 14.1.2/14.2.3/14.1.3三个commit反复修改，是profiler中最脆弱的模块。

Pattern B: 配置维度爆炸
level × aicore_metrics × activities × export_type × async_mode × rank_set构成高维配置空间。
proxy判断(Case 14.1.2)和无条件赋值(Case 14.2.4)是配置维度交叉的两种典型故障模式。

Pattern C: CANN版本兼容的标准模式
GET_FUNC → IsExist* → wrapper三件套(Case 14.2.1)。
已有20+实例。新增CANN接口调用时，不遵循此模式就是bug。

Pattern D: 核心采集稳定，边界处理脆弱
8个bugfix中0个涉及数据正确性(如采集到错误的性能数据)，
全部是鲁棒性和可用性问题(crash、误告警、性能退化、数据缺失)。
说明profiler的核心采集逻辑经过充分验证，问题主要出在边界处理和配置管理层。


---

## Ch.15 ACL运行时接口兼容性与版本守卫

torch_npu作为CANN SDK的消费者，不能假设特定ACL接口在所有目标环境中都可用。
AclInterface.cpp(586次bugfix touch, Top-9文件)是ACL接口的集中绑定层，
通过dlopen/dlsym动态加载模式实现运行时兼容。
本章的7个case揭示了API可用性的"三层依赖模型":
CANN runtime版本(用户态库) → 驱动版本(内核态) → SoC型号(硬件能力)。


### 15.1 符号解析与动态加载


#### Case 15.1.1: 静态链接新API导致.so无法加载 -- aclrtGetDeviceInfo

Commit: b6e893ee1
Files: AclInterface.cpp, Module.cpp

Scenario: 开发者在initDeviceProperty中调用aclrtGetDeviceInfo来获取设备总内存大小
(替代精度较低的aclrtGetMemInfo)。该接口在CANN 9.0.0.beta2中引入。

Constraints: aclrtGetDeviceInfo是编译期直接链接调用的(静态符号引用)。
当运行时CANN版本低于9.0.0.beta2，libascendcl.so中不存在该符号，
导致libtorch_npu.so加载时直接报`undefined symbol`。
这是在函数调用之前发生的，任何运行时版本检查(IsGteCANNVersion)都来不及执行。

Decision: 将aclrtGetDeviceInfo从静态链接改为dlopen/dlsym动态加载模式:
(1) LOAD_FUNCTION(aclrtGetDeviceInfo)注册符号名
(2) 新增AclrtGetDeviceInfo包装函数，通过GET_FUNC在运行时dlsym查找
(3) 新增IsExistAclrtGetDeviceInfo存在性检查函数
(4) 调用点改为双重守卫: if (is_cann_900beta2 && IsExistAclrtGetDeviceInfo())

Alternatives:
- 编译期通过CANN版本宏条件编译(不支持同一二进制跨CANN版本)
- 将aclrtGetDeviceInfo放入第三方mock头文件(编译通过但运行时行为未定义，见Ch.13 Pattern 4)

Consequence: 修复了旧CANN版本上的加载失败。这个case是torch_npu中
"如何正确引入新ACL API"的标准参考。

Transferable: 插件式架构中调用宿主SDK的新增API，必须走dlopen/dlsym动态加载。
否则SDK版本回退时，插件.so本身就无法加载(链接器在load time解析所有符号)。
标准模式: LOAD_FUNCTION注册 + IsExist守卫 + wrapper函数三件套。
这与Case 14.2.1中profiler对aclrtStreamGetId的处理是完全相同的模式。

Error-fix链: 此case的failure mode(undefined symbol at load time)与Ch.13 Case 13.3.2
(mock acl_rt.h中不存在的API)是同一类问题的两个变体:
一个是调用了不存在的符号，另一个是mock了不存在的符号让编译通过但运行失败。


#### Case 15.1.2: 异步拷贝的隐式前提 -- aclrtMemcpyAsyncWithCondition

Commit: 5d39f5d9b
Files: CachingHostAllocator.cpp, AclInterface.cpp

Scenario: aclrtMemcpyAsyncWithCondition是带条件的异步内存拷贝接口。
原代码在process_unregistered_mem_location_type中有一段优化:
如果该接口存在且方向是D2H，就跳过同步直接返回。

Constraints: "异步拷贝"的前提是目标buffer在ACL runtime的管控范围内(pinned memory)。
对普通malloc分配的host memory(unregistered)，runtime无法在正确的时间点完成同步。
跳过同步意味着host侧读到的是拷贝未完成的脏数据。

Decision: 删除D2H跳过同步的优化(5行代码)，
让所有unregistered memory的拷贝都走同步路径。
同时在AclrtPointerGetAttributesExist()中补充CANN runtime版本检查(>=8.5.0)。

Alternatives:
- 将unregistered memory临时pin再做异步拷贝(额外的pin/unpin开销)
- 用event等待替代stream同步(细粒度但实现复杂)

Consequence: 修复了D2H数据竞争。性能回退是可接受的: unregistered memory
是非热路径(热路径应该用pinned memory allocator)。

Transferable: API名字中有"Async"不等于上下文已被完全异步化。
当host端buffer不在runtime管控范围内时，runtime无法插入正确的同步点。
性能优化必须严格验证前置条件，否则引入数据竞争。
判断标准: "谁拥有这块内存的生命周期管理权?"
如果答案是"runtime"，异步安全; 如果答案是"用户代码"，必须同步。


### 15.2 版本兼容性三层模型


#### Case 15.2.1: 驱动版本不满足 -- aclrtPointerGetAttributes

Commit: f8a0f174c
Files: AclInterface.cpp

Scenario: aclrtPointerGetAttributes用于查询指针属于host还是device内存。
已有CANN runtime版本检查(>=8.5.0)和dlsym可达性检查，但在旧驱动上仍然失败。

Constraints: ACL接口的可用性存在三层依赖:
(1) CANN runtime版本 -- 用户态库是否包含实现
(2) 驱动版本 -- 内核态是否支持
(3) SoC型号 -- 硬件是否具备能力
dlsym找到符号只能证明用户态库有声明，不代表驱动层有实现。

Decision: 在exist检查的lambda中，在runtime版本检查之前，
先增加驱动版本守卫: IsGteDriverVersion("25.5.0")。

Alternatives:
- try-catch包裹调用，失败时fallback(运行时开销且不够确定性)
- 在驱动层面查询feature capability(如果驱动提供此接口的话)

Consequence: 修复了旧驱动上的运行时错误。
但暴露了exist检查的迭代修复模式: 先加runtime check → 后补driver check → 再补SoC check。

Transferable: API可用性守卫必须覆盖所有依赖层。
dlsym成功 ≠ 调用安全。中间还有驱动层和硬件层。
最佳实践: 引入新API wrapper时一次性确认所有层的最低版本要求，
而不是等线上报错后逐一补丁。


#### Case 15.2.2: 同一接口的递进修复 -- aclrtMallocHostWithCfg的驱动+SoC守卫

Commits: 503381ca1 (driver check), cd5f447d5 (SoC check)
Files: AclInterface.cpp, NPUAllocatorConfig.cpp

Scenario: aclrtMallocHostWithCfg用于分配具有特定配置的host pinned memory(支持VA一致性)。
在Case 15.2.1的同一时期，该接口也经历了完全相同的问题: 缺少驱动版本守卫。

Constraints: 第一轮修复(503381ca1)补充了IsGteDriverVersion("25.5.2")。
但随后发现Ascend910_95 SoC虽然编号更高，底层固件不支持该特性。
SoC版本号的大小顺序不等于功能集的超集关系。

Decision: 两轮修复:
(1) 第一轮: 增加IsGteDriverVersion("25.5.2")，同时更新NPUAllocatorConfig中的
用户提示信息，明确告知需要同时升级CANN版本和驱动版本
(2) 第二轮: 将SoC版本检查从单边下界(>= Ascend910B1)改为区间约束:
>= Ascend910B1 && < Ascend910_95

Alternatives:
- 用CANN SDK提供的feature capability查询(如果有的话)替代硬编码版本号
- 维护一个SoC→feature的映射表(准确但需要频繁更新)

Consequence: 两轮修复完成后，aclrtMallocHostWithCfg的exist检查同时守卫了
三个层次: CANN runtime版本 + 驱动版本 + SoC型号范围。
这是codebase中最完整的版本兼容性守卫实例。

Transferable: 硬件SoC版本号不是线性递增的功能超集。
新一代芯片可能在某些特性上不兼容老接口(架构差异、固件差异)。
API可用性检查不能用简单的`>=`，必须精确指定支持的SoC范围。
这与CUDA中某些feature在特定GPU架构上disabled的情况类似
(如Tensor Core在Volta/Ampere/Hopper上的能力差异)。

Error-fix链: 503381ca1 → cd5f447d5是同一接口的两轮递进修复，
证实了"exist检查"需要一次性覆盖所有层，否则会产生连环修复。


### 15.3 设备管理与执行路径


#### Case 15.3.1: current_device假设在多卡下不成立 -- getDeviceFromPtr

Commit: 1341fbe93
Files: NPUHooksInterface.cpp, AclInterface.cpp, acl_rt.h

Scenario: PyTorch的getDeviceFromPtr接口需要根据data指针返回它所在的设备。
这是tensor在多卡间迁移、序列化等场景的基础设施。

Constraints: 原实现直接返回c10_npu::current_device()，即当前线程设定的设备。
这在多卡场景下是错误的: tensor的数据可能分配在device 3上，
但当前线程的current device是device 0。
返回错误的设备ID会导致后续操作在错误的设备上执行。

Decision: 引入aclrtPointerGetAttributes接口:
(1) 在acl_rt.h中添加aclrtPtrAttributes结构体(含location.id和location.type)
(2) 在AclInterface中添加dlopen wrapper
(3) getDeviceFromPtr改为从attributes.location.id获取真实设备ID

Alternatives:
- 维护一个指针→设备的全局映射表(allocator层面记录，但增加分配开销)
- 在tensor metadata中记录分配时的device_id(需要修改上游的StorageImpl)

Consequence: 修复了多卡场景下设备ID错误的问题。
这个bug在单卡环境下永远不会暴露，只在多卡场景触发。

Transferable: current_device()是线程局部状态，指针的物理归属是全局属性，
两者没有必然关系。在多设备环境中，任何涉及"这个指针属于哪个设备"的判断，
都必须查询runtime而不是依赖线程上下文。
推广: 线程局部状态(TLS)的值只能用于"当前线程打算做什么"的语义，
不能用于"这个数据是什么"的语义。


#### Case 15.3.2: 多路径执行入口的初始化泄漏 -- aclop与deterministic

Commit: a78f7121c
Files: OpCommand.cpp, OpParamMaker.cpp

Scenario: torch_npu同时支持aclop(旧路径，通过ACL算子编译器)和opapi/aclnn(新路径)。
deterministic模式需要通过AclSetCompileopt(ACL_OP_DETERMINISTIC, ...)设置。
用户在纯aclnn流程中开启deterministic模式时，程序freeze。

Constraints: 两个bug叠加:
(1) LazyInitAclops()和AclSetCompileopt(ACL_OP_JIT_COMPILE, ...)被无条件调用，
即使当前走的是opapi/aclnn路径。aclop初始化涉及op模型加载等重量级操作。
(2) SetDeterministicOption中，当!isOpapi时会调用AclSetCompileopt(ACL_OP_DETERMINISTIC)，
但如果从未使用过aclop(g_used_aclop == false)，aclop编译器未初始化，
调用其compile option设置接口会导致hang。

Decision: 两处修改:
(1) LazyInitAclops()和JIT compile option设置移入aclop分支(CheckCustomHandlerNull()为true时)
(2) SetDeterministicOption中增加g_used_aclop条件守卫

Alternatives:
- 让LazyInitAclops成为真正的no-op(如果已有aclnn handler则跳过)(需要理清init的依赖关系)
- 拆分OpCommand::Run为两个独立入口(aclop_run/aclnn_run)(大改)

Consequence: 修复了纯aclnn场景下的freeze。g_used_aclop标志位是轻量有效的方案。

Transferable: 当系统存在多条执行路径且共享统一入口时，
路径A的初始化/配置操作不能放在公共代码段中无条件执行。
"lazy init"类操作隐含假设"一定会走到需要它的路径"，
但如果另一条路径也触发公共入口，lazy init就会在错误的上下文中执行。
通过引入"是否曾经使用过该路径"的标志位守卫，是最小化改动的正确方案。


### 15.4 横切总结: ACL接口兼容性模式

7个case构成一个完整的"API兼容性守卫知识体系":

三层依赖模型:
- Layer 1: CANN runtime版本 → dlsym可达性(Case 15.1.1, 14.2.1)
- Layer 2: 驱动版本 → 调用安全性(Case 15.2.1, 15.2.2第一轮)
- Layer 3: SoC型号 → 功能可用性(Case 15.2.2第二轮)

标准守卫模式(从Case 15.1.1提炼):
```
LOAD_FUNCTION(aclrtXxx);         // 注册符号名
bool IsExistAclrtXxx() {          // 存在性检查
  return GET_FUNC(aclrtXxx) != nullptr
    && IsGteCANNVersion(...)      // Layer 1
    && IsGteDriverVersion(...)    // Layer 2
    && IsSupportedSocRange(...);  // Layer 3
}
aclError AclrtXxx(...) {          // 安全wrapper
  GET_FUNC(aclrtXxx)(...);
}
```

隐式前提条件(Case 15.1.2, 15.3.1):
API名称中的语义(Async, GetDeviceFromPtr)隐含了使用前提
(buffer在runtime管控下, 线程与设备的绑定关系)，违反前提不会编译报错，
只会在运行时产生数据竞争或逻辑错误。

多路径入口污染(Case 15.3.2):
公共入口中的路径特化代码会泄漏到其他路径。
守卫方式: 路径使用标志位(g_used_aclop)或条件分支。


---

## Ch.16 Inductor编译器后端边界适配

torch_npu的Inductor后端将PyTorch的图IR编译为NPU可执行的Triton-like kernel。
与CUDA后端的关键差异:
(1) Dense tile layout不固定(CUDA假设[X, R]，NPU可能是[R, X...]或[X..., R])
(2) 显式mask管理(NPU有自定义filter_masks逻辑，CUDA不需要)
(3) golden_var_list是NPU独有的维度推导概念，有多种推导路径

本章8个case(含1个revert)覆盖了codegen模板适配、upstream扩展机制、和NPU特化逻辑的常见陷阱。


### 16.1 维度推导与数据源一致性


#### Case 16.1.1: Contiguous Reduction判断不一致 -- 及其revert

Commits: 6446a3adb (fix), 1ddb00b4b (revert)
Files: triton.py, kernel_analysis.py

Scenario: dual reduction contiguous场景(如多维sum: shape (256,32,640)经permute/clone后做sum)，
Triton kernel生成了错误的reduction代码，计算结果不正确。

Constraints: NPU codegen中存在两个is_contiguous_reduction()方法，
使用了不同的数据源确定维度顺序:
- SplitTiling版本: parse_golden_from_load_store_index()(按stride排序)
- TritonKernel版本: self.golden_var_list(按最长index分析结果)
当reduction维度的stride相同时，两者给出的维度顺序不同。

Decision(fix): 引入get_reduction_layout_var_list()，在multi-reduction场景下
优先使用stride排序数据源，同时将kernel_analysis.py中的reduction_dim()和
dense_size_list()也统一到stride排序。

Decision(revert): fix合入v2.7.1后引发其他场景回归。原fix在kernel_analysis.py中
过于激进地替换了golden_var_list的所有使用，但dense_size_list()中的代码
依赖golden_var_list已被提前初始化(通过select_golden_varlist())。
移除延迟初始化保护后，部分边角case产生初始化顺序问题。

Alternatives:
- 从根本上统一golden_var_list的推导路径(大重构，影响所有codegen)
- 只修SplitTiling版本的is_contiguous_reduction使用相同数据源(局部修复)

Consequence: revert → 回到原始bug状态。后续需要更保守的修复方案，
只统一contiguous reduction判断的数据源，不动kernel_analysis的核心逻辑。

Transferable: 当codegen中有多处使用同一概念(维度顺序)做判断时，
必须确保数据源一致(Single Source of Truth)。
修复不一致时，影响面评估至关重要: 仅验证触发bug的case不够，
需要对上下游所有消费同一数据源的路径做回归。
revert-first-then-re-fix是正确的风险控制策略。


#### Case 16.1.2: golden_var_list为None -- constant tensor不产生索引分析

Commit: db1de538e
Files: triton.py, lowering.py, utils.py

Scenario: 两个独立问题:
(1) patch_expr_fits_within_32bit全局monkey-patch导致性能下降
(2) cat操作在输入为constant tensor时crash

Constraints:
(1) patch_expr_fits_within_32bit将所有表达式强制标记为超出32位范围，
禁用了upstream的32位索引优化路径。原意是绕过NPU不支持某些32位优化的问题，
但副作用是全局性能下降。
(2) cat codegen的cat_store()直接访问self.golden_var_list[::-1][-1]，
但当所有输入都是constant(如torch.full)时，没有load/store索引可分析，
golden_var_list为None，触发TypeError。

Decision:
(1) 移除patch_expr_fits_within_32bit整个patch，恢复upstream行为
(2) cat_store()入口增加golden_var_list is None的guard，fallback到简单tl.store
(3) lowering层增加NPU device type检查，确保只有NPU设备才走自定义cat kernel

Alternatives:
- 对patch_expr_fits_within_32bit做细粒度条件(只对NPU不支持的表达式返回False)
- 对constant tensor在lowering阶段就分流到eager(避免进入codegen)

Consequence: 移除粗粒度patch恢复性能，constant tensor不再crash。

Transferable: monkey-patch upstream函数时应最小化影响范围。
返回False的副作用远超预期，因为它影响所有索引表达式的codegen路径。
codegen中任何依赖golden_var_list的路径都必须处理None case:
constant tensor不产生索引分析结果，这是合法输入。


### 16.2 Upstream IR扩展与兼容


#### Case 16.2.1: Scan IR打破inside_reduction假设

Commit: fcd98896b
Files: scheduling.py, triton.py

Scenario: aten.cumprod/aten.cumsum等前缀累计算子在NPU+Inductor下编译失败，
报RuntimeError: failed to get one reduction node。

Constraints: scan kernel的inside_reduction标志为True(因为它有r*轴)，
但它不包含ir.Reduction节点(scan的IR是ir.Scan)。
NPU codegen在三个地方假设inside_reduction=True等价于存在ir.Reduction节点:
(1) scheduling.py的decide_codegen_dims_in_kernel()无条件构建ReductionAnalysis
(2) triton.py的dense_size_list()/dense_size_str()在inside_reduction时无条件构建
(3) 没有scan()的codegen override

Decision: 四处修改:
(1) scan禁用cooperative reduction(对齐upstream)
(2) decide_codegen_dims_in_kernel()增加find_reduction_node()非None检查
(3) dense_size_list()/dense_size_str()增加同样的guard
(4) 重写NPUIndexTritonKernel.scan()，动态从golden_var_list推断scan axis

Alternatives:
- 将scan降级为fallback eager执行(功能正确但放弃编译优化)
- 修改inside_reduction的语义为inside_reduction_or_scan(影响面太大)

Consequence: cumprod/cumsum等scan类算子可以正常编译执行。
新增的scan()方法需要跟踪upstream scan IR的演进。

Transferable: upstream新增IR类型(如Scan)可能悄悄打破downstream的隐式假设。
inside_reduction=True曾经等价于"有Reduction节点"，引入Scan后不再成立。
这是downstream fork维护的核心挑战: upstream的语义变化不一定体现在接口签名上，
需要阅读upstream的changelog和IR定义来发现。


#### Case 16.2.2: invoke_subgraph被错误fallback

Commit: 3182ab047
Files: lowering_op_list.py

Scenario: test_invoke_subgraph.py在NPU上报错。

Constraints: torch_npu的Inductor适配对higher-order ops做了全局fallback处理(让它们走eager执行)。
但invoke_subgraph不应被fallback: 它是一种子图调用机制，
fallback会丢弃子图内的编译优化。

Decision: 将torch.ops.higher_order.invoke_subgraph加入GENERATE_LIST(允许正常lowering的算子白名单)。

Alternatives:
- 改为黑名单策略(只列出需要fallback的higher-order ops)(需要枚举所有不支持的op)
- 为invoke_subgraph实现NPU特化的lowering(过度工程化，upstream实现已足够)

Consequence: invoke_subgraph正常lowering。

Transferable: "默认fallback + 白名单放行"策略下，
新增的higher-order op需要逐个评估是否适合fallback。
invoke_subgraph的lowering不涉及device-specific逻辑，
fallback反而破坏语义。白名单需要随upstream版本持续维护。


### 16.3 Mask管理与Codegen模板


#### Case 16.3.1: filter_masks丢弃semantic mask -- load中的other值

Commit: 6dc00c5df
Files: triton.py

Scenario: NPU Inductor支持aten.constant_pad_nd等需要非零fill value的op时，
生成的tl.load缺少必要的tmp mask，且other值被硬编码为0.0。

Constraints: NPU codegen有自定义的filter_masks()逻辑:
原实现是"只保留在masked_axis_name中的mask，丢弃其余"。
但这会错误丢弃semantic mask(如tmp mask，用于guard padded/indirect access)。
这些mask不是axis tail mask，不应被过滤。
同时load()中masked load时other被硬编码为0.0，但constant_pad_nd的padding value可能非零。

Decision: 重构filter_masks():
对每个mask判断它是否匹配某个axis name。
不匹配任何axis name → semantic mask → 保留。
匹配但该axis不在masked_axis_name中(redundant) → 丢弃。
load()中当self._load_other is not None时使用其值，否则fallback到0.0。

Alternatives:
- 另一个PR(#29941)尝试的启发式: "如果有tmp mask就丢弃所有axis mask"
  被评估为"fundamentally unsafe"而拒绝
- 完全不过滤mask(安全但可能引入冗余mask导致性能下降)

Consequence: 修复了constant_pad_nd等op的codegen。
filter_masks的设计决策是: 对"不认识的mask"采取保守策略(保留而非丢弃)。

Transferable: 自定义过滤器中，对"不认识的输入"应默认保留而非默认丢弃。
这是安全性原则: 丢弃一个必要的mask会导致计算错误(静默数据损坏)，
保留一个冗余的mask最多影响性能(多一个条件判断)。
两害取其轻，保守策略是正确的默认选择。


#### Case 16.3.2: persistent_reduction下多余的r-axis mask

Commit: 41cd21e3b
Files: triton.py

Scenario: persistent_reduction模式下，单reduction轴场景的reduction结果不正确。

Constraints: persistent_reduction的核心语义是"一次处理完整个reduction轴"，
因此不需要tail mask(不存在tail block越界)。
但NPU codegen的reduction方法中，初始mask集合包含了所有axis的mask(含r*轴)。
包含多余的r-axis mask会导致部分元素被错误mask掉。

Decision: 当persistent_reduction=True且numof_reduction_axis()==1时，
从初始mask集合中排除r*开头的axis mask。

Alternatives:
- 在filter_masks()层面处理(但persistent_reduction的语义不应泄漏到通用过滤逻辑)
- 让上层构造mask时就不包含r-axis mask(需要修改mask构造的通用逻辑)

Consequence: 修复了persistent_reduction模式下的计算错误。

Transferable: persistent vs non-persistent reduction对mask的需求不同:
- non-persistent: 按block循环处理reduction轴，最后一个block需要tail mask
- persistent: 一次性处理完整轴，不需要tail mask
upstream CUDA codegen中这个区分是隐式处理的
(通过RBLOCK等constexpr自动消除)，但NPU的显式mask管理需要手动对齐。


#### Case 16.3.3: call_args vs call_arg -- 单字符typo导致static mode失败

Commit: 0f59193ef (部分)
Files: cpp_wrapper.py

Scenario: static mode下编译失败。

Constraints: list comprehension中过滤sympy.Integer参数时，
循环变量名call_arg被错写为call_args(外层变量名)。
判断的是整个列表而非单个元素，逻辑完全错误。
Python中{var: var}和变量名只差一个字符的情况不会在语法层面报错。

Decision: call_args修正为call_arg(单字符typo fix)。

Alternatives: 静态分析工具(如mypy strict mode)可以捕获这类变量scope混淆。

Consequence: static mode恢复正常。

Transferable: list comprehension中循环变量名和外层变量名的混淆
是Python中极易犯的错误，特别是当变量名只差一个字符时。
code review中这类错误极难肉眼发现。
防御: 循环变量使用与外层变量明确不同的命名(如item/elem/entry)，
不要用外层变量名加/减一个字符。


#### Case 16.3.4: compile_fx options参数传递 -- Python dict key陷阱

Commit: 315342892
Files: npugraph_ex/__init__.py

Scenario: 用户通过compile_fx(options={"clone_input": False})传入options dict时crash。

Constraints: 代码中写了`{options: options}`而非`{"options": options}`。
变量options可能为None，此时传入的dict是{None: None}，下游解析失败。
Python中用变量作为dict key是合法语法，不会报错。

Decision: 修正为`{"options": {} if options is None else options}`。
补充UT覆盖带options参数的调用路径。

Alternatives: 使用TypedDict或dataclass替代raw dict传递配置(类型安全)。

Consequence: compile_fx的options参数正常工作。

Transferable: Python中`{var: val}`和`{"key": val}`的混淆是常见typo。
当变量值恰好是合法的dict key类型(str, int, None)时不会报语法错误。
防御: (1) 配置传递使用dataclass/TypedDict (2) mypy strict mode检查dict literal


### 16.4 横切总结: Inductor NPU后端的三个核心挑战

8个case(含1个revert)揭示了NPU Inductor后端适配的三类反复出现的问题:

Challenge 1: golden_var_list一致性(Case 16.1.1, 16.1.2)
这是NPU独有概念(upstream不存在)，用于确定tile维度顺序。
有多种推导路径(stride排序、index分析)，不同路径可能产生不同结果。
且constant tensor不产生索引分析结果(为None)。
所有使用golden_var_list的代码都必须: (1) 处理None case (2) 确保推导路径一致。

Challenge 2: upstream IR演进跟踪(Case 16.2.1, 16.2.2)
upstream新增IR类型(Scan)或higher-order op(invoke_subgraph)时，
NPU downstream的隐式假设可能被打破。
"默认fallback + 白名单放行"策略需要持续维护。
upstream的CHANGELOG和IR变更是NPU后端维护者必须跟踪的上游信号。

Challenge 3: 显式mask管理(Case 16.3.1, 16.3.2)
NPU的tile/axis系统与CUDA不同，需要自定义filter_masks逻辑。
核心原则: 对不认识的mask保留(安全)而非丢弃。
persistent vs non-persistent reduction对mask需求不同，需要手动区分
(CUDA中由constexpr自动消除)。



