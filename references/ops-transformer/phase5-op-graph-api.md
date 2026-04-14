# Phase 5: op_graph & op_api Layer Analysis

## 5.1 op_graph层: gen_task + fallback

### 5.1.1 gen_task文件结构

每个算子的op_graph/包含两个文件:
1. `OpName_gen_task.cpp` -- 任务生成(CalcOpParam + GenerateTask)
2. `fallback_OpName.cpp` -- 降级执行(调用aclnn API)

#### gen_task标准模式

所有26个gen_task文件遵循完全一致的结构:

```
#ifdef BUILD_OPEN_PROJECT
  // 开源版(ops-transformer仓库)
  #include "mc2_gen_task_ops_utils.h"
  #include "register/op_impl_registry.h"

  ge::Status CalcParamFunc(ExeResGenerationContext *context) {
      // arch分派: A5用"ccu server"/"ccu_stream", 其他用"aicpu kfc server"/"kfc_stream"
      return CommonKFCMc2CalcParamFunc(context, serverName, reuseKey);
  }
  ge::Status GenTaskFunc(const ExeResGenerationContext *context, tasks) {
      // arch分派: A5用CCU回调, 其他用算子特有回调
      return callback(context, tasks);
  }
  IMPL_OP(OpName).CalcOpParam(CalcParamFunc).GenerateTask(GenTaskFunc);
#else
  // 内部版(canndev仓库)
  #include "mc2_gen_task_utils.h"
  #include "register/op_ct_impl_registry.h"

  // 类似结构，使用IMPL_OP_CT替代IMPL_OP
  IMPL_OP_CT(OpName).CalcOpParam(CalcParamFunc).GenerateTask(GenTaskFunc);
#endif
```

量化统计:
- IMPL_OP: 26处/26文件 (开源版注册)
- IMPL_OP_CT: 24处/24文件 (内部版注册)
- CommonKFCMc2CalcParamFunc: 72处/27文件 (100%覆盖)
- BUILD_OPEN_PROJECT: 61处/27文件 (100%双版本)

#### arch分派模式(gen_task中)

matmul_all_reduce (支持A5):
```
if (IsTargetPlatformNpuArch(nodeName, NPUARCH_A5)) {
    // A5: CCU server + CCU GenTask回调
    CalcParamFunc: "ccu server" / "ccu_stream"
    GenTaskFunc: Mc2Arch35GenTaskOpsUtils::Mc2Arch35GenTaskCallBack
} else {
    // 非A5: AICPU server + 算子特有回调
    CalcParamFunc: "aicpu kfc server" / "kfc_stream"
    GenTaskFunc: MatmulAllReduceGenTaskOpsUtils::MatmulAllReduceGenTaskCallback
}
```

all_gather_matmul (不支持A5):
```
// 无条件: 固定AICPU server
CalcParamFunc: "aicpu kfc server" / "kfc_stream"
GenTaskFunc: CommonKFCMc2GenTask (无算子特有回调)
```

moe_distribute_combine (按SoC版本):
```
if (IsTargetPlatformSocVersion(nodeName, PLATFORM_A2)) {
    // A2: MoE V1回调
    GenTaskFunc: Mc2MoeGenTaskCallback
} else {
    // A3/A5: MoE V2回调
    GenTaskFunc: Mc2MoeGenTaskCallbackV2
}
```

#### REGISTER_EXT_TASK_TYPE

11处/11文件: MoE系列 + quant系列 + distribute_barrier + attention_to_ffn + ffn_to_attention
仅在#else(内部版)中出现，注册扩展任务类型(kAicoreTask)

### 5.1.2 fallback文件结构

fallback文件实现"降级到aclnn API"的逻辑，当算子无法在NPU上执行时回退。

标准模式:
```
namespace fallback {
  // 1. 参数提取: 从OpExecuteContext获取Tensor和Attrs
  // 2. 参数转换: ConvertMmType等
  // 3. 按量化类型分派: switch (quantType) 调用不同aclnn API
  // 4. 注册: IMPL_OP(OpName).OpExecuteFunc(ExecuteFunc);
}
```

EXEC_OPAPI_CMD: 36处/23文件 -- fallback文件的核心调用宏
- 封装了aclnn API的完整调用流程(workspace分配 + launch)

matmul_all_reduce的fallback复杂度最高(245行):
- 区分3种量化类型(NONE/WEIGHT/QUANT)
- 调用6种不同的aclnn API (v1/v2/v3/v4 + weight_quant)
- 使用枚举索引MC2InputIdx/MC2OutputIdx/MmAllReduceAttrIdx

all_gather_matmul无独立fallback文件(由all_gather_matmul_v1版处理)

### 5.1.3 op_graph层规范

强制规范:
1. 每个算子必须同时有gen_task.cpp和fallback.cpp两个文件
2. gen_task.cpp必须有BUILD_OPEN_PROJECT双版本(IMPL_OP + IMPL_OP_CT)
3. CalcOpParam固定调用CommonKFCMc2CalcParamFunc
4. GenerateTask通过CommonKFCMc2GenTask + 算子特有回调实现
5. fallback通过EXEC_OPAPI_CMD调用aclnn API

推荐规范:
1. arch35算子在CalcParam中使用"ccu server"/"ccu_stream"
2. MoE算子按SoC版本(PLATFORM_A2 vs A3/A5)选择不同GenTask回调
3. MatMul类算子按NpuArch(NPUARCH_A5)选择CCU vs AICPU路径

## 5.2 op_api层: aclnn外部接口

### 5.2.1 API命名规范

标准格式: `aclnnOpNameGetWorkspaceSize` + `aclnnOpName`

两阶段API模式:
1. GetWorkspaceSize阶段: 参数校验 + workspace计算 + executor创建
2. Execute阶段: 实际launch kernel

统计: 110处GetWorkspaceSize/32文件(op_api), 377处NnopbaseSetHcclServerType/88文件(含tests)

### 5.2.2 aclnn API标准结构

以aclnn_matmul_all_reduce.cpp为例:

```
#ifdef __cplusplus
extern "C" {
#endif

// 1. 声明Inner版本(由框架生成)
extern aclnnStatus aclnnInnerOpNameGetWorkspaceSize(...);
extern aclnnStatus aclnnInnerOpName(...);
// 2. 声明Nnopbase工具函数(weak符号)
extern "C" void __attribute__((weak)) NnopbaseSetHcclServerType(void*, NnopbaseHcclServerType);
extern "C" aclnnStatus __attribute__((weak)) NnopbaseDisableOptionalInput(void*, const size_t);

// 3. GetWorkspaceSize: 参数校验 → 调Inner → DisableOptionalInput → 返回
aclnnStatus aclnnOpNameGetWorkspaceSize(...) {
    uint64_t timeStamp = NnopbaseMsprofSysTime();  // DFX打点
    auto retParam = CheckParams(...);               // 参数校验
    CHECK_RET(retParam == ACLNN_SUCCESS, retParam);

    aclnnStatus ret = aclnnInnerOpNameGetWorkspaceSize(...);  // 调用Inner

    if (ret == 0 && NnopbaseDisableOptionalInput != nullptr) {
        NnopbaseDisableOptionalInput(*executor, irIndex);    // 禁用可选输入
    }

    NnopbaseReportApiInfo(timeStamp, dfxId);        // DFX上报
    return ret;
}

// 4. Execute: SetHcclServerType → 调Inner → 返回
aclnnStatus aclnnOpName(...) {
    if (NnopbaseSetHcclServerType) {
        if (DAV_3510) SetHcclServerType(CCU);       // arch35: CCU
    }
    aclnnStatus ret = aclnnInnerOpName(...);
    return ret;
}

#ifdef __cplusplus
}
#endif
```

### 5.2.3 参数校验模式

校验层次:
1. 空指针检查: OP_CHECK_NULL(x1, return false)
2. dtype校验: OP_CHECK_DTYPE_NOT_SUPPORT + OP_CHECK_DTYPE_NOT_SAME
3. shape校验: OP_CHECK_WRONG_DIMENSION + 自定义shape约束
4. 属性校验: streamMode == 1 (STOP_ON_FAILURE)

all_gather_matmul的校验最详细(CheckShape 40行):
- 维度必须2D
- x1不支持transpose
- K轴一致性(x1.dim1 == x2.dim0)
- K轴范围: [256, 65535)
- N轴一致性(x2.dim1 == output.dim1)
- gatherOut的K轴一致性

### 5.2.4 v1→v2重定向模式

aclnn_all_gather_matmul.cpp展示了关键模式:
```
if (IsAscend910A5()) {
    // v1 API自动重定向到v2 API(支持arch35)
    return aclnnAllGatherMatmulV2GetWorkspaceSize(...);
}
// 非A5: 调用旧版Inner
return aclnnInnerAllGatherMatmulGetWorkspaceSize(...);
```

Execute阶段同理:
```
if (IsAscend910A5()) {
    return aclnnAllGatherMatmulV2(workspace, workspaceSize, executor, stream);
}
```

这意味着: 旧版API(v1)在arch35上自动透传到v2实现，对用户透明。

NnopbaseDisableOptionalInput: 41处/8文件
- 仅matmul_all_reduce和matmul_reduce_scatter使用
- 禁用不需要的可选输入(如非量化模式下禁用scale/offset/dequant等)
- 通过irIndex逐个禁用(magic number: 3,4,5,6,7,8,9)

### 5.2.5 NnopbaseSetHcclServerType

在Execute阶段设置HCCL Server类型:
```
if (NnopbaseSetHcclServerType) {
    if (GetCurrentPlatformInfo().GetCurNpuArch() == NpuArch::DAV_3510) {
        NnopbaseSetHcclServerType(executor, NNOPBASE_HCCL_SERVER_TYPE_CCU);
    }
}
```

统计: 出现在matmul_all_reduce系列的op_api中(4文件)。
其他算子(如all_gather_matmul_v2)直接在v2 API中硬编码CCU。

### 5.2.6 op_api层规范

强制规范:
1. 两阶段API: GetWorkspaceSize + Execute(每个aclnn API必须有这两个函数)
2. `extern "C"`包裹: 所有aclnn函数必须C链接
3. DFX打点: NnopbaseMsprofSysTime() + NnopbaseReportApiInfo() (性能监控)
4. 参数校验在GetWorkspaceSize中完成(不在Execute中重复)
5. 调用Inner版本(aclnnInnerXxx): 实际workspace计算和kernel准备由框架完成

推荐规范:
1. arch35在Execute中设置NnopbaseSetHcclServerType为CCU
2. v1 API在arch35上自动重定向到v2
3. 使用NnopbaseDisableOptionalInput禁用不需要的可选输入(减少无效数据传输)
4. 空tensor处理: DealWithX1Empty返回空executor(workspaceSize=0)

反模式:
1. NnopbaseDisableOptionalInput使用magic number(irIndex=3,4,5...)
   - 正确做法: 使用枚举常量(如MC2InputIdx::K_X3)
2. weak符号声明分散在每个op_api文件中
   - 正确做法: 提取到公共头文件(hccl_util.h已部分实现)

## 5.3 跨算子一致性

### op_graph层

一致性非常高:
- 100%使用IMPL_OP/IMPL_OP_CT注册
- 100%使用CommonKFCMc2CalcParamFunc
- 100%有BUILD_OPEN_PROJECT双版本
- gen_task.cpp结构高度统一，差异仅在arch分派逻辑

### op_api层

一致性中等:
- 100%遵循两阶段API模式(GetWorkspaceSize + Execute)
- 100%有DFX打点
- 参数校验覆盖率不一致: matmul_all_reduce最完整，小型算子可能缺少shape校验
- v1→v2重定向仅在all_gather_matmul和matmul_reduce_scatter中出现(2个算子)
- NnopbaseDisableOptionalInput仅在matmul_all_reduce系列使用(8个文件)

### 四层职责总结

| 层 | 职责 | 注册宏 | 关键函数 |
|---|---|---|---|
| op_host | Tiling计算 + Shape推导 + Op定义 | IMPL_OP_OPTILING + IMPL_OP_INFERSHAPE + OP_ADD | DoTiling/InferShape |
| op_kernel | 设备端kernel执行 | REGISTER_TILING_DEFAULT | __global__ __aicore__ void |
| op_graph | 任务生成 + Fallback降级 | IMPL_OP(.CalcOpParam.GenerateTask) | CalcParam/GenTask/ExecuteFunc |
| op_api | 外部C API接口 | extern "C" | aclnnXxxGetWorkspaceSize/aclnnXxx |

## 5.4 Phase 5总结

### 强制规范

1. gen_task双版本: BUILD_OPEN_PROJECT切换IMPL_OP/IMPL_OP_CT
   - 26处/26文件, 100%覆盖

2. CalcOpParam固定模式: CommonKFCMc2CalcParamFunc(context, serverName, reuseKey)
   - serverName: "aicpu kfc server"(非A5) 或 "ccu server"(A5)
   - reuseKey: "kfc_stream" 或 "ccu_stream"
   - 72处/27文件

3. 两阶段API: 每个aclnn接口必须实现GetWorkspaceSize + Execute
   - 110处GetWorkspaceSize/32文件

4. DFX打点: NnopbaseMsprofSysTime + NnopbaseReportApiInfo
   - 每个aclnn函数的开头和结尾

5. fallback通过EXEC_OPAPI_CMD调用aclnn API
   - 36处/23文件

### 推荐规范

1. arch35在CalcParam中使用"ccu server"，在Execute中SetHcclServerType(CCU)
2. v1 API在arch35上重定向到v2(用户无感知升级)
3. op_api参数校验应完整覆盖: 空指针 → dtype → shape → 属性
4. 使用NnopbaseDisableOptionalInput减少无效数据传输

### 反模式

1. NnopbaseDisableOptionalInput的irIndex使用magic number
   - aclnn_matmul_all_reduce.cpp:75-81行
   - 正确做法: 定义枚举常量

2. weak符号声明重复
   - NnopbaseSetHcclServerType/NnopbaseDisableOptionalInput在每个op_api文件中重复声明
   - 正确做法: 统一在公共头文件中声明

3. MoE gen_task使用IsTargetPlatformSocVersion而非IsTargetPlatformNpuArch
   - moe_distribute_combine_gen_task.cpp:43行
   - 与MatMul类(使用NpuArch)不一致
   - 反映了Phase 1中发现的socVersion→npuArch迁移未完全完成

## 5.5 tests/和docs/模式

### 测试文件统计

UT测试按三层组织(无op_graph层测试):
- op_host UT: 61个.cpp文件
- op_api UT: 38个.cpp文件
- op_kernel UT: 22个.cpp文件

命名规范: `test_OpName.cpp` (op_kernel) / `test_aclnn_OpName.cpp` (op_api)

#### op_kernel UT模式

以matmul_all_reduce为例:
- 使用gtest框架(testing::Test)
- `__CCE_KT_TEST__`宏包裹测试专用include
- 直接`#include "../../../op_kernel/matmul_all_reduce.cpp"` -- 包含整个kernel源文件
- SetUpTestCase中: GmAlloc分配HcclContext模拟通信环境
- TearDownTestCase中: GmFree释放
- SetKernelMode(KernelMode::MIX_MODE)设置混合核模式

关键观察: kernel UT通过include .cpp文件来测试，不是独立编译链接。这意味着测试和生产代码编译在同一翻译单元中。

辅助文件:
- *_tiling_def.h: 测试用TilingData的定义和初始化
- data_utils.h: 测试数据生成工具
- rac_server_stub.h: RAC Server的打桩实现

#### op_api UT模式

命名: `test_aclnn_*.cpp`
测试aclnn两阶段接口的参数校验和返回值

### docs/文档组织

每个算子目录下有docs/子目录，内含:
- 每个aclnn API对应一个.md文件
- 命名: `aclnnOpName.md` (如aclnnMatmulAllReduce.md)

matmul_all_reduce有7个文档文件(对应7个API版本)。

文档结构标准化:
1. 产品支持情况表(哪些Ascend产品支持)
2. 功能说明(计算公式)
3. 函数原型(两段式接口声明)
4. 参数说明(表格形式)
5. 支持的数据类型
6. 约束与限制
7. 调用示例

### 5.5/5.6小结

测试规范:
- 每个算子应有op_host/op_api/op_kernel三层UT(当前22个kernel UT覆盖不完整)
- kernel UT通过include .cpp + gtest + GmAlloc模拟环境
- op_api UT测试两阶段接口的参数校验

文档规范:
- 每个aclnn API对应一个独立.md文件
- 文档结构标准化: 产品支持 → 功能 → 原型 → 参数 → dtype → 约束 → 示例
