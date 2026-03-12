# GetStride-Layout-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00081
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00081.html
---

# GetStride

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

返回描述内存访问步长的Stride对象，与Shape的维度信息一一对应。

#### 函数原型

| 12  | __aicore__inlineconstexprdecltype(auto)GetStride(){}__aicore__inlineconstexprdecltype(auto)GetStride()const{} |
| --- | ------------------------------------------------------------------------------------------------------------- |

#### 参数说明

无

#### 返回值说明

描述内存访问步长的Stride对象，Stride结构类型（Std:tuple类型的别名），定义如下：

| 12  | template<typename...Strides>usingStride=Std:tuple<Strides...>; |
| --- | -------------------------------------------------------------- |

#### 约束说明

无

#### 调用示例

| 12345678910 | // 初始化Layout数据结构，获取对应数值AscendC:Shape<int,int,int>shape=AscendC:MakeShape(10,20,30);AscendC:Stride<int,int,int>stride=AscendC:MakeStride(1,100,200);autolayoutMake=AscendC:MakeLayout(shape,stride);AscendC:Layout<AscendC:Shape<int,int,int>,AscendC:Stride<int,int,int>>layoutInit(shape,stride);intvalue=AscendC:Std:get<0>(layoutInit.GetStride());// value = 1value=AscendC:Std:get<1>(layoutInit.GetStride());// value = 100value=AscendC:Std:get<2>(layoutInit.GetStride());// value = 200 |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
