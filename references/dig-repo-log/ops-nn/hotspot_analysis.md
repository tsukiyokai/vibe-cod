# ops-nn 代码热点与结构性风险分析

## 热点文件统计

从380条缺陷提交的defect_analysis.md中提取被修复触及的文件频次（仅含具体路径）：

| 排名 | 文件 | 触及次数 | 模块 |
|------|------|---------|------|
| 1 | build.sh | 10 | 构建 |
| 2 | cmake/ut.cmake | 9 | 构建/测试 |
| 3 | cmake/gen_ops_info.cmake | 8 | 构建 |
| 4 | tests/ut/op_host/CMakeLists.txt | 5 | 测试 |
| 5 | scripts/kernel/binary_config/ascendc_config.json | 4 | 配置 |
| 5 | matmul/quant_batch_matmul_v4/.../aclnn_quant_matmul_v5.cpp | 4 | matmul/op_api |
| 5 | matmul/common/op_host/op_api/matmul_util.cpp | 4 | matmul/op_api |
| 8 | matmul/mat_mul_v3/op_kernel/mat_mul_deterministic_splitk_kernel.h | 3 | matmul/kernel |
| 8 | matmul/mat_mul_v3/op_host/op_tiling/matmul_v3_base_tiling.cpp | 3 | matmul/tiling |
| 8 | matmul/mat_mul_v3/op_host/op_api/aclnn_addmm.cpp | 3 | matmul/op_api |
| 8 | matmul/batch_mat_mul_v3/op_host/op_api/aclnn_addbmm.cpp | 3 | matmul/op_api |
| 8 | conv/convolution_backward/op_api/aclnn_convolution_backward.cpp | 3 | conv/op_api |
| 8 | CMakeLists.txt | 3 | 构建 |
| 8 | cmake/variables.cmake | 3 | 构建 |
| 8 | cmake/func.cmake | 3 | 构建 |
| 8 | scripts/kernel/binary_script/build_binary_single_op_gen_task.sh | 3 | 构建脚本 |

热点分布特征：
- 构建系统(build.sh + cmake + 脚本)占据Top3，且总触及次数达30+，是ops-nn最大的缺陷聚集区
- matmul模块有6个文件上榜，覆盖tiling/kernel/op_api三层
- conv仅1个文件上榜但是最大的单文件(2500+行)

---

## 一、build.sh 结构性风险（10次触及）

### 实际Bug

1. `]`前缺少空格（行1278）:
```bash
[ -f ${EAGER_LIBRARY_OPP_PATH}/libascendcl.so]
```
右括号与文件名粘连，`test`命令寻找名为`libascendcl.so]`的文件，条件永远为false。

2. 数组备份/恢复错误（行1060-1065）:
```bash
local TMP_BUILD_LIBS=${BUILD_LIBS}     # 只取数组第一个元素
# ... build_binary_one执行后 ...
BUILD_LIBS=${TMP_BUILD_LIBS}           # 恢复时变成字符串
```
`BUILD_LIBS`是数组，`${BUILD_LIBS}`只取首元素。多target构建时BUILD_LIBS数据丢失。

3. `--mssanitizer`选项设为FALSE（行796）:
```bash
mssanitizer) ENABLE_MSSANITIZER=FALSE ;;
```
启用选项却赋值FALSE，该flag无效化。

4. sed空操作（行779）:
```bash
COMPUTE_UNIT=$(echo "$COMPUTE_UNIT" | sed 's/ascend950/ascend950/g')
```
替换前后相同，是遗留的死代码。

### 环境安全风险

5. ASCEND_HOME_PATH/ASCEND_OPP_PATH无检查（行129-139）: 未设置时所有路径变成`/include`、`/lib64`等根目录。

6. `set +u`从未恢复（行35）: 整个脚本运行期间未定义变量静默为空，拼写错误的变量名不会报错。

7. LD_LIBRARY_PATH追加时可能以冒号开头（行1069等）: `$LD_LIBRARY_PATH:`前缀为空时包含当前目录。

8. source使用相对路径（行33）: `source "./install_deps.sh"`在非脚本目录执行时加载错误文件。

### 其他风险

9. `$(rm -f asan_test)`命令替换误用（行943）
10. `gen_op()`缺少python未找到时的else分支（行1363-1385）
11. `dirname $0`未加引号（行118）
12. `rm -rf $BUILD_OUT_PATH`未加引号（行600）
13. CMAKE_ARGS字符串拼接（非数组），参数含空格时错误拆分
14. main通过管道传给gawk，exit在subshell中执行（行1531）
15. CORE_NUMS赋值后从未使用（行125，死代码）

---

## 二、CMake文件结构性风险（cmake/ut.cmake 9次 + gen_ops_info.cmake 8次 + 其他）

### 实际Bug

1. `compile_from_config`使用循环后遗留的`op_name`（gen_ops_info.cmake:654）:
```cmake
compile_from_config(TARGET ascendc_bin_${compute_unit}_${op_name} ...)
```
在foreach循环结束后调用，`${op_name}`仅为最后一次迭代的值。

2. `CUSTOM_TILING_DATA_KEYS`先用后赋值（ut.cmake:571 vs 587）:
```cmake
set(gen_cmd "... ${CUSTOM_TILING_DATA_KEYS}")  # 行571，先用
# ...
set(CUSTOM_TILING_DATA_KEYS "")                 # 行587，后赋值
```

3. `target_dir`空值判断缺引号（ut.cmake:354）:
```cmake
if(${target_dir} STREQUAL "")    # 空时展开为 if( STREQUAL "")
```

4. `OPS_TEST_DIR`未定义即使用（ut.cmake:51）

### 配置/依赖风险

5. `ARCH`变量在Unknown架构时未设置（variables.cmake:27-33）: else分支只打印WARNING不设默认值。

6. `MODULE_EXT`未在parse_arguments中声明（func.cmake:584）: 分支永远不执行（dead code或功能缺陷）。

7. `kernel_src_copy`接受了未声明的`OP_LIST`参数（gen_ops_info.cmake:14-17）: 参数被静默忽略。

8. `atvoss_src_copy`target无条件重建（gen_ops_info.cmake:50-60）: 多次调用时target重复定义。

9. gen_ops_info.cmake被include三次无guard（CMakeLists.txt:144/158/168）

10. `map_compute_unit`与`get_target_dir`映射表硬编码且仅覆盖5/12个芯片版本（func.cmake:695/710）

11. `_GLIBCXX_USE_CXX11_ABI`在UT模块间不一致: AICPU=1 vs 其他=0，可能导致ABI兼容性问题

12. `add_version_info_targets`错误消息使用了不存在的`${pkg_name}`（func.cmake:899-907）

---

## 三、matmul模块结构性风险（6个文件上榜）

### 空指针/先用后查

1. self先Contiguous后才检查null（aclnn_addmm.cpp:298-299）:
```cpp
auto selfContiguous = l0op::Contiguous(addmmTensor.self, uniqueExecutor);
if (addmmTensor.self != nullptr && ...)  // 检查太晚
```

2. InplaceAddmm未检查selfRef为null（aclnn_addmm.cpp:729）

3. InplaceAddbmm未检查selfRef为null（aclnn_addbmm.cpp:547）

4. AllocTensor结果未检查null即调用IsEmpty()（matmul_util.cpp:214-225）

### CHECK_RET检查错误变量

5. 赋值给reformatedX1但检查x1（aclnn_quant_matmul_v5.cpp:907-908）:
```cpp
reformatedX1 = l0op::TransData(reformatedX1, Format::FORMAT_FRACTAL_NZ, 0, executor);
CHECK_RET(x1 != nullptr, ACLNN_ERR_INNER_NULLPTR);  // 应检查reformatedX1
```

### 除零/溢出

6. baseM/baseK/dtypeSize连续除法可能除零（matmul_v3_base_tiling.cpp:605-606）

7. nCoreTail=0时AlignTo256B(0)=0导致除零（mat_mul_deterministic_splitk_kernel.h:80-81）

8. baseM*baseN乘法可能溢出（matmul_v3_base_tiling.cpp:1366-1367）

9. uint64->uint32截断无保护（matmul_v3_base_tiling.cpp:909-912）

### Copy-paste / 逻辑错误

10. rowBlockNum和colBlockNum公式相同（mat_mul_deterministic_splitk_kernel.h:404-405）:
```cpp
uint64_t rowBlockNum = alignedN / 16;    // 应为alignedM / 16?
uint64_t colBlockNum = alignedN / 16;
```
疑似copy-paste bug，rowBlockNum应基于alignedM。

11. 注释与条件判断矛盾（aclnn_addbmm.cpp:427-432）: 条件判断`alpha == 1.0`但注释写`alpha == 0`。

12. SELF_MIN=0导致维度检查失效（aclnn_addbmm.cpp:147-148）: self为0维/1维时GetDim(1)越界。

13. batch2用batch1DimNum索引可能越界（matmul_util.cpp:1663）

### 类型安全

14. const_cast修改入参dtype（aclnn_quant_matmul_v5.cpp:813-831）: INT64改为UINT64的语义差异在负值场景致命。

15. union type punning是C++ UB（matmul_util.cpp:2099-2107）

16. bf16与half sizeof相同走同一Cast路径（mat_mul_deterministic_splitk_kernel.h:100-102）: RoundMode可能不适用于bf16。

---

## 四、conv模块结构性风险（aclnn_convolution_backward.cpp 3次触及）

### 空指针/返回值忽略

1. Cast后先LOG再CHECK（行363-368）:
```cpp
l0ResultTensor = l0op::Cast(...);
OP_LOGD("...", l0ResultTensor->ToString()...);  // Cast返回null时崩溃
OP_CHECK(l0ResultTensor != nullptr, ...);        // 检查太晚
```

2. InputPreProcess返回值被忽略x3（行1014-1016）: 其他位置都用CHECK_RET包裹。

3. CalculateBiasGrad返回值被忽略（行1312）

### 其他

4. 栈数组append_dim未完全初始化（行510-514）: inputDim>=4时循环不执行，数组内容未定义。

5. 错误日志与实际函数不匹配（行2228/2361）: Conv3D的代码打印"Conv2dBackpropInput"。

6. promoteType可能未初始化（行2300-2324）: outputMask[0]和[1]同时为false时。

---

## 五、ascendc_config.json 结构性风险（4次触及）

### 完全重复条目

| 算子 | 重复行号 |
|------|----------|
| AdaptiveAvgPool3d | 行2 vs 行5 |
| DeformableOffsetsGrad | 行275 vs 行294 |
| LayerNormQuant | 行571/600/601 (三份) |
| GetPaddingOffset | 行572/604/622 (三份) |

### 同名但配置冲突的条目

| 算子 | 差异 |
|------|------|
| AdaptiveAvgPool3dGrad | compute_units含/不含ascend950 |
| AdaptiveMaxPool3d | compute_units含/不含ascend950 |
| STFT | compute_units含/不含ascend910_93 |
| Split/SplitV | impl_mode有/无 |
| DeformableOffsets | compile_options有/无 |

哪一条生效取决于JSON解析器实现——构建行为不确定。

### 无效配置

compile_options中引用`ascend910_95`（行283/363）: 不在compute_units列表中也非已知平台标识，疑似`ascend910_93`或`ascend950`的拼写错误。

---

## 六、tests/ut/op_host/CMakeLists.txt 结构性风险（5次触及）

1. ENABLE_ASAN/ENABLE_VALGRIND未定义时直接STREQUAL比较（行98/114）: 展开为空导致cmake报错。

2. ASAN LD_PRELOAD路径硬编码x86_64（行101-102）: 昇腾设备通常为aarch64，路径不存在。

3. ENABLE_UT_EXEC和ENABLE_VALGRIND可能同时为TRUE（行97-120）: 缺乏互斥检查，POST_BUILD执行两次。

4. SKIP_BUILD_RPATH=TRUE（行64-66）: 依赖环境变量LD_LIBRARY_PATH，未设置时运行时链接失败。

---

## 风险汇总与优先级

### P0 (确认的Bug)

| # | 位置 | 描述 |
|---|------|------|
| 1 | build.sh:1278 | `]`前缺空格，条件永远false |
| 2 | build.sh:1060-1065 | 数组备份/恢复错误 |
| 3 | gen_ops_info.cmake:654 | 循环后遗留变量作为target名 |
| 4 | ut.cmake:571/587 | CUSTOM_TILING_DATA_KEYS先用后赋值 |
| 5 | aclnn_addmm.cpp:298-299 | 空指针先用后查 |
| 6 | aclnn_quant_matmul_v5.cpp:907-908 | CHECK_RET检查错误变量 |
| 7 | mat_mul_deterministic_splitk_kernel.h:404-405 | rowBlockNum/colBlockNum公式相同(疑似copy-paste) |
| 8 | aclnn_convolution_backward.cpp:363-368 | Cast后先LOG再CHECK |

### P1 (高风险)

| # | 位置 | 描述 |
|---|------|------|
| 9 | build.sh:129-139 | ASCEND_HOME_PATH无检查 |
| 10 | build.sh:796 | --mssanitizer设为FALSE |
| 11 | matmul_v3_base_tiling.cpp:605-606 | 连续除法潜在除零 |
| 12 | mat_mul_deterministic_splitk_kernel.h:80-81 | nCoreTail=0除零 |
| 13 | aclnn_addbmm.cpp:547 | InplaceAddbmm未检查null |
| 14 | aclnn_addmm.cpp:729 | InplaceAddmm未检查null |
| 15 | ascendc_config.json多处 | 同名不同配置冲突 |
| 16 | aclnn_convolution_backward.cpp:1014-1016 | InputPreProcess返回值忽略x3 |

### P2 (中风险)

| # | 位置 | 描述 |
|---|------|------|
| 17 | ut.cmake:354 | target_dir空值判断缺引号 |
| 18 | matmul_v3_base_tiling.cpp:909-912 | uint64->uint32截断 |
| 19 | aclnn_addbmm.cpp:147-148/227-230 | SELF_MIN=0+GetDim越界 |
| 20 | matmul_util.cpp:1663 | batch索引用错DimNum |
| 21 | aclnn_quant_matmul_v5.cpp:813-831 | const_cast修改入参dtype |
| 22 | ascendc_config.json:283/363 | ascend910_95疑似拼写错误 |
| 23 | variables.cmake:27-33 | ARCH未设置默认值 |
| 24 | func.cmake:695/710 | 芯片映射表仅覆盖5/12 |
