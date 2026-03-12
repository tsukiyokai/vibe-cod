# GetTensorC-Conv3DBackpropInput Kernel侧接口-Conv3DBackpropInput-卷积计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0927
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0927.html
---

# GetTensorC

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

在完成Iterate操作后调用本接口，获取结果矩阵块，完成数据从L0C到GM的搬运。此接口与Iterate接口配合使用，用于在Iterate执行迭代计算后，获取结果矩阵。

#### 函数原型

| 12  | template<boolsync=true>__aicore__inlinevoidGetTensorC(constAscendC:GlobalTensor<DstT>&output,uint8_tenAtomic=0,boolenSequentialWrite=false) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                     |
| ------ | ------------------------ |
| sync   | 预留参数，用户无需感知。 |

| 参数名            | 输入/输出 | 描述                                  |
| ----------------- | --------- | ------------------------------------- |
| output            | 输入      | 将计算结果搬至Global Memory的GM地址。 |
| enAtomic          | 输入      | 预留参数，用户无需感知。              |
| enSequentialWrite | 输入      | 预留参数，用户无需感知。              |

#### 返回值说明

无

#### 约束说明

| 123 | while(Iterate()){GetTensorC();} |
| --- | ------------------------------- |

#### 调用示例

| 123 | while(gradInput_.Iterate()){gradInput_.GetTensorC(gradInputGm_[offsetC_]);} |
| --- | --------------------------------------------------------------------------- |
