# GetMatmulApiTiling-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0666
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0666.html
---

# GetMatmulApiTiling

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

本接口用于在编译期间获取常量化的Matmul Tiling参数。

Matmul Tiling常量化功能为在编译期期间获取常量化的Matmul Tiling参数并进行算子编译，从而减少Scalar计算开销，提升算子整体性能。具体为，在获取Matmul模板时，可以确定MatmulConfig的singleCore Shape（MatmulConfig中的singleCoreM/singleCoreN/singleCoreK）和Base Shape（MatmulConfig中的basicM/basicN/basicK）参数，或者只确定Base Shape参数；通过指定获取模板的接口中的singleCore Shape和Base Shape参数，或者只指定Base Shape参数，获取自定义模板；然后调用本接口，得到常量化的Matmul Tiling参数。

当在调用获取MatmulConfig模板的接口时，只将(baseM, baseN, baseK)设置为常数值时，称为部分常量化，此时(singleCoreM, singleCoreN, singleCoreK)都保持默认值0，部分常量化场景在Kernel侧使用REGIST_MATMUL_OBJ初始化Matmul对象时，仍需要使用Tiling；将(baseM, baseN, baseK, singleCoreM, singleCoreN, singleCoreK)都设置为常数值时，称为全量常量化，这时可以在REGIST_MATMUL_OBJ的入参传递Tiling参数的位置，使用空指针替代。

经过上述部分常量化或全部常量化后，将得到带有常量化参数的MatmulConfig模板，然后使用本接口将Tiling参数常量化。本接口的返回值包含常量化的Matmul Tiling参数和MatmulConfig模板。

#### 函数原型

| 12  | template<classA_TYPE,classB_TYPE,classC_TYPE,classBIAS_TYPE>__aicore__constexprMatmulApiStaticTilingGetMatmulApiTiling(constMatmulConfig&mmCFG,int32_tl1Size=Impl:L1_SIZE) |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名    | 描述                                     |
| --------- | ---------------------------------------- |
| A_TYPE    | A矩阵类型信息，通过MatmulType来定义。    |
| B_TYPE    | B矩阵类型信息，通过MatmulType来定义。    |
| C_TYPE    | C矩阵类型信息，通过MatmulType来定义。    |
| BIAS_TYPE | BIAS矩阵类型信息，通过MatmulType来定义。 |

| 参数名 | 输入/输出 | 描述                                                                                                                                                                                             |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| mmCFG  | 输入      | 获取的MatmulConfig模板。对于Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持常量化的模板有：Norm, MDL模板。对于Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持常量化的模板有：Norm, MDL模板。 |
| l1Size | 输入      | 可用的L1大小，默认值L1_SIZE。                                                                                                                                                                    |

#### 返回值说明

MatmulApiStaticTiling

#### 约束说明

- 入参mmCFG，在调用获取MatmulConfig模板的接口获取时，需要使用常数值指定(baseM, baseN, baseK)或者指定(baseM, baseN, baseK, singleCoreM, singleCoreN, singleCoreK)，并且指定的参数值需要和Tiling计算的值保持一致。
- Batch Matmul场景支持全量常量化，但不支持使用空指针替代REGIST_MATMUL_OBJ的入参Tiling。

#### 调用示例

完整算子样例请参考Matmul Tiling常量化的算子样例。

| 12345678 | //定义Matmul对象typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half>aType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half>bType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>cType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>biasType;// 这里CFG使用GetNormalConfig接口获取，并指定已知的singleshape信息和baseM,baseN,baseK，指定的数值跟运行时tiling保持一致constexprautostaticTiling=GetMatmulApiTiling<aType,bType,cType,biasType>(CFG,524288);// 该示例L1 Buffer可用大小为512KBAscendC:Matmul<aType,bType,cType,biasType,staticTiling>mm; |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
