# DataCachePreload-缓存控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0275
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0275.html
---

# DataCachePreload

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

从源地址所在的特定GM地址预加载数据到data cache中。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidDataCachePreload(constGlobalTensor<uint64_t>&src,constTcacheOffset) |
| --- | ---------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名      | 输入/输出 | 描述                                                                                         |
| ----------- | --------- | -------------------------------------------------------------------------------------------- |
| src         | 输入      | 源操作数，类型为GlobalTensor。支持的数据类型为：uint64_t。                                   |
| cacheOffset | 输入      | 在源操作数上偏移cacheOffset大小开始加载数据，单位为byte，支持的数据类型为：int16_t/int64_t。 |

#### 返回值说明

无

#### 约束说明

频繁调用此接口可能导致保留站拥塞，这种情况下，此指令将被视为NOP指令，阻塞Scalar流水。因此不建议频繁调用该接口。

#### 调用示例

| 123 | AscendC:GlobalTensor<uint64_t>srcGlobal;int64_tcacheOffset=0;AscendC:DataCachePreload(srcGlobal,cacheOffset); |
| --- | ------------------------------------------------------------------------------------------------------------- |
