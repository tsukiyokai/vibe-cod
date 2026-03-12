# DataSyncBarrier(ISASI)-核内同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0272
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0272.html
---

# DataSyncBarrier(ISASI)

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

用于阻塞后续的指令执行，直到所有之前的内存访问指令（需要等待的内存位置可通过参数控制）执行结束。

#### 函数原型

| 12  | template<MemDsbTarg0>__aicore__inlinevoidDataSyncBarrier() |
| --- | ---------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                                                                                             |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| arg0   | 模板参数，表示需要等待的内存位置，类型为MemDsbT，可取值为：ALL，等待所有内存访问指令。DDR，等待GM访问指令。UB，等待UB访问指令。SEQ，预留参数，暂未启用，为后续的功能扩展做保留。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 123 | AscendC:Mmad(...);AscendC:DataSyncBarrier<MemDsbT:ALL>();AscendC:Fixpipe(...); |
| --- | ------------------------------------------------------------------------------ |
