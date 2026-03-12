# CANN C++ Conventions (hccl & hcomm)

本文档从hccl和hcomm两个仓库的系统性代码分析中提取，供vibe-coding skill消费。
每条规范可直接转化为编码动作或检查项。

---

# 第1章: 全局架构概览

## 1.1 仓库结构与层次

### R-1.1: hcomm三层架构

适用范围: hcomm
强制程度: 强制

hcomm采用严格的三层架构，依赖方向自上而下单向:

```
algorithm(882文件) → framework(567文件) → platform(394文件)
                                        + legacy(1221文件, 旧代码)
```

- algorithm: 通信算法实现(AllReduce/AllGather/ReduceScatter等)，含base(模板)和impl(具体实现)
- framework: 通信域管理、资源编排、API入口、AICPU协调
- platform: 通信原语、传输通道、内存管理、硬件抽象
- legacy: 旧版实现，与framework/platform并行存在，按芯片代际分流

新代码应放入对应层次，不要在上层模块直接依赖下层的内部实现头文件。

出处: hcomm src/ 目录结构，4466个C++文件

### R-1.2: hccl算子统一结构

适用范围: hccl
强制程度: 强制

hccl由common(22文件)和13个算子(408文件)组成。每个算子必须遵循统一的三件套目录结构:

```
ops/<algo>/
  ├── *_op.h/cc          # 算子接口
  ├── executor/          # 执行器(编排通信步骤)
  ├── selector/          # 算法选择器(根据拓扑选算法)
  └── template/          # 硬件特定模板(aicpu/aiv/ccu)
```

新增算子必须提供executor、selector、template三个子模块。

出处: hccl src/ops/ 下13个算子目录全部遵循此模式

### R-1.3: hccl单向依赖hcomm

适用范围: 项目级
强制程度: 强制

hccl单向依赖hcomm，通过5大接口类别调用:
1. 通信域生命周期 (hccl_comm.h, ~30个函数)
2. 资源管理 (hccl_res.h, ~15个函数)
3. 拓扑查询 (hccl_rank_graph.h, ~10个函数)
4. 数据面通信原语 (hcomm_primitives.h, ~30个函数)
5. CCU资源 (hccl_ccu_res.h, 3个函数)

hcomm源码不包含hccl的实现头文件。libhccl.so链接libhcomm.so。

```
// good: hccl调用hcomm公共接口
#include "hccl/hcomm_primitives.h"
CHK_RET(HcommWriteOnThread(thread, channel, dst, src, len));

// bad: hcomm反向依赖hccl内部头文件
#include "ops/all_reduce/executor/all_reduce_executor.h"  // 禁止
```

出处: hccl/CMakeLists.txt target_link_libraries; hccl src/ops/op_common/inc/alg_param.h 包含6个hcomm头文件

## 1.2 编译配置

### R-1.4: C++14标准 + 严格编译选项

适用范围: 项目级
强制程度: 强制

两仓统一使用C++14，编译选项:

```
-std=c++14 -Werror -Wall -O3
-fno-common -fno-strict-aliasing
-fstack-protector-all -D_FORTIFY_SOURCE=2
```

-Werror意味着所有警告都是编译错误。新代码不能引入任何编译警告。

出处: hccl/CMakeLists.txt, hcomm/CMakeLists.txt

### R-1.5: Stub库机制

适用范围: 项目级
强制程度: 强制

对无源码的外部库生成stub进行链接。hccl stub: ccl_kernel。hcomm stub: ascend_hal, slog, aicpu_sharder, c_sec, unified_dlog, mmpa, ascendcl, tsdclient, runtime等13个。

新增外部API依赖时，需检查该API是否已有stub，否则补充stub声明。

出处: hccl/cmake/, hcomm/cmake/

### R-1.6: Host/Device编译分裂

适用范围: 项目级
强制程度: 强制

同一模块的Host和Device编译产出不同实现:
- `xxx.cc`: 共享实现(构造/析构/简单getter)
- `xxx_host.cc`: Host侧完整实现
- `xxx_device.cc`: Device侧stub(返回HCCL_SUCCESS或打印WARNING)

条件编译宏: AICPU_COMPILE(AICPU kernel编译), HOST_COMPILE(Host端编译), CCL_KERNEL(kernel编译)

```cpp
// op_base_device.cc — Device侧是空壳
HcclResult HcclAllReduceInner(...) {
    HCCL_WARNING("device side not support");
    return HCCL_SUCCESS;
}
```

出处: hcomm中6组Host/Device文件对(hccl_communicator, hccl_communicator_attrs, op_base等)

## 1.3 公共基础设施

### R-1.7: HcclResult错误码体系

适用范围: 项目级
强制程度: 强制

统一使用HcclResult枚举(25个错误码)，定义在hcomm的include/hccl/hccl_types.h。hccl依赖此定义，不重定义。

核心错误码:
- HCCL_SUCCESS(0): 成功
- HCCL_E_PARA(1): 参数错误
- HCCL_E_PTR(2): 空指针
- HCCL_E_INTERNAL(5): 内部错误
- HCCL_E_NOT_SUPPORT(6): 不支持
- HCCL_E_AGAIN(20): 重试
- HCCL_E_SUSPENDING(22): 暂停中

新代码必须使用已有错误码，不要自定义数值。

出处: include/hccl/hccl_types.h:23-51

### R-1.8: 基础类型别名

适用范围: 项目级
强制程度: 强制

使用项目定义的类型别名替代标准整数类型:

```cpp
using s8/s16/s32/s64 = signed char/short/int/long long;
using u8/u16/u32/u64 = unsigned char/short/int/long long;
using char_t = char;
```

通信句柄统一为void*: HcclComm, HcclConn, HcclRtStream, HcclRtNotify, HcclRtEvent

出处: src/pub_inc/hccl_common.h:37-49, pkg_inc/hccl/base.h:19-26

### R-1.9: 数据类型枚举(18种)

适用范围: 项目级
强制程度: 强制

HcclDataType支持18种数据类型: INT8/16/32/64, UINT8/16/32/64, FP16/32/64, BFP16, INT128, HIF8, FP8E4M3/E5M2/E8M0。

数据类型枚举值不等于字节数。获取字节数必须通过SIZE_TABLE查表:

```cpp
// good
u64 count = totalBytes / SIZE_TABLE[dataType];

// bad — 枚举值不等于字节数，这是已知bug
u64 count = totalBytes / dataType;
```

出处: include/hccl/hccl_types.h:90-109; hccl_primitive_remote.cc:87-88(bug实例)

## 1.4 核心设计模式

### R-1.10: V1/V2双代架构并存

适用范围: 项目级
强制程度: 强制

910B/910_93走V1路径，910_95走V2路径。分派通过HCCLV2_FUNC_RUN宏实现:

```cpp
#define HCCLV2_FUNC_RUN(func, ...) \
    do { \
        const char *socNamePtr = aclrtGetSocName(); \
        CHK_PTR_NULL(socNamePtr); \
        if (IsSupportHCCLV2(socNamePtr)) { \
            return func; \
        } \
    } while (0)
```

V2函数声明为`__attribute__((weak))`，链接V2库时覆盖，未链接时为nullptr。

一旦走V2路径，不会fallback到V1。新功能需明确标注支持的芯片代际。

出处: param_check_basic_v2.h:14-21; HCCLV2_FUNC_RUN在op_base.cc中出现40次，在framework中出现145次

### R-1.11: 四大核心设计模式

适用范围: 项目级
强制程度: 推荐

1. Template Method: ExecutorBase/AlgTemplateBase/CommBase/CollNativeExecutorBase定义算法骨架，子类覆盖特定步骤
2. Strategy: AutoSelectorBase按设备类型分策略，TopoMatchBase按拓扑分策略
3. Registry: SelectorRegistry + REGISTER_SELECTOR宏实现自动注册
4. State: OpRetryBase + Agent/Server状态子类实现故障恢复状态机

新算子的executor、selector、template必须继承对应基类。

出处: hccl ops/op_common/executor/executor_base.h(Template Method); ops/op_common/selector/selector_registry.h(Registry)

---

# 第2章: Platform层(hcomm)惯用法与规范

Platform层包含5个功能域: comm_primitive(通信原语)、resource(资源管理)、hccp(协议栈)、task(任务调度)、debug(维测)。

## 2.1 错误处理

### P-ID-01: CHK_RET 100%覆盖

适用范围: 项目级
强制程度: 强制

每个返回HcclResult的函数调用必须用CHK_RET检查。每个指针参数入口必须CHK_PTR_NULL。

例外:
- 析构/清理路径中CHK_RET会中断后续清理,应使用(void)忽略返回值或记录后继续
- extern "C"函数返回unsigned int等非HcclResult类型时,CHK_RET不适用(类型不匹配),需手动if检查
- void返回值的STL函数(如std::copy)不需要也不能CHK_RET

```cpp
// good
HcclResult Foo(void *ptr, StreamHandle stream) {
    CHK_PTR_NULL(ptr);
    CHK_PTR_NULL(stream);
    CHK_RET(Bar(ptr));
    CHK_RET(Baz(stream));
    return HCCL_SUCCESS;
}

// bad — 遗漏检查
HcclResult Foo(void *ptr) {
    Bar(ptr);              // 返回值未检查
    return HCCL_SUCCESS;
}
```

CHK_RET在hcomm中出现14,173次。Platform层comm_primitive 3个文件中99次CHK_RET/CHK_PTR_NULL。

出处: src/pub_inc/log.h:139-279

### P-ID-02: C ABI门面(extern "C" + void*句柄)

适用范围: hcomm Platform层
强制程度: 强制

所有对外接口头文件使用extern "C"包装，参数使用void*不透明句柄，实现内部用reinterpret_cast还原类型:

```cpp
// 头文件: extern "C" 声明
#ifdef __cplusplus
extern "C" {
#endif
HcclResult HcclLocalCopy(StreamHandle streamHandle, HcclBuf *dst, HcclBuf *src);
#ifdef __cplusplus
}
#endif

// 实现: reinterpret_cast还原
HcclResult HcclLocalCopy(StreamHandle streamHandle, HcclBuf *dst, HcclBuf *src) {
    hccl::Stream *stream = reinterpret_cast<hccl::Stream*>(streamHandle);
    // ...
}
```

类型映射: StreamHandle↔Stream*, HcclMemTransport↔Transport*, DispatcherCtxPtr↔DispatcherCtx*

出处: pub_inc/new/ 4个头文件; comm_primitive/ 3个实现文件中35次reinterpret_cast

### P-ID-03: owner_ + move语义的内存管理

适用范围: hcomm Platform层
强制程度: 强制

DeviceMem/HostMem使用owner_布尔标志区分所有权，不使用shared_ptr:
- `alloc(size)`: 创建拥有所有权的对象(owner_=true)
- `create(ptr, size)`: 包装已有指针(owner_=false)
- `range(offset, size)`: 切割子块(owner_=false)

复制构造产生非拥有副本，移动构造转移所有权。析构时仅owner释放。

```cpp
DeviceMem mem = DeviceMem::alloc(4096);      // owner_=true, 析构释放
DeviceMem ref = DeviceMem::create(ptr, len);  // owner_=false, 析构不释放
DeviceMem sub = mem.range(0, 1024);           // owner_=false
```

出处: platform/resource/mem/mem_device.cc:15-130, mem_host.cc:15-125

### P-ID-04: new(std::nothrow) + CHK_PTR_NULL

适用范围: 项目级
强制程度: 强制

堆分配使用new(std::nothrow)替代标准new，分配后立即CHK_PTR_NULL:

```cpp
// good
auto *obj = new (std::nothrow) DispatcherCtx(devPhyId);
CHK_PTR_NULL(obj);

// bad — 标准new在OOM时抛异常
auto *obj = new DispatcherCtx(devPhyId);
```

出处: hccl_dispatcher_ctx.cc:80; NEW_NOTHROW宏定义于log.h; framework层127次使用

### P-ID-05: Pimpl模式(_pub.h)

适用范围: hcomm
强制程度: 推荐

公共头文件后缀_pub.h，暴露接口；实现细节隐藏在impl类中:

```
transport_base_pub.h  → 公共接口
transport_base.cc     → 实现
comm_base_pub.h       → 公共接口
comm_base.cc          → 实现
```

出处: platform/resource/transport/transport_base_pub.h; algorithm/base/inc/comm_base_pub.h

### P-ID-06: 引用计数去重

适用范围: hcomm Platform层
强制程度: 推荐

RmaBufferMgr使用区间树+引用计数实现内存注册去重。Referenced基类提供Ref/Unref/Count/Clear/IsZero接口。MemMappingManager缓存Host→Device地址映射也使用引用计数。

同一(addr, size)可被多次Add，Del时引用计数减1而非直接释放。

出处: src/pub_inc/hccl_common.h:334-369(Referenced类); platform/resource/mem/mem_mapping_manager.cc

### P-ID-07: 日志级别规范

适用范围: 项目级
强制程度: 推荐

- HCCL_DEBUG: 高频数据面操作(远端原语入口)，使用UNLIKELY()优化分支预测
- HCCL_INFO: 本地操作入口、管理操作
- HCCL_WARNING: 非致命异常、可恢复错误、HCCL_E_AGAIN场景
- HCCL_ERROR: 致命错误(支持ErrToWarn开关降级)
- HCCL_RUN_INFO: 操作级追踪

日志格式: `[函数名]关键参数名[值]`:
```cpp
HCCL_DEBUG("[HcclRemoteWrite]streamHandle[%p], memTransport[%p], len[%llu].",
    streamHandle, memTransport, rmtBuf->len);
```

出处: 日志宏在hcomm中21,460次调用覆盖1,094个文件

### P-ID-08: LIKELY/UNLIKELY分支预测

适用范围: 项目级
强制程度: 推荐

热路径中使用LIKELY/UNLIKELY优化分支预测:

```cpp
if (UNLIKELY(commId == nullptr)) { /* 异常路径 */ }
if (LIKELY(gDispatcherCtx != nullptr)) { /* 快速路径 */ }
```

出处: hccl_dispatcher_ctx.cc 3处; HCCL_DEBUG宏内置UNLIKELY检查

### P-ID-09: securec替代标准C函数

适用范围: 项目级
强制程度: 强制

C接口层使用华为securec安全函数替代标准C函数:
- memcpy_s 替代 memcpy
- sprintf_s / snprintf_s 替代 sprintf / snprintf
- memset_s 替代 memset

用CHK_SAFETY_FUNC_RET检查返回值(期望EOK):

```cpp
CHK_SAFETY_FUNC_RET(memcpy_s(dst, dstSize, src, srcSize));
```

C++代码优先使用类型安全的stream/string操作替代snprintf_s(如std::ostringstream)。securec规则主要适用于C接口层和与C库交互的场景。

出处: src/pub_inc/log.h CHK_SAFETY_FUNC_RET定义

### P-ID-10: 4GB SDMA切分

适用范围: hcomm Platform层
强制程度: 强制

单次SDMA传输上限4GB。超过4GB的数据搬运需切分为多个小于4GB的块:

出处: platform层SDMA传输实现

### P-ID-11: 内存屏障平台差异

适用范围: hcomm Platform层
强制程度: 强制

x86使用compiler fence(编译器屏障)，aarch64使用dsb st(数据同步屏障):

```cpp
#if defined(__x86_64__)
    std::atomic_thread_fence(std::memory_order_seq_cst);
#elif defined(__aarch64__)
    asm volatile("dsb st" ::: "memory");
#endif
```

出处: platform层HDC实现, hdc.cc

## 2.2 架构模式

### P-AR-01: Transport继承体系(9种传输类型)

适用范围: hcomm Platform层
强制程度: 强制

TransportBase是传输层基类，派生9种具体传输:
IBV_EXP, P2P, HOST_SHM, HOST_TCP, ROCE, HETEROG_P2P/ROCE, DEVICE_P2P/IBVERBS/DIRECT

transport/目录62个文件，是platform层最大子模块。

出处: platform/resource/transport/transport_base_pub.h:52

### P-AR-02: 三种执行后端

适用范围: hcomm Platform层
强制程度: 强制

- DispatcherPub: 通过thread_local上下文获取，用于本地原语
- Graph模式: FFTS缓存编译结果
- AiCpu: 通过kernel launch执行

出处: comm_primitive/hccl_dispatcher_ctx.cc; hccl_primitive_local.cc

### P-AR-03: hccp C/S RPC模式

适用范围: hcomm Platform层 hccp模块
强制程度: 强制

hccp是纯C代码(内核驱动风格)，实现Host-Device协议栈。其余Platform模块为C++。

出处: platform/hccp/

## 2.3 业务知识

### P-BIZ-01: DMA默认策略

适用范围: hcomm Platform层
强制程度: 强制

- P2P(HCCS)链路: 默认GET模式(接收方拉取) — HCCS支持高效远端读
- DEV_NET(RoCE)链路: 默认PUT模式(发送方推送) — RoCE远端读需额外往返

出处: legacy prim_rules.cc:438-478

### P-BIZ-02: Notify三轮握手

适用范围: hcomm Platform层
强制程度: 强制

通信原语使用三轮(可选四轮)握手:
1. PostReady/WaitReady — 准备就绪
2. Write/Read — 数据传输
3. PostFin/WaitFin — 传输完成
4. (可选) PostFinAck/WaitFinAck — DEV_NET额外确认

WriteWithNotify优化: 将Write+PostFin合并为一个硬件操作(部分平台支持)。

出处: prim_rules.cc:130-178

### P-BIZ-03: InlineReduce降级

适用范围: hcomm Platform层
强制程度: 推荐

硬件不支持InlineReduce时的降级策略:
- SendReduce: 先Write到remoteSrc，再远端LocalReduce
- RecvReduce: 先Read到localSrc，再本地LocalReduce

是否支持取决于数据类型、归约操作、链路类型三个条件。

出处: prim_rules.cc:66-79, 480-570

### P-BIZ-04: 链路分层

适用范围: 项目级
强制程度: 强制

三层网络:
- L0(server内): HCCS(片上高速互连) / SIO / PCIe
- L1(superPod内/跨server): HCCS(910_93) / RoCE(910B)
- L2(跨superPod): RoCE

链路类型选择依赖设备类型和网络层级。910B: L0=HCCS mesh, L1=RoCE。910_93: L0=全互连, L1=HCCS, L2=RoCE。

出处: rank_graph.cc GetCommProtocolFromRankInfo行206-248

## 2.4 本层反模式

### P-AP-01: 全局可变状态

适用范围: hcomm Platform层
强制程度: 避免

DispatcherCtx使用全局thread_local + 全局map双层存储，存在并发和泄漏风险。
AcquireDispatcherCtx创建的ctx只存thread_local不注册到全局map，线程结束后泄漏。

正确做法: 新创建的资源必须同时注册到全局管理容器。

出处: hccl_dispatcher_ctx.cc:213-231

### P-AP-02: reinterpret_cast无类型安全

适用范围: hcomm Platform层
强制程度: 避免

void*句柄还原时使用reinterpret_cast，无任何运行时检查。任何非法值导致未定义行为。

正确做法: 在Debug模式加magic number校验或使用tagged pointer。

出处: comm_primitive/ 3个文件35次reinterpret_cast

### P-AP-03: RemoteReadReduce用dataType枚举做除法(bug)

适用范围: hcomm Platform层
强制程度: 避免(已确认bug)

```cpp
// bug: dataType是枚举值，不是字节数
rmtBuf->len / reduceInfo.dataType

// 正确: 通过SIZE_TABLE查表
src->len / SIZE_TABLE[reduceInfo.dataType]
```

出处: hccl_primitive_remote.cc:87-88 vs hccl_primitive_local.cc:64

### P-AP-04: DispatcherCtx泄漏

适用范围: hcomm Platform层
强制程度: 避免

CreateDispatcherCtx绑定失败后返回SUCCESS并静默复用已有ctx，掩盖竞态问题。
DestroyDispatcherCtx使用函数级static mutex，所有通信域销毁串行化。

正确做法: 绑定失败应返回特定错误码让调用方感知。

出处: hccl_dispatcher_ctx.cc:90-103, 129-130

### P-AP-05: busy-wait无退避

适用范围: hcomm Platform层
强制程度: 避免

KFC协议轮询等待、HDC通道轮询均为tight loop无sleep/yield。

正确做法: 使用指数退避或条件变量。

出处: communicator_impl.cc:1793+; hdc.cc

### P-AP-06: close()的EINTR重试

适用范围: hcomm Platform层
强制程度: 注意

close(fd)在EINTR后不应重试(POSIX规定close后fd状态不确定)。

出处: platform层socket实现

---

# 第3章: Framework层(hcomm)惯用法与规范

Framework层包含567个文件，核心模块: communicator/(通信域管理)、op_base/(API入口)、hcom/(HCOM入口)、cluster_maintenance/(故障恢复)。

## 3.1 错误处理

### FW-ID-01: CHK_RET链式错误传播

适用范围: 项目级
强制程度: 强制

CHK_RET在framework层出现5233次覆盖228个文件。是最核心的惯用法，无例外。

出处: framework层228个文件，5233次使用

### FW-ID-02: RPT_INPUT_ERR + CHK_PTR_NULL配对

适用范围: hcomm Framework层 API入口
强制程度: 强制

公共API入口每个参数先RPT_INPUT_ERR上报结构化错误(面向IDE)，再CHK_PTR_NULL实际检查:

```cpp
RPT_INPUT_ERR(comm == nullptr, "EI0003",
    std::vector<std::string>({"ccl_op", "parameter", "value", "tips"}),
    std::vector<std::string>({"HcclAllReduceInner", "comm", "nullptr", "please check comm"}));
CHK_PTR_NULL(comm);
```

RPT_INPUT_ERR在framework层出现282次。两者必须成对出现。

出处: op_base.cc 68次RPT_INPUT_ERR; one_sided_service_adapt.cc

### FW-ID-03: HCCLV2_FUNC_RUN V1/V2分派

适用范围: hcomm Framework层
强制程度: 强制

每个API入口函数在V1逻辑前用HCCLV2_FUNC_RUN尝试V2分派。V2成功则直接return，不回退到V1。

framework层出现145次。位置应在参数校验之后(部分现有代码位置不一致)。

出处: op_base.cc 40次, framework层共145次

### FW-ID-04: EXECEPTION_CATCH异常捕获

适用范围: hcomm Framework层
强制程度: 强制

用EXECEPTION_CATCH宏捕获bad_alloc等标准异常，在分配失败时执行fallback:

```cpp
EXECEPTION_CATCH(
    threadMgr_ = std::make_unique<ThreadMgr>(...),
    return HCCL_E_PTR
);
```

framework层出现206次。用于make_unique/make_shared/push_back等可能抛异常的操作。

出处: independent_op模块29次; framework层共206次覆盖多个模块

### FW-ID-05: EXCEPTION_HANDLE桥接(C API层)

适用范围: hcomm Framework层 C API入口
强制程度: 强制

C API函数用EXCEPTION_HANDLE_BEGIN/END包裹，桥接C++异常到HcclResult返回码:

```cpp
EXCEPTION_HANDLE_BEGIN()
    // C++ 业务逻辑
EXCEPTION_HANDLE_END(funcName)
```

framework层出现89次，集中在C API胶水层。

出处: one_sided_service_adapt.cc 34处; framework C API文件

### FW-ID-06: unique_ptr + 显式析构序

适用范围: hcomm Framework层
强制程度: 推荐

子系统成员使用unique_ptr管理，析构时按依赖顺序显式reset:

```cpp
class CommunicatorImpl {
    unique_ptr<StreamManager>     streamManager;
    unique_ptr<SocketManager>     socketManager;
    unique_ptr<RmaConnManager>    rmaConnectionManager;
    // 析构: 先reset依赖者，再reset被依赖者
};
```

出处: communicator_impl.h:393-418(20+ unique_ptr成员)

### FW-ID-07: 双轨错误处理(CCU路径)

适用范围: hcomm Framework层 CCU相关
强制程度: 推荐

legacy层的三层异常捕获:
```cpp
try { /* 业务逻辑 */ }
catch (HcclException &e) { PrintBackTrace(e); return e.GetErrorCode(); }
catch (exception &e) { return HCCL_E_INTERNAL; }
catch (...) { return HCCL_E_INTERNAL; }
```

与CHK_RET返回码模式并存。Legacy代码保留异常，新代码(platform新接口)只用返回码。

出处: communicator_impl.cc:98-122

### FW-ID-08: MAKE_ENUM宏

适用范围: hcomm
强制程度: 推荐

生成类型安全的枚举类，自带Describe()方法:

```cpp
MAKE_ENUM(PrimType, POST_TO, WAIT_FROM, LOCAL_COPY, LOCAL_REDUCE, SEND, RECV, ...)
// 自动生成 PrimType::POST_TO 等枚举值和 Describe(PrimType) 函数
```

framework层出现19次。

出处: primitive.h:31; legacy层多个枚举定义

### FW-ID-09: Entry/Exit日志对

适用范围: hcomm Framework层 API入口
强制程度: 推荐

API入口函数在开始和结束各记一条日志，形成配对:

```cpp
if (GetExternalInputHcclEnableEntryLog()) {
    std::string logInfo = "Entry-AllReduce:" + stackLogBuffer;
    hcclComm->SaveTraceInfo(logInfo);
}
// ... 实际操作 ...
if (GetExternalInputHcclEnableEntryLog()) {
    std::string endInfo = "HcclAllReduceInner:success,take time: " + duration;
    hcclComm->SaveTraceInfo(endInfo);
}
```

使用栈上buffer(char stackLogBuffer[LOG_TMPBUF_SIZE])避免堆分配。

出处: op_base.cc 32次GetExternalInputHcclEnableEntryLog检查

### FW-ID-10: (void)忽略返回值

适用范围: hcomm Framework层
强制程度: 推荐

析构/清理路径中对允许失败的调用用(void)显式忽略返回值，避免-Werror报警:

```cpp
(void)HcclCommDestroy(pComm.get());
```

出处: framework层多处析构路径

### FW-ID-11: BinaryStream序列化

适用范围: hcomm Framework层
强制程度: 推荐

使用BinaryStream做通信数据结构的序列化/反序列化，支持operator<</>>:

```cpp
BinaryStream stream;
stream << rankSize << rankId << configData;
// 传输...
stream >> rankSize >> rankId >> configData;
```

用于Snapshot保存/恢复和Host→Device资源打包。

出处: src/common/binary_stream.h; op_base_v2.cc Snapshot机制

### FW-ID-12: Per-Device Singleton

适用范围: hcomm Framework层
强制程度: 推荐

使用static数组[MAX_MODULE_DEVICE_NUM]实现per-device单例:

```cpp
static PreemptPortManager instance[MAX_MODULE_DEVICE_NUM];
static PreemptPortManager& GetInstance(u32 deviceId) {
    return instance[deviceId];
}
```

framework层出现54次。避免了全局单例的设备间干扰。

出处: preempt_port_manager.cc GetInstance(); CommTopoDesc; CommManager

### FW-ID-13: do-while(0) + errorFlag

适用范围: hcomm Framework层
强制程度: 推荐

通信域初始化使用do-while(0) + errorFlag模拟goto清理:

```cpp
bool errorFlag = false;
do {
    ret = step1();
    CHK_PRT_BREAK(ret != HCCL_SUCCESS, HCCL_ERROR(...), errorFlag = true);
    ret = step2();
    CHK_PRT_BREAK(ret != HCCL_SUCCESS, HCCL_ERROR(...), errorFlag = true);
} while (0);
if (errorFlag) {
    (void)HcclCommDestroy(pComm.get());
    return ret;
}
```

出处: op_base.cc InitCommClusterInfo(L396-531) 13步; 3处初始化函数

### FW-ID-14: Packed Struct

适用范围: hcomm Framework层
强制程度: 推荐

Host↔Device传输的结构体使用`__attribute__((packed))`确保内存布局一致:

出处: framework层H2D资源序列化相关结构体

## 3.2 架构模式

### FW-AR-01: 三层委托 + Facade

适用范围: hcomm Framework层 communicator模块
强制程度: 强制

```
外部API → hcclComm(参数校验+委托, 3168行)
        → HcclCommunicator(核心协调, 9152行host)
        → HcclCommunicatorAttrs(拓扑属性, 1108行)
```

hcclComm持有shared_ptr<HcclCommunicator>。所有实际工作委托给communicator_:

```cpp
HcclResult hcclComm::AllReduce(tag, input, output, count, dataType, op, stream) {
    CHK_RET(communicator_->CheckCount(count));
    CHK_RET(communicator_->CheckDataType(dataType, true));
    return communicator_->AllReduce(tag, input, output, count, dataType, op, stream);
}
```

出处: hccl_comm.cc 3168行; hccl_communicator_host.cc 9152行

### FW-AR-02: V1/V2双代并存的接口风格

适用范围: hcomm Framework层
强制程度: 强制

两套API入口共存:
- op_base(NCCL风格): HcclComm comm(void*句柄), 自动生成tag, communicator->LoadOpbasedCollOp()
- hcom(HCOM风格): const char *group(字符串组名), 调用方传tag, hcclComm->LoadOffloadCollOp()

V1: HcclComm→hcclComm*→communicator_→HcclCommunicator
V2: HcclComm→直接static_cast<HcclCommunicator*>(comm)

两代的HcclComm句柄指向不同类型对象，通过SoC类型分隔，混用会类型混淆。

出处: op_base.cc 4695行; legacy/entrance/op_base/op_base_v2.cc 2902行

### FW-AR-03: 编译器式CCU指令编排

适用范围: hcomm Framework层 legacy
强制程度: 推荐

Legacy层实现了"原语编译器"，将高层通信原语翻译为底层硬件指令:

```
PrimQueue → PrimTranslator::Translate() → InsQueue
```

指令按优先级排序: LOCAL_COPY(100) > POST_TO(90) > WAIT_FROM(85) > WRITE/READ(60) > POST_FIN(50)

出处: prim_translator.cc, prim_rules.cc(622行核心逻辑)

### FW-AR-04: Strategy模式(5处)

适用范围: hcomm Framework层
强制程度: 推荐

Strategy模式的应用:
1. AcceleratorState→3种CollService(CCU/AICPU/Host)
2. MulQpInfo优先级队列(4个Cache子类)
3. RankGraph V1完整实现 vs V2 pImpl委托
4. 芯片代际分流(HCCLV2_FUNC_RUN)
5. DMA模式选择(PUT/GET/DEFAULT)

出处: communicator_impl.cc collServices map; multi_qpInfo_manager.h; rank_graph.h/rank_graph_v2.h

### FW-AR-05: 状态机驱动(7个)

适用范围: hcomm Framework层
强制程度: 推荐

framework层包含7个状态机:
- CommStatus: IDLE→READY→INUSE→READY / COMM_ERROR
- OpRetryBase: Agent/Server各5种状态
- DeviceStatus: IDLE→RECOVERED→READY
- KFC协议: Command/Status配对

StateGuard RAII保护comm状态:
```cpp
StateGuard<hccl::hcclComm, HcclCommState> guard(hcclComm, HcclCommState::INUSE);
// 函数返回时自动恢复状态
```

出处: hccl_communicator.cc CommStatus; opretry_base.h; op_base_host.cc StateGuard(13处)

### FW-AR-06: API入口13步模板

适用范围: hcomm Framework层 op_base
强制程度: 强制

所有集合通信API入口遵循相同的13步序列:
1. Group检查(hcclGroupDepth>0则延迟执行)
2. 计时+Capture检测
3. Zero count提前返回
4. RPT_INPUT_ERR + CHK_PTR_NULL参数校验
5. HCCLV2_FUNC_RUN V2分派
6. Lock + StateGuard
7. Tag构造("OpType_" + identifier)
8. 特定校验(per-op)
9. Entry日志
10. SetWorkflowMode + PrintMemoryAttr
11. 委托hcclComm执行
12. Profiling上报
13. Exit日志

新增集合操作API必须遵循此模板。

出处: op_base_host.cc HcclAllReduceInner(L72-179); op_base.cc 所有集合操作入口

## 3.3 业务知识

### FW-BIZ-01: 通信域层次

适用范围: hcomm Framework层
强制程度: 强制

四种通信域:
1. World: 全局通信域(所有rank)
2. SubComm: 子通信域(rank子集)
3. OneSided: 单边RDMA通信域(Put/Get语义)
4. Group: 类NCCL的GroupStart/End批量调度

通过hcclGroupMap(组名→通信域映射)管理。

出处: op_base.cc g_opHcomInfos; hccl_group.cc

### FW-BIZ-02: 操作矩阵(13种集合操作)

适用范围: 项目级
强制程度: 强制

支持的集合操作: AllReduce, AllGather, AllGatherV, ReduceScatter, ReduceScatterV, Broadcast, Scatter, Reduce, AlltoAll, AlltoAllV, AlltoAllVC, Send, Recv, BatchSendRecv

每种操作通过HcclCMDType枚举标识。DataTypeBitmap验证矩阵控制OpType x DataType的合法组合。

出处: include/hccl/hccl_types.h HcclCMDType; op_params_checker.h DataTypeBitmap

### FW-BIZ-03: Group异步语义

适用范围: hcomm Framework层
强制程度: 推荐

```
HcclGroupStart()
    HcclAllReduce(...)   // 延迟到taskAppend
    HcclSend(...)        // 延迟到taskAppend
HcclGroupEnd()           // 批量执行
```

嵌套计数hcclGroupDepth++/--，只有最外层GroupEnd触发执行。
P2P调度: myRank < peer → 先send后recv; myRank >= peer → 先recv后send。

出处: hccl_group.cc(492行全文)

### FW-BIZ-04: 故障恢复机制

适用范围: hcomm Framework层
强制程度: 推荐

KFC协议实现Host-AICPU的命令/状态交互:
- Suspend: H2D发NS_STOP_LAUNCH → 等STOP_LAUNCH_DONE
- Clean: 等STOP_LAUNCH_DONE → 发NS_CLEAN → 等CLEAN_DONE
- Resume: collService->Resume() + 重置flags

DiffRankUpdater支持64+1替换和Pod替换两种恢复策略。

出处: communicator_impl.cc:1775-1921; diff_rank_updater.cc

## 3.4 硬件抽象

### FW-HW-01: 芯片代际

适用范围: 项目级
强制程度: 强制

7种设备类型(DevType): 910, 310P3, 910B, 310P1, 910_93, NOSOC, 910_95

关键分界:
- 910B/910_93: V1架构
- 910_95: V2架构(多个功能返回NOT_SUPPORT)
- 910_93: 主力优化目标(ZeroCopy/SuperPod等专属优化)

出处: pkg_inc/hccl/dtype_common.h:22-32

### FW-HW-02: AICPU模型

适用范围: hcomm Framework层
强制程度: 推荐

AICPU是Device侧的ARM核心，每cluster 8核 x 2 cluster。
Host打包资源→HDC传输→AICPU ParsePackedData恢复7类资源(Stream/Notify/Link/Transport等)。

Kernel Launch固定模式: 记时→launch→sync→profiling上报。

出处: aicpu/communicator_impl_lite.cc; kernel_entrance.cc

### FW-HW-03: CCU硬件模型

适用范围: hcomm Framework层
强制程度: 推荐

CCU(Communication Control Unit)位于IO Die上，每IO Die一个CCU，双die配置。
6种CommEngine: CPU, CPU_TS, AICPU, AICPU_TS, AIV, CCU。
CCU Super Fast Load通过缓存CCU算子参数避免重复编排。

出处: include/hccl/hccl_res.h:72-80; ccu_super_fast_load.h

## 3.5 本层反模式

### FW-AP-01: God Class

适用范围: hcomm Framework层
强制程度: 避免

4个文件超过9000行:
- HcclCommunicator: 100+方法, 100+成员, 9152行(host) — communicator核心
- CommunicatorImpl(legacy): 20+子系统, 585行头文件, ~3200行实现
- op_base.cc: 4695行
- op_base_v2.cc: 2902行

正确做法: 按职责拆分为独立的Manager类(已有IndependentOp的Facade+Composition作为正面示范)。

出处: hccl_communicator_host.cc 9152行; communicator_impl.h 585行

### FW-AP-02: 代码重复

适用范围: hcomm Framework层
强制程度: 避免

初始化序列在3处逐行重复(每处约60行): InitCommClusterInfo/InitCommRootInfo/CreateSubComm。
日志模板在12+处copy-paste。

已确认的copy-paste bug: 5处结束日志使用了错误函数名(HcclSendV2/RecvV2等使用"HcclAllGatherVV2")。

正确做法: 提取公共函数(CreateAndRegisterComm等)。

出处: op_base_v2.cc行1969,2028,2091,2183,2240(copy-paste日志错误)

### FW-AP-03: 全局状态无锁

适用范围: hcomm Framework层
强制程度: 避免

7+处全局可变状态缺少锁保护:
- g_taskServiceMap: 无锁保护(op_base_v2.cc)
- hcclGroupDepth/hcclGroupLock: 全局变量(hccl_group.cc:18-21)
- g_hcclCommunicators: 声明但未使用的遗留全局数组
- static全局 g_launchStream/g_launchMutex

正确做法: 全局可变状态必须有mutex保护或使用atomic。

出处: op_base_v2.cc:43; hccl_group.cc:18-21

### FW-AP-04: reinterpret_cast滥用

适用范围: hcomm Framework层
强制程度: 避免

framework层1376次reinterpret_cast。特别危险的场景:
- hcom_v2 Graph接口: int64_t直接reinterpret_cast为HcclCommunicator*(13处)
- RankGraph: static_cast向下转型无安全检查(6处)

正确做法: 使用dynamic_cast + nullptr检查，或通过基类虚方法避免向下转型。

出处: hcom_v2.cc:781+; hccl_independent_rank_graph.cc:260+

### FW-AP-05: switch缺break

适用范围: hcomm Framework层
强制程度: 避免(已确认bug)

hccl_group.cc:383 switch语句缺break导致fall-through(已确认功能bug)。

正确做法: 所有switch case必须有break/return/fallthrough注释。

出处: hccl_group.cc:383

### FW-AP-06: 忙等待

适用范围: hcomm Framework层
强制程度: 避免

6处busy-wait无退避:
- hccl_group.cc:180 usleep轮询
- symmetric_memory_agent.cc SaluSleep(1000) spin-wait
- WaitAllCommReady: 持锁tight loop 10秒(阻塞所有hcclGroupMap操作)
- KFC协议轮询

正确做法: 使用condition_variable或指数退避。

出处: hccl_group.cc:180; symmetric_memory_agent.cc; op_base_v2.cc WaitAllCommReady

### FW-AP-07: getenv热路径调用

适用范围: hcomm Framework层
强制程度: 避免

getenv("HCCL_INDEPENDENT_OP")在每次API调用时执行(18+处)。getenv不是线程安全的(C标准)，且有性能开销。

正确做法: 启动时读取一次，缓存到static变量。

出处: hccl_independent_rank_graph.cc(18处); hccl_independent_op_mem.cc(2处)

### FW-AP-08: HcclGetOpArgsV2 null检查bug

适用范围: hcomm Framework层
强制程度: 避免(已确认bug)

```cpp
HcclOpArgs *opArgsMem = (HcclOpArgs *)malloc(sizeof(HcclOpArgs));
if (opArgs == nullptr) {   // BUG: 应检查 opArgsMem == nullptr
    return HCCL_E_INTERNAL;
}
```

检查了入参opArgs而非malloc返回的opArgsMem。

出处: op_base_v2.cc:1296-1310

### FW-AP-09: ContextManager析构不释放内存

适用范围: hcomm Framework层
强制程度: 避免

ContextManager析构函数为空(`~ContextManager() {}`)，但contextMap_中可能有未Destroy的Host/Device内存。

正确做法: 析构函数中遍历contextMap_释放所有未销毁的context。

出处: independent_op_context_manager.cc:36-60; independent_op_context_manager.h 析构函数

### FW-AP-10: ChannelManager无锁保护

适用范围: hcomm Framework层
强制程度: 避免

ChannelManager的channelHandleMap_/keyMap_/engineMap_等map都没有mutex保护，多线程并发调用会数据竞争。
对比: 同模块ContextManager每个方法都有lock_guard。

正确做法: 参照ContextManager添加mutex保护。

出处: channel_manager.h:96-109

---

# 第4章: Algorithm层(hcomm)惯用法与规范

hcomm算法层是最大的代码模块(882文件/122,497行), 分为base/(算法模板+拓扑基础设施)和impl/(算法选择+执行编排+资源管理)两部分。

### ALG-01: CHK_RET全覆盖

适用范围: hcomm Algorithm层
强制程度: 强制

所有函数调用返回值必须用CHK_RET包裹, 无例外。algorithm/层无一处裸调用。

```cpp
// good
CHK_RET(alg->Prepare(inputMem_, outputMem_, count_, ...));
CHK_RET(alg->RunAsync(rank, rankSize, links));

// bad
alg->RunAsync(rank, rankSize, links);  // 返回值被忽略
```

出处: 5476次/301文件, 辅助宏CHK_SMART_PTR_NULL 727次/213文件

### ALG-02: 三层Registry自注册工厂

适用范围: hcomm Algorithm层
强制程度: 强制

使用三个独立Registry, 全部基于Meyer's单例 + 文件级静态注册宏 + static_assert继承检查。每个.cc文件末尾固定放置一行注册宏。

| Registry | 宏 | 频次 | 键类型 | 值类型 |
|----------|-----|------|--------|--------|
| AlgTemplateRegistry | REGISTER_TEMPLATE | 104次 | TemplateType枚举 | unique_ptr\<AlgTemplateBase\> |
| CollAlgExecRegistry | REGISTER_EXEC | 148次 | string | unique_ptr\<CollExecutorBase\> |
| CollAlgOpRegistry | REGISTER_OP | 16次 | HcclCMDType枚举 | unique_ptr\<CollAlgOperator\> |

三级调用链: REGISTER_OP选operator -> SelectAlg产出algName字符串 -> REGISTER_EXEC创建executor -> GetAlgTemplate(531次/121文件)获取算法模板。

出处: alg_template_register.h, coll_alg_exec_registry.h, coll_alg_op_registry.h

### ALG-03: Template Method骨架(多层嵌套)

适用范围: hcomm Algorithm层
强制程度: 强制

Algorithm层使用多层嵌套的Template Method, 各层有固定的骨架步骤:

L1 CalcResRequest(资源计算):
  CalcScratchMemSize -> CalcStreamNum -> CalcNotifyNum -> CalcAivBufferRequest -> CalcCommInfo -> BuildResourceRequest

L3 Orchestrate(9种操作统一):
  startut -> tag/algRes赋值 -> 图模式/单卡/多卡分发 -> RunLoop/KernelRun -> 强制LaunchTaskExtend -> 结束日志

Orchestrate结尾的强制LaunchTaskExtend出现在所有9个操作基类中(35文件/57次), 注释标注"不要删除, 否则导致aicpu cache功能问题"。

出处: coll_native_executor_base.cc, coll_all_reduce_executor.cc等

### ALG-04: 两段式构造(Prepare)

适用范围: hcomm Algorithm层
强制程度: 强制

所有算法模板和executor使用两段式构造: 工厂创建空壳 -> Prepare()初始化参数。

```cpp
auto alg = AlgTemplateRegistry::Instance().GetAlgTemplate(templateType, dispatcher_);
alg->Prepare(reduceAttr_);
alg->Prepare(inputMem_, outputMem_, count_, dataType_, stream_, ...);
```

基类有约30个virtual Prepare重载(按参数数量1~19分组)。子类须用`using AlgTemplateBase::Prepare;`引入基类重载避免C++ name hiding(23个文件这样做)。新代码改用PrepareData结构体。

出处: alg_template_base_pub.h

### ALG-05: desc_描述符声明能力

适用范围: hcomm Algorithm层
强制程度: 强制

每个Executor在构造函数中通过desc_声明自己的能力: isAivMode, isAivCrossNode, isZeroCopy, deterministic, level1SupportedAlgos等。运行时SetAlgType()检查requested type是否在supported列表中, 不在则回退。

出处: coll_executor_base.cc:21-48, 142个Executor构造函数均设置desc_

### ALG-06: DMAReduceFlag_优化标志

适用范围: hcomm Algorithm层
强制程度: 强制

标记是否跳过User<->CCL buffer拷贝(OP_BASE+910_93下启用)。贯穿所有executor的Orchestrate/RunLoop/KernelRun。

出处: 173次/114文件

### ALG-07: Setter统一返回HcclResult

适用范围: hcomm Algorithm层
强制程度: 推荐

即使setter不可能失败, 也统一返回HcclResult, 使CHK_RET链式调用不中断:

```cpp
HcclResult SetCCLInBuffer(u64 size) { inCCLbufferSize_ = size; return HCCL_SUCCESS; }
```

出处: coll_executor相关头文件

### ALG-08: (void)标记未使用参数

适用范围: hcomm Algorithm层
强制程度: 推荐

基类虚方法的默认实现中统一使用`(void)param;`抑制编译器未使用参数警告。

出处: coll_native_executor_base.cc, executor_base.cc:20-28

### ALG-09: 中文星号分隔注释

适用范围: hcomm Algorithm层
强制程度: 推荐

头文件使用中文星号分隔线划分区域:
```cpp
/* *************** 资源计算 *************** */
/* *************** 算法编排 *************** */
```

### ALG-10: rankSize=1快速路径

适用范围: hcomm Algorithm层
强制程度: 强制

所有算法模板和executor统一处理单卡: input!=output则D2DMemcpy, 否则直接返回HCCL_SUCCESS。

出处: 全部102个算法模板一致

### ALG-11: AIV设备侧错误处理

适用范围: hcomm AIV模块
强制程度: 强制

AIV kernel使用不同于主机侧的错误处理: AIV_ERROR -> PrintfImpl + __builtin_trap()(不可恢复, 直接停机), 无CHK_RET。HeadCounter/TailCounter用于维测。

出处: aiv_communication_base.h

### ALG-12: 四层继承 + 双Registry工厂

适用范围: hcomm impl/
强制程度: 强制(架构模式)

```
CollExecutorBase (L0: 纯虚CalcResRequest+Orchestrate)
  +- CollNativeExecutorBase (L1: Template Method骨架)
       +- CollCommExecutor (L2: Multi-Ring/Stream编排)
       |    +- CollAllReduceExecutor (L3: 操作特化) -> 33个具体变体
       +- CollAlltoAllExecutor (L1直接, 跳过L2: O(N^2)模式不同)
       +- CollSend/ReceiveExecutor (L1直接: P2P不需多级通信域)
```

AllToAll跳过L2是因为数据流模式与reduce+gather完全不同。

出处: coll_executor_base.h, coll_native_executor_base.h, coll_comm_executor.h

### ALG-13: AllReduce = RS + AG组合模式

适用范围: hcomm Algorithm层
强制程度: 强制(算法规范)

AllReduce统一分解为ReduceScatter+AllGather两阶段。组合通过GetAlgTemplate(531次/121文件)动态获取子模板实现, 非继承。RS/AG在执行顺序上完美反向对称: RS先内后外(L0->L1->L2), AG先外后内(L2->L1->L0)。

例外: AllReduceReduceBcast = ReduceRing + BroadcastRing。

出处: all_reduce_ring.cc等, GetAlgTemplate调用处

### ALG-14: Strategy设备分发

适用范围: hcomm impl/
强制程度: 强制

Operator层通过selectFuncMap_[DevType]分发到SelectAlgfor910A/910B/91093/310P3/Mix。共175处分发实现/14文件。AllReduce最复杂(721行), Send/Receive固定(无选择)。

出处: coll_alg_operator.cc:42-80

### ALG-15: CommBase与AlgTemplateBase组合

适用范围: hcomm Algorithm层
强制程度: 强制(架构规范)

两者非继承关系: CommBase是"链路管理者"(创建和持有Transport links), AlgTemplateBase是"算法执行者"(接收links作为参数)。桥接: CommBase::RunTemplateAlg(unique_ptr\<AlgTemplateBase\>&)。

出处: comm_base_pub.h, alg_template_base_pub.h

### ALG-16: 三层算法分工

适用范围: hcomm Algorithm层
强制程度: 强制(架构规范)

三个并行子体系协作:
1. alg_template/ (主机侧generic): 102个模板, 决定step/slice/barrier, 通过Transport link发数据
2. alg_aiv_template/ (设备侧): AIV核间/卡间GM搬运和reduce, header-only
3. communicator/ (基础设施): 拓扑计算、传输需求计算、建链

接触点: executor通过GetAlgTemplate获取template -> template通过CommBase的transportInfo_使用communicator建的链路 -> template编排的kernel launch执行AIV代码。

### ALG-17: 操作分解关系

适用范围: hcomm Algorithm层
强制程度: 强制(领域规范)

| 操作 | 分解 |
|------|------|
| AllReduce | Level0 RS + Level1 AR + Level0 AG |
| Broadcast | Scatter + 机间Broadcast + AllGather |
| Reduce | RS + Gather到root |
| AllToAll | 机内fullmesh + 机间send/recv(独立模式, 跳过通用分解) |

### ALG-18: 三级通信拓扑与算法映射

适用范围: 项目级
强制程度: 强制(领域规范)

| 层级 | 物理含义 | 链路类型 | 可选算法 |
|------|---------|---------|---------|
| Level0(机内) | 同节点各卡 | HCCS/PCIe | Ring/Mesh/HD |
| Level1(机间) | 不同节点 | RoCE/RDMA | Ring/NHR/NHR_V1/NB/HD/AHC/AHC_BROKE |
| Level2(超节点) | 不同超节点 | RoCE | Ring/NHR/NB |

910B支持两级(L0+L1), 910_93支持三级(L0+L1+L2)。

### ALG-19: 数据量分档阈值

适用范围: hcomm Algorithm层
强制程度: 推荐(领域知识)

AIV层: 64KB(small) / 16MB(big) / 190KB(UB容量) / 1MB-8MB(确定性)
Generic层: 128字节(slice对齐) / 16384字节(910B对齐) / 8MB(跨机切分) / 1GB(chunk) / 128KB(NB小数据)
Executor层HugeData判定: size/deviceNum > 2GB(RDMA) 或 size > 4GB(SDMA)

出处: aiv_communication_base.h, alg_template_base_pub.h

### ALG-20: Loop机制与CCL Buffer布局

适用范围: hcomm Algorithm层
强制程度: 强制

数据量超过CCL Buffer时通过RunLoop切循环。CCL Buffer布局:
- inCCLbuffer_(前半) + outCCLbuffer_(后半) + winExpBuffer_(1MB MC2扩展)
- AIV独立buffer: 4MB flag + 36MB data + 32KB commInfo
- 零拷贝(DMAReduceFlag_=true)跳过拷入/拷出

出处: coll_all_reduce_executor.cc:186-243, ccl_buffer_manager.cc

### ALG-21: BatchSendRecv死锁预防

适用范围: hcomm Algorithm层
强制程度: 强制

pair-wise不对称排序: Send队列从remoteRank<=localRank开始递减, Recv队列从remoteRank>=localRank开始递增(注意<=与<不对称)。确保全局send/recv顺序形成无环图。

### ALG-22: GM flag同步协议(AIV)

适用范围: hcomm AIV模块
强制程度: 强制

三种GM flag同步语义: 1v1(Record/Wait点对点), 1vN(广播), Nv1(汇聚)。Ping-Pong机制: tag%2决定flag offset, 避免相邻调用冲突。CpGM2GM必须经UB中转(190KB单buffer/95KB double buffer)。

出处: aiv_communication_base.h:247-598

### ALG-23: 设备能力矩阵

适用范围: hcomm Algorithm层
强制程度: 强制(硬件知识)

| 设备 | AIV | Pipeline | InlineReduce | 拓扑层级 | ARS | ZeroCopy |
|------|-----|----------|-------------|---------|-----|----------|
| 910A | N | N | N | 2级 | N | N |
| 910B | Y | Y | Y | 2级 | N | N |
| 910_93 | Y | Y | Y | 3级 | Y | Y |
| 310P | N | N | N | 1级 | N | N |

出处: SelectAlgforXXX分发(205次/18文件)

### ALG-24: 对齐约束

适用范围: hcomm Algorithm层
强制程度: 强制(硬件约束)

- slice通用对齐: 128字节, 910B/910_93: 16384字节
- CCE reduce对齐: 32字节(CCE_REDUCE_ALIGN_FACTOR=2)
- CCL Buffer: 4K页对齐
- SDMA单次传输上限: 4GB, RDMA单次: 2GB

出处: alg_template_base_pub.h, executor_base.h:27-28

### ALG-25: InlineReduce vs TBE Reduce

适用范围: hcomm Algorithm层
强制程度: 强制(硬件约束)

PROD运算或INT64类型必须用TBE Reduce(硬件不支持inline)。其他情况根据link能力和reduceAttr位掩码(RDMA_REDUCE_BITMASK/INLINE_REDUCE_BITMASK)自动选择。

出处: coll_reduce_scatter_executor.cc:143-145, sender_pub.h, reducer_pub.h

## 4.1 Algorithm层反模式

### AP-ALG-01: Executor变体爆炸

适用范围: hcomm impl/
强制程度: 强制关注

142个具体Executor变体由9个正交维度(拓扑/算法/硬件/数据量/模式/引擎/确定性/内存/流水线)组合产生。导致大量copy-paste(CalcCommInfo/CalcTransportMemType等逐变体复制)。

出处: impl/coll_executor/ 319个文件

### AP-ALG-02: Level1算法选择代码重复

适用范围: hcomm impl/
强制程度: 强制关注

KernelRun()中的Level1算法选择if-else链在Ring和Mesh的AllReduce/ReduceScatter/AllGather中几乎逐行重复(6+处)。SelectTempAlg方法已存在但未被一致使用。

出处: coll_all_reduce_ring_executor.cc:161-196, coll_all_reduce_mesh_executor.cc:140-175

### AP-ALG-03: Prepare重载膨胀

适用范围: hcomm base/
强制程度: 推荐改进

基类约30个virtual Prepare重载(1~19个参数)。新代码已用PrepareData结构体, 但旧重载未清理。

出处: alg_template_base_pub.h

### AP-ALG-04: 默认空实现掩盖遗漏

适用范围: hcomm base/
强制程度: 推荐改进

V1基类所有虚方法返回HCCL_SUCCESS(非pure virtual), 子类忘记override RunAsync不会编译报错。V2已改为返回HCCL_E_INTERNAL+日志。

出处: alg_template_base_pub.h

### AP-ALG-05: dynamic_cast打破多态

适用范围: hcomm impl/
强制程度: 推荐改进

alltoall_operator.cc中3处dynamic_cast\<CollAlltoAllExecutor*\>打破Operator-Executor多态。正确做法: 通过虚方法或desc_声明额外接口。

出处: alltoall_operator.cc:454,460,491

### AP-ALG-06: PostSync三方法结构重复

适用范围: hcomm impl/
强制程度: 可选改进

PostSyncWithSubstream/PostSyncWithoutSubstream/InplaceOpSync结构几乎完全相同, 只是核心操作不同。

出处: coll_native_executor_base.cc:622-777

### AP-ALG-07: 25参数方法

适用范围: hcomm impl/
强制程度: 推荐改进

ThreadManage::Prepare有25个参数, CommBase构造函数25+个参数。应使用参数结构体。

出处: threadManage.cc:187-227, comm_base_pub.h

### AP-ALG-08: Header guard冲突

适用范围: hcomm impl/
强制程度: 推荐修复

多个executor头文件的header guard与其他文件冲突(如mesh_aiv_for_910_93与mesh_aiv共用同一guard)。

### AP-ALG-09: 拼写错误固化

适用范围: hcomm impl/
强制程度: 可选(已被API引用固化)

RunIntraSeverReduceScatter("Server"->"Sever"), sendDataSilces_("Slices"->"Silces"), excecutor等。

### AP-ALG-10: CalcLoopMaxCount接口不一致

适用范围: hcomm impl/
强制程度: 推荐修复

RS用成员变量直接访问, AG用参数传入。同一抽象层的相同概念应保持一致。

出处: coll_reduce_scatter_executor.h:28 vs coll_all_gather_executor.h:24

### AP-ALG-11: friend class架构债务

适用范围: hcomm impl/
强制程度: 可选(遗留)

hccl_impl.h中`friend class CollAlgOperator`暴露新旧框架过渡期债务。

出处: hccl_impl.h:78-79

---

# 第5章: 算子框架(hccl op_common)惯用法与规范

hccl op_common(70文件/10,212行)提供算子框架: topo(拓扑) -> selector(选择) -> executor(编排) -> template(执行)四子模块。

### OPC-01: V2执行器为主力

适用范围: hccl op_common
强制程度: 强制

V2(InsCollAlgBase)是主力, 60个注册覆盖所有主要算子。V1(ExecutorBase)仅scatter使用(4个注册), 属遗留代码。新算子必须使用V2。

V1 vs V2关键差异:
- V1: 1个纯虚(Orchestrate), 宽松默认, unique_ptr, 单维Registry(tag)
- V2: 3个纯虚(CalcAlgHierarchyInfo+CalcRes+Orchestrate), 严格强制, shared_ptr, 双维Registry(HcclCMDType+tag)

出处: executor_base.h:44, executor_v2_base.h:28

### OPC-02: V2 Registry双维索引 + 7种注册宏

适用范围: hccl op_common
强制程度: 强制

V2 Registry用(HcclCMDType, tag)双维索引。7种注册宏支持0~4个模板参数, 实现拓扑匹配+算法模板的编译时组合注入。所有宏用三层嵌套(PUBLIC->_HELPER_1->_HELPER)确保__COUNTER__正确展开。

```cpp
REGISTER_EXEC_V2(type, name, base, AlgTopoMatch, InsAlgTemplate)
REGISTER_EXECUTOR_BY_TWO_TEMPS(type, name, ..., Template0, Template1)
REGISTER_EXECUTOR_BY_FOUR_TEMPS(type, name, ..., T0, T1, T2, T3)
```

出处: coll_alg_v2_exec_registry.h:43-104 (60个注册)

### OPC-03: 5路执行模式分派

适用范围: hccl op_common selector
强制程度: 强制

基类Select()按固定序分派: DPU -> CCU_MS -> CCU_SCHED -> AIV -> AICPU(兜底)。5个virtual方法默认返回NOT_MATCH, 子类按需override。降级链: CCU_MS失败 -> CCU_SCHED -> AICPU。

出处: auto_selector_base.cc:16-66

### OPC-04: Selector优先级全部硬编码18

适用范围: hccl op_common selector
强制程度: 强制(现状)

全仓14个REGISTER_SELECTOR_BY_OPTYPE注册的优先级全部是18。MC2路径通过`selectors.find(18)`硬编码查找。

出处: execute_selector.cc:28

### OPC-05: SelectorStatus二值返回

适用范围: hccl op_common selector
强制程度: 强制

算法选择使用MATCH/NOT_MATCH简洁二值枚举(非HcclResult错误码), 与CHK_RET体系分离。

出处: auto_selector_base.h:28 (366次/31文件)

### OPC-06: 四代Template基类独立不继承

适用范围: hccl op_common template
强制程度: 强制(架构规范)

四种处理器路径各有独立基类, 互不继承:
- AlgTemplateBase(V1/SDMA): 仅scatter使用, virtual默认返回SUCCESS(危险)
- InsAlgTemplateBase(V2/AICPU): 主力, KernelRun默认返回HCCL_E_INTERNAL(安全)
- AivAlgTemplateBase(AIV): 多CalNumBlocks和Pre/PostSync
- CcuAlgTemplateBase(CCU): 多PointerToAddr/GetChannelDieId(die感知)

各基类templateRankSize_计算不同: AICPU=乘积, AIV=第一维, CCU=求和。
含义: AICPU看逻辑rank(层级乘积), AIV看同层rank(单级), CCU看物理die(所有die之和)。

出处: alg_template_base.h, alg_v2_template_base.h, aiv_alg_template_base.h, ccu_alg_template_base.h

### OPC-07: AICPU Kernel Launch双路径

适用范围: hccl op_common template/aicpu
强制程度: 强制

V1(非910_95): reinterpret_cast直接转换资源上下文 + 手动内存偏移取变长数据, 完整profiling。
V2(910_95): DeSerialize()正式反序列化 + RestoreVarData重建指针, 无profiling。

Device侧入口`extern "C" unsigned int HcclLaunchAicpuKernel(OpParam *param)`按DevType分派。

出处: kernel_launch.cc:41

### OPC-08: CCU编译器式DSL编程模型

适用范围: hccl op_common CCU template
强制程度: 强制(CCU开发)

CCU模板采用三层编译模型:
1. 前端(Kernel API): CreateVariable/Load/NotifyRecord/NotifyWait/GroupReduce/WriteReduceNb/CCU_IF
2. 中间表示(CcuRep IR): Variable/LocalAddr/RemoteAddr/CompletedEvent
3. 后端(Microcode): GeneArgs序列化为uint64数组

同步: NotifyRecord/Wait + CKE 16位信号掩码。非阻塞操作每16个WaitEvent一次(RANK_NUM_PER_CKE=16)。

出处: 各ccu_kernel_*.cc

### OPC-09: 三层拓扑模型 + 4种TopoMatch

适用范围: hccl op_common topo
强制程度: 强制(拓扑模块)

三层通道计算(Level0/1/2)各一个入口函数, 5种拓扑算法(Mesh1D/Mesh2D/NHR/Mesh1DWithPriority/NHRWithPriority)。连接计算输出set\<u32\>(自动去重)。4种TopoMatch类型: 1D/2D/Multilevel/UBX。

出处: channel.h, channel.cc, channel_request.cc

### OPC-10: Notify双阶段同步

适用范围: hccl op_common wrapper
强制程度: 强制

12个远端传输函数(SendWrite/RecvWrite/SendRecvWrite等)。ACK+DATA_SIGNAL双阶段同步: 接收端先RecordAck(buffer ready), 发送端WaitAck后写数据再RecordDataSignal, 接收端WaitDataSignal后使用数据。

出处: alg_data_trans_wrapper.h/cc

### OPC-11: 算法命名规范

适用范围: hccl op_common
强制程度: 推荐

算法名由四段构成: {Engine}{OpName}{Topology}{Variant}
- Engine前缀: Ccu(CCU) / Ins(AICPU) / Aiv(AIV) / InsV2(DPU)
- Topology: Mesh1D / Mesh2D / NHR / NHR1D
- Variant后缀: OneShot / TwoShot / Mem2Mem / MeshChunk / MultiJetty

出处: 各selector算法名定义

### OPC-12: 条件编译 #ifndef AICPU_COMPILE

适用范围: hccl op_common
强制程度: 强制

channel.cc中8处`#ifndef AICPU_COMPILE`保护。AICPU编译时通道计算函数变为空操作(直接返回HCCL_SUCCESS)。

出处: channel.cc

### OPC-13: 数据结构序列化(跨进程)

适用范围: hccl op_common template
强制程度: 强制(跨进程通信)

TemplateDataParams和DPURunInfo实现Serialize/DeSerialize, 使用BinaryStream << 和 >> 操作符, 支持AICPU和DPU的跨进程数据传递。

出处: template_utils.h

## 5.1 op_common反模式

### AP-OPC-01: Describe()返回"111"占位符

适用范围: hccl op_common executor
强制程度: 推荐修复

executor_v2_base.cc默认Describe()返回"111", 28个子类可能继承此无意义值。应返回实际描述或声明为纯虚。

出处: executor_v2_base.cc:26

### AP-OPC-02: namespace注释错误

适用范围: hccl op_common
强制程度: 可选修复

多个文件结尾写`} // namespace Hccl`但实际namespace是`ops_hccl`。8个selector头文件有此错误。

出处: executor_v2_base.h:67, 各selector .h

### AP-OPC-03: static_assert消息错误

适用范围: hccl op_common
强制程度: 可选修复

V2 Registry工厂的static_assert消息写"derived from DefaultExecCreatorV2", 但实际检查InsCollAlgBase继承。Copy-paste错误。

出处: coll_alg_v2_exec_registry.h:29

### AP-OPC-04: malloc无free(内存泄漏)

适用范围: hccl op_common
强制程度: 强制修复

channel.cc中GetTopoTypeByLink使用malloc分配EndpointDesc数组, 使用后无free()。应使用std::vector。

出处: channel.cc:314

### AP-OPC-05: Registry日志级别不一致

适用范围: hccl op_common
强制程度: 可选修复

V1重复注册用HCCL_WARNING, V2用HCCL_ERROR。同一错误条件不同日志级别。

### AP-OPC-06: FP64重复/INT64遗漏

适用范围: hccl op_common selector
强制程度: 强制修复

3个selector的CcuMs模式中检查FP64两次, 遗漏INT64:

```cpp
// bad: FP64出现两次, 缺INT64
if (dataType == FP64 || dataType == UINT64 || dataType == FP64) { ... }

// good: 使用基类提供的Is64BitDataType()
if (Is64BitDataType(dataType)) { ... }
```

出处: all_reduce_auto_selector.cc:42-44, reduce_auto_selector.cc:36-38, reduce_scatter_v_auto_selector.cc:38-40

### AP-OPC-07: levle0Algo拼写错误

适用范围: hccl op_common selector
强制程度: 可选(已固化)

变量名`levle0Algo`(应为level0Algo)出现在8个selector文件共57处。

### AP-OPC-08: AllGatherV fall-through bug

适用范围: hccl op_common selector
强制程度: 强制修复

all_gather_v_auto_selector.cc中SelectCcuMsAlgo的多层拓扑分支设置算法名后没有return, 代码继续到SelectMeshAlgo覆盖算法名。多层Parallel算法永远不生效。

出处: all_gather_v_auto_selector.cc:23-43

### AP-OPC-09: copy-paste错误集中

适用范围: hccl op_common
强制程度: 推荐注意

约72条反模式中约30%为copy-paste引发: 日志类名错误、header guard不匹配、namespace注释错误、参数拼写错误等。reduce_scatter是最常被复制的代码源头。

---

# 第6章: 算子实现(hccl)惯用法与规范

hccl的12个算子实现分布在src/ops/下, 遵循统一的op -> selector -> executor -> template四件套模式。

### OP-01: 三重门控渐进迁移

适用范围: hccl全部算子
强制程度: 强制

所有新算子入口(op.cc)使用三重门控进入新路径:
1. CheckHCCLIndependentOp() -- 非独立算子模式则fallback
2. DevType != DEV_TYPE_910_95 -- 非A5芯片则fallback
3. GetWorkflowMode() != OP_BASE -- 图模式则fallback

通过门控后: InitEnvConfig -> 参数校验 -> OpParam组装 -> Selector -> HcclExecOp

出处: all_reduce_op.cc:43-55, 所有12个算子的op.cc

### OP-02: CHK_RET全覆盖

适用范围: hccl全部算子
强制程度: 强制

1197次, 覆盖全部12个算子。scatter最密(237次), P2P最少(21-22次)。

### OP-03: RPT_INPUT_ERR + CHK_PTR_NULL双重空指针检查

适用范围: hccl全部算子
强制程度: 强制

API入口先RPT_INPUT_ERR报用户友好错误, 再CHK_PTR_NULL做空检查:
```cpp
RPT_INPUT_ERR(sendBuf == nullptr, ..., "please check sendBuf");
CHK_PTR_NULL(sendBuf);
```

出处: all_reduce_op.cc:86-97, 所有op.cc

### OP-04: 单卡快速路径

适用范围: hccl全部算子
强制程度: 强制

rankSize==1时跳过selector和executor, 直接SingleRankProc快速返回。

出处: all_reduce_op.cc:143-147

### OP-05: Tag命名规范

适用范围: hccl全部算子
强制程度: 强制

Tag构造: "{OpName}_{commName}" 或 "{OpName}_{commName}_{srcRank}_{dstRank}"(P2P)。

出处: all_reduce_op.cc:69 ("AllReduce_"+commName)

### OP-06: NOT_MATCH降级链

适用范围: hccl全部算子selector
强制程度: 强制

每路返回NOT_MATCH尝试下一路: DPU -> CCU_MS -> CCU_SCHED -> AIV -> AICPU(兜底)。AICPU是最后防线, 必须提供fallback算法。

### OP-07: 数据量三档分流

适用范围: hccl集合操作算子
强制程度: 推荐

small/medium/big使用不同算法。AllReduce为例:
- AICPU: <=8MB OneShot, 8~32MB TwoShot, >32MB MeshChunk
- CCU_MS: <=512KB OneShot, >512KB标准
- AIV: <=512KB OneShot, >512KB TwoShot

不同引擎分档阈值不同, 因硬件特性(UB大小/SDMA带宽)各异。

出处: all_reduce_auto_selector.cc常量定义

### OP-08: Executor 4种Strategy矩阵

适用范围: hccl AR/RS/AG算子
强制程度: 强制(架构模式)

| Strategy | 特征 | 数据切分 | 适用 |
|----------|------|---------|------|
| Sole | 单template, 不切分 | loop | 单级拓扑, 通用 |
| Parallel | 双template, 50:50等分 | 等分+换轴 | 多级/UBX |
| Concurrent | 双template, 按端口比例 | mesh:CLOS=portNum:4 | UBX rank<=4 |
| Sequence/2Die | 双template, 串行RS+AG | 接力 | 2die或DPU |

并非所有算子都实现全部4种: P2P仅Sole, broadcast 2种。

出处: ins_v2_all_reduce_sole_executor.h等

### OP-09: RS/AG完美反向对称

适用范围: hccl RS和AG算子
强制程度: 强制(架构规范)

RS先内后外(intra->inter), AG先外后内(inter->intra)。Executor/Template/CCU kernel全部1:1对应。CCU操作对称: RS用GroupReduce/WriteReduceNb(含reduce), AG用GroupCopy/WriteNb(仅copy)。RS比AG多确定性变体(因reduce浮点非确定性)。

### OP-10: P2P Producer-Consumer协议

适用范围: hccl send/recv
强制程度: 强制

send/recv通过两阶段握手: Recv先RecordAck(buffer ready) -> Send WaitAck后写数据并RecordDataSignal -> Recv WaitDataSignal后使用数据。PUT DMA模式(发送端推)。OFFLOAD直接写output, OPBASE经ccl buffer中转。

出处: send/ins_send_executor.cc, recv/ins_recv_executor.cc

### OP-11: AllReduce是最复杂的算子

适用范围: hccl全部算子
强制程度: 推荐(认知)

AllReduce(55文件)拥有22种算法和4种executor策略, 是模式发源地。其他算子(RS/AG/BC/SC)从AllReduce或RS copy-paste而来。

### OP-12: reduce_scatter是CCU模板代码发源地

适用范围: hccl CCU模板
强制程度: 推荐(认知)

AR的CCU模板日志全写成ReduceScatter类名(如CcuTempReduceScatterMeshMem2Mem1D), 证明代码从RS copy创建AR。copy-paste bug会向所有算子扩散。

出处: ccu_temp_all_reduce_mesh_1D_one_shot.cc:91等

### OP-13: CCU Algorithm()统一生命周期

适用范围: hccl CCU模板
强制程度: 强制

所有CCU kernel遵循: InitResource -> LoadArgs -> [LocalCopy] -> PreSync -> DoXxx -> PostSync

### OP-14: AIV Producer-Consumer核分工

适用范围: hccl AIV模板
强制程度: 强制

AIV kernel按GetBlockIdx()区间分配角色: Producer核负责CpGM2GM+Record, Consumer核负责WaitFlag+Reduce。GM搬运后必跟pipe_barrier(PIPE_ALL)确保DMA完成。

出处: aiv_all_reduce_mesh_1d_oneshot.h:57-90

### OP-15: 三种设备编程模型根本不同

适用范围: hccl全部算子template
强制程度: 强制(认知)

- AICPU: Host主控, 多slave线程并行, PreSync/PostSync线程同步
- AIV: 设备侧直接执行, GM flag busy-poll, Producer/Consumer核分工
- CCU: 编译器式DSL, Host构造微码IR, CKE硬件同步, 非阻塞+event等待

### OP-16: AllToAll的O(N^2)特殊性

适用范围: hccl all_to_all_v
强制程度: 强制(架构规范)

AllToAll跳过CollCommExecutor范式(直接继承CollNativeExecutorBase), O(N^2)不适合层次化分解。独有: HasMassTasks(>65535), per-pair三分支处理, 三个API合一(AlltoAll/AlltoAllV/AlltoAllVC)。

### OP-17: V变体极简模式

适用范围: hccl all_gather_v, reduce_scatter_v
强制程度: 推荐(架构规范)

V变体仅8文件最小集合, 只支持AICPU, CCU/AIV复用固定长度算子实现。变长核心: per-rank tracking allRankProcessedDataCount[i]。

### OP-18: scatter是唯一V1->V2过渡态

适用范围: hccl scatter
强制程度: 推荐(认知)

唯一同时拥有algo/(V1, 4个executor+6个template)和executor/selector/template/(V2)的算子。通过设备类型判断(910_95走V2, 其他走V1)共存。

### OP-19: batch_send_recv死锁预防

适用范围: hccl batch_send_recv
强制程度: 强制

pair-wise不对称排序: Send用remoteRank<=myRank加rankSize后降序, Recv用remoteRank<myRank(注意<而非<=)加rankSize后升序。使用Read语义(接收端拉)。

### OP-20: 数据类型限制

适用范围: hccl selector
强制程度: 强制(硬件约束)

- PROD运算: 所有CCU/AIV/AICPU路径NOT_MATCH
- 64位类型(FP64/INT64/UINT64): CCU_MS/CCU_SCHED/AIV不支持, 仅AICPU特殊算法
- INT8: CCU_MS部分不支持

出处: 各selector前置排除逻辑

## 6.1 算子层反模式

### AP-OP-01: 幽灵算法(selector选择但executor未注册)

适用范围: hccl RS/AG selector
强制程度: 强制修复

RS: InsReduceScatterAicpuReduce(64位数据类型触发, 全仓库无注册, 运行时失败)。AG: 4个CCU算法名与executor注册名不匹配。

出处: reduce_scatter_auto_selector.cc:232,260; all_gather_auto_selector.cc

### AP-OP-02: FP64重复/INT64遗漏

适用范围: hccl 3+个selector
强制程度: 强制修复

CcuMs模式检查FP64两次遗漏INT64。基类已提供Is64BitDataType()工具函数但未统一使用。

出处: all_reduce_auto_selector.cc:42-44等

### AP-OP-03: Selector缺return

适用范围: hccl AG selector
强制程度: 强制修复

AG SelectCcuMsAlgo设置算法名后未return, 落入SelectMeshAlgo覆写。Reduce selector也有类似死代码。

出处: all_gather_auto_selector.cc:24-26, reduce_auto_selector.cc:165-181

### AP-OP-04: NHR recv用错变量

适用范围: hccl AllReduce NHR template
强制程度: 强制修复

recvSrcSlicesList.emplace_back(sendSrcSlice) -- 应为recvSrcSlice。RunReduceScatter和RunAllGather中都有。

出处: ins_temp_all_reduce_nhr.cc:210-211, 266-267

### AP-OP-05: CCU goSize选择反转

适用范围: hccl CCU NHR multi_jetty
强制程度: 强制修复

LocalCopySlices中islastSlice为true时应用lastSlice的goSize, 但代码用了正常goSize。

出处: ccu_kernel_all_reduce_nhr_mem2mem_1D_multi_jetty.cc:192

### AP-OP-06: Broadcast push到错误容器

适用范围: hccl Broadcast NHR template
强制程度: 强制修复

txDstSlice被push到txSrcSlices(源容器)而非目标容器。

出处: ins_temp_broadcast_nhr.cc:378-381

### AP-OP-07: copy-paste系统性错误

适用范围: hccl全部算子
强制程度: 推荐修复

- "excutor"拼写: 22处/9个算子
- Header guard不匹配: ~76处(reduce_scatter是最常被复制的源)
- 日志算子名错误: 15+处
- "levle0Algo": 8+文件/30+处
- "Silces"/"snedInfo"/"faled"/"Setlect"

### AP-OP-08: malloc+free泄漏路径

适用范围: hccl all_to_all_v, all_gather_v, reduce_scatter_v
强制程度: 推荐修复

op.cc中malloc分配后, 多个CHK_RET提前返回路径不free。应用std::vector。

出处: all_to_all_v_op.cc:333-414等

### AP-OP-09: SIZE_TABLE与DATATYPE_SIZE_TABLE混用

适用范围: hccl executor
强制程度: 推荐统一

同一算子不同executor混用两种查表方式。

### AP-OP-10: CalcRes WARNING生产环境

适用范围: hccl AICPU/AIV template
强制程度: 推荐修复

多个template的CalcRes打印WARNING "Resource calculation is temporarily not performed", 资源计算未完整实现但每次调用都触发WARNING。

出处: one_shot.cc:38, mesh_chunk.cc:41等

### AP-OP-11: Parallel硬编码50:50

适用范围: hccl AR/RS/AG Parallel executor
强制程度: 可选改进

数据50:50等分, 注释"to do 先做等分", 同一TODO在AR/RS/AG中各有一份。

出处: parallel_executor.cc:156

---

# 第7章: 跨层通用规范

从Phase 4跨仓对比和Phase 5端到端切片中提取的项目级和跨层规范。这些规则在两个仓库中一致遵循或跨越多层生效。

## 7.1 错误处理

### CROSS-01: HcclResult统一返回类型

适用范围: 项目级(两仓一致)
强制程度: 强制

所有函数返回HcclResult枚举。错误码定义在hcomm/include/hccl/hccl_types.h, hccl依赖此定义。

```cpp
// good
HcclResult MyFunction(const Param& param) {
    CHK_RET(SubCall(param));
    return HCCL_SUCCESS;
}

// bad
int MyFunction(const Param& param);     // 不使用int返回值
bool MyFunction(const Param& param);    // 不使用bool
void MyFunction(const Param& param);    // 不忽略错误
```

出处: hccl_types.h:23-51, CHK_RET 14457次(hcomm) + 1483次(hccl)

### CROSS-02: CHK_RET透传语义

适用范围: 项目级(两仓一致)
强制程度: 强制

CHK_RET永远不修改错误码值, 原样向上透传。HCCL_E_AGAIN走WARNING而非ERROR。整个9层调用栈中只有底层(driver调用点)产生新错误码, 其余全部透传。

```cpp
// CHK_RET展开形式(pub_inc/log.h:188)
do {
    HcclResult hcclRet = call;
    if (UNLIKELY(hcclRet != HCCL_SUCCESS)) {
        if (hcclRet == HCCL_E_AGAIN) {
            HCCL_WARNING("[%s]call trace: hcclRet -> %d", __func__, hcclRet);
        } else {
            HCCL_ERROR("[%s]call trace: hcclRet -> %d", __func__, hcclRet);
        }
        return hcclRet;
    }
} while(0)
```

出处: pub_inc/log.h:188, 两仓宏定义完全一致

### CROSS-03: 错误码产生点极少

适用范围: 项目级
强制程度: 强制

只在4个位置做错误码转换/产生: CHK_PRT_RET(driver调用失败, 1131次)、CHK_SAFETY_FUNC_RET(安全函数, 261处)、ExceptionHandler(C++异常映射, 39处)、ExceptionInfo(CCU异常映射)。其余15545次CHK_RET全部原样透传。

出处: phase5-slice-error-propagation.md, hccl_primitive_remote.cc

### CROSS-04: 不使用goto

适用范围: 项目级
强制程度: 强制

goto在两仓中极罕见(hcomm 10处/6文件, hccl 0处)。替代方案: do-while(0)+errorFlag(hcomm framework层, 3处)或CHK_RET+RAII自动清理。

```cpp
// good: do-while(0)+errorFlag模式
bool errorFlag = false;
do {
    ret = Step1(); if (ret) { errorFlag = true; break; }
    ret = Step2(); if (ret) { errorFlag = true; break; }
} while(0);
if (errorFlag) { Cleanup(); }

// bad: goto
if (Step1() != HCCL_SUCCESS) goto cleanup;
```

出处: op_base.cc通信域初始化函数(3处)

### CROSS-05: hccl->hcomm边界双返回类型

适用范围: hccl/hcomm边界
强制程度: 强制

控制面接口(hccl_res.h/hccl_comm.h)返回HcclResult, hccl侧直接CHK_RET。数据面接口(hcomm_primitives.h, 22个函数)返回int32_t, hccl侧必须static_cast。

```cpp
// 控制面: 直接CHK_RET
CHK_RET(HcclThreadAcquire(comm, engine, threadNum, notifyNum, &thread));

// 数据面: 必须static_cast
CHK_RET(static_cast<HcclResult>(HcommWriteOnThread(thread, channel, dst, src, len)));
```

出处: static_cast<HcclResult>(Hcomm...): 165次/27文件

## 7.2 内存管理

### CROSS-06: 智能指针替代裸new/delete

适用范围: 项目级
强制程度: 强制

两仓均无裸delete。unique_ptr(hcomm 283/hccl 95)和shared_ptr(hcomm 248/hccl 70)管理所有动态分配。

例外: C ABI接口层的POD struct(如需传递给C函数或跨语言边界)可使用malloc/free,因为这些场景不能使用C++析构语义。

```cpp
// good
auto ptr = std::make_unique<MyClass>();
auto ptr = std::make_shared<MyClass>();

// hcomm特有: new(nothrow)(324次)
auto ptr = std::unique_ptr<MyClass>(new (std::nothrow) MyClass());

// C ABI场景: malloc/free管理POD struct(可接受)
HcclOpArgs *args = static_cast<HcclOpArgs *>(malloc(sizeof(HcclOpArgs)));
free(args);

// bad
MyClass* ptr = new MyClass();
delete ptr;
```

出处: phase4-contrastive-analysis.md 4.2节

### CROSS-07: 连续内存切分模式

适用范围: 项目级
强制程度: 推荐

一次分配连续内存, 偏移切分为多个逻辑区域。减少alloc次数, 保证局部性。

```cpp
// CCL Buffer: 一次alloc切分三区
hrtMalloc(inSize + outSize + expSize);
inputBuf  = base + 0;
outputBuf = base + inSize;
expBuf    = base + inSize + outSize;

// AICPU SQE: 连续分配按64B步长切分
```

出处: hccl_communicator_host.cc(AllocCCLBuffer), dispatcher_aicpu.cc

## 7.3 日志规范

### CROSS-08: 统一日志宏体系

适用范围: 项目级(两仓一致)
强制程度: 强制

所有日志通过HCCL_DEBUG/INFO/WARNING/ERROR宏。宏自动添加[file:line][tid]前缀。不使用直接DLOG。两仓pub_inc/log.h定义完全一致。

出处: HCCL_DEBUG 3097+545, HCCL_INFO 7679+935, HCCL_WARNING 1362+194, HCCL_ERROR 9605+664

### CROSS-09: 日志级别分支预测

适用范围: 项目级
强制程度: 强制

DEBUG/INFO/WARNING使用UNLIKELY(假设通常不打), ERROR使用LIKELY(假设通常会打)。

出处: pub_inc/log.h中HCCL_DEBUG/INFO/WARNING定义

### CROSS-10: hcomm通信日志必须带tag

适用范围: hcomm
强制程度: 强制

hcomm中与通信域操作相关的日志必须包含tag[%s]标识符(1061次), 用于分布式多rank日志追踪。hccl不需要。

```cpp
// hcomm
HCCL_INFO("[%s] AllReduce start, count[%llu] tag[%s]", __func__, count, tag.c_str());

// hccl
HCCL_INFO("[Algo][Selector] selectAlgName = %s", algName.c_str());
```

出处: tag[模式 1061次(hcomm) vs 16次(hccl)

## 7.4 命名规范

### CROSS-11: 文件和类命名统一

适用范围: 项目级(两仓100%一致)
强制程度: 强制

文件名: snake_case(.h/.cc)。类名: PascalCase。函数名: PascalCase。成员变量: trailing underscore(count_)。宏/常量: ALL_CAPS。无m_前缀。

出处: 两仓100%文件一致遵循

### CROSS-12: 命名分区

适用范围: 仓库级
强制程度: 推荐

hcomm使用enterprise风格: _pub.h后缀(187文件)标记公开头文件, Manager后缀(63类), Impl后缀(Pimpl)。hccl使用algorithm风格: Executor/Selector/Template/Registry命名。hcomm用hccl命名空间, hccl用ops_hccl命名空间。

出处: phase4-contrastive-analysis.md 4.4节

## 7.5 设计模式

### CROSS-13: Per-Device单例模式

适用范围: 项目级
强制程度: 强制

两仓使用static T instances[MAX_DEVICE_ID + 1]数组实现per-device单例, 而非传统全局单例。通过deviceLogicId索引。

出处: GetInstance约54次(hcomm) + 77次(hccl)

### CROSS-14: 句柄化C ABI接口

适用范围: hccl/hcomm边界
强制程度: 强制

所有跨仓接口使用extern "C" + uint64_t不透明句柄。22个C ABI函数, 句柄通过reinterpret_cast在指针和uint64_t间转换。

```cpp
// hcomm侧: 指针转句柄
*threadHandle = reinterpret_cast<uint64_t>(thread);

// hccl侧: 句柄传入hcomm
CHK_RET(HcommWriteOnThread(threadHandle, channelHandle, dst, src, len));
```

出处: hcomm_primitives.h(10个数据面), hccl_res.h(8个资源面), hccl_ccu_res.h(3个CCU面)

### CROSS-15: Lazy资源初始化

适用范围: 项目级
强制程度: 推荐

Init阶段只记录配置和准备基础设施(NIC打开、监听端口), 不分配具体资源。CCL Buffer、Transport连接等在首次操作时lazy创建。避免分配可能用不到的资源。

出处: HcclAlg::Init不建链, TransportManager在首次ExecOp时CreateLink

### CROSS-16: 严格逆序销毁

适用范围: 项目级
强制程度: 强制

析构顺序是初始化的严格逆序。28步init对应43步destroy(destroy步骤更多因包含运行时资源)。每步销毁都best-effort继续, 不因某步失败中断后续清理。

出处: hccl_communicator_host.cc析构函数, 171-317行

## 7.6 跨仓数据流

### CROSS-17: AllReduce = ReduceScatter + AllGather

适用范围: 项目级
强制程度: 推荐(理解用)

AllReduce在逻辑上分解为ReduceScatter(每rank得到reduce结果的一个slice) + AllGather(广播所有slice)。Executor中Sequence2Die和TwoShot模板直接体现这种分解。

出处: ins_v2_allreduce_sequence2die_executor.cc, phase5-slice-allreduce.md

### CROSS-18: Selector->Executor->Template三层分离

适用范围: hccl
强制程度: 强制

Selector负责策略决策(拓扑x数据量x引擎->algName字符串), Executor负责资源编排(资源计算+数据切分+循环控制), Template负责任务生成(单次通信的具体操作序列)。通过Registry+字符串名称松耦合。

出处: op_common.cc, all_reduce_auto_selector.cc, phase5-slice-allreduce.md

### CROSS-19: 三种设备编程模型差异

适用范围: hccl template层
强制程度: 推荐(理解用)

AICPU: 多线程模型, Host编排, wrapper函数调hcomm原语。AIV: 单kernel模型, Host构造参数, Device侧GM flag同步。CCU: 编译器模型, Host构造微码IR, CCU硬件执行。

出处: phase5-slice-allreduce.md 4.1-4.3节

## 7.7 条件编译

### CROSS-20: include guard而非pragma once

适用范围: 项目级
强制程度: 强制

两仓接近100%使用#ifndef include guard, #pragma once仅个位数。

出处: #pragma once: hcomm 6处, hccl 4处

### CROSS-21: AICPU_COMPILE设备侧分裂

适用范围: 项目级
强制程度: 强制

AICPU_COMPILE / CCL_KERNEL_AICPU用于Host/Device编译分裂。hcomm中Host/Device成对文件(xxx_host.cc / xxx_device.cc), hccl按template分裂。

出处: AICPU_COMPILE引用 hcomm ~1596次, hccl ~142次

---

# 第8章: 反模式汇总清单

按类别组织的全项目反模式清单。每条附来源层和严重程度。前6章已包含各层反模式的详细描述, 本章做跨层汇总索引。

## 8.1 God Class / God File

### AP-GLOBAL-01: 超大文件

适用范围: 项目级
严重程度: 高

hcomm中4个文件超过4000行: op_base.cc(4695行), hcom.cc(4076行), hccl_communicator_host.cc(9152行), communicator_impl.cc(3200行)。hccl中无此问题。

来源: phase2-hcomm-framework.md

### AP-GLOBAL-02: God Class

适用范围: hcomm
严重程度: 高

HcclCommunicator(100+方法/100+成员), CommunicatorImpl(3200行), HcclCommAicpu(580行头文件/150+方法)。核心协调仍高度耦合。

来源: FW-AP-1, AP-FW-1

## 8.2 全局可变状态

### AP-GLOBAL-03: 全局static map无锁/无清理

适用范围: 项目级
严重程度: 高

hcomm platform层3处、framework next层4处、hccl 0处。全局static map存储句柄-对象映射, 无大小限制(OOM风险), 部分无锁保护。

来源: AP-FW-1, AP-FW-3, phase5-slice-resource-lifecycle.md AP-RL-11

### AP-GLOBAL-04: getenv热路径调用

适用范围: hcomm
严重程度: 中

getenv()在API入口等热路径多处调用, 且无缓存。应在初始化时读取一次并缓存。

来源: AP-FW-11

## 8.3 类型安全

### AP-GLOBAL-05: reinterpret_cast无检查

适用范围: 项目级
严重程度: 中

hcomm 1376次, hccl边界22个接口全部使用。C ABI句柄转换无类型安全机制, 传入错误句柄导致未定义行为。

来源: AP-FW-4, AP-RL-10, phase5-slice-allreduce.md 9.4

### AP-GLOBAL-06: int32_t->HcclResult裸转换

适用范围: hccl/hcomm边界
严重程度: 中

数据面165次static_cast<HcclResult>无值域校验。如果两边错误码枚举不同步, CHK_RET仍触发但语义错误。

来源: AP-EP-1, phase5-slice-error-propagation.md

## 8.4 并发安全

### AP-GLOBAL-07: 忙等待无退避

适用范围: hcomm
严重程度: 中

6处usleep/busy-wait轮询无指数退避: HostNIC TCP(200us), Group等待(usleep), 建链等待, HDC轮询等。

来源: AP-FW-8, AP-PL-8

### AP-GLOBAL-08: 析构竞态

适用范围: hcomm
严重程度: 中

ShareCCLbuffer引用计数与销毁竞态(AP-RL-15)、Transport DeInit顺序不确定(AP-RL-14)、HcclCommDestroy后全局状态残留(AP-RL-16)。

来源: phase5-slice-resource-lifecycle.md

## 8.5 代码重复

### AP-GLOBAL-09: copy-paste错误

适用范围: 项目级
严重程度: 高

hcomm: 5处日志写错函数名(Send/Recv/RS/BSR结束日志都写成HcclAllGatherVV2)。hccl: AllReduce模式向RS/AG/Broadcast复制时遗留痕迹。

来源: AP-FW-5, AP-OP-02/03/05

### AP-GLOBAL-10: 配置代码重复

适用范围: hcomm
严重程度: 中

后置配置(Set*Config)在InitCommRootInfo/InitCommClusterInfo中重复~80行。8步序列重复3次(~180行冗余)。

来源: AP-RL-3, phase2-hcomm-framework.md

## 8.6 设计问题

### AP-GLOBAL-11: V1/V2双代并存

适用范围: hcomm
严重程度: 低(已知技术债务)

V1(HcclCommunicator)/V2(CommunicatorImpl)通过IsCommunicatorV2()分叉, HCCLV2_FUNC_RUN宏145次使用。分派位置不一致(有的在校验前, 有的在校验后)。

来源: AP-FW-12, FW-AP-4

### AP-GLOBAL-12: 两套异常类并存

适用范围: hcomm CCU层
严重程度: 低(迁移中)

Hccl::HcclException(旧)和hccl::HcclException(新)并存, 映射表不同步。CCU层TODO注释"需要统一整改为不抛异常"。

来源: AP-EP-2, phase5-slice-error-propagation.md

## 8.7 已确认bug索引

| ID | 描述 | 位置 | 来源 |
|----|------|------|------|
| BUG-01 | REDUCE_SCATTER缺失break导致fall-through | hccl_group.cc:383 | AP-FW-6 |
| BUG-02 | zero_copy operator<违反strict weak ordering | zero_copy_memory_agent.h | AP-FW-13 |
| BUG-03 | HcclGetOpArgsV2检查错误变量 | op_base_v2.cc | Iter 12 |
| BUG-04 | HcclFreeOpArgsV2局部置null无效 | op_base_v2.cc | Iter 12 |
| BUG-05 | 变量遮蔽(int32_t family重声明) | orion_adpt_utils.cc:30 | Iter 15 |
| BUG-06 | NHR recv slice用错send变量 | ins_temp_all_reduce_nhr.cc | AP-OP-06 |
| BUG-07 | Parallel CalcSendDataSize赋值错误 | ins_all_reduce_parallel_executor.cc | AP-OP-07 |
| BUG-08 | CCU multi_jetty goSize选择反了 | ccu_kernel_all_reduce_mesh1d_multi_jetty.cc | AP-OP-08 |
| BUG-09 | sizeof(dataType)取enum大小而非类型宽度 | all_gather_v_testcase.cc | HC-TST-AP-FW-5 |
| BUG-10 | TopoModel 910C链路写入910B map | topo_model.cc:414-453 | HC-TST-AP-FW-8 |
| BUG-11 | Stream移动赋值双重释放 | stream.cc | AP-PL-5 |
| BUG-12 | RemoteReadReduce用dataType枚举值做除法 | hccl_primitive_local.cc | AP-PL-3 |

---

# 第9章: 测试与示例规范

hcomm和hccl的测试体系差异巨大: hcomm重UT(273文件, MockCPP白盒测试), hccl重ST(SimWorld仿真框架, 0个真正UT)。两者共享DAG成图+语义校验的验证方法论。

## 9.1 测试框架与工具

### TST-01: Google Test + MockCPP

适用范围: 项目级
强制程度: 强制

Google Test v1.14.0用于测试组织和断言。MockCPP v2.7(非gmock)用于C函数mock, 适合大量C ABI和extern "C"接口的场景。

出处: test/ut/CMakeLists.txt, cmake/third_party/gtest.cmake

### TST-02: TEST_F为唯一测试宏

适用范围: 项目级
强制程度: 强制

几乎100%使用TEST_F(223文件), TEST_P极少(hcomm 8次/3文件, hccl 1次/1文件)。依赖fixture管理mock初始化和资源。

出处: TEST_F 223+222文件

### TST-03: TearDown必须verify

适用范围: 项目级
强制程度: 强制

每个fixture的TearDown必须调用GlobalMockObject::verify()确认mock期望满足。

```cpp
virtual void TearDown() {
    GlobalMockObject::verify();
}
```

出处: GlobalMockObject::verify 190个文件

### TST-04: EXPECT_*优先于ASSERT_*

适用范围: 项目级
强制程度: 强制

EXPECT_EQ(210文件)为主, ASSERT_EQ(15次/5文件)仅用于必须成功的前置条件(如资源分配)。允许单个测试收集多个失败点。

出处: phase2-hcomm-test-examples.md

## 9.2 hcomm单元测试规范

### TST-05: MockCPP用法

适用范围: hcomm UT
强制程度: 强制

C函数用MOCKER(funcName), C++成员用MOCKER_CPP(&Class::Method), 重载函数需显式指定签名。

```cpp
// C函数mock
MOCKER(HcomGetCommByGroup)
    .stubs()
    .with(any(), outBound(comm))
    .will(returnValue(HCCL_SUCCESS));

// C++成员mock
MOCKER_CPP(&hcclComm::GetInCCLbuffer)
    .stubs()
    .will(returnValue(HCCL_SUCCESS));

// 重载函数mock: 显式签名
MOCKER_CPP(&DispatcherAiCpu::LaunchTask,
           HcclResult(DispatcherAiCpu::*)(hccl::Stream &, bool))
    .stubs()
    .will(returnValue(HCCL_E_INTERNAL))
    .then(returnValue(HCCL_SUCCESS));    // 连续调用返回不同值
```

出处: MOCKER 186文件, MOCKER_CPP 113文件

### TST-06: 私有成员白盒访问

适用范围: hcomm UT(66%的文件)
强制程度: 常见做法

通过#define private public在include被测头文件前定义, 之后#undef。这是白盒测试核心手段。

```cpp
#ifndef private
#define private public
#define protected public
#include "dispatcher_aicpu.h"
#endif
// ...
#undef private
#undef protected
```

出处: 147/223 UT文件使用

### TST-07: 测试命名规范

适用范围: hcomm UT
强制程度: 推荐

方法名: Ut_Component_When_Scenario_Expect_Result。类名: ClassName_UT或ClassNameTest(两种风格并存)。文件名: ut_component_test.cc。

出处: phase2-hcomm-test-examples.md

### TST-08: 对象实例化

适用范围: hcomm UT
强制程度: 强制

被测对象用new(nothrow)创建, unique_ptr管理生命周期, 与源码保持一致。

```cpp
std::unique_ptr<DispatcherAiCpu> dispatcherAiCpu =
    std::unique_ptr<DispatcherAiCpu>(new (std::nothrow) DispatcherAiCpu(1));
```

出处: 几乎所有UT fixture

## 9.3 hccl系统测试规范(SimWorld)

### TST-09: SimWorld仿真模式

适用范围: hccl ST
强制程度: 强制

所有ST通过SimWorld::Global()单例管理仿真世界。SimNpu模拟NPU(虚拟内存/Stream/Notify), 地址仿真而非数据仿真。

```cpp
// ST标准流程
SimWorld::Global()->Init(topoMeta, DEV_TYPE_910_95);
// ... 多线程模拟各rank ...
Check*(SimTaskQueue::Global()->GetAllRankTaskQueues(), ...);
SimWorld::Global()->Deinit();
```

出处: SimWorld::Global 80次/23文件

### TST-10: TopoMeta拓扑表达

适用范围: hccl ST
强制程度: 强制

拓扑通过三维vector表达: SuperPod -> Server -> Device。testcase内联初始化列表构造。

```cpp
TopoMeta{{{0, 1, 2, 3}}};           // 1超节点, 1server, 4device (Mesh1D)
TopoMeta{{{0}, {0}, {0}, {0}}};     // 1超节点, 4server, 各1device (NHR)
TopoMeta{{{0, 1}, {0, 1}}};         // 1超节点, 2server, 各2device
TopoMeta{{{0}}, {{1}}};             // 2超节点(跨SuperPod)
```

出处: 所有hccl ST testcase

### TST-11: TaskStub记录+Checker验证

适用范围: hccl ST
强制程度: 强制

被stub替换的runtime API将操作记录为TaskStub(11种类型)而非真正执行。Checker框架7步流水线验证:
1. SingleTaskCheck(流不变量)
2. GenGraph(DAG构建, 含死锁检测)
3. CheckTaskMem(内存合法性)
4. CopyTaskGraph(深拷贝)
5. GraphRevamp(双边语义增强)
6. CheckRankMem(碎片队列+并发矩阵检测内存冲突)
7. OpSemantics(BFS拓扑排序模拟执行, 追踪BufferSemantic)

出处: checker.cc, task_graph_generator.cc

### TST-12: ST Fixture模式

适用范围: hccl ST
强制程度: 强制

Fixture无状态(stateless), SetUp统一调ResetAlgEnvConfigInitState(), TearDown清理环境变量。命名: ST_OPNAME_TEST。断言统一用EXPECT_TRUE(Check*(...) == HCCL_SUCCESS)。

```cpp
class ST_ALL_REDUCE_TEST : public ::testing::Test {
protected:
    void SetUp() override { ResetAlgEnvConfigInitState(); }
    void TearDown() override { unsetenv("HCCL_OP_EXPANSION_MODE"); }
};
```

出处: 所有hccl ST testcase

### TST-13: 环境变量管理

适用范围: 两仓ST
强制程度: 强制

ST测试用setenv()配置算法行为(HCCL_OP_EXPANSION_MODE等), TearDown必须清理。hcomm额外需ClearHcclEnv()。

出处: phase2-hcomm-test-examples.md TST-ID-8, phase3-hccl-test-examples.md

## 9.4 示例代码规范

### TST-14: 7步标准API流程

适用范围: 两仓examples
强制程度: 推荐

所有示例遵循统一流程: ACL Init -> SetDevice -> CreateStream -> CommInit(RootInfo+MPI Bcast) -> Op -> Sync -> Destroy(逆序)。

出处: hcomm examples/01_communicators/, hccl examples/02_collectives/

### TST-15: 错误检查宏

适用范围: examples
强制程度: 推荐

示例使用简洁的ACLCHECK/HCCLCHECK宏, 与生产代码CHK_RET风格一致但更简单。

```cpp
#define HCCLCHECK(ret) do { \
    if (ret != HCCL_SUCCESS) { printf("HCCL error %d\n", ret); return ret; } \
} while (0)
```

出处: hcomm examples, hccl examples

## 9.5 测试反模式

### AP-TST-01: #define private public

适用范围: hcomm UT(147文件)
严重程度: 中

测试与实现高度耦合, 重构时大量编译失败。正确做法: 友元测试类或依赖注入。

### AP-TST-02: rankSize计算逻辑重复

适用范围: hccl ST
严重程度: 低

同一逻辑在5个testcase中有5个不同名函数(AnalyseRankSize/CalsRankSize/CountElements/GetRankSize/内联)。应提取公共函数。

出处: phase3-hccl-test-examples.md HC-TST-AP-FW-1

### AP-TST-03: DATATYPE_SIZE_TABLE重复定义

适用范围: hccl ST
严重程度: 低

各testcase各自定义数据类型大小表(3+次), 名称各异。check_utils.h中有SIZE_TABLE但仅部分使用。

出处: phase3-hccl-test-examples.md HC-TST-AP-FW-2

### AP-TST-04: ST TearDown清理不完整

适用范围: hccl ST
严重程度: 中

各文件TearDown清理的环境变量不一致, 有的清3个有的只清1个。Run中setenv的变量可能遗漏。

出处: phase3-hccl-test-examples.md HC-TST-AP-FW-4

### AP-TST-05: ODR冲突

适用范围: hccl ST
严重程度: 高

scatter_testcase.cc和scatter_testcase_a3.cc都定义同名类ST_SCATTER_TEST, 编译到同一可执行文件违反One Definition Rule。

出处: phase3-hccl-test-examples.md HC-TST-AP-FW-3

### AP-TST-06: Checker中vector::erase(begin())做BFS

适用范围: hccl ST
严重程度: 低

用erase(begin())模拟queue, O(n)复杂度。应使用std::queue(O(1)出队)。

出处: checker.cc:139, task_graph_generator.cc:69

---

# 第10章: 补充规范(Phase 7验证回补)

以下规范从Phase 7真实commit验证中发现缺失,回补至此。

## 10.1 代码质量

### VAL-01: struct字段默认初始化一致性

适用范围: 项目级
强制程度: 推荐

POD结构体字段要么全部提供默认值,要么全部不提供。对外暴露的struct应确保零初始化。

```cpp
// good -- 全部初始化
struct ScatterOpInfo {
    void *inputPtr = nullptr;
    void *outputPtr = nullptr;
    u32 root = INVALID_VALUE_RANKID;
    u64 count = 0;
    HcclDataType dataType = HCCL_DATA_TYPE_RESERVED;
};

// bad -- 部分初始化,count和dataType遗漏
struct ScatterOpInfo {
    void *inputPtr = nullptr;
    void *outputPtr = nullptr;
    u32 root = INVALID_VALUE_RANKID;
    u64 count;           // 遗漏
    HcclDataType dataType;  // 遗漏
};
```

出处: phase7-validation-report.md GAP-01, commit deb37cc2/4a033793

### VAL-02: const正确性

适用范围: 项目级
强制程度: 推荐

只读参数必须标记const。函数参数如果不修改指向的内容,使用const指针/引用。

```cpp
// good
HcclResult CreateScatter(const OpParam *param, ScatterOpInfo *opInfo);

// bad -- param只被读取但未标记const
HcclResult CreateScatter(OpParam *param, ScatterOpInfo *opInfo);
```

出处: phase7-validation-report.md GAP-02, commit deb37cc2

### VAL-03: format string类型安全

适用范围: 项目级
强制程度: 强制

HCCL_ERROR/HCCL_INFO等日志宏的format string格式符必须与参数类型严格匹配。常见错误: %s对应整数参数(undefined behavior,可能导致crash)。

```cpp
// bad -- %s期望const char*, 但传入s32
HCCL_ERROR("[Func] logicId[%s]", deviceLogicId_);   // UB: s32当指针解引用

// good
HCCL_ERROR("[Func] logicId[%d]", deviceLogicId_);
```

出处: phase7-validation-report.md GAP-10, commit 0aec21e1

## 10.2 资源管理

### VAL-04: 参数置null无效(反模式)

适用范围: 项目级
强制程度: 强制

free/delete后对函数参数指针置null无效,因为参数是值拷贝,不影响调用者。需使用二级指针或返回值。

```cpp
// bad -- opArgs是局部变量,置null不影响调用者
void HcclKfcFreeOpArgs(HcclOpArgs *opArgs) {
    free(opArgs);
    opArgs = nullptr;   // 无效: 调用者的指针不变
}

// good -- 使用二级指针
void HcclKfcFreeOpArgs(HcclOpArgs **opArgs) {
    free(*opArgs);
    *opArgs = nullptr;
}
```

出处: phase7-validation-report.md GAP-05, commit 4a033793; 与BUG-04同模式

### VAL-05: 析构路径容器遍历安全

适用范围: 项目级
强制程度: 强制

禁止在range-based for遍历容器时通过函数调用间接修改该容器(erase/insert)。这是未定义行为,可能导致崩溃。

```cpp
// bad -- DeregisterSymmetricMem内部会erase windowMap_的元素
for (auto& pair : windowMap_) {
    DeregisterSymmetricMem(pair.first);  // UB: 遍历中修改容器
}

// good -- 先收集key再逐个删除
std::vector<void*> keys;
for (auto& pair : windowMap_) {
    keys.push_back(pair.first);
}
for (auto key : keys) {
    DeregisterSymmetricMem(key);
}
```

出处: phase7-validation-report.md GAP-08, commit a9fea200

### VAL-06: 注册数据生命周期

适用范围: 项目级
强制程度: 推荐

通过HcommRegOpInfo等注册接口传递的数据,其生命周期必须覆盖外部系统的使用期。栈上局部变量不适合作为注册数据,除非确认外部系统在注册时就完成了深拷贝。

出处: phase7-validation-report.md GAP-07, commit deb37cc2

## 10.3 编码风格

### VAL-07: CHK_PRT_RET不用于正常流程控制

适用范围: 项目级
强制程度: 推荐

CHK_PRT_RET/CHK_RET等错误检查宏只用于错误路径。正常的条件分支(如单rank快速返回HCCL_SUCCESS)应使用普通if语句,避免误导读者以为这是错误处理。

```cpp
// bad -- 用错误检查宏做正常分支
CHK_PRT_RET(isSingleRank_,
    HCCL_INFO("single rank, skip"), HCCL_SUCCESS);

// good -- 普通if做正常分支
if (isSingleRank_) {
    HCCL_INFO("single rank, skip");
    return HCCL_SUCCESS;
}
```

出处: phase7-validation-report.md GAP-09, commit a9fea200

### VAL-08: 文件末尾换行符

适用范围: 项目级
强制程度: 推荐

所有源文件必须以换行符结尾(POSIX规范)。缺少换行符会导致git diff显示额外警告。

出处: phase7-validation-report.md GAP-04, commit 9bcb1bdc/4a033793
