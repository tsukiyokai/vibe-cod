# MakeShape-Layout-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00082
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00082.html
---

# MakeShape

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

将传入的数据打包成Shape数据结构。

#### 函数原型

| 12  | template<typename...Ts>__aicore__inlineconstexprShape<Ts...>MakeShape(constTs&...t) |
| --- | ----------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                  |
| ------ | --------- | ----------------------------------------------------- |
| Ts...  | 输入      | 表示输入类型的形参包，使用方法和约束说明同Std:tuple。 |

#### 返回值说明

Shape结构类型（Std:tuple类型的别名），用于定义数据的逻辑形状，例如二维矩阵的行数和列数或多维张量的各维度大小。定义如下：

| 12  | template<typename...Shapes>usingShape=Std:tuple<Shapes...>; |
| --- | ----------------------------------------------------------- |

#### 约束说明

同Std:tuple。

#### 调用示例

参见调用示例。
