# CANN项目C++约束与偏好

本文件记录CANN (hccl/hcomm)项目特有的C++约束，只包含Claude不知道的项目特有信息。
实现时需从hccl/hcomm仓库的实际代码中验证和补充。

---

## 错误处理

- CHK_RET(expr): 检查返回值，失败时记录日志并返回错误码。最常用的错误处理宏。
- CHK_PTR_NULL(ptr): 空指针检查宏
- CHK_SMART_PTR_NULL(ptr): 智能指针空检查
- CHK_PRT_RET(cond, log, ret): 条件检查 + 日志 + 返回
- 禁止使用C++ exception（编译时关闭-fno-exceptions）
- 错误码通过HcclResult枚举返回
- 每个CHK_RET提前return的路径必须确保资源已释放

## 内存管理

- NEW_NOTHROW: 替代标准new，失败返回nullptr而非抛异常
- 智能指针优先，禁止裸new/delete
- 大buffer使用workspace分配（从运行时申请的连续内存池）
- 注意: hcomm-dev中存在资源释放顺序问题（析构先销毁AlgResource导致transport link悬垂），需检查组件间资源依赖

## 日志

- HCCL_DEBUG/INFO/WARNING/ERROR 宏
- 日志tag必须与当前类名/模块名匹配
- 性能敏感路径避免日志（数据面/kernel侧）
- 格式串参数类型必须匹配（%s对应字符串，%u对应unsigned，%llu对应uint64_t）
- 可恢复场景不用ERROR级别，用WARNING
- 日志变量打印当前值，非累积值

## 命名规范

- 类名: PascalCase (如 `TopoMatchMultiLevel`)
- 成员变量: camelCase_ (尾部下划线，如 `rankSizeLevel0_`)
- 函数: PascalCase (如 `MatchTopo`, `CalcGcdGroupSize`)
- 常量: ALL_CAPS 或 k前缀PascalCase (如 `MAX_RETRY_COUNT` 或 `kMaxRetryCount`)
- 枚举值: ALL_CAPS (如 `HCCL_REDUCE_SUM`)
- 文件名: snake_case (如 `topo_match_multilevel.h`)
- 命名空间: snake_case

## C++特性使用偏好

- 优先使用auto类型推导（但不滥用，复杂类型保留显式声明）
- 优先使用range-based for
- 优先使用智能指针(unique_ptr > shared_ptr)
- 字符串使用std::string，性能敏感路径可用string_view(如果C++17可用)
- 禁用RTTI(-fno-rtti)和exception(-fno-exceptions)
- 内置类型成员必须显式初始化（in-class initializer或构造函数初始化列表）

## 代码组织

- 头文件include顺序: 自身头文件 → 项目头文件 → 第三方 → 标准库
- 头文件保护: #ifndef guard格式（非#pragma once）
  格式: `#ifndef HCCL_MODULE_FILENAME_H` / `#define ...` / `#endif`
- 一个类一个文件（大型类除外）
- .h声明 + .cc实现，不在头文件中放实现（模板除外）

## 构建配置

新增.cpp/.cc文件时必须同步更新:
- CMakeLists.txt（添加源文件）
- 检查所有编译目标（CCL_KERNEL_AICPU、HCCD等）是否都需要包含
- 条件编译块的开闭路径保持对称

新增外部API依赖时:
- 检查该API在所有编译目标中都可用
- 公共头文件仅使用标准C/C++类型（不引入平台特有类型到ABI边界）

## 并发安全

- static/全局变量多线程访问需thread_local或mutex保护
- "先数据后标志位"的模式必须插入内存屏障
- 对象析构路径的裸指针delete需检查并发
- 持锁检查结果不在解锁后使用（TOCTOU防御）
- 资源释放后立即置空

## 枚举与类型

- 新增枚举值后必须全局搜索所有switch/if-else/map/数组，确保同步更新
- dataType枚举值禁止直接参与算术运算（应通过查表获取字节数）
- 函数返回类型与内部计算类型一致（避免隐式截断）
- 禁止map::at()处理外部输入（at()在key不存在时抛异常，但项目禁用exception）

## 平台适配

- 910_93/910_95等不同芯片可能有不同的行为路径
- 硬件协议乘数因子统一封装为宏/函数（避免散布在代码各处的魔数乘法）
- 同组宏常量用static_assert检查无重复值
