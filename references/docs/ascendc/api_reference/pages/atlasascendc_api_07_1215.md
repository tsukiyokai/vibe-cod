# MetricsProfStop-性能统计-调试接口-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1215
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1215.html
---

# MetricsProfStop

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

设置性能数据采集信号停止，和MetricsProfStart配合使用。使用msProf工具进行算子上板调优时，可在kernel侧代码段前后分别调用MetricsProfStart和MetricsProfStop来指定需要调优的代码段范围。

#### 函数原型

| 1   | __aicore__inlinevoidMetricsProfStop() |
| --- | ------------------------------------- |

#### 参数说明

无

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 1   | MetricsProfStop(); |
| --- | ------------------ |
