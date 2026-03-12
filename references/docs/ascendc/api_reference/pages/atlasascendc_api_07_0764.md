# SoftmaxGrad Tiling接口-SoftMax接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0764
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0764.html
---

# SoftmaxGrad Tiling接口

#### 功能说明

用于获取SoftmaxGrad Tiling参数。

#### 函数原型

- 获取Kernel接口计算所需最小/最大临时空间的接口1uint32_tGetSoftMaxGradMaxTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisFront,constboolisReuseSource)1uint32_tGetSoftMaxGradMinTmpSize(constge:Shape&srcShape,constuint32_tdataTypeSize,constboolisFront,constboolisReuseSource)

- Tiling计算接口AscendC:optiling命名空间下的计算接口1voidSoftMaxGradTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,optiling:SoftMaxTiling&softmaxGradTiling,constboolisFront=false)AscendC命名空间下的计算接口1voidSoftMaxGradTilingFunc(constge:Shape&srcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,AscendC:tiling:SoftMaxTiling&softmaxGradTiling,constboolisFront=false)

#### 参数说明

| 接口          | 输入/输出 | 功能                                                     |
| ------------- | --------- | -------------------------------------------------------- |
| srcShape      | 输入      | 输入srcTensor的shape信息。                               |
| dataTypeSize  | 输入      | 计算的数据类型，比如half=2。                             |
| isFront       | 输入      | 是否只计算，和kernel侧的SoftmaxGrad接口一致，默认false。 |
| isReuseSource | 输入      | 与kernel侧接口配置保持一致。                             |

| 接口               | 输入/输出 | 功能                                                                                                                                                    |
| ------------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| srcShape           | 输入      | 输入srcTensor的shape信息。                                                                                                                              |
| localWorkSpaceSize | 输入      | 剩余的可供SoftmaxGrad接口计算的临时空间大小，单位为Byte。localWorkSpaceSize的取值必须大于GetSoftMaxGradMinTmpSize接口返回的计算所需的最小临时空间大小。 |
| dataTypeSize       | 输入      | 计算的数据类型，比如half=2。                                                                                                                            |
| isFront            | 输入      | 是否只计算，和kernel侧的SoftmaxGrad接口一致，默认false。                                                                                                |
| softmaxGradTiling  | 输出      | 输出SoftmaxGrad接口所需的tiling信息，支持optiling:SoftMaxTiling形式入参和AscendC:tiling:SoftMaxTiling形式入参。                                         |

#### 返回值说明

GetSoftMaxGradMinTmpSize返回SoftmaxGrad接口能完成计算所需最小临时空间大小，单位为Byte。

GetSoftMaxGradMaxTmpSize返回SoftmaxGrad接口能完成计算所需最大临时空间大小，单位为Byte。

#### 约束说明

无
