# InitSpmBuffer-TPipe-Pipe和Que框架-资源管理-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0166
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0166.html
---

# InitSpmBuffer

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | √        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | √        |

#### 功能说明

初始化SPM Buffer。

#### 函数原型

- 暂存到workspace初始化，需要指定GM地址为SPM Buffer：12template<typenameT>__aicore__inlinevoidInitSpmBuffer(constGlobalTensor<T>&workspace,constint32_tbufferSize)
- 暂存到L1 Buffer初始化，不需要指定地址，会默认暂存到L1 Buffer，只需要传入需要的SPM Buffer大小：1__aicore__inlinevoidInitSpmBuffer(constint32_tbufferSize)Atlas A2 训练系列产品/Atlas A2 推理系列产品，不支持暂存到L1 Buffer初始化接口。Atlas A3 训练系列产品/Atlas A3 推理系列产品，不支持暂存到L1 Buffer初始化接口。

#### 参数说明

| 参数名     | 输入/输出 | 含义                           |
| ---------- | --------- | ------------------------------ |
| workspace  | 输入      | workspace地址。                |
| bufferSize | 输入      | SPM Buffer的大小，单位是字节。 |

#### 约束说明

无

#### 返回值说明

无

#### 调用示例

- 暂存到workspace初始化12345AscendC:TPipepipe;intlen=1024;// 设置spm buffer为1024个类型为T的数据workspace_gm.SetGlobalBuffer((__gm__T*)usrWorkspace,len);// 此处的usrWorkspace为用户自定义的workspaceautogm=workspace_gm[AscendC:GetBlockIdx()*len];pipe.InitSpmBuffer(gm,len*sizeof(T));
- 暂存到L1 Buffer初始化123AscendC:TPipepipe;intlen=1024;// 设置spm buffer为1024个类型为T的数据pipe.InitSpmBuffer(len*sizeof(T));
