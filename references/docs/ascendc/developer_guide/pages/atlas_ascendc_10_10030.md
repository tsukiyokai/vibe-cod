# Batch Matmul复用Bias矩阵-特性场景-矩阵编程（高阶API）-SIMD算子实现-算子实践参考-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_10_10030
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_10_10030.html
---

# Batch Matmul复用Bias矩阵

#### 功能介绍

在Batch Matmul场景中，Matmul API可以一次性计算出多个大小为singleCoreM * singleCoreN的C矩阵。当Batch Matmul场景有Bias输入时，默认的Bias输入矩阵包含Batch轴，即Bias的大小为Batch * N。通过开启Bias复用功能，当每个Batch计算使用的Bias数据相同时，只需输入一个不带Batch轴的Bias矩阵。Batch Matmul的Bias矩阵复用功能默认不启用，用户需要设置MatmulConfig中的isBiasBatch参数为false来开启此功能。

![](../images/atlas_ascendc_10_10030_img_001.png)

如上图所示，Batch Matmul中未复用Bias矩阵的场景，每计算出一个singleCoreM * singleCoreN大小的C矩阵，都会与1 * singleCoreN大小的Bias矩阵相加。若不同Batch的计算使用的Bias数据相同，则多Batch计算可以复用同一个Bias矩阵，如下图所示，此场景中调用SetBias接口时，只需设置一个1 * singleCoreN大小的Bias矩阵。

![](../images/atlas_ascendc_10_10030_img_002.png)

#### 使用场景

Batch Matmul中每个Batch的Matmul计算可以使用相同的Bias矩阵。

#### 约束说明

A、B、C矩阵的Layout类型都为NORMAL时，不支持batchMode参数设为SINGLE_LARGE_THAN_L1，即Bias复用场景下，单Batch的A、B矩阵数据总和不得超过L1 Buffer的大小。

#### 调用示例

完整的算子样例请参考BatchMatmul复用Bias算子样例。

| 1234567891011121314 | // 自定义MatmulConfig参数，将其中的isBiasBatch参数设置为false，使能BatchMatmul的Bias复用功能。constexprMatmulConfigModeconfigMode=MatmulConfigMode:CONFIG_NORM;constexprMatmulBatchParamsbatchParams={false,BatchMode:BATCH_LESS_THAN_L1,false/* isBiasBatch */};constexprMatmulConfigCFG_MM=GetMMConfig<configMode>(batchParams);AscendC:Matmul<A_TYPE,B_TYPE,C_TYPE,BIAS_TYPE,CFG_MM>mm;REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);// 初始化matmul对象mm.SetTensorA(gm_a);// 设置左矩阵Amm.SetTensorB(gm_b);// 设置右矩阵Bmm.SetBias(gm_bias);// 设置Bias，矩阵大小为1 * singleCoreNmm.IterateBatch(gm_c,batchA,batchB,false);mm.End(); |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
