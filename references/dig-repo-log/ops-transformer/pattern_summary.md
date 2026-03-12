# ops-transformer 缺陷模式归纳与分类

数据来源：defect_analysis.md(243条缺陷深度分析) + revert_analysis.md(10条Revert) + hotspot_analysis.md(20个热点文件)

缺陷总数：243条(含compound缺陷按主根因计一次)

---

## 类别总览

| 序号 | 缺陷类别 | 频次 | 占比 | 可审查性 |
|------|---------|------|------|---------|
| 1 | 计算/Tiling参数错误 | 35 | 14.4% | 高 |
| 2 | 构建/配置/部署缺陷 | 34 | 14.0% | 高 |
| 3 | 条件分支/逻辑覆盖不完整 | 30 | 12.3% | 高 |
| 4 | 输入/参数校验缺陷 | 28 | 11.5% | 高 |
| 5 | 空值/边界/特殊值处理缺失 | 25 | 10.3% | 中 |
| 6 | 硬件流水线同步缺陷 | 20 | 8.2% | 低 |
| 7 | 整数类型安全缺陷 | 18 | 7.4% | 高 |
| 8 | UT/测试代码缺陷 | 16 | 6.6% | 高 |
| 9 | API误用/接口错误 | 12 | 4.9% | 中 |
| 10 | 状态管理/赋值/传参缺陷 | 12 | 4.9% | 中 |
| 11 | Revert/回归引入 | 6 | 2.5% | 中 |
| 12 | 其他(平台兼容/文档/性能) | 7 | 2.9% | - |

注：部分compound缺陷(如"计算错误+条件分支遗漏")按主根因归入一个类别，子缺陷在对应类别中交叉引用。

---

## 类别1：计算/Tiling参数错误 (35条, 14.4%)

算子仓库中最高频的缺陷类别，覆盖workspace/buffer大小计算、地址偏移、除零、公式错误等。根本原因是Tiling-Kernel两阶段架构下，host侧计算的参数与kernel侧实际使用之间缺乏强一致性约束。

### 子模式

#### 1.1 workspace/buffer大小计算错误 (14条)
错误源头分布：维度因子遗漏(3)、字节/元素/block数单位混淆(3)、数据类型宽度错误(2)、对齐遗漏(2)、API语义混淆(1)、硬编码(1)、layout分支遗漏(1)、归约API隐式buffer需求(1)

典型案例：
- `9691bcc3` buffer大小计算单位错误+分配不足 — 对齐后的值已是字节数，再除sizeof(half)导致单位变为元素数，后续分配错误。审查点：对齐计算后的值是否被多余的类型大小除法破坏了单位一致性
- `69698404` workspace大小计算错误(类型宽度+数量因子错误) — 偏移量是int64(8字节)但按FP32_BYTES(4字节)算少一半；乘NUM_TWO(2)但实际有3组偏移量。审查点：每个乘法因子的语义是否与实际数据类型和数据组数对应
- `e233e106` tiling侧与kernel侧不一致 — tiling侧用vHeadSize，kernel侧用AlignUp(vHeadSize, BYTE_BLOCK)。审查点：workspace大小计算公式在tiling和kernel两侧是否完全一致
- `84ca56e9` CeilDiv与CeilAlign语义混淆 — 需要对齐的场景错误使用CeilDiv(得到block数)而非CeilAlign(得到对齐后字节数)。审查点：赋给buffer size的应用CeilAlign，赋给loop count的应用CeilDiv

#### 1.2 偏移量/地址计算错误 (11条)
错误源头分布：整数类型溢出/宽度不足(4)、workspace多段地址重叠(3)、GQA gSize缩放遗漏(3)、维度变量用错(1)

典型案例：
- `3ca57b44` workspace地址偏移计算遗漏 — kernel侧计算pseAlibiAddr时未考虑dsinksum占用空间，导致地址重叠数据互踩。审查点：新增条件性workspace段时检查所有下游地址计算是否累加了该段大小
- `d452dde6` + `81213d24` GQA维度缩放遗漏 — MLA/GQA路径中preTokens/nextTokens/actualSeqLengthKV未乘gSize。审查点：MLA/GQA场景中Q-KV维度交叉计算处须逐一检查gSize缩放
- `6b029d5f` GM偏移量类型溢出 — GM偏移ind使用uint32_t，progress*tileLength超过4GB时溢出。审查点：GM地址偏移计算变量不应使用32位整型

#### 1.3 除零错误 (4条)
典型案例：
- `234c1c8b` tiling计算除数为零 — maxNPerLoopForUb在Ascend950+headDim>8192时为0，4处CeilDiv触发CoreDump。审查点：所有除法运算必须确保除数非零，特别是从UB容量推导的中间变量
- `1f3291bf` 空tensor边界除零 — n2=0时用n2做除数计算dk和dv。审查点：shape解析函数中空tensor(维度为0)边界case必须被覆盖

#### 1.4 运算优先级/公式错误 (6条)
典型案例：
- `16bc59a1` CeilDiv不满足分配律 — `sum(a_i)/B`不等于`sum(CeilDiv(a_i,B))`，累积seqlen简单右移得到错误scale offset。审查点：向上取整/整数除法对求和不满足分配律
- `1ceed275` 整数除法运算符优先级 — 整数乘除左结合，`a*b*c/d`中除法最后执行截断误差最大。审查点：涉及整数除法的复合表达式必须用括号显式标注运算顺序

### 审查检查点
- [ ] workspace/buffer大小计算公式在tiling(host)和kernel两侧是否完全一致？
- [ ] 每个乘法因子的语义(维度数 x 类型宽度 x 组数)是否正确？
- [ ] 对齐后的值单位是字节还是元素？是否被后续计算破坏？
- [ ] CeilDiv(分块数)和CeilAlign(对齐后大小)是否用对了？
- [ ] 新增workspace段后，所有下游偏移地址是否都加上了新段大小？
- [ ] GM偏移量变量是否使用64位整型？
- [ ] 除数是否可能为零(特别是从shape/UB容量推导的中间变量)？
- [ ] GQA场景中Q-KV交叉计算是否正确应用gSize缩放？
- [ ] 整数除法的运算顺序是否可能导致截断误差放大？

---

## 类别2：构建/配置/部署缺陷 (34条, 14.0%)

非功能性缺陷中占比最大。核心矛盾是构建系统(CMake)的复杂度随算子数量增长而爆炸，加上experimental/开源/闭源多种构建模式切换，配置遗漏几乎不可避免。

### 子模式

#### 2.1 CMake配置遗漏 (18条)
新算子集成时CMakeLists.txt缺少add_subdirectory、list(APPEND)、依赖声明、源文件引用等。

典型案例：
- `80329f3a` 新算子集成遗漏 — CMake缺少依赖声明、kernel文件命名不符约定、CI列表未包含。算子根本没被编译进产物
- `a54af6d2e` CMake宏参数缺失 — add_modules_sources()缺少OPTYPE/ACLNNTYPE参数，编译不报错但运行时dlopen undefined symbol
- `f05c0242` CMake引用已删除文件 — CMakeLists.txt引用4个已不存在的.cpp文件

#### 2.2 构建模式/路径适配 (10条)
ENABLE_EXPERIMENTAL模式、开源/闭源路径、out-of-source构建等场景下路径拼接错误。

典型案例：
- `fbade778b` + `b0bf9fa2` experimental模式路径缺前缀 — 同一函数连续两次修复，op_add_depend_directory中EXISTS检查路径不带experimental/前缀
- `ef22ac8f1` include路径顺序错误 — -I路径顺序导致自定义头文件被系统内置覆盖

#### 2.3 构建脚本/打包/安装 (6条)
典型案例：
- `83be205f` 覆盖安装误删 — entity="true"导致安装时清除目录下其他组件文件
- `7d1f015a` + `7ed8fea3` build.sh参数解析缺陷 — --noexec缺少shift、-j参数不支持-j8紧跟数字格式

### 审查检查点
- [ ] 新算子是否在CMakeLists.txt中添加了add_subdirectory和list(APPEND)？
- [ ] CI的operator_list.yaml或等价配置是否包含新算子？
- [ ] CMake宏调用的必选参数(OPTYPE/ACLNNTYPE等)是否完整？
- [ ] ENABLE_EXPERIMENTAL模式下路径拼接是否正确加了前缀？
- [ ] include路径的-I顺序是否正确(项目路径优先于系统路径)？
- [ ] 安装/卸载脚本是否会影响其他组件的文件？

---

## 类别3：条件分支/逻辑覆盖不完整 (30条, 12.3%)

算子仓库特有的高频模式。根因是多layout(BNSD/BSND/TND/NTD) x 多模式(GQA/MHA/MQA) x 多量化(FP8/INT8/Perblock) x 多sparse mode的组合空间爆炸，开发者通常只验证常见组合。

### 子模式

#### 3.1 layout分支遗漏 (9条)
新增layout类型时未在所有layout判断分支做系统排查，NTD是最容易被遗漏的layout。

典型案例：
- `ad627275f` NTD layout多处遗漏 — SoftmaxLseCopyOut中Lse输出stride的GQA处理仅在TND生效，遗漏NTD；CheckIO中NTD不应走IFA路径
- `e5dbe5b5c` 双缓冲分支逻辑错误 — bias Add被放在第二个DB分支统一处理，但各DB的invalid row判断是独立的
- `fcff1be7` workspace大小计算遗漏TND分支 — TND layout下`dKeyIndexFloatSzie`用`bSize*s2Size`但TND格式token维度是T维度

#### 3.2 模式组合未覆盖 (6条)
多维枚举组合中遗漏"边角组合"。

典型案例：
- `8a852918` GQA模式组合未覆盖 — if-else只处理hasRope+Aligned576+非BNSD和默认，缺少GQA+s1Size==1+非BNSD
- `b48e75b7c` 模板分支未覆盖所有参数组合 — BIAS_CAST的true/false两分支中bType固定NZ，未考虑ND2NZ_OPT为false的4种组合

#### 3.3 条件守卫缺失 (7条)
可选输入/输出的null检查、tpWorldSize==1退化场景、不满足对齐条件时的fallback路径缺失。

典型案例：
- `a75a92e2` fallback路径缺失 — orgMValue不满足PERBLOCK_SIZE*rankDim整数倍时仍走公式化切分路径
- `f91e8dee` 多平台分支逻辑错误 — kernel中alltoallOut数据拷贝缺少isAlltoallOut条件判断，直接操作可能不存在的输出

#### 3.4 布尔逻辑错误 (4条)
循环内布尔赋值覆盖(=应为|=)、指针非空检查与布尔标志语义混淆、短路求值顺序不当。

典型案例：
- `f5f79a3e` 循环内布尔覆盖 — `isSeqExistZero = (qLen == 0 || kvLen == 0)`在循环中每次覆盖，应用`|=`
- `6ed5ed33` 短路求值顺序错误 — `CheckIsEnabledActive(gmmParams) || isNoActivation`，应将轻量无副作用条件放前面

#### 3.5 其他逻辑缺陷 (4条)
- `b9d7361` tiling key路由优先级错误 — isA8W4FakeA8W8_判断被放在isA8W8_分支后通过`||`合并
- `befd8bcd` 占位符代码未实现 — CheckGqaConstrain和CheckMlaConstrain函数体仅`return false`

### 审查检查点
- [ ] 引入新layout类型时，是否全局搜索了所有layout判断分支(包括workspace、stride、offset、InferShape)？
- [ ] 多模式组合(GQA/MHA x BNSD/TND x 量化/非量化)是否系统枚举了所有有效组合？
- [ ] 可选输入/输出在kernel和host是否都有null/flag守卫？
- [ ] tpWorldSize==1时是否需要豁免通信操作？
- [ ] 不满足对齐/整除条件时是否有fallback路径？
- [ ] 循环内对布尔标志的赋值是`=`还是`|=`？语义是"最后一次"还是"存在某条件"？
- [ ] 多个互斥条件分支是否使用if-else if而非独立if？更特化路径是否在前？
- [ ] 短路求值中，轻量无副作用条件是否放在前面？

---

## 类别4：输入/参数校验缺陷 (28条, 11.5%)

包含校验缺失(未拦截不合法输入)和过度校验(拦截了合法输入)两个对立方向，核心矛盾是算子支持的feature组合快速增长时校验代码未同步更新。

### 子模式

#### 4.1 校验缺失 (16条)
不支持的dtype/shape/参数组合穿透校验层进入计算，产生静默错误。

典型案例：
- `8ceb72b6` dtype白名单缺失 — x或weight的dtype不在支持列表时未拦截，穿透到组合逻辑产生未定义行为
- `f8882f78` 不支持dtype静默通过 — float8_e5m2未在支持列表中但未拦截，静默走默认路径产生错误结果
- `f314395a` 多芯片场景参数校验未解耦 — 多芯片场景下参数校验混在一起，单芯片校验逻辑覆盖了多芯片特有约束

#### 4.2 过度校验/过度拦截 (5条)
合法输入被校验拒绝，常见于空tensor(N=0)和新增模式扩展后旧校验条件未放宽。

典型案例：
- `54692072` 空tensor过度拦截 — 伪量化场景N=0空tensor合法但`<=0`拦截了`==0`
- `38c7162c` 合法输入被拒 — 校验条件比API规格更严格
- `aa9faa8c` 校验范围过大 — 白名单不完整导致部分合法配置被拒

#### 4.3 校验逻辑错误 (7条)
校验顺序依赖、条件取反、语义混淆等。

典型案例：
- `44e84d52` 校验顺序错误 — 空tensor检查在BatchDim校验之后，空tensor的维度值检查无意义且会误判
- `9b12d59b` 校验宏语义混淆 — CHECK_COND_RETURN宏的条件取反与直觉相反
- `44338c14` 量化模式互斥校验错误 — 独立if块覆盖了DYN_PERTOKEN_QUANT的特殊处理，应用else if

### 审查检查点
- [ ] 算子入口是否有全量dtype白名单校验(而非仅在子分支内局部检查)？
- [ ] 空tensor(维度=0)是否作为合法输入被正确处理而非过度拦截？
- [ ] 校验条件是`<0`还是`<=0`？0是否是合法值？
- [ ] 校验函数的执行顺序是否满足逻辑依赖(如空tensor判断须在维度值判断之前)？
- [ ] 新增feature/mode后，现有校验路径是否需要放宽或增加豁免？
- [ ] CHECK宏的条件语义(true=通过 vs true=拦截)是否清晰一致？

---

## 类别5：空值/边界/特殊值处理缺失 (25条, 10.3%)

与类别4(输入校验)的区别：类别4是API边界处的参数合法性检查，类别5是计算逻辑内部对极端值的处理。空tensor问题跨越两个类别(校验层+处理层)。

### 子模式

#### 5.1 空tensor/零维度场景 (10条)
空tensor问题不是单层能解决的，必须aclnn/infershape/tiling/kernel全链路配套处理。

典型案例：
- `b5d7c0fe` 空tensor多层联动缺失 — BSH/SBH layout下dDim=0直接报错但空tensor合法；InferShape输出shape未赋值；kernel输出含脏数据。跨4层联动修复
- `236a5fdb` GMM空tensor三层防御 — K=0进入matmul tiling触发除零。修复：kernel跳过+API校验+tiling标志位
- `78e07300` 空tensor输出未初始化 — 判断不精确+输出未ZerosLike初始化

#### 5.2 空指针解引用 (5条)
典型案例：
- `d3aa4960` 空指针格式化崩溃 — commModePtr为nullptr时OP_LOGE用%s格式化导致崩溃
- `9aa05a4b` 场景未区分导致空指针 — A4W4场景weightAssistMatrix应为nullptr但被无条件解引用

#### 5.3 尾块/边界条件 (5条)
典型案例：
- `b59cf016` 尾块边界处理错误 — singleN未考虑lastBlockSize，MM参数超出有效范围
- `ccd9bc9b` 负值未clamp — actualS1Size减法后可能为负但未做下界保护

#### 5.4 特殊值(inf/NaN) (2条)
典型案例：
- `4801ecb2` inf输入未处理 — 多SP域场景lse为+inf，exp(+inf - +inf) = NaN污染输出。修复：检测+inf替换为-inf

#### 5.5 无符号整数下溢 (3条)
从整数安全角度看也属于类别7，但核心触发条件是"边界场景"(尾核无数据)。

典型案例：
- `99a876f9` uint32减法下溢 — aivIdx*singleCoreSize > totalOutputSize时uint32_t回绕为极大值，该核在错误GM偏移写入
- `b4baa0d2` 多核分核6处下溢 — SplitCore中total-coreIdx*perCoreSize形式在尾核普遍触发

### 审查检查点
- [ ] 空tensor(维度=0)是否在aclnn/infershape/tiling/kernel四层全链路处理？
- [ ] 可选输入指针使用前是否有nullptr守卫？OP_LOGE中%s是否可能传入nullptr？
- [ ] 尾块处理中，最后一块的大小是否min(remainSize, blockSize)？
- [ ] exp/log/div/sqrt等数学运算是否处理了0/+inf/-inf/NaN边界？
- [ ] 多核分核场景中uint减法`total - coreIdx * perCore`是否有下溢保护？

---

## 类别6：硬件流水线同步缺陷 (20条, 8.2%)

AscendC NPU特有的高危缺陷类别。可审查性最低——需要理解MTE2/MTE3/Vector/Scalar四条流水线的数据依赖关系。但模式高度可归纳。

### 子模式

#### 6.1 缺失同步barrier (13条)
最高频模式。典型场景：Vector/MTE2/MTE3/Scalar流水线切换时遗漏对应的PipeBarrier或SyncFunc。最常见遗漏点是DataCopy/DataCopyPad前后。

典型案例：
- `984d8a72` Vector到MTE2同步缺失 — RoPE切换处理query到key时，DataCopy前只调了MTE3ToVSync()缺少VToMTE2Sync()，Vector可能仍在处理query导致数据竞争
- `368c7325` V_MTE3同步位置错误 — Cast后DataCopyPad写GM前缺V_MTE3同步；同步放在Div后而非Cast后
- `d1d95dfb` 多流水共享UB竞争 — matmul和通信都使用vec单元且共享UB，缺SyncAll<false>()

#### 6.2 同步方向/ID错误 (4条)
更隐蔽的模式，无法通过简单"有无同步"检查发现。

典型案例：
- `662f162c` 同步方向反转 — SyncFunc<V_MTE2>()语义是"V完成后启MTE2"，但实际需要"MTE2完成后做V"。方向完全反
- `5644a7b` 同步ID冲突 — super-kernel场景下MlaProlog同步常量(0x0/0x1)与前序算子冲突
- `a58c8a0b9` event信号积压 — 两个if分支分别SetFlag不同event，但WaitFlag只等了其中一个，另一个信号未消费导致死锁

#### 6.3 跨迭代/全局同步 (3条)
涉及SyncAll模板参数语义、SetScheduleMode时序、同步优化引入正确性退化。

典型案例：
- `5dce387d` SyncAll模板参数错误 — SyncAll<true>()和SyncAll<false>()语义不同，用错导致同步不充分
- `65cafae0` SetScheduleMode时序遗漏 — 使用SyncAll但部分路径缺少BATCH_MODE_SCHEDULE设置导致多流死锁

### 审查检查点
- [ ] DataCopy/DataCopyPad操作前后是否有对应的pipeline barrier？(V->MTE2: VToMTE2Sync; V->MTE3: PipeBarrier<PIPE_V>; MTE3完成后: PipeBarrier<PIPE_MTE3>)
- [ ] 切换处理不同tensor并复用同一块local memory时，是否补全了所有必要sync？
- [ ] SyncFunc模板参数的方向(A_B表示"A完成后B可以开始")是否与数据流一致？
- [ ] 同步ID/flag常量值是否全局唯一(特别是super-kernel场景)？
- [ ] SetFlag和WaitFlag是否严格配对？有无未消费的event信号？
- [ ] 使用SyncAll的kernel，host tiling是否设置了SetScheduleMode(BATCH_MODE_SCHEDULE)？
- [ ] SyncAll<true>和SyncAll<false>的语义差异是否被正确理解？
- [ ] 循环体中DMA操作的迭代间同步是否充分？

---

## 类别7：整数类型安全缺陷 (18条, 7.4%)

三种系统性子模式：无符号减法下溢、窄类型(uint16_t)截断、unsigned/signed混淆。每种都有固定的检测和修复方法。

### 子模式

#### 7.1 无符号整数下溢 (5条)
`a - b` 当 `a < b` 时uint32_t/uint64_t回绕为极大正数。

典型案例：
- `99a876f9` 多核分核下溢 — `totalOutputSize - aivIdx * singleCoreSize`在尾核下溢为接近2^32，写入错误GM偏移触发硬件异常
- `b4baa0d2` 系统性下溢 — SplitCore函数中6处uint32_t减法下溢

修复模式固定：`a > b ? a - b : 0U`

#### 7.2 窄类型截断 (5条)
uint16_t最大值65535，DataCopy参数(blockLen/stride)和repeat参数在大数据量时截断。

典型案例：
- `e13d1ec19` DataCopy长度截断 — cpInLen/cpOutLen声明为uint16_t，size超过32767个bfloat16元素时截断
- `e96b7b2a` repeat参数溢出 — 硬件指令repeat参数uint16在大数据量时溢出

修复模式：中间变量扩宽到uint32_t + 切换到Ext版本API

#### 7.3 unsigned/signed混淆 (3条)
框架API用int64_t(-1表示动态shape)，结构体字段用uint64_t导致-1变为极大正数。

典型案例：
- `78282635` uint64_t与int64_t不匹配 — QuantAllReduceShapeInfo字段为uint64_t，-1(动态shape)隐式转为18446744073709551615

修复模式：字段类型与API类型对齐

#### 7.4 乘法溢出 (5条)
uint32_t乘法在offset/size计算中溢出。

典型案例：
- `831ab170` uint32乘法溢出 — `s1BottomTok * params.gSize`溢出uint32_t
- `6b029d5f` GM偏移溢出 — `progress * tileLength`超过4GB时溢出

修复模式：提升为uint64_t

### 审查检查点
- [ ] uint减法`a - b`：a是否可能小于b？是否需要安全减法`a > b ? a - b : 0U`？
- [ ] DataCopy参数(blockLen/stride)赋值是否可能超过uint16_t(65535)？是否应使用Ext版本API？
- [ ] 结构体字段类型是否与上游API的类型一致(int64_t vs uint64_t)？
- [ ] offset/size计算中间结果是否可能超过uint32_t(4GB)？是否需要uint64_t？

---

## 类别8：UT/测试代码缺陷 (16条, 6.6%)

测试代码本身的缺陷，而非被测production代码的缺陷。核心问题是UT编写质量标准低于production代码。

### 子模式

#### 8.1 数据类型不匹配 (6条)
UT中host数据类型与aclDataType不一致，或sizeof用错类型。

典型案例：
- `b9a02e9d` 多处类型不匹配 — gen_data.py中position用float而非int64；test_case_fp32中sizeof(half)应为sizeof(float)导致内存只分配一半
- `a911707e` host与acl类型不一致 — host用float但声明ACL_FLOAT16

#### 8.2 路径/依赖错误 (5条)
UT include引用闭源路径、CMake路径错误等。

典型案例：
- `29b08607` 闭源路径依赖 — include引用level2/路径，system()拷贝闭源仓库脚本。开源仓库无法编译
- `79cd3244` CMake路径错误 — GLOB路径缺前缀+多余反斜杠+模板参数缺失

#### 8.3 UT与production不同步 (3条)
头文件名、命名空间、结构体名变更后UT未更新。

典型案例：
- `8d72712c` UT落后于production — 头文件路径/命名空间/期望输出字符串全部过期

#### 8.4 tiling数据未初始化 (2条)
典型案例：
- `f1a24bd` GmAlloc后未赋值 — tiling结构体21个字段未初始化导致随机值

### 审查检查点
- [ ] UT中CreateAclTensor的dataType与host数据的C++类型是否匹配？
- [ ] sizeof()参数的类型是否与实际测试数据类型一致？
- [ ] UT的include路径是否使用相对路径(而非闭源绝对路径)？
- [ ] UT中tiling结构体的所有字段是否初始化？
- [ ] 对production代码重命名/重构后，对应UT是否同步更新？

---

## 类别9：API误用/接口错误 (12条, 4.9%)

使用了错误的API或传递了错误的参数，通常因为API名称相似或语义不够直观。

### 子模式

#### 9.1 相似API混淆 (5条)
- `73c4fdaa` + `d0ede159b` + `a7010fc9` GetInputShape vs GetOptionalInputShape — 对可选输入使用了必选输入的访问接口，可选输入不存在时崩溃
- `84ca56e9` CeilDiv vs CeilAlign — 需要字节对齐的场景用了分块计算的API
- `c2250fc9` Size() vs GetShapeSize() — 非连续tensor的Size()返回包含stride的逻辑大小

#### 9.2 参数传递错误 (4条)
- `6978efa7` 参数顺序反转 — INFER_SHAPE注册时参数顺序与函数签名不匹配
- `ac0860f7` 参数结构体类型错误 — DataCopyExtParams被误用于DataCopyPad
- `e40a4176` Attr注册顺序不匹配 — 与aclnn接口参数顺序不一致

#### 9.3 返回值/格式化错误 (3条)
- `c2d880ba` API返回值类型错误
- `99342fa4` logging格式占位符缺失
- `bfbbec83` 底层intrinsics接口误用

### 审查检查点
- [ ] 可选输入是否使用GetOptionalInputShape/GetOptionalInputTensor而非Get*Input*？
- [ ] CeilDiv(分块数)和CeilAlign(对齐大小)是否用对？
- [ ] 接口注册(INFER_SHAPE等)的参数顺序是否与函数签名匹配？
- [ ] DataCopy系列API的参数结构体类型是否正确(DataCopyParams vs DataCopyExtParams vs DataCopyPadParams)？

---

## 类别10：状态管理/赋值/传参缺陷 (12条, 4.9%)

### 子模式

#### 10.1 赋值遗漏/结构体字段遗漏 (5条)
- `8e9fe171` 结构体字段遗漏 — 数据传递链路中某个字段未赋值，下游读到未初始化值
- `312d30359` 硬件对齐未满足 — tiling结构体未满足64位对齐要求
- `64be6dd6` 复制粘贴错误 — 粘贴后变量名未全部替换
- `90cc4df64` SetBlockDim缺失 — aicpu task缺少块维度配置

#### 10.2 变量名错误 (3条)
- `b96fd79d` typo — consInfo应为constInfo
- `6e16f3dc` 头文件名拼写 — 缺少ed后缀
- `69f8dff2` 重命名不同步 — 函数重命名后调用点未更新

#### 10.3 变量遮蔽/作用域 (2条)
- `a882f8c0` variable shadowing — 内层变量遮蔽外层同名变量
- `685562cc` 作用域错误+条件判断对象混淆

#### 10.4 参数传递错误 (2条)
- `64dedf37` 语义混淆 — 分块处理量vs总大小传错参数
- `372b8e80` key名传错 — KEY_NAME应为KEY_SHARED_PREFIX_NAME

### 审查检查点
- [ ] 新增struct字段后，所有初始化/赋值路径是否都设置了该字段？
- [ ] 复制粘贴的代码块中，变量名是否全部替换完毕？
- [ ] 函数/变量重命名后，是否全局搜索确认所有调用点同步更新？
- [ ] 内层作用域是否存在与外层同名的变量遮蔽？

---

## 类别11：Revert/回归引入 (6条, 2.5%)

被Revert的提交暴露了代码质量流程中的系统性缺陷。

### 关键发现

6个独立Revert事件的触发原因分布：
- 编译/构建问题 (3): 头文件依赖不完整、模板编译超时、框架API不可用
- 并发/同步正确性 (2): dispatch syncall优化引入数据竞争、fullmesh状态管理
- 示例代码集成 (1): 图模式GE IR依赖不适用于开源仓

### 系统性模式

1. 大型变更缺乏分阶段策略 — ROPEV2(27文件/3000行)和tilingkey模板化(32文件/20000行)都一次性提交，回退代价极高
2. CI门禁"合入后才发现" — cann-robot自动回退属于post-merge拦截，成本远高于pre-merge
3. 功能被revert后最终重新引入 — tilingkey模板化和schedule mode都在改进后重新合入，问题是实现质量而非功能方向

### 审查检查点
- [ ] 大型变更(>10文件/1000行)是否可以分阶段提交？
- [ ] 同步原语的"优化"是否证明了等价性(而非依赖CI验证)？
- [ ] 依赖的框架API是否已稳定发布(而非内部staging版本)？

---

## 类别12：其他 (7条, 2.9%)

#### 硬件平台兼容性 (3条)
- `4d24f605` 310P三类问题(内存对齐+workspace偏移+废弃API)
- `c04bfe85` 架构标识符错误(Arch::DAV_3510应为Arch::Ascend950)
- `66b12bf2` constexpr误用(编译态硬件参数运行时可能不同)

#### 文档/示例 (2条)
- `4e4f9168` 文档约束公式括号遗漏
- `08fbc029` 脚本硬编码sudo

#### 性能回退 (1条)
- `c25b4a43` 变量存储位置不当导致间接寻址开销

#### 操作失误 (1条)
- `e2ba8a5d` Git merge冲突标记残留

---

## 跨类别系统性风险

### 风险1：GQA gSize缩放因子遗漏 (跨类别1/3)
跨6条独立commit反复出现(d452dde6, 81213d24, 8a852918, 3bb75678, e44028b1, 16bc59a1)。是本仓库最频繁的单一具体根因。Q和KV的head数不同时，所有涉及Q-KV维度交叉计算的地方都需要gSize缩放。

### 风险2：空tensor全链路处理 (跨类别4/5)
空tensor问题跨越API校验层(类别4)和内部处理层(类别5)。必须aclnn/infershape/tiling/kernel四层联动，任何一层遗漏都会导致问题。`<=0` vs `<0`的校验边界差异是最常见的具体表现。

### 风险3：host(tiling)侧与kernel侧不一致 (跨类别1/10)
workspace大小、buffer对齐、struct类型均出现tiling/kernel两侧独立计算不匹配。根因是两侧代码物理分离(不同目录/不同编译单元)缺乏编译期一致性约束。

### 风险4：大型Tiling类的状态累积 (跨类别10/热点分析)
prompt_flash_attention_tiling_v2.cpp有80+成员变量无统一reset机制，不同tiling路径可能读到上一次计算的残留状态。hotspot分析中排名第一的结构性风险。

### 风险5：构建模式组合爆炸 (跨类别2/11)
ENABLE_EXPERIMENTAL、开源/闭源、多平台(310P/910B/Ascend950)组合下，CMake路径/条件编译/依赖关系难以维护。是revert的首要触发因素。

---

## 缺陷分布可视化

### 按模块分布(基于defect_stats.md)
```
attention  ████████████████████████████████████  42.3%  (103条)
mc2        ████████████████                      18.5%  (45条)
moe        ██████████████                        13.3%  (32条)
gmm        ████████                               8.6%  (21条)
构建/配置   ██████████████                        14.0%  (34条)
其他        ███                                    3.3%  (8条)
```

### 按代码层分布(基于defect_stats.md)
```
op_host/tiling  ████████████████████████████████████  42.9%
op_kernel       ██████████████████████████            33.3%
aclnn/infershape ████████                              9.5%
构建/测试配置     ████████████                          14.3%
```

### 按严重程度分布(基于可审查性和影响)
```
P0-硬件异常/数据损坏   ████████████████████████  (同步缺失+内存越界+除零)  ~45条
P1-特定配置崩溃        ████████████████          (条件分支遗漏+空指针)      ~35条
P2-功能错误/静默错误    ████████████████████████  (计算错误+校验缺失)        ~55条
P3-构建/测试/可维护性   ████████████████████████████  (构建配置+UT+状态管理) ~65条
```
