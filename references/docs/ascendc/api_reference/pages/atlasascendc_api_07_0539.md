# Frac-Frac接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0539
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0539.html
---

# Frac

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

按元素做取小数计算。计算公式如下：

![](../images/atlasascendc_api_07_0540_img_001.png)

举例如下：

Frac(-258.41888) = -0.41888428

Frac(5592.625) = 0.625

#### 函数原型

- 接口框架申请临时空间源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFrac(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFrac(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constuint32_tcalCount)

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFrac(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFrac(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

接口框架申请的方式，开发者需要预留临时空间；通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间。临时空间大小BufferSize的获取方式如下：通过GetFracMaxMinTmpSize中提供的GetFracMaxMinTmpSize接口获取需要预留空间大小的上下限。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                               |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                             |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                   |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于Frac内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetFracMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 针对Atlas推理系列产品AI Core，输入数据限制在[-2147483647.0, 2147483647.0]范围内。
- 不支持源操作数与目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566 | #include"kernel_operator.h"template<typenamesrcType>classKernelFrac{public:__aicore__inlineKernelFrac(){}__aicore__inlinevoidInit(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){srcGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(srcGm),srcSize);dstGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),srcSize);pipe.InitBuffer(inQueueX,1,srcSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,srcSize*sizeof(srcType));bufferSize=srcSize;}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,srcGlobal,bufferSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:Frac<srcType,false>(dstLocal,srcLocal);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dstGlobal,dstLocal,bufferSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>srcGlobal;AscendC:GlobalTensor<srcType>dstGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tbufferSize=0;};template<typenamedataType>__aicore__voidkernel_frac_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){KernelFrac<dataType>op;op.Init(srcGm,dstGm,srcSize);op.Process();}extern"C"__global____aicore__voidkernel_frac_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){kernel_frac_operator<half>(srcGm,dstGm,srcSize);} |
| --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12  | 输入数据(srcLocal):[-258.418885592.625-5312.416...9423.014-8336.825]输出数据(dstLocal):[-0.418884280.625-0.41601562...0.013671875-0.8251953] |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------- |
