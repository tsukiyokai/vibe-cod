# torch_npu Architecture Map

Repository: `/Users/shanshan/repo/torch/torch_npu`
Branch: v2.9.0-26.0.0 (PyTorch 2.9.0 + CANN 26.0.0)
Language: C++ / Python (hybrid)

torch_npu 是华为昇腾 NPU 的 PyTorch out-of-tree 后端扩展。
它通过 PyTorch 的设备扩展机制(`torch.utils.backend_registration`)将
Ascend NPU 注册为一等设备，提供算子、内存管理、分布式通信、编译器后端等完整栈。


## 1. 顶层目录结构

    目录                 子目录   文件    主要语言     职责
    ──────────────────────────────────────────────────────────────────────
    torch_npu/             18     735    C++/Python   核心扩展实现
    torchnpugen/            2      28    Python       代码生成 (autograd/backend stub)
    third_party/           14   6,924    C++          外部依赖 (op-plugin/torchair/hccl等)
    test/                  24     689    Python       测试套件
    benchmarks/             0      10    Python       TorchBench 集成
    ci/                     2      17    Python/Bash  CI/CD 与 Docker
    cmake/                  0       1    CMake        包配置模板
    codegen/                0       1    -            (空, 占位)
    docs/                   3      16    Markdown     文档与 Release Notes
    examples/               3      11    C++/Python   libtorch/hccl 示例
    tools/                  1      11    Python       flight_recorder 等工具
    .ci/                    1       1    Docker       Legacy CI
    .github/                2       6    YAML         GitHub Actions

数据来源: `find <dir> -type f | wc -l`, `find <dir> -maxdepth 1 -type d | wc -l`

顶层文件:

    文件                     行数      职责
    ──────────────────────────────────────────────────────────────────────
    setup.py                 772      Python 安装与构建编排
    CMakeLists.txt           380      CMake 构建配置 (单一 torch_npu.so)
    build_libtorch_npu.py    288      纯 C++ libtorch_npu 构建脚本
    generate_code.sh          47      代码生成入口脚本
    version.txt               12      版本号 (2.9.0)


## 2. torch_npu/ 核心模块概览

总计: 342 Python 文件 (70,866 LOC) + 368 C++/H 文件 (78,093 LOC)

### 2.1 按模块统计

    模块              Py文件   Py LOC   C++文件   C++ LOC   定位
    ──────────────────────────────────────────────────────────────────────
    csrc/                0        0      367    77,751   C++ 核心层
    _inductor/          75   28,407        1       342   编译器后端(Inductor)
    profiler/          105   12,838        0         0   性能分析
    npu/                34    8,860        0         0   NPU 设备接口
    utils/              33    5,084        0         0   补丁与工具
    distributed/        18    4,493        0         0   分布式训练
    contrib/            36    4,271        0         0   扩展算子/模块
    optim/              11    2,058        0         0   优化器
    onnx/                3    1,664        0         0   ONNX 导出
    testing/             9    1,483        0         0   测试基础设施
    asd/                 5      929        0         0   自动稀疏检测
    dynamo/              2      271        0         0   TorchDynamo 集成
    multiprocessing/     1      203        0         0   多进程 spawn
    _afd/                2      192        0         0   自动故障诊断
    _logging/            2       70        0         0   日志配置
    jit/                 4       34        0         0   JIT 集成 (stub)
    _compiler/           1        9        0         0   编译器入口 (stub)

数据来源: `find torch_npu/<mod> -name "*.py" | wc -l` 等

### 2.2 csrc/ C++ 子模块

    子模块             文件数    LOC      定位
    ──────────────────────────────────────────────────────────────────────
    core/               90   20,872   设备抽象/存储/张量/序列化
    distributed/        42   20,182   集合通信 (HCCL/LCCL)
    framework/          69    9,825   算子命令框架 (OpCommand/OpHook/Format)
    inductor/           31    6,146   AOT Inductor C++ 运行时
    aten/               47    6,033   ATen 算子注册与分发
    npu/                17    5,482   NPU 设备/流/事件/内存
    profiler/           22    3,838   C++ 性能分析
    toolkit/             9    1,508   npu_profiler 共享库
    ipc/                 4      749   进程间通信 (张量共享)
    afd/                 4      668   自动故障诊断 C++ 层
    flopcount/           7      571   算力统计
    logging/             6      499   日志基础设施
    utils/               6      498   C++ 工具函数
    custom_dtype/        4      304   自定义数据类型 (HiFloat8等)
    sanitizer/           4      199   NPU Sanitizer
    libs/                3      151   libtorch 模式入口
    include/             1        3   头文件 (转发)

数据来源: `find torch_npu/csrc/<mod> \( -name "*.cpp" -o -name "*.h" \) | wc -l` 等

csrc/ 呈现清晰的三支柱格局:
- core/ (20,872 LOC): 设备层抽象, 是 npu/ 和 framework/ 的基础
- distributed/ (20,182 LOC): HCCL 集合通信, 几乎自成体系
- framework/ (9,825 LOC): 算子执行框架, 是 aten/ 的直接依赖


## 3. 构建系统

### 3.1 构建模式

两种构建模式共享同一 CMakeLists.txt:

1. Python Extension (默认): `python setup.py bdist_wheel`
   - 产物: `torch_npu/_C.*.so` (Python 扩展) + `torch_npu/lib/torch_npu.so`
   - 包含全部模块

2. LibTorch C++ (BUILD_LIBTORCH=ON): `python build_libtorch_npu.py`
   - 产物: `libtorch_npu/lib/libtorch_npu.so`
   - 裁剪 npu/profiler/ipc/utils/sanitizer/afd, 只保留核心

### 3.2 CMake 目标与链接

单一共享库 `torch_npu` (CMakeLists.txt:297):

    链接依赖              来源               性质
    ──────────────────────────────────────────────────────────────────────
    libhccl.so           third_party/acl/    HCCL 集合通信
    libascendcl.so       third_party/acl/    ACL 运行时
    libacl_op_compiler   third_party/acl/    ACL 算子编译器
    libge_runner.so      third_party/acl/    GE 图引擎
    libgraph.so          third_party/acl/    GE 图定义
    torch, torch_cpu, c10  PyTorch           PyTorch 核心
    fmt::fmt-header-only   third_party/fmt   格式化库
    npu_profiler         toolkit/profiler/   性能分析 (非LIBTORCH)
    libtensorpipe.so     third_party/        RPC 通信 (可选)
    torch_python         PyTorch             Python绑定 (非LIBTORCH)
    libdvm.a             third_party/dvm/    DVM 编译器 (静态)

### 3.3 条件编译标志

    标志                 效果
    ──────────────────────────────────────────────────────────────────────
    BUILD_LIBTORCH       纯 C++ 构建, 裁剪 Python 相关模块
    BUILD_TENSORPIPE     启用 RPC 框架 (定义 USE_RPC_FRAMEWORK)
    BUILD_TORCHAIR       构建 TorchAIR (GE图编译器前端)
    BUILD_GTEST          构建 C++ 测试
    ENABLE_LTO           薄 LTO (仅 Clang)
    PGO_MODE             Profile-Guided Optimization (1=生成, 2=使用)

### 3.4 代码生成链

    generate_code.sh (generate_code.sh:1-47)
    ├── third_party/op-plugin/gencode.sh        op-plugin 算子绑定
    ├── torchnpugen.gen_backend_stubs            ATen 后端 stub
    │     输入: npu_native_functions.yaml
    │     输出: csrc/aten/RegisterNPU.cpp, NPUNativeFunctions.h 等
    ├── torchnpugen.autograd.gen_autograd        自动微分代码
    │     输出: csrc/aten/ 下 autograd 相关文件
    └── torchnpugen.codegen_ops_info             算子信息表

### 3.5 Git Submodules

    子模块                    分支        职责
    ──────────────────────────────────────────────────────────────────────
    op-plugin                26.0.0     NPU 算子实现 (最大的子模块)
    torchair                 26.0.0     GE 图编译器前端
    dvm                      r2.9       DVM 向量化编译器
    Tensorpipe               -          RPC 传输层
    torch-mlir               -          MLIR 编译器集成
    fmt                      -          格式化库
    nlohmann                 -          JSON 库
    googletest               -          C++ 测试框架


## 4. csrc/ 层间依赖

### 4.1 #include 引用热度 (被 include 的目标模块, 跨文件计数)

    目标模块              被引用次数    含义
    ──────────────────────────────────────────────────────────────────────
    core/npu/              394         设备/流/事件/内存/异常 (绝对核心)
    framework/utils/        71         OpAdapter/NpuUtils/FormatHelper
    framework/              71         算子命令框架根目录
    aten/                   68         ATen 后端注册与算子
    core/                   59         NPUStorageImpl/NPUBridge/序列化
    core/npu/register/      52         动态库加载/选项管理
    core/npu/interface/     40         异步任务队列接口
    profiler/               29         性能分析钩子
    framework/interface/    27         OpCommand 接口
    aten/common/            27         EmptyTensor/ResizeNpu

数据来源: `grep -rh '#include.*"torch_npu/csrc/' torch_npu/csrc/ | sed ... | uniq -c`

### 4.2 最常被 include 的头文件 (Top-15)

    引用次数  头文件                              所属模块    职责
    ──────────────────────────────────────────────────────────────────────
       75    NPUException.h                     core/npu   异常类型定义
       48    NPUMacros.h                        core/npu   编译器宏/断言
       41    NPUNativeFunctions.h               aten       Native 函数声明
       34    NPUStream.h                        core/npu   流管理
       33    NPUFunctions.h                     core/npu   设备查询函数
       31    npu_log.h                          core/npu   日志宏
       28    NPUCachingAllocator.h              core/npu   缓存分配器接口
       27    NPUStorageImpl.h                   core       存储实现
       27    NPUGuard.h                         core/npu   设备 Guard
       25    NPUBridge.h                        core       设备桥接
       25    OptionsManager.h                   core/npu   运行时选项
       22    FormatHelper.h                     framework  格式转换辅助
       20    npu_sys_ctrl.h                     core/npu   系统控制 (Init/Finalize)
       19    NpuVariables.h                     core/npu   全局变量
       18    AsyncTaskQueueInterface.h          core/npu   异步任务队列

数据来源: `grep -rh '#include.*"torch_npu/csrc/' torch_npu/csrc/ | sed ...`

### 4.3 模块间依赖矩阵 (来源 -> 目标, 按引用量)

    来源模块       -> 最重依赖              引用文件数
    ──────────────────────────────────────────────────────────────────────
    framework/     -> core/npu              13 个文件  (OpCommand 需要流/事件/内存)
    aten/          -> core/npu              18 个文件  (ATen 算子需要设备抽象)
    distributed/   -> core/npu              12 个文件  (HCCL 需要设备管理)
    npu/           -> core/npu              15 个文件  (Python binding 层桥接 core)
    profiler/      -> core/npu              11 个文件  (Profiler 钩入设备事件)
    aten/          -> framework/            17 个文件  (ATen 算子使用 OpCommand)
    framework/     -> aten/                 17 个文件  (框架引用 ATen 类型)

三角耦合: core <-> framework <-> aten 之间存在双向引用,
但方向明确: core 提供基础设施, framework 提供执行框架, aten 提供算子注册。

叶子模块 (无被依赖或依赖极少): afd, flopcount, inductor, libs
服务模块 (被广泛使用但不依赖业务): logging, sanitizer


## 5. 数据流与初始化

### 5.1 Python 初始化序列

`import torch_npu` 触发 `torch_npu/__init__.py` (336行), 按以下顺序执行:

    阶段    操作                                              引用
    ──────────────────────────────────────────────────────────────────────
    1      禁用 TORCH_DEVICE_BACKEND_AUTOLOAD, import torch   __init__.py:14-17
    2      检查无其他加速器冲突 (互斥检测)                     __init__.py:22-38
    3      import torch_npu.npu (加载 C++ .so)                __init__.py:41
    4      import amp/aclnn/optim/dynamo/_C/_logging/_afd     __init__.py:62-70
    5      import profiler/contrib/distributed                 __init__.py:71-98
    6      rename_privateuse1_backend("npu")                  __init__.py:210
    7      _register_device_module('npu', torch_npu.npu)      __init__.py:212
    8      _apply_patches / _apply_class_patches (monkey-patch)__init__.py:219-220
    9      _C._initExtension() (C++ 设备初始化)                __init__.py:239
    10     注册 hccl/lccl 分布式后端                            __init__.py:275
    11     atexit 注册 _npu_shutdown                           __init__.py:296

关键设计: 通过 `rename_privateuse1_backend` 将 PyTorch 的 PrivateUse1
dispatch key 重命名为 "npu", 使 `torch.device("npu")` 生效。这是 PyTorch
out-of-tree 后端的标准集成方式。

### 5.2 C++ 绑定入口

`torch_npu._C` 的 CPython 入口为 `PyInit__C` (InitNpuBindings.cpp:220)。
`initModule()` 聚合以下 PyMethodDef 组:

    绑定组                 来源文件                        内容
    ──────────────────────────────────────────────────────────────────────
    TorchNpuMethods        InitNpuBindings.cpp            shutdown 等核心方法
    THNPModule methods     csrc/npu/Module.cpp            设备/内存/流管理 (主体)
    profiler methods       csrc/profiler/init.cpp         性能分析 API
    distributed methods    csrc/distributed/Init.cpp      HCCL/LCCL 初始化
    autocast methods       csrc/utils/AutocastMode.cpp    混合精度
    flopcount methods      csrc/flopcount/Init.cpp        算力统计
    logging methods        csrc/logging/Init.cpp          日志控制
    reductions methods     csrc/ipc/StorageSharing.cpp    IPC 张量共享
    custom_dtype methods   csrc/custom_dtype/Init.cpp     HiFloat8 等
    afd methods            csrc/afd/Init.cpp              故障诊断

注册的 Python 类型对象:
- THNPStream (流), THNPEvent (事件), THNPGraph (图), THNPMemPool (内存池)
- THNPMLIR (MLIR 编译), THDVM (DVM 编译)

### 5.3 算子派发链

    层级     机制                                        引用
    ──────────────────────────────────────────────────────────────────────
    声明     npu_native_functions.yaml (95行)            csrc/aten/npu_native_functions.yaml
             backend: NPU, namespace: at_npu::native
             声明约60个原生算子 (clone/copy_/empty/resize_等)

    代码生成  generate_code.sh -> torchnpugen              generate_code.sh:26-36
             生成 RegisterNPU.cpp, NPUNativeFunctions.h

    注册     TORCH_LIBRARY_IMPL(_, PrivateUse1, m)       VariableFallbackKernel.cpp:259
             未在 yaml 中声明的算子 fallback 到 CPU
             TORCH_LIBRARY_IMPL(_, AutogradPrivateUse1)  VariableFallbackKernel.cpp:233
             autograd 通用 fallback

    实现     op-plugin (third_party/op-plugin/)           最大子模块
             大部分算子的实际内核实现在此
             通过 ACL API 调用昇腾算子库

    特殊注册  BinaryOps.cpp: CPU dispatch key 补丁        csrc/aten/BinaryOps.cpp:24
             PinMemory.cpp: BackendSelect dispatch       csrc/aten/PinMemory.cpp:43
             AutoCastOps.cpp: autocast 策略              csrc/aten/AutoCastOps.cpp:20

调用路径:
Python `torch.add(a, b)` -> PyTorch dispatcher 按 DispatchKey 路由
-> PrivateUse1 key -> yaml 注册的 native 实现 (op_plugin)
-> ACL op_compiler -> 昇腾硬件执行

未注册算子: fallback 到 `at::native::cpu_fallback` 在 CPU 执行。

### 5.4 核心数据结构

    类型                   基类                 文件                        关键字段
    ──────────────────────────────────────────────────────────────────────
    NPUStorageDesc         (plain struct)       core/NPUStorageImpl.h:16   origin_format_, npu_format_
                                                                           base_sizes_, storage_sizes_
    NPUStorageImpl         c10::StorageImpl     core/NPUStorageImpl.h:31   npu_desc_, unique_id_
    NPUTensorImpl          c10::TensorImpl      core/NPUTensorImpl.h:11   dispatch key:
                                                                           {PrivateUse1, AutogradPrivateUse1}

NPUStorageDesc 是 CANN 格式系统的桥梁:
- origin_format_: 用户逻辑格式 (如 NCHW)
- npu_format_: 硬件内部格式 (如 5HD, FZ, NZ)
- storage_sizes_: 内部格式下的实际存储尺寸

内存分配链:
`make_npu_storage_impl` (NPUStorageImpl.cpp:28)
-> allocator 参数 (通常 NPUCachingAllocator)
-> `aclrtMalloc` (ACL 运行时)

### 5.5 架构分层总览

    ┌─────────────────────────────────────────────────────┐
    │ Python Layer                                        │
    │ __init__.py -> npu/ -> contrib/ -> distributed/     │
    │ _inductor/ (编译器) | profiler/ | utils/ (patches)  │
    ├─────────────────────────────────────────────────────┤
    │ Binding Layer: _C (InitNpuBindings.cpp)             │
    │ THNPStream / THNPEvent / THNPGraph / THNPMemPool    │
    ├─────────────────────────────────────────────────────┤
    │ C++ Core (csrc/)                                    │
    │ aten/ ─── framework/ ─── core/npu/                  │
    │ (算子注册)  (OpCommand)   (设备/流/内存)            │
    │                                                     │
    │ distributed/ (HCCL/LCCL)  │ inductor/ (AOT)         │
    │ profiler/   │ toolkit/    │ ipc/ │ sanitizer/       │
    ├─────────────────────────────────────────────────────┤
    │ External: ACL runtime | HCCL | GE | op-plugin       │
    │           DVM | TorchAIR | Tensorpipe               │
    └─────────────────────────────────────────────────────┘


## 6. 公共基础设施

### 6.1 命名空间

    命名空间                  覆盖模块           用途
    ──────────────────────────────────────────────────────────────────────
    c10_npu::                core/npu/          设备/流/事件/内存/系统控制
    at_npu::native           aten/, framework/  ATen 算子实现与分发
    torch_npu::              core/, csrc根      Storage/Tensor/Bridge
    torch_npu::profiler      profiler/          性能分析
    torch_npu::toolkit       toolkit/           npu_profiler

### 6.2 关键单例/全局状态

    单例                      文件                           职责
    ──────────────────────────────────────────────────────────────────────
    NpuSysCtrl::GetInstance() core/npu/sys_ctrl/npu_sys_ctrl ACL 初始化/终结
    NPUCachingAllocator       core/npu/NPUCachingAllocator   设备内存池
    CachingHostAllocator      core/npu/CachingHostAllocator  主机内存池
    OptionsManager            core/npu/register/OptionsManager 运行时选项

### 6.3 编译器后端 (_inductor/)

_inductor/ 是 Python 侧最大的模块 (28,407 LOC), 支持三种编译后端:

    后端        入口                       产物
    ──────────────────────────────────────────────────────────────────────
    MLIR        ascend_npu_ir/ (13,519 LOC)  通过 torch-mlir 生成 NPU IR
    DVM         dvm/ (1,963 LOC)             DVM 向量化代码
    Default     codegen/ (5,816 LOC)         基于 Triton 模板的代码生成

注册方式: `torch._inductor.select_algorithm` 框架的 NPU 后端,
通过 `_inductor/__init__.py` 注册到 PyTorch Inductor。
