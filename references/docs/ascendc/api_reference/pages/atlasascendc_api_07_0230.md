# MrgSort4-排序组合(ISASI)-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0230
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0230.html
---

# MrgSort4

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | x        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

将已经排好序的最多4条Region Proposals队列，排列并合并成1条队列，结果按照score域由大到小排序。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidMrgSort4(constLocalTensor<T>&dst,constMrgSortSrcList<T>&src,constMrgSort4Info&params) |
| --- | ---------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------------------- |
| T      | 操作数数据类型。Atlas训练系列产品，支持的数据类型为：halfAtlas推理系列产品AI Core，支持的数据类型为：half/float |

| 参数名                | 输入/输出                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | 含义                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dst                   | 输出                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 目的操作数，存储经过排序后的Region Proposals。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要保证16字节对齐（针对half数据类型），32字节对齐（针对float数据类型）。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| src                   | 输入                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 源操作数，4个Region Proposals队列，并且每个Region Proposal队列都已经排好序，类型为MrgSortSrcList结构体，具体定义如下：123456789101112131415template<typenameT>structMrgSortSrcList{__aicore__MrgSortSrcList(){}__aicore__MrgSortSrcList(constLocalTensor<T>&src1In,constLocalTensor<T>&src2In,constLocalTensor<T>&src3In,constLocalTensor<T>&src4In){src1=src1In[0];src2=src2In[0];src3=src3In[0];src4=src4In[0];}LocalTensor<T>src1;// 第一个已经排好序的Region Proposals队列LocalTensor<T>src2;// 第二个已经排好序的Region Proposals队列LocalTensor<T>src3;// 第三个已经排好序的Region Proposals队列LocalTensor<T>src4;// 第四个已经排好序的Region Proposals队列};Region Proposal队列的数据类型与目的操作数保持一致。src1、src2、src3、src4类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要保证16字节对齐（针对half数据类型），32字节对齐（针对float数据类型）。 | 123456789101112131415 | template<typenameT>structMrgSortSrcList{__aicore__MrgSortSrcList(){}__aicore__MrgSortSrcList(constLocalTensor<T>&src1In,constLocalTensor<T>&src2In,constLocalTensor<T>&src3In,constLocalTensor<T>&src4In){src1=src1In[0];src2=src2In[0];src3=src3In[0];src4=src4In[0];}LocalTensor<T>src1;// 第一个已经排好序的Region Proposals队列LocalTensor<T>src2;// 第二个已经排好序的Region Proposals队列LocalTensor<T>src3;// 第三个已经排好序的Region Proposals队列LocalTensor<T>src4;// 第四个已经排好序的Region Proposals队列}; |
| 123456789101112131415 | template<typenameT>structMrgSortSrcList{__aicore__MrgSortSrcList(){}__aicore__MrgSortSrcList(constLocalTensor<T>&src1In,constLocalTensor<T>&src2In,constLocalTensor<T>&src3In,constLocalTensor<T>&src4In){src1=src1In[0];src2=src2In[0];src3=src3In[0];src4=src4In[0];}LocalTensor<T>src1;// 第一个已经排好序的Region Proposals队列LocalTensor<T>src2;// 第二个已经排好序的Region Proposals队列LocalTensor<T>src3;// 第三个已经排好序的Region Proposals队列LocalTensor<T>src4;// 第四个已经排好序的Region Proposals队列}; |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| params                | 输入                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | 排序所需参数，类型为MrgSort4Info结构体。具体定义请参考${INSTALL_DIR}/include/ascendc/basic_api/interface/kernel_struct_proposal.h，${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。参数说明请参考表3。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

| 参数名称              | 输入/输出 | 含义                                                                                                                                                                                                                                                           |
| --------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| elementLengths        | 输入      | 四个源Region Proposals队列的长度（Region Proposal数目），类型为长度为4的uint16_t数据类型的数组，理论上每个元素取值范围[0, 4095]，但不能超出UB的存储空间。                                                                                                      |
| ifExhaustedSuspension | 输入      | 某条队列耗尽后，指令是否需要停止，类型为bool，默认false。                                                                                                                                                                                                      |
| validBit              | 输入      | 有效队列个数，取值如下：3：前两条队列有效7：前三条队列有效15：四条队列全部有效                                                                                                                                                                                 |
| repeatTimes           | 输入      | 迭代次数，每一次源操作数和目的操作数跳过四个队列总长度。取值范围：repeatTimes∈[1,255]。repeatTimes参数生效是有条件的，需要同时满足以下四个条件：四个源Region Proposals队列的长度一致四个源Region Proposals队列连续存储ifExhaustedSuspension = FalsevalidBit=15 |

#### 约束说明

- 当存在proposal[i]与proposal[j]的score值相同时，如果i>j，则proposal[j]将首先被选出来，排在前面。
- 操作数地址对齐要求请参见通用地址对齐约束。

- 不支持源操作数与目的操作数之间存在地址重叠。

#### 调用示例

- 接口使用样例12345// vconcatWorkLocal为已经创建并且完成排序的4个Region Proposals，每个Region Proposal数目是16个structMrgSortSrcList<half>srcList(vconcatWorkLocal[0],vconcatWorkLocal[1],vconcatWorkLocal[2],vconcatWorkLocal[3]);uint16_telementLengths[4]={16,16,16,16};structMrgSort4InfosrcInfo(elementLengths,false,15,1);AscendC:MrgSort4(dstLocal,srcList,srcInfo);

- 完整样例12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485#include"kernel_operator.h"classKernelVecProposal{public:__aicore__inlineKernelVecProposal(){}__aicore__inlinevoidInit(__gm__uint8_t*src,__gm__uint8_t*dstGm){srcGlobal.SetGlobalBuffer(reinterpret_cast<__gm__half*>(src),srcDataSize);dstGlobal.SetGlobalBuffer((__gm__half*)dstGm);pipe.InitBuffer(inQueueSrc,1,srcDataSize*sizeof(half));pipe.InitBuffer(workQueue,1,dstDataSize*sizeof(half));pipe.InitBuffer(outQueueDst,1,dstDataSize*sizeof(half));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<half>srcLocal=inQueueSrc.AllocTensor<half>();AscendC:DataCopy(srcLocal,srcGlobal,srcDataSize);inQueueSrc.EnQue(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<half>srcLocal=inQueueSrc.DeQue<half>();AscendC:LocalTensor<half>vconcatWorkLocal=workQueue.AllocTensor<half>();AscendC:LocalTensor<half>dstLocal=outQueueDst.AllocTensor<half>();// 先构造4个region proposal然后进行合并排序AscendC:ProposalConcat(vconcatWorkLocal[0],srcLocal[0],repeat,mode);AscendC:RpSort16(vconcatWorkLocal[0],vconcatWorkLocal[0],repeat);AscendC:ProposalConcat(vconcatWorkLocal[workDataSize],srcLocal[singleDataSize],repeat,mode);AscendC:RpSort16(vconcatWorkLocal[workDataSize],vconcatWorkLocal[workDataSize],repeat);AscendC:ProposalConcat(vconcatWorkLocal[workDataSize*2],srcLocal[singleDataSize*2],repeat,mode);AscendC:RpSort16(vconcatWorkLocal[workDataSize*2],vconcatWorkLocal[workDataSize*2],repeat);AscendC:ProposalConcat(vconcatWorkLocal[workDataSize*3],srcLocal[singleDataSize*3],repeat,mode);AscendC:RpSort16(vconcatWorkLocal[workDataSize*3],vconcatWorkLocal[workDataSize*3],repeat);AscendC:MrgSortSrcList<half>srcList(vconcatWorkLocal[0],vconcatWorkLocal[workDataSize],vconcatWorkLocal[workDataSize*2],vconcatWorkLocal[workDataSize*3]);uint16_telementLengths[4]={singleDataSize,singleDataSize,singleDataSize,singleDataSize};AscendC:MrgSort4InfosrcInfo(elementLengths,false,15,1);AscendC:MrgSort4(dstLocal,srcList,srcInfo);outQueueDst.EnQue<half>(dstLocal);inQueueSrc.FreeTensor(srcLocal);workQueue.FreeTensor(vconcatWorkLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<half>dstLocal=outQueueDst.DeQue<half>();AscendC:DataCopy(dstGlobal,dstLocal,dstDataSize);outQueueDst.FreeTensor(dstLocal);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueSrc;AscendC:TQue<AscendC:TPosition:VECIN,1>workQueue;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueueDst;AscendC:GlobalTensor<half>srcGlobal,dstGlobal;intsrcDataSize=64;uint16_tsingleDataSize=srcDataSize/4;intdstDataSize=512;intworkDataSize=dstDataSize/4;intrepeat=srcDataSize/4/16;intmode=4;};extern"C"__global____aicore__voidvec_proposal_kernel(__gm__uint8_t*src,__gm__uint8_t*dstGm){KernelVecProposalop;op.Init(src,dstGm);op.Process();}示例结果
输入数据(src_gm):
[-38.1    82.7   -40.75  -54.62   21.67  -58.53   25.94  -79.5   -61.44
  26.7   -27.45   48.78   86.75  -18.1   -58.8    62.38   46.38  -78.94
 -87.7   -13.81  -13.25   46.94  -47.8   -50.44   34.16   20.3    80.1
 -94.1    52.4   -42.75   83.4    80.44  -66.8   -82.7   -91.44  -95.6
  66.2   -30.97  -36.53   61.66   24.92  -45.1    38.97  -34.62  -69.8
  59.1    34.22   11.695 -33.47   52.1    -4.832  46.88   56.78   71.4
  13.29  -35.78   52.44  -46.03   83.8    83.56   71.3    -9.086 -65.06
  46.25 ]
输出数据(dst_gm):
[  0.      0.      0.      0.     86.75    0.      0.      0.      0.
0.      0.      0.     83.8     0.      0.      0.      0.      0.
0.      0.     83.56    0.      0.      0.      0.      0.      0.
0.     83.4     0.      0.      0.      0.      0.      0.      0.
  82.7     0.      0.      0.      0.      0.      0.      0.     80.44
0.      0.      0.      0.      0.      0.      0.     80.1     0.
0.      0.      0.      0.      0.      0.     71.4     0.      0.
0.      0.      0.      0.      0.     71.3     0.      0.      0.
0.      0.      0.      0.     66.2     0.      0.      0.      0.
0.      0.      0.     62.38    0.      0.      0.      0.      0.
0.      0.     61.66    0.      0.      0.      0.      0.      0.
0.     59.1     0.      0.      0.      0.      0.      0.      0.
  56.78    0.      0.      0.      0.      0.      0.      0.     52.44
0.      0.      0.      0.      0.      0.      0.     52.4     0.
0.      0.      0.      0.      0.      0.     52.1     0.      0.
0.      0.      0.      0.      0.     48.78    0.      0.      0.
0.      0.      0.      0.     46.94    0.      0.      0.      0.
0.      0.      0.     46.88    0.      0.      0.      0.      0.
0.      0.     46.38    0.      0.      0.      0.      0.      0.
0.     46.25    0.      0.      0.      0.      0.      0.      0.
  38.97    0.      0.      0.      0.      0.      0.      0.     34.22
0.      0.      0.      0.      0.      0.      0.     34.16    0.
0.      0.      0.      0.      0.      0.     26.7     0.      0.
0.      0.      0.      0.      0.     25.94    0.      0.      0.
0.      0.      0.      0.     24.92    0.      0.      0.      0.
0.      0.      0.     21.67    0.      0.      0.      0.      0.
0.      0.     20.3     0.      0.      0.      0.      0.      0.
0.     13.29    0.      0.      0.      0.      0.      0.      0.
  11.695   0.      0.      0.      0.      0.      0.      0.     -4.832
0.      0.      0.      0.      0.      0.      0.     -9.086   0.
0.      0.      0.      0.      0.      0.    -13.25    0.      0.
0.      0.      0.      0.      0.    -13.81    0.      0.      0.
0.      0.      0.      0.    -18.1     0.      0.      0.      0.
0.      0.      0.    -27.45    0.      0.      0.      0.      0.
0.      0.    -30.97    0.      0.      0.      0.      0.      0.
0.    -33.47    0.      0.      0.      0.      0.      0.      0.
 -34.62    0.      0.      0.      0.      0.      0.      0.    -35.78
0.      0.      0.      0.      0.      0.      0.    -36.53    0.
0.      0.      0.      0.      0.      0.    -38.1     0.      0.
0.      0.      0.      0.      0.    -40.75    0.      0.      0.
0.      0.      0.      0.    -42.75    0.      0.      0.      0.
0.      0.      0.    -45.1     0.      0.      0.      0.      0.
0.      0.    -46.03    0.      0.      0.      0.      0.      0.
0.    -47.8     0.      0.      0.      0.      0.      0.      0.
 -50.44    0.      0.      0.      0.      0.      0.      0.    -54.62
0.      0.      0.      0.      0.      0.      0.    -58.53    0.
0.      0.      0.      0.      0.      0.    -58.8     0.      0.
0.      0.      0.      0.      0.    -61.44    0.      0.      0.
0.      0.      0.      0.    -65.06    0.      0.      0.      0.
0.      0.      0.    -66.8     0.      0.      0.      0.      0.
0.      0.    -69.8     0.      0.      0.      0.      0.      0.
0.    -78.94    0.      0.      0.      0.      0.      0.      0.
 -79.5     0.      0.      0.      0.      0.      0.      0.    -82.7
0.      0.      0.      0.      0.      0.      0.    -87.7     0.
0.      0.      0.      0.      0.      0.    -91.44    0.      0.
0.      0.      0.      0.      0.    -94.1     0.      0.      0.
0.      0.      0.      0.    -95.6     0.      0.      0.   ]
