# ReadSpmBuffer-TPipe-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0168
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0168.html
---

# ReadSpmBuffer

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

从SPM Buffer读回到local数据中。

#### 函数原型

- 适用于连续和不连续的数据回读：12template<typenameT>__aicore__inlinevoidReadSpmBuffer(constLocalTensor<T>&readBuffer,constDataCopyParams&copyParams,int32_treadOffset=0)

- 适用于连续的数据回读：12template<typenameT>__aicore__inlinevoidReadSpmBuffer(constLocalTensor<T>&readBuffer,constint32_treadSize,int32_treadOffset=0)

#### 参数说明

| 参数名称   | 输入/输出 | 含义                                                            |
| ---------- | --------- | --------------------------------------------------------------- |
| readBuffer | 输入      | 读回的目标local内存。                                           |
| copyParams | 输入      | 搬运参数，DataCopyParams类型，DataCopyParams结构定义请参考表2。 |
| readSize   | 输入      | 读回的元素个数。                                                |
| readOffset | 输入      | SPM Buffer的偏移，单位为字节。                                  |

| 参数名称   | 含义                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| blockCount | 待搬运的连续传输数据块个数。uint16_t类型，取值范围：blockCount∈[1, 4095]。                                                                                                                                                                                                                                                                                                                                                                      |
| blockLen   | 待搬运的每个连续传输数据块长度，单位为DataBlock（32字节）。uint16_t类型，取值范围：blockLen∈[1, 65535]。特别地，当dst位于C2PIPE2GM时，单位为128B；当dst位于C2时，表示源操作数的连续传输数据块长度，单位为64B。                                                                                                                                                                                                                                  |
| srcGap     | 源操作数相邻连续数据块的间隔（前面一个数据块的尾与后面数据块的头的间隔），单位为DataBlock（32字节）。uint16_t类型，srcGap不要超出该数据类型的取值范围。在L1 Buffer -> Fixpipe Buffer场景中，srcGap特指源操作数相邻连续数据块的间隔（前面一个数据块的头与后面数据块的头的间隔），单位为DataBlock（32字节）。uint16_t类型，srcGap不要超出该数据类型的取值范围。                                                                                   |
| dstGap     | 目的操作数相邻连续数据块间的间隔（前面一个数据块的尾与后面数据块的头的间隔），单位为DataBlock（32字节）。uint16_t类型，dstGap不要超出该数据类型的取值范围。特别地，当dstLocal位于C2PIPE2GM时，单位为128B；当dstLocal位于C2时，单位为64B。在L1 Buffer -> Fixpipe Buffer场景中，dstGap特指源操作数相邻连续数据块的间隔（前面一个数据块的头与后面数据块的头的间隔），单位为DataBlock（32字节）。uint16_t类型，dstGap不要超出该数据类型的取值范围。 |

#### 约束说明

无

#### 返回值说明

无

#### 调用示例

| 12345678 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueSrcVecIn;intdataSize=32;// 假设T为half类型，从UB上申请一块内存32 * sizeof(half)字节intoffset=32;// 读回时在spmBuffer上偏移32字节pipe.InitBuffer(inQueueSrcVecIn,1,dataSize*sizeof(half));AscendC:LocalTensor<half>writeLocal=inQueueSrcVecIn.AllocTensor<half>();AscendC:DataCopyParamscopyParams{1,2,0,0};// 搬运一个连续传输数据块，连续传输数据块的长度为2个datablock，一个datablock32字节pipe.ReadSpmBuffer(writeLocal,copyParams,offset); |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1234567 | AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueSrcVecIn;intdataSize=64;// 从UB上申请一块内存64*sizeof(half)字节intoffset=32;// 读回时在spmBuffer上偏移32字节pipe.InitBuffer(inQueueSrcVecIn,1,dataSize*sizeof(half));AscendC:LocalTensor<half>writeLocal=inQueueSrcVecIn.AllocTensor<half>();pipe.ReadSpmBuffer(writeLocal,dataSize,offset); |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
