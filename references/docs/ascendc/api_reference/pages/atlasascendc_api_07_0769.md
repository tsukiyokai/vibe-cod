# LogSoftMax Tiling-LogSoftMax接口-激活函数-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0769
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0769.html
---

# LogSoftMax Tiling

#### 功能说明

kernel侧LogSoftMax接口的计算需要开发者预留/申请临时空间，以下接口用于在host侧获取预留/申请的最大最小临时空间大小，开发者基于此范围选择合适的空间大小，并调用LogSoftMaxTilingFunc函数获取reduceSize，splitSize等参数，作为Tiling参数传递到kernel侧使用。

- 为保证功能正确，预留/申请的临时空间大小不能小于最小临时空间大小；
- 在最小临时空间-最大临时空间范围内，随着临时空间增大，kernel侧接口计算性能会有一定程度的优化提升。为了达到更好的性能，开发者可以根据实际的内存使用情况进行空间预留/申请。

#### 函数原型

- 获取Kernel接口计算所需最大/最小临时空间的接口1uint32_tGetLogSoftMaxMaxTmpSize(constge:ShapesrcShape,constuint32_tdataTypeSize,constboolisReuseSource)1uint32_tGetLogSoftMaxMinTmpSize(constge:ShapesrcShape,constuint32_tdataTypeSize,constboolisReuseSource)

- Tiling计算接口1voidLogSoftMaxTilingFunc(constge:ShapesrcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,optiling:LogSoftMaxTiling&softmaxTiling)1voidLogSoftMaxTilingFunc(constge:ShapesrcShape,constuint32_tdataTypeSize,constuint32_tlocalWorkSpaceSize,AscendC:tiling:LogSoftMaxTiling&softmaxTiling)

#### 参数说明

| 接口          | 输入/输出 | 功能                                                                    |
| ------------- | --------- | ----------------------------------------------------------------------- |
| srcShape      | 输入      | 输入的shape信息。                                                       |
| dataTypeSize  | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。 |
| isReuseSource | 输入      | 是否复用源操作数输入的空间，与LogSoftMax接口一致。                      |

| 接口               | 输入/输出 | 功能                                                                    |
| ------------------ | --------- | ----------------------------------------------------------------------- |
| srcShape           | 输入      | 输入的shape信息。                                                       |
| dataTypeSize       | 输入      | 输入的数据类型大小，单位为字节。比如输入的数据类型为half，此处应传入2。 |
| localWorkSpaceSize | 输入      | 输入的临时空间大小。                                                    |
| softmaxTiling      | 输出      | 传递到kernel侧使用的Tiling参数。                                        |

#### 返回值说明

GetLogSoftMaxMaxTmpSize/GetLogSoftMaxMinTmpSize接口返回值为最大/最小临时空间。

LogSoftMaxTilingFunc接口无返回值。

#### 约束说明

无

#### 调用示例

| 12345678 | staticge:graphStatusTilingFunc(gert:TilingContext*context){std:vector<int64_t>srcDims={outter,inner};ge:Shapeshape(srcDims);constuint32_ttmpsize=AscendC:GetLogSoftMaxMaxTmpSize(shape,dtypesize,false);AscendC:LogSoftMaxTilingFunc(shape,dtypesize,tmpsize,tiling.logSoftmaxTilingData);...} |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
