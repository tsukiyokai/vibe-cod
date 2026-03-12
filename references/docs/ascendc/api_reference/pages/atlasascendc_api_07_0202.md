# IBSet-核间同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0202
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0202.html
---

# IBSet

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

当不同核之间操作同一块全局内存且可能存在读后写、写后读以及写后写等数据依赖问题时，通过调用该函数来插入同步语句来避免上述数据依赖时可能出现的数据读写错误问题。调用IBSet设置某一个核的标志位，与IBWait成对出现配合使用，表示核之间的同步等待指令，等待某一个核操作完成。

#### 函数原型

| 12  | template<boolisAIVOnly=true>__aicore__inlinevoidIBSet(constGlobalTensor<int32_t>&gmWorkspace,constLocalTensor<int32_t>&ubWorkspace,int32_tblockIdx,int32_teventID) |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

#### 参数说明

| 参数名    | 描述                                |
| --------- | ----------------------------------- |
| isAIVOnly | 控制是否为AIVOnly模式，默认为true。 |

| 参数名      | 输入/输出 | 描述                                                                                         |
| ----------- | --------- | -------------------------------------------------------------------------------------------- |
| gmWorkspace | 输出      | 外部存储核状态的公共缓存，类型为GlobalTensor。GlobalTensor数据结构的定义请参考GlobalTensor。 |
| ubWorkspace | 输入      | 存储当前核状态的公共缓存。类型为LocalTensor，支持的TPosition为VECIN/VECCALC/VECOUT。         |
| blockIdx    | 输入      | 表示等待核的idx号，取值范围：[0, 核数-1]。                                                   |
| eventID     | 输入      | 用来控制当前核的set、wait事件。                                                              |

#### 返回值说明

无

#### 约束说明

- gmWorkspace申请的空间最少要求为：核数 * 32Bytes * eventID_max + blockIdx_max * 32Bytes + 32Bytes。（eventID_max和blockIdx_max分别指eventID、blockIdx的最大值）；
- 注意：如果是AIVOnly模式，核数 = GetBlockNum()；如果是MIX模式，核数 = GetBlockNum() * 2；
- ubWorkspace申请的空间最少要求为：32Bytes；
- gmWorkspace缓存的值需要初始化为0。
- 使用该接口进行多核控制时，算子调用时指定的逻辑blockDim必须保证不大于实际运行该算子的AI处理器核数，否则框架进行多轮调度时会插入异常同步，导致Kernel“卡死”现象。

#### 调用示例

| 123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384 | #include"kernel_operator.h"constexprint32_tTOTAL_LENGTH=2*256;constexprint32_tUSE_CORE_NUM=2;constexprint32_tBLOCK_LENGTH=TOTAL_LENGTH/USE_CORE_NUM;classKernelAdd{public:__aicore__inlineKernelAdd(){}__aicore__inlinevoidInit(__gm__uint8_t*x,__gm__uint8_t*y,__gm__uint8_t*sync,__gm__uint8_t*z){blockIdx=AscendC:GetBlockIdx();xGm.SetGlobalBuffer((__gm__half*)x);yGm.SetGlobalBuffer((__gm__half*)y);sync_gm.SetGlobalBuffer((__gm__int32_t*)(sync),256);zGm.SetGlobalBuffer((__gm__half*)z);pipe.InitBuffer(inQueueX,1,BLOCK_LENGTH*sizeof(half));pipe.InitBuffer(inQueueY,1,BLOCK_LENGTH*sizeof(half));pipe.InitBuffer(vecIn,1,8*sizeof(int32_t));pipe.InitBuffer(outQueueZ,1,BLOCK_LENGTH*sizeof(half));}__aicore__inlinevoidProcess(){if(blockIdx==1){autosync_buf=vecIn.AllocTensor<int32_t>();AscendC:IBWait(sync_gm,sync_buf,0,0);vecIn.FreeTensor(sync_buf);}CopyIn();Compute();CopyOut();if(blockIdx==0){autosync_buf=vecIn.AllocTensor<int32_t>();AscendC:IBSet(sync_gm,sync_buf,0,0);vecIn.FreeTensor(sync_buf);}}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<half>xLocal=inQueueX.AllocTensor<half>();AscendC:LocalTensor<half>yLocal=inQueueY.AllocTensor<half>();if(blockIdx==1){AscendC:DataCopy(xLocal,zGm[0*BLOCK_LENGTH],BLOCK_LENGTH);AscendC:DataCopy(yLocal,yGm[1*BLOCK_LENGTH],BLOCK_LENGTH);}else{AscendC:DataCopy(xLocal,xGm[0],BLOCK_LENGTH);AscendC:DataCopy(yLocal,yGm[0],BLOCK_LENGTH);}inQueueX.EnQue(xLocal);inQueueY.EnQue(yLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<half>xLocal=inQueueX.DeQue<half>();AscendC:LocalTensor<half>yLocal=inQueueY.DeQue<half>();AscendC:LocalTensor<half>zLocal=outQueueZ.AllocTensor<half>();AscendC:Add(zLocal,xLocal,yLocal,BLOCK_LENGTH);outQueueZ.EnQue<half>(zLocal);inQueueX.FreeTensor(xLocal);inQueueY.FreeTensor(yLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<half>zLocal=outQueueZ.DeQue<half>();AscendC:DataCopy(zGm[blockIdx*BLOCK_LENGTH],zLocal,BLOCK_LENGTH);outQueueZ.FreeTensor(zLocal);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX,inQueueY,vecIn;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueueZ;AscendC:GlobalTensor<half>xGm,yGm,zGm;AscendC:GlobalTensor<int32_t>sync_gm;int32_tblockIdx=0;};extern"C"__global____aicore__voidadd_simple_kernel(__gm__uint8_t*x,__gm__uint8_t*y,__gm__uint8_t*sync,__gm__uint8_t*z){KernelAddop;op.Init(x,y,sync,z);op.Process();} |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
