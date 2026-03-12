# Reset-TPipe-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0113
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0113.html
---

# Reset

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

完成资源的释放与eventId等变量的初始化操作，恢复到TPipe的初始化状态。

#### 函数原型

| 1   | __aicore__inlinevoidReset() |
| --- | --------------------------- |

#### 约束说明

无

#### 返回值说明

无

#### 调用示例

| 123456789 | AscendC:TPipepipe;// Pipe内存管理对象AscendC:TQue<AscendC:TPosition:VECOUT,1>que;// 输出数据Queue队列管理对象，TPosition为VECOUTuint8_tnum=1;uint32_tlen=192*1024;for(inti=0;i<2;i++){pipe.InitBuffer(que,num,len);...// processpipe.Reset();} |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
