# Log-Log接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0512
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0512.html
---

# Log

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

按元素以e、2、10为底做对数运算，计算公式如下：

![](../images/atlasascendc_api_07_0512_img_001.png)

![](../images/atlasascendc_api_07_0512_img_002.png)

![](../images/atlasascendc_api_07_0512_img_003.png)

![](../images/atlasascendc_api_07_0512_img_004.png)

![](../images/atlasascendc_api_07_0512_img_005.png)

![](../images/atlasascendc_api_07_0512_img_006.png)

#### 函数原型

- 以e为底：源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)
- 以2为底通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog2(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog2(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog2(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog2(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

- 以10为底：源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog10(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLog10(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetLogMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                            |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                          |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。临时空间大小BufferSize的获取方式请参考GetLogMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                            |

#### 返回值说明

无

#### 约束说明

- 不支持源操作数与目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

本样例中只展示Compute流程中的部分代码。如果您需要运行样例代码，请将该代码段拷贝并替换样例模板中Compute函数的部分代码即可。

- Log接口样例1Log(dstLocal,srcLocal);结果示例如下：12输入数据(srcLocal):[144.226079634.764...1835.12453145.5125]输出数据(dstLocal):[4.9713829.173133...7.5148688.053732]
- Log2接口样例1Log2(dstLocal,srcLocal);结果示例如下：12输入数据(srcLocal):[6299.54338.45963...2.8535255752.1323]输出数据(dstLocal):[12.6210318.40284...1.512745112.4898815]
- Log10接口样例1Log10(dstLocal,srcLocal);结果示例如下：12输入数据(srcLocal):[712.753578.36265...3099.05719313.082]输出数据(dstLocal):[2.85293941.8941091...3.49122953.9690933]

#### 样例模板

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667 | #include"kernel_operator.h"template<typenamesrcType>classKernelLog{public:__aicore__inlineKernelLog(){}__aicore__inlinevoidInit(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){src_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(srcGm),srcSize);dst_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),srcSize);pipe.InitBuffer(inQueueX,1,srcSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,srcSize*sizeof(srcType));bufferSize=srcSize;}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,src_global,bufferSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:Log(dstLocal,srcLocal);// 或可调用AscendC:Log10(dstLocal, srcLocal);// 或可调用AscendC:Log2(dstLocal, srcLocal);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dst_global,dstLocal,bufferSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src_global;AscendC:GlobalTensor<srcType>dst_global;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tbufferSize=0;};template<typenamedataType>__aicore__voidkernel_log_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){KernelLog<dataType>op;op.Init(srcGm,dstGm,srcSize);op.Process();}extern"C"__global____aicore__voidlog_operator_custom(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){kernel_log_operator<half>(srcGm,dstGm,srcSize);// 传入类型和大小} |
| ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
