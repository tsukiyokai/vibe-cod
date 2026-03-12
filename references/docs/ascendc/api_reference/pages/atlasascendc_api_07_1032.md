# GetCoreNumVector-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1032
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1032.html
---

# GetCoreNumVector

#### 功能说明

用于获取硬件平台独立的Vector Core的核数。

该接口仅在Atlas推理系列产品有效，其他硬件平台型号均返回0。

#### 函数原型

| 1   | uint32_tGetCoreNumVector(void)const |
| --- | ----------------------------------- |

#### 参数说明

无

#### 返回值说明

返回硬件平台Vector Core的核数。

#### 约束说明

Atlas训练系列产品，不支持该接口，返回0

Atlas推理系列产品，支持该接口，返回硬件平台Vector Core的核数

Atlas A2 训练系列产品/Atlas A2 推理系列产品不支持该接口，返回0

Atlas A3 训练系列产品/Atlas A3 推理系列产品不支持该接口，返回0

Atlas 200I/500 A2 推理产品不支持该接口，返回0

#### 调用示例

| 12345678 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());autoaivCoreNum=ascendcPlatform.GetCoreNumAiv();autovectorCoreNum=ascendcPlatform.GetCoreNumVector();autoallVecCoreNums=aivCoreNum+vectorCoreNum;// ...按照allVecCoreNums切分returnret;} |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
