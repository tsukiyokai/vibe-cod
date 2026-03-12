# Layout简介-Layout-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00076
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00076.html
---

# Layout简介

Layout<Shape, Stride>数据结构是描述多维张量内存布局的基础模板类，通过编译时的形状(Shape)和步长(Stride)信息，实现逻辑坐标空间到一维内存地址空间的映射，为复杂张量操作和硬件优化提供基础支持。借助模板元编程技术，该类在编译时完成计算和代码生成，从而降低运行时开销。

Layout包含两个核心组成部分：

- Shape：定义数据的逻辑形状，例如二维矩阵的行数和列数或多维张量的各维度大小。
- Stride：定义各维度在内存中的步长，即同维度相邻元素在内存中的间隔，间隔的单位为元素，与Shape的维度信息一一对应。

例如，一个二维矩阵的Shape为(4, 2)，Stride为(4, 1)，表示：

- 矩阵有4行、2列。
- 列方向上的步长为1，即每行中相邻元素在内存中的间隔为1个元素；行方向上的步长为4，即相邻行的起始地址间隔为4个元素。

表1中给出了一维内存地址空间视图，表2中给出了该二维矩阵的逻辑视图。

| 地址 | 0   | 1   | 2、3 | 4   | 5   | 6、7 | 8   | 9   | 10、11 | 12  | 13  |
| ---- | --- | --- | ---- | --- | --- | ---- | --- | --- | ------ | --- | --- |
| 元素 | a00 | a01 | -    | a10 | a11 | -    | a20 | a21 | -      | a30 | a31 |

| 索引 | 列0           | 列1           |
| ---- | ------------- | ------------- |
| 行0  | a00（地址0）  | a01（地址1）  |
| 行1  | a10（地址4）  | a11（地址5）  |
| 行2  | a20（地址8）  | a21（地址9）  |
| 行3  | a30（地址12） | a31（地址13） |

#### 需要包含的头文件

| 1   | #include"kernel_operator_layout.h" |
| --- | ---------------------------------- |

#### 原型定义

| 12345678910111213141516 | template<typenameShapeType,typenameStrideType>structLayout:privateStd:tuple<ShapeType,StrideType>{__aicore__inlineconstexprLayout(constShapeType&shape={},constStrideType&stride={}):Std:tuple<ShapeType,StrideType>(shape,stride){}__aicore__inlineconstexprdecltype(auto)layout(){}__aicore__inlineconstexprdecltype(auto)layout()const{}__aicore__inlineconstexprdecltype(auto)GetShape(){}__aicore__inlineconstexprdecltype(auto)GetShape()const{}__aicore__inlineconstexprdecltype(auto)GetStride(){}__aicore__inlineconstexprdecltype(auto)GetStride()const{}template<typenameCoordType>__aicore__inlineconstexprautooperator()(constCoordType&coord)const{}} |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 模板参数

| 参数名     | 描述                                                                                                                           |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------ |
| ShapeType  | Std:tuple结构类型，用于定义数据的逻辑形状，例如二维矩阵的行数和列数或多维张量的各维度大小。                                    |
| StrideType | Std:tuple结构类型，用于定义各维度在内存中的步长，即同维度相邻元素在内存中的间隔，间隔的单位为元素，与Shape的维度信息一一对应。 |

#### 成员函数

#### 相关接口
