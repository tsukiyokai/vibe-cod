# AllocEventID-TPipe-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0114
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0114.html
---

# AllocEventID

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

用于申请HardEvent（硬件类型同步事件）的TEventID，必须与ReleaseEventID搭配使用，调用该接口后，会占用申请的TEventID，直至调用ReleaseEventID释放。

#### 函数原型

| 12  | template<HardEventevt>__aicore__inlineTEventIDAllocEventID() |
| --- | ------------------------------------------------------------ |

#### 参数说明

| 参数名 | 描述                                                                                   |
| ------ | -------------------------------------------------------------------------------------- |
| evt    | HardEvent硬件同步类型。该类型的具体说明请参考SetFlag/WaitFlag(ISASI)中同步类型的说明。 |

#### 约束说明

TEventID有数量限制，使用结束后应该立刻调用ReleaseEventID释放，防止TEventID耗尽。

#### 返回值说明

TEventID

#### 调用示例

| 12345678 | AscendC:TEventIDeventID=GetTPipePtr()->AllocEventID<AscendC:HardEvent:V_S>();//需要插入scalar等vector的同步，申请对应的HardEvent的IDAscendC:SetFlag<AscendC:HardEvent:V_S>(eventID);..................AscendC:WaitFlag<AscendC:HardEvent:V_S>(eventID);GetTPipePtr()->ReleaseEventID<AscendC:HardEvent:V_S>(eventID);//释放scalar等vector的同步HardEvent的ID...... |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
