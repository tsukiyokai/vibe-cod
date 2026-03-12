# GetTiling-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10037
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10037.html
---

# GetTiling

#### 功能说明

获取Mc2InitTiling参数和Mc2CcTiling参数。

#### 函数原型

| 1   | uint32_tGetTiling(:Mc2InitTiling&tiling) |
| --- | ---------------------------------------- |

| 1   | uint32_tGetTiling(:Mc2CcTiling&tiling) |
| --- | -------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                           |
| ------ | --------- | ------------------------------ |
| tiling | 输出      | Tiling结构体存储的Tiling信息。 |

#### 返回值说明

- 返回值为0，则tiling计算成功，该Tiling结构体的值可以用于后续计算。
- 返回值非0，则tiling计算失败，该Tiling结果无法使用。

#### 约束说明

无

#### 调用示例

| 1234567 | constchar*groupName="testGroup";uint32_topType=HCCL_CMD_REDUCE_SCATTER;std:stringalgConfig="ReduceScatter=level0:fullmesh";uint32_treduceType=HCCL_REDUCE_SUM;AscendC:Mc2CcTilingConfigmc2CcTilingConfig(groupName,opType,algConfig,reduceType);mc2CcTilingConfig.GetTiling(tiling->mc2InitTiling);// tiling为算子组装的TilingData结构体，获取Mc2InitTilingmc2CcTilingConfig.GetTiling(tiling->reduceScatterTiling);// tiling为算子组装的TilingData结构体，获取Mc2CcTiling |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
