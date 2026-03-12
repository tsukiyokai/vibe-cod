# Cosh-Cosh接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0527
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0527.html
---

# Cosh

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

按元素做双曲余弦函数计算，计算公式如下：

![](../images/atlasascendc_api_07_0528_img_001.png)

![](../images/atlasascendc_api_07_0528_img_002.png)

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidCosh(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidCosh(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constLocalTensor<uint8_t>&sharedTmpBuffer)

- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidCosh(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidCosh(constLocalTensor<T>&dstTensor,constLocalTensor<T>&srcTensor)

由于该接口的内部实现中涉及复杂的数学计算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持接口框架申请和开发者通过sharedTmpBuffer入参传入两种方式。

- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

接口框架申请的方式，开发者需要预留临时空间；通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间。临时空间大小BufferSize的获取方式如下：通过GetCoshMaxMinTmpSize中提供的GetCoshMaxMinTmpSize接口获取需要预留空间大小的上下限。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名          | 输入/输出 | 描述                                                                                                                             |
| --------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                           |
| srcTensor       | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                 |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。临时空间大小BufferSize的获取方式请参考GetCoshMaxMinTmpSize。 |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                             |

#### 返回值说明

无

#### 约束说明

- 不支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

| 123456 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECCALC,1>tmpQue;pipe.InitBuffer(tmpQue,1,bufferSize);// bufferSize通过Host侧tiling参数获取AscendC:LocalTensor<uint8_t>sharedTmpBuffer=tmpQue.AllocTensor<uint8_t>();// 输入tensor长度为1024, 算子输入的数据类型为half, 实际计算个数为512AscendC:Cosh(dstLocal,srcLocal,sharedTmpBuffer,512); |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12  | 输入数据(srcLocal):[0.80468755.92578125-10.9609375...-8.1562511.0859375]输出数据(dstLocal):[1.341796875187.2528784.0...1743.032624.0] |
| --- | ------------------------------------------------------------------------------------------------------------------------------------- |
