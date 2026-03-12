# ParamType-OpParamDef-原型注册与管理-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0957
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0957.html
---

# ParamType

#### 功能说明

定义算子参数类型。

#### 函数原型

| 1   | OpParamDef&ParamType(Optionparam_type) |
| --- | -------------------------------------- |

#### 参数说明

| 参数       | 输入/输出 | 说明                                                                              |
| ---------- | --------- | --------------------------------------------------------------------------------- |
| param_type | 输入      | 参数类型，Option取值为：OPTIONAL（可选）、REQUIRED（必选）、DYNAMIC（动态输入）。 |

#### 返回值说明

OpParamDef算子定义，OpParamDef请参考OpParamDef。

#### 约束说明

无
