# Phase 11 Part A: 场景化开发指南 (场景 11.1 / 11.2 / 11.3)

Date: 2026-03-16
Phase: 11 (场景化开发指南, Part A: 前3个核心场景)

---

## 11.1 场景: 给hccl新增一个集合通信算子

### 场景简述

你需要在hccl仓库中新增一个全新的集合通信算子(如ReduceScatter的DPU版本、一个新的Scatter变体)，从Op入口定义到Selector分派、Executor编排、hcomm算法调用，完成全链路搭建。

这是hccl/hcomm中最复杂的开发任务之一。一个中等规模的新算子涉及25-37个文件、1000-1500行新增代码(Case FT-4: 27 files, +1535行)，大规模的架构变更可达220个文件(Case FT-1)。

### 前置知识

- Phase 3(hccl算子层): `phase3-hccl-operators.md` — 14个算子目录的统一模式、Selector/Executor/Template三件套
- Phase 5.4(AllReduce完整路径): `phase5-slice-allreduce.md` — API→Op→Selector(5路)→Executor→hcomm算法的9层调用栈
- Phase 3(执行器双代体系): V1 ExecutorBase(仅scatter使用) vs V2 InsCollAlgBase(28个子类, 主力)
- Phase 3(ExecMem): 执行器间共享的内存/参数上下文结构体
- Phase 3(Selector 5路分派): CCU MS / CCU Schedule / AICPU+AIV / AICPU / Legacy

### 决策清单

#### 步骤1: 确定执行模式——Device-side还是DPU Host-side?

在写任何代码之前，先判断新算子走哪条执行路径。hccl存在两种本质不同的执行模型:

- Device-side(主流): 算法逻辑在AICPU/AIV/CCU上执行，通过device侧HCCS/RoCE网卡通信。路径是Op→Selector→Executor→hcomm三层。
- Host DPU(专用): 算法逻辑在Host CPU上执行，通过Host网卡通信。路径是Op→Executor→Template两层，跳过hcomm。

判断标准: 如果目标硬件是DPU网卡且算法模型与device-side差异太大，走DPU路径。Case FT-4(add host DPU support in reducescatter, hccl:3e916555)展示了完整的DPU新算子搭建——新建独立的`src/ops/reduce_scatter/`目录，包含reduce_scatter_op.cc(683行)和DPU专用Executor/Template。FT-4选择独立目录而非在device-side代码中添加分支，正是因为DPU的执行模型与device-side本质不同(Phase 3分析: Op→Executor→Template两层 vs 三层路径)。

如果走Device-side路径(大多数情况)，继续步骤2-7。

#### 步骤2: Op定义——参照all_reduce_op.cc的三重门控+OpParam构建

新Op的入口文件(xxx_op.cc)需要遵循标准模式(Phase 5.4 AllReduce分析):

1. 三重门控:
   - `CheckHCCLIndependentOp()` — 是否走独立算子模式
   - `DevType != DEV_TYPE_910_95` — 是否A5芯片(走新路径)
   - `GetWorkflowMode() != OP_BASE` — 是否图模式(走旧路径fallback)
2. 前置校验: `InitEnvConfig()` → 参数校验(非空检查、数据类型检查) → `HcclGetRankSize/RankId` → `CheckTag/CheckUserRank`
3. OpParam构建: 组装inputPtr/outputPtr/DataDes/opType/reduceType/stream/commName等字段
4. 单卡快速路径: `rankSize==1 → SingleRankProc → return`
5. 正常路径: `Selector(comm, param, topoInfo, algName) → HcclExecOp(comm, param, topoInfo, algName)`

Case FT-4展示了从零搭建Op的完整结构: reduce_scatter_op.cc共683行，涵盖参数校验→拓扑信息获取→算法选择→资源分配→执行全流程。其中独有的`CheckHostDPUOnly()`函数通过检测最高level网络的链路类型来判断纯DPU场景——这体现了Op层需要的拓扑感知能力。

关键原则: Op入口不做任何算法选择逻辑，完全委托Selector和HcclExecOp。算法选择的职责边界必须干净。

#### 步骤3: Selector实现——5路分派中选择路径

Selector是算法选择的核心。每个算子有自己的AutoSelector子类，继承AutoSelectorBase，override以下虚方法(按分派优先级):

1. `SelectDPUAlgo()` — DPU路径(最高优先级)
2. `SelectCcuMsAlgo()` — CCU MS模式
3. `SelectCcuScheduleAlgo()` — CCU Schedule模式
4. `SelectAivAlgo()` — AICPU+AIV模式
5. `SelectAicpuAlgo()` — 纯AICPU模式(兜底)

每路返回NOT_MATCH则自动降级到下一路。注册使用宏: `REGISTER_SELECTOR_BY_OPTYPE(HCCL_CMD_XXX, priority, XxxAutoSelector)`。

Case FT-1(new selector, hccl:549c4537, 220文件)展示了Selector架构的核心设计:
- 引入TopoInfoWithNetLayerDetails继承自TopoInfo，保持旧代码兼容
- 序列化采用"尾部追加"策略——基类字段照常序列化，子类新增字段append到末尾
- 将HcclExecOp拆分为Selector+HcclExecOp两个独立阶段(算法选择和执行分离)
- 所有14个算子的Selector虚方法签名必须同步修改(继承体系要求)

Case AA-3(AG&AR midcount optim for A3, hcomm:c8a15c3e)展示了在已有Selector中精确插入新分支的方法:
```
} else if (smallCountOptimMultiServer || smallCountOptimSingleServer) {
    algName = "AllGatherSmallCount";
} else if (midCountOptimMultiPod) {            // <-- 新增位置
    algName = "AllGatherMidCountFor91093Executor";
} else if ((param.supportSymmetricMemory || param.supportZeroCopy) && ...) {
```
midcount分支精确插入在smallCount之后、symmetricMemory之前——优先级链的顺序决定算法选择结果，插入位置必须仔细推敲。

#### 步骤4: Executor选择——继承InsCollAlgBase(V2主力)

hccl的执行器有两代:
- V1 ExecutorBase: 仅scatter使用，提供基本的RunAsync接口
- V2 InsCollAlgBase: 28个子类，是所有新算子的主力基类

新算子应该继承InsCollAlgBase(V2)。注册使用宏: `REGISTER_EXEC_V2(algName, ExecClassName)`。

Executor的核心职责:
- 从ExecMem中获取inputMem/outputMem/scratchMem
- 调用hcomm的通信原语(通过C ABI边界HcommXxx系列函数)
- 编排多步通信操作的顺序和依赖

Case FT-4展示了DPU模式的简化Executor(209行): 直接编排Host DPU传输和本地Reduce两个模板，不走hcomm communicator——DPU Executor比Device-side Executor简单得多，因为跳过了hcomm框架的调度层。

#### 步骤5: hcomm接口调用——V1还是V2路径

hccl通过C ABI边界调用hcomm。关键判断是走V1还是V2:

- V2路径(推荐，新芯片): `IsNewDevice()`返回true时走framework/next/coll_comms
- V1路径(旧芯片兼容): 走framework/device

Case FT-2(add A5 aicpu communicator, hccl:12a2c84f + hcomm:dba4d87b)展示了这个分支:
- hccl侧kernel_launch.cc中仅增加4行DevType判断: `if (param->deviceType != DevType::DEV_TYPE_910_95) { HcommAcquireComm(param->commName); }` — A5芯片不再需要旧的HcommAcquireComm
- hcomm侧新建独立模块(4个新文件, +728行): CollCommAicpu在framework/next下独立运行

关键原则: 为新芯片添加通信域管理时，优先在framework/next下新建独立模块，从旧的巨型类(如HcclCommAicpu，5000+行)中删除芯片专用代码，而非在旧类中添加分支。跨仓提交必须同步(FT-2两仓间隔仅4分钟)。

#### 步骤6: alg_param.h扩展——评估变更放大效应

alg_param.h是hccl/hcomm的跨仓接口核心头文件，Phase 1已知它被修改14次，是"变更放大器"——每次修改都引发联动编译。

Case AA-1(兼容Abi&接口改名, hccl-dev:217dd2cf + hcomm-dev:f50889af)展示了正确的扩展模式:
- 定义HcclAbiHeader结构体(version + magicWord + size + reserved): 所有跨仓结构体必须有ABI头保证版本兼容
- 接口声明为`__attribute__((weak))`弱符号: 允许hccl声明接口但不实现，运行时由hcomm提供实际实现。这是解耦两仓编译依赖的标准手段

新增参数结构体时:
- 使用继承+尾部追加序列化(如FT-1的TopoInfoWithNetLayerDetails继承自TopoInfo)
- 在枚举末尾追加新值(如FT-3在CcuInstType枚举末尾新增4个值)，不要在中间插入
- 考虑向后兼容: 旧版本的hccl能和新版本的hcomm配合工作

#### 步骤7: ST测试——测试桩同步更新

hccl的ST测试依赖hccl_stub.cc提供hcomm接口的mock实现。新增算子时:
- 在hccl_stub.cc中添加新接口的stub
- 编写覆盖各执行模式的测试用例(至少覆盖: 单卡/多卡、各数据类型、各拓扑)
- Case FT-7(symmetric memory)展示了标杆级的UT比例: 3930行新代码中1245行是UT(32%)

### 常见陷阱

#### 陷阱1: 大规模Selector重构后未做全模式回归测试

Case FT-1(new selector, hccl:549c4537, 220文件)在合入次日即暴露BF-9(hccl:80a70729): CCU模式scatter_mesh1d算子的地址设置时机出错。220个文件的联动修改中，真正的架构变更只集中在5个文件，其余180个是机械性签名修改——但正是这种"看似机械性"的修改掩盖了CCU scatter中的时序问题。

教训: Selector架构变更必须配合全算子(14个) x 全设备模式(CCU/AICPU/AIV) x 全拓扑(MESH/CLOS/2-die)的回归测试矩阵。

#### 陷阱2: 跨V1/V2边界+大改op_base.cc+无feature flag = 当日revert

Case FT-6(coll comm & orion comm mix-running, hcomm:ebb54874, 37文件+1140行)在合入仅10小时后被完整Revert(hcomm:0c033874)。致命因素:
- 同时修改next框架和legacy框架(跨越V1/V2边界)
- op_base.cc大量修改(560行新增)——这是算子执行的核心编排文件
- 两位作者联合开发增加了代码一致性风险
- 缺少feature flag意味着一旦出问题只能全量回退

正确做法: 影响op_base.cc的大功能应分步合入(先资源管理→再编排→再端到端功能)，每步独立验证。带feature flag让功能可运行时关闭。

#### 陷阱3: 多芯片适配时fallback路由条件顺序错误

Case BF-7(fix a3 bug, hccl:3e9fbbd)揭示: scatter_op.cc中有复杂的5级fallback路由(CheckHCCLIndependentOp → deviceType → RunIndependentOpExpansion → AclGraph → WorkflowMode)，A3芯片因为条件顺序问题错误地走到了老流程。修复后还特意加了注释: "调用位置有特殊要求，不要变化"——这是血泪教训的沉淀。

同时BF-7还揭示了结构体指针偏移的危险: AlgResourceCtx的sizeof计算少算了两个字段导致指针偏移错位。结构体新增字段后所有手动偏移计算(reinterpret_cast + sizeof)都必须同步更新。

#### 陷阱4: scatter是最活跃算子(20次修改)——初始设计不稳定

Phase 1统计显示scatter被修改20次(其他算子4-8次)。scatter正在经历从旧架构到新架构的迁移(Case XR-2展示了scatter从device launch切换到aicpu process)。新增类似规模的算子时，预期初始版本会经历多轮迭代，应在设计时就考虑可演进性:
- Op入口不做算法选择(委托Selector)
- Executor可替换(通过注册宏动态绑定)
- alg_param.h扩展使用继承而非修改(向后兼容)

---

## 11.2 场景: 给hcomm新增一种通信算法

### 场景简述

你需要在hcomm仓库中实现一种新的集合通信算法(如双die全互联的AllGather算法、一种新的pipeline策略)。这涉及algorithm层的AIV/CCU kernel编写、framework层的通信器集成、以及与hccl Selector的协调。

一个中等复杂度的新算法涉及10-15个新文件、2000-3000行新增代码(Case FT-3: 29 files, +3057行)。如果同时适配多种算子，规模更大。

### 前置知识

- Phase 2.3(hcomm algorithm层): `phase2-hcomm-algorithm.md` — 两个base class(AlgorithmAivBase / AllReduceOperatorAiv)、宏驱动kernel分派、同步协议、数据搬移引擎
- Phase 3(CCU算法): `phase3-hccl-operators.md` — CCU三层文件(context + instruction + template)
- Phase 2.2(hcomm framework层): `phase2-hcomm-framework.md` — V1/V2双路径分派、通信器注册

### 决策清单

#### 步骤1: Base class选择——单算法还是复合pipeline?

hcomm的AIV算法模板有两个独立的基类体系(Phase 2.3分析):

- AlgorithmAivBase: 适用于单阶段算法。直接继承，实现Run()方法。适合AllGather、ReduceScatter等一次性数据搬移的算法。
- AllReduceOperatorAiv: 适用于复合+pipeline算法。AllReduce = ReduceScatter + AllGather的经典分解，需要多阶段流水线编排。适合需要多步协调的复杂算法。

判断标准: 如果算法可以用"一次遍历所有rank"完成，用AlgorithmAivBase；如果需要"先做A操作再做B操作"且两者可以pipeline重叠，用AllReduceOperatorAiv。

Case FT-3(double die alg, hcomm:fa8ed355)为双die拓扑新增CCU算法时，4种算子(AllGather/AllToAll/AllToAllV/ReduceScatter)都继承各自的base class。每个新算法需要context+instruction+template三层文件。

#### 步骤2: Kernel分派——使用HCCL_KERNEL_DISPATCH_XXX宏

AIV kernel入口点通过宏生成per-type的kernel函数(Phase 2.3分析):

```
#define AIV_ALL_REDUCE_KERNEL_BATCH_DEF(type) \
extern "C" __global__ __aicore__ void aiv_all_reduce_##type(KERNEL_ARGS_DEF) { ... }
AIV_ATOMIC_DATA_TYPE_DEF(AIV_ALL_REDUCE_KERNEL_BATCH_DEF);
```

两类数据类型:
- AIV_ATOMIC_DATA_TYPE_DEF: float/half/int16_t/int32_t/int8_t/bfloat16_t(6种，支持原子操作) — 用于AllReduce/ReduceScatter
- AIV_COPY_DATA_TYPE_DEF: 上述6种 + uint16_t/uint8_t/uint32_t(9种，仅拷贝) — 用于AllGather/AllToAll

Case AA-2(Aiv A3 Broadcast AGV RSV, hcomm-dev:88f70272)展示了为新芯片注册kernel的标准模式:
- hccl_aiv.cc中批量注册A3专用kernel: 每种操作6-9个新条目，使用KernelArgsType::ARGS_TYPE_SIMPLE标记
- A3 kernel名称带"_cn"后缀(如aiv_broadcast_cn_half)以区分
- 引入AivKernelArgsV4结构体(扩展V3，添加serverNum/devType/deterministic/extraArgs)
- 每种数据类型需要单独注册，不能省略——这是编译器限制(设备侧kernel在编译时确定参数格式)

#### 步骤3: 同步协议——WaitLocal→DataCopy→SignalRemote→WaitRemote

所有AIV通信算法都遵循统一的同步四步(Phase 2.3分析):

1. WaitLocal: 等待本地数据就绪(本地rank的计算/上一步搬移完成)
2. DataCopy: 执行数据搬移(MTE2: DDR→L1 或 MTE3: L1→DDR)
3. SignalRemote: 通知远端rank数据已到位
4. WaitRemote: 等待远端rank的确认

这四步的顺序不可颠倒。Case BF-4(fix bugs of CP algorithm, hcomm:9bcb1bdc)的根因之一就是inter/intra两层的启动顺序颠倒: 等待应在inter之前而非intra之前——因为inter才需要远端信息。

同步原语:
- Record/Wait/Record1vN/WaitNv1: 基于Notify的点对点和广播同步
- CountRecord/CountWait: 基于计数的聚合同步
- 这些原语定义在AivCommBase基类中(aiv_communication_base.h:247-598)

#### 步骤4: 数据搬移——MTE2(DDR→L1) / MTE3(L1→DDR)

AIV kernel使用AscendC编程模型的两个搬移引擎:

- MTE2: DDR→L1(Global Memory → Unified Buffer)，用DataCopyGM2UB
- MTE3: L1→DDR(Unified Buffer → Global Memory)，用DataCopyUB2GM

GM_IN[16]/GM_OUT[16]数组管理16卡间的CCL buffer地址。L1 buffer使用TQueBind/TBuf管理UB(Unified Buffer)。

跨节点通信时涉及SDMA/RDMA:
- SDMA: 节点内DMA(同一server内)
- RDMA: 跨节点DMA(RoCE网络)
- P2P: PCIe直连

Case BF-4展示了搬移方向选择的陷阱: 原设计在收到receive信息前用SDMA写模式(InterSdmaTx)，收到后用SDMA读模式(InterSdmaRx)——写模式的同步协议更复杂，最终被简化为统一使用读模式。教训: 优先使用更简单的搬移模式(读优于写)，除非有明确的性能理由。

#### 步骤5: CCU算法三层文件——context+instruction+template

如果新算法走CCU模式(设备侧Communication Control Unit)，需要遵循严格的三层文件结构(Phase 3分析):

1. ccu_context_xxx: 准备CCU执行所需的参数(地址计算、notify映射、slice分配)
2. ccu_instruction_xxx: 定义CCU微指令序列(DataCopy/WaitNotify/SignalNotify)
3. ccu_temp_xxx: 从AlgTemplateBase继承，组装context和instruction

Case FT-3(double die alg, hcomm:fa8ed355, +3057行)展示了完整的CCU算法搭建:
- 在CcuInstType枚举末尾新增4个值(CCU_ALLTOALLV_MESH_2DIE_DIRECT等)——必须在末尾追加，不能在中间插入
- 4种算子各新增3个文件(context+instruction+template)，共12个新文件
- legacy/unified_platform/ccu_context/ccu_context.cpp新增241行case分支处理新指令类型
- 4个Executor各新增3行注册新算法名

一个拓扑形状通常需要同时适配4种以上集合通信算子，规模在10-15个新文件(+2000-3000行)。

Case OPT-4(RS & AAV FastLoad, hcomm:8b0dc7b9)展示了CCU快速下发(SuperFastLoad)的扩展模式:
- 在TryFastCcuLaunch中增加opType判断
- 为新算子设计ccuParamsMappingKey(缓存命中的关键)
- 如果参数格式特殊(如AlltoallV的变长args)，新增专用的FillArgs函数
- 特殊拓扑(如2-die)需要额外的后处理步骤(inline reduce)

#### 步骤6: Communicator集成——V1/V2哪条路径注册

新算法需要在hcomm的通信器中注册。两条路径:
- V1路径(legacy): 在legacy/service/collective/alg/下注册，走旧的communicator_impl
- V2路径(next): 在framework/next/coll_comms/下注册，走新的IndependentOp流程

Case FT-5(alg selector remake, hcomm:0d9239b5, 25文件+1396行)展示了hcomm层Legacy Selector的重塑——这是FT-1(hccl new selector)在hcomm层的对应物:
- 在base_selector.h中引入与hccl层概念对齐的嵌套结构体(NetLayerDetails, TopoInstDetails)
- 8个算子selector各自更新算法选择分支
- 概念相同但枚举值不同: Level0Shape在hccl是{CLOS=0, MESH_1D=1, MESH_1D_CLOS=2}，在hcomm是{MESH_1D=1, MESH_2D=2, CLOS=3, MESH_1D_CLOS=4}

关键认识: hccl层和hcomm层的selector是独立演进的平行体系——概念对齐但实现独立。修改算法选择逻辑时，两层都需要更新(先更新概念层hccl，再更新实现层hcomm)。

Case FT-2(add A5 aicpu communicator)展示了新芯片走framework/next的模式: A5(910_95)走新框架，旧芯片走旧框架(framework/device)。新算法如果仅面向新芯片，只需在next路径注册；如果需要兼容旧芯片，两条路径都要注册。

#### 步骤7: 拓扑约束——ring/mesh/tree/2-die

每种算法对拓扑结构有特定约束。常见拓扑:
- MESH_1D: 单层全互联(最常见)
- MESH_1D_CLOS: CLOS+MESH混合拓扑
- 2-die fullmesh: 双die全互联(910_95新拓扑)
- ring: 环形拓扑(经典但性能次优)

拓扑判断使用CalcTopoLevelNums/CalcLevel0TopoShape/Is2DieFullMesh等工具方法(FT-1引入)。新算法必须明确声明自己适用的拓扑范围，在Selector中做前置排除(如AllReduce的CCU MS模式排除多级拓扑/INT8/PROD/FP64)。

### 常见陷阱

#### 陷阱1: 分层pipeline算法中层间数据依赖未显式建模

Case BF-4(fix bugs of CP algorithm, hcomm:9bcb1bdc)在AlltoAllV的Continuous Pipeline算法中发现三个独立bug:
1. intraRecvCounts_初始化遗漏: needCollectInfo_=true后没有初始化接收计数，导致recvCount全为0
2. inter/intra顺序颠倒: "第一轮等待receive信息"的条件从intraState改为interState——因为inter才需要远端信息
3. SDMA路径冗余: 删除整个InterSdmaTx函数，统一使用InterSdmaRx

教训: pipeline算法中"哪层先获得信息、哪层先启动"必须用显式的数据依赖图来建模，不能依赖代码顺序隐式表达。

#### 陷阱2: CCU template参数错误导致死锁——症状与根因距离远

Case BF-5(CCU NHR template bugfix, hccl:992fd5b)仅改了1个数字: GetThreadNum()从1改为2。NHR 1D mem2mem模式需要2个线程(一个计算搬移、一个通信同步)，错误的线程数导致CCU调度死锁。

这印证了Phase 1结论: "CCU template是bug集中区"。CCU参数(threadNum/loopNum)影响设备侧调度，错误参数导致的症状(死锁/超时)与根因(参数数值错误)距离很远，定位成本极高。新增CCU template时必须对每个GetXxxNum方法做单元验证。

#### 陷阱3: CCU参数修改的验证周期是17天而非6小时

Case RV-8/RV-9(loopnum fix-revert-revert): `CCU_MS_DEFAULT_LOOP_COUNT`从64改为128(仅1行常量)，6小时后被revert(hcomm:72cdf80e)，17天后经过充分硬件验证才re-revert恢复(hcomm:0c3be05f)。

时间线: fb56d64b(2/11 11:37修改) → 72cdf80e(2/11 17:39 revert) → 0c3be05f(2/28 15:37 re-revert)

这个chain证明: CCU设备端参数修改即使代码量极小(1行)，影响面也覆盖所有CCU调度路径。"当日合入当日revert"是硬件验证不足的典型信号。CCU参数变更需要:
1. UT通过 + 真实硬件验证(不能只靠UT)
2. 用Grep统计所有引用该常量的地方，评估影响面
3. 不要在无法快速回退的时间点提交(周末/假期前)

#### 陷阱4: CCU SuperFastLoad扩展遗漏insType字段

Case OPT-4(RS & AAV FastLoad, hcomm:8b0dc7b9)在扩展CCU快速下发时，CachedCCUParams的move构造函数需要正确传递新增的insType字段。AlltoallV使用的ccuParamsMappingKey与其他算子不同(`{reduceOp, sendType, 0}`)，且需要专用的FillAllToAllVArgs函数(跳过index=2的token info)。

教训: 缓存机制的扩展必须检查: key设计是否唯一(不与已有算子冲突)、参数填充是否完整(move/copy时所有新字段都传递)、特殊后处理是否覆盖(如2-die的inline reduce)。

---

## 11.3 场景: 修复一个通信超时/死锁bug

### 场景简述

你在集群环境中遇到了通信超时或死锁——集合通信算子挂住不返回。你需要定位根因并修复。这是hccl/hcomm中最高频的bug类型: Phase 8统计中BUGFIX占303条(21.3%)，其中时序/顺序错误占33%、并发/锁问题占22%、编译条件遗漏占22%。

通信超时/死锁的最大挑战在于"症状与根因的距离": host侧看到的是超时，但根因可能是CCU参数错误(BF-5)、设备侧地址时机问题(BF-9)、或重执行状态机初始化遗漏(BF-3)。13个BUGFIX landmark case中11个涉及Host-Device边界——设备侧代码的bug密度显著高于host侧。

### 前置知识

- Phase 5.3(错误传播切片): `phase5-slice-error-propagation.md` — CHK_RET 9层传播、hccl-hcomm边界双返回类型(HcclResult↔HcomResult)、CQE异步错误检测
- Phase 2.1(platform层): `phase2-hcomm-platform.md` — Notify三轮握手(Send→Wait→Release)、DMA传输方向(SDMA/RDMA/P2P)
- Phase 1(bug统计): communicator_impl.cc(17次bugfix修改)、重执行3个bugfix、并发控制薄弱

### 决策清单

#### 步骤1: 日志定位——按九层传播路径逆向追踪

通信超时的日志定位应从外向内、按九层传播路径逆向追踪(Phase 5.3分析):

```
层1 [hccl API] → 层2 [hccl Op] → 层3 [hccl Selector]
→ 层4 [hccl Executor] → 层5 [hccl Template, 跨仓边界]
→ 层6 [hcomm dispatcher] → 层7 [hcomm comm_primitive C ABI]
→ 层8 [hcomm Transport] → 层9 [hcomm driver/runtime]
```

核心传播机制是CHK_RET宏(hccl使用1448次, hcomm使用14097次)，它会在每一层自动追加`[__func__]call trace: hcclRet -> N`日志，形成完整的调用栈追踪。

关键: CHK_RET永远不改变错误码值——错误码从最底层原封不动透传到最顶层。唯一转换错误码的是CHK_PRT_RET宏(hcomm中1131次使用)。

特殊处理: HCCL_E_AGAIN走WARNING而非ERROR(重试语义)，日志级别不同需注意筛选。

#### 步骤2: Notify状态检查——三轮握手哪一步卡住

Notify三轮握手(Phase 2.1分析):
1. Send(NotifySend/NotifyRecord): 通知对端数据就绪
2. Wait(NotifyWait/NotifyWaitAndReset): 等待对端通知
3. Release(NotifyRelease): 释放Notify资源

如果超时在Wait阶段，说明对端的Send没有执行或延迟——需要检查对端rank的执行状态。

Case PL-6(host&device sync, hccl:89a09db2 + hcomm:d813063d)展示了从裸ACL Notify迁移到Thread Export体系的过程。旧方案通过thread_local全局数组`g_notifies_host_with_device[2]`管理控制Notify，绕过了hcomm的资源管理体系——导致资源泄漏风险和无法被框架统一追踪。

迁移后的同步流程: `HcclThreadAcquireWithStream()` → `HcclThreadExportToCommEngine()` → `HcommThreadNotifyRecordOnThread()` / `HcommThreadNotifyWaitOnThread()`

关键原则: 跨Host-Device边界的同步机制必须纳入框架的资源管理体系(Thread/Notify/Stream)，而非直接调用底层ACL接口。如果看到代码直接使用aclrtRecordNotify/aclrtWaitAndResetNotify，这可能是bug的温床。

#### 步骤3: 错误码追踪——HcclResult↔HcomResult边界转换

hccl和hcomm使用不同的返回类型:
- hccl: HcclResult(全局枚举，HCCL_SUCCESS=0到HCCL_E_SUSPENDING=22)
- hcomm: HcomResult(框架内部枚举)

在跨仓边界(层5 hccl Template → 层6 hcomm dispatcher)存在类型转换:
```cpp
CHK_RET(static_cast<HcclResult>(HcommWriteOnThread(...)))
```
如果hcomm侧返回了新定义的错误码但hccl侧没有对应枚举值，static_cast可能产生无意义的值。排查时需要同时检查两侧的错误码定义。

#### 步骤4: CQE异步错误——区分同步传播vs异步检测

通信错误有两条检测路径:
- 同步传播: CHK_RET逐层返回，调用者立即知道失败
- CQE异步检测: Completion Queue Entry报告的异步错误，可能在操作完成后才被发现

Case PL-2(HB check op inconsistent, hcomm-dev:ebfcde5c)展示了CQE检查与算子一致性检查的交互: `HcclGetCommAsyncError()`中先检查CQE(同步路径)，CQE通过(HCCL_SUCCESS)后才检查算子不一致。

如果超时但没有错误日志，很可能是异步错误——需要检查CQE状态。重执行路径的BF-1也涉及CQE异常状态重置。

#### 步骤5: 重执行路径——"先清理物理资源，后清除异常标记"

Case BF-1(修复重执行多线程下的时序问题, hcomm:75659d24)是经典的error-fix pair:

错误引入: d17b1f3a("supoprt aicpu cache for alltoallv", 34文件+3684行)——大型特性commit提取了aicpu_communicator.cc中400行OpUnfoldCache逻辑到独立文件。这个重构间接影响了retry路径的时序。

修复: 仅2行语句顺序调换——
1. HcclOpExecFsmWaitRetryProcess: 将dfxExtendInfo_状态重置移到ResetOpRetryException之前(原: Reset→dfxReset; 改: dfxReset→Reset)
2. CleanStream: 将ResetStreamCqeExceptionStatus移到ClearLocalBuff+UpdateSqStatus之后(原: ResetCqe→ClearBuff→UpdateSq; 改: ClearBuff→UpdateSq→ResetCqe)

两处修改的共同逻辑: 先完成物理层面的清理(buffer/SQ/DFX状态)，再清除异常标记。确保"状态一致性窗口"最小化——在异常标记被清除时，底层已经是干净的状态。

这个原则适用于所有重执行/恢复路径: 先做物理清理，后做逻辑重置。

#### 步骤6: 并发检查——锁粒度是生死线

Case BF-2(读写锁改为基于原子变量的实现, hcomm:1535b1c4)展示了正确的并发修复:
- 将ReadWriteLockBase从mutex+condition_variable改为纯atomic实现
- 单个atomic<uint64_t> state_编码所有状态: bit63=WRITING_BIT, bit62=WAITING_BIT, bit0-61=reader count
- 同时调整executor_tracer.cc中的锁粒度: 从"整个循环外加writeLock"改为"每次迭代内加writeLock"

对比反面案例: hcomm-dev中c546d353在op_base_mc2.cc的HcclAllocComResourceByTiling中加std::lock_guard<std::mutex>(全局锁)，随后被merge-revert-mr回退。根因: 高层API加全局mutex，锁粒度太粗，导致整个资源分配序列化，性能退化严重。

教训: 纯atomic成功(BF-2) vs 粗mutex被revert(c546d353)——通信框架中锁粒度是生死线。并发问题的修复不能简单地"加一把大锁"，需要分析读写模式后选择合适粒度。

Case PL-5(fix changelink for OpRetry WaitResume, hcomm-dev:28f18ab5)进一步展示了全局变量的危害: 删除了`g_isRdmaError`全局变量(外部bool)，消除了多线程竞态风险。合并重执行的两条分支为一条"无条件changeLink"路径——"宁可多做恢复动作也不要漏做"是更安全的默认策略。

用全局变量在多线程环境中传递状态机决策是反模式。

#### 步骤7: 回归测试——semantics_check框架

修复后的验证不能只跑复现场景。需要:
- semantics_check框架验证语义正确性
- 全算子 x 全拓扑的回归测试
- 重执行路径的独立测试(Phase 7 validation中发现的缺失规则)

### 常见陷阱

#### 陷阱1: 大型feature commit间接影响retry路径——"修改X的调用环境也能引入X的bug"

Case BF-1(error-fix pair): 错误commit d17b1f3a(34文件)本身没有直接修改CleanStream或retry路径的语句，但它重构了cache逻辑，使得新的cache miss/hit路径引入了更多并发访问。这导致了retry路径的时序问题。

更严重的是: 错误commit的MR描述几乎为空(所有字段未填)，review时可能因此跳过了深入审查。

教训: feature review时需要特别审查retry/异常/并发路径是否被间接影响。即使commit没有直接修改这些路径，如果修改了这些路径的调用环境(如cache/buffer管理)，也可能引入bug。2行修复 vs 34文件引入——定位成本远大于修复成本。

#### 陷阱2: 状态机初始状态必须在构造/注册时显式设置

Case BF-3(修复Step快恢失败问题, hcomm:994390df)揭示了两个问题:
1. Agent/Server的retry状态机在注册时没有正确初始化状态——retryCtx创建后状态为未定义值，必须显式调用SetRetryState(RUNNING)
2. CheckExitWaitResumeState在非AICPU环境下也被调用——必须有环境判断门控

教训: 不要依赖默认值或后续流程隐式初始化。状态机注册时就应处于定义明确的初始状态。设备侧路径(AICPU/CCU)的容错逻辑必须有环境判断门控(`GetAicpuUnfoldFlag() || GetAicpuCommEngine()`)。

#### 陷阱3: CCU_WHILE内部不应放初始化代码——CCU控制流不等于C++控制流

Case BF-9(CCU模式scatter_mesh1d修改地址设置时机, hccl:80a70729)揭示了CCU编程模型的核心陷阱:

原代码将地址设置循环(for curId = 0 to rankSize_)放在DoScatter()内的CCU_IF块中。CCU_WHILE/CCU_IF是CCU控制指令，不是普通的while/if——当DoRepeatScatter使用CCU_WHILE循环多轮scatter时，地址设置在循环体内部被重复执行。

修复: 将地址设置循环从DoScatter()移到DoRepeatScatter()中，放在CCU_WHILE循环之前。确保地址在循环开始前正确初始化，CCU_WHILE内部只做增量偏移。

教训: CCU微码中，初始化代码(地址设置/参数计算)必须放在CCU_WHILE循环外部。CCU_WHILE内部应只包含每轮需要变化的增量逻辑。这是CCU编程模型与普通C++的关键区别。

#### 陷阱4: 完全验证设备侧代码的bug密度更高(13个BUGFIX中11个涉及Host-Device边界)

Phase 10.2的交叉分析显示:
- 9个BUGFIX中7个涉及Host-Device边界(BF-4到BF-9 + BF-1间接涉及)
- 4个LIFECYCLE case中3个涉及Host-Device边界(LC-1/LC-3/LC-4)

原因分析:
1. 编译目标分裂: 同一代码需要编译为host.so和aicpu.so，条件编译(AICPU_COMPILE宏)容易遗漏。Case BF-6(fix missing signals, hccl:b779699)就是因为channel.cc中的GetTopoTypeByLink函数在AICPU编译时链接了Host-only API，需要#ifndef AICPU_COMPILE条件编译包裹
2. 语义分裂: CCU微码的控制流(CCU_WHILE/CCU_IF)不等同于C++控制流(陷阱3)
3. 调试困难: 设备侧bug的表现(超时/死锁)与根因(参数错误/地址时机)距离远(BF-5: 仅改1个数字修复死锁)
4. 状态同步: host和device之间的flag/状态传递是脆弱的。Case LC-3(Judge resource Update From host to kernel, hcomm:9167c142)将"是否需要刷新资源"的判断从host侧移到device侧——因为使用方(device kernel)最清楚自己的当前状态

修复通信超时/死锁时的优先检查顺序:
1. 设备侧参数是否正确(CCU threadNum/loopNum, AICPU kernel参数)
2. Host-Device同步Notify是否正确(三轮握手、Thread Export)
3. 条件编译是否遗漏(AICPU_COMPILE保护)
4. 重执行状态机是否正确初始化
5. 并发锁/原子变量是否使用正确

#### 陷阱5: 语句重排2行修复——但定位可能花费一整天

Case BF-1的修复只有2行(+2/-2)，但错误引入commit有34文件。hcomm:75659d24(修复重执行多线程下的时序问题)和hccl:80a70729(CCU地址设置时机)都属于"修复极小但定位极难"的bug类型。

对于这类bug，可利用的线索:
- git log --follow -- <出问题的文件>: 追踪最近的变更，特别关注"特性+重构"混合commit
- Phase 1已知的bug热点: communicator_impl.cc(17次bugfix)、scatter(20次修改)、CCU template(bug集中区)
- BUGFIX根因分布: 时序/顺序(33%) > 并发/锁(22%) > 编译条件遗漏(22%) > 参数(11%) > 状态机(11%)——按此概率依次排查

---

# Phase 11 场景化开发指南 (Part B): 场景 11.4 / 11.5 / 11.6

日期: 2026-03-16
产出: 3个场景化开发指南(适配新芯片 / 优化通信性能 / 跨模块重构)
方法: 从Phase 10 landmark cases中提炼决策点，结合Phase 0-7架构知识，重组为面向任务的决策清单

---

## 11.4 场景: "适配一个新芯片平台"

### 场景简述

你的团队需要让hccl/hcomm通信框架在一款新芯片上跑通。这不是"从零写一套"，而是在已有的多芯片支持体系(910B/910_93/910_95等)中增加一个新成员。核心挑战在于: 新芯片的硬件能力与现有芯片存在差异(某些功能不支持或行为不同)，你需要决定在哪些层做适配、用什么策略处理差异、如何保证旧芯片路径不受影响。

### 前置知识

- Phase 0 (phase0-architecture-map.md): hccl有14个算子目录，统一selector/executor/template结构; hcomm五层架构algorithm/framework/platform/legacy/common
- Phase 4 (phase4-contrastive-analysis.md): hccl与hcomm的跨仓边界、条件编译模式、能力降级策略
- Phase 2.1 (phase2-hcomm-platform.md): platform层的C ABI边界(extern "C" + void*句柄)，comm_primitive接口是跨仓调用的底层通道
- Phase 3 (phase3-hccl-ops.md): hccl算子层的5路分派Selector(CCU MS/CCU Schedule/AICPU+AIV/AICPU/Legacy)

### 决策清单

#### 第1步: 确定能力探测策略 — capability-based还是device-type-based?

这是整个适配工作中最关键的架构决策。两种策略在hccl/hcomm中都有使用，但它们的适用场景和维护成本完全不同。

capability-based(运行时能力检测): 通过调用硬件查询API，在运行时探测芯片的实际能力，再据此选择执行路径。Case AA-5(hcomm:4afb1c39, RoCE flush适配)是这种策略的典范: FlushHandle::Init()中通过`CHK_RET(GetLbMax(&rdmaHandle, &lbMax))`查询网卡的负载均衡上限，`lbMax > 0`则标记支持flush opcode。这段代码只有5文件36行，但它的设计精髓在于 —— 同一芯片搭配不同网卡时会自动选择正确路径，不需要为每种芯片-网卡组合硬编码分支。

device-type-based(设备类型硬编码): 通过`DevType`枚举值做if/else分支。Case AA-2(hcomm-dev:88f70272, A3 AIV适配)展示了这种策略: operator层通过`if (devType == 910_93 && isAivMode)`分派到A3专用executor。这种策略直接、易读，但每新增一款芯片都要在所有分支点添加新条目。

判断标准: 如果差异来自芯片固有特性(指令集/内存模型/计算单元数量)，用device-type-based — 这些特性在编译期就确定了，不会因运行环境变化。如果差异来自外围设备或可配置能力(网卡类型/链路带宽/flush支持)，用capability-based — 这些特性在不同部署环境中可能不同。

反模式: Case RV-10(hcomm:f0666214, sdma revert)展示了错误做法: 用`aclrtGetSocName()`获取芯片名字符串，然后做`targetChipVerStr.find("Ascend950") != std::string::npos`匹配。这种做法有三个问题: (1)芯片名可能随SDK版本变化(如"Ascend950"改为"Ascend950A"); (2)绕过了device_capacity.cc中已有的DevType枚举体系; (3)字符串比较在热路径上有不必要的开销。正确做法是在device_capacity.cc中注册新芯片的能力标记，使用方通过DevType枚举判断。

#### 第2步: 处理条件编译 — #ifdef还是runtime分支?

hccl/hcomm的代码需要编译为两个目标: host侧.so和AICPU侧.so(设备端)。`AICPU_COMPILE`宏区分这两个编译目标。

Case BF-6(hccl:b779699, missing signals)展示了条件编译的典型需求: channel.cc中的`GetTopoTypeByLink`函数调用了`HcclRankGraphGetTopoInstsByLayer`——这是一个host-only API，在AICPU编译环境中不存在。修复方法是用`#ifndef AICPU_COMPILE`包裹host-only实现，AICPU编译时直接返回`HCCL_SUCCESS`。只有2文件5行改动，但这个bug直到链接阶段才暴露(编译不报错)。

操作规则:
- 新增的.cc文件必须同时检查host和AICPU两个CMakeLists.txt。BF-6的根因就是scatter_aicpu_kernel的CMakeLists.txt中遗漏了topo_match_ubx.cc
- 引用host-only API的函数(以Hccl/Hcomm开头的RankGraph/Topo查询函数)必须有`#ifndef AICPU_COMPILE`保护
- runtime分支(if/else)用于host侧内部的设备类型区分; 条件编译(#ifdef)用于host/device目标区分

Case BF-7(hccl:3e9fbbd, A3 bug)进一步展示了device-type条件判断的位置敏感性: scatter_op.cc中有5级fallback路由(CheckHCCLIndependentOp → deviceType → RunIndependentOpExpansion → AclGraph → WorkflowMode)，A3芯片(910_93)应该走新流程(IndependentOp)，但旧代码在检查IndependentOp环境变量之前就做了fallback判断。修复方法是将条件改为`!CheckHCCLIndependentOp() && deviceType != DEV_TYPE_910_93`——A3芯片即使没设环境变量也走新流程。同时将CalcTopoLevelNums和CalcLevel0TopoShape调用包在`if (deviceType == DEV_TYPE_910_95)`条件内，隔离仅910_95支持的API。

教训: fallback路由的条件顺序决定了芯片走哪条路径。新增芯片支持时，不能只在链条末尾添加分支，而要逐行审查整个fallback链，找到正确的插入位置。BF-7的修复中还有一个注释"调用位置有特殊要求，不要变化"——这是前人踩坑后沉淀的经验，遇到这类注释要认真对待。

#### 第3步: 设计降级路径 — 新芯片不支持的功能怎么办?

降级路径的设计原则是"显式且安全"。Case RV-10(hcomm:0e5be69c → f0666214)提供了三个反面教材:

反模式1 — nullptr作为功能不可用信号: `GetPubDispatcher()`在Ascend950上返回`HCCL_SUCCESS`但dispatcherPtr保持nullptr，然后在`HcclLocalCopy()`中检查`dispatcherPtr==nullptr`走降级路径。问题在于: 函数契约是"成功则指针有效"，调用方(不只是当前这一处)合理地假设成功后指针非NULL。正确做法是返回一个显式的错误码(如`HCCL_E_NOT_SUPPORT`)或用Optional/枚举明确表达"功能不可用"。

反模式2 — 枚举语义混用: 代码中用`ACL_RT_MEMCPY_SDMA_AUTOMATIC_SUM`(这是memcpy的kind)作为reduce操作的参数。memcpy kind和reduce kind是两个不同的概念域，不能混用。

反模式3 — 无测试: MR的测试字段写"不涉及"——51行新代码没有任何测试就合入了主干。结果6小时后被revert，且截至分析时点未重新引入。

正确的降级设计参考AA-5(RoCE flush): 通过GetLbMax运行时检测能力，`lbMax > 0`走优化路径，否则走保守路径。两条路径都是完整的、可测试的实现，而非一条路径返回成功但实际什么都没做。

#### 第4步: 执行hccl+hcomm跨仓联动修改

新芯片适配几乎必然涉及hccl和hcomm两个仓库的同步修改。Phase 10的跨仓案例展示了一个一致的规律: hccl侧修改极少(1-4文件)，hcomm侧承担主要工作(20-47文件)，比例约为1:10。

Case FT-2(hccl:12a2c84f + hcomm:dba4d87b, A5 aicpu communicator)是最典型的案例: hccl改1文件+4行(kernel_launch.cc中增加DevType判断)，hcomm改26文件+728行(新建CollCommAicpu独立模块、从旧的巨型类HcclCommAicpu中删除A5专用代码)。两仓提交间隔仅4分钟。

操作步骤:
1. hcomm侧先完成实现并export接口(hcomm先提交，因为hccl编译依赖hcomm)
2. hccl侧添加设备类型判断，调用hcomm的新接口
3. 两仓提交间隔应尽量短(FT-2是4分钟，XR-2是18秒)——中间窗口期存在编译不一致风险

关键决策: 对于新芯片在hcomm侧是修改现有巨型类还是新建独立模块? FT-2的选择是新建CollCommAicpu(独立模块)+从HcclCommAicpu中删除A5代码。判断标准: 如果新芯片的执行模型(初始化流程/Channel管理/资源分配)与旧芯片差异超过30%，就应该新建独立模块——在已经5000+行且bug密度最高的巨型类中添加更多if/else分支只会让情况更糟。

#### 第5步: 处理跨仓ABI兼容

Case AA-1(hccl-dev:217dd2cf + hcomm-dev:f50889af, ABI兼容)定义了hccl/hcomm跨仓接口的标准模式:

ABI头结构: 每个跨仓传递的结构体必须有`HcclAbiHeader`头(包含version+magicWord+size+reserved四个字段)。version用于前向兼容(新版本hccl遇到旧版本hcomm时降级处理)，magicWord用于快速校验(防止传入错误类型的指针)，size用于动态分割(基类字段和扩展字段的边界)。

弱符号策略: 所有跨仓接口声明为`__attribute__((weak))`。弱符号允许hccl在编译时只声明接口不提供实现，运行时由hcomm动态库提供实际实现。如果hcomm版本中没有某个函数，链接时不报错而是指向nullptr(调用前需检查)。这是解耦两仓独立编译的关键手段。

初始化函数: 每个跨仓结构体必须有对应的Init函数(如`HcclChannelDescInit`)，确保version/magicWord/size被正确填充。不要依赖memset零初始化——magicWord和version有非零默认值。

#### 第6步: 遵循芯片适配的标准文件命名和注册流程

Case AA-2(hcomm-dev:88f70272, A3 AIV适配)和AA-3(hcomm:c8a15c3e, A3 midcount优化)展示了A3芯片适配的完整流程，这个流程是新芯片适配的标准模板:

第一阶段(AA-2): 基础kernel注册 + executor创建
- 在hccl_aiv.cc中批量注册带`_cn`后缀的kernel条目，标记`KernelArgsType::ARGS_TYPE_SIMPLE`
- 每种数据类型单独注册(如`aiv_broadcast_cn_half`, `aiv_broadcast_cn_float`)——约6-9种数据类型 x N种操作 = 数十个新注册条目
- 新建`_for_910_93`后缀的Executor类(如`CollBroadcastMeshAivFor91093`)
- 新建`_crossnode_91093`后缀的Template头文件
- operator层添加devType分支: `if (devType == 910_93 && isAivMode) { executor = make_unique<CollXxxFor91093>(...); }`

第二阶段(AA-3): 性能优化路径
- 在operator层的`SelectAlgfor91093`方法中添加数据量区间判断(如midcount: 16KB < dataSize <= 256KB)
- 新建专用Executor(如`CollAllGatherMidCountFor91093Executor`)
- 将新分支插入selector的优先级链中正确位置(在smallCount之后、symmetricMemory之前)

命名惯例(必须严格遵循):
- Executor: `_for_910_93`后缀(如`CollAllGathervMeshAivFor91093`)
- Kernel: `_cn`后缀(如`aiv_broadcast_cn_half`)
- Template: `_crossnode_91093`后缀

规模预估: 基础适配(AA-2级别)约18文件+1700行; 性能优化(AA-3级别)约13文件+600行。一个新芯片的完整适配(基础+优化)通常需要30+文件+2000-3000行。

#### 第7步: 编写平台ST测试

新芯片适配的测试不能用"不涉及"搪塞(RV-10的51行代码写"不涉及"测试，6小时后被revert)。

测试范围:
- 每种新注册的kernel都需要基础功能测试(数据类型覆盖)
- 降级路径必须有独立测试(验证新芯片不支持的功能正确降级而非崩溃)
- 跨仓接口必须有集成测试(验证hccl+hcomm联动正确)
- 如果涉及CCU微码参数，需要真实硬件验证(参见RV-9的教训: CCU参数验证周期是17天而非6小时)

### 常见陷阱

陷阱1 — 硬编码芯片名 + nullptr信号 + 枚举混用三合一(RV-10, hcomm:f0666214):
51行新代码集齐了三条反模式且无测试，6小时后被revert且截至分析时点未重新引入。这个case应该被钉在每个新芯片适配PR的review清单上。正确做法: 通过device_capacity.cc的DevType枚举做能力判断 + 用显式错误码表达"功能不可用" + 为每条新路径写测试。

陷阱2 — fallback路由条件顺序错误(BF-7, hccl:3e9fbbd):
scatter_op.cc的5级fallback路由中，A3芯片因为条件顺序问题走到了错误路径。同时`CalcTopoLevelNums`没有用设备类型条件保护，调用了仅910_95支持的API。另一个隐藏问题: 结构体指针偏移的sizeof计算漏算了两个字段——`sizeof(uint32_t)*AICPU_CONTROL_NOTIFY_NUM + sizeof(void*)`。结构体新增字段后所有手动偏移计算都需要同步更新，这种reinterpret_cast+sizeof是高危操作。

陷阱3 — Host/Device双目标编译遗漏(BF-6, hccl:b779699):
新增.cc文件只加到了host侧CMakeLists.txt但忘了AICPU侧的。引用host-only API的函数没有AICPU_COMPILE条件编译保护。这类错误在编译时不报错(因为host目标编译正常通过)，只在AICPU目标的链接阶段才暴露——此时已经延误了整个CI流水线。自检规则: 每新增/修改一个.cc文件，搜索项目中所有CMakeLists.txt确认是否需要在多个target中注册。

---

## 11.5 场景: "优化某算法的通信性能"

### 场景简述

你发现某个集合通信算子(AllReduce/ReduceScatter/AllGather等)在特定场景下(特定数据量/特定拓扑/特定芯片)性能不达标。你需要定位瓶颈、设计优化方案、实现并验证。核心挑战在于: 通信性能优化涉及9层调用栈(API→Op→Selector→Executor→hcomm algorithm→framework→platform→device→硬件)，瓶颈可能在任何一层; 设备端参数修改的验证周期远超预期; 优化方案一旦引入回归很难快速回退。

### 前置知识

- Phase 2.3 (phase2-hcomm-algorithm.md): AIV算法的base class、同步协议(WaitLocal→DataCopy→SignalRemote→WaitRemote)、数据搬移引擎(MTE2/MTE3)
- Phase 5.1 (phase5-slice-allreduce.md): AllReduce从API到硬件的完整9层调用栈
- Phase 3 (phase3-hccl-ops.md): ExecMem结构体(scratchMem/inputMem/outputMem)、5路分派Selector
- Phase 1 (phase1-hcomm-git-history.md): CCU template是bug集中区; loopnum fix-revert-revert循环

### 决策清单

#### 第1步: 定位瓶颈层级 — Selector/Executor/Algorithm/CCU microcode?

性能瓶颈不一定在你以为的地方。通信性能问题通常不是"算法计算太慢"，而是"编排开销太大"或"流水线没有充分利用"。

Case OPT-1(hcomm:fd9d488c, groupSendRecv小数据优化)展示了编排层面的瓶颈: Group特性允许用户批量调用HcclSend/HcclRecv，但原实现是逐个执行——每个Send/Recv都要经过communicator层调度。对于小数据量(<128KB)，逐个执行的编排开销远大于数据传输本身。优化方案不是"让单次Send更快"，而是"把N次单独的Send/Recv合并为1次批量executor调用"(新增CollBatchSendRecvGroupExecutor)。

瓶颈定位的层级分类:
- Selector层: 算法选择策略不优(如没有为特定数据量区间选择专用算法)。AA-3(A3 midcount)就是这类——16KB到256KB的中间数据量两头不靠
- Executor层: 编排逻辑有冗余(如逐个执行改批量)。OPT-1就是这类
- Algorithm层: 算法实现本身可优化(如流水线深度不够、同步点过多)。BF-4(CP算法)涉及的inter/intra两层流水线属于这类
- CCU microcode层: 设备端微指令参数可调优(如loopnum/threadNum)。RV-9(loopnum)和BF-5(threadNum)属于这类

提示: 从上层(Selector)往下排查——上层的编排优化通常比下层的微指令调优风险更低、收益更大。OPT-1的28文件变更没有被revert，而RV-9的1行CCU常量变更触发了fix-revert-revert循环。

#### 第2步: 分析流水线和层间数据依赖

如果瓶颈在algorithm层或executor层，需要理解现有的流水线结构和层间依赖。

Case BF-4(hcomm:9bcb1bdc, CP算法bugfix)是一个典型的流水线分析案例。AlltoAllV的Continuous Pipeline(CP)算法分为intra(节点内)和inter(节点间)两层。这个case中同时存在三个独立bug:

bug 1 — 初始化遗漏: PrepareSendRecvInfo中设置了`needCollectInfo_=true`但没有初始化`intraRecvCounts_`。std::copy填充本rank的recvCounts是必须的，否则后续读取长度为0。

bug 2 — 层间顺序颠倒: RunAsync中"第一轮等待receive信息"的条件从`intraState.loopNum==0`改为`interState.loopNum==0`。等待应在inter之前(因为inter才需要远端信息)而非intra之前。

bug 3 — 冗余路径: InterSdmaTx(写模式)和InterSdmaRx(读模式)并存。写模式需要额外的前后同步协议，但在有receive信息后RX完全够用。删除整个InterSdmaTx函数并统一使用RX是更安全的选择。

教训: 分层pipeline中，层间的数据依赖(哪层先获得信息、哪层先启动)必须显式建模。"同时修复多个独立bug"时，每个fix需要独立测试——这里3个fix的根因各不相同(初始化遗漏/顺序错误/冗余路径)，不能用一个端到端测试覆盖所有问题。

#### 第3步: 设计buffer策略

Phase 3中记载了ExecMem结构体是执行器间共享的内存/参数上下文: inputMem(输入)、outputMem(输出)、scratchMem(临时)。优化时经常需要调整scratchMem的大小或引入额外的buffer。

OPT-1的GroupSendRecv优化中引入了slice切分策略: `CalcBufferSliceSize() = cclInputMem.size() / alignSize / GROUP_MAX_CONCURRENT * alignSize`，SendRecvSlice结构体包含`{addr, size, remoteRank}`。大数据(>128KB)按stream分组(`OrganizeSendItemByStream`，每个remoteRank通过`%streamNum`分配到固定stream)，小数据不按stream分组，主流直接编排。

buffer设计的注意事项:
- scratchMem大小计算必须考虑alignment——Phase 3中记录了对齐要求是32B或512B(取决于DMA引擎)
- 多stream并发时，每个stream需要独立的buffer区间(不能重叠)
- 128KB这类阈值的选择需要实测数据支撑，不能凭直觉(OPT-1的MR描述明确标注了A2/A3单机双机+增量建链的测试场景)

#### 第4步: CCU微指令级优化 — 风险极高的领域

如果优化方案涉及CCU参数修改，请格外谨慎。Phase 1已知CCU template是bug集中区，而Phase 10的两个case从正反两面展示了CCU优化的风险与收益。

高风险案例 — RV-9/RV-8(hcomm:72cdf80e/0c3be05f, loopnum): 1行常量变更`CCU_MS_DEFAULT_LOOP_COUNT = 64 → 128`引发了fix-revert-revert三段式循环。时间线: 2/11上午提交 → 2/11下午revert(6小时) → 2/28 re-revert恢复128(17天后)。1行代码的影响面覆盖了所有CCU调度路径。UT中profiling数量从3变为2是正确的行为变化，但在某些边界场景下触发了硬件异常。CCU参数修改的验证周期是周级(17天)而非小时级(6小时)。

成功案例 — OPT-4(hcomm:8b0dc7b9, RS & AAV FastLoad): 将CCU快速下发(SuperFastLoad)机制扩展到ReduceScatter和AlltoallV。10文件+215行，未被revert。成功的关键在于:

1. 遵循已有的扩展模式(而非发明新机制): 在TryFastCcuLaunch中增加opType判断
2. 为新算子设计专用的ccuParamsMappingKey: AlltoallV使用`{reduceOp, sendType, 0}`(第三个字段为0)，与AllToAll区分
3. 特殊参数格式有专用填充函数: AlltoallV的变长args通过`FillAllToAllVArgs`处理，跳过index=2(token info)逐个填入
4. 特殊拓扑有额外后处理: ReduceScatter的2-die mesh需要额外执行inline reduce(`HrtReduceAsync`)
5. CachedCCUParams缓存机制: 首次执行建立缓存(参数序列化)，后续直接复用——需要正确传递新增的insType字段的move语义

CCU优化的操作规程:
- 修改全局常量(如loop count)前: 用Grep统计所有引用该常量的位置，评估影响面
- 提交前: 除UT外还需要真实硬件跑完全量ST(不要在周末/假期前提交)
- 准备好回退预案: 如果可能，用环境变量或配置项控制新值，而非直接修改常量

#### 第5步: 应用低风险微优化手段

如果不涉及CCU参数修改，有一类"最低风险最高收益"的微优化手段值得优先尝试。

Case OPT-3(hcomm:a98eb47a, improve aicpu performance)在15文件中做了三类改动，总计+129/-130行(几乎纯替换)，未被revert:

手段1 — UNLIKELY宏标注错误检查分支(约30处):
```
// before
if (ptr == nullptr) { ... }
// after
if (UNLIKELY(ptr == nullptr)) { ... }
```
覆盖ins_to_sqe_rule_v82.cc(约20处)、rtsq_a5.cc(约8处)等AICPU热路径文件。编译器默认按50%概率预测分支，UNLIKELY提示编译器将这些错误检查分支移出指令缓存热区。

手段2 — 早退出分支重排:
```
// before: 少数情况(达到128)在if内处理
if (pendingSqeCnt == perLaunchSqeCnt) { LaunchTask(); }
// after: 多数情况(未达到128)早退出
if (pendingSqeCnt != perLaunchSqeCnt) { return; }
LaunchTask();
```
将常见路径(未达阈值)放在函数开头return，少见路径(达到阈值需要launch)放在函数末尾。

手段3 — 附带可观测性改进:
在rtsq_a5.cc的CopyLocBufToSq中增加sqId_、streamId_、sqHead_的日志输出。"优化附带可观测性改进"是良好实践——优化后如果出现异常，增强的日志可以更快定位问题。

适用条件: UNLIKELY只在热循环中有显著效果。对偶尔执行的初始化代码(如HcclCommInit的28步序列)无意义。确认目标代码确实在热路径上(如CCU指令解释循环、RTSQ任务下发循环)后再应用。

#### 第6步: 验证优化效果并准备回退预案

大规模优化(如OPT-1的28文件)需要充分的多平台测试和多场景验证。OPT-1的MR描述明确列出了: A2单机/A2双机/A3单机/A3双机/增量建链——5个测试维度。28文件的大规模变更没有被revert，证明测试覆盖是充分的。

对比: RV-9的1行CCU常量变更，UT通过后直接合入，6小时后被revert。差距不在于代码量，而在于验证充分性——设备端参数的影响面远超host端编排的影响面。

回退预案的设计取决于优化的层级:
- Selector/Executor层优化(OPT-1类型): 可以用环境变量开关控制新旧路径。OPT-1中的`isGroupBigCount()`阈值(128KB)可以改为可配置
- CCU参数调优(RV-9类型): 要么用环境变量控制常量值，要么在dev分支先验证2周再合入main
- 微优化(OPT-3类型): UNLIKELY宏不改语义，不需要回退预案; 但分支重排(手段2)如果改变了语义需要谨慎

### 常见陷阱

陷阱1 — CCU参数"一行代码"的验证陷阱(RV-9, hcomm:72cdf80e / RV-8, hcomm:0c3be05f):
`CCU_MS_DEFAULT_LOOP_COUNT`从64改为128，仅1行常量变更+UT更新。当日合入当日revert。17天后re-revert恢复，证明128是正确值——问题不在代码正确性而在验证周期不足。设备端参数修改即使代码量极小(1行)，影响面可覆盖所有CCU调度路径。预防: 全局CCU常量变更前先在dev分支验证至少2周(完整硬件回归)。

陷阱2 — 分层pipeline的多个独立bug(BF-4, hcomm:9bcb1bdc):
CP算法修复中有三个独立bug(初始化遗漏/层间顺序颠倒/冗余SDMA路径)在同一个commit中修复。风险在于: 如果其中一个fix引入了新bug，难以区分是哪个fix导致的。操作规则: "同时修复多个独立bug"时，每个fix需要独立的单元测试。BF-4新增了181行测试用例，用PCIe链路场景验证修复效果——这是正确做法。

陷阱3 — 大规模编排优化的阈值选择(OPT-1, hcomm:fd9d488c):
128KB是GroupSendRecv中Big/Small的分界阈值。阈值选择需要在编排开销(每次Send/Recv的communicator调度成本)和传输时间(数据量越大越应该并发)之间权衡。Big/Small双路径增加了维护成本: 大数据用多stream分组+流水线同步(MainPostSubWait/MainWaitSubPost)，小数据用单从流直接编排(ProcessDataSliceSmall)。两条路径的bug需要分别排查。决策: 如果优化收益不显著(如只快了5%)，单路径方案优于双路径方案——维护成本是长期的。

---

## 11.6 场景: "做一次跨模块重构"

### 场景简述

你需要对hccl/hcomm代码库做结构性调整: 可能是移动代码到更合适的位置、拆分巨型文件、调整跨仓边界、或统一接口命名。与功能开发不同，重构的目标是"不改变行为，改善结构"。核心挑战在于: hccl和hcomm是独立编译的两个仓库(跨仓依赖通过弱符号链接)、legacy/next双路径共存(修改一个接口可能需要双路径都改)、公共头文件是变更放大器(改一个.h引发全仓编译)。

### 前置知识

- Phase 4 (phase4-contrastive-analysis.md): hccl/hcomm跨仓接口边界、弱符号机制、编译依赖方向(hccl → hcomm)
- Phase 1 (phase1-hccl-git-history.md / phase1-hcomm-git-history.md): 仅3个重构commit，处于功能快速叠加期; alg_param.h是变更放大器(14次修改)
- Phase 0 (phase0-architecture-map.md): legacy层占修改40%，仍是最活跃的层; V1/V2双架构共存
- 跨仓提交不对称: hccl改1-4文件 vs hcomm改20-47文件(约1:10)

### 决策清单

#### 第1步: 评估影响范围 — Grep统计所有调用点

重构的第一步不是写代码，而是精确量化影响范围。

Case RF-1(hccl-dev:c696e52f + hcomm-dev:ba56e500, 调整算子入口到算子仓)展示了一个hccl/hcomm仓之间最大规模的边界调整: hccl-dev 9文件+368行, hcomm-dev 109文件+2484/-2601行。其中109文件中约80%是机械性重命名(函数名+日志字符串)——用sed/grep批量替换即可。真正的架构变更集中在5个文件。

在开始重构之前需要回答的量化问题:
- `grep -rn "旧函数名"` 在多少个文件中出现? 出现次数?
- 这些文件分布在哪些层(algorithm/framework/platform/legacy)? 是否跨仓?
- 有没有test/st/ut文件需要同步更新?
- 是否涉及头文件(.h)? 如果是，修改这个头文件会触发多少文件重新编译?

Phase 1中记录的教训: alg_param.h被修改了14次，每次修改都引发联动编译。如果你的重构涉及alg_param.h或类似的公共头文件，需要评估编译时间的增加对CI流水线的影响。

#### 第2步: 制定分步执行策略 — 基础设施先行

"一次性全量提交"是大规模重构被revert的头号原因。

Case RV-1(hcomm-dev:1844b823, ars code revert)是最典型的反面教材: 51文件的新算法次日被revert，6天后以几乎完全相同的代码(仅差2行)重新引入。revert的原因不是代码错误，而是合入时机——ARS算法依赖device_capacity.cc中新增的带宽常量和topo_matcher中的ARS拓扑判断，这些基础设施变更与其他并行开发的MR冲突。

正确的分步提交顺序:
1. 基础设施/依赖变更: device_capacity.cc中的能力常量、topo_matcher中的拓扑判断函数、公共头文件中的新类型定义。这一步应该独立为先导commit，等CI验证通过后再继续
2. 核心逻辑: executor类、算法实现、Op层入口
3. 注册和联动: selector中的分支注册、CMakeLists.txt更新
4. 测试: UT/ST文件

RV-1的教训凝练为一句话: "代码正确但时机不对"是大规模变更被revert的最常见原因。分步提交可以让每一步都在稳定的基础上进行。

#### 第3步: 处理跨仓接口变更 — thin wrapper + Inner后缀

Case RF-1展示了跨仓边界调整的最小侵入方案:

hcomm-dev侧(先提交): 将所有对外API函数重命名为XXXInner。例如`HcclBroadcast`改为`HcclBroadcastInner`。109文件的改动虽然多但都是机械性重命名(sed替换函数名+日志字符串)，风险可控。

hccl-dev侧(后提交): 新增`hccl_collective_op.cc`(91行)，每个API函数只是一行转发:
```
HcclResult HcclBroadcast(...) {
    return HcclBroadcastInner(buf, count, dataType, root, comm, stream);
}
```

关键的提交顺序: hcomm-dev在17:23提交(export Inner函数)，hccl-dev在17:34提交(引用Inner函数)，间隔11分钟。跨仓编译依赖决定了合入顺序 —— hccl编译时需要链接hcomm的符号，所以hcomm必须先export新函数，hccl才能引用。

这个方案的优点: 业务逻辑完全不动(校验、日志、调度全留在hcomm的Inner函数中)，hccl侧只是ABI入口。两仓的变更可以分别review、分别测试。

#### 第4步: 处理V1/V2双路径 — 双路径实现 + 双路径测试

Phase 0记录了hcomm framework层有V1(legacy-compatible)和V2(IndependentOp，910_95专用)双架构共存。在legacy/next双路径共存期间，新增或修改API必须双路径都处理。

Case CL-2(hcomm:c5709dd4, HcclEngineCtxDestroy API)展示了双路径的成本: 29文件+808行，其中约一半是测试。关键决策:
- next路径: 新增`engine_ctxs/`目录，包含engine_ctxs.cc/h(100行)，实现CreateCommEngineCtx/DestroyCommEngineCtx/GetCommEngineCtx的完整生命周期
- legacy路径: 修改StreamLite构造函数签名(新增bool参数)
- 测试: 两个独立UT文件 —— ut_HcclEngineCtxDestroy_API_test.cc(V1路径)和ut_HcclEngineCtxDestroy_API_V2_test.cc(V2路径)

操作规则: 如果只能择一，先实现legacy路径——Phase 0和Phase 1都确认legacy仍占修改40%，是当前主力。但"只做legacy不做next"是技术债，会在后续迁移时爆发。正确做法是每个新API同时在两条路径实现，并为每条路径写独立UT。

Case CL-1(hcomm:d0d5cce1, multi process per rank)进一步佐证了legacy层是现实: 30文件全在legacy/下，新增动态端口管理、子通信域端口继承等非平凡功能。legacy层不是"只维护不开发"——它仍然是新功能的着陆场。

#### 第5步: 处理legacy层 — 仍是功能开发场

Phase 0记录了legacy占修改40%。CL-1(30文件全在legacy/)证明legacy层不只是维护——它仍在承载新功能开发。重构时如果完全忽略legacy层，会导致两个后果:

1. 重构不完整: 用户代码仍走legacy路径，新接口在legacy路径上不可用
2. 后续功能开发时被迫在legacy和next两个不一致的代码结构上同时工作

但也不应该在legacy层做大规模重构——legacy层的高bug密度(Phase 1: communicator_impl.cc有47次修改，其中60.7%是bugfix)意味着大改动的回归风险极高。推荐策略: 在legacy层做最小适配(函数重命名/参数透传)，在next层做结构性改进。

#### 第6步: 验证回归 — 行数持平是纯重构的健康指标

Case RF-3(hcomm-dev:e10d1002, Split huge source files)展示了纯重构的健康指标: 8文件+659/-631行。行数基本持平说明这是纯移动+微调，没有夹带功能修改。

这是纯重构commit的自检规则: 如果你的重构commit的新增行数显著超过删除行数(比如+2000/-500)，大概率不是纯重构，而是夹带了功能修改——应该拆分为独立commit。

RF-3的操作模式 —— "extract + redirect":
1. 新增.c/.h文件承接被抽出的函数(rs_drv_socket.c/h承接rs_socket.c中的驱动层函数)
2. 原文件的调用方新增include(rs.c和rs_ping_roce.c新增`#include "rs_drv_socket.h"`)
3. 确保编译依赖无破坏
4. 删除原文件中被抽出的部分

这个模式适用于C文件拆分。对于C++类的拆分，类似但需要额外处理头文件中的前向声明。

### 常见陷阱

陷阱1 — 代码正确但时机不对(RV-1, hcomm-dev:1844b823):
51文件新算法次日revert，6天后原样重新引入(仅差2行)。根因: ARS算法依赖device_capacity和topo_matcher的基础设施变更，这些变更未先导合入就直接全量提交。预防: 基础设施变更独立为先导commit，等CI验证通过(至少1-2天)后再合入依赖它的代码。

陷阱2 — cmake文件重命名引发CI构建失败(RV-3, hcomm:753ba8c2):
utils.cmake重命名为hcomm_utils.cmake，但有其他地方引用了旧文件名。1小时内被CI捕获并revert。次日修复重新引入(+18行修补)。预防: cmake文件重命名时，执行`grep -r "旧文件名" .`全量搜索所有引用点(包括其他分支和并行MR中的代码)。分步提交时，重命名应放在第一步。

陷阱3 — 跨仓提交的顺序和间隔(RF-1, hccl-dev:c696e52f + hcomm-dev:ba56e500):
两仓间隔11分钟，hcomm先提交(export Inner函数)再hccl提交(引用Inner函数)。如果顺序颠倒(hccl先提交)，hccl的编译会因找不到Inner函数符号而失败。操作规则: 跨仓变更中，被依赖方(hcomm)先提交，依赖方(hccl)后提交。间隔应尽量短(本例11分钟)但不能为零(需要等hcomm CI通过)。

陷阱4 — 跨仓API变更需要多轮迭代(XR-4, hccl-dev:de127490系列):
API Changes经历了4轮: 首次引入(23:37) → 22分钟后revert(23:58) → 1分钟后重试(23:59) → 2天后最终版(12-01)。跨仓API变更平均需要2-3轮迭代才能稳定。预防: 不要期望跨仓API变更一次做对。首次变更尽量小(只改必须的签名)，不要搭便车夹带其他修改。提交前准备好revert分支。

陷阱5 — 公共头文件是变更放大器(Phase 1):
alg_param.h被修改14次，每次修改引发全仓联动编译。如果重构涉及公共头文件中的类型定义(如新增/修改结构体字段)，所有include该头文件的.cc都需要重新编译。评估手段: `grep -rl "alg_param.h" src/` 查看引用文件数量。如果超过50个文件，考虑: (1)能否将新增定义放在独立的新头文件中; (2)能否用前向声明减少编译依赖。

---

# Phase 11 Part C: 场景化开发指南(场景11.7-11.10)

日期: 2026-03-16
覆盖场景: 修复资源泄漏 / 修改CCU/AICPU设备侧代码 / 修改通信协议 / Legacy迁移到Next

---

## 11.7 场景: "修复资源泄漏"

### 场景概述

通信框架管理着大量硬件资源: Notify、Channel、CommMem(通信内存)、对称内存(Symmetric Memory)、Stream、Event。这些资源的生命周期横跨Host和Device两个执行域，且多个通信域/模型可能共享同一资源。资源泄漏在通信框架中的危害远超普通应用——一次泄漏可能导致后续所有通信域创建失败(LC-4展示了stream耗尽的连锁反应)，或者在长时间训练中逐渐恶化直到OOM。

修复资源泄漏不只是"找到没释放的地方加一行free"。Phase 10的案例反复证明: 资源管理的本质挑战在于(1)跨Host-Device边界的所有权不清晰，(2)引用计数的初始版本几乎必然有遗漏，(3)异常路径(CHK_RET失败/retry/快恢)是泄漏高发区。

### 前置知识

- Phase 5.2: HcclCommInit 28步初始化序列, V2路径独立初始化流程, 资源创建/销毁的完整链路
- Phase 2.2: CommMemMgr(区间树+引用计数), CommEngineResMgr(线程和Notify资源管理), ChannelManager通信通道完整生命周期(Create->Register->MapTag->Destroy)
- Phase 2.1: new(std::nothrow)+CHK_PTR_NULL堆分配模式; 100% CHK_PTR_NULL/CHK_RET覆盖

### 决策清单

#### 步骤1: 确认泄漏资源的类型和所在层

通信框架的资源分为六大类，每类有不同的管理机制和泄漏模式:

| 资源类型 | 管理者 | 泄漏检测线索 | 典型案例 |
|----------|--------|-------------|----------|
| Stream | OrderLaunchManager / HcclComm | "exhausted stream resources"错误码 | LC-4 |
| Notify | CommEngineResMgr / ThreadMgr | 心跳超时 / Notify pool exhaustion | PL-3 |
| Channel | ChannelManager | 建链失败 / 通信超时 | Phase 2.2 |
| CommMem | CommMemMgr(引用计数) | OOM / refCount不归零 | FT-7 |
| 对称内存 | SymmetricMemory(PaMappingInfo) | VA空间碎片 / 映射残留 | FT-7 |
| Thread handle | ThreadMgr(g_ThreadMap+g_ThreadD2HMap) | D2H映射残留 / use-after-free | BF-8 |

判断方法: 从错误现象(超时?OOM?崩溃?)反推资源类型。LC-4展示了stream耗尽的典型症状——aclgraph launch返回资源不足错误，但根因是per-model stream分配策略不合理。关键认知: 硬件资源(Stream/Event/Notify)是有限的，"每个实例分配独立资源"的设计模式在通信框架中往往不可行，因为实例数=model数x rank数(LC-4, hcomm:c5443da0)。

#### 步骤2: 追踪资源的完整生命周期(创建点->使用点->销毁点)

拿到泄漏资源后，必须画出完整的生命周期链路。这不是简单地grep malloc/free，而是需要理解所有权(ownership)在各层之间的传递。

LC-2(hcomm:3315532a, memDesc生命周期管理)是所有权分析的典范:
- 问题: MemoryExport生成的memDesc(序列化二进制数据)的所有权在调用者还是产生者?
- 原始设计: 调用者提供buffer，MemoryExport做memcpy到buffer中——调用者负责生命周期，但调用者必须知道buffer大小(TRANSPORT_EMD_ESC_SIZE)，这是知识耦合
- 修复后: memDesc存储在buffer对象本身的Desc成员中(move语义)——产生者管理生命周期
- 教训: 序列化产物(memDesc/config/metadata)的所有权应属于产生它的对象本身

追踪方法:
1. 用Grep搜索资源创建点(new/alloc/create/acquire等关键词)
2. 用Grep搜索资源销毁点(delete/free/destroy/release等关键词)
3. 在两者之间画出所有权传递路径——特别注意函数参数是raw pointer还是unique_ptr/shared_ptr
4. 检查每个CHK_RET/CHK_PTR_NULL的early return路径上是否有对应的cleanup

特别注意LC-2的另一个教训: RegisterMemory重复注册(返回HCCL_E_AGAIN)时也必须设置有效的memHandle。否则调用者拿不到handle就无法操作已注册的资源——这是一个容易被忽视的非正常路径。

#### 步骤3: 检查引用计数的正确性

FT-7(hcomm:4c779e13, symmetric memory)展示了引用计数管理的完整范例:
- PaMappingInfo.refCount管理每个物理内存块的引用计数
- 多个SymmetricWindow可以共享同一块物理内存
- refCount降为0时才执行Unmap和Release VA

但FT-7也同时展示了引用计数的陷阱: 引入后不久即需修复(hcomm:a9fea200修复资源释放不完全)。3930行的新增内存管理子系统中，初始版本几乎必然有资源释放遗漏。

引用计数检查清单:
- 每个refCount++是否都有对应的refCount--路径?
- refCount==0时的释放逻辑是否在所有代码路径上都能触发?
- 异常路径(CHK_RET失败、timeout、retry)中refCount是否正确递减?
- 多线程环境下refCount操作是否是原子的? BF-2(hcomm:1535b1c4)展示了通信框架的并发控制应优先使用原子操作(CAS+spinloop)而非mutex

建议: 写完资源管理核心逻辑后，先写资源泄漏检测的单元测试(FT-7的1245行UT占总代码量的32%，这个比例是本仓库的标杆)。

#### 步骤4: 审查异常路径的资源释放

CHK_RET宏在hcomm中有9层传播(Phase 5.3)。每层的early return都可能跳过后续的资源释放代码。

BF-3(hcomm:994390df, Step快恢失败)展示了异常路径的两类问题:
1. 状态机初始化遗漏: RegisterAgentRetryMachine/RegisterServerRetryMachine中创建retryCtx后状态为未定义——必须在注册时显式调用SetRetryState设为RUNNING。不能依赖默认值或后续流程隐式初始化
2. 环境判断缺失: CheckExitWaitResumeState在非AICPU环境下也被调用导致状态检查失败——设备侧路径的容错逻辑必须有环境判断门控: `if (GetAicpuUnfoldFlag() || GetAicpuCommEngine())`

异常路径审查步骤:
- 找出资源创建和释放之间的所有CHK_RET调用
- 对每个CHK_RET: 如果返回错误，此前已分配的资源是否被释放?
- 特别检查retry路径——BF-1(hcomm:75659d24)展示了retry FSM中"先清理物理资源，后清除异常标记"的顺序约束

#### 步骤5: 评估RAII vs 手动管理

不是所有资源都能用RAII管理。BF-8(hcomm:fa61b9fc, Thread API bugfix)明确展示了这个限制:

g_ThreadD2HMap中的thread handle是device侧指针，不能用C++ RAII管理——销毁需要通过kernel launch在device侧执行。跨Host-Device边界的资源需要特殊处理:
- Host侧资源(std::mutex, std::vector等): 可以用RAII/智能指针
- Device侧资源(thread handle, notify, stream): 必须显式管理(Create/Destroy API对)
- 跨域映射(g_ThreadD2HMap): 销毁时必须完整清理所有映射条目，残留的映射条目是use-after-free的根源(BF-8的核心教训)

BF-8同时展示了对外C API层的设计原则: thin wrapper + 内部模块分离。HcommThreadAlloc/Free作为extern "C" API只做参数适配，核心逻辑(ValidateThreadParams, CreateAndInitThreads, SaveThreads, FreeThreadHandlesLocked)放在内部的thread.cc中。这使得泄漏修复只需要修改内部模块，不影响ABI。

#### 步骤6: 并发安全检查

资源释放的并发安全是最后也是最重要的检查。

BF-2(hcomm:1535b1c4)的教训: ReadWriteLockBase从mutex+condition_variable改为纯atomic实现。通信框架天然多线程(每个通信域有独立的executor线程)，资源释放时的锁粒度直接影响性能和正确性。

对比同一仓库中两个并发修复的成败:
- 失败: c546d353在高层API(HcclAllocComResourceByTiling)加全局std::lock_guard<std::mutex>——锁粒度太粗，导致整个资源分配序列化，被revert
- 成功: 1535b1c4在底层(ReadWriteLockBase)替换锁实现——保持读写分离，多读者并发，读者不被写者饥饿(WAITING_BIT机制)

PL-3(hcomm-dev:0e82e543)展示了跨引擎资源映射的并发保护模式: threadHandleOthersToCpu_映射表 + threadMapMutex_互斥锁。这是处理跨通信引擎句柄映射的标准做法。

### 常见陷阱

陷阱1 -- 新增内存管理子系统的初始版本必有泄漏:
FT-7(hcomm:4c779e13, 3930行新增) + a9fea200修复。Phase 8.8数据显示设备侧BUGFIX密度43.5%显著高于Host侧30.3%。设备侧资源管理是bug密集区——不要期望初始版本完美，而是在设计时就预留泄漏检测机制(UT+日志)。

陷阱2 -- 残留映射条目是use-after-free的根源:
BF-8(hcomm:fa61b9fc)。Thread handle的D2H映射在销毁时如果没有遍历g_ThreadD2HMap删除所有指向被销毁handle的映射，后续代码可能通过残留映射访问已释放的device内存。修复方法: 新增FreeThreadHandlesLocked函数专门清理映射。

陷阱3 -- per-model/per-instance资源分配在通信框架中不可行:
LC-4(hcomm:c5443da0)。aclgraph模式下per-model分配独立stream导致资源耗尽。实例数=model数x rank数，硬件资源无法承受。修复: 用全局唯一的order stream + Event对替代per-model stream。

陷阱4 -- 重复注册路径必须返回有效handle:
LC-2(hcomm:3315532a)。RegisterMemory返回HCCL_E_AGAIN(表示"已经注册过")时，原代码没有设置memHandle就直接return——调用者拿不到handle就无法操作已注册的资源。修复: `*memHandle = static_cast<void*>(localRegisteredBuffer.get());` 然后再return HCCL_E_AGAIN。

陷阱5 -- 跨Host-Device边界传递flag控制行为是脆弱的:
LC-3(hcomm:9167c142)。host侧通过needUpdateRes flag告诉device侧"是否需要刷新资源"，但在aclgraph replay场景下host无法准确判断。修复: 将判断权下放到device侧(idempotent check模式)，用loadedOpSet在device侧记录已加载的tag。更好的模式: 使用方自治判断，而非依赖跨域传递的flag。

---

## 11.8 场景: "修改CCU/AICPU设备侧代码"

### 场景概述

hccl/hcomm中的设备侧代码运行在NPU上的CCU(Communication Control Unit)或AICPU(AI CPU)上。Phase 8.8的数据给出了明确的风险画像: 纯设备侧BUGFIX密度43.5%，显著高于Host侧的30.3%(差值13.2个百分点)。其中hccl:template/ccu的BUGFIX密度高达44.4%，且每commit平均影响34个文件——这是"改一个地方波及全部模板"的广播效应。

设备侧代码的危险之处在于: bug的表现(超时、死锁、结果错误)与根因(参数错误、地址时机、控制流语义差异)之间的距离极远。BF-5修改1个数字(GetThreadNum从1改为2)就修复了死锁，BF-9把几行地址设置代码从循环内移到循环外就修复了结果错误。定位成本远大于修复成本。

### 前置知识

- Phase 2.3: AIV算法模板体系(AlgorithmAivBase, HCCL_KERNEL_DISPATCH_XXX宏), MTE2/MTE3引擎, 同步协议(WaitLocal->DataCopy->SignalRemote->WaitRemote)
- Phase 3: CCU 5路分派Selector(CCU MS / CCU Schedule / AICPU+AIV / AICPU / Legacy), template目录结构
- Phase 1: CCU template是bug集中区, loopnum fix-revert-revert循环(3 commits跨17天)
- Phase 8.8: 设备侧BUGFIX密度43.5% vs Host侧30.3%; hccl:template/ccu的BF密度44.4%且每commit影响34文件

### 决策清单

#### 步骤1: 判断变更类型

设备侧的变更分为三个风险等级完全不同的类型:

类型A -- microcode参数调优(loopnum/threadnum等):
代码改动量极小(通常1-3行)，但影响面可能覆盖所有CCU调度路径。RV-9(hcomm:72cdf80e)是教科书案例: CCU_MS_DEFAULT_LOOP_COUNT从64改为128，仅1行常量变更，却引发了3个commit(fix->revert->re-revert)跨越17天的验证周期。

类型B -- kernel逻辑修改(地址计算/控制流/同步):
中等代码量(10-50行)，但需要深入理解CCU/AICPU的执行模型。BF-9(hccl:80a70729)将地址设置从CCU_WHILE循环内移到循环外——这在普通C++中只是代码位置调整，但在CCU微码中是正确性与错误的分界线。

类型C -- 新kernel开发(新算法/新拓扑/新芯片适配):
大代码量(数百到数千行)，需要遵循严格的文件结构约定。FT-3(hcomm:fa8ed355, double die算法)为4种算子新增12个文件(context+instruction+template各4组)，总计+3057行。

每种类型的验证策略完全不同(见步骤6)。

#### 步骤2: 确保Host-Device参数一致性

host侧的Selector/Executor设置参数(topo信息、地址、线程数等)，通过序列化传递给device侧kernel。参数不一致是设备侧bug的首要根因。

BF-9(hccl:80a70729, scatter_mesh1d地址设置时机)展示了这个问题的具体表现:
- Host侧op_common.cc中param.hcclComm的赋值原来在HcclExecOp中(太晚)，修复后移到Selector阶段(更早)
- CCU kernel中DoScatter函数内的地址设置循环(for curId = 0 to rankSize_)原来在CCU_WHILE循环内部，修复后移到DoRepeatScatter的CCU_WHILE循环之前

核心原则: CCU微码中，初始化代码(地址设置/参数计算)必须放在CCU_WHILE循环外部。CCU_WHILE内部应只包含每轮需要变化的增量逻辑。这是CCU编程模型与普通C++的关键区别——CCU控制流(CCU_WHILE/CCU_IF)是CCU控制指令而非普通的while/if(BF-9的核心教训)。

检查清单:
- host侧Selector设置的参数是否在序列化前完成? (BF-9: param.hcclComm需在Selector阶段赋值)
- 序列化结构体(AlgResourceCtx等)的sizeof计算是否正确? (BF-7: 手动偏移计算遗漏两个字段)
- device侧反序列化后的参数值是否与host侧一致? (FT-1: TopoInfoWithNetLayerDetails的尾部追加序列化策略)

#### 步骤3: 选择执行模式(CCU vs AICPU)

hccl使用5路分派Selector(Phase 3):
1. CCU MS(Multi-Stream): CCU硬件级流水线编排，性能最佳
2. CCU Schedule: CCU调度模式
3. AICPU+AIV: AICPU控制+AIV计算
4. AICPU: 纯AICPU执行
5. Legacy: 旧路径

XR-2(hccl-dev:0ca13cd6 + hcomm-dev:79e73f3c, scatter switch to aicpu)展示了执行模式切换的标准做法:
- 跨仓原子提交(18秒间隔)
- hccl侧是"减法"(删除旧的kernel加载逻辑-99行，简化为ASCEND_HOME_PATH直接获取)
- hcomm侧是"调整"(适配新的通信器配置-88行)
- 非对称但互补——hccl改调度逻辑，hcomm改实现逻辑

执行模式选择的判断标准:
- CCU: 需要硬件级流水线编排、多rank通信同步紧密耦合的场景
- AICPU: 控制流复杂(多分支)、需要host侧灵活交互的场景
- 如果一个算法从CCU切换到AICPU(或反向)，两仓必须原子性提交

#### 步骤4: 理解同步模型

设备侧有两个层次的同步:
- Kernel内部同步: SyncAll(CCU内部)、WaitLocal/SignalRemote(AIV内部)
- Kernel间同步: Notify(Send->Wait->Release三轮握手)

PL-6(hccl:89a09db2 + hcomm:d813063d, host&device sync)展示了从裸ACL Notify迁移到Thread Export体系的完整过程:
- 旧方案: 直接调用aclrtRecordNotify/aclrtWaitAndResetNotify，通过thread_local全局数组g_notifies_host_with_device[2]管理两个控制Notify
- 新方案: HcclThreadAcquireWithStream -> HcclThreadExportToCommEngine -> HcommThreadNotifyRecordOnThread/WaitOnThread
- 迁移原因: 裸ACL调用绕过了hcomm的Thread资源管理体系，导致资源泄漏风险和无法被框架统一追踪

教训: 跨Host-Device边界的同步机制必须纳入框架的资源管理体系(Thread/Notify/Stream)。裸ACL调用虽然更快实现，但会导致资源管理失控。

PL-3(hcomm-dev:0e82e543, scatter notify&wait)展示了跨通信引擎导出的具体实现:
- ThreadExportToCommEngineCpu: AICPU->CPU方向，通过threadHandleOthersToCpu_映射表查找
- ThreadExportToCommEngineAicpu: CPU->AICPU方向，需调用AicpuLaunchMgr::ThreadKernelLaunch做设备侧展开
- CpuTsThread补全GetUniqueId实现(从不支持到完整实现)

#### 步骤5: 添加DFX可观测性

设备侧代码的调试极度依赖日志和诊断信息。OPT-3(hcomm:a98eb47a, improve aicpu performance)在做性能优化的同时增加了rtsq_a5.cc CopyLocBufToSq中的sqId_/streamId_/sqHead_输出——这是"优化附带可观测性改进"的良好实践。

设备侧DFX要点:
- task_exception路径是诊断入口(Phase 5.3: CQE异步错误检测)
- CCU/AICPU kernel中的关键变量(notify id、buffer地址、slice大小)必须可通过DFX导出
- FastCcuLaunchSaveDfxTaskInfo需要记录remoteRankId(OPT-4: 用GetMyRank()作为reduce任务的remoteRankId)

#### 步骤6: 选择验证策略

不同变更类型需要完全不同的验证策略:

类型A(参数调优):
BF-5(hccl:992fd5b, GetThreadNum 1->2)展示了CCU参数错误的症状特征——死锁/超时与根因(参数错误)距离很远。新增CCU template时必须对每个GetXxxNum方法做单元验证。RV-9展示了CCU参数的完整验证周期是17天而非6小时——UT通过不代表硬件验证通过。建议: CCU参数应考虑用环境变量或配置项控制以便回退(而非hardcode常量)。

类型B(逻辑修改):
BF-6(hccl:b779699)展示了编译目标的陷阱——Host/Device双目标编译时，新增的.cc文件必须同时检查是否需要添加到两个CMakeLists.txt。AICPU_COMPILE宏必须保护Host-only API，否则链接时才发现符号缺失。

类型C(新kernel开发):
FT-3(hcomm:fa8ed355, double die算法)展示了严格的三层文件结构: ccu_context(参数准备) + ccu_instruction(微指令序列) + ccu_temp(模板组装)。新增CCU算法必须遵循此结构，且CcuInstType枚举值必须添加在末尾(而非插入中间)。

#### 步骤7: 准备回退预案

RV-9的三commit循环(fix 2/11 11:37 -> revert 2/11 17:39 -> re-revert 2/28 15:37)清楚证明: 设备侧变更被当日revert的概率远高于host侧变更。

回退预案要点:
- 参数类修改: 用环境变量/配置项控制(而非hardcode)，回退时改配置而非改代码
- kernel逻辑修改: 保留旧实现的条件编译路径(#ifdef/feature flag)
- 新kernel: 确保Selector中有fallback到旧算法的路径

### 常见陷阱

陷阱1 -- 1行CCU常量引发3个commit循环:
RV-9/RV-8(hcomm:72cdf80e / hcomm:0c3be05f)。CCU_MS_DEFAULT_LOOP_COUNT从64改为128。当日合入当日revert是硬件验证不足的典型信号。验证周期17天不是小时级——CCU参数修改即使代码量极小(1行)，其影响面也可能覆盖所有CCU调度路径。

陷阱2 -- CCU微码中地址设置必须在循环外:
BF-9(hccl:80a70729)。CCU_WHILE内部的CCU_IF(flag_ != 0)分支中的增量偏移逻辑依赖地址已在循环外正确初始化。在CCU_WHILE循环内部做初始化意味着每次迭代都重复初始化，与增量逻辑矛盾。CCU_IF/CCU_WHILE是CCU控制指令而非普通C++控制流——不要按C++思维组织CCU代码。

陷阱3 -- GetThreadNum从1改为2修复死锁:
BF-5(hccl:992fd5b)。CCU NHR 1D mem2mem模式需要2个线程(一个做计算/搬移，一个做通信同步)。仅改一个数字就修复了死锁。CCU template的参数看似微小但影响调度——每个GetXxxNum方法的返回值都是关键参数。

陷阱4 -- AICPU_COMPILE宏保护遗漏:
BF-6(hccl:b779699)。channel.cc中GetTopoTypeByLink调用了HcclRankGraphGetTopoInstsByLayer(Host-only API)。AICPU编译时没有保护，链接阶段才发现符号缺失。引用Host-only API的函数必须有`#ifndef AICPU_COMPILE`条件编译保护。

陷阱5 -- 设备侧BUGFIX密度远高于Host侧:
Phase 8.8数据: 纯设备侧BUGFIX密度43.5%(92个commit中40个是BUGFIX)，hccl:template/ccu的BF密度44.4%且每commit影响34文件。CCU修改是"低频高影响"——改动少但每次改动都有广播效应。

陷阱6 -- CCU算法必须遵循三层文件结构:
FT-3(hcomm:fa8ed355, double die算法)。新增CCU算法必须有: ccu_context_xxx(参数准备，地址计算/notify映射/slice分配) + ccu_instruction_xxx(微指令序列，DataCopy/WaitNotify/SignalNotify) + ccu_temp_xxx(继承AlgTemplateBase，组装context和instruction)。不遵循此结构的代码无法通过现有框架的dispatch机制调度。

---

## 11.9 场景: "修改通信协议(Notify/重执行/心跳)"

### 场景概述

Notify、重执行(Retry/OpRetry)、心跳(Heartbeat/HB Check)是hccl/hcomm的三大核心协议。Phase 8.9的数据显示: 重执行最活跃(21条commit)，Notify有16条，心跳有8条。这三个协议的共同特征是: 修改影响范围远超代码变更本身——一端的行为变化可能导致对端死锁、状态不一致、或集群级故障。

协议变更的核心困难在于: 你无法在单机上完整测试分布式协议的所有边界条件。Phase 8.9的重执行时间线清楚展示了这一点: 2025-12-12默认关闭retry(c303e62a) -> 12月下旬4天5个commit密集修复 -> 2月连续4个BUGFIX -> 直到75659d24仅改2行才真正修复多线程时序问题。从"基本能用"到"生产可靠"历经3个月。

### 前置知识

- Phase 2.1: Notify三轮握手协议(Send->Wait->Release)
- Phase 5.3: CHK_RET 9层传播链路, CQE异步错误检测机制
- Phase 8.9: Notify 16条commit时间线; 重执行21条commit时间线(最活跃); 心跳8条commit(经典三阶段)
- Phase 8.9: 重执行时间线关键转折——2025-12默认关闭retry -> 12月下旬密集修复 -> 2月连续4个BUGFIX

### 决策清单

#### 步骤1: 评估协议变更的范围——只改一端还是两端

通信协议天然是双边的。改一端而不改另一端可能导致协议不兼容。

PL-4(hcomm-dev:9ceead3b, topo detect retry)展示了正确的双边变更模式:
- Agent端新增: ConnectWithRetry(最多3次重试)，将总超时三等分
- Server端配合: Accept成功后立即Send确认消息(TOPO_EXCHANGE_CHECK_MESSAGE)
- 两端都改: Agent可以通过确认消息快速判断连接是否真正建立

如果只改Agent端(加retry但Server不发确认)，Agent重试时无法区分"Server忙"和"连接失败"，可能无限重试直到超时。

双边变更检查清单:
- 发送方行为变化(新增字段/新状态/新消息类型) -> 接收方是否能正确解析?
- 状态机新增状态 -> 对端状态机是否有对应的转换路径?
- 超时时间变化 -> 对端的等待时间是否匹配?
- 跨仓变更 -> hccl和hcomm是否需要原子提交? (PL-6: 同作者21秒内提交两仓是标准做法)

#### 步骤2: 为新功能添加运行时开关(默认关闭)

这是从PL-1/PL-2(HB check三阶段链)中提炼的最重要的教训。时间线:
- 阶段1: 随初始化代码引入HB check op inconsistent功能，默认开启
- 阶段2(RV-7, hcomm-dev:a37e6cf1, 2025-12-03): 整体Revert(7 files, +141/-249)
- 阶段3(PL-2, hcomm-dev:ebfcde5c, 2026-01-06): 用环境变量开关重新引入，默认关闭

PL-2相比被revert的原版做了三个关键改进:
1. 环境变量开关: EnvConfig.inconsistentCheckSwitch默认false，通过HCCL_DFS_CONFIG中的inconsistent_check:on/off控制。GetSendOpInfoList入口处直接`if (!GetExternalInconsistentCheckSwitch()) return;`短路
2. 帧大小动态选择: 心跳socket buffer大小根据开关选择`sizeof(HeartBeatFrameWithOpCheck) : sizeof(HeartBeatFrame)`
3. 错误上报路径: CommCheckOpInconsistentError仅在CQE检查通过后才执行

功能未被再次revert，证明"默认关闭+开关控制"策略有效。

开关设计检查清单:
- 开关默认值必须为"关闭"(opt-in而非opt-out)
- 在功能入口处用开关做短路判断(避免未开启时的任何性能开销)
- 开关控制的粒度: 运行时环境变量(推荐) > 编译时宏(不够灵活) > 重启时配置文件
- 心跳帧大小需根据开关动态调整(50ms间隔下帧越大越影响网络——RV-7的revert原因之一)

重执行协议同样遵循此模式: c303e62a(2025-12-12)将retry默认设为disabled，同时在12月19日和22日连续提交"开启retry时禁用X功能"的commit(af9d9a52/08870717)——说明retry与多种优化路径存在互斥关系，开启retry需要关闭部分优化。

#### 步骤3: 确保状态机的完整性

协议状态机的每个状态转换都必须显式处理。遗漏转换路径是协议bug的第二大根因(仅次于缺少开关)。

BF-3(hcomm:994390df, Step快恢失败)展示了两个状态机完整性问题:
1. 初始状态未设置: RegisterAgentRetryMachine和RegisterServerRetryMachine中创建retryCtx后，状态为未定义。修复: 注册时显式SetRetryState为RUNNING。规则: 状态机的初始状态必须在构造/注册时显式设置，不能依赖默认值
2. 环境门控缺失: CheckExitWaitResumeState在非AICPU环境下也被调用。修复: 添加`if (GetAicpuUnfoldFlag() || GetAicpuCommEngine())`条件

状态机检查清单:
- 每个状态是否都有明确的进入条件和退出条件?
- 每个状态转换是否在所有可能的错误路径上都被正确处理?
- 初始状态是否显式设置(而非依赖零初始化)?
- 非适用环境(如非AICPU)是否被门控?

#### 步骤4: 简化重执行路径

PL-5(hcomm-dev:28f18ab5, fix changelink for OpRetry WaitResume)展示了重执行协议简化的最佳实践:

原设计中，链路切换(changeLink)只在RDMA错误时触发(g_isRdmaError全局变量控制)。三处核心简化:
1. 删除g_isRdmaError全局变量: 消除多线程竞态风险(Phase 1已知并发控制薄弱)
2. 合并双路径为单路径: 原来`!isRdmaError`直接resume、`isRdmaError`走changeLink；合并为统一走changeLink
3. isChangedLink无条件设为true: 宁可多做恢复动作(changeLink)也不要漏做

这个修复净减25行代码，同时解决了正确性和并发安全两个问题。核心哲学: 重执行路径的错误分类(RDMA/SDMA/约束违反)不应影响恢复策略的骨干逻辑——安全的默认策略是"总是做最完整的恢复"。

#### 步骤5: 评估心跳帧的性能影响

心跳协议特有的约束: 帧大小敏感(50ms广播周期)。

RV-7(hcomm-dev:a37e6cf1, Revert HB check)的revert根因之一是HeartBeatFrame大小显著增大——从opInfoList[16]变为OpInfoTagQueueFrame(10个tag x 500个op)。心跳帧膨胀会:
- 增加网络带宽消耗(每50ms广播一次)
- 增加序列化/反序列化延迟
- 可能触发分片(帧超过MTU)

心跳帧变更检查清单:
- 新增数据结构后帧大小是否显著增大?
- 帧大小是否可根据功能开关动态选择? (PL-2: HeartBeatFrameWithOpCheck vs HeartBeatFrame)
- 序列化格式是否向后兼容? (新版本心跳帧能否被旧版本节点正确解析)

#### 步骤6: 处理多平台差异

不同芯片的协议行为可能不一致。AA-5(hcomm:4afb1c39, RoCE flush方案适配)展示了协议层的硬件适配——不同网卡的flush能力不同，需要检测网卡是否支持flush opcode。

#### 步骤7: 确保并发安全

通信协议天然涉及多线程(每个通信域有独立线程)。

PL-5的教训: 用全局变量(g_isRdmaError)在多线程环境中传递状态机决策是反模式。全局变量在多个通信域的executor线程间共享，任何线程的写入都可能影响其他线程的判断。

BF-1(hcomm:75659d24, 重执行多线程时序)是并发bug定位的典范:
- 34文件的feature commit(d17b1f3a)引入了间接的时序问题
- fix只有2行: 将DFX状态重置移到ResetOpRetryException之前；将ResetStreamCqeExceptionStatus移到ClearLocalBuff+UpdateSqStatus之后
- 共同逻辑: 先完成物理层面的清理(buffer/SQ/DFX状态)，再清除异常标记
- 教训: "先清物理资源再清异常标记"是设备侧FSM的通用原则

PL-3(hcomm-dev:0e82e543)展示了并发安全的正面做法: threadHandleOthersToCpu_映射表 + threadMapMutex_互斥锁保护跨引擎句柄映射。

### 常见陷阱

陷阱1 -- 核心基础设施新功能不可关闭则被整体Revert:
RV-7(hcomm-dev:a37e6cf1)。HB check op inconsistent功能没有运行时开关，出问题时只能Revert整个commit(7 files, +141/-249)。34天后ebfcde5c加开关重做才稳定。"introduce -> revert -> add-switch-redo"是hcomm中反复出现的三阶段模式(PL-1/PL-2 + retry默认关闭c303e62a)。

陷阱2 -- 用全局变量传递状态机决策:
PL-5(hcomm-dev:28f18ab5)。g_isRdmaError全局变量在多线程下有竞态风险。删除它并合并双路径为统一走changeLink，净减25行且修复并发问题。

陷阱3 -- 重执行路径2行语句重排修复多线程时序:
BF-1(hcomm:75659d24)。先清物理资源(ClearLocalBuff+UpdateSqStatus)再清异常标记(ResetStreamCqeExceptionStatus)。定位成本极高(从34文件feature commit中找到2行根因)。feature review时需特别审查retry/异常/并发路径是否被间接影响。

陷阱4 -- 重执行从"基本能用"到"生产可靠"需要3个月:
Phase 8.9 retry时间线: 2025-12-12默认关闭retry(c303e62a) -> 12月下旬4天5个commit密集修复(9ceead3b/fa053984/26f5813f/f597dd47) -> 2月连续4个BUGFIX(6b354394/e025b6c5/96087ffb/75659d24)。75659d24仅改2行但定位极难——竞态条件的修复往往代码量极小但定位成本极高。

陷阱5 -- 系统接口迁移必须adapter+动态fallback:
RV-5/RV-6(hcomm-dev:30e25e50 / hcomm-dev:1d8e2c14, RTS chain)。一次性从旧RTS接口切到新ACL接口失败后，RV-6引入了hrtGetPairDeviceLinkTypeRaw做动态dispatch: 先通过dlsym检测新接口可用性，可用调新接口，不可用fallback旧接口。这比条件编译更灵活，比一次性切换更安全。

陷阱6 -- 建链重试的总超时不变且非末次重试降日志级别:
PL-4(hcomm-dev:9ceead3b)。ConnectWithRetry将总超时三等分(timeout = GetExternalInputHcclLinkTimeOut() / AGENT_MAX_RETRY_TIME)。前2次重试通过SetErrToWarnSwitch(true)将ERROR降级为WARNING，避免误触告警系统。最后一次失败恢复ERROR级别。两个保护措施缺一不可。

---

## 11.10 场景: "将功能从legacy迁移到next/framework"

### 场景概述

Phase 8.7的数据揭示了一个与直觉相反的现实: legacy不是在衰退，而是在爆发。198条commit触及legacy/目录(20.5%占比)，从2026-02起legacy占hcomm总commit约40%。真正的功能迁移仅13/97条跨层commit(13%)，绝大多数跨层修改是"新功能同时在两条路径实现"而非"功能从legacy搬到next"。

更重要的发现: 40条legacy-only FEATURE commit证明新功能仍在legacy中开发。legacy/framework/以116条commit成为绝对修改中心，communicator_impl.cc以47次修改高居榜首。legacy目录名本身来自2026-01-31的orion->legacy大规模重命名(hcomm-dev:62ba4500, 2432文件)——团队将这部分代码从"产品名"重新定位为"遗留代码"，但实际活跃度丝毫未降。

因此，"将功能从legacy迁移到next"这个任务需要重新理解: 你不是在做"老代码的一次性搬家"，而是在一个双路径长期共存的架构中管理功能的渐进过渡。

### 前置知识

- Phase 0: legacy层结构——legacy/framework(116 commits) + legacy/unified_platform(69) + legacy/service(39) + legacy/common(23) + legacy/hccl_tbe_task(12) + legacy/graph_ctx_mgr(7) + legacy/operator(2,冻结) + legacy/local_build(2,冻结)
- Phase 1: legacy修改占40%; 仅3个重构commit; legacy/service是bug密度最高区域(603 file-touches)
- Phase 8.7: 198条commit触及legacy/; 真正功能迁移仅13条(13%); 88.9%的legacy commit无芯片关键词(legacy不是旧芯片专用); communicator_impl.cc 47次修改; 40条legacy-only FEATURE说明新功能仍在legacy开发

### 决策清单

#### 步骤1: 评估迁移时机——功能在legacy中稳定了吗?

这是最重要的前置判断。Phase 8.7的数据给出了明确的信号:
- 40条legacy-only FEATURE commit: 新功能仍在legacy中开发，不是只在做维护
- legacy/framework/的FEATURE占37.1%、BUGFIX占27.6%: 功能开发和bug修复齐飞
- communicator_impl.cc的47次修改中FEATURE(13)和BUGFIX(11)各占大量: 既是功能枢纽也是bug集中区

判断标准:
- 如果目标功能在legacy中最近3个月内仍有活跃的FEATURE/BUGFIX commit -> 迁移时机不成熟，优先在legacy中稳定
- 如果目标功能在legacy中已经3个月无修改且被标记为"stable" -> 可以考虑迁移
- 如果next路径已经有该功能的skeleton(骨架代码+TODO) -> 可以开始填充实现
- Phase 8.7: 17个TODO在framework/next/——next自身也未完成，迁移target可能也在变化

关键认知: 不要因为目录叫"legacy"就急于迁移。legacy不是在衰退——它是当前的主力开发场，迁移过早只会造成两条路径同时不稳定。

#### 步骤2: 确认功能等价性——next版本是否完全覆盖legacy?

CL-2(hcomm:c5709dd4, HcclEngineCtxDestroy API)展示了双路径共存期间新增API的成本:

问题: HcommBatchModeStart/End这对API在不调用时缺少独立的Destroy机制。需要新增HcclEngineCtxDestroy。

关键决策: 必须同时在legacy路径和next路径实现。
- next路径: 新增EngineCtxs类(engine_ctxs.cc/h, 100行)，实现CreateCommEngineCtx/DestroyCommEngineCtx/GetCommEngineCtx的完整生命周期
- legacy路径: 修改StreamLite构造函数签名(新增bool参数)来区分是否需要独立销毁
- 测试: 两个独立的UT文件(ut_HcclEngineCtxDestroy_API_test.cc和ut_HcclEngineCtxDestroy_API_V2_test.cc)分别测试V1和V2路径

29文件中约一半是测试——这展示了双路径的真实维护成本。

功能等价性检查清单:
- 新增API: 两条路径都提供实现 + 都有独立UT
- 如果只能择一: 先实现legacy路径(它是当前主力——Phase 8.7证实)
- 新API设计时考虑未来legacy退场后的简化路径: 避免把legacy的技术债带入next
- V1/V2路径分派: 通过IsNewDevice()/IsCommunicatorV2()判断走哪条路径(Phase 2.2)

#### 步骤3: 选择迁移策略——thin wrapper + Inner suffix

RF-1(hccl-dev:c696e52f + hcomm-dev:ba56e500, 调整算子入口到算子仓)展示了最小侵入的跨仓边界调整方案:

问题: 算子的外部API入口(HcclAllReduce等)定义在hcomm的op_base.cc中，但从架构角度应在hccl(算子仓)中定义。

方案: thin wrapper + Inner suffix
- hcomm侧: 将HcclBroadcast等函数全部重命名为HcclBroadcastInner(109文件批量改名)
- hccl侧: 新增hccl_collective_op.cc(91行)，每个API函数只是一行转发: `return HcclBroadcastInner(buf, count, dataType, root, comm, stream);`
- hccl侧: 新增inc/hccl.h(253行)，将公开头文件声明从hcomm移到hccl

为什么选这个方案:
- Inner函数保持原有的完整实现逻辑不动(校验/日志/调度全在hcomm中)
- hccl侧只提供ABI入口——一行转发的wrapper不可能引入bug
- 109个文件改动虽然多但都是机械性重命名(sed替换函数名+日志字符串)，风险可控
- 提交时间差11分钟(hcomm-dev先export Inner函数 -> hccl-dev引用)说明了合入顺序要求

迁移策略决策矩阵:
- 整个文件搬家: 风险极大(op_base.cc有2000+行且深度依赖hcomm内部头文件)——不推荐
- 只移头文件不移实现: 不够彻底，hcomm仍export全部实现
- 渐进式迁移(每次一个算子): 可行但效率低，中间状态API分散在两仓造成混淆
- thin wrapper + Inner suffix: 最小侵入，推荐

#### 步骤4: 理解legacy层的依赖结构

communicator_impl.cc是legacy层的绝对枢纽(47次修改，FEATURE 13次、BUGFIX 11次、INFRA 9次)。迁移legacy功能时，几乎不可能绕过这个文件。

CL-1(hcomm:d0d5cce1, multi process per rank)展示了legacy层新功能开发的典型模式:
- 30文件全在legacy/——没有任何next路径的修改
- 修改communicator_impl.cc: 子通信域创建后继承父通信域的端口映射
- 修改socket_manager: 构造函数签名变更(用deviceLogicId替代serverListenPort)
- 修改env_config.cc: 从配置文件读取设备端口范围

legacy层依赖热点(按修改频次排序):
1. communicator_impl.cc(47次): 通信域管理核心，所有通信域操作的枢纽
2. op_base_v2.cc(24次): V2算子执行入口
3. task_exception_handler.cpp(17次): 异常处理
4. orion_adapter_hccp.cc(17次): HCCP适配层
5. coll_service_ai_cpu_impl.cc(17次): AICPU服务层

迁移前必须理解这些文件之间的依赖关系。legacy/service是bug密度最高区域——迁移前必须理解bug模式(Phase 1: 603 file-touches)。

#### 步骤5: 处理测试覆盖

legacy路径的测试能否直接复用到next路径?

CL-2的实践: 两个独立的UT文件分别测试V1和V2路径。这说明测试不能简单复用——V1和V2的内部实现不同，即使外部API相同，测试用例的构造(mock对象/前置条件)也需要独立编写。

但测试用例的输入/预期输出通常可以共享——只是测试框架(mock/stub)需要为两条路径分别搭建。

#### 步骤6: 处理V1/V2分离

FT-2(hccl:12a2c84f + hcomm:dba4d87b, A5 aicpu communicator)展示了新芯片走framework/next、旧芯片走framework/device的分离策略:
- A5(910_95)走新框架: 在framework/next/coll_comms/communicator/aicpu/下新建独立模块(CollCommAicpu, CollCommAicpuMgr等4个新文件)
- 旧芯片走旧框架: 从巨型类HcclCommAicpu(5000+行)中删除63行A5专用代码
- hccl侧: kernel_launch.cc增加DevType判断——`if (param->deviceType != DevType::DEV_TYPE_910_95) { HcommAcquireComm(...); }`

为什么选"新建独立类+删除旧类中芯片专用代码"而非"在旧类中加if/else分支":
- HcclCommAicpu已经是bug密度最高的文件之一(Phase 1: 18次BUGFIX修改)
- 在5000+行的巨型类中添加更多分支只会使情况更糟
- 新建独立类意味着旧路径零修改——风险最小

### 常见陷阱

陷阱1 -- legacy/service是bug密度最高区域但也是最活跃模块:
Phase 8.7: legacy/service有1748 file-touches(最高)但只有39条commit，平均每commit触及45个legacy文件——高度耦合，牵一发动全身。迁移前必须理解这些文件之间的依赖关系，而非盲目搬代码。

陷阱2 -- communicator_impl.cc 47次修改:
Phase 8.7数据。这个文件是legacy层的枢纽，它的复杂度被严重低估。FEATURE(13)和BUGFIX(11)各占大量说明它同时是功能开发的核心入口和bug的集中区。修改它或从它迁出功能都需要格外谨慎。

陷阱3 -- 真正的功能迁移仅13%:
Phase 8.7: 97条跨legacy/next层的commit中，真正的迁移/重构仅13条。大多数跨层commit(84条)是"同步开发"(两边同时加特性)或"跨边界Bugfix"(bug根因跨越两层)。不要把跨层开发等同于迁移——它们是完全不同的工作。

陷阱4 -- 跨V1/V2边界的大功能同日引入同日回退:
FT-6(hcomm:ebb54874 -> hcomm:0c033874)。37文件+1140行的"coll comm & orion comm mix-running"功能合入仅10小时即被revert。三个致命问题: (1)op_base.cc大改560行; (2)代码中自标"临时方案"(lazy init的GetOpHcomInfo); (3)无feature flag。跨legacy/next的大功能必须分步合入: 先资源管理 -> 再编排 -> 再端到端功能。

陷阱5 -- Legacy层仍为新功能开发场:
CL-1(hcomm:d0d5cce1, multi process per rank): 30文件全在legacy/，是一个完整的新功能。Phase 8.7: 40条legacy-only FEATURE commit。legacy不是旧芯片专用代码(88.9%无芯片关键词)。这个现实不可回避——在legacy稳定之前，迁移只会增加两条路径的维护负担。

陷阱6 -- 17个TODO在framework/next/:
Phase 1发现。next框架自身也未完成，迁移的target可能在变化。在next稳定前做迁移，可能需要二次返工。
