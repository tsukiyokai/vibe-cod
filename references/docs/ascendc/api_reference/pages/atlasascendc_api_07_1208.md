# ICPU_RUN_KF-CPU孪生调试-调试接口-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_1208
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_1208.html
---

# ICPU_RUN_KF

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

进行核函数的CPU侧运行验证时，CPU调测总入口，完成CPU侧的算子程序调用。

#### 函数原型

| 1   | #define ICPU_RUN_KF(func, blkdim, ...) |
| --- | -------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                                                         |
| ------ | --------- | ---------------------------------------------------------------------------- |
| func   | 输入      | 算子的kernel函数指针。                                                       |
| blkdim | 输入      | 算子的核心数，corenum。                                                      |
| ...    | 输入      | 所有的入参和出参，依次填入，当前参数个数限制为32个，超出32时会出现编译错误。 |

#### 返回值说明

无

#### 约束说明

除了func、blkdim以外，其他的变量都必须是通过GmAlloc分配的共享内存的指针；传入的参数的数量和顺序都必须和kernel保持一致。

#### 调用示例

下面代码以add_custom算子为例，介绍算子核函数在CPU侧验证时，算子调用的应用程序如何编写。您在实现自己的应用程序时，需要关注由于算子核函数不同带来的修改，包括算子核函数名，入参出参的不同等，合理安排相应的内存分配、内存拷贝和文件读写等，相关API的调用方式直接复用即可。

1. 按需包含头文件，通过ASCENDC_CPU_DEBUG宏区分CPU和NPU侧需要包含的头文件。1234567#include"data_utils.h"#ifndef ASCENDC_CPU_DEBUG#include"acl/acl.h"#else#include"tikicpulib.h"extern"C"__global____aicore__voidadd_custom(GM_ADDRx,GM_ADDRy,GM_ADDRz);// 核函数声明#endif
1. CPU侧运行验证。完成算子核函数CPU侧运行验证的步骤如下：图1CPU侧运行验证步骤12345678910111213141516171819202122232425int32_tmain(int32_targc,char*argv[]){uint32_tblockDim=8;size_tinputByteSize=8*2048*sizeof(uint16_t);size_toutputByteSize=8*2048*sizeof(uint16_t);// 使用GmAlloc分配共享内存，并进行数据初始化uint8_t*x=(uint8_t*)AscendC:GmAlloc(inputByteSize);uint8_t*y=(uint8_t*)AscendC:GmAlloc(inputByteSize);uint8_t*z=(uint8_t*)AscendC:GmAlloc(outputByteSize);ReadFile("./input/input_x.bin",inputByteSize,x,inputByteSize);ReadFile("./input/input_y.bin",inputByteSize,y,inputByteSize);// 矢量算子需要设置内核模式为AIV模式AscendC:SetKernelMode(KernelMode:AIV_MODE);// 调用ICPU_RUN_KF调测宏，完成核函数CPU侧的调用ICPU_RUN_KF(add_custom,blockDim,x,y,z);// 输出数据写出WriteFile("./output/output_z.bin",z,outputByteSize);// 调用GmFree释放申请的资源AscendC:GmFree((void*)x);AscendC:GmFree((void*)y);AscendC:GmFree((void*)z);return0;}
