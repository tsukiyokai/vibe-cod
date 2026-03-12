# 简介-OpTilingRegistry-Tiling调测-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00071
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00071.html
---

# 简介

OpTilingRegistry类属于context_ascendc命名空间，主要用于加载Tiling实现的动态库，并获取算子的Tiling函数指针以进行调试和验证。该类通常与ContextBuilder类配合使用，后者用于构造Tiling函数所需的输入参数。使用OpTilingRegistry类需要链接libtiling_api.so，具体用法可参考如何进行Tiling调测。

#### 需要包含的头文件

| 1   | #include"tiling/context/context_builder.h" |
| --- | ------------------------------------------ |

#### Public成员函数
