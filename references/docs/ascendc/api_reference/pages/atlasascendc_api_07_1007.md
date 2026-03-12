# 简介-ContextBuilder-Tiling调测-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1007
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1007.html
---

# 简介

ContextBuilder类提供一系列的API接口，支持手动构造TilingContext类来验证Tiling函数以及KernelContext类用于TilingParse函数的验证。

#### 调用示例

| 1234567891011121314151617 | // 构造KernelContextautokernelContextHolder=context_ascendc:ContextBuilder().Inputs(...).Outputs(...).BuildKernelRunContext();gert:KernelContext*tilingParseContext=kernelContextHolder->GetContext<gert:KernelContext>();// 构造TilingContextautotilingContextHolder=context_ascendc:ContextBuilder().SetOpNameType(...,...).NodeIoNum(...).IrInstanceNum(...).AddInputTd(...).AddOutputTd(...).AddAttr(...).BuildTilingContext(...);gert:TilingContext*tilingContext=tilingContextHolder->GetContext<gert:TilingContext>(); |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
