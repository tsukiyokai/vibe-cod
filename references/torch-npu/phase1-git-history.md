# torch_npu Git History Analysis

Generated: 2026-04-12
Repository: torch_npu (Ascend PyTorch Adapter)
Branches: all (--all --no-merges)
Total commits analyzed: 28,222

## 1. Category Distribution

| Category     | Count  | Pct    |
|:-------------|-------:|-------:|
| OTHER        | 10,866 | 38.5%  |
| INFRA        | 10,604 | 37.6%  |
| BUGFIX       |  3,401 | 12.0%  |
| FEATURE      |  1,698 |  6.0%  |
| REFACTOR     |    731 |  2.6%  |
| DOC_TEST     |    689 |  2.4%  |
| OPTIMIZATION |    132 |  0.5%  |
| REVERT       |    101 |  0.4%  |

Classification methodology:
- Keyword-based with priority ordering (REVERT > INFRA > DOC_TEST > BUGFIX > OPTIMIZATION > REFACTOR > FEATURE)
- Handles conventional commits (feat/fix/refactor), bracket prefix ([fix]/【fix】), and Chinese keywords (修复/新增/优化)
- OTHER includes commits with no clear category marker in the subject line
- INFRA is dominated by automated "Update op_plugin commit id" (7,651) and "Update torchair commit id" (696)

## 2. Year-by-Year Commit Trend

| Year | Total  | FEATURE | BUGFIX | INFRA | REFACTOR | OPTIM | REVERT | DOC_TEST | OTHER |
|:-----|-------:|--------:|-------:|------:|---------:|------:|-------:|---------:|------:|
| 2021 |    344 |      20 |      2 |   213 |        3 |     0 |      0 |       35 |    71 |
| 2022 |  1,795 |     334 |    279 |   236 |       68 |    25 |      5 |      117 |   731 |
| 2023 |  4,961 |     554 |    887 |   782 |      242 |    74 |     45 |       90 | 2,287 |
| 2024 |  7,224 |      14 |    526 | 3,019 |       43 |    13 |      1 |      125 | 3,483 |
| 2025 | 10,663 |     415 |    895 | 5,173 |      295 |    10 |     15 |      144 | 3,716 |
| 2026 |  3,235 |     361 |    812 | 1,181 |       80 |    10 |     35 |      178 |   578 |

Key observations:
- 2024 anomaly: FEATURE dropped to 14 while INFRA surged to 3,019. The year was dominated by op_plugin/torchair synchronization, not feature development.
- BUGFIX sustained at 500-900/year from 2022 onward, indicating continuous quality investment.
- REVERT peaked in 2023 (45) alongside the highest REFACTOR count (242), reflecting aggressive code reorganization.
- 2026 partial year (Q1) already shows 812 BUGFIX, 361 FEATURE, projecting to the most active year.

## 3. Top-30 Most Frequently Modified Files

| Rank | Touches | File |
|-----:|--------:|:-----|
|    1 |   7,220 | third_party/op-plugin |
|    2 |   2,286 | third_party/torchair/torchair |
|    3 |   1,472 | torch_npu/csrc/distributed/ProcessGroupHCCL.cpp |
|    4 |     919 | torch_npu/csrc/core/npu/NPUCachingAllocator.cpp |
|    5 |     788 | torch_npu/csrc/npu/Module.cpp |
|    6 |     707 | test/torch_npu_schema.json |
|    7 |     689 | torch_npu/__init__.py |
|    8 |     631 | torch_npu/csrc/aten/npu_native_functions.yaml |
|    9 |     586 | torch_npu/csrc/core/npu/interface/AclInterface.cpp |
|   10 |     563 | torch_npu/csrc/distributed/ProcessGroupHCCL.hpp |
|   11 |     540 | torch_npu/csrc/core/npu/NPUQueue.cpp |
|   12 |     473 | README.zh.md |
|   13 |     452 | torch_npu/csrc/core/npu/register/OptionsManager.cpp |
|   14 |     434 | torch_npu/csrc/framework/OpParamMaker.cpp |
|   15 |     429 | torch_npu/csrc/core/npu/sys_ctrl/npu_sys_ctrl.cpp |
|   16 |     406 | torch_npu/csrc/core/npu/interface/AclInterface.h |
|   17 |     406 | test/unsupported_test_cases/.pytorch-disabled-tests.json |
|   18 |     404 | setup.py |
|   19 |     396 | torch_npu/csrc/core/npu/NPUStream.cpp |
|   20 |     385 | torch_npu/npu/__init__.py |
|   21 |     335 | torch_npu/csrc/framework/OpCommand.cpp |
|   22 |     334 | torch_npu/csrc/core/npu/register/OptionsManager.h |
|   23 |     327 | third_party/acl/inc/acl/acl_rt.h |
|   24 |     308 | test/allowlist_for_publicAPI.json |
|   25 |     308 | patch/pytorch1.5.0_npu.patch |
|   26 |     294 | torch_npu/contrib/transfer_to_npu.py |
|   27 |     275 | torch_npu/csrc/core/npu/NPUFunctions.cpp |
|   28 |     270 | torch_npu/npu/utils.py |
|   29 |     266 | torch_npu/csrc/InitNpuBindings.cpp |
|   30 |     260 | torch_npu/csrc/core/npu/NPUCachingAllocator.h |

Hotspot clusters:
- Distributed/HCCL: ProcessGroupHCCL.cpp(1,472) + .hpp(563) = 2,035 total touches
- Memory management: NPUCachingAllocator.cpp(919) + .h(260) = 1,179 total touches
- NPU runtime core: AclInterface.cpp(586) + .h(406) + NPUStream(396) + NPUFunctions(275) = 1,663
- Third-party sync: op-plugin(7,220) + torchair(2,286) = 9,506 (mostly automated bumps)

## 4. Bug Density per Module

Raw BUGFIX commit touches per top-level directory:

| Directory    | Bug touches | File count | Density (bugs/file) |
|:-------------|------------:|-----------:|--------------------:|
| torch_npu    |       6,068 |        735 |                8.26 |
| test         |       2,290 |        689 |                3.32 |
| third_party  |         124 |      6,924 |                0.02 |
| docs         |         113 |         16 |                7.06 |
| codegen      |          66 |          1 |               66.00 |
| ci           |          66 |         17 |                3.88 |
| scripts      |          59 |          - |                   - |
| benchmarks   |          40 |         10 |                4.00 |

Notes:
- codegen density is artificially high due to single-file directory structure
- torch_npu at 8.26 bugs/file reflects the core codebase complexity
- third_party at 0.02 is expected (externally maintained, synced via automation)
- docs directory "bugs" are mostly content corrections, not code bugs

## 5. REVERT Commits (101 total)

### 5.1 REVERT Clusters (by original subject)

| Original subject | Revert count | Branches affected |
|:-----------------|-------------:|------------------:|
| revert removing the libhccl dependency | 9 | multi-version |
| Revert "memory optimaze" | 9 | multi-version |
| revent use the storage for record stream ... in ProcessGroupHCCL | 9 | multi-version |
| Revert "Correcte Time calculate-func" | 7 | multi-version |
| Revert "Second-work flow must get more time-slot" | 7 | multi-version |
| Revert "[fix] set core num to hccl thread when calling hccl operators" | 6 | multi-version |
| Revert AsyncCompile before register | 6 | multi-version |
| Revert "Combine format_contiguous with format_contiguous_add_copy_optimize" | 6 | multi-version |
| Revert export_stacks from Profiling | 4 | multi-version |

Most reverts are cherry-picked across multiple version branches, which inflates the count.
Unique revert events (deduplicated by subject): ~30.

### 5.2 Revert-prone Areas

- ProcessGroupHCCL (distributed): 3 distinct revert events
- Memory management (NPUCachingAllocator, record stream): 2 distinct events
- Inductor/codegen: 3 distinct events
- Profiler: 2 distinct events
- Op framework (format_contiguous, IndexPut): 2 distinct events

## 6. Evolution Direction

### 6.1 Module Activity by Year (file touches)

| Module       | 2021   | 2022   | 2023    | 2024    | 2025    | 2026(Q1) | Trend      |
|:-------------|-------:|-------:|--------:|--------:|--------:|---------:|:-----------|
| torch_npu    |      - |  4,736 |  15,056 |  11,350 |  14,428 |    4,604 | GROWING    |
| test         |  1,050 |  2,066 |  10,385 |   5,637 |   3,690 |    2,127 | STABLE     |
| third_party  |      - |    178 |   1,076 |   3,065 |   5,907 |    1,442 | GROWING    |
| codegen      |      - |      - |     838 |     197 |     116 |        - | SHRINKING  |
| docs         |    415 |    394 |     549 |       - |     951 |    2,007 | GROWING    |
| ci           |      - |      - |     141 |     203 |       - |        - | STABLE     |
| pytorch1.5.0 |  2,602 |  1,085 |       - |       - |       - |        - | REMOVED    |
| pytorch1.8.1 |  1,858 |    950 |       - |       - |       - |        - | REMOVED    |
| test_upstream |      - |      - |       - |       - |       - |      666 | NEW (2026) |
| torchnpugen  |      - |      - |       - |       - |       - |      286 | NEW (2026) |

### 6.2 Structural Evolution

Phase 1 (2021-2022): Fragmentation
- Separate directories per PyTorch version (pytorch1.5.0, pytorch1.8.1, src)
- Heavy patching approach (patch/ directory active)

Phase 2 (2022-2023): Consolidation
- Unified into torch_npu/ directory structure
- Largest REFACTOR year (242 commits in 2023)
- codegen peaked (838 touches) then declined

Phase 3 (2024-2025): Automation and Externalization
- op_plugin and torchair externalized to third_party (5,907 touches in 2025)
- INFRA automation dominated (5,173 INFRA commits in 2025)
- FEATURE development dipped in 2024, recovered in 2025

Phase 4 (2026): Diversification (emerging)
- New modules: test_upstream (upstream test alignment), torchnpugen (code generation)
- docs activity surging (2,007 touches in Q1 alone)
- Highest REVERT rate (35 in Q1), suggesting rapid iteration with rollbacks

---

## BUGFIX+REVERT Index (machine-parseable)

Format: hash | category | summary

<!-- BEGIN INDEX -->
c86a6dda3 | BUGFIX | fix(distributed): batch_isend_irecv补录tensors[1..n]的recordStream
cb3c1e95b | BUGFIX | index on fix/batch-isend-irecv-recordstream: 0e93c91c3 [profiler-2.6.0]修复动态Profiling采部分rank无法解析&解析打屏问题
1ddb00b4b | REVERT | Revert "[inductor] 修复Triton Kernel Codegen中Contiguous Reduction判断不一致问题"
7deab3a5e | REVERT | revert removing the libhccl dependency
fe710e921 | REVERT | revert removing the libhccl dependency
d62c20b4e | REVERT | revert removing the libhccl dependency
1a5666577 | REVERT | revert removing the libhccl dependency
82b8aff3a | REVERT | revert removing the libhccl dependency
18b748325 | REVERT | revert removing the libhccl dependency
7a1a37da5 | REVERT | revert removing the libhccl dependency
3eb2cfdc0 | REVERT | revert removing the libhccl dependency
971d7e602 | REVERT | revert removing the libhccl dependency
ec0bf4010 | BUGFIX | 【fix】fix A3 inductor flags bug
b70ee2d8f | BUGFIX | fix dispatch log
e7c19a6a0 | BUGFIX | fix dispatch log
8cb23d098 | BUGFIX | fix dispatch log
d99df75fb | BUGFIX | fix dispatch log
e68812a07 | BUGFIX | fix dispatch log
916426933 | BUGFIX | fix: fix task queue aclgraph bug
8147d1150 | BUGFIX | fix: fix task queue aclgraph bug
9eb868378 | BUGFIX | fix: fix task queue aclgraph bug
7d86011b9 | BUGFIX | fix: fix task queue aclgraph bug
30cfde000 | BUGFIX | fix: fix task queue aclgraph bug
2b31be7f8 | BUGFIX | fix: fix task queue aclgraph bug
f83b2829d | BUGFIX | fix: fix task queue aclgraph bug
c86f48f9d | BUGFIX | fix: fix task queue aclgraph bug
680435630 | BUGFIX | fix: fix task queue aclgraph bug
8f6a67abf | BUGFIX | [inductor][check accuracy][v2.7.1-26.0.0] fix cat tracing issue for accuracy compare.
048efeeee | BUGFIX | [v2.10.0-26.0.0][bugfix]GetDeviceInfo do not support A5
2e5b8909c | BUGFIX | [v2.9.0-26.0.0][bugfix]GetDeviceInfo do not support A5
876e3f11f | BUGFIX | [v2.8.0-26.0.0][bugfix]GetDeviceInfo do not support A5
826d9dc6a | BUGFIX | [v2.7.1-26.0.0][bugfix]GetDeviceInfo do not support A5
18b902d9f | BUGFIX | [v2.11.0][bugfix]GetDeviceInfo do not support A5
437c909ea | BUGFIX | [v2.10.0][bugfix]GetDeviceInfo do not support A5
0833cf4ae | BUGFIX | [v2.8.0][bugfix]GetDeviceInfo do not support A5
f03c512ed | BUGFIX | [v2.7.1][bugfix]GetDeviceInfo do not support A5
33c4ffdcc | BUGFIX | [inductor][check accuracy][v2.7.1] fix cat tracing issue for accuracy compare.
5a798156e | BUGFIX | [fix] tensor.clone support uncontiguous tensor
9cea7a3dc | BUGFIX | [fix] tensor.clone support uncontiguous tensor
70c8e58fd | BUGFIX | [fix] tensor.clone support uncontiguous tensor
7be454ddb | BUGFIX | [fix] tensor.clone support uncontiguous tensor
a4bbd71ad | BUGFIX | [fix]pow fp64 fallback
193938062 | BUGFIX | 【fix】Modify the tilling splitting logic to advance the judgment logic
dd34f22e8 | BUGFIX | [fix]meta_check-on_rebuild-npu_tensor
461490b38 | BUGFIX | [fix]meta_check-on_rebuild-npu_tensor
7d7551adf | BUGFIX | [fix]meta_check-on_rebuild-npu_tensor
f20606512 | BUGFIX | [fix]meta_check-on_rebuild-npu_tensor
32bef956f | BUGFIX | [fix]meta_check_on_rebuild-npu_tensor
5cc3d16e5 | BUGFIX | 【fix】add_models_patch
fea7396e4 | BUGFIX | 【fix】add_models_patch
6446a3adb | BUGFIX | [inductor] 修复Triton Kernel Codegen中Contiguous Reduction判断不一致问题
c33d1b59d | BUGFIX | fix empty_like output contiguous tensor in copy h2d/d2h
0fa6abf33 | BUGFIX | fix empty_like output contiguous tensor in copy h2d/d2h
cc5a0166b | BUGFIX | fix empty_like output contiguous tensor in copy h2d/d2h
272747028 | BUGFIX | fix empty_like output contiguous tensor in copy h2d/d2h
b0d7dcb30 | BUGFIX | 【fix】fix empty_like output contiguous tensor in copy h2d/d2h
76a5ee379 | BUGFIX | 【fix】fix empty_like output contiguous tensor in copy h2d/d2h
41b22ff93 | BUGFIX | 【fix】fix empty_like output contiguous tensor in copy h2d/d2h
1a7fbb67e | BUGFIX | 【fix】fix empty_like output contiguous tensor in copy h2d/d2h
5806547ae | BUGFIX | 【fix】fix empty_like output contiguous tensor in copy h2d/d2h
5a2f74df4 | BUGFIX | [fix] tensor.clone support uncontiguous tensor
ec32c3775 | BUGFIX | [fix] tensor.clone support uncontiguous tensor
1ac142ae7 | BUGFIX | [fix] tensor.clone support uncontiguous tensor
bdd0b97fd | BUGFIX | [fix] tensor.clone support uncontiguous tensor
595cc6a98 | BUGFIX | [fix] tensor.clone support uncontiguous tensor
db1de538e | REVERT | [Inductor] revert patch_expr_fits_within_32bit and add support for cat constant
7d08fde3d | BUGFIX | [fix][v2.8.0-26.0.0]fix OrderedSet scope and missing argument
00da0b258 | BUGFIX | [fix][v2.9.0-26.0.0]fix OrderedSet scope and missing argument
747d94038 | BUGFIX | [fix][v2.10.0-26.0.0]fix OrderedSet scope and missing argument
f12196ba3 | BUGFIX | [fix][v2.7.1-26.0.0]fix OrderedSet scope and missing argument
f0f390e0f | BUGFIX | [inductor][lowering_fx]fix A5 ETA/HLLM
fb54a6184 | BUGFIX | 【fix】fix_native_api_md
5034e00f5 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
fd3f902f7 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
d53782e30 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
8c4d58ad5 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
0f59193ef | BUGFIX | [Inductor] fix static mode and cat with reindex case
8826eda65 | BUGFIX | 【fix】fix A3 inductor flags bug
093276fe8 | BUGFIX | 【fix】fix_native_api_md
541e8c6e7 | BUGFIX | 【fix】 env control acc-check during tune
0e5459381 | BUGFIX | 【fix】 env control acc-check during tune
0b72dfd4f | BUGFIX | 【fix】 env control acc-check during tune
c067d12d4 | BUGFIX | [v2.7.1-26.0.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
354be76a7 | BUGFIX | [v2.8.0-26.0.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
7d894245f | BUGFIX | [v2.9.0-26.0.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
aba73bd6b | BUGFIX | [v2.10.0-26.0.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
d2bee375a | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
eedc1b312 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
6cc4abb13 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
03d326b9e | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
71f287c39 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
0a2a808d1 | BUGFIX | fix torch.UntypedStorage with empty tensor on npu
98eec0a0b | BUGFIX | [fix][v2.7.1]Modify the operator's entry into the aclgraph
653bc3e57 | BUGFIX | [inductor]fix dynamicshape cantsplit issue
d78d50b30 | BUGFIX | [v2.10.0-26.0.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
4c87b9f9e | BUGFIX | [v2.9.0-26.0.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
f282447c9 | BUGFIX | [v2.8.0-26.0.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
7ab606761 | BUGFIX | [v2.7.1-26.0.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
95ffd8d96 | BUGFIX | [fix][master]fix OrderedSet scope and missing argument
d7413000d | BUGFIX | [fix][v2.11.0]fix OrderedSet scope and missing argument
7af67e4ea | BUGFIX | [fix][v2.10.0]fix OrderedSet scope and missing argument
b717be638 | BUGFIX | [fix][v2.8.0]fix OrderedSet scope and missing argument
34f57866d | BUGFIX | [fix][v2.7.1]fix OrderedSet scope and missing argument
9577d9be8 | BUGFIX | 【inductor】fix static kernel bug
aa73dac55 | BUGFIX | 【inductor】fix static kernel bug
e95fa0ad1 | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.7.1-26.0.0
8a34c6089 | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.8.0-26.0.0
a9d6ee4b3 | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.9.0-26.0.0
20beb0fec | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.10.0-26.0.0
52825b302 | BUGFIX | 【fix】fix A3 inductor flags bug
5294f5ffe | BUGFIX | fix issues with npu_format_cast function parameters
fc8fcc8f3 | BUGFIX | 【fix】fix A3 inductor flags bug
0f47888fd | BUGFIX | fix issues with npu_format_cast function parameters
f6c956d14 | BUGFIX | fix issues with npu_format_cast function parameters
d3ebae158 | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.11.0
f924f9aeb | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.7.1
67723ed3e | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.8.0
01978bf4b | BUGFIX | fix npu_fusion_attention_grad sharding strategy on v2.10.0
4bdc6f560 | BUGFIX | [v2.7.1][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
9d370bd2c | BUGFIX | [v2.11.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
cfe71904f | BUGFIX | [v2.10.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
604573915 | BUGFIX | [v2.8.0][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
c7ef25183 | BUGFIX | [master][fix] fix shmem init by setting ACLSHMEMX_INIT_WITH_UNIQUEID
831ded11e | BUGFIX | [inductor][lowering_fx]fix A5 ETA/HLLM
b6e893ee1 | BUGFIX | [master][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
2768a7ff9 | BUGFIX | fix_rng_state
daaf69d9f | BUGFIX | fix_rng_state
2a57ff9e3 | BUGFIX | fix_rng_state
b511fef76 | BUGFIX | fix_rng_state
35aac025d | BUGFIX | fix_rng_state
1fe4a6f3f | BUGFIX | [v2.11.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
4b6da615b | BUGFIX | [v2.10.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
f7c6f023e | BUGFIX | [v2.8.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
49b320e68 | BUGFIX | [v2.7.1][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
ce56e9db2 | BUGFIX | [v2.6.0][bugfix]fix undefined symbol aclrtGetDeviceInfo with dlopen
c8e320fe2 | BUGFIX | 【inductor】fix static kernel bug
d6fdcbf9a | BUGFIX | fix(distributed): eagerConnectSingleDevice异常分层，区分可恢复/不可恢复
7fde7932b | BUGFIX | [fix]tiling_key_error
828e670fc | BUGFIX | [fix]tiling_key_error
658d42025 | BUGFIX | [fix]tiling_key_error
148d84e79 | BUGFIX | [fix]tiling_key_error
bfe98c932 | BUGFIX | [fix]tiling_key_error
f9487d048 | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
66121d8ee | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
d018d5d8d | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
d0584da54 | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
643967066 | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
1abdc019c | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
f0a99cf2f | BUGFIX | [bugfix]自定义scope_begin/end修改为显式注册
24f51db28 | BUGFIX | fix: add rethrow in ProcessGroupHCCL destructor to coredump on shutdown failure
4fbd0b5db | BUGFIX | fix: add rethrow in ProcessGroupHCCL destructor to coredump on shutdown failure
d1524d5ac | BUGFIX | fix: add rethrow in ProcessGroupHCCL destructor to coredump on shutdown failure
13d8427f6 | BUGFIX | fix: add rethrow in ProcessGroupHCCL destructor to coredump on shutdown failure
100992763 | BUGFIX | fix: add rethrow in ProcessGroupHCCL destructor to coredump on shutdown failure
0b2319f09 | BUGFIX | fix mlir dynamic shape launch error
a6f79de99 | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
fca88bf40 | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
21944b85b | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
2b2d226dc | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
85fcfa984 | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
6811fdb0b | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
adf751562 | BUGFIX | [fix] reduce memory usage in alltoall via compatible mode
e6445bec9 | BUGFIX | fix incorrect fallback fuseop in dynamic shape
2c2e89982 | BUGFIX | [bugfix]fix rts invalid params
08d567865 | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
273e153ed | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
6885f86b2 | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
4e16f3f21 | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
70db2fe64 | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
cebf05bcc | BUGFIX | [fix] Revert "[fix] tensor clone api consistency"
ccbbde6d8 | BUGFIX | [fix]  Revert "[fix] tensor clone api consistency"
51eaa5cba | BUGFIX | [fix]  Revert "[fix] tensor clone api consistency"
addc1979c | BUGFIX | [fix]  Revert "[fix] tensor clone api consistency"
faa83c691 | BUGFIX | [fix]  Revert "[fix] tensor clone api consistency"
42767ed8c | BUGFIX | [fix]  Revert "[fix] tensor clone api consistency"
a5dbb5989 | BUGFIX | [fix] Revert "[fix] tensor clone api consistency"
7636929db | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
6ce506f79 | BUGFIX | [fix] fix call of UseStreamInCurrentThread in hccl process
0a00c8d4e | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
cf61a0b80 | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
edbea6b08 | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
520e842d7 | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
4a3351bae | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
b84b13155 | BUGFIX | [fix]add torch_npu._C._npu_hasPrimaryContext
df8008c2f | BUGFIX | bugfix：step_trace_time计算时出现负数
129908d34 | BUGFIX | bugfix：step_trace_time计算时出现负数
6af12da7c | BUGFIX | bugfix：step_trace_time计算时出现负数
d82547269 | BUGFIX | bugfix：step_trace_time计算时出现负数
488ad3ccf | BUGFIX | bugfix：step_trace_time计算时出现负数
2ed77407e | BUGFIX | bugfix：step_trace_time计算时出现负数
337272901 | BUGFIX | bugfix：step_trace_time计算时出现负数
69baadba1 | BUGFIX | [Inductor] bugfix of cat when load with other=0.0
df136434f | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
9c3b1233a | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
36a2f423d | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
679d27b3b | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
58f2263f1 | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
1fbf80ae3 | BUGFIX | fix(nn): fix test for jit api: torch.jit.script、torch.jit.trace、torch.jit.save、torch.jit.load
57c6f0b17 | BUGFIX | fix_test
5d3160fe2 | BUGFIX | [fix] batch_isend_irecv oom
f200c7191 | BUGFIX | [fix] batch_isend_irecv oom
f5d63c8ba | BUGFIX | [fix] batch_isend_irecv oom
f096fbafa | BUGFIX | [fix] batch_isend_irecv oom
dccb6eeeb | BUGFIX | [fix] batch_isend_irecv oom
46fd1ca50 | BUGFIX | [fix] batch_isend_irecv oom
67eee1b3c | BUGFIX | Fix UT issues in the pipeline
32033f92d | BUGFIX | [fix] keep npu tensor strides for empty_like
abc89cdc3 | BUGFIX | [fix] batch_isend_irecv oom
81ee865af | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
2b550202a | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
7c8966f44 | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
34762ed81 | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
d23118fbe | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
593281aef | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
ef23cce69 | BUGFIX | [fix] reduce memory usage in scatter/gather via compatible mode
8343dcb87 | BUGFIX | fix: fix npugrphs backend bug and add indexput test case
bc651e172 | BUGFIX | [fix]add TORCH_NPU_API api
a51a3a164 | BUGFIX | [fix]add TORCH_NPU_API api
0a34cbf75 | BUGFIX | [fix]add TORCH_NPU_API api
73978ae7e | BUGFIX | [fix]add TORCH_NPU_API api
5defd4ef0 | BUGFIX | [fix]add TORCH_NPU_API api
5281d5de6 | BUGFIX | [fix]add TORCH_NPU_API api
7d80e64d2 | BUGFIX | [fix]add TORCH_NPU_API api
c74e69da7 | BUGFIX | [fix] keep npu tensor strides for empty_like
d4a9ccdd7 | BUGFIX | [fix] keep npu tensor strides for empty_like
6a527ab33 | BUGFIX | [fix] keep npu tensor strides for empty_like
9886a2ddb | BUGFIX | [fix] keep npu tensor strides for empty_like
51b180ccf | BUGFIX | [fix] keep npu tensor strides for empty_like
38812a880 | BUGFIX | [fix] keep npu tensor strides for empty_like
ecf541367 | BUGFIX | fix: fix npugrphs backend bug and add indexput test case
339dd8295 | BUGFIX | fix: fix npugrphs backend bug and add indexput test case
16bf63db4 | BUGFIX | fix: fix npugrphs backend bug and add indexput test case
24b84e5d3 | BUGFIX | [bugfix]fix rts invalid params
1286cc5a6 | BUGFIX | [bugfix]fix rts invalid params
2fdeb42e7 | BUGFIX | [bugfix]fix rts invalid params
acd84c123 | BUGFIX | [bugfix]fix rts invalid params
2ed861f4e | BUGFIX | [bugfix]fix rts invalid params
01fa00ece | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
79c7a1ad9 | BUGFIX | [bugfix]fix rts invalid params
f9e6c3c6c | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
093337cbc | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
01c4b5c52 | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
4e79b30ec | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
9b2a385ba | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
3144f8a29 | BUGFIX | fix(nn): fix test for nn api: torch.nn.ParameterDict, torch.nn.ParameterList, torch.nn.Sequential
6f42bdec7 | BUGFIX | fix d2h pinned memeory bug
4700f2420 | BUGFIX | fix d2h pinned memeory bug
00e474c7f | BUGFIX | [fix]pickle_local_object_error
2b1cc5ea2 | BUGFIX | Fixed IsGteCANNVersion
dfaa63163 | BUGFIX | [fix]pickle_local_object_error
7ae6a64cf | BUGFIX | [fix]pickle_local_object_error
856009930 | BUGFIX | [fix]pickle_local_object_error
fe8d11b24 | BUGFIX | [fix]pickle_local_object_error
497cccb5f | BUGFIX | [fix]pickle_local_object_error
1dc2cc2f2 | BUGFIX | fix reshape issue for dynamic
b4b4bbe0b | BUGFIX | [inductor] fix: remove debug prints for x0 indexing_code
ce677248f | BUGFIX | Fixed IsGteCANNVersion
de754b224 | BUGFIX | Fixed IsGteCANNVersion
4e9cf465a | BUGFIX | Fixed IsGteCANNVersion
6d6d6a9ad | BUGFIX | Fixed IsGteCANNVersion
ea685493e | BUGFIX | Fixed IsGteCANNVersion
2e2d7dcab | BUGFIX | Fixed IsGteCANNVersion
3c4ee22b2 | BUGFIX | 【fix】fix inductor bug in 2.11
c2f432df8 | BUGFIX | [inductor]fix reshape issue and optimized codegen for dynamic shape
38b6728af | BUGFIX | fix mlir dynamic shape error in kernel launch
d05465aad | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
e52be7356 | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
d86573267 | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
c578cc473 | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
3756b48a5 | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
564efc34e | BUGFIX | [fix] group only supports in A2, A3 for all2all, scatter and gather
0b9d27722 | BUGFIX | [fix] Synchronize HCCL streams instead of device in ProcessGroupHCCL shutdown
f556a8fe1 | BUGFIX | [fix] Synchronize HCCL streams instead of device in ProcessGroupHCCL shutdown
e623fd18a | BUGFIX | [fix] Synchronize HCCL streams instead of device in ProcessGroupHCCL shutdown
351d11824 | BUGFIX | [fix] Synchronize HCCL streams instead of device in ProcessGroupHCCL shutdown
70f3f5bc0 | BUGFIX | [fix] Synchronize HCCL streams instead of device in ProcessGroupHCCL shutdown
65fd39a25 | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
ac5cf101d | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
1425e0ec9 | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
1dd206de4 | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
0b5813d04 | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
a0395276f | BUGFIX | fix: custom_fwd and custom_bwd lags behind the target torch version
37b4199cf | BUGFIX | [inductor][test][fix] [精度工具]fix dump_fx_graph NotImplementedError and remove skip decorators
761cff284 | BUGFIX | fix dropout sharding strategy on v2.10.0
d52abb7a0 | BUGFIX | fix dropout sharding strategy on v2.9.0
a6e1bd4d9 | BUGFIX | fix(distributed): eagerConnectSingleDevice异常分层，区分可恢复/不可恢复
d08e44dcb | BUGFIX | [inductor]fix reshape issue and optimized codegen for dynamic shape
8c340c09f | BUGFIX | 【bug-fix】修复torch_npu2.9.0版本分布式用例未适配torch2.9.0的问题
30aaa07f4 | BUGFIX | [fix_torch_profiler_2.7.1]修复profiler L0级别kernel_details.csv无shape信息问题
e506bcb47 | BUGFIX | [fix_torch_profiler_2.10.0]修复profiler L0级别kernel_details.csv无shape信息问题
ab9d5d862 | BUGFIX | [fix_torch_profiler_2.6.0]修复profiler L0级别kernel_details.csv无shape信息问题
2f9aa0594 | BUGFIX | [fix_torch_profiler_master]修复profiler L0级别kernel_details.csv无shape信息问题
dc133ed4d | BUGFIX | [fix_torch_profiler_2.8.0]修复profiler L0级别kernel_details.csv无shape信息问题
ce103a784 | BUGFIX | [fix_torch_profiler_2.9.0]修复profiler L0级别kernel_details.csv无shape信息问题
56971a7d4 | BUGFIX | 【bug-fix】修复torch_npu2.8.0版本分布式用例未适配torch2.8.0的问题
57414ce67 | BUGFIX | 【bug-fix】修复torch_npu2.10.0版本分布式用例未适配torch2.10.0的问题
f8a0f174c | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
b56a42282 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
7b3f03568 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
807bbc9bf | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
75073c825 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
63b049b49 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtPointerGetAttributes`
336b176a5 | BUGFIX | bugfix:kSynchronizeBusyWaitMillis
588b30e2d | BUGFIX | bugfix:kSynchronizeBusyWaitMillis
9515ad443 | BUGFIX | bugfix:kSynchronizeBusyWaitMillis
642313d63 | BUGFIX | bugfix:kSynchronizeBusyWaitMillis
7deab6fd1 | BUGFIX | bugfix:kSynchronizeBusyWaitMillis
322588027 | BUGFIX | [fix] pin memory free core dump
2dc36a9e9 | BUGFIX | 【inductor】fix generate valid config
ce9672dda | BUGFIX | 【inductor】fix generate valid config
f4e5747fb | BUGFIX | [fix][v2.10.0]Fix the core dump issue of the COW feature.
aa92d145a | BUGFIX | [fix][v2.9.0]Fix the core dump issue of the COW feature.
a7ff9f24c | BUGFIX | [fix][v2.8.0]Fix the core dump issue of the COW feature.
6777fa347 | BUGFIX | [fix][v2.6.0]Fix the core dump issue of the COW feature.
cff185731 | BUGFIX | [fix][master]Fix the core dump issue of the COW feature.
967c03ea3 | BUGFIX | [fix][v2.7.1]Fix the core dump issue of the COW feature.
f5589e3f1 | BUGFIX | [fix]add empty_with_swapped_memory ut
59f32a3d5 | BUGFIX | [fix]add empty_with_swapped_memory ut
29a349bf1 | BUGFIX | [fix]add empty_with_swapped_memory ut
57094aaaf | BUGFIX | [fix]add empty_with_swapped_memory ut
6b024c491 | BUGFIX | [fix]add empty_with_swapped_memory ut
d2406c660 | BUGFIX | [fix]add empty_with_swapped_memory ut
9f030ebcd | BUGFIX | [fix]user_autotune_npu
7f4be1158 | BUGFIX | [fix]user_autotune_npu
6b5c697df | BUGFIX | [fix]user_autotune_npu
2c194e8af | BUGFIX | [fix]user_autotune_npu
8544fbf6b | BUGFIX | [fix]user_autotune_npu
8b4c3e678 | BUGFIX | [fix] memory leak
7672a5b86 | BUGFIX | [fix]NPUSwappedMemoryAllocator memory leak
e0e3016ff | BUGFIX | [fix] memory leak
a0c758865 | BUGFIX | [fix] memory leak
0bbadebc7 | BUGFIX | [fix] memory leak
8f790fe78 | BUGFIX | [fix] memory leak
2fbda6782 | BUGFIX | Fixed IsGteCANNVersion
ddbfedff4 | BUGFIX | Fixed IsGteCANNVersion
0ff9bf558 | BUGFIX | Fixed IsGteCANNVersion
811521a83 | BUGFIX | Fixed IsGteCANNVersion
a2c9adc94 | BUGFIX | Fixed IsGteCANNVersion
9409d43fc | BUGFIX | Fixed IsGteCANNVersion
1d637cec4 | BUGFIX | [fix]mlir_enable
3c13ae5e7 | BUGFIX | [fix]mlir_enable
7ca34ff50 | BUGFIX | [fix]mlir_enable
2b2424790 | BUGFIX | [fix]mlir_enable
125ff02a4 | BUGFIX | [fix]mlir_enable
841207b00 | BUGFIX | [fix]mlir_enable
dd0b44f52 | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
98029ad2a | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
768d3c4e9 | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
240dcb176 | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
a23345516 | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
14b889b7c | BUGFIX | fix: create inv_scale_factor on same NPU as qkv in FA pattern example
d2ac93043 | BUGFIX | [fix][test] fix test_schedule ut
21039cff3 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api.
31205c19a | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api.
47f9516b0 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api.
3f5607af6 | BUGFIX | fix: fix storage weakref wrapper
ee2ccd7fb | BUGFIX | [v2.10.0][bugfix]get_device_properties报错NN processes exceeds the limit
a073707a2 | BUGFIX | [v2.9.0][bugfix]get_device_properties报错NN processes exceeds the limit
7f721effd | BUGFIX | [v2.8.0][bugfix]get_device_properties报错NN processes exceeds the limit
7434319c3 | BUGFIX | [v2.7.1][bugfix]get_device_properties报错NN processes exceeds the limit
728471e82 | BUGFIX | [v2.6.0][bugfix]get_device_properties报错NN processes exceeds the limit
adce5117f | BUGFIX | [master][bugfix]get_device_properties报错NN processes exceeds the limit
c6f2bcc52 | BUGFIX | fix ignore it when device_index < 0 on with device
33f80abb1 | BUGFIX | fix ignore it when device_index < 0 on with device
5840910e1 | BUGFIX | fix ignore it when device_index < 0 on with device
b0d91f540 | BUGFIX | fix ignore it when device_index < 0 on with device
24a6a8d52 | BUGFIX | fix ignore it when device_index < 0 on with device
63c448335 | BUGFIX | fix ignore it when device_index < 0 on with device
8d87c3d39 | BUGFIX | fix: fix storage weakref wrapper
21cbb8715 | BUGFIX | fix: restore limit_core_num implemented by the scope
1833d6d52 | BUGFIX | fix: restore limit_core_num implemented by the scope
0a14e5734 | BUGFIX | fix: restore limit_core_num implemented by the scope
993ce9488 | BUGFIX | fix: restore limit_core_num implemented by the scope
52c8c065d | BUGFIX | fix: restore limit_core_num implemented by the scope
503381ca1 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtMallocHostWithCfg`
afe23cd3f | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtMallocHostWithCfg`
e791ba1ed | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtMallocHostWithCfg`
4aaa20af1 | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtMallocHostWithCfg`
b15480b1a | BUGFIX | [fix]Perform driver version check prior to invoking interface `aclrtMallocHostWithCfg`
4e6a200fd | BUGFIX | fix: restore limit_core_num implemented by the scope
8831262dc | BUGFIX | [fix]Modify customize_dtype to keyword argument
30d7606e4 | BUGFIX | [fix]Modify customize_dtype to keyword argument
fbb4a145c | BUGFIX | [fix]Modify customize_dtype to keyword argument
db10780dc | BUGFIX | [fix]Modify customize_dtype to keyword argument
120341a1b | BUGFIX | [fix]Modify customize_dtype to keyword argument
f1e37df8a | BUGFIX | [fix]Modify customize_dtype to keyword argument
c90658d5b | BUGFIX | [fix] Solve recursivly initialize device
37f9bf745 | BUGFIX | [fix] Solve recursivly initialize device
f007a999e | BUGFIX | [fix] Solve recursivly initialize device
be49dc036 | BUGFIX | [fix] Solve recursivly initialize device
cf25b793c | BUGFIX | [fix] Solve recursivly initialize device
e1992186f | BUGFIX | [fix] Solve recursivly initialize device
fd6927797 | BUGFIX | [fix]fix profiler warn when cann path does not exist
972e30401 | BUGFIX | [fix]fix profiler warn when cann path does not exist
a886a76b8 | BUGFIX | [fix]fix profiler warn when cann path does not exist
7243d5fee | BUGFIX | [fix]fix profiler warn when cann path does not exist
2ddd04c9a | BUGFIX | [fix]fix profiler warn when cann path does not exist
15b82b22a | BUGFIX | [fix] fix profiler warn when cann path does not exist
c1e1d3461 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
d38c6fe18 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
78f8fcee8 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
8d9f4c542 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
197908353 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
2adc584a8 | BUGFIX | 【bug-fix】DeviceProperty属性中multi_processor_count设置默认值
4b042a40b | BUGFIX | 【fix】修复assertError类型报错的问题
795654945 | BUGFIX | 【fix】修复assertError类型报错的问题
6744f15df | BUGFIX | [master][bugfix] update Tensorpipe
00c78a87c | BUGFIX | [master][bugfix] update Tensorpipe
725b3505c | BUGFIX | [master][bugfix] update Tensorpipe
6a16cb054 | BUGFIX | [master][bugfix] update Tensorpipe
0f90c8b93 | BUGFIX | [master][bugfix] update Tensorpipe
615a3982c | BUGFIX | [master][bugfix] update Tensorpipe
380968919 | BUGFIX | fix serialization offset filelike bug
c85822829 | BUGFIX | fix serialization offset filelike bug
09f67a700 | BUGFIX | fix serialization offset filelike bug
d45a68df7 | BUGFIX | fix serialization offset filelike bug
4511a1da1 | BUGFIX | fix serialization offset filelike bug
8f5e606dd | BUGFIX | fix serialization offset filelike bug
b2f171ffb | BUGFIX | 动态符号报错修复
a32837844 | BUGFIX | [Inductor] fix symbolic cat fallback
7db57459f | BUGFIX | [inductor][精度工具]fix _patch__npu_concatkernel_create/_patch_reduction_create
9eaf2f3bd | BUGFIX | [fix][v2.10.0]aclgraph testcase add 910C support
9ff7ea1d3 | BUGFIX | [fix][v2.9.0]aclgraph testcase add 910C support
0a649b884 | BUGFIX | [fix][v2.8.0]aclgraph testcase add 910C support
cd111d1e3 | BUGFIX | [fix][v2.7.1]aclgraph testcase add 910C support
4f4625184 | BUGFIX | [fix][v2.6.0]aclgraph testcase add 910C support
d43940c8e | BUGFIX | [fix] load api consistency
573177afd | BUGFIX | [fix] load api consistency
75a699b37 | BUGFIX | [fix] load api consistency
762804418 | BUGFIX | [fix] load api consistency
f00e39b37 | BUGFIX | [fix] load api consistency
9218a4b3e | BUGFIX | [fix] load api consistency
a260b2448 | BUGFIX | [fix] tensor clone api consistency
64ebb6e0f | BUGFIX | [fix] tensor clone api consistency
76e13e58c | BUGFIX | [fix] tensor clone api consistency
1b0f3a334 | BUGFIX | [fix] tensor clone api consistency
2d86ea375 | BUGFIX | [fix] tensor clone api consistency
4b9557261 | BUGFIX | [fix] tensor clone api consistency
7492ad614 | BUGFIX | [fix] as_strided consistency
22b82f571 | BUGFIX | [fix] as_strided consistency
ca4ef0313 | BUGFIX | [fix] as_strided consistency
14ac3813e | BUGFIX | [fix] as_strided consistency
c4568eb31 | BUGFIX | [fix] as_strided consistency
55de335f0 | BUGFIX | [fix] as_strided consistency
6b4394105 | BUGFIX | [fix] cast stridedshard error
4d4a7a5a8 | BUGFIX | [fix] cast stridedshard error
c45faa651 | BUGFIX | [fix] cast stridedshard error
1f261796c | BUGFIX | [fix] cast stridedshard error
99d85d2c8 | BUGFIX | [fix] cast stridedshard error
b956cdc34 | BUGFIX | fix cast stridedshard error
ffa1aa316 | BUGFIX | fix:Optimize the implementation of compile + npugraphs
af5c93b5b | BUGFIX | [fix][v2.9.0] add assert_alignment import and replace evaluate_static_shape
a2f6dd0ac | BUGFIX | [fix][v2.8.0] add assert_alignment import
f00fae974 | BUGFIX | fix: fix index_put_ bool error
25b5698c8 | BUGFIX | fix: fix index_put_ bool error
e800763c3 | BUGFIX | fix: fix index_put_ bool error
f0c9519f8 | BUGFIX | fix: fix index_put_ bool error
2a110a7b8 | BUGFIX | [fix][2.10.0]add getMemoryFraction attribute for torch._C
2bb674096 | BUGFIX | [fix][2.9.0]add getMemoryFraction attribute for torch._C
423c90ae7 | BUGFIX | [fix][2.8.0]add getMemoryFraction attribute for torch._C
cb7166e01 | BUGFIX | [fix][2.7.1]add getMemoryFraction attribute for torch._C
68f0940f4 | BUGFIX | [fix][2.6.0]add getMemoryFraction attribute for torch._C
7ecdce1eb | BUGFIX | [fix]add getMemoryFraction attribute for torch._C
dc8207073 | BUGFIX | fix flight recorder dump file name bug
f8f1b77ab | BUGFIX | fix: 恢复test_init_process_group用例
feaf1a4cf | BUGFIX | fix flight recorder dump file name bug
fc04b3e18 | BUGFIX | fix flight recorder dump file name bug
b96f50962 | BUGFIX | fix flight recorder dump file name bug
c59d7cedd | BUGFIX | fix flight recorder dump file name bug
8dee4ac67 | BUGFIX | fix flight recorder dump file name bug
878a8cb28 | BUGFIX | [fix][inductor]fix aten.any/prod codegen issue
864e16ded | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
02ffd5090 | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
926239533 | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
21080d172 | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
c258f9278 | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
1c12209fe | BUGFIX | 【bug-fix】在torch_npu注册后同步更新rpc模块的BackendType引用
cd8d62488 | BUGFIX | Fix Dockerfile (2.10)
638106a64 | BUGFIX | fix_device_dynamo_trace
5b26465e3 | BUGFIX | fix_device_dynamo_trace
6b8b7cb1f | BUGFIX | fix_device_dynamo_trace
42067023d | BUGFIX | fix_device_dynamo_trace
e88b5b1c6 | BUGFIX | fix_device_dynamo_trace
0d45fe33f | BUGFIX | fix_device_dynamo_trace
ce153f7cd | BUGFIX | fix core num to hccl
10bc06261 | BUGFIX | [fix]mlir_import
44e0d99f6 | BUGFIX | [fix]profiler export_type warning mod
0d2cd849d | BUGFIX | [fix]profiler export_type warning mod
6004dd07c | BUGFIX | [fix]profiler export_type warning mod
cb72ca1a9 | BUGFIX | [fix]profiler export_type warning mod
0d330429d | BUGFIX | [fix]profiler export_type warning mod
4da21f35d | BUGFIX | [fix]profiler export_type warning mod
41cd21e3b | BUGFIX | 【inductor】fix reduction bug when in persistent_reduction
f225ea66c | BUGFIX | Fix Dockerfile (2.9)
fcd98896b | BUGFIX | [Fix] [inductor] Fix the compilation failure issue of Scan IR under NPU + Inductor environment
e1e87269d | BUGFIX | fix typing in python 3.9
8f6ca366d | BUGFIX | fix typing in python 3.9
a18841cce | BUGFIX | fix typing in python 3.9
4aa14f7c7 | BUGFIX | [v2.6.0][bugfix]code Rollback
0e7887883 | BUGFIX | [v2.7.1][bugfix]code Rollback
09d2c55fd | BUGFIX | [master][bugfix]code Rollback
4f778c95e | BUGFIX | [v2.9.0][bugfix]code Rollback
924517bb9 | BUGFIX | [v2.10.0][bugfix]code Rollback
a0b0e61bd | BUGFIX | [v2.8.0][bugfix]code Rollback
a86817e1f | BUGFIX | Fix Tensor.empty deterministic bug.
dd06f35ae | BUGFIX | Fix Tensor.empty deterministic bug.
3318e5c80 | BUGFIX | Fix Tensor.empty deterministic bug.
920efd90f | BUGFIX | Fix Tensor.empty deterministic bug.
38c049c83 | BUGFIX | Fix Tensor.empty deterministic bug.
ad1698310 | BUGFIX | Fix Tensor.empty deterministic bug.
1fbd67778 | BUGFIX | [fix]trace_fix
8fd7f72b6 | BUGFIX | [fix]trace_fix
2d54cf64b | BUGFIX | [fix]trace_fix
5a7684e9a | BUGFIX | [fix]trace_fix
5416b915d | BUGFIX | [fix]trace_fix
f9a89bb2e | BUGFIX | [fix]trace_fix
bcec1ab67 | BUGFIX | 【inductor】fix generate valid config
54273eef4 | BUGFIX | 【inductor】fix generate valid config
8c6ae9a7c | BUGFIX | 【inductor】fix generate valid config
42da1f41a | BUGFIX | [v2.10.0][bugfix]get_device_properties报错NN processes exceeds the limit
9a1a043db | BUGFIX | [v2.9.0][bugfix]get_device_properties报错NN processes exceeds the limit
35cd8b576 | BUGFIX | [v2.8.0][bugfix]get_device_properties报错NN processes exceeds the limit
33f2b034e | BUGFIX | [v2.7.1][bugfix]get_device_properties报错NN processes exceeds the limit
460f536ec | BUGFIX | [v2.6.0][bugfix]get_device_properties报错NN processes exceeds the limit
4db67ffe2 | BUGFIX | [master][bugfix]get_device_properties报错NN processes exceeds the limit
0e4718038 | BUGFIX | fix: 修复仅fake后端增加npu支持
aeabca9cd | BUGFIX | Fix Tensor.empty deterministic bug.
8c0b41d61 | BUGFIX | Fix Tensor.empty deterministic bug.
2ce79111e | BUGFIX | Fix Tensor.empty deterministic bug.
ae467d2ca | BUGFIX | Fix Tensor.empty deterministic bug.
2a0266cf0 | BUGFIX | Fix Tensor.empty deterministic bug.
557a510db | BUGFIX | Fix Tensor.empty deterministic bug.
29be895a8 | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
96a55c00f | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
6e7280ecd | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
30becddab | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
5660d5196 | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
3bf6c6f33 | BUGFIX | [fix]Enhanced the interface with null pointer validation for is_pinned
9c205684a | BUGFIX | [fix]rollback del 32 allocator
bd57c5c46 | BUGFIX | [fix]rollback del 32 allocator
fd61281e9 | BUGFIX | [fix]rollback del 32 allocator
7e071d01a | BUGFIX | [fix]rollback del 32 allocator
bbe936df1 | BUGFIX | [fix]rollback del 32 allocator
6113f7b7f | BUGFIX | [fix]rollback del 32 allocator
3182ab047 | BUGFIX | 【inductor】 fix invoke_subgraph fallback bug
ada7f96e2 | BUGFIX | 【inductor】 fix invoke_subgraph fallback bug
b099e0682 | BUGFIX | 【inductor】 fix invoke_subgraph fallback bug
f58d695bb | BUGFIX | 【inductor】 fix invoke_subgraph fallback bug
9cebf9ea0 | BUGFIX | 【inductor】fix invoke_subgraph fallback bug
e2dac9e99 | BUGFIX | 【inductor】fix invoke_subgraph fallback bug
17d2b4404 | BUGFIX | fix test case
50b4c1fa0 | BUGFIX | fix test case
2f63ecaa9 | BUGFIX | fix test case
a99b25572 | BUGFIX | fix test case
92bd0c9bb | BUGFIX | fix test case
bf722958d | BUGFIX | fix: fix fallbacklist embedding cat mode error
652bc2ba4 | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
70e2290c5 | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
aa5be1e4d | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
7fc7414dd | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
edb94d7f6 | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
e9dbf867e | BUGFIX | 【bugfix】patch_event_variable_python_type for profiling bugs
c83bac42c | BUGFIX | [bugfix] The issue of tensor lifecycle being too long and the stride validation problem.
c088eab9e | BUGFIX | [bugfix] The issue of tensor lifecycle being too long and the stride validation problem.
c4f4b33a9 | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
8f95fd396 | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
9c9e4640e | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
f73b9e0db | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
090d91f0e | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
0115c476c | BUGFIX | fix hostmemory leak, release it after aclCreateTensor
d0a18e3e8 | BUGFIX | fix: copy_ bug for negative view
b123a91a6 | BUGFIX | fix: copy_ bug for negative view
d0a321c18 | BUGFIX | fix: copy_ bug for negative view
a8e90f69d | BUGFIX | fix: copy_ bug for negative view
d404451f1 | BUGFIX | fix: copy_ bug for negative view
04c01018b | BUGFIX | 【inductor】fix ci bug
a1c188aaa | BUGFIX | fix: support nest subblock
d69f2f060 | BUGFIX | fix v2.10.0 aoti bug
a8073bc0d | BUGFIX | fix: copy_ bug for negative view
e904e77fb | BUGFIX | [feature] fix utils path not found
43600c94f | BUGFIX | 【fix】fix tiling bug
0dfce7921 | BUGFIX | [inductor] fix flex atten bug
661a5c908 | BUGFIX | [feature] fix utils path not found
aaa0d54d2 | BUGFIX | [feature] fix utils path not found
2b5600b31 | BUGFIX | [feature] fix utils path not found
dc5b1bb84 | BUGFIX | [feature] fix utils path not found
a6d23cde1 | BUGFIX | fix: remove slice_scatter fusion limit
88efa1f78 | BUGFIX | fix bug in transfer_to_npu
c6d865667 | BUGFIX | [fix] revert !30248 triton.py and apply !30034 tmp mask fix
57ad44dd5 | BUGFIX | [ut] fix test_ifa_update
7650811e2 | BUGFIX | 【inductor】fix master inductor bug
9a42c115b | BUGFIX | fix mlir aten.pow lowering bug
1bc0b2dfa | BUGFIX | fix mlir aten.pow lowering bug
cfde3e6a1 | BUGFIX | fix mlir aten.pow lowering bug
591354a8e | BUGFIX | [feature] fix utils path not found
6d7d33c5c | BUGFIX | fix as_strided bug
8b6a21a64 | BUGFIX | fix as_strided bug
173b7e8dd | BUGFIX | fix as_strided bug
fb919407d | BUGFIX | fix as_strided bug
cf4aa50ee | BUGFIX | fix as_strided bug
5bc5d6f01 | BUGFIX | Fix __rmod__ testcase.
014c2699e | BUGFIX | Fix __rmod__ testcase.
91b0c4f24 | BUGFIX | Fix __rmod__ testcase.
283f3e2ee | BUGFIX | Fix __rmod__ testcase.
7e0ca3731 | BUGFIX | Fix __rmod__ testcase.
93f4c01d0 | BUGFIX | fix torch.randint testcases
fbef11ea8 | REVERT | Revert PR and fix bug
ca0081639 | REVERT | Revert PR and fix bug
cbdd5711c | BUGFIX | Fix __rmod__ testcase.
79168bcce | REVERT | revert fix bug in importing mlir module
c627ad792 | BUGFIX | fix zero copy set device
b8416a6d0 | BUGFIX | fix zero copy set device
7b8a4dcd1 | BUGFIX | fix zero copy set device
9a2878f25 | BUGFIX | fix zero copy set device
c7ea32e79 | BUGFIX | fix zero copy set device
81ee1f4ab | BUGFIX | fix zero copy set device
3fc72466f | BUGFIX | fix as_strided bug
9ca5ac6b5 | BUGFIX | fix mem_get_info
663b21efa | BUGFIX | fix mem_get_info
bf45c8945 | BUGFIX | fix mem_get_info
7f1b28459 | BUGFIX | fix mem_get_info
a145e5118 | BUGFIX | fix mem_get_info
39f439d19 | BUGFIX | fix mem_get_info
315342892 | BUGFIX | fix: process compile_fx options
cef9936cf | BUGFIX | fix: process compile_fx options
04e619ffd | BUGFIX | fix: process compile_fx options
a8caee6bc | BUGFIX | fix: process compile_fx options
df80288cb | BUGFIX | fix: process compile_fx options
2fbaa35d4 | BUGFIX | fix: process compile_fx options
92a86938a | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
344193ced | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
68c4ec1ec | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
4c8e85d69 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
e1941792a | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
f8bbb1272 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
beac5137b | BUGFIX | fix bug in importing mlir module
00f6cd17d | BUGFIX | Fix MLIR enabling option bug
dfc2f5c99 | BUGFIX | fix the duplicate patch issue
cd5f447d5 | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
6f79c0561 | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
4e5208f3e | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
f12941356 | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
0a2ed18e2 | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
67df0b3ea | BUGFIX | [fix]add socversion check for aclrtMallocHostWithCfg
270878a56 | BUGFIX | fix zero copy set device
7f6c48319 | BUGFIX | fix zero copy set device
a1ab12002 | BUGFIX | fix zero copy set device
a4b4cb778 | BUGFIX | fix zero copy set device
1043e2655 | BUGFIX | fix: fix masked_subblock mask
5f5f6836c | BUGFIX | fix_npu_function
e32c44c98 | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
c167f69aa | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
e316d194e | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
7dee20890 | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
bb00700e2 | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
a2899f601 | REVERT | Revert "[fix] set core num to hccl thread when calling hccl operators"
e8efb02ef | BUGFIX | 【inductor】Fix catlass incorrect output stride for grouped matmul
74ff0ccda | BUGFIX | [fix][inductor] lowering_index_select
75c77b902 | BUGFIX | [fix]profiler db analyse optimize
35504124f | BUGFIX | [fix]profiler db analyse optimize
68258aab5 | BUGFIX | [fix]profiler db analyse optimize
172313789 | BUGFIX | [fix]profiler db analyse optimize
678efd810 | BUGFIX | [fix]profiler db analyse optimize
5212b988c | BUGFIX | [fix]profiler db analyse optimize
f0dd33414 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
dbf85ebb2 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
0bba9eaa5 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
3a2ad3b24 | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
81ffe272e | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
46a64be5f | BUGFIX | [fix] set core num to hccl thread when calling hccl operators
19b0a1d5a | BUGFIX | hllm model bug fix
6dc00c5df | BUGFIX | [Inductor] Fix Bugs in `load` Generator of NPU Kernel in Inductor.
6fd0dd77d | BUGFIX | fix: fix low dims rules in indirect memory access
f685c01af | BUGFIX | 【inductor】fix thread pool when inductor is imported
4f614caa1 | BUGFIX | 【inductor】fix thread pool when inductor is imported
bdec384b6 | BUGFIX | 【inductor】fix thread pool when inductor is imported
3cf5ee027 | BUGFIX | 【inductor】fix thread pool when inductor is imported
6ddaf5ad8 | BUGFIX | 【inductor】fix thread pool when inductor is imported
d09f482c7 | BUGFIX | 【inductor】fix thread pool when inductor is imported
9ae029a17 | BUGFIX | fix: add ut and fix for view combine
f17d4cc79 | REVERT | Revert 【inductor】AsyncCompile before register
f9f9f5c3c | REVERT | Revert 【inductor】AsyncCompile before register
d9d99885e | REVERT | Revert 【inductor】AsyncCompile before register
018517f93 | REVERT | Revert 【inductor】AsyncCompile before register
af1298cc9 | REVERT | Revert 【inductor】AsyncCompile before register
f76927b83 | REVERT | Revert 【inductor】AsyncCompile before register
5b8aba051 | BUGFIX | 【inductor】fix thread pool when inductor is imported
bc9fafbb3 | BUGFIX | 【inductor】fix thread pool when inductor is imported
367bfe1ba | BUGFIX | [master]fix dropout
b09773813 | BUGFIX | [2.10.0]fix dropout
598a78574 | BUGFIX | 【inductor】fix thread pool when inductor is imported
7c11912e1 | BUGFIX | 【inductor】fix thread pool when inductor is imported
85fa92548 | BUGFIX | fix thread pool when inductor is imported
daa38df2b | BUGFIX | fix thread pool when inductor is imported
33182e78d | BUGFIX | 【Inductor】 fix catlass bias tiling and layout problem
665c0117c | BUGFIX | [2.9.0]fix dropout
c472b5b58 | BUGFIX | [2.8.0]fix dropout
578bbd2f1 | BUGFIX | [2.6.0]fix dropout
30dc60baf | BUGFIX | [fix] fsdp patch fix all_gather_copy_in.default unsupported issue
401dfddfb | BUGFIX | [fix] fsdp patch fix all_gather_copy_in.default unsupported issue
bf13a531f | BUGFIX | [fix] fsdp patch fix all_gather_copy_in.default unsupported issue
d547bcfcb | BUGFIX | [fix] fsdp patch fix all_gather_copy_in.default unsupported issue
7d69f9b48 | BUGFIX | fix: fix index_select code cache with cat
f92c0e316 | BUGFIX | fix: remove indirect mem default, adapt old env variable
d2f944a90 | BUGFIX | 【inductor】fix inductor ci bug
4a63d3f26 | BUGFIX | 【inductor】fix inductor ci bug
dce775286 | BUGFIX | 【inductor】fix inductor ci bug
dd84d7f26 | BUGFIX | 【inductor】fix inductor ci bug
07d3542cb | BUGFIX | fix head path
e52a8a79d | BUGFIX | fix bug in transfer_to_npu
3b1e87223 | BUGFIX | fix bug in transfer_to_npu
281b1b32e | BUGFIX | fix bug in transfer_to_npu
dcecc4223 | BUGFIX | fix bug in transfer_to_npu
9b8d3dde4 | BUGFIX | fix bug in transfer_to_npu
a91753d0b | BUGFIX | fix: add kernel schedule fusion limit
d5e590c8a | BUGFIX | fix: add simt fallback
a296e9a91 | BUGFIX | fix: simt must use load store
12ed2409e | BUGFIX | [bugfix]: add cstring head file for std::memset, otherwise may bring Probabilistic compile error.
20357b76d | REVERT | Revert "memory optimaze"
9cad09cc8 | REVERT | Revert "memory optimaze"
5e2b786a2 | REVERT | Revert "memory optimaze"
3c7d45683 | REVERT | Revert "memory optimaze"
9ea3f12d4 | BUGFIX | fix dropout
a01560d77 | BUGFIX | Fixed test_modules_can_be_imported
869571937 | BUGFIX | [inductor] fix triton codegen for symbolic shape
de6c3901f | BUGFIX | [profiler-v2.8]修复动态Profiling采部分rank无法解析&解析打屏问题
31b2af97c | BUGFIX | [profiler-v2.7]修复动态Profiling采部分rank无法解析&解析打屏问题
2a97ce715 | BUGFIX | [profiler-v2.6]修复动态Profiling采部分rank无法解析&解析打屏问题
dc3839ac1 | BUGFIX | [profiler-v2.9]修复动态Profiling采部分rank无法解析&解析打屏问题
d61ec33ea | BUGFIX | [profiler]修复动态Profiling采部分rank无法解析&解析打屏问题
0e93c91c3 | BUGFIX | [profiler-2.6.0]修复动态Profiling采部分rank无法解析&解析打屏问题
c9964e8a3 | BUGFIX | [profiler-2.7.1]修复动态Profiling采部分rank无法解析&解析打屏问题
651b592f9 | BUGFIX | [profiler-2.8.0]修复动态Profiling采部分rank无法解析&解析打屏问题
b705b90fa | BUGFIX | [profiler-2.9.0]修复动态Profiling采部分rank无法解析&解析打屏问题
cb99a41d1 | BUGFIX | [inductor] [torch-mlir] fix acc issue cause by non contiguous inputs and outputs.
0c1774b9a | BUGFIX | fix libdevice path
230e8f344 | BUGFIX | fix bug
a3af87c1c | BUGFIX | bugfix when the sum axis is negative
fbfffe65f | BUGFIX | bugfix when the sum axis is negative
7fa4ab558 | BUGFIX | bugfix when the sum axis is negative
1de60d5c3 | BUGFIX | bugfix when the sum axis is negative
0b6e500d8 | BUGFIX | fix A5 cumsum output dtype
6413cdfd1 | BUGFIX | fix the bug of int32 dtype overflow
b2fde6fce | BUGFIX | fix transfer_to_npu close torch.jit.script in every situation
cc5635212 | BUGFIX | fix transfer_to_npu close torch.jit.script in every situation
2c7eedbf2 | BUGFIX | fix transfer_to_npu close torch.jit.script in every situation
441c1f628 | BUGFIX | fix transfer_to_npu close torch.jit.script in every situation
8e989ea2e | BUGFIX | fix transfer_to_npu close torch.jit.script in every situation
bedc8509b | BUGFIX | deal with event query() failed
98b71a48e | BUGFIX | deal with event query() failed
2b756f1cc | BUGFIX | deal with event query() failed
1d91259b4 | BUGFIX | deal with event query() failed
7c03236ad | BUGFIX | deal with event query() failed
c35dcc617 | BUGFIX | deal with event query() failed
6a9521bb2 | BUGFIX | deal with event query() failed
ea4bec8b0 | BUGFIX | deal with event query() failed
ece588474 | BUGFIX | deal with event query() failed
26b2aa069 | BUGFIX | deal with event query() failed
78fdc82ca | REVERT | Revert "memory optimaze"
de58dc6d4 | REVERT | Revert "memory optimaze"
d7f322ccb | REVERT | Revert "memory optimaze"
79d51c855 | REVERT | Revert "memory optimaze"
91493c027 | REVERT | Revert "memory optimaze"
5d87acc01 | BUGFIX | fix no module named torch._inductor.triton_heuristics
e6319cfb9 | BUGFIX | fix no module named torch._inductor.triton_heuristics
543a29369 | BUGFIX | fix no module named torch._inductor.triton_heuristics
2b4e0a1b1 | BUGFIX | fix no module named torch._inductor.triton_heuristics
9c1608cef | BUGFIX | [A5]indirect mem bugfix
5721ff35d | BUGFIX | [bugfix] change iterator method to avoid memory error.
12d689a08 | BUGFIX | fix error
5f8e5afe1 | BUGFIX | fix error
6cb77d54c | BUGFIX | [fix] test_npu_graph_attention_function
49d228b23 | BUGFIX | [fix] test_npu_graph_attention_function
d2d756fc2 | BUGFIX | Fix UT world_size=2
cc4452eb1 | BUGFIX | Fix UT world_size=2
2d709e649 | BUGFIX | Fix UT world_size=2
b8665e3fb | BUGFIX | Fix UT world_size=2
45aee0d1d | BUGFIX | fix send/recv to graph model
f5f5a4103 | BUGFIX | fix send/recv to graph model
e50c73c83 | BUGFIX | fix send/recv to graph model
75ee6309a | BUGFIX | fix send/recv to graph model
962e9341d | BUGFIX | fix send/recv to graph model
f78a4af3d | BUGFIX | fix docs
f1a8ebd87 | BUGFIX | fix docs
cf130e34a | BUGFIX | fix docs
9904b02d2 | BUGFIX | fix docs
5d39f5d9b | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
f8aa1d3ab | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
db4040f29 | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
bb716b9a4 | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
6ab82720c | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
f5c2dde96 | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
1eb1bdb7d | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
e4d2f37fb | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
ffa7b89eb | BUGFIX | [fix]synchronize when aclrtMemcpyAsyncWithCondition
f0efeb6d4 | BUGFIX | fix error
582f4262a | BUGFIX | fix:several docs issues
304a5f5fb | BUGFIX | fix:several docs issues
0bef95fd1 | BUGFIX | fix:several docs issues
1cfa301c5 | BUGFIX | fix:seceral docs issues
bbd34a48a | BUGFIX | fix fake tensor serialiazation core dump bug
bc3ab9849 | BUGFIX | fix fake tensor serialiazation core dump bug
14acfe401 | BUGFIX | fix fake tensor serialiazation core dump bug
daf71a8fc | BUGFIX | fix fake tensor serialiazation core dump bug
5b503a600 | BUGFIX | fix_bug
b5aaa00d9 | BUGFIX | fix_bug
9fd43a36b | BUGFIX | fix_bug
c0dd484c8 | BUGFIX | fix_bug
a5bc853df | BUGFIX | Fix UT from test_register_sharding
1e5d07c63 | BUGFIX | Fix UT from test_register_sharding
e6c573751 | BUGFIX | Fix UT from test_register_sharding
858f2a44e | BUGFIX | Fix UT from test_register_sharding
0b05532e2 | BUGFIX | fix inductor bug
a40b5b7cb | BUGFIX | Fix UT from test_register_sharding
45352669d | BUGFIX | Fix UT from test_register_sharding
b4a3eb18d | BUGFIX | Fix UT from test_register_sharding
97b173366 | BUGFIX | Fix UT from test_register_sharding
572453bec | BUGFIX | Fix UT from test_register_sharding
e8e435d5b | BUGFIX | [shmem] fix with aclshmem
a934fdb06 | BUGFIX | [shmem] fix with aclshmem
c8a00c45c | BUGFIX | [shmem] fix with aclshmem
26e5a7707 | BUGFIX | [shmem] fix with aclshmem
efd75bb60 | BUGFIX | [fix] test_npu_graph_attention_function
b27111fba | BUGFIX | [fix] test_npu_graph_attention_function
631325fab | BUGFIX | [fix] test_npu_graph_attention_function
fad558bf2 | BUGFIX | [fix] test_npu_graph_attention_function
f9fcb8e42 | BUGFIX | [fix] test_npu_graph_attention_function
dabe2e4b4 | BUGFIX | [fix] test_npu_graph_attention_function
f954d811f | BUGFIX | [fix] test_npu_graph_attention_function
fb2839ec6 | BUGFIX | fix apis and framework_feature_guide_pytorch
4bff295e8 | BUGFIX | fix torch_npu synchronize rule
092a66d7d | BUGFIX | fix torch_npu synchronize
357ebbb94 | BUGFIX | fix torch_npu synchronize
a6b31344c | BUGFIX | fix torch_npu synchronize rule
ddd361768 | BUGFIX | fix torch_npu synchronize rule
750a0f616 | BUGFIX | fix torch npu synchronize rule
d894259ea | BUGFIX | fix torch_npu synchronize rule
8d2e08f89 | BUGFIX | fix DTensor
c72f087f7 | BUGFIX | fix DTensor
89289f4b5 | BUGFIX | fix DTensor
1d3f5bc9f | BUGFIX | fix DTensor UT
be140503a | BUGFIX | fix DTensor UT
13b32d623 | BUGFIX | fix master ci bug
706c556cd | BUGFIX | fix master ci bug
94c03dc9e | BUGFIX | [shmem] fix with aclshmem
5888baa9d | BUGFIX | [shmem] fix with aclshmem
0b66ea43e | BUGFIX | [shmem] fix with aclshmem
3f372302e | REVERT | revert syncop changes
25ba98faf | REVERT | revert syncop changes
8020c856b | REVERT | revert syncop changes
3d26770f5 | BUGFIX | fix docs
66cb389e5 | BUGFIX | bugfix and apply scheduler patch
89384951c | BUGFIX | 【bugfix】[pytorch/docs]资料PROF_CONFIG_PATH.md修改简介描述
7465f3176 | BUGFIX | Fixed test_compatibility.py
1beb488ad | BUGFIX | Fixed test_compatibility.py
f82af319a | BUGFIX | Fixed test_compatibility.py
bcf9c84c0 | BUGFIX | Fixed test_compatibility.py
44620ee65 | BUGFIX | Fixed test_compatibility.py
32b8fd81a | BUGFIX | fix master inductor bug
64180f487 | BUGFIX | Fixed test_compatibility.py
5f5ea8d19 | BUGFIX | Fixed test_compatibility.py
0db512708 | BUGFIX | Fixed test_compatibility.py
9b8c5b436 | BUGFIX | Fixed test_compatibility.py
694a74cce | BUGFIX | Fixed test_compatibility.py
0bc2a875e | BUGFIX | fix p2p limitation
cdc425d3d | BUGFIX | fix p2p limitation
9b1ba6398 | BUGFIX | fix p2p limitation
bc097148c | BUGFIX | fix p2p limitation
67c9b1a00 | BUGFIX | fix p2p limitation
2e3b9d167 | BUGFIX | fix p2p limitation
c4a63049a | BUGFIX | fix p2p limitation
efa6ad424 | BUGFIX | fix p2p limitation
e6612cf04 | BUGFIX | fix p2p limitation
de4e20b1f | BUGFIX | fix gcc patch bug for A3
4d8a225ea | BUGFIX | fix gcc patch bug for A3
353e57ffd | BUGFIX | fix gcc patch bug for A3
b9d4e7b0f | BUGFIX | fix gcc patch bug for A3
4d88e0118 | BUGFIX | fix IsGteDriverVersion
4dcc0c0c2 | BUGFIX | fix gcc bug in A3
abb12bc68 | BUGFIX | fix gcc bug in A3
785e3cd2d | BUGFIX | fix gcc bug in A3 for 2.9.0
4833cf2f0 | BUGFIX | fix gcc bugs in A3
8a5b0ea30 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
df4480731 | BUGFIX | [bugfix]remove option aclgraph_report_shape
c656e58cd | BUGFIX | [bugfix]remove option aclgraph_report_shape
1dcaddabf | BUGFIX | [bugfix]remove option aclgraph_report_shape
598d02a4f | BUGFIX | [bugfix]change feature report shape log
f7315c879 | BUGFIX | [bugfix]remove option aclgraph_report_shape
874f2c20e | BUGFIX | [bugfix]remove option aclgraph_report_shape
2314a8253 | BUGFIX | [bugfix]remove option aclgraph_report_shape
797953e40 | BUGFIX | [bugfix]remove option aclgraph_report_shape
edfcfcf76 | BUGFIX | fix IsGteDriverVersion
3a9f8bb9d | BUGFIX | fix IsGteDriverVersion
1f2b8815d | BUGFIX | fix IsGteDriverVersion
4fdeb0a91 | BUGFIX | fix IsGteDriverVersion
b9ea48873 | BUGFIX | Fix shutdown process host block destruction bug
c1bc30c27 | BUGFIX | Fix shutdown process host block destruction bug
e1c2702f6 | BUGFIX | Fix shutdown process host block destruction bug
c71310d65 | BUGFIX | Fix shutdown process host block destruction bug
4c52cd49b | BUGFIX | Fix shutdown process host block destruction bug
3b16abab2 | BUGFIX | Fix shutdown process host block destruction bug
633a7a860 | BUGFIX | Fix shutdown process host block destruction bug
999800822 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
d36e6868e | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
4f66d033e | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
788916261 | BUGFIX | fix sharding strategy for npu_fusion_attention
aba8ee7c0 | BUGFIX | fix sharding strategy for npu_fusion_attention
eca259dfe | BUGFIX | fix sharding strategy for npu_fusion_attention
532f7762f | BUGFIX | fix sharding strategy for npu_fusion_attention
97ead3efa | BUGFIX | fix next_power_of_2
eb01c9656 | BUGFIX | fix next_power_of_2
c929b08e1 | BUGFIX | fix next_power_of_2
57efa0e83 | BUGFIX | fix next_power_of_2
0152af552 | BUGFIX | fix next_power_of_2
9a2791aeb | BUGFIX | fix next_power_of_2
334321b90 | BUGFIX | fix IsGteDriverVersion
e72cf7234 | BUGFIX | fix IsGteDriverVersion
cd4a1b156 | BUGFIX | fix IsGteDriverVersion
9781f9c89 | BUGFIX | fix IsGteDriverVersion
d317b1543 | BUGFIX | fix IsGteDriverVersion
9a5595ac9 | BUGFIX | fix next_power_of_2
913083da4 | BUGFIX | fix val num_power of 2 bug
31d5bf49a | BUGFIX | bug fix
eee5ce316 | BUGFIX | fix header file to match cann version
8824d6955 | BUGFIX | fix header file to match cann version
572dd5e81 | BUGFIX | fix header file to match cann version
fe1833546 | BUGFIX | fix header file to match cann version
d52f1833b | BUGFIX | fix header file to match cann version
c79d6def4 | BUGFIX | fix sharding strategy for npu_fusion_attention
e51a429b6 | BUGFIX | fix sharding strategy for npu_fusion_attention
cb005597f | BUGFIX | fix sharding strategy for npu_fusion_attention
e944fcbdb | BUGFIX | fix sharding strategy for npu_fusion_attention
488c9a0ae | BUGFIX | [fix] register strategy for npu_fusion_attention
be62cae61 | BUGFIX | [fix] fix test case in test_schedule_multiproc
ce5e9bbaa | BUGFIX | [fix] fix test case in test_schedule_multiproc
21c04c5ab | BUGFIX | [torch-v2.9.0-7.3.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
ecf46ee1f | BUGFIX | [torch-v2.8.0-7.3.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
28f7be5f8 | BUGFIX | [torch-v2.7.1-7.3.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
ac6df14fd | BUGFIX | [torch-v2.6.0-7.3.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
b26da6b9d | BUGFIX | [torch-2.6.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
7c5e50b8c | BUGFIX | [torch-2.9.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
80e96f25e | BUGFIX | [torch-2.8.0]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
9e6310780 | BUGFIX | [torch-2.7.1]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
85d2a49b9 | BUGFIX | [torch-master]修复profiler采集内存流ID数据没有aclrtStreamGetId接口兼容性问题
409ddefdd | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.6.0-7.3.0
e449a1218 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.7.1-7.3.0
7ae88b2b0 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.8.0-7.3.0
bd20aa3e2 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.9.0-7.3.0
e0938e9c3 | BUGFIX | [bugfix]revert api profiler_trace
247be491d | BUGFIX | [bugfix]revert api profiler_trace
d5718a6be | BUGFIX | [bugfix]revert api profiler_trace
62f59dbff | BUGFIX | [bugfix]revert api profiler_trace
c7ff45a33 | BUGFIX | [bugfix]revert api profiler_trace
aaedfff41 | BUGFIX | [bugfix]revert api profiler_trace
5cfc8d170 | BUGFIX | [bugfix]revert api profiler_trace
a1b7124e6 | BUGFIX | [bugfix]revert api profiler_trace
6927b61c4 | BUGFIX | [bugfix]revert api profiler_trace
f0484f937 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
be3542363 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
3dfc7ca39 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
09fa492eb | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
6dc9588a6 | BUGFIX | [fix] fix empty_like_npu faketensor issue to support _native_mha in compile
0961d04f2 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.6.0
99bd2e1db | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.7.1
82e8ff48d | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--v2.8.0
200b6e925 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--V2.9.0
4a0406549 | BUGFIX | 【bugfix】Fix when taskQueue is closed with aclgraph enabled--master
9cc603764 | BUGFIX | fix bug of storage.cpu
b857cb6ec | BUGFIX | fix bug of storage.cpu
47c1a1e51 | BUGFIX | fix bug of storage.cpu
ceb2c1a47 | BUGFIX | fix bug of storage.cpu
00fdd2235 | BUGFIX | fix bug of storage.cpu
5578cae86 | BUGFIX | fix bug of storage.cpu
c8dbbbd01 | BUGFIX | fix bug of storage.cpu
41610c183 | BUGFIX | fix bug of storage.cpu
0bbab657b | BUGFIX | fix bug of storage.cpu
db175875b | BUGFIX | fix issue1445: Explicitly include stdint.h
5cc8eafa0 | BUGFIX | fix issue1445: Explicitly include stdint.h
2da3c5d01 | BUGFIX | fix issue1445: Explicitly include stdint.h
69077215c | BUGFIX | fix issue1445: Explicitly include stdint.h
fe123a372 | BUGFIX | fix issue1445: Explicitly include stdint.h
50364683a | BUGFIX | [v2.7.1-7.3.0]fix aoe ut
f1b7c18f7 | BUGFIX | [v2.6.0-7.3.0]fix aoe ut
095cdc49e | BUGFIX | [v2.8.0-7.3.0]fix aoe ut
d457c8c18 | BUGFIX | [v2.9.0-7.3.0]fix aoe ut
3a78063d7 | BUGFIX | [master]fix aoe ut
182016ccf | BUGFIX | [2.9.0]fix aoe ut
834cf1b9f | BUGFIX | [2.8.0]fix aoe ut
13755e4e0 | BUGFIX | [2.7.1]fix aoe ut
0fd75f68c | BUGFIX | [2.6.0]fix aoe ut
313a0cf35 | BUGFIX | fix_header_file_reference_issue
7618e05ca | BUGFIX | fix_header_file_reference_issue
f685919a2 | BUGFIX | Fix memset not found
b9f914b9e | BUGFIX | Fix memset not found
e3b16aade | BUGFIX | fix inductor ci
43c2a628e | BUGFIX | Fix plog print in checkHcclComms
b64ce8479 | BUGFIX | Fix plog print in checkHcclComms
d7aded94c | BUGFIX | Fix plog print in checkHcclComms
c05e5bf54 | BUGFIX | Fix plog print in checkHcclComms
d717ca8ad | BUGFIX | Fix plog print in checkHcclComms
5da928113 | BUGFIX | Fix plog print in checkHcclComms
d38d1c540 | BUGFIX | Fix plog print in checkHcclComms
c9671953b | BUGFIX | Fix plog print in checkHcclComms
4721dea7b | BUGFIX | Fix plog print in checkHcclComms
003752f5f | BUGFIX | 【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
fca0cd7ad | BUGFIX | 【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
9336717ab | BUGFIX | 【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
f9fdb1aff | BUGFIX | 【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
29ec3eb7d | BUGFIX | 【v2.9.0】【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
d1a0c1b1b | BUGFIX | 【v2.8.0】【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
1961e6bfe | BUGFIX | 【v2.7.1】【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
d53591376 | BUGFIX | 【v2.6.0】【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
1ce96b899 | BUGFIX | 【v2.1.0】【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
6b134373a | REVERT | revert torchair commitid to 20251110,2.1 version no need to update。
0fc22c75a | BUGFIX | 【bugfix】Fixed a bug that could cause an exception when calling prof backend stop without having invoked the profile object after initialization.
7644f42b5 | BUGFIX | [torch_v2.9]修复动态profiling异常配置刷屏问题
c262d3be1 | BUGFIX | [torch_v2.8]修复动态profiling异常配置刷屏问题
0bc187fab | BUGFIX | [torch_v2.7]修复动态profiling异常配置刷屏问题
e4834758d | BUGFIX | [torch_v2.6]修复动态profiling异常配置刷屏问题
00572e020 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
a27b653b1 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
f06512f76 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
132d28068 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
b059af929 | BUGFIX | Fix process synchronization issues in distributed communication test cases
43d5b99eb | BUGFIX | Fix process synchronization issues in distributed communication test cases
2d9fd3b6a | BUGFIX | Fix process synchronization issues in distributed communication test cases
5dd80ebfe | BUGFIX | Fix process synchronization issues in distributed communication test cases
c7a504a9a | BUGFIX | Fix process synchronization issues in distributed communication test cases
a15a7867c | BUGFIX | Fix process synchronization issues in distributed communication test cases
e9052ad36 | BUGFIX | Fix process synchronization issues in distributed communication test cases
4a95417c8 | BUGFIX | Fix process synchronization issues in distributed communication test cases
cfec83e96 | BUGFIX | Fix process synchronization issues in distributed communication test cases
2a090d9b6 | BUGFIX | Fix process synchronization issues in distributed communication test cases
fa9394588 | BUGFIX | [torch_2.9]修复动态profiling异常配置刷屏问题
4d809214a | BUGFIX | [torch_2.8]修复动态profiling异常配置刷屏问题
b13c1a535 | BUGFIX | [torch_2.7]修复动态profiling异常配置刷屏问题
8dffceb94 | BUGFIX | [torch_2.6]修复动态profiling异常配置刷屏问题
b4867a3db | BUGFIX | [torch_master]修复动态profiling异常配置刷屏问题
00774a137 | BUGFIX | [inductor][ascend_npu_ir] fix: host memory leak while run in dynamic shape.
45a7d16a7 | BUGFIX | fix(torch-event): add elapsedTime, queryStream and synchronizeStream methods
39ca660a5 | BUGFIX | fix(torch-event): add elapsedTime, queryStream and synchronizeStream methods
d78fcb44a | BUGFIX | fix(torch-event): add elapsedTime, queryStream and synchronizeStream methods
31d6136fc | BUGFIX | fix(torch-event): add elapsedTime, queryStream and synchronizeStream methods
d233178a9 | BUGFIX | fix(torch-event): add elapsedTime, queryStream and synchronizeStream methods
2cac358b2 | BUGFIX | fix test unit bug in test_torch_mlir.py
d1610913a | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
3dd9b3adc | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
201dc9a1c | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
72fff6022 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
dd2ca86e9 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
4a2b6d7b1 | BUGFIX | [RANKTABLE] Fix p2p ranks error in ranktable
77bc22cc2 | BUGFIX | TORCH MAIN SYNC : add update_wrapped_number (bugfix to ForwardADWithScalars)
3867857f4 | BUGFIX | fix code issue. Variables must be declared before use
1341fbe93 | BUGFIX | [fix] Fix NPUHooksInterface::getDeviceFromPtr with aclrtPointerGetAttributes
e047e1285 | BUGFIX | [fix] Fix NPUHooksInterface::getDeviceFromPtr with aclrtPointerGetAttributes
705f4d90b | BUGFIX | [fix] Fix NPUHooksInterface::getDeviceFromPtr with aclrtPointerGetAttributes
9d1413769 | BUGFIX | [fix] Fix NPUHooksInterface::getDeviceFromPtr with aclrtPointerGetAttributes
c018e741b | BUGFIX | Fixed skipIfUnsupportMultiNPU
709a6f2bd | BUGFIX | Fixed skipIfUnsupportMultiNPU
48ba1315c | BUGFIX | Fixed skipIfUnsupportMultiNPU
f7b97d633 | BUGFIX | Fixed skipIfUnsupportMultiNPU
0962e6e4c | BUGFIX | Fixed skipIfUnsupportMultiNPU
df2a02dcf | BUGFIX | Fixed skipIfUnsupportMultiNPU
4db49e496 | BUGFIX | [bugfix] Add proper handling for view and factory function for csan, and supplement the corresponding test cases.
52880af8c | BUGFIX | [bugfix] Add proper handling for view and factory function for csan, and supplement the corresponding test cases.
2399d5b1f | BUGFIX | [bugfix] Add proper handling for view and factory function for csan, and supplement the corresponding test cases.
13cf547b9 | BUGFIX | [bugfix] Add proper handling for view and factory function for csan, and supplement the corresponding test cases.
f03f8d66a | BUGFIX | [bugfix] Add proper handling for view and factory function for csan, and supplement the corresponding test cases.
3b78acbcc | BUGFIX | fix code issue. Variables must be declared before use
5883d8a05 | BUGFIX | fix code issue. Variables must be declared before use
4eed62202 | BUGFIX | fix code issue. Variables must be declared before use
2612872ce | BUGFIX | fix code issue. Variables must be declared before use
2f2f4dc0b | BUGFIX | fix code issue. Variables must be declared before use
05b154336 | BUGFIX | [RANKTABLE] Fix with P2P group
eb7844b92 | BUGFIX | [RANKTABLE] Fix with P2P group
f20aa3d8e | BUGFIX | [RANKTABLE] Fix with P2P group
7ee173d02 | BUGFIX | [RANKTABLE] Fix with P2P group
cbdc376ed | BUGFIX | [RANKTABLE] Fix with P2P group
d000588fd | BUGFIX | [RANKTABLE] Fix with P2P group
be7f213a8 | BUGFIX | fix the circular dependency issue & add ut
cb9a6dbde | BUGFIX | Fixed failed testcases
957721c81 | BUGFIX | Fixed failed testcases
4db4167c0 | BUGFIX | Fixed failed testcases
a95855310 | BUGFIX | Fixed failed testcases
73a46b9f4 | BUGFIX | [torch_2.9.0]修复profiler cann parser校验CANN侧文件未try-except问题
52e1fb617 | BUGFIX | [torch_2.8.0]修复profiler cann parser校验CANN侧文件未try-except问题
2f18354a6 | BUGFIX | [torch_2.7.1]修复profiler cann parser校验CANN侧文件未try-except问题
a61af8585 | BUGFIX | [torch_2.6.0]修复profiler cann parser校验CANN侧文件未try-except问题
9501617aa | BUGFIX | [torch_2.1.0]修复profiler cann parser校验CANN侧文件未try-except问题
fa4250ebb | BUGFIX | [torch_master]修复profiler cann parser校验CANN侧文件未try-except问题
ccfc91ee3 | BUGFIX | [bug fix]Delete unnecessary function
1618eff7c | BUGFIX | [bug fix]Delete unnecessary function
3f842e4c8 | BUGFIX | [bug fix]Delete unnecessary function
898b7f502 | BUGFIX | [bug fix]Delete unnecessary function
10dfafc1f | BUGFIX | [bug fix]Delete unnecessary function
ba62134a8 | BUGFIX | [bug fix]Delete unnecessary function
aeefb263e | BUGFIX | [fix] with_comms can't set self.device in 2.8.0+ bugfix
192d0fe7b | BUGFIX | Fixed failed tests
64389feec | BUGFIX | Fixed failed tests
d72b61d80 | BUGFIX | Fixed failed tests
788179f82 | BUGFIX | Fixed failed tests
28054b75f | BUGFIX | Fixed failed tests
6a09c4cce | BUGFIX | Fixed failed tests
ffb503304 | BUGFIX | fix the circular dependency issue & add ut
771103cd7 | BUGFIX | fix the circular dependency issue & add ut
f68a95a64 | BUGFIX | fix the circular dependency issue & add ut
af6d8a4d9 | BUGFIX | fix the circular dependency issue & add ut
737202b33 | BUGFIX | [UT] Fix with distributed ut, because PyTorch does not automatically set seeds.
83f6bf8f7 | BUGFIX | [security]fix 'new' and 'stoi'
72005560c | BUGFIX | [security]fix 'new' and 'stoi'
568fa7383 | BUGFIX | [security]fix 'new' and 'stoi'
f01285d41 | BUGFIX | [security]fix 'new' and 'stoi'
102c95455 | BUGFIX | [security]fix 'new' and 'stoi'
d80ec9c8b | BUGFIX | [security]fix 'new' and 'stoi'
4f470aca4 | BUGFIX | [bugfix]update torch_npu_schema.json
cd5ee8cb2 | BUGFIX | [bugfix]update torch_npu_schema.json
c8c5deead | BUGFIX | [bugfix]update torch_npu_schema.json
fb5de229b | BUGFIX | [bugfix]update torch_npu_schema.json
fe3c725ea | BUGFIX | [bugfix]update torch_npu_schema.json
27e963f4b | BUGFIX | [bugfix]update torch_npu_schema.json
19461e12e | BUGFIX | [SHMEM]Fix with multi host.
7e45549fa | BUGFIX | Fixed typeError
fde7a65ab | BUGFIX | [SHMEM]fix with GetShmemSymmetricSize
38eb0a0a6 | BUGFIX | 增加PA和MLA算子更新，并同时修复MLA更新时weak ref引发的问题
be5403ca8 | BUGFIX | 增加PA和MLA算子更新，并同时修复MLA更新时weak ref引发的问题
da06bf49b | BUGFIX | 增加PA和MLA算子更新，并同时修复MLA更新时weak ref引发的问题
9ed244e6f | BUGFIX | 增加PA和MLA算子更新，并同时修复MLA更新时weak ref引发的问题
d8c00fa30 | BUGFIX | 增加PA和MLA算子更新，并同时修复MLA更新时weak ref引发的问题
5189313b5 | BUGFIX | [IPC]: fix with meta tensor
f4ffe9b92 | BUGFIX | [IPC]: fix with meta tensor
38d533999 | BUGFIX | [IPC]: fix with meta tensor
64aded1e7 | BUGFIX | [IPC]: fix with meta tensor
f91b26ae1 | BUGFIX | [IPC]: fix with meta tensor
bc51ab84c | BUGFIX | [IPC]: fix with meta tensor
9ae698c45 | BUGFIX | [IPC]: fix with meta tensor
5cbbf7528 | BUGFIX | [IPC]: fix with meta tensor
12a6dcea6 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
28b0c5382 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
fa1d403fa | BUGFIX | fix empty npu_desc in aclgraph
35ee52898 | BUGFIX | fix empty npu_desc in aclgraph
f036e8701 | BUGFIX | fix empty npu_desc in aclgraph
beccd204a | BUGFIX | [inductor][ascend_npu_ir] fix: memory-leak issue while run in dynamic shape.
d7f1d9e2d | BUGFIX | Fixed typeError
81872688e | BUGFIX | Fixed typeError
afd035cd7 | BUGFIX | Fixed typeError
d9ef9038c | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
186968273 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
6a9eb8c47 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
dd892d5f8 | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
bd8d61eea | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
9ac3362ed | BUGFIX | [onnx] fix group_norm_silu and rotary_mul onnx api
d886740a3 | BUGFIX | [BugFix]add driver version check
e0b833064 | BUGFIX | [BugFix]add driver version check
8e73bff8c | BUGFIX | [BugFix]add driver version check
451b9760b | BUGFIX | [BugFix]add driver version check
3f3e3333b | BUGFIX | [BugFix]add driver version check
5f9c30c39 | BUGFIX | [BugFix]add driver version check
898aff263 | BUGFIX | [BugFix]add driver version check
e89d11528 | BUGFIX | [BugFix]add driver version check
7787cfc0b | BUGFIX | [BugFix]add driver version check
de4db7a2d | BUGFIX | [BugFix]add driver version check
94eb3aebe | BUGFIX | Fix the issue where torch.amp.autocast does not work
9a520050d | BUGFIX | Fix the issue where torch.amp.autocast does not work
8725693dc | BUGFIX | Fix the issue where torch.amp.autocast does not work
35cc5f3d6 | BUGFIX | Fix the issue where torch.amp.autocast does not work
01442bd42 | BUGFIX | [recovery]Fix with watchdog recovery exception.
377526a3a | BUGFIX | [recovery]Fix with watchdog recovery exception.
59948b315 | BUGFIX | [recovery]Fix with watchdog recovery exception.
8cdaa39ff | BUGFIX | [recovery]Fix with watchdog recovery exception.
8c16e9a91 | BUGFIX | [recovery]Fix with watchdog recovery exception.
a8ad0cd6b | BUGFIX | [recovery]Fix with watchdog recovery exception.
8bace4284 | BUGFIX | [recovery]Fix with watchdog recovery exception.
d5ccccaa0 | BUGFIX | [recovery]Fix with watchdog recovery exception.
b1161fd9a | BUGFIX | [recovery]Fix with watchdog recovery exception.
3c496912c | BUGFIX | [recovery]Fix with watchdog recovery exception.
d1deab965 | BUGFIX | Transfer_to_npu PipelineStage bugfix
4d9efe967 | BUGFIX | Transfer_to_npu PipelineStage bugfix
d8a4486aa | BUGFIX | Transfer_to_npu PipelineStage bugfix
7a648f602 | BUGFIX | Transfer_to_npu PipelineStage bugfix
7855d9f0a | BUGFIX | Fix the issue where torch.amp.autocast does not work
d45aa7f64 | BUGFIX | Fix the issue where torch.amp.autocast does not work
b7cb40ceb | BUGFIX | Fix the issue where torch.amp.autocast does not work
71f61da9b | BUGFIX | Fix the issue where torch.amp.autocast does not work
69550dfc2 | BUGFIX | Fix the issue where torch.amp.autocast does not work
fee27460d | BUGFIX | Fix the issue where torch.amp.autocast does not work
e3268ca5f | BUGFIX | Fix the issue where torch.amp.autocast does not work
441df77c6 | BUGFIX | Transfer_to_npu PipelineStage bugfix
39bc155b0 | BUGFIX | Transfer_to_npu PipelineStage bugfix
c5e0af8ea | BUGFIX | Transfer_to_npu PipelineStage bugfix
d038a508a | BUGFIX | Transfer_to_npu PipelineStage bugfix
7f46e1979 | BUGFIX | Transfer_to_npu PipelineStage bugfix
2cc06d0d1 | BUGFIX | Transfer_to_npu PipelineStage bugfix
8c8352a9e | BUGFIX | Transfer_to_npu PipelineStage bugfix
a78f7121c | BUGFIX | fix frozen bug for using aclop and deterministic
1fcf77c3b | BUGFIX | fix frozen bug for using aclop and deterministic
03cb322df | BUGFIX | fix frozen bug for using aclop and deterministic
39cfa9383 | BUGFIX | fix frozen bug for using aclop and deterministic
2820b8b04 | BUGFIX | fix frozen bug for using aclop and deterministic
1f20cdfe1 | BUGFIX | fix frozen bug for using aclop and deterministic
a389c71f5 | BUGFIX | fix frozen bug for using aclop and deterministic
cc069beb4 | BUGFIX | fix frozen bug for using aclop and deterministic
f73fc7390 | BUGFIX | fix frozen bug for using aclop and deterministic
269f28504 | BUGFIX | fix frozen bug for using aclop and deterministic
72136f3a7 | BUGFIX | fix frozen bug for using aclop and deterministic
79639fdc4 | BUGFIX | fix frozen bug for using aclop and deterministic
c3b1e3712 | BUGFIX | [FIX] Initialize WatchdogStatus object
249262281 | BUGFIX | fix frozen bug for using aclop and deterministic
70c28da7b | BUGFIX | fix frozen bug for using aclop and deterministic
5b19a95ad | BUGFIX | fix frozen bug for using aclop and deterministic
a2c6fd8df | BUGFIX | fix frozen bug for using aclop and deterministic
9fd311960 | BUGFIX | fix frozen bug for using aclop and deterministic
5b11889c8 | BUGFIX | fix frozen bug for using aclop and deterministic
77ae68115 | BUGFIX | fix frozen bug for using aclop and deterministic
4ffb08d69 | BUGFIX | fix frozen bug for using aclop and deterministic
103b3ecc0 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
26a240370 | BUGFIX | [FIX] Initialize WatchdogStatus object
303cff222 | BUGFIX | Rollback for bugfix
57f1c3373 | BUGFIX | Rollback for bugfix
f68270c37 | BUGFIX | Rollback for bugfix
b744aa68b | BUGFIX | Rollback for bugfix
e03caca23 | BUGFIX | [FIX] Initialize WatchdogStatus object
d2e4d933a | BUGFIX | [FIX] Initialize WatchdogStatus object
2e9d43890 | BUGFIX | [FIX] Initialize WatchdogStatus object
f8a862ac0 | BUGFIX | [FIX] Initialize WatchdogStatus object
8b54c6cab | BUGFIX | [FIX] Initialize WatchdogStatus object
e8f76c5a5 | BUGFIX | [FIX] Initialize WatchdogStatus object
d443a8646 | BUGFIX | [FIX] Initialize WatchdogStatus object
e3549d3e9 | BUGFIX | [FIX] Initialize WatchdogStatus object
e4a6ae5f8 | BUGFIX | fix device mesh ut
f266a285d | BUGFIX | fix device mesh ut
904398b0e | BUGFIX | fix device mesh ut
9632bfc3f | BUGFIX | fix device mesh ut
213f96e85 | BUGFIX | fix device mesh ut
2bbd0418c | BUGFIX | fix device mesh ut
68dce0887 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
4b9f9edaf | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
c7336f723 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
f867c0392 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
4306b24ef | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
547f5d384 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
59573af1b | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
8fb7c5275 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
1043cea29 | BUGFIX | fix with uce error, switch to NPU_CHECK_ERROR
902c98ad4 | BUGFIX | [fix] with_comms can't set self.device in 2.8.0 bugfix
d7b0d4d9a | REVERT | Revert re package that was deleted by mistake
833c426c7 | REVERT | Revert re package that was deleted by mistake
b885e9763 | BUGFIX | [inductor][ascend_npu_ir] fix bugs: api change in v2.7.1.
1cf77447f | BUGFIX | [inductor][ascend_npu_ir] fix bugs: api change in v2.7.1.
2b3fecd70 | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
40dce02b3 | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
a26ffba94 | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
767a7d031 | BUGFIX | transfer_to_npu DeviceMesh bugfix
3e0dd348a | BUGFIX | transfer_to_npu DeviceMesh bugfix
460f01b65 | BUGFIX | transfer_to_npu DeviceMesh bugfix
ee1870167 | BUGFIX | Fixed import problems on py3.11
dc4bb5e30 | BUGFIX | Fixed import problems on py3.11
81aba0c87 | BUGFIX | Fixed import problems on py3.11
8378703e9 | BUGFIX | Fixed import problems on py3.11
e56cad147 | BUGFIX | Fixed import problems on py3.11
38fffc29d | BUGFIX | Fixed import problems on py3.11
2836f6c12 | BUGFIX | Fixed import problems on py3.11
513f301ed | BUGFIX | Fixed import problems on py3.11
b27e5f091 | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
86c69647c | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
4d2fb4e03 | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
9bd43a258 | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
8e55eb62a | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
fbee5944a | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
0555aef8b | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
f6b6bada5 | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
36c76bd70 | REVERT | revent use the storage for record stream and erase stream in ProcessGroupHCCL
cf5eec50d | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
c921e054c | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
16bc32f02 | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
c82165990 | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
1e3340d69 | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
6f5136cab | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
b3c9d4f84 | BUGFIX | 【bugfix】Fix the bug to adapt the torch Event
c443bb314 | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
82a2c01e3 | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
f540db14a | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
428568d7c | BUGFIX | [fix] add _npu_dtype_cast_backward sharding strategy
e532498ff | BUGFIX | transfer_to_npu DeviceMesh bugfix
6a9ae966e | BUGFIX | transfer_to_npu DeviceMesh bugfix
72fb04b1e | BUGFIX | transfer_to_npu DeviceMesh bugfix
d086dc709 | BUGFIX | transfer_to_npu DeviceMesh bugfix
51116a73f | BUGFIX | transfer_to_npu DeviceMesh bugfix
ed39d74c1 | BUGFIX | fix ut
1f8a3a343 | BUGFIX | fix ut
2a0f89be0 | BUGFIX | fix ut
bd3fb0762 | BUGFIX | fix ut
e74476bbb | BUGFIX | fix ut
83260b4f8 | BUGFIX | fix default value of ACL_PRECISION_MODE
6774b53cc | BUGFIX | fix default value of ACL_PRECISION_MODE
0fbf6df60 | BUGFIX | fix default value of ACL_PRECISION_MODE
2a6c49deb | BUGFIX | fix default value of ACL_PRECISION_MODE
67747b155 | BUGFIX | fix default value of ACL_PRECISION_MODE
551cbb598 | BUGFIX | fix default value of ACL_PRECISION_MODE
11cf2d7c5 | BUGFIX | fix default value of ACL_PRECISION_MODE
6059f22dc | BUGFIX | fix default value of ACL_PRECISION_MODE
94ac4aaff | BUGFIX | fix default value of ACL_PRECISION_MODE
1bc675ec3 | BUGFIX | fix some error when using triton
8e546c7aa | BUGFIX | fix some error when using triton
1f2febca0 | BUGFIX | fix some error when using triton
81d01ab4d | BUGFIX | fix some error when using triton
4c27f6749 | BUGFIX | fix some error when using triton
0e85c7849 | BUGFIX | fix some error when using triton
85a0ccdbd | BUGFIX | fix some error when using triton
828f1806d | BUGFIX | fix some error when using triton
6df8a784a | BUGFIX | fix some error when using triton
82f01d06d | BUGFIX | fix fia lse ut
72866c1ee | BUGFIX | fix fia lse ut
4115acbb1 | BUGFIX | fix fia lse ut
c0426c7eb | BUGFIX | fix fia lse ut
4de195620 | BUGFIX | fix error for get ACL_OP_INIT_MODE
9edd24949 | BUGFIX | fix error for get ACL_OP_INIT_MODE
b964e42f0 | BUGFIX | Fixed import problems on py3.11
d131c9a99 | BUGFIX | fix empty tensor
ec60174ce | BUGFIX | fix empty npu_desc in aclgraph
e69a7cbf5 | BUGFIX | fix empty npu_desc in aclgraph
f391533a6 | BUGFIX | fix empty npu_desc in aclgraph
5e57d9dd8 | BUGFIX | fix empty npu_desc in aclgraph
1b8151cc6 | BUGFIX | [IPC]fix with nullptr call
f877a3e6f | BUGFIX | [IPC]fix with nullptr call
a99625624 | BUGFIX | [IPC]fix with nullptr call
c70d8433a | BUGFIX | [IPC]fix with nullptr call
afba2f179 | BUGFIX | [IPC]fix with nullptr call
868567887 | BUGFIX | [IPC]fix with nullptr call
3809f39bc | BUGFIX | [IPC]fix with nullptr call
fb610565a | BUGFIX | [IPC]fix with nullptr call
c4fe9ee14 | BUGFIX | [IPC]fix with nullptr call
24a31844c | BUGFIX | [IPC]fix with nullptr call
f2e280ea6 | BUGFIX | 【bugfix】修改sanitizer的apply应用至开启流间竞争检测时
2eb2048fc | BUGFIX | 【bugfix】修改sanitizer的apply应用至开启流间竞争检测时
c568e44c7 | BUGFIX | 【bugfix】修改sanitizer的apply应用至开启流间竞争检测时
3aeeea99c | BUGFIX | 【bugfix】修改sanitizer的apply应用至开启流间竞争检测时
d31ac6096 | BUGFIX | 【bugfix】修改sanitizer的apply应用至开启流间竞争检测时
ba4812b79 | BUGFIX | Fixed a storage bug
07359a1a5 | BUGFIX | Fixed a storage bug
7ee5d115d | BUGFIX | [inductor] 修复推荐领域pattern问题
733b24f4f | BUGFIX | [SilentCheck] fix with DTensor of grad
b624e1b00 | BUGFIX | [SilentCheck] fix with DTensor of grad
f65fa375d | BUGFIX | [SilentCheck] fix with DTensor of grad
112a0d893 | BUGFIX | [SilentCheck] fix with DTensor of grad
21a7a7cbd | BUGFIX | [SilentCheck] fix with DTensor of grad
14fdbd02f | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
adcbcc26b | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
4e14f3a0e | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
7aa6f9234 | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
21fad641a | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
c59e6e7fa | BUGFIX | [Silent check] Fix inplace problem with reinforcement learning.
8e0d9916b | BUGFIX | [torch_master] 修复db场景下step trace time报错问题
d6547dd0c | BUGFIX | [torch_2.1.0] 修复db场景下step trace time报错问题
9c4289630 | BUGFIX | [torch_2.6.0] 修复db场景下step trace time报错问题
10465763c | BUGFIX | [torch_2.7.1] 修复db场景下step trace time报错问题
7c94f8f7b | BUGFIX | [torch_2.8.0] 修复db场景下step trace time报错问题
128d7e3d0 | BUGFIX | fix logic of default value for allow_hf32
5d4776d23 | BUGFIX | fix logic of default value for allow_hf32
87e24d31d | BUGFIX | fix logic of default value for allow_hf32
7bbcfd5ae | BUGFIX | fix logic of default value for allow_hf32
51642b217 | BUGFIX | fix logic of default value for allow_hf32
7a0e21491 | BUGFIX | [PROFILING] fix err of unable to create STEP_TIME table while only CPU enabled
cda3cae55 | BUGFIX | [PROFILING] fix err of unable to create STEP_TIME table while only CPU enabled
4608953c8 | BUGFIX | [PROFILING] fix err of unable to create STEP_TIME table while only CPU enabled
9f06fb064 | BUGFIX | [PROFILING] fix err of unable to create STEP_TIME table while only CPU enabled
991c39fbb | BUGFIX | [PROFILING] fix err of unable to create STEP_TIME table while only CPU enabled
0fd3db0b0 | BUGFIX | fix space error
5a7912953 | BUGFIX | fix space error
9bbb3b50e | BUGFIX | fix space error
d85cc3655 | BUGFIX | fix space error
c41fa1f0b | BUGFIX | fix space error
1b04447a8 | BUGFIX | !24576 [Profiler]fix thread gil Merge pull request !24576 from hhz886/v2.8.0
76f43ebf6 | BUGFIX | !24578 [Profiler]fix thread gil Merge pull request !24578 from hhz886/v2.6.0
0daba0e58 | BUGFIX | !24339 [Profiler]fix thread gil Merge pull request !24339 from hhz886/v2.1.0
fd25bbd40 | BUGFIX | !24575 [Profiler]fix thread gil Merge pull request !24575 from hhz886/master
12f8d82cd | BUGFIX | !24577 [Profiler]fix thread gil Merge pull request !24577 from hhz886/v2.7.1
8b5662bcf | BUGFIX | !24772 [inductor] fix dump_fx_graph bugs in lowering_fx Merge pull request !24772 from 杜承昆/lowering_fix_26
e53f69457 | BUGFIX | !24773 [inductor] fix dump_fx_graph bugs in lowering_fx Merge pull request !24773 from 杜承昆/lowering_fix_27
f687bdc28 | BUGFIX | !24647 [bugfix]add fsdp patch for supporting host inputs Merge pull request !24647 from zhangqiongwen/v2.7.1_add_fsdp_patch
dddf43d3d | BUGFIX | !24648 [bugfix]add fsdp patch for supporting host inputs Merge pull request !24648 from zhangqiongwen/v2.6.0_add_fsdp_patch
374288af0 | BUGFIX | !24703 Fixed test_set_snapshort_fn Merge pull request !24703 from yuhaiyan/v2.1.0-dev1
604964eaf | BUGFIX | !24286 [inductor]Fix triton codegen for dynamic shape Merge pull request !24286 from Xuan Peng/inductor-ds
dbe7a336c | BUGFIX | !24598 [Bugfix]NSLB-DP config err Merge pull request !24598 from SCh-zx/bug28
a6e4bbbbc | BUGFIX | !24599 [Bugfix]NSLB-DP config err Merge pull request !24599 from SCh-zx/bugm
557b7b9b7 | BUGFIX | !24597 [Bugfix]NSLB-DP config err Merge pull request !24597 from SCh-zx/bug27
393637b23 | BUGFIX | !24596 [Bugfix]NSLB-DP config err Merge pull request !24596 from SCh-zx/bug26
55ab952c7 | BUGFIX | !24595 [Bugfix]NSLB-DP config err Merge pull request !24595 from SCh-zx/bug25
a76cc2bb2 | BUGFIX | !24594 [Bugfix]NSLB-DP config err Merge pull request !24594 from SCh-zx/bug21
a94661424 | BUGFIX | !24547 1. remove(temporary) aten.sin, aten.cos from lowering list Merge pull request !24547 from zhuceHW/bugfix
8bb075835 | BUGFIX | !24520 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24520 from SCh-zx/hsm
a9a107dcf | BUGFIX | !24519 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24519 from SCh-zx/hs28
4faba86c5 | BUGFIX | !24387 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24387 from SCh-zx/hs21
2d23bdcaa | BUGFIX | !24413 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24413 from SCh-zx/hs25
87ab6514c | BUGFIX | !24307 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24307 from SCh-zx/hs26
c8b6a6003 | BUGFIX | !24518 [Bugfix] aclrtMemcpyAsyncWithConditon for d2h copy Merge pull request !24518 from SCh-zx/hs27
256c01976 | BUGFIX | !24444 [torch_2.6.0]修复profiler 解析相关问题 Merge pull request !24444 from hhz886/bug_fix_db_v2.6.0
6a949ae01 | BUGFIX | !24440 [torch_2.1.0]修复profiler 解析相关问题 Merge pull request !24440 from hhz886/hhztest
fca6ef028 | BUGFIX | !24443 [torch_2.5.1]修复profiler 解析相关问题 Merge pull request !24443 from hhz886/bug_fix_db_v2.5.1
6b7387c42 | BUGFIX | !24445 [torch_2.7.1]修复profiler 解析相关问题 Merge pull request !24445 from hhz886/bug_fix_db_v2.7.1
238135598 | BUGFIX | !24446 [torch_2.8.0]修复profiler 解析相关问题 Merge pull request !24446 from hhz886/bug_fix_db_v2.8.0
198982e4a | BUGFIX | !24439 [torch_master]修复profiler 解析相关问题 Merge pull request !24439 from yuliangbin/bug_fix_db
79ecfa94d | BUGFIX | !24329 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24329 from yuhaiyan/v2.1.0-dev1
e35fc4764 | BUGFIX | !24431 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24431 from yuhaiyan/v2.5.1-dev3
222a984c0 | BUGFIX | !24433 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24433 from yuhaiyan/v2.6.0-dev3
dbe0f0bf7 | BUGFIX | !24434 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24434 from yuhaiyan/v2.7.1-dev3
716340ef9 | BUGFIX | !24435 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24435 from yuhaiyan/v2.8.0-dev3
1b258ef0f | BUGFIX | !24436 Add testcases for driver version and fixed the test_fake_tensor.py Merge pull request !24436 from yuhaiyan/master-dev3
8ae87c5e8 | BUGFIX | !24382 [bugfix]]add _npu_dtype_cast sharding strategy Merge pull request !24382 from zhangqiongwen/master_add_npu_format_cast_sharding_strategy
d3423e588 | BUGFIX | !24384 [bugfix]]add _npu_dtype_cast sharding strategy Merge pull request !24384 from zhangqiongwen/v2.8.0_add_npu_format_cast_sharding_strategy
0a6c135b7 | BUGFIX | !24385 [bugfix]]add _npu_dtype_cast sharding strategy Merge pull request !24385 from zhangqiongwen/v2.7.1_add_npu_format_cast_sharding_strategy
be4f75b7a | BUGFIX | !24386 [bugfix]]add _npu_dtype_cast sharding strategy Merge pull request !24386 from zhangqiongwen/v2.6.0_add_npu_format_cast_sharding_strategy
eef1d5ae6 | BUGFIX | !24140 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !24140 from weili10/v2.6.0-7.1.0
31c6152c6 | BUGFIX | !24139 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !24139 from weili10/v2.5.1-7.1.0
ef59fd4b5 | BUGFIX | !24138 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !24138 from weili10/v2.1.0-7.1.0
041062cf1 | BUGFIX | !23528 [inductor]add inductor test Merge pull request !23528 from wl1259/inductor_saowang_bug_fix
e75616e0b | BUGFIX | !24299 [Inductor] Inductor no more need decomposition op overload list, remove it Merge pull request !24299 from zhuceHW/bugfix
d3f87cd3b | BUGFIX | !24095 logging: Fix the core when the process is being destructed. Merge pull request !24095 from 王超/v2.99.0_taskqueue1
c4866cac6 | BUGFIX | !24094 logging: Fix the core when the process is being destructed. Merge pull request !24094 from 王超/v2.8.0_taskqueue1
e5e270539 | BUGFIX | !24093 logging: Fix the core when the process is being destructed. Merge pull request !24093 from 王超/v2.7.0_taskqueue1
5054b5702 | BUGFIX | !24092 logging: Fix the core when the process is being destructed. Merge pull request !24092 from 王超/v2.6.0_taskqueue1
34e9aac21 | BUGFIX | !24091 logging: Fix the core when the process is being destructed. Merge pull request !24091 from 王超/v2.5.0_taskqueue1
fe195b067 | BUGFIX | !24090 logging: Fix the core when the process is being destructed. Merge pull request !24090 from 王超/v2.1.0_taskqueue1
f5dcbb8a9 | BUGFIX | !24046 [AOTInductor]Fix bug case when launch triton kernel DDR address out of range error Merge pull request !24046 from zhuceHW/bugfix
76913c983 | BUGFIX | !24045 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !24045 from weili10/v2.1.0-7.0.0
34f04920f | BUGFIX | !23996 [bugfix] resolve deadlock and resource inconsistent problem Merge pull request !23996 from weili10/v2.4.0-7.0.0
1fbb06aa1 | BUGFIX | !23938 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !23938 from weili10/v2.8.0
2a478c985 | BUGFIX | !23940 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !23940 from weili10/master
3f63621bc | BUGFIX | !23935 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !23935 from weili10/v2.7.1
8bb6c26ad | BUGFIX | !23933 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !23933 from weili10/v2.6.0
593697137 | BUGFIX | !23932 [bugfix] resolve deadlock and resource inconsistent problem Merge pull request !23932 from weili10/v2.5.1
174650f13 | BUGFIX | !23931 [bugfix] resolve deadlock and resource inconsistent problem Merge pull request !23931 from weili10/v2.4.0
ff523b7cb | BUGFIX | !23930 [bugfix] resolve deadlock and resource inconsistent problem Merge pull request !23930 from weili10/v2.3.1
b744cca34 | BUGFIX | !23928 [bugfix] resolve deadlock and device inconsistent problem. Merge pull request !23928 from weili10/v2.1.0
c1b4e4ded | BUGFIX | !23957 [torch_2.1.0] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23957 from yuliangbin/dy_bug_fix_818_2.1.0
a306d60f3 | BUGFIX | !23958 [torch_2.5.1] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23958 from yuliangbin/dy_bug_fix_818_2.5.1
81f44fbe4 | BUGFIX | !23959 [torch_2.6.0] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23959 from yuliangbin/dy_bug_fix_818_2.6.0
8ce5e5906 | BUGFIX | !23960 [torch_2.7.1] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23960 from yuliangbin/dy_bug_fix_818_2.7.1
4f5bd6d15 | BUGFIX | !23961 [torch_2.8.0] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23961 from yuliangbin/dy_bug_fix_818_2.8.0
ad3f9ebcc | BUGFIX | !23846 [torch_master] 动态profiling增加配置文件相关问题的日志打屏 Merge pull request !23846 from yuliangbin/dy_bug_fix
16fb415d7 | BUGFIX | !23881 [Bugfix]d2h copy pin_mem fix Merge pull request !23881 from zhangqiongwen/v2.1.0_fix_copy_ptr
d558488c5 | BUGFIX | !23882 [Bugfix]d2h copy pin_mem fix Merge pull request !23882 from zhangqiongwen/v2.5.1_fix_copy_ptr
410df9e51 | BUGFIX | !23883 [Bugfix]d2h copy pin_mem fix Merge pull request !23883 from zhangqiongwen/v2.6.0_fix_copy_ptr
2d200de86 | BUGFIX | !23884 [Bugfix]d2h copy pin_mem fix Merge pull request !23884 from zhangqiongwen/v2.7.1_fix_copy_ptr
5b357efea | BUGFIX | !23885 [Bugfix]d2h copy pin_mem fix Merge pull request !23885 from zhangqiongwen/v2.8.0_fix_copy_ptr
5112c08df | BUGFIX | !23811 [Bugfix]d2h copy pin_mem fix Merge pull request !23811 from zhangqiongwen/master_fix_copy_ptr
309c3ccab | BUGFIX | !23843 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23843 from kuhn/v2.1.0_fix
6b47857fd | BUGFIX | !23845 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23845 from kuhn/v2.5.1_fix
19083f991 | BUGFIX | !23848 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23848 from kuhn/v2.6.0_fix
8715f494f | BUGFIX | !23849 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23849 from kuhn/v2.7.1_fix
137d07f48 | BUGFIX | !23851 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23851 from kuhn/v2.8.0_fix
20824785a | BUGFIX | !23852 [Bugfix] Optimized the P2P Enable connection limit Merge pull request !23852 from kuhn/master_fix
482eb0ab9 | BUGFIX | !23715 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23715 from 王超/v2.99.0_stresshccl4
9584c1632 | BUGFIX | !23714 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23714 from 王超/v2.8.0_stresshccl4
a373cd329 | BUGFIX | !23713 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23713 from 王超/v2.7.0_stresshccl4
e213d0494 | BUGFIX | !23712 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23712 from 王超/v2.6.0_stresshccl4
ba874e36c | BUGFIX | !23711 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23711 from 王超/v2.5.0_stresshccl4
ff901d5b7 | BUGFIX | !23710 StressDetect: Fix with multiple workers; Replace "mode" with "detect_type", and the values can be "aic" or "hccs". Merge pull request !23710 from 王超/v2.1.0_stresshccl4
d85991f31 | BUGFIX | !23770 [feature_torch_2.8.0] profiler增加msprof权限校验 Merge pull request !23770 from yuliangbin/bug_fix_2.8.0_1
a82d52ced | BUGFIX | !23734 [torch_2.1.0]修复profiler内存数据解析bug Merge pull request !23734 from yuliangbin/bug_fix_2.1.0
81f53e9d5 | BUGFIX | !23735 [torch_2.5.1]修复profiler内存数据解析bug Merge pull request !23735 from yuliangbin/bug_fix_2.5.1
5eac8683a | BUGFIX | !23738 [torch_2.6.0]修复profiler内存数据解析bug Merge pull request !23738 from yuliangbin/bug_fix_2.6.0
02a3e756f | BUGFIX | !23739 [torch_2.7.1]修复profiler内存数据解析bug Merge pull request !23739 from yuliangbin/bug_fix_2.7.1
a2e91d057 | BUGFIX | !23740 [torch_2.8.0]修复profiler内存数据解析bug Merge pull request !23740 from yuliangbin/bug_fix_2.8.0
88c1a9abb | BUGFIX | !23727 [torch_master]修复profiler内存数据解析bug Merge pull request !23727 from yuliangbin/bug_fix_master
11610b99b | BUGFIX | !23684 [torch_master] 修复单CPU场景PROF文件权限校验 Merge pull request !23684 from yuliangbin/bug_fix_memory
090d1a0bf | BUGFIX | !23695 [torch_2.8.0] 修复单CPU场景PROF文件权限校验 Merge pull request !23695 from yuliangbin/bug_fix_memory_v2.8.0
8f7309944 | BUGFIX | !23694 [torch_2.7.1] 修复单CPU场景PROF文件权限校验 Merge pull request !23694 from yuliangbin/bug_fix_memory_v2.7.1
0110981e8 | BUGFIX | !23693 [torch_2.6.0] 修复单CPU场景PROF文件权限校验 Merge pull request !23693 from yuliangbin/bug_fix_memory_v2.6.0
d378e3a85 | BUGFIX | !23692 [torch_2.5.1] 修复单CPU场景PROF文件权限校验 Merge pull request !23692 from yuliangbin/bug_fix_memory_v2.5.1
dfde70a41 | BUGFIX | !23691 [torch_2.1.0] 修复单CPU场景PROF文件权限校验 Merge pull request !23691 from yuliangbin/bug_fix_memory_v2.1.0_1
adc3e0d5a | BUGFIX | !23566 get streamId from npustream Merge pull request !23566 from Gallium/bug_fix2.8.0
071f0c7f7 | BUGFIX | !23567 get stream id from npustream Merge pull request !23567 from Gallium/bugfix_2.7.1
966667047 | BUGFIX | !22886 Fix the bug to adapt the torch Generator Merge pull request !22886 from louyujing/v2.6.0-7.1.0_20250710_160035
fb1a11418 | BUGFIX | !22885 Fix the bug to adapt the torch Generator Merge pull request !22885 from louyujing/v2.5.1-7.1.0_20250710_160035
805b2636b | BUGFIX | !22906 Fix the bug to adapt the torch Generator Merge pull request !22906 from louyujing/v2.1.0-7.1.0_0710
bbef45273 | BUGFIX | !22884 Fix the bug to adapt the torch Generator Merge pull request !22884 from louyujing/v2.7.1_20250710_160035
9216e5dae | BUGFIX | !22883 Fix the bug to adapt the torch Generator Merge pull request !22883 from louyujing/v2.6.0_20250710_160035
69ada4efc | BUGFIX | !22882 Fix the bug to adapt the torch Generator Merge pull request !22882 from louyujing/v2.5.1_20250710_160035
87978c523 | BUGFIX | !22905 Fix the bug to adapt the torch Generator Merge pull request !22905 from louyujing/v2.1.0_0710
6a81c58d9 | BUGFIX | !22881 Fix the bug to adapt the torch Generator Merge pull request !22881 from louyujing/master_3
b9d77b951 | BUGFIX | !22785 【PROF】fix dynamic prof step id err Merge pull request !22785 from 梅飞要/6_7
445ab9e0c | BUGFIX | !22784 【PROF】fix dynamic prof step id err Merge pull request !22784 from 梅飞要/5_7
7dac0b622 | BUGFIX | !22783 【PROF】fix dynamic prof step id err Merge pull request !22783 from 梅飞要/1_7
49994df4c | BUGFIX | !22733 【PROF】fix dynamic prof step id err Merge pull request !22733 from 梅飞要/maaa
e79ffb427 | BUGFIX | !22725 【PROF】fix dynamic prof step id err Merge pull request !22725 from 梅飞要/2.7
061e2bea0 | BUGFIX | !22721 【PROF】fix dynamic prof step id err Merge pull request !22721 from 梅飞要/2.6
0d0002080 | BUGFIX | !22724 【PROF】fix dynamic prof step id err Merge pull request !22724 from 梅飞要/2.5
90fb9f981 | BUGFIX | !22722 【PROF】fix dynamic prof step id err Merge pull request !22722 from 梅飞要/2.1
26c4fa914 | BUGFIX | !22634 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22634 from louyujing/v2.7.1
24d7f38a3 | BUGFIX | !22639 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22639 from louyujing/v2.1.0-7.1.0
9e9af8a7f | BUGFIX | !22640 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22640 from louyujing/v2.5.1-7.1.0
de612ffee | BUGFIX | !22641 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22641 from louyujing/v2.6.0-7.1.0
e42728a75 | BUGFIX | !22638 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22638 from louyujing/v2.1.0
6167bdf40 | BUGFIX | !22635 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22635 from louyujing/v2.6.0
44717c42b | BUGFIX | !22637 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22637 from louyujing/v2.5.1
f7ccd9090 | BUGFIX | !22633 【bugfix】Adapt torch distributed _new_group_with_tag Merge pull request !22633 from louyujing/master
fb2391450 | BUGFIX | !22557 [Inductor] fix bug of force_fallback_kernel Merge pull request !22557 from 杜承昆/force_fallback_fix
d80a46978 | BUGFIX | !22601 [v2.6.0-7.1.0]fix dump_dir Merge pull request !22601 from DaiFu/v2.6.0-7.1.0
82c067ec8 | BUGFIX | !22445 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22445 from 闫鹏全/master
304693a64 | BUGFIX | !22444 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22444 from 闫鹏全/v2.7.1
d8422a715 | BUGFIX | !22443 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22443 from 闫鹏全/v2.6.0_recovery
e064d92f6 | BUGFIX | !22442 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22442 from 闫鹏全/v2.5.1
c068895d2 | BUGFIX | !22441 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22441 from 闫鹏全/v2.6.0-7.1.0
1cfebdb12 | BUGFIX | !22440 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22440 from 闫鹏全/v2.5.1-7.1.0
55c1c51d4 | BUGFIX | !22439 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22439 from 闫鹏全/v2.1.0-7.1.0
6bbef6c02 | BUGFIX | !22398 Fixed the issue of fuzzy error message in multiple N-second fast recovery scenarios Merge pull request !22398 from 闫鹏全/v2.1.0_fix
d08ef6395 | BUGFIX | !22310 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22310 from 梅飞要/2.6_7
42dbcafb4 | BUGFIX | !22311 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22311 from 梅飞要/2.5_7
f7bcce4c7 | BUGFIX | !22312 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22312 from 梅飞要/2.1_7
78d96069a | BUGFIX | !22187 【inductor】fix the way to get store_cubin Merge pull request !22187 from 杜承昆/hardcode-7.1.0
9610b96c2 | BUGFIX | !22235 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22235 from 梅飞要/mm
c6a35b898 | BUGFIX | !22234 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22234 from 梅飞要/2.7
d43e784a7 | BUGFIX | !22233 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22233 from 梅飞要/2.6
571218749 | BUGFIX | !22232 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22232 from 梅飞要/2.5
3f19c3d45 | BUGFIX | !22222 [PROF]fix dynamic prof step id; dynamic prof add step data Merge pull request !22222 from 梅飞要/2.1
62e277c68 | BUGFIX | !22256 Remove the word warning and fix the spelling of replace Merge pull request !22256 from yuhaiyan/v2.1.0-dev3
7f27f5e5f | BUGFIX | !22257 Remove the word warning and fix the spelling of replace Merge pull request !22257 from yuhaiyan/v2.5.1-dev3
69cb6ef3f | BUGFIX | !22258 Remove the word warning and fix the spelling of replace Merge pull request !22258 from yuhaiyan/v2.6.0-dev3
dcf39b65b | BUGFIX | !22259 Remove the word warning and fix the spelling of replace Merge pull request !22259 from yuhaiyan/v2.7.1-dev3
014634026 | BUGFIX | !22260 Remove the word warning and fix the spelling of replace Merge pull request !22260 from yuhaiyan/master-dev3
d88345a24 | BUGFIX | !22184 【inductor】fix the way to get store_cubin Merge pull request !22184 from 杜承昆/hardcode
0c7ba1f7d | BUGFIX | !22090 bugfix: DestroyUsedStreams use stream.stream(false) Merge pull request !22090 from 王超/v2.8.0_checkfix3
120619c6a | BUGFIX | !22089 bugfix: DestroyUsedStreams use stream.stream(false) Merge pull request !22089 from 王超/v2.7.0_checkfix3
eef2a97f8 | BUGFIX | !22088 bugfix: DestroyUsedStreams use stream.stream(false) Merge pull request !22088 from 王超/v2.6.0_checkfix3
ec9b3740c | BUGFIX | !22087 bugfix: DestroyUsedStreams use stream.stream(false) Merge pull request !22087 from 王超/v2.5.0_checkfix3
49da77dae | BUGFIX | !22086 bugfix: DestroyUsedStreams use stream.stream(false) Merge pull request !22086 from 王超/v2.1.0_memstream
daa9ef1e5 | BUGFIX | !22011 [PTA 2.1.0] fix typo for flight_recorder Merge pull request !22011 from yinqian/v2.1.0
d5215ddd8 | BUGFIX | !22009 [PTA 2.6.0] fix typo for flight_recorder Merge pull request !22009 from yinqian/v2.6.0
f7e46a0cd | BUGFIX | !22007 [master] fix typo for flight_recorder Merge pull request !22007 from yinqian/master
416ce2569 | BUGFIX | !22010 [PTA 2.5.1] fix typo for flight_recorder Merge pull request !22010 from yinqian/v2.5.1
be2158f51 | BUGFIX | !21833 silent checkv3 fix for recalculation Merge pull request !21833 from 王超/v2.8.0_silentfix4
e51a48fb1 | BUGFIX | !21831 silent checkv3 fix for recalculation Merge pull request !21831 from 王超/v2.6.0_silentfix4
03c16ad02 | BUGFIX | !21832 silent checkv3 fix for recalculation Merge pull request !21832 from 王超/v2.7.0_silentfix4
83718846e | BUGFIX | !21830 silent checkv3 fix for recalculation Merge pull request !21830 from 王超/v2.5.0_silentfix4
1999adc48 | BUGFIX | !21808 silent checkv3 fix for recalculation Merge pull request !21808 from 王超/v2.1.0_silentfix4
4d1bc797f | BUGFIX | !21732 【Inductor】Fix codegen bug in rebuild_flattened dim Merge pull request !21732 from 杜承昆/bugfix1
3cc3eace5 | BUGFIX | !21720 silent check v3 fix for tcpstore thread Merge pull request !21720 from 王超/v2.8.0_silentfix2
b73c1d922 | BUGFIX | !21719 silent check v3 fix for tcpstore thread Merge pull request !21719 from 王超/v2.7.0_silentfix2
c2d0769df | BUGFIX | !21718 silent check v3 fix for tcpstore thread Merge pull request !21718 from 王超/v2.6.0_silentfix2
812b86bcf | BUGFIX | !21717 silent check v3 fix for tcpstore thread Merge pull request !21717 from 王超/v2.5.0_silentfix2
a59bf0653 | BUGFIX | !21697 silent check v3 fix for tcpstore thread Merge pull request !21697 from 王超/v2.1.0_silentfix2
51dfb9339 | BUGFIX | !21270 codegen fix for index expr Merge pull request !21270 from wl1259/codegen_fix_index_expr
62d92b595 | BUGFIX | !21475 bugfix for filename Merge pull request !21475 from 郭光浩/v2.5.1
19f58182e | BUGFIX | !21476 bugfix for filename Merge pull request !21476 from 郭光浩/v2.6.0
fbf4b63f4 | BUGFIX | !21477 bugfix for filename Merge pull request !21477 from 郭光浩/v2.7.1
667683734 | BUGFIX | !21473 bugfix for filename Merge pull request !21473 from 郭光浩/master
8e4f00360 | BUGFIX | !21474 bugfix for filename Merge pull request !21474 from 郭光浩/v2.1.0
442016cf1 | BUGFIX | !20970 [PROF]fix mstx err Merge pull request !20970 from 梅飞要/2.6
1ca68f961 | BUGFIX | !20969 [PROF]fix mstx err Merge pull request !20969 from 梅飞要/2.5
97c317d27 | BUGFIX | !20968 [PROF]fix mstx err Merge pull request !20968 from 梅飞要/2.4
39ce02bd1 | BUGFIX | !20967 [PROF]fix mstx err Merge pull request !20967 from 梅飞要/2.3
d5523d1ec | BUGFIX | !20971 [PROF]fix mstx err Merge pull request !20971 from 梅飞要/ma
3649f1508 | BUGFIX | !20905 [PROF]fix mstx err Merge pull request !20905 from 梅飞要/2.1
830222c89 | BUGFIX | !20430 Fixed the failed testcases Merge pull request !20430 from yuhaiyan/v2.6.0-dev1
cef697fe5 | BUGFIX | !20432 Fixed the failed testcases Merge pull request !20432 from yuhaiyan/master-dev1
f5d701b90 | BUGFIX | !20429 Fixed the failed testcases Merge pull request !20429 from yuhaiyan/v2.5.1-dev1
cb1b0cd39 | BUGFIX | !20407 Fixed the failed testcases Merge pull request !20407 from yuhaiyan/v2.1.0-dev1
5e8f91824 | BUGFIX | !20427 Fixed the failed testcases Merge pull request !20427 from yuhaiyan/v2.4.0-dev1
dfa212415 | BUGFIX | !20412 Fixed the failed testcases Merge pull request !20412 from yuhaiyan/v2.3.1-dev1
e89c7eae0 | BUGFIX | !20235 [DTensor] fix DTensor D2H copy by replacing aten::to.dtype_layout with aten::_to_copy Merge pull request !20235 from zhangqiongwen/v2.5.1_copy_fix
01970494c | BUGFIX | !20242 [DTensor] fix DTensor D2H copy by replacing aten::to.dtype_layout with aten::_to_copy Merge pull request !20242 from zhangqiongwen/v2.4.0_copy_fix
fc05e05bd | BUGFIX | !20241 [DTensor] fix DTensor D2H copy by replacing aten::to.dtype_layout with aten::_to_copy Merge pull request !20241 from zhangqiongwen/v2.3.1_copy_fix
317b7a92c | BUGFIX | !20240 [DTensor] fix DTensor D2H copy by replacing aten::to.dtype_layout with aten::_to_copy Merge pull request !20240 from zhangqiongwen/v2.1.0_copy_fix
1893913bd | BUGFIX | !20320 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20320 from dilililiwhy/build_bugfix_240700
02f33326d | BUGFIX | !20321 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20321 from dilililiwhy/build_bugfix_251700
3e5bbf6a9 | BUGFIX | !20319 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20319 from dilililiwhy/build_bugfix_231700
d59941eb5 | BUGFIX | !20318 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20318 from dilililiwhy/build_bugfix_210700
361aa468f | BUGFIX | !20267 [bugfix]update torch_npu_schema.json Merge pull request !20267 from 陈威亨/v2.1.0-7.0.0
ab692a84f | BUGFIX | !20310 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20310 from dilililiwhy/build_bugfix_240
31674538b | BUGFIX | !20312 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20312 from dilililiwhy/build_bugfix_210
898d7b338 | BUGFIX | !20311 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20311 from dilililiwhy/build_bugfix_231
c9a933b64 | BUGFIX | !20308 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20308 from dilililiwhy/build_bugfix_260
96c6443ae | BUGFIX | !20309 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20309 from dilililiwhy/build_bugfix_251
9cafd5af3 | BUGFIX | !20306 Remove redundant DISABLE_RPC_FRAMEWORK=FALSE Merge pull request !20306 from dilililiwhy/build_bugfix
c4fd1d9df | BUGFIX | !19925 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19925 from 杜金航/v2.4.0-clean
f87f72ed2 | BUGFIX | !19923 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19923 from 杜金航/v2.1_cleancode
1a9c68f85 | BUGFIX | !19924 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19924 from 杜金航/v2.3.1-clean
d56522b52 | BUGFIX | !19926 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19926 from 杜金航/v2.5.1-clean
bb4885114 | BUGFIX | !19904 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19904 from 杜金航/master
a09ce7fd1 | BUGFIX | !19903 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19903 from 杜金航/v2.6.0
c04caff71 | BUGFIX | !19902 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19902 from 杜金航/v2.5.1
a6b2b2f47 | BUGFIX | !19901 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19901 from 杜金航/v2.4.0
f92a37f3e | BUGFIX | !19900 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19900 from 杜金航/v2.3.1
c001fd854 | BUGFIX | !19876 Fixed interlocks caused by recursive_mutex and eventfd_read Merge pull request !19876 from 杜金航/v2.1.0
25dc8713a | BUGFIX | !19759 [bugfix]update torch_npu_schema.json Merge pull request !19759 from 陈威亨/master
ddf7e7b4c | BUGFIX | !19719 fix for cann compatibility Merge pull request !19719 from 王强/master
6c8350401 | BUGFIX | !19834 【flight recorder】fix dict Merge pull request !19834 from yinqian/v2.1.0
9568dfbd4 | BUGFIX | !19877 Fix the test_dist_get_hccl_stream_id_same Merge pull request !19877 from yuhaiyan/v2.1.0-dev2
e2bd6cf55 | BUGFIX | !19837 Delete the @unittest.skip and fix the torch.load failure Merge pull request !19837 from yuhaiyan/v2.6.0-dev2
959f6e1f8 | BUGFIX | !19838 Fix the torch.load failure. Merge pull request !19838 from yuhaiyan/master-dev2
93cb90ddf | BUGFIX | !19752 fix for cann compatibility Merge pull request !19752 from 王强/v2.4.0-7.0.0
fb8ae90f5 | BUGFIX | !19753 fix for cann compatibility Merge pull request !19753 from 王强/v2.5.1-7.0.0
613f70023 | BUGFIX | !19751 fix for cann compatibility Merge pull request !19751 from 王强/v2.3.1-7.0.0
382341fb5 | BUGFIX | !19750 fix for cann compatibility Merge pull request !19750 from 王强/v2.1.0-7.0.0
455ba3e39 | BUGFIX | !19709 fix for cann compatibility Merge pull request !19709 from 王强/v2.1.0
f37d0b8e4 | BUGFIX | !19715 fix for cann compatibility Merge pull request !19715 from 王强/v2.3.1
5fed87c21 | BUGFIX | !19716 fix for cann compatibility Merge pull request !19716 from 王强/v2.4.0
b822f9c81 | BUGFIX | !19717 fix for cann compatibility Merge pull request !19717 from 王强/v2.5.1
833646886 | BUGFIX | !19718 fix for cann compatibility Merge pull request !19718 from 王强/v2.6.0
b35e7baa3 | BUGFIX | !19763 [bugfix]update torch_npu_schema.json Merge pull request !19763 from 陈威亨/v2.6.0
ef0148b21 | BUGFIX | !19762 [bugfix]update torch_npu_schema.json Merge pull request !19762 from 陈威亨/v2.5.1
b26e5db3b | BUGFIX | !19761 [bugfix]update torch_npu_schema.json Merge pull request !19761 from 陈威亨/v2.4.0
c9de4dc68 | BUGFIX | !19760 [bugfix]update torch_npu_schema.json Merge pull request !19760 from 陈威亨/v2.3.1
3d0a6b484 | BUGFIX | !19710 Revert 'Pull Request !19664 : [bugfix]update torch_npu_schema.json' Merge pull request !19710 from dilililiwhy/revert-merge-19664-v2.6.0
ad7575d9e | BUGFIX | !19702 【Profiler】warmup bugfix Merge pull request !19702 from hhz886/v2.5.1-7.0.0
6b9828a35 | BUGFIX | !19701 【Profiler】warmup bugfix Merge pull request !19701 from hhz886/v2.4.0-7.0.0
d7b33294c | BUGFIX | !19699 【Profiler】warmup bugfix Merge pull request !19699 from hhz886/v2.3.1-7.0.0
d85428b5d | BUGFIX | !19698 【Profiler】warmup bugfix Merge pull request !19698 from hhz886/v2.1.0-7.0.0
5b24022fe | BUGFIX | !19706 Revert 'Pull Request !19660 : [bugfix]update torch_npu_schema.json' Merge pull request !19706 from shaoyf/revert-merge-19660-master
d7237e8e8 | BUGFIX | !19705 Revert 'Pull Request !19661 : [bugfix]update torch_npu_schema.json' Merge pull request !19705 from shaoyf/revert-merge-19661-v2.3.1
35ba2ffe3 | BUGFIX | !19704 Revert 'Pull Request !19662 : [bugfix]update torch_npu_schema.json' Merge pull request !19704 from shaoyf/revert-merge-19662-v2.4.0
419ee1040 | BUGFIX | !19703 回退 'Pull Request !19663 : [bugfix]update torch_npu_schema.json' Merge pull request !19703 from shaoyf/revert-merge-19663-v2.5.1
11019beee | BUGFIX | !19664 [bugfix]update torch_npu_schema.json Merge pull request !19664 from 陈威亨/v2.6.0
5bb7794e9 | BUGFIX | !19663 [bugfix]update torch_npu_schema.json Merge pull request !19663 from 陈威亨/v2.5.1
8ce8d4653 | BUGFIX | !19662 [bugfix]update torch_npu_schema.json Merge pull request !19662 from 陈威亨/v2.4.0
52ccc2d14 | BUGFIX | !19661 [bugfix]update torch_npu_schema.json Merge pull request !19661 from 陈威亨/v2.3.1
bfe450c61 | BUGFIX | !19660 [bugfix]update torch_npu_schema.json Merge pull request !19660 from 陈威亨/master
c4e78bb25 | BUGFIX | !19629 [Profiler]warmup bugfix Merge pull request !19629 from hhz886/v2.1.0
9e8a3b94e | BUGFIX | !19628 [Profiler]warmup bugfix Merge pull request !19628 from hhz886/v2.3.1
899884fdd | BUGFIX | !19627 [Profiler]warmup bugfix Merge pull request !19627 from hhz886/v2.4.0
5b6a1f4d4 | BUGFIX | !19626 [Profiler]warmup bugfix Merge pull request !19626 from hhz886/v2.5.1
d7862c5d2 | BUGFIX | !19625 [Profiler]warmup bugfix Merge pull request !19625 from hhz886/v2.6.0
ff314ef75 | BUGFIX | !19624 [Profiler]warmup bugfix Merge pull request !19624 from hhz886/master
bf06bdb54 | BUGFIX | !19638 Fix for missing parameter issue Merge pull request !19638 from 叶子凡/master_bugfix
eba5a322d | BUGFIX | !19582 Fix for missing parameter issue Merge pull request !19582 from 叶子凡/v2.5.1-7.0.0_bugfix
9c28e5124 | BUGFIX | !19580 Fix for missing parameter issue Merge pull request !19580 from 叶子凡/v2.5.1_bugfix
ac1fdede2 | BUGFIX | !19579 Fix for missing parameter issue Merge pull request !19579 from 叶子凡/v2.6.0_bugfix
79a0c17d7 | BUGFIX | !19623 [bugfix]update torch_npu_schema.json Merge pull request !19623 from 陈威亨/v2.1.0
3fd0391ce | BUGFIX | !19425 bugfix communication function error Merge pull request !19425 from wangzixuan/v2.5.1-7.0.0
89be513e1 | BUGFIX | !19426 bugfix communication function error Merge pull request !19426 from wangzixuan/v2.4.0-7.0.0
ababff55d | BUGFIX | !19427 bugfix communication function error Merge pull request !19427 from wangzixuan/v2.3.1-7.0.0
78df3165e | BUGFIX | !19428 bugfix communication function error Merge pull request !19428 from wangzixuan/v2.1.0-7.0.0
2c2f797a9 | BUGFIX | !19373 [bugfix] [profiler] communication function error Merge pull request !19373 from wangzixuan/master
0a9b1d168 | BUGFIX | !19374 [bugfix] [profiler] communication function error Merge pull request !19374 from wangzixuan/dev-2.6.0
e0da93ef0 | BUGFIX | !19375 [bugfix] [profiler] communication function error Merge pull request !19375 from wangzixuan/dev-2.5.1
0d87a8925 | BUGFIX | !19376 [bugfix] [profiler] communication function error Merge pull request !19376 from wangzixuan/dev-2.4
5eac6f4d7 | BUGFIX | !19377 [bugfix] [profiler] communication function error Merge pull request !19377 from wangzixuan/dev-2.3.1
ee2381ee8 | BUGFIX | !19378 [bugfix] [profiler] communication function error Merge pull request !19378 from wangzixuan/dev-2.1.0
2646225a8 | BUGFIX | !19290 Fixed failed testcases. Merge pull request !19290 from yuhaiyan/v2.1.0-dev2
e17a59ab3 | BUGFIX | !19253 Fixed failed testcases. Merge pull request !19253 from yuhaiyan/v2.3.1-dev1
8851cad5e | BUGFIX | !19254 Fixed failed testcases. Merge pull request !19254 from yuhaiyan/v2.4.0-dev1
5a6e13a2f | BUGFIX | !19255 Fixed failed testcases. Merge pull request !19255 from yuhaiyan/v2.5.1-dev1
cfcacb752 | BUGFIX | !19256 Fixed failed testcases. Merge pull request !19256 from yuhaiyan/v2.6.0-dev1
bdfb33cfa | BUGFIX | !19262 Fixed failed testcases. Merge pull request !19262 from yuhaiyan/master-dev1
6c142a8d1 | BUGFIX | !19185 memory bugfix Merge pull request !19185 from hhz886/v2.6.0
cdff2439b | BUGFIX | !19184 memory bugfix Merge pull request !19184 from hhz886/v2.5.1
cf5c541bd | BUGFIX | !19183 memory bugfix Merge pull request !19183 from hhz886/v2.4.0
4af2a74b4 | BUGFIX | !19182 memory bugfix Merge pull request !19182 from hhz886/v2.3.1
7773cfafb | BUGFIX | !19181 memory bugfix Merge pull request !19181 from hhz886/v2.1.0
ea69e998f | BUGFIX | !19180 memory bugfix Merge pull request !19180 from hhz886/master
7292c6894 | BUGFIX | !19160 Fixed some failed testcases Merge pull request !19160 from yuhaiyan/v2.4.0-dev1
9bc1fffb0 | BUGFIX | !19158 Fixed some failed testcases Merge pull request !19158 from yuhaiyan/v2.3.1-dev1
c886d9614 | BUGFIX | !19167 Fixed some failed testcases Merge pull request !19167 from yuhaiyan/v2.5.1-dev1
0d4a03c07 | BUGFIX | !19168 Fixed some failed testcases Merge pull request !19168 from yuhaiyan/v2.6.0-dev1
ecc73b3f9 | BUGFIX | !19170 Fixed some failed testcases Merge pull request !19170 from yuhaiyan/master-dev1
ac8f4abfe | BUGFIX | !19077 lazy init dynolog Merge pull request !19077 from Gallium/bug_fix_master
1452b864b | BUGFIX | !19087 lazy init dynolog Merge pull request !19087 from Gallium/bug_fix_2.6.0
0c9b297a2 | BUGFIX | !19062 lazy init dynolog Merge pull request !19062 from Gallium/bug_fix_2.5.1
f3550c1b5 | BUGFIX | !19075 lazy init dynolog Merge pull request !19075 from Gallium/bug_fix_2.4.0
03af99856 | BUGFIX | !19079 lazy init dynolog Merge pull request !19079 from Gallium/bug_fix_v2.3.1
1a552cd62 | BUGFIX | !19078 lazy init dynolog Merge pull request !19078 from Gallium/bug_fix_2.1.0
a66e54f84 | BUGFIX | !18909 stop dynolog when exit Merge pull request !18909 from Gallium/bug_fix_2.1.0
202c68bd0 | BUGFIX | !18911 stop dynolog when exit Merge pull request !18911 from Gallium/bug_fix_v2.3.1
b609b7405 | BUGFIX | !18912 stop dynolog when exit Merge pull request !18912 from Gallium/bug_fix_2.4.0
c71555b2d | BUGFIX | !18913 stop dynolog when process exit Merge pull request !18913 from Gallium/bug_fix_2.5.1
e7b72a9a6 | BUGFIX | !18914 stop dynolog when exit Merge pull request !18914 from Gallium/bug_fix_2.6.0
957e6695d | BUGFIX | !18915 stop dynolog when exit Merge pull request !18915 from Gallium/bug_fix_master
0dfd68f62 | BUGFIX | !18953 [PROF] Profiler bugfix for CannPackageManager Merge pull request !18953 from wangjie/profiler_fix_cann_manager_210
f47e8c1e5 | BUGFIX | !18955 [PORF] Profiler bugfix for CannPackageManager Merge pull request !18955 from wangjie/profiler_fix_cann_manager_240
96ea4dd4e | BUGFIX | !18954 [PORF] Profiler bugfix for CannPackageManager Merge pull request !18954 from wangjie/profiler_fix_cann_manager_231
3a359b8f2 | BUGFIX | !18958 [PROF] Profiler bugfix for CannPackageManager Merge pull request !18958 from wangjie/profiler_fix_cann_manager_master
456aa19ff | BUGFIX | !18956 [PORF] Profiler bugfix for CannPackageManager Merge pull request !18956 from wangjie/profiler_fix_cann_manager_251
ed920b1da | BUGFIX | !18957 [PORF] Profiler bugfix for CannPackageManager Merge pull request !18957 from wangjie/profiler_fix_cann_manager_260
e1e392b68 | BUGFIX | !18279 [v2.5.1][Docs] Fix minial version desp for autoloading Merge pull request !18279 from Yuanhao Ji/fix/docs/251
e870bd466 | BUGFIX | !18278 [v2.6.0][Docs] Fix minial version desp for autoloading Merge pull request !18278 from Yuanhao Ji/fix/docs/260
078cda284 | BUGFIX | !18277 [Master][Docs] Fix minial version desp for autoloading Merge pull request !18277 from Yuanhao Ji/fix/docs/master
44d2aad41 | BUGFIX | !16668 Snapeshot data dump supports to deal with GE error Merge pull request !16668 from 杜金航/master
1c2110299 | BUGFIX | !18381 Snapeshot data dump supports to deal with GE error Merge pull request !18381 from 杜金航/v2.6.0
055c5be00 | BUGFIX | !16672 Snapeshot data dump supports to deal with GE error Merge pull request !16672 from 杜金航/v2.5.1
a47377f7f | BUGFIX | !16671 Snapeshot data dump supports to deal with GE error Merge pull request !16671 from 杜金航/v2.4.0
1cea1c910 | BUGFIX | !16670 Snapeshot data dump supports to deal with GE error Merge pull request !16670 from 杜金航/v2.3.1
d7ef5286c | BUGFIX | !18380 Snapeshot data dump supports to deal with GE error Merge pull request !18380 from 杜金航/v2.1.0
715ccfd44 | BUGFIX | !18751 Fixed a value in the slow_test_blocklist. Merge pull request !18751 from yuhaiyan/v2.1.0-dev1
4744d0fbb | BUGFIX | !18752 Fixed a value in the slow_test_blocklist. Merge pull request !18752 from yuhaiyan/v2.3.1-dev1
78dfb70cb | BUGFIX | !18754 Fixed a value in the slow_test_blocklist. Merge pull request !18754 from yuhaiyan/v2.4.0-dev1
111aad72b | BUGFIX | !18755 Fixed a value in the slow_test_blocklist. Merge pull request !18755 from yuhaiyan/v2.5.1-dev1
ab85b4f9c | BUGFIX | !18756 Fixed a value in the slow_test_blocklist. Merge pull request !18756 from yuhaiyan/v2.6.0-dev1
4c9d9ff4b | BUGFIX | !18757 Fixed a value in the slow_test_blocklist. Merge pull request !18757 from yuhaiyan/master-dev1
72e58143e | BUGFIX | !17358 Fixed Failure to save data on oom devices except device 0 Merge pull request !17358 from 杜金航/v2.1_cleancode
5621a5b76 | BUGFIX | !18746 Fixed Failure to save data on oom devices except device 0 Merge pull request !18746 from 杜金航/v2.3.1-clean
c52c1b6d6 | BUGFIX | !18747 Fixed Failure to save data on oom devices except device 0 Merge pull request !18747 from 杜金航/v2.4.0-clean
a0542467a | BUGFIX | !18748 Fixed Failure to save data on oom devices except device 0 Merge pull request !18748 from 杜金航/v2.5.1-clean
6e7b49be1 | BUGFIX | !18749 Fixed Failure to save data on oom devices except device 0 Merge pull request !18749 from 杜金航/v2.6.0-clean
31ee13bd5 | BUGFIX | !18750 Fixed Failure to save data on oom devices except device 0 Merge pull request !18750 from 杜金航/master_cleancode
6c885279d | BUGFIX | !18341 Fixed the failed tests. Merge pull request !18341 from yuhaiyan/v2.3.1-dev1
a4f98979a | BUGFIX | !18342 Fixed the failed tests. Merge pull request !18342 from yuhaiyan/v2.4.0-dev1
424c68a00 | BUGFIX | !18343 Fixed the failed tests. Merge pull request !18343 from yuhaiyan/v2.5.1-dev1
9cf9857ad | BUGFIX | !18344 Fixed the failed tests. Merge pull request !18344 from yuhaiyan/v2.6.0-dev1
ebaff0054 | BUGFIX | !18345 Fixed the failed tests. Merge pull request !18345 from yuhaiyan/master-dev1
060bf8e9c | BUGFIX | !18340 Fixed the failed tests. Merge pull request !18340 from yuhaiyan/v2.1.0-dev1
6b44e6282 | BUGFIX | !18257 Fix the judgment condition of device_check Merge pull request !18257 from wgb/master
a330367dd | BUGFIX | !18258 Fix the judgment condition of device_check Merge pull request !18258 from wgb/v2.6.0
c38e2b85a | BUGFIX | !18259 Fix the judgment condition of device_check Merge pull request !18259 from wgb/v2.5.1
d4841401e | BUGFIX | !18260 Fix the judgment condition of device_check Merge pull request !18260 from wgb/v2.4.0
8f78f4e17 | BUGFIX | !18261 Fix the judgment condition of device_check Merge pull request !18261 from wgb/v2.3.1
b6eed5b40 | BUGFIX | !18262 Fix the judgment condition of device_check Merge pull request !18262 from wgb/v2.1.0
9cf59347b | BUGFIX | !17760 Fixed the failed tests. Merge pull request !17760 from yuhaiyan/v2.1.0-dev2
8ab42acd9 | BUGFIX | !17763 Fixed the failed tests. Merge pull request !17763 from yuhaiyan/v2.3.1-dev2
1c90b2860 | BUGFIX | !17761 Fixed the failed tests. Merge pull request !17761 from yuhaiyan/v2.4.0-dev2
50641c1d1 | BUGFIX | !17762 Fixed the failed tests. Merge pull request !17762 from yuhaiyan/v2.5.1-dev2
ba493ea1c | BUGFIX | !17981 Fixed the failed tests. Merge pull request !17981 from yuhaiyan/v2.6.0-dev2
cbfac3079 | BUGFIX | !17764 Fixed the failed tests. Merge pull request !17764 from yuhaiyan/master-dev2
82c91af45 | BUGFIX | !17817 [fix] fix third lib depend on header file Merge pull request !17817 from xudaohong/v2.3.1
fd424421d | BUGFIX | !17814 [fix] fix third lib depend on header file Merge pull request !17814 from xudaohong/master
cf5fee609 | BUGFIX | !17815 [fix] fix third lib depend on header file Merge pull request !17815 from xudaohong/v2.5.1
bda57f140 | BUGFIX | !17816 [fix] fix third lib depend on header file Merge pull request !17816 from xudaohong/v2.4.0
f361e1cfd | BUGFIX | !17806 [fix] fix third lib depend on header file Merge pull request !17806 from xudaohong/hccl
8fd0ed8d4 | BUGFIX | !17654 [Profiler] fix clean code on branch master Merge pull request !17654 from Mrtutu/fix_signed_clean_code_master
956a7c9e3 | BUGFIX | !17657 [Profiler] fix clean code on branch v2.5.1 Merge pull request !17657 from Mrtutu/fix_signed_clean_code_v2.5.1
ed1b26851 | BUGFIX | !17656 [Profiler] fix clean code on branch v2.4.0 Merge pull request !17656 from Mrtutu/fix_signed_clean_code_v2.4.0
7a840f897 | BUGFIX | !17655 [Profiler] fix clean code on branch v2.3.1 Merge pull request !17655 from Mrtutu/fix_signed_clean_code_v2.3.1
fa9bda590 | BUGFIX | !17653 [Profiler] fix clean code on branch v2.1.0 Merge pull request !17653 from Mrtutu/fix_signed_clean_code_v2.1.0
225f38670 | BUGFIX | !17342 Fixed some failed tests. Merge pull request !17342 from yuhaiyan/master-dev1
fba7939c4 | BUGFIX | !17340 Fixed some failed tests. Merge pull request !17340 from yuhaiyan/v2.5.1-dev1
3553dba6b | BUGFIX | !17334 Fixed some failed tests. Merge pull request !17334 from yuhaiyan/v2.4.0-dev1
9b22bc023 | BUGFIX | !17333 Fixed some failed tests. Merge pull request !17333 from yuhaiyan/v2.3.1-dev1
459abbed0 | BUGFIX | !17314 Fixed some failed tests. Merge pull request !17314 from yuhaiyan/v2.1.0-dev1
ba645b3ff | BUGFIX | !17147 Fix the out of place custom functions in variabletype Merge pull request !17147 from wgb/bugfix_2.4
9ad992184 | BUGFIX | !17149 Fix the out of place custom functions in variabletype Merge pull request !17149 from wgb/bugfix_master
997222363 | BUGFIX | !17148 Fix the out of place custom functions in variabletype Merge pull request !17148 from wgb/bugfix_2.5.1
83d0826b7 | BUGFIX | !17146 Fix the out of place custom functions in variabletype Merge pull request !17146 from wgb/bugfix_2.3.0
f78274bc0 | BUGFIX | !17145 Fix the out of place custom functions in variabletype Merge pull request !17145 from wgb/bugfix_2.1
ebfa1de36 | BUGFIX | !17310 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17310 from shaojieMike/v2.4.0-6.0.0_PR
f9397a415 | BUGFIX | !17309 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17309 from shaojieMike/v2.3.1-6.0.0_PR
b561b654f | BUGFIX | !17308 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17308 from shaojieMike/v2.1.0-6.0.0_PR
b269d90cd | BUGFIX | !17174 【dyno】【PROF】Master: codecheck for dynolog Merge pull request !17174 from liyou_b/dynolog_bug_fixed_master
1a665439c | BUGFIX | !17170 【dyno】【PROF】V2.1.0: codecheck for dynolog Merge pull request !17170 from liyou_b/dynolog_bug_fixed_210
c35e2a524 | BUGFIX | !17171 【dyno】【PROF】V2.3.1: codecheck for dynolog Merge pull request !17171 from liyou_b/dynolog_bug_fixed_231
1a95740ee | BUGFIX | !17172 【dyno】【PROF】V2.4.0: codecheck for dynolog Merge pull request !17172 from liyou_b/dynolog_bug_fixed_240
8ea0d01fd | BUGFIX | !17173 【dyno】【PROF】V2.5.1: codecheck for dynolog Merge pull request !17173 from liyou_b/dynolog_bug_fixed_251
0aefa68f7 | BUGFIX | !17265 Fixed the test_fault_mode.py Merge pull request !17265 from yuhaiyan/v2.1.0-dev1
6e08e7a5f | BUGFIX | !17266 Fixed the test_fault_mode.py Merge pull request !17266 from yuhaiyan/v2.3.1-dev1
4ecc73f00 | BUGFIX | !17269 Fixed the test_fault_mode.py Merge pull request !17269 from yuhaiyan/v2.4.0-dev1
9156b01f6 | BUGFIX | !17267 Fixed the test_fault_mode.py Merge pull request !17267 from yuhaiyan/v2.5.1-dev1
478041129 | BUGFIX | !17268 Fixed the test_fault_mode.py Merge pull request !17268 from yuhaiyan/master-dev1
c66a1fafe | BUGFIX | !17231 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17231 from shaojieMike/master_PR
d26a42a10 | BUGFIX | !17229 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17229 from shaojieMike/v2.1.0_PR2One
b00a97565 | BUGFIX | !17232 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17232 from shaojieMike/v2.3.1_PR
b0daf6f81 | BUGFIX | !17233 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17233 from shaojieMike/v2.4.0_PR
d2b5db86f | BUGFIX | !17230 [bugfix] Refine Fine-Grained Core Binding Logic Merge pull request !17230 from shaojieMike/v2.5.1_PR
88dab78e8 | BUGFIX | !17180 [Bugfix] Copy operator misses memory_format. Merge pull request !17180 from 王夏夏/v2.4.0-6.0.0
62e67c351 | BUGFIX | !17178 [Bugfix] Copy operator misses memory_format. Merge pull request !17178 from 王夏夏/v2.3.1-6.0.0
b5caf8a68 | BUGFIX | !16765 [Bugfix] Copy operator misses memory_format. Merge pull request !16765 from 王夏夏/v2.1.0
7dcade8b8 | BUGFIX | !16767 [Bugfix] Copy operator misses memory_format. Merge pull request !16767 from 王夏夏/v2.3.1
639f65b8c | BUGFIX | !16768 [Bugfix] Copy operator misses memory_format. Merge pull request !16768 from 王夏夏/v2.4.0
40f1862d6 | BUGFIX | !16769 [Bugfix] Copy operator misses memory_format. Merge pull request !16769 from 王夏夏/v2.5.1
97b80840e | BUGFIX | !16764 [Bugfix] Copy operator misses memory_format. Merge pull request !16764 from 王夏夏/master
e0288cfe9 | BUGFIX | !17055 Fixed the requirements.txt Merge pull request !17055 from yuhaiyan/v2.1.0-dev2
15b544002 | BUGFIX | !17056 Fixed the requirements.txt Merge pull request !17056 from yuhaiyan/v2.3.1-dev2
db9c9c41b | BUGFIX | !17057 Fixed the requirements.txt Merge pull request !17057 from yuhaiyan/v2.4.0-dev2
078b7e4db | BUGFIX | !17058 Fixed the requirements.txt Merge pull request !17058 from yuhaiyan/v2.5.1-dev2
68dbd155b | BUGFIX | !17059 Fixed the requirements.txt Merge pull request !17059 from yuhaiyan/master-dev2
ec02dd402 | BUGFIX | !17029 bugfix for ddp ut Merge pull request !17029 from 邵非凡/dptest21
50653e103 | BUGFIX | !17045 bugfix for ddp ut Merge pull request !17045 from 邵非凡/dptest23
576845946 | BUGFIX | !17046 bugfix for ddp ut Merge pull request !17046 from 邵非凡/dptest24
acd0eaeca | BUGFIX | !17047 bugfix for ddp ut Merge pull request !17047 from 邵非凡/dptest25
8de4d5c0e | BUGFIX | !17048 bugfix for ddp ut Merge pull request !17048 from 邵非凡/dptest
5f144ceb3 | BUGFIX | !16970 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16970 from fanglanyue/bugfix_fwd_v2.3.1-6.0.0
1a2d6e563 | BUGFIX | !16965 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16965 from fanglanyue/bugfix_fwd_v2.4.0-6.0.0
0c9e83ddd | BUGFIX | !16964 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16964 from fanglanyue/bugfix_fwd_v2.1.0-6.0.0
43a44d4e6 | BUGFIX | !16826 [PROF]Fix: use GB instead of GiB Merge pull request !16826 from chenjunjie/240_600_cjj
82d492dc9 | BUGFIX | !16825 [PROF]Fix: use GB instead of GiB Merge pull request !16825 from chenjunjie/231_600_cjj
bc5c69259 | BUGFIX | !16824 [PROF]Fix: use GB instead of GiB Merge pull request !16824 from chenjunjie/210_600_cjj
a098b2cdd | BUGFIX | !16969 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16969 from fanglanyue/bugfix_fwd_v2.1.0
f20eebc91 | BUGFIX | !16968 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16968 from fanglanyue/bugfix_fwd_v2.3.1
64443a306 | BUGFIX | !16967 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16967 from fanglanyue/bugfix_fwd_v2.4.0
67a155419 | BUGFIX | !16966 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16966 from fanglanyue/bugfix_fwd_v2.5.1
708122295 | BUGFIX | !16963 【BUG】【PROF】Master：filter fwdbwd flow Merge pull request !16963 from fanglanyue/bugfix_fwd_master
046e462e9 | BUGFIX | !16823 [PROF]Fix: use GB instead of GiB Merge pull request !16823 from chenjunjie/251_cjj
44870965b | BUGFIX | !16822 [PROF]Fix: use GB instead of GiB Merge pull request !16822 from chenjunjie/240_cjj
cb01cbebb | BUGFIX | !16820 [PROF]Fix: use GB instead of GiB Merge pull request !16820 from chenjunjie/210_cjj
0c22ed405 | BUGFIX | !16821 [PROF]Fix: use GB instead of GiB Merge pull request !16821 from chenjunjie/231_cjj
be87dc019 | BUGFIX | !16819 [PROF]Fix: use GB instead of GiB Merge pull request !16819 from chenjunjie/master_cjj
f8bb0ed1c | BUGFIX | !16894 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16894 from yinglinwei/v2.1.0-6.0.0
719c6d806 | BUGFIX | !16892 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16892 from yinglinwei/v2.4.0-6.0.0
c60916f0a | BUGFIX | !16893 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16893 from yinglinwei/v2.3.1-6.0.0
a902deea6 | BUGFIX | !16163 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16163 from yinglinwei/v2.5.1
f06e40e07 | BUGFIX | !16162 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16162 from yinglinwei/v2.4.0
90c676dff | BUGFIX | !16160 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16160 from yinglinwei/v2.1.0
4612b9bf8 | BUGFIX | !16161 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16161 from yinglinwei/v2.3.1
d297401e6 | BUGFIX | !16152 bugfix: Fix conflict between torch.Tensor.to and lazymodule in transfer_to_npu Merge pull request !16152 from yinglinwei/master
b109da06f | BUGFIX | !16849 Fixed test_bidirectional_lstm.py Merge pull request !16849 from yuhaiyan/v2.1.0-dev1
a12719068 | BUGFIX | !16852 Fixed test_bidirectional_lstm.py Merge pull request !16852 from yuhaiyan/v2.4.0-dev1
e2a9106cf | BUGFIX | !16850 Fixed test_bidirectional_lstm.py Merge pull request !16850 from yuhaiyan/v2.3.1-dev1
37696b11d | BUGFIX | !16853 Fixed test_bidirectional_lstm.py Merge pull request !16853 from yuhaiyan/v2.5.1-dev1
fb7fe4aac | BUGFIX | !16854 Fixed test_bidirectional_lstm.py Merge pull request !16854 from yuhaiyan/master-dev1
35c025e2e | BUGFIX | !16693 Fixed for the public API. Merge pull request !16693 from yuhaiyan/v2.4.0-6.0.0-dev1
c265329b9 | BUGFIX | !16692 Fixed for the public API. Merge pull request !16692 from yuhaiyan/v2.3.1-6.0.0-dev1
befa0f5b7 | BUGFIX | !16691 Fixed for the public API. Merge pull request !16691 from yuhaiyan/v2.1.0-6.0.0-dev1
5bd2bbf27 | BUGFIX | !16684 Fixed for the public API. Merge pull request !16684 from yuhaiyan/v2.4.0-dev1
903fadc4d | BUGFIX | !16681 Fixed for the public API. Merge pull request !16681 from yuhaiyan/v2.1.0-dev1
17095a211 | BUGFIX | !16683 Fixed for the public API. Merge pull request !16683 from yuhaiyan/v2.3.1-dev1
fbb17a2f5 | BUGFIX | !16685 Fixed for the public API. Merge pull request !16685 from yuhaiyan/v2.5.1-dev1
4d34c0297 | BUGFIX | !16690 Fixed for the public API. Merge pull request !16690 from yuhaiyan/master-dev1
9ae16f183 | BUGFIX | !16613 【PROF】【BUG】V2.4.0-6.0.0rc3: add start step for dynamic profiling Merge pull request !16613 from liyou_b/bug_fixed_6.0rc3_240
92574d141 | BUGFIX | !16616 【PROF】【BUG】V2.3.1-6.0.0rc3: add start step for dynamic profiling Merge pull request !16616 from liyou_b/bug_fixed_6.0rc3_231
4ad973c47 | BUGFIX | !16614 【PROF】【BUG】V2.1.0-6.0.0rc3: add start step for dynamic profiling Merge pull request !16614 from liyou_b/bug_fixed_6.0rc3_210
0a6f87fc9 | BUGFIX | !16490 【PROF】【BUG】V2.4.0-6.0.0: add start step for dynamic profiling Merge pull request !16490 from liyou_b/bug_fixed_6.0_240
382e0351e | BUGFIX | !16489 【PROF】【BUG】V2.3.1-6.0.0: add start step for dynamic profiling Merge pull request !16489 from liyou_b/bug_fixed_6.0_231
8548f379c | BUGFIX | !16488 【PROF】【BUG】V2.1.0-6.0.0: add start step for dynamic profiling Merge pull request !16488 from liyou_b/bug_fixed_6.0_210
b9c78de79 | BUGFIX | !16648 Fixed for the public API. Merge pull request !16648 from yuhaiyan/v2.1.0-6.0.0-dev2
2d098799f | BUGFIX | !16649 Fixed for the public API. Merge pull request !16649 from yuhaiyan/v2.3.1-6.0.0-dev2
8ceffb7f6 | BUGFIX | !16650 Fixed for the public API. Merge pull request !16650 from yuhaiyan/v2.4.0-6.0.0-dev2
5225d6aa3 | BUGFIX | !16647 Fixed for the public API. Merge pull request !16647 from yuhaiyan/master-dev6
4a92f8f9a | BUGFIX | !16646 Fixed for the public API. Merge pull request !16646 from yuhaiyan/v2.5.1-dev6
c9860d3e6 | BUGFIX | !16645 Fixed for the public API. Merge pull request !16645 from yuhaiyan/v2.4.0-dev6
b00b02de7 | BUGFIX | !16644 Fixed for the public API. Merge pull request !16644 from yuhaiyan/v2.3.1-dev6
8283f4e1e | BUGFIX | !16643 Fixed for the public API. Merge pull request !16643 from yuhaiyan/v2.1.0-dev6
5a5782aac | BUGFIX | !16147 【PROF】【BUG】V2.1.0: add start step for dynamic profiling Merge pull request !16147 from liyou_b/bug_fixed_tr6_210
5ee399af7 | BUGFIX | !16144 【PROF】【BUG】V2.5.1: add start step for dynamic profiling Merge pull request !16144 from liyou_b/bug_fixed_tr6_251
cdbc5850d | BUGFIX | !16145 【PROF】【BUG】V2.4.0: add start step for dynamic profiling Merge pull request !16145 from liyou_b/bug_fixed_tr6_240
78b9079be | BUGFIX | !16146 【PROF】【BUG】V2.3.1 add start step for dynamic profiling Merge pull request !16146 from liyou_b/bug_fixed_tr6_231
30206a682 | BUGFIX | !16143 【PROF】【BUG】Master: add start step for dynamic profiling Merge pull request !16143 from liyou_b/bug_fixed_tr6_master
e3c37645b | BUGFIX | !16303 Fix the bug of incorrect default values for the dynamicQuantization operator dst_type Merge pull request !16303 from Tangmenhao/v2.1.0_1125
835ae41ba | BUGFIX | !16224 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16224 from 王超/cherry-pick-1732181620
42e86510b | BUGFIX | !16225 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16225 from 王超/cherry-pick-1732181762
c990923da | BUGFIX | !16226 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16226 from 王超/cherry-pick-1732181818
8f0ec202a | BUGFIX | !16222 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16222 from 王超/cherry-pick-1732181484
68199d58e | BUGFIX | !16221 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16221 from 王超/cherry-pick-1732181438
c8df4c007 | BUGFIX | !16218 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16218 from 王超/v2.1.0_hcclbuffsize
17bce4ff2 | BUGFIX | !16220 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16220 from 王超/cherry-pick-1732181356
c2281b974 | BUGFIX | !16223 bugfix: Create hccl common with HCCL_BUFFSIZE when using the RANK_TABLE_FILE method Merge pull request !16223 from 王超/cherry-pick-1732181531
896dc891d | BUGFIX | !16149 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16149 from 王超/cherry-pick-1732068824
3d15026fc | BUGFIX | !16148 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16148 from 王超/cherry-pick-1732068762
c8ff2208d | BUGFIX | !16131 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16131 from 王超/cherry-pick-1732017886
f1af634e6 | BUGFIX | !16032 [PROF]fix the incorrect order of torch_op input type and shape Merge pull request !16032 from xuzhubin/v2.1.0
13db5471e | BUGFIX | !16031 [PROF]fix the incorrect order of torch_op input type and shape Merge pull request !16031 from xuzhubin/v2.3.1
6f1e63fd8 | BUGFIX | !16030 [PROF]fix the incorrect order of torch_op input type and shape Merge pull request !16030 from xuzhubin/v2.4.0
b2234bfd1 | BUGFIX | !16029 [PROF]fix the incorrect order of torch_op input type and shape Merge pull request !16029 from xuzhubin/v2.5.1
1ac959e42 | BUGFIX | !16028 [PROF]fix the incorrect order of torch_op input type and shape Merge pull request !16028 from xuzhubin/master
10bd43a82 | BUGFIX | !16171 [PROF] fix mstx patch err in pytorch graph mode Merge pull request !16171 from 梅飞要/2.1
bc5ec9149 | BUGFIX | !16175 [PROF] fix mstx patch err in pytorch graph mode Merge pull request !16175 from 梅飞要/mmm
1db828a23 | BUGFIX | !16174 [PROF] fix mstx patch err in pytorch graph mode Merge pull request !16174 from 梅飞要/2.5
1b13a878c | BUGFIX | !16173 [PROF] fix mstx patch err in pytorch graph mode Merge pull request !16173 from 梅飞要/2.4
486d76589 | BUGFIX | !16172 [PROF] fix mstx patch err in pytorch graph mode Merge pull request !16172 from 梅飞要/2.3
84839918a | BUGFIX | !16120 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16120 from 王超/cherry-pick-1732001127
3821471bb | BUGFIX | !16119 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16119 from 王超/cherry-pick-1732001074
baa150a7b | BUGFIX | !16118 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16118 from 王超/cherry-pick-1732001018
be77f5ddb | BUGFIX | !16117 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16117 from 王超/cherry-pick-1732000820
f561de871 | BUGFIX | !16067 bugfix: UnrecordedCount should clear when taskqueue status is STOP_EXIT Merge pull request !16067 from 王超/v2.1.0_watchdog
0fcf48af1 | BUGFIX | !16094 [Bugfix] remove duplicate destroyHcclComm and disable heartbeat monitor by default Merge pull request !16094 from weili10/v2.1.0
018838294 | BUGFIX | !16104 [bugfix] db写入死锁 Merge pull request !16104 from Sean1840/cherry-pick-1731980288
8ecbc5956 | BUGFIX | !16103 [bugfix] db写入死锁 Merge pull request !16103 from Sean1840/cherry-pick-1731980270
a551968c0 | BUGFIX | !16102 [bugfix] db写入死锁 Merge pull request !16102 from Sean1840/cherry-pick-1731980249
0f83e9e20 | BUGFIX | !16021 [PROF]Fix: Do not output empty JSON file Merge pull request !16021 from chenjunjie/check_240rc6
861a733f5 | BUGFIX | !16020 [PROF]Fix: Do not output empty JSON file Merge pull request !16020 from chenjunjie/check_231rc6
0feee4aa2 | BUGFIX | !16019 [PROF]Fix: Do not output empty JSON file Merge pull request !16019 from chenjunjie/check_210rc6
b429f3c6f | BUGFIX | !16038 [bugfix] db写入死锁 Merge pull request !16038 from Sean1840/master
148aa4aae | BUGFIX | !16061 [bugfix] db写入死锁 Merge pull request !16061 from Sean1840/dev-2.5.1
1770e1329 | BUGFIX | !16062 [bugfix] db写入死锁 Merge pull request !16062 from Sean1840/dev-2.4
dc93a0527 | BUGFIX | !16063 [bugfix] db写入死锁 Merge pull request !16063 from Sean1840/dev-2.3.1
d2232316d | BUGFIX | !16064 [bugfix] db写入死锁 Merge pull request !16064 from Sean1840/dev-2.1.0
9c854a09c | BUGFIX | !15976 [PROF]Fix: Do not output empty JSON file Merge pull request !15976 from chenjunjie/check_251
a066f4764 | BUGFIX | !15975 [PROF]Fix: Do not output empty JSON file Merge pull request !15975 from chenjunjie/check_240
574d22afe | BUGFIX | !15974 [PROF]Fix: Do not output empty JSON file Merge pull request !15974 from chenjunjie/check_231
858398166 | BUGFIX | !15973 [PROF]Fix: Do not output empty JSON file Merge pull request !15973 from chenjunjie/check_210
d97301efe | BUGFIX | !15972 [PROF]Fix: Do not output empty JSON file Merge pull request !15972 from chenjunjie/check_master
470452e7f | BUGFIX | !15911 [bugfix] step info 表导出失败 Merge pull request !15911 from Sean1840/v2.1.0-6.0.0
e9aa18d55 | BUGFIX | !15910 [bugfix] step info 表导出失败 Merge pull request !15910 from Sean1840/v2.3.1-6.0.0
38f844922 | BUGFIX | !15909 [bugfix] step info 表导出失败 Merge pull request !15909 from Sean1840/v2.4.0-6.0.0
938310f74 | BUGFIX | !15791 [bugfix] step info 表导出失败 Merge pull request !15791 from Sean1840/master
7571f58bb | BUGFIX | !15792 [bugfix] step info 表导出失败 Merge pull request !15792 from Sean1840/dev-2.5.1
70032b8e4 | BUGFIX | !15793 [bugfix] step info 表导出失败 Merge pull request !15793 from Sean1840/dev-2.4
a51476d92 | BUGFIX | !15794 [bugfix] step info 表导出失败 Merge pull request !15794 from Sean1840/dev-2.3.1
6b7f1eca2 | BUGFIX | !15795 [bugfix] step info 表导出失败 Merge pull request !15795 from Sean1840/dev-2.1.0
580b78f61 | BUGFIX | !15757 Fix the foreach_optim Merge pull request !15757 from 张向龙3/fix_patch_25
5de781c7b | BUGFIX | !15758 Fix the foreach_optim Merge pull request !15758 from 张向龙3/fix_patch_26
755d41728 | BUGFIX | !15598 【PROF】【Bug】V2.4.0: Fixed bug when enable DB export type and profiler_memory Merge pull request !15598 from liyou_b/bug_fix_db_v2.4.0
a90a053b6 | BUGFIX | !15196 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15196 from lilei zheng/cherry-pick-1728385947
b8565edde | BUGFIX | !15194 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15194 from lilei zheng/cherry-pick-1728385893
6efd72ed7 | BUGFIX | !15192 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15192 from lilei zheng/cherry-pick-1728385829
6cb91f651 | BUGFIX | !15191 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15191 from lilei zheng/cherry-pick-1728377988
97ae53a25 | BUGFIX | !15190 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15190 from lilei zheng/cherry-pick-1728377926
6025cdce5 | BUGFIX | !15189 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15189 from lilei zheng/cherry-pick-1728377862
6aba1934e | BUGFIX | !15155 Fix the accuracy issue in the aclop conv3d fp32 scenario Merge pull request !15155 from lilei zheng/fix_conv3d_fp32_acl_op_bug
b7504311a | BUGFIX | !15180 【bugfix】fix the bug of intreactive Merge pull request !15180 from 郭光浩/v2.1.0
bd8189ee0 | BUGFIX | !15179 【bugfix】fix the bug of intreactive Merge pull request !15179 from 郭光浩/v2.4.0
9b673343d | BUGFIX | !15178 【bugfix】fix the bug of intreactive Merge pull request !15178 from 郭光浩/v2.3.1
2560d9259 | BUGFIX | !15177 【bugfix】fix the bug of intreactive Merge pull request !15177 from 郭光浩/v2.4.0-6.0.rc3
0672cf7c2 | BUGFIX | !15176 【bugfix】fix the bug of intreactive Merge pull request !15176 from 郭光浩/v2.3.1-6.0.rc3
a62d352dd | BUGFIX | !15175 【bugfix】fix the bug of intreactive Merge pull request !15175 from 郭光浩/v2.1.0-6.0.rc3
cf14e09e5 | BUGFIX | !15174 【bugfix】fix the bug of intreactive Merge pull request !15174 from 郭光浩/master
095e0c4c8 | BUGFIX | !15052 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15052 from 王超/v2.1.0-6.0.rc3_fix
2bc96781a | BUGFIX | !15053 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15053 from 王超/v2.3.1-6.0.rc3_fix
087c236fc | BUGFIX | !15051 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15051 from 王超/v2.4.0-6.0.rc3_fix
e09bff8a8 | BUGFIX | !15046 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15046 from 王超/v2.1.0_fix
35973013f | BUGFIX | !15047 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15047 from 王超/v2.3.1_fix
d7178a68e | BUGFIX | !15050 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15050 from 王超/v2.4.0_fix
c25460d5d | BUGFIX | !15049 ranktable bug fix: global processgroup may not the first to be created Merge pull request !15049 from 王超/master_fix
5f289ba62 | BUGFIX | !15097 [Bug] Fix profiler db small timeout value on v2.4.0-6.0.rc3 Merge pull request !15097 from Mrtutu/db_timeout_v2.4.0-6.0.rc3
ded1cc36d | BUGFIX | !15096 [Bug] Fix profiler db small timeout value on v2.3.1-6.0.rc3 Merge pull request !15096 from Mrtutu/db_timeout_v2.3.1-6.0.rc3
5205b032d | BUGFIX | !15093 [Bug] Fix profiler db small timeout value on v2.1.0-rc3 Merge pull request !15093 from Mrtutu/db_timeout_v2.1.0-6.0.rc3
5fffefb0a | BUGFIX | !15082 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15082 from 王超/v2.1.0-6.0.rc3_uce2
13ed17c12 | BUGFIX | !15081 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15081 from 王超/v2.3.1-6.0.rc3_uce2
e93776181 | BUGFIX | !15079 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15079 from 王超/v2.4.0-6.0.rc3_uce1
bf3f16638 | BUGFIX | !15067 [Bug] Fix profiler db small timeout value on v2.3.1 Merge pull request !15067 from Mrtutu/db_timeout_v2.3.1
d4d250ce5 | BUGFIX | !15065 [Bug] Fix profiler db small timeout value on master Merge pull request !15065 from Mrtutu/db_timeout
a9db2d23d | BUGFIX | !15068 [Bug] Fix profiler db small timeout value on v2.4.0 Merge pull request !15068 from Mrtutu/db_timeout_v2.4.0
2a97e8dd8 | BUGFIX | !15066 [Bug] Fix profiler db small timeout value on v2.1.0 Merge pull request !15066 from Mrtutu/db_timeout_v2.1.0
0560a38ca | BUGFIX | !15060 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15060 from 王超/v2.4.0_uce1
6a6c7d125 | BUGFIX | !15061 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15061 from 王超/v2.3.1_uce1
1fddac5f5 | BUGFIX | !15058 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15058 from 王超/v2.1.0_uce1
7d9801c83 | BUGFIX | !15059 uce bug fix: In order to prevent the dequeue thread from terminating, ReadQueue should set uce status. Merge pull request !15059 from 王超/master_uce1
492910ad8 | BUGFIX | !15002 v2.1.0-6.0-rc3-bugfix Merge pull request !15002 from tangmengcheng/v2.1.0-6.0-rc3-buf-fix
77bf4a5b6 | BUGFIX | !15003 v2.3.1-6.0.rc3-bugfix Merge pull request !15003 from tangmengcheng/v2.3.1-6.0.rc3-bugfix
71d40df5b | BUGFIX | !15004 v2.4.0-6.0.rc3-bugfix Merge pull request !15004 from tangmengcheng/v2.4.0-6.0.rc3-bug-fix
4c99b92ac | BUGFIX | !14981 【v2.4.0】subprocess bug fix Merge pull request !14981 from tangmengcheng/v2.4.0_subprocess
47cfaac58 | BUGFIX | !14980 【v2.3.1】subprocess bug fix Merge pull request !14980 from tangmengcheng/v2.3.1_subprocess
423b9c3f9 | BUGFIX | !14979 【v2.1.0】subprocess bug fix Merge pull request !14979 from tangmengcheng/v2.1.0_subprocess
28843179b | BUGFIX | !14978 【master】subprocess bug fix Merge pull request !14978 from tangmengcheng/master_subprocess
63a4835ca | BUGFIX | !14412 fixed test_torch.py Merge pull request !14412 from yuhaiyan/v2.4.0-dev1
0358d8125 | BUGFIX | !14419 fixed test_torch.py Merge pull request !14419 from yuhaiyan/master-dev2
199b2e16e | BUGFIX | !14322 fixed test_torch.py Merge pull request !14322 from yuhaiyan/v2.1.0-dev1
b2eeeb0de | BUGFIX | !14411 fixed test_torch.py Merge pull request !14411 from yuhaiyan/v2.3.1-dev1
8dd144706 | BUGFIX | !14861 [PROF] fix mstx.range_start err without input stream Merge pull request !14861 from 梅飞要/2.4_rc3
684b30818 | BUGFIX | !14860 [PROF] fix mstx.range_start err without input stream Merge pull request !14860 from 梅飞要/2.3_rc3
c92557a83 | BUGFIX | !14859 [PROF] fix mstx.range_start err without input stream Merge pull request !14859 from 梅飞要/2.1_rc3
f6c9fa2af | BUGFIX | !14876 【Bugfix】Fix profiler task_manager sleep time on v2.1.0-6.0.rc3 Merge pull request !14876 from Mrtutu/task_mgr_v2.1.0-6.0.rc3
1dc862af7 | BUGFIX | !14877 【Bugfix】Fix profiler task_manager sleep time on v2.3.1-6.0.rc3 Merge pull request !14877 from Mrtutu/task_mgr_v2.3.1-6.0.rc3
e98fa36ba | BUGFIX | !14878 【Bugfix】Fix profiler task_manager sleep time on v2.4.0.rc3 Merge pull request !14878 from Mrtutu/task_mgr_v2.4.0-6.0.rc3
4736e2ccb | BUGFIX | !14855 [PROF] fix mstx.range_start err without input stream Merge pull request !14855 from 梅飞要/txxxxx
c792bdf90 | BUGFIX | !14853 [PROF] fix mstx.range_start err without input stream Merge pull request !14853 from 梅飞要/tx_2.1
2cf3a07a4 | BUGFIX | !14854 [PROF] fix mstx.range_start err without input stream Merge pull request !14854 from 梅飞要/tx_2.3
be7ba196a | BUGFIX | !14852 [PROF] fix mstx.range_start err without input stream Merge pull request !14852 from 梅飞要/tx_2.4
f1fe3b1a0 | BUGFIX | !14872 【Bugfix】Fix profiler task_manager sleep time Merge pull request !14872 from Mrtutu/task_mgr_master
4a04b8a4c | BUGFIX | !14873 【Bugfix】Fix profiler task_manager sleep time on v2.1.0 Merge pull request !14873 from Mrtutu/task_mgr_v2.1.0
b61219127 | BUGFIX | !14874 【Bugfix】Fix profiler task_manager sleep time on v2.3.1 Merge pull request !14874 from Mrtutu/task_mgr_v2.3.1
5a12e9e72 | BUGFIX | !14875 【Bugfix】Fix profiler task_manager sleep time on v2.4.0 Merge pull request !14875 from Mrtutu/task_mgr_v2.4.0
4b1e2837f | BUGFIX | !14761 mem uce bug fix Merge pull request !14761 from sunjiayang/mem_uce_240_rc3
b2ea69db9 | BUGFIX | !14763 mem uce bug fix Merge pull request !14763 from sunjiayang/mem_uce_231_rc3
02bcb117b | BUGFIX | !14762 mem uce bug fix Merge pull request !14762 from sunjiayang/mem_uce_210_rc3
6729cd8da | BUGFIX | !14792 [Fix] Fix public bindings. Merge pull request !14792 from 刘嘉巍/v2.4.0-6.0.rc3
e599a0c59 | BUGFIX | !14791 [Fix] Fix public bindings. Merge pull request !14791 from 刘嘉巍/v2.3.1-6.0.rc3
d9d36cbf7 | BUGFIX | !14790 [Fix] Fix public bindings. Merge pull request !14790 from 刘嘉巍/v2.1.0-6.0.rc3
3a34bb7ec | BUGFIX | !14796 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14796 from will-devil/v2.1.0-6.0.rc3
5348ba6fc | BUGFIX | !14797 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14797 from will-devil/v2.3.1-6.0.rc3
8cc4bc183 | BUGFIX | !14798 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14798 from will-devil/v2.4.0-6.0.rc3
69f6e6c18 | BUGFIX | !14780 [Fix] Fix public bindings. Merge pull request !14780 from 刘嘉巍/v2.3.1-dev
3fc55f6e4 | BUGFIX | !14782 [Fix] Fix public bindings. Merge pull request !14782 from 刘嘉巍/master-dev
2bbb5d7f2 | BUGFIX | !14781 [Fix] Fix public bindings. Merge pull request !14781 from 刘嘉巍/v2.4.0-dev
27eb785d1 | BUGFIX | !14756 [Fix] Fix public bindings. Merge pull request !14756 from 刘嘉巍/v2.1.0
500503a16 | BUGFIX | !14734 mem uce bug fix Merge pull request !14734 from sunjiayang/mem_uce_master
7b0461755 | BUGFIX | !14736 mem uce bug fix Merge pull request !14736 from sunjiayang/mem_uce_231
5b1c6d26f | BUGFIX | !14728 mem uce bug fix Merge pull request !14728 from sunjiayang/mem_uce_210
70660968c | BUGFIX | !14735 mem uce bug fix Merge pull request !14735 from sunjiayang/mem_uce_240
fe6d0e58a | BUGFIX | !14661 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14661 from will-devil/v2.4.0
584565e44 | BUGFIX | !14650 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14650 from will-devil/conjm
2836adde0 | BUGFIX | !14651 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14651 from will-devil/conj231
67e3c8a9e | BUGFIX | !14589 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14589 from will-devil/conj21
42bb49b2c | BUGFIX | !14652 [Bugfix] Reduce unnecessary memory allocation. Merge pull request !14652 from will-devil/v1.11.0
27de0972d | BUGFIX | !14612 Bugfix: module 'torch_npu' has no attribute '_C' Merge pull request !14612 from 叶子凡/master_fix
6479f2196 | BUGFIX | !14613 Bugfix: module 'torch_npu' has no attribute '_C' Merge pull request !14613 from 叶子凡/v2.4.0_fix
d08ba2bf5 | BUGFIX | !14614 Bugfix: module 'torch_npu' has no attribute '_C' Merge pull request !14614 from 叶子凡/v2.3.1_fix
a693e9669 | BUGFIX | !14616 Bugfix: module 'torch_npu' has no attribute '_C' Merge pull request !14616 from 叶子凡/v2.1.0_fix
55e570797 | BUGFIX | !14594 Bugfix for dynamic profiler when set wrong args. Merge pull request !14594 from 裘凯达/bugfix_231
14ce764d3 | BUGFIX | !14595 Bugfix for dynamic profiler when set wrong args. Merge pull request !14595 from 裘凯达/bugfix_220
ae0db9bce | BUGFIX | !14588 Bugfix for dynamic profiler when set wrong args. Merge pull request !14588 from 裘凯达/bugfix_210
2bbaa7681 | BUGFIX | !14640 modify print_error_msg with print_warn_msg Merge pull request !14640 from 裘凯达/bugfix_111
6c95f7ac4 | BUGFIX | !14621 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14621 from 梅飞要/fix_mm
a8c2ad867 | BUGFIX | !14622 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14622 from 梅飞要/fix_2.1_rc3
4640f9081 | BUGFIX | !14623 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14623 from 梅飞要/fix_2.3_rc3
2733b9345 | BUGFIX | !14624 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14624 from 梅飞要/fix_2.4_rc3
34d5234f2 | BUGFIX | !14592 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14592 from 梅飞要/fix_2.2
188f2666a | BUGFIX | !14593 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14593 from 梅飞要/fix_2.3
4709bfd39 | BUGFIX | !14590 [PROF]fix profiler report npu mem err in multi thread Merge pull request !14590 from 梅飞要/fix_2.1
c7846a67c | BUGFIX | !14609 [PROF] Bugfixs for dynamic profiler when set wrong args Merge pull request !14609 from 梅飞要/dp_2.2.0
a857e60f2 | BUGFIX | !14610 [PROF] Bugfixs for dynamic profiler when set wrong args Merge pull request !14610 from 梅飞要/dp_2.3
2bac16c78 | BUGFIX | !14596 Bugfix for dynamic profiler when set wrong args. Merge pull request !14596 from 裘凯达/bugfix_111
e300f173a | BUGFIX | !14574 【BUG FIX】-2.4.0- Profiling Python part has not released the open file resources. Merge pull request !14574 from tangmengcheng/v2.4.0_file_close
6b5234e59 | BUGFIX | !14573 【BUG FIX】-2.3.1- Profiling Python part has not released the open file resources. Merge pull request !14573 from tangmengcheng/v2.3.1_file_close
e13523e1a | BUGFIX | !14572 【BUG FIX】-v2.1.0- Profiling Python part has not released the open file resources. Merge pull request !14572 from tangmengcheng/v2.1.0_file_close
09d822f98 | BUGFIX | !14571 【BUG FIX】-master- Profiling Python part has not released the open file resources. Merge pull request !14571 from tangmengcheng/master_file_close
8a908c88a | BUGFIX | !14340 Bug Fix For Non Blocking Check Merge pull request !14340 from 周先琪/master_2
1c5e47a54 | BUGFIX | !14341 Bug Fix For Non Blocking Check Merge pull request !14341 from 周先琪/v1.11.0_2
8bbdd6808 | BUGFIX | !14342 Bug Fix For Non Blocking Check Merge pull request !14342 from 周先琪/v2.1.0_2
585a2d5a6 | BUGFIX | !14345 Bug Fix For Non Blocking Check Merge pull request !14345 from 周先琪/v2.3.1_2
99ce76914 | BUGFIX | !14343 Bug Fix For Non Blocking Check Merge pull request !14343 from 周先琪/v2.2.0_2
756f80bfa | BUGFIX | !14344 Bug Fix For Non Blocking Check Merge pull request !14344 from 周先琪/v2.4.0_2
396877389 | BUGFIX | !14316 Bug Fix For Delete Prof Dir Merge pull request !14316 from 周先琪/master_1
e30e3712e | BUGFIX | !14317 Bug Fix For Delete Prof Dir Merge pull request !14317 from 周先琪/v1.11.0_1
3cdbb1ae1 | BUGFIX | !14318 Bug Fix For Delete Prof Dir Merge pull request !14318 from 周先琪/v2.1.0_1
104937bd9 | BUGFIX | !14319 Bug Fix For Delete Prof Dir Merge pull request !14319 from 周先琪/v2.2.0_1
c32e97836 | BUGFIX | !14320 Bug Fix For Delete Prof Dir Merge pull request !14320 from 周先琪/v2.3.1_1
5343e0bb8 | BUGFIX | !14321 Bug Fix For Delete Prof Dir Merge pull request !14321 from 周先琪/v2.4.0_1
73c4a1ed9 | BUGFIX | !14371 fixed torch_npu_schema.json Merge pull request !14371 from yuhaiyan/master-dev1
3e19c937c | BUGFIX | !14171 [PROF]Fix: variable name and device index Merge pull request !14171 from chenjunjie/mem_dev_1110
c8b7f454e | BUGFIX | !14233 [v2.1.0] fix sparse ut Merge pull request !14233 from DaiFu/v2.1.0
288e41892 | BUGFIX | !14234 [v2.2.0] fix sparse ut Merge pull request !14234 from DaiFu/v2.2.0
2e649fe40 | BUGFIX | !14235 [v2.3.1] fix sparse ut Merge pull request !14235 from DaiFu/v2.3.1
bd2639659 | BUGFIX | !14236 [v2.4.0] fix sparse ut Merge pull request !14236 from DaiFu/v2.4.0
0e72684a5 | BUGFIX | !14232 [master] fix sparse ut Merge pull request !14232 from DaiFu/2407SparseUT
e17b9cc3e | BUGFIX | !14023 【PROF】Master: fixed bug for pytorch dynamic profiling Merge pull request !14023 from liyou_b/it3_fixed_master
604103557 | BUGFIX | !14024 【PROF】V1.11.0: fixed bug for pytorch dynamic profiling Merge pull request !14024 from liyou_b/it3_fixed_111
e51bfb36b | BUGFIX | !14025 【PROF】V2.1.0: fixed bug for pytorch dynamic profiling Merge pull request !14025 from liyou_b/it3_fixed_210
323da971f | BUGFIX | !14026 【PROF】V2.2.0: fixed bug for pytorch dynamic profiling Merge pull request !14026 from liyou_b/it3_fixed_220
72cedbf8d | BUGFIX | !14027 【PROF】V2.3.1: fixed bug for pytorch dynamic profiling Merge pull request !14027 from liyou_b/it3_fixed_231
4ca6ae420 | BUGFIX | !14028 【PROF】V2.4.0: fixed bug for pytorch dynamic profiling Merge pull request !14028 from liyou_b/it3_fixed_240
84c0df049 | BUGFIX | !14079 Fixed some unit tests. Merge pull request !14079 from yuhaiyan/v2.1.0-dev1
9271f94b5 | BUGFIX | !14092 Fixed some unit tests. Merge pull request !14092 from yuhaiyan/v2.2.0-dev1
66ef44363 | BUGFIX | !14093 Fixed some unit tests. Merge pull request !14093 from yuhaiyan/v2.3.1-dev1
253b79836 | BUGFIX | !14095 Fixed some unit tests. Merge pull request !14095 from yuhaiyan/v2.4.0-dev1
31413c723 | BUGFIX | !14100 Fixed some unit tests. Merge pull request !14100 from yuhaiyan/master-dev1
03be6752b | BUGFIX | !14157 [Fix] Fix optim patch Merge pull request !14157 from 刘嘉巍/v2.4.0-dev
9e98679be | BUGFIX | !14156 [Fix] Fix optim patch Merge pull request !14156 from 刘嘉巍/v2.3.1-dev
6a77345b1 | BUGFIX | !14155 [Fix] Fix optim patch Merge pull request !14155 from 刘嘉巍/v2.2.0-dev
8409a9f13 | BUGFIX | !13867 [Fix] Fix optim patch Merge pull request !13867 from 刘嘉巍/v2.1.0
d9f3a49f7 | BUGFIX | !14129 Bugfix: Capture exceptions from the start interface Merge pull request !14129 from tangmengcheng/v2.3.1_bug_fix
e9fd66af3 | BUGFIX | !14130 Bugfix: Capture exceptions from the start interface Merge pull request !14130 from tangmengcheng/v2.4.0_bug_fix
0b42fdf13 | BUGFIX | !14128 Bugfix: Capture exceptions from the start interface Merge pull request !14128 from tangmengcheng/v2.2.0_bug_fix
c181dcf88 | BUGFIX | !14125 Bugfix: Capture exceptions from the start interface Merge pull request !14125 from tangmengcheng/v2.1.0_bug_fix
15580dc01 | BUGFIX | !14127 Bugfix: Capture exceptions from the start interface Merge pull request !14127 from tangmengcheng/v1.11.0_bug_fix
46e52c3d1 | BUGFIX | !14126 Bugfix: Capture exceptions from the start interface Merge pull request !14126 from tangmengcheng/master_bug_fix
1e7a9fb79 | BUGFIX | !14057 Bug Fix For InterreputedError Merge pull request !14057 from 周先琪/v2.4.0
5ab9d35cc | BUGFIX | !14062 Bug Fix For InterreputedError Merge pull request !14062 from 周先琪/v2.3.1
3710b6b96 | BUGFIX | !14061 Bug Fix For InterreputedError Merge pull request !14061 from 周先琪/v2.2.0
a80d944c4 | BUGFIX | !14060 Bug Fix For InterreputedError Merge pull request !14060 from 周先琪/v2.1.0
3abf3e513 | BUGFIX | !14058 Bug Fix For InterreputedError Merge pull request !14058 from 周先琪/master
10864804e | BUGFIX | !14059 Bug Fix For InterreputedError Merge pull request !14059 from 周先琪/v1.11.0
6e069f72f | BUGFIX | !14018 Fix the foreach_optim Merge pull request !14018 from 张向龙3/fix_patch_23
495ffb8bb | BUGFIX | !14017 Fix the foreach_optim Merge pull request !14017 from 张向龙3/fix_patch_24
a22e7b609 | BUGFIX | !14016 Fix the foreach_optim Merge pull request !14016 from 张向龙3/fix_patch_22
4b58baf3e | BUGFIX | !14009 Fix the foreach_optim Merge pull request !14009 from 张向龙3/fix_patch_21
dc182c63a | BUGFIX | !13794 Bugfix: Custom operators support input keyword arguments Merge pull request !13794 from 王广斌/bugfix_master
ba32b2cc8 | BUGFIX | !13795 Bugfix: Custom operators support input keyword arguments Merge pull request !13795 from 王广斌/bugfix_2.4
9fb9a9f58 | BUGFIX | !13748 Bugfix: Custom operators support input keyword arguments Merge pull request !13748 from 王广斌/bugfix_2.3.0
d3f1efe2e | BUGFIX | !13747 Bugfix: Custom operators support input keyword arguments Merge pull request !13747 from 王广斌/bugfix_2.2
62055febe | BUGFIX | !13745 Bugfix: Custom operators support input keyword arguments Merge pull request !13745 from 王广斌/bugfix_2.1
9cf37d171 | BUGFIX | !13724 [PROF]fix mstx err Merge pull request !13724 from 梅飞要/1.11.0
b2c5805bb | BUGFIX | !13730 [PROF]fix mstx err Merge pull request !13730 from 梅飞要/2.1
56ffb3286 | BUGFIX | !13732 [PROF]fix mstx err Merge pull request !13732 from 梅飞要/2.2
5012165c3 | BUGFIX | !13733 [PROF]fix mstx err Merge pull request !13733 from 梅飞要/2.3
d6d94a8ce | BUGFIX | !13734 [PROF]fix mstx err Merge pull request !13734 from 梅飞要/2.4
fa72dba50 | BUGFIX | !13731 [PROF]fix mstx err Merge pull request !13731 from 梅飞要/mm
2010cdbe1 | BUGFIX | !13009 Fixed for the public APIs. Merge pull request !13009 from yuhaiyan/v2.3.1-dev2
ddd7cada7 | BUGFIX | !13006 Fixed for the public APIs. Merge pull request !13006 from yuhaiyan/master-dev2
2c867c4e4 | BUGFIX | !13013 Fixed for the public APIs. Merge pull request !13013 from yuhaiyan/v2.4.0-dev2
1947ee269 | BUGFIX | !12451 To be consistent with the official code, fix the file 'collect_env.py'. Merge pull request !12451 from yuhaiyan/v2.2.0-dev3
292ac6c53 | BUGFIX | !12837 【fix】fix for test_correct_module_names Merge pull request !12837 from 张伟康/ptmaster_publicapi
85458ad49 | BUGFIX | !12838 【fix】fix for test_correct_module_names Merge pull request !12838 from 张伟康/pt231_publicapi
79624c55c | BUGFIX | !12839 【fix】fix for test_correct_module_names Merge pull request !12839 from 张伟康/pt22_publicapi
0d16d477b | BUGFIX | !12450 To be consistent with the official code, fix the file 'collect_env.py'. Merge pull request !12450 from yuhaiyan/v2.1.0-dev3
15538503e | BUGFIX | !12600 To be consistent with the official code, fix the file 'collect_env.py'. Merge pull request !12600 from yuhaiyan/v2.3.1-dev3
d61a20d3d | BUGFIX | !12603 To be consistent with the official code, fix the file 'collect_env.py'. Merge pull request !12603 from yuhaiyan/master-dev3
4c4915023 | BUGFIX | !12836 【fix】fix for test_correct_module_names Merge pull request !12836 from 张伟康/pt21_publicapi
f41a765e5 | BUGFIX | !12580 [PROF]Bugfix for empty_tensor Merge pull request !12580 from 裘凯达/v2.3.1-6.0.rc2
e26600bbd | BUGFIX | !12581 [PROF]Bugfix for empty_tensor Merge pull request !12581 from 裘凯达/v2.2.0-6.0.rc2
5705b3d09 | BUGFIX | !12582 [PROF]Bugfix for empty_tensor Merge pull request !12582 from 裘凯达/v2.1.0-6.0.rc2
f441e24a6 | BUGFIX | !12583 [PROF]Bugfix for empty_tensor Merge pull request !12583 from 裘凯达/v1.11.0-6.0.rc2
aa40e1cd8 | BUGFIX | !12579 [PROF]Bugfix for empty_tensor Merge pull request !12579 from 裘凯达/v2.3.1
0395424d4 | BUGFIX | !10766 [PROF]Bugfix for empty_tensor Merge pull request !10766 from 裘凯达/v2.2.0
836c5a543 | BUGFIX | !10767 [PROF]Bugfix for empty_tensor Merge pull request !10767 from 裘凯达/v2.1.0
a3b3f1466 | BUGFIX | !12764 [PROF]Bugfix for empty_tensor Merge pull request !12764 from 裘凯达/v1.11.0
256f5bf9a | BUGFIX | !10765 [PROF]Bugfix for empty_tensor Merge pull request !10765 from 裘凯达/master
5efeb3711 | BUGFIX | !12710 Fix the relative path of op-plugin's UT to run load_all_ut Merge pull request !12710 from 张向龙3/fix_ld_ut_11
c7cdc0057 | BUGFIX | !12687 Fix the relative path of op-plugin's UT to run load_all_ut Merge pull request !12687 from 张向龙3/fix_ld_ut_24
ce26ff295 | BUGFIX | !12702 Fix the relative path of op-plugin's UT to run load_all_ut Merge pull request !12702 from 张向龙3/fix_ld_ut_23
43f030537 | BUGFIX | !12698 Fix the relative path of op-plugin's UT to run load_all_ut Merge pull request !12698 from 张向龙3/fix_ld_ut_22
fdb44f540 | BUGFIX | !12697 Fix the relative path of op-plugin's UT to run load_all_ut Merge pull request !12697 from 张向龙3/fix_ld_ut_21
f42a57b11 | BUGFIX | !12595 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12595 from 王超/v2.1.0-6.0.rc2_ut
331698260 | BUGFIX | !12596 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12596 from 王超/v2.2.0-6.0.rc2_ut
28c63a98e | BUGFIX | !12597 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12597 from 王超/v2.3.1-6.0.rc2_ut
fefc27222 | BUGFIX | !12271 [fix] fix assertValidDevice bug. Merge pull request !12271 from 杜金航/v1.11.0_cleancode
ebb6df013 | BUGFIX | !12510 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12510 from 王超/v2.1.0_improve
dbadcdd8a | BUGFIX | !12511 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12511 from 王超/v2.2.0_improve
0d40d2bf9 | BUGFIX | !12512 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12512 from 王超/v2.3.1_ut
9746bcfe6 | BUGFIX | !12513 Fix the execution mode of ops test cases by removing the init_method parameter. Merge pull request !12513 from 王超/master_ut
6c82b702a | BUGFIX | !12338 ut fix with test_common_rules Merge pull request !12338 from 王超/master_case
26e2b1fb2 | BUGFIX | !12297 Fix the failed tests with dtensor. Skip some unmatched test cases. Merge pull request !12297 from 王超/master_problem
2c6cc3547 | BUGFIX | !12293 Fix the failed tests with dtensor Merge pull request !12293 from 王超/v2.2.0_problem
2eef82967 | BUGFIX | !12296 Fix the failed tests with dtensor. Skip some unmatched test cases. Merge pull request !12296 from 王超/v2.3.1_problem
655bb61db | BUGFIX | !12300 Fixed for the public APIs. Merge pull request !12300 from yuhaiyan/v2.1.0-dev10
4ee8ead91 | BUGFIX | !12298 Fixed for the public APIs. Merge pull request !12298 from yuhaiyan/v2.2.0-dev7
c81a670c0 | BUGFIX | !12263 Fixed for the public APIs. Merge pull request !12263 from yuhaiyan/v2.1.0-dev9
57c0d892f | BUGFIX | !12288 [PROF]fix torch op api handle err Merge pull request !12288 from 梅飞要/fix_main
7d15dea19 | BUGFIX | !12289 [PROF]fix torch op api handle err Merge pull request !12289 from 梅飞要/fix_1.11.0
3c3697637 | BUGFIX | !12290 [PROF]fix torch op api handle err Merge pull request !12290 from 梅飞要/fix_2.1.0
c79eeeac8 | BUGFIX | !12291 [PROF]fix torch op api handle err Merge pull request !12291 from 梅飞要/fix_2.2.0
3301c06cf | BUGFIX | !12292 [PROF]fix torch op api handle err Merge pull request !12292 from 梅飞要/fix_2.3.1
5edbe2b84 | BUGFIX | !12221 [PROF]fix prof public apis Merge pull request !12221 from 梅飞要/cl_2.1.0
9c8983a94 | BUGFIX | !12223 [PROF]fix prof public apis Merge pull request !12223 from 梅飞要/cl_2.2.0
8b16b6335 | BUGFIX | !12222 [PROF]fix prof public apis Merge pull request !12222 from 梅飞要/cl_2.3.1
b2ba1b293 | BUGFIX | !12220 [PROF]fix prof public apis Merge pull request !12220 from 梅飞要/cl_main
50a5c57f1 | BUGFIX | !12143 [Fix] Fix testcase & support name for several apis. Merge pull request !12143 from 刘嘉巍/v2.3.0
ceff50bcf | BUGFIX | !12144 [Fix] Fix testcase & support name for several apis. Merge pull request !12144 from 刘嘉巍/master
8b2d2bc5c | BUGFIX | !12142 [Fix] Fix testcase & support name for several apis. Merge pull request !12142 from 刘嘉巍/v2.2.0
09450fa23 | BUGFIX | !12204 Fix the failed tests. Merge pull request !12204 from yuhaiyan/master-dev4
7ca4bf758 | BUGFIX | !12203 Fix the failed tests. Merge pull request !12203 from yuhaiyan/v2.3.1-dev4
6a772e8dc | BUGFIX | !12202 Fix the failed tests. Merge pull request !12202 from yuhaiyan/v2.2.0-dev4
1b6d31470 | BUGFIX | !12194 Fix the failed tests. Merge pull request !12194 from yuhaiyan/v2.1.0-dev4
39976d38b | BUGFIX | !12028 [Fix] Fix testcase & support name for  several apis. Merge pull request !12028 from 刘嘉巍/v2.1.0-named-dev
4dd120867 | BUGFIX | !11981 BugFix: shouldn't compare boolean values to True or False using '==' or 'is'. Merge pull request !11981 from yuhaiyan/v2.1.0-dev4
2cd9fd3ad | BUGFIX | !11982 BugFix: shouldn't compare boolean values to True or False using '==' or 'is'. Merge pull request !11982 from yuhaiyan/v2.2.0-dev4
3508c211b | BUGFIX | !11983 BugFix: shouldn't compare boolean values to True or False using '==' or 'is'. Merge pull request !11983 from yuhaiyan/v2.3.1-dev4
5a37004c0 | BUGFIX | !11984 BugFix: shouldn't compare boolean values to True or False using '==' or 'is'. Merge pull request !11984 from yuhaiyan/master-dev4
0c68ea0f1 | BUGFIX | !11980 BugFix: shouldn't compare boolean values to True or False using '==' or 'is'. Merge pull request !11980 from yuhaiyan/v1.11.0-dev4
186d11b41 | BUGFIX | !11794 Use correct error class in unavailable_type Merge pull request !11794 from wuhy/unavailable_type_bugfix_1110
e5b9af0c6 | BUGFIX | !11795 Use correct error class in unavailable_type Merge pull request !11795 from wuhy/unavailable_type_bugfix_21
88eb05cec | BUGFIX | !11797 Use correct error class in unavailable_type Merge pull request !11797 from wuhy/unavailable_type_bugfix_22
ccc88d436 | BUGFIX | !11919 Fixed the bug that the dynamic quant op fails to export ONNX Merge pull request !11919 from Tangmenhao/2.3.1_0521
950ba77da | BUGFIX | !11755 Fixed the bug that the dynamic quant op fails to export ONNX Merge pull request !11755 from Tangmenhao/torch_2.1_515
5a2b9bc37 | BUGFIX | !11765 Fixed the bug that the dynamic quant op fails to export ONNX Merge pull request !11765 from Tangmenhao/torch_2.2_515
89217a06e | BUGFIX | !11816 bugfix: rankid can not trans to device id, use current device Merge pull request !11816 from yangxiaorun/cherry-pick-1715995071
2e97e2dce | BUGFIX | !11815 bugfix: rankid can not trans to device id, use current device Merge pull request !11815 from yangxiaorun/cherry-pick-1715995004
e2b9096e0 | BUGFIX | !11814 bugfix: rankid can not trans to device id, use current device Merge pull request !11814 from yangxiaorun/cherry-pick-1715994899
cc7805d53 | BUGFIX | !11653 bugfix: rankid can not trans to device id, use current device Merge pull request !11653 from yangxiaorun/get_hcom_name_bug_fix
36065a8b2 | BUGFIX | !11751 【torchair】fix torchair lazy init bug about sys.modules. Merge pull request !11751 from 杜承昆/warning_master
d996ff171 | BUGFIX | !11750 【torchair】fix torchair lazy init bug about sys.modules. Merge pull request !11750 from 杜承昆/warning_v2.3.0
0d1e0cf59 | BUGFIX | !11749 【torchair】fix torchair lazy init bug about sys.modules. Merge pull request !11749 from 杜承昆/warning_v2.2.0
7aac6ba20 | BUGFIX | !11748 【torchair】fix torchair lazy init bug about sys.modules. Merge pull request !11748 from 杜承昆/warning_v2.1.0
b2c86d4bf | BUGFIX | !11549 asd patch bugfix Merge pull request !11549 from sunjiayang/asd_patch_master
b3775c72f | BUGFIX | !11548 asd patch bugfix Merge pull request !11548 from sunjiayang/asd_patch_230
1af4e3e0b | BUGFIX | !11579 asd patch bugfix Merge pull request !11579 from sunjiayang/asd_patch_210
df7d76564 | BUGFIX | !11550 asd patch bugfix Merge pull request !11550 from sunjiayang/asd_patch_220
7215bee31 | BUGFIX | !11551 asd patch bugfix Merge pull request !11551 from sunjiayang/asd_patch_111
cc987e691 | BUGFIX | !11504 Fixed for the public APIs. Merge pull request !11504 from yuhaiyan/master-dev3
ad2102331 | BUGFIX | !10962 Add functionalize codegen feature Merge pull request !10962 from 王广斌/bugfix_master
590e2799a | BUGFIX | !10963 Add functionalize codegen feature Merge pull request !10963 from 王广斌/bugfix_2.3.0
f329f9c44 | BUGFIX | !10960 Add functionalize codegen feature Merge pull request !10960 from 王广斌/bugfix_2.2
cb272f655 | BUGFIX | !11336 Syscnt Bugfix for Ascned PyTorch Profiler. Merge pull request !11336 from 裘凯达/pta111_500_bug
cab2809ee | BUGFIX | !11338 Syscnt Bugfix for Ascned PyTorch Profiler. Merge pull request !11338 from 裘凯达/pta210_500_bug
7e7e6922d | BUGFIX | !11337 Syscnt Bugfix for Ascned PyTorch Profiler. Merge pull request !11337 from 裘凯达/pta201_500_bug
af6612899 | BUGFIX | !11276 Fixed for the public API. Merge pull request !11276 from yuhaiyan/v2.1.0-dev2
f3e32d85b | BUGFIX | !11179 Fixed for the public APIs. Merge pull request !11179 from yuhaiyan/v2.1.0-dev4
f95216d05 | BUGFIX | !11163 Fixed for the public APIs. Merge pull request !11163 from yuhaiyan/v2.2.0-dev3
e96b53a92 | BUGFIX | !11204 Fixed for the public APIs. Merge pull request !11204 from yuhaiyan/v2.3.0-dev3
cd163d152 | BUGFIX | !11073 Fixed ParallelStore bug: When thousands of clients wait for the same key at the same time, a few clients cannot be woken up Merge pull request !11073 from brian-liu9973/v1.11.0-6.0.rc1
bc4b4ff8c | BUGFIX | !11072 Fixed ParallelStore bug: When thousands of clients wait for the same key at the same time, a few clients cannot be woken up Merge pull request !11072 from brian-liu9973/v1.11.0
51276f85d | BUGFIX | !10993 [PROF]bugfix: avoid ReDos risk Merge pull request !10993 from stby/cherry-pick-1712475969
122abc245 | BUGFIX | !10992 [PROF]bugfix: avoid ReDos risk Merge pull request !10992 from stby/cherry-pick-1712475957
17c8a582f | BUGFIX | !10991 [PROF]bugfix: avoid ReDos risk Merge pull request !10991 from stby/cherry-pick-1712475915
b76f9419a | BUGFIX | !10990 【Bugfix】fix ReDos and div zero risk on branch v2.2.0-6.0.rc1 Merge pull request !10990 from Mrtutu/cherry-pick-1712475132
d7630f43b | BUGFIX | !10989 【Bugfix】fix ReDos and div zero risk on branch v2.1.0-6.0.rc1 Merge pull request !10989 from Mrtutu/cherry-pick-1712474870
d9102c87e | BUGFIX | !10987 【Bugfix】fix ReDos and div zero risk on branch v1.11.0-6.0rc1 Merge pull request !10987 from Mrtutu/cherry-pick-1712474678
688839b8f | BUGFIX | !10937 [fix] fix segmentation fault without npu uninitialized. Merge pull request !10937 from 杜金航/master
d1689e7bb | BUGFIX | !10935 [fix] fix segmentation fault without npu uninitialized. Merge pull request !10935 from 杜金航/v2.2.0
d1ed33d8f | BUGFIX | !10934 [fix] fix segmentation fault without npu uninitialized. Merge pull request !10934 from 杜金航/v2.1.0
c1d731578 | BUGFIX | !10936 [fix] fix segmentation fault without npu uninitialized. Merge pull request !10936 from 杜金航/v2.3.0
38c44b9ca | BUGFIX | !10451 [PROF] Fix profiling warmup action Merge pull request !10451 from wangjie/profiling_action_master
89d6d56f2 | BUGFIX | !10450 [PROF] Fix profiling warmup action Merge pull request !10450 from wangjie/profiling_action_220
84297dda4 | BUGFIX | !10449 [PROF] Fix profiling warmup action Merge pull request !10449 from wangjie/profiling_action_210
6489cad74 | BUGFIX | !10426 [PROF] Fix profiling warmup action Merge pull request !10426 from wangjie/profiling_action_111
cb0e5f7ce | BUGFIX | !10719 Fixed ParallelStore add connection to epoll failed: Bad file descriptor. Merge pull request !10719 from brian-liu9973/liuqzh_v1.11.0-v6
3d2a69687 | BUGFIX | !10842 [PROF]bugfix: avoid ReDos risk Merge pull request !10842 from stby/bugfix_330_220
f86630b6e | BUGFIX | !10843 [PROF]bugfix: avoid ReDos risk Merge pull request !10843 from stby/bugfix_330_210
017110a5e | BUGFIX | !10844 [PROF]bugfix: avoid ReDos risk Merge pull request !10844 from stby/bugfix_330_1110
4021f65f3 | BUGFIX | !10840 [PROF]bugfix: avoid ReDos risk Merge pull request !10840 from stby/bugfix_0330
572429bd2 | BUGFIX | !10830 【Bugfix】fix code security on branch master Merge pull request !10830 from Mrtutu/ser_master
d4c10c546 | BUGFIX | !10829 【Bugfix】fix code security on branch v2.2.0 Merge pull request !10829 from Mrtutu/ser_v2.2.0
0ef083612 | BUGFIX | !10828 【Bugfix】fix code security on branch v2.1.0 Merge pull request !10828 from Mrtutu/ser_v2.1.0
691767459 | BUGFIX | !10827 【Bugfix】fix code security on branch v1.11.0 Merge pull request !10827 from Mrtutu/ser_v1.11.0
bb5a9056b | BUGFIX | !10498 Fixed for the public API. Merge pull request !10498 from yuhaiyan/master-dev6
a3eec6a53 | BUGFIX | !10499 Fixed for the public API. Merge pull request !10499 from yuhaiyan/v2.2.0-dev6
44b23d706 | BUGFIX | !10500 Fixed for the public API. Merge pull request !10500 from yuhaiyan/v2.1.0-dev6
1df22ff7d | BUGFIX | !10706 [Cleancode]fix the mix signed and unsigned numbers，explicit initialization of class member variables Merge pull request !10706 from huangyunlong/2.2rch
955def62f | BUGFIX | !10704 [Cleancode]fix the mix signed and unsigned numbers，explicit initialization of class member variables Merge pull request !10704 from huangyunlong/2.1rch
0a304a623 | BUGFIX | !10707 [Cleancode]fix the mix signed and unsigned numbers，explicit initialization of class member variables Merge pull request !10707 from huangyunlong/2.3ch
3b9a4cda1 | BUGFIX | !10703 [Cleancode]fix the mix signed and unsigned numbers，explicit initialization of class member variables Merge pull request !10703 from huangyunlong/2.2ch
fafbfb4a8 | BUGFIX | !10702 [Cleancode]fix the mix signed and unsigned numbers，explicit initialization of class member variables Merge pull request !10702 from huangyunlong/2.1ch
56351782c | BUGFIX | !10548 Fixed ParallelStore add connection to epoll failed: Bad file descriptor. Merge pull request !10548 from brian-liu9973/liuqzh_v1.11.0_store_0322
eca968617 | BUGFIX | !10565 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10565 from 王佰如/v2.2.0-6.0.rc1
4273a6158 | BUGFIX | !10564 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10564 from 王佰如/v2.1.0-6.0.rc1
c4b5f7d81 | BUGFIX | !10563 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10563 from 王佰如/v1.11.0-6.0.rc1
929725c19 | BUGFIX | !10560 bug fix Merge pull request !10560 from fanxiaotong/v1.11.0
7adf8c259 | BUGFIX | !10546 bug fix Merge pull request !10546 from fanxiaotong/master
c389d9210 | BUGFIX | !10585 [bugfix] Patch torch.Generator for npu Merge pull request !10585 from MooYeh/v2.2.0-6.0.rc1-transfer_dts
15f503e3b | BUGFIX | !10584 [bugfix] Patch torch.Generator for npu Merge pull request !10584 from MooYeh/v2.1.0-6.0.rc1-transfer_dts
042ac53be | BUGFIX | !10583 [bugfix] Patch torch.Generator for npu Merge pull request !10583 from MooYeh/v1.11.0-6.0.rc1-transfer_dts
904d48758 | BUGFIX | !10578 [bugfix] Patch torch.Generator for npu Merge pull request !10578 from MooYeh/v1.11.0-transfer_dts
fe15a4812 | BUGFIX | !10580 [bugfix] Patch torch.Generator for npu Merge pull request !10580 from MooYeh/v2.1.0-transfer_dts
e37770cfa | BUGFIX | !10581 [bugfix] Patch torch.Generator for npu Merge pull request !10581 from MooYeh/v2.2.0-transfer_dts
0d293cfbe | BUGFIX | !10582 [bugfix] Patch torch.Generator for npu Merge pull request !10582 from MooYeh/master-transfer_dts
5d4055e11 | BUGFIX | !10482 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10482 from 王佰如/master
19e63315a | BUGFIX | !10539 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10539 from 王佰如/v1.11.0
fad32851a | BUGFIX | !10540 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10540 from 王佰如/v2.1.0
0be977eba | BUGFIX | !10541 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10541 from 王佰如/v2.2.0
19968b7be | BUGFIX | !10429 Bugfix: add saved_for param for unpack_list func Merge pull request !10429 from 王广斌/bugfix_2.1
4b902f96c | BUGFIX | !10443 Bugfix: add saved_for param for unpack_list func Merge pull request !10443 from 王广斌/bugfix_2.2
1287dcaaa | BUGFIX | !10444 Bugfix: add saved_for param for unpack_list func Merge pull request !10444 from 王广斌/bugfix_master
d607445aa | BUGFIX | !10389 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10389 from 王广斌/bugfix_6.0rc1_111
1480d9ace | BUGFIX | !10391 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10391 from 王广斌/bugfix_6.0.rc1_220
9cf6f699a | BUGFIX | !10390 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10390 from 王广斌/bugfix_6.0.rc1-210
e10fe016d | BUGFIX | !10324 Fixed the failed test cases. Merge pull request !10324 from yuhaiyan/v210-6rc-dev1
d7e437d25 | BUGFIX | !10326 Fixed the failed test cases. Merge pull request !10326 from yuhaiyan/v111-6rc1-dev1
9986b244d | BUGFIX | !10322 Fixed the failed test cases. Merge pull request !10322 from yuhaiyan/v2.2.0-6.0.rc1-dev1
8c53a1f72 | BUGFIX | !10362 [Bugfix] fix gather dispatched to hccl backend when another backend is initialized. Merge pull request !10362 from Leon/21_rc1
b5ac020e1 | BUGFIX | !10353 [Bugfix] fix gather dispatched to hccl backend when another backend is initialized. Merge pull request !10353 from Leon/21_
e3942bc04 | BUGFIX | !10363 [Bugfix] fix gather dispatched to hccl backend when another backend is initialized. Merge pull request !10363 from Leon/22_rc1
98bbd70b0 | BUGFIX | !10355 [Bugfix] fix gather dispatched to hccl backend when another backend is initialized. Merge pull request !10355 from Leon/22_
4d2ad2867 | BUGFIX | !10358 [Bugfix] fix gather dispatched to hccl backend when another backend is initialized. Merge pull request !10358 from Leon/23_
02ba0f00f | BUGFIX | !10325 Fixed the failed test case. Merge pull request !10325 from yuhaiyan/v1.11.0-dev1
418a20e72 | BUGFIX | !10323 Fixed the failed test cases. Merge pull request !10323 from yuhaiyan/v2.1.0-dev1
b385a7780 | BUGFIX | !10321 Fixed the failed test case. Merge pull request !10321 from yuhaiyan/v2.2.0-dev1
05bbb2e64 | BUGFIX | !10320 Fixed the failed test cases. Merge pull request !10320 from yuhaiyan/master-dev1
936791087 | BUGFIX | !10276 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10276 from 王广斌/bugfix_master
cfe4bfff2 | BUGFIX | !10278 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10278 from 王广斌/2.2_copy
ca4e0120f | BUGFIX | !10275 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10275 from 王广斌/bugfix_2.1
c10ab30b2 | BUGFIX | !10279 Bugfix: fix copy_ not supported broadcast problem Merge pull request !10279 from 王广斌/bugfix_1.11
9b80a73ac | BUGFIX | !10202 [Refactoring] fix codecheck Merge pull request !10202 from 郭光浩/master
ff215c6a3 | BUGFIX | !10204 [Refactoring] fix codecheck Merge pull request !10204 from 郭光浩/v2.2.0
cda222fb4 | BUGFIX | !10203 [Refactoring] fix codecheck Merge pull request !10203 from 郭光浩/v2.1.0
6da5d64fa | BUGFIX | !10205 [Refactoring] fix codecheck Merge pull request !10205 from 郭光浩/v1.11.0
527a7fb9c | BUGFIX | !10241 [PROF][BUGFIX] judge tables if exist before get data from tables Merge pull request !10241 from 梅飞要/maa
c649b3b09 | BUGFIX | !10242 [PROF][BUGFIX] judge tables if exist before get data from tables Merge pull request !10242 from 梅飞要/ma1.11.0
891a3a07b | BUGFIX | !10248 [PROF][BUGFIX] judge tables if exist before get data from tables Merge pull request !10248 from 梅飞要/ma2.1.0
6e154f9ac | BUGFIX | !10250 [PROF][BUGFIX] judge tables if exist before get data from tables Merge pull request !10250 from 梅飞要/ma2.2.0
c4e2fbe97 | BUGFIX | !10197 [2.0.1]fix is_scalar_wrapped_to_device Merge pull request !10197 from DaiFu/v2.0.1
8de4db69c | BUGFIX | !10196 [1.11.0]fix is_scalar_wrapped_to_device Merge pull request !10196 from DaiFu/v1.11.0
89da42786 | BUGFIX | !10133 fix the way to call aclinit and aclfinalize Merge pull request !10133 from 闫鹏全/master_aclinit
6a86ebe79 | BUGFIX | !10134 fix the way to call aclinit and aclfinalize Merge pull request !10134 from 闫鹏全/v2.2.0
c3976d42b | BUGFIX | !9559 fix the way to call aclinit and aclfinalize Merge pull request !9559 from 闫鹏全/v2.1.0
1a01eb5bd | BUGFIX | !9990 [master]fix is_scalar_wrapped_to_device Merge pull request !9990 from DaiFu/2403scalar_device
9b17c1dac | BUGFIX | !9989 [2.2.0]fix is_scalar_wrapped_to_device Merge pull request !9989 from DaiFu/v2.2.0_fix_scalar_device
4b4cbe26b | BUGFIX | !9987 [2.1.0]fix is_scalar_wrapped_to_device Merge pull request !9987 from DaiFu/v2.1.0
39e90e16c | BUGFIX | !9975 [master]fix apply_tensor_use_empty Merge pull request !9975 from DaiFu/2403fix_PU1
d7cdabe70 | BUGFIX | !9974 [2.2.0]fix apply_tensor_use_empty Merge pull request !9974 from DaiFu/v2.2.0_fix_PU1
d37d2baf9 | BUGFIX | !9607 Fixed for the public API. Merge pull request !9607 from yuhaiyan/v2.1.0-dev1
3f562da33 | BUGFIX | !9608 Fixed for the public API. Merge pull request !9608 from yuhaiyan/v2.2.0-dev1
3f23bac00 | BUGFIX | !9609 Fixed for the public API. Merge pull request !9609 from yuhaiyan/master-dev1
96a6b2d4d | BUGFIX | !9572 fix the problem of the same log printed repeatedly Merge pull request !9572 from dongwenbo6/v1.11.0
fee61752f | BUGFIX | !9578 fix the problem of the same log printed repeatedly Merge pull request !9578 from dongwenbo6/v2.1.0
748c8c3cf | BUGFIX | !9561 bugfix: report frist error in cann Merge pull request !9561 from yangxiaorun/cherry-pick-1708225497
13c0602f7 | BUGFIX | !9560 bugfix: report frist error in cann Merge pull request !9560 from yangxiaorun/v1.11
74ec44e7b | BUGFIX | !9568 Bug Fix For Max Process Number Merge pull request !9568 from 周先琪/master
50df00ba2 | BUGFIX | !9569 Bug Fix For Max Process Number Merge pull request !9569 from 周先琪/v2.2.0
6f1cfabd5 | BUGFIX | !9567 Bug Fix For Max Process Number Merge pull request !9567 from 周先琪/v2.1.0
d7b9d948b | BUGFIX | !9566 Bug Fix For Max Process Number Merge pull request !9566 from 周先琪/v2.0.1
e42645fd2 | BUGFIX | !9565 Bug Fix For Max Process Number Merge pull request !9565 from 周先琪/v1.11.0
0d3bd7d3d | BUGFIX | !9520 bugfix: report frist error in cann Merge pull request !9520 from yangxiaorun/fix_error
9bb8d3e0a | BUGFIX | !9530 bugfix: report frist error in cann Merge pull request !9530 from yangxiaorun/cherry-pick-1707356931
2a4d90cdc | BUGFIX | !9532 bugfix: report frist error in cann Merge pull request !9532 from yangxiaorun/cherry-pick-1707360976
510ef9bb6 | BUGFIX | !9357 [fix] fix autograd.profiler recursive call MakeSureQueueEmpty Merge pull request !9357 from xudaohong/cherry-pick-1706840261
a8f32bd3a | BUGFIX | !9356 [fix] fix autograd.profiler recursive call MakeSureQueueEmpty Merge pull request !9356 from xudaohong/cherry-pick-1706840252
e09fac2ac | BUGFIX | !9358 [fix] fix autograd.profiler recursive call MakeSureQueueEmpty Merge pull request !9358 from xudaohong/cherry-pick-1706840256
e3f9c857c | BUGFIX | !9416 Fixed for the public API. Merge pull request !9416 from yuhaiyan/master-dev2
e772f64c4 | BUGFIX | !9415 Fixed for the public API. Merge pull request !9415 from yuhaiyan/v2.2.0-dev2
458833773 | BUGFIX | !9418 Fix a bug. Merge pull request !9418 from yuhaiyan/v2.1.0-dev1
0bbbc43d8 | BUGFIX | !9354 [fix] fix autograd.profiler recursive call MakeSureQueueEmpty Merge pull request !9354 from xudaohong/v2.0.1
25821b6f5 | BUGFIX | !9355 [fix] fix autograd.profiler recursive call MakeSureQueueEmpty Merge pull request !9355 from xudaohong/cherry-pick-1706840239
255e36775 | REVERT | Revert "Use raise instead of exit to ensure that errors can be captured."
231c3a474 | BUGFIX | !9256 Add Stream Ptr entry in memory_record.csv Merge pull request !9256 from 裘凯达/pta111_bugfix
f74bf385b | BUGFIX | !9277 Add Stream Ptr entry in memory_record.csv Merge pull request !9277 from 裘凯达/pta201_bugfix
fbf5cebbc | BUGFIX | !9280 Add Stream Ptr entry in memory_record.csv Merge pull request !9280 from 裘凯达/master_bugfix
a032837e3 | BUGFIX | !9278 Add Stream Ptr entry in memory_record.csv Merge pull request !9278 from 裘凯达/pta210_bugfix
9bc54bd1c | BUGFIX | !9279 Add Stream Ptr entry in memory_record.csv Merge pull request !9279 from 裘凯达/pta220_bugfix
9c4a3a671 | BUGFIX | fixed for publicAPI
2b301732c | BUGFIX | !9151 Bugfix for Ascend PyTorch Profiler. Merge pull request !9151 from 裘凯达/master_bugfix
a73a2edb4 | BUGFIX | !9152 Bugfix for Ascend PyTorch Profiler. Merge pull request !9152 from 裘凯达/pta220_bugfix
b88cd9c08 | BUGFIX | !9153 Bugfix for Ascend PyTorch Profiler. Merge pull request !9153 from 裘凯达/pta210_bugfix
7fd472027 | BUGFIX | !9154 Bugfix for Ascend PyTorch Profiler. Merge pull request !9154 from 裘凯达/pta201_bugfix
7f1d4e416 | BUGFIX | !9155 Bugfix for Ascend PyTorch Profiler. Merge pull request !9155 from 裘凯达/pta111_bugfix
86a877f78 | BUGFIX | !8852 Fixed the failed test cases. Merge pull request !8852 from 王夏夏/v2.0.1
0e7ed24e7 | BUGFIX | !8853 Fixed the failed test cases. Merge pull request !8853 from 王夏夏/v1.11.0
2ad395251 | BUGFIX | !9009 Fix For Level0 Aic_metrics Conflict Merge pull request !9009 from 周先琪/master
09650ba29 | BUGFIX | !9007 Fix For Level0 Aic_metrics Conflict Merge pull request !9007 from 周先琪/v2.1.0
6a8626d18 | BUGFIX | !9010 Fix For Level0 Aic_metrics Conflict Merge pull request !9010 from 周先琪/v2.0.1
04a5a9ce4 | BUGFIX | !9006 Fix For Level0 Aic_metrics Conflict Merge pull request !9006 from 周先琪/v1.11.0
ff535c80a | BUGFIX | !8915 Fix the testcases for layernorm. Merge pull request !8915 from shaoyf/ut/layernorm_210_500
3e6731b8a | BUGFIX | !8812 Fixed the failed test cases. Merge pull request !8812 from yuhaiyan/v1.11.0-dev1
d95068e48 | BUGFIX | !8811 Fixed the failed test cases. Merge pull request !8811 from yuhaiyan/v2.0.1-dev1
b0533aa89 | BUGFIX | !8723 Fixed the failed test cases. Merge pull request !8723 from yuhaiyan/v2.1.0-dev2
663b99587 | BUGFIX | !8810 Fixed the failed test cases. Merge pull request !8810 from yuhaiyan/master-dev3
6aee2edb3 | BUGFIX | !7042 Fix the problem of UT Merge pull request !7042 from chlme16/multi11
60cea2181 | BUGFIX | !7036 Fix the problem of UT Merge pull request !7036 from chlme16/multi1102
e18b6eaa5 | BUGFIX | !7044 Fix the problem of UT Merge pull request !7044 from chlme16/multi21
8034fad47 | BUGFIX | !7043 Fix the problem of UT Merge pull request !7043 from chlme16/multi20
40f0e8766 | BUGFIX | !8751 Fix the failed UT. Merge pull request !8751 from 文俊/utv1.11.0
b8350e5bc | BUGFIX | !8768 [Fix]fix the bug for lazy_init. Merge pull request !8768 from huangyunlong/2.1lazy
1b6c0a4e4 | BUGFIX | !8767 [Fix]fix the bug for lazy_init. Merge pull request !8767 from huangyunlong/2.2lazy
f9718497c | BUGFIX | !8766 [Fix]fix the bug for lazy_init. Merge pull request !8766 from huangyunlong/2.0lazy
f249faa37 | BUGFIX | !8765 [Fix]fix the bug for lazy_init. Merge pull request !8765 from huangyunlong/1.11lazy
81f163c7c | BUGFIX | !8664 Fix the failed UT. Merge pull request !8664 from 文俊/master
ac0afdf34 | BUGFIX | !8665 Fix the failed UT. Merge pull request !8665 from 文俊/v1.11.0
38d2039e6 | BUGFIX | !8666 Fix the failed UT. Merge pull request !8666 from 文俊/v2.1.0
6829cc2a5 | BUGFIX | !8711 Fix the failed UT. Merge pull request !8711 from 文俊/v2.0.1
c54560874 | BUGFIX | !8610 Fixed the failed test cases. Merge pull request !8610 from yuhaiyan/master-dev1
754e8fa8c | BUGFIX | !8606 Fixed the failed test cases. Merge pull request !8606 from yuhaiyan/v2.1.0-dev4
afd303b42 | BUGFIX | !8609 Fixed the failed test cases. Merge pull request !8609 from yuhaiyan/v2.0.1-dev1
65fa2a351 | BUGFIX | !8607 Fixed the failed test cases. Merge pull request !8607 from yuhaiyan/v1.11.0-dev1
5aafb85f7 | BUGFIX | !8459 [BUGFIX] Fix bugs for GuessFormatWhenContiguous when input tensor is a FakeTensor without desc. Merge pull request !8459 from 陈睿敏/master
54b430a01 | BUGFIX | !8458 [BUGFIX] Fix bugs for GuessFormatWhenContiguous when input tensor is a FakeTensor without desc. Merge pull request !8458 from 陈睿敏/v2.1.0
c047c84cb | BUGFIX | !8680 Bug Fix For Step Trace Time Merge pull request !8680 from 周先琪/master
d71ba0ea1 | BUGFIX | !8679 Bug Fix For Step Trace Time Merge pull request !8679 from 周先琪/v2.1.0
fd1d4c33a | BUGFIX | !8678 Bug Fix For Step Trace Time Merge pull request !8678 from 周先琪/v2.0.1
00a41c50a | BUGFIX | !8677 Bug Fix For Step Trace Time Merge pull request !8677 from 周先琪/v1.11.0
de389f100 | BUGFIX | !8565 Bugfix for syscnt or monotonic in Ascend PyTorch Profiler. Merge pull request !8565 from 裘凯达/pta210_syscnt
5b597d3aa | BUGFIX | !8564 Bugfix for syscnt or monotonic in Ascend PyTorch Profiler. Merge pull request !8564 from 裘凯达/master_syscnt
68f7edf64 | BUGFIX | !8563 Bugfix for syscnt or monotonic in Ascend PyTorch Profiler. Merge pull request !8563 from 裘凯达/pta201_syscnt
f4bac8be5 | BUGFIX | !8562 Bugfix for syscnt or monotonic in Ascend PyTorch Profiler. Merge pull request !8562 from 裘凯达/pta111_syscnt
a3e45942c | BUGFIX | !8612 Fix the bug in wrapper_onnx_ops.py. Merge pull request !8612 from yuhaiyan/v1.11.0-dev2
3bc4e98ab | BUGFIX | !8514 Fix the failed UT. Merge pull request !8514 from 文俊/v1.11.0
912404085 | BUGFIX | !8513 Fix the failed UT. Merge pull request !8513 from 文俊/v2.1.0
9894b57e8 | BUGFIX | !8512 Fix the failed UT. Merge pull request !8512 from 文俊/master
8131e218b | BUGFIX | !8558 [Bugfix]: torch2.2 privateuse1 to privateuseone Merge pull request !8558 from 王广斌/master_copy
bee31a324 | BUGFIX | !8414 ffn infershape bug fix Merge pull request !8414 from 徐柯南/ffn_debug
6f0e61ad6 | BUGFIX | !8415 ffn infershape bug fix Merge pull request !8415 from 徐柯南/ffn_debug_v2
34ec56616 | BUGFIX | !8400 Fix the failed UT. Merge pull request !8400 from 文俊/v1.11.0
a1db8b206 | BUGFIX | !8142 Fix the failed UT. Merge pull request !8142 from 文俊/v2.1.0
f626a51d8 | BUGFIX | !8379 Fix the failed UT. Merge pull request !8379 from 文俊/v2.0.1
d11373633 | BUGFIX | !8378 Fix the failed UT. Merge pull request !8378 from 文俊/master
3d578506b | BUGFIX | !8172 [Fix]fix distributed ut. Merge pull request !8172 from huangyunlong/2.2ut
d5ca7a810 | BUGFIX | !8173 [Fix]fix distributed ut. Merge pull request !8173 from huangyunlong/2.1ut
fdc6327d4 | BUGFIX | !8174 [Fix]fix distributed ut. Merge pull request !8174 from huangyunlong/2.0ut
0d8208d44 | BUGFIX | !8171 [Fix]fix distributed ut. Merge pull request !8171 from huangyunlong/1.11ut
be2cb99c4 | BUGFIX | !8376 [FIX] fix error [context is empty] Merge pull request !8376 from dongwenbo6/v2.1.0-5.0.0
ca1a86059 | BUGFIX | !8373 [FIX] fix error [context is empty] Merge pull request !8373 from dongwenbo6/v2.0.1-5.0.0
7d10ee526 | BUGFIX | !8374 [FIX] fix error [context is empty] Merge pull request !8374 from dongwenbo6/v1.11.0-5.0.0
516d978a1 | BUGFIX | !8372 [FIX] fix error [context is empty] Merge pull request !8372 from dongwenbo6/v2.0.1
5571f271a | BUGFIX | !8370 [FIX] fix error [context is empty] Merge pull request !8370 from dongwenbo6/v2.1.0
7092169a8 | BUGFIX | !8371 [FIX] 修复context is empty报错 Merge pull request !8371 from dongwenbo6/v1.11.0
a7fbb1ec3 | BUGFIX | !8329 Bug Fix For Step Id Merge pull request !8329 from 周先琪/master
4471e772d | BUGFIX | !8322 Bug Fix For Step Id Merge pull request !8322 from 周先琪/v2.1.0
7302a7781 | BUGFIX | !8328 Bug Fix For Step Id Merge pull request !8328 from 周先琪/v2.0.1
9bf033dec | BUGFIX | !8321 Bug Fix For Step Id Merge pull request !8321 from 周先琪/v1.11.0
43ac5c9d6 | BUGFIX | !8327 【bugfix】Catch and process errors occurred during the profiling parsing phase. Merge pull request !8327 from 毛晨/v1.11.0_1212
b94b93c06 | BUGFIX | !8326 【bugfix】Catch and process errors occurred during the profiling parsing phase. Merge pull request !8326 from 毛晨/v2.0.1_1212
a37b47fb7 | BUGFIX | !8325 【bugfix】Catch and process errors occurred during the profiling parsing phase. Merge pull request !8325 from 毛晨/v2.1.0_1212
a3d5c2c7e | BUGFIX | !8324 【bugfix】Catch and process errors occurred during the profiling parsing phase. Merge pull request !8324 from 毛晨/master_1212
f82a64769 | BUGFIX | !8095 [BUGFIX] HCCL_BLOCKING_WAIT fails on master and 2.1 Merge pull request !8095 from jiangpengfei/blockingwait_2.1.0
0ef45ac03 | BUGFIX | !8094 [BUGFIX] HCCL_BLOCKING_WAIT fails on master and 2.1 Merge pull request !8094 from jiangpengfei/blockingwait_master
865a7f0d2 | BUGFIX | !8188 A test case is fixed Merge pull request !8188 from yuhaiyan/v2.1.0-5.0.0-dev2
a537711ac | BUGFIX | !8187 A test case is fixed Merge pull request !8187 from yuhaiyan/v2.1.0-dev2
8f688cdec | BUGFIX | !8183 A test case is fixed Merge pull request !8183 from yuhaiyan/master-dev2
1ad120683 | BUGFIX | !8167 Bug Fix For Memory Match Merge pull request !8167 from 周先琪/master
8e81f0878 | BUGFIX | !8205 Bug Fix For Memory Match Merge pull request !8205 from 周先琪/v2.1.0
6cf6e855a | BUGFIX | !8206 Bug Fix For Memory Match Merge pull request !8206 from 周先琪/v2.0.1
0291c8bbc | BUGFIX | !8207 Bug Fix For Memory Match Merge pull request !8207 from 周先琪/v1.11.0
3139af083 | BUGFIX | !8239 [Fix] Fix bug in testcase. Merge pull request !8239 from 刘嘉巍/v1.11.0-5.0.0
012920eba | BUGFIX | !8238 [Fix] Fix bug in testcase. Merge pull request !8238 from 刘嘉巍/v2.0.1-5.0.0
e451b0a02 | BUGFIX | !8237 [Fix] Fix bug in testcase. Merge pull request !8237 from 刘嘉巍/v2.1.0-5.0.0
70c2df915 | BUGFIX | !8204 【bugfix】Change data type of reserved memory from int to float. Merge pull request !8204 from 毛晨/v1.11.0_1208
46d91c447 | BUGFIX | !8203 【bugfix】Change data type of reserved memory from int to float. Merge pull request !8203 from 毛晨/v2.0.1_1208
917a6c53b | BUGFIX | !8201 【bugfix】Change data type of reserved memory from int to float. Merge pull request !8201 from 毛晨/v2.1.0_1208
b299a2018 | BUGFIX | !8198 【bugfix】Change data type of reserved memory from int to float. Merge pull request !8198 from 毛晨/master_1208
f34342ed0 | BUGFIX | !7886 fixed c1fa6aa from https://gitee.com/zhoubin812/pytorch/pulls/7323 And torch_npu.utils.get_cann_version() Merge pull request !7886 from zhoubin812/cherry-pick-1701182455
e929dfab4 | BUGFIX | !8051 [bugfix] fix asyncronize error between HCCL stream and computation stream Merge pull request !8051 from Leon/210500
b3ca7202a | BUGFIX | !8085 Fix the ut of im2col_backward. Merge pull request !8085 from 戢芳/im2col
824c63d67 | BUGFIX | !8029 Bugfix for calling aclprofDestroyConfig. Merge pull request !8029 from 裘凯达/pta111
b8323d960 | BUGFIX | !8039 Bugfix for calling aclprofDestroyConfig. Merge pull request !8039 from 裘凯达/pta201
0aaaeb0c2 | BUGFIX | !8028 Bugfix for calling aclprofDestroyConfig. Merge pull request !8028 from 裘凯达/pta210
b0cf83e37 | BUGFIX | !8027 Bugfix for calling aclprofDestroyConfig. Merge pull request !8027 from 裘凯达/master
5e32ac1bd | BUGFIX | !7604 Fix the failed UT. Merge pull request !7604 from 文俊/v1.11.0_copy
d35164a2c | BUGFIX | !7745 Fix the failed UT. Merge pull request !7745 from 文俊/master
5af6ad745 | BUGFIX | !7743 Fix the failed UT. Merge pull request !7743 from 文俊/v2.0.1
13461ae3b | BUGFIX | !7744 Fix the failed UT. Merge pull request !7744 from 文俊/v2.1.0
ab65d02a6 | BUGFIX | !7772 [fix] fix subthread destroy event fail bug Merge pull request !7772 from xudaohong/cherry-pick-1700883258
59e29f4e4 | BUGFIX | !7860 [fix] fix subthread destroy event fail bug Merge pull request !7860 from xudaohong/cherry-pick-1701156956
c5ba2a7a4 | BUGFIX | !7861 [fix] fix subthread destroy event fail bug Merge pull request !7861 from xudaohong/cherry-pick-1701156998
e9a6d2bc8 | BUGFIX | !8009 RMSNorm meta函数修复 v2.1 Merge pull request !8009 from 张超凡/rms_norm_meta_r21_1202
bb2da3e9a | BUGFIX | !7931 RMSNorm meta函数修复 Merge pull request !7931 from 张超凡/master
4f18e3d11 | BUGFIX | !7870 【Bugfix】Solve the problem that the profiling is stuck in specific scenarios. Merge pull request !7870 from 毛晨/v1.11.0_1128
65b599d53 | BUGFIX | !7871 【Bugfix】Solve the problem that the profiling is stuck in specific scenarios. Merge pull request !7871 from 毛晨/v2.0.1_1128
68c1fdc21 | BUGFIX | !7869 【Bugfix】Solve the problem that the profiling is stuck in specific scenarios. Merge pull request !7869 from 毛晨/v2.1.0_1128
967a0650e | BUGFIX | !7868 【Bugfix】Solve the problem that the profiling is stuck in specific scenarios. Merge pull request !7868 from 毛晨/master_1128
416c9e2dd | BUGFIX | !7850 Fix the failed UT. Merge pull request !7850 from 文俊/v1.11.0-5.0.0
1c41b7bd3 | BUGFIX | !7852 Fix the failed UT. Merge pull request !7852 from 文俊/v2.1.0-5.0.0
44a5278a7 | BUGFIX | !7853 Fix the failed UT. Merge pull request !7853 from 文俊/v2.0.1-5.0.0
1eec95714 | BUGFIX | !7766 [fix] fix subthread destroy event fail bug Merge pull request !7766 from xudaohong/v1.11.0
28020069b | BUGFIX | !7764 [fix] fix subthread destroy event fail bug Merge pull request !7764 from xudaohong/cherry-pick-1700875585
9dc04638c | BUGFIX | !7763 [fix] fix subthread destroy event fail bug Merge pull request !7763 from xudaohong/cherry-pick-1700875414
a0aec26fe | BUGFIX | !7723 bugfix, modify aclnn route logic Merge pull request !7723 from 邵非凡/base_format20-5.0.0
9741809eb | BUGFIX | !7722 bugfix, modify aclnn route logic Merge pull request !7722 from 邵非凡/base_format21-5.0.0
cdf443718 | BUGFIX | !7657 [bugfix] fix the bug of schedule. Merge pull request !7657 from MooYeh/v1.11.0
156c517b2 | BUGFIX | !7751 [fix] fix subthread destroy event fail bug Merge pull request !7751 from xudaohong/master
66df92d75 | BUGFIX | !7615 [bugfix] fix the bug of schedule. Merge pull request !7615 from MooYeh/master
2c2f0bc09 | BUGFIX | !7677 [bugfix] fix the bug of schedule. Merge pull request !7677 from MooYeh/v2.1.0
82ded5534 | BUGFIX | !7676 [bugfix] fix the bug of schedule. Merge pull request !7676 from MooYeh/v2.0.1
86d1674ce | BUGFIX | !7693 [bugfix] fix asyncronize error between HCCL stream and computation stream Merge pull request !7693 from Leon/master
315f762dd | BUGFIX | !7704 [bugfix] fix asyncronize error between HCCL stream and computation stream Merge pull request !7704 from Leon/210_stream
3c3dc2dd3 | BUGFIX | !7721 bugfix, modify aclnn route logic Merge pull request !7721 from 邵非凡/base_format20
c8f84d7c5 | BUGFIX | !7720 bugfix, modify aclnn route logic Merge pull request !7720 from 邵非凡/base_format21
f684df52c | BUGFIX | !7719 bugfix, modify aclnn route logic Merge pull request !7719 from 邵非凡/base_format
0681cc7e7 | BUGFIX | !7678 Bugfix for Ascend PyTorch Profiler Merge pull request !7678 from 裘凯达/pta_master
a5b9c007e | BUGFIX | !7679 Bugfix for Ascend PyTorch Profiler Merge pull request !7679 from 裘凯达/pta210
628574837 | BUGFIX | !7680 Bugfix for Ascend PyTorch Profiler Merge pull request !7680 from 裘凯达/pta201
707f2dee8 | BUGFIX | !7681 Bugfix for Ascend PyTorch Profiler Merge pull request !7681 from 裘凯达/pta111
c6c30fc37 | BUGFIX | !7541 Bugfix for losing Profiling data occasionally. Merge pull request !7541 from 裘凯达/pta111_5.0.0
efbbcb9f8 | BUGFIX | !7540 Bugfix for losing Profiling data occasionally. Merge pull request !7540 from 裘凯达/pta201_5.0.0
e27f8d27c | BUGFIX | !7539 Bugfix for losing Profiling data occasionally. Merge pull request !7539 from 裘凯达/pta210_5.0.0
f4e1f8ba1 | BUGFIX | !7344 Fix the failed UT. Merge pull request !7344 from 文俊/v1.11.0
7eda0337d | BUGFIX | !7642 [bugfix] fix the bug introduced while solving conflicts Merge pull request !7642 from MooYeh/master
c494718ca | BUGFIX | !7643 [bugfix] fix the bug introduced while solving conflicts Merge pull request !7643 from MooYeh/v2.1.0
38f5aee34 | BUGFIX | !7644 [bugfix] fix the bug introduced while solving conflicts Merge pull request !7644 from MooYeh/v2.0.1
4631210fc | BUGFIX | !7641  [bugfix] fix the bug introduced while solving conflicts Merge pull request !7641 from MooYeh/v1.11.0
eb5a5c9d4 | BUGFIX | !7637 Bug Fix For Analyse Performance Merge pull request !7637 from 周先琪/v2.0.1
87a038eea | BUGFIX | !7636 Bug Fix For Analyse Performance Merge pull request !7636 from 周先琪/v1.11.0
3b345a25a | BUGFIX | !7610 【bugfix】fix the way of adding synchronize when init ddp Merge pull request !7610 from 郭光浩/v2.1.0-5.0.0
e7203c28f | BUGFIX | !7609 【bugfix】fix the way of adding synchronize when init ddp Merge pull request !7609 from 郭光浩/v2.1.0
00a50a5b0 | BUGFIX | !7608 【bugfix】fix the way of adding synchronize when init ddp Merge pull request !7608 from 郭光浩/master
86c2fc7c1 | BUGFIX | !7290 save_async bug fix Merge pull request !7290 from sunjiayang/asave_111_500
56187d68f | BUGFIX | !7294 save_async bug fix Merge pull request !7294 from sunjiayang/asave_210_500
2eee1249a | BUGFIX | !7292 save_async bug fix Merge pull request !7292 from sunjiayang/asave_201_500
29cafe17e | BUGFIX | !7245 async save bug fix Merge pull request !7245 from sunjiayang/asave_111
b40f11451 | BUGFIX | !7291 save_async bug fix Merge pull request !7291 from sunjiayang/asave_201
120c25207 | BUGFIX | !7293 save_async bug fix Merge pull request !7293 from sunjiayang/asave_210
3ccfa41d2 | BUGFIX | !7477 [bug fix] Update the memory record matching logic Merge pull request !7477 from MooYeh/v1.11.0-5.0.0
245c35d24 | BUGFIX | !7478 [bug fix] Update the memory record matching logic Merge pull request !7478 from MooYeh/v2.0.1-5.0.0
7f2048ba2 | BUGFIX | !7479 [bug fix] Update the memory record matching logic Merge pull request !7479 from MooYeh/v2.1.0-5.0.0
7446412a3 | BUGFIX | !7464 [Bugfix] Remove field_tag to all_data in codegen Merge pull request !7464 from 王广斌/field_tag
4e09e4d49 | BUGFIX | !7420 Fix the failed UT. Merge pull request !7420 from 文俊/v2.0.1
c60422445 | BUGFIX | !7419 Fix the failed UT. Merge pull request !7419 from 文俊/v2.1.0
87aa96204 | BUGFIX | !7418 Fix the failed UT. Merge pull request !7418 from 文俊/master
e5b5acd05 | BUGFIX | !7379 Memory reocrd match logic bug fix. Merge pull request !7379 from MooYeh/master
4763c797b | BUGFIX | !7382 Memory reocrd match logic bug fix. Merge pull request !7382 from MooYeh/v2.1.0
a633b846a | BUGFIX | !7381 Memory reocrd match logic bug fix. Merge pull request !7381 from MooYeh/v2.0.1
fe95e020b | BUGFIX | !7380 Memory reocrd match logic bug fix. Merge pull request !7380 from MooYeh/v1.11.0
757fd9ca2 | BUGFIX | !7329 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7329 from Leon/master
77a18bc9b | BUGFIX | !7331 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7331 from Leon/210500
aa5e748ae | BUGFIX | !7330 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7330 from Leon/21
294b36865 | BUGFIX | !7333 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7333 from Leon/201500
8209e18ef | BUGFIX | !7332 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7332 from Leon/20
66f36ccc1 | BUGFIX | !7335 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7335 from Leon/111500
8f6d51de2 | BUGFIX | !7334 [bugfix] fix grad shape error when gradient_as_view_in is turned on. Merge pull request !7334 from Leon/111
435387dfc | BUGFIX | !7257 Bugfix: tuple object has no attribute 'get' Merge pull request !7257 from yuhaiyan/v2.1.0-5.0.0-dev2
375309721 | BUGFIX | !7258 Bugfix: tuple object has no attribute 'get' Merge pull request !7258 from yuhaiyan/v2.1.0-dev3
1833890ba | BUGFIX | !7249 Bugfix: tuple object has no attribute 'get' Merge pull request !7249 from yuhaiyan/master-dev3
dd72de532 | BUGFIX | !7239 [bugfix] delete  redundant code Merge pull request !7239 from 郭光浩/v2.1.0
a56ebc25a | BUGFIX | !7088 [fix] fix problems introduced by cleancode. Merge pull request !7088 from 杜金航/v1.11.0_cleancode
e142f9e30 | BUGFIX | !7092 [fix] fix problems introduced by cleancode. Merge pull request !7092 from 杜金航/v1.11.0-5.0.0
827844276 | BUGFIX | !7089 [fix] fix problems introduced by cleancode. Merge pull request !7089 from 杜金航/2.0_cleancode
02517559e | BUGFIX | !7093 [fix] fix problems introduced by cleancode. Merge pull request !7093 from 杜金航/v2.0.1-5.0.0
7cc6a55b9 | BUGFIX | !7091 [fix] fix problems introduced by cleancode. Merge pull request !7091 from 杜金航/v2.1_cleancode
cd40b4380 | BUGFIX | !7094 [fix] fix problems introduced by cleancode. Merge pull request !7094 from 杜金航/v2.1.0-5.0.0
eb2d2e5b8 | BUGFIX | !7090 [fix] fix problems introduced by cleancode. Merge pull request !7090 from 杜金航/master_cleancode
ee398aebd | BUGFIX | !7174 [bugfix] Supplemented with init parameters Merge pull request !7174 from 郭光浩/v2.1.0-5.0.0
564847175 | BUGFIX | !7176 [bugfix] add synchronize when init ddp Merge pull request !7176 from 郭光浩/v1.11.0-5.0.0
54caac2a4 | BUGFIX | !7175 [bugfix] add synchronize when init ddp Merge pull request !7175 from 郭光浩/v1.11.0
ee137a872 | BUGFIX | !7178 [bugfix] add synchronize when init ddp Merge pull request !7178 from 郭光浩/v2.0.1-5.0.0
519a8aad9 | BUGFIX | !7177 [bugfix] add synchronize when init ddp Merge pull request !7177 from 郭光浩/v2.0.1
434dc97e5 | BUGFIX | !7172 [bugfix] add synchronize when init ddp Merge pull request !7172 from 郭光浩/master
a695055da | BUGFIX | !6859 Fix the failed UT. Merge pull request !6859 from 文俊/utv1.11.0
b2f2eaf05 | BUGFIX | !6903 Fix the failed UT. Merge pull request !6903 from 文俊/v2.0.1
cb637546f | BUGFIX | !6935 Fix the failed UT. Merge pull request !6935 from 文俊/v2.1.0
d7853ec8e | BUGFIX | !6900 Fix the failed UT. Merge pull request !6900 from 文俊/master
1f0e7d3a4 | BUGFIX | !7173 [bugfix] Supplemented with init parameters Merge pull request !7173 from 郭光浩/v2.1.0
755b997d9 | BUGFIX | !7147 [bugfix] add synchronize when init ddp Merge pull request !7147 from 郭光浩/v2.1.0-5.0.0
3b4d6d24a | BUGFIX | !7144 [bugfix] add synchronize when init ddp Merge pull request !7144 from 郭光浩/v2.1.0
a0f5b0d92 | BUGFIX | !6909 Fixed security issues, log includs sensitive information. Merge pull request !6909 from 徐季伟/master
6afa92ac9 | BUGFIX | !7000 Fixed security issues, log includes sensitive information. Merge pull request !7000 from shaoyf/sc/master_rpc_log
d9eb084d6 | BUGFIX | !6999 Fixed security issues, log includes sensitive information. Merge pull request !6999 from shaoyf/sc/210_rpc_log
99454afd6 | BUGFIX | !6823 Fix the failed UT. Merge pull request !6823 from 戢芳/ut_2.1
8dd2c72d0 | BUGFIX | !6822 Fix the failed UT. Merge pull request !6822 from 戢芳/ut_2.0.1
f7836e781 | BUGFIX | !6821 Fix the failed UT. Merge pull request !6821 from 戢芳/ut_1.11
5cb3f74c9 | BUGFIX | !6825 Fix the failed UT. Merge pull request !6825 from 戢芳/ut_master
5b4062fa8 | BUGFIX | !6886 【bugfix】bugfix for the communication operator enters the TaskQueue Merge pull request !6886 from 郭光浩/v2.0.1
5936c41b0 | BUGFIX | !6885 【bugfix】bugfix for the communication operator enters the TaskQueue Merge pull request !6885 from 郭光浩/v1.11.0
117c6a8fe | BUGFIX | !6884 [BUgFix]bugfix for the communication operator enters the TaskQueue Merge pull request !6884 from 郭光浩/master
089fd3278 | BUGFIX | !6828 [fix] Fixed performance degradation introduced by the for loop. Merge pull request !6828 from 杜金航/master_cleancode
9b3c10db6 | BUGFIX | !6827 [fix] Fixed performance degradation introduced by the for loop. Merge pull request !6827 from 杜金航/2.0_cleancode
c460a49e7 | BUGFIX | !6824 [fix] Fixed performance degradation introduced by the for loop. Merge pull request !6824 from 杜金航/v1.11.0_cleancode
f69dab77b | BUGFIX | !6790 [Bugfix] TensorOptions init problem Merge pull request !6790 from 王广斌/clean_code
aa3d3f84d | BUGFIX | !6791 [Bugfix] TensorOptions init problem Merge pull request !6791 from 王广斌/clean_code2.0
b5456be4b | BUGFIX | !6789 [Bugfix] TensorOptions init problem Merge pull request !6789 from 王广斌/clean_code1.11
139de6ae0 | BUGFIX | !6743 [bugfix] Fix the bug that the empty_like operator dytpe parameter is invalid Merge pull request !6743 from shaoyf/fix/master_empty_like
b69b09529 | BUGFIX | !6744 [bugfix] Fix the bug that the empty_like operator dytpe parameter is invalid Merge pull request !6744 from shaoyf/fix/201_empty_like
9ad3c7a6a | BUGFIX | !6680 [bugfix]Fix compilation cache invalidation issue Merge pull request !6680 from 万俊伟/cherry-pick-1697685653
f612d08ca | BUGFIX | !6670 [bugfix]If the environment variable has set the op compile cache path, use; If not set, the default path will not be processed, and it will be handled by Cann. Merge pull request !6670 from 万俊伟/cache_v1.8
0d8d24129 | BUGFIX | !6671 [bugfix]If the environment variable has set the op compile cache path, use; If not set, the default path will not be processed, and it will be handled by Cann. Merge pull request !6671 from 万俊伟/cache_v1.11
71e490847 | BUGFIX | !6669 [bugfix]If the environment variable has set the op compile cache path, use; If not set, the default path will not be processed, and it will be handled by Cann. Merge pull request !6669 from 万俊伟/cache_v2.0.1
85e7f6bdc | BUGFIX | !6667 [bugfix]If the environment variable has set the op compile cache path, use; If not set, the default path will not be processed, and it will be handled by Cann. Merge pull request !6667 from 万俊伟/compile_path_master
0510bed37 | BUGFIX | !6578 [Bugfix] Tuple input bugfix. Merge pull request !6578 from will-devil/codecheckbug
be640bed2 | BUGFIX | !6568 [Clean Code] Fix codecheck problem. Merge pull request !6568 from 刘嘉巍/master
32383d4e6 | BUGFIX | !6567 [Clean Code] Fix codecheck problem. Merge pull request !6567 from 刘嘉巍/v2.0.1
59099ac63 | BUGFIX | !6566 [Clean Code] Fix codecheck problem. Merge pull request !6566 from 刘嘉巍/1.11-codegen
b49bf5d54 | BUGFIX | !6514 Bugfix for dirname. Merge pull request !6514 from 裘凯达/master_bugfix
6a371ab96 | BUGFIX | !6515 Bugfix for dirname. Merge pull request !6515 from 裘凯达/pta201_bugfix
d0e70b6a2 | BUGFIX | !6516 Bugfix for dirname. Merge pull request !6516 from 裘凯达/pta111_bugfix
505d216d8 | BUGFIX | !6517 Bugfix for dirname. Merge pull request !6517 from 裘凯达/pta181_bugfix
7f6cab22e | BUGFIX | !6486 [Fix] fix aclrt call by saving and recovering context. Merge pull request !6486 from 闫鹏全/v1.11.0-5.0.rc3
9a3a91e7b | BUGFIX | !6432 [Fix] fix fsdp by saving and recovering context. Merge pull request !6432 from 闫鹏全/v2.1.0-5.0.rc3
bbcad4677 | BUGFIX | Fix CodeCheck problems
38334d68e | BUGFIX | !6427 [Fix] fix aclrt call by saving and recovering context. Merge pull request !6427 from 闫鹏全/v1.11.0
756d24e47 | BUGFIX | !6464 【cleancode】 Fix some white box issues. Merge pull request !6464 from 毛晨/v1.11.0_1008
3623c676e | BUGFIX | !6467 【cleancode】 Fix some white box issues. Merge pull request !6467 from 毛晨/v2.0.1_1008
49b41f30e | BUGFIX | !6370 【cleancode】 Fix some white box issues. Merge pull request !6370 from 毛晨/master_1008
4f8deaa91 | BUGFIX | !6419 [bugfix]Use work wait instead of future wait in FSDP Merge pull request !6419 from lj_new/v2.1.0-5.0.rc3
265f0a1b1 | BUGFIX | !6400 [Fix] fix fsdp by saving and recovering context. Merge pull request !6400 from 闫鹏全/fsdp_test
33d545848 | BUGFIX | !6287 Add format_cast test case Merge pull request !6287 from 王广斌/bugfix_format_cast_200
4d7a11b0d | BUGFIX | !6389 [bugfix]Use work wait instead of future wait in FSDP Merge pull request !6389 from lj_new/master
5768b78f6 | BUGFIX | !6284 Add format_cast test case Merge pull request !6284 from 王广斌/bugfix_format_cast
5a11e7fc6 | BUGFIX | !6286 Add format_cast test case Merge pull request !6286 from 王广斌/bugfix_format_cast_111
595d7faf3 | BUGFIX | !6379 [Bugfix] 0D Tensor has no index. Merge pull request !6379 from will-devil/v1.11.0-5.0.rc3
d670b5c24 | BUGFIX | !6376 [PT2.1] fix bugs while deepcopy cpu tensors Merge pull request !6376 from 陈睿敏/v2.1.0-5.0.rc3
7655a9344 | BUGFIX | !6375 [PT2.1] fix bugs while deepcopy cpu tensors Merge pull request !6375 from 陈睿敏/master
a5ec7386b | BUGFIX | !6349 [Bugfix] 0D Tensor has no index. Merge pull request !6349 from will-devil/211d
670383b8f | BUGFIX | !6365 [fix] fix storage deepcopy patch Merge pull request !6365 from 王姜奔/v2.1.0-5.0.rc3
d3b84d12c | BUGFIX | !6330 [fix] fix storage deepcopy patch Merge pull request !6330 from 王姜奔/master
3e37d8a44 | BUGFIX | !6261 bugfix: Set op execute timeout to max 547s Merge pull request !6261 from yangxiaorun/cherry-pick-1695863299
aa7baf269 | BUGFIX | !6259 bugfix: Set op execute timeout to max 547s Merge pull request !6259 from yangxiaorun/cherry-pick-1695862672
54ca3a38b | BUGFIX | !6260 bugfix: Set op execute timeout to max 547s Merge pull request !6260 from yangxiaorun/cherry-pick-1695862943
f41db8596 | BUGFIX | !6291 bugfix:delete future_work in ASSERT Merge pull request !6291 from 邵非凡/reducer
45ce7466f | BUGFIX | !6177 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6177 from 周先琪/v2.1.0-5.0.rc3
d76d8ceef | BUGFIX | !6176 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6176 from 周先琪/v2.0.1-5.0.rc3
389554c00 | BUGFIX | !6175 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6175 from 周先琪/v1.11.0-5.0.rc3
9de3a537d | BUGFIX | fixed d024cbc from https://gitee.com/yangxiaorun1/pytorch/pulls/6071 fixed 6f840e0 from https://gitee.com/yangxiaorun1/pytorch/pulls/6070 Change INFNAN default mode
450ccdf8c | BUGFIX | !6149 [Bugfix]Verify acl_format before npu_format_cast_impl Merge pull request !6149 from 王广斌/bugfix_format_cast_rc3
4affb9520 | BUGFIX | !6146 [Bugfix]Verify acl_format before npu_format_cast_impl Merge pull request !6146 from 王广斌/bugfix_format_cast_200_rc3
cd8228c5e | BUGFIX | !6143 [Bugfix]Verify acl_format before npu_format_cast_impl Merge pull request !6143 from 王广斌/bugfix_npu_format_cast_111rc
c5af08508 | BUGFIX | !6160 bugfix: Set op execute timeout to max 547 Merge pull request !6160 from yangxiaorun/timeout
07fb8415e | BUGFIX | !6201 bugfix: Set op execute to max 547 Merge pull request !6201 from yangxiaorun/cherry-pick-1695731634
72bc54541 | BUGFIX | !6165 bugfix: Set op execute to max 547 Merge pull request !6165 from yangxiaorun/1.11_timeout
6b1946bc5 | BUGFIX | !6174 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6174 from jiangpengfei/v1.11.0-5.0.rc3
151586af0 | BUGFIX | !6181 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6181 from jiangpengfei/v2.0.1-5.0.rc3
af6f7bf66 | BUGFIX | !6182 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6182 from jiangpengfei/v2.1.0-5.0.rc3
93abbed22 | BUGFIX | !6144 [Bugfix]Verify acl_format before npu_format_cast_impl Merge pull request !6144 from 王广斌/bugfix_format_cast_111
18079cc19 | BUGFIX | !6147 [Bugfix]Verify acl_format before npu_format_cast_impl Merge pull request !6147 from 王广斌/bugfix_format_cast_200
c05568d61 | BUGFIX | !6142 [Bugfix] Verify acl_format before npu_format_cast_impl Merge pull request !6142 from 王广斌/bugfix_format_cast
735a6e135 | BUGFIX | !6163 [hostapi] fix NanToNum Merge pull request !6163 from zhongliang/v1.11.0
ec302d643 | BUGFIX | !6138 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6138 from 周先琪/v1.8.1
26bd37e9b | BUGFIX | !6136 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6136 from 周先琪/v1.11.0
e219e3ade | BUGFIX | !6130 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6130 from 周先琪/v2.0.1
3dc5331dc | BUGFIX | !6132 Bug Fix For Trace View Not Include Msprof Slice Merge pull request !6132 from 周先琪/master
f6a6bf5f0 | BUGFIX | bug_fix
b67cb0137 | BUGFIX | bug_fix
af55d0ba1 | BUGFIX | !6091 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6091 from jiangpengfei/master
28750aff5 | BUGFIX | !6089 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6089 from jiangpengfei/v2.0.1
69fd3ad81 | BUGFIX | !6085 [BugFix] Solve the problem that std::call_once stuck at the second time the function be called if the function throw an exception at first time be called Merge pull request !6085 from jiangpengfei/v1.11.0
09379ffa8 | BUGFIX | !6038 [Bugfix] Do not cast the weight of LazyConv3d. Merge pull request !6038 from will-devil/lazy111
304a4c5ae | BUGFIX | !6065 [Bugfix] Do not cast the weight of LazyConv3d. Merge pull request !6065 from will-devil/lazy201
97c0f422a | BUGFIX | !6037 [Bugfix] Do not cast the weight of LazyConv3d. Merge pull request !6037 from will-devil/lazymaster
7a0978a11 | BUGFIX | !6042 bugfix for custom op Merge pull request !6042 from 王强/master
188009fe7 | BUGFIX | bugfix: torch.compile affected by layernormal patch
05a355355 | BUGFIX | [fix] Rollback syncbatchnorm
91895483d | REVERT | Revert export_stacks from Profiling.
8d06036cb | REVERT | Revert export_stacks from Profiling.
77d27b6fb | REVERT | Revert export_stacks from Profiling.
c237c2d60 | REVERT | Revert export_stacks from Profiling.
875a3f38d | BUGFIX | fix dpse defined case
2c8c976cb | BUGFIX | msprof_adapt_bug_fix
ded03ec63 | BUGFIX | msprof_adapt_bug_fix
e859b828a | BUGFIX | msprof_adapt_bug_fix
0a27da6da | BUGFIX | msprof_adapt_bug_fix
662d25462 | REVERT | revert event
629524ea9 | BUGFIX | Fix bug in is_npu when device is none
a81d895fd | BUGFIX | [hostapi]fix argmin
7845c8333 | BUGFIX | fixed 2b156de from https://gitee.com/guan-longfeng/pytorch/pulls/5890 hostapi support deterministic
574616fc4 | BUGFIX | fixed 2b156de from https://gitee.com/guan-longfeng/pytorch/pulls/5890 hostapi support deterministic
159eee9a0 | BUGFIX | [fix] Rollback syncbatchnorm
9fd849b76 | BUGFIX | [Bugfix]patch torch.nn.DataParallel.__init__
576b792ad | BUGFIX | [fix] codecheck
7579f927c | BUGFIX | Bugfix return opplug_in impl name
f22ac0072 | BUGFIX | bugfix -- communication matrix step info
1057611de | BUGFIX | bugfix -- communication matrix step info
66ad5803d | BUGFIX | bugfix -- communication matrix step info
56ced99b6 | BUGFIX | bugfix -- communication matrix step info
1b9e838a9 | BUGFIX | fixed c95b466 from https://gitee.com/xudaohong/pytorch/pulls/5884 [feat] cherry-pick from v1.11.0 add env var get interface for AllowFP32ToFP16 AllowConvHF32 AllowMatmulHF32
ef4f8d765 | BUGFIX | fix complex_out
d9ac8f5a4 | BUGFIX | fixed 3af789e from https://gitee.com/xudaohong/pytorch/pulls/5810 [feat] add torch.npu.Stream.query interface
f09654ee6 | BUGFIX | [Fix] enalbe torchair compile default.
b84d01055 | BUGFIX | Fix npu_format_cast namepace and delete manual autograd
73b69ede7 | BUGFIX | Fix building error without git installed
ee15111a9 | BUGFIX | Fix warning in parse_backend_yaml
b6fb80926 | BUGFIX | Fix building error without git installed
1fced115f | BUGFIX | Fix building error without git installed
df1f3dac3 | BUGFIX | Fix building error without git installed
edf4ce651 | BUGFIX | Fix building error without git installed
d507f1415 | BUGFIX | Fix dropout's bug when shape is too big
0672849f6 | BUGFIX | Fix dropout's bug when shape is too big.
9d97c545b | BUGFIX | Fix dropout's bug when shape is too big.
864c8c78b | BUGFIX | fix:Syntax error in function supported_profiler_level and supported_ai_core_metrics.
8c027528f | BUGFIX | fix warning in parse_backend_yaml
98734303a | BUGFIX | When the Foreach operator is invoked, the foreach operator is split according to a fixed size, and then the operator is invoked several times.
96c09b999 | REVERT | Revert "!3352 Add manylinux support for pypi"
3d419ddbe | BUGFIX | Fix torch.load with different income parameters
07292ff3b | BUGFIX | fixed 2e55ccf from https://gitee.com/xudaohong/pytorch/pulls/5562 [fix] revert dropout genmask memory reuse
f4dcee9e7 | BUGFIX | fix grad out fp32
e2684eeec | BUGFIX | [fix]set_stream repair
c5fd45f94 | BUGFIX | 修复faketensor coredump问题
09ae87c2c | BUGFIX | fix trace
3937a2f89 | BUGFIX | fix cant parse bubble type problem
4f45fbcab | BUGFIX | fix cant parse bubble type problem
24e741a90 | BUGFIX | fix cant parse bubble type problem
f21e8c1ff | BUGFIX | fix cant parse bubble type problem
71d0eafa9 | BUGFIX | fix 2.0 tensorlist
2e4d9b2f7 | BUGFIX | fixed 2286ed0 from https://gitee.com/Liulufang_gitee/pytorch/pulls/5600 fix auto backward
7030efd0e | BUGFIX | fix 1.11 tensorlist
2286ed026 | BUGFIX | fix auto backward
bcc6459e4 | BUGFIX | fixed b5143fb from https://gitee.com/wan-junwei4/pytorch/pulls/5593 fixed de87b3a from https://gitee.com/wan-junwei4/pytorch/pulls/5592 fixed 0252fba from https://gitee.com/wan-junwei4/pytorch/pulls/5591 fixed 7c08927 from https://gitee.com/wan-junwei4/pytorch/pulls/5578 [feat]Provide ACL_OP_COMPILER_CACHE_MODE and ACL_OP_COMPILER_CACHE_DIR  environment variables to enable cache.
b5143fb3d | BUGFIX | fixed de87b3a from https://gitee.com/wan-junwei4/pytorch/pulls/5592 fixed 0252fba from https://gitee.com/wan-junwei4/pytorch/pulls/5591 fixed 7c08927 from https://gitee.com/wan-junwei4/pytorch/pulls/5578 [feat]Provide ACL_OP_COMPILER_CACHE_MODE and ACL_OP_COMPILER_CACHE_DIR  environment variables to enable cache.
de87b3afa | BUGFIX | fixed 0252fba from https://gitee.com/wan-junwei4/pytorch/pulls/5591 fixed 7c08927 from https://gitee.com/wan-junwei4/pytorch/pulls/5578 [feat]Provide ACL_OP_COMPILER_CACHE_MODE and ACL_OP_COMPILER_CACHE_DIR  environment variables to enable cache.
0252fbaa4 | BUGFIX | fixed 7c08927 from https://gitee.com/wan-junwei4/pytorch/pulls/5578 [feat]Provide ACL_OP_COMPILER_CACHE_MODE and ACL_OP_COMPILER_CACHE_DIR  environment variables to enable cache.
44552f800 | BUGFIX | Fix how torch_npu.optim is used
fecd826bb | BUGFIX | Fix result dtype and supprt bool dtype.
7e738aacf | BUGFIX | Fix mul to support scalar input.
9466cda8b | BUGFIX | Fix the issue of error: torch_npu has no attribute distributed
fdfbded60 | BUGFIX | fixed 7b8da44 from https://gitee.com/xudaohong/pytorch/pulls/5485 [fix] add block check in eraseStream
4bf75e9ec | BUGFIX | fixed 7b8da44 from https://gitee.com/xudaohong/pytorch/pulls/5485 [fix] add block check in eraseStream
a6228de62 | BUGFIX | fixed 7b8da44 from https://gitee.com/xudaohong/pytorch/pulls/5485 [fix] add block check in eraseStream
cf2a68d3a | BUGFIX | fixed ada0910 from https://gitee.com/Liulufang_gitee/pytorch/pulls/5526 remove scale for fusion kernel
331df2fa5 | BUGFIX | fix dpse pta
ee49cd216 | BUGFIX | fixed 6d9f72b from https://gitee.com/guan-longfeng/pytorch/pulls/5520 fixed 8922771 from https://gitee.com/guan-longfeng/pytorch/pulls/5519 fixed cd361fd from https://gitee.com/guan-longfeng/pytorch/pulls/5458 set deterministic in aclctxSysParam
6d9f72bfe | BUGFIX | fixed 8922771 from https://gitee.com/guan-longfeng/pytorch/pulls/5519 fixed cd361fd from https://gitee.com/guan-longfeng/pytorch/pulls/5458 set deterministic in aclctxSysParam
892277165 | BUGFIX | fixed cd361fd from https://gitee.com/guan-longfeng/pytorch/pulls/5458 set deterministic in aclctxSysParam
91ec21aa6 | BUGFIX | Fix duplicate log printing in logging.
7b8da4418 | BUGFIX | [fix] add block check in eraseStream
177d8a313 | BUGFIX | fix DEVICE_NAME and warning
bb771ba3d | BUGFIX | fix DEVICE_NAME and warning
8311f6cc8 | BUGFIX | fixed 2a6cf54 from https://gitee.com/xudaohong/pytorch/pulls/5448 [fix] fix eraseStream iterator code err
df5eaa17c | BUGFIX | fixed 2a6cf54 from https://gitee.com/xudaohong/pytorch/pulls/5448 [fix] fix eraseStream iterator code err
7a71bc820 | BUGFIX | fixed 2a6cf54 from https://gitee.com/xudaohong/pytorch/pulls/5448 [fix] fix eraseStream iterator code err
e5e40187b | BUGFIX | fix err
fc0e22faa | BUGFIX | Fix duplicate log printing in logging.
2a6cf5491 | BUGFIX | [fix] fix eraseStream iterator code err
5101b63f2 | REVERT | Revert 'Pull Request !5397 : Parse trave_view to Time statistics CSV'
790baacc8 | REVERT | Revert 'Pull Request !5400 : Parse trave_view to Time statistics CSV'
ee20c8918 | REVERT | Revert 'Pull Request !5398 : Parse trave_view to Time statistics CSV'
42373ef2d | REVERT | Revert 'Pull Request !5321 : trace_step_time'
46e0efa01 | BUGFIX | [Fix]Update env variable inteface
d1c83bf17 | BUGFIX | [fix] modify aclnn route logic
67619ce81 | BUGFIX | [fix] modify make_tensor complie err
bd74f3f47 | BUGFIX | [HOSTAPI]fix to.dtype low performance
5fd5bc92b | BUGFIX | fixed ec4dc0f from https://gitee.com/xudaohong/pytorch/pulls/5207 [fix] fix tensor.data memory not free
cefe1daef | BUGFIX | [bugfix]value of ACL_OP_DEBUG_LEVEL should be in [0,1,2,3,4]
23672f97c | BUGFIX | [bugfix]value of ACL_OP_DEBUG_LEVEL should be in ["0", "1", "2", "3", "4"]
2970efb15 | BUGFIX | fixed 39677e2 from https://gitee.com/wan-junwei4/pytorch/pulls/5356 fixed 45dd7ae from https://gitee.com/wan-junwei4/pytorch/pulls/5346 [bugfix]value of ACL_OP_DEBUG_LEVEL should be in ["0", "1", "2", "3", "4"]
7a0033114 | BUGFIX | fixed 45dd7ae from https://gitee.com/wan-junwei4/pytorch/pulls/5346 [bugfix]value of ACL_OP_DEBUG_LEVEL should be in ["0", "1", "2", "3", "4"]
9fd0b9267 | BUGFIX | Fix result tensor shape of avg_pool_3d.
23ace4d55 | BUGFIX | Fix the output tensor shape of avg_pool_3d.
2509fb638 | BUGFIX | [fix] fix tensor.data memory not free
06eeee147 | BUGFIX | fix bug for v2.0.1 checkpoint
66b1e3195 | BUGFIX | [HOSTAPI]fix cast_back can not automatically route
1eef5ca0d | BUGFIX | bugfix for device_id in master.
d0c7954c8 | BUGFIX | Fix the dtype of true_divide and support scalar input.
20198dff6 | BUGFIX | Fix the dtype of true_divide and support scalar input.
018e98545 | BUGFIX | Fix matmul backward and reopen its config
953497739 | BUGFIX | Fix indexput hostapi and update upsamplebilinear2d name.
206f2cc9e | BUGFIX | fix prelu ut
0337402f8 | BUGFIX | fix prelu ut
6f7642cf2 | BUGFIX | fix ones_like
49148ae69 | BUGFIX | [Fix] supprot torch._C._nn._parse_to.
42bbed92f | BUGFIX | [Fix] supprot torch._C._nn._parse_to.
f806f6daf | BUGFIX | [fix] fix tensor.data memory not free
9be98a43b | REVERT | Revert "!5072 support flash_attention kernel modification"
d8b5011e8 | BUGFIX | Fix stride check
ec4dc0f6d | BUGFIX | [fix] fix tensor.data memory not free
ba70dece2 | BUGFIX | [fix] revert aclrtMalloc
2cc9bc67c | BUGFIX | fixed d4f27ed from https://gitee.com/darin123/pytorch/pulls/5124 [Feature] tensor support complex32
e6688feeb | BUGFIX | Fix complex bug.
6a858ae5f | BUGFIX | Fix building error when ccache doesn't exist
d104b149e | REVERT | Revert "Second-work flow must get more time-slot"
58ae668ce | REVERT | Revert "Correcte Time calculate-func"
37e48d896 | BUGFIX | Fix logaddexp/logaddexp2 PTA
d588783d1 | BUGFIX | fix remainder broadcast
858ed72ec | BUGFIX | fix TensorIterator bug for bf16
179a28982 | BUGFIX | fix TensorIterator bug for bf16
60074b509 | BUGFIX | fix TensorIterator bug for bf16
a4a3fde1f | BUGFIX | fix matmul oom temporarily
22217225f | BUGFIX | Fix subclass
ac06388a9 | BUGFIX | fix syncbatchnorm
f0bbf40d0 | BUGFIX | [fix] revert recorded_events
96abe60a8 | BUGFIX | [fix] revert recorded_events
674cbdaff | BUGFIX | [fix]hccl real time memory reuse
e8b2e78e0 | BUGFIX | [fix] revert recorded_events
1a9e24331 | BUGFIX | [fix] erase recorded_events
5fc2a836f | BUGFIX | [fix] revert recorded_events
46b6e2011 | BUGFIX | [fix]hccl real time memory reuse
ed069e9f7 | BUGFIX | 修复lt输出类型校验bug
67de7133f | BUGFIX | fix rnngard with seqlength, forget set direction
63f195876 | BUGFIX | Fix npu_autograd_functions by custom autograd
b15319ee4 | BUGFIX | fix bn
58fc914b2 | BUGFIX | [hostapi] embedding_bag bug fix
c92556c2e | REVERT | Revert "Combine format_contiguous with format_contiguous_add_copy_optimize"
aba7acb87 | REVERT | Revert "Combine format_contiguous with format_contiguous_add_copy_optimize"
6823afe2c | REVERT | Revert "Combine format_contiguous with format_contiguous_add_copy_optimize"
4e0517b78 | REVERT | Revert "Combine format_contiguous with format_contiguous_add_copy_optimize"
a182e63ee | REVERT | Revert "Second-work flow must get more time-slot"
9636cb900 | REVERT | Revert "Correcte Time calculate-func"
722afd152 | BUGFIX | fix bug for hccl_adaptor for bf16
3bba18365 | BUGFIX | fix bug for hccl_adaptor for bf16
4fe53fd51 | BUGFIX | fix bug for hccl_adaptor for bf16
990510041 | BUGFIX | Fix gen_variable_type func and fix filt npu func
e9c033771 | BUGFIX | fixed 3db4a20 from https://gitee.com/darin123/pytorch/pulls/4776 [Feature] check compile option
f9b761589 | BUGFIX | [fix] dropout genmask memory reuse
8ae04a1f5 | BUGFIX | fix: Non Empty Tensor Out when minlength > 0
bd9c6a307 | BUGFIX | [fix]hccl real time memory reuse
5287a5de5 | BUGFIX | [fix]hccl real time memory reuse
f893659cd | BUGFIX | [fix]hccl real time memory reuse
e8fa2c34d | REVERT | Revert "Second-work flow must get more time-slot"
87b1be0a9 | REVERT | Revert "Correcte Time calculate-func"
abb6b87b4 | REVERT | Revert "Second-work flow must get more time-slot"
3209c3042 | REVERT | Revert "Correcte Time calculate-func"
44fc879f4 | REVERT | Revert "Second-work flow must get more time-slot"
7ffe768d5 | REVERT | Revert "Correcte Time calculate-func"
4688f7352 | REVERT | Revert "Second-work flow must get more time-slot"
5c493f3cb | REVERT | Revert "Correcte Time calculate-func"
cba9c532c | REVERT | Revert "Second-work flow must get more time-slot"
9199e26eb | REVERT | Revert "Correcte Time calculate-func"
4a4d91d84 | BUGFIX | fixed 6f31c06 from https://gitee.com/xudaohong/pytorch/pulls/4834 [fix] TORCH_WARN_ONCE replace TORCH_WARN
0355a61cc | BUGFIX | fixed 6f31c06 from https://gitee.com/xudaohong/pytorch/pulls/4834 [fix] TORCH_WARN_ONCE replace TORCH_WARN
1b5858bc1 | BUGFIX | [fix]rollback
045412563 | BUGFIX | fixed d8ce5d4 from https://gitee.com/xudaohong/pytorch/pulls/4836 [fix] rollback
d8ce5d4e3 | BUGFIX | [fix] rollback
6f31c06fa | BUGFIX | [fix] TORCH_WARN_ONCE replace TORCH_WARN
8be844c47 | BUGFIX | [fix] TORCH_NPU_WARN_ONCE replace TORCH_WARN
01463a0c4 | BUGFIX | fix threshold:self dtype can cast result dtype
d3914eb96 | BUGFIX | bugfix:cumsum supports specified output dtype.
65b4c6991 | BUGFIX | bugfix:cumsum supports specified output dtype.
4e3f021b7 | BUGFIX | fix sum/upsample_nearest_1d
55e052cdf | BUGFIX | fix nn.Linear nz bug
bbcbbdc51 | BUGFIX | fix mm format ND bug
f13d3d0d2 | BUGFIX | Fix test_distributed
f781f20ac | BUGFIX | fixed c02baea from https://gitee.com/chen-tianyu19/pytorch_old/pulls/4773 revert repeatInterleave.Tensor
c02baeae4 | REVERT | revert repeatInterleave.Tensor
72e4e3675 | BUGFIX | Bug fix: filt torch functions in derivatives yaml
704133f50 | BUGFIX | fixed e703614 from https://gitee.com/xudaohong/pytorch/pulls/4683 [feat]aclrtMallocAlign32 replace aclrtMalloc
738ed8cdf | BUGFIX | fixed e703614 from https://gitee.com/xudaohong/pytorch/pulls/4683 [feat]aclrtMallocAlign32 replace aclrtMalloc
704191b4e | BUGFIX | fixed 1124fd1 from https://gitee.com/xudaohong/pytorch/pulls/4744 [feat]aclrtMallocAlign32 replace aclrtMalloc
50ff5a32d | BUGFIX | fixed e703614 from https://gitee.com/xudaohong/pytorch/pulls/4683 [feat]aclrtMallocAlign32 replace aclrtMalloc
d5f765f58 | BUGFIX | Fix a case where elapsed_time could return negative values
dc07ea827 | BUGFIX | fix header file and checkout func
6bfdbb5e9 | BUGFIX | [bugfix] Add patch for gru forward to support fixed length packed sequence, like lstm.
3606ae5ba | BUGFIX | [bugfix] Add patch for gru forward to support fixed length packed sequence, like lstm.
3243fbc09 | BUGFIX | fixed d438033 from https://gitee.com/wbigat/pytorch_master_wq/pulls/4725 set op wait timeout when hccl is initializing.
e9a9cbe52 | BUGFIX | fixed d438033 from https://gitee.com/wbigat/pytorch_master_wq/pulls/4725 set op wait timeout when hccl is initializing.
c50aa1a10 | BUGFIX | Fix the dtype in the calculation of sum.
6e349607c | BUGFIX | fix torch_warn invalid bug
2c986cccd | REVERT | Revert 'Pull Request !4660 : enable hostapi path as default after test ok'
1fc8ebeaf | REVERT | Revert Combine format_contiguous with format_contiguous_add_copy_optimize
406e891a9 | BUGFIX | fix write file bug
9be27aaae | REVERT | Revert Combine format_contiguous with format_contiguous_add_copy_optimize
82b6576b3 | BUGFIX | fixed 429a9b0 from https://gitee.com/guan-longfeng/pytorch/pulls/4637 update aclrtUtilizationInfo
ee4d612f8 | BUGFIX | fixed 429a9b0 from https://gitee.com/guan-longfeng/pytorch/pulls/4637 update aclrtUtilizationInfo
3efa2031d | BUGFIX | fixed 429a9b0 from https://gitee.com/guan-longfeng/pytorch/pulls/4637 update aclrtUtilizationInfo
865b36da2 | BUGFIX | Fix bug of indexput
a49578111 | BUGFIX | Fix bug of indexput
39f3848e9 | BUGFIX | Fix bug of indexput
d2c3b057a | BUGFIX | fix nn.Linear nz bug
138029b88 | BUGFIX | [fix]Fix warning flooding the screen.
78794eb6d | BUGFIX | [fix]Fix warning flooding the screen.
30d1d839e | BUGFIX | [fix]Fix warning flooding the screen.
88b5c210a | BUGFIX | [fix]Fix warning flooding the screen.
f3bc0d8a8 | BUGFIX | fixed e1503de from https://gitee.com/yangxiaorun1/pytorch/pulls/4567 [style]change print address style
26349294c | BUGFIX | Fix bug of torch.std
fc866df66 | BUGFIX | Fix the dtype in the calculation of sum.
e478f34fd | BUGFIX | Fix the dtype in the calculation of sum.
df37928cd | BUGFIX | [Fix]Fix return type to support arm
27a68e192 | BUGFIX | [Fix]Fix relative path join
bd787b08c | BUGFIX | fix baddbmm InputWithoutContiguous bug
b66ea151c | BUGFIX | fix baddbmm InputWithoutContiguous bug
f339e1d19 | BUGFIX | bugfix : gen result tensor with promote type
bd44ded85 | BUGFIX | Fix bug of torch.std
d05bf20b4 | BUGFIX | Bugfix for recordfunction.
aaa827173 | BUGFIX | fix operator with device arg to init device lazy
9ed8a9223 | BUGFIX | bug fix gelu_backward add promote & broadcast
90cbe38de | BUGFIX | Fix the output_size of nllloss2d.
afdf16c9b | BUGFIX | Fix the output_size of nllloss2d.
dd1eb5e21 | BUGFIX | Fix the output_size of nllloss2d.
4964f5349 | BUGFIX | Fix the output_size of nllloss2d.
e37db238e | BUGFIX | [fix]Using NPU_LOGW, output warning to screen.
b3ae0c2df | BUGFIX | [fix]Using NPU_LOGW, output warning to screen.
8666e23d4 | BUGFIX | [fix]Using NPU_LOGW, output warning to screen.
532d78016 | BUGFIX | [fix]Using NPU_LOGW, output warning to screen.
11ad58e79 | BUGFIX | Vecter copy bugfix.
65ecb9ee6 | BUGFIX | Vecter copy bugfix.
1237fbb8a | BUGFIX | Fix bug of torch.var
4a443b89a | BUGFIX | Fix bugs: when use mmcv,  inspect package raises built-in module errors
9e6845d90 | BUGFIX | fix dot output_size from 1d to 0d
26f55250f | BUGFIX | fix dot output_size from 1d to 0d
1e82ba7e1 | BUGFIX | fix dot output_size from 1d to 0d
b6a82e05c | BUGFIX | fix bug for v1.11.0 checkpoint
90cf09bab | BUGFIX | fix dot output_size from 1d to 0d
037a1eab9 | BUGFIX | [fix] Added error messages when using torch.Generator to create npu device objects.
90a89af3b | BUGFIX | Fix bug of var
8a5eaec25 | BUGFIX | [fix] Added error messages when using torch.Generator to create npu device objects.
f935b0477 | REVERT | Revert "[BugFix]Using item instead of ConvertTensorToScalar."
9cbd94e17 | BUGFIX | [DFX]add fixed device num check.
11b7b9a01 | BUGFIX | 1.pta of addmm and addmm_; 2.fix a bug of rrelu_with_noise_
cb0d7ba81 | BUGFIX | Fix bug of flip.
b2f3dc8a8 | BUGFIX | Fix bug of flip.
4fff1d7ef | BUGFIX | [Fix] When element <= correction, the result of cpu and npu is not same
38a166b42 | BUGFIX | fixed 9c81e7f from https://gitee.com/guan-longfeng/pytorch/pulls/4381 clean the torch_npu_get_utilization test
ade4ba75e | BUGFIX | Fix bug of flip.
5963fcc57 | BUGFIX | Fix bug of flip.
f8065f951 | BUGFIX | Fix the output_size of lerp.
6c36b379c | BUGFIX | Fix the output_size of lerp.
0b799a34b | BUGFIX | fixed e70e192 from https://gitee.com/guan-longfeng/pytorch/pulls/4293 clean the acl warn
4f3c4e69f | BUGFIX | Fix bug of torch.std
9f7d5a54f | BUGFIX | fixed 4e6aaa1 from https://gitee.com/wan-junwei4/pytorch/pulls/4324 fixed 268d477 from https://gitee.com/wan-junwei4/pytorch/pulls/4323 [BugFix]Add warn logs and remove feature not support errors.
28235008a | BUGFIX | [BugFix]Add warn logs and remove feature not support errors.
140e658bf | BUGFIX | Fix tensor compare bug
5253e8083 | BUGFIX | bugfix:eq support scalar
8108ea03f | BUGFIX | Fix repeatinterleave torch check
f4c5a0c5d | BUGFIX | Fix performance of contiguous with broadcast.
f19095e07 | BUGFIX | Fix RepeatInterleave output size infer
e42abbf08 | BUGFIX | Fix RepeatInterleave output size infer
d8fc7fbe0 | BUGFIX | Fix tensor compare bug
799f67468 | BUGFIX | bugfix:eq support scalar
c1c2002d6 | BUGFIX | Fix the output_size of lerp.
ede413279 | BUGFIX | Fix the output_size of lerp.
d958b5ecb | BUGFIX | fix empty_like dtype
a0e4f70fb | BUGFIX | fix dts: self-proposed
d57671049 | BUGFIX | fix bincount add parameter depend on hostapi
1eb7dd432 | BUGFIX | bugfix:cummax support indices
a80bc56b4 | BUGFIX | [fix] Modify autograd.profile device parameters in the test file.
2c943b20a | BUGFIX | Fix scatter + sort + Add repeatInterleave
f842e440c | BUGFIX | [BugFix]Using item instead of ConvertTensorToScalar.
101a0db75 | BUGFIX | fixed 00342e1 from https://gitee.com/guan-longfeng/pytorch/pulls/4182 fixed bdc5dcf from https://gitee.com/guan-longfeng/pytorch/pulls/4019 add device utilizationRate
3f506e64a | BUGFIX | fix addr: 1.use mul instead of matmul;2. fix err msg;3. fix result dtype err
4d76ca918 | BUGFIX | fix addr: 1.use mul instead of matmul;2. fix err msg;3. fix result dtype err
c70dd2201 | BUGFIX | fix addr: 1.use mul instead of matmul;2. fix err msg;3. fix result dtype err
e2d85de18 | BUGFIX | fix addr: 1.use mul instead of matmul;2. fix err msg;3. fix result dtype err
2c451fe74 | BUGFIX | bugfix:cummax support indices
26e295d31 | BUGFIX | bugfix:cummax support indices
e89aef06e | BUGFIX | bugfix:cummax support indices
00342e1a3 | BUGFIX | fixed bdc5dcf from https://gitee.com/guan-longfeng/pytorch/pulls/4019 add device utilizationRate
f25d55277 | BUGFIX | fix pta for erf
ba39b7810 | BUGFIX | Fix checkout of addcdiv_out.
8b10c0553 | BUGFIX | Fix checkout of addcdiv_out.
3dbaf4d9d | BUGFIX | [DFX]add fixed device num check.
66f9b7fd1 | BUGFIX | [DFX]add fixed device num check.
92cfdb4f8 | BUGFIX | Fix checkout of addcdiv_out.
84509df89 | BUGFIX | Fix checkout of addcdiv_out.
a7054b30c | BUGFIX | [Fix]: add switch for tanh in native_functions.yaml
ff538b25f | BUGFIX | [fix]Using item instead of ConvertTensorToScalar.
69b8a738f | BUGFIX | fix init bug: import torch must be at the begining
b3cc9da85 | BUGFIX | [fix] amp _amp_foreach_non_finite_check_and_unscale_ op performance issue
c0daf5dfa | BUGFIX | [fix] amp _amp_foreach_non_finite_check_and_unscale_ op performance issue
79b7158ac | BUGFIX | [fix] amp _amp_foreach_non_finite_check_and_unscale_ op performance issue
addb8348a | BUGFIX | [fix] amp _amp_foreach_non_finite_check_and_unscale_ op performance issue
54150af02 | BUGFIX | fix adaptive_avg_pool2d bug
639089d4b | BUGFIX | [fix]Using item instead of ConvertTensorToScalar.
a3a89f9ff | BUGFIX | Fix raw_delete bug
ea5ce0b28 | BUGFIX | Fix the output_size of normal.
965a23519 | BUGFIX | Fix the output_size of normal.
adafe9332 | BUGFIX | Fix the output_size of normal.
0b931ddb8 | BUGFIX | Fix the output_size of normal.
606585778 | BUGFIX | fix cluster stuck during training.
d9fd86b35 | BUGFIX | fix cluster stuck during training.
8b2089429 | BUGFIX | fix cluster stuck during training.
154cc4cc9 | BUGFIX | fix sum
18d61921d | BUGFIX | fixed b6d05e9 from https://gitee.com/lingyu-ye/pytorch/pulls/4055 Fix functional errors in nms_rotated ut.
30130645d | BUGFIX | fixed b6d05e9 from https://gitee.com/lingyu-ye/pytorch/pulls/4055 Fix functional errors in nms_rotated ut.
37f362be9 | BUGFIX | fixed b6d05e9 from https://gitee.com/lingyu-ye/pytorch/pulls/4055 Fix functional errors in nms_rotated ut.
b6d05e93d | BUGFIX | Fix functional errors in nms_rotated ut.
ca1772ac5 | BUGFIX | fix div: add check 'has_value' before '*rounding_mode'
bfa55bc0c | BUGFIX | fix div: add check 'has_value' before '*rounding_mode'
c73782b81 | BUGFIX | fix sub: device err with sub(scalar, tensor)
38d8123c9 | BUGFIX | fix sub: device err with sub(scalar, tensor)
e4504f0ff | BUGFIX | fix Path3 code
7d87d5791 | BUGFIX | fix stack
5da25378a | BUGFIX | fix div: add check 'has_value' before '*rounding_mode'
2cbd7189c | BUGFIX | fixed 0640dd2 from https://gitee.com/wan-junwei4/pytorch/pulls/4000 fixed c4e7ff1 from https://gitee.com/wan-junwei4/pytorch/pulls/3998 [BugFix]Fix bug, add the processing of bool, bfloat16, and complex types to ConverTensorToScalar.
0640dd2be | BUGFIX | fixed c4e7ff1 from https://gitee.com/wan-junwei4/pytorch/pulls/3998 [BugFix]Fix bug, add the processing of bool, bfloat16, and complex types to ConverTensorToScalar.
c4e7ff1af | BUGFIX | [BugFix]Fix bug, add the processing of bool, bfloat16, and complex types to ConverTensorToScalar.
e531cfaaa | BUGFIX | [BugFix]: fix the checkout bugs in tanh
66585a052 | BUGFIX | fix sub: device err with sub(scalar, tensor)
f4f60a00b | BUGFIX | fix sub: device err with sub(scalar, tensor)
7c1800f80 | BUGFIX | bugfix:remove dtype check between self and out
cb281820f | BUGFIX | [bugfix] modify ut frame function error
0167ac0c2 | BUGFIX | fix shape dtype
1d5ac4bef | BUGFIX | Fix raw_delete bug
f42bb3fe4 | BUGFIX | Fix raw_delete bug
56322f6bf | BUGFIX | Fix raw_delete bug
2376bf83f | BUGFIX | Fix raw_delete bug
244b0df9e | BUGFIX | fix aclnnLtTensor pta
1912c24b2 | BUGFIX | bugfix_memory
1afeb923b | BUGFIX | bugfix_memory
a92ab0ce5 | BUGFIX | bugfix_memory
be580b7e0 | BUGFIX | [fix] new_ones argument supports passing tensor.
72d62f183 | BUGFIX | [fix] new_ones argument supports passing tensor.
0f009733f | BUGFIX | Fix all_gather_object always use hccl.
0241d0580 | BUGFIX | [Fix] torch.Generator supports passing npu device parameters.
46a574a43 | BUGFIX | [Fix] torch.Generator supports passing npu device parameters.
d318a95e0 | BUGFIX | [Fix] torch.Generator supports passing npu device parameters.
47a03bbb0 | BUGFIX | Bugfix in stream judgement
a70950331 | BUGFIX | Bugfix in stream judgement
891e2d00d | BUGFIX | Bugfix in stream judgement
0e2a0a80e | BUGFIX | Bugfix in stream judgement
63250f2e2 | BUGFIX | fix thresholdback and hardswish
ecee54117 | BUGFIX | Fix all_gather_object always use hccl.
98b4fa2f9 | BUGFIX | Fix CorrelationID.
ff425a6d3 | BUGFIX | Fix bug of native_dropout.
c557a2d6d | BUGFIX | Fix bug of native_dropout.
4e8bf01b0 | BUGFIX | Fix bug of native_dropout.
115dae2a8 | BUGFIX | Add UT test cases for transfer to npu and fix the issue of cuda.random not being replaced
736131835 | BUGFIX | fix sum: ut add format
3ca9e4f24 | BUGFIX | fix max ops to handle tensors from different devices.
ce1df092e | BUGFIX | fix max ops to handle tensors from different devices.
ce1c5b585 | BUGFIX | fix sum: dtype of result for empty tensor with bool different with cpu
093a48066 | BUGFIX | fix max ops to handle tensors from different devices.
a52213d92 | BUGFIX | fix max ops to handle tensors from different devices.
33bba6dc4 | BUGFIX | Fix max_pool2d_with_indices to support 3D-input.
a10cae357 | BUGFIX | Fix max_pool2d_with_indices to support 3D-input.
79d70331e | BUGFIX | [Fix] fix bug of event reuse by reset event.
f03cad93e | BUGFIX | fixed dfc1795 from https://gitee.com/wbigat/pytorch_master_wq/pulls/3595 [Fix] fix bug of event reuse by reset event.
b8a162fc8 | BUGFIX | [Fix] fix bug of event reuse by reset event.
ab0263426 | BUGFIX | Fix max_pool2d_with_indices to support 3D-input.
5d92078d3 | BUGFIX | fix var op with correction param
a23d65492 | BUGFIX | fix sum: dtype of result for empty tensor with bool different with cpu
dfc17957e | BUGFIX | [Fix] fix bug of event reuse by reset event.
eb2ac96c2 | BUGFIX | Add UT test cases for transfer to npu and fix the issue of cuda.random not being replaced
ba794130f | BUGFIX | Add UT test cases for transfer to npu and fix the issue of cuda.random not being replaced
aaf042b21 | BUGFIX | fix calling geinit more than once when sympy pkg is not installed.
8aec0dc45 | BUGFIX | Fix NPU selection 0 card set_device failure issue, and handle the issue of "module torch_npu._C has no attribute device" and remove redundant code
e6e690e84 | BUGFIX | fix calling geinit more than once when sympy pkg is not installed.
9b53a5422 | BUGFIX | fix calling geinit more than once when sympy pkg is not installed.
45ce39039 | BUGFIX | fix calling geinit more than once when sympy pkg is not installed.
d85133983 | BUGFIX | Fixed _DDPJoinHook undefined.
577860d35 | BUGFIX | Fixed _DDPJoinHook undefined.
fd2d9dada | BUGFIX | Fix the bug of torch.median API not support empty input parameters
2cbf6d785 | BUGFIX | fix masked_fill ops when the inputs are placed on different devices.
57fd19f7b | BUGFIX | fix masked_fill ops when the inputs are placed on different devices.
e46d1d858 | BUGFIX | fix masked_fill ops when the inputs are placed on different devices.
f97d21f62 | BUGFIX | fix masked_fill ops when the inputs are placed on different devices.
b2d6b8416 | BUGFIX | Fix the bug of torch.median API not support empty input parameters
ad98a6df3 | BUGFIX | fix gather element bug
84eafd702 | BUGFIX | fix index interrupt graph bug when index is int/long
632e9fc09 | BUGFIX | Fix the output type of mul_out.
1db43a649 | BUGFIX | Fix the output type of mul_out.
f227d8db9 | BUGFIX | fixed 67fadc1 from https://gitee.com/wbigat/pytorch_master_wq/pulls/3571 [Fix] Set HF32 default value of matmul to False in all version.
67fadc155 | BUGFIX | [Fix] Set HF32 default value of matmul to False in all version.
b0d8eac92 | BUGFIX | Fix bug when raising error in _empty_with_format
01b954d95 | BUGFIX | Fix torch.all for supporting all dtypes.
677499e4c | BUGFIX | Bugfix.
18682df05 | BUGFIX | Fix torch.all for supporting all dtypes.
0f776024f | BUGFIX | Fix torch.all for supporting all dtypes.
6b9284c55 | BUGFIX | Bugfix.
139aebde1 | BUGFIX | Fix the output type of mul_out.
a7eae332b | BUGFIX | Fix the output type of mul_out.
418d67c2f | BUGFIX | fix sum: dtype of result for empty tensor with bool different with cpu
63654084a | BUGFIX | 【bugfix】Add patch for gru forward to support fixed length packed sequence, like lstm.
42bd5684b | BUGFIX | 【bugfix】Add patch for gru forward to support fixed length packed sequence, like lstm.
d3b4b871e | BUGFIX | Fix the output when the padding in the PadV3Grad is all 0.
5d82b479c | BUGFIX | Fix the output when the padding in the PadV3Grad is all 0.
604f016b2 | BUGFIX | Fix the output when the padding in the PadV3Grad is all 0.
0d4775157 | BUGFIX | Fix the output when the padding in the PadV3Grad is all 0.
44547966f | BUGFIX | Fix bug in warning decorator
f2b3dbb53 | BUGFIX | Fix bug in warning decorator
ff4fdb616 | BUGFIX | [fix] fix npu_format_cast op deprecated problem
82c2474e5 | BUGFIX | fixed 7cbc84b from https://gitee.com/wbigat/pytorch_master_wq/pulls/3350 [Feature] Complete the functions of storage on NPU.
d6d580fc1 | BUGFIX | Fix logger is null.
926d82e7a | REVERT | Revert 'Pull Request !3382 : delete pin_memory and is_pinned warning'
9158a1500 | BUGFIX | Fix bug of indexput
a22107277 | BUGFIX | Fix bug of indexput
ec4055228 | BUGFIX | Fix bug of indexput
a9c484f1b | BUGFIX | Fix bug of indexput
1be5186e1 | BUGFIX | Fix logger is null.
d76bf081c | REVERT | Revert 'Pull Request !3383 : delete pin_memory and is_pinned warning in 2.0'
f230c3e27 | BUGFIX | fix the bug that testcase doesn't run with testing.run_test
375917cef | BUGFIX | [Fix]Fix security vulnerabilities in torch.jit.annotations.parse_type_line
59bce6b03 | BUGFIX | [Fix]Fix security vulnerabilities in torch.jit.annotations.parse_type_line.
f33097e03 | BUGFIX | Fix bug of of indexput
7444b8433 | BUGFIX | Fix bug of of indexput
c67b7ab0e | BUGFIX | Fix bug of of indexput
dc1e74154 | BUGFIX | Fix bug of of indexput
34950b0f8 | BUGFIX | fix the UT for error interception.
7ed9f89e2 | BUGFIX | fix the UT for error interception.
b8a9c8e50 | BUGFIX | fix the UT for error interception.
86464c7db | BUGFIX | fix attribute check of fused optimizers
cf0249cae | BUGFIX | fix attribute check of fused optimizers
6007bc108 | BUGFIX | fix attribute check of fused optimizers
d3674383c | BUGFIX | fix attribute check of fused optimizers
f1616ae3e | BUGFIX | fix attribute check of fused optimizers
8b082e7e2 | BUGFIX | [Fix]Fix security vulnerabilities in torch.jit.annotations.parse_type_line
c5efbc0e8 | BUGFIX | [Fix]Fix security vulnerabilities in torch.jit.annotations.parse_type_line.
51a72ed0e | BUGFIX | Fix NPU selection 0 card set_device failure issue
cb0bbee67 | BUGFIX | Fix NPU selection 0 card set_device failure issue
e0a360958 | BUGFIX | Fix NPU selection 0 card set_device failure issue
c2bb75554 | BUGFIX | fixed e0e2e82 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/3321 update replay graph and optimze memory usage.
7b7fd29ef | BUGFIX | fixed e0e2e82 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/3321 update replay graph and optimze memory usage.
17689d60b | BUGFIX | fix the bug that testcase doesn't run with testing.run_test
584b03308 | BUGFIX | fix the bug that testcase doesn't run with testing.run_test
29f0e092f | BUGFIX | fix the bug that testcase doesn't run with testing.run_test
356a29da7 | BUGFIX | fixed e0e2e82 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/3321 update replay graph and optimze memory usage.
430f7f1eb | BUGFIX | Fix wild pointer
62155d8f6 | BUGFIX | fixed 514a9f3 from https://gitee.com/guan-longfeng/pytorch/pulls/3314 fixed 725540a from https://gitee.com/guan-longfeng/pytorch/pulls/3304 Add a conversion type from complex Aten dtype to acl dtype
514a9f31d | BUGFIX | fixed 725540a from https://gitee.com/guan-longfeng/pytorch/pulls/3304 Add a conversion type from complex Aten dtype to acl dtype
1fd423491 | BUGFIX | fixed 725540a from https://gitee.com/guan-longfeng/pytorch/pulls/3304 Add a conversion type from complex Aten dtype to acl dtype
37f725760 | BUGFIX | Fix the test case(test_c10d.py). Format NZ cast dose not support one dimensional tensor. Change the tensor to 2 dimensions to support the private format NZ.
83ff2ca5a | BUGFIX | Fix the test case(test_c10d.py). Format NZ cast dose not support one dimensional tensor. Change the tensor to 2 dimensions to support the private format NZ.
4040f9f1c | BUGFIX | fix amp&npu ut
1721f1534 | BUGFIX | Fix bug of argmin.
43c766bc8 | BUGFIX | Fix bug of argmin.
7630784a3 | BUGFIX | Fix bug of argmin.
c3496da62 | BUGFIX | Fix bug of argmin.
6337b8b97 | BUGFIX | Fix no Tensor in random.py.
a5f8402e8 | BUGFIX | fix bug for scatter.
d0dc162e9 | BUGFIX | fixed 7ef8d47 from https://gitee.com/guan-longfeng/pytorch/pulls/3094 ops add deterministicAlgorithms config
e5d78c846 | BUGFIX | fixed 7ef8d47 from https://gitee.com/guan-longfeng/pytorch/pulls/3094 ops add deterministicAlgorithms config
05d248761 | BUGFIX | Fix noncontiguous tensor serialization load error
af5a572ab | BUGFIX | Fix noncontiguous tensor serialization load error
4c8dc7cfa | BUGFIX | Fix noncontiguous tensor serialization load error
dc1f1844e | BUGFIX | Fix noncontiguous tensor serialization load error
e94b9a4ef | BUGFIX | Fix noncontiguous tensor serialization load error
7078c4823 | BUGFIX | Fix noncontiguous tensor serialization load error
4b10cfe76 | BUGFIX | fixed c962f21 from https://gitee.com/liu_zhi_xu/pytorch/pulls/3092 fixed 64ec395 from https://gitee.com/liu_zhi_xu/pytorch/pulls/3091 fixed 3ec2c0d from https://gitee.com/liu_zhi_xu/pytorch/pulls/3090 Type：Modify CTC. Team：PyTorch_Ops_Dev
c962f2176 | BUGFIX | fixed 64ec395 from https://gitee.com/liu_zhi_xu/pytorch/pulls/3091 fixed 3ec2c0d from https://gitee.com/liu_zhi_xu/pytorch/pulls/3090 Type：Modify CTC. Team：PyTorch_Ops_Dev
64ec395b2 | BUGFIX | fixed 3ec2c0d from https://gitee.com/liu_zhi_xu/pytorch/pulls/3090 Type：Modify CTC. Team：PyTorch_Ops_Dev
035270e64 | BUGFIX | Fix bug for DDPJoinHook_init_.
41ab95456 | BUGFIX | Fix bug for DDPJoinHook_init_.
07bbcc195 | BUGFIX | fixed 0a57755 from https://gitee.com/guan-longfeng/pytorch/pulls/2976 add torch.npu.get_device_capability, torch.npu.aclnn.version new interface
950bf7335 | BUGFIX | fixed 6454d4a from https://gitee.com/yangxiaorun1/pytorch/pulls/3044 style：change Log format
055745ae0 | BUGFIX | fixed 6454d4a from https://gitee.com/yangxiaorun1/pytorch/pulls/3044 style：change Log format
410e1a890 | BUGFIX | fixed 4b6531d from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/3005 [Featur] support HF32
b50381663 | BUGFIX | Fix TORCH_CHECK in logSpace
5b27a8fac | BUGFIX | Fix adapter for random_
93ee72eaa | BUGFIX | Fix TORCH_CHECK in logSpace
6acebe4da | REVERT | Revert "[Fix] During shutdown, removing operations for device resources such as events and stream."
520c9b8e5 | REVERT | Revert "fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq/pulls/2905"
bc3b0f667 | REVERT | Revert "fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/2905"
9415c1dcf | REVERT | Revert "fixed 63d8ec5 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/2948"
ac336f97d | REVERT | Revert "fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq/pulls/2905"
b581b6c8c | REVERT | Revert "!2949 [Fix] fix core dump bug after npu shut down."
a243de6e2 | BUGFIX | !2949 [Fix] fix core dump bug after npu shut down. * fixed 63d8ec5 from https://gitee.com/wbigat/pytorch_master_wq/pulls/2948
f2613f58e | BUGFIX | fixed 63d8ec5 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/2948 [Fix] fix core dump bug after npu shut down.
bac7acf61 | BUGFIX | fix MakeSureQueueEmpty deadlock
d46d13162 | BUGFIX | fix MakeSureQueueEmpty deadlock
b5d0083a4 | BUGFIX | fix MakeSureQueueEmpty deadlock
c2bb08568 | BUGFIX | fix MakeSureQueueEmpty deadlock
6a9436c77 | BUGFIX | [Bugfix] throw error when PAT version mismatch
177dfbdab | BUGFIX | [Bugfix] throw error when PAT version mismatch
2aec3fbd4 | BUGFIX | [Bugfix] throw error when PAT version mismatch
e81428eab | BUGFIX | [Bugfix] throw error when PAT version mismatch
7773589d2 | BUGFIX | fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq/pulls/2905 [Fix] During shutdown, removing operations for device resources such as events and stream.
4159cdc2b | BUGFIX | fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq/pulls/2905 [Fix] During shutdown, removing operations for device resources such as events and stream.
30d6c720d | BUGFIX | fixed c180c88 from https://gitee.com/wbigat/pytorch_master_wq_2/pulls/2905 [Fix] During shutdown, removing operations for device resources such as events and stream.
360419c44 | BUGFIX | fix tensor record stream
956581db7 | BUGFIX | [Fix]add new_ones and new_tensor for npu device initialization.
2ba670813 | BUGFIX | [Fix]Parse the size argument of new_ones from int to tuple.
e160dbfe4 | BUGFIX | Type：BugFix Index op Team：PyTorch_Ops_Dev
2412c5ad8 | BUGFIX | Type：BugFix Index op Team：PyTorch_Ops_Dev
c180c8839 | BUGFIX | [Fix] During shutdown, removing operations for device resources such as events and stream.
83504ce1f | BUGFIX | fix sensitive format bug for Bn
6cce587de | BUGFIX | Type：BugFix Index op Team：PyTorch_Ops_Dev
cfc12d5d1 | BUGFIX | Type：BugFix Index op Team：PyTorch_Ops_Dev
0f28eb1ea | BUGFIX | fix the conflict
c909295e5 | BUGFIX | Fix the bug of torch.median API not support empty input parameters
7d1f9a066 | BUGFIX | fixed e783f0a from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2862 fix: overflow should enable in stream init
05a148acc | BUGFIX | Fix ci, use python3.8 to run ut.
e4555f325 | BUGFIX | Fix ci, use python3.8.
cedde8a21 | BUGFIX | Fix bug of clear_npu_overflow_flag
128eca241 | BUGFIX | Fix bug of clear_npu_overflow_flag
3c5c0a09e | BUGFIX | fixed 2d4c60d from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2856 feat: change jit_compile default value from ACL
0968bc3e1 | BUGFIX | fixed 2d4c60d from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2856 feat: change jit_compile default value from ACL
2e676bbb5 | BUGFIX | fixed 2d4c60d from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2856 feat: change jit_compile default value from ACL
55a787c10 | BUGFIX | fixed e783f0a from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2862 fix: overflow should enable in stream init
a258b284a | BUGFIX | fixed e783f0a from https://gitee.com/yangxiaorun1/pytorch_1/pulls/2862 fix: overflow should enable in stream init
e783f0a99 | BUGFIX | fix: overflow should enable in stream init
dd90ae923 | BUGFIX | [Fix]Fix the bug of mul_ when input is bool.
0301996b0 | BUGFIX | [Fix] Syncbatchnorm precision overflow problem.
b6111aaf0 | BUGFIX | [Fix] Syncbatchnorm precision overflow problem.
6edc2cf97 | BUGFIX | [Fix] Syncbatchnorm precision overflow problem.
463770c83 | BUGFIX | [Fix]add new_ones and new_tensor for npu device initialization.
83649ad54 | BUGFIX | [Fix]add new_ones and new_tensor for npu device initialization.
1da500443 | BUGFIX | fix vsplit test
160a8a03f | BUGFIX | fixed 7634feb from https://gitee.com/lingyu-ye/pytorch/pulls/2747 Correct apply-tensor-with object for the output selectedBox.
4a9bed6a6 | BUGFIX | Fix ascend profiler bug
9bf987e17 | BUGFIX | Fix ascend profiler bug
ff7b21de8 | BUGFIX | Fix ascend profile bug
c2b2d67b1 | BUGFIX | fixed 61cfa41 from https://gitee.com/lingyu-ye/pytorch/pulls/2747 Correct apply-tensor-with object for the output selectedBox.
711c8bd40 | BUGFIX | fixed 61cfa41 from https://gitee.com/lingyu-ye/pytorch/pulls/2747 Correct apply-tensor-with object for the output selectedBox.
5450f8627 | BUGFIX | [Fix] Fix the bug of mm when input is empty tensor.
561a19dd1 | REVERT | Revert 'Pull Request !2695 : [Fix] add _lazy_init in torch_device_guard.'
62660805f | REVERT | Revert 'Pull Request !2699 : [Fix] add _lazy_init in torch_device_guard.'
b7b7c4b23 | BUGFIX | Fix the bug of the SigmoidCrossEntropyWithLogitsV2 and SigmoidCrossEntropyWithLogitsGradV2.
04b70e70a | BUGFIX | Fix the bug of the SigmoidCrossEntropyWithLogitsV2 and SigmoidCrossEntropyWithLogitsGradV2.
ee6de8fe3 | BUGFIX | Fix the bug of the SigmoidCrossEntropyWithLogitsV2 and SigmoidCrossEntropyWithLogitsGradV2.
697ddb085 | BUGFIX | Fix bug when printing npu native format tensor.
a01c2da2e | BUGFIX | Fix the bug of the SigmoidCrossEntropyWithLogitsV2 and SigmoidCrossEntropyWithLogitsGradV2.
749c13879 | BUGFIX | Fix the bug of the SigmoidCrossEntropyWithLogitsV2 and SigmoidCrossEntropyWithLogitsGradV2.
03615052e | BUGFIX | [Bugfix] modify event timeout.
c4f4b16c2 | BUGFIX | [Bugfix] modify event timeout.
b081ddfc8 | BUGFIX | [Bugfix] modify event timeout.
46dba976a | BUGFIX | Fix the outputpadding of conv_transpose2d.
ae42d2cd5 | BUGFIX | [Fix] fix AclGetErrMsg and do not return nullptr.
cfe666a3a | BUGFIX | [Fix] fix AclGetErrMsg and do not return nullptr.
43446edb9 | BUGFIX | [Fix] fix AclGetErrMsg and do not return nullptr.
3b5d1b2ec | BUGFIX | Fix the outputpadding of conv_transpose2d.
8ab398391 | BUGFIX | [Fix] add _lazy_init in torch_device_guard.
731ebffb8 | BUGFIX | [Fix] add _lazy_init in torch_device_guard.
517d0fe14 | BUGFIX | Fixed memory_summary function call errors.
ddda05b1b | BUGFIX | Fixed memory_summary function call errors.
62441e8bd | BUGFIX | 回退 'Pull Request !2599 : [BugFix] Change unique_consecutive's output[1] to dynamic output.'
81451d727 | BUGFIX | fixed c529aee from https://gitee.com/lingyu-ye/pytorch/pulls/2683 Add testcases for custom torch_npu ops.
a0fb8365e | BUGFIX | fixed c529aee from https://gitee.com/lingyu-ye/pytorch/pulls/2683 Add testcases for custom torch_npu ops.
52790f0c6 | BUGFIX | Update BiShengCPP examples. Fixed afd4fdf from https://gitee.com/shaoyf/pytorch/pulls/2549.
104170bf2 | BUGFIX | [fix] fix binary_cross_entropy_with_logits_backward bug.
4df628307 | BUGFIX | Fix bug when print tensor with npu native format
adc2c0ec0 | BUGFIX | Fix bug when print tensor with npu native format
b0601ff8e | BUGFIX | Fix bug when print tensor with npu native format
9c325d3d0 | BUGFIX | bugfix: set attr is_fused_optimizer in base
6652e9ee0 | BUGFIX | Fix gru_backward when batch_size==1.
946a0e8ec | BUGFIX | Fix gru_backward when batch_size==1.
e5bfbb640 | BUGFIX | Fix gru_backward when batch_size==1.
b6ac1263b | BUGFIX | Fix gru_backward when batch_size==1.
c9b798158 | BUGFIX | Fix gru_backward when batch_size==1.
6663f93b4 | BUGFIX | [bugfix] Sync with origin torch, modify the patch func to support the dim of input tensor is 2 in lstm.
e01534d9d | BUGFIX | bugfix: set attr is_fused_optimizer in base
ac7df74e4 | BUGFIX | bugfix: set attr is_fused_optimizer in base
78d98ce6f | BUGFIX | Fix onnx export for npu_nms_v4
6942e329d | BUGFIX | Fix onnx export for npu_nms_v4
24e589fce | BUGFIX | fixed 854e13a from https://gitee.com/yangxiaorun1/pytorch/pulls/2540 feat: change default configuration: precison_mode as must_keep_origin_dtype in Ascend910B1
9203c3932 | BUGFIX | [fix] fix binary_cross_entropy_with_logits_backward bug
5351ba61c | BUGFIX | [fix] fix datadump bugs
4ecb83fbc | BUGFIX | [fix] fix datadump bugs
2e865c392 | BUGFIX | [bugfix] Add dtype cast when input dtype is int64, because the OneHot cann't support it.
8e33c02f0 | BUGFIX | [bugfix] Add dtype cast when input dtype is int64, because the OneHot cann't support it.
679506d1e | BUGFIX | fixed 0a0b9bb from https://gitee.com/yangxiaorun1/pytorch/pulls/2423 fix: [BugFix] API compatibility is maintained with CUDA
185b7b4e0 | BUGFIX | fix libtorch build bug
0a0b9bb85 | BUGFIX | fix: [BugFix] API compatibility is maintained with CUDA
c01fcce50 | BUGFIX | Fix uniform kernel error.
2236df3fd | BUGFIX | Fix uniform kernel error.
7393283fa | BUGFIX | Fix uniform kernel error.
d8bdf5426 | BUGFIX | Fix uniform kernel error.
bfbda2826 | BUGFIX | [Fix]index_put supports the tensors of the input for different devices
68f90c1b8 | BUGFIX | [Fix]index_put supports the tensors of the input for different devices
ae5da959b | BUGFIX | [Fix] add interface for overflow enable switch.
12068adce | BUGFIX | [Fix] add interface for overflow enable switch.
eb13e4af2 | BUGFIX | fix build shell bug in ubuntu with dash
ef47a4d8e | REVERT | Revert npu memory management.
11fc2307f | BUGFIX | [Bugfix] Check aclrtSetOpWaitTimeout interface.
a577007cd | BUGFIX | [Bugfix] Check aclrtSetOpWaitTimeout interface.
dde0aa020 | BUGFIX | 【bugfix】Add patch for gru forward to support fixed length packed sequence, like lstm.
2fa5dc6b0 | BUGFIX | 【bugfix】Support the param out is a slice of other tensor, in max.
3e5d81d50 | BUGFIX | [Fix]Improve test cases for npu_strided_copy.
cc5760e51 | BUGFIX | Fix embedding_bag in 0D scenario.
7d4457922 | BUGFIX | Fix embedding_bag in 0D scenario.
883d7e923 | BUGFIX | Fix embedding_bag in 0D scenario.
512569b74 | BUGFIX | Fix embedding_bag in 0D scenario.
f5dc3c11e | BUGFIX | [Fix]Improve test cases for npu_strided_copy.
20750f0a9 | BUGFIX | Fix uniform kernel forward error.
ef4ff6cce | BUGFIX | Fix uniform kernel forward error.
fa84cc897 | BUGFIX | Fix uniform kernel forward error.
39a2f3a71 | BUGFIX | [Fix] revert overflow switch set code for cordump problem.
ec1395541 | BUGFIX | [fix] bugfix of supported data types by hccl ops
2b057da5f | BUGFIX | [FIX] Torch_npu is compatible for rts feature err.
c98e57cf8 | BUGFIX | [FIX] Torch_npu is compatible for rts feature err.
3545252c1 | BUGFIX | Fix the deadlock in taskqueue.
0d27919b9 | BUGFIX | Fix the deadlock in taskqueue
89c641229 | BUGFIX | Fix the deadlock in taskqueue
3eabfefbd | BUGFIX | Fix the deadlock in taskqueue
a0f5177f8 | BUGFIX | Fix uniform kernel forward error.
e9d4bc1f5 | BUGFIX | Fix input device for get_rng_state and set_rng_state.
1dbace382 | BUGFIX | Fix the accuracy problem of nllloss2d.
50d3504b5 | BUGFIX | Fix the input tensor src_stride under aicore mode.
03d2c5ee2 | BUGFIX | Fix div dtype bug.
29dc2ec3f | BUGFIX | Fix the input tensor src_stride under aicore mode.
309027abb | BUGFIX | Fix the input tensor src_stride under aicore mode.
9af82239b | BUGFIX | Fix the input tensor src_stride under aicore mode.
9b37b6f55 | BUGFIX | Fix input device for get_rng_state and set_rng_state.
1cb32e90a | BUGFIX | Fix input device for get_rng_state and set_rng_state.
3b45846a5 | BUGFIX | Fix input device for get_rng_state and set_rng_state.
746688c5b | BUGFIX | Fix output shape of lstm_backward.
1e4f0d803 | BUGFIX | Fix output shape of lstm_backward.
3316e55c7 | BUGFIX | Fix output shape of lstm_backward.
2d4d9718b | BUGFIX | Fix output shape of lstm_backward.
98c5b0950 | BUGFIX | fix isinstance bug in transfer_to_npu tools
e8697157a | BUGFIX | fix isinstance bug in transfer_to_npu tools
1d39838a3 | BUGFIX | fix isinstance bug in transfer_to_npu tools
418b3a398 | BUGFIX | fix isinstance bug in transfer_to_npu tools
5913f4384 | BUGFIX | fix device constructor bug
f65d54dcf | BUGFIX | fix device constructor bug
3c7fc6917 | BUGFIX | fix device constructor bug
97264f66e | BUGFIX | fix device ctor bug
0edce2564 | BUGFIX | Fix clamp in int64 scenario.
667cead43 | BUGFIX | Fix clamp in int64 scenario.
c95745e79 | BUGFIX | [Fix] parameter parsing error when 'device=None'.
f3a7068e3 | BUGFIX | [Fix] parameter parsing error when 'device=None'.
53fbed4d3 | BUGFIX | [Fix] parameter parsing error when 'device=None'.
632ecf49e | BUGFIX | fix device class bug
11a6a3ba0 | BUGFIX | fix device class bug
3633a172e | BUGFIX | fix device class bug
e57a52fe9 | BUGFIX | fix device class bug
24d1c22e1 | BUGFIX | fix assert for torch_npu._C.device
6603a558a | BUGFIX | fix assert for torch_npu._C.device
d3d4f0385 | BUGFIX | fix assert for torch_npu._C.device
e46b4eafe | BUGFIX | Fix clamp in int64 scenario.
cf493af9d | BUGFIX | Fix clamp in int64 scenario.
41a0d2e91 | BUGFIX | fix device type bug in serialization
ef26f74b4 | BUGFIX | Fix bug in storage_resize_npu.
329b39a5a | BUGFIX | Fix bug in storage_resize_npu.
e7bf9e6a1 | BUGFIX | Fix bug in storage_resize_npu
589c19a5e | BUGFIX | [fix] fix datadump enque input is cpu tensor
244c83b81 | BUGFIX | [fix] fix datadump enque input is cpu tensor
3e74e3328 | BUGFIX | fix(CalcuOpUtil.*): Scalartype QUInt2x4 has been added into pytorch1.11,so kATenScalarTypeToAclDataTypeTable must be updated
d28187650 | BUGFIX | [Fix] Fix format bug of resize
3644383fa | BUGFIX | [Fix] Fix format bug of resize
e2a540870 | BUGFIX | Fix invalid output shape allocation of mask
d7b5bd636 | BUGFIX | Fix invalid output shape allocation of mask
d161b1867 | BUGFIX | [Fix] fix the bug of empty_strided, when resize occurs.
748a00159 | BUGFIX | [Fix] fix the bug of empty_strided, when resize occurs.
34c9f17eb | BUGFIX | [fix] fix datadump problem caused by torch v1.11.0 code merge
0d2b78ca2 | BUGFIX | Fix upsample_nearest1d to support dynamic scenes.
3356f3701 | BUGFIX | Fix upsample_nearest1d to support dynamic scenes.
0ee3963ed | BUGFIX | [Fix] Fixed npu device not torch.device type issue.
3593e4865 | BUGFIX | [Fix] Fixed npu device not torch.device type issue.
56d1f2db3 | BUGFIX | [Fix] Fixed npu device not torch.device type issue.
fe9f1af62 | BUGFIX | Fixed the map_location parameter processing in torch.load.
c833b916c | BUGFIX | Fixed the map_location parameter processing in torch.load.
8615ed3b4 | BUGFIX | fixed e2603be from https://gitee.com/yangxiaorun1/pytorch/pulls/2107 perf: topk perf optimize
5b67a3746 | BUGFIX | Fix the problem of cnt can not be 0.
8fced2d8d | BUGFIX | Fix silu inplace backward.
ccb2200a1 | BUGFIX | Fix silu inplace backward.
bf96de52c | BUGFIX | Fix silu inplace backward.
b45567abd | BUGFIX | Fix silu inplace backward.
180377bc3 | BUGFIX | Fix the problem of cnt can not be 0.
81cd832b5 | BUGFIX | Fix the problem of cnt can not be 0.
6f147f05a | BUGFIX | Fix the problem of cnt can not be 0.
9edb9058f | BUGFIX | Fix the problem of cnt can not be 0.
b595c7afd | BUGFIX | Fix the fp16 precision problem of scatter.
4a314b171 | BUGFIX | Fix the compilation errors.
148f00563 | BUGFIX | Fix new_zeros and new_empty without args.
abe363894 | BUGFIX | fixed ac8cade from https://gitee.com/shaoyf/pytorch/pulls/2035 Fix new_zeros and new_empty without args.
68e6904aa | BUGFIX | Fix hardsigmoid_ and add corresponding cases.
500b50ff9 | BUGFIX | Fix hardsigmoid_ and add corresponding cases.
182ff768d | BUGFIX | Fix the fp16 precision problem of scatter.
ac8caded5 | BUGFIX | Fix new_zeros and new_empty without args.
4a079401b | BUGFIX | [fix] Reduce unnecessary item when the Tensor is on device
c837ed95f | BUGFIX | [bugfix]Set gate order of LSTM to ifjo, same as CPU's.
bde8b6534 | BUGFIX | Fix layer_norm shape error.
69c409266 | BUGFIX | Fix layer_norm shape error.
28be3708c | BUGFIX | Fix compilation errors when the version.py file exists.
4c32e388f | BUGFIX | Fix compilation errors when the version.py file exists.
0fa9e7353 | BUGFIX | [fix] hccl op supported data type check bug fix
bffaedbba | BUGFIX | Fix compilation errors when the version.py file exists.
4dd75516a | BUGFIX | Fix the error of median and fill_.
58f18b529 | BUGFIX | [BugFix] Support None for Tensor.npu.
ef3204b2f | BUGFIX | Fix the error of median and fill_.
8a18825d2 | BUGFIX | fixed 0b5ebb4 from https://gitee.com/shaoyf/pytorch/pulls/1950 [Fix] Modify the default device of as_tensor.
0b5ebb435 | BUGFIX | [Fix] Modify the default device of as_tensor.
ac490f798 | BUGFIX | Fix bug in indexing_opt
3fb810007 | BUGFIX | Fix the accuracy problem of nllloss2d.
4605b0db1 | BUGFIX | Fix the accuracy problem of nllloss2d.
905996ae5 | BUGFIX | Fix: modify error msg in HCCL op
28d2812eb | BUGFIX | Fix: modify error msg in HCCL op
75d123497 | BUGFIX | Fix bug in indexing_opt
9e27f353f | BUGFIX | Fix bug in indexing_opt
400f4c985 | BUGFIX | [Fix]Update support_wrap_ops.yaml.
394b9560c | BUGFIX | [Fix]Specified process processing in overflow detection tool.
89179e914 | BUGFIX | fixed ca5852a from https://gitee.com/shaoyf/pytorch/pulls/1913 Support npu tensor for PyTorch installed by wheel(x86_64)
57321fab5 | REVERT | Revert Index/IndexPut API.
98af8a627 | BUGFIX | [Fix]Only dump specified process in accuracy comparison tool.
0bb47aa17 | BUGFIX | [Fix]Patch nn.parallel._functions._get_stream to support npu.
856791449 | BUGFIX | Bugfix: Fix the precision error in allgather operator
41a9302ec | BUGFIX | Fix bug for checkpoint
795bff936 | REVERT | Revert Index/IndexPut Adaptive logic
3bf1ddde1 | REVERT | Revert Index/IndexPut adaptive logic.
4ef89b500 | BUGFIX | Bugfix: Fix the precision error in allgather operator
a03ac753d | BUGFIX | Fixed the issue that unique2 applied for empty tensor
ed40d2c20 | BUGFIX | bugfix:解决allgather计算输入为私有格式时的精度问题
06e197ed6 | BUGFIX | 修复unique2申请空tensor的问题
4df4b1d02 | BUGFIX | fixed 2df5874 from https://gitee.com/yangxiaorun1/pytorch/pulls/1502 feat: add memory log for pytorch
29c8a0f8a | BUGFIX | bugfix】修改maximum未做broadcast计算导致输出shape错误问题
0b9161dec | BUGFIX | 重定义empty_strided和empty的cpu实现&&修复reducer的bug
17c239a06 | BUGFIX | bugfix AsStridedKernelNpu
e557a670f | BUGFIX | bugfix:修复smooth_l1_loss_backward的beta
50fe7a9dc | BUGFIX | 修复smooth_l1_loss_backward的beta
e6dbc7511 | BUGFIX | 修复调用destroy_process_group接口未释放HCCL资源问题
d9497766b | BUGFIX | fix wrapper bug
f198c3954 | BUGFIX | BugFix x.npu(device=None)
c7ecb68fd | BUGFIX | fix div fifferent dtype
034943195 | BUGFIX | fix div different dtype bug
66e720dff | BUGFIX | bugfix of upsamplenearest2dkernel & optimize for graph mode
1cf96d4ef | BUGFIX | bugfix of upsamplenearest2dkernel
bd5d280d2 | BUGFIX | bugfix of upsamplenearest2dkernel & optimize for graph mode
c8aff7cb3 | BUGFIX | 修复DDP场景多进程场景创建文件失败问题
c5924abc1 | BUGFIX | fix bn3d format
a370d2c5e | BUGFIX | bugfix:删除不合理的copy_tensor_host_to_device
716cdd9d3 | BUGFIX | bugfix:masked_select的output shape对齐pytorch1.8
c75e3453b | BUGFIX | bugfix:删除不合理的copy_tensor_host_to_device
605bc33d1 | BUGFIX | fix cat null 3.0.0
79f97d6fa | BUGFIX | fix cat null
e29cdc0ce | BUGFIX | fix bn3d format
a5261b877 | BUGFIX | bugfix:修复conv调用sum_out的反向问题
84277629b | BUGFIX | bugfix:修复conv调用sum_out的反向问题
291e32ad7 | BUGFIX | bugfix:删除indexput中的异步拷贝
13e74f596 | BUGFIX | bugfix:删除indexput适配中的异步拷贝
510bf64e0 | BUGFIX | fixed c39619f from https://gitee.com/shaoyf/pytorch/pulls/1606 Bugfix Support npu:x format for Tensor.npu
c39619f3b | BUGFIX | Bugfix Support npu:x format for Tensor.npu
be54a1ae1 | BUGFIX | 修复mul在输入有scalar参数时(tensor * scalar)输出类型处理错误问题
cddeb21ae | BUGFIX | bugfix resize_impl_npu_
58a3cd26d | BUGFIX | bugfix:max支持不同dtype
97b5cc374 | BUGFIX | bugfix:max支持不同dtype输入
8fe250db9 | BUGFIX | 修复DDP场景多进程场景创建文件失败问题
254e2075a | BUGFIX | fix json.load bug
9c70fd645 | BUGFIX | 修复当dynamic=False时的参数GradScaler父类Cuda_GradScaler第337行的校验错误
c5f54b8e8 | REVERT | revert d6fb78 + a283f4 + 1af6a3
80852a385 | BUGFIX | API修正
ec151adbb | BUGFIX | API修正
f72dcf467 | BUGFIX | API修正
2daba0109 | BUGFIX | API修正
45963d632 | BUGFIX | API修正
fd2aead75 | BUGFIX | API修正
58611cfdc | BUGFIX | [fix] do not support NZ tensor when rank<2
2762fd58d | BUGFIX | [fix] do not support NZ tensor when rank<2
07ceee671 | BUGFIX | 修复byte输入时参数类型未对齐问题
13a03f75f | BUGFIX | [fix] scalar scene and rank=1 scene do not support NZ
cc1821587 | BUGFIX | bugfix:补充nllloss_2d对weight赋0的判断条件
212e4f00d | BUGFIX | fix: dvpp interface joint commissioning
15bb908e6 | BUGFIX | fixed e5f6878 from https://gitee.com/shaoyf/pytorch/pulls/1472 Change the dtype of num_batches_tracked in _NormBase from int64 to int32
1cf773731 | BUGFIX | [bugfix]cancel lstm npu_dtype_cast
5aa01f8f7 | BUGFIX | [bugfix]cancel lstm npu_dtype_cast
d44c3a69c | BUGFIX | [fix] bn use 5HD only when weight is 5hd
ea84a6622 | BUGFIX | fix: optimize cast op forward: use less npu memory to save cast dtype
00178dbab | BUGFIX | [Fix] use dynamicCompileswitch=enable to represent the case of binary
e679e0148 | BUGFIX | fix add of different dtype
c95df4bd0 | BUGFIX | 【BugFix】mul去除多余copy操作
434bce5cb | BUGFIX | fixed e369a14 from https://gitee.com/shaoyf/pytorch/pulls/1334 fix __setstate__ and clean code for wrong import order
f6b260955 | BUGFIX | 修复adaptive_avg_pool3d bug
c35da010d | BUGFIX | bugfix:去除mul中多余的copy操作
df4d3cd0a | BUGFIX | bugfix:去除mul中多余的copy操作
7842f91b4 | BUGFIX | 修复batchnorm、masked_select、upsample_linear1d和upsample_nearest1d
8e794d254 | BUGFIX | 【同步代码】index_put、maksed_select index_select等bug修复
5a41e76c8 | BUGFIX | 【同步代码】dump算子刷新、刷新cumprod、masked_scatter、upsample_nearest3d、upsample_trilinear3d、triu、scatter_add、修复task queue和修正hccl接口reduce标杆
b70ff1686 | BUGFIX | 修复算子ut调用方式、conv类算子适配方式等
0f30ce7c9 | BUGFIX | bugfix：torch.device for Module.to
520a201d8 | BUGFIX | bugfix：torch.device for Module.to
7cb2f3f61 | BUGFIX | fixed f296e85 from https://gitee.com/shaoyf/pytorch/pulls/1352 Clean code for wrong import order.
6f29462a9 | BUGFIX | fix index bug
22305f9a3 | BUGFIX | fix index bug
71f4a3b8a | BUGFIX | 修复当dynamic=False时的参数GradScaler父类Cuda_GradScaler第337行的校验错误
9fa200d68 | BUGFIX | fix index_add
9c14ce9fb | BUGFIX | fix bug index_add
a23d501c2 | BUGFIX | fix: Fix HCCL error does not have complete error message
1d88b003f | BUGFIX | fix: Fix HCCL error does not have complete error message
09ed1d1ec | BUGFIX | 修复正向计算result类型推断错误，导致的反向hook执行报错的问题
c378e36a4 | BUGFIX | 修复正向计算result类型推断错误，导致的反向hook执行报错的问题
59974ffb1 | BUGFIX | bugfix:_unique2算子fp16场景存在精度问题，转fp32规避
2f2105d11 | BUGFIX | bugfix:_unique2算子fp16场景存在精度问题，转fp32规避
6fa6a8272 | BUGFIX | [Bugfix] Support npu:x format for Module.to.
753c3f59e | BUGFIX | [Bugfix] Support npu:x format for Module.to.
4ea78ef83 | BUGFIX | fix: fix memory leak of task queue
dfd3398c6 | BUGFIX | bugfix:解决使用device6, device7无法barrier的问题 bugfix:解决使用device6, device7无法barrier的问题
9c73a875e | BUGFIX | bugfix:修正reduce的标杆
a3abdfeb0 | BUGFIX | bugfix:修正reduce的标杆
4618770dd | BUGFIX | [Bugfix] fix as_tensor operator.
a27ed0472 | BUGFIX | 修复avgpool2d 算子输出shape错误问题， 对不支持三维输入做增维操作
1c60a5418 | BUGFIX | fix new_zeros new_full new_ones new_tensor when device='npu'
7d20630d3 | BUGFIX | fix new_zeros new_full new_ones new_tensor when device='npu'
378c16678 | BUGFIX | fix bug in test_memcpy.py
78285a1d9 | BUGFIX | BugFix:修改rrelu_with_noise的输出format
14ffdaadc | BUGFIX | bugfix:修复logical_not的checkout中的format
bad33087f | BUGFIX | bugfix:reduce纠正输入numel
740309cf9 | BUGFIX | BugFix:upsample_bilinear2d删除过适配操作
5744a20a5 | BUGFIX | fix random default to
8cc17f0f5 | BUGFIX | [fixbug]修复silu_backward调用SwishGrad算子
9dd29b3ea | BUGFIX | [bugfix]修复pad_packed_sequence算子中npu_transpose对输入维度的限制
8960e7ba2 | BUGFIX | [fixbug]修复CdistGrad算子inf场景
00300f658 | BUGFIX | fix the bug of repeated statements
7087fa22c | BUGFIX | fix the bug of repeated statements
989aaa706 | BUGFIX | fix ut case issue, which format is not matched with shape
1265f8a72 | BUGFIX | [bugfix] Not support exchange device.
5b0cfc561 | BUGFIX | fix as_strided acc
6871ad926 | BUGFIX | fix random and unifrom
3306101e6 | BUGFIX | 网络模型移植与训练指南全文问题修改&环境变量代码缺少空格
a4ebef8fe | BUGFIX | fix bug of syncbn_forward
01dc029f6 | BUGFIX | fix as_strided bug of recompile
933c0f2d6 | BUGFIX | fix two c10d barrier bugs
7d8b95f64 | BUGFIX | 修复OneHot算子输入scalar,导致模型性能下降问题
1a935d0e9 | BUGFIX | fix build issue, which get size failed
950be4667 | BUGFIX | fix build issue, which use wrong item
39242292c | BUGFIX | fix build issue
015614af6 | BUGFIX | fix bug in view for graph mode
be5873c2a | BUGFIX | fix profiler return bug
eaf3cd42c | BUGFIX | bug fix: two tensor share a storage, then use hccl to communicate
fccd335bf | BUGFIX | [fixbug] 修复one_hot算子走aicpu分支报错场景
d29d8ee92 | BUGFIX | fix oom when calling npu_transpose
0a10966ac | BUGFIX | fix oom when calling npu_transpose
e86c20210 | BUGFIX | fix cast weight for tensors with different devices
4b185a10f | BUGFIX | fix cast weight for tensors with different devices
b57324fa0 | BUGFIX | fix torch.device('npu').index=None
ecbfb8761 | BUGFIX | fix torch.device('npu').index=None
a5b2c332f | BUGFIX | fix bug index_put
68d1c05dd | BUGFIX | bugfix: support creating npu tensor using torch.as_tensor
5e99b4ec4 | BUGFIX | bugfix: support creating npu tensor using torch.as_tensor
8ae3d43be | BUGFIX | [Bugfix] Fix new empty error.
027f6f626 | BUGFIX | 【bugfix】 修复cdist算子，属性p=inf,npu报错
2f17fe4e0 | BUGFIX | fix index_put
6a35c276a | BUGFIX | fix model performance declining problems caused by the 5HD to 4D format conversion，
8420ab4c8 | REVERT | Revert to deepcopy.
ca2567ca6 | BUGFIX | 修复图模式下 nbytes() 问题 & 算子适配 1.8
1022bc44c | BUGFIX | bugfix】 修复cdist算子，属性p=inf,npu报错
54e803545 | BUGFIX | 【bugfix】修复all算子输入为空Tensor，计算结果错误场景
10bbdb0b7 | BUGFIX | fix new_empty error.
9f3d065e9 | BUGFIX | fix torch.device.type & parse torch.device
537c69084 | BUGFIX | bugfix:修复divide算子input为int32类型的精度问题
90709ed4f | BUGFIX | bugfix:修复floor_divide算子input为int32格式的精度问题
52fc7825c | BUGFIX | Fix hccl stub code.
d0184a573 | BUGFIX | fix device init
978896426 | BUGFIX | 修复device参数解析导致的out接口错误
cd86ed8fe | BUGFIX | fix graph executor and constructor & modify graph_mode
3b7a1886a | BUGFIX | 修复**_like类型算子在指定format时device解析错误
853fed5ef | BUGFIX | [Bugfix] Right lock for TaskQueue.
ee34cf3cc | BUGFIX | fix ddp and is_npu
78bc569b9 | BUGFIX | [Bugfix] Missed compiling for Generator.
659c6f794 | BUGFIX | fix cann prof api bug
800aaec50 | BUGFIX | [Bugfix] Add HalfTensor.
10c6a8585 | BUGFIX | [Bugfix] Lt support LongTensor.
a31db9389 | BUGFIX | fix init npu bug
bddd26b75 | BUGFIX | fix init index !=0 bug
d0094263d | BUGFIX | fix tensor.to(index) bug
157c98eb1 | BUGFIX | fix build bug
a6bcf346c | BUGFIX | fix code check
5c5a1aeae | BUGFIX | fix tensor.npu(index) bug
9cc3aab6c | BUGFIX | fix codecheck
a96623a5e | BUGFIX | fix randperm for equals to operator prototype
35fc19e4d | BUGFIX | fix scalar_tensor op
9d520d824 | BUGFIX | fix InnerRun blocking caused by GIL
28b116fe9 | BUGFIX | [Bugfix] Avoid 5HD Error.
8c5338ce2 | BUGFIX | [Bugfix] eval mode changes the requires_grad attr.
920bafb35 | BUGFIX | [Bugfix] Fix storage change while cast module to cpu.
e910896bc | BUGFIX | [bugfix] support list type for get_device_index.
c5e60f832 | BUGFIX | [bugfix] missed keys for torch serialization.
2b4ea84e2 | BUGFIX | 【BugFix】mul、logical_and与1.5对齐
1d9e3ffb6 | BUGFIX | [bugfix]fix tensor with dimensions of 0 or shape = [1]
1b48c38c3 | BUGFIX | Bugfix for mseloss.
a60f2a260 | BUGFIX | Bugfix for mv ops without contiguous input.
d03a26c95 | BUGFIX | Fix 5hd error for softmax.
e78fe9ef5 | BUGFIX | Fix NLLLoss.
a15547288 | BUGFIX | bugfix:更改lstm_backward用例
d6274b4c8 | BUGFIX | bugfix:take算子ApplyTensor使用错误
f162523c2 | BUGFIX | fix atan2
7c31404fc | BUGFIX | [bugfix] fix offset of astrided to be 0
f54d20ac1 | BUGFIX | [Bugfix] Add strict check for parameters.
ec91fd265 | BUGFIX | [Bugfix] Copy save obj to avoid changes to mutable obj.
972d02ed7 | BUGFIX | fix elu
ca685b4c3 | BUGFIX | fix get_device_index error.
23b050f6e | BUGFIX | fix bug for e2e error in dynamic shape
8850cdb8d | BUGFIX | fix test_grid_assign_positive.py ut 问题
6b35e4b88 | BUGFIX | [Bugfix] change to physical_numel.
698226843 | BUGFIX | fix baddbmm bug
e98a02cf8 | BUGFIX | 修复dropout_with_add_softmax导致内存泄露的问题
af6f2cbbe | BUGFIX | [Bugfix] Fix the computation of bucket size & broadcast_coalesced.
4342e86fa | BUGFIX | Fix import error of release_process_group.
3aff14fda | BUGFIX | fix test_transpose.py ut
557b9dc9c | BUGFIX | 修复batch_nms问题
a3ca88990 | BUGFIX | fix bug of aoe dir
e37609fdc | BUGFIX | [bugfix] BNTrainingUpdate only support fp32 input
c9806c139 | BUGFIX | fix bug upsample
72da3c847 | BUGFIX | 修复addcdiv算子精度问题 刷新api资料
b857397ec | BUGFIX | Fix codecheck for testcases.
ffc44d534 | BUGFIX | 修复codeclean问题
7799ad7f9 | BUGFIX | fix_bug
d5e5911a7 | BUGFIX | Fix NormKernel ops.
3ff0bd3a8 | BUGFIX | Fix codecheck for yaml & open.
c88c9e303 | BUGFIX | fix bug of inverse out fp16
127244b33 | BUGFIX | 修复 BinaryCrossEntropyWithLogitsBackward算子 精度问题
945b115c0 | BUGFIX | [bugfix] fix assertRtolEqual.
ee96aab88 | BUGFIX | Fix huge depth & huge method.
077d05f25 | BUGFIX | Fix codecheck for test_npu/c10d & c10d.
4c2d860ae | BUGFIX | 修复资料拼写错误
346e2b4d1 | BUGFIX | bugfix truedivide Operator
630686cec | BUGFIX | Fix codecheck for codegen & testing & ci.
74bf07651 | BUGFIX | Fix Embedding Kernel.
17f8cece9 | BUGFIX | 修复one_ bug, 移除empty_strided 单算子问题
f7ba314b3 | BUGFIX | fix test_embedding_bag_backward
a506480a3 | BUGFIX | Fix codecheck for codegen api.
e33763bce | BUGFIX | 修复scatter算子src数据类型不匹配的问题
cb17f984b | BUGFIX | fix h2d error.
c24bd882b | BUGFIX | 修复grid_sampler_2d算子精度问题
e8018c5cb | BUGFIX | fix_codesmell
229a73b32 | BUGFIX | Fix to ops.
436d42247 | BUGFIX | bugfix: asstride do not support unmatched tensors input
5cb59101e | BUGFIX | 修复save bug
bc754be07 | BUGFIX | fix roll Operator
f5801a3c3 | BUGFIX | fix pin_memory & device error.
c874a1049 | BUGFIX | fix bug of conv3d
573645c2d | BUGFIX | fix cast_weight error.
101edaa28 | BUGFIX | add bugfix Operator
b83f8cbe5 | BUGFIX | 修复grid_sampler算子问题，算子现支持正反向调用
6112da8da | BUGFIX | fix SortWithoutIndices Operator
9910354a6 | BUGFIX | fix bugs of floor, sort, upsample_bilinear2d, upsample_bilinear2d_backward, upsample_nearest2d, upsample_nearest2d_backward
68c95af3c | BUGFIX | Add YoloBugfix Operator
055cf8dc5 | BUGFIX | 修复profiler功能record_function功能
82ac4a5e6 | BUGFIX | fix ut of format_cast
e825d02dd | BUGFIX | fix ddp forward.
b372e5fd2 | BUGFIX | fix ut of gelu_backward
36428b7e7 | BUGFIX | fix bug of atan, ceil, celu, argsort, cdist, celu
9c0ab7ef8 | BUGFIX | Fix apply patch.
f3fa9d44f | BUGFIX | Fix codecheck.
3c03ae440 | BUGFIX | fix NPUEvent.
6babefead | BUGFIX | 修复检查意见
762f2a273 | BUGFIX | topk算子问题修复与重构
70603c3e6 | BUGFIX | Fix LayerNorm(recursive init).
88dcc7576 | BUGFIX | fix layer_norm
cfbe896ea | BUGFIX | Fix Cyclomatic Complexity.
083ffb938 | BUGFIX | Fix codecheck.
c374454b4 | BUGFIX | Fix modification for const vector.
cbdf278b3 | BUGFIX | fix bug of where
83bb19062 | BUGFIX | Fix calling custom ops.
48e7068db | BUGFIX | fix index, random, where
388945c11 | BUGFIX | fix bugs
4b771736c | BUGFIX | 修复英文资料的路径和拼写错误
6e3d10447 | BUGFIX | 修复英文资料的路径和拼写错误
193772bbe | BUGFIX | 修复拼写错误
abe5f8f9b | BUGFIX | 修复拼写错误，增加路径说明
e43f2fa4f | BUGFIX | fix native func header.
ec5fb68ae | BUGFIX | fix bug and add UT
a933102a6 | BUGFIX | Fix path of default ut file.
91e80eda1 | BUGFIX | fix error
8c5dd060e | BUGFIX | fix erro
1fcf3a003 | BUGFIX | !45 修复中文文档链接不跳转的问题 * 修改链接显示问题 * 修改链接跳转问题 * 删除链接跳转中的括号 * 处理括号使链接跳转失败的问题 * 修改内部文件链接跳转问题 * 修改 分布式训练修改 链接跳转问题 * 修改链接跳转失败问题 * 修改链接显示问题 * 使目录连接生效 * solve the problem of anchor link - installation guide
8fa79690b | BUGFIX | fix env.sh ASCEND typo

<!-- END INDEX -->
