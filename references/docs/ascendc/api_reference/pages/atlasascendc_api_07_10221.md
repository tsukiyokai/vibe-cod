# InitV2-HCCL Kernel侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10221
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10221.html
---

# InitV2

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

HCCL客户端初始化接口。该接口默认在所有核上工作，用户也可以在调用前通过GetBlockIdx指定其在某一个核上运行。

#### 函数原型

| 1   | __aicore__inlinevoidInitV2(GM_ADDRcontext,constvoid*initTiling) |
| --- | --------------------------------------------------------------- |

#### 参数说明

| 参数名     | 输入/输出 | 描述                                                                                                                                |
| ---------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| context    | 输入      | 通信上下文，包含rankDim，rankID等相关信息。通过框架提供的获取通信上下文的接口GetHcclContext获取context。                            |
| initTiling | 输入      | 通信域初始化Mc2InitTiling的地址。Mc2InitTiling在Host侧计算得出，具体请参考表1 Mc2InitTiling参数说明，由框架传递到Kernel函数中使用。 |

#### 返回值说明

无

#### 约束说明

- 本接口必须与SetCcTilingV2接口配合使用。
- 调用本接口时，必须使用标准C++语法定义TilingData结构体的开发方式，具体请参考使用标准C++语法定义Tiling结构体。
- 调用本接口传入的initTiling参数，不能使用Global Memory地址，建议通过GET_TILING_DATA_WITH_STRUCT接口获取TilingData的栈地址。
- 本接口不支持使用相同的context初始化多个HCCL对象。

#### 调用示例

用户自定义TilingData结构体：

| 12345 | classUserCustomTilingData{AscendC:tiling:Mc2InitTilinginitTiling;AscendC:tiling:Mc2CcTilingtiling;CustomTilingparam;}; |
| ----- | ---------------------------------------------------------------------------------------------------------------------- |

| 1234567891011 | extern"C"__global____aicore__voiduserKernel(GM_ADDRaGM,GM_ADDRworkspaceGM,GM_ADDRtilingGM){REGISTER_TILING_DEFAULT(UserCustomTilingData);GET_TILING_DATA_WITH_STRUCT(UserCustomTilingData,tilingData,tilingGM);GM_ADDRcontextGM=AscendC:GetHcclContext<0>();Hcclhccl;hccl.InitV2(contextGM,&tilingData);hccl.SetCcTilingV2(offsetof(UserCustomTilingData,tiling));// 调用HCCL的Prepare、Commit、Wait、Finalize接口} |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
