# Validation Report: Coding Conventions & Review Standards Cross-Validation

Generated: 2026-04-13 (iteration 2, supersedes iteration 1)
Source documents:
- coding-conventions.md (39 rules, Ch.1-9)
- review-standards.md (44 checkpoints, P0-P3 + Quick Reference)
Method: apply ALL rules to 5 real commits, classify each as HIT/MISS/UNACTIONABLE/CONTRADICTORY


## 1. Selected Commits

| # | Hash      | Type     | Area         | Summary                              |
|--:|:----------|:---------|:-------------|:-------------------------------------|
| 1 | 34d76cbf3 | FEATURE  | distributed  | 解耦子comm派生，rootinfo本地派生子comm |
| 2 | f83b2829d | BUGFIX   | core/runtime | fix task queue aclgraph multi-stream |
| 3 | 5a798156e | BUGFIX   | aten/ops     | clone support uncontiguous           |
| 4 | bcec360bb | REFACTOR | build        | remove needed of libhccl.so          |
| 5 | 2e5b8909c | BUGFIX   | core/device  | GetDeviceInfo do not support A5      |

Selection: covers distributed/core/aten/build areas; #3 relates to P1-1 (D-9),
#5 relates to P2-4 (D-8).


## 2. Per-Commit Review Results

### 2.1 Commit 34d76cbf3 (FEATURE distributed)

Files: ProcessGroupHCCL.cpp (+63-30), ProcessGroupHCCL.hpp (+9)

| Rule      | Verdict       | Evidence                                          |
|:----------|:--------------|:--------------------------------------------------|
| Conv 1.3  | HIT           | `createHCCLCommSub` follows camelCase             |
| Conv 4.1  | HIT           | `OptionalNPUGuard` RAII for device context        |
| Conv 6.1  | HIT           | `TORCH_NPU_HCCL_LOGI` for lifecycle logging      |
| Conv 9.6  | HIT           | ProcessGroupHCCL = #1 revert-prone hotspot        |
| Conv 9.7  | HIT (partial) | Sub method casts `count()` to int; global method still has `%d` with `long long` on moved line |
| RS P1-2   | HIT           | Dispatch: `isSubComm` bool, Sub->fail->Origin     |
| RS P3-1   | HIT           | Naming consistent, no near-duplicates             |
| Conv 2.2  | UNACTIONABLE  | Include ordering: 60% consistency, no enforcement |
| RS P0-1.1 | UNACTIONABLE  | Lock ordering: infeasible from diff review        |
| (gap)     | MISS          | No regression test; no rule requires it for any commit type |

Key: Conv 9.7 caught a real inconsistency -- `static_cast<int>(subTimeElapsed.count())` fixed
in new method, but `timeElapsed.count()` unfixed in the moved global-comm log line.


### 2.2 Commit f83b2829d (BUGFIX aclgraph)

Files: NPUCachingAllocator.cpp (+12), .h (+7), NPUEvent.cpp (+4),
test_aclgraph_multi_stream.py (+135 new)

| Rule      | Verdict      | Evidence                                          |
|:----------|:-------------|:--------------------------------------------------|
| Conv 1.3  | HIT          | `hasCapturesUnderway` camelCase                   |
| Conv 7.1  | HIT          | `test_aclgraph_multi_stream.py` naming            |
| Conv 7.2  | HIT          | Inherits `TestCase` from torch_npu.testing        |
| Conv 7.3  | HIT          | `@SupportedDevices(['Ascend910B', 'Ascend910_93'])`|
| Conv 7.4  | HIT          | `assertRtolEqual` for numerical comparison        |
| Conv 7.5  | HIT          | Uses `"npu:0"` string literal                     |
| Conv 7.6  | HIT (caught) | Module-level `os.environ[...]=...` line 8, never restored |
| RS P0-1   | HIT          | `lock_guard<recursive_mutex>` in new method       |
| RS P2-1   | HIT          | Virtual + override + inline wrapper complete      |
| RS P2-4   | HIT          | `assertValidDevice(device)` edge case             |
| (gap)     | MISS         | `virtual bool hasCapturesUnderway() { return false; }` -- no rule on safe vs dangerous silent defaults |

Key: Conv 7.6 is the star rule -- correctly identifies env var pollution pattern.
The test sets `PYTORCH_NPU_ALLOC_CONF` at module scope without cleanup.


### 2.3 Commit 5a798156e (BUGFIX clone)

Files: CloneKernelOpApi.cpp (+9-4)

| Rule      | Verdict       | Evidence                                          |
|:----------|:--------------|:--------------------------------------------------|
| Conv 5.3  | HIT           | Fixes D-9: clone ignored Preserve format          |
| RS P1-1.1 | HIT (partial) | Preserve fixed via `empty_strided_symint`; ChannelsLast still falls through to `apply_tensor_without_format` |
| RS P1-1.5 | HIT           | "Does a unit test compare?" -- no test added      |
| (gap)     | MISS          | No rule mandates regression test for bug fixes    |

Key: RS P1-1 checkpoint 1 precisely identified the fix AND its incompleteness.
The note "(commit 5a798156e fixed Preserve but missed explicit formats)" is
confirmed by the diff. `apply_tensor_without_format(src)` at CloneKernelOpApi.cpp:33
ignores format entirely; PyTorch native uses `at::empty_like(src, options, memory_format)`.


### 2.4 Commit bcec360bb (REFACTOR build)

Files: setup.py (+11-3)

| Rule      | Verdict      | Evidence                                          |
|:----------|:-------------|:--------------------------------------------------|
| RS P2-5.1 | HIT          | Modifies link stripping, adds `_C.cpython*.so`   |
| RS P2-5.2 | HIT          | `--remove-needed` strips DT_NEEDED; GET_FUNC dlopen intact |
| RS P2-5.3 | HIT          | `glob('_C.cpython*.so')` specific enough          |
| RS P3-1   | HIT          | Variable names: `lib_files`, `c_files`, `all_files`|
| RS P2-5.1 | UNACTIONABLE | "verify on clean environment" -- no concrete command specified |
| RS P2-5.4 | UNACTIONABLE | Cross-platform: patchelf inherently Linux-only    |

Key: P2-5 (added in iteration 1) proved immediately useful -- all 4 applicable checkpoints
provide actionable guidance. Checkpoint 1's verification method needs concretization.


### 2.5 Commit 2e5b8909c (BUGFIX GetDeviceInfo A5)

Files: Module.cpp (+1-1)

| Rule      | Verdict | Evidence                                              |
|:----------|:--------|:------------------------------------------------------|
| Conv 9.3  | HIT     | SoC compat check added -- exact documented pattern    |
| RS P2-4.2 | HIT     | Guard present; design guidance flags `< Ascend950` as suboptimal vs `!= Ascend950` |
| (gap)     | MISS    | No regression test; no rule requires it               |

Key: RS P2-4.2 design guidance is precisely calibrated. Recommends point exclusion
over range exclusion -- correct judgment-aiding guidance.


## 3. Coverage Statistics

| Metric        | Count | Pct   |
|:--------------|------:|------:|
| HIT           |    26 | 76.5% |
| MISS          |     4 | 11.8% |
| UNACTIONABLE  |     4 | 11.8% |
| CONTRADICTORY |     0 |  0.0% |
| Total applied |    34 |       |

Effective coverage (HIT / applicable, excluding UNACTIONABLE): 26/30 = 86.7%


## 4. Gap Analysis

### Gaps found in iteration 1 (already fixed)

These were identified and patched in the previous iteration:

| ID   | Gap                                   | Fix applied                     |
|:-----|:--------------------------------------|:--------------------------------|
| G1-1 | Printf format safety for log macros  | Added Conv 9.7                  |
| G1-2 | Test env var isolation                | Added Conv 7.6                  |
| G1-3 | MemoryFormat beyond Preserve          | Expanded RS P1-1.1              |
| G1-4 | Build/packaging review coverage       | Added RS P2-5                   |
| G1-5 | SoC guard design guidance             | Expanded RS P2-4.2              |

### New gaps found in iteration 2

| ID   | Type         | Description                                      | Commits |
|:-----|:-------------|:-------------------------------------------------|:--------|
| G2-1 | MISS         | No rule requires regression tests for bug fixes  | 3,5 (+ 1 as FEATURE) |
| G2-2 | MISS         | No guidance on virtual function default safety   | 2       |
| G2-3 | UNACTIONABLE | P2-5.1 "verify on clean env" no specific command | 4       |
| G2-4 | (cross-ref)  | Conv 5.3 and RS P1-1 cover same domain, no link  | 3       |
| G2-5 | (cross-ref)  | RS P2-4.2 and Conv 9.3 cover same domain, no link| 5       |

G2-1 detail: 3 of 5 commits are bug fixes. Commits 5a798156e (clone) and
2e5b8909c (GetDeviceInfo) fix real bugs without regression tests. Commit
f83b2829d (aclgraph) does include tests -- but there is no rule that mandates
this. The gap is systemic: testing conventions (Ch.7) describe HOW to write
tests but not WHEN they are required.

G2-2 detail: `virtual bool hasCapturesUnderway() { return false; }` in
NPUCachingAllocator.h:261 -- the base class default silently returns false.
If a future allocator subclass uses graph captures but does not override this,
it silently misreports capture state. No convention or review standard addresses
the safety of "silent no-op" virtual defaults.

G2-3 detail: RS P2-5 checkpoint 1 says "verify on a clean environment"
but does not specify HOW. A concrete minimum gate (e.g.
`python -c "import torch_npu"`) would make this actionable.


## 5. Edits Made to Upstream Artifacts

### 5.1 Iteration 1 edits (already applied, verified correct)

coding-conventions.md:
- Conv 9.7: Printf format specifier safety (validated by commit 1)
- Conv 7.6: Test env var isolation (validated by commit 2)
- Conv 1.3: PascalCase vs camelCase decision guide (validated by commit 2)

review-standards.md:
- RS P1-1.1: Expanded to cover all MemoryFormat values (validated by commit 3)
- RS P2-4.2: SoC guard granularity guidance (validated by commit 5)
- RS P2-5: Build & Packaging section (validated by commit 4)

### 5.2 Iteration 2 edits (applied in this iteration)

coding-conventions.md:

| # | Section | Change                                          |
|--:|:--------|:------------------------------------------------|
| 1 | 7.7     | Added: Bug Fix Regression Test rule             |
| 2 | 9.8     | Added: Virtual Function Default Implementation  |
| 3 | 5.3     | Added: cross-reference to RS P1-1               |

review-standards.md:

| # | Section         | Change                                     |
|--:|:----------------|:-------------------------------------------|
| 4 | P2-5            | Added checkpoint 5: import smoke test      |
| 5 | Appendix C (P1) | Added: bug fix regression test checklist   |
| 6 | P2-4.2          | Added: cross-reference to Conv 9.3         |


## 6. Cross-Document Consistency Check

| Convention         | Review Standard  | Status                          |
|:-------------------|:-----------------|:--------------------------------|
| Conv 9.3 SoC compat| RS P2-4.2       | Consistent; cross-ref added     |
| Conv 5.3 op semantic| RS P1-1         | Consistent; cross-ref added     |
| Conv 3.6 bare catch| RS P1-3          | Consistent                      |
| Conv 4.1 RAII guard| RS P0-2          | Consistent                      |
| Conv 7.6 env isolation| (none)        | Gap: RS lacks test quality checkpoint. Partial fix: added regression test to Appendix C but env isolation remains Conv-only |
| Conv 7.7 regression test| RS Appendix C| Consistent; both now require it |

Result: 0 contradictions. 1 remaining asymmetry (env isolation rule in Conv
but not in RS). Acceptable because test quality is more convention than
review-gate material.


## 7. Rule Effectiveness Ranking

Rules providing most value across this 5-commit sample:

| Rank | Rule     | Hits | Value                                    |
|-----:|:---------|-----:|:-----------------------------------------|
|    1 | RS P1-1  |    2 | Caught partial fix + missing path        |
|    2 | RS P2-5  |    4 | New rule, all checkpoints useful         |
|    3 | Conv 7.6 |    1 | Caught env var pollution                 |
|    4 | Conv 9.7 |    1 | Caught printf format inconsistency       |
|    5 | RS P2-4  |    2 | Caught suboptimal guard + granularity    |

Rules with zero hits (not tested in this sample, not necessarily low-value):
P0-2 (resource leak), P2-2 (upstream sync), P2-3 (monkey-patch).
These address real defect clusters; they did not fire on this 5-commit sample.
