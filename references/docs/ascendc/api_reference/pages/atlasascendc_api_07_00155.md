# aclrtcCreateProg-RTC-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00155
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00155.html
---

# aclrtcCreateProg

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

通过给定的参数，创建编译程序的实例。

#### 函数原型

| 1   | aclErroraclrtcCreateProg(aclrtcProg*prog,constchar*src,constchar*name,intnumHeaders,constchar**headers,constchar**includeNames) |
| --- | ------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名       | 输入/输出 | 描述                                                                                                                                         |
| ------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| prog         | 输出      | 运行时编译程序的句柄。                                                                                                                       |
| src          | 输入      | 以字符串形式提供的Ascend C Device侧源代码内容。                                                                                              |
| name         | 输入      | 用户自定义的程序名称，用于标识和区分不同的编译程序，默认值为"default_program"。                                                              |
| numHeaders   | 输入      | 指定要包含的头文件数量，必须为非负整数。无需包含头文件或者Ascend C Device侧源代码中已包含所需头文件时，此参数需设置为0。                     |
| headers      | 输入      | 一个指向数组的指针，数组中的每个元素都是以'\0'结尾的字符串，表示头文件的源代码内容。当numHeaders为0时，此参数可以设置为nullptr。             |
| includeNames | 输入      | 一个指向数组的指针，数组中的每个元素都是以'\0'结尾的字符串，表示头文件的名称。这些名称必须与源代码中#include指令中包含的头文件名称完全一致。 |

#### 返回值说明

aclError为int类型变量，详细说明请参考RTC错误码。

#### 约束说明

无

#### 调用示例

| 12345678910111213141516171819202122232425262728293031 | aclrtcProgprog;constchar*src=R""""(#include "kernel_operator.h"#include "my_const_a.h"#include "my_const_b.h"extern "C" __global__ __aicore__ void hello_world(GM_ADDR x){KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_AIC_ONLY);*x = *x + MY_CONST_A + MY_CONST_B;})"""";constchar*headerSrcA=R"(#ifndef CONST_A_H#define CONST_A_Hconst int MY_CONST_A = 100;#endif // CONST_A_H)";constchar*includeNameA="my_const_a.h";constchar*headerSrcB=R"(#ifndef CONST_B_H#define CONST_B_Hconst int MY_CONST_B = 50;#endif // CONST_B_H)";constchar*includeNameB="my_const_b.h";constchar*headersArray[]={headerSrcA,headerSrcB};constchar*includeNameArray[]={includeNameA,includeNameB};aclErrorresult=aclrtcCreateProg(&prog,src,"hello_world",2,headersArray,includeNameArray); |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
