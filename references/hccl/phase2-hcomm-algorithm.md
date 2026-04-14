# Phase 2.3: hcomm Algorithm Layer

## Phase 2.3.1: base/ — AIV Algorithm Templates & Communicator Base

### Module Overview

`src/algorithm/base/` 包含两大子模块:
1. **alg_aiv_template/** — AIV(AI Vector)核上运行的集合通信kernel模板代码(设备侧C++, AscendC编程模型)
2. **communicator/** — 主机侧通信域拓扑计算和传输需求计算(标准C++)

这两个子模块在编程模型上完全不同: AIV模板是设备侧kernel代码(使用`__aicore__`/`__global__`修饰, AscendC API如DataCopy/PipeBarrier等), communicator是标准主机侧C++。

---

## Part 1: AIV Algorithm Templates

### 1.1 File Inventory

`alg_aiv_template/` 包含76个.h文件 + 1个CMakeLists.txt + 1个子目录(aiv_interface/)。
总行数: 12098行。全部代码实现在header中(header-only, 因为是设备侧kernel模板代码)。

最大文件:
- `aiv_communication_base.h` (1052行) — 基类AivCommBase + 所有公共常量/宏/同步原语
- `aiv_crossnode_91093_base.h` (835行) — 91093芯片跨节点通信基类
- `aiv_all_reduce_deter_910b_bigdata.h` (413行) — 确定性AllReduce大数据实现

### 1.2 Architecture: Two Base Classes

AIV模板有两个独立的基类体系:

**AivCommBase** (`aiv_communication_base.h:247-598`)
- 适用于910B芯片和91093芯片的节点内(intra-node)通信
- 管理16卡间的GM_IN[16]/GM_OUT[16]数组(CCL buffer地址)
- 提供同步原语: Record/Wait/Record1vN/WaitNv1/CountRecord/CountWait
- 提供数据搬运: CpGM2GM/CpGM2GMWithFlagWrap/DataCopyGM2UB/DataCopyUB2GM
- 提供维测: HeadCounter/TailCounter/AIV_INFO/AIV_ERROR
- 使用AscendC TQueBind/TBuf管理UB(Unified Buffer)

**AivCrossNode91093Base** (`aiv_crossnode_91093_base.h:34-224`)
- 适用于91093芯片的跨节点(cross-node)通信(A3超节点, 最多768卡)
- 独立的flagAddr/dataAddr/commAddr地址管理
- 专门的多核分工: CalcNumTargetsAndTargetRanks/CalcNumTargetsAndTargetRanksDeter/CalcNumTargetsAndTargetRanksGroup
- 支持blockGroup概念(多个AIV核服务同一个对端rank)

### 1.3 Key Design Pattern: Macro-Based Kernel Dispatch

`aiv_communication.h` 定义了所有kernel入口点。核心模式是通过宏生成per-type的kernel函数:

```
// 定义宏: 一个per-type kernel生成器
#define AIV_ALL_REDUCE_KERNEL_BATCH_DEF(type) \
extern "C" __global__ __aicore__ void aiv_all_reduce_##type(KERNEL_ARGS_DEF) { \
    // 分派到具体实现...
}

// 实例化: 对每个数据类型展开
AIV_ATOMIC_DATA_TYPE_DEF(AIV_ALL_REDUCE_KERNEL_BATCH_DEF);
// 展开为 aiv_all_reduce_float, aiv_all_reduce_half, ... 共6种
```

数据类型分两类:
- **AIV_ATOMIC_DATA_TYPE_DEF**: float, half, int16_t, int32_t, int8_t, bfloat16_t (6种, 支持原子操作)
- **AIV_COPY_DATA_TYPE_DEF**: 上述6种 + uint16_t, uint8_t, uint32_t (9种, 仅支持拷贝)

AllReduce/ReduceScatter需要原子操作(求和/最大/最小), 所以用ATOMIC类型; AllGather/AllToAll只需拷贝, 用COPY类型。

### 1.4 Kernel Dispatch Decision Tree

每个kernel入口函数内部是一个多层if-else分派树, 根据运行时参数选择具体实现:

```
AllReduce分派逻辑(aiv_communication.h:82-122):
1. devType==910B && deterministic==1 → deter_smalldata / deter_middata / deter_bigdata (按数据量)
2. isOpBase (单算子模式):
   a. aivRdmaStep>=0 → rdma_middata / rdma_smalldata (按useAivRdmaSmall)
   b. len >= 16MB → bigdata
   c. len > 64KB → middata
   d. else → smalldata
3. !isOpBase (图模式):
   a. devType==910_93 → 91093
   b. aivRdmaStep>=0 → rdma_middata_graph / rdma_smalldata_graph
   c. len > UB_MAX → bigdata_graph
   d. else → smalldata_graph
```

决策因子: devType(芯片) × isOpBase(执行模式) × deterministic(确定性) × len(数据量) × aivRdmaStep(RDMA阶段)

### 1.5 Per-Algorithm Class Structure

每个具体实现文件定义一个类, 继承AivCommBase:

```cpp
// aiv_all_reduce_910b_smalldata.h
class AivAllReduceSmall910B : public AivCommBase {
    template<typename T>
    __aicore__ inline void Process(GM_ADDR input, GM_ADDR output, uint64_t len, int32_t tag);
};
```

标准使用模式(4步):
```cpp
template<typename T>
__aicore__ inline void aiv_all_reduce_910b_smalldata(KERNEL_ARGS_DEF) {
    AivAllReduceSmall910B op;       // 1. 实例化
    op.Init(KERNEL_CLASS_INIT, false); // 2. 初始化
    op.HeadCounter();                // 3. 维测: 头计数
    op.Process<T>(input, output, len, tag); // 4. 执行
    op.TailCounter();                // 5. 维测: 尾计数
}
```

### 1.6 Synchronization Protocol

AIV核间/卡间同步使用GM(Global Memory)上的flag信号, 有三种语义:

1. **Record/Wait** — 1v1点对点信号: 写对端的flag区域, 轮询读自己的flag区域
2. **Record1vN/Wait1vN** — 1对多广播信号: 一个核写flag, 多个核等待
3. **RecordNv1/WaitNv1** — 多对1汇聚信号: 多个核原子累加, 一个核等待达到预期值

flag区域布局:
- 每个rank在GM_OUT上保留3MB flag空间 (AIV_FLAG_BUFFER_SIZE = 3MB)
- 按offset分段: 0~1MB(数据flag) / 1~2MB(清空用) / 2~3MB(sync用) / 3~4MB(偶数tag日志) / 4~5MB(奇数tag日志)
- 每个flag占32字节(FLAG_SIZE=32), 对齐要求

Ping-Pong机制: tag % 2 == 0 用offset 0, tag % 2 != 0 用offset 16MB, 避免相邻调用的flag冲突。

### 1.7 Data Movement Patterns

**CpGM2GM**: GM到GM的数据搬运, 必须经过UB中转:
```
while (count > 0) {
    localIn = inOutQue.AllocTensor();   // UB分配
    DataCopyGM2UB(localIn, inputGT);    // GM→UB (MTE2通道)
    inOutQue.EnQue(localIn);            // 入队
    localOut = inOutQue.DeQue();        // 出队
    DataCopyUB2GM(outputGT, localOut);  // UB→GM (MTE3通道)
    inOutQue.FreeTensor(localOut);      // UB释放
}
```

Double-Buffer优化: UB_MAX_DATA_SIZE=190KB, double buffer时每段95KB, 允许MTE2和MTE3流水并行。

对齐处理: 32B对齐时用DataCopy, 非对齐时用DataCopyPad(逐字节参数)。

**CpGM2GMWithFlagWrap**: 带flag通知的GM搬运, 每搬flushFrequency(默认8)段后写一次flag, 供对端跟踪进度。

### 1.8 Chip-Specific Differences: 910B vs 91093

| 维度 | 910B | 91093 (A3) |
|------|------|-----------|
| 最大卡数 | 16 (server内) | 768 (超节点内) |
| AIV核数 | 与rankSize相关 | 48核(MAX_NUM_BLOCKS) |
| 基类 | AivCommBase | AivCrossNode91093Base |
| 核分工 | block_idx与rank直接对应 | blockGroup/blockNumPerGroup多核服务一个rank |
| RDMA | 支持(aivRdmaStep) | 无(跨节点靠CCL buffer) |
| 确定性 | 专门deter_*实现 | 专门91093_deter实现 |

91093的核分工逻辑特殊: 48个AIV核可能服务768个rank, 所以每个核要服务numTargets个对端rank, 通过targetRanks[]数组记录。

### 1.9 SuperKernel Mode

SuperKernel将多个独立kernel调用融合为一次kernel launch, 减少host-device交互开销:

```cpp
// aiv_ar_superkernel.h
extern "C" __aicore__ void sk_allreduce(SUPERKERNEL_LITE_ARGS_DEF) {
    SUPERKERNEL_LITE_ARGS_EXTRACT;  // 从参数基地址(get_para_base)取3个指针
    if (devType == DEV_TYPE_910_93) return sk_all_reduce_91093(SUPERKERNEL_ARGS_CALL);
    else if (devType == DEV_TYPE_910B) {
        if (args->len * args->unitSize > UB_MAX_DATA_SIZE)
            return sk_all_reduce_910b_bigdata_graph(SUPERKERNEL_ARGS_CALL);
        else return sk_all_reduce_910b_smalldata_graph(SUPERKERNEL_ARGS_CALL);
    }
}
```

SuperKernel参数通过AivSuperKernelArgs结构体传递(在GM上), 包含buffersIn[16]/buffersOut[16]/rank/rankSize等, 避免参数数量膨胀。

### 1.10 NPU Direct RDMA

`aiv_npu_direct_base.h` 定义了AIV核直接操控RoCE QP的数据结构:

- **HcclRMAInfo**: 全局QP/MR信息(curRankId, rankNum, qpNum, sqPtr, scqPtr, rqPtr, rcqPtr, memPtr)
- **HcclAiRMAWQ**: 发送/接收WQ(wqn, bufAddr, wqeSize, depth, headAddr, tailAddr, dbMode, dbAddr)
- **hns_roce_rc_sq_wqe**: WQE格式(byte_4, msg_len, immtdata, rkey, remoteVA)

AIVRDMAPostSend (aiv_communication_base.h:964-999): AIV核直接写WQE到SQ, 包含:
1. 缓存刷新(cacheWriteThrough)
2. Spin-wait等SQ有空位
3. 构造WQE(opcode=RDMA_WRITE, owner_bit, IBV_SEND_SIGNALED)
4. 设置数据段(len, lkey, localVA)
5. 写doorbell(DB)触发硬件发送

### 1.11 Anti-Patterns

**AP-AIV-1: 超长宏参数列表**
`KERNEL_ARGS_DEF` 宏展开为32个GM_ADDR + 20+标量参数 = 52+个参数。
这是设备侧kernel ABI限制(参数必须通过kernel launch传递), 但使得接口极难维护。
SuperKernel通过AivSuperKernelArgs结构体部分缓解。
文件: `aiv_communication_base.h:86-99`

**AP-AIV-2: 所有成员变量public**
AivCommBase和AivCrossNode91093Base的所有成员变量都是public(注释掉了protected), 无封装。
文件: `aiv_communication_base.h:551` (注释 `//protected:`)

**AP-AIV-3: IDX_0~IDX_15常量定义**
定义了16个常量IDX_0到IDX_15, 值就是0到15, 用于InitBuffArray中避免"魔法数字"。
但这种做法增加了代码噪声(arr[IDX_3]不比arr[3]更清晰), 应该直接使用循环。
文件: `aiv_communication_base.h:209-224`, 使用处: `aiv_communication_base.h:363-396`

**AP-AIV-4: CeilDiv除零保护返回a**
```cpp
if (b == 0) return a;  // 应该返回0或trap
```
文件: `aiv_communication_base.h:675-680`, `aiv_crossnode_91093_base.h:226-232`

**AP-AIV-5: 分派树代码重复**
`aiv_communication.h` 中各算子的分派宏存在大量结构性重复(isOpBase/devType/len判断), 难以维护。

---

## Part 2: Communicator Base (New Framework)

### 2.1 File Inventory

`communicator/` (不含legacy/) 包含30个文件(15对.h/.cc + 4个独立.h)。
总行数: 3604行。

最大文件:
- `topo_info_extractor.cc` (1477行) — 拓扑信息提取核心实现
- `comm_ahc_base_pub.h` (178行) — AHC算法通信域基类
- `calc_ahc_transport_req_base.cc` (155行) — AHC传输需求计算基类

### 2.2 CalcTransportReqBase Hierarchy

传输需求计算(确定每个rank需要与哪些rank建链)采用继承体系:

```
CalcTransportReqBase (base, 63行)
  ├── CalcRingTransportReq     — Ring拓扑: 与前后各1个rank建链
  ├── CalcMeshTransportReq     — Mesh拓扑: 与所有其他rank建链
  ├── CalcHDTransportReq       — Halving-Doubling拓扑
  ├── CalcP2PTransportReq      — 点对点: 仅与peer建链
  ├── CalcNBTransportReq       — Nonuniform Bruck
  ├── CalcNHRTransportReq      — Nonuniform Hierarchical Ring
  ├── CalcNHRV1TransportReq    — NHR V1变体
  ├── CalcPartialMeshTransportReq — 部分Mesh
  ├── CalcHccsPlusSioTransportReq — HCCS+SIO
  ├── CalcAHCTransportReqBase  — AHC算法基类
  │   ├── CalcAHCTransportReq      — AHC标准
  │   └── CalcAHCBrokeTransportReq — AHC非对称
  └── (未来扩展)
```

基类接口:
```cpp
class CalcTransportReqBase {
    virtual HcclResult CalcTransportRequest(tag, inputMemType, outputMemType,
        commParaInfo, commTransport, subUserRankRoot);
protected:
    const u32 GetSubCollectiveRank(vecPara) const;  // userRank→子通信域rank
    HcclResult GetRankByUserRank(vecPara, userRank, &rank) const;
};
```

### 2.3 Ring Transport Req (calc_ring_transport_req.cc)

Ring拓扑建链逻辑:
- 每个rank与前后各1个rank建链: `(rank + 1) % rankSize` 和 `(rank - 1 + rankSize) % rankSize`
- 跳过Level1中非bridge rank(级间通信只有bridge rank参与)
- CalcDstRanks静态方法: 同样的前后neighbor逻辑, 用于AHC注册

注册惯用法: `REGISTER_AHC_COMM_CALC_FUNC(AHC_TEMPLATE_RING, CalcRingTransportReq, CalcRingTransportReq::CalcDstRanks);`
这是一个编译时注册宏, 将CalcDstRanks函数指针注册到全局AHCCommCalcFuncRegistry单例。

### 2.4 Mesh Transport Req (calc_mesh_transport_req.cc)

Mesh拓扑: 每个rank与所有其他rank建链(`rankIndex != rank` → isValid=true)。
特殊: `meshSinglePlane==true`时只创建1个通信平面(非确定性910B场景)。

### 2.5 AHC (Asymmetric Hierarchical Concatenate) Framework

AHC是处理非对称拓扑(不同server卡数不等)的算法框架:

```
CommAHCBaseInfo (base)
  ├── CommBrokeAlignInfo  — Broke对齐(按组切分数据)
  └── CommAHCAlignInfo    — AHC对齐(逻辑同号卡映射)
```

核心概念:
- **subGroups**: 第一层组内分组, 每组包含多个rank
- **AHCLevel**: 0级(组内) / 1级(组间)
- **AHCOpType**: AllGather / AllReduce / ReduceScatter
- **AHCTemplateType**: NB / Ring / NHR (组内/组间可用不同算法)
- **AHCConcOpType**: (AHCLevel, ConcType, AHCOpType) 三元组做map key
- **AHCAlgOption**: 配置每个(level, concType, opType)使用哪种TemplateType

CalcAHCTransportReqBase:
- 继承CalcTransportReqBase
- 额外持有: globalSubGroups(全局分组), ahcAlgOption(算法选项), isUsedRdmaMap(RDMA使用)
- CalcDstRanks额外依赖ringIndex(子通信域索引)
- RefreshTransportIsUsedRdma: 根据isUsedRdmaMap刷新每条链路的RDMA标志

### 2.6 AHC Template Register (calc_ahc_template_register.h)

```cpp
// TemplateType → AHCTemplateType 映射表
const std::map<TemplateType, AHCTemplateType> templateToAHCCalcTemplateMap = {
    {TEMPLATE_REDUCESCATTER_NB, AHC_TEMPLATE_NB},
    {TEMPLATE_ALL_REDUCE_RING, AHC_TEMPLATE_RING},
    {TEMPLATE_ALL_GATHER_NHR, AHC_TEMPLATE_NHR},
    ...
};

// 注册宏: 将静态函数指针注册到全局Registry
#define REGISTER_AHC_COMM_CALC_FUNC(type, calcAlgName, calcFunc) \
    static HcclResult g_func_##calcAlgName##_##__COUNTER__ \
        = AHCCommCalcFuncRegistry::Instance().Register(type, calcFunc)
```

### 2.7 SearchPath (search_path.h)

DFS路径搜索, 用于在910_93设备的SIO互联拓扑中寻找环路:

```cpp
class SearchPath {
    std::vector<u32> Search(const std::vector<u32> &nicList, bool isDoubleRingMap = false);
private:
    bool dfs(nicList, arrived, idx, nowNicIdx, isDoubleRingMap);

    // 硬编码的910_93拓扑可达关系(16个NIC)
    std::map<int, std::vector<u32>> reachableRank_ = {
        {0, {1, 2, 4, 6, 8, 10, 12, 14}},  // NIC 0可达NIC 1,2,4,...
        ...
    };
    std::map<int, std::vector<u32>> reachableDoubleRing_ = {...};
};
```

这是一个纯算法类, 硬编码了910_93的16-NIC拓扑可达矩阵:
- 偶数NIC(0,2,4,...): 可达所有偶数NIC + 相邻奇数NIC
- 奇数NIC(1,3,5,...): 可达所有奇数NIC + 相邻偶数NIC
- DoubleRing模式: 限制连接(偶数NIC仅连相邻奇数, 奇数NIC连所有偶数)

### 2.8 CommUtils (comm_utils.h)

通信域类型定义:

**CommPlane枚举** (16种):
```
COMM_LEVEL0(server内) / COMM_LEVEL0_ANYPATH_RDMA / COMM_LEVEL1(server间) /
COMM_LEVEL1_ANYPATH_RDMA / COMM_LEVEL1_AHC / COMM_LEVEL2(超节点间) /
COMM_MESH_L0/L1 / COMM_COMBINE / COMM_COMBINE_ORDER /
COMM_LEVEL0/1_ANYPATH_SDMA / COMM_LEVEL0/1_LOGICAL / COMM_ARS / COMM_COMBINE_L1
```

**CommType枚举** (18种):
```
RING_INNER / RING_COMBINED / HALVING_DOUBLING / STAR /
NONUNIFORM_HIERARCHICAL_RING / WHOLE_NHR / NHR_V1 / WHOLE_NHR_V1 /
AHC / WHOLE_AHC / AHC_BROKE / WHOLE_AHC_BROKE /
NB / WHOLE_NB / MESH_COMBINED / MESH / P2P / PARTIAL_MESH / HCCS_PLUS_SIO
```

**CommParaInfo结构体**: 建链参数(commPlane, commType, root, peerUserRank, isAicpuModeEn, meshSinglePlane, forceRdma等)

### 2.9 TopoInfoExtractor (topo_info_extractor.h/cc, 1477+149行)

拓扑信息提取器, 是communicator/目录中最大最复杂的文件。

职责: 根据rankVector(全部rank信息)和topoType, 计算出各级通信域的rank分组关系。

初始化流程:
```
Init()
  ├── CheckInitInfo()      — 参数校验(rankSize, topoType, deviceType)
  ├── InitAHCConfig()      — AHC算法配置初始化
  ├── SetRankInfo()         — 填充serverToRank_, superPodToRank_, deviceLinkTypeMap_
  ├── SetTopologyInfo()     — 计算各级通信域
  │   ├── SetTopoDefaultInfo()   — Combined平面(所有rank蛇形排列)
  │   ├── SetTopoInfoForLevel0() — Server内通信域
  │   ├── SetTopoInfoForLevel1() — Server间通信域(bridge rank参与)
  │   ├── SetTopoInfoForLevel2() — 超节点间通信域
  │   ├── SetTopoInfoForARS()    — ARS通信域
  │   ├── SetTopoInfoForCombineL1() — 超节点打平通信域
  │   ├── SetAHCSubGroupsAndAlgOption() — AHC分组和算法选项
  │   ├── SetTopoInfoForMeshL0/L1() — Mesh通信域(可选)
  │   └── COMM_COMBINE_ORDER    — 按rank排序的打平通信域
  ├── CheckPlaneInfo()     — 校验平面数与拓扑类型匹配
  └── SetRankMap()          — 构建userRank↔subCommRank双向映射
```

核心数据结构:
- `CommPlaneVector_[COMM_LEVEL_RESERVED]`: 各级通信域的rank列表(3维: [level][ringIndex][rankIndex])
- `CommPlaneSubGroupVector_[COMM_LEVEL_RESERVED]`: 各级通信域的AHC分组(4维)
- `serverToRank_`: 按serverIdx分组的rank信息
- `superPodToRank_`: 按superPodIdx分组的rank信息
- `isBridgeVector_`: 每个ring中当前rank是否为bridge rank

拓扑类型(TopoType)映射到ranksOneNode_:
```cpp
ranksOneNode_ = { 0, 8, 4, 2, 1, 4, rankNumPerAggregation, 0, rankNumPerAggregation, rankNumPerAggregation };
// index: RESERVED/8P_RING/4P_MESH/2P_MESH/1P_MESH/4P_RING/NP_SINGLE_RING/...
```

---

## Part 3: Legacy Communicator

### 3.1 File Inventory

`communicator/legacy/` 包含14个文件(7对.h/.cc) + CMakeLists.txt。
总行数: 2938行。

最大文件:
- `comm_base.cc` (1229行) — CommBase实现(链路创建、初始化)
- `comm_factory.cc` (745行) — CommFactory(通信域工厂)
- `comm_star.cc` (309行) — Star拓扑通信域
- `comm_factory_pub.h` (166行) — CommFactory公开接口
- `comm_ring.cc` (125行) — Ring拓扑通信域

### 3.2 Thin Wrapper Pattern

所有legacy/*.h文件都是超薄包装器, 仅包含一行include指向`inc/`目录下的`_pub.h`:

```cpp
// comm_base.h
#include "comm_base_pub.h"
// comm_ring.h
#include "comm_ring_pub.h"
// comm_factory.h
#include "comm_impl.h" + "calc_impl.h" + "comm_factory_pub.h"
```

实际的类定义在`src/algorithm/base/inc/`下的_pub.h文件中。这是hcomm的标准接口隔离模式: 实现目录中的.h只做转发, 真正的接口在inc/中。

### 3.3 CommBase Class (comm_base_pub.h)

CommBase是legacy通信域的基类, 管理rank间Transport链路的创建和生命周期:

构造函数参数(25+个):
- collectiveId, userRank, userRankSize, rank, rankSize, paraVector
- topoFlag, dispatcher, notifyPool, netDevCtxMap, exchanger
- inputMem, outputMem, isUsedRdmaLevel0, transportResourceInfo
- tag, nicDeployInner, isAlltoAllCommMesh, useOneDoorbell
- isAicpuModeEn, rankRoot, isHaveCpuRank, useSuperPodMode, expMem

初始化流程(comm_base.cc:81-100):
```
Init()
  ├── SetRankMap()    — 构建rank↔userRank映射
  ├── hrtGetDevice()  — 获取设备ID
  ├── CreateLinks()   — 创建与其他rank的Transport链路
  └── CheckLinks()    — 校验链路有效性
```

析构: DeInit()先join所有linkThreads_(多线程建链), 然后逐个DeInit transportInfo_。

### 3.4 CommRing / CommMesh / CommStar / CommHD / CommP2P

这些类继承CommBase, 定义在`inc/comm_*_pub.h`中, 当前目录下只有空的.h和简单的.cc:

- **CommRing**: Ring拓扑, `comm_ring.cc`(125行) — CreateLinks的Ring特化
- **CommMesh**: Mesh拓扑, `comm_mesh.cc`(64行) — CreateLinks的Mesh特化
- **CommStar**: Star拓扑, `comm_star.cc`(309行) — root-based星形拓扑
- **CommHD**: Halving-Doubling, `comm_halving_doubling.cc`(83行)
- **CommP2P**: 点对点, `comm_p2p.cc`(88行) — 仅与peer建链

### 3.5 CommFactory (comm_factory_pub.h, comm_factory.cc)

通信域工厂, 根据CommParaInfo创建对应类型的CommBase实例:

```cpp
class CommFactory {
    HcclResult Init();
    HcclResult CreateCommPlane(tag, inputMem, outputMem, commParaInfo, commVec, expMem);

    // 内部: 按CommType分派到具体创建方法
    HcclResult CreateCommRing(...);
    HcclResult CreateCommHD(...);
    HcclResult CreateCommStar(...);
    HcclResult CreateCommMesh(...);
    HcclResult CreateCommP2P(...);
};
```

Init流程:
1. 从TopoInfoExtractor获取CommPlaneVector_/isBridgeVector_/rankData_/serverToRank_等
2. 初始化reusedSocketManager_(连接复用)
3. 获取设备ID

关键设计:
- CommFactory持有`shared_ptr<TopoInfoExtractor>`, 拓扑信息由外部注入
- 创建通信域时通过CalcTransportReq系列类计算建链需求, 然后传给CommBase的子类
- 支持异步P2P创建: `CreateCommP2PAsync` + `CreateCommP2PQuarry`(轮询查询状态)

### 3.6 CommFactory与New Framework的关系

CommFactory是V1框架的核心, 在V2框架中被TopoInfoExtractor + CalcTransportReq + CollComm替代:
- V1: CommFactory.CreateCommPlane() → 内部计算拓扑+创建CommBase实例
- V2: TopoInfoExtractor计算拓扑 → CalcTransportReq计算建链需求 → CollComm/MyRank管理连接

但当前两者共存: CommFactory仍被legacy CommunicatorImpl使用。

---

## Part 4: Generic Algorithm Templates (alg_template/)

### 4.1 File Inventory

`alg_template/` 包含303个文件(185个.h + 118个.cc), 按集合操作类型分12个子目录:

| 子目录 | 集合操作 | 文件数(估) |
|--------|----------|-----------|
| temp_all_reduce/ | AllReduce | ~40 |
| temp_reduce_scatter/ | ReduceScatter | ~50 |
| temp_all_gather/ | AllGather | ~45 |
| temp_broadcast/ | Broadcast | ~25 |
| temp_scatter/ | Scatter | ~20 |
| temp_gather/ | Gather | ~8 |
| temp_reduce/ | Reduce | ~8 |
| temp_alltoall/ | AllToAll | ~8 |
| temp_alltoallv/ | AllToAllV | ~15 |
| temp_send_recv/ | SendRecv | ~5 |
| component/ | 共享组件(Sender/Reducer) | ~6 |
| inc_all_reduce_deter/ | 确定性AllReduce辅助 | ~6 |

量化指标(Grep统计):
- REGISTER_TEMPLATE: 104次/103文件 (102个具体算法模板 + 注册基础设施)
- CHK_RET: 2786次/122文件
- RunAsync调用: 200+次
- GetAlgTemplate: 29次/18文件 (跨算法动态获取子模板)
- `using AlgTemplateBase::Prepare`: 23文件 (避免C++ name hiding)
- reinterpret_cast: 36次/12文件 (主要集中在alltoallv)
- TODO/FIXME/HACK: 0次 (干净)

### 4.2 ExecutorBase (= AlgTemplateBase) — 核心基类

文件: `alg_template_base_pub.h` (682行)

ExecutorBase是全部集合通信算法的基类。末尾有类型别名 `using AlgTemplateBase = ExecutorBase`。

核心虚方法(扩展点):

| 方法 | 默认实现 | 语义 |
|------|---------|------|
| `RunAsync(rank, rankSize, links)` | 返回HCCL_SUCCESS | 算法主执行入口, 子类必须override |
| `RunAsyncStaged(rank, rankSize, links, stage)` | 返回HCCL_SUCCESS | 分阶段执行(prepare/reduce_scatter/allgather) |
| `Prepare(...)` | 赋值成员变量 | 两段式构造的第二步, 30+个重载 |
| `PrepareRunAsync(rank, rankSize, links)` | 空 | RunAsync前的准备(slice计算等) |
| `GetNslbAdjInfo(...)` | 空 | NSLB调度邻接信息 |
| `GetHcclOffsetDstRanksMap(...)` | 空 | alltoallv类AICPU cache |

设计决策: 全部虚方法提供默认空实现(返回HCCL_SUCCESS)而非pure virtual。
优点: 子类只需override自己关心的方法。
代价: 编译器不会捕获子类忘记override RunAsync的情况。

关键protected成员变量:
```
dispatcher_       — HcclDispatcher (signal调度)
inputMem_         — DeviceMem (用户输入)
outputMem_        — DeviceMem (用户输出)
scratchMem_       — DeviceMem (临时buffer)
slices_           — vector<Slice>& (数据分片)
count_            — u64 (元素数)
dataType_         — HcclDataType
reductionOp_      — HcclReduceOp
root_             — u32 (root rank)
stream_           — Stream (主stream)
barrierSwitchOn_  — bool (barrier开关)
```

工具方法:
- `ExecuteBarrier(preLink, aftLink)` — 5个重载, link间同步
- `ExecuteTxSync/ExecuteRxSync` — 同步发送/接收
- `DataUnitSize(dataType)` — 查表获取字节大小
- `RoundUpWithDivisor` — 向上对齐
- `PrepareSliceData/PrepareSliceMeshStreams` — 静态slice准备
- `CalcLinksRelation` — 静态方法, 计算halving-doubling的link关系
- `RegisterProfiler` — 性能profile注册

业务常量:
- `HCCL_CHUNK_SIZE = 1GB` — chunk大小
- `HCCL_MIN_SLICE_ALIGN_910B = 16384` / `_910_93 = 16384` — 910B/910_93 slice对齐
- `HCCL_MIN_SLICE_ALIGN = 128` — 通用slice对齐
- `HCCL_NIC_MAX_NUM = 8` — 最多8个NIC
- `HCCL_SPLIT_SIZE_INTER_SERVER = 8MB` — 跨机通信切分边界

RunStage枚举: `RUN_PREPARE / RUN_REDUCE_SCATTER / RUN_ALLGATHER / RUN_ALLREDUCE / RUN_DEFAULT`

TemplateType枚举: 定义103个内置模板类型(0~102) + 自定义范围[1000, 2000)。

### 4.3 三层继承体系

```
ExecutorBase (= AlgTemplateBase)           ~71 个直接子类
  |
  +-- NHRBase (Nonuniform Hierarchical Ring)   8 个子类
  |     AllReduceNHR, AllReduceNHROneshot, ReduceScatterNHR,
  |     AllGatherNHR, BroadcastNHR, BroadcastNHROneshot,
  |     ScatterNHR, ReduceNHROneshot
  |
  +-- NBBase (Nonuniform Bruck)                6 个子类
  |     AllReduceNB, AllGatherNB, ReduceScatterNB,
  |     BroadcastNB, BroadcastNBBinary, ScatterNB
  |
  +-- NHRV1Base                                独立分支(V1遗留)
  +-- AHCAlgTemplateBase                       非对称层次拼接
  +-- RecursiveHalvingDoublingBase              递归倍增
  +-- AlltoallPipelineBase                     AllToAll流水线
  +-- MultiDeterPipeline                       确定性多流水线
```

中间基类提供的抽象:
- NHRBase: rank映射(线性→二叉树通信序列)、slice合并、步数计算(`ceil(log2(rankSize))`)、三阶段barrier
- NBBase: `CalcCeilLog2`工具、小数据常量(`NB_ALLREDUCE_SMALL_SIZE = 128KB`)
- AHCAlgTemplateBase: 非对称拓扑的subGroup划分和算法配置

### 4.4 Registry — 静态注册 + 工厂模式

文件: `alg_template_register.h` + `alg_template_register.cc`

`AlgTemplateRegistry` 是singleton, 内部持有 `vector<AlgTemplateCreator>`(大小2000, 支持内置+自定义模板)。

注册流程:
1. 每个.cc文件末尾使用宏: `REGISTER_TEMPLATE(TemplateType::TEMPLATE_XXX, ClassName)`
2. 宏展开为文件级静态变量, 调用 `AlgTemplateRegistry::Instance().Register(type, DefaultTemplateCreator<T>)`
3. `DefaultTemplateCreator<T>` 使用 `static_assert(is_base_of<AlgTemplateBase, P>)` 做编译期继承检查

获取模板:
```cpp
unique_ptr<AlgTemplateBase> alg = AlgTemplateRegistry::Instance().GetAlgTemplate(type, dispatcher);
```

这是项目中最彻底的Registry模式实现, 覆盖全部102个算法模板。

### 4.5 两段式构造 (Prepare 模式)

每个算法模板使用两段式构造:
1. 通过Registry工厂获取实例: `GetAlgTemplate(type, dispatcher)` → `unique_ptr<AlgTemplateBase>`
2. 调用Prepare初始化: 先调算法特有的Prepare重载, 再调基类通用Prepare设置inputMem_/outputMem_/count_等

代码注释明确说明:
```cpp
// 新增的两段式构造函数，获取实例后要无脑调用实现构造函数功能，
// 后续还要调用其它的基类Prepare函数实现其它成员变量初始化
```

基类声明了~30个virtual Prepare重载(按参数数量1~19分组)。
子类使用 `using AlgTemplateBase::Prepare;` 引入基类重载避免name hiding (23个文件)。
新代码使用PrepareData结构体替代长参数列表, 减少重载数量。

### 4.6 Component: Sender & Reducer

`component/` 封装了算法模板共享的点对点通信原语:

Sender (`sender_pub.h`, 62行): 封装向远端发送数据, 根据link能力自动选路径:
1. RDMA Write With Reduce (标准RoCE或支持RDMA reduce)
2. Inline Reduce信号通知 (link支持inline reduce时, 仅发信号不传数据)
3. TxAsync fallback

Reducer (`reducer_pub.h` 89行 + `reducer.cc` 293行): 接收数据并reduce, 四条路径:
1. RDMA Reduce路径: RxAsync + 可选D2DMemcpy
2. Standard RoCE RxWithReduce: 一次调用完成接收+规约
3. Inline Reduce: RxDataSignal + GetRemoteMem + HcclReduceAsync (直接读远端内存)
4. 普通路径: RxAsync + HcclReduceAsync (先接收到scratch, 再reduce)

支持pre/post sync hook (`SetPreSyncFunc`/`SetPostSyncFunc`注入lambda)和batch操作。

### 4.7 典型算法结构: AllReduce分解

AllReduce统一分解为ReduceScatter + AllGather两阶段(所有变体一致):

```cpp
// AllReduceRing::RunAsync
HcclResult RunAsync(rank, rankSize, links) {
    PrepareRunAsync(rank, rankSize, links);  // slice计算
    RunReduceScatter(rank, rankSize, links); // 阶段1
    RunAllGather(rank, rankSize, links);     // 阶段2
}
```

内部通过Registry动态获取子算法模板:
```cpp
// AllReduceRing::RunReduceScatter
auto tempAlg = AlgTemplateRegistry::Instance().GetAlgTemplate(
    TemplateType::TEMPLATE_REDUCESCATTER_RING, dispatcher_);
tempAlg->Prepare(reduceAttr_);
tempAlg->Prepare(inputMem_, inputMem_, outputMem_, count_, dataType_, stream_, ...);
tempAlg->RunAsync(rank, rankSize, links);
```

RunAsyncStaged通过RunStage枚举将执行拆为三步, 允许上层在阶段间插入同步。

例外: AllReduceReduceBcast使用Reduce + Broadcast(而非RS+AG)。

### 4.8 Buffer三区域模型与Slice切分

三块设备内存:
- `inputMem_`: 用户输入数据
- `outputMem_`: 用户输出(AllReduce场景可与input相同)
- `scratchMem_`: 临时buffer(接收远端数据再reduce)

数据按rankSize切分为`vector<Slice>`, 每个Slice = `{offset, size}`。
切分对齐: 通用128字节, 910B/910_93为16384字节。

### 4.9 Ring算法数据流

ReduceScatterRing:
- 每步向右发送(Sender), 从左接收并reduce(Reducer)
- TxAck/RxAck同步, RxWaitDone/TxWaitDone确认
- 循环rankSize-2次, 最后一步reduce到output

AllGatherRing:
- 先将inputMem_拷贝到outputMem_对应位置
- 每步向右发送/从左接收(纯搬运, 不reduce)
- 使用link->TxAsync/RxAsync(不经Sender/Reducer)

网口裁剪(NIC trimming): rankSize=8但实际网口<8时启用Chunk变体, 合并多个slice适配物理拓扑。

### 4.10 值得注意的模式

1. Barrier开关: `barrierSwitchOn_`可通过`CloseBarrier()`关闭。RS+AG组合时, RS尾部barrier可省略(AG首次Ack已隐含同步)。

2. rankSize=1快速路径: 所有模板一致: input!=output则D2DMemcpy, 否则直接返回。

3. reduceAttr位掩码: `RDMA_REDUCE_BITMASK`/`INLINE_REDUCE_BITMASK`控制reduce路径选择, 根据dataType+reductionOp组合决定是否硬件加速。

4. Profiler集成: 每个模板在RunAsync前调用RegisterProfiler(planeID, stage, step, stream)。

5. GetNslbAdjInfo: 每个模板实现此方法返回通信拓扑邻接信息(目标rank/phase ID), 供NSLB做负载均衡。

6. AllReduce内部组合: AllReduce通过GetAlgTemplate动态获取RS/AG子模板(29处调用), 是算法层最重要的跨模板组合模式。

### 4.11 CommBase与AlgTemplateBase的关系

CommBase和AlgTemplateBase不是继承关系, 而是组合:
- CommBase是"链路管理者" — 创建和持有rank间的Transport links
- AlgTemplateBase是"算法执行者" — 接收links作为RunAsync参数
- 桥接: `CommBase::RunTemplateAlg(unique_ptr<AlgTemplateBase>&)` 调用 `tempAlg->RunAsync(Rank(), RankSize(), transportInfo_)`

### 4.12 Anti-Patterns

AP-ALG-1: Prepare重载膨胀
基类声明~30个virtual Prepare重载(1~19个参数), 是God Interface反模式。
注释按参数数量分组(`/* 12个参数 */`)是应对策略, 但根本原因是缺乏参数聚合。
新代码已开始用PrepareData结构体, 但旧重载未清理。
文件: `alg_template_base_pub.h`

AP-ALG-2: 默认空实现掩盖遗漏
所有虚方法返回HCCL_SUCCESS而非pure virtual, 子类忘记override RunAsync不会报编译错误。
文件: `alg_template_base_pub.h` 所有virtual方法

AP-ALG-3: const修饰标量返回值
`inline const u32 Rank() const` — 返回值的const对标量类型无意义。
文件: `alg_template_base_pub.h`

AP-ALG-4: CalcLinksRelation职责错位
`CalcLinksRelation`是static方法, 计算halving-doubling的link关系, 与ExecutorBase的"算法执行"职责不匹配。
应提取到独立工具类或HalvingDoubling相关基类中。
文件: `alg_template_base_pub.h`

AP-ALG-5: linkDummy_ public泄漏
`linkDummy_`是public成员, 作为"无效link"的哨兵值使用, 泄漏实现细节。
文件: `alg_template_base_pub.h:133`

AP-ALG-6: 长构造函数参数列表
CommBase构造函数25+个参数。子类(CommRing, CommMesh等)构造函数也有20+个参数。
应使用Builder或参数结构体。
文件: `comm_base_pub.h`

---

## Cross-Cutting Observations

### O1: Header-Only vs .h/.cc Split

- AIV模板: 100% header-only (设备侧kernel的模板实例化需要)
- communicator/: 标准.h/.cc分离 (主机侧C++)
- legacy/: .h仅做转发, 实际接口在inc/_pub.h中

### O2: Naming Conventions

- AIV: 类名`AivAllReduceSmall910B`(Aiv前缀 + 算子 + 芯片), 函数`aiv_all_reduce_910b_smalldata`(下划线, C ABI)
- communicator: 类名`CalcRingTransportReq`(PascalCase), 方法`CalcTransportRequest`
- 芯片: 910B / 910_93 / 91093 命名不统一(下划线时有时无)

### O3: Error Handling Divergence

- AIV (设备侧): AIV_ERROR宏 → PrintfImpl + trap() (不可恢复, 直接停机)
- communicator (主机侧): CHK_RET/HCCL_ERROR → HcclResult返回值 (可恢复, 上层处理)

### O4: Topology Knowledge Distribution

拓扑知识分散在三处:
1. `search_path.h`: 硬编码910_93的16-NIC可达矩阵
2. `topo_info_extractor.cc`: 通过ranksOneNode_数组编码各TopoType的每节点设备数
3. AIV base: MAX_RANK_SIZE=16(910B), MAX_RANK_SIZE_A3=768(91093), MAX_NUM_BLOCKS=48

### O5: Data Size Thresholds

AIV算法根据数据量选择实现, 阈值分散在aiv_communication_base.h:
- AIV_ALL_REDUCE_BIG_SIZE = 16MB
- AIV_ALL_REDUCE_SMALL_SIZE = 64KB
- UB_MAX_DATA_SIZE = 190KB (UB缓冲区大小)
- AIV_ALL_GATHER_SMALL_SIZE = 700KB
- AIV_REDUCE_SCATTER_MID_SIZE = 2MB
- AIV_ALL_TO_ALL_BIG_SIZE = 512KB
- AIV_ALL_REDUCE_DETER_SMALL/MID_SIZE = 1MB/8MB

Generic模板层的常量:
- HCCL_CHUNK_SIZE = 1GB
- HCCL_SPLIT_SIZE_INTER_SERVER = 8MB (跨机切分)
- HCCL_MIN_SLICE_ALIGN = 128 (通用对齐)
- HCCL_MIN_SLICE_ALIGN_910B/910_93 = 16384 (芯片对齐)
- NB_ALLREDUCE_SMALL_SIZE = 128KB (NB算法小数据阈值)

### O6: 三层算法分工

Algorithm层存在三个并行的算法实现体系, 各有独立职责:
1. alg_template/ (generic): 主机侧编排——决定step/slice/barrier, 通过Transport link发数据
2. alg_aiv_template/ (device): 设备侧执行——AIV核间/卡间的GM搬运和reduce
3. communicator/ (infra): 拓扑计算和建链——决定谁和谁通信, 用什么协议

它们的接触点:
- generic模板通过CommBase的transportInfo_使用communicator建的链路
- generic模板编排的"kernel launch"最终执行AIV模板代码
- communicator的TopoInfoExtractor为generic模板提供rank映射信息

### O7: Registry模式全局统一

AlgTemplateRegistry是算法层唯一的工厂, 所有102个算法模板都通过REGISTER_TEMPLATE宏注册。
运行时通过TemplateType枚举值获取算法实例, 实现了算法选择(selector)与算法实现(template)的解耦。
这与hccl层的selector/executor配合: hccl selector选TemplateType → hcomm Registry创建具体算法。

### O8: AllReduce的组合模式

AllReduce是最复杂的集合操作, 其实现组合了RS+AG两个子算法:
- AllReduceRing = ReduceScatterRing + AllGatherRing
- AllReduceNHR = ReduceScatterNHR + AllGatherNHR
- AllReduceNB = ReduceScatterNB + AllGatherNB
- AllReduceReduceBcast = ReduceRing + BroadcastRing (例外)

这种组合通过Registry动态获取子模板实现(29处GetAlgTemplate调用), 而非继承。

---

## Phase 2.3.2: impl/ — 算法实现层

### Module Overview

`src/algorithm/impl/` 是hcomm算法层的实现层，位于算法模板(base/)之上，负责:
1. 算法选择: 根据设备类型、拓扑、数据量选择具体算法(operator/)
2. 算法编排: 管理多步算法的执行顺序、数据分片、Loop循环(coll_executor/)
3. 资源管理: CCL buffer、stream、workspace、socket等资源的生命周期(resource_manager/)
4. 任务调度: 子环线程管理、task loading(task/)
5. 遗留实现: 旧框架的通信域创建(legacy/)

文件统计: 389个C++文件，其中coll_executor/ 319个(占82%)。

### Key Files

顶层调用链:
```
CollAlgOpRegistry::GetAlgOp(opType)       -- operator/registry/
    -> CollAlgOperator::SelectAlg()       -- operator/xxx_operator.cc
        -> algName字符串                  -- 如 "AllReduceMeshAivExecutor"
            -> CollAlgExecRegistry::GetAlgExec(algName) -- coll_executor/registry/
                -> CollExecutorBase::Orchestrate()      -- 具体executor
```

---

## Part 1: CollExecutor基类体系

### 1.1 四层继承

```
CollExecutorBase                              (L0: 纯虚接口)
  +-- CollNativeExecutorBase                  (L1: Template Method骨架)
        +-- CollCommExecutor                  (L2: Multi-Ring/Multi-Stream编排工具)
        |     +-- CollAllReduceExecutor       (L3: AllReduce特化)
        |     |     +-- 33个具体Executor       (L4: 具体算法+硬件变体)
        |     +-- CollReduceScatterExecutor   (L3: RS特化, 30个变体)
        |     +-- CollAllGatherExecutor       (L3: AG特化, 25个变体)
        |     +-- CollBroadcastExecutor       (L3: Broadcast, 10个变体)
        |     +-- CollReduceExecutor          (L3: Reduce, 5个变体)
        |     +-- CollScatterExecutor         (L3: Scatter, 5个变体)
        |     +-- CollReduceScatterVExecutor  (L3: RSV, 10个变体)
        |     +-- CollAllGatherVExecutor      (L3: AGV, 9个变体)
        |     +-- CollBatchSendRecvExecutor   (L3: BatchSR, 2个变体)
        +-- CollAlltoAllExecutor              (L1直接: 跳过L2, 13个变体)
        +-- CollSendExecutor                  (L1直接: P2P Send)
        +-- CollReceiveExecutor               (L1直接: P2P Receive)
  +-- CollBatchWriteExecutor                  (L0直接: 最轻量)
```

路径: coll_executor_base.h:23-78, coll_native_executor_base.h, coll_comm_executor.h

AllToAll和Send/Receive跳过CollCommExecutor的原因:
- AllToAll的数据流模式(每rank向每rank发送不同量数据)与reduce+gather模式完全不同
- Send/Receive是P2P操作，不需要多级通信域(Level0/Level1)抽象

### 1.2 关键虚方法和Template Method

L0定义两个纯虚方法 (coll_executor_base.h:36-37):
- `CalcResRequest(OpParam&, AlgResourceRequest&)` = 0 -- 计算资源需求
- `Orchestrate(OpParam&, AlgResourceResponse&)` = 0 -- 执行算法编排

L1的CalcResRequest骨架 (coll_native_executor_base.cc:33-58):
```
CalcResRequest()
  -> CalcScratchMemSize()      // virtual, 默认返回0
  -> CalcOptimalIntraRing()    // virtual, 默认空
  -> CalcStreamNum()           // virtual, 默认返回0
  -> CalcNotifyNum()           // virtual, 默认2*streamNum
  -> CalcAivBufferRequest()    // virtual, 根据desc_标志位
  -> CalcCommInfo()            // virtual, 默认空
  -> BuildResourceRequest()    // 非virtual, 组装结果
```

L3的Orchestrate骨架(所有操作共享同一模板):
```
Orchestrate(param, algRes):
  1. startut = TIME_NOW()
  2. tag_ = param.tag; algResResp_ = &algRes;
  3. 图模式 -> 直接KernelRun(param, execMem)
     单卡 -> 短路返回或简化KernelRun
     OP_BASE多卡 -> RunLoop(param, algRes)
     零拷贝 -> KernelRunIntraServerPre/Post + RunLoop
  4. 强制LaunchTaskExtend (注释: "不要删除, 否则导致aicpu cache功能问题")
  5. HCCL_INFO("...orchestrate success, take time...")
```

强制LaunchTaskExtend出现在所有9个基类的Orchestrate结尾，是项目级强制规范。

### 1.3 desc_描述符模式

每个Executor在构造函数中设置desc_声明自己的能力:
- desc_.isAivMode -- 是否使用AIV计算引擎
- desc_.isAivCrossNode -- 是否支持跨节点AIV
- desc_.isZeroCopy -- 是否支持零拷贝
- desc_.deterministic -- 确定性等级(0/1)
- desc_.level1SupportedAlgos -- 支持的Level1算法白名单
- desc_.level2SupportedAlgos -- 支持的Level2算法白名单

SetAlgType()在运行时检查requested algType是否在supported列表中，不在则回退 (coll_executor_base.cc:21-48)。

---

## Part 2: AllReduce Executor (33个变体, 68个文件)

### 2.1 变体矩阵

AllReduce的33个注册变体由以下正交维度组合:

选择维度:
1. Level0拓扑: Ring(8P/NP_SINGLE/NP_DOUBLE) vs Mesh vs Null(打平COMM_COMBINE)
2. Level1算法: Ring / HD / NHR / NHR_V1 / NB / AHC / AHC_BROKE
3. 硬件平台: 910B / 910_93 / 310P
4. 数据量: SmallCount / MidCount / 普通 / HugeData
5. 工作模式: OP_BASE(单算子) vs OPS_KERNEL_INFO_LIB(图模式)
6. 计算引擎: SDMA / AIV / AIV+RDMA
7. 确定性: DISABLE / 普通 / STRICT
8. 内存模式: ZeroCopy vs 非ZeroCopy(CCL Buffer中转)
9. 流水线: Pipeline vs 非Pipeline

主要变体:
- Ring系列(8个): Ring, RingFor91093, ARS, FastDoubleRing, AlignedDoubleRing, Zerocopy, SmallCount910, ReducePlusBcast
- Mesh系列(10个): Mesh, MeshOpbase, MeshOpbasePipeline, MeshGraphPipeline, MeshOneshot, MeshMidCount, MeshSmallCount, MeshAiv, MeshAiv91093, MeshAivSmallCount
- Comm打平(1个): CommExecutor
- 确定性(5个): DeterPipeline, AivDeter, AivDeterSmall, MeshOpbaseSmallDeter, MeshOpbaseMidDeter
- AIV RDMA(2个): SmallCountAivRdma, MidCountAivRdma
- 其他(4个): SingleRank, Mix, OrderPreserved, MidCount91093
- 310P(3个): Ring, Doubling, DoublingDirect

### 2.2 三步AllReduce算法

核心AllReduce分解为三步:
1. Level0 ReduceScatter (PROF_STAGE_0): 节点内各rank归约自己负责的一份
2. Level1 AllReduce (PROF_STAGE_1): 节点间对每个rank的数据做AllReduce
3. Level0 AllGather (PROF_STAGE_2): 节点内各rank收集完整结果

Ring变体: MultiRingReduceScatter -> Level1AllReduce -> MultiRingAllGather
Mesh变体: MultiStreamReduceScatterMesh -> Level1AllReduce -> AllGatherMesh

### 2.3 Level1算法选择(Strategy模式)

在KernelRun中根据algType_.algoLevel1选择:
- ALG_LEVEL1_RING -> TEMPLATE_ALL_REDUCE_RING
- ALG_LEVEL1_NHR -> TEMPLATE_ALL_REDUCE_NHR
- ALG_LEVEL1_NHR_V1 -> TEMPLATE_ALL_REDUCE_NHR_V1
- ALG_LEVEL1_NB -> TEMPLATE_ALL_REDUCE_NB
- ALG_LEVEL1_AHC -> TEMPLATE_ALL_REDUCE_AHC
- ALG_LEVEL1_AHC_BROKE -> TEMPLATE_ALL_REDUCE_AHC_BROKE
- 默认 -> TEMPLATE_ALL_REDUCE_RECURSIVE_HALVING_DOUBLING

此if-else链在ring_executor.cc:161-196和mesh_executor.cc:140-175中几乎逐行重复。

### 2.4 Loop机制

当数据量超过CCL Buffer大小时，通过RunLoop切循环 (coll_all_reduce_executor.cc:186-243):
- CalcLoopMaxCount()计算每次Loop最大处理量(由CCL Buffer大小决定)
- 每次Loop: 拷入CCL Buffer -> KernelRun -> 拷出CCL Buffer
- 零拷贝场景跳过拷入/拷出

---

## Part 3: ReduceScatter与AllGather (RS 30个变体/62文件, AG 25个变体/52文件)

### 3.1 RS与AG的对称性

RS和AG继承体系高度对称，一一对应:
- CollReduceScatterRingExecutor <-> CollAllGatherRingExecutor
- CollReduceScatterMeshExecutor <-> CollAllGatherMeshExecutor
- CollReduceScatterRingFor91093Executor <-> CollAllGatherRingFor91093Executor
- 以此类推...

RS比AG多8个变体(AivDeter, AivDeterSmall, Deter, DeterPipeline, MeshDmaElimination, OrderPreserved, SmallCountDeter, FastDoubleRing)。
根本原因: RS涉及reduce操作，存在浮点非确定性问题，需要更多确定性变体。AG只做数据搬运，不需要。

AG比RS多2个变体(SmallCount, MidCountFor91093)。

### 3.2 执行顺序的反向对称

RS零拷贝: IntraServer(节点内RS) -> InterServer(节点间RS) -- 先内后外
AG零拷贝: InterServer(节点间AG) -> IntraServer(节点内AG) -- 先外后内

RS Ring KernelRun顺序:
1. 节点间RS (Level1 template)
2. 节点内RS (MultiRingReduceScatter)
3. memcpy到outputMem

AG Ring KernelRun顺序:
1. memcpy input到output
2. 节点内AG (MultiRingAllGather)
3. 节点间AG (Level1 template)

910_93三级拓扑:
RS: 节点内 -> 节点间 -> 超节点间 (从内到外)
AG: 超节点间 -> 节点间 -> 节点内 (从外到内)

### 3.3 RS特有的scratchMem和ReduceType

RS需要scratchMem暂存中间reduce结果(通过scratchMemFlag_控制)，AG不需要。

RS的ReduceType判断 (coll_reduce_scatter_executor.cc:143-145):
```cpp
ReduceType reduceType = ((param.reduceType != HCCL_REDUCE_PROD) &&
    (param.DataDes.dataType != HCCL_DATA_TYPE_INT64)) ?
    ReduceType::INLINE_REDUCE : ReduceType::TBE_REDUCE;
```
含义: PROD运算和INT64类型不支持inline reduce，必须用TBE reduce。AG无此判断。

### 3.4 CalcLoopMaxCount接口不一致

RS: `virtual u64 CalcLoopMaxCount(const u32 unitSize)` -- 通过成员变量inCCLbufferSize_访问
AG: `virtual u64 CalcLoopMaxCount(const u64 cclBuffSize, const u32 unitSize)` -- 参数传入

这是一个接口设计不一致(coll_reduce_scatter_executor.h:28 vs coll_all_gather_executor.h:24)。

---

## Part 4: 其他集合操作

### 4.1 Broadcast (10个变体, 22文件)

经典三阶段: MultiRingScatter -> 机间Broadcast(HD/NHR/NB) -> MultiRingAllGather
(coll_broadcast_ring_executor.cc:65-207)

特殊: 使用inputMem同时作input和output(broadcast只需一块CCL buffer)。

### 4.2 Reduce (5个变体, 12文件)

比AllReduce简单: 只需在所有rank做reduce，结果只送到root，不需最后的allgather。
只root需要CCL->User拷贝 (coll_reduce_executor.cc:192-196)。

### 4.3 Scatter (5个变体, 12文件)

Broadcast的逆: 只在root做User->CCL(所有N份数据)，通信后每rank只取自己那份
(coll_scatter_executor.cc:133-141)。

### 4.4 AllToAll (13个变体, 30文件)

最独特的集合操作:
- 跳过CollCommExecutor直接继承CollNativeExecutorBase(数据流模式不同)
- 独有SetExcutorExtraInfo接口: 提供所有rank的SendRecvInfo
- 独有HasMassTasks()检测(>65535个task时切换模式) -- 反映O(N^2)特性
- 两阶段Staged分解: 机内fullmesh -> 机间send/recv (非传统的层级拓扑分解)
- ZeroCopy判定基于maxSendSize/maxRecvSize(非desc_标志)
- 13个变体是最多的(仅次于AllReduce)

### 4.5 可变长操作 (RSV 10个变体, AGV 9个变体)

相比定长版本的额外复杂度:
- 使用param.VDataDes.counts/displs(每rank数组)替代param.DataDes.count(单值)
- Loop切分需要CalcCurCountsAndCurDispls()(virtual, 子类有不同实现)
- 需要处理某rank count为0的边界: 空指针防御性赋值为CCL buffer地址
- memcpy方向按displs逐rank拷贝，非整块拷贝

### 4.6 Send/Receive (P2P)

直接继承CollNativeExecutorBase，使用COMM_COMBINE通信平面，只需一条P2P链路。
RunTemplate直接从commInfo.links[0]取链路，构造SendReceive executor。

BatchSendRecv的pair-wise排序算法: Send队列递减、Recv队列递增(形成无环图防死锁)。
BatchSendRecv是唯一实现CalcIncreLinkRequest的操作(动态按需建链)。

---

## Part 5: Operator层 (28文件)

### 5.1 自注册工厂

CollAlgOpRegistry(Meyer's单例) + REGISTER_OP宏(18处/15文件):
```cpp
REGISTER_OP(HCCL_CMD_ALLREDUCE, AllReduce, AllReduceOperator);
```

### 5.2 设备类型分发表

基类构造函数中用std::bind建立DevType分发表 (coll_alg_operator.cc:42-80):
```cpp
selectFuncMap_[DevType::DEV_TYPE_910B] = std::bind(&CollAlgOperator::SelectAlgfor910B, ...);
```

每种操作实现SelectAlgfor910A/910B/91093/310P3/Mix，共175处SelectAlgforXXX(14文件)。

### 5.3 算法选择复杂度分级

按SelectAlg复杂度从高到低:
1. AllReduce (721行) -- 支持AIV/确定性/Pipeline/InlineReduce/ARS/AHC/NHR/HD/Ring
2. ReduceScatter (544行)
3. AlltoAll (713行) -- 3个op类型共用
4. AllGather (413行)
5. Broadcast/Reduce/Scatter (100-242行)
6. Send/Receive/BatchWrite -- 固定executor

### 5.4 Level1自动选择

AutoSelectAlgTypeLevel1基于理论代价模型:
- 代价公式: time = latency * steps + dataSize / bandwidth
- 比较Ring vs NHR vs HD vs Pipeline
- 考虑FFTS+ context数量限制

---

## Part 6: Resource Manager (22文件)

### 6.1 CCL Buffer管理

CCLBufferManager内存布局:
```
cclBuffer_ (总)
├── inCCLbuffer_  (前半)
├── outCCLbuffer_ (后半)
└── winExpBuffer_ (1MB MC2扩展)

AIV buffers (独立):
├── aivInputBuffer_  (opbase)   : 4MB flag + 36MB data + 32KB commInfo
├── aivOutputBuffer_ (opbase)
├── aivInputBufferForOffload_  (graph)
└── aivOutputBufferForOffload_ (graph)
```

ShareCCLbufferMgr: per-device单例(static array[MAX])，引用计数管理共享buffer。

### 6.2 Stream管理 -- 双层设计

OpBaseStreamManager: 面向单算子模式，master+slave向量，按需扩展
OffloadStreamManager: 面向graph模式，tag-keyed，获取即消费(GetSlaves erase已返回的stream)

### 6.3 Workspace资源

WorkspaceResource使用Pimpl模式隐藏实现。
WorkSpaceMem: tag-based线性分配器(只移动指针，不回收，整体释放)。

### 6.4 Socket管理

HcclSocketManager(900行): 静态serverSocketMap_跨实例共享 + 引用计数。
白名单机制 + Socket复用 + 优雅停止(原子stopFlag_) + 超时处理(steady_clock)。

### 6.5 Per-Device单例模式(项目惯用法)

多个管理器使用`static T array[MAX_MODULE_DEVICE_NUM]`实现per-device单例:
- StreamActiveManager::GetInstance(deviceLogicId) (stream_active_manager.cc:30)
- ShareCCLbufferMgr::GetInstance(deviceLogicId) (share_ccl_buffer_manager.cc:18)

---

## Part 7: Task层 (5文件) + Legacy层 (3文件)

### 7.1 TaskLoader

Worker线程与条件变量驱动:
```
主线程: NotifyStart() -> WaitDone()
Worker: WaitStart() -> ExecuteTask() -> NotifyDone()
```
双条件变量(startCv_ + doneCv_) + 双bool标志。

ParallelTaskLoader管理TaskLoader向量: 懒扩展 + 全部notify -> 全部wait。

### 7.2 ThreadManage (旧式)

面向Ring算法的子环执行，Prepare方法有25个参数(极长参数列表)。
使用AlgTemplateRegistry获取算法模板，与legacy hcclImpl紧密关联。

### 7.3 Legacy hcclImpl

旧框架核心类(1106行)，职责: 通信域创建(按tag管理三层) + 多流资源管理 + 线程并行建链。
新框架通过friend class CollAlgOperator访问其私有成员(架构债务)。
executor_impl.h已是空壳(原有实现迁移到新coll_executor/架构)。

---

## Part 8: 跨模块编码惯用法

### ID-1: CHK_RET 100%覆盖

CHK_RET在coll_executor/下出现3000+次。所有executor的每个函数调用都用CHK_RET包裹。
辅助宏: CHK_PRT_RET(69次/20文件), CHK_SMART_PTR_NULL(125次/26文件)。

### ID-2: REGISTER_EXEC自注册工厂

每个.cc文件末尾用REGISTER_EXEC宏注册(共计约130处):
```cpp
REGISTER_EXEC("AllReduceRingExecutor", AllReduceRing, CollAllReduceRingExecutor);
```
使用__COUNTER__生成唯一静态变量 + static_assert确保继承关系。

### ID-3: Setter返回HcclResult

所有setter方法返回HcclResult而非void(即使不可能失败)，保持接口一致性:
```cpp
HcclResult SetCCLInBuffer(u64 size) { inCCLbufferSize_ = size; return HCCL_SUCCESS; }
```

### ID-4: 未使用参数(void)标记

基类虚方法的默认实现中: `(void)ranksLinked; (void)needIncreLink;`

### ID-5: 中文星号分隔注释

每个头文件使用中文星号分隔线划分区域:
```cpp
/* *************** 资源计算 *************** */
/* *************** 算法编排 *************** */
```

### ID-6: DMAReduceFlag_优化标志

173处/114文件。OP_BASE + 910_93下启用，跳过User<->CCL拷贝。

### ID-7: new(nothrow) + CHK_SMART_PTR_NULL

resource_manager/中32处/10文件:
```cpp
ringThread_.reset(new (std::nothrow) std::thread(...));
CHK_SMART_PTR_NULL(ringThread_);
```

### ID-8: Per-Device Static Array单例

项目惯用法: `static T array[MAX_MODULE_DEVICE_NUM]` + deviceLogicId索引。

---

## Part 9: 架构模式

### AP-1: 双Registry自注册工厂

两级Registry: CollAlgOpRegistry(按OpType->Operator) + CollAlgExecRegistry(按algName->Executor)。
都是Meyer's单例 + REGISTER宏 + __COUNTER__。
字符串驱动创建: algName如"AllReduceMeshAivExecutor"是两级之间的桥梁。

### AP-2: Template Method(三层嵌套)

L1: CalcResRequest() -- 资源计算骨架
L3: Orchestrate() -- 算法编排骨架
L3: RunLoop() -> RunLoopInner() -- 循环执行骨架

### AP-3: Strategy(设备类型分发)

Operator层: selectFuncMap_[DevType] -> SelectAlgforXXX
Executor层: Level1算法选择if-else链

### AP-4: AllReduce = RS + AG组合

AllReduce的KernelRun组合了RS和AG两个子步骤，通过Registry动态获取子模板。

### AP-5: Orchestrate模板9处统一

所有9种集合操作的Orchestrate遵循相同骨架(时间度量 -> 条件分发 -> RunLoop/KernelRun -> 强制Launch -> 日志)。

---

## Part 10: 业务领域知识

### DK-1: 操作矩阵与分解关系

- AllReduce = ReduceScatter + AllGather
- Broadcast = Scatter + 机间Broadcast + AllGather
- Reduce = ReduceScatter + Gather(只到root)
- AllToAll 独立(非reduce+gather模式)

### DK-2: 三级通信拓扑

Level0(机内): Ring/Mesh/HD, 使用HCCS/PCIe链路
Level1(机间): Ring/NHR/NHR_V1/NB/HD/AHC/AHC_BROKE, 使用RoCE/RDMA
Level2(超节点): Ring/NHR/NB

910B支持两级(L0+L1)，910_93支持三级(L0+L1+L2)。

### DK-3: 数据量分档

- SmallCount: 小数据，可能走ONESHOT变体(一次性完成)
- MidCount: 中等数据
- 普通: 标准路径
- HugeData: 超大数据，禁用子图复用，走Loop切循环

判定公式: `size/deviceNum/INTERNODE_MAX_DATA_RATE > RDMA_SEND_MAX_SIZE || size > SDMA_SEND_MAX_SIZE`

### DK-4: TransportMemType

OP_BASE模式: 使用CCL Buffer(CCL_INPUT/CCL_OUTPUT)中转
图模式/零拷贝: 使用用户Buffer(PARAM_INPUT/PARAM_OUTPUT)直通

### DK-5: InlineReduce vs TBE Reduce

InlineReduce: SDMA在搬运时直接归约(省一次kernel launch)
TBE Reduce: 额外TBE kernel执行归约
PROD运算/INT64类型必须用TBE(硬件限制)。

### DK-6: 确定性计算分级

DETERMINISTIC_DISABLE: 允许非确定性优化(如Mesh Atomic)
DETERMINISTIC: 普通确定性
DETERMINISTIC_STRICT: 严格确定性(影响Slice切分策略)

RS因涉及reduce操作，确定性变体最多(8个); AG只做搬运，不需要。

### DK-7: ARS (Adaptive Ring Size)

动态调整Ring大小: 将原始通信域拆为COMM_LEVEL0_LOGICAL(环内) + COMM_LEVEL1_LOGICAL(环间)。
支持DoubleRing时使用GetARSRingsOrder()生成正反两个环的rank顺序。

### DK-8: BatchSendRecv死锁预防

pair-wise排序算法: Send队列从remoteRank<=localRank开始递减, Recv队列从remoteRank>=localRank开始递增。
确保全局send/recv顺序形成无环图。

---

## Part 11: 硬件抽象

### HW-1: 设备类型能力矩阵

| 设备 | AIV | Pipeline | InlineReduce | 三级拓扑 | ARS | 零拷贝 |
|------|-----|----------|-------------|---------|-----|--------|
| 910A | N | N | N | N | N | N |
| 910B | Y | Y | Y | N(两级) | N | N |
| 910_93 | Y | Y | Y | Y(三级) | Y | Y |
| 310P | N | N | N | N(单级) | N | N |

### HW-2: 通信链路层次

COMM_LEVEL0: 节点内(HCCS/PCIe)
COMM_LEVEL1: 节点间(RoCE/RDMA)
COMM_LEVEL2: 超节点间

特殊平面: COMM_COMBINE(打平), COMM_LEVEL0/1_LOGICAL(ARS动态重组)

### HW-3: CCE对齐约束

ReduceScatter特有: CCE_REDUCE_ALIGN_FACTOR=2, 数据32字节对齐(前后各有一份)
(coll_reduce_scatter_executor.h:17)

---

## Part 12: 反模式清单

### AP-IMP-1: Level1算法选择代码大量重复 [强制关注]

SelectTempAlg()和KernelRun()中的Level1算法选择if-else链在Ring和Mesh中几乎逐行重复。
- coll_all_reduce_ring_executor.cc:161-196
- coll_all_reduce_mesh_executor.cc:140-175
- coll_reduce_scatter_ring_executor.cc:183-244
- coll_reduce_scatter_mesh_executor.cc:148-198
- AG侧也完全对应

SelectTempAlg方法存在但未被一致使用(KernelRun中仍直接写if-else)。

### AP-IMP-2: Executor变体爆炸 (33+30+25+13+10+10+9+5+5+2=142个)

维度组合爆炸导致:
- 大量代码重复(CalcCommInfo/CalcTransportMemType等)
- 新增维度需增加多个Executor
- desc_字段在142个构造函数中分散设置

### AP-IMP-3: Header guard冲突

- coll_all_reduce_mesh_aiv_for_910_93_executor.h使用COLL_ALLREDUCE_MESH_AIV_EXECUTOR_H(与mesh_aiv_executor.h冲突)
- coll_broadcast_comm_executor.h使用COLL_BROADCAST_EXECUTOR_H(与broadcast_executor.h冲突)
- coll_broadcast_mesh_aiv_executor.h使用COLL_BROADCAST_MESH_EXECUTOR_H

### AP-IMP-4: PostSync三方法大量重复

PostSyncWithSubstream(), PostSyncWithoutSubstream(), InplaceOpSync()三个方法结构几乎完全相同，
只是核心操作不同。可提取为Template Method。
(coll_native_executor_base.cc:622-777)

### AP-IMP-5: CalcLoopMaxCount RS/AG接口不一致

RS用成员变量直接访问，AG通过参数传入cclBuffSize。
(coll_reduce_scatter_executor.h:28 vs coll_all_gather_executor.h:24)

### AP-IMP-6: 拼写错误固化

- RunIntraSeverReduceScatter/RunIntraSeverAllGather -- "Server"拼成"Sever"
  (coll_reduce_scatter_ring_for_910_93_executor.h:43, coll_all_gather_ring_for_910_93_executor.h:47)
- coll_all_to_all_v_2level_pipeline_excecutor.h -- "excecutor"
- sendDataSilces_/recvDataSilces_ -- "Slices"拼成"Silces"
  (coll_batch_send_recv_executor.h:50-51)

### AP-IMP-7: REGISTER_EXEC tag命名不统一

混合风格: "SendExecutor" vs "ReduceComm" vs "RunAlltoAllVFullMesh" vs "BroadCastComm"
大小写不一致: "BroadCastComm" vs "BroadcastMesh"

### AP-IMP-8: CalcTotalCount未初始化 + 按值传递

coll_all_gather_v_executor.cc:220-228:
```cpp
HcclResult CalcTotalCount(std::vector<u64> curCounts, u64 &totalCount)
```
- totalCount未初始化为0(依赖调用方)
- curCounts按值传递(应为const引用)

### AP-IMP-9: const_cast绕过接口

PostSyncWithoutSubstream中:
```cpp
const_cast<Stream &>(param.stream)
const_cast<std::vector<Stream> &>(algResResp_->slaveStreams)
```
暗示LaunchTaskExtend接口设计不够严谨(coll_native_executor_base.cc:718-720)。

### AP-IMP-10: AlltoAll的dynamic_cast打破多态

alltoall_operator.cc:454,460,491: 3处dynamic_cast<CollAlltoAllExecutor*>
打破了Operator-Executor之间的多态抽象。

### AP-IMP-11: ThreadManage::Prepare 25个参数

threadManage.cc:187-227: 典型的"parameter object缺失"。

### AP-IMP-12: hcclImpl的friend class架构债务

hccl_impl.h:78-79: `friend class CollAlgOperator`暴露新旧框架过渡期债务。

### AP-IMP-13: OffloadStreamManager获取即消费

offload_stream_manager.cc:79: GetSlaves在返回stream时同时erase，丢失引用即泄漏。

### AP-IMP-14: Receive端DMA消减条件过于复杂

coll_receive_executor.cc:146-148: 4个独立排斥条件组合成一个难以理解的条件表达式。

### AP-IMP-15: namespace结束注释错误

- coll_send_executor.cc:208: `} // namespace hcclss` (多了"ss")
- coll_alg_exec_registry.cc:43: `} // namespace Hccl` (大小写不一致)

---

## Part 13: 特征总结

| 维度 | 特征 |
|------|------|
| 总文件数 | 389 (coll_executor 319 + operator 28 + resource_mgr 22 + task 5 + legacy 3) |
| Executor变体总数 | 142个 (AllReduce 33, RS 30, AG 25, AllToAll 13, Broadcast 10, RSV 10, AGV 9, Reduce 5, Scatter 5, P2P 2) |
| 继承层数 | 4层(ExecutorBase -> NativeBase -> CommExecutor -> OpExecutor -> 具体变体) |
| 核心设计模式 | Template Method(三层嵌套) + 双Registry自注册工厂 + Strategy(设备分发) |
| 错误处理 | CHK_RET 3000+次(100%覆盖) |
| 最大反模式 | Level1算法选择代码重复(6+处)、Executor变体爆炸(142个) |
| 新旧并存 | legacy/ hcclImpl通过friend class被新框架访问 |
| 硬件适配 | 4种设备类型 x SelectAlgforXXX分发 |

---

## Phase 2.3.3: Algorithm层综合分析

基于2.3.1(base/)和2.3.2(impl/)的全量分析 + Grep量化验证的综合提炼。

### 模块全景

Algorithm层是hcomm三层架构的最顶层，负责集合通信算法的选择、编排和执行。

```
总规模: 882个C++文件, 122,497行代码
  base/   479文件, 64,110行 — 算法模板(AIV kernel + 主机侧generic模板 + 拓扑/建链基础设施)
  impl/   389文件, 58,387行 — 算法选择(operator) + 执行编排(coll_executor) + 资源管理

三个并行子体系:
  1. alg_aiv_template/ (设备侧):  AIV核上kernel代码, header-only, AscendC编程模型
  2. alg_template/     (主机侧):  通用算法模板, 通过Transport link发数据, 102个注册模板
  3. communicator/     (基础设施): 拓扑计算、传输需求计算、建链
  + impl/ 将三者编排为完整的集合通信流程
```

---

### 一、跨子模块惯用法 (Idioms & Conventions)

#### ALG-ID-1: CHK_RET 100%覆盖 [项目级强制]

所有函数调用的返回值必须用CHK_RET检查。

量化: 5476次/301文件 (覆盖率100%, 在algorithm层零遗漏)。
辅助宏: CHK_SMART_PTR_NULL 727次/213文件, CHK_PRT_RET见于operator/。

```cpp
// good
CHK_RET(alg->Prepare(inputMem_, outputMem_, count_, ...));
CHK_RET(alg->RunAsync(rank, rankSize, links));

// bad — 裸调用
alg->Prepare(inputMem_, outputMem_, count_, ...);
```

#### ALG-ID-2: 三层Registry自注册工厂 [项目级强制]

Algorithm层使用三个独立的Registry，全部基于Meyer's单例 + 文件级静态注册宏 + static_assert继承检查:

| Registry | 宏 | 量化 | 键类型 | 值类型 |
|----------|-----|------|--------|--------|
| AlgTemplateRegistry | REGISTER_TEMPLATE | 104次/103文件 | TemplateType枚举(0~102) | unique_ptr<AlgTemplateBase> |
| CollAlgExecRegistry | REGISTER_EXEC | 148次/146文件 | string(如"AllReduceRingExecutor") | unique_ptr<CollExecutorBase> |
| CollAlgOpRegistry | REGISTER_OP | 16次/14文件 | HcclCMDType枚举 | unique_ptr<CollAlgOperator> |

三级调用链: REGISTER_OP → SelectAlg(algName字符串) → REGISTER_EXEC(创建executor) → executor内部通过GetAlgTemplate(531次/121文件)动态获取算法模板。

每个.cc文件末尾固定放置一行注册宏，这是项目级编码规范。

#### ALG-ID-3: Template Method骨架 [项目级强制]

Algorithm层使用多层嵌套的Template Method，各层有固定的骨架步骤:

L1 CalcResRequest骨架(资源计算):
  CalcScratchMemSize → CalcStreamNum → CalcNotifyNum → CalcAivBufferRequest → CalcCommInfo → BuildResourceRequest

L3 Orchestrate骨架(9种操作统一):
  startut → tag/algRes赋值 → 图模式/单卡/多卡分发 → RunLoop/KernelRun → 强制LaunchTaskExtend → 结束日志

L3 KernelRun骨架(AllReduce典型):
  Level0 ReduceScatter → Level1 AllReduce → Level0 AllGather

强制LaunchTaskExtend: 出现在所有9个操作基类的Orchestrate结尾(35文件/57次)。
注释明确标注"不要删除, 否则导致aicpu cache功能问题" — 项目级强制规范。

#### ALG-ID-4: 两段式构造 (Prepare模式) [项目级强制]

所有算法模板和executor都使用两段式构造: 工厂创建 → Prepare()初始化。

```cpp
// 1. 工厂创建(空壳)
auto alg = AlgTemplateRegistry::Instance().GetAlgTemplate(templateType, dispatcher_);
// 2. Prepare初始化(实际参数绑定)
alg->Prepare(reduceAttr_);
alg->Prepare(inputMem_, outputMem_, count_, dataType_, stream_, ...);
```

基类声明~30个virtual Prepare重载(按参数数量1~19分组)。
子类使用`using AlgTemplateBase::Prepare;`引入基类重载避免C++ name hiding(23个文件)。
新代码用PrepareData结构体替代长参数列表。

#### ALG-ID-5: desc_描述符声明能力 [算法层强制]

每个Executor在构造函数中通过desc_结构体声明自己的能力:
- isAivMode / isAivCrossNode / isZeroCopy / deterministic
- level1SupportedAlgos / level2SupportedAlgos (白名单vector)

运行时SetAlgType()检查requested type是否在supported列表中，不在则回退。
这是一种"声明式配置"模式，将算法能力与选择逻辑分离。

#### ALG-ID-6: DMAReduceFlag_优化标志 [算法层强制]

173次/114文件。标记是否跳过User<->CCL buffer拷贝(OP_BASE+910_93下启用)。
贯穿所有executor的Orchestrate/RunLoop/KernelRun，是算法层最广泛的优化开关。

#### ALG-ID-7: Setter返回HcclResult [算法层推荐]

即使setter不可能失败，也统一返回HcclResult:
```cpp
HcclResult SetCCLInBuffer(u64 size) { inCCLbufferSize_ = size; return HCCL_SUCCESS; }
```
保持接口一致性，使CHK_RET链式调用不中断。

#### ALG-ID-8: (void)标记未使用参数 [算法层推荐]

基类虚方法的默认实现中统一使用`(void)param;`标记未使用参数，抑制编译器警告:
```cpp
virtual HcclResult CalcIncreLinkRequest(...) {
    (void)ranksLinked; (void)needIncreLink;
    return HCCL_SUCCESS;
}
```

#### ALG-ID-9: 中文星号分隔注释 [算法层推荐]

头文件中使用中文星号分隔线划分区域:
```cpp
/* *************** 资源计算 *************** */
/* *************** 算法编排 *************** */
```

#### ALG-ID-10: rankSize=1快速路径 [算法层强制]

所有算法模板和executor统一处理单卡场景: input!=output则D2DMemcpy, 否则直接返回HCCL_SUCCESS。
这是项目级规范，在所有102个算法模板中一致实施。

#### ALG-ID-11: AIV设备侧错误处理 [AIV模块强制]

AIV kernel代码使用不同于主机侧的错误处理:
- AIV_ERROR → PrintfImpl + __builtin_trap() (不可恢复, 直接停机)
- 无CHK_RET(设备侧没有错误传播链)
- HeadCounter/TailCounter用于维测(而非日志宏)

---

### 二、架构模式 (Architecture Patterns)

#### ALG-AP-1: 四层继承 + 双Registry工厂

```
CollExecutorBase (L0: 2个纯虚方法)
  +-- CollNativeExecutorBase (L1: Template Method骨架)
        +-- CollCommExecutor (L2: Multi-Ring/Multi-Stream编排)
        |     +-- CollAllReduceExecutor (L3: 操作特化)
        |           +-- 33个具体Executor (L4: 硬件x算法变体)
        +-- CollAlltoAllExecutor (L1直接: 跳过L2)
        +-- CollSend/ReceiveExecutor (L1直接: P2P)
  +-- CollBatchWriteExecutor (L0直接: 最轻量)
```

AllToAll和Send/Receive跳过L2(CollCommExecutor)的原因:
- AllToAll: O(N^2)数据流模式，不适用reduce+gather分解
- Send/Receive: P2P操作，不需要多级通信域抽象

#### ALG-AP-2: AllReduce = RS + AG 组合模式

AllReduce统一分解为ReduceScatter + AllGather两阶段(所有变体一致):
```
AllReduceRing = ReduceScatterRing + AllGatherRing
AllReduceNHR = ReduceScatterNHR + AllGatherNHR
AllReduceNB  = ReduceScatterNB  + AllGatherNB
例外: AllReduceReduceBcast = ReduceRing + BroadcastRing
```

组合通过GetAlgTemplate(531次/121文件)动态获取子模板实现，非继承。
这是算法层最重要的跨模板组合模式。

#### ALG-AP-3: RS与AG完美反向对称

ReduceScatter和AllGather在算法结构上完全镜像:
- RS先内后外(Level0→Level1→Level2)，AG先外后内(Level2→Level1→Level0)
- Executor变体一一对应(Ring/Mesh/91093各有RS版和AG版)
- RS多8个确定性变体(因reduce浮点非确定性)，AG只做搬运不需要

#### ALG-AP-4: Strategy模式 — 两级设备分发

Operator层: selectFuncMap_[DevType] → SelectAlgforXXX (205次/18文件)
Executor层: Level1算法选择if-else链(KernelRun内)

SelectAlg复杂度分级:
- AllReduce 721行 > AlltoAll 713行 > ReduceScatter 544行 > AllGather 413行
- Send/Receive/BatchWrite: 固定executor，无选择逻辑

#### ALG-AP-5: CommBase与AlgTemplateBase的组合关系

两者不是继承关系:
- CommBase = "链路管理者" — 创建和持有rank间的Transport links
- AlgTemplateBase = "算法执行者" — 接收links作为RunAsync参数
- 桥接: CommBase::RunTemplateAlg(unique_ptr<AlgTemplateBase>&)

#### ALG-AP-6: AIV双基类体系

设备侧AIV有两个独立基类(非继承):
- AivCommBase: 节点内(910B/91093, 最多16卡)
- AivCrossNode91093Base: 跨节点(91093, 最多768卡)

kernel入口通过宏生成 + 多层if-else分派(devType×isOpBase×deterministic×len×aivRdmaStep)。

#### ALG-AP-7: 三层算法分工协作

```
selector(operator/) — 决定用哪个算法(algName字符串)
     ↓
executor(coll_executor/) — 编排多步执行(slice/loop/barrier)
     ↓
template(alg_template/) — 执行单步算法(Ring/Mesh/NHR的具体step)
     ↓ (launch)
AIV kernel(alg_aiv_template/) — 设备侧执行数据搬运和reduce
```

接触点:
- executor通过GetAlgTemplate获取template实例
- template通过CommBase的transportInfo_使用communicator建的链路
- template编排的kernel launch最终执行AIV template代码
- communicator的TopoInfoExtractor为executor提供rank映射

---

### 三、业务领域知识 (Domain Knowledge)

#### ALG-DK-1: 集合操作分解关系

| 操作 | 分解 | 变体数 |
|------|------|--------|
| AllReduce | Level0 RS + Level1 AR + Level0 AG | 33 |
| ReduceScatter | 独立实现(+ 确定性变体) | 30 |
| AllGather | RS的完美反向 | 25 |
| AllToAll | 机内fullmesh + 机间send/recv (独立模式) | 13 |
| Broadcast | Scatter + 机间Broadcast + AllGather | 10 |
| ReduceScatterV | RS + 可变长额外复杂度 | 10 |
| AllGatherV | AG + 可变长额外复杂度 | 9 |
| Reduce | RS + Gather到root | 5 |
| Scatter | Broadcast的逆 | 5 |
| Send/Receive | P2P, 单链路 | 2 |
| BatchSendRecv | pair-wise防死锁排序 | 2 |

#### ALG-DK-2: 三级通信拓扑与算法映射

| 层级 | 物理含义 | 链路类型 | 可选算法 |
|------|---------|---------|---------|
| Level0 (机内) | 同节点各卡 | HCCS/PCIe | Ring/Mesh/HD |
| Level1 (机间) | 不同节点 | RoCE/RDMA | Ring/NHR/NHR_V1/NB/HD/AHC/AHC_BROKE |
| Level2 (超节点) | 不同超节点 | RoCE | Ring/NHR/NB |

910B支持两级(L0+L1)，910_93支持三级(L0+L1+L2)。

Level1自动选择: AutoSelectAlgTypeLevel1基于理论代价模型:
  time = latency * steps + dataSize / bandwidth

#### ALG-DK-3: 数据量分档与阈值

算法层根据数据量选择不同实现路径:

AIV层阈值(aiv_communication_base.h):
- AIV_ALL_REDUCE_SMALL_SIZE = 64KB
- AIV_ALL_REDUCE_BIG_SIZE = 16MB
- UB_MAX_DATA_SIZE = 190KB (UB缓冲区)
- AIV_ALL_REDUCE_DETER_SMALL/MID_SIZE = 1MB/8MB

Generic层阈值(alg_template_base_pub.h):
- HCCL_CHUNK_SIZE = 1GB
- HCCL_SPLIT_SIZE_INTER_SERVER = 8MB (跨机切分)
- HCCL_MIN_SLICE_ALIGN = 128 / 16384 (通用/910B对齐)
- NB_ALLREDUCE_SMALL_SIZE = 128KB

Executor层判定(HugeData):
  size/deviceNum/INTERNODE_MAX_DATA_RATE > RDMA_SEND_MAX_SIZE || size > SDMA_SEND_MAX_SIZE

#### ALG-DK-4: InlineReduce vs TBE Reduce

```
支持InlineReduce的条件: dataType + reductionOp + 链路能力(DevCapability)
  支持 → SDMA搬运时直接归约(省一次kernel launch)
  不支持 → TBE kernel单独执行归约

强制TBE: PROD运算 或 INT64类型(硬件限制)
```

reduceAttr位掩码: RDMA_REDUCE_BITMASK / INLINE_REDUCE_BITMASK 控制路径选择。
Sender/Reducer组件自动根据link能力选择4条reduce路径之一。

#### ALG-DK-5: 确定性计算三级

| 级别 | 含义 | 影响 |
|------|------|------|
| DETERMINISTIC_DISABLE | 允许非确定性优化 | Mesh Atomic等 |
| DETERMINISTIC | 普通确定性 | 固定数据路径 |
| DETERMINISTIC_STRICT | 严格确定性 | 影响Slice切分策略 |

RS因涉及reduce操作，确定性变体最多(8个); AG只做搬运，不需要。

#### ALG-DK-6: Loop机制与CCL Buffer

当数据量超过CCL Buffer时，通过RunLoop切循环:
```
while (remainCount > 0):
    loopCount = min(remainCount, CalcLoopMaxCount())
    拷入CCL Buffer → KernelRun → 拷出CCL Buffer
```

CCL Buffer内存布局:
```
cclBuffer_ (总)
├── inCCLbuffer_  (前半)
├── outCCLbuffer_ (后半)
└── winExpBuffer_ (1MB MC2扩展)

AIV独立buffer:
├── aivInputBuffer_   4MB flag + 36MB data + 32KB commInfo
└── aivOutputBuffer_
```

零拷贝场景(DMAReduceFlag_=true)跳过拷入/拷出，直接操作用户buffer。

#### ALG-DK-7: BatchSendRecv死锁预防

pair-wise排序算法:
- Send队列: 从remoteRank<=localRank开始递减
- Recv队列: 从remoteRank>=localRank开始递增
- 确保全局send/recv顺序形成无环图

BatchSendRecv是唯一实现CalcIncreLinkRequest的操作(动态按需建链)。

#### ALG-DK-8: GM flag同步协议(AIV)

设备侧AIV使用GM(Global Memory) flag实现核间同步:
```
1v1:  Record/Wait (点对点)
1vN:  Record1vN/Wait1vN (广播)
Nv1:  RecordNv1/WaitNv1 (汇聚)
```

CpGM2GM必须经UB中转: 190KB单buffer / 95KB double buffer。

---

### 四、硬件抽象 (Hardware Abstractions)

#### ALG-HW-1: 设备能力矩阵

| 设备 | AIV | Pipeline | InlineReduce | 拓扑层级 | ARS | ZeroCopy |
|------|-----|----------|-------------|---------|-----|----------|
| 910A | N | N | N | 2级 | N | N |
| 910B | Y | Y | Y | 2级 | N | N |
| 910_93 | Y | Y | Y | 3级 | Y | Y |
| 310P | N | N | N | 1级 | N | N |

每种设备通过SelectAlgforXXX(205次/18文件)选择对应的Executor变体。

#### ALG-HW-2: AIV硬件模型

- 910B: 16卡/节点, MAX_RANK_SIZE=16, 节点内HCCS
- 91093(A3): 768卡/超节点, MAX_RANK_SIZE_A3=768, 48核多rank分工
- UB(Unified Buffer): 190KB单buffer, 用于GM2GM中转
- GM: Global Memory, 用于核间flag同步和数据交换

AIV kernel使用AscendC编程模型: DataCopy/PipeBarrier/TQueBind/TBuf等API。

#### ALG-HW-3: 对齐约束

- 通用slice对齐: 128字节
- 910B/910_93 slice对齐: 16384字节
- CCE reduce对齐: 32字节(CCE_REDUCE_ALIGN_FACTOR=2, 前后各一份)
- CCL Buffer 4K页对齐
- SDMA单次传输上限: 4GB (HCCL_SDMA_MAX_COUNT_4GB)

#### ALG-HW-4: 拓扑知识分布

拓扑知识分散在三处(应统一):
1. search_path.h: 硬编码910_93的16-NIC可达矩阵
2. topo_info_extractor.cc: ranksOneNode_数组编码各TopoType的每节点设备数
3. AIV base: MAX_RANK_SIZE常量编码设备拓扑限制

---

### 五、反模式清单 (Anti-Patterns)

#### AP-ALG-1: Executor变体爆炸 [严重/架构级]

142个具体Executor变体(33+30+25+13+10+10+9+5+5+2)，由9个正交维度组合。
后果: 大量代码重复(CalcCommInfo/CalcTransportMemType等几乎逐变体copy-paste)，
新增任何一个维度需同步创建多个Executor。
根本原因: 缺少组合模式(Strategy/Policy)将各维度解耦。
文件: impl/coll_executor/ 319个文件

#### AP-ALG-2: Level1算法选择代码大量重复 [严重/可修复]

SelectTempAlg()和KernelRun()中的Level1算法选择if-else链在Ring和Mesh中几乎逐行重复(6+处):
- coll_all_reduce_ring_executor.cc:161-196
- coll_all_reduce_mesh_executor.cc:140-175
- coll_reduce_scatter_ring_executor.cc:183-244
- coll_reduce_scatter_mesh_executor.cc:148-198
- AG侧完全对应

SelectTempAlg方法已存在但未被一致使用(KernelRun中仍直接写if-else)。

#### AP-ALG-3: Prepare重载膨胀 [中等/设计债务]

基类声明~30个virtual Prepare重载(1~19个参数)，按参数数量分组注释是应对策略。
新代码已开始用PrepareData结构体，但旧重载未清理。
文件: alg_template_base_pub.h

#### AP-ALG-4: 默认空实现掩盖遗漏 [中等/风险]

所有虚方法返回HCCL_SUCCESS而非pure virtual。
子类忘记override RunAsync不会报编译错误，只会在运行时无操作返回SUCCESS。
文件: alg_template_base_pub.h

#### AP-ALG-5: Header guard冲突 [低/可修复]

- coll_all_reduce_mesh_aiv_for_910_93_executor.h使用与mesh_aiv_executor.h相同的guard
- coll_broadcast_comm_executor.h使用与broadcast_executor.h相同的guard
可能导致编译时头文件被跳过。

#### AP-ALG-6: 拼写错误固化 [低/注意]

- RunIntraSeverReduceScatter/AllGather — "Server"拼成"Sever"
- coll_all_to_all_v_2level_pipeline_excecutor.h — "excecutor"
- sendDataSilces_/recvDataSilces_ — "Slices"拼成"Silces"

这些错误已被API/头文件引用固化，修改需全局替换。

#### AP-ALG-7: CalcLoopMaxCount接口不一致 [低/设计]

RS用成员变量直接访问: `CalcLoopMaxCount(u32 unitSize)`
AG用参数传入: `CalcLoopMaxCount(u64 cclBuffSize, u32 unitSize)`
同一抽象层的相同概念，接口应保持一致。

#### AP-ALG-8: AlltoAll的dynamic_cast打破多态 [中等/设计]

alltoall_operator.cc中3处dynamic_cast<CollAlltoAllExecutor*>打破了Operator-Executor之间的多态抽象。
正确做法: 通过虚方法或desc_声明需要的额外接口。

#### AP-ALG-9: PostSync三方法结构重复 [中等/可修复]

PostSyncWithSubstream/PostSyncWithoutSubstream/InplaceOpSync三个方法结构几乎完全相同，
只是核心操作不同。应提取为Template Method。
文件: coll_native_executor_base.cc:622-777

#### AP-ALG-10: const_cast绕过接口 [中等/设计]

PostSyncWithoutSubstream中:
```cpp
const_cast<Stream &>(param.stream)
const_cast<std::vector<Stream> &>(algResResp_->slaveStreams)
```
暗示LaunchTaskExtend接口设计不够严谨(接受了non-const但调用方只有const)。

#### AP-ALG-11: ThreadManage::Prepare 25个参数 [中等/可修复]

典型的"parameter object缺失"。应使用结构体聚合参数。
文件: threadManage.cc:187-227

#### AP-ALG-12: CommBase 25+参数构造函数 [中等/可修复]

子类(CommRing/CommMesh等)构造函数也有20+参数。应使用Builder或参数结构体。
文件: comm_base_pub.h

#### AP-ALG-13: hcclImpl的friend class架构债务 [低/遗留]

hccl_impl.h: `friend class CollAlgOperator` 暴露新旧框架过渡期债务。
新框架通过friend class访问旧框架私有成员，而非提取公共接口。

#### AP-ALG-14: OffloadStreamManager获取即消费 [中等/风险]

GetSlaves在返回stream时同时erase。调用方如果不持有返回值就导致stream泄漏。
文件: offload_stream_manager.cc:79

#### AP-ALG-15: REGISTER_EXEC tag命名不统一 [低/规范]

混合风格: "SendExecutor" vs "ReduceComm" vs "RunAlltoAllVFullMesh" vs "BroadCastComm"
大小写不一致: "BroadCastComm" vs "BroadcastMesh"

---

### 六、特征总结

| 维度 | 量化数据 |
|------|---------|
| 总规模 | 882文件, 122,497行 (base/ 479文件 64,110行 + impl/ 389文件 58,387行) |
| 注册模板 | REGISTER_TEMPLATE 104次(102个算法模板), REGISTER_EXEC 148次, REGISTER_OP 16次 |
| 错误检查 | CHK_RET 5476次/301文件, CHK_SMART_PTR_NULL 727次/213文件 |
| Executor变体 | 142个 (9种操作 x 多种硬件/算法/模式组合) |
| 跨模板组合 | GetAlgTemplate 531次/121文件 |
| 设备分发 | SelectAlgfor 205次/18文件 |
| 优化标志 | DMAReduceFlag_ 173次/114文件 |
| cast使用 | reinterpret_cast 62次/17文件 (远低于Framework层) |
| 代码卫生 | TODO/FIXME/HACK: 0 (整个algorithm/模块零技术债注释) |
| 核心设计模式 | Template Method(三层嵌套) + 三级Registry + Strategy(设备分发) + 组合(RS+AG) |
| 最大设计挑战 | Executor变体爆炸(142个) + Level1选择代码重复(6+处) |
| 新旧并存 | legacy/ hcclImpl通过friend class被新框架访问 |

### 七、与Platform/Framework层对比

| 维度 | Platform层 | Framework层 | Algorithm层 |
|------|-----------|-------------|-------------|
| 主要语言 | C+C++ | C++ | C++(主机) + AscendC(设备) |
| 核心模式 | C ABI+Strategy+RPC | Facade+分派+State | Template Method+Registry+Strategy |
| 错误处理 | CHK_RET+C错误码 | CHK_RET+RPT+EXCEPTION | CHK_RET(主机) / trap(AIV) |
| 生命周期 | owner_+move+RAII | unique_ptr+显式析构序 | 两段式构造+Registry管理 |
| 代码卫生 | 少量TODO | 13个TODO(next/) | 零TODO |
| reinterpret_cast密度 | 中 | 高(1376次) | 低(62次) |
| 反模式严重度 | 全局可变状态/busy-wait | God Class/switch缺break | 变体爆炸/代码重复 |
