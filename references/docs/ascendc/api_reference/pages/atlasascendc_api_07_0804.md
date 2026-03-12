# RmsNorm-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0804
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0804.html
---

# RmsNorm

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

实现对shape大小为[B，S，H]的输入数据的RmsNorm归一化，其计算公式如下：

![](../images/atlasascendc_api_07_0804_img_001.png)

其中，γ为缩放系数，ε为防除零的权重系数。

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolisBasicBlock=false>__aicore__inlinevoidRmsNorm(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constLocalTensor<T>&gammaLocal,constLocalTensor<uint8_t>&sharedTmpBuffer,constTepsilon,constRmsNormTiling&tiling)

- 接口框架申请临时空间12template<typenameT,boolisBasicBlock=false>__aicore__inlinevoidRmsNorm(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,constLocalTensor<T>&gammaLocal,constTepsilon,constRmsNormTiling&tiling)

由于该接口的内部实现中涉及复杂的计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持接口框架申请和开发者通过sharedTmpBuffer入参传入两种方式。

- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

接口框架申请的方式，开发者需要预留临时空间；通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间。临时空间大小BufferSize的获取方式如下：通过RmsNorm Tiling中提供的GetRmsNormMaxMinTmpSize接口获取所需最大和最小临时空间大小，最小空间可以保证功能正确，最大空间用于提升性能。

#### 参数说明

| 参数名       | 描述                                                                                                                                                                                                                                      |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T            | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。       |
| isBasicBlock | srcTensor和dstTensor的shape信息和Tiling切分策略满足基本块要求的情况下，可以使能该参数用于提升性能，默认不使能。基本块要求srcTensor和dstTensor的shape需要满足如下条件：last轴即H的长度为64的倍数，但小于2048；非last轴长度(B*S)为8的倍数。 |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                         |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstLocal        | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。dstLocal的shape和源操作数srcLocal需要保持一致。                                                        |
| srcLocal        | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。shape为[B, S, H]，尾轴H长度需要满足32字节对齐。                                                          |
| gammaLocal      | 输入      | 缩放系数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。shape需要与srcLocal和dstLocal的尾轴H长度相等，即shape为[H]。                                             |
| sharedTmpBuffer | 输入      | 临时空间。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。接口内部复杂计算时用于存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考RmsNorm Tiling。 |
| epsilon         | 输入      | 防除零的权重系数，数据类型需要与srcLocal/dstLocal保持一致。                                                                                                                  |
| tiling          | 输入      | RmsNorm计算所需Tiling信息，Tiling信息的获取请参考RmsNorm Tiling。                                                                                                            |

#### 返回值说明

无

#### 约束说明

- dstLocal和gammaLocal的Tensor空间不允许复用。
- 当前仅支持ND格式的输入，不支持其他格式。
- 操作数地址对齐要求请参见通用地址对齐约束。
- 当srcLocal的原始shape中H轴非32字节对齐时，开发者需要对原始输入在H轴方向补齐数据到32字节对齐，API的计算结果会覆盖dstLocal中对应srcLocal补齐位置的数据。

#### 调用示例

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374757677787980818283848586878889909192939495 | #include"kernel_operator.h"inline__aicore__uint32_tAlignToBlock(constuint32_tinputValue,constuint32_ttypeSize){constexpruint32_tONE_BLK_SIZE=32;uint32_talignUnit=ONE_BLK_SIZE/typeSize;return(inputValue+alignUnit-1)/alignUnit*alignUnit;}template<typenamedataType,boolisBasicBlock=false>classKernelRmsNorm{public:__aicore__inlineKernelRmsNorm(){}__aicore__inlinevoidInit(GM_ADDRinputGm,GM_ADDRgammaGm,GM_ADDRoutputGm,constRmsNormCustomTiling&customTiling){tiling=customTiling.tiling;constuint32_tbLength=tiling.bLength;constuint32_tsLength=tiling.sLength;hLength=tiling.hLength;bshLength=bLength*sLength*hLength;constexpruint32_ttypeSize=sizeof(dataType);constuint32_tbsLength=AlignToBlock(bLength*sLength,typeSize);constuint32_ttmpBufferSize=bshLength*2+bsLength;epsilon=customTiling.epsilon;inputGlobal.SetGlobalBuffer(reinterpret_cast<__gm__dataType*>(inputGm),bshLength);gammaGlobal.SetGlobalBuffer(reinterpret_cast<__gm__dataType*>(gammaGm),hLength);outputGlobal.SetGlobalBuffer(reinterpret_cast<__gm__dataType*>(outputGm),bshLength);pipe.InitBuffer(inQueue,1,bshLength*typeSize);pipe.InitBuffer(inQueueGamma,1,hLength*typeSize);pipe.InitBuffer(outQueue,1,bshLength*typeSize);pipe.InitBuffer(tmpQueue,1,tmpBufferSize);}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<dataType>inputLocal=inQueue.AllocTensor<dataType>();AscendC:DataCopy(inputLocal,inputGlobal,bshLength);inQueue.EnQue(inputLocal);AscendC:LocalTensor<dataType>gammaLocal=inQueueGamma.AllocTensor<dataType>();AscendC:DataCopy(gammaLocal,gammaGlobal,hLength);inQueueGamma.EnQue(gammaLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<dataType>inputLocal=inQueue.DeQue<dataType>();AscendC:LocalTensor<dataType>gammaLocal=inQueueGamma.DeQue<dataType>();AscendC:LocalTensor<dataType>outputLocal=outQueue.AllocTensor<dataType>();AscendC:LocalTensor<uint8_t>stackBuffer=tmpQueue.AllocTensor<uint8_t>();AscendC:RmsNorm<dataType,isBasicBlock>(outputLocal,inputLocal,gammaLocal,stackBuffer,epsilon,tiling);inQueue.FreeTensor(inputLocal);inQueueGamma.FreeTensor(gammaLocal);tmpQueue.FreeTensor(stackBuffer);outQueue.EnQue(outputLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<dataType>outputLocal=outQueue.DeQue<dataType>();AscendC:DataCopy(outputGlobal,outputLocal,bshLength);outQueue.FreeTensor(outputLocal);}private:AscendC:GlobalTensor<dataType>inputGlobal;AscendC:GlobalTensor<dataType>gammaGlobal;AscendC:GlobalTensor<dataType>outputGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueue;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueGamma;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;AscendC:TQue<AscendC:TPosition:VECCALC,1>tmpQueue;RmsNormTilingtiling;uint32_thLength;dataTypeepsilon;uint32_tbshLength;};template<typenamedataType,boolisBasicBlock=false>__aicore__inlinevoidkernel_rmsnorm_operator(GM_ADDRinputGm,GM_ADDRgammaGm,GM_ADDRoutputGm,GM_ADDRtiling){GET_TILING_DATA(customTilingData,tiling)KernelRmsNorm<dataType,isBasicBlock>op;op.Init(inputGm,gammaGm,outputGm,customTilingData);op.Process();} |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
