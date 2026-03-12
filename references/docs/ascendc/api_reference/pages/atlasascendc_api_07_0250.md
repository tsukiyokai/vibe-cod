# MmadWithSparse-矩阵计算-矩阵计算(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0250
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0250.html
---

# MmadWithSparse

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

完成矩阵乘加操作，传入的左矩阵A为稀疏矩阵，右矩阵B为稠密矩阵。对于矩阵A，在MmadWithSparse计算时完成稠密化；对于矩阵B，在计算执行前的输入数据准备时自行完成稠密化（按照下文中介绍的稠密算法进行稠密化），所以输入本接口的B矩阵为稠密矩阵。B稠密矩阵需要通过调用LoadDataWithSparse载入，同时加载索引矩阵，索引矩阵在矩阵B稠密化的过程中生成，再用于A矩阵的稠密化。

#### 函数原型

| 12  | template<typenameT=int32_t,typenameU=int8_t,typenameStd:enable_if<Std:is_same<PrimT<T>,int32_t>:value,bool>:type=true,typenameStd:enable_if<Std:is_same<PrimT<U>,int8_t>:value,bool>:type=true>__aicore__inlinevoidMmadWithSparse(constLocalTensor<T>&dst,constLocalTensor<U>&fm,constLocalTensor<U>&filter,constMmadParams&mmadParams) |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                                                                                                                                                                                                |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T      | dst的数据类型。                                                                                                                                                                                                                                                                     |
| U      | fm、filter的数据类型。当dst、fm、filter为基础数据类型时，T必须为int32_t类型，U必须为int8_t类型，否则编译失败。当dst、fm、filter为TensorTrait类型时，T的LiteType必须为int32_t类型，U的LiteType必须为int8_t类型，否则编译失败。最后两个模板参数仅用于上述数据类型检查，用户无需关注。 |

| 参数名称   | 输入/输出 | 含义                                                                                                                                                                                         |
| ---------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dst        | 输出      | 目的操作数，结果矩阵，类型为LocalTensor，支持的TPosition为CO1。LocalTensor的起始地址需要256个元素（1024字节）对齐。                                                                          |
| fm         | 输入      | 源操作数，左矩阵A，类型为LocalTensor，支持的TPosition为A2。LocalTensor的起始地址需要512字节对齐。                                                                                            |
| filter     | 输入      | 源操作数，右矩阵B，类型为LocalTensor，支持的TPosition为B2。LocalTensor的起始地址需要512字节对齐。                                                                                            |
| mmadParams | 输入      | 矩阵乘相关参数，类型为MmadParams。具体定义请参考${INSTALL_DIR}/include/ascendc/basic_api/interface/kernel_struct_mm.h，${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。参数说明请参考表3。 |

#### 约束说明

- 原始稀疏矩阵B每4个元素中应保证最多2个非零元素，如果存在3个或更多非零元素，则仅使用前2个非零元素。
- 当M、K、N中的任意一个值为0时，该指令不会被执行。

- 操作数地址对齐要求请参见通用地址对齐约束。

#### 稠密算法说明

假设原始稀疏矩阵B的每4个元素中至少有2个零，稠密化后的矩阵B是一个在每4个元素中过滤掉2个零的稠密矩阵。矩阵B稠密化的过程中生成索引矩阵，过程如下：对于稀疏矩阵B中的每4个元素，将在index矩阵中生成2个2位索引，并按照以下规则进行编码。索引必须在{0, 1, 2}范围内。

- 第一个索引用于指示前3个元素中第1个非零元素的相对位置。
- 第二个索引用于指示第2个非零元素在后3个元素中的相对位置。

具体可参考下表。其中，“-”表示算法不关心该位置上的值，因为其会被过滤。

| 示例                  | ele0 | ele1 | ele2 | ele3  | Index_a[i] | Index_b[i] |
| --------------------- | ---- | ---- | ---- | ----- | ---------- | ---------- |
| Two non-zero elements | 0    | 0    | X    | Y     | 2’b10      | 2’b10      |
| 0                     | X    | 0    | Y    | 2’b01 | 2’b10      |            |
| X                     | 0    | 0    | Y    | 2’b00 | 2’b10      |            |
| 0                     | X    | Y    | -    | 2’b01 | 2’b01      |            |
| X                     | 0    | Y    | -    | 2’b00 | 2’b01      |            |
| X                     | Y    | -    | -    | 2’b00 | 2’b00      |            |
| One non-zero element  | 0    | 0    | 0    | X     | 2’b00      | 2’b10      |
| 0                     | 0    | X    | 0    | 2’b10 | 2’b00      |            |
| 0                     | X    | 0    | 0    | 2’b01 | 2’b00      |            |
| X                     | 0    | 0    | 0    | 2’b00 | 2’b00      |            |
| All zero              | 0    | 0    | 0    | 0     | 2’b00      | 2’b00      |

该索引矩阵用于A矩阵的稠密化，根据索引矩阵从MatrixA中的4个元素中选择2个元素参与计算，如下图所示：

![](../images/atlasascendc_api_07_0250_img_001.png)

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149150151152 | #include"kernel_operator.h"classKernelMatmul{public:__aicore__inlineKernelMatmul(){}__aicore__inlinevoidInit(__gm__uint8_t*a,__gm__uint8_t*b,__gm__uint8_t*idx,__gm__uint8_t*c,uint16_tm,uint16_tk,uint16_tn){this->m=m;this->k=k;this->n=n;aSize=m*k;bSize=k/2*n;cSize=m*n;mBlocks=m/16;nBlocks=n/16;kBlocks=k/32;aGM.SetGlobalBuffer((__gm__int8_t*)a);bGM.SetGlobalBuffer((__gm__int8_t*)b);idxGM.SetGlobalBuffer((__gm__uint8_t*)idx);cGM.SetGlobalBuffer((__gm__int32_t*)c);pipe.InitBuffer(inQueueA1,1,aSize*sizeof(int8_t));pipe.InitBuffer(inQueueA2,1,aSize*sizeof(int8_t));pipe.InitBuffer(inQueueB1,1,bSize*sizeof(int8_t));pipe.InitBuffer(inQueueIdxB1,1,(bSize/4)*sizeof(int8_t));pipe.InitBuffer(inQueueB2,1,bSize*sizeof(int8_t));pipe.InitBuffer(outQueueCO1,1,cSize*sizeof(int32_t));}__aicore__inlinevoidProcess(){CopyIn();SplitA();AscendC:LocalTensor<int8_t>b1Local=inQueueB1.DeQue<int8_t>();AscendC:LocalTensor<uint8_t>idexb1Local=inQueueIdxB1.DeQue<uint8_t>();AscendC:LocalTensor<int8_t>a2Local=inQueueA2.DeQue<int8_t>();SplitB(b1Local,idexb1Local);Compute(a2Local);inQueueB1.FreeTensor(b1Local);inQueueIdxB1.FreeTensor(idexb1Local);inQueueA2.FreeTensor(a2Local);CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<int8_t>a1Local=inQueueA1.AllocTensor<int8_t>();AscendC:LocalTensor<int8_t>b1Local=inQueueB1.AllocTensor<int8_t>();AscendC:LocalTensor<uint8_t>idxb1Local=inQueueIdxB1.AllocTensor<uint8_t>();AscendC:DataCopy(a1Local,aGM,{1,static_cast<uint16_t>(aSize*sizeof(int8_t)/32),0,0});AscendC:DataCopy(b1Local,bGM,{1,static_cast<uint16_t>(bSize*sizeof(int8_t)/32),0,0});AscendC:DataCopy(idxb1Local,idxGM,{1,static_cast<uint16_t>(bSize/4*sizeof(int8_t)/32),0,0});inQueueA1.EnQue(a1Local);inQueueB1.EnQue(b1Local);inQueueIdxB1.EnQue(idxb1Local);}__aicore__inlinevoidSplitA(){intsrcOffset=0;intdstOffset=0;AscendC:LocalTensor<int8_t>a1Local=inQueueA1.DeQue<int8_t>();AscendC:LocalTensor<int8_t>a2Local=inQueueA2.AllocTensor<int8_t>();AscendC:LoadData2DParamsloadDataParams;loadDataParams.repeatTimes=kBlocks*mBlocks;loadDataParams.srcStride=1;loadDataParams.ifTranspose=false;AscendC:LoadData(a2Local,a1Local,loadDataParams);inQueueA2.EnQue<int8_t>(a2Local);inQueueA1.FreeTensor(a1Local);}__aicore__inlinevoidSplitB(AscendC:LocalTensor<int8_t>&b1Local,AscendC:LocalTensor<uint8_t>&idxb1Local){AscendC:LocalTensor<int8_t>b2Local=inQueueB2.AllocTensor<int8_t>();// transform nz to znAscendC:LoadData2DParamsloadDataParams;loadDataParams.repeatTimes=kBlocks*nBlocks/2;loadDataParams.srcStride=0;loadDataParams.ifTranspose=false;AscendC:LoadDataWithSparse(b2Local,b1Local,idxb1Local,loadDataParams);inQueueB2.EnQue<int8_t>(b2Local);}__aicore__inlinevoidCompute(constAscendC:LocalTensor<int8_t>&a2Local){AscendC:LocalTensor<int8_t>b2Local=inQueueB2.DeQue<int8_t>();AscendC:LocalTensor<int32_t>c1Local=outQueueCO1.AllocTensor<int32_t>();AscendC:MmadWithSparse(c1Local,a2Local,b2Local,{m,n,k,false,0,false,false,false});outQueueCO1.EnQue<int32_t>(c1Local);inQueueB2.FreeTensor(b2Local);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<int32_t>c1Local=outQueueCO1.DeQue<int32_t>();AscendC:FixpipeParamsV220fixpipeParams;fixpipeParams.nSize=n;fixpipeParams.mSize=m;fixpipeParams.srcStride=m;fixpipeParams.dstStride=n;fixpipeParams.ndNum=1;fixpipeParams.srcNdStride=0;fixpipeParams.dstNdStride=0;AscendC:Fixpipe(cGM,c1Local,fixpipeParams);outQueueCO1.FreeTensor(c1Local);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:A1,1>inQueueA1;AscendC:TQue<AscendC:TPosition:A2,1>inQueueA2;AscendC:TQue<AscendC:TPosition:B1,1>inQueueB1;AscendC:TQue<AscendC:TPosition:B1,1>inQueueIdxB1;AscendC:TQue<AscendC:TPosition:B2,1>inQueueB2;// dst queueAscendC:TQue<AscendC:TPosition:CO1,1>outQueueCO1;AscendC:GlobalTensor<int8_t>aGM,bGM;AscendC:GlobalTensor<uint8_t>idxGM;AscendC:GlobalTensor<int32_t>cGM;uint16_tm;uint16_tn;uint16_tk;uint16_taSize,bSize,cSize,mBlocks,nBlocks,kBlocks;};#define KERNEL_MMAD_WITH_SPARSE_OPERATOR_TEST(m, k, n)                                        \extern "C" __global__ __aicore__ void kernel_mmad_with_sparse_operator##_##m##_##k##_##n( \GM_ADDR a, GM_ADDR b, GM_ADDR idx, GM_ADDR c)                                         \{                                                                                         \KernelMatmul op;                                                                      \op.Init(a, b, idx, c, m, k, n);                                                       \op.Process();                                                                         \}KERNEL_MMAD_WITH_SPARSE_OPERATOR_TEST(16,64,16) |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
