# Erf-Erf接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0543
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0543.html
---

# Erf

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

按元素做误差函数计算（也称为高斯误差函数，error function or Gauss error function）。计算公式如下：

![](../images/atlasascendc_api_07_0544_img_001.png)

![](../images/atlasascendc_api_07_0544_img_002.png)

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErf(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErf(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)
- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErf(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErf(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetErfMaxMinTmpSize接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                            |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                          |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。临时空间大小BufferSize的获取方式请参考GetErfMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                            |

#### 返回值说明

无

#### 约束说明

- 不支持源操作数与目的操作数地址重叠。

- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859 | #include"kernel_operator.h"template<typenamesrcType>classKernelErf{public:__aicore__inlineKernelErf(){}__aicore__inlinevoidInit(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){srcGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(srcGm),srcSize);dstGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),srcSize);pipe.InitBuffer(inQueueX,1,srcSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,srcSize*sizeof(srcType));}__aicore__inlinevoidProcess(uint32_toffset,uint32_tcalSize){bufferSize=calSize;CopyIn(offset);Compute();CopyOut(offset);}private:__aicore__inlinevoidCopyIn(uint32_toffset){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,srcGlobal[offset],bufferSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:Erf<srcType,false>(dstLocal,srcLocal);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(uint32_toffset){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dstGlobal[offset],dstLocal,bufferSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>srcGlobal;AscendC:GlobalTensor<srcType>dstGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tbufferSize=0;};template<typenamedataType>__aicore__voidkernel_erf_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){KernelErf<dataType>op;op.Init(srcGm,dstGm,srcSize);op.Process();} |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

| 12  | 输入数据(srcLocal):[2.015634,-2.3880906,-0.2151161,...,-2.5,0.,2.5]输出数据(dstLocal):[0.99563545,-0.999268,-0.23903976,...,-0.9995931,0.,0.9995931] |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
