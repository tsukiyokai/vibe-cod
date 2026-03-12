# GetBlockIdx-系统变量访问-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0185
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0185.html
---

# GetBlockIdx

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

获取当前核的index，用于代码内部的多核逻辑控制及多核偏移量计算等。

#### 函数原型

| 1   | __aicore__inlineint64_tGetBlockIdx() |
| --- | ------------------------------------ |

#### 参数说明

无

#### 返回值说明

当前核的index。

index的范围为[0, 用户配置的BlockDim数量 - 1]。

#### 约束说明

GetBlockIdx为一个系统内置函数，返回当前核的index。

#### 调用示例

| 123456789101112131415161718 | #include"kernel_operator.h"constexprint32_tSINGLE_CORE_OFFSET=256;classKernelGetBlockIdx{public:__aicore__inlineKernelGetBlockIdx(){}__aicore__inlinevoidInit(__gm__uint8_t*src0Gm,__gm__uint8_t*src1Gm,__gm__uint8_t*dstGm){// 根据index对每个核进行地址偏移src0Global.SetGlobalBuffer((__gm__float*)src0Gm+AscendC:GetBlockIdx()*SINGLE_CORE_OFFSET);src1Global.SetGlobalBuffer((__gm__float*)src1Gm+AscendC:GetBlockIdx()*SINGLE_CORE_OFFSET);dstGlobal.SetGlobalBuffer((__gm__float*)dstGm+AscendC:GetBlockIdx()*SINGLE_CORE_OFFSET);pipe.InitBuffer(inQueueSrc0,1,256*sizeof(float));pipe.InitBuffer(inQueueSrc1,1,256*sizeof(float));pipe.InitBuffer(selMask,1,256);pipe.InitBuffer(outQueueDst,1,256*sizeof(float));}......}; |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
