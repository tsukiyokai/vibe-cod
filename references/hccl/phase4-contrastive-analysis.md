# Phase 4: Cross-Repo Contrastive Analysis

hcomm (~3183 C++ files in src/) vs hccl (~430 C++ files in src/)

---

## 4.1 Error Handling Comparison

### Quantitative Overview

| Metric | hcomm | hccl | Ratio | Scope |
|--------|-------|------|-------|-------|
| CHK_RET | 14,457 / 286 files | 1,483 / 164 files | 9.7x | Project-level mandatory |
| CHK_PTR_NULL | 2,449 / 146 files | 85 / 24 files | 28.8x | Project-level mandatory |
| CHK_SMART_PTR_NULL | 1,394 / 108 files | 7 / 4 files | 199x | hcomm mandatory, hccl optional |
| EXECEPTION_CATCH | 32 / 8 files | 2 / 2 files | 16x | Recommended |
| EXCEPTION_HANDLE | 93 / 20 files | 0 | hcomm only | hcomm framework-level |
| RPT_INPUT_ERR | 305 / 34 files | 58 / 15 files | 5.3x | Project-level recommended |
| HCCL_SUCCESS refs | 5,300+ / 272 files | 1,158 / 189 files | 4.6x | Project-level mandatory |
| HCCL_E_ error codes | 3,750+ / 198 files | 492 / 138 files | 7.6x | Project-level mandatory |
| return HCCL_SUCCESS | 1,245 / 115 files | 551 / 125 files | 2.3x | Project-level mandatory |
| return HCCL_E_ | 830 / 92 files | 149 / 48 files | 5.6x | Project-level mandatory |
| goto | 10 / 6 files | 0 | - | Avoided (project-level) |
| try { | 100 / 55 files | 6 / 6 files | 16.7x | Rare in both |
| THROW | 165 / 51 files | 3 / 2 files | 55x | hcomm CCU layer mainly |
| errorFlag | 142 / 14 files | 0 | hcomm only | hcomm specific pattern |

### Macro Definition Comparison

| Macro | pub_inc vs hccl | legacy variant | Verdict |
|-------|-----------------|----------------|---------|
| CHK_RET | Identical | Uses `HcclResult::HCCL_E_AGAIN` scoped enum | Project-level core |
| CHK_PTR_NULL | Identical | "is NULL" vs "is nullptr", scoped enum | Project-level core |
| CHK_SMART_PTR_NULL | Identical across all | No variant | Project-level core |
| EXECEPTION_CATCH | Identical | Legacy enhanced: catches HcclException + std::exception + ... | Project-level base |
| RPT_INPUT_ERR | Identical | N/A | Project-level core |
| HCCL_ERROR_CODE | Identical | Identical | Project-level core |
| HcclResult enum | Defined in hcomm/include/hccl/hccl_types.h, hccl depends on it | N/A | Project-level core |

### Error Handling by Layer (hcomm)

| Layer | CHK_RET | CHK_PTR_NULL | RPT_INPUT_ERR | EXCEPTION_HANDLE | Character |
|-------|---------|-------------|---------------|------------------|-----------|
| algorithm/ | 5,476 (38%) | 182 (7%) | 0 | 0 | Pure CHK_RET |
| framework/ | 4,357 (30%) | 1,040 (42%) | 276 (90%) | 83 (89%) | Strictest validation |
| platform/ | 2,110 (15%) | 871 (35%) | 8 (~0%) | 0 | Balanced checking |
| legacy/ | 2,251 (16%) | 274 (11%) | 8 (~0%) | 0 | Lowest density |

### Project-Level Mandatory (both repos identical)

1. All functions return HcclResult; CHK_RET is the universal error propagation mechanism
2. CHK_RET auto-logs with HCCL_ERROR (HCCL_E_AGAIN downgrades to HCCL_WARNING)
3. do { ... } while(0) wraps all error-checking macros
4. UNLIKELY() hint on all error paths
5. No goto, no errorFlag, no C++ exceptions as primary error handling
6. Shared error code enum (HCCL_E_PARA, HCCL_E_PTR, etc.) from hcomm/include/

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| CHK_SMART_PTR_NULL density | High (1,394x) | Near zero (7x) |
| EXCEPTION_HANDLE | 93 uses (framework C API boundary) | Not used |
| THROW macro | 165 uses (CCU representation layer) | 3 uses |
| errorFlag pattern | 142 uses (hccd compatibility) | Not used |
| try/catch | 100 try blocks (platform transport) | 6 try blocks (sal.cc only) |
| RPT_INPUT_ERR density | 90% concentrated in framework/ | Spread across ops/ |
| Error handling dual-track | CCU layer uses THROW, others use CHK_RET | Uniform CHK_RET only |

### Key Insight

hcomm has a dual-track error handling system: CHK_RET (majority) + THROW (CCU representation/microcode layer).
hccl uses exclusively CHK_RET with zero THROW and zero try/catch in ops/.
This reflects hcomm's multi-paradigm nature (C ABI boundary + C++ internals + compiler-style CCU) vs hccl's pure algorithmic focus.

---

## 4.2 Memory Management Comparison

### Quantitative Overview

| Metric | hcomm | hccl | Ratio | Scope |
|--------|-------|------|-------|-------|
| unique_ptr | ~283 / 100+ files | 95 / 58 files | 3x | Project-level preferred |
| shared_ptr | ~248 / 100+ files | 70 / 30 files | 3.5x | Project-level preferred |
| make_unique | 47 | 15 | 3.1x | Recommended |
| make_shared | 82 | 93 | 0.9x | hccl uses more per-file |
| new(std::nothrow) | 324 | 5 | 64.8x | hcomm pattern |
| malloc | 61 | 12 | 5.1x | Legacy/C-interop |
| free( | 57 | 7 | 8.1x | Legacy/C-interop |
| DeviceMem | ~130+ | 0 | hcomm only | hcomm specific |
| HostMem | 275 | 0 | hcomm only | hcomm specific |
| owner_ | 38 | 0 | hcomm only | hcomm move-semantic tag |
| std::move | 441 / 206 files | 3 / 2 files | 147x | hcomm core idiom |
| Pimpl (impl_ member) | 57 / 75 files | 0 | hcomm only | hcomm API stability |

### Project-Level Mandatory

1. Smart pointers (unique_ptr/shared_ptr) over raw new/delete
2. Zero bare delete in both repos (all managed through smart pointers or custom wrappers)
3. new(std::nothrow) preferred over bare new when manual allocation needed

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| Memory abstraction | DeviceMem/HostMem custom types | Standard STL only |
| Move semantics | Systematic (441 std::move) | Minimal (3 std::move) |
| Pimpl pattern | 75 files, ABI stability | Not used |
| owner_ convention | 38 uses, ownership transfer | Not used |
| malloc/free | 61/57, C-interop & legacy | 12/7, rare (logging init) |
| new(nothrow) | 324 uses (platform layer) | 5 uses |

### Memory Management Archetypes

hcomm = "Resource Infrastructure" — Multi-layer abstraction (DeviceMem/HostMem), move semantics pervasive, Pimpl for API stability, handles hardware memory heterogeneity (HBM/DDR/SRAM).

hccl = "Algorithm Expression" — Pure STL, no custom memory types, minimal move semantics, no hardware memory management. Memory complexity delegated to hcomm.

### Rules for vibe-cod

- In hcomm: use owner_+move semantics for resource types, new(std::nothrow) for manual allocation, Pimpl for public headers
- In hccl: use make_unique/make_shared, avoid malloc/free, no need for Pimpl
- Cross-boundary: DeviceMem/HostMem passed by rvalue reference (ownership transfer)

---

## 4.3 Logging Comparison

### Quantitative Overview

| Metric | hcomm | hccl | Ratio | Scope |
|--------|-------|------|-------|-------|
| HCCL_DEBUG | 3,097 / 534 files | 545 / 160 files | 5.7x | Project-level |
| HCCL_INFO | 7,679 / 955 files | 935 / 169 files | 8.2x | Project-level |
| HCCL_WARNING | 1,362 / 344 files | 194 / 60 files | 7.0x | Project-level |
| HCCL_ERROR | 9,605 / 877 files | 664 / 160 files | 14.5x | Project-level |
| HCCL_RUN_INFO | 995 | 31 | 32.1x | Mostly hcomm |
| HCCL_PROFILER | 73 | 0 | hcomm only | hcomm specific |
| RPT_INPUT_ERR | 305 | 58 | 5.3x | Project-level |
| RPT_ENV_ERR | 70 | 9 | 7.8x | Project-level |
| tag[ pattern | 1,061 | 16 | 66.3x | hcomm core practice |
| __func__ in logs | 2,295 | 126 | 18.2x | Both use, hcomm more |

### Log Level Distribution

| Level | hcomm % | hccl % | Observation |
|-------|---------|--------|-------------|
| ERROR | 42.4% | 28.4% | hcomm more error-heavy |
| INFO | 33.9% | 39.9% | hccl more info-balanced |
| DEBUG | 13.7% | 23.3% | hccl more debug-heavy |
| WARNING | 6.0% | 8.3% | Similar |

hcomm ERROR proportion significantly higher (42% vs 28%) — reflects more defensive error capturing in communication infrastructure.

### Project-Level Mandatory

1. All logging through HCCL_DEBUG/INFO/WARNING/ERROR macros (never direct DLOG)
2. Log format auto-includes [file:line] [tid]
3. Conditional level check via UNLIKELY(HcclCheckLogLevel(...))
4. ErrToWarn switch supported (SetErrToWarnSwitch())
5. Log definitions in both repos derive from common dlog_pub.h backend
6. Macro definitions in pub_inc/log.h (hcomm) and common/log.h (hccl) are identical

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| tag[ in logs | 1,061 uses — mandatory for comm operations | 16 uses — rare |
| Log style | `[Module][SubModule] ... tag[%s]` | `[Algo][Selector] ... ` functional path |
| HCCL_RUN_INFO | 995 uses (critical path tracing) | 31 uses |
| HCCL_PROFILER | 73 uses | Not used |
| RPT_INNER_ERR | 4 uses | Not used |
| config_log.h | Not used | hccl specific (HCCL_CONFIG_DEBUG/INFO) |

### Key Insight

hcomm mandates tag[%s] in all communication-related logs (1,061 occurrences) for distributed tracing across ranks/groups. This is a critical hcomm-specific practice — every log that relates to a communication domain operation must include the tag identifier.

hccl uses functional module prefix (`[Algo]`, `[Selector]`, `[Template]`) to organize log output — simpler but sufficient for algorithm-focused debugging.

---

## 4.4 Naming Convention Comparison

### Project-Level Mandatory (both repos)

| Category | Convention | Evidence |
|----------|-----------|----------|
| File names | snake_case | 100% in both repos |
| Class names | PascalCase | 100% in both repos |
| Function names | PascalCase | 100% in both repos |
| Member variables | trailing underscore (e.g. `count_`) | Dominant in both |
| Macros/constants | ALL_CAPS | 100% in both repos |
| No m_ prefix | Avoided | Neither repo uses m_ systematically |

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| _pub.h suffix | 187 files (explicit API visibility) | 1 file |
| _base.h suffix | 14 files | Used but fewer |
| Manager class suffix | 63 classes | 0 classes |
| Impl class suffix | Common (Pimpl pattern) | Not used |
| Primary namespace | `hccl` (shared) | `ops_hccl` (unique) |
| Class taxonomy | Base/Manager/Impl/Factory | Base/Executor/Selector/Registry |

### Naming Archetypes

hcomm uses enterprise-style naming: explicit visibility markers (_pub.h), Manager classes (63), Impl classes for Pimpl. Reflects a mature infrastructure codebase with strong API boundary discipline.

hccl uses algorithm-oriented naming: Executor/Selector/Template/Registry. Reflects a focused algorithmic codebase where execution semantics drive naming.

### Key Insight

The _pub.h convention (187 files in hcomm, 1 in hccl) is the most significant naming difference. hcomm enforces API/implementation separation through filename convention; hccl relies on directory structure (common/ vs ops/) for the same purpose.

---

## 4.5 Design Pattern Comparison

### Quantitative Overview

| Pattern Metric | hcomm | hccl | Scope |
|---------------|-------|------|-------|
| Pure virtual (= 0) | 168 | 8 | hcomm deeply abstract |
| override methods | 573 | 152 | hcomm deep inheritance |
| REGISTER_* macros | Low freq (legacy) | 173 | hccl systematic |
| Executor classes | 187 | 34 | Both use, hcomm more |
| Selector classes | 0 (implicit) | 17 (explicit) | hccl specific |
| Callback/callback | 927 | 2 | hcomm event-driven |
| GetInstance (Singleton) | ~54 | ~77 | Both use extensively |

### Pattern Application Comparison

| Pattern | hcomm | hccl | Verdict |
|---------|-------|------|---------|
| Template Method | Deep (6-7 layer inheritance) | Shallow (2-3 layers) | Both use, different depth |
| Strategy | Implicit in Executor internals | Explicit Selector classes | Architectural difference |
| Registry | Legacy function-pointer map | Modern REGISTER_* macros (173) | hccl more systematic |
| Factory | CommFactory (explicit) | Via Registry (implicit) | Different approach |
| Observer | 927 callback uses | 2 callback uses | hcomm event-driven |
| Singleton | Per-device array pattern | Per-device array pattern | Project-level |
| Pimpl | 75 files | 0 files | hcomm only |
| State Machine | 7+ FSMs (OpRetry 22-state) | Minimal | hcomm only |

### Project-Level Shared Patterns

1. Per-device Singleton: `static T instances[MAX_DEVICE_ID + 1]` (both repos)
2. Template Method: Virtual override for algorithm customization (both repos)
3. Executor as primary algorithm container (both repos)
4. Base class suffix convention (XxxBase) (both repos)

### Repo-Level Pattern Differences

hcomm architectural style:
- Traditional OOP: deep inheritance, runtime virtual dispatch, event-driven callbacks
- Defensive design: Pimpl for ABI, State machines for reliability, Observer for async
- Enterprise complexity: 168 pure virtual functions, 573 overrides, 7+ state machines

hccl architectural style:
- Modern C++ template: compile-time type injection, shallow hierarchy, synchronous execution
- Algorithm-focused: explicit Selector separation, Registry macros, template parameter packs
- Lean design: 8 pure virtual functions, 152 overrides, minimal state machines

### Key Insight

The Strategy pattern implementation diverges most: hcomm embeds algorithm selection inside Executor virtual methods (implicit Strategy), while hccl extracts it into independent AutoSelector classes (explicit Strategy with SRP). This is the defining architectural difference.

---

## 4.6 Conditional Compilation Comparison

### Quantitative Overview

| Metric | hcomm | hccl | Ratio |
|--------|-------|------|-------|
| #ifdef total | 419 / 187 files | 33 / 17 files | 12.7x |
| __attribute__ | 301 / 75 files | 17 / 6 files | 17.7x |
| #pragma once | 6 | 4 | Similar (both rare) |
| Include guard (#ifndef) | ~187 files | ~17+ files | Standard in both |
| AICPU_COMPILE refs | ~1,596 | ~142 | hcomm much more |

### Project-Level Mandatory

1. Include guard (#ifndef) over #pragma once (both repos, near-zero #pragma once)
2. AICPU_COMPILE / CCL_KERNEL_AICPU for device-side compilation split
3. Conditional compilation flags declared in CMakeLists.txt

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| #ifdef density | 5.9% of files | 3.9% of files |
| __attribute__((weak)) | Extensive (V2 dispatch) | Minimal |
| Device-type branching | AICPU/AIV/CCU/Host 4-way | AICPU/AIV 2-way mainly |
| Host/Device file split | 6+ paired files per module | Per-template split |
| Platform adaptation | HCCS/RoCE/PCIe/TCP selection | Delegated to hcomm |

### Key Insight

hcomm's conditional compilation complexity (12.7x more #ifdef) directly reflects its role as the hardware abstraction layer — it must adapt to multiple chip generations, communication links, and execution engines. hccl delegates all hardware complexity to hcomm, keeping its own conditional compilation minimal.

---

## 4.7 Testing Style Comparison

### Quantitative Overview

| Metric | hcomm | hccl | Scope |
|--------|-------|------|-------|
| test/ut/ .cc files | 273 | 1 | hcomm UT-heavy |
| test/st/ .cc files | ~120 | 62 | Both have ST |
| TEST_F count | 222 | 222 | Coincidentally equal |
| TEST_P count | 37 | 1 | hcomm uses parametric |
| Fixture classes | 660 | 17 | hcomm 38.8x more |
| EXPECT_EQ/TRUE | ~1600+ | 22 | hcomm assertion-heavy |
| MOCKER/MockCPP | ~1000+ | 0 | hcomm only |
| #define private public | ~57 files | 0 | hcomm only |
| Mock classes | 5 | 0 | hcomm only |

### Project-Level Shared

1. Google Test (gtest) as test framework (both repos)
2. TEST_F as dominant test macro (both repos)
3. ST simulation framework: SimWorld + DAG-based verification + memory conflict detection (shared methodology)
4. Checker-based semantic validation (shared approach)
5. MAKE_ENUM for test assertions (both repos)

### Repo-Level Differences

| Aspect | hcomm | hccl |
|--------|-------|------|
| Testing strategy | UT-heavy (273 UT files) + ST | ST-only (1 UT file = env cleanup) |
| Mock framework | MockCPP (MOCKER/MOCKER_CPP) | No mocking |
| White-box testing | #define private public (57 files) | Black-box only |
| Assertion density | ~1600+ EXPECT_* | 22 EXPECT_* |
| Test granularity | Fine-grained (per-method) | Coarse (per-algorithm scenario) |
| Parametric testing | 37 TEST_P | 1 TEST_P (reduce only) |
| Stub library | hccl_llt (33 stub files) | SimWorld (94 files) |
| Build flags | -fno-access-control, -O0 -g --coverage | Standard |

### Testing Philosophy

hcomm = "Layer-by-layer unit testing" — Each internal module has UT coverage, extensive mocking of external dependencies, white-box access to private members. Reflects mature infrastructure with complex internal state.

hccl = "End-to-end scenario testing" — Algorithms verified through complete simulation (SimWorld), no internal state inspection needed. Reflects pure algorithm focus where functional correctness is the only metric.

### Key Insight

The 273:1 UT file ratio is the most striking difference. hcomm tests internal implementation (white-box), hccl tests algorithmic behavior (black-box). Both approaches are valid for their respective domains — infrastructure needs internal correctness guarantees, algorithms need functional correctness guarantees.

---

## Summary: Project-Level vs Repo-Level Classification

### Project-Level Mandatory (both repos identical)

| # | Rule | Category | Evidence |
|---|------|----------|----------|
| P1 | HcclResult return type + CHK_RET propagation | Error handling | 14,457 + 1,483 uses |
| P2 | do{...}while(0) + UNLIKELY() in all macros | Error handling | All macro definitions |
| P3 | No goto, no C++ exceptions as primary mechanism | Error handling | 10+0 goto, 100+6 try |
| P4 | Smart pointers over raw new/delete | Memory | Zero bare delete |
| P5 | HCCL_DEBUG/INFO/WARNING/ERROR macros | Logging | Both repo log.h identical |
| P6 | snake_case files, PascalCase classes/functions | Naming | 100% consistency |
| P7 | trailing underscore for members | Naming | Dominant in both |
| P8 | Per-device Singleton pattern | Design | 54 + 77 GetInstance |
| P9 | Template Method for algorithm extension | Design | Both use Executor/Template hierarchy |
| P10 | Google Test + SimWorld for testing | Testing | Shared framework |
| P11 | #ifndef include guard | Build | Near-zero #pragma once |
| P12 | AICPU_COMPILE device-side split | Build | Both use |
| P13 | Shared error code enum (HcclResult) | Error handling | hccl depends on hcomm definition |

### Repo-Level: hcomm-Specific

| # | Rule | Category | Evidence |
|---|------|----------|----------|
| H1 | DeviceMem/HostMem + owner_ + move semantics | Memory | 130+275+38+441 uses |
| H2 | Pimpl for public API headers (_pub.h) | Memory/Naming | 75 files, 187 _pub.h |
| H3 | new(std::nothrow) for manual allocation | Memory | 324 uses |
| H4 | EXCEPTION_HANDLE at C API boundary | Error handling | 93 uses in framework |
| H5 | THROW in CCU representation layer | Error handling | 165 uses |
| H6 | tag[%s] in all communication-related logs | Logging | 1,061 uses |
| H7 | HCCL_RUN_INFO for critical path tracing | Logging | 995 uses |
| H8 | Manager class suffix (63 classes) | Naming | Enterprise-style naming |
| H9 | Deep inheritance (6-7 layers) | Design | 168 pure virtual, 573 override |
| H10 | Event-driven callbacks (927 uses) | Design | Observer pattern pervasive |
| H11 | State machine driven design (7+ FSMs) | Design | OpRetry 22-state |
| H12 | White-box UT + MockCPP | Testing | 273 UT files, #define private public |
| H13 | Heavy conditional compilation | Build | 419 #ifdef, 301 __attribute__ |

### Repo-Level: hccl-Specific

| # | Rule | Category | Evidence |
|---|------|----------|----------|
| K1 | Pure STL memory, no custom types | Memory | Zero DeviceMem/HostMem |
| K2 | Uniform CHK_RET only (no THROW) | Error handling | 3 THROW total |
| K3 | ops_hccl namespace | Naming | vs hccl in hcomm |
| K4 | Executor/Selector/Registry naming | Naming | Algorithm-oriented taxonomy |
| K5 | Explicit Selector classes (17) | Design | Separated from Executor |
| K6 | REGISTER_* macro system (173 uses) | Design | Systematic modern registry |
| K7 | Shallow inheritance (2-3 layers) | Design | 8 pure virtual, 152 override |
| K8 | Synchronous execution model | Design | 2 callback uses total |
| K9 | ST-only testing (no UT) | Testing | 1 UT file = env cleanup |
| K10 | Minimal conditional compilation | Build | 33 #ifdef |

### Maturity and Evolution

| Dimension | hcomm | hccl |
|-----------|-------|------|
| Code age | Mixed (legacy + next framework) | Mostly modern |
| Architecture style | Enterprise OOP | Modern C++ template |
| Abstraction depth | Deep (multiple indirection layers) | Shallow (direct) |
| Pattern complexity | High (Observer, State, Pimpl) | Moderate (Registry, Template Method) |
| Technical debt markers | 17 TODO in next/, legacy 40%+ activity | 0 TODO, 0 FIXME |
| God Class problem | 4 files > 4000 lines | None |

---

## Actionable Rules for vibe-cod

### When writing code in hcomm:

1. Every function returns HcclResult; wrap every call in CHK_RET
2. Use new(std::nothrow) for manual allocation; prefer unique_ptr with Pimpl for public APIs
3. Add tag[%s] to every log related to communication operations
4. Use EXCEPTION_HANDLE at C API boundary functions
5. Use _pub.h suffix for public headers, Manager suffix for management classes
6. Expect deep inheritance; override virtual methods from base classes
7. Follow V1/V2 branching pattern (HCCLV2_FUNC_RUN) when touching API entry points
8. Write UT with MockCPP for new modules; use MOCKER for C function mocking

### When writing code in hccl:

1. Every function returns HcclResult; wrap every call in CHK_RET (same as hcomm)
2. Use make_unique/make_shared; avoid malloc/free and Pimpl
3. Use [Module][Function] prefix in logs; tag[ is not needed
4. Register new executors via REGISTER_EXEC_V2 macro; implement AutoSelector for algorithm selection
5. Keep inheritance shallow (inherit InsCollAlgBase or similar, 2-3 levels max)
6. Write ST tests with SimWorld simulation; UT is not the norm
7. Use ops_hccl namespace
8. No THROW, no try/catch — CHK_RET only
