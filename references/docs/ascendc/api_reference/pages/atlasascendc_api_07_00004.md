# REGISTER_TILING_FOR_TILINGKEY-Kernel Tiling-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00004
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00004.html
---

# REGISTER_TILING_FOR_TILINGKEY

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

用于在kernel侧注册与TilingKey相匹配的TilingData自定义结构体；该接口需提供一个逻辑表达式，逻辑表达式以字符串“TILING_KEY_VAR”代指实际TilingKey，表达TilingKey所满足的范围。

#### 函数原型

| 1   | REGISTER_TILING_FOR_TILINGKEY(EXPRESSION,TILING_STRUCT) |
| --- | ------------------------------------------------------- |

#### 参数说明

| 参数          | 输入/输出 | 说明                                                      |
| ------------- | --------- | --------------------------------------------------------- |
| EXPRESSION    | 输入      | EXPRESSION为逻辑运算，其中用TILING_KEY_VAR指代TilingKey。 |
| TILING_STRUCT | 输入      | 用户注册的与TilingKey相匹配的TilingData自定义结构体。     |

#### 约束说明

- 使用该接口时，需确保已使用REGISTER_TILING_DEFAULT注册默认的用户自定义TilingData结构体，用于告知框架侧用户使用标准C++语法来定义TilingData。
- EXPRESSION当前支持位运算：&、|、~、^；移位运算符：<<、>>；算术运算：+、-、*、/、%；条件运算符：==、！=、>、<、>=、<=；逻辑与&&、或||以及()。优先级同C++。
- 若TilingData结构体在命名空间内，注册时需要携带对应的命名空间作用域符。
- 不支持同个TilingKey指向不同TilingData结构体，会出现拦截报错。
- 暂不支持kernel直调工程。

#### 调用示例

| 1234567891011121314151617181920212223 | extern"C"__global____aicore__voidadd_custom(__gm__uint8_t*x,__gm__uint8_t*y,__gm__uint8_t*z,__gm__uint8_t*tiling){REGISTER_TILING_DEFAULT(optiling:TilingData);// 注册用户默认自定义TilingData结构体REGISTER_TILING_FOR_TILINGKEY("TILING_KEY_VAR == 1",optiling:TilingDataA);// 注册TilingKey为1的TilingData结构体REGISTER_TILING_FOR_TILINGKEY("(TILING_KEY_VAR >= 10) && (TILING_KEY_VAR <= 15)",optiling:TilingDataB);// 注册TilingKey在[10,15]之间的TilingData结构体REGISTER_TILING_FOR_TILINGKEY("TILING_KEY_VAR & 0xFF",optiling:TilingDataC);// 注册TilingKey低16位为1的TilingData结构体if(TILING_KEY_IS(1)){GET_TILING_DATA_WITH_STRUCT(optiling:TilingDataA,tilingData,tiling);......}elseif(TILING_KEY_IS(11)){GET_TILING_DATA_WITH_STRUCT(optiling:TilingDataB,tilingData,tiling);......}elseif(TILING_KEY_IS(14)){GET_TILING_DATA_WITH_STRUCT(optiling:TilingDataB,tilingData,tiling);......}elseif(TILING_KEY_IS(255)){GET_TILING_DATA_WITH_STRUCT(optiling:TilingDataC,tilingData,tiling);......}else{GET_TILING_DATA(tilingData,tiling);......}} |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

使用标准C++语法注册tiling结构体：

| 123456789101112 | classTilingDataA{public:...};classTilingDataB{public:...};classTilingDataC{public:...}; |
| --------------- | --------------------------------------------------------------------------------------- |

配套的host侧tiling函数示例：

| 123456789101112131415161718192021222324 | ge:graphStatusTilingFunc(gert:TilingContext*context){// 其他代码逻辑...if(condition1){context->SetTilingKey(1);optiling:TilingDataA*Addtiling=context->GetTilingData<optiling:TilingDataA>();...}elseif(condition2){context->SetTilingKey(11);optiling:TilingDataB*Addtiling=context->GetTilingData<optiling:TilingDataB>();...}elseif(condition3){context->SetTilingKey(14);optiling:TilingDataB*Addtiling=context->GetTilingData<optiling:TilingDataB>();...}elseif(condition4){context->SetTilingKey(255);optiling:TilingDataC*Addtiling=context->GetTilingData<optiling:TilingDataC>();...}...// 其他代码逻辑} |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
