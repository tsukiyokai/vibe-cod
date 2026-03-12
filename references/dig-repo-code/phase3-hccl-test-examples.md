# Phase 3.3: hccl Test & Example Code

## Module Overview

hccl测试体系与hcomm截然不同: UT几乎为空(仅环境清理工具), ST是真正的测试核心(仿真验证框架),
examples提供完整的API使用示例覆盖P2P/集合通信/AI框架/自定义算子四个层次。

```
test/
  ut/                   1个.cc文件(环境清理, 非真正测试)
  st/algorithm/
    testcase/           17个testcase文件, 222个TEST_F + 1个TEST_P
    utils/src/
      sim_world/        14文件 -- 多NPU仿真世界
      hccl_proxy/       24文件 -- 通信域/线程/通道模拟
      hccl_verifier/    40文件 -- DAG成图+内存冲突+语义校验
      common/           15文件 -- 异常体系+工具
      hccl_depends_stub/ 1文件 -- 空stub
    utils/ut/           2文件 -- 验证器自身的UT
examples/
  01_point_to_point/    2个示例(send_recv, batch_send_recv_ring)
  02_collectives/       9个示例(覆盖全部集合通信操作)
  03_ai_framework/      2个示例(PyTorch + TensorFlow)
  04_custom_ops_p2p/    1个复杂示例(自定义P2P算子)
```

## 3.3.1 Unit Tests (test/ut/)

### UT实质: 环境清理工具, 非真正测试

test/ut/仅包含1个源文件: `common/prepare_ut_env/main.cc`。
这不是单元测试, 而是CI管线中的环境准备步骤:

```cpp
// main.cc -- 唯一的"测试"用例
TEST_F(UtPrepareEnv, hccl_prepare_ut_env)
{
    system("rm -rf /dev/shm/hccl*");    // 删除共享内存文件
    sleep(3);                            // 等待清理完成
}
```

- SetUp/TearDown全部为空(注释掉)
- 被注释掉的gtest filter `HcomKernelInfoTest.st_LoadTask_comm` 暴露代码来自hcomm复制
- `--whole-archive`/`--no-whole-archive`之间无库名(模板残留)

### 构建配置

```
CMakeLists.txt层次:
  test/CMakeLists.txt           -- ENABLE_UT/ENABLE_ST独立开关
  test/ut/CMakeLists.txt        -- 仅add_subdirectory(common/prepare_ut_env)
  test/ut/common/.../CMake...   -- 单一目标hccl_ut_prepare_env
```

编译选项: `-std=c++14 -fno-access-control -O0 -g --coverage`
可选ASAN: `-fsanitize=address` (ENABLE_ASAN), `ASAN_OPTIONS=detect_leaks=0`
LLT执行: `run_llt_test`宏(POST_BUILD自动执行, SIGKILL超时, XML报告)

### 与hcomm UT对比

| 维度 | hcomm | hccl |
|------|-------|------|
| UT源文件 | 273个.cc | 1个.cc(环境清理) |
| 测试目标 | 29个可执行文件 | 1个 |
| Mock框架 | MockCPP v2.7 (MOCKER 186文件) | 无 |
| 私有访问 | `#define private public` (66%) | `-fno-access-control`(flag准备但未使用) |
| 测试粒度 | 单元级(mock单个类/函数) | 零UT; ST为系统级仿真 |
| 代码规模 | 33个stub + 273个测试 | 0个stub + 0个真正测试 |

差异根本原因:
1. hccl是上层编排层, 核心逻辑是多rank拓扑编排, 单元测试价值有限
2. hccl选择"重ST轻UT"策略 -- 投资仿真框架替代单元测试
3. 开源版本可能裁剪了内部UT(模板残留和注释可佐证)


## 3.3.2 System Tests (test/st/)

### ST架构: 仿真+验证流水线

```
                     +-----------+
                     | Testcases |     222个TEST_F + 1个TEST_P
                     +-----+-----+
                           |
               +-----------+-----------+
               |                       |
        +------v------+        +------v------+
        |  SimWorld   |        | SimTaskQueue|
        | (多NPU仿真)  |        | (任务记录)   |
        +------+------+        +------+------+
               |                       |
        +------v------+        +------v------+
        | HCCL Proxy  |        |  Checker    |
        | (Stub层)     |        | (验证流水线)  |
        +-------------+        +------+------+
                                       |
                          +------------+------------+
                          |            |            |
                   +------v--+  +-----v-----+  +--v--------+
                   |SingleTask|  |MemConflict|  |Semantics  |
                   |  Check   |  |  Check    |  |  Check    |
                   +---------+  +-----------+  +-----------+
```

### Testcase模式

Fixture结构高度统一:
```cpp
class ST_ALL_REDUCE_TEST : public ::testing::Test {
protected:
    void SetUp() override { ResetAlgEnvConfigInitState(); }
    void TearDown() override { unsetenv("HCCL_OP_EXPANSION_MODE"); }
};
```

- 命名: `ST_` + 操作名大写 + `_TEST`
- SetUp统一调ResetAlgEnvConfigInitState()
- TearDown清理环境变量(但各文件不一致)
- Fixture无成员变量(stateless)

测试执行流程(每个TEST_F内部):
```
1. SimWorld::Global()->Init(topoMeta, DEV_TYPE_910_95)
2. setenv("HCCL_OP_EXPANSION_MODE", "AI_CPU", 1)
3. 多线程模拟各rank:
   aclrtSetDevice → aclrtCreateStream → HcclCommInitClusterInfo
   → aclrtMalloc(BUFFER_INPUT_MARK/OUTPUT_MARK) → HcclXxx → HcclCommDestroy
4. thread.join() 等待所有线程
5. Check*(SimTaskQueue::Global()->GetAllRankTaskQueues(), ...)
6. SimWorld::Global()->Deinit()
```

拓扑通过`TopoMeta`三维vector表达: SuperPod → Server → Device
```cpp
TopoMeta{{{0, 1, 2, 3}}};           // 1超节点, 1server, 4device (Mesh1D)
TopoMeta{{{0}, {0}, {0}, {0}}};     // 1超节点, 4server, 各1device (NHR)
TopoMeta{{{0, 1}, {0, 1}}};         // 1超节点, 2server, 各2device
TopoMeta{{{0}}, {{1}}};             // 2超节点 (跨SuperPod)
```

断言模式统一: `EXPECT_TRUE(Check*(...) == HCCL_SUCCESS)`
无EXPECT_EQ/ASSERT_*, 只通过Check函数返回值判断。

### Grep量化

| 指标 | 数值 |
|------|------|
| TEST_F | 222次/17文件 |
| TEST_P | 1次/1文件(reduce_testcase.cc) |
| EXPECT_TRUE | 22次/17文件(每个testcase至少1次) |
| CHK_RET | 144次/28文件(验证框架内部) |
| MAKE_ENUM | 6次/4文件 |
| SimWorld::Global | 80次/23文件 |

### Testcase覆盖矩阵

| 算子 | TEST_F数 | 拓扑覆盖 | 数据类型 | 设备类型 |
|------|----------|----------|----------|----------|
| all_reduce | 19 | Mesh1D/NHR/混合 | INT32/FP16/BFP16等 | 910_95 |
| all_reduce_parallel | 9 | Mesh1D/混合 | INT32/FP16 | 910_95 |
| all_gather_aicpu | 18 | Mesh1D/NHR/混合 | INT32/FP16/INT8等 | 910_95 |
| reduce_scatter_aicpu | 24 | Mesh1D/NHR/混合 | INT32/FP16/BFP16等 | 910_95 |
| alltoallv | 30 | Mesh1D/NHR/混合 | 12种全覆盖 | 910_95 |
| alltoallvc | 30 | Mesh1D/NHR/混合 | 12种全覆盖 | 910_95 |
| broadcast | 23 | Mesh1D/NHR/混合 | INT32/FP16/FP32等 | 910_95 |
| batch_send_recv | 19 | Mesh1D/NHR/混合 | INT32/FP16/INT8 | 910_95 |
| scatter | 18 | Mesh1D/NHR/混合/非连续ID | INT32/FP16/BFP16等 | 910_95 |
| scatter_a3 | 1 | Mesh1D | INT32 | 910B |
| send_recv | 17 | Mesh1D/NHR/跨SuperPod | INT32/FP16 | 910_95 |
| alltoall | 11 | Mesh1D/NHR/混合 | 12种全覆盖 | 910_95 |
| reduce | 60+(参数化) | Mesh1D/2D(x*y)/NHR | INT32/FP16/FP32/BFP16 | 910_95 |
| all_gather_v | 1 | Mesh1D | INT32 | 910_95 |
| reduce_scatter_v | 1 | Mesh1D | INT32 | 910_95 |
| hccl_mc2 | 1 | Mesh1D | INT32 | 910B |


### 仿真框架 (sim_world + hccl_proxy)

SimWorld单例管理仿真世界:
- 数据结构: `map<PodId, map<SerId, map<PhyId, SimNpu>>> simNpus_`
- SimNpu模拟每个NPU: 三块内存(INPUT/OUTPUT/CCL各200MB虚拟地址段), 1984条Stream, 8192个Notify
- 地址仿真而非数据仿真: 不搬运真实数据, 只追踪数据来源

SimTaskQueue全局单例:
- 被stub替换的runtime API将操作记录为TaskStub(而非真正执行)
- TaskStub继承体系(11种操作):
  本地: LOCAL_COPY, LOCAL_REDUCE, LOCAL_BATCH_REDUCE
  流间: LOCAL_POST_TO, LOCAL_WAIT_FROM
  跨rank: POST, WAIT, READ, READ_REDUCE, WRITE, WRITE_REDUCE

HCCL Proxy层:
- SimCommunicator替换HcclCommunicator
- SimThreadMgr替换线程管理(不创建真实线程)
- SimChannelMgr替换通信链路(polling+sleep 100ms等待, 20s超时)
- TopoModel替换拓扑查询(硬编码910B/C/D三种设备L0-L2链路)

### 验证流水线 (Checker 7步)

```
Step 1: SingleTaskCheck -- 从流不变量(WAIT_FROM开头, POST_TO结尾)
Step 2: GenGraph -- DAG构建(单rank内+跨rank)
  - LOCAL_POST_TO/WAIT_FROM三元组匹配(topicId+postQid+waitQid)
  - POST/WAIT跨rank匹配(rankId对端+LinkType+topicId+notifyType)
  - 死锁检测(无法配对时报告)
Step 3: CheckTaskMem -- 每个task的内存合法性(不越界/不自重叠)
Step 4: CopyTaskGraph -- 深拷贝DAG
Step 5: GraphRevamp -- 双边语义增强(为READ/WRITE添加跨rank依赖)
  - SDMA: 前向搜索WAIT屏障, 后向搜索POST屏障
  - RDMA: 单边写特殊处理
Step 6: CheckRankMem -- 跨流内存冲突检测
  - 碎片队列: POST_TO/WAIT_FROM为边界切割流
  - 并发矩阵: 追踪依赖排除不可能并行的片段对
  - 冲突判定: 并行片段同地址重叠且至少一方WRITE
Step 7: OpSemantics -- 最高层语义校验
  - BFS拓扑序"模拟执行", 追踪BufferSemantic(数据来源)
  - SliceOpPair: OVERRIDE(数据搬运) / REDUCE(归约操作)
  - 委托operation-specific checker验证最终OUTPUT
```

语义检查器覆盖:
- CheckAllReduce: 每个rank的OUTPUT = 所有rank的INPUT的reduce结果
- CheckAllGather: 每个rank的OUTPUT = 所有rank的INPUT拼接
- CheckReduceScatter: 每个rank的OUTPUT = reduce结果的对应分片
- CheckScatter: 各rank的OUTPUT = root的INPUT的对应分片
- CheckBroadcast: 所有rank的OUTPUT = root的INPUT
- CheckSend/Recv: dst的OUTPUT = src的INPUT
- CheckBatchSendRecv: 多对多P2P语义
- CheckAll2All/V/VC: 全交换语义
- CheckReduce: root的OUTPUT = 所有rank的INPUT的reduce结果

### 异常体系

测试框架有独立的异常层次(与业务代码分离):
```
std::exception
  └── HcclException (ExceptionType枚举 + 消息 + backtrace)
      ├── InvalidParamsException → HCCL_E_PARA
      ├── NullPtrException → HCCL_E_PTR
      ├── InternalException → HCCL_E_INTERNAL
      ├── NotSupportException → HCCL_E_NOT_SUPPORT
      └── TimeoutException → HCCL_E_TIMEOUT
```

`THROW<T>(format, args...)`模板函数: HCCL_ERROR记录日志后throw。
模拟层倾向用异常, 验证层倾向用CHK_RET返回错误码。


## 3.3.3 Examples

### 目录结构

```
examples/
  01_point_to_point/
    01_send_recv/           -- P2P收发
    02_batch_send_recv_ring/-- 批量P2P环形
  02_collectives/
    01_allreduce/ ~ 09_scatter/ -- 9种集合通信
  03_ai_framework/
    01_pytorch/             -- PyTorch集成
    02_tensorflow/          -- TensorFlow集成
  04_custom_ops_p2p/        -- 自定义P2P算子(复杂示例)
  build.sh                  -- 顶层构建脚本
```

每个简单示例: 单个main.cc + Makefile + README.md

### 标准API使用模式

所有示例遵循统一的7步流程:
```cpp
// Step 1: 获取RootInfo(rank 0生成, 其他rank接收)
HcclRootInfo rootInfo;
if (rankId == 0) HcclGetRootInfo(&rootInfo);
MPI_Bcast(&rootInfo, sizeof(rootInfo), MPI_BYTE, 0, MPI_COMM_WORLD);

// Step 2: 设备绑定
aclInit(nullptr);
aclrtSetDevice(deviceId);
aclrtCreateStream(&stream);

// Step 3: 通信域创建
HcclComm comm;
HcclCommInitRootInfo(rankSize, &rootInfo, rankId, &comm);

// Step 4: 内存分配
aclrtMalloc(&sendBuf, bufferSize, ACL_MEM_MALLOC_HUGE_FIRST);
aclrtMalloc(&recvBuf, bufferSize, ACL_MEM_MALLOC_HUGE_FIRST);
aclrtMemcpy(sendBuf, ..., ACL_MEMCPY_HOST_TO_DEVICE);

// Step 5: 通信操作
HcclAllReduce(sendBuf, recvBuf, count, dataType, op, comm, stream);

// Step 6: 同步+验证
aclrtSynchronizeStream(stream);
aclrtMemcpy(hostBuf, ..., ACL_MEMCPY_DEVICE_TO_HOST);

// Step 7: 清理(逆序)
aclrtFree(sendBuf); aclrtFree(recvBuf);
HcclCommDestroy(comm);
aclrtDestroyStream(stream);
aclrtResetDevice(deviceId);
aclFinalize();
```

通信域初始化: 全部使用RootInfo模式(rank 0生成rootInfo, MPI广播到其他rank)。
无RankTable模式(仅ranktable.json配置文件供ST使用)。

### P2P示例特点

send_recv: 最简P2P, rank0发rank1收
batch_send_recv_ring: N个rank环形排列, 每个rank同时向下一个发送和从上一个接收

### AI框架集成

PyTorch:
```python
import torch
import torch_npu
torch.distributed.init_process_group(backend='hccl')
# AllReduce自动走HCCL后端
```

TensorFlow:
```python
import tensorflow as tf
import npu_bridge
# 使用hccl.ops.allreduce / tf.distribute.Strategy
```

### 自定义算子示例 (04_custom_ops_p2p)

展示HCCL底层P2P通信三要素: 中转内存 + 单边RDMA读 + Notify同步

架构:
```
op_host/      -- Host侧: 申请中转内存, launch kernel
  send.cc     -- HcclSend实现: AllocMem→LaunchKernel→Notify对端
  recv.cc     -- HcclRecv实现: Wait通知→Read数据→DeallocMem
  launch_kernel.cc -- aclrtLaunchKernel封装
  load_kernel.cc   -- aclOpenSo + aclFindSymbol加载设备算子
op_kernel_aicpu/ -- Device侧: AICPU执行体
  exec_op.cc  -- 反序列化参数→调用HcclSend/Recv
  aicpu_kernel.cc -- 入口点(注册到AICPU框架)
testcase/     -- 测试(依赖MPI环境)
```

关键模式:
- allocType参数: HOST(CPU内存, 跨节点RoCE) vs DEVICE(HBM, 节点内)
- 中转buffer: 接收方分配→交换地址→发送方RDMA Read→释放
- Notify同步: PostNotify(发送方→接收方) + WaitNotify(接收方阻塞)

### 示例中的反模式

1. Scatter示例内存分配bug: `aclrtMalloc(&recvBuf, recvCount, ...)` 应为recvSize(字节数而非元素数)
2. 大部分示例错误处理不一致: 有的用if检查, 有的用assert, 有的不检查
3. 注释和变量命名不统一: 有的用中文注释, 有的用英文
4. deviceId硬编码`rankId % 8`(假设8卡/server)
5. 自定义算子示例的`new/delete`手动内存管理(应用unique_ptr)


## 3.3.4 综合分析

### 跨子模块惯用法

HC-TST-ID-1: SimWorld单例仿真模式
所有ST测试通过SimWorld::Global()全局单例管理仿真世界(80次/23文件)。
Init(topoMeta, devType)初始化 → 多线程执行 → Deinit()清理。
适用范围: hccl ST

HC-TST-ID-2: TaskStub记录而非执行
被stub替换的API将操作记录为TaskStub(11种类型)而非真正执行。
验证基于task graph的正确性而非数据的正确性。
适用范围: hccl ST

HC-TST-ID-3: Check*函数统一验证入口
每个集合操作对应一个Check*函数, testcase只需调用Check*(taskQueues, params)。
验证结果通过EXPECT_TRUE(res == HCCL_SUCCESS)断言。
适用范围: hccl ST

HC-TST-ID-4: TopoMeta三维vector表达拓扑
`vector<vector<vector<PhyDeviceId>>>` = SuperPod → Server → Device。
testcase内联初始化列表直接构造(不依赖配置文件)。
适用范围: hccl ST

HC-TST-ID-5: ResetAlgEnvConfigInitState + unsetenv环境清理
所有fixture的SetUp统一重置全局算法配置状态, TearDown清理环境变量。
适用范围: hccl ST

HC-TST-ID-6: RootInfo模式通信域初始化
所有examples使用HcclGetRootInfo + MPI_Bcast + HcclCommInitRootInfo。
7步标准流程: ACL Init → SetDevice → CreateStream → CommInit → Op → Sync → Destroy。
适用范围: hccl examples

HC-TST-ID-7: MAKE_ENUM自描述枚举
仿真框架使用MAKE_ENUM生成enum wrapper(6次/4文件), 自带Describe()和operator<<。
与hcomm生产代码一致(项目级模式)。
适用范围: hccl ST, 项目级

### 架构模式

HC-TST-AP-1: 仿真替代真实硬件
完整的软件仿真层替代NPU硬件: SimNpu(虚拟内存/Stream/Notify) + SimCommunicator +
SimChannel + TopoModel。地址仿真而非数据仿真。
适用范围: hccl ST

HC-TST-AP-2: DAG成图+拓扑排序验证
将并行task流→DAG→BFS拓扑排序→模拟执行→语义校验。
包含死锁检测、内存冲突检测、单task合法性检查。
7步流水线分层捕获不同类别的bug。
适用范围: hccl ST

HC-TST-AP-3: BufferSemantic追踪
不搬运真实数据, 追踪"这块内存的数据来自哪里"的元信息。
SliceOpPair记录OVERRIDE(搬运)/REDUCE(归约)操作。
最终检查OUTPUT的语义是否符合操作的数学定义。
适用范围: hccl ST

HC-TST-AP-4: 碎片队列+并发矩阵
内存冲突检测的独特设计:
POST_TO/WAIT_FROM为边界切割流→碎片队列→依赖关系排除不可能并行对→
并行碎片同地址重叠+至少一方WRITE→报告冲突。
适用范围: hccl ST

HC-TST-AP-5: 双边语义增强(图改造)
READ/WRITE是跨rank操作, 原始DAG只记录本端。
图改造步骤为远端添加依赖边, 使内存冲突检测能发现跨rank竞争。
SDMA(前向WAIT+后向POST屏障) vs RDMA(单边写特殊处理)。
适用范围: hccl ST

### 反模式

HC-TST-AP-FW-1: rankSize计算逻辑重复5次
同一逻辑在不同testcase中有5个不同名函数: AnalyseRankSize / CalsRankSize /
CountElements / GetRankSize / 内联计算。应提取到公共工具函数。
文件: all_reduce_testcase.cc, all_gather_aicpu_testcase.cc, reduce_scatter_aicpu_testcase.cc,
scatter_testcase.cc, send_recv_testcase.cc, reduce_testcase.cc

HC-TST-AP-FW-2: DATATYPE_SIZE_TABLE重复定义3+次
各testcase各自定义数据类型大小表(constexpr数组/switch-case/直接传参), 名称各异。
check_utils.h中有SIZE_TABLE但仅alltoall系列使用。
文件: scatter_testcase.cc, all_gather_aicpu_testcase.cc, reduce_scatter_aicpu_testcase.cc, reduce_testcase.cc

HC-TST-AP-FW-3: ODR冲突
scatter_testcase.cc和scatter_testcase_a3.cc都定义了同名类`ST_SCATTER_TEST`,
编译到同一可执行文件中违反One Definition Rule。
文件: scatter_testcase.cc:L22, scatter_testcase_a3.cc

HC-TST-AP-FW-4: TearDown清理不完整
各文件TearDown清理的环境变量不一致: 有的清3个(OP_EXPANSION_MODE + INDEPENDENT_OP +
BUFFSIZE), 有的只清1个。Run函数中setenv的变量在TearDown中可能被遗漏。
文件: 大部分testcase

HC-TST-AP-FW-5: sizeof(dataType)疑似bug
all_gather_v_testcase.cc中`sizeof(dataType)` -- dataType是enum值不是类型,
sizeof取的是enum本身大小而非数据类型宽度。
文件: all_gather_v_testcase.cc

HC-TST-AP-FW-6: 拼写错误
`CalsRankSize`(应为CalcRankSize)。
文件: reduce_scatter_aicpu_testcase.cc

HC-TST-AP-FW-7: 验证框架vector::erase(begin())做BFS
Checker和TaskGraphGenerator中用`erase(begin())`模拟queue, O(n)复杂度。
应该用`std::queue`(O(1)出队)。
文件: checker.cc:L139, task_graph_generator.cc:L69,L263-264

HC-TST-AP-FW-8: TopoModel copy-paste错误
`Init910CLinkMap()`写入了`link910b_`而非`link910c_`, 910C的链路信息被写入910B的map。
文件: topo_model.cc:L414-453

HC-TST-AP-FW-9: TaskNodePtr裸指针typedef
`TaskNodePtr = TaskNode*`不是智能指针。设计原因: 避免大规模图递归析构时栈溢出。
有意为之的权衡, 但typedef名容易误导。
文件: task_def.h

HC-TST-AP-FW-10: SimNpu startAddr未使用
`SimNpu::InitMemLayOut()`中计算了startAddr但实际赋值用的是addr变量。
文件: sim_npu.cc:L39,44

HC-TST-AP-FW-11: testcase命名风格不统一
存在4种风格混用: 描述型(st_all_reduce_1shot_boundary_dataCount),
编号型(st_alltoall_0), 混合型(st_broadcast_a5_aicpu_1DTwoShot_one_four_test),
详细型(test_aicpu_scatter_mesh_1d_success_1x3_root0_int32_small_data)。

HC-TST-AP-FW-12: examples错误处理不一致
有的用if检查返回值, 有的用assert, 有的不检查。
scatter示例aclrtMalloc传入recvCount(元素数)而非recvSize(字节数)。

### 与hcomm测试体系对比

| 维度 | hcomm | hccl |
|------|-------|------|
| UT | 273文件, MockCPP, 白盒测试 | 1文件(非测试), 无mock |
| ST框架 | Checker(DAG+内存+语义) | 相同理念但独立实现 |
| ST testcase | 22个testcase_*.cc | 17个testcase文件 |
| 仿真深度 | - | SimWorld完整仿真(NPU/Stream/Notify) |
| 异常体系 | 生产代码共用 | 独立HcclException层次 |
| 设备覆盖 | 多种芯片 | 主要910_95(A5), 少量910B |
| 参数化测试 | 少量(TEST_P 8次) | 极少(TEST_P 1次) |
| mock策略 | MockCPP(C函数mock) | 编译期stub替换 |
| 验证方式 | Check*函数 | 相同(Check*函数) |
| MAKE_ENUM | 生产代码中使用 | 仿真框架中使用 |

共享模式(项目级):
- DAG成图+拓扑排序的验证方法论
- 碎片队列+并发矩阵的内存冲突检测
- BufferSemantic追踪而非数据搬运
- MAKE_ENUM自描述枚举
- CHK_RET错误处理宏
- Check*函数统一验证入口

差异:
- hccl有GraphRevamp双边语义增强(处理SDMA/RDMA跨rank内存引用)
- hccl的TopoModel更复杂(三种设备类型, L0-L2三层网络)
- hccl的SimChannelExchangeHandler使用多线程polling建链
- hccl完全放弃单元测试, hcomm两者兼备

### 值得注意的好实践

1. reduce_testcase.cc参数化测试: 唯一使用TEST_P, 通过::testing::Values组合60+参数组
   覆盖面最广且代码最紧凑, 是其他testcase应该学习的模式

2. send_recv_testcase.cc设计最优雅: 用std::map<RankId,RankId>描述收发映射,
   只为参与通信的rank创建线程(usedRankIds过滤)

3. batch_send_recv_testcase.cc测试状态重入性: MultiThreadExecOpMultiTimes测试
   同一通信域多次算子下发

4. Checker 7步流水线分层设计清晰: 每层捕获不同类别的bug

5. all_reduce_testcase.cc使用static_assert编译期校验常量

6. 自定义算子示例展示HCCL底层三要素: 中转内存 + 单边RDMA读 + Notify同步
