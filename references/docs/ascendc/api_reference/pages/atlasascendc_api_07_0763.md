# SoftmaxFlash Tiling接口-SoftMax接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0763
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0763.html
---

# SoftmaxFlash Tiling接口

#### 功能说明

注意：该接口后续即将废弃，新开发内容不要使用该接口。

用于获取SoftmaxFlash Tiling参数。

#### 函数原型

- 获取Kernel接口计算所需最小/最大临时空间的接口1uint32_tGetSoftMaxFlashMaxTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisUpdate,constboolisReuseSource)1uint32_tGetSoftMaxFlashMinTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisUpdate,constboolisReuseSource)

- Tiling计算接口AscendC:optiling命名空间下的计算接口1voidSoftMaxFlashTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,optiling:SoftMaxTiling&softmaxFlashTiling,constboolisUpdate=false)AscendC命名空间下的计算接口1voidSoftMaxFlashTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,AscendC:tiling:SoftMaxTiling&softmaxFlashTiling,constboolisUpdate=false)

#### 参数说明

| 接口          | 输入/输出 | 功能                                                          |
| ------------- | --------- | ------------------------------------------------------------- |
| srcShape      | 输入      | 输入srcTensor的shape信息。                                    |
| dataTypeSize  | 输入      | 参与计算的maxTensor和sumTensor的数据类型，比如half=2。        |
| isUpdate      | 输入      | 是否使能刷新功能，和kernel侧SoftmaxFlash接口一致，默认false。 |
| isReuseSource | 输入      | 与kernel侧接口配置保持一致。                                  |

| 接口               | 输入/输出 | 功能                                                                                                                                                  |
| ------------------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape           | 输入      | 输入srcTensor的shape信息。                                                                                                                            |
| dataTypeSize       | 输入      | 参与计算的maxTensor和sumTensor的数据类型，比如half=2。                                                                                                |
| localWorkSpaceSize | 输入      | 剩余的可供SoftmaxFlash接口计算的空间大小，单位为Byte。localWorkSpaceSize的取值必须大于GetSoftMaxFlashMinTmpSize接口返回的计算所需的最小临时空间大小。 |
| isUpdate           | 输入      | 是否使能刷新功能，和kernel侧SoftmaxFlash接口一致，默认false。                                                                                         |
| softmaxFlashTiling | 输出      | 输出SoftmaxFlash接口所需的tiling信息，支持optiling:SoftMaxTiling形式入参和AscendC:tiling:SoftMaxTiling形式入参。                                      |

#### 返回值说明

GetSoftMaxFlashMaxTmpSize返回SoftmaxFlash接口能完成计算所需最大临时空间大小，单位为Byte。

GetSoftMaxFlashMinTmpSize返回SoftmaxFlash接口能完成计算所需最小临时空间大小，单位为Byte。

#### 约束说明

无
