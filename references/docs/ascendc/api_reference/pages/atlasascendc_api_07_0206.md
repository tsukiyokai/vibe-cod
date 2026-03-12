# InitDetermineComputeWorkspace-核间同步-同步控制-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0206
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0206.html
---

# InitDetermineComputeWorkspace

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | x        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

初始化GM共享内存的值，完成初始化后才可以调用WaitPreBlock和NotifyNextBlock。

#### 函数原型

| 1   | __aicore__inlinevoidInitDetermineComputeWorkspace(GlobalTensor<int32_t>&gmWorkspace,LocalTensor<int32_t>&ubWorkspace) |
| --- | --------------------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名称    | 输入/输出 | 含义                                                     |
| ----------- | --------- | -------------------------------------------------------- |
| gmWorkspace | 输入      | 临时空间，初始化核间同步的共享内存，类型为GlobalTensor。 |
| ubWorkspace | 输入      | 临时空间，用于操作gmWorkspace，类型为LocalTensor。       |

#### 返回值说明

无

#### 约束说明

- gmWorkspace申请的空间最少要求为：blockNum * 32Bytes；ubWorkspace申请的空间最少要求为：blockNum * 32 + 32Bytes；其中blockNum为调用的核数，可调用GetBlockNum获取。
- 使用该接口进行多核控制时，算子调用时指定的逻辑blockDim必须保证不大于实际运行该算子的AI处理器核数，否则框架进行多轮调度时会插入异常同步，导致Kernel“卡死”现象。

#### 调用示例

如下示例模拟8个核进行数据处理，使用确定性计算接口保证核间运行顺序，进行原子累加。

| 1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162 | #include"kernel_operator.h"template<typenameT>classSyncTest{public:__aicore__inlineSyncTest(){}__aicore__inlinevoidInit(GM_ADDRdstGm,GM_ADDRsrcGm,GM_ADDRgmWorkspace,constDetermineComputeSyncTilingData&tiling_data){m_elementCount=tiling_data.size;m_tileNum=tiling_data.tileNum;m_tileCount=m_elementCount/m_tileNum;m_dstGlobal.SetGlobalBuffer((__gm__T*)dstGm);m_srcGlobal.SetGlobalBuffer((__gm__T*)srcGm);m_gmWorkspace.SetGlobalBuffer((__gm__int32_t*)gmWorkspace);m_pipe.InitBuffer(m_que,1,m_elementCount*sizeof(T));m_pipe.InitBuffer(m_queTmp,1,8*sizeof(int32_t));}__aicore__inlinevoidProcess(){AscendC:LocalTensor<int32_t>ubWorkspace=m_queTmp.AllocTensor<int32_t>();AscendC:InitDetermineComputeWorkspace(m_gmWorkspace,ubWorkspace);for(int64_ti=0;i<m_tileNum;i++){// copy inAscendC:LocalTensor<T>srcLocal=m_que.AllocTensor<T>();AscendC:DataCopy(srcLocal,m_srcGlobal[i*m_tileCount],m_tileCount);// copy outAscendC:WaitPreBlock(m_gmWorkspace,ubWorkspace);AscendC:SetAtomicAdd<T>();AscendC:DataCopy(m_dstGlobal[i*m_tileCount],srcLocal,m_tileCount);AscendC:SetAtomicNone();AscendC:NotifyNextBlock(m_gmWorkspace,ubWorkspace);m_que.FreeTensor(srcLocal);}m_queTmp.FreeTensor(ubWorkspace);}private:AscendC:TPipem_pipe;int64_tm_elementCount;int64_tm_tileNum;int64_tm_tileCount;AscendC:GlobalTensor<T>m_srcGlobal;AscendC:GlobalTensor<T>m_dstGlobal;AscendC:GlobalTensor<int32_t>m_gmWorkspace;AscendC:TQue<AscendC:TPosition:VECIN,1>m_que;AscendC:TQue<AscendC:TPosition:VECIN,1>m_queTmp;};// class SyncTestextern"C"__global____aicore__voiddetermine_compute_sync(GM_ADDRx,GM_ADDRy,GM_ADDRworkspace,GM_ADDRtiling){GET_TILING_DATA(tiling_data,tiling);GM_ADDRusrWorkspace=AscendC:GetUserWorkspace(workspace);// 获取用户workspace指针SyncTest<float>op;op.Init(y,x,usrWorkspace,tiling_data);op.Process();} |
| ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
