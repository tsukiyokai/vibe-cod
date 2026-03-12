# hccl-dev 缺陷热点文件分析

## 热点文件排名

| 文件 | 缺陷触及次数 | 缺陷commit |
|------|-------------|-----------|
| src/ops/scatter/scatter_op.cc | 4 | 2d8d548a, 13b6032d, 7347fee3, 19d20206 |
| test/.../sim_communicator.cc | 2 | c11a5289, 8222fcf8 |
| build.sh | 2 | 11b7211a, 8222fcf8 |

## 热点1：src/ops/scatter/scatter_op.cc（813行）

4/10的缺陷集中于此文件。当前代码中仍存在的结构性风险：

### 风险1：错误日志引用错误变量（当前bug）

第356行:
```cpp
aclError aclRet = aclrtLaunchKernelWithConfig(funcHandle, numBlocks, param.stream, &cfg, argsHandle, nullptr);
CHK_PRT_RET(aclRet != ACL_SUCCESS,
    HCCL_ERROR("[LoadCustomKernel][aclrtLaunchKernelWithConfig]errNo[0x%016llx] launch kernel failed", ret),
    HCCL_E_OPEN_FILE_FAILURE);
```

错误信息打印的是`ret`（第328行aclrtKernelArgsInit的返回值），应该是`aclRet`。这意味着当LaunchKernel失败时，日志中记录的错误码是错误的，会误导调试。

### 风险2：全局状态不安全

第275行:
```cpp
aclrtNotify g_notifies[AICPU_CONTROL_NOTIFY_NUM];
```

全局notify数组在多线程/多算子并发调用时存在竞争风险。如果多个scatter操作同时执行，g_notifies会被覆盖。

### 风险3：函数内宏定义

第715行:
```cpp
#define ACL_NOTIFY_DEFAULT          0x00000000U
```

在函数体内定义宏，宏的作用域不受函数限制，可能导致命名冲突。应改为constexpr常量。

### 风险4：拼写错误（"faled"）

第307、610、623行均出现`"faled"`拼写错误（应为`"failed"`），说明这些代码段是复制粘贴产生，未经仔细审查。

### 风险5：CommEngine枚举命名变迁的残留不一致

文件中混用了多代API命名：
- 第250行使用`COMM_ENGINE_CPU_TS`
- 第376行使用`COMM_ENGINE_CPU_TS`
- 但历史上同一语义曾用`COMM_ENGINE_HOSTCPU_TS`

枚举经历了多次重命名（AICPU→AICPU_TS, CommXxx→HcclXxx），每次重命名都是缺陷源。

### 风险6：AICPU launch代码块过长且内联

第301-362行的AICPU kernel launch逻辑（~60行）直接内联在ExecOp函数中，混合了kernel参数准备、notify管理、错误处理，难以独立审查和测试。

## 热点2：test/.../sim_communicator.cc

2次缺陷均因主代码API变更（结构体字段增删）未同步到测试代码。测试代码作为API的消费者，是API变更传播不完整的典型受害者。

## 热点3：build.sh

2次缺陷：一次是复制粘贴引入的变量名错误，另一次是同一错误的修复前提。构建脚本中的函数复制是高风险操作。
