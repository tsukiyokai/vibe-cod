# GmFree-CPU孪生调试-调试接口-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1210
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1210.html
---

# GmFree

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

进行核函数的CPU侧运行验证时，用于释放通过GmAlloc申请的共享内存。

#### 函数原型

| 1   | voidGmFree(void*ptr) |
| --- | -------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                       |
| ------ | --------- | -------------------------- |
| ptr    | 输入      | 需要释放的共享内存的指针。 |

#### 返回值说明

无

#### 约束说明

传入的指针必须是之前通过GmAlloc申请过的共享内存的指针。

#### 调用示例

| 1   | AscendC:GmFree((void*)x); |
| --- | ------------------------- |
