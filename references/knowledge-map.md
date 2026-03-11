# 知识路由表

本文件是外部知识资源的索引，指导按需查阅。不要一次性加载所有资源。

---

## 1. 缺陷模式知识库

来源: /Users/shanshan/note/proj/repo-dig/
基于9237条提交、2279条确认缺陷的系统性分析。

### 跨仓通用模式 (8个)
路径: /Users/shanshan/note/proj/repo-dig/cross_repo_synthesis.md
内容: 8个跨仓共性缺陷模式的详细分析（整数溢出、分支不完整、CMake遗漏、复制粘贴、流水线同步、Host-Kernel不一致、空指针、Revert）
何时读: 开始任何编码任务前，通读与任务相关的模式章节
查阅方式: Grep "共性-N" 或关键词跳转到对应章节

### 仓库专项分析

hccl-dev:
  根目录: /Users/shanshan/note/proj/repo-dig/hccl-dev/
  pattern_summary.md — 5个缺陷类别: 枚举混淆(30%)、接口变更传播(30%)、复制粘贴(20%)、条件不完整(10%)、CMake(10%)
  standards_project_hccl-dev.md — 审查规则集
  defect_analysis.md — 10条缺陷逐条分析（需要看具体案例时读）

hcomm-dev:
  根目录: /Users/shanshan/note/proj/repo-dig/hcomm-dev/
  pattern_summary.md — 12个缺陷类别(162条缺陷): 构建(14.8%)、初始化时序(10.5%)、条件分支(9.9%)、数据类型(9.9%)、命名拼写(9.3%)、资源管理(7.4%)、并发安全(6.8%)、日志(6.8%)、状态缓存超时(6.2%)、空指针(4.9%)、API设计(4.9%)、硬件适配(3.7%)
  standards_project_hcomm-dev.md — 审查规则集
  defect_analysis.md — 162条缺陷逐条分析
  hotspot_analysis.md — 18个热点文件风险分析

ops-transformer[-dev]:
  根目录: /Users/shanshan/note/proj/repo-dig/ops-transformer/ 和 ops-transformer-dev/
  pattern_summary.md — 算子开发缺陷模式
  适用场景: 涉及算子tiling、kernel开发时参考

ops-nn[-dev]:
  根目录: /Users/shanshan/note/proj/repo-dig/ops-nn/ 和 ops-nn-dev/
  pattern_summary.md — 算子开发缺陷模式
  适用场景: 涉及NN算子开发时参考

---

## 2. Ascend C文档

### API参考 (661页)
索引: /Users/shanshan/repo/me/docs/ascendc/api_reference/INDEX.md
页面目录: /Users/shanshan/repo/me/docs/ascendc/api_reference/pages/
查阅方式: Grep INDEX.md搜API名 → 获取页面文件名 → Read pages/下对应.md

高频API直接路径:
  AllReduce    → pages/atlasascendc_api_07_0872.md
  AllGather    → pages/atlasascendc_api_07_0873.md
  ReduceScatter → pages/atlasascendc_api_07_0874.md
  AlltoAll     → pages/atlasascendc_api_07_0875.md
  HCCL InitV2  → pages/atlasascendc_api_07_10221.md
  HCCL Finalize → pages/atlasascendc_api_07_0881.md
  DataCopy(概览) → pages/atlasascendc_api_07_0101.md
  DataCopy(基础) → pages/atlasascendc_api_07_0103.md
  SetFlag/WaitFlag → pages/atlasascendc_api_07_0270.md
  CrossCoreSetFlag → pages/atlasascendc_api_07_0273.md
  CrossCoreWaitFlag → pages/atlasascendc_api_07_0274.md
  PipeBarrier  → pages/atlasascendc_api_07_0271.md
  Muls         → pages/atlasascendc_api_07_0055.md
  TPipe        → pages/atlasascendc_api_07_0108.md

### 基础知识 (92页)
索引: /Users/shanshan/repo/me/docs/ascendc/basic_knowledge/INDEX.md
页面目录: /Users/shanshan/repo/me/docs/ascendc/basic_knowledge/pages/
高频主题:
  核函数          → pages/atlas_ascendc_10_0014.md
  TPipe和TQue编程 → pages/atlas_ascendc_10_0016.md
  编程模型设计原理  → pages/atlas_ascendc_10_00015.md
  DoubleBuffer    → pages/atlas_ascendc_10_00016.md
  Scalar读写数据   → pages/atlas_ascendc_10_00030.md

### 最佳实践 (54页)
索引: /Users/shanshan/repo/me/docs/ascendc/best_practice/INDEX.md
页面目录: /Users/shanshan/repo/me/docs/ascendc/best_practice/pages/

---

## 3. HCCL文档

根目录: /Users/shanshan/repo/me/docs/hccl/

按场景选择:
  通信整体流程、拓扑、算法   → user-guide.md (3911行，用Grep搜索关键词)
  C语言API用法              → api-c.md (1514行)
  Python API               → api-python.md (888行)
  AscendC + HCCL集成开发    → ascendc-hccl.md (1009行，15个核心接口)
  环境变量配置              → env-vars.md (1051行)
  性能测试工具              → hccl-test.md (712行)
  故障排查                  → faq.md
  版本迁移                  → migration.md

---

## 4. AI4SE分析

根目录: /Users/shanshan/repo/me/docs/ai4se/
HCCL业务分析: phase1_hccl_analysis.md
理论框架: phase2_theory_framework.md
阶段分析: phase3_stage_analysis.md
