# BinaryRepeatParams-其他数据类型-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0013
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0013.html
---

# BinaryRepeatParams

BinaryRepeatParams为用于控制操作数地址步长的数据结构。结构体内包含操作数相邻迭代间相同DataBlock的地址步长，操作数同一迭代内不同DataBlock的地址步长等参数。

相邻迭代间的地址步长参数说明请参考repeatStride；同一迭代内DataBlock的地址步长参数说明请参考dataBlockStride。

结构体具体定义为：

| 1234567891011121314151617181920212223242526 | constint32_tDEFAULT_BLK_NUM=8;constint32_tDEFAULT_BLK_STRIDE=1;constuint8_tDEFAULT_REPEAT_STRIDE=8;structBinaryRepeatParams{__aicore__BinaryRepeatParams(){}__aicore__BinaryRepeatParams(constuint8_tdstBlkStrideIn,constuint8_tsrc0BlkStrideIn,constuint8_tsrc1BlkStrideIn,constuint8_tdstRepStrideIn,constuint8_tsrc0RepStrideIn,constuint8_tsrc1RepStrideIn):dstBlkStride(dstBlkStrideIn),src0BlkStride(src0BlkStrideIn),src1BlkStride(src1BlkStrideIn),dstRepStride(dstRepStrideIn),src0RepStride(src0RepStrideIn),src1RepStride(src1RepStrideIn){}uint32_tblockNumber=DEFAULT_BLK_NUM;uint8_tdstBlkStride=DEFAULT_BLK_STRIDE;uint8_tsrc0BlkStride=DEFAULT_BLK_STRIDE;uint8_tsrc1BlkStride=DEFAULT_BLK_STRIDE;uint8_tdstRepStride=DEFAULT_REPEAT_STRIDE;uint8_tsrc0RepStride=DEFAULT_REPEAT_STRIDE;uint8_tsrc1RepStride=DEFAULT_REPEAT_STRIDE;boolrepeatStrideMode=false;boolstrideSizeMode=false;}; |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

其中，blockNumber，repeatStrideMode和strideSizeMode为保留参数，用户无需关心，使用默认值即可。用户需要自行定义DataBlock Stride参数，包含dstBlkStride，src0BlkStride和src1BlkStride，以及Repeat Stride参数，包含dstRepStride，src0RepStride和src1RepStride。
