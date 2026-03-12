# SoftMax/SimpleSoftMax Tiling-SoftMax接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0762
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0762.html
---

# SoftMax/SimpleSoftMax Tiling

#### 功能说明

用于获取SoftMax/SimpleSoftMax Tiling参数。

#### 函数原型

- 获取Kernel接口计算所需最大/最小临时空间的接口1uint32_tGetSoftMaxMaxTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisReuseSource)1uint32_tGetSoftMaxMinTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisReuseSource)

- Tiling计算接口AscendC:optiling命名空间下的计算接口1voidSoftMaxTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,optiling:SoftMaxTiling&softmaxTiling)AscendC命名空间下的计算接口1voidSoftMaxTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,AscendC:tiling:SoftMaxTiling&softmaxTiling)

#### 参数说明

| 接口          | 输入/输出 | 功能                                       |
| ------------- | --------- | ------------------------------------------ |
| srcShape      | 输入      | 输入srcTensor的shape信息。                 |
| dataTypeSize  | 输入      | 参与计算的max和sum的数据类型，比如half=2。 |
| isReuseSource | 输入      | 与kernel侧接口配置保持一致。               |

| 接口               | 输入/输出 | 功能                                                                                                                                        |
| ------------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape           | 输入      | 输入srcTensor的shape信息。                                                                                                                  |
| dataTypeSize       | 输入      | 参与计算的max和sum的数据类型，比如half=2。                                                                                                  |
| localWorkSpaceSize | 输入      | 剩余的可供SoftMax接口计算的空间大小，单位为Byte。localWorkSpaceSize的取值必须大于GetSoftMaxMinTmpSize接口返回的计算所需的最小临时空间大小。 |
| softmaxTiling      | 输出      | 输出SoftMax接口所需的tiling信息，支持optiling:SoftMaxTiling形式入参和AscendC:tiling:SoftMaxTiling形式入参。                                 |

#### 返回值说明

GetSoftMaxMaxTmpSize返回SoftMax/SimpleSoftMax接口能完成计算所需最大临时空间大小，单位为Byte。

GetSoftMaxMinTmpSize返回SoftMax/SimpleSoftMax接口能完成计算所需最小临时空间大小，单位为Byte。

#### 约束说明

无
