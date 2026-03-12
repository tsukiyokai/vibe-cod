# 尾核&尾块-多核&Tiling切分-矢量编程-SIMD算子实现-算子实践参考-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_10_10009
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_10_10009.html
---

# 尾核&尾块

对于不同shape的输入进行数据切分时，可能会发生数据无法平均分配到多个核、同时每个核内的数据无法均分的情况。参考核间均分场景下的尾块处理与核间不均分场景下的尾核处理的处理方式，将两者结合起来考虑整核的尾块、尾核的尾块的处理方式。

#### Tiling实现

由于本场景中核间、核内的数据均无法均分，在核间不均分场景下的尾核处理定义的Tiling结构体的基础上增加两个成员变量：

- formerLastTileLength：数据量多的核最后一个分块大小，即整核的尾块大小。计算时，先按尾核Tiling中提到的分核策略，切分数据量多的核。123456// shape需要对齐到的datablock,uint32_ttotalLengthAligned=((totalLength+ALIGN_NUM-1)/ALIGN_NUM)*ALIGN_NUM;// 计算整核数量uint32_tformerNum=(totalLengthAligned/ALIGN_NUM)%BLOCK_DIM;// 计算整核的数据量uint32_tformerLength=((totalLengthAligned/BLOCK_DIM+ALIGN_NUM-1)/ALIGN_NUM)*ALIGN_NUM;再按尾块Tiling中的切分策略，计算尾块长度。1234567891011uint32_tformerTileNum=formerLength/ALIGN_NUM/UB_BLOCK_NUM;if(formerTileNum==0){formerTileLength=0;formerLastTileLength=formerLength/ALIGN_NUM*ALIGN_NUM;}elseif((formerLength/ALIGN_NUM)%UB_BLOCK_NUM==0){formerTileLength=UB_BLOCK_NUM*ALIGN_NUM;lastTileLength=0;}else{formerTileLength=UB_BLOCK_NUM*ALIGN_NUM;formerLastTileLength=formerLength-formerTileNum*formerTileLength;}
- tailLastTileLength：数据量少的核最后一个分块大小，即尾核的尾块大小。计算时，先按尾核Tiling中提到的分核策略，切分数据量少的核。1234// 计算尾核数量uint32_ttailNum=BLOCK_DIM-formerNum;// 计算尾核的数据量uint32_ttailLength=(totalLengthAligned/BLOCK_DIM/ALIGN_NUM)*ALIGN_NUM;再按尾块Tiling中的切分策略，计算尾块长度。1234567891011uint32_ttailTileNum=tailLength/ALIGN_NUM/UB_BLOCK_NUM;if(tailTileNum==0){tailTileLength=0;tailLastTileLength=tailLength/ALIGN_NUM*ALIGN_NUM;}elseif((tailLength/ALIGN_NUM)%UB_BLOCK_NUM==0){tailTileLength=UB_BLOCK_NUM*ALIGN_NUM;tailLastTileLength=0;}else{tailTileLength=UB_BLOCK_NUM*ALIGN_NUM;tailLastTileLength=tailLength-tailTileNum*tailTileLength;}

#### 算子类实现

Kernel侧Init函数和Process函数的实现需将核间均分场景下的尾块处理与核间不均分场景下的尾核处理的实现结合起来。

Init函数中由于整核和尾核对应的tileLength和lastTileLength不同。因此需按照核间不均分场景下的尾核处理中提到的分别处理整核和尾核。后续对主块和尾块的CopyIn、Compute、CopyOut函数的处理方式与核间均分场景下的处理方式相同。

Init函数实现代码如下：

| 1234567891011121314151617181920212223242526272829303132 | __aicore__inlinevoidInit(GM_ADDRx,GM_ADDRy,GM_ADDRz,AddCustomTilingDatatiling){if(AscendC:GetBlockIdx()<formerNum){this->tileNum=tiling.formerTileNum;this->tileLength=tiling.formerTileLength;this->lastTileLength=tiling.formerLastTileLength;xGm.SetGlobalBuffer((__gm__half*)x+tiling.formerLength*AscendC:GetBlockIdx(),tiling.formerLength);yGm.SetGlobalBuffer((__gm__half*)y+tiling.formerLength*AscendC:GetBlockIdx(),tiling.formerLength);zGm.SetGlobalBuffer((__gm__half*)z+tiling.formerLength*AscendC:GetBlockIdx(),tiling.formerLength);}else{this->tileNum=tiling.tailTileNum;this->tileLength=tiling.tailTileLength;this->lastTileLength=tiling.tailLastTileLength;xGm.SetGlobalBuffer((__gm__half*)x+tiling.formerLength*tiling.formerNum+tiling.tailLength*(AscendC:GetBlockIdx()-tiling.formerNum),tiling.tailLength);yGm.SetGlobalBuffer((__gm__half*)y+tiling.formerLength*tiling.formerNum+tiling.tailLength*(AscendC:GetBlockIdx()-tiling.formerNum),tiling.tailLength);zGm.SetGlobalBuffer((__gm__half*)z+tiling.formerLength*tiling.formerNum+tiling.tailLength*(AscendC:GetBlockIdx()-tiling.formerNum),tiling.tailLength);}// 只有尾块的场景下，tileLength为0，因此取tileLength和lastTileLength的最大值来初始化uint32_tinitBufferLength=AscendC:Std:max(this->tileLength,this->lastTileLength);pipe.InitBuffer(inQueueX,1,initBufferLength*sizeof(half));pipe.InitBuffer(inQueueY,1,initBufferLength*sizeof(half));pipe.InitBuffer(outQueueZ,1,initBufferLength*sizeof(half));} |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
