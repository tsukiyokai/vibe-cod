# HCCL API(Python)

*   **[接口列表](hcclpython_07_0001.html)**
    TF Adapter提供的分布式优化器NPUDistributedOptimizer和npu_distributed_optimizer_wrapper可以让用户在不需要感知allreduce的情况下自动完成梯度聚合功能，实现数据并行训练方式。但为了能够同时满足用户灵活的使用方式，集合通信库HCCL提供了常用的rank管理、梯度切分功能、集合通信原型等接口。

*   **[hccl.manage.api](hcclpython_07_0002.html)**

*   **[hccl.split.api](hcclpython_07_0014.html)**

*   **[npu_bridge.hccl.hccl_ops](hcclpython_07_0017.html)**

# 接口列表

## 概述

TF Adapter提供的分布式优化器NPUDistributedOptimizer和npu_distributed_optimizer_wrapper可以让用户在不需要感知allreduce的情况下自动完成梯度聚合功能，实现数据并行训练方式。但为了能够同时满足用户灵活的使用方式，集合通信库HCCL提供了常用的rank管理、梯度切分功能、集合通信原型等接口。

## HCCL(Python)接口列表

| 分类     | 接口                           | 简介                                                                                                                                                                                       | 定义文件                                                         |
| -------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| rank管理 | create_group                   | 创建集合通信group                                                                                                                                                                          | ${install_path}/python/site-packages/hccl/manage/api.py          |
|          | destroy_group                  | 销毁集合通信group                                                                                                                                                                          |                                                                  |
|          | get_rank_size                  | 获取group内rank数量（即Device数量）                                                                                                                                                        |                                                                  |
|          | get_local_rank_size            | 获取group内device所在服务器内的local rank数量                                                                                                                                              |                                                                  |
|          | get_rank_id                    | 获取device在group中对应的rank序号                                                                                                                                                          |                                                                  |
|          | get_local_rank_id              | 获取device在group中对应的local rank序号                                                                                                                                                    |                                                                  |
|          | get_world_rank_from_group_rank | 从group rank id，获取该进程对应的world rank id                                                                                                                                             |                                                                  |
|          | get_group_rank_from_world_rank | 从world rank id，获取该进程在group中的group rank id                                                                                                                                        |                                                                  |
| 梯度切分 | set_split_strategy_by_idx      | 基于梯度的索引id，在集合通信group内设置反向梯度切分策略                                                                                                                                    | ${install_path}/python/site-packages/hccl/split/api.py           |
|          | set_split_strategy_by_size     | 基于梯度数据量百分比，在集合通信group内设置反向梯度切分策略                                                                                                                                |                                                                  |
| 集合通信 | allreduce                      | 提供group内的集合通信allreduce功能，对所有节点的同名张量进行约减                                                                                                                           | ${install_path}/python/site-packages/npu_bridge/hccl/hccl_ops.py |
|          | allgather                      | 提供group内的集合通信allgather功能，将所有节点的输入Tensor合并起来                                                                                                                         |                                                                  |
|          | broadcast                      | 提供group内的集合通信broadcast功能，将root节点的数据广播到其他rank                                                                                                                         |                                                                  |
|          | reduce_scatter                 | 提供group内的集合通信reducescatter功能                                                                                                                                                     |                                                                  |
|          | send                           | 提供group内点对点通信发送数据的send功能                                                                                                                                                    |                                                                  |
|          | receive                        | 提供group内点对点通信发送数据的receive功能                                                                                                                                                 |                                                                  |
|          | reduce                         | 提供group内的集合通信reduce功能，对所有节点的同名张量进行规约，并将数据输出至root节点                                                                                                      |                                                                  |
|          | alltoallv                      | 集合通信域alltoallv操作接口。向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据                                                                                            |                                                                  |
|          | alltoallvc                     | 集合通信域alltoallvc操作接口。向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。alltoallvc通过输入参数send_count_matrix传入所有rank的收发参数，与alltoallv相比，性能更优 |                                                                  |

# create_group

## 功能说明

创建集合通信用户自定义group。

如果开发者不调用此接口创建用户自定义group，则默认将所有参与集群训练的设备创建为全局的hccl_world_group。

## 函数原型

```python
def create_group(group, rank_num, rank_ids)
```

## 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                                                              |
| -------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| group    | 输入      | String类型，最大长度为128字节，含结束符。group名称，集合通信group的标识，不能为hccl_world_group。hccl_world_group是默认group，由ranktable文件创建，不能通过该接口创建，如果用户传入的group如果是hccl_world_group，创建group失败。 |
| rank_num | 输入      | int类型。组成该group的rank数量。最大值为32768。                                                                                                                                                                                   |
| rank_ids | 输入      | list类型。组成该group的world_rank_id列表。在不同单板类型上，有不同的限制。                                                                                                                                                        |

## rank_ids限制条件

### Atlas训练系列产品

**Server单机场景：**
- rank数量必须为1/2/4/8
- 0-3卡与4-7卡各为一个组网
- rank数量为2/4时要求选取的昇腾AI处理器同属一个cluster

**Server集群场景：**
- 各Server要选取相同数量的rank（且数量要求为1/2/4/8）
- 各Server选取rank数量为2/4时要求选取的昇腾AI处理器同属一个cluster（即rank id按8取模余数都小于4或都大于等于4）

示例（三台Server）：
```
Server 1: {0,1,2,3,4,5,6,7}
Server 2: {8,9,10,11,12,13,14,15}
Server 3: {16,17,18,19,20,21,22,23}

满足要求的rank_ids列表：
rank_ids=[1,9,17]
rank_ids=[1,2,9,10,17,18]
rank_ids=[4,5,6,7,12,13,14,15,20,21,22,23]
```

### Atlas 300I Duo推理卡

**Server单机场景：**
- rank_ids无限制条件

**Server集群场景：**
- 建议各Server要选取相同数量的rank（数量大小无要求）
- 各Server选取的rank对应位置要相等（即rank id按8取模相等）
- 若各Server选取的rank数量不同，会造成性能裂化

### Atlas A2训练系列产品

**Server单机场景：**
- rank_ids无限制条件

**Server集群场景：**
- 建议各Server要选取相同数量的rank（数量大小无要求）
- 各Server选取的rank对应位置要相等（即rank id按8取模相等）
- 若各Server选取的rank数量不同，会造成性能裂化

## 补充说明

建议rank_ids按照Device物理连接顺序进行排序，即将物理连接上较近的device编排在一起。例如，若device_ip按照物理连接从小到大设置，则rank_ids也建议按照从小到大的顺序设置。

## 返回值

无。

## 约束说明

- 必须在集合通信初始化完成之后调用
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败
- 重复创建名称相同的group，会创建失败

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
```

# destroy_group

## 功能说明

销毁用户自定义group。

## 函数原型

```python
def destroy_group(group)
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                                      |
| ------ | --------- | ------------------------------------------------------------------------- |
| group  | 输入      | String类型，最大长度为128字节，含结束符。group名称，集合通信group的标识。 |

## 返回值

无。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 相同名称的group，destroy_group和create_group配套使用，必须在create_group完成之后调用。
- 如果用户传入的group恰好是hccl_world_group（默认group），销毁group失败。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
destroy_group("myGroup")
```

# get_rank_size

## 功能说明

获取group内的rank数量（即Device数量）。

## 函数原型

```python
def get_rank_size(group="hccl_world_group")
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------- |
| group  | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group，如果用户不传，默认为"hccl_world_group"。 |

## 返回值

int类型，返回group内rank数量。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API获取本group的rank数量。
- 如果传入"hccl_world_group"，返回world_group的rank数量。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
rankSize = get_rank_size("myGroup")
#rankSize = 4
```

# get_local_rank_size

## 功能说明

获取group内device所在服务器内的local rank数量。

## 函数原型

```python
def get_local_rank_size(group="hccl_world_group")
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------- |
| group  | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group，如果用户不传，默认为"hccl_world_group"。 |

## 返回值

int类型，返回device所在服务器内的local rank数量。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API获取本group的local rank数量。
- 如果传入"hccl_world_group"，返回world_group的local rank数量。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
localRankSize = get_local_rank_size("myGroup")
#localRankSize = 1
```

# get_rank_id

## 功能说明

获取device在group中对应的rank序号。

## 函数原型

```python
def get_rank_id(group="hccl_world_group")
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------- |
| group  | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group，如果用户不传，默认为"hccl_world_group"。 |

## 返回值

int类型，返回device所在group的rank id。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API获取进程在group中的rank id。
- 如果传入"hccl_world_group"，返回进程在world_group的rank id。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
rankId = get_rank_id("myGroup")
#rankId = 0/1/2/3
```

# get_local_rank_id

## 功能说明

获取device在group中对应的local rank序号。

## 函数原型

```python
def get_local_rank_id(group="hccl_world_group")
```

## 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------- |
| group  | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group，如果用户不传，默认为"hccl_world_group"。 |

## 返回值

int类型，返回device所在服务器内的local rank id序号。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API获取进程在group中的local rank id。
- 如果传入"hccl_world_group"，返回进程在world_group的local rank id。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
localRankId = get_local_rank_id("myGroup")
#rankId = 0
```

# get_world_rank_from_group_rank

## 功能说明

从group rank id，获取该进程对应的world rank id。

## 函数原型

```python
def get_world_rank_from_group_rank(group, group_rank_id)
```

## 参数说明

| 参数名        | 输入/输出 | 描述                                                                                              |
| ------------- | --------- | ------------------------------------------------------------------------------------------------- |
| group         | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。 |
| group_rank_id | 输入      | int类型。进程在group中的rank id。                                                                 |

## 返回值

int类型，进程在"hccl_world_group"中的rank id。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API转换group rank id到world rank id。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
worldRankId = get_world_rank_from_group_rank("myGroup", 1)
#worldRankId = 8
```

# get_group_rank_from_world_rank

## 功能说明

从world rank id，获取该进程在group中的group rank id。

## 函数原型

```python
def get_group_rank_from_world_rank(world_rank_id, group)
```

## 参数说明

| 参数名        | 输入/输出 | 描述                                                                                              |
| ------------- | --------- | ------------------------------------------------------------------------------------------------- |
| world_rank_id | 输入      | int类型。进程在"hccl_world_group"中的rank id。                                                    |
| group         | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。 |

## 返回值

int类型，正常返回进程在group中的rank id。

## 约束说明

- 必须在集合通信初始化完成之后调用。
- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- create_group完成之后，调用此API转换world rank id到group rank id。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
create_group("myGroup", 4, [0, 1, 2, 3])
groupRankId = get_group_rank_from_world_rank(8, "myGroup")
#groupRankId = 1
```

# set_split_strategy_by_idx

## 功能说明

基于梯度的索引id，在集合通信group内设置反向梯度切分策略，实现allreduce的融合，用于进行集合通信的性能调优。

## 函数原型

```python
def set_split_strategy_by_idx(idxList, group="hccl_world_group")
```

## 参数说明

| 参数名  | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| idxList | 输入      | list类型。梯度的索引id列表。梯度的索引id列表需为非负，升序序列。梯度的索引id必须基于模型的总梯度参数个数去设置。索引id从0开始，最大值可通过以下方法获得：不调用梯度切分接口设置梯度切分策略进行训练，此时脚本会使用set_split_strategy_by_size中的默认梯度切分方式进行训练。训练结束后，在INFO级别的host训练日志中搜索"segment result"关键字，可以得到梯度切分的分段情况如：segment index list: [0,107] [108,159]。此分段序列中最大的数字（例如159）即总梯度参数索引最大的值。梯度的切分最多支持8段。比如模型总共有160个参数会产生梯度，需要切分[0,20]、[21,100]和[101,159]三段，则可以设置为idxList=[20,100,159]。 |
| group   | 输入      | String类型。group名称，可以为"hccl_world_group"或自定义group，默认为"hccl_world_group"。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

## 返回值

无。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 若用户不调用梯度切分接口设置切分策略，则会按默认反向梯度切分策略切分。

默认切分策略：按梯度数据量切分为2段，第一段数据量为96.54%，第二段数据量为3.46%（部分情况可能出现为一段情况）。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
set_split_strategy_by_idx([20, 100, 159], "group")
```

# set_split_strategy_by_size

## 功能说明

基于梯度数据量百分比，在集合通信group内设置反向梯度切分策略，实现allreduce的融合，用于进行集合通信的性能调优。

## 函数原型

```python
def set_split_strategy_by_size(dataSizeList, group="hccl_world_group")
```

## 参数说明

| 参数名       | 输入/输出 | 描述                                                                                                                                                                                                                     |
| ------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dataSizeList | 输入      | list类型。梯度参数数据量百分比列表。梯度的索引id列表需为非负，且梯度数据量序列总百分比之和必须为100。梯度的切分最多支持8段。比如模型总共有150M梯度数据量，需要切分90M，30M，30M三段，则可以设置dataSizeList=[60,20,20]。 |
| group        | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为"hccl_world_group"或自定义group，默认为"hccl_world_group"。                                                                                                    |

## 返回值

无。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 在同时基于梯度数据量百分比及梯度的索引id设置反向梯度切分策略时，以基于梯度数据量百分比设置结果优先。
- 若用户不调用梯度切分接口设置切分策略，则会按默认反向梯度切分策略切分。

默认切分策略：ResNet50的最优切分位置，即按梯度数据量切分为2段，第一段数据量为96.54%，第二段数据量为3.46%。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
set_split_strategy_by_size([60, 20, 20], "group")
```

# allreduce

## 功能说明

集合通信算子AllReduce的操作接口，将group内所有节点的输入数据进行reduce操作后，再把结果发送到所有节点的输出buf，其中reduce操作类型由reduction参数指定。

## 函数原型

```python
def allreduce(tensor, reduction, fusion=1, fusion_id=-1, group="hccl_world_group")
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                  |
| --------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tensor    | 输入      | TensorFlow的tensor类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、int32、int64、float16、float32。针对Atlas 300I Duo推理卡，支持的数据类型为：int8、int16、int32、float16、float32。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、int16、int32、int64、float16、float32、bfp16。 |
| reduction | 输入      | String类型。reduce的op类型，可以为max、min、prod和sum。针对Atlas 300I Duo推理卡，当前版本"prod"、"max"、"min"操作不支持int16数据类型。针对Atlas A2训练系列产品，当前版本"prod"操作不支持int16、bfp16数据类型。                                                                                        |
| fusion    | 输入      | int类型。allreduce算子融合标识。0：不融合，该allreduce算子不和其他allreduce算子融合。1：按照梯度切分策略进行融合，默认为1。2：按照相同fusion_id进行融合。                                                                                                                                             |
| fusion_id | 输入      | allreduce算子的融合id。对相同fusion_id的allreduce算子进行融合。                                                                                                                                                                                                                                       |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                     |

## 返回值

tensor：对输入tensor执行完allreduce操作之后的结果tensor。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 每个rank只能有一个输入。
- allreduce上游节点暂不支持variable算子。
- 该接口要求输入tensor的数据量不超过8GB。
- allreduce算子融合只支持reduction为sum类型的算子。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
tensor = tf.random_uniform((1, 3), minval=1, maxval=10, dtype=tf.float32)
result = hccl_ops.allreduce(tensor, "sum")
```

# allgather

## 功能说明

集合通信算子AllGather的操作接口，将通信域内所有节点的输入按照rank id重新排序，然后拼接起来，再将结果发送到所有节点的输出。

针对AllGather操作，每个节点都接收按照rank id重新排序后的数据集合，即每个节点的AllGather输出都是一样的。

## 函数原型

```python
def allgather(tensor, rank_size, group="hccl_world_group", fusion=0, fusion_id=-1)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                          |
| --------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tensor    | 输入      | TensorFlow的tensor类型。针对Atlas训练系列产品，支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas 300I Duo推理卡，支持数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| rank_size | 输入      | int类型。group内device的数量。最大值为32768。                                                                                                                                                                                                                                                                                                                                                                                 |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                                                                                                                                             |
| fusion    | 输入      | int类型。allgather算子融合标识，支持以下取值：0：标识网络编译时不会对该算子进行融合，即该allgather算子不和其他allgather算子融合。2：网络编译时，会对allgather算子按照相同的fusion_id进行融合，即"fusion_id"相同的allgather算子之间会进行融合。默认值为"0"。                                                                                                                                                                   |
| fusion_id | 输入      | allgather算子的融合id。对相同fusion_id的allgather算子进行融合。                                                                                                                                                                                                                                                                                                                                                               |

## 返回值

tensor：对输入tensor执行完allgather操作之后的结果tensor。

## 约束说明

调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
rank_size = 2
result = hccl_ops.allgather(tensor, rank_size)
```

# broadcast

## 功能说明

集合通信算子Broadcast的操作接口，将通信域内root节点的数据广播到其他rank。

## 函数原型

```python
def broadcast(tensor, root_rank, fusion=2, fusion_id=0, group="hccl_world_group")
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tensor    | 输入      | TensorFlow的tensor类型，为list。针对Atlas训练系列产品，支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas 300I Duo推理卡，支持数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| root_rank | 输入      | int类型。作为root节点的rank_id，该id是group内的rank id。                                                                                                                                                                                                                                                                                                                                                                              |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                                                                                                                                                     |
| fusion    | 输入      | int类型。broadcast算子融合标识。0：不融合，该broadcast算子不和其他broadcast算子融合。2：按照相同fusion_id进行融合。默认为2。其他值非法。                                                                                                                                                                                                                                                                                              |
| fusion_id | 输入      | broadcast算子的融合id。对相同fusion_id的broadcast算子进行融合。                                                                                                                                                                                                                                                                                                                                                                       |

## 返回值

tensor：对输入tensor执行完broadcast操作之后的结果tensor。

## 约束说明

调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
root = 0
inputs = [tensor]
result = hccl_ops.broadcast(inputs, root)
```

# reduce_scatter

## 功能说明

集合通信算子ReduceScatter的操作接口。将所有rank的输入相加（或其他操作）后，再把结果按照rank编号均匀分散的到各个rank的输出buffer，每个进程拿到其他进程1/ranksize份的数据进行归约操作。

## 函数原型

```python
def reduce_scatter(tensor, reduction, rank_size, group="hccl_world_group", fusion=0, fusion_id=-1)
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                               |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| tensor    | 输入      | TensorFlow的tensor类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、int32、int64、float16、float32。针对Atlas 300I Duo推理卡，支持的数据类型为：int8、int16、int32、float16、float32。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、int16、int32、int64、float16、float32、bfp16。需要注意tensor的第一个维度的元素个数必需是rank size的整数倍。 |
| reduction | 输入      | String类型。reduce的op类型，可以为max、min、prod和sum。说明：针对Atlas 300I Duo推理卡，当前版本"prod"、"max"、"min"操作不支持int16数据类型。针对Atlas A2训练系列产品，当前版本"prod"操作不支持int16、bfp16数据类型。                                                                                                                                               |
| rank_size | 输入      | int类型。group内device的数量。最大值：32768。                                                                                                                                                                                                                                                                                                                      |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                                                                                  |
| fusion    | 输入      | int类型。reducescatter算子融合标识，支持以下取值：0：标识网络编译时不会对该算子进行融合，即该reducescatter算子不和其他reducescatter算子融合。2：网络编译时，会对reducescatter算子按照相同的fusion_id进行融合，即"fusion_id"相同的reducescatter算子之间会进行融合。默认值为"0"。                                                                                    |
| fusion_id | 输入      | reducescatter算子的融合id。对相同fusion_id的reducescatter算子进行融合。                                                                                                                                                                                                                                                                                            |

## 返回值

tensor：对输入tensor执行完reducescatter操作之后的结果tensor。建议返回tensor的数据量为32Byte对齐，否则会导致性能下降。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 该接口要求输入tensor的数据量不超过8GB。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
rank_size = 2
result = hccl_ops.reduce_scatter(tensor, "sum", rank_size)
```

# reduce

## 功能说明

集合通信算子Reduce的操作接口，将所有rank的input相加（或其他操作）后，再把结果发送到root节点的输出buffer。

## 函数原型

```python
def reduce(tensor, reduction, root_rank, fusion=0, fusion_id=-1, group="hccl_world_group")
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tensor    | 输入      | TensorFlow的tensor类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、int32、int64、float16、float32。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、int16、int32、int64、float16、float32、bfp16。 |
| reduction | 输入      | String类型。reduce的op类型，可以为max、min、prod和sum。说明：针对Atlas A2训练系列产品，当前版本"prod"操作不支持int16、bfp16数据类型。                                                                               |
| root_rank | 输入      | int类型。作为root节点的rank_id，该id是group内的rank id。                                                                                                                                                            |
| fusion    | 输入      | int类型。reduce算子融合标识。0：不融合，该reduce算子不和其他reduce算子融合。2：按照相同fusion_id进行融合。                                                                                                          |
| fusion_id | 输入      | reduce算子的融合id。对相同fusion_id的reduce算子进行融合。                                                                                                                                                           |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                   |

## 返回值

tensor：对输入tensor执行完reduce操作之后的结果tensor。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 该接口要求输入tensor的数据量不超过8GB。
- reduce算子融合只支持reduction为sum类型的算子。

## 支持的型号

- Atlas训练系列产品
- Atlas A2训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
result = hccl_ops.reduce(tensor, "sum", 0)
```

# send

## 功能说明

提供group内点对点通信发送数据的send功能。

## 函数原型

```python
def send(tensor, sr_tag, dest_rank, group="hccl_world_group")
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tensor    | 输入      | TensorFlow的tensor类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas 300I Duo推理卡，支持数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| sr_tag    | 输入      | int类型。消息标签，相同sr_tag的send/recv对可以收发数据。                                                                                                                                                                                                                                                                                                                                                                                  |
| dest_rank | 输入      | int类型。数据的目标节点，该rank是group中的rank id。                                                                                                                                                                                                                                                                                                                                                                                       |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                                                                                                                                                         |

## 返回值

无。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 如果点对点收发数据的rank属于不同的AI Server，要保证昇腾AI处理器的物理id相同。
- 要求send和receive必须配对使用，且和图中其他算子要有依赖关系。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
sr_tag = 0
dest_rank = 1
hccl_ops.send(tensor, sr_tag, dest_rank)
```

# receive

## 功能说明

提供group内点对点通信发送数据的receive功能。

## 函数原型

```python
def receive(shape, data_type, sr_tag, src_rank, group="hccl_world_group")
```

## 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| shape     | 输入      | 接收tensor的shape。                                                                                                                                                                                                                                                                                                                                                                                                                   |
| data_type | 输入      | 接收数据的数据类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas 300I Duo推理卡，支持数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。 |
| sr_tag    | 输入      | int类型。消息标签，相同sr_tag的send/recv对可以收发数据。                                                                                                                                                                                                                                                                                                                                                                              |
| src_rank  | 输入      | int类型。接收数据的源节点，该rank是group中的rank id。                                                                                                                                                                                                                                                                                                                                                                                 |
| group     | 输入      | String类型，最大长度为128字节，含结束符。group名称，可以为用户自定义group或者"hccl_world_group"。                                                                                                                                                                                                                                                                                                                                     |

## 返回值

tensor：进行receive操作之后接收到对端的结果tensor。

## 约束说明

- 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。
- 点对点收发数据的rank要属于不同的Server，且昇腾AI处理器物理ID必须相同。
- 要求send和receive必须配对使用，且和图中其他算子要有依赖关系。

## 支持的型号

- Atlas训练系列产品
- Atlas 300I Duo推理卡
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
sr_tag = 0
src_rank = 0
tensor = hccl_ops.receive(tensor.shape, tensor.dtype, sr_tag, src_rank)
```

# alltoallv

## 功能说明

集合通信域alltoallv操作接口。向通信域内所有rank发送数据（数据量可以定制），并从所有rank接收数据。

两个NPU之间共享数据的缓存区大小默认为200M，可通过环境变量HCCL_BUFFIZE进行调整，单位为M，取值需要大于等于1，默认值是200M。集合通信网络中，每一个集合通信域都会占用HCCL_BUFFSIZE大小的缓冲区，用户可以根据通信数据量与业务模型数据量的大小，适当调整HCCL_BUFFSIZE的大小，提升网络执行性能，例如：

```
export HCCL_BUFFSIZE=2048
```

## 函数原型

```python
def all_to_all_v(send_data, send_counts, send_displacements, recv_counts, recv_displacements, group="hccl_world_group")
```

## 参数说明

| 参数名             | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| send_data          | 输入      | 待发送的数据。TensorFlow的tensor类型。针对Atlas训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64。针对Atlas A2训练系列产品，tensor支持的数据类型为：int8、uint8、int16、uint16、int32、uint32、int64、uint64、float16、float32、float64、bfp16。                                                                                                    |
| send_counts        | 输入      | 发送的数据量，send_counts[i]表示本rank发给rank i的数据个数，基本单位是send_data数据类型对应的字节数。例：send_data的数据类型为int32，send_counts[0]=1,send_count[1]=2，表示本rank给rank0发送1个int32类型的数据，给rank1发送2个int32类型的数据。TensorFlow的tensor类型，tensor支持的数据类型为int64。                                                                                                                        |
| send_displacements | 输入      | 发送数据的偏移量，send_displacements[i]表示本rank发送给rank i的数据块相对于send_data的偏移量，基本单位是send_data数据类型对应字节数。例：send_data的数据类型为int32。send_counts[0]=1,send_counts[1]=2；send_displacements[0]=0,send_displacements[1]=1。则表示本rank给rank0发送send_data上的第1个int32类型的数据，给rank1发送send_data上第2个与第3个int32类型的数据。TensorFlow的tensor类型，tensor支持的数据类型为int64。 |
| recv_counts        | 输入      | 接收的数据量，recv_counts[i]表示本rank从rank i收到的数据量。使用方法与send_counts类似。TensorFlow的tensor类型。tensor支持的数据类型为int64。                                                                                                                                                                                                                                                                                |
| recv_displacements | 输入      | 接收数据的偏移量，recv_displacements[i]表示本rank发送给rank i数据块相对于recv_data的偏移量，基本单位是recv_data_type的字节数。使用方法与send_displacements类似。TensorFlow的tensor类型。tensor支持的数据类型为int64。                                                                                                                                                                                                       |
| group              | 输入      | group名称，可以为用户自定义group或者"hccl_world_group"。String类型，最大长度为128字节，含结束符。                                                                                                                                                                                                                                                                                                                           |

## 返回值

recv_data：对输入tensor执行完all_to_all_v操作之后的结果tensor。

## 约束说明

1. 调用该接口的rank必须在当前接口入参group定义的范围内，不在此范围内的rank调用该接口会失败。

2. 针对Atlas训练系列产品，alltoallv的通信域需要满足如下约束：
  - 集群组网下，单server 1p、2p通信域要在同一个cluster内（server内0-3卡和4-7卡各为一个cluster），单server4p、8p和多server通信域中rank要以cluster为基本单位，并且server间cluster选取要一致。

3. alltoallv操作的性能与NPU之间共享数据的缓存区大小有关，当通信数据量超过缓存区大小时性能将出现明显下降。若业务中alltoallv通信数据量较大，建议通过配置环境变量HCCL_BUFFSIZE适当增大缓存区大小以提升通信性能。

4. 该接口不支持非集群场景下使用。

## 支持的型号

- Atlas训练系列产品
- Atlas A2 训练系列产品

## 调用示例

```python
from npu_bridge.npu_init import *
result = hccl_ops.all_to_all_v(send_data_tensor, send_counts_tensor, send_displacements_tensor, recv_counts_tensor, recv_displacements_tensor)
```
