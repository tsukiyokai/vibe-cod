# GetCoreMemBw-PlatformAscendC-平台信息获取-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1035
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1035.html
---

# GetCoreMemBw

#### 功能说明

获取硬件平台存储空间的带宽大小。硬件存储空间类型定义如下：

| 12345678910 | enumclassCoreMemType{L0_A=0,// 预留参数，暂不支持L0_B=1,// 预留参数，暂不支持L0_C=2,// 预留参数，暂不支持L1=3,// 预留参数，暂不支持L2=4,UB=5,// 预留参数，暂不支持HBM=6,RESERVED}; |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 函数原型

| 1   | voidGetCoreMemBw(constCoreMemType&memType,uint64_t&bwSize)const |
| --- | --------------------------------------------------------------- |

#### 参数说明

| 参数    | 输入/输出 | 说明                                                                |
| ------- | --------- | ------------------------------------------------------------------- |
| memType | 输入      | 硬件存储空间类型。                                                  |
| bwSize  | 输出      | 对应硬件的存储空间的带宽大小。单位是Byte/cycle，cycle代表时钟周期。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 1234567 | ge:graphStatusTilingXXX(gert:TilingContext*context){autoascendcPlatform=platform_ascendc:PlatformAscendC(context->GetPlatformInfo());uint64_tl2_bw;ascendcPlatform.GetCoreMemBw(platform_ascendc:CoreMemType:L2,l2_bw);// ...returnret;} |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
