# RpSort16-排序组合(ISASI)-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0229
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0229.html
---

# RpSort16

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

根据Region Proposals中的score域对其进行排序（score大的排前面），每次排16个Region Proposals。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidRpSort16(constLocalTensor<T>&dst,constLocalTensor<T>&src,constint32_trepeatTime) |
| --- | ----------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------------------- |
| T      | 操作数数据类型。Atlas训练系列产品，支持的数据类型为：halfAtlas推理系列产品AI Core，支持的数据类型为：half/float |

| 参数名称   | 输入/输出 | 含义                                                                                                                                          |
| ---------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| dst        | 输出      | 目的操作数，存储经过排序后的Region Proposals。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要32字节对齐。 |
| src        | 输入      | 源操作数，存储未经过排序的Region Proposals。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。LocalTensor的起始地址需要32字节对齐。   |
| repeatTime | 输入      | 重复迭代次数，int32_t类型，每次排16个Region Proposals。取值范围：repeatTime∈[0,255]。                                                         |

#### 约束说明

- 用户需保证src和dst中存储的Region Proposal数目大于实际所需数据，否则会存在tensor越界错误。
- 当存在proposal[i]与proposal[j]的score值相同时，如果i>j，则proposal[j]将首先被选出来，排在前面。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

- 接口使用样例12// repeatTime = 2, 对2个Region Proposal进行排序AscendC:RpSort16(dstLocal,dstLocal,2);
- 完整样例123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566#include"kernel_operator.h"classKernelVecProposal{public:__aicore__inlineKernelVecProposal(){}__aicore__inlinevoidInit(__gm__uint8_t*src,__gm__uint8_t*dstGm){srcGlobal.SetGlobalBuffer((__gm__half*)src);dstGlobal.SetGlobalBuffer((__gm__half*)dstGm);pipe.InitBuffer(inQueueSrc,1,srcDataSize*sizeof(half));pipe.InitBuffer(outQueueDst,1,dstDataSize*sizeof(half));}__aicore__inlinevoidProcess(){CopyIn();PreProcess();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<half>srcLocal=inQueueSrc.AllocTensor<half>();AscendC:DataCopy(srcLocal,srcGlobal,srcDataSize);inQueueSrc.EnQue(srcLocal);}__aicore__inlinevoidPreProcess(){AscendC:LocalTensor<half>srcLocal=inQueueSrc.DeQue<half>();AscendC:LocalTensor<half>dstLocal=outQueueDst.AllocTensor<half>();AscendC:ProposalConcat(dstLocal,srcLocal,repeat,mode);// sort排序是基于score的，此处先创建一个有score数据的proposal，需要注意的是，非score处的数据可能是随机值outQueueDst.EnQue<half>(dstLocal);inQueueSrc.FreeTensor(srcLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<half>dstLocal=outQueueDst.DeQue<half>();AscendC:RpSort16(dstLocal,dstLocal,repeat);outQueueDst.EnQue<half>(dstLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<half>dstLocal=outQueueDst.DeQue<half>();AscendC:DataCopy(dstGlobal,dstLocal,dstDataSize);outQueueDst.FreeTensor(dstLocal);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueSrc;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueueDst;AscendC:GlobalTensor<half>srcGlobal,dstGlobal;intsrcDataSize=32;intdstDataSize=256;intrepeat=srcDataSize/16;intmode=4;};extern"C"__global____aicore__voidvec_proposal_kernel(__gm__uint8_t*src,__gm__uint8_t*dstGm){KernelVecProposalop;op.Init(src,dstGm);op.Process();}示例结果
输入数据(src_gm):
[ -1.624 -42.3   -54.12   91.25  -99.4    36.72   67.44  -66.3   -52.53
   3.377 -62.47  -15.85  -31.47    3.143  58.47  -83.75 21.58   63.47    
   7.234  35.16  -39.72   37.8    73.06  -98.7    44.1 -77.2    67.2    
   19.62  -87.9   -14.875  15.86  -77.75]
输出数据(dst_gm):
[  0.      0.      0.      0.     91.25    0.      0.      0.      0.
0.      0.      0.     67.44    0.      0.      0.      0.      0.
0.      0.     58.47    0.      0.      0.      0.      0.      0.
0.     36.72    0.      0.      0.      0.      0.      0.      0.
   3.377   0.      0.      0.      0.      0.      0.      0.      3.143
0.      0.      0.      0.      0.      0.      0.     -1.624   0.
0.      0.      0.      0.      0.      0.    -15.85    0.      0.
0.      0.      0.      0.      0.    -31.47    0.      0.      0.
0.      0.      0.      0.    -42.3     0.      0.      0.      0.
0.      0.      0.    -52.53    0.      0.      0.      0.      0.
0.      0.    -54.12    0.      0.      0.      0.      0.      0.
0.    -62.47    0.      0.      0.      0.      0.      0.      0.
 -66.3     0.      0.      0.      0.      0.      0.      0.    -83.75
0.      0.      0.      0.      0.      0.      0.    -99.4     0.
0.      0.      0.      0.      0.      0.     73.06    0.      0.      
0.      0.      0.      0.      0.     67.2     0.      0.      0.      
0.      0.      0.      0.     63.47    0.      0.      0.      0.      
0.      0.      0.     44.1     0.      0.      0.      0.      0.      
0.      0.     37.8     0.      0.      0.      0.      0.      0.      
0.     35.16    0.      0.      0.      0.      0.      0.      0.     
  21.58    0.      0.      0.      0.      0.      0.      0.     19.62    
0.      0.      0.      0.      0.      0.      0.     15.86    0.      
0.      0.      0.      0.      0.      0.      7.234   0.      0.      
0.      0.      0.      0.      0.    -14.875   0.      0.      0.      
0.      0.      0.      0.    -39.72    0.      0.      0.      0.      
0.      0.      0.    -77.2     0.      0.      0.      0.      0.      
0.      0.    -77.75    0.      0.      0.      0.      0.      0.      
0.    -87.9     0.      0.      0.      0.      0.      0.      0.    
 -98.7     0.      0.      0.   ]
