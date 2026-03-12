# RmsNorm Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0805
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0805.html
---

# RmsNorm Tiling

#### 功能说明

Ascend C提供RmsNorm Tiling API，方便用户获取RmsNorm kernel计算时所需的Tiling参数。

获取Tiling参数主要分为如下两步：

1. 通过GetRmsNormMaxMinTmpSize获取RmsNorm接口计算所需最大和最小临时空间大小。kernel侧RmsNorm接口的计算需要开发者预留/申请临时空间，GetRmsNormMaxMinTmpSize用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为Tiling参数传递到kernel侧使用。为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。
1. 通过GetRmsNormTilingInfo获取RmsNorm kernel侧接口所需tiling参数。RmsNorm Tiling结构体的定义如下，开发者无需关注该tiling结构的具体信息，只需要传递到kernel侧，传入RmsNorm高阶API接口，直接进行使用即可。1234567891011121314structRmsNormTiling{uint32_tbLength=0;uint32_tsLength=0;uint32_thLength=0;uint32_toriginalHLength=0;floatreciprocalOfHLength=0;uint32_tmainBshLength=0;uint32_tmainBsLength=0;uint32_tmainBsLengthAlign=0;uint32_tloopRound=0;uint32_tinputTailPos=0;uint32_ttailBshLength=0;uint32_ttailBsLength=0;};

#### 函数原型

| 1   | boolGetRmsNormMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,uint32_t&maxValue,uint32_t&minValue,constboolisBasicBlock=false) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetRmsNormTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferByteSize,constuint32_ttypeSize,optiling:RmsNormTiling&tiling,constboolisBasicBlock=false) |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetRmsNormTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferByteSize,constuint32_ttypeSize,AscendC:tiling:RmsNormTiling&tiling,constboolisBasicBlock=false) |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名       | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape     | 输入      | 输入的shape信息。                                                                                                                                                                                                                                                                                                                                                                                                                              |
| typeSize     | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                                                                        |
| maxValue     | 输出      | RmsNorm接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。最大空间大小为0表示计算不需要临时空间。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue     | 输出      | RmsNorm接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。最小空间大小为0表示计算不需要临时空间。                                                                                                                                                                                                                                                                                                |
| isBasicBlock | 输入      | 是否要使能基本块计算，与kernel侧接口一致，默认false。                                                                                                                                                                                                                                                                                                                                                                                          |

| 参数名              | 输入/输出 | 描述                                                                                                                                                                  |
| ------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape            | 输入      | 输入的tensor的shape信息，这里是H轴向上32B对齐后的shape。需要保证srcShape的B/S和originSrcShape的B/S一致。                                                              |
| originSrcShape      | 输入      | 输入的原始shape信息。                                                                                                                                                 |
| stackBufferByteSize | 输入      | 剩余的可供RmsNorm接口计算的空间大小，单位为Byte。通过GetRmsNormMaxMinTmpSize获取最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为stackBufferByteSize传入。 |
| typeSize            | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                               |
| tiling              | 输出      | RmsNorm计算所需Tiling信息。                                                                                                                                           |
| isBasicBlock        | 输入      | 是否要使能基本块计算，与kernel侧接口一致，默认false。若使能基本块，则需要保证originSrcShape的H也是32B对齐。                                                           |

#### 返回值说明

- GetRmsNormMaxMinTmpSize返回值为true/false，true表示成功拿到RmsNorm接口内部计算需要的最大和最小临时空间大小；false表示获取失败，获取失败情况下，需要检查输入的shape是否符合要求。
- GetRmsNormTilingInfo返回类型为true/false，true表示成功拿到RmsNorm的Tiling各项参数值；false表示获取失败，获取失败情况下需要检查输入的stackBufferByteSize是否满足最小临时空间要求，若开启isBasicBlock开关，则需要检查输入shape是否满足基本块的要求。

#### 约束说明

无

#### 调用示例

1. 将RmsNorm Tiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。1234567BEGIN_TILING_DATA_DEF(RmsnormCustomTilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,totalLength);// 添加tiling字段，总计算数据量TILING_DATA_FIELD_DEF(uint32_t,tileNum);// 添加tiling字段，每个核上总计算数据分块个数TILING_DATA_FIELD_DEF(uint32_t,tmpBufSize);// 添加tiling字段，临时空间大小...// 添加其他tiling字段TILING_DATA_FIELD_DEF_STRUCT(RmsNormTiling,rmsnormTilingData);// 将RmsNormTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetRmsNormMaxMinTmpSize接口获取RmsNorm接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后根据输入shape、剩余的可供计算的空间大小等信息获取RmsNorm kernel侧接口所需tiling参数。12345678910111213141516171819202122232425262728293031namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){RmsNormCustomTilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_totalLength(totalLength);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...std:vector<int64_t>shapeVec={2,16,64};ge:ShapesrcShape(shapeVec);std:vector<int64_t>oriShapeVec={2,16,64};ge:ShapeoriSrcShape(oriShapeVec);// 本样例中仅作为样例说明，通过GetRmsNormMaxMinTmpSize获取最小值并传入，来保证功能正确，开发者可以根据需要传入合适的空间大小uint32_tminValue=0;uint32_tmaxValue=0;AscendC:GetRmsNormMaxMinTmpSize(srcShape,sizeof(half),maxValue,minValue,isBasicBlock);tiling.set_tmpBufSize(minValue);// 获取RmsNorm Tiling参数AscendC:GetRmsNormTilingInfo(srcShape,oriSrcShape,minValue,sizeof(half),tiling.rmsnormTilingData,false);...// 其他逻辑tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的RmsNorm Tiling信息传入RmsNorm接口参与计算。完整的kernel侧样例请参考RmsNorm。123456789extern"C"__global____aicore__voidrmsnorm_custom(GM_ADDRinputGm,GM_ADDRgammaGm,GM_ADDRoutputGm,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelRmsNormop;op.Init(inputGm,gammaGm,outputGm,tilingData.totalLength,tilingData.tileNum,tilingData.rmsnormTilingData);if(TILING_KEY_IS(1)){op.Process();}}
