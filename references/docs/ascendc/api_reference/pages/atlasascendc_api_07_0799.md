# LayerNormGradBeta-归一化操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0799
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0799.html
---

# LayerNormGradBeta

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

LayerNormGradBeta接口用于获取反向beta/gamma的数值，和LayerNormGrad共同输出pdx, gamma和beta：

算法公式为：

![](../images/atlasascendc_api_07_0799_img_001.png)

![](../images/atlasascendc_api_07_0799_img_002.png)

#### 函数原型

由于该接口的内部实现中涉及复杂的计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间大小BufferSize的获取方法：通过LayerNormGradBeta Tiling中提供的GetLayerNormGradBetaMaxMinTmpSize接口获取所需最大和最小临时空间大小，最小空间可以保证功能正确，最大空间用于提升性能。

临时空间支持接口框架申请和开发者通过sharedTmpBuffer入参传入两种方式，因此LayerNormGradBeta接口的函数原型有两种：

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLayerNormGradBeta(constLocalTensor<T>&outputPdGamma,constLocalTensor<T>&outputPdBeta,constLocalTensor<T>&resForGamma,constLocalTensor<T>&inputDy,constLocalTensor<uint8_t>&sharedTmpBuffer,constLayerNormGradBetaTiling&tiling)该方式下开发者需自行申请并管理临时内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

- 接口框架申请临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidLayerNormGradBeta(constLocalTensor<T>&outputPdGamma,constLocalTensor<T>&outputPdBeta,constLocalTensor<T>&resForGamma,constLocalTensor<T>&inputDy,LayerNormGradBetaTiling&tiling)该方式下开发者无需申请，但是需要预留临时空间的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                                                                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。                                                                                                                      |
| isReuseSource | 是否允许修改源操作数，默认值为false。如果开发者允许源操作数被改写，可以使能该参数，使能后能够节省部分内存空间。设置为true，则本接口内部计算时复用inputDy的内存空间，节省内存空间；设置为false，则本接口内部计算时不复用inputDy的内存空间。对于float数据类型输入支持开启该参数，half数据类型输入不支持开启该参数。isReuseSource的使用样例请参考更多样例。 |

| 参数名称        | 输入/输出 | 含义                                                                                                                                                                                                                                                                                                       |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| outputPdGamma   | 输出      | 目的操作数，shape为[H]，LocalTensor数据结构的定义请参考LocalTensor。尾轴长度需要32B对齐类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                                                                          |
| outputPdBeta    | 输出      | 目的操作数，shape为[H]，LocalTensor数据结构的定义请参考LocalTensor。尾轴长度需要32B对齐类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                                                                          |
| resForGamma     | 输入      | 源操作数，shape为[B, S, H]，LocalTensor数据结构的定义请参考LocalTensor。resForGamma的数据类型需要与目的操作数保持一致，尾轴长度需要32B对齐。需提前调用LayerNormGrad接口获取resForGamma参数值。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                   |
| inputDy         | 输入      | 源操作数，shape为[B, S, H]，LocalTensor数据结构的定义请参考LocalTensor。inputDy的数据类型需要与目的操作数保持一致，尾轴长度需要32B对齐。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                         |
| sharedTmpBuffer | 输入      | 共享缓冲区，用于存放API内部计算产生的临时数据。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。共享缓冲区大小的获取方式请参考LayerNormGradBeta Tiling。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。 |
| tiling          | 输入      | LayerNormGradBeta计算所需Tiling信息，Tiling信息的获取请参考LayerNormGradBeta Tiling。                                                                                                                                                                                                                      |

#### 返回值说明

无

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。
- 源操作数和目的操作数的Tensor空间可以复用。
- 仅支持输入shape为ND格式。
- 输入数据不满足对齐要求时，开发者需要进行补齐，补齐的数据应设置为0，防止出现异常值从而影响网络计算。
- 不支持对尾轴H轴的切分。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485868788899091 | #include"kernel_operator.h"template<typenameT,boolisReuseSource=false>classKernelLayernormGradBeta{public:__aicore__inlineKernelLayernormGradBeta(){}__aicore__inlinevoidInit(__gm__uint8_t*resForGammaGm,__gm__uint8_t*inputDyGm,__gm__uint8_t*outputPdGammaGm,__gm__uint8_t*outputPdBetaGm,constLayerNormGradBetaTiling&tiling){this->bLength=tiling.bLength;this->sLength=tiling.sLength;this->hLength=tiling.hLength;this->tiling=tiling;bshLength=bLength*sLength*hLength;bsLength=bLength*sLength;resForGammaGlobal.SetGlobalBuffer(reinterpret_cast<__gm__T*>(resForGammaGm),bshLength);inputDyGlobal.SetGlobalBuffer(reinterpret_cast<__gm__T*>(inputDyGm),bshLength);outputPdGammaGlobal.SetGlobalBuffer(reinterpret_cast<__gm__T*>(outputPdGammaGm),hLength);outputPdBetaGlobal.SetGlobalBuffer(reinterpret_cast<__gm__T*>(outputPdBetaGm),hLength);pipe.InitBuffer(inQueueResForGamma,1,sizeof(T)*bshLength);pipe.InitBuffer(inQueueDy,1,sizeof(T)*bshLength);pipe.InitBuffer(outQueuePdGamma,1,sizeof(T)*hLength);pipe.InitBuffer(outQueuePdBeta,1,sizeof(T)*hLength);}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<T>resForGammaLocal=inQueueResForGamma.AllocTensor<T>();AscendC:LocalTensor<T>inputDyLocal=inQueueDy.AllocTensor<T>();AscendC:DataCopy(resForGammaLocal,resForGammaGlobal,bshLength);AscendC:DataCopy(inputDyLocal,inputDyGlobal,bshLength);inQueueResForGamma.EnQue(resForGammaLocal);inQueueDy.EnQue(inputDyLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<T>resForGammaLocal=inQueueResForGamma.DeQue<T>();AscendC:LocalTensor<T>inputDyLocal=inQueueDy.DeQue<T>();AscendC:LocalTensor<T>outputPdGammaLocal=outQueuePdGamma.AllocTensor<T>();AscendC:LocalTensor<T>outputPdBetaLocal=outQueuePdBeta.AllocTensor<T>();AscendC:LayerNormGradBeta<T,isReuseSource>(outputPdGammaLocal,outputPdBetaLocal,resForGammaLocal,inputDyLocal,tiling);outQueuePdGamma.EnQue<T>(outputPdGammaLocal);outQueuePdBeta.EnQue<T>(outputPdBetaLocal);inQueueResForGamma.FreeTensor(resForGammaLocal);inQueueDy.FreeTensor(inputDyLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<T>outputPdGammaLocal=outQueuePdGamma.DeQue<T>();AscendC:LocalTensor<T>outputPdBetaLocal=outQueuePdBeta.DeQue<T>();AscendC:DataCopy(outputPdGammaGlobal,outputPdGammaLocal,hLength);AscendC:DataCopy(outputPdBetaGlobal,outputPdBetaLocal,hLength);outQueuePdGamma.FreeTensor(outputPdGammaLocal);outQueuePdBeta.FreeTensor(outputPdBetaLocal);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueResForGamma,inQueueDy;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueuePdGamma,outQueuePdBeta;AscendC:GlobalTensor<T>resForGammaGlobal;AscendC:GlobalTensor<T>inputDyGlobal;AscendC:GlobalTensor<T>outputPdGammaGlobal;AscendC:GlobalTensor<T>outputPdBetaGlobal;uint32_tbLength;uint32_tsLength;uint32_thLength;uint32_tbshLength;uint32_tbsLength;LayerNormGradBetaTilingtiling;};extern"C"__global____aicore__voidkernel_layernorm_grad_beta_operator(GM_ADDRoutputPdGammaGm,GM_ADDRoutputPdBetaGm,GM_ADDRresForGammaGm,GM_ADDRinputDyGm,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelLayernormGradBeta<half,false>op;op.Init(resForGammaGm,inputDyGm,outputPdGammaGm,outputPdBetaGm,tilingData.layerNormGradBetaTiling);op.Process();} |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
