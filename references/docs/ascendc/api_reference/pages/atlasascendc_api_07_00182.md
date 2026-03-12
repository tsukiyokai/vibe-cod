# ShapeInfo-其他数据类型-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00182
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00182.html
---

# ShapeInfo

#### 功能说明

ShapeInfo用来存放LocalTensor或GlobalTensor的shape信息。

#### 函数原型

- ShapeInfo结构定义12345678910111213structShapeInfo{public:__aicore__inlineShapeInfo();__aicore__inlineShapeInfo(constuint8_tinputShapeDim,constuint32_tinputShape[],constuint8_tinputOriginalShapeDim,constuint32_tinputOriginalShape[],constDataFormatinputFormat);__aicore__inlineShapeInfo(constuint8_tinputShapeDim,constuint32_tinputShape[],constDataFormatinputFormat);__aicore__inlineShapeInfo(constuint8_tinputShapeDim,constuint32_tinputShape[]);uint8_tshapeDim;uint8_toriginalShapeDim;uint32_tshape[K_MAX_DIM];uint32_toriginalShape[K_MAX_DIM];DataFormatdataFormat;};

- 获取Shape中所有dim的累乘结果1__aicore__inlineintGetShapeSize(constShapeInfo&shapeInfo)

#### 函数说明

| 参数名称         | 描述                                                                                                    |         |                                                          |
| ---------------- | ------------------------------------------------------------------------------------------------------- | ------- | -------------------------------------------------------- |
| shapeDim         | 现有的shape维度。                                                                                       |         |                                                          |
| shape            | 现有的shape。                                                                                           |         |                                                          |
| originalShapeDim | 原始的shape维度。                                                                                       |         |                                                          |
| originalShape    | 原始的shape。                                                                                           |         |                                                          |
| dataFormat       | 数据排布格式，DataFormat类型，定义如下：1234567enumclassDataFormat:uint8_t{ND=0,NZ,NCHW,NC1HWC0,NHWC,}; | 1234567 | enumclassDataFormat:uint8_t{ND=0,NZ,NCHW,NC1HWC0,NHWC,}; |
| 1234567          | enumclassDataFormat:uint8_t{ND=0,NZ,NCHW,NC1HWC0,NHWC,};                                                |         |                                                          |

| 参数名    | 输入/输出 | 描述                                                  |
| --------- | --------- | ----------------------------------------------------- |
| shapeInfo | 输入      | ShapeInfo类型，LocalTensor或GlobalTensor的shape信息。 |
