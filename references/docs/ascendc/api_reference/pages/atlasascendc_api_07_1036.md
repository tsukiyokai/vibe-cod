# GetLibApiWorkSpaceSize-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1036
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1036.html
---

# GetLibApiWorkSpaceSize

#### 功能说明

获取AscendC API需要的workspace空间大小。

#### 函数原型

| 1   | uint32_tGetLibApiWorkSpaceSize(void)const |
| --- | ----------------------------------------- |

#### 参数说明

无

#### 返回值说明

返回uint32_t数据类型的结果，该结果代表当前系统workspace的大小，单位为字节。

#### 约束说明

无

#### 调用示例

| 12345678910111213 | // 用户自定义的tiling函数staticge:graphStatusTilingFunc(gert:TilingContext*context){AddApiTilingtiling;...size_tusrSize=256;// 设置用户需要使用的workspace大小。// 如需要使用系统workspace需要调用GetLibApiWorkSpaceSize获取系统workspace的大小。autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint32_tsysWorkspaceSize=ascendcPlatform.GetLibApiWorkSpaceSize();size_t*currentWorkspace=context->GetWorkspaceSizes(1);// 通过框架获取workspace的指针，GetWorkspaceSizes入参为所需workspace的块数。当前限制使用一块。currentWorkspace[0]=usrSize+sysWorkspaceSize;// 设置总的workspace的数值大小，总的workspace空间由框架来申请并管理。...} |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
