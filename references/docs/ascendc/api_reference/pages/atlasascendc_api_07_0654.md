# SetWorkspace-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0654
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0654.html
---

# SetWorkspace

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

Iterate计算的异步场景，调用本接口申请一块临时空间来缓存计算结果，然后调用GetTensorC时会在该临时空间中获取C的矩阵分片。

IterateNBatch计算时，调用本接口申请一块临时空间来缓存计算结果，然后根据同步或异步场景进行其它接口的调用。

#### 函数原型

建议用户使用GlobalTensor类型传入：

| 1   | template<classT>__aicore__inlinevoidSetWorkspace(GlobalTensor<T>&addr) |
| --- | ---------------------------------------------------------------------- |

| 1   | template<classT>__aicore__inlinevoidSetWorkspace(__gm__constT*addr,intsize) |
| --- | --------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                              |
| ------ | --------- | ------------------------------------------------- |
| addr   | 输入      | 用户传入的GM上的workspace空间，GlobalTensor类型。 |
| addr   | 输入      | 用户传入的GM上的workspace空间，GM地址类型。       |
| size   | 输入      | 传入GM地址时，需要配合传入元素个数。              |

#### 返回值说明

无

#### 约束说明

当使能MixDualMaster（双主模式）场景时，即模板参数enableMixDualMaster设置为true，不支持使用该接口。

#### 调用示例

| 123456789 | REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);mm.SetWorkspace(workspaceGM);//设置异步时使用的临时空间mm.SetTensorA(gm_a);mm.SetTensorB(gm_b);mm.SetBias(gm_bias);mm.templateIterate<false>();for(inti=0;i<singleCoreM/baseM*singleCoreN/baseN;++i){mm.templateGetTensorC<false>(ub_c);} |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
