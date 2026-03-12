# HCCL接口简介

HCCL（华为集合通信库）提供了Python与C两种语言的接口。

- Python接口：实现TensorFlow网络在昇腾AI处理器执行分布式优化。
- C语言接口：实现单算子模式下的框架适配，实现分布式能力。以PyTorch为例，将HCCL单算子API嵌入到PyTorch后端代码中后，PyTorch用户可以直接利用框架原生的集合通信接口实现分布式功能。

页面包含一张图表（图1），标题为"集合通信API概览"，展示HCCL相关的集合通信接口结构。

# HCCL API(C)

框架开发者可以通过HCCL提供的C语言接口进行单算子模式下的框架适配，实现分布式能力。

## 章节目录

1. 接口列表
2. 通信域管理
3. 集合通信
4. 点对点通信
5. 异常处理
6. 数据类型
7. 预留接口

# 接口列表

接口定义位于CANN软件安装目录/include/hccl/下：
- `hccl.h`：包含通信域管理与算子接口
- `hccl_types.h`：数据类型定义

## 通信域管理

| 接口                          | 说明                              |
| ----------------------------- | --------------------------------- |
| HcclCommInitClusterInfo       | 基于ranktable初始化通信域         |
| HcclCommInitClusterInfoConfig | 基于ranktable和config初始化通信域 |
| HcclGetRootInfo               | 获取Root信息                      |
| HcclCommInitRootInfo          | 基于RootInfo初始化通信域          |
| HcclCommInitRootInfoConfig    | 基于RootInfo和config初始化通信域  |
| HcclCommConfigInit            | 通信域配置初始化                  |
| HcclCommInitAll               | 初始化所有通信域                  |
| HcclCommDestroy               | 销毁通信域                        |
| HcclGetRankSize               | 获取rank数量                      |
| HcclGetRankId                 | 获取rank id                       |
| HcclSetConfig                 | 设置配置                          |
| HcclGetConfig                 | 获取配置                          |
| HcclGetCommName               | 获取通信域名称                    |
| HcclGetCommConfigCapability   | 获取通信域配置能力                |

## 集合通信

| 接口              | 说明           |
| ----------------- | -------------- |
| HcclAllReduce     | 归约并广播结果 |
| HcclBroadcast     | 广播           |
| HcclAllGather     | 全收集         |
| HcclReduceScatter | 归约分发       |
| HcclReduce        | 归约           |
| HcclAlltoAllV     | 全交换（变长） |
| HcclAlltoAll      | 全交换         |
| HcclBarrier       | 屏障同步       |
| HcclScatter       | 分发           |

## 点对点通信

| 接口              | 说明               |
| ----------------- | ------------------ |
| HcclSend          | 发送               |
| HcclRecv          | 接收               |
| HcclBatchSendRecv | 异步批量点对点通信 |

## 异常处理

| 接口                  | 说明         |
| --------------------- | ------------ |
| HcclGetCommAsyncError | 检测RDMA错误 |
| HcclGetErrorString    | 解析错误码   |

# 通信域管理

通信域管理包含以下API函数：

1. HcclCommInitClusterInfo
2. HcclCommInitClusterInfoConfig
3. HcclGetRootInfo
4. HcclCommInitRootInfo
5. HcclCommInitRootInfoConfig
6. HcclCommConfigInit
7. HcclCommInitAll
8. HcclCommDestroy
9. HcclGetRankSize
10. HcclGetRankId
11. HcclSetConfig
12. HcclGetConfig
13. HcclGetCommName
14. HcclGetCommConfigCapability
15. HcclCreateSubCommConfig

上级主题：HCCL API(C)

# HcclCommInitClusterInfo

## 功能说明

基于ranktable文件初始化HCCL通信域。ranktable是JSON格式文件，配置参与集合通信的NPU资源信息。

## 函数原型

```c
HcclResult HcclCommInitClusterInfo(const char *clusterInfo, uint32_t rank, HcclComm *comm)
```

## 参数说明

| 参数名      | 输入/输出 | 描述                                                              |
| ----------- | --------- | ----------------------------------------------------------------- |
| clusterInfo | 输入      | ranktable文件路径（含文件名），字符串最大长度4096字节（含结束符） |
| rank        | 输入      | 本rank的rank id，需与ranktable中"rank_id"字段一致                 |
| comm        | 输出      | 初始化后的通信域指针信息回传                                      |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

重复初始化会报错。

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

详见HcclCommInitClusterInfo初始化方式相关文档。

# HcclCommInitClusterInfoConfig

## 功能说明

基于ranktable初始化具有特定配置的HCCL通信域。

## 函数原型

```c
HcclResult HcclCommInitClusterInfoConfig(const char *clusterInfo, uint32_t rank, HcclCommConfig *config, HcclComm *comm)
```

## 参数说明

| 参数名      | 输入/输出 | 描述                                                                                                                                                                                                                                                 |
| ----------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| clusterInfo | 输入      | ranktable文件路径（含文件名），字符串最大长度4096字节（含结束符）                                                                                                                                                                                    |
| rank        | 输入      | 本rank的rank id，需与ranktable中"rank_id"字段一致                                                                                                                                                                                                    |
| config      | 输入      | 通信域配置项（缓冲区大小、确定性计算开关、通信域名称）。必须先调用HcclCommConfigInit初始化。参数需在合法值域内。通信域名称需与其他通信域不重复。hcclBufferSize优先级高于HCCL_BUFFSIZE环境变量。hcclDeterministic优先级高于HCCL_DETERMINISTIC环境变量 |
| comm        | 输出      | 初始化后的通信域指针信息                                                                                                                                                                                                                             |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

同一通信域不支持重复初始化。

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

# HcclGetRootInfo

## 功能说明

在HCCL初始化前调用，仅在root节点执行，用于生成root节点的rank标识信息(HcclRootInfo)。

## 函数原型

```c
HcclResult HcclGetRootInfo(HcclRootInfo *rootInfo)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                               |
| -------- | --------- | ---------------------------------------------------------------------------------- |
| rootInfo | 输出      | 本rank的标识信息，包含device ip、device id等，需广播至集群内所有rank进行HCCL初始化 |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

多机集合通信场景操作（非必选）：
- 配置HCCL_IF_IP或HCCL_SOCKET_IFNAME环境变量指定root网卡IP
- HCCL_IF_IP优先级高于HCCL_SOCKET_IFNAME
- 默认使用网卡名字典序升序选择

白名单配置：
- 通过HCCL_WHITELIST_DISABLE开启白名单校验
- 使用HCCL_WHITELIST_FILE指定配置文件（默认关闭）

使用说明：
- 调用后需将rank标识信息广播至所有集群节点
- 随后调用HcclCommInitRootInfo接口进行初始化
- 与HcclCommInitRootInfo接口配对使用，不能单独使用

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

# HcclCommInitRootInfo

## 功能说明

根据rootInfo初始化HCCL，并建立HCCL通信域。

## 函数原型

```c
HcclResult HcclCommInitRootInfo(uint32_t nRanks, const HcclRootInfo *rootInfo, uint32_t rank, HcclComm *comm)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                      |
| -------- | --------- | ------------------------------------------------------------------------- |
| nRanks   | 输入      | 集群中的rank数量                                                          |
| rootInfo | 输入      | root rank信息，主要包含root rank的ip、id等信息，由HcclGetRootInfo接口生成 |
| rank     | 输入      | 本rank的rank id                                                           |
| comm     | 输出      | 初始化后的通信域指针，HcclComm类型定义详见相关文档                        |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 同一通信域中所有rank的nRanks、rootInfo均应相同
- 该接口只能串行调用，不支持并发调用

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

# HcclCommInitRootInfoConfig

## 功能说明

基于root信息初始化具有特定配置的HCCL通信域。

## 函数原型

```c
HcclResult HcclCommInitRootInfoConfig(uint32_t nRanks, const HcclRootInfo *rootInfo, uint32_t rank, const HcclCommConfig *config, HcclComm *comm)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                                                                                   |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| nRanks   | 输入      | 集群中的rank数量                                                                                                                                                                                                                                       |
| rootInfo | 输入      | root rank信息，包含root rank的IP和ID等信息，由HcclGetRootInfo接口生成                                                                                                                                                                                  |
| rank     | 输入      | 本rank的rank id                                                                                                                                                                                                                                        |
| config   | 输入      | 通信域配置项，包括缓冲区大小、确定性计算开关、通信域名称。必须先调用HcclCommConfigInit初始化。参数需在合法值域内。通信域名称需与其他通信域不重复。hcclBufferSize优先级高于HCCL_BUFFSIZE环境变量。hcclDeterministic优先级高于HCCL_DETERMINISTIC环境变量 |
| comm     | 输出      | 初始化后的通信域指针                                                                                                                                                                                                                                   |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 同一通信域中所有rank的nRanks、rootInfo和config均应相同
- 该接口只能串行调用，不支持并发调用

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

# HcclCommConfigInit

## 功能说明

初始化通信域配置项，将可配置参数设置为默认值。默认值：hcclBufferSize为200，hcclDeterministic为0。

## 函数原型

```c
inline void HcclCommConfigInit(HcclCommConfig *config)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                     |
| ------ | --------- | ------------------------ |
| config | 输出      | 需要初始化的通信域配置项 |

HcclCommConfig类型定义详见数据类型章节。

## 返回值

无。

## 约束说明

无。

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```c
HcclCommConfig config;
HcclCommConfigInit(&config);
```

# HcclCommInitAll

## 功能说明

在单机通信场景中通过一个进程为多张卡创建通信域，其中devices[0]作为root rank自动收集集群信息。

## 函数原型

```c
HcclResult HcclCommInitAll(uint32_t ndev, int32_t* devices, HcclComm* comms)
```

## 参数说明

| 参数名  | 输入/输出 | 描述                                                    |
| ------- | --------- | ------------------------------------------------------- |
| ndev    | 输入      | 通信域内的device个数                                    |
| devices | 输入      | 通信域中的device列表（逻辑ID），不能包含重复的device ID |
| comms   | 输出      | 生成的通信域句柄数组，大小为：ndev * sizeof(HcclComm)   |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 仅支持单机通信场景，不支持多机通信
- 多线程调用时需确保不同线程间调用时间差不超过环境变量HCCL_CONNECT_TIMEOUT
- 单张卡不支持同时调用多个通信操作API

## 支持型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

# HcclCommDestroy

## 功能描述

销毁指定的HCCL通信域。

## 原型

```c
HcclResult HcclCommDestroy(HcclComm comm)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                         |
| ------ | --------- | ------------------------------------------------------------ |
| comm   | 输入      | 指向需要销毁的通信域的指针。HcclComm类型的定义可参见HcclComm |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclGetRankSize

## 功能描述

查询当前集合通信域的rank总数。

## 原型

```c
HcclResult HcclGetRankSize(HcclComm comm, uint32_t *rankSize)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                 |
| -------- | --------- | ------------------------------------ |
| comm     | 输入      | 集合通信操作所在的通信域             |
| rankSize | 输出      | 指定集合通信域的rank总数输出地址指针 |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```c
uint32_t rankSize;
HcclComm comm;
HcclGetRankSize(comm, &rankSize);
```

# HcclGetRankId

## 功能描述

获取device在集合通信域中对应的rank序号。

## 原型

```c
HcclResult HcclGetRankId(HcclComm comm, uint32_t *rank)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                       |
| ------ | --------- | ---------------------------------------------------------- |
| comm   | 输入      | 集合通信操作所在的通信域。HcclComm类型的定义可参见HcclComm |
| rank   | 输出      | 指定集合通信域中rank序号的输出地址指针                     |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```c
uint32_t rank;
HcclComm comm;
HcclGetRankId(comm, &rank);
```

# HcclSetConfig

## 功能描述

进行集合通信相关配置。当前版本仅支持配置是否启用确定性计算。在启用确定性计算的情况下，算子在相同的硬件和输入下，多次执行将产生相同的输出。默认情况下无需开启，但可在调试或精度优化时使用，需注意性能会有所下降。

## 原型

```c
HcclResult HcclSetConfig(HcclConfig config, HcclConfigValue configValue)
```

## 参数说明

| 参数名      | 输入/输出 | 描述                                                               |
| ----------- | --------- | ------------------------------------------------------------------ |
| config      | 输入      | 指定可配置参数，当前版本仅支持HCCL_DETERMINISTIC                   |
| configValue | 输入      | 参数取值。对于HCCL_DETERMINISTIC，0表示不支持确定性计算，1表示支持 |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclGetConfig

## 功能描述

获取集合通信相关配置。

## 原型

```c
HcclResult HcclGetConfig(HcclConfig config, HcclConfigValue *configValue)
```

## 参数说明

| 参数名      | 输入/输出 | 描述                                                                                       |
| ----------- | --------- | ------------------------------------------------------------------------------------------ |
| config      | 输入      | 集合通信配置参数。当前版本仅支持配置为HCCL_DETERMINISTIC                                   |
| configValue | 输出      | 获取config中可配置的参数的值。针对HCCL_DETERMINISTIC参数，0代表不支持确定性计算，1代表支持 |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclGetCommName

## 功能描述

获取当前集合通信操作所在的通信域的名称。

## 原型

```c
HcclResult HcclGetCommName(HcclComm comm, char* commName)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                |
| -------- | --------- | ----------------------------------- |
| comm     | 输入      | 集合通信操作所在的通信域            |
| commName | 输出      | 获取到的通信域名称，最大长度支持128 |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```c
char commName[128];
HcclComm comm;
HcclGetCommName(comm, commName);
```

# HcclGetCommConfigCapability

## 功能描述

判断当前版本软件是否支持某项通信域初始化配置。支持的完整配置项包括共享数据的缓存区大小、确定性计算开关、通信域名称等。

使用流程：
1. 调用接口获取代表通信域初始化配置能力的数值
2. 将该数值与HcclCommConfigCapability中的枚举值进行比较
3. 若返回值大于枚举值，则支持该配置；若小于等于，则不支持

## 原型

```c
uint32_t HcclGetCommConfigCapability()
```

## 参数说明

无参数。

## 返回值

uint32_t类型，表示通信域初始化配置能力的数值。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```c
uint32_t configCapability = HcclGetCommConfigCapability();
bool isSupportCommName = configCapability > HCCL_COMM_CONFIG_COMM_NAME;
```

# HcclCreateSubCommConfig

## 功能描述

基于既有的全局通信域，切分具有特定配置的子通信域。该创建方式无需进行socket建链与rank信息交换，可应用于业务故障下的快速通信域创建。

## 原型

```c
HcclResult HcclCreateSubCommConfig(HcclComm comm, uint32_t rankNum,
    uint32_t *rankIds, uint64_t subCommId, uint32_t subCommRankId,
    HcclCommConfig *config, HcclComm *subComm)
```

## 参数说明

| 参数名        | 输入/输出 | 描述                                                             |
| ------------- | --------- | ---------------------------------------------------------------- |
| comm          | 输入      | 被切分的全局通信域                                               |
| rankNum       | 输入      | 子通信域中的rank数量                                             |
| rankIds       | 输入      | 子通信域中各rank在全局通信域的id数组（需有序排列）               |
| subCommId     | 输入      | 当前子通信域标识（用户自定义，在全局通信域中唯一）               |
| subCommRankId | 输入      | 本rank在子通信域中的rank id（应为当前rank在rankIds数组中的下标） |
| config        | 输入      | 通信域配置项（buffer大小、确定性计算开关、通信域名称）           |
| subComm       | 输出      | 初始化后的子通信域指针                                           |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 同一子通信域的rank传入参数应一致
- 不需要创建子通信域的rank应传入rankIds==nullptr和subCommId=0xFFFFFFFF
- 不可使用重复的subCommId
- 仅支持从全局通信域切分，不支持通信域嵌套切分

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# 集合通信

本章节为集合通信API目录页，包含以下API：

- HcclAllReduce
- HcclBroadcast
- HcclAllGather
- HcclReduceScatter
- HcclReduce
- HcclAlltoAll
- HcclAlltoAllV
- HcclBarrier
- HcclScatter

# HcclAllReduce

## 功能描述

集合通信算子，将通信域内所有节点的输入数据进行reduce操作后，再把结果发送到所有节点的缓冲区中，其中reduce操作类型由op参数指定。

## 原型

```c
HcclResult HcclAllReduce(void *sendBuf, void *recvBuf, uint64_t count,
    HcclDataType dataType, HcclReduceOp op,
    HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                      |
| -------- | --------- | --------------------------------------------------------------------------------------------------------- |
| sendBuf  | 输入      | 源数据buffer地址                                                                                          |
| recvBuf  | 输出      | 目的数据buffer地址，集合通信结果输出至此                                                                  |
| count    | 输入      | 参与allreduce操作的数据个数                                                                               |
| dataType | 输入      | allreduce操作的数据类型（HcclDataType类型）。支持类型因产品而异（int8、int32、int64、float16、float32等） |
| op       | 输入      | reduce的操作类型，支持：sum、prod、max、min                                                               |
| comm     | 输入      | 集合通信操作所在的通信域                                                                                  |
| stream   | 输入      | 本rank所使用的stream                                                                                      |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 所有rank的count、dataType、op均应相同
- 每个rank只能有一个输入

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclBroadcast

## 功能描述

将通信域内root节点的数据广播到其他rank。

## 原型

```c
HcclResult HcclBroadcast(void *buf, uint64_t count, HcclDataType dataType,
    uint32_t root, HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                     |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------ |
| buf      | 输入/输出 | 对于root节点，是数据源；对于非root节点，是数据接收buffer                                                                 |
| count    | 输入      | 参与操作的数据个数                                                                                                       |
| dataType | 输入      | 数据类型，支持int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64，Atlas A2系列增加bfp16 |
| root     | 输入      | 作为broadcast root的rank id                                                                                              |
| comm     | 输入      | 集合通信操作所在的通信域                                                                                                 |
| stream   | 输入      | 本rank所使用的stream                                                                                                     |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 所有rank的count、dataType、root均应相同
- 全局只能有1个root节点

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclAllGather

## 功能描述

集合通信算子，将通信域内所有节点的输入按照rank id重新排序，然后拼接起来，最终向所有节点的输出发送结果。每个节点接收的AllGather输出内容相同。

## 原型

```c
HcclResult HcclAllGather(void *sendBuf, void *recvBuf, uint64_t sendCount,
    HcclDataType dataType, HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                          |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf   | 输入      | 源数据buffer地址                                                                                                                                              |
| recvBuf   | 输出      | 目的数据buffer地址，集合通信结果输出至此                                                                                                                      |
| sendCount | 输入      | 参与allgather操作的sendBuf数据size，recvBuf的数据size等于count乘以rank size                                                                                   |
| dataType  | 输入      | allgather操作的数据类型（HcclDataType类型）。支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64，及bfp16（仅Atlas A2） |
| comm      | 输入      | 集合通信操作所在的通信域                                                                                                                                      |
| stream    | 输入      | 本rank所使用的stream                                                                                                                                          |

## 返回值

HcclResult类型。接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

所有rank的sendCount、dataType均应相同。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclReduceScatter

## 功能说明

集合通信算子ReduceScatter的操作接口，将所有rank的input相加（或其他操作）后，再把结果均匀分发到各rank的输出buffer中，每个进程根据rank编号获取1/ranksize的数据。

## 函数原型

```c
HcclResult HcclReduceScatter(void *sendBuf, void *recvBuf, uint64_t recvCount,
                             HcclDataType dataType, HcclReduceOp op,
                             HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                    |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf   | 输入      | 源数据buffer地址。                                                                                                                                                                                      |
| recvBuf   | 输出      | 目的数据buffer地址，集合通信结果输出至此buffer中。                                                                                                                                                      |
| recvCount | 输入      | 参与操作的recvBuf的数据个数；sendBuf大小等于recvCount × rank数量。                                                                                                                                      |
| dataType  | 输入      | reduce-scatter操作的数据类型，HcclDataType类型。针对Atlas训练系列产品，支持：int8、int32、int64、float16、float32。针对Atlas A2训练系列产品，支持：int8、int16、int32、int64、float16、float32、bfp16。 |
| op        | 输入      | reduce的操作类型，目前支持：sum、prod、max、min。注：Atlas A2训练系列产品中，"prod"操作不支持int16、bfp16数据类型。                                                                                     |
| comm      | 输入      | 集合通信操作所在的通信域。                                                                                                                                                                              |
| stream    | 输入      | 本rank所使用的stream。                                                                                                                                                                                  |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

所有rank的recvCount、dataType、op均应相同。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclReduce

## 功能说明

集合通信算子Reduce的操作接口，将所有rank的input相加（或其他操作）后，再把结果发送到root节点的输出buffer。

## 函数原型

```c
HcclResult HcclReduce(void *sendBuf, void *recvBuf, uint64_t count,
                      HcclDataType dataType, HcclReduceOp op,
                      uint32_t root, HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                            |
| -------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf  | 输入      | 源数据buffer地址。                                                                                                                                                                              |
| recvBuf  | 输出      | 目的数据buffer地址，集合通信结果输出至此buffer中。                                                                                                                                              |
| count    | 输入      | 参与reduce操作的数据个数，比如只有一个int32数据参与，则count=1。                                                                                                                                |
| dataType | 输入      | reduce操作的数据类型，HcclDataType类型。针对Atlas训练系列产品，支持：int8、int32、int64、float16、float32。针对Atlas A2训练系列产品，支持：int8、int16、int32、int64、float16、float32、bfp16。 |
| op       | 输入      | reduce的操作类型，目前支持：sum、prod、max、min。注：Atlas A2训练系列产品中，"prod"操作不支持int16、bfp16数据类型。                                                                             |
| root     | 输入      | 作为reduce root的rankid。                                                                                                                                                                       |
| comm     | 输入      | 集合通信操作所在的通信域。                                                                                                                                                                      |
| stream   | 输入      | 本rank所使用的stream。                                                                                                                                                                          |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

所有rank的count、dataType、op均应相同。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclAlltoAll

## 功能说明

向通信域内所有rank发送相同数据量的数据，并从所有rank接收相同数据量的数据的集合通信操作接口。

## 函数原型

```c
HcclResult HcclAlltoAll(const void *sendBuf, uint64_t sendCount,
                        HcclDataType sendType, const void *recvBuf,
                        uint64_t recvCount, HcclDataType recvType,
                        HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                  |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf   | 输入      | 源数据buffer地址。                                                                                                                                                                    |
| sendCount | 输入      | 向每个rank发送的数据量。                                                                                                                                                              |
| sendType  | 输入      | 发送数据的HcclDataType类型。针对Atlas训练系列产品，支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，另支持bfp16。 |
| recvBuf   | 输出      | 目的数据buffer地址，集合通信结果输出至此buffer中。                                                                                                                                    |
| recvCount | 输入      | 从每个rank接收的数据量。                                                                                                                                                              |
| recvType  | 输入      | 接收数据的HcclDataType类型。支持数据类型同sendType。                                                                                                                                  |
| comm      | 输入      | 集合通信操作所在的通信域。                                                                                                                                                            |
| stream    | 输入      | 本rank所使用的stream。                                                                                                                                                                |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 所有rank的sendCount、sendType、recvCount、recvType均应相同。
- alltoall操作的性能与NPU之间共享数据的缓存区大小有关，超大数据量时建议通过HCCL_BUFFSIZE环境变量增大缓存区。
- 针对Atlas训练系列产品，alltoall通信域需满足：集群组网下，单server 1p、2p通信域要在同一个cluster内，单server 4p、8p和多server通信域中rank需以cluster为基本单位，server间cluster选取一致。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclAlltoAllV

## 功能说明

集合通信域alltoallv操作接口。向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。

## 函数原型

```c
HcclResult HcclAlltoAllV(const void *sendBuf, const void *sendCounts,
                         const void *sdispls, HcclDataType sendType,
                         const void *recvBuf, const void *recvCounts,
                         const void *rdispls, HcclDataType recvType,
                         HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名     | 输入/输出 | 描述                                                                                                            |
| ---------- | --------- | --------------------------------------------------------------------------------------------------------------- |
| sendBuf    | 输入      | 源数据buffer地址。                                                                                              |
| sendCounts | 输入      | 表示发送数据量的uint64数组，sendCounts[i] = n表示本rank发给rank i的数据量为n。                                  |
| sdispls    | 输入      | 表示发送偏移量的uint64数组，以sendType为基本单位。                                                              |
| sendType   | 输入      | 发送数据的数据类型，支持int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64等。 |
| recvBuf    | 输出      | 目的数据buffer地址，集合通信结果输出至此buffer中。                                                              |
| recvCounts | 输入      | 表示接收数据量的uint64数组，recvCounts[i] = n表示本rank从rank i收到的数据量为n。                                |
| rdispls    | 输入      | 表示接收偏移量的uint64数组，以recvType为基本单位。                                                              |
| recvType   | 输入      | 接收数据的数据类型，支持int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64等。 |
| comm       | 输入      | 集合通信操作所在的通信域。                                                                                      |
| stream     | 输入      | 本rank所使用的stream。                                                                                          |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- alltoallv操作性能与NPU之间共享数据的缓存区大小有关。当通信数据量超过缓存区大小时性能下降明显，建议配置环境变量HCCL_BUFFSIZE增大缓存区。
- 针对Atlas训练系列产品，alltoallv通信域需满足：集群组网下，单server 1p、2p通信域要在同一个cluster内，单server 4p、8p和多server通信域中rank需以cluster为基本单位。
- 该接口不支持非集群场景下使用。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclBarrier

## 功能说明

此函数用于阻塞指定通信域内所有rank的stream，直到所有rank都执行该操作为止。

## 函数原型

```c
HcclResult HcclBarrier(HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                       |
| ------ | --------- | -------------------------- |
| comm   | 输入      | 集合通信操作所在的通信域。 |
| stream | 输入      | 本rank所使用的stream。     |

## 返回值

HcclResult：操作成功返回HCCL_SUCCESS，失败返回其他值。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclScatter

## 功能说明

集合通信域Scatter操作接口，将root节点的数据均分并散布至其他rank。

## 函数原型

```c
HcclResult HcclScatter(void *sendBuf, void *recvBuf, uint64_t recvCount,
                       HcclDataType dataType, uint32_t root,
                       HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                    |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf   | 输入      | 源数据buffer地址。                                                                                                                                                                                                                                                                                      |
| recvBuf   | 输出      | 目的数据buffer地址，集合通信结果输出至此buffer中。                                                                                                                                                                                                                                                      |
| recvCount | 输入      | 参与scatter操作的recvBuf的数据个数，比如只有一个int32数据参与，则count=1。                                                                                                                                                                                                                              |
| dataType  | 输入      | scatter操作的数据类型，HcclDataType类型。针对Atlas训练系列产品，支持数据类型：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，支持数据类型：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| root      | 输入      | 作为scatter root的rank id。                                                                                                                                                                                                                                                                             |
| comm      | 输入      | 集合通信操作所在的通信域。                                                                                                                                                                                                                                                                              |
| stream    | 输入      | 本rank所使用的stream。                                                                                                                                                                                                                                                                                  |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- 所有rank的recvCount、dataType、root均应相同。
- 全局只能有1个root节点。
- 非root节点的sendBuf可以为空。root节点的sendBuf不能为空。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# 点对点通信

## API列表

- HcclSend
- HcclRecv
- HcclBatchSendRecv

父主题：HCCL API (C/C++)

# HcclSend

## 功能说明

集合通信域Send操作接口。将当前rank的sendBuf中的数据发送至目标rank。

## 函数原型

```c
HcclResult HcclSend(void* sendBuf, uint64_t count, HcclDataType dataType,
                    uint32_t destRank, HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                                                                                                                 |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| sendBuf  | 输入      | 源数据buffer地址。                                                                                                                                                                                                                                                                   |
| count    | 输入      | 发送数据的个数。                                                                                                                                                                                                                                                                     |
| dataType | 输入      | 发送数据的数据类型，HcclDataType类型。针对Atlas训练系列产品，支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| destRank | 输入      | 通信域内数据接收端的rank编号。                                                                                                                                                                                                                                                       |
| comm     | 输入      | 集合通信操作所在的通信域。                                                                                                                                                                                                                                                           |
| stream   | 输入      | 本rank所使用的stream。                                                                                                                                                                                                                                                               |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

HcclSend与HcclRecv接口采用同步调用方式，且必须配对使用。一个进程调用HcclSend接口后，需要等到与之配对的HcclRecv接口接收数据后，才可以进行下一个接口调用。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclRecv

## 功能说明

集合通信域Recv操作接口。从srcRank接收数据到当前rank的recvBuf。

## 函数原型

```c
HcclResult HcclRecv(void* recvBuf, uint64_t count, HcclDataType dataType,
                    uint32_t srcRank, HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                                                                                                                 |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| recvBuf  | 输入      | 数据接收buffer地址。                                                                                                                                                                                                                                                                 |
| count    | 输入      | 接收数据的个数。                                                                                                                                                                                                                                                                     |
| dataType | 输入      | 接收数据的数据类型，HcclDataType类型。针对Atlas训练系列产品，支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，支持：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| srcRank  | 输入      | 通信域内数据发送端的rank编号。                                                                                                                                                                                                                                                       |
| comm     | 输入      | 集合通信操作所在的通信域。                                                                                                                                                                                                                                                           |
| stream   | 输入      | 本rank所使用的stream。                                                                                                                                                                                                                                                               |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

HcclSend与HcclRecv接口采用同步调用方式，且必须配对使用。一个进程调用HcclSend接口后，需要等到与之配对的HcclRecv接口接收数据后，才可以进行下一个接口调用。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# HcclBatchSendRecv

## 功能说明

集合通信域异步批量点对点通信操作接口。调用一次接口可以完成本rank上的多个收发任务，本rank发送和接收之间是异步的，发送和接收任务之间不会相互阻塞。

## 函数原型

```c
HcclResult HcclBatchSendRecv(HcclSendRecvItem* sendRecvInfo, uint32_t itemNum,
                             HcclComm comm, aclrtStream stream)
```

## 参数说明

| 参数名       | 输入/输出 | 描述                                                         |
| ------------ | --------- | ------------------------------------------------------------ |
| sendRecvInfo | 输入      | 本rank需要下发的收发任务列表的首地址。HcclSendRecvItem类型。 |
| itemNum      | 输入      | 本rank需要接收和发送的任务个数。                             |
| comm         | 输入      | 集合通信操作所在的通信域。                                   |
| stream       | 输入      | 本rank所使用的stream。                                       |

## 返回值

HcclResult：接口成功返回HCCL_SUCCESS，其他失败。

## 约束说明

- "异步"是指同一张卡上的接收和发送任务是异步的，不会相互阻塞。但是在卡间，收发任务依旧是同步的，因此卡间的收发任务也同HcclSend、HcclRecv一样，必须是一一对应的。
- 任务列表中不能有重复的send/recv任务，重复指向（从）同一rank发送（接收）的两个任务。
- 当前版本此接口不支持Virtual Pipeline(VPP)开启的场景。
- 针对Atlas 200T A2 Box16异构子框，若Server内卡间出现建链失败的情况（错误码：EI0010），需要将环境变量HCCL_INTRA_ROCE_ENABLE配置为1，HCCL_INTRA_PCIE_ENABLE配置为0，让Server内采用RoCE环路进行多卡间的通信。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

# 异常处理

## API列表

- HcclGetCommAsyncError
- HcclGetErrorString

父主题：HCCL API (C/C++)

# HcclGetCommAsyncError

## 功能说明

该接口用于检测通信域内的RDMA ERROR CQE错误。当设备网络链路出现不稳定或拥塞时，设备日志中会出现error cqe信息。

当前版本支持查询通信域内是否存在此类RDMA错误。该接口为同步接口，调用者需要等待结果返回。

## 函数原型

```c
HcclResult HcclGetCommAsyncError(HcclComm comm, HcclResult *asyncError)
```

## 参数说明

| 参数名     | 输入/输出 | 描述                                                |
| ---------- | --------- | --------------------------------------------------- |
| comm       | 输入      | 通信域。                                            |
| asyncError | 输出      | 查询结果，为0表示无错误；其他值参见HcclResult类型。 |

## 返回值

HcclResult类型。当前版本仅返回HCCL_E_REMOTE错误类型。

## 约束说明

- 调用此接口前通信域必须已建立。
- 通信域销毁后不能调用此接口。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# HcclGetErrorString

## 功能说明

将HCCL_RESULT类型的错误码转换为可读的字符串。

## 函数原型

```c
const char* HcclGetErrorString(HcclResult code)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                      |
| ------ | --------- | ------------------------- |
| code   | 输入      | HCCL_RESULT类型的错误码。 |

## 返回值

HcclResult错误码对应的字符串表示。

## 约束说明

无。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

# 数据类型

本页面列举了HCCL API中的核心数据类型定义。

## 数据类型列表

- HcclResult - 结果状态类型
- HcclReduceOp - 归约操作类型
- HcclDataType - 数据类型枚举
- HcclConfig - 配置类型
- HcclConfigValue - 配置值类型
- HcclRootInfo - 根节点信息类型
- HcclComm - 通信器类型
- HcclSendRecvType - 发送接收类型
- HcclSendRecvItem - 发送接收项类型
- HcclCommConfig - 通信配置类型
- HcclCommConfigCapability - 通信配置能力类型

# HcclResult

## 功能

定义集合通信相关操作的返回值。

## 定义原型

```c
typedef enum {
    HCCL_SUCCESS = 0,               /* success */
    HCCL_E_PARA = 1,                /* parameter error */
    HCCL_E_PTR = 2,                 /* empty pointer */
    HCCL_E_MEMORY = 3,              /* memory error */
    HCCL_E_INTERNAL = 4,            /* internal error */
    HCCL_E_NOT_SUPPORT = 5,         /* not support feature */
    HCCL_E_NOT_FOUND = 6,           /* not found specific resource */
    HCCL_E_UNAVAIL = 7,             /* resource unavailable */
    HCCL_E_SYSCALL = 8,             /* call system interface error */
    HCCL_E_TIMEOUT = 9,             /* timeout */
    HCCL_E_OPEN_FILE_FAILURE = 10,  /* open file fail */
    HCCL_E_TCP_CONNECT = 11,        /* tcp connect fail */
    HCCL_E_ROCE_CONNECT = 12,       /* roce connect fail */
    HCCL_E_TCP_TRANSFER = 13,       /* tcp transfer fail */
    HCCL_E_ROCE_TRANSFER = 14,      /* roce transfer fail */
    HCCL_E_RUNTIME = 15,            /* call runtime api fail */
    HCCL_E_DRV = 16,                /* call driver api fail */
    HCCL_E_PROFILING = 17,          /* call profiling api fail */
    HCCL_E_CCE = 18,                /* call cce api fail */
    HCCL_E_NETWORK = 19,            /* call network api fail */
    HCCL_E_AGAIN = 20,              /* try again */
    HCCL_E_REMOTE = 21,             /* error cqe */
    HCCL_E_SUSPENDING = 22,         /* error communicator suspending */
    HCCL_E_RESERVED                 /* reserved */
} HcclResult;
```

父主题：数据类型

# HcclReduceOp

## 功能

定义集合通信reduce操作的类型。

## 定义原型

```c
typedef enum {
    HCCL_REDUCE_SUM = 0,    /* sum */
    HCCL_REDUCE_PROD = 1,   /* prod */
    HCCL_REDUCE_MAX = 2,    /* max */
    HCCL_REDUCE_MIN = 3,    /* min */
    HCCL_REDUCE_RESERVED    /* reserved */
} HcclReduceOp;
```

## 枚举值说明

| 枚举常量               | 说明         |
| ---------------------- | ------------ |
| `HCCL_REDUCE_SUM`      | 求和操作     |
| `HCCL_REDUCE_PROD`     | 求积操作     |
| `HCCL_REDUCE_MAX`      | 求最大值操作 |
| `HCCL_REDUCE_MIN`      | 求最小值操作 |
| `HCCL_REDUCE_RESERVED` | 保留         |

父主题：数据类型

# HcclDataType

## 功能

定义集合通信操作所支持的各种数据格式。

## 定义原型

```c
typedef enum {
    HCCL_DATA_TYPE_INT8 = 0,      /* int8 */
    HCCL_DATA_TYPE_INT16 = 1,     /* int16 */
    HCCL_DATA_TYPE_INT32 = 2,     /* int32 */
    HCCL_DATA_TYPE_FP16 = 3,      /* float16 */
    HCCL_DATA_TYPE_FP32 = 4,      /* float32 */
    HCCL_DATA_TYPE_INT64 = 5,     /* int64 */
    HCCL_DATA_TYPE_UINT64 = 6,    /* uint64 */
    HCCL_DATA_TYPE_UINT8 = 7,     /* uint8 */
    HCCL_DATA_TYPE_UINT16 = 8,    /* uint16 */
    HCCL_DATA_TYPE_UINT32 = 9,    /* uint32 */
    HCCL_DATA_TYPE_FP64 = 10,     /* fp64 */
    HCCL_DATA_TYPE_BFP16 = 11,    /* bfp16 */
    HCCL_DATA_TYPE_INT128 = 12,   /* int128 */
    HCCL_DATA_TYPE_RESERVED        /* reserved */
} HcclDataType;
```

父主题：数据类型

# HcclConfig

## 功能

定义集合通信相关配置。当前实现仅允许用户配置集合通信是否支持确定性计算。

## 定义原型

```c
typedef enum {
    HCCL_DETERMINISTIC = 0,
    HCCL_CONFIG_RESERVED
} HcclConfig;
```

父主题：数据类型

# HcclConfigValue

## 功能

定义HcclConfig中可配置的参数值，用于控制HCCL操作中的计算行为。

## 定义原型

```c
union HcclConfigValue {
    int32_t value;     /* 0: 不支持确定性计算, 1: 支持确定性计算 */
};
```

## 参数说明

当前支持HcclConfig中的HCCL_DETERMINISTIC参数配置：

- value (int32_t): 控制确定性计算支持
  - `0`：不支持确定性计算
  - `1`：支持确定性计算

父主题：数据类型

# HcclRootInfo

## 功能

存储根节点的排名信息，包括主机IP地址、主机端口以及根节点的唯一标识符（由设备ID和时间戳信息组合生成）。

## 定义原型

```c
const uint32_t HCCL_ROOT_INFO_BYTES = 4108; // 4108: root info length
typedef struct HcclRootInfoDef {
    char internal[HCCL_ROOT_INFO_BYTES];
} HcclRootInfo;
```

父主题：数据类型

# HcclComm

## 功能

用于指向当前通信域的句柄。

## 定义原型

```c
typedef void *HcclComm;
```

## 说明

HcclComm是一个指针类型定义，代表与通信域相关联的不透明句柄。该类型用于在HCCL（华为集合通信库）中标识和管理通信操作的上下文环境。

父主题：数据类型

# HcclSendRecvType

## 功能

用于批量点对点通信操作，用来标识当前的任务类型是发送还是接收。

## 定义原型

```c
typedef enum {
    HCCL_SEND = 0,           /* 当前任务是发送任务 */
    HCCL_RECV = 1,           /* 当前任务是接收任务 */
    HCCL_SEND_RECV_RESERVED  /* 保留字段 */
} HcclSendRecvType;
```

## 枚举值说明

| 值                        | 描述           |
| ------------------------- | -------------- |
| `HCCL_SEND`               | 标识为发送操作 |
| `HCCL_RECV`               | 标识为接收操作 |
| `HCCL_SEND_RECV_RESERVED` | 预留扩展字段   |

父主题：数据类型

# HcclSendRecvItem

## 功能

用于批量点对点通信操作，定义每个通信任务的基本信息，包括操作类型、缓冲区地址、数据个数、数据类型和远端rank标识。

## 定义原型

```c
typedef struct HcclSendRecvItemDef {
    HcclSendRecvType sendRecvType;    /* 标识当前任务类型是发送还是接收 */
    void *buf;                        /* 数据发送/接收的buffer地址 */
    uint64_t count;                   /* 发送/接收数据的个数 */
    HcclDataType dataType;            /* 发送/接收数据的数据类型 */
    uint32_t remoteRank;              /* 通信域内数据接收/发送端的rank编号 */
} HcclSendRecvItem;
```

## 参数说明

| 参数         | 说明                           |
| ------------ | ------------------------------ |
| sendRecvType | 标识当前任务是发送还是接收操作 |
| buf          | 发送或接收数据的内存地址       |
| count        | 发送或接收的数据元素个数       |
| dataType     | 传输数据元素的数据类型         |
| remoteRank   | 通信域内远端端点的rank标识     |

父主题：数据类型

# HcclCommConfig

## 功能

用于在通信域初始化过程中定义配置信息，包含缓存区大小、确定性计算开关和通信域名称。

## 定义原型

```c
const uint32_t HCCL_COMM_CONFIG_INFO_BYTES = 24;
const uint32_t COMM_NAME_MAX_LENGTH = 128;

typedef struct HcclCommConfigDef {
    char reserved[HCCL_COMM_CONFIG_INFO_BYTES];
    uint32_t hcclBufferSize;
    uint32_t hcclDeterministic;
    char hcclCommName[COMM_NAME_MAX_LENGTH];
} HcclCommConfig;
```

## 参数说明

| 参数                | 说明                                                |
| ------------------- | --------------------------------------------------- |
| `reserved`          | 保留字段，不可修改                                  |
| `hcclBufferSize`    | 共享数据的缓存区大小，最小值为1 MB，默认值为200 MB  |
| `hcclDeterministic` | 确定性计算开关，0表示关闭，1表示开启，默认值为0     |
| `hcclCommName`      | 通信域名称，最大长度128字符；若不指定，系统自动生成 |

父主题：数据类型

# HcclCommConfigCapability

## 功能

定义在通信域初始化期间可支持的配置选项。

## 定义原型

```c
typedef enum {
    HCCL_COMM_CONFIG_BUFFER_SIZE = 0,       /* 共享数据的缓存区大小 */
    HCCL_COMM_CONFIG_DETERMINISTIC = 1,    /* 确定性计算开关 */
    HCCL_COMM_CONFIG_COMM_NAME = 2,        /* 通信域名称 */
    HCCL_COMM_CONFIG_RESERVED              /* 预留字段 */
} HcclCommConfigCapability;
```

## 枚举成员说明

| 成员                             | 说明                          |
| -------------------------------- | ----------------------------- |
| `HCCL_COMM_CONFIG_BUFFER_SIZE`   | 配置共享数据缓冲区大小        |
| `HCCL_COMM_CONFIG_DETERMINISTIC` | 控制确定性计算功能的开启/关闭 |
| `HCCL_COMM_CONFIG_COMM_NAME`     | 设置通信域的名称              |
| `HCCL_COMM_CONFIG_RESERVED`      | 保留供未来使用的字段          |

父主题：数据类型

# 预留接口

本章节列出的接口均为预留接口，后续有可能变更，不支持开发者使用，开发者无需关注。

## 函数签名

```c
HcclResult HcclCommSuspend(HcclComm comm)
```

```c
HcclResult HcclCommResume(HcclComm comm)
```
