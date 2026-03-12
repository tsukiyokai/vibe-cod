# hccl-dev Revert专项分析

## 概览

共发现4个revert相关提交，涉及2个独立事件。

## 事件1：打包目录结构反复变更（share/info flip-flop）

时间线：
1. MR !18 (b6aec59, 2025-11-26)：将打包路径从`hccl/`改为`share/info/hccl/`，新增`--use-share-info`命令行参数，影响18个文件
2. MR !20 (d67b6a5/52bda68, 2025-11-27)：24小时内revert MR !18，回退到`hccl/`路径
3. MR !22 (9e7ea4f, 2025-11-27)：同日再次revert MR !20（即恢复MR !18的修改），重新使用`share/info/hccl/`路径

分析：
- 2天内对同一个18文件的大规模修改执行了3次（加-撤-再加），说明打包目录结构的设计决策在合入前未充分确认
- 每次变更涉及cmake、shell脚本、xml配置等多种文件类型，反复修改增加了遗漏风险
- 该事件本身不是代码缺陷，但反映了架构决策确认不足的流程问题

审查启示：
- 大规模路径/目录结构变更应有明确的设计评审记录，避免合入后反悔
- 涉及多种配置文件的变更应有完整的回归测试

## 事件2：API命名批量重命名的premature merge

时间线：
1. MR !34 (0148fc4, 2025-11-29)：API函数批量重命名，影响6个文件
   - `CommGetEngineCtx` → `HcclGetEngineCtx`
   - `CommCreateEngineCtx` → `HcclCreateEngineCtx`
   - `CommAllocThreadResByStream` → `HcclAllocThreadResByStream`
   - `CommChannelCreate` → `HcclChannelCreate`
   - `CommGetRankGraph` → `HcclGetRankGraph`
   - `CommLocalBareNotifyRecord` → `HcommInterOpNotifyRecordOnThread`
   - `CommLocalBareNotifyWait` → `HcommInterOpNotifyWaitOnThread`
   - 还添加了`__attribute__((weak))`前向声明
2. MR !44 (29c59f3, 2025-11-29)：同日revert MR !34

分析：
- API重命名从`Comm`前缀统一到`Hccl`前缀，方向合理（统一命名空间），但实现时机不对
- 被revert可能原因：(1)下游依赖尚未同步更新；(2)weak symbol方案引入的不确定性；(3)重命名范围未覆盖所有调用点
- 同日合入又revert说明变更未经充分的集成测试验证

审查启示：
- 批量API重命名需制定迁移计划，确认所有调用方已准备好
- 使用`__attribute__((weak))`做过渡的方案需要明确weak symbol的生命周期和回退策略
- 大规模重命名应在独立分支充分测试后再合入

## 与缺陷提交的关联

事件2的revert与缺陷提交c11a5289、8222fcf8有间接关联：这些编译错误修复涉及HcclCommConfig结构体字段的变更（commEngine/threadNum/notifyNumPerThread的增删），属于同一时期的API迭代。频繁的API变更-revert-再变更增加了不同提交之间的接口不一致风险。
