# Ascend C HCCL高阶API

> 来源：CANN商用版8.0.RC3 hiascend.com

## Hccl核心接口

### Hccl高阶API使用说明

#### 概述

Ascend C提供了一套Hccl高阶API，支持在AI Core侧灵活编排集合通信任务。Hccl作为集合通信任务客户端，主要提供集合通信原语接口。

#### 核心概念

通信任务类型：
- 先通信后计算（如AllGather + MatMul）
- 先计算后通信（如MatMul + AllReduce）

关键接口：
- Prepare接口：AllReduce、AllGather、ReduceScatter、AlltoAll
- 控制接口：Commit、Wait、Finalize

#### 使用步骤

##### 步骤1：初始化

```cpp
Hccl<HCCL_SERVER_TYPE_AICPU> hccl;
GM_ADDR context = AscendC::GetHcclContext<HCCL_GROUP_ID_0>();
hccl.Init(context);
```

HcclCombinOpParam结构体包含workspace信息、rank详情和通信窗口地址等信息。

##### 步骤2：下发通信任务

调用对应的Prepare接口（如ReduceScatter），获取handleId用于任务标识。

##### 步骤3：提交执行

```cpp
hccl.Commit(handleId);
```

##### 步骤4：等待完成

```cpp
hccl.Wait(handleId);
```

##### 步骤5：结束

```cpp
hccl.Finalize();
```

#### 数据类型和操作

HcclDataType: INT8, INT16, INT32, FP16, FP32, INT64, UINT64, UINT8, UINT16, UINT32, FP64, BFP16

HcclReduceOp: HCCL_REDUCE_SUM, HCCL_REDUCE_PROD, HCCL_REDUCE_MAX, HCCL_REDUCE_MIN

#### 约束说明

- 数据分块不超过16块
- 每个通信域内Prepare接口调用总次数不超过32次
- 需要指定接口代码运行在AI Cube核或AI Vector核上

---

### Hccl模板参数

#### 概述

本文档描述创建Hccl对象时所需的模板参数。

#### 函数原型

```cpp
template <HcclServerType serverType = HcclServerType::HCCL_SERVER_TYPE_AICPU>
class Hccl;
```

#### 参数说明

HcclServerType指定支持的服务端类型，当前仅支持AICPU服务端类型。

```cpp
enum HcclServerType {
    HCCL_SERVER_TYPE_AICPU = 0,  // 当前仅支持AICPU服务端
    HCCL_SERVER_TYPE_END         // 保留参数，不支持
}
```

#### 返回值

无

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 使用示例

文档提供了一个完整的"Add计算 + AllReduce通信 + Mul计算"任务编排示例。关键操作包括：

1. 初始化Hccl客户端：创建并初始化Hccl对象，获取通信上下文
2. 提交AllReduce任务：预通知服务端组装通信任务，获取handleId标识
3. 执行Add计算：完成计算并同步
4. 提交任务：通知通信侧异步执行已调度的任务
5. 等待完成：阻塞等待特定通信任务完成后再执行依赖计算
6. 执行Mul计算：执行逐元素乘法操作
7. 结束：通知服务端所有通信任务已完成

该示例展示了通过任务流水线实现计算与通信重叠以优化性能。

---

### Init

#### 功能说明

Init函数用于初始化Hccl客户端。默认在所有核上执行，可通过`GetBlockIdx()`限制在特定核上执行。

#### 函数原型

```cpp
__aicore__ inline void Init(GM_ADDR context)
```

#### 参数说明

| 参数    | 输入/输出 | 说明                                                                      |
| ------- | --------- | ------------------------------------------------------------------------- |
| context | 输入      | 通信上下文，对应`HcclCombineOpParam`结构体，包含rankDim、rankID等相关信息 |

#### 返回值

无

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 注意事项

该接口推荐在AI Cube核或者AI Vector核两者之一上调用。

#### 使用示例

示例1 - 默认在所有核上初始化：

```cpp
Hccl hccl;
GM_ADDR contextGM1 = GetHcclContext<HCCL_GROUP_ID_0>();
hccl.Init(contextGM1);
```

示例2 - 仅在Vector核上初始化：

```cpp
if ASCEND_IS_AIV {
  Hccl hccl;
  GM_ADDR contextGM1 = GetHcclContext<HCCL_GROUP_ID_0>();
  hccl.Init(contextGM1);
}
```

示例3 - 仅在特定Vector核(block 0)上初始化：

```cpp
if ASCEND_IS_AIV {
  if (GetBlockIdx() == 0) {
    Hccl hccl;
    GM_ADDR contextGM1 = GetHcclContext<HCCL_GROUP_ID_0>();
    hccl.Init(contextGM1);
  }
}
```

---

### AllReduce

#### 功能说明

AllReduce集合通信算子提交任务并返回handle ID。它对通信域内所有节点的tensor执行reduce操作，然后将结果广播到所有节点的输出缓冲区。

#### 函数原型

```cpp
template <bool commit = false>
__aicore__ inline HcclHandle AllReduce(GM_ADDR sendBuf, GM_ADDR recvBuf,
    uint64_t count, HcclDataType dataType, HcclReduceOp op, uint8_t repeat = 1)
```

#### 模板参数

| 参数   | 类型 | 说明                                                       |
| ------ | ---- | ---------------------------------------------------------- |
| commit | bool | 为true时，在Prepare阶段通知服务端执行任务；为false时不通知 |

#### 接口参数

| 参数     | 输入/输出 | 说明                                                                                                         |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------ |
| sendBuf  | 输入      | 源数据缓冲区地址                                                                                             |
| recvBuf  | 输出      | 目标缓冲区，集合通信结果写入此处                                                                             |
| count    | 输入      | AllReduce操作的数据元素个数                                                                                  |
| dataType | 输入      | 支持类型：float32, float16, int8_t, int16_t, int32_t, bfloat16_t                                             |
| op       | 输入      | Reduce操作：HCCL_REDUCE_SUM, HCCL_REDUCE_MAX, HCCL_REDUCE_MIN                                                |
| repeat   | 输入      | AllReduce任务数量（>=1，默认1）。当repeat>1时，地址自动计算：sendBuf[i] = sendBuf + count*sizeof(datatype)*i |

#### 返回值

成功返回handle ID(>=0)；失败返回-1。

#### 支持型号

Atlas A2训练系列产品和Atlas 800I A2推理产品

#### 约束说明

- 需要先调用Init接口
- 仅可在AI Cube核或AI Vector核上调用
- 仅在core 0上执行
- 每个通信域内Prepare调用总次数不超过32次

---

### AllGather

#### 功能说明

AllGather接口提交集合通信任务并返回handle ID。该算子将通信域内所有节点的输入数据按rank ID排序后拼接，并将结果分发到所有节点。

#### 函数原型

```cpp
template <bool commit = false>
__aicore__ inline HcclHandle AllGather(GM_ADDR sendBuf, GM_ADDR recvBuf,
    uint64_t sendCount, HcclDataType dataType, uint64_t strideCount,
    uint8_t repeat = 1)
```

#### 模板参数

| 参数   | 输入/输出 | 说明                                                               |
| ------ | --------- | ------------------------------------------------------------------ |
| commit | 输入      | 布尔标志。为true时Prepare接口通知服务端执行任务；为false时延迟执行 |

#### 接口参数

| 参数        | 输入/输出 | 说明                                                                                                                                                                                |
| ----------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf     | 输入      | 源数据缓冲区地址                                                                                                                                                                    |
| recvBuf     | 输出      | 存储AllGather结果的目标缓冲区                                                                                                                                                       |
| sendCount   | 输入      | sendBuf中参与操作的数据元素个数；recvBuf包含sendCount x rank数量个元素                                                                                                              |
| dataType    | 输入      | 操作的数据类型；支持HcclDataType枚举中的所有类型                                                                                                                                    |
| strideCount | 输入      | 控制recvBuf中数据块间距。为0时块连续排列；大于0时块之间间隔strideCount个数据元素。rank[i]的偏移为i x strideCount                                                                    |
| repeat      | 输入      | AllGather任务数量（>=1，默认1）。当repeat>1时，缓冲区地址自动递增：sendBuf[i] = sendBuf + sendCount x sizeof(datatype) x i；recvBuf[i] = recvBuf + sendCount x sizeof(datatype) x i |

#### 返回值

返回任务handle ID（成功>=0，失败返回-1）。

#### 支持型号

Atlas A2训练系列产品和Atlas 800I A2推理产品。

#### 约束说明

- 需要先调用Init接口
- 仅可在AI Cube核或AI Vector核上调用
- 仅在core 0上执行
- 每个通信域内Prepare调用总次数不超过32次

#### 使用示例

##### 非切分场景

四张卡各有300个float16元素执行AllGather，将拼接结果输出到每张卡的内存：

```cpp
extern "C" __global__ __aicore__ void all_gather_custom(GM_ADDR xGM, GM_ADDR yGM) {
    auto sendBuf = xGM;
    auto recvBuf = yGM;
    uint64_t sendCount = 300;
    uint64_t strideCount = 0;
    GM_ADDR contextGM = GetHcclContext<HCCL_GROUP_ID_0>();
    Hccl hccl;
    if (g_coreType == AIV) {
        hccl.Init(contextGM);
        HcclHandle handleId1 = hccl.AllGather<true>(sendBuf, recvBuf,
            sendCount, HcclDataType::HCCL_DATA_TYPE_FP16, strideCount);
        hccl.Wait(handleId1);
        SyncAll<true>();
        hccl.Finalize();
    }
}
```

##### 多轮切分场景

300个元素切分为2块128元素和1块44元素，分3轮通信处理：

```cpp
extern "C" __global__ __aicore__ void all_gather_custom(GM_ADDR xGM, GM_ADDR yGM) {
    constexpr uint32_t tileNum = 2U;
    constexpr uint64_t tileLen = 128U;
    constexpr uint32_t tailNum = 1U;
    constexpr uint64_t tailLen = 44U;
    auto sendBuf = xGM;
    auto recvBuf = yGM;
    GM_ADDR contextGM = GetHcclContext<HCCL_GROUP_ID_0>();
    Hccl hccl;
    if (g_coreType == AIV) {
        hccl.Init(contextGM);
        uint64_t strideCount = tileLen * tileNum + tailLen * tailNum;
        HcclHandle handleId1 = hccl.AllGather<true>(sendBuf, recvBuf,
            tileLen, HcclDataType::HCCL_DATA_TYPE_FP16, strideCount, tileNum);
        constexpr uint32_t kSizeOfFloat16 = 2U;
        sendBuf += tileLen * tileNum * kSizeOfFloat16;
        recvBuf += tileLen * tileNum * kSizeOfFloat16;
        HcclHandle handleId2 = hccl.AllGather<true>(sendBuf, recvBuf,
            tailLen, HcclDataType::HCCL_DATA_TYPE_FP16, strideCount, tailNum);
        hccl.Wait(handleId1);
        hccl.Wait(handleId2);
        SyncAll<true>();
        hccl.Finalize();
    }
}
```

---

### ReduceScatter

#### 功能说明

ReduceScatter是集合通信算子，对所有rank的输入数据进行求和（或其他规约操作），然后根据rank编号将结果均匀分散到各rank。每个进程从其他进程接收并规约1/ranksize的数据。

#### 函数原型

```cpp
template <bool commit = false>
__aicore__ inline HcclHandle ReduceScatter(GM_ADDR sendBuf, GM_ADDR recvBuf,
    uint64_t recvCount, HcclDataType dataType, HcclReduceOp op,
    uint64_t strideCount, uint8_t repeat = 1)
```

#### 模板参数

| 参数   | 类型 | 说明                                               |
| ------ | ---- | -------------------------------------------------- |
| commit | bool | true：在Prepare阶段通知服务端执行；false：延迟执行 |

#### 接口参数

| 参数        | 输入/输出 | 说明                                                                      |
| ----------- | --------- | ------------------------------------------------------------------------- |
| sendBuf     | 输入      | 源数据缓冲区地址                                                          |
| recvBuf     | 输出      | 结果目标缓冲区                                                            |
| recvCount   | 输入      | 每个rank在recvBuf中的数据元素个数；sendBuf包含recvCount x rank_size个元素 |
| dataType    | 输入      | 数据类型(FP32, FP16, INT8, INT16, INT32, BFP16)                           |
| op          | 输入      | 规约操作(HCCL_REDUCE_SUM, HCCL_REDUCE_MAX, HCCL_REDUCE_MIN)               |
| strideCount | 输入      | 从一张卡分散到多张卡时相邻数据块之间的地址偏移；0表示连续寻址             |
| repeat      | 输入      | ReduceScatter任务数量（>=1，默认1）；repeat>1时缓冲区自动计算             |

#### 返回值

成功返回handleId(>=0)；失败返回-1。

#### 约束说明

- 必须先调用Init接口
- 仅可在AI Cube核或AI Vector核上调用
- 仅在core 0上执行
- 每个通信域内Prepare调用总次数不超过32次

#### 关键要求

- 支持Atlas A2训练系列产品/Atlas 800I A2推理产品
- repeat>1时需配合strideCount进行地址规划
- 缓冲区计算：sendBuf[i] = sendBuf + recvCount x sizeof(datatype) x i

---

### AlltoAll

#### 功能说明

AlltoAll接口是集合通信任务提交API，返回任务handle标识。它使通信域内每张卡能够向所有卡发送等量数据，同时从所有卡接收等量数据。

#### 函数原型

```cpp
template <bool commit = false>
__aicore__ inline HcclHandle AlltoAll(GM_ADDR sendBuf, GM_ADDR recvBuf,
                                       uint64_t dataCount, HcclDataType dataType,
                                       uint64_t strideCount = 0, uint8_t repeat = 1);
```

#### 模板参数

| 参数   | 类型 | 说明                                                                   |
| ------ | ---- | ---------------------------------------------------------------------- |
| commit | bool | 控制与服务端的同步：true在Prepare调用期间通知服务端执行；false延迟执行 |

#### 接口参数

| 参数        | 输入/输出 | 说明                                                                                                                           |
| ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| sendBuf     | 输入      | 源数据缓冲区地址                                                                                                               |
| recvBuf     | 输出      | 集合通信结果的目标缓冲区                                                                                                       |
| dataCount   | 输入      | 每张卡发送/接收的数据量（单位：sizeof(dataType)）                                                                              |
| dataType    | 输入      | AlltoAll操作的数据类型；支持所有HcclDataType变体                                                                               |
| strideCount | 输入      | 多轮场景中数据块之间的内存步长。默认0表示连续块。大于0时相邻块偏移strideCount个单位                                            |
| repeat      | 输入      | 单次提交的AlltoAll任务数量（>=1，默认1）。大于1时缓冲区地址按轮次更新：sendBuf[i] = sendBuf + dataCount * sizeof(datatype) * i |

#### 返回值

成功返回任务handle标识(handleId >= 0)，失败返回-1。

#### 支持型号

Atlas A2训练系列产品/Atlas 800I A2推理产品

#### 约束说明

- 需要先调用Init接口
- 仅可在AI Cube核或AI Vector核上执行
- 仅core 0可提交通信任务
- 每个通信域内Prepare接口调用总次数不超过32次

---

### Commit

#### 功能说明

Commit接口通知服务端正式执行handleId对应的任务。每调用一次本接口，则通知服务端可以正式执行handleId对应的任务一次，调用次数应与Prepare接口的repeat参数对齐。

#### 函数原型

```cpp
__aicore__ inline void Commit(HcclHandle handleId)
```

#### 参数说明

| 参数     | 输入/输出 | 说明                                            |
| -------- | --------- | ----------------------------------------------- |
| handleId | 输入      | 通信任务标识ID；必须使用Prepare原语接口的返回值 |

类型定义：

```cpp
using HcclHandle = int8_t;
```

#### 返回值

无

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 约束说明

- 调用前需先调用Init接口
- handleId参数必须来自Prepare接口的返回值
- 调用次数应与Prepare接口的repeat次数匹配
- 仅可在AI Cube核或AI Vector核两者之一上调用（不可同时在两种核上调用）
- 仅在core 0上执行

#### 使用示例

```cpp
Hccl hccl;
hccl.Init(reinterpret_cast<GM_ADDR>(&hcclCombineOpParam));
if (g_coreType == AIC) {
    HcclHandle handleId = hccl.ReduceScatter(sendBuf, recvBuf, 100,
                                              HcclDataType::HCCL_DATA_TYPE_INT8,
                                              HcclReduceOp::HCCL_REDUCE_SUM, 10);
    hccl.Commit(handleId);
    int32_t ret = hccl.Wait(handleId);
    hccl.Finalize();
}
```

---

### Wait

#### 功能说明

Wait接口阻塞AI Core，等待handleId对应的通信任务完成。调用本接口前确保已调用过Init接口。

#### 函数原型

```cpp
__aicore__ inline int32_t Wait(HcclHandle handleId)
```

#### 参数说明

| 参数     | 输入/输出 | 说明                                            |
| -------- | --------- | ----------------------------------------------- |
| handleId | 输入      | 通信任务标识ID；必须使用Prepare原语接口的返回值 |

类型定义：`using HcclHandle = int8_t;`

#### 返回值

- 0：成功
- -1：失败

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 约束说明

- 必须先调用Init接口
- handleId参数必须来自Prepare原语接口的返回值
- 调用次数必须与Prepare调用中的repeat参数匹配
- 调用顺序必须与对应的Prepare调用一致
- 仅可在AI Cube核或AI Vector核两者之一上执行（不可同时在两种核上执行）

#### 使用示例

```cpp
Hccl hccl;
hccl.Init(reinterpret_cast<GM_ADDR>(&hcclCombineOpParam));
if (g_coreType == AIC) {
    HcclHandle handleId = hccl.ReduceScatter(sendBuf, recvBuf, 100,
        HcclDataType::HCCL_DATA_TYPE_INT8, HcclReduceOp::HCCL_REDUCE_SUM, 10);
    hccl.Commit(handleId);
    int32_t ret = hccl.Wait(handleId);
    hccl.Finalize();
}
```

---

### Query

#### 功能说明

Query函数获取handleId对应的通信任务已完成的轮次数，最多返回创建任务时指定的repeat值。该接口默认在所有核上执行，用户可在调用前通过GetBlockIdx指定特定核。

#### 函数原型

```cpp
__aicore__ inline int32_t Query(HcclHandle handleId)
```

#### 参数说明

| 参数     | 输入/输出 | 说明                                                  |
| -------- | --------- | ----------------------------------------------------- |
| handleId | 输入      | 对应通信任务的标识ID，只能使用Prepare原语接口的返回值 |

类型定义：

```cpp
using HcclHandle = int8_t;
```

#### 返回值

- 返回handleId对应通信任务已完成的执行次数，最大值等于repeat
- 执行异常时返回-1

#### 支持型号

Atlas A2训练系列产品/Atlas 800I A2推理产品

#### 约束说明

- 调用本接口前确保已调用过Init接口
- handleId参数必须来自对应的Prepare原语接口
- 该接口只能在AI Cube核或AI Vector核两者之一上执行

#### 使用示例

```cpp
Hccl hccl;
hccl.Init(reinterpret_cast<GM_ADDR>(&hcclCombineOpParam));
if (g_coreType == AIC) {
    auto repeat = 10;
    HcclHandle handleId = hccl.ReduceScatter(sendBuf, recvBuf, 100,
        HcclDataType::HCCL_DATA_TYPE_INT8, HcclReduceOp::HCCL_REDUCE_SUM, repeat);
    hccl.Commit(handleId);
    int32_t finishedCount = hccl.Query(handleId);
    while (hccl.Query(handleId) < repeat) {}
    hccl.Finalize();
}
```

---

### Finalize

#### 功能说明

通知服务端后续不会再有通信任务，允许服务端在执行完成后退出。客户端会检测并等待最后一个通信任务完成。

#### 函数原型

```cpp
__aicore__ inline void Finalize()
```

#### 返回值

无

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 注意事项

- 调用本接口前必须先调用Init接口
- 该接口只能在AI Cube核或AI Vector核两者之一上调用
- 该接口仅在core 0上执行

#### 使用示例

```cpp
// 假设HcclCombineOpParam对象hcclCombineOpParam、recvBuf和sendBuf已构造
Hccl hccl;
hccl.Init(reinterpret_cast<GM_ADDR>(&hcclCombineOpParam));
if (g_coreType == AIC) {
    HcclHandle handleId = hccl.ReduceScatter(sendBuf, recvBuf, 100,
                          HcclDataType::HCCL_DATA_TYPE_INT8,
                          HcclReduceOp::HCCL_REDUCE_SUM, 10);
    hccl.Commit(handleId);      // 通知服务端执行ReduceScatter任务
    int32_t ret = hccl.Wait(handleId);  // 阻塞等待ReduceScatter完成
    hccl.Finalize();            // 通知服务端可以在ReduceScatter任务后退出
}
```

---

### GetWindowsInAddr

#### 功能说明

`GetWindowsInAddr`函数获取卡间通信数据(WindowsIn)的起始地址，可直接作为计算的输入/输出使用，减少数据拷贝开销。默认在所有核上执行，用户可通过`GetBlockIdx`指定在特定核上执行。

#### 函数原型

```cpp
__aicore__ inline GM_ADDR GetWindowsInAddr(uint32_t rankId)
```

#### 参数说明

| 参数   | 输入/输出 | 说明           |
| ------ | --------- | -------------- |
| rankId | 输入      | 待查询的卡的Id |

#### 返回值

返回指定卡的WindowsIn卡间通信数据的起始地址。当rankId无效时返回nullptr。

#### 支持型号

Atlas A2训练系列产品/Atlas 800I A2推理产品

#### 注意事项

该接口只能在AI Cube核或AI Vector核两者之一上调用。

#### 使用示例

```cpp
Hccl hccl;
// 在AscendC自定义算子kernel中获取Hccl上下文
// 假设一个通信域内有4张卡
GM_ADDR contextGM1 = GetHcclContext<HCCL_GROUP_ID_0>();
hccl.Init(contextGM1);

auto winInAddr = hccl.GetWindowsInAddr(0);
auto winOutAddr = hccl.GetWindowsOutAddr(0);
auto rankId = hccl.GetRankId();
auto rankDim = hccl.GetRankDim();  // 4张卡
```

---

### GetWindowsOutAddr

#### 功能说明

获取卡间通信数据(WindowsOut)的起始地址，可直接作为计算的输入/输出使用，从而减少拷贝操作。默认在所有核上执行，用户可在调用前通过`GetBlockIdx`指定在特定核上执行。

#### 函数原型

```cpp
__aicore__ inline GM_ADDR GetWindowsOutAddr(uint32_t rankId)
```

#### 参数说明

| 参数   | 输入/输出 | 说明           |
| ------ | --------- | -------------- |
| rankId | 输入      | 待查询的卡的Id |

#### 返回值

返回指定卡的WindowsOut卡间通信数据的起始地址。当rankId无效时返回`nullptr`。

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 约束说明

该接口只能在AI Cube核或者AI Vector核两者之一上调用。

#### 相关参考

使用示例请参考`GetWindowsInAddr`文档及`GetBlockIdx`函数说明。

---

### GetRankId

#### 功能说明

获取当前卡的RankId。默认在所有核上执行，用户可在调用前通过GetBlockIdx指定在特定核上执行。

#### 函数原型

```cpp
__aicore__ inline uint32_t GetRankId()
```

#### 返回值

返回当前卡的RankId。

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 约束说明

该接口只能在AI Cube核或者AI Vector核两者之一上调用。

#### 使用示例

使用示例请参考GetWindowsInAddr文档。

---

### GetRankDim

#### 功能说明

获取通信域内的卡数量。该接口默认在所有核上执行，用户可在调用前通过GetBlockIdx指定在特定核上执行。

#### 函数原型

```cpp
__aicore__ inline uint32_t GetRankDim()
```

#### 返回值

返回通信域内的卡数量。

#### 支持型号

- Atlas A2训练系列产品
- Atlas 800I A2推理产品

#### 约束说明

该接口只能在AI Cube核或者AI Vector核两者之一上调用，不可同时在两种核上调用。

#### 使用示例

使用示例请参考GetWindowsInAddr文档。

---

## Hccl Tiling

### v1版本TilingData

#### 功能说明

AI CPU在启动通信任务前需获取固定通信配置。Tiling通过组装通信配置项，使用固定参数和固定顺序的方式将信息传递给AI CPU通信接口。

#### Mc2Msg参数说明

##### 需要用户显式赋值的参数

preparePosition (uint32_t)
- 设置服务端组装任务的方式
- 取值1：AI Core通过消息通知时设置，用于Hccl调用

useBufferType (uint8_t)
- 设置通信算法获取输入数据的位置
- 0: 默认值，通信输入不在windows中
- 1: 功能同0
- 2: 通信输入放在windows中（仅适用AllReduce）

commAlg (uint8_t)
- 设置具体通信算法
- 1: FullMesh算法（NPU全连接）

hasCommOut (uint8_t)
- 本卡通信结果是否输出到recvBuf（仅AllGather和AlltoAll支持）
- 0: 不输出（可提升性能）
- 1: 输出本卡计算结果

##### 保留参数（不可配置）

sendOff、recvOff、tailSendOff、tailRecvOff、sendCnt、recvCnt、tailSendCnt、tailRecvCnt、totalCnt、turnNum、tailNum、stride、workspaceOff、notifyOff、notifyBeginCnt、notifyEndCnt、funID、dataType、groupNum、reuseMode、commType、reduceOp、commOrder、waitPolicy、rspPolicy、exitPolicy、taskType、debugMode、stepSize、sendArgIndex、recvArgIndex、commOutArgIndex、reserve、reserve2

#### 支持型号

Atlas A2训练系列产品/Atlas 800I A2推理产品

#### 注意事项

- Tiling Data结构需按顺序完整包含所有Mc2Msg参数
- 算子注册时保持结构一致性

#### 代码示例

```cpp
BEGIN_TILING_DATA_DEF(Mc2Msg)
    TILING_DATA_FIELD_DEF(uint32_t, preparePosition);
    TILING_DATA_FIELD_DEF(uint8_t, useBufferType);
    TILING_DATA_FIELD_DEF(uint8_t, commAlg);
    TILING_DATA_FIELD_DEF(uint8_t, hasCommOut);
    // ... 其他字段
END_TILING_DATA_DEF;

AllGatherMatmulCustomTilingData tiling;
tiling.msg.set_preparePosition(1);
tiling.msg.set_commAlg(1);
tiling.msg.set_useBufferType(1);
tiling.msg.set_hasCommOut(1);
```

---

### v2版本TilingData

#### 状态说明

本接口暂不支持使用。请开发者关注后续版本更新。

#### 功能说明

AI CPU在启动通信任务前需获取固定通信配置。Tiling组件通过组装通信配置，以标准化TilingData参数的方式将信息传递给AI CPU通信接口。

#### 参数说明

##### 表1：v2 Hccl TilingData参数

| 参数        | 类型         | 说明                                                               |
| ----------- | ------------ | ------------------------------------------------------------------ |
| version     | uint32_t     | 区分TilingData版本。v2需设为2。占用与v1的preparePosition相同的位置 |
| mc2HcommCnt | uint32_t     | 所有域中通信任务总数。最大支持值：3                                |
| serverCfg   | Mc2ServerCfg | 通用服务端通信配置                                                 |
| hcom        | Mc2HcommCfg  | 每个任务的配置参数。定义mc2HcommCnt个实例                          |

##### 表2：Mc2ServerCfg结构体

所有字段为保留字段，无需配置：version、debugMode、sendArgIndex、recvArgIndex、commOutArgIndex、reserved。

##### 表3：Mc2HcommCfg结构体

| 参数                 | 类型             | 说明                                                                                                                         |
| -------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| skipLocalRankCopy    | 保留             | --                                                                                                                           |
| skipBufferWindowCopy | 保留             | --                                                                                                                           |
| stepSize             | 保留             | --                                                                                                                           |
| reserved             | 保留             | --                                                                                                                           |
| groupName            | char*（最大128） | 通信域名称                                                                                                                   |
| algConfig            | char*（最大128） | 算法配置。支持："AllGather=level0:doublering"、"ReduceScatter=level0:doublering"、"AlltoAll=level0:fullmesh;level1:pairwise" |
| opType               | uint32_t         | 通信任务类型（见HcclCMDType）                                                                                                |
| reduceType           | uint32_t         | 适用任务的规约操作类型                                                                                                       |

##### 表4：HcclCMDType枚举

```cpp
enum class HcclCMDType {
    HCCL_CMD_INVALID = 0,
    HCCL_CMD_BROADCAST = 1,
    HCCL_CMD_ALLREDUCE,
    HCCL_CMD_REDUCE,
    HCCL_CMD_SEND,
    HCCL_CMD_RECEIVE,
    HCCL_CMD_ALLGATHER,
    HCCL_CMD_REDUCE_SCATTER,
    HCCL_CMD_ALLTOALLV,
    HCCL_CMD_ALLTOALLVC,
    HCCL_CMD_ALLTOALL,
    HCCL_CMD_GATHER,
    HCCL_CMD_SCATTER,
    HCCL_CMD_BATCH_SEND_RECV,
    HCCL_CMD_FINALIZE = 100,
    HCCL_CMD_INTER_GROUP_SYNC,
    HCCL_CMD_MAX
};
```

当前支持类型：HCCL_CMD_ALLGATHER、HCCL_CMD_REDUCE_SCATTER、HCCL_CMD_ALLTOALLV。

#### 关键要求

- 使用v2结构时，第一个参数需设version=2
- 算子TilingData必须完整包含所有v2参数，严格遵循结构要求

#### 示例实现

算子定义：

```cpp
namespace ops {
class AlltoallvDoubleCommCustom : public OpDef {
public:
    explicit AlltoallvDoubleCommCustom(const char *name) : OpDef(name) {
        this->Input("x1").ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16, ge::DT_BF16})
            .Format({ge::FORMAT_ND, ge::FORMAT_ND});
        this->Input("x2").ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16, ge::DT_BF16})
            .Format({ge::FORMAT_ND, ge::FORMAT_ND})
            .IgnoreContiguous();
        this->Output("y1").ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16, ge::DT_BF16})
            .Format({ge::FORMAT_ND, ge::FORMAT_ND});
        this->Output("y2").ParamType(REQUIRED)
            .DataType({ge::DT_FLOAT16, ge::DT_BF16})
            .Format({ge::FORMAT_ND, ge::FORMAT_ND});
        this->Attr("group_ep").AttrType(REQUIRED).String();
        this->Attr("group_tp").AttrType(REQUIRED).String();
        this->Attr("ep_world_size").AttrType(REQUIRED).Int();
        this->Attr("tp_world_size").AttrType(REQUIRED).Int();
        this->AICore().SetTiling(optiling::AlltoAllVDoubleCommCustomTilingFunc);
        this->AICore().AddConfig("ascendxxx");
        this->MC2().HcclGroup({"group_ep", "group_tp"});
    }
};
OP_ADD(AlltoallvDoubleCommCustom);
}
```

TilingData声明：

```cpp
BEGIN_TILING_DATA_DEF(AlltoallvDoubleCommCustomTilingData)
    TILING_DATA_FIELD_DEF(uint32_t, version);
    TILING_DATA_FIELD_DEF(uint32_t, mc2HcommCnt);
    TILING_DATA_FIELD_DEF_STRUCT(Mc2ServerCfg, serverCfg);
    TILING_DATA_FIELD_DEF_STRUCT(Mc2HcommCfg, hcom1);
    TILING_DATA_FIELD_DEF_STRUCT(Mc2HcommCfg, hcom2);
END_TILING_DATA_DEF;

REGISTER_TILING_DATA_CLASS(AlltoallvDoubleCommCustom, AlltoallvDoubleCommCustomTilingData);
```

TilingData实现：

```cpp
static ge::graphStatus AlltoAllVDoubleCommCustomTilingFunc(gert::TilingContext *context) {
    char *group1 = const_cast<char *>(context->GetAttrs()->GetAttrPointer<char>(0));
    char *group2 = const_cast<char *>(context->GetAttrs()->GetAttrPointer<char>(1));

    AlltoallvDoubleCommCustomTilingData tiling;
    tiling.set_version(2);
    tiling.set_mc2HcommCnt(2);
    tiling.serverCfg.set_debugMode(0);

    tiling.hcom1.set_opType(8);
    tiling.hcom1.set_reduceType(4);
    tiling.hcom1.set_groupName(group1);
    tiling.hcom1.set_algConfig("AlltoAll=level0:fullmesh;level1:pairwise");

    tiling.hcom2.set_opType(8);
    tiling.hcom2.set_reduceType(4);
    tiling.hcom2.set_groupName(group2);
    tiling.hcom2.set_algConfig("AlltoAll=level0:fullmesh;level1:pairwise");

    tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),
                        context->GetRawTilingData()->GetCapacity());
    context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());
    return ge::GRAPH_SUCCESS;
}
```
