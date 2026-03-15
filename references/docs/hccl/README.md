# HCCL文档集

来源：华为CANN商用版8.0.RC3 (hiascend.com)
抓取时间：2026-03-08

## 文件索引

| 文件            | 内容                                                                                                                     | 行数 |
| --------------- | ------------------------------------------------------------------------------------------------------------------------ | ---- |
| user-guide.md   | 集合通信用户指南（概述、术语、通信原语、分级通信、通信算法、功能开发、集群配置、样例代码、6种初始化方式、编译运行、FAQ） | 3911 |
| api-c.md        | HCCL API参考(C) — 通信域管理、集合通信、点对点通信、异常处理、数据类型                                                   | 1514 |
| api-cpp.md      | HCCL API参考(C++) — 与C接口共用，指向api-c.md                                                                            | 5    |
| api-python.md   | HCCL API参考(Python) — hccl.manage.api、hccl.split.api、npu_bridge.hccl.hccl_ops                                         | 888  |
| env-vars.md     | HCCL相关环境变量 — PyTorch扩展（4个） + 集合通信（23个）                                                                 | 1051 |
| ascendc-hccl.md | Ascend C HCCL高阶API — 15个核心接口 + 2个Tiling数据结构                                                                  | 1009 |
| hccl-test.md    | HCCL性能测试工具 — 工具介绍/编译/执行/参数/结果/约束/FAQ                                                                 | 712  |
| faq.md          | 故障处理（用户指南外的条目）                                                                                             | 44   |
| migration.md    | GPU单卡→NPU多卡迁移指南                                                                                                  | 84   |
| Ascend C算子开发指南.md/.pdf | Ascend C算子开发指南（含PDF原件）                                                                        | 13333 |
| HCCL集合通信库使用指南.md/.pdf | HCCL集合通信库使用指南（含PDF原件）                                                                    | 2903  |

合计：25454行，13个文件

## 页面来源

共抓取178个页面（doc_center静态HTML）：

| 类别            | 页面数 | doc_center路径前缀            |
| --------------- | ------ | ----------------------------- |
| 用户指南        | 52     | developmentguide/hccl/hcclug/ |
| API (C/C++)     | 49     | apiref/hcclapiref/            |
| API (Python)    | 19     | apiref/hcclapiref/            |
| 环境变量        | 27     | apiref/envvar/                |
| HCCL Test       | 12     | devaids/devtools/hccltool/    |
| Ascend C高阶API | 17     | apiref/ascendcopapi/          |
| 其他            | 2      | （故障处理 + 迁移工具）       |

详细URL索引见中间产物：`docs-dig/hccl/url-index.md`

## 已知限制

- 原网站图片/图表无法提取，以文字描述替代
- 原始HTML中的站内相对链接未全部转换
- C++接口文档与C共用同一套hcclcpp头文件，未单独展开
