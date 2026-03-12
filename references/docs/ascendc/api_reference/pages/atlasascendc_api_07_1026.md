# PlatformAscendC简介-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1026
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1026.html
---

# PlatformAscendC简介

在实现Host侧的Tiling函数时，可能需要获取一些硬件平台的信息，来支撑Tiling的计算，比如获取硬件平台的核数等信息。PlatformAscendC类提供获取这些平台信息的功能。

#### 需要包含的头文件

使用该功能需要包含"tiling/platform/platform_ascendc.h"头文件。样例如下：

| 1   | #include"tiling/platform/platform_ascendc.h" |
| --- | -------------------------------------------- |

#### Public成员函数
