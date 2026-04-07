## ops-transformer项目缺陷模式与审查规则

数据来源: ops-transformer(1323次提交, 243条缺陷中提炼46条规则) + ops-transformer-dev(2822次提交, 528条缺陷中提炼42条规则)
合并后: 去重整合为72条统一规则集(含SYS跨类别和TEST附录)

ops-transformer是算子仓库，缺陷模式与通信库(hcomm/hccl)有本质差异。通信库侧重并发/协议/资源生命周期，算子仓库侧重精度/tiling/shape/数据类型/kernel参数。两仓提交历史完全独立(零共享hash)，缺陷类别从数据中自然涌现。

### 严重等级定义

| 等级 | 含义 | 影响范围                                    |
|------|------|---------------------------------------------|
| P0   | 致命 | CoreDump、数据损坏、内存越界、死锁           |
| P1   | 严重 | 特定配置崩溃、静默精度错误、功能不可用        |
| P2   | 一般 | 边界条件异常、构建失败、测试不通过            |
| P3   | 建议 | 代码质量、可维护性、潜在隐患                  |

### 规则总览

| 规则ID     | 类别           | 严重等级 | 规则名                                   | 来源                                    |
|------------|----------------|----------|------------------------------------------|-----------------------------------------|
| COND-01    | 条件分支/校验  | P0       | OP_CHECK_IF/OP_CHECK条件语义反转         | [ops-transformer][ops-transformer-dev]   |
| COND-02    | 条件分支/校验  | P0       | 逻辑运算符OR/AND混淆导致恒真/恒假        | [ops-transformer][ops-transformer-dev]   |
| COND-03    | 条件分支/校验  | P1       | 新增枚举/Layout/模式时分支覆盖不全       | [ops-transformer][ops-transformer-dev]   |
| COND-04    | 条件分支/校验  | P1       | 校验过严误拦合法输入                     | [ops-transformer][ops-transformer-dev]   |
| COND-05    | 条件分支/校验  | P1       | 零值/除零/边界未防御                     | [ops-transformer][ops-transformer-dev]   |
| COND-06    | 条件分支/校验  | P1       | dtype白名单校验缺失                      | [ops-transformer]                       |
| COND-07    | 条件分支/校验  | P2       | 校验顺序依赖错误                         | [ops-transformer]                       |
| BUILD-01   | 构建           | P2       | CMake OP_NAME/target名复制粘贴错误       | [ops-transformer][ops-transformer-dev]   |
| BUILD-02   | 构建           | P1       | 新平台编译选项/分支未覆盖                | [ops-transformer-dev]                   |
| BUILD-03   | 构建           | P1       | auto-sync编译选项与kernel代码冲突        | [ops-transformer-dev]                   |
| BUILD-04   | 构建           | P2       | include路径变更未同步                    | [ops-transformer][ops-transformer-dev]   |
| BUILD-05   | 构建           | P2       | 黑名单/分包/注册配置未更新               | [ops-transformer-dev]                   |
| BUILD-06   | 构建           | P2       | ENABLE_EXPERIMENTAL模式路径适配错误      | [ops-transformer]                       |
| BUILD-07   | 构建           | P1       | 安装脚本误删其他组件文件                 | [ops-transformer]                       |
| BUF-01     | Buffer/内存    | P0       | workspace偏移/大小计算溢出               | [ops-transformer][ops-transformer-dev]   |
| BUF-02     | Buffer/内存    | P0       | DataCopy参数/边界越界                    | [ops-transformer-dev]                   |
| BUF-03     | Buffer/内存    | P0       | GM访问前缺边界检查(check-before-access)  | [ops-transformer-dev]                   |
| BUF-04     | Buffer/内存    | P1       | UB buffer复用缺同步/清零                 | [ops-transformer-dev]                   |
| BUF-05     | Buffer/内存    | P0       | workspace多段地址偏移重叠                | [ops-transformer]                       |
| SYNC-01    | 同步/流水线    | P0       | 流水线屏障缺失(RAW hazard)               | [ops-transformer][ops-transformer-dev]   |
| SYNC-02    | 同步/流水线    | P0       | HardEvent类型与数据流方向不匹配          | [ops-transformer][ops-transformer-dev]   |
| SYNC-03    | 同步/流水线    | P0       | SetFlag/WaitFlag不配对                   | [ops-transformer][ops-transformer-dev]   |
| SYNC-04    | 同步/流水线    | P0       | SyncAll模板参数误用                      | [ops-transformer]                       |
| TPARAM-01  | Tiling参数     | P1       | tiling计算逻辑/函数替换错误              | [ops-transformer-dev]                   |
| TPARAM-02  | Tiling参数     | P1       | GQA场景gSize缩放因子遗漏                 | [ops-transformer]                       |
| TPARAM-03  | Tiling参数     | P0       | tiling除数为零                           | [ops-transformer]                       |
| TPARAM-04  | Tiling参数     | P1       | 整数除法不满足分配律                     | [ops-transformer]                       |
| TPARAM-05  | Tiling参数     | P1       | CeilDiv与CeilAlign语义混淆              | [ops-transformer]                       |
| SHAPE-01   | Shape/Layout   | P1       | 新Layout分支遗漏                         | [ops-transformer][ops-transformer-dev]   |
| SHAPE-02   | Shape/Layout   | P1       | Layout专属逻辑在错误Layout下执行         | [ops-transformer-dev]                   |
| SHAPE-03   | Shape/Layout   | P0       | 空tensor全链路处理缺失                   | [ops-transformer]                       |
| SHAPE-04   | Shape/Layout   | P0       | 空指针/特殊值未处理                      | [ops-transformer]                       |
| OVFL-01    | 整数溢出/类型  | P0       | 字节数/元素数混淆                        | [ops-transformer-dev]                   |
| OVFL-02    | 整数溢出/类型  | P0       | DataCopy stride/count截断溢出            | [ops-transformer][ops-transformer-dev]   |
| OVFL-03    | 整数溢出/类型  | P1       | 多维度连乘int32溢出                      | [ops-transformer][ops-transformer-dev]   |
| OVFL-04    | 整数溢出/类型  | P0       | 无符号整数减法下溢                       | [ops-transformer]                       |
| INIT-01    | 初始化         | P1       | 条件分支遗漏赋值(if赋值else未赋值)       | [ops-transformer-dev]                   |
| INIT-02    | 初始化         | P1       | 构造/基类初始化缺失                      | [ops-transformer][ops-transformer-dev]   |
| INIT-03    | 初始化         | P1       | 循环/batch切换时状态残留                 | [ops-transformer-dev]                   |
| API-01     | 接口           | P0       | Check函数返回值被丢弃                    | [ops-transformer-dev]                   |
| API-02     | 接口           | P1       | return false在graphStatus函数中(语义反转)| [ops-transformer-dev]                   |
| API-03     | 接口           | P1       | 函数签名/参数不匹配                      | [ops-transformer-dev]                   |
| API-04     | 接口           | P0       | GetInputShape与GetOptionalInputShape混淆 | [ops-transformer]                       |
| API-05     | 接口           | P1       | DataCopy参数结构体类型错误               | [ops-transformer]                       |
| PARAM-01   | 参数           | P1       | API参数语义/顺序混淆                     | [ops-transformer][ops-transformer-dev]   |
| PARAM-02   | 参数           | P2       | 常量名值不匹配/魔法数字                  | [ops-transformer-dev]                   |
| PLAT-01    | 硬件平台       | P1       | 新平台运行时/编译期分支未覆盖            | [ops-transformer][ops-transformer-dev]   |
| PLAT-02    | 硬件平台       | P2       | 平台特有寄存器/配置遗漏                  | [ops-transformer-dev]                   |
| TKEY-01    | TilingKey      | P0       | host/kernel tilingKey常量值不一致        | [ops-transformer-dev]                   |
| TKEY-02    | TilingKey      | P1       | tilingKey编码维度缺失                    | [ops-transformer-dev]                   |
| ALIGN-01   | 对齐           | P1       | 数据搬运对齐不满足硬件要求               | [ops-transformer-dev]                   |
| ALIGN-02   | 对齐           | P2       | 不必要的对齐破坏地址计算                 | [ops-transformer-dev]                   |
| CPASTE-01  | 复制粘贴       | P1       | 复制粘贴后标识符未替换                   | [ops-transformer][ops-transformer-dev]   |
| CPASTE-02  | 复制粘贴       | P1       | 平行函数组修改不同步                     | [ops-transformer-dev]                   |
| REVERT-01  | 功能回退       | P1       | 架构方案首次失败后应暂停评审             | [ops-transformer-dev]                   |
| REVERT-02  | 功能回退       | P2       | 大型新算子/大型变更合入门禁              | [ops-transformer-dev]                   |
| SYS-01     | 系统性         | P1       | 代码克隆/平行实现"修一处漏一处"          | [ops-transformer][ops-transformer-dev]   |
| SYS-02     | 系统性         | P0       | 跨文件隐式一致性约束                     | [ops-transformer][ops-transformer-dev]   |
| SYS-03     | 系统性         | P0       | 整数溢出/除零静默失败                    | [ops-transformer][ops-transformer-dev]   |
| SYS-04     | 系统性         | P1       | 硬件平台分支不完整                       | [ops-transformer][ops-transformer-dev]   |
| SYS-05     | 系统性         | P1       | GQA gSize缩放因子遗漏                    | [ops-transformer]                       |
| TEST-01    | 测试           | P2       | UT数据类型不匹配                         | [ops-transformer]                       |
| TEST-02    | 测试           | P2       | UT路径依赖闭源组件                       | [ops-transformer]                       |
| TEST-03    | 测试           | P3       | UT与production不同步                     | [ops-transformer]                       |

---

### 类别: 条件分支/校验逻辑 (COND)

两仓频次最高的缺陷类别(~105条)。集中在op_host的tiling层和参数校验层。子类型: 条件覆盖不全(~30)、校验过严误拦(~20)、逻辑取反/运算符错(~15)、校验缺失(~15)、early return跳过逻辑(~8)、零值/边界(~12)。

#### COND-01: OP_CHECK_IF/OP_CHECK条件语义反转 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: OP_CHECK_IF宏的设计语义是"参数为true时触发失败"，但开发者常将"合法条件"直接传入(不加取反)，导致合法输入被拒绝、非法输入被放行。两仓合计此模式出现约15次。

典型代码示例:
```cpp
// 缺陷代码 (b6ec4bfe86) [ops-transformer-dev]
OP_CHECK_IF(CheckSameShape(qShape, kShape), ...);  // true=合法 -> 触发失败

// 修复代码
OP_CHECK_IF(!CheckSameShape(qShape, kShape), ...);  // true=不合法 -> 正确
```

审查检查方法:
- 逐个核对OP_CHECK_IF参数的语义方向: 条件为true应表示"错误状态"
- 当参数是函数调用(如CheckXxx())时，确认函数返回true的含义是否与CHECK宏的fail语义一致
- 搜索diff中所有OP_CHECK_IF/OP_CHECK调用，特别关注直接传入bool函数(未取反)的用法
- CHECK_COND_RETURN宏条件取反与直觉相反(`9b12d59b`) [ops-transformer]

关联commit: `b6ec4bfe86` [ops-transformer-dev], `479084fc` [ops-transformer-dev], `c8cf3f011d` [ops-transformer-dev], `9b12d59b` [ops-transformer]

---

#### COND-02: 逻辑运算符OR/AND混淆导致恒真/恒假 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: `(x != A || x != B)` 恒为true、`(x == A && x == B)` 恒为false是经典逻辑错误。两仓多次出现在多枚举值校验场景中。

典型代码示例:
```cpp
// 缺陷代码 (67f1fffc) [ops-transformer-dev]
if (dimNum != DIM3 || dimNum != DIM4) {  // 恒true
    return GRAPH_FAILED;
}
// 修复代码
if (dimNum != DIM3 && dimNum != DIM4) {  // 正确
    return GRAPH_FAILED;
}
```

审查检查方法:
- 搜索diff中`!= ... ||`和`== ... &&`模式，逐一验证逻辑意图
- 多枚举值OR连接时确认每个值是否属于同一约束集
- 短路求值中，轻量无副作用条件是否放在前面(`6ed5ed33`) [ops-transformer]

关联commit: `67f1fffc` [ops-transformer-dev], `b6bdcb97` [ops-transformer-dev], `6ed5ed33` [ops-transformer]

---

#### COND-03: 新增枚举/Layout/模式时分支覆盖不全 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 新增枚举值(sparseMode、quantMode、Layout等)或运行模式后，代码中多处if/switch分支未覆盖新值，导致走入错误的默认路径。两仓合计约30条。

典型代码示例:
```cpp
// 缺陷代码 (c8cf3f011d) [ops-transformer-dev]
if (layoutType == NTD) {
    return GRAPH_FAILED;  // 误拦NTD下的合法Decode路径
}
// 修复代码
if (layoutType == NTD && isPrefillMLA) {
    return GRAPH_FAILED;
}
```

```cpp
// 缺陷代码 (8a852918) [ops-transformer]
// if-else只处理了常见路径，遗漏GQA+s1Size==1+非BNSD组合
if (isIFAMLA) { ... }
else if (!isGQA) { ... }
else { ... }  // GQA边角组合穿透到默认分支

// 修复代码
if (isIFAMLA) { ... }
else if (isGQA && s1Size == 1 && layoutType != BNSD) {
    actualS1Size = s1Size * gSize;  // 新增分支
}
else if (!isGQA) { ... }
```

审查检查方法:
- 新增枚举值时全局搜索该枚举类型的所有switch/if-else，确认新值已覆盖
- 检查default/else分支是否会对新值产生意外行为
- 多模式组合用表格系统枚举所有有效组合 [ops-transformer]
- if-else中更特化的条件必须在更泛化的条件之前 [ops-transformer]

关联commit: `c8cf3f011d` [ops-transformer-dev], `b4b9198148` [ops-transformer-dev], `a0077ae9` [ops-transformer-dev], `82dbcd14` [ops-transformer-dev], `ad627275f` [ops-transformer], `8a852918` [ops-transformer], `b48e75b7c` [ops-transformer]

---

#### COND-04: 校验过严误拦合法输入 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 防御性校验的条件范围过宽，将合法但非典型的输入路径拒绝。表现为新功能/新数据类型上线后被已有校验误拦。两仓合计约20条。包含空tensor过度拦截(<=0应为<0)。

典型代码示例:
```cpp
// 缺陷代码 (54692072) [ops-transformer]
OP_CHECK_IF(weightNDim_ <= 0,  // <=0 拦截了合法的 N=0
    OP_LOGE(..., "The n dim value should be positive..."),
    return false);

// 修复代码
OP_CHECK_IF(weightNDim_ < 0,   // <0 只拦截负值，放行 N=0
    OP_LOGE(..., "The n dim value should not be negative..."),
    return false);
```

审查检查方法:
- 新增校验时反向思考"是否会误拦合法场景"
- 新增数据类型/格式时检查已有校验是否需要扩展白名单
- 校验条件是`<0`还是`<=0`？0是否是合法值？ [ops-transformer]
- 新增feature/mode后现有校验路径是否需要放宽 [ops-transformer]

关联commit: `c8cf3f011d` [ops-transformer-dev], `1eb3f0ce04` [ops-transformer-dev], `54692072` [ops-transformer], `38c7162c` [ops-transformer], `aa9faa8c` [ops-transformer]

---

#### COND-05: 零值/除零/边界未防御 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 除数为0未检查、==0 vs <=0混淆、off-by-one错误。AlignUp/CeilDivision在除数为0时静默返回0，下游使用错误参数。两仓合计约15条。

审查检查方法:
- 每个除法/取模表达式核查除数是否可能为0
- AlignUp/CeilDivision调用的第二参数必须>0
- 无符号减法(a-b)检查a>=b，否则下溢为极大值

关联commit: `b7a0e9fced` [ops-transformer-dev], `479084fc` [ops-transformer-dev], `234c1c8b` [ops-transformer], `1f3291bf` [ops-transformer]

---

#### COND-06: dtype白名单校验缺失 [ops-transformer]

严重等级: P1

缺陷描述: 算子入口对输入tensor的dtype没有做合法性前置检查。不支持的dtype穿透校验层进入计算，产生未定义行为或误导性错误信息。

典型代码示例:
```cpp
// 缺陷代码 (8ceb72b6) [ops-transformer]
if (isPerTokenQuant) { ... }
else if (isPerBlockQuant) { ... }
// 不支持的dtype静默走入错误分支

// 修复代码
if (std::find(X_DTYPE_SUPPORT_LIST.begin(), X_DTYPE_SUPPORT_LIST.end(), xDtype)
    == X_DTYPE_SUPPORT_LIST.end()) {
    OP_LOGE(ACLNN_ERR_PARAM_INVALID, "x dtype %s is not supported", ...);
    return false;
}
```

审查检查方法:
- 算子入口是否有全量dtype白名单校验(而非仅在子分支内局部检查)
- 白名单是否覆盖了所有新增的dtype支持

关联commit: `8ceb72b6` [ops-transformer], `f8882f78` [ops-transformer]

---

#### COND-07: 校验顺序依赖错误 [ops-transformer]

严重等级: P2

缺陷描述: 校验函数的执行顺序不满足逻辑依赖。典型: 空tensor检查应在维度值检查之前，否则空tensor的dim(0)==0会触发误拦截。

典型代码示例:
```cpp
// 缺陷代码 (44e84d52) [ops-transformer]
while (GetDynamicInputShape(KEY_INDEX, i) != nullptr) {
    kTensorList[i] = GetDynamicInputShape(KEY_INDEX, i);
    OP_CHECK_IF(kTensorList[i]->GetDim(0) != 1, ...);  // 空tensor dim(0)==0 误拦
    i++;
}

// 修复代码 -- 先收集再判空再校验
```

审查检查方法:
- 校验函数执行顺序是否满足逻辑依赖
- 空tensor判断须在维度值判断之前
- CHECK宏的条件语义(true=通过 vs true=拦截)是否清晰一致

关联commit: `44e84d52` [ops-transformer], `9b12d59b` [ops-transformer]

---

### 类别: CMake/构建配置 (BUILD)

两仓最具结构性的缺陷来源(~99条)。cmake/custom_build.cmake(23次触及)和cmake/func.cmake(17次触及)是Top1和Top3缺陷热点文件。

#### BUILD-01: CMake OP_NAME/target名复制粘贴错误 [ops-transformer][ops-transformer-dev]

严重等级: P2

缺陷描述: 新增算子时从已有算子复制CMakeLists.txt，OP_NAME等关键标识符未替换为新算子名。两仓合计约15条。

典型代码示例:
```cmake
# 缺陷代码 (5508fbb6c9) [ops-transformer-dev]
set(OP_NAME MoeFinalizeRouting)  # 从finalize复制未改, 应为MoeInitRouting
```

审查检查方法:
- CMakeLists中OP_NAME、OPTYPE与所在目录名/算子实际名称逐字核对
- 新增算子PR中搜索diff是否包含其他算子名(复制来源的残留)

关联commit: `5508fbb6c9` [ops-transformer-dev], `d69eec6dab` [ops-transformer-dev], `80329f3a` [ops-transformer]

---

#### BUILD-02: 新平台编译选项/分支未覆盖 [ops-transformer-dev]

严重等级: P1

缺陷描述: 新增硬件平台(如ascend910_95/A5)时，多处CMakeLists/cmake中的平台条件分支(ASCEND_COMPUTE_UNIT)未添加新平台。8个moe CMakeLists一次性遗漏A5平台分支(eeba666256)是典型案例。

典型代码示例:
```cmake
# 缺陷代码 (eeba666256) [ops-transformer-dev]
if(ASCEND_COMPUTE_UNIT STREQUAL "ascend910b")
    target_compile_options(...)
elseif(ASCEND_COMPUTE_UNIT STREQUAL "ascend310p")
    target_compile_options(...)
# 缺少: elseif(ASCEND_COMPUTE_UNIT STREQUAL "ascend910_95")
```

审查检查方法:
- 新平台PR: 全局搜索ASCEND_COMPUTE_UNIT/socVersion条件，逐一确认新平台已覆盖
- 统计搜索命中数与实际修改数是否一致(遗漏=命中但未修改)

关联commit: `eeba666256` [ops-transformer-dev], `e9ec0142` [ops-transformer-dev]

---

#### BUILD-03: auto-sync编译选项与kernel代码冲突 [ops-transformer-dev]

严重等级: P1

缺陷描述: --cce-auto-sync=on时编译器自动插入同步屏障，但kernel代码中已手动插入同步指令。两者叠加导致重复同步或数据竞争。

审查检查方法:
- diff中出现auto-sync设置变更时，检查对应kernel是否有手动SetFlag/WaitFlag
- kernel中新增手动同步时，检查CMakeLists中auto-sync是否为off

关联commit: `38606b2b` [ops-transformer-dev], `e9ec0142` [ops-transformer-dev]

---

#### BUILD-04: include路径变更未同步 [ops-transformer][ops-transformer-dev]

严重等级: P2

缺陷描述: 头文件路径重组后，引用该头文件的cmake/源码未同步更新。include guard名冲突、依赖缺失也属此类。

审查检查方法:
- include路径变更时grep所有引用该头文件的文件，确认路径同步更新
- 新增include guard时搜索同名guard是否已存在

关联commit: `05c16e686c` [ops-transformer-dev], `bb90fa511` [ops-transformer-dev], `ef22ac8f1` [ops-transformer]

---

#### BUILD-05: 黑名单/分包/注册配置未更新 [ops-transformer-dev]

严重等级: P2

缺陷描述: 新增算子后operator_list.yaml、classify_rule.yaml、A5_OPS_BLACK_LIST等配置文件未同步更新。

审查检查方法:
- 新增算子PR中检查: classify_rule.yaml、operator_list.yaml是否已添加新算子
- 算子从黑名单中移除时确认对应平台编译/测试已通过

关联commit: `eeba666256` [ops-transformer-dev], `422548259b` [ops-transformer-dev]

---

#### BUILD-06: ENABLE_EXPERIMENTAL模式路径适配错误 [ops-transformer]

严重等级: P2

缺陷描述: ENABLE_EXPERIMENTAL模式下源文件在experimental/子目录中，但EXISTS检查和include路径拼接缺少experimental/前缀。同一函数连续两次修复说明此路径问题极易遗漏。

典型代码示例:
```cmake
# 缺陷代码 (fbade778b) [ops-transformer]
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/op_dir")  # 缺少experimental/前缀

# 修复代码
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/experimental/op_dir")
```

审查检查方法:
- ENABLE_EXPERIMENTAL模式下所有路径拼接是否正确加了experimental/前缀
- include路径的-I顺序是否正确(项目路径优先于系统路径)

关联commit: `fbade778b` [ops-transformer], `b0bf9fa2` [ops-transformer], `ef22ac8f1` [ops-transformer]

---

#### BUILD-07: 安装脚本误删其他组件文件 [ops-transformer]

严重等级: P1

缺陷描述: 安装脚本中entity="true"导致安装时先清除目标目录，误删该目录下其他组件的文件。

审查检查方法:
- 安装/卸载脚本是否会影响目标目录下其他组件的文件
- entity/clean类选项是否必要

关联commit: `83be205f` [ops-transformer]

---

### 类别: Buffer/内存/地址 (BUF)

kernel层和tiling层最危险的缺陷类别(~74条)。可导致OOM、精度错误、静默数据损坏。

#### BUF-01: workspace偏移/大小计算溢出 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: workspace大小 = batch * head * seq * dim * sizeof(type)等多维度连乘，int32乘法溢出导致workspace分配不足、后续写入越界。offset累加链中sizeof类型交替时任一类型错误导致后续全部错位。两仓合计约20条。

典型代码示例:
```cpp
// 缺陷场景 [ops-transformer-dev]
uint32_t wsSize = batchSize * gSize * headNumSize * kvSplitPart * vHeadSize * sizeof(float);
// 修复: 至少一个操作数提升到uint64
uint64_t wsSize = (uint64_t)batchSize * gSize * headNumSize * kvSplitPart * vHeadSize * sizeof(float);
```

```cpp
// 缺陷代码 (9691bcc3) [ops-transformer]
bufferSize = AlignUp(rawSize, BLOCK_SIZE) / sizeof(half);  // 单位错误
// 修复: AlignUp后直接使用，已是字节数
bufferSize = AlignUp(rawSize, BLOCK_SIZE);
```

审查检查方法:
- workspace/buffer大小计算: 涉及3个以上维度连乘时至少一个操作数cast到uint64
- sizeof链: 核对每段sizeof(type)与实际数据类型是否匹配
- offset累加: 验证每个offset = 前一个offset + 前一个size，链条无断裂
- AlignUp/CeilAlign后的值通常是字节数，不应再除以sizeof(type) [ops-transformer]
- 每个乘法因子的语义(维度数 x 类型宽度 x 组数)是否正确 [ops-transformer]

关联commit: `e5817c02` [ops-transformer-dev], `29edf728` [ops-transformer-dev], `9691bcc3` [ops-transformer], `69698404` [ops-transformer], `e233e106` [ops-transformer]

---

#### BUF-02: DataCopy参数/边界越界 [ops-transformer-dev]

严重等级: P0

缺陷描述: DataCopy搬运长度超buffer分配大小、stride超uint16/uint32上限截断、尾块搬运用固定块大小而非实际剩余大小。两仓合计约15条。

典型代码示例:
```cpp
// 缺陷代码 (2efc569c) [ops-transformer-dev]
DataCopy(dst, src, blockSize);
// 修复: 尾块用实际剩余大小
uint32_t copyLen = (isLastBlock) ? remainSize : blockSize;
DataCopy(dst, src, copyLen);
```

审查检查方法:
- DataCopy搬运长度 <= 目标buffer分配大小
- stride参数不超uint16(65535)/uint32上限; 超出时用DataCopyExtParams
- 尾块/边界块搬运需用min(blockSize, remainSize)

关联commit: `2efc569c` [ops-transformer-dev], `1dbd7856` [ops-transformer-dev], `cefdc8776e` [ops-transformer-dev]

---

#### BUF-03: GM访问前缺边界检查(check-before-access) [ops-transformer-dev]

严重等级: P0

缺陷描述: GetValue/SetValue等GM访问操作在边界检查之前执行，index越界时直接访问非法地址。

典型代码示例:
```cpp
// 缺陷代码 (af72ccb938) [ops-transformer-dev]
auto val = blockTablesGm_.GetValue(offset);  // 先读
if (offset >= blockTableWidth_) { break; }   // 后检查

// 修复代码
if (offset >= blockTableWidth_) { break; }   // 先检查
auto val = blockTablesGm_.GetValue(offset);  // 后读
```

审查检查方法:
- 每个GM GetValue/SetValue调用前是否有边界检查(index < size)
- 循环中的GM访问: 循环条件是否包含边界约束

关联commit: `af72ccb938` [ops-transformer-dev]

---

#### BUF-04: UB buffer复用缺同步/清零 [ops-transformer-dev]

严重等级: P1

缺陷描述: 两个逻辑buffer映射同一物理UB区间时缺同步屏障; buffer复用前未清零导致脏数据残留; 清零长度小于分配大小。

审查检查方法:
- buffer复用: 新用途开始前是否已清零或完全覆写
- 清零范围: Duplicate/memset的长度 >= buffer实际分配大小
- 同一物理UB被两个逻辑buffer使用时，中间必须有PipeBarrier

关联commit: `f55d2a35` [ops-transformer-dev], `64f59aae` [ops-transformer-dev], `949f8193fd` [ops-transformer-dev]

---

#### BUF-05: workspace多段地址偏移重叠 [ops-transformer]

严重等级: P0

缺陷描述: workspace分为多个段(如softmax段、sink段、alibi段)，新增条件性段后未更新所有下游段的起始地址计算，导致相邻段地址重叠、数据互踩。

典型代码示例:
```cpp
// 缺陷代码 (3ca57b44) [ops-transformer]
pseAlibiAddr = workspaceBase + softmaxSize;  // 遗漏了dsinksum段

// 修复代码
pseAlibiAddr = workspaceBase + softmaxSize + dsinkSumSize;
```

审查检查方法:
- 画出workspace内存布局图: 段0起始 | 段0大小 | 段1起始 | ...
- 新增段后检查所有后续段的偏移计算是否都加上了新段大小
- 条件性段(仅某些模式下存在)需要在偏移计算中也做条件判断

关联commit: `3ca57b44` [ops-transformer]

---

### 类别: 同步/流水线时序 (SYNC)

NPU kernel中最难调试的缺陷类型(~55条)。表现为偶现nan/精度抖动/hang死。

#### SYNC-01: 流水线屏障缺失(RAW hazard) [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: 数据在管道A(如V/MTE2)产生后被管道B(如MTE3/V)消费，中间缺少对应管道的PipeBarrier或HardEvent同步。两仓合计约15条。

典型代码示例:
```cpp
// 缺陷代码 (d2601de86d) [ops-transformer-dev]
DataCopyPad(gmDst, v0ValidSizeUb_, ...);  // MTE3写GM
// 缺少: WaitFlag<MTE3_S>
// 后续代码复用v0ValidSizeUb_ -> 数据竞争
```

```cpp
// 缺陷代码 (984d8a72) [ops-transformer]
this->MTE3ToVSync();
DataCopy(inQQueBeforeCastLocal, key_in_GM[key_in_offset], ...);
// 缺少VToMTE2Sync() -> Vector可能仍在处理query
```

审查检查方法:
- 每个DataCopy/Duplicate后: 在消费者读取同一buffer前有对应管道的barrier/event
- 确认生产者管道(写)和消费者管道(读)不同时有跨管道同步
- 特别注意buffer被写入GM后立即复用的场景

关联commit: `d2601de86d` [ops-transformer-dev], `949f8193fd` [ops-transformer-dev], `37105bc0` [ops-transformer-dev], `984d8a72` [ops-transformer], `368c7325` [ops-transformer]

---

#### SYNC-02: HardEvent类型与数据流方向不匹配 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: HardEvent类型指定了生产者->消费者的管道对，类型选错导致同步错误管道对。SyncFunc模板参数方向反转也属此类。

典型代码示例:
```cpp
// 缺陷代码 (d5c79c44c7) [ops-transformer-dev]
SetFlag<HardEvent::V_MTE2>(eventId);    // 同步V->MTE2
// 实际数据流: MTE3产出 -> MTE2消费, 应为MTE3_MTE2

// 缺陷代码 (662f162c) [ops-transformer]
SyncFunc<AscendC::HardEvent::V_MTE2>();  // 方向反了
// 数据流是MTE2先加载、Vector后消费，应是MTE2->V方向
```

审查检查方法:
- HardEvent类型: 核对模板参数中的管道对是否与实际数据流方向一致
- SyncFunc模板参数A_B表示"A完成后B可以开始"，与数据流方向一致
- 方向错误的同步不仅无效，还可能掩盖真正需要的同步点 [ops-transformer]

关联commit: `d5c79c44c7` [ops-transformer-dev], `662f162c` [ops-transformer]

---

#### SYNC-03: SetFlag/WaitFlag不配对 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: 未Set即Wait导致死锁; 多余Wait导致流水线停顿; 条件分支中Set/Wait路径不对称。

典型代码示例:
```cpp
// 缺陷代码 (ee07e5dd25) [ops-transformer-dev]
// sink场景分配独立event, 但首次迭代未set即wait -> 死锁

// 缺陷代码 (a58c8a0b9) [ops-transformer]
// 两分支SetFlag不同event，WaitFlag只等其中一个 -> 信号积压
```

审查检查方法:
- 全局搜索SetFlag/WaitFlag配对，确认每个Wait都有对应的Set
- 同一eventId的Set和Wait之间无条件跳转
- 条件分支中的event使用: if分支Set了event则else分支也必须处理

关联commit: `ee07e5dd25` [ops-transformer-dev], `a58c8a0b9` [ops-transformer]

---

#### SYNC-04: SyncAll模板参数误用 [ops-transformer]

严重等级: P0

缺陷描述: SyncAll<true>和SyncAll<false>语义不同，用错导致同步不充分或多流死锁。使用SyncAll的kernel在host tiling侧必须设置SetScheduleMode(BATCH_MODE_SCHEDULE)。

审查检查方法:
- SyncAll<true>和SyncAll<false>的语义差异是否被正确理解
- 使用SyncAll的kernel，host tiling是否设置了SetScheduleMode
- 同步ID/flag常量值是否全局唯一(特别是super-kernel场景)

关联commit: `5dce387d` [ops-transformer], `65cafae0` [ops-transformer], `5644a7b` [ops-transformer]

---

### 类别: Tiling参数/计算 (TPARAM)

tiling层计算逻辑错误(~55条)。主仓CALC类别中非buffer子模式映射到此处。

#### TPARAM-01: tiling计算逻辑/函数替换错误 [ops-transformer-dev]

严重等级: P1

缺陷描述: CeilAlign被错误替换为CeilDiv; early return跳过tiling计算导致字段未赋值; 校验魔数与tiling实际输出范围不匹配。

典型代码示例:
```cpp
// 缺陷代码 (586493dc) [ops-transformer-dev]
ubFactor = CeilDiv(rawSize, alignment);  // 应为CeilAlign, 结果130016 vs 4063
```

审查检查方法:
- CeilAlign vs CeilDiv: 两者语义完全不同(对齐 vs 除法)，替换需验证
- early return: 确认return前所有输出字段都已被赋值

关联commit: `586493dc` [ops-transformer-dev], `66337ddb6b` [ops-transformer-dev], `fb58b43e98` [ops-transformer-dev], `1fab7bab45` [ops-transformer-dev]

---

#### TPARAM-02: GQA场景gSize缩放因子遗漏 [ops-transformer]

严重等级: P1

缺陷描述: GQA(Grouped Query Attention)中Q head数是KV head数的gSize倍。涉及Q-KV维度交叉计算时遗漏gSize缩放因子。两仓最频繁的单一具体根因，跨6条独立commit反复出现。

典型代码示例:
```cpp
// 缺陷代码 (d452dde6) [ops-transformer]
tilingData.preTokens = preTokens;            // 缺少 * gSize
tilingData.nextTokens = nextTokens;          // 缺少 * gSize

// 修复代码
tilingData.preTokens = preTokens * gSize;
tilingData.nextTokens = nextTokens * gSize;
```

审查检查方法:
- 全局搜索preTokens/nextTokens/actualSeqLength等Q-KV交叉变量，确认GQA场景下是否乘了gSize
- MLA/GQA的条件分支是否完整覆盖所有layout x mode组合

关联commit: `d452dde6` [ops-transformer], `81213d24` [ops-transformer], `8a852918` [ops-transformer], `3bb75678` [ops-transformer], `e44028b1` [ops-transformer], `16bc59a1` [ops-transformer]

---

#### TPARAM-03: tiling除数为零 [ops-transformer]

严重等级: P0

缺陷描述: tiling计算中从UB容量或shape维度推导的中间变量可能为零，后续CeilDiv触发CoreDump。

典型代码示例:
```cpp
// 缺陷代码 (234c1c8b) [ops-transformer]
uint32_t maxNPerLoopForUb = ubSize / headDim / sizeof(float);  // = 0
uint32_t loopCount = CeilDiv(totalN, maxNPerLoopForUb);  // 除零CoreDump
```

审查检查方法:
- 所有除法/CeilDiv的除数变量，追溯其来源是否可能为零
- 从UB容量推导的中间变量在极端参数下的值
- shape解析中空tensor(维度为0)场景必须在除法前拦截

关联commit: `234c1c8b` [ops-transformer], `1f3291bf` [ops-transformer]

---

#### TPARAM-04: 整数除法不满足分配律 [ops-transformer]

严重等级: P1

缺陷描述: `CeilDiv(a+b, n)` != `CeilDiv(a, n) + CeilDiv(b, n)`。NTD/变长序列场景中全局计算与逐batch计算结果不同。

典型代码示例:
```cpp
// 缺陷代码 (16bc59a1) [ops-transformer]
scaleOffset = totalSeqLen >> log2BlockSize;  // sum(a_i)/B

// 修复代码
for (int b = 0; b < batchSize; b++) {
    scaleOffset += CeilDiv(seqLen[b], blockSize);  // sum(CeilDiv(a_i, B))
}
```

审查检查方法:
- 涉及整数除法的复合表达式，检查是否依赖了分配律
- NTD/变长序列场景确认是逐batch计算还是全局计算

关联commit: `16bc59a1` [ops-transformer], `1ceed275` [ops-transformer]

---

#### TPARAM-05: CeilDiv与CeilAlign语义混淆 [ops-transformer]

严重等级: P1

缺陷描述: CeilDiv(x,n)返回"x需要多少个n大小的块"(分块数)，CeilAlign(x,n)返回"x对齐到n倍后的大小"(对齐后字节数)。两者差n倍。

审查检查方法:
- 赋给buffer size/offset的变量应使用CeilAlign
- 赋给loop count/block count的变量应使用CeilDiv

关联commit: `84ca56e9` [ops-transformer]

---

### 类别: Shape/Layout处理 (SHAPE)

~53条缺陷。

#### SHAPE-01: 新Layout分支遗漏 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 新增Layout(NTD/TND/BSND等)后，代码中多处if/switch未覆盖新Layout。两仓合计约15条。

典型代码示例:
```cpp
// 缺陷代码 (b4b9198148) [ops-transformer-dev]
if (layout == BSH) { ... }
else if (layout == BNSD) { ... }
// 缺少: else if (layout == NTD) { ... }
```

审查检查方法:
- 新增Layout PR: 搜索所有引用现有Layout枚举的条件分支，逐一确认新Layout已覆盖
- 统计搜索命中数与实际修改数

关联commit: `b4b9198148` [ops-transformer-dev], `a0077ae9` [ops-transformer-dev], `82dbcd14` [ops-transformer-dev], `ad627275f` [ops-transformer], `fcff1be7` [ops-transformer]

---

#### SHAPE-02: Layout专属逻辑在错误Layout下执行 [ops-transformer-dev]

严重等级: P1

缺陷描述: 某些计算逻辑仅对特定Layout有效，但条件范围过宽导致对所有Layout执行。

典型代码示例:
```cpp
// 缺陷代码 (0166982e) [ops-transformer-dev]
gmOffset += batchIdx * seqLen * headDim;  // BSND下不需此计算

// 修复: 仅BSH/BNSD下执行
if (layout == BSH || layout == BNSD) {
    gmOffset += batchIdx * seqLen * headDim;
}
```

审查检查方法:
- 偏移量/stride计算: 确认计算公式是否对当前Layout成立
- 使用if constexpr限制Layout专属逻辑

关联commit: `0166982e` [ops-transformer-dev], `ec9278e0` [ops-transformer-dev]

---

#### SHAPE-03: 空tensor全链路处理缺失 [ops-transformer]

严重等级: P0

缺陷描述: 空tensor处理必须aclnn/infershape/tiling/kernel四层全链路配套。任何一层遗漏都会导致问题。

典型代码示例:
```cpp
// 缺陷代码 (b5d7c0fe) [ops-transformer]
// InferShape层: 空tensor的shape未拷贝到输出
// Tiling层: 完全缺少清零逻辑
// Kernel层: 输出含脏数据
// 修复: 四层联动
```

审查检查方法:
- 空tensor(维度=0)是否在aclnn/infershape/tiling/kernel四层全链路处理
- kernel中空tensor输出是否初始化(ZerosLike或清零)

关联commit: `b5d7c0fe` [ops-transformer], `236a5fdb` [ops-transformer], `78e07300` [ops-transformer]

---

#### SHAPE-04: 空指针/特殊值未处理 [ops-transformer]

严重等级: P0

缺陷描述: OP_LOGE中%s格式化传入nullptr导致崩溃; exp(+inf - +inf) = NaN污染输出。

典型代码示例:
```cpp
// 缺陷代码 (d3aa4960) [ops-transformer]
OP_TILING_CHECK((commModePtr == nullptr || ...),
    OP_LOGE("commMode is %s", commModePtr),  // nullptr时崩溃
    return ge::GRAPH_FAILED);

// 修复: 拆分为两个独立CHECK
```

审查检查方法:
- OP_LOGE/OP_LOGW中%s是否可能传入nullptr
- exp/log/div/sqrt等数学运算是否处理了inf/NaN边界

关联commit: `d3aa4960` [ops-transformer], `4801ecb2` [ops-transformer]

---

### 类别: 整数溢出/类型转换 (OVFL)

~48条缺陷。

#### OVFL-01: 字节数/元素数混淆 [ops-transformer-dev]

严重等级: P0

缺陷描述: API期望元素数但传入字节数(或反之)，导致操作范围成倍偏差。

典型代码示例:
```cpp
// 缺陷代码 (798597da0a) [ops-transformer-dev]
Duplicate(dst, value, byteOffset);  // API期望元素数, 传入字节偏移, 4倍越界

// 修复代码
Duplicate(dst, value, elementCount);
```

审查检查方法:
- 每个API调用核对文档确认参数单位
- Duplicate/InitOutput/DataCopy: 参数名含"count/num"则为元素数

关联commit: `798597da0a` [ops-transformer-dev], `45cfad32` [ops-transformer-dev]

---

#### OVFL-02: DataCopy stride/count截断溢出 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: DataCopyParams的srcStride/dstStride为uint16_t(上限65535)，大shape时截断。

典型代码示例:
```cpp
// 缺陷代码 (cefdc8776e) [ops-transformer-dev]
params.srcStride = (uint16_t)srcStride;  // srcStride > 65535时截断

// 修复: 使用DataCopyExtParams
```

审查检查方法:
- DataCopy的stride/count参数: 检查输入shape是否可能超uint16上限
- 大shape场景优先使用DataCopyExtParams

关联commit: `cefdc8776e` [ops-transformer-dev], `025c1734` [ops-transformer-dev], `e13d1ec19` [ops-transformer], `e96b7b2a` [ops-transformer]

---

#### OVFL-03: 多维度连乘int32溢出 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 地址偏移 = uint32 * uint32在大数据场景(>4GB)溢出。

审查检查方法:
- 3个以上维度连乘: 至少一个操作数cast到uint64/int64
- GM地址计算: 结果变量必须为uint64_t

关联commit: `29edf728` [ops-transformer-dev], `e5817c02` [ops-transformer-dev], `831ab170` [ops-transformer], `6b029d5f` [ops-transformer]

---

#### OVFL-04: 无符号整数减法下溢 [ops-transformer]

严重等级: P0

缺陷描述: `a - b`当a < b时uint32_t回绕为极大正数。多核分核场景尾核最易触发。

典型代码示例:
```cpp
// 缺陷代码 (99a876f9) [ops-transformer]
uint32_t tailSize = totalOutputSize - aivIdx * singleCoreSize;  // 下溢

// 修复代码 -- 安全减法
uint32_t product = aivIdx * singleCoreSize;
uint32_t tailSize = (totalOutputSize > product) ? (totalOutputSize - product) : 0;
```

审查检查方法:
- 所有uint减法: a是否可能小于b？固定修复: `a > b ? a - b : 0U`
- 重点关注多核分核(SplitCore)场景
- 结构体字段类型是否与上游API类型一致(int64_t vs uint64_t)(`78282635`) [ops-transformer]

关联commit: `99a876f9` [ops-transformer], `b4baa0d2` [ops-transformer], `78282635` [ops-transformer]

---

### 类别: 接口/返回值 (API)

~37条缺陷。

#### API-01: Check函数返回值被丢弃 [ops-transformer-dev]

严重等级: P0

缺陷描述: 调用CheckXxx()函数后不检查返回值，即使返回GRAPH_FAILED也继续执行。

典型代码示例:
```cpp
// 缺陷代码 (0f363800) [ops-transformer-dev]
CheckFAIIsTND(context);  // 返回值被完全丢弃

// 修复代码
auto ret = CheckFAIIsTND(context);
if (ret != ge::GRAPH_SUCCESS) { return ret; }
```

审查检查方法:
- diff中所有Check/Verify函数调用: 返回值是否被检查并传播
- 考虑对Check函数添加[[nodiscard]]属性

关联commit: `0f363800` [ops-transformer-dev], `e7278f18` [ops-transformer-dev]

---

#### API-02: return false在graphStatus函数中(语义反转) [ops-transformer-dev]

严重等级: P1

缺陷描述: 函数声明返回ge::graphStatus(整型)但return false/true。bool->int隐式转换: false=0=GRAPH_SUCCESS。

典型代码示例:
```cpp
// 缺陷代码 (479084fc) [ops-transformer-dev]
ge::graphStatus CheckUbSpace(...) {
    if (condition) {
        return false;  // 意图返回失败, 但 false=0=GRAPH_SUCCESS
    }
}
// 修复: return ge::GRAPH_FAILED
```

审查检查方法:
- 返回ge::graphStatus的函数中搜索return true/false
- LOGE后检查是否紧跟return ge::GRAPH_FAILED

关联commit: `479084fc` [ops-transformer-dev]

---

#### API-03: 函数签名/参数不匹配 [ops-transformer-dev]

严重等级: P1

缺陷描述: extern声明与实际定义参数列表不一致; 指针值传递应为引用; SetOutputDataType参数写反。

典型代码示例:
```cpp
// 缺陷代码 (0c941da3) [ops-transformer-dev]
void ProcessData(SomeType* ptr) {  // 值传递指针
    ptr = newPtr;  // 修改局部副本, 调用方不可见
}
// 修复: void ProcessData(SomeType*& ptr)
```

审查检查方法:
- extern声明与实际函数定义参数列表逐一比对
- 函数内修改指针本身时: 参数应为指针引用或二级指针

关联commit: `0c941da3` [ops-transformer-dev], `7eee21c7` [ops-transformer-dev], `051f878c` [ops-transformer-dev]

---

#### API-04: GetInputShape与GetOptionalInputShape混淆 [ops-transformer]

严重等级: P0

缺陷描述: 对可选输入使用了必选输入的访问接口，可选输入不存在时越界或返回无效指针。

典型代码示例:
```cpp
// 缺陷代码 (73c4fdaa) [ops-transformer]
auto* shape = context->GetInputShape(LEARNABLE_SINK_INDEX);  // 可选输入不存在时越界
// 修复: GetOptionalInputShape + nullptr检查
```

审查检查方法:
- 可选输入是否使用GetOptionalInputShape而非GetInputShape
- 检查算子注册中哪些输入标记为optional

关联commit: `73c4fdaa` [ops-transformer], `d0ede159b` [ops-transformer], `a7010fc9` [ops-transformer]

---

#### API-05: DataCopy参数结构体类型错误 [ops-transformer]

严重等级: P1

缺陷描述: DataCopy系列API有多个参数结构体(DataCopyParams / DataCopyExtParams / DataCopyPadParams)，传错类型运行时行为未定义。

审查检查方法:
- DataCopyParams用于DataCopy，DataCopyExtParams用于DataCopyExt，DataCopyPadParams用于DataCopyPad
- 不依赖编译器隐式转换

关联commit: `ac0860f7` [ops-transformer]

---

### 类别: 初始化/赋值遗漏 (INIT)

~42条缺陷。

#### INIT-01: 条件分支遗漏赋值(if赋值else未赋值) [ops-transformer-dev]

严重等级: P1

缺陷描述: 变量仅在if分支赋值，else分支保持默认值(通常为0)，unsigned的0减1下溢为UINT32_MAX。

典型代码示例:
```cpp
// 缺陷代码 (b7a0e9fced) [ops-transformer-dev]
uint32_t kvSLoopNumTotal = 0;
if (condition) {
    kvSLoopNumTotal = calcValue;
}
// else: 保持0, 后续 0-1U = UINT32_MAX
```

审查检查方法:
- if赋值的变量: 检查else/default分支是否也赋值
- unsigned变量: 减法操作前确认变量>0

关联commit: `b7a0e9fced` [ops-transformer-dev], `3aacdac4` [ops-transformer-dev], `66337ddb6b` [ops-transformer-dev]

---

#### INIT-02: 构造/基类初始化缺失 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 子类构造函数未调用基类InitCompileInfo()等初始化方法，或结构体字段声明时未给初始值。

审查检查方法:
- 新增子类: 是否调用基类Init方法
- 结构体新增字段: 是否有默认初始值或在所有构造路径被赋值
- tiling结构体是否满足硬件对齐要求(64位对齐)(`312d30359`) [ops-transformer]

关联commit: `1fab7bab45` [ops-transformer-dev], `3aacdac4` [ops-transformer-dev], `8e9fe171` [ops-transformer], `312d30359` [ops-transformer]

---

#### INIT-03: 循环/batch切换时状态残留 [ops-transformer-dev]

严重等级: P1

缺陷描述: 多batch/多循环迭代间，上一轮的中间状态未被正确reset。

典型代码示例:
```cpp
// 缺陷代码 (1d256572) [ops-transformer-dev]
for (int b = 0; b < batchSize; b++) {
    // ropeOffset未在batch切换时重新计算
    ProcessBatch(b, ropeOffset);  // 第二个batch用了第一个batch的offset
}
```

审查检查方法:
- 循环体: 上一迭代的中间变量是否在新迭代开始时reset
- buffer复用: 跨迭代buffer是否需要清零

关联commit: `1d256572` [ops-transformer-dev], `64f59aae` [ops-transformer-dev]

---

### 类别: 参数值/顺序/魔数 (PARAM)

~37条缺陷。

#### PARAM-01: API参数语义/顺序混淆 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 函数参数位置调换; 参数语义与实际传值相反。参数>3个时尤其易错。

典型代码示例:
```cpp
// 缺陷代码 (e4b3ba6d) [ops-transformer-dev]
params.elementLengths[0] = mrgDstNum;  // 应为mrgSrcNum
srcList.src1 = mrgDst;                 // 应为mrgSrc
// Dst/Src的index顺序互换
```

```cpp
// 缺陷代码 (6978efa7) [ops-transformer]
IMPL_INFER_SHAPE(OpName, paramB, paramA)  // 顺序反了
```

审查检查方法:
- API调用核对参数与函数声明中参数名的语义对应(不仅看类型)
- 函数参数>3个: 考虑用结构体封装避免位置混淆

关联commit: `e4b3ba6d` [ops-transformer-dev], `7075b6143f` [ops-transformer-dev], `6978efa7` [ops-transformer], `e40a4176` [ops-transformer]

---

#### PARAM-02: 常量名值不匹配/魔法数字 [ops-transformer-dev]

严重等级: P2

缺陷描述: 常量名与值矛盾(ZERO_DIM=1); 非标准无穷大表示(3e+99而非IEEE 754)。

典型代码示例:
```cpp
// 缺陷代码 (cfe618f7) [ops-transformer-dev]
constexpr int ZERO_DIM = 1;  // 名为ZERO实际值为1
constexpr int TWO_DIM = 3;   // 名为TWO实际值为3
```

审查检查方法:
- 常量定义: 名称与值必须语义一致
- 魔法数字: 边界值/阈值需定义为命名常量并附注推导来源
- 无穷大: 使用std::numeric_limits<T>::infinity()而非字面量

关联commit: `cfe618f7` [ops-transformer-dev], `cb1c8346` [ops-transformer-dev], `fb58b43e98` [ops-transformer-dev]

---

### 类别: 硬件平台兼容性 (PLAT)

~28条缺陷。

#### PLAT-01: 新平台运行时/编译期分支未覆盖 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: socVersion字符串比较路由、`#if __CCE_AICORE__`预处理分支、ASCEND_COMPUTE_UNIT条件，新增硬件型号时多处遗漏。

典型代码示例:
```cpp
// 缺陷代码 (4bc16847) [ops-transformer-dev]
constexpr int cvRatio_ = 2;  // 硬编码, A5平台c:v=1:1

// 修复: 运行时获取
int cvRatio_ = GetSubBlockNum();
```

审查检查方法:
- 新平台适配PR: 搜索socVersion/ASCEND_COMPUTE_UNIT/__CCE_AICORE__所有出现位置
- 硬编码硬件参数: 应通过API获取而非hardcode
- constexpr用于平台参数时确认是否需要改为运行时获取

关联commit: `4bc16847` [ops-transformer-dev], `bf74834` [ops-transformer-dev], `b85709b25e` [ops-transformer-dev], `4d24f605` [ops-transformer], `c04bfe85` [ops-transformer], `66b12bf2` [ops-transformer]

---

#### PLAT-02: 平台特有寄存器/配置遗漏 [ops-transformer-dev]

严重等级: P2

缺陷描述: 新arch适配时遗漏平台特有的SPR/控制寄存器设置; InitGlobalMemory对齐要求跨平台不同。

审查检查方法:
- 新arch kernel: 检查是否需要特殊SPR/控制寄存器设置
- 平台切换时: InitGlobalMemory/对齐参数是否需要调整

关联commit: `b85709b25e` [ops-transformer-dev]

---

### 类别: TilingKey一致性 (TKEY)

~20条缺陷。

#### TKEY-01: host/kernel tilingKey常量值不一致 [ops-transformer-dev]

严重等级: P0

缺陷描述: host侧通过GET_TPL_TILING_KEY宏生成tilingKey，kernel侧通过TILING_KEY_IS匹配。两侧值不一致导致kernel走入错误分支。

审查检查方法:
- tilingKey修改的diff中: 必须同时出现host侧和kernel侧的对应修改
- GET_TPL_TILING_KEY宏参数顺序与kernel侧ASCENDC_TPL_SEL严格一致
- 建议: host/kernel共享tilingKey常量定义头文件

关联commit: `02505c922e` [ops-transformer-dev], `ee638d9b` [ops-transformer-dev], `31044c91` [ops-transformer-dev]

---

#### TKEY-02: tilingKey编码维度缺失 [ops-transformer-dev]

严重等级: P1

缺陷描述: 新增feature/配置参数时未编入tilingKey，不同配置映射到同一key。

典型代码示例:
```cpp
// 缺陷代码 (aba3bd8f) [ops-transformer-dev]
GetTilingKey(..., /*bit15=*/0, ...);  // 第15位传0而非tndS1Pingpong
```

审查检查方法:
- 新增配置参数时: 检查是否需要编入tilingKey
- 位域顺序: 核对每一位的含义注释与实际传值
- tilingKey位数: 确认未溢出uint64范围

关联commit: `31044c91` [ops-transformer-dev], `aba3bd8f` [ops-transformer-dev], `b41315b3` [ops-transformer-dev], `13ae078c` [ops-transformer-dev]

---

### 类别: 对齐计算 (ALIGN)

~18条缺陷。

#### ALIGN-01: 数据搬运对齐不满足硬件要求 [ops-transformer-dev]

严重等级: P1

缺陷描述: DataCopy搬运长度未满足32B对齐; NZ格式行数未2行对齐; Mmad的m/n/k维度未BLOCK_SIZE对齐。

审查检查方法:
- DataCopy/DataCopyPad: 搬运长度是否32B对齐; 不满足时用DataCopyPad
- NZ格式: 行数需2行(16元素行)对齐
- Mmad: m/n/k需BLOCK_SIZE对齐

关联commit: `3cfe2f3b64` [ops-transformer-dev], `446d15c9` [ops-transformer-dev], `c439c4d7` [ops-transformer-dev]

---

#### ALIGN-02: 不必要的对齐破坏地址计算 [ops-transformer-dev]

严重等级: P2

缺陷描述: 对不需要对齐的值执行对齐操作，对齐后值偏大破坏后续偏移量计算。

审查检查方法:
- 每个CeilAlign/RoundUp: 确认对齐是否确实需要
- 对齐后值作为偏移量乘数时: 偏大会导致后续所有地址错位

关联commit: `73a82c36` [ops-transformer-dev], `ac141d52` [ops-transformer-dev]

---

### 类别: 复制粘贴/拼写 (CPASTE)

~18条缺陷。

#### CPASTE-01: 复制粘贴后标识符未替换 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 从已有文件/函数复制代码后，关键标识符未替换为新上下文的正确值。

典型代码示例:
```cpp
// 缺陷代码 (120106d9) [ops-transformer-dev]
InitBuffer(inQueueL0A_, ASCEND_CB, ...);  // 应为ASCEND_L0A
InitBuffer(inQueueL0B_, ASCEND_CB, ...);  // 应为ASCEND_L0B
InitBuffer(inQueueL0C_, ASCEND_CB, ...);  // 应为ASCEND_L0C
```

审查检查方法:
- 新增文件/函数从已有代码复制时，逐行检查所有标识符是否已替换
- grep源算子/函数名: diff中不应残留复制来源的标识符

关联commit: `a02041efea` [ops-transformer-dev], `e8bdf8f2` [ops-transformer-dev], `120106d9` [ops-transformer-dev], `64be6dd6` [ops-transformer]

---

#### CPASTE-02: 平行函数组修改不同步 [ops-transformer-dev]

严重等级: P1

缺陷描述: V1/V2/V3等平行函数或clone文件，仅修改一处而未同步到平行代码。

审查检查方法:
- diff中只修改了某函数的一个版本: 检查平行版本是否也需同样修改
- aclnn接口V1-V5: 新增参数/check需同步所有入口

关联commit: `5508fbb6c9` [ops-transformer-dev], `e8bdf8f2` [ops-transformer-dev]

---

### 类别: 功能回退/Revert (REVERT)

~22条缺陷。

#### REVERT-01: 架构方案首次失败后应暂停评审 [ops-transformer-dev]

严重等级: P1

缺陷描述: tilingKey模板化整改在首个模块失败后未暂停，继续推进到其他MC2算子，5天内连续6次Revert。

审查检查方法:
- 同一架构方案首个模块失败并revert后: 要求根因分析和方案修正再继续
- 跨模块推广: 确认已有至少一个模块稳定运行

关联commit: `2001261f7c` [ops-transformer-dev], `8ead68ccfa` [ops-transformer-dev], `20f4d5412b` [ops-transformer-dev]

---

#### REVERT-02: 大型新算子/大型变更合入门禁 [ops-transformer-dev]

严重等级: P2

缺陷描述: premature merge中7个在合入后2小时内revert。commit message"接口调试"、MR描述空白是预警信号。大型变更(>10文件/1000行)一次性提交回退代价极高。

审查检查方法:
- 新增算子(>10新文件): 必须填写测试栏并提供CI通过证据
- commit message含"调试"/"临时"/"WIP": 不应合入主干
- 大型变更是否可以分阶段提交 [ops-transformer]
- 同步原语的"优化"是否证明了等价性 [ops-transformer]

关联commit: `1788aaceea` [ops-transformer-dev], `8611973812` [ops-transformer-dev]

---

### 类别: 跨类别系统性规则 (SYS)

#### SYS-01: 代码克隆/平行实现"修一处漏一处" [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 两仓23次cmake缺陷和11次MC2 kernel缺陷的根因。已识别克隆对: custom_build.cmake vs CMakeLists.txt(RTY) 285行平行; func.cmake两个macro 80%重复; combine_v2.h vs _add_rms_norm.h; aclnn_grouped_matmul.cpp 6个版本入口; PFA/IFA各自TilingDataconvert。

审查检查方法:
- diff仅修改一个克隆体时: 搜索对应克隆体是否需同步修改
- cmake/custom_build.cmake修改时: 检查CMakeLists.txt(RTY分支)是否需同步
- MC2 MoE: combine_v2.h修改时检查_add_rms_norm.h

---

#### SYS-02: 跨文件隐式一致性约束 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: 无编译期保证的跨文件约束是两仓高频缺陷源。tilingKey(host) vs 模板匹配(kernel): ~20条; workspace offset(host) vs kernel读取: ~8条; 宏参数顺序 vs 宏定义: ~5条; tiling/kernel计算公式不一致: ~6条 [ops-transformer]。

审查检查方法:
- tilingKey修改: diff必须同时包含host和kernel两侧
- workspace offset变更: 验证host侧sizeof链与kernel侧读取逻辑一致
- 建议: host/kernel共享常量定义(如tiling_key.h)

---

#### SYS-03: 整数溢出/除零静默失败 [ops-transformer][ops-transformer-dev]

严重等级: P0

缺陷描述: 几乎所有tiling和kernel文件都涉及多维度连乘或除法，普遍缺少溢出/除零检查。

审查检查方法:
- 多维度连乘: 至少一个操作数提升到int64/uint64
- 除法/取模: 除数为0时应报错而非静默返回0
- 变量类型: 地址/偏移量统一使用uint64_t

---

#### SYS-04: 硬件平台分支不完整 [ops-transformer][ops-transformer-dev]

严重等级: P1

缺陷描述: 新增硬件型号时多处遗漏。散布在socVersion路由、#if预处理分支、CMake条件、平台常量。

审查检查方法:
- 新平台适配PR: 搜索所有平台分支入口
- 平台参数: 优先用API获取而非hardcode
- checklist: 编译选项、运行时分支、寄存器配置、对齐要求四项逐一确认

---

#### SYS-05: GQA gSize缩放因子遗漏 [ops-transformer]

严重等级: P1

缺陷描述: 两仓最频繁的单一具体根因。GQA中Q head数是KV head数的gSize倍，所有Q-KV维度交叉计算都需要gSize缩放。问题反复出现的根因是gSize影响的变量散布在多个函数中，没有统一的"GQA适配层"。

审查检查方法:
- 涉及attention算子的PR，全局搜索preTokens/nextTokens/actualSeqLength等变量
- GQA路径中每个Q-KV维度交叉计算是否乘了gSize
- 新增layout或模式时，GQA组合是否被覆盖

关联commit: `d452dde6` [ops-transformer], `81213d24` [ops-transformer], `8a852918` [ops-transformer], `3bb75678` [ops-transformer], `e44028b1` [ops-transformer], `16bc59a1` [ops-transformer]

---

### 类别: UT/测试代码 (TEST)

主仓16条缺陷。

#### TEST-01: UT数据类型不匹配 [ops-transformer]

严重等级: P2

缺陷描述: UT中host数据类型与aclDataType不一致，或sizeof用错类型。

审查检查方法:
- CreateAclTensor的dataType与host数据的C++类型是否匹配
- sizeof()参数类型是否与实际测试数据类型一致

关联commit: `b9a02e9d` [ops-transformer], `a911707e` [ops-transformer]

---

#### TEST-02: UT路径依赖闭源组件 [ops-transformer]

严重等级: P2

缺陷描述: UT include引用闭源路径, system()拷贝闭源脚本，开源仓库无法编译。

审查检查方法:
- UT的include路径是否使用相对路径
- 开源环境下UT是否可独立编译运行

关联commit: `29b08607` [ops-transformer]

---

#### TEST-03: UT与production不同步 [ops-transformer]

严重等级: P3

缺陷描述: production代码重命名后UT未同步更新。

审查检查方法:
- production代码重命名/重构后，搜索UT中是否有旧名称引用
- UT中tiling结构体的所有字段是否初始化

关联commit: `8d72712c` [ops-transformer], `f1a24bd` [ops-transformer]

---

### 附录: 建议自动化检查 (AUTO)

| 规则ID   | 规则                           | 实现方式                                        |
|----------|--------------------------------|-------------------------------------------------|
| AUTO-01  | graphStatus函数中return bool   | 静态分析: 检测graphStatus函数中的bool返回值      |
| AUTO-02  | Check函数返回值丢弃            | clang-tidy: [[nodiscard]]                        |
| AUTO-03  | LOGE后缺return                 | 静态分析: LOGE调用后下一条非return语句            |
| AUTO-04  | 窄类型截断                     | -Wconversion/-Wnarrowing                         |
| AUTO-05  | 常量名值一致性                 | lint: 命名含数字的常量其值应匹配                  |
| AUTO-06  | CMake OP_NAME与目录名一致性     | CI脚本: 验证OP_NAME == 所在目录名                |

---

### 附录: 已确认存量bug

| 文件                                   | 缺陷                                          | 来源                 |
|----------------------------------------|-----------------------------------------------|----------------------|
| CMakeLists.txt:60-61                   | INDXE变量名拼写错误，arch32检测使用循环残留值   | [ops-transformer-dev]|
| incre_flash_attention_tiling_v2.cpp    | SetLayoutTypefaRun中BSH_BSND重复key            | [ops-transformer-dev]|
| incre_flash_attention_tiling_v2.cpp    | CheckUbSpace()返回bool而声明graphStatus        | [ops-transformer-dev]|
| cmake/variables.cmake:267             | set(AICPU_INCLUDE ...)无条件覆盖list(APPEND)   | [ops-transformer-dev]|

---

### 附录: 数据来源

| 指标       | ops-transformer                | ops-transformer-dev                |
|-----------|--------------------------------|------------------------------------|
| 仓库路径   | ~/repo/cann/ops-transformer/   | ~/repo/cann/ops-transformer-dev/   |
| 分析周期   | 全量git历史                     | 2025-08至2026-01                    |
| 总提交     | 1323                            | 2822 (非merge: 2820)                |
| 确认缺陷   | 243 (18.4%)                     | 788 (27.9%)                         |
| 代码缺陷   | 243                             | 528                                 |
| 缺陷类别   | 11                              | 15                                  |
| 审查规则   | 46                              | 42                                  |
| 合并后规则 | 72条 (含SYS和TEST)              |                                     |
