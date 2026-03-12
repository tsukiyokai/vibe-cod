# ICachePreLoad(ISASI)-缓存控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0276
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0276.html
---

# ICachePreLoad(ISASI)

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

从指令所在DDR地址预加载指令到ICache中。

#### 函数原型

| 1   | __aicore__inlinevoidICachePreLoad(constint64_tpreFetchLen) |
| --- | ---------------------------------------------------------- |

#### 参数说明

| 参数名      | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                          |
| ----------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| preFetchLen | 输入      | 预取长度。针对Atlas A2 训练系列产品/Atlas A2 推理系列产品：preFetchLen参数单位为2K Byte, 取值应小于ICache的大小/2K。AIC和AIV的ICache大小分别为32KB和16KB。针对Atlas A3 训练系列产品/Atlas A3 推理系列产品：preFetchLen参数单位为2K Byte, 取值应小于ICache的大小/2K。AIC和AIV的ICache大小分别为32KB和16KB。针对Atlas推理系列产品AI Core：传入该参数无效，预取长度均为128Byte。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 12  | int64_tpreFetchLen=2;AscendC:ICachePreLoad(preFetchLen); |
| --- | -------------------------------------------------------- |
