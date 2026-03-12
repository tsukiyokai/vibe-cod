# SetTensorA-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0631
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0631.html
---

# SetTensorA

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

设置矩阵乘的左矩阵A。

#### 函数原型

| 1   | __aicore__inlinevoidSetTensorA(constGlobalTensor<SrcAT>&gm,boolisTransposeA=false) |
| --- | ---------------------------------------------------------------------------------- |

| 1   | __aicore__inlinevoidSetTensorA(constLocalTensor<SrcAT>&leftMatrix,boolisTransposeA=false) |
| --- | ----------------------------------------------------------------------------------------- |

| 1   | __aicore__inlinevoidSetTensorA(SrcATaScalar) |
| --- | -------------------------------------------- |

Atlas推理系列产品AI Core不支持SetTensorA(SrcAT aScalar)接口原型。

Atlas 200I/500 A2 推理产品，不支持SetTensorA(SrcAT aScalar)接口原型。

#### 参数说明

| 参数名       | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| gm           | 输入      | A矩阵。类型为GlobalTensor。SrcAT参数表示A矩阵的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int8_t/int4b_tAtlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int8_t/int4b_tAtlas推理系列产品AI Core，支持的数据类型为：half/float/int8_tAtlas 200I/500 A2 推理产品，支持的数据类型为：half/float/bfloat16_t/int8_t                                                                                                                                                                                                                                                                                                                                                                                                                         |
| leftMatrix   | 输入      | A矩阵。类型为LocalTensor，支持的TPosition为TSCM/VECOUT。SrcAT参数表示A矩阵的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int8_t/int4b_tAtlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half/float/bfloat16_t/int8_t/int4b_tAtlas推理系列产品AI Core，支持的数据类型为：half/float/int8_tAtlas 200I/500 A2 推理产品，支持的数据类型为：half/float/bfloat16_t/int8_t若设置TSCM首地址，默认矩阵可全载，已经位于TSCM，Iterate接口无需再进行GM->A1/B1搬运。                                                                                                                                                                                                                                                                                                        |
| aScalar      | 输入      | A矩阵中设置的值。支持传入标量数据，标量数据会被扩展为一个形状为[1, K]的tensor参与矩阵乘计算，tensor的数值均为该标量值。例如，开发者可以通过将aScalar设置为1来实现矩阵B在K方向的reduce sum操作。SrcAT参数表示A矩阵的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：half/floatAtlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：half/floatAtlas推理系列产品AI Core不支持该参数。Atlas 200I/500 A2 推理产品不支持该参数。                                                                                                                                                                                                                                                                                                                                                                  |
| isTransposeA | 输入      | A矩阵是否需要转置。注意：若A矩阵MatmulType的ISTRANS参数设置为true，该参数可以为true也可以为false，即运行时可以转置和非转置交替使用；若A矩阵MatmulType的ISTRANS参数设置为false，该参数只能设置为false，若强行设置为true，精度会有异常；对于非half、非bfloat16_t输入类型的场景，为了确保Tiling侧与Kernel侧L1 Buffer空间计算大小保持一致及结果精度正确，该参数取值必须与Kernel侧定义A矩阵MatmulType的ISTRANS参数以及Tiling侧SetAType()接口的isTrans参数保持一致，即上述三个参数必须同时设置为true或同时设置为false。Atlas推理系列产品AI Core，A矩阵为int8_t数据类型时不支持转置，即不支持该参数设置为true。Atlas A2 训练系列产品/Atlas A2 推理系列产品，A矩阵为int4b_t数据类型时不支持转置，即不支持该参数设置为true。Atlas A3 训练系列产品/Atlas A3 推理系列产品，A矩阵为int4b_t数据类型时不支持转置，即不支持该参数设置为true。 |

#### 返回值说明

无

#### 约束说明

传入的TensorA地址空间大小需要保证不小于singleM * singleK。

#### 调用示例

| 12345678910 | REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);// 示例一：左矩阵在Global Memorymm.SetTensorA(gm_a);mm.SetTensorB(gm_b);mm.SetBias(gm_bias);mm.IterateAll(gm_c);// 示例二：左矩阵在Local Memorymm.SetTensorA(local_a);// 示例三：设置标量数据mm.SetTensorA(scalar_a); |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
