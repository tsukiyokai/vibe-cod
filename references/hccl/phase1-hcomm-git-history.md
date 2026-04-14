# Phase 1.1: hcomm Git History Mining

## Repository Overview

- 总 commit 数: 428
- 时间范围: 2025-12-27 ~ 2026-02-27 (约 2 个月)
- C++ 文件 (*.cc, *.h, *.cpp, *.hpp) 变更频次总计约 7500+ file-touches

## 1.1.1 高频修改文件 (Top 50 Hotspots)

### 按层统计修改占比

| 层 | file-touch 次数 | 占比 |
|---|---|---|
| legacy | 2262 | 39.7% |
| algorithm | 1236 | 16.4% |
| framework | 1135 | 15.1% |
| platform | 746 | 9.9% |
| common | 63 | 0.8% |
| other (test/include/pkg_inc) | 1357 | 18.0% |

发现: legacy 层虽然是"遗留代码"，但修改频率最高 (近 40%)，说明仍在大量活跃维护。

### Top 15 热点文件

| 修改次数 | 增行 | 删行 | 文件 |
|---|---|---|---|
| 30 | 9366 | 214 | src/framework/communicator/impl/hccl_communicator_host.cc |
| 28 | 3937 | 148 | src/legacy/framework/communicator/communicator_impl.cc |
| 18 | 6055 | 505 | src/framework/device/framework/aicpu_communicator.cc |
| 15 | 595 | 10 | src/legacy/framework/communicator/communicator_impl.h |
| 14 | 4830 | 135 | src/framework/op_base/src/op_base.cc |
| 14 | 1171 | 29 | src/framework/communicator/impl/hccl_communicator.h |
| 14 | 476 | 12 | src/framework/inc/hccl_comm_pub.h |
| 11 | 3035 | 133 | src/legacy/framework/entrance/op_base/op_base_v2.cc |
| 11 | 1652 | 12 | src/framework/communicator/impl/hccl_communicator_device.cc |
| 11 | 397 | 10 | src/framework/communicator/hccl_comm_host.cc |
| 10 | 2274 | 234 | src/platform/common/externalinput.cc |
| 10 | 1333 | 235 | src/legacy/framework/dfx/task_exception/task_exception_handler.cpp |
| 10 | 4154 | 78 | src/framework/hcom/hcom.cc |
| 10 | 1588 | 12 | src/framework/communicator/hccl_comm.cc |
| 10 | 759 | 36 | src/algorithm/impl/operator/all_reduce_operator.cc |

关键特征:
- hccl_communicator_host.cc 以 30 次修改、9366 行新增遥遥领先，是整个仓库最活跃的文件
- 增删比极度不对称 (增 >> 删)，说明项目处于快速增长期而非重构收敛期
- communicator 相关文件占据 top 15 中的 8 个位置，是核心变更热区

### framework 层内部热点

| file-touch 次数 | 子模块 |
|---|---|
| 393 | next (新框架) |
| 309 | communicator |
| 164 | device |
| 126 | common |
| 46 | cluster_maintenance |
| 32 | hcom |
| 29 | inc |
| 26 | op_base |

发现: next 子模块 (新框架) 修改频率最高，说明新架构正在积极建设中。

### legacy 层内部热点

| file-touch 次数 | 子模块 |
|---|---|
| 1115 | service/collective (集合通信算法选择/执行/模板) |
| 183 | unified_platform/ccu |
| 173 | unified_platform/resource |
| 147 | framework/communicator |
| 144 | framework/topo |
| 78 | framework/resource_manager |
| 65 | framework/dfx |

发现: legacy/service/collective 占 legacy 修改量的近一半，这里是旧版算法选择和执行的核心。

### 热点目录 Top 10 (最细粒度)

| file-touch 次数 | 目录 |
|---|---|
| 284 | src/legacy/service/collective/alg/coll_alg_factory/alg_ccu_context |
| 182 | src/legacy/service/collective/alg/coll_alg_factory/alg_template/ccu_alg_template |
| 144 | src/legacy/service/collective/alg/coll_alg_factory/alg_template/ins_alg_template |
| 123 | src/legacy/service/collective/alg/coll_alg_factory/alg_executor/ins_alg_executor |
| 120 | src/legacy/service/collective/alg/selector |
| 120 | src/algorithm/base/alg_aiv_template |
| 98 | src/algorithm/impl/coll_executor/coll_all_reduce |
| 91 | src/algorithm/impl/coll_executor/coll_reduce_scatter |
| 80 | src/framework/communicator/impl |
| 76 | test/st/algorithm/utils/checker/semantics_check |

发现:
- CCU context 和 CCU/INS 模板是最活跃的子目录，说明设备侧算法模板是变更核心
- allreduce 和 reduce_scatter 是最活跃的算子实现
- 语义检查测试工具 (semantics_check) 也在积极建设

### commit 规模分布

| 修改文件数 | commit 数 |
|---|---|
| 1 | 66 |
| 2 | 43 |
| 3 | 25 |
| 4 | 19 |
| 5-10 | 36 |
| 11-20 | 15 |
| 21-50 | 12 |
| 51-100 | 8 |
| 100+ | 5 |

中位数约为 2-3 个文件/commit，但存在少量巨型 commit (最大 2629 文件)，可能是初始开源或大规模重构。

## 1.1.2 Revert Commit 分析

总计仅 4 个 revert commit，revert 率极低 (4/428 = 0.9%)。

### Revert 1: 910_95 host sdma (f0666214)

- 原始 commit: 0e5be69c "910_95支持host侧本地sdma拷贝和本地reduce操作"
- 影响: 仅 1 个文件 (hccl_primitive_local.cc)，删除 51 行
- 分析: 新硬件特性支持被回退，可能因为功能不稳定或下游依赖未就绪

### Revert 2+3: loopnum = 128 (72cdf80e → 0c3be05f)

- 原始 commit: "fix loopnum = 128"
- 第一次 revert (72cdf80e): 回退该 fix
- 第二次 revert (0c3be05f): 恢复该 fix (Revert of Revert)
- 影响: CCU microcode (ccu_microcode.h) + UT (ut_ccu_dfx.cpp)
- 分析: CCU microcode 的循环次数修复经历了 fix → revert → re-apply 的过程，
  说明 CCU 相关改动的影响范围不易评估，需要更充分的测试覆盖

### Revert 4: offline compile (753ba8c2)

- 影响: 24 个文件，主要是 cmake 构建脚本
- 分析: 构建系统改动被回退，说明离线编译支持的方案需要重新设计

反模式总结:
- CCU/设备侧代码的 loopnum 修改经历了 fix-revert-revert 循环，提示此类改动需要更严格的预验证
- 硬件特性支持 (sdma) 的 revert 提示: 新硬件特性应有 feature flag 保护或分阶段合入
- 构建系统改动影响面广，需要完整的 CI 验证后再合入

## 1.1.3 Bug-fix Commit 分析

匹配 fix/bug/defect/issue 关键词的 commit: 310 个 (占总 commit 的 72.4%)

注意: 这个数字偏高，因为许多 merge commit 消息中包含 "issue" 字样；
明确标记为 [Bugfix] 或 【Bugfix】 的 commit 为 28 个。

### 按模块统计 bugfix 文件变更

| 模块 | bug-related file-touch | 占该模块总修改的比例 |
|---|---|---|
| legacy | 874 | 38.6% (874/2262) |
| platform | 246 | 33.0% (246/746) |
| framework | 218 | 19.2% (218/1135) |
| algorithm | 151 | 12.2% (151/1236) |

发现:
- legacy 和 platform 的 bug 密度最高
- algorithm 层 bug 密度最低，可能因为有较完善的 ST 语义检查

### 按二级模块统计 bugfix 热点

| file-touch | 模块 | 备注 |
|---|---|---|
| 603 | legacy/service | 旧版集合通信算法，bug 最密集 |
| 202 | legacy/framework | 旧版通信器、拓扑等 |
| 143 | platform/hccp | HCCP 协议栈 |
| 106 | algorithm/impl | 新版算法实现 |
| 78 | framework/next | 新框架 |
| 58 | framework/communicator | 通信器 |
| 54 | legacy/unified_platform | 旧版统一平台 |

### bugfix 文件热点 Top 10

| 次数 | 文件 |
|---|---|
| 17 | src/legacy/framework/communicator/communicator_impl.cc |
| 15 | src/framework/communicator/impl/hccl_communicator_host.cc |
| 11 | src/framework/device/framework/aicpu_communicator.cc |
| 9 | src/legacy/framework/communicator/communicator_impl.h |
| 7 | src/legacy/service/collective/alg/interface/host/coll_alg_component.cc |
| 6 | src/legacy/framework/entrance/op_base/op_base_v2.cc |
| 6 | src/legacy/framework/dfx/task_exception/task_exception_handler.cpp |
| 6 | src/framework/hcom/hcom.cc |
| 5 | src/platform/ping_mesh/ping_mesh.cc |
| 5 | src/platform/hccp/inc/network/hccp_ctx.h |

发现: communicator_impl.cc 在 bugfix 中被修改 17 次 (总 28 次中的 60.7%)，是 bug 密度最高的单个文件。

### Bug 分类统计

| 类别 | 数量 | 典型 commit |
|---|---|---|
| 日志/DFX 相关 | 20 | 日志格式、打印内容整改、DFX 功能修复 |
| AICPU/AIV 设备侧 | 16 | 重执行时序、AIV bugfix、AICPU 展开模式 |
| CCU 相关 | 11 | CCU fallback、algname cache、loopnum |
| 算子相关 | 11 | alltoallv pipeline、broadcast bigdata、allreduce |
| 内存/资源 | 8 | 资源释放不完全、对称内存指针判空、buffer 管理 |
| 重执行 (Retry) | 4 | 多线程时序、超时时间、约束故障上报 |

### 明确 [Bugfix] 标记的 commit 模式分析

7 个【Bugfix】标记的 commit 涉及的典型缺陷:

1. 重执行 (Retry) 相关 (3 个):
   - 75659d24: 重执行多线程下的时序问题 → 竞态条件
   - e025b6c5: 重执行失败无约束打印 → 错误路径缺少日志
   - 6b354394: 借轨条件判断 & 重执行超时时间 → 条件判断错误

2. 并发/锁相关 (1 个):
   - 1535b1c4: 读写锁改为基于原子变量的实现 → 锁粒度过粗导致性能问题

3. 接口一致性 (1 个):
   - 2a4ec69d: 函数参数名定义与声明不一致 → 头文件和实现不同步

4. 恢复路径 (1 个):
   - 994390df: Step快恢失败问题 → 故障恢复路径未充分测试

5. 算法正确性 (1 个):
   - 9bcb1bdc: CP (Continuous Pipeline) 算法bug → 算法实现缺陷

反模式总结:
- 重执行/故障恢复是高频 bug 来源，这类代码路径容易被忽视测试
- 多线程时序问题反复出现，说明并发控制是薄弱环节
- 头文件和实现不同步的问题可用编译器 warning 预防

## 1.1.4 重构 Commit 分析

匹配 refactor/rework/redesign/cleanup 的 commit 仅 3 个:
- 55ef3df7: "Add ACL version retrieval functions and refactor ParseCannVersion"
- 4dd79ef6: merge issue-EI0008
- 2c6992bc: merge ccu_dfx

重构 commit 极少，说明项目处于功能快速叠加阶段，尚未进入系统性重构期。

## 1.1.5 综合发现

### Commit 消息风格

使用的标签不统一，存在多种风格:
- 英文方括号: [Fix], [Docs], [Build], [Feature], [Feat], [Update]
- 中文方括号: 【Bugfix】
- 无标签: 大多数 commit 无统一标签前缀

### 代码注释中的 TODO

代码中共 17 个 TODO 标记，全部集中在 framework/next/ 目录 (新框架):
- 环境变量获取方式待定
- Channel 赋值逻辑未完成
- 异常处理待补充
- 内存 handle 管理待清理
- 握手消息格式待定义

无 FIXME 和 HACK 标记，说明团队对代码质量标记的使用较保守。

### 演进趋势

1. 双代架构并行: legacy 和 framework/next 同时活跃修改，新旧框架共存
2. 快速增长期: 增行远多于删行，commit 以新增功能和 bugfix 为主
3. CCU/AIV 设备侧是变更焦点: alg_ccu_context 和 alg_aiv_template 是最热门目录
4. 通信器 (communicator) 是 bug 密集区: 新旧通信器实现各自都是 bug 热点
5. 测试工具在同步建设: semantics_check 测试框架活跃度排 top 10

### 重点关注区域 (供后续 Phase 2 深入分析)

| 优先级 | 文件/目录 | 原因 |
|---|---|---|
| P0 | communicator_impl.cc / hccl_communicator_host.cc | 最高修改频率 + 最高 bug 密度 |
| P0 | aicpu_communicator.cc | 设备侧通信器，重执行 bug 集中 |
| P1 | legacy/service/collective/alg/ | 最热门目录，旧版算法核心 |
| P1 | framework/next/ | 新框架建设中，大量 TODO |
| P2 | platform/hccp/ | 协议栈，bug 密度第三 |
| P2 | task_exception_handler.cpp | DFX 异常处理，修改频繁 |
