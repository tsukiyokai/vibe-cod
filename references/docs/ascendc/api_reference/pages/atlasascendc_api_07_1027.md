# 构造及析构函数-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1027
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1027.html
---

# 构造及析构函数

#### 功能说明

在实现Host侧的Tiling函数时，可能需要获取一些硬件平台的信息，来支撑Tiling的计算，比如获取硬件平台的核数等信息。PlatformAscendC类提供获取这些平台信息的功能。

#### 函数原型

| 123 | PlatformAscendC()=delete~PlatformAscendC()=defaultexplicitPlatformAscendC(fe:PlatFormInfos*platformInfo):platformInfo_(platformInfo){} |
| --- | -------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数         | 输入/输出 | 说明                                                  |
| ------------ | --------- | ----------------------------------------------------- |
| platformlnfo | 输入      | platformInfo结构体，通过GetPlatformInfo接口可以获取。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 1234567891011 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint64_tub_size,l1_size;ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:UB,ub_size);ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:L1,l1_size);autoaicNum=ascendcPlatform.GetCoreNumAic();autoaivNum=ascendcPlatform.GetCoreNumAiv();// ... 按照aivNum切分context->SetBlockDim(ascendcPlatform.CalcTschBlockDim(aivNum,aicNum,aivNum));returnret;} |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
