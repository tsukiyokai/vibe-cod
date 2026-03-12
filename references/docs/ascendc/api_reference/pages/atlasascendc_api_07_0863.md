# GetDropOutMaxMinTmpSize-数据过滤-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0863
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0863.html
---

# GetDropOutMaxMinTmpSize

#### 功能说明

用于获取DropOut Tiling参数。

#### 函数原型

| 1   | uint32_tGetDropOutMaxTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource) |
| --- | ------------------------------------------------------------------------------------------------- |

| 1   | uint32_tGetDropOutMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource) |
| --- | ------------------------------------------------------------------------------------------------- |

| 1   | voidGetDropOutMaxMinTmpSize(constge:Shape&srcShape,constuint32_ttypeSize,constboolisReuseSource,uint32_t&maxValue,uint32_t&minValue) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名        | 输入/输出 | 描述                                                                                                                                                                                                    |
| ------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape      | 输入      | 输入的shape信息。                                                                                                                                                                                       |
| typeSize      | 输入      | 计算的数据类型大小，half=2，float=4。                                                                                                                                                                   |
| isReuseSource | 输入      | 预留参数，暂未启用，保持默认值false即可。                                                                                                                                                               |
| maxValue      | 输出      | 输出DropOut接口所需的tiling信息（最大临时空间大小）。说明：maxValue仅作为参考值，有可能大于Unified Buffer剩余空间的大小，该场景下，开发者需要根据Unified Buffer剩余空间的大小来选取合适的临时空间大小。 |
| minValue      | 输出      | 输出DropOut接口所需的tiling信息（最小临时空间大小）。                                                                                                                                                   |

#### 返回值说明

GetDropOutMaxTmpSize返回DropOut接口能完成计算所需最大临时空间大小。

GetDropOutMinTmpSize返回DropOut接口能完成计算所需最小临时空间大小。

GetDropOutMaxMinTmpSize无返回值。

#### 约束说明

无

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849 | #include<vector>#include"register/op_def_registry.h"#include"register/tilingdata_base.h"#include"tiling/tiling_api.h"namespaceoptiling{BEGIN_TILING_DATA_DEF(DropoutCustomTilingData)TILING_DATA_FIELD_DEF(uint32_t,firstAxis);TILING_DATA_FIELD_DEF(uint32_t,srcLastAxis);TILING_DATA_FIELD_DEF(uint32_t,maskLastAxis);TILING_DATA_FIELD_DEF(uint32_t,tmpBufferSize);END_TILING_DATA_DEF;staticge:graphStatusTilingFunc(gert:TilingContext*context){// Input source shapes.int64_tfirstAxis=16;int64_tsrcLastAxis=64;int64_tmaskLastAxis=64;std:vector<int64_t>srcDims={firstAxis,srcLastAxis,maskLastAxis};uint32_ttypeSize=2;ge:Shapeshape(srcDims);uint32_tminValue=0;uint32_tmaxValue=0;AscendC:GetDropOutMaxMinTmpSize(shape,typeSize,false,maxValue,minValue);autoplatformInfo=context->GetPlatformInfo();autoascendcPlatform=platform_ascendc:PlatformAscendC(platformInfo);uint64_ttailSize=0;// ub剩余空间大小ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:UB,tailSize);// 本样例中使用完整的ub空间，实际情况下tailSize需要减掉用户已使用的ub空间autotmpSize=tailSize>=maxValue?maxValue:tailSize;DropoutCustomTilingDatatiling;tiling.set_firstAxis(firstAxis);tiling.set_srcLastAxis(srcLastAxis);tiling.set_maskLastAxis(maskLastAxis);tiling.set_tmpBufferSize(tmpSize);context->SetBlockDim(1);tiling.SaveToBuffer(context->GetRawTilingData()->GetData(),context->GetRawTilingData()->GetCapacity());context->GetRawTilingData()->SetDataSize(tiling.GetDataSize());context->SetTilingKey(1);returnge:GRAPH_SUCCESS;}}// namespace optiling |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
