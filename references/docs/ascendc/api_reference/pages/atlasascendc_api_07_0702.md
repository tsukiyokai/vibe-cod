# SetBatchNum-Matmul Tiling类-Matmul Tiling侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0702
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0702.html
---

# SetBatchNum

#### 功能说明

设置多Batch计算的最大Batch数，最大Batch数为A矩阵batchA和B矩阵batchB中的最大值。调用IterateBatch接口之前，需要在Host侧Tiling实现中通过本接口设置多Batch计算的Batch数。

#### 函数原型

| 1   | int32_tSetBatchNum(int32_tbatch) |
| --- | -------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                                |
| ------ | --------- | ------------------------------------------------------------------- |
| batch  | 输入      | 多Batch计算的Batch数，Batch数为A矩阵batchA和B矩阵batchB中的最大值。 |

#### 返回值说明

-1表示设置失败；0表示设置成功。

#### 约束说明

调用IterateBatch接口之前，需要在Host侧Tiling实现中通过本接口设置多Batch计算的Batch数。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536 | autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());matmul_tiling:MultiCoreMatmulTilingtiling(ascendcPlatform);int32_tM=32;int32_tN=256;int32_tK=64;tiling->SetDim(1);tiling->SetAType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);tiling->SetBType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);tiling->SetCType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling->SetBiasType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling->SetShape(M,N,K);tiling->SetOrgShape(M,N,K);tiling->SetBias(true);tiling->SetBufferSpace(-1,-1,-1);constexprint32_tA_BNUM=2;constexprint32_tA_SNUM=32;constexprint32_tA_GNUM=3;constexprint32_tA_DNUM=64;constexprint32_tB_BNUM=2;constexprint32_tB_SNUM=256;constexprint32_tB_GNUM=3;constexprint32_tB_DNUM=64;constexprint32_tC_BNUM=2;constexprint32_tC_SNUM=32;constexprint32_tC_GNUM=3;constexprint32_tC_DNUM=256;constexprint32_tBATCH_NUM=3;tiling->SetALayout(A_BNUM,A_SNUM,1,A_GNUM,A_DNUM);tiling->SetBLayout(B_BNUM,B_SNUM,1,B_GNUM,B_DNUM);tiling->SetCLayout(C_BNUM,C_SNUM,1,C_GNUM,C_DNUM);tiling->SetBatchNum(BATCH_NUM);// 设置Batch数tiling->SetBufferSpace(-1,-1,-1);optiling:TCubeTilingtilingData;intret=tiling.GetTiling(tilingData); |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
