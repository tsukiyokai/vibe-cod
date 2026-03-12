# Normalize Tiling-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0811
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0811.html
---

# Normalize Tiling

#### 功能说明

Ascend C提供Normalize Tiling API，方便用户获取Normalize kernel计算时所需的Tiling参数。

具体为，通过GetNormalizeMaxMinTmpSize获取Normalize接口计算所需最大和最小临时空间大小。

- 为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；
- 在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。

#### 函数原型

| 1   | voidGetNormalizeMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSizeU,constuint32_ttypeSizeT,constboolisReuseSource,constboolisComputeRstd,constboolisOnlyOutput,uint32_t&maxValue,uint32_t&minValue) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名        | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape      | 输入      | Normalize输入数据inputX的shape信息{A, R}。                                                                                                                                                                                                                                                                                                                                                                                                                       |
| typeSizeU     | 输入      | 输入数据gamma, beta的数据类型大小，单位为字节。比如输入的数据类型为float，此处应传入4。                                                                                                                                                                                                                                                                                                                                                                          |
| typeSizeT     | 输入      | 输入数据inputX的数据类型大小，单位为字节。比如输入的数据类型为float，此处应传入4。                                                                                                                                                                                                                                                                                                                                                                               |
| isReuseSource | 输入      | 是否复用源操作数的内存空间，与Normalize接口一致。                                                                                                                                                                                                                                                                                                                                                                                                                |
| isComputeRstd | 输入      | 是否计算rstd。该参数的取值只支持true。                                                                                                                                                                                                                                                                                                                                                                                                                           |
| isOnlyOutput  | 输入      | 是否只输出y，不输出标准差的倒数rstd。当前该参数仅支持取值为false，表示y和rstd的结果全部输出。                                                                                                                                                                                                                                                                                                                                                                    |
| maxValue      | 输出      | 输出Normalize接口所需的tiling信息（最大临时空间大小）。Normalize接口能完成计算所需的最大临时空间大小，超出该值的空间不会被该接口使用。在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue      | 输出      | 输出Normalize接口所需的tiling信息（最小临时空间大小）。Normalize接口能完成计算所需最小临时空间大小。为保证功能正确，接口计算时预留/申请的临时空间不能小于该数值。                                                                                                                                                                                                                                                                                                |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

1. 将Normalize接口所需参数增加至TilingData结构体，作为TilingData结构体的一个字段。12345678910BEGIN_TILING_DATA_DEF(NormalizeCustomTilingData)TILING_DATA_FIELD_DEF(float,epsilon);TILING_DATA_FIELD_DEF(uint32_t,isNoBeta);TILING_DATA_FIELD_DEF(uint32_t,isNoGamma);TILING_DATA_FIELD_DEF(uint32_t,isOnlyOutput);TILING_DATA_FIELD_DEF(uint32_t,aLength);TILING_DATA_FIELD_DEF(uint32_t,rLength);TILING_DATA_FIELD_DEF(uint32_t,rLengthWithPadding);...// 添加其他tiling字段END_TILING_DATA_DEF;
1. Tiling实现函数中，首先调用GetNormalizeMaxMinTmpSize接口获取Normalize接口能完成计算所需最大/最小临时空间大小，根据该范围结合实际的内存使用情况设置合适的空间大小，然后根据输入shape、剩余的可供计算的空间大小等信息获取Normalize kernel侧接口所需tiling参数。1234567891011121314151617181920212223242526272829303132namespaceoptiling{staticge:graphStatusTilingFunc(gert:TilingContext*context){NormalizeCustomTilingDatatiling;constgert:RuntimeAttrs*attrs=context->GetAttrs();constfloatepsilon=*(attrs->GetAttrPointer<float>(0));constuint32_tisNoBeta=*(attrs->GetAttrPointer<uint32_t>(1));constuint32_tisNoGamma=*(attrs->GetAttrPointer<uint32_t>(2));constuint32_tisOnlyOutput=*(attrs->GetAttrPointer<uint32_t>(3));constgert:StorageShape*x1_shape=context->GetInputShape(0);...// 其他逻辑constgert:Shapeshape=x1_shape->GetStorageShape();uint32_taLength=shape.GetDim(0);uint32_trLength=shape.GetDim(1);uint32_trLengthWithPadding=(rLength+alignNum-1)/alignNum*alignNum;std:vector<int64_t>srcDims={aLength,rLength};ge:ShapesrcShape(srcDims);uint32_tmaxTmpsize=0;uint32_tminTmpsize=0;AscendC:GetNormalizeMaxMinTmpSize(srcShape,typeSizeU,typeSizeT,false,true,isOnlyOutput,maxTmpsize,minTmpsize);...// 其他逻辑context->SetTilingKey(1);tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());size_t*currentWorkspace=context->GetWorkspaceSizes(1);currentWorkspace[0]=0;returnge:GRAPH_SUCCESS;}}// namespace optiling
1. 对应的kernel侧通过在核函数中调用GET_TILING_DATA获取TilingData，继而将TilingData中的Normalize Tiling信息传入Normalize接口参与计算。完整的kernel侧样例请参考Normalize。123456789101112131415161718192021222324extern"C"__global____aicore__voidnormalize_custom(GM_ADDRx,GM_ADDRmean,GM_ADDRvariance,GM_ADDRgamma,GM_ADDRbeta,GM_ADDRrstd,GM_ADDRy,GM_ADDRworkspace,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);floatepsilon=tilingData.epsilon;NormalizeParapara(tilingData.aLength,tilingData.rLength,tilingData.rLengthWithPadding);if(TILING_KEY_IS(1)){if(!tilingData.isNoBeta&&!tilingData.isNoGamma){KernelNormalize<NLCFG_NORM>op;op.Init(x,mean,variance,gamma,beta,rstd,y,epsilon,para);op.Process();}elseif(!tilingData.isNoBeta&&tilingData.isNoGamma){KernelNormalize<NLCFG_NOGAMMA>op;op.Init(x,mean,variance,gamma,beta,rstd,y,epsilon,para);op.Process();}elseif(tilingData.isNoBeta&&!tilingData.isNoGamma){KernelNormalize<NLCFG_NOBETA>op;op.Init(x,mean,variance,gamma,beta,rstd,y,epsilon,para);op.Process();}elseif(tilingData.isNoBeta&&tilingData.isNoGamma){KernelNormalize<NLCFG_NOOPT>op;op.Init(x,mean,variance,gamma,beta,rstd,y,epsilon,para);op.Process();}}}
