# SetUserDefInfo-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0664
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0664.html
---

# SetUserDefInfo

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

使能模板参数MatmulCallBackFunc（自定义回调函数）时，设置算子tiling地址，用于回调函数使用，该接口仅需调用一次。

#### 函数原型

| 1   | __aicore__inlinevoidSetUserDefInfo(constuint64_ttilingPtr) |
| --- | ---------------------------------------------------------- |

#### 参数说明

| 参数名    | 输入/输出 | 描述                   |
| --------- | --------- | ---------------------- |
| tilingPtr | 输入      | 设置的算子tiling地址。 |

#### 返回值说明

无

#### 约束说明

- 若回调函数中需要使用tilingPtr参数时，必须调用此接口；若回调函数不使用tilingPtr参数，无需调用此接口。
- 当使能MixDualMaster（双主模式）场景时，即模板参数enableMixDualMaster设置为true，不支持使用该接口。

#### 调用示例

| 12345678910111213141516 | //用户自定义回调函数voidDataCopyOut(const__gm__void*gm,constLocalTensor<int8_t>&co1Local,constvoid*dataCopyOutParams,constuint64_ttilingPtr,constuint64_tdataPtr);voidCopyA1(constLocalTensor<int8_t>&aMatrix,const__gm__void*gm,introw,intcol,intuseM,intuseK,constuint64_ttilingPtr,constuint64_tdataPtr);voidCopyB1(constLocalTensor<int8_t>&bMatrix,const__gm__void*gm,introw,intcol,intuseK,intuseN,constuint64_ttilingPtr,constuint64_tdataPtr);typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half>aType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half>bType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>cType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>biasType;AscendC:Matmul<aType,bType,cType,biasType,CFG_NORM,MatmulCallBackFunc<DataCopyOut,CopyA1,CopyB1>>mm;REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm,&tiling);uint64_ttilingPtr=reinterpret_cast<uint64_t>(tiling);mm.SetUserDefInfo(tilingPtr);mm.SetTensorA(gmA);mm.SetTensorB(gmB);mm.IterateAll(); |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
