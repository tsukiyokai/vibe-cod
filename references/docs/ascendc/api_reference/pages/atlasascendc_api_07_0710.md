# MatmulGetTmpBufSize-获取Matmul计算所需空间-Matmul Tiling侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0710
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0710.html
---

# MatmulGetTmpBufSize

#### 功能说明

本接口用于在调用GetTiling接口获取Tiling参数后，根据Tiling结构体信息获取L1 Buffer/Unified Buffer/L0C Buffer的使用大小。

#### 函数原型

| 1   | int32_tMatmulGetTmpBufSize(optiling:TCubeTiling&tiling,matmul_tiling:SysTilingTempBufSize&bufSize) |
| --- | -------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名  | 输入/输出                                                                                                                            | 描述                                                                                                                                                                                                                               |       |                                                                                                                                      |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------ |
| tiling  | 输入                                                                                                                                 | Matmul单核Tiling的结构体，即MatmulTiling对象得到的TCubeTiling结构体。                                                                                                                                                              |       |                                                                                                                                      |
| bufSize | 输出                                                                                                                                 | Tiling中L1 Buffer/Unified Buffer/L0C Buffer的使用大小。SysTilingTempBufSize结构定义如下：12345structSysTilingTempBufSize{int32_tubSize=0;// Unified Buffer大小int32_tl1Size=0;// L1 Buffer大小int32_tl0cSize=0;// L0C Buffer大小}; | 12345 | structSysTilingTempBufSize{int32_tubSize=0;// Unified Buffer大小int32_tl1Size=0;// L1 Buffer大小int32_tl0cSize=0;// L0C Buffer大小}; |
| 12345   | structSysTilingTempBufSize{int32_tubSize=0;// Unified Buffer大小int32_tl1Size=0;// L1 Buffer大小int32_tl0cSize=0;// L0C Buffer大小}; |                                                                                                                                                                                                                                    |       |                                                                                                                                      |

#### 返回值说明

-1表示获取失败；0表示获取成功。

#### 约束说明

无

#### 调用示例

| 1234567 | autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());matmul_tiling:MatmulApiTilingtiling(ascendcPlatform);optiling:TCubeTilingtilingData;...// 初始化tilingData，详见MatmulTiling类使用说明intret=tiling.GetTiling(tilingData);// 获取Tiling参数SysTilingTempBufSizebufSize;MatmulGetTmpBufSize(tilingData,bufSize); |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
