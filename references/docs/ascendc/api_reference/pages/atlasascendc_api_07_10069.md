# Conv3D使用说明-Conv3D Kernel侧接口-Conv3D-卷积计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10069
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10069.html
---

# Conv3D使用说明

Ascend C提供一组Conv3D高阶API，方便用户快速实现3维卷积正向矩阵运算。3维正向卷积的示意图如图1，其计算公式为：

![](../images/atlasascendc_api_07_10005_img_001.png)

- X为Conv3D卷积的特征矩阵Input。
- W为Conv3D卷积的权重矩阵Weight。
- B为Conv3D卷积的偏置矩阵Bias。
- Y为完成卷积及偏置操作之后的结果矩阵Output。

![](../images/atlasascendc_api_07_10005_img_002.png)

![](../images/atlasascendc_api_07_10005_img_003.png)

Cin为Input的输入通道大小Channel；Din为Input的Depth维度大小；Hin为Input的Height维度大小；Win为Input的Width维度大小；Cout为Weight、Output的输出通道大小；Dout为Output的Depth维度的大小；Hout为Output的Height维度大小；Wout为Output的Width维度大小；下文中提及的M维度，为卷积正向操作过程中的输入Input在img2col展开后的纵轴，数值上等于Hout * Wout。

Channel、Depth、Height、Width后续简称为C、D、H、W。

除上述基础运算外，在Conv3D计算中可以设置参数Padding、Stride和Dilation，具体含义如下。

- Padding代表在输入矩阵的三个维度上填充0，见图2。
- Stride代表卷积核三个维度上滑动的距离，见图3。
- Dilation代表卷积核三个维度上每个数据的间距，见图4。

![](../images/atlasascendc_api_07_10005_img_004.png)

![](../images/atlasascendc_api_07_10005_img_005.png)

![](../images/atlasascendc_api_07_10005_img_006.png)

Kernel侧实现Conv3D运算的步骤概括为：

1. 创建Conv3D对象。
1. 初始化操作。
1. 设置3D卷积输入Input、Weight、Bias和输出Output。
1. 完成3D卷积操作。
1. 结束3D卷积操作。

使用Conv3D高阶API实现卷积正向的具体步骤如下：

1. 创建Conv3D对象。12345678#include"lib/conv/conv3d/conv3d_api.h"usinginputType=ConvApi:ConvType<AscendC:TPosition:GM,ConvFormat:NDC1HWC0,bfloat16_t>;usingweightType=ConvApi:ConvType<AscendC:TPosition:GM,ConvFormat:FRACTAL_Z_3D,bfloat16_t>;usingoutputType=ConvApi:ConvType<AscendC:TPosition:GM,ConvFormat:NDC1HWC0,bfloat16_t>;usingbiasType=ConvApi:ConvType<AscendC:TPosition:GM,ConvFormat:ND,float>;// 可选参数Conv3dApi:Conv3D<inputType,weightType,outputType,biasType>conv3dApi;创建对象时需要传入Input、Weight和Output参数类型信息；Bias的参数类型为可选参数，不带Bias输入的卷积计算场景，不传入该参数。类型信息通过ConvType来定义，包括：内存逻辑位置、数据格式、数据类型。123456template<TPositionPOSITION,ConvFormatFORMAT,typenameTYPE>structConvType{constexprstaticTPositionpos=POSITION;// Conv3d输入或输出在内存上的位置constexprstaticConvFormatformat=FORMAT;// Conv3d输入或者输出的数据格式usingT=TYPE;// Conv3d输入或输出的数据类型};下面简要介绍在创建对象时使用到的相关数据结构，开发者可选择性地了解这些内容。用于创建Conv3D对象的数据结构定义如下：12template<classINPUT_TYPE,classWEIGHT_TYPE,classOUTPUT_TYPE,classBIAS_TYPE=biasType,classCONV_CFG=Conv3dParam>usingConv3D=Conv3dIntfExt<Config<ConvApi:ConvDataType<INPUT_TYPE,WEIGHT_TYPE,OUTPUT_TYPE,BIAS_TYPE,CONV_CFG>>,Impl,Intf>其中，Conv3dIntfExt和Conv3dParam数据结构定义如下：123456789template<classConv3dCfg,template<typename,class,bool>classImpl=Conv3dApiImpl,template<class,template<typename,class,bool>class>classIntf=Conv3dIntf>structConv3dIntfExt:publicIntf<Conv3dCfg,Impl>{__aicore__inlineConv3dIntfExt(){}};structConv3dParam:publicConvApi:ConvParam{__aicore__inlineConv3dParam(){};};这里的Conv3dIntf是Conv3dIntfExt的基类，Conv3dCfg是Conv3dIntf模板入参，数据结构定义如下：123456789101112131415161718192021template<classConfig,template<typename,class,bool>classImpl>structConv3dIntf{usingInputT=typenameConfig:SrcAT;usingWeightT=typenameConfig:SrcBT;usingOutputT=typenameConfig:DstT;usingBiasT=typenameConfig:BiasT;usingL0cT=typenameConfig:L0cT;usingConvParam=typenameConfig:ConvParam;__aicore__inlineConv3dIntf(){}}template<classConvDataType>structConv3dCfg:publicConvApi:ConvConfig<ConvDataType>{public:__aicore__inlineConv3dCfg(){}usingContextData=struct_:publicConvApi:ConvConfig<ConvDataType>:ContextData{__aicore__inline_(){}};};表1ConvType说明参数说明TPosition内存逻辑位置。Input矩阵可设置为TPosition:GMWeight矩阵可设置为TPosition:GMBias矩阵可设置为TPosition:GMOutput矩阵可设置为TPosition:GMConvFormat数据格式。Input矩阵可设置为ConvFormat:NDC1HWC0Weight矩阵可设置为ConvFormat:FRACTAL_Z_3DBias矩阵可设置为ConvFormat:NDOutput矩阵可设置为ConvFormat:NDC1HWC0TYPE数据类型。Input矩阵可设置为half、bfloat16_tWeight矩阵可设置为half、bfloat16_tBias矩阵可设置为half、floatOutput矩阵可设置为half、bfloat16_t注意：输入输出的矩阵数据类型需要对应，具体支持的数据类型组合关系请参考表2。表2Conv3D输入输出数据类型的组合说明Input矩阵Weight矩阵BiasOutput矩阵支持平台halfhalfhalfhalfAtlas A3 训练系列产品/Atlas A3 推理系列产品Atlas A2 训练系列产品/Atlas A2 推理系列产品bfloat16_tbfloat16_tfloatbfloat16_tAtlas A3 训练系列产品/Atlas A3 推理系列产品Atlas A2 训练系列产品/Atlas A2 推理系列产品
1. 初始化操作。123Conv3dApi:Conv3D<inputType,weightType,outputType,biasType>conv3dApi;TPipepipe;// 初始化TPipeconv3dApi.Init(&tiling);// 初始化conv3dApi
1. 设置3D卷积的输入Input、Weight、Bias和输出Output。12345678910111213conv3dApi.SetWeight(weightGm);// 设置当前核的输入weight在gm上的地址if(biasFlag){conv3dApi.SetBias(biasGm);// 设置当前核的输入bias在gm上的地址}// 设置input各个维度在当前核的偏移conv3dApi.SetInputStartPosition(diStartPos,mStartPos);// 设置当前核的cout,dout,m大小conv3dApi.SetSingleOutputShape(singleCoreCout,singleCoreDout,singleCoreM);// 当前Conv3D仅支持单batch的卷积计算，多batch场景通过for循环实现，在循环间计算当前batch的地址偏移for(uint64_tbatchIter=0;batchIter<singleCoreBatch;++batchIter){conv3dApi.SetInput(inputGm[batchIter*inputOneBatchSize]);// 设置当前核的输入input在gm上的地址}
1. 完成3D卷积操作。调用IterateAll完成单核上所有数据的计算。12345for(uint64_tbatchIter=0;batchIter<singleCoreBatch;++batchIter){...conv3dApi.IterateAll(outputGm[batchIter*outputOneBatchSize]);// 调用IterateAll完成Conv3D计算...}
1. 结束3D卷积操作。1234for(uint64_tbatchIter=0;batchIter<singleCoreBatch;++batchIter){...conv3dApi.End();//清除EventID和释放内部申请的临时内存}

#### 需要包含的头文件

| 1   | #include"lib/conv/conv3d/conv3d_api.h" |
| --- | -------------------------------------- |
