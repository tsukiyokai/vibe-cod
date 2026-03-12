# aclrtcGetBinDataSize-RTC-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00158
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00158.html
---

# aclrtcGetBinDataSize

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

获取编译的二进制数据大小。用于在aclrtcGetBinData获取二进制数据时分配对应大小的内存空间。

#### 函数原型

| 1   | aclErroraclrtcGetBinDataSize(aclrtcProgprog,size_t*binDataSizeRet) |
| --- | ------------------------------------------------------------------ |

#### 参数说明

| 参数名         | 输入/输出 | 描述                               |
| -------------- | --------- | ---------------------------------- |
| prog           | 输入      | 运行时编译程序的句柄。             |
| binDataSizeRet | 输出      | 编译得到的Device侧二进制数据大小。 |

#### 返回值说明

aclError为int类型变量，详细说明请参考RTC错误码。

#### 约束说明

无

#### 调用示例

| 123 | aclrtcProgprog;size_tbinDataSizeRet=0;aclErrorresult=aclrtcGetBinDataSize(prog,&binDataSizeRet); |
| --- | ------------------------------------------------------------------------------------------------ |
