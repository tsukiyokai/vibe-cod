# 构造函数与析构函数-KfcWorkspace-Cube分组管理(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0307
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0307.html
---

# 构造函数与析构函数

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

用于创建KfcWorkspace对象，Kfc全称为kernel function call，表示核间通信调用。

#### 函数原型

| 123 | classKfcWorkspace;__aicore__inlineKfcWorkspace(GM_ADDRworkspace)__aicore__inline~KfcWorkspace() |
| --- | ----------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数      | 输入/输出 | 说明                                                      |
| --------- | --------- | --------------------------------------------------------- |
| workspace | 输入      | Global Memory上的消息空间地址，用户需保证地址对齐和清零。 |

#### 返回值说明

KfcWorkspace对象实例。

#### 约束说明

不能和REGIST_MATMUL_OBJ接口同时使用。使用资源管理API时，用户自主管理AIC和AIV的核间通信，REGIST_MATMUL_OBJ内部是由框架管理AIC和AIV的核间通信，同时使用可能会导致通信消息错误等异常。

#### 调用示例

| 1   | AscendC:KfcWorkspacedesc(workspaceGM); |
| --- | -------------------------------------- |
