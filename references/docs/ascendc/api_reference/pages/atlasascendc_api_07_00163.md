# Async-工具函数-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00163
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00163.html
---

# Async

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

Async提供了一个统一的接口，用于在不同模式下（AIC或AIV）执行特定函数，从而避免代码中直接的硬件条件判断（如使用ASCEND_IS_AIV或ASCEND_IS_AIC）。

#### 函数原型

| 12  | template<EngineTypeengine,autofunPtr,class...Args>__aicore__voidAsync(Args...args) |
| --- | ---------------------------------------------------------------------------------- |

#### 参数说明

| 参数名        | 描述                                                                                            |      |                                                           |
| ------------- | ----------------------------------------------------------------------------------------------- | ---- | --------------------------------------------------------- |
| engine        | 引擎模式，参数取值分别为AIC、AIV。1234enumclassEngineType:int32_t{AIC=1,// 仅AICAIV=2// 仅AIV}; | 1234 | enumclassEngineType:int32_t{AIC=1,// 仅AICAIV=2// 仅AIV}; |
| 1234          | enumclassEngineType:int32_t{AIC=1,// 仅AICAIV=2// 仅AIV};                                       |      |                                                           |
| funPtr        | 函数指针，指定要执行的函数，函数签名和参数类型由class... Args决定。                             |      |                                                           |
| class... Args | 可变参数模板，表示函数参数的类型列表，用于传递给funPtr。                                        |      |                                                           |

| 参数名       | 输入/输出 | 描述                                                        |
| ------------ | --------- | ----------------------------------------------------------- |
| Args... args | 输入      | 与class... Args对应的参数列表，表示传递给funPtr的实际参数。 |

#### 返回值说明

无

#### 约束说明

无

#### 调用示例

| 123456789101112 | extern"C"__global____aicore__voidbaremix_custom(GM_ADDRa,GM_ADDRb,GM_ADDRbias,GM_ADDRc,GM_ADDRworkspace,GM_ADDRtilingGm){KERNEL_TASK_TYPE_DEFAULT(KERNEL_TYPE_MIX_AIC_1_2);AscendC:TPipepipe;TCubeTilingtiling;CopyTiling(&tiling,tilingGm);Async<EngineType:AIC,aicOperation>(a,b,bias,c,workspace,tiling,&pipe);Async<EngineType:AIV,aivOperation>(c,tiling,&pipe);} |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12345678910111213141516 | // 其他代码逻辑...__aicore__inlinevoidaicOperation(GM_ADDRa,GM_ADDRb,GM_ADDRbias,GM_ADDRc,GM_ADDRworkspace,constTCubeTiling&tiling,AscendC:TPipe*pipe){MatmulLeakyKernel<half,half,float,float>matmulLeakyKernel;matmulLeakyKernel.Init(a,b,bias,c,workspace,tiling,pipe);REGIST_MATMUL_OBJ(pipe,GetSysWorkSpacePtr(),matmulLeakyKernel.matmulObj,&matmulLeakyKernel.tiling);matmulLeakyKernel.Process(pipe);}__aicore__inlinevoidaivOperation(GM_ADDRc,constTCubeTiling&tiling,AscendC:TPipe*pipe){LeakyReluKernel<float>leakyReluKernel;leakyReluKernel.Init(c,tiling,pipe);leakyReluKernel.Process(pipe);}...// 其他代码逻辑 |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
