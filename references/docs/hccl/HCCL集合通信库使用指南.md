# HCCL集合通信库使用指南

CANN社区版 8.5.0.alpha002 文档版本 01 (2025-12-12)

---

# 1 HCCL简介

集合通信库HCCL（Huawei Collective Communication Library）是基于昇腾硬件的高性能集合通信库，为计算集群提供高性能、高可靠的通信方案。

## 核心功能

- 提供单机、多机环境中的高性能集合通信和点对点通信。
- 支持AllReduce、Broadcast、AllGather、ReduceScatter、AlltoAll、Send、Receive等集合通信原语。
- 支持Ring、Mesh、Recursive Halving-Doubling（RHD）等通信算法。
- 支持HCCS、RoCE、PCIe等高速通信链路。
- 支持单算子和图模式两种执行模式。

## 软件架构

HCCL是CANN的核心组件，为NPU集群提供高性能、高可靠性的通信方案。HCCL向上支持多种AI框架，向下实现多款昇腾AI处理器之间的高效互联，其架构如图1-1所示。

[图: 集合通信库软件架构图 - 自上而下为: AI模型 -> AI框架(MindSpore/PyTorch/TensorFlow) -> HCCL(HCCL集合通信库(内置通信算子/扩展通信算子) + HCOMM通信基础库) -> Runtime -> Driver -> AI处理器]

HCCL包含HCCL集合通信库与HCOMM（Huawei Communication）通信基础库：

- HCCL集合通信库：提供多种集合通信算子与点对点通信算子，同时也支持开发者自定义扩展通信算子。
- HCOMM通信基础库：提供集合通信域以及通信资源（内存、流、通信连接等）的管理能力。

## 支持的产品型号

- Atlas A3 训练系列产品/Atlas A3 推理系列产品
- Atlas A2 训练系列产品
- Atlas 训练系列产品
- Atlas 300I Duo 推理卡

---

# 2 相关概念

为了您有更好的阅读体验，使用本文档前请先了解HCCL相关概念。

## HCCL基本概念

HCCL的典型通信组网如图2-1所示。

[图: 典型通信组网示例 - 展示AI集群中两个AI节点(AI Server)，每个节点包含rank0-rank3共4个设备，节点内通过总线/网络互联，节点间通过通信域连接，底部展示通信算法示例(Ring)]

上图中涉及如下基本概念：

- AI节点：又称AI Server、计算节点，通常是8卡或16卡的昇腾NPU设备组成的服务器形态的统称。
- AI集群：多个AI节点通过交换设备互联后用于分布式训练或推理的系统。若AI节点间通过灵衢总线交换设备进行连接，组成的组网称之为超节点组网。
- 通信成员：通常称为rank，是参与通信的最小逻辑实体，每个rank都会分配一个唯一标识。
- 通信域：一组通信成员的组合，描述通信范围。一个计算任务可以创建多个通信域，通信成员也可以加入多个通信域。
- 通信算子：在通信域内完成通信任务的算子，集合通信指所有成员一起参与的通信操作，如Broadcast、AllReduce等。
- 通信算法：针对不同网络拓扑、数据量、硬件资源等场景，通信算子通常会采用不同的通信算法实现。

## 术语缩略语

| 名称 | 说明 |
|---|---|
| NPU | Neural Network Processing Unit，神经网络处理单元。采用"数据驱动并行计算"的架构，擅长处理海量的视频和图像类多媒体业务数据，专门用于处理人工智能应用中的大量计算任务。 |
| HCCL | Huawei Collective Communication Library，华为集合通信库。提供单机多卡以及多机多卡间的数据并行、模型并行集合通信方案。 |
| HCOMM | Huawei Communication，华为通信基础库。 |
| HCCS | Huawei Cache Coherence System，华为缓存一致性系统。用于CPU/NPU之间的高速互联。 |
| HCCP | Huawei Collective Communication adaptive Protocol，集合通信适配协议。提供跨NPU设备通信能力，向上屏蔽具体通信协议差异。 |
| TOPO | 拓扑、拓扑结构。一个局域网内或者多个局域网之间的设备连接所构成的网络配置或者布置。 |
| PCIe | Peripheral Component Interconnect Express，一种串行外设扩展总线标准，常用于计算机系统中的外设扩展。 |
| PCIe-SW | PCIe Switch，符合PCIe总线扩展的交换设备。 |
| QP | Queue Pair，队列对。QP是远程直接内存访问技术的核心通信单元，由发送队列（Send Queue，SQ）和接收队列（Receive Queue，RQ）组成，用于管理数据传输任务。 |
| SDMA | System Direct Memory Access，系统直接内存访问技术，简称DMA，允许外围设备直接访问系统内存，而不需要CPU的干预。 |
| RDMA | Remote Direct Memory Access，远程直接内存访问技术，能够直接将数据从一台机器的内存传输到另一台机器，无需双方操作系统的介入，一般指可以跨网络的内存访问方式。 |
| RoCE | RDMA over Converged Ethernet，承载在融合以太网上的RDMA技术，即跨越以太网的RDMA通信方式。 |
| AIV | AI Core中的Vector Core。 |
| TS | Task Scheduler，任务调度器。 |

---

# 3 环境准备

在使用HCCL集合通信库之前，需要先准备运行环境。本章节将详细介绍安装软件包和设置环境变量的步骤。

## 安装驱动固件与CANN软件包

HCCL集合通信库的使用依赖CANN软件包，CANN软件包的详细安装步骤请参考《CANN 软件安装指南》。

1. 安装驱动固件（仅昇腾设备需要），安装步骤请参见"安装NPU驱动和固件"章节。
2. 安装CANN软件包，可参考"快速安装CANN"完成快速安装，可参考其他章节了解更多场景的安装步骤。

CANN软件包安装完成后，HCCL集合通信库的头文件与库文件存储在`${ASCEND_HOME_PATH}/hccl`目录下：

- `${ASCEND_HOME_PATH}/hccl/include`：HCCL提供的C接口头文件。
- `${ASCEND_HOME_PATH}/hccl/lib64`：HCCL的库文件。
- `${ASCEND_HOME_PATH}/hccl/python`：HCCL提供的Python接口。

> 说明：`${ASCEND_HOME_PATH}`为CANN软件包安装后文件存储路径，例如/usr/local/Ascend/ascend-toolkit/latest。

## 设置环境变量

进行程序的编译运行前，需要设置CANN软件环境变量。

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
```

"/usr/local/Ascend"为CANN软件root用户的默认安装路径，如果使用普通用户安装，或指定路径安装，请自行替换。

---

# 4 快速入门

本节以AllReduce算子为例，介绍其在单算子执行模式下的使用方式，帮助用户快速体验集合通信功能。

## AllReduce算子介绍

AllReduce操作是将通信域内所有节点的输入数据进行归约操作后（支持sum、prod、max、min），再把结果发送到所有节点的输出buffer。

[图: AllReduce操作示意图 - 左侧rank0-rank3分别有输入in0-in3，经过AllReduce后，右侧rank0-rank3均输出out，若操作类型为sum，则out[i]=sum(inX[i])]

注意：每个rank只能有一个输入。

## 样例介绍

用户可以点击Link获取完整样例代码，该样例基于root节点信息创建通信域，在一个进程中管理一个AI Server，其中每个NPU设备由一个线程进行管理，主要包含以下功能点：

- 设备检测，通过aclrtGetDeviceCount()接口查询可用设备数量。
- 将rank0作为root节点，通过HcclGetRootInfo()接口生成root节点的rootInfo标识信息。
- 基于rootInfo，在每个线程中通过HcclCommInitRootInfo()接口初始化通信域。
- 调用HcclAllReduce()接口，将通信域内所有rank的输入数据进行相加后，再把结果发送到所有节点，并打印结果。

## 编译运行

在本样例代码目录下执行如下命令：

```bash
make
make test
```

## 结果解析

每个rank的数据初始化为0~7，经过AllReduce操作后，每个rank的结果是所有rank对应位置数据的和（8个rank的数据相加）。

```
Found 8 NPU device(s) available
rankId: 0, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 1, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 2, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 3, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 4, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 5, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 6, output: [ 0 8 16 24 32 40 48 56 ]
rankId: 7, output: [ 0 8 16 24 32 40 48 56 ]
```

## 关键代码解析

1. 将rank0作为root节点，生成rootInfo标识信息，主要包含：Device IP、Device ID等信息。此信息需广播至集群内所有rank用来初始化通信域。

```c
int rootRank = 0;
ACLCHECK(aclrtSetDevice(rootRank));
// 生成root节点信息，各线程使用同一份rootInfo
void *rootInfoBuf = nullptr;
ACLCHECK(aclrtMallocHost(&rootInfoBuf, sizeof(HcclRootInfo)));
HcclRootInfo *rootInfo = (HcclRootInfo *)rootInfoBuf;
HCCLCHECK(HcclGetRootInfo(rootInfo));
```

2. 申请内存，构造输入数据。

```c
// 设置当前线程操作的设备
ACLCHECK(aclrtSetDevice(ctx->device));

// 申请集合通信操作的Device内存
size_t count = ctx->devCount;
size_t mallocSize = count * sizeof(float);
ACLCHECK(aclrtMalloc(&sendBuf, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
ACLCHECK(aclrtMalloc(&recvBuf, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));

// 申请 Host 内存用于存放输入数据，并将内容初始化为：0-7
void *hostBuf = nullptr;
ACLCHECK(aclrtMallocHost(&hostBuf, mallocSize));
float *tmpHostBuff = static_cast<float *>(hostBuf);
for (uint32_t i = 0; i < count; ++i) {
    tmpHostBuff[i] = static_cast<float>(i);
}

// 将Host侧输入数据拷贝到Device侧
ACLCHECK(aclrtMemcpy(sendBuf, mallocSize, hostBuf, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
```

3. 初始化通信域。

```c
HcclComm hcclComm;
HCCLCHECK(HcclCommInitRootInfo(ctx->devCount, ctx->rootInfo, ctx->device, &hcclComm));
```

4. 执行AllReduce集合通信算子。

```c
// 创建任务流
aclrtStream stream;
ACLCHECK(aclrtCreateStream(&stream));

// 执行 AllReduce，将通信域内所有节点的 sendBuf 进行相加后，再把结果发送到所有节点的 recvBuf
HCCLCHECK(HcclAllReduce(sendBuf, recvBuf, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM,
    hcclComm, stream));

// 阻塞等待任务流中的集合通信任务执行完成
ACLCHECK(aclrtSynchronizeStream(stream));
```

5. 释放资源。

```c
ACLCHECK(aclrtFree(sendBuf));     // 释放 Device 侧内存
ACLCHECK(aclrtFree(recvBuf));     // 释放 Device 侧内存
ACLCHECK(aclrtFreeHost(hostBuf)); // 释放 Host 侧内存
ACLCHECK(aclrtDestroyStream(stream)); // 销毁任务流
HCCLCHECK(HcclCommDestroy(hcclComm)); // 销毁通信域
ACLCHECK(aclFinalize());          // 设备去初始化
```

---

# 5 主流框架集成

HCCL在系统中的位置如下图所示。

[图: HCCL在系统中的位置示例 - 上层为AI框架(PyTorch通过Ascend Extension for PyTorch适配、TensorFlow通过TF Adapter适配、MindSpore直接集成)，下层为CANN(Graph Engine中的通信算子图适配 -> HCCL -> Runtime)]

AI框架主要有两种编程执行形态，单算子模式和图模式。因此HCCL也提供了单算子和图模式两种工作方式。

- 单算子模式下，AI框架直接调用HCCL的C接口，下发通信算子给HCCL执行。
- 图模式下，AI框架使用Ascend算子IR将模型的计算过程构造成一张图，通过Graph Engine（简称GE）将图中的通信算子下发给HCCL执行，关于图模式的详细介绍，可参见《图模式开发指南》。

PyTorch和MindSpore框架网络支持HCCL的单算子和图模式两种工作方式，TensorFlow框架网络支持HCCL的图模式。

- PyTorch框架：HCCL API已集成到PyTorch适配插件Ascend Extension for PyTorch的代码中，PyTorch用户指定使用HCCL作为分布式后端，直接使用PyTorch原生通信API，即可实现分布式能力，详细使用方法可参见《Ascend Extension for PyTorch 产品文档》。
- MindSpore框架：HCCL API已集成到MindSpore框架代码中，MindSpore用户指定使用HCCL作为分布式后端，直接使用MindSpore原生通信API，即可实现分布式能力，详细使用方法可参见MindSpore官网。
- TensorFlow：HCCL通过TensorFlow适配插件TF Adapter对接TensorFlow框架，详细使用方法可参见《TensorFlow 1.15模型迁移指南》与《TensorFlow 2.6.5模型迁移指南》。

---

# 6 使用通信库API实现通信功能

## 6.1 简介

HCCL提供了C与Python两种语言的开发接口，用于实现分布式能力。

- C语言接口用于实现单算子模式下的框架适配，实现分布式能力。
- Python语言接口用于实现图模式下的框架适配，当前仅用于实现TensorFlow网络在昇腾AI处理器执行分布式优化。

本章节针对如何调用HCCL的C语言接口开发集合通信功能进行介绍。

开发者调用HCCL C语言接口实现集合通信功能的主要开发流程如下所示。

[图: 集合通信操作流程 - 通信域创建 -> 通信操作 -> 通信域销毁]

1. 首先进行集群信息配置，创建通信域句柄，并初始化HCCL通信域。
2. 实现通信操作，HCCL通信操作包含两大类：点对点通信与集合通信。
   - 点对点通信，指多NPU环境下两个NPU之间直接传输数据的过程，常用于pipeline并行场景下对激活值的数据收发。HCCL提供了不同粒度的点对点通信，包括单rank到单rank的单收单发接口，以及多个rank之间的批量收发接口。
   - 集合通信，指多个NPU共同参与进行数据传输操作，例如AllReduce、AllGather、Broadcast等，常用于大规模集群中不同NPU之间的梯度同步和参数更新等操作。集合通信操作可让所有计算节点并行、高效、有序执行数据交换，提升数据传输效率。
3. 集合通信操作完成后，需要销毁通信域，释放相关内存、流资源。

## 6.2 通信域管理

通信域是集合通信算子执行的上下文，管理对应的通信对象（例如一个NPU就是一个通信对象）和通信所需的资源。通信域中的每个通信对象称为一个rank，每个rank都会分配一个介于0~n-1（n为NPU的数量）的唯一标识。

通信域创建根据用户场景的不同主要有以下几种方式：

- 多机集合通信场景
  - 如果有完整的描述集群信息的rank table文件，可通过HcclCommInitClusterInfo接口创建通信域，或者通过HcclCommInitClusterInfoConfig接口创建具有特定配置的通信域。
  - 如果无完整的rank table文件，可通过HcclGetRootInfo接口与HcclCommInitRootInfo/HcclCommInitRootInfoConfig接口配合使用，基于root节点信息创建通信域。
- 单机集合通信场景，可通过HcclCommInitAll接口在单机内批量创建通信域。
- 基于已有的通信域，可通过HcclCreateSubCommConfig接口切分具有特定配置的子通信域。

> 须知
> - 针对同一个rank，通信算子需要one by one保序调用，不支持并发下发。
> - 针对整个通信域，通信算子的调用在不同rank间要保证同一下发顺序。
> - 同一个通信域内不支持图模式通信和单算子通信混合执行。
> - 同一个通信域内的算子需要由使用者确保串行执行。
> - 同一个NPU上需要串行创建多个通信域。
> - 针对Atlas A3 训练系列产品/Atlas A3 推理系列产品，通信域初始化时，如果组网中存在多个超节点，请将属于同一超节点内的AI Server信息配置在一起。假设有两个超节点，标识分别为"0"和"1"，请先配置"0"中的AI Server信息，再配置"1"中的AI Server信息，不支持"0"中的AI Server信息与"1"中的AI Server信息交叉配置。

### 基于rank table创建通信域

多机集合通信、基于集群信息配置文件（rank table文件）创建通信域的场景，每张卡需要使用一个单独的进程参考如下流程创建通信域：

1. 构造rank table文件（rank table文件的配置可参见10 集群信息配置）。
2. 每张卡分别调用HcclCommInitClusterInfo接口创建通信域，或者调用HcclCommInitClusterInfoConfig接口创建具有特定配置的通信域。

一个简单的代码示例片段如下：

```c
int devId = 0;
// 配置rank table文件路径
char* rankTableFile = "/home/rank_table.json";
// 定义通信域句柄
HcclComm hcclComm;
// 初始化HCCL通信域
HcclCommInitClusterInfo(rankTableFile, devId, &hcclComm);

/* 集合通信操作 */

// 销毁HCCL通信域
HcclCommDestroy(hcclComm);
```

> 说明：针对Atlas A3 训练系列产品/Atlas A3 推理系列产品、Atlas A2 训练系列产品，若业务为单卡多进程场景，建议在rank table配置文件中配置"device_port"字段，且不同的业务进程需要设置不同的端口号，否则业务可能会因为端口冲突运行失败。但需要注意，多进程会对资源开销、通信性能产生一定的影响。

### 基于root节点信息创建通信域

多机集合通信场景，若无完整的集群信息配置文件（rank table文件），HCCL提供了基于root节点信息创建通信域的方式，主要有如下两种典型使用场景：

- 每个Device对应一个业务进程的场景，实现流程如下所示：
  a. 指定HCCL初始化时Host节点使用的通信IP地址或通信网卡（可选）。
     - 方式一：在每个Host节点通过环境变量HCCL_IF_IP配置通信IP地址，该IP地址用于与root节点通信，可以是IPv4或IPv6格式，仅支持配置一个IP地址。配置示例如下：
       ```bash
       export HCCL_IF_IP=10.10.10.1
       ```
     - 方式二：在每个Host节点通过环境变量HCCL_SOCKET_IFNAME配置通信网卡名，通过HCCL_SOCKET_FAMILY配置网卡使用的通信协议，HCCL将通过该网卡名获取Host IP，与root节点通信。配置示例如下：
       ```bash
       # 配置HCCL初始化时通信网卡使用的IP协议版本，AF_INET: IPv4; AF_INET6: IPv6
       export HCCL_SOCKET_FAMILY=AF_INET

       # 支持以下格式的网卡名配置（4种规格自行选择1种即可，环境变量中可配置多个网卡，多个网卡间使用英文逗号分隔，取最先匹配到的网卡作为通信网卡）
       # 精确匹配网卡
       export HCCL_SOCKET_IFNAME==eth0,enp0  # 使用指定的eth0或enp0网卡
       export HCCL_SOCKET_IFNAME=^=eth0,enp0 # 不使用eth0与enp0网卡
       # 模糊匹配网卡
       export HCCL_SOCKET_IFNAME=eth,enp     # 使用所有以eth或enp为前缀的网卡
       export HCCL_SOCKET_IFNAME=^eth,enp    # 不使用任何以eth或enp为前缀的网卡
       ```
       环境变量HCCL_IF_IP的优先级高于HCCL_SOCKET_IFNAME。如果不配置HCCL_IF_IP或HCCL_SOCKET_IFNAME，系统将按照如下优先级自动选择网卡。若当前节点选择的网卡与root节点选择的网卡链路不通，将导致HCCL建链失败。
       docker/lol以外网卡(网卡名称的字典序升序) > docker 网卡 > lo网卡
  b. 在root节点调用HcclGetRootInfo接口，生成root节点的rank标识信息"rootInfo"，包括device ip、device id等信息。
  c. 将root节点的rank信息广播至通信域中的所有rank。
  d. 在通信域中所有节点调用HcclCommInitRootInfo或者HcclCommInitRootInfoConfig接口（创建具有特定配置的通信域），基于接收到的"rootInfo"，以及本rank的rank id等信息，进行通信域初始化。

- 每个AI Server对应一个业务进程，每个线程对应一个Device，通过多线程的方式创建多个通信域的场景，实现流程如下所示：
  a. 参见"每个Device对应一个业务进程"场景的步骤1，指定HCCL初始化时Host节点使用的通信IP地址或通信网卡（可选）。
  b. 在主进程中循环执行"指定不同的Device + 调用HcclGetRootInfo接口"，获取多个"rootInfo"信息。
  c. 每个Device匹配一个线程，分别根据不同的"rootInfo"信息，并发调用HcclCommInitRootInfo或者HcclCommInitRootInfoConfig接口，进行通信域初始化。

> 说明：针对Atlas A3 训练系列产品/Atlas A3 推理系列产品、Atlas A2 训练系列产品，若业务为单卡多进程场景，建议通过环境变量"HCCL_HOST_SOCKET_PORT_RANGE"与"HCCL_NPU_SOCKET_PORT_RANGE"分别配置HCCL在Host侧与NPU侧使用的通信端口，否则可能会导致端口冲突，配置示例如下所示。但需要注意，多进程会对资源开销、通信性能产生一定的影响。
> ```bash
> export HCCL_HOST_SOCKET_PORT_RANGE="auto"
> export HCCL_NPU_SOCKET_PORT_RANGE="auto"
> ```

### 单机内批量创建通信域

单机通信场景中，开发者可通过一个进程统一创建多张卡的通信域，每中一张卡对应一个线程，创建流程如下：

1. 构造通信域中的Device列表，例如：{0, 1, 2, 3, 4, 5, 6, 7}，其中列表中的Device ID是逻辑ID（可通过npu-smi info -m命令查询），HCCL会按照列表中设置的顺序创建通信域。
2. 在进程中调用HcclCommInitAll接口创建通信域。

```c
uint32_t ndev = 8;
// 构造Device的逻辑ID列表
int32_t devices[8] = {0, 1, 2, 3, 4, 5, 6, 7};
// 定义通信域句柄
HcclComm comms[ndev];
// 初始化HCCL通信域
HcclCommInitAll(ndev, devices, comms);

// 启动线程执行集合通信操作
std::vector<std::unique_ptr<std::thread>> threads(ndev);
struct ThreadContext args[ndev];
for (uint32_t i = 0; i < ndev; i++) {
    args[i].device = i;
    args[i].comm = comms[i];
    /* 集合通信操作 */
}

// 销毁HCCL通信域
for (uint32_t i = 0; i < ndev; i++) {
    HcclCommDestroy(comms[i]);
}
```

需要注意，多线程调用集合通信操作API时（例如HcclAllReduce），需要确保不同线程中调用集合通信操作API的前后时间差不超过集合通信的建链超时等待时间（可通过环境变量HCCL_CONNECT_TIMEOUT设置，默认120s），避免建链超时。

### 基于已有通信域切分子通信域

HCCL提供了HcclCreateSubCommConfig接口，实现基于已有通信域切分具有特性配置的子通信域的功能。该子通信域创建方式无需进行socket建链与rank信息交换，可应用于业务故障下的快速通信域创建。

```c
// 初始化全局通信域
HcclComm globalHcclComm;
HcclCommInitClusterInfo(rankTableFile, devId, &globalHcclComm);
// 通信域配置
HcclCommConfig config;
HcclCommConfigInit(&config);
config.hcclBufferSize = 50;
strcpy(config.hcclCommName, "comm_1");
// 初始化子通信域
HcclComm hcclComm;
uint32_t rankIds[4] = {0, 1, 2, 3}; // 子通信域的 Rank 列表
HcclCreateSubCommConfig(&globalHcclComm, 4, rankIds, 1, devId, &config, &hcclComm);
```

> 须知：该接口不支持通信域的嵌套切分，即不支持在子通信域中进一步切分子通信域。

### 销毁通信域

集合通信操作完成后，需要调用HcclCommDestroy接口销毁指定的通信域，并调用运行时管理接口释放通信所用的内存、Stream、Device资源。

## 6.3 集合通信

集合通信是指多个NPU共同参与进行数据传输，从而形成一次集体操作的通信模式，常用于大规模集群中不同NPU之间的梯度同步和参数更新等场景。

HCCL支持AllReduce、Broadcast、AllGather、Scatter、ReduceScatter、Reduce、AlltoAll和AlltoAllV等通信算子，并提供了对应的API供开发者调用，用于快速实现集合通信能力。

### Broadcast

Broadcast操作是将通信域内root节点的数据广播到其他rank。

[图: Broadcast操作示意图 - rank1(root)的输入in广播到所有rank的输出out]

注意：通信域内只能有一个root节点。

相关接口：HcclBroadcast。

### Scatter

Scatter操作是将通信域内root节点的数据均分并散布至其他rank。

[图: Scatter操作示意图 - rank1(root)的输入in被均分为4份，分别散布到rank0(out0)、rank1(out1)、rank2(out2)、rank3(out3)]

注意：通信域内只能有一个root节点。

相关接口：HcclScatter。

### AllGather

AllGather操作是将通信域内所有节点的输入按照rank id重新排序（rank id按照从小到大的顺序排序），然后拼接起来，再将结果发送到所有节点的输出buffer。

[图: AllGather操作示意图 - rank0-rank3分别有输入in0-in3，经AllGather后所有rank均输出拼接结果out]

> 说明：针对AllGather操作，每个节点都接收按照rank id重新排序后的数据集合，即每个节点的AllGather输出都是一样的。

相关接口：HcclAllGather。

### AllGatherV

AllGatherV操作是将通信域内所有节点的输入按照rank id重新排序（rank id按照从小到大的顺序排序），然后拼接起来，再将结果发送到所有节点的输出。与AllGather操作不同的是，AllGatherV操作支持通信域内不同节点的输入配置不同大小的数据量。

[图: AllGatherV操作示意图 - 与AllGather类似，但各rank输入数据量可不同]

> 说明：针对AllGatherV操作，每个节点都接收按照rank id重新排序后的数据集合，即每个节点的AllGatherV输出都是一样的。

相关接口：HcclAllGatherV。

### Reduce

Reduce操作是将通信域内所有rank的输入数据进行归约操作后（支持sum、prod、max、min），再把结果发送到root节点的buffer。

[图: Reduce操作示意图 - rank0-rank3的输入in0-in3经归约后，结果仅发送到rank2(root)的输出out]

注意：通信域内只能有一个root节点。

相关接口：HcclAllReduce。

### AllReduce

AllReduce操作是将通信域内所有节点的输入数据进行归约操作后（支持sum、prod、max、min），再把结果发送到所有节点的输出buffer。

[图: AllReduce操作示意图 - rank0-rank3的输入in0-in3经归约后，结果发送到所有rank的输出out]

注意：每个rank只能有一个输入。

相关接口：HcclAllReduce。

### ReduceScatter

ReduceScatter操作是将通信域内所有rank的输入进行归约操作后（支持sum、prod、max、min），再把结果按照rank编号均匀分散到各个rank的输出buffer，每个进程拿到其他进程1/rank_size份的数据进行归约操作。

如下图所示，有rank0、rank1、rank2、rank3四个rank，每个rank的输入数据切分成4份，每个进程分别取每个rank的1/4份数据进行sum操作（或其他操作），将结果发送到输出buffer。

[图: ReduceScatter操作示意图 - rank0-rank3的输入in0-in3经归约后分散到各rank的输出out0-out3]

相关接口：HcclReduceScatter。

### ReduceScatterV

ReduceScatterV操作是将所有rank的输入进行归约操作后（支持sum、prod、max、min），再把结果按照rank编号分散到各个rank的输出buffer，每个进程拿到其他进程对应rank编号的数据进行归约操作。与ReduceScatter操作不同的是，ReduceScatterV操作支持为通信域内不同的节点配置不同大小的数据量。

如下图所示，有rank0、rank1、rank2、rank3四个rank，每个进程分别取其他进程对应rank编号的数据进行sum操作（或其他操作），将结果发送到输出buffer。

[图: ReduceScatterV操作示意图 - 各rank输入数据按编号分散归约到对应rank的输出]

相关接口：HcclReduceScatterV。

### AlltoAll

AlltoAll操作是向通信域内所有rank发送相同数据量的数据，并从所有rank接收相同数据量的数据。

[图: AlltoAll操作示意图 - 每个rank将数据切分成4份分别发送给4个rank，同时从4个rank接收数据]

AlltoAll操作将输入数据在特定的维度切分成特定的块数，并按顺序发送给其他rank，同时从其他rank接收输入数据，按顺序在特定的维度拼接数据。

相关接口：HcclAlltoAll。

### AlltoAllV

AlltoAllV操作是向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。

[图: AlltoAllV操作示意图 - 与AlltoAll类似，但各rank间发送和接收的数据量可以不同]

相关接口：HcclAlltoAllV。

### AlltoAllVC

AlltoAllVC操作是向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。相比于AlltoAllV，AlltoAllVC通过输入参数sendCountMatrix传入所有rank的收发参数。

[图: AlltoAllVC操作示意图 - 与AlltoAllV类似，通过sendCountMatrix矩阵定义收发参数]

相关接口：HcclAlltoAllVC。

### 接口调用

下面以HcclAllReduce接口为例，介绍其使用示例，HcclAllReduce接口的原型定义如下：

```c
HcclResult HcclAllReduce(void *sendBuf, void *recvBuf, uint64_t count, HcclDataType dataType,
    HcclReduceOp op, HcclComm comm, aclrtStream stream)
```

HcclAllReduce用于将通信域内所有节点的输入进行归约操作，再把结果发送到所有节点的输出，其中op参数用于指定归约的操作类型，当前版本支持的操作类型有sum、prod、max、min。HcclAllReduce允许每个节点只有一个输入。

如下代码片段所示，将通信域内所有输入内存中的数据，按照float32的数据格式执行加法操作（示例中每个rank中只有一个数据参与），然后把相加结果发送到所有节点的输出内存。

```c
void* hostBuf = nullptr;
void* sendBuf = nullptr;
void* recvBuf = nullptr;
uint64_t count = 1;
int malloc_kSize = count * sizeof(float);
aclrtStream stream;
aclrtCreateStream(&stream);

//申请集合通信操作的内存
aclrtMalloc((void**)&sendbuff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST);
aclrtMalloc((void**)&recvbuff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST);

//初始化输入内存
aclrtMallocHost((void**)&hostBuf, malloc_kSize);
aclrtMemcpy((void*)sendBuf, malloc_kSize, (void*)hostBuf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE);

//执行集合通信操作
HcclAllReduce((void *)sendBuf, (void*)recvBuf, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM,
    hcclComm, stream);
```

HcclAllReduce接口调用的完整示例可参见7 代码示例章节不同通信域初始化方式中的"HcclAllReduce操作代码样例"。

## 6.4 点对点通信

点对点通信是指多NPU环境下两个NPU之间直接传输数据的通信模式，常用于pipeline并行场景下对激活值的数据收发。

HCCL提供了不同粒度的点对点通信算子，包括单rank到单rank的单发单收算子（Send/Receive），以及多个rank之间的批量收发算子(BatchSendRecv)，HCCL同步提供了对应的接口供开发者调用。

### Send/Receive（单发/单收）

- Send：将一个rank的数据发送到另外一个rank。
- Receive：接收从另一个rank上发送来的数据。

HCCL提供了对应的接口HcclSend、HcclRecv用于单收单发场景，需严格保序下发并配对使用，收发两端需完成同步后才能进行数据收发，数据收发完成后才能执行后续算子任务。

[图: Send/Receive操作示意图 - Process 1中HcclSend_1发送给Process 2的HcclRecv_1，Process 2的HcclSend_2发送给Process 1的HcclRecv_2]

一个简单的代码示例片段如下：

```c
if(rankId == 0){
    uint32_t destRank = 1;
    uint32_t srcRank = 1;
    HcclSend(sendBuf, count, dataType, destRank, hcclComm, stream);
    HcclRecv(recvBuf, count, dataType, srcRank, hcclComm, stream);
}
if(rankId == 1){
    uint32_t srcRank = 0;
    uint32_t destRank = 0;
    HcclRecv(recvBuf, count, dataType, srcRank, hcclComm, stream);
    HcclSend(sendBuf, count, dataType, destRank, hcclComm, stream);
}
```

### BatchSendRecv（批量收发）

HCCL提供了HcclBatchSendRecv接口用于通信域内多个rank之间的数据收发，该接口有两个特征：

- 接口内部会对批量数据收发顺序进行重排，所以不严格要求单次接口调用中批量收发的任务顺序，但需要确保一次接口调用中的数据发送与数据接收操作个数完全匹配。
- 收发过程独立调度执行，收发不相互阻塞，从而实现双工链路并发。

该接口使用时需要注意：单次接口调用下，两个rank之间单向数据流仅支持传递一块内存数据，避免收发过程中混淆多块内存数据的收发地址。

一个简单的代码示例片段如下：

```cpp
HcclSendRecvItem sendRecvInfo[itemNum];
HcclSendRecvType currType;
for (size_t i = 0; i < op_type.size(); ++i) {
    if (op_type[i] == "isend") {
        currType = HcclSendRecvType::HCCL_SEND;
    } else if (op_type[i] == "irecv") {
        currType = HcclSendRecvType::HCCL_RECV;
    }
    sendRecvInfo[i] = HcclSendRecvItem(currType,
                        tensor_ptr_list[i],
                        count_list[i],
                        type_list[i],
                        remote_rank_list[i]
                        );
}
HcclBatchSendRecv(sendRecvInfo, itemNum, hcclComm, stream);
```

---

# 7 代码示例

"CANN/hcomm"开源项目中提供了不同场景下使用HCCL接口实现集合通信功能的代码示例，开发者可根据实际需求选择参考。

## 通信域管理

- 每个进程管理一个NPU设备（基于root节点信息初始化通信域）
- 每个进程管理一个NPU设备（基于rank table初始化通信域）
- 每个线程管理一个NPU设备

## 点对点通信

- HcclSend/HcclRecv（基础收发功能）
- HcclBatchSendRecv（实现Ring环状通信）

## 集合通信

- AllReduce
- Broadcast
- AllGather
- ReduceScatter
- Reduce
- AlltoAll
- AlltoAllV
- AlltoAllVC
- Scatter

## AI框架调用

- PyTorch框架调用
- TensorFlow框架调用

---

# 8 性能分析

## 8.1 分析流程

集群性能会受到昇腾AI处理器类型、网络、通信算法、通信配置等多方面因素影响，对于性能问题，会通过Profiling工具进行性能分析，主要分析流程如下：

1. 采集全量的Profiling数据，详见8.2 性能数据采集。
2. 判断整网性能的瓶颈点，根据通信算子下发、执行的不同阶段做进一步分析和优化，详见8.3 性能数据分析。

本节内容主要关注HCCL相关的Profiling信息识别及常见案例的分析思路，更多的性能调优案例请参考《性能问题通用定位指南》中的"通信问题优化方案"章节，采集到全量的Profiling数据后，参考《MindStudio Insight工具用户指南》对Profiling数据进行分析。

## 8.2 性能数据采集

集合通信是一个通信域内的全局协同行为，只通过一个rank的Profiling数据往往难以分析集合通信的性能问题，因此需要采集到全量rank的Profiling数据才能精确的分析到集合通信的性能瓶颈点所在。当前支持通过两种方式采集性能数据：

- 方式一：参考《性能调优工具用户指南》采集Profiling性能数据。
- 方式二：参考《HCCL性能测试工具用户指南》，使用HCCL TEST进行Profiling数据的采集和性能测试。

参考以下步骤，执行HCCL Test采集性能数据：

```bash
# "1"代表开启Profiling，"0"代表关闭Profiling，默认值为"0"，开启时，执行HCCL Test时采集性能数据
export HCCL_TEST_PROFILING=1
# 指定Profiling数据存放路径，默认为/var/log/npu/profiling
export HCCL_TEST_PROFILING_PATH=/home/profiling
```

若开启HCCL_TEST_PROFILING，HCCL Test工具执行完成后会在HCCL_TEST_PROFILING_PATH指定目录下生成Profiling数据，性能数据的解析请参考《性能调优工具用户指南》的"离线解析"章节。

## 8.3 性能数据分析

### 8.3.1 Profiling数据中通信算子行为分析

#### 8.3.1.1 通信算子下发

通信算子的下发在Profiling数据中的CANN层，如图所示，一个"AscendCL@hcom_allReduce_"则对应一次allreduce算子下发。

[图: Profiling数据中CANN层的通信算子下发示例]

集合通信算子在host侧编排下发，并在device侧异步执行。一般来说，通信算子的下发时间和异步执行时间互相掩盖，可以充分利用device的资源。当通信算子的下发成为瓶颈时，device侧需等待通信算子下发，此时会出现空泡，针对利用率下降的情况，需要优化集合通信算子的下发性能，常见的优化手段包括：

- 由于通信算子的下发是在host侧执行，因此下发的耗时受host侧CPU的调度影响，可通过CPU绑核的方式防止CPU切核带来的性能损耗，提升通信算子的下发性能。
- 通过环境变量切换为AIV模式，执行`export HCCL_OP_EXPANSION_MODE="AIV"`，但需注意AIV模式的支持场景有限，若业务中存在多个通信域并发执行的场景，会出现互相抢核导致死锁等不可预期的行为。

#### 8.3.1.2 通信算子执行

通信算子的执行对应Profiling数据中的Communication(HCCL)层，如下图所示：

[图: Profiling数据中HCCL层的通信算子执行示例，展示Group、Plane0-Plane7等通信流]

- Group：表示一个通信域。
- Plane0-X：表示不同通信流，每个Plane对应一个通信流，HCCL的通信算子编排会通过多流并发来充分利用HCCS物理链路资源。
- hcom_allReduce_xx：表示通信算子的执行流程，在详细信息中可以看到通信算子的耗时、数据量及数据类型等信息。

由于通信算子由多个notify同步任务及memcpy内存拷贝任务编排而成，若需要在Profiling中显示具体的通信任务编排信息，需至少采集level 1级别的Profiling数据。

同步任务：

- Notify Record：同步任务，置notify寄存器为1。
- Notify Wait：同步任务，等待notify寄存器为1，然后将其清0。
- RDMASend：机间Roce同步任务，置对端notify寄存器为1。

同时对于同步任务可以从任务的详细信息中获取到任务的耗时、notify id、本端（src rank）及对端（dst rank）等。

[图: Notify_Wait任务详细信息示例，展示notify_id、src rank、dst rank、transport type等参数]

数据通信任务：

- Memcpy：内存拷贝任务，机内或者片内的内存拷贝。
- Reduce_Inline：内存拷贝任务，数据拷贝的同时完成随路规约计算。
- RDMASend：机间Roce通信任务，对应着机间的内存拷贝任务。

同时对于数据通信任务可以从任务的详细信息中获取到任务的耗时、本端（src rank）及对端（dst rank）、数据量（size）、带宽（bandwidth）等。

[图: Memcpy任务详细信息示例，展示transport type(SDMA)、size(Byte)、link type(HCCS)、bandwidth(GB/s)等参数]

> 说明
> - 在Profiling数据中RDMASend任务会对应同步或者数据通信任务，可通过其数据量分析区分是同步任务还是数据通信任务，同步任务的数据量为固定的4字节，而数据通信任务的数据量以实际通信量为准。
> - 若RDMASend任务为数据通信任务时，其任务执行的耗时并不等于实际的通信耗时，只是将通信任务的WQE下发到QP队列中的耗时，实际的通信耗时可参考数据量和带宽值计算得到或参考其后续紧跟着的下一个notify wait任务的耗时。

### 8.3.2 典型算子行为分析

以Atlas 800T A2双机场景下，AllReduce算子的Profiling数据为例，介绍如何将通信算子的任务编排与Profiling中的task对应。下图为其中一个rank上完整的AllReduce算子执行流程，同时将AllReduce的各个算子执行步骤与Profiling进行对应。

[图: AllReduce算子完整执行流程的Profiling示例]

步骤1：将通信数据从用户输入内存拷贝至HCCL Buffer内存中。

[图: Memcpy步骤的Profiling示例]

步骤2：节点内实现ReduceScatter通信语义，包括notify前同步、ReduceInline内存拷贝、随路运算以及notify尾同步。

[图: ReduceScatter步骤的Profiling示例]

步骤3：节点间实现AllReduce通信语义。由于节点间通过RoCE来实现notify同步及数据的通信，且notify record任务及数据通信任务均已RDMASend下发WQE的形式实现，因此在Profiling中会以RDMASend（notify record）+ notify wait的组合对应着机间前同步和尾同步任务，同时会以RDMASend（数据通信）+ RDMASend（notify record）+ notify wait的组合对应着机间的数据通信。

[图: 节点间AllReduce步骤的Profiling示例，展示RDMASend任务详细信息]

步骤4：节点内实现AllGather通信语义，包括notify前同步、memcpy内存拷贝以及notify尾同步。

[图: AllGather步骤的Profiling示例]

步骤5：将通信数据从HCCL Buffer拷贝到用户输出内存中。

[图: 最终Memcpy步骤的Profiling示例]

### 8.3.3 快慢卡问题分析

集合通信算子在执行内存拷贝任务前，需要与对端进行一次前同步，以保证对端已准备好接收本端的数据，因此若对端还没有执行到同一个通信算子，本端则需要等待对端算子同步后才能往下执行，而这个notify wait的等待时间也会被统计到通信算子执行中，形成了通信算子性能慢的现象，这就是常见的快慢卡现象。

[图: 快慢卡现象的Profiling示例，展示device0和device1的执行时间差异]

如上图中的示例，device1先执行到AllReduce算子，但是device0的AllReduce算子执行到的时间更慢，因此device1上的AllReduce算子会有一段notify wait的时间被统计到算子的耗时中，此时需要进一步去排查卡间算子的执行有快慢卡的原因，常见的快慢卡现象的原因有：

- 前置计算算子性能波动。
- 通信算子下发瓶颈，如host侧其他行为导致慢卡的通信算子没能及时下发。

### 8.3.4 交换机存在流量拥塞反压或丢包重传

若在Profiling数据中观察到有长达4秒左右的notify wait任务，一般对应的网络配置出现问题，导致出现了丢包重传的现象，可以通过hccn_tool工具查看统计值中的roce_new_pkt_rty_num字段来定位。

若在任务执行过程中该字段统计值增加，则说明了网络出现了丢包重传现象，此时需进一步排查交换机的配置。

执行如下指令查看统计值：

```bash
hccn_tool -i {DeviceId} -stat -g
```

---

# 9 故障诊断

## 9.1 定位思路

### 9.1.1 应知应会

在故障定位之前，请确保您已熟悉HCCL相关基本概念及故障定位辅助功能。

对于HCCL来说，故障码会涵盖大部分常见问题，如果报错中未包含故障码信息，或故障码信息为EI9999，可能为较为少见的故障场景或HCCL内部问题，请基于实际的CANN日志和代码进行分析，如果无法解决请联系技术支持。

对于没有清晰首报错的问题，大集群故障定位时，需要梳理每个rank的行为，通过rank之间的依赖关系找到根节点。面对这个难题，HCCL提供了建链根节点定位能力和集群心跳能力，并会在常见问题中给出诊断结果，相关原理请参见9.3.1 建链失败定位思路、9.4.2 集群心跳机制。

故障诊断相关环境变量：

- HCCL_CONNECT_TIMEOUT、HCCL_EXEC_TIMEOUT：HCCL在建链阶段和执行阶段的超时时间，建议HCCL_CONNECT_TIMEOUT配置的时间小于HCCL_EXEC_TIMEOUT配置的时间，以保证复杂场景下能够正确的上报首报错信息，以区分异常业务进程被阻塞的原因是本端还是远端。
- HCCL_ENTRY_LOG_ENABLE：HCCL算子级入参记录开关，如果集群行为一致性问题无法通过其他手段锁定异常原因时，可以使能此环境变量，记录不同rank上的集合通信行为，通过卡间横向比对辅助找到行为差异引入点。
- HCCL_DEBUG_CONFIG：HCCL模块级日志开关，进行算子开发调试时可以通过此配置分析算子内部的算法选择、任务编排等日志信息。
- HCCL_DFS_CONFIG：HCCL高级故障探测配置能力，详见环境变量说明，建议使用默认值。

### HCCL相关日志说明

HCCL的日志信息会记录在CANN日志中，CANN的相关日志说明请参考《日志参考》。

- 当HCCL报错时会在CANN日志的debug目录下打印关键的故障信息；同时在使用部分训练框架的业务场景下，HCCL也会在业务的日志中打印关键的报错信息。
- HCCL在CANN日志的run目录下会默认记录一些关键运行日志，如通信域的初始化与析构（默认打印）、通信算子的下发（需开启HCCL_ENTRY_LOG_ENABLE环境变量）等，关键日志示例如下：

通信域初始化：
```
Entry-HcclGetRootInfo:rootInfo[0x7fffcd65f130], deviceLogicId[0]
Entry-HcclCommInitRootInfoConfigInner:ranks[16], rank[0], rootinfo: host ip[127.10.0.1] port[60000]
nicDeploy[1] identifier[group_name_0], deviceLogicId[0]
```
- ranks：通信域大小。
- rank：当前rank在通信域内的rank编号。
- rootinfo：root节点的信息。
- identifier：通信域名。

通信域析构：
```
Entry-HcclCommDestroy: op_base comm destroy begin
```

通信算子下发：
```
Entry-HcclAllReduce: tag[AllReduce_127.10.0.1%eth1_30000_0_1736576907435382],
sendBuf[0x12e7bf550000], recvBuf[0x12e7bf550000], count[531260224], dataType[float32], op[sum],
localRank[0], streamId[5],comm[0x331c9c00], deviceLogicId[0]
```
- tag：通信算子标识符。
- sendBuf：输入数据地址指针。
- recvBuf：输出数据地址指针。
- count：数据量。
- dataType：数据类型。
- op：reduce计算类型。
- localRank：本端rank号。
- stream：通信算子执行流。
- comm：通信域指正。
- deviceLogicid：通信算子下发的设备逻辑Id。

### 9.1.2 快速定位定界思路

1. 确认是否为HCCL相关的异常报错。
   - HCCL针对常见的报错场景，会在业务打屏日志中上报错误信息及故障信息，若在业务日志中存在"EI\*\*\*\*"或"EJ\*\*\*\*"的故障码，则可根据对应的故障信息排查故障，或结合CANN日志中的报错信息对对应章节进行排查，故障码列表可见HCCL相关故障码。
   - 除了打屏的故障码信息，HCCL在CANN日志中会打印HCCL组件的ERROR级别日志，因此若在CANN日志中没有发现HCCL组件的报错日志，需排查是否有其他组件的报错信息，若无报错，请注意训练脚本本身有无异常、是否存在core dump或进程卡住等其他异常。

2. 收集全量CANN日志。

3. 确认当前报错阶段，并根据不同阶段进行排查。
   HCCL业务存在三个阶段，分别是通信域初始化、参数面建链和通信算子执行，由于不同阶段使用的硬件资源、通信拓扑和同步方式有明显差异，因此可先确认当前HCCL报错所在的阶段，再根据不同的阶段找到对应的章节做进一步排查。
   - 若业务在调用通信域创建接口失败时，或在报错日志中有"topoinfo"、"ranktable"关键字打印，可参考9.2 通信域初始化阶段章节进一步排查。
   - 若业务在调用通信算子接口失败时，或在报错日志中有"transport"关键字打印，可参考9.3 参数面建链阶段章节进一步排查。
   - 若业务创建通信域接口和通信算子下发均成功，而是在触发流同步时有HCCL的算子执行失败，或在报错日志中有"TaskExceptionHandler"、"FFTS+ run failed"、"Task run failed"关键字打印，可参考9.4 任务下发执行阶段章节做进一步排查。

### HCCL相关故障码

| 故障码 | 故障码说明 |
|---|---|
| EI0001 | 环境变量配置异常 |
| EI0002 | 通信算子执行超时 |
| EI0004 | rankTable校验失败 |
| EI0005 | 参数一致性校验失败 |
| EI0006 | 通信域集群信息协商阶段超时或通信算子参数面建链超时 |
| EI0011 | QP内存资源申请失败 |
| EI0012 | 算子执行时发生SDMA任务异常 |
| EI0013 | 算子执行时发生ROCE CQE ERROR异常 |
| EI0014 | rankTable内容校验失败 |
| EI0015 | 通信域集群信息协商阶段超时 |
| EJ0003 | 通信域创建阶段server节点端口绑定失败或参数面建链阶段端口绑定失败 |

## 9.2 通信域初始化阶段

### 9.2.1 总体流程

[图: 通信域初始化阶段流程 - 环境变量初始化 -> 资源申请与初始化 -> 基于rank table创建通信域 / 基于root节点信息创建通信域 -> 集群信息校验]

HCCL在通信域初始化阶段会经历上图所示的多个流程，首先进行环境变量初始化和资源初始化，接着获取整个通信域的集群信息，其过程一般有两种方式：

- 基于rank table文件：通过其他途径生成rank table（集群信息配置文件），调用HCCL创建通信域的接口读取对应文件，这种方式对于rank table本身的格式要求请参照10 集群信息配置。
- 基于root节点信息创建通信域：又称为集群协商方式创建通信域，通过HCCL提供的通信域创建接口基于Host侧网卡向root节点建立socket连接，从而进行信息的汇聚和分发，以此生成集群信息。

### 9.2.2 环境变量配置异常（EI0001）

#### 9.2.2.1 定位思路

在业务的日志中出现"EI0001"故障码意味着HCCL的环境变量配置异常，一般情况下打印日志的ERROR MESSAGE和CANN日志中会显示配置异常的环境变量名称及错误原因，以及合理的配置范围，如果有疑问请参照11 环境变量参考。

#### 9.2.2.2 HCCL_RDMA_SL配置错误

问题现象：在打印日志中存在关键字"EI0001"或"Environment variable [\*\*\*] is invalid."。

可能的原因及解决方法：环境变量配置参数不符合要求，请基于日志打印的建议调整取值范围，如果仍然有疑问，请参照对应11 环境变量参考。

#### 9.2.2.3 HCCL_SOCKET_IFNAME配置错误

问题现象：在CANN日志中存在关键字"get host ip fail by socket Ifname"。

问题根因：通过HCCL_SOCKET_IFNAME环境变量指定了Host网卡，但在当前的环境上没有找到对应的网卡（若为容器场景需指定容器内可用的Host网卡），报错日志打印列举了当前环境上查询到的Host网卡。

解决方法：修改HCCL_SOCKET_IFNAME环境变量，指定为环境上存在的Host网卡。

### 9.2.3 集群信息协商相关(EI0006)

#### 9.2.3.1 业务流程及定位思路

HCCL在集群信息协商时（基于root节点信息创建通信域的场景），通过server节点和每个rank节点之间建立socket连接，互相交换本端信息的方式来获取整个通信域集群信息完成通信域初始化。

[图: 集群信息协商业务流程 - 步骤1: HcclGetRootInfo获取Host网卡IP及端口信息、生成rootInfo、绑定和监听端口 -> 步骤2: HcclCommInitRootInfo向server节点建立连接 -> 步骤3: server节点收集全量集群信息并发送给每个rank]

集群信息协商阶段失败的常见原因及关键日志：

- server节点的端口绑定失败，可通过以下命令排查是否有端口绑定失败问题，详细信息可参考9.2.3.2 server节点端口绑定失败（EJ0003）。
  ```bash
  grep -r "socket type\[2\].*Please check the port status and whether the port is being used by other process"
  ```
- server节点未收到通信域下全部rank的socket连接，可通过以下命令快捷查询，详细信息可参考9.2.3.3 部分rank未连接到server节点。
  ```bash
  grep -r "topo exchange server get socket timeout"
  ```
- rank节点与server节点建立socket超时，可通过以下命令快捷查询，详细信息可参考9.2.3.4 rank与server节点建立socket超时。
  ```bash
  grep -r "topo exchange agent get socket timeout"
  ```

#### 9.2.3.2 server节点端口绑定失败（EJ0003）

问题现象：打印日志中会有EJ0003的报错。

问题根因：HCCL端口绑定失败，在通信域创建阶段，HCCL需要默认绑定60000-60015端口，若此时该端口已被绑定，则HCCL会绑定端口失败从而导致通信域创建失败。

解决方案：
- 通过HCCL_IF_BASE_PORT环境变量指定Host网卡的起始端口号及设置端口预留范围。
- 若业务上需要在单个npu上同时执行多个进程，则需通过HCCL_HOST_SOCKET_PORT_RANGE设置HCCL在Host侧使用的通信端口范围来避免多进程之间端口使用冲突。

#### 9.2.3.3 部分rank未连接到server节点

问题现象：在CANN日志中存在关键字"topo exchange server get socket timeout!"或"Failed to connect agent"。

问题根因：server节点在调用HcclGetRootInfo接口后会拉起一个背景线程等待所有的rank来连接，直到超时时间为止。因此若在超时时间内，通信域的所有rank没有成功连接到server线程，server线程会等待超时报错。

定位思路：从报错信息中确认未连接的rank，确认未连接rank是否有下发通信域创建接口，若有则查看该rank的报错信息，进一步排查。

#### 9.2.3.4 rank与server节点建立socket超时

问题现象：在CANN日志中存在关键字"topo exchange agent get socket timeout!"。

问题根因：在通过集群协商创建通信域阶段，当前rank会根据rootInfo信息与server节点的ip和port创建socket，但由于网络连通性问题导致socket建立超时。

定位思路：
1. 检查host侧网络和对应端口的联通性。
2. 检查通信域内每个agent与server节点的下发通信域创建时间间隔是否超过超时时间，可通过HCCL_CONNECT_TIMEOUT配置建链超时时间，默认为120秒。

#### 9.2.3.5 典型多机场景通信域初始化失败

当集群发生通信域创建建链超时时，无论是server节点还是已成功连接的节点，都可以从报错日志中快速确认未连接的rank，此时仅需可以重点排查未连接rank的失败原因即可，如常见的连接超时原因为未配置HCCL_SOCKET_IFNAME环境变量导致使用未连通的Host网卡。

### 9.2.4 rank table校验问题(EI0004、EI0014)

#### 9.2.4.1 定位思路

HCCL会对rank table文件或协商收集到的rank table信息进行校验，若校验失败HCCL会直接报错退出，请基于实际报错内容进行定位。

可能的原因有：rank table文件读取/校验失败、内容与硬件配置不符合、TLS配置不一致或者superDeviceId重复。

#### 9.2.4.2 rank table文件读取失败

问题现象：在CANN日志中存在关键字"is not a valid real path"。

问题根因：基于rank table文件初始化通信域时，传入的rank table文件路径不存在或者权限不足。

解决方法：修改正确的rank table文件路径或者配置正确的可读权限。

#### 9.2.4.3 rank table字段配置错误

问题现象：在CANN日志中存在关键字"JsonArrayMemberProperty"。

问题根因：rank table的"version"字段为"1.2"，对应于Atlas A3 训练系列产品/Atlas A3 推理系列产品的rank table文件，但是rank table里没有填写"super_pod_id"字段，导致rank table校验失败。

解决方法：在rankTable文件中补充"super_pod_id"字段。

#### 9.2.4.4 rank table文件中device_ip字段校验失败

问题现象：在CANN日志中存在关键字"the IP address(\*\*\*) in the ranktable is inconsistent with the IP(\*\*\*)address of the network adapter"。

问题根因：HCCL在校验device ip时发现当前device侧获取的device ip与rank table中给当前rank配置的device ip不一致，因此校验失败。

解决方法：需检查rank table的配置与通信域中每个rank实际执行的device ip是否一致。

#### 9.2.4.5 IP Family校验不一致

问题现象：在CANN日志中存在关键字"rank[\*] device ip family[2] is not same with others[\*]."。

问题根因：两个rank获取到的IP Family不同，比如一边是IPv4，而另一边是IPv6。

解决方法：
- 查询是否配置了IPv4：`hccn_tool -i {deviceId} -ip -g`
- 查询是否配置了IPv6：`hccn_tool -i {deviceId} -ip -inet6 -g`
- 同一次作业的所有rank的IP Family应保持一致。HCCL默认先使用IPv4协议，若Device侧没有配置IPv4协议的IP，则会使用IPv6协议对应的ip。

#### 9.2.4.6 TLS信息配置不一致

问题现象：在CANN日志中存在关键字"Some ranks tlsStatus are inconsistent."。

原因分析：通信域创建过程中server节点收到通信域内所有rank的信息后，会校验通信域内所有rank的tls配置是否一致，若存在配置不一致场景，则会直接校验失败退出。

解决方法：
1. 查询集合通信的各服务器TLS状态开关：`hccn_tool -i <device_id> -tls -g`
2. 判断各服务器中所有Device的TLS状态开关是否一致。若不一致，建议统一修改TLS状态为使能。
3. 查看所有服务器的各Device的TLS证书信息是否一致。若不一致，可以通过如下命令替换证书套件：
   `hccn_tool -i 0 -tls -s path /root pri pri.pem pub pub.pem ca1 ca1.pem ca2 ca2.pem crl xxx.crl`

#### 9.2.4.7 superDeviceId重复

问题现象：在CANN日志中存在关键字"superDeviceId[\*\*\*] in superPod[\*\*\*]is already exist"。

问题根因：superDeviceId是Atlas A3 训练系列产品/Atlas A3 推理系列产品内device的唯一标识，HCCL在一致性校验时发现一个超节点内有相同的superDeviceId，因此校验失败。

解决方法：修改硬件配置或正确配置HCCL_LOGIC_SUPERPOD_ID环境变量，避免同一个超节点内出现SDID相同的设备。

## 9.3 参数面建链阶段

### 9.3.1 建链失败定位思路

在调用通信算子时，HCCL会通过参数面网络基于TCP协议进行socket链接创建，以此来基于业务需要进行地址等信息交换，此时如果出现某种故障导致部分rank未调用到预期的通信算子，导致无法发起建链请求，或由于网络连通性、行为一致性问题导致无法响应彼此之间的建链请求，就会导致其他rank出现socket连接超时报错。

建链根节点定位机制：HCCL会在业务上建链失败后立即启动故障探测链路，其主要的实现原理为：
1. 每个rank在建链失败后会启动监听能够响应所有rank故障探测链路的server端。
2. 向无法响应自己业务建链请求的远端发起故障探测链路connect请求。
3. 如果远端无法响应自己的探测建链请求，则认为和远端的链路，或远端的业务进程存在问题，产生探测失败事件。并向已经在server端建立成功的其他链路发送扩散该事件。
4. 如果远端建立起了探测链路，则接收对端发送的探测失败事件并进行转发。

一致性校验机制：HCCL在与对端成功创建socket链接后，会互相交换算子入参、CANN版本等信息并与本端的信息做校验，如果此时校验结果存在不一致的情况，则会在CANN日志及打屏日志中上报错误并返回错误码。详细问题定位流程可参考9.3.5 参数一致性校验(EI0005)。

报错阶段分析：HCCL在通信算子参数面建链阶段会有以下几个常见的报错阶段场景，对应的关键日志信息如下：
- device网卡端口绑定失败，可参考9.3.2 参数面端口绑定失败（EJ0003）。
- 参数面socket建链超时，可参考9.3.4 建链超时(EI0006)。
- 通信算子一致性校验失败，可参考9.3.5 参数一致性校验(EI0005)。

### 9.3.2 参数面端口绑定失败（EJ0003）

问题现象：在CANN日志中存在关键字"Please check the port status and whether the port is being used by other process."。

问题根因：当前rank或进程在通信算子参数面建链时需要绑定一个device侧网卡的端口，但发现端口已被其他进程占用。

解决方法：HCCL使用device侧网卡的端口时默认需绑定16666端口，因此若有多个进程执行在同一个device上，且均会调用HCCL的通信算子接口，那么就会出现端口已被其他进程绑定导致失败的问题。此时可先从业务上排查多个进程跑在同一个device上是否符合任务预期，若符合任务预期结果，可通过配置HCCL_NPU_SOCKET_PORT_RANGE环境变量使能多进程场景：
```bash
export HCCL_NPU_SOCKET_PORT_RANGE="auto"
```

### 9.3.3 QP内存资源申请相关(EI0011)

问题现象：在打屏日志中存在关键字"EI0011"或"Resource_Error_Insufficient_Device_Memory"。

解决方法：调整业务配置（如batchSize）、减少ROCE链路的使用数量，或释放部分内存解决问题。HCCL的其他内存申请如cclBuffer内存申请若出现oom报错，会由drv组件上报错误码及打屏的ERROR MESSAGE，若为HCCL的cclBuffer内存申请失败，可通过配置HCCL_BUFFSIZE环境变量调整申请的内存大小。

### 9.3.4 建链超时(EI0006)

HCCL建链超时受HCCL_CONNECT_TIMEOUT环境变量配置影响，若在超时时间内对端无法响应业务建链请求，则会上报"socket timeout"，同时如果远端由于超时等故障退出，已经建好的链路在等待数据交换的过程中也可能会伴随"recv fail"的报错。

问题现象：在CANN日志中存在关键字"wait socket establish timeout"。

根据日志确认需排查的建链对端：
- 若报错日志中有"DETECT EVENT LIST"打印，可先重点关注日志中失败的建链对，如"DETECT EVENT[1]"异常事件显示的两个节点之间的建链失败根因。
- 若报错日志中无"DETECT EVENT LIST"打印，那么可从报错日志的"LINK_ERROR_INFO"表格中获取建链两端的device ip，同时可从"Transport init error! createLink para:"关键日志信息中获取本端和对端的节点信息。

确认对端行为排查是否有卡间行为不一致：参数面建链是一个两端的互动流程，需要两端在超时时间内均发起建链请求才能创建成功，可以根据本端的报错信息中找到对端的节点信息，查看对端的日志做进一步的判断。排查点包括：
1. 若对端没有任何报错日志，需从业务上排查两端的通信算子下发行为是否一致。
2. 若对端发生了除参数面建链超时外的其他报错，则需要先排查对端的报错原因。
3. 若对端也发生了参数面建链超时报错，但对端的报错信息中并不在和本端建链，则需要按照流程先排查对端的参数面建链超时原因。
4. 若对端也在和本端参数面建链超时，可先排查两端的报错时间是否超过了建链等待时间，如超过了建链超时时间，需要业务上排查两端通信算子下超时时间的根因。建链等待时间可通过HCCL_CONNECT_TIMEOUT指定，默认为120秒。
5. 若对端和本端的参数面建链超时在建链超时时间内，则需要进一步排查两端的网络连通性：
   - 排查两端的tls开关是否一致。
   - 若建链的两端在不同的节点上，则需要检查本端和对端的device网口之间的网络连通性，使用hccn_tool命令在其中一个节点ping另一个节点的device ip。

### 9.3.5 参数一致性校验(EI0005)

问题现象：在打屏日志中存在关键字"The arguments for collective communication are inconsistent between ranks"，或在CANN日志中存在关键字"CMD information \*\*\* check fail"。

问题根因：参数面建链时，在socket建立完成后会进行两端的参数一致性校验，校验的范围包括算子标识符tag、算子类型opType、数据量dataSize、HCCL Buffer的大小cclbufferSize、数据类型datatype等，可根据报错里的信息确定不一致的数据。

解决方法：需根据报错信息从业务上排查参数校验不一致的两端下发的算子不一致的根因。

## 9.4 任务下发执行阶段

### 9.4.1 定位思路（EI0002）

完成通信域初始化及参数面建链后，HCCL会进行通信算子的任务编排及下发，在通信算子的任务编排上，进行数据通信前会有notify同步机制来确保对端已准备好接收本端的数据，因此如果有rank由于某种异常导致进程卡死或退出、网络故障或调用的通信算子不一致，会导致大部分rank出现执行等待超时。遇到此类问题，定位的首要条件是找到故障点位置，整体定位思路如下图。

[图: 任务下发执行阶段报错定位思路 - 打开任意notify超时异常rank的plog日志 -> 检查是否有"base info"报错关键字 -> 是则基于ERROR日志(检查Cluster Exception Location关键字) / 否则基于run目录日志(判定业务卡住时间、查找心跳时间) -> 存在异常事件则分析心跳事件 / 不存在则为集群行为一致性问题]

HCCL DFX机制说明：HCCL在通信算子任务下发执行阶段提供了以下DFX机制来辅助问题快速定位：

- HCCL存在集群心跳机制，当某个rank节点发现异常时，会通过心跳机制扩散到集群的每个节点上，因此可以先在集群中的任意节点的CANN日志中检索是否有心跳的异常事件信息打印，机制说明及日志信息可以参考9.4.2 集群心跳机制。
- 若未检索到心跳的异常事件信息日志打印，可通过task exception报错信息排查是否有集群行为不一致问题，排查方法可参考9.4.3 task exception机制。

### 9.4.2 集群心跳机制

#### 9.4.2.1 定位思路

HCCL会基于已有的通信域信息与邻近的rank建立起独立的维测链路，以此提供集群的单点故障广播扩散能力（注：HCCL会控制维测链路数量和通信数据量，用户不用担心其对通信链路造成的性能损失），使任意rank的plog日志中均包含故障根节点信息。当前支持的故障探测能力参见下表。

| 异常类型 | 异常打印 | 判断标准 |
|---|---|---|
| 进程退出 | Heartbeat Lost Occurred | 30s时间内未收到远端的心跳报文 |
| 进程卡死 | Stuck Occurred | 每隔1/3的HCCL_EXEC_TIMEOUT时间轮询所有算子的入/出次数分析是否卡住 |
| 网络问题 | Error cqe Occurred | 定期轮询ROCE驱动重传超次事件，通过QPN映射远端IP |

HCCL在检测到异常事件后，会在集群中进行信息的扩散转发，HCCL在运行过程中会将收到的异常事件打印到run日志，其打印格式如下：

```
[INFO] HCCL(686,python):2025-10-23-07:52:59.191.363 [heartbeat.cc:951] [8970][HCCL_TRACE]rank
[127.10.0.1/1]: rank [127.10.0.1/9] status[LOST] by rank [127.10.0.1/1]
```

如果后续出现算子执行报错，并且调用了task exception回调函数通知了HCCL，则HCCL会根据已经收到的异常事件，结合HCCL_EXEC_TIMEOUT等超时事件配置，推测出最可能的单点故障原因，打印在ERROR日志中，如：

```
[ERROR]HCCL(835695,all_reduce_test):2025-10-23-17:28:06.049.385[task_exception_handler.cc:610]
[835695]Cluster Exception Location[IP/ID]:[127.10.0.1/1], Arrival Time:[Thu Oct 23 17:25:58 2025],
Discoverer:[127.10.0.1/2], ExceptionType:[Heartbeat Lost Occurred], Possible Reason:1. Process has exited, 2.
Network Disconnected
```

#### 9.4.2.2 示例：进程卡死或对端心跳丢失

问题现象：在CANN日志中存在关键字"Cluster Exception Location"。

问题根因及定位思路：可从报错日志中识别异常类型及异常所在的节点信息：
- Cluster Exception Location：表示异常所在的节点信息。
- Arrival Time：表示异常广播到本端的时间。
- ExceptionType：异常类型，包括心跳丢失（Heartbeat Lost Occurred）、进程卡死（Stuck Occurred）、网络丢包（Error cqe Occurred）等。
- Possible Reason：异常可能的发生原因及排查思路。

### 9.4.3 task exception机制

#### 9.4.3.1 定位思路

HCCL通信算子的任务编排完成后会下发到Device侧上异步执行，若此时HCCL下发的任务执行失败，会通过调用回调函数通知HCCL异常task信息（stream和taskId），HCCL会以此检索下发时的task信息，打印失败task的详细信息及其所在的算子信息。

此时在CANN日志中打印的task exception的关键日志为"FFTS+ run failed"或"TaskExceptionHandler"。

首先可以从task exception信息中识别出通信算子关键信息：
- base information：HCCL算子所在的stream、taskId，以及算子的tag，可根据tag识别当前报错的HCCL算子。
- groupRank information：通信域名（group）、通信域的大小（rankSize）以及当前卡在通信域内的rankId。
- opData information：当前算子的入参信息，所在的deviceId，该通信域下的第几个算子（index）、数据量（count）、reduce类型（reduceType）以及输入（src）和输出（dst）的地址。

#### 9.4.3.2 notify wait超时

由于集合通信是一个通信域内的全局协同行为，若通信域内rank之间下发的通信算子、数据量等不一致，则会由于rank之间的任务不匹配导致执行超时，或者其中一个rank发生了其他报错，则其他rank则会等待报错rank超时从而失败。整体的定位思路如下：

1. 首先需要确认通信域内所有rank所在的节点进程。
2. 若通信域内某个rank存在其他报错，需要先排查对应rank报错的原因。
3. 若通信域内下发的算子、数据量等一致，可排查通信域内rank之间的报错时间间隔是否超过了HCCL_EXEC_TIMEOUT配置的超时时间，默认为1836秒。

确认通信算子下发行为是否异常：若遇到较难排查的算子执行报错问题，可开启"HCCL_ENTRY_LOG_ENABLE"环境变量，该环境变量使用后会在每次通信算子下发后，在log/run/plog目录下的日志文件中打印一次日志记录通信算子下发的入参信息，用例执行失败后便可排查每个rank上下发的通信算子是否存在异常。

#### 9.4.3.3 SDMA ERROR（EI0012）

问题现象：在CANN日志中存在关键字"fftsplus sdma error"。

问题根因：在执行SDMA内存拷贝任务时发生了页表转换失败，也就是内存拷贝的输入或者输出地址未分配内存、分配的内存小于内存拷贝大小或者分配的内存已被释放。

常见的问题根因有以下场景：
- 下发通信算子后，未执行流同步确认通信算子执行完成就析构通信域，由于通信域析构时会释放集合通信的HCCL Buffer地址，因此会导致SDMA内存拷贝时页表转换失败。
- Atlas A3 训练系列产品/Atlas A3 推理系列产品下，网络链路故障也会导致SDMA ERROR，此时需要检查两端之间的链路状态。
- 调用HCCL通信算子时，传入的输入或输出的地址实际分配的内存大小与传入的数据量Count不一致。

#### 9.4.4 ERROR CQE报错（EI0013）

ERROR CQE在HCCL中代表RoCE报文的重传超次，出现后必然会伴随集群卡死导致超时。HCCL会定期轮询RoCE驱动以获取其事件，用户可以通过接口HcclGetCommAsyncError进行查询是否有发生ERROR CQE报错。

问题现象：在CANN日志中存在关键字"error cqe status"。

问题根因：发生ERROR CQE的问题根因在于本端给对端发包后在指定的时间段内没有收到对端的确认回复，本端就会有ERROR CQE报错上报。

排查思路：
1. 排查是否有网络问题，可通过hccn_tool工具查询是否有网口闪断记录。
2. 排查对端的业务进程在本端发生ERROR CQE时是否先异常退出或已进入资源销毁流程。
3. 若环境变量配置的HCCL_RDMA_TIMEOUT重传超时时间及HCCL_RDMA_RETRY_CNT重传次数比较小，在链路状态不佳时容易出现ERROR CQE错误，将环境变量调大即可。

#### 9.4.5 aiv通信算子执行失败

问题现象：在通过`export HCCL_OP_EXPANSION_MODE="AIV"`使能aiv模式之后，HCCL会以kernel的执行方式实现HCCL通信算子的编排及执行，此时若通信算子执行异常，在日志中会有一行如下关键日志打印"fault kernel_name=aiv_all_reduce_910b_bfloat16_t"表明当前为HCCL的aiv算子执行失败。

此外也会有上述同样的task exception信息打印，仍可以通过9.4.3.2 notify wait超时排查思路分析任务失败的根因。

---

# 10 集群信息配置

HCCL支持通过rank table文件或环境变量配置集群资源信息，不同产品类型对应不同的rank table模板版本和字段要求。

## 10.1 rank table配置资源信息（Atlas A3训练系列产品/Atlas A3推理系列产品）

针对Atlas A3训练系列产品/Atlas A3推理系列产品，rank table模板版本为"1.2"。

> 须知：本节所示JSON文件示例中的注释仅为方便理解，实际使用时请删除JSON文件中的注释。

### 配置示例

以包含两个AI Server，每个AI Server内2个Device为例，rank table文件配置示例如下：

```json
{
  "status":"completed",
  "version":"1.2",
  "server_count":"2",
  "server_list": [
    {
      "server_id":"node_0",
      "host_ip":"172.16.0.110",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.1.8",
          "super_device_id":"0",
          "rank_id":"0"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.1.9",
          "super_device_id":"1",
          "rank_id":"1"
        }
      ]
    },
    {
      "server_id":"node_1",
      "host_ip":"172.16.0.111",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.2.8",
          "super_device_id":"2",
          "rank_id":"2"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.2.9",
          "super_device_id":"3",
          "rank_id":"3"
        }
      ]
    }
  ],
  "super_pod_list": [
    {
      "super_pod_id":"0",
      "server_list": [
        { "server_id":"node_0" },
        { "server_id":"node_1" }
      ]
    }
  ]
}
```

### 表10-1 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 配置说明 |
|---|---|---|---|
| status | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| version | | | 必选。rank table模板版本信息，配置为：1.2。 |
| server_count | | | 必选。参与集合通信的AI Server个数。 |
| server_list | | | 必选。参与集合通信的AI Server列表。 |
| | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。配置示例：node_0。 |
| | host_ip | | 可选。AI Server的Host IP地址，要求为常规IPv4格式。开启HCCL重执行特性的场景下，此字段必须配置，否则重执行失败，走无重执行流程。Server间通信task重执行默认开启，可参见环境变量HCCL_OP_RETRY_ENABLE。 |
| | device | | 必选。Device列表。 |
| | | device_id | 必选。昇腾AI处理器的物理ID，即Device在AI Server上的序列号。可通过执行`ls /dev/davinci*`命令获取。取值范围：[0, 实际Device数量-1]。"device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 |
| | | device_ip | 可选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。当组网中包含多个超节点时，device_ip必须配置。若组网仅包含一个超节点，device_ip在超节点内使用RDMA通信时（即HCCL_INTER_HCCS_DISABLE配置为TRUE，禁用了HCCS功能的场景）必须配置。可通过`cat /etc/hccn.conf`获取网卡IP。 |
| | | device_port | 可选。Device网卡的通信端口，取值范围为[1,65535]，需要确保指定的端口未被其他进程占用。[1,1023]为系统保留端口，应避免使用。单卡多进程的业务场景下，建议配置此字段，并且不同的业务进程需要设置不同的端口号。 |
| | | backup_device_ip | 可选。当超节点间通信开启了算子重执行特性时，如果Device网卡故障（RDMA链路故障），可通过此参数将同一NPU中的另一个Die网卡指定为备用Device网卡，提升算子重执行的成功率，此种启用备用Device网卡的通信方式称之为借轨通信。"backup_device_ip"为常规IPv4或IPv6格式。 |
| | | backup_device_port | 可选。备用Device网卡的通信端口，取值范围为[1,65535]。同一个Device网卡作为主网卡与备网卡时配置的通信端口号不能相同。 |
| | | host_port | 可选。Host网卡的通信端口，取值范围为[1,65535]，同一AI Server内每个Device对应的host_port应各不相同。若通过环境变量HCCL_OP_RETRY_ENABLE开启了HCCL重执行特性，且业务为单卡多进程场景，建议配置此字段，并且不同的业务进程需要设置不同的端口号。 |
| | | rank_id | 必选。rank唯一标识，请配置为整数，从0开始配置，且全局唯一，取值范围：[0, 总Device数量-1]。建议rank_id按照Device物理连接顺序进行排序，即将物理连接上较近的Device编排在一起，否则可能会对性能造成影响。 |
| | | super_device_id | 可选（若不配置该字段，会采用"AI Server模式"）。昇腾AI处理器在超节点系统中的物理ID，为超节点系统中NPU唯一标识。可通过npu-smi命令查询：`npu-smi info -t spod-info -i id -c chip_id`。回显中的"SDID"即为超节点系统中的NPU唯一标识。 |
| super_pod_list | | | 可选（若不配置该字段，会采用"AI Server模式"）。参与集合通信的超节点列表。 |
| | super_pod_id | | 若配置了"super_pod_list"，该字段必选。超节点唯一标识，全局唯一，支持配置为超节点物理ID或用户自定义编号。 |
| | server_list | | 必选。超节点内的AI Server列表。 |
| | | server_id | 必选。Server标识，字符串类型，与"server_list"中的server_id对应。配置示例：node_0。 |

> 须知：如果组网中存在多个超节点，请将属于同一超节点内的AI Server信息配置在一起。假设有两个超节点，标识分别为"0"和"1"，请先配置"0"中的AI Server信息，再配置"1"中的AI Server信息，不支持"0"中的AI Server信息与"1"中的AI Server信息交叉配置。

[图: 10-1 借轨通信切换示例，展示NPU0和NPU1之间Die0和Die1的主备切换]

[图: 10-2 同一NPU仅支持一次借轨示例，展示NPU0-NPU3之间的链路故障借轨和二次故障报错退出]

## 10.2 典型集群组网（即AI Server模式）

以包含两个AI Server，每个AI Server内2个Device为例，rank table文件配置示例如下：

```json
{
  "status":"completed",
  "version":"1.0",
  "server_count":"2",
  "server_list": [
    {
      "server_id":"node_0",
      "host_ip":"172.16.0.110",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.1.8",
          "device_port":"16667",
          "host_port":"16666",
          "rank_id":"0"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.1.9",
          "device_port":"16667",
          "host_port":"16667",
          "rank_id":"1"
        }
      ]
    },
    {
      "server_id":"node_1",
      "host_ip":"172.16.0.111",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.2.8",
          "device_port":"16667",
          "host_port":"16666",
          "rank_id":"2"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.2.9",
          "device_port":"16667",
          "host_port":"16667",
          "rank_id":"3"
        }
      ]
    }
  ]
}
```

### 表10-2 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 配置说明 |
|---|---|---|---|
| status | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| version | | | 必选。rank table模板版本信息。针对典型集群组网，配置为：1.0。 |
| server_count | | | 必选。参与集合通信的AI Server个数。 |
| server_list | | | 必选。参与集合通信的AI Server列表。 |
| | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。配置示例：node_0。 |
| | host_ip | | 可选。AI Server的Host IP地址，要求为常规IPv4格式。开启HCCL重执行特性的场景下，此字段必须配置。 |
| | device | | 必选。AI Server中的Device列表。 |
| | | device_id | 必选。昇腾AI处理器的物理ID，即Device在AI Server上的序列号。取值范围：[0, 实际Device数量-1]。 |
| | | device_ip | 可选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。多机场景下device_ip必须配置；单机场景下可不配置。 |
| | | device_port | 可选。Device网卡的通信端口，取值范围为[1,65535]。单卡多进程的业务场景下，建议配置此字段。 |
| | | host_port | 可选。Host网卡的通信端口，取值范围为[1,65535]，同一AI Server内每个Device对应的host_port应各不相同。 |
| | | rank_id | 必选。rank唯一标识，从0开始配置，且全局唯一，取值范围：[0, 总Device数量-1]。建议rank_id按照Device物理连接顺序进行排序。 |

## 10.3 rank table配置资源信息（Atlas A2训练系列产品）

针对Atlas A2训练系列产品，以包含两个AI Server，每个AI Server内2个Device为例，rank table文件配置示例如下：

> 须知：本节所示JSON文件示例中的注释仅为方便理解，实际使用时请删除JSON文件中的注释。

```json
{
  "status":"completed",
  "version":"1.0",
  "server_count":"2",
  "server_list": [
    {
      "server_id":"node_0",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.1.8",
          "device_port":"16667",
          "rank_id":"0"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.1.9",
          "device_port":"16667",
          "rank_id":"1"
        }
      ]
    },
    {
      "server_id":"node_1",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.2.8",
          "device_port":"16667",
          "rank_id":"2"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.2.9",
          "device_port":"16667",
          "rank_id":"3"
        }
      ]
    }
  ]
}
```

### 表10-3 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 配置说明 |
|---|---|---|---|
| status | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| version | | | 必选。rank table模板版本信息。配置为：1.0。 |
| server_count | | | 必选。参与集合通信的AI Server个数。 |
| server_list | | | 必选。参与集合通信的AI Server列表。 |
| | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。 |
| | device | | 必选。AI Server中的Device列表。 |
| | | device_id | 必选。昇腾AI处理器的物理ID，取值范围：[0, 实际Device数量-1]。"device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 |
| | | device_ip | 可选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。多机场景下device_ip必须配置；单机场景下可不配置。 |
| | | device_port | 可选。Device网卡的通信端口，取值范围为[1,65535]。单卡多进程场景下建议配置此字段。 |
| | | rank_id | 必选。rank唯一标识，从0开始配置，全局唯一，取值范围：[0, 总Device数量-1]。建议rank_id按照Device物理连接顺序进行排序。 |

## 10.4 rank table配置资源信息（Atlas训练系列产品）

针对Atlas训练系列产品，在rank table文件中配置参与训练的昇腾AI处理器信息支持两种配置模板，全新场景推荐使用模板一，模板二用于兼容部分已有场景。

> 须知：本节所示JSON文件示例中的注释仅为方便理解，实际使用时请删除JSON文件中的注释。

### 模板一（推荐使用）

以包含两个AI Server，每个AI Server内2个Device为例：

```json
{
  "status":"completed",
  "version":"1.0",
  "server_count":"2",
  "server_list": [
    {
      "server_id":"node_0",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.1.8",
          "rank_id":"0"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.1.9",
          "rank_id":"1"
        }
      ]
    },
    {
      "server_id":"node_1",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.2.8",
          "rank_id":"2"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.2.9",
          "rank_id":"3"
        }
      ]
    }
  ]
}
```

### 表10-4 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 配置说明 |
|---|---|---|---|
| status | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| version | | | 必选。rank table模板版本信息。配置为：1.0。 |
| server_count | | | 必选。参与集合通信的AI Server个数。 |
| server_list | | | 必选。参与集合通信的AI Server列表。 |
| | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。 |
| | device | | 必选。AI Server中的Device列表。 |
| | | device_id | 必选。昇腾AI处理器的物理ID，取值范围：[0, 实际Device数量-1]。"device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 |
| | | device_ip | 必选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。 |
| | | rank_id | 必选。rank唯一标识，从0开始配置，全局唯一，取值范围：[0, 总Device数量-1]。建议rank_id按照Device物理连接顺序进行排序。 |

### 模板二（兼容部分已有场景，新版本不推荐使用）

```json
{
  "status":"completed",
  "group_count":"1",
  "group_list": [
    {
      "group_name":"hccl_world_group",
      "instance_count":"2",
      "device_count":"2",
      "instance_list": [
        {
          "pod_name":"tf-bae41",
          "server_id":"node_0",
          "devices": [
            {
              "device_id":"0",
              "device_ip":"192.168.1.8"
            }
          ]
        },
        {
          "pod_name":"tf-tbdf1",
          "server_id":"node_1",
          "devices": [
            {
              "device_id":"1",
              "device_ip":"192.168.1.9"
            }
          ]
        }
      ]
    }
  ]
}
```

### 表10-5 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 四级配置项 | 配置说明 |
|---|---|---|---|---|
| status | | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| group_count | | | | 必选。用户申请的group数量，建议配置为1。 |
| group_list | | | | 必选。Group列表。 |
| | group_name | | | 可选。Group名称，当group_count为1时，建议配置为hccl_world_group或者空。 |
| | instance_count | | | 必选。和instance_list中pod_name个数保持一致，例如：容器场景下为容器实际数量。 |
| | device_count | | | 必选。group中设备数量。 |
| | instance_list | | | 必选。instance实例信息列表。 |
| | | pod_name | | 必选。用户自定义配置，保持instance_list内全局唯一。 |
| | | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。 |
| | | devices | | 必选。devices信息列表。 |
| | | | device_id | 必选。昇腾AI处理器的物理ID，取值范围：[0, 实际Device数量-1]。 |
| | | | device_ip | 必选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。 |

## 10.5 rank table配置资源信息（Atlas 300I Duo推理卡）

针对Atlas 300I Duo推理卡，以包含两个AI Server，每个AI Server内2个Device为例，rank table文件配置示例如下：

> 须知：本节所示JSON文件示例中的注释仅为方便理解，实际使用时请删除JSON文件中的注释。

```json
{
  "status":"completed",
  "version":"1.0",
  "server_count":"2",
  "server_list": [
    {
      "server_id":"node_0",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.1.8",
          "rank_id":"0"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.1.9",
          "rank_id":"1"
        }
      ]
    },
    {
      "server_id":"node_1",
      "device": [
        {
          "device_id":"0",
          "device_ip":"192.168.2.8",
          "rank_id":"2"
        },
        {
          "device_id":"1",
          "device_ip":"192.168.2.9",
          "rank_id":"3"
        }
      ]
    }
  ]
}
```

### 表10-6 rank table文件说明

| 一级配置项 | 二级配置项 | 三级配置项 | 配置说明 |
|---|---|---|---|
| status | | | 必选。rank table可用标识。completed：可用；initializing：不可用。 |
| version | | | 必选。rank table模板版本信息。配置为：1.0。 |
| server_count | | | 必选。参与集合通信的AI Server个数。 |
| server_list | | | 必选。参与集合通信的AI Server列表。 |
| | server_id | | 必选。AI Server标识，字符串类型，长度小于等于64，请确保全局唯一。 |
| | device | | 必选。AI Server中的Device列表。 |
| | | device_id | 必选。昇腾AI处理器的物理ID，取值范围：[0, 实际Device数量-1]。"device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 |
| | | device_ip | 必选。昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。 |
| | | rank_id | 必选。rank唯一标识，从0开始配置，全局唯一，取值范围：[0, 总Device数量-1]。建议rank_id按照Device物理连接顺序进行排序。 |

## 10.6 环境变量配置资源信息

除了通过rank table文件配置资源信息的方式外，开发者还可以通过本节所述环境变量组合的方式配置资源信息。

环境变量配置资源信息的方式仅适用于TensorFlow框架网络的通信域初始化，仅支持如下型号的产品：

- Atlas A2训练系列产品
- Atlas训练系列产品

### 配置说明

需要在执行训练的每个AI Server节点上分别配置如下环境变量，进行资源信息的配置：

```bash
export CM_CHIEF_IP=192.168.1.1
export CM_CHIEF_PORT=6000
export CM_CHIEF_DEVICE=0
export CM_WORKER_SIZE=8
export CM_WORKER_IP=192.168.0.1
export HCCL_SOCKET_FAMILY=AF_INET
```

- CM_CHIEF_IP：Master节点的Host监听IP，即与其他节点进行通信的IP地址，要求为常规IPv4或IPv6格式。
- CM_CHIEF_PORT：Master节点的监听端口，需要配置为整数，取值范围"0~65520"，请确保端口未被其他进程占用。
- CM_CHIEF_DEVICE：Master节点中统计Server端集群信息的Device逻辑ID，取值范围：[0, Server内的最大Device数量-1]。
- CM_WORKER_SIZE：用于配置组网中参与集群训练的Device总数量，需要配置为整数，取值范围"0~32768"。
- CM_WORKER_IP：用于配置当前节点与Master进行通信时所用的网卡IP，要求为常规IPv4或IPv6格式。
- HCCL_SOCKET_FAMILY：此环境变量可选，用于控制Device侧通信网卡使用的IP协议版本。AF_INET代表使用IPv4协议，AF_INET6代表使用IPv6协议，缺省时优先使用IPv4协议。

> 说明
> - 如果环境变量"HCCL_SOCKET_FAMILY"指定的IP协议与实际获取到的网卡信息不匹配，则以实际环境上的网卡信息为准。
> - 通过以上环境变量的方式配置集群信息时，环境中不能存在环境变量RANK_TABLE_FILE、RANK_ID、RANK_SIZE。
> - 针对Atlas A2训练系列产品，若业务为单卡多进程场景，建议通过环境变量"HCCL_NPU_SOCKET_PORT_RANGE"配置HCCL在NPU侧使用的通信端口，配置示例：`export HCCL_NPU_SOCKET_PORT_RANGE="auto"`

### 配置示例

假设执行分布式训练的AI Server节点数量为2，Device数量为16，每个AI Server节点有8个Device。启动每个Device上的训练进程前，在对应的shell窗口中配置如下环境变量：

- 节点0，此节点为Master节点，负责集群信息管理、资源分配与调度：
  ```bash
  export CM_CHIEF_IP=192.168.1.1
  export CM_CHIEF_PORT=6000
  export CM_CHIEF_DEVICE=0
  export CM_WORKER_SIZE=16
  export CM_WORKER_IP=192.168.1.1
  ```

- 节点1：
  ```bash
  export CM_CHIEF_IP=192.168.1.1
  export CM_CHIEF_PORT=6000
  export CM_CHIEF_DEVICE=0
  export CM_WORKER_SIZE=16
  export CM_WORKER_IP=192.168.2.1
  ```

---

# 11 环境变量参考

本章介绍HCCL支持的环境变量，分为以下几类：

- 11.1 功能相关
- 11.2 性能相关
- 11.3 网络相关
- 11.4 调试相关
- 11.5 可靠性相关
- 11.6 安全相关
- 11.7 参考

## 11.1 功能相关

### 11.1.1 HCCL_CONNECT_TIMEOUT

功能描述：分布式训练或推理场景下，用于限制不同设备之间socket建链过程的超时等待时间。不同设备进程在集合通信初始化之前由于其他因素会导致执行不同步，该环境变量控制设备间的建链超时等待时间，在该配置时间内各设备进程等待其他设备建链同步。

该环境变量需要配置为整数，取值范围[120,7200]，默认值为120，单位秒。

需要注意的是：实际的建链超时等待时间是该环境变量的值加上20秒。例如，如果该环境变量设置为150秒，则实际的超时等待时间为170秒。额外的20秒用于通知各个节点通信域初始化失败的原因。

> 须知：此环境变量的值会影响链路故障场景的异常上报时间。

配置示例：`export HCCL_CONNECT_TIMEOUT=200`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.1.2 HCCL_EXEC_TIMEOUT

功能描述：不同设备进程在分布式训练或推理过程中存在在卡间执行任务不一致的场景（如仅特定进程会保存checkpoint数据），通过该环境变量可控制设备间执行时同步等待的时间，在该配置时间内各设备进程等待其他设备执行通信同步。

- 针对Atlas A3训练系列产品/Atlas A3推理系列产品，单位为s，取值范围为：[0, 2147483647]，默认值为1836，当配置为0时代表永不超时。若算法的编排展开位置设置为了"AIV"（详情请参见11.1.8 HCCL_OP_EXPANSION_MODE），此环境变量的取值范围为[0, 1091]，默认值为1091。
- 针对Atlas A2训练系列产品，单位为s，取值范围为：[0, 2147483647]，默认值为1836，当配置为0时代表永不超时。若算法的编排展开位置设置为了"AIV"，此环境变量的取值范围为[0, 1091]，默认值为1091。
- 针对Atlas训练系列产品，单位为s，取值范围为：(0, 17340]，默认值为1836。需要注意：针对Atlas训练系列产品，系统实际设置的超时时间 = 环境变量的取值先整除"68"，然后再乘以"68"，单位s。如果环境变量的取值小于68，则默认按照68s进行处理。
- 针对Atlas 300I Duo推理卡，单位为s，取值范围为：(0, 17340]，默认值为1836。需要注意：系统实际设置的超时时间 = 环境变量的取值先整除"68"，然后再乘以"68"。

> 说明：一般情况下，用户保持默认值即可。当默认值无法满足设备间执行通信同步的需求时，可通过此环境变量适当增大设备间的同步等待时间。

配置示例：`export HCCL_EXEC_TIMEOUT=1800`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.1.3 HCCL_ALGO

功能描述：此环境变量用于配置集合通信Server间通信算法以及超节点间通信算法，支持全局配置算法类型与按算子配置算法类型两种配置方式。

> 须知：HCCL提供自适应算法选择功能，默认会根据产品形态、数据量和Server个数选择合适的算法，一般情况下用户无需手工指定。若通过此环境变量指定了Server间通信算法，则自适应算法选择功能不再生效。

全局配置算法类型，配置方式如下：

```bash
export HCCL_ALGO="level0:NA;level1:<algo>;level2:<algo>"
```

- level0代表Server内通信算法，当前仅支持配置为NA。
- level1代表Server间通信算法，支持如下取值：
  - ring：基于环结构的通信算法，通信步数多（线性复杂度），时延相对较高，但通信关系简单，受网络拥塞影响较小。适合通信域内Server个数较少、通信数据量较小、网络存在明显拥塞、且pipeline算法不适用的场景。
  - H-D_R：递归二分和倍增算法（Recursive Halving-Doubling: RHD），通信步数少（对数复杂度），时延相对较低，但在非2的整数次幂节点规模下会引入额外的通信量。适合通信域内Server个数是2的整数次幂且pipeline算法不适用的场景，或Server个数不是2的整数次幂但通信数据量较小的场景。
  - NHR：非均衡的层次环算法（Nonuniform Hierarchical Ring），通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且pipeline算法不适用的场景。
  - NHR_V1：对应历史版本的NHR算法，通信步数少（根复杂度），时延相对较低，适合通信域内Server数为非2的整数次幂且pipeline算法不适用的场景。该配置项未来会逐步下线，建议开发者使用NHR算法。
  - NB：非均匀的数据块通信算法（Nonuniform Bruck），通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且pipeline算法不适用的场景。
  - AHC：层次化集合通信算法（Asymmetric Hierarchical Concatenate），适用于通信域内NPU分布存在多个层次、多个层次间NPU对称或者非对称分布（即卡数非对称）的场景，当通信域内层次间存在带宽收敛时相对收益会更好。注意：当level1（Server间通信算法）配置为"AHC"时，level2（超节点间通信算法）将自动采用"AHC"算法，无需另行配置，即使level2设置了其他算法，这些设置也不会生效。
  - pipeline：流水线并行算法，可并发使用Server内与Server间的链路，适合通信数据量较大且通信域内每机包含多卡的场景。
  - pairwise：逐对通信算法，仅用于AlltoAll、AlltoAllV与AlltoAllVC算子，通信步数较多（线性复杂度），时延相对较高，且需要额外申请内存，内存大小与数据量成正比，但可以避免网络中出现一打多现象，适合通信数据量较大、需要规避网络一打多的场景。
- level2代表超节点间通信算法，支持ring、H-D_R、NHR、NB、pipeline等取值（含义同level1）。

按算子类型配置通信算法，配置方式如下：

```bash
export HCCL_ALGO="<op0>=level0:NA;level1:<algo0>;level2:<algo1>/<op1>=level0:NA;level1:<algo3>;level2:<algo4>"
```

其中`<op>`为通信算子的类型，支持：allgather、reducescatter、allreduce、broadcast、reduce、scatter、alltoall。多个算子之间的配置使用"/"分隔。

配置示例：

```bash
# 全局配置算法类型
export HCCL_ALGO="level0:NA;level1:H-D_R"
# 按算子配置算法类型
export HCCL_ALGO="allreduce=level0:NA;level1:ring/allgather=level0:NA;level1:H-D_R"
```

使用约束：
- 当前版本Server内通信算法仅支持配置为"NA"。
- 针对Atlas A2训练系列产品，在严格确定性计算的保序场景下，不建议配置HCCL_ALGO环境变量。

支持的型号：Atlas训练系列产品、Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.1.4 HCCL_BUFFSIZE

功能描述：此环境变量用于控制两个NPU之间共享数据的缓存区大小。需要配置为整数，取值大于等于1，默认值为200，单位MB。

集合通信网络中，每一个HCCL通信域都会占用HCCL_BUFFSIZE大小的缓存区。若集群网络中存在较多的HCCL通信域，此缓存区占用量就会增多，可能存在影响模型数据正常存放的风险，此种场景下可通过此环境变量减少通信域占用的缓存区大小；若业务的模型数据量较小，但通信数据量较大，则可通过此环境变量增大HCCL通信域占用的缓存区大小，提升数据通信效率。

大语言模型中的建议配置值如下：
(MicrobatchSize * SequenceLength * hiddenSize * sizeof(DataType)) / (1024*1024)，向上取整。

此环境变量一般用于以下场景：
- 动态shape网络场景。
- 开发人员调用集合通信库HCCL的C语言接口进行框架对接的场景。

需要注意：
- 该环境变量申请的内存为HCCL独占，不可与其他业务内存复用。
- 每个通信域占用"2*HCCL_BUFFSIZE"大小的内存，分别用于收发内存。
- 该资源按通信域粒度管理，每个通信域独占一组内存，保证多通信域并发算子互不影响。
- 针对集合通信算子，当数据量超过HCCL_BUFFSIZE的取值时，可能会出现性能下降的情况，建议HCCL_BUFFSIZE的取值大于数据量。

配置示例：`export HCCL_BUFFSIZE=200`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.1.5 HCCL_INTRA_PCIE_ENABLE

功能描述：用于配置Server内是否使用PCIe链路进行通信。该环境变量默认值为1，可以单独配置，也可以与环境变量HCCL_INTRA_ROCE_ENABLE同时使用，配置说明如下：

- HCCL_INTRA_PCIE_ENABLE为1，HCCL_INTRA_ROCE_ENABLE不配置，Server内采用PCIe链路进行通信。
- HCCL_INTRA_PCIE_ENABLE不配置，HCCL_INTRA_ROCE_ENABLE为1，Server内采用RoCE链路进行通信。
- HCCL_INTRA_PCIE_ENABLE为1，HCCL_INTRA_ROCE_ENABLE为0，Server内采用PCIe链路进行通信。
- HCCL_INTRA_PCIE_ENABLE为0，HCCL_INTRA_ROCE_ENABLE为1，Server内采用RoCE链路进行通信。
- HCCL_INTRA_PCIE_ENABLE为0，HCCL_INTRA_ROCE_ENABLE为0，Server内采用PCIe链路进行通信。

> 说明
> - 若HCCL_INTRA_PCIE_ENABLE和HCCL_INTRA_ROCE_ENABLE均未配置，默认Server内采用PCIe链路通信。
> - 不支持HCCL_INTRA_PCIE_ENABLE和HCCL_INTRA_ROCE_ENABLE同时配置为1。

配置示例：`export HCCL_INTRA_PCIE_ENABLE=1`

使用约束：Atlas 200T A2 Box16异构子框存在左右两个模组，分别为0~7卡和8~15卡，针对此产品：单机场景下，当Server内采用PCIe链路通信时，若需要同时使用两个模组的卡，两个模组需使用相同的卡数目且在同一平面，即0卡和8卡、1卡和9卡（以此类推）需要同时使用；当Server内采用RoCE链路通信时，无此限制。

支持的型号：Atlas训练系列产品（仅支持Atlas 300T Pro训练卡）、Atlas A2训练系列产品（仅支持Atlas 200T A2 Box16异构子框）。

### 11.1.6 HCCL_INTRA_ROCE_ENABLE

功能描述：用于配置Server内或超节点内是否使用RoCE链路进行通信。

- 针对Atlas训练系列产品与Atlas A2训练系列产品，该环境变量用于配置Sever内是否使用RoCE链路进行通信，默认值0，配置说明同11.1.5节。
- 针对Atlas A3训练系列产品/Atlas A3推理系列产品，该环境变量仅在使用LLM-DataDist作为集群管理组件的场景下生效，用于配置超节点内是否使用RoCE链路进行通信，默认值0。配置为1时，针对Atlas 800T A3超节点、Atlas 800I A3超节点与Atlas 900 A3 SuperPoD超节点，超节点内LLM-DataDist通信采用RoCE链路，HCCL通信不受影响；针对A200T A3 Box8超节点，LLM-DataDist与HCCL通信都采用RoCE链路。

配置示例：`export HCCL_INTRA_ROCE_ENABLE=1`

使用约束：同11.1.5。

支持的型号：Atlas训练系列产品（仅支持Atlas 300T Pro训练卡）、Atlas A2训练系列产品（仅支持Atlas 200T A2 Box16异构子框）、Atlas A3训练系列产品/Atlas A3推理系列产品（仅在使用LLM-DataDist作为集群管理组件的场景下生效）。

### 11.1.7 HCCL_INTER_HCCS_DISABLE

功能描述：此环境变量用于配置超节点模式组网中超节点内的通信链路类型，支持如下取值：

- TRUE：代表超节点内的AI节点间使用RoCE进行RDMA通信。
- FALSE：代表超节点内的AI节点间使用HCCS通信链路进行SDMA通信。

默认值为"FALSE"。

配置示例：`export HCCL_INTER_HCCS_DISABLE=FALSE`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.1.8 HCCL_OP_EXPANSION_MODE

功能描述：该环境变量用于配置通信算法的编排展开位置，支持如下取值：

- AI_CPU：代表通信算法的编排展开位置在Device侧的AI CPU，Device侧根据硬件型号自动选择相应的调度器。
- AIV：代表通信算法的编排展开位置在Device侧的Vector Core，执行也在Vector Core。
- HOST：代表通信算法的编排展开位置为Host侧CPU，Device侧根据硬件型号自动选择相应的调度器。
- HOST_TS：代表通信算法的编排展开位置为Host侧CPU，Host向Device的Task Scheduler下发任务，Device的Task Scheduler进行任务调度执行。

| 产品型号 | 支持的配置 | 约束说明 | 默认值 |
|---|---|---|---|
| Atlas 300I Duo推理卡 | AI_CPU | 仅支持单机单通信域场景；仅支持AllReduce算子；配置为"AI_CPU"后，通信算子不再支持profiling性能数据采集与分析功能；对于静态shape图，不支持此配置项。 | HOST |
| Atlas A2训练系列产品 | AIV | 仅支持推理特性；不支持多通信域并行场景；支持Broadcast、AllReduce、AlltoAll、AlltoAllV、AlltoAllVC、AllGather、ReduceScatter、AllGatherV、ReduceScatterV算子（各算子有具体的数据类型约束）；对业务编译时分配的Vector Core核数有最低数量要求。 | HOST |
| Atlas A2训练系列产品 | HOST_TS | 无 | HOST |
| Atlas A3训练系列产品/Atlas A3推理系列产品 | AI_CPU | 在超节点内与超节点间支持全量通信算子。采用AI CPU模式时，单卡上的并发通信域数量不能超过6个，否则可能会因AI CPU核被占满而导致通信阻塞。 | AI_CPU |
| Atlas A3训练系列产品/Atlas A3推理系列产品 | AIV | 该配置项仅支持Atlas A3训练系列产品/Atlas A3推理系列产品的推理特性。该配置项不支持多通信域并行的场景（因为不支持多个通信域同时配置为"AIV"模式），否则可能会导致不可预期行为。您可以在初始化具有特定配置的通信域时，通过"HcclCommConfig"将某个通信域的算法编排展开位置设置为"AIV"。该配置项仅支持Broadcast、AllReduce、ReduceScatter、AllGather、AlltoAll、AlltoAllV、AlltoAllVC算子。针对Broadcast算子，数据类型支持int8、uint8、int16、uint16、int32、uint32、float16、float32、bfp16，仅支持单机场景8卡以内的单算子模式。针对AllReduce算子，数据类型支持int8、int16、int32、float16、float32、bfp16，reduce的操作类型仅支持sum、max、min。针对ReduceScatter算子，数据类型支持int8、int16、int32、float16、float32、bfp16，reduce的操作类型仅支持sum、max、min，仅支持超节点内的单机/多机通信，不支持跨超节点间通信。针对AllGather、AlltoAll、AlltoAllV、AlltoAllVC算子，数据类型支持int8、uint8、int16、uint16、int32、float16、float32、bfp16，仅支持超节点内的单机/多机通信，不支持跨超节点间通信。针对AllReduce、ReduceScatter、AllGather、AlltoAll（单机通信场景）算子，当数据量超过一定值时，为防止性能下降，系统会自动切换为AI_CPU模式（该阈值并非固定，会根据算子运行模式、是否启动确定性计算及网络规模等因素有所调整）；针对AlltoAllV、AlltoAllVC、AlltoAll（多机通信场景）算子，当配置为AIV模式时，系统不会自动切换为AI_CPU模式，为避免性能劣化，在任意两个rank之间的最大通信数据量不超过1MB时，建议配置为AIV模式，否则请采用AI_CPU模式。该配置项对业务编译时分配的Vector Core核数有最低数量要求，若业务编译分配的Vector Core核数无法满足算法编排的要求，HCCL会报错并提示所需要的最低Vector Core核数。注意：算法编排展开位置设置为"AIV"时，若同时设置了HCCL_DETERMINISTIC环境变量为"true"或"strict"，当数据量小于8MB时，仅AllReduce和ReduceScatter算子的确定性计算生效，其他场景和算子则以HCCL_DETERMINISTIC配置为准。 | AI_CPU |

配置示例：`export HCCL_OP_EXPANSION_MODE="HOST"`

使用约束：针对Atlas A2训练系列产品的推理特性：配置为"AIV"的场景下，若通过"CTRL+C"方式强制结束进程，在msnpureport工具导出的Device侧日志文件中可能会出现Device访问非法地址的错误，日志关键词为"devmm_page_fault_d2h_query_flag"、"devmm_svm_device_fault"或"ipc_fault_msg_para_check"，此种场景不会影响Device上卡的状态，不会影响后续新起任务的执行。

### 11.1.9 HCCL_DETERMINISTIC

功能描述：此环境变量用于配置是否开启归约类通信算子的确定性计算或保序功能，其中归约类通信算子包括AllReduce、ReduceScatter、ReduceScatterV、Reduce，归约保序是指严格的确定性计算，在确定性的基础上保证所有bit位的归约顺序一致。

开启归约算子的确定性计算或保序功能后，算子在相同的硬件和输入下，多次执行将产生相同的输出。

HCCL_DETERMINISTIC支持的取值如下：

- false：默认值，关闭确定性计算。
- true：开启归约类通信算子的确定性计算。
  - 针对Atlas A2训练系列产品，支持通信算子AllReduce、ReduceScatter、ReduceScatterV、Reduce。
  - 针对Atlas A3训练系列产品/Atlas A3推理系列产品，若算法的编排展开位置为AI CPU，所有归约类算子都为确定性计算，且不受此环境变量影响；若算法的编排展开位置为Vector Core，仅通信算子AllReduce和ReduceScatter涉及非确定性计算，配置为"true"后支持切换为确定性计算。
- strict：开启归约类通信算子的严格确定性计算，即保序功能（在确定性的基础上保证所有bit位的归约顺序均一致），配置为该参数时需满足以下条件：
  - 仅支持Atlas A2训练系列产品，且仅支持多机对称分布场景，不支持非对称（即卡数非对称）的场景。
  - 针对Atlas A3训练系列产品/Atlas A3推理系列产品，配置为"strict"时与配置为"true"的功能保持一致。
  - 支持通信算子AllReduce和ReduceScatter、ReduceScatterV。
  - 开启保序时，不支持饱和模式，仅支持INF/NaN模式。
  - 相较于确定性计算，开启保序功能后会产生一定的性能下降，建议在推理场景下使用该功能。

一般情况下无需开启归约算子的确定性计算或保序功能，当模型多次执行结果不同或者精度调优时，可通过此环境变量开启确定性计算或保序功能进行辅助调试调优，但开启后，算子执行时间会变慢，导致性能下降。

配置示例：`export HCCL_DETERMINISTIC=true`

使用约束：无

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.1.10 HCCL_LOGIC_SUPERPOD_ID

功能描述：针对Atlas A3训练系列产品/Atlas A3推理系列产品的超节点模式组网，若不使用rank table文件配置集群资源信息，可通过此环境变量指定当前节点运行进程所属的超节点ID，实现将一个物理超节点划分为多个逻辑超节点的功能。

该环境变量取值为string类型，长度需要小于128个字符，默认值为空字符串。

若不配置此环境变量，会获取环境中"Super Pod ID"的值作为超节点ID，"Super Pod ID"的取值可通过`npu-smi info -t spod-info -i id -c chip_id`命令查看。

配置示例：`export HCCL_LOGIC_SUPERPOD_ID=super_pod_id_1`

使用约束：

- 此环境变量仅适用于超节点模式组网下未使用rank table文件配置集群信息的场景，若使用了rank table文件，则优先使用rank table文件中的配置。
- 此环境变量的作用为将一个物理超节点划分为多个逻辑超节点，不支持将归属于不同物理超节点的rank配置到一个逻辑超节点内。
- 归属于同一超节点的rank id需要按照Device物理连接顺序连续排布，不支持交叉配置。例如一个超节点中有4个rank，按物理连接顺序分别为rank0、rank1、rank2、rank3，假设将这四个rank划分为两个超节点，分别标识为"super_pod_0"与"super_pod_1"：
  - 正确配置示例：将rank0、rank1归属为"super_pod_0"，将rank2、rank3归属为"super_pod_1"。
  - 错误配置示例：将rank0、rank2归属为"super_pod_0"，将rank1、rank3归属为"super_pod_1"。

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品。

## 11.2 性能相关

### 11.2.1 HCCL_RDMA_PCIE_DIRECT_POST_NOSTRICT

功能描述：多机通信且Host操作系统小页内存页表大小非4KB的场景，当通信算子下发性能Host Bound时，开发者可通过此环境变量设置通过PCIe Direct的方式提交RDMA任务，提升通信算子下发性能。

此环境变量支持如下取值：

- TRUE：代表通过PCIe Direct（Host与Device之间的高速通信接口）的方式提交RDMA任务。
- FALSE（默认值）：代表通过HDC（Host Device Communication，主机设备通信）的方式提交RDMA任务。

此环境变量仅对Host侧小页内存页表大小非4KB的场景生效，若Host侧小页内存页表大小是4KB，无论此环境变量取值如何，都采用PCIe Direct的方式提交RDMA任务。

> 须知
> - 此环境变量设置为TRUE时，会额外消耗Device侧的大页内存（每个通信链路会额外多占用1MB的大页内存）。
> - 如果开发者既想通过此环境变量提升通信算子下发性能，又想节省Device侧大页内存占用，可通过HCCL_ALGO环境变量将server间通信算法设置为ring，以控制通信链路数量。
>   `export HCCL_ALGO="level0:NA;level1:ring"`

配置示例：`export HCCL_RDMA_PCIE_DIRECT_POST_NOSTRICT=TRUE`

使用约束：需要满足功能描述中所述场景，即多机通信场景且Host操作系统小页内存页表大小非4KB。

支持的型号：Atlas A2训练系列产品。

### 11.2.2 HCCL_RDMA_QPS_PER_CONNECTION

功能描述：两个rank之间RDMA通信时会默认创建1个QP（Queue Pair）进行数据传输，若开发者想让两个rank之间的RDMA通信使用多个QP，可通过此环境变量实现。

此环境变量代表两个rank间需要使用的QP个数，需要配置为整数，取值范围：[1,32]，建议配置范围：[1,8]，QP个数超过8时无法确保性能收益，还可能会造成由于内存占用过多导致业务运行失败的情况。默认值：1。

假设HCCL_RDMA_QPS_PER_CONNECTION环境变量配置为N1，则会在每两个rank之间创建N1个QP，两个rank之间通过RDMA传递的业务数据会平均分配到N1个QP上并行收发。

开启多QP传输的功能后，开发者可通过环境变量11.2.4 HCCL_MULTI_QP_THRESHOLD设置每个QP分担数据量的最小阈值；若开发者想指定每个QP使用的源端口号，可通过环境变量11.2.3 HCCL_RDMA_QP_PORT_CONFIG_PATH实现。

配置示例：`export HCCL_RDMA_QPS_PER_CONNECTION=4`

使用约束：

- 该环境变量仅支持单算子调用方式，不支持静态图模式。
- QP相关配置的优先级如下：管理面多QP配置（通过hccn_tool工具的"-s multi_qp"参数配置） > NSLB的QP配置（通过hccn_tool工具的"-t nslb-dp"参数配置） > 环境变量HCCL_RDMA_QP_PORT_CONFIG_PATH > 环境变量HCCL_RDMA_QPS_PER_CONNECTION。

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.2.3 HCCL_RDMA_QP_PORT_CONFIG_PATH

功能描述：两个rank之间RDMA通信时会默认创建1个QP（Queue Pair）进行数据传输，若开发者想让两个rank之间的RDMA通信使用多个QP，并指定多QP通信时使用的源端口号，可通过此环境变量实现。

开发者可通过此环境变量指定<srcIP,dstIP>与端口映射关系配置文件的存储路径，当<srcIP,dstIP>配置多个端口号时，即开启多QP通信，所配置的端口号即为每个QP使用的源端口。

该环境变量配置示例如下：

`export HCCL_RDMA_QP_PORT_CONFIG_PATH=/home/tmp`

其中"/home/tmp"为<srcIP,dstIP>与端口映射关系配置文件"MultiQpSrcPort.cfg"的存储路径，支持配置为绝对路径或相对路径，该路径最大长度需要小于等于4096个字符。

"MultiQpSrcPort.cfg"文件需要用户自定义（注意文件命名需要保持为"MultiQpSrcPort.cfg"），配置格式如下：

```
srcIP1,dstIP1=srcPort0,srcPort1,...,srcPortN
srcIPN,dstIPN=srcPort0,srcPort1,...,srcPortN
```

- 该文件支持的最大配置行数为128*1024=131072。
- 每个<srcIP,dstIP>地址对最多支持配置32个端口，但建议不超过8个端口，因为QP个数超过8时无法确保性能收益，还可能会造成由于内存占用过多导致业务运行失败的情况。
- 每个<srcIP,dstIP>地址对在该文件中仅允许出现一次。
- srcIP、dstIP需要为常规IPv4格式，不支持IPv6格式。
- srcIP、dstIP支持配置为"0.0.0.0"，代表所有IP地址。

"MultiQpSrcPort.cfg"文件配置示例如下：

```
192.168.100.2,192.168.100.3=61100,61101,61102
192.168.100.4,192.168.100.5=61100,61101,61102,61104
0.0.0.0,192.168.100.122=65515,65516,65513
```

配置示例：`export HCCL_RDMA_QP_PORT_CONFIG_PATH=/home/tmp`

使用约束：

- 该环境变量仅支持单算子调用方式，不支持静态图模式。
- 该环境变量的优先级高于环境变量HCCL_RDMA_QPS_PER_CONNECTION，此环境变量配置后，两个rank间通信时使用的QP个数以"MultiQpSrcPort.cfg"文件中配置的源端口号个数为准。
- QP相关配置的优先级如下：管理面多QP配置（通过hccn_tool工具的"-s multi_qp"参数配置） > NSLB的QP配置（通过hccn_tool工具的"-t nslb-dp"参数配置） > 环境变量HCCL_RDMA_QP_PORT_CONFIG_PATH > 环境变量HCCL_RDMA_QPS_PER_CONNECTION。

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.2.4 HCCL_MULTI_QP_THRESHOLD

功能描述：rank间RDMA通信使用多QP通信的场景下，开发者可通过本环境变量设置每个QP分担数据量的最小阈值。

该环境变量需要配置为整数，取值范围：[1,8192]，默认值：512，单位：KB。

- 如果"(rank间单次通信数据量 / HCCL_RDMA_QPS_PER_CONNECTION取值) < HCCL_MULTI_QP_THRESHOLD取值"，则HCCL执行时会自动减少QP个数，使得每个QP上分担的数据量大于等于HCCL_MULTI_QP_THRESHOLD的取值，例如：rank间单次通信数据量为1MB，HCCL_RDMA_QPS_PER_CONNECTION配置为4，HCCL_MULTI_QP_THRESHOLD配置为512，此时每个QP最少要求分担512KB的数据量，则HCCL执行时，会减少QP个数为2，仅使用2个QP进行rank间的数据传输。
- 当rank间数据量小于HCCL_MULTI_QP_THRESHOLD时使用单QP传输。
- 当每个QP分担的数据量大于512KB时，使用HCCL Test工具进行RDMA流量测试时（仅测试跨机流量，不使用HCCS链路），多QP场景的下发调度开销相对于单QP场景性能劣化小于3%。

> 说明：可通过环境变量11.2.2 HCCL_RDMA_QPS_PER_CONNECTION或11.2.3 HCCL_RDMA_QP_PORT_CONFIG_PATH开启多QP通信。

配置示例：`export HCCL_MULTI_QP_THRESHOLD=512`

使用约束：该环境变量仅支持单算子调用方式，不支持静态图模式。

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

## 11.3 网络相关

### 11.3.1 HCCL_IF_IP

功能描述：配置HCCL初始化时Host使用的通信IP地址，此IP地址用于与root节点通信，以完成通信域的创建。

格式为字符串，要求为常规IPv4或IPv6格式，暂只支持Host网卡，且只能配置一个IP地址。

HCCL按照如下优先级顺序选择Host通信网卡：

环境变量HCCL_IF_IP > 环境变量HCCL_SOCKET_IFNAME > docker/lo以外网卡（网卡名字典序升序） > docker网卡 > lo网卡。

> 须知：如果不配置HCCL_IF_IP或HCCL_SOCKET_IFNAME，系统将按照优先级自动选择网卡。若当前节点选择的网卡与root节点选择的网卡链路不通，将导致HCCL建链失败。

配置示例：`export HCCL_IF_IP=10.10.10.1`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.2 HCCL_IF_BASE_PORT

功能描述：单算子模式下，使用Host网卡进行HCCL初始化或集合通信计算时（即通信域的创建方式为"基于root节点信息创建"时），可以通过该环境变量指定Host网卡起始端口号，配置后系统默认占用以该端口起始的32个端口进行集群信息收集。

该环境变量需要配置为整数，取值范围为[1024,65520]，请确保分配的端口未被占用。

配置示例：`export HCCL_IF_BASE_PORT=50000`

使用约束：分布式场景下，HCCL会使用Host服务器的部分端口进行集群信息收集，需要操作系统预留该部分端口。

- 若不通过HCCL_IF_BASE_PORT环境变量指定端口，默认HCCL使用60000-60031端口，需要执行如下命令预留此范围的操作系统端口：`sysctl -w net.ipv4.ip_local_reserved_ports=60000-60015`
- 若通过HCCL_IF_BASE_PORT环境变量指定端口，例如指定端口为50000，则HCCL使用50000-50031端口，需要执行如下命令预留此范围的操作系统端口：`sysctl -w net.ipv4.ip_local_reserved_ports=50000-50015`

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.3 HCCL_HOST_SOCKET_PORT_RANGE

功能描述：当通信域的创建方式为"基于root节点信息创建"时，开发者可通过此环境变量配置HCCL在Host侧使用的通信端口。

该环境变量支持配置为具体的端口、端口范围或者字符串"auto"。

- 若指定具体的端口号或端口范围，规划的端口数量建议不小于单个NPU上的HCCL进程数，端口号取值范围为[1,65535]，且需要确保指定的端口未被其他进程占用。需要注意，[1,1023]为系统保留端口，应避免使用这些端口。具体的端口号与端口范围可以组合使用，中间使用英文","分隔，逗号之间的端口号/端口范围不能存在范围交叉。
- 若指定为字符串"auto"，代表HCCL使用的Host通信端口由操作系统动态分配。

配置示例：

```bash
# 方式一：配置为端口范围。
export HCCL_HOST_SOCKET_PORT_RANGE="60000-60050"
# 方式二：具体的端口号与端口范围配合使用，使用英文","分隔。
export HCCL_HOST_SOCKET_PORT_RANGE="60000,60050-60100,60150-60160"
# 方式三：指定具体的端口号，使用英文","分隔。
export HCCL_HOST_SOCKET_PORT_RANGE="56000,56005,56007,56008,56100,56105,56107,56108"
# 方式四：操作系统动态分配端口号
export HCCL_HOST_SOCKET_PORT_RANGE="auto"
```

使用约束：

- 若业务为单卡多进程场景（即多个业务进程同时共用一个NPU），建议配置此环境变量，否则业务可能会因为端口冲突运行失败。但需要注意，多进程会对资源开销、通信性能产生一定的影响。
- 此环境变量优先级高于11.3.2 HCCL_IF_BASE_PORT，若配置了此环境变量，HCCL在Host侧使用的通信端口以此环境变量为准。
- 针对Atlas A2训练系列产品，若网络中存在MC2通算融合算子（计算和通信融合的算子，例如AllGatherMatmul、MatmulReduceScatter、AlltoAllAllGatherBatchMatMul等），不支持配置此环境变量。

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.3.4 HCCL_NPU_SOCKET_PORT_RANGE

功能描述：当通信域的创建方式为"基于root节点信息创建"时，开发者可通过此环境变量配置HCCL在NPU侧使用的通信端口。

该环境变量支持配置为具体的端口、端口范围或者字符串"auto"。

- 若指定具体的端口号或端口范围，规划的端口数量建议不小于单个NPU上的HCCL进程数，端口号取值范围为[1,65535]，且需要确保指定的端口未被其他进程占用。需要注意，[1,1023]为系统保留端口，应避免使用这些端口。具体端口号与端口范围可以组合使用，中间使用英文","分隔，但逗号之间的端口号/端口范围不能存在范围交叉。
- 若指定为字符串"auto"，代表HCCL使用的NPU端口号由操作系统动态分配。

配置示例：

```bash
# 方式一：配置为端口范围。
export HCCL_NPU_SOCKET_PORT_RANGE="61000-61050"
# 方式二：具体的端口号与端口范围配合使用，使用英文","分隔。
export HCCL_NPU_SOCKET_PORT_RANGE="61000,61050-61100,61200-61210"
# 方式三：指定具体的端口号，使用英文","分隔。
export HCCL_NPU_SOCKET_PORT_RANGE="57000,57005,57007,58008,58100,58105,58107,58108"
# 方式四：操作系统动态分配端口号
export HCCL_NPU_SOCKET_PORT_RANGE="auto"
```

使用约束：

- 若业务为单卡多进程场景（即多个业务进程同时共用一个NPU），建议配置此环境变量，否则业务可能会因为端口冲突运行失败。但需要注意，多进程会对资源开销、通信性能产生一定的影响。
- 针对Atlas A2训练系列产品，若网络中存在MC2通算融合算子（计算和通信融合的算子，例如AllGatherMatmul、MatmulReduceScatter、AlltoAllAllGatherBatchMatMul等），不支持配置此环境变量。

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.3.5 HCCL_SOCKET_IFNAME

功能描述：配置HCCL初始化时Host使用的通信网卡名，HCCL将通过该网卡名获取Host IP，与root节点通信，以完成通信域的创建。

支持以下格式配置：

- eth：使用所有以eth为前缀的网卡。若指定多个网卡前缀，多个网卡前缀间用英文逗号分隔。例如：`export HCCL_SOCKET_IFNAME=eth,enp`，表示使用所有以eth或enp为前缀的网卡。
- ^eth：不使用以eth为前缀的网卡。若指定多个网卡前缀，多个网卡前缀间用英文逗号分隔。例如：`export HCCL_SOCKET_IFNAME=^eth,enp`，表示不使用任何以eth或enp为前缀的网卡。
- =eth0：使用指定eth0网卡。若指定多个网卡，多个网卡间用英文逗号分隔。例如：`export HCCL_SOCKET_IFNAME==eth0,enp0`，表示使用eth0网卡或enp0网卡。
- ^=eth0：不使用指定eth0网卡。若指定多个网卡，多个网卡间用英文逗号分隔。例如：`export HCCL_SOCKET_IFNAME=^=eth0,enp0`，表示不使用eth0与enp0网卡。

> 须知
> - HCCL_SOCKET_IFNAME中可配置多个网卡，取最先匹配到的网卡作为通信网卡。
> - 环境变量HCCL_IF_IP的优先级高于HCCL_SOCKET_IFNAME。
> - 如果用户未指定HCCL_IF_IP和HCCL_SOCKET_IFNAME，按照如下优先级选择：docker/lo以外网卡（网卡名字典序升序） > docker网卡 > lo网卡。需要注意，如果不配置HCCL_IF_IP或HCCL_SOCKET_IFNAME，系统将按照优先级自动选择网卡。若当前节点选择的网卡与root节点选择的网卡链路不通，将导致HCCL建链失败。

配置示例：

```bash
# 使用eth0或endvnic的网卡
export HCCL_SOCKET_IFNAME==eth0,endvnic
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.6 HCCL_SOCKET_FAMILY

功能描述：该环境变量指定通信网卡使用的IP协议，支持如下两种配置：

- AF_INET：代表使用IPv4协议。
- AF_INET6：代表使用IPv6协议。

缺省使用IPv4协议。

该环境变量有以下两种使用场景：

- 配置HCCL初始化时，Host侧通信网卡使用的IP协议版本。此场景下，该环境变量需要与HCCL_SOCKET_IFNAME同时使用，当HCCL通过指定网卡名获取Host IP时，通过该环境变量指定使用网卡的socket通信IP协议。
- 配置HCCL初始化时，Device侧通信网卡使用的IP协议版本。此场景下，如果该环境变量指定的IP协议与实际获取到的网卡信息不匹配，则以实际环境上的网卡信息为准。例如，该环境变量指定为IPv6协议，但Device侧只存在IPv4协议的网卡，则实际会使用IPv4协议的网卡。

配置示例：

```bash
export HCCL_SOCKET_FAMILY=AF_INET    #IPv4
export HCCL_SOCKET_FAMILY=AF_INET6   #IPv6
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.7 HCCL_RDMA_TC

功能描述：用于配置RDMA网卡的traffic class。

该环境变量的取值范围为[0,255]，且需要配置为4的整数倍，默认值为132。

在RoCE V2协议中，该值对应IP报文头中ToS（Type of Service）域。共8个bit，其中bit[0,1]固定为0，bit[2,7]为DSCP，因此该值除以4即为DSCP的值。

[图: ToS (1 Byte) bit field diagram showing DSCP in bit7-bit2 and Reserved in bit1-bit0]

配置示例：

```bash
# 该环境变量配置为25*4 = 100，则DSCP为25
export HCCL_RDMA_TC=100
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.8 HCCL_RDMA_SL

功能描述：用于配置RDMA网卡的service level，该值需要和网卡配置的PFC优先级保持一致，若配置不一致可能导致性能劣化。

该环境变量需要配置为整数，取值范围：[0,7]，默认值：4。

配置示例：

```bash
# 优先级配置为3
export HCCL_RDMA_SL=3
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.9 HCCL_RDMA_TIMEOUT

功能描述：用于配置RDMA网卡重传超时时间的系数timeout。

RDMA网卡重传超时时间最小值的计算公式为：4.096us * 2^timeout，其中timeout为该环境变量配置值，且实际重传超时时间与用户网络状况有关。

- 针对Atlas A3训练系列产品/Atlas A3推理系列产品，该环境变量配置为整数，取值范围为[5,20]，默认值为20。
- 针对Atlas A2训练系列产品，该环境变量配置为整数，取值范围为[5,20]，默认值为20。
- 针对Atlas训练系列产品，该环境变量配置为整数，取值范围为[5,24]，默认值为20。
- 针对Atlas 300I Duo推理卡，该环境变量配置为整数，取值范围是[5,24]，默认值为20。

配置示例：

```bash
# RDMA网卡重传超时时间的系数配置为6，则网卡启用RDMA功能时，重传超时时间最小值为：4.096us * 2^6
export HCCL_RDMA_TIMEOUT=6
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.3.10 HCCL_RDMA_RETRY_CNT

功能描述：用于配置RDMA网卡的重传次数，需要配置为整数，取值范围为[1,7]，默认值为7。

配置示例：

```bash
# 重传次数配置为5
export HCCL_RDMA_RETRY_CNT=5
```

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

## 11.4 调试相关

### 11.4.1 HCCL_DIAGNOSE_ENABLE

功能描述：此环境变量用于配置集合通信是否缓存部分任务的详细信息，以便任务执行失败时，打印详细日志，用于问题定位。

支持如下取值：

- 1：代表开启集合通信缓存。
- 0：代表不开启集合通信缓存。

默认值为"0"。

需要注意，此环境变量开启后会对性能产生一定的影响。

配置示例：`export HCCL_DIAGNOSE_ENABLE=1`

使用约束：最多保存最新的2000个算子信息。

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品。

### 11.4.2 HCCL_ENTRY_LOG_ENABLE

功能描述：此环境变量用于控制是否实时打印通信算子的调用行为日志。

- 1：代表实时打印通信算子的调用行为日志，即调用一次通信算子，打印一条运行日志。
- 0：代表不打印通信算子的调用行为运行日志。

默认值为"0"。

HCCL的默认运行日志存储路径为：$HOME/ascend/log/run/plog/plog-pid_*.log，关于日志的详细说明可参见"日志参考"。

配置示例：`export HCCL_ENTRY_LOG_ENABLE=1`

使用约束：仅用于集合通信算子的单算子调用场景。

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.4.3 HCCL_DEBUG_CONFIG

功能描述：启用此环境变量后，运行日志（即"$HOME/ascend/log/run"目录下的日志）将包含HCCL特定子模块的详细运行信息。目前支持ALG或alg（算法编排模块）、TASK或task（任务编排模块）、RESOURCE或resource（资源管理模块，包括资源的申请和释放操作）、AIV_OPS_EXC或aiv_ops_exc（AIV算子日志打印模块，包括算子执行过程中的通信内存操作和资源同步操作）四个配置项。

该环境变量支持如下两种形式的配置：

- 正向配置：支持配置1个或多个模块，各模块间使用英文逗号分隔，其中TASK（或task）、ALG（或alg）、RESOURCE（或resource）、AIV_OPS_EXC（或aiv_ops_exc）不区分大小写。
  ```bash
  # 运行日志中记录task模块的运行信息。
  export HCCL_DEBUG_CONFIG="TASK"
  # 运行日志中记录alg、task、resource模块的运行信息。
  export HCCL_DEBUG_CONFIG="alg,task,resource,aiv_ops_exc"
  ```
- 反向配置：在第一个模块名前面加上"^"，表示除了配置的子模块外，运行日志中会记录其他模块的详细运行信息。
  ```bash
  # 运行日志中记录除了task模块之外的其他所有模块的运行信息（当前版本代表仅记录alg、resource、aiv_ops_exc模块的运行信息）。
  export HCCL_DEBUG_CONFIG="^task"
  # 运行日志中记录除了task与alg模块之外的其他所有模块的运行信息（当前版本代表记录resource、aiv_ops_exc模块的运行信息）。
  export HCCL_DEBUG_CONFIG="^task,alg"
  ```

注意：环境变量配置时，不允许存在多余空格，否则配置无效，例如：`export HCCL_DEBUG_CONFIG="alg, task "`，task前后存在多余空格，此环境变量配置无效。

配置示例：`export HCCL_DEBUG_CONFIG="ALG,TASK,RESOURCE,AIV_OPS_EXC"`

使用约束：无

支持的型号：Atlas A2训练系列产品、Atlas A3训练系列产品/Atlas A3推理系列产品。

### 11.4.4 HCCL_DFS_CONFIG

功能描述：HCCL提供了多种故障检测功能的开关设置，包括建链故障探测时间配置、集群心跳监测开关以及进程卡死检测开关。这些检测功能默认开启，能够在业务出现异常时快速定位并显示故障根节点信息，有助于问题及时排查处理。在某些特定场景下，用户也可以通过该环境变量选择性地关闭这些检测功能。

该环境变量有以下三个配置项：

- connection_fault_detection_time：建链故障探测时间。HCCL会在建链超时时启动建链失败根节点定位能力，并将失败根节点信息传播。整个过程耗时为："connection_fault_detection_time"参数取值 + 10s的根节点信息传播时间。"connection_fault_detection_time"参数支持的取值：0，[20, 7200]。单位s，默认为20。该参数配置为"0"时，代表关闭建链故障探测功能，即建链失败时无额外等待时间，建链进程立即退出。
- cluster_heartbeat：集群心跳监测开关，用于通信操作执行超时的情况下，扩散故障信息，并在运行日志中记录故障根节点信息。该参数支持两种取值：on（开启心跳监测功能）、off（关闭心跳监测功能），默认值为on。说明：关闭集群心跳监测开关后，通信操作执行超时的异常情况无法探测，集群故障扩散能力丢失，且根节点故障信息不会记录到运行日志中。
- stuck_detection：进程卡死检测开关。该参数支持两种取值：on（开启进程卡死检测能力）、off（关闭进程卡死检测能力），默认值为on。对于通信性能非常敏感的场景，可通过此参数关闭进程卡死检测能力，但需要注意，关闭进程卡死检测能力后，不会再主动探测上报业务异常卡死故障。

> 须知：本检测功能仅用于辅助定位集群故障点位置，在某些复杂场景下可能不是集群业务失败的根因位置。请基于探测事件的生成时间、被检测节点的具体报错进一步确认故障根节点位置。

配置示例：`export HCCL_DFS_CONFIG="connection_fault_detection_time:30;cluster_heartbeat:on;stuck_detection:on"`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品。

## 11.5 可靠性相关

### 11.5.1 HCCL_OP_RETRY_ENABLE

功能描述：此环境变量用于配置是否开启HCCL算子的重执行特性。HCCL算子重执行以通信域为粒度，当通信算子执行报SDMA或者RDMA CQE类型的错误时，HCCL会尝试重新执行此通信算子。

在集群环境中，可能会存在硬件闪断的情况，此时通信算子会执行报错，通过此环境变量开启HCCL的重执行特性，可以更好地避免由于硬件闪断造成的通信中断，提升通信稳定性。HCCL算子重执行实际就是提供一个软件层面的尽力而为的故障恢复手段，执行示意图如下所示：

[图: 重执行流程示意图，展示Host端集群管理/HCCL重执行状态机、Device端的计算stream/通信stream/AI CPU以及通信执行主流/从流的交互过程]

主要步骤有如下三步：

1. 故障发现：AI CPU检测到故障信号，通知Host开始准备进入重执行流程。
2. 集群管理：Host通过Host Socket进行信息交互，并判断当前故障算子是否满足一系列重执行条件，详细可参见"重执行使用须知"。
3. 重新下发：通知AI CPU Kernel重新下发SQE、WQE，进行HCCL算子重执行。

配置说明：通过此环境变量，开发者可以在Server间、超节点间两个物理层级的通信域中配置是否开启重执行特性，每个层级支持配置两种状态：开启或关闭。

配置方法如下：

`export HCCL_OP_RETRY_ENABLE="L1:1,L2:0"`

- L1代表通信域的物理范围为Server间通信域，取值为0表示通信域内Server间通信task不开启重执行，取值为1表示通信域内Server间通信task开启重执行，默认值为1。
- L2代表通信域的物理范围为超节点间通信域，取值为0表示通信域内超节点间通信task不开启重执行，取值为1表示通信域内超节点间通信task开启重执行，默认值为0。L2配置为"1"时，超节点间通信若某一Device网卡故障，重执行时会使用备用Device网卡进行通信，称之为"借轨通信"。备用网卡为属于同一NPU中的另一个Die网卡，借轨通信正常执行的条件以及借轨通信的影响详细可参见"借轨通信使用须知"。
  - 如果通信域的创建方式为"基于rank table创建通信域"，需要开发者在rank table文件中通过"backup_device_ip"参数配置备用网卡。
  - 如果通信域的创建方式为"基于root节点信息创建通信域"，会自动将同一NPU下的两个Die互为备用网卡，无需开发者手工配置。

另外，开发者可以通过环境变量11.5.2 HCCL_OP_RETRY_PARAMS配置最大重执行的次数与重传间隔时间等信息。

配置建议：

- 开启重执行特性后会有一定的性能损失，针对Atlas A3训练系列产品/Atlas A3推理系列产品，Server间与超节点间，会经过光互联域，稳定性较低，建议开启HCCL重执行。
- 此环境变量在各个超节点上的配置需要保持一致，否则超节点间建链会超时。

重执行使用须知：

开启HCCL重执行特性时，需要满足以下约束条件，约束条件不满足时重执行会失败。

1. 通信算法的编排展开位置在Device的AI CPU计算单元，重执行特性仅在AI CPU调度模式下使能，需要通过环境变量HCCL_OP_EXPANSION_MODE设置，非AI CPU调度模式下会走无重执行流程。`export HCCL_OP_EXPANSION_MODE="AI_CPU"`
2. 基于rank table创建通信域的场景下，rank table中"host_ip"字段必须配置，否则重执行不生效，走无重执行流程。
3. 通信算子的输入内存在执行过程中不能存在被污染的风险。一个集合通信算子是一系列任务的组合，HCCL重执行以通信算子为粒度，从算子的输入内存开始将一个通信算子的系列任务重新执行一遍。若通信算子的输入内存在执行过程中存在被污染的风险，可能会造成重执行失败、系统报错退出。输入内存存在被污染风险的场景主要有以下几种：
   - 开启零拷贝功能的场景：零拷贝功能开启后，ReduceScatter和AllReduce算子会修改用户的输入内存，因此这两类算子无法支持重执行。
   - 包含In-Place操作的场景：此场景下，算子的输入和输出共享同一块内存，例如PyTorch的ReduceScatter/AllGather算子，所以包含In-Place操作的场景也不支持重执行。
   - 图模式场景：在图模式下，通信可以直接在算子输入输出上进行，例如PyTorch的AllReduce算子，其入参tensor即作为算子的输入，又作为算子的输出，在算子的通信过程中，部分结果写入后，tensor内容就会发生变化，如果在污染的input上重新执行一遍，会得到错误的计算结果。所以此场景也不支持重执行。
4. 故障发生时通信域内所有rank都停在同一个通信算子。若不同的rank停在不同的通信算子上，则不能重执行。故障发生的时刻是不可预测的，发生故障时，整个通信域各rank处于什么状态与重执行的成功率相关。
5. 判断Host侧socket网络通信是否正常。重执行时会使用Host侧socket通信进行通信域中各卡状态的协商，如果socket网络故障，则无法进行重执行。
6. 确保故障的链路已恢复，例如路由收敛成功，光模块闪断恢复成功或者借用备用网卡通信成功等。如果故障的链路无法恢复，再次执行通信任务仍然会失败，当重执行次数超过设置的最大重传次数后（可参见环境变量HCCL_OP_RETRY_PARAMS），重执行失败。

> 说明
> - 若Host侧调试日志中出现关键字为"[OpRetry]...timeout"的ERROR错误信息，说明HCCL重执行过程中Host侧socket通信异常，此时可以搜集通信域内所有节点的日志，进一步定位故障发生位置。
> - 若Host侧调试日志中出现关键字为"can not retry"的ERROR错误信息，说明当前场景不满足HCCL重执行条件。Host侧应用程序产生的调试日志默认存储路径为：$HOME/ascend/log/debug/plog/。

借轨通信使用须知：

1. 为了确保借轨通信功能正常执行，需要满足如下条件：备用网卡的通信链路正常；互为主备的Device均需在业务可见范围内。例如，NPU1中包含Device0与Device1两个Die，互为主备，假设通过环境变量ASCEND_RT_VISIBLE_DEVICES指定了业务可见Device为Device0，Device1不对业务可见，则借轨功能无法执行。
2. 如果通信过程中发生了借轨（假设某个NPU的Die0网卡故障，启用了备用的Die1网卡），原Die0的网卡流量也会通过Die1网卡收发，导致Die1的流量增大，总体性能会由于物理带宽减半、端口冲突导致下降。
3. 开启借轨通信场景下，若NPU0的Die0网卡故障，会切换到其备用网卡Die1，由于两个NPU之间的通信要求本端与对端同时切换为备用网卡，因此NPU1也会从Die0切换为Die1，即"图示二"所示。但如果Die0与Die1之间本身就存在通信任务，此时借轨功能无法执行。
4. 开启借轨通信功能时，建议一个NPU的两个Die分配给同一个训练或推理任务。如果同一个NPU的两个Die分给两个不同的训练或推理任务，一个任务发生故障，会借用另一个任务的网卡，两个任务均会发生一定程度的性能下降。
5. 同一NPU仅支持发生一次借轨，且不支持回切。如果NPU0 -> NPU1间通信链路故障，启用了备用链路，发生借轨，通信正常进行；若再发生故障，则不再支持借轨，会报错退出。

异常处理：若开启重执行特性后，出现"[OpRetryConnection][RecvAckTag] Recv unmatched ack"的错误，可能是由于HCCL通信时使用的默认端口被占用，导致HCCL连接了错误的Server，解决方法如下：

1. 使用"sysctl -w net.ipv4.ip_local_reserved_ports=60000-60015"命令预留HCCL使用的默认端口60000-60015，避免端口被操作系统随机分配。
2. 如果前一种方法仍出现该错误，那么建议使用HCCL_IF_BASE_PORT环境变量修改HCCL使用的默认端口，同时使用"sysctl -w net.ipv4.ip_local_reserved_ports"命令预留指定的端口。

重执行对整网性能说明：请参见11.7.1 通信算子重执行对整网性能说明。

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品。

### 11.5.2 HCCL_OP_RETRY_PARAMS

功能描述：当开发者通过环境变量11.5.1 HCCL_OP_RETRY_ENABLE开启了HCCL的算子重执行特性时，可通过本环境变量配置第一次重执行的等待时间、最大重执行的次数以及两次重执行的间隔时间。

配置方法如下：

`export HCCL_OP_RETRY_PARAMS="MaxCnt:3,HoldTime:5000,IntervalTime:1000"`

- MaxCnt：最大重传次数，uint32类型，取值范围为[1,10]，默认值为1，单位次。
- HoldTime：从检测到通信算子执行失败到开始第一次重新执行的等待时间，uint32类型，取值范围[0,60000]，默认值为5000，单位ms。
- IntervalTime：同一个通信算子两次重执行的间隔时间，uint32类型，取值范围[0,60000]，默认值为1000，单位ms。

配置示例：`export HCCL_OP_RETRY_PARAMS="MaxCnt:5,HoldTime:5000,IntervalTime:5000"`

使用约束：仅通过环境变量11.5.1 HCCL_OP_RETRY_ENABLE开启了HCCL的重执行特性时（开启任一层级的重执行特性即可），此环境变量才生效。

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品。

## 11.6 安全相关

### 11.6.1 HCCL_WHITELIST_DISABLE

功能描述：配置在使用HCCL时是否开启通信白名单。

- 0：开启白名单，校验HCCL通信白名单，只有在通信白名单中的IP地址才允许进行集合通信。
- 1：关闭白名单，无需校验HCCL通信白名单。

缺省值为1，默认关闭白名单。如果开启了白名单校验，需要通过HCCL_WHITELIST_FILE指定白名单配置文件路径。

配置示例：`export HCCL_WHITELIST_DISABLE=1`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

### 11.6.2 HCCL_WHITELIST_FILE

功能描述：当通过HCCL_WHITELIST_DISABLE开启了通信白名单校验功能时，需要通过此环境变量配置指向HCCL通信白名单配置文件的路径，只有在通信白名单中的IP地址才允许进行集合通信。

HCCL通信白名单配置文件格式为：

```json
{ "host_ip": ["ip1", "ip2"], "device_ip": ["ip1", "ip2"] }
```

其中：

- device_ip为预留字段，当前版本暂不支持。
- IP地址格式为点分十进制。

> 注意：白名单IP需要指定为集群通信使用的有效IP。

配置示例：`export HCCL_WHITELIST_FILE=/home/test/whitelist`

使用约束：无

支持的型号：Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品、Atlas训练系列产品、Atlas 300I Duo推理卡。

## 11.7 参考

### 11.7.1 通信算子重执行对整网性能说明

开启HCCL通信算子重执行功能后，整网端到端性能的变化与模型的切分部署方式密切相关，本节详细讲述重执行功能与网络性能的关系。

定义"关键通信域"：关键通信域为该通信域性能的变化将会带来整网端到端性能的较大的变化。意味着该通信域非常重要，是整网的性能瓶颈。

一般而言，整网有多个通信域，多个通信域中往往存在1个关键通信域，本节性能分析就围绕该"关键通信域"展开。

整网性能劣化与"关键通信域"的关系：

- 关注点1：关键通信域是否开了重执行。一些常见的部署方式，例如张量并行（TP: Tensor Parallelism）叠加数据并行（Data Parallelism: DP），其中TP是"关键通信域"，如果TP的范围在Server内（TP<=16），由于Server内不会开启通信算子重执行，所以不会影响端到端性能。而非关键通信域，对整网的性能影响很小。

| 模型 | 切分方式 | 劣化比例 | 说明 |
|---|---|---|---|
| Llama3-8B（运行在64die规模集群上） | TP=16（关键通信域）DP=4 | 0.03% | 仅非关键通信域DP开启重执行，对端到端性能影响较小。 |
| GPT4_dropLess（运行在128die规模集群） | TP=8（关键通信域）PP=1、EP=1、CP=16 | 0.99% | 仅非关键通信域CP（Context Parallelism，上下文并行）开启重执行，对端到端性能影响较小。 |
| Qwen3-moe-235B（运行在128die规模集群） | TP=8（关键通信域）PP=1、EP=64 | -0.1% | 仅非关键通信域EP（Expert Parallelism，专家并行）开启重执行，对端到端性能影响较小。 |

- 关注点2：关键通信域的通信展开和计算能否重叠。如果关键通信域开了重执行，那么该通信域的性能一定会有劣化；但是该劣化是否会引发整网劣化，还需要看该关键通信域的AI CPU展开是否能够与计算重叠（overlap）。单个通信域开了重执行后，最大的差异是由异步展开模式变为同步展开模式。通信展开时间能否被计算掩盖，是决定该通信域是否对端到端性能有影响的关键因素，具体需要结合计算算子的情况（模型结构）进行分析。

[图: 重执行开启后算子展开方式变化示意图，展示不开重执行时AI CPU Kernel异步展开与STARS执行，以及开重执行时同步展开带来的前同步和后同步开销]

| 模型 | 切分方式 | 劣化比例 | 说明 |
|---|---|---|---|
| DeepSeekV3（运行在64die规模集群） | EP=64 | 0.06% | 关键通信域EP开重执行，但该模型计算时间长，重执行开销能够被计算掩盖，整网端到端性能劣化不严重。 |
| qwen3-moe-30b（运行在64die规模集群） | EP=64 | 3% | 关键通信域EP开重执行，重执行开销不能被计算掩盖，整网端到端有性能劣化。 |

由此可见，模型端到端影响因素与模型结构强相关，重执行对整网性能的影响需要根据实际情况进行评估。

---

# 12 推荐业务配置

本节分别针对Atlas A3训练系列产品/Atlas A3推理系列产品、Atlas A2训练系列产品的常见业务场景，提供推荐的业务配置。

> 须知：本节仅给出了推荐配置环境变量的功能说明和配置示例，详细使用说明可参见"环境变量参考"的"集合通信"章节。

### Atlas A3训练系列产品/Atlas A3推理系列产品

- 训练场景

| 环境变量 | 配置说明 |
|---|---|
| HCCL_CONNECT_TIMEOUT | 配置socket建链超时等待时间，默认值为120，单位s。该场景下，建议根据网络规模大小适当调整建链超时等待时间。`export HCCL_CONNECT_TIMEOUT=1200` |
| HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议保持默认值"AI_CPU"，代表通信算法的编排展开位置为AI CPU。`export HCCL_OP_EXPANSION_MODE="AI_CPU"` |

- 推理场景

| 部署方式 | 环境变量 | 配置说明 |
|---|---|---|
| Prefill-Decode混合部署 | HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议配置为"AIV"，代表通信算法的编排展开位置为Vector Core。`export HCCL_OP_EXPANSION_MODE="AIV"` |
| | HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |
| Prefill-Decode分离部署 | HCCL_INTRA_ROCE_ENABLE | 仅使用LLM-DataDist作为集群管理组件的场景下，建议通过此环境变量配置超节点内使用RoCE链路进行通信；非LLM-DataDist场景，无需配置。`export HCCL_INTRA_ROCE_ENABLE=1` |
| | HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议配置为"AIV"，代表通信算法的编排展开位置为Vector Core。`export HCCL_OP_EXPANSION_MODE="AIV"` |
| | HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |

- 强化学习训推一体

| 环境变量 | 配置说明 |
|---|---|
| HCCL_CONNECT_TIMEOUT | 配置socket建链超时等待时间，默认值为120，单位s。该场景下，建议根据网络规模大小适当调整建链超时等待时间。`export HCCL_CONNECT_TIMEOUT=1200` |
| HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议保持默认值"AI_CPU"，代表通信算法的编排展开位置为AI CPU。`export HCCL_OP_EXPANSION_MODE="AI_CPU"`。需要注意：针对推理通信域，需要通过通信域级别的配置参数将推理通信域的算法编排展开位置设置为"Vector Core"，针对PyTorch框架网络，可通过"hccl_op_expansion_mode"参数配置，配置方法如下：`options = torch_npu._C._distributed_c10d.ProcessGroupHCCL.Options()` `options.hccl_config ={"hccl_op_expansion_mode":3}` `torch.distributed.init_process_group(backend="hccl", pg_options=options)` |
| HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |

### Atlas A2训练系列产品

- 训练场景

| 环境变量 | 配置说明 |
|---|---|
| HCCL_CONNECT_TIMEOUT | 配置socket建链超时等待时间，默认值为120，单位s。该场景下，建议根据网络规模大小适当调整建链超时等待时间。`export HCCL_CONNECT_TIMEOUT=1200` |
| HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议保持默认值"HOST"，代表通信算法的编排展开位置为Host侧CPU。`export HCCL_OP_EXPANSION_MODE="HOST"` |
| HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |

- 推理场景

| 环境变量 | 配置说明 |
|---|---|
| HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议保持默认值"HOST"，代表通信算法的编排展开位置为Host侧CPU。`export HCCL_OP_EXPANSION_MODE="HOST"` |
| HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |

- 强化学习训推一体

| 环境变量 | 配置说明 |
|---|---|
| HCCL_CONNECT_TIMEOUT | 配置socket建链超时等待时间，默认值为120，单位s。该场景下，建议根据网络规模大小适当调整建链超时等待时间。`export HCCL_CONNECT_TIMEOUT=1200` |
| HCCL_OP_EXPANSION_MODE | 配置通信算法的编排展开位置。该场景下建议保持默认值"HOST"，代表通信算法的编排展开位置为Host侧CPU。`export HCCL_OP_EXPANSION_MODE="HOST"` |
| HCCL_DETERMINISTIC | 是否开启确定性计算，用户可以根据使用场景选择开启或关闭，默认值为false，代表关闭确定性计算。`export HCCL_DETERMINISTIC=false` |

---

# 13 通信算子支持情况

本节提供Atlas A3训练系列产品/Atlas A3推理系列产品的训练与推理场景下常用通信算子的支持情况。

- 单算子零拷贝：为了降低内存拷贝开销，使得HCCL可以直接对业务传入的内存进行操作，提升通信性能。
- 通信算子重执行：网络故障导致通信闪断时，HCCL会尝试重新执行此通信算子，提升通信稳定性。
- 确定性计算：归约类通信算子在相同的硬件和输入下，多次执行将产生相同的输出。

> 说明：本节表格中"V"代表支持，"x"代表不支持，"NA"代表不涉及。

### 训练场景常用通信算子

| 算子类型 | 执行模式 | 通信算法编排展开位置 | 单算子零拷贝 | 确定性计算 | 算子重执行 | 节点内通信 | 超节点内通信 | 超节点间通信 |
|---|---|---|---|---|---|---|---|---|
| AllGather | 单算子 | AI_CPU | V | NA | V | V | V | V |
| AllGatherV | 单算子 | AI_CPU | x | NA | V | V | V | V |
| AllReduce | 单算子 | AI_CPU | V | V | V | V | V | V |
| AlltoAll | 单算子 | AI_CPU | x | NA | V | V | V | V |
| AlltoAllV | 单算子 | AI_CPU | x | NA | V | V | V | V |
| AlltoAllVC | 单算子 | AI_CPU | x | NA | V | V | V | V |
| Broadcast | 单算子 | AI_CPU | V | NA | V | V | V | V |
| Reduce | 单算子 | AI_CPU | x | V | V | V | V | V |
| ReduceScatter | 单算子 | AI_CPU | V | V | V | V | V | V |
| ReduceScatterV | 单算子 | AI_CPU | x | V | V | V | V | V |
| Send | 单算子 | AI_CPU | x | NA | V | V | V | V |
| Recv | 单算子 | AI_CPU | x | NA | V | V | V | V |
| BatchSendRecv | 单算子 | AI_CPU | x | NA | V | V | V | V |

### 推理场景常用通信算子

- 通信算法编排展开位置为AI CPU

| 算子类型 | 执行模式 | 通信算法编排展开位置 | 单算子零拷贝 | 确定性计算 | 算子重执行 | 节点内通信 | 超节点内通信 | 超节点间通信 |
|---|---|---|---|---|---|---|---|---|
| AllGather | 单算子 | AI_CPU | V | NA | V | V | V | V |
| | 图模式（Ascend IR） | AI_CPU | NA | NA | V | V | V | V |
| | 图捕获（aclgraph） | AI_CPU | NA | NA | V | V | V | V |
| AllReduce | 单算子 | AI_CPU | V | V | V | V | V | V |
| | 图模式（Ascend IR） | AI_CPU | NA | V | V | V | V | V |
| | aclgraph | AI_CPU | NA | V | V | V | V | V |
| AlltoAllV | 单算子 | AI_CPU | x | NA | V | V | V | V |
| | 图模式（Ascend IR） | AI_CPU | NA | NA | V | V | V | V |
| | 图捕获（aclgraph） | AI_CPU | NA | NA | V | V | V | V |
| ReduceScatter | 单算子 | AI_CPU | V | V | V | V | V | V |
| | 图模式（Ascend IR） | AI_CPU | NA | V | V | V | V | V |
| | 图捕获（aclgraph） | AI_CPU | NA | V | V | V | V | V |

- 通信算法编排展开位置为Vector Core

| 算子类型 | 执行模式 | 通信算法编排展开位置 | 确定性计算 | 节点内通信 | 超节点内通信 |
|---|---|---|---|---|---|
| AllGather | 单算子 | AIV | NA | V | V |
| | 图模式（Ascend IR） | AIV | NA | V | V |
| | 图捕获（aclgraph） | AIV | NA | V | V |
| AllReduce | 单算子 | AIV | V | V | V |
| | 图模式（Ascend IR） | AIV | V | V | V |
| | 图捕获（aclgraph） | AIV | V | V | V |
| AlltoAllV | 单算子 | AIV | NA | V | V |
| | 图模式（Ascend IR） | AIV | NA | V | V |
| | 图捕获（aclgraph） | AIV | NA | V | V |
| ReduceScatter | 单算子 | AIV | V | V | V |
| | 图模式（Ascend IR） | AIV | V | V | V |
| | 图捕获（aclgraph） | AIV | V | V | V |

> 说明：AIV模式在小数据量通信时性能较优，主要用于推理场景，此模式下：
> - 单算子零拷贝会引入执行时内存协商，导致通信时延变大，所以当前AIV模式下不支持单算子零拷贝。
> - 重执行特性会增加执行耗时，所以AIV模式不支持算子重执行。
> - 仅支持超节点内通信，不支持超节点间通信。

---

# 14 集合通信算法介绍

## 14.1 算法简介

针对同一个通信算子，随着网络拓扑、通信数据量、硬件资源等的不同，往往会采用不同的通信算法，从而最大化集群通信性能。HCCL提供了Mesh、Ring、Recursive Halving-Doubling（RHD）、Pairwise、Pipeline等拓扑算法用于Server内、Server间和超节点间的集合通信。

### Server内通信算法

HCCL通信域Server内支持Mesh、Ring、Double-Ring和Star算法，具体使用的算法根据硬件拓扑自动选择，用户无须配置也不支持配置。

### Server间/超节点间通信算法

HCCL通信域Server间/超节点间支持如下算法的自适应选择，自适应算法会根据产品形态、数据量和Server个数进行选择，用户默认无需配置。

- Ring算法：基于环结构的通信算法，通信步数多（线性复杂度），时延相对较高，但通信关系简单，受网络拥塞影响较小。适合通信域内Server个数较少、通信数据量较小、网络存在明显拥塞、且Pipeline算法不适用的场景。
- RHD（Recursive Halving-Doubling）算法：递归二分和倍增算法，通信步数少（对数复杂度），时延相对较低，但在非2次幂节点规模下会引入额外的通信量。适合通信域内Server个数是2的整数次幂且Pipeline算法不适用的场景，或Server个数不是2的整数次幂但通信数据量较小的场景。
- NHR（Nonuniform Hierarchical Ring）算法：非均衡的层次环算法，通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且Pipeline算法不适用的场景。
- NB（Nonuniform Bruck）算法：非均匀的数据块通信算法，通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且Pipeline算法不适用的场景。
- Pipeline算法：流水线并行算法，可并发使用Server内与Server间的链路，适合通信数据量较大且通信域内每机包含多卡的场景。
- Pairwise算法：逐对通信算法，仅用于AlltoAll、AlltoAllV与AlltoAllVC算子，通信步数较多（线性复杂度），时延相对较高，但可以避免网络中出现一打多现象（指一个rank通过同一个端口给多个rank发送数据），适合通信数据量较大、需要规避网络一打多的场景。
- AHC（Asymmetric Hierarchical Concatenate）算法：层次化集合通信算法，仅用于ReduceScatter、AllGather、AllReduce算子，适用于通信域内NPU分布存在多个层次、同时支持多个层次间NPU对称或者非对称分布的场景，当通信域内层次间存在带宽收敛时相对收益会更好。

> 说明
> - 开发者若想指定Server间或超节点间通信算法，可通过环境变量HCCL_ALGO进行设置。需要注意，若通过环境变量HCCL_ALGO指定了Server间或超节点间通信算法，通信算法的自适应选择功能不再生效，以用户指定的算法为准。
> - 每种算法支持的算子以及产品型号可参见环境变量HCCL_ALGO。

### 耗时评估

HCCL采用alpha-beta模型（Hockney）进行性能评估，算法耗时计算用到的变量定义如下：

- alpha：节点间的固定时延，单位为s，由通信硬件设备与底层软件栈决定。
- beta：每byte数据传输耗时，单位为s/byte，由通信链路能力决定。
- n：节点间通信的数据大小，单位为byte，由通信算法决定。
- gamma：每byte数据规约计算耗时，单位为s/byte，由计算硬件设备能力决定。
- p：通信域节点个数，影响通信步数，由通信算子所在通信域决定。

单步传输并规约计算n byte数据的耗时为：D = alpha + n*beta + n*gamma。

集合通信算法通过结合网络拓扑，优化通信关系和通信步骤，减少通信次数以减少固定时延，减少实际通信数据量以减少传输耗时和计算耗时，从而达到优化集合通信性能的目的。

### 分级通信原理

HCCL通常按节点内/节点间分为两级拓扑，或按节点内/节点间/超节点间分为三级拓扑，分级执行集合通信，不同的层级间链路带宽不同。分级通信可以使通信任务编排与网络拓扑亲和，从而最大化利用链路能力。

以Atlas A2训练系列产品的单算子模式、节点内/节点间两级拓扑为例，各集合通信算子的具体分级通信过程如下表所示：

| 集合通信算子 | 阶段一 | 阶段二 | 阶段三 |
|---|---|---|---|
| ReduceScatter | Server间ReduceScatter | Server内ReduceScatter | / |
| ReduceScatterV | Server间ReduceScatterV | Server内ReduceScatterV | / |
| AllGather | Server内AllGather | Server间AllGather | / |
| AllGatherV | Server内AllGatherV | Server间AllGatherV | / |
| AllReduce | Server内ReduceScatter | Server间AllReduce | Server内AllGather |
| Scatter | Server间Scatter | Server内Scatter | / |
| Broadcast | Server内Scatter | Server间Broadcast | Server内AllGather |
| Reduce | Server内ReduceScatter | Server间Reduce | Server内Gather（HCCL未提供Gather算子，Gather操作与AllGather操作的区别是仅将结果发送到root节点的输出buffer。） |
| AlltoAll | Server内AlltoAll | Server间AlltoAll | / |
| AlltoAllV | Server内AlltoAllV | Server间AlltoAllV | / |
| AlltoAllVC | Server内AlltoAllVC | Server间AlltoAllVC | / |

详细分级通信流程示例可参见14.11 分级通信原理。

## 14.2 Mesh

算法描述：Mesh是FullMesh互联拓扑内的基础算法，是NPU之间的全连接，任意两个NPU之间可以直接进行数据收发。

[图: FullMesh网络拓扑示意图，展示NPU1-NPU8的全连接结构]

Mesh算法实现AllReduce算子的流程如下图所示，每个NPU并发的使用多路HCCS链路从对端读取或者写入数据，使双工互联链路的双向带宽同时得到利用。

[图: Mesh算法AllReduce流程示意图，展示4个Rank的ReduceScatter和AllGather过程]

Mesh算法的时间复杂度是O(1)。

耗时计算：

| 操作 | 耗时 |
|---|---|
| Scatter | alpha + (1/p)*n*beta |
| Gather | alpha + (1/p)*n*beta |
| Broadcast | 实现为Scatter+AllGather，耗时为 2*alpha + (2/p)*n*beta |
| Reduce | 实现为ReduceScatter+Gather，耗时为 2*alpha + (2/p)*n*beta + ((p-1)/p)*n*gamma |
| ReduceScatter | alpha + (1/p)*n*beta + ((p-1)/p)*n*gamma |
| AllGather | alpha + (1/p)*n*beta |
| AllReduce | 实现为ReduceScatter+AllGather，耗时为 2*alpha + (2/p)*n*beta + ((p-1)/p)*n*gamma |

## 14.3 Ring

算法描述：Ring算法，所有的NPU以环形相连，每张卡都有左手卡与右手卡，一个负责数据接收，一个负责数据发送，循环完成梯度累加，再循环做参数同步。

[图: Ring环形拓扑结构示意图，展示A->B->C->D->A的环形连接]

Ring算法适用于"星型"或"胖树"拓扑互联，其特点是通过Ring环将所有NPU设备的单端口双工链路串联起来。

Ring算法实现AllReduce算子的流程如下图所示，每一步依次给下游发送对应的数据块，沿着环转一圈之后完成ReduceScatter阶段，再沿环转一圈完成AllGather阶段。

[图: Ring算法AllReduce流程示意图，展示4个Rank的ReduceScatter和AllGather详细数据交换过程]

Ring算法的时间复杂度是O(n-1)，n为Ring环上的NPU设备个数。

耗时计算：整体思路为将所有参与的节点构成环，每个节点只和左右节点通信，如果节点数为p，则需要的通信次数为p-1，每次交换1/p的数据。

| 操作 | 耗时 |
|---|---|
| Scatter | (p-1)*alpha + ((p-1)/p)*n*beta |
| Gather | (p-1)*alpha + ((p-1)/p)*n*beta |
| Broadcast | 实现为Scatter+AllGather，耗时为 2(p-1)*(alpha+n*beta) = 2(p-1)*alpha + 2(p-1)*n*beta |
| Reduce | 实现为ReduceScatter+Gather，耗时为 (p-1)*(alpha+n*beta+n*gamma)+(p-1)*(alpha+n*beta+n*gamma) = 2(p-1)*alpha + 2(p-1)*n*beta + (p-1)*n*gamma |
| ReduceScatter | (p-1)*(alpha + (n/p)*beta + (n/p)*gamma) = (p-1)*alpha + ((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |
| AllGather | (p-1)*(alpha + (n/p)*beta) = (p-1)*alpha + ((p-1)/p)*n*beta |
| AllReduce | 实现为ReduceScatter+AllGather，耗时为 2(p-1)*alpha + 2*((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |

## 14.4 RHD

算法描述：当组网增大时，例如增大至4K个rank的场景，Mesh很难组成4K个rank的全连接网络（全连接一个时钟周期就可以完成操作），且资源开销（链路资源，交换资源，同步资源）太大，还可能存在算力和资源开销不匹配的问题。Ring在这种情况下虽然节省资源（只用左手卡和右手卡进行一次收发），但是环内要做太多次，流转太慢。大型规模集群运算有服务器内数据量庞大。Ring环极长的特点，Ring的这种切分数据块的方式就不再占优势。

RHD（Recursive Halving-Doubling）算法通过递归加倍及递归折半方式完成NPU间的数据交换，相对Mesh资源消耗较小，相对Ring效率会更高。

[图: RHD算法AllReduce流程示意图，展示5个Rank的rank1数据合并、两两交换和、数据拼接、AllGather过程]

RHD算法同样适用于"星型"或"胖树"拓扑互联，算法的时间复杂度是ceil(log2(N))。

耗时计算：Recursive Halving-Doubling为递归二分和倍增算法，对于2的整数次幂，使用Vector/Distance Halving/Doubling策略，对于非2的整数次幂，划分为2r（part1）和p-2r两部分（r = p - 2^floor(log(p))），先将part1部分合并为r，使得剩余的rank之和为p-r（block），再执行2的整数次幂的HD（Halving-Doubling）算法，最后再在part1部分恢复出2r，得到最终结果。

| 操作 | 耗时 |
|---|---|
| Broadcast | ceil(log(p))*(alpha + n*beta) |
| ReduceScatter | 2的整数次幂：log(p)*alpha + ((p-1)/p)*n*beta + ((p-1)/p)*n*gamma。非2的整数次幂：涉及多步计算，详见文档。 |
| AllGather | 耗时同ReduceScatter，无gamma相关部分。 |
| AllReduce | ReduceScatter+AllGather。2的整数次幂：2*log(p)*alpha + 2*((p-1)/p)*n*beta + ((p-1)/p)*n*gamma。非2的整数次幂总耗时：(2*floor(log(p))+2)*alpha + (2*((p'-1)/p')+2)*n*beta + ((p'-1)/p'+1)*n*gamma，其中p' = 2^floor(log(p))。 |
| Reduce | 当前实现为ReduceScatter+Gather。2的整数次幂：2*log(p)*alpha + 2*((p-1)/p)*n*beta + ((p-1)/p)*n*gamma。非2的整数次幂总耗时：(2*floor(log(p))+1)*alpha + (2*((p'-1)/p')+1)*n*beta + ((p'-1)/p'+1)*n*gamma，其中p' = 2^floor(log(p))。 |

## 14.5 Pairwise

算法描述：通常每个节点只有一个RDMA网口，如果在RDMA链路上使用Mesh算法完成AlltoAll，存在同时从多个节点接收数据、向多个节点发送数据的"多打多"问题，多个流在同一条链路上肆意争抢资源，可能反而导致整体性能下降。

Pairwise算法是Mesh算法的分步执行版本，通过合理的规划，将通信分解成多个步骤，每一步只从一个节点接收数据、向一个节点发送数据。比如对于rankid为i的节点，第一步从(i-1)节点接收数据，向(i+1)节点发送数据；第二步从(i-2)节点接收数据，向(i+2)节点发送数据......以此类推。

[图: Pairwise算法步骤示意图，展示N+1个节点在第一步和第二步的通信对]

耗时计算：定义节点i需要给节点j发送的数据大小为n_ij。对于第k步，节点i发送大小为n_{i,i+k}的数据给节点i+k，则第k步的耗时为：alpha + beta * max_i(n_{i,i+k})。完成整个Pairwise的耗时为：(p-1)*alpha + beta * sum_k(max_i(n_{i,i+k}))。

## 14.6 Star

算法描述：Star算法适用于有根节点的通信操作（如Broadcast、Reduce、Gather、Scatter等），利用星型拓扑或全连接拓扑一步完成通信操作。以Broadcast算子为例，Star算法实现如下图所示，根节点Root利用星型拓扑从其他各节点收集数据。

[图: Star算法星型拓扑示意图，展示Root节点与A-H节点的星型连接]

耗时计算：定义每个非根节点与根节点的通信数据大小为n，那么完成整个Star算法的通信耗时为：alpha + beta*n。

## 14.7 NHR

算法描述：在规模较大、通信节点数量较多的组网中，由于Ring算法的通信步数与通信节点规模成正相关，所以容易出现较大延迟，因此Ring算法的切分数据块的方式对小数据包场景不友好，更适合大数据包传输场景。在集群规模不是2的整数次幂时，使用RHD算法会引入额外的通信步数和开销，产生N-1规模集群下的通信性能比N规模集群通信性能还差的现象。此外，由于RHD算法每个通信阶段的对象在变化，导致通信链路也发生变化，这在大流量场景下可能会引起交换机的流量冲突，从而导致带宽下降。

NHR（Nonuniform Hierarchical Ring，非均衡的层次环）算法对N个节点构建成树构建N棵生成树，通过N个生成树构建出最优通信关系。树的深度（即通信步数）是ceil(log2(N))，并通过重排数据片编号聚合发送，保证通信算法的理论性能最优。

该算法的最大通信流量集中在物理位置相近的节点之间，能够有效利用物理距离带来的性能优势，减少流量冲突。同时无论集群规模是否是2的幂次，NHR算法都能充分利用链路资源。对于小数据包通信场景，NHR算法进一步优化，采用N个节点仅构建1棵树的方法，从而减少网络中数据包数量和芯片并发执行的任务数，提升通信效率。

[图: rank size为4时NHR算法通信过程示意图]

[图: rank size为5时NHR算法通信过程示意图]

NHR算法同样适用于"星型"或"胖树"拓扑互联，算法的时间复杂度是ceil(log2(N))。

耗时计算：NHR为非均衡的层次环算法，当集群规模为2的整数次幂或者非2的整数次幂时，算法时间复杂度均为O(ceil(log2(N)))。如果节点数为p，则需要的通信次数为ceil(log2(p))，对ReduceScatter算子，第一步交换n/2的数据，每次通信数据量减半，最后一步交换1份数据，AllGather算子的收发关系则完全相反。

| 操作 | 耗时 |
|---|---|
| ReduceScatter | ceil(log2(p))*alpha + ((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |
| AllGather | 耗时同ReduceScatter，无gamma相关部分。ceil(log2(p))*alpha + ((p-1)/p)*n*beta |
| AllReduce | 实现为ReduceScatter+AllGather：2*ceil(log2(p))*alpha + 2*((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |
| Scatter | ceil(log2(p))*alpha + ((p-1)/p)*n*beta |
| Broadcast | 实现为Scatter+AllGather：2*ceil(log2(p))*alpha + 2*((p-1)/p)*n*beta |

## 14.8 NB

算法描述：集合通信中，Ring算法通信步数为O(N-1)，其中N表示参与集合通信的rank数量，随着网络规模的增加，通信开销也会显著增加。RHD算法虽然将通信步数减少到了log2(N)，但在rank数量不是2的幂时，需要进行数据合并操作，导致通信数据量增加。而NB算法（Nonuniform Bruck，非均匀的数据块通信算法）通过动态调整步长的多重环状结构，实现不同rank数量下均保持通信步数为ceil(log2(N))，同时避免了额外的通信数据量增长。

[图: rank size为4时NB算法通信过程示意图]

[图: rank size为5时NB算法通信过程示意图]

对于ReduceScatter和AllGather算子，通信步数均为ceil(log2(N))。

- 针对ReduceScatter算子，每一步通信中，每张卡向通信步长为2^k（0<=k<ceil(log2(N))）的目标卡发送数据，每步发送数据的份数为floor((N-1+2^k)/2^(k+1))。
- 对于AllGather算子，每一步的通信步长递减，而通信数据量递增。当卡数不是2的幂时，最后一步的通信数据量为N - 2^floor(log2(N))。

NB算法同样适用于"星型"和"胖树"拓扑，算法的时间复杂度为ceil(log2(N))。

耗时计算：

| 操作 | 耗时 |
|---|---|
| ReduceScatter | ceil(log(p))*alpha + ((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |
| AllGather | ceil(log(p))*alpha + ((p-1)/p)*n*beta |
| AllReduce | 实现为ReduceScatter+AllGather，耗时为：2*ceil(log(p))*alpha + 2*((p-1)/p)*n*beta + ((p-1)/p)*n*gamma |
| Scatter | ceil(log(p))*alpha + ((p-1)/p)*n*beta |
| Broadcast | 实现为Scatter+AllGather，耗时为：2*ceil(log(p))*alpha + 2*((p-1)/p)*n*beta |

## 14.9 AHC

算法描述：当集群网络存在层次化特征且层次间存在带宽收敛时，集合通信面临两大技术挑战：一是由于不同区域间存在带宽收敛问题，传统的单层集合通信算法性能下降；二是不同区域的计算单元数量不同（即卡数非对称），这使得常规的层次化算法不再适用。例如，在一个集群中，同一通信域可能横跨两个超节点，且两个超节点中的卡数量不一致（比如一个超节点中有64张卡，另一个则有128张卡），这种情况对集合通信算法的性能带来巨大挑战。

[图: AHC基于逻辑同号卡实现AllReduce过程（5个rank，2+3两个分组）]

本算法的核心思想是基于拓扑将通信域内NPU及各NPU上的数据重新分组，组内充分利用高速网络带宽，组间实现基于"逻辑同号卡"的非对称拼接。具体流程参考上图，实现分为如下三个步骤：

1. 基于物理拓扑对计算单元分组。临近的NPU划分为一个group，各group内卡数无需一致，group间带宽相比group内可能存在收敛。
   a. 求解所有分组数的最小公倍数LCM，若G个分组则将数据划分为LCM*G个切片。如上图所示，分组为2和3，则LCM=6，G=2，将数据切分成12份切片。
   b. 每个分组内并行执行标准的ReduceScatter。
2. 划分"逻辑同号卡"，基于逻辑同号卡实现组间allreduce。
   a. 将每个group中待执行reduce操作的数据，按照group内各NPU卡间的数据边界进行切分，形成若干不均匀的数据块。
   b. 每个group中的每份数据，在其他所有group中各有一份对应的、大小相同的数据。按照数据对应关系，group间的NPU也存在对应关系，我们称存在对应关系的NPU为"逻辑同号卡"。
   c. 在逻辑同号卡间执行AllReduce操作。
3. 各group内的NPU之间执行AllGather操作。

具体的组内和组间的ReduceScatter、AllGather、AllReduce等操作，其实现算法可以是任意已知的算法，如NB、NHR、Ring等，当前AHC算法内部根据具体场景和策略选择性能更优的拼接算法类型。

耗时计算：当组内和组间都采用NB算法时，AllReduce算子的算法耗时如下：

| 操作 | 耗时 |
|---|---|
| AllReduce | 2*(ceil(log(m+d)) + ceil(log(G)))*alpha + 2*((m+d-1)/(m+d) + (G-1)*C/(G*m))*n*beta + ((m+d-1)/(m+d) + (G-1)/(G*m))*n*gamma。其中m为最小分组数、m+d为最大分组数、G为分组数、C为组间带宽相对于组内带宽的收敛比。 |

## 14.10 Pipeline

算法描述：为降低网络流量冲突，AI计算集群中常常采用分级网络架构，即Server内通过直连电缆互联，Server间同号卡通过交换机互联。为适配这种网络拓扑结构，集合通信采用分级算法策略，即将全局通信操作分解为多个层级的局部操作，利用分阶段、分层递进的方法以提升通信效率。

以AllGather算子为例，Server间先执行一次同号卡间的AllGather操作，再在Server内执行一次AllGather操作，以完成整个集群的数据集合过程。然而，这种方式会造成一定程度上的链路带宽浪费：当Server间数据传输时，Server内的链路处于空闲状态，未能充分利用带宽资源。

为了解决这一问题，HCCL采用了细粒度的分级流水（pipeline）算法，通过挖掘通信算法本身的数据依赖，并且结合流水并行的方式，解决带宽利用率不足的问题。

[图: AllGather算子Pipeline算法示意图]

[图: AllGather算子Pipeline算法时序示意图]

耗时计算：

| 操作 | 耗时 |
|---|---|
| ReduceScatter | max(s/p * beta_inter + alpha_inter, s/p * beta_intra + alpha_intra) * (p_inter - 1) + s/p * beta_intra + alpha_intra |
| AllGather | max(s/p * beta_inter + alpha_inter, s/p * beta_intra + alpha_intra) * (p_inter - 1) + s/p * beta_intra + alpha_intra |
| AllReduce | 2 * (max(s/p * beta_inter + alpha_inter, s/p * beta_intra + alpha_intra) * (p_inter - 1) + s/p * beta_intra + alpha_intra) |

其中p表示完成集合通信的总卡数，p_inter表示server数，s表示集合通信操作总数据量，beta_inter表示server间链路每byte数据传输耗时，beta_intra表示server内链路每byte数据传输耗时，alpha_inter表示server间链路传输固定耗时，alpha_intra表示server内链路传输固定耗时。

## 14.11 分级通信原理

下面以通信算子ReduceScatter、AllGather、AllReduce为例介绍分级通信的流程。

### ReduceScatter

ReduceScatter算子要求第i个rank最终得到第i份规约结果，为了保证Server间通信数据块的连续性，首先在Server间执行ReduceScatter操作，再在Server内执行ReduceScatter操作。

[图: ReduceScatter算子分级通信流程示意图，展示2个Server各3个Rank的Server间和Server内ReduceScatter过程]

### AllGather

AllGather算子要求第i个rank的输入数据出现在结果的第i个位置上，为了保证Server间通信数据块的连续性，首先在Server内执行AllGather操作，再在Server间执行AllGather操作。

[图: AllGather算子分级通信流程示意图，展示2个Server各3个Rank的Server内和Server间AllGather过程]

### AllReduce

AllReduce算子的输出是完整的规约结果，因此虽然拆解为了ReduceScatter和AllGather两个阶段，但不需要严格遵循ReduceScatter和AllGather的语义，可以将较大数据量的通信过程放在带宽更高的Server内，即先在Server内执行ReduceScatter操作，然后在Server间执行AllReduce操作，最后在Server内执行AllGather操作。

[图: AllReduce算子分级通信流程示意图，展示2个Server各3个Rank的Server内ReduceScatter、Server间AllReduce、Server内AllGather过程]

---

# 15 系统约束与限制

### 公共约束

- 集合通信不支持应用于昇腾虚拟化实例场景。昇腾虚拟化实例是指通过资源虚拟化的方式将物理机或虚拟机配置的NPU切分成多个vNPU（虚拟NPU实例）挂载到目标环境使用，虚拟化管理方式能够实现统一不同规格资源的分配和回收处理，满足多用户反复申请/释放的资源操作请求。
- 容器场景下部署集合通信相关业务时，各服务器仅支持单容器多进程部署，不支持多容器部署。

### Atlas A3训练系列产品/Atlas A3推理系列产品

- 若您的驱动固件是25.0.RC1或更高版本，支持单卡多进程的业务场景，即支持多个业务进程同时共用一个NPU。需要注意，多进程会对资源开销、通信性能有一定的影响，若同一个NPU上进程过多，可能会由于资源不足造成业务运行失败。若您的驱动固件不满足版本要求，会使用单进程运行。若通信算法的编排展开位置为AI_CPU（默认值），单卡进程并发数量不建议超过6个；若通信算法的编排展开位置为AIV，不建议单卡多进程并发执行，多个进程之间建议串行执行。请参考以上建议配置，否则存在任务死锁的风险。Atlas A3训练系列产品/Atlas A3推理系列产品的算法编排展开位置可通过环境变量"HCCL_OP_EXPANSION_MODE"设置。
- 建议每个超节点中的Server数量一致，每个Server中的昇腾AI处理器数量一致，若不一致，会造成性能劣化。
- 当通信算法采用默认的AI CPU模式时，单卡上的并发通信域数量不能超过6个，否则可能会因AI CPU核被占满而导致通信阻塞。

### Atlas A2训练系列产品

- 若您的驱动固件是25.0.RC1或更高版本，支持单卡多进程的业务场景，即支持多个业务进程同时共用一个NPU。若网络中存在MC2通算融合算子（计算和通信融合的算子，例如AllGatherMatmul、MatmulReduceScatter、AlltoAllAllGatherBatchMatMul等），则不支持单卡多进程的业务场景。需要注意，多进程会对资源开销、通信性能有一定的影响，若同一个NPU上进程过多，可能会由于资源不足造成业务运行失败。若您的驱动固件不满足版本要求，会使用单进程运行。若通信算法的编排展开位置为HOST（默认值），单卡进程并发数量不建议超过8个；若通信算法的编排展开位置为AIV，不建议单卡多进程并发执行，多个进程之间建议串行执行。请参考以上建议配置，否则存在任务死锁的风险。Atlas A2训练系列产品的算法编排展开位置可通过环境变量"HCCL_OP_EXPANSION_MODE"设置。
- 单Server场景，对参与集合通信的昇腾AI处理器数量无限制；Server集群场景要求参与集合通信的昇腾AI处理器数量为（1~8）* n（n为参与训练的Server个数）。建议每个Server中参与集合通信的昇腾AI处理器数量保持一致，若不一致，会造成性能劣化。

### Atlas训练系列产品

- 不支持单卡多进程的业务场景，即不支持多个业务进程同时共用一个NPU。
- 单Server场景下，要求实际参与集合通信的昇腾AI处理器数目只能为1/2/4/8，且0-3卡和4-7卡各为一个组网，使用2张卡或4张卡训练时，不支持跨组网创建设备集群；Server集群场景下，要求参与集合通信的昇腾AI处理器数目只能为1*n、2*n、4*n、8*n（n为参与训练的Server个数），且n为2的指数倍情况下，集群性能最好，建议用户优先采用此种方式进行集群组网。

### Atlas 300I Duo推理卡

- 不支持单卡多进程的业务场景，即不支持多个业务进程同时共用一个NPU。
- 仅支持单Server场景，每个集合通信操作支持的最大NPU数量详见具体API。
- 通信域初始化需要在其他任何涉及Device内存申请的操作之前，否则可能因P2P内存不足导致初始化失败。
