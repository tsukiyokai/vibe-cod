# Get-TBuf-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0163
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0163.html
---

# Get

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

从TBuf上获取指定长度的Tensor，或者获取全部长度的Tensor。

#### 函数原型

- 获取全部长度的Tensor12template<typenameT>__aicore__inlineLocalTensor<T>Get()
- 获取指定长度的Tensor12template<typenameT>__aicore__inlineLocalTensor<T>Get(uint32_tlen)

#### 参数说明

| 参数名称 | 含义                     |
| -------- | ------------------------ |
| T        | 待获取Tensor的数据类型。 |

| 参数名称 | 输入/输出 | 含义                       |
| -------- | --------- | -------------------------- |
| len      | 输入      | 需要获取的Tensor元素个数。 |

#### 约束说明

len的数值是Tensor中元素的个数，len*sizeof(T)不能超过TBuf初始化时的长度。

#### 返回值说明

获取到的LocalTensor。

#### 调用示例

| 123456789 | // 为TBuf初始化分配内存，分配内存长度为1024字节AscendC:TPipepipe;AscendC:TBuf<AscendC:TPosition:VECCALC>calcBuf;// 模板参数为TPosition中的VECCALC类型uint32_tbyteLen=1024;pipe.InitBuffer(calcBuf,byteLen);// 从calcBuf获取Tensor,Tensor为pipe分配的所有内存大小，为1024字节AscendC:LocalTensor<int32_t>tempTensor1=calcBuf.Get<int32_t>();// 从calcBuf获取Tensor,Tensor为128个int32_t类型元素的内存大小，为512字节AscendC:LocalTensor<int32_t>tempTensor2=calcBuf.Get<int32_t>(128); |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

对同一个TBuf对象连续调用Get接口，获取到的Tensor首地址是相同的，不会依次向后偏移。如果需要获取偏移之后的Tensor，可以使用如下方法：
