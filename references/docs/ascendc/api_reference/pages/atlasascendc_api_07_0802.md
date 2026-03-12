# LayerNormGrad Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0802
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0802.html
---

# LayerNormGrad Tiling

#### 功能说明

LayerNormGrad Tiling的功能如下：

- 在host侧获取预留/申请的最大最小临时空间大小：kernel侧LayerNormGrad接口的计算需要开发者预留/申请临时空间，GetLayerNormGradMaxMinTmpSize接口用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小作为Tiling参数传递到kernel侧使用。为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。
- 通过GetLayerNormGradNDTilingInfo获取LayerNormGrad kernel侧接口所需tiling参数，需要传入输入shape，剩余的可供LayerNormGrad接口计算的空间大小和计算的数据类型。LayerNormGrad Tiling结构体的定义如下，开发者无需关注该Tiling结构的具体信息，只需要传递到kernel侧，传入LayerNormGrad高阶API接口，直接进行使用即可。1234567891011121314151617181920212223242526272829303132structLayerNormGradTiling{uint32_tstackBufferSize=0;uint32_tbLength=0;uint32_tsLength=0;uint32_thLength=0;uint32_toriginalHLength=0;uint32_toneCalSize=0;uint32_tnohCalSize=0;uint32_tloopNum=0;uint32_ttailSize=0;uint32_tnohTailSize=0;uint32_ttmpTensorBSHPos=0;uint32_ttmpTensorBSHSize=0;uint32_tpdVarTensorPos=0;uint32_tpdVarTensorSize=0;uint32_tpdMeanTensorPos=0;uint32_tpdMeanTensorSize=0;uint32_tx1TensorPos=0;uint32_tx1TensorSize=0;uint32_tx2TensorPos=0;uint32_tx2TensorSize=0;uint32_tx3TensorPos=0;uint32_tx3TensorSize=0;uint32_ttmpTensorPos=0;uint32_ttmpTensorSize=0;uint32_ttmpTensor1Pos=0;uint32_ttmpTensor1Size=0;uint32_ttmpTensor2Pos=0;uint32_ttmpTensor2Size=0;uint32_tlastDimValueBack=0;uint32_tlastDimValueBackMulTwo=0;};

#### 函数原型

| 1   | voidGetLayerNormGradMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource,uint32_t&maxValue,uint32_t&minValue) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------ |

| 1   | voidGetLayerNormGradNDTilingInfo(constge:ShapesrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,optiling:LayerNormGradTiling&tiling) |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | voidGetLayerNormGradNDTilingInfo(constge:ShapesrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,constboolisReuseSource,AscendC:tiling:LayerNormGradTiling&tiling) |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名称      | 输入/输出 | 含义                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape      | 输入      | 输入数据inputDy的shape信息{B, S, storageHLength, originHLength}，包括当前输入的inputDy的shape信息，以及地址对齐前（如存在H轴补齐操作）的原有shape信息。在API支持的场景下，storageHLength和originHLength保持一致。                                                                                                                                                                                                                                    |
| typeSize      | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                                                                              |
| isReuseSource | 输入      | 是否复用源操作数的内存空间，与LayerNorm接口一致。                                                                                                                                                                                                                                                                                                                                                                                                    |
| maxValue      | 输出      | LayerNormGrad接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。最大空间大小为0表示计算不需要临时空间。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue      | 输出      | LayerNormGrad接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。最小空间大小为0表示计算不需要临时空间。                                                                                                                                                                                                                                                                                                |

| 参数名称        | 输入/输出 | 含义                                                                               |
| --------------- | --------- | ---------------------------------------------------------------------------------- |
| srcShape        | 输入      | 输入数据inputDy的shape信息，包括当前输入的shape信息，以及地址对齐前的原有shape信息 |
| stackBufferSize | 输入      | 可供接口使用的空间大小，单位元素个数                                               |
| typeSize        | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。            |
| isReuseSource   | 输入      | 是否可以复用inputX和inputDy的内存空间                                              |
| tiling          | 输出      | 输入数据的切分信息                                                                 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

如下样例介绍了使用LayerNormGrad高阶API时host侧获取Tiling参数的流程以及该参数如何在kernel侧使用。样例中输入Tensor的shape大小为[2, 16, 64]，输入的数据类型为half。

1. 将LayerNormGradTiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。123456BEGIN_TILING_DATA_DEF(TilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,totalLength);// 添加tiling字段，总计算数据量TILING_DATA_FIELD_DEF(uint32_t,tileNum);// 添加tiling字段，每个核上总计算数据分块个数...// 添加其他tiling字段TILING_DATA_FIELD_DEF_STRUCT(LayerNormGradTiling,layernormGradTilingData);// 将LayerNormGradTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetLayerNormGradMaxMinTmpSize接口获取LayerNormGrad接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后调用GetLayerNormGradNDTilingInfo接口根据输入shape、剩余的可供计算的空间大小等信息获取LayerNormGradBeta kernel侧接口所需tiling参数。12345678910111213141516171819202122232425262728namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){TilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_totalLength(totalLength);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...// {B, S, storageHLength, originHLength}std:vector<int64_t>shapeVec={2,16,64,64};ge:ShapesrcShape(shapeVec);// 本样例中仅作为样例说明，通过GetLayerNormGradMaxMinTmpSize获取最小值并传入，来保证功能正确，开发者可以根据需要传入合适的空间大小uint32_tmax;uint32_tmin;AscendC:GetLayerNormGradMaxMinTmpSize(srcShape,sizeof(half),false,max,min);// 获取LayerNormGrad Tiling参数AscendC:GetLayerNormGradNDTilingInfo(srcShape,min,sizeof(half),false,tiling.layernormGradTilingData);...// 其他逻辑tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的LayerNormGradTiling信息传入LayerNormGrad接口参与计算。完整的kernel侧样例请参考调用示例。123456789extern"C"__global____aicore__voidfunc_custom(GM_ADDRx,GM_ADDRy,GM_ADDRz,GM_ADDRworkspace,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelFuncop;op.Init(x,y,z,tilingData.totalLength,tilingData.tileNum,tilingData.layernormGradTilingData);if(TILING_KEY_IS(1)){op.Process();}}
