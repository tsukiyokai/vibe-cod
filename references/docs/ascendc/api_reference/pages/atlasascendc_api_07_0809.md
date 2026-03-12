# DeepNorm Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0809
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0809.html
---

# DeepNorm Tiling

#### 功能说明

Ascend C提供DeepNorm Tiling API，方便用户获取DeepNorm kernel计算时所需的Tiling参数。

获取Tiling参数主要分为如下两步：

1. 通过GetDeepNormMaxMinTmpSize获取DeepNorm接口计算所需最大和最小临时空间大小。kernel侧DeepNorm接口的计算需要开发者预留/申请临时空间，GetDeepNormMaxMinTmpSize用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为Tiling参数传递到kernel侧使用。为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。
1. 通过GetDeepNormTilingInfo获取DeepNormkernel侧接口所需tiling参数。DeepNormTiling结构体的定义如下，开发者无需关注该tiling结构的具体信息，只需要传递到kernel侧，传入DeepNorm高阶API接口，直接进行使用即可。12345678910111213141516171819202122232425262728structDeepNormTiling{uint32_tbLength=0;uint32_tsLength=0;uint32_thLength=0;uint32_toriginalHLength=0;uint32_tinputXSize=0;uint32_tmeanVarSize=0;uint32_tnumberOfTmpBuf=0;uint32_tmeanTmpTensorPos=0;uint32_tmeanTmpTensorSize=0;uint32_tvarianceTmpTensorPos=0;uint32_tvarianceTmpTensorSize=0;uint32_ttmpBufSize=0;uint32_toneTmpSize=0;uint32_tfirstTmpStartPos=0;uint32_tsecondTmpStartPos=0;uint32_tthirdTmpStartPos=0;uint32_tloopRound=0;uint32_tinputRoundSize=0;uint32_tinputTailSize=0;uint32_tinputTailPos=0;uint32_tmeanVarRoundSize=0;uint32_tmeanVarTailSize=0;uint32_tmeanVarTailPos=0;uint32_tbshCurLength=0;uint32_tbsCurLength=0;floatlastDimValueBack=0;};

#### 函数原型

| 1   | boolGetDeepNormMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource,constboolisBasicBlock,uint32_t&maxValue,uint32_t&minValue) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetDeepNormTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,constboolisBasicBlock,optiling:DeepNormTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetDeepNormTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,constboolisBasicBlock,AscendC:tiling:DeepNormTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名        | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape      | 输入      | 输入的shape信息。                                                                                                                                                                                                                                                                                                                                                                                                                               |
| typeSize      | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                                                                         |
| isReuseSource | 输入      | 是否复用源操作数输入的空间，与DeepNorm接口一致。                                                                                                                                                                                                                                                                                                                                                                                                |
| isBasicBlock  | 输入      | srcShape是否符合基本块定义：尾轴H的长度为64的倍数（不超过2040），B*S为8的倍数。                                                                                                                                                                                                                                                                                                                                                                 |
| maxValue      | 输出      | DeepNorm接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。最大空间大小为0表示计算不需要临时空间。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue      | 输出      | DeepNorm接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。最小空间大小为0表示计算不需要临时空间。                                                                                                                                                                                                                                                                                                |

表2GetDeepNormTilingInfo接口参数说明

| 参数名          | 输入/输出 | 描述                                                                                                                                                    |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape        | 输入      | 输入的shape信息[B, S, H]。                                                                                                                              |
| originSrcShape  | 输入      | 32B对齐前的输入shape信息[B, S, originH]。originH的长度应该在(0, H]的范围内。如果isBasicBlock置为true，originH必须与H一致。                              |
| stackBufferSize | 输入      | 临时空间的buffer大小，单位为Byte。通过GetDeepNormMaxMinTmpSize获取最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为stackBufferByteSize传入。 |
| typeSize        | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                 |
| isReuseSource   | 输入      | 是否复用源操作数输入的空间，与DeepNorm接口一致。                                                                                                        |
| isBasicBlock    | 输入      | srcShape是否符合基本块定义：尾轴H的长度为64的倍数（不超过2040），B*S为8的倍数。                                                                         |
| tiling          | 输出      | DeepNorm计算所需Tiling信息。                                                                                                                            |

#### 返回值说明

- GetDeepNormMaxMinTmpSize返回值为true/false，true表示成功拿到DeepNorm接口内部计算需要的最大和最小临时空间大小；false表示获取失败。
- GetDeepNormTilingInfo返回类型为true/false，true表示成功拿到DeepNorm的Tiling各项参数值；false表示获取失败。

#### 约束说明

无

#### 调用示例

1. 将DeepNorm Tiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。123456BEGIN_TILING_DATA_DEF(TilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,totalLength);// 添加tiling字段，总计算数据量TILING_DATA_FIELD_DEF(uint32_t,tileNum);// 添加tiling字段，每个核上总计算数据分块个数...// 添加其他tiling字段TILING_DATA_FIELD_DEF_STRUCT(DeepNormTiling,deepnormTilingData);// 将DeepNormTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetDeepNormMaxMinTmpSize接口获取DeepNorm接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后根据输入shape、剩余的可供计算的空间大小等信息获取DeepNorm kernel侧接口所需tiling参数。12345678910111213141516171819202122232425262728293031namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){TilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_totalLength(totalLength);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...std:vector<int64_t>shapeVec={2,16,64};std:vector<int64_t>oriShapeVec={2,16,64};ge:ShapesrcShape(shapeVec);ge:ShapeoriginSrcShape(oriShapeVec);// 本样例中仅作为样例说明，通过GetDeepNormMaxMinTmpSize获取最小值并传入，来保证功能正确，开发者可以根据需要传入合适的空间大小uint32_tminValue=0;uint32_tmaxValue=0;AscendC:GetDeepNormMaxMinTmpSize(srcShape,sizeof(half),isReuseSrc,isBasicBlock,maxValue,minValue);// 获取DeepNorm Tiling参数AscendC:GetDeepNormTilingInfo(srcShape,originSrcShape,minValue,sizeof(half),isReuseSrc,isBasicBlock,tiling.deepnormTilingData);...// 其他逻辑tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的DeepNorm Tiling信息传入DeepNorm接口参与计算。完整的kernel侧样例请参考DeepNorm。123456789extern"C"__global____aicore__voiddeepnorm_custom(GM_ADDRinputX,GM_ADDRinputGx,GM_ADDRbeta,GM_ADDRgamma,GM_ADDRoutput,GM_ADDRoutputMean,GM_ADDRoutputVariance,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelDeepNormop;op.Init(inputX,inputGx,beta,gamma,output,outputMean,outputVariance,tilingData.totalLength,tilingData.tileNum,tilingData.deepnormTilingData);if(TILING_KEY_IS(1)){op.Process();}}
