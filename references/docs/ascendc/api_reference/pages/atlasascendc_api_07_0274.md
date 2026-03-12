# CrossCoreWaitFlag(ISASI)-核间同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0274
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0274.html
---

# CrossCoreWaitFlag(ISASI)

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

面向分离模式的核间同步控制接口。该接口和CrossCoreSetFlag接口配合使用。具体使用方法请参考CrossCoreSetFlag。

#### 函数原型

| 12  | template<uint8_tmodeId=0,pipe_tpipe=PIPE_S>__aicore__inlinevoidCrossCoreWaitFlag(uint16_tflagId) |
| --- | ------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名 | 描述                                                                                                                                                                    |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| modeId | 核间同步的模式，取值如下：模式0：AI Core核间的同步控制。模式1：AI Core内部，Vector核(AIV)之间的同步控制。模式2：AI Core内部，Cube核(AIC)与Vector核(AIV)之间的同步控制。 |
| pipe   | 设置这条指令所在的流水类型，流水类型可参考硬件流水类型。                                                                                                                |

| 参数名 | 输入/输出 | 描述                                                                                                                                       |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| flagId | 输入      | 核间同步的标记。Atlas A2 训练系列产品/Atlas A2 推理系列产品，取值范围是0-10。Atlas A3 训练系列产品/Atlas A3 推理系列产品，取值范围是0-10。 |

#### 返回值说明

无

#### 约束说明

- 使用该同步接口时，需要按照如下规则设置Kernel类型：在纯Vector/Cube场景下，需设置Kernel类型为KERNEL_TYPE_MIX_AIV_1_0或KERNEL_TYPE_MIX_AIC_1_0。对于Vector和Cube混合场景，需根据实际情况灵活配置Kernel类型。

- CrossCoreWaitFlag必须与CrossCoreSetFlag接口配合使用，避免计算核一直处于阻塞阶段。
- 如果执行CrossCoreWaitFlag时该flagId的计数器的值为0，则CrossCoreWaitFlag之后的所有指令都将被阻塞，直到该flagId的计数器的值不为0。同一个flagId的计数器最多设置15次。

#### 调用示例

请参考调用示例。
