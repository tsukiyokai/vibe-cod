# GetIBShareNormConfig-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0624
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0624.html
---

# GetIBShareNormConfig

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

用于配置IBShare模板的参数，获取自定义IBShare模板。IBShare模板的介绍请参考表 模板特性。

#### 函数原型

| 1   | __aicore__constexprMatmulConfigGetIBShareNormConfig(constboolintrinsicsLimit=false,constboolbatchLoop=false,constboolisVecND2NZ=false,constBatchModebmmMode=BatchMode:BATCH_LESS_THAN_L1,constboolisDoubleCache=false,constboolenUnitFlag=true) |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

本接口的所有参数用于设置MatmulConfig结构体中的参数，其中互相对应的参数的功能作用相同。

| 参数名          | 输入/输出 | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| intrinsicsLimit | 输入      | 用于设置参数intrinsicsCheck。当左矩阵或右矩阵在单核上内轴（即尾轴）大于等于65535（元素个数）时，是否使能循环执行数据从Global Memory到L1 Buffer的搬入。例如，左矩阵A[M, K]，单核上的内轴数据singleCoreK大于65535，配置该参数为true后，API内部通过循环执行数据的搬入。参数取值如下：false：当左矩阵或右矩阵在单核上内轴大于等于65535时，不使能循环执行数据的搬入（默认值）。true：当左矩阵或右矩阵在单核上内轴大于等于65535时，使能循环执行数据的搬入。 |
| batchLoop       | 输入      | 用于设置参数isNBatch。是否多Batch输入多Batch输出。仅对BatchMatmul有效，使能该参数后，仅支持Norm模板，且需调用IterateNBatch实现多Batch输入多Batch输出。参数取值如下：false：不使能多Batch（默认值）。true：使能多Batch。                                                                                                                                                                                                                               |
| isVecND2NZ      | 输入      | 用于设置参数enVecND2NZ。使能通过vector指令进行ND2NZ。使能时需要设置SetLocalWorkspace。参数取值如下：false：不使能通过vector指令进行ND2NZ（默认值）。true：使能通过vector指令进行ND2NZ。针对Atlas推理系列产品AI Core，在Unified Buffer空间足够的条件下（Unified Buffer空间大于2倍TCubeTiling的transLength参数），建议优先使能该参数，搬运性能更好。                                                                                                    |
| bmmMode         | 输入      | 用于设置参数batchMode。该参数用于BatchMatmul场景，关于BatchMatmul的介绍请参考Batch Matmul基础功能。BatchMatmul场景中Layout类型为NORMAL时，设置BatchMatmul输入A/B矩阵的多batch数据总和与L1 Buffer的大小关系。参数取值如下：BatchMode:BATCH_LESS_THAN_L1：多batch数据总和<L1 BufferSize；BatchMode:BATCH_LARGE_THAN_L1：多batch数据总和>L1 BufferSize；BatchMode:SINGLE_LARGE_THAN_L1：单batch数据总和>L1 BufferSize。                                  |
| isDoubleCache   | 输入      | 用于设置参数enableDoubleCache。开启IBShare模板后，在L1 Buffer上是否同时缓存两块数据。参数取值如下：false：L1 Buffer上同时缓存一块数据（默认值）。true：使能L1 Buffer上同时缓存两块数据。注意：该参数取值为true时，需要控制基本块大小，防止两块数据的缓存超过L1 Buffer大小限制。                                                                                                                                                                       |
| enUnitFlag      | 输入      | 用于设置参数enUnitFlag。使能UnitFlag功能，使计算与搬运流水并行，提高性能。Norm, IBShare下默认使能，MDL下默认不使能。参数取值如下：false：不使能UnitFlag功能。true：使能UnitFlag功能。                                                                                                                                                                                                                                                                 |

#### 返回值说明

MatmulConfig结构体。

#### 约束说明

IBShare模板当前仅适用于MIX场景，不支持纯CUBE场景。

#### 调用示例

| 1234567891011 | constexprMatmulConfigMM_CFG=GetIBShareNormConfig();typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half>aType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half,true,LayoutMode:NONE,true>bType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>cType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>biasType;AscendC:Matmul<A_TYPE,B_TYPE,C_TYPE,BIAS_TYPE,MM_CFG>mm;REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);mm.SetTensorA(gm_a);mm.SetTensorB(gm_b);mm.SetBias(gm_bias);mm.IterateAll(gm_c); |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
