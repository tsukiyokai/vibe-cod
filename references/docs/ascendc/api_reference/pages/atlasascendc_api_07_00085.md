# is_layout-Layout-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00085
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00085.html
---

# is_layout

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

判断输入的数据结构是否为Layout数据结构，可通过检查其成员常量value的值来判断。当value为true时，表示输入的数据结构是Layout类型；反之则为非Layout类型。

#### 函数原型

| 1   | template<typenameT>structis_layout |
| --- | ---------------------------------- |

#### 参数说明

| 参数名 | 描述                                           |
| ------ | ---------------------------------------------- |
| T      | 根据输入的数据类型，判断是否为Layout数据结构。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 123456789101112 | // 初始化Layout数据结构并判断其类型AscendC:Shape<int,int,int>shape=AscendC:MakeShape(10,20,30);AscendC:Stride<int,int,int>stride=AscendC:MakeStride(1,100,200);autolayoutMake=AscendC:MakeLayout(shape,stride);AscendC:Layout<AscendC:Shape<int,int,int>,AscendC:Stride<int,int,int>>layoutInit(shape,stride);boolvalue=AscendC:is_layout<decltype(shape)>:value;//value = falsevalue=AscendC:is_layout<decltype(stride)>:value;//value = falsevalue=AscendC:is_layout<decltype(layoutMake)>:value;//value = truevalue=AscendC:is_layout<decltype(layoutInit)>:value;//value = true |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
