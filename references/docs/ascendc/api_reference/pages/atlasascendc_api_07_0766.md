# IsBasicBlockInSoftMax-SoftMax接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0766
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0766.html
---

# IsBasicBlockInSoftMax

#### 功能说明

用于判断SoftMaxTiling结构是否符合基本块特征。

#### 函数原型

- AscendC:optiling命名空间下的计算接口1boolIsBasicBlockInSoftMax(optiling:SoftMaxTiling&tiling,constuint32_tdataTypeSize=2)
- AscendC命名空间下的计算接口1boolIsBasicBlockInSoftMax(AscendC:tiling:SoftMaxTiling&tiling,constuint32_tdataTypeSize=2)

#### 参数说明

| 接口         | 输入/输出 | 功能                                                                                                  |
| ------------ | --------- | ----------------------------------------------------------------------------------------------------- |
| tiling       | 输入      | 待判断的SoftMaxTiling结构，支持optiling:SoftMaxTiling形式入参和AscendC:tiling:SoftMaxTiling形式入参。 |
| dataTypeSize | 输入      | 参与计算的srcTensor的数据类型大小，比如half=2。                                                       |

#### 返回值说明

- 返回true表示SoftMaxTiling结构满足基本块Tiling特征。
- 返回false表示SoftMaxTiling结构不满足基本块Tiling特征。

#### 约束说明

无
