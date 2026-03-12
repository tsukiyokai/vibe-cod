# HCCL模板参数-HCCL Kernel侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0870
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0870.html
---

# HCCL模板参数

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

创建HCCL对象时需要传入模板参数。

#### 函数原型

Hccl类定义如下，模板参数说明见表1 Hccl类模板参数说明。

| 12  | template<HcclServerTypeserverType=HcclServerType:HCCL_SERVER_TYPE_AICPU,constauto&config=DEFAULT_CFG>classHccl; |
| --- | --------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名称   | 描述                                                                                                                                                                                                                                                                                                                                                                |      |                                                                                                               |       |                                                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------- | ----- | ---------------------------------------------------------------------------------------------------------------- |
| serverType | 支持的服务端类型。HcclServerType类型，定义如下。对于Atlas A2 训练系列产品/Atlas A2 推理系列产品，当前仅支持HCCL_SERVER_TYPE_AICPU。对于Atlas A3 训练系列产品/Atlas A3 推理系列产品，当前仅支持HCCL_SERVER_TYPE_AICPU。1234enumHcclServerType{HCCL_SERVER_TYPE_AICPU=0,HCCL_SERVER_TYPE_END// 预留参数，不支持使用}                                                  | 1234 | enumHcclServerType{HCCL_SERVER_TYPE_AICPU=0,HCCL_SERVER_TYPE_END// 预留参数，不支持使用}                      |       |                                                                                                                  |
| 1234       | enumHcclServerType{HCCL_SERVER_TYPE_AICPU=0,HCCL_SERVER_TYPE_END// 预留参数，不支持使用}                                                                                                                                                                                                                                                                            |      |                                                                                                               |       |                                                                                                                  |
| config     | 用于指定向服务端下发任务的核。HcclServerConfig类型，定义如下，默认值DEFAULT_CFG = {CoreType:DEFAULT, 0}。1234structHcclServerConfig{CoreTypetype;// 向服务端下发任务的核的类型int64_tblockId;// 向服务端下发任务的核的ID};CoreType的定义如下：12345enumclassCoreType:uint8_t{DEFAULT,// 表示不指定AIC核或者AIV核ON_AIV,// 表示指定为AIV核ON_AIC// 表示指定为AIC核}; | 1234 | structHcclServerConfig{CoreTypetype;// 向服务端下发任务的核的类型int64_tblockId;// 向服务端下发任务的核的ID}; | 12345 | enumclassCoreType:uint8_t{DEFAULT,// 表示不指定AIC核或者AIV核ON_AIV,// 表示指定为AIV核ON_AIC// 表示指定为AIC核}; |
| 1234       | structHcclServerConfig{CoreTypetype;// 向服务端下发任务的核的类型int64_tblockId;// 向服务端下发任务的核的ID};                                                                                                                                                                                                                                                       |      |                                                                                                               |       |                                                                                                                  |
| 12345      | enumclassCoreType:uint8_t{DEFAULT,// 表示不指定AIC核或者AIV核ON_AIV,// 表示指定为AIV核ON_AIC// 表示指定为AIC核};                                                                                                                                                                                                                                                    |      |                                                                                                               |       |                                                                                                                  |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 123 | staticconstexprHcclServerConfigHCCL_CFG={CoreType:ON_AIV,10};// 选择AICPU作为服务端Hccl<HcclServerType:HCCL_SERVER_TYPE_AICPU,HCCL_CFG>hccl; |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------- |
