# Swish-Swish接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0783
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0783.html
---

# Swish

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

在神经网络中，Swish是一个重要的激活函数。计算公式如下，其中β为常数：

![](../images/atlasascendc_api_07_0783_img_001.png)

![](../images/atlasascendc_api_07_0783_img_002.png)

#### 函数原型

| 12  | template<typenameT,boolisReuseSource=false>__aicore__inlinevoidSwish(constLocalTensor<T>&dstLocal,constLocalTensor<T>&srcLocal,uint32_tdataSize,constTscalarValue) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名      | 输入/输出 | 描述                                                                                                             |
| ----------- | --------- | ---------------------------------------------------------------------------------------------------------------- |
| dstLocal    | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                           |
| srcLocal    | 输入      | 源操作数。源操作数的数据类型需要与目的操作数保持一致。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。 |
| dataSize    | 输入      | 实际计算数据元素个数。                                                                                           |
| scalarValue | 输入      | 激活函数中的β参数。支持的数据类型为：half、float。β参数的数据类型需要与源操作数和目的操作数保持一致。            |

#### 返回值说明

无

#### 约束说明

- 操作数地址偏移对齐要求请参见通用说明和约束。
- 不支持源操作数与目的操作数地址重叠。
- 当前仅支持ND格式的输入，不支持其他格式。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061 | #include"kernel_operator.h"template<typenamesrcType>classKernelSwish{public:__aicore__inlineKernelSwish(){}__aicore__inlinevoidInit(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tinputSize,srcTypescalar){dataSize=inputSize;scalarValue=scalar;srcGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(srcGm),dataSize);dstGlobal.SetGlobalBuffer(reinterpret_cast<__gm__srcType*>(dstGm),dataSize);pipe.InitBuffer(inQueueX,1,dataSize*sizeof(srcType));pipe.InitBuffer(outQueue,1,dataSize*sizeof(srcType));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<srcType>srcLocal=inQueueX.AllocTensor<srcType>();AscendC:DataCopy(srcLocal,srcGlobal,dataSize);inQueueX.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<srcType>dstLocal=outQueue.AllocTensor<srcType>();AscendC:LocalTensor<srcType>srcLocal=inQueueX.DeQue<srcType>();Swish(dstLocal,srcLocal,dataSize,scalarValue);outQueue.EnQue<srcType>(dstLocal);inQueueX.FreeTensor(srcLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<srcType>dstLocal=outQueue.DeQue<srcType>();AscendC:DataCopy(dstGlobal,dstLocal,dataSize);outQueue.FreeTensor(dstLocal);}private:AscendC:GlobalTensor<srcType>srcGlobal;AscendC:GlobalTensor<srcType>dstGlobal;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;uint32_tdataSize=0;srcTypescalarValue=0;};template<typenamedataType>__aicore__voidkernel_Swish_operator(GM_ADDRsrcGm,GM_ADDRdstGm,uint32_tdataSize){KernelSwish<dataType>op;dataTypescalarValue=1.702;op.Init(srcGm,dstGm,dataSize,scalarValue);op.Process();} |
| ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1234567891011 | 输入数据(srcLocal):[0.5312-3.654-2.923.787-3.0593.770.571-0.668-0.095340.5454-1.801-1.7911.5630.8783.9731.7992.0231.0183.082-3.8142.254-3.7170.4675-0.4631-2.470.9814-0.8543.313.2563.7641.867-1.773]输出数据(dstLocal):[0.3784-0.007263-0.020163.78-0.016663.7620.414-0.1622-0.043820.3909-0.0803-0.081051.4610.7173.9691.7191.960.86473.066-0.005772.207-0.0066260.3223-0.1448-0.036220.8257-0.16173.2973.2443.7561.792-0.0825] |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
