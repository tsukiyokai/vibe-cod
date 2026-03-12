# GeGLU-GeGLU接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0785
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0785.html
---

# GeGLU

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

GeGLU是采用GELU作为激活函数的GLU变体。具体计算公式如下：

![](../images/atlasascendc_api_07_0786_img_001.png)

其中GELU激活函数的计算公式如下：

![](../images/atlasascendc_api_07_0786_img_002.png)

![](../images/atlasascendc_api_07_0786_img_003.png)

上述公式中的erf为误差函数：

![](../images/atlasascendc_api_07_0786_img_004.png)

误差函数没有解析表达式，按照业界普遍使用的tanh近似表达式：

将GELU近似公式代入可得GeGLU表达式为：

![](../images/atlasascendc_api_07_0786_img_005.png)

其中a=-0.0713548162726,b=2.2363860002236e1，x1和x0代表srcTensor1和srcTensor0中的元素。

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidGeGLU(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1,constLocalTensor<uint8_t>&sharedTmpBuffer,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidGeGLU(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1,constLocalTensor<uint8_t>&sharedTmpBuffer)

- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidGeGLU(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1,uint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidGeGLU(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor0,constLocalTensor<T>&srcTensor1)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetGeGLUMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名                | 输入/输出 | 描述                                                                                                                                                                                 |
| --------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dstTensor             | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                               |
| srcTensor0/srcTensor1 | 输入      | 源操作数。源操作数的数据类型需要与目的操作数保持一致。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                     |
| sharedTmpBuffer       | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于GeGLU内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetGeGLUMaxMinTmpSize。 |
| calCount              | 输入      | 实际计算数据元素个数。                                                                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。
- 不支持源操作数与目的操作数地址重叠。
- 当前仅支持ND格式的输入，不支持其他格式。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485868788 | #include"kernel_operator.h"template<typenamesrcType>classKernelGeGLU{public:__aicore__inlineKernelGeGLU(){}__aicore__inlinevoidInit(GM_ADDRsrc0Gm,GM_ADDRsrc1Gm,GM_ADDRdstGm,uint32_tinputSize,uint32_ttmpBufSize){dataSize=inputSize;uint32_tbufSize=4*tmpBufSize;src0Global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src0Gm),dataSize);src1Global.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(src1Gm),dataSize);dstGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),dataSize);pipe.InitBuffer(inQueue0,1,dataSize*sizeof(srcType));pipe.InitBuffer(inQueue1,1,dataSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,dataSize*sizeof(srcType));if((sizeof(srcType)==sizeof(half))&&(tmpBufSize>0)){pipe.InitBuffer(buf,bufSize*sizeof(srcType));}}__aicore__inlinevoidProcess(uint32_ttmpBufSize,uint32_tcalCount){CopyIn();Compute(tmpBufSize,calCount);CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>src0Local=inQueue0.AllocTensor<srcType>();AscendC:LocalTensor<srcType>src1Local=inQueue1.AllocTensor<srcType>();AscendC:DataCopy(src0Local,src0Global,dataSize);AscendC:DataCopy(src1Local,src1Global,dataSize);inQueue0.EnQue(src0Local);inQueue1.EnQue(src1Local);}__aicore__inlinevoidCompute(uint32_ttmpBufSize,uint32_tcalCount){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>src0Local=inQueue0.DeQue<srcType>();AscendC:LocalTensor<srcType>src1Local=inQueue1.DeQue<srcType>();AscendC:LocalTensor<uint8_t>temp;if((sizeof(srcType)==sizeof(half))&&(tmpBufSize>0)){temp=buf.Get<uint8_t>();}if((tmpBufSize>0)&&(calCount>0)){AscendC:GeGLU<srcType,false>(dstLocal,src0Local,src1Local,temp,calCount);}elseif(tmpBufSize>0){AscendC:GeGLU<srcType,false>(dstLocal,src0Local,src1Local,temp);}elseif(calCount>0){AscendC:GeGLU<srcType,false>(dstLocal,src0Local,src1Local,calCount);}else{AscendC:GeGLU<srcType,false>(dstLocal,src0Local,src1Local);}outQueue.EnQue<srcType>(dstLocal);inQueue0.FreeTensor(src0Local);inQueue1.FreeTensor(src1Local);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dstGlobal,dstLocal,dataSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>src0Global;AscendC:GlobalTensor<srcType>src1Global;AscendC:GlobalTensor<srcType>dstGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueue0;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueue1;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;AscendC:TBuf<AscendC:TPosition:VECCALC>buf;uint32_tdataSize=0;};template<typenamedataType>__aicore__voidkernel_geglu_operator(GM_ADDRsrc0Gm,GM_ADDRsrc1Gm,GM_ADDRdstGm,uint32_tsrcSize,uint32_ttmpBufSize,uint32_tcalCount){KernelGeGLU<dataType>op;op.Init(src0Gm,src1Gm,dstGm,srcSize,tmpBufSize);op.Process(tmpBufSize,calCount);} |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 123456789101112131415 | 输入数据(srcTensor0):[1.60253913.47656253.43164063.7539062-1.33300780.72314453-3.00781250.85498047-1.36914062.6894531-2.9101562-3.6992188-2.2734375-2.8593752.5683594-1.7802734]输入数据(srcTensor1)[-0.60156251.95898441.92578123.87695310.58789062.9179688-1.88476563.23046882.89453122.45507811.3730469-1.92480470.7919922-2.5332031-2.1425781-2.9433594]输出数据(dstLocal):[-0.2639160156250000006.6406250000000000006.42968750000000000014.554687500000000000-0.5654296875000000002.1074218750000000000.1685791015625000002.759765625000000000-3.9570312500000000006.558593750000000000-3.6562500000000000000.192993164062500000-1.4150390625000000000.039642333984375000-0.0878906250000000000.007740020751953125] |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
