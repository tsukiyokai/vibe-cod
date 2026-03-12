# InitBufPool-TBufPool-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0124
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0124.html
---

# InitBufPool

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

通过Tpipe:InitBufPool接口可划分出整块资源，整块TbufPool资源可以继续通过TBufPool:InitBufPool接口划分成小块资源。

#### 函数原型

- 非共享模式12template<classT>__aicore__inlineboolInitBufPool(T&bufPool,uint32_tlen)
- 共享模式12template<classT,classU>__aicore__inlineboolInitBufPool(T&bufPool,uint32_tlen,U&shareBuf)

#### 参数说明

| 参数名 | 说明                 |
| ------ | -------------------- |
| T      | bufPool参数的类型。  |
| U      | shareBuf参数的类型。 |

| 参数名称 | 输入/输出 | 含义                                                                   |
| -------- | --------- | ---------------------------------------------------------------------- |
| bufPool  | 输入      | 新划分的资源池，类型为TBufPool。                                       |
| len      | 输入      | 新划分资源池长度，单位为字节，非32字节对齐会自动向上补齐至32字节对齐。 |

| 参数名称 | 输入/输出 | 含义                                                                         |
| -------- | --------- | ---------------------------------------------------------------------------- |
| bufPool  | 输入      | 新划分的资源池，类型为TBufPool。                                             |
| len      | 输入      | 新划分资源池长度，单位为字节，非32字节对齐会自动向上补齐至32字节对齐。       |
| shareBuf | 输入      | 被复用资源池，类型为TBufPool，新划分资源池与被复用资源池共享起始地址及长度。 |

#### 约束说明

1. 新划分的资源池与被复用资源池的物理内存需要一致，两者共享起始地址及长度；
1. 输入长度需要小于等于被复用资源池长度；
1. 其他泛用约束参考TBufPool；

#### 返回值说明

无

#### 调用示例

数据量较大且内存有限时，无法一次完成所有数据搬运，需要拆分成多个阶段计算，每次计算使用其中的一部分数据，可以通过TBufPool资源池进行内存地址复用。本例中，从Tpipe划分出资源池tbufPool0，tbufPool0为src0Gm分配空间后，继续分配了资源池tbufPool1，指定tbufPool1与tbufPool2复用并分别运用于第一、二轮计算，此时tbufPool1及tbufPool2共享起始地址及长度。

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879808182838485868788899091929394 | classResetApi{public:__aicore__inlineResetApi(){}__aicore__inlinevoidInit(__gm__uint8_t*src0Gm,__gm__uint8_t*src1Gm,__gm__uint8_t*dstGm){src0Global.SetGlobalBuffer((__gm__half*)src0Gm);src1Global.SetGlobalBuffer((__gm__half*)src1Gm);dstGlobal.SetGlobalBuffer((__gm__half*)dstGm);pipe.InitBufPool(tbufPool0,131072);tbufPool0.InitBuffer(srcQue0,1,65536);// Total src0tbufPool0.InitBufPool(tbufPool1,65536);tbufPool0.InitBufPool(tbufPool2,65536,tbufPool1);}__aicore__inlinevoidProcess(){tbufPool1.InitBuffer(srcQue1,1,32768);tbufPool1.InitBuffer(dstQue0,1,32768);CopyIn();Compute();CopyOut();tbufPool1.Reset();tbufPool2.InitBuffer(srcQue2,1,32768);tbufPool2.InitBuffer(dstQue1,1,32768);CopyIn1();Compute1();CopyOut1();tbufPool2.Reset();tbufPool0.Reset();pipe.Reset();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<half>src0Local=srcQue0.AllocTensor<half>();AscendC:LocalTensor<half>src1Local=srcQue1.AllocTensor<half>();AscendC:DataCopy(src0Local,src0Global,16384);AscendC:DataCopy(src1Local,src1Global,16384);srcQue0.EnQue(src0Local);srcQue1.EnQue(src1Local);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<half>src0Local=srcQue0.DeQue<half>();AscendC:LocalTensor<half>src1Local=srcQue1.DeQue<half>();AscendC:LocalTensor<half>dstLocal=dstQue0.AllocTensor<half>();AscendC:Add(dstLocal,src0Local,src1Local,16384);dstQue0.EnQue<half>(dstLocal);srcQue0.FreeTensor(src0Local);srcQue1.FreeTensor(src1Local);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<half>dstLocal=dstQue0.DeQue<half>();AscendC:DataCopy(dstGlobal,dstLocal,16384);dstQue0.FreeTensor(dstLocal);}__aicore__inlinevoidCopyIn1(){AscendC:LocalTensor<half>src0Local=srcQue0.AllocTensor<half>();AscendC:LocalTensor<half>src1Local=srcQue2.AllocTensor<half>();AscendC:DataCopy(src0Local,src0Global[16384],16384);AscendC:DataCopy(src1Local,src1Global[16384],16384);srcQue0.EnQue(src0Local);srcQue2.EnQue(src1Local);}__aicore__inlinevoidCompute1(){AscendC:LocalTensor<half>src0Local=srcQue0.DeQue<half>();AscendC:LocalTensor<half>src1Local=srcQue2.DeQue<half>();AscendC:LocalTensor<half>dstLocal=dstQue1.AllocTensor<half>();AscendC:Add(dstLocal,src0Local,src1Local,16384);dstQue1.EnQue<half>(dstLocal);srcQue0.FreeTensor(src0Local);srcQue2.FreeTensor(src1Local);}__aicore__inlinevoidCopyOut1(){AscendC:LocalTensor<half>dstLocal=dstQue1.DeQue<half>();AscendC:DataCopy(dstGlobal[16384],dstLocal,16384);dstQue1.FreeTensor(dstLocal);}private:AscendC:TPipepipe;AscendC:TBufPool<AscendC:TPosition:VECCALC>tbufPool0,tbufPool1,tbufPool2;AscendC:TQue<AscendC:TPosition:VECIN,1>srcQue0,srcQue1,srcQue2;AscendC:TQue<AscendC:TPosition:VECOUT,1>dstQue0,dstQue1;AscendC:GlobalTensor<half>src0Global,src1Global,dstGlobal;};extern"C"__global____aicore__voidtbufpool_kernel(__gm__uint8_t*src0Gm,__gm__uint8_t*src1Gm,__gm__uint8_t*dstGm){ResetApiop;op.Init(src0Gm,src1Gm,dstGm);op.Process();} |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
