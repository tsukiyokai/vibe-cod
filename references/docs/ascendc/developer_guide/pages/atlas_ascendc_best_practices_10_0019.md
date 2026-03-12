# 通过缩减Tensor ShapeInfo维度，优化栈空间-内存访问-SIMD算子性能优化-算子实践参考-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_best_practices_10_0019
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_best_practices_10_0019.html
---

# 通过缩减Tensor ShapeInfo维度，优化栈空间

【优先级】中

【描述】GlobalTensor和LocalTensor中通过ShapeInfo类型的成员变量来保存shape信息，SetShapeInfo/GetShapeInfo可以设置或者获取ShapeInfo，在算子实现内部用于shape信息保存和传递。默认情况下支持的最大维度为8。在不使用上述ShapeInfo功能的情况下，不需要这些信息，可以通过K_MAX_SHAPE_DIM宏将其设置为0。经实测减小K_MAX_SHAPE_DIM值，可缩减栈空间，减少scalar指令和cache miss几率，提升算子性能。

| 1234567891011121314151617181920212223 | ...#ifndef K_MAX_SHAPE_DIM#define K_MAX_SHAPE_DIM 8#endif...structShapeInfo{public:...uint32_tshape[K_MAX_SHAPE_DIM];uint32_toriginalShape[K_MAX_SHAPE_DIM];};template<typenameT>classGlobalTensor{....private:ShapeInfoshapeInfo_;}template<typenameT>classLocalTensor{....private:ShapeInfoshapeInfo_;}... |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

【反例】

| 123456789101112 | ...#include"kernel_operator.h"...extern"C"__global____aicore__voidadd_custom(GM_ADDRx,GM_ADDRx,GM_ADDRz,GM_ADDRworkspace,GM_ADDRtiling){...GlobalTensor<T>dataIn;GlobalTensor<T>dataOut;LocalTensor<T>vecIn;LocalTensor<T>vecOut;...}... |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

【正例】

| 1234567891011121314 | #define K_MAX_SHAPE_DIM 0...#include"kernel_operator.h"//需注意定义K_MAX_SHAPE_DIM宏的位置须在包含Ascend C相关头文件之前...extern"C"__global____aicore__voidadd_custom(GM_ADDRx,GM_ADDRx,GM_ADDRz,GM_ADDRworkspace,GM_ADDRtiling){...GlobalTensor<T>dataIn;GlobalTensor<T>dataOut;LocalTensor<T>vecIn;LocalTensor<T>vecOut;...}... |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
