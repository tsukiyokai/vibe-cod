# hccl-dev 缺陷提交全量分析

共10条缺陷提交，全量分析如下。

---

### 2d8d548a fix blockdim -> numblocks
- 根因类别：API参数名不一致
- 涉及文件：examples/04_custom_ops_p2p/op_host/launch_kernel.cc, src/ops/scatter/scatter_op.cc, test/st/algorithm/utils/src/hccl_proxy/aclrt_stub.cc
- 缺陷描述：`aclrtLaunchKernelWithConfig`的第二个参数在上游API中从`blockDim`重命名为`numBlocks`，但调用侧和stub仍使用旧名`blockDim`。涉及3个文件5处修改。虽然编译不报错（参数名不影响调用），但与API定义不一致，造成维护困惑。
- 修复模式：将所有`blockDim`变量名统一改为`numBlocks`
- 可审查性：中
- 审查规则建议：API参数名变更后，全局搜索旧名确保所有调用点和stub同步更新

### 13b6032d fix AICPU
- 根因类别：字符串字面量拼写错误
- 涉及文件：src/ops/scatter/scatter_op.cc
- 缺陷描述：`SetLaunchMode`函数中AICPU引擎的launch mode字符串写成`"AICPU"`，正确值应为`"AI_CPU"`（带下划线）。平台对该字符串做精确匹配，拼写不一致会导致launch mode设置失效。
- 修复模式：`"AICPU"` → `"AI_CPU"`
- 可审查性：高
- 审查规则建议：平台魔法字符串应定义为常量或枚举，避免裸字符串拼写错误；审查时关注字符串字面量与平台约定是否一致

### 7347fee3 modifyAicpuToAicpuTs251219
- 根因类别：枚举值混淆
- 涉及文件：src/ops/scatter/scatter_op.cc（6处修改）
- 缺陷描述：scatter算子的通信引擎应设为`COMM_ENGINE_AICPU_TS`，但错误使用了`COMM_ENGINE_AICPU`。两者是不同的执行模式（AICPU vs AICPU_TS），混用导致launch路径和资源分配逻辑错误。6处比较和赋值全部用错。
- 修复模式：全部6处 `COMM_ENGINE_AICPU` → `COMM_ENGINE_AICPU_TS`
- 可审查性：高
- 审查规则建议：当存在语义相近的枚举值（AICPU/AICPU_TS/AIV/HOSTCPU_TS等）时，审查每个使用点是否选择了正确的变体；新代码引入引擎类型时需与设计文档对照

### 19d20206 fix bug（设备类型判断逻辑不完整）
- 根因类别：条件判断逻辑不完整
- 涉及文件：src/ops/scatter/scatter_op.cc, src/common/alg_env_config.cc, src/common/alg_env_config.h
- 缺陷描述：原代码仅以设备类型（910_93/910B）判断是否走独立算子展开路径：`if (deviceType != DEV_TYPE_910_93 && deviceType != DEV_TYPE_910B)`。但实际还需考虑环境变量`HCCL_OP_EXPANSION_MODE`的值：910_93支持HOST_TS和AI_CPU两种模式，910B仅支持HOST_TS。硬编码设备类型检查会导致在非预期模式下也走新路径。
- 修复模式：提取`RunIndependentOpExpansion()`函数，同时检查设备类型和环境变量
- 可审查性：中
- 审查规则建议：设备类型分支逻辑中，关注是否遗漏了运行模式/环境变量等组合条件；新设备类型加入时是否需要更新所有判断点

### 11b7211a fix st test bug（复制粘贴变量名错误）
- 根因类别：复制粘贴错误
- 涉及文件：build.sh
- 缺陷描述：`run_st()`函数中条件判断误用了`$ENABLE_UT`变量（从`run_ut()`复制而来），应为`$ENABLE_ST`。导致ST测试永远不会执行（需要`-s`参数设置`ENABLE_ST`，但函数检查的是`ENABLE_UT`）。
- 修复模式：`ENABLE_UT` → `ENABLE_ST`
- 可审查性：高
- 审查规则建议：从相似函数复制代码后，逐行检查所有变量名是否已替换；构建脚本中的条件变量应与对应的命令行参数匹配

### c11a5289 fix compile error（结构体字段移除未同步）
- 根因类别：接口变更传播不完整
- 涉及文件：test/st/algorithm/utils/src/hccl_proxy/communicator/sim_communicator.cc
- 缺陷描述：`HcclCommConfig`结构体中移除了`commEngine`、`threadNum`、`notifyNumPerThread`三个字段（前序提交修改了头文件），但sim_communicator.cc中`GetDefaultCommConfig`函数仍在设置这些已不存在的字段，导致编译失败。
- 修复模式：删除对已移除字段的赋值语句
- 可审查性：中
- 审查规则建议：修改结构体/类定义时，全局搜索所有使用该结构体的位置确认同步更新；测试代码和mock/stub也需纳入搜索范围

### 8222fcf8 fix test algorithm compile error（多处编译错误修复）
- 根因类别：接口变更传播不完整 + 缺少头文件
- 涉及文件：build.sh, test/st/algorithm/utils/src/hccl_proxy/communicator/sim_communicator.cc, test/st/algorithm/utils/src/hccl_proxy/communicator/sim_context_manager.h
- 缺陷描述：(1) sim_communicator.cc中`SetIndependentOpConfig`函数使用了`HcclCommConfig`已移除的字段`commEngine/threadNum/notifyNumPerThread`；(2) sim_context_manager.h缺少`#include "dtype_common.h"`导致编译错误；(3) build.sh新增`run_st()`函数但误用了`ENABLE_UT`变量（后在11b7211a修复）。注意：本提交引入的build.sh bug在3天后被11b7211a修复。
- 修复模式：删除对已移除字段的使用、补充缺失include、新增ST构建功能
- 可审查性：中
- 审查规则建议：一个提交包含多类修改时需逐项审查；新增函数如果复制自相似函数，核查变量名替换是否完整

### 1f13573a fix cmake bug（CMake兼容性和参数格式）
- 根因类别：CMake兼容性 + 参数格式错误
- 涉及文件：cmake/func.cmake
- 缺陷描述：两个问题：(1) 使用了`CMAKE_CURRENT_FUNCTION_LIST_DIR`变量（CMake 3.17+才支持），在低版本CMake上为空导致找不到`_pack_stage.cmake`脚本；修复为在文件作用域用`CMAKE_CURRENT_LIST_DIR`保存路径。(2) `-D _MANIFEST_FILE=`中`-D`和变量名间有空格，某些CMake版本下会解析失败；改为`-D_MANIFEST_FILE=`。
- 修复模式：用文件级变量替代函数级变量；去除-D后多余空格
- 可审查性：高
- 审查规则建议：CMake脚本中避免使用高版本特性（检查项目最低CMake版本要求）；`-D`参数后不要加空格

### cae52923 fix batch send recv api name（API函数名错误）
- 根因类别：函数命名错误
- 涉及文件：src/ops/interface/hccl_collective_op.cc
- 缺陷描述：公共API函数应命名为`HcclBatchSendRecv`，但实际被命名为`HcclBatchSendRecvInner`（与内部实现函数同名）。导致链接时找不到`HcclBatchSendRecv`符号（头文件声明的是`HcclBatchSendRecv`，但实现用了`Inner`后缀）。函数体内调用`HcclBatchSendRecvInner`是正确的，只是外层包装函数名写错。
- 修复模式：函数名 `HcclBatchSendRecvInner` → `HcclBatchSendRecv`
- 可审查性：高
- 审查规则建议：公共API函数名必须与头文件声明精确匹配；审查时对照.h声明和.cc定义的函数签名

### beb0ed54 undefined symbol bugfix（链接缺失 + 编译环境隔离）
- 根因类别：构建配置不完整 + 编译宏隔离缺失
- 涉及文件：src/CMakeLists.txt, src/common/adapter_acl.cc
- 缺陷描述：两个问题：(1) `scatter_aicpu_kernel`库的CMakeLists遗漏了5个依赖源文件（config_log.cc, sal.cc, log.cc, adapter_error_manager_pub.cc, alg_env_config.cc），链接时出现undefined symbol。(2) `hcalrtGetDeviceInfo`函数使用了ACL设备查询API，在AICPU编译环境下这些API不可用，需要用`#ifndef AICPU_COMPILE`条件编译隔离。
- 修复模式：补全CMakeLists源文件列表 + 增加条件编译宏
- 可审查性：低（链接错误只能在构建时发现）/ 中（条件编译隔离可通过审查发现）
- 审查规则建议：新增库目标时检查是否包含所有被引用的源文件；跨编译环境（Host/AICPU/Device）的代码需确认API可用性，必要时用编译宏隔离
