# GmAlloc-CPU孪生调试-调试接口-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00179
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00179.html
---

# GmAlloc

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

进行核函数的CPU侧运行验证时，用于创建共享内存：在/tmp目录下创建一个共享文件，并返回该文件的映射指针。

#### 函数原型

| 1   | void*GmAlloc(size_tsize) |
| --- | ------------------------ |

#### 参数说明

| 参数名 | 输入/输出 | 描述                       |
| ------ | --------- | -------------------------- |
| size   | 输入      | 用户想要申请的共享内存大小 |

#### 返回值说明

返回该共享内存空间的首地址。

#### 约束说明

该接口在系统的/tmp目录下生成临时文件，故需要磁盘空间足够才可以正常生成共享内存。

#### 调用示例

| 12  | constexprsize_tlen=8*32*1024*8;half*x=(half*)GmAlloc(len*sizeof(half)); |
| --- | ----------------------------------------------------------------------- |
