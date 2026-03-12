# SetGroupName-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10039
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10039.html
---

# SetGroupName

#### 功能说明

设置通信任务所在的通信域。

#### 函数原型

| 1   | uint32_tSetGroupName(conststd:string&groupName) |
| --- | ----------------------------------------------- |

#### 参数说明

| 参数名    | 输入/输出 | 描述                                                            |
| --------- | --------- | --------------------------------------------------------------- |
| groupName | 输入      | 当前通信任务所在的通信域。string类型，支持的最大长度为128字节。 |

#### 返回值说明

- 0表示设置成功。
- 非0表示设置失败。

#### 约束说明

无

#### 调用示例

本接口的调用示例请见调用示例。
