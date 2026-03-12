# GetBlockNum-系统变量访问-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0184
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0184.html
---

# GetBlockNum

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

获取当前任务配置的核数，用于代码内部的多核逻辑控制等。

#### 函数原型

| 1   | __aicore__inlineint64_tGetBlockNum() |
| --- | ------------------------------------ |

#### 参数说明

无

#### 返回值说明

当前任务配置的核数。

#### 约束说明

无。

#### 调用示例

| 123456 | #include"kernel_operator.h"// 在核内做简单的tiling计算时使用block_num，复杂tiling建议在host侧完成__aicore__inlinevoidInitTilingParam(int32_t&totalSize,int32_t&loopSize){loopSize=totalSize/AscendC:GetBlockNum();}; |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
