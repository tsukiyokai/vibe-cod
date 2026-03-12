# ToFloat-标量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0022
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0022.html
---

# ToFloat

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

将输入数据转换为float类型。

#### 函数原型

- bfloat16_t类型转换为float类型1__aicore__inlinefloatToFloat(constbfloat16_t&bVal)

#### 参数说明

| 参数名称 | 输入/输出 | 含义               |
| -------- | --------- | ------------------ |
| bVal     | 输入      | 待转换的标量数据。 |

#### 返回值说明

转换后的float类型标量数据。

#### 约束说明

无

#### 调用示例

| 1234567891011121314151617181920 | voidCalcFunc(bfloat16_tn){intdataLen=32;AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueSrcVecIn;AscendC:TQue<AscendC:TPosition:VECOUT,1>inQueueDstVecIn;pipe.InitBuffer(inQueueDstVecIn,1,dataLen*sizeof(bfloat16_t));pipe.InitBuffer(inQueueSrcVecIn,1,dataLen*sizeof(float));AscendC:LocalTensor<bfloat16_t>dstLocal=inQueueDstVecIn.AllocTensor<bfloat16_t>();AscendC:LocalTensor<float>srcLocal=inQueueSrcVecIn.AllocTensor<float>();floatt=AscendC:ToFloat(n);// 对标量进行加法，不支持bfloat16_t，需要先转换成floatPipeBarrier<PIPE_ALL>();AscendC:Duplicate(srcLocal,float(4.0f),dataLen);PipeBarrier<PIPE_ALL>();Adds(srcLocal,srcLocal,t,dataLen);PipeBarrier<PIPE_ALL>();// 做加法运算后，输出bfloat16_t类型tensorCast(dstLocal,srcLocal,AscendC:RoundMode:CAST_ROUND,dataLen);// ……} |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
