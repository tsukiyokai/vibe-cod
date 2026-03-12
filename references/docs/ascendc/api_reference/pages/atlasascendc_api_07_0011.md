# TensorTrait简介-TensorTrait-基础数据结构-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0011
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0011.html
---

# TensorTrait简介

TensorTrait数据结构是描述Tensor相关信息的基础模板类，包含Tensor的数据类型、逻辑位置和Layout内存布局。借助模板元编程技术，该类在编译时完成计算和代码生成，从而降低运行时开销。

#### 需要包含的头文件

| 1   | #include"kernel_operator_tensor_trait.h" |
| --- | ---------------------------------------- |

#### 原型定义

#### 模板参数

| 参数名     | 描述                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T          | 只支持如下基础数据类型：int4b_t、uint8_t、int8_t、int16_t、uint16_t、bfloat16_t、int32_t、uint32_t、int64_t、uint64_t、float、half。在TensorTrait结构体内部，使用using关键字定义了一个类型别名LiteType，与模板参数T类型一致。通过TensorTrait定义的LocalTensor/GlobalTensor不包含ShapeInfo信息。例如：LocalTensor<float>对应的不含ShapeInfo信息的Tensor为LocalTensor<TensorTrait<float>>。 |
| pos        | 数据存放的逻辑位置，Tposition类型，默认为TPosition:GM。                                                                                                                                                                                                                                                                                                                                   |
| LayoutType | Layout数据类型，默认为空类型，即Layout<Shape<>, Stride<>>。输入的数据类型LayoutType，需满足约束说明。                                                                                                                                                                                                                                                                                     |

#### 成员函数

#### 相关接口

#### 约束说明

- 同一接口不支持同时输入TensorTrait类型的GlobalTensor/LocalTensor和非TensorTrait类型的GlobalTensor/LocalTensor。
- 非TensorTrait类型和TensorTrait类型的GlobalTensor/LocalTensor相互之间不支持拷贝构造和赋值运算符。
- TensorTrait特性当前仅支持如下接口：和API配合使用时，当前暂不支持TensorTrait结构配置pos、LayoutType模板参数，需要使用构造函数构造TensorTrait，pos、LayoutType保持默认值即可。DataCopy切片数据搬运接口需要ShapeInfo信息，不支持输入TensorTrait类型的GlobalTensor/LocalTensor。表2TensorTrait特性支持的接口列表接口分类接口名称基础API>资源管理>TQue/TQueBindAllocTensor、FreeTensor、EnQue、DeQue基础API>矢量计算>基础算术Exp、Ln、Abs、Reciprocal、Sqrt、Rsqrt、Relu、Add、Sub、Mul、Div、Max、Min、Adds、Muls、Maxs、Mins、VectorPadding、BilinearInterpolation、LeakyRelu基础API>矢量计算>逻辑计算And、Or基础API>矢量计算>复合计算CastDeq、AddRelu、AddDeqRelu、SubRelu、MulAddDst、FusedMulAdd、FusedMulAddRelu、AddReluCast、SubReluCast、MulCast基础API>数据搬运DataCopy、Copy基础API>矩阵计算InitConstValue、LoadData、LoadDataWithTranspose、SetAippFunctions、LoadImageToLocal、LoadUnzipIndex、LoadDataUnzip、LoadDataWithSparse、Mmad、MmadWithSparse、BroadCastVecToMM、Gemm、Fixpipe基础API>矢量计算>比较与选择Compare、GetCmpMask、SetCmpMask、Select、GatherMask基础API>矢量计算>类型转换Cast基础API>矢量计算>归约计算ReduceMax、BlockReduceMax、WholeReduceMax、ReduceMin、BlockReduceMin、WholeReduceMin、ReduceSum、BlockReduceSum、WholeReduceSum、RepeatReduceSum、PairReduceSum基础API>矢量计算>数据转换Transpose、TransDataTo5HD基础API>矢量计算>数据填充Brcb基础API>矢量计算>离散与聚合Gather、Gatherb、Scatter基础API>矢量计算>排序组合(ISASI)ProposalConcat、ProposalExtract、RpSort16、MrgSort4、Sort32
