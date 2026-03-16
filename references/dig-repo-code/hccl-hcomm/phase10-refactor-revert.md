# Phase 10.4: REFACTOR + REVERT + INFRA 类 Landmark Case 深度分析

Date: 2026-03-16
Phase: 10.4 (Decision Point Deep Analysis — REFACTOR/REVERT/INFRA batch)

---

## REFACTOR Cases

### Case RF-1: 调整算子入口到算子仓 — hccl/hcomm 边界的架构级重新划分

[Commit]: hccl-dev:c696e52f + hcomm-dev:ba56e500 ("调整算子入口到算子仓")
[类别]: REFACTOR
[涉及层]: hccl:ops/interface + hcomm:framework/op_base
[变更规模]: hccl-dev 9 files, +368/-5; hcomm-dev 109 files, +2484/-2601
[维度标注]: A(跨仓边界)

[场景]:
hccl和hcomm的仓库边界需要重新划分。此前，所有集合通信算子的公开API入口(HcclAllReduce、HcclBroadcast、HcclReduceScatter等)
直接定义在hcomm仓库的`src/framework/op_base/src/op_base.cc`中。这意味着算子的"外壳"(参数校验、日志、MR描述中的entry
point)和"内核"(调度、执行)分处同一个仓库中。但从架构角度看，hccl是算子仓(编排)、hcomm是通信框架仓(实现)，
算子的外部API入口应该在hccl中定义。

[约束]:
1. 函数签名不能变(对外API的ABI兼容性)
2. 109个文件需要同步修改(hcomm-dev侧)，主要是函数重命名和日志字符串更新
3. 两仓必须原子性提交(hccl-dev先拿到hcomm-dev export的Inner函数才能编译通过)
4. 涉及大量UT/ST测试文件需要同步更新函数调用名

[决策]:
采用"thin wrapper + Inner suffix"模式:
- hcomm-dev侧: 将`HcclBroadcast`等函数全部重命名为`HcclBroadcastInner`(109文件批量改名)
- hccl-dev侧: 新增`src/ops/interface/hccl_collective_op.cc`(91行)，每个API函数只是一行转发:
  `return HcclBroadcastInner(buf, count, dataType, root, comm, stream);`
- hccl-dev侧同时新增`inc/hccl.h`(253行)，将原来定义在hcomm中的公开头文件声明移到hccl中

[替代方案]:
1. 直接将op_base.cc整个文件移到hccl仓——风险太大，op_base.cc有2000+行代码且深度依赖hcomm内部头文件
2. 只移头文件不移实现——不够彻底，仍然是hcomm export函数、hccl只是重新声明
3. 渐进式迁移(每次移一个算子)——可行但效率低，且中间状态API分散在两仓中造成混淆

选择方案的理由: thin wrapper是最小侵入性方案。Inner函数保持原有的完整实现逻辑不动(校验、日志、
调度全在hcomm中)，hccl侧只是提供ABI入口。这样hcomm的109个文件改动虽然多但都是机械性重命名
(sed替换函数名+日志字符串)，风险可控。

[后果]:
提交时间差: hcomm-dev在17:23提交(先)，hccl-dev在17:34提交(后)，间隔11分钟。说明需要协调
两仓的合入顺序(hcomm-dev先export Inner函数，hccl-dev才能引用它们)。后续没有revert记录，
说明方案执行顺利。

[可迁移经验]:
跨仓边界调整应采用"thin wrapper + Inner suffix"模式: 在旧位置将函数重命名为XXXInner，
在新位置添加只有一行转发的wrapper。这样可以在不改变任何业务逻辑的情况下完成架构调整，
且两仓的变更可以分别review。

---

### Case RF-2: hccl_fwk -> hcomm 模块重命名 — 仓库身份认同变化

[Commit]: hcomm-dev:4fc686be ("hccl_fwk -> hcomm & tar包名修改")
[类别]: REFACTOR
[涉及层]: hcomm:构建系统(CMakeLists.txt) + 打包脚本 + HLT测试
[变更规模]: 41 files, +102/-97

[场景]:
hcomm仓库原来叫"hccl_fwk"(HCCL Framework)，需要改名为"hcomm"以反映其独立的通信框架身份。
这不仅是代码层面的重命名，还涉及编译产物名(库名)、tar包名、xml配置、测试文件名等。

[约束]:
1. 所有CMakeLists.txt中的target名(hccl_fwk -> hcomm)需要统一替换
2. 打包脚本(CommLib.xml)中的组件名需要同步
3. HLT测试入口文件需要从hccl_fwk_test.cpp重命名为hcomm_test.cpp
4. 必须确保下游依赖(hccl仓)能正确找到新库名

[决策]:
一次性批量替换: 40个CMakeLists.txt文件中的target_link_libraries、add_library、include目录
全部从hccl_fwk替换为hcomm。测试入口文件直接rename。

[替代方案]:
1. 保留旧名(不改名)——技术上可行但造成名实不符的长期困扰
2. 保留别名(alias target)做渐进迁移——增加维护负担
3. 只改外部可见名、保留内部编译target名——中间状态更混乱

[后果]:
没有后续revert，执行顺利。此commit发生在仓库初始化阶段(2025-11-29)，属于仓库建立后的
第一波基础设施整理。

[可迁移经验]:
模块重命名应在仓库初始化阶段一次性完成，不要留到功能开发期。涉及编译target名变更时，
需要检查所有CMakeLists.txt中的target引用(grep hccl_fwk -> 批量替换)和打包脚本。

---

### Case RF-3: Split huge source files — HCCP RDMA服务层的代码拆分

[Commit]: hcomm-dev:e10d1002 ("Split huge source files")
[类别]: REFACTOR
[涉及层]: hcomm:platform/hccp/rdma_service
[变更规模]: 8 files, +659/-631

[场景]:
rs_socket.c是HCCP RDMA服务的核心文件，代码量过大(600+行)。需要按职责拆分:
socket管理逻辑保留在rs_socket.c中，driver-level socket操作(驱动层socket创建、bind、listen等)
抽出到新文件rs_drv_socket.c中。

[约束]:
1. 这是C语言文件(不是C++)，拆分时需要正确处理头文件依赖(新增rs_drv_socket.h)
2. rs_socket.c被rs_epoll.c、rs_ping_roce.c、rs.c等多个文件引用
3. 拆分后的编译依赖关系不能破坏

[决策]:
将rs_socket.c中的驱动层函数(约600行)移到rs_drv_socket.c，新增rs_drv_socket.h声明接口。
rs.c和rs_ping_roce.c新增`#include "rs_drv_socket.h"`。rs_epoll.c的include路径调整。
删除原rs_socket.h(25行)和rs_socket.c(597行)。

总行数基本不变(+659/-631)，说明这是纯移动+微调，没有功能修改。

[替代方案]:
1. 不拆分，保持大文件——长期维护成本高，git diff可读性差
2. 更细粒度拆分(3-4个文件)——对C文件来说过度拆分反而增加头文件管理负担

[后果]:
无后续revert。这个commit展示了C代码拆分的标准做法:
提取职责明确的子模块 → 新增.c/.h对 → 调整引用 → 删除原文件。

[可迁移经验]:
C文件拆分遵循"extract+redirect"模式: 新增.c/.h文件承接被抽出的函数，原文件的调用方
新增include，确保编译依赖无破坏。拆分时不要夹带功能修改(行数应基本持平)。

---

## INFRA Cases

### Case XR-4: API Changes 多轮迭代 — 跨仓接口变更的不稳定性

[Commit]: hccl-dev:de127490 + hcomm-dev:616e8044 ("API Changes", 2025-12-01)，
  前序: hccl-dev:bdeb5ee + hcomm-dev:8d4437c2 (2025-11-29 23:37，首次引入)
  revert: hccl-dev:29c59f3 + hcomm-dev:97754320 (2025-11-29 23:58，22分钟后回退)
  retry: hccl-dev:0d4cdb7 + hcomm-dev:94d74bfe (2025-11-29 23:59，1分钟后重试)
  final: hccl-dev:de127490 + hcomm-dev:616e8044 (2025-12-01，2天后最终版)
[类别]: INFRA
[涉及层]: hccl:ops/aicpu + hccl:ops/scatter + hcomm:framework/resource + hcomm:framework/communicator
[变更规模]: hccl-dev 6 files, +33/-20; hcomm-dev 20 files, +99/-161
[维度标注]: A(跨仓边界)

[场景]:
hccl/hcomm仓库从内部代码同步到开源仓库(gitcode)时，需要适配API差异。这些差异包括:
kernel_launch.cc中的API调用方式、scatter_op.cc的接口签名、thread_manager的类型签名等。

[约束]:
1. 两仓必须同步变更(API签名在hcomm中定义，调用在hccl中)
2. 开源仓库的API可能与内部版本存在差异
3. 仓库初始化阶段，CI验证流水线可能不完善

[决策]:
经历了三轮迭代:
- Round 1 (11-29 23:37): 首次尝试，两仓同步提交
- Revert (11-29 23:58): 22分钟后发现兼容性问题，两仓同步revert(merge-revert-mr)
- Round 2 (11-29 23:59): 1分钟后小幅调整重试(与Round 1不同的content commit)
- Round 3 (12-01): 2天后最终稳定版，hccl-dev 6文件、hcomm-dev 20文件

[替代方案]:
1. 一次性做对——理想但在仓库初始化阶段几乎不可能(环境不确定性太高)
2. 先只改hcomm再改hccl——打破原子性，中间状态编译失败

[后果]:
Round 3后稳定，无后续revert。但这个三轮迭代揭示了一个模式:
跨仓API变更平均需要2-3轮迭代才能稳定。

[可迁移经验]:
跨仓API变更不要期望一次做对。预留revert余量: 提交前准备好revert分支。
首次变更尽量小(只改必须的签名)，不要搭便车夹带其他修改。

---

### Case CL-2: HcclEngineCtxDestroy API — 跨legacy/next双路径的生命周期管理

[Commit]: hcomm:c5709dd4 ("HcclEngineCtxDestroy API")
[类别]: INFRA
[涉及层]: hcomm:framework/next + hcomm:framework/communicator/independent_op + hcomm:legacy/interface
[变更规模]: 29 files, +808/-164
[维度标注]: C(V1-V2-Legacy)

[场景]:
HcommBatchModeStart和HcommBatchModeEnd这对API原本负责Engine上下文的创建和销毁。
但在不调用这对API的情况下(直接下发任务)，缺少独立的Destroy机制。需要新增HcclEngineCtxDestroy
对外接口。

[约束]:
1. 必须同时在legacy路径和next路径实现(因为两条路径共存)
2. legacy侧: StreamLite构造函数需要新增参数(bool标记)来区分是否需要独立销毁
3. next侧: 需要新增EngineCtxs类(engine_ctxs.cc/h)来管理Engine上下文的完整生命周期
4. 需要新增UT测试覆盖Destroy功能(两个测试文件: ut_HcclEngineCtxDestroy_API_test.cc +
   ut_HcclEngineCtxDestroy_API_V2_test.cc)

[决策]:
- next路径: 新增`src/framework/next/coll_comms/rank/engine_ctxs/`目录，包含engine_ctxs.cc/h(100行)
  实现CreateCommEngineCtx/DestroyCommEngineCtx/GetCommEngineCtx的完整生命周期
- legacy路径: 修改StreamLite构造函数签名(新增bool参数)
- 接口层: 在hccl_api.h中新增HcclEngineCtxDestroy声明
- 测试: 两个独立的UT文件(分别测试V1和V2路径的Destroy)

[替代方案]:
1. 只在next路径实现——不可行，legacy路径仍是主力(Phase 8.7: legacy占修改40%)
2. 改造BatchModeEnd来兼容独立Destroy——侵入性太强，BatchModeEnd承载了其他逻辑
3. 不新增EngineCtxs类，直接在my_rank.cc中实现——违反单一职责原则

[后果]:
无后续revert。但这个case清楚地展示了在legacy/next双路径共存期间新增API的成本:
必须双路径实现 + 双路径测试(29文件中约一半是测试)。

[可迁移经验]:
在legacy/next双路径共存期间新增API，必须在两条路径都提供实现和独立UT。
如果只能择一，先实现legacy路径(它是当前主力)。新API设计时应考虑未来legacy退场后的
简化路径。

---

## REVERT Cases

### Case RV-1: Revert "ars code" — 51文件新算法的合入时机问题 (Chain 6)

[Commit]: hcomm-dev:1844b823 (Revert "ars code", 2026-01-08)
[类别]: REVERT
[涉及层]: hcomm:algorithm + hcomm:framework + hcomm:platform
[变更规模]: 51 files, +179/-2283 (revert方向)

[Revert分析]:
  - 原始变更: hcomm-dev:99d2a2b3 ("ars code", 2026-01-07 14:11), 51 files, +2283/-179
    为910_93平台引入ARS(Allreduce-Reduce-Scatter)算法。核心改动:
    * 新增3个executor类: coll_all_gather_ars, coll_all_reduce_ars, coll_reduce_scatter_ars(各for_910_93)
    * 新增1048行ST测试(testcase_ars_alg.cc)
    * 修改topo_matcher.cc: 新增GetARSFlag()/EditCommPlaneVector()/GetCommPlaneRanks()
    * 修改device_capacity.cc: 新增910_93带宽常量(HBM 650GB/s, SIO 240GB/s)并重构GetBandWidthPerNPU()
    * 修改coll_alg_operator.cc: 注册ARS算法路径(+62行)
    * 修改aicpu_communicator.cc: 适配ARS在device侧的处理
  - revert原因: 合入时机问题。次日(不到24小时)被revert。
    ARS依赖device_capacity.cc中新增的带宽常量和topo_matcher中的ARS拓扑判断，
    这些基础设施可能与其他并行开发的变更冲突。MR描述无详细说明("ars code")，
    暗示是计划性回退(等其他依赖就位)而非紧急修复。
  - 修复方案: hcomm-dev:62e4b1c2 ("ars code", 2026-01-14), 6天后重新引入。
    stat几乎完全一致(51 files, +2285/-179，仅差2行)。差异极小说明不是代码修改的问题，
    而是合入时序的问题——等到依赖项(如device_capacity变更或topo_matcher其他修改)合入后再提交。
  - 预防性规则: 大规模新算法引入(50+文件)前，需确认: (1)所有依赖的基础设施变更已先合入;
    (2)没有与当前变更冲突的并行MR在合入队列中。合入顺序: 基础设施→核心算法→测试。

[场景]: 为910_93平台引入全新的ARS集合通信算法
[约束]: 51文件跨algorithm/framework/platform三层，依赖device_capacity和topo_matcher的变更
[决策]: 先全量提交，不做分步
[替代方案]: 分3步提交(1.基础设施变更 2.executor类 3.注册+测试)——但ARS的3个executor互相依赖，
  难以进一步拆分。更可行的替代方案是: 先单独提交device_capacity.cc和topo_matcher.cc的变更，
  确认稳定后再提交ARS算法本体。
[后果]: 次日revert，6天后原样重新引入(仅差2行)
[可迁移经验]: 50+文件的新算法如果依赖基础设施变更(如device_capacity/topo_matcher)，
应将基础设施变更独立为先导commit，等CI验证通过后再合入算法本体。"代码正确但时机不对"
是大规模变更被revert的最常见原因。

---

### Case RV-2: Revert "coll comm mix-running" — "临时方案"不应合入主干 (Chain 3)

[Commit]: hcomm:0c033874 (Revert, 2026-03-10 19:48)
[类别]: REVERT
[涉及层]: hcomm:framework/op_base + hcomm:framework/next + hcomm:framework/hcom
[变更规模]: 37 files, +322/-1145 (revert方向)

[Revert分析]:
  - 原始变更: hcomm:ebb54874 ("support coll comm & orion comm mix-running", 2026-03-10 10:10)
    37 files, +1140/-317。支持A5平台新老算子(coll comm和orion comm)混合运行。
    核心改动:
    * op_base.cc: 将全局数组`g_opHcomInfos[MAX_MODULE_DEVICE_NUM + 1]`改为static局部变量 +
      `GetOpHcomInfo(uint32_t devId)`访问函数。在获取OpHcomInfo时触发`HcommResMgrInit(devId)`
      来延迟初始化资源管理器(标注为"临时方案"!)
    * 删除了hcomm_res_mgr.cc/.h两个文件
    * next/coll_comms/rank/my_rank.cc: CCU资源初始化添加环境变量门控
      (`HCCL_INDEPENDENT_OP`非空时才初始化)
    * hcom.cc: 新增mix-running相关的hcom接口适配
  - revert原因: 设计缺陷。代码中自带注释"临时方案"，合入仅10小时即被revert。
    三个致命问题:
    1. 全局数组改static局部变量 + lazy init: 多线程下首次调用GetOpHcomInfo可能竞态初始化
    2. 删除hcomm_res_mgr.cc/.h: 架构变更幅度过大，资源管理器的职责被打散
    3. HcommResMgrInit在GetOpHcomInfo中被隐式调用: 违反了显式初始化原则
  - 修复方案: 截至分析时点未重新引入。后续有fix nullptr core dump等修复，但并非重新引入
    mix-running功能。说明方案需要根本性重新设计。
  - 预防性规则:
    (1) 标注"临时方案"的代码不应合入主干——如果自己都认为是临时方案，说明设计没想清楚
    (2) 删除整个文件(hcomm_res_mgr.cc/.h)的重构必须有充分的多线程场景验证
    (3) 资源管理器的初始化时机变更(显式→隐式)需要证明线程安全

[场景]: 支持A5平台新老算子混合运行
[约束]: 需要同时修改legacy和next路径; 资源管理器初始化时机需要变更
[决策]: 采用lazy init(在GetOpHcomInfo中触发资源管理器初始化)
[替代方案]: (1)在HcclCommInit流程中显式初始化mix-running所需资源; (2)使用std::call_once
确保线程安全的延迟初始化; (3)先做最小改动(不删除res_mgr文件)，仅添加混跑的dispatch逻辑
[后果]: 10小时后revert，未重新引入
[可迁移经验]: 涉及资源管理器生命周期变更(尤其是将显式初始化改为隐式延迟初始化)时，
绝不能自标"临时方案"后合入。必须先证明线程安全性，然后通过code review确认设计意图。

---

### Case RV-3: Revert offline compile [2/2] — 构建系统文件重命名的破坏性 (Chain 4)

[Commit]: hcomm:753ba8c2 (Revert, 2026-02-26 23:21)
[类别]: REVERT
[涉及层]: hcomm:构建系统(cmake/)
[变更规模]: 24 files, +501/-642 (revert方向)

[Revert分析]:
  - 原始变更: hcomm:05b38411 ("[Build] Add support for offline compile [2/2]", 2026-02-26 22:22)
    24 files, +640/-499。核心改动:
    * cmake/utils.cmake重命名为cmake/hcomm_utils.cmake(破坏性重命名)
    * 重写cmake/third_party/openssl.cmake的编译逻辑
    * 更新多个third_party cmake文件的路径引用和编译选项
    * 修改docs/build.md构建文档
    注意: 这是[2/2]分步提交的第二步; [1/2]是hcomm:b3c0ad3e(53 files, +602/-458)，
    [1/2]没有被revert。
  - revert原因: 编译失败。合入仅1小时即被revert，几乎可以确定是CI构建失败触发的。
    utils.cmake -> hcomm_utils.cmake的重命名是一个破坏性变更:
    如果有任何地方(包括[1/2]引入的代码)引用了utils.cmake这个路径，就会立即编译失败。
  - 修复方案: hcomm:6a7c0c81 (次日17:35重新引入，24 files, +658/-499，比原始多18行)。
    通过新的MR(!613)引入，修复了导致构建失败的路径引用问题。
  - 预防性规则:
    (1) cmake文件重命名时，需grep搜索所有引用旧文件名的地方(包括其他MR中的代码)
    (2) 分步提交的构建系统变更，[2/2]必须能在[1/2]的基础上独立编译通过
    (3) 构建系统变更提交前必须在所有目标平台上完整编译验证

[场景]: 为hcomm支持离线编译(offline compile)
[约束]: 分两步提交([1/2]改53个文件，[2/2]改24个文件); cmake文件路径变更影响全局
[决策]: 在[2/2]中做utils.cmake→hcomm_utils.cmake的重命名
[替代方案]: (1)在[1/2]中就完成重命名(更安全，因为[1/2]的文件列表更完整);
(2)不做重命名，保留utils.cmake名称; (3)用cmake alias机制做渐进迁移
[后果]: 1小时后revert，次日修复重新引入(+18行修补)
[可迁移经验]: 构建系统文件重命名是高风险操作。分步提交时，重命名应放在第一步(让后续步骤
可以基于新名称开发)。提交前必须`grep -r "旧文件名" .`确保无遗漏引用。

---

### Case RV-4: revert//add share/info — 安装路径变更的增量演进 (独立case)

[Commit]: hccl-dev:52bda686 ("revert//add share/info", 2025-11-27 00:11)
[类别]: REVERT
[涉及层]: hccl:scripts/package(安装/卸载/升级脚本)
[变更规模]: 18 files, +93/-107 (revert方向)

[Revert分析]:
  - 原始变更: hccl-dev:d6cea823 ("add share/info", 2025-11-26 16:36)
    18 files, +107/-93。将HCCL安装路径从`${STAGING_DIR}/hccl/`改为
    `${STAGING_DIR}/share/info/hccl/`。修改了install.sh、uninstall.sh、
    upgrade.sh、setenv.bash等安装脚本以及version管理脚本。
  - revert原因: 路径不兼容。约8小时后被revert，可能导致与其他CANN组件的安装路径约定
    不一致，或与已部署环境不兼容。
  - 修复方案: hccl-dev:e348c99a ("add share/info", 2025-11-27 14:04)
    仅1个文件(hccl.xml)+7行。与原始的18文件全面改动相比，这是极度保守的增量方案——
    只在包配置文件中新增share/info路径声明，不修改安装/卸载脚本。
    后续还有hccl-dev:6f03650/00f1a0c进一步迭代安装路径。
  - 预防性规则: 安装路径变更不能一步到位。应采用增量方式:
    (1)先在配置文件中声明新路径; (2)在脚本中添加对新旧路径的双向兼容;
    (3)确认所有组件适配后再移除旧路径支持。

[场景]: 调整HCCL安装路径布局以符合标准目录结构(share/info)
[约束]: 安装路径影响整个CANN生态的部署链条(安装/卸载/升级/版本管理/其他组件的路径引用)
[决策]: 一次性修改全部18个脚本文件
[替代方案]: (1)只改配置文件(最终采用的方案); (2)新旧路径双写(两个安装位置同时存在)
[后果]: 8小时后revert; 14小时后用1文件的极简方案重新引入; 后续数个commit继续迭代
[可迁移经验]: 安装路径变更的正确策略是"声明先行，脚本后改"——先在包配置(xml)中声明
新路径，验证兼容性后再修改安装脚本。18文件的全面改动不如1文件的精确声明。

---

### Case RV-5: [Master] Revert Runtime interfaces — 外部依赖未就绪的接口迁移 (RTS Chain 1/2)

[Commit]: hcomm-dev:30e25e50 ("[Master] Revert Runtime interfaces", 2025-12-16)
[类别]: REVERT
[涉及层]: hcomm:platform/adapter + hcomm:framework/op_base + hcomm:common/stream
[变更规模]: 9 files, +61/-44

[Revert分析]:
  - 原始变更: 多个commit逐步引入的Runtime接口适配。核心变更:
    * adapter_rts.cc: `rtRegTaskFailCallbackByModule`(旧RTS接口) →
      `aclrtSetExceptionInfoCallback`(新ACL接口)
    * hcomm_primitives.h: uint32_t → u32类型回退
    * stream_utils.cc: 运行时接口签名调整
  - revert原因: 外部依赖未就绪。新ACL接口(aclrtSetExceptionInfoCallback)在目标
    Runtime版本中可能尚未就绪。[Master]前缀表明这是主干分支的有计划revert，
    同时存在[C25]分支的平行revert(hcomm-dev:dca1e7f3)。
  - 修复方案: 分阶段重新引入:
    * hcomm-dev:9968572e ([C25] Use external runtime interfaces, 2025-12-21)
    * hcomm-dev:b17691b7 (primitives error code revision, 2025-12-22)
    说明Runtime接口迁移改为渐进式(先在feature分支验证，再合入master)。
  - 预防性规则: 对外部依赖(Runtime/ACL)的接口迁移，必须:
    (1)确认目标版本中新接口已可用; (2)多分支([Master]+[C25])需要协同操作;
    (3)改为分阶段迁移而非一次到位。

[场景]: 将hcomm的Runtime接口从旧版RTS API迁移到新版ACL API
[约束]: ACL接口在目标Runtime版本中可能尚未就绪; 需要在Master和C25分支同步操作
[决策]: 先一次性迁移所有接口
[替代方案]: (1)分批迁移(每次只迁移一个接口); (2)添加兼容层(adapter)做运行时检测
[后果]: revert后分阶段重新引入
[可迁移经验]: 外部依赖(Runtime/ACL等系统级接口)的迁移不能假设新接口已就绪。
提交前必须与依赖方确认版本对齐，多分支需同步操作。

---

### Case RV-6: [Master] Revert rts interfaces — 兼容层设计的典范 (RTS Chain 2/2)

[Commit]: hcomm-dev:1d8e2c14 ("[Master] Revert rts interfaces", 2025-12-23)
[类别]: REVERT
[涉及层]: hcomm:platform/adapter + hcomm:platform/p2p_mgmt + test stub
[变更规模]: 7 files, +421/-100

[Revert分析]:
  - 原始变更: 多个commit的累积效果(包括d38afa51 "Use runtime interface to calculate
    notify offset")。核心: 将设备信息查询和设备间拓扑查询从RTS接口全部切换到ACL接口。
  - revert原因: 接口兼容性。但这不是简单的"回退"——从行数看(+421/-100)，
    revert后的代码比原来更复杂，说明这实际上是一次"回退+增强"。
    核心改进:
    * 新增`hrtGetPairDeviceLinkTypeRaw()`函数: 做动态dispatch。
      先通过dlsym检测rtGetPairPhyDevicesInfo(新接口)是否可用，
      可用则调用新接口(hrtGetPairPhyDevicesInfo)，
      不可用则fallback到旧接口(hrtGetPairDevicesInfo + aclrtGetDevicesTopo)。
    * hrtGetPairDevicesInfo中增加了拓扑类型转换逻辑(ACL_RT_DEVS_TOPOLOGY_HCCS → TOPOLOGY_HCCS等)
    * 扩展了test stub中的接口mock
  - 修复方案: 后续hcomm-dev:9968572e重新引入但采用了更完善的兼容层(adapter)。
  - 预防性规则:
    系统接口迁移(RTS→ACL)必须通过兼容层(adapter)同时支持新旧接口。
    正确的做法是: `hrtGetPairDeviceLinkTypeRaw()`先尝试新接口 → 不可用则fallback旧接口。
    不要假设所有环境都有新接口。

[场景]: 接续RV-5，继续回退设备信息查询和拓扑查询的RTS接口迁移
[约束]: 部署环境的Runtime版本不统一，新旧接口必须共存
[决策]: 不做简单回退，而是引入兼容层做动态fallback
[替代方案]: (1)简单回退到旧接口(更安全但失去新接口优势);
(2)通过编译时宏做条件编译(不够灵活，无法应对运行时版本差异)
[后果]: 兼容层方案被保留，后续成为标准做法
[可迁移经验]: 系统接口迁移的标准模式是"adapter + 动态fallback":
新增一个dispatch函数，运行时检测新接口是否可用(dlsym/函数指针)，
可用走新路径，不可用走旧路径。这比条件编译更灵活，比一次性切换更安全。

---

### Case RV-7: Revert HB check opinconsistent — 功能开关(feature switch)的必要性 (Chain 2)

[Commit]: hcomm-dev:a37e6cf1 ("Revert HB check opinconsistent", 2025-12-03)
[类别]: REVERT
[涉及层]: hcomm:framework/cluster_maintenance/health/heartbeat + hcomm:framework/communicator
[变更规模]: 7 files, +141/-249

[Revert分析]:
  - 原始变更: hcomm-dev:12f84a1a ("HcclCommConfig", 2025-12-01)中包含的heartbeat变更部分。
    核心变更(从revert diff反推):
    * heartbeat.h:
      - OPINFO_SEND_NUM(16) → OPINFO_SEND_NUM_BY_TAG(500) + OPINFO_TAG_QUEUE_NUM(10)
      - OpInfoDesc中identifier字段被移除，root/count字段顺序调整
      - 新增OpInfoTagQueue和OpInfoTagQueueFrame结构体(按tag分组的算子信息)
      - HeartBeatFrame中的opInfoList[16] → opInfoTagQueueFrame(复杂的二维结构)
      - 新增CheckOpInconsistentError()、RegisterSROpIdentifier()、AddInconsistentOpRecord()方法
      - opInfoQueue_元素从OpInfoDesc → pair<string, OpInfoDesc>(添加tag)
    * heartbeat.cc: SendFrame/GetOneOpInfo/GetSendOpInfoList/SaveOpInfo签名全部变更
    * hccl_communicator系列: 多处添加/删除AddOpInfo调用
  - revert原因: 设计缺陷(核心数据结构和接口签名的大面积变更)。
    三个关键问题:
    1. HeartBeatFrame大小显著增大: 从opInfoList[16]变为opInfoTagQueueFrame(10个tag x 500个op)，
       心跳帧大小敏感(Phase 8.9: 50ms间隔)，帧变大可能影响心跳性能
    2. 接口签名不兼容: GetOneOpInfo()从引用参数改为值返回，GetSendOpInfoList从vector&改为Frame结构体
    3. 锁降级: unique_lock → lock_guard(失去条件变量支持)
  - 修复方案: hcomm-dev:ebfcde5c ("HB check op insistent with switch", 2026-01-06)
    34天后重新引入，关键改进:
    1. 新增GetExternalInconsistentCheckSwitch()开关——默认关闭，可按需开启
    2. 心跳帧大小根据开关动态选择(HeartBeatFrameWithOpCheck vs HeartBeatFrame)
    3. 在env_config.cc中解析HCCL_DFS_CONFIG的inconsistent_check配置
    4. OPINFO_QUEUE_MAX_SIZE从65536提升到131072
  - 预防性规则:
    修改核心基础设施(心跳、资源管理器)时必须:
    (1) 添加功能开关(feature switch)——出问题时关闭开关比revert整个commit成本低得多
    (2) 不要在一个commit中同时修改数据结构和接口签名——分步: 先加新结构，再迁移接口
    (3) 心跳帧大小变更需要评估对心跳性能的影响

[场景]: 在心跳模块中引入算子一致性检查(op inconsistency check)功能
[约束]: 心跳是核心基础设施，修改影响集群健康检测; 帧大小和频率(50ms)敏感
[决策]: 直接修改HeartBeatFrame结构体和所有相关接口，不加开关
[替代方案]: (1)加功能开关(最终采用的方案); (2)新增独立的OpCheckFrame而非修改HeartBeatFrame;
(3)在心跳之外建立独立的一致性检查通道
[后果]: revert后34天，加了开关重新引入
[可迁移经验]: 核心基础设施(心跳/重执行/Notify)的功能增强必须带功能开关。
"不可关闭的新功能"是被整体revert的头号原因。开关的默认值应为"关闭"(opt-in)，
让使用方主动启用。

---

### Case RV-8: Revert "Revert 'fix loopnum = 128'" — CCU参数的验证周期 (Chain 1, 3/3)

[Commit]: hcomm:0c3be05f (Revert "Revert 'fix loopnum = 128'", 2026-02-28)
[类别]: REVERT(re-revert，即恢复原始修改)
[涉及层]: hcomm:legacy/unified_platform/ccu/ccu_microcode
[变更规模]: 2 files, +26/-29

[Revert分析]:
  - 这是Chain 1的第三个commit(re-revert)，恢复了CCU_MS_DEFAULT_LOOP_COUNT=128。
  - 时间线: fix(2/11 11:37) → revert(2/11 17:39, 6小时后) → re-revert(2/28 15:37, 17天后)
  - MR !622描述: "fix loopnum = 128"，标记为Bug修复
  - 与原始commit(fb56d64b)的diff完全一致(相同的2文件、相同的行变更)
  - 17天的间隔代表了CCU参数变更所需的完整硬件验证周期:
    在真实硬件上跑完性能和稳定性测试需要数周，而非数小时。

[场景]: 确认CCU_MS_DEFAULT_LOOP_COUNT=128是正确值后，恢复原始修改
[约束]: CCU microcode参数影响设备端所有CCU调度，需要真实硬件验证
[决策]: 直接re-revert(git revert of revert)
[替代方案]: 提交新commit重新设置128(而非revert of revert)——效果相同但git历史更清晰
[后果]: 128被最终保留，无后续修改
[可迁移经验]: 见RV-9分析。re-revert的存在证实了原始修改的正确性，
中间的revert是验证周期不足的产物。

---

### Case RV-9: Revert "fix loopnum = 128" — 1行改动引发3个commit的教科书案例 (Chain 1, 2/3)

[Commit]: hcomm:72cdf80e (Revert "fix loopnum = 128", 2026-02-11 17:39)
[类别]: REVERT
[涉及层]: hcomm:legacy/unified_platform/ccu/ccu_microcode
[变更规模]: 2 files, +29/-26

[Revert分析]:
  - 原始变更: hcomm:fb56d64b ("fix loopnum = 128", 2026-02-11 11:37)
    核心改动: 仅1行常量变更
    `constexpr uint64_t CCU_MS_DEFAULT_LOOP_COUNT = 64;` → `= 128;`
    附带UT测试更新: 53行变更，其中大部分是代码风格(空格)，关键行为变更是
    `EXPECT_EQ(ccuprofilinginfo.size(), 3)` → `EXPECT_EQ(ccuprofilinginfo.size(), 2)`
    (loopnum翻倍后profiling分块从3变为2)
  - revert原因: UT期望值变化+硬件验证不足。
    MR !488描述为空(无说明)，暗示紧急回退。
    合入仅6小时。CCU_MS_DEFAULT_LOOP_COUNT影响CCU微码的循环行为，
    从64改到128意味着CCU每次处理的数据量翻倍。
    UT中profiling数量从3变为2(分块合并)是正确的行为变化，但可能在某些边界场景下
    导致UT失败或硬件异常。
  - 修复方案: hcomm:0c3be05f (2026-02-28, 17天后re-revert恢复128)
  - 预防性规则:
    CCU设备端参数修改(尤其是loop count等影响微码执行行为的常量)需要:
    (1) UT通过 + 真实硬件验证(不能只靠UT)
    (2) 修改影响分析: 用Grep统计所有引用CCU_MS_DEFAULT_LOOP_COUNT的地方
    (3) 预留验证窗口: 不要在无法快速回退的时间点提交(如周末/假期前)

[场景]: 调整CCU微码默认循环计数以优化性能
[约束]: CCU是设备端代码，参数变更影响所有CCU调度场景; 缺乏完整的硬件验证环境
[决策]: 直接修改常量值，同步更新UT预期
[替代方案]: (1)用环境变量或配置项控制loop count，便于回退;
(2)仅在特定算法路径覆盖默认值(不改全局默认);
(3)先在dev分支验证2周再合入main
[后果]: 6小时后revert，17天后re-revert恢复
[可迁移经验]: CCU参数修改(1行常量!)的验证周期是17天而非6小时。
设备端参数修改即使代码量极小(1行)，其影响面也可能覆盖所有CCU调度路径。
"当日合入当日revert"是硬件验证不足的典型信号。

---

### Case RV-10: Revert "910_95 sdma" — 三条反模式的集中展示 (Chain 5)

[Commit]: hcomm:f0666214 (Revert, 2026-02-28 23:28)
[类别]: REVERT
[涉及层]: hcomm:platform/comm_primitive
[变更规模]: 1 file, -51 lines (revert方向)

[Revert分析]:
  - 原始变更: hcomm:0e5be69c ("910_95支持host侧本地sdma拷贝和本地reduce操作", 2026-02-28 17:42)
    1 file(hccl_primitive_local.cc), +51 lines。核心改动:
    * GetPubDispatcher()中: 用aclrtGetSocName()获取芯片名，字符串匹配"Ascend950"则跳过dispatcher(返回HCCL_SUCCESS但dispatcherPtr保持nullptr)
    * HcclLocalCopy()中: dispatcherPtr==nullptr时用hrtMemAsyncCopy降级
    * HcclLocalCopyReduce()中: dispatcherPtr==nullptr时用hrtReduceAsync降级
    * 新增hccl2rtDataTypeMap和hccl2rtReduceOpMap映射表
  - revert原因: 三条反模式集于一身:
    1. 硬编码芯片名: `targetChipVerStr.find("Ascend950") != std::string::npos`
       应该用device_capacity机制(DevType枚举)而非字符串匹配
    2. nullptr作为功能不可用信号: GetPubDispatcher()返回HCCL_SUCCESS但dispatcherPtr=nullptr，
       改变了函数契约(调用方原本假设成功后指针非NULL)
    3. 枚举语义混用: hccl2rtReduceOpMap中用ACL_RT_MEMCPY_SDMA_AUTOMATIC_SUM作为reduce操作，
       但这些是memcpy的kind而非reduce的kind
    额外问题: MR !648测试字段写"不涉及"——51行新代码没有任何测试
  - 修复方案: 截至分析时点未重新引入。需要根本性重新设计。
  - 预防性规则:
    (1) 不要用字符串匹配芯片名做能力判断——应通过device_capacity.cc中的DevType枚举
    (2) 不要用nullptr作为"功能不可用"的信号——应用显式的枚举/bool/Optional返回
    (3) 不要混用不同语义的枚举值——memcpy kind和reduce kind是两个不同的概念
    (4) 新功能代码必须有测试("不涉及"测试的代码不应合入)

[场景]: 为Ascend950芯片添加host侧本地SDMA拷贝和reduce操作的降级路径
[约束]: Ascend950不支持通过dispatcher下发DMA任务，需要走host侧直接拷贝
[决策]: 在GetPubDispatcher中通过芯片名字符串检测，跳过dispatcher
[替代方案]: (1)在device_capacity.cc中新增Ascend950的能力标记，通过DevType枚举判断;
(2)在dispatcher层添加Ascend950的specialization而非绕过dispatcher;
(3)在platform层增加transport_sdma_host.cc专门处理host侧SDMA
[后果]: 6小时后revert，未重新引入
[可迁移经验]: 新芯片适配应通过device_capacity机制(能力枚举)而非硬编码芯片名。
当你需要"如果芯片X则走不同路径"时，正确做法是: 在device_capacity.cc中注册芯片能力 →
在使用点通过能力枚举判断 → 而非在业务代码中做芯片名字符串匹配。

---

## Revert Chain Pattern 总结

### Chain 1: loopnum fix-revert-revert (1行改动 → 3个commit → 17天)

时间线:
- fb56d64b (2/11 11:37): CCU_MS_DEFAULT_LOOP_COUNT 64→128 (1行常量+UT)
- 72cdf80e (2/11 17:39): Revert (6小时后)
- 0c3be05f (2/28 15:37): Re-revert (17天后)

模式: "快速提交 → 验证不足回退 → 充分验证后恢复"
核心教训: CCU设备端参数的验证周期不是小时级而是周级。1行代码的影响面可以覆盖所有CCU调度路径。
Chain特征: 代码量极小(1行)、影响面极大(全局常量)、验证周期极长(17天)。

### Chain 2: HB check introduce→revert→add-switch-redo (34天)

时间线:
- 12f84a1a (12/01): 随HcclCommConfig引入HB check功能(无开关)
- a37e6cf1 (12/03): Revert (2天后)
- ebfcde5c (01/06): 加功能开关重新引入(34天后)

模式: "功能直接上线 → 被迫revert → 加灰度控制重新上线"
核心教训: 核心基础设施的新功能必须带开关。开关的成本(几十行env_config解析)远低于revert的成本(全team中断)。
Chain特征: 功能本身正确但缺少灰度控制机制。

### Chain 3: coll comm mix-running (10小时，未恢复)

时间线:
- ebb54874 (3/10 10:10): 引入mix-running (37文件)
- 0c033874 (3/10 19:48): Revert (10小时后)

模式: "设计缺陷导致的永久回退"
核心教训: "临时方案"不应合入主干。代码中自标"临时方案"的变更在生产环境中几乎必然出问题。
Chain特征: 未恢复，说明需要根本性重新设计。

### Chain 4: offline compile (1小时)

时间线:
- b3c0ad3e (2/26): offline compile [1/2] (53文件，未被revert)
- 05b38411 (2/26 22:22): offline compile [2/2] (24文件)
- 753ba8c2 (2/26 23:21): Revert [2/2] (1小时后)
- 6a7c0c81 (2/27 17:35): 修复后重新引入

模式: "CI快速捕获 → 修复重提"
核心教训: cmake文件重命名需要grep全量搜索引用点。分步提交的后续步骤如果依赖前序步骤，必须独立验证。
Chain特征: <1小时回退 = CI构建失败自动捕获，修复成本低。

### Chain 5: 910_95 sdma (6小时，未恢复)

时间线:
- 0e5be69c (2/28 17:42): 引入host侧SDMA降级
- f0666214 (2/28 23:28): Revert (6小时后)

模式: "设计反模式导致的永久回退"
核心教训: 三条反模式(硬编码芯片名/nullptr信号/枚举混用)集于一身，且无测试。
Chain特征: 未恢复，代码质量问题(非时机问题)。

### Chain 6: ars code (次日回退，6天后恢复)

时间线:
- 99d2a2b3 (1/07): 引入ARS算法 (51文件)
- 1844b823 (1/08): Revert (次日)
- 62e4b1c2 (1/14): 原样重新引入 (6天后，仅差2行)

模式: "合入时机问题 → 等待依赖就位 → 原样恢复"
核心教训: 大规模变更的revert原因不一定是代码错误，可能是合入时序问题(与其他MR冲突/依赖未就位)。
Chain特征: revert和re-introduce的diff几乎完全一致(差2行)。

### RTS Chain: Runtime interfaces (分阶段回退+兼容层演进)

时间线:
- d38afa51 (12/11): 引入runtime接口适配
- f9d03829: [C25]分支先回退
- 30e25e50 (12/16): [Master]回退Runtime interfaces(RV-5)
- 1d8e2c14 (12/23): [Master]回退rts interfaces(RV-6)，引入兼容层
- 9968572e (12/21): [C25]重新引入外部runtime接口
- b17691b7 (12/22): primitives error code revision

模式: "一步到位迁移 → 发现环境不统一 → 引入兼容层 → 分阶段重新迁移"
核心教训: 系统接口迁移(RTS→ACL)必须通过adapter+动态fallback实现。
RV-6的hrtGetPairDeviceLinkTypeRaw()是正确范式: 运行时检测新接口可用性，不可用则fallback。
Chain特征: 多分支([Master]+[C25])协同操作，最终方案比原始方案更复杂但更健壮。

---

## 横切面洞察

### Revert时间窗口与根因的相关性

| 时间窗口 | 典型根因 | 案例 |
|----------|----------|------|
| <1小时 | CI/编译失败 | RV-3(offline compile) |
| 6-10小时 | 功能测试/设计缺陷 | RV-9(loopnum), RV-10(sdma), RV-2(mix-running) |
| 次日 | 集成问题/合入时机 | RV-1(ars) |
| 周级 | 外部依赖未就绪 | RV-5/RV-6(RTS chain) |
| 月级 | 充分验证后恢复 | RV-7→PL-2(HB check, 34天), RV-9→RV-8(loopnum, 17天) |

### 被revert变更的共性缺陷

1. 缺少功能开关(RV-7): 3/10的revert可通过添加开关避免
2. 缺少测试(RV-10): MR描述写"不涉及"测试的代码不应合入
3. 过大的变更范围(RV-1, RV-2): 50+文件的变更更容易因时机问题被revert
4. 隐式假设(RV-5/6/10): 假设外部接口已就绪、假设指针非NULL、假设芯片名不变

### REFACTOR的成功模式

1. thin wrapper(RF-1): 最小侵入性的跨仓边界调整
2. 一次性重命名(RF-2): 在仓库初始化阶段完成
3. extract+redirect(RF-3): C文件拆分的标准做法
共性: 不夹带功能修改，变更行数基本持平(+/- 接近)。
