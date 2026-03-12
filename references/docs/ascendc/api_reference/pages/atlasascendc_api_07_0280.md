# GetSubBlockNum(ISASI)-系统变量访问-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0280
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0280.html
---

# GetSubBlockNum(ISASI)

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

分离模式下，获取一个AI Core上Cube Core(AIC)或者Vector Core(AIV)的数量。

#### 函数原型

| 1   | __aicore__inlineint64_tGetSubBlockNum() |
| --- | --------------------------------------- |

#### 参数说明

无

#### 返回值说明

不同Kernel类型下（通过设置Kernel类型设置），在AIC和AIV上调用该接口的返回值如下：

| Kernel类型 | KERNEL_TYPE_AIV_ONLY | KERNEL_TYPE_AIC_ONLY | KERNEL_TYPE_MIX_AIC_1_2 | KERNEL_TYPE_MIX_AIC_1_1 | KERNEL_TYPE_MIX_AIC_1_0 | KERNEL_TYPE_MIX_AIV_1_0 |
| ---------- | -------------------- | -------------------- | ----------------------- | ----------------------- | ----------------------- | ----------------------- |
| AIV        | 1                    | -                    | 2                       | 1                       | -                       | 1                       |
| AIC        | -                    | 1                    | 1                       | 1                       | 1                       | -                       |

#### 约束说明

无

#### 调用示例

| 1   | int64_tsubBlockNum=AscendC:GetSubBlockNum(); |
| --- | -------------------------------------------- |
