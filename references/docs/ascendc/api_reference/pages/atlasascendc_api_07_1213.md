# TRACE_STOP-性能统计-调试接口-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1213
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1213.html
---

# TRACE_STOP

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

通过CAModel进行算子性能仿真时，可对算子任意运行阶段打点，从而分析不同指令的流水图，以便进一步性能调优。

用于表示终止位置打点，一般与TRACE_START配套使用。

![](../images/atlasascendc_api_07_1213_img_001.png)

该功能主要用于调试和性能分析，开启后会对算子性能产生一定影响，通常在调测阶段使用，生产环境建议关闭。

默认情况下，该功能关闭，开发者可以按需通过如下方式开启打点功能。

| 1234 | // 打开算子的打点功能ascendc_compile_definitions(ascendc_kernels_${RUN_MODE}PRIVATE-DASCENDC_TRACE_ON) |
| ---- | ------------------------------------------------------------------------------------------------------ |

#### 函数原型

| 1   | #define TRACE_STOP(TraceId apid) |
| --- | -------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                    |
| ------ | --------- | ------------------------------------------------------- |
| apid   | 输入      | 取值需与TRACE_START参数取值保持一致，否则影响打点结果。 |

#### 返回值说明

无

#### 约束说明

- TRACE_START/TRACE_STOP需配套使用，若Trace图上未显示打点，则说明两者没有配对。
- 不支持跨核使用，例如TRACE_START在AI Cube打点，则TRACE_STOP打点也需要在AI Cube上，不能在AI Vector上。
- 宏支持所有的产品型号，但实际调用时需与调测工具支持的型号保持一致。
- 仅支持Kernel直调工程，不支持自定义算子工程下开启打点功能。

#### 调用示例

在Kernel代码中特定指令位置打上TRACE_START/TRACE_STOP：

| 123 | TRACE_START(0x1);DataCopy(zGm,zLocal,this->totalLength);TRACE_STOP(0x1); |
| --- | ------------------------------------------------------------------------ |
