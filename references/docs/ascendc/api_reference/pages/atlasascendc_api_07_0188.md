# GetTaskRatio-工具函数-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0188
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0188.html
---

# GetTaskRatio

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

分离模式下，获取一个AI Core上Cube Core(AIC)或者Vector Core(AIV)的数量与AI Core数量的比例。耦合模式下，固定返回1。

#### 函数原型

| 1   | __aicore__inlineint64_tGetTaskRatio() |
| --- | ------------------------------------- |

#### 参数说明

无

#### 返回值说明

- 针对分离模式，不同Kernel类型下（通过设置Kernel类型接口设置），在AIC和AIV上调用该接口的返回值如下：表1返回值列表Kernel类型KERNEL_TYPE_AIV_ONLYKERNEL_TYPE_AIC_ONLYKERNEL_TYPE_MIX_AIC_1_2KERNEL_TYPE_MIX_AIC_1_1KERNEL_TYPE_MIX_AIC_1_0KERNEL_TYPE_MIX_AIV_1_0AIV1-21-1AIC-1111-
- 针对耦合模式，固定返回1。

#### 约束说明

无

#### 调用示例

| 12  | uint64_tratio=AscendC:GetTaskRatio();AscendC:PRINTF("task ratio is %u",ratio); |
| --- | ------------------------------------------------------------------------------ |
