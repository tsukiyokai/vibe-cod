# GetResGroupBarrierWorkSpaceSize-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1037
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1037.html
---

# GetResGroupBarrierWorkSpaceSize

#### 功能说明

获取GroupBarrier所需要的workspace空间大小。

#### 函数原型

| 1   | uint32_tGetResGroupBarrierWorkSpaceSize(void)const |
| --- | -------------------------------------------------- |

#### 参数说明

无

#### 返回值说明

当前GroupBarrier所需要的workspace大小。

#### 约束说明

无。

#### 调用示例

| 1234567891011121314 | // 用户自定义的tiling函数staticge:graphStatusTilingFunc(gert:TilingContext*context){AddApiTilingtiling;...// 如需要使用系统workspace需要调用GetLibApiWorkSpaceSize获取系统workspace的大小。autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint32_tsysWorkspaceSize=ascendcPlatform.GetLibApiWorkSpaceSize();// 设置用户需要使用的workspace和GroupBarrier需要的大小作为usrWorkspace的总大小。size_tusrSize=256+ascendcPlatform.GetResGroupBarrierWorkSpaceSize();// 设置用户需要使用的workspace大小。size_t*currentWorkspace=context->GetWorkspaceSizes(1);// 通过框架获取workspace的指针，GetWorkspaceSizes入参为所需workspace的块数。当前限制使用一块。currentWorkspace[0]=usrSize+sysWorkspaceSize;// 设置总的workspace的数值大小，总的workspace空间由框架来申请并管理。...} |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
