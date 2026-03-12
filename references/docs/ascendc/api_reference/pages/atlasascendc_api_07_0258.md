# SetHF32Mode-矩阵计算-矩阵计算(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0258
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0258.html
---

# SetHF32Mode

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

用于设置Mmad计算是否开启HF32模式，开启该模式后L0A/L0B中的FP32数据将在参与Mmad计算之前被舍入为HF32。

#### 函数原型

| 1   | __aicore__inlinevoidSetHF32Mode(boolhf32Mode) |
| --- | --------------------------------------------- |

#### 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                                        |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| hf32Mode | 输入      | Mmad HF32模式控制入参，bool类型。支持如下两种取值：true：L0A/L0B中的FP32数据将在矩阵乘法之前被舍入为HF32。false：将执行常规的FP32矩阵乘法。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 12  | boolhf32Mode=true;AscendC:SetHF32Mode(hf32Mode); |
| --- | ------------------------------------------------ |
