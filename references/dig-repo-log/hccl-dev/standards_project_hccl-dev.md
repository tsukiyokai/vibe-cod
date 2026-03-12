# hccl-dev 代码审查规则

基于133条非merge提交中10条确认缺陷(7.5%)的全量分析。hccl-dev是HCCL的开源开发仓库，缺陷以API迭代期的接口不匹配为主。

---

## 规则总览

| 规则 | 类别 | 严重等级 |
|------|------|---------|
| ENUM-01 | 枚举/标识符混淆 | 严重 |
| ENUM-02 | 枚举/标识符混淆 | 严重 |
| ENUM-03 | 枚举/标识符混淆 | 中等 |
| PROP-01 | 接口变更传播 | 严重 |
| PROP-02 | 接口变更传播 | 严重 |
| PROP-03 | 接口变更传播 | 中等 |
| COPY-01 | 复制粘贴错误 | 严重 |
| COND-01 | 条件判断逻辑 | 严重 |
| CMAKE-01 | CMake配置 | 中等 |

---

## ENUM-01：通信引擎枚举变体混淆

严重等级：严重

缺陷描述：CANN平台存在多个语义相近的通信引擎枚举值（COMM_ENGINE_AICPU / COMM_ENGINE_AICPU_TS / COMM_ENGINE_CPU / COMM_ENGINE_CPU_TS / COMM_ENGINE_AIV等），使用了错误的变体会导致执行路径、资源分配和launch模式全部错误。

典型代码示例：

缺陷代码（commit 7347fee3，src/ops/scatter/scatter_op.cc 6处全部用错）:
```cpp
// 错误：使用AICPU而非AICPU_TS
param.engine = CommEngine::COMM_ENGINE_AICPU;
// ...
if (param.engine == COMM_ENGINE_AICPU) {
    // AICPU和AICPU_TS是不同的执行模式
}
```

修复后:
```cpp
param.engine = CommEngine::COMM_ENGINE_AICPU_TS;
if (param.engine == COMM_ENGINE_AICPU_TS) {
    // 正确使用AICPU_TS变体
}
```

审查检查方法：
1. 搜索所有CommEngine枚举使用点，确认变体选择与设计文档一致
2. 同一文件/函数内的引擎类型应前后一致
3. 新增引擎类型使用时，要求开发者说明选择理由

关联commit：7347fee3

---

## ENUM-02：平台字符串字面量拼写错误

严重等级：严重

缺陷描述：平台对launch mode等字符串做精确匹配，拼写不一致（如"AICPU" vs "AI_CPU"）会导致功能静默失效。

典型代码示例：

缺陷代码（commit 13b6032d，src/ops/scatter/scatter_op.cc）:
```cpp
// 错误：平台期望的是"AI_CPU"（带下划线）
launchMode = "AICPU";
```

修复后:
```cpp
launchMode = "AI_CPU";
```

审查检查方法：
1. 所有平台相关字符串字面量应有对应常量定义，禁止裸字符串
2. 审查时将字符串字面量与平台文档/头文件中的定义逐字符比对
3. 搜索相似但不同的字符串（如"AICPU"/"AI_CPU"/"AiCpu"）是否混用

关联commit：13b6032d

---

## ENUM-03：API参数名不同步

严重等级：中等

缺陷描述：上游API参数名变更后（如blockDim→numBlocks），调用侧变量名未同步。虽不影响编译和运行，但导致代码与API定义不一致，增加维护成本和理解难度。

典型代码示例：

缺陷代码（commit 2d8d548a，3个文件）:
```cpp
// API已将参数重命名为numBlocks，但调用侧仍用旧名
constexpr uint32_t blockDim = 1;
aclrtLaunchKernelWithConfig(funcHandle, blockDim, stream, &cfg, argsHandle, nullptr);
```

修复后:
```cpp
constexpr uint32_t numBlocks = 1;
aclrtLaunchKernelWithConfig(funcHandle, numBlocks, stream, &cfg, argsHandle, nullptr);
```

审查检查方法：
1. API参数名变更时，全局搜索旧名确保所有调用点、stub、示例代码同步
2. 代码中的变量命名应与被调API的参数名保持一致

关联commit：2d8d548a

---

## PROP-01：结构体/类字段变更未同步到所有使用方

严重等级：严重

缺陷描述：修改结构体定义（增删字段）后，部分使用方（特别是测试代码、mock、stub）未同步更新，导致编译失败。

典型代码示例：

缺陷代码（commit c11a5289，test/.../sim_communicator.cc）:
```cpp
// HcclCommConfig中commEngine/threadNum/notifyNumPerThread已被移除
// 但测试代码仍在设置这些字段
commConfig.commEngine = HCCL_COMM_ENGINE_CONFIG_NOT_SET;
commConfig.threadNum  = HCCL_COMM_THREADNUM_CONFIG_NOT_SET;
commConfig.notifyNumPerThread = HCCL_COMM_NOTIFY_NUM_PER_THREAD_CONFIG_NOT_SET;
```

修复：删除对已移除字段的赋值。

审查检查方法：
1. 修改struct/class定义时，全局grep所有使用点，范围必须包含test/目录
2. 提交变更前在本地完成全量编译
3. 注意同一struct可能在.cc和.h中都有使用

关联commit：c11a5289, 8222fcf8

---

## PROP-02：构建目标缺少依赖源文件

严重等级：严重

缺陷描述：CMakeLists中新增library target时遗漏了被引用的源文件，链接阶段出现undefined symbol。

典型代码示例：

缺陷代码（commit beb0ed54，src/CMakeLists.txt）:
```cmake
# scatter_aicpu_kernel库遗漏了5个依赖源文件
add_library(scatter_aicpu_kernel SHARED
    ${CMAKE_CURRENT_SOURCE_DIR}/common/utils.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/common/adapter_acl.cc
    # 缺少: config_log.cc, sal.cc, log.cc,
    #        adapter_error_manager_pub.cc, alg_env_config.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/ops/aicpu/kernel_launch.cc
    ...
)
```

审查检查方法：
1. 新增add_library时，逐个检查源文件中的#include和函数调用是否有对应.cc文件在target中
2. 关注间接依赖：A.cc include了B.h，B.h的实现在B.cc中，B.cc是否在target里
3. 不同编译环境（Host/AICPU）下API可用性不同，需要条件编译隔离

关联commit：beb0ed54

---

## PROP-03：公共API函数名与头文件声明不匹配

严重等级：中等

缺陷描述：公共API的.cc实现使用了错误的函数名（与内部函数同名），与.h声明不一致，导致链接失败。

典型代码示例：

缺陷代码（commit cae52923，src/ops/interface/hccl_collective_op.cc）:
```cpp
// 头文件声明的是HcclBatchSendRecv，但实现写成了HcclBatchSendRecvInner
HcclResult HcclBatchSendRecvInner(HcclSendRecvItem* sendRecvInfo, ...) {
    return HcclBatchSendRecvInner(sendRecvInfo, itemNum, comm, stream);
    // 这还会导致无限递归（自己调自己）
}
```

修复后:
```cpp
HcclResult HcclBatchSendRecv(HcclSendRecvItem* sendRecvInfo, ...) {
    return HcclBatchSendRecvInner(sendRecvInfo, itemNum, comm, stream);
}
```

审查检查方法：
1. 公共API函数的.cc定义必须与.h声明的函数签名精确匹配
2. 包装函数的名字不应与被包装函数相同（否则变成递归调用）

关联commit：cae52923

---

## COPY-01：复制粘贴后变量名未替换

严重等级：严重

缺陷描述：从功能相似的代码块复制后，未将所有变量名替换为新的上下文对应值，导致逻辑完全错误。

典型代码示例：

缺陷代码（commit 8222fcf8引入，11b7211a修复，build.sh）:
```bash
# run_st()从run_ut()复制而来，但条件变量未替换
function run_st() {
  if [[ "X$ENABLE_UT" = "Xon" ]]; then  # 应该是ENABLE_ST
    ...
  fi
}
```

修复后:
```bash
function run_st() {
  if [[ "X$ENABLE_ST" = "Xon" ]]; then
    ...
  fi
}
```

审查检查方法：
1. 当新增函数与已有函数结构高度雷同时，要求开发者列出所有替换点
2. 构建脚本中函数名后缀（_ut/_st）与检查的变量名后缀（ENABLE_UT/ENABLE_ST）必须对应
3. 可用grep确认：函数名中的关键字是否在函数体中一致出现

关联commit：8222fcf8(引入), 11b7211a(修复)

---

## COND-01：设备类型判断遗漏组合条件

严重等级：严重

缺陷描述：设备类型分支仅检查硬件型号，遗漏了运行模式/环境变量等组合约束，导致在非预期配置下走入错误路径。

典型代码示例：

缺陷代码（commit 19d20206，src/ops/scatter/scatter_op.cc）:
```cpp
// 仅检查设备类型，忽略了运行模式环境变量
if (deviceType != DevType::DEV_TYPE_910_93 && deviceType != DevType::DEV_TYPE_910B) {
    return HcclScatterInner(...);
}
```

修复后:
```cpp
// 同时检查设备类型和环境变量HCCL_OP_EXPANSION_MODE
if (!RunIndependentOpExpansion(deviceType)) {
    return HcclScatterInner(...);
}

// RunIndependentOpExpansion的实现：
// 910_93: 支持HOST_TS和AI_CPU
// 910B: 仅支持HOST_TS
// 其他: 不支持
```

审查检查方法：
1. 设备类型分支逻辑是否考虑了环境变量、运行模式等配置维度
2. 新增设备类型时是否更新了所有设备类型判断点
3. 提取设备能力判断为独立函数，避免多处硬编码同一逻辑

关联commit：19d20206

---

## CMAKE-01：CMake版本兼容性与参数格式

严重等级：中等

缺陷描述：CMake脚本中使用了高版本特性（如CMAKE_CURRENT_FUNCTION_LIST_DIR需要3.17+），或`-D`参数后加空格导致解析失败。

典型代码示例：

缺陷代码（commit 1f13573a，cmake/func.cmake）:
```cmake
# 问题1: CMAKE_CURRENT_FUNCTION_LIST_DIR需要CMake 3.17+
-P "${CMAKE_CURRENT_FUNCTION_LIST_DIR}/_pack_stage.cmake"

# 问题2: -D后有空格导致某些版本解析失败
set(manifest_arg "-D _MANIFEST_FILE=${staging_dir}/${ARG_MANIFEST}")
```

修复后:
```cmake
# 在文件作用域保存路径（兼容低版本CMake）
set(_FUNC_CMAKE_DIR "${CMAKE_CURRENT_LIST_DIR}")
# ...
-P "${_FUNC_CMAKE_DIR}/_pack_stage.cmake"

# -D后不加空格
set(manifest_arg -D_MANIFEST_FILE=${staging_dir}/${ARG_MANIFEST})
```

审查检查方法：
1. 对照项目cmake_minimum_required确认使用的CMake特性版本要求
2. `-D`后紧跟变量名，不加空格
3. 函数内引用路径变量时，确认变量在函数作用域内可用

关联commit：1f13573a

---

## 附录：缺陷统计

| 指标 | 数值 |
|------|------|
| 总提交(非merge) | 133 |
| 确认缺陷 | 10 |
| 缺陷率 | 7.5% |
| Revert事件 | 2 (MR !18 打包路径flip-flop, MR !34 API重命名撤回) |
| 热点文件 | scatter_op.cc (4/10) |

| 缺陷类别 | 数量 | 占比 |
|----------|------|------|
| 枚举/标识符混淆 | 3 | 30% |
| 接口变更传播不完整 | 3 | 30% |
| 复制粘贴错误 | 2 | 20% |
| 条件判断逻辑不完整 | 1 | 10% |
| CMake配置错误 | 1 | 10% |

### 与hccl主仓对比

| 维度 | hccl | hccl-dev |
|------|------|----------|
| 缺陷率 | ~20% | 7.5% |
| 主要缺陷类型 | 并发/资源/初始化 | 接口不匹配/枚举混淆 |
| 缺陷集中度 | 分散 | 高度集中于scatter_op.cc |
| 仓库特征 | 成熟代码维护 | 开发期API快速迭代 |

hccl-dev的低缺陷率和缺陷类型（以接口不匹配为主而非逻辑错误）反映了开发期仓库的特点：大量提交是API设计/重构，实际功能代码（scatter算子）集中且处于快速迭代。

### 热点文件当前仍存在的风险

scatter_op.cc当前代码中发现以下问题（详见hotspot_analysis.md）：
1. 第356行错误日志引用错误变量（aclRet vs ret） - 当前bug
2. 全局g_notifies数组的并发安全性
3. 函数内宏定义(#define ACL_NOTIFY_DEFAULT)
4. 多处"faled"拼写错误
