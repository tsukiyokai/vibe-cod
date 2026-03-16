# Phase 1.2: hccl Git History Mining

## Repository Overview

- 总 commit 数: 175
- 时间范围: 2025-11-15 ~ 2026-03-07 (约 3.5 个月)
- 0 个 revert commit (零回退率)
- 0 个 TODO/FIXME/HACK 标记 (代码中无遗留标记)

## 1.2.1 高频修改文件 (Top 30 Hotspots)

### 按顶层模块修改分布

| 模块 | file-touch 次数 | 占比 |
|---|---|---|
| src/ops | 912 | 82.0% |
| src/common | 51 | 4.6% |
| test/st | ~150 | 13.5% |

src/ops 占绝对主导地位 (82%)，hccl 的变更几乎全部围绕算子实现。

### 按算子聚合修改频次

| file-touch | 算子 | 备注 |
|---|---|---|
| 136 | scatter | 修改频率最高，近期重点开发 |
| 129 | op_common | 算子公共框架 |
| 103 | reduce_scatter | 第三活跃 |
| 91 | all_reduce | 核心算子 |
| 86 | reduce | |
| 84 | all_to_all_v | |
| 78 | all_gather | |
| 51 | broadcast | |
| 18 | aicpu | AICPU 内核启动 |
| 17 | all_gather_v | 可变长版本 |
| 16 | reduce_scatter_v | 可变长版本 |
| 15 | channel | 通道管理 |
| 14 | send | 点对点发送 |
| 14 | recv | 点对点接收 |
| 13 | topo | 拓扑信息 |
| 11 | batch_send_recv | 批量收发 |

发现:
- scatter 算子修改频率出乎意料地最高 (136 次)，可能是近期新增或重大重构
- op_common (129 次) 承载了跨算子的公共逻辑变更
- 可变长算子 (all_gather_v, reduce_scatter_v) 修改较少，说明其接口较为稳定

### Top 15 热点文件

| 修改次数 | 增行 | 删行 | 文件 |
|---|---|---|---|
| 20 | 1116 | 259 | src/ops/scatter/scatter_op.cc |
| 16 | 1037 | 173 | test/st/algorithm/utils/src/hccl_proxy/hccl_stub.cc |
| 14 | 353 | 139 | src/ops/inc/alg_param.h |
| 12 | 194 | 194 | src/ops/aicpu/kernel_launch.cc |
| 8 | 934 | 176 | src/ops/topo/topo.cc |
| 8 | 988 | 124 | src/common/alg_env_config.cc |
| 8 | 270 | 26 | src/ops/scatter/algo/scatter_executor_base.cc |
| 7 | 1210 | 199 | src/ops/op_common/op_common.cc |
| 7 | 184 | 184 | src/ops/channel/channel.cc |
| 6 | 177 | 50 | test/st/.../sim_communicator.cc |
| 5 | 493 | 5 | src/ops/op_common/executor/channel/channel.cc |
| 5 | 440 | 9 | src/ops/reduce_scatter/executor/ins_reduce_scatter_parallel_executor.cc |
| 5 | 393 | 23 | src/ops/scatter/algo/scatter_ring_executor.cc |
| 5 | 356 | 56 | src/ops/op_common/template/aicpu/kernel_launch.cc |
| 5 | 330 | 14 | src/ops/reduce_scatter/template/ccu/ccu_temp_reduce_scatter_nhr_1D_mem2mem.cc |

关键特征:
- scatter_op.cc (20 次修改，1116 行新增) 是最活跃文件
- alg_param.h (14 次) 是公共参数定义，频繁被各算子修改引发联动
- aicpu/kernel_launch.cc 增删完全对等 (194/194)，说明经历了大规模重写
- hccl_stub.cc (ST 测试桩) 紧随其后，说明测试桩需要跟随产品代码同步更新
- 增删比相比 hcomm 更均衡，说明 hccl 有更多的重构和代码替换

## 1.2.2 Revert Commit 分析

hccl 仓库 0 个 revert commit，零回退率。

可能原因:
- hccl 仓库较新/较小 (175 commit vs hcomm 428)，代码成熟度可能不同
- 可能采用了更严格的 MR review 流程
- 也可能 revert 通过其他方式 (如新 commit 覆盖) 而非 git revert

## 1.2.3 Bug-fix Commit 分析

匹配 fix/bug/defect/issue 的 commit: 112 个 (占 64%)

### 按算子统计 bugfix 文件变更

| file-touch | 算子 | bug 占比 |
|---|---|---|
| 129 | op_common | 100% (全部修改都与 bug 相关) |
| 103 | reduce_scatter | 100% |
| 91 | all_reduce | 100% |
| 86 | reduce | 100% |
| 84 | all_to_all_v | 100% |
| 78 | all_gather | 100% |
| 59 | scatter | 43% (136 总修改中 59 与 bug 相关) |
| 51 | broadcast | 100% |

注意: 由于 hccl 大部分 commit message 包含 "fix" 相关词，bugfix 比例被高估。
scatter 的比例最低 (43%)，因为它有大量新 feature commit。

### 典型 bugfix 模式

1. CCU 模板修复:
   - 992fd5b: ccu nhr template bugfix (reduce_scatter)
   - f623ff7: ccu mode reducescatter bugfix (跨 4 文件)
   - 80a7072: ccu模式scatter_mesh1d 地址设置时机

2. 信号/同步修复:
   - b779699: [Bugfix] fix missing signals (channel.cc)

3. 硬件适配修复:
   - 3e9fbbd: fix a3 bug (topo + scatter)

4. 可变长算子修复:
   - cf9a331: bugfix for allgatherv (单文件单行修复)

### Bug 热点文件

| 次数 | 文件 |
|---|---|
| 同 hotspot | scatter_op.cc |
| 同 hotspot | op_common.cc |
| 同 hotspot | alg_param.h |
| 5 | reduce_scatter/template/ccu/ccu_temp_reduce_scatter_nhr_1D_mem2mem.cc |
| 4 | reduce_scatter/selector/reduce_scatter_auto_selector.cc |
| 4 | all_reduce/selector/all_reduce_auto_selector.cc |

发现: selector (算法选择器) 和 CCU 模板是 bug 的两大集中区。

## 1.2.4 重构 Commit 分析

仅 1 个重构 commit:
- fcc7f8f: "refactor(send&recv): optimize code comments and testcase"

与 hcomm 一样，hccl 也处于功能建设期而非重构期。

## 1.2.5 综合发现

### 与 hcomm 对比

| 维度 | hcomm | hccl |
|---|---|---|
| commit 数 | 428 | 175 |
| revert 数 | 4 (0.9%) | 0 |
| bugfix 占比 | ~72% | ~64% |
| 重构 commit | 3 | 1 |
| 增删比 | 极度偏增 | 较均衡 |
| TODO 标记 | 17 (全在 next/) | 0 |
| 热点集中度 | communicator 模块 | scatter 算子 |

### hccl 特有的模式

1. scatter 算子异常活跃: 20 次修改 (其他算子 4-8 次)，可能是近期重点重构的算子
2. alg_param.h 是变更放大器: 作为公共参数定义头文件，每次新增算法参数都要修改它，引发广泛的联动编译
3. 测试桩同步更新: hccl_stub.cc (16 次修改) 紧跟产品代码，说明测试桩维护成本高
4. 零 TODO/FIXME: 代码洁净度高于 hcomm

### Commit 消息风格

与 hcomm 类似的不统一风格:
- [Docs] x7, [Fix] x4, [Build] x4, [Feat] x3, [Sync] x1, [Bugfix] x1
- 大部分 commit 无标签前缀

### 重点关注区域 (供后续 Phase 3 深入分析)

| 优先级 | 文件/目录 | 原因 |
|---|---|---|
| P0 | scatter/ | 最高修改频率，近期重点 |
| P0 | op_common/ | 公共框架，bug 密集 |
| P1 | alg_param.h | 公共参数，变更放大器 |
| P1 | reduce_scatter/ selector + CCU 模板 | bug 集中区 |
| P2 | all_reduce/ selector | 核心算子的算法选择 |
| P2 | aicpu/kernel_launch.cc | 经历大规模重写 |
