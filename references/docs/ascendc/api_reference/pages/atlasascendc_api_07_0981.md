# SetCheckSupport-OpAICoreDef-原型注册与管理-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0981
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0981.html
---

# SetCheckSupport

#### 功能说明

如果您需要在算子融合阶段进行算子参数校验，则可实现算子参数校验回调函数，并通过该接口进行注册。同时，需要将NeedCheckSupportFlag参数配置为true，则算子编译和融合阶段会调用注册的算子参数校验函数进行相关信息的校验。

若算子参数校验函数校验通过，则代表AI Core支持此算子参数，会选择AI Core上相应的算子执行；否则，会继续查询AI CPU算子库然后执行。

#### 函数原型

| 1   | OpAICoreDef&SetCheckSupport(optiling:OP_CHECK_FUNCfunc) |
| --- | ------------------------------------------------------- |

#### 参数说明

| 参数 | 输入/输出                                                                         | 说明                                                                                                                                                                                                                                                                                                                                                                |     |                                                                                   |
| ---- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- | --------------------------------------------------------------------------------- |
| func | 输入                                                                              | 参数校验函数。OP_CHECK_FUNC类型定义如下：1usingOP_CHECK_FUNC=ge:graphStatus(*)(constge:Operator&op,ge:AscendString&result);该函数的入参是算子的描述，包括算子的输入、输出、属性等信息，出参为包含了校验返回码和原因的字符串，字符串的格式如下：{"ret_code": "1","reason": "your reason"}若校验成功，则函数返回ge:GRAPH_SUCCESS；若校验失败，则返回ge:GRAPH_FAILED。 | 1   | usingOP_CHECK_FUNC=ge:graphStatus(*)(constge:Operator&op,ge:AscendString&result); |
| 1    | usingOP_CHECK_FUNC=ge:graphStatus(*)(constge:Operator&op,ge:AscendString&result); |                                                                                                                                                                                                                                                                                                                                                                     |     |                                                                                   |

#### 返回值说明

OpAICoreDef请参考OpAICoreDef。

#### 约束说明

无

#### 调用示例

下文展示了自定义Add算子参数校验函数实现和注册的样例。

- 参数校验函数实现如下：对第一个输入参数的shape进行校验，仅支持输入x shape的第一个维度为8，否则不支持。12345678910111213staticge:graphStatusCheckSupported(constge:Operator&op,ge:AscendString&result){std:stringresultJsonStr;// 仅支持第一个输入参数shape的第一个维度为8，其他shape不支持if(op.GetInputDesc(0).GetShape().GetDim(0)==8){resultJsonStr=R"({"ret_code": "1","reason": "x.dim[0] is 8"})";result=ge:AscendString(resultJsonStr.c_str());returnge:GRAPH_SUCCESS;}resultJsonStr=R"({"ret_code": "0","reason": "xxx"})";result=ge:AscendString(resultJsonStr.c_str());returnge:GRAPH_FAILED;}
- 参数校验函数注册的样例如下：1234567891011121314151617181920212223242526272829classAddCustom:publicOpDef{public:AddCustom(constchar*name):OpDef(name){this->Input("x").ParamType(REQUIRED);this->Input("y").ParamType(REQUIRED);this->Output("z").ParamType(REQUIRED);this->SetInferShape(ge:InferShape);this->AICore().SetTiling(optiling:TilingFunc).SetTilingParse(optiling:TilingPrepare).SetOpSelectFormat(optiling:OpSelectFormat).SetCheckSupport(optiling:CheckSupported);OpAICoreConfigaicConfig;aicConfig.DynamicCompileStaticFlag(true).DynamicFormatFlag(true).DynamicRankSupportFlag(true).DynamicShapeSupportFlag(true).NeedCheckSupportFlag(true).PrecisionReduceFlag(true);// 注意：soc_version请替换成实际的AI处理器型号this->AICore().AddConfig("soc_version",aicConfig);}};
