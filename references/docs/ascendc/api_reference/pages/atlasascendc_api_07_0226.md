# GetReduceMaxMinCount(ISASI)-归约计算-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0226
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0226.html
---

# GetReduceMaxMinCount(ISASI)

#### 产品支持情况

| 产品                                        | 是否支持（仅获取最值的原型） | 是否支持（获取最值和索引的原型） |
| ------------------------------------------- | ---------------------------- | -------------------------------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x                            | √                                |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | x                            | √                                |
| Atlas 200I/500 A2 推理产品                  | x                            | x                                |
| Atlas推理系列产品AI Core                    | √                            | x                                |
| Atlas推理系列产品Vector Core                | x                            | x                                |
| Atlas训练系列产品                           | x                            | x                                |

#### 功能说明

获取ReduceMax、ReduceMin连续场景下的最大/最小值以及相应的索引值。

#### 函数原型

- 获取ReduceMax、ReduceMin连续场景下的最大值与最小值，以及相应的索引值。12template<typenameT>__aicore__inlinevoidGetReduceMaxMinCount(T&maxMinValue,T&maxMinIndex)
- 获取ReduceMax、ReduceMin连续场景下的最大值与最小值。12template<typenameT>__aicore__inlinevoidGetReduceMaxMinCount(T&maxMinValue)

#### 参数说明

| 参数名 | 描述                                                |
| ------ | --------------------------------------------------- |
| T      | ReduceMax/ReduceMin指令的数据类型，支持half/float。 |

| 参数名      | 输入/输出 | 描述                                        |
| ----------- | --------- | ------------------------------------------- |
| maxMinValue | 输出      | ReduceMax/ReduceMin指令的最大值/最小值。    |
| maxMinIndex | 输出      | ReduceMax/ReduceMin指令的最值对应的索引值。 |

#### 返回值说明

无

#### 约束说明

- 针对Atlas A2 训练系列产品/Atlas A2 推理系列产品，由于ReduceMax/ReduceMin的内部实现原因，直接调用GetReduceMaxMinCount接口无法获取到准确的索引值，验证时需要使用WholeReduceMax/WholeReduceMin接口来获取准确的索引值。
- 针对Atlas A3 训练系列产品/Atlas A3 推理系列产品，由于ReduceMax/ReduceMin的内部实现原因，直接调用GetReduceMaxMinCount接口无法获取到准确的索引值，验证时需要使用WholeReduceMax/WholeReduceMin接口来获取准确的索引值。
- 索引maxMinIndex数据`是按照ReduceMax/ReduceMin的数据类型进行存储的，比如ReduceMax/ReduceMin使用half类型时，maxMinIndex是按照half类型进行存储的，如果按照half格式进行读取，maxMinIndex的值是不对的，因此maxMinIndex的读取需要使用reinterpret_cast方法转换到整数类型，若输入数据类型是half，需要使用reinterpret_cast<uint16_t*>，若输入是float，需要使用reinterpret_cast<uint32_t*>。

#### 调用示例

1. 以ReduceMax指令为例，首先执行ReduceMax指令。12345AscendC:LocalTensor<float>src;AscendC:LocalTensor<float>work;AscendC:LocalTensor<float>dst;int32_tmask=64;AscendC:ReduceMax(dst,src,work,mask,1,8,true);// 连续场景，srcRepStride = 8，且calIndex = true
1. 获取上述ReduceMax指令的最值与索引值。针对Atlas A2 训练系列产品/Atlas A2 推理系列产品，需要使用WholeReduceMax指令获取准确的索引值，然后再调用GetReduceMaxMinCount指令。1234567AscendC:LocalTensor<float>src;AscendC:LocalTensor<float>dst;int32_tmask=64;AscendC:WholeReduceMax(dst,src,mask,1,1,1,8);floatval=0;// 最值floatidx=0;// 最值的索引值，与ReduceMax的结果相同AscendC:GetReduceMaxMinCount<float>(val,idx);针对Atlas A3 训练系列产品/Atlas A3 推理系列产品，需要使用WholeReduceMax指令获取准确的索引值，然后再调用GetReduceMaxMinCount指令。1234567AscendC:LocalTensor<float>src;AscendC:LocalTensor<float>dst;int32_tmask=64;AscendC:WholeReduceMax(dst,src,mask,1,1,1,8);floatval=0;// 最值floatidx=0;// 最值的索引值，与ReduceMax的结果相同AscendC:GetReduceMaxMinCount<float>(val,idx);针对Atlas推理系列产品AI Core版本，则可在调用ReduceMax后直接调用GetReduceMaxMinCount指令获取其最大/最小值。1234567AscendC:LocalTensor<float>src;AscendC:LocalTensor<float>work;AscendC:LocalTensor<float>dst;int32_tmask=64;AscendC:ReduceMax(dst,src,work,mask,1,8,true);floatval=0;// 最值AscendC:GetReduceMaxMinCount<float>(val);
