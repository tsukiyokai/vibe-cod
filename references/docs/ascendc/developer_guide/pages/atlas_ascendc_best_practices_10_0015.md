# 高效的使用搬运API-内存访问-SIMD算子性能优化-算子实践参考-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_best_practices_10_0015
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_best_practices_10_0015.html
---

# 高效的使用搬运API

【优先级】高

【描述】在使用搬运API时，应该尽可能地通过配置搬运控制参数实现连续搬运或者固定间隔搬运，避免使用for循环，二者效率差距极大。如下图示例，图片的每一行为16KB，需要从每一行中搬运前2KB，针对这种场景，使用for循环遍历每行，每次仅能搬运2KB。若直接配置DataCopyParams参数（包含srcStride/dstStride/blockLen/blockCount），则可以达到一次搬完的效果，每次搬运32KB；参考尽量一次搬运较大的数据块章节介绍的搬运数据量和实际带宽的关系，建议一次搬完。

![](../images/atlas_ascendc_best_practices_10_0015_img_001.png)

| 1234567891011 | // 搬运数据存在间隔，从GM上每行16KB中搬运2KB数据，共16行LocalTensor<float>tensorIn;GlobalTensor<float>tensorGM;...constexprint32_tcopyWidth=2*1024/sizeof(float);constexprint32_timgWidth=16*1024/sizeof(float);constexprint32_timgHeight=16;// 使用for循环，每次只能搬运2K，重复16次for(inti=0;i<imgHeight;i++){DataCopy(tensorIn[i*copyWidth],tensorGM[i*imgWidth],copyWidth);} |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

【正例】

| 12345678910111213 | LocalTensor<float>tensorIn;GlobalTensor<float>tensorGM;...constexprint32_tcopyWidth=2*1024/sizeof(float);constexprint32_timgWidth=16*1024/sizeof(float);constexprint32_timgHeight=16;// 通过DataCopy包含DataCopyParams的接口一次搬完DataCopyParamscopyParams;copyParams.blockCount=imgHeight;copyParams.blockLen=copyWidth/8;// 搬运的单位为DataBlock(32Byte)，每个DataBlock内有8个floatcopyParams.srcStride=(imgWidth-copyWidth)/8;// 表示两次搬运src之间的间隔，单位为DataBlockcopyParams.dstStride=0;// 连续写，两次搬运之间dst的间隔为0，单位为DataBlockDataCopy(tensorGM,tensorIn,copyParams); |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
