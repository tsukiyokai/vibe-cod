# GetBatchTensorC-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0637
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0637.html
---

# GetBatchTensorC

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

调用一次GetBatchTensorC，会获取C矩阵片，该接口可以与IterateNBatch异步接口配合使用。用于在调用IterateNBatch迭代计算后，获取一片std:max(batchA, batchB) * singleCoreM * singleCoreN大小的矩阵分片。

#### 函数原型

| 12  | template<boolsync=true>__aicore__inlineGlobalTensor<DstT>GetBatchTensorC(uint32_tbatchA,uint32_tbatchB,boolenSequentialWrite=false) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- |

| 12  | template<boolsync=true>__aicore__inlinevoidGetBatchTensorC(constLocalTensor<DstT>&c,uint32_tbatchA,uint32_tbatchB,boolenSequentialWrite=false) |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                            |
| ------ | ----------------------------------------------- |
| sync   | 当前仅支持异步模式，即该参数只支持取值为false。 |

| 参数名            | 输入/输出 | 描述                                              |
| ----------------- | --------- | ------------------------------------------------- |
| batchA            | 输入      | 左矩阵的batch数。                                 |
| batchB            | 输入      | 右矩阵的batch数。                                 |
| enSequentialWrite | 输入      | 该参数预留，开发者无需关注。                      |
| c                 | 输入      | C矩阵放置于Local Memory的地址，用于保存矩阵分片。 |

#### 返回值说明

GlobalTensor<DstT>，返回计算的矩阵分片。

#### 约束说明

- 当使能MixDualMaster（双主模式）场景时，即模板参数enableMixDualMaster设置为true，不支持使用该接口。
- C矩阵片输出到Local Memory，且单核计算的N方向大小singleCoreN非32字节对齐的场景，C矩阵的CubeFormat仅支持ND_ALIGN格式，输出C矩阵片时，自动将singleCoreN方向上的数据补齐至32字节。

#### 调用示例

| 1234567891011121314 | // 计算需要多Batch计算循环次数intfor_extent=tiling.ALayoutInfoB*tiling.ALayoutInfoN*g_lay/tiling.BatchNum;mm1.SetTensorA(gm_a[0],isTransposeAIn);mm1.SetTensorB(gm_b[0],isTransposeBIn);if(tiling.isBias){mm1.SetBias(gm_bias[0]);}// 多batch Matmul计算mm1.templateIterateNBatch<false>(for_extent,batchA,batchB,false);...othercomputefor(inti=0;i<for_extent;++i){mm1.templateGetBatchTensorC<false>(ubCmatrix);...othercompute} |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
