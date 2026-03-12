# ScalarGetCountOfValue-标量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0016
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0016.html
---

# ScalarGetCountOfValue

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

获取一个uint64_t类型数字的二进制中0或者1的个数。

#### 函数原型

| 12  | template<intcountValue>__aicore__inlineint64_tScalarGetCountOfValue(uint64_tvalueIn) |
| --- | ------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名     | 描述                                     |
| ---------- | ---------------------------------------- |
| countValue | 指定统计0还是统计1的个数。只能输入0或1。 |

| 参数名  | 输入/输出 | 描述                 |
| ------- | --------- | -------------------- |
| valueIn | 输入      | 被统计的二进制数字。 |

#### 返回值说明

valueIn中0或者1的个数。

#### 约束说明

无

#### 调用示例

| 123 | uint64_tvalueIn=0xffff;// 输出数据oneCount: 16int64_toneCount=AscendC:ScalarGetCountOfValue<1>(valueIn); |
| --- | -------------------------------------------------------------------------------------------------------- |
