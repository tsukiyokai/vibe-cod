# SetCommBlockNum-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10202
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10202.html
---

# SetCommBlockNum

#### 功能说明

设置参与BatchWrite通信的核数。

#### 函数原型

| 1   | uint32_tSetCommBlockNum(uint16_tnum) |
| --- | ------------------------------------ |

#### 参数说明

| 参数名 | 输入/输出 | 描述           |
| ------ | --------- | -------------- |
| num    | 输入      | 表示核的数量。 |

#### 返回值说明

- 0表示设置成功。
- 非0表示设置失败。

#### 约束说明

本接口仅在Atlas A3 训练系列产品/Atlas A3 推理系列产品上通信类型为HCCL_CMD_BATCH_WRITE时生效。

#### 调用示例

| 123456 | constchar*groupName="testGroup";uint32_topType=HCCL_CMD_BATCH_WRITE;std:stringalgConfig="BatchWrite=level0:fullmesh";uint32_treduceType=HCCL_REDUCE_SUM;AscendC:Mc2CcTilingConfigmc2CcTilingConfig(groupName,opType,algConfig,reduceType);mc2CcTilingConfig.SetCommBlockNum(24U); |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
