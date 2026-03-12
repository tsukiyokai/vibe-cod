# HCCL集合通信用户指南

> 来源：CANN商用版8.0.RC3 (hiascend.com)

---

# HCCL概述

集合通信库HCCL(Huawei Collective Communication Library)是基于昇腾AI处理器的高性能集合通信库，提供单机多卡以及多机多卡间的数据并行、模型并行集合通信方案。

HCCL支持AllReduce、Broadcast、AllGather、ReduceScatter、AlltoAll等通信原语，Ring、Mesh、Halving-Doubling(HD)等通信算法，基于HCCS、RoCE和PCIe高速链路实现集合通信。

## 支持的产品型号

- Atlas训练系列产品
- Atlas A2 训练系列产品
- Atlas 300I Duo推理卡

## HCCL在系统中的位置

HCCL提供了Python与C++两种语言的接口，其中Python语言的接口用于实现图模式下的框架适配，例如TensorFlow网络基于HCCL的Python API实现分布式优化；C语言接口用于实现单算子模式下的框架适配，例如HCCL单算子API嵌入到PyTorch后端代码中，PyTorch用户直接使用PyTorch原生集合通信API，即可实现分布式能力。

## HCCL软件架构

集合通信库软件架构分为三层：

- **适配层**：图引擎与单算子适配，进行通信切分寻优等操作，提供通信域管理及通信算子接口。
- **集合通信业务层**：包含通信框架与通信算法两个模块
  - 通信框架：负责通信域管理，通信算子的业务串联，协同通信算法模块完成算法选择，协同通信平台模块完成资源申请并实现集合通信任务的下发。
  - 通信算法：作为集合通信算法的承载模块，提供特定集合通信操作的资源计算，并根据通信域信息完成通信任务编排。
- **集合通信平台层**：提供NPU之上与集合通信关联的资源抽象，并提供集合通信的相关维护、测试能力。

## 集合通信流程

分布式场景中，HCCL提供了服务器间高性能集合通信功能。服务器间通信过程分为四个阶段：

1. **通信域初始化**：获取必要的集合通信配置参数并初始化通信域。通信初始化阶段不涉及NPU设备之间的交互。

2. **建立通信连接**：建立socket连接并交换通信两端的通信参数和内存信息。HCCL根据用户提供的集群信息并结合网络拓扑与其他NPU设备建链，交换用于通信的参数信息。如果在建链超时时间内未得到其他NPU设备的及时响应，会上报建链超时错误并退出业务进程。

3. **执行通信操作**：通过"等待-通知"机制同步设备执行状态，传递内存数据。HCCL会将通信算法编排、内存访问等任务通过Runtime下发给昇腾设备的任务调度器，设备根据编排信息调度并执行任务。

4. **通信域销毁**：销毁通信域，释放通信资源。

---

# 术语与相关概念

为了您有更好的阅读体验，使用本文档前请先了解如下术语、缩略语与基本概念。

## 表1术语与相关概念

| 名称    | 说明                                                                                                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| NPU     | Neural Network Processing Unit，神经网络处理单元。采用"数据驱动并行计算"的架构，特别擅长处理视频、图像类的海量多媒体业务数据，专门用于处理人工智能应用中的大量计算任务。 |
| HCCL    | Huawei Collective Communication Library，华为集合通信库。提供单机多卡以及多机多卡间的数据并行、模型并行集合通信方案。                                                    |
| HCCS    | Huawei Cache Coherence System，华为缓存一致性系统。HCCS用于CPU/NPU之间的高速互联。                                                                                       |
| HCCP    | Huawei Collective Communication adaptive Protocol，集合通信适配协议。提供跨NPU设备通信能力，向上屏蔽具体通讯协议差异。                                                   |
| TOPO    | 拓扑、拓扑结构。一个局域网内或者多个局域网之间的设备连接所构成的网络配置或者布置。                                                                                       |
| PCIe    | Peripheral Component Interconnect Express，一种串行外设扩展总线标准，通常用于计算机系统中外设扩展使用。                                                                  |
| PCIe-SW | PCIe Switch，符合PCIe总线扩展的交换设备。                                                                                                                                |
| SDMA    | System Direct Memory Access，系统直接内存访问技术，简称DMA，允许外围设备直接访问系统内存，而不需要CPU的干预。                                                            |
| RDMA    | Remote Direct Memory Access，远程直接内存访问技术，它将数据直接从一台机器的内存传输到另一台机器，无需双方操作系统的介入，一般指可以跨过网络的内存访问方式。              |
| RoCE    | RDMA over Converged Ethernet，承载在融合以太网上的RDMA技术，即跨越以太网的RDMA通信方式。                                                                                 |
| AI节点  | 昇腾AI节点，又称昇腾AI Server，通常是8卡或16卡的昇腾NPU设备组成的服务器形态的统称。                                                                                      |
| AI集群  | 多个AI节点通过交换机(Switch)互联后用于分布式训练或推理的系统。                                                                                                           |
| 通信域  | 通信域是集合通信执行的上下文，管理对应的通信实体（例如一个NPU就是一个通信实体）和通信所需的资源。                                                                        |
| Rank    | 通信域中的每个通信实体称为一个Rank。                                                                                                                                     |

---

# 集合通信原语

集合通信是一个进程组的所有进程都参与的全局通信操作，其最为基础的操作有发送、接收、复制、节点间进程同步等，这些基本的操作经过组合构成了一组通信模板，也称为通信原语，这些通信原语通过相应的集合通信算子实现。

下面展开介绍HCCL提供的通信原语。

#### Broadcast

Broadcast操作是将通信域内root节点的数据广播到其他rank。

注意：全局只能有一个root节点。

#### AllGather

AllGather操作是将group内所有节点的输入按照rank id重新排序，然后拼接起来，再将结果发送到所有节点的输出。

针对AllGather操作，每个节点都接收按照rank id重新排序后的数据集合，即每个节点的AllGather输出都是一样的。

#### Scatter

Scatter操作是将通信域内root节点的数据均分并散布至其他rank。

注意：全局只能有一个root节点。

#### Reduce

Reduce操作是将所有rank的input相加（或prod/max/min）后，再把结果发送到root节点的输出buffer。

#### AllReduce

AllReduce操作是将通信域内所有节点的输入数据进行reduce操作后，再把结果发送到所有节点的输出buffer，其中reduce操作类型支持sum、prod、max、min。

注意：每个rank只能有一个输入。

#### ReduceScatter

ReduceScatter操作是将所有rank的输入相加（或其他操作）后，再把结果按照rank编号均匀分散到各个rank的输出buffer，每个进程拿到其他进程1/ranksize份的数据进行归约操作。

如下图所示，有rank0、rank1、rank2、rank3四个rank，每个rank的输入数据切分成4份，每个进程分别取每个rank的1/4数据进行sum操作（或其他操作），将结果发送到输出buffer。

#### AlltoAll

AlltoAll操作是向通信域内所有rank发送相同数据量的数据，并从所有rank接收相同数据量的数据。

AlltoAll操作将输入数据在特定的维度切分成特定的块数，并按顺序发送给其他rank，同时从其他rank接收输入，按顺序在特定的维度拼接数据。

#### AlltoAllV

AlltoAllV操作是向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。

---

# 分级通信原理

HCCL通常按Server内和Server间分为两级拓扑，分级执行集合通信，各集合通信算子的具体分级通信过程如下表所示：

## 表1集合通信算子分级通信过程

| 集合通信算子  | 阶段一                | 阶段二                | 阶段三            |
| ------------- | --------------------- | --------------------- | ----------------- |
| ReduceScatter | Server间ReduceScatter | Server内ReduceScatter | /                 |
| AllGather     | Server内Allgather     | Server间AllGather     | /                 |
| AllReduce     | Server内ReduceScatter | Server间AllReduce     | Server内AllGather |
| Scatter       | Server间Scatter       | Server内Scatter       | /                 |
| Broadcast     | Server内Scatter       | Server间Broadcast     | Server内AllGather |
| Reduce        | Server内ReduceScatter | Server间Reduce        | Server内Gather    |
| AlltoAll      | Server内AlltoAll      | Server间AlltoAll      | /                 |
| AlltoAllV     | Server内AlltoAllV     | Server间AlltoAllV     | /                 |

## ReduceScatter

ReduceScatter算子严格要求第i个rank最终得到第i份规约结果，为了保证Server间通信数据块的连续性，首先在Server间执行ReduceScatter操作，再在Server内执行ReduceScatter操作。

## AllGather

AllGather算子严格要求第i个rank的输入数据出现在结果的第i个位置上，为了保证Server间通信数据块的连续性，首先在Server内执行AllGather操作，然后在Server间执行AllGather操作。

## AllReduce

AllReduce算子的输出是完整的规约结果，因此虽然拆解为了ReduceScatter和AllGather两个阶段，但不需要严格遵循ReduceScatter和AllGather的语义，可以将较大数据量的通信过程放在带宽更高的Server内，即先在Server内执行ReduceScatter操作，然后在Server间执行AllReduce操作，最后在Server内执行AllGather操作。

---

# 集合通信算法

针对同一个通信算子，随着网络拓扑、通信数据量、硬件资源等的不同，往往会采用不同的通信算法，从而最大化集群通信性能。HCCL提供了Mesh、Ring、Recursive Halving-Doubling(RHD)、NHR(Nonuniform Hierarchical Ring)、NB(Nonuniform Bruck)、Pipeline、Pairwise等拓扑算法用于Server内和Server间的集合通信。

## Server内通信算法

HCCL通信域Server内支持Mesh、Ring、Double-Ring和Star算法，具体使用的算法根据硬件拓扑自动选择，用户无须配置也不支持配置。

## Server间通信算法

HCCL通信域Server间支持如下算法的自适应选择，自适应算法会根据产品形态、数据量和Server个数进行选择，用户默认无需配置。

* **Ring算法**：基于环结构的通信算法，通信步数多（线性复杂度），时延相对较高，但通信关系简单，受网络拥塞影响较小。适合通信域内Server个数较少、通信数据量较小、网络存在明显拥塞、且pipeline算法不适用的场景。

* **RHD(Recursive Halving-Doubling)算法**：递归二分和倍增算法，通信步数少（对数复杂度），时延相对较低，但在非2次幂节点规模下会引入额外的通信量。适合通信域内Server个数是2的整数次幂且pipeline算法不适用，或Server个数不是2的整数次幂但通信数据量较小的场景。

* **NHR(Nonuniform Hierarchical Ring)算法**：非均衡的层次环算法，通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且pipeline算法不适用的场景。

* **NB(Nonuniform Bruck)算法**：非均匀的数据块通信算法，通信步数少（对数复杂度），时延相对较低。适合通信域内Server个数较多且pipeline算法不适用的场景。

* **Pipeline算法**：流水线并行算法，可并发使用Server内与Server间的链路，适合通信数据量较大且通信域内每机包含多卡的场景。

* **Pairwise算法**：逐对通信算法，仅用于AllToAll、AlltoAllV与AlltoAllVC算子，通信步数较多（线性复杂度），时延相对较高，但可以避免网络中出现一打多现象（指一个rank通过同一个端口给多个rank发送数据），适合通信数据量较大、需要规避网络一打多的场景。

* cann-hccl仓当前开放的算法有Mesh、Ring、RHD、PairWise、Star，开发者可访问[Gitee-cann-hccl](https://gitee.com/ascend/cann-hccl)详细了解对应实现。

* 开发者若想指定集合通信Server间跨机通信算法，可通过环境变量HCCL_ALGO进行设置。需要注意，若通过环境变量HCCL_ALGO指定了Server间跨机通信算法，Server间的通信算法自适应选择功能不再生效，以用户指定的算法为准。

---

# 通信功能开发

* **[简介](hcclug_000008.html)**

* **[通信域管理](hcclug_000009.html)**

* **[集合通信](hcclug_000010.html)**

* **[点对点通信](hcclug_000011.html)**

* **[集群信息配置](hcclug_000012.html)**

* **[样例代码](hcclug_000016.html)**

---

# 简介

HCCL提供了C与Python两种语言的开发接口，用于实现分布式能力。

* C语言接口用于实现单算子模式下的框架适配，实现分布式能力。

  针对PyTorch框架网络，HCCL单算子API已嵌入到Ascend Extension for PyTorch后端代码中，PyTorch用户直接使用PyTorch原生集合通信API，即可实现分布式能力。

* Python语言接口用于实现图模式下的框架适配，当前仅用于实现TensorFlow网络在昇腾AI处理器执行分布式优化。

**本章节针对如何调用HCCL的C语言接口进行集合通信功能的开发进行介绍。**

开发者调用HCCL C语言接口实现集合通信功能的主要开发流程如下所示。

## 集合通信操作流程

1. 首先进行集群信息配置，创建通信域句柄，并初始化HCCL通信域。

2. 实现通信操作，HCCL通信操作包含两大类：点对点通信与集合通信。
  * 点对点通信，指在多NPU环境下两个NPU之间直接传输数据的过程，常用于pipeline并行场景下对激活值的数据收发。HCCL提供了不同粒度的点对点通信，包括单rank到单rank的单收单发接口，以及多个rank之间的批量收发接口。
  * 集合通信，指多个NPU共同参与进行数据传输操作，例如AllReduce，AllGather，Broadcast等，常用于大规模集群中不同NPU之间的梯度同步和参数更新等操作。集合通信操作可让所有计算节点并行、高效、有序执行数据交换，提升数据传输效率。

3. 集合通信操作完成后，需要释放相关资源，销毁通信域。

---

# 通信域管理

## 概述

通信域是集合通信算子执行的上下文，管理对应的通信对象（例如一个NPU就是一个通信对象）和通信所需的资源。通信域中的每个通信对象称为一个rank，每个rank都会分配一个介于0~n-1（n为npu的数量）的唯一标识。

## 通信域创建方式

通信域创建根据用户场景的不同主要有以下几种方式：

- 多机集合通信场景，如果有完整的描述集群信息的ranktable文件，可通过HcclCommInitClusterInfo接口创建通信域，或者通过HcclCommInitClusterInfoConfig接口创建具有特定配置的通信域。
- 多机集合通信场景，如果无完整的ranktable，可通过HcclGetRootInfo接口与HcclCommInitRootInfo/HcclCommInitRootInfoConfig接口配合使用，基于root节点广播方式创建通信域。
- 单机集合通信场景，可通过HcclCommInitAll接口在单机内批量创建通信域。
- 基于已有的通信域，可通过HcclCreateSubCommConfig接口切分具有特定配置的子通信域。

## 重要注意事项

- 针对同一个rank，通信算子需要one by one保序调用，不支持并发下发。
- 针对整个通信域，通信算子的调用在不同rank间要保证同一下发顺序。
- 同一个通信域内不支持图模式通信和单算子通信混合执行。

## 基于ranktable创建通信域

多机集合通信、基于集群信息配置ranktable文件创建通信域的场景，每张卡需要使用一个单独的进程参考如下流程创建通信域：

1. 构造ranktable文件（ranktable文件的配置可参见ranktable文件配置资源信息），如果已存在ranktable文件，也可以通过环境变量RANK_TABLE_FILE获取。
2. 每张卡使用HcclCommInitClusterInfo接口创建通信域，或者使用HcclCommInitClusterInfoConfig接口创建具有特定配置的通信域。

### 代码示例

```c
// 获取ranktable路径
char* rankTableFile = getenv("RANK_TABLE_FILE");
// 定义通信域句柄
HcclComm hcclComm;
// 初始化HCCL通信域
HcclCommInitClusterInfo(rankTableFile, devId, &hcclComm);

/*  集合通信操作   */

// 销毁HCCL通信域
HcclCommDestroy(hcclComm);
```

## 基于root节点广播方式创建通信域

多机集合通信场景，若无完整的集群信息配置ranktable文件，HCCL提供了基于root节点广播的方式创建通信域，详细流程如下：

1. 配置环境变量HCCL_IF_IP，指定root通信网卡的IP地址。

   需要配置为root节点Host网卡的IP地址，可以是IPv4或IPv6格式，仅支持配置一个IP地址，配置示例如下：

   ```
   export HCCL_IF_IP=10.10.10.1
   ```

2. 在root节点调用HcclGetRootInfo接口，生成root节点rank标识信息"rootInfo"，包括device ip、device id等信息。
3. 将root节点的rank信息广播至集群中的所有rank。
4. 在所有节点调用HcclCommInitRootInfo或者HcclCommInitRootInfoConfig接口（创建具有特定配置的通信域），基于接收到的"rootInfo"，以及本rank的rank id等信息，进行HCCL初始化。

需要注意：每个卡需要使用一个单独的进程进行以上操作。

### 代码示例

```c
// 获取当前进程在所属进程组的编号
MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
int devId = procRank;
// 在root节点获取其rank信息
HcclRootInfo rootInfo;
int32_t rootRank = 0;
if(devId == rootRank) {
    HcclGetRootInfo(&rootInfo);
}
// 将root_info广播到通信域内的其他rank
MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
MPI_Barrier(MPI_COMM_WORLD);
// 定义通信域句柄
HcclComm hcclComm;
// 初始化HCCL通信域
HcclCommInitRootInfo(devCount, &rootInfo, devId, &hcclComm);

/*  集合通信操作   */

// 销毁HCCL通信域
HcclCommDestroy(hcclComm);
```

## 单机内批量创建通信域

单机通信场景中，开发者通过一个进程统一创建多张卡的通信域，其中一张卡对应一个线程，创建流程如下：

1. 构造通信域中的Device列表，例如：{0, 1, 2, 3, 4, 5, 6, 7}，其中列表中的Device ID是逻辑ID。
2. 在进程中调用HcclCommInitAll接口创建通信域。

### 代码示例

```c
uint32_t ndev = 8;
// 构造Device的逻辑ID列表
int32_t devices[8] = {0, 1, 2, 3, 4, 5, 6, 7};
// 定义通信域句柄
HcclComm comms[ndev];
// 初始化HCCL通信域
HcclCommInitAll(ndev, devices, comms);

// 启动线程执行集合通信操作
std::vector<std::unique_ptr<std::thread> > threads(ndev);
struct ThreadContext args[ndev];
for (uint32_t i = 0; i < ndev; i++) {
    args[i].device = i;
    args[i].comm = comms[i];
   /*  集合通信操作   */
}

// 销毁HCCL通信域
for (uint32_t i = 0; i < ndev; i++) {
    HcclCommDestroy(comms[i]);
}
```

需要注意，多线程调用集合通信操作API时（例如HcclAllReduce时），需要确保不同线程中调用集合通信操作API的前后时间差不超过环境变量HCCL_CONNECT_TIMEOUT的时间，避免建链超时。

## 基于已有通信域切分子通信域

HCCL提供了HcclCreateSubCommConfig接口，实现基于已有通信域切分具有特性配置子通信域的功能。该子通信域创建方式无需进行socket建链与rank信息交换，可应用于业务故障下的快速通信域创建。

**注意：** 该接口仅支持从全局通信域切分子通信域，不支持通信域的嵌套切分。

## 销毁通信域

集合通信操作完成后，需要调用运行时管理接口释放通信所用的内存、Stream、Device资源，并调用HcclCommDestroy接口销毁指定的通信域。

---

# 集合通信

## 概述

集合通信是指多个NPU共同参与进行数据传输，从而形成一次集体操作的通信模式，常用于大规模集群中不同NPU之间的梯度同步和参数更新等场景。

## 支持的通信原语

HCCL支持以下通信原语：
- AllReduce
- Broadcast
- AllGather
- Scatter
- ReduceScatter
- Reduce
- AlltoAll
- AlltoAllV

关于这些通信原语的详细含义可参见集合通信原语相关文档。

## HcclAllReduce接口

HCCL针对AllReduce操作提供了HcclAllReduce接口，原型定义如下：

```c
HcclResult HcclAllReduce(void *sendBuf, void *recvBuf, uint64_t count,
                         HcclDataType dataType, HcclReduceOp op,
                         HcclComm comm, aclrtStream stream)
```

### 功能说明

HcclAllReduce用于将通信域内所有节点的输入进行Reduce操作，再把结果发送到所有节点的输出。op参数用于指定Reduce的操作类型，当前版本支持的操作类型有：sum、prod、max、min。

## 代码示例

```c
void* host_buf = nullptr;
void* send_buff = nullptr;
void* recv_buff = nullptr;
uint64_t count = 1;
int malloc_kSize = count * sizeof(float);
aclrtStream stream;
aclrtCreateStream(&stream);

//申请集合通信操作的内存
aclrtMalloc((void**)&sendbuff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST);
aclrtMalloc((void**)&recvbuff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST);

//初始化输入内存
aclrtMallocHost((void**)&host_buf, malloc_kSize);
aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf,
            malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE);

//执行集合通信操作
HcclAllReduce((void *)send_buff, (void*)recv_buff, count,
              HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, hcclComm, stream);
```

---

# 点对点通信

点对点通信是指多NPU环境下两个NPU之间直接传输数据的通信模式，常用于pipeline并行场景下对激活值的数据收发。

HCCL提供了不同粒度的点对点通信，包括单rank到单rank的单收单发接口，以及多个rank之间的批量收发接口。

## HcclSend/HcclRecv

HcclSend/HcclRecv用于单收单发场景，需严格保序下发并配对使用，收发两端需完成同步后才能进行数据收发，数据收发完成后才能执行后续算子任务。

### 代码示例

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

## HcclBatchSendRecv

HcclBatchSendRecv可用于通信域内多个rank之间的数据收发，该接口有两个特征：

- 接口内部会对批量数据收发顺序进行重排，所以不严格要求单次接口调用中批量收发的任务顺序，但需要确保一次接口调用中的数据发送与数据接收操作个数完全匹配。
- 收发过程独立调度执行，收发不相互阻塞，从而实现双工链路并发。

### 使用注意

单次接口调用下，两个rank之间单向数据流仅支持传递一块内存数据，避免收发过程中混淆多块内存数据的收发地址。

### 代码示例

```c
HcclSendRecvItem sendRecvInfo[itemNum];
HcclSendRecvType currType;
for (size_t i = 0; i < op_type.size(); ++i) {
    if (op_type[i] == "isend") {
        currType = HcclSendRecvType::HCCL_SEND;
    } else if (op_type[i] == "irecv") {
        currType = HcclSendRecvType::HCCL_RECV;
    }
    sendRecvInfo[i] = HcclSendRecvItem{currType,
                                       tensor_ptr_list[i],
                                       count_list[i],
                                       type_list[i],
                                       remote_rank_list[i]
                                       };
}
HcclBatchSendRecv(sendRecvInfo, itemNum, hcclComm, stream);
```

---

# 集群信息配置

## 简介

[简介](hcclug_000013.html)

## 文档导航

- **[ranktable文件配置资源信息](hcclug_000014.html)**

- **[环境变量配置资源信息](hcclug_000015.html)**

  开发者可以通过"环境变量组合的方式配置资源信息"，这是除了ranktable文件方法外的另一种选择。

## 父主题

[通信功能开发](hcclug_000007.html)

---

# 简介

HCCL提供了两种集群信息配置方式用于通信域初始化：ranktable配置文件的方式和环境变量的方式，开发者可根据使用场景选择其中一种方式，但两种方式不可以混用。

## ranktable配置文件方式使用场景

* 通过C语言接口HcclCommInitClusterInfo或者HcclCommInitClusterInfoConfig初始化通信域时
* TensorFlow分布式网络通信域初始化时

## 环境变量方式适用范围

通过环境变量的方式配置资源信息仅适用于TensorFlow框架网络的通信域初始化，仅支持如下型号的产品：

* Atlas训练系列产品
* Atlas A2 训练系列产品

**父主题：** 集群信息配置

---

# ranktable文件配置资源信息

开发者可以通过ranktable文件配置参与集合通信的NPU资源信息，ranktable文件为json格式，开发者可以在此文件中配置全量NPU资源信息，后续进程启动时可使用其中指定的几个NPU资源。

## 配置文件说明（**Atlas A2 训练系列产品**）

针对Atlas A2 训练系列产品，ranktable文件配置说明如下：

```json
{
"status":"completed",
"version":"1.0",
"server_count":"1",
"server_list":
[
   {
        "device":[
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
                  ],
         "server_id":"node_0"
    }
]
}
```

**表1** ranktable文件说明

| 配置项       | 配置说明                                                                                                                                                                                                                                                                                                                          | 可选/必选 |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| status       | ranktable可用标识。completed：表示ranktable可用，可执行训练。initializing：表示ranktable不可用，不可执行训练。                                                                                                                                                                                                                    | 必选      |
| version      | ranktable模板版本信息。配置为1.0。                                                                                                                                                                                                                                                                                                | 必选      |
| server_count | 本次参与训练的AI Server个数。                                                                                                                                                                                                                                                                                                     | 必选      |
| server_list  | 本次参与训练的AI Server列表。                                                                                                                                                                                                                                                                                                     | 必选      |
| server_id    | AI Server标识，字符串类型，长度小于64，请确保全局唯一。配置示例：node_0                                                                                                                                                                                                                                                           | 必选      |
| device_id    | 昇腾AI处理器的物理ID，即Device在AI Server上的序列号。可通过执行"**ls /dev/davinci***"命令获取昇腾AI处理器的物理ID。例如：显示/dev/davinci0，表示昇腾AI处理器的物理ID为0。取值范围：[0，实际Device数量-1]。须知："device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。                                                         | 必选      |
| device_ip    | 昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。需要注意：多机场景下，device_ip必须配置。单机场景下，device_ip可不配置。可以在当前AI Server执行指令cat /etc/hccn.conf获取网卡IP。查询到的address_xx即为网卡IP，address后的序号为昇腾AI处理器物理ID，即device_id，后面的ip地址即为需要用户填入的该device对应的网卡IP。 | 可选      |
| rank_id      | Rank唯一标识，请配置为整数，从0开始配置，且全局唯一，取值范围：[0, 总Device数量-1]。为方便管理，建议rank_id按照Device物理连接顺序进行排序，即将物理连接上较近的device编排在一起。例如，若device_ip按照物理连接从小到大设置，则rank_id也建议按照从小到大的顺序设置。                                                               | 必选      |

## 配置文件说明（Atlas 300I Duo推理卡）

针对Atlas 300I Duo推理卡，ranktable文件配置说明如下：

```json
{
"status":"completed",
"version":"1.0",
"server_count":"1",
"server_list":
[
   {
        "device":[
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
                  ],
         "server_id":"node_0"
    }
]
}
```

**表2** ranktable文件说明

| 配置项       | 配置说明                                                                                                                                                                                                                                                                  | 可选/必选 |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| status       | ranktable可用标识。completed：表示ranktable可用，可执行训练。initializing：表示ranktable不可用，不可执行训练。                                                                                                                                                            | 必选      |
| version      | ranktable模板版本信息，当前仅支持配置为1.0。                                                                                                                                                                                                                              | 必选      |
| server_count | 本次参与训练的AI Server个数。                                                                                                                                                                                                                                             | 必选      |
| server_list  | 本次参与训练的AI Server列表。                                                                                                                                                                                                                                             | 必选      |
| server_id    | AI Server标识，字符串类型，长度小于64，请确保全局唯一。配置示例：node_0                                                                                                                                                                                                   | 必选      |
| device_id    | 昇腾AI处理器的物理ID，即Device在AI Server上的序列号。可通过执行"**ls /dev/davinci***"命令获取昇腾AI处理器的物理ID。例如：显示/dev/davinci0，表示昇腾AI处理器的物理ID为0。取值范围：[0，实际Device数量-1]。须知："device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 | 必选      |
| device_ip    | 昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。可以在当前AI Server执行指令"cat /etc/hccn.conf"获取网卡IP。查询到的address_xx即为网卡IP，address后的序号为昇腾AI处理器的物理ID，即device_id，后面的ip地址即为需要用户填入的该device对应的网卡IP。             | 必选      |
| rank_id      | Rank唯一标识，请配置为整数，从0开始配置，且全局唯一，取值范围：[0, 总Device数量-1]。为方便管理，建议rank_id按照Device物理连接顺序进行排序，即将物理连接上较近的device编排在一起。例如，若device_ip按照物理连接从小到大设置，则rank_id也建议按照从小到大的顺序设置。       | 必选      |

## 配置文件说明（**Atlas训练系列产品**）

针对Atlas训练系列产品，在ranktable文件中配置参与训练的昇腾AI处理器信息支持两种配置模板，全新场景推荐使用模板一，模板二用于兼容部分已有场景。

### 模板一（推荐使用）

```json
{
"status":"completed",
"version":"1.0",
"server_count":"1",
"server_list":
[
   {
        "device":[
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
                  ],
         "server_id":"node_0"
    }
]
}
```

**表3** ranktable文件说明

| 配置项       | 配置说明                                                                                                                                                                                                                                                                  | 可选/必选 |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| status       | ranktable可用标识。completed：表示ranktable可用，可执行训练。initializing：表示ranktable不可用，不可执行训练。                                                                                                                                                            | 必选      |
| version      | ranktable模板版本信息，当前仅支持配置为1.0。                                                                                                                                                                                                                              | 必选      |
| server_count | 本次参与训练的AI Server个数。                                                                                                                                                                                                                                             | 必选      |
| server_list  | 本次参与训练的AI Server列表。                                                                                                                                                                                                                                             | 必选      |
| server_id    | AI Server标识，字符串类型，长度小于64，请确保全局唯一。配置示例：node_0                                                                                                                                                                                                   | 必选      |
| device_id    | 昇腾AI处理器的物理ID，即Device在AI Server上的序列号。可通过执行"**ls /dev/davinci***"命令获取昇腾AI处理器的物理ID。例如：显示/dev/davinci0，表示昇腾AI处理器的物理ID为0。取值范围：[0，实际Device数量-1]。须知："device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 | 必选      |
| device_ip    | 昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。可以在当前AI Server执行指令"cat /etc/hccn.conf"获取网卡IP。查询到的address_xx即为网卡IP，address后的序号为昇腾AI处理器的物理ID，即device_id，后面的ip地址即为需要用户填入的该device对应的网卡IP。             | 必选      |
| rank_id      | Rank唯一标识，请配置为整数，从0开始配置，且全局唯一，取值范围：[0, 总Device数量-1]。为方便管理，建议rank_id按照Device物理连接顺序进行排序，即将物理连接上较近的device编排在一起。例如，若device_ip按照物理连接从小到大设置，则rank_id也建议按照从小到大的顺序设置。       | 必选      |

### 模板二（兼容部分已有场景，新版本不推荐使用）

```json
{
"status":"completed",
"group_count":"1",
"group_list":
[
   {
    "group_name":"hccl_world_group",
    "instance_count":"2",
    "device_count":"2",
    "instance_list":[
        {
           "pod_name":"tf-bae41",
           "server_id":"node_0",
           "devices":[
            {
              "device_id":"0",
              "device_ip":"192.168.1.8"
            }
           ]
        },
        {
            "pod_name":"tf-tbdf1",
            "server_id":"node_1",
            "devices":[
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

**表4** ranktable文件说明

| 配置项         | 配置说明                                                                                                                                                                                                                                                               | 可选/必选 |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| status         | ranktable可用标识。completed：表示ranktable可用，可执行训练。initializing：表示ranktable不可用，不可执行训练。                                                                                                                                                         | 必选      |
| group_count    | 用户申请的group数量，建议配置为1。                                                                                                                                                                                                                                     | 必选      |
| group_list     | Group列表。                                                                                                                                                                                                                                                            | 必选      |
| group_name     | Group名称，当group_count为1时，建议配置为hccl_world_group或者空。因为当前版本无论定义为任何值，都会创建名称为hccl_world_group的group。如果通过该配置文件创建了多个group，则系统会自动将多个group合并为一个名称为"hccl_world_group"的group资源。                        | 可选      |
| instance_count | 和instance_list中pod_name个数保持一致，例如：容器场景下为容器实际数量。                                                                                                                                                                                                | 必选      |
| device_count   | group中设备数量。                                                                                                                                                                                                                                                      | 必选      |
| instance_list  | instance实例信息列表。                                                                                                                                                                                                                                                 | 必选      |
| pod_name       | 用户自定义配置，保持instance_list内全局唯一。                                                                                                                                                                                                                          | 必选      |
| server_id      | AI Server标识，字符串类型，长度小于64，请确保全局唯一。配置示例：node_0                                                                                                                                                                                                | 必选      |
| devices        | devices信息列表。                                                                                                                                                                                                                                                      | 必选      |
| device_id      | 昇腾AI处理器的物理ID，即Device在Server上的序列号。可通过执行"**ls /dev/davinci***"命令获取昇腾AI处理器的物理ID。例如：显示/dev/davinci0，表示昇腾AI处理器的物理ID为0。取值范围：[0，实际Device数量-1]。须知："device_id"配置项的优先级高于环境变量"ASCEND_DEVICE_ID"。 | 必选      |
| device_ip      | 昇腾AI处理器集成网卡IP，全局唯一，要求为常规IPv4或IPv6格式。可以在当前Server执行指令cat /etc/hccn.conf获取网卡IP。                                                                                                                                                     | 必选      |

---

# 环境变量配置资源信息

除了通过ranktable文件配置资源信息的方式外，开发者还可以通过环境变量组合的方式配置资源信息。

该资源信息配置方式仅适用于TensorFlow框架网络的通信域初始化，仅支持如下产品型号：

- Atlas训练系列产品
- Atlas A2 训练系列产品

## 配置说明

需要在执行训练的每个AI Server节点上分别配置如下环境变量：

```
export CM_CHIEF_IP = 192.168.1.1
export CM_CHIEF_PORT = 6000
export CM_CHIEF_DEVICE = 0
export CM_WORKER_SIZE = 8
export CM_WORKER_IP = 192.168.0.1
export HCCL_SOCKET_FAMILY=AF_INET
```

### 环境变量说明

- **CM_CHIEF_IP、CM_CHIEF_PORT、CM_CHIEF_DEVICE**：用于配置Master节点的Host监听IP、监听端口与主Device ID。Master节点为集群管理主节点，负责集群内设备信息的管理、资源的分配和调度等。
  - 监听IP可以通过ifconfig命令进行查询，要求为常规IPv4或IPv6格式
  - 指定的监听端口号需要确保在训练进程拉起时，无其他业务占用

- **CM_WORKER_SIZE**：用于配置组网中参与集群训练的Device总数量

- **CM_WORKER_IP**：用于配置当前节点与Master进行通信时所用的网卡IP，可通过ifconfig命令查询，要求为常规IPv4或IPv6格式。需要确保指定的网卡IP能够与Master节点正常通信。

- **HCCL_SOCKET_FAMILY**：此环境变量可选，用于控制Device侧通信网卡使用的IP协议版本。AF_INET代表使用IPv4协议，AF_INET6代表使用IPv6协议，缺省时，优先使用IPv4协议。

  如果环境变量"HCCL_SOCKET_FAMILY"指定的IP协议与实际获取到的网卡信息不匹配，则以实际环境上的网卡信息为准。例如，环境变量"HCCL_SOCKET_FAMILY"指定为"AF_INET6"，但Device侧只存在IPv4协议的网卡，则实际会使用IPv4协议的网卡。

## 配置示例

假设执行分布式训练的Server节点数量为2，Device数量为16为例，每个Server节点有8个Device。拉起每个Device上的训练进程前，在对应的shell窗口中配置如下环境变量：

### 节点0（Master节点）

```
export CM_CHIEF_IP = 192.168.1.1
export CM_CHIEF_PORT = 6000
export CM_CHIEF_DEVICE = 0
export CM_WORKER_SIZE = 16
export CM_WORKER_IP = 192.168.1.1
```

### 节点1

```
export CM_CHIEF_IP = 192.168.1.1
export CM_CHIEF_PORT = 6000
export CM_CHIEF_DEVICE = 0
export CM_WORKER_SIZE = 16
export CM_WORKER_IP = 192.168.2.1
```

---

# 样例代码

*   **[安装配置MPI](hcclug_000017.html)**

*   **[HcclCommInitClusterInfo初始化方式](hcclug_000019.html)**

*   **[HcclCommInitClusterInfoConfig初始化方式](hcclug_000020.html)**

*   **[HcclCommInitRootInfo初始化方式](hcclug_000021.html)**

*   **[HcclCommInitRootInfoConfig初始化方式](hcclug_000022.html)**

*   **[HcclCommInitAll初始化](hcclug_000023.html)**

*   **[HcclCreateSubCommConfig方式创建子通信域](hcclug_000024.html)**

*   **[样例编译运行](hcclug_000032.html)**

**父主题：** [通信功能开发](hcclug_000007.html)

---

# 安装配置MPI

HCCL的通信域初始化依赖MPI拉起多个进程，所以进行HCCL的代码样例编写前，需要先安装配置MPI软件包。

MPI软件包的安装配置可参见《[HCCL性能测试工具使用指南](https://www.hiascend.com/document/detail/zh/canncommercial/80RC3/devaids/devtools/hccltool/HCCLpertest_16_0001.html)》中的[MPI安装与配置](/document/detail/zh/canncommercial/80RC3/devaids/devtools/hccltool/HCCLpertest_16_0002.html)。

**父主题：** [样例代码](hcclug_000016.html)

---

# HcclCommInitClusterInfo初始化方式

## 准备ranktable文件

该样例通过获取ranktable的方式进行初始化，所以需准备一份ranktable文件配置集群信息，供后续调用接口时使用。

配置"RANK_TABLE_FILE"环境变量，指定ranktable文件所在路径，如下所示，文件名称为"ranktable.json"。

```
export RANK_TABLE_FILE=/home/test/ranktable.json
```

以**Atlas A2 训练系列产品**，组网为单机8卡为例，ranktable.json配置示例如下，不同产品形态ranktable文件的配置示例及详细参数说明可参见[ranktable文件配置资源信息](hcclug_000014.html)。

```json
{
        "status":"completed",
        "version": "1.0",
        "server_count": "1",
        "server_list": [{
                "server_id": "SERVER_ID_SV1",
                "device": [{
                        "device_id": "0",
                        "device_ip": "192.168.1.8",
                        "rank_id": "0"
                },
                {
                        "device_id": "1",
                        "device_ip": "192.168.1.9",
                        "rank_id": "1"
                },
                {
                        "device_id": "2",
                        "device_ip": "192.168.1.10",
                        "rank_id": "2"
                },
                {
                        "device_id": "3",
                        "device_ip": "192.168.1.11",
                        "rank_id": "3"
                },
                {
                        "device_id": "4",
                        "device_ip": "192.168.1.12",
                        "rank_id": "4"
                },
                {
                        "device_id": "5",
                        "device_ip": "192.168.1.13",
                        "rank_id": "5"
                },
                {
                        "device_id": "6",
                        "device_ip": "192.168.1.14",
                        "rank_id": "6"
                },
                {
                        "device_id": "7",
                        "device_ip": "192.168.1.15",
                        "rank_id": "7"
                }]
        }]
}
```

## HcclSend/HcclRecv操作代码样例

该样例仅支持单机8卡的组网。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 8;
    int mallocSize = count * sizeof(float);
    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    // 执行SendRecv操作
    if (ctx->device / 4 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 4, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 4, ctx->comm, stream));
    }
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device / 4 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }
    // 释放通信用sendBuf、recvBuf内存，stream等资源
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    return 0;
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));
    HcclComm hcclComm;
    HcclCommInitClusterInfo(rankTableFile, devId, &hcclComm);
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    HCCLCHECK(HcclCommDestroy(hcclComm));
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例支持单机N卡的组网，N需要小于等于8。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    void* host_buf = nullptr;
    void* send_buff = nullptr;
    void* recv_buff = nullptr;
    uint64_t count = 1;
    int malloc_kSize = count * sizeof(float);
    aclrtEvent start_event, end_event;
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    ACLCHECK(aclrtCreateEvent(&start_event));
    ACLCHECK(aclrtCreateEvent(&end_event));

    //申请集合通信操作的内存
    ACLCHECK(aclrtMalloc((void**)&send_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMalloc((void**)&recv_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));

    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&host_buf, malloc_kSize));
    ACLCHECK(aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE));

    //执行集合通信操作
    HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->comm, stream));

    //等待stream中集合通信任务执行完成
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, malloc_kSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, malloc_kSize, (void*)recv_buff, malloc_kSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFree(send_buff));
    ACLCHECK(aclrtFree(recv_buff));
    ACLCHECK(aclrtFreeHost(host_buf));
    //销毁任务流
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtDestroyEvent(start_event));
    ACLCHECK(aclrtDestroyEvent(end_event));
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));
    HcclComm hcclComm;
    HcclCommInitClusterInfo(rankTableFile, devId, &hcclComm);
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    HCCLCHECK(HcclCommDestroy(hcclComm));
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

**父主题：** [样例代码](hcclug_000016.html)

---

# HcclCommInitClusterInfoConfig初始化方式

## 准备ranktable文件

该样例通过获取ranktable的方式进行初始化，所以需准备一份ranktable文件配置集群信息，供后续调用接口时使用。

配置"RANK_TABLE_FILE"环境变量，指定ranktable文件所在路径，如下所示，文件名称为"ranktable.json"。

```
export RANK_TABLE_FILE=/home/test/ranktable.json
```

以**Atlas A2 训练系列产品**，组网为单机8卡为例，ranktable.json配置示例如下，不同产品形态ranktable文件的配置示例及详细参数说明可参见[ranktable文件配置资源信息](hcclug_000014.html)。

```json
{
        "status":"completed",
        "version": "1.0",
        "server_count": "1",
        "server_list": [{
                "server_id": "SERVER_ID_SV1",
                "device": [{
                        "device_id": "0",
                        "device_ip": "192.168.1.8",
                        "rank_id": "0"
                },
                {
                        "device_id": "1",
                        "device_ip": "192.168.1.9",
                        "rank_id": "1"
                },
                {
                        "device_id": "2",
                        "device_ip": "192.168.1.10",
                        "rank_id": "2"
                },
                {
                        "device_id": "3",
                        "device_ip": "192.168.1.11",
                        "rank_id": "3"
                },
                {
                        "device_id": "4",
                        "device_ip": "192.168.1.12",
                        "rank_id": "4"
                },
                {
                        "device_id": "5",
                        "device_ip": "192.168.1.13",
                        "rank_id": "5"
                },
                {
                        "device_id": "6",
                        "device_ip": "192.168.1.14",
                        "rank_id": "6"
                },
                {
                        "device_id": "7",
                        "device_ip": "192.168.1.15",
                        "rank_id": "7"
                }]
        }]
}
```

## HcclSend/HcclRecv操作代码样例

该样例仅支持单机8卡的组网。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include <cstring>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 8;
    int mallocSize = count * sizeof(float);
     //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    // 执行SendRecv操作
    if (ctx->device / 4 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 4, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 4, ctx->comm, stream));
    }
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device / 4 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }
    // 释放通信用sendBuf、recvBuf内存，stream等资源
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    return 0;
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));

    // 创建并初始化通信域配置项
    HcclCommConfig config;
    HcclCommConfigInit(&config);
    // 根据需要修改通信域配置
    config.hcclBufferSize = 50;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm hcclComm;
    HcclCommInitClusterInfoConfig(rankTableFile, devId, &config, &hcclComm);
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    HCCLCHECK(HcclCommDestroy(hcclComm));
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例支持单机N卡的组网，N需要小于等于8。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include <cstring>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    void* host_buf = nullptr;
    void* send_buff = nullptr;
    void* recv_buff = nullptr;
    uint64_t count = 1;
    int malloc_kSize = count * sizeof(float);
    aclrtEvent start_event, end_event;
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    ACLCHECK(aclrtCreateEvent(&start_event));
    ACLCHECK(aclrtCreateEvent(&end_event));

    //申请集合通信操作的内存
    ACLCHECK(aclrtMalloc((void**)&send_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMalloc((void**)&recv_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));

    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&host_buf, malloc_kSize));
    ACLCHECK(aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE));

    //执行集合通信操作
    HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->comm, stream));
    //等待stream中集合通信任务执行完成
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, malloc_kSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, malloc_kSize, (void*)recv_buff, malloc_kSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFree(send_buff));
    ACLCHECK(aclrtFree(recv_buff));
    ACLCHECK(aclrtFreeHost(host_buf));
    //销毁任务流
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtDestroyEvent(start_event));
    ACLCHECK(aclrtDestroyEvent(end_event));
    return 0;
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));

    // 创建并初始化通信域配置项
    HcclCommConfig config;
    HcclCommConfigInit(&config);
    // 根据需要修改通信域配置
    config.hcclBufferSize = 50;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm hcclComm;
    HcclCommInitClusterInfoConfig(rankTableFile, devId, &config, &hcclComm);
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    HCCLCHECK(HcclCommDestroy(hcclComm));
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

**父主题：** [样例代码](hcclug_000016.html)

---

# HcclCommInitRootInfo初始化方式

## HcclSend/HcclRecv操作代码样例

该样例仅支持单机8卡的组网。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do {\
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)\

#define HCCLCHECK(ret) do {\
    if(ret != HCCL_SUCCESS)\
    {\
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 8;
    int mallocSize = count * sizeof(float);
     //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    //执行SendRecv操作
    if (ctx->device / 4 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 4, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 4, ctx->comm, stream));
    }

    ACLCHECK(aclrtSynchronizeStream(stream));
    if (ctx->device / 4 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }
    // 释放通信用sendBuf、recvBuf内存，stream等资源
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    HCCLCHECK(HcclCommDestroy(ctx->comm));
    return 0;
}

int main() {
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    //设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));
    // 在 rootRank 获取 rootInfo
    HcclRootInfo rootInfo;
    int32_t rootRank = 0;
    if(devId == rootRank) {
        HCCLCHECK(HcclGetRootInfo(&rootInfo));
    }
    // 将root_info广播到通信域内的其他rank
    MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);
    // 初始化集合通信域
    HcclComm hcclComm;
    HCCLCHECK(HcclCommInitRootInfo(devCount, &rootInfo, devId, &hcclComm));
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例支持单机N卡的组网，N需要小于等于8。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do {\
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)\

#define HCCLCHECK(ret) do {\
    if(ret != HCCL_SUCCESS)\
    {\
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    void* host_buf = nullptr;
    void* send_buff = nullptr;
    void* recv_buff = nullptr;
    uint64_t count = 1;
    int malloc_kSize = count * sizeof(float);
    aclrtEvent start_event, end_event;
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    ACLCHECK(aclrtCreateEvent(&start_event));
    ACLCHECK(aclrtCreateEvent(&end_event));

    //申请集合通信操作的内存
    ACLCHECK(aclrtMalloc((void**)&send_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMalloc((void**)&recv_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));

    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&host_buf, malloc_kSize));
    ACLCHECK(aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE));

    //执行集合通信操作
    HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->comm, stream));

    //等待stream中集合通信任务执行完成
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, malloc_kSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, malloc_kSize, (void*)recv_buff, malloc_kSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFree(send_buff));
    ACLCHECK(aclrtFree(recv_buff));
    ACLCHECK(aclrtFreeHost(host_buf));
    //销毁任务流
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtDestroyEvent(start_event));
    ACLCHECK(aclrtDestroyEvent(end_event));
    return 0;
}

int main(int argc, char*argv[])
{
    MPI_Init(&argc, &argv);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    //设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));
    // 在 rootRank 获取 rootInfo
    HcclRootInfo rootInfo;
    int32_t rootRank = 0;
    if(devId == rootRank) {
        HCCLCHECK(HcclGetRootInfo(&rootInfo));
    }
    // 将root_info广播到通信域内的其他rank
    MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);
    // 初始化集合通信域
    HcclComm hcclComm;
    HCCLCHECK(HcclCommInitRootInfo(devCount, &rootInfo, devId, &hcclComm));
    // 创建任务stream
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    //销毁集合通信域
    HCCLCHECK(HcclCommDestroy(hcclComm));
    //重置设备
    ACLCHECK(aclrtResetDevice(devId));
    //设备去初始化
    ACLCHECK(aclFinalize());
    return 0;
}
```

---

# HcclCommInitRootInfoConfig初始化方式

## HcclSend/HcclRecv操作代码样例

该样例仅支持单机8卡的组网。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do {\
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)\

#define HCCLCHECK(ret) do {\
    if(ret != HCCL_SUCCESS)\
    {\
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 8;
    int mallocSize = count * sizeof(float);

    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));

    if (ctx->device / 4 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 4, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 4, ctx->comm, stream));
    }

    ACLCHECK(aclrtSynchronizeStream(stream));
    if (ctx->device / 4 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    HCCLCHECK(HcclCommDestroy(ctx->comm));
    return 0;
}

int main() {
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;

    ACLCHECK(aclInit(NULL));
    ACLCHECK(aclrtSetDevice(devId));

    HcclRootInfo rootInfo;
    int32_t rootRank = 0;
    if(devId == rootRank) {
        HCCLCHECK(HcclGetRootInfo(&rootInfo));
    }

    MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);

    HcclCommConfig config;
    HcclCommConfigInit(&config);

    config.hcclBufferSize = 50;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm hcclComm;
    HCCLCHECK(HcclCommInitRootInfoConfig(devCount, &rootInfo, devId, &config, &hcclComm));
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);

    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例支持单机N卡的组网，N需要小于等于8。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do {\
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)\

#define HCCLCHECK(ret) do {\
    if(ret != HCCL_SUCCESS)\
    {\
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    void* host_buf = nullptr;
    void* send_buff = nullptr;
    void* recv_buff = nullptr;
    uint64_t count = 1;
    int malloc_kSize = count * sizeof(float);
    aclrtEvent start_event, end_event;
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    ACLCHECK(aclrtCreateEvent(&start_event));
    ACLCHECK(aclrtCreateEvent(&end_event));

    ACLCHECK(aclrtMalloc((void**)&send_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMalloc((void**)&recv_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));

    ACLCHECK(aclrtMallocHost((void**)&host_buf, malloc_kSize));
    ACLCHECK(aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE));

    HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->comm, stream));

    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, malloc_kSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, malloc_kSize, (void*)recv_buff, malloc_kSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFree(send_buff));
    ACLCHECK(aclrtFree(recv_buff));
    ACLCHECK(aclrtFreeHost(host_buf));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtDestroyEvent(start_event));
    ACLCHECK(aclrtDestroyEvent(end_event));
    return 0;
}

int main(int argc, char*argv[])
{
    MPI_Init(&argc, &argv);
    int procSize = 0;
    int procRank = 0;
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;

    ACLCHECK(aclInit(NULL));
    ACLCHECK(aclrtSetDevice(devId));

    HcclRootInfo rootInfo;
    int32_t rootRank = 0;
    if(devId == rootRank) {
        HCCLCHECK(HcclGetRootInfo(&rootInfo));
    }

    MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);

    HcclCommConfig config;
    HcclCommConfigInit(&config);

    config.hcclBufferSize = 1024;
    config.hcclDeterministic = 1;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm hcclComm;
    HCCLCHECK(HcclCommInitRootInfoConfig(devCount, &rootInfo, devId, &config, &hcclComm));

    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);

    HCCLCHECK(HcclCommDestroy(hcclComm));
    ACLCHECK(aclrtResetDevice(devId));
    ACLCHECK(aclFinalize());
    return 0;
}
```

---

# HcclCommInitAll初始化

## HcclSend/HcclRecv操作代码样例

该样例仅支持单机8卡的组网，且仅支持单进程方式拉起。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 8;
    int mallocSize = count * sizeof(float);
    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    // 执行SendRecv操作
    if (ctx->device / 4 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 4, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 4, ctx->comm, stream));
    }

    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device / 4 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    // 释放通信用sendBuf、recvBuf内存，stream等资源
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    HCCLCHECK(HcclCommDestroy(ctx->comm));
    return 0;
}

int main() {
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    uint32_t ndev = 8;
    int32_t devices[8] = {0, 1, 2, 3, 4, 5, 6, 7};
    HcclComm comms[ndev];
    for (int32_t i = 0; i < ndev; i++) {
        ACLCHECK(aclrtSetDevice(devices[i]));
    }
    // 初始化通信域
    HCCLCHECK(HcclCommInitAll(ndev, devices, comms));

    // 启动线程执行集合通信操作
    std::vector<std::unique_ptr<std::thread> > threads(ndev);
    struct ThreadContext args[ndev];
    for (uint32_t i = 0; i < ndev; i++) {
        args[i].device = i;
        args[i].comm = comms[i];
        threads[i].reset(new (std::nothrow) std::thread(&Sample, (void *)&args[i]));
    }

    for (uint32_t i = 0; i < ndev; i++) {
        threads[i]->join();
    }

    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例仅支持单机8卡的组网，且仅支持单进程方式拉起。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;

    uint64_t count = 8;
    int mallocSize = count * sizeof(float);
    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));

    // 执行AllReduce操作
    HCCLCHECK(HcclAllReduce((void *)sendBuff, (void*)recvBuff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->comm, stream));
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }
    // 释放通信用sendBuf、recvBuf内存、stream等资源
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    return 0;
}

int main() {
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    uint32_t ndev = 8;
    int32_t devices[8] = {0, 1, 2, 3, 4, 5, 6, 7};
    HcclComm comms[ndev];
    for (int32_t i = 0; i < ndev; i++) {
        ACLCHECK(aclrtSetDevice(devices[i]));
    }
    // 初始化通信域
    HCCLCHECK(HcclCommInitAll(ndev, devices, comms));
    // 启动线程执行集合通信操作
    std::vector<std::unique_ptr<std::thread> > threads(ndev);
    struct ThreadContext args[ndev];
    for (uint32_t i = 0; i < ndev; i++) {
        args[i].device = i;
        args[i].comm = comms[i];
        threads[i].reset(new (std::nothrow) std::thread(&Sample, (void *)&args[i]));
        std::chrono::seconds duration(6);
        std::this_thread::sleep_for(duration);
    }
    for (uint32_t i = 0; i < ndev; i++) {
        threads[i]->join();
    }
    // 释放通信域等相关资源
    for (uint32_t i = 0; i < ndev; i++) {
         HCCLCHECK(HcclCommDestroy(comms[i]));
    }
    std::cout << "end end end" << std::endl;
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    return 0;
}
```

---

# HcclCreateSubCommConfig方式创建子通信域

## 概述

HcclCreateSubCommConfig接口可以基于全局通信域创建子通信域。全局通信域可以通过ranktable文件或root info协商方式创建。本文档以基于ranktable文件创建的全局通信域为例。

## 准备ranktable文件

需要准备ranktable文件来配置集群信息。配置环境变量：

```
export RANK_TABLE_FILE=/home/test/ranktable.json
```

### ranktable.json配置示例

以Atlas A2训练系列产品单机8卡为例：

```json
{
  "status": "completed",
  "version": "1.0",
  "server_count": "1",
  "server_list": [{
    "server_id": "SERVER_ID_SV1",
    "device": [
      {
        "device_id": "0",
        "device_ip": "192.168.1.8",
        "rank_id": "0"
      },
      {
        "device_id": "1",
        "device_ip": "192.168.1.9",
        "rank_id": "1"
      },
      {
        "device_id": "2",
        "device_ip": "192.168.1.10",
        "rank_id": "2"
      },
      {
        "device_id": "3",
        "device_ip": "192.168.1.11",
        "rank_id": "3"
      },
      {
        "device_id": "4",
        "device_ip": "192.168.1.12",
        "rank_id": "4"
      },
      {
        "device_id": "5",
        "device_ip": "192.168.1.13",
        "rank_id": "5"
      },
      {
        "device_id": "6",
        "device_ip": "192.168.1.14",
        "rank_id": "6"
      },
      {
        "device_id": "7",
        "device_ip": "192.168.1.15",
        "rank_id": "7"
      }
    ]
  }]
}
```

## HcclSend/HcclRecv操作代码样例

该样例支持从单机N卡组网中切分出1个4卡子通信域（N需>=4且<=8）。仅属于子通信域的4张卡可执行此用例。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include <cstring>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm comm;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    // 申请通信用device、sendBuf，recvBuf内存、stream等资源
    ACLCHECK(aclrtSetDevice(ctx->device));
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    void* sendBuff;
    void* recvBuff;
    void* hostBuff;
    uint64_t count = 4;
    int mallocSize = count * sizeof(float);

    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&hostBuff, mallocSize));
    float* tmpHostBuff = static_cast<float*>(hostBuff);
    for (uint32_t i = 0; i < count; ++i) {
        tmpHostBuff[i] = 2;
    }
    ACLCHECK(aclrtMalloc((void**)&sendBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMemcpy((void*)sendBuff, mallocSize, (void*)hostBuff, mallocSize, ACL_MEMCPY_HOST_TO_DEVICE));
    ACLCHECK(aclrtMalloc((void**)&recvBuff, mallocSize, ACL_MEM_MALLOC_HUGE_FIRST));

    // 执行SendRecv操作
    if (ctx->device / 2 == 0) {
        HCCLCHECK(HcclSend(sendBuff, count, HCCL_DATA_TYPE_FP32, ctx->device + 2, ctx->comm, stream));
    } else {
        HCCLCHECK(HcclRecv(recvBuff, count, HCCL_DATA_TYPE_FP32, ctx->device - 2, ctx->comm, stream));
    }
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device / 2 == 1) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, mallocSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, mallocSize, (void*)recvBuff, mallocSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    // 释放通信用sendBuf、recvBuf内存，stream等资源
    ACLCHECK(aclrtFreeHost(hostBuff));
    ACLCHECK(aclrtFree(recvBuff));
    ACLCHECK(aclrtFree(sendBuff));
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtResetDevice(ctx->device));
    return 0;
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));

    // 创建并初始化通信域配置项
    HcclCommConfig config;
    HcclCommConfigInit(&config);
    // 根据需要修改通信域配置
    config.hcclBufferSize = 50;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm globalHcclComm;
    HcclCommInitClusterInfoConfig(rankTableFile, devId, &config, &globalHcclComm);
    HcclComm hcclComm;
    strcpy(config.hcclCommName, "comm_2");
    uint32_t rankIds[4] = {0, 1, 2, 3};
    HCCLCHECK(HcclCreateSubCommConfig(&globalHcclComm, 4, rankIds, 1, devId, &config, &hcclComm));
    struct ThreadContext args;
    args.comm = hcclComm;
    args.device = devId;
    Sample((void *)&args);
    HCCLCHECK(HcclCommDestroy(hcclComm));
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

## HcclAllReduce操作代码样例

该样例支持从单机8卡组网中切分出2个4卡子通信域。

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <thread>
#include <chrono>
#include <cstring>
#include "hccl/hccl.h"
#include "hccl/hccl_types.h"
#include "mpi.h"

#define ACLCHECK(ret) do { \
    if(ret != ACL_SUCCESS)\
    {\
        printf("acl interface return err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret);\
        return ret;\
    }\
} while(0)

#define HCCLCHECK(ret) do {  \
    if(ret != HCCL_SUCCESS) \
    {   \
        printf("hccl interface return errreturn err %s:%d, retcode: %d \n", __FILE__, __LINE__, ret); \
        return ret;\
    } \
} while(0)

struct ThreadContext {
    HcclComm commFront;
    HcclComm commBack;
    int32_t device;
};

int Sample(void *arg)
{
    ThreadContext* ctx = (ThreadContext *)arg;
    void* host_buf = nullptr;
    void* send_buff = nullptr;
    void* recv_buff = nullptr;
    uint64_t count = 1;
    int malloc_kSize = count * sizeof(float);
    aclrtEvent start_event, end_event;
    aclrtStream stream;
    ACLCHECK(aclrtCreateStream(&stream));
    ACLCHECK(aclrtCreateEvent(&start_event));
    ACLCHECK(aclrtCreateEvent(&end_event));

    //申请集合通信操作的内存
    ACLCHECK(aclrtMalloc((void**)&send_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));
    ACLCHECK(aclrtMalloc((void**)&recv_buff, malloc_kSize, ACL_MEM_MALLOC_HUGE_FIRST));

    //初始化输入内存
    ACLCHECK(aclrtMallocHost((void**)&host_buf, malloc_kSize));
    ACLCHECK(aclrtMemcpy((void*)send_buff, malloc_kSize, (void*)host_buf, malloc_kSize, ACL_MEMCPY_HOST_TO_DEVICE));

    //执行集合通信操作
    if (ctx->device < 4) {
        HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->commFront, stream));
    } else {
        HCCLCHECK(HcclAllReduce((void *)send_buff, (void*)recv_buff, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, ctx->commBack, stream));
    }
    //等待stream中集合通信任务执行完成
    ACLCHECK(aclrtSynchronizeStream(stream));

    if (ctx->device < 8) {
        void* resultBuff;
        ACLCHECK(aclrtMallocHost((void**)&resultBuff, malloc_kSize));
        ACLCHECK(aclrtMemcpy((void*)resultBuff, malloc_kSize, (void*)recv_buff, malloc_kSize, ACL_MEMCPY_DEVICE_TO_HOST));
        float* tmpResBuff = static_cast<float*>(resultBuff);
        for (uint32_t i = 0; i < count; ++i) {
            std::cout <<  "rankId:" << ctx->device << ",i" << i << " " << tmpResBuff[i] << std::endl;
        }
        ACLCHECK(aclrtFreeHost(resultBuff));
    }

    ACLCHECK(aclrtFree(send_buff));
    ACLCHECK(aclrtFree(recv_buff));
    ACLCHECK(aclrtFreeHost(host_buf));
    //销毁任务流
    ACLCHECK(aclrtDestroyStream(stream));
    ACLCHECK(aclrtDestroyEvent(start_event));
    ACLCHECK(aclrtDestroyEvent(end_event));
    return 0;
}

int main()
{
    MPI_Init(NULL, NULL);
    int procSize = 0;
    int procRank = 0;
    // 获取当前进程在所属进程组的编号
    MPI_Comm_size(MPI_COMM_WORLD, &procSize);
    MPI_Comm_rank(MPI_COMM_WORLD, &procRank);
    int devId = procRank;
    int devCount = procSize;
    // 设备资源初始化
    ACLCHECK(aclInit(NULL));
    // 获取ranktable路径
    char* rankTableFile = getenv("RANK_TABLE_FILE");
    // 指定集合通信操作使用的设备
    ACLCHECK(aclrtSetDevice(devId));

    // 创建并初始化通信域配置项
    HcclCommConfig config;
    HcclCommConfigInit(&config);
    // 根据需要修改通信域配置
    config.hcclBufferSize = 50;
    strcpy(config.hcclCommName, "comm_1");

    HcclComm globalHcclComm;
    HcclCommInitClusterInfoConfig(rankTableFile, devId, &config, &globalHcclComm);
    struct ThreadContext args;
    if (devId < 4) {
        HcclComm hcclCommFront;
        strcpy(config.hcclCommName, "comm_2");
        uint32_t rankIdsFront[4] = {0, 1, 2, 3};
        HCCLCHECK(HcclCreateSubCommConfig(&globalHcclComm, 4, rankIdsFront, 1, devId, &config, &hcclCommFront));
        args.commFront = hcclCommFront;
        args.device = devId;
        Sample((void *)&args);
        HCCLCHECK(HcclCommDestroy(hcclCommFront));
    } else {
        HcclComm hcclCommBack;
        strcpy(config.hcclCommName, "comm_3");
        uint32_t rankIdsBack[4] = {4, 5, 6, 7};
        HCCLCHECK(HcclCreateSubCommConfig(&globalHcclComm, 4, rankIdsBack, 2, devId - 4, &config, &hcclCommBack));
        args.commBack = hcclCommBack;
        args.device = devId;
        Sample((void *)&args);
        HCCLCHECK(HcclCommDestroy(hcclCommBack));
    }
    // 设备资源去初始化
    ACLCHECK(aclFinalize());
    MPI_Finalize();
    return 0;
}
```

---

# 样例编译运行

## 介绍目录结构

以HcclComminitRootInfo初始化方式为例，样例代码目录结构如下所示：

```
CommInitRootInfo
├── bin
│   └── CommInitTest
├── Makefile
└── src
    └── main.cc
```

- Makefile文件内容可参见配置Makefile文件部分
- main.cc的样例代码可参见HcclCommInitRootInfo初始化方式
- bin目录下的CommInitTest为编译后生成的可执行文件

## 配置环境变量

配置样例编译时依赖的环境变量：

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
export INSTALL_DIR=/usr/local/Ascend/ascend-toolkit/latest
export PATH=/usr/local/mpich-3.2.1/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/mpich-3.2.1/lib:${INSTALL_DIR}:$LD_LIBRARY_PATH
```

- "INSTALL_DIR"是CANN软件安装后文件存储路径，其中"/usr/local/Ascend"为root用户的默认安装路径，如果使用普通用户安装，或指定路径安装，请自行替换
- "/usr/local/mpich-3.2.1/lib"为安装后MPI的库文件所在路径，请根据实际情况替换

## 配置Makefile文件

```makefile
#
#loading path
#--------------------------------------------------------------------------------------------------------------------------------------------------
CXXFLAGS := -std=c++11\
        -Werror\
        -fstack-protector-strong\
        -fPIE -pie\
        -O2\
        -s\
        -Wl,-z,relro\
        -Wl,-z,now\
        -Wl,-z,noexecstack\
        -Wl,--copy-dt-needed-entries
Common_DIR = ./src
Common_SRC = $(wildcard ${Common_DIR}/*.cc)
HCCL_INC_DIR = ${ASCEND_DIR}/include
HCCL_LIB_DIR = ${ASCEND_DIR}/lib64
ACL_INC_DIR = ${ASCEND_DIR}/include
ACL_LIB_DIR = ${ASCEND_DIR}/lib64
MPI_INC_DIR = ${MPI_HOME}/include
MPI_LIB_DIR = ${MPI_HOME}/lib
LIST = CommInitTest
#
#library flags
#--------------------------------------------------------------------------------------------------------------------------------------------------
LIBS = -L$(HCCL_LIB_DIR) -lhccl\
                -L$(ACL_LIB_DIR) -lascendcl\
                -L$(MPI_LIB_DIR) -lmpi
INCLUDEDIRS = -I$(HCCL_INC_DIR)\
                                -I$(ACL_INC_DIR)\
                                -I$(MPI_INC_DIR)\
                                -I$(Common_DIR)
#
#make
#--------------------------------------------------------------------------------------------------------------------------------------------------
all:
        @mkdir -p bin
        g++ $(CXXFLAGS) $(Common_SRC) $(INCLUDEDIRS)  -o CommInitTest $(LIBS)
        @printf "\nCommInitTest compile completed\n"
        mv $(LIST) ./bin
.PHONY: clean
clean:
        rm -rf ./bin/*Test
```

## 编译样例

执行如下命令编译样例：

```
make MPI_HOME=/usr/local/mpich-3.2.1/ ASCEND_DIR=/usr/local/Ascend/ascend-toolkit/latest
```

编译成功后，会在执行编译命令的bin/目录下生成CommInitTest可执行文件。

## 执行样例

编译完成后，通过以下指令启动：

```
mpirun -n <npu_num> ./bin/CommInitTest
```

其中"-n"是需要启动的NPU总数。

**注意：** HcclCommInitAll初始化方式为单进程多线程的拉起方式，所以该初始化方式的样例执行命令为 `mpirun -1 ./bin/CommInitTest`。

---

# 常用工具与配置

## HCCL业务配置

HCCL提供了一系列环境变量用于进行集合通信业务侧配置，详细可参见集合通信。

## HCCL性能测试

分布式训练场景下，开发者可以通过HCCL Test工具测试集合通信的功能以及性能。此工具支持测试的集合通信算子包含：
- all_gather
- all_reduce
- alltoallv
- alltoall
- broadcast
- reduce_scatter
- reduce
- scatter

当前集群场景的集合通信常用的是all_reduce和alltoallv算子，因此测试集合通信性能当前也建议基于all_reduce和alltoallv算子。

HCCL Test工具的详细使用可参见《HCCL性能测试工具使用指南》。

---

# FAQ

- [HCCL常见问题总结](ug_000037.md)
- [HCCP常见问题总结](hcclug_000049.html)
- [HCCL Test常见问题总结](hcclug_000056.html)

---

# HCCL常见问题总结

- [环境变量配置错误（EI0001）](ug_000038.md)
- [执行通信操作超时（EI0002）](ug_000039.md)
- [集合通信操作参数无效（EI0003）](ug_000040.md)
- [无效的RankTable配置（EI00004）](ug_000041.md)
- [卡间集合通信参数不一致（EI0005）](ug_000042.md)
- [建链接超时（EI0006）](ug_000043.md)
- [CANN版本不一致，HCCL返回错误码EI0008](ug_000045.md)

---

# 环境变量配置错误(EI0001)

## 问题现象

执行日志报错：**EI0001 "Environment variable [***] is invalid"**，其中"***"是报错的环境变量名称。

## 原因分析

环境变量配置有问题，通常是参数超过可配置范围或在识别范围以外。

### 表1 EI0001报错信息关键字汇总

| 报错信息关键字                                                                                                                    | 含义                                                                                        |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| RankIpFamily rank[ ] device ip family[ ] is not same with others[ ].                                                              | IPv4和IPv6混用。                                                                            |
| HCCL_CONNECT_TIMEOUT it should be a number greater than or equal to 120s and less than or equal to 7200s                          | HCCL_CONNECT_TIMEOUT的取值不在允许范围内。                                                  |
| HCCL_INTRA_PCIE_ENABLE or HCCL_INTRA_ROCE_ENABLE HCCL_INTRA_PCIE_ENABLE and HCCL_INTRA_ROCE_ENABLE cannot be both configured to 1 | HCCL_INTRA_PCIE_ENABLE和HCCL_INTRA_ROCE_ENABLE两配置互斥，不能同时配置为1。                 |
| HCCL_WHITELIST_DISABLE It must be 0 or 1.                                                                                         | HCCL_WHITELIST_DISABLE的取值不在允许范围内。                                                |
| HCCL_WHITELIST_FILE Please check env config                                                                                       | HCCL_WHITELIST_FILE配置有问题，通常为HCCL通信白名单配置文件内容异常，或者文件不存在。       |
| HCCL_IF_IP it should be ip[ ]                                                                                                     | HCCL_IF_IP配置的IP格式不正确。                                                              |
| HCCL_SOCKET_IFNAME Please check env config                                                                                        | HCCL_SOCKET_IFNAME配置格式不正确，请确认"，"位置。                                          |
| HCCL_SOCKET_FAMILY it should be AF_INET or AF_INET6                                                                               | HCCL_SOCKET_FAMILY配置参数不正确，需要是AF_INET或者AF_INET6。                               |
| HCCL_IF_BASE_PORT Value range[0,65520]                                                                                            | HCCL_IF_BASE_PORT的取值不在允许范围内。                                                     |
| HCCL_ALGO expect: levelX:algo1;levelY:algo2                                                                                       | HCCL_ALGO配置错误，通常为格式不符合要求、长度异常或内容不符合预期（重复配置、字段不正确）。 |
| HCCL_RDMA_TC Value range[0, 255], Must be a multiple of 4                                                                         | HCCL_RDMA_TC配置错误，通常为数值超范围、非数字、长度过长等。                                |
| HCCL_RDMA_SL Value range[0, 7]                                                                                                    | HCCL_RDMA_SL的取值不在允许范围内。                                                          |
| HCCL_RDMA_TIMEOUT Value range[5, 24]                                                                                              | HCCL_RDMA_TIMEOUT的取值不在允许范围内。                                                     |
| HCCL_RDMA_RETRY_CNT Value range[1, 7]                                                                                             | HCCL_RDMA_RETRY_CNT的取值不在允许范围内。                                                   |
| HCCL_BUFFSIZE Value should be equal to or greater than 1(MB).                                                                     | HCCL_BUFFSIZE的取值不在允许范围内。                                                         |
| HCCL_DETERMINISTIC Value should be true or false.                                                                                 | HCCL_DETERMINISTIC的取值不在允许范围内。                                                    |
| HCCL_ENTRY_LOG_ENABLE It must be 0 or 1.                                                                                          | HCCL_ENTRY_LOG_ENABLE的取值不在允许范围内。                                                 |
| HCCL_INTER_HCCS_DISABLE Value should be true or false.                                                                            | HCCL_INTER_HCCS_DISABLE的取值不在允许范围内。                                               |
| HCCL_OP_EXPANSION_MODE it should be "AI_CPU"                                                                                      | HCCL_OP_EXPANSION_MODE配置只能填AI_CPU                                                      |
| HCCL_EXEC_TIMEOUT it should be a number greater than or equal to 0s and less than or equal to                                     | HCCL_EXEC_TIMEOUT配置不在取值范围内                                                         |
| CM_CHIEF_IP it should be an available ip.                                                                                         | CM_CHIEF_IP配置的IP不可用。                                                                 |
| CM_CHIEF_PORT it should be a unsigned number less than the max port num                                                           | CM_CHIEF_PORT的取值不在允许范围内。                                                         |
| CM_CHIEF_DEVICE it should be a unsigned number less than the max device num                                                       | CM_CHIEF_DEVICE的取值不在允许范围内。                                                       |
| CM_WORKER_IP it should be an available ip.                                                                                        | CM_WORKER_IP配置的IP不可用。                                                                |
| HCCL_WHITELIST_FILE HCCL_WHITELIST_DISABLE is [0] but HCCL_WHITELIST_FILE is not set                                              | HCCL_WHITELIST_DISABLE配置为0，HCCL_WHITELIST_FILE却没有设置。                              |
| HCCL_WHITELIST_FILE hccl whitelist load config file[ ] failed.                                                                    | HCCL_WHITELIST_FILE参数指定的文件打开失败，请确认路径是否正确。                             |

## 解决方法

确认报错提示的"环境变量"配置是否正确，并参见表1的报错信息进行修改。

---

# 执行通信操作超时(EI0002)

## 问题现象

该错误通常出现在算子执行阶段。屏显日志报错信息："EI0002: The wait execution of the Notify register times out."

Plog日志中查询Notify会显示类似的超时信息。

## 原因分析

HCCL算子的task在指定集群的每个Device上执行，并通过notify进行状态同步。若任何rank或通信链路在执行前/中发生异常，会导致集群同步失败，剩余卡出现notify wait timeout。

常见原因包括：

1. 部分卡被耗时较长的任务阻塞，超过设置的执行同步等待时间（可通过环境变量HCCL_EXEC_TIMEOUT配置）后才继续执行
2. 部分rank未执行到notify同步阶段
3. 网络模型等原因导致某些rank间的task执行序列不一致
4. 节点间通信链路不稳定
5. 训练场景下，用户设置的HCCL_EXEC_TIMEOUT小于HCCL_CONNECT_TIMEOUT时，建链失败，先报notify wait超时

## 解决方法

收集所有卡的plog日志后，按以下步骤排查：

1. 检查所有卡的报错情况，若有卡未报notify超时错误，检查此卡是否存在业务进程报错退出、卡死或core宕机的情况

2. 若所有卡均上报notify超时错误，检查错误日志中最早报错和最晚报错时间差是否超过算子执行超时阈值。若超过阈值，定位报错时间最晚的rank执行阻塞原因或通过 `export HCCL_EXEC_TIMEOUT=3600` 调整超时阈值

3. 检查集群中是否存在Device网口通信链路不稳定的情况。搜索所有卡的Device侧日志，若存在error cqe的打印且时间位于业务区间内，可能是网络拥塞，需要排查交换机配置和训练过程中的网口link down情况

搜索命令样例：`grep –rn "err cqe" | grep HCCL`

---

# 集合通信操作参数无效(EI0003)

## 问题现象

执行日志报错：EI0003 "value [***] for parameter [***] is invalid"

```
custom_group :None
test_type :all
dtype :float32
data :1024
iter :1
timeout ：None
profiling :false
staged :false
pid :1036690
[2024-04-24 06:30:56.081654: F ge_plugin.cc:338] [GePlugin] Initialize ge failed, ret : failed
Error Message is:
EI0003: 2024-04-24-06:30:52.388.386 In [HcomInitByFile], value [0] for parameter [rankTablePath] is invalid. Reason: The collective communication operator has an invalid argument. Reason[0]
        Solution: Try again with a valid argument.
        TraceBack (most recent call last):
        PluginManager InvokeAll failed.[FUNC:Initialize] [FILE:ops_kernel_manager.cc][LINE :89]
        OpsManager initialize failed.[FUNC:InnerInitialize] [FILE:gelib.cc] [LINE:241]
        GELib::InnerInitialize failed.[FUNC :Initialize][FILE:gelib.cc][LINE:169]
        GEInitialize failed. [FUNC:GEInitialize] [FILE:ge_api. cc][LINE :307]
```

## 原因分析

该报错常见于输入数据校验阶段，通常可能有以下几种原因：

1. 接口入参为指针，但传入指针为空。
2. 接口传入的参数超过预期枚举值范围。
3. 接口传入的字符串无效，或长度不符合预期。
4. 接口传入的路径无效，或者路径文件不符合预期。

具体报错字段及报错原因可以参考日志提示信息"value[]"和"parameter[]"进行确认。

## 解决方法

针对日志提示字段"value[]"和"parameter[]"进行排查，排查原因见上述原因分析。

---

# 无效的RankTable配置(EI00004)

## 问题现象

执行日志报错：EI0004 "The ranktable or rank is invalid"

```
custom_group :None
dtype :float32
data :1
iter :1
profiling :false
pid :1040540
[2024-04-24 06:31:38.087571: F ge_plugin.cc:338] [GePlugin] Initialize ge failed, ret :failed
Error Message is:
EI0004: 2024-04-24-06:31:33.915.634 The ranktable or rank is invalid, Reason:[The ip in ranktable is not a valid ip address]. Please check the configured ranktable. [The ranktable path configured in the training can be found in the plogs.]
        Solution: Try again with a valid cluster configuration in the ranktable file. Ensure that the configuration matches the operating environment.
        TraceBack (most recent call last):
        PluginManager InvokeAll failed. [FUNC:Initialize][FILE:ops_kernel_manager.cc][LINE :89]
        OpsManager initialize failed. [FUNC:InnerInitialize][FILE:gelib.cc] [LINE :241]
        GELib::InnerInitialize failed. [FUNC:Initialize] [FILE:gelib.cc][LINE:169]
        GEInitialize failed.[FUNC:GEInitialize][FILE:ge_api.cc][LINE :307]
```

## 原因分析

该报错常见于rank id或ranktable的数据校验阶段，通常有以下几种原因：

1. rank id的值不符合预期（过大的非法值、非数值、超过ranktable中的rank个数），或者同Server内已有rank id重复。
2. ranktable的格式或参数错误：版本不对，路径不对，格式不对，起始rank不为0，Server内的rank个数不正确，server_count、server_list、super_pod_id、server_id、server_index、ip等参数为空或配置不正确。

## 解决方法

针对执行日志提示字段中的"Reason"，可以基本看出问题方向，基于此排查配置下发参数的rank id或者ranktable，即可确认问题点。

---

# 卡间集合通信参数不一致(EI0005)

## 问题现象

执行日志报错：EI0005"The arguments for collective communication are inconsistent between ranks"。

示例错误日志显示：
- custom_group :None
- test_type : sum
- dtype :float32
- data :1024
- fusion :0
- fusion_id :1
- fusion_num :2
- iter :1
- profiling :false
- para_err_type :1
- pid :1052803
- ranksize is 8, rankid is 4.

关键错误信息：
"EI0005: 2024-04-24-06:32:27.781.599 The arguments for collective communication are inconsistent between ranks:tag[HcomAllReduce_6629421139219749105_0],parameter[count],local[16512],remote [8320]"

## 原因分析

该报错源于本端和对端的校验参数不一致，通常发生在建链阶段。建链过程中，本端接收对端校验帧后与本地数据对比验证。

常见原因包括：上层框架或用户调用传参问题。根据报错信息中的"tag"可确定出错算子，"parameter"字段可确定出错参数。

## 解决方法

- 用户直调场景：根据错误信息中的"parameter"字段，确认算子接口入参的正确性。
- 框架调用场景：获取日志后联系技术支持。

---

# 建链接超时(EI0006)

## 问题现象

常见于算子加载阶段，有以下2种情况：

* Server间的建链超时现象，出现"LINK_ERROR_INFO"报错信息
* Server内的建链超时现象，日志报错信息显示相关错误

## 原因分析

HCCL会在指定集群的每个Device上运行，并在集群间建立socket链接。若任一个rank或通信链路在建链前/中发生异常，则会导致集群建链失败。常见原因包括：

1. 部分rank未执行到正确的建链阶段
2. 部分卡被某些耗时较长的任务阻塞，在超过socket建链超时等待时间后才执行到对应阶段
3. 脚本等原因导致某些rank间的通信算子数量或排序不一致
4. 节点间通信链路不通或不稳定

## 解决方法

收集所有卡的plog日志后，按以下步骤排查：

1. 检查报错情况：若有卡未报建链超时错误，检查此卡是否存在业务卡死情况

2. 检查时间差异：若所有卡均上报建链超时错误，检查错误日志中最早和最晚时间差异是否超过超时阈值。可通过配置 `export HCCL_CONNECT_TIMEOUT=3600` 增大建链超时时间

3. 检查算子一致性：检查各rank执行的集合通信算子数量和加载顺序是否一致。需开启INFO日志，在Host日志中搜索：
  - 关键字1：`GenerateOpTag:graph`
  - 关键字2：`GenerateTask:graph`

4. 检查网络链路：检查集群中是否存在Device网口通信链路不通，常见原因包括：
  - IP不在同一网段或子网掩码配置问题
  - IP冲突
  - 交换机vlan配置不一致
  - 链路不通
  - 各rank的TLS设置不一致

### TLS检查和配置

查询TLS状态命令：
```
hccn_tool -i 0 -tls -g
hccn_tool -i 1 -tls -g
hccn_tool -i 2 -tls -g
hccn_tool -i 3 -tls -g
hccn_tool -i 4 -tls -g
hccn_tool -i 5 -tls -g
hccn_tool -i 6 -tls -g
hccn_tool -i 7 -tls -g
```

关闭TLS功能命令：
```
hccn_tool -i 0 -tls -s enable 0
hccn_tool -i 1 -tls -s enable 0
hccn_tool -i 2 -tls -s enable 0
hccn_tool -i 3 -tls -s enable 0
hccn_tool -i 4 -tls -s enable 0
hccn_tool -i 5 -tls -s enable 0
hccn_tool -i 6 -tls -s enable 0
hccn_tool -i 7 -tls -s enable 0
```

TLS switch值为0表示关闭，1表示开启。如果提示no certificate found，也表示TLS功能关闭。

---

# CANN版本不一致，HCCL返回错误码EI0008

## 问题现象

此错误常见于通信建链阶段，屏显日志报错信息为："The CANN versions are inconsistent"（错误码EI0008）。

## 原因分析

在集合通信建链过程中，系统会校验本端与对端rank的CANN版本一致性。当CANN版本不一致时，HCCL会返回错误并打印错误码EI0008。

## 解决方法

需确认不同服务器上安装的CANN软件版本是否相同，若不一致则需安装一致的版本。

### 具体步骤：

1. 在plog日志（INFO级别）中grep关键字CannVersion，确认各节点的CANN软件版本
2. 重新安装一致的CANN软件版本

---

# HCCP常见问题总结

*   **[EJ0001打屏报错](hcclug_000050.html)**

*   **[EJ0002打屏报错](hcclug_000051.html)**

*   **[EJ0003打屏报错](hcclug_000052.html)**

*   **[EJ0004打屏报错](hcclug_000053.html)**

*   **[参与集合通信的服务器TLS信息不一致，HCCL初始化失败](hcclug_000055.html)**

**父主题：** [FAQ](hcclug_000036.html)

---

# EJ0001打屏报错

## 问题现象

plog日志中，TDT首报错为：`[DeviceMsgProcess][tid:1241254] [TsdClient] DeviceMsgProc errcode[EJ0001]`

```
[ERROR] TDT(685010,all_reduce_test):2023-11-29-11:55:41.334.702 [process_mode_manager.cpp:587][DeviceMsgProcess][tid:685010] [TsdClient] DeviceMsgProc  errcode[EJ0001]
[ERROR] TDT(685010,all_reduce_test):2023-11-29-11:55:41.334.873 [process_mode_manager.cpp:269][WaitRsp][tid:685010] tsd client wait response fail, device response code[1]. unknown device error.
[ERROR] TDT(685010,all_reduce_test):2023-11-29-11:55:41.334.893 [process_mode_manager.cpp:123][OpenProcess][tid:685010] Wait open response from device failed.
[ERROR] TDT(685010,all_reduce_test):2023-11-29-11:55:41.334.897 [tsd_client.cpp:31][TsdOpen][tid:685010] TsdOpen failed, deviceId[4].
```

EJ0001错误只能说明拉起device HCCP进程失败，具体失败原因需要根据device报错进一步区分，在debug目录下，使用`/usr/local/Ascend/driver/tools/msnpureport -f`导出device日志，`grep -rn ERROR * | grep HCCP`查看HCCP首报错，Device报错有以下两种场景。

- Device报错"Create Server failed, ret(61)"

  EJ0001报错，通常看到HCCP报错：`[ra_adp.c:14c_server_init(1425) : Create Server failed, ret(61)`

  ```
  [ERROR] HCCP(13722,hccp_service.bin):2023-11-29-11:55:41.806.183 [ra_adp.c:1425]tid:13722,ra_hdc_server_init(1425) : Create Server failed, ret(61)
  [ERROR] HCCP(13722,hccp_service.bin):2023-11-29-11:55:41.806.200 [ra_adp.c:1546]tid:13722,hccp_init(1546) : chip_id[0] ra_hdc_server_init failed, ret[-22]
  [ERROR] HCCP(13722,hccp_service.bin):2023-11-29-11:55:41.806.265 [main.c:224]tid:13722,main(224) : hccp init error[-22]
  ```

- Device报错"certificate is not yet valid"

## 原因分析与解决方法（Device报错"Create Server failed, ret(61)"）

若Device报错"Create Server failed, ret(61)"，可能有以下两种原因：

### 人为操作问题（偶现）

上一次训练未完全退出，就拉起新的训练。

**原因分析：**

在Host侧执行如下hccn_tool命令：

```
for i in {0..7}; do hccn_tool -i $i -process -g ; done
```

查看Device侧是否存在hccp进程。

**解决办法：**

稍等一会（通常停掉训练任务脚本，会进资源销毁释放流程），确认进程完全退出后再拉起训练进程。

### 业务脚本问题（必现）

每次拉起训练时，会在一个device上拉起2个及以上的hccp进程。

**原因分析：**

执行如下命令，导出日志，并确认是否在临近间隔时间，通常是ms级时间间隔内，有两次及以上hccp进程拉起的日志记录。

```
cd /root/ascend/log/run/
grep -rn "hccp init" *
```

如下所示，即可判定是由于业务脚本在一个Device上拉起2个及以上的HCCP进程导致的，需要用户排查业务脚本。

**解决方法：**

需要自行排查业务训练脚本中问题，如下是几个常见案例。

1. 单机16卡跑hccl test报错：-p与-n不配套导致多次拉起。

2. "pytorch nproc_per_node"参数需要填写节点上要启动的训练进程数，假设只有2个卡可用，但此参数填写的8，所以重复拉起4次，导致出错。如下所示，对应日志中看到重复拉起了4次。

## 原因分析及解决方法（Device报错"certificate is not yet valid"）

**原因分析：**

若Device报错"certificate is not yet valid"，则原因为：

Host时钟异常，导致系统时间早于TLS证书有效时间，TLS开关打开情况下校验证书失败，拉起训练失败。

**解决方法：**

同步正常时间后，可以恢复。

---

# EJ0002打屏报错

## 问题现象

拉起训练进程时报EJ0002 Environment Error，通常由环境异常导致rdev初始化失败。

HCCP初始化ra_rdev阶段报错信息：

```
[ERROR] HCCP(46430,alltoallv_test):2023-09-21-03:43:49.546.469 [ra hdc.c:1270]tid:46430,ra_hdc_rdev_init(1270) : [init][ra_hdc_rdev]ra hdc message process failed ret(-67) phy_id(3)
[ERROR] HCCP(46430,alltoallv_test):2023-09-21-03:43:49.546.488 [ra_host.c:621]tid:46430,ra_rdev_init(621) : [init][ra_rdev]ra rdev init failed. ret(-67)
```

## 原因分析

网卡down导致初始化时，HCCP调用HDC接口返回-67，对应错误码定义：`#define ENOLINK 67 /* Link has been severed */`

## 解决方法

**1. 检查网口状态**

```bash
for i in {0..7}; do hccn_tool -i $i -link -g ; done
```

**2. 用户自行排查物理链路**

- 重新配置ip和netmask（有可能未配置ip）
  ```bash
  hccn_tool -a -cfg recovery
  ```
  基于/etc/hccn.conf中配置恢复环境配置

- 查询光模块是否在位（evb环境无光模块）
  ```bash
  for i in {0..7}; do hccn_tool -i $i -optical -g; done
  ```

- 查询交换机信息
  ```bash
  for i in {0..7}; do hccn_tool -i $i -lldp -g; done
  ```

- 咨询环境人员光纤类型、交换机FEC策略、CDR版本及光模块状态

---

# EJ0003打屏报错

## 问题现象

建链阶段报EJ0003的错误，HCCP调系统函数bind失败，常见报错**error: 98**或**error: 99**。

该报错对应Linux标准报错定义如下：

```
# define EADDRINUSE      98      /* Address already in use */
# define EADDRNOTAVAIL   99      /* Cannot assign requested address */
```

查看plog日志中关于HCCP的报错：

```
grep -rn ERROR * | grep HCCP | head
```

### 报错1：

```
plog/plog-3541138_20231113190511833.log:30551:[ERROR] HCCP(3541138,host_cpu_executor):2023-11-13-19:05:49.833.681 [rs_socket.c:671]tid:3541138,rs_socket_listen_bind_listen(671) : bind fail! family:2, IP:127.0.0.1, port:50000, sock:27, ret:0xffffffff, error:98, Possible Cause: the IP address and port have been bound already

plog/plog-3541138_20231113190511833.log:30552:[ERROR] HCCP(3541138,host_cpu_executor):2023-11-13-19:05:49.833.687 [rs_socket.c:860]tid:3541138,rs_socket_listen_start(860) : bind and listen fail, err_no:98, listen_fd:27, listen state:1, IP(127.0.0.1) server_port:50000

plog/plog-3541138_20231113190511833.log:30554:[ERROR] HCCP(3541138,host_cpu_executor):2023-11-13-19:05:49.833.704 [ra_peer.c:114]tid:3541138,ra_peer_socket_listen_start(114) : [listen_start][ra_peer_socket]ra listen start failed ret(-98).
```

### 报错2：

```
[ERROR] HCCP(160639,all_gather_test):2023-10-17-07:28:13.178.137 [rs_socket.c:671]tid:160639,rs_socket_listen_bind_listen(671) : bind fail! ::350d:755b:add4:ada6, port:60000, sock:25, ret:0xffffffff, error:99, Possible Cause: the IP address and port have been bound already
```

### 报错3：

```
75346:[ERROR] HCCP(13854,python3):2023-09-04-14:37:34.444.964 [rs_socket.c:1124]tid:21717,rs_connect_bind_client(1124) : client bind fail! IP:fe80::4b79:e6c9:31eb:f019, sock:530, ret:-1, error:99

75347:[ERROR] HCCP(13854,python3):2023-09-04-14:37:34.444.968 [rs_socket.c:1164]tid:21717,rs_socket_state_reset(1164) : rs_connect_bind_client failed, ret[-99]
```

## 原因分析

- 配置错误，可能有以下情况：
  - 未配置网卡IP。
  - 网卡IP与业务配置的IP不匹配。
  - 对一个IP地址及端口重复监听了多次。

- Host侧未指定网卡建链，导致进程绑定IP地址报错。
- 网卡状态异常。

## 解决方法

### 1. 检查配置

- 查看Device网卡的IP地址是否存在。

  执行**hccn_tool -i dev_id -ip -g**命令查询对应的Device网卡的IP地址是否存在。

  若不存在，请执行如下命令进行配置。

  ```
  hccn_tool -i dev_id -ip -s address 192.168.0.3 netmask 255.255.255.0
  ```

- 检查ranktable文件中配置的IP地址与网卡实际IP是否一致，若不一致，请修改。

  说明：可以在Device日志中搜索关键字"listen fail"查看监听失败的IP地址。

- 检查应用程序端口与HCCL系统进程端口是否冲突。

  分布式训练场景下，HCCL默认会使用Host服务器的60000-60015端口收集集群信息，若通过环境变量HCCL_IF_BASE_PORT指定了Host网卡起始端口，则需要预留以该端口起始的16个端口。

  可通过**lsof -i:60000**确认占用端口的进程信息，通过**netstat -ant | grep WAIT**命令查看端口状态。

  若确认是应用程序端口与HCCL系统进程端口冲突，可通过如下命令预留系统端口。

  ```
  sysctl -w net.ipv4.ip_local_reserved_ports=60000-60015
  ```

- 检查是否存在IP与端口重复监听的问题。

  设置日志级别为EVENT，然后在日志中**grep -rn "socket bind" ***，确认当前训练端口监听情况。

### 2. 指定网卡初始化

若绑定网卡报错，可通过环境变量HCCL_SOCKET_IFNAME指定初始化Host通信网卡，HCCL可通过该网卡名获取Host IP，完成通信域创建。

配置示例如下：

```
export HCCL_SOCKET_IFNAME=eth0
```

### 3. 检查网卡状态

执行**hccn_tool -i dev_id -link -g**命令查看对应的Device网卡状态，若网卡状态异常，请排查网卡错误。

---

# EJ0004打屏报错

## 问题现象

训练拉起失败，报**EJ0004**错误。HCCP首报错示例：

```
[ERROR] HCCP(1609,python3):2023-10-19-12:33:13.313.594 [ra_host.c:544]tid:1609,ra_rdev_init_check_ip(544) : [check][ip]fail, ret(-16) the IP address(29.4.84.36) in the ranktable is inconsistent with the IP(29.4.76.85)address of the network adapter, please make sure they're consistent. num(1)
```

搜索Error日志方法：

```
grep -rn ERROR | grep HCCP | grep inconsistent
```

发现当前节点ranktable中的ip和device ip不一致的多个错误实例。

## 原因分析

此错误通常由业务配置错误导致。

## 解决方法

请检查ranktable配置，确认是否存在以下问题：

- **存在重复的rank id**

  若存在重复rank id，请修改，确保rank id全局唯一。

- **配置的Device IP地址与实际Device的网卡地址不一致**

  可通过如下命令查询Device IP：

  ```
  for i in `seq 0 7`; do echo "===================> dev$i, NPU$((i+1))"; hccn_tool -i $i -ip -g; done
  ```

  若不一致，请修改配置的Device IP地址为实际的网卡地址。

---

# 参与集合通信的服务器TLS信息不一致，HCCL初始化失败

## 问题现象

分布式训练场景下，集合通信建链失败，HCCL关键日志信息显示连接错误。HCCP Debug日志中出现以下错误：

"ssl fd 42 err 1 errno 0, err code 167773208, err msg error:0A000418:SSL routines:tlsv1 alert unknown ca, The possible cause is that the TLS switch is inconsistent."

## 原因分析

参与集合通信的各服务器TLS状态开关不一致，或当TLS状态开关统一打开时，TLS证书信息不一致。

## 解决方法

### 1. 查询集合通信的各服务器TLS状态开关

在服务器中执行命令获取TLS开关使能状态：

```
hccn_tool -i <device_id> -tls -g [host]
```

查询所有Device设备的TLS信息：

```
for i in `seq 0 7`; do hccn_tool -i $i -tls -g; done
```

其中tls switch[0]代表TLS状态为关闭，switch[1]代表TLS状态为使能。

### 2. 判断各服务器中所有Device的TLS状态开关是否一致

- **若不一致**：建议统一修改TLS状态为使能。若TLS开关关闭，集合通信时会存在信息被窃听、篡改、仿冒的风险。

  修改命令：
  ```
  hccn_tool -i <device_id> -tls -s enable 1
  ```

- **若一致且状态为使能**：继续执行步骤3判断各节点的TLS证书信息是否一致。

### 3. 查看所有服务器中各Device的TLS证书信息是否一致

若证书信息不一致，可通过以下命令替换证书套件：

```
hccn_tool -i 0 -tls -s path /root pri pri.pem pub pub.pem ca1 ca1.pem ca2 ca2.pem crl xxx.crl
```

参数说明：
- `-i`:Device ID
- `-path`：指定证书/私钥/吊销列表存放路径
- `pri`：私钥名字
- `pub`：设备证书文件名
- `ca1/ca2/crl`：根证书、二级根证书、吊销列表文件名

更多用法及参数解释可查看《HCCN Tool接口参考》。

---

# HCCL Test常见问题总结

*   **[单机带宽低](hcclug_000057.html)**

*   **[多机峰值带宽低](hcclug_000058.html)**

*   **[小数据量时延大](hcclug_000059.html)**

*   **[多机测试大数据量带宽骤降为0](hcclug_000060.html)**

*   **[测试命令卡数与实际卡数不一致，返回retcode 11](hcclug_000061.html)**

*   **[未设置hostname与ip对应关系导致MPI error报错](hcclug_000062.html)**

*   **[HCCLComm初始化失败](hcclug_000063.html)**

*   **[防火墙未关闭](hcclug_000064.html)**

*   **[SSH连接超时错误](hcclug_000065.html)**

*   **[多台机器MPI版本不一致问题](hcclug_000066.html)**

*   **[Device网络不通报错retcode 4](hcclug_000067.html)**

*   **[host ip之间两两不通报MPI错误](hcclug_000068.html)**

*   **[网口down导致报错ret(-67)](hcclug_000069.html)**

*   **[Device IP冲突导致get socket超时](hcclug_000070.html)**

*   **[未指定网卡名报错retcode 9](hcclug_000071.html)**

*   **[hostfile与测试命令不匹配报错retcode 11](hcclug_000072.html)**

*   **[大规模集群场景报建链失败错误](hcclug_000073.html)**

*   **[大规模集群场景HCCP报errno 24](hcclug_000074.html)**

**父主题：** [FAQ](hcclug_000036.html)

---

# 单机带宽低

## 问题现象

使用HCCL Test工具测试单机带宽时，单机带宽峰值低于预期值。

## 原因分析

1. 日志等级设置不是error。

   若某些卡的日志等级设置不是error，可能会导致对应卡的带宽降低（某些场景下，可能会导致带宽降低2GB左右）。

2. 某条链路带宽低。

   某条链路带宽低会导致单机整机单宽低。

## 解决步骤

1. 若是日志级别设置不是error导致的带宽降低，可通过重新设置Host侧与Device侧日志级别为error解决。

   Host侧日志级别：
   ```
   export ASCEND_GLOBAL_LOG_LEVEL=3   // (0:debug 1:info 2:warning 3:error)
   ```

   Device侧日志操作：
   ```
   # 查询
   for i in {0..7}; do /usr/local/Ascend/driver/tools/msnpureport -r -d $i; done
   # 设置
   for i in {0..7}; do msnpureport -g error -d $i; done
   ```

2. 若重设日志级别后，带宽仍然较低，可分析是否某条链路HCCS带宽低，或者PCIe带宽低。

   可通过ascend-dmi命令测p2p的带宽，命令示例如下：

- 测试卡0到卡8的链路带宽：
     ```
     ascend-dmi --bw -t p2p --ds 0 --dd 8
     ```

- 测试全量p2p带宽：
     ```
     ascend-dmi --bw -t p2p
     ```

     全量p2p带宽测试时间较长，预计20分钟左右。

   获取以上数据后，联系技术支持。

---

# 多机峰值带宽低

## 问题现象

使用HCCL Test工具测试多机带宽时，多机带宽峰值低于预期值。

## 原因分析

- 使用HCCL Test性能测试工具时，未设置HCCL_BUFFSIZE环境变量，或者测试命令中设置的测试数据量较小。
- 开启了Profiling功能或日志等级未设置为默认的ERROR级别，会导致带宽较低。
- 某台机器带宽较低，导致多机带宽低。
- 交换机负载分担方式配置不合理，产生拥塞。
- spine和leaf交换机之间的某个端口状态异常。

## 解决步骤

### 1. 确认HCCL_BUFFSIZE环境变量是否设置

环境变量HCCL_BUFFSIZE用于调整两个NPU之间共享数据的缓存区大小，默认为200M。在性能测试场景下，可适当增大该值提升通信效率。

配置示例：
```
export HCCL_BUFFSIZE=2048
```

### 2. 确认hccl_test测试命令中的参数"-e"是否配置过小

"-e"代表测试数据大小的结束值，若值较小则带宽会较小，建议增大参数值。

示例：
```
mpirun -n 8 ./bin/all_reduce_test -b 8K -e 4G -f 2 -d fp32 -o sum -p 8
```

### 3. 查看是否开启了Profiling性能数据采集功能

Profiling功能会导致带宽变低，请关闭此功能。详细方法可参见《性能调优工具指南》中的"性能分析"章节。

### 4. 查看日志级别是否为"ERROR"

**查看日志级别：**

- Host侧：`echo $ASCEND_GLOBAL_LOG_LEVEL`(0:debug, 1:info, 2:warning, 3:error)
- Device侧：`for i in {0..7}; do /usr/local/Ascend/driver/tools/msnpureport -r -d $i; done`

**修改日志级别：**

- Host侧：`export ASCEND_GLOBAL_LOG_LEVEL=3`
- Device侧：`for i in {0..7}; do msnpureport -g error -d $i; done`

### 5. 检查某台机器带宽或网络配置

通过二分法找到带宽较低的机器，参考单机带宽低的排查方法。检查所有服务器配置一致性：
```
cat /etc/hccn.conf
```

### 6. 检查交换机负载分担方式配置

执行命令查看服务器统计信息：
```
for i in $(seq 0 15); do echo "==============> $i"; hccn_tool -i $i -stat -g |grep pfc ;done
```

若出现多个"rx pfc"表示负载分担不均衡产生拥塞。解决方法：

- 解决交换机的pfc问题，使pfc很少甚至没有
- 检查交换机流量是否不均衡
- 若使用udp端口进行哈希选路，检查某些机器的udp port配置

---

# 小数据量时延大

## 问题现象

在小数据量场景下出现较大时延，具体表现为"数据量小于4M时单机8卡allreduce时延大于300us"。

## 原因分析

Device日志级别设置不当是导致性能问题的根本原因。若日志级别未保持为默认的ERROR级别，则会引发显著的时延增加。

可通过执行以下命令查询当前Device日志等级：

```bash
for i in {0..7}; do /usr/local/Ascend/driver/tools/msnpureport -r -d $i; done
```

## 解决步骤

执行下述命令将Device日志等级调整至error：

```bash
for i in {0..7}; do msnpureport -g error -d $i; done
```

---

# 多机测试大数据量带宽骤降为0

## 问题现象

多机大数据量进行性能测试时出现以下情况：

* 带宽骤降为0点几。
* 查看统计数据，重传次数(roce_new_pkt_rty_num)会增长。
* 设置**export HCCL_RDMA_TIMEOUT=10** 缩短重传等待时间，带宽不会骤降为0点几。

## 原因分析

"数据转发过程中发生丢包，产生了重传，从而拖慢时间。"

## 解决步骤

根因是找到丢包所在的位置，例如spine交换机、leaf交换机、服务器、线缆等，可按照如下顺序排查丢包的地方。

### 1. 丢包在线缆，端口闪断丢包

查询端口link记录

```
for i in $(seq 0 7); do echo "==============> $i"; hccn_tool -i $i -link_stat -g ;done
```

找到闪断的端口，然后排查硬件闪断原因。

示例输出：
```
[devid 7]current time Wed Dec 20 17:13:50 2023
[devid 7]link up count 2
[devid 7]link down count 1
[devid 7]link change records
[devid 7] Thu Dec ：14 10:57:34 2023 LINK UP
[devid 7] Thu Dec 14 10:57:33 2023 LINK DOWN
[devid 7] Thu Dec 14 10:57:19 2023 LINK UP
```

### 2. 丢包在交换机——交换机配置原因

需要排查交换机配置，分别登录spine和leaf交换机检查丢包原因，可能存在如下原因：

* 交换机buffer水线设置不合理，流控方式不对。
* 只配了spine的PFC，没有配leaf交换机PFC，spine反压给leaf，leaf没有降速，导致丢包。
* ECN和PFC水线配置配合不合理导致丢包，若不是一定要用ECN，建议关掉ECN。

### 3. 丢包在交换机——交换机和服务器配合原因

交换机和服务器流控参数配置不匹配，流量进错队列丢包。

交换机和服务器的dscp、pfc优先级队列等需要协商一致，原理请参见《Ascend Training Solution组网指南》中的"拥塞控制与纠错配置策略"章节。

#### 排查交换机配置

可以在交换机侧看对应的报文dscp对不对，有没有走到对应的队列

在交换机看是否进到对应队列，命令：
```
display qos queue statistics interface 10ge 1/0/1
```

#### 排查服务器流控参配置

查询dscp_to_tc
```
for i in $(seq 0 7); do echo "==============> $i"; hccn_tool -i $i -dscp_to_tc -g ;done
```

查询PFC
```
for i in $(seq 0 7); do echo "==============> $i"; hccn_tool -i $i -pfc -g;done
```

设置dscp_to_tc
```
for i in $(seq 0 7); do echo "==============> $i"; hccn_tool -i $i -dscp_to_tc -s dscp 33 tc 2;done
```

设置PFC
```
for i in $(seq 0 7); do echo "==============> $i";hccn_tool -i $i -pfc -s bitmap 0,0,0,0,1,0,0,0;done
```

#### 排查流控环境变量

测试之前要确保这两个环境变量已经设置，不配置采用默认参数(TC=132，SL=4)

```
export HCCL_RDMA_TC=132    # 数值为dscp左移两位 （乘以4），根据和交换机协商配置
export HCCL_RDMA_SL=4      # 数值为PFC的队列，根据和交换机协商配置
```

### 4. 丢包在服务器（TC buffer溢出导致丢包）

原因可能是服务器buffer设置不合理，或者是交换机未响应服务器的PFC，最大使用的buffer达到了TCbuffer最大值，buffer溢出导致服务器丢包。

#### 排查服务器统计是否有tx pfc

```
for i in $(seq 0 15); do echo "==============> $i"; hccn_tool -i $i -stat -g ;done | grep pfc
```

#### 排查buffer和上下水线，判断使用的最大buffer是否达到了设置的buffer上限

查询tc buffer配置
```
for i in $(seq 0 15); do hccn_tool -i $i -tc_cfg -g;done
```

查询某个TC使用的最大buffer（读清）
```
for i in $(seq 0 15); do hccn_tool -i $i -cur_tc_buf -g tc 1;done #tc根据使用的tc改
```

---

# 测试命令卡数与实际卡数不一致，返回retcode 11

## 问题现象

执行HCCL Test测试命令时，返回"return code 11"错误，例如：

"hccl interface return errreturn err ./common/src/hccl_test_common.cc:499, retcode: 11"

## 原因分析

HCCL Test测试命令中配置的卡数与实际的卡数不一致。如下所示为错误命令示例：

```
mpirun -n 16 ./bin/all_reduce_test -b 8K -e 1G -f 2 -d int8 -o sum -p 8 -c 0
```

* **-n**：需要启动的NPU总数。
* **-p**：单个计算节点上参与训练的NPU个数。

示例命令错误的原因为：这是一个单机测试命令，"-n 16"说明要启动的NPU总数为16个，"-p 8"单个节点上参与训练的NPU个数为8，这个节点的实际卡数达不到配置中的NPU总数16。

## 解决步骤

修改测试命令，检查"-n"与"-p"的NPU个数（进程数）是否配置正确。

* **-n**：需要启动的NPU总数。
* **-p**：单个计算节点上参与训练的NPU个数。

---

**父主题：** HCCL Test常见问题总结

---

# 未设置hostname与ip对应关系导致MPI error报错

## 问题现象

报错MPI error，错误堆栈信息示例：

```
Fatal error in MPI_Init_thread: Other MPI error, error stack:
MPIR_Init_thread(474)
MPID_Init(190)  channel initialization failed
MPIDI_CH3_Init(89)
MPID_nem_init(320).
MPID_nem_tcp_init(173)
MPID_nem_tcp_get_business_card(42θ):
MPID_nem_tcp_init(379).
.: gethostbyname failed, Georgreens-MacBook-Pro.local (errno 1)
```

## 原因分析

"/etc/hosts" 文件中未配置相应的主机IP地址和主机名映射关系。

## 解决步骤

在 /etc/hosts中添加主机IP地址(Host IP)和主机名(Host Name)，示例如下：

```
172.16.0.100 HW-AI-LC-1-1
```

---

**相关主题：** HCCL Test常见问题总结

---

# HCCLComm初始化失败

## 问题现象

执行HCCL Test测试工具时，报"This is an error in init_hcclComm"的错误。

## 原因分析

某些卡被进程占用，导致无法使用HCCL Test工具进行测试。

## 解决步骤

1. 执行"npu-smi info"命令查看卡的占用情况。

   某些场景下，npu-smi info未显示卡被占用，但片上内存使用非常高，此种情况下，也会引起HCCL Test测试工具执行失败。

2. 确认被占用卡上的进程释放后，再重新执行HCCL Test测试工具。

   通常停掉训练任务脚本，会进入资源释放销毁流程，可在Host侧执行如下命令，确认进程完全退出后，再执行测试工具。

```
for i in {0..7}; do hccn_tool -i $i -process -g ; done
```

---

# 防火墙未关闭

## 问题现象

测试报错如下，查询机器防火墙发现防火墙未关闭

```
[root@node-87-66 hccl_test]# mpirun -f hostfile -n 16 ./bin/all_reduce_test -b 8K -e 4G -f 2 -d fp32 -p 8
the minbytes is 8192, maxbytes is 4294967296, iters is 20, warmup_iters is 5
Fatal error in PMPI_Barrier: Unknown error class, error stack:
PMPI_Barrier(425)...............: MPI_Barrier(MPI_COMM_WORLD) failed
MPIR_Barrier_impl(332)..........: Failure during collective
MPIR_Barrier_impl(327)..........:
MPIR_Barrier(292)...............:
MPIR_Barrier_intra(150).........:
barrier_smp_intra(96)...........:
MPIR_Barrier_impl(332)..........: Failure during collective
MPIR_Barrier_impl(327)..........:
MPIR_Barrier(292)...............:
MPIR_Barrier_intra(169).........:
MPIDI_CH3U_Recvq_FDU_or_AEP(629): Communication error with rank 8
barrier_smp_intra(111)..........:
MPIR_Bcast_impl(1452)...........:
MPIR_Bcast(1476)................:
MPIR_Bcast_intra(1287)..........:
MPIR_Bcast_binomial(310)........: Failure during collective
```

## 原因分析

发生此问题的原因一般是防火墙未关闭。

不同系统防火墙查询命令略有不同，例如：

**systemctl status firewalld**

## 解决步骤

通过systemctl命令关闭防火墙，不同系统设置命令略有不同，命令示例如下：

- 关闭防火墙：

  **systemctl stop firewalld**

- 禁用防火墙（开机不启动）：

  **systemctl disable firewalld**

  **sudo ufw status**

**父主题：** HCCL Test常见问题总结

---

# SSH连接超时错误

## 问题现象

在多机场景下执行HCCL Test测试工具时，会报出以下错误：

```
ssh：connect to host xx.xx.xx.xx port 22: Connection time out
```

## 原因分析

此错误是由测试主机无法远程登录到其他机器所引起的。多机测试场景需要配置操作节点到集群通信所有节点的SSH信任关系，以支持集群通信节点远程登录。

## 解决步骤

1. 在当前操作节点生成密钥信息（如环境中已存在，可跳过）：
   ```
   ssh-keygen -t rsa
   ```
   密钥信息生成后存储在 `/root/.ssh/id_rsa.pub` 文件中。

2. 将操作节点公钥复制到集群通信其他节点，实现SSH密钥登录：
   ```
   ssh-copy-id -i /root/.ssh/id_rsa.pub node1_address
   ssh-copy-id -i /root/.ssh/id_rsa.pub node2_address
   ```

3. 在操作节点上SSH远程登录其他节点，确认是否可以直接登录。

---

**父主题：** HCCL Test常见问题总结

---

# 多台机器MPI版本不一致问题

## 问题现象

多机场景下，HCCL Test测试工具执行过程中发生概率性失败，coredump等随机问题。

## 原因分析

可能是多台机器的MPI软件版本不一致导致，可通过如下命令查询MPI软件版本：

```
which mpirun
find / -name mpirun
```

## 解决步骤

当前版本，如果通信网卡使用IPv4协议，建议安装 [MPI 3.2.1](https://www.mpich.org/static/downloads/) 版本；如果通信网卡使用IPv6协议，建议安装 [Open MPI-4.1.5](https://www.open-mpi.org/software/ompi/v4.1/) 版本。

且所有机器的MPI软件版本需要保持一致。

**父主题：** HCCL Test常见问题总结

---

# Device网络不通报错retcode 4

## 问题现象

在多机场景下使用HCCL Test工具执行时出现"retcode: 4"错误。错误日志显示：

```
hccl interface return err./opbase_test/hccl_allreduce_rootinfo_test.cc:135,retcode:4
hccl_op_base execute failed,Detailed logs are stored in path:/root/ascend/log/
```

## 原因分析

Device网络不通，导致建链失败。此问题源于设备间的网络连接中断。

## 解决步骤

在Host侧执行网络诊断命令验证连接：

```
hccn_tool -i 0 -ping -g address 192.169.150.60
```

### 网络互通要求

- 同号卡需要两两互通（所有机器的0卡互相ping通、1卡互相ping通等）
- 对于单机16卡配置，需满足：0卡与8卡、1卡与9卡、2卡与11卡等配对互通

### 配置注意事项

修改IP地址后，必须同时更新对应的gateway配置。ip、netmask、gateway需要对应配置，以确保设备间网络可达性。

---

**父主题：** HCCL Test常见问题总结

---

# Host IP之间两两不通报MPI错误

## 问题现象

在多机场景下执行HCCL Test测试工具时，出现错误："Fatal error in PMPI_Barrier: Unknown error class, error stack"

## 原因分析

HCCL Test测试要求所有机器的Host网卡两两能够ping通。此问题通常由某台机器的Host IP与其他机器网络不通引起。

## 解决步骤

1. **定位网络不通的Host机器**

   采用二分方法逐步添加机器进行测试。当添加某台机器时报错，登录该机器并ping其他机器的Host IP。若无法ping通，即确认是该机器的网络问题。

2. **解决该机器与其他机器的Host IP无法ping通的问题**

**相关主题：** HCCL Test常见问题总结

---

# 网口down导致报错ret(-67)

## 问题现象

HCCL Test工具执行报错，在Host日志目录"~/ascend/log"下执行命令 **grep -rn "ERROR"**，有关键报错：ra hdc message process failed ret(-67)。

## 原因分析

可能是某网口状态为down导致其他正常卡与其通信时，建链超时。

## 解决步骤

在Host侧执行如下命令，查询网口状态：

```
for i in {0..7}; do hccn_tool -i $i -link -g ; done
```

若某网卡link状态为down，初始化网卡时会失败，从而导致其他正常卡与其通信时，上报建链超时的错误。

定位到问题网口后，即可联系硬件工程师排查网口问题；若需紧急恢复可尝试重启或重新插拔光模块。

**父主题：** HCCL Test常见问题总结

---

# Device IP冲突导致get socket超时

## 问题现象

网络正常连通且TLS状态一致的场景下，日志存在get socket超时的错误，使用hccn_tool进行卡之间ping操作，正向ping与反向ping存在丢包情况。

## 原因分析

可能是由于两台Device IP地址冲突导致。

## 解决步骤

重新配置Device IP，避免IP地址冲突。

---

**父主题：** HCCL Test常见问题总结

---

