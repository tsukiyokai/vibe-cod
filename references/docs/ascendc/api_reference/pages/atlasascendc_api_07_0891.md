# Fill-张量变换-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0891
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0891.html
---

# Fill

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

将Global Memory上的数据初始化为指定值。该接口可用于对workspace地址或输出数据进行清零。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidFill(GlobalTensor<T>&gmWorkspaceAddr,constuint64_tsize,constTvalue) |
| --- | ---------------------------------------------------------------------------------------------------------- |

#### 参数说明

| 参数名 | 含义                                                                                                                                                                                                                                                                                                                                                  |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T      | 操作数的数据类型。Atlas A3 训练系列产品/Atlas A3 推理系列产品，支持的数据类型为：uint16_t、int16_t、half、uint32_t、int32_t、float。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持的数据类型为：uint16_t、int16_t、half、uint32_t、int32_t、float。Atlas推理系列产品AI Core，支持的数据类型为：uint16_t、int16_t、half、uint32_t、int32_t、float。 |

| 参数名          | 输入/输出 | 含义                                                                                                                                |
| --------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| gmWorkspaceAddr | 输入      | gmWorkspaceAddr为用户定义的全局Global空间，是需要被初始化的空间，类型为GlobalTensor。GlobalTensor数据结构的定义请参考GlobalTensor。 |
| size            | 输入      | 需要初始化的空间大小，单位为元素个数。                                                                                              |
| value           | 输入      | 初始化的值，数据类型与gmWorkspaceAddr保持一致。                                                                                     |

#### 返回值说明

无

#### 约束说明

- 单核调用此接口时，如果后续操作涉及Unified Buffer的使用，则需要在调用接口后，设置MTE2流水等待MTE3流水(MTE3_MTE2)的同步。
- 当多个核调用此接口对Global Memory进行初始化时，所有核对Global Memory的初始化未必会同时结束，也可能存在核之间读后写、写后读以及写后写等数据依赖问题。这种使用场景下，可以在本接口后调用SyncAll接口保证多核间同步正确。
- 该接口仅支持在程序内存分配InitBuffer接口前使用。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273 | #include"kernel_operator.h"constexprint32_tINIT_SIZE=65536;classKernelFill{public:__aicore__inlineKernelFill(){}__aicore__inlinevoidInit(GM_ADDRx,GM_ADDRy,GM_ADDRz,TPipe*pipe){xGm.SetGlobalBuffer((__gm__half*)x+INIT_SIZE*AscendC:GetBlockIdx(),INIT_SIZE);yGm.SetGlobalBuffer((__gm__half*)y+INIT_SIZE*AscendC:GetBlockIdx(),INIT_SIZE);zGm.SetGlobalBuffer((__gm__half*)z+INIT_SIZE*AscendC:GetBlockIdx(),INIT_SIZE);// init zGm valueAscendC:Fill(zGm,INIT_SIZE,(half)(AscendC:GetBlockIdx()));AscendC:TEventIDeventIdMTE3ToMTE2=GetTPipePtr()->FetchEventID(AscendC:HardEvent:MTE3_MTE2);AscendC:SetFlag<AscendC:HardEvent:MTE3_MTE2>(eventIdMTE3ToMTE2);AscendC:WaitFlag<AscendC:HardEvent:MTE3_MTE2>(eventIdMTE3ToMTE2);pipe->InitBuffer(inQueueX,1,INIT_SIZE*sizeof(half));pipe->InitBuffer(inQueueY,1,INIT_SIZE*sizeof(half));pipe->InitBuffer(outQueueZ,1,INIT_SIZE*sizeof(half));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<half>xLocal=inQueueX.AllocTensor<half>();AscendC:LocalTensor<half>yLocal=inQueueY.AllocTensor<half>();AscendC:DataCopy(xLocal,xGm,INIT_SIZE);AscendC:DataCopy(yLocal,yGm,INIT_SIZE);inQueueX.EnQue(xLocal);inQueueY.EnQue(yLocal);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<half>xLocal=inQueueX.DeQue<half>();AscendC:LocalTensor<half>yLocal=inQueueY.DeQue<half>();AscendC:LocalTensor<half>zLocal=outQueueZ.AllocTensor<half>();AscendC:Add(zLocal,xLocal,yLocal,INIT_SIZE);outQueueZ.EnQue<half>(zLocal);inQueueX.FreeTensor(xLocal);inQueueY.FreeTensor(yLocal);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<half>zLocal=outQueueZ.DeQue<half>();// add result to zGmAscendC:SetAtomicAdd<half>();AscendC:DataCopy(zGm,zLocal,INIT_SIZE);AscendC:SetAtomicNone();outQueueZ.FreeTensor(zLocal);}private:AscendC:TQue<AscendC:TPosition:VECIN,1>inQueueX,inQueueY;AscendC:TQue<AscendC:TPosition:VECOUT,1>outQueueZ;AscendC:GlobalTensor<half>xGm;AscendC:GlobalTensor<half>yGm;AscendC:GlobalTensor<half>zGm;};extern"C"__global____aicore__voidinit_global_memory_custom(GM_ADDRx,GM_ADDRy,GM_ADDRz){KernelFillop;TPipepipe;op.Init(x,y,z,&pipe);op.Process();} |
| ----------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

结果示例如下：

| 12345678910111213 | 输入数据(x):[1.1.1.1.1....1.]输入数据(y):[1.1.1.1.1....1.]输出数据(z):[2.2.2.2.2....2.3.3.3.3.3....3.4.4.4.4.4....4.5.5.5.5.5....5.6.6.6.6.6....6.7.7.7.7.7....7.8.8.8.8.8....8.9.9.9.9.9....9.] |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
