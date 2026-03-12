# IBWait-核间同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0203
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0203.html
---

# IBWait

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

当不同核之间操作同一块全局内存且可能存在读后写、写后读以及写后写等数据依赖问题时，通过调用该函数来插入同步语句来避免上述数据依赖时可能出现的数据读写错误问题。IBWait与IBSet成对出现配合使用，表示核之间的同步等待指令，等待某一个核操作完成。

#### 函数原型

| 12  | template<boolisAIVOnly=true>__aicore__inlinevoidIBWait(constGlobalTensor<int32_t>&gmWorkspace,constLocalTensor<int32_t>&ubWorkspace,int32_tblockIdx,int32_teventID) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名    | 描述                                |
| --------- | ----------------------------------- |
| isAIVOnly | 控制是否为AIVOnly模式，默认为true。 |

| 参数名      | 输入/输出 | 描述                                                                                         |
| ----------- | --------- | -------------------------------------------------------------------------------------------- |
| gmWorkspace | 输出      | 外部存储核状态的公共缓存，类型为GlobalTensor。GlobalTensor数据结构的定义请参考GlobalTensor。 |
| ubWorkspace | 输入      | 当前核的公共缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                 |
| blockIdx    | 输入      | 表示等待核的idx号，取值范围：[0, 核数-1]，不包含自身blockIdx。                               |
| eventID     | 输入      | 用来控制当前核的set、wait事件。                                                              |

#### 返回值说明

无

#### 约束说明

- gmWorkspace申请的空间最少要求为：核数 * 32Bytes * eventID_max + blockIdx_max * 32Bytes + 32Bytes。（eventID_max和blockIdx_max分别指eventID、blockIdx的最大值）
- ubWorkspace申请的空间最少要求为：32Bytes。
- 使用该接口进行多核控制时，算子调用时指定的逻辑blockDim必须保证不大于实际运行该算子的AI处理器核数，否则框架进行多轮调度时会插入异常同步，导致Kernel“卡死”现象。

#### 调用示例

调用样例请参考调用示例。
