# SetStepSize-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10042
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10042.html
---

# SetStepSize

#### 功能说明

设置细粒度通信时，通信算法的步长，即设置细粒度通信时，一次子通信任务执行或准备执行的通信算法的步骤数。例如，图1使用pairwise算法的AlltoAllV通信步骤示意图中，该细粒度通信场景下，AlltoAllV通信任务的通信步长为1。

#### 函数原型

| 1   | uint32_tSetStepSize(uint8_tstepSize) |
| --- | ------------------------------------ |

#### 参数说明

| 参数名   | 输入/输出 | 描述                                                      |
| -------- | --------- | --------------------------------------------------------- |
| stepSize | 输入      | 设置细粒度通信时，每次通信的步长。0表示当前非细粒度通信。 |

#### 返回值说明

- 0表示设置成功。
- 非0表示设置失败。

#### 约束说明

Atlas A2 训练系列产品/Atlas A2 推理系列产品暂不支持该接口。

#### 调用示例

| 1234567891011 | staticge:graphStatusAllToAllVCustomTilingFunc(gert:TilingContext*context){AllToAllVCustomV3TilingData*tiling=context->GetTilingData<AllToAllVCustomV3TilingData>();conststd:stringgroupName="testGroup";conststd:stringalgConfig="AlltoAll=level0:fullmesh;level1:pairwise";AscendC:Mc2CcTilingConfigmc2CcTilingConfig(groupName,HCCL_CMD_ALLTOALLV,algConfig,0);mc2CcTilingConfig.SetStepSize(1U);mc2CcTilingConfig.GetTiling(tiling->mc2InitTiling);mc2CcTilingConfig.GetTiling(tiling->mc2CcTiling);returnge:GRAPH_SUCCESS;} |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
