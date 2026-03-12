# 昇腾社区官网-昇腾万里 让智能无所不及

**页面ID:** atlasascendc_api_07_0178
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0178.html
---

# 模板参数

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

TQueSync类的模板参数用于指定源流水和目标流水，目的流水等待源流水。

#### 函数原型

| 123456 | template<pipe_tsrc,pipe_tdst>classTQueSync{public:__aicore__inlinevoidSetFlag(TEventIDid);__aicore__inlinevoidWaitFlag(TEventIDid);}; |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                          |
| ------ | --------- | ----------------------------- |
| src    | 输入      | 源流水。支持的流水参考表1。   |
| dst    | 输入      | 目的流水。支持的流水参考表1。 |

#### 返回值说明

无

#### 约束说明

无
