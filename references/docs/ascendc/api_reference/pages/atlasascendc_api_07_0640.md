# IterateAll-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0640
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0640.html
---

# IterateAll

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

调用一次IterateAll，会计算出singleCoreM * singleCoreN大小的C矩阵。迭代顺序可通过tiling参数iterateOrder调整。

#### 函数原型

| 1   | template<boolsync=true>__aicore__inlinevoidIterateAll(constGlobalTensor<DstT>&gm,uint8_tenAtomic=0,boolenSequentialWrite=false,boolwaitIterateAll=false,boolfakeMsg=false) |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1   | template<boolsync=true>__aicore__inlinevoidIterateAll(constLocalTensor<DstT>&ubCmatrix,uint8_tenAtomic=0) |
| --- | --------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                                                                                                                                                                 |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sync   | 获取C矩阵过程分为同步和异步两种模式：同步：需要同步等待IterateAll执行结束异步：不需要同步等待IterateAll执行结束通过该参数设置同步或者异步模式：同步模式设置为true；异步模式设置为false，默认为同步模式。Atlas 200I/500 A2 推理产品只支持设置为true。 |

| 参数名            | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| gm                | 输出      | C矩阵。类型为GlobalTensor。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int32_t/int8_tAtlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int32_t/int8_tAtlas推理系列产品AI Core，支持的数据类型为：half/float/int8_t/int32_tAtlas 200I/500 A2 推理产品，支持的数据类型为half/float/bfloat16_t/int32_t                                                                                                                                                                                             |
| ubCmatrix         | 输出      | C矩阵。类型为LocalTensor，支持的TPosition为TSCM。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int32_t/int8_tAtlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int32_t/int8_tAtlas推理系列产品AI Core不支持包含该参数的原型接口Atlas 200I/500 A2 推理产品，支持的数据类型为half/float/bfloat16_t/int32_t                                                                                                                                                                                          |
| enAtomic          | 输入      | 是否开启Atomic操作，默认值为0。参数取值：0：不开启Atomic操作1：开启AtomicAdd累加操作2：开启AtomicMax求最大值操作3：开启AtomicMin求最小值操作对于Atlas推理系列产品AI Core，只有输出位置是GM才支持开启Atomic操作。对于Atlas 200I/500 A2 推理产品，只有输出位置是GM才支持开启Atomic操作。                                                                                                                                                                                                                                                                                      |
| enSequentialWrite | 输入      | 是否开启连续写模式（连续写，写入[baseM, baseN]；非连续写，写入[singleCoreM, singleCoreN]中对应的位置），默认值false（非连续写模式）。Atlas 200I/500 A2 推理产品不支持该参数。                                                                                                                                                                                                                                                                                                                                                                                               |
| waitIterateAll    | 输入      | 仅在异步场景下使用，是否需要通过WaitIterateAll接口等待IterateAll执行结束。true：需要通过WaitIterateAll接口等待IterateAll执行结束。false：不需要通过WaitIterateAll接口等待IterateAll执行结束，开发者自行处理等待IterateAll执行结束的过程。                                                                                                                                                                                                                                                                                                                                   |
| fakeMsg           | 输入      | 仅在IBShare场景（模板参数中开启了doIBShareNorm开关）和IntraBlockPartSum场景（模板参数中开启了intraBlockPartSum开关）使用。IBShare场景该场景复用L1上相同的A矩阵或B矩阵数据，要求AIV分核调用IterateAll的次数必须匹配，此时需要调用IterateAll并设置fakeMsg为true，不执行真正的计算，仅用来保证IterateAll调用成对出现。默认值为false，表示执行真正的计算。IntraBlockPartSum场景用于分离模式下的Vector计算、Cube计算融合，实现多个AIV核的一次Matmul计算结果（baseM * baseN大小的矩阵分片）在L0C Buffer上累加。默认值为false，表示执行各AIV核的Matmul计算结果在L0C Buffer上累加。 |

#### 返回值说明

无

#### 约束说明

传入的C矩阵地址空间大小需要保证不小于singleCoreM * singleCoreN个元素。

#### 调用示例

IterateAll接口的调用示例如下，更多异步场景的算子样例请参考IterateAll异步场景矩阵乘法。

| 12345 | REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);mm.SetTensorA(gm_a);mm.SetTensorB(gm_b);mm.SetBias(gm_bias);mm.IterateAll(gm_c);// 计算 |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
