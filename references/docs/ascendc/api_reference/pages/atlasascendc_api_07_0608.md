# Fmod-Fmod接口-数学计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0608
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0608.html
---

# Fmod

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

按元素计算两个浮点数a, b相除后的余数。计算公式如下：

![](../images/atlasascendc_api_07_0608_img_001.png)

![](../images/atlasascendc_api_07_0608_img_002.png)

其中，Trunc为向零取整操作。举例如下：

Fmod(2.0, 1.5) = 0.5

Fmod(-3.0, 1.1) = -0.8

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFmod(constLocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor,constLocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFmod(constLocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor,constLocalTensor<uint8_t>&sharedTmpBuffer)

- 接口框架申请临时空间源操作数Tensor全部/部分参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFmod(constLocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor,constuint32_tcalCount)源操作数Tensor全部参与计算12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidFmod(constLocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor)

由于该接口的内部实现中涉及精度转换。需要额外的临时空间来存储计算过程中的中间变量。临时空间支持接口框架申请和开发者通过sharedTmpBuffer入参传入两种方式。

- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。

接口框架申请的方式，开发者需要预留临时空间；通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间。临时空间大小BufferSize的获取方式如下：通过GetFmodMaxMinTmpSize中提供的接口获取需要预留空间的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| isReuseSource | 是否允许修改源操作数。该参数预留，传入默认值false即可。                                                                                                                                                                             |

| 参数名                 | 输入/输出 | 描述                                                                                                                                                                               |
| ---------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor              | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。                                                                                                             |
| src0Tensor、src1Tensor | 输入      | 源操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                   |
| sharedTmpBuffer        | 输入      | 临时空间。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于Fmod内部复杂计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetFmodMaxMinTmpSize。 |
| calCount               | 输入      | 参与计算的元素个数。                                                                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 针对Atlas推理系列产品AI Core，输入数据限制在[-2147483647.0, 2147483647.0]范围内。
- 源操作数src0Tensor与src1Tensor的数据长度必须保持一致。
- 不支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

| 123456 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECCALC,1>tmpQue;pipe.InitBuffer(tmpQue,1,bufferSize);// bufferSize通过Host侧tiling参数获取AscendC:LocalTensor<uint8_t>sharedTmpBuffer=tmpQue.AllocTensor<uint8_t>();// 输入tensor长度为1024，算子输入的数据类型为half，实际计算个数为512AscendC:Fmod(dstLocal,src0Local,src1Local,sharedTmpBuffer,512); |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 123 | 输入数据(src0Local):[0.5317103-6.379120325.53408647...11.11059642-11.67860335]输入数据(src1Local):[2.125268343.09347812-0.327234...5.643342325.97345923]输出数据(dstLocal):[0.5317-0.19220.2983...5.4673-5.7051] |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
