# CountBitsCntSameAsSignBit-标量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0019
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0019.html
---

# CountBitsCntSameAsSignBit

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

计算一个int64_t类型数字的二进制中，从最高数值位开始与符号位相同的连续比特位的个数。

当输入是-1（比特位全1）或者0（比特位全0）时，返回-1。

#### 函数原型

| 1   | __aicore__inlineint64_tCountBitsCntSameAsSignBit(int64_tvalueIn) |
| --- | ---------------------------------------------------------------- |

#### 参数说明

| 参数名  | 输入/输出 | 描述                        |
| ------- | --------- | --------------------------- |
| valueIn | 输入      | 输入数据，数据类型int64_t。 |

#### 返回值说明

返回从最高数值位开始和符号位相同的连续比特位的个数。

#### 约束说明

无

#### 调用示例

| 123 | int64_tvalueIn=0x0f00000000000000;// 输出数据(ans): 3int64_tans=AscendC:CountBitsCntSameAsSignBit(valueIn); |
| --- | ----------------------------------------------------------------------------------------------------------- |
