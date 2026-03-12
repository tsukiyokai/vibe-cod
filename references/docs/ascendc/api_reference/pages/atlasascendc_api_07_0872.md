# AllReduce-HCCL Kernel侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0872
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0872.html
---

# AllReduce

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

集合通信算子AllReduce的任务下发接口，返回该任务的标识handleId给用户。AllReduce功能为：将通信域内所有节点的同名张量进行reduce操作后，再把结果发送到所有节点的输出buffer。

![](../images/atlasascendc_api_07_0872_img_001.png)

#### 函数原型

| 12  | template<boolcommit=false>__aicore__inlineHcclHandleAllReduce(GM_ADDRsendBuf,GM_ADDRrecvBuf,uint64_tcount,HcclDataTypedataType,HcclReduceOpop,uint8_trepeat=1) |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                                                |
| ------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| commit | 输入      | bool类型。参数取值如下：true：在调用Prepare接口时，Commit同步通知服务端可以执行该通信任务。false：在调用Prepare接口时，不通知服务端执行该通信任务。 |

| 参数名   | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                               |
| -------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sendBuf  | 输入      | 源数据buffer地址。                                                                                                                                                                                                                                                                                                                                 |
| recvBuf  | 输出      | 目的数据buffer地址，集合通信结果输出到此buffer中。                                                                                                                                                                                                                                                                                                 |
| count    | 输入      | 参与AllReduce操作的数据个数，比如只有一个int32数据参与，则count=1。                                                                                                                                                                                                                                                                                |
| dataType | 输入      | AllReduce操作的数据类型，目前支持float、half（即float16）、int8_t、int16_t、int32_t、bfloat16_t数据类型，即支持取值为HCCL_DATA_TYPE_FP32、HCCL_DATA_TYPE_FP16、HCCL_DATA_TYPE_INT8、HCCL_DATA_TYPE_INT16、HCCL_DATA_TYPE_INT32、HCCL_DATA_TYPE_BFP16。HcclDataType数据类型的介绍请参考表1。                                                        |
| op       | 输入      | Reduce的操作类型，目前支持sum、max、min操作类型，即支持取值为HCCL_REDUCE_SUM、HCCL_REDUCE_MAX、HCCL_REDUCE_MIN。HcclReduceOp数据类型的介绍请参考表2。                                                                                                                                                                                              |
| repeat   | 输入      | 一次下发的AllReduce通信任务个数。repeat取值≥1，默认值为1。当repeat>1时，每个AllReduce任务的sendBuf和recvBuf地址由服务端自动算出，计算公式如下：sendBuf[i] = sendBuf + count* sizeof(datatype) * i, i∈[0, repeat)recvBuf[i] = recvBuf + count* sizeof(datatype) * i, i∈[0, repeat)注意：当设置repeat>1时，须与count参数配合使用，规划通信数据地址。 |

![](../images/atlasascendc_api_07_0872_img_002.png)

#### 返回值说明

返回该任务的标识handleId，handleId大于等于0。调用失败时，返回 -1。

#### 约束说明

- 调用本接口前确保已调用过InitV2和SetCcTilingV2接口。
- 若HCCL对象的config模板参数未指定下发通信任务的核，该接口只能在AIC核或者AIV核两者之一上调用。若HCCL对象的config模板参数中指定了下发通信任务的核，则该接口可以在AIC核和AIV核上同时调用，接口内部会根据指定的核的类型，只在AIC核、AIV核二者之一下发该通信任务。
- 对于Atlas A2 训练系列产品/Atlas A2 推理系列产品，一个通信域内，所有Prepare接口的总调用次数不能超过63。
- 对于Atlas A3 训练系列产品/Atlas A3 推理系列产品，一个通信域内，所有Prepare接口和InterHcclGroupSync接口的总调用次数不能超过63。

#### 调用示例

- 非多轮切分场景如下图所示，4张卡上均有count=300个float16数据，每张卡从xGM内存中获取到本卡数据，各卡的数据进行reduce sum计算后，将结果输出到各卡的yGM。图2非多轮切分场景下4卡AllReduce通信1234567891011121314151617181920212223extern"C"__global____aicore__voidall_reduce_custom(GM_ADDRxGM,GM_ADDRyGM,GM_ADDRworkspaceGM,GM_ADDRtilingGM){autosendBuf=xGM;// xGM为AllReduce的输入GM地址autorecvBuf=yGM;// yGM为AllReduce的输出GM地址uint64_tsendCount=300;// 每张卡上均有300个float16的数据HcclReduceOpreduceOp=HcclReduceOp:HCCL_REDUCE_SUM;REGISTER_TILING_DEFAULT(AllReduceCustomTilingData);//AllReduceCustomTilingData为对应算子头文件定义的结构体GET_TILING_DATA_WITH_STRUCT(AllReduceCustomTilingData,tilingData,tilingGM);Hcclhccl;GM_ADDRcontextGM=AscendC:GetHcclContext<0>();// AscendC自定义算子kernel中，通过此方式获取HCCL contextif(AscendC:g_coreType==AIV){// 指定AIV核通信hccl.InitV2(contextGM,&tilingData);autoret=hccl.SetCcTilingV2(offsetof(AllReduceCustomTilingData,mc2CcTiling));if(ret){return;}HcclHandlehandleId1=hccl.AllReduce<true>(sendBuf,recvBuf,sendCount,HcclDataType:HCCL_DATA_TYPE_FP16,reduceOp);hccl.Wait(handleId1);AscendC:SyncAll<true>();// 全AIV核同步，防止0核执行过快，提前调用hccl.Finalize()接口，导致其他核Wait卡死hccl.Finalize();}}
- 多轮切分场景使能多轮切分，等效处理上述非多轮切分示例的通信。如下图所示，每张卡的300个float16数据，被切分为2个首块数据，1个尾块数据。每个首块的数据量tileLen为128个float16数据，尾块的数据量tailLen为44个float16数据。在算子内部实现时，需要对切分后的数据分3轮进行AllReduce通信任务，将等效上述非多轮切分的通信结果。图3各卡数据切分示意图具体实现为，第1轮通信，每个rank上0-0\1-0\2-0\3-0数据块进行AllReduce处理。第2轮通信，每个rank上0-1\1-1\2-1\3-1数据块进行AllReduce处理。第3轮通信，每个rank上0-2\1-2\2-2\3-2数据块进行AllReduce处理，图示及代码示例如下。图44卡AllReduce示意图1234567891011121314151617181920212223242526272829303132333435363738extern"C"__global____aicore__voidall_reduce_custom(GM_ADDRxGM,GM_ADDRyGM,GM_ADDRworkspaceGM,GM_ADDRtilingGM){constexpruint32_ttileNum=2U;// 首块数量constexpruint64_ttileLen=128U;// 首块数据个数constexpruint32_ttailNum=1U;// 尾块数量constexpruint64_ttailLen=44U;// 尾块数据个数autosendBuf=xGM;// xGM为AllReduce的输入GM地址autorecvBuf=yGM;// yGM为AllReduce的输出GM地址HcclReduceOpreduceOp=HcclReduceOp:HCCL_REDUCE_SUM;REGISTER_TILING_DEFAULT(AllReduceCustomTilingData);//AllReduceCustomTilingData为对应算子头文件定义的结构体GET_TILING_DATA_WITH_STRUCT(AllReduceCustomTilingData,tilingData,tilingGM);Hcclhccl;GM_ADDRcontextGM=AscendC:GetHcclContext<0>();// AscendC自定义算子kernel中，通过此方式获取HCCL contextif(AscendC:g_coreType==AIV){// 指定AIV核通信hccl.InitV2(contextGM,&tilingData);autoret=hccl.SetCcTilingV2(offsetof(AllReduceCustomTilingData,mc2CcTiling));if(ret!=HCCL_SUCCESS){return;}// 2个首块处理constexpruint32_ttileRepeat=tileNum;// 除了sendBuf和recvBuf入参不同，对2个首块处理的其余参数相同。故使用repeat=2，第2个首块AllReduce任务的sendBuf、recvBuf将由API内部自行更新HcclHandlehandleId1=hccl.AllReduce<true>(sendBuf,recvBuf,tileLen,HcclDataType:HCCL_DATA_TYPE_FP16,reduceOp,tileRepeat);// 1个尾块处理constexpruint32_tkSizeOfFloat16=2U;sendBuf+=tileLen*tileNum*kSizeOfFloat16;recvBuf+=tileLen*tileNum*kSizeOfFloat16;constexpruint32_ttailRepeat=tailNum;HcclHandlehandleId2=hccl.AllReduce<true>(sendBuf,recvBuf,tailLen,HcclDataType:HCCL_DATA_TYPE_FP16,reduceOp,tailRepeat);for(uint8_ti=0;i<tileRepeat;i++){hccl.Wait(handleId1);}hccl.Wait(handleId2);AscendC:SyncAll<true>();// 全AIV核同步，防止0核执行过快，提前调用hccl.Finalize()接口，导致其他核Wait卡死hccl.Finalize();}}
