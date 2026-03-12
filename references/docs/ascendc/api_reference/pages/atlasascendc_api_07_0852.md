# UnPad Tiling-张量变换-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0852
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0852.html
---

# UnPad Tiling

#### 功能说明

用于获取UnPad Tiling参数。

#### 函数原型

| 1   | voidGetUnPadMaxMinTmpSize(constplatform_ascendc:PlatformAscendC&ascendcPlatform,constge:Shape&srcShape,constuint32_ttypeSize,uint32_t&maxValue,uint32_t&minValue) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | voidUnPadTilingFunc(constge:ShapesrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,optiling:UnPadTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------- |

| 1   | voidUnPadTilingFunc(constge:ShapesrcShape,constuint32_tstackBufferSize,constuint32_ttypeSize,AscendC:tiling:UnPadTiling&tiling) |
| --- | ------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名          | 输入/输出 | 含义                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ascendcPlatform | 输入      | 传入硬件平台的信息，PlatformAscendC定义请参见构造及析构函数。                                                                                                                                                                                                                                                                                                                                                                                                                         |
| srcShape        | 输入      | 输入Tensor的shape信息，shape为二维。                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| typeSize        | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。                                                                                                                                                                                                                                                                                                                                                                                                               |
| maxValue        | 输出      | UnPad接口能完成计算所需最大临时空间大小。UnPad接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。最大空间大小为0表示计算不需要临时空间。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue        | 输出      | UnPad接口能完成计算所需最小临时空间大小。Pad接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。最小空间大小为0表示计算不需要临时空间。                                                                                                                                                                                                                                                                                                  |

| 参数名          | 输入/输出 | 含义                                                                    |
| --------------- | --------- | ----------------------------------------------------------------------- |
| srcShape        | 输入      | 输入Tensor的shape信息，shape为二维。                                    |
| stackBufferSize | 输入      | 可供UnPad接口计算的空间大小。                                           |
| typeSize        | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。 |
| tiling          | 输出      | 输出UnPad接口所需的tiling信息。                                         |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

如下样例介绍了使用UnPad高阶API时host侧获取Tiling参数的流程以及该参数如何在kernel侧使用。样例中原始shape的大小为[320, 64]，需要unpad的目标shape大小为[320, 63]，输入的数据类型为half。

1. 将UnPadTiling结构体参数增加至TilingData结构体，作为TilingData结构体的一个字段。123456BEGIN_TILING_DATA_DEF(TilingData)// 注册一个tiling的类，以tiling的名字作为入参TILING_DATA_FIELD_DEF(uint32_t,totalLength);// 添加tiling字段，总计算数据量TILING_DATA_FIELD_DEF(uint32_t,tileNum);// 添加tiling字段，每个核上总计算数据分块个数...// 添加其他tiling字段TILING_DATA_FIELD_DEF_STRUCT(UnPadTiling,unpadTilingData);// 将UnPadTiling结构体参数增加至TilingData结构体END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetUnPadMaxMinTmpSize接口获取UnPad接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小；然后根据输入shape、剩余的可供计算的空间大小等信息获取UnPad kernel侧接口所需tiling参数。1234567891011121314151617181920212223242526272829namespaceoptiling{constuint32_tBLOCK_DIM=8;constuint32_tTILE_NUM=8;staticge:graphStatusTilingFunc(gert:TilingContext*context){TilingDatatiling;uint32_ttotalLength=context->GetInputTensor(0)->GetShapeSize();context->SetBlockDim(BLOCK_DIM);tiling.set_totalLength(totalLength);tiling.set_tileNum(TILE_NUM);// 设置其他Tiling参数...std:vector<int64_t>shapeVec={320,64};ge:ShapesrcShape(shapeVec);uint32_tmaxValue=0;uint32_tminValue=0;autoplatformInfo=context->GetPlatformInfo();autoascendcPlatform=platform_ascendc:PlatformAscendC(platformInfo);AscendC:GetUnPadMaxMinTmpSize(ascendcPlatform,srcShape,sizeof(half),maxValue,minValue);// 本样例中仅作为样例说明，获取最小值并传入，来保证功能正确，开发者可以根据需要传入合适的空间大小constuint32_tlocalWorkSpaceSize=minValue;AscendC:UnPadTilingFunc(srcShape,localWorkSpaceSize,sizeof(half),tiling.unpadTilingData);...tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的UnPad Tiling信息传入UnPad接口参与计算。完整的kernel侧样例请参考调用示例。123456789extern"C"__global____aicore__voidfunc_custom(GM_ADDRx,GM_ADDRy,GM_ADDRz,GM_ADDRworkspace,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelFuncop;op.Init(x,y,z,tilingData.totalLength,tilingData.tileNum,tilingData.unpadTilingData);if(TILING_KEY_IS(1)){op.Process();}}
