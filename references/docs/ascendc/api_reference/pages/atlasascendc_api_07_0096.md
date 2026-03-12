# SetVectorMask-掩码操作-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0096
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0096.html
---

# SetVectorMask

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

- Normal模式下，mask参数用来控制单次迭代内参与计算的元素个数。此时又可以划分为如下两种模式：连续模式：表示前面连续的多少个元素参与计算。取值范围和操作数的数据类型有关，数据类型不同，每次迭代内能够处理的元素个数最大值不同。当操作数为16位时，mask∈[1, 128]；当操作数为32位时，mask∈[1, 64]；当操作数为64位时，mask∈[1, 32]。逐比特模式：可以按位控制哪些元素参与计算，bit位的值为1表示参与计算，0表示不参与。分为maskHigh（高位mask）和maskLow（低位mask）。参数取值范围和操作数的数据类型有关，数据类型不同，每次迭代内能够处理的元素个数最大值不同。当操作数为16位时，maskLow、maskHigh∈[0, 264-1]，并且不同时为0；当操作数为32位时，maskHigh为0，maskLow∈(0, 264-1]；当操作数为64位时，maskHigh为0，maskLow∈(0, 232-1]。
- Counter模式下，mask参数表示整个矢量计算参与计算的元素个数。

#### 函数原型

- 适用于Normal模式下mask逐比特模式和Counter模式12template<typenameT,MaskModemode=MaskMode:NORMAL>__aicore__staticinlinevoidSetVectorMask(constuint64_tmaskHigh,constuint64_tmaskLow)
- 适用于Normal模式下mask连续模式和Counter模式12template<typenameT,MaskModemode=MaskMode:NORMAL>__aicore__staticinlinevoidSetVectorMask(int32_tlen)

#### 参数说明

| 参数名 | 描述                                                                                                          |      |                                                                         |
| ------ | ------------------------------------------------------------------------------------------------------------- | ---- | ----------------------------------------------------------------------- |
| T      | 矢量计算操作数数据类型。                                                                                      |      |                                                                         |
| mode   | mask模式，MaskMode类型，定义如下：1234enumclassMaskMode:uint8_t{NORMAL=0,// Normal模式COUNTER// Counter模式}; | 1234 | enumclassMaskMode:uint8_t{NORMAL=0,// Normal模式COUNTER// Counter模式}; |
| 1234   | enumclassMaskMode:uint8_t{NORMAL=0,// Normal模式COUNTER// Counter模式};                                       |      |                                                                         |

| 参数名   | 输入/输出 | 描述                                                                                                                                              |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| maskHigh | 输入      | Normal模式：对应Normal模式下的逐比特模式，可以按位控制哪些元素参与计算。传入高位mask值。Counter模式：需要置0，本入参不生效。                      |
| maskLow  | 输入      | Normal模式：对应Normal模式下的逐比特模式，可以按位控制哪些元素参与计算。传入低位mask值。Counter模式：整个矢量计算过程中，参与计算的元素个数。     |
| len      | 输入      | Normal模式：对应Normal模式下的mask连续模式，表示单次迭代内表示前面连续的多少个元素参与计算。Counter模式：整个矢量计算过程中，参与计算的元素个数。 |

#### 返回值说明

无

#### 约束说明

该接口仅在矢量计算API的isSetMask模板参数为false时生效，使用完成后需要使用ResetMask将mask恢复为默认值。

#### 调用示例

可结合SetMaskCount与SetMaskNorm使用，先设置mask的模式再设置mask：

- Normal模式调用示例12345678910111213141516AscendC:LocalTensor<half>dstLocal;AscendC:LocalTensor<half>src0Local;AscendC:LocalTensor<half>src1Local;// Normal模式AscendC:SetMaskNorm();AscendC:SetVectorMask<half,AscendC:MaskMode:NORMAL>(0xffffffffffffffff,0xffffffffffffffff);// 逐bit模式// SetVectorMask<half, MaskMode:NORMAL>(128);  // 连续模式// 多次调用矢量计算API, 可以统一设置为Normal模式，并设置mask参数，无需在API内部反复设置，省去了在API反复设置的过程，会有一定的性能优势// dstBlkStride, src0BlkStride, src1BlkStride = 1, 单次迭代内数据连续读取和写入// dstRepStride, src0RepStride, src1RepStride = 8, 相邻迭代间数据连续读取和写入AscendC:Add<half,false>(dstLocal,src0Local,src1Local,AscendC:MASK_PLACEHOLDER,1,{2,2,2,8,8,8});AscendC:Sub<half,false>(src0Local,dstLocal,src1Local,AscendC:MASK_PLACEHOLDER,1,{2,2,2,8,8,8});AscendC:Mul<half,false>(src1Local,dstLocal,src0Local,AscendC:MASK_PLACEHOLDER,1,{2,2,2,8,8,8});AscendC:ResetMask();
- Counter模式调用示例1234567891011121314151617181920212223// Counter模式和tensor高维切分计算接口配合使用AscendC:LocalTensor<half>dstLocal;AscendC:LocalTensor<half>src0Local;AscendC:LocalTensor<half>src1Local;int32_tlen=128;// 参与计算的元素个数AscendC:SetMaskCount();AscendC:SetVectorMask<half,AscendC:MaskMode:COUNTER>(len);AscendC:Add<half,false>(dstLocal,src0Local,src1Local,AscendC:MASK_PLACEHOLDER,1,{1,1,1,8,8,8});AscendC:Sub<half,false>(src0Local,dstLocal,src1Local,AscendC:MASK_PLACEHOLDER,1,{1,1,1,8,8,8});AscendC:Mul<half,false>(src1Local,dstLocal,src0Local,AscendC:MASK_PLACEHOLDER,1,{1,1,1,8,8,8});AscendC:SetMaskNorm();AscendC:ResetMask();// Counter模式和tensor前n个数据计算接口配合使用AscendC:LocalTensor<half>dstLocal;AscendC:LocalTensor<half>src0Local;halfnum=2;AscendC:SetMaskCount();AscendC:SetVectorMask<half,AscendC:MaskMode:COUNTER>(128);// 参与计算的元素个数为128AscendC:Adds<half,false>(dstLocal,src0Local,num,1);AscendC:Muls<half,false>(dstLocal,src0Local,num,1);AscendC:SetMaskNorm();AscendC:ResetMask();
