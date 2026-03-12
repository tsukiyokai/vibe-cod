# SetAtomicMax(ISASI)-原子操作-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0284
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0284.html
---

# SetAtomicMax(ISASI)

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

原子操作函数，设置后续从VECOUT传输到GM的数据是否执行原子比较：将待拷贝的内容和GM已有内容进行比较，将最大值写入GM。

可通过设置模板参数来设定不同的数据类型。

#### 函数原型

| 12  | template<typenameT>__aicore__inlinevoidSetAtomicMax() |
| --- | ----------------------------------------------------- |

#### 参数说明

| 参数名 | 描述                                                                                                                                                                                                           |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T      | 设定不同的数据类型。Atlas A2 训练系列产品/Atlas A2 推理系列产品，支持int8_t/int16_t/half/bfloat16_t/int32_t/floatAtlas A3 训练系列产品/Atlas A3 推理系列产品，支持int8_t/int16_t/half/bfloat16_t/int32_t/float |

#### 返回值说明

无

#### 约束说明

- 使用完后，建议通过SetAtomicNone关闭原子最大操作，以免影响后续相关功能。
- 对于Atlas A2 训练系列产品/Atlas A2 推理系列产品，目前无法对bfloat16_t类型设置inf/nan模式。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061626364656667686970717273747576777879 | // 本演示示例使用DataCopy从VECOUT搬出到外部dstGlobal时进行原子最大操作。#include"kernel_operator.h"staticconstintdata_size=256;template<typenameT>classKernelDataCopyAtomicMax{public:__aicore__inlineKernelDataCopyAtomicMax(){}__aicore__inlinevoidInit(GM_ADDRsrc0_gm,GM_ADDRsrc1_gm,GM_ADDRdst_gm,uint32_tsize){this->size=size;src0Global.SetGlobalBuffer((__gm__T*)src0_gm);src1Global.SetGlobalBuffer((__gm__T*)src1_gm);dstGlobal.SetGlobalBuffer((__gm__T*)dst_gm);pipe.InitBuffer(queueSrc0,1,size*sizeof(T));pipe.InitBuffer(queueSrc1,1,size*sizeof(T));pipe.InitBuffer(queueDst0,1,size*sizeof(T));pipe.InitBuffer(queueDst1,1,size*sizeof(T));}__aicore__inlinevoidProcess(){CopyIn();Compute();CopyOut();}private:__aicore__inlinevoidCopyIn(){AscendC:LocalTensor<T>src0local=queueSrc0.AllocTensor<T>();AscendC:LocalTensor<T>src1local=queueSrc1.AllocTensor<T>();AscendC:DataCopy(src0local,src0Global,size);AscendC:DataCopy(src1local,src1Global,size);queueSrc0.EnQue(src0local);queueSrc1.EnQue(src1local);}__aicore__inlinevoidCompute(){AscendC:LocalTensor<T>src0local=queueSrc0.DeQue<T>();AscendC:LocalTensor<T>src1local=queueSrc1.DeQue<T>();AscendC:LocalTensor<T>dst0Local=queueDst0.AllocTensor<T>();AscendC:LocalTensor<T>dst1Local=queueDst1.AllocTensor<T>();AscendC:Abs(dst0Local,src0local,size);AscendC:Abs(dst1Local,src1local,size);queueDst0.EnQue(dst0Local);queueDst1.EnQue(dst1Local);queueSrc0.FreeTensor(src0local);queueSrc1.FreeTensor(src1local);}__aicore__inlinevoidCopyOut(){AscendC:LocalTensor<T>dst0Local=queueDst0.DeQue<T>();AscendC:LocalTensor<T>dst1Local=queueDst1.DeQue<T>();AscendC:DataCopy(dstGlobal,dst1Local,size);AscendC:PipeBarrier<PIPE_MTE3>();AscendC:SetAtomicMax<T>();AscendC:DataCopy(dstGlobal,dst0Local,size);queueDst0.FreeTensor(dst0Local);queueDst1.FreeTensor(dst1Local);AscendC:SetAtomicNone();}private:AscendC:TPipepipe;AscendC:TQue<AscendC:TPosition:VECIN,1>queueSrc0;AscendC:TQue<AscendC:TPosition:VECIN,1>queueSrc1;AscendC:TQue<AscendC:TPosition:VECOUT,1>queueDst0;AscendC:TQue<AscendC:TPosition:VECOUT,1>queueDst1;AscendC:GlobalTensor<T>src0Global,src1Global,dstGlobal;uint32_tsize;};extern"C"__global____aicore__voiddata_copy_atomic_max_kernel(GM_ADDRsrc0_gm,GM_ADDRsrc1_gm,GM_ADDRdst_gm){KernelDataCopyAtomicMax<half>op;op.Init(src0_gm,src1_gm,dst_gm,data_size);op.Process();}每个核的输入数据为：Src0:[1,1,1,1,1,...,1]// 256个1Src1:[2,2,2,2,2,...,2]// 256个2最终输出数据：[2,2,2,2,2,...,2]// 256个2 |
| ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
