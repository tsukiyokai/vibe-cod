# ReduceXorSum-ReduceXorSum接口-归约操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0832
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0832.html
---

# ReduceXorSum

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

按照元素执行Xor（按位异或）运算，并将计算结果ReduceSum求和。

![](../images/atlasascendc_api_07_10005_img_003.png)

#### 函数原型

- 通过sharedTmpBuffer入参传入临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidReduceXorSum(LocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor,LocalTensor<uint8_t>&sharedTmpBuffer,constuint32_tcalCount)

- 接口框架申请临时空间12template<typenameT,boolisReuseSource=false>__aicore__inlinevoidReduceXorSum(LocalTensor<T>&dstTensor,constLocalTensor<T>&src0Tensor,constLocalTensor<T>&src1Tensor,constuint32_tcalCount);

由于该接口的内部实现中需要保存异或结果以及进行其他运算，需要额外的临时空间来存储计算过程中的中间变量。临时空间支持开发者通过sharedTmpBuffer入参传入和接口框架申请两种方式。

- 通过sharedTmpBuffer入参传入，使用该tensor作为临时空间进行处理，接口框架不再申请。该方式开发者可以自行管理sharedTmpBuffer内存空间，并在接口调用完成后，复用该部分内存，内存不会反复申请释放，灵活性较高，内存利用率也较高。
- 接口框架申请临时空间，开发者无需申请，但是需要预留临时空间的大小。

通过sharedTmpBuffer传入的情况，开发者需要为tensor申请空间；接口框架申请的方式，开发者需要预留临时空间。临时空间大小BufferSize的获取方式如下：通过GetReduceXorSumMaxMinTmpSize中提供的接口获取需要预留空间范围的大小。

#### 参数说明

| 参数名        | 描述                                                                                                                                                                                                                                                                                                            |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T             | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：int16_t。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：int16_t。Atlas推理系列产品AI Core，支持的数据类型为：int16_t。                                                                                         |
| isReuseSource | 是否允许修改源操作数，默认值为false。如果开发者允许源操作数被改写，可以使能该参数，使能后能够节省部分内存空间。设置为true，则本接口内部计算时复用src0Tensor和src1Tensor的内存空间，节省内存空间；设置为false，则本接口内部计算时不复用src0Tensor和src1Tensor的内存空间。isReuseSource的使用样例请参考更多样例。 |

| 参数名          | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                      |
| --------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dstTensor       | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。输出值需要sizeof(T)大小的空间进行保存。开发者要根据该大小和框架的对齐要求来为dstTensor分配实际内存空间。说明：注意：遵循框架对内存开辟的要求（开辟内存的大小满足32Byte对齐），即sizeof(T)不是32Byte对齐时，需要向上进行32Byte对齐。为了对齐而多开辟的内存空间不填值，为一些随机值。 |
| src0Tensor      | 输入      | 源操作数0。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                                                                                                                                                                                         |
| src1Tensor      | 输入      | 源操作数1。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。源操作数的数据类型需要与目的操作数保持一致。                                                                                                                                                                                                                                         |
| sharedTmpBuffer | 输入      | 临时缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。用于ReduceXorSum计算时存储中间变量，由开发者提供。临时空间大小BufferSize的获取方式请参考GetReduceXorSumMaxMinTmpSize。                                                                                                                                                                |
| calCount        | 输入      | 参与计算的元素个数。                                                                                                                                                                                                                                                                                                                                      |

#### 返回值说明

无

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。

- 不支持源操作数与目的操作数地址重叠。
- 不支持sharedTmpBuffer与源操作数和目的操作数地址重叠。
- calCount需要保证小于或等于src0Tensor和src1Tensor的元素范围。
- 当最终计算结果超出int16范围[-32768，32767]后，将输出-32768或者32767。
- 对于Atlas推理系列产品AI Core，中间计算数据会采用half类型存储，最终计算结果的误差相对于其他处理器较大。

#### 调用示例

| 123456789101112131415161718 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueY;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueue;AscendC:TQue<AscendC:TPosition:VECCALC,1>tmpQue;pipe.InitBuffer(inQueueX,1,32*sizeof(int16_t));pipe.InitBuffer(inQueueY,1,32*sizeof(int16_t));pipe.InitBuffer(outQueue,1,32);pipe.InitBuffer(tmpQue,1,bufferSize);// bufferSize通过Host侧tiling参数获取AscendC:LocalTensor<int16_t>dstLocal=outQueue.AllocTensor<int16_t>();AscendC:LocalTensor<int16_t>src0Local=inQueueX.AllocTensor<int16_t>();AscendC:LocalTensor<int16_t>src1Local=inQueueY.AllocTensor<int16_t>();AscendC:LocalTensor<uint8_t>sharedTmpBuffer=tmpQue.AllocTensor<uint8_t>();// 不使用输入内存，输入shape信息为32, 算子输入的数据类型为int16_t, 实际计算个数为前32AscendC:ReduceXorSum<int16_t,false>(dstLocal,src0Local,src1Local,sharedTmpBuffer,32); |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12345 | 输入输出的数据类型为int16_t输入数据(src0Local):[00000000000000000000000000000000]输入数据(src1Local):[11111111111111111111111111111111]输出数据(dstLocal):[32000000000000000]// 仅32为有效值 |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
