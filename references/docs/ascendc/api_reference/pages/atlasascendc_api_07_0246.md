# SetLoadDataBoundary-数据搬运-矩阵计算(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0246
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0246.html
---

# SetLoadDataBoundary

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

设置Load3D时A1/B1边界值。

如果Load3D指令在处理源操作数时，源操作数在A1/B1上的地址超出设置的边界，则会从A1/B1起始地址开始读取数据。

#### 函数原型

| 1   | __aicore__inlinevoidSetLoadDataBoundary(uint32_tboundaryValue) |
| --- | -------------------------------------------------------------- |

#### 参数说明

| 参数名称      | 输入/输出 | 含义                                                           |
| ------------- | --------- | -------------------------------------------------------------- |
| boundaryValue | 输入      | 边界值。Load3Dv1指令：单位是32字节。Load3Dv2指令：单位是字节。 |

#### 约束说明

- 用于Load3Dv1时，boundaryValue的最小值是16（单位：32字节）；用于Load3Dv2时，boundaryValue的最小值是1024（单位：字节）。
- 如果使用SetLoadDataBoundary接口设置了边界值，配合Load3D指令使用时，Load3D指令的A1/B1初始地址要在设置的边界内。
- 如果boundaryValue设置为0，则表示无边界，可使用整个A1/B1。
- 操作数地址对齐要求请参见通用地址对齐约束。

#### 调用示例

参考调用示例。
