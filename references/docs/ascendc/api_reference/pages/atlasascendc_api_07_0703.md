# SetBatchInfoForNormal-Matmul Tiling类-Matmul Tiling侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0703
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0703.html
---

# SetBatchInfoForNormal

#### 功能说明

设置A/B矩阵的M/N/K轴信息，以及A/B矩阵的Batch数。Layout类型为NORMAL的场景，调用IterateBatch或者IterateNBatch接口之前，需要在Host侧Tiling实现中通过本接口设置A/B矩阵的M/N/K轴等信息。

#### 函数原型

| 1   | int32_tSetBatchInfoForNormal(int32_tbatchA,int32_tbatchB,int32_tm,int32_tn,int32_tk) |
| --- | ------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名 | 输入/输出 | 描述             |
| ------ | --------- | ---------------- |
| batchA | 输入      | A矩阵的batch数   |
| batchB | 输入      | B矩阵的batch数   |
| m      | 输入      | A矩阵的M轴信息   |
| n      | 输入      | B矩阵的N轴信息   |
| k      | 输入      | A/B矩阵的K轴信息 |

#### 返回值说明

-1表示设置失败；0表示设置成功。

#### 约束说明

Layout类型为NORMAL的场景，调用IterateBatch或者IterateNBatch接口之前，需要在Host侧Tiling实现中通过本接口设置A/B矩阵的M/N/K轴等信息。

#### 调用示例

| 123456789101112131415161718192021 | autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());matmul_tiling:MultiCoreMatmulTilingtiling(ascendcPlatform);int32_tM=32;int32_tN=256;int32_tK=64;tiling->SetDim(1);tiling->SetAType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);tiling->SetBType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);tiling->SetCType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling->SetBiasType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling->SetShape(M,N,K);tiling->SetOrgShape(M,N,K);tiling->SetBias(true);tiling->SetBufferSpace(-1,-1,-1);constexprint32_tBATCH_NUM=3;tiling->SetBatchInfoForNormal(BATCH_NUM,BATCH_NUM,M,N,K);// 设置矩阵排布tiling->SetBufferSpace(-1,-1,-1);optiling:TCubeTilingtilingData;intret=tiling.GetTiling(tilingData); |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
