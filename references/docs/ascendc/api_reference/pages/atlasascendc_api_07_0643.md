# WaitIterateBatch-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0643
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0643.html
---

# WaitIterateBatch

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

等待IterateBatch异步接口或IterateNBatch异步接口返回，支持连续输出到Global Memory。

#### 函数原型

| 1   | __aicore__inlinevoidWaitIterateBatch() |
| --- | -------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述 |
| ------ | --------- | ---- |
| 无     | 无        | NA   |

#### 返回值说明

无

#### 约束说明

- 配套IterateBatch或IterateNBatch异步接口使用。
- 仅支持连续输出至Global Memory。
- 当使能MixDualMaster（双主模式）场景时，即模板参数enableMixDualMaster设置为true，不支持使用该接口。

#### 调用示例

| 123456789 | AscendC:Matmul<aType,bType,cType,biasType>mm;mm.SetTensorA(gm_a[offsetA]);mm.SetTensorB(gm_b[offsetB]);if(tiling.isBias){mm.SetBias(gm_bias[offsetBias]);}mm.IterateBatch(gm_c[offsetC],batchA,batchB,false);// do some other compute tasksmm.WaitIterateBatch();// 等待IterateBatch完成 |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
