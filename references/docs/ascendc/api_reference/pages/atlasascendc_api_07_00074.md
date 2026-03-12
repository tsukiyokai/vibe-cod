# GetTilingFunc-OpTilingRegistry-Tiling调测-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_00074
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_00074.html
---

# GetTilingFunc

#### 功能说明

根据算子类型OpType获取对应的Tiling函数。

#### 函数原型

| 1   | TilingFuncGetTilingFunc(constchar*opType)const |
| --- | ---------------------------------------------- |

#### 参数说明

| 参数名 | 输入/输出 | 描述                                       |
| ------ | --------- | ------------------------------------------ |
| opType | 输入      | 算子类型，与原型定义时的算子类型保持一致。 |

#### 返回值说明

获取成功，则返回对应的Tiling函数指针；失败则返回空指针。Tiling函数指针定义如下：

| 1   | usingTilingFunc=uint32_t(*)(gert:TilingContext*) |
| --- | ------------------------------------------------ |

#### 约束说明

无

#### 调用示例

| 123456789101112131415 | context_ascendc:OpTilingRegistrytmpIns;boolflag=tmpIns.LoadTilingLibrary("/your/path/to/so_path/liboptiling.so");if(flag==false){std:cout<<"Load tiling so failed"<<std:endl;return-1;}context_ascendc:TilingFunctilingFunc=tmpIns.GetTilingFunc("AddCustom");if(tilingFunc!=nullptr){//  use tiling func...}else{std:cout<<"get tiling func failed."<<std:endl;return-1;}// ... |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
