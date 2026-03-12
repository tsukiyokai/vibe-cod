# 分布式训练迁移中的HCCL相关内容

> 来源：CANN商用版8.0.RC3 hiascend.com

---

## GPU单卡脚本迁移为NPU多卡脚本

### 概述

本指南说明如何将GPU单卡训练脚本迁移为NPU多卡训练脚本。

### 操作步骤

#### 1. 训练脚本语句替换

修改生成的 `run_distributed_npu.sh` 文件，将占位符 "please input your shell script here" 替换为实际的模型训练命令。

示例：将其替换为 `bash model_train_script.sh --data_path data_path`

##### run_distributed_npu.sh文件示例

```bash
export MASTER_ADDR=127.0.0.1
export MASTER_PORT=29688
export HCCL_WHITELIST_DISABLE=1

NPUS=($(seq 0 7))
export RANK_SIZE=${#NPUS[@]}
rank=0
for i in ${NPUS[@]}
do
    export DEVICE_ID=${i}
    export RANK_ID=${rank}
    echo run process ${rank}
    please input your shell script here > output_npu_${i}.log 2>&1 &
    let rank++
done
```

##### 参数说明

| 参数                   | 说明               |
| ---------------------- | ------------------ |
| MASTER_ADDR            | 训练服务器的IP地址 |
| MASTER_PORT            | 训练服务器的端口号 |
| HCCL_WHITELIST_DISABLE | HCCL通信白名单开关 |
| NPUS                   | 指定运行的NPU设备  |
| RANK_SIZE              | 调用卡的数量       |
| DEVICE_ID              | 调用的device_id    |
| RANK_ID                | 调用卡的逻辑ID     |

#### 2. 执行脚本

运行修改后的 `run_distributed_npu.sh` 文件，将生成各NPU的日志文件。

#### 3. 查看结果文件

迁移完成后，结果目录结构如下：

```
xxx_msft/xxx_msft_multi/
├── 生成脚本文件                     # 与迁移前目录结构一致
├── msFmkTranspltlog.txt             # 迁移过程日志（限制1M）
├── cuda_op_list.csv                 # CUDA算子列表
├── unknown_api.csv                  # 支持情况存疑的API
├── unsupported_api.csv              # 不支持的API列表
├── change_list.csv                  # 修改记录
├── run_distributed_npu.sh           # 多卡启动脚本
└── ascend_modelarts_function/
    ├── modelarts_path_manager.py    # 路径映射适配层代码
    └── path_mapping_config.py       # 路径映射配置文件
```

#### 4. 验证迁移结果

检查迁移后的Python脚本，确认CUDA API已被替换为NPU API。关键变化包括：

```python
if torch_npu.npu.is_available():
    ngpus_per_node = torch_npu.npu.device_count()
else:
    ngpus_per_node = 1
```
