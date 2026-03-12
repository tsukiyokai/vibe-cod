# UnaryRepeatParams-其他数据类型-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0012
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0012.html
---

# UnaryRepeatParams

UnaryRepeatParams为用于控制操作数地址步长的数据结构。结构体内包含操作数相邻迭代间相同DataBlock的地址步长，操作数同一迭代内不同DataBlock的地址步长等参数。

相邻迭代间的地址步长参数说明请参考repeatStride；同一迭代内DataBlock的地址步长参数说明请参考dataBlockStride。

结构体具体定义为：

| 123456789101112131415161718192021222324252627282930 | constint32_tDEFAULT_BLK_NUM=8;constint32_tDEFAULT_BLK_STRIDE=1;constuint8_tDEFAULT_REPEAT_STRIDE=8;structUnaryRepeatParams{__aicore__UnaryRepeatParams(){}__aicore__UnaryRepeatParams(constuint16_tdstBlkStrideIn,constuint16_tsrcBlkStrideIn,constuint8_tdstRepStrideIn,constuint8_tsrcRepStrideIn):dstBlkStride(dstBlkStrideIn),srcBlkStride(srcBlkStrideIn),dstRepStride(dstRepStrideIn),srcRepStride(srcRepStrideIn){}__aicore__UnaryRepeatParams(constuint16_tdstBlkStrideIn,constuint16_tsrcBlkStrideIn,constuint8_tdstRepStrideIn,constuint8_tsrcRepStrideIn,constboolhalfBlockIn):dstBlkStride(dstBlkStrideIn),srcBlkStride(srcBlkStrideIn),dstRepStride(dstRepStrideIn),srcRepStride(srcRepStrideIn),halfBlock(halfBlockIn){}uint32_tblockNumber=DEFAULT_BLK_NUM;uint16_tdstBlkStride=DEFAULT_BLK_STRIDE;uint16_tsrcBlkStride=DEFAULT_BLK_STRIDE;uint8_tdstRepStride=DEFAULT_REPEAT_STRIDE;uint8_tsrcRepStride=DEFAULT_REPEAT_STRIDE;boolrepeatStrideMode=false;boolstrideSizeMode=false;boolhalfBlock=false;}; |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

其中，blockNumber，repeatStrideMode，strideSizeMode为保留参数，用户无需关心，使用默认值即可。halfBlock表示CastDeq指令的结果写入对应UB的上半(halfBlock = true)还是下半(halfBlock = false)部分。用户需要自行定义DataBlock Stride参数，包含dstBlkStride，srcBlkStride，以及Repeat Stride参数，包含dstRepStride，srcRepStride。
