# ops-nn Revert专项分析

共发现5个Revert提交，对应4个独立Revert事件。

## 事件总览

| # | Revert Hash | 原始提交 | 主题 | Revert间隔 | 根因类别 |
|---|------------|---------|------|-----------|---------|
| 1 | d91e1757a | 1962c3991 | unsorted_segment_sum算子迁移Ascend950 | ~30小时 | CI/集成失败 + 搭车提交 |
| 2 | 852f21d6b | ecd59361a | batchmatmul AB矩阵非连续输入 | 1天 | 变量遮蔽 + 空指针检查缺失 |
| 3 | 4482238c0 | 08b547c40 | LogSigmoid算子Ascend950实现 | ~13小时 | 目录命名/构建配置问题 |
| 4 | e248762f7 / a39e18548 | (init导入) | MatmulV3 small shape优化 | N/A(init即存在) | workspace脏数据(并发竞争) |

---

## 事件1: unsorted_segment_sum算子迁移

- Revert: `d91e1757ab41c3620057477c28adc62d395c2e00` (2026-03-07, cann-robot自动)
- 原始: `1962c3991dde8c03f1fd1e414583b94ae07c1a1a` (2026-03-06, Huang-Peng)
- MR: 原始 !1460, Revert !2421 (revert-mr-1460-auto)

### 原始提交内容

新增UnsortedSegmentSum算子对Ascend950的支持，38个文件/8892行新增。包含算子定义、infershape、8种tiling策略(SIMT/SIMD/Sort/Deterministic等)、kernel实现、编译配置(binary.json 1769行)、测试。

### Revert原因

cann-robot自动执行，分支名含`auto`后缀，大概率CI流水线检测到编译/集成测试失败后自动revert。原始MR仅声称"已通过冒烟测试"，在更全面的集成测试中失败。

### 关键发现: 搭车提交

原始提交在`docs/zh/op_list.md`中添加的不是unsorted_segment_sum条目，而是scatter算子条目。这是典型的"搭车提交"反模式——不相关改动混入同一MR。Revert时scatter条目也被一并删除（附带伤害）。同时，unsorted_segment_sum自身的op_list条目反而遗漏未添加。

### Code Review可发现性: 高

1. 不相关改动混入(scatter文档搭车) — 违反单一职责，reviewer应要求拆分
2. 自身算子文档遗漏 — 该添加的没添加，不该添加的却添加了
3. 单次提交8892行 — 体量过大应拆分
4. 测试覆盖声明不足 — 7种数据类型x8种tiling策略，仅声称"冒烟测试"通过

---

## 事件2: batchmatmul AB矩阵非连续输入

- Revert: `852f21d6ba12a54de4f47f30fb5337ebb67473ae` (2026-02-14, szhexin)
- 原始: `ecd59361a980ebfb6d3c09d76b05edaf26d8d783` (2026-02-13, shangzhexin)
- 修复重提交: `da61cacb9503d4f34f221a9576fc0600888368ac` (2026-02-28)
- Issue: 原始 #864, Revert #1088, 修复 #1119

### 原始提交内容

为BatchMatMul算子增加A矩阵非连续输入支持（原先只支持B矩阵非连续）。引入NonContiguousMode枚举，修改tiling/kernel/op_api三层，18个文件/422行新增。在MatMulV3BasicTilingData结构体末尾新增innerBatch字段。

### Revert原因: 两个代码缺陷

1. 变量遮蔽bug(variable shadowing): `ExecBatchMatmulOpWithBiasAndAttrsV2`函数中，AB_NON_CONTINUOUS分支的else路径写了`auto selfCast = l0op::Cast(...)`，用auto重新声明了局部变量，遮蔽了外部作用域的同名变量。结果: Cast操作结果被丢弃，后续使用了未经类型转换的tensor。同样问题出现在selfTransdata。

2. CreateView空指针检查缺失: 多处CreateView调用后缺少`CHECK_RET(... != nullptr, nullptr)`检查，CreateView失败时后续代码对nullptr调用方法导致crash。

这两个bug的组合导致: 不仅新增的非连续场景出错，连续输入的正常计算路径也受影响（selfCast跳过了Cast和TransData），产生计算结果错误或运行时崩溃。

### Code Review可发现性: 高

1. `auto selfCast = ...`在已有同名变量的作用域中 — C++经典shadowing错误，开启`-Wshadow`即可自动捕获
2. CreateView后缺少空指针检查 — 同文件其他调用都有CHECK_RET，对比即可发现不一致
3. 跨三层(op_api/tiling/kernel)的变更 + tiling数据结构新增字段 — 级联影响大，是review风险信号
4. `static_cast<int32_t>`比较枚举值 — 脆弱且不直观的做法

---

## 事件3: LogSigmoid算子实现

- Revert: `4482238c056b34c004de679244ef183176e854d1` (2026-02-11 23:04, cann-robot)
- 原始: `08b547c40d976cf4b7fe17ed2478eb20fc26ed0e` (2026-02-11 09:45, huangyuxiaaaaa)
- 修复重提交: `9140e706d57e7c2c78dd41e7a77fd32724bf87e6` (2026-02-12)
- 完整cycle: 提交->revert->修复重提交，不到48小时

### 原始提交内容

为LogSigmoid算子新增Ascend950 kernel实现。同时将算子目录从logsigmoid(无下划线)重命名为log_sigmoid(有下划线)，38个文件/+1190行。

### Revert原因: 构建/命名/搭车问题

1. 目录重命名破坏性变更: `logsigmoid` -> `log_sigmoid`，破坏了既有的aclnn接口和UT的路径依赖
2. 目录结构不规范: op_api目录层级位置不符合仓库标准
3. CMakeLists配置问题: `ACLNNTYPE aclnn_exclude`排除了aclnn构建，暗示集成有问题
4. 搭车改动: 修改了不相关的`aclnn_binary_cross_entropy_with_logits_target_backward.cpp`中的平台判断逻辑(从架构判断改为SoC版本范围判断)
5. cann-robot自动revert，CI/门禁检查未通过

### Code Review可发现性: 高

1. 目录重命名是破坏性变更 — 应要求提交者说明理由并确认全量CI通过
2. `ACLNNTYPE aclnn_exclude` — 新增kernel实现却排除aclnn构建，明显的配置异常
3. 不相关模块的改动(binary_cross_entropy) — 应要求拆分为独立MR
4. 修复版减少了~320行代码(871 vs 1190) — 说明原始提交包含不必要的内容

---

## 事件4: MatmulV3 small shape优化

- Revert: `e248762f7` (8.5.0分支) + `a39e18548` (master分支), 2026-02-05, llqx-1
- 原始: 从`e11fe07f3 init`(2025-09-28)导入，无独立引入提交
- MR: !1540 (8.5.0), !1553 (master)

两个revert提交是同一修复在不同分支上的同步cherry-pick，diff内容完全一致。

### 被revert的优化内容

在`GetMixNd2nzType()`函数中(matmul_v3_base_tiling.cpp)，在V_HEAD_ND2NZ分支前插入了更高优先级的判断：当满足(仅A矩阵需要nd2nz + 非对齐处理 + BL1满载 + 基础分核 + 基础FixOpti)时，选择`V_PARALELL_ND2NZ`模板——让vector和cube管道并行执行ND到NZ格式转换，以提升小shape矩阵乘法性能。

### Revert原因: workspace脏数据

commit message明确说明:

> 回退小shape场景下走入MatmulV3特殊优化模板，该模板可能导致出现workspace脏数据参与计算的问题。

V_PARALELL_ND2NZ模板让vector和cube并行操作workspace时，workspace buffer中可能残留上一次计算的脏数据。未被正确清零/覆盖的workspace数据被cube管道读取并参与矩阵乘法计算，导致计算结果精度错误。

UT测试中tiling_key从131074(0x20002，高位编码V_PARALELL_ND2NZ)回退到2(V_HEAD_ND2NZ)。

### Code Review可发现性: 中

能被发现的线索:
- 引入并行执行路径时应追问workspace生命周期和初始化保证
- `unAlignProcessType > 0`(非对齐)是危险信号 — 非对齐场景buffer边界处理复杂，workspace可能有未写入的padding区域
- "非对齐 + 并行ND2NZ + workspace复用"是高风险组合

难以发现的因素:
- 问题根因在kernel侧vector/cube管道的执行时序，仅从host侧tiling代码难以看出
- 特定shape组合才触发，难以穷举边界case
- 需要对Ascend硬件pipeline交互有深入理解

---

## 跨事件模式总结

### 模式1: 搭车提交 (事件1, 3)

两个事件都在主功能MR中混入了不相关改动:
- 事件1: unsorted_segment_sum的MR里加了scatter的文档条目
- 事件3: LogSigmoid的MR里改了binary_cross_entropy的平台判断

搭车提交的危害: Revert时不相关改动被一并回滚(附带伤害)，且增加了review难度。

审查规则: 检查MR中每个文件的改动是否与标题描述的功能直接相关。

### 模式2: 变量遮蔽/初始化类缺陷 (事件2, 4)

- 事件2: auto重新声明导致外部同名变量被遮蔽，Cast结果丢失
- 事件4: workspace未正确初始化/清零，脏数据参与计算

两者都是"数据不是预期的值"类问题，一个在编译期可检测(-Wshadow)，一个在运行时才暴露。

### 模式3: 大规模提交缺乏充分测试 (事件1, 2, 3)

- 事件1: 8892行，仅声称"冒烟测试"通过
- 事件2: 跨三层18文件改动，隔天revert
- 事件3: 1190行38文件，当天revert

大体量提交应拆分为可增量review的小MR，且需要比冒烟测试更充分的验证。

### 模式4: 自动化revert机制 (事件1, 3)

cann-robot自动执行revert，分支名含`auto`后缀。说明CI/门禁流水线有自动revert能力——这是好的实践，但也意味着提交者过度依赖CI而非提交前充分自测。

### 缺陷严重度排序

1. 事件2(变量遮蔽) — 最严重: 影响正常路径计算正确性，可导致crash
2. 事件4(workspace脏数据) — 严重: 静默精度错误，难以定位
3. 事件1(集成失败) — 中等: CI拦截有效，但搭车提交造成附带伤害
4. 事件3(构建配置) — 中等: 48小时内修复重提交，影响范围有限
