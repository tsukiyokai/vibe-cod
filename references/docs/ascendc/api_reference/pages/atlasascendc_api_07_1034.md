# GetCoreMemSize-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1034
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1034.html
---

# GetCoreMemSize

#### 功能说明

获取硬件平台存储空间的内存大小，例如L1、L0_A、L0_B、L2等，支持的存储空间类型定义如下：

| 123456789101112 | enumclassCoreMemType{L0_A=0,// L0A BufferL0_B=1,// L0B BufferL0_C=2,// L0C BufferL1=3,// L1 BufferL2=4,// L2 CacheUB=5,// Unified BufferHBM=6,// GMFB=7,// Fixpipe BufferBT=8,// BiasTable BufferRESERVED}; |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 函数原型

| 1   | voidGetCoreMemSize(constCoreMemType&memType,uint64_t&size)const |
| --- | --------------------------------------------------------------- |

#### 参数说明

| 参数    | 输入/输出 | 说明                                 |
| ------- | --------- | ------------------------------------ |
| memType | 输入      | 硬件存储空间类型。                   |
| size    | 输出      | 对应类型的存储空间大小，单位：字节。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 12345678 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint64_tub_size,l1_size;ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:UB,ub_size);ascendcPlatform.GetCoreMemSize(platform_ascendc:CoreMemType:L1,l1_size);// ...returnret;} |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
