# ScalarCountLeadingZero-标量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0017
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0017.html
---

# ScalarCountLeadingZero

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

计算一个uint64_t类型数字前导0的个数（二进制从最高位到第一个1一共有多少个0）。

#### 函数原型

| 1   | __aicore__inlineint64_tScalarCountLeadingZero(uint64_tvalueIn) |
| --- | -------------------------------------------------------------- |

#### 参数说明

| 参数名  | 输入/输出 | 描述                 |
| ------- | --------- | -------------------- |
| valueIn | 输入      | 被统计的二进制数字。 |

#### 返回值说明

返回valueIn的前导0的个数。

#### 约束说明

无

#### 调用示例

| 123 | uint64_tvalueIn=0x0fffffffffffffff;// 输出数据ans：4int64_tans=AscendC:ScalarCountLeadingZero(valueIn); |
| --- | ------------------------------------------------------------------------------------------------------- |
