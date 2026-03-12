# 昇腾社区官网-昇腾万里 让智能无所不及

**页面ID:** atlasascendc_api_07_0877
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0877.html
---

# Commit

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

非细粒度通信时，每调用一次本接口，则通知服务端可以执行handleId对应的任务一次。细粒度通信时，每调用一次本接口，则通知服务端可以执行handleId对应的步长长度的子通信任务。

#### 函数原型

| 1   | __aicore__inlinevoidCommit(HcclHandlehandleId) |
| --- | ---------------------------------------------- |

#### 参数说明

| 参数名   | 输入/输出               | 描述                                                                            |     |                         |
| -------- | ----------------------- | ------------------------------------------------------------------------------- | --- | ----------------------- |
| handleId | 输入                    | 对应通信任务的标识ID，只能使用Prepare原语接口的返回值。1usingHcclHandle=int8_t; | 1   | usingHcclHandle=int8_t; |
| 1        | usingHcclHandle=int8_t; |                                                                                 |     |                         |

#### 返回值说明

无

#### 约束说明

- 调用本接口前确保已调用过InitV2和SetCcTilingV2接口。
- 入参handleId只能使用Prepare原语对应接口的返回值。
- 非细粒度通信时，本接口的调用次数应该与Prepare的repeat次数一致。细粒度通信时，本接口的调用次数应该与通信任务的总步骤数/步长*Prepare的repeat次数一致。
- 本接口在AIC核或者AIV核上调用必须与对应的Prepare接口的调用核保持一致。

#### 调用示例

| 1234567891011121314151617181920 | REGISTER_TILING_DEFAULT(ReduceScatterCustomTilingData);//ReduceScatterCustomTilingData为对应算子头文件定义的结构体GET_TILING_DATA_WITH_STRUCT(ReduceScatterCustomTilingData,tilingData,tilingGM);Hcclhccl;GM_ADDRcontextGM=AscendC:GetHcclContext<0>();// AscendC自定义算子kernel中，通过此方式获取HCCL contexthccl.InitV2(contextGM,&tilingData);autoret=hccl.SetCcTilingV2(offsetof(ReduceScatterCustomTilingData,mc2CcTiling));if(ret!=HCCL_SUCCESS){return;}if(AscendC:g_coreType==AIC){HcclHandlehandleId=hccl.ReduceScatter(sendBuf,recvBuf,100,HcclDataType:HCCL_DATA_TYPE_INT8,HcclReduceOp:HCCL_REDUCE_SUM,10);for(uint8_ti=0;i<10;i++){hccl.Commit(handleId);// 通知服务端可以执行上述的ReduceScatter任务}for(uint8_ti=0;i<10;i++){hccl.Wait(handleId);// 阻塞接口，需等待上述ReduceScatter任务执行完毕}hccl.Finalize();// 后续无其他通信任务，通知服务端执行上述ReduceScatter任务之后即可以退出} |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
