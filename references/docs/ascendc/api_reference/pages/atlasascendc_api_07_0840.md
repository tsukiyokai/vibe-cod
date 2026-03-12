# GetConcatTmpSize-排序操作-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0840
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0840.html
---

# GetConcatTmpSize

#### 功能说明

获取Concat接口所需的临时空间大小。

#### 函数原型

| 1   | uint32_tGetConcatTmpSize(constplatform_ascendc:PlatformAscendC&ascendcPlatform,constuint32_telemCount,constuint32_tdataTypeSize) |
| --- | -------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名          | 输入/输出 | 描述                                                          |
| --------------- | --------- | ------------------------------------------------------------- |
| ascendcPlatform | 输入      | 传入硬件平台的信息，PlatformAscendC定义请参见构造及析构函数。 |
| elemCount       | 输入      | 输入元素个数。                                                |
| dataTypeSize    | 输入      | 输入数据大小（单位为字节）。                                  |

#### 返回值说明

Concat接口所需的临时空间大小。

#### 约束说明

无

#### 调用示例

| 1234 | fe:PlatFormInfosplatform_info;autoplat=platform_ascendc:PlatformAscendC(&platform_info);constuint32_telemCount=128;AscendC:GetConcatTmpSize(plat,elemCount,2); |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
