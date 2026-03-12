# REGISTER_NONE_TILING-Kernel Tiling-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00164
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00164.html
---

# REGISTER_NONE_TILING

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

在Kernel侧使用标准C++语法自定义的TilingData结构体时，若用户不确定需要注册哪些结构体，可使用该接口告知框架侧需使用未注册的标准C++语法来定义TilingData，并配套GET_TILING_DATA_WITH_STRUCT，GET_TILING_DATA_MEMBER，GET_TILING_DATA_PTR_WITH_STRUCT来获取对应的TilingData。

#### 函数原型

| 1   | REGISTER_NONE_TILING |
| --- | -------------------- |

#### 参数说明

无

#### 约束说明

- 暂不支持Kernel直调工程。
- 使用GET_TILING_DATA需提供默认注册的TilingData结构体，但本接口不注册TilingData结构体，故不支持与5.11.1-GET_TILING_DATA组合使用。
- 不支持和REGISTER_TILING_DEFAULT或REGISTER_TILING_FOR_TILINGKEY混用，即不支持注册TilingData结构体的场景与非注册场景混合使用。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051 | # Tiling模板库提供方，无法预知用户实例化何种TilingData结构体template<classBrcDag>structBroadcastBaseTilingData{int32_tscheMode;int32_tshapeLen;int32_tubSplitAxis;int32_tubFormer;int32_tubTail;int64_tubOuter;int64_tblockFormer;int64_tblockTail;int64_tdimProductBeforeUbInner;int64_telemNum;int64_tblockNum;int64_toutputDims[BROADCAST_MAX_DIMS_NUM];int64_toutputStrides[BROADCAST_MAX_DIMS_NUM];int64_tinputDims[BrcDag:InputSize][2];// 整块 + 尾块int64_tinputBrcDims[BrcDag:CopyBrcSize][BROADCAST_MAX_DIMS_NUM];int64_tinputVecBrcDims[BrcDag:VecBrcSize][BROADCAST_MAX_DIMS_NUM];int64_tinputStrides[BrcDag:InputSize][BROADCAST_MAX_DIMS_NUM];int64_tinputBrcStrides[BrcDag:CopyBrcSize][BROADCAST_MAX_DIMS_NUM];int64_tinputVecBrcStrides[BrcDag:VecBrcSize];charscalarData[BROADCAST_MAX_SCALAR_BYTES];};template<uint64_tschMode,classBrcDag>classBroadcastSch{public:__aicore__inlineexplicitBroadcastSch(GM_ADDR&tmpTiling):tiling(tmpTiling){}template<class...Args>__aicore__inlinevoidProcess(Args...args){REGISTER_NONE_TILING;// 告知框架侧使用未注册的TilingData结构体ifconstexpr(schMode==1){GET_TILING_DATA_WITH_STRUCT(BroadcastBaseTilingData<BrcDag>,tilingData,tiling);GET_TILING_DATA_MEMBER(BroadcastBaseTilingData<BrcDag>,blockNum,blockNumVar,tiling);TPipepipe;BroadcastNddmaSch<BrcDag,false>sch(&tilingData);// 获取Schedulesch.Init(&pipe,args...);sch.Process();}elseifconstexpr(schMode==202){GET_TILING_DATA_PTR_WITH_STRUCT(BroadcastOneDimTilingDataAdvance,tilingDataPtr,tiling);BroadcastOneDimAdvanceSch<BrcDag>sch(tilingDataPtr);// 获取Schedulesch.Init(args...);sch.Process();}}public:GM_ADDRtiling;}; |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1234567891011121314151617 | #用户通过传入schMode, OpDag模板参数来实例化模板库usingnamespaceAscendC;template<uint64_tschMode>__global____aicore__voidmul(GM_ADDRx1,GM_ADDRx2,GM_ADDRy,GM_ADDRworkspace,GM_ADDRtiling){ifconstexpr(std:is_same<DTYPE_X1,int8_t>:value){// int8usingOpDag=MulDag:MulInt8Op:OpDag;BroadcastSch<schMode,OpDag>sch(tiling);sch.Process(x1,x2,y);}elseifconstexpr(std:is_same<DTYPE_X1,uint8_t>:value){// uint8usingOpDag=MulDag:MulUint8Op:OpDag;BroadcastSch<schMode,OpDag>sch(tiling);sch.Process(x1,x2,y);}} |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
