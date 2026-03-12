# SetCcTilingV2-HCCL Kernel侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10222
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10222.html
---

# SetCcTilingV2

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

用于设置HCCL客户端中某个通信算法配置的TilingData地址。

#### 函数原型

| 1   | __aicore__inlineint32_tSetCcTilingV2(uint64_toffset) |
| --- | ---------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                                                                                                                             |
| ------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| offset | 输入      | 通信算法配置Mc2CcTiling参数地址相对于Mc2InitTiling起始地址的偏移。Mc2CcTiling在Host侧计算得出，具体请参考表2 Mc2CcTiling参数说明，由框架传递到Kernel函数中使用。 |

#### 返回值说明

- HCCL_SUCCESS，表示成功。
- HCCL_FAILED，表示失败。

#### 约束说明

- 若调用本接口，必须保证InitV2在本接口前被调用。
- Tiling参数相同的同一种通信算法在调用Prepare接口前，只需要调用一次本接口，请参考调用示例：类型不同、Tiling参数不同的通信。
- 对于同一种通信算法，如果Tiling参数不同，重复调用本接口会覆盖之前的Tiling参数地址，因此需要在调用Prepare接口后再调用本接口设置新的Tiling参数。请参考调用示例：类型相同、Tiling参数不同的通信。
- 若调用本接口，必须使用标准C++语法定义TilingData结构体的开发方式，具体请参考使用标准C++语法定义Tiling结构体。

#### 调用示例

- 用户自定义TilingData结构体：1234567classUserCustomTilingData{AscendC:tiling:Mc2InitTilinginitTiling;AscendC:tiling:Mc2CcTilingallGatherTiling;AscendC:tiling:Mc2CcTilingallReduceTiling1;AscendC:tiling:Mc2CcTilingallReduceTiling2;CustomTilingparam;};
- 类型不同、Tiling参数不同的通信123456789101112131415161718192021extern"C"__global____aicore__voiduserKernel(GM_ADDRaGM,GM_ADDRworkspaceGM,GM_ADDRtilingGM){REGISTER_TILING_DEFAULT(UserCustomTilingData);GET_TILING_DATA_WITH_STRUCT(UserCustomTilingData,tilingData,tilingGM);Hcclhccl;GM_ADDRcontextGM=AscendC:GetHcclContext<0>();hccl.InitV2(contextGM,&tilingData);// 在下发任务之前，通过SetCcTilingV2设置对应的tilingif(hccl.SetCcTilingV2(offsetof(UserCustomTilingData,allGatherTiling))!=HCCL_SUCCESS||hccl.SetCcTilingV2(offsetof(UserCustomTilingData,allReduceTiling1))!=HCCL_SUCCESS){return;}constautoagHandleId=hccl.AllGather<true>(sendBuf,recvBuf,dataCount,HcclDataType:HCCL_DATA_TYPE_FP16);hccl.Wait(agHandleId);constautoarHandleId=hccl.AllReduce<true>(sendBuf,recvBuf,dataCount,HcclDataType:HCCL_DATA_TYPE_FP16,HcclReduceOp:HCCL_REDUCE_SUM);hccl.Wait(arHandleId);hccl.Finalize();}
- 类型相同、Tiling参数不同的通信123456789101112131415161718192021222324extern"C"__global____aicore__voiduserKernel(GM_ADDRaGM,GM_ADDRworkspaceGM,GM_ADDRtilingGM){REGISTER_TILING_DEFAULT(UserCustomTilingData);GET_TILING_DATA_WITH_STRUCT(UserCustomTilingData,tilingData,tilingGM);Hcclhccl;GM_ADDRcontextGM=AscendC:GetHcclContext<0>();hccl.InitV2(contextGM,&tilingData);// 在下发通信任务之前，通过SetCcTilingV2设置对应的Tiling参数地址if(hccl.SetCcTilingV2(offsetof(UserCustomTilingData,allReduceTiling1))!=HCCL_SUCCESS){return;}constautoarHandleId1=hccl.AllReduce<true>(sendBuf,recvBuf,dataCount,HcclDataType:HCCL_DATA_TYPE_FP16,HcclReduceOp:HCCL_REDUCE_SUM);hccl.Wait(arHandleId1);// 第二次AllReduce的Tiling参数与第一次不同，在第一次Prepare之后再调用SetCcTilingV2if(hccl.SetCcTilingV2(offsetof(UserCustomTilingData,allReduceTiling2))!=HCCL_SUCCESS){return;}constautoarHandleId2=hccl.AllReduce<true>(sendBuf,recvBuf,dataCount,HcclDataType:HCCL_DATA_TYPE_FP16,HcclReduceOp:HCCL_REDUCE_SUM);hccl.Wait(arHandleId2);hccl.Finalize();}
