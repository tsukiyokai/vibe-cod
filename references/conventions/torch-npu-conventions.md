# torch_npu Coding Conventions

Repository: `torch_npu` (Ascend PyTorch Adapter)
Branch: v2.9.0-26.0.0
Generated: 2026-04-12
Evidence standard: every rule has file:line or grep count; unverified claims marked "unconfirmed inference"

## Methodology

Sources:
- Static analysis: grep/glob counts across torch_npu/csrc/ (354 C++ files) and test/ (622 test files)
- Upstream digests: architecture-map.md, design-patterns.md, git-history.md, defect-analysis.md
- Git history: 28,222 commits, 3,401 BUGFIX + 101 REVERT entries

Strength classification:
- mandatory: 100% consistent across measured scope (zero counter-examples found)
- recommended: >80% consistent; deviations exist but are minority
- observed: 50-80% consistent; clear dominant pattern but significant exceptions
- optional: <50% or no enforcement visible

---

## Ch.1 Naming Conventions

### 1.1 File Naming: PascalCase for C++ Sources

- Scope: repo-wide (csrc/)
- Strength: recommended (73%)
- Rule: C++ header and source files use PascalCase naming (e.g. `NPUStream.cpp`, `OpCommand.h`).
  Exceptions exist in inductor/ and toolkit/ modules where upstream PyTorch code uses snake_case.

Frequency:

    Pattern      .h files    .cpp files    Total     Pct
    PascalCase     118          143          261     73.7%
    snake_case      53           40           93     26.3%

Good example: `torch_npu/csrc/core/npu/NPUStream.h` -- PascalCase, matches class name NPUStream
Good example: `torch_npu/csrc/framework/OpCommand.h` -- PascalCase, matches class OpCommand
Exception: `torch_npu/csrc/inductor/array_ref_impl.h` -- snake_case, follows PyTorch inductor upstream style

Module breakdown:

    Module              PascalCase    snake_case    Notes
    core/                    ~90          ~5        Near-mandatory
    distributed/             ~40          ~2        Near-mandatory
    framework/               ~60          ~9        Dominant
    aten/                    ~47          ~0        Mandatory (codegen-driven)
    inductor/                 ~3         ~28        snake_case dominant (upstream)
    profiler/                ~10         ~12        Mixed
    toolkit/                  ~2          ~7        snake_case dominant


### 1.2 Class Naming: PascalCase

- Scope: repo-wide
- Strength: mandatory (100%)
- Rule: All class and struct names use PascalCase. Acronyms are uppercased as a block
  (NPU, HCCL, ACL, DVM).

Frequency: 286/286 sampled class definitions follow PascalCase.

Good examples:
- `struct NPUGuard` -- torch_npu/csrc/core/npu/NPUGuard.h:20
- `class ProcessGroupHCCL` -- torch_npu/csrc/distributed/ProcessGroupHCCL.hpp:280 (inferred from design-patterns)
- `class OpCommand` -- torch_npu/csrc/framework/OpCommand.h:27 (inferred from design-patterns)
- `struct NPUStorageDesc` -- torch_npu/csrc/core/NPUStorageImpl.h:16 (from architecture-map)

Acronym convention: NPU (not Npu), HCCL (not Hccl), ACL (not Acl), DVM (not Dvm).
This matches Ascend SDK naming (ACL = Ascend Computing Language, HCCL = Huawei Collective
Communication Library).


### 1.3 Function Naming: camelCase (Internal), PascalCase (Public API)

- Scope: repo-wide
- Strength: observed (~70% camelCase, ~30% PascalCase)
- Rule: Internal methods and free functions use camelCase. Public API methods that mirror
  ACL library style use PascalCase.

camelCase (internal, ~70%):
- `getStreamFromPool()` -- torch_npu/csrc/core/npu/NPUStream.h:117
- `getCurrentNPUStream()` -- torch_npu/csrc/core/npu/NPUStream.h:125
- `synchronize()` -- torch_npu/csrc/core/npu/NPUStream.h:77
- `parseFilterFromEnv()` -- torch_npu/csrc/logging/LogContext.h:26

PascalCase (public API, ~30%):
- `PushToReleaseQueue()` -- torch_npu/csrc/core/npu/NPUQueue.h:44
- `GetInstance()` -- singleton accessor, 193 call sites across 53 files
- `Start()`, `Stop()`, `Run()` -- lifecycle methods

PyTorch-inherited snake_case (exception):
- `device_index()` -- torch_npu/csrc/core/npu/NPUStream.h:58
- `original_device()` -- torch_npu/csrc/core/npu/NPUGuard.h:61
- `current_device()` -- torch_npu/csrc/core/npu/NPUGuard.h:68

Design rationale: PascalCase public methods align with ACL SDK conventions (e.g. `aclrtMalloc`,
`aclrtSetDevice`). PyTorch interface methods follow PyTorch's snake_case convention. Internal
helper methods use camelCase.

Decision guide for new methods:
- Lifecycle/factory/singleton verbs (Create, Init, Release, Get, Start, Stop): PascalCase
- PyTorch virtual interface overrides: snake_case (match upstream signature)
- All other public and private methods: camelCase (dominant 70% pattern)


### 1.4 Namespace Naming: lowercase with Underscores

- Scope: repo-wide
- Strength: mandatory (100% for top-level namespaces)
- Rule: All top-level namespaces use lowercase with underscores.

    Namespace               Modules covered                Usage
    c10_npu::               core/npu/ (device/stream/mem)  Primary device layer
    at_npu::native          aten/, framework/              Operator implementations
    torch_npu::             core/, csrc root               Storage/Tensor/Bridge
    torch_npu::profiler     profiler/                      Performance analysis
    torch_npu::toolkit      toolkit/                       npu_profiler shared lib
    c10d_npu::              distributed/                   Collective communication
    npu_logging::           logging/                       Log infrastructure

Exception: nested namespaces like `NPUCachingAllocator`, `NPUSwappedMemoryAllocator` use
PascalCase. These are conceptually class-scoped namespaces, not library namespaces.

Source: architecture-map.md Ch.6.1, verified by grep.


### 1.5 Member Variable Naming: Trailing Underscore for Private Members

- Scope: repo-wide
- Strength: recommended (>80%)
- Rule: Private member variables use trailing underscore suffix. Public members omit it.

Good examples:
- `guard_` (private) -- torch_npu/csrc/core/npu/NPUGuard.h:75
- `stream_` (private) -- torch_npu/csrc/core/npu/NPUStream.h:115 (from design-patterns)
- `mutex_` (private) -- frequent across Guard/Allocator classes

Exception: CachingAllocatorConfig uses `m_` prefix (Hungarian notation):
- `m_max_split_size` -- torch_npu/csrc/core/npu/NPUAllocatorConfig.h:85
- `m_garbage_collection_threshold` -- torch_npu/csrc/core/npu/NPUAllocatorConfig.h:86

Note: the `m_` prefix appears concentrated in NPUAllocatorConfig, likely inherited from
upstream PyTorch CUDACachingAllocator. The trailing underscore is the native convention.


### 1.6 Macro Naming: UPPER_SNAKE_CASE

- Scope: repo-wide
- Strength: mandatory (100%)
- Rule: All preprocessor macros use UPPER_SNAKE_CASE.

Good examples:
- `NPU_CHECK_ERROR` -- torch_npu/csrc/core/npu/NPUException.h:212
- `ASCEND_LOGE` -- torch_npu/csrc/core/npu/npu_log.h:24
- `C10_NPU_SHOW_ERR_MSG` -- torch_npu/csrc/core/npu/NPUException.h:25
- `PTA_ERROR` -- torch_npu/csrc/core/npu/NPUException.h:89
- `C10_NPU_API` -- torch_npu/csrc/core/npu/NPUMacros.h (visibility macro)

No counter-examples found across 354 C++ files.


### 1.7 Enum Naming: PascalCase Types, UPPER_SNAKE_CASE Values

- Scope: repo-wide
- Strength: recommended (85% UPPER_SNAKE_CASE values)
- Rule: Enum type names use PascalCase. Enum values use UPPER_SNAKE_CASE. Prefer `enum class`
  over unscoped `enum`.

Good examples:
- `enum class SubModule { PTA = 0, OPS = 1, DIST = 2 }` -- torch_npu/csrc/core/npu/NPUException.h:60
- `enum class ErrCode { SUC = 0, PARAM = 1, ... ACL = 100, HCCL = 200 }` -- NPUException.h:68
- `enum RepoStatus { INIT, RUN, NEED_EXIT, ... UCE_EXIT }` -- torch_npu/csrc/core/npu/NPUQueue.h:19

Exception: k-prefix camelCase style in some newer code:
- `enum class Command { kStartOne, kStartAll, kStop }` -- profiler module (from design-patterns)

---

## Ch.2 File Organization

### 2.1 Header Guard: Prefer #pragma once

- Scope: repo-wide (csrc/)
- Strength: recommended (63%)
- Rule: New header files should use `#pragma once`. Legacy `#ifndef` guards coexist
  but are not the preferred style.

Frequency:

    Style            Files    Pct
    #pragma once       109    63.4%
    #ifndef guards      63    36.6%

Module distribution:
- core/, framework/, aten/: mixed (both styles present)
- inductor/: predominantly `#pragma once`
- profiler/: mixed

`#ifndef` guard naming convention (when used):
- `__PLUGIN_NATIVE_UTILS_OP_COMMAND__` style (double underscore prefix+suffix)
- `__PLUGIN_NATIVE_NPU_COMMON_FORMAT_CAST_HELPER__`

Note: double underscore identifiers are technically reserved by the C++ standard (UB).
The `#pragma once` migration avoids this issue.


### 2.2 Include Ordering: System, Library, Project (Not Strictly Enforced)

- Scope: repo-wide
- Strength: observed (~60% follow the pattern)
- Rule: The dominant pattern is system headers, then library (ATen/c10/torch),
  then project headers. But enforcement is weak and ordering varies by module.

Good example (system -> library -> project):
- torch_npu/csrc/distributed/ProcessGroupHCCL.cpp:1-40 (from file-org agent):
  system headers (`<chrono>`, `<thread>`) -> PyBind11 conditional -> third-party ACL -> project headers

Counter-example:
- torch_npu/csrc/core/npu/NPUEvent.cpp:1-14: project first, system last

The lack of a clang-format or include-what-you-use enforcement means this is a guideline,
not a mandatory rule.


### 2.3 Source-Header Pairing

- Scope: repo-wide (csrc/)
- Strength: observed (68% paired)
- Rule: Most .cpp files have a corresponding .h file with the same base name.

Frequency:

    Category                Count    Pct
    Matched pairs            ~125    ~68%
    Orphan .cpp (no .h)       58     32%
    Orphan .h (no .cpp)       46     27%

Orphan .cpp files are typically implementation-only files (framework optimizations, distributed
communication workers). Orphan .h files are header-only or pure interface definitions (inductor
runtime, framework interfaces).


### 2.4 Copyright Headers: Not Standardized

- Scope: repo-wide
- Strength: optional (4.5%)
- Rule: Only 16/354 C++ files (4.5%) have copyright headers. No standard template is enforced.

When present, format is:
- `// Copyright (c) 2025 Huawei Technologies Co., Ltd`
- `// Licensed under the BSD 3-Clause License`

---

## Ch.3 Error Handling

### 3.1 NPU_CHECK_ERROR for ACL Runtime Calls

- Scope: repo-wide
- Strength: mandatory for ACL call sites
- Rule: Every ACL runtime API call (`aclrt*`) must be wrapped with `NPU_CHECK_ERROR`.
  This macro checks the return code, detects UCE (Uncorrectable Error) via
  `AclrtPeekAtLastError`, and throws a c10::Error with formatted error message.

Definition: torch_npu/csrc/core/npu/NPUException.h:212

    #define NPU_CHECK_ERROR(err_code, ...) \
        NPU_CHECK_ERROR_CHECK_UCE(err_code, true, ##__VA_ARGS__)

The base macro `NPU_CHECK_ERROR_CHECK_UCE` (NPUException.h:142-209) handles:
1. UCE detection via `AclrtPeekAtLastError(ACL_RT_THREAD_LEVEL)`
2. Error code mapping via static `AclErrorCode` lookup table
3. OOM detection with `isCannOOM()` -> `OutOfMemoryError`
4. Feature-not-supported warning (ACL_ERROR_RT_FEATURE_NOT_SUPPORT)
5. Compact error output mode via OptionsManager

Frequency: 232 call sites across 40 files (csrc/ only; additional calls exist in third_party/)
Hotspot: torch_npu/csrc/core/npu/interface/AclInterface.cpp, NPUCachingAllocator.cpp

Good example:

    // torch_npu/csrc/core/npu/NPUFunctions.cpp:366
    NPU_CHECK_ERROR(aclrtSynchronizeStream(stream));

Variant: `NPU_CHECK_ERROR_WITHOUT_UCE` (NPUException.h:211) -- 69 usages across 9 files.
Used when UCE check is explicitly unwanted (e.g. during error query itself).

Anti-pattern from defect D-8: `aclrtGetDeviceInfo()` not supported on A5 (Ascend950).
ACL API compatibility varies across SoCs -- NPU_CHECK_ERROR alone is not sufficient for
platform-conditional APIs. Needs explicit SoC version check.
Commit: 2e5b8909c (v2.9.0-26.0.0)


### 3.2 TORCH_CHECK for Logic Assertions

- Scope: repo-wide
- Strength: mandatory for assertion/validation
- Rule: Use `TORCH_CHECK(condition, message)` for parameter validation, algorithm assertions,
  and any non-ACL error condition. This throws `c10::Error`.

Frequency: 824 call sites across 99 files (csrc/ only)
Hotspot: ProcessGroupHCCL.cpp (97 usages), Module.cpp (33 usages)

Good example:

    // torch_npu/csrc/distributed/ProcessGroupHCCL.cpp:218
    TORCH_CHECK(rank >= 0, "Invalid rank ", rank, DIST_ERROR(ErrCode::VALUE));

Convention: append domain-specific error code as last argument:

    TORCH_CHECK(cond, "message", PTA_ERROR(ErrCode::PARAM));   // core module
    TORCH_CHECK(cond, "message", OPS_ERROR(ErrCode::VALUE));   // operator module
    TORCH_CHECK(cond, "message", DIST_ERROR(ErrCode::HCCL));   // distributed module


### 3.3 Error Code Domain System

- Scope: repo-wide
- Strength: recommended
- Rule: Error messages should include a domain-specific error code suffix to help triage.

Definitions at torch_npu/csrc/core/npu/NPUException.h:89-93:

    #define PTA_ERROR(error)   formatErrorCode(SubModule::PTA, error)
    #define OPS_ERROR(error)   formatErrorCode(SubModule::OPS, error)
    #define DIST_ERROR(error)  formatErrorCode(SubModule::DIST, error)
    #define GRAPH_ERROR(error) formatErrorCode(SubModule::GRAPH, error)
    #define PROF_ERROR(error)  formatErrorCode(SubModule::PROF, error)

SubModule enum (NPUException.h:60-66): PTA=0, OPS=1, DIST=2, GRAPH=3, PROF=4
ErrCode enum (NPUException.h:68-84): SUC=0 through PERMISSION=12, then ACL=100, HCCL=200, GE=300

Frequency:
- PTA_ERROR: 89 usages
- OPS_ERROR: 90 usages
- DIST_ERROR: 91 usages
- GRAPH_ERROR, PROF_ERROR: lower usage

Triage value: the formatted code `[SubModule.ErrCode]` uniquely identifies which module
and error category, enabling log-based automated triage.


### 3.4 NPU_CHECK_WARN for Non-Fatal Warnings

- Scope: repo-wide
- Strength: recommended (low usage suggests underuse)
- Rule: Use `NPU_CHECK_WARN(err_code)` when ACL call failure is recoverable and
  should not abort execution.

Definition: torch_npu/csrc/core/npu/NPUException.h:30-42
Frequency: 30 usages across 16 files (vs 232 for NPU_CHECK_ERROR in csrc/)

Good example:

    // Non-fatal: log warning but continue execution
    NPU_CHECK_WARN(aclrtResetDevice(deviceId));


### 3.5 UCE (Uncorrectable Error) Detection

- Scope: repo-wide (critical path)
- Strength: mandatory for ACL calls on critical paths
- Rule: NPU_CHECK_ERROR includes UCE detection by default. The macro calls
  `AclrtPeekAtLastError(ACL_RT_THREAD_LEVEL)` to check for hardware errors that
  the normal return code might not reflect.

UCE error patterns detected (NPUException.h:95-109):
- `DEVICE_TASK_ABORT` -- device task aborted
- `DEVICE_MEM_ERROR` -- device memory error
- `DEVICE_HBM_ECC_ERROR` -- HBM multi-bit ECC error
- `HCCS_LINK_ERROR` -- inter-chip link error
- `HCCL_OP_RETRY_FAILED` -- collective op retry exhausted

Frequency: 18 AclrtPeekAtLastError call sites

When to skip UCE check: Use `NPU_CHECK_ERROR_WITHOUT_UCE` (69 usages, 9 files) for
calls that are themselves error queries (e.g. checking device status during error recovery).


### 3.6 Anti-Pattern: Bare catch(...) Blocks

- Scope: repo-wide (27 instances identified)
- Strength: known deficiency
- Rule: Avoid catch(...) blocks that silently swallow exceptions. At minimum,
  log the exception via `ASCEND_LOGE` before continuing.

Bad examples:
- torch_npu/csrc/InitNpuBindings.cpp:75,81,87 -- empty catch blocks in allocator cleanup
- torch_npu/csrc/distributed/ProcessGroupHCCL.cpp:268,488,1271,1376,1420 -- 10+ instances
- torch_npu/csrc/core/npu/CachingHostAllocator.cpp:309,868,912

Some locations do log before swallowing:
- torch_npu/csrc/InitNpuBindings.cpp:76 uses `ASCEND_LOGE` -- the correct pattern

Anti-pattern from defect D-5: new enum value `EXECUTE_OPAPI_V2` added to NPUQueue but
switch/if branches not updated in all consumers, causing silent misclassification.
Fix required grep-all-consumers discipline.
Commit: b70ee2d8f

---

## Ch.4 Resource Management

### 4.1 RAII Guards for Device/Stream Context

- Scope: repo-wide
- Strength: mandatory for device context switching
- Rule: Use RAII Guard objects for device/stream context management. Guards must:
  - Delete copy and move constructors/operators
  - Restore original state in destructor

Guard classes:

    Guard                    File                                      Usage
    NPUGuard                 core/npu/NPUGuard.h:20                   Device context guard
    OptionalNPUGuard         core/npu/NPUGuard.h:80                   Optional device guard
    NPUStreamGuard           core/npu/NPUGuard.h (near end)           Stream context guard
    OptionalNPUStreamGuard   core/npu/NPUGuard.h (near end)           Optional stream guard
    SecondaryStreamGuard     core/npu/SecondaryStreamGuard.h           Cross-stream sync guard
    NpuStorageOffsetGuard    framework/utils/NpuStorageOffsetGuard.h:10  Storage offset guard
    UnlockGuard              core/npu/NPUCachingAllocator.cpp:942     Reverse RAII (unlock)

Good example (NPUGuard.h:20-37):

    struct NPUGuard {
        explicit NPUGuard() = delete;
        explicit NPUGuard(c10::DeviceIndex device_index) : guard_(device_index) {}
        NPUGuard(const NPUGuard&) = delete;
        NPUGuard& operator=(const NPUGuard&) = delete;
        NPUGuard(NPUGuard&&) = delete;
        NPUGuard& operator=(NPUGuard&&) = delete;
        // ...
    private:
        c10::impl::InlineDeviceGuard<c10_npu::impl::NPUGuardImpl> guard_;
    };

Key pattern: NPUGuard wraps `c10::impl::InlineDeviceGuard` -- it delegates to PyTorch's
guard infrastructure rather than reimplementing. This ensures consistency with PyTorch's
device management semantics.

Anti-pattern: SecondaryStreamGuard records event on secondary stream, waits on primary in
destructor. If the guard scope is too wide, this creates unnecessary synchronization.

Anti-pattern from defect D-6: ACLGraph capture + multi-stream wait_stream interaction.
NPUEvent::block() failed to check graph capture state, causing replay errors.
Commit: 916426933


### 4.2 Singleton Lifecycle: Meyer's Singleton with GetInstance()

- Scope: repo-wide
- Strength: mandatory for global state
- Rule: Use Meyer's singleton (function-local static) for global state objects.
  Access via `ClassName::GetInstance()`.

Frequency: 193 GetInstance() calls across 53 files (from RAII agent)

Identified singletons:
- NpuSysCtrl -- core/npu/sys_ctrl/npu_sys_ctrl (ACL init/finalize)
- NPUCachingAllocator -- core/npu/NPUCachingAllocator (device memory pool)
- OptionsManager -- core/npu/register/OptionsManager (runtime options)
- NPUEventManager -- core/npu/NPUEventManager (event lifecycle)
- OpCommandImpls -- framework/OpParamMaker.cpp (thread_local variant)
- ForceAclnn -- framework/utils/ForceAclnnList.h
- CopyOptRegister -- framework/contiguous/contiguous_register.h

Variant: thread_local singletons for hot-path objects (OpCommandImpls) -- avoids
lock contention in per-operator paths.

Variant: std::once_flag for per-device/per-function lazy init (7 occurrences in
NPUStream.cpp, CachingHostAllocator.cpp, OptionRegister.h).

Teardown: NpuSysCtrl::Finalize() uses ReleasePriority enum (First/Middle/Last) for
ordered cleanup -- unique to this codebase.


### 4.3 Thread-Local Storage for Hot-Path State

- Scope: core/, distributed/
- Strength: recommended for performance-critical paths
- Rule: Use `thread_local` for per-thread state that would otherwise require locking
  on hot paths.

Frequency: 30 thread_local declarations

Key instances:
- `thread_local bool need_check_error` -- NPUCachingAllocator.cpp
- `thread_local MemPool* active_mempool_` -- NPUCachingAllocator.cpp
- `thread_local std::unique_ptr<LeakyStreamInternals*[]> current_streams` -- NPUStream.cpp
- `thread_local aclrtStream tls_prev_stream` -- NPUStream.cpp
- `thread_local int local_device` -- NPUFunctions.cpp
- `thread_local ThreadType local_thread` -- NPUAffinityController.cpp
- `thread_local uint64_t hcclActiveGroupCounter_` -- ProcessGroupHCCL.cpp

Anti-pattern: thread_local objects with non-trivial destructors can cause
shutdown ordering issues. NPU's LazyInit.cpp explicitly avoids `std::call_once`
due to ASAN deadlock on exception paths -- uses GIL + bool flag instead.

---

## Ch.5 API Design

### 5.1 OpCommand Fluent Builder

- Scope: framework/, aten/
- Strength: mandatory for operator implementation
- Rule: Operators use the `OpCommand` fluent builder API to construct and execute
  NPU kernel calls: `OpCommand().Name("op").Input(t).Output(r).Attr(a).Run()`.

Definition: torch_npu/csrc/framework/OpCommand.h:27
Frequency: 157 OpCommand usages across 16 files

Builder chain methods (each returns `OpCommand&`):
- `Name(string)` -- sets operator name (OpCommand.cpp:54-58)
- `Input(Tensor, ...)` -- adds input tensor (OpCommand.cpp:78-82)
- `Output(Tensor, ...)` -- adds output tensor (OpCommand.cpp:124-129)
- `Attr(key, value)` -- adds attribute
- `Expect(UnifiedResult)` -- sets expected result
- `Run()` -- executes the operator

Design rationale: the fluent interface encapsulates all operator parameters into a single
OpCommand object, which is then dispatched through the async task queue or sync path.
OpCommandImpls uses a per-thread object pool with stack-based Push/Pop reuse to avoid
allocation on hot paths.


### 5.2 Registration Macros for Plugin Architecture

- Scope: repo-wide
- Strength: mandatory for extending functionality
- Rule: Use the appropriate registration macro to register new capabilities:

    Macro                    Purpose                              Count
    TORCH_LIBRARY_IMPL       PyTorch dispatch key registration     12
    GET_FUNC                 Runtime dlsym loading                276 (design-patterns)
    REGISTER_LIBRARY         Shared library registration           20+
    REGISTER_COPY_OPT        Format copy optimization strategy      8
    REGISTER_OPTION          Runtime environment variable binding   15+
    C10_DEFINE_REGISTRY      c10 type registry                      4

Good example (TORCH_LIBRARY_IMPL):

    // torch_npu/csrc/aten/VariableFallbackKernel.cpp:259
    TORCH_LIBRARY_IMPL(_, PrivateUse1, m) {
        m.fallback(torch::CppFunction::makeFromBoxedFunction<&npu_fallback>());
    }

Good example (GET_FUNC):

    // Runtime loading without compile-time dependency on ACL SDK
    GET_FUNC(aclrtSetDevice, libascendcl, ...)

Design rationale: GET_FUNC enables building torch_npu without CANN SDK installed.
All ACL/HCCL/LCCL symbols are resolved at runtime via dlsym, eliminating compile-time
linking dependency. This is the heaviest registration macro (276 calls).


### 5.3 Operator Semantic Alignment with PyTorch

- Scope: aten/, framework/
- Strength: mandatory
- Rule: NPU operator implementations must match PyTorch native operator semantics exactly.
  Any deviation (different output format, different stride handling, different dtype behavior)
  is a correctness bug.

Anti-pattern from defect D-9: `clone()` implementation ignored `MemoryFormat::Preserve` and
unconditionally returned contiguous tensor. PyTorch native preserves original stride layout.
Commit: 5a798156e

Anti-pattern from defect D-10: inductor `pow` lowering missing fp64 fallback. NPU Triton
doesn't support fp64 pow, but no explicit fallback was registered, causing compilation failure.
Commit: a4bbd71ad

Review rule: NPU custom operator implementations must be validated against PyTorch op spec.
Lowering registrations must cover the full dtype matrix with explicit fallback for unsupported types.

Cross-ref: review-standards.md P1-1 provides detailed checkpoints for verifying
op semantic alignment (stride, dtype, MemoryFormat, output shape, out-variant).

---

## Ch.6 Logging

### 6.1 ASCEND_LOG{E,W,I,D} for Device-Level Logs

- Scope: repo-wide
- Strength: mandatory for device/runtime logging
- Rule: Use the 4-level ASCEND_LOG macros for device-level logging. These route through
  ACL's `aclAppLog` and respect the global log level setting.

Definitions at torch_npu/csrc/core/npu/npu_log.h:24-47:

    ASCEND_LOGE(fmt, ...)  -- Error level (ACL_ERROR)
    ASCEND_LOGW(fmt, ...)  -- Warning level (ACL_WARNING)
    ASCEND_LOGI(fmt, ...)  -- Info level (ACL_INFO)
    ASCEND_LOGD(fmt, ...)  -- Debug level (ACL_DEBUG)

Each macro:
1. Checks log level via `OptionsManager::isACLGlobalLogOn(level)`
2. Calls `aclAppLog(level, __FUNCTION__, FILE_NAME, __LINE__, "[PTA]:" fmt, ...)`
3. Uses printf-style format strings (not stream-style)

Frequency:

    Macro        Usages    Files    Level
    ASCEND_LOGE     163       32    Error
    ASCEND_LOGI     103       28    Info
    ASCEND_LOGD      77       22    Debug
    ASCEND_LOGW      75       23    Warning

Note: Total 418 ASCEND_LOG calls in csrc/. The Error:Debug ratio (~2:1) suggests
reasonable debug instrumentation in core infrastructure modules.


### 6.2 TORCH_NPU_WARN for User-Facing Warnings

- Scope: repo-wide
- Strength: recommended
- Rule: Use `TORCH_NPU_WARN(...)` for warnings that should be visible to Python users.
  This routes through PyTorch's c10::Warning system.

Definition: torch_npu/csrc/core/npu/NPUException.h:46-51

    #define TORCH_NPU_WARN(...)  \
        warn_(::c10::Warning(    \
            ::c10::UserWarning(),\
            {__func__, __FILE__, static_cast<uint32_t>(__LINE__)}, \
            ::c10::str(__VA_ARGS__), false));

Frequency: 124 usages across 27 files (csrc/ only)

Variant: `TORCH_NPU_WARN_ONCE(...)` (NPUException.h:53-58) -- emits warning only once.
Uses C10_ANONYMOUS_VARIABLE + lambda trick for thread-safe one-shot.

Module-specific log macros (from design-patterns):
- `TORCH_NPU_MEMORY_LOG` -- memory allocation tracing
- `TORCH_NPU_HCCL_LOG` -- HCCL communication tracing

---

## Ch.7 Testing Conventions

### 7.1 Test File Naming: test_*.py

- Scope: test/
- Strength: mandatory (99.7%)
- Rule: Test files follow `test_<module>_<feature>.py` naming convention.

Frequency: 622 test_*.py files, 2 exceptions (*_test.py)

Good examples:
- `test/npu/test_aclgraph_multi_stream.py`
- `test/distributed/test_hccl_shared_buffer.py`
- `test/_inductor/test_abs.py`

Directory organization by module/feature:

    test/
    |-- npu/            -- NPU device tests
    |-- distributed/    -- collective communication
    |-- _inductor/      -- compiler backend
    |-- nn/             -- neural network module
    |-- profiler/       -- profiling tests
    |-- contrib/        -- contrib module tests
    |-- trans_contiguous/ -- format conversion tests


### 7.2 Test Base Class: torch_npu.testing.testcase.TestCase

- Scope: test/
- Strength: mandatory (549/622 files)
- Rule: Test classes inherit from `torch_npu.testing.testcase.TestCase`, not
  `unittest.TestCase` directly.

Frequency:

    Base class                            Files    Usage
    TestCase (torch_npu.testing)            549    Primary
    JitTestCase                              41    JIT-specific tests
    NPUDTensorTestBase                       27    Distributed tensor
    NNTestCase                               20    Neural network
    unittest.TestCase                        20    Low-level/standalone
    FSDPNPUTest                               7    FSDP-specific

Class naming: `Test<Feature>(TestCase)` pattern.
- `TestAutocast(JitTestCase)` -- test_jit_autocast.py:18
- `TestPromptFlashAttention(TestCase)` -- test_prompt_flash_attention.py:11


### 7.3 Device-Specific Test Decorators

- Scope: test/
- Strength: recommended
- Rule: Use torch_npu-provided decorators for device-conditional test execution.

    Decorator                     Usages    Purpose
    @unittest.skip                   988    Skip unconditionally
    @skipIf                          948    Conditional skip
    @dtypes                          598    Data type parametrize
    @parametrize                     577    General parametrize
    @skipIfUnsupportMultiNPU         503    Multi-NPU availability
    @with_comms                      300    Distributed comms setup
    @SupportedDevices                279    Device filter (torch_npu-specific)
    @skipIfTorchDynamo               196    Dynamo incompatibility
    @onlyPRIVATEUSE1                 199    NPU-only test
    @onlyCPU                         136    CPU-only test
    @requires_npu                     35    NPU hardware requirement

Good example:

    @SupportedDevices(["npu"])
    def test_prompt_flash_attention(self):
        ...

Source: torch_npu.testing.common_utils.SupportedDevices (45 files import it)


### 7.4 Assertion Methods

- Scope: test/
- Strength: recommended
- Rule: Use `self.assertEqual` as the primary assertion. For floating-point comparison,
  use `self.assertRtolEqual` (torch_npu-specific) or `torch.testing.assert_close`.

Frequency:

    Method                       Usages
    self.assertEqual              12,648
    self.assertTrue                3,182
    self.assertRaisesRegex         1,980
    self.assertFalse                 823
    self.assertRaises                790
    self.assertRtolEqual             600
    self.assertIn                    327
    torch.testing.assert_close       191

`assertRtolEqual` is a torch_npu extension for relative tolerance comparison -- the
preferred method for numerical accuracy testing on NPU.


### 7.5 Device Specification in Tests

- Scope: test/
- Strength: observed
- Rule: Tests reference NPU device as string literal `"npu"`. The internal
  `"privateuse1"` name should not appear in test code.

Frequency:
- `"npu"` or `'npu'`: ~1,982 occurrences
- `"privateuse1"` or `'privateuse1'`: 41 occurrences (legacy, should be migrated)


### 7.6 Test Environment Variable Isolation

- Scope: test/
- Strength: recommended (not yet enforced)
- Rule: Tests that modify `os.environ` at module or class level must restore
  original values after the test to avoid cross-test contamination.

Anti-pattern (from validation, commit f83b2829d):

    # test_aclgraph_multi_stream.py:8 -- module-level, never cleaned up
    os.environ["PYTORCH_NPU_ALLOC_CONF"] = "expandable_segments:True"

Recommended pattern: use `unittest.mock.patch.dict(os.environ, {...})` as a
decorator or context manager, or save/restore in setUp/tearDown.

Rationale: module-level env mutations persist across all tests in the same
process, creating hidden dependencies between test files and non-deterministic
behavior depending on test execution order.


### 7.7 Bug Fix Regression Test

- Scope: test/
- Strength: recommended
- Rule: Bug fix commits should include or reference a regression test that reproduces
  the original failure scenario. The test should fail before the fix and pass after.

Validation evidence: 2 of 3 validated bug fix commits lack regression tests:
- 5a798156e (clone MemoryFormat) -- no test, ChannelsLast path remains untested
- 2e5b8909c (GetDeviceInfo A5) -- no test (hardware-specific, see exception)
- f83b2829d (aclgraph multi-stream) -- test included (good example)

Exception: hardware-specific fixes (SoC compat guards, device-specific behavior)
that cannot be reproduced without specific hardware should document the manual
test procedure in the commit message or PR description.


---

## Ch.8 Commit Message Conventions

### 8.1 No Enforced Standard (Highly Inconsistent)

- Scope: repo-wide
- Strength: optional (no enforcement)
- Rule: There is no consistently enforced commit message convention. Multiple styles coexist.

Distribution across 28,222 commits:

    Style                              Count    Pct     Example
    Automated (op_plugin/torchair)     8,347   29.6%   "Update op_plugin commit id"
    Free-form (no prefix)             ~18,000   63.8%   "fix empty_like output contiguous..."
    Bracket prefix [fix]/[feat]          709    2.5%   "[fix] tensor.clone support..."
    Chinese bracket 【】                  167    0.6%   "【fix】env control acc-check..."
    Conventional commits                  75    0.3%   "feat(distributed): 解耦子通信域..."
    Version prefix [v2.x.x]              62    0.2%   "[v2.9.0-26.0.0][bugfix]..."

Observed patterns:
1. Bug fixes: `fix`, `[fix]`, `[bugfix]`, `bugfix:`, `【fix】`, `修复`
2. Features: `feat`, `[feat]`, `[feature]`, `[Add]`, `add`, `新增`
3. Refactors: `[refactor]`, `refactor`
4. Tests: `[test]`, `test:`, `test(scope):`
5. Chinese and English mixed freely
6. Some commits include version bracket `[v2.9.0-26.0.0]` prefix

Recommendation: the repo would benefit from adopting a single convention.
Conventional Commits (`<type>(<scope>): <description>`) is already used by some
contributors and has the richest signal.

---

## Ch.9 Cross-Cutting Anti-Patterns (from Defect Analysis)

These anti-patterns recur across multiple defect entries and represent systemic
convention violations rather than one-off bugs.

### 9.1 Incomplete Enum Branch Coverage

- Frequency: defects D-5, D-7
- Rule: When adding a new enum value, grep all switch/if consumers and update every branch.
  Enable `-Wswitch-enum` compiler flag.
- Example: D-5 -- `EXECUTE_OPAPI_V2` added to NPUQueue but `get_func_error_msg()` and
  `Enqueue()` not updated. Commit: b70ee2d8f

### 9.2 Batch Operation Only Processing First Element

- Frequency: defect D-1
- Rule: In batch operations, verify that all elements in the container receive the same
  resource management treatment (e.g. recordStream).
- Example: D-1 -- `collective()` pre callback only called `recordStream()` on `tensors[0]`,
  skipping `tensors[1..n]`. Commit: c86a6dda3

### 9.3 Platform/SoC API Compatibility

- Frequency: defects D-4, D-8
- Rule: New ACL/driver API calls need a SoC compatibility matrix check.
  Hardware-specific APIs (e.g. `aclrtGetDeviceInfo`) may not be available on all SoC versions.
- Example: D-8 -- `aclrtGetDeviceInfo` not supported on A5 (Ascend950).
  Fix: SoC version check before API call. Commit: 2e5b8909c

### 9.4 Mutation-Before-Check in Loops

- Frequency: defect D-11
- Rule: In loops that modify state and then check conditions, verify the order is
  correct: check/record first, then mutate for next iteration.
- Example: D-11 -- tiling loop modified `blocks[axis]` after the `total_programs` check,
  so the check used stale values. Commit: 193938062

### 9.5 Missing fake_tensor Mode Handling

- Frequency: defect D-12
- Rule: Any code path that creates or rebuilds tensors must handle
  `torch._guards.detect_fake_mode()` for inductor compatibility.
- Example: D-12 -- `_rebuild_npu_tensor()` called `tensor.npu()` in fake mode,
  causing failure. Fix: detect fake mode and set `fake_device` instead. Commit: dd34f22e8

### 9.6 Revert-Prone Areas (from git-history.md)

The following areas have the highest revert frequency, indicating systemic fragility:

    Area                        Unique revert events    Example
    ProcessGroupHCCL                   3                Memory recording, shutdown
    Memory management                  2                NPUCachingAllocator optimizations
    Inductor/codegen                   3                Contiguous reduction, AsyncCompile
    Profiler                           2                export_stacks
    Op framework                       2                format_contiguous, IndexPut

Total: 101 REVERT commits (~30 unique events after dedup across version branches).

The pattern: aggressive optimization or refactoring in these areas without sufficient
regression testing leads to revert. Changes to ProcessGroupHCCL and NPUCachingAllocator
require extra scrutiny.


### 9.7 Printf Format Specifier Safety in Log Macros

- Frequency: from validation (commit 34d76cbf3)
- Rule: ASCEND_LOG{E,W,I,D} and TORCH_NPU_HCCL_LOG macros use C-style printf
  format strings internally. Format specifiers must match argument types exactly.
- Example: `TORCH_NPU_HCCL_LOGI("... use %d ms.", subTimeElapsed.count())`
  is wrong because `count()` returns `long long` but `%d` expects `int`.
  Fix: `static_cast<int>(...)` or use `%lld`.
- Common mismatch: `std::chrono::duration::count()` returns `rep` type
  (typically `long long`), not `int`.
- Detection: enable `-Wformat` compiler flag (usually on by default);
  use `%PRId64` or cast for std::chrono values.


### 9.8 Virtual Function Default Implementation Safety

- Frequency: from validation (commit f83b2829d)
- Rule: When adding a new `virtual` method to a base class with a "silent no-op"
  default (e.g. `{ return false; }`, `{ return 0; }`, `{}`), document why the
  default is safe for ALL existing subclasses. If the method controls critical
  behavior (resource management, synchronization state), prefer `= 0` (pure virtual)
  or `TORCH_CHECK(false, "not implemented")` over a silent default.
- Example: `virtual bool hasCapturesUnderway(c10::DeviceIndex) { return false; }`
  in NPUCachingAllocator.h:261 -- safe because only NpuCachingAllocator uses
  graph captures. If a future allocator implements captures but does not override,
  it silently returns false, causing `NPUEvent::block()` to skip the task queue flush.
- Detection: grep for `virtual.*\{ return false; \}` and `virtual.*\{ return 0; \}`
  in header files; verify each default has a safety comment.
