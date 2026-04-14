# Phase 6: Cross-Operator Contrastive Analysis

## 6.1 标准MC2算子 vs MoE算子

### 结构差异

| 维度 | MatMul+Comm家族 | MoE家族 |
|------|----------------|---------|
| 核心计算 | MatMul(Cube) + 通信 | AlltoAll数据重排 + Scale(Vector) |
| MatMul库 | 3rd/mat_mul_v3, common/new_mc2_mm | 无(自实现向量计算) |
| 通信方式 | HCCL高级API(AllReduce/AllGather/ReduceScatter) | RDMA窗口直操作 + 状态位轮询 |
| 通信域 | 单域(group) | 双域(group_ep + group_tp) |
| arch适配 | arch31/32/35目录分离 | 运行时if/else分派(SoC/NpuArch) |
| tiling架构 | 深继承链(TilingBaseClass→Base→10+具体类) | MoeTilingBase单子类 + 内部if分派 |
| op_host注册 | REGISTER_TILING_TEMPLATE_WITH_ARCH(arch35) | REGISTER_OPS_TILING_TEMPLATE(自管arch) |
| AIC/AIV分工 | AIC做MatMul, AIV做通信+搬运 | 全部AIV(无Cube计算) |
| tiling key | 12-16个参数(业务复杂) | 4个参数(相对简单) |

### 代码组织差异

MatMul+Comm(以matmul_all_reduce为例):
- op_kernel: 51个文件, arch31/32/35三个子目录
- 每个arch有独立的base类和变体实现
- 通过INVOKE宏封装标准Init+Process流程

MoE(以moe_distribute_combine为例):
- op_kernel: 3个文件, 无arch子目录
- v2代码通过跨目录include引用
- Process()是880行的单文件巨型实现

### 四层接口差异

op_host层:
- MatMul: IgnoreContiguous (16处/7文件) -- MatMul特有
- MoE: AutoContiguous (230处/23文件) -- 默认模式
- MatMul: config/下无binary.json/simplified_key.ini
- MoE: config/下有binary.json + simplified_key.ini

op_graph层:
- MatMul(matmul_all_reduce): GenTask按NpuArch分派(NPUARCH_A5)
- MoE(moe_distribute_combine): GenTask按SocVersion分派(PLATFORM_A2)
- MoE额外有REGISTER_EXT_TASK_TYPE注册(11处/11文件)

op_api层:
- MatMul: NnopbaseDisableOptionalInput禁用可选输入(大量可选参数)
- MoE: 无此需求(参数较固定)
- MatMul: NnopbaseSetHcclServerType运行时设CCU
- MoE: 无此步骤(通信不经HCCL高级API)

### 关键结论

MatMul+Comm和MoE虽然同属MC2模块，但编程范式差异很大:
1. MatMul+Comm是"Cube计算 + HCCL通信"的标准组合
2. MoE是"Vector计算 + RDMA窗口"的自定义组合
3. 新算子应根据是否涉及MatMul选择对应范式，不应混搭

## 6.2 v1/v2/v3版本演进对比

### 版本分布

有v2的算子(4个): all_gather_matmul, matmul_reduce_scatter, moe_distribute_combine, moe_distribute_dispatch
有v3的算子(2个): moe_distribute_combine, moe_distribute_dispatch

### all_gather_matmul vs all_gather_matmul_v2

| 维度 | v1 | v2 |
|------|----|----|
| arch支持 | arch32 only | arch35 only |
| HCCL类型 | HCCL_SERVER_TYPE_AICPU | HCCL_SERVER_TYPE_CCU |
| MatMul库 | mc2/common/MatmulCompute | Mc2MatmulAswBlock(ASW) |
| kernel入口 | 旧tiling key(3 bool) | APT tiling key |
| 量化支持 | 无 | FP8/HIF8/MX量化 |
| 文件数 | 5 | 多(arch35/目录) |

v2的本质: 为arch35(Ascend 950)重新实现，非v1的简单升级。

### moe_distribute_combine v1/v2/v3

v1(moe_distribute_combine/): 基础AlltoAll + Scale(arch32/35共用)
v2(moe_distribute_combine_v2/): 多拓扑(layered/fullmesh/host_kfc) + arch35专用路径 + AICPU/AIV DataplaneMode分派
v3(moe_distribute_combine_v3/): 动态context传递(早期开发)

演进方向:
- v1→v2: 从单一拓扑到多拓扑适配(layered/fullmesh)
- v2→v3: 从静态配置到动态context(setup/teardown生命周期)
- v1仍保留: A3/A5简单场景下的fallback

### setup/teardown模式(v3新增)

setup(moe_distribute_combine_setup/): 通信初始化，分配窗口资源
teardown(moe_distribute_combine_teardown/): 释放通信资源

这是v3的架构创新: 将通信生命周期从kernel执行中解耦，允许跨多次kernel调用复用通信资源。

### 版本演进规律

1. v1: 基础功能，单arch
2. v2: 多arch适配 + 多拓扑 + 量化扩展 (当前主力)
3. v3: 架构创新(动态context/生命周期解耦) (早期开发)

MatMul类: v2是独立算子(arch35专用)，而非v1的升级
MoE类: v2是v1的超集(共用部分代码)，v3是架构重构

## 6.3 量化算子对比

### 独立量化算子 vs 内嵌量化变体

独立量化算子(3个):
- quant_all_reduce: 独立通信量化
- quant_reduce_scatter: 独立通信量化
- quant_grouped_mat_mul_allto_allv: 量化GMM+AlltoAll

内嵌量化变体(通过tiling key区分):
- matmul_all_reduce: FP/INT8_QUANT/WEIGHT_QUANT三种模式
- all_gather_matmul_v2: FP/FP8/HIF8/MX量化
- moe_distribute_combine: NO_QUANT/INT8_QUANT两种模式

### 量化带来的额外复杂度

op_kernel层:
- 额外tiling key参数: COMMDTPYE(通信量化类型), QUANTTYPE, HAS_ANTIQUANT_OFFSET
- 额外INVOKE宏: INVOKE_MC2_QUANT_*_OP_IMPL (6种变体)
- 额外impl文件: matmul_all_reduce有quant/weight_quant/pertoken/perblock/commfp8等5+个变体

op_host层:
- 额外tiling类: QuantMatmulAllReduceTiling, WeightQuantMatmulAllReduceTiling
- 更低priority(优先匹配量化tiling)
- 额外属性: antiquant_group_size, comm_quant_mode, pertoken_scale

op_api层:
- 独立API版本: aclnnQuantMatmulAllReduce, aclnnQuantMatmulAllReduceV2/V3/V4
- matmul_all_reduce有7个API版本(非量化2 + 量化4 + 伪量化1)
- 独立文档: 每个量化API版本都有独立.md

### 量化演进

v1(INT8): aclnnQuantMatmulAllReduce -- 基础INT8量化
v2: aclnnQuantMatmulAllReduceV2 -- 增加pertoken_scale
v3: aclnnQuantMatmulAllReduceV3 -- 增加comm_quant_scale(通信量化)
v4: aclnnQuantMatmulAllReduceV4 -- 增加group_size/comm_quant_mode

FP8/HIF8量化: 在APT tiling key中通过SCENARIO_MXFP8和COMMDTPYE表达
Weight量化(W4/W8): 独立的INVOKE宏和实现路径(weight_quant_batch_matmul_v2)

## 6.4 四层一致性检查

### 目录结构一致性

100%一致: 所有33个算子目录都有op_host/op_kernel/op_graph/op_api四个子目录
大部分有: docs/(31个), tests/(31个), examples/(31个)
缺少docs/tests/examples: moe_distribute_combine_v3, moe_distribute_dispatch_v3 (最新v3缺失)
缺少examples: moe_distribute_combine_setup (只有docs+op_api+op_graph+op_host+op_kernel)

### 注册宏一致性

| 注册点 | 宏 | 覆盖率 |
|--------|---|--------|
| op_host/tiling | IMPL_OP_OPTILING | 37/37 (100%) |
| op_host/infershape | IMPL_OP_INFERSHAPE | 33/33 (100%) |
| op_host/def | OP_ADD | 35/35 (100%) |
| op_kernel | REGISTER_TILING_DEFAULT | 44/44 (100%) |
| op_graph/gen_task | IMPL_OP | 26/26 (100%) |
| op_graph/fallback | IMPL_OP(.OpExecuteFunc) | 23/23 (100%) |
| op_api | extern "C" | 32/32 (100%) |

所有注册宏的覆盖率都是100%，说明MC2的四层架构规范执行非常严格。

### 跨层命名一致性

规律: 同一算子在四层中使用相同的Pascal Case名称。
- 例: MatmulAllReduce在OP_ADD/IMPL_OP/IMPL_OP_OPTILING/IMPL_OP_INFERSHAPE中完全一致
- op_api的C函数名使用camelCase前缀: aclnnMatmulAllReduce

文件命名:
- op_host: matmul_all_reduce_def.cpp / *_tiling.cpp / *_infershape.cpp
- op_kernel: matmul_all_reduce.cpp / *_apt.cpp
- op_graph: matmul_all_reduce_gen_task.cpp / fallback_matmul_all_reduce.cpp
- op_api: aclnn_matmul_all_reduce.cpp

## 6.5 命名和编码风格一致性

### 文件命名

100%小写蛇形(snake_case): 所有.cpp/.h文件
op名在文件名中使用蛇形: matmul_all_reduce, moe_distribute_combine
aclnn API文件带前缀: aclnn_

### 类和结构体命名

PascalCase: MatmulAllReduceBase, MoeDistributeCombine, HcclCombinOpParam
模板参数: 全大写蛇形: HCCL_SERVER_TYPE_CCU, ON_CUBE_AND_VECTOR

### 宏命名

全大写: REGISTER_TILING_DEFAULT, INVOKE_MC2_910_OP_IMPL, BUILD_OPEN_PROJECT
条件编译: 双下划线前缀: __CCE_AICORE__, __NPU_ARCH__, __CCE_KT_TEST__

### namespace

ops: gen_task和fallback文件的顶层namespace
fallback: fallback文件的嵌套namespace
AscendC: kernel层的通用namespace
MatmulAllReduceImpl/AllGatherMatmulImpl: 各算子kernel的专用namespace

### 不一致之处

1. HCCL枚举风格:
   - arch32: HCCL_SERVER_TYPE_AICPU (C风格常量)
   - arch35: HcclServerType::HCCL_SERVER_TYPE_CCU (C++枚举类)
   - 同一概念两种表达

2. 属性访问方式:
   - matmul_all_reduce: MmAllReduceAttrIdx枚举 + static_cast<size_t>
   - all_gather_matmul: magic index++ (反模式)
   - moe_distribute_combine: attrs->GetBool/GetInt + 内联解析

3. 平台判断方式:
   - NpuArch: IsTargetPlatformNpuArch(nodeName, NPUARCH_A5) -- MatMul类
   - SocVersion: IsTargetPlatformSocVersion(nodeName, PLATFORM_A2) -- MoE类
   - 直接判断: GetCurrentPlatformInfo().GetCurNpuArch() == DAV_3510 -- op_api

## 6.6 Phase 6总结

### "MC2模块级强制"(所有算子一致)

1. 四层目录结构: op_host + op_kernel + op_graph + op_api (100%)
2. 六个注册点: IMPL_OP_OPTILING + IMPL_OP_INFERSHAPE + OP_ADD + REGISTER_TILING_DEFAULT + IMPL_OP(gen_task) + extern "C"(op_api) (100%)
3. BUILD_OPEN_PROJECT双版本gen_task (100%)
4. CommonKFCMc2CalcParamFunc (100%)
5. 两阶段API模式(GetWorkspaceSize + Execute) (100%)
6. DFX打点(NnopbaseMsprofSysTime) (100%)
7. SetScheduleMode(1) (MC2模块级)

### "常见做法"(多数算子遵循)

1. MatMul类使用arch目录隔离(31/32/35)，MoE类使用运行时分派
2. arch35使用CCU + APT + ASW三件套
3. 量化变体通过tiling key区分(而非独立算子)
4. v1/v2共存: v1处理旧arch，v2处理arch35
5. 使用Mc2SyncAll封装同步(MatMul类)

### "算子级局部做法"

1. mc2_templates流水线框架: 仅matmul_allto_all和allto_allv_quant_grouped_mat_mul
2. REGISTER_EXT_TASK_TYPE: 仅MoE/quant/barrier等11个算子
3. MoE窗口自实现通信: 仅MoE家族
4. 跨版本include: 仅moe_distribute_combine引用v2
5. config/binary.json: 仅MoE系列

### 演进方向总结

| 方面 | 旧模式 | 新模式 | 状态 |
|------|--------|--------|------|
| 通信控制 | AICPU代理 | CCU直驱 | arch35全面采用 |
| tiling key | 实现导向(16参数) | 业务导向APT(12参数) | 6个算子已迁移 |
| MatMul Block | BaseBlock | ASW Block | arch35全面采用 |
| 流水线编排 | 手工编排 | mc2_templates | 2个算子试点 |
| 版本管理 | 同目录条件编译 | 独立v2目录 | 普遍采用 |
| 通信生命周期 | kernel内嵌 | setup/teardown解耦 | v3试点 |
| 平台判断 | socVersion字符串 | NpuArch枚举 | 迁移进行中 |
| HCCL初始化 | Init(context) | InitV2(context, tiling) | arch35采用 |
