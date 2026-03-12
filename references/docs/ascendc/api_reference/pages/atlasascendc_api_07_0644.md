# IterateNBatch-Matmul Kernel侧接口-矩阵计算-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0644
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0644.html
---

# IterateNBatch

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

调用一次IterateNBatch，会进行N次IterateBatch计算，计算出N个多Batch的singleCoreM * singleCoreN大小的C矩阵。在调用该接口前，需将MatmulConfig中的isNBatch参数设为true，使能多Batch输入多Batch输出功能，并调用SetWorkspace接口申请临时空间，用于缓存计算结果，即IterateNBatch的结果输出至SetWorkspace指定的Global Memory内存中。

对于BSNGD、SBNGD、BNGS1S2的Layout格式，调用该接口之前需要在tiling中使用SetALayout/SetBLayout/SetCLayout/SetBatchNum设置A/B/C的Layout轴信息和最大BatchNum数；对于Normal数据格式则需使用SetBatchInfoForNormal设置A/B/C的M/N/K轴信息和A/B矩阵的BatchNum数。实例化Matmul时，通过MatmulType设置Layout类型，当前支持3种Layout类型：BSNGD、SBNGD、BNGS1S2。

#### 函数原型

| 12  | template<boolsync=true,boolwaitIterateBatch=false>__aicore__inlinevoidIterateNBatch(constuint32_tbatchLoop,uint32_tbatchA,uint32_tbatchB,boolenSequentialWrite,constuint32_tmatrixStrideA=0,constuint32_tmatrixStrideB=0,constuint32_tmatrixStrideC=0) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名           | 描述                                                                                                                                                                                                                                                                                                                                                                                    |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sync             | 获取C矩阵过程分为同步和异步两种模式：同步：需要同步等待IterateNBatch执行结束，后续由开发者自行获取输出到Global Memory上的计算结果。异步：不需要同步等待IterateNBatch执行结束。通过该参数设置同步或者异步模式：同步模式设置为true；异步模式设置为false。默认为同步模式。                                                                                                                 |
| waitIterateBatch | 是否需要通过WaitIterateBatch接口等待IterateNBatch执行结束，仅在异步场景下使用。默认为false。true：需要通过WaitIterateBatch接口等待IterateNBatch执行结束，然后由开发者自行获取输出到Global Memory上的计算结果。false：不需要通过WaitIterateBatch接口等待IterateNBatch执行结束。调用本接口后，需要调用GetBatchTensorC接口获取C矩阵，或者由开发者自行处理等待IterateNBatch执行结束的过程。 |

| 参数名            | 输入/输出 | 描述                                                        |
| ----------------- | --------- | ----------------------------------------------------------- |
| batchLoop         | 输入      | 当前计算的BMM个数。                                         |
| batchA            | 输入      | 当前单次BMM调用计算左矩阵的batch数。                        |
| batchB            | 输入      | 当前单次BMM调用计算右矩阵的batch数，brc场景batchA/B不相同。 |
| enSequentialWrite | 输入      | 输出是否连续存放数据。                                      |
| matrixStrideA     | 输入      | A矩阵源操作数相邻nd矩阵起始地址间的偏移，默认值是0。        |
| matrixStrideB     | 输入      | B矩阵源操作数相邻nd矩阵起始地址间的偏移，默认值是0。        |
| matrixStrideC     | 输入      | 该参数预留，开发者无需关注。                                |

#### 返回值说明

无

#### 约束说明

- 单BMM内计算遵循之前的约束条件。
- 对于BSNGD、SBNGD、BNGS1S2 Layout格式，输入A、B矩阵多Batch数据总和应小于L1 Buffer的大小。
- 当使能MixDualMaster（双主模式）场景时，即模板参数enableMixDualMaster设置为true，不支持使用该接口。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960 | #include"kernel_operator.h"#include"lib/matmul_intf.h"extern"C"__global____aicore__voidkernel_matmul_rpc_batch(GM_ADDRaGM,GM_ADDRbGM,GM_ADDRcGM,GM_ADDRbiasGM,GM_ADDRtilingGM,GM_ADDRworkspaceGM,uint32_tisTransposeAIn,uint32_tisTransposeBIn,int32_tbatchA,int32_tbatchB){// 定义matmul typetypedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half,false,LayoutMode:BSNGD>aType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,half,true,LayoutMode:BSNGD>bType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float,false,LayoutMode:BNGS1S2>cType;typedefAscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,float>biasType;SetAtomicNone();// 初始化tiling数据TCubeTilingtiling;autotempTilingGM=(__gm__uint32_t*)tilingGM;autotempTiling=(uint32_t*)&tiling;for(inti=0;i<sizeof(TCubeTiling)/sizeof(int32_t);++i,++tempTilingGM,++tempTiling){*tempTiling=*tempTilingGM;}// 初始化gm数据AscendC:GlobalTensor<half>aGlobal;AscendC:GlobalTensor<half>bGlobal;AscendC:GlobalTensor<float>cGlobal;AscendC:GlobalTensor<float>biasGlobal;int32_tsizeA=tiling.ALayoutInfoB*tiling.ALayoutInfoS*tiling.ALayoutInfoN*tiling.ALayoutInfoG*tiling.ALayoutInfoD*sizeof(half);int32_tsizeB=tiling.BLayoutInfoB*tiling.BLayoutInfoS*tiling.BLayoutInfoN*tiling.BLayoutInfoG*tiling.BLayoutInfoD*sizeof(half);int32_tsizebias=tiling.CLayoutInfoB*tiling.CLayoutInfoN*tiling.CLayoutInfoG*tiling.CLayoutInfoS2*sizeof(float);aGlobal.SetGlobalBuffer(reinterpret_cast<__gm__half*>(aGM),sizeA);bGlobal.SetGlobalBuffer(reinterpret_cast<__gm__half*>(bGM),sizeB);biasGlobal.SetGlobalBuffer(reinterpret_cast<__gm__float*>(biasGM),sizebias);tiling.shareMode=0;tiling.shareL1Size=512*1024;tiling.shareL0CSize=128*1024;tiling.shareUbSize=0;intoffset_a=0,offset_b=0,offset_c=0,offset_bias=0;AscendC:GlobalTensor<A_T>gm_a;gm_a.SetGlobalBuffer(const_cast<__gm__half*>(aGlobal[offset_a].GetPhyAddr()),tiling.ALayoutInfoS*tiling.ALayoutInfoN*tiling.ALayoutInfoG*tiling.ALayoutInfoD);AscendC:GlobalTensor<B_T>gm_b;gm_b.SetGlobalBuffer(const_cast<__gm__half*>(bGlobal[offset_b].GetPhyAddr()),tiling.BLayoutInfoS*tiling.BLayoutInfoN*tiling.BLayoutInfoG*tiling.BLayoutInfoD);AscendC:GlobalTensor<BiasT>gm_bias;gm_bias.SetGlobalBuffer(const_cast<__gm__float*>(biasGlobal[offset_bias].GetPhyAddr()),tiling.CLayoutInfoN*tiling.CLayoutInfoG*tiling.CLayoutInfoS2);// 创建Matmul实例AscendC:Matmul<aType,bType,cType,biasType>mm1;AscendC:TPipepipe;g_cubeTPipePtr=&pipe;REGIST_MATMUL_OBJ(&pipe,GetSysWorkSpacePtr(),mm1);mm1.Init(&tiling);intg_lay=tiling.ALayoutInfoG>tiling.BLayoutInfoG?tiling.ALayoutInfoG:tiling.BLayoutInfoG;intfor_extent=tiling.ALayoutInfoB*tiling.ALayoutInfoN*g_lay/tiling.BatchNum;mm1.SetTensorA(gm_a[0],isTransposeAIn);mm1.SetTensorB(gm_b[0],isTransposeBIn);mm1.SetWorkspace(workspaceGM,0);if(tiling.isBias){mm1.SetBias(gm_bias[0]);}// 多batch Matmul计算mm1.IterateNBatch(for_extent,batchA,batchB,false);} |
| --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
