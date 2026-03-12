# layout-Layout-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00079
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00079.html
---

# layout

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

获取Layout实例化对象。

#### 函数原型

| 12  | __aicore__inlineconstexprdecltype(auto)layout(){}__aicore__inlineconstexprdecltype(auto)layout()const{} |
| --- | ------------------------------------------------------------------------------------------------------- |

#### 参数说明

无

#### 返回值说明

返回Layout实例化对象。

#### 约束说明

构造Layout对象时传入的Shape和Stride结构，需是Std:tuple结构类型，且满足Std:tuple结构类型的使用约束。

#### 调用示例

| 1234567 | AscendC:Shape<int,int,int>shape=AscendC:MakeShape(10,20,30);AscendC:Stride<int,int,int>stride=AscendC:MakeStride(1,100,200);AscendC:Layout<AscendC:Shape<int,int,int>,AscendC:Stride<int,int,int>>layoutInit(shape,stride);// 使用layout函数获取实例化对象constexprauto&layout=layoutInit.layout(); |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
