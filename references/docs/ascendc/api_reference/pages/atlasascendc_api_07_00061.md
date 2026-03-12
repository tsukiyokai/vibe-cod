# ReserveLocalMemory-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00061
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00061.html
---

# ReserveLocalMemory

#### 功能说明

该函数用于在Unified Buffer中预留指定大小的内存空间。调用该接口后，使用GetCoreMemSize可以获取实际可用的剩余Unified Buffer空间大小。

#### 函数原型

| 1   | voidReserveLocalMemory(ReservedSizesize) |
| --- | ---------------------------------------- |

#### 参数说明

| 参数名       | 输入/输出                                                                                                                                   | 说明                                                                                                                                                                 |       |                                                                                                                                             |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| ReservedSize | 输入                                                                                                                                        | 需要预留的空间大小。12345enumclassReservedSize{RESERVED_SIZE_8K,// 预留8 * 1024B空间RESERVED_SIZE_16K,// 预留16 * 1024B空间RESERVED_SIZE_32K,// 预留32 * 1024B空间}; | 12345 | enumclassReservedSize{RESERVED_SIZE_8K,// 预留8 * 1024B空间RESERVED_SIZE_16K,// 预留16 * 1024B空间RESERVED_SIZE_32K,// 预留32 * 1024B空间}; |
| 12345        | enumclassReservedSize{RESERVED_SIZE_8K,// 预留8 * 1024B空间RESERVED_SIZE_16K,// 预留16 * 1024B空间RESERVED_SIZE_32K,// 预留32 * 1024B空间}; |                                                                                                                                                                      |       |                                                                                                                                             |

#### 返回值说明

无

#### 约束说明

多次调用该函数时，仅保留最后一次调用的结果。

#### 调用示例

| 1234567891011 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint64_tub_size,l1_size;// 预留8KB的Unified Buffer内存空间ascendcPlatform.ReserveLocalMemory(platform_ascendc:ReservedSize:RESERVED_SIZE_8K);// 获取Unified Buffer和L1的实际可用内存大小ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:UB,ub_size);ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:L1,l1_size);// ...returnret;} |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

完整样例可参考与数学库高阶API配合使用的样例。
