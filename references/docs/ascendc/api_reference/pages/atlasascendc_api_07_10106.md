# integer_sequence-通用工具-C++标准库-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10106
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10106.html
---

# integer_sequence

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

index_sequence是Ascend C提供的一个类模板，用于生成一个编译时的整数序列，适用于模板元编程。

make_index_sequence是Ascend C提供的一个模板，通常使用make_index_sequence创建一个index_sequence类型的对象，用于生成一个从0到N-1的整数序列。

#### 函数原型

| 12  | template<size_t...Idx>usingindex_sequence=IntegerSequence<size_t,Idx...>; |
| --- | ------------------------------------------------------------------------- |

| 12  | template<size_tN>usingmake_index_sequence=MakeIntegerSequence<size_t,N>; |
| --- | ------------------------------------------------------------------------ |

#### 参数说明

| 参数名 | 含义                                                                                        |
| ------ | ------------------------------------------------------------------------------------------- |
| ...Idx | 表示序列的形参包。size_t，在64位系统中为long unsigned int，非64位系统中为unsigned int。     |
| N      | 生成的整数序列的大小。size_t，在64位系统中为long unsigned int，非64位系统中为unsigned int。 |

#### 约束说明

- N的范围为[0, 64]。
- index_sequence作为序列，长度最大为64。

#### 返回值说明

无

#### 调用示例

生成并打印一个长度为5的整数序列。

| 123456789 | template<size_t...Is>__aicore__inlinevoidPrintIndexSequence(AscendC:Std:index_sequence<Is...>){((AscendC:printf(" Is:%lu",Is)),...);}__aicore__inlinevoidProcess(){PrintIndexSequence(AscendC:Std:make_index_sequence<5>{});// 打印结果：0，1，2，3，4PrintIndexSequence(AscendC:Std:index_sequence<0,1,2,10,8000>{});// 打印结果：0，1，2，10, 8000} |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
