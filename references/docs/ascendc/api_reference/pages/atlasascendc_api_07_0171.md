# SetSysWorkSpace-workspace-临时空间管理-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0171
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0171.html
---

# SetSysWorkSpace

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

框架需要使用的workspace称之为系统workspace。Matmul Kernel侧接口等高阶API需要系统workspace，所以在使用该类API时，需要调用该接口，设置系统workspace的指针。采用工程化算子开发方式或者kernel直调方式（开启HAVE_WORKSPACE编译选项）时，不需要开发者手动设置，框架会自动设置。其他场景下，需要开发者调用SetSysWorkSpace进行设置。

在kernel侧调用该接口前，需要在host侧调用GetLibApiWorkSpaceSize获取系统workspace的大小，并在host侧设置workspacesize大小。样例如下：

| 12345678910111213 | // 用户自定义的tiling函数staticge:graphStatusTilingFunc(gert:TilingContext*context){AddApiTilingtiling;...size_tusrSize=256;// 设置用户需要使用的workspace大小。// 如需要使用系统workspace需要调用GetLibApiWorkSpaceSize获取系统workspace的大小。autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint32_tsysWorkspaceSize=ascendcPlatform.GetLibApiWorkSpaceSize();size_t*currentWorkspace=context->GetWorkspaceSizes(1);// 通过框架获取workspace的指针，GetWorkspaceSizes入参为所需workspace的块数。当前限制使用一块。currentWorkspace[0]=usrSize+sysWorkspaceSize;// 设置总的workspace的数值大小，总的workspace空间由框架来申请并管理。...} |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### 函数原型

| 1   | __aicore__inlinevoidSetSysWorkspace(GM_ADDRworkspace) |
| --- | ----------------------------------------------------- |

#### 参数说明

| 参数名称  | 输入/输出 | 描述                                                                  |
| --------- | --------- | --------------------------------------------------------------------- |
| workspace | 输入      | 核函数传入的workspace的指针，包括系统workspace和用户使用的workspace。 |

#### 约束说明

无

#### 返回值说明

无

#### 调用示例

| 1234567891011 | template<typenameaType,typenamebType,typenamecType,typenamebiasType>__aicore__inlinevoidMatmulLeakyKernel<aType,bType,cType,biasType>:Init(GM_ADDRa,GM_ADDRb,GM_ADDRbias,GM_ADDRc,GM_ADDRworkspace,constTCubeTiling&tiling,floatalpha){// 融合算子的初始化操作// ...AscendC:SetSysWorkspace(workspace);if(GetSysWorkSpacePtr()==nullptr){return;}} |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
