# REGISTER_TILING_DEFAULT-Kernel Tiling-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00003
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00003.html
---

# REGISTER_TILING_DEFAULT

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | √        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | √        |
| Atlas训练系列产品                           | x        |

#### 功能说明

用于在kernel侧注册用户使用标准C++语法自定义的默认TilingData结构体。

注册TilingData结构体用于告知框架侧用户使用标准C++语法来定义TilingData，同时告知框架TilingData结构体类型，用于框架做tiling数据解析。

#### 函数原型

| 1   | REGISTER_TILING_DEFAULT(TILING_STRUCT) |
| --- | -------------------------------------- |

#### 参数说明

| 参数          | 输入/输出 | 说明                                   |
| ------------- | --------- | -------------------------------------- |
| TILING_STRUCT | 输入      | 用户注册的默认自定义TilingData结构体。 |

#### 约束说明

- 若TilingData结构体在命名空间内，注册时需要携带对应的命名空间作用域符。
- 暂不支持Kernel直调工程。

#### 调用示例

| 12345678 | extern"C"__global____aicore__voidadd_custom(__gm__uint8_t*x,__gm__uint8_t*y,__gm__uint8_t*z,__gm__uint8_t*tiling){REGISTER_TILING_DEFAULT(optiling:TilingData);GET_TILING_DATA(tilingData,tiling);KernelAddop;op.Init(x,y,z,tilingData.blkDim,tilingData.totalSize,tilingData.splitTile);op.Process();} |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
