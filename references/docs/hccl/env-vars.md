# HCCL相关环境变量

> 来源：CANN商用版8.0.RC3 hiascend.com

## Ascend Extension for PyTorch

### HCCL_ASYNC_ERROR_HANDLING

#### 功能描述

在PyTorch训练或在线推理中，当采用HCCL通信后端时，该环境变量可控制是否启用异步错误处理机制。

#### 有效值

- 0：禁用异步错误处理
- 1：启用异步错误处理

#### 默认值

PyTorch 1.11.0版本默认为0；PyTorch 2.1.0及更高版本默认为1

#### 重要说明

启用异步处理时，ERROR CQE错误会导致进程终止；其他错误仅提示不停止进程。该环境变量目前处于试用阶段，后续版本可能调整。

#### 配置示例

```bash
export HCCL_ASYNC_ERROR_HANDLING=1
```

#### 使用约束

- 仅适用于以HCCL为通信后端的PyTorch场景
- 启用异步错误处理时，建议设置init_process_group的timeout参数大于HCCL_CONNECT_TIMEOUT和HCCL_EXEC_TIMEOUT的配置时间，以便更清晰地识别HCCL超时原因

#### 支持的产品

- Atlas训练系列产品
- Atlas A2训练系列产品

---

### HCCL_DESYNC_DEBUG

#### 功能描述

该环境变量用于控制PyTorch训练或在线推理中的通信超时分析。启用时，可通过此环境变量控制是否进行通信超时分析。

#### 有效值

- 0：禁用通信超时分析
- 1：启用通信超时分析

#### 默认值

0

#### 重要说明

系统仅输出分析结果，不会中止进程。该变量为试用功能，后续版本可能变更。在大规模集群部署中启用可能导致训练异常延迟。

#### 配置示例

```bash
export HCCL_DESYNC_DEBUG=1
```

#### 使用约束

- PyTorch 1.11.0版本需与HCCL_ASYNC_ERROR_HANDLING同时配置为1
- 仅适用于使用HCCL作为通信后端的PyTorch场景

#### 支持的产品

- Atlas训练系列产品
- Atlas A2训练系列产品

---

### HCCL_EVENT_TIMEOUT

#### 功能描述

在PyTorch训练或在线推理场景中，当采用HCCL通信后端时，该变量用于设置等待Event完成的超时时间。

#### 技术细节

- 需先调用`acl.init`初始化pyACL
- 通过`acl.rt.set_op_wait_timeout`接口设置超时参数
- 后续`acl.rt.stream_wait_event`接口下发的任务将遵守此设置

#### 参数规范

- 单位：秒(s)
- 有效范围：[0, 2147483647]
- 默认值：0

#### 配置示例

```bash
export HCCL_EVENT_TIMEOUT=1800
```

#### 使用约束

仅支持PyTorch网络且使用HCCL通信后端的环境

#### 支持的产品

- Atlas训练系列产品
- Atlas A2训练系列产品

---

### P2P_HCCL_BUFFSIZE

#### 功能描述

此环境变量控制点对点通信（torch.distributed.isend、torch.distributed.irecv和torch.distributed.batch_isend_irecv）是否使用独立通信域功能。

#### 有效值

- 0或未配置：关闭点对点通信独立通信域功能
- >=1：启用该功能，缓存区大小为设置值（单位：MB）

#### 建议配置

推荐配置值为20MB。

#### 内存占用说明

- 申请的内存为HCCL独占，不可与其他业务复用
- 每个通信域额外占用2 x P2P_HCCL_BUFFSIZE的内存（分别用于收发）
- 按通信域粒度管理，每个通信域独占一组缓存

#### 配置示例

```bash
export P2P_HCCL_BUFFSIZE=20
```

#### 使用约束

仅适用于PyTorch网络，且使用HCCL作为通信后端的场景。

#### 支持的产品

- Atlas训练系列产品
- Atlas A2训练系列产品

## 集合通信

### HCCL_IF_IP

#### 功能描述

该环境变量用于配置HCCL初始化时root通信网卡的IP地址。值为字符串格式，需遵循标准IPv4或IPv6格式，仅支持Host网卡，且只能指定一个IP地址。

#### 网卡选择优先级

HCCL按以下顺序选择Host通信网卡：

1. 环境变量HCCL_IF_IP
2. 环境变量HCCL_SOCKET_IFNAME
3. docker/lo以外的网卡（按名字字典序升序）
4. docker网卡
5. lo网卡

#### 配置示例

```bash
export HCCL_IF_IP=10.10.10.1
```

#### 使用约束

无特殊限制

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_IF_BASE_PORT

#### 功能描述

在单算子模式下，通过该环境变量可指定Host网卡起始端口号。系统默认占用以该端口起始的16个端口进行集群信息收集。取值范围为[1024, 65520]的整数，需确保分配的端口未被占用。

#### 配置示例

```bash
export HCCL_IF_BASE_PORT=50000
```

#### 使用约束

分布式场景下，需要通过系统命令预留HCCL使用的端口：

- 未指定变量时：默认使用60000-60015端口
  ```bash
  sysctl -w net.ipv4.ip_local_reserved_ports=60000-60015
  ```

- 指定变量时：若设置为50000，则使用50000-50015端口
  ```bash
  sysctl -w net.ipv4.ip_local_reserved_ports=50000-50015
  ```

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_SOCKET_IFNAME

#### 功能描述

此环境变量用于配置HCCL初始化Host侧root通信网卡名，HCCL可通过该网卡名获取Host IP，完成通信域创建。

#### 支持的配置格式

该变量支持四种主要配置方式：

1. 前缀匹配（如`eth`）：选择所有以指定前缀开头的网卡
2. 前缀排除（如`^eth`）：排除所有以指定前缀开头的网卡
3. 精确指定（如`=eth0`）：选择具体指定的网卡
4. 精确排除（如`^=eth0`）：排除具体指定的网卡

多个网卡或前缀之间用英文逗号分隔。

#### 配置示例

```bash
export HCCL_SOCKET_IFNAME==eth0,endvnic
```

#### 关键特性

- 配置多个网卡时，系统采用最先匹配到的网卡作为root网卡
- 环境变量HCCL_IF_IP的优先级高于HCCL_SOCKET_IFNAME
- 未指定两个变量时，按以下优先级选择：docker/lo以外网卡 > docker网卡 > lo网卡

#### 使用约束

无

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_SOCKET_FAMILY

#### 功能描述

该变量用于指定通信网卡使用的IP协议版本。

#### 有效值

- `AF_INET`：使用IPv4协议
- `AF_INET6`：使用IPv6协议

#### 默认值

IPv4协议

#### 使用场景

##### 场景一：Host侧root通信网卡配置

需与HCCL_SOCKET_IFNAME环境变量配合使用。通过该变量指定HCCL获取Host IP时使用的socket通信协议版本。

##### 场景二：Device侧通信网卡配置

若指定的IP协议与实际网卡信息不匹配，则以实际环境上的网卡信息为准。例如指定IPv6但仅存在IPv4网卡时，实际会使用IPv4网卡。

#### 配置示例

```bash
export HCCL_SOCKET_FAMILY=AF_INET     # IPv4
export HCCL_SOCKET_FAMILY=AF_INET6    # IPv6
```

#### 使用约束

无特殊约束

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_CONNECT_TIMEOUT

#### 功能描述

此环境变量用于在分布式训练或推理场景中，限制不同设备之间socket建链过程的超时等待时间。

在集合通信初始化前，由于其他因素导致不同设备进程执行不同步时，该变量控制设备间的建链超时行为。各设备进程在配置的时间内等待其他设备完成建链同步。

#### 参数配置

| 项目     | 值         |
| -------- | ---------- |
| 数据类型 | 整数       |
| 取值范围 | 120 - 7200 |
| 默认值   | 120        |
| 单位     | 秒(s)      |

#### 配置示例

```bash
export HCCL_CONNECT_TIMEOUT=200
```

#### 使用约束

无特殊限制

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_EXEC_TIMEOUT

#### 功能描述

该环境变量控制分布式训练或推理中设备间执行任务不一致场景（如仅特定进程保存checkpoint）的同步等待时间。

#### 参数配置

##### Atlas训练系列产品

- 单位：秒
- 取值范围：(0, 17340]
- 默认值：1836s
- 注意：实际超时时间 = （环境变量值 / 68）x 68，小于68时按68s处理
- 示例：HCCL_EXEC_TIMEOUT=600 -> 实际值为544s

##### Atlas 300I Duo推理卡

- 单位：秒
- 取值范围：(0, 17340]
- 默认值：1836s
- 计算方式同上述训练系列产品

##### Atlas A2训练系列产品

- 单位：秒
- 取值范围：[0, 2147483647]
- 默认值：1836s
- 配置为0时表示永不超时

#### 配置建议

通常保持默认值即可。仅当默认设置无法满足设备间通信同步需求时，才需增大该参数。

#### 配置示例

```bash
export HCCL_EXEC_TIMEOUT=1800
```

#### 使用约束

无

#### 支持的产品

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

---

### HCCL_INTRA_PCIE_ENABLE

#### 功能描述

用于配置Server内是否使用PCIe环路进行多卡间的通信。

该环境变量可以单独配置，也可以与环境变量HCCL_INTRA_ROCE_ENABLE同时使用。

#### 配置规则

| HCCL_INTRA_PCIE_ENABLE | HCCL_INTRA_ROCE_ENABLE | 结果                 |
| ---------------------- | ---------------------- | -------------------- |
| 1                      | 未配置                 | 采用PCIe环路         |
| 未配置                 | 1                      | 采用RoCE环路         |
| 1                      | 0                      | 采用PCIe环路         |
| 0                      | 1                      | 采用RoCE环路         |
| 0                      | 0                      | 采用PCIe环路         |
| 未配置                 | 未配置                 | 采用PCIe环路（默认） |

注意：不支持两个变量同时配置为1。默认采用PCIe环路通信。

#### 配置示例

```bash
export HCCL_INTRA_PCIE_ENABLE=1
```

#### 使用约束

对于Atlas 200T A2 Box16异构子框（左右两个模组：0~7卡和8~15卡）：

单机场景下，采用PCIe环路时，若使用两个模组的卡，需满足：
- 两个模组使用相同卡数且在同一平面
- 例如需同时使用0卡和8卡、1卡和9卡等

采用RoCE环路时无此限制。

#### 支持的产品

- Atlas 300T Pro训练卡（Atlas训练系列）
- Atlas 200T A2 Box16异构子框（Atlas A2训练系列）

---

### HCCL_INTRA_ROCE_ENABLE

#### 功能描述

该环境变量用于配置服务器内是否启用RoCE环路进行多卡间通信。

可与HCCL_INTRA_PCIE_ENABLE组合使用。

#### 配置规则

| HCCL_INTRA_PCIE_ENABLE | HCCL_INTRA_ROCE_ENABLE | 结果             |
| ---------------------- | ---------------------- | ---------------- |
| 1                      | 未配置                 | PCIe环路         |
| 未配置                 | 1                      | RoCE环路         |
| 1                      | 0                      | PCIe环路         |
| 0                      | 1                      | RoCE环路         |
| 0                      | 0                      | PCIe环路         |
| 未配置                 | 未配置                 | PCIe环路（默认） |

限制：不支持两个变量同时配置为1。

#### 配置示例

```bash
export HCCL_INTRA_ROCE_ENABLE=1
```

#### 使用约束

针对Atlas 200T A2 Box16异构子框（具有左右两个模组，分别为0-7卡和8-15卡）：

单机PCIe场景下，若需要同时使用两个模组的卡，两个模组需使用相同的卡数且在同一平面（如0卡和8卡需同时使用）。RoCE环路无此限制。

#### 支持的产品

- Atlas 300T Pro训练卡（Atlas训练系列）
- Atlas 200T A2 Box16异构子框（Atlas A2训练系列）

---

### HCCL_WHITELIST_DISABLE

#### 功能描述

配置在使用HCCL时是否开启通信白名单。

#### 取值说明

| 取值 | 说明                                                                         |
| ---- | ---------------------------------------------------------------------------- |
| 0    | 开启白名单，校验HCCL通信白名单，只有在通信白名单中的IP地址才允许进行集合通信 |
| 1    | 关闭白名单，无需校验HCCL通信白名单                                           |

#### 缺省值

1（默认关闭白名单）

#### 相关配置

如果开启了白名单校验，需要通过`HCCL_WHITELIST_FILE`指定白名单配置文件路径。

#### 配置示例

```bash
export HCCL_WHITELIST_DISABLE=1
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_WHITELIST_FILE

#### 功能描述

当通过HCCL_WHITELIST_DISABLE开启了通信白名单校验功能时，需要通过此环境变量配置指向HCCL通信白名单配置文件的路径，只有在通信白名单中的IP地址才允许进行集合通信。

HCCL通信白名单配置文件格式为：

```json
{ "host_ip": ["ip1", "ip2"], "device_ip": ["ip1", "ip2"] }
```

##### 配置参数说明

- device_ip：预留字段，当前版本暂不支持
- IP格式：点分十进制

##### 重要提示

白名单IP需要指定为集群通信使用的有效IP。

#### 配置示例

```bash
export HCCL_WHITELIST_FILE=/home/test/whitelist
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_RDMA_TC

#### 功能描述

用于配置RDMA网卡的traffic class。该环境变量的值对应RoCE报文的DSCP。

由于IP报文头中DSCP在DS域的高6bit中bit0~1固定为零，因此该值应配置为DSCP值乘以4。

#### 默认值

132（对应DSCP为33，计算方式：132=33×4）

#### 配置示例

当DSCP为25时：

```bash
export HCCL_RDMA_TC=100
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_RDMA_SL

#### 功能描述

用于配置RDMA网卡的service level，该值需要和网卡配置的PFC优先级保持一致，若配置不一致可能导致性能劣化。

#### 配置规范

- 数据类型：整数
- 取值范围：[0, 7]
- 默认值：4

#### 配置示例

```bash
export HCCL_RDMA_SL=3
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_RDMA_TIMEOUT

#### 功能描述

用于配置RDMA网卡重传超时时间的系数timeout。

RDMA网卡重传超时时间最小值的计算公式为：`4.096 μs * 2 ^ timeout`，其中timeout为该环境变量配置值，且实际重传超时时间与用户网络状况有关。

#### 配置范围

| 产品型号              | 取值范围 | 默认值 |
| --------------------- | -------- | ------ |
| Atlas训练系列产品     | [5, 24]  | 20     |
| Atlas 300I Duo推理卡  | [5, 24]  | 20     |
| Atlas A2 训练系列产品 | [5, 20]  | 20     |

#### 配置示例

```bash
export HCCL_RDMA_TIMEOUT=6
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_RDMA_RETRY_CNT

#### 功能描述

配置RDMA网卡的重传次数，需要配置为整数。

#### 有效范围

- 取值范围：[1, 7]
- 默认值：7

#### 配置示例

```bash
export HCCL_RDMA_RETRY_CNT=5
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_BUFFSIZE

#### 功能描述

此环境变量用于控制两个NPU之间共享数据的缓存区大小。

- 单位：M（整数）
- 默认值：200
- 取值范围：>=1

#### 工作原理

集合通信网络中，每一个HCCL通信域都会占用HCCL_BUFFSIZE大小的缓存区。每个通信域实际占用2*HCCL_BUFFSIZE大小的内存，分别用于收发。

#### 使用场景

- 动态shape网络场景
- 开发人员调用集合通信库HCCL的C语言接口进行框架对接

#### 大语言模型配置建议

推荐值计算公式：

(MircobatchSize × SequenceLength × hiddenSize × sizeOf(DataType)) / (1024×1024)，向上取整

#### 重要注意事项

- 该环境变量申请的内存为HCCL独占，不可与其他业务内存复用
- 每个通信域独占一组内存，保证多通信域并发算子互不影响
- 当数据量超过HCCL_BUFFSIZE的取值时，可能会出现性能下降

#### 配置示例

```bash
export HCCL_BUFFSIZE=200
```

#### 使用约束

无

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_ALGO

#### 功能描述

此环境变量用于配置集合通信Server间跨机通信算法，支持全局配置与按算子配置两种方式。

HCCL提供自适应算法选择功能，默认根据产品形态、数据量和Server个数选择合适的算法。若通过此环境变量指定了算法，则自适应功能不再生效。

#### 配置方式

##### 全局配置

```bash
export HCCL_ALGO="level0:NA;level1:<algo>"
```

- level0：Server内通信算法，仅支持NA
- level1：Server间通信算法，支持以下取值：
  - ring：环结构算法，通信步数多，时延较高，适合Server个数少、数据量小、网络拥塞场景
  - H-D_R：递归二分和倍增算法，通信步数少，适合2的整数次幂节点规模
  - NHR：非均衡层次环算法，通信步数少，适合Server个数较多场景
  - NHR_V1：历史版本NHR算法，性能低于新版，逐步下线
  - NB：非均匀数据块通信算法，通信步数少，适合Server个数较多
  - pipeline：流水线并行算法，适合数据量大且每机多卡场景
  - pairwise：逐对通信算法，仅用于AlltoAll、AlltoAllV、AlltoAllVC，适合规避网络一打多

##### 按算子配置

```bash
export HCCL_ALGO="<op0>=level0:NA;level1:<algo0>/<op1>=level0:NA;level1:<algo1>"
```

支持的算子类型：allgather、reducescatter、allreduce、broadcast、reduce、scatter、alltoall

#### 默认行为

Atlas训练系列产品：
- Server个数为非2的整数次幂时，默认使用ring算法
- 其他场景默认使用H-D_R算法

Atlas A2训练系列产品：根据产品形态、节点数及数据量自动选择

#### 配置示例

```bash
# 全局配置
export HCCL_ALGO="level0:NA;level1:H-D_R"

# 按算子配置
export HCCL_ALGO="alltoall=level0:NA;level1:pairwise"
```

#### 使用约束

Server内通信算法仅支持配置为"NA"。

#### 支持的型号

- Atlas训练系列产品
- Atlas A2 训练系列产品

---

### HCCL_DIAGNOSE_ENABLE

#### 功能描述

此环境变量用于配置集合通信是否缓存部分任务的详细信息，以便任务执行失败时，打印详细日志，用于问题定位。

#### 取值说明

| 取值 | 含义               |
| ---- | ------------------ |
| 1    | 开启集合通信缓存   |
| 0    | 不开启集合通信缓存 |

#### 默认值

0

#### 性能影响

此环境变量开启后会对性能产生一定的影响。

#### 配置示例

```bash
export HCCL_DIAGNOSE_ENABLE=1
```

#### 使用约束

最多保存最新的2000个算子信息。

#### 支持的型号

- Atlas A2 训练系列产品

---

### HCCL_ENTRY_LOG_ENABLE

#### 功能描述

此环境变量用于控制是否实时打印通信算子的调用行为日志。

#### 取值说明

- 1：实时打印通信算子的调用行为日志，每调用一次通信算子打印一条运行日志
- 0：不打印通信算子的调用行为运行日志。程序运行出错或进程结束时可在trace日志查看通信算子的详细调用信息

#### 默认值

0

#### 配置示例

```bash
export HCCL_ENTRY_LOG_ENABLE=1
```

#### 使用约束

仅用于集合通信算子的单算子调用场景。

#### 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_OP_EXPANSION_MODE

#### 功能描述

该环境变量用于配置通信算法的编排展开位置。

#### 支持的取值

- AI_CPU：通信算法的编排展开位置在Device侧的AI CPU计算单元
- AIV：通信算法的编排展开位置在Device侧的AI Vector Core计算单元
- HOST：通信算法的编排展开位置为Host侧CPU

#### 产品支持情况

| 产品型号                                                                              | 支持的配置 | 默认值 | 约束说明                                                                                          |
| ------------------------------------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------------------- |
| Atlas 300I Duo推理卡                                                                  | AI_CPU     | HOST   | 仅支持单机单通信域；仅支持AllReduce算子；不支持profiling性能数据采集；对于静态shape图不支持此配置 |
| Atlas 800T A2 训练服务器 / Atlas 900 A2 PoD集群基础单元 / Atlas 200T A2 Box16异构子框 | AIV        | HOST   | 仅支持推理特性；支持AllReduce、AlltoAll、AlltoAllV算子；数据类型和操作类型有限制；不支持跨框通信  |

#### 配置示例

```bash
export HCCL_OP_EXPANSION_MODE="HOST"
```

#### 使用约束

- 若HCCL_DETERMINISTIC环境变量配置为"true"，此配置项不再生效
- 配置为"AIV"时，强制结束进程可能在Device侧日志中出现访问非法地址错误，但不影响Device状态
- 配置为"AIV"时，图模式执行不支持开启单流模式

#### 支持的型号

- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

---

### HCCL_DETERMINISTIC

#### 功能描述

此环境变量用于配置是否开启归约类通信算子的确定性计算。归约类通信算子包括AllReduce、ReduceScatter、Reduce。开启确定性计算后，算子在相同的硬件和输入下，多次执行将产生相同的输出。

#### 支持的取值

- true：开启归约类通信算子的确定性计算
- false：不开启确定性计算

#### 默认值

false

#### 使用说明

一般情况下无需开启确定性计算。当模型多次执行结果不同或进行精度调优时，可通过此环境变量开启确定性计算进行辅助调试调优。

注意：开启确定性计算后，算子执行时间会变慢，导致性能下降。

#### 配置示例

```bash
export HCCL_DETERMINISTIC=true
```

#### 使用约束

无

#### 支持的型号

- Atlas A2 训练系列产品

---

### HCCL_RDMA_PCIE_DIRECT_POST_NOSTRICT

#### 功能描述

在以下场景中，当通信算子下发性能受Host限制时，可通过此环境变量设置PCIe Direct方式提交RDMA任务来改善性能：

- 多机通信场景
- Host操作系统小页内存页表大小不是4KB

#### 取值说明

| 取值  | 说明                                               |
| ----- | -------------------------------------------------- |
| TRUE  | 通过PCIe Direct方式提交RDMA任务                    |
| FALSE | 通过HDC(Host Device Communication)方式提交RDMA任务 |

#### 默认行为

若未配置此环境变量：
- Host侧小页内存页表大小为4KB时：采用RDMA Direct方式
- Host侧小页内存页表大小不是4KB时：采用HDC方式

#### 配置示例

```bash
export HCCL_RDMA_PCIE_DIRECT_POST_NOSTRICT=TRUE
```

#### 注意事项

- 设置为TRUE时会额外消耗Device侧大页内存（每条通信链路额外占用约1MB）
- 若需平衡性能提升与内存节省，可通过HCCL_ALGO环境变量将server间通信算法设置为ring模式以减少通信链接数量

#### 使用约束

需满足功能描述中的条件：多机通信场景且Host小页内存页表大小非4KB。

#### 支持的型号

- Atlas A2 训练系列产品

---

### HCCL_RDMA_QPS_PER_CONNECTION

#### 功能描述

此环境变量控制两个rank之间RDMA通信时使用的Queue Pair(QP)个数。默认情况下创建1个QP，通过配置此变量可实现多QP并行传输。

业务数据会平均分配到配置的QP数量上进行并行收发。

#### 配置参数

- 数据类型：整数
- 取值范围：[1, 8]
- 默认值：1

#### 配置示例

```bash
export HCCL_RDMA_QPS_PER_CONNECTION=4
```

#### 相关配置

开启多QP传输功能后，可通过环境变量`HCCL_MULTI_QP_THRESHOLD`设置每个QP分担数据量的最小阈值。

#### 使用约束

- 仅支持Atlas A2训练系列产品的单算子调用方式
- 不支持静态图模式
- 取值超过8时，可能因内存占用过多导致业务运行失败

#### 支持的型号

- Atlas A2 训练系列产品

---

### HCCL_MULTI_QP_THRESHOLD

#### 功能描述

在两个rank之间使用多QP通信的场景下（即HCCL_RDMA_QPS_PER_CONNECTION取值大于1），此环境变量可设置每个QP分担数据量的最小阈值。

#### 配置参数

- 数据类型：整数
- 取值范围：[1, 8192]
- 默认值：512
- 单位：KB

#### 工作机制

当（rank间单次通信数据量 / HCCL_RDMA_QPS_PER_CONNECTION取值） < HCCL_MULTI_QP_THRESHOLD取值 时，HCCL会自动减少QP个数，确保每个QP分担的数据量>=该阈值。

示例：通信数据量1MB、HCCL_RDMA_QPS_PER_CONNECTION为4、阈值为512KB，则HCCL会减少QP个数至2个。

- rank间数据量小于阈值时使用单QP传输
- 每个QP分担数据量大于512KB时，多QP场景性能劣化小于3%

#### 配置示例

```bash
export HCCL_MULTI_QP_THRESHOLD=512
```

#### 使用约束

需要配合HCCL_RDMA_QPS_PER_CONNECTION环境变量使用。

#### 支持的型号

- Atlas A2 训练系列产品
