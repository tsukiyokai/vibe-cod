# Erfc-Erfc接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0548
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0548.html
---

# Erfc

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

返回输入x的互补误差函数结果，积分区间为x到无穷大。原始的理论计算公式如下：

![](../images/atlasascendc_api_07_0548_img_001.png)

![](../images/atlasascendc_api_07_0548_img_002.png)

由于Erfc函数没有初等函数表达方式，一般通过函数逼近的方式计算，近似计算公式如下所示：

![](../images/atlasascendc_api_07_0548_img_003.png)

其中，

R(z) = (((((((z * R0 + R1) * z + R2) * z + R3) * z + R4) * z + R5) * z + R6) * z + R7) * z + R8是关于z的8次多项式；

S(z) = ((((z + S1) * z + S2) * z + S3) * z + S4) * z + S5是关于z的4次多项式。

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErfc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErfc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)

- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErfc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidErfc(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetErfcMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                               |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                             |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                   |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于Erfc内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetErfcMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 输入源数据需保持值域在[-inf, inf]。若输入不在范围内，输出结果无效。
- 不支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

完整的调用样例请参考更多样例。

| 123456 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECCALC,1>tmpQue;pipe.InitBuffer(tmpQue,1,bufferSize);// bufferSize通过Host侧tiling参数获取AscendC:LocalTensor<uint8_t>sharedTmpBuffer=tmpQue.AllocTensor<uint8_t>();// 输入tensor长度为1024, 算子输入的数据类型为half, 实际计算个数为512AscendC:Erfc(dstLocal,srcLocal,sharedTmpBuffer,512); |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12  | 输入数据(srcLocal):[-inf-10...1inf]输出数据(dstLocal):[2.00000001.84270381.0000000...0.15729610] |
| --- | ------------------------------------------------------------------------------------------------ |
