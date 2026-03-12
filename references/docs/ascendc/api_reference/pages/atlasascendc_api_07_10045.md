# SetDebugMode-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10045
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10045.html
---

# SetDebugMode

#### 功能说明

设置调测模式。

#### 函数原型

| 1   | uint32_tSetDebugMode(uint8_tdebugMode) |
| --- | -------------------------------------- |

#### 参数说明

| 参数名    | 输入/输出 | 描述                                                                                                                                                                                                                                                               |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| debugMode | 输入      | 表示选择的调测模式，uint8_t类型。参数支持的取值如下：1：关闭HCCL高阶API通信功能。2：打印消息队列和Prepare消息执行次数等信息。3：打印Prepare消息源数据buffer和目的数据buffer中的数据。4：打印AI CPU服务端执行通信任务的各阶段时间戳和耗时，每执行30个算子打印一次。 |

#### 返回值说明

- 0表示设置成功。
- 非0表示设置失败。

#### 约束说明

无

#### 调用示例

| 12345678 | constchar*groupName="testGroup";uint32_topType=HCCL_CMD_REDUCE_SCATTER;std:stringalgConfig="ReduceScatter=level0:fullmesh";uint32_treduceType=HCCL_REDUCE_SUM;AscendC:Mc2CcTilingConfigmc2CcTilingConfig(groupName,opType,algConfig,reduceType);mc2CcTilingConfig.SetDebugMode(3);// 设置调测模式mc2CcTilingConfig.GetTiling(tiling->mc2InitTiling);mc2CcTilingConfig.GetTiling(tiling->reduceScatterTiling); |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
