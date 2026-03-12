# WaitIterateAll-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0641
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0641.html
---

# WaitIterateAll

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

等待IterateAll异步接口返回，支持连续输出到Global Memory。

#### 函数原型

| 1   | __aicore__inlinevoidWaitIterateAll() |
| --- | ------------------------------------ |

#### 参数说明

无

#### 返回值说明

无

#### 约束说明

- 配套IterateAll异步接口使用。
- 仅支持连续输出至Global Memory。

#### 调用示例

更多本接口的使用样例请参考IterateAll异步场景矩阵乘法。

| 123456789 | AscendC:Matmul<aType,bType,cType,biasType>mm;mm.SetTensorA(gm_a[offsetA]);mm.SetTensorB(gm_b[offsetA]);if(tiling.isBias){mm.SetBias(gm_bias[offsetBias]);}mm.templateIterateAll<false>(gm_c[offsetC],0,false,true);// do some others computemm.WaitIterateAll();// 等待IterateAll完成 |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
