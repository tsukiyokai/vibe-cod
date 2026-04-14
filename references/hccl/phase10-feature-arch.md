# Phase 10.1: FEATURE + ARCH_ADAPT Landmark Cases Deep Analysis

Date: 2026-03-16
Phase: 10.1 (Decision Point Deep Analysis — FEATURE + ARCH_ADAPT)
Cases analyzed: 13 (9 FEATURE + 4 ARCH_ADAPT)

---

## FEATURE Cases

### Case FT-1: new selector — 架构级算法选择重构

[Commit]: hccl:549c4537 (new selector / !114 merge bigSelector into master)
[类别]: FEATURE
[涉及层]: hccl:ops/op_common(selector/topo) + hccl:ops/全部14个算子(selector+executor+template)
[变更规模]: 220 files, +1123/-666 lines

[场景]: 开发者需要将算法选择机制从"仅支持简单的MESH_1D/CLOS两种拓扑"升级为"支持多层拓扑网络的详细信息提取和精细化算法选择"。旧的TopoInfo只记录简单的层级数和level0形状，无法表达CLOS+MESH混合拓扑(MESH_1D_CLOS)或双die全互联等复杂物理拓扑。

[约束]:
1. TopoInfo是序列化传输给device侧AICPU kernel的核心结构体，修改它必须同时修改Serialize/DeSerialize方法
2. 220个文件中约180个是"机械性"签名修改(TopoInfo*→TopoInfoWithNetLayerDetails*)，真正的架构变更集中在5个文件
3. 所有14个算子的Selector虚方法签名必须同步修改(继承体系要求)
4. alg_param.h是"变更放大器"(Phase 1已知，14次修改)，修改它会引发联动编译

[决策]: 选择继承+扩展方案:
- 新增TopoInfoWithNetLayerDetails继承自TopoInfo，保持原有字段不变
- 新增NetLayerDetails和TopoInstDetails两个辅助结构体，提供每层网络实例的详细拓扑信息
- 将CalcTopoLevelNums+CalcLevel0TopoShape合并重构为CalcTopoShape→(ExtractNetLayerDetails+ExtractTopoDetails+CalcLevel0TopoShape+Is2DieFullMesh)四步pipeline
- 将HcclExecOp拆分为Selector+HcclExecOp两个独立阶段(算法选择和执行分离)
- AICPU和AIV的kernel launch逻辑分别提取为HcclAicpuKernelEntranceLaunch和HcclAivKernelEntranceLaunch独立函数

核心架构决策:
- 选择继承而非替换: TopoInfoWithNetLayerDetails : public TopoInfo。这样旧代码中只需TopoInfo的地方不受影响，需要详细拓扑信息的地方传子类指针。
- 序列化采用"尾部追加"策略: 基类字段照常序列化，子类新增字段的序列化结果append到末尾。DeSerialize时通过topoInfoSeqSize从末尾分割。这是一种版本兼容的序列化方案。
- 新增Level0Shape::MESH_1D_CLOS枚举值(值=2)，支持CLOS和MESH_1D并存的混合拓扑。
- 新增IsLayerAllConnetedWithTopo工具方法，用于判断某一层网络的某种拓扑类型是否能覆盖所有本地卡。
- 从HcclExecOp中删除了引擎类型的硬编码设置(opParam.engine = CommEngine::COMM_ENGINE_AIV等)，改由SetCommEngine统一处理。

[替代方案]:
1. 直接在TopoInfo中新增字段: 会破坏所有已有的序列化格式和所有使用TopoInfo的代码。
2. 不用继承，在Selector层面单独获取拓扑详情: 会导致TopoInfo和拓扑详情分离传递，增加接口复杂度。
选择继承方案的核心原因是向后兼容——旧的Executor和Template只需要TopoInfo基类信息，新的Selector需要完整的拓扑详情。

[后果]:
- 次日BF-9(80a70729): "ccu模式scatter_mesh1d算子修改每张卡设置地址的时机"，14文件修复。selector重构暴露了CCU scatter中地址设置时机问题(EF-4 probable pair)。
- 后续8d3cd1d: "cumstomized_ccu algorithm"在新selector框架上新增CCU算法。
- 后续84ddec9+a00e3be: 两次CleanCode清理，说明大规模commit后需要收尾工作。

[可迁移经验]: 当核心参数结构体需要扩展时，使用继承+尾部追加序列化的策略可以最小化对已有代码的冲击。但220文件的联动修改在合入后1天即暴露CCU scatter bug，说明大规模selector重构必须配合全算子+全设备模式的回归测试。

---

### Case FT-2: add A5 aicpu communicator — 新芯片通信域管理独立化

[Commit]: hccl:12a2c84f + hcomm:dba4d87b (add A5 aicpu communicator)
[类别]: FEATURE
[涉及层]: hccl:ops/template/aicpu + hcomm:framework/next/coll_comms + hcomm:framework/device
[变更规模]: hccl: 1 file, +4/-4; hcomm: 26 files, +728/-104

[场景]: A5芯片(910_95)需要独立的AICPU通信域管理类。旧架构中HcclCommAicpu(aicpu_communicator.cc)是所有芯片共用的巨型类(5000+行)，通信域初始化、URMA channel分配、Notify管理等功能全部耦合在一起。A5芯片需要一套新的IndependentOp流程，与旧芯片的流程差异很大。

[约束]:
1. A5(910_95)走新框架(framework/next/coll_comms)，旧芯片走旧框架(framework/device)
2. HcclCommAicpu已经是bug密度最高的文件之一(Phase 1: 18次BUGFIX修改)
3. hccl侧的kernel_launch.cc需要按芯片类型区分通信域获取方式
4. 跨仓提交间隔仅4分钟(hccl 10:47, hcomm 10:51)

[决策]:
- hcomm侧: 在framework/next/coll_comms/communicator/aicpu/下新建独立模块(4个新文件):
  - CollCommAicpu: 独立算子AICPU通信域管理(替代HcclCommAicpu中A5专用逻辑)
  - CollCommAicpuMgr: 管理器(全局单例模式)
  - AicpuIndopProcess: 独立算子流程处理
  - 同时将coll_comm.cc/coll_comm_mgr.cc迁移到communicator/子目录(目录重组)
- hcomm侧: 从HcclCommAicpu中删除63行A5专用代码(AllocChannelResourceV2/InitUrmaChannel/ParsePackData)
- hccl侧: kernel_launch.cc中增加DevType判断:
  ```
  if (param->deviceType != DevType::DEV_TYPE_910_95) {
      HcommAcquireComm(param->commName); // 旧芯片走旧流程
  }
  ```
  A5芯片不再需要HcommAcquireComm，因为通信域管理已转移到CollCommAicpu。

[替代方案]:
1. 在HcclCommAicpu中添加if/else分支: 会使已经5000+行的巨型类更加臃肿。
2. 使用虚函数多态+策略模式: 改动量更大，且旧芯片路径的稳定性可能被破坏。
选择"新建独立类+删除旧类中的A5代码"是风险最小的方案——旧路径零修改。

[后果]: 无revert。后续多个commit继续在CollCommAicpu上添加功能，证明架构决策正确。

[可迁移经验]: 为新芯片添加通信域管理时，优先在framework/next下新建独立模块，从旧的巨型类(如HcclCommAicpu)中删除芯片专用代码，而非在旧类中添加分支。跨仓提交必须同步(本例4分钟内)，hccl侧修改极少(仅DevType判断)而hcomm侧承担主要工作(728行新增)。

---

### Case FT-3: double die alg — 双die拓扑CCU算法族

[Commit]: hcomm:fa8ed355 (double die alg)
[类别]: FEATURE
[涉及层]: hcomm:legacy/framework/ccu + hcomm:legacy/unified_platform
[变更规模]: 29 files, +3057/-18 lines

[场景]: 需要为双die互联拓扑(2-die fullmesh)新增CCU调度算法。双die拓扑是910_95芯片的新硬件拓扑——一个server内有两组die，每组die内部全互联，两组die之间也有链路。传统的mesh_1D/mesh_2D算法无法利用die间链路。

[约束]:
1. 必须在legacy框架的CCU指令集体系内实现(CcuInstruction继承体系)
2. 4种算子同时适配: AllGather/AllToAll/AllToAllV/ReduceScatter
3. CCU指令集通过MAKE_ENUM宏注册，新增4个枚举值
4. 每个算法需要ccu_context(上下文准备) + ccu_instruction(指令模板) + ccu_temp(执行模板)三层文件

[决策]:
- 在CcuInstType枚举末尾新增4个值: CCU_ALLTOALLV_MESH_2DIE_DIRECT, CCU_ALLGATHER_MESH_1D_2DIE, CCU_ALLTOALL_MESH_1D_2DIE, CCU_REDUCE_SCATTER_MESH_1D_2DIE
- 每个算子新增3个文件(context+instruction+template)，共12个新文件
- legacy/unified_platform/ccu_context/ccu_context.cpp新增241行case分支来处理新指令类型
- 4个Executor(V2 sole_executor)各新增3行来注册新算法名

代码组织模式: 严格遵循既有的CCU算法三层结构:
- ccu_context_xxx: 负责准备CCU执行所需的参数(地址计算、notify映射、slice分配)
- ccu_instruction_xxx: 定义CCU微指令序列(DataCopy/WaitNotify/SignalNotify)
- ccu_temp_xxx: 从AlgTemplateBase继承，组装context和instruction

[替代方案]:
1. 使用AICPU算法而非CCU: 可行但性能差——CCU可以做硬件级的通信流水线编排，AICPU需要软件循环。
2. 在已有mesh_1D算法中添加die感知分支: 会使现有稳定算法复杂化。

[后果]: 无revert。FT-1(new selector)中新增了Is2DieFullMesh()判断来与这些双die算法配合。

[可迁移经验]: 新增CCU算法必须遵循context+instruction+template三层文件结构，每个新拓扑形状需要在CcuInstType枚举末尾添加值(而非插入中间)。一个拓扑形状通常需要同时适配4种以上集合通信算子，规模在10-15个新文件(+2000-3000行)。

---

### Case FT-4: add host DPU support in reducescatter — 从零搭建DPU算子

[Commit]: hccl:3e916555 + hccl-dev:925ffbb (add host dpu support in reducescatter)
[类别]: FEATURE
[涉及层]: hccl:ops/reduce_scatter(新建) + hccl:ops/common(扩展)
[变更规模]: 27 files, +1535/-26 lines (hccl main); 高度对称的镜像提交在hccl-dev

[场景]: 为ReduceScatter算子添加Host DPU(数据处理单元)网卡支持。DPU是一种不同于device-side AICPU/AIV/CCU的执行模式——算法逻辑运行在Host CPU上，通过Host网卡(而非device侧HCCS/RoCE)进行通信。此前只有scatter支持DPU模式。

[约束]:
1. 需要在hccl仓内新建完整的reduce_scatter算子目录(与已有的device-side算子平行)
2. DPU模式下不走已有的Selector→Executor→hcomm链路，而是Op层直接编排
3. 必须扩展通用框架(executor_base/alg_template_base/channel)来支持Host DPU通信
4. 两仓(main+dev)近乎完全对称的代码，说明此时两仓的代码分歧尚小

[决策]:
- 新建src/ops/reduce_scatter/完整目录，包含:
  - reduce_scatter_op.cc(683行): 完整的Op层入口，包含参数校验、拓扑信息获取、算法选择、资源分配、执行全流程
  - algo/reduce_scatter_mesh_executor.cc(209行): DPU模式的Executor
  - algo/template/两个template文件: Host DPU传输和本地Reduce
- 扩展通用框架:
  - alg_param.h: 新增DPUAlgResourceCtx结构体(DPU专用资源上下文)
  - executor_base.cc: 新增DPU Channel建立方法
  - channel.cc: 新增DPU协议的Channel配置逻辑(37行)
  - topo.cc: 新增DPU拓扑检测逻辑(+86行)

关键设计选择:
- reduce_scatter_op.cc中有独立的CheckHostDPUOnly()函数——通过检测最高level网络的链路类型来判断是否是纯DPU场景
- DPU模式走Executor→Template两层(不走hcomm)，与device-side的Selector→Executor→hcomm三层路径不同
- 注册了HcclLaunchDpuKernel回调函数用于DPU kernel启动

[替代方案]:
1. 在已有的device-side reduce_scatter中添加DPU分支: 不可行，因为DPU的执行模型(Host CPU+Host NIC)与device-side(AICPU/CCU+Device NIC)完全不同。
2. 仅在hcomm层实现: 不可行，DPU模式需要在hccl:Op层直接编排(不经过hcomm的communicator)。

[后果]: hccl main和hccl-dev高度对称(差异仅在topo.cc的+86/+99行)。后续多个commit继续完善DPU支持(如FT-1中新增SelectDPUAlgo路径)。

[可迁移经验]: 新执行模式(如DPU)如果与现有device-side模型差异太大，应该新建独立的Op目录和Executor，而非在device-side代码中添加分支。DPU算子的架构是Op→Executor→Template两层(跳过hcomm)，与device-side的三层路径本质不同。

---

### Case FT-5: alg selector remake — hcomm层Legacy Selector重塑

[Commit]: hcomm:0d9239b5 (alg selector remake)
[类别]: FEATURE
[涉及层]: hcomm:legacy/service/collective/alg/selector + hcomm:legacy/unified_platform/topo
[变更规模]: 25 files, +1396/-881 lines

[场景]: hcomm legacy层的算法选择器需要与hccl层(FT-1)的new selector保持概念同步——支持同样的多层拓扑信息提取和精细化算法选择。这是FT-1在hcomm层的对应物。

[约束]:
1. 修改在legacy层(legacy/service/)，不是next层——因为大量生产代码仍走legacy路径
2. 需要修改8个算子的auto_selector子类和通用的base_selector
3. 必须与hccl层FT-1引入的NetLayerDetails/TopoInstDetails概念保持对齐
4. RankGraph API(GetLayers/GetInstSizeListByLayer/GetTopoInstsByLayer)是两层共用的底层接口

[决策]:
- 在base_selector.h中引入与FT-1概念对齐的嵌套结构体(NetLayerDetails, TopoInstDetails)
- 新增Level0Shape::CLOS(=3)和MESH_1D_CLOS(=4)枚举值(注意与hccl层的值不同: hccl用0/1/2, hcomm用1/2/3/4)
- 新增方法: ExtractNetLayerDetails/ExtractTopoDetails/CalcLevel0TopoShape/IsLayerAllConnetedWithTopo/Is2DieFullMesh
- 在base_selector.cc中实现328行的拓扑提取和判断逻辑
- 8个算子selector各自更新算法选择分支(利用新的拓扑信息做更精细的决策)

设计对比(FT-1 vs FT-5):
- FT-1(hccl): 拓扑结构体定义在alg_param.h(公共头文件)，全局可见
- FT-5(hcomm): 拓扑结构体定义为base_selector.h的protected嵌套结构体，仅selector子类可见
- 概念相同但枚举值不同: Level0Shape在hccl是{CLOS=0, MESH_1D=1, MESH_1D_CLOS=2}，在hcomm是{MESH_1D=1, MESH_2D=2, CLOS=3, MESH_1D_CLOS=4}
- 这种不一致是跨仓独立演进的典型后果

[替代方案]:
1. 直接复用hccl层的TopoInfoWithNetLayerDetails: 不可行，hccl和hcomm的Selector体系完全独立(不共享头文件)
2. 在公共头文件(hcomm_primitives.h)中定义共享结构体: 可行但增加跨仓耦合
选择在各层独立定义结构体是当前架构下的务实选择。

[后果]: 无revert。后续AA-3在此基础上为A3芯片新增midcount优化路径。

[可迁移经验]: hccl层和hcomm层的selector是独立演进的平行体系——概念对齐但实现独立。修改算法选择逻辑时，两层都需要更新(先更新概念层hccl FT-1，再更新实现层hcomm FT-5)。枚举值不一致是跨仓演进的代价，需要在文档中明确映射关系。

---

### Case FT-6: coll comm & orion comm mix-running — 同日引入+同日回退

[Commit]: hcomm:ebb54874 (support coll comm & orion comm mix-running) → hcomm:0c033874 (Revert)
[类别]: FEATURE (被REVERT)
[涉及层]: hcomm:framework/next/comms + hcomm:framework/op_base + hcomm:legacy/framework
[变更规模]: 37 files, +1140/-317 lines (引入); 37 files, +322/-1145 (回退, 精确镜像)

[场景]: A5芯片需要支持新老算子混合运行——coll_comm(新V2集合通信)和orion_comm(旧通信域)在同一进程中并存。这需要修改资源管理器、op_base编排、socket管理、CCU资源分配等多个模块。

[约束]:
1. 涉及next框架和legacy框架的交叉修改(跨越V1/V2边界)
2. op_base.cc修改量最大(560行新增)——这是算子执行的核心编排文件
3. hcomm_res_mgr.cc/h新建(120行)——引入了新的资源管理器
4. 两位作者联合开发(shibingchen+renmengguang)
5. MR描述: "支持A5新老算子混跑"

[决策]:
- 在include/hccl/hccl_comm.h新增20行公开API(跨仓接口)
- op_base.cc大量重构: 560行新增，改变了算子执行的编排流程
- 引入hcomm_res_mgr.cc(87行): 管理新老通信域的资源协调
- endpoint_pair.cc增加37行: 新增endpoint pair管理
- legacy communicator_impl.cc仅修改5行(最小化legacy影响)

[替代方案]:
1. 不支持混跑，要求用户全量迁移到新算子: 对用户不友好
2. 在更高层(hccl Op层)做模式切换: 会将混跑复杂度上推到算子层

[后果]:
- 同日9.5小时后被完整Revert(0c033874)——37文件精确还原
- 回退原因: 从时间间隔(同日)和精确镜像回退来看，最可能是集成测试失败
- 后续1abd85b8 "fix nullptr core dump"修复了op_base.cc中的空指针问题(但此时mix-running已回退，fix可能针对回退后残留的问题)
- 截至分析时点(2026-03-16)，mix-running功能尚未重新引入main仓

[可迁移经验]: 跨越V1/V2边界的大规模功能(37文件+1140行)合入即回退的概率极高。核心教训: (1)影响op_base.cc(算子执行核心编排)的大功能应该分步合入(先资源管理→再编排→再端到端功能)，而非一次性all-in; (2)两位作者联合开发增加了代码一致性风险; (3)缺少feature flag意味着一旦出问题只能全量回退。

---

### Case FT-7: symmetric memory for aicpu unfold mode — 对称内存管理系统

[Commit]: hcomm:4c779e13 (Support symmetric memory for aicpu unfold mode)
[类别]: FEATURE
[涉及层]: hcomm:framework/communicator/impl + hcomm:framework/device + hcomm:framework/common + hcomm:framework/op_base + test/ut
[变更规模]: 47 files, +3930/-28 lines

[场景]: AICPU展开模式需要对称内存支持——所有rank在同一虚拟地址空间中映射物理内存，使得任意rank可以直接通过VA访问任意远端rank的数据，无需显式通信。这是高性能集合通信(如zero-copy AllGather)的基础设施。

[约束]:
1. 依赖NPU VMM API(aclrtDrvMemHandle/aclrtMemFabricHandle)进行物理内存管理
2. 必须支持跨rank的VA预留和物理内存映射(symmetric = 所有rank在相同VA偏移处映射)
3. 需要与已有的CommMemMgr引用计数机制兼容
4. Device侧(aicpu_communicator)需要能访问对称内存

[决策]:
- 新建symmetric_memory/目录(4个文件, 1000+行核心代码):
  - SymmetricMemory: 核心管理器(612行)，负责VA空间预留、注册、映射和查找
  - SymmetricMemoryAgent: 代理类(249行)，封装实际的VMM API调用
  - 关键数据结构: FreeBlock(空闲块)/ShareableInfo(共享信息)/SymmetricWindow(对称窗口)/PaMappingInfo(物理地址映射+引用计数)
- 在hccl_comm_pub.h/hccl_types.h新增公开API
- hccl_communicator_host.cc新增103行: 对称内存的初始化和管理集成
- op_base.cc新增66行: 算子执行流程中的对称内存参数传递
- aicpu_communicator.cc新增108行: device侧的对称内存访问接口
- aicpu_symmetric_memory.cc(52行新文件): device侧对称内存操作封装
- 1245行单元测试(ut_symmetric_memory.cc): 覆盖率极高

引用计数设计:
- PaMappingInfo.refCount: 每个物理内存块有引用计数，多个Window可以共享同一块物理内存
- 当refCount降为0时才执行Unmap和Release VA
- 这与Phase 2.2中CommMemMgr的引用计数模式一致

[替代方案]:
1. 每次操作动态分配+释放: 性能差，无法做到zero-copy
2. 使用传统的MPI Shared Memory: 不支持NPU VA空间的特殊需求

[后果]:
- a9fea200 "修复资源释放不完全问题": 证明初始版本存在资源泄漏bug
- 后续aab53593+86fcecba+e58087aa: 三次CleanCode清理
- 资源泄漏修复发生在引入后不久，符合Phase 1的规律(设备侧资源管理是bug密集区)

[可迁移经验]: 新增内存管理子系统时，引用计数是必须的(本例PaMappingInfo.refCount)，但初始版本几乎必然有资源释放遗漏(本例a9fea200修复)。建议: (1)写完核心逻辑后，先写资源泄漏检测的单元测试; (2)3930行的大feature中1245行是UT(32%)，这个UT比例是本仓库的标杆。

---

### Case XR-2: scatter switch to aicpu process — 跨仓执行模式切换

[Commit]: hccl-dev:0ca13cd6 + hcomm-dev:79e73f3c (scatter switch to aicpu process)
[类别]: FEATURE
[涉及层]: hccl-dev:ops/aicpu + hcomm-dev:framework/communicator + hcomm-dev:framework/device
[变更规模]: hccl-dev: 4 files, +31/-99; hcomm-dev: 14 files, +31/-88

[场景]: 将scatter算子从原有的执行模式切换到AICPU进程模式。涉及kernel加载路径简化(hccl侧)和通信器配置调整(hcomm侧)。

[约束]:
1. 同一作者(wengjingchong)同日提交两仓(hccl-dev 19:15:43, hcomm-dev 19:15:25)，间隔仅18秒
2. 这是dev仓的实验性变更(两仓都在dev)
3. kernel加载路径从复杂的LD_LIBRARY_PATH解析改为简单的ASCEND_HOME_PATH

[决策]:
- hccl-dev侧: load_kernel.cc大幅简化(-99行): 删除复杂的ParseLibraryPath/GetKeyWordPath路径解析逻辑，改为直接从ASCEND_HOME_PATH环境变量获取路径
- hcomm-dev侧: 删除不再需要的device launch逻辑(-88行)，简化communicator的设备类型判断
- ccl_kernel.ini更新: 添加新的kernel注册条目(+22行)

核心简化:
```
旧: LD_LIBRARY_PATH解析→遍历每个路径→查找关键词→拼接cannPath
新: getenv("ASCEND_HOME_PATH") + 固定后缀路径
```

[替代方案]:
1. 保留旧的路径解析逻辑，仅添加新模式: 增加维护负担，且旧逻辑中有注释掉的TODO代码
选择删除旧逻辑是因为ASCEND_HOME_PATH是CANN官方推荐的环境变量。

[后果]: dev仓变更，后续在main仓(FT-1等)中看到了更完善的kernel加载逻辑。

[可迁移经验]: 跨仓的执行模式切换必须原子提交(本例18秒间隔)。hccl侧通常是"减法"(删除旧逻辑)，hcomm侧是"调整"(适配新流程)——非对称但互补。

---

### Case CL-1: multi process per rank — Legacy层端口管理升级

[Commit]: hcomm:d0d5cce1 (multi process per rank)
[类别]: FEATURE
[涉及层]: hcomm:legacy/framework/communicator + hcomm:legacy/framework/resource_manager/socket + hcomm:legacy/framework/env_config + hcomm:legacy/unified_platform
[变更规模]: 30 files, +217/-136 lines

[场景]: 支持每个rank运行多个进程——需要解决端口冲突问题。旧设计中端口号是硬编码的(60001)，多进程场景下会冲突。

[约束]:
1. 所有修改在legacy层——这是legacy层仍为新功能开发场的证据
2. communicator_impl.cc是最高频修改文件(47次)，修改它需要格外谨慎
3. socket_manager既要支持旧的固定端口方式，又要支持新的动态端口方式
4. 子通信域(CreateSubComm)需要继承父通信域的端口映射

[决策]:
- 删除硬编码端口号(60001): `u32 stubListenPort = 60001;` → 从ranktable配置获取
- socket_manager构造函数签名变更: 用deviceLogicId替代serverListenPort
- 新增SetDeviceServerListenPortMap/GetDeviceServerListenPortMap: 每个设备可配置独立监听端口
- 新增PreemptPortManager集成: 支持端口范围内的抢占式监听(多进程争抢端口)
- communicator_impl.cc中子通信域创建后继承父通信域的端口映射:
  ```
  subCommImpl->GetSocketManager().SetDeviceServerListenPortMap(
      GetSocketManager().GetDeviceServerListenPortMap());
  ```
- env_config.cc扩展: 从配置文件读取设备端口范围

关键修改模式: 原来的`return subCommImpl->Init(...)` 改为 `CHK_RET(subCommImpl->Init(...)); subCommImpl->SetPortMap(...); return HCCL_SUCCESS;`——在Init之后追加端口配置步骤。

[替代方案]:
1. 在Init内部自动获取端口映射: 需要修改Init签名和所有调用者
2. 通过环境变量传递端口: 不够灵活，无法支持per-device端口配置

[后果]: 无revert。后续的socket相关修改继续基于此框架。

[可迁移经验]: Legacy层的新功能开发是现实(本例30文件全在legacy/)。端口管理从硬编码升级到配置化时，关键是子通信域必须继承父通信域的端口映射——这种"初始化后补充配置"模式在communicator_impl.cc中反复出现。

---

## ARCH_ADAPT Cases

### Case AA-1: 兼容Abi&接口改名 — 跨仓ABI兼容层引入

[Commit]: hccl-dev:217dd2cf + hcomm-dev:f50889af (兼容Abi&接口改名)
[类别]: ARCH_ADAPT
[涉及层]: hccl-dev:ops/inc + hccl-dev:ops/scatter + hcomm-dev:framework + hcomm-dev:platform
[变更规模]: hccl-dev: 21 files, +243/-99; hcomm-dev: 23 files, +346/-193

[场景]: hccl和hcomm仓之间的Channel接口需要ABI(Application Binary Interface)兼容层。旧接口直接使用C++结构体传递Channel参数，但两仓独立编译，结构体布局不一致会导致ABI不兼容。

[约束]:
1. hccl和hcomm是独立编译的两个库，通过C ABI边界交互
2. 已有的ChannelDesc结构体在两仓定义不一致(字段顺序/大小可能不同)
3. 需要向后兼容——旧版本的hccl能和新版本的hcomm配合工作

[决策]:
- 在alg_param.h中定义HcclAbiHeader结构体(version+magicWord+size+reserved)
- 新定义HcclChannelDesc结构体(带ABI头+通信参数)
- 所有跨仓接口声明为`__attribute__((weak))`弱符号——允许运行时动态绑定
- 接口改名: HcommAclrtNotifyRecordOnThread等一批通信原语接口统一命名规范
- 初始化函数HcclChannelDescInit: 确保每个HcclChannelDesc都正确填充version/magicWord/size

弱符号策略:
```c
HcclResult HcclChannelAcquire(...) __attribute__((weak));
int32_t HcommAclrtNotifyRecordOnThread(...) __attribute__((weak));
```
弱符号允许: 如果hcomm提供了实现就用hcomm的，否则链接到默认stub——这是解耦两仓编译依赖的关键手段。

[替代方案]:
1. 统一两仓编译: 不可行，两仓是独立的发布单元
2. 使用纯C void*接口: 已有的comm_primitive就是这样做的，但Channel接口需要更多类型信息

[后果]: 无revert。后续FT-2(A5 communicator)等都基于这套ABI兼容接口开发。

[可迁移经验]: 跨仓接口必须有ABI头(version+magicWord+size)来保证版本兼容。使用`__attribute__((weak))`弱符号是解耦两仓编译的标准手段——hccl仓可以声明接口但不实现，运行时由hcomm仓提供实际实现。

---

### Case AA-2: Aiv A3 Broadcast AGV RSV — A3芯片AIV算法批量适配

[Commit]: hcomm-dev:88f70272 (Aiv A3 Broadcast AGV RSV)
[类别]: ARCH_ADAPT
[涉及层]: hcomm:algorithm/impl + hcomm:algorithm/base/alg_aiv_template
[变更规模]: 18 files, +1724/-49 lines

[场景]: A3芯片(910_93)需要AIV(AI Vector)算法支持Broadcast、AllGatherV、ReduceScatterV三种集合通信操作。A3与A5的AIV kernel参数格式不同(A3使用ARGS_TYPE_SIMPLE)，需要为A3新增专用kernel和executor。

[约束]:
1. A3的kernel参数使用AivKernelArgsV3/V4格式(带massArgs数组)，与A5的AivKernelArgsV1/V2不同
2. A3 kernel名称带"_cn"后缀(如aiv_broadcast_cn_half)以区分
3. 每种数据类型需要单独注册kernel(6-9种数据类型 x 3种操作 = 27个新kernel注册)
4. 跨节点(crossnode)算法需要特殊处理

[决策]:
- hccl_aiv.cc中批量注册A3专用kernel(每种操作6-9个新条目，使用KernelArgsType::ARGS_TYPE_SIMPLE标记)
- 新增3个executor类: CollBroadcastMeshAivFor91093, CollAllGathervMeshAivFor91093, CollReduceScatterVMeshAivFor91093
- 新增3个A3专用AIV template头文件(crossnode_91093系列)
- 引入AivKernelArgsV4结构体(扩展V3，添加serverNum/devType/deterministic/extraArgs)
- operator层(all_gather_v_operator.cc等)新增A3分支:
  ```
  if (devType == 910_93 && isAivMode) {
      executor = make_unique<CollAllGathervMeshAivFor91093>(...);
  }
  ```

模式: "后缀区分法"——A3 kernel用_cn后缀，A3 executor用_for_910_93后缀，保持命名空间清晰。

[替代方案]:
1. 在已有kernel中通过运行时参数区分A3/A5: 不可行，kernel编译为二进制，参数格式在编译时确定
2. 使用模板元编程消除重复: 可行但当前代码库的惯用法是显式枚举每种数据类型

[后果]: 无revert。后续AA-3在此基础上添加A3 midcount优化。

[可迁移经验]: A3芯片AIV适配的标准模式: (1)hccl_aiv.cc注册带_cn后缀的kernel条目+ARGS_TYPE_SIMPLE标记; (2)新建_for_910_93后缀的Executor类; (3)新建_crossnode_91093后缀的Template头文件; (4)operator层添加devType分支。每种操作x每种数据类型=约10个新注册条目。

---

### Case AA-3: AG&AR midcount optim for A3 — A3中数据量优化路径

[Commit]: hcomm:c8a15c3e (AG&AR midcount optim for A3)
[类别]: ARCH_ADAPT
[涉及层]: hcomm:algorithm/impl/operator + hcomm:algorithm/base/communicator + hcomm:algorithm/impl/coll_executor
[变更规模]: 13 files, +579/-25 lines

[场景]: A3芯片(910_93)在多超节点(multi super pod)场景下，数据量处于16KB到256KB之间("midcount")时需要专用优化算法。现有算法在此区间性能不佳——小数据走small count优化，大数据走标准pipeline，中间数据量两者都不适配。

[约束]:
1. midcount阈值: 16KB < dataSize <= 256KB(HCCL_SMALL_COUNT_16_KB到HCCL_SMALL_COUNT_256_KB)
2. 仅在superPodNum>1且未启用retry时生效(涉及RoCE平面)
3. 需要为AllGather和AllReduce各新建一个executor

[决策]:
- 在AllGather/AllReduce operator的SelectAlgfor91093方法中新增midcount分支:
  ```
  bool midCountOptimMultiPod = (superPodNum_ > 1) &&
      (count * unitSize > HCCL_SMALL_COUNT_16_KB) &&
      (count * unitSize <= HCCL_SMALL_COUNT_256_KB) && !retryEnable_;
  ```
- 新建CollAllGatherMidCountFor91093Executor(235行)和CollAllReduceMidCountFor91093Executor(183行)
- topo_info_extractor.cc新增15行: 提取midcount优化所需的拓扑信息
- 选择条件: midCountOptimMultiPod优先级介于small count和standard之间

Selector分支插入位置精心选择——在smallCountOptimMultiServer之后、symmetricMemory/zeroCopy之前:
```
} else if (smallCountOptimMultiServer || smallCountOptimSingleServer) {
    algName = "AllGatherSmallCount";
} else if (midCountOptimMultiPod) {            // <-- 新增
    algName = "AllGatherMidCountFor91093Executor";
} else if ((param.supportSymmetricMemory || param.supportZeroCopy) && ...) {
```

[替代方案]:
1. 调整现有small count/standard的阈值而非新建executor: 可能影响已有场景的性能
2. 在framework层做midcount优化: 不合适，midcount是算法层面的选择

[后果]: 无revert。13文件的精简修改，没有引发后续bugfix。

[可迁移经验]: A3芯片性能优化路径的添加模式: (1)在operator层的SelectAlgfor91093方法中添加数据量区间判断; (2)新建For91093Executor; (3)将新分支插入selector的优先级链中正确位置。阈值选择(16KB-256KB)必须有性能测试数据支撑，不能凭直觉。

---

### Case AA-5: RoCE协议场景下flush方案适配

[Commit]: hcomm:4afb1c39 (RoCE协议场景下flush方案适配)
[类别]: ARCH_ADAPT
[涉及层]: hcomm:legacy/framework/communicator/hostdpu
[变更规模]: 5 files, +36/-1 lines

[场景]: Host DPU(Data Processing Unit)场景下，RoCE协议需要flush操作来确保RDMA写入的数据对远端可见。不同网卡的flush能力不同——需要检测网卡是否支持flush opcode。

[约束]:
1. flush_handle.cc是Host DPU通信的关键组件
2. 需要运行时检测网卡load balance能力(GetLbMax)来决定flush策略
3. RoCE协议和HCCS协议的flush行为不同

[决策]:
- 在FlushHandle::Init中新增LbMax检测:
  ```
  CHK_RET(GetLbMax(&rdmaHandle, &lbMax));
  if (lbMax > 0) {
      SetFlushOpcodeSupport(); // 标记支持flush opcode
  }
  ```
- 新增FlushHandle::GetLbMax方法(12行): 封装RaGetLbMax API调用
- flush_manager.cc添加1行条件判断

设计精髓: 通过硬件能力检测(LbMax>0)来决定是否启用flush opcode，而非通过芯片型号硬编码——这使得同一代码可以在不同网卡上自动选择最优flush方案。

[替代方案]:
1. 按芯片型号硬编码flush行为: 不灵活，无法适应同一芯片搭配不同网卡的场景
2. 始终使用保守的flush方案: 性能损失

[后果]: 无revert。5文件36行的精简修改，展示了正确的硬件适配粒度。

[可迁移经验]: 协议层适配应该基于运行时硬件能力检测(如GetLbMax)而非芯片型号硬编码。这种"capability-based"方案比"device-type-based"方案更具前向兼容性——新网卡自动被正确处理，无需代码修改。

---

## 横切面分析

### FEATURE类的共性模式

1. 分层职责明确:
   - hccl层负责算法选择(Selector)和执行编排(Op/Executor)
   - hcomm层负责通信实现(algorithm/framework/platform)
   - 新功能遵循"选择在hccl，实现在hcomm"的原则(FT-1 vs FT-5)

2. 跨仓不对称性:
   - hccl侧通常改1-4文件(FT-2: 1文件, XR-2: 4文件)
   - hcomm侧通常改20-47文件(FT-2: 26文件, FT-7: 47文件)
   - 比例约1:10——hcomm承担绝大部分实现工作

3. 新功能的标准规模:
   - 小功能: 5-15文件, 300-600行(AA-3, AA-5)
   - 中等功能: 25-37文件, 1000-1500行(FT-2, FT-4, FT-6, CL-1)
   - 大功能: 47-220文件, 3000+行(FT-1, FT-7, FT-3)

4. 被Revert的特征(FT-6):
   - 跨越V1/V2边界(同时修改next和legacy)
   - 多作者联合开发
   - op_base.cc大量修改(算子编排核心)
   - 缺少feature flag
   - 无分步合入策略

### ARCH_ADAPT类的共性模式

1. 双重适配策略:
   - capability-based(AA-5: GetLbMax运行时检测): 更灵活，前向兼容
   - device-type-based(AA-2: devType==910_93分支): 更直接，但需要每个新芯片手动适配

2. A3芯片适配标准流程(AA-2→AA-3):
   - 先做基础kernel注册(AA-2: 批量注册_cn后缀kernel)
   - 再做性能优化(AA-3: midcount区间优化)
   - 命名惯例: _for_910_93后缀(executor), _cn后缀(kernel), _crossnode_91093后缀(template)

3. ABI兼容是跨仓的刚性约束(AA-1):
   - version+magicWord+size头是必须的
   - 弱符号(__attribute__((weak)))是解耦两仓编译的标准手段
