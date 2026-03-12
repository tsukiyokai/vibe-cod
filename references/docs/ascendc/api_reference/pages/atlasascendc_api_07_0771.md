# Gelu-Gelu接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0771
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0771.html
---

# Gelu

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

在神经网络中，GELU是一个重要的激活函数，其灵感来源于Relu和Dropout，在激活中引入了随机正则的思想。计算公式如下：

![](../images/atlasascendc_api_07_0771_img_001.png)

![](../images/atlasascendc_api_07_0771_img_002.png)

![](../images/atlasascendc_api_07_0771_img_003.png)

，化简后可得

#### 函数原型

- 接口框架申请临时空间12template<typenameT,boolhighPrecision=false,boolhighPerformance=false>__aicore__inlinevoidGelu(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constuint32_tdataSize)

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolhighPrecision=false,boolhighPerformance=false>__aicore__inlinevoidGelu(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tdataSize)

#### 参数说明

| 参数名          | 描述                                                                                                                                                                                                                                |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T               | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| highPrecision   | 是否使能高精度模式，以提升运算准确度。默认值为false，表示不使能高精度模式。注意：高精度模式只在half数据类型下使能后生效，该参数的取值不影响float数据类型下的接口精度和性能。                                                        |
| highPerformance | 是否使能高性能模式，以提升运算效率。默认值为false，表示不使能高性能模式。注意：开启高性能模式相比于默认不开启高精度和高性能模式会有精度下降，同时开启高精度和高性能模式相比于仅开启高性能模式可能会有性能下降。                     |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                               |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstLocal        | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                             |
| srcLocal        | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                   |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于接口内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetGeluMaxMinTmpSize。 |
| dataSize        | 输入      | 实际计算数据元素个数。                                                                                                                                                             |

#### 返回值说明

无

#### 约束说明

- 源操作数和目的操作数的Tensor空间可以复用。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。
- 仅支持输入shape为ND格式。

#### 调用示例

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162 | #include"kernel_operator.h"template<typenamesrcType>classKernelGelu{public:__aicore__inlineKernelGelu(){}__aicore__inlinevoidInit(GM_ADDRsrc_gm,GM_ADDRdst_gm,uint32_tinputSize){dataSize=inputSize;src_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src_gm),dataSize);dst_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dst_gm),dataSize);pipe.InitBuffer(inQueueX,1,dataSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,dataSize*sizeof(srcType));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,src_global,dataSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:Gelu(dstLocal,srcLocal,dataSize);// AscendC:Gelu<srcType, true, false>(dstLocal, srcLocal, dataSize);// AscendC:Gelu<srcType, false, true>(dstLocal, srcLocal, dataSize);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dst_global,dstLocal,dataSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src_global;AscendC:GlobalTensor<srcType>dst_global;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tdataSize=0;};template<typenamedataType>__aicore__voidkernel_Gelu_operator(GM_ADDRsrc_gm,GM_ADDRdst_gm,uint32_tdataSize){KernelGelu<dataType>op;op.Init(src_gm,dst_gm,dataSize);op.Process();} |
| ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12345678910 | 输入数据(srcLocal):[-1.2511.074-6.137-9.67-5.066-9.44-3.588-5.758-7.484-5.35-9.62-4.33-6.66-3.7320.0841-8.59-6.3-4.62-3.059-8.34-8.24-7.617-7.93-3.592-3.268-5.406-9.495.633-5.3-9.36-6.715-5.727]输出数据(dstLocal):[-0.14110.916-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.0.0486-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.-0.5.633-0.-0.-0.-0.] |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
