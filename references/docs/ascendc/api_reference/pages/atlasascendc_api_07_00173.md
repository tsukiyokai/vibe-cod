# TPipe简介-TPipe-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00173
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00173.html
---

# TPipe简介

TPipe用于统一管理Device端内存等资源，一个Kernel函数必须且只能初始化一个TPipe对象。其主要功能包括：

- 内存资源管理：通过TPipe的InitBuffer接口，可以为TQue和TBuf分配内存，分别用于队列的内存初始化和临时变量内存的初始化。
- 同步事件管理：通过TPipe的AllocEventID、ReleaseEventID等接口，可以申请和释放事件ID，用于同步控制。
