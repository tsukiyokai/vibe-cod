# 算子与高阶API共享临时Buffer-内存访问-SIMD算子性能优化-算子实践参考-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_best_practices_10_0017
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_best_practices_10_0017.html
---

# 算子与高阶API共享临时Buffer

【优先级】高

【描述】如果算子使用的高阶API需要传入临时Buffer，如SoftMax，该临时空间会挤占算子其他计算的空间，从而导致单次计算搬运的数据量变少，搬运的次数变多。此场景可通过共享临时Buffer空间，提升单次搬运的数据量，减少搬运的次数，提升内存使用效率。

【反例】

| 12345678910111213141516171819 | ...constexprint32_tblockLen=32*1024;TBuf<TPosition:VECCALC>tmpSoftmaxBuf;pipe.InitBuffer(tmpSoftmaxBuf,softmaxBufSize*sizeof(uint8_t));// 单独分配Softmax的临时Buf 32KBTBuf<TPosition:VECCALC>tmpSumBuf;pipe.InitBuffer(tmpSumBuf,sumBufSize*sizeof(T));// 单独分配Add的临时Buf，且softmaxBufSize * sizeof(uint8_t) + sumBufSize * sizeof(T) <= 64KB...for(inti=0;i<16;i++){...LocalTensor<uint8_t>tmpSoftmaxTensor=tmpSoftmaxBuf.Get<uint8_t>(softmaxBufSize);SoftMax<T,true,true>(dstTensor,expSumTensor,dstMaxTensor,srcTensor,tmpSoftmaxTensor,tiling);...DataCopy(src0Tensor,src0Gm[i*blockLen/sizeof(T)],Params);...LocalTensor<T>tmpSumTensor=tmpSumBuf.Get<T>(sumBufSize);Add<T>(tmpSumTensor,src0Tensor,src1Tensor,count);...}... |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

【正例】

SoftMax高阶API计算需要临时Buffer空间，算子在进行其他计算时可以共享此临时Buffer，按照上述假设只需要搬运512 / 64 =8次。

| 1234567891011121314151617 | ...constexprint32_tblockLen=64*1024;TBuf<TPosition:VECCALC>tmpSharedBuf;pipe.InitBuffer(tmpSharedBuf,bufferSize);// 共享分配bufferSize = MAX(softmaxBufSize * sizeof(uint8_t), sumBufSize * sizeof(T)) <= 64KB...for(inti=0;i<8;i++){...LocalTensor<uint8_t>tmpSharedTensor=tmpSharedBuf.Get<uint8_t>(softmaxBufSize);SoftMax<T,true,true>(dstTensor,expSumTensor,dstMaxTensor,srcTensor,tmpSharedTensor,tiling);...DataCopy(src0Tensor,src0Gm[i*blockLen/sizeof(T)],Params);...LocalTensor<T>tmpSumTensor=tmpSharedBuf.Get<T>(sumBufSize);Add<T>(tmpSumTensor,src0Tensor,src1Tensor,count);...}... |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
