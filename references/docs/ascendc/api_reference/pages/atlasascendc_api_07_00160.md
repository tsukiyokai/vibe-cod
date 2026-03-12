# aclrtcGetCompileLog-RTC-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00160
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00160.html
---

# aclrtcGetCompileLog

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

获取编译日志的内容，以字符串形式保存。

#### 函数原型

| 1   | aclErroraclrtcGetCompileLog(aclrtcProgprog,char*log) |
| --- | ---------------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                               |
| ------ | --------- | ---------------------------------- |
| prog   | 输入      | 运行时编译程序的句柄。             |
| log    | 输出      | 编译日志的内容，以字符串形式保存。 |

#### 返回值说明

aclError为int类型变量，详细说明请参考RTC错误码。

#### 约束说明

无

#### 调用示例

| 12345 | aclrtcProgprog;size_tlogSize;aclrtcGetCompileLogSize(prog,&logSize);char*log=(char*)malloc(logSize);aclErrorresult=aclrtcGetCompileLog(prog,log); |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
