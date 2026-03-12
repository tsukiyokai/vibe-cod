# InitBuffer-TBufPool-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0125
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0125.html
---

# InitBuffer

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

调用TBufPool:InitBuffer接口为TQue/TBuf进行内存分配。

#### 函数原型

| 12  | template<classT>__aicore__inlineboolInitBuffer(T&que,uint8_tnum,uint32_tlen)template<TPositionpos>__aicore__inlineboolInitBuffer(TBuf<pos>&buf,uint32_tlen) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 说明                                                                                               |
| ------ | -------------------------------------------------------------------------------------------------- |
| T      | que参数的类型。                                                                                    |
| pos    | Buffer逻辑位置，可以为VECIN、VECOUT、VECCALC、A1、B1、C1。关于TPosition的具体介绍请参考TPosition。 |

| 参数名称 | 输入/输出 | 含义                                                                    |
| -------- | --------- | ----------------------------------------------------------------------- |
| que      | 输入      | 需要分配内存的TQue对象                                                  |
| num      | 输入      | 分配内存块的个数                                                        |
| len      | 输入      | 每个内存块的大小，单位为Bytes，非32Bytes对齐会自动向上补齐至32Bytes对齐 |

| 参数名称 | 输入/输出 | 含义                                                                        |
| -------- | --------- | --------------------------------------------------------------------------- |
| buf      | 输入      | 需要分配内存的TBuf对象                                                      |
| len      | 输入      | 为TBuf分配的内存大小，单位为Bytes，非32Bytes对齐会自动向上补齐至32Bytes对齐 |

#### 约束说明

声明TBufPool时，可以通过bufIDSize指定可分配Buffer的最大数量，默认上限为4，最大为16。TQue或TBuf的物理内存需要和TBufPool一致。

#### 返回值说明

无

#### 调用示例

参考InitBufPool
