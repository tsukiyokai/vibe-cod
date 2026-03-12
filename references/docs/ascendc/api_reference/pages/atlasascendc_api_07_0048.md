# MulAddDst-复合计算-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0048
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0048.html
---

# MulAddDst

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

按元素将src0和src1相乘并和dst相加，将最终结果存放进dst中。计算公式如下：

![](../images/atlasascendc_api_07_0048_img_001.png)

#### 函数原型

- tensor前n个数据计算12template<typenameT,typenameU>__aicore__inlinevoidMulAddDst(constLocalTensor<T>&dst,constLocalTensor<U>&src0,constLocalTensor<U>&src1,constint32_t&count)
- tensor高维切分计算mask逐bit模式12template<typenameT,typenameU,boolisSetMask=true>__aicore__inlinevoidMulAddDst(constLocalTensor<T>&dst,constLocalTensor<U>&src0,constLocalTensor<U>&src1,constuint64_tmask[],constuint8_trepeatTime,constBinaryRepeatParams&repeatParams)mask连续模式12template<typenameT,typenameU,boolisSetMask=true>__aicore__inlinevoidMulAddDst(constLocalTensor<T>&dst,constLocalTensor<U>&src0,constLocalTensor<U>&src1,uint64_tmask,constuint8_trepeatTime,constBinaryRepeatParams&repeatParams)

#### 参数说明

| 参数名    | 描述                                                                                                                                                                                                                                                                                                                                                                                |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T         | 目的操作数数据类型。目的操作数和源操作数的数据类型约束请参考表3。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas 200I/500 A2 推理产品，支持的数据类型为：int16_t、uint16_t、half、int32_t、uint32_t、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。 |
| U         | 源操作数数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half、float。Atlas 200I/500 A2 推理产品，支持的数据类型为：int16_t、uint16_t、half、int32_t、uint32_t、float。Atlas推理系列产品AI Core，支持的数据类型为：half、float。                                                |
| isSetMask | 是否在接口内部设置mask。true，表示在接口内部设置mask。false，表示在接口外部设置mask，开发者需要使用SetVectorMask接口设置mask值。这种模式下，本接口入参中的mask值必须设置为占位符MASK_PLACEHOLDER。                                                                                                                                                                                  |

| 参数名       | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dst          | 输出      | 目的操作数。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要32字节对齐。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| src0、src1   | 输入      | 源操作数。源操作数数据类型和目的操作数数据类型可以不一致。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要32字节对齐。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| count        | 输入      | 参与计算的元素个数。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| mask[]/mask  | 输入      | mask用于控制每次迭代内参与计算的元素。逐bit模式：可以按位控制哪些元素参与计算，bit位的值为1表示参与计算，0表示不参与。mask为数组形式，数组长度和数组元素的取值范围和操作数的数据类型有关。当操作数为16位时，数组长度为2，mask[0]、mask[1]∈[0, 264-1]并且不同时为0；当操作数为32位时，数组长度为1，mask[0]∈(0, 264-1]；当操作数为64位时，数组长度为1，mask[0]∈(0, 232-1]。例如，mask=[8, 0]，8=0b1000，表示仅第4个元素参与计算。连续模式：表示前面连续的多少个元素参与计算。取值范围和操作数的数据类型有关，数据类型不同，每次迭代内能够处理的元素个数最大值不同。当操作数为16位时，mask∈[1, 128]；当操作数为32位时，mask∈[1, 64]；当操作数为64位时，mask∈[1, 32]。 |
| repeatTime   | 输入      | 重复迭代次数。矢量计算单元，每次读取连续的256Bytes数据进行计算，为完成对输入数据的处理，必须通过多次迭代(repeat)才能完成所有数据的读取与计算。repeatTime表示迭代的次数。关于该参数的具体描述请参考高维切分API。                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| repeatParams | 输入      | 控制操作数地址步长的参数。BinaryRepeatParams类型，包含操作数相邻迭代间相同datablock的地址步长，操作数同一迭代内不同datablock的地址步长等参数。相邻迭代间的地址步长参数说明请参考repeatStride；同一迭代内DataBlock的地址步长参数说明请参考dataBlockStride。                                                                                                                                                                                                                                                                                                                                                                                                         |

| src0数据类型 | src1数据类型 | dst数据类型 | PAR | 支持的型号                                                                                                                               |
| ------------ | ------------ | ----------- | --- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| half         | half         | half        | 128 | Atlas A2 训练系列产品/Atlas A2 推理系列产品Atlas A3 训练系列产品/Atlas A3 推理系列产品Atlas推理系列产品AI CoreAtlas 200I/500 A2 推理产品 |
| float        | float        | float       | 64  | Atlas A2 训练系列产品/Atlas A2 推理系列产品Atlas A3 训练系列产品/Atlas A3 推理系列产品Atlas推理系列产品AI CoreAtlas 200I/500 A2 推理产品 |
| half         | half         | float       | 64  | Atlas A2 训练系列产品/Atlas A2 推理系列产品Atlas A3 训练系列产品/Atlas A3 推理系列产品Atlas推理系列产品AI Core                           |
| int16_t      | int16_t      | int16_t     | 128 | Atlas 200I/500 A2 推理产品                                                                                                               |
| uint16_t     | uint16_t     | uint16_t    | 128 | Atlas 200I/500 A2 推理产品                                                                                                               |
| int32_t      | int32_t      | int32_t     | 64  | Atlas 200I/500 A2 推理产品                                                                                                               |
| uint32_t     | uint32_t     | uint32_t    | 64  | Atlas 200I/500 A2 推理产品                                                                                                               |

#### 返回值说明

无

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。
- 操作数地址重叠约束请参考通用地址重叠约束。特别地，源操作数的数据类型为half、目的操作数的数据类型为float的情况下，源操作数和目的操作数无法100%重叠，故不支持地址重叠。

#### 调用示例

- 高维切分计算接口样例-mask连续模式（源操作数的数据类型为half、目的操作数的数据类型为float）12345uint64_tmask=64;// repeatTime = 4，一次迭代计算64个数，共计算256个数// dstBlkStride, src0BlkStride, src1BlkStride = 1，单次迭代内数据连续读取和写入// dstRepStride = 8，src0RepStride, src1RepStride = 4，相邻迭代间数据连续读取和写入AscendC:MulAddDst(dstLocal,src0Local,src1Local,64,4,{1,1,1,8,4,4});
- 高维切分计算接口样例-mask逐bit模式（源操作数的数据类型为half、目的操作数的数据类型为float）12345uint64_tmask[2]={UINT64_MAX,0};// repeatTime = 4，一次迭代计算64个数，共计算256个数// dstBlkStride, src0BlkStride, src1BlkStride = 1，单次迭代内数据连续读取和写入// dstRepStride = 8，src0RepStride, src1RepStride = 4，相邻迭代间数据连续读取和写入AscendC:MulAddDst(dstLocal,src0Local,src1Local,mask,4,{1,1,1,8,4,4});
- tensor前n个数据计算样例（源操作数的数据类型为half、目的操作数的数据类型为float）1AscendC:MulAddDst(dstLocal,src0Local,src1Local,256);
