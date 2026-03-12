# ReGlu-ReGlu接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0790
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0790.html
---

# ReGlu

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

ReGlu是一种GLU变体，使用Relu作为激活函数，计算公式如下：

![](../images/atlasascendc_api_07_0790_img_001.png)

其中Relu激活函数的计算公式如下：

![](../images/atlasascendc_api_07_0790_img_002.png)

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidReGlu(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)

- 接口框架申请临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidReGlu(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1,constuint32_tcalCount)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetReGluMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、bfloat16_t、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、bfloat16_t、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                                                     |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                                 |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                               |
| srcTensor0      | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                     |
| srcTensor1      | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                     |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于ReGlu内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetReGluMaxMinTmpSize。 |
| calCount        | 输入      | 实际计算数据元素个数。                                                                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。
- 不支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 当前仅支持ND格式的输入，不支持其他格式。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081 | #include"kernel_operator.h"template<typenamesrcType>classKernelReGlu{public:__aicore__inlineKernelReGlu(){}__aicore__inlinevoidInit(GM_ADDRsrc0Gm,GM_ADDRsrc1Gm,GM_ADDRdstGm,uint32_tsrcSize){dataSize=srcSize;src0Global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src0Gm),dataSize);src1Global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src1Gm),dataSize);dstGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),dataSize);pipe.InitBuffer(inQueueX,1,dataSize*sizeof(srcType));pipe.InitBuffer(inQueueY,1,dataSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,dataSize*sizeof(srcType));if(sizeof(srcType)!=sizeof(float)){pipe.InitBuffer(calcBufs,dataSize*(sizeof(float)/sizeof(uint8_t))*3);}}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>src0Local=inQueueX.AllocTensor<srcType>();AscendC:LocalTensor<srcType>src1Local=inQueueY.AllocTensor<srcType>();AscendC:DataCopy(src0Local,src0Global,dataSize);AscendC:DataCopy(src1Local,src1Global,dataSize);inQueueX.EnQue(src0Local);inQueueY.EnQue(src1Local);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>src0Local=inQueueX.DeQue<srcType>();AscendC:LocalTensor<srcType>src1Local=inQueueY.DeQue<srcType>();AscendC:LocalTensor<uint8_t>tmpLocal;if(sizeof(srcType)!=sizeof(float)){tmpLocal=calcBufs.Get<uint8_t>();AscendC:ReGlu<srcType,false>(dstLocal,src0Local,src1Local,tmpLocal,dataSize);}else{AscendC:ReGlu<srcType,false>(dstLocal,src0Local,src1Local,dataSize);}outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(src0Local);inQueueY.FreeTensor(src1Local);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dstGlobal,dstLocal,dataSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src0Global;AscendC:GlobalTensor<srcType>src1Global;AscendC:GlobalTensor<srcType>dstGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueY;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;AscendC:TBuf<AscendC:TPosition:VECCALC>calcBufs;uint32_tdataSize=0;};template<typenamedataType>__aicore__voidkernel_reglu_operator(GM_ADDRsrc0Gm,GM_ADDRsrc1Gm,GM_ADDRdstGm,uint32_tsrcSize){KernelReGlu<dataType>op;op.Init(src0Gm,src1Gm,dstGm,srcSize);op.Process();} |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 123456789101112131415161718192021 | 输入数据(srcLocal0):[22.2812578.375-10.3515625-80.75-22.812584.375-8.9687570.5-51.7566.87569.81255.2734375-51.50.5-30.765625-52.1258.0312575.812550.4375-97.1875-80.687517.125-30.640625-13.67187592.37568.812553.755.105468839.6875-46.7187590.2567.75]输入数据(srcLocal1):[61.46875-36.5625-93.3125-87.6875-17.96875-88.125-46.65625-18.7812513.4921875-87.87565.75-25.96875-44.562553.-69.37596.5-24.70312577.562578.875-6.0898438-40.5625-69.62557.18.640625-73.87594.37591.5-9.710937584.12579.062588.596.3125]输出数据(dstLocal):[0.0000e+000.0000e+000.0000e+00-6.5450e+020.0000e+001.2544e+023.7880e+031.0519e+02-0.0000e+00-0.0000e+00-0.0000e+000.0000e+00-2.0110e+030.0000e+00-2.8020e+03-0.0000e+000.0000e+00-2.6120e+036.8840e+03-0.0000e+008.6550e+02-0.0000e+000.0000e+00-7.4120e+03-1.9700e+032.3140e+03-0.0000e+000.0000e+00-0.0000e+007.6760e+03-4.8828e-01-0.0000e+00] |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
