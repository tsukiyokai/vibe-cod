# LoadDataUnzip-数据搬运-矩阵计算(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0243
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0243.html
---

# LoadDataUnzip

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | x        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

将GM上的数据解压并搬运到A1/B1/B2上。执行该API前需要执行LoadUnzipIndex加载压缩索引表。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidLoadDataUnzip(constLocalTensor<T>&dst,constGlobalTensor<T>&src) |
| --- | ------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名称 | 输入/输出 | 含义                                                                                                                                                                           |
| -------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| dst      | 输出      | 目的操作数，类型为LocalTensor，支持的TPosition为A1/B1/B2。LocalTensor的起始地址需要保证：TPosition为A1/B1时，32字节对齐；TPosition为B2时，512B对齐。支持的数据类型为：int8_t。 |
| src      | 输入      | 源操作数，类型为GlobalTensor。数据类型需要与dst保持一致。                                                                                                                      |

#### 约束说明

- 操作数地址对齐要求请参见通用地址对齐约束。

#### 返回值说明

无

#### 调用示例

该调用示例支持的运行平台为Atlas推理系列产品AI Core。

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162 | #include"kernel_operator.h"classKernelLoadUnzip{public:__aicore__inlineKernelLoadUnzip(){}__aicore__inlinevoidInit(__gm__int8_t*weGm,__gm__int8_t*indexGm,__gm__int8_t*dstGm){weGlobal.SetGlobalBuffer((__gm__int8_t*)weGm);indexGlobal.SetGlobalBuffer((__gm__int8_t*)indexGm);dstGlobal.SetGlobalBuffer((__gm__int8_t*)dstGm);pipe.InitBuffer(inQueueB1,1,dstLen*sizeof(int8_t));pipe.InitBuffer(outQueueUB,1,dstLen*sizeof(int8_t));}__aicore__inlinevoidProcess(){CopyIn();CopyToUB();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<int8_t>weightB1=inQueueB1.AllocTensor<int8_t>();AscendC:LoadUnzipIndex(indexGlobal,numOfIndexTabEntry);AscendC:LoadDataUnzip(weightB1,weGlobal);inQueueB1.EnQue(weightB1);}__aicore__inlinevoidCopyToUB(){AscendC:LocalTensor<int8_t>weightB1=inQueueB1.DeQue<int8_t>();AscendC:LocalTensor<int8_t>featureMapUB=outQueueUB.AllocTensor<int8_t>();AscendC:DataCopy(featureMapUB,weightB1,dstLen);outQueueUB.EnQue<int8_t>(featureMapUB);inQueueB1.FreeTensor(weightB1);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<int8_t>featureMapUB=outQueueUB.DeQue<int8_t>();event_teventIdMTE1ToMTE3=static_cast<event_t>(GetTPipePtr()->FetchEventID(AscendC:HardEvent:MTE1_MTE3));AscendC:SetFlag<AscendC:HardEvent:MTE1_MTE3>(eventIdMTE1ToMTE3);AscendC:WaitFlag<AscendC:HardEvent:MTE1_MTE3>(eventIdMTE1ToMTE3);AscendC:DataCopy(dstGlobal,featureMapUB,dstLen);outQueueUB.FreeTensor(featureMapUB);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:B1,1>inQueueB1;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueueUB;AscendC:GlobalTensor<int8_t>weGlobal;AscendC:GlobalTensor<int8_t>dstGlobal;AscendC:GlobalTensor<int8_t>indexGlobal;uint32_tsrcLen=896,dstLen=1024,numOfIndexTabEntry=1;};extern"C"__global____aicore__voidcube_load_unzip_simple_kernel(__gm__int8_t*weightGm,__gm__int8_t*indexGm,__gm__int8_t*dstGm){KernelLoadUnzipop;op.Init(weightGm,indexGm,dstGm);op.Process();} |
| ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
