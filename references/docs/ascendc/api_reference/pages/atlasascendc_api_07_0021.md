# ToBfloat16-标量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0021
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0021.html
---

# ToBfloat16

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

float类型标量数据转换成bfloat16_t类型标量数据。

#### 函数原型

| 1   | __aicore__inlinebfloat16_tToBfloat16(constfloat&fVal) |
| --- | ----------------------------------------------------- |

#### 参数说明

| 参数名称 | 输入/输出 | 含义                |
| -------- | --------- | ------------------- |
| fVal     | 输入      | float类型标量数据。 |

#### 返回值说明

转换后的bfloat16_t类型标量数据。

#### 约束说明

无

#### 调用示例

| 12  | floatm=3.0f;bfloat16_tn=AscendC:ToBfloat16(m); |
| --- | ---------------------------------------------- |
