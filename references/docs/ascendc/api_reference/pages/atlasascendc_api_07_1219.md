# GetHcclContext-HCCL Context-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1219
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1219.html
---

# GetHcclContext

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | √        |
| Atlas训练系列产品                           | x        |

#### 功能说明

获取指定Index通信域的context（消息区）地址。

#### 函数原型

| 12  | template<uint32_tindex>__aicore__inline__gm__uint8_t*__gm__GetHcclContext(void) |
| --- | ------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                      |
| ------ | ------------------------------------------------------------------------- |
| index  | 模板参数，用来表示要设置的通信域ID，当前只支持2个通信域，index只能为0/1。 |

#### 返回值说明

指定通信域的context（消息区）地址。

#### 约束说明

当前最多只支持2个通信域。

#### 调用示例

| 1   | GM_ADDRcontextGM=AscendC:GetHcclContext<0>(); |
| --- | --------------------------------------------- |
