# GetDataBlockSizeInBytes-系统变量访问-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0186
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0186.html
---

# GetDataBlockSizeInBytes

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

获取当前芯片版本一个datablock的大小，单位为byte。开发者根据datablock的大小来计算API指令中待传入的repeatTime 、DataBlock Stride、Repeat Stride等参数值。

#### 函数原型

| 1   | __aicore__inlineconstexprint16_tGetDataBlockSizeInBytes() |
| --- | --------------------------------------------------------- |

#### 参数说明

无

#### 返回值说明

当前芯片版本一个datablock的大小，单位为byte。

#### 约束说明

无

#### 调用示例

如下样例通过GetDataBlockSizeInBytes获取的datablock值，来计算repeatTime的值：

| 12345678 | int16_tdataBlockSize=AscendC:GetDataBlockSizeInBytes();// 每个repeat有8个datablock,可计算8 * dataBlockSize / sizeof(half)个数，mask配置为迭代内所有元素均参与计算uint64_tmask=8*dataBlockSize/sizeof(half);// 共计算512个数，除以每个repeat参与计算的元素个数，得到repeatTimeuint8_trepeatTime=512/mask;// dstBlkStride, src0BlkStride, src1BlkStride = 1, 单次迭代内数据连续读取和写入// dstRepStride, src0RepStride, src1RepStride = 8, 相邻迭代间数据连续读取和写入AscendC:Add(dstLocal,src0Local,src1Local,mask,repeatTime,{1,1,1,8,8,8}); |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
