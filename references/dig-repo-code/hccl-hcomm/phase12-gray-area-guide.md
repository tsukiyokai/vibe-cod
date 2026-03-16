# Phase 12: 灰色地带指南

Date: 2026-03-16
Phase: 12 (灰色地带指南)

灰色地带是指conventions无法给出确定答案、需要开发者运用判断力的决策场景。
每个灰色地带都有支持两种方案的真实案例——这正是它"灰色"的原因。

---

## 12.1 何时用V1路径 vs V2路径

### 核心矛盾

hccl/hcomm存在两套并行的执行架构:
- V2(IndependentOp/pImpl委托): 新架构，通过CommunicatorImpl + pImpl委托到CollComm/MyRank，HCCP HDC直接初始化。设计更简洁(InsCollAlgBase只有3个纯虚方法)，资源上下文可序列化(AlgResourceCtxSerializable)，拓扑信息更丰富(TopoInfoWithNetLayerDetails)。但目前只有910_95(A5)芯片支持。
- V1(RankGraph全量实现): 旧架构，通过HcclCommunicator + CommFactory + RankGraph + ContextManager实现，28步初始化序列。设计宽松(ExecutorBase只有1个纯虚方法)，覆盖所有平台，代码成熟但臃肿(communicator_impl.cc被修改47次居全仓首位)。

矛盾的本质不是"新vs旧"的简单取舍，而是一个更深层的张力: V2确实更优，但legacy层从2026-02起占hcomm总commit的40%且仍在爆发性增长(Phase 8.7数据)，40条legacy-only FEATURE commit证明新功能仍在legacy中开发。legacy从"产品名"(orion)重新定位为"遗留代码"(2026-01-31 commit 62ba4500做了orion→legacy重命名)，但实际活跃度不降反升。这意味着选V1还是V2不能简单看"哪个更新"，而要看你的变更落在哪个执行域。

### 支持V2路径的案例

Case FT-2 (hccl:12a2c84f + hcomm:dba4d87b): A5 AICPU communicator。为910_95新芯片新增AICPU通信域管理，选择在framework/next/coll_comms/下新建独立模块CollCommAicpu(4个新文件, +728行)，同时从旧的HcclCommAicpu巨型类中删除63行A5专用代码。hccl侧仅增加4行DevType判断。后续无revert，多个commit继续在CollCommAicpu上添加功能，证明架构决策正确。关键判断点: A5芯片天然走V2路径(IsNewDevice()返回true)，为它写的代码就应该放在next下。

Case BF-7 (hccl:3e9fbbd): A3 IndependentOp路由修复。修复后A3(910_93)即使没有设置HCCL_INDEPENDENT_OP环境变量也走新流程(IndependentOp)。这说明V2路径的覆盖范围正在从"只有A5"向"A3也走"扩展。开发者在scatter_op.cc中特意加了注释"调用位置有特殊要求，不要变化"——这是将更多芯片导向V2的趋势性证据。

### 支持V1路径的案例

Case FT-6 (hcomm:ebb54874, 同日revert hcomm:0c033874): coll comm & orion comm mix-running。37文件+1140行的大型功能，试图跨越V1/V2边界实现新老算子混合运行，同日10小时后被完整revert。三个致命问题: (1)全局数组改static局部变量+lazy init导致多线程竞态; (2)删除hcomm_res_mgr.cc/h架构变更幅度过大; (3)代码自标"临时方案"。截至分析时点未重新引入，说明跨V1/V2边界的功能需要根本性重新设计。

Case CL-2 (hcomm:c5709dd4): HcclEngineCtxDestroy API。29文件+808行，必须同时在legacy路径和next路径实现+独立UT(两个测试文件分别测试V1和V2路径的Destroy)。这个case证明: 在双路径共存期间，新增公开API的成本是单路径的2倍，因为你无法假设用户走哪条路径。

Case CL-1 (hcomm:d0d5cce1): multi process per rank。30文件全部修改在legacy/目录下，为每个rank支持多进程的端口管理升级。这是legacy层仍为新功能开发场所的直接证据。

### 判断标准

"V2不够用"的三个场景:
1. 目标芯片不是910_95(A5)——当前V2的硬件覆盖范围就是A5。对910_93(A3)虽然BF-7显示在扩展，但大部分A3功能仍走V1(Case AA-2/AA-3的A3 AIV适配全在algorithm/impl中做，通过旧的coll_alg_operator.cc注册)。
2. 修改的功能需要同时影响legacy和next路径——如CL-2的API新增。如果只能择一，先实现legacy路径(它是当前主力，Phase 8.7: legacy占修改40%)。
3. 修改的是communicator_impl.cc等V1核心文件中与芯片无关的通用逻辑——如CL-1的端口管理，这类逻辑在legacy中且影响所有芯片，不适合仅在V2中实现。

"应该走V2"的判断:
1. 目标是910_95(A5)芯片且功能与旧芯片无关——如FT-2的A5 AICPU communicator。
2. 新建的执行器/算法模块——新建的子类应继承InsCollAlgBase(V2), 已有60个注册vs V1仅4个。
3. 涉及可序列化资源上下文(AlgResourceCtxSerializable)或TopoInfoWithNetLayerDetails的拓扑信息——这些是V2独有能力。

### 推荐默认选择

如果目标芯片是A5且功能不需要向下兼容旧芯片: 走V2路径，在framework/next下实现。
如果需要覆盖多芯片(含A3/旧芯片): 先在legacy中实现(保证功能可用)，待V2覆盖范围扩大后迁移。
绝不要在一个commit中同时大幅修改legacy和next路径(FT-6的教训: 37文件跨V1/V2即revert)。

### 决策树

```
你的变更目标是什么芯片?
  仅A5(910_95):
    功能是否完全独立于旧芯片?
      是 → V2路径(framework/next下新建模块)
           参考FT-2: 新建CollCommAicpu, 从旧巨型类中删除A5专用代码
      否(需要与旧芯片共用的通用逻辑) → 双路径实现
           参考CL-2: legacy和next都实现, 独立UT
           先实现legacy路径(它是主力)
  包含A3(910_93)或更旧芯片:
    是否新建执行器/算法子类?
      是 → 用V2基类InsCollAlgBase, 在注册时限定芯片条件
           参考AA-3: SelectAlgfor91093方法, For91093Executor后缀
      否(修改已有文件) → 在legacy路径中修改
           参考CL-1: 30文件全在legacy/
  跨越V1/V2边界的大功能?
    → 分步合入, 绝不一次性all-in
      绝不自标"临时方案"后合入(RV-2教训)
      影响op_base.cc的改动需feature flag
      参考FT-6: 37文件跨V1/V2, 10小时后revert
```

---

## 12.2 何时在hccl层实现 vs 下沉到hcomm

### 核心矛盾

hccl是算子层(编排)，hcomm是通信框架层(实现)——这个职责划分看起来清晰，但实际边界模糊。hccl通过C ABI边界调用hcomm(165处static_cast<HcclResult>(HcommXxx(...))分布在27个文件中)，int32_t→HcclResult的转换无校验。两仓独立编译、独立发布，接口通过弱符号(__attribute__((weak)))解耦。但"什么逻辑属于编排、什么逻辑属于实现"在实践中没有明确的判断标准。

算法选择(Selector)就是一个典型的模糊地带: hccl层有显式的AutoSelector类体系(17个Selector类, 22种算法)，hcomm legacy层也有独立的selector体系(base_selector + 8个算子selector子类)。FT-1(hccl:549c4537)和FT-5(hcomm:0d9239b5)分别在两层各自引入了概念相同但实现独立的selector remake——两层都有NetLayerDetails/TopoInstDetails结构体，但枚举值不一致(hccl的Level0Shape: CLOS=0/MESH_1D=1/MESH_1D_CLOS=2; hcomm的Level0Shape: MESH_1D=1/MESH_2D=2/CLOS=3/MESH_1D_CLOS=4)。这种"概念对齐但枚举值不一致"是跨仓独立演进的典型代价。

另一个模糊地带是CCU调度参数(loopnum等)。CCU微码运行在device侧，其template定义在hccl(hccl:ops/xxx/template/ccu/)，但CCU的context/instruction/template三层在hcomm legacy层(legacy/unified_platform/ccu_context/ccu_instruction/ccu_temp/)也有一套独立实现。FT-3(hcomm:fa8ed355)的double die算法就是在hcomm legacy的CCU体系中实现的，与hccl的CCU template体系概念相似但架构独立。

### 支持在hccl层实现的案例

Case FT-1 (hccl:549c4537, 220文件): new selector。整个Selector架构重构都在hccl层完成: 新增TopoInfoWithNetLayerDetails(继承TopoInfo)、将HcclExecOp拆分为Selector+HcclExecOp两个独立阶段、所有14个算子的Selector虚方法签名同步修改。hcomm层没有参与这个变更。理由: Selector的职责是"从多种算法中选择一个"，这是编排层面的决策，不涉及通信实现细节。Selector不需要知道数据怎么搬运，只需要知道拓扑形状和数据量。

Case BF-9 (hccl:80a70729): CCU模式scatter_mesh1d地址设置时机修复。修复在hccl模板层(template/ccu)，因为CCU微码的参数编排(地址设置/loopnum计算)属于hccl的Executor→Template链路，不经过hcomm。CCU微码的编排逻辑天然属于hccl层——它是算子的device侧实现，而非通信框架的职责。

Case FT-4 (hccl:3e916555): Host DPU reducescatter。DPU模式下Op→Executor→Template两层全在hccl中实现，跳过hcomm。DPU的执行模型与device-side本质不同(Host CPU + Host NIC vs AICPU/CCU + Device NIC)，因此DPU算子的编排逻辑不应下沉到hcomm——hcomm的框架层设计假设了device-side的执行模型。

### 支持下沉到hcomm的案例

Case OPT-1 (hcomm:fd9d488c): GroupSendRecv批量编排优化。优化在hcomm framework层(新增executor)，同时删除hcomm framework层逐个执行的旧代码。这个优化涉及通信框架层面的执行策略改进(从逐个SendRecv改为批量编排)，是通信实现细节而非算法选择，因此属于hcomm。

Case FT-5 (hcomm:0d9239b5): alg selector remake。hcomm legacy层建立了独立于hccl的selector体系(base_selector + 8个算子selector)。为什么hcomm也需要selector? 因为legacy路径的调用链不经过hccl的Selector——legacy路径从communicator_impl直接进入hcomm的service层，在那里做算法选择。这个case揭示了一个重要事实: hccl的Selector只管V2路径(IndependentOp)，V1路径的算法选择在hcomm内部完成。

Case RF-1 (hccl-dev:c696e52f + hcomm-dev:ba56e500): 调整算子入口到算子仓。采用"thin wrapper + Inner suffix"模式，将算子API入口从hcomm迁移到hccl。hccl新增91行wrapper(每个API一行转发)，hcomm 109文件批量改名(HcclBroadcast → HcclBroadcastInner)。这个case定义了正确的边界: 算子的外部API入口应该在hccl中(即使只是一行转发)，实际实现在hcomm中。

### 跨仓不对称性的量化证据

Phase 8跨仓分析: 62条跨hccl/hcomm边界的commit中，Feature类commit的文件修改严重不对称——hccl改1-9文件，hcomm改18-110文件(约1:10)。API变更平均需要2-3轮迭代才能稳定(Case XR-4: 同一API Changes经历了3轮提交含1次revert)。hccl中仅2个commit触及hcomm*文件，耦合点集中在hcomm_primitives.h等少数头文件。

这意味着: hccl层是"薄的决策层"，hcomm层是"厚的实现层"。新功能的代码重心几乎必然在hcomm侧。hccl侧的修改应该尽量少(FT-2: hccl 1文件+4行, hcomm 26文件+728行)。

### 判断标准

回答第一个问题——算法选择逻辑(22种算法)属于hccl还是hcomm?

答案: 两边都有，但职责不同。
- V2路径(IndependentOp)的算法选择在hccl层的AutoSelector中完成，基于拓扑形状+数据量+硬件能力做决策。
- V1路径(legacy)的算法选择在hcomm legacy层的base_selector中完成，逻辑独立于hccl。
- 新增算法时，如果走V2路径，只需在hccl的AutoSelector中添加分支；如果需要同时支持V1路径，还需要在hcomm的base_selector中添加对应分支(FT-1 + FT-5的pattern)。

回答第二个问题——CCU调度参数(loopnum等)应该在哪层设置?

答案: 取决于CCU模板属于哪个体系。
- hccl的CCU template(src/ops/xxx/template/ccu/): 参数在hccl模板层设置(BF-9的scatter_mesh1d就在hccl模板中修复)。
- hcomm legacy的CCU体系(legacy/unified_platform/ccu_*): 参数在hcomm legacy层设置(FT-3的double die算法在hcomm的CCU context中设置)。
- 两套CCU体系并行存在，概念相似但架构独立。新的CCU算法倾向于在hccl模板层实现(Phase 3: hccl有28个CCU template文件，5280行)。

### 推荐默认选择

"该在哪层实现"的核心判断依据: 这个逻辑是否需要知道通信链路的具体实现细节(socket/RDMA/HCCS)?
- 不需要(只关心拓扑形状、数据量、芯片类型) → hccl层
- 需要(涉及链路建立、buffer管理、设备资源分配) → hcomm层

### 决策树

```
你要实现的功能类型?
  算法选择/分派逻辑:
    走V2路径(IndependentOp)?
      是 → hccl层(AutoSelector子类)
           参考FT-1: 220文件selector重构全在hccl
      否(V1/legacy路径) → hcomm legacy层(base_selector)
           参考FT-5: 25文件selector remake在hcomm
    两个路径都需要?
      → 两层分别实现, 概念对齐但实现独立
         注意: 枚举值可能不一致(FT-5: 两层Level0Shape枚举值不同)
  CCU微码/模板:
    新建CCU算法?
      hccl CCU体系(V2, src/ops/xxx/template/ccu/) → hccl层
      hcomm legacy CCU体系(legacy/unified_platform/) → hcomm层
    修复已有CCU bug → 在bug所在的层修复
         参考BF-9: hccl模板层的地址时机问题在hccl修复
  通信链路/资源管理:
    → hcomm层(framework或platform)
       参考OPT-1: 批量编排优化在hcomm framework
       参考FT-7: 对称内存管理在hcomm framework
  新算子Op入口:
    → hccl层(thin wrapper)
       参考RF-1: 算子API入口在hccl, Inner实现在hcomm
  新芯片通信域管理:
    → hcomm framework/next层
       参考FT-2: CollCommAicpu在hcomm framework/next下新建
       hccl侧只加DevType判断(FT-2: hccl仅4行)
```

---

## 12.3 何时用深继承 vs 扁平实现

### 核心矛盾

hcomm和hccl展示了两种截然不同的面向对象风格:
- hcomm(Enterprise OOP): 168个pure virtual方法，573个override方法，继承深度6-7层(如ExecutorBase→NHRBase→AllReduceNHROneshot)，Template Method模式广泛使用，63个Manager类，Pimpl模式(75个文件)。
- hccl(Modern C++): 8个pure virtual方法，152个override方法，继承深度2-3层(如InsCollAlgBase→InsV2AllReduceSoleExecutor)，编译期类型注入via模板参数，显式Registry宏注册(173处)。

核心矛盾在于InsCollAlgBase(V2)的28个子类——这到底是合理的多态扩展还是过度设计? 每新增一种算法就需要新建一个子类文件，但子类之间很少共享逻辑(各自独立)。与此同时，scatter是唯一仍使用V1 ExecutorBase(继承深度2-3层)的算子，而scatter恰恰被修改了20次(所有算子中最多)。这引出了一个深层问题: scatter频繁修改是因为V1的抽象不稳定，还是因为scatter本身的业务复杂性?

要理解这个矛盾，需要区分两种不同的"深继承"场景:
1. 框架级继承(hcomm): 定义扩展点，约束子类行为。168个pure virtual就是框架对子类的契约。继承深度反映了抽象层次的精细度。
2. 算子级继承(hccl): 复用计算逻辑。InsCollAlgBase的28个子类主要是"具体算法的容器"，继承只是为了获得CalcRes/Orchestrate的框架调用链。

### 支持深继承的案例

Case FT-1 (hccl:549c4537): TopoInfoWithNetLayerDetails继承TopoInfo。这是继承的经典正确用法: 新结构体需要扩展旧结构体，但旧代码中只需要TopoInfo的地方不受影响。序列化采用"尾部追加"策略——基类字段照常序列化，子类新增字段append到末尾。继承在这里提供了向后兼容性，而非代码复用。

Case FT-3 (hcomm:fa8ed355): double die算法。4算子x3层=12个新文件，严格遵循context+instruction+template三层。每个新算法都继承自对应的base class(CcuInstruction/AlgTemplateBase)。这里的继承提供了Template Method框架——base class定义了Algorithm()的生命周期(InitResource→LoadArgs→PreSync→Do→PostSync)，子类只需填充特定步骤。12个新文件虽然看起来数量多，但每个文件职责清晰、相互独立，维护时不需要理解其他子类的实现。

Case AA-2 (hcomm-dev:88f70272): A3 AIV适配。新增3个executor类(CollBroadcastMeshAivFor91093, CollAllGathervMeshAivFor91093, CollReduceScatterVMeshAivFor91093)，通过继承已有base获得框架能力，通过_for_910_93后缀命名保持清晰。这是"为新芯片适配而继承"的标准模式——继承的价值在于复用框架的初始化/资源管理/错误处理逻辑，子类只关注芯片差异化的算法实现。

### 支持扁平实现的案例

Case FT-4 (hccl:3e916555): Host DPU reducescatter。新建独立的Op目录和Executor(非继承扩展)，因为DPU执行模式与device-side本质不同(两层vs三层)。reduce_scatter_op.cc有683行包含完整的Op层入口(参数校验→拓扑信息获取→算法选择→资源分配→执行)，DPU Executor(209行)不继承InsCollAlgBase也不继承ExecutorBase，而是使用简化的独立类。这个case的判断依据是: 当子类与基类的执行模型差异太大(DPU跳过hcomm, 两层vs三层路径)，继承带来的约束大于收益。

scatter算子: V1 ExecutorBase的4个子类(ScatterCommExecutor, ScatterMeshExecutor, ScatterSingleExecutor, ScatterRingFor91093Executor)。scatter是唯一仍使用V1 ExecutorBase的算子，而V2 InsCollAlgBase已有28个子类且覆盖了其他所有算子。scatter被修改了20次(Phase 1统计)，是最活跃的算子。scatter为什么还在V1?

从证据链推断scatter留在V1的原因:
1. scatter正在经历执行模式迁移(Case XR-2: "scatter switch to aicpu process"，从device launch切换到AICPU进程)。迁移期间同时做V1→V2重构风险叠加。
2. scatter的DPU版本(FT-4)需要完全不同的执行模型。如果scatter同时支持device-side(V2路径)和DPU(独立路径)，需要处理两套不同的Executor，迁移工作量更大。
3. scatter的Op入口(scatter_op.cc)有复杂的5级fallback路由(Case BF-7)，加上结构体指针偏移的手动计算，技术债密度高。迁移V2需要同时清理这些技术债。

scatter留在V1是"时机不对"而非"V1更好"。频繁修改(20次)恰恰说明V1的抽象不够稳定——V1 ExecutorBase只有1个纯虚方法(Orchestrate)，过于宽松的接口意味着每次修改都可能影响执行行为而编译器不会警告。相比之下，V2 InsCollAlgBase有3个纯虚方法(CalcAlgHierarchyInfo+CalcRes+Orchestrate)，强制子类显式实现资源计算和层级信息，编译时就能捕获遗漏。

### 28个子类是合理抽象还是过度设计?

从Phase 3和Phase 4的数据来看:

支持"合理抽象"的证据:
- 28个子类覆盖了5种执行模式x多种拓扑x多种数据量的组合，每个子类解决特定场景的编排问题。
- 子类之间相互独立(很少有跨子类共享的逻辑)，修改一个子类不影响其他子类。
- 通过REGISTER_EXEC_V2宏注册，新增算法只需新建子类文件+注册一行，不修改任何已有代码(开闭原则)。
- 全仓60个V2注册分布在14个文件中(Phase 3统计)，命名清晰(Sole/Parallel/Concurrent/Sequence2Die等)。

支持"需要警惕"的证据:
- alg_param.h被修改14次是"变更放大器"(Phase 1)，每次修改引发联动编译。如果基类成员(InsCollAlgBase的protected成员)发生变化，28个子类都需要重新编译。
- 28个子类的继承层数只有1层(直接继承InsCollAlgBase)，没有形成真正的深继承树。这说明28个子类的"继承"更像是"接口实现"而非"行为特化"——如果是Go语言，这些子类就是实现了同一个interface的28个struct。
- FT-1(220文件)在修改Selector虚方法签名时，所有14个算子的Selector必须同步修改——这种"一改全改"的联动成本随子类数量线性增长。

判断: InsCollAlgBase的28个子类是合理的。理由不是"多态好"而是"独立性好"——每个子类独立、可替换、通过宏注册动态绑定，新增不需要修改已有代码。这种设计的成本(子类数量多、联动编译)在当前规模(28个)是可接受的。但如果子类数量继续增长(比如到100+)，应考虑用编译期策略注入(模板参数)替代运行时多态，hccl已经在部分Executor中做了这种实践(如InsV2AllReduceSoleExecutor是类模板，通过模板参数注入AlgTopoMatch策略)。

### 推荐默认选择

新增算法执行器: 继承InsCollAlgBase(V2)，一个子类解决一个特定场景。不要在已有子类中用if/else分支处理多种场景——这会破坏子类的独立性和可替换性。

新增数据结构扩展: 使用继承+尾部追加序列化(如FT-1的TopoInfoWithNetLayerDetails)，而非修改已有结构体字段。

新执行模式与现有模型差异太大: 新建独立类，不继承已有基类(如FT-4的DPU算子)。判断标准: 如果基类的CalcRes/Orchestrate/KernelRun三阶段生命周期对新场景不适用，就不要强行继承。

### 决策树

```
你要新增什么类型的代码?
  新的集合通信算法(device-side):
    是否与已有算子的执行模型相同(三层路径)?
      是 → 继承InsCollAlgBase(V2), 新建子类
           注册: REGISTER_EXEC_V2(algName, ExecClassName)
           参考AA-2: _for_910_93后缀, 标准继承扩展
           参考AA-3: 新建Executor+selector分支插入
      否(执行模型根本不同) → 独立实现, 不继承
           参考FT-4: DPU算子新建独立Op目录和Executor
  新的CCU算法:
    → 遵循context+instruction+template三层文件结构
       参考FT-3: 12个新文件, 每层继承对应base class
       在CcuInstType枚举末尾追加(不在中间插入)
  核心参数结构体扩展:
    → 继承+尾部追加, 不修改已有字段
       参考FT-1: TopoInfoWithNetLayerDetails继承TopoInfo
       保持向后兼容(旧代码只用基类指针)
  修改已有Executor/Selector:
    修改范围是否影响所有子类(如基类签名)?
      是 → 评估联动编译成本
           FT-1: 220文件中180个是签名联动修改
           必须配合全算子x全模式的回归测试(FT-1次日暴露BF-9)
      否(只改特定子类) → 直接修改
           参考BF-5: 只改1文件1行(GetThreadNum返回值)
  scatter算子的修改:
    → 当前仍在V1(ExecutorBase), 预期会迁移到V2
       修改时注意: 5级fallback路由条件顺序(BF-7)
       手动指针偏移计算(sizeof)必须与结构体字段同步
       scatter是最活跃算子(20次修改), 初始设计不稳定
```

---

## 12.4 新功能放在哪一层(algorithm / framework / platform)

### 核心矛盾

hcomm的三层架构(algorithm / framework / platform)各有明确的设计职责:
platform层提供C ABI的通信原语(comm_primitive)和硬件资源管理(transport/resource/task)，
framework层提供通信域管理(communicator)和资源编排(V1 RankGraph / V2 CollComm)，
algorithm层提供具体的集合通信算法实现(AlgTemplateBase继承体系)。

但现实中这三层的边界并不稳固。新功能该放在哪一层，取决于功能的抽象层次、复用范围和对已有代码的影响。问题的根源在于: hcomm的三层不是一次性设计出来的，而是在持续演进中逐步分化的。legacy层占修改40%这个事实说明，层间边界至今仍在调整。一个典型的灰色场景是: 你要新增一个跨Host-Device的资源管理能力(比如对称内存)——它既涉及platform层的硬件资源操作(VMM API调用)，又涉及framework层的生命周期管理(引用计数+通信域集成)，还涉及algorithm层的算法适配(zero-copy标记)。放在哪一层，决定了谁来"拥有"这个功能的核心逻辑。

### 正面案例: 自顶向下的层次选择

Case FT-7 (hcomm:4c779e13, symmetric memory) 展示了一个做对了的层次选择。对称内存的核心管理器SymmetricMemory(612行)放在framework层的symmetric_memory/目录下，负责VA空间预留、注册、映射和查找。引用计数(PaMappingInfo.refCount)也在framework层管理。platform层的VMM API调用被封装在SymmetricMemoryAgent(249行)中——Agent是platform级别的操作，但由framework层持有和调度。algorithm层只需要设置一个isZeroCopy标记。

这个选择之所以正确，是因为对称内存的核心复杂度在于生命周期管理(何时预留VA、何时注册物理内存、何时释放)，而非硬件操作本身。生命周期管理是framework层的天然职责。将核心逻辑放在framework层，使得多个algorithm层的算子(AllGather/AllReduce/Broadcast/ReduceScatter的ring_zerocopy系列)可以通过简单的isZeroCopy标记复用同一套基础设施。

Case LC-1 (hcomm:e63840b6, alltoall symmetric memory) 验证了这个层次选择。当AlltoAll算子需要对称内存支持时，algorithm层只需新增AlltoAllFullMeshSymmetricMemory类(继承AlgTemplateBase)，framework层的PrepareSymmetricMemory扩展为遍历所有topology level——核心管理逻辑完全复用FT-7建立的framework层基础设施。

Case PL-3 (hcomm-dev:0e82e543, thread跨通信引擎导出) 是另一个典型案例。Thread的跨引擎导出(CPU_TS↔AICPU_TS)放在framework层的ThreadMgr中，新增HcclThreadExportToCommEngine()公共API。这个功能不属于platform层(它不是硬件操作)，也不属于algorithm层(它不是特定算法的逻辑)——它是资源管理能力，属于framework层的职责。

### 反面案例: 层次选择错误导致的问题

Case FT-6 (hcomm:ebb54874, mix-running) 展示了跨层修改的风险。这个功能同时修改了framework/next/comms、framework/op_base和legacy/framework三个层面，37个文件+1140行。它被同日revert的直接原因可能是测试失败，但深层原因是: mix-running的核心逻辑(新老通信域如何协调)没有找到一个清晰的层来"拥有"。op_base.cc承担了560行新增代码——这个文件是算子执行编排的核心，往里塞入混跑的资源协调逻辑违反了单一职责原则。

Case BF-8 (hcomm:fa61b9fc, Thread API bugfix) 暴露了platform和framework层边界模糊的代价。Thread的创建需要跨越framework层(ThreadMgr管理句柄映射)和platform层(kernel binary加载、device launch)。当Thread API出bug时，修复涉及18个文件、跨越两个层——EnsureKernelBinLoaded(平台级操作)不得不加到framework层的hcomm_c_adpt.cc中(C API适配层)。这说明当功能天然跨层时，应该明确一个"拥有者层"来承担协调职责，而非让逻辑散落在两层中。

### platform层C ABI何时该扩展

platform层的comm_primitive是hccl→hcomm的C ABI边界(extern "C"包装 + void*句柄 + reinterpret_cast委托)。扩展comm_primitive意味着修改跨仓公开接口，影响范围极大。

判断标准: 只有当新能力是所有通信算法的共同基础设施(如transport建链、Notify收发、内存注册)时，才应扩展comm_primitive。如果能力是特定算法或特定场景专用的(如对称内存的VA管理、Thread跨引擎导出)，应在framework层提供新的C API(如HcclThreadExportToCommEngine)，而非修改comm_primitive。

Case AA-5 (hcomm:4afb1c39, RoCE flush适配) 是扩展platform层的正确案例: FlushHandle新增GetLbMax()能力检测，这是transport级别的硬件能力查询，天然属于platform层。修改只有5个文件36行——platform层扩展应该是这种规模。

### algorithm层 vs framework层的初始化逻辑归属

LC-3 (hcomm:9167c142) 将"是否需要更新资源"的判断从host侧service层(framework)移到device侧kernel(algorithm)。修改的理由是: 使用方最清楚自己的当前状态。跨Host-Device边界传递flag来控制行为是脆弱的。这个案例确认了: 初始化的"何时"判断应靠近使用方(algorithm层)，初始化的"怎么做"实现应留在管理方(framework层)。

### 推荐默认选择

新功能默认放在framework层。原因: (1) framework层是三层中复用度最高的层，功能放在这里最容易被多个算法共享; (2) FT-7、PL-3等成功案例证明framework层能有效承载资源管理类新功能; (3) platform层扩展影响C ABI边界(跨仓)，algorithm层扩展只服务单一算法，都不是好的默认选择。

### 决策树

```
新功能需要修改C ABI边界(comm_primitive)吗?
  是: 该功能是所有通信算法的共同基础设施吗(transport/notify/mem级别)?
    是 → platform层(参考AA-5: 5文件36行的精确扩展)
    否 → 不要修改comm_primitive，在framework层新增C API
  否: 该功能需要被多个算法/算子复用吗?
    是 → framework层(参考FT-7: symmetric memory管理器)
         附加: 如果涉及硬件操作，用Agent模式封装platform调用
    否: 该功能是特定算法的专有逻辑吗?
      是 → algorithm层(参考LC-1: AlltoAll专用的对称内存算法类)
           附加: 确保framework层提供足够的基础设施接口
      否(不确定) → 先放framework层，后续根据复用模式再迁移
          警示: 避免FT-6的教训——跨三层的大规模修改极易失败
```

---

## 12.5 何时做保护性编程 vs 信任调用者

### 核心矛盾

hcomm的防御性编程密度极高: CHK_PTR_NULL出现2683次，CHK_RET出现14943次，几乎每个函数入口都有参数校验，每个函数调用返回值都被检查。这种100%覆盖的风格在Phase 2.1中被识别为hcomm的核心convention。

与此同时，hccl采用了截然不同的策略: CHK_PTR_NULL仅159次，参数由Selector预校验，信任下游不会破坏状态。两个紧密协作的仓库采用了两种极端策略——这本身就说明"该做多少保护性编程"没有唯一正确答案。

真正的矛盾在于: 保护性编程的收益(捕获意外错误、快速定位问题)和代价(性能开销、代码膨胀、错误传播链过长)之间的权衡，在不同的层次和场景下有不同的最优解。一个9层的CHK_RET传播链(API→Op→Selector→Executor→hcomm algorithm→framework→platform)意味着一个底层错误会在9个层面都产生日志——这是信息冗余还是诊断便利?

### 正面案例: 保护性编程拯救了局面

Case BF-3 (hcomm:994390df, 快恢失败) 是防御性编程缺失导致问题的典型案例。重执行状态机的初始状态未在RegisterAgentRetryMachine/RegisterServerRetryMachine中显式设置——构造retryCtx后状态为未定义值。修复方法是添加AddSetRetryState调用，在注册时显式设为RUNNING。这个bug的根因是"信任了默认值"——开发者假设retryCtx创建后状态默认是合理的，但事实上C++不保证未初始化变量有确定值。

Case LC-2 (hcomm:3315532a, memDesc生命周期) 展示了另一种形式的防御性设计。RegisterMemory在重复注册时(HCCL_E_AGAIN)也返回memHandle——即使调用方式"不标准"(重复注册理论上不应该发生)，也要返回有用的结果而非让调用者手足无措。这种"对不规范输入也给出合理响应"的设计哲学，超越了简单的CHK_PTR_NULL式参数校验。

Case BF-6 (hccl:b779699, missing symbols) 证明编译期防御同样重要。channel.cc中GetTopoTypeByLink引用了Host-only API(HcclRankGraphGetTopoInstsByLayer)，在AICPU编译时链接失败。修复方法是#ifndef AICPU_COMPILE条件编译保护。这类编译期防御是零运行时开销的——没有理由不做。

### 反面案例: 保护性编程的边界

Case OPT-3 (hcomm:a98eb47a, AICPU微优化) 直接量化了保护性编程的性能影响。这个commit在15个文件中做了约30处UNLIKELY宏标注——将错误检查分支(if (ptr == nullptr)等)标记为不太可能命中。这些检查本身是正确的(100% CHK_PTR_NULL覆盖是强制规范)，但编译器默认按50%概率预测每个分支，导致热路径的指令缓存效率下降。UNLIKELY宏是一种折中: 保留保护性检查，但告诉编译器"这些分支几乎不会走到"。

Case BF-1 (hcomm:75659d24, 重执行时序) 揭示了CHK_RET无法捕获的错误类型。这个bug的根因是两条语句的执行顺序错误——先调用ResetOpRetryException(清理stream)，后重置dfxExtendInfo_状态。CHK_RET只检查函数返回值(是否HCCL_SUCCESS)，无法检测"操作顺序是否正确"。2行语句重排修复了问题，但没有任何CHK_RET能预防它。

Case BF-7 (hccl:3e9fbbd, A3 bug) 展示了CHK_PTR_NULL无法捕获的另一类问题: 指针偏移计算错误。结构体新增字段后，手动的sizeof偏移计算未同步更新，导致reinterpret_cast指向了错误的位置。指针非空，CHK_PTR_NULL通过，但指向的内容是错误的。

### 内部接口 vs 外部接口的防御力度差异

hcomm作为框架层，被多个不同的调用者(hccl算子层、用户API、内部模块间)使用。每个层边界都做参数校验是合理的——框架不能假设所有调用者都是正确的。2683次CHK_PTR_NULL覆盖146个文件，是框架层的合理密度。

hccl作为算子层，调用链是单向的(Op→Selector→Executor)，Selector预校验后数据流向是确定的。159次CHK_PTR_NULL覆盖24个文件——只在入口(Op层)和跨仓边界做校验，内部信任上游传递的参数。

这个自然形成的分工是合理的: 跨仓边界(hccl→hcomm的C ABI)和API入口(HcclAllReduce等)是防御性编程的"高价值区"，内部模块间的调用是"低价值区"。

### 性能关键路径的处理

CHK_RET是否应该在性能关键路径被编译期移除? OPT-3给出了实践答案: 不移除检查，而是用UNLIKELY标注。目前hcomm中UNLIKELY使用了281次——这个数字远小于CHK_RET的14943次，说明UNLIKELY标注还有大量空间可以应用到algorithm层和legacy层的热路径中。

编译期移除CHK_RET(通过#ifdef NDEBUG等)的风险在于: AICPU/CCU设备侧的错误如果不被检测，症状(死锁/超时/数据错误)与根因的距离极远(Case BF-5: GetThreadNum返回值错误导致CCU调度死锁)。保留检查但优化分支预测是更安全的策略。

### 推荐默认选择

保持当前的100% CHK_RET/CHK_PTR_NULL覆盖作为基线，在三个维度做差异化:

1. 跨层边界(hccl→hcomm C ABI, framework→platform comm_primitive): 强制校验，无例外
2. 设备侧热路径(algorithm层的kernel循环体): 保留检查 + UNLIKELY宏标注
3. 编译目标差异: Host/Device双目标编译的函数必须有AICPU_COMPILE条件编译保护

不推荐在任何层面移除运行时检查。BF-3(状态机未初始化)和BF-7(指针偏移错误)证明，看似多余的检查有时是发现隐性bug的唯一线索。

### 决策树

```
你在写什么类型的代码?
  跨仓/跨层边界的公共接口:
    → 强制: CHK_PTR_NULL + RPT_INPUT_ERR + 完整参数校验
       参考: hcomm每个层边界的100%覆盖
  设备侧热路径(AICPU/CCU的算法循环体):
    → 保留CHK_RET + 标注UNLIKELY(参考OPT-3: ~30处UNLIKELY标注)
       但不要在循环体内做重复的参数校验(循环外校验一次即可)
  状态机/FSM的状态转换:
    → CHK_RET不够: 需要显式设置初始状态(参考BF-3)
       考虑assert或static_assert验证状态转换的合法性
  内部模块间调用(同层内的helper函数):
    → CHK_RET保留(保持一致性)，但CHK_PTR_NULL可以省略
       如果上游已经校验过参数，内部函数可以信任
  Host/Device双编译目标的函数:
    → 强制: #ifndef AICPU_COMPILE保护Host-only API调用(参考BF-6)
       编译期防御是零开销的，必须做
  不确定:
    → 默认做保护性编程(宁可多检查不要漏检查)
       BF-3和BF-7的教训: 你以为多余的检查，在某个你没想到的场景下可能是救命的
```

---

## 12.6 何时重构legacy代码 vs 在next中重新实现

### 核心矛盾

hcomm的legacy层在2026年1月orion→legacy大规模重命名(commit 62ba4500, 2432文件)后，不是在衰退而是在爆发。统计数据显示: 从2026-02起legacy占hcomm总commit的约40%。88.9%的legacy commit无芯片关键词——legacy不是旧芯片专用代码，而是当前生产环境的主力代码路径。

同时，framework/next目录中有17个TODO标记，V2(pImpl委托+CollComm/MyRank)还在建设中。97条跨层commit中真正的功能迁移(legacy→next)仅13条(13%)，大多数是功能同步开发或跨层bugfix。

这构成了一个深刻的矛盾: legacy层既是bug密度最高的区域(legacy/service每commit触及45个文件，耦合度极高；communicator_impl.cc 47次修改，是最高频修改文件)，又是新功能开发仍然不得不使用的现实路径(CL-1展示了30个文件全在legacy/的新功能开发)。你面前有一个legacy模块的bug——修它，意味着在一个高耦合、高风险的代码库中投入精力；不修它等next替代，但next的建设进度未知(17个TODO)且功能迁移比例仅13%。

### 正面案例: 在legacy中修复/开发是正确的

Case CL-1 (hcomm:d0d5cce1, multi process per rank) 是legacy层新功能开发的标杆案例。30个文件全在legacy/目录下——从硬编码端口号(60001)升级到配置化的per-device端口映射。这个功能不可能"等next替代"再做: 多进程场景的端口冲突是生产环境的实际痛点，而next层的communicator模块尚未成熟到能承接socket管理的职责。

CL-1成功的关键在于: 它在legacy层做了"渐进式改进"(删除硬编码→引入配置化)，而非"架构级重构"。communicator_impl.cc(最高频修改文件，47次修改)的变更是受控的——在Init之后追加端口配置步骤，不改变Init本身的流程。

Case BF-3 (hcomm:994390df, 快恢失败) 也属于"legacy中修复是正确的"。快恢(NsRecovery)状态机的bug(初始状态未显式设置)只需3个文件+12/-8行就能修复。这类精准的小修复在legacy中做是效率最高的。

### 正面案例: 在next中重新实现是正确的

Case FT-2 (hccl:12a2c84f + hcomm:dba4d87b, A5 aicpu communicator) 展示了"新芯片走next"的正确模式。A5芯片(910_95)需要独立的AICPU通信域管理类，开发者选择在framework/next/coll_comms/下新建独立模块(CollCommAicpu+CollCommAicpuMgr+AicpuIndopProcess)，同时从旧的巨型类HcclCommAicpu中删除63行A5专用代码。

Case RF-1 (hccl-dev:c696e52f + hcomm-dev:ba56e500, 调整算子入口到算子仓) 展示了从legacy向next迁移的安全模式。采用"thin wrapper + Inner suffix"模式: hcomm侧将HcclBroadcast等函数重命名为HcclBroadcastInner(109文件批量改名)，hccl侧新增只有一行转发的wrapper。不改变任何业务逻辑，只调整架构归属。

### 反面案例: 跨越legacy/next边界的大改失败

Case FT-6 (hcomm:ebb54874, mix-running) 是跨越V1/V2边界失败的典型案例。37个文件+1140行，横跨framework/next/comms、framework/op_base和legacy/framework三个层面。同日被revert。FT-6失败的根本原因不是代码质量，而是违反了一个关键原则: 跨越legacy/next边界的功能变更必须可以增量回退。

### 判断标准

在legacy中修复的三个条件(满足任一):
1. 紧迫性: 生产环境的bug或性能问题，不能等next建设完成。
2. 修改范围可控: 变更集中在legacy层的单一模块内，不跨越legacy/next边界。
3. next层对应功能尚未成熟: 17个TODO说明next层本身还在建设中。

在next中重新实现的三个条件(需同时满足):
1. 新芯片/新场景: 与旧代码的执行模型差异太大，添加分支会使legacy更加臃肿。
2. next层基础设施已就位: CollComm/MyRank等V2基础类已经存在且稳定。
3. 旧代码可以"减法式"清理: 在next层实现后，能从legacy中删除对应的代码。

### 推荐默认选择

在当前共存期，默认在legacy中做维护性修改(bugfix/小功能增量)，在next中做新芯片/新架构的功能开发。不要尝试跨legacy/next边界的大规模重构——FT-6(37文件同日revert)的教训清晰表明这种尝试的失败概率极高。

如果你被要求在两条路径都实现同一个功能(如CL-2的HcclEngineCtxDestroy)，接受双倍工作量的现实，并确保两条路径都有独立的UT覆盖。

### 决策树

```
你要做的是什么类型的变更?
  Bug修复:
    Bug在legacy中还是next中?
      Legacy中 → 在legacy中修复(参考BF-3: 3文件12行精准修复)
           附加: 如果next中有同名函数，检查是否也有同样的bug
      Next中 → 在next中修复
  新功能开发:
    功能面向哪些芯片?
      新芯片(A5/910_95)专有:
        next层的基础设施就位了吗?
          是 → 在next中新建独立模块(参考FT-2: 新建CollCommAicpu)
               附加: 从legacy中删除芯片专用代码(做减法)
          否 → 在legacy中开发，标注TODO待迁移
      旧芯片(A2/A3)相关:
        → 在legacy中开发(参考CL-1: 30文件全在legacy/)
      跨芯片通用:
        是否需要修改legacy和next两条路径?
          是 → 双路径实现 + 双路径UT(参考CL-2: 29文件)
               警示: 预期工作量翻倍; 先实现legacy路径(当前主力)
          否 → 在framework层实现通用能力，供两条路径调用
  重构:
    重构范围跨越legacy/next边界吗?
      是(超过20个文件跨两条路径) → 极高风险，强烈建议拆分
           参考FT-6: 37文件跨边界同日revert
           替代: 用"thin wrapper + Inner suffix"模式渐进迁移(参考RF-1)
      否(仅在legacy或next内部):
        → 在当前层内重构，不跨边界
           参考RF-3: 文件拆分保持行数平衡，不夹带功能修改
  不确定:
    → 默认在legacy中做(它是当前主力，40% commit量证明)
       只有在明确看到next层基础设施就绪时才考虑迁移
```

---

## 12.7 CCU调度 vs AICPU执行: 何时用哪种设备侧模式

### 核心矛盾

hccl/hcomm的集合通信算子有五种设备侧执行模式(Selector五路分派): DPU-only、CCU_MS(CCU master-slave)、CCU_SCHED(CCU schedule)、AIV、AICPU。排除DPU和AIV两个特殊场景，核心抉择落在CCU与AICPU之间。

为什么这是灰色地带? 因为CCU和AICPU各自的优势恰好是对方的劣势:
- CCU模式通过微码(microcode)在硬件调度单元上直接编排通信，消除了Host→Device的反复交互开销。OPT-4(hcomm:8b0dc7b9)的CCU快速下发(SuperFastLoad)进一步把参数缓存在设备端，后续执行连H2D传参都省掉了。性能天花板高。
- AICPU模式通过Host侧编排+Device侧多线程执行，流程与普通C++类似，调试手段丰富(HCCL_INFO/CHK_RET链路完整)，修改灵活。OPT-3(hcomm:a98eb47a)证明AICPU热路径可以通过UNLIKELY宏做到接近CCU的分支预测效率。

但Phase 8.8的统计数据揭示了一个关键不对称: hccl:template/ccu的BUGFIX密度44.4%，而且每个commit平均触达34个文件(广播式修改)。CCU是"低频高影响"的风险区——改一个地方可能波及所有算子的CCU模板。AICPU/AIV的BUGFIX绝对数量更大(80条)，但分布更广、每次修改影响范围更小。

矛盾的本质是: CCU性能好但犯错代价极高，AICPU灵活但性能可能不够。更深层的问题是——CCU微码的编程模型与C++完全不同(CCU_WHILE/CCU_IF是硬件控制指令而非普通控制流)，这意味着开发者在CCU模式下犯错的概率天然更高，而发现错误的成本也更高(症状是死锁/超时，根因是参数错误)。

### 正面案例: 选择CCU是正确的

Case OPT-4 (hcomm:8b0dc7b9): RS & AAV CCU快速下发。将ReduceScatter和AlltoallV纳入CCU SuperFastLoad体系，通过TryFastCcuLaunch路径实现缓存参数复用。这个case展示了CCU模式在成熟阶段的优势: 扩展已有的快速下发框架只需要新增CcuInstType枚举值+FillArgs函数+ccuParamsMappingKey设计。10文件+215行的修改没有被revert，说明在已有CCU基础设施完善的前提下，新增算子的CCU支持是可控的。

Case FT-3 (hcomm:fa8ed355): double die算法。4算子x3层=12个新文件，全部在CCU体系内实现。这里选择CCU而非AICPU的判断依据是: 双die全互联拓扑的通信编排需要硬件级的流水线控制(DataCopy/WaitNotify/SignalNotify的精确时序)，AICPU的软件循环无法做到同样的通信-计算重叠效率。

### 反面案例: CCU带来了高昂代价

Case BF-5 (hccl:992fd5b): CCU NHR template。GetThreadNum()返回值从1改为2，仅修改一个数字。但这1个数字的错误导致CCU调度器资源分配不足，表现为设备侧死锁/超时。根因和症状之间的距离极远——开发者看到的是"通信超时"，需要追溯到CCU模板的线程数参数才能定位。

Case BF-9 (hccl:80a70729): CCU scatter_mesh1d地址设置时机。地址初始化代码放在了CCU_WHILE循环内部而非之前。这在普通C++中无关紧要(循环首次迭代会执行)，但CCU微码中CCU_WHILE/CCU_IF的语义不等同于CPU控制流——CCU_IF(flag_ != 0)分支中的增量偏移逻辑依赖地址已经在循环外正确初始化。

Case RV-8/RV-9 (hcomm:72cdf80e/0c3be05f): loopnum=128。CCU_MS_DEFAULT_LOOP_COUNT从64改到128，仅改1行常量+UT更新。6小时后被revert(验证不足)，17天后才re-revert恢复(完成真实硬件验证)。CCU参数修改的验证周期是周级而非小时级。

Case XR-2 (hccl-dev:0ca13cd6 + hcomm-dev:79e73f3c): scatter从CCU切换到AICPU进程。hccl删除ParseLibraryPath(-99行)，hcomm删除device launch(-88行)。scatter是hccl中被修改最多的算子(20次)，选择从CCU切换到AICPU说明: 当算子正在经历架构迁移且CCU模式的bug密度过高时，降级到AICPU是务实的选择。

### CCU MS vs CCU Schedule两种CCU模式如何选择

从Selector的五路分派逻辑:
- CCU_MS(SelectCcuMsAlgo): 适用于topoLevelNums<=1的简单拓扑(单层网络)，CCU快速下发(SuperFastLoad)目前主要支持CCU_MS模式。
- CCU_SCHED(SelectCcuScheduleAlgo): 适用于多层拓扑(topoLevelNums>1)，需要更复杂的size matrices。

判断标准: 单层拓扑(MESH_1D)且在CCU_MS的size矩阵覆盖范围内→CCU_MS。多层拓扑(多级CLOS/MESH_1D_CLOS)或双die→CCU_SCHED。

### CCU微码修改的安全准则

从BF-5、BF-9、RV-8/RV-9三个案例提炼:

1. 参数影响面评估: CCU模板中的Get*Num()/Get*Count()等方法是全局参数，修改前用Grep统计所有引用点。1个数字或1行常量就能导致全链路故障。
2. 控制流语义: CCU_WHILE/CCU_IF不等于while/if。初始化代码必须在CCU_WHILE外部，循环体内仅放增量逻辑。
3. 验证周期: CCU参数修改需要真实硬件全场景验证，周期是周级(17天)。不要在无法快速回退的时间点提交CCU参数变更。

### 推荐默认选择

新算法的首次实现应优先使用AICPU模式，待功能稳定后再优化为CCU。理由:
- AICPU的调试成本远低于CCU(CHK_RET链路完整，HCCL_INFO可用)
- AICPU的修改风险可控(单文件修改，不会广播式波及所有模板)
- XR-2(scatter从CCU切换到AICPU)和OPT-4(成熟算子才扩展CCU快速下发)共同印证了"先AICPU再CCU"的演进路径

例外: 如果算法需要硬件级通信-计算重叠(如FT-3的double die流水线)，且已有同类型的CCU context+instruction+template可以参考，直接用CCU。

### 决策树

```
你要实现的功能场景?
  新集合通信算法(首次实现):
    是否需要硬件级通信-计算流水线编排?
      是(如double die跨die流水) → CCU模式
           遵循context+instruction+template三层结构
           参考FT-3: 12个新文件, 严格继承base class
           新增CcuInstType枚举值追加在末尾
      否 → 先AICPU模式实现, 功能稳定后优化为CCU
           参考OPT-4: 在已有CCU基础设施上扩展
           先做AICPU验证, 再做CCU快速下发
  修改已有CCU模板参数(threadNum/loopNum等):
    → 高危操作, 必须遵循安全准则:
       1. Grep全量搜索引用点(参考BF-5: 1个数字导致死锁)
       2. 确认修改不影响CCU_WHILE内增量逻辑(参考BF-9)
       3. 预留周级验证窗口(参考RV-8/RV-9: 17天验证周期)
       4. 考虑用环境变量/配置项而非硬编码常量
  已有算子从CCU切换到AICPU(或反向):
    → 跨仓原子提交(参考XR-2: 同作者18秒内提交两仓)
       hccl侧通常是"减法"(删除旧模式代码)
       hcomm侧是"调整"(适配新执行流程)
  CCU_MS vs CCU_SCHED:
    topoLevelNums <= 1(单层网络) → CCU_MS
    topoLevelNums > 1(多层拓扑) → CCU_SCHED
    不确定 → 查看对应算子SelectAlgo方法中的size threshold矩阵
```

---

## 12.8 通信协议修改: 渐进式演进 vs 一步到位

### 核心矛盾

hccl/hcomm的通信协议包含三个核心子系统: Notify同步(三轮握手)、重执行(OpRetry, 22态状态机)、心跳(25ms/50ms周期)。Phase 8.9的统计显示: 重执行21条commit最活跃，心跳8条commit经历了"引入→revert→加开关重做"的完整三阶段，Notify 16条commit从底层offset计算到上层多任务共享持续演进。

矛盾在于: 渐进式修改每次只动一小步，风险可控但容易导致协议碎片化——同一个子系统经历5-10次增量修改后，代码中充满了历史妥协和补丁逻辑(重执行的aicpu_communicator.cc被修改48次，18次是BUGFIX，38%的bug密度)。一步到位则意味着大规模重写，风险极高——FT-6(37文件+1140行跨V1/V2边界mix-running)10小时即revert，RV-1(51文件ARS算法)次日revert。

更深层的矛盾是: 通信协议是分布式系统中最难测试的部分。集群环境中的边界条件(网络抖动、节点故障时序、多线程竞态)极难在开发阶段覆盖。PL-1/PL-2(心跳检测)的"introduce→revert→add-switch-redo"三阶段模式反复出现，本质原因是: 协议功能在单机测试中看起来正确，但部署到集群后暴露的问题只能通过revert止血。

### 正面案例: 渐进式演进成功的范例

Case PL-5 (hcomm-dev:28f18ab5): 简化重执行WaitResume。修改只有9文件+7/-32行(净减25行)，核心决策是: 删除g_isRdmaError全局变量，将链路切换(changeLink)从"仅RDMA错误时触发"简化为"无条件触发"。遵循了"宁可多做恢复动作也不要漏做"的安全原则。

Case PL-4 (hcomm-dev:9ceead3b): topo detect retry。在拓扑发现阶段新增ConnectWithRetry()重试机制。设计精妙之处在于两个保护: (1)总超时时间三等分(不增加总超时); (2)前2次重试的ERROR降级为WARNING(SetErrToWarnSwitch)。"不改变系统整体行为边界，只在内部增加容错"是最安全的协议修改模式。

Case RV-6 (hcomm-dev:1d8e2c14): Revert rts interfaces后引入adapter+动态fallback。核心改进是新增hrtGetPairDeviceLinkTypeRaw()函数，通过dlsym运行时检测新接口是否可用，可用则调新接口，不可用则fallback旧接口。这种"adapter+动态fallback"模式是系统接口迁移的标准做法。

### 反面案例: 一步到位失败的范例

Case PL-1/PL-2 (hcomm-dev:ebfcde5c): HB check加开关重做。心跳算子不一致检测的原始版本没有功能开关，HeartBeatFrame中嵌入了10个tag队列x500条opInfo的庞大结构，心跳帧大小显著膨胀。结果: 2天后被整体revert(RV-7, hcomm-dev:a37e6cf1)。

34天后的重做版做了三个关键改进: (1)inconsistentCheckSwitch默认false，通过环境变量控制; (2)心跳socket buffer大小根据开关动态选择; (3)不一致状态存入inconsistentOpMap_按identifier查询。重做版没有再被revert。

核心教训: 不可关闭的协议功能在出问题时只能整体revert，代价极高。开关的成本(几十行env_config解析)远低于revert的成本。

Case BF-1 (hcomm:75659d24): 重执行时序。即使是2行的修改，如果涉及协议状态机的时序变更，也可能引发严重后果。原始的大型feature commit(d17b1f3a, 34文件+3684行)重构了cache逻辑，间接改变了retry路径的并发访问模式。

### Notify协议修改的安全粒度

安全粒度的判断标准:
- 修改Notify的资源管理(pool/offset/分配): 每次只动一个子问题。notifyPoolMap原子化重构(dc864f28)聚焦在单一问题(并发安全)上，效果稳定。
- 修改Notify的使用方式(新增同步模式): 可以适度大一些。PL-3(hcomm-dev:0e82e543)一次性引入了Thread跨引擎导出。但前提是: 新增独立能力(Export)，不修改已有Notify收发逻辑。
- 修改Notify的帧结构/协议格式: 极高风险，必须带开关。参考RV-7(心跳帧结构变更被revert)。

### 重执行协议变更的验证级别

- 状态机状态转换路径变更(如PL-5): 需要覆盖所有22个状态的转换路径。
- 时序相关变更(如BF-1的语句重排): 需要多线程压力测试。
- 与其他功能的交互(如retry与cache的交互): 需要组合测试。

### 何时引入新Notify类型 vs 复用已有Notify

现有三种Notify类型: RtsNotify(硬件)、BareNotify(空)、EschedNotify(事件)。目前的实践是通过Thread Export体系和Channel抽象来组合使用现有Notify类型，而非在底层扩展类型枚举。不应该引入新Notify类型，除非新的同步语义与现有三种类型完全不兼容。

### 推荐默认选择

协议修改的默认策略是渐进式演进+功能开关:

1. 新协议功能必须带运行时开关(默认关闭)。开关通过HCCL_DFS_CONFIG环境变量控制，在env_config.cc中解析。
2. 每次只修改协议的一个维度。不要在一个commit中同时修改协议的数据结构和接口签名。
3. 系统接口迁移用adapter+动态fallback。RV-6的hrtGetPairDeviceLinkTypeRaw()是标准模式。

### 决策树

```
你要做的协议修改类型?
  新增协议功能(如心跳检测/新同步模式):
    → 必须带运行时开关, 默认关闭
       参考PL-1/PL-2: inconsistentCheckSwitch默认false
       不加开关 = 出问题时只能整体revert(RV-7教训)
  修改重执行(OpRetry)状态机:
    修改状态转换路径?
      → 覆盖22态所有转换路径的测试
         参考PL-5: 简化比增加更安全(删除分支而非新增)
         状态重置顺序: 先清理物理资源, 后清除异常标记(BF-1)
    修改与其他功能的交互(cache/优化路径)?
      → 组合测试: retry ON + 新功能 ON 的交叉场景
  修改Notify底层(offset/pool/资源管理):
    → 每次只动一个子问题
       参考dc864f28: notifyPoolMap原子化, 聚焦并发安全
  修改Notify使用方式(新增同步模式):
    → 可以较大范围修改, 但不要动已有Notify收发逻辑
       参考PL-3: 新增Thread Export能力, 不改现有Notify
  修改心跳帧结构:
    → 极高风险, 必须带开关
       帧大小变更需评估网络带宽影响(RV-7教训)
  系统接口迁移(RTS→ACL等):
    → adapter + 动态fallback(dlsym检测)
       参考RV-6: 先尝试新接口, 不可用走旧接口
```

---

## 12.9 错误处理: 快速失败 vs 容错继续

### 核心矛盾

hccl和hcomm在错误处理哲学上形成了鲜明的二元对立:
- hccl: Fail-fast。任何错误通过CHK_RET立即返回，9层传播链确保错误不被吞没。hccl不知道对端是否存活，不做任何重试。
- hcomm: Fault-tolerant。RDMA CQE错误通过ExceptionHandler异步处理，触发OpRetry的22态状态机。心跳25ms/50ms监测对端存活。重执行子系统21条commit经历了从"默认开启"到"默认关闭(c303e62a)"再到"密集修复后逐步恢复"的完整演进。

这两种哲学并非简单的"选A还是选B"。它们分别适用于不同的系统层次: hccl作为算子编排层，最合理的行为是快速把错误报告给用户(用户可以决定是否重试); hcomm作为通信框架层，有能力也有责任在某些场景下自动恢复(因为它掌握了链路状态和对端信息)。

真正的灰色地带在于: hcomm内部的某个具体错误点，到底应该快速失败(CHK_RET返回HCCL_E_INTERNAL)还是尝试恢复(触发OpRetry)? BF-3(hcomm:994390df)证明容错路径本身也会出bug(状态机初始状态未设置+环境门控缺失)。LC-4(hcomm:c5443da0)证明如果不考虑资源耗尽的最坏情况，设计本身就是一种"慢速失败"。选择容错继续意味着承担更高的代码复杂度和更多的bug面积。

### 正面案例: 快速失败是正确的

Case RV-10 (hcomm:f0666214): sdma回退。原始commit在GetPubDispatcher()中为Ascend950芯片做了"降级": 返回HCCL_SUCCESS但dispatcherPtr保持nullptr。nullptr作为"功能不可用"的信号改变了函数契约——调用方原本假设: 返回HCCL_SUCCESS意味着dispatcherPtr有效。6小时后被revert。正确的做法应该是快速失败: 返回明确的错误码(如HCCL_E_NOT_SUPPORT)。

Case LC-4 (hcomm:c5443da0): stream资源耗尽。原设计为每个model分配独立的order stream，在model数量增长时stream资源枯竭。这个"不会立即失败但会慢速耗尽资源"的设计比快速失败更危险——症状出现时已难以追溯根因。修复方案: 用全局唯一的order stream + Event对替代。

### 正面案例: 容错继续是正确的

Case PL-5 (hcomm-dev:28f18ab5): 简化重执行WaitResume。删除g_isRdmaError全局变量后，链路切换(changeLink)从"仅RDMA错误时触发"简化为"无条件触发"。"宁可多做恢复动作(changeLink)也不要漏做"——changeLink的代价是百毫秒级链路重建，漏做的代价是整个通信域不可恢复。

Case BF-1 (hcomm:75659d24): 重执行时序。2行语句重排修复了多线程下的状态不一致。修复原则: "先清理物理资源(ClearBuff+UpdateSq)，后清除异常标记(ResetCqeExceptionStatus)"。

### 反面案例: 容错路径本身的bug

Case BF-3 (hcomm:994390df): 快恢失败。Step快恢(NsRecovery)是容错恢复机制，但它自身出了两个bug: (1)retry状态机注册时没有显式设置初始状态; (2)CheckExitWaitResumeState在非AICPU环境也被调用。容错路径在正常执行时不被触发，因此其bug不会在常规测试中暴露——只有系统真的故障时才显现，叠加容错路径的bug会导致不可恢复的失败。

Case BF-2 (hcomm:1535b1c4): 读写锁→原子变量。与被revert的c546d353(全局mutex)对比: 粗粒度mutex在通信框架中几乎必然性能退化。如果选择容错继续(加锁保护共享状态)，锁的粒度选择本身就是一个可能失败的决策点。

### 判断标准

快速失败的场景:
1. 参数校验失败(CHK_PTR_NULL/CHK_RET): 调用者传入了无效参数，不存在"恢复"的可能。
2. 编译/链接时可检测的错误(BF-6): Host-only API在AICPU环境不可用。
3. 函数契约违反(RV-10): 返回成功但输出参数无效是对契约的破坏。
4. 硬件资源分配失败(LC-4): stream/event创建失败说明资源已耗尽。

尝试恢复的场景:
1. 网络/链路故障(CQE错误): 对端可能暂时不可达，重试或changeLink有可能恢复。
2. 通信超时: 可能是网络抖动而非永久故障。PL-4用3次重试+总超时三等分处理建链阶段的抖动。
3. 可恢复的状态不一致: 如重执行路径中的状态重置(BF-1)。

### CQE异步错误的处理策略

CQE错误不走CHK_RET链路，而是通过ExceptionHandler回调异步检测。CQE异步错误检测到问题时，不应该中断当前操作(可能多个线程同时在执行通信操作)，而应该标记异常状态(dfxExtendInfo_.pollStatus/cqeStatus)，让正在执行的操作在下一个同步点检查异常状态并走重执行路径。

但标记的时序至关重要(BF-1的教训): 清除异常标记必须在物理资源清理完成之后。

### 推荐默认选择

默认选择快速失败(CHK_RET返回)，仅在以下三个条件同时满足时选择容错继续:

1. 错误类型是可恢复的(网络故障/超时/链路异常)，而非不可恢复的(参数错误/资源耗尽/编程错误)。
2. 恢复路径已经过充分验证(重执行子系统经历了21条commit的密集修复后才逐步稳定)。
3. 容错路径的代价(changeLink/重执行耗时)远小于失败代价(整个通信域不可用)。

对于容错路径自身的质量保障:
- 状态机初始状态必须显式设置(BF-3)
- FSM状态重置顺序必须遵循"先物理清理后标记清除"(BF-1)
- 容错路径中的锁必须是细粒度的(BF-2: atomic CAS而非全局mutex)
- 容错路径必须有环境门控(BF-3: 非AICPU环境跳过快恢)

### 决策树

```
你遇到的错误类型?
  参数校验失败(空指针/越界/类型不匹配):
    → 快速失败: CHK_PTR_NULL / CHK_RET 返回HCCL_E_PARA
       不要尝试修复调用者的错误
  硬件资源不可用(stream/event/notify创建失败):
    → 快速失败: 返回HCCL_E_RESOURCE
       参考LC-4: 资源耗尽后继续执行只会产生更多失败
       设计层面应该用共享资源+同步原语(event)预防
  网络/链路错误(CQE异常/超时):
    是否在重执行支持的范围内?
      是且重执行已开启(retryEnable=true):
        → 容错继续: 标记异常状态, 走OpRetry路径
           标记时序: 先清理物理资源, 后清除异常标记(BF-1)
           恢复策略: 无条件changeLink比条件changeLink更安全(PL-5)
      否(重执行关闭或不支持的操作):
        → 快速失败: 返回HCCL_E_TIMEOUT或HCCL_E_INTERNAL
  建链阶段的网络抖动:
    → 有限重试: 总超时切分, 非末次降级日志
       参考PL-4: 最多3次重试, 总超时三等分
  容错路径中又出错了(容错的容错):
    → 快速失败: 返回错误码, 不要嵌套重试
       参考BF-3: 容错路径本身的bug比原始错误更难定位
  不确定:
    → 默认快速失败(CHK_RET返回)
       理由: 快速失败的代价是用户看到错误(可重试)
             容错继续的代价是系统可能进入不一致状态(不可恢复)
       新写的容错路径几乎必然有bug(BF-3教训)
```

---

## 12.10 综合灰色地带判断框架

以上9个灰色地带覆盖了hccl/hcomm开发中最常见的"怎么做都有道理"的决策场景。
本节将它们整理为统一的判断框架，方便开发者在面对新场景时快速定位适用的灰色地带。

### 判断入口: 你面临什么类型的决策?

```
架构路径选择:
  "我的代码应该走V1还是V2?" → 12.1
  "我应该在hccl还是hcomm中实现?" → 12.2
  "我应该继承已有基类还是独立实现?" → 12.3
  "我应该放在algorithm/framework/platform哪一层?" → 12.4

代码质量权衡:
  "这里需要加CHK_RET/CHK_PTR_NULL吗?" → 12.5
  "这个legacy模块我应该修还是等next替代?" → 12.6

设备侧决策(hccl/hcomm独有):
  "新算法用CCU还是AICPU?" → 12.7

协议和容错决策(hccl/hcomm独有):
  "协议改动应该一步到位还是渐进式?" → 12.8
  "这个错误应该快速失败还是尝试恢复?" → 12.9
```

### 九大灰色地带的共性规律

从50个landmark case和9个灰色地带的分析中，提炼出五条贯穿所有灰色地带的共性规律:

规律一: 渐进优于激进。
FT-6(37文件跨V1/V2同日revert)、RV-1(51文件次日revert)、RV-7(心跳帧膨胀2天revert)反复证明: 大规模一步到位的变更在hccl/hcomm中几乎注定失败。成功的案例(PL-5净减25行、PL-4总超时切分、RF-1 thin wrapper)都是小步快走。量化标准: 单次commit超过30个文件时需要特别审慎，超过50个文件时需要拆分。

规律二: 开关保护是生命线。
PL-1/PL-2(心跳introduce→revert→add-switch-redo)和Phase 8.9(重执行默认关闭c303e62a)反复验证: 新协议功能不带开关=出问题时只能整体revert。开关的实现成本(几十行env_config解析)远低于revert的成本(release分支联动+团队中断)。推广到所有灰色地带: 任何不确定结果的变更都应该提供回退手段(环境变量/配置项/feature flag)。

规律三: 跨边界变更需要原子性。
hccl-hcomm跨仓(PL-6同作者21秒、XR-2同作者18秒、FT-2间隔4分钟)、V1-V2跨路径(CL-2双路径实现+双路径UT)、Host-Device跨域(BF-9 CCU控制流语义)——所有跨边界变更都要求原子性提交或明确的分步策略(RF-1 thin wrapper先hcomm后hccl间隔11分钟)。

规律四: 验证周期与变更层次成正比。
构建系统变更(RV-3 cmake): 1小时内CI捕获。
Host侧逻辑变更(FT-6 mix-running): 6-10小时功能测试捕获。
Device侧参数变更(RV-9 loopnum): 17天硬件全场景验证。
协议功能变更(RV-7 HB check): 34天集群验证。
含义: 不要用Host侧的验证标准去要求Device侧/协议侧的变更。

规律五: 设备侧是bug富矿，简化是最有效的修复策略。
Phase 8.8: 设备侧BUGFIX密度43.5% vs Host侧30.3%。13个BUGFIX案例中11个涉及Host-Device边界。PL-5(删除g_isRdmaError净减25行)、BF-1(2行语句重排)、BF-5(1个数字修复)——最有效的设备侧修复往往是简化而非增加代码。通信框架中，少即是多。

### 快速参考表

| 灰色地带 | 推荐默认选择 | 反模式 | 关键case |
|----------|-------------|--------|---------|
| 12.1 V1 vs V2 | A5走V2, 多芯片先legacy | 跨V1/V2大改(FT-6) | FT-2, FT-6, CL-2 |
| 12.2 hccl vs hcomm | 不涉及链路细节→hccl | 跨仓API变更不预留迭代(XR-4) | FT-1, RF-1, OPT-1 |
| 12.3 深继承 vs 扁平 | 继承InsCollAlgBase(V2) | 执行模型不同仍强行继承 | FT-3, FT-4, AA-2 |
| 12.4 三层放置 | 默认framework层 | 跨三层大改(FT-6) | FT-7, AA-5, BF-8 |
| 12.5 防御 vs 信任 | 100%覆盖+UNLIKELY标注 | 移除运行时检查 | BF-3, OPT-3, BF-6 |
| 12.6 legacy vs next | 共存期legacy优先 | 跨边界大重构(FT-6) | CL-1, FT-2, RF-1 |
| 12.7 CCU vs AICPU | 先AICPU再CCU | CCU参数当日合入当日revert(RV-9) | OPT-4, BF-5, XR-2 |
| 12.8 渐进 vs 一步到位 | 渐进+开关 | 不可关闭的协议功能(RV-7) | PL-5, PL-1/2, RV-6 |
| 12.9 快速失败 vs 容错 | 默认快速失败 | nullptr作功能不可用信号(RV-10) | PL-5, BF-3, LC-4 |
