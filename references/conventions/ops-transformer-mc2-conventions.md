# ops-transformer MC2 Module Coding Conventions

Source: /Users/shanshan/repo/cann/ops-transformer (mc2/ + common/)
Analysis: /Users/shanshan/note/proj/dig-repo-cod/ops-transformer/phase0~6
Date: 2026-03-15

---

## 1. Architecture Overview

### 1.1 Four-Layer Structure

MC2(MatMul-Communication)算子采用严格的四层架构，33个算子目录100%遵循:

```
OpName/
  op_host/    -- Tiling计算 + Shape推导 + Op定义 (host侧)
  op_kernel/  -- 设备端kernel执行 (device侧)
  op_graph/   -- 任务生成 + Fallback降级
  op_api/     -- 外部C API接口 (aclnn两阶段)
  docs/       -- API文档 (每个aclnn API一个.md)
  tests/      -- UT测试 (op_host/op_kernel/op_api三层)
  examples/   -- 使用示例
```

### 1.2 Algorithm Families

MC2算子按通信类型分为两大家族:

MatMul+Comm家族(AIC Cube计算 + HCCL通信):
- AllReduce类: matmul_all_reduce, quant_all_reduce, grouped_mat_mul_all_reduce
- AllGather类: all_gather_matmul, all_gather_matmul_v2
- ReduceScatter类: matmul_reduce_scatter, matmul_reduce_scatter_v2, quant_reduce_scatter
- AlltoAll类: matmul_allto_all, allto_all_matmul

MoE家族(AIV Vector计算 + RDMA窗口通信):
- moe_distribute_dispatch/v2/v3, moe_distribute_combine/v2/v3
- setup/teardown (通信生命周期管理)

### 1.3 Three-Level Common Infrastructure

1. ops-transformer/common/ -- 仓库级: TilingBaseClass, Fallback框架, 注册模板
2. mc2/common/ -- MC2模块级: TilingData结构, CcTiling通信配置, kernel工具
3. mc2/3rd/ -- 第三方MatMul库: mat_mul_v3, batch_mat_mul_v3, weight_quant_batch_matmul_v2

### 1.4 Arch Generations

| Arch | Product | AICore | NPU_ARCH | HCCL Server | MatMul Block |
|------|---------|--------|----------|-------------|--------------|
| arch31 | 310P | 200 | - | AICPU | BaseBlock |
| arch32 | 910B | 220 | - | AICPU | BaseBlock |
| arch35 | 950 | 310 | 3510 | CCU | AswBlock |

---

## 2. op_host Layer (Tiling & Shape Inference)

### 2.1 Op Registration (OP_ADD)

[Scope]: MC2 module
[Severity]: Mandatory

Op定义必须使用OP_ADD宏注册，位于op_host/目录的def.cpp中。

```cpp
// good
OP_ADD(MatmulAllReduce)
    .INPUT(x1, TensorType({DT_FLOAT16, DT_BF16}))
    .INPUT(x2, TensorType({DT_FLOAT16, DT_BF16}))
    .OPTIONAL_INPUT(bias, TensorType({DT_FLOAT16, DT_BF16}))
    .OUTPUT(y, TensorType({DT_FLOAT16, DT_BF16}))
    .ATTR(group, AttrValue::STR {"hccl_world_group"})
    .OP_END_FACTORY_REG(MatmulAllReduce);
```

Coverage: 35处/35文件 (100%)

### 2.2 InferShape Registration

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
// good
IMPL_OP_INFERSHAPE(MatmulAllReduce)
{
    // ... shape推导逻辑
    return GRAPH_SUCCESS;
}
```

Coverage: 33处/33文件 (100%)

### 2.3 Tiling Registration

[Scope]: MC2 module
[Severity]: Mandatory

每个算子必须用IMPL_OP_OPTILING注册tiling函数:

```cpp
IMPL_OP_OPTILING(MatmulAllReduce)
{
    // context参数中获取shape/attrs/platform信息
    // 计算tiling参数写入TilingData
    return GRAPH_SUCCESS;
}
```

Coverage: 37处/37文件 (100%)

### 2.4 Tiling Template Registration

[Scope]: MC2 module
[Severity]: Mandatory

三种注册方式(按使用场景选择):

```cpp
// arch35专用算子(NpuArch分派)
REGISTER_TILING_TEMPLATE_WITH_ARCH(MatmulAllReduce, MatmulAllReduceTiling, DAV_3510, priority);
// 13处

// arch31/32算子(SocVersion分派)
REGISTER_TILING_TEMPLATE_WITH_SOCVERSION(AllGatherMatmul, AllGatherMatmulTiling, "Ascend910B", priority);
// 11处

// MoE/GMM等自管arch分派的算子
REGISTER_OPS_TILING_TEMPLATE(MoeDistributeCombine, MoeDistributeCombineTiling, priority);
// 33处
```

Priority chain: 数值越小越优先，量化tiling通常低priority(优先匹配)。

### 2.5 Tiling Key (GET_TPL_TILING_KEY)

[Scope]: MC2 module
[Severity]: Mandatory

TilingKey是十进制编码的整数，贯穿op_host→op_kernel，驱动编译期多态:

```cpp
// op_host: 计算tiling key
context->SetTilingKey(GET_TPL_TILING_KEY(MatmulAllReduce, ...));

// op_kernel: tiling key解码为模板参数
ASCENDC_TPL_ARGS_DECL(MatmulAllReduce, AICORE_TYPE, MM_TYPE, ...);
```

Coverage: 99处/77文件, 36个tiling_key.h文件

### 2.6 SetScheduleMode(1)

[Scope]: MC2 module
[Severity]: Mandatory

MC2算子必须设置调度模式为1(AIC+AIV混合核):

```cpp
context->SetScheduleMode(1);
```

Coverage: 39处/31文件

### 2.7 Mc2CcTilingConfig (Communication Configuration)

[Scope]: MC2 module
[Severity]: Mandatory

所有MC2算子的tiling必须配置通信参数:

```cpp
Mc2CcTilingConfig ccTiling;
ccTiling.SetRankId(rankId);
ccTiling.SetRankDim(rankDim);
// ... 其他通信参数
```

Coverage: 103处/47文件

### 2.8 HcclGroup Declaration

[Scope]: MC2 module
[Severity]: Mandatory

Op定义中必须声明HCCL通信组:

```cpp
// 单通信域(MatMul类)
.ATTR(group, AttrValue::STR {"hccl_world_group"})
// 18处

// 双通信域(MoE类)
.ATTR(group_ep, AttrValue::STR {"hccl_world_group"})
.ATTR(group_tp, AttrValue::STR {"hccl_world_group"})
// 8处
```

Total: 38处

---

## 3. op_kernel Layer (Kernel Implementation)

### 3.1 Kernel Entry Signature

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
// good: workspaceGM和tilingGM固定为最后两个参数
template<TplParams...>
__global__ __aicore__ void OpName(GM_ADDR input1, ..., GM_ADDR workspaceGM, GM_ADDR tilingGM)
{
    REGISTER_TILING_DEFAULT(LargestTilingDataStruct);
    // ...
}
```

Coverage: 44个kernel入口.cpp文件 (100%)

### 3.2 REGISTER_TILING

[Scope]: MC2 module
[Severity]: Mandatory

每个kernel入口函数开头必须注册TilingData:

```cpp
// 最少: 只有一种TilingData
REGISTER_TILING_DEFAULT(TilingDataStruct);

// 多种TilingData时: DEFAULT注册最大的，FOR_TILINGKEY注册条件匹配的
REGISTER_TILING_DEFAULT(WeightQuantMatmulAllReduceTilingData); // 最大
REGISTER_TILING_FOR_TILINGKEY("MM_TYPE == MMTYPE_FP_MM", MatmulAllReduceTilingData);
REGISTER_TILING_FOR_TILINGKEY("MM_TYPE == MMTYPE_QUANT_MM", QuantMatmulAllReduceTilingData);
```

REGISTER_TILING_DEFAULT: 59处/44文件 (100%)
REGISTER_TILING_FOR_TILINGKEY: 26处/10文件

### 3.3 if constexpr for Template Dispatch

[Scope]: MC2 module
[Severity]: Mandatory

模板参数的所有运行时分支必须用if constexpr:

```cpp
// good
if constexpr (MM_TYPE == MMTYPE_FP_MM) {
    fp_matmul_all_reduce<...>(...);
} else if constexpr (MM_TYPE == MMTYPE_QUANT_MM) {
    quant_matmul_allreduce<...>(...);
}

// bad: 编译报错(NPU不支持运行时模板分支)
if (MM_TYPE == MMTYPE_FP_MM) {
    fp_matmul_all_reduce<...>(...);
}
```

### 3.4 AIC/AIV Core Division

[Scope]: MC2 module
[Severity]: Mandatory

AIC核做Cube计算(MatMul)，AIV核做通信+数据搬运+向量运算:

```cpp
if ASCEND_IS_AIV {
    commStage_->Process(index);   // 通信
    SyncAll<true>();
    transStage_->Process(index);  // 数据搬运
}
if ASCEND_IS_AIC {
    CrossCoreWaitFlag(9);
    computeStage_->Process(index); // MatMul
}
```

ASCEND_IS_AIC/AIV: 399处/126文件

### 3.5 GetHcclContext

[Scope]: MC2 module
[Severity]: Mandatory

获取通信上下文:

```cpp
auto context = GetHcclContext<0>();  // GroupId 0
// MoE双域:
auto epContext = GetHcclContext<HCCL_GROUP_ID_0>();
auto tpContext = GetHcclContext<HCCL_GROUP_ID_1>();
```

Coverage: 93处/60文件 (100%)

### 3.6 Workspace Null Check

[Scope]: MC2 module (APT entries)
[Severity]: Recommended

```cpp
// good: APT入口中的标准检查
if (workspaceGM == nullptr) {
    return;
}
GM_ADDR userWS = GetUserWorkspace(workspaceGM);
if (userWS == nullptr) {
    return;
}
```

GetUserWorkspace: 22处/22文件

### 3.7 Arch35 CCU Server

[Scope]: arch35 operators
[Severity]: Mandatory (for arch35)

```cpp
// arch32: AICPU代理通信
Hccl<HCCL_SERVER_TYPE_AICPU> hccl_;
hccl_.Init(GetHcclContext<0>());

// arch35: CCU直驱通信
Hccl<HcclServerType::HCCL_SERVER_TYPE_CCU> hccl_;
hccl_.InitV2(GetHcclContext<0>(), tilingData_);
hccl_.SetCcTilingV2(offsetof(MC2TilingHeader, mc2CcTiling));
```

AICPU: 64处/55文件; CCU: 63处/50文件; InitV2: 35处/16文件

### 3.8 APT Tiling Key (arch35)

[Scope]: arch35 operators
[Severity]: Recommended

APT版tiling key面向业务语义:

```cpp
// APT (新): 业务导向参数
ASCENDC_TPL_ARGS_DECL(Mc2MatmulAllReduceApt,
    APT_MM_TYPE, TPL_TRANS_B, TPL_EXIST_BIAS, MATMUL_WITH_ADD,
    KERNEL_TYPE, COMMDTPYE, SCENARIO_MXFP8, ...);

// Old (旧): 实现导向参数
ASCENDC_TPL_ARGS_DECL(MatmulAllReduce,
    AICORE_TYPE, MM_TYPE, EMPTY_INPUT, INT8_COMM,
    ENABLE_L2_CACHE, SHARE_MM, MIXND2NZ, ...);
```

APT入口文件: 6个 (*_apt.cpp)
APT tiling key: 3个 (*_apt_tiling_key.h)

### 3.9 Mc2SyncAll Template

[Scope]: MatMul+Comm operators
[Severity]: Recommended

```cpp
// good: 封装同步，按核类型自动选择
template <Mc2CoreType coreType>
__aicore__ inline void Mc2SyncAll() {
    if constexpr (coreType == ON_CUBE_AND_VECTOR) SyncAll<false>();
    else if constexpr (coreType == ON_VECTOR) SyncAll();
    else { PipeBarrier<PIPE_ALL>(); CrossCore... }
}

// bad: 裸调SyncAll可能遗漏核类型判断
SyncAll();  // 混合核场景下行为不确定
```

Coverage: 12处/6文件

### 3.10 Conditional Compilation Pattern

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
// __CCE_AICORE__: 仅在kernel入口.cpp中使用，顶层arch分派
#if __CCE_AICORE__ == 220   // arch32 (910B)
#elif __CCE_AICORE__ == 200  // arch31 (310P)
#elif __CCE_AICORE__ == 310  // arch35 (950)
#endif
// 21处/8文件

// __NPU_ARCH__: 在.h实现文件中使用，细粒度条件编译
#if defined(__NPU_ARCH__) && (__NPU_ARCH__ == 3510)
// arch35专用代码
#endif
// 94处/34文件
```

---

## 4. op_graph Layer (Task Generation & Fallback)

### 4.1 gen_task Dual Version

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
#ifdef BUILD_OPEN_PROJECT
    // 开源版(ops-transformer仓库)
    IMPL_OP(OpName).CalcOpParam(CalcFunc).GenerateTask(GenFunc);
#else
    // 内部版(canndev仓库)
    IMPL_OP_CT(OpName).CalcOpParam(CalcFunc).GenerateTask(GenFunc);
#endif
```

Coverage: IMPL_OP 26处/26文件, IMPL_OP_CT 24处/24文件

### 4.2 CalcOpParam Standard Pattern

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
ge::Status CalcParamFunc(gert::ExeResGenerationContext *context) {
    if (IsTargetPlatformNpuArch(context->GetNodeName(), NPUARCH_A5)) {
        return CommonKFCMc2CalcParamFunc(context, "ccu server", "ccu_stream");
    }
    return CommonKFCMc2CalcParamFunc(context, "aicpu kfc server", "kfc_stream");
}
```

CommonKFCMc2CalcParamFunc: 72处/27文件 (100%)

### 4.3 Fallback

[Scope]: MC2 module
[Severity]: Mandatory

每个算子必须在op_graph/中提供fallback实现，通过IMPL_OP(OpName).OpExecuteFunc()注册。

两种实现方式:

```cpp
// 方式1(标准): 使用EXEC_OPAPI_CMD宏编排 (36处/23文件)
namespace fallback {
    ge::graphStatus ExecuteFunc(gert::OpExecuteContext* ctx) {
        // 提取参数 → 按量化类型分派
        return EXEC_OPAPI_CMD(aclnnXxx, param1, param2, ...);
    }
    IMPL_OP(OpName).OpExecuteFunc(ExecuteFunc);
}

// 方式2(自实现): 参数较复杂的算子自行解析和调用aclnn API
namespace fallback {
    ge::graphStatus ExecuteFunc(gert::OpExecuteContext* ctx) {
        // 手动提取tensor/attr → 参数校验 → 调用aclnn API
    }
    IMPL_OP(OpName).OpExecuteFunc(ExecuteFunc);
}
```

IMPL_OP(OpName).OpExecuteFunc: 23处/23文件 (100%)

---

## 5. op_api Layer (External C API)

### 5.1 Two-Phase API

[Scope]: MC2 module
[Severity]: Mandatory

```cpp
extern "C" {
// Phase 1: 参数校验 + workspace计算
aclnnStatus aclnnOpNameGetWorkspaceSize(
    const aclTensor *x1, ..., uint64_t *workspaceSize, aclOpExecutor **executor) {
    uint64_t timeStamp = NnopbaseMsprofSysTime();  // DFX start
    auto ret = CheckParams(...);                     // 校验
    CHECK_RET(ret == ACLNN_SUCCESS, ret);
    ret = aclnnInnerOpNameGetWorkspaceSize(...);      // 调Inner
    NnopbaseReportApiInfo(timeStamp, dfxId);          // DFX end
    return ret;
}

// Phase 2: 执行kernel
aclnnStatus aclnnOpName(void *workspace, uint64_t workspaceSize,
                        aclOpExecutor *executor, const aclrtStream stream) {
    if (NnopbaseSetHcclServerType && DAV_3510) {
        NnopbaseSetHcclServerType(executor, CCU);     // arch35: CCU
    }
    return aclnnInnerOpName(workspace, workspaceSize, executor, stream);
}
}
```

Coverage: 110处GetWorkspaceSize/32文件

### 5.2 DFX Instrumentation

[Scope]: MC2 module
[Severity]: Mandatory

每个aclnn函数必须有DFX打点:

```cpp
uint64_t timeStamp = NnopbaseMsprofSysTime();
// ... 业务逻辑 ...
static NnopbaseDfxId dfxId = {0x60000, __func__, false};
NnopbaseReportApiInfo(timeStamp, dfxId);
```

### 5.3 v1 to v2 Redirect (arch35)

[Scope]: v1 operators with v2 counterpart
[Severity]: Recommended

```cpp
// good: v1 API在arch35自动透传到v2
if (IsAscend910A5()) {
    return aclnnOpNameV2GetWorkspaceSize(...);
}
return aclnnInnerOpNameGetWorkspaceSize(...);
```

出现: all_gather_matmul, matmul_reduce_scatter (2个算子)

---

## 6. Cross-Layer Patterns

### 6.1 Naming Conventions

| Element | Style | Example |
|---------|-------|---------|
| 文件名 | snake_case | matmul_all_reduce_tiling.cpp |
| 类名 | PascalCase | MatmulAllReduceBase |
| 宏名 | UPPER_SNAKE | REGISTER_TILING_DEFAULT |
| aclnn函数 | camelCase(带前缀) | aclnnMatmulAllReduce |
| namespace | lowercase | ops, fallback, AscendC |
| 模板参数 | UPPER_SNAKE | HCCL_SERVER_TYPE_CCU |
| 条件编译宏 | __双下划线__ | __CCE_AICORE__, __NPU_ARCH__ |

### 6.2 Error Handling

op_host: 返回ge::GRAPH_SUCCESS/ge::GRAPH_FAILED
op_kernel: 无显式错误返回(空指针检查后直接return)
op_graph: 返回ge::Status
op_api: 返回aclnnStatus (ACLNN_SUCCESS/ACLNN_ERR_INNER/ACLNN_ERR_PARAM_INVALID)
fallback: 返回ge::graphStatus

### 6.3 TilingData Dual Definition

MC2的TilingData结构体在op_host和op_kernel两处定义:
- op_host: mc2/common/inc/mc2_tiling_struct.h (标注@deprecated)
- op_kernel: mc2/common/inc/kernel/mc2_tiling_struct.h

两份字段必须完全一致。op_host版用于host侧填充，op_kernel版用于device侧读取。

---

## 7. Anti-Patterns

### 7.1 Cross-Version Include

[Severity]: Error

```cpp
// bad: 跨版本目录include
#include "../moe_distribute_combine_v2/moe_distribute_combine_a2.h"

// good: 提取公共代码到common/或建立明确复用关系
#include "mc2/common/inc/kernel/shared_impl.h"
```

Source: moe_distribute_combine.cpp:24-27

### 7.2 CeilDiv Zero-Division Inconsistency

[Severity]: Error

至少5处CeilDiv定义，b==0时行为不一致(返回T_MAX/0/a)。

```cpp
// 不同文件中的不同实现:
T CeilDiv(T a, T b) { return (a + b - 1) / b; }      // b==0: 除零崩溃
T CeilDiv(T a, T b) { return b == 0 ? a : ...; }       // b==0: 返回a
T CeilDiv(T a, T b) { return b == 0 ? T_MAX : ...; }   // b==0: 返回最大值
```

### 7.3 Magic Number Attribute Index

[Severity]: Warning

```cpp
// bad: magic number索引
NnopbaseDisableOptionalInput(*executor, 3U); // 3 is input irIndex
NnopbaseDisableOptionalInput(*executor, 4U); // 4 is input irIndex

// good: 使用枚举
NnopbaseDisableOptionalInput(*executor, static_cast<size_t>(MC2InputIdx::K_X3));
```

Source: aclnn_matmul_all_reduce.cpp:75-81

### 7.4 Cross-Operator Constant/Utility Copy-Paste

[Severity]: Warning

多类代码在算子间copy-paste:
- Mc2SyncAll/Mc2CoreType: matmul_all_reduce/common.h和all_gather_matmul_v2/common.h定义相同
- GROUP_MNK_BIT_SIZE/GROUP_M_OFFSET等: alltoall_mx_quant和mx_quant_matmul_alltoall两处重复定义
- CeilDiv: 5+处定义且b==0行为不一致(见7.2)

```
// bad: 各算子copy自己的版本
matmul_all_reduce/op_kernel/common.h  -- 定义Mc2SyncAll
allto_all_mx_quant_matmul_tiling_base.h  -- 定义GROUP_MNK_BIT_SIZE

// good: 提取到mc2/common/
mc2/common/inc/kernel/mc2_sync_utils.h
mc2/common/inc/mc2_group_utils.h
```

Git evidence: 8709a41a (GROUP_MNK_BIT_SIZE在两个算子tiling_base中重复定义)

### 7.5 Tiling Function Bloat

[Severity]: Warning

Tiling函数膨胀到数百行后被拆分(有git证据: Cmetric驱动的split重构)。

经验法则: 单个tiling函数超过200行应考虑拆分为子函数。

Git evidence: multiple "split tiling" commits in mc2/matmul_all_reduce/op_host/

### 7.6 Hasty Sync Optimization

[Severity]: Critical

kernel同步机制的优化变更需要全链路验证，不能仅凭单测通过就合入。

Git evidence: 2次revert在moe_distribute_dispatch_v2(syncall优化和fullmesh模板)

### 7.7 SocVersion vs NpuArch Inconsistency

[Severity]: Warning

```cpp
// 旧模式(MoE): socVersion字符串
IsTargetPlatformSocVersion(nodeName, PLATFORM_A2)

// 新模式(MatMul): NpuArch枚举
IsTargetPlatformNpuArch(nodeName, NPUARCH_A5)
```

新代码应统一使用NpuArch枚举，避免socVersion字符串比较。

### 7.8 Weak Symbol Duplication

[Severity]: Warning

NnopbaseSetHcclServerType/NnopbaseDisableOptionalInput等weak符号在每个op_api文件中重复声明。应提取到公共头文件。

新算子op_api必须复用已有的公共枚举和工具定义(如matmul_all_reduce_util.h中的NnopbaseHcclServerType)，不应重新定义。

Git evidence: e8d52ffb (matmul_all_reduce clean code已将6个文件的重复定义集中到util.h)

### 7.9 sizeof(DataType Enum) vs GetSizeByDataType

[Severity]: Error

对ge::DataType枚举变量使用sizeof()得到的是枚举的存储大小(通常4字节)，而非该数据类型的实际字节数。

```cpp
// bad: sizeof取枚举存储大小，而非数据类型字节数
uint64_t sizeOfSingleM = args_.kValue * sizeof(args_.geAType);

// good: 使用API获取实际数据类型大小
uint64_t sizeOfSingleM = args_.kValue * ge::GetSizeByDataType(args_.geAType);
```

Git evidence: 809fc00e (AllGatherMMV2和MMReduceScatterV2的output limit check修复)

### 7.10 Attr Pointer Use-Before-Null-Check

[Severity]: Error

在op_host的tiling函数中，必须先判空再解引用attr指针。

```cpp
// bad: 先解引用再判空
int64_t epWorldSize = *epWorldSizePtr;
OP_TILING_CHECK(epWorldSizePtr == nullptr, ...);

// good: 先判空再解引用
OP_TILING_CHECK(epWorldSizePtr == nullptr, OP_LOGE(...), return ge::GRAPH_FAILED);
int64_t epWorldSize = *epWorldSizePtr;
```

Git evidence: 916aa93c (combineV2 tiling split中可见此模式)

### 7.11 OP_LOGE Without Runtime Values

[Severity]: Warning

错误日志应包含关键运行时数值，便于问题定位。

```cpp
// bad: 只有描述没有数值
OP_LOGE(opName_, "Unsupported x1 size. Size exceeds 256MB.");

// good: 包含实际数值
OP_LOGE(opName_, "Unsupported x1 size %lu exceeds 256MB limit.", sizeOfSplitM);
```

Git evidence: 809fc00e (修复后的OP_LOGE增加了sizeOfSingleM/sizeOfSplitM打印)

### 7.12 Boundary Comparison Off-by-One in OP_TILING_CHECK

[Scope]: op_host (tiling layer)
[Severity]: Error

OP_TILING_CHECK中的边界比较运算符(>=, >, <, <=)容易写错，导致off-by-one bug。

```cpp
// bad: >= 应为 > (允许等于MAX_VALUE的合法值)
OP_TILING_CHECK(n1_ >= N1_MAX_VALUE, return ..., "n1 exceeds limit");

// good: > (正确边界)
OP_TILING_CHECK(n1_ > N1_MAX_VALUE, return ..., "n1 exceeds limit");
```

Git evidence: 156765f3 (alltoallv_quant_gmm的n1/h2/bs边界比较修复)

### 7.13 Validation Logic Layer Misplacement

[Scope]: op_api / op_host
[Severity]: Recommended

校验逻辑应放在拥有最完整信息的层:
- op_api(aclnn层): 仅做null/type/dtype基础校验(不需要shape推导)
- op_host(tiling层): 做需要shape信息的校验(如groupSize推导、维度范围校验)

演进方向: 校验从op_api下沉到op_host(tiling)，因为tiling层有更完整的shape和attr信息。

```cpp
// bad: op_api层硬编码校验(无法利用shape推导)
if (groupSize != expectedGroupSize) return ERR;

// good: tiling层用公共工具推导后校验
auto inferredGroupSize = mc2tiling::Mc2TilingUtils::InferGroupSize(m, n, k);
OP_TILING_CHECK(groupSize != inferredGroupSize, ...);
```

Git evidence: 8709a41a (groupSize checking移到tiling侧并支持自动推导), 156765f3 (CheckEmptyTensor从aclnn删除下沉到tiling)

### 7.14 Kernel Buffer Address Calculation Without Layout Documentation

[Scope]: op_kernel (MoE kernel layer)
[Severity]: Recommended

kernel中超过3个地址偏移计算的缓冲区划分必须附带ASCII art内存布局注释图。
裸地址偏移计算应封装为语义化方法。

```cpp
// bad: 裸地址偏移散布在kernel各处
auto addr = windowInGM_ + halfWinSize_ * bufferId_ + WIN_SIZE + serverId_ * STATE_OFFSET;

// good: 封装为语义化方法 + 内存布局注释
/*
 * WindowIn Layout:
 * |--- Half 0 ---|--- Half 1 ---|  (Ping-Pong)
 * | RDMA Flag | RDMA Data | IPC Data | IPC Flag | Magic |
 * | flagSize  | dataSize  | dataSize | flagSize | 4B    |
 */
auto addr = GetLocalRecvBuffFlagAddr(srcRankId);
```

Git evidence: 77411b7d (Dispatch/Combine HCCL Buffer地址计算提取到MoeDistributeA2AddrInfo基类)

---

## 8. Evolution Directions

| Aspect | Old | New | Status |
|--------|-----|-----|--------|
| Communication control | AICPU proxy | CCU direct drive | arch35 fully adopted |
| Tiling key design | Implementation-oriented (16 params) | Business-oriented APT (12 params) | 6 operators migrated |
| MatMul execution | BaseBlock | ASW Block | arch35 fully adopted |
| Pipeline scheduling | Manual | mc2_templates framework | 4 operators adopted |
| Version management | In-directory ifdef | Separate v2 directory | Widely adopted |
| Communication lifecycle | Kernel-embedded | setup/teardown decoupled | v3 piloting |
| Platform detection | socVersion string | NpuArch enum | Migration in progress |
| HCCL init | Init(context) | InitV2(context, tiling) | arch35 adopted |

---

## 9. Testing Conventions

### 9.1 Test Structure

```
tests/ut/
  op_host/   -- tiling/infershape UT (61 files)
  op_api/    -- aclnn API UT (38 files)
  op_kernel/ -- kernel UT (22 files)
```

### 9.2 Kernel UT Pattern

```cpp
#include "gtest/gtest.h"
#ifdef __CCE_KT_TEST__
#include "tikicpulib.h"
#include "data_utils.h"
#endif

#include "../../../op_kernel/op_name.cpp"  // 直接include源码

class OpNameTest : public testing::Test {
    static void SetUpTestCase() {
        g_hcclContextReserved[0] = (uint8_t*)AscendC::GmAlloc(ctxSize);
    }
    static void TearDownTestCase() {
        AscendC::GmFree((void*)g_hcclContextReserved[0]);
    }
};

TEST_F(OpNameTest, basic_test) {
    AscendC::SetKernelMode(KernelMode::MIX_MODE);
    // ... 构造输入 → 调用kernel → 验证输出
}
```

### 9.3 Documentation Pattern

每个aclnn API对应一个.md文件，结构:
1. Product support table
2. Feature description + formula
3. Function prototype (两段式)
4. Parameter description (table)
5. Supported data types
6. Constraints and limitations
7. Usage example

---

## 10. Quick Reference: Registration Macros

| Layer | Macro | File | Coverage |
|-------|-------|------|----------|
| op_host | OP_ADD(OpName) | def.cpp | 35/35 |
| op_host | IMPL_OP_INFERSHAPE(OpName) | infershape.cpp | 33/33 |
| op_host | IMPL_OP_OPTILING(OpName) | tiling.cpp | 37/37 |
| op_kernel | REGISTER_TILING_DEFAULT(TilingData) | op_name.cpp | 44/44 |
| op_graph | IMPL_OP(OpName).CalcOpParam().GenerateTask() | gen_task.cpp | 26/26 |
| op_graph | IMPL_OP(OpName).OpExecuteFunc() | fallback.cpp | 23/23 |
| op_api | extern "C" aclnnOpNameGetWorkspaceSize/aclnnOpName | aclnn_*.cpp | 32/32 |

---

## 11. Developer's Casebook (案例索引)

基于991条commit(main 363 + dev 680，去重后991条独有message)的全量分析，
精选29个landmark case做了深度diff阅读和决策点分析。

详细案例分析见: references/dig-repo-code/ops-transformer-mc2/developers-casebook.md

### 11.1 Commit Distribution

| Category | Count | % | Key Insight |
|----------|-------|---|-------------|
| BUGFIX | 243 | 24.5% | 同步bug和类型错误是两大类 |
| DOC_TEST | 210 | 21.2% | 每4个code commit对应1个doc commit |
| FEATURE | 198 | 20.0% | 新功能是最大的"生产性"类别 |
| REFACTOR | 188 | 19.0% | socVersion迁移和tiling拆分是主要主题 |
| INFRA | 68 | 6.9% | 公共类型变更波及11~24个算子目录 |
| ARCH_ADAPT | 30 | 3.0% | arch35适配的专门类别 |
| OPTIMIZATION | 20 | 2.0% | 仅2%说明性能优化非常谨慎 |
| REVERT | 19 | 1.9% | dev仓16个(实验场) vs main仓3个 |
| NEW_OP | 15 | 1.5% | 稀有里程碑事件 |

### 11.2 Error-Fix Pairs (5 documented)

1. Sync optimization: 4-hour revert cycle (MAIN:81ef4922 → 8ade8c3e)
2. TilingKey rectification: 5 sequential reverts in dev (batch apply failure)
3. mm_rs optimization: 3549-line full revert (SmallM decision tree)
4. MC2 TilingData restructuring: full revert (broke downstream consumers)
5. mc2_3rd decoupling: full revert (broke build chain)

### 11.3 Key Insight: OPT-2 vs OPT-3

同一开发者同一天提交的两个SyncAll优化，一个成功一个在5.5小时内被revert。
根因差异:

| Dimension | OPT-3 (Failed) | OPT-2 (Succeeded) |
|-----------|-----------------|---------------------|
| statusBuf size | totalExpertNum/aivNum approximation | moeExpertNumPerRank * epWorldSize exact |
| ReduceSum workspace | Reused gatherMaskOutBuf_ | Independent workLocalBuf_ |
| tokenNumBuf size | Missing sizeof(int64_t) factor | Correctly included |

Iron rules for kernel buffer optimization:
1. Buffer size must be derived from exact data dimensions (no approximation)
2. ReduceSum workspace must be independently allocated (no reuse)
3. Each HardEvent must precisely match a real data dependency

---

## 12. Scenario-Based Development Guide (场景化指南)

### 12.1 "新增MC2算子" Decision Checklist

[Scope]: Complete checklist for new operator development
[Cases]: NOP-1(4-layer), NOP-2(flat), NOP-3(batch scaffold), NOP-4(arch extend)

1. Determine operator type: MatMul+Comm(AIC+AIV) vs pure data routing(AIV only)
2. Determine arch scope: arch32+arch35(dual kernel entry) vs single arch
3. Create 4-layer directory skeleton in one commit (op_host + op_kernel + op_graph + op_api)
4. op_host: def.cpp(OP_ADD) + infershape(IMPL_OP_INFERSHAPE) + tiling(IMPL_OP_OPTILING)
   - TilingBaseClass inheritance for MatMul+Comm; flat for simple operators
   - Mc2CcTilingConfig mandatory; tiling_key.h with ASCENDC_TPL_ARGS_DECL
5. op_kernel: REGISTER_TILING_DEFAULT + AIC/AIV split + GetHcclContext + pipeline
   - Dual entry: .cpp(arch32) + _apt.cpp(arch35)
6. op_graph: gen_task(IMPL_OP + CommonKFCMc2CalcParamFunc) + fallback(EXEC_OPAPI_CMD)
7. op_api: two-phase API + HcclServerType dispatch(CCU for arch35, AICPU/MTE for arch32)
8. Tests(op_host UT priority) + docs(per aclnn API) + examples

Pitfalls:
- __NPU_ARCH__ == 3101 typo (should be 3510), compiles silently [ARC-4]
- Quantization variants must be physically isolated via ORIG_DTYPE macro [NOP-1]
- New shared code goes in operator-local directory first, move to mc2/common/ after 2+ users [NOP-1]

### 12.2 "给已有算子添加arch35支持" Decision Checklist

[Scope]: Adapting existing arch32 operator for Ascend950
[Cases]: NOP-4, ARC-1~4

1. op_host: New *_tiling_950.cpp with REGISTER_TILING_TEMPLATE_WITH_ARCH
   - Use Mc2InitTiling + Mc2CcTiling (not legacy Init + SetReduceDataTypeAbility)
   - Add DoXxxReTiling() for CCU hardware constraints (256MB AlltoAll limit) [ARC-3]
2. op_kernel: New _apt.cpp + arch35/ directory
   - InitV2 + SetCcTilingV2 (not Init)
   - Per-tile HcclWait (not single Wait)
3. op_api: Add DAV_3510 branch for NnopbaseSetHcclServerType(CCU)
4. Post-validation: grep -r '__NPU_ARCH__.*3101' to catch typos [ARC-4]
5. Clean legacy: Remove redundant old API calls after new API is stable [ARC-2]

### 12.3 "修复通信同步bug" Diagnosis Guide

[Scope]: Diagnosing and fixing sync-related bugs
[Cases]: BUG-1~5, OPT-2/OPT-3, REV-1

Diagnosis by symptom:
- Large shape failure → Check constant limits (MAX_HCCL_HANDLE = 63, not 16) [BUG-1]
- Specific rank count failure → Check integer division/alignment edge cases [BUG-3]
- OOM → Check bytes vs elements unit confusion [BUG-4]
- Data corruption → Check SyncAll/HardEvent completeness [OPT-3]

3-layer audit method (from BUG-3):
1. infershape: bracket precedence in division/alignment formulas
2. tiling: blockDim condition coverage for all legal rank counts
3. kernel: RoundUp alignment, Mask bit-width, DataCopyPad, sync event matching

Parameter boundary test matrix: rank={1,2,7,8,9,16,64} x shape={1,small,exact-tile,large} x dtype={fp16,bf16,fp8,int8}

Fast revert rule: If bug found within 4 hours of merge, revert immediately rather than hot-fix [REV-1]

### 12.4 "添加量化支持" Decision Checklist

[Scope]: Adding quantization mode to existing MC2 operator
[Cases]: FEA-2, NOP-1, REF-5

Decision: Embedded variant vs separate operator?
- Embedded (tiling_key QUANTMODE dimension): quant logic tightly coupled with main logic
- Separate (quant_* directory): completely different quant logic or many extra inputs
- Cost awareness: Adding 4 API parameters costs 11 files; removing them costs 330 lines [REF-5]

Checklist:
1. Extend tiling_key.h with quantMode dimension
2. Add Scale/Offset as OPTIONAL_INPUT in def.cpp
3. Kernel: ORIG_DTYPE macro isolation for int4/int8 vs fp16/bf16 [NOP-1]
4. If using mc2_templates: add QuantizeStage + extend Pipeline [FEA-2]
5. GetAttrPointer type must match op_def: always use int64_t for attr pointers [BUG-2]

### 12.5 "优化tiling性能" Safety Spectrum

[Scope]: Performance optimization risk assessment
[Cases]: OPT-1~3, REV-2, REF-2

Safety spectrum (safe → risky):
1. Workspace layout reorganization: Change "when to write GM" not "what to write" [OPT-1] -- SAFE
2. Tiling parameter tuning: Change split strategy -- MEDIUM RISK
   - Decision tree autotuning has coverage problems [REV-2: 3549-line full revert]
   - Must have fallback to general path
3. SyncAll replacement: Replace sync mechanism -- HIGH RISK
   - 3 iron rules: exact buffer size / independent workspace / matched HardEvent [OPT-2 vs OPT-3]
   - Must draw data dependency graph before attempting

Large optimization staging: Split into framework + rules, not one mega-commit [REV-2 lesson]

---

## 13. Gray Area Judgment Framework (灰色地带判断)

### 13.1 When to Split Tiling Functions

Split signals (any 2 = must split):
- Lines > 300 [REF-2: combineV2 triggered at ~350]
- 3+ semantically unrelated parameter groups
- 2+ SoC/arch branch paths in one function

Split strategy: By "parameter domain" (expert/shared/TP_EP), NOT by "operation type" (read/validate/set)

Exception: Linear single-path tiling (like AllGatherMatmul 634 lines) needs no split

### 13.2 When to Use Inheritance vs Flat Structure

| Condition | Recommended |
|-----------|-------------|
| Variants <= 3 | Flat single file |
| Variants 4-7, > 50% shared | Base + 2-3 subclasses |
| Variants >= 8, > 70% shared | Deep inheritance |
| First implementation | Always flat first |
| Non-MatMul+Comm paradigm | Flat (don't force TilingBaseClass) [NOP-2] |

Evolution direction: flat → inherited is safe; reverse is expensive.

### 13.3 When to Reuse vs Copy-Paste

"Local first, public later" strategy [NOP-1]:
1. Initial: Create in operator-local directory
2. Stable: Copy to mc2/common/ when 2nd operator needs it
3. Mature: Becomes official MC2 infrastructure after 3+ users

Must publicize: When .cpp and _stub.cpp have identical code [REF-1]
OK to copy: Utilities under 3 lines (Mc2SyncAll/Mc2CoreType) where include dependencies are complex

### 13.4 Which Layer for New Functionality

| Feature Type | Primary Layer | Evidence |
|-------------|---------------|----------|
| New input parameter | op_host(def) + op_graph(proto) + op_api(signature) | REF-5: 4 params = 11 files |
| New comm pattern | op_kernel + op_host(tiling_key) | FEA-4 |
| New dtype/quant | op_kernel + op_host(tiling_key) + op_api(validation) | FEA-2 |
| Hardware constraint | op_host(tiling ReTiling only) | ARC-3: CCU hang has no error code |
| New arch | All 4 layers | NOP-4, ARC-1~4 |
| Validation migration | op_api(delete) + op_host(add tiling check) | 8709a41a, 156765f3 |
| Kernel buffer refactor | op_kernel(new base class + layout doc) | 77411b7d |

Key: Hardware constraints can ONLY be handled in tiling layer (kernel has no recovery mechanism).

Validation layer migration rule:
- op_api: null/type/dtype basic checks only (no shape derivation needed)
- op_host(tiling): shape-dependent validation (groupSize inference, dimension range checks)
- Direction: validation migrates from op_api to op_host as operators mature [7.13]

### 13.5 Three Meta-Principles

1. Reversibility first: Batch apply OK for single-layer changes; per-operator validation required for cross-layer interface changes [REV-3: 5 reverts]

2. Exact modeling over approximation: Buffer sizes from exact dimensions, constants from specs, types from op_def cross-check [OPT-2 success vs OPT-3 failure]

3. Run first, refactor later: Flat → inherited / manual → template / local → public. Don't pursue "final architecture" on first implementation [REV-2: 3549-line SmallM revert]
