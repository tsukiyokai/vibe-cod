# hcomm-dev Revert专项分析

共4条Revert提交，涉及3个独立回退事件（Revert#2和Revert#3属于同一runtime接口迁移系列）。

---

## Revert#1: a37e6cf1 — Revert HB check opinconsistent

| 属性 | 值 |
|------|-----|
| Revert hash | a37e6cf15c75b6db581a07f433eeea45bf3c21f7 |
| 日期 | 2025-12-03 |
| 作者 | dingzhiqiang |
| 原始提交 | b51f8cc4（2025-11-18, yanyefeng, "update"） |
| 涉及文件 | 7个（heartbeat.cc/h, hccl_communicator_host.cc等） |

### 原始变更内容

为心跳模块引入算子不一致性检测(Op Inconsistency Check)功能：

1. 重构心跳帧数据结构：从`OpInfoDesc opInfoList[16]`扁平数组改为按tag分组的两层嵌套结构`OpInfoTagQueueFrame`（内含`OpInfoTagQueue[10]`，每个tag下可存`OpInfoDesc[500]`）
2. 新增`inconsistentOpMap_`和`srTagMap_`两个map，增加`RegisterSROpIdentifier()`/`AddInconsistentOpRecord()`/`CheckOpInconsistentError()`三个函数
3. 将opInfo注册条件从仅`SEND/RECEIVE`扩大到所有集合通信算子
4. 发送逻辑重构：引入`opInfoQueueForSend_`发送缓冲队列，增加`HBFRAME_SEND_LOOP_MAX_NUM=120`的发送重试循环（带100us sleep）
5. 新增`CommCheckOpInconsistentError`公开接口

### Revert根因

HeartBeatFrame结构体膨胀导致性能和内存灾难：

- 帧大小膨胀约30倍：原版`OpInfoDesc opInfoList[16]`约6-7KB，修改后`OpInfoTagQueue[10] * OpInfoDesc[500]`约200KB
- 心跳每50ms广播一次，发送缓冲区`sendBuffer`（最大3072帧）内存占用从约20MB暴增到约600MB
- 120次重试循环（每次sleep 100us = 最多30ms）阻塞心跳线程，50ms周期内60%时间被sleep占用
- `inconsistentOpMap_`和`srTagMap_`只增不清理，长时间运行存在内存泄漏风险
- 功能直接全量生效，无灰度开关（后续重新引入ebfcde5c加了"with switch"证实此教训）

### 暴露的缺陷模式

1. 数据结构膨胀无约束：嵌套定长数组`[500]*[10]=5000`个元素嵌入每帧，远超实际需要，设计时未考虑网络I/O中结构体大小的乘法效应
2. 巨型提交掩盖问题：原始功能隐藏在678文件的"update"大批量同步提交中，审查者无法在海量无关变更中发现帧膨胀30倍
3. 阻塞式重试设计：在周期性心跳线程中引入sleep重试循环，违反非阻塞心跳的设计原则
4. 缺少feature flag：影响跨节点协议的功能变更直接全量生效，无灰度开关

### 审查教训

- 改变跨节点传输帧格式的提交，必须单独提交并附带帧大小分析（before/after sizeof）
- 包含嵌套定长数组的结构体出现在网络传输路径上时，必须有sizeof的审查检查项
- 大批量同步提交不应包含功能变更——同步应是纯机械操作，功能修改必须独立提交
- 在周期性线程中引入sleep/重试时，必须评估对周期的影响（30ms vs 50ms = 60%占用）
- 影响跨节点协议的功能上线应具备灰度能力（环境变量开关）

---

## Revert#2: 30e25e50 — Revert Runtime interfaces

| 属性 | 值 |
|------|-----|
| Revert hash | 30e25e509b127c077358877259046992f7111d34 |
| 日期 | 2025-12-16 |
| 作者 | jiyuanhao |
| 原始变更 | 来自d38afa51等多个commit的runtime接口替换 |
| 涉及文件 | 9个（adapter_rts.cc/h, stream_utils.cc, op_base_host.cc等） |

### 原始变更内容

将ACL高层接口替换为RT层直接接口：

1. `reinterpret_cast<uint64_t>(rtModel)`(hack写法) → `rtModelGetId()`(正规API)
2. `aclrtSetExceptionInfoCallback` → `rtRegTaskFailCallbackByModule`(引入模块名"HCCL"参数)
3. 公共头文件`hcomm_primitives.h`中`uint32_t` → `u32`(内部typedef)

### Revert根因

1. 对未发布API的抢跑依赖：`rtModelGetId`和`rtRegTaskFailCallbackByModule`在当时的runtime SDK中尚未发布，构建时链接失败
2. 公共头文件ABI破坏：`uint32_t`改为`u32`出现在对外头文件中，破坏外部用户ABI兼容性
3. 引入非法头文件依赖：`adapter_rts.cc`中引入了`acl/error_codes/rt_error_codes.h`内部头文件

### 暴露的缺陷模式

1. 抢跑依赖未发布API：在runtime SDK正式发布新接口前就在消费端代码中使用，导致链接失败
2. 公共头文件类型不一致：对外API头文件必须使用标准C/C++类型，不能使用内部typedef
3. "看起来是改进"的重构放松审查警惕：去掉hack、使用正式API看起来"显然更好"，容易让审查者忽略前置条件是否满足

---

## Revert#3: 1d8e2c14 — [Master] Revert rts interfaces

| 属性 | 值 |
|------|-----|
| Revert hash | 1d8e2c143905a5f2f035df01d7e0b12304e3fc13 |
| 日期 | 2025-12-23 |
| 作者 | jiyuanhao |
| 前序事件 | f9d03829(12/12, C25 Revert) → 9968572e(12/21, 重新提交) → 本次(12/23) |
| 涉及文件 | 7个（adapter_rts.cc/h, p2p_mgmt.cc, UT头文件等） |

注意：Revert#2和Revert#3属于同一runtime接口迁移系列，涉及同一组文件的反复修改。

### 原始变更内容

commit message名为"Revert"，但实际是正向引入rt底层接口替换ACL接口：

1. `aclrtGetDeviceInfo` → `rtGetPhyDeviceInfo`（设备信息查询）
2. 引入`hrtGetPairDeviceLinkTypeRaw`双路径函数（先尝试dlopen加载`rtGetPairPhyDevicesInfo`，失败fallback到ACL的`aclrtGetDevicesTopo`）
3. P2P管理接口从ACL(`aclrtDeviceEnablePeerAccess`等) → RT(`rtEnableP2P`等)，函数签名从`(u32 peerDevPhyId)`改为`(u32 deviceLogicId, u32 devicePhyId)`

### "乒乓Revert"时间线

```
12/11  d38afa51  原始runtime接口替换
12/12  f9d03829  [C25] Revert rts interfaces
12/16  30e25e50  [Master] Revert Runtime interfaces    ← Revert#2
12/21  9968572e  [C25] Use external runtime interfaces  (部分切回ACL+混合变更)
12/23  1d8e2c14  [Master] Revert rts interfaces        ← Revert#3(实际是再次引入rt接口)
```

12天内对同一组文件进行了5次方向性变更。

### Revert根因

1. rt接口可用性不确定：某些rt接口在特定runtime版本中不可用，不得不引入dlopen动态探测+fallback机制
2. 返回值语义差异：ACL的`aclrtGetDevicesTopo`返回bitmask，rt的`rtGetPairPhyDevicesInfo`返回enum值，切换时调用方解析逻辑未同步适配
3. 缺少前期接口兼容性验证：在切换底层runtime依赖前未确认目标接口在所有部署环境都可用

### 暴露的缺陷模式

1. 命名与行为严重脱节：多个commit message都叫"Revert rts interfaces"，但实际方向完全是"引入rt接口"。后续维护者无法从提交历史推断意图
2. 乒乓Revert模式：同一功能12天内5次方向性变更，说明缺少前期的接口兼容性验证和完整迁移方案
3. 混合提交：`9968572e`同时做了code hygiene(删extern改头文件)、功能回退(切回ACL)、UT更新，应拆为独立提交
4. 缺少运行时兼容性设计：dlopen双路径fallback在第一次引入时就应考虑，而非经过两轮revert后才补上

### 审查教训（Revert#2和Revert#3共享）

- 跨层接口迁移需要兼容性验证前置：切换runtime依赖前需checklist确认目标接口在所有受支持版本已发布
- "Revert"一词仅用于真正的回退操作，不应描述正向迁移
- 返回值语义差异（bitmask vs enum）极易引入silent bug，审查时重点检查
- 反复revert是设计不充分的信号：审查者应暂停合并，要求提供完整迁移方案（含兼容性矩阵和rollback策略）
- 同一功能在多分支(Master/C25)反复revert，说明需要跨分支集成验证

---

## Revert#4: 1844b823 — Revert "ars code"

| 属性 | 值 |
|------|-----|
| Revert hash | 1844b8232208fcffe2d5e2676282ddf4ad700d0a |
| 日期 | 2026-01-08 |
| 作者 | liuwanke152 |
| 原始提交 | 99d2a2b3（2026-01-07, MR !713, "ars code"） |
| 涉及文件 | 51个（跨topo/executor/operator/platform四层） |

### 原始变更内容

为HCCL在910_93芯片上引入ARS(Adaptive Ring Size)算法，核心思想是在server内卡数不对称拓扑下动态计算最优环内大小。包含：

1. 新增3个ARS executor文件（AllReduce/AllGather/ReduceScatter各一个）
2. 新增`CalcOptimalIntraRingsize()`带宽代价模型优化器——但在`coll_comm_executor.cc`和`coll_alg_operator.cc`中被完整复制了两份，且存在微妙的成员变量引用差异
3. 修改`topo_matcher`新增COMM_LEVEL0_LOGICAL/COMM_LEVEL1_LOGICAL逻辑通信平面
4. 重构`GetBandWidthPerNPU()`从switch-case改为查表法
5. 修改`MultiRingAllGather`和`MultiRingReduceScatter`函数签名（新增`CommPlane levelIndex`参数）
6. 基类`CollAllReduceRingFor91093Executor`侵入式修改，新增protected成员
7. `isARSDoubleRing`初始化为`true`——新特性默认开启
8. 净增2104行（2283+, 179-），commit message仅两个单词"ars code"

### Revert根因

提交1天后即被revert（1/7提交 → 1/8 revert），推断原因：

1. 侵入性过强：51个文件跨4个代码层次的全栈修改，函数签名变更影响所有调用者，未同步更新的调用点导致编译失败
2. 代码重复：`CalcOptimalIntraRingsize()`复制两份，一份用`topoAttr_.userRankSize`另一份用`userRankSize_`，一份用`topoAttr_.isARSDoubleRing`另一份用`isARSDoubleRing_`
3. 公共接口非向后兼容修改：`GetBandWidthPerNPU()`语义变更，`SetRankMap()`从private变public
4. 默认值陷阱：`isARSDoubleRing = true`让新特性默认开启，所有910_93场景都可能走到ARS路径

### 暴露的缺陷模式

1. 大爆炸提交(Big Bang Commit)：51个文件、2000+行、跨4层的变更压缩在commit message只有"ars code"的单个提交中
2. Copy-Paste代码：核心算法函数被完整复制到两个类中，带有微妙的成员变量引用差异，典型DRY违反
3. 函数签名的破坏性变更：`MultiRingAllGather`/`MultiRingReduceScatter`新增参数改变签名，影响所有调用者
4. 默认值陷阱：新特性flag默认true，违反feature flag安全引入原则（新特性应默认关闭）
5. 缺少commit message：两个单词的message无法传达变更意图、影响范围、测试策略

### 审查教训

- 拆分提交是有效审查的前提：至少应拆为5个独立提交（拓扑发现/带宽重构/优化算法提取/executor实现/operator选择逻辑）
- 函数签名变更需单独审查，配合全量搜索确认所有调用点已更新
- 重复代码应在审查阶段被拦截，要求提取为共享实现
- 新增布尔flag的默认值应被质疑：新特性默认开启是否安全？
- 快速revert（仅1天）暴露CI/测试覆盖不足：编译或功能回归应在MR阶段被拦截
- commit message是审查入口，reviewer应要求提交者提供足够的上下文说明

---

## 跨Revert共性模式总结

### 模式1：大批量/巨型提交掩盖关键变更
- Revert#1：功能隐藏在678文件的"update"同步提交中
- Revert#4：51文件跨4层的全栈变更压缩在"ars code"中
- 教训：功能变更必须独立提交，大批量同步不应包含功能修改

### 模式2：对未就绪依赖的抢跑集成
- Revert#2/#3：依赖未发布的runtime API导致链接失败，12天内5次方向性变更
- 教训：跨团队接口迁移需前置兼容性验证，确认目标API在所有环境可用

### 模式3：缺少灰度/开关机制
- Revert#1：心跳帧格式变更直接全量生效，无法线上回退
- Revert#4：`isARSDoubleRing = true`默认开启新特性
- 教训：影响跨节点协议或全局行为的功能需feature flag

### 模式4：数据结构/接口的破坏性变更未隔离审查
- Revert#1：帧结构膨胀30倍未做sizeof分析
- Revert#2：公共头文件`uint32_t`→`u32`破坏ABI
- Revert#4：函数签名变更未全量检查调用点
- 教训：涉及ABI/协议/签名的变更必须单独提交并配套影响分析

### 模式5：commit message质量低下
- Revert#1原始提交："update"（678文件）
- Revert#3："Revert rts interfaces"（实际是正向引入）
- Revert#4："ars code"（51文件2000+行）
- 教训：commit message是审查第一入口，应准确描述变更意图和影响范围
