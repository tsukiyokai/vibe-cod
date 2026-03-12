# GetUserWorkspace-workspace-临时空间管理-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0172
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0172.html
---

# GetUserWorkspace

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

获取用户使用的workspace指针。workspace的具体介绍请参考如何使用workspace。Kernel直调开发方式下，如果未开启HAVE_WORKSPACE编译选项，框架不会自动设置系统workspace。如果使用了Matmul Kernel侧接口等需要系统workspace的高阶API，kernel侧需要通过SetSysWorkSpace设置系统workspace，此时用户workspace需要通过该接口获取。

#### 函数原型

| 1   | __aicore__inlineGM_ADDRGetUserWorkspace(GM_ADDRworkspace) |
| --- | --------------------------------------------------------- |

#### 参数说明

| 参数名称  | 输入/输出 | 描述                                                          |
| --------- | --------- | ------------------------------------------------------------- |
| workspace | 输入      | 传入workspace的指针，包括系统workspace和用户使用的workspace。 |

#### 约束说明

无

#### 返回值说明

用户使用workspace指针。
