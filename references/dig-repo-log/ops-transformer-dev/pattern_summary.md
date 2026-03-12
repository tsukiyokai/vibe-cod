# ops-transformer-dev 缺陷模式归纳与分类

基于528条确认代码缺陷(阶段2)、35条Revert事件(阶段3)、30个热点文件分析(阶段4)的综合归纳。

## 总览

| 类别 | 频次 | 占比 | 可审查性 | 典型模块 |
|------|------|------|---------|---------|
| BUILD: CMake/构建配置 | ~65 | 12.3% | 高 | cmake/, CMakeLists.txt |
| COND: 条件分支/校验逻辑 | ~75 | 14.2% | 高 | tiling层, op_host |
| BUF: Buffer/内存/地址 | ~60 | 11.4% | 中 | kernel, tiling |
| SYNC: 同步/流水线时序 | ~35 | 6.6% | 中 | kernel (arch32/35) |
| TKEY: TilingKey一致性 | ~20 | 3.8% | 高 | host tiling + kernel |
| TPARAM: Tiling参数/计算 | ~20 | 3.8% | 中 | tiling层 |
| INIT: 初始化/赋值遗漏 | ~30 | 5.7% | 高 | kernel, tiling |
| OVFL: 整数溢出/类型/单位 | ~30 | 5.7% | 高 | tiling, kernel |
| PLAT: 硬件平台兼容性 | ~25 | 4.7% | 中 | 全局 |
| API: 接口/返回值 | ~25 | 4.7% | 高 | op_host, op_api |
| SHAPE: Shape/Layout处理 | ~28 | 5.3% | 高 | tiling, infershape |
| ALIGN: 对齐计算 | ~18 | 3.4% | 中 | kernel (DataCopy) |
| CPASTE: 复制粘贴/拼写 | ~18 | 3.4% | 高 | 全局 |
| PARAM: 参数值/顺序/魔数 | ~25 | 4.7% | 高 | kernel, tiling |
| REVERT: 功能回退 | ~16 | 3.0% | - | MC2, MoE, GMM |
| 其他/复合 | ~18 | 3.4% | - | - |

注: 频次为估算值，部分条目跨类别(如"条件分支缺失导致buffer越界")按主要根因归类。

---

## BUILD: CMake/构建配置缺陷 (~65条, 12.3%)

本仓最具结构性的缺陷来源。cmake/custom_build.cmake(23次触及)和cmake/func.cmake(17次触及)是缺陷热点Top1和Top3。

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 编译选项缺失/错误 | ~15 | 平台编译选项遗漏、拼写错误、auto-sync冲突 |
| CMake target名/路径错误 | ~12 | OP_NAME写错、target不存在、路径拼写 |
| 头文件include路径错误 | ~12 | 路径变更未同步、guard名冲突、依赖缺失 |
| 编译黑名单/分包配置 | ~10 | 算子注册位置错、黑名单遗漏、分包yaml |
| 链接配置缺失 | ~8 | 库依赖遗漏、stub缺失、undefined symbol |
| 环境变量/条件变量 | ~8 | 变量名拼写(INDXE)、条件判断错误 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 5508fbb6c9 | CMakeLists中OP_NAME写成MoeFinalizeRouting | copy-paste后未改OP_NAME |
| 5eb3fef2 | --cce-aicore-dcci-before-kernle-end拼写错误 | kernle应为kernel |
| d69eec6dab | add_modules_sources()缺少OPTYPE/ACLNNTYPE参数 | undefined symbol |
| 38606b2b | auto-sync=on与手动同步冲突 | 编译选项与kernel代码不匹配 |
| e9ec0142 | ascend910_95缺少--cce-auto-sync=off | 新平台编译选项分支未添加 |

### 审查检查点

1. 新增算子: CMakeLists中的OP_NAME、OPTYPE与算子实际名称逐字核对
2. 新增平台: 搜索所有CMakeLists/cmake中的平台条件分支(ASCEND_COMPUTE_UNIT)，确认新平台已覆盖
3. 编译选项: auto-sync设置与kernel中手动同步代码是否冲突
4. include路径变更: grep所有引用该头文件的cmake/源码，确认路径同步更新
5. 黑名单/分包: 新增算子后operator_list.yaml、classify_rule.yaml是否已更新

### 结构性风险(来自阶段4)

- custom_build.cmake与CMakeLists.txt(RTY分支)285行平行实现 -- "修一处漏一处"
- func.cmake两个macro(add_modules_sources / add_modules_sources_with_soc) 80%重复代码
- variables.cmake中AICPU_INCLUDE被set无条件覆盖前序list(APPEND)结果(已确认存量bug)
- CMakeLists.txt:60行INDXE拼写错误导致arch32检测逻辑使用循环残留值(已确认存量bug)

---

## COND: 条件分支/校验逻辑错误 (~75条, 14.2%)

本仓频次最高的缺陷类别。集中在op_host的tiling层和参数校验层。

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 条件覆盖不全(枚举/layout/模式) | ~20 | 新增枚举值/layout/sparseMode时遗漏分支 |
| 校验过严误拦合法输入 | ~15 | 条件范围过宽、边界值过严、新数据类型未豁免 |
| 逻辑取反/运算符错误 | ~12 | !=应为==、\|\|应为&&、OP_CHECK_IF缺少取反 |
| 校验缺失(空tensor/dtype等) | ~10 | 空tensor未拦截、非法dtype未检查 |
| early return跳过必要逻辑 | ~8 | 提前return导致后续初始化/计算被跳过 |
| 零值/边界条件 | ~10 | 除数为0未防御、==0 vs <=0、off-by-one |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| b6ec4bfe86 | 4处OP_CHECK_IF条件缺少取反 | 合法shape被拒绝，非法shape被放行 |
| 67f1fffc | `!= DIM3 \|\| != DIM4`恒为true | 经典"OR两个不等式"逻辑错误 |
| c8cf3f011d | NTD全量化拦截范围过宽 | 只有Prefill MLA不支持，但对所有NTD生效 |
| 479084fc | return false在graphStatus函数中 | bool->int隐式转换: false=0=GRAPH_SUCCESS |
| b6bdcb97 | \|\|应为&& | `(D<=256) \|\| (D相等)`导致误判 |

### 审查检查点

1. OP_CHECK_IF/OP_CHECK宏: 参数应表示"错误条件"(true触发失败)，审查时核对语义方向
2. 多枚举值OR连接: 逐一确认每个枚举值是否属于同一约束集
3. 新增枚举/layout/模式: 全局搜索switch/if-else分支，确认新值已覆盖
4. 空tensor路径: 每个可选输入都需检查nullptr/shape=0的处理路径
5. return false vs GRAPH_FAILED: 函数返回ge::graphStatus时禁止return bool值

---

## BUF: Buffer/内存/地址缺陷 (~60条, 11.4%)

kernel层和tiling层最危险的缺陷类别，可导致OOM/精度错误/静默数据损坏。

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| workspace偏移/大小计算错误 | ~15 | 多维度连乘溢出、offset累加遗漏、sizeof链断裂 |
| UB分配不足/复用冲突 | ~12 | InitBuffer传0、buffer复用缺同步、清零不足 |
| DataCopy参数/边界错误 | ~12 | stride溢出、搬运长度超界、Ext版本未使用 |
| GM地址计算/越界 | ~10 | check-before-access违反、uint32地址溢出 |
| buffer偏移链错误 | ~8 | tiling侧计算offset与kernel侧读取不匹配 |
| DMA对齐/padding | ~5 | 非32B对齐截断、padding脏数据 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 798597da0a | Duplicate的count传入字节偏移而非元素数 | 写入量4倍越界致OOM |
| af72ccb938 | GM读取在边界检查之前 | check-before-access违反 |
| 2efc569c | 尾块搬运超界覆写scale数据 | 应用实际剩余大小而非固定块大小 |
| 1dbd7856 | DataCopy按RoundUp(headSize)搬运 | 非32B对齐时超出GM连续空间 |
| e5817c02 | GM偏移量uint32溢出(>4GB) | 大数据场景需uint64 |

### 审查检查点

1. workspace偏移计算: 核对sizeof链(类型交替half->float->half时每段sizeof是否正确)
2. DataCopy: 搬运长度 <= buffer分配大小; stride参数不超uint16/uint32上限
3. GM访问: 所有GetValue/SetValue前有边界检查(index < size)
4. buffer复用: 两个逻辑buffer映射同一物理UB区间时，中间必须有同步屏障
5. 多维度连乘: workspace = batch * head * seq * dim * sizeof(type)至少一个提升到int64

### 结构性风险(来自阶段4)

- prompt_flash_attention_tiling_v2.cpp(4762行): workspace六值连乘溢出风险; PFATilingDataconvert 100+行手工字段搬运
- incre_flash_attention_tiling_v2.cpp: CheckUbSpace()返回bool但声明graphStatus，语义反转(已确认存量bug)
- fia_kernel_nonquant_mla.h: workspace每个buffer的offset依赖前一个sizeof链，任一修改导致后续全错位

---

## SYNC: 同步/流水线时序缺陷 (~35条, 6.6%)

NPU kernel中最难调试的缺陷类型。表现为偶现nan/精度抖动/hang死，难以复现。

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 流水线屏障(barrier)缺失 | ~10 | V->MTE3/MTE2->V等管道间缺同步 |
| 硬件事件(HardEvent)类型错误 | ~8 | V_MTE2应为MTE3_MTE2等 |
| 同步事件set/wait不配对 | ~6 | 未set即wait导致死锁; 多余wait导致停顿 |
| 多核同步遗漏 | ~5 | 缺SyncAll; cross-core buffer访问前缺同步 |
| PipeBarrier<PIPE_ALL>滥用 | ~3 | 全管道同步导致不必要的流水线停顿 |
| HCCL通信同步 | ~3 | Wait次数不足、通信操作顺序错 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| d2601de86d | DataCopyPad写GM后未等MTE3完成即复用buffer | RAW数据竞争 |
| 949f8193fd | Duplicate后未加PipeBarrier\<PIPE_V\>即读同一buffer | RAW hazard |
| d5c79c44c7 | 事件类型V_MTE2应为MTE3_MTE2 | 生产者-消费者关系错配 |
| 37105bc0 | Cast(V)输出未完成就被DataCopyPad(MTE3)读取 | 缺V->MTE3同步屏障 |
| ee07e5dd25 | sink场景分配独立事件但未set即wait | 死锁 |

### 审查检查点

1. 每个DataCopy/Duplicate后: 在消费者读取前是否有对应管道的barrier/event
2. HardEvent类型: 核对生产者管道->消费者管道的实际数据流方向
3. SetFlag/WaitFlag: 全局搜索配对，确认每个Wait都有对应的Set
4. PipeBarrier: PIPE_ALL仅用于真正需要全管道同步的场景; 精确指定管道类型
5. 多核: Process入口是否有SyncAll; cross-core tensor访问前是否有SyncAll

---

## TKEY: TilingKey一致性缺陷 (~20条, 3.8%)

host侧tiling生成的key值与kernel侧模板匹配常量不一致。本仓最具特征性的缺陷模式，也是Revert分析中最显著的系统性失败(事件群1: 6次tilingKey模板化整改连续失败)。

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| host/kernel常量值不一致 | ~8 | host下发key与kernel中TILING_KEY_IS值不匹配 |
| tiling key编码维度缺失 | ~5 | 新参数未编入key导致不同配置映射到同一key |
| tiling key重构/整改回归 | ~4 | 模板化整改引入新的不匹配 |
| 跨模块key常量不同步 | ~3 | 上游修改key编码值下游未同步 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 31044c91 | tilingKey未编码mte2Config | 不同vec_config映射到同一key |
| ee638d9b | 上游wqbmm修改编码值下游未同步 | 跨模块隐式一致性 |
| aba3bd8f | GetTilingKey第15位传0而非tndS1Pingpong | 不同配置生成相同key |
| b41315b3 | tilingKey使用UINT64_MAX(溢出值) | key计算溢出 |
| 13ae078c | FAG kernel中Layout==4应为3 | 枚举值写错 |

### 审查检查点

1. tilingKey新增/修改: diff中必须同时出现host侧(tiling.cpp)和kernel侧(kernel.h/.cpp)的对应修改
2. key编码维度: 新增feature/配置参数时检查是否需要编入key(否则不同配置共用同一kernel)
3. 位域顺序: GET_TPL_TILING_KEY宏参数顺序与kernel侧ASCENDC_TPL_SEL严格一致
4. 跨模块同步: 修改某算子的key编码后搜索所有引用该key的下游模块

### Revert关联(来自阶段3)

事件群1(6次Revert): tilingKey模板化整改在MatmulAllReduce、MatmulReduceScatterV2、MoeDistributeCombine/Dispatch四组MC2算子上连续失败。方案在首次失败后未暂停评审，继续推进到其他算子。审查启示: 架构方案首次失败后应要求根因分析再继续。

---

## TPARAM: Tiling参数/计算缺陷 (~20条, 3.8%)

tiling层计算逻辑错误，但不属于key一致性问题。

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 66337ddb6b | DUAL_IN_OUT模式early return跳过tiling计算 | vLoopNum_等未赋值 |
| fb58b43e98 | swizzlCount上界写死9(应16) | 校验魔数与实际输出范围不匹配 |
| 586493dc | CeilAlign被错误替换为CeilDiv | ubFactor从130016变4063 |
| 1fab7bab45 | 子类构造函数未调用基类InitCompileInfo() | 平台信息未初始化 |

### 审查检查点

1. tiling校验边界值: 魔数需标注推导来源; 变更后与tiling生成实际输出范围对齐
2. CeilAlign vs CeilDiv: 两者语义完全不同，替换需验证
3. 新增模式/分支: 检查所有tiling字段是否都被赋值(无遗漏分支)

---

## INIT: 初始化/赋值遗漏 (~30条, 5.7%)

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 条件分支遗漏赋值 | ~10 | if赋值else未赋值; switch缺default |
| 构造/基类初始化缺失 | ~6 | 未调基类Init; 结构体字段未初始化 |
| 清零/reset不完整 | ~5 | 清零长度<分配大小; 跨batch残留 |
| 初始化时序错误 | ~5 | 清零在early return后; 赋值在使用之后 |
| UB脏数据残留 | ~4 | buffer复用后未清零; 尾块padding脏数据 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| b7a0e9fced | kvSLoopNumTotal仅if赋值else保持0 | 0-1U下溢为UINT32_MAX |
| 3aacdac4 | AnalyzeAttrs未设groupType | 后续tiling使用未初始化值 |
| 1d256572 | batch切换缺ROPE偏移量更新 | 使用上一batch残留值 |
| f55d2a35 | statusTensor清零长度<buffer大小 | 尾部脏数据 |
| 64f59aae | FlashDecode场景未初始化softmaxLse | 使用脏值导致精度错误 |

### 审查检查点

1. 所有条件分支: 每个分支(含else/default)都必须对输出变量赋值
2. 构造函数: 是否调用基类Init; 所有成员是否有初始值
3. 循环/batch切换: 上一轮迭代的状态是否被正确reset
4. buffer复用: 新用途开始前是否已清零或完全覆写

---

## OVFL: 整数溢出/类型转换/单位混淆 (~30条, 5.7%)

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 字节数/元素数混淆 | ~8 | API期望元素数但传入字节数(或反之) |
| uint16/uint32截断 | ~8 | DataCopy stride溢出、CeilDivision返回值截断 |
| int32乘法溢出 | ~6 | 地址偏移 = uint32 * uint32溢出 |
| 数据类型字面量混用 | ~4 | 4-bit类型sizeof不符预期; half/bf16混淆 |
| 有符号/无符号比较 | ~4 | unsigned减法下溢; signed负数比较 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 798597da0a | Duplicate count传字节偏移而非元素数 | 4倍越界写入 |
| cefdc8776e | srcStride超uint16最大值截断 | 需用DataCopyExtParams |
| 025c1734 | CeilDivision返回uint16大s2截断 | 应用uint64版CeilDiv |
| 29edf728 | uint32*uint32 TND场景溢出 | 需先cast\<uint64_t\> |
| 45cfad32 | InitOutput参数语义为元素数传入字节数 | 初始化范围翻倍越界 |

### 审查检查点

1. 字节vs元素: 每个API调用核对文档确认参数单位(字节数需除以sizeof)
2. 乘法表达式: 涉及batch*head*seq*dim的乘法链，至少一个操作数cast到uint64
3. DataCopy stride/count: 检查实际数据规模是否可能超uint16(65535)/uint32上限
4. CeilDiv/CeilAlign返回类型: 确认与接收变量类型匹配

---

## PLAT: 硬件平台兼容性 (~25条, 4.7%)

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 新平台编译选项/分支未覆盖 | ~10 | A5(910_95)编译选项缺失 |
| 平台特有硬件行为差异 | ~8 | 寄存器配置、对齐要求、c:v比例 |
| 平台条件分支使用错误 | ~5 | 字符串比较路由遗漏; constexpr应为运行时 |
| 跨平台API差异 | ~2 | InitGlobalMemory对齐要求不同 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| b85709b25e | David平台MXFP8未设SPR溢出模式寄存器 | 平台特有寄存器需求 |
| eeba666256 | 8个moe CMakeLists未区分A5平台 | 新平台编译选项分支未添加 |
| 4bc16847 | cvRatio_为constexpr但运行时硬件不同 | 应改为运行时GetSubBlockNum() |
| bf74834 | 硬编码c:v=1:2假设 | c:v=1:1配置下AIV核数错误 |

### 审查检查点

1. 新平台PR: 全局搜索ASCEND_COMPUTE_UNIT/socVersion/\__CCE_AICORE\__条件，确认新平台已覆盖
2. 硬编码硬件参数: cvRatio/对齐粒度/核数不应hardcode，应通过API获取
3. 寄存器配置: 新arch适配时检查是否需要特殊SPR/控制寄存器设置

---

## API: 接口/返回值错误 (~25条, 4.7%)

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 返回值未检查/吞没 | ~8 | Check函数返回值被丢弃 |
| 函数签名不匹配 | ~6 | 参数数量/类型/顺序与定义不一致 |
| 错误码语义误用 | ~5 | return false = GRAPH_SUCCESS; LOGE后缺return |
| 接口版本兼容 | ~4 | V1/V2调用V3专属属性; 旧PTA传shape=0 |
| 文档/错误信息不一致 | ~2 | 枚举值文档遗漏; 错误日志参数名不匹配 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 0f363800 | CheckFAIIsTND()返回值完全忽略 | GRAPH_FAILED也继续返回SUCCESS |
| e7278f18 | CheckNonQuantMatmulDataType返回值被丢弃 | 非法dtype不被拦截 |
| 0c941da3 | 指针值传递应为引用 | 函数内修改无法传回调用方 |
| 7eee21c7 | extern声明缺少rotate参数 | 与实际定义签名不一致 |
| 051f878c | SetOutputDataType参数写反 | inputDataType当输出index |

### 审查检查点

1. Check函数返回值: 调用后必须检查并传播(或用\[\[nodiscard\]\])
2. LOGE后: 必须紧跟return错误码，否则错误检查无效
3. extern声明: 与实际函数定义参数列表逐一比对
4. API版本升级: V(N+1)新增参数/属性不应在V(N)接口中调用

---

## SHAPE: Shape/Layout处理错误 (~28条, 5.3%)

### 子类型分布

| 子类型 | 频次 | 说明 |
|--------|------|------|
| 新Layout分支遗漏 | ~10 | NTD/TND/BSND等新增Layout多处未覆盖 |
| Layout专属逻辑在错误Layout下执行 | ~6 | 非TND执行TND专属; BSH下不需累加batch偏移 |
| 维度索引/来源错误 | ~6 | 用batch维代替numHeads; 降维丢失信息 |
| Shape校验覆盖不全 | ~4 | InferShape fallback遗漏; 新shape组合未校验 |
| TND/NTD特殊处理 | ~2 | tSize计算、batchSize赋值、IsLastBN逻辑 |

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| b4b9198148 | 三处遗漏NTD layout处理 | dstStride/IFAMask/isGqa |
| 0166982e | BSND不需累加batch偏移但对所有layout执行 | Layout专属逻辑范围过宽 |
| a0077ae9 | GS1合轴仅处理BSNGD遗漏TNGD | 语义等价格式6处需扩展 |
| ec9278e0 | TND下tSize用batchSize*qSeqSize计算 | 应用GetTSize()获取实际值 |
| 82dbcd14 | TND布局batchSize_未赋值 | GenTilingKey永远匹配不到TND |

### 审查检查点

1. 新增Layout: 搜索所有if/switch中现有Layout枚举值的分支，逐一确认新Layout覆盖
2. Layout专属逻辑: 使用if constexpr限制只在目标Layout下执行
3. 维度取值: 不同Layout下batch/seq/head维度的位置不同，核对取值来源

---

## ALIGN: 对齐计算错误 (~18条, 3.4%)

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 3cfe2f3b64 | copyTotalS未32对齐 | NZ格式scale搬运不满足对齐要求 |
| 446d15c9 | NZ格式奇数行stride偏小 | 需2行对齐 |
| ac141d52 | axisH_非32对齐截断 | 尾部UB残留值参与计算 |
| c439c4d7 | Mmad的m维度未BLOCK_SIZE对齐 | L0C矩阵乘精度问题 |
| 73a82c36 | perTokenSize_做512对齐但不需要 | 对齐后偏大导致地址错位 |

### 审查检查点

1. DataCopy/DataCopyPad: 搬运长度是否满足32B(256bit)对齐; 不满足时用DataCopyPad
2. NZ格式: 行数维度需2行(16元素行)对齐
3. Mmad: m/n/k维度需BLOCK_SIZE对齐
4. 不必要的对齐: 对齐操作是否会导致偏移量偏大，破坏后续地址计算

---

## CPASTE: 复制粘贴/拼写/命名错误 (~18条, 3.4%)

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| 5508fbb6c9 | OP_NAME从MoeFinalizeRouting复制未改 | CMake中算子名错误 |
| a02041efea | 用valueShapeInfo.b而非.n | 从另一校验复制未改字段名 |
| e8bdf8f2 | DivCastImplGeneral调用DivCastImpl128VF | 从128路径复制未改函数名 |
| 120106d9 | L0A/L0B/L0C索引全写成ASCEND_CB | 从一行复制未改索引 |
| a350f640 | namespace FaiKenel缺字母r | FaiKernel拼写错误 |

### 审查检查点

1. 新增文件/函数: 如果是从已有文件/函数复制，逐行检查所有标识符是否已替换
2. 相似函数组(V1/V2/V3等): diff中只修改了一处时，检查平行函数是否也需同样修改
3. CMakeLists新增算子: OP_NAME与目录名/源文件名逐字核对

---

## PARAM: 参数值/顺序/魔法数字 (~25条, 4.7%)

### 典型案例

| Hash | 缺陷 | 根因 |
|------|------|------|
| e4b3ba6d | MrgSort4Info的elementLengths和srcList赋值语义相反 | API参数语义混淆 |
| 7075b6143f | rowOuterIdx/rowInnerIdx顺序与调用侧不一致 | 参数位置交换 |
| cb1c8346 | POS_INF用3e+99而非IEEE 754 infinity | 非标准无穷大表示 |
| cfe618f7 | ZERO_DIM=1(应0)、TWO_DIM=3(应2) | 常量名值不匹配 |
| fb58b43e98 | swizzlCount上界写死9(应16) | 魔数无推导来源注释 |

### 审查检查点

1. API调用: 核对参数与文档/函数声明中参数名的语义对应
2. 魔法数字: 边界值/常量需定义为命名常量并附注推导来源
3. 常量定义: 名称与值必须匹配(ZERO=0, ONE=1)
4. 函数参数>3个: 考虑是否需要结构体封装避免位置混淆

---

## REVERT: 功能回退 (~16条, 3.0%)

阶段3详细分析了35个Revert提交(含正向修复后的二次revert)，归为8个事件群。此处摘要核心发现。

### 事件群分布

| 事件群 | 次数 | 说明 |
|--------|------|------|
| TilingKey模板化整改 | 6 | 同一方案4组MC2算子连续失败 |
| 大型新算子premature merge | 10 | 29%，合入后极短时间revert |
| 跨仓迁移协调失败 | 3 | 缺乏原子性保障 |
| 性能优化引入正确性问题 | 4 | L1/L2/TSCM缓存策略类 |
| 大规模重构一次性合入 | 4 | MC2 TilingData整改(161文件)等 |
| Cleancode伪装下的语义变更 | 3 | 删错误处理、改函数签名 |
| 构建系统/CI配置错误 | 3 | cmake/yaml/python跨层改动 |
| 功能开关批量开启 | 2 | 开关开启但测试跑不通 |

### 核心审查启示

1. 架构方案首次失败后: 要求根因分析和方案修正再继续推进其他模块
2. 新增算子(>10新文件): 必须填写测试栏并提供CI通过证据
3. 缓存策略优化: 必须附带硬件实测精度数据
4. "Cleancode" PR: 区分纯格式vs语义变更(删错误处理/改签名)，后者需全面审查
5. 跨仓变更: 上下游就绪检查不能仅靠文字约定

---

## 跨类别系统性风险 (来自阶段4热点分析)

### 风险1: 代码克隆/平行实现

"修一处漏一处"是本仓23次cmake缺陷和11次MC2 kernel缺陷的根因。

| 克隆对 | 相似度 | 影响 |
|--------|--------|------|
| custom_build.cmake vs CMakeLists.txt(RTY) | 285行平行 | feature增减需两处同步 |
| func.cmake两个macro | 80% | 新include需两处同步 |
| combine_v2.h vs _add_rms_norm.h | 80% | bug修复不同步致行为不一致 |
| aclnn_grouped_matmul.cpp 6个版本入口 | 主体相似 | 新参数/check需6处同步 |
| PFA/IFA各自TilingDataconvert | 独立实现 | 新tiling字段遗漏 |

### 风险2: 跨文件隐式一致性约束

无编译期保证的跨文件约束是本仓高频缺陷源。

| 约束 | 涉及文件 | 失败频次 |
|------|---------|---------|
| tilingKey编码(host) vs 模板匹配(kernel) | tiling.cpp + kernel.h | ~20条 |
| workspace offset(host) vs 读取offset(kernel) | tiling.cpp + kernel.h | ~8条 |
| 宏参数顺序 vs 宏定义 | matmul_all_reduce.cpp等 | ~5条 |

### 风险3: 整数溢出/除零

几乎所有tiling文件和kernel文件都涉及多维度连乘或除法，普遍缺少溢出/除零检查。

- workspace计算: batch * head * seq * dim * sizeof(type) 六值连乘
- AlignUp/CeilDivision: 除数为0时静默返回0，下游使用错误参数
- 地址计算: uint32乘积在大规模场景溢出

### 风险4: 硬件平台分支不完整

新增硬件型号时多处遗漏是常见缺陷来源。

- socVersion字符串比较路由("Ascend910_95")
- `#if __CCE_AICORE__ == 220`预处理分支
- 编译选项/对齐要求/c:v比例平台差异

---

## 缺陷模式与Revert/热点的交叉验证

| 缺陷类别 | 阶段2频次 | 阶段3(Revert)佐证 | 阶段4(热点)佐证 |
|----------|----------|-------------------|----------------|
| BUILD | ~65 | 事件群7(3次CI/构建revert) | custom_build.cmake Top1(23次) |
| COND | ~75 | 事件群6(cleancode删错误处理) | tiling_v2.cpp dVBasicBlock=0未赋值 |
| BUF | ~60 | 事件群4(性能优化buffer公式) | fia_kernel_nonquant_mla.h sizeof链 |
| SYNC | ~35 | 事件群4(cross core同步) | combine_v2.h WaitDispatch浮点比较 |
| TKEY | ~20 | 事件群1(6次系统性失败) | regbase.cpp 18参数tilingKey组装 |
| PLAT | ~25 | 事件群2(新算子平台不兼容) | matmul_all_reduce.cpp #if分支 |
| CPASTE | ~18 | 事件群5(423572ba大杂烩MR) | aclnn_grouped_matmul 6版本入口 |
