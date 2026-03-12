# 更多样例-基础算术-矢量计算-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0052
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0052.html
---

# 更多样例

- 通过tensor高维切分计算接口中的mask连续模式，实现数据非连续计算。12uint64_tmask=64;// 每个迭代内只计算前64个数AscendC:Add(dstLocal,src0Local,src1Local,mask,4,{1,1,1,8,8,8});结果示例如下：输入数据src0Local：[1 2 3 ... 512]
输入数据src1Local：[513 514 515 ... 1024]
输出数据dstLocal：
[514 516 518 ... 640 undefined ... undefined
 770 772 774 ... 896 undefined ... undefined
 1026 1028 1030 ... 1152 undefined ... undefined
 1282 1284 1286 ... 1408 undefined ... undefined]
- 通过tensor高维切分计算接口中的mask逐比特模式，实现数据非连续计算。12uint64_tmask[2]={UINT64_MAX,0};// mask[0]满，mask[1]空，每次只计算前64个数AscendC:Add(dstLocal,src0Local,src1Local,mask,4,{1,1,1,8,8,8});结果示例如下：输入数据src0Local：[1 2 3 ... 512]
输入数据src1Local：[513 514 515 ... 1024]
输出数据dstLocal：
[514 516 518 ... 640 undefined ... undefined
 770 772 774 ... 896 undefined ... undefined
 1026 1028 1030 ... 1152 undefined ... undefined
 1282 1284 1286 ... 1408 undefined ... undefined]
- 通过控制tensor高维切分计算接口的Repeat Stride参数，实现数据非连续计算。12345uint64_tmask=128;// repeatTime设置为2，表示一共需要进行2次迭代// src0BlkStride, src1BlkStride设置为1，表示每个迭代内src0参与计算的数据地址间隔为1个DataBlock// src0RepStride设置为16, 表示相邻迭代之间src0起始地址间隔为16个datablockAscendC:Add(dstLocal,src0Local,src1Local,mask,2,{1,1,1,8,16,8});结果示例如下：输入数据src0Local：[1 2 3 ... 512]
输入数据src1Local：[513 514 515 ... 1024]
输出数据dstLocal：
[514 516 518 ...768 898 900 902 ... 1150 1152 undefined ... undefined]
- 通过控制tensor高维切分计算接口的DataBlock Stride和Repeat Stride参数，实现数据非连续计算。12345uint64_tmask=128;// repeatTime设置为2，表示一共需要进行2次迭代// src0BlkStride设置为2，表示每个迭代内src0参与计算的数据地址间隔为2个datablock// src0RepStride设置为16, 表示相邻迭代之间src0起始地址间隔为16个datablockAscendC:Add(dstLocal,src0Local,src1Local,mask,2,{1,2,1,8,16,8});结果示例如下：输入数据src0Local：[1 2 3 ... 512]
输入数据src1Local：[513 514 515 ... 1024]
输出数据dstLocal：
[514 516 518 ... 544  562 564 566 ... 592  610 612 614 ... 640  658 660 662 ... 688
 706 708 710 ... 736  754 756 758 ... 784  802 804 806 ... 832  850 852 854 ... 880 
 898 900 902 ... 928  946 948 950 ... 976  994 996 998 ... 1024  1042 1044 1046 ... 1072
1090 1092 1094 ... 1120  1138 1140 1142 ... 1168  1186 1188 1190 ... 1216 1234 1236 1238 … 1264
undefined ... undefined]
- 需要传入标量参数的API使用样例。1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768#include"kernel_operator.h"constexprint32_tBUFFER_NUM=2;classKernelBinaryScalar{public:__aicore__inlineKernelBinaryScalar(){}__aicore__inlinevoidInit(GM_ADDRx,GM_ADDRz,floatscalar,uint32_ttotalLength,uint32_ttileNum){ASSERT(GetBlockNum()!=0&&"block dim can not be zero!");this->blockLength=totalLength/AscendC:GetBlockNum();this->scalar=scalar;this->tileNum=tileNum;ASSERT(tileNum!=0&&"tile num can not be zero!");this->tileLength=this->blockLength/tileNum/BUFFER_NUM;xGm.SetGlobalBuffer((__gm__DTYPE_X*)x+this->blockLength*AscendC:GetBlockIdx(),this->blockLength);zGm.SetGlobalBuffer((__gm__DTYPE_Z*)z+this->blockLength*AscendC:GetBlockIdx(),this->blockLength);pipe.InitBuffer(inQueueX,BUFFER_NUM,this->tileLength*sizeof(DTYPE_X));pipe.InitBuffer(outQueueZ,BUFFER_NUM,this->tileLength*sizeof(DTYPE_Z));}__aicore__inlinevoidProcess(){int32_tloopCount=this->tileNum*BUFFER_NUM;for(int32_ti=0;i<loopCount;i++){CopyIn(i);Compute(i);CopyOut(i);}}private:__aicore__inlinevoidCopyIn(int32_tprogress){AscendC:LocalTensor<DTYPE_X>xLocal=inQueueX.AllocTensor<DTYPE_X>();AscendC:DataCopy(xLocal,xGm[progress*this->tileLength],this->tileLength);inQueueX.EnQue(xLocal);}__aicore__inlinevoidCompute(int32_tprogress){AscendC:LocalTensor<DTYPE_X>xLocal=inQueueX.DeQue<DTYPE_X>();AscendC:LocalTensor<DTYPE_Z>zLocal=outQueueZ.AllocTensor<DTYPE_Z>();AscendC:Adds(zLocal,xLocal,(DTYPE_X)scalar,this->tileLength);outQueueZ.EnQue<DTYPE_Z>(zLocal);inQueueX.FreeTensor(xLocal);}__aicore__inlinevoidCopyOut(int32_tprogress){AscendC:LocalTensor<DTYPE_Z>zLocal=outQueueZ.DeQue<DTYPE_Z>();AscendC:DataCopy(zGm[progress*this->tileLength],zLocal,this->tileLength);outQueueZ.FreeTensor(zLocal);}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,BUFFER_NUM>inQueueX;AscendC:TQue<AscendC:TPosition:VECOUT,BUFFER_NUM>outQueueZ;AscendC:GlobalTensor<DTYPE_X>xGm;AscendC:GlobalTensor<DTYPE_Z>zGm;floatscalar;uint32_tblockLength;uint32_ttileNum;uint32_ttileLength;};extern"C"__global____aicore__voidbinary_scalar_simple_kernel(GM_ADDRx,GM_ADDRz,GM_ADDRworkspace,GM_ADDRtiling){GET_TILING_DATA(tilingData,tiling);KernelBinaryScalarop;op.Init(x,z,tilingData.scalar,tilingData.totalLength,tilingData.tileNum);if(TILING_KEY_IS(1)){op.Process();}}
