# Phase 2.4: hcomm Test & Example Code

## Module Overview

hcomm测试体系分三层: test/ut/(单元测试)、test/st/(系统测试/算法验证)、examples/(API示例)。
另有test/legacy/(遗留测试, 496文件)保留历史实现，已被新框架大部分替代。

## 2.4.1 Unit Tests (test/ut/)

### Key Files

```
test/ut/
  framework/    76文件  communicator/hcom/next/op_base_api/resource/dpu_npu_sync
  inter/        80文件  comm/hcom/misc (跨模块集成测试)
  misc/         47文件  transport/stream/opretry/heartbeat等组件
  platform/     23文件  hccp/hcom/ping_mesh/resource
  aicpu_kfc/    12文件  AICPU KFC算子 (allreduce/alltoall/reducescatter/retry等)
  device/        5文件  AIV/communicator/stream/zero_copy设备侧测试
  impl/         10文件  hccl_impl/netdevice/dispatcher_ctx/zero_copy等
  stub/         33文件  hccl_llt共享stub库
  common/        2文件  prepare_ut_env/binary_package
  depends/            第三方头文件桩
```

### Build System

构建架构: 单一共享stub库 + 29个独立测试可执行文件

```
test/CMakeLists.txt           -- ENABLE_UT/ENABLE_ST条件开关
test/ut/CMakeLists.txt        -- 275+行include路径定义(UT_COMMON_INCLUDE_LIST) + 30个子模块
test/ut/stub/CMakeLists.txt   -- hccl_llt共享库(GLOB_RECURSE源码树 + 33个stub文件)
cmake/function.cmake          -- run_llt_test()运行函数(SIGKILL超时+XML报告)
```

关键编译定义(所有UT目标):
- `CCL_LLT` -- LLT模式标记
- `OPEN_HCCL_TEST` / `OPEN_BUILD_PROJECT` -- 开源构建标记
- `_GLIBCXX_USE_CXX11_ABI=0` -- ABI兼容
- `HALF_T=float` -- 半精度映射
- `google=ascend_private` -- 命名空间私有化
- `-fno-access-control` -- 允许测试访问私有成员
- `-O0 -g --coverage` -- 调试+覆盖率
- ASAN可选: `-fsanitize=address` (通过 ENABLE_ASAN 条件)

第三方依赖:
- Google Test v1.14.0 (gtest + gmock, 静态链接)
- MockCPP v2.7 (静态链接, 依赖Boost)
- nlohmann JSON v3.11.3 (纯头文件)

### Test Framework Selection

Google Test (gtest) + MockCPP (mockcpp)

gtest用于测试组织和断言, mockcpp用于函数级mock。
值得注意的是未使用gtest自带的gmock, 而是选择了MockCPP -- 可能因为MockCPP支持C函数mock(不需要虚函数),
适合hcomm大量C ABI函数和extern "C"接口的场景。

### Fixture Pattern

量化数据:
- TEST_F: 223个文件使用 (几乎100%)
- TEST_P: 8次/3个文件 (极少, 仅ut_reducescatter_common.cc/ut_op_base.cc/ut_hccl_comm_910B.cc)
- TEST: 个别文件

几乎100%使用TEST_F, 表明强烈依赖共享fixture进行mock初始化和资源管理。

Fixture标准模式 (test/ut/platform/resource/dispatcher/ut_aicpu_dispatcher_utest.cc:39-73):
```cpp
class DispatcherAiCpu_UT : public testing::Test {
protected:
    static void SetUpTestCase()    // 类级初始化(一次)
    static void TearDownTestCase() // 类级清理(一次)
    virtual void SetUp() {         // 每个测试前: 初始化+设置mock
        dispatcherAiCpu->aicpuInfo_.devType = DevType::DEV_TYPE_910B;
        MOCKER(hrtGetHccsPortNum).stubs()...
    }
    virtual void TearDown() {      // 每个测试后: 验证mock
        GlobalMockObject::verify();
    }
    // 成员: 被测对象 + 辅助数据
    std::unique_ptr<DispatcherAiCpu> dispatcherAiCpu =
        std::unique_ptr<DispatcherAiCpu>(new (std::nothrow) DispatcherAiCpu(1));
};
```

关键规则: TearDown必须调用GlobalMockObject::verify()确认mock期望满足。
量化: GlobalMockObject::verify -- 190个文件使用。

### Fixture Inheritance

BaseInit基类提供通信域级测试环境 (framework/resource/hccl_api_base_test.h):
- SetUp: 初始化设备/通信域/buffer, 设置大量MOCKER
- TearDown: GlobalMockObject::verify + 清理
- 辅助宏: UT_USE_RANK_TABLE_910_1SERVER_1RANK, UT_USE_RANK_TABLE_910_2SERVER_4RANK等
- 辅助函数: Ut_Device_Set, Ut_Comm_Create, Ut_Buf_Create, Ut_Stream_Create等

多个测试类继承BaseInit (如CcuDevMgrTest, op_base_api测试等)。

### Mock/Stub Patterns

#### MockCPP基本用法

量化:
- MOCKER(): 186个文件 (mock C函数)
- MOCKER_CPP(): 113个文件 (mock C++成员函数)

```cpp
// C函数mock (test/ut/framework/hcom/ut_HcclGetOpScratchMemSize_API_test.cc:68-71)
MOCKER(HcomGetCommByGroup)
    .stubs()                           // 可被调用任意次
    .with(any(), outBound(comm))       // 参数匹配 + 输出参数绑定
    .will(returnValue(HCCL_SUCCESS));   // 返回值

// C++成员函数mock (test/ut/platform/hcom/ut_hcom_common.cc:82)
MOCKER_CPP(&hcclComm::GetInCCLbuffer)
    .stubs()
    .will(returnValue(HCCL_SUCCESS));

// 重载函数mock: 需显式指定函数签名
MOCKER_CPP(&DispatcherAiCpu::LaunchTask,
           HcclResult(DispatcherAiCpu::*)(hccl::Stream &, bool))
    .stubs()
    .will(returnValue(HCCL_E_INTERNAL))
    .then(returnValue(HCCL_SUCCESS));    // 连续调用返回不同值
```

#### MockCPP API清单

| API | 用途 | 示例 |
|-----|------|------|
| .stubs() | 任意次调用 | 最常用 |
| .expects(once()) | 恰好1次 | 严格验证 |
| .expects(atMost(N)) | 至多N次 | 宽松验证 |
| .will(returnValue(X)) | 返回固定值 | 最常用 |
| .then(returnValue(Y)) | 下次调用返回 | 模拟状态变化 |
| .with(any()) | 任意参数 | 通配 |
| .with(outBound(X)) | 输出参数绑定 | 模拟指针输出 |
| .will(invoke(fn)) | 委托到桩函数 | 复杂行为 |

#### HCCP模块的C ABI Mock封装

test/ut/platform/hccp/stub/ut_dispatch.cc提供了一套宏生成的C ABI mock封装:

```cpp
// 为不同参数数量的C函数生成mock包装
#define MOCKER_APIs_DEFINE(n) \
extern "C" void mocker_p##n(char *nameOfCaller, stub_fn_p##n##_t addrOfCaller, int most, int ret) { \
    MOCKER(addrOfCaller).expects(atMost(most)).will(returnValue(ret)); \
}
```

这允许纯C模块(hccp)在C代码中调用mock注册。

#### Stub库结构

test/ut/stub/目录包含33个stub源文件:
- llt_hccl_stub.h/cc -- 主stub: Runtime模拟(event/stream/thread/memory/notify)
- llt_hccl_stub_pub.h -- 公共stub接口
- llt_hccl_stub_mc2.h -- MC2资源stub (StubHccCommRes, StubSqeBuffer, MC2AicpuProcessStub)
- llt_hccl_stub_fe.h/cc -- 编译器前端stub
- llt_hccl_stub_gdr.h/cc -- GDR stub
- llt_hccl_stub_gert.h -- TBE/GERT图执行stub
- llt_hccl_stub_profiling_plugin.h/cc -- Profiling stub
- llt_hccl_stub_rank_graph.h -- RankGraph拓扑stub
- llt_hccl_stub_sal_pub.h -- SAL stub
- llt_mmpa_stub.c / llt_stub_mm.c / llt_stub_sec.c -- 底层C桩
- tester_pub.h / tester.cc -- 集成测试harness(tester类)
- v80_rank_table.h -- 测试拓扑配置

llt_hccl_stub.h (544行)定义了完整的Runtime模拟基础设施:
- rt_event_stub_t -- 事件模拟(sem+mutex)
- rt_notify_t / rt_shm_notify_t -- Notify模拟(共享内存)
- thread_class -- 线程模拟类(start/stop/handler虚函数)
- stream_task_s -- Stream任务队列模拟
- 各种参数结构(memcpy_async, vector_reduce, rdma_send等)

#### 私有成员访问

量化: `#define private public` -- 147个文件使用

```cpp
// 标准模式 (ut_aicpu_dispatcher_utest.cc:14-18)
#ifndef private
#define private public
#define protected public
#include "dispatcher_aicpu.h"     // 在重定义下include
#endif
// ... 其他include ...
#undef private
#undef protected
```

这是白盒测试的核心手段, 允许直接访问和设置被测对象的内部状态。
147/223 = 66%的测试文件使用此技巧。

### Assertion Style

量化:
- EXPECT_EQ: 210个文件 (主导, 约占断言的90%+)
- EXPECT_NE: 少量使用 (null检查)
- EXPECT_TRUE: 42次/少量文件
- ASSERT_EQ: 15次/5个文件 (极少)

设计选择: 几乎100%使用EXPECT_*而非ASSERT_*, 允许单个测试收集多个失败点,
不会因第一个失败就终止测试。只在关键资源分配必须成功时使用ASSERT_*。

典型断言模式:
```cpp
// 调用被测函数, 检查返回值
HcclResult result = GetOpScratchMemSize(...);
EXPECT_EQ(result, HCCL_SUCCESS);        // 检查成功
EXPECT_EQ(opMemSize, expectedValue);     // 检查输出参数
```

### Test Naming Convention

统一格式: `Ut_<Component>_When_<Scenario>_Expect_<Result>`

示例 (ut_HcclGetOpScratchMemSize_API_test.cc):
```
Ut_GetOpScratchMemSize_When_ReduceScatterNormal_Expect_Success
Ut_GetOpScratchMemSize_When_ReduceScatterDeterministic_Expect_DoubleMemSUCCESS
Ut_GetOpScratchMemSize_When_AllToAllNormal_Expect_Success
Ut_GetOpScratchMemSize_When_BroadcastSmallData_Expect_MemSUCCESS
Ut_GetOpScratchMemSize_When_GetCommByGroupFail_Expect_NotFound
```

也存在简短命名 (ut_aicpu_dispatcher_utest.cc):
```
ut_DispatcherAiCpuSignalRecord
ut_DispatcherAiCpuMemcpy
ut_DispatcherAiCpu_StreamSync
ut_launchTask_rtsqfull
```

类名命名两种风格:
- `ClassName_UT` (如DispatcherAiCpu_UT) -- 下划线后缀
- `ClassNameTest` (如GetOpScratchMemSizeTest) -- Test后缀

### Object Instantiation

两种主要模式:

1. unique_ptr + new(nothrow) (最常用):
```cpp
std::unique_ptr<DispatcherAiCpu> dispatcherAiCpu =
    std::unique_ptr<DispatcherAiCpu>(new (std::nothrow) DispatcherAiCpu(1));
```

2. shared_ptr (需要MOCKER_CPP outBound时):
```cpp
std::shared_ptr<hccl::hcclComm> comm;
// SetUp中:
comm.reset(new (std::nothrow) hccl::hcclComm());
```

均使用nothrow, 与源码保持一致。

### Test Organization

29个独立可执行目标, 每个模块编译为独立二进制:
- hccl_utest_misc -- 最大, 47个测试文件
- hccl_utest_inter_comm / hccl_utest_inter_hcom / hccl_utest_inter_misc -- 跨模块集成
- hccl_utest_framework_communicator / _hcom / _next 等 -- 框架层各子模块
- hccl_utest_platform_* -- 平台层
- hccl_utest_impl -- 实现层
- hccl_utest_device -- 设备侧
- hccl_utest_aicpu_kfc -- AICPU KFC子系统

所有目标链接到hccl_llt共享stub库。

### Anti-Patterns in Tests

AP-UT-1: #define private public破坏封装
- 147个文件使用, 使测试与实现高度耦合
- 任何成员名/类型变化都可能导致编译失败
- 正确做法: 使用友元类或提取接口

AP-UT-2: 代码大量重复
- ut_aicpu_dispatcher_utest.cc每个测试方法重复相同的Stream初始化序列(~8行)
- 应提取为fixture的辅助方法

AP-UT-3: TearDown中GlobalMockObject::verify可能掩盖失败
- 如果SetUp中mock设置失败, TearDown仍调用verify, 可能产生误导性错误

AP-UT-4: #if 0注释掉的测试
- ut_aicpu_dispatcher_utest.cc:83-128 一个完整测试被#if 0禁用
- 应删除或修复

AP-UT-5: 测试中使用HCCL_ERROR日志
- ut_aicpu_dispatcher_utest.cc:121 在测试代码中调用HCCL_ERROR
- 测试应使用gtest输出, 不应使用生产日志系统

AP-UT-6: mock覆盖的函数无类型安全检查
- MOCKER宏通过函数指针工作, 参数类型在编译期不完全检查
- 函数签名变化后mock可能静默失效

AP-UT-7: 测试中裸new/delete
- ut_aicpu_dispatcher_utest.cc:218,227 -- `new u64(1)` + `delete ptr`
- 应使用DeviceMem::alloc或unique_ptr

AP-UT-8: 测试中使用std::cout而非gtest输出
- SetUpTestCase/TearDownTestCase中使用std::cout
- 干扰gtest的标准输出格式

---

## 2.4.2 System Tests (test/st/)

### Overview

test/st/algorithm/ 是hcomm的算法级系统测试, 离线验证算法正确性(无需实际硬件)。

```
test/st/algorithm/
  testcase/testcase/          22个testcase_*.cc (核心测试用例)
  testcase/executor_testcase_generalization/       8个子目录
  testcase/executor_reduce_testcase_generalization/ 8个子目录
  testcase/executor_alltoall_A3_pipeline_testcase/
  utils/adapter_v1/           5个子目录 (运行时适配层)
  utils/checker/              10个子目录 (验证工具)
  utils/inc/ + pub_inc/       公共头文件
```

总计164个源文件。

### Checker Framework

ST的核心创新是Checker验证框架:

```cpp
// test/st/algorithm/testcase/testcase/testcase_all_reduce.cc:67-89
TEST_F(AllReduceTest, allreduce_test)
{
    RankTable_For_LLT gen;
    TopoMeta topoMeta;
    gen.GenTopoMeta(topoMeta, 1, 2, 4);    // 1 superPod, 2 server, 4 dev/server

    CheckerOpParam checkerOpParam;
    checkerOpParam.opType = CheckerOpType::ALLREDUCE;
    checkerOpParam.devtype = CheckerDevType::DEV_TYPE_910B;
    checkerOpParam.DataDes.count = 800;
    checkerOpParam.DataDes.dataType = CheckerDataType::DATA_TYPE_INT32;

    Checker checker;
    HcclResult ret = checker.Check(checkerOpParam, topoMeta);
    EXPECT_EQ(ret, HcclResult::HCCL_SUCCESS);
}
```

Checker验证流程:
1. 通过stub拦截平台/框架层调用
2. 收集所有rank的Task序列
3. 构建有向无环图(DAG)
4. 检测内存读写冲突
5. 验证数据搬运语义正确性

### ST Fixture Pattern

```cpp
// test/st/algorithm/testcase/testcase/testcase_all_reduce.cc:38-65
class AllReduceTest : public testing::Test {
protected:
    virtual void SetUp() {
        // 1. 获取当前测试名, 设置dump文件名
        Checker::SetDumpFileName(caseName);
        // 2. Mock kernel launch(stub掉设备执行)
        MOCKER(ExecuteKernelLaunch).stubs().will(returnValue(HCCL_SUCCESS));
    }
    virtual void TearDown() {
        Checker::SetDumpFileName("analysis_result");
        GlobalMockObject::verify();
        ClearHcclEnv();    // 清理所有setenv设置的环境变量
    }
};
```

与UT的区别:
- ST测试被测对象是完整的算法执行路径(而非单个类/方法)
- ST通过Checker做语义验证(而非简单的返回值检查)
- ST使用setenv配置算法行为(HCCL_DETERMINISTIC等)

### ST Coverage

覆盖的集合通信算子:
- AllReduce, AllGather, AllGatherV
- ReduceScatter, ReduceScatterV
- Broadcast, Reduce, Scatter
- AllToAll, AllToAllV, AllToAllVC
- SendRecv, BatchSendRecv
- AHC(非对称层次), ARS(特殊算法)

硬件设备覆盖: 310P3/910A/910B/910C
拓扑配置: 参数化循环(1-2 server x 1-8 dev/server)
模式: OPBASE/Offload

### Topo Generation

RankTable_For_LLT类提供拓扑生成:
```cpp
gen.GenTopoMeta(topoMeta, superPodNum, serverNum, devPerServer);
```

支持对称拓扑(参数化)和非对称拓扑(自定义vector)。

---

## 2.4.3 Examples

### Overview

examples/仅有1个主题: 通信域初始化, 包含3个场景。

```
examples/01_communicators/
  01_one_device_per_process/         MPI多进程, RootInfo广播
  02_one_device_per_process_rank_table/  RankTable文件配置
  03_one_device_per_pthread/         单进程多线程
```

总计3个main.cc + 1个rank_table.json + 3个Makefile。

### API Usage Pattern

示例展示的标准调用流程 (examples/01_communicators/01_one_device_per_process/main.cc):

```cpp
// 1. 初始化
aclInit(NULL);
aclrtSetDevice(devId);

// 2. 建立通信域
HcclRootInfo rootInfo;
if (devId == rootRank) {
    HcclGetRootInfo(&rootInfo);
}
MPI_Bcast(&rootInfo, HCCL_ROOT_INFO_BYTES, MPI_CHAR, rootRank, MPI_COMM_WORLD);
HcclCommConfig config;
HcclCommConfigInit(&config);
config.hcclBufferSize = 1024;      // MB
config.hcclDeterministic = 1;
HcclCommInitRootInfoConfig(devCount, &rootInfo, devId, &config, &hcclComm);

// 3. 执行通信
aclrtMalloc(&sendBuf, mallocSize, ACL_MEM_MALLOC_HUGE_ONLY);
aclrtMalloc(&recvBuf, mallocSize, ACL_MEM_MALLOC_HUGE_ONLY);
HcclAllReduce(sendBuf, recvBuf, count, HCCL_DATA_TYPE_FP32, HCCL_REDUCE_SUM, comm, stream);
aclrtSynchronizeStream(stream);

// 4. 清理
HcclCommDestroy(hcclComm);
aclrtResetDevice(devId);
aclFinalize();
MPI_Finalize();
```

### Example Error Handling

示例定义了简洁的错误检查宏:
```cpp
#define ACLCHECK(ret)  do { if (ret != ACL_SUCCESS) { printf(...); return ret; } } while (0)
#define HCCLCHECK(ret) do { if (ret != HCCL_SUCCESS) { printf(...); return ret; } } while (0)
```

与源码的CHK_RET相似但更简单, 适合示例代码的可读性需求。

### Tester Harness (test/ut/stub/tester_pub.h)

tester_pub.h定义了一个集成测试harness类:
```cpp
class tester {
    std::vector<tester_rank_info> rank_list;    // per-rank信息
    std::vector<test_task> task_vec;            // 测试任务队列
    HcclResult init_comm(comm_init_para_t&);    // 初始化通信域(All/ByRank两种)
    HcclResult add_test_task(hccl_excute_t, test_para*);  // 添加测试任务
    HcclResult run(task_run_type_t);            // 执行(串行/并行)
    HcclResult check_result();                  // 自动验证结果
};
```

支持6种操作类型(Broadcast/Reduce/AllGather/ReduceScatter/AllReduce/SendReceive),
6种初始值模式(递增/全0/全1/S8递增/DevID/Offset),
自动内存分配/初始化/结果验证/打印。

---

## Comprehensive Test Anti-Patterns & Conventions

### Conventions (测试编写规范)

TST-ID-1: 测试框架选型
- Google Test + MockCPP (非gmock)
- 适用范围: 项目级
- 强制程度: 强制
- 出处: test/ut/CMakeLists.txt, cmake/third_party/gtest.cmake

TST-ID-2: Fixture模式
- 100%使用TEST_F (非TEST/TEST_P)
- TearDown必须调用GlobalMockObject::verify()
- 适用范围: 项目级
- 强制程度: 强制
- 出处: 223个文件使用TEST_F, 190个文件调用verify

TST-ID-3: 命名规范
- 测试方法: Ut_<Component>_When_<Scenario>_Expect_<Result>
- 测试类: ClassName_UT 或 ClassNameTest
- 文件名: ut_<component>_[API_]test.cc 或 ut_<component>_utest.cc
- 适用范围: 项目级
- 强制程度: 推荐 (部分文件简短命名)

TST-ID-4: 断言选择
- EXPECT_*为主 (允许继续执行)
- ASSERT_*仅用于必须成功的前置条件
- 适用范围: 项目级
- 强制程度: 强制

TST-ID-5: 对象实例化
- new(nothrow)创建被测对象
- unique_ptr管理生命周期
- 适用范围: 项目级
- 强制程度: 强制

TST-ID-6: 私有成员访问
- #define private public / #define protected public
- 在include被测头文件前定义, 之后#undef
- 适用范围: 项目级 (147/223文件)
- 强制程度: 常见做法 (非所有测试需要)

TST-ID-7: MOCKER使用
- C函数: MOCKER(funcName)
- C++成员: MOCKER_CPP(&Class::Method)
- 重载函数: MOCKER_CPP(&Class::Method, RetType(Class::*)(Args...))
- 适用范围: 项目级
- 强制程度: 强制

TST-ID-8: 环境变量管理
- ST测试使用setenv()配置算法行为
- TearDown中必须调用ClearHcclEnv()清理
- 适用范围: ST测试
- 强制程度: 强制

TST-ID-9: ST Checker验证
- 使用Checker框架做算法语义验证
- 通过RankTable_For_LLT生成拓扑
- 参数化遍历(serverNum x devPerServer)
- 适用范围: ST测试
- 强制程度: 强制

TST-ID-10: 示例代码风格
- 简洁的ACLCHECK/HCCLCHECK错误检查宏
- 标准流程: Init→SetDevice→GetRootInfo→Bcast→CommInit→Op→Sync→Destroy
- 中文注释说明每步用途
- 适用范围: examples/
- 强制程度: 推荐

### Anti-Patterns in Test Code

AP-TST-1: #define private public
- 范围: 147个文件, 项目级
- 问题: 测试与实现高度耦合, 重构时大量编译失败
- 正确做法: 友元测试类或依赖注入

AP-TST-2: 测试代码大量重复
- 范围: ut_aicpu_dispatcher_utest.cc, 多个测试重复Stream初始化~8行
- 问题: 维护成本高, DRY违反
- 正确做法: 提取为fixture辅助方法CreateStream()

AP-TST-3: 禁用的测试代码
- 范围: ut_aicpu_dispatcher_utest.cc:83-128 (#if 0)
- 问题: 死代码积累
- 正确做法: 删除或修复后启用

AP-TST-4: 测试中使用生产日志
- 范围: 少量文件中HCCL_ERROR出现在测试代码中
- 问题: 混淆测试输出和生产日志
- 正确做法: 使用gtest的SCOPED_TRACE或std::cout

AP-TST-5: 测试中裸new/delete
- 范围: ut_aicpu_dispatcher_utest.cc:218,540
- 问题: 异常时内存泄漏
- 正确做法: DeviceMem::alloc或unique_ptr

AP-TST-6: 测试类名不统一
- 范围: 项目级, _UT后缀 vs Test后缀
- 问题: 不一致降低可读性
- 正确做法: 统一使用一种后缀

AP-TST-7: legacy测试代码未清理
- 范围: test/legacy/ 496个文件
- 问题: 维护两套测试, 旧测试可能失效
- 正确做法: 迁移到新框架后删除

### Feature Summary

| Dimension | Value |
|-----------|-------|
| UT files | 273 (.cc/.cpp) |
| UT targets | 29 independent executables |
| ST files | 164 |
| Example files | 7 |
| Legacy test files | 496 |
| Test framework | Google Test v1.14.0 + MockCPP v2.7 |
| TEST_F usage | 223 files (near 100%) |
| TEST_P usage | 8 times / 3 files |
| MOCKER() | 186 files |
| MOCKER_CPP() | 113 files |
| EXPECT_EQ | 210 files |
| ASSERT_EQ | 15 times / 5 files |
| GlobalMockObject::verify | 190 files |
| #define private public | 147 files (66%) |
| Stub source files | 33 in hccl_llt |
| Include paths in UT | 275+ |
| Third-party deps | 3 (gtest/mockcpp/json) |
| Coverage | gcov-based (--coverage flag) |
| ASAN support | Optional (ENABLE_ASAN) |
