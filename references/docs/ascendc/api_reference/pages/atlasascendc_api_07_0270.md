# SetFlag/WaitFlag(ISASI)-核内同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0270
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0270.html
---

# SetFlag/WaitFlag(ISASI)

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

同一核内不同流水之间的同步指令。具有数据依赖的不同流水指令之间需要插此同步。

#### 函数原型

| 1234 | template<HardEventevent>__aicore__inlinevoidSetFlag(int32_teventID)template<HardEventevent>__aicore__inlinevoidWaitFlag(int32_teventID) |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名  | 输入/输出 | 描述                                                                                                                                                                                                                                                                                              |
| ------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| event   | 输入      | 模板参数。同步事件，数据类型为HardEvent。详细内容参考下文中的同步类型说明。                                                                                                                                                                                                                       |
| eventID | 输入      | 事件ID。数据类型为int32_t类型。其定义如下：eventID需要通过AllocEventID或者FetchEventID来获取。Atlas训练系列产品，数据范围为：0-3Atlas推理系列产品AI Core，数据范围为：0-7Atlas A2 训练系列产品/Atlas A2 推理系列产品，数据范围为：0-7Atlas A3 训练系列产品/Atlas A3 推理系列产品，数据范围为：0-7 |

同步类型说明如下：

| 1234567891011121314151617181920212223242526272829303132333435 | enumclassHardEvent:uint8_t{// 名称（源流水_目标流水），例如MTE2_V，代表PIPE_MTE2为源流水，PIPE_V为目标流水。标识从PIPE_MTE2到PIPE_V的同步，PIPE_V等待PIPE_MTE2。MTE2_MTE1MTE1_MTE2MTE1_MM_MTE1MTE2_VV_MTE2MTE3_VV_MTE3M_VV_MV_VMTE3_MTE1MTE1_MTE3MTE1_VMTE2_MM_MTE2V_MTE1M_FIXFIX_MMTE3_MTE2MTE2_MTE3S_VV_SS_MTE2MTE2_SS_MTE3MTE3_SMTE2_FIXFIX_MTE2FIX_SM_SFIX_MTE3} |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 返回值说明

无

#### 约束说明

- SetFlag/WaitFlag必须成对出现。
- 禁止用户在使用SetFlag和WaitFlag时，自行指定eventID，容易与框架同步事件冲突，导致卡死问题。eventID需要通过AllocEventID或者FetchEventID来获取。

#### 调用示例

如DataCopy需要等待SetValue执行完成后才能执行，需要插入PIPE_S到PIPE_MTE3的同步。

| 12345678 | AscendC:GlobalTensor<half>dstGlobal;AscendC:LocalTensor<half>dstLocal;dstLocal.SetValue(0,0);uint32_tdataSize=512;int32_teventIDSToMTE3=static_cast<int32_t>(GetTPipePtr()->FetchEventID(AscendC:HardEvent:S_MTE3));AscendC:SetFlag<AscendC:HardEvent:S_MTE3>(eventIDSToMTE3);AscendC:WaitFlag<AscendC:HardEvent:S_MTE3>(eventIDSToMTE3);AscendC:DataCopy(dstGlobal,dstLocal,dataSize); |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
