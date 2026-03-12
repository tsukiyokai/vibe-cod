# FasterGelu-Gelu接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0772
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0772.html
---

# FasterGelu

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

在神经网络中，GELU是一个重要的激活函数，其灵感来源于relu和dropout，在激活中引入了随机正则的思想。为了降低GELU的算力需求，业界提出了FastGelu等版本。本接口FasterGelu是针对FastGelu的化简版本，公式化简可以大幅度提升计算性能。计算公式如下：

![](../images/atlasascendc_api_07_0772_img_001.png)

![](../images/atlasascendc_api_07_0772_img_002.png)

![](../images/atlasascendc_api_07_0772_img_003.png)

，化简后可得

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolhighPrecision=false,boolhighPerformance=false>__aicore__inlinevoidFasterGelu(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tdataSize)
- 接口框架申请临时空间12template<typenameT,boolhighPrecision=false,boolhighPerformance=false>__aicore__inlinevoidFasterGelu(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constuint32_tdataSize)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetGeluMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

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
- 操作数地址对齐要求请参见通用地址对齐约束。
- 当前仅支持ND格式的输入，不支持其他格式。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。

#### 调用示例

更多完整调用样例请参考fastergelu。

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364 | #include"kernel_operator.h"template<typenamesrcType>classKernelFasterGelu{public:__aicore__inlineKernelFasterGelu(){}__aicore__inlinevoidInit(GM_ADDRsrc_gm,GM_ADDRdst_gm,uint32_tinputSize){dataSize=inputSize;src_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src_gm),dataSize);dst_global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dst_gm),dataSize);pipe.InitBuffer(inQueueX,1,dataSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,dataSize*sizeof(srcType));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,src_global,dataSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();AscendC:FasterGelu(dstLocal,srcLocal,dataSize);// AscendC:FasterGelu<srcType, true, false>(dstLocal, srcLocal, dataSize);开启高精度模式// AscendC:FasterGelu<srcType, false, true>(dstLocal, srcLocal, dataSize);开启高性能模式outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dst_global,dstLocal,dataSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src_global;AscendC:GlobalTensor<srcType>dst_global;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tdataSize=0;};template<typenamedataType>__aicore__voidkernel_FasterGelu_operator(GM_ADDRsrc_gm,GM_ADDRdst_gm,uint32_tdataSize){KernelFasterGelu<dataType>op;op.Init(src_gm,dst_gm,dataSize);op.Process();} |
| ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12  | 输入数据(srcLocal):[-1.83887-3.607423.12891-0.6206052.0625-2.77344-0.04422-3.54297-3.162112.673831.3291-1.57617-0.01239013.77539-1.61621-0.616699]输出数据(dstLocal):[-0.0769653-0.007755283.11328-0.1600342.00195-0.0244446-0.021286-0.00849152-0.01446532.644531.20312-0.100769-0.006130223.76758-0.0969238-0.159912] |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
