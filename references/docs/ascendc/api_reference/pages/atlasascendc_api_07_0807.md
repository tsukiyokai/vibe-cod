# BatchNorm Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0807
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0807.html
---

# BatchNorm Tiling

#### 功能说明

BatchNorm Tiling API用于获取BatchNorm kernel计算时所需的Tiling参数。获取Tiling参数主要分为如下两步：

1. 通过GetBatchNormMaxMinTmpSize获取BatchNorm接口计算所需最大和最小临时空间大小。kernel侧BatchNorm接口的计算需要开发者预留/申请临时空间，GetBatchNormMaxMinTmpSize用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为Tiling参数传递到kernel侧使用。为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。
1. 通过GetBatchNormNDTilingInfo获取BatchNorm kernel侧接口所需tiling参数。BatchNorm Tiling结构体的定义如下，开发者无需关注该tiling结构的具体信息，只需要传递到kernel侧，传入BatchNorm高阶API接口，直接进行使用即可。12345678910111213141516171819202122structBatchNormTiling{uint32_toriginalBLength=0;uint32_tmeanVarSize=0;uint32_tmeanTmpTensorPos=0;uint32_tvarianceTmpTensorPos=0;uint32_ttmpBufSize=0;uint32_toneTmpSize=0;uint32_tfirstTmpStartPos=0;uint32_tsecondTmpStartPos=0;uint32_tthirdTmpStartPos=0;uint32_tloopRound=0;uint32_tinputTailSize=0;uint32_tinputTailPos=0;uint32_tmeanVarTailSize=0;uint32_tmeanVarTailPos=0;uint32_tbshCurLength=0;uint32_tshCurLength=0;floatfirstDimValueBack=0;uint32_tcastHalfRepStride=0;uint32_tshCurLengthBlockNum=0;uint32_tcastHalfOutRepStride=0;};

#### 函数原型

| 1   | boolGetBatchNormMaxMinTmpSize(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_ttypeSize,constboolisReuseSource,uint32_t&maxValue,uint32_t&minValue,constboolisBasicBlock=false) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetBatchNormNDTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferByteSize,constuint32_ttypeSize,constboolisReuseSource,optiling:BatchNormTiling&tilling,constboolisBasicBlock=false) |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | boolGetBatchNormNDTilingInfo(constge:Shape&srcShape,constge:Shape&originSrcShape,constuint32_tstackBufferByteSize,constuint32_ttypeSize,constboolisReuseSource,AscendC:tiling:BatchNormTiling&tilling,constboolisBasicBlock=false) |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名         | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| srcShape       | 输入      | 输入数据inputX的shape信息[B, S, H]，S*H需要32B对齐。                                                                                                                                                                                                                                                                                                                                             |
| originSrcShape | 输入      | 输入数据inputX的originshape信息[originB, originS, originH]。                                                                                                                                                                                                                                                                                                                                     |
| typeSize       | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                          |
| isReuseSource  | 输入      | 中间变量是否能够复用输入内存。该参数预留，传入默认值false即可。                                                                                                                                                                                                                                                                                                                                  |
| maxValue       | 输出      | BatchNorm接口能完成计算所需的最大临时空间大小，超出max的空间不会被该接口使用。在min-max范围内，预留/申请空间越大，接口计算性能越好。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。maxValue为0表示计算不需要临时空间。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue       | 输出      | BatchNorm接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于min的数值。最小空间为0表示计算不需要临时空间。                                                                                                                                                                                                                                                 |
| isBasicBlock   | 输入      | 是否使能基本块，与BatchNorm接口一致。                                                                                                                                                                                                                                                                                                                                                            |

表2GetBatchNormNDTilingInfo接口参数列表：

| 参数名称            | 输入/输出 | 含义                                                            |
| ------------------- | --------- | --------------------------------------------------------------- |
| srcShape            | 输入      | 输入数据inputX的shape信息[B, S, H]，S*H需要32B对齐。            |
| originSrcShape      | 输入      | 输入数据inputX的originshape信息[originB, originS, originH]。    |
| stackBufferByteSize | 输入      | 可供BatchNorm接口使用的空间大小，单位Byte。                     |
| typeSize            | 输入      | 输入数据类型的字节大小。                                        |
| isReuseSource       | 输入      | 中间变量是否能够复用输入内存。该参数预留，传入默认值false即可。 |
| tilling             | 输出      | 输入数据的切分信息。                                            |
| isBasicBlock        | 输入      | 是否使能基本块，与BatchNorm接口一致。                           |

#### 返回值说明

- GetBatchNormMaxMinTmpSize返回值为true/false，true表示成功拿到BatchNorm接口内部计算需要的最大和最小临时空间大小；false表示获取失败。
- GetBatchNormNDTilingInfo返回类型为true/false，true表示成功拿到BatchNorm的Tiling各项参数值；false表示获取失败。

#### 约束说明

无

#### 调用示例

如下样例介绍了host侧获取Tiling参数的流程以及该参数如何在kernel侧使用。样例中输入Tensor的shape大小为[16，16，16]，输入的数据类型为half。

1. 将BatchNormTiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。123456789BEGIN_TILING_DATA_DEF(TilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,tileNum);// 添加tiling字段，每个核上总计算数据分块个数TILING_DATA_FIELD_DEF(uint32_t,bLength);// 添加tiling字段，输入shape的b维度长度TILING_DATA_FIELD_DEF(uint32_t,sLength);// 添加tiling字段，输入shape的s维度长度TILING_DATA_FIELD_DEF(uint32_t,hLength);// 添加tiling字段，输入shape的h维度长度TILING_DATA_FIELD_DEF(uint32_t,originalBLength);// 添加tiling字段，输入shape原始b维度长度...// 添加其他tiling字段TILING_DATA_FIELD_DEF_STRUCT(BatchNormTiling,batchNormTilingData);// 将BatchNormTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetBatchNormMaxMinTmpSize接口获取BatchNorm接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后根据输入shape、剩余的可供计算的空间大小等信息获取BatchNorm kernel侧接口所需tiling参数。12345678910111213141516171819202122232425262728namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){TilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...std:vector<int64_t>shapeVec={16,16,16};//{b,s,h}std:vector<int64_t>originShapeVec={15,16,16};//{originB,originS,originH}ge:ShapesrcShape(shapeVec);ge:ShapeoriginSrcShape(originShapeVec);uint32_tminSize=0;uint32_tmaxSize=0;// 本样例中仅作为样例说明，通过GetBatchNormMaxMinTmpSize获取最小值并传入，来保证功能正确，开发者可以根据需要传入合适的空间大小AscendC:GetBatchNormMaxMinTmpSize(srcShape,originSrcShape,sizeof(half),false,maxSize,minSize,false);// 获取BatchNorm Tiling参数AscendC:GetBatchNormNDTilingInfo(srcShape,originSrcShape,minSize,sizeof(half),false,tiling.batchNormTilingData,false);...// 其他逻辑tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的BatchNormTiling信息传入BatchNorm接口参与计算。完整的kernel侧样例请参考BatchNorm123456789extern"C"__global____aicore__voidfunc_custom(GM_ADDRinputX_gm,GM_ADDRgamm_gm,GM_ADDRbeta_gm,GM_ADDRoutput_gm,GM_ADDRoutputMean_gm,GM_ADDRoutputVariance_gm,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelBatchnorm<half,false,false>op;op.Init(inputX_gm,gamm_gm,beta_gm,output_gm,outputMean_gm,outputVariance_gm,tilingData.batchNormTilingData);if(TILING_KEY_IS(1)){op.Process();}}
