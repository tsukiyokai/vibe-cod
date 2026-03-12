# Phase 3.2: hccl Operator Implementations

## 3.2.1 all_reduce/ -- 最核心的集合通信算子

### Module Overview

all_reduce/ 是 hccl 最核心的算子，55个文件/9768行，是所有算子中最大的。
位于 `src/ops/all_reduce/`，遵循 executor/selector/template 三件套模式。

AllReduce 语义: 每个rank持有一份数据，操作后所有rank都获得全局reduce结果。
经典分解: AllReduce = ReduceScatter + AllGather。

### Key Files (按子模块)

| 子模块 | 文件数 | 行数 | 职责 |
|--------|--------|------|------|
| all_reduce_op.h/cc | 2 | 196 | C ABI入口, 参数校验, 委托Selector+HcclExecOp |
| selector/ | 2 | 386 | 22种算法的选择逻辑(5路执行模式) |
| executor/ | 8 | 1534 | 4种执行策略(Sole/Parallel/Concurrent/Sequence2Die) |
| template/aicpu/ | 8 | 1648 | AICPU模板(OneShot/TwoShot/MeshChunk/NHR) |
| template/aiv/ | 7 | 734 | AIV模板(OneShot/TwoShot) + device kernel |
| template/ccu/ | 28 | 5280 | CCU模板(7种) + kernel微码(7种) |

### Grep量化

| 模式 | 次数 | 文件数 |
|------|------|--------|
| CHK_RET | 197 | 22 |
| REGISTER_EXEC_V2 | 14 | 1 |
| REGISTER_EXECUTOR_BY_TWO_TEMPS | 8 | 4 |
| reinterpret_cast | 22 | 10 |
| HCCL_WARNING/ERROR/DEBUG/INFO | 297 | 30 |
| TODO/FIXME/HACK | 0 | 0 |

---

### Op Entry (all_reduce_op.cc)

#### 入口结构

HcclAllReduce (extern "C") 有三重门控:
1. CheckHCCLIndependentOp() -- 非独立算子模式 -> fallback HcclAllReduceInner
2. DevType != DEV_TYPE_910_95 -- 非A5芯片 -> fallback HcclAllReduceInner
3. GetWorkflowMode() != OP_BASE -- 图模式 -> fallback HcclAllReduceInner

通过门控后进入新路径:
```
InitEnvConfig -> recvCount==0 early return -> CheckAllReduceInputPara
-> GetRankSize/RankId/CommName -> tag构造("AllReduce_"+commName)
-> CheckTag/CheckUserRank/CheckCount/CheckDataType
-> AllReduceOutPlace
```

AllReduceOutPlace:
- 组装OpParam(stream/reduceType/opMode/tag/inputPtr/outputPtr/DataDes/opType)
- 单卡快速路径: rankSize==1 -> SingleRankProc -> return
- 正常路径: Selector(comm, param, topoInfo, algName, opExecuteConfig) -> HcclExecOp(comm, param, topoInfo, algName)

关键: op入口不做任何算法选择，完全委托op_common框架的Selector和HcclExecOp自由函数。

---

### Selector (all_reduce_auto_selector.cc)

```
REGISTER_SELECTOR_BY_OPTYPE(HCCL_CMD_ALLREDUCE, 18, AllReduceAutoSelector)
```

继承AutoSelectorBase, override 4个虚方法(CCU_MS/CCU_SCHED/AIV/AICPU)。
未override SelectDPUAlgo(默认NOT_MATCH)。

#### 5路分派与22种算法

基类分派链: DPU -> CCU_MS -> CCU_SCHED -> AIV -> AICPU(兜底)
每路NOT_MATCH则降级到下一路。

常量:
- AR_M2M_1D_MAX_DATA_SIZE = 16MB
- AR_AICPU_1D_SMALL_DATA_SIZE = 8MB
- AR_AICPU_1D_MAX_DATA_SIZE = 32MB
- MAX_RANK_NUM_FOR_CONCURRENT_ALGO = 4

#### CCU MS模式 (SelectCcuMsAlgo, 6种算法)

前置排除: 多级拓扑 / INT8 / PROD / FP64/UINT64

MESH_1D:
- TWO_DIE_REGULAR: small->CcuAllReduceMesh2Die, big->CcuAllreduceMesh2DieBigMs
- 普通: small->CcuAllReduceMesh1DOneShot, big->CcuAllReduceMesh1D

MESH_1D_CLOS (UBX):
- rankSize<=4: small->CcuAllReduceMesh1D, big->CcuAllReduceConcurrentMs
- closMultiple && big -> NOT_MATCH
- else -> CcuAllReduceMesh1D

#### CCU Schedule模式 (SelectCcuScheduleAlgo, 8种算法)

前置排除: PROD / 64位类型

多级拓扑 MESH_1D:
- deviceNumPerModule>1 && !INT8 -> CcuAllReduceParallelMesh1DNHR
- else -> CcuAllReduceNHR1D

单级拓扑 MESH_1D:
- INT8 -> NOT_MATCH
- dataSize>16MB -> NOT_MATCH(降级AICPU)
- TWO_DIE_REGULAR: small->CcuAllReduceMesh1DMem2Mem2DieOneShot, big->CcuAllreduceMesh2DieBigSche
- 普通 -> CcuAllReduceMesh1DMem2Mem

单级拓扑 MESH_1D_CLOS (UBX):
- rankSize<=4: small->CcuAllReduceMesh1DMem2Mem, big->CcuAllReduceConcurrentSche
- closMultiple && big -> CcuAllReduceParallelNHR1DMutiJetty
- else -> CcuAllReduceNHR1DMem2MemMultiJetty

#### AIV模式 (SelectAivAlgo, 2种算法)

前置排除: PROD / 64位类型

仅支持单级MESH_1D:
- small -> AivAllReduceMesh1DOneShot
- big -> AivAllReduceMesh1DTwoShot

#### AICPU模式 (SelectAicpuAlgo, 6种算法)

前置排除: PROD

多级拓扑:
- MESH_1D && deviceNumPerModule>1 -> InsAllReduceParallelMesh1DNHR
- else -> InsAllReduceNHR
- 64位类型后排除

单级拓扑 MESH_1D:
- <=8MB -> InsAllReduceMesh1DOneShot
- >32MB -> InsAllReduceMesh1DTwoShotMeshChunk
- 8-32MB -> InsAllReduceMesh1DTwoShot

单级拓扑 MESH_1D_CLOS (UBX):
- rankSize<=4: small->InsAllReduceMesh1DOneShot, big->InsAllReduceConcurrent
- closMultiple && big -> InsAllReduceParallelMesh1DNHR
- else -> InsAllReduceNHR

#### 算法总览表 (22种)

| 算法名 | 执行模式 | Engine | 拓扑 | 数据量 | 特殊条件 |
|--------|---------|--------|------|--------|---------|
| CcuAllReduceMesh2Die | CCU_MS | CCU | MESH_1D | small | 2die regular |
| CcuAllreduceMesh2DieBigMs | CCU_MS | CCU | MESH_1D | big | 2die regular |
| CcuAllReduceMesh1DOneShot | CCU_MS | CCU | MESH_1D | small | 非2die |
| CcuAllReduceMesh1D | CCU_MS | CCU | MESH_1D | big | 非2die |
| CcuAllReduceConcurrentMs | CCU_MS | CCU | UBX | big | rankSize<=4 |
| CcuAllReduceParallelMesh1DNHR | CCU_SCHED | CCU | multi-level | - | module内多卡 |
| CcuAllReduceNHR1D | CCU_SCHED | CCU | multi-level | - | module单卡 |
| CcuAllReduceMesh1DMem2Mem2DieOneShot | CCU_SCHED | CCU | MESH_1D | small | 2die, <=16MB |
| CcuAllreduceMesh2DieBigSche | CCU_SCHED | CCU | MESH_1D | big | 2die, <=16MB |
| CcuAllReduceMesh1DMem2Mem | CCU_SCHED | CCU | MESH_1D | - | 非2die, <=16MB |
| CcuAllReduceConcurrentSche | CCU_SCHED | CCU | UBX | big | rankSize<=4 |
| CcuAllReduceParallelNHR1DMutiJetty | CCU_SCHED | CCU | UBX | big | closMultiple |
| CcuAllReduceNHR1DMem2MemMultiJetty | CCU_SCHED | CCU | UBX | - | else |
| AivAllReduceMesh1DOneShot | AIV | AIV | MESH_1D | small | 单级 |
| AivAllReduceMesh1DTwoShot | AIV | AIV | MESH_1D | big | 单级 |
| InsAllReduceParallelMesh1DNHR | AICPU | AICPU | multi-level | - | module内多卡 |
| InsAllReduceNHR | AICPU | AICPU | multi/UBX | - | 兜底 |
| InsAllReduceMesh1DOneShot | AICPU | AICPU | MESH_1D | <=8MB | 单级 |
| InsAllReduceMesh1DTwoShot | AICPU | AICPU | MESH_1D | 8-32MB | 单级 |
| InsAllReduceMesh1DTwoShotMeshChunk | AICPU | AICPU | MESH_1D | >32MB | 单级 |
| InsAllReduceConcurrent | AICPU | AICPU | UBX | big | rankSize<=4 |

---

### Executor (4种执行策略)

全部继承InsCollAlgBase (V2), 都是类模板, 通过模板参数注入AlgTopoMatch策略和算法template。

#### Sole (InsV2AllReduceSoleExecutor) -- 最通用

- 14个注册tag, 覆盖AICPU/AIV/CCU三种engine
- 1个template, 不切分数据(仅loop)
- Orchestrate: 计算maxDataCountPerLoop, 循环调template.KernelRun
- 尾块处理: currCount = (loop==last) ? total-processed : maxPerLoop

#### Parallel (InsAllReduceParallelExecutor) -- 双轴并行

- 3个注册tag (Multilevel/UBX拓扑)
- 2个template(intra+inter), 数据50:50等分
- 两步流水+换轴: Stage0 intra处理data0+inter处理data1,
  Stage1 数据"换轴"使两种链路都参与所有数据的reduce
- PreSyncInterThreads/PostSyncInterThreads做步间同步
- 线程数 = 两template之和 + 2(额外主线程)

#### Concurrent (InsV2AllReduceConcurrentExecutor) -- 端口比例并行

- 3个注册tag (UBX拓扑)
- 2个template, 按端口带宽比例切分(mesh端口数:CLOS端口数=4)
- while循环中交替下发两个template的KernelRun
- std::copy_if过滤COMM_PROTOCOL_UBC_CTP协议的channel
- 线程数 = 两template之和 + 1

#### Sequence2Die (InsV2AllReduceSequence2DieExecutor) -- 经典RS+AG分解

- 2个注册tag (1D拓扑, 2-die场景)
- 2个template(RS+AG), 串行两步
- 数据接力: RS输出到HCCL_BUFFER -> AG从HCCL_BUFFER读取
- 线程/notify取std::max而非相加(串行复用)
- CCU模式下做resGroup标记区分两步kernel

#### Executor对比表

| 维度 | Sole | Parallel | Concurrent | Sequence2Die |
|------|------|----------|------------|-------------|
| Template数 | 1 | 2 | 2 | 2(RS+AG) |
| 数据切分 | 不切分(loop) | 50:50等分 | 端口比例 | rank切片 |
| 执行方式 | 单template串行 | 双template并行+换轴 | 交替下发 | 串行RS->AG |
| 拓扑 | 1D | Multilevel/UBX | UBX | 1D(2-die) |
| 资源合并 | 透传 | 线程=和+2 | 线程=和+1 | max(复用) |

---

### AICPU Templates (4种)

全部继承InsAlgTemplateBase, 在namespace ops_hccl中。
RS+AG在模板内部直接实现(私有方法), 不通过GetAlgTemplate组合子模板。

#### OneShot (InsTempAllReduceMesh1DOneShot)

- 全对全直接通信: 每rank广播完整数据给所有其他rank, 然后本地reduce
- 通信量: O(N*D)
- 从线程: rankSize-1个(每个负责与一个peer通信)
- scratch倍率: rankSize (所有rank数据都复制到本地)
- KernelRun: PreSync -> LocalCopy(input->output) -> 并行SendRecvWrite -> PostSync -> 顺序LocalReduce
- 适用: 小数据量, rank数少

#### TwoShot (InsTempAllReduceMesh1DTwoShot)

- 经典ReduceScatter + AllGather两阶段
- 通信量: O(2*(N-1)/N * D)
- 从线程: rankSize-1个
- scratch倍率: 2
- KernelRun: SplitData -> RunReduceScatter(PreSync->ScatterData->PostSync->ReduceData) -> RunAllGather(PreSync->GatherData->PostSync)
- 适用: 8-32MB, 标准mesh拓扑

#### MeshChunk (InsTempAllReduceMesh1DTwoShotMeshChunk)

- TwoShot的流水线变体, RS阶段二级切分
- chunk-based通信: (N-1)步, 每步处理不同chunk
- 使用SendRecvWriteReduce(通信+reduce融合)
- PreCopy只拷贝本rank对应切片(非全量)
- 步间额外同步(PostSync+PreSync交替)
- 适用: >32MB, 通过pipeline隐藏通信延迟

#### NHR (InsTempAllReduceNHR)

- Non-uniform Hierarchical Ring
- 单线程(无从线程), log2(rankSize)步
- 每步通信对象: deltaRank = 2^step
- PreCopy(input->hcclBuffer) -> RS(log2步SendRecvWriteReduce) -> AG(log2步SendRecvWrite) -> PostCopy(hcclBuffer->output)
- scratch倍率: 1
- 适用: 跨层级通信, 非2幂rankSize也支持

#### AICPU Template对比

| 维度 | OneShot | TwoShot | MeshChunk | NHR |
|------|---------|---------|-----------|-----|
| 通信量 | O(N*D) | O(2D) | O(2D) | O(2D) |
| 步数 | 1 | 2*N | 2*(N-1) | 2*log2(N) |
| 从线程 | N-1 | N-1 | N-1 | 0 |
| scratch倍率 | N | 2 | 2 | 1 |
| 通信+计算融合 | 否 | 否 | 是(SWWR) | 是(SWWR) |

---

### AIV Templates (2种)

全部继承AivAlgTemplateBase, 设备侧继承AivCommBase。

#### Host侧

CalcRes: threadNum=1(无slave线程), CalcChannelRequestMesh1D
CalcScratchMultiple: oneshot=rankSize, twoshot=2
CalNumBlocks: oneshot=rankSize+1核, twoshot=coreNumPerRank*(rankSize+1)核
KernelRun: 填充AivOpArgs(地址/count/topo/tag) -> CalNumBlocks -> ExecuteKernelLaunch

aiv_communication_v2.h: 通过AIV_ATOMIC_DATA_TYPE_DEF宏为6种数据类型批量注册kernel入口。
每个kernel用EXPORT_AIV_META_INFO在ELF中嵌入K_TYPE_AIV元数据。

#### AIV Oneshot (AivAllReduceMesh1DOneShot<T>)

Producer-Consumer核分工:
- Producer核(block_idx 0..rankSize-1): CpGM2GM(本rank数据->目标rank的IPC buffer) + pipe_barrier + Record
- Consumer核(block_idx==rankSize): WaitFlag等每个rank数据到达 -> CpGM2GM(带reduceOp的原子reduce)

数据流: 每rank广播完整数据到所有rank的IPC buffer, Consumer核串行reduce。

#### AIV TwoShot (AivAllReduceMesh1DTwoShot<T>)

经典两阶段, 三种核角色:
- Scatter核: 将input[target的chunk]复制到GM_IN[target][本rank的slot]
- Reduce核: WaitFlag等所有rank数据到达 -> CpGM2GM(带reduceOp逐rank reduce)
- Gather核(复用Scatter核): 将reduce结果从GM_IN[target]复制到output[target的chunk]

支持每rank多核并行: coreNumPerRank = numBlocks/(rankSize+1)

#### GM flag同步机制

- Record(targetRank, flag_offset, curTag): 将curTag写入GM_OUT[target]+flag_offset*32B
- WaitFlag(targetRank, flag_offset, curTag): 忙等读GM_OUT[target]+flag_offset*32B直到==curTag
- BarrierForFirstOP: 仅首个kernel首次执行时全rank barrier(tag>>16==1 && tag&0xFFFF==1)
- 三段式同步: SyncAll -> BarrierForFirstOP -> SyncAll -> 业务 -> BarrierAll

---

### CCU Templates (7种template + 7种kernel)

全部继承CcuAlgTemplateBase, kernel通过CcuKernel API构造微码IR。

#### Template层共同模式

- 构造函数: std::find+std::distance计算子通信域rank
- CalcRes: 创建kernel, 计算channel(Mesh/NHR), die分组
- KernelRun: 计算slice切分 -> 序列化taskArg -> HcclCcuKernelLaunch

#### CCU编程模型(编译器式)

1. 前端(Kernel API): CreateVariable/Load/NotifyRecord/NotifyWait/GroupReduce/GroupBroadcast/WriteReduceNb/WriteNb/ReadNb/CCU_IF
2. 中间表示(CcuRep IR): Variable/LocalAddr/RemoteAddr/CompletedEvent/GroupOpSize
3. 后端(Microcode): GeneArgs序列化为uint64数组

同步:
- 跨rank: NotifyRecord(channel, ckeIdx, bit) + NotifyWait(channel, ckeIdx, bit)
- CKE有16位信号掩码, 不同用途分配不同bit
- 步间同步: NHR每步用独立的STEP*_PRE/POST_SYNC bit

事件模型:
- WriteNb/ReadNb非阻塞, 完成设置event对应bit
- WaitEvent(event)等待指定mask的所有bit
- 每16个非阻塞操作WaitEvent一次(RANK_NUM_PER_CKE=16)

#### Kernel Algorithm()统一生命周期

```
InitResource() -> LoadArgs() -> [LocalCopy()] -> PreSync() -> DoXxx() -> PostSync()
```

#### 7种CCU Template特征

| Tag | 拓扑 | 算法 | mem2mem | die | 特征 |
|-----|------|------|---------|-----|------|
| CcuAllReduceMesh1D | mesh | RS+AG | 否 | 1 | GroupReduce/GroupBroadcast高阶API |
| CcuAllReduceMesh1DOneShot | mesh | oneshot | 否 | 1 | 小数据, 全交换 |
| CcuAllReduceMesh2Die | mesh | oneshot | 否 | 2 | 2die, channel按dieId分组 |
| CcuAllReduceMesh1DMem2Mem | mesh | RS+AG | 是 | 1 | ReadNb+本地Reduce, LoopGroup机制 |
| CcuAllReduceMesh1DMem2Mem2DieOneShot | mesh | oneshot | 是 | 2 | 2die+mem2mem |
| CcuAllReduceNHR1D | NHR | NHR | 是 | 1/2自适应 | WriteReduceNb, log2步, GetDieNumFromChannelDescs |
| CcuAllReduceNHR1DMem2MemMultiJetty | NHR | NHR | 是 | - | 4端口并行, enum class类型安全, 最成熟 |

#### mesh vs NHR 的CCU实现差异

| 维度 | mesh | NHR |
|------|------|-----|
| 步数 | O(1)(RS+AG) | O(logN)(RS+AG) |
| 通信API | GroupReduce/GroupBroadcast(高阶) | WriteReduceNb/WriteNb(低阶) |
| channel使用 | 同时用所有channel | 每步只用1对channel |
| 同步复杂度 | 仅PreSync+PostSync | 每步步间同步 |
| 适用 | rank少, 全连接 | rank多, 非2幂次 |

#### mem2mem vs 非mem2mem

- 非mem2mem: 远端数据由硬件通过GroupReduce自动拉取, 无需scratch
- mem2mem: 本地主动写远端(WriteReduceNb), 需要scratch, 支持大数据量分块流水(LoopGroup)

#### 2die处理

- 非2die: 1个kernel, 所有channel在一个kernel内
- 2die: 2个kernel, channel按dieId分组, 需要die间同步
- NHR 2die: axisId_(0/1)决定die偏移, die0SliceSize_/die1SliceSize_区分数据量

---

### Idioms & Conventions (惯用法)

AR-ID-1. CHK_RET 100%覆盖 -- 197次/22文件, 所有可能失败的调用都检查
  出处: all_reduce_op.cc:47,58,62-74 (17处)

AR-ID-2. 三重门控渐进迁移 -- 新路径三个条件依次检查, 不满足则fallback旧路径
  出处: all_reduce_op.cc:43-55

AR-ID-3. 单卡快速路径 -- rankSize==1直接SingleRankProc, 跳过selector和executor
  出处: all_reduce_op.cc:143-147

AR-ID-4. NOT_MATCH降级链 -- 每路返回NOT_MATCH尝试下一路, AICPU是兜底
  出处: all_reduce_auto_selector.cc (整体结构)

AR-ID-5. 数据量三档分流 -- small/medium/big使用不同算法(OneShot/TwoShot/Pipeline)
  出处: selector:74,79,161,169,278,288,321,333

AR-ID-6. Producer-Consumer核分工 -- AIV kernel按GetBlockIdx()区间分配角色
  出处: aiv_all_reduce_mesh_1d_oneshot.h:57-90, twoshot.h:42-118

AR-ID-7. CpGM2GM + pipe_barrier(PIPE_ALL) -- GM搬运后必跟barrier确保DMA完成
  出处: aiv_all_reduce_mesh_1d_oneshot.h:30-31, twoshot.h:92-93

AR-ID-8. CCU Algorithm()统一生命周期 -- InitResource->LoadArgs->PreSync->Do->PostSync
  出处: 所有7个ccu_kernel_*.cc

AR-ID-9. event批量等待 -- 每RANK_NUM_PER_CKE(16)个非阻塞操作WaitEvent一次
  出处: ccu_kernel_all_reduce_nhr_mem2mem_1D_multi_jetty.cc

AR-ID-10. 类模板+Strategy注入 -- Executor通过模板参数注入TopoMatch和Template策略
  出处: ins_v2_all_reduce_sole_executor.h, parallel_executor.h, concurrent_executor.h

AR-ID-11. RPT_INPUT_ERR + CHK_PTR_NULL配对 -- API入口先报用户友好错误, 再做空检查
  出处: all_reduce_op.cc:86-97

AR-ID-12. tag构造模式 -- "AllReduce_" + commName, 每个算子类型唯一tag前缀
  出处: all_reduce_op.cc:69

---

### Architecture & Patterns (架构模式)

AR-AP-1. 三层分工: Op(入口+参数校验) -> Selector(算法选择) -> Executor(编排) -> Template(算法实现)
  Op只组装OpParam, 不做任何算法决策; Selector通过Registry自注册; Executor通过Registry按tag查找

AR-AP-2. Executor 4种Strategy: Sole(通用单template) / Parallel(双轴并行+换轴) / Concurrent(端口比例) / Sequence2Die(经典RS+AG)
  通过模板参数注入, 同一Executor类服务多种拓扑和engine

AR-AP-3. AllReduce = RS + AG 分解在不同层级:
  - Executor层: Sequence2Die显式分解, 其他3种将AllReduce视为整体
  - Template层: TwoShot/MeshChunk/NHR内部实现RS+AG, OneShot不分解
  - Executor层组合: Parallel/Concurrent通过REGISTER_EXECUTOR_BY_TWO_TEMPS组合两个template(如TwoShot+NHR)

AR-AP-4. 三种设备编程模型根本不同:
  - AICPU: Host主控, 多slave线程并行通信, PreSync/PostSync线程同步
  - AIV: 设备侧直接执行, GM flag busy-poll同步, Producer/Consumer核分工
  - CCU: 编译器式DSL, Host构造微码IR, CKE硬件同步, 非阻塞+event等待

AR-AP-5. 选择矩阵: 执行模式(5路) x 拓扑(4种) x 数据量(3档) x 硬件(2die/数据类型)
  22种算法通过selector的决策树和executor+template的注册组合实现

AR-AP-6. multi_jetty代表最新设计方向:
  - enum class(非裸int)做信号位, 更类型安全
  - channelIdxMap_(非遍历)做查找
  - 多端口并行传输
  - CalcScratchMultiple=0(直接写对端output, 无需scratch)
  - 错误处理最完整(CHK_RET 38次/单文件)

---

### Domain Knowledge (业务知识)

AR-DK-1. AllReduce分解原理:
  - AllReduce = ReduceScatter + AllGather (通信量最优: O(2*(N-1)/N * D))
  - OneShot: 全交换+本地reduce (O(N*D), 仅小数据适用)
  - NHR: log2步递归减半 (O(2D), 适合大规模/非2幂)
  - MeshChunk: chunk流水线 (O(2D), 大数据隐藏延迟)

AR-DK-2. 数据量阈值:
  - 小数据/大数据分界: 512KB/1MB(基类IsSmallData/IsLargeData)
  - AICPU OneShot/TwoShot分界: 8MB
  - AICPU TwoShot/MeshChunk分界: 32MB
  - CCU Schedule mem2mem上限: 16MB(超过降级AICPU)

AR-DK-3. UBX拓扑特殊处理:
  - MESH_1D_CLOS拓扑, 有mesh端口和CLOS端口
  - rankSize<=4: Concurrent(同时利用两种链路)
  - closMultiple: Parallel(NHR跨节点)
  - else: NHR兜底

AR-DK-4. 2-die场景:
  - channel按dieId分组, 每die一个kernel
  - TWO_DIE_REGULAR vs NOT_REGULAR决定是否支持(NOT_REGULAR大部分返回NOT_MATCH)
  - rmtReduceWithMyRank: channel少的die负责包含本rank数据的reduce

AR-DK-5. mem2mem的意义:
  - 非mem2mem: 硬件自动remote read + reduce(GroupReduce, 高层API)
  - mem2mem: 软件主动read+write+reduce(ReadNb/WriteReduceNb, 低层API)
  - mem2mem支持更灵活的内存编排和大数据分块流水(LoopGroup)

AR-DK-6. 数据类型限制:
  - PROD运算: 所有CCU/AIV路径NOT_MATCH, 只有AICPU(仍排除PROD, 实际AICPU也不支持)
  - 64位类型(FP64/INT64/UINT64): CCU Schedule和AIV不支持, 只有AICPU且受限
  - INT8: CCU MS不支持, CCU Schedule module内不支持

---

### Anti-Patterns (反模式)

#### 确认bug

AP-AR-1. FP64重复/INT64遗漏 (selector:42-44)
  第三个条件应是HCCL_DATA_TYPE_INT64但写成重复的FP64, 日志说不支持INT64但代码不检查。
  与Phase 3.1.2发现的模式一致(3个selector文件同样bug)。

AP-AR-2. NHR recv slice构造错误 (ins_temp_all_reduce_nhr.cc:210-211, 266-267)
  recvSrcSlicesList.emplace_back(sendSrcSlice) -- 应为recvSrcSlice。
  RunReduceScatter和RunAllGather中都有, 导致NHR使用错误的收端slice信息。

AP-AR-3. Parallel executor CalcSendDataSize赋值错误 (parallel_executor.cc:305-306, 320)
  interHcclBuffSizeStage0_被赋值两次, 应有一次是interHcclBuffSizeStage1_。
  导致multipleInter==0&&multipleIntra>0分支中interHcclBuffSizeStage1_永远为0。

AP-AR-4. Sequence2Die CalcRes缺CHK_RET (sequence2die_executor.cc:56-57)
  两个CalcRes调用返回值被忽略, 错误会被静默吞掉。

AP-AR-5. CCU HcclCcuKernelLaunch未检查返回值 (ccu_temp_all_reduce_mesh_1D_mem2mem_2die_oneshot.cc:106)
  其他所有template都用CHK_RET, 唯独此处遗漏。

AP-AR-6. CCU NHR multi_jetty LocalCopySlices goSize选择反了 (ccu_kernel_all_reduce_nhr_mem2mem_1D_multi_jetty.cc:192)
  islastSlice为true时应用lastSlice的goSize, 但代码用了正常goSize。

AP-AR-7. AIV twoshot SmallCoreAllgather用return而非continue (aiv_all_reduce_mesh_1d_twoshot.h:213-215)
  rankChunkSize<=0时return导致后续rank的gather都不执行, 应用continue。

#### copy-paste错误

AP-AR-8. sendBuf提示写成recvBuf (all_reduce_op.cc:93)
  sendBuf为空时tips说"please check recvBuf"。

AP-AR-9. TwoShot GatherData日志写成ScatterData (ins_temp_all_reduce_mesh_1D_two_shot.cc:323,349,357,365)
  4处GatherData方法中的错误日志标记[ScatterData]。

AP-AR-10. CCU template日志类名大量错误
  - ccu_temp_all_reduce_mesh_1D_one_shot.cc:91 写成CcuTempReduceScatterMeshMem2Mem1D
  - ccu_temp_all_reduce_nhr_1D_mem2mem.cc:60,78,87 写成CcuTempReduceScatterNHR1DMem2Mem
  - ccu_temp_all_reduce_mesh_1D_mem2mem.cc:97 写成CcuTempReduceScatterMesh1DMem2Mem
  反映项目级反模式: 从reduce_scatter模板copy创建all_reduce, 未全面更新日志类名。

AP-AR-11. Sequence2Die RoundUp日志写错类名 (sequence2die_executor.cc:146)
  InsV2AllReduceSequence2DieExecutor中写成InsTempAllReduceMesh1DTwoShot。

AP-AR-12. AICPU Describe()拼写错误: "resduce"应为"reduce" (one_shot.h:29)
  MeshChunk Describe()写成"reduce scatter"而非"all reduce" (mesh_chunk.h:30)

#### 设计/代码质量问题

AP-AR-13. 算法名大小写不一致
  AllReduce vs Allreduce(小写r): CcuAllReduceMesh2Die vs CcuAllreduceMesh2DieBigMs。
  出处: selector:71,73,167

AP-AR-14. levle0Algo拼写错误 -- 8个文件57处一致出现(Phase 3.1.2已发现)
  出处: selector:49,149

AP-AR-15. SIZE_TABLE vs DATATYPE_SIZE_TABLE不一致
  parallel/sole用DATATYPE_SIZE_TABLE, concurrent/sequence2die用SIZE_TABLE。
  出处: parallel:113 vs concurrent:38,159

AP-AR-16. 64位检查写法三种不一致
  手动枚举3类型(selector:243) vs 手动枚举3类型(selector:259) vs Is64BitDataType() helper(selector:123)。

AP-AR-17. opExecuteConfig冗余 (all_reduce_op.cc:149,152)
  Selector填充但HcclExecOp不接受, 变量浪费。

AP-AR-18. CalcRes中HCCL_WARNING "Resource calculation is temporarily not performed"
  出处: one_shot.cc:38, mesh_chunk.cc:41, aiv oneshot.cc:48, aiv twoshot.cc:48
  生产环境每次调用都打WARNING, 说明资源计算尚未完整实现。

AP-AR-19. AICPU模板重复代码
  - SplitData在TwoShot和NHR中几乎完全相同(应提取到基类)
  - SliceInfo/SplitSliceInfo/NHRSliceInfo三种结构表达同一概念
  - CalcSlice/CalcSliceInfoVec/SplitData三种切片函数本质相同

AP-AR-20. Parallel GetParallelDataSplit硬编码0.5 (parallel_executor.cc:156)
  注释"// to do"说明未来要改为动态比例, 但已发布。

AP-AR-21. CCU LINK_NUM_1和LINK_NUM_2值相同(都=2) (ccu_temp_all_reduce_nhr_1D_mem2mem.cc:40-41)
  命名暗示LINK_NUM_1可能应为1, 表示"同die=1个die"。

AP-AR-22. CCU格式化日志用[]而非%占位符 (ccu_temp_all_reduce_nhr_1D_mem2mem.cc:60)
  "there are [] link to rank []" -- 不会打印具体数值。

AP-AR-23. MeshChunk RunReduceScatter返回值未检查 (mesh_chunk.cc:151)
  ReduceScatterMeshChunk缺CHK_RET。

AP-AR-24. CCU 2die oneshot定义了repeatNum等局部变量但未使用 (2die_oneshot.cc:109-120)

AP-AR-25. AIV CalcRes中threadNum硬编码1, for循环是死代码 (oneshot.cc:37-42, twoshot.cc:37-42)

AP-AR-26. Namespace注释不一致 -- namespace ops_hccl但结尾写} // namespace Hccl

---

### Notable Patterns (值得注意的模式)

1. all_reduce是模式发源地: 其他算子(reduce_scatter/all_gather/broadcast)都从all_reduce copy-paste而来,
   这从CCU模板日志中全写成ReduceScatter类名可以确认。all_reduce的bug和反模式会扩散到所有算子。

2. CCU模板演进轨迹: mesh_1D(最简) -> one_shot/2die(特化) -> mem2mem(灵活) -> multi_jetty(最新最成熟)。
   multi_jetty使用enum class/channelIdxMap/scratch=0等改进, 代表未来方向。

3. 22种算法的维度正交性: 执行模式 x 拓扑 x 引擎 x 数据量 x 2die。
   但并非所有组合都存在, 很多组合返回NOT_MATCH。实际算法选择是稀疏矩阵。

4. RS+AG组合的三种层级:
   - Executor层组合: Sequence2Die, Parallel(两template)
   - Template内部组合: TwoShot/MeshChunk/NHR的私有方法
   - Kernel层: CCU的DoReduceScatter+DoAllGather, AIV的scatter+reduce+gather核
   同一个数学分解在每层都有不同粒度的实现。

---

## 3.2.3 reduce_scatter/ -- AllReduce的RS半

### Module Overview

reduce_scatter/ 是 AllReduce = RS + AG 分解中的 ReduceScatter 半，49个文件，
位于 `src/ops/reduce_scatter/`。与 all_gather/(46个文件)结构对称，
比 all_reduce/(55个文件)多出 meshchunk、dpu、multi_jetty 的独立模板变体。

ReduceScatter 语义: N 个 rank 各持有一份数据(大小 count*N)，操作后每个 rank
获得自己对应的 1/N 切片的全局 reduce 结果(大小 count)。

RS 与 AG 的反向对称: RS 先内后外(intra then inter)，AG 先外后内(inter then intra)。
RS 需要 reduce 计算(GroupReduce/LocalReduce/WriteReduceNb)，AG 只需要拷贝(GroupCopy/LocalCopyNb/WriteNb)。

### Key Files (按子模块)

| 子模块 | 文件数 | 职责 |
|--------|--------|------|
| reduce_scatter_op.h/cc | 2 | C ABI入口, 参数校验, 委托Selector+HcclExecOp |
| selector/ | 2 | 20+种算法的选择逻辑(5路执行模式) |
| executor/ | 10 | 4种执行策略(Sole/Parallel/Concurrent/Sequence) |
| template/aicpu/ | 8 | AICPU模板(Mesh1D/MeshChunk/NHR/DPU) |
| template/aiv/ | 5 | AIV模板(Mesh1D) + device kernel |
| template/ccu/ | 22 | CCU模板(7种) + kernel微码(7种) |

### 与 all_reduce 的结构对比

| 维度 | all_reduce | reduce_scatter | 差异原因 |
|------|-----------|---------------|---------|
| 总文件数 | 55 | 49 | AR多: AIV TwoShot/OneShot分离; RS多: DPU模板 |
| Executor种类 | 4(Sole/Parallel/Concurrent/Seq2Die) | 4(Sole/Parallel/Concurrent/Sequence) | 名称不同但1:1对应 |
| AICPU模板 | 4(OneShot/TwoShot/MeshChunk/NHR) | 4(Mesh1D/MeshChunk/NHR/DPU) | RS无TwoShot(RS不含AG阶段); RS多DPU |
| AIV模板 | 2(OneShot/TwoShot) | 1(Mesh1D) | RS只有OneShot风格; AR的TwoShot含RS+AG两阶段 |
| CCU Kernel | 7 | 7 | 1:1对应, RS用GroupReduce替换AR的GroupReduce+GroupBroadcast |
| 注册数 | 22(14 EXEC_V2 + 8 TWO_TEMPS) | 20(10 EXEC_V2 + 7 TWO_TEMPS + 2 TEMPLATE_V2 + 1 DPU) | RS多DPU注册 |
| 确定性变体 | 无 | 无 | 两者都没有deterministic变体 |

---

### Op Entry (reduce_scatter_op.cc, 153行)

#### 入口结构

HcclReduceScatter (extern "C") 三重门控与 AllReduce 完全相同:
1. CheckHCCLIndependentOp() -- 非独立算子模式 -> fallback HcclReduceScatterInner
2. DevType != DEV_TYPE_910_95 -- 非A5芯片 -> fallback HcclReduceScatterInner
3. GetWorkflowMode() != OP_BASE -- 图模式 -> fallback HcclReduceScatterInner

RS 特有参数: HcclReduceOp op (AllGather没有此参数)。

```
// reduce_scatter_op.cc:39-41
HcclResult HcclReduceScatter(void *sendBuf, void *recvBuf, uint64_t recvCount,
    HcclDataType dataType, HcclReduceOp op, HcclComm comm, aclrtStream stream)
```

ReduceScatterOutPlace 与 AllReduceOutPlace 结构完全一致:
- 组装 OpParam(含 reduceType = op)
- rankSize==1 -> SingleRankProc 快速返回
- Selector -> HcclExecOp

RS 特有的输入/输出大小关系:
- inputSize = recvCount * perDataSize * userRankSize (每rank持有全量)
- outputSize = recvCount * perDataSize (每rank只保留1/N)
  出处: reduce_scatter_op.cc:110-112

RS 特有限制: sendBuf == recvBuf 直接返回 HCCL_E_PARA (不支持 in-place)
  出处: reduce_scatter_op.cc:97-98

---

### Selector (reduce_scatter_auto_selector.cc, 353行)

```
REGISTER_SELECTOR_BY_OPTYPE(HcclCMDType::HCCL_CMD_REDUCE_SCATTER, 18, ReduceScatterAutoSelector)
```

继承 AutoSelectorBase, override 5个虚方法(CCU_MS/CCU_SCHED/AIV/AICPU/DPU)。
比 AllReduce 多一个 SelectDPUAlgo override。

常量:
- RS_2D_SMALL_DATA_SIZE = 1MB (用于多级拓扑的小数据判断)
- MAX_RANK_NUM_FOR_CONCURRENT_ALGO = 4

#### RS Selector 与 AR Selector 的差异

1. RS 有 DPU 路径(AR 没有): 多级拓扑 -> InsV2ReduceScatterSequenceMeshMesh
2. RS AICPU 有 CLOS 拓扑支持(AR 没有明确的 CLOS 路径)
3. RS AICPU 的 UBX 路径更复杂: 有 ratio-based 阈值判断(8卡基线系数)
   出处: reduce_scatter_auto_selector.cc:264-275
4. RS 有 SelectMeshAlgoAicpu 私有辅助方法(AR 直接内联)
5. RS CCU_MS 模式没有数据量分档: 2die固定用CcuReduceScatterMesh2Die(AR分big/small)

#### 5路分派与20+种算法

| 路径 | 算法名 | 条件 |
|------|--------|------|
| CCU_MS | CcuReduceScatterMesh2Die | MESH_1D, 2die regular |
| CCU_MS | CcuReduceScatterMesh1D | MESH_1D, 普通 |
| CCU_MS | CcuReduceScatterConcurrentMeshNHRMs | UBX, rank<=4, 大数据 |
| CCU_SCHED | CcuReduceScatterParallelMesh1DNHR | 多级MESH_1D, module多卡 |
| CCU_SCHED | CcuReduceScatterNHR1DMem2Mem | 多级MESH_1D, module单卡 |
| CCU_SCHED | CcuReduceScatterMeshMem2Mem1D2Die | MESH_1D, 2die regular |
| CCU_SCHED | CcuReduceScatterMesh1DMem2Mem | MESH_1D, 普通 |
| CCU_SCHED | CcuReduceScatterConcurrentMeshNHRSche | UBX, rank<=4, 大数据 |
| CCU_SCHED | CcuReduceScatterParallelMesh1DNHRMultiJetty | UBX, closMultiple, 大数据 |
| CCU_SCHED | CcuReduceScatterNhr1DMem2MemMultiJetty | UBX, 其他 |
| AIV | AivReduceScatterMesh1D | 仅MESH_1D |
| AICPU | InsReduceScatterParallelMesh1DNHR | 多级, module多卡 |
| AICPU | InsReduceScatterNHR | 多级, module单卡; CLOS; UBX兜底 |
| AICPU | InsReduceScatterMesh1D | MESH_1D, 小数据 |
| AICPU | InsReduceScatterMesh1DMeshChunk | MESH_1D, 大数据 |
| AICPU | InsReduceScatterConcurrentMeshNHR | UBX, 全连接, 大数据 |
| AICPU | InsReduceScatterAicpuReduce | 64位数据类型(幽灵算法, 见反模式) |
| DPU | InsV2ReduceScatterSequenceMeshMesh | 多级拓扑 |

注: 与AllReduce不同, RS Selector没有数据量上限导致CCU降级AICPU的逻辑
(AR有AR_M2M_1D_MAX_DATA_SIZE=16MB限制)。

注: RS AIV只有1种算法(AR有2种: OneShot+TwoShot), 因RS不需要AG阶段。

---

### Executor (4种执行策略, 10文件)

与 AllReduce 的 4 种策略1:1对应, 结构高度一致。

#### Sole (InsV2ReduceScatterSoleExecutor, 193行)

- 10个注册tag(3 AICPU + 1 AIV + 6 CCU), 是注册最多的executor
- 与AR的Sole几乎完全相同: OrchestrateLoop计算maxDataCountPerLoop后循环调template.KernelRun
- RS特有: inputSliceStride = dataSize_ (总输入大小, 因RS每rank的输入是完整数据)
  而非AR的outputSliceStride = maxDataCountPerLoop * dataTypeSize_
  出处: ins_v2_reduce_scatter_sole_executor.cc:147-148

#### Parallel (InsReduceScatterParallelExecutor, 431行)

- 3个注册tag(1 AICPU + 2 CCU, 其中1个含MultiJetty)
- RS的两步流水与AR完全对称但方向相反:
  - RS Stage1: intra0(mesh RS, 先内) + inter1(NHR RS, 同时处理另一半)
  - RS Stage2: inter0(NHR RS) + intra1(mesh RS)
  - AR Stage1: intra0(mesh AR) + inter1(NHR AR)
- 数据50:50等分, 注释"to do 先做等分"(与AR一致)
  出处: ins_reduce_scatter_parallel_executor.cc:224-229

#### Concurrent (InsReduceScatterConcurrentExecutor, 369行)

- 3个注册tag(1 AICPU + 2 CCU)
- 与AR的Concurrent完全相同: 按端口带宽比例切分(mesh:CLOS = portNum:4)
- portNum = 4 硬编码
  出处: ins_reduce_scatter_concurrent_executor.cc:213

#### Sequence (InsV2ReduceScatterSequenceExecutor, 256行)

- 1个注册tag(DPU路径)
- 与AR的Sequence2Die不同: RS的Sequence是DPU卸载(先inter再intra), 而非RS+AG分解
- 串行执行: 先inter(framework内部RS) -> 再intra(DPU外部RS)
- 使用InsTempReduceScatterMesh1D + InsTempReduceScatterMesh1dDpu两个模板
  出处: ins_v2_reduce_scatter_sequence_executor.cc:250-253

---

### AICPU Templates (4种, 8文件)

#### Mesh1D (InsTempReduceScatterMesh1D, 222行)

RS核心模板, 等价于AR的OneShot但只做RS阶段:
- 每rank将自己的数据write到所有peer的scratch buffer
- 本地reduce所有收到的数据

RS特有的PostCopy步骤:
- 通信结束后, 遍历所有非self rank, 逐个LocalReduce到output
- 第一个非self rank做copy(首次), 后续做reduce
  出处: ins_temp_reduce_scatter_mesh_1D.cc:74-107

通信API: SendRecvWrite(RDMA write到远端scratch)
scratch倍率: templateRankSize_ (需N个slot)
线程数: rankSize - 1 (每个peer一个线程)

与AG Mesh1D的对称差异:
- RS PostCopy做LocalReduce; AG PostCopy只做CpyAndSendRecvCopy(不reduce)
- RS通信方向: 本地input -> 远端scratch(写)
- AG通信方向: 远端input -> 本地scratch(读, 或远端写本地)

#### MeshChunk (InsTempReduceScatterMesh1DMeshChunk, 264行)

RS的流水线变体:
- 数据切分为sliceNum = templateRankSize_ - 1 份
- 每步处理一个chunk, 使用SendRecvWriteReduce(通信+reduce融合)
- scratch倍率: templateRankSize_ - 1 (比Mesh1D少1)

AR没有独立的RS MeshChunk(AR的MeshChunk在TwoShot内部)。

#### NHR (InsTempReduceScatterNHR, 267行)

RS的Non-Hierarchical Recursive算法:
- log2(rankSize)步, 每步deltaRank = 1 << step
- 三阶段: LocalDataCopy(input->scratch) -> RunNHR(步进式SendRecvWriteReduce) -> PostLocalCopy(scratch->output)
- 单线程(GetThreadNum returns 1)
- scratch倍率: 1

RS与AG NHR的关键差异:
- RS使用SendRecvWriteReduce(写+reduce融合)
- AG使用SendRecvWrite(只写)
- RS先内后外(step从小到大); AG先外后内(step从大到小)

#### DPU (InsTempReduceScatterMesh1dDpu, 280行)

RS特有(AR没有DPU模板):
- 序列化DPURunInfo -> HcommSendRequest发送到DPU -> HcommWaitResponse等待完成
- PostLocalReduce: 将self数据从input拷贝到scratch, 逐rank reduce, 最后拷贝到output
- 这是唯一使用REGISTER_TEMPLATE_V2的模板(其他都用REGISTER_EXEC_V2)

---

### AIV Templates (1种, 5文件)

只有1种: AivTempReduceScatterMesh1D(AR有2种: OneShot + TwoShot)。
RS不需要AG阶段, 所以不需要TwoShot。

#### Host侧 (aiv_temp_reduce_scatter_mesh_1D.cc, 118行)

- CalNumBlocks: stepNum(2) * tempRankSize_ (AR OneShot是rankSize+1)
- KernelRun: 填充AivOpArgs, 包含reduceOp(AR也有)
- CalcScratchMultiple: templateRankSize_

#### Device Kernel (aiv_reduce_scatter_mesh_1d.h, 112行)

Producer-Consumer核分工:
- Producer核: CpGM2GM将本rank的input切片拷贝到目标rank的IPC buffer + Record flag
- Consumer核: WaitFlag等待每个rank数据到达 -> 第一个rank做CpGM2GM(copy), 后续rank做CpGM2GM(带reduceOp的reduce)

RS vs AG的AIV kernel差异:
- RS Consumer: copy + reduce(CpGM2GM with reduceOp_) -- 需要reduce
- AG Consumer: 只copy(CpGM2GM without reduce) -- 不需要reduce

TODO注释: "todo 简化参数" (aiv_reduce_scatter_mesh_1d.h:16)

#### Copy-Paste Bug: header guard

```
// aiv_communication_v2.h:11-12 (在reduce_scatter目录下)
#ifndef ALL_GATHER_AIV_COMMUNICATION_V2_H
#define ALL_GATHER_AIV_COMMUNICATION_V2_H
```
Header guard是ALL_GATHER而非REDUCE_SCATTER, 是从all_gather目录copy时遗留的。

---

### CCU Templates (7种模板 + 7种kernel, 22文件)

与AR的7种CCU 1:1对应, 核心差异是GroupReduce替换GroupBroadcast。

#### RS vs AR CCU的根本差异

| 维度 | AllReduce CCU | ReduceScatter CCU |
|------|-------------|-------------------|
| 高阶API | GroupReduce(RS) + GroupBroadcast(AG) | GroupReduce only |
| 低阶API(mem2mem) | ReadNb + ReduceLoop + WriteNb | ReadNb + ReduceLoop only |
| 低阶API(NHR) | WriteReduceNb(RS) + WriteNb(AG) | WriteReduceNb only |
| 阶段数 | 2(RS+AG) | 1(RS only) |
| 生命周期 | Init->Load->PreSync->DoRS->DoAG->PostSync | Init->Load->PreSync->DoRS->PostSync |

#### 7种CCU Template概览

| Tag | 拓扑 | 模式 | die | 特征 |
|-----|------|------|-----|------|
| CcuReduceScatterMesh1D | mesh | MS | 1 | GroupReduce高阶API |
| CcuReduceScatterMesh2Die | mesh | MS | 2 | RmtReduce, rmtReduceWithMyRank |
| CcuReduceScatterMesh1DMem2Mem | mesh | SCHED | 1 | ReadNb+ReduceLoopGroup, 最复杂(423行) |
| CcuReduceScatterMeshMem2Mem1D2Die | mesh | SCHED | 2 | 2die变体, isReduceToOutput_ |
| CcuReduceScatterNHR1DMem2Mem | NHR | SCHED | 1 | WriteReduceNb, 步进式 |
| CcuReduceScatterNhr1DMem2MemMultiJetty | NHR | SCHED | - | 多端口并行, 最成熟 |
| CcuReduceScatterParallelMesh1DNHR | mesh+NHR | SCHED | - | 双level并行 |

#### CCU mesh1d mem2mem (最复杂的kernel, 423行)

ccu_kernel_reduce_scatter_mesh1d_mem2mem.cc 是RS最复杂的CCU kernel:

Algorithm()生命周期:
```
InitResource -> LoadArgs -> PreSync -> DoReduceScatter -> PostSync
               ↓ (或 repeat模式)
InitResource -> LoadArgs -> PreSync -> DoRepeatReduceScatter -> PostSync
```

DoReduceScatter核心流程:
1. ReadNb: 从所有peer远端读取数据到本地scratch
2. WaitEvent: 等待所有读取完成
3. ReduceLoopGroup: 将所有scratch中的数据reduce到output

ReduceLoopGroup机制(RS独有的复杂reduce流程):
- m-part(主循环) + p-part(并行剩余部分)
- GoResource(LOOP_NUM=16): 分配16个循环组的资源
- LoopGroup调度: LocalCopyNb -> LocalReduceNb -> LocalCopyNb 流水线
  出处: ccu_kernel_reduce_scatter_mesh1d_mem2mem.cc (CreateReduceLoop方法)

DoRepeatReduceScatter(repeat模式):
- CCU_WHILE循环, 处理超过单次capacity的数据
- flag_变量判断是否是第一次迭代(首次需要copy, 后续可以直接reduce到已有结果)
  出处: ccu_kernel_reduce_scatter_mesh1d_mem2mem.cc (DoRepeatReduceScatter方法)

对比AR的等价kernel:
- AR的DoReduceScatter只是一半, 后面还有DoAllGather
- RS的DoReduceScatter是完整操作, 不需要AG阶段
- 但RS的ReduceLoopGroup更复杂(AR也有, 但RS的scratch管理更精细)

#### CCU NHR1D mem2mem (282行)

NHR on CCU, 步进式:
- 每步: PreSync -> WriteReduceNb(发送并reduce) -> PostSync
- DoRepeatWriteReduceSlices: CCU_WHILE循环
- NHRStepInfo: step/myRank/nSlices/toRank/fromRank/txSliceIdxs/rxSliceIdxs

RS vs AR NHR的差异:
- RS只有WriteReduceNb阶段(RS步)
- AR有WriteReduceNb(RS步) + WriteNb(AG步)

#### CCU mesh2die (110行)

2-die变体:
- rmtReduceWithMyRank_: 控制是否本rank也参与remote reduce
- channel按dieId分组, 不同die独立处理
  出处: ccu_kernel_reduce_scatter_mesh2die.h:82

---

### RS-Specific Complexities (RS独有的复杂性)

1. PostCopy/PostLocalReduce步骤:
   RS在通信后需要将多份scratch数据reduce到output。
   AG在通信后只需要copy(或什么都不做, 直接写到output)。
   这是RS最关键的独有复杂性。

2. reduce计算的设备差异:
   - AICPU: LocalReduce函数调用(CPU侧reduce)
   - AIV: CpGM2GM with reduceOp(GM上直接reduce, 需等前序数据到达)
   - CCU: GroupReduce高阶API / LocalReduceNb低阶API / WriteReduceNb融合API

3. scratch buffer需求:
   - RS Mesh1D: scratch = rankSize份(收集所有rank的数据再reduce)
   - RS MeshChunk: scratch = rankSize-1份
   - RS NHR: scratch = 1份(原地reduce)
   AG的scratch需求通常更小(AG只是collect, 不需要临时空间做reduce)。

4. ReduceLoopGroup机制:
   CCU mem2mem的RS需要多轮循环reduce(m-part + p-part), 是CCU中最复杂的
   计算流水线。AG的等价操作只是GroupBroadcast/LocalCopyNb, 简单得多。

---

### Idioms & Conventions (惯用法)

RS-ID-1. 与AR完全一致的入口模式 -- 三重门控 + OpParam组装 + rankSize==1快速路径
  出处: reduce_scatter_op.cc:43-144

RS-ID-2. sendBuf != recvBuf强制检查 -- RS不支持in-place(AR同样)
  出处: reduce_scatter_op.cc:97-98

RS-ID-3. NOT_MATCH降级链 -- DPU -> CCU_MS -> CCU_SCHED -> AIV -> AICPU
  出处: reduce_scatter_auto_selector.cc (整体结构)

RS-ID-4. inputSliceStride = dataSize_ -- RS的input stride是完整输出大小的rankSize倍
  而非按rank切片。这是RS数据布局的核心(每rank持有全量, output只持有1/N)。
  出处: ins_v2_reduce_scatter_sole_executor.cc:147

RS-ID-5. PostCopy/PostLocalReduce模式 -- 通信完成后的reduce步骤, RS独有
  出处: ins_temp_reduce_scatter_mesh_1D.cc:74-107, ins_temp_reduce_scatter_mesh_1d_dpu.cc

RS-ID-6. CCU Algorithm()统一生命周期 -- InitResource->LoadArgs->PreSync->DoRS->PostSync
  (与AR一致但少了DoAG阶段)
  出处: 所有7个ccu_kernel_reduce_scatter_*.cc

RS-ID-7. ratio-based阈值(AICPU UBX路径) -- 以8卡为基线的二次方系数调整阈值
  出处: reduce_scatter_auto_selector.cc:264-271

---

### Architecture & Patterns (架构模式)

RS-AP-1. 与AR完全相同的三层分工: Op -> Selector -> Executor -> Template
  RS不引入任何新的架构概念。

RS-AP-2. Executor 4种Strategy与AR 1:1对应:
  - Sole(10注册) <-> AR Sole(14注册)
  - Parallel(3注册) <-> AR Parallel(3注册)
  - Concurrent(3注册) <-> AR Concurrent(3注册)
  - Sequence(1注册, DPU) <-> AR Sequence2Die(2注册, RS+AG分解)
  差异: RS的Sequence是DPU卸载, AR的Sequence2Die是RS+AG分解。

RS-AP-3. RS/AG反向对称在executor层体现:
  RS Parallel的OrchestrateLoop:
  - Stage1: intra0(mesh RS先内) + inter1(NHR RS)
  - Stage2: inter0(NHR RS) + intra1(mesh RS)
  AG Parallel的OrchestrateLoop(根据hcomm分析):
  - Stage1: inter0(NHR AG先外) + intra1(mesh AG)
  - Stage2: intra0(mesh AG) + inter0(NHR AG)
  方向完全相反。

RS-AP-4. RS比AR多出的独有模板: DPU (InsTempReduceScatterMesh1dDpu)
  这是因为DPU场景下RS和AG需要分开独立执行(不像AR可以在一个template内完成)。

RS-AP-5. CCU GroupReduce vs GroupBroadcast:
  RS使用GroupReduce(拉取远端数据并本地reduce), AR使用GroupReduce+GroupBroadcast(RS+AG)。
  在CCU的非mem2mem模式下, RS只需要一次GroupReduce调用即可完成整个操作。

---

### Domain Knowledge (业务知识)

RS-DK-1. RS输入输出关系:
  - input: count * rankSize 个元素 (每rank持有全量)
  - output: count 个元素 (每rank持有1/N的reduce结果)
  - inputSliceStride = count * dataTypeSize (每个rank的切片在input中的间距)
  对比AG: input是count, output是count*rankSize (RS的反操作)

RS-DK-2. RS算法选择中无数据量上限限制:
  AllReduce的CCU Schedule有16MB上限(超过降级AICPU), RS没有这个限制。
  原因可能是RS的CCU mem2mem模式已支持repeat循环处理大数据。

RS-DK-3. RS的CLOS拓扑支持:
  RS AICPU的SelectMeshAlgoAicpu支持Level0Shape::CLOS(直接用NHR),
  AR没有明确的CLOS路径。

RS-DK-4. DPU路径(RS独有):
  在DPU部署模式下, RS和AG必须分开执行(因为RS在NPU上做, AG通过DPU卸载)。
  AR没有DPU路径, 因为AR = RS + AG的分解在Sequence2Die中已完成。

RS-DK-5. 64位数据类型的幽灵路径:
  RS selector对64位类型(INT64/UINT64/FP64)选择 "InsReduceScatterAicpuReduce",
  但此算法名从未注册到任何executor。这意味着64位数据类型在RS的新路径上
  会导致executor查找失败。

---

### Anti-Patterns (反模式)

#### 确认bug

AP-RS-1. 幽灵算法名 InsReduceScatterAicpuReduce (严重)
  selector在lines 232和260选择此算法, 但全仓库没有任何executor注册此名称。
  64位数据类型(INT64/UINT64/FP64)在MESH_1D和UBX全连接拓扑下会触发此路径,
  导致运行时executor查找失败。
  出处: reduce_scatter_auto_selector.cc:232, :260
  验证: `grep -r "InsReduceScatterAicpuReduce" src/ops/reduce_scatter/executor/` 返回空

AP-RS-2. sendBuf空检查提示错误 (低)
  sendBuf为空时tips说"please check recvBuf", 应为"please check sendBuf"。
  从recvBuf检查copy-paste过来, 只改了parameter名但没改tips。
  出处: reduce_scatter_op.cc:92

AP-RS-3. Parallel executor copy-paste错误消息 (低)
  InsReduceScatterParallelExecutor中的错误消息写成"InsV2AllGatherParallelExecutor"和
  "All Gather excutor kernel run failed"。从all_gather的parallel executor直接copy过来。
  出处: ins_reduce_scatter_parallel_executor.cc:310

AP-RS-4. header guard三重T拼写错误
  "HCCLV2_REDUCESCATTTER_AUTO_SELECTOR" -- SCATTER多了一个T。
  出处: reduce_scatter_auto_selector.h:11-12

AP-RS-5. "excutor"拼写错误 (executor少了e)
  出现在4个executor文件的错误消息中:
  - ins_v2_reduce_scatter_sole_executor.cc:78 "Reduce scatter excutor kernel run failed"
  - ins_reduce_scatter_parallel_executor.cc:310 "All Gather excutor kernel run failed"
  - ins_reduce_scatter_concurrent_executor.cc (同样模式)
  - ins_v2_reduce_scatter_sequence_executor.cc (同样模式)

AP-RS-6. AIV aiv_communication_v2.h header guard写成 ALL_GATHER
  ```
  #ifndef ALL_GATHER_AIV_COMMUNICATION_V2_H
  #define ALL_GATHER_AIV_COMMUNICATION_V2_H
  ```
  应为 REDUCE_SCATTER_AIV_COMMUNICATION_V2_H。从all_gather目录copy时遗留。
  出处: template/aiv/kernel/aiv_communication_v2.h:11-12

AP-RS-7. namespace注释不一致
  reduce_scatter_auto_selector.h 结尾: `} // namespace Hccl` (大写H)
  其他文件结尾: `} // namespace ops_hccl`
  出处: reduce_scatter_auto_selector.h:41

AP-RS-8. Parallel executor GetParallelDataSplit硬编码50:50
  注释"to do 先做等分，后续根据性能做调整", 与AR完全相同的TODO。
  出处: ins_reduce_scatter_parallel_executor.cc:224-229

AP-RS-9. Concurrent executor portNum = 4 硬编码
  const u64 portNum = 4; 没有从拓扑信息中获取。
  出处: ins_reduce_scatter_concurrent_executor.cc:213

#### copy-paste痕迹(非bug, 但反映代码演化模式)

CP-RS-1. RS从AG copy而来的证据:
  - Parallel executor错误消息写成AllGather (AP-RS-3)
  - AIV header guard写成ALL_GATHER (AP-RS-6)
  - Sole executor注册在文件末尾, 与AR的注册顺序完全一致

CP-RS-2. RS的selector比AR更复杂:
  RS selector (353行) > AR selector (386行但AR有22种算法 vs RS约20种)。
  RS的SelectMeshAlgoAicpu有复杂的ratio-based分支, AR没有。
  但RS缺少CCU Schedule的数据量上限检查(AR有16MB限制)。

CP-RS-3. CCU模板日志类名:
  AR的CCU模板日志全写成ReduceScatter类名(AP-AR-10), 说明AR是从RS copy出来的。
  RS的CCU模板日志是正确的(因为RS是原始版本)。
  这证实了代码演化方向: RS(原始) -> AR(copy from RS)。

---

### Notable Patterns (值得注意的模式)

1. RS是模板代码的发源地(非AR):
   AR的CCU模板日志全写成ReduceScatter类名(AP-AR-10), 证明AR是从RS copy创建的。
   这与直觉相反 -- 通常认为AllReduce是最核心的算子应该最先实现,
   但证据表明RS的CCU模板代码先于AR存在。
   可能的原因: AllReduce = RS + AG, 所以先实现RS和AG, 再组合成AR。

2. RS与AG的完美反向对称:
   在hcomm algorithm层已确认RS与AG完美对称(Phase 2.3.2)。
   在hccl算子层同样如此: executor结构1:1对应, 模板数量接近(RS 49 vs AG 46),
   CCU kernel操作对称(GroupReduce vs GroupCopy, WriteReduceNb vs WriteNb)。
   额外的3个文件差异来自: RS的DPU模板(AG有对应的DPU但可能在其他位置)。

3. 幽灵算法是严重bug:
   InsReduceScatterAicpuReduce从未注册, 意味着64位数据类型的RS在
   MESH_1D和UBX全连接拓扑下会运行时失败。这是一个功能缺失而非
   简单的typo -- 需要实现一个支持64位reduce的AICPU模板并注册。

4. 确定性变体缺失:
   grep "deterministic\|Deterministic\|DETERMINISTIC" 在整个reduce_scatter/目录返回0结果。
   hcomm algorithm层有确定性变体(Phase 2.3.2), 但hccl的RS算子没有对应实现。
   这可能是一个计划中但尚未完成的功能。

5. 数据量阈值的简化:
   RS selector比AR selector对数据量阈值的处理更简单:
   - AR: 8MB/32MB/16MB等多档明确阈值
   - RS: 主要依赖IsSmallData()/LARGE_COUNT_1024KB, 无CCU数据量上限
   这可能是因为RS的CCU mem2mem模式的repeat循环已经能处理任意大小的数据。

---

## 3.2.2 send/ + recv/ -- 点对点通信

### Module Overview

send/和recv/是hccl最简的算子对, 各6个文件(~460行), 实现RDMA语义的点对点通信。
两者精确对称, 形成Producer-Consumer握手协议。
仅在910_95设备 + OP_BASE模式下走新路径, 其他设备fallback到HcclSendInner/HcclRecvInner。

### Key Files

| 文件 | 行数 | 职责 |
|------|------|------|
| send_op.h/cc | 37+151 | C ABI入口, 三重fallback guard, 参数校验, 委托Selector+HcclExecOp |
| send_auto_selector.h/cc | 25+26 | 无条件返回"InsSend" (零分支) |
| ins_send_executor.h/cc | 53+167 | 资源计算, OFFLOAD/OPBASE双模式数据切片+RDMA写 |
| recv_op.h/cc | 37+153 | 与send镜像: inputPtr=nullptr, outputPtr=recvBuf |
| recv_auto_selector.h/cc | 25+26 | 无条件返回"InsRecv" |
| ins_recv_executor.h/cc | 54+178 | 与send对称: RecvWrite协议(Record ACK + Wait DATA_SIGNAL) |

### 与all_reduce的结构对比

| 维度 | all_reduce | send/recv |
|------|-----------|-----------|
| 文件数 | 55 | 6 |
| 算法数 | 22 | 1 |
| Executor策略 | 4种 | 1种 |
| Template层 | aicpu+aiv+ccu | 无 |
| Selector复杂度 | 349行, 多维分支 | 26行, 零分支 |
| 拓扑层级 | 多层(Level0/1/2) | 单层(强制resize(1)) |
| 资源需求 | 多线程+多notify+scratch | 0 notify, 0 slave thread |

本质: P2P只有一对(src,dst), 不涉及reduce计算, 不需要多步拓扑编排。

### Architecture: Producer-Consumer协议

send/recv通过alg_data_trans_wrapper.cc中的SendWrite/RecvWrite实现两阶段握手:
```
SendWrite: WaitAck -> HcommWrite(逐片) -> RecordDataSignal
RecvWrite: RecordAck -> WaitDataSignal
```
Recv先发ACK(buffer ready) -> Send收到后写数据+发DATA_SIGNAL -> Recv等DATA_SIGNAL后使用数据。

PUT DMA模式: 数据由发送端主动推到接收端内存(RDMA Write语义)。
recv端构建的srcSlices是无效占位值(注释明确说明), 因PUT模式下此地址无用。

### OFFLOAD vs OPBASE双模式

- OFFLOAD(图模式): send写到对端output buffer(直接落地), 一次性切片+一次SendWrite。
  传输上限仅受UB_MAX_DATA_SIZE(256MB)约束。
- OPBASE(单算子): send写到对端ccl buffer(中转), 每片独立SendWrite;
  recv端每片RecvWrite后紧跟LocalCopy从ccl buffer拷到output。
  传输上限取min(UB_MAX_DATA_SIZE, cclBufferSize)。

多了一次拷贝的原因: 单算子模式下无法提前获取对端output buffer地址, 只能通过ccl buffer中转。

### Idioms

- 三重fallback guard: CheckHCCLIndependentOp -> DevType -> WorkflowMode, 渐进迁移模式
- Tag命名: "Send_{commName}_{srcRank}_{dstRank}" / "Recv_{commName}_{srcRank}_{dstRank}"
- 日志方向: send用`[%d]->[%d]`, recv用`[%d]<-[%d]`
- RPT_INPUT_ERR + CHK_PTR_NULL双重空指针检查
- DATATYPE_SIZE_TABLE[dataType]直接索引(依赖上层CheckDataType保证合法性)
- std::find_if查找channel(线性搜索, channel列表规模小)

### Anti-Patterns

AP-SR-1. 参数拼写错误: alg_data_trans_wrapper.h中7处函数参数名写成`snedInfo`而非`sendInfo`。
AP-SR-2. 冗余系统调用: HcclGetRankSize和HcclGetCommName在HcclSend/SendExec中各调用两次。
AP-SR-3. extern "C" LaunchAicpuKernel声明但未使用(send_op.cc:35, recv_op.cc:35), 早期架构遗留。
AP-SR-4. recv端srcBufferPtr是无效占位值, 增加维护负担和误解风险。
AP-SR-5. OFFLOAD/OPBASE双分支数据切片代码高度重复(差异仅在目标buffer和同步方式)。
AP-SR-6. sprintf_s错误日志缺少tag内容和返回值, 不利于问题定位。
AP-SR-7. using namespace std在.cc文件顶部。

---

## 3.2.3 all_gather/ + reduce_scatter/ -- RS/AG对称半边

### all_gather/ (46文件)

all_gather是AllReduce=RS+AG分解的AG半边。仅做数据收集(每rank发自己的数据给所有其他rank), 不涉及reduce操作。

Executor 4种策略:
- Sole: 单级拓扑, 15个注册(4 AICPU + 10 CCU + 1 AIV)
- Parallel: 双级并行(intra+inter), 4注册(2 AICPU + 2 CCU), data 50:50等分
- Concurrent: UBX(CLOS)机型, mesh和NHR在同级同时执行, 3注册, 按端口比例分数据
- Sequence: DPU场景(inter先做再intra), 1注册(仅AICPU)

Selector: 22个算法名, 5路分派(AICPU/CCU_MS/CCU_SCHED/AIV/DPU)。

Template:
- AICPU(3): Mesh1D/NHR/NHR_DPU(比AR少TwoShot/MeshChunk, 因AG不需要reduce步)
- AIV(1): Mesh1D(比AR少TwoShot)
- CCU(6): mesh1D/mesh1D_mem2mem/2dies_mesh1D/2dies_mesh_mem2mem/nhr_1D_mem2mem/nhr_1D_multi_jetty_mem2mem

CCU特点: CalcScratchMultiple返回0(不需要scratch buffer), AG直接通过硬件通信引擎在内存间传输。
AICPU对比: CalcScratchMultiple返回rankSize(需要scratch做中转)。

### all_gather确认Bug

AG-BUG-1. Selector命名不匹配(4处): selector输出的算法名与executor注册名不一致:
  - CcuAllGatherParallelMeshNHR vs CcuAllGatherParallelMesh1DNHR (缺"1D")
  - CcuAllGatherMesh2Die vs CcuKernelAllGather2DiesMesh1D (完全不同格式)
  - CcuAllGatherMeshMem2Mem1D vs CcuAllGatherMesh1DMem2Mem ("1D"和"Mem2Mem"顺序互换)
  - CcuAllGatherMesh2DMem2Mem 无对应executor注册

AG-BUG-2. SelectCcuMsAlgo缺少return(all_gather_auto_selector.cc:24-26):
  topoLevelNums>1时设置名称后未return, 直接落入SelectMeshAlgo覆写名称。

AG-BUG-3. typo "levle0Algo" 出现9处(应为level0Algo)。

AG-BUG-4. "excutor"拼写错误(ins_v2_all_gather_sole_executor.cc:83, parallel_executor:305)。

AG-BUG-5. Header guard copy-paste: ccu_temp_all_gather_mesh_1D.h的guard结尾含MEM2MEM(从mem2mem版copy时遗留)。

AG-BUG-6. AIV numBlocks=4硬编码(注释掉了动态获取)。

AG-BUG-7. CalcRes WARNING "Resource calculation is temporarily not performed"(2处AICPU/AIV template)。

AG-BUG-8. NHRStepInfo在namespace外部定义(全局命名空间污染)。

AG-BUG-9. SIZE_TABLE与DATATYPE_SIZE_TABLE混用(同一算子concurrent_executor.cc内同时使用两种)。

### reduce_scatter/ (49文件)

reduce_scatter是AllReduce=RS+AG分解的RS半边, 与all_gather完美反向对称。
RS先内后外(先组内reduce再跨组scatter), AG先外后内。

RS独有复杂度: PostCopy/PostLocalReduce步骤(AG只需copy, RS需reduce);
CCU的ReduceLoopGroup机制(m-part+p-part流水线); scratch buffer需求更大(Mesh1D需rankSize份)。

CCU kernel操作对称: RS用GroupReduce/WriteReduceNb/LocalReduceNb(含reduce),
AG用GroupCopy/WriteNb/LocalCopyNb(仅copy)。

确定性变体: 不存在。grep在整个reduce_scatter/目录返回0匹配。
(hcomm algorithm层有确定性变体, 但hccl的RS算子未实现)

### reduce_scatter确认Bug

RS-BUG-1. 幽灵算法(严重): InsReduceScatterAicpuReduce在selector的lines 232和260被选择,
  但全仓库没有任何executor注册此名称。64位数据类型(INT64/UINT64/FP64)在MESH_1D和
  UBX全连接拓扑下会触发此路径导致运行时失败。

RS-BUG-2. FP64重复/INT64遗漏: selector中检查FP64两次、遗漏INT64(3个文件都有此bug)。

RS-BUG-3-9. (其他bug同all_reduce分析中已记录的copy-paste和命名问题)

---

## 3.2.4 batch_send_recv/ + broadcast/ + scatter/

### batch_send_recv/ (6文件)

batch_send_recv将多个send/recv任务打包执行, 核心在死锁预防和流水线调度。

Pair-wise不对称排序算法防止死锁:
- Send排序: remoteRank<=myRank的加rankSize, 按降序(先发给"环形左边")
- Recv排序: remoteRank<myRank的加rankSize(注意<而非<=), 按升序(先从"环形右边"收)
- 不对称确保: rank A发给rank B时, rank B恰好在从rank A收

与单独send/recv的差异:
- Channel数: 每remote 2条(vs 1条), 一条send一条recv
- 通信模式: Read语义(接收端拉, 利于流水线) vs Write语义(发送端推)
- 线程模型: 主+1从(vs 仅主线程)
- 自发自收: remoteRank==myRank通过LocalCopy直接DMA, 不走网络

反模式:
AP-BSR-1. 变量名"Silces"(应为Slices), 出现11次(h+cc)。
AP-BSR-2. 使用malloc+placement new而非RAII(CHK_RET提前返回泄漏路径)。

### broadcast/ (28文件)

Broadcast = Scatter + AllGather 二阶段分解:
- TwoShot: root将数据N等分scatter, 然后allgather交换
- NHR: PreCopy -> NHR scatter(log2步) -> NHR allgather(log2步) -> PostCopy
- CCU Mem2Mem: CCU_WHILE循环中交替执行scatter和allgather

Executor 2种: Sole(单级, 6注册) + Parallel(双级, 数据50:50等分, TODO标记后续调优)。

Selector: AICPU(4算法) + CCU_MS(1) + CCU_SCHED(3) + AIV(1), 无DPU。

反模式:
AP-BC-1. broadcast_op.h include guard写成REDUCE_SCATTER_OP(copy-paste)。
AP-BC-2. Sole executor错误消息说"Reduce scatter excutor kernel run failed"(三重错误: 算子名+拼写+)。
AP-BC-3. NHR BatchSR中txDstSlice被push到txSrcSlices(ins_temp_broadcast_nhr.cc:378-381), 疑似bug。
AP-BC-4. TwoShot RootSendData中sendDstSlice1的size参数误用offset(ins_temp_broadcast_mesh_1D_two_shot.cc:173)。
AP-BC-5. ParallelExecutor中interlocalroot/intralocalroot成员未使用(debug日志打印始终为0)。
AP-BC-6. CalcRes WARNING "Resource calculation is temporarily not performed"(3处)。
AP-BC-7. GetParallelDataSplit硬编码50:50, TODO标记未完成。

### scatter/ (46文件, V1架构)

scatter是全仓库唯一的V1->V2过渡中间态算子:
- algo/(V1): 4个executor(comm/mesh/ring/single) + 6个template(mesh/ring/ring_direct/nb/nhr/nhr_base)
  使用ExecutorBase(V1), 直接继承, 无Registry
- executor/selector/template/(V2): 标准V2架构, 通过910_95设备类型分流
- 两套架构通过scatter_op.cc中的设备类型判断共存

V1 algo/特有结构:
- ScatterExecutorBase(algo/scatter_executor_base.h): 基类, 定义RunTemplate虚方法
- ScatterCommExecutor: 主executor, 按算法名(ring/mesh/nhr等)委托给具体template
  V2中这个角色由Registry+Selector承担
- PrepareSlicesData有4份完全相同的实现(在4个executor中各copy一份)

scatter高修改频率的根因: V1/V2双轨维护成本 x 多变体组合(4 V1 executor x 6 V1 template + V2)。

反模式:
AP-SC-1. PrepareSlicesData 4份重复代码(应提取到基类)。
AP-SC-2. 日志copy-paste: ScatterMeshExecutor/ScatterCommExecutor都写成"ScatterRingExecutor"。
AP-SC-3. ScatterSingleExecutor错误消息写着"all reduce executor"。
AP-SC-4. scatter_op.cc中"faled"拼写错误(应为"failed")。
AP-SC-5. V1架构是技术债务, 其他所有算子已迁移V2, 仅scatter保留V1。

---

## 3.2.5 all_to_all_v/ + all_gather_v/ + reduce_scatter_v/ + reduce/

### all_to_all_v/ (50文件, 最大算子目录)

all_to_all_v包含三个API: HcclAlltoAll(等长) + HcclAlltoAllV(变长+位移) + HcclAlltoAllVC(count matrix)。
三个API底层统一转换为AlltoAllV参数格式执行。

O(N^2)通信模式: N个rank中每一个都要向其他N-1个rank发送不同量的数据。
这与集合操作(AllGather/ReduceScatter)有根本区别:
- 需要per-rank的send/recv处理(不是统一的拓扑编排)
- 参数需要4个N维数组(sendCounts/recvCounts/sdispls/rdispls)
- CCU层需要区分AlltoAll和AlltoAllV两套template

3组独立Selector: 分别注册HcclCMDType::ALLTOALL/ALLTOALLV/ALLTOALLVC。

Executor: Sole(新旧两版) + Concurrent(AlltoAll和AlltoAllV各一个)。
旧版SoleExecutor有冗余的A2ASendRecvInfo(8个vector, count和byte-size双重表示)。

Template:
- AICPU(1): InsTempAlltoAllVMesh1D, per-pair三分支处理(send+recv/仅send/仅recv/skip)
- AIV(2): alltoall mesh1D + alltoallv mesh1D
- CCU(6): alltoall/alltoallv x (mesh1D/mesh2die/multi_jetty)

反模式:
AP-ATV-1. 双重return dead code(ins_all_to_all_v_sole_executor.cc:154-155)。
AP-ATV-2. malloc+free内存泄漏路径(all_to_all_v_op.cc:333-414, 多个CHK_RET提前返回不free)。
AP-ATV-3. SIZE_TABLE(旧)和DATATYPE_SIZE_TABLE(新)同文件混用。
AP-ATV-4. "Setlect"拼写错误(应为Select), 出现在3个selector日志中。

### all_gather_v/ (8文件) + reduce_scatter_v/ (8文件)

V变体是固定长度算子的"功能补丁", 极简结构: op + selector + 1个sole executor + 1个AICPU template。
CCU/AIV template不在自己目录下, 复用其他目录实现(selector声明了更多算法名但executor在其他位置)。

all_gather_v: recv端变长(recvCounts[]+recvDispls[]), varData编码2*rankSize个u64。
reduce_scatter_v: send端变长(sendCounts[]+sendDispls[]), 与AG-V形成对偶。

变长复杂度核心: 固定长度只需一个processedDataCount, 变长需要per-rank tracking allRankProcessedDataCount[i]。

反模式:
AP-AGV-1. all_gather_v_auto_selector.h include guard写成HCCLV2_REDUCESCATTTER_AUTO_SELECTOR
  (从reduce_scatter复制, 还带三个T的拼写错误)。
AP-AGV-2. all_gather_v_sole_executor.h include guard写成REDUCE_SCATTER_SOLE_EXECUTOR_H。
AP-RSV-1. malloc+free泄漏路径(与all_to_all_v相同问题)。

### reduce/ (30文件)

Reduce = ReduceScatter + Gather-to-root。reduce将所有rank数据reduce后结果集中在root rank。
与reduce_scatter的差异: reduce有root参数, template有SetRoot()方法。

三种Executor:
- Sole: 单级, 6注册(AICPU 2 + CCU 3 + AIV 1)
- Parallel: 双级并行(intra Mesh + inter NHR), data 50:50, 包含CalcLocalRoot逻辑
- Sequence: 双级串行AICPU only(先框内后框间)

Template:
- AICPU(2): ReduceMesh1D(三步GatherData->ReduceData->SendData) + ReduceNHR(log2步)
- AIV(1): AivTempReduceMesh1D
- CCU(3): mesh1D/mesh1D_mem2mem/nhr1d_mem2mem, NHR含die拆分逻辑(SplitDataFor2Dies)

反模式:
AP-RD-1. Selector SelectMeshAlgoAicpu逻辑死代码(reduce_auto_selector.cc:165-181):
  两个if分支检查同一条件(level0Topo==MESH_1D), 第二个永远不执行。
AP-RD-2. CCU template日志写成"CcuTempBroadcastMesh1DMem2Mem"(应为Reduce)。

---

## 3.2.6 算子层综合分析

### 全量Grep量化

| 模式 | 总计 | 分布特征 |
|------|------|----------|
| CHK_RET | 1197次 | scatter最密(237), P2P最少(21-22) |
| REGISTER_EXEC_V2 | 64次 | AR/AG各13(最多), V变体各1 |
| reinterpret_cast | 128次 | all_to_all_v最多(27), send/recv零 |
| HCCL_ERROR | 461次 | AR最多(75), send/recv各4 |
| "excutor"拼写 | 22次 | 9/12算子都有, 系统性错误 |
| TODO/FIXME/HACK | 0次 | 整个ops/目录零技术债标记 |
| Header guard不匹配 | ~76处 | reduce_scatter是最常被复制的源 |

### 跨算子惯用法 (OP-ID-1~15)

OP-ID-1. 三重fallback guard: 所有新算子入口(op.cc)都有CheckHCCLIndependentOp/DevType/WorkflowMode三重检查, 仅910_95+OPBASE走新路径。
OP-ID-2. CHK_RET 100%覆盖: 1197次/全部12个算子。
OP-ID-3. RPT_INPUT_ERR+CHK_PTR_NULL双重空指针检查(所有op.cc)。
OP-ID-4. Tag命名规范: "{OpName}_{commName}_{rank信息}"。
OP-ID-5. Selector优先级统一18: 所有14个REGISTER_SELECTOR_BY_OPTYPE都用18。
OP-ID-6. 5路执行模式分派: DPU -> CCU_MS -> CCU_SCHED -> AIV -> AICPU(兜底), 全部算子一致。
OP-ID-7. Executor C++模板参数注入: `<TopoMatch, Template0[, Template1]>`实现静态多态。
OP-ID-8. CalcScratchMultiple差异化: reduce类返回rankSize(需scratch), scatter/alltoall返回0-1。
OP-ID-9. OFFLOAD/OPBASE双模式: P2P算子在executor中区分图模式(直接写output)和单算子(经ccl buffer中转)。
OP-ID-10. Entry/Exit日志对: "[OpName] Start." / "[OpName] End. ret=[%d]"。
OP-ID-11. SingleRankProc快速路径: rankSize==1时跳过通信。
OP-ID-12. varData柔性数组: V变体通过OpParam末尾的varData传递per-rank count/displ。
OP-ID-13. CCU_WHILE repeat循环: CCU mem2mem模板统一使用repeatNum循环处理任意大小数据。
OP-ID-14. DMAReduceFlag优化: reduce相关操作根据链路能力选择Inline/TBE/CPU路径。
OP-ID-15. 二阶段分解模式: Broadcast=Scatter+AllGather, AllReduce=RS+AG。

### 架构模式 (OP-AP-1~8)

OP-AP-1. 四子模块分工统一: op.cc(入口)->selector(选算法)->executor(编排)->template(设备相关实现)。
  例外: P2P算子(send/recv/batch_send_recv)无template层; scatter保留V1 algo/架构。
OP-AP-2. 4种Executor策略矩阵: Sole(单级)/Parallel(双级并行)/Concurrent(CLOS同级)/Sequence(DPU串行)。
  并非所有算子都实现全部4种: AR/AG/RS全4种, broadcast 2种, reduce 3种, P2P仅Sole。
OP-AP-3. RS/AG完美反向对称: 1:1对应的executor/template/kernel, CCU操作对称(GroupReduce vs GroupCopy)。
OP-AP-4. AllReduce=RS+AG组合: 三层体现(Executor Sequence2Die / Template TwoShot / Kernel DoRS+DoAG)。
OP-AP-5. O(N^2) AlltoAll独特性: 跳过CollCommExecutor范式, per-pair三分支处理, 三个API合一。
OP-AP-6. V变体极简模式: 8文件最小集合(仅AICPU), CCU/AIV复用固定长度算子的template。
OP-AP-7. scatter V1->V2过渡态: 唯一同时拥有algo/(V1)和executor/selector/template/(V2)的算子。
OP-AP-8. reduce_scatter是代码发源地: AR的CCU模板日志全写成RS类名, 证明代码从RS copy到AR/AG/BC/SC。

### 业务知识 (OP-DK-1~6)

OP-DK-1. 算子分类:
  - 集合操作: all_reduce/all_gather/reduce_scatter/broadcast/reduce/scatter
  - 点对点: send/recv/batch_send_recv
  - 变长变体: all_to_all_v/all_gather_v/reduce_scatter_v
OP-DK-2. 操作分解关系: AllReduce=RS+AG, Broadcast=Scatter+AllGather, Reduce=RS+Gather-to-root。
OP-DK-3. 通信模式: 集合操作用Write语义(发送端推), batch_send_recv用Read语义(接收端拉)。
OP-DK-4. P2P握手协议: RecordAck->WaitAck->HcommWrite->RecordDataSignal->WaitDataSignal。
OP-DK-5. 数据量分档影响算法: 小数据优先简单算法, 大数据用分步/流水线算法。
OP-DK-6. batch_send_recv死锁预防: pair-wise不对称排序(Send降序+Recv升序, <=与<不对称)。

### 硬件抽象 (OP-HW-1~4)

OP-HW-1. 三种设备编程模型根本不同:
  - AICPU: 多线程, SlaveThread并行, HcommRead/Write指令
  - AIV: Producer-Consumer核, GM flag busy-poll, 设备侧直接执行
  - CCU: 编译器式DSL, Host构造微码IR, CCU_IF/CCU_WHILE硬件条件执行
OP-HW-2. UB_MAX_DATA_SIZE双定义:
  - Host端 256MB(alg_param.h): RDMA引擎单次传输上限
  - Device端 190KB(aiv_communication_base_v2.h): AIV物理UB SRAM大小
OP-HW-3. 910_95设备门控: 所有新路径仅对910_95启用, 其他设备fallback旧路径。
OP-HW-4. CCU ms vs schedule: ms模式用GroupBroadcast/GroupReduce一步完成,
  schedule模式用mem2mem+repeat循环支持任意数据量。

### 反模式汇总 (按严重程度)

严重(功能Bug):
1. 幽灵算法: RS/AG/Broadcast selector输出的算法名在executor中无注册(运行时失败)。
   RS: InsReduceScatterAicpuReduce(64位数据类型); AG: 4个CCU名称不匹配。
2. FP64重复/INT64遗漏: 3+个selector中检查FP64两次, 遗漏INT64(可绕过类型检查)。
3. Selector缺少return: AG SelectCcuMsAlgo设置名称后未return, 被后续代码覆写。
4. Reduce selector死代码: 两个if检查同一条件, 第二个永远不执行。
5. Broadcast NHR txDstSlice push到错误容器(ins_temp_broadcast_nhr.cc:378-381)。
6. Broadcast TwoShot size参数误用offset(ins_temp_broadcast_mesh_1D_two_shot.cc:173)。

中等(代码质量):
7. Copy-paste日志错误: 至少15处算子名引用错误(RS->AR, AR->AG, RS->BC, BC->RD等)。
8. "excutor"系统性拼写: 22处/9个算子。
9. Header guard不匹配: ~76处, 3个算子的aiv_communication_v2.h共享同一guard(编译陷阱)。
10. "levle0Algo"拼写: 8+个selector文件/30+处。
11. "Silces"/"Scatchte"/"Setlect"/"faled"等拼写错误。
12. malloc+free泄漏路径: all_to_all_v/all_gather_v/reduce_scatter_v的op.cc中CHK_RET提前返回不free。
13. SIZE_TABLE与DATATYPE_SIZE_TABLE混用(接口不统一, 同文件内两种共存)。

低等(技术债):
14. scatter V1架构: PrepareSlicesData 4份重复代码。
15. CalcRes WARNING "Resource calculation temporarily not performed"(多处)。
16. AIV numBlocks硬编码。
17. GetParallelDataSplit硬编码50:50(TODO未完成)。
18. recv端srcSlices无效占位值(接口设计不合理)。

### 算子规模总表

| 算子 | 文件数 | CHK_RET | EXEC注册 | 执行模式 | 硬件后端 |
|------|--------|---------|----------|----------|----------|
| all_reduce | 55 | 197 | 13+8 | AICPU/CCU_MS/CCU_SCHED/AIV/DPU | aicpu+aiv+ccu |
| all_to_all_v | 50 | 161 | 11 | AICPU/CCU_SCHED/AIV | aicpu+aiv+ccu |
| reduce_scatter | 49 | 164 | 10 | AICPU/CCU_MS/CCU_SCHED/AIV | aicpu+aiv+ccu |
| scatter | 46 | 237 | 5(V2)+4(V1) | AICPU/AIV + V1 | aicpu+aiv(V2)+V1 |
| all_gather | 46 | 133 | 13 | AICPU/CCU_MS/CCU_SCHED/AIV/DPU | aicpu+aiv+ccu |
| reduce | 30 | 86 | 5 | AICPU/CCU_MS/CCU_SCHED/AIV | aicpu+aiv+ccu |
| broadcast | 28 | 90 | 6 | AICPU/CCU_MS/CCU_SCHED/AIV | aicpu+aiv+ccu |
| all_gather_v | 8 | 29 | 1 | AICPU(仅) | aicpu |
| reduce_scatter_v | 8 | 26 | 1 | AICPU(仅) | aicpu |
| send | 6 | 21 | 1 | AICPU(仅) | 无template |
| recv | 6 | 22 | 1 | AICPU(仅) | 无template |
| batch_send_recv | 6 | 31 | 1 | AICPU(仅) | 无template |
