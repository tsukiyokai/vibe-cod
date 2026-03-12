# GroupNorm Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10003
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10003.html
---

# GroupNorm Tiling

#### 功能说明

GroupNorm Tiling API用于获取GroupNorm kernel计算时所需的Tiling参数。获取Tiling参数主要分为如下两步：

1. 通过GetGroupNormMaxMinTmpSize获取GroupNorm接口计算所需最大和最小临时空间大小。kernel侧GroupNorm接口的计算需要开发者预留/申请临时空间，GetGroupNormMaxMinTmpSize用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为Tiling参数传递到kernel侧使用。为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。
1. 通过GetGroupNormNDTilingInfo获取GroupNorm kernel侧接口所需tiling参数。GroupNorm Tiling结构体的定义如下，开发者无需关注该tiling结构的具体信息，只需要传递到kernel侧，传入GroupNorm高阶API接口，直接进行使用即可。1234567891011121314151617181920212223242526272829303132structGroupNormTiling{uint32_tn=0;uint32_tc=0;uint32_thw=0;uint32_tg=0;uint32_td=0;uint32_thwAlignSize=0;uint32_tdhwAlignSize=0;uint32_tinputXSize=0;uint32_tmeanVarSize=0;uint32_tnumberOfTmpBuf=0;uint32_tmeanTmpTensorPos=0;uint32_tmeanTmpTensorSize=0;uint32_tvarianceTmpTensorPos=0;uint32_tvarianceTmpTensorSize=0;uint32_ttmpBufSize=0;uint32_toneTmpSize=0;uint32_tfirstTmpStartPos=0;uint32_tsecondTmpStartPos=0;uint32_tthirdTmpStartPos=0;uint32_tloopRound=0;uint32_tinputRoundSize=0;uint32_tinputTailSize=0;uint32_tinputTailPos=0;uint32_tmeanVarRoundSize=0;uint32_tmeanVarTailSize=0;uint32_tmeanVarTailPos=0;uint32_tbshCurLength=0;uint32_tbsCurLength=0;floatfactor=0;boolsmallShape=0;};

#### 函数原型

| 1   | voidGetGroupNormMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource,constuint32_tgroupNum,uint32_t&maxValue,uint32_t&minValue) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |

| 1   | voidGetGroupNormNDTilingInfo(constge:Shape&srcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,constuint32_tgroupNum,optiling:GroupNormTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

| 1   | voidGetGroupNormNDTilingInfo(constge:Shape&srcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,constuint32_tgroupNum,AscendC:tiling:GroupNormTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 接口          | 输入/输出 | 功能                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape      | 输入      | 输入数据inputX的shape信息[N, C, H, W]。                                                                                                                                                                                                                                                                                                                                                                                                                          |
| typeSize      | 输入      | 输入数据inputX的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                                                                                |
| isReuseSource | 输入      | 中间变量是否能够复用输入内存。该参数预留，传入默认值false即可。                                                                                                                                                                                                                                                                                                                                                                                                  |
| groupNum      | 输入      | 在C维度上的分组数。                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| maxValue      | 输出      | 输出GroupNorm接口所需的tiling信息（最大临时空间大小）。GroupNorm接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue      | 输出      | 输出GroupNorm接口所需的tiling信息（最小临时空间大小）。GroupNorm接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。                                                                                                                                                                                                                                                                                                |

| 参数名称        | 输入/输出 | 含义                                                                    |
| --------------- | --------- | ----------------------------------------------------------------------- |
| srcShape        | 输入      | 输入数据inputX的shape信息[N, C, H, W]。                                 |
| stackBufferSize | 输入      | 可供GroupNorm接口使用的空间大小，单位Byte。                             |
| typeSize        | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。 |
| isReuseSource   | 输入      | 是否可以复用inputX的内存空间。                                          |
| groupNum        | 输入      | 在C维度上的分组数。                                                     |
| tiling          | 输出      | 输入数据的切分信息。                                                    |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

如下样例介绍了host侧获取Tiling参数的流程以及该参数如何在kernel侧使用。样例中输入Tensor的shape大小为[2，16，8, 8]，输入的数据类型为half。

1. 将GroupNormTiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。12345678910BEGIN_TILING_DATA_DEF(TilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,n);TILING_DATA_FIELD_DEF(uint32_t,c);TILING_DATA_FIELD_DEF(uint32_t,h);TILING_DATA_FIELD_DEF(uint32_t,w);TILING_DATA_FIELD_DEF(uint32_t,group);// 添加其他tiling字段...TILING_DATA_FIELD_DEF_STRUCT(GroupNormTiling,GroupNormTilingData);// 将GroupNormTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetGroupNormMaxMinTmpSize接口获取GroupNorm接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后根据输入shape、剩余的可供计算的空间大小等信息获取GroupNorm kernel侧接口所需tiling参数。123456789101112131415161718192021222324252627namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){TilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...std:vector<int64_t>shapeVec={2,16,8,8};// {n, c, h, w}ge:ShapesrcShape(shapeVec);uint32_tgroupNum=4uint32_tminSize=0;uint32_tmaxSize=0;// 本样例中仅作为样例说明，通过GetGroupNormMaxMinTmpSize接口获取GroupNorm接口能完成计算所需最大/最小临时空间大小，开发者可以根据该范围结合实际的内存使用情况设置合适的空间大小AscendC:GetGroupNormMaxMinTmpSize(srcShape,sizeof(half),false,groupNum,maxSize,minSize);// 获取GroupNorm Tiling参数AscendC:GetGroupNormNDTilingInfo(srcShape,maxSize,sizeof(half),false,groupNum,tiling.groupNormTilingData);...// 其他逻辑tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的GroupNorm Tiling信息传入GroupNorm接口参与计算。123456789extern"C"__global____aicore__voidgroupnorm_custom(GM_ADDRinputX_gm,GM_ADDRgamm_gm,GM_ADDRbeta_gm,GM_ADDRoutput_gm,GM_ADDRoutputMean_gm,GM_ADDRoutputVariance_gm,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelGroupNorm<half,false>op;op.Init(inputX_gm,gamm_gm,beta_gm,output_gm,outputMean_gm,outputVariance_gm,tilingData.groupNormTilingData);if(TILING_KEY_IS(1)){op.Process();}}
