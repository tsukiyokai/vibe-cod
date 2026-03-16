# Phase 7: Validation Report

## 验证方法

从hcomm和hccl两仓的近期git log中挑选5个真实commit(2 bugfix + 1 refactor from hcomm, 1 feature + 1 bugfix from hccl),
用cann-cpp-conventions.md对每个commit做模拟code review, 记录规则覆盖/缺失/矛盾情况。

## 选取的Commit

| # | 仓库 | Hash | 类型 | 模块 | 描述 |
|---|------|------|------|------|------|
| 1 | hcomm | 9bcb1bdc | bugfix | algorithm/alltoallv | CP算法PCIe链路修复 |
| 2 | hcomm | 0aec21e1 | refactor | platform/ping_mesh | 接口重命名+code review整改 |
| 3 | hcomm | a9fea200 | bugfix | framework/symmetric_memory | 资源释放修复 |
| 4 | hccl | 4a033793 | feature | common/hccl_mc2 | MC2 context API新增 |
| 5 | hccl | deb37cc2 | bugfix | op_common/template/aicpu | A3 task exception处理 |

## 1. 规则覆盖统计

### 被触发的规则(按频次排序)

以下conventions规则在5个commit的review中被触发:

| 规则ID | 名称 | 触发次数 | 遵循/违反 |
|--------|------|----------|-----------|
| P-ID-01 | CHK_RET 100%覆盖 | 5/5 | 4遵循, 1部分 |
| R-1.7 | HcclResult错误码体系 | 5/5 | 5遵循 |
| CROSS-08 | 统一日志宏体系 | 5/5 | 5遵循 |
| P-ID-07 | 日志级别规范 | 4/5 | 4遵循 |
| CROSS-01 | HcclResult统一返回 | 4/5 | 4遵循 |
| CROSS-11 | 命名规范 | 4/5 | 3遵循, 1部分 |
| CROSS-04 | 不使用goto | 4/5 | 3遵循, 1部分(已有goto未改) |
| CROSS-02 | 错误码透传 | 3/5 | 2遵循, 1违反(错误码被覆盖) |
| P-ID-09 | securec安全函数 | 2/5 | 2遵循 |
| CROSS-20 | include guard | 2/5 | 2遵循 |
| CROSS-06 | 智能指针替代裸new | 2/5 | 1遵循, 1违反(malloc管理C struct) |
| R-1.8 | 基础类型别名 | 2/5 | 2遵循 |
| TST-02 | TEST_F唯一测试宏 | 2/5 | 2遵循 |
| TST-04 | EXPECT优先于ASSERT | 2/5 | 2遵循 |
| P-ID-06 | 引用计数去重 | 1/5 | 1遵循 |
| FW-ID-06 | unique_ptr析构序 | 1/5 | 1遵循 |
| ALG-10 | rankSize=1快速路径 | 1/5 | 1遵循 |
| P-BIZ-01 | DMA传输方向策略 | 1/5 | 1遵循(正是bugfix的核心) |
| P-BIZ-02 | Notify三轮握手 | 1/5 | 1遵循 |
| OPC-07 | AICPU Kernel Launch双路径 | 1/5 | 1遵循 |

结论: 核心规则(CHK_RET/HcclResult/日志/命名)覆盖率高, 几乎每个commit都会触发。
业务域规则(DMA方向/Notify握手)只在特定模块触发但非常有用。

### 未被触发的主要规则

以下重要规则在5个commit中未被触发(因commit范围不涉及):
- R-1.10 V1/V2双代架构(仅commit 4涉及weak symbol但非V2分派)
- FW-AP-06 状态机驱动(无FSM改动)
- ALG-02 三层Registry自注册(无新注册)
- OPC-01~06 Selector/Executor/Template相关(无算法选择改动)
- OP-01~OP-20 大部分算子实现规则(commit改动粒度较小)

## 2. 规则缺失(应有但conventions中没有)

以下是从5个commit的review中发现的、conventions中缺失的规则:

### 高优先级(多个commit触发)

GAP-01: struct字段默认初始化一致性
- 触发: commit 4(HcclOpArgs.Init()不完整), commit 5(ScatterOpInfo count/dataType无默认值)
- 规则: 所有POD结构体字段要么全部提供默认值,要么全部不提供,保持一致性。对外暴露的struct应使用Init()或= {}确保零初始化。

GAP-02: const正确性
- 触发: commit 5(CreateScatter的param应为const OpParam*)
- 规则: 只读参数必须标记const。函数参数如果不修改指向的内容,使用const指针/引用。

GAP-03: 函数命名必须准确反映语义
- 触发: commit 5(CreateScatter名称误导, 实际是Fill/Init操作), commit 4(HcclKfcFreeOpArgs暗示释放但还有置null操作)
- 规则: 函数名中Create/Alloc暗示所有权转移, Fill/Init暗示就地修改, Get暗示查询。名称应与实际行为匹配。

GAP-04: 文件末尾换行符
- 触发: commit 1(测试文件), commit 4(新文件)
- 规则: 所有源文件必须以换行符结尾(POSIX规范)。

GAP-05: 参数置null无效(函数参数是局部变量)
- 触发: commit 4(HcclKfcFreeOpArgs中opArgs=nullptr无效, 与BUG-04同模式)
- 规则: free/delete后的指针置null只对调用者可见的引用有效,函数参数是值拷贝,置null不影响调用者。需使用二级指针或引用。

### 中优先级(单个commit触发但具有普适性)

GAP-06: ABI兼容性和公共接口废弃流程
- 触发: commit 2(hrtRaGetDevEidInfoNum签名变更, HccnRpingAddTargetV2删除)
- 规则: 通过dlsym加载的函数签名变更需与.so提供方同步确认。extern "C"公共接口删除前应经过deprecated标记阶段。

GAP-07: 注册到外部系统的数据生命周期
- 触发: commit 5(栈上ScatterOpInfo注册给hcomm后, 函数返回即销毁)
- 规则: 通过HcommRegOpInfo等注册接口传递的数据,必须确认其生命周期覆盖外部系统的使用期。栈变量不适合作为注册数据(除非外部系统在注册时就完成拷贝)。

GAP-08: 析构/清理路径中的容器遍历安全性
- 触发: commit 3(析构函数中range-based for遍历windowMap_同时erase)
- 规则: 禁止在遍历容器时通过调用函数间接修改该容器。析构函数中应先收集key到临时容器再逐个删除,或直接做内联清理。

GAP-09: CHK_PRT_RET不应用于正常流程控制
- 触发: commit 3(CHK_PRT_RET(isSingleRank_, ..., HCCL_SUCCESS)用于正常分支)
- 规则: CHK_PRT_RET/CHK_RET等错误检查宏只用于错误路径。正常的条件分支(如单rank快速返回)应使用普通if语句,避免误导读者以为这是错误处理。

GAP-10: format string类型安全
- 触发: commit 2(HCCL_ERROR中%s对应整数参数), commit 5(同类问题pre-existing)
- 规则: 日志format string的%s/%d/%u等格式符必须与对应参数类型匹配。考虑使用类型安全的日志接口或编译器format检查attribute。

GAP-11: 正则表达式性能
- 触发: commit 2(IsIPv6每次调用编译超长regex)
- 规则: std::regex编译开销大,频繁调用的函数中应使用static local缓存编译结果。

GAP-12: 多步资源分配的回滚完整性
- 触发: commit 3(RegisterInternal中goto MAP_ERROR未清理importAddrs_)
- 规则: 当资源分配涉及多个步骤时,任一步骤失败的清理路径必须覆盖所有已完成步骤。使用RAII或reverse-order清理。

## 3. 规则矛盾/不可操作

### 矛盾

CON-01: P-ID-01(CHK_RET 100%) vs 析构路径best-effort
- 场景: commit 3中析构函数遍历调用DeregisterSymmetricMem, CHK_RET会在错误时中断清理
- 建议: P-ID-01应增加例外说明: "析构/清理路径中CHK_RET不适用,应使用(void)忽略或记录后继续"

CON-02: P-ID-01(CHK_RET 100%) vs extern "C"函数返回unsigned int
- 场景: commit 5中HcclLaunchAicpuKernel返回unsigned int, 无法使用CHK_RET(类型不匹配)
- 建议: 增加说明: "extern C入口函数因返回类型限制不适用CHK_RET,需手动if检查"

CON-03: CROSS-06(智能指针) vs C struct场景
- 场景: commit 4中HcclOpArgs是C ABI结构体,用malloc/free管理
- 建议: CROSS-06应增加: "C ABI接口层的POD struct可使用malloc/free, C++内部代码使用智能指针"

CON-04: P-ID-09(securec) vs C++ stream
- 场景: commit 2中删除snprintf_s改用ostringstream, 后者更安全
- 建议: P-ID-09应修改为: "C接口层使用securec, C++代码优先使用stream/string操作"

### 不可操作

INOP-01: ALG-01描述"无一处裸调用"过于绝对
- 场景: commit 1中std::copy等void返回值的STL函数不需要也不能CHK_RET
- 建议: 限定为"所有返回HcclResult的函数调用"

INOP-02: P-BIZ-01(DMA方向策略)抽象层级与代码不匹配
- 场景: commit 1中通过IsSpInlineReduce()判断链路类型, 而非conventions描述的P2P/HCCS枚举
- 建议: 补充代码层面的判断方法, 而不仅仅是概念描述

## 4. 各Commit Review发现的代码问题汇总

### Commit 1 (9bcb1bdc, hcomm bugfix)
- [低] 测试文件末尾缺换行符
- [低] 7个ST测试高度重复,可TEST_P参数化
- [低] std::copy目标容器未显式校验大小

### Commit 2 (0aec21e1, hcomm refactor)
- [严重] ping_mesh.cc: HCCL_ERROR格式串%d只有1个占位符但传2个参数
- [严重] ping_mesh.cc: deviceLogicId_(s32)使用%s格式符, undefined behavior
- [中] GetUbToken二次调用返回未初始化token值(static once-init后不填充输出参数)
- [中] HccnSupportedAndGetphyid返回值被覆盖为HCCL_E_NOT_SUPPORT
- [低] IsIPv6正则表达式无static缓存
- [低] adapter_hccp.cc日志函数名标签不一致

### Commit 3 (a9fea200, hcomm bugfix)
- [严重] 析构函数range-based for遍历windowMap_同时erase,未定义行为
- [严重] DeregisterSymmetricMem refCount>1时释放共享paHandle,可能影响其他引用者
- [中] RegisterInternal goto MAP_ERROR未清理importAddrs_
- [中] CHK_PRT_RET(aclrtFreePhysical)失败时跳过后续清理
- [低] RegisterSymmetricMem单rank路径未检查devWin空指针
- [低] FindSymmetricWindow单rank返回NOT_FOUND与多rank行为不一致

### Commit 4 (4a033793, hccl feature)
- [中] HcclKfcFreeOpArgs中opArgs=nullptr只修改局部变量(与BUG-04同模式)
- [中] HcclCreateOpResCtx weak函数nullptr时返回SUCCESS但opResCtx未初始化
- [中] HcclOpArgs Init()未初始化所有字段(algConfig/commEngine)
- [低] extern "C"块内HcclOpArgs含C++成员函数Init()
- [低] 新文件末尾缺换行符

### Commit 5 (deb37cc2, hccl bugfix)
- [中] ScatterOpInfo count/dataType无默认值,其他字段有默认值(不一致)
- [中] CreateScatter param参数缺const修饰
- [低] 函数名CreateScatter与实际语义(Fill/Init)不匹配
- [低] GetScatterOpInfo中pre-existing %s对应整数参数bug未修复

## 5. Conventions改进建议优先级

根据验证结果,建议对cann-cpp-conventions.md做以下修改:

### 必须修改(规则矛盾/不可操作)
1. P-ID-01: 增加析构路径和extern "C"返回类型的例外说明
2. CROSS-06: 增加C ABI场景的malloc/free例外
3. P-ID-09: 区分C接口层(securec)和C++代码(stream/string)

### 建议新增(高频缺失)
4. GAP-01: struct字段默认初始化一致性
5. GAP-02: const正确性
6. GAP-05: 参数置null无效的反模式(已有BUG-04但未提升为通用规则)
7. GAP-10: format string类型安全
8. GAP-08: 析构路径容器遍历安全

### 可选新增(低频但有价值)
9. GAP-04: 文件末尾换行符
10. GAP-06: ABI兼容性规则
11. GAP-07: 注册数据生命周期
12. GAP-09: CHK_PRT_RET不用于正常流程
