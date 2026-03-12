# Trunc-Trunc接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0535
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0535.html
---

# Trunc

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

按元素做浮点数截断，即向零取整操作。计算公式如下：

![](../images/atlasascendc_api_07_0536_img_001.png)

举例如下：

Trunc(3.9) = 3

Trunc(-3.9) = -3

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidTrunc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidTrunc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)

- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidTrunc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidTrunc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

由于该接口的内部实现中涉及精度转换。需要额外的临时空间来存储计算过程中的中间变量。临时空间支持接口框架申请和开发者通过sharedTmpBuffer入参传入两种方式。

- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

接口框架申请的方式，开发者需要预留临时空间；通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间。临时空间大小BufferSize的获取方式如下：通过GetTruncMaxMinTmpSize中提供的GetTruncMaxMinTmpSize接口获取需要预留空间的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                                 |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                               |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                                 |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于Trunc内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetTruncMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                                                                                 |

#### 返回值说明

无

#### 约束说明

- 针对Atlas推理系列产品AI Core，输入数据限制在[-2147483647.0, 2147483647.0]范围内。
- 支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364 | #include"kernel_operator.h"template<typenamesrcType>classKernelTrunc{public:__aicore__inlineKernelTrunc(){}__aicore__inlinevoidInit(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){src_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(srcGm),srcSize);dst_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),srcSize);pipe.InitBuffer(inQueueX,1,srcSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,srcSize*sizeof(srcType));bufferSize=srcSize;}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,src_global,bufferSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:Trunc<srcType,false>(dstLocal,srcLocal);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dst_global,dstLocal,bufferSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src_global;AscendC:GlobalTensor<srcType>dst_global;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tbufferSize=0;};template<typenamedataType>__aicore__voidkernel_trunc_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){KernelTrunc<dataType>op;op.Init(srcGm,dstGm,srcSize);op.Process();}extern"C"__global____aicore__voidtrunc_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tsrcSize){kernel_trunc_operator<half>(srcGm,dstGm,srcSize);} |
| ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12  | 输入数据(srcLocal):[0.5317103-6.379120325.53408647...11.11059642-11.67860335]输出数据(dstLocal):[0.0-6.05.0...11.0-11.0] |
| --- | ------------------------------------------------------------------------------------------------------------------------ |
