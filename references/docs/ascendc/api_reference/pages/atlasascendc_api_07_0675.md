# SetBType-Matmul Tiling类-Matmul Tiling侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0675
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0675.html
---

# SetBType

#### 功能说明

设置B矩阵的位置，数据格式，数据类型，是否转置等信息，这些信息需要和kernel侧的设置保持一致。

#### 函数原型

| 1   | int32_tSetBType(TPositionpos,CubeFormattype,DataTypedataType,boolisTrans=false) |
| --- | ------------------------------------------------------------------------------- |

#### 参数说明

| 参数名   | 输入/输出 | 描述                                                                                                                       |
| -------- | --------- | -------------------------------------------------------------------------------------------------------------------------- |
| pos      | 输入      | B矩阵所在的buffer位置，可设置为：TPosition:GM, TPosition:VECOUT, TPosition:TSCM。                                          |
| type     | 输入      | B矩阵的数据格式，可设置为：CubeFormat:ND，CubeFormat:NZ。                                                                  |
| dataType | 输入      | B矩阵的数据类型，可设置为：DataType:DT_FLOAT/DataType:DT_FLOAT16/DataType:DT_BFLOAT16 /DataType:DT_INT8/DataType:DT_INT4。 |
| isTrans  | 输入      | B矩阵是否转置。参数取值：true：B矩阵转置；false：B矩阵不转置。                                                             |

#### 返回值说明

-1表示设置失败；0表示设置成功。

#### 约束说明

无

#### 调用示例

| 12345678910111213 | autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());matmul_tiling:MatmulApiTilingtiling(ascendcPlatform);tiling.SetAType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);// 设置B矩阵，buffer位置为GM，数据格式为ND，数据类型为bfloat16，默认不转置tiling.SetBType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT16);tiling.SetCType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling.SetBiasType(matmul_tiling:TPosition:GM,matmul_tiling:CubeFormat:ND,matmul_tiling:DataType:DT_FLOAT);tiling.SetShape(1024,1024,1024);tiling.SetOrgShape(1024,1024,1024);tiling.SetBias(true);tiling.SetBufferSpace(-1,-1,-1);optiling:TCubeTilingtilingData;intret=tiling.GetTiling(tilingData); |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
