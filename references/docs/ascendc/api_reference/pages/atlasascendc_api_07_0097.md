# ResetMask-掩码操作-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0097
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0097.html
---

# ResetMask

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

恢复mask的值为默认值（全1），表示矢量计算中每次迭代内的所有元素都将参与运算。

#### 函数原型

| 1   | __aicore__inlinevoidResetMask() |
| --- | ------------------------------- |

#### 参数说明

无

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

用SetVectorMask设置mask值并使用完成后，使用ResetMask恢复mask的值为默认值。

| 12  | AscendC:SetVectorMask<half,AscendC:MaskMode:NORMAL>(128);AscendC:ResetMask(); |
| --- | ----------------------------------------------------------------------------- |
