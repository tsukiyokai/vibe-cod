# GetCoreNum-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1028
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1028.html
---

# GetCoreNum

#### 功能说明

获取当前硬件平台的核数。若AI Core的架构为Cube、Vector分离模式，返回Vector Core的核数；耦合模式返回AI Core的核数。

#### 函数原型

| 1   | uint32_tGetCoreNum(void)const |
| --- | ----------------------------- |

#### 参数说明

无

#### 返回值说明

Atlas训练系列产品，耦合模式，返回AI Core的核数

Atlas推理系列产品，耦合模式，返回AI Core的核数

Atlas A2 训练系列产品/Atlas A2 推理系列产品，分离模式，返回Vector Core的核数

Atlas A3 训练系列产品/Atlas A3 推理系列产品，分离模式，返回Vector Core的核数

#### 约束说明

无

#### 调用示例

| 1234567 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());autocoreNum=ascendcPlatform.GetCoreNum();// ... 根据核数自行设计Tiling策略context->SetBlockDim(coreNum);returnret;} |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
