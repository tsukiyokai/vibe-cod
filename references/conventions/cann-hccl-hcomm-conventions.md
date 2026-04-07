# CANN C++ Conventions (hccl & hcomm)

本文档从hccl和hcomm两个仓库的系统性代码分析中提取，供vibe-cod skill消费。
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

### VAL-09: 跨线程共享的成员变量必须使用atomic或mutex

适用范围: 项目级
强制程度: 强制

跨线程访问的标志位/计数器，即使是成员变量(非全局)，也必须使用std::atomic<T>或mutex保护。裸bool/int在C++标准下的并发读写是未定义行为(即使实际上在x86/ARM64通常"可以工作")。

```cpp
// bad -- 裸bool跨线程
class MemNameRepository {
    bool unavailable_ = false;  // Recovery线程写，清理线程读
};

// good -- atomic保护
class MemNameRepository {
    std::atomic<bool> unavailable_{false};
};
```

现有AP-GLOBAL-03/FW-AP-03只覆盖全局变量; 本规则扩展到per-device单例等非全局但跨线程共享的成员变量。

出处: phase13-final-validation.md GAP-V1, commit b791de5e(ipc_memory_destroy中unavailable_裸bool跨线程)

### VAL-10: 禁止throw字符串字面量

适用范围: 项目级
强制程度: 强制

throw必须抛std::exception派生类(如std::runtime_error/std::invalid_argument)。禁止throw字符串字面量、int等非标准类型——它们无法被catch(const std::exception&)捕获，会穿透FW-ID-04的EXCEPTION_CATCH三层捕获中的前两层，丢失错误信息和调用栈。

```cpp
// bad -- throw字符串字面量
throw "Invalid Arithmetic Operator";

// good -- throw标准异常
throw std::runtime_error("Invalid Arithmetic Operator");
```

注: CCU层TODO标注"需要统一整改为不抛异常"。长期方向是用HcclResult替代throw; 短期修复用std::runtime_error替代字符串字面量。

出处: phase13-final-validation.md GAP-V2, commit 58037fd5(ccu_operator_v1.h throw字符串字面量修复)

### VAL-11: 头文件自包含

适用范围: 项目级
强制程度: 推荐

每个头文件必须include它使用的所有类型定义，不依赖间接include传递。即: 单独include这个头文件不会产生编译错误。

```cpp
// bad -- 依赖间接include
// ccu_common.h 使用uint32_t但没有 #include <cstdint>
struct CcuParam { uint32_t loopNum; };  // 依赖某个先于此头文件被include的文件提供uint32_t

// good -- 自包含
#include <cstdint>
struct CcuParam { uint32_t loopNum; };
```

R-1.4的-Werror会在某些编译配置下暴露此问题(不同平台的间接include链不同)，但显式自包含是更可靠的做法。

出处: phase13-final-validation.md GAP-V3, commit 58037fd5(5个CCU头文件补充缺失include)

---

# 第11章: Developer's Casebook (案例判例法)

本章从四仓库1735条commit中精选50个landmark case，提供conventions无法覆盖的"判断力"。
conventions告诉你"做什么"，casebook告诉你"在具体场景下怎么判断"。

深度分析详见: references/dig-repo-code/hccl-hcomm/phase10-feature-arch.md / phase10-bugfix-lifecycle.md / phase10-opt-protocol.md / phase10-refactor-revert.md

## 11.1 案例索引

### FEATURE类(9个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| FT-1 | hccl | 549c4537 | new selector(220文件) | 核心参数结构体扩展用继承+尾部追加序列化; 220文件联动次日暴露CCU scatter bug(BF-9)，大规模selector重构必须全算子全设备回归 |
| FT-2 | hccl+hcomm | 12a2c84f | A5 aicpu communicator | 新芯片通信域优先在framework/next新建独立模块，从旧巨型类中删芯片专用代码; 跨仓4分钟内同步，hccl侧极少hcomm侧承担主要工作 |
| FT-3 | hcomm | fa8ed355 | double die alg | CCU算法必须遵循context+instruction+template三层文件结构; 一个拓扑通常适配4+种集合通信算子，10-15新文件+2000-3000行 |
| FT-4 | hccl | 3e916555 | DPU reducescatter | 与现有device-side差异太大时应新建独立Op目录和Executor; DPU是Op→Executor→Template两层(跳过hcomm) |
| FT-5 | hcomm | 0d9239b5 | alg selector remake | hccl层和hcomm层selector独立演进、概念对齐但实现独立; 枚举值不一致是跨仓演进代价 |
| FT-6 | hcomm | ebb54874 | mix-running(同日revert) | 跨V1/V2边界37文件+1140行合入即回退; 影响op_base.cc的大功能应分步合入+feature flag |
| FT-7 | hcomm | 4c779e13 | symmetric memory | 新增内存管理子系统初始版本几乎必然有资源释放遗漏; 3930行中1245行UT(32%)是标杆 |
| XR-2 | hccl-dev+hcomm-dev | 0ca13cd6+79e73f3c | scatter switch to aicpu | 跨仓执行模式切换必须原子提交(18秒间隔); hccl做减法hcomm做调整 |
| CL-1 | hcomm | d0d5cce1 | multi process per rank | Legacy层30文件全新功能开发是现实; 端口管理从硬编码升级配置化的"初始化后补充配置"模式 |

### BUGFIX类(9个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| BF-1 | hcomm | 75659d24 | 重执行多线程时序 | 先清物理资源再清异常标记; 34文件feature commit间接破坏retry路径——feature review时需审查retry/异常/并发路径 |
| BF-2 | hcomm | 1535b1c4 | 读写锁→原子变量 | 通信框架并发控制优先atomic(CAS+spinloop); 粗粒度mutex几乎必然导致性能退化(对比c546d353被revert) |
| BF-3 | hcomm | 994390df | Step快恢失败 | 状态机初始状态必须在构造/注册时显式设置; 设备侧容错逻辑必须有环境判断门控 |
| BF-4 | hcomm | 9bcb1bdc | CP算法三独立bug | 分层pipeline层间数据依赖必须显式建模; 同commit多fix应确保每个fix有独立测试 |
| BF-5 | hccl | 992fd5b | CCU NHR参数错误 | CCU threadNum/loopNum错误值导致症状(死锁)与根因(参数)距离很远; 新CCU template必须对每个GetXxxNum做单元验证 |
| BF-6 | hccl | b779699 | AICPU .so缺符号 | 新增.cc必须同时检查两个CMakeLists.txt(Host/Device); 引用Host-only API需AICPU_COMPILE条件编译 |
| BF-7 | hccl | 3e9fbbd | A3 fallback路由 | fallback条件顺序决定芯片走哪条路径; 结构体新增字段后所有手动偏移计算需同步更新 |
| BF-8 | hcomm | fa61b9fc | Thread API跨层bug | 跨Host-Device资源需双向映射管理(D2H Map); 销毁时必须完整清理所有映射条目 |
| BF-9 | hccl | 80a70729 | CCU scatter地址时机 | CCU初始化必须放CCU_WHILE外部; CCU控制流不等同于C++控制流 |

### OPTIMIZATION类(3个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| OPT-1 | hcomm | fd9d488c | groupSendRecv小数据 | 批量编排替代逐个执行; 128KB阈值需权衡编排开销和传输时间 |
| OPT-3 | hcomm | a98eb47a | AICPU微优化 | UNLIKELY宏是最低风险最高收益的热路径优化; 同时增加日志是良好实践 |
| OPT-4 | hcomm | 8b0dc7b9 | RS&AAV FastLoad | CCU FastLoad扩展: 增opType判断+设计ccuParamsMappingKey+特殊拓扑后处理; CachedCCUParams需正确传递move语义 |

### PROTOCOL类(6个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| PL-1 | hcomm | 93222924 | HB check(main版) | 协议功能必须提供运行时开关(默认关闭); "introduce→revert→add-switch-redo"是标准上线模式 |
| PL-2 | hcomm-dev | ebfcde5c | HB check重做(dev版) | 同PL-1; inconsistentCheckSwitch默认false，通过HCCL_DFS_CONFIG控制 |
| PL-3 | hcomm-dev | 0e82e543 | scatter notify跨引擎 | 跨引擎资源共享应在Thread管理层做抽象(Export/Import); 双向映射表+互斥锁是标准模式 |
| PL-4 | hcomm-dev | 9ceead3b | topo detect retry | 建链重试必须: (1)总超时不变从中切分; (2)非末次降低日志级别; Server/Agent双边同步修改 |
| PL-5 | hcomm-dev | 28f18ab5 | OpRetry简化changeLink | 全局变量传递状态机决策是反模式; "宁可多做恢复也不漏做"更安全; 净减25行同时解决正确性和并发问题 |
| PL-6 | hccl+hcomm | 89a09db2 | host&device sync(3维度) | 跨Host-Device同步必须纳入框架资源管理(Thread/Notify/Stream); 裸ACL调用导致泄漏风险; 跨仓21秒原子提交 |

### ARCH_ADAPT类(4个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| AA-1 | hccl-dev+hcomm-dev | 217dd2cf | ABI兼容+接口改名 | 跨仓接口必须有ABI头(version+magicWord+size); __attribute__((weak))弱符号解耦编译 |
| AA-2 | hcomm-dev | 88f70272 | A3 AIV批量适配 | 标准模式: _cn后缀kernel注册 + _for_910_93 Executor + _crossnode_91093 Template |
| AA-3 | hcomm | c8a15c3e | A3 midcount优化 | 新分支插入selector优先级链正确位置; 阈值(16KB-256KB)必须有性能数据支撑 |
| AA-5 | hcomm | 4afb1c39 | RoCE flush适配 | capability-based(GetLbMax)比device-type-based更具前向兼容性 |

### LIFECYCLE类(4个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| LC-1 | hcomm | e63840b6 | AlltoAll symmetric mem | isZeroCopy标记贯穿algorithm→framework→device; PrepareSymmetricMemory必须遍历所有topology level |
| LC-2 | hcomm | 3315532a | memDesc所有权 | 序列化产物所有权应属于产生它的对象; HCCL_E_AGAIN路径也必须返回有效handle |
| LC-3 | hcomm | 9167c142 | 资源刷新判断 | 是否需要刷新的判断应在使用方(device kernel)而非提供方(host service); 跨域flag传递脆弱 |
| LC-4 | hcomm | c5443da0 | stream资源耗尽 | 硬件资源(stream/event/notify)有限; "每实例独占"不可行; 优先共享资源+同步原语(event) |

### REFACTOR类(3个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| RF-1 | hccl-dev+hcomm-dev | c696e52f | 算子入口调整 | "thin wrapper+Inner suffix"模式: 旧位置重命名XXXInner，新位置一行转发; 两仓11分钟间隔 |
| RF-2 | hcomm-dev | 4fc686be | hccl_fwk→hcomm重命名 | 模块重命名应在仓库初始化阶段一次性完成; 检查所有CMakeLists.txt target引用 |
| RF-3 | hcomm-dev | e10d1002 | 拆分大文件 | extract+redirect模式: 新文件承接被抽出函数，原文件include新头文件; 行数应基本持平 |

### INFRA类(2个)

| ID | Repo | Hash | 标题 | 可迁移经验 |
|----|------|------|------|-----------|
| XR-4 | hccl-dev+hcomm-dev | de127490 | API Changes多轮 | 跨仓API变更不要期望一次做对; 预留revert余量; 首次变更尽量小 |
| CL-2 | hcomm | c5709dd4 | EngineCtxDestroy API | legacy/next共存期新增API必须双路径实现+双路径UT; 先实现legacy路径(当前主力) |

### REVERT类(10个)

| ID | Repo | Hash | 原始变更 | Revert原因 | 时间窗口 |
|----|------|------|---------|-----------|----------|
| RV-1 | hcomm-dev | 1844b823 | ars code(51文件) | 依赖的基础设施未先导合入 | 次日 |
| RV-2 | hcomm | 0c033874 | mix-running(37文件) | 设计缺陷(临时方案合入主干) | 10小时 |
| RV-3 | hcomm | 753ba8c2 | offline compile(24文件) | cmake文件重命名遗漏引用 | 1小时 |
| RV-4 | hccl-dev | 52bda686 | 安装路径变更(18文件) | 路径不兼容 | 未知 |
| RV-5 | hcomm-dev | 30e25e50 | Runtime interfaces | 外部依赖(Runtime/ACL)未就绪 | 4天 |
| RV-6 | hcomm-dev | 1d8e2c14 | rts interfaces | 接口兼容性问题 | 7天 |
| RV-7 | hcomm-dev | a37e6cf1 | HB check(不可关闭) | 功能无开关只能整体Revert | 2天 |
| RV-8 | hcomm | 0c3be05f | Revert "Revert loopnum" | 重新引入(17天验证后) | 17天 |
| RV-9 | hcomm | 72cdf80e | fix loopnum=128 | CCU参数验证不足 | 6小时 |
| RV-10 | hcomm | f0666214 | 910_95 sdma | 三合一反模式(硬编码芯片名/nullptr信号/枚举混用) | 6小时 |

## 11.2 Error→Fix对

变更引入问题 → 后续commit修复。间隔越短 = 问题越表层。

| ID | Error Commit | Fix Commit | 间隔 | 模式 |
|----|-------------|-----------|------|------|
| EF-1 | d17b1f3a(alltoallv cache, 34文件) | BF-1(75659d24, 2行) | 10小时 | 大feature间接破坏retry时序 |
| EF-2 | FT-6(ebb54874, 37文件) | RV-2(0c033874, 全量回退) | 10小时 | 设计缺陷只能全量回退 |
| EF-3 | 99d2a2b3(ars code, 51文件) | RV-1(1844b823) → 62e4b1c2(6天后重引入) | 次日+6天 | 时机问题需等基础设施就绪 |
| EF-4 | FT-1(549c4537, 220文件) | BF-9(80a70729) | 1天 | 大规模selector重构暴露CCU bug |
| EF-5 | SetDispatcherCtx + OpRetry timeout | PL-5(28f18ab5) | 同日 | 全局变量竞态 |

## 11.3 Revert链模式

6条典型Revert链揭示不同时间尺度的失败根因:

Chain 1 — loopnum(1行→3commit→17天):
  fb56d64b(fix loopnum=128) → RV-9(6小时Revert) → RV-8(17天后重引入)
  教训: CCU参数验证周期是周级不是小时级

Chain 2 — HB check(引入→Revert→加开关重做，34天):
  (init引入) → RV-7(a37e6cf1, 2天Revert) → PL-2(ebfcde5c, 加开关) → PL-1(93222924, main版)
  教训: 核心基础设施功能必须带开关; "introduce→revert→add-switch-redo"是标准模式

Chain 3 — mix-running(10小时，未恢复):
  FT-6(ebb54874) → RV-2(0c033874)
  教训: "临时方案"不应合入主干; 跨V1/V2大改无feature flag必被revert

Chain 4 — offline compile(1小时):
  b3c0ad3e(Part1) → 05b38411(Part2) → RV-3(753ba8c2, 1小时) → 6a7c0c81(次日重引入)
  教训: cmake重命名前grep -r确保无遗漏引用

Chain 5 — sdma(6小时，未恢复):
  0e5be69c(sdma support) → RV-10(f0666214)
  教训: 三条反模式(硬编码芯片名/nullptr信号/枚举混用)

Chain 6 — ars code(次日→6天恢复):
  99d2a2b3 → RV-1(1844b823, 次日) → 62e4b1c2(6天后)
  教训: 50+文件算法引入应将基础设施变更独立为先导commit

Revert时间窗口与根因的相关性:
- <1小时 = CI/编译失败(RV-3)
- 6-10小时 = 功能测试/设计缺陷(RV-2/RV-9/RV-10)
- 次日 = 集成测试问题(RV-1)
- 周级 = 外部依赖/硬件验证(RV-5/RV-6/RV-8)
- 月级 = 生产暴露(RV-7→PL-2, 34天)

## 11.4 跨类别横切面模式

### CB-CROSS-01: 跨仓提交不对称性

hccl改1-4文件, hcomm改20-47文件(比例约1:10)。
hcomm先提交(承载主要实现), hccl后提交(thin wrapper/selector适配)。
时间窗口: 18秒(XR-2) ~ 4分钟(FT-2) ~ 11分钟(RF-1)。

出处: FT-2/XR-2/RF-1/PL-6/AA-1, phase10-feature-arch.md

### CB-CROSS-02: 被Revert变更的共性缺陷

被revert变更的四大共性: (1)缺功能开关(30%); (2)缺测试; (3)变更范围过大; (4)隐式假设。
3/10未恢复(设计层面问题), 7/10恢复(时机/验证/灰度问题)。

出处: RV-1~RV-10完整分析, phase10-refactor-revert.md

### CB-CROSS-03: Host-Device边界是bug富矿

13个BUGFIX案例中11个涉及Host-Device边界。
设备侧BUGFIX密度43.5% vs Host侧30.3%(Phase 8.8数据)。
根因分布: 时序/顺序(33%) > 并发/锁(22%) = 编译条件遗漏(22%)。

出处: BF-1~BF-9, phase10-bugfix-lifecycle.md + phase8-device-vs-host.md

### CB-CROSS-04: 全局变量是通信框架反模式

PL-5删g_isRdmaError, PL-6删g_notifies_host_with_device。
替代方案: per-communicator状态 + 框架资源管理体系。

出处: PL-5/PL-6, phase10-opt-protocol.md

### CB-CROSS-05: 核心基础设施新功能的成功模式

步骤: (1)功能开发+默认关闭开关; (2)内部验证; (3)小范围灰度; (4)开关默认打开。
env_config解析成本远低于revert成本。

出处: PL-1/PL-2/RV-7/Phase 8.9(retry默认关闭), phase10-opt-protocol.md

## 11.5 规则-案例交叉引用

现有convention规则与案例的对应关系(供自检时快速定位证据):

| 规则 | 验证案例(遵循) | 违反案例(反例) |
|------|---------------|---------------|
| P-ID-01 CHK_RET 100%覆盖 | BF-3(缺检查→快恢失败) | BF-1(CHK_RET无法捕获时序错误) |
| P-ID-04 new(nothrow)+CHK_PTR_NULL | FT-7(symmetric mem) | RV-10(nullptr作为"不可用"信号) |
| P-AP-01 全局可变状态 | — | PL-5(g_isRdmaError), PL-6(g_notifies) |
| R-1.3 hccl单向依赖hcomm | RF-1(thin wrapper), PL-6(21秒原子) | — |
| R-1.10 V1/V2双代架构 | FT-2(A5走V2), CL-2(双路径UT) | FT-6(跨V1/V2 37文件revert) |
| FW-ID-06 unique_ptr+显式析构序 | LC-2(所有权转移) | BF-8(D2H映射残留) |
| AP-GLOBAL-07 忙等待无退避 | BF-2(CAS+CPU_PAUSE) | c546d353(被revert的粗mutex) |
| TST-02 TEST_F唯一测试宏 | FT-7(32% UT覆盖) | — |
| VAL-01 struct字段默认初始化 | BF-3(状态机初始值) | — |

---

# 第12章: 场景化开发指南

本章面向开发任务组织，每个场景 = 决策清单 + 常见陷阱 + 案例引用。
决策清单按执行顺序排列，每步可直接转化为编码动作。

## 12.1 给hccl新增一个集合通信算子

决策清单:
1. 确定执行模式: Device-side(主路径) vs DPU Host-side(FT-4独立两层架构，跳过hcomm)
2. 定义Op入口: 参照AllReduce的三重门控→参数校验→TopoInfo→算法选择模式(Phase 5.4)
3. 实现Selector 5路分派: SelectDPUAlgo→SelectCcuMsAlgo→SelectCcuScheduleAlgo→SelectAivAlgo→SelectAicpuAlgo(Phase 3)
4. 选择Executor: 继承InsCollAlgBase(V2主力，28个子类) 而非ExecutorBase(V1仅scatter用)
5. 决定hcomm路径: V2 framework/next(新芯片) vs V1 framework/device(旧芯片)(FT-2)
6. 扩展alg_param.h: 继承+尾部追加序列化，避免中间插入枚举值(AA-1 ABI头)
7. 更新ST测试: hccl_stub.cc同步更新，覆盖全模式/全数据类型/全拓扑; 目标UT占比32%(FT-7标杆)

常见陷阱:
- FT-1→BF-9: 大规模selector重构(220文件)次日暴露CCU scatter地址时机bug
- FT-6: 跨V1/V2边界+op_base.cc大改+无feature flag = 10小时revert
- BF-7: 多芯片fallback路由条件顺序决定走哪条路径，新增芯片必须逐行审查整个fallback链
- scatter(20次修改): 初始设计不稳定导致反复修改，设计上应为演进而非完美

## 12.2 给hcomm新增一种通信算法

决策清单:
1. 选择base class: AlgorithmAivBase(单阶段算法) vs AllReduceOperatorAiv(多阶段pipeline)
2. 使用HCCL_KERNEL_DISPATCH_XXX宏按数据类型注册kernel(6-9种原子类型，9种copy-only)
3. 严格遵循同步四步: WaitLocal→DataCopy→SignalRemote→WaitRemote(顺序不可变)(BF-4)
4. 选择数据搬移引擎: MTE2(DDR→L1) / MTE3(L1→DDR) / SDMA/RDMA(跨节点)
5. 如果走CCU: 创建三层文件(context+instruction+template)，enum在末尾追加(FT-3)
6. 注册到communicator: V1(legacy) vs V2(next)根据设备类型决定(FT-2/FT-5)
7. 定义拓扑约束: ring/mesh/2-die fullmesh，在Selector中显式预过滤

常见陷阱:
- BF-4: pipeline层间数据依赖(谁先获信息、谁先启动)未显式建模导致三个独立bug
- BF-5: CCU threadNum 1→2一个数字的差异导致死锁——参数影响的间接性极高
- RV-9: CCU参数(loopnum=128)验证需17天硬件周期，不是6小时UT能覆盖
- OPT-4: CachedCCUParams的move构造必须传递所有新增字段(insType)

## 12.3 修复一个通信超时/死锁bug

决策清单:
1. 日志溯源: 沿CHK_RET九层传播路径从API→driver/runtime反向追踪(Phase 5.3)
2. 检查Notify三步: Send/Wait/Release哪步卡住(Phase 2.1)
3. 验证错误码转换: hccl↔hcomm边界HcclResult↔HcomResult，static_cast可能产生无意义值
4. 区分同步传播(CHK_RET立即返回) vs CQE异步检测(延迟发现)(Phase 5.3)
5. 重执行路径: 先清物理资源(buffer/SQ)再清异常标记，顺序决定正确性(BF-1)
6. 检查锁粒度: 纯atomic优于全局mutex; spinloop CAS比condition_variable更稳(BF-2)
7. 全量回归: semantics_check + 全算子×全拓扑，不要只跑复现场景

常见陷阱:
- BF-1: 34文件feature commit间接破坏retry时序，2行语句重排修复——根因定位极难
- BF-3: 重执行FSM初始状态未在注册时显式设为RUNNING，默认值未定义
- BF-9: CCU_WHILE/CCU_IF是CCU控制指令不是C++控制流，地址初始化放循环体内会每轮执行
- Phase 8.8: 13个BUGFIX案例11个涉及Host-Device边界; 设备侧BUGFIX密度43.5% vs Host侧30.3%

## 12.4 适配一个新芯片平台

决策清单:
1. 能力探测: 优先capability-based(运行时API检测如GetLbMax)(AA-5) 而非device-type-based(硬编码if/else)(AA-2)
2. 条件编译: 区分Host/AICPU双目标; #ifndef AICPU_COMPILE保护Host-only API(BF-6)
3. 降级路径: 显式错误码(非nullptr)表示"功能不可用"; 两条路径都需完整可测(RV-10反模式)
4. 跨仓原子执行: hcomm先提交(主要实现)，hccl后提交(wrapper); 1:10文件比例(FT-2)
5. ABI兼容: HcclAbiHeader(version+magicWord+size+reserved) + __attribute__((weak))弱符号(AA-1)
6. 标准命名: Executor用_for_910_93后缀, kernel用_cn后缀, template用_crossnode_91093(AA-2)
7. 平台ST测试: 覆盖kernel注册+降级路径+CCU参数+跨仓集成

常见陷阱:
- RV-10(f0666214): 硬编码芯片名+nullptr信号+枚举混用三合一反模式，无测试→6小时revert
- BF-7(3e9fbbd): A3 fallback路由条件顺序错误; sizeof偏移量在结构体变更后未同步
- BF-6(b779699): 新.cc仅加入一个CMakeLists.txt; AICPU_COMPILE guard遗漏; 链接时才暴露

## 12.5 优化某算法的通信性能

决策清单:
1. 定位瓶颈层: Selector(算法选择)(AA-3) vs Executor(编排)(OPT-1) vs Algorithm(pipeline)(BF-4) vs CCU microcode(参数)(RV-9)
2. 分析pipeline层间依赖: 多阶段pipeline需显式依赖建模(BF-4三独立bug)
3. buffer策略: slice对齐(32B/512B)，per-stream隔离，数据量阈值有测量支撑(OPT-1: 128KB)
4. CCU参数优化是极高风险: loopnum一行改动验证周期17天(RV-9); FastLoad扩展需正确传递所有字段(OPT-4)
5. 低风险微优化: UNLIKELY标注错误分支(约30处)(OPT-3); early-exit重排; 同步增加可观测性日志
6. 多平台验证: A2/A3单机/双机，测量实际效果; 准备回退预案(环境变量控制)

常见陷阱:
- RV-9: CCU_MS_DEFAULT_LOOP_COUNT 64→128, 6小时revert, 17天后才重新引入——1行代码的影响面覆盖所有CCU调度路径
- BF-4: 多层pipeline同一commit修复3个独立bug，每个fix需独立UT验证
- OPT-1: big/small双路径增加维护成本; 如果收益<5%，单路径更好

## 12.6 做一次跨模块重构

决策清单:
1. 影响范围量化: grep统计被修改接口的所有调用点; 头文件变更触发多少文件重编译(Phase 1 alg_param.h)
2. 分步提交: 基础设施/依赖先行→核心逻辑→注册/链接→测试(RV-1反例: 基础设施未先导→51文件次日revert)
3. 跨仓接口: thin wrapper+Inner suffix; hcomm先提交(依赖方), hccl后提交(被依赖方); 11分钟间隔(RF-1)
4. V1/V2双路径: 两条路径都需实现+独立UT文件(CL-2: 29文件含双路径测试)
5. legacy层: 40%的commit触及legacy(Phase 8.7); 重构legacy需极度谨慎，优先最小适配而非大改
6. 回归验证: 纯重构行数应基本持平(+X/-X); 如果+2000/-500则可能夹带功能修改(RF-3标杆)

常见陷阱:
- RV-1(1844b823): 代码正确但时机不对——51文件算法依赖的基础设施未先导合入
- RV-3(753ba8c2): cmake文件重命名遗漏引用→1小时CI失败revert; 提交前grep -r确认
- RF-1: 跨仓提交顺序: hcomm(依赖方)先导出，hccl(被依赖方)后导入
- XR-4: 跨仓API变更平均需2-3轮迭代才能稳定; 预留revert余量

## 12.7 修复资源泄漏

决策清单:
1. 识别资源类型: Stream/Notify/Channel/CommMem/对称内存/Thread handle各有独立管理机制
2. 追踪完整生命周期: 创建点→使用点→销毁点; 绘制所有权转移图(LC-2 memDesc所有权属于对象)
3. 验证引用计数: 每个increment有对应decrement; refCount==0在所有路径触发; 多线程需atomic(FT-7)
4. 审计异常路径: CHK_RET每层的early return不能跳过cleanup; retry FSM路径正确释放(BF-3)
5. 评估RAII可行性: host侧可用(mutex/vector); device侧需显式API(Create/Destroy对); 跨域映射需完整清理(BF-8)
6. 并发安全: atomic+spinloop优于mutex(BF-2); 资源释放的线程安全性

常见陷阱:
- FT-7: 新内存子系统初始版本必有泄漏(3930行后续修复a9fea200); 先写leak检测UT
- BF-8: D2H映射残留→use-after-free; 销毁时清理所有映射条目
- LC-4: per-model/instance独占分配耗尽stream; 改为全局stream+Event
- LC-2: HCCL_E_AGAIN路径也必须返回有效handle

## 12.8 修改CCU/AICPU设备侧代码(hccl/hcomm独有)

决策清单:
1. 判断变更类型: Type A(microcode参数, 1-3行, 影响巨大) / Type B(kernel逻辑, 10-50行) / Type C(新kernel, 100-1000行)
2. Host-Device参数一致性: param.hcclComm赋值时机、结构体偏移计算、反序列化正确性(BF-7/BF-9)
3. 执行模式选择: CCU MS(pipeline)/CCU Schedule/AICPU+AIV/AICPU/Legacy; 模式切换需跨仓原子(XR-2, 18秒)
4. 同步模型: kernel内部(SyncAll/WaitLocal/SignalRemote) vs kernel间(Notify三步); 从裸ACL迁移到Thread Export框架(PL-6)
5. DFX可观测性: task_exception入口, 导出notify_id/buffer_addr/slice_size(OPT-3)
6. 验证策略: Type A需17天硬件验证周期(RV-9); Type B需AICPU_COMPILE guard; Type C需三层文件结构(FT-3)
7. 回退预案: Type A通过环境变量; Type B通过feature flag; Type C通过Selector fallback路径

常见陷阱:
- RV-9: CCU常量(loopnum)1行修改→17天才完成硬件验证; 影响覆盖所有CCU调度路径
- BF-9: CCU_WHILE循环内的地址初始化每轮执行; CCU微码语义不等于C++循环语义
- BF-5: GetThreadNum 1→2一个数字修复死锁; 参数对调度的影响路径极长
- BF-6: AICPU_COMPILE guard遗漏→链接阶段才暴露; 新.cc必须检查两个CMakeLists.txt
- Phase 8.8: hccl:template/ccu的BUGFIX密度44.4%——每次CCU模板修改波及34文件

## 12.9 修改通信协议(Notify/重执行/心跳)(hccl/hcomm独有)

决策清单:
1. 评估双边变更: 对端是否正确处理新字段/状态/消息; 超时是否匹配(PL-4 Server+Agent同步修改)
2. 添加运行时开关(默认关闭): 环境变量控制，入口短路，关闭时零开销(PL-2用开关逆转revert)
3. 确保FSM完整性: 每个状态有进入/退出条件，所有错误路径处理，初始状态显式(BF-3)
4. 简化重执行路径: 删全局变量(竞态), 合并双路径为单一"总是做完整恢复"路径(PL-5净减25行)
5. 评估心跳帧大小: 新数据结构对50ms帧周期的影响; 动态帧大小per feature flag
6. 多平台差异: 基于硬件能力检测(AA-5 RoCE flush)
7. 并发安全: FSM状态不用全局变量; per-communicator互斥锁; 优先atomic(BF-2)

常见陷阱:
- RV-7: 核心基础设施无开关→只能整体Revert; 34天后才用开关重做(PL-2)
- PL-5: 全局变量传递FSM决策跨线程竞态; 合并双路径更安全
- BF-1: retry路径2行时序修复——先清物理资源再清异常标记
- Phase 8.9: retry从"基本能用"到"生产可靠"需3个月; 2行修复极难定位

## 12.10 将功能从legacy迁移到next/framework(hccl/hcomm独有)

决策清单:
1. 评估迁移时机: 如果目标功能近3个月仍有FEATURE/BUGFIX在legacy中，先稳定; legacy占比40%且在爆发而非衰退(Phase 8.7)
2. 确认功能等价: next实现必须覆盖所有legacy路径; 需V1/V2双UT文件; 分别mock/stub(CL-2)
3. 选择迁移策略: thin wrapper+Inner suffix(RF-1) > 一次性文件移动; 2432文件orion→legacy重命名证明legacy仍是主力
4. 理解legacy依赖结构: communicator_impl.cc(47次修改)是hub; legacy/service(39 commits, 每commit触及45文件)耦合度极高
5. V1/V2分离: 新芯片路由到next(framework/next/coll_comms), 旧芯片留在device; 新建独立模块而非在巨型类中加分支(FT-2)
6. 测试覆盖: legacy路径测试不可直接复用(V1/V2 mock/stub不同); input/output可共享但framework必须分离

常见陷阱:
- legacy/service是最高耦合区: 1748 file-touches in 39 commits(平均45文件/commit)
- communicator_impl.cc: 47次修改(FEATURE 13+BUGFIX 11)——复杂度被低估
- 真正的迁移仅13%: 84/97跨层commit是"同步开发"而非真正迁移
- FT-6: 37文件跨V1/V2功能10小时revert; 必须分步+feature flag
- legacy仍是新功能开发场: 40条legacy-only FEATURE, 88.9%无芯片关键词

---

# 第13章: 灰色地带指南

本章提炼conventions无法覆盖的判断力。每个灰色地带有正反两面案例支撑。
当convention规则不足以决策时，参考本章的判断框架。

## 13.1 何时用V1路径 vs V2路径

核心矛盾: V2(IndependentOp/pImpl委托)更干净但仅A5(910_95)支持; V1(RankGraph全栈)覆盖全平台且仍是活跃开发路径。

判断标准:
- V2路径: 目标仅A5 + 功能完全独立 + next基础设施已就位(FT-2: A5新建独立模块)
- V1路径: 需多芯片支持 + legacy生态依赖 + 修改现有文件为主(CL-1: 30文件全legacy)
- 绝不在单个大commit中跨越V1/V2边界(FT-6: 37文件同日revert的教训)

推荐默认: A5独立功能→V2; 其他→V1。共存期双路径实现成本2x(CL-2)是必须接受的代价。

Phase 8.7数据: legacy从2026-02起占hcomm 40%，不是在衰退而是在爆发; 40条legacy-only FEATURE证明V1仍是主力。

## 13.2 何时在hccl层实现 vs 下沉到hcomm

核心矛盾: hccl(编排) vs hcomm(实现)的边界模糊。两层各有独立selector/CCU模板体系。

判断标准:
- 逻辑是否需要知道通信链路实现细节(socket/RDMA/buffer)? 需要→hcomm; 不需要(仅拓扑/数据量/芯片类型)→hccl
- 算法选择: V2→hccl AutoSelector; V1→hcomm legacy base_selector(FT-1 vs FT-5独立演进)
- CCU模板: hccl templates/→hccl CCU体系; hcomm legacy/→hcomm CCU体系

推荐默认: hccl是薄决策层, hcomm是厚实现层。Feature跨仓比例约1:10(hccl改1-9文件, hcomm改18-110文件)。

## 13.3 何时用深继承 vs 扁平实现

核心矛盾: hcomm(168 pure virtual, 6-7层继承) vs hccl(8 pure virtual, 2-3层)。InsCollAlgBase有28个子类。

判断标准:
- 新子类与父类共享相同执行模型(CalcRes/Orchestrate/KernelRun三步)→继承InsCollAlgBase(FT-1/AA-2)
- 执行模型根本不同(如DPU Host-side)→独立类不继承(FT-4)
- 数据结构扩展→继承+尾部追加(保持向后兼容)(FT-1 TopoInfoWithNetLayerDetails)

推荐默认: 同执行模型→继承V2 InsCollAlgBase; 不同执行模型→独立实现。scatter留在V1不是因为V1更好，而是时机不对(正在迁移执行模式+DPU双路径+技术债密度高)。

## 13.4 新功能放在hcomm哪一层(algorithm/framework/platform)

核心矛盾: 三层各有职责但边界在演进中。对称内存跨三层，Thread Export跨两层。

判断标准:
- 修改C ABI边界(comm_primitive): 仅当transport/notify/mem级别共同基础设施才扩展(AA-5: 5文件36行)
- 多算法复用: →framework层(FT-7 symmetric mem, PL-3 Thread Export)
- 算法专用: →algorithm层
- 初始化判断: "何时判断"在使用方(algorithm), "怎么做"在管理方(framework)(LC-3)

推荐默认: framework层(复用度最高)。不确定时先放framework，根据实际复用模式再迁移。
反面教训: FT-6跨三层37文件同日revert; BF-8跨framework/platform修18文件。

## 13.5 何时做保护性编程 vs 信任调用者

核心矛盾: hcomm CHK_PTR_NULL 2683次、CHK_RET 14943次(极度防御); hccl CHK_PTR_NULL仅159次(信任调用)。

判断标准:
- 跨层边界/API入口: 100%防御(P-ID-01)
- 设备侧热路径: 保留CHK_RET + UNLIKELY宏标注(OPT-3: 约30处UNLIKELY已证明收益)
- 状态机转换: 显式初始化+guard; assert合法性(BF-3: 初始状态未设置→快恢失败)
- Host/Device双编译: 必须guard Host-only API(BF-6)
- 注意: CHK_RET无法捕获时序错误(BF-1) 和偏移错误(BF-7)——防御编程有盲区

推荐默认: 保持100%覆盖基线; 设备侧热路径UNLIKELY标注; 不移除运行时检查。UNLIKELY使用281次(仍有巨大标注空间)。

## 13.6 何时重构legacy vs 在next中重新实现

核心矛盾: legacy占修改40%且在爆发(非衰退); next仍在建设(17个TODO); 真正迁移仅13%。

判断标准:
- 紧迫修复(生产bug) + 修改范围可控(legacy内部) + next未成熟→在legacy修复(BF-3: 3文件精准修复)
- 新芯片 + next基础设施就位 + 可做减法(从legacy删代码)→在next实现(FT-2: A5独立模块+从legacy删63行)
- 跨legacy/next边界大改→极高风险(FT-6: 37文件同日revert)

推荐默认: 共存期在legacy做维护，在next做新芯片功能; 不跨边界大改。88.9%的legacy commit无芯片关键词——legacy不是旧芯片专用代码。

## 13.7 CCU调度 vs AICPU执行: 何时用哪种(hccl/hcomm独有)

核心矛盾: CCU性能高但BUGFIX密度44.4%(每commit波及34文件); AICPU灵活但绝对bug量更多(80条)。

判断标准:
- 新算法首次实现 + 无需硬件级pipeline→先AICPU, 稳定后再优化为CCU
- 需要硬件级通信-计算overlap(如double-die pipeline) + 模板base class已存在→直接CCU(FT-3)
- CCU参数修改: 极高风险，grep所有引用, 确认在CCU_WHILE外, 预期周级硬件验证(RV-9)
- CCU↔AICPU模式切换: 跨仓原子提交(XR-2: 同作者18秒间隔)

CCU微码安全准则:
- 初始化代码必须在CCU_WHILE外(BF-9)
- GetThreadNum/GetLoopNum等每个方法需单元验证(BF-5)
- 1行参数改动的影响面可覆盖所有CCU调度路径(RV-9)
- CCU MS(单层拓扑) vs CCU SCHED(多层拓扑)

推荐默认: 新算法→AICPU mode first, 稳定后再CCU。例外: 已有同族CCU模板时可直接扩展(OPT-4 SuperFastLoad)。

## 13.8 通信协议修改: 渐进式 vs 一步到位(hccl/hcomm独有)

核心矛盾: 渐进式每commit改一个维度风险低但可能碎片化; 一步到位干净但>30文件变更几乎必然失败。

判断标准:
- 新协议功能: 必须有运行时开关(默认关闭); 不可关闭=只能整体Revert(RV-7)
- 开关成本(几十行env_config解析) << revert成本(发布分支协调+团队中断)
- OpRetry状态机变更: 覆盖所有22个状态转换路径的测试
- 系统接口迁移(RTS→ACL): adapter+dlsym动态fallback(RV-5/RV-6)
- Notify帧结构: 极高风险, 必须有开关, 验证对网络带宽的影响

推荐默认: 每commit改一个协议维度 + 运行时开关(默认关闭)。不要force-enable协议功能上线。

Phase 8.9数据: retry从"基本能用"到"生产可靠"经历3个月密集修复期(2025-12到2026-02)。

## 13.9 错误处理: 快速失败 vs 容错继续(hccl/hcomm独有)

核心矛盾: hccl哲学是fail-fast(CHK_RET九层传播); hcomm哲学是fault-tolerant(ExceptionHandler+OpRetry 22状态机)。

判断标准:
- 参数错误/资源耗尽/契约违反→快速失败(CHK_PTR_NULL/CHK_RET)(RV-10 nullptr违反契约)
- 网络/链路故障(可恢复超时)→容错继续: 标记状态, 走OpRetry; 资源cleanup先于异常标记清除(BF-1)
- 建链阶段超时→有限重试: 最多3次, 总超时三等分, 非末次降级为warn(PL-4)
- 容错路径自身出bug→快速失败: 不做嵌套retry(BF-3: 容错路径未初始化)
- CQE异步错误: 不走CHK_RET, 标记状态在下一个同步点重试

推荐默认: 快速失败(fast fail=用户看到错误可重试; 容错继续=系统不一致无法恢复)。
仅当: (1)错误类型可恢复(网络/超时, 非参数错误); (2)恢复路径经过测试验证; (3)故障恢复成本<失败成本 时才容错继续。

## 13.10 五个横切模式(综合)

### GRAY-CROSS-01: 渐进优于激进

>30文件需特别审查, >50文件必须拆分。
成功案例均为小步快走: PL-5(净减25行), PL-4(超时三等分), RF-1(thin wrapper)。
失败案例均为大步快跑: FT-6(37文件同日revert), RV-1(51文件次日revert), RV-7(HB 2天revert)。

### GRAY-CROSS-02: 开关是生命线

无开关的新功能 = 只有revert一条退路。
开关成本(几十行env_config) << revert成本(发布分支协调+团队中断)。
证据: PL-1/PL-2(HB check introduce→revert→redo-with-switch), Phase 8.9(retry默认关闭)。

### GRAY-CROSS-03: 跨边界变更需原子性

hccl-hcomm(PL-6 21秒, XR-2 18秒, FT-2 4分钟), V1-V2(CL-2双UT), Host-Device(BF-9 CCU语义)。
要么原子提交，要么显式分步策略(RF-1 thin wrapper 11分钟间隔)。

### GRAY-CROSS-04: 验证周期与变更层级成正比

构建变更: 1小时CI。Host逻辑: 6-10小时功能测试。设备参数: 17天硬件全场景。协议: 34天集群验证。
不要用host侧验证时间线评估device/协议变更。

### GRAY-CROSS-05: 设备侧简化最有效

设备侧BUGFIX密度43.5% vs Host侧30.3%。13个BUGFIX案例11个触及Host-Device边界。
top修复: PL-5(删全局变量净减25行), BF-1(2行重排), BF-5(1个数字)。
在通信框架中，less is more——简化优于增加复杂度。
